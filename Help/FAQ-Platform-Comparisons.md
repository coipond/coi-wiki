# FAQ: Platform Comparisons

Common questions about how Coi compares to other isolation and containerization approaches.

## Why Not Just Use Docker With a Volume?

A Docker container with a volume mount gives you filesystem isolation - but that is only one layer of defense, and not the most important one for AI agents.

| Capability | Docker + Volume | Coi |
|------------|----------------|-----|
| **Filesystem isolation** | Yes | Yes |
| **Credential isolation** | Manual (you decide what to mount) | Automatic (nothing exposed by default) |
| **Reverse shell detection** | No | Yes (real-time process monitoring) |
| **Data exfiltration detection** | No | Yes (filesystem I/O + network monitoring) |
| **Network filtering** | Basic (`--network none` or full access) | Granular (restricted/allowlist/open modes) |
| **C2 port blocking** | No | Yes (nftables blocks known C2 ports) |
| **Cloud metadata protection** | No | Yes (169.254.169.254 blocked) |
| **Supply-chain attack prevention** | No | Yes (`.git/hooks`, `.vscode`, `.husky` read-only) |
| **Automated threat response** | No | Yes (pause on HIGH, kill on CRITICAL) |
| **Audit logging** | No | Yes (JSONL forensic logs) |
| **Session resume** | No | Yes (conversation history + credentials restored) |
| **File permission handling** | Manual UID mapping | Automatic UID mapping (idmapped mounts or raw.idmap, auto-selected) |
| **Docker-in-Docker** | Requires `--privileged` | Works unprivileged (systemd + nesting) |

Docker gives you a box. Coi gives you a box with [security monitoring](Security-Monitoring) - real-time kernel-level threat detection, automated response, and forensic audit logging.

Additionally, Docker application containers run a single process without an init system. Coi uses Incus system containers with full systemd, which means Docker, systemd services, and other system-level tools work natively inside the container without privileged mode.

## How Is Coi Different From Docker Sandboxes?

Docker Sandboxes is a Docker Desktop feature that uses microVMs for isolation on macOS/Windows. On Linux, it falls back to traditional containers. Coi is built specifically for Linux using Incus system containers:

- **No Docker Desktop needed** - Coi uses Incus (fully open source), while Docker Sandboxes requires Docker Desktop (not open source, commercial licensing for organizations)
- **System containers, not microVMs** - One clean isolation layer vs. containers-in-VMs complexity
- **Linux-first design** - Built for Linux from day one, not as an afterthought

Beyond architectural differences, Coi includes a [security monitoring](Security-Monitoring) layer that Docker Sandboxes lack entirely. Coi monitors processes for reverse shells and credential scanning, tracks filesystem I/O for data exfiltration, and uses kernel-level nftables rules to detect and block suspicious network connections (C2 ports, cloud metadata endpoints, private network access). Threats trigger automated responses - HIGH severity events pause the container, CRITICAL events kill it - with all events logged to JSONL audit files for forensic review.

## How Is Coi Different From DevContainers?

**Purpose:** DevContainers are for setting up development environments. Coi is for securely running AI coding tools that need broad system access.

**Security model:**

- **DevContainers** - Your code runs in the container, but typically with your host credentials mounted
- **Coi** - AI tools run in isolated containers without your credentials. Only your workspace is mounted, nothing else unless explicitly configured

**Architecture:**

- **DevContainers** - Application containers (Docker) without init systems
- **Coi** - System containers (Incus) with full systemd, can run Docker inside

Coi also includes [security monitoring](Security-Monitoring) that does not exist in the DevContainers ecosystem. Real-time process, filesystem, and kernel-level network monitoring detect threats like reverse shells, data exfiltration, and C2 connections - with automated container pause/kill responses based on threat severity.

## How Is Coi Different From Distrobox?

[Distrobox](https://github.com/89luca89/distrobox) is designed to feel like you are not in a container. It shares the host's home directory, network stack, display server, and often the entire filesystem. This is great for running desktop applications from different distros, but it is the opposite of what you want for an AI coding agent.

| Aspect | Distrobox | Coi |
|--------|-----------|-----|
| **Home directory** | Shared with host | Isolated per slot |
| **Network** | Shared with host | Isolated (restricted/allowlist/open) |
| **Host filesystem** | Broadly accessible | Only workspace mounted |
| **Credentials** | Fully exposed | Never exposed by default |
| **Security monitoring** | None | Real-time threat detection |
| **Purpose** | Run apps from other distros | Secure AI agent isolation |

Running an AI agent in Distrobox is essentially the same as running it directly on your host - it can access your SSH keys, read your `.env` files, reach your local network, and modify git hooks. Coi isolates all of these by default and adds [security monitoring](Security-Monitoring) to detect malicious behavior in real time.

## Can I Use Lima/limactl or a Plain VM Instead of Coi?

You can run AI tools inside any Linux VM, but a plain VM provides only isolation - no security monitoring, no credential protection, no session management.

| Capability | Plain VM (Lima/etc.) | Coi |
|------------|---------------------|-----|
| **Filesystem isolation** | Yes | Yes |
| **Reverse shell detection** | No | Yes |
| **Data exfiltration monitoring** | No | Yes |
| **Network threat filtering** | Manual firewall setup | Automatic (nftables) |
| **Credential isolation** | Manual | Automatic |
| **Supply-chain protection** | No | Yes (read-only `.git/hooks`, `.vscode`) |
| **Session resume** | No | Yes |
| **Audit logging** | No | Yes |
| **Startup time** | 30-60 seconds | 1-2 seconds |
| **Resource overhead** | Full VM (kernel + OS) | Container (shared kernel) |

Coi's value is not just isolation - it is the [security monitoring](Security-Monitoring) layer that detects and responds to threats in real time. A VM is a locked room; Coi is a locked room with security cameras, motion sensors, and an automated response system.

## Quick Comparison: Coi vs. Other Tools

| Tool | Purpose | Credentials Isolated | Security Monitoring | AI Tool Support | Session Management | Best For |
|------|---------|---------------------|---------------------|-----------------|-------------------|----------|
| **Coi** | AI coding isolation | ✅ Yes (default) | ✅ Yes (process, filesystem, network) | Built-in | Auto save/resume | Running AI coding tools securely |
| **Docker + Volume** | Basic isolation | ⚠️ Manual | ❌ No | Manual | Manual | Simple filesystem isolation |
| **Docker Sandboxes** | AI tool isolation | ✅ Yes | ❌ No | Limited | Manual | macOS/Windows users (requires Docker Desktop) |
| **DevContainers** | Dev environment | ❌ No (typically mounted) | ❌ No | Manual | Manual | Reproducible development environments |
| **Distrobox** | Desktop apps/dev | ❌ No (shares home) | ❌ No | Manual | Manual | Running apps from different distros |
| **Plain VM** | General isolation | ⚠️ Manual | ❌ No | Manual | Manual | Full OS isolation without monitoring |
| **Bare metal** | Direct execution | ❌ No (full access) | ❌ No | Manual | None | Maximum performance, trusted environments |

**Choose Coi if:** You want AI to modify code without credential exposure, with real-time [security monitoring](Security-Monitoring), automatic session management, and strong isolation.

## See Also

- [Architecture and Security Model](Architecture-and-Security-Model) - How Coi's defense layers work and what threat model it addresses
- [Security Monitoring](Security-Monitoring) - Real-time threat detection and automated response
- [Network Isolation](Network-Isolation) - Configuring network modes for different risk levels
- [FAQ](FAQ) - Return to the FAQ index
