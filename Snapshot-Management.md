# Snapshot Management

Snapshots capture the complete state of a container at a point in time. Use them to checkpoint before a risky operation, roll back to a known-good state, or branch into experimental work from the same starting point.

Stateless snapshots (default) capture the filesystem — installed packages, configuration changes, and any state written to the container's disk. They are fast and sufficient for most use cases.

Stateful snapshots (`--stateful`) additionally capture process memory, allowing you to restore a running session exactly mid-execution. Stateful snapshots are slower to create and require the container to be running at snapshot time; stateless snapshots can be taken from a stopped or running container.

> **Note: Restore Requires a Stopped Container**
> Before restoring any snapshot, stop the container with `coi container stop <name>`. Restoring into a running container is not supported.

## Commands

### Create Snapshots

```bash
# Auto-named snapshot (snap-YYYYMMDD-HHMMSS)
coi snapshot create

# Named snapshot
coi snapshot create checkpoint-1

# Include process memory state
coi snapshot create --stateful live

# Specific container
coi snapshot create -c coi-abc-1 backup
```

### List Snapshots

```bash
# Current workspace container
coi snapshot list

# Specific container
coi snapshot list -c coi-abc-1

# All Coi containers
coi snapshot list --all

# JSON output
coi snapshot list --format json
coi snapshot list --json         # shorthand for --format json
```

### Restore From Snapshot

**Note:** Requires container to be stopped first (`coi container stop <name>`)

```bash
# Restore with confirmation
coi snapshot restore checkpoint-1

# Skip confirmation
coi snapshot restore checkpoint-1 -f

# Restore with process state
coi snapshot restore checkpoint-1 --stateful
```

### Delete Snapshots

```bash
# Delete specific snapshot
coi snapshot delete checkpoint-1

# Delete all (with confirmation)
coi snapshot delete --all

# Delete all without confirmation
coi snapshot delete --all -f
```

### Show Snapshot Details

```bash
# Text output
coi snapshot info checkpoint-1

# JSON output
coi snapshot info checkpoint-1 --format json
```

## Container Resolution

When you do not specify a container, Coi resolves it in this order:

1. Uses `--container` flag if provided
2. Falls back to `COI_CONTAINER` environment variable
3. Auto-resolves from current workspace if exactly one container exists
4. Error if multiple containers found (use `--container` to specify)

## Safety Features

- **Restore requires stopped container** - Run `coi container stop <name>` first
- **Destructive operations require confirmation** - Skip with `--force` flag
- **Complete state capture** - Snapshots include container state and session data
- **Stateful snapshots** - Include process memory for live state preservation

## Use Cases

- **Checkpointing** - Save state before risky operations
- **Rollback** - Restore to previous working state
- **Branching experiments** - Try different approaches from same starting point
- **Backup** - Preserve container state before major changes

## Best Practices

1. **Snapshot before risky operations** - Create a checkpoint before applying large refactors, running migrations, or letting the AI make sweeping changes across many files

2. **Prefer stateless unless you need exact process state** - Stateless snapshots are significantly faster and smaller; use `--stateful` only when you need to resume a specific mid-execution process state

3. **Use descriptive names** - Auto-named snapshots (`snap-20260526-143022`) are hard to distinguish; prefer names like `before-db-migration` or `clean-deps`

4. **Clean up regularly** - Snapshots consume Incus storage pool space. Use `coi snapshot delete --all` or `coi image cleanup` to prevent accumulation

5. **Snapshots are not a substitute for git** - Snapshots capture container state, not just workspace files. For code changes, commit to git; use snapshots for environment state (installed tools, build caches)

## See Also

- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How snapshots fit into session persistence
- [Container Operations](Container-Operations) - Related container management commands
- [Configuration](Configuration) - Persistence mode and snapshot settings
