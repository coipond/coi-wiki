# Troubleshooting

## Agent Freezes Completely Mid-Task

**Symptom:** The agent stops producing output entirely - no error, no progress, just silence for minutes at a time. Often happens partway through `npm install`, a `cargo build`, running tests, or any operation that writes a lot of temporary data.

**Cause:** `/tmp` inside the container is full. Since v0.7 `/tmp` is disk-backed by default (`tmpfs_size = ""`), so this freeze only applies when you have set `[limits.disk] tmpfs_size` in your config (or are running an older version) - in that case `/tmp` is a tmpfs, a RAM-backed filesystem with a hard size cap. When it fills up the kernel returns `ENOSPC` to every process that tries to write. Most build tools (npm, cargo, pytest, TypeScript compilers, Docker) are not written to handle this gracefully - they freeze waiting for a write that never succeeds.

Common sources of large `/tmp` usage:

- `npm` / `yarn` / `pnpm` - package tarballs and unpack staging
- `cargo` - incremental compilation artefacts and linker inputs
- Test runners - coverage reports, snapshots, JUnit XML output
- `tsc` / Babel - intermediate `.js` files and source maps
- Docker builds - layer staging and build context tarballs
- `sort`, `grep`, `awk` on large datasets - write temp files to `/tmp` by default

**Why Linux does not auto-clean it:** Ubuntu's `systemd-tmpfiles-clean.timer` runs daily and only removes files older than 10 days. Nothing in a normal session ever ages out. There is no back-pressure mechanism - the kernel does not evict files when space runs low the way it pages out memory.

**Diagnosis:** From your host while the agent is running:

```bash
# Check how full /tmp is
incus exec <container-name> -- df -h /tmp

# Find the largest files/directories
incus exec <container-name> -- du -sh /tmp/* 2>/dev/null | sort -rh | head -20
```

**Immediate fix (without restarting):** Clear known large cache directories:

```bash
incus exec <container-name> -- find /tmp -maxdepth 1 -name 'npm-*' -exec rm -rf {} +
incus exec <container-name> -- find /tmp -maxdepth 1 -size +100M -delete
```

**Permanent fix:** Increase the `/tmp` size cap in your config (takes effect on next session start):

```toml
# ~/.coi/config.toml  or  .coi/config.toml in your project
[limits.disk]
tmpfs_size = "8GiB"   # default is "" (disk-backed, no size cap)
```

**Alternative:** Move `TMPDIR` onto disk instead of the RAM-backed tmpfs, avoiding the size limit entirely at the cost of slightly slower temp IO:

```bash
# Add to your project's .claude.json or run at the start of the session
export TMPDIR=/workspace/.tmp
mkdir -p "$TMPDIR"
```

Note: `/workspace/.tmp` is on your host filesystem, so files persist after the session - add it to `.gitignore` and clean it up periodically.

**Coi's built-in protection (since v0.7.0):**

- `/tmp` defaults to the container's root virtual disk - no RAM cap, shares the storage pool
- `/etc/tmpfiles.d/coi-tmp-cleanup.conf` in the base image removes files not accessed for 1 hour
- `systemd-tmpfiles-clean.timer` overridden to run every 15 minutes so abandoned artefacts are reclaimed automatically
- Optional RAM-backed tmpfs available by setting `tmpfs_size = "4GiB"` in `[limits.disk]`

## DNS Issues During Build

**Symptom:** `coi build` hangs at "Still waiting for network..." even though the container has an IP address.

**Cause:** On Ubuntu systems with systemd-resolved, containers may receive `127.0.0.53` as their DNS server via DHCP. This is the host's stub resolver which only works on the host, not inside containers.

**Automatic Fix:** Coi automatically detects and fixes this issue during build by:

1. Detecting if DNS resolution fails but IP connectivity works
2. Injecting public DNS servers (8.8.8.8, 8.8.4.4, 1.1.1.1) into the container
3. The resulting image uses static DNS configuration

**Permanent Fix:** Configure your Incus network to provide proper DNS to containers:

```bash
# Option 1: Enable managed DNS (recommended)
incus network set incusbr0 dns.mode managed

# Option 2: Use public DNS servers
incus network set incusbr0 raw.dnsmasq "dhcp-option=6,8.8.8.8,8.8.4.4"
```

After applying either fix, future containers have working DNS automatically.

**Note:** The automatic fix only affects the built image. Other Incus containers on your system may still experience DNS issues until you apply the permanent fix.

**Why does Coi not automatically run `incus network set` for me?**

Coi deliberately uses an in-container fix rather than modifying your Incus network configuration:

1. **System-level impact** - Changing Incus network settings affects all containers on that bridge, not just Coi containers
2. **Network name varies** - The bridge might not be named `incusbr0` on all systems
3. **Permissions** - Users running `coi build` might not have permission to modify Incus network settings
4. **Intentional configurations** - Some users have custom DNS configurations for their other containers
5. **Principle of least surprise** - Modifying system-level Incus config without explicit consent could break other setups

The in-container approach is self-contained and only affects Coi images, leaving your Incus configuration untouched.

## Container Paused by Security Monitoring

**Symptom:** Your session suddenly freezes and `coi list` shows the container in "Frozen" state.

**Cause:** The security monitoring daemon detected a HIGH-severity threat and automatically paused the container to prevent potential data exfiltration or malicious activity.

**Common triggers:**

- Large file read operations (>50MB) - could be legitimate if analyzing big files
- Large file write operations (>50MB) - could be legitimate if creating archives
- Suspicious process patterns that match known attack tools

**Diagnosis:**

```bash
# Check the audit log for what triggered the pause
cat ~/.coi/audit/<container-name>.jsonl

# Look for HIGH-level threats
cat ~/.coi/audit/<container-name>.jsonl | grep '"level":"high"'
```

**Resolution:**

If the activity was legitimate (e.g., you asked the AI to analyze a large codebase):

```bash
# Unfreeze the specific container
coi unfreeze <container-name>

# Or unfreeze all frozen Coi containers
coi unfreeze
```

If you are unsure, review the audit log first. Look at the `title`, `category`, and `evidence` fields to understand what was detected.

**Prevention:**

For projects that legitimately need large file operations, increase the thresholds:

```toml
# ~/.coi/config.toml or .coi/config.toml
[monitoring]
file_read_threshold_mb = 200    # Increase from default 50MB
```

**Note:** Only increase thresholds if you understand the security implications. The defaults are set to catch most exfiltration attempts while allowing normal development work.

## Container Killed by Security Monitoring

**Symptom:** Your session terminates unexpectedly and `coi list` shows no container for your workspace.

**Cause:** The security monitoring daemon detected a CRITICAL-severity threat and automatically killed the container to prevent malicious activity.

**Common triggers:**

- Reverse shell patterns detected (an interactive `bash -i`, a shell opening a `/dev/tcp/` connection to another machine, `nc` with suspicious flags)
- Metadata endpoint access (169.254.169.254) - cloud credential theft attempt
- Connection to known attack ports (4444, 5555, 31337)

**Diagnosis:**

```bash
# Check the audit log (persists after container is killed)
cat ~/.coi/audit/<container-name>.jsonl

# Look for CRITICAL-level threats
cat ~/.coi/audit/<container-name>.jsonl | grep '"level":"critical"'
```

**Resolution:**

CRITICAL threats are serious and typically indicate:

1. A prompt injection attack tricked the AI into malicious behavior
2. Malicious code in the project attempted to run
3. A false positive (rare, but possible)

Before starting a new session:

1. Review what the AI was doing when killed
2. Check if the project contains suspicious code
3. If it was a false positive, report it as an issue

**Interpreter one-liners (`python -c`, `perl -e`, `ruby -e`, `php -r`):** these
are flagged only when the command also carries a real network indicator (a
`socket`/`tcp`/`udp` keyword, an IP, or a `host:port` endpoint) — a benign
`python3 -c "print(2+2)"` will not kill the container. If your workflow
legitimately runs interpreter one-liners that reach the network (and you already
enforce a network allowlist), you can downgrade just this class instead of
disabling auto-kill:

```toml
[monitoring]
auto_kill_on_critical = true
reverse_shell_one_liners = "warn"   # audit interpreter one-liners; "off" to ignore them
```

The unambiguous patterns (`nc -e`, `socat EXEC:`, `/dev/tcp/`, `bash -i`, ...)
always stay CRITICAL. See [Security Monitoring → Interpreter one-liner policy](Security-Monitoring#interpreter-one-liner-policy-reverse_shell_one_liners).

**Note:** Unlike HIGH threats (pause), CRITICAL threats (kill) require starting a new session. This is intentional - the container state may be compromised.

## Docker Compose Fails Inside Session Containers

**Symptom:** `docker compose up` or `docker-compose` commands fail inside containers started with `coi shell`. Docker itself may work fine, but Compose specifically errors out.

**Cause:** Docker support flags (`security.nesting`, `security.syscalls.intercept.mknod`, `security.syscalls.intercept.setxattr`) were not being set on session containers. This was a race condition where `incus launch` started the container before the configuration was applied.

**Fix:** This was fixed in Coi. The container launch now uses a three-step sequence: `incus init` → configure flags → `incus start`, ensuring Docker support flags are always applied before the container starts. Update to the latest version of Coi.

## "missing or unsuitable terminal" Under kitty (and foot/rio/contour/st)

**Symptom:** `coi shell` fails immediately with a "missing or unsuitable terminal" error in a terminal that exports a non-standard `TERM` — kitty (`TERM=xterm-kitty`), plus `foot`, `rio`, `contour`, and `st-*` terminals.

**Cause:** These terminals advertise a `TERM` value that is not present in the container's terminfo database, so tmux refused to start the session.

**Fix:** Handled automatically now — Coi maps these `TERM` values to `xterm-256color` inside the container. Update to the latest version and run `coi shell` normally. The old workaround `env TERM=xterm-256color coi shell` is no longer needed.

## "Permission denied" or "I have no name!" in Container

**Symptom:** You see `I have no name!` as your shell prompt, or get `Permission denied` errors when accessing files like `.bashrc` inside the container.

**Cause:** The container's `code` user has a default UID/GID of 1000 baked into the base image, but your `code_uid` config is set to a different value. The mismatch means files owned by UID 1000 are inaccessible to the new UID.

**Fix:** Coi now automatically remaps the container user's UID/GID when `code_uid` differs from the image default, running `groupmod`, `usermod`, and `chown` during session setup. Update to the latest version.

If you need to fix this manually for a persistent container:

```bash
incus exec <container-name> -- usermod -u <your-uid> code
incus exec <container-name> -- groupmod -g <your-uid> code
incus exec <container-name> -- chown -R <your-uid>:<your-uid> /home/code
```

## Security Settings Silently Disabled

**Symptom:** Security features like `block_private_networks`, `auto_pause_on_high`, or `auto_kill_on_critical` appear to be disabled even though you have not explicitly turned them off.

**Cause:** In older versions, the multi-layer config merge (global → project → CLI) used plain `bool` fields. When a higher-priority config file omitted a boolean field, it defaulted to `false` and overwrote the `true` value from a lower-priority config. This meant security-critical defaults could be silently lost.

**Fix:** This was fixed by converting 13 boolean config fields to pointer types (`*bool`), so omitted fields are `nil` (no override) rather than `false`. Update to the latest version.

**Verification:** After updating, check that your security settings are applied:

```bash
coi health --verbose
```

Look for the Monitoring section to confirm `auto_pause=true` and other security settings.

## Firewall Rules Accumulating (Thousands of Rules)

**Symptom:** `sudo nft list ruleset` shows hundreds or thousands of stale rules. System may slow down processing a bloated ruleset.

**Cause:** This could happen when containers were killed via signals, `coi shutdown` was used without proper cleanup, or containers crashed. Older versions had paths where deferred firewall cleanup was skipped.

**Fix:** Coi now cleans up firewall rules on all termination paths (normal exit, shutdown, kill, security responder auto-kill). To clean up existing stale rules:

```bash
# Dry run - see what would be cleaned
coi clean --orphans --dry-run

# Clean orphaned veth interfaces and stale nft rules
coi clean --orphans
```

**Prevention:** Always stop containers via `coi shutdown` or `coi kill` rather than directly via `incus stop/delete`.

## Settings.json Overwritten (Lost API Credentials)

**Symptom:** Your `~/.claude/settings.json` is overwritten with sandbox/bypass settings after running `coi shell`. Custom settings like AWS Bedrock credentials, environment variables, or personal preferences are lost.

**Cause:** Older versions would overwrite the entire settings file with sandbox permissions rather than merging.

**Fix:** Coi now performs a deep merge of settings, preserving your existing configuration while adding sandbox permissions. Your `env` variables, `allowedTools`, and other custom settings are preserved.

If your settings were already lost, restore from backup or recreate them. Going forward, updates should preserve your configuration.

## Sandbox Context Duplicated in ~/.claude/CLAUDE.md

**Symptom:** `~/.claude/CLAUDE.md` inside a persistent container keeps growing — the Coi sandbox context appears multiple times — and Claude Code eventually warns the file exceeds its 40k-character limit.

**Cause:** Versions before v0.11.1 appended the sandbox context block on every session without checking for a prior copy, so a persistent container reused across many sessions accumulated one copy per session (one report hit 16 copies / 108k characters).

**Fix:** As of v0.11.1 the block is delimited by `# BEGIN COI Sandbox Context` / `# END COI Sandbox Context` markers and injection is idempotent: each session replaces the managed block with a single fresh copy while preserving your own content. Already-bloated files are healed automatically on the next session — old unmarked copies are removed. Just update Coi and start a session; no manual cleanup needed.

## Cross-Device Link Error When Saving Session

**Symptom:** Session save fails with `EXDEV` (cross-device link) error, typically when `/tmp` and the session storage directory are on different filesystems or mount points.

**Fix:** Coi now uses a recursive copy with proper symlink handling as a fallback when `os.Rename` fails with `EXDEV`. Update to the latest version.

## Coi Refuses to Start: "security.privileged=true"

**Symptom:** `coi shell`, `coi run`, or `coi build` fails immediately with an error about `security.privileged=true` being detected.

**Cause:** The default Incus profile (or the container config) has `security.privileged=true` set. Privileged containers disable all container isolation - seccomp, AppArmor, and UID mapping are all bypassed. Coi refuses to run in this configuration because it defeats the entire security model.

**Fix:**

```bash
# Remove the privileged setting from the default profile
incus profile unset default security.privileged

# Verify it's gone
incus profile get default security.privileged
# Should return empty or error (meaning it's unset - which is the safe default)
```

**Verification:**

```bash
coi health
# Should show:
#   [OK]   Privileged check  : Default profile uses unprivileged containers
#   [OK]   Security posture  : Full isolation - unprivileged containers with seccomp and AppArmor
```

**Note:** Incus containers are unprivileged by default. This setting is only present if someone explicitly set it. If you need privileged containers for other (non-Coi) workloads, use a separate Incus profile rather than changing the default.

## Kernel Version Warning on Startup

**Symptom:** `coi shell`, `coi run`, or `coi build` prints a warning on stderr about the host kernel being below 5.15.

**Cause:** Kernels older than 5.15 may lack security features (user namespaces, seccomp improvements, cgroup v2) that Incus relies on for safe container isolation. Coi warns but does not block - containers still start.

**What to do:**

- If your system is running a recent distribution (Ubuntu 22.04+, Fedora 36+, Debian 12+), you likely already have kernel >= 5.15
- If you see this warning, consider upgrading your kernel or distribution
- The warning is informational - Coi continues to work, but isolation may be weaker on very old kernels

**Verification:**

```bash
uname -r
# Should show 5.15 or higher

coi health
# Should show:
#   [OK]   Kernel version    : Kernel 6.x.x (>= 5.15)
```

**Note:** This check is skipped on macOS/darwin and on any failure to read the kernel version.

## Container Fails to Start

**Symptom:** `coi shell` / `coi run` stops with a start error, or a container that briefly looked up stops again during startup.

**What to do:** Coi stops waiting as soon as the container is no longer running and quotes the first errors from its start log (`lxc.log`) — read those first. See the full log with `incus info --show-log <container>`. Common causes are a missing mount source (a path in `[[mounts]]` or a protected path that was deleted) and an idmap/shift problem on the mounted filesystem (`coi health` checks both).

## "Waiting for another coi launch of this workspace"

Another `coi shell` / `coi run` for the same workspace is picking its slot at the same moment; launches take turns until each one's container is up, which usually takes a few seconds. If it waits much longer, the other launch is probably building the image. Ctrl-C cancels the wait.

## Profiling Slow Startup

**Symptom:** `coi run`, `coi shell`, or `coi build` takes tens of seconds to start and you want to see where the time actually goes before changing anything.

**Cause:** Almost all of a session's startup is `incus` subprocesses — typically ~95%+ of wall time — and most of that is a single `incus init` that unpacks the base image. On a `dir` storage pool that one unpack can be ~70% of a `coi run` (~5-6s per unpacked GB, every session).

**Check the storage pool first (v0.11.1):** `coi health` names each pool's driver in its storage line (`default (zfs): …`) and warns outright on `dir` pools. If it warns, the fix is recreating the pool with a copy-on-write driver (zfs/btrfs) — re-running `install.sh` sets one up — and startup cost drops to near-free cloning. On `dir` pools, image size is a per-session cost, so lean images pay off twice.

**What to do:** Set `COI_TIMING_DEBUG=1` on any command to print a wall-clock startup timeline to stderr at exit — every pipeline phase, every teardown, and each `incus` and `nft` call, nested by containment, followed by per-category totals and the slowest `incus` calls. It also shows how long coi took before its first phase (process start, config load):

```bash
COI_TIMING_DEBUG=1 coi run -- true
```

For a machine-readable dump instead of stderr output, point `COI_TIMING_DEBUG_JSON` at a file:

```bash
COI_TIMING_DEBUG_JSON=/tmp/coi-timing.json coi run -- true
```

To average across several runs, `scripts/bench-run.py` in the source tree runs `coi run -- true` N times and reports the median split:

```bash
scripts/bench-run.py -n 5
```

The instrumentation lives permanently on the hot paths but records nothing unless one of these variables is set, so it is free to leave in place when unset.

## Firewall Ruleset Grows Huge (firewalld + NetworkManager)

**Symptom:** `sudo nft list ruleset` shows tens of thousands of rules in `table inet firewalld` — one report reached 101,888 rules — even with only a few containers running.

**Cause:** NetworkManager enrolls each container's host-side `veth*` interface into firewalld's default zone. When the container is deleted the veth vanishes, but the zone registration leaks — and firewalld generates its FORWARD policy rules as the cross product of zone interfaces, so the ruleset grows with the square of leaked veths (145 dead veths ≈ 145² × 4 rule shapes ≈ 100k rules). This is a NetworkManager/firewalld interaction, not a Coi rule leak: Coi's own `ip coi` tables stay small.

**Diagnose:**

```bash
# Which table holds the bulk?
sudo nft list ruleset | awk '/^table/ {t=$0} /^\t\t/ {r[t]++} END {for (x in r) print r[x], x}' | sort -rn | head -5

# Dead veths registered in zones (interfaces listed that no longer exist)
sudo firewall-cmd --get-active-zones
```

As of v0.11.2, `coi health` detects this directly (the `firewalld_veth_bloat` check warns when dead veth registrations accumulate).

**Fix:**

```bash
# 1. FIRST verify the Incus bridge is in the trusted zone PERMANENTLY —
#    a runtime-only binding would be lost by the reload and break new
#    container traffic:
sudo firewall-cmd --permanent --zone=trusted --list-interfaces   # must include incusbr0
# if missing: sudo firewall-cmd --permanent --zone=trusted --add-interface=incusbr0

# 2. Rebuild runtime state — dead veth registrations drop out instantly:
sudo firewall-cmd --reload

# 3. Prevent recurrence — stop NM enrolling veths at all
#    (the install script sets this up automatically as of v0.11.2):
sudo tee /etc/NetworkManager/conf.d/99-coi-unmanaged.conf <<'EOF2'
[keyfile]
unmanaged-devices+=interface-name:veth*
EOF2
sudo systemctl reload NetworkManager

# 4. Sweep Coi's own orphaned rules while you're at it:
coi clean --orphans
```

Established container connections survive the reload (conntrack state is kept, and only firewalld's own table is rebuilt — the `incus`/`coi` tables are untouched). Docker re-applies its firewalld state on reload automatically; if Docker networking looks off afterward, `sudo systemctl restart docker`. Excluding `veth*` from NetworkManager is safe alongside Docker and Incus: container traffic policy lives on the bridges (`incusbr0`, `docker0`, `br-*`), which stay zone-managed.

## Host Will Not Suspend While a Container Is Running

**Symptom:** With a coi container running (typically a laptop with the lid closed), the host refuses to sleep. Suspend aborts with `Device or resource busy`, `logind` retries forever so the machine never sleeps, and on lid-open it takes ~20 s to accept input.

**Cause:** The Ubuntu base image pulls in `udisks2` (as an `fwupd` dependency). `udisksd` keeps `/proc/swaps` open in its main loop to watch for swap changes. Under Incus, `/proc/swaps` is backed by `lxcfs`, so that `poll()` is a FUSE request served by a daemon on the host. When the host suspends, the kernel freezer stops `lxcfs` while the request is in flight - it can never be answered, `udisksd` wedges in uninterruptible `D` state and refuses to freeze, and after the freezer's 20 s timeout the whole suspend is aborted.

**Fix:** The image build now disables and masks `udisks2` (it is disk / removable-media management, useless inside a sandbox), closing the deadlock. This shipped in 0.12.0 (#706).

**What to do:**

- Rebuild your base image so the mask is applied: `coi build --force`
- If you cannot rebuild yet, mask it in a running container as a stopgap:

```bash
sudo systemctl disable --now udisks2.service
sudo systemctl mask udisks2.service
```

**Verification:**

```bash
# Inside a session container - should report "masked"
systemctl is-enabled udisks2.service
```

## Still Stuck?

If none of the above solved your problem, [join us on Slack](https://slack.karafka.io) - the community is happy to help.

## See Also

- [FAQ](FAQ) - Common questions with conceptual answers
- [System Health Check](System-Health-Check) - Automated environment verification
- [Security Monitoring](Security-Monitoring) - Threat events that may look like errors
- [Network Isolation](Network-Isolation) - Network-related issues and diagnosis
