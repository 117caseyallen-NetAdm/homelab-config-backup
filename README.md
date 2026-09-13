# Config Backup — Oxidized to Self-Hosted Git, Four Vendors

> Part of [CASEY-LAB](https://github.com/117caseyallen-NetAdm/casey-lab), a
> dual-site multi-vendor homelab. The hub has the topology and the
> [verification output](https://github.com/117caseyallen-NetAdm/casey-lab/blob/main/docs/verification.md).

Automated running-config backup for six network devices across **Cisco IOS
(15.0 and 12.1), Juniper Junos, Palo Alto PAN-OS, and Arista EOS**, polled
hourly by [Oxidized](https://github.com/ytti/oxidized), committed to Git only
when something changed, and pushed to a self-hosted Gitea server — unattended,
under systemd, surviving a reboot.

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
- **[docs/build-notes.md](docs/build-notes.md)** — ten stages, each ending in the
  check that proved it
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

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Placement | Management VLAN | Already permitted on every device; no ACL changes across four syntaxes |
| Install | Native gem, not Docker | Docker would hide the files this build is about |
| Device source | CSV `router.db` | NetBox is the intended source; CSV until it exists |
| Output | Local bare repo first, Gitea second | Prove reaching six devices before adding a second service to the failure surface |
| Push to Gitea | `exec` hook running the `git` binary | Oxidized's `githubrepo` hook uses rugged/libgit2, compiled without SSH transport; rebuilding it broke backups |
| Git server | Gitea, SQLite | ~200 MB of RAM against several GB for GitLab CE |
| Interval | Hourly | Commits happen only on change, so a short interval adds detection speed, not noise |
| Timezone | UTC on every device | Local time isn't monotonic — DST duplicates an hour and skips an hour, so timestamps stop being unique or ordered |
| Secrets | `remove_secret: true`; Gitea repo private; no device config published here | Redaction is a regex, so the repo stays private regardless |

## What this is, and isn't

Config archival with change detection, **not** a CI/CD pipeline: the device is
the source of truth and Git records what it finds, so nothing here can stop a bad
change from reaching a device. Inverting that — Batfish validating a proposed
change before deployment — is the NetDevOps item on the
[hub roadmap](https://github.com/117caseyallen-NetAdm/casey-lab#roadmap).

**Next:** a per-device read-only service account (Oxidized currently authenticates
with the fleet admin credential), a `post_store` hook to syslog for change
alerts, and NetBox as the device source.

---

*Internal RFC1918 addressing and hostnames are real. No credentials, keys, public
addresses, or device configurations appear in this repository.*
