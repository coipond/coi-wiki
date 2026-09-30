Welcome to the Coi (Code on Incus) wiki. Coi is a session manager that runs AI coding tools (Claude Code, opencode, Codex, pi, omp, and more) inside isolated Incus system containers — with credential protection, network controls, and real-time security monitoring built in.

> **New to Coi?** Start with [Getting Started](Getting-Started) for a step-by-step walkthrough, then read [Architecture and Security Model](Architecture-and-Security-Model) to understand what Coi protects against and how.

## Documentation

### Getting Started
- **[Getting Started](Getting-Started)** - Step-by-step setup and first session walkthrough
- **[Architecture and Security Model](Architecture-and-Security-Model)** - What Coi protects against and how its defense layers work

### Setup and Installation
- **[Linux Setup Guide](Linux-Setup-Guide)** - Distro-specific setup (Arch, Fedora, openSUSE, Ubuntu)
- **[macOS Setup Guide](macOS-Setup-Guide)** - Running Coi on macOS with Colima/Lima

### Configuration and Usage
- **[Best Practices](Best-Practices)** - Session modes, network selection, team workflows, and storage management
- **[Configuration](Configuration)** - Config files, hierarchy, per-repo `.coi/config.toml`, full reference, and environment variables
- **[Profiles](Profiles)** - Self-contained profile directories with build scripts, context files, and full config bundling
- **[Supported Tools](Supported-Tools)** - Claude Code, opencode, Codex, pi, omp, and tool selection
  - **[Sandbox Context](Sandbox-Context)** - The `~/SANDBOX_CONTEXT.md` / `.json` environment description and auto-context injection
  - **[Adding New Tools](Adding-New-Tools)** - Implement the `Tool` interface to add a new AI assistant
- **[Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions)** - Understanding how containers and sessions work
- **[Container Operations](Container-Operations)** - Container management and low-level operations
- **[Snapshot Management](Snapshot-Management)** - Create checkpoints, rollback, and branch experiments
- **[File Transfer](File-Transfer)** - Push/pull files between host and containers
- **[Port Publishing](Port-Publishing)** - Publish container TCP ports on the host so agent-started services are reachable at `localhost:<port>`
- **[Tmux Automation](Tmux-Automation)** - Automate AI sessions with tmux commands
- **[Headless Orchestration](Headless-Orchestration)** - Drive any supported tool from an external orchestrator with `coi tool spec`
- **[Image Management](Image-Management)** - Create and manage custom images
- **[Resource and Time Limits](Resource-and-Time-Limits)** - Control container resource consumption and runtime
- **[Resource Usage (`coi top`)](Resource-Usage)** - Live CPU, memory, disk, and network usage per container and process

### Security
- **[Threat Model: Containment Limits](Threat-Model-Containment-Limits)** - What the shared-kernel boundary does and does not contain, and the kernel attack-surface hardening flags
- **[Security Monitoring](Security-Monitoring)** - Real-time threat detection and automated response (the detection engine)
- **[Audit Log](Audit-Log)** - On-disk audit log format, field reference, and `coi audit` live streaming
- **[Session Logs](Session-Logs)** - Coi operational logs and `coi logs` command
- **[Security Best Practices](Security-Best-Practices)** - Git hooks protection, path security, and safe commit practices
- **[Network Isolation](Network-Isolation)** - Network security modes and egress hardening
  - **[Static Host Entries](Static-Host-Entries)** - Custom `/etc/hosts` name→IP mappings with mode-aware firewall reachability
  - **[nftables Setup](nftables-Setup)** - Installing nftables + the sudoers rule for restricted/allowlist modes

### Maintenance
- **[Updating Coi](Updating-Coi)** - Keep Coi current with `coi update` (binary + detection databases)
- **[System Health Check](System-Health-Check)** - Diagnose setup issues and verify configuration
- **[Migration Guide](Migration-Guide)** - Breaking changes and upgrade steps between versions ([Upgrading from 0.11 to 0.12](Migration-Guide#upgrading-from-011-to-012), [Upgrading from 0.10.1 to 0.11.0](Migration-Guide#upgrading-from-0101-to-0110), [Upgrading from 0.9 to 0.10](Migration-Guide#upgrading-from-09-to-010), [Upgrading from 0.8 to 0.9](Migration-Guide#upgrading-from-08-to-09))

### Help and Reference
- **[FAQ](FAQ)** - Frequently Asked Questions about Coi
- **[FAQ: Platform Comparisons](FAQ-Platform-Comparisons)** - How Coi differs from Docker, DevContainers, Distrobox, and VMs
- **[FAQ: Security and Trust](FAQ-Security-and-Trust)** - Credential handling, threat model, and trustworthiness
- **[FAQ: Setup and Operation](FAQ-Setup-and-Operation)** - Platform support, Docker-in-Coi, local AI models
- **[Troubleshooting](Troubleshooting)** - Common issues and solutions
