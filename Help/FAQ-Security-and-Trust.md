# FAQ: Security and Trust

Common questions about what Coi protects against, credential handling, and why Coi is trustworthy.

## Does Coi Prevent Prompt Injection Attacks?

**No**, Coi does not prevent prompt injection. What Coi does protect against:

- ✅ **Credential exposure** - Coi passes your SSH keys, host environment variables (except `TERM`) and credential files in only when you configure it. Exception: the AI tool's own credentials, copied in if they exist on the host, are readable by the AI tool and every process it starts (see [Limits and Non-Goals](Limits-and-Non-Goals#copied-tool-credentials-are-readable-by-the-agents-processes))
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
- During session setup, Coi copies the AI tool's essential config files into the container if they exist on the host
- The copied files belong to the user that runs the AI tool (on Coi images, `code`). So the AI tool and every process it starts can read the key (see [Limits and Non-Goals](Limits-and-Non-Goals#copied-tool-credentials-are-readable-by-the-agents-processes))
- **Financial blast radius:** For subscription models (e.g., Claude Pro/Max), worst case is exhausting your daily quota. For pay-per-token models, set spending caps on your API key to limit potential abuse

**If you are giving API keys to the AI for it to use** (e.g., AWS keys for the AI to deploy things):

- **Do not do this unless you fully trust the project and AI's capabilities**
- Coi isolation prevents credential leakage to your host, but the AI tool and every process it starts can read those credentials, so a compromised AI could misuse them or send them out
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

- Host SSH keys and env vars (except `TERM`) stay out unless configured; the AI tool and every process it starts can read the AI tool's copied credentials ([Limits and Non-Goals](Limits-and-Non-Goals#copied-tool-credentials-are-readable-by-the-agents-processes))
- Network isolation with nftables (blocks private networks by default)
- Workspace-only mounting (AI cannot access your entire filesystem)

**Active development** - Regular updates, responsive to issues, community-driven improvements.

## Is Coi Built Using AI Coding Agents?

**Yes - partially.** Coi's development process is partially agentic, not fully automated. Human developers remain in the loop for design decisions, code review, and merging, but AI coding agents assist with implementation, refactoring, and other tasks.

One of the more fitting aspects of this workflow: Coi is often built inside Coi itself. Developers use Coi containers to run AI coding agents while working on Coi, which means the tool is continuously dogfooded during its own development. This helps catch real-world issues early and keeps the developer experience grounded in actual use.

## Do You Need to Give Coi "Full Access"?

**The AI tools get limited access, but you, the host user who runs Coi, need root-equivalent access to Incus.** Here is what actually happens:

**Coi requires:**

- Incus permissions (you must be in the `incus-admin` group: see [Why Does Coi Need the `incus-admin` Group?](#why-does-coi-need-the-incus-admin-group))
- Access to your workspace directory (the project you are working on)
- Optional: passwordless sudo for `nft` for network isolation (you can decline it by setting `[network] use_sudo = false` and `mode = "open"` in config)

**What AI tools can access:**

- ✅ **Your workspace only** - The project directory you explicitly mount
- ✅ **Container filesystem** - Temporary files that get deleted (ephemeral mode)
- ✅ **The AI tool's own credentials** - Coi copies them in if they exist on the host; the AI tool and every process it starts can read them (see [Limits and Non-Goals](Limits-and-Non-Goals#copied-tool-credentials-are-readable-by-the-agents-processes))
- ❌ **Your SSH keys** - Not accessible unless you explicitly mount `~/.ssh`
- ❌ **Your home directory** - Not accessible
- ❌ **Your environment variables** - Not passed to the container (except `TERM`) unless explicitly forwarded via `forward_env` config
- ❌ **Your local network** - Blocked by default (restricted mode)

You control what gets mounted.

## Why Does Coi Need the `incus-admin` Group?

The `incus-admin` group gives full access to the Incus socket, and that is root-equivalent on the host. Incus says local socket access "always grants full access to Incus", including the ability to attach file system paths or devices to any instance and to change its security features, so you should only give it to users you would trust with root access to your system.

The `incus` group alone is not enough for Coi today:

- Users in the `incus` group get a restricted Incus project of their own, `user-<uid>`, and can use only that project. Coi uses the `default` project unless you set `[incus] project`, so its Incus check fails and Coi asks you to join `incus-admin`.
- Setting `[incus] project = "user-<uid>"` does not make a default Coi container work. That project forbids syscall interception, and Coi sets `security.syscalls.intercept.mknod` and `security.syscalls.intercept.setxattr` when Docker support is on, which is the default (`[container] docker`). Incus refuses the container.

For the layers that limit what the AI tool inside the container can reach, see [Architecture and Security Model](Architecture-and-Security-Model).

## See Also

- [Architecture and Security Model](Architecture-and-Security-Model) - Full explanation of Coi's threat model and defense layers
- [Security Monitoring](Security-Monitoring) - How Coi detects and responds to threats in real time
- [Security Best Practices](Security-Best-Practices) - Recommended settings for safe operation
- [Network Isolation](Network-Isolation) - Configuring network restrictions
- [FAQ](FAQ) - Return to the FAQ index
