# Sandbox Context File

Coi automatically injects a `~/SANDBOX_CONTEXT.md` file into every container, describing the sandbox environment to AI tools. This is part of Coi's isolation model — tools receive accurate information about what they can and cannot do without needing host configuration. See [Architecture and Security Model](Architecture-and-Security-Model) for how this fits into the broader security design. This includes:

- Workspace path and home directory
- OS and architecture
- Container mode (ephemeral/persistent)
- Network mode and restrictions
- SSH agent availability
- Docker availability
- User privileges and sudo access
- Protected paths and limitations

The file is rendered from a built-in template with session-specific values and regenerated on every session start (including resume), so AI tools always have current information about their environment.

## Auto-Context Injection

By default, Coi also injects the sandbox context into each tool's native context-loading mechanism (`auto_context = true`):

- **Claude Code**: Sandbox context is written to `~/.claude/CLAUDE.md`, which Claude auto-reads at every session start. If the host has a `CLAUDE.md` with user instructions, those are preserved: the sandbox context is written as a single marker-delimited managed block (`# BEGIN COI Sandbox Context (managed by coi - do not edit this block)` … `# END COI Sandbox Context (managed by coi)`), and injection is idempotent — each session replaces the managed block with one fresh copy, healing any duplicated pre-v0.11.1 copies, while non-managed content is left untouched.
- **OpenCode**: The `instructions` field in `opencode.json` is set to reference `~/SANDBOX_CONTEXT.md`.

This means AI tools automatically have full sandbox awareness without users needing to manually reference `~/SANDBOX_CONTEXT.md`.

To disable (the `~/SANDBOX_CONTEXT.md` file is still created):

```toml
[tool]
auto_context = false
```

**Override with a custom file:**

```toml
[tool]
context_file = "~/my-sandbox-context.md"
```

When `context_file` is set, Coi uses your custom file instead of the built-in template. Supports `~` expansion.

**Machine-readable companion (`~/SANDBOX_CONTEXT.json`):**

For programmatic consumers, Coi can also emit a structured, versioned JSON
companion to the Markdown context. Enable it with `context_json`:

```toml
[tool]
context_json = true                      # also write ~/SANDBOX_CONTEXT.json
context_json_file = "~/my-context.json"  # optional: use your own file instead of the built-in template
```

Custom context files (`context_file` / `context_json_file`) are honored only
from trusted-scope config (`~/.coi/config.toml` / `$COI_CONFIG`), not from an
untrusted project `./.coi/config.toml`.

## See Also

- [Supported Tools](Supported-Tools) - per-tool context-injection behavior
- [Configuration](Configuration) - the `[tool]` context options (`context_file`, `context_json`, `auto_context`)
- [Architecture and Security Model](Architecture-and-Security-Model) - how context injection fits the security model
- [Profiles](Profiles) - per-profile context files
