# Supported AI Coding Tools

Coi supports multiple AI coding assistants through a pluggable tool abstraction.

## Currently Supported

### Claude Code (Default)

Anthropic's official CLI tool for AI-assisted coding.

```bash
# Use Claude Code (default)
coi shell
```

```toml
# Explicit tool selection (config or profile; the --tool flag was removed in v0.10.0)
[tool]
name = "claude"
```

**Configuration:**

- Config directory: `~/.claude/`
- Session storage: `~/.coi/sessions-claude/`
- Resume: `coi shell --resume` (auto-continues last conversation)

**Claude-Specific Settings:**

You can configure Claude Code-specific settings in your `.coi/config.toml`:

```toml
[tool]
name = "claude"

[tool.claude]
effort_level = "medium"  # "low", "medium", "high", "xhigh", "max", or "auto"
```

The `effort_level` setting controls Claude's response thoroughness. By default it is unset, leaving the user in control of the effort level interactively inside Claude; when set, Coi injects the configured level into the session. Either way, Coi suppresses the interactive effort selection prompt during autonomous shell sessions.

**Requirements:**

- Claude account (shows login screen if not authenticated)
- API access via Anthropic or AWS Bedrock

### opencode

Open-source AI coding agent from [opencode.ai](https://opencode.ai).

```toml
# Select opencode in .coi/config.toml, ~/.coi/config.toml, or a profile
[tool]
name = "opencode"
```

A per-tool [profile](Profiles) is the ergonomic equivalent of the removed `--tool` flag: `coi shell --profile opencode-dev` carries the tool's whole setup, not just its name.

**Configuration:**

- Config file: `~/.config/opencode/opencode.json` (XDG-compliant location)
- Session storage: `.opencode/` in workspace directory (SQLite database)
- Resume: `coi shell --resume` with `[tool] name = "opencode"` configured (or `--continue`, both are aliases)

**Requirements:**

- API key required (unlike Claude, opencode will not start without one)
- Set `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` environment variable
- Or configure in `~/.opencode.json`:

```json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    }
  }
}
```

**Key Differences from Claude:**

| Feature | Claude Code | opencode |
|---------|-------------|----------|
| Login screen | Shows if no API key | Fails if no API key |
| Session storage | `~/.claude/` (home dir) | `.opencode/` (workspace, SQLite) |
| Resume behavior | `--resume` continues last conversation | `--resume` or `--continue` (identical aliases) |
| Config location | `~/.claude/` directory | `~/.config/opencode/opencode.json` |
| Permission bypass | Auto-injected into settings.json | Auto-injected `"permission": {"*": "allow"}` |

### pi

AI coding assistant from [pi.dev](https://pi.dev).

```toml
# Select pi in .coi/config.toml, ~/.coi/config.toml, or a profile
[tool]
name = "pi"
```

**Configuration:**

- Config directory: `~/.pi/agent/`
- Essential config files copied from host: `settings.json`, `models.json`, `auth.json`, `AGENTS.md`
- Session storage: redirected to `.pi-sessions/` in the workspace (via `PI_CODING_AGENT_SESSION_DIR`) so sessions survive ephemeral container recreation
- Resume: `coi shell --resume` with `[tool] name = "pi"` configured (pi auto-continues via `--continue`; it manages its own session discovery)

**Context injection:** pi has no permission-bypass system (it runs autonomously without a permission gate), so `permission_mode` is effectively a no-op. Sandbox context is injected by symlinking `~/.pi/agent/APPEND_SYSTEM.md` to `~/SANDBOX_CONTEXT.md` — pi appends `APPEND_SYSTEM.md` to its system prompt without replacing your `AGENTS.md`.

**Requirements:**

- pi authenticated (`auth.json`) or an API key forwarded via `forward_env`

### omp (Oh My Pi)

[Oh My Pi (`omp`)](https://github.com/can1357/oh-my-pi), a coding agent in the pi family.

```toml
# Select omp in .coi/config.toml, ~/.coi/config.toml, or a profile
[tool]
name = "omp"
```

**Not in the default image** - omp is opt-in at build time. Add it to the agent set and rebuild:

```toml
[container.build]
agents = ["claude", "omp"]   # omit this key to install ALL supported agents
```
```bash
coi build --force
```

**Configuration:**

- Config directory: `~/.omp/`
- Session storage: redirected to the workspace mount (via `OMP_SESSION_DIR`) so sessions survive ephemeral container recreation — the same mechanism `pi` uses
- Context injection: the sandbox context is wired into omp's config dir at `~/.omp/APPEND_SYSTEM.md` (appended to omp's system prompt, like pi's `APPEND_SYSTEM.md`)

**Note:** the session-dir and system-prompt mechanisms are modeled on `pi`'s and should be validated against the omp version you run.

### Codex CLI

OpenAI's coding agent, [Codex CLI](https://developers.openai.com/codex/cli).

```toml
# Select codex in .coi/config.toml, ~/.coi/config.toml, or a profile
[tool]
name = "codex"

# Optional model / reasoning knobs
[tool.codex]
# model = "o4-mini"          # -> codex -m <model>
# reasoning_effort = "high"  # -> codex -c model_reasoning_effort=high
```

**Not in the default image** - Codex is opt-in at build time. Add it to the agent set and rebuild:

```toml
[container.build]
agents = ["claude", "codex"]   # omit this key to install ALL supported agents
```
```bash
coi build --force
```

If your configured tool is missing from the image, coi warns before launch rather than failing with `exit 127`.

**Permission mode:** `bypass` (default) maps to `--dangerously-bypass-approvals-and-sandbox` - the Incus container is the sandbox, codex's own Landlock sandbox may not work nested inside it, and the flag also skips the first-run folder-trust prompt. `interactive` keeps codex's own approval prompts (`-s workspace-write -a on-request`).

**Configuration:**

- Config directory: `~/.codex/`
- Host files seeded into the container: `auth.json`, `config.toml`, `AGENTS.md` (via the same credential catalog claude/opencode/pi use).
- Unlike the others there is no settings-file injection - codex config is TOML while coi's settings merge is JSON-only, so `config.toml` is copied verbatim and everything coi controls is passed as launch flags (centralized in `CodexTool.BuildCommand`).
- Context injection: the sandbox context block is written into `~/.codex/AGENTS.md` (container-global - never the workspace `AGENTS.md`, which lives on the host bind-mount).
- Resume: `coi shell --resume` discovers the newest `sessions/**/rollout-*.jsonl` UUID and runs `codex resume <uuid>`, falling back to `codex resume --last`.
- The workspace `.codex/config.toml` is a protected path (read-only + placeholder), like `.claude/settings.json`, so a contained agent cannot plant a host-auto-executed command (notify hooks, MCP launchers).

**Authentication:** log in on the host first with `codex login`, and coi seeds `~/.codex/auth.json` into the container. If the host stores credentials in the OS keyring (no `auth.json`) or you have never logged in, authenticate inside the container with `codex login --device-auth` (needs device-auth enabled in your org) or `codex login --with-api-key` - the plain `codex login` browser flow cannot work in a container because its OAuth localhost callback is unreachable from the host browser.

## Passing API Keys

For opencode, pi, and other tools that require API keys:

```toml
# Via forward_env in config (~/.coi/config.toml)
[defaults]
forward_env = ["ANTHROPIC_API_KEY"]
```

```bash
# Via host config file (copied into container automatically)
# ~/.opencode.json on host is copied and merged with permission bypass
```

## Tool Selection

Tool selection is config-only (as of v0.10.0 — the `--tool` flag was removed). For per-invocation switching, use a per-tool [profile](Profiles): `coi shell --profile opencode-dev`. To switch tools inside one persistent container (keeping its code and packages), give two per-tool profiles the same `[container] session_name` — see [Running a different AI tool in the same container](Container-Lifecycle-and-Sessions#running-a-different-ai-tool-in-the-same-container-v012).

### Per-Project (.coi/config.toml)

```toml
# .coi/config.toml in project root
[tool]
name = "opencode"
```

> **Migration from 0.7.x:** Project config has moved from `.coi.toml` to `.coi/config.toml`. See [Configuration](Configuration#per-repository-configuration) for details.

### Global Default (~/.coi/config.toml)

```toml
[tool]
name = "opencode"
```

**Precedence:** profile (`--profile`) > `.coi/config.toml` > global config > `"claude"` (default)

## Tool Credentials and Third-Party Providers

The credential files each built-in tool needs (listed under "Configuration" above) come from Coi's embedded credential catalog and are copied into fresh containers automatically — pushed, chowned to the container user, never bind-mounted. As of v0.10.0 the same catalog serves tools that are not Coi-managed AI tools: a profile or config can reference a named bundle, or declare an ad-hoc file, via `[[credentials]]`:

```toml
# e.g. a profile that runs Claude through Ollama Cloud
[[credentials]]
bundle = "ollama"          # copies ~/.ollama/id_ed25519 into the container, mode 0600
```

See [Configuration](Configuration) for the ad-hoc form and the trust model (ad-hoc entries from untrusted project configs are gated behind `coi trust`).

## Permission Mode

By default, Coi bypasses all permission prompts inside containers so tools run autonomously. If you prefer a human-in-the-loop workflow where the tool asks before running commands, set `permission_mode` under `[tool]`:

```toml
# .coi/config.toml or ~/.coi/config.toml
[tool]
permission_mode = "interactive"  # "bypass" (default) or "interactive"
```

| Mode | Claude Code | opencode |
|------|-------------|----------|
| `bypass` (default) | `--permission-mode bypassPermissions` flag + bypass settings injected into `settings.json` | `"permission": {"*": "allow"}` injected into `opencode.json` |
| `interactive` | No bypass flag or settings - Claude asks before running commands, and **auto mode stays selectable in-session** (Shift+Tab) | No permission bypass injected - opencode asks before running commands |

**Notes:**

- This is a cross-cutting setting that applies to whichever tool is active
- Effort level settings (for Claude) are always injected regardless of permission mode
- Empty or omitted value defaults to `"bypass"` for backward compatibility
- **Auto mode under `interactive` (as of v0.12.0, #764):** earlier versions also wrote a Claude Code managed-settings policy (`disableAutoMode`) whenever the tool was Claude — the highest-precedence tier, un-overridable by any user setting — which stripped auto mode from the in-session Shift+Tab cycle even under `interactive`. That policy is now written only under `bypass`. So `interactive` is a genuinely interactive session: you get manual approval prompts by default and can opt into auto mode yourself. `bypass` is unchanged (the policy still suppresses Claude's startup auto-mode prompt).

## Adding New Tools

Coi's tool abstraction (the `Tool` interface plus optional capability interfaces) makes it straightforward to add a new AI coding assistant.

**→ See [Adding New Tools](Adding-New-Tools)** for the interfaces to implement and worked examples.

## Sandbox Context File

Coi injects a `~/SANDBOX_CONTEXT.md` (and an optional structured `~/SANDBOX_CONTEXT.json`) describing the sandbox to the tool, and by default wires it into each tool's native context system.

**→ See [Sandbox Context](Sandbox-Context)** for what it includes, auto-context injection per tool, disabling it, and custom / JSON context files.

## Coming Soon

- **Aider** - AI pair programming in your terminal
- **Cursor** - AI-first code editor (CLI mode)

The tool abstraction layer makes it easy to add support for new AI coding assistants as they become available.

## See Also

- [Configuration](Configuration) - Tool-specific configuration options
- [Profiles](Profiles) - Per-profile tool selection
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How tools run inside containers
- [Sandbox Context](Sandbox-Context) - The injected environment description and auto-context injection
- [Adding New Tools](Adding-New-Tools) - Implement the `Tool` interface to add a new assistant
- [Architecture and Security Model](Architecture-and-Security-Model) - How sandbox context injection fits into the security model
