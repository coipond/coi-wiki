# Session Logs

Coi records operational messages from each shell session to log files on the host. The `coi logs` command reads those files, making it easy to review what happened in a session after it ends or to tail output in real time.

> **Note:** Session logs contain Coi's own setup and housekeeping messages — things like "Setting up claude config…" or network-refresh notices. They are distinct from security audit events, which are written to `~/.coi/audit/<container>.jsonl` and accessed via [`coi audit`](Audit-Log).

## Log File Locations

| File | Contents |
|------|----------|
| `~/.coi/logs/<container>.stdout.log` | Informational messages (setup steps, status updates) |
| `~/.coi/logs/<container>.stderr.log` | Warnings and errors |

Both files are appended to across resume cycles, so a single container name accumulates logs from all of its sessions.

## Usage

```bash
# Print logs for the container associated with the current workspace
coi logs

# Print logs for a specific container
coi logs coi-abc123-1

# Tail logs in real time (both files, Ctrl+C to stop)
coi logs coi-abc123-1 --follow
coi logs coi-abc123-1 -f
```

### Auto-Detection

When no container name is given, `coi logs` resolves the container the same way `coi monitor` does — from the current workspace directory. If multiple containers are running, pass the container name explicitly.

### Output Format

Stdout log lines are printed as-is. Stderr log lines are prefixed with `[ERR]` so they are visually distinct even when both streams are interleaved:

```text
2026/05/22 10:15:01 [setup] Setting up claude config...
2026/05/22 10:15:01 [setup] Container setup complete!
[ERR] 2026/05/22 10:15:03 [network] allowlist refresh failed: dial tcp: lookup example.com: no such host
```

### Follow Mode

With `--follow`, `coi logs` seeks to the end of both log files and polls for new content every 200 ms until you press Ctrl+C. If a log file does not exist yet (e.g. the session is still starting), `coi logs --follow` retries until it appears.

```bash
# In one terminal: start a session
coi shell

# In another terminal: watch its logs live
coi logs --follow
```

## What Goes Into Session Logs

All background subsystem output is routed to the session log files:

| Subsystem | Where it goes |
|-----------|--------------|
| Network manager (allowlist IP refresh, TTL resolution) | `<container>.stdout.log` |
| Resource/time limit notifications | `<container>.stdout.log` |
| Security monitoring errors (`OnError` callbacks) | `<container>.stderr.log` |
| Security threat detections (`OnThreat` callbacks) | `<container>.stdout.log` |

> **Note:** The old `~/.coi/logs/network-refresh-<container>.log` file is no longer created. All background goroutine output goes to `<container>.stdout.log` instead.

## See Also

- [Audit Log](Audit-Log) — threat-event audit logs and `coi audit`
- [Security Monitoring](Security-Monitoring) — the detection engine behind those events
- [Network Isolation](Network-Isolation) — allowlist mode and the refresh background task
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) — session resume and cleanup
