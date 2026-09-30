# Container Lifecycle and Sessions

Understanding how containers and sessions work in Coi.

## How It Works Internally

1. Containers are always launched as non-ephemeral (persistent in Incus terms)
   - This allows saving session data even if the container is stopped from within (e.g., `sudo shutdown 0`)
   - Session data can be pulled from stopped containers, but not from deleted ones

2. **Inside the container**: `tmux` → `bash` → `<ai-tool>`
   - When the AI tool exits, you are dropped to bash
   - From bash you can: type `exit`, press `Ctrl+b d` to detach, or run `sudo shutdown 0`

3. **On cleanup** (when you exit/detach):
   - Session data (tool config directory) is always saved to `~/.coi/sessions-<tool>/`
   - If persistent mode is NOT enabled: container is deleted after saving
   - If persistent mode is enabled (`[container] persistent = true` in config or profile, as of v0.10.0 — previously the `--persistent` flag): container is kept for reuse
   - Cleanup is protected by `sync.Once` to prevent race conditions between signal handlers and deferred cleanup

4. **Docker/Compose support**: Session containers automatically get Docker support flags (`security.nesting`, `security.syscalls.intercept.mknod/setxattr`) applied before the container starts, so Docker and Docker Compose work out of the box inside sessions.

## What Gets Preserved

| Mode | Workspace Files | AI Tool Session | Container State |
|------|----------------|-----------------|-----------------|
| **Default (ephemeral)** | Always saved | Always saved | Deleted |
| **Persistent (`persistent = true`)** | Always saved | Always saved | Kept |

## Session vs Container Persistence

### `--resume` Flag

Restores the AI tool conversation in a fresh container:

- Use when you want to continue a conversation but do not need installed packages
- Container is recreated, only tool session data is restored
- **Profile auto-restore**: The profile used when the session was originally created is automatically restored - no need to pass `--profile` again. Explicitly passing `--profile` on resume overrides the saved profile.
- **Workspace-scoped**: Only finds sessions from the current workspace directory (security feature) — unless `session_name` is set: named sessions find their saved sessions by NAME from any workspace location (see [Named Sessions](#named-sessions-session_name--v0111))

### Persistent Mode (`[container] persistent = true`)

Keeps the entire container with all modifications:

- Use when you have installed tools, built artifacts, or modified the environment
- `coi attach` reconnects to the same container with everything intact
- Set it in the project's `./.coi/config.toml`, your user config, or a profile — there is no CLI flag (as of v0.10.0)
- On `--resume`, an explicitly configured `persistent` wins over the resumed session's saved mode, so config can convert a session's persistence; when unset, the saved mode is inherited

### `coi persist`

Converts one or more ephemeral containers to persistent mode:

```bash
# Convert a specific container to persistent (keeps it on exit)
coi persist <container-name>

# Convert all containers (with confirmation)
coi persist --all
```

Use `coi list` to see active containers and their persistence mode.

This is useful when you started an ephemeral session but later decide you want to keep the container (e.g., after installing tools or setting up the environment).

## SSH Agent Forwarding

Forward your host's SSH agent into the container so git-over-SSH works without copying private keys:

```toml
# ~/.coi/config.toml
[ssh]
forward_agent = true
```

The host's `SSH_AUTH_SOCK` is bridged into the container at `/tmp/ssh-agent.sock` via an Incus proxy device. Coi automatically sets `SSH_AUTH_SOCK` inside the container and includes retry logic to handle an Incus race condition where proxy devices on freshly-launched containers may not create the listen socket immediately.

**Key points:**

- Disabled by default (opt-in for security)
- Gracefully skips if no SSH agent is running on the host
- Works with both ephemeral and persistent containers
- Device is replaced on persistent container re-entry

## Environment Variable Forwarding

Selectively forward host environment variables into the container by name:

```toml
# ~/.coi/config.toml
[defaults]
forward_env = ["ANTHROPIC_API_KEY", "GITHUB_TOKEN", "AWS_ACCESS_KEY_ID"]

# Static environment variables (always set)
[defaults.environment]
RUST_BACKTRACE = "1"
NODE_ENV = "development"
```

**Key points:**

- Values are read from the host at session start - never stored in config files
- Forwarded variables are passed securely via `tmux new-session -e` and `tmux set-environment` instead of being inlined as `export KEY=VAL` in the command string - secrets do not appear in `ps` output and propagate to new tmux windows/panes
- Missing variables produce a warning but do not fail the session
- Config values from different levels are merged (deduplicated)
- Profile-level `environment` vars are also applied

## Stopping Containers

### From Inside the Container

- `exit` in bash → exits bash but keeps container running (use for temporary shell exit)
- `Ctrl+b d` → detaches from tmux, container stays running
- `close` or `sudo poweroff` → stops container, session is saved, then container is deleted (or kept in persistent mode). The `close` command is a safe alias for `poweroff` that only exists inside Coi containers, preventing accidental host shutdowns if typed outside the container.

### From Outside (Host)

- `coi shutdown <name>` → graceful stop with session save, then delete (60s grace window by default)
- The grace window is config, not a flag (as of v0.10.0): `[container] shutdown_timeout = 30`
- `coi shutdown --all` → graceful stop all containers (with confirmation)
- `coi shutdown --all --force` → graceful stop all without confirmation
- `coi close` → accepted alias for `coi shutdown` (as of v0.11.0), identical flags and all (e.g. `coi close --all`). It echoes the in-container `close` verb: for the usual ephemeral container the two match (both end with the container gone), but for a persistent container they differ — the in-container `close` keeps it (stopped, reused next session) whereas `coi shutdown`/`coi close` deletes it. Shells will not tab-suggest `close`, since completion covers command names only, not aliases.
- `coi kill <name>` → force stop and delete immediately
- `coi kill --all` → force stop and delete all containers (with confirmation)
- `coi kill --all --force` → force stop all without confirmation
- `coi unfreeze` → unfreeze a paused container (e.g., after security monitor auto-pause)


## Slot System

A slot is a numbered container instance tied to the same workspace. Slots let you run multiple independent AI sessions against the same project directory in parallel, each with its own isolated home directory, installed packages, and conversation history.

### Container Naming

Coi derives a short hash from your workspace path (or from `[container] session_name`, when set) and uses it in the container name:

```text
coi-abc12345-1    # first slot
coi-abc12345-2    # second slot
coi-abc12345-3    # third slot
```

The hash is stable — the same workspace always gets the same prefix.

### Slot Allocation

Slots are allocated automatically when you run `coi shell`:

- If no container exists for this workspace, slot 1 is created
- If slot 1 is already running, slot 2 is created
- If slots 1 and 2 are running, slot 3 is created

By default slots are allocated automatically — each `coi shell` in a new terminal picks the next available number. You can also pin a specific slot with `--slot N` (`0` means auto-allocate).

### What Is Isolated Per Slot

Each slot gets its own:

- Home directory (`/home/code`)
- Installed packages and system state
- Running processes
- AI tool conversation history

All slots share the same `/workspace` mount — they read and write the same project files.

### Aliases and Slots

When a container alias is configured, additional slots are named with a numeric suffix:

```bash
# Config: alias = "myproject"
coi shell    # → myproject (slot 1)
coi shell    # → myproject-2 (slot 2)
coi shell    # → myproject-3 (slot 3)

# Attach by alias
coi attach myproject      # slot 1
coi attach myproject-2    # slot 2
```

### Managing Multiple Slots

```bash
coi list              # shows all slots with their names and status
coi list --running    # only slots whose container is Running (as of v0.10.0;
                      # also --stopped, or --status <any Incus state>)
coi shutdown --all    # gracefully shut down all running slots
coi kill myproject-2  # force-stop a specific slot by alias
```

## Example Workflows

### Quick Task (Default Mode)

```bash
coi shell                    # Start session with default AI tool
# ... work with AI assistant ...
sudo poweroff                # Shutdown container → session saved, container deleted
coi shell --resume           # Continue conversation in fresh container
```

**Note:** `exit` in bash keeps the container running - use `sudo poweroff` or `sudo shutdown 0` to properly end the session. Both require sudo but no password.

### Long-Running Project (Persistent Mode)

```toml
# ./.coi/config.toml
[container]
persistent = true
```

```bash
coi shell                    # Start persistent session (per config above)
# ... install tools, build things ...
# Press Ctrl+b d to detach
coi attach                   # Reconnect to same container with all tools
sudo poweroff                # When done, shutdown and save
coi shell --resume           # Resume with all installed tools intact
```

### Parallel Sessions (Multi-Slot)

```bash
# Terminal 1: Start first session (auto-allocates slot 1)
coi shell
# ... working on feature A ...
# Press Ctrl+b d to detach (container stays running)

# Terminal 2: Start second session (auto-allocates slot 2)
coi shell
# ... working on feature B in parallel ...

# Both sessions share the same workspace but have isolated:
# - Home directories (~/slot1_file won't appear in slot 2)
# - Installed packages
# - Running processes
# - AI tool conversation history

# List both running sessions
coi list
#   coi-abc12345-1 (ephemeral)
#   coi-abc12345-2 (ephemeral)

# When done, shutdown all sessions
coi shutdown --all
```

### Container Aliases

Assign human-friendly names to your containers for easy management from any directory:

```toml
# .coi/config.toml
[container]
alias = "myproject"
```

```bash
coi shell myproject              # Launch session using alias (from any directory)
coi attach myproject             # Attach to running aliased container
coi kill myproject --force       # Kill by alias
coi attach myproject-2           # Attach to slot 2 by alias
```

Aliases are registered in `~/.coi/aliases.json` on first use and stored as `user.coi.alias` metadata on each container. `coi list` shows aliases next to container names.

## Named Sessions (`session_name`) — v0.11.1

By default a session's identity — its container name, slot and port allocation, and the saved-session store `--resume`/`--continue` searches — is keyed on a hash of the workspace's absolute path. Setting `[container] session_name` (typically in a profile, paired with `persistent = true`) keys all of that on the name instead, so the same session continues from any workspace location: a moved checkout, or several checkouts sharing one session.

```toml
# ~/.coi/profiles/myproj/config.toml
[container]
persistent = true
session_name = "myproj"
```

Notes:

- On reuse from a new location the workspace mount is reconciled automatically (the container's workspace device is remounted from the current path before protections are re-applied). A running named session still mounting a different checkout is refused — stop it first.
- The name is honored from trusted scope only (`~/.coi` / `COI_CONFIG`, and profiles under them): it selects which persistent container and saved state a launch attaches to, so a cloned repo's `.coi` cannot attach itself to your session.
- Launching a named session while it is already active on another slot creates a fork (a fresh container without the session's state) — coi warns loudly and names the slot to reuse instead.
- Adopting a name over an existing path-keyed workspace starts a fresh container lineage but carries the saved conversation history forward.
- If the name-carrying profile is selected via `--profile` or an alias (rather than `[defaults] profile`), repeat that flag on operational commands (`coi attach`/`monitor`/`snapshot`) so they resolve the same identity.
- An explicit `coi shell --container <name>` bypasses both the running-workspace refusal and the workspace remount: naming a container means "enter it as it is", with its existing mounts.

### Running a Different AI Tool in the Same Container (v0.12)

Because a named session's identity is a hash of workspace + `session_name` — not the tool — two profiles that share one `session_name` (with `persistent = true`) resolve to the same container. Each profile sets its own `[tool] name`, so you can re-enter one persistent box under a different assistant while keeping its code, installed packages, and running services:

```toml
# ~/.coi/profiles/box-claude/config.toml
[container]
persistent = true
session_name = "box"

[tool]
name = "claude"
```

```toml
# ~/.coi/profiles/box-codex/config.toml
[container]
persistent = true
session_name = "box"

[tool]
name = "codex"
```

```bash
coi shell --profile box-claude   # creates/enters "box", running Claude Code
# ... install tools, build things, then exit (the container is kept) ...
coi shell --profile box-codex    # re-enters the SAME box, now running Codex
```

There is no `--tool` flag — the tool is chosen entirely by which profile you launch. Both profiles must set the same `session_name` with `persistent = true`, and both must live in trusted scope (see the trusted-scope note above).

**Credentials are seeded on first switch (#708).** The first time a newly-selected tool runs in a reused container, coi seeds that tool's CLI config and credentials into it, so the tool you just switched to is authenticated even though it was never used in this container before. coi does not re-copy the original tool's config, and it never touches conversation history — history is per-tool, so each assistant keeps its own transcripts across switches.

Limitations (by design in v0.11.1):

- Two truly concurrent launches of one name from different workspaces can race the workspace remount.
- Two profiles sharing a `session_name` share one container regardless of their images — nothing registers a name's "owner".
- `[[mount]]` sources that lived under a moved workspace keep their old host paths (use absolute, workspace-independent mount paths with named sessions).
- With `preserve_workspace_path` the in-container path changes across checkouts, so cwd-keyed tool history (e.g. Claude's per-project store) does not follow; the default `/workspace` mount keeps history continuous.

## See Also

- [Container Operations](Container-Operations) - Commands for managing running containers
- [Snapshot Management](Snapshot-Management) - Checkpointing and restoring container state
- [Tmux Automation](Tmux-Automation) - Interacting with AI sessions programmatically
- [File Transfer](File-Transfer) - Moving files between host and container
- [Configuration](Configuration) - Persistence, ephemeral mode, and slot settings
