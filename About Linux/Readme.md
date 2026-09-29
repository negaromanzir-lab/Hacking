# About Linux

## Overview

Notes on the Linux fundamentals that matter when working on a target: how identity, permissions, processes, services, and the file system fit together, and where the usual misconfigurations live. Intended as a reference sheet to consult alongside the hands-on material in [`../tryHackmeNotes-main/`](../tryHackmeNotes-main/), not as a substitute for a full systems course.

## Repository Structure

- Identity, users, and groups
- File system and permissions
- Processes and signals
- Services, `init`, and scheduling
- Kernel, modules, and capabilities
- Package management and logging

## Identity, Users, and Groups

Linux has no built-in administrator account. Every process runs with a real user identity, and privileges come from membership in groups and from capabilities granted to specific binaries.

```bash
id                       # current uid, gid, and supplementary groups
whoami                   # effective user name
groups                   # groups the current user belongs to
getent passwd            # all local accounts, from /etc/passwd
getent group sudo        # members of the sudo group
last                     # recent login history per user
```

Password hashes live in `/etc/shadow` (mode `000`, group `shadow`). The fields of `/etc/passwd` are `name:passwd:uid:gid:gecos:home:shell`. An account with `/bin/false` or `/sbin/nologin` as its shell is a service account even if it can technically log in.

Names to recognise during enumeration: `root` (uid 0), `www-data`, `apache`, `nginx`, `mysql`, `postgres`, `git`, `deploy`, and anything with uid/gid below 1000 that looks like a real person.

## File System and Permissions

```bash
ls -la                        # long listing, including dotfiles
stat <file>                   # type, permissions, owner, size, timestamps
find / -perm -4000 -type f    # SUID executables
find / -perm -2000 -type f    # SGID executables
find / -perm -0002 -type f    # world-writable files
```

Permissions are `rwx` triples for user, group, other, encoded in octal. `chmod 4755` sets SUID, `2775` sets SGID, and the leading `4`/`2`/`7` sticky bits behave differently (`/tmp` uses `1777`).

Permission bits only matter if they grant access to something. The realistic impact of a world-readable file is entirely determined by who else can reach the host and what the file contains, so read `/etc/shadow`, backups, and config files in context rather than treating any one finding as critical.

### Common Weak Permissions

- Writable by everyone and executed by a privileged service (service reload, cron, or unit file injection).
- Sensitive files readable by the service account: `.env`, `wp-config.php`, `settings.py`, `id_rsa`, `.aws/credentials`, `.docker/config.json`, `/etc/shadow` backups.
- Overly broad directory permissions that let a user replace a binary the service later executes.

### Hard Links, Symlinks, and Namespaces

A hard link is another name for the same inode, so a second path can reach a file a service restricts by path. A symlink stores a path that is resolved on access, so a link pointing outside a web root is a common way out of a chroot-like restriction. Overlay and bind mounts change what a path resolves to, which is why `/proc`, `/sys`, and container mounts are worth reading before trusting a path.

## Processes and Signals

```bash
ps aux                       # all processes, sorted by user
ps -ef --forest              # parent/child relationships
top / htop                   # live view
kill -l                      # available signals
```

Signals: `SIGTERM` (15) asks a process to exit, `SIGKILL` (9) cannot be caught or ignored, `SIGHUP` (1) conventionally means reload configuration. `ps auxww` widens output so arguments are not truncated, which matters for spotting command-line secrets and interpreters invoked with inline code.

`/proc` is a virtual view of the kernel per process: `/proc/<pid>/cmdline`, `/proc/<pid>/environ` (own processes only), `/proc/<pid>/cwd`, and `/proc/<pid>/exe` (the real binary behind a deleted or symlinked path).

## Services, init, and Scheduling

```bash
systemctl list-units --type=service
systemctl cat <service>         # unit file contents
systemctl status <service>
journalctl -u <service> -n 100  # recent service logs
```

Reading a unit file is often the highest-value enumeration step: `ExecStart`, `User`, and `Environment` reveal what runs, as whom, and with what configuration. Config files usually live in `/etc/<service>/` with an optional `.service` override in `/etc/systemd/system/`.

Scheduling:

```bash
crontab -l                     # current user's jobs
cat /etc/crontab               # system-wide jobs
ls -la /etc/cron.*             # periodic jobs
systemctl list-timers          # systemd timers
```

Anything executing as root from a world-writable script, config, or directory deserves a close read.

## Kernel, Modules, and Capabilities

```bash
uname -a
cat /proc/version
lsmod                        # loaded kernel modules
cat /proc/modules
cat /proc/cmdline            # kernel boot arguments
dmesg | tail                 # ring buffer, may be restricted
```

Capabilities split root's authority into individual bits, so a binary holding only `CAP_NET_BIND_SERVICE` can bind port 80 without being root. Check what a binary can do with `getcap <path>` or `getcap -r /`.

Kernel hardening is visible in `/proc/sys` and `/etc/sysctl.conf`: `kernel.kptr_restrict`, `kernel.dmesg_restrict`, `kernel.yama.ptrace_scope`, `fs.protected_*`, and `net.ipv4.conf.all.accept_redirects`. These map directly to what an attacker can do with a foothold, so they are worth reading before deciding whether a finding is exploitable.

## Package Management and Logging

```bash
dpkg -l                       # Debian/Ubuntu installed packages
rpm -qa                       # RHEL family installed packages
apt list --installed
```

Package provenance matters. A binary that has no corresponding package entry was likely dropped manually, which is a much stronger signal than a version number alone.

Authentication and service logs:

```bash
/var/log/auth.log      # Debian/Ubuntu authentication
/var/log/secure         # RHEL family authentication
/var/log/syslog, /var/log/messages
/var/log/wtmp, /var/log/btmp, /var/log/lastlog
```

`tally` shows failed-login counts per account, which separates scripted guessing from targeted credential use.

## Shell and Scripting Notes

```bash
which cmd; type -a cmd; command -v cmd   # resolution order and builtins
alias                                    # what is shadowed in this session
echo "$PATH"                             # writable directories in PATH are a risk
```

Reviewing a shell script for an unquoted variable, an unvalidated path, or a command built from input is often faster and more reliable than trying to reason about a compiled binary.

## Hardening Checklist

- No unexpected accounts, and no account with a login shell that should not have one.
- `sudo` restricted with a dedicated group and a versioned rule rather than blanket `NOPASSWD: ALL`.
- No world-writable files in locations a privileged process reads or executes.
- Services run as unprivileged users with the minimum capabilities.
- `sshd` with key-based auth, no root login, and no password auth.
- Kernel hardening sysctls applied and persistent across reboot.
- Outbound-only network egress where the workload does not need inbound.

## Related Material

- [`../tryHackmeNotes-main/Linux commands/`](../tryHackmeNotes-main/Linux%20commands/)
- [`../tryHackmeNotes-main/Linux PrivEsc/`](../tryHackmeNotes-main/Linux%20PrivEsc/)
- [`../tryHackmeNotes-main/Kerberos/`](../tryHackmeNotes-main/Kerberos/)
- [`../About Networking/`](../About%20Networking/)

## Scope and Legal Notice

Everything here is for systems you own or have explicit written authorisation to test. Most of these techniques are dual-use: the same enumeration commands are used for legitimate administration and for someone deciding whether a host is worth targeting, so keep notes on authorisation scope, timestamps, and the exact reason each finding matters.

## Contribution Guidelines

1. Keep one topic per section and lead with the command, then explain what the output means.
2. Prefer a real command and its real output over a paraphrase.
3. Note where behaviour differs between Debian/Ubuntu and RHEL-family distributions.
4. Add the defensive consequence alongside the technique; a note that does not say why it matters is only half a note.
5. Do not paste real credentials, real hostnames, or real keys into notes.
