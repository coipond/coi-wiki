# Architecture and Security Model

Coi is a session manager for AI coding tools built on Incus system containers. It combines container isolation with an active security monitoring layer to address the specific risks that arise when giving an AI agent broad system access to a development environment. This page explains the threat model Coi was designed for, how its defense layers work, and where its boundaries are.

## The Problem

AI coding tools are powerful because they can run arbitrary shell commands, install packages, modify files, and interact with your development environment directly. That same power is what makes them risky to run without containment.

Running an AI tool directly on your host exposes:

- **Credentials** - SSH keys, shell environment variables, API tokens, and anything in your home directory
- **Local network** - Internal services, databases, other machines on your subnet
- **Supply chain** - Git hooks, VS Code extension configs, and other tooling that executes automatically during normal development workflows
- **Persistent state** - Any file the AI can write, it can also corrupt or exfiltrate

Coi's design goal is to give AI tools everything they need to be useful (your workspace, your project's tools, internet access) while containing the blast radius if something goes wrong.

## Threat Model

Coi is designed to limit damage from these specific threat categories:

**Credential exposure** - An AI tool operating in a compromised codebase, or tricked via prompt injection, may attempt to read SSH keys, environment variables, or cloud credentials and send them out over the network.

**Lateral movement** - Once inside a developer's machine, a malicious agent can scan and probe internal network services - CI systems, internal APIs, databases, other developer machines.

**Supply-chain attacks** - Git hooks (`.git/hooks/`), VS Code settings (`.vscode/`), and Husky configs (`.husky/`) execute automatically during commits, pushes, and editor events. A modified hook can silently run attacker code on the developer's host machine.

**Data exfiltration** - Source code, secrets embedded in files, and environment state can be copied and sent to an attacker-controlled server.

**Reverse shells** - An agent can spawn a process that connects back to an attacker, providing persistent remote access that survives the AI session.

**Persistent damage** - Without containment, AI-generated changes to system files, package managers, or shell configuration persist after the session ends.

## Defense Layers

Coi addresses these threats through multiple independent layers. No single layer is sufficient on its own.

### Container Isolation

Each session runs inside an Incus system container - a full Linux OS instance with its own init system (systemd), user space, and filesystem. The container shares the host kernel but has isolated everything else.

What is mounted into the container:

- Your workspace directory (read-write)
- Essential AI tool config files (credentials, settings) - copied in at session start, not bind-mounted from your live home directory

What is not accessible inside the container by default:

- Your SSH keys and `~/.ssh`
- Your home directory
- Your shell environment variables (must be explicitly forwarded via `forward_env`)
- Any other host filesystem path

File ownership is handled automatically: Coi selects per container between Incus idmapped (`shift=true`) mounts and a `raw.idmap` mapping, checking each mount source's filesystem before start (FUSE-family/9p sources get `raw.idmap`) and falling back automatically when a kernel or filesystem cannot do idmapped mounts. Either way the AI tool sees files owned by its `code` user without any manual `chown` or privilege escalation. On macOS VM hosts (Colima/Lima/OrbStack) see the [macOS Setup Guide](macOS-Setup-Guide) for the mechanism details.

### Credential Isolation

Coi copies AI tool credentials (e.g., `~/.claude/credentials`) into the container at session start rather than bind-mounting live config directories. This means:

- The AI tool can authenticate, but the credentials are a point-in-time copy, not a live window into your home directory
- No arbitrary credential files are reachable - only what Coi explicitly copies
- SSH keys are never copied unless you explicitly enable SSH agent forwarding (`[ssh] forward_agent = true`), which forwards via a socket proxy rather than copying key material

### Network Isolation

Network access is controlled by host-side nftables rules in the FORWARD chain, applied per container IP address. Three modes are available:

**`restricted` (default)** - Internet access is allowed, but all RFC1918 private network ranges are blocked. The container can reach public services but cannot reach your internal network, databases, CI systems, or other machines on your subnet. Cloud metadata endpoints (169.254.169.254) and known C2 ports are also blocked.

**`allowlist`** - Only explicitly listed domains or IP ranges are reachable. Everything else is denied. Use for untrusted projects where you want to constrain which external services the AI can contact.

**`open`** - No network filtering. Use only when nftables/passwordless sudo is unavailable (or `use_sudo = false` is set) or for fully trusted projects that require local network access.

See [Network Isolation](Network-Isolation) for the full configuration reference.

### Supply-Chain Protection

Selected paths inside the container are mounted read-only to prevent an AI tool from modifying code that executes automatically during normal developer workflows:

- `.git/hooks/` - Scripts that run on commit, push, merge, etc.
- `.vscode/` - VS Code workspace settings and extension configs
- `.husky/` - Git hook manager configs

These paths are write-protected from inside the container regardless of the network mode. An AI tool cannot make a change here that would silently execute on the developer's host at the next `git commit`.

> **Warning: Read-Only Protection Can Be Disabled**
> Setting `[security] disable_protection = true` removes all read-only enforcement on these paths. Only do this if you fully understand the supply-chain risk. See [Security Best Practices](Security-Best-Practices) for more.

### Security Monitoring

Coi runs an active monitoring process that watches AI tool behavior in real time and responds automatically to detected threats. Monitoring operates at three levels:

**Process monitoring** - Watches all processes spawned inside the container for patterns associated with reverse shells (outbound connections on non-standard ports, common shell-over-network tools), credential scanning (access to known credential file patterns), and suspicious process trees.

**Filesystem monitoring** - Tracks I/O patterns associated with bulk data exfiltration - unusually high read rates across large numbers of files, access to paths outside the workspace, etc.

**Network monitoring** - Uses kernel-level nftables rules to detect and log connections to blocked destinations. Connections to known C2 infrastructure, cloud metadata services, and private network ranges are detected and logged even when the container is in `open` mode.

Detected threats are classified by severity and trigger automatic responses:

| Severity | Response |
|----------|----------|
| INFO | Log to audit file |
| WARNING | Log and alert |
| HIGH | Log, alert, and pause the container |
| CRITICAL | Log, alert, and kill the container |

All events are written to `~/.coi/audit/<container-name>.jsonl` for forensic review after a session.

See [Security Monitoring](Security-Monitoring) for the full event catalog and configuration.

### Session Management

Sessions are ephemeral by default. When a session ends:

- The AI tool's conversation history and config are saved to `~/.coi/sessions-<tool>/`
- The container is deleted - any AI-generated changes to system state, installed packages, or container files are discarded
- Only your workspace files (which are mounted from the host) persist

This means a compromised or corrupted container is automatically cleaned up on exit. The next `coi shell` starts from a clean image.

Use persistent mode (`[container] persistent = true` in config or a profile) when you need the container state to survive (e.g., after installing tools or building artifacts), with the understanding that persistent containers accumulate state between sessions.

## How the Pieces Fit Together

```text
Host machine
├── Incus system container
│   ├── systemd (init)
│   ├── tmux
│   │   └── AI coding tool (Claude Code, opencode, etc.)
│   ├── /workspace  ← bind-mounted from host (read-write)
│   ├── ~/.git/hooks  ← read-only (supply-chain protection)
│   └── (rest of filesystem: isolated, ephemeral)
│
├── Security monitor (watches container from host)
│   ├── Process watcher
│   ├── Filesystem watcher
│   └── Network watcher → ~/.coi/audit/<name>.jsonl
│
└── nftables (Coi-managed rules in the host FORWARD chain)
    └── rules applied per container IP
```

## What Coi Does Not Protect Against

Coi contains the blast radius of AI tool compromise. It does not prevent the AI from being tricked in the first place.

**Prompt injection** - Malicious content in files the AI reads (README, source code, web pages) can instruct the AI to take actions it otherwise would not. Coi limits what those actions can accomplish, but it cannot detect that an instruction is malicious.

**API key misuse** - If you give the AI an API key to use (e.g., AWS credentials to deploy something), a prompted AI can send those credentials to an attacker. Coi isolation only protects credentials that are not explicitly given to the AI.

**Insecure code generation** - Coi has no visibility into the quality or security of code the AI writes. Generated code may contain SQL injection, XSS, hardcoded secrets, or other vulnerabilities.

**Workspace file exfiltration** - The workspace is mounted read-write and the AI can read any file in it. In `restricted` and `allowlist` network modes, what the AI can do with those files externally is limited, but Coi does not prevent the AI from reading your source code.

For a full picture of what Coi protects and does not protect, see [Does Coi prevent prompt injection attacks?](FAQ-Security-and-Trust#does-coi-prevent-prompt-injection-attacks) in the FAQ.

## When Coi Is the Right Tool

Coi is well-suited when:

- You are running an AI tool with broad shell access (Claude Code, opencode, Aider)
- You are working in a codebase you did not write and have not fully audited
- You want AI help on a project without exposing your host credentials
- You run multiple AI sessions in parallel across different projects
- You want forensic logs of what an AI tool did during a session

Coi is probably overkill when:

- You are running a purely generative AI (no tool use, no shell access)
- You fully trust the codebase and have no sensitive credentials on your machine
- You only need filesystem isolation and have no threat model for network or supply-chain attacks

## See Also

- [Security Monitoring](Security-Monitoring) - Full event catalog, severity levels, and monitoring configuration
- [Network Isolation](Network-Isolation) - Configuring restricted, allowlist, and open network modes
- [Security Best Practices](Security-Best-Practices) - Recommended settings for different risk levels
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How ephemeral and persistent sessions work
- [FAQ](FAQ) - Common questions about Coi's security model and comparisons with other tools
