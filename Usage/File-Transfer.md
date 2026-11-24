# File Transfer

Transfer files and directories between host and containers.

## Push Files to Container

Copy files from your host into a container:

```bash
# Push single file
coi file push ./config.json my-container:/workspace/config.json

# Push directory (creates /workspace/app; into an existing
# directory it creates <dir>/src)
coi file push -r ./src my-container:/workspace/app

# Push to a system path
coi file push ./script.sh my-container:/usr/local/bin/script.sh
```

For a directory push (`-r`), a destination ending in `/` must be an existing directory. An existing directory, with or without the slash, receives the pushed directory inside it, under the local directory's name. Any other destination is the exact target. The final target must not already exist: coi does not merge into or replace an existing directory, and refuses the push with exit code 2. So pushing `./src` again to an existing `my-container:/workspace/src` creates `/workspace/src/src`, and is refused if that exists too. To update a copy, remove it in the container and push to the parent (`my-container:/workspace`), or push to a new name. A single file push does not follow these rules.

## Pull Files from Container

Copy files from a container to your host:

```bash
# Pull single file
coi file pull my-container:/workspace/build.log ./build.log

# Pull directory (recursive)
coi file pull -r my-container:/home/code/.claude ./saved-sessions/session-123

# Pull build artifacts
coi file pull -r my-container:/workspace/dist ./output
```

A destination ending in `/` must be an existing directory. An existing directory, with or without the slash, receives the pulled entry inside it. Any other destination is the exact target and must not already exist (a single file may overwrite an existing file).

A single file pull copies only a regular file. If the container path is a symlink, coi refuses the pull with `refusing to pull <path>: not a regular file` and exit code 1, on every supported Incus version, and creates nothing on the host. To get the content, pull the link's target path. A recursive pull (`-r`) leaves symlinks and special files out of the pulled tree.

## Common Use Cases

### Backup Session Data

```bash
# Save AI tool session to host
coi file pull -r my-container:/home/code/.claude ./backups/session-$(date +%Y%m%d)
```

### Inject Configuration

```bash
# Push config files before starting session
coi file push ./custom-config.toml my-container:/home/code/.config/tool/config.toml
```

### Extract Build Artifacts

```bash
# Copy compiled binaries or build output
coi file pull -r my-container:/workspace/build ./artifacts
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
- Push and pull take a container path without a leading `/` from the container root, with and without `-r`: `my-container:workspace/notes.txt` is `/workspace/notes.txt`. Absolute container paths are clearer
- Works with both running and stopped containers

### File Ownership and UID Shifting

Coi containers use Incus UID mapping — idmapped (`shift=true`) mounts or a per-instance `raw.idmap`, selected automatically per host/filesystem — so that files owned by your host user appear as the `code` user (UID 1000) inside the container. `coi file push` does not change ownership: a pushed file keeps the UID, GID and mode it has on the host, on every supported Incus version (from Incus 7.5, `incus file push` itself makes a new file root-owned and applies the umask to its mode, so coi passes `--uid`, `--gid` and `--mode` explicitly). With `-r`, each entry of the tree keeps its own UID, GID and mode; with an Incus 7.5+ client coi pushes with `--archive`, which also keeps each entry's modification time. One exception: a `-r` push to a target with a name other than the local directory's, when the tree holds a FIFO, socket or device (or with an Incus 6.x client). coi then pushes a copy of the tree made on the host. The copy leaves out FIFOs, sockets and devices, so they do not arrive in the container. When coi runs as a non-root user, entries that belong to root or another user can come out owned by your user; entries you own keep their owner. If an entry's owner differs from the `code` user's, `chown` the file inside the container after pushing (for example `coi container exec <name> -- chown 1000:1000 /path/to/file`).

A pulled file is owned by the host user who runs `coi file pull`, and keeps the mode it has in the container, masked by your umask. coi pulls the entry with `incus file pull` into a temporary directory on the host, then moves it to the destination.

> **Note: Pushing to System Paths**
> Pushing files to paths outside `/home/code` and `/workspace` (e.g., `/usr/local/bin`) may require the file to be owned by root inside the container. Use `coi container exec <name> -- chown root:root /path/to/file` after pushing if the tool requires root ownership.

## See Also

- [Container Operations](Container-Operations) - Full list of container management commands
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - Session save, resume, and slot management
