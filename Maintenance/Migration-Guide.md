# Migration Guide

This page documents breaking changes between Coi versions and how to migrate. Check this page after a `coi update` if the release notes mention configuration or behavior changes.

# Upgrading from 0.11 to 0.12

0.12.0 is additive — no breaking changes and no config migration required. Existing configs and profiles keep working as-is; everything below is opt-in.

```bash
coi update          # get the 0.12.0 binary
coi version         # confirm 0.12.0
```

## New in 0.12 (Opt-in, No Action Needed)

- **Switch AI tools in one persistent container** — point two profiles at the same `[container] session_name` (both `persistent = true`) and each re-enters that same box under its own `[tool] name`; the new tool's credentials are seeded on first switch. See [Running a different AI tool in the same container](Container-Lifecycle-and-Sessions#running-a-different-ai-tool-in-the-same-container-v012).
- **Headless "fire and forget" prompt runs** — `coi run --prompt` / `--prompt-file` / `--prompt-name` runs the agent to completion on a predefined prompt and exits with its status code; register reusable prompts in a `[prompts]` table. See [Headless Orchestration](Headless-Orchestration#fire-and-forget-prompt-runs-coi-run---prompt).
- **Config/profile symmetry** — a `[mounts]` block works in both the flat `[[mounts]]` and nested `[[mounts.default]]` form in either top-level config or a profile (copy it between them verbatim), and `env_command_timeout` is now settable at a [profile](Profiles)'s root.
- **[`coi tool spec`](Headless-Orchestration)** — print a tool's launch command + env for an external orchestrator to run (with `--continue` / `--resume-id` / `--resume`).
- **[`coi top`](Resource-Usage)** — live per-container and per-process CPU / memory / disk / network usage.
- **New tools**: [Codex CLI](Supported-Tools#codex-cli) (`[tool] name = "codex"`) and [omp (Oh My Pi)](Supported-Tools#omp-oh-my-pi) (`[tool] name = "omp"`) — both opt-in at image build via `[container.build] agents`.
- **[`~/SANDBOX_CONTEXT.json`](Sandbox-Context)** — structured JSON companion to the Markdown context (`[tool] context_json = true`).
- **Egress hardening** ([Network Isolation](Network-Isolation)): per-host `ports` on [`[[network.hosts]]`](Static-Host-Entries), per-destination `:ports` in `allowed_domains`, `[network] dns_servers`, and `[network] allowed_ports`.
- **[`[git] readonly = true`](Configuration)** — lock the container's git commit identity read-only.

## Notes

- **`[tool.*]` model/effort now reaches an external `coi container exec` and reused containers (#744).** Previously a profile's `[tool.claude] model` / `effort_level` only landed in the container's `~/.claude/settings.json` (written on fresh setup only), so an orchestrator preparing a container with `coi shell --background` and then running the tool via a separate `coi container exec`, or any reused/persistent container, silently fell back to the tool's default. The resolved tool env is now persisted as container-level Incus config on every setup. No action needed — per-workflow model selection now works.
- **`[limits.disk] tmpfs_size` now actually resizes `/tmp` (#733)** — if you set it before and saw no effect, it takes effect on 0.12 (via a systemd `tmp.mount` unit). See [Resource and Time Limits](Resource-and-Time-Limits).
- **`coi shell` now honors `[container] storage_pool` (#726)** — previously only `coi run`/`coi build` did. If you configured a pool, interactive sessions now land on it (not the Incus default); no config change needed.

# Upgrading from 0.10.1 to 0.11.0

0.11.0 has one breaking change: the `model` setting moved from the config root / `[defaults]` into a `[tool.claude]` table — and, for the first time, is actually wired. Previously `model` was a Claude-specific value stranded at the config root: it was stored, merged, and printed by `coi profile`, but never reached the launched tool. It now lives alongside its sibling `effort_level` under `[tool.claude]`, and a configured value is delivered to Claude Code as the `ANTHROPIC_MODEL` env var.

```bash
coi update          # get the 0.11.0 binary
coi version         # confirm 0.11.0
```

## Action May Be Required

### Move `model` into `[tool.claude]`

If you set `model` at the config root or under `[defaults]` (in `~/.coi/config.toml`, a project `.coi/config.toml`, or a profile's `config.toml`), move it under `[tool.claude]`:

```toml
# Before (no longer honored)
[defaults]
model = "opus"

# After
[tool.claude]
model = "opus"          # alias like "opus", or a full ID like "claude-opus-4-8"
```

The old location is no longer accepted: a profile `config.toml` with a root `model` now fails schema validation, and a global `[defaults] model` is silently ignored. The value is passed through to Claude Code verbatim as `ANTHROPIC_MODEL`.

## Notes

- The default profile no longer pins a model, so if you never set one, behavior is unchanged — Claude Code uses its own default.
- Only Claude is wired to this setting; opencode is unaffected.

# Upgrading from 0.9 to 0.10

0.10 completes a single principle: the configuration hierarchy is exactly `defaults → user config ($COI_CONFIG) → project config → profile` — nothing else. All config-shaped CLI flags and both legacy env-var config layers are gone. Using a removed flag fails with a migration hint naming the exact replacement key, so `coi update` + running your usual command tells you precisely what to move where.

```bash
coi update          # get the 0.10 binary
coi version         # confirm 0.10.x
coi health --verbose
```

## Action May Be Required

### 1. Config-Shaped CLI Flags Were Removed

| Removed flag | Config replacement (global, per-project, or per-profile) |
|--------------|----------------------------------------------------------|
| `--image` | `[container] image = "..."` |
| `--persistent` | `[container] persistent = true` |
| `--tmux` / `--tmux=false` | `[shell] use_tmux = true/false` |
| `--tool <name>` | `[tool] name = "..."` (a per-tool [profile](Profiles) carries the tool's whole setup) |
| `coi build --compression` | `[container.build] compression = "..."` |
| `coi shutdown --timeout` | `[container] shutdown_timeout = <seconds>` |
| `coi profile create --image/--persistent` | edit the created profile's `config.toml` (`coi profile edit <name>`); only `--inherits` still seeds the scaffold |

Only operational flags remain: `--workspace`, `--slot`, `--resume`/`--continue`, `--profile` (plus `coi shell`'s `--debug`, `--background`, `--container`). `coi image publish --compression` stays — it is raw plumbing.

### 2. Env-var Config Overrides Were Removed Entirely

`CLAUDE_ON_INCUS_IMAGE` / `_PERSISTENT` / `_SESSIONS_DIR` / `_STORAGE_DIR` and every `COI_LIMIT_*` variable are now ignored. If any live in your shell profile or CI, move them to config keys — see the table on the [Configuration](Configuration#environment-variables) page. `$COI_CONFIG` still selects the user config file; it does not configure anything itself.

### 3. The `claude-on-incus` Name Is Fully Retired

The installer and Makefile no longer create the `claude-on-incus` symlink, and the installer removes a leftover one when upgrading. An existing old symlink still executes (it identifies as `coi`), but scripts should call `coi`.

### 4. Resume Can Now Convert a Session's Persistence

Previously `coi shell --resume` inherited the resumed session's persistence unconditionally. Now an explicitly configured `[container] persistent` wins over the session metadata; when unset, the saved mode is inherited as before. If you relied on resume preserving ephemerality despite a `persistent = true` config, remove that config for the workspace.

### 5. An Explicit `--profile` Now Beats the Workspace's Project Config

`coi run/shell --profile X` re-applies the profile's `[container]` settings after the workspace overlay, so a cloned repo's `.coi/config.toml` can no longer silently override the profile you selected. A project config that enables persistence now prints a visible note.

## New in 0.10 (Opt-in, No Action Needed)

- **`coi run` workspace script** — an executable `coi-run` at the workspace root runs in the sandbox when you type `coi run` with no command; output streams live and piped stdin works.
- **`[[credentials]]`** — copy credential files (catalog bundles like `bundle = "ollama"`, or ad-hoc host/container pairs) into containers; ad-hoc entries from untrusted project configs are gated behind `coi trust`. See [Configuration](Configuration) and [Supported Tools](Supported-Tools#tool-credentials-and-third-party-providers).
- **Built-in `hardened` profile** — one-flag safe-open for untrusted repos: `coi shell --profile hardened`. See [Profiles](Profiles).
- **`[network] use_sudo = false`** — non-sudoers mode; restricted/allowlist fail closed instead of silently downgrading. See [Network Isolation](Network-Isolation#running-without-sudo-use_sudo--false).
- **`[container] shutdown_timeout` / `ready_timeout`** — graceful-shutdown and container-readiness windows as config policy.
- **`coi list --running / --stopped / --status <state>`** — status filters over the container listing.
- **`coi health` runtime proofs** — secret masking, host-credential isolation, and metadata-endpoint blocking are now verified with real probe containers.

# Upgrading from 0.8 to 0.9

0.9 is primarily a security-hardening release. Most changes are transparent after `coi update`, but a few tighten what an untrusted project config can do — if you rely on a project `.coi/config.toml` (especially one that mounts host paths or relaxes the network), read the "Action May Be Required" items below.

```bash
coi update          # get the 0.9 binary
coi version         # confirm 0.9.x
coi health --verbose
```

> **One-line summary:** a project's own `.coi/config.toml` (which a cloned repo can ship) is now treated as untrusted. It can no longer silently mount host paths outside the workspace, forward host sockets, run host commands, or weaken network isolation. Your own `~/.coi/config.toml` (and anything pointed to by `$COI_CONFIG`) is trusted and unaffected.

## Action May Be Required

### 1. Out-of-Workspace Mounts From a Project Config Now Need `coi trust`

If a project `.coi/config.toml` (or a project-scoped profile under `./.coi/profiles/`) declares a mount whose host path resolves outside the workspace, it is now ignored at launch until you approve it:

```bash
coi trust            # approve this workspace's out-of-workspace mounts/sockets
coi trust --list     # see what's approved
coi untrust          # revoke
```

- Approval is pinned to a fingerprint of the approved mounts/sockets — if they change, you must re-approve.
- In-workspace mounts, and all mounts from your trusted `~/.coi/config.toml` / `$COI_CONFIG`, are never gated.
- **CI / automation:** set `COI_TRUST_ALL=1` to bypass the gate (safe — only the invoking shell can set it, not a cloned repo).
- Building a config in a temp dir? Point at it with `$COI_CONFIG` (trusted); do not aim `$COI_CONFIG` at an untrusted repo's own config.

### 2. Project Config Can No Longer Weaken Network Isolation

Security-downgrading network settings in a project `.coi/config.toml` are now refused (with a warning): `network.mode = "open"`, `block_private_networks = false`, `block_metadata_endpoint = false`, `allow_local_network_access = true`. Move them to `~/.coi/config.toml` or `$COI_CONFIG` if you genuinely want them. Strengthening values from a project config are still honored.

### 3. Workspace `.coi/` Is Read-Only Inside the Container

`<workspace>/.coi/` is now mounted read-only (and added to default `protected_paths`), so an in-container agent cannot plant a config/profile that would apply on the next launch. Nothing legitimate writes there at runtime, but if you had tooling that did, move it out.

### 4. More git Paths Protected Read-Only

`.git/config.worktree` and `.git/info/attributes` are now in the default `protected_paths` (alongside `.git/hooks`, `.git/config`, `.husky`, `.vscode`, `.coi`) to close git filter/attribute code-execution sinks. Repos that do not write these at runtime are unaffected.

### 5. Allowlist Mode Tightened to TCP/UDP + Rate-Limited ICMP

In `network.mode = "allowlist"`, egress to an allowed host is now limited to TCP/UDP plus rate-limited ICMP echo; raw-IP / other protocols fall through to deny. HTTPS/git/npm/DNS/QUIC and `ping` still work — only unusual non-TCP/UDP egress to allowed hosts is affected. `restricted` and `open` modes are unchanged.

### 6. Stricter IPv6 / Boot-Time Network Enforcement

IPv6 egress is now blocked host-side (previously a container-reversible sysctl), the boot-time network block fails closed in restricted/allowlist mode, and bridge NIC anti-spoofing + port isolation block container-to-container and host-gateway traffic. These only surface if something relied on the previous gaps; legitimate egress is unaffected.

## New in 0.9 (Opt-in, No Action Needed)

- **`[[sockets]]`** — forward arbitrary host Unix sockets into the container (credential-broker building block). See [Configuration](Configuration); gated by `coi trust`.
- **`[defaults.env_commands]`** — mint short-lived secrets by running a host command at session start and injecting its stdout as an env var. **Trusted-scope config only.** See [Configuration](Configuration).
- **`coi trust` / `coi untrust` / `coi trust --list`** — manage approvals for out-of-workspace mounts and forwarded sockets.
- **`coi audit`** — stream the structured (JSON Lines) threat-event audit log for a session. See [Audit Log](Audit-Log).
- **pi tool support** — `coi shell --tool pi` (in 0.10+ the flag is gone: set `[tool] name = "pi"` instead). See [Supported Tools](Supported-Tools).
- **Stronger monitoring** — host-side fanotify file monitoring, Sigma + runtime-loadable exec-pattern detection (`coi update patterns`), fork-bomb / spawn-rate detection, UID-namespace isolation per container.
- **`coi profile create default`** scaffolds a documented starter config.

## Notes

- The trust store lives at `~/.coi/trust.toml` and is created on first `coi trust`. Inside a container `~/.coi` is read-only, so an agent cannot forge approvals.
- No config-file format changes are required moving from 0.8 to 0.9.

## Config File Location: `.coi.toml` → `.coi/config.toml`

In earlier versions of Coi, the project config file was `.coi.toml` at the project root. This has been replaced by `.coi/config.toml` in a directory.

**Before:**

```text
your-project/
├── .coi.toml       ← old location
└── ...
```

**After:**

```text
your-project/
├── .coi/
│   └── config.toml ← new location
└── ...
```

### Migration Steps

1. Create the `.coi/` directory in your project root:

```bash
mkdir -p .coi
```

2. Move the old config file:

```bash
mv .coi.toml .coi/config.toml
```

3. Update your `.gitignore` if needed — the directory form is typically still excluded:

```gitignore
# If you were ignoring the old file, update to the directory
.coi/
```

4. Profile directories also move: `~/.coi-profiles/` → `~/.coi/profiles/` (user-level) and `.coi-profiles/` → `.coi/profiles/` (project-level). Move your profile directories accordingly.

Coi warns on startup if it detects a `.coi.toml` at the project root and no `.coi/config.toml`, so you will not silently lose configuration.

## Mount Syntax: `[[mounts]]` vs `[[mounts.default]]`

The mount table key differs between the main config and profile configs:

| Context | Correct key |
|---------|-------------|
| `~/.coi/config.toml` or `.coi/config.toml` | `[[mounts.default]]` |
| `.coi/profiles/NAME/config.toml` | `[[mounts]]` |

**Main config:**

```toml
# .coi/config.toml
[[mounts.default]]
host = "~/.claude/skills"
container = "/home/code/.claude/skills"
readonly = true
```

**Profile config:**

```toml
# .coi/profiles/rust-dev/config.toml
[[mounts]]
host = "~/.cargo"
container = "/home/code/.cargo"
```

Using `[[mounts]]` in the main config or `[[mounts.default]]` in a profile config results in the mounts being silently ignored. If extra files are not appearing in the container, check the mount key.

## Checking Your Current Version

```bash
coi version
```

Compare against the [releases page](https://github.com/coipond/coi/releases) to identify which migrations apply to your upgrade path.

## Boolean Config Fields Now Use Pointer Semantics

**Affected versions:** All versions before this fix was introduced.

**Symptom:** Security settings like `block_private_networks`, `auto_pause_on_high`, or `auto_kill_on_critical` appeared disabled even though they were set to `true` in a lower-priority config file.

**Cause:** The multi-layer config merge (global → project → profile → CLI) used plain `bool` fields. When a higher-priority config omitted a boolean field, it defaulted to `false` and silently overwrote the `true` value from a lower-priority file.

**Fix:** 13 boolean config fields were converted to pointer types (`*bool`). Omitted fields are now `nil` (no override) rather than `false`. Update Coi and verify your security settings are applied:

```bash
coi update
coi health --verbose
# Check the Monitoring section for auto_pause=true and other security settings
```

No config changes needed — the fix is transparent once you update the binary.

## Claude Settings.json Deep Merge

**Symptom:** `~/.claude/settings.json` was overwritten with Coi's sandbox/bypass settings after `coi shell`. Custom settings (AWS Bedrock credentials, `env` variables, `allowedTools`) were lost.

**Fix:** Coi now performs a deep merge rather than replacing the file. Existing fields are preserved; Coi only adds the sandbox permissions it needs. If your settings were already overwritten, recreate them — going forward the merge is safe.

## Docker Compose Support: Three-Step Launch

**Symptom:** `docker compose up` failed inside containers started with `coi shell`. Docker itself worked but Compose errored out.

**Cause:** A race condition where `incus launch` started the container before Docker support flags (`security.nesting`, `security.syscalls.intercept.mknod`, `security.syscalls.intercept.setxattr`) were applied.

**Fix:** Container launch now uses a three-step sequence: `incus init` → apply flags → `incus start`. Update Coi to get this fix; no config changes needed.

## Session Save: Cross-Device Link (EXDEV) Error

**Symptom:** Session save failed with an `EXDEV` (cross-device link) error when `/tmp` and the session storage directory were on different filesystems.

**Fix:** Coi now falls back to a recursive copy when `os.Rename` fails with `EXDEV`. Update Coi to get this fix.

## UID/GID Remapping for Non-Default code_uid

**Symptom:** `I have no name!` shell prompt, or `Permission denied` errors on `/home/code` files, when `code_uid` in config was set to a value other than 1000.

**Fix:** Coi now automatically remaps the container user's UID/GID during session setup when `code_uid` differs from the image default. Update Coi. If you have an existing persistent container with broken ownership, fix manually:

```bash
incus exec <container-name> -- usermod -u <your-uid> code
incus exec <container-name> -- groupmod -g <your-uid> code
incus exec <container-name> -- chown -R <your-uid>:<your-uid> /home/code
```

## See Also

- [Configuration](Configuration) - Full configuration reference with all current options
- [Profiles](Profiles) - Profile directory structure and config reference
- [System Health Check](System-Health-Check) - Verify your environment after migrating
