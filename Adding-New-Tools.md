# Adding New Tools

Coi's tool abstraction makes it straightforward to add new AI coding assistants. Each tool implements the `Tool` interface:

```go
type Tool interface {
    Name() string
    Binary() string
    ConfigDirName() string
    SessionsDirName() string
    BuildCommand(sessionID string, resume bool, resumeSessionID string) []string
    DiscoverSessionID(stateDir string) string
    GetSandboxSettings() map[string]interface{}
}
```

Tools that use a config directory (most tools) should also implement `ToolWithConfigDirFiles`, which tells Coi which files to copy from the host and where to inject sandbox settings:

```go
type ToolWithConfigDirFiles interface {
    Tool
    EssentialConfigFiles() []string       // Files to copy from host config dir
    SandboxSettingsFileName() string      // File to inject sandbox/bypass settings into
    StateConfigFileName() string          // Sibling state file (e.g. ".claude.json"), "" if none
    AlwaysSetupConfig() bool              // Run config setup even when the host config dir is absent
}
```

Tools that support configurable reasoning effort (like Claude) can implement `ToolWithEffortLevel`:

```go
type ToolWithEffortLevel interface {
    Tool
    SetEffortLevel(level string)  // "low", "medium", "high", "xhigh", "max", or "auto"
}
```

Tools that support configurable permission modes can implement `ToolWithPermissionMode`:

```go
type ToolWithPermissionMode interface {
    Tool
    SetPermissionMode(mode string)  // "bypass" (default) or "interactive"
}
```

Tools that support auto-loading context from a file (like Claude's `~/.claude/CLAUDE.md`) can implement `ToolWithAutoContextFile`:

```go
type ToolWithAutoContextFile interface {
    Tool
    AutoContextFile() string  // Path relative to home dir (e.g. ".claude/CLAUDE.md")
}
```

Tools that reference the context file path in their config (like OpenCode's `instructions` field) can implement `ToolWithAutoContextPath`:

```go
type ToolWithAutoContextPath interface {
    Tool
    SetAutoContextPath(path string)  // Absolute path to sandbox context file
}
```

Tools that can be launched non-interactively with an initial prompt (for `coi tool spec`) can implement `ToolWithPrompt` and expose model/effort as container env via `ToolWithContainerEnv`; see [Headless Orchestration](Headless-Orchestration) for how those are used.

See `internal/tool/tool.go` and `internal/tool/opencode.go` for complete examples.

## See Also

- [Supported Tools](Supported-Tools) - the tools Coi ships with today
- [Sandbox Context](Sandbox-Context) - the context a new tool inherits and can auto-load
- [Headless Orchestration](Headless-Orchestration) - the `ToolWithPrompt` / `ToolWithContainerEnv` capabilities `coi tool spec` uses
- [Architecture and Security Model](Architecture-and-Security-Model) - the isolation model a new tool runs inside
