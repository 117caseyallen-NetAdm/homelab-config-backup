# Build Notes

The build in the order it happened, with the check at the end of each stage.
Two unprivileged Debian 12 LXC containers on Proxmox VE, both on the
management VLAN.

| | CA-OXI-LAB | CA-GIT-LAB |
|---|---|---|
| Role | Oxidized 0.37.0 + web UI | Gitea 1.27.3 |
| Bridge | `mgmt99` (VLAN 99) | `mgmt99` |
| IP | `10.99.20.30/24` | `10.99.20.31/24` |
| Size | 2 vCPU / 1 GB / 8 GB | 2 vCPU / 2 GB / 16 GB |
| Runs as | user `oxidized`, systemd | user `git`, systemd |

**The ordering principle: prove each layer before adding the next.** SSH by hand
before configuring Oxidized, one device before six, local git before a remote.

## 1. Containers

```
pct create 102 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
  --hostname CA-OXI-LAB --cores 2 --memory 1024 --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=mgmt99,ip=10.99.20.30/24,gw=10.99.20.1 \
  --nameserver 10.20.2.5 --searchdomain casey.corp \
  --unprivileged 1 --features nesting=1 --onboot 1 --start 1
```

`mgmt99` is a VLAN-aware bridge stacked on the trunk (`vmbr1.99`); it applies
the tag itself, so the guest interface carries **no `tag=`**. Adding one
double-tags the frames. Same for the Gitea container at `.31`.

**Check:** ping the local gateway, then a device at the *far* site
(`10.99.10.1`, reached across OSPF and the IPsec tunnel), then the internet.

```
64 bytes from 10.99.10.1: icmp_seq=1 ttl=252 time=1.87 ms
```

`ttl=252` — three routers in the return path. The management plane is routed
end to end; anything failing after this is the application's problem.

## 2. Oxidized install

```
apt -y install ruby ruby-dev ruby-bundler build-essential \
  pkg-config cmake libssl-dev libssh2-1-dev libicu-dev libyaml-dev \
  libsqlite3-dev zlib1g-dev git
gem install oxidized oxidized-script oxidized-web
```

Why the odd ones: `cmake` + `libssl-dev` because the `rugged` gem compiles
**libgit2** from source (Oxidized writes commits through a library, not the
`git` binary); `libyaml-dev` because the `psych` YAML parser is a C binding —
omitting it fails the build in a way that surfaces as `oxidized: command not
found` three commands later (see troubleshooting).

Dedicated service account — the tool has no reason to run as root:

```
adduser --system --group --home /opt/oxidized --shell /bin/bash oxidized
mkdir -p /opt/oxidized/.config/oxidized
chown -R oxidized:oxidized /opt/oxidized
```

**Check:** `oxidized --version` and
`ruby -e "require 'rugged'; puts Rugged::Version"` both print a version.

## 3. SSH to every device by hand first

Before Oxidized exists, as the `oxidized` user, SSH to all six and record
exactly which fail and how:

| Device | Result | Meaning |
|---|---|---|
| Arista, SRX | logged in | fine |
| 3560CG-1, 3560CG-2, C2940 | `no matching key exchange method` | ACL let it in; crypto negotiation failed |
| PA-440 | `no matching host key type found. Their offer: ssh-rsa` | **different half of the handshake** — host key, not kex |

None returned `Connection refused` or timed out — so the management ACL
permitted the container everywhere, and the placement decision was validated on
first contact.

For interactive use, `~/.ssh/config` stanzas re-enable the legacy algorithms
(`KexAlgorithms +diffie-hellman-group1-sha1` for the 2940,
`+diffie-hellman-group14-sha1` for the 3560s, `HostKeyAlgorithms +ssh-rsa` for
the PA). Accept every host key now — an unanswered prompt later stalls the first
automated run in a way that looks like a hung device.

**Check:** all six log in. Working algorithm set per device written down.

## 4. One device, local git

Minimal config — one node, git output to a local bare repo. See
[configs/oxidized-config.example.yml](../configs/oxidized-config.example.yml).

```
3560CG-1:ios:10.99.10.1
```

**Check:**

```
git -C /opt/oxidized/configs.git log --oneline
git -C /opt/oxidized/configs.git show HEAD --stat
```

One commit containing the running config. Every moving part — SSH, model, git
backend, source parsing — is now proven.

## 5. All six devices

```
3560CG-1:ios:10.99.10.1
3560CG-2:ios:10.99.20.1
C2940-LAB:ios:10.99.20.20
ARISTA710P-LAB:eos:10.99.10.14
SRX345-LAB:junos:10.99.0.2
PA440-LAB:panos:10.99.0.1
```

**No algorithm vars on any line.** Five of six committed on the first run,
including the C2940. The predicted crypto wall did not exist for `net-ssh`.

The sixth — the Arista — failed authentication. Fix in the eos model vars:

```yaml
models:
  eos:
    vars:
      enable: true
      auth_methods: [none, publickey, password, keyboard-interactive]
```

**Check:** `git ls-tree -r HEAD --name-only` lists six devices.

## 6. Turn debug off, turn secret removal on

`debug: true` resolves and logs every node key at startup — **password
included** — to the console and to `~/.config/oxidized/logs/`. Off once the
mechanism is proven. `input: debug` can stay for session transcripts without
the credential dump.

`vars: remove_secret: true` strips `password 7`, SNMP communities, and similar
per model. It's a regex, not a guarantee. Because it was enabled *after* the
first commits, the repository history held unredacted configs — it was deleted
and allowed to rebuild clean while it was still disposable. Git keeps
everything; a secret committed once is in the history until the history is
rewritten.

## 7. systemd

[configs/oxidized.service](../configs/oxidized.service). Three lines do the
work: `User=oxidized`, `Environment=HOME=/opt/oxidized` (Oxidized resolves its
config path from `HOME`, and systemd does not set it reliably), and an absolute
`ExecStart` (systemd never inherits a shell's `PATH`).

**Check:** reboot the container. `systemctl status oxidized` is active without
anyone typing anything.

## 8. Gitea

Single static Go binary — no runtime, no dependency tree:

```
GITEA_VER=$(curl -sL https://api.github.com/repos/go-gitea/gitea/releases/latest \
  | sed -n 's/.*"tag_name": "v\(.*\)".*/\1/p')
wget -O /usr/local/bin/gitea https://dl.gitea.com/gitea/${GITEA_VER}/gitea-${GITEA_VER}-linux-amd64
chmod +x /usr/local/bin/gitea
```

Service account, directories, [configs/gitea.service](../configs/gitea.service).
Web setup at `:3000`: SQLite, base URL set to the container's address (it's
baked into every clone URL Gitea hands out), **self-registration disabled**,
admin account created on the setup page rather than left for the first visitor.

Repository `network-configs`, **private**. Oxidized's ed25519 public key added
as a **deploy key with write access** — scoped to one repository, so a
compromised backup host can push to that repo and nothing else.

**Check:** `ssh -T git@10.99.20.31` as the `oxidized` user returns Gitea's
greeting, not a shell. `cat /home/git/.ssh/authorized_keys` on the Gitea box
shows the key wrapped in a `command="gitea ... serv"` forced command with
`no-pty,restrict` — that's why a key with SSH access to a Linux box cannot get
a shell.

## 9. Push on change

Register the remote once by hand (Oxidized never does this), then the hook:

```yaml
hooks:
  push_to_gitea:
    type: exec
    events: [post_store]
    cmd: 'git -C /opt/oxidized/configs.git push origin master'
    async: false
    timeout: 120
```

`post_store` fires **only when a config changed** — exactly when a push is
worth doing. The hook inherits the `oxidized` user and its ed25519 key. Why this
and not the built-in `githubrepo` hook is the longest entry in troubleshooting.

**Check** — change an interface description on a switch, wait one poll:

```
Oxidized::Worker -- Configuration updated for /3560CG-1
To 10.99.20.31:OWNER/network-configs.git
   9f62ffe..0c18205  master -> master
0c18205 (HEAD -> master, origin/master) update /3560CG-1
```

Polled, detected, committed, pushed — `HEAD` and `origin/master` on the same
commit, with no human action.

## 10. Time

Every device was free-running. Oxidized timestamps commits in UTC; correlating
one against a device's own log requires the clocks to agree. Five of the six now
sync to the domain's PDC emulator, which follows four external peers; the sixth,
the PA-440, sources service traffic from an uncabled MGT port and needs a service
route. Devices set to UTC. What to look for in `show ntp associations` is
the `*` — the peer the system **selected**, not merely configured — and `reach`
climbing to `377`, octal for eight consecutive good polls. Captured output in
the hub's
[verification.md](https://github.com/117caseyallen-NetAdm/casey-lab/blob/main/docs/verification.md#6-one-time-hierarchy-across-the-fabric).

## Open

- **Per-device read-only service account.** Oxidized currently authenticates
  with the fleet admin credential, held in a `0600` file. It needs exactly one
  capability — read the running config. TACACS+ with command authorization is
  the real answer; a local read-only user is the interim one.
- **`post_store` → syslog.** Same hook mechanism, pointed at a log collector,
  turns "the config on X changed" into a push notification.
- **NetBox as the device source** once NetBox exists, replacing `router.db`.
