# FAQ: Setup and Operation

Common questions about platform support, configuration, and day-to-day use.

## Can I Run Coi on Windows?

Not directly. Incus is Linux-only. However, you can:

1. **WSL2 (Windows Subsystem for Linux)** - Install a Linux distribution in WSL2, then install Incus and Coi inside WSL2
2. **VM** - Run a Linux VM (Ubuntu, Debian, etc.) and install Coi there

Windows support via WSL2 is experimental and not officially tested. Linux or macOS (via Colima/Lima/OrbStack) are the recommended platforms.

## Why Does Coi Need a VM (Colima/Lima/OrbStack) on macOS? Can I Use Incus Directly?

Incus requires a Linux kernel - LXC (Linux Containers) is a Linux kernel feature. On macOS, a Linux VM is needed to run Incus.

- **On Linux**: Coi uses Incus directly. No Colima, no VM layer.
- **On macOS**: [Colima](https://github.com/abiosoft/colima), [Lima](https://github.com/lima-vm/lima), or [OrbStack](https://orbstack.dev) provides a lightweight Linux VM where Incus runs. As of v0.11.1 Coi handles each VM's UID-mapping quirks automatically (including OrbStack's FUSE-backed macOS share, which gets a `raw.idmap` mapping instead of idmapped mounts).

If you are on Linux, there is no Colima involved. See the [macOS Setup Guide](macOS-Setup-Guide) for macOS-specific instructions.

## Is This Really Simpler Than Just Running Claude Code Directly?

**First-time setup:** `coi build` (one time, 5-10 minutes), then `coi shell` - that is it.

**After setup:** `cd your-project && coi shell` - same simplicity as running Claude Code directly, but with:

- ✅ Automatic file ownership (no permission issues)
- ✅ Credential isolation (your SSH keys stay safe)
- ✅ Session save/resume (continue later)
- ✅ Parallel sessions (multiple workspaces/slots)
- ✅ Clean environment (no host pollution)

The complexity is hidden. You get security and isolation with the same simple workflow.

## What Is Incus? (Is It the Same as tmux?)

**No, Incus is not tmux.** They are completely different tools.

**Incus** - Linux container and VM manager (like Docker, but for system containers):

- Manages containers with full operating systems inside
- Provides isolation, networking, storage, etc.
- Coi uses Incus to create isolated environments

**tmux** - Terminal multiplexer for managing shell sessions:

- Lets you detach/reattach terminal sessions
- Coi uses tmux inside containers to manage AI tool sessions

**In Coi:** Incus creates the container, tmux runs inside it to manage your session with the AI tool.

## Can I Preserve the Workspace Path Inside the Container?

**Yes.** By default, your project is mounted at `/workspace` inside the container. If you need the container workspace to use the same absolute path as on your host (useful for tools that store session data relative to the workspace directory), enable `preserve_workspace_path`:

```toml
# .coi/config.toml or ~/.coi/config.toml
[paths]
preserve_workspace_path = true
```

With this enabled, if your project is at `/home/user/projects/my-app`, it is mounted at the same path inside the container.

## Can I Use Docker and Docker Compose Inside Coi?

**Yes.** Coi automatically enables Docker support flags (`security.nesting`, `security.syscalls.intercept.mknod`, `security.syscalls.intercept.setxattr`) on session containers. Docker and Docker Compose work out of the box without any additional configuration.

Docker commands also work without `sudo` - the container's `code` user is added to the `docker` group, and the Docker socket is configured with the correct group ownership.

If Docker commands fail, verify that:

1. Your Coi image was built with the latest `coi build`
2. You are using `coi shell` (not raw `incus exec`)

## How Do I Add Extra Context Files (Agents, Rules, Configs) Into My Container?

Use `[[mounts.default]]` in your config to mount host directories into the container. The key rule is: mount subdirectories, not the parent config directory (e.g., mount `~/.claude/skills`, not `~/.claude`).

See the [Configuration - Mounting Additional Files](Configuration#mounting-additional-files) section for full examples, common patterns (Claude skills/commands/plugins, opencode agents), and security guidance.

## Why Run an AI Agent Locally Instead of in the Cloud?

AI coding agents are most effective when they can:

- Access your actual project files and directory structure
- Run your test suite and build tools
- Interact with local services (databases, APIs)
- Use your project's Docker Compose setup
- Work with low latency and no network dependency

Running in the cloud adds latency, requires syncing files, and complicates access to local development services. Coi's approach is: run locally but contained - you get the convenience of local execution with the safety of isolation and [security monitoring](Security-Monitoring).

## Can I Use Coi With Local/Self-Hosted AI Models?

Coi supports multiple AI coding tools through its extensible tool interface. Currently supported:

- **Claude Code** (default) - Anthropic's CLI agent
- **opencode** - open-source AI coding agent
- **pi** - open-source AI coding agent

Coming soon:

- **Aider** - AI pair programming tool
- **Cursor** (CLI mode)

See [Supported Tools](Supported-Tools) for the full list and configuration details.

Adding support for other tools (including those backed by local models like Ollama) is possible by implementing the tool interface. If you would like to see a specific tool supported, [open an issue](https://github.com/coipond/coi/issues).

## More Questions?

[Join the Coi community on Slack](https://slack.karafka.io) for live discussion and support.

## See Also

- [Getting Started](Getting-Started) - Step-by-step setup and first session walkthrough
- [Configuration](Configuration) - Full configuration reference
- [Supported Tools](Supported-Tools) - AI tool options and configuration
- [FAQ](FAQ) - Return to the FAQ index
