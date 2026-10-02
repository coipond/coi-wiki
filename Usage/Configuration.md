# Configuration

## Configuration Files

Coi uses TOML configuration files. The primary user config lives at:

```text
~/.coi/config.toml
```

Create this file manually - no generator command is needed. See the [Full Config Reference](#full-config-reference) below for a complete example you can copy and trim down.

## Configuration Hierarchy

Settings are loaded in order, with later sources overriding earlier ones:

| Priority | Source | Path |
|----------|--------|------|
| 1 (lowest) | Built-in defaults | embedded `profiles/default/config.toml` |
| 2 | User config | `~/.coi/config.toml` (or the file `$COI_CONFIG` points at) |
| 3 | Project config | `./.coi/config.toml` (repo root) |
| 4 (highest) | Profile | selected with `coi shell/run --profile <name>` |

Each layer only overrides the fields it explicitly sets. Unset fields inherit from the previous layer.

There is no environment-variable config layer (as of v0.10.0): the legacy `CLAUDE_ON_INCUS_*` and `COI_LIMIT_*` overrides were removed — they silently outranked explicit config. `$COI_CONFIG` itself and test/debug toggles still exist, but they select or diagnose, they do not configure. CLI flags are operational only (`--workspace`, `--slot`, `--resume`/`--continue`, `--profile`) and never carry configuration.

## Per-Repository Configuration

Place a `.coi/config.toml` file in any repository root to auto-configure Coi for that project. When you run `coi shell` from that directory, the project config is automatically loaded.

**Key points:**

- Only the fields you set are overridden - everything else inherits from your user and system defaults
- The file uses the same TOML format and sections as the user config
- The `.coi/` directory can also contain build scripts, profile directories, and other project assets
- Useful for teams: commit `.coi/` to the repo so every developer gets the same container image, environment, and resource limits without touching their personal config

> **Migration from 0.7.x:** Project config has moved from `.coi.toml` to `.coi/config.toml`. Coi refuses to start if a `.coi.toml` file is detected, displaying the migration command: `mkdir -p .coi && mv .coi.toml .coi/config.toml`.

### Example `.coi/config.toml`

```toml
# my-project/.coi/config.toml - project-specific overrides

[container]
image = "coi-rust"
persistent = true
# alias = "my-rust-project"
# storage_pool = ""

[defaults]
forward_env = ["CARGO_REGISTRY_TOKEN"]

[defaults.environment]
RUST_BACKTRACE = "1"

[tool]
name = "claude"
permission_mode = "bypass"

[tool.claude]
effort_level = "high"

[limits.cpu]
count = "4"

[limits.memory]
limit = "4GiB"

[limits.runtime]
max_duration = "4h"
```

With this file in your repo root and a user config that sets `code_uid = 1000` and `group = "incus-admin"`, those user settings still apply - only `image`, `persistent`, `forward_env`, `environment`, and `limits` are overridden by the project config.

### What You Can Set In `.coi/config.toml`

Any section from the full config is valid in a project config:

| Section | Example use case |
|---------|-----------------|
| `[container]` | Image, persistence, alias, storage pool, named session (`session_name`), shutdown/readiness timeouts |
| `[defaults]` | Environment variables, `forward_env`, and `profile` — the profile a no-flag `coi` uses (`profile` is trusted-scope only; ignored with a warning from a project config). In profiles, `forward_env` is a top-level key, not under `[defaults]`. |
| `[limits.*]` | CPU, memory, disk, and runtime limits |
| `[tool]` | Default AI tool, permission mode, executable (`binary`) — but `context_file`, `context_json_file` and `pre_launch` are **trusted-scope config only** (ignored with a warning from a project config or a project profile) |
| `[network]` | Network isolation mode and allowed domains (but `[[network.hosts]]` is **trusted-scope config only** — ignored from project config) |
| `[mounts]` | Additional mount points (out-of-workspace mounts gated behind `coi trust`) |
| `[[sockets]]` | Forward host Unix sockets into the container (gated behind `coi trust`) |
| `[ports]` | Publish container TCP ports on the host — `pool` + `[[ports.map]]` (gated behind `coi trust`; see [Port Publishing](Port-Publishing)) |
| `[[credentials]]` | Copy credential files from host into the container (ad-hoc entries gated behind `coi trust`) |
| `[defaults.env_commands]` (+ `env_command_timeout`) | Mint env vars from host commands, and the duration bounding each invocation — **trusted-scope config only** (ignored from project config) |
| `[prompts]` | Named prompts for headless `coi run --prompt-name` — **trusted-scope config only** (ignored from project config) |
| `[git]` | Git hooks write access (but `protected_branches` is **trusted-scope config only** — a project config can't change the list) |
| `[ssh]` | SSH agent forwarding |
| `[timezone]` | Container timezone (host, fixed, or UTC) |
| `[security]` | Protected path overrides |
| `[monitoring]` | Security monitoring settings |
| `profiles/` | Named configuration profiles (as [profile directories](Profiles)) |

## Full Config Reference

```toml
[container]
image = "coi-default"
persistent = false
storage_pool = ""              # empty = Incus default pool
# session_name = ""           # Key the session identity on this name instead of the
                              # workspace path: the same persistent session (container,
                              # slots, ports, saved sessions for --resume) continues
                              # from ANY workspace location. Typically set in a profile
                              # with persistent = true. Trusted scope only;
                              # letters/digits/._-, starting alphanumeric.
# alias = "myproject"          # Human-friendly name for this workspace's containers
# shutdown_timeout = 60        # Seconds to wait for graceful shutdown before force-killing (default: 60)
# ready_timeout = 30           # Seconds to wait for a launched container to become ready (default: 30)
#                              # Raise on slow hosts (nested virt, cold storage pools)

[container.build]
# base = "coi-default"
# script = "build.sh"

[defaults]
# Profile a bare `coi` (no --profile) launches into. Lowest-precedence profile
# source: an explicit --profile, an alias, and a --resume-remembered profile all
# win over it. Honored ONLY from trusted-scope config (~/.coi/config.toml /
# $COI_CONFIG); ignored (with a warning) from a project ./.coi/config.toml.
# An unknown name is a hard error only when `coi shell`/`coi run` applies it.
# profile = "pickles"
# Forward host environment variables into the container by name
# Values are read from the host at session start - never stored in config
# forward_env = ["ANTHROPIC_API_KEY", "GITHUB_TOKEN"]
# Static environment variables set inside the container
# [defaults.environment]
# MY_VAR = "value"
# Command-sourced env vars: run a host command at session start and inject its
# trimmed stdout as the value — for minting short-lived secrets per session.
# Honored ONLY from trusted-scope config (~/.coi/config.toml / $COI_CONFIG);
# ignored (with a warning) from an untrusted project ./.coi/config.toml.
# [defaults]
# env_command_timeout = "30s"   # bounds each [env_commands] invocation (default 30s);
#                               # trusted-scope only. Also settable at a profile's root (see Profiles).
# [defaults.env_commands]
# AWS_BEARER_TOKEN_BEDROCK = "~/bin/mint-bedrock-key.sh"

# Named prompts for headless `coi run --prompt-name <name>` (see Headless
# Orchestration). Each value is an inline string OR a { file = "..." } table
# whose path resolves relative to this config file. TRUSTED-SCOPE ONLY: a
# [prompts] table in an untrusted project ./.coi/config.toml is stripped at load.
# [prompts]
# tidy = "Run the formatter and linter, commit any fixes."
# nightly-maintenance = { file = "prompts/nightly.md" }

[tool]
name = "claude"              # AI coding tool: "claude", "opencode", "pi"
permission_mode = "bypass"   # "bypass" (default) or "interactive"
# binary = "/workspace/scripts/claude-wrapper.sh"
#                            # Optional: executable to launch instead of the tool's default
#                            # (e.g. a wrapper script); it receives the tool's usual arguments.
#                            # A plain command name or path — no spaces or shell syntax.
# Commands run inside the container, in order, before the tool starts in
# `coi shell` — e.g. keep the agent current without rebuilding the image.
# Output shows in the session; each command gets 5 minutes (it and anything it
# started are stopped after that); a failing or slow command is reported and
# the tool starts anyway; Ctrl+C skips the rest. Commands get no terminal input.
# TRUSTED-SCOPE ONLY: ignored (with a warning) from a project ./.coi/config.toml
# or a project profile. See "Running commands before the agent" below.
# pre_launch = ["claude update"]
# Path to a custom context file injected as ~/SANDBOX_CONTEXT.md in every container.
# Supports ~ expansion. If empty, the built-in template is used.
# context_file = "~/my-sandbox-context.md"
# Also emit a structured, versioned ~/SANDBOX_CONTEXT.json companion for
# programmatic consumers (default: false).
# context_json = true
# context_json_file = "~/my-context.json"   # optional: custom JSON template
# NOTE: context_file / context_json_file are honored only from trusted-scope
# config (~/.coi/config.toml / $COI_CONFIG), not from a project ./.coi/config.toml.
# Auto-inject sandbox context into tool's native context system (default: true)
# Claude: writes to ~/.claude/CLAUDE.md; OpenCode: sets instructions field in opencode.json
# auto_context = true

# Claude-specific settings
[tool.claude]
# model = "opus"             # Claude model, delivered as ANTHROPIC_MODEL: an alias
#                            # like "opus" or a full ID like "claude-opus-4-8".
#                            # Unset (default): Claude Code uses its own default.
effort_level = "medium"      # "low", "medium", "high", "xhigh", "max", or "auto"
                             # unset (default): user controls it interactively

[shell]
# use_tmux = true            # false = run the tool directly without a tmux
#                            # session (was the removed `--tmux` flag)

[paths]
sessions_dir = "~/.coi/sessions"
storage_dir = "~/.coi/storage"
logs_dir = "~/.coi/logs"
# preserve_workspace_path = true  # Mount at same path as host instead of /workspace

[incus]
project = "default"          # Incus project for Coi containers/images; `coi build` works in any project, not just "default"
group = "incus-admin"
code_uid = 1000
code_user = "code"
# disable_shift = false      # Force UID shifting off; raw.idmap is used instead. Rarely
                             # needed as of v0.11.1: Coi checks the workspace/mount source
                             # filesystems itself and skips shift for FUSE-family/9p sources
                             # (OrbStack/Colima/Lima host shares), with an automatic
                             # raw.idmap fallback when an idmapped-mount start fails anyway.
                             # Keep it only for a source those checks clear but that still
                             # can't do idmapped mounts. See the macOS Setup Guide.

[ssh]
forward_agent = false  # Forward host SSH agent (keys stay on host)

[network]
# mode = "restricted"        # "restricted" (default), "open", or "allowlist"
# block_private_networks = true
# block_metadata_endpoint = true
# allowed_domains = ["github.com", "pypi.org"]       # Only used in "allowlist" mode
# refresh_interval_minutes = 30                       # Max DNS refresh interval (actual uses DNS TTL if shorter)
# allow_local_network_access = false                  # Allow connections from entire local network
# use_sudo = true                                      # false = never invoke sudo for nft/iptables (see "Running without sudo")

# Static /etc/hosts entries with mode-aware firewall reachability (v0.11.0).
# TRUSTED-SCOPE ONLY: entries in a project ./.coi/config.toml are ignored with a
# warning. See the Static Host Entries wiki page for the per-mode reachability rules.
# [[network.hosts]]
# ip        = "10.0.0.5"              # IPv4 address
# hostnames = ["db.internal", "db"]   # one or more names for that address

[network.logging]
# enabled = true                   # Network logging enabled by default
# path = "~/.coi/logs/network.log" # Default log path

[mounts]
# Default mounts applied to all sessions
# [[mounts.default]]
# host = "~/.aws"
# container = "/home/code/.aws"
# readonly = false              # Set to true for read-only mount
# shift = true                  # Override UID/GID shifting for this mount (default: inherit
#                               # the session decision). Use shift = true when a writable mount
#                               # shows up owned by nobody:nogroup instead of the `code` user.

# See "Mounting Additional Files" section below for common mount patterns

# Forward host Unix sockets into the container via an Incus proxy device, so the
# host endpoint never enters the container (the building block for credential
# brokers). Sockets from an untrusted project ./.coi/config.toml are gated behind
# `coi trust`, like out-of-workspace mounts.
# [[sockets]]
# host = "~/.coi/broker.sock"    # host socket (~ expanded)
# container = "/run/broker.sock" # absolute path inside the container
# env = "AWS_BROKER_SOCK"        # optional: env var set to the container path

# Publish container TCP ports on the host via an Incus proxy device (the same
# primitive as [[sockets]], pointed the other way), so agent-started services
# are reachable at localhost:<port> (as of v0.10.1). Loopback-only by default;
# gated behind `coi trust` from an untrusted project ./.coi/config.toml.
# See the Port Publishing wiki page for allocation, preflight, and trust details.
# [ports]
# pool = 3                       # identity-mapped ports per session (0-10):
#                                # host port == container port; exported as COI_PORTS,
#                                # and the sandbox context tells the agent to use them
# [[ports.map]]
# name = "web"                   # unique; exported as COI_PORT_WEB
# container = 3000               # TCP port inside the container
# host = 13000                   # optional exact host pin (omit = auto per workspace/slot)
# listen = "127.0.0.1"           # optional host listen address (default loopback;
#                                # "0.0.0.0" opts into LAN exposure)

# Copy credential FILES from host into the container at session setup (as of
# v0.10.0) — for tools that read credentials from disk rather than an env var
# or a socket. Files are pushed (not mounted), chowned to the container user,
# and optionally chmod'd; a missing host file is skipped with a log line.
# Reference a named bundle from Coi's built-in catalog (the same catalog
# claude/opencode/pi use for their own credentials):
# [[credentials]]
# bundle = "ollama"              # copies ~/.ollama/id_ed25519, mode 0600
# ...or declare an ad-hoc host/container pair for anything not yet cataloged.
# Ad-hoc entries from an untrusted project ./.coi/config.toml are gated
# behind `coi trust` (bundle references are never gated — the host path comes
# from Coi's own catalog, not the referencing config):
# [[credentials]]
# host = "~/.config/some-tool/token.json"            # host file (~ expanded)
# container = "/home/code/.config/some-tool/token.json" # absolute in-container path
# mode = "0600"                  # optional chmod after copy

[limits.cpu]
count = ""           # "2", "0-3", "0,1,3" or "" for unlimited
allowance = ""       # "50%", "25ms/100ms" or "" for unlimited
priority = 0         # 0-10

[limits.memory]
limit = ""           # "512MiB", "2GiB", "50%" or "" for unlimited
enforce = "soft"     # "hard" or "soft"
swap = "true"        # "true", "false", or size like "1GiB"

[limits.disk]
read = ""            # bytes/sec: "10MiB", "1000iops" or "" for unlimited (no "/s" suffix)
write = ""           # bytes/sec: "5MiB" or "" for unlimited (no "/s" suffix)
max = ""             # Combined read+write limit
priority = 0
size = ""            # Rootfs disk quota: "" = unlimited, "20GiB" = cap (needs btrfs/zfs/lvm pool)
tmpfs_size = ""      # "" = disk-backed /tmp (default), "4GiB" = RAM-backed (opt-in)

[limits.runtime]
max_duration = ""    # "2h", "30m" or "" for unlimited
max_processes = 0    # 0 for unlimited
auto_stop = true
stop_graceful = true

[git]
writable_hooks = false  # Allow container to write .git/hooks
# name  = "Jane Dev"                  # pin the commit identity (overrides host git config)
# email = "jane@corp.example"         # ...both name+email required to take effect
# seed_host_identity = true           # default: copy host `git config --global` user.name/email
#                                     # into the container. Set false to keep only the fail-closed guard.
# readonly = true                     # LOCK the identity so the agent cannot commit as anyone
#                                     # else: mounts ~/.gitconfig read-only AND pins GIT_AUTHOR_*/
#                                     # GIT_COMMITTER_* env (beats `git -c user.*`) AND installs a
#                                     # root-owned post-commit hook that re-stamps any commit whose
#                                     # author/committer differs (corrects `--author=`, survives
#                                     # `--no-verify`). Needs a resolvable name/email; default off.
# strip_attribution = true            # default: a global commit-msg hook strips auto-injected AI
#                                     # attribution (Co-Authored-By trailers, "Generated with" footers)
#                                     # from every commit. Set false to keep tool attribution.
# strip_attribution_patterns = ["^X-Bot:"]  # replace the default strip patterns (grep -E, per line)
# protected_branches = ["main", "master"]   # default: the agent can't commit on, or push to,
#                                     # these branches — it works on a feature branch and opens
#                                     # a PR. Set [] to disable. See "Protected branches" below.

[security]
# host_immutable = true        # Apply chattr +i to protected paths (default: true)
# protected_paths = [".git/hooks", ".git/config", ".git/config.worktree", ".git/info/attributes", ".husky", ".vscode", ".coi", ".claude/settings.json", ".claude/settings.local.json"]
# additional_protected_paths = [".idea", "Makefile"]
# disable_protection = false
# reduce_kernel_surface = false        # opt-in kernel attack-surface hardening: deny the syscall
#                                      # families behind recent kernel escape chains (io_uring, bpf,
#                                      # userfaultfd, keyring) via security.syscalls.deny, and turn
#                                      # off Docker/nesting. Denied syscalls return EPERM. Default off;
#                                      # enabled by the built-in "hardened" profile. Trusted scope only.
# reduce_kernel_surface_strict = false # STRICT tier: also deny perf_event_open (a kernel-LPE vector).
#                                      # Implies reduce_kernel_surface. Removes kernel profiling (perf,
#                                      # async-profiler) but degrades gracefully. Trusted scope only.

[timezone]
mode = "host"              # "host" (inherit from host, default), "fixed", or "utc"
# name = "Europe/Warsaw"   # IANA timezone name, only used when mode = "fixed"

[monitoring]
# enabled = false                  # Enable background security monitoring
# auto_pause_on_high = true        # Pause container on high-severity threats
# auto_kill_on_critical = true     # Kill container on critical threats
# reverse_shell_one_liners = "critical"  # python -c/perl -e/ruby -e/php -r class: "critical" | "warn" | "off"
# poll_interval_sec = 2
# file_read_threshold_mb = 50.0    # Alert if >N MB read in poll interval
# file_read_rate_mb_per_sec = 10.0 # Alert on sustained read rate
# audit_log_retention_days = 30

[monitoring.nft]
# enabled = false                  # Enable nftables network monitoring
# rate_limit_per_second = 100
# dns_query_threshold = 100        # Alert if >N DNS queries/min
# log_dns_queries = true
# lima_host = ""                   # Lima host for macOS (e.g., "lima-default")

# === Profiles ===
# Profiles are self-contained directories under profiles/.
# See the Profiles wiki page for full documentation.
#
# Directory structure:
#   .coi/profiles/rust-dev/config.toml
#   .coi/profiles/python-ml/config.toml
#
# Use: coi shell --profile rust-dev
# List: coi profile list
# Show: coi profile info rust-dev
```

## Environment Variables

There is no env-var config layer (as of v0.10.0). The legacy `CLAUDE_ON_INCUS_*` and `COI_LIMIT_*` overrides were removed entirely — a stray value in a shell profile could silently defeat explicit config. Their replacements are ordinary config keys:

| Removed variable | Config replacement |
|------------------|--------------------|
| `CLAUDE_ON_INCUS_IMAGE` | `[container] image` |
| `CLAUDE_ON_INCUS_PERSISTENT` | `[container] persistent` |
| `CLAUDE_ON_INCUS_SESSIONS_DIR` / `_STORAGE_DIR` | `[paths] sessions_dir` / `storage_dir` |
| `COI_LIMIT_CPU` / `_CPU_ALLOWANCE` | `[limits.cpu] count` / `allowance` |
| `COI_LIMIT_MEMORY` / `_MEMORY_SWAP` | `[limits.memory] limit` / `swap` |
| `COI_LIMIT_DISK_READ` / `_DISK_WRITE` | `[limits.disk] read` / `write` |
| `COI_LIMIT_DURATION` | `[limits.runtime] max_duration` |

`COI_CONFIG` still exists — it selects which user config file to load instead of `~/.coi/config.toml`; it does not configure anything itself.

Two diagnostic env vars also exist and, like `COI_CONFIG`, never configure anything — they only observe: `COI_TIMING_DEBUG=1` prints a startup timing breakdown to stderr at exit, and `COI_TIMING_DEBUG_JSON=<path>` writes the same data as JSON. See [Profiling Slow Startup](Troubleshooting#profiling-slow-startup) in Troubleshooting.

## CLI Flags

Only operational, per-invocation flags remain on the CLI (as of v0.10.0). Everything config-shaped — image, persistence, tool selection, tmux, network, mounts, environment, SSH, monitoring, timezone, resource limits — lives exclusively in config files or [profiles](Profiles). Using a removed flag fails with a migration hint naming the exact replacement key.

**Operational flags (global):**

```bash
coi shell \
  --workspace /path/to/project \
  --slot 2 \
  --resume \
  --profile rust-dev
```

**Shell-only flags:**

```bash
coi shell --debug          # Launch bash instead of AI tool (for debugging)
coi shell --background     # Run in background
coi shell --container NAME # Target specific container
```

**Removed flags and their config keys:**

| Removed flag | Config replacement |
|--------------|--------------------|
| `--image` | `[container] image = "..."` |
| `--persistent` | `[container] persistent = true` |
| `--tmux` / `--tmux=false` | `[shell] use_tmux = true/false` |
| `--tool <name>` | `[tool] name = "..."` (a per-tool [profile](Profiles) also carries the tool's whole setup) |
| `coi build --compression` | `[container.build] compression = "..."` |
| `coi shutdown --timeout` | `[container] shutdown_timeout = <seconds>` |

## Sandbox Context File

Coi automatically injects a `~/SANDBOX_CONTEXT.md` file into every container describing the sandbox environment, and by default injects it into each tool's native context-loading mechanism (`auto_context = true`).

See the [Sandbox Context](Sandbox-Context) wiki page for full details on auto-context injection, per-tool behavior, disabling it, and using a custom (or JSON) context file.

## Mounting Additional Files

Use `[[mounts.default]]` to bring additional host directories into the container. Either TOML shape works in either scope: the nested `[[mounts.default]]` form shown here and the flat `[[mounts]]` form (same `host`/`container`/`readonly`/`shift` keys) are both accepted in top-level config **and** in a profile's `config.toml`, so a mount block can be copied between them verbatim.

Each mount entry accepts these fields:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `host` | string | *(required)* | Host path to mount (supports `~` expansion) |
| `container` | string | *(required)* | Absolute path inside the container |
| `readonly` | bool | `false` | Mount the path read-only |
| `shift` | bool | *(inherit)* | Override UID/GID shifting for this mount (see [Per-mount UID/GID shifting](#per-mount-uidgid-shifting) below) |

> **Warning:** Always mount subdirectories, not the parent config directory. Mounting `~/.claude` or `~/.config/opencode` directly will conflict with Coi's own config injection and can silently overwrite your credentials and settings. Mount `~/.claude/skills` or `~/.config/opencode/agents` instead.

```toml
# Mount specific subdirectories - NOT the parent config dir
[[mounts.default]]
host = "~/.config/opencode/agents"
container = "/home/code/.config/opencode/agents"
readonly = true
```

**Mounting Claude Code skills, commands, and plugins:**

Coi copies essential Claude config files (credentials, settings) automatically, but custom subdirectories like `skills`, `commands`, and `plugins` are not included by default. Mount them individually as read-only for security:

```toml
[[mounts.default]]
host = "~/.claude/skills"
container = "/home/code/.claude/skills"
readonly = true

[[mounts.default]]
host = "~/.claude/commands"
container = "/home/code/.claude/commands"
readonly = true

[[mounts.default]]
host = "~/.claude/plugins"
container = "/home/code/.claude/plugins"
readonly = true
```

**Important:** Always use `readonly = true` for these mounts to prevent the AI tool from modifying your host configuration. See issue [#260](https://github.com/coipond/coi/issues/260) for background on this pattern.

**Common use cases:**

| What to mount | Host path | Container path |
|---------------|-----------|----------------|
| opencode custom agents | `~/.config/opencode/agents` | `/home/code/.config/opencode/agents` |
| opencode global AGENTS.md | `~/.config/opencode/AGENTS.md` | `/home/code/.config/opencode/AGENTS.md` |
| Shared coding standards | `~/my-standards` | `/home/code/standards` |

Do not mount the entire `~/.config/opencode` or `~/.claude` directory - Coi manages those automatically and mounting over them will cause config files to be wiped on both host and container.

### Per-Mount UID/GID Shifting

By default Coi decides UID/GID shifting once per session and applies the same decision to the workspace and every configured mount, so files usually appear owned by the container's `code` user. On some setups a mount can instead show up owned by `nobody:nogroup` (UID 65534) — readable and executable, but not writable by `code`. This happens when the mount is attached without idmap shifting, e.g. a writable directory a downstream tool expects to write into:

```text
$ stat /coipond
Access: (0775/drwxrwxr-x)  Uid: (65534/nobody)   Gid: (65534/nogroup)
$ date +%s > /coipond/stops.log
/bin/sh: cannot create /coipond/stops.log: Permission denied
```

The `shift` field overrides the shifting decision for a single mount, mapping directly to `incus config device add … shift=<value>`:

```toml
[[mounts.default]]
host = "/home/user/some/host/path"
container = "/some/container/path"
shift = true      # force an idmapped mount → host files show up owned by `code`
```

- **`shift = true`** — force an idmapped mount so host files appear owned by the container `code` user instead of `nobody:nogroup`.
- **`shift = false`** — opt this mount out of shifting.
- **unset** (default) — inherit the session-wide decision the workspace uses. Leave it unset unless you have a specific reason; the default is correct for the vast majority of mounts.

**Interaction with `raw.idmap` / `disable_shift`:** `shift` and `raw.idmap` are mutually exclusive (Incus rejects the combination). When Coi maps UIDs via `raw.idmap` instead of shifting — a host/container UID mismatch, a Colima/Lima guest that maps UIDs itself, `disable_shift = true`, or a source filesystem that cannot do idmapped mounts (FUSE-family, 9p) — the whole container is already remapped, so a per-mount `shift = true` is unnecessary and is ignored, with a warning. In those environments the mount is already writable by `code` without any override.

## Replacing the Tool's Executable (`binary`)

`[tool] binary` makes `coi shell` (and `coi run --prompt`) launch a different executable in place of the tool's default (`claude`, `codex`, `opencode`, `pi`, `omp`). The executable receives the tool's usual arguments, so a wrapper script can do its own setup and then hand off:

```toml
[tool]
binary = "/workspace/scripts/claude-wrapper.sh"
```

```sh
#!/bin/sh
# claude-wrapper.sh — runs in place of `claude`, with claude's arguments
exec claude "$@"
```

The value must be a plain command name or path (letters, digits and `._:/@+-`; no spaces, quotes or shell syntax) — it is placed directly on the launch command line. An invalid value stops the launch with an error rather than being ignored. The executable must exist inside the container: in the workspace (`/workspace/...`), a [mounted directory](#mounting-additional-files), or the image.

## Running Commands Before the Agent (`pre_launch`)

`[tool] pre_launch` runs commands inside the container, in order, every time `coi shell` starts the tool — for example to keep the agent current without rebuilding the image:

```toml
# ~/.coi/config.toml (or a profile under ~/.coi/profiles)
[tool]
pre_launch = ["claude update"]
```

- Each entry is a full shell command, so it can also run a script with arguments: `pre_launch = ["/workspace/scripts/before-agent.sh --quiet"]`.
- Commands run as the container user, in the workspace, with the session's environment. Their output shows in the session.
- Each command gets **5 minutes**; after that it — and anything it started — is stopped. A failing or timed-out command is reported and the next one runs; the tool **always starts**. Ctrl+C skips the remaining commands.
- Commands get no terminal input, so they must not prompt.
- It applies to `coi shell` (new sessions, re-launching into an existing tmux session, and `use_tmux = false`), not to headless `coi run`.
- **Trusted-scope only.** Commands that run automatically at every session start are honored from `~/.coi/config.toml`, `$COI_CONFIG` and profiles under `~/.coi/profiles` — never from a project's `.coi/config.toml` or a project profile, where they are ignored with a warning (a cloned repo can't switch them on).

`pre_launch` and `binary` combine: the `pre_launch` commands finish first, then `binary` (or the default tool) starts.

## Protected Branches

By default the agent can't commit on, or push to, `main` or `master` — it works on a feature branch and opens a pull request instead. coi enforces this with root-owned git hooks in the container (the agent can't edit them):

```toml
[git]
protected_branches = ["main", "master", "release"]   # replace the list
# protected_branches = []                            # disable the guard
```

- Refused on a protected branch: commits and merge commits, and moving the branch to a commit the remote doesn't already have — cherry-pick, revert, `git am`, fast-forward merge, rebase, `reset`, `update-ref`, `branch -f`, or deleting and recreating it. Pushes whose destination is a protected branch are refused too.
- Still allowed: `git pull` and `git reset --hard origin/main` (commits the remote already has), deleting the branch, the first commit of a brand-new repository, and pushes **into** a local bare repository.
- `git fetch origin main:main` is refused (git moves `main` before `origin/main`, so the guard can't tell it from a local commit); use `git fetch origin && git branch -f main origin/main`, which the refusal message suggests.
- Only your own config (`~/.coi/config.toml`, `$COI_CONFIG`) can change or disable the list; a project's `.coi/config.toml` can't, and the default stays on.
- The hooks guard against accidental commits, not a determined workaround: `git commit --no-verify`, a repository-local `core.hooksPath` (e.g. husky), `git branch -M`/`-C`, `git symbolic-ref`, or writing a commit under `refs/remotes/` first get around the local checks. Server-side branch protection is the real backstop.

## Profiles

Profiles are self-contained directories that bundle image, tool, limits, mounts, build scripts, context files, and environment into named templates. Use `coi shell --profile NAME` to activate one.

See the [Profiles](Profiles) wiki page for complete documentation including config reference, inheritance, context files, build scripts, and the `create`/`edit`/`delete` commands.

## See Also

- [Profiles](Profiles) - Reusable config bundles that extend per-project settings
- [Network Isolation](Network-Isolation) - Configuring restricted, allowlist, and open network modes
- [Supported Tools](Supported-Tools) - Tool-specific configuration options
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How containers and sessions use your config
- [Security Best Practices](Security-Best-Practices) - Recommended security settings
