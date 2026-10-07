# Security Monitoring

Coi includes a built-in security monitoring system that actively watches AI tool behavior and responds to potential threats in real-time.

## Overview

The security monitoring daemon provides:

- **Real-time threat detection** for reverse shells, data exfiltration, and credential scanning
- **Automated response** based on threat severity (log → alert → pause → kill)
- **Persistent audit logging** in JSON Lines format
- **nftables-based network monitoring** for kernel-level packet visibility
- **Large file I/O detection** to catch data packaging/exfiltration attempts
- **Disk space monitoring** to prevent /tmp exhaustion

### Background and Detached Sessions

Monitoring (and the `max_duration` runtime limit) runs for as long as the container does — including `coi shell --background`, a detached tmux session, or leaving the agent with the container still running. A small supervisor process keeps it going after `coi shell` returns and stops when the container stops; `coi shell` prints a `[supervisor]` line saying what is covered and where it logs (`~/.coi/logs/<container>.stderr.log`).

## Enabling Security Monitoring

Security monitoring has two independent subsystems. Enable each separately:

```toml
# ~/.coi/config.toml

# Process and filesystem monitoring (CPU-only, no kernel hooks required)
[monitoring]
enabled = true

# Network monitoring via nftables (requires nftables + systemd-journal access)
[monitoring.nft]
enabled = true
```

`[monitoring] enabled = true` activates process-level and filesystem-level threat detection — it watches spawned processes and file I/O rates. It requires no additional system privileges beyond what Coi already has.

`[monitoring.nft] enabled = true` activates kernel-level network monitoring via nftables, which logs packet metadata to systemd-journal. This subsystem requires `nftables`, `libsystemd`, and the current user to be in the `systemd-journal` group (see Setup below). Run `coi health --verbose` to verify all prerequisites are met before enabling it.

You can enable either subsystem independently — process/filesystem monitoring without network monitoring, or vice versa.

## Threat Detection

The monitoring system detects multiple threat categories:

### Process-Level Threats

| Threat | Detection Method | Severity | Response |
|--------|-----------------|----------|----------|
| Reverse shells (unambiguous) | Pattern matching on process commands (`nc -e`, interactive shells such as `bash -i`, a shell opening a `/dev/tcp/` connection to another machine, `socat EXEC:`, `socket.socket`, `fsockopen`, ...) | CRITICAL | Kill |
| Reverse shells (interpreter one-liners) | `python -c` / `python3 -c` / `perl -e` / `ruby -e` / `php -r` **combined with a real network indicator** (a socket/tcp/udp keyword, an IP address, or a `host:port` endpoint). Severity is configurable — see [Interpreter one-liner policy](#interpreter-one-liner-policy-reverse_shell_one_liners) | CRITICAL (default) | Kill |
| Environment scanning | Detecting reads of /proc/*/environ, credential files, language-specific env access patterns | WARNING | Alert |
| Large file reads | File read rate exceeds threshold (default 50MB) | HIGH | Pause |
| Large file writes | File write rate exceeds threshold (mirrors read threshold) | HIGH | Pause |
| Suspicious processes | Known attack tool patterns (socat, ncat, etc.) | HIGH | Alert/Pause |

### Filesystem Threats

| Threat | Detection Method | Severity | Response |
|--------|-----------------|----------|----------|
| Data exfiltration (read) | Reading >50MB of data (configurable) | HIGH | Pause |
| Data exfiltration (write) | Writing >50MB of data (tar, dd, etc.) | HIGH | Pause |
| /tmp exhaustion | /tmp usage exceeds 80% | WARNING | Alert |

### Network-Level Threats (nftables)

| Threat | Detection Method | Severity | Response |
|--------|-----------------|----------|----------|
| Private network access | Connections to RFC1918 addresses (10.x, 172.16.x, 192.168.x) | WARNING | Alert |
| Metadata endpoint | Access to 169.254.169.254 (cloud metadata service) | CRITICAL | Kill |
| Suspicious ports | Connections to common attack ports (4444, 5555, 31337) | HIGH | Alert |
| DNS anomalies | Unusual DNS query patterns | INFO | Log |

**Note:** The Incus bridge gateway IP is automatically excluded from RFC1918 private network checks. This prevents false-positive HIGH alerts on routine DNS/NTP traffic routed through the gateway, which could otherwise incorrectly pause or freeze the container.

## Threat Levels

| Level | Description | Default Action |
|-------|-------------|----------------|
| INFO | Normal activity, logged for audit | Log only |
| WARNING | Suspicious but not necessarily malicious | Alert user |
| HIGH | Likely malicious, requires attention | Pause container (if `auto_pause_on_high` enabled) |
| CRITICAL | Confirmed malicious activity | Kill container (if `auto_kill_on_critical` enabled) |

## Automated Response

The system responds based on threat severity:

1. **Log**: Record the event for later review
2. **Alert**: Display warning to user in real-time
3. **Pause**: Suspend the container (user can resume or kill)
4. **Kill**: Terminate the container immediately

### Unfreezing Paused Containers

When a container is paused due to a HIGH threat, you can investigate and unfreeze:

```bash
# List frozen containers
coi list

# Unfreeze a specific container
coi unfreeze <container-name>

# Unfreeze all frozen Coi containers
coi unfreeze
```

**Note:** Only containers in the Frozen state can be unfrozen. Review the audit log before unfreezing to understand what triggered the pause.

### Preserving a Killed Container for Forensics (`forensics_on_kill`)

An auto-kill fires exactly when the container's state is most worth
investigating — but by default the killed (ephemeral) container is deleted, so
only the audit log survives. Set `forensics_on_kill = true` to keep the
evidence:

```toml
[monitoring]
auto_kill_on_critical = true
forensics_on_kill = true    # opt-in; default false
```

When enabled, the responder copies the still-running container to a stopped,
non-ephemeral `<container>-forensics-<timestamp>` before the kill, so the
on-disk state at the moment of the threat survives the response ("snapshot
state for investigation before deactivating", per Trail of Bits'
[VMs won't contain cyber-capable agents](https://blog.trailofbits.com/2026/08/26/vms-wont-contain-cyber-capable-agents/)).
Inspect or revive it with ordinary Incus commands, and dispose when done:

```bash
incus list <container>-forensics-*        # find the preserved copy
incus file pull <name>/path/to/file ./    # pull artifacts out
incus start <name> && incus exec <name> -- sh   # or boot it read/inspect
incus delete --force <name>               # dispose when finished
```

At most 3 copies are kept per container (oldest pruned) so repeated incidents
cannot fill the pool; on a btrfs/zfs pool each copy is a near-instant COW
reflink. It is off by default because preserving a container on every kill
would otherwise accumulate stopped containers — enable it deliberately when you
want post-incident forensics.

### Interpreter one-liner policy (`reverse_shell_one_liners`)

Reverse-shell detection splits interpreter patterns into two classes:

- **Unambiguous** — `nc -e`, `socat`, `EXEC:`, `/dev/tcp/`, `/dev/udp/`, an
  interactive shell (`bash -i` / `sh -i`), and socket keywords like
  `socket.socket` / `fsockopen`. These are never benign and always fire at
  CRITICAL, regardless of the setting below.
- **Interpreter one-liners** — `python -c`, `python3 -c`, `perl -e`, `ruby -e`,
  `php -r`. A coding agent runs these constantly for legitimate work, so they are
  flagged **only when the command also carries a real network indicator** (a
  `socket`/`tcp`/`udp` keyword, an IP address, or a `host:port` endpoint — local
  addresses such as `127.0.0.1` or `localhost` don't count). A bare
  `python3 -c "print(2+2)"` — or any one-liner whose text merely contains a colon
  (a `PATH`, a dict literal like `{"k": v}`, a URL, a timestamp) — is **not** a
  threat and is left alone.

Everyday agent commands that merely *resemble* these patterns are not flagged,
for example:

- installing or mentioning tools (`apt-get install socat`, `rg -i powershell docs/`);
- searching code or docs for the patterns themselves (`grep -rn /dev/tcp/ docs`,
  `rg "exec:"`), or `/dev/tcp` text inside another program's arguments
  (`git commit -m`, `sed`);
- waiting for a local service, e.g.
  `until (echo > /dev/tcp/localhost/5432) 2>/dev/null; do sleep 1; done`;
- `rsync -e ssh`, `ssh -i key host`, `./setup.sh -i`, `perl -MIO::File`;
- writing a Kubernetes manifest whose probes contain `exec:` and `tcpSocket:`.

`reverse_shell_one_liners` controls the severity of the one-liner class:

| Value | Behavior |
|-------|----------|
| `"critical"` (default) | CRITICAL — auto-kills when `auto_kill_on_critical` is on |
| `"warn"` | WARNING — logged and audited, never kills or pauses |
| `"off"` | Not reported at all |

```toml
[monitoring]
auto_kill_on_critical = true
reverse_shell_one_liners = "warn"   # audit interpreter one-liners instead of killing
```

This is useful when you already enforce a network allowlist (`[network] mode =
"allowlist"`): a reverse shell then has nowhere to connect, so you can downgrade
the noisy one-liner class to `warn` while keeping the unambiguous patterns at
CRITICAL — rather than turning `auto_kill_on_critical` off entirely. A downgrade
never affects the unambiguous class: a one-liner that *also* contains an
unambiguous indicator (e.g. `python3 -c '...socket.socket()...'`) stays CRITICAL.

> Historically the detector treated a bare `:` in a command as a "network
> indicator," which made agent-run `python -c` / `perl -e` / `ruby -e` / `php -r`
> commands kill-on-sight in headless runs (they nearly always contain a colon via
> `PATH`, dict literals, or URLs). That is fixed: a colon alone is no longer an
> indicator.

## Configuration

Full configuration options for `~/.coi/config.toml`:

```toml
[monitoring]
enabled = true                    # Enable security monitoring
poll_interval_sec = 2             # How often to check (seconds)
auto_pause_on_high = true         # Pause container on HIGH threats
auto_kill_on_critical = true      # Kill container on CRITICAL threats
forensics_on_kill = false         # Keep a *-forensics-* copy before an auto-kill (opt-in)
reverse_shell_one_liners = "critical"  # python -c/perl -e/ruby -e/php -r class: "critical" | "warn" | "off"

# File I/O thresholds
file_read_threshold_mb = 50       # Alert if >50MB read in one cycle
file_read_rate_mb_per_sec = 10    # Alert if reading >10MB/sec sustained

[monitoring.nft]
enabled = true                    # Enable nftables network monitoring
rate_limit_per_second = 100       # Normal traffic: 100 packets/second logged
dns_query_threshold = 100         # Alert if >N DNS queries/min
log_dns_queries = true            # Separate DNS logging
```

### Threat Deduplication

The monitoring system includes automatic threat deduplication with a 30-second window. This prevents alert spam when the same threat pattern is detected repeatedly (e.g., a script that continuously scans environment variables).

## Audit Logs & `coi audit`

All security events are written to a structured JSON Lines audit log
(`~/.coi/audit/<container-name>.jsonl`) and can be streamed live with `coi audit`.
This is the forensic record of what the monitor detected — it persists after the
container is gone and is queryable with `jq`/`grep`.

➡️ See the dedicated [Audit Log](Audit-Log) page for the on-disk format, the full
field reference, and the `coi audit` command (dump/follow modes, event sources,
filtering, tuning).

## nftables Network Monitoring

For complete network visibility, Coi uses nftables kernel-level packet filtering:

### Setup

```bash
# Install dependencies
sudo apt install nftables libsystemd-dev

# Add user to systemd-journal group
sudo usermod -aG systemd-journal $USER

# Allow passwordless nft commands (writes a visudo-validated /etc/sudoers.d/coi-nft)
coi health --fix

# Verify setup
coi health --verbose
```

Or use the provided setup script:

```bash
./scripts/install-nft-deps.sh
```

### How It Works

1. Coi injects nftables rules that log packet metadata to systemd-journal
2. The monitoring daemon streams logs from journald in real-time
3. Suspicious patterns trigger alerts based on destination IPs and ports
4. All events are recorded to the audit log
5. Rules are automatically cleaned up when containers are killed or stopped

### Environment Scanning Detection

The environment scanning detector catches a wide range of techniques used to harvest secrets:

- **Shell commands:** `env`, `printenv`, `set`, `export`
- **Direct `/proc` access:** `grep`, `cat`, `strings`, `xxd`, `hexdump`, or `xargs` reading `/proc/*/environ`
- **Language-specific patterns:**
  - Python: `os.environ`, `os.getenv`
  - Node.js: `process.env`
  - Ruby: `ENV[]`
  - awk: `ENVIRON[]`
- **Text search for secrets:** `grep`, `sed`, or `awk` with keywords like `api`, `key`, `password`, `secret`, `token`, `credential`, `auth`

### Dropped Event Tracking

Under heavy network traffic, the event channel may fill up. The NFT monitor tracks dropped events atomically and reports via the `OnError` callback on the 1st drop and every 100th subsequent drop. The event channel buffer is sized at 1000 events to absorb traffic bursts.

### Orphan Rule Cleanup

If containers are killed without proper cleanup (e.g., force-kill, crash), nftables rules can accumulate. Coi handles this automatically:

- `coi clean --orphans` detects and removes orphaned nftables rules and chains
- Chain existence is verified before rule operations to prevent errors
- Rules are cleaned up on all termination paths: normal exit, `coi shutdown`, `coi kill`, and security responder auto-kill

### Configuration

```toml
[monitoring.nft]
enabled = true                    # Enable nftables monitoring
rate_limit_per_second = 100       # Normal traffic: 100 packets/second logged
dns_query_threshold = 100         # Alert if >N DNS queries/min
log_dns_queries = true            # Separate DNS logging
```

## Health Checks

The `coi health` command verifies monitoring prerequisites:

```text
MONITORING:
  [OK]   nftables           Available and configured
  [OK]   systemd journal    Access granted (systemd-journal group)
  [OK]   libsystemd         Development library installed
  [OK]   Monitoring config  Enabled with auto_pause=true
  [OK]   Audit log dir      ~/.coi/audit (writable)
  [OK]   Cgroup availability Cgroups v2 available
```

## Best Practices

1. **Always enable monitoring for untrusted projects** - Set `[monitoring] enabled = true` in config when working with unfamiliar codebases

2. **Review audit logs after sessions** - Check `~/.coi/audit/<container-name>.jsonl` for any suspicious activity

3. **Configure appropriate thresholds** - Adjust `file_read_threshold_mb` based on your project's normal behavior (large codebases may need higher thresholds)

4. **Use network isolation together** - Combine monitoring with [Network Isolation](Network-Isolation) for defense-in-depth

5. **Do not disable auto_pause without good reason** - It is your last line of defense against active threats

6. **Investigate before unfreezing** - When a container is paused, check `~/.coi/audit/<container-name>.jsonl` before using `coi unfreeze`

## Example: Detecting Data Exfiltration

The monitoring system can detect both read-based and write-based exfiltration attempts:

**Read-based exfiltration** (agent reads sensitive files):
```bash
# Agent tries to read all source files
find . -name "*.py" -exec cat {} \;
# → Triggers HIGH threat if total read exceeds threshold
```

**Write-based exfiltration** (agent packages data for transfer):
```bash
# Agent creates archive for exfiltration
tar -czf /tmp/data.tar.gz /workspace
# → Triggers HIGH threat when write exceeds threshold
```

Both patterns are detected and can automatically pause the container before the data leaves.

## Limitations

- **Process monitoring requires cgroups v2** - Most modern Linux distributions use cgroupsv2 by default
- **nftables monitoring requires root (via sudo)** - Passwordless sudo is configured for specific nft commands only
- **macOS/Colima** - nftables monitoring is not available; process monitoring works via limactl wrapper
- **Disk space monitoring** - Requires /tmp to be a separate filesystem (tmpfs) to detect usage percentage
- **NFT monitoring errors route through OnError callback** - Errors are written to `~/.coi/logs/<container>.stderr.log` to avoid corrupting the TUI; review them with `coi logs <container>`

## Troubleshooting

### "nftables not available"

```bash
sudo apt install nftables
sudo systemctl enable --now nftables
```

### "systemd-journal access denied"

```bash
sudo usermod -aG systemd-journal $USER
# Log out and back in
```

### "Monitoring config disabled"

Add to `~/.coi/config.toml`:

```toml
[monitoring]
enabled = true
```

See the configuration section above for how to enable monitoring.

### Container paused unexpectedly

1. Check the audit log:
   ```bash
   cat ~/.coi/audit/<name>.jsonl
   ```

2. Look for HIGH-level threats that triggered the pause

3. If it is a false positive (e.g., legitimate large file operation), you can:

   - Increase the threshold in config
   - Unfreeze the container: `coi unfreeze <name>`

### NFT rules not cleaned up

NFT monitoring rules are automatically cleaned up when containers are:

- Killed via `coi kill`
- Stopped via `coi shutdown`
- Auto-killed by the security responder

If rules persist after container deletion, run:
```bash
coi clean --orphans
```

## See Also

- [nftables Monitoring Internals](nftables-Monitoring-Internals) - Technical deep-dive: LOG-rule layout, kernel log format, detection pipeline, and threat model
- [Audit Log](Audit-Log) - On-disk audit log format, field reference, and the `coi audit` command
- [Session Logs](Session-Logs) - Coi operational logs (`coi logs`)
- [Security Best Practices](Security-Best-Practices) - Recommended configuration for different threat levels
- [Network Isolation](Network-Isolation) - Network-layer threat filtering
- [Troubleshooting](Troubleshooting) - Diagnosing false positives and monitoring issues
- [Configuration](Configuration) - Enabling and tuning the monitoring system
