# Resource and Time Limits

Control container resource consumption and runtime with configurable limits. All limits are set via config files or profiles.

## Configuration

Add to your `~/.coi/config.toml`:

```toml
[limits.cpu]
count = "2"              # CPU cores: "2", "0-3", "0,1,3" or "" (unlimited)
allowance = "50%"        # CPU time: "50%", "25ms/100ms" or "" (unlimited)
priority = 0             # CPU priority: 0-10 (higher = more priority)

[limits.memory]
limit = "2GiB"           # Memory: "512MiB", "2GiB", "50%" or "" (unlimited)
enforce = "soft"         # Enforcement: "hard" or "soft"
swap = "true"            # Swap: "true", "false", or size like "1GiB"

[limits.disk]
read = "10MiB"           # Read rate in bytes/sec: "10MiB", "1000iops" or "" (unlimited)
write = "5MiB"           # Write rate in bytes/sec: "5MiB", "1000iops" or "" (unlimited)
max = ""                 # Combined I/O limit (overrides read/write)
priority = 0             # Disk priority: 0-10
size = ""                # Rootfs quota: "" (unlimited) or "20GiB" (needs a btrfs/zfs/lvm pool)
tmpfs_size = ""          # /tmp size: "" (disk-backed, unlimited) or "4GiB" (RAM-backed, opt-in)

[limits.runtime]
max_duration = "2h"      # Max runtime: "2h", "30m", "1h30m" or "" (unlimited)
max_processes = 0        # Max processes: 100 or 0 (unlimited)
auto_stop = true         # Auto-stop when max_duration reached
stop_graceful = true     # Graceful (true) vs force (false) stop
```

All limits are configured via config files or [profiles](Profiles). There are no CLI flags for resource limits.


## What Limits Actually Do

### CPU Limits

`count` sets how many CPU cores the container can use. `"2"` means two cores total; `"0-3"` pins to specific cores; `"0,1,3"` uses three non-contiguous cores.

`allowance` sets how much of the allocated cores the container can consume. `"50%"` means the container gets half of each assigned core's time. `"25ms/100ms"` is the equivalent expressed as a CFS quota: 25ms of CPU time per 100ms period. An allowance of `""` means no throttling.

`priority` is a relative weight (0-10) used when multiple containers compete for CPU. Higher values win more CPU time under contention. At 0, the container gets a fair share; raising it makes this container's tasks favored over lower-priority containers.

### Memory Limits

`limit` is a hard ceiling on memory use (e.g., `"2GiB"`). When the container exceeds this limit:

- With `enforce = "hard"`: the kernel OOM-kills the process that pushed it over the limit. The container keeps running but the killed process is gone.
- With `enforce = "soft"` (default): the kernel attempts to reclaim memory by swapping and evicting caches before killing anything. The container may slow down significantly under memory pressure but is less likely to have processes killed.

`swap` controls whether the container's processes can use swap space:

- `"true"`: the container inherits the host's swap configuration (processes can swap to disk)
- `"false"`: swap is disabled for the container; when memory runs out, the OOM killer fires immediately with no swap buffer
- A size like `"1GiB"`: sets an explicit swap limit, additive to `limit` (total virtual memory = limit + swap size)

Disabling swap (`"false"`) is useful for latency-sensitive workloads where you would rather a process be killed than allow it to degrade performance by swapping.

### Disk I/O Limits

`read` and `write` throttle I/O throughput. Size values set a bytes-per-second ceiling and are written without a `/s` suffix — `"10MiB"` means 10 MiB per second; `"10MiB/s"` is rejected by validation. `"1000iops"` sets an operations-per-second ceiling. SI units use lowercase k (`"100kB"`), IEC units uppercase K (`"100KiB"`). These are enforced by Linux cgroup blkio and apply to block device I/O (not tmpfs, which is memory-backed).

**Do not set a very low read rate** (as of v0.10.0): disk limits apply to the container's root disk while it boots, so a pathological ceiling like `read = "100kB"` can starve startup past the readiness window and fail the session with "container failed to become ready" — the error names the active `[limits.disk]` values when this happens. If a slow host legitimately needs a longer boot window, raise `[container] ready_timeout` (seconds, default 30) rather than removing the limit.

### Disk Size Quota

`size` (as of v0.12.0) caps the container's entire root filesystem — a hard quota on how much disk the container can consume, applied to the Incus root disk device (`size=`). A value like `"20GiB"` bounds everything under `/`, including a disk-backed `/tmp`, `/home`, build artifacts, and Docker layers.

Because Incus only enforces a root-disk quota on quota-capable storage pools — btrfs, zfs, or lvm — coi checks the container's pool driver up front and refuses to launch with a clear error if `size` is set on a pool that can't enforce it (notably the `dir` driver, which silently ignores the quota). This is deliberate: a quota you believe is active but isn't is worse than none. If you hit this, either create a btrfs/zfs/lvm storage pool (`incus storage create …`) and point coi at it via `[container] storage_pool`, or remove `size`.

**`size` is the recommended way to keep `/tmp` from exhausting the container** — because `/tmp` is disk-backed by default, bounding the rootfs bounds `/tmp` too, with no RAM cost and no behavior change.

### `/tmp` Sizing (`tmpfs_size`)

`tmpfs_size` is an opt-in knob that makes `/tmp` a RAM-backed tmpfs of the given size. An empty value (default) leaves `/tmp` on the container's root filesystem — disk-backed, effectively unlimited (or bounded by `size` above). A value like `"4GiB"` mounts a RAM-backed tmpfs of that size at `/tmp` (via a systemd `tmp.mount` unit, so it also survives container reboots).

coi does not convert `/tmp` to tmpfs on its own — set `tmpfs_size` only if you specifically want a fast RAM-backed `/tmp` and understand it draws from container memory. Large build operations can fill a RAM-backed `/tmp` and cause tool freezes; see [Troubleshooting - Agent Freezes Completely Mid-Task](Troubleshooting#agent-freezes-completely-mid-task). To bound `/tmp` without moving it to RAM, prefer `size`.

Both `size` and `tmpfs_size` apply identically whether the container is launched via `coi shell` or `coi run`.

### Runtime Limits

`max_duration` stops the container after the specified time has elapsed since session start. Combine with `auto_stop = true` (the default) to get automatic cleanup. `stop_graceful = true` sends SIGTERM and waits for a clean exit; `stop_graceful = false` force-kills immediately.

`max_processes` caps the total number of processes (threads included) the container can spawn. Useful for preventing runaway process trees. `0` means unlimited.

## Profile-Specific Limits

Define limits per profile (each profile is a directory under `profiles/` with its own `config.toml`):

```toml
# .coi/profiles/limited/config.toml
[container]
image = "coi-default"
persistent = false

[limits.cpu]
count = "2"
allowance = "50%"

[limits.memory]
limit = "2GiB"

[limits.runtime]
max_duration = "2h"
auto_stop = true
```

Use with: `coi shell --profile limited`

## Time Limits and Auto-Stop

When `max_duration` is set and `auto_stop = true`:

- Container automatically stops after the specified duration
- Graceful stop preserves session data
- Force stop (`stop_graceful = false`) terminates immediately
- Useful for preventing runaway sessions or managing costs

Example:
```toml
# Auto-stop after 2 hours
[limits.runtime]
max_duration = "2h"
auto_stop = true
```

## Precedence

Limits are applied with this precedence (highest to lowest):
1. Profile limits (if `--profile` specified)
2. Project config (`[limits]` section)
3. User/system config (`[limits]` section)
4. Unlimited (Incus defaults)

## Examples

**Limit resources for expensive operations:**
```toml
# .coi/config.toml
[limits.cpu]
count = "4"

[limits.memory]
limit = "4GiB"

[limits.runtime]
max_duration = "30m"
```

**Prevent runaway processes:**
```toml
[limits.runtime]
max_processes = 100
max_duration = "1h"

[limits.memory]
limit = "2GiB"
```

**Development profile with limits:**
```toml
# .coi/profiles/dev/config.toml
[container]
image = "coi-default"
persistent = true

[limits.cpu]
count = "2"

[limits.memory]
limit = "4GiB"

[limits.runtime]
max_duration = "4h"
```

## See Also

- [Configuration](Configuration) - Full configuration reference including all limit options
- [Profiles](Profiles) - Per-profile resource limits
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How limits interact with session lifecycle
