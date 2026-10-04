# Profiles

Profiles are reusable, self-contained container configurations. Each profile bundles image, tool settings, limits, mounts, build scripts, environment variables, network config, and optional context files into a named directory.

## Directory Structure

Each profile is a directory under `profiles/` containing a `config.toml` and optional supporting files:

```text
.coi/
├── config.toml              # project config
└── profiles/
    ├── rust-dev/
    │   ├── config.toml      # profile config
    │   ├── build.sh         # profile-specific build script
    │   └── CONTEXT.md       # AI agent context (appended to sandbox context)
    └── python-ml/
        ├── config.toml
        └── setup.sh
```

Profile directories are scanned at two config levels:

| Priority | Location |
|----------|----------|
| 1 (lowest) | `~/.coi/profiles/NAME/config.toml` (user) |
| 2 (highest) | `./.coi/profiles/NAME/config.toml` (project) |

Profiles from all discovered locations are merged into a single namespace. If the same profile name is defined in more than one location, Coi refuses to start and asks you to rename one so it is always unambiguous which profile is being applied. Operational CLI flags (`--slot`, `--workspace`, `--resume`) are per-invocation choices, not configuration, and never conflict with profile settings.

## Profile Config Reference

A profile `config.toml` uses the same sections as the main config. All fields are optional - only set what you want to override.

```toml
# .coi/profiles/rust-dev/config.toml

context = "CONTEXT.md"                    # context file (see below)
forward_env = ["CARGO_REGISTRY_TOKEN"]

[container]
image = "coi-rust"
persistent = true

[container.build]
base = "coi-default"
script = "build.sh"           # resolved relative to this config.toml

[environment]
RUST_BACKTRACE = "1"

[tool]
name = "claude"
permission_mode = "bypass"

[tool.claude]
effort_level = "high"

[[mounts]]
host = "~/.cargo"
container = "/home/code/.cargo"

[network]
mode = "restricted"
# allowed_domains = ["crates.io", "github.com"]

[limits.cpu]
count = "4"

[limits.memory]
limit = "4GiB"

[limits.runtime]
max_duration = "4h"
```

### Available Fields

| Field | Type | Description |
|-------|------|-------------|
| `context` | string | Path to context file (see [Context Files](#context-files)) |
| `forward_env` | string[] | Host env vars to forward into the container |
| `[env_commands]` | map | Env vars minted from host commands at session start (`VAR = "~/bin/mint.sh"`). **Trusted-scope only** — stripped from an untrusted project profile |
| `env_command_timeout` | string | Profile-root duration bounding each `[env_commands]` invocation (e.g. `"5s"`; default `30s`). **Trusted-scope only**. Echoed by `coi profile info` |
| `[container]` | section | Container settings (`image`, `persistent`, `storage_pool`, `alias`, `session_name` — see [Named Sessions](Container-Lifecycle-and-Sessions#named-sessions-session_name--v0111), `shutdown_timeout`, `ready_timeout`) |
| `[container.build]` | section | Custom image build (`base`, `script`, `commands`) |
| `[environment]` | map | Static environment variables |
| `[tool]` | section | AI tool config (`name`, `binary`, `permission_mode`, `context_file`, `auto_context`, `pre_launch`). `context_file`, `context_json_file` and `pre_launch` are **trusted-scope only** — stripped (with a warning) from a project profile, since they read host files into the container or run commands at every session start |
| `[tool.claude]` | section | Claude-specific settings (`model`, `effort_level`); `model` is delivered to Claude Code as `ANTHROPIC_MODEL` |
| `[[mounts]]` / `[[mounts.default]]` | array | Additional mount points (`host`, `container`, `readonly`). Both the flat `[[mounts]]` form and the nested `[[mounts.default]]` form are accepted here and in the main config, so a mount block reads identically in either scope — you can copy it between a profile and your top-level config verbatim |
| `[[credentials]]` | array | Credential files to copy into the container (as of v0.10.0): `bundle = "<catalog name>"`, or ad-hoc `host`/`container`/`mode`. See [Configuration](Configuration) for the trust model |
| `[network]` | section | Network isolation (`mode`, `allowed_domains`) |
| `[limits.cpu]` | section | CPU limits (`count`, `allowance`, `priority`) |
| `[limits.memory]` | section | Memory limits (`limit`, `enforce`, `swap`) |
| `[limits.disk]` | section | Disk limits: IO rates (`read`, `write`, `max`), root disk `size`, `tmpfs_size` |
| `[limits.runtime]` | section | Runtime limits (`max_duration`, `max_processes`) |
| `inherits` | string | Parent profile name for inheritance (see [Inheritance](#profile-inheritance)) |
| `[paths]` | section | Path overrides (`sessions_dir`, `storage_dir`, `logs_dir`, `preserve_workspace_path`) |
| `[incus]` | section | Incus settings (`project`, `group`, `code_uid`, `code_user`) |
| `[git]` | section | Git settings (`writable_hooks`, `protected_branches`, …). `protected_branches` is **trusted-scope only** — ignored from a project profile |
| `[ssh]` | section | SSH settings (`forward_agent`) |
| `[security]` | section | Security settings (`host_immutable`, `protected_paths`) |
| `[monitoring]` | section | Security monitoring settings |
| `[timezone]` | section | Timezone settings (`mode`, `name`) |

## Profile Inheritance

Profiles can inherit from a parent using `inherits = "parent-name"`,
so you only override what differs:

```toml
# .coi/profiles/rust-dev-nightly/config.toml
inherits = "rust-dev"

[container]
image = "coi-rust-nightly"

[environment]
RUST_CHANNEL = "nightly"
```

**Merge strategy:**

- Environment maps deep-merge (child keys win; set to `""` to clear a parent key)
- Arrays (`mounts`, `forward_env`) fully replace if the child defines them
- Scalars override if set by the child
- Struct sections (`limits`, `tool`, `build`, `network`) deep-merge field by field

Inheritance works across config levels (a project profile can inherit from a
user-level profile), supports chains up to 10 levels, and has cycle detection.
The built-in `default` profile can be used as a parent via `inherits = "default"`.

## Context Files

Profiles can include a context file - a markdown file with AI-agent-specific instructions that gets automatically appended to the sandbox context when the profile is used.

```toml
# .coi/profiles/python-ml/config.toml
context = "CONTEXT.md"
```

```markdown
<!-- .coi/profiles/python-ml/CONTEXT.md -->
## Python ML Project Guidelines

- Use pytest for all testing
- Follow PEP 8 style conventions
- Use type hints for all function signatures
- Prefer numpy vectorized operations over loops
```

### How It Works

When `coi shell --profile python-ml` is used:

1. The context file path is resolved relative to the profile directory (absolute and `~` paths also work)
2. The file is validated - if it does not exist, the session fails with a clear error
3. The content is appended to `~/SANDBOX_CONTEXT.md` under a `# User-Provided Profile Context` heading
4. The content is also rendered into tool-native auto-context files (e.g., `~/.claude/CLAUDE.md` for Claude Code) as part of Coi's single marker-delimited managed block, which is replaced — not appended — each session

This means the AI agent automatically receives your profile-specific instructions on top of the standard sandbox environment info - no manual setup needed.

### Naming

The context file can be named anything (not just `CONTEXT.md`). The `context` field needs to point to a valid file:

```toml
context = "instructions.md"
context = "AI_GUIDELINES.md"
context = "../shared/common-context.md"    # relative paths work
context = "~/global-context.md"            # tilde expansion works
```

## Build Scripts

Profiles can specify a build script that runs on top of a base image to create a custom container image:

```toml
[container.build]
base = "coi-default"              # base image to build on
script = "build.sh"       # script path (relative to profile dir)
```

The script is resolved relative to the profile directory. Example build script:

```bash
#!/bin/bash
# .coi/profiles/rust-dev/build.sh
apt-get update && apt-get install -y rustup
rustup default stable
```

You can also use inline commands instead of a script:

```toml
[container.build]
base = "coi-default"
commands = ["apt-get update", "apt-get install -y rustup"]
```

## Commands

### List Profiles

```bash
coi profile list
```

Shows all loaded profiles in a table:

```text
NAME        IMAGE         PERSISTENT  SOURCE
default     coi-default   -           (built-in)
python-ml   coi-default   -           .coi/profiles/python-ml/config.toml
rust-dev    coi-rust      true        .coi/profiles/rust-dev/config.toml
limited     coi-default   false       ~/.coi/profiles/limited/config.toml
```

The built-in `default` profile is always present and reflects the embedded default configuration. It can be used as a parent via `inherits = "default"` but cannot be edited or deleted.

### Selecting the Profile a Bare `coi` Uses (`[defaults] profile`)

By default a plain `coi` (no `--profile`) launches the synthesized `default`
profile — a clean clone of your global config. Set `[defaults] profile` in your
trusted config to point the no-flag `coi` at a profile of your choice instead:

```toml
# ~/.coi/config.toml
[defaults]
profile = "pickles"
```

Now `coi shell` / `coi run` launch the `pickles` profile, while `coi --profile default`
still gives you the clean clone of global config — so opinionated setup (git identity,
extra tools, a custom image) lives in a profile and vanilla stays reachable.

**Precedence (lowest to highest).** `[defaults] profile` is the lowest-precedence
profile source. Anything more explicit wins, and the default never stacks on top of it:

| Priority | Source |
|----------|--------|
| 1 (lowest) | `[defaults] profile` (this setting) |
| 2 | a `--resume`-remembered profile (the profile the resumed session ran) |
| 3 | an alias-saved profile |
| 4 (highest) | an explicit `--profile <name>` on the command line |

**Trusted scope only.** Honored only from trusted config (`~/.coi/config.toml` or
`$COI_CONFIG`). A project `./.coi/config.toml` naming a default profile is ignored
with a warning — a cloned or agent-planted repo cannot silently redirect your no-flag
default to a weaker profile.

**Unknown name is a hard error only at session launch.** If `[defaults] profile`
names a profile that does not exist, the error surfaces only when a session command
(`coi shell` / `coi run`) goes to apply it, with a message pointing at
`coi profile list`. Every other command (`coi list`, `coi kill`, `coi profile list`, …)
keeps working, since they never apply a profile and should not break over a typo in it.

Applies to the session-launching commands (`coi shell`, `coi run`).

### Built-in `hardened` Profile (Safe-Open for Untrusted Repos)

Coi also ships a built-in `hardened` profile — a hardened, one-flag preset for inspecting code you do not trust (a freshly-cloned repo). It bundles Coi's strongest controls (no in-shell command policing):

```bash
coi shell --profile hardened        # open an untrusted repo safely
coi run --profile hardened -- ...    # run a one-off command, hardened
coi profile info hardened           # see exactly what it locks down
```

| Setting | Value | Why |
|---|---|---|
| `network.mode` | `restricted` (+ block private nets & metadata endpoint) | no exfil path / SSRF |
| `security.secret_paths` | `.env`, `*.pem`, `*.key`, `*.p12`, `id_rsa`, `id_ed25519`, `*.tfvars`, `*.tfstate`, `.npmrc`, `.netrc`, `.git-credentials`, `credentials.json`, `service_account.json`, `kubeconfig`, `database.yml`, `secrets/**` | mask repo-local secrets from the agent |
| `security.host_immutable` | `true` | lock host-side protection of protected paths |
| `container.persistent` | `false` | ephemeral — nothing from the session persists |
| `ssh.forward_agent` | `false` | never hand an untrusted repo your SSH agent |
| `monitoring` (+ nft) | enabled, auto-pause/kill | catch in-container exfil / reverse shells |
| `security.reduce_kernel_surface` | `true` | no Docker/nesting, and the syscall families behind most recent kernel escape chains denied (io_uring, bpf, userfaultfd, keyring) — shrink the shared-kernel escape surface |
| `limits.runtime.max_duration` | `"4h"` | bound autonomous persistence — the [Trail of Bits escapes](https://blog.trailofbits.com/2026/08/26/vms-wont-contain-cyber-capable-agents/) took 12+ hours of uninterrupted autonomy |

It is a fixed baseline that overrides a weaker base config (a global `mode = "open"` still becomes `restricted` under `--profile hardened`), ships built-in (no setup), and can be overridden by a same-named disk profile (`~/.coi/profiles/hardened/` or `.coi/profiles/hardened/`).

**Limitations** (additive slice merges): the preset cannot subtract a `forward_env` you configured globally, nor re-enable protections if you globally set `disable_protection = true`. It hardens network egress, secrets, immutability, ephemerality, SSH-agent forwarding, monitoring, the kernel attack surface, and session duration. `secret_paths` is a sensible default set, not exhaustive — add repo-specific paths via your own config if needed.

### Show Profile Details

```bash
coi profile info rust-dev
```

Displays the full configuration of a named profile:

```text
Profile: rust-dev
Source:  .coi/profiles/rust-dev/config.toml

context = "/path/to/.coi/profiles/rust-dev/CONTEXT.md"
forward_env = ["CARGO_REGISTRY_TOKEN"]

[container]
image = "coi-rust"
persistent = true

[container.build]
base = "coi-default"
script = "/path/to/.coi/profiles/rust-dev/build.sh"

[environment]
RUST_BACKTRACE = "1"

[tool]
name = "claude"
permission_mode = "bypass"

[tool.claude]
effort_level = "high"

[limits.cpu]
count = "4"
```

### Use a Profile

```bash
coi shell --profile rust-dev
```

Profile settings are merged into the running config. As of v0.10.0, an explicitly selected profile's `[container]` settings win over the workspace's project config (so an untrusted repo's `.coi/config.toml` cannot silently override the profile you asked for).

### Create a Profile

```bash
coi profile create rust-dev --inherits default
coi profile create limited --project
```

Flags (as of v0.10.0 — profiles are authored by editing their `config.toml`, so `--image`/`--persistent` scaffolding flags are gone):

- `--inherits <parent>` - set a parent profile to inherit from
- `--user` - force creation in `~/.coi/profiles/`
- `--project` - force creation in `./.coi/profiles/`

Without `--user`/`--project`, the location auto-detects: project `.coi/` if it exists, otherwise `~/.coi/`. Settings like `[container] image` / `persistent` are edited into the created `config.toml` afterwards (`coi profile edit <name>`).

### Edit a Profile

```bash
coi profile edit rust-dev
EDITOR=nano coi profile edit rust-dev
```

Opens the profile's `config.toml` in `$VISUAL`, `$EDITOR`, or `vi`. After the editor exits, the file is re-parsed and validated - warnings are printed on invalid TOML but the file is not deleted.

### Delete a Profile

```bash
coi profile delete rust-dev           # Interactive confirmation
coi profile delete old-profile --force  # Skip confirmation
```

The built-in `default` profile cannot be edited or deleted.

## Examples

### Minimal Profile

```toml
# .coi/profiles/quick/config.toml
[container]
persistent = false

[limits.runtime]
max_duration = "30m"
```

### Full-Stack Development

```toml
# .coi/profiles/fullstack/config.toml
forward_env = ["GITHUB_TOKEN", "DATABASE_URL"]

[container]
image = "coi-default"
persistent = true

[environment]
NODE_ENV = "development"

[tool]
name = "claude"
permission_mode = "bypass"

[[mounts]]
host = "~/.npm"
container = "/home/code/.npm"

[limits.cpu]
count = "4"

[limits.memory]
limit = "8GiB"
```

### Restricted Research

```toml
# .coi/profiles/research/config.toml
context = "CONTEXT.md"

[tool]
name = "claude"
permission_mode = "interactive"

[network]
mode = "allowlist"
allowed_domains = ["github.com", "stackoverflow.com", "docs.python.org"]

[limits.runtime]
max_duration = "1h"

[limits.memory]
limit = "2GiB"
```

## Validation

- Profile directories without a `config.toml` are silently skipped
- Invalid TOML in a profile `config.toml` causes a fatal error at config load time
- Missing context files are validated at `--profile` usage time (not at config load) - this allows profiles to be loaded from all sources without requiring every referenced file to exist
- Missing build scripts are validated at `--profile` usage time
- `coi profile info` displays all resolved paths (build scripts, context files) so you can verify they are correct

## JSON Schema

Coi ships a JSON Schema 2020-12 document that describes every field accepted by a profile `config.toml`. External tools — a web UI, an editor plugin, a CI validator — can consume it to validate profile data without duplicating Coi's validation logic.

```bash
# Print the schema
coi schema profile

# Save it for use in another tool
coi schema profile > profile.schema.json
```

The self-contained schema (all `$defs` bundled inline) is produced by `coi schema profile`; the source files live under [`schema/`](https://github.com/coipond/coi/tree/master/schema) but are incomplete without the bundling step. It covers all field types, enum values (`network.mode`, `tool.permission_mode`, `tool.claude.effort_level`, `timezone.mode`), required fields on mount and socket entries, and rejects unknown keys.

Any JSON Schema 2020-12 validator can use it — for example the Ruby [`json_schemer`](https://github.com/davishmcclurg/json_schemer) gem or the Python [`jsonschema`](https://python-jsonschema.readthedocs.io/) package.

## Best Practices

1. **One profile per distinct workload type, not per project** - Profiles work best as reusable templates: `rust-dev`, `python-ml`, `restricted-research`. Avoid creating a profile for every individual project; use per-project `.coi/config.toml` for project-specific overrides instead.

2. **Use inheritance to avoid duplication** - If you have a `rust-dev` profile and a `rust-dev-nightly` variant, use `inherits = "rust-dev"` and override only what differs. This keeps profiles maintainable when you need to change a shared setting.

3. **Check profiles into the project repo** - Project-level profiles (`.coi/profiles/NAME/`) should be committed to the repo. This makes the development environment reproducible for all contributors without per-machine setup.

4. **Name context files clearly** - `CONTEXT.md` is conventional but `AI_GUIDELINES.md` or `AGENT_CONTEXT.md` are more self-explanatory for contributors who do not know Coi. The filename does not affect behavior.

5. **Validate profile paths with `coi profile info`** - After creating or editing a profile, run `coi profile info <name>` to verify that build script and context file paths resolve correctly before using the profile in a session.

6. **Pin image versions in team profiles** - If multiple contributors share a profile, use a specific versioned image (`myproject-v1.2`) rather than a mutable name. This prevents "works on my machine" issues when the image is rebuilt.

## See Also

- [Configuration](Configuration) - Base configuration reference
- [Network Isolation](Network-Isolation) - Per-profile network mode settings
- [Image Management](Image-Management) - Per-profile image configuration
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How profiles affect session behavior
