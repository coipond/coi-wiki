# Resource Usage (`coi top`)

`coi top` shows live CPU, memory, disk I/O, and network I/O for your running
Coi containers — resolved to each container's friendly context (alias +
workspace) so you can tell which container, or which process inside one, is
loading your machine without mapping PIDs by hand.

It reuses the same cgroup/`/proc` collectors as the security monitor, so the
numbers match what `coi monitor` and the auto-response thresholds see.

## Usage

```bash
# One row per container: CPU%, memory, disk I/O, net I/O — busiest first
coi top

# Drill into one container: one row per process inside it
coi top <name|alias>

# Every container's processes at once (adds a CONTAINER column)
coi top --procs
```

CPU%, disk I/O, and network I/O are rates, sampled over a short interval
(`--interval`, default 2s), so the command pauses briefly before printing. CPU%
is aggregate across host cores (`top` convention): a container pegging two cores
reads ~200%.

## Options

| Flag | Description |
|------|-------------|
| `-i, --interval <sec>` | Seconds to sample CPU/IO rates over (default `2`) |
| `--sort <key>` | Sort key: `cpu`, `mem`, `disk`, or `net` for containers; `cpu` or `mem` for processes (default `cpu`) |
| `--procs` | Show processes (across **all** containers when no container is named) |
| `--watch <N>` | Re-render every N seconds until `Ctrl+C` (0 = one-shot, the default) |
| `--json` | Machine-readable JSON output |

## Examples

```bash
coi top                  # all containers, busiest first
coi top --sort mem       # sort by memory instead of CPU
coi top -i 5             # sample over 5 seconds (steadier rates)
coi top my-api           # processes inside the 'my-api' container
coi top --procs          # every container's processes, busiest first
coi top --watch 2        # live dashboard, re-render every 2s until Ctrl+C
coi top --json           # for scripting / dashboards
```

## Killing a Runaway

Process rows show the host-side PID, not the in-container PID, so a runaway
is directly actionable from the host:

```bash
coi top my-api           # find the offending process + its host PID
sudo kill <PID>          # kill it on the host
```

## How It Reads the Numbers

- The per-container row reads the container's top-level cgroup, so its
  memory/CPU/I/O aggregate the whole process tree — not just the container's
  `init` process — matching what the per-process view sums.
- On Incus/LXC layouts that split an instance into separate monitor and
  payload cgroups, the collectors probe the payload cgroup (the container's
  process tree) before the monitor cgroup (the host-side forkstart), so stats and
  init-PID resolution target the right one there too.

## `coi top` vs `coi monitor`

- **`coi top`** is a general resource viewer — CPU/mem/disk/net for any running
  container, for spotting load. It takes no security action.
- **`coi monitor`** ([Security Monitoring](Security-Monitoring)) is the threat
  detection engine — it watches for suspicious activity and can auto-pause/kill a
  container. Its `--watch` mode renders a similar live view but is scoped to the
  monitored session.

## See Also

- [Resource and Time Limits](Resource-and-Time-Limits) - Cap what a container may consume
- [Security Monitoring](Security-Monitoring) - Real-time threat detection and automated response
- [Container Operations](Container-Operations) - Manage containers and sessions
- [System Health Check](System-Health-Check) - Diagnose your Coi/Incus setup
