# Container Operations

## Container Management

### List Containers

```bash
# List all containers and sessions
coi list --all

# Filter by container status (as of v0.10.0)
coi list --running            # only Running containers
coi list --stopped            # only Stopped containers
coi list --status frozen      # general form; accepts every Incus instance
                              # state: running, stopped, frozen, error,
                              # starting, stopping, freezing, thawed,
                              # aborting, ready
# The three filter flags are mutually exclusive. Filters narrow the
# active-containers section only; saved sessions shown by --all are unaffected.

# Machine-readable JSON output (for programmatic use)
coi list --format=json
coi list --all --format=json
coi list --running --format=json   # filters compose with JSON output
coi list --json                    # shorthand for --format json (see below)

# Output shows container mode:
#   coi-abc12345-1 (ephemeral)   - will be deleted on exit
#   coi-abc12345-2 (persistent)  - will be kept for reuse
```

### JSON Output: `--json`

Every command that offers `--format text|json` also accepts `--json` as a
convenience alias for `--format json` (matching the `--json` that `coi top` and
`coi monitor` already had). `--format` remains the canonical selector; when both
are given, `--json` wins over `--format text`.

```bash
coi list --json
coi health --json
coi image list --json
coi image info <image> --json
coi container list --json
coi container info <ctr> --json
coi snapshot list --json
coi snapshot info <snap> --json
coi profile list --json
coi validate profile --json
coi tmux list --json
coi version --json
```

> **Exception:** `coi container exec` has its own `--format json|raw` (a different
> choice — how to frame the captured output, not text-vs-json), so it does not
> take `--json`.

### Kill Containers

```bash
# Kill specific container (stop and delete)
coi kill <container-name>

# Kill multiple containers
coi kill <container1> <container2>

# Kill all containers (with confirmation)
coi kill --all

# Kill all without confirmation
coi kill --all --force
```

### Clean Up

```bash
# Clean up stopped/orphaned containers
coi clean

# Skip confirmation
coi clean --force

# Detect containers in unused storage pools
coi clean --pools

# Clean orphaned veths and stale nft/iptables rules
coi clean --orphans

# Dry run to see what would be cleaned
coi clean --orphans --dry-run
```

### Session Info

```bash
# Show details of the current workspace's saved session (auto-detected)
coi info

# Show details of a specific saved session
coi info <session-id>
```

`coi info` shows detailed information about a saved session, not a container — pass a session ID (see `coi sessions`), or omit it to auto-detect the current workspace's session.

### Version

```bash
# Show Coi version
coi version
```


## Running Sessions Non-Interactively (`coi run`)

`coi run` starts a Coi session container like `coi shell`, but is designed for non-interactive use — CI pipelines, scripts, and automated workflows where there is no TTY.

```bash
# Run a command non-interactively inside a Coi container
coi run -- <command> [args...]

# Examples
coi run -- npm test
coi run -- pytest tests/
coi run -- bash -c "pip install -r requirements.txt && python validate.py"
```

Key differences from `coi shell`:

- **No image-build prompt without a TTY** - When run from an interactive terminal, a missing image triggers the same build-it-now prompt as `coi shell`; when stdin is not a terminal (CI, pipes), the error is returned immediately instead
- **No tmux** - The command runs directly without a tmux session wrapper
- **Suitable for piped input** - Works correctly when stdin is not a terminal
- **Same isolation** - Uses the same container isolation, credential protection, and network modes as `coi shell`

All operational session flags are supported; image and persistence come from config or the selected profile (as of v0.10.0):

```bash
coi run --profile rust-dev -- cargo test    # profile carries image/persistence/limits
coi run --slot 2 -- npm install
coi run -- ./build.sh                       # [container] image/persistent from config
```

With `[container] persistent = true`, `coi run` reuses its stopped container on the next run (installed packages and caches survive between runs).

> **Note: Build the Image First in CI**
> Without an interactive terminal (CI, scripts, piped input), `coi run` cannot prompt to build a missing image and fails immediately. Run `coi build` before using `coi run` in CI environments.

## Advanced Container Operations

Low-level container commands for advanced use cases and automation.

### List Containers

```bash
# List all containers in text format (default)
coi container list

# List all containers in JSON format (for programmatic use)
coi container list --format=json

# Example JSON output structure:
# [
#   {
#     "name": "my-container",
#     "status": "Running",
#     "created_at": "2026-02-09T10:30:00Z",
#     "config": {...},
#     "state": {...}
#   }
# ]
```

**Use Cases:**

- **Automation** - Query container state in scripts
- **Monitoring** - Check container status programmatically
- **Integration** - Parse container info in other tools
- **Admin checks** - Verify running containers exist

**Note:** For a higher-level view with session info and workspace details, use `coi list` instead. The `coi container list` command provides raw Incus container data without session enrichment.

### Launch Containers

```bash
# Launch a new container
coi container launch coi-default my-container

# Launch ephemeral container (auto-delete on stop)
coi container launch coi-default my-container --ephemeral
```

**Note:** Coi uses a three-step launch sequence (`incus init` → configure → `incus start`) to ensure Docker support flags and other configuration are applied before the container starts. This guarantees Docker and Docker Compose work correctly inside session containers.

### Start/Stop/Delete

```bash
# Start container
coi container start my-container

# Stop container
coi container stop my-container

# Force stop
coi container stop my-container --force

# Delete container
coi container delete my-container

# Force delete
coi container delete my-container --force
```

### Execute Commands

```bash
# Execute command in container
coi container exec my-container -- ls -la /workspace

# With user, environment, and working directory
coi container exec my-container --user 1000 --env FOO=bar --cwd /workspace -- npm test

# Allocate a PTY for interactive sessions (shells, tmux, etc.)
coi container exec my-container -t -- bash
coi container exec my-container --tty -- bash

# Interactive tmux session with PTY
coi container exec my-container -t -- tmux attach

# Capture output in JSON format
coi container exec my-container --capture -- echo "hello"

# Capture output in raw format (for scripting)
coi container exec my-container --capture --format=raw -- pwd
```

**PTY Allocation (`-t, --tty`)**

The `-t` or `--tty` flag allocates a pseudo-terminal (PTY) for the command execution. This is essential for:

- **Interactive shells** - Running bash, zsh, or other shells interactively
- **Tmux sessions** - Attaching to or creating tmux sessions
- **Terminal applications** - Any application that requires terminal capabilities (vim, nano, htop, etc.)
- **Programs checking for TTY** - Applications that behave differently when stdin is a terminal

**Important Notes:**

- The `--tty` and `--capture` flags are mutually exclusive (cannot be used together)
- PTY allocation requires stdin/stdout/stderr to be connected to the terminal
- Without `-t`, the command runs in non-interactive mode (stdin is not a terminal)

**Example Use Cases:**

```bash
# Start an interactive shell
coi container exec my-container -t -- bash

# Run a command that requires TTY
coi container exec my-container -t -- python -c "import sys; print(sys.stdin.isatty())"
# Output: True

# Without -t, stdin is not a TTY
coi container exec my-container -- python -c "import sys; print(sys.stdin.isatty())"
# Output: False

# Attach to tmux session in container
coi container exec my-container -t -- tmux attach -t session-name

# Run vim editor
coi container exec my-container -t -- vim /workspace/file.txt
```

### Check Status

```bash
# Check if container exists
coi container exists my-container

# Check if container is running
coi container running my-container
```

### Mount Directories

```bash
# Mount directory into container
coi container mount my-container workspace /home/user/project /workspace --shift
```

Extra mounts are configured via config file:

```toml
# ~/.coi/config.toml
[[mounts.default]]
host = "/path/to/data"
container = "/data"

[[mounts.default]]
host = "/path/to/config"
container = "/config"
readonly = true
```

**UID/GID Remapping:** When the `code_uid` config differs from the image default (1000), Coi automatically remaps the container user's UID/GID and re-owns home directory files. This prevents "Permission denied" errors and `I have no name!` shell prompts.

### Manage `/etc/hosts` at Runtime (`coi hosts`)

`coi hosts add|list|remove <container> …` manages static host entries in a running container's `/etc/hosts`, applying firewall reachability that matches the container's network mode. Entries are session-scoped (torn down when the container stops).

```bash
coi hosts add my-container 10.0.0.5 db.internal   # add an IP → hostname(s) mapping
coi hosts list my-container                        # list coi-managed entries
coi hosts remove my-container db.internal          # remove by hostname
```

For entries that should be present on every launch, declare them in `[[network.hosts]]` config instead — see [Static Host Entries](Static-Host-Entries) for the full reference and the per-mode reachability rules.

## Use Cases

- **Automation** - Script container operations for CI/CD
- **Debugging** - Direct container access for troubleshooting
- **Custom workflows** - Build specialized container workflows
- **Testing** - Create test environments programmatically

## See Also

- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How containers start, stop, and resume
- [Tmux Automation](Tmux-Automation) - Automating session interaction
- [File Transfer](File-Transfer) - Transferring files to and from containers
- [Snapshot Management](Snapshot-Management) - Saving and restoring container state
