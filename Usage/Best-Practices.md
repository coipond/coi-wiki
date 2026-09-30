# Best Practices

Recommended patterns for getting the most out of Coi safely and efficiently, drawn from common workflows and operational experience.

## Choosing a Session Mode

### Ephemeral vs. Persistent

| Use case | Recommended mode |
|----------|-----------------|
| Code review, refactoring, writing tests | Ephemeral (default) |
| Installing language runtimes or build tools | Persistent (`[container] persistent = true`) |
| One session shared by several checkouts, or a workspace you plan to move | Persistent + `[container] session_name` ([Named Sessions](Container-Lifecycle-and-Sessions#named-sessions-session_name--v0111)) |
| Long-running overnight tasks | Persistent |
| Untrusted or unfamiliar codebase | Ephemeral — malicious container state is discarded on exit |
| Parallel feature work | Ephemeral multi-slot (each slot isolated, same workspace) |

**Default to ephemeral.** Ephemeral sessions are cleaner: any tool the AI installs, any temp file it creates, any system state it modifies is gone when the session ends. The conversation history and your workspace files are always preserved regardless of mode.

Switch to persistent only when you have invested setup time in the container (installing compiled tools, building caches) that would be expensive to repeat.

### Snapshots vs. Resume

Use `coi snapshot create` when:

- You are about to let the AI make sweeping changes and want a rollback point
- You want to branch — try two different approaches from the same starting state
- You have spent significant time building up container state (compiled deps, mise installs) and do not want to lose it

Use `coi shell --resume` when:

- You want to continue an interrupted conversation
- The container state does not matter — only the AI context does

Snapshots persist the full container disk state. `--resume` restores only the AI tool's conversation history in a fresh container.

## Network Mode Selection

Match the network mode to your threat model:

| Project type | Recommended mode | Rationale |
|-------------|-----------------|-----------|
| Your own trusted project | `restricted` (default) | Internet for packages; no internal network access |
| Third-party or unfamiliar repo | `allowlist` | Constrain egress to only the services the project needs |
| Internal tooling that needs local services | `restricted` + `allow_local_network_access = true` | Only if you genuinely need cross-machine access |
| Fully offline work or testing | `open` with monitoring | No egress filtering; rely on monitoring instead |

When in doubt, start with `restricted`. If package managers fail, add the required registries to an allowlist config rather than switching to open.

## Security Monitoring

- **Enable monitoring for any project you did not write** — Set `[monitoring] enabled = true` in your global `~/.coi/config.toml` so it applies everywhere, then disable per-project only if you have a specific reason.
- **Review the audit log after suspicious sessions** — `cat ~/.coi/audit/<container-name>.jsonl` before deleting the container. Once the container is gone, the audit log is the only record.
- **Tune thresholds for large codebases** — Projects with many large files (ML datasets, compiled artifacts) may trigger the default 50MB read threshold legitimately. Increase `file_read_threshold_mb` to reduce false positives.
- **Do not disable `auto_pause_on_high`** — A paused container is recoverable (`coi unfreeze`). A killed process is not. Leave auto-pause enabled and investigate before resuming.

## Long-Running and Overnight Tasks

- Use persistent mode (`[container] persistent = true` in your project config or profile) so the container survives if the session disconnects
- Use `coi attach` to reconnect to a running session from a different terminal
- Set a `max_duration` limit in your profile to prevent runaway tasks from consuming resources indefinitely:

```toml
[limits.runtime]
max_duration = "8h"
```

- Check `coi list` on return to see whether the container is still running or was killed by the security monitor

## Team and Shared Environments

- **Check in profile configs** — `.coi/profiles/` belongs in your repo alongside your code. New team members get a working environment without manual setup.
- **Use pinned image versions** — Reference `coi-myproject-v1.2` not `coi-myproject` in shared profiles. Mutable image names cause drift between team members' environments.
- **Document profiles in the README** — List available profiles and their intended use (e.g., "use `--profile research` for untrusted dependencies"). Contributors should not have to read `config.toml` to understand the workflow.
- **Set resource limits in shared profiles** — Prevent runaway AI sessions from monopolising memory or I/O bandwidth on shared machines:

```toml
[limits.memory]
limit = "8GiB"

[limits.disk]
read = "50MiB"   # max read rate (bytes/sec)
write = "50MiB"  # max write rate (bytes/sec)
```

The `[limits.disk]` settings (`read`, `write`, `max`) are I/O rate limits (bytes per second), not storage caps — they keep one container from starving others of disk bandwidth, but they do not bound how much disk space a session can consume. The only size cap available is `tmpfs_size`, which limits `/tmp` only.

## AI-Generated Code

- **Commit with hooks disabled** — AI-generated changes may break git hooks or trigger unexpected linters. Review and commit manually:

```bash
git -c core.hooksPath=/dev/null commit --no-verify -m "ai: your message"
```

- **Review before committing** — `git diff` is your friend. Coi limits what the AI can do outside your workspace, but it cannot review code quality.
- **Use `allowlist` mode for commit-triggering workflows** — If your git hooks push to external services, allowlist mode limits which services they can reach.

## Storage and Cleanup

- Run `coi clean` periodically to remove stopped containers that were not cleanly deleted
- Run `coi clean --orphans` if you notice nftables rule accumulation (especially if Docker is also running on the host)
- Run `coi image cleanup myproject- --keep 3` to prune old custom image versions
- Check storage pool health with `coi health` — it reports per-pool free space (warning at 80% usage) and, as of v0.11.1, each pool's driver: a `dir`-driver pool is flagged because every launch re-unpacks the full image (~5-6s per unpacked GB); recreate it with zfs/btrfs (re-running `install.sh` sets one up). On `dir` pools image size is a per-session cost, so lean images pay off twice

## See Also

- [Configuration](Configuration) - Full configuration reference
- [Security Best Practices](Security-Best-Practices) - Security-specific recommendations
- [Network Isolation](Network-Isolation) - Network mode configuration
- [Profiles](Profiles) - Reusable configuration bundles for different workloads
- [Snapshot Management](Snapshot-Management) - When and how to use snapshots
