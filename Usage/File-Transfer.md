# File Transfer

Transfer files and directories between host and containers.

## Push Files to Container

Copy files from your host into a container:

```bash
# Push single file
coi file push ./config.json my-container:/workspace/config.json

# Push directory (recursive)
coi file push -r ./src my-container:/workspace/src

# Push with custom permissions
coi file push ./script.sh my-container:/usr/local/bin/script.sh
```

## Pull Files from Container

Copy files from a container to your host:

```bash
# Pull single file
coi file pull my-container:/workspace/build.log ./build.log

# Pull directory (recursive)
coi file pull -r my-container:/home/code/.claude ./saved-sessions/session-123/

# Pull build artifacts
coi file pull -r my-container:/workspace/dist ./output/
```

## Common Use Cases

### Backup Session Data

```bash
# Save AI tool session to host
coi file pull -r my-container:/home/code/.claude ./backups/session-$(date +%Y%m%d)/
```

### Inject Configuration

```bash
# Push config files before starting session
coi file push ./custom-config.toml my-container:/home/code/.config/tool/config.toml
```

### Extract Build Artifacts

```bash
# Copy compiled binaries or build output
coi file pull -r my-container:/workspace/build ./artifacts/
```

### Share Files Between Containers

```bash
# Pull from one container, push to another
coi file pull my-container-1:/workspace/data.json ./temp.json
coi file push ./temp.json my-container-2:/workspace/data.json
```

## Notes

- File ownership is preserved when possible
- Use `-r` flag for recursive directory operations
- Paths in containers must be absolute
- Works with both running and stopped containers

### File Ownership and UID Shifting

Coi containers use Incus UID mapping — idmapped (`shift=true`) mounts or a per-instance `raw.idmap`, selected automatically per host/filesystem — so that files owned by your host user appear as the `code` user (UID 1000) inside the container. When pushing files, the ownership is shifted automatically — you do not need to `chown` files after pushing.

When pulling files, the reverse shift applies: files owned by `code` inside the container appear owned by your host user on the filesystem.

> **Note: Pushing to System Paths**
> Pushing files to paths outside `/home/code` and `/workspace` (e.g., `/usr/local/bin`) may require the file to be owned by root inside the container. Use `coi container exec <name> -- chown root:root /path/to/file` after pushing if the tool requires root ownership.

## See Also

- [Container Operations](Container-Operations) - Full list of container management commands
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - Session save, resume, and slot management
