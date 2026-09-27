# Config Backup — Oxidized to Self-Hosted Git, Four Vendors

> Part of [CASEY-LAB](https://github.com/117caseyallen-NetAdm/casey-lab), a
> dual-site multi-vendor homelab. The hub has the topology and the
> [verification output](https://github.com/117caseyallen-NetAdm/casey-lab/blob/main/docs/verification.md).

Automated running-config backup for six network devices across **Cisco IOS
(15.x and 12.1), Juniper Junos, Palo Alto PAN-OS, and Arista EOS**, polled
hourly by [Oxidized](https://github.com/ytti/oxidized), committed to Git only
when something changed, and pushed to a self-hosted Gitea server — unattended,
under systemd, surviving a reboot. It logs in as a read-only service account
authenticated by TACACS+, not as an administrator.

An interface description added on a switch appears in Gitea as a one-line diff,
attributed and timestamped, with no human action:

```
Oxidized::Worker -- Configuration updated for /3560CG-1
To 10.99.20.31:OWNER/network-configs.git
   9f62ffe..0c18205  master -> master
```

## What's here

- **[configs/](configs/)** — the Oxidized configuration, the device source, both
  systemd units, and the push hook. Sanitized; the comments explain each
  non-obvious line
- **[docs/build-notes.md](docs/build-notes.md)** — eleven stages, each ending in
  the check that proved it
- **[docs/troubleshooting.md](docs/troubleshooting.md)** — every problem that
  actually occurred, including four dead ends on the Gitea push, one of which
  took backups down

## Placement: management VLAN, no ACL changes

Every device in the lab restricts management access to a short permit list — two
jumpboxes and the management subnets — enforced in four vendor syntaxes (IOS
`access-class`, a numbered ACL on IOS 12.1, a Junos `lo0` filter, a PAN-OS
Interface Management Profile). Detail in the
[hub README](https://github.com/117caseyallen-NetAdm/casey-lab#management-access).

The obvious place for a new container is the server VLAN. That would have meant
editing six device ACLs in four syntaxes, each edit a lockout risk and a
permanent piece of drift, to grant an access the design already accounts for.

Instead the container sits on the **management VLAN** (`10.99.20.0/24`), which
the permit list already includes. It reached all six devices on first contact
with no ACL changes.

The cost: anything on that subnet inherits device-admin reach. It is now a trust
boundary rather than just a subnet, so only management tooling goes there — and
if that discipline slips, the permit lists have to become host-based after all.

## SSH: what failed and what did not

Two findings that are not in the Oxidized documentation.

**The three oldest switches needed no workarounds.** Modern OpenSSH refuses to
connect to the two Catalyst 3560CGs and the 2003 Catalyst 2940 — they offer only
SHA-1 key exchange, and the 2940 offers *only* `diffie-hellman-group1-sha1`.
Reaching them interactively needs `KexAlgorithms` overrides. Oxidized reached all
three with no algorithm configuration at all, because it speaks SSH through Ruby's
`net-ssh`, which still implements those algorithms. The refusal was **OpenSSH
client policy** — a deliberate decision by one implementation — not the switches,
the protocol, or SSH libraries generally.

**The device that failed was the newest one, and not for the reason it looked
like.** The Arista 710P failed authentication, which presented as a credentials
problem:

```
ssh -v admin@<arista>   → Authentications that can continue: publickey,keyboard-interactive
ssh -v admin@<catalyst> → Authentications that can continue: publickey,keyboard-interactive,password
```

EOS doesn't offer the `password` authentication method, and Oxidized's default
list is `[none, publickey, password]`. `keyboard-interactive` renders as
`Password:` on screen — it is a generic challenge-response mechanism and a
password prompt is its most common payload — so the difference is invisible to a
person and decisive for automation. Fixed with two model vars:
`auth_methods` to log in, `enable: true` to reach privileged mode afterwards.
Full account in
[docs/troubleshooting.md](docs/troubleshooting.md#the-arista-keyboard-interactive-not-enable).

## After TACACS+: a backup tool is also a login

When the fleet moved to [centralized AAA](https://github.com/117caseyallen-NetAdm/homelab-tacacs-aaa),
Oxidized turned out to be the most demanding user on the network — it logs into
every device every hour, and it had been doing so with the fleet admin password.

**Converting the first device broke its backup.** Oxidized authenticated as the
local admin account, and on IOS a local account can't log in over SSH while the
TACACS+ server is up and rejecting it. The failure was visible from both ends:

```
CA-OXI-LAB:  Net::SSH::AuthenticationFailed … @10.99.20.1
CA-TAC-LAB:  10.99.20.1  …  tty1  10.99.20.30  shell login failed
```

And the credential was global, so fixing that one device would have broken the
other five. Rolling out AAA to devices an automated system already logs into is
a **credential migration**, not a device-config change. Per-node credentials in
`router.db` had to come first, and each device's entry moved to the service
account as that device was converted.

Once it ran as a read-only account, it found things the admin account had been
hiding:

| Device | What happened | Fix |
|---|---|---|
| SRX345 | The stock Junos `read-only` class has `view` but not `view-configuration`. The backup "succeeded" with 33 lines of empty hierarchy, and Oxidized committed and pushed it | A custom class with `view-configuration` — and **without** the `secret` bit, so secrets come back as `## SECRET-DATA` from the device itself |
| Arista 710P | The `eos` model redacts `tacacs-server key 7 …` but not the key on the host line. The TACACS+ key reached the backup repo | Model override, borrowing the general pattern the `ios` model already has |
| PA-440 | Backups carried the chassis serial; the model strips eight version fields from `show system info` and leaves `serial:` | Model override |
| PA-440 | `show config running` includes the whole App-ID catalogue — ~70,000 lines, 39–50 seconds against a 30-second timeout. Retried three times, gave up, and git history still looked fine, just stale | `timeout: 600` as a stopgap; a scoped collection command is the real fix |

Both overrides live in
[homelab-tacacs-aaa/configs/oxidized-model-overrides](https://github.com/117caseyallen-NetAdm/homelab-tacacs-aaa/tree/main/configs/oxidized-model-overrides).

The lesson from the whole table: **a backup system reports success until the
day you need a config.** Three of those four committed something wrong; the
fourth committed nothing while the history still looked healthy. The REST API (`/nodes.json`) returns
each node's last status and duration, and duration is the leading indicator:
the PA-440 took six times the median before it ever timed out.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Credentials | Per-node entries in `router.db`, all a read-only TACACS+ service account | Least privilege, one identity in every audit log, and a device can be converted without touching the others |
| Placement | Management VLAN | Already permitted on every device; no ACL changes across four syntaxes |
| Install | Native gem, not Docker | Docker would hide the files this build is about |
| Device source | CSV `router.db` | NetBox is the intended source; CSV until it exists |
| Output | Local bare repo first, Gitea second | Prove reaching six devices before adding a second service to the failure surface |
| Push to Gitea | `exec` hook running the `git` binary | Oxidized's `githubrepo` hook uses rugged/libgit2, compiled without SSH transport; rebuilding it broke backups |
| Git server | Gitea, SQLite | ~200 MB of RAM against several GB for GitLab CE |
| Interval | Hourly | Commits happen only on change, so a short interval adds detection speed, not noise |
| Timezone | UTC on every device | Local time isn't monotonic — DST duplicates an hour and skips an hour, so timestamps stop being unique or ordered |
| Secrets | `remove_secret: true`, two model overrides; Gitea repo private; no device config published here | Redaction is a regex, so it fails open — it missed the Arista's key and the PA's serial. The repo stays private regardless |
| Web UI / API | `oxidized-web` bound to `127.0.0.1` | It has no authentication and serves every device config. Local-only still lets a monitoring agent on the same host poll it |

## What this is, and isn't

Config archival with change detection, **not** a CI/CD pipeline: the device is
the source of truth and Git records what it finds, so nothing here can stop a bad
change from reaching a device. Inverting that — Batfish validating a proposed
change before deployment — is the NetDevOps item on the
[hub roadmap](https://github.com/117caseyallen-NetAdm/casey-lab#roadmap).

It has since become the independent witness for the lab's automation. The
[Ansible pipeline](https://github.com/117caseyallen-NetAdm/homelab-network-automation)
knows nothing about this tool and this tool knows nothing about it, yet every
change the pipeline made appeared here on the next poll, in each device's own
syntax — `logging host 10.99.20.32` on the 3560s and the Arista,
`logging 10.99.20.32` on the 2003 switch.
[Evidence](https://github.com/117caseyallen-NetAdm/homelab-network-automation/blob/main/docs/verification.md#8-the-backup-system-saw-every-change).

**Next:** alerting on backup age and duration from the REST API, a scoped
PAN-OS collection command, a `post_store` hook to syslog for change alerts, and
NetBox as the device source. ~~A per-device read-only service account~~ — done,
via [TACACS+](https://github.com/117caseyallen-NetAdm/homelab-tacacs-aaa).

---

*Internal RFC1918 addressing and hostnames are real. No credentials, keys, public
addresses, or device configurations appear in this repository.*
