# Getting Started

Coi wraps your AI coding tool in an Incus system container with credential isolation, network controls, and security monitoring. Setup takes about 10 minutes the first time — after that, starting a session is a single command from any project directory.

## Prerequisites

- **Linux** (Ubuntu, Fedora, Arch, and others) — native Incus support
- **macOS** — requires [Colima](https://github.com/abiosoft/colima) or [Lima](https://github.com/lima-vm/lima) to provide a Linux VM; Coi manages this transparently
- **Windows** — not natively supported; use WSL2 with a Linux distribution inside

Platform-specific setup (Incus installation, nftables/passwordless sudo, group membership) is covered in the setup guides:

- [Linux Setup Guide](Linux-Setup-Guide)
- [macOS Setup Guide](macOS-Setup-Guide)

Complete those steps first, then return here.

## Step 1: Install Coi

Run the install script from the [Coi repository](https://github.com/coipond/coi):

```bash
curl -fsSL https://raw.githubusercontent.com/coipond/coi/master/install.sh | bash
```

This installs the `coi` binary and sets up shell completions. Verify it worked:

```bash
coi --version
```

## Step 2: Build the Container Image

Coi needs a base container image before it can start sessions. This is a one-time step that downloads and configures the Ubuntu base image with your AI tool and Coi's monitoring components:

```bash
coi build
```

This takes 5-10 minutes depending on your connection. You only need to run it again after a Coi update that changes the image, or when you want to rebuild with a new Ubuntu base.

> **Note: Interactive Build Prompt**
> If you run `coi shell` before building, Coi detects the missing image and offers to build it interactively. You can answer yes and skip this step entirely.

## Step 3: Start Your First Session

Navigate to a project directory and start a session:

```bash
cd ~/your-project
coi shell
```

Coi creates an isolated container, mounts your project at `/workspace` inside it, starts tmux, and launches your AI tool. You are now inside the container with your AI assistant ready to use.

The first session for a given project takes a few seconds to start. If startup takes tens of seconds instead, run `coi health` — a warning on the storage pool means it is on the slow `dir` driver; recreate it with zfs/btrfs (re-running `install.sh` sets one up). Subsequent sessions on the same project are faster because Coi caches setup state.

## Step 4: Do Some Work

Inside the container, everything works as you would expect. Your AI tool can read and modify files in `/workspace` (your project), run shell commands, install packages, and use the internet. What it cannot do by default:

- Access your SSH keys or home directory on the host
- Reach other machines on your local network
- Modify git hooks, `.vscode/`, or other supply-chain-sensitive paths

When you are done, end the session cleanly from inside the container:

```bash
close        # or: sudo poweroff
```

> **Note: Use `close` Not `exit`**
> Typing `exit` in the bash prompt exits the shell but leaves the container running. Use `close` (a Coi-provided alias for `poweroff`) to properly end the session, save state, and clean up the container.

Coi saves your AI tool's conversation history to `~/.coi/sessions-<tool>/` on the host, then deletes the container.

## Step 5: Resume Where You Left Off

To pick up where you left off:

```bash
coi shell --resume
```

This restores your AI tool's conversation history in a fresh container. The container itself is new (clean system state), but your conversation context is intact.

## Parallel Sessions

You can run multiple sessions against the same project simultaneously - each gets its own isolated container with a separate home directory and process space:

```bash
# Terminal 1
coi shell    # slot 1 — working on feature A

# Terminal 2
coi shell    # slot 2 — working on feature B in parallel
```

List all running sessions:

```bash
coi list
```

## Persistent Sessions

By default, containers are deleted when a session ends. If you install packages or build artifacts that you want to keep, enable persistence in config (as of v0.10.0 there is no `--persistent` flag — persistence is a per-project or per-profile setting):

```toml
# ./.coi/config.toml (or a profile's config.toml)
[container]
persistent = true
```

```bash
coi shell                 # container now survives exit
coi attach                # reconnect to the same container later
```

## What to Read Next

Once you have a session working, the most useful next steps are:

- [Configuration](Configuration) - Customize network mode, resource limits, environment variable forwarding, and mounts
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - Understand ephemeral vs. persistent sessions, slots, SSH agent forwarding, and aliases
- [Network Isolation](Network-Isolation) - Choose the right network mode for your threat model
- [Security Best Practices](Security-Best-Practices) - Recommended settings for working with untrusted codebases
- [Architecture and Security Model](Architecture-and-Security-Model) - Understand what Coi protects against and where its limits are

## See Also

- [Linux Setup Guide](Linux-Setup-Guide) - Platform prerequisites for Linux
- [macOS Setup Guide](macOS-Setup-Guide) - Platform prerequisites for macOS via Colima/Lima
- [Configuration](Configuration) - Full configuration reference
- [Troubleshooting](Troubleshooting) - If something does not work as expected
