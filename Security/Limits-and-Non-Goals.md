# Limits and Non-Goals

Coi isolates an AI coding agent from your host. Inside that boundary, the agent can still act, and an allowed action is a real action. This page lists what Coi does not protect with its default settings, and what it does not try to do at all. Use it to decide whether Coi alone is enough for your threat model.

For the shared-kernel risk, see [Threat Model: Containment Limits](Threat-Model-Containment-Limits). For prompt injection, API key misuse, insecure code and workspace reads, see [What Coi Does Not Protect Against](Architecture-and-Security-Model#what-coi-does-not-protect-against).

## Non-Goals

Coi does not try to:

- **Prevent prompt injection.** Coi limits what a tricked agent can do; it cannot tell that an instruction is malicious ([Architecture and Security Model](Architecture-and-Security-Model#what-coi-does-not-protect-against)).
- **Resist kernel exploits.** `reduce_kernel_surface` shrinks the reachable kernel surface, but a kernel 0-day still defeats containment ([Threat Model: Containment Limits](Threat-Model-Containment-Limits#what-reduce_kernel_surface-does--and-does-not)).

## Allowed Actions Can Still Cause Harm

### Writes to the Workspace Are Real

The workspace is mounted read-write. Only the paths in `protected_paths` (and `additional_protected_paths`) are read-only. The agent can change or delete every other file, and the change lands on your host.

Files that host tools run on their own are not protected unless you add them. Examples: `.envrc`, `.pre-commit-config.yaml`. Review the diff before you run host tools in the workspace. To protect more paths, see [Configuring Protected Paths](Security-Best-Practices#configuring-protected-paths).

### An Allowed Destination Can Receive Anything the Agent Can Read

In `restricted` mode (the default), Coi blocks private networks and metadata endpoints and accepts all other traffic. Any public host can receive any file or credential the agent can read. Without `allowed_ports`, all ports are open. Only `allowlist` mode ends in a default reject. See [Network Isolation](Network-Isolation).

### DNS Still Resolves in `restricted` Mode

`dns_servers` pins off-box DNS to the listed resolvers. It does not filter queries to the bridge's own resolver, so a lookup through the normal container DNS still works. Only `allowlist` mode blocks all DNS. See [DNS Resolver Pinning](Network-Isolation#dns-resolver-pinning-dns_servers).

## Protections Coi Does Not Provide by Default

### Copied Tool Credentials Are Readable by the Agent's Processes

Coi does not pass host environment variables into the container unless you list them in `forward_env`. But it copies the AI tool's own credential files in (for Claude Code, `~/.claude/.credentials.json`) and gives them to the `code` user. The agent, and every command it starts, runs as that user. The `code` user also has passwordless sudo for all commands in the Coi image. So any process the agent starts can read these credentials, and the same applies to every `forward_env`, `[[credentials]]` and `environment` value.

### Permission Prompts Are Bypassed

The AI tool runs with its permission prompts bypassed by default (`permission_mode = "bypass"`; for Claude Code, `--permission-mode bypassPermissions`). Nobody approves a command before it runs. To approve commands yourself, set `permission_mode = "interactive"` ([Permission Mode](Supported-Tools#permission-mode)).

### Monitoring Is Off by Default and Only Matches Patterns

`[monitoring] enabled` is `false` by default, so a default session has no threat detection. When you turn it on, it matches process command lines, file activity and network patterns against known attack shapes. A payload that does not match a pattern is not detected. Treat monitoring as a detective control, not a guarantee. See [Security Monitoring](Security-Monitoring).

## Checklist

Before you trust Coi with a repository you did not write:

- [ ] Use `allowlist` mode, or at least set `allowed_ports`, if the agent must not reach arbitrary hosts.
- [ ] Turn on `[monitoring]` (and `[monitoring.nft]` for network detection).
- [ ] Assume the tool credentials in the container can leak. Use a credential you can revoke.
- [ ] Add the workspace files your host tools run to `additional_protected_paths`.
- [ ] Review workspace changes before you run them on the host.
- [ ] For an adversarial agent, put a VM or a dedicated host under Coi ([When a Container Is Not Enough](Threat-Model-Containment-Limits#when-a-container-is-not-enough)).

## See Also

- [Threat Model: Containment Limits](Threat-Model-Containment-Limits) - The shared-kernel residual risk
- [Architecture and Security Model](Architecture-and-Security-Model) - How the isolation layers fit together
- [Network Isolation](Network-Isolation) - `restricted`, `allowlist` and `open` modes, `dns_servers`, `allowed_ports`
- [Security Monitoring](Security-Monitoring) - What monitoring detects and how it responds
- [Security Best Practices](Security-Best-Practices) - Protected paths and safe commit practices
- [Profiles](Profiles) - The built-in `hardened` profile
