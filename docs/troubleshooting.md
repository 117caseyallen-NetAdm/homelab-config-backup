# Troubleshooting Log

Every real problem hit during the build, what it looked like, and what it
actually was.

The two network-specific ones are
[the Arista](#the-arista-keyboard-interactive-not-enable) and
[the legacy-crypto wall](#the-legacy-crypto-wall-that-wasnt-there); the longest
is [pushing to Gitea](#pushing-to-gitea-four-dead-ends).

## Contents

1. [`oxidized: command not found` — three different causes](#oxidized-command-not-found--three-different-causes)
2. [`debug: true` logs the device password](#debug-true-logs-the-device-password)
3. [The Arista: keyboard-interactive, not enable](#the-arista-keyboard-interactive-not-enable)
4. [Pushing to Gitea: four dead ends](#pushing-to-gitea-four-dead-ends)
5. [Secrets in the git history](#secrets-in-the-git-history)
6. [The legacy-crypto wall that wasn't there](#the-legacy-crypto-wall-that-wasnt-there)
7. [After TACACS+: the login that broke first](#after-tacacs-the-login-that-broke-first)
8. [A Junos backup that succeeded with nothing in it](#a-junos-backup-that-succeeded-with-nothing-in-it)
9. [The PA-440: three retries, then silence](#the-pa-440-three-retries-then-silence)
10. [Two redaction gaps in the bundled models](#two-redaction-gaps-in-the-bundled-models)

---

## `oxidized: command not found` — three different causes

The same message, three unrelated problems, in one afternoon. None of them was a
broken install.

### 1. A build dependency, three commands earlier

`gem install oxidized` printed several hundred lines and, buried in the middle:

```
checking for yaml.h... no
yaml.h not found
*** extconf.rb failed ***
ERROR:  Error installing oxidized:
        ERROR: Failed to build gem native extension.
```

The `psych` gem — Ruby's YAML parser — is a C binding to libyaml, and
`libyaml-dev` wasn't installed. `psych` is a dependency of `oxidized`, so the
whole install aborted. But the *visible* symptom was `command not found` at the
verification step, which reads like a PATH problem.

A native-extension failure always names the gem that failed, and the gem's name
tells you which `-dev` package is absent: `psych` → YAML → `libyaml-dev`. The
genuinely hard build — `rugged`, which compiles all of libgit2 — had succeeded;
only the trivial dependency broke.

Also: the failure took `oxidized-script` and `oxidized-web` down with it, and
re-running `gem install oxidized` alone did not fix those.

### 2. `/usr/local/bin` not on PATH inside `pct enter`

After the dependency was fixed and the gem installed cleanly:

```
$ oxidized --version
bash: oxidized: command not found
$ find / -name oxidized -type f 2>/dev/null
/usr/local/bin/oxidized
$ echo $PATH
/sbin:/bin:/usr/sbin:/usr/bin
```

Proxmox's `pct enter` uses `lxc-attach`, which builds a minimal environment
rather than running a login shell. `/usr/local/bin` — where Debian's
`rubygems-integration` places gem executables — isn't in it. Same thing bit the
Gitea container later. **`command not found` is a claim about the environment,
not about the software.** `find` settles it before any reinstall.

### 3. Wrong user's home directory

After a reboot, running `oxidized` as root instead of as the `oxidized` user:

```
Oxidized crashed, crashfile written in /root/.config/oxidized/crash
No such file or directory @ rb_sysopen - /root/.config/oxidized/router.db
```

Oxidized reads `~/.config/oxidized/config`, and `~` for root is `/root`. It
found nothing there, **wrote a default stub config**, and crashed on the missing
`router.db`. The real config in `/opt/oxidized` was untouched. The stub had to
be deleted or it would have confused the next person.

The permanent fix is the systemd unit — `User=oxidized` plus
`Environment=HOME=/opt/oxidized` — which makes "which user am I" a question
nobody has to answer.

A fourth variant, same family: running `git` as root against the repo owned by
`oxidized` fails with `fatal: detected dubious ownership` — git's
`safe.directory` protection (CVE-2022-24765). Repo-local config can specify
commands git will execute; running as root inside another user's directory
hands that user root. The fix is to be the right user, not to add the exception.

---

## `debug: true` logs the device password

The first successful run printed, at `D` level:

```
Oxidized::Node -- setting node key 'password' to value '••••••••' from global
```

**The global `debug: true` resolves and logs every node key at startup,
credentials included** — to the console, and into the session files under
`~/.config/oxidized/logs/`. Fine for a five-minute proof, dangerous as a
default.

Turn the global logger off once the mechanism works; `input: debug: true` can
stay — it gives session transcripts without the credential dump. Delete the log
directory from the debug runs.

The structural issue is that the credential in that file is the fleet-wide admin
password. Oxidized needs exactly one capability: read the running config. A
per-device read-only service account shrinks the blast radius; TACACS+ with
command authorization is the proper version.

*Update:* done. It now runs as a read-only TACACS+ service account, and the
fleet admin password is gone from its config — see
[the TACACS+ entries below](#after-tacacs-the-login-that-broke-first).

---

## The Arista: keyboard-interactive, not enable

Five of six devices backed up on the first full run. The Arista 710P — the
*modern* switch — did not. I misdiagnosed it twice before reading the actual
error.

**Theory one (wrong): privilege.** Interactively, the Ciscos land at `#` and
the Arista lands at `>`. EOS needs `enable` before `show running-config`, and
the eos model's source confirmed the whole `post_login` block is gated on an
`enable` var. Plausible. Wrong.

**What the log said** once global debug was off and stopped burying it:

```
Oxidized::Node -- 10.99.10.14 raised Net::SSH::AuthenticationFailed
  with msg "Authentication failed for user Admin@10.99.10.14"
```

It never logged in. Every theory about what happens *after* login was reasoning
about a stage the connection never reached.

**One command settled it:**

```
ssh -v admin@10.99.10.14 2>&1 | grep -i "authentications that can continue"
debug1: Authentications that can continue: publickey,keyboard-interactive

ssh -v admin@10.99.10.1 2>&1 | grep -i "authentications that can continue"
debug1: Authentications that can continue: publickey,keyboard-interactive,password
```

**The Arista does not offer the `password` authentication method.** Oxidized's
SSH input defaults to `auth_methods: [none, publickey, password]`. There's no
key, `password` isn't offered, `none` fails — methods exhausted, reported as an
authentication failure.

`keyboard-interactive` renders as `Password:` on screen. It is a generic
challenge-response mechanism — the server sends arbitrary prompts, the client
answers — and a password prompt is merely its most common payload. The
`password` method is a separate, dedicated protocol message. Identical at a
terminal, different on the wire.

Fix — both vars, for two different stages:

```yaml
models:
  eos:
    vars:
      enable: true
      auth_methods: [none, publickey, password, keyboard-interactive]
```

`auth_methods` to log in; `enable` to reach privileged mode afterwards. The
first theory wasn't wrong about EOS needing enable — it was wrong about which
failure was happening. The value has to be a real YAML boolean (`true`), not a
string from the CSV source, because the model tests `is_a? TrueClass`.

The wrong theory came from a plausible observation formed before the real error
was readable: global `debug: true` had buried the one useful line under
thousands of packet traces.

---

## Pushing to Gitea: four dead ends

Backups were committing locally. Getting them to push to Gitea automatically
took four attempts, one of which took backups down.

### Dead end 1 — a config key that didn't exist

`remote_repo:` was set under `output: git:`, where it does nothing. No error, no
warning, no log line — Oxidized created the repo, committed six configs, and
never registered a remote. `git remote -v` was empty.

**An unrecognized YAML key is not an error.** It parses, nothing reads it, and
you get silence. `grep -rn remote_repo <gem directory>` settled it: the key
belongs to the **`githubrepo` hook**, not the output.

### Dead end 2 — an ed25519 key that can't be PEM

The `githubrepo` hook's docs:

> The `privatekey` must be in the legacy PEM format (`BEGIN RSA PRIVATE KEY`),
> not the newer OpenSSH format.

`BEGIN RSA PRIVATE KEY` is an RSA-only container. Ed25519 private keys are
*only* defined in the OpenSSH format — `ssh-keygen -m PEM` can't convert one
because there's nothing to convert it into. A separate RSA key
(`ssh-keygen -t rsa -b 4096 -m PEM`) was required. **A library's age dictated
the key algorithm.**

### Dead end 3 — the library has no SSH transport

With the hook correctly configured and an RSA/PEM key in place:

```
GithubRepo -- Rugged does not support the git URL 'git@10.99.20.31:OWNER/network-configs.git'.
GithubRepo -- Note: Rugged isn't installed with ssh support. You may need "gem install rugged -- --with-ssh"
Rugged::NetworkError: unsupported URL protocol
```

The `rugged` gem had compiled libgit2 **without linking libssh2**. `git@host:path`
is a protocol it doesn't recognize. Not an auth failure — a *capability*
failure, which is why the error talks about protocols rather than credentials.

### Dead end 4 — the rebuild broke backups

The error message suggested the fix, so it felt safe:

```
gem install rugged -- --with-ssh
```

libgit2 configured and compiled **successfully with SSH** (`* SSH, using
libssh2` in its enabled-features list). The failure came at the next step —
linking `rugged` against the freshly built libgit2, which now carried an
undeclared dependency on libssh2:

```
checking for -lgit2... *** extconf.rb failed ***
The compiler failed to generate an executable file. (RuntimeError)
You have to install development tools first.
```

That last line is `mkmf`'s generic message for any failed link, and sends
people installing packages they already have. **Read which step failed, not the
message it printed.**

Worse, the failed build left the gem unusable:

```
Ignoring rugged-1.9.6 because its extensions are not built
cannot load such file -- rugged (LoadError)
```

Oxidized's git output depends on rugged. **Config backup was down** until
`gem pristine rugged --version 1.9.6` restored the original build.

**Recompiling a dependency that a running service uses is a change with a
regression path**, and the fact that the suggestion came from the tool's own
error message doesn't change that.

### What worked — an `exec` hook calling the `git` binary

```yaml
hooks:
  push_to_gitea:
    type: exec
    events: [post_store]
    cmd: 'git -C /opt/oxidized/configs.git push origin master'
    async: false
    timeout: 120
```

Oxidized runs as the `oxidized` user; the hook inherits that identity and uses
`~/.ssh/id_ed25519` — the key that had already pushed to Gitea by hand. Rugged
is not involved in the push at all. The RSA/PEM key from dead end 2 became
unnecessary. The same hook mechanism later points at syslog for change alerts.

---

## Secrets in the git history

`remove_secret: true` went into the config after the first few commits. Those
commits held unredacted running configs — SNMP communities, `password 7`
strings, the lot.

Removing a secret in a later commit changes only what's checked out. The blob
is still in the object database and still reachable. **A secret committed to
git is committed forever unless the history is rewritten.**

Here it cost sixty seconds: the repo was still local and disposable, so it was
deleted and Oxidized rebuilt it with redaction active from the first commit,
*before* anything was pushed to Gitea.

---

## The legacy-crypto wall that wasn't there

Three switches in this lab — two Catalyst 3560CGs and a 2003 Catalyst 2940 —
offer only SHA-1 key exchange. The 2940 offers only
`diffie-hellman-group1-sha1`, a 1024-bit fixed group from RFC 2409 (1998).
Modern OpenSSH refuses all three by default; reaching them interactively needs
`KexAlgorithms` overrides. The PA-440 has a related but distinct problem: its
key exchange is fine, but it holds only an `ssh-rsa` host key, which OpenSSH
also dropped — a different error, `no matching host key type`, for a different
half of the handshake.

The plan assumed Oxidized would hit the same walls and need per-device
algorithm settings. Prediction, before running it:

```
ruby -e "require 'net/ssh'; puts Net::SSH::Transport::Algorithms::ALGORITHMS[:kex]"
...
diffie-hellman-group14-sha1
diffie-hellman-group-exchange-sha1
diffie-hellman-group1-sha1
```

Both legacy groups still present in `net-ssh` 7.3.3. And in practice, **all
four devices backed up with no algorithm configuration at all.** Oxidized passes
`append_all_supported_algorithms`, so net-ssh proposes everything it has, and the
switches accept.

The wall was OpenSSH's **client policy** — a deliberate decision by one
implementation to stop offering algorithms it considers weak. Swap the
implementation and the wall isn't there. Same devices, same protocol, same
network; the only variable was which client was imposing the policy.

NAPALM and Nornir use Paramiko, which has its own algorithm policy and will need
its own test.

---

## After TACACS+: the login that broke first

Converting the first switch to TACACS+ broke its backup within the hour.
Oxidized was logging in as the local admin account, and on IOS the method list
`group TAC-GROUP local` only falls back to `local` when the server is
*unreachable*. A server that answers "no" is final, so a local account the
server has never heard of can't log in over SSH while the server is healthy.

Visible from both ends, with the backup host's address in the server's log:

```text
CA-OXI-LAB:  Net::SSH::AuthenticationFailed … @10.99.20.1
CA-TAC-LAB:  10.99.20.1  …  tty1  10.99.20.30  shell login failed
```

The fix wasn't on the switch. The credential was **global**, so the next five
conversions would each have broken another device's backup. `router.db` went to
five fields, with a credential per node, and each device moved to the read-only
service account as it was converted. The global pair was deleted last, and only
after all six had backed up without it.

---

## A Junos backup that succeeded with nothing in it

With the service account mapped to the SRX's stock `read-only` login class, the
next backup committed and pushed **33 lines**: hierarchy headers with almost
nothing underneath. The admin account's backup had been 625 lines.

Junos gates what you can see on permission bits. `read-only` has `view` but not
`view-configuration`, so the account could confirm each hierarchy existed and
see nothing inside it. Oxidized had no way to know that, so it reported success.

A custom class fixed it, and chose what to leave out as carefully as what to
put in:

```text
set system login class netops-ro permissions [ view view-configuration ]
```

No `secret` bit, so Junos itself replaces every encrypted value with
`## SECRET-DATA`. That's a redaction done by a permission bit, which fails
closed, instead of a regex, which fails open. Result: 654 lines, five
`SECRET-DATA` markers, nothing unredacted.

This was worse than a loud failure: it would have been found during a restore.

---

## The PA-440: three retries, then silence

```text
W 10.99.0.1 raised Timeout::Error with msg "execution expired"
W /PA440-LAB status no_connection, retry attempt 1
W /PA440-LAB status no_connection, retry attempt 2
W /PA440-LAB status no_connection, retry attempt 3
W /PA440-LAB status no_connection, retries exhausted, giving up
```

Authentication succeeded every time; the firewall's own log showed the service
account logging in via CLI, which also settled that the `panos` model uses SSH,
not the XML API. The job was timing out *after* login.

The model runs `show config running`, and on PAN-OS that includes the entire
predefined App-ID catalogue — about 70,000 lines. It takes 39 to 50 seconds. The
top-level `timeout:` was 30.

Two traps on the way to that:

- **The first `timeout:` changed was the wrong one.** There's also one under
  `hooks:`, and it governs the git push. The 40-second spacing between retries
  had been showing the real value, 30, all along.
- **Git history looked fine.** Six commits for the device, all stale, and
  nothing reported the failure. It turned up only because the firewall's syslog
  was being forwarded for an unrelated reason and the login pattern looked odd.

`timeout: 600` is a stopgap. The fix is to collect the local configuration
rather than the vendor's catalogue.

---

## Two redaction gaps in the bundled models

`remove_secret: true` is a per-model denylist, and moving to TACACS+ put new
kinds of line into the configs. Two got through:

- **The Arista's TACACS+ key.** The `eos` model redacts
  `tacacs-server key 7 …`, but EOS puts the key on the host line:
  `tacacs-server host 10.99.20.32 timeout 5 key 7 …`. The `ios` model has
  always had the general pattern, `^(tacacs-server (.+ )?key) .+`. Copying that
  one line into an `eos.rb` override fixed it.
- **The PA-440's chassis serial.** The `panos` model strips eight version fields
  from `show system info` and leaves `serial:`.

Both overrides, and how to install them so they survive a gem upgrade, are in
[homelab-tacacs-aaa](https://github.com/117caseyallen-NetAdm/homelab-tacacs-aaa/tree/main/configs/oxidized-model-overrides).
Neither leak left the private Gitea server, and this is why that server stays
private.
