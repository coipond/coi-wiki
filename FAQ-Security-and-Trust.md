# FAQ: Security and Trust

Common questions about what Coi protects against, credential handling, and why Coi is trustworthy.

## Does Coi Prevent Prompt Injection Attacks?

**No**, Coi does not prevent prompt injection. What Coi does protect against:

- ✅ **Credential exposure** - Your SSH keys, environment variables, and API tokens are not accessible to AI tools
- ✅ **Host system access** - AI tools cannot access your entire filesystem, only the mounted workspace
- ✅ **Lateral movement** - Network isolation prevents access to local network resources (in restricted mode)
- ✅ **Remote code execution blast radius** - Built-in nftables-based network filtering limits access to private networks and metadata services in restricted mode, and allowlist mode can constrain egress to approved domains/IPs, reducing (but not eliminating) data exfiltration and command-and-control risk
- ✅ **Persistent damage** - Ephemeral containers mean any malicious modifications are discarded
- ✅ **Real-time threat detection** - Security monitoring detects reverse shells, data exfiltration attempts, and malicious patterns, automatically pausing or killing the container (see [Security Monitoring](Security-Monitoring))

What Coi does not protect against:

- ❌ **Prompt injection** - Malicious prompts can still trick the AI into generating harmful code, but even if the AI goes rogue, damage is limited to your workspace by filesystem isolation and network controls
- ❌ **API key leakage via AI** - If you give the AI your API key, it could be prompted to send it elsewhere
- ❌ **Insecure code generation** - AI-generated code might have vulnerabilities (SQL injection, XSS, etc.)

**Best practices:**

- Review AI-generated code before committing
- Do not mount sensitive credentials into containers
- Use network isolation (restricted/allowlist modes) to limit data exfiltration
- Enable security monitoring (`[monitoring] enabled = true`) for untrusted projects
- Review audit logs after sessions: `cat ~/.coi/audit/<container-name>.jsonl`
- Commit AI changes with git hooks disabled (see [Security Best Practices](Security-Best-Practices))

## What About API Key Security?

**If the API key is for the AI tool itself** (e.g., Anthropic API key for Claude):

- Store it in your host `~/.claude/settings.json` or similar config
- Coi automatically copies essential config files from the host into the container during session setup
- The AI tool uses the key to authenticate, but it is not available to arbitrary commands in the container
- **Financial blast radius:** For subscription models (e.g., Claude Pro/Max), worst case is exhausting your daily quota. For pay-per-token models, set spending caps on your API key to limit potential abuse

**If you are giving API keys to the AI for it to use** (e.g., AWS keys for the AI to deploy things):

- **Do not do this unless you fully trust the project and AI's capabilities**
- Coi isolation prevents credential leakage to your host, but a compromised AI could still misuse those credentials
- Use temporary/scoped credentials with minimal permissions
- Prefer explicit mounting of credentials rather than storing them in the workspace

## Why Should You Trust This?

Fair question. Here is what makes Coi trustworthy:

**Open Source** - Full source code at [github.com/coipond/coi](https://github.com/coipond/coi) (MIT license):

- Review the code yourself
- Community can audit and contribute
- No hidden behavior

**Transparent architecture:**

- Uses standard Linux tools (Incus, tmux, systemd)
- No custom daemons or proprietary components
- Easy to inspect running containers (`incus list`, `incus exec`)

**Security by design:**

- Credentials isolated by default (not mounted unless you configure it)
- Network isolation with nftables (blocks private networks by default)
- Workspace-only mounting (AI cannot access your entire filesystem)

**Active development** - Regular updates, responsive to issues, community-driven improvements.

## Is Coi Built Using AI Coding Agents?

**Yes - partially.** Coi's development process is partially agentic, not fully automated. Human developers remain in the loop for design decisions, code review, and merging, but AI coding agents assist with implementation, refactoring, and other tasks.

One of the more fitting aspects of this workflow: Coi is often built inside Coi itself. Developers use Coi containers to run AI coding agents while working on Coi, which means the tool is continuously dogfooded during its own development. This helps catch real-world issues early and keeps the developer experience grounded in actual use.

## Do You Need to Give Coi "Full Access"?

**Coi itself does not need "full access" to anything.** Here is what actually happens:

**Coi requires:**

- Incus permissions (you must be in the `incus-admin` group)
- Access to your workspace directory (the project you are working on)
- Optional: passwordless sudo for `nft` for network isolation (you can decline it by setting `[network] use_sudo = false` and `mode = "open"` in config)

**What AI tools can access:**

- ✅ **Your workspace only** - The project directory you explicitly mount
- ✅ **Container filesystem** - Temporary files that get deleted (ephemeral mode)
- ❌ **Your SSH keys** - Not accessible unless you explicitly mount `~/.ssh`
- ❌ **Your home directory** - Not accessible
- ❌ **Your environment variables** - Not passed to the container unless explicitly forwarded via `forward_env` config
- ❌ **Your local network** - Blocked by default (restricted mode)

You control what gets mounted. By default, Coi is locked down.

## See Also

- [Architecture and Security Model](Architecture-and-Security-Model) - Full explanation of Coi's threat model and defense layers
- [Security Monitoring](Security-Monitoring) - How Coi detects and responds to threats in real time
- [Security Best Practices](Security-Best-Practices) - Recommended settings for safe operation
- [Network Isolation](Network-Isolation) - Configuring network restrictions
- [FAQ](FAQ) - Return to the FAQ index
