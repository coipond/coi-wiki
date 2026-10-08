# Security Best Practices

Coi provides multiple layers of security to protect your host system from potentially malicious code generated or modified by AI tools.

## Automatic Path Protection (Default)

Coi automatically mounts security-sensitive paths as read-only by default. This prevents containers from modifying files that could execute automatically on your host system.

### Default Protected Paths

| Path | Risk | Protection |
|------|------|------------|
| `.git/hooks` | Git hooks execute on commits, pushes, etc. | Read-only mount |
| `.git/config` | Can set `core.hooksPath` to bypass hooks protection | Read-only mount |
| `.git/config.worktree` | Per-worktree config sink (with `extensions.worktreeConfig`) — can plant filter/diff/textconv drivers or `core.hooksPath` | Read-only mount |
| `.git/info/attributes` | Can name filter/textconv drivers that run on the host's next git operation | Read-only mount |
| `.husky` | Husky git hooks manager | Read-only mount |
| `.vscode` | VS Code `tasks.json` can auto-execute, `settings.json` can inject shell args | Read-only mount |
| `.coi` | Project Coi config — an agent could otherwise weaken the next session's settings | Read-only mount |
| `.claude/settings.json` | Claude Code project settings carry `hooks` the host auto-executes | Read-only mount |
| `.claude/settings.local.json` | Same auto-executed `hooks` risk as `settings.json` | Read-only mount |

In addition to this static list, Coi dynamically discovers and protects each existing per-worktree git config file (`.git/worktrees/<name>/config.worktree`) at session setup, since those are config sinks too when `extensions.worktreeConfig` is enabled.

### How It Works

When you start a session, Coi:
1. Detects which protected paths exist in your workspace
2. Mounts them as separate read-only devices over the workspace mount
3. Reports which paths were protected in the startup output

```text
Protected paths (mounted read-only): .git/hooks, .git/config, .vscode
```

### Host-Side Immutable Protection

In addition to read-only mounts, Coi applies the Linux immutable attribute (`chattr +i`) on protected paths on the host during sessions. This prevents the `unshare -m` + `umount` bypass where a container process could remount a read-only path as writable.

**Key points:**

- Immutable bits are automatically cleared on session stop/cleanup
- Requires `CAP_LINUX_IMMUTABLE` on the `coi` binary (granted by `install.sh`)
- Manual install: `sudo setcap cap_linux_immutable=ep "$(readlink -f "$(which coi)")"`
- macOS/Colima/Lima: Graceful degradation (shared filesystems do not support immutable)
- Opt out: `[security] host_immutable = false`

### Guest API Disabled

The Incus guest API (`/dev/incus`) is disabled on all Coi containers via `security.guestapi=false`. This prevents container processes from querying host source paths via the device topology API, which would leak the host username and workspace layout.

### Opting Out

If you need the AI to manage git hooks or other protected paths:

```toml
# Via trusted-scope config: ~/.coi/config.toml (or the file $COI_CONFIG points at)
[git]
writable_hooks = true
```

Protection-weakening settings are honored only from trusted-scope config. A project config (`.coi/config.toml` inside the workspace) cannot weaken protections: Coi's untrusted-config sanitizer strips keys like `writable_hooks = true` from it with a warning, since a cloned repo must not be able to disable its own read-only protection.

## Configuring Protected Paths

You can customize which paths are protected via the `[security]` config section.

### Add Additional Paths

Protect additional paths without replacing the defaults:

```toml
# ~/.coi/config.toml or .coi/config.toml
[security]
additional_protected_paths = [".idea", "Makefile", ".gradle"]
```

### Replace Default Paths

Replace the default list entirely (use with caution):

```toml
[security]
protected_paths = [".git/hooks"]  # Only protect .git/hooks
```

**Scope note:** because replacing the list can drop defaults, `protected_paths` is honored only from trusted-scope config (`~/.coi/config.toml` or `$COI_CONFIG`). It is stripped, with a warning, from a cloned repo's `.coi/config.toml`; project configs can only extend protection via `additional_protected_paths`.

### Disable Protection

> **Warning:** Disabling protection removes all `.git/hooks`, `.vscode`, and `.husky` read-only enforcement. An AI tool operating in a compromised project can then modify your git hooks to execute arbitrary code on every commit. Only use this if you fully understand and accept the supply-chain risk.

Not recommended, but available if needed:

```toml
[security]
disable_protection = true
```

**Scope note:** `disable_protection` is honored only from trusted-scope config (`~/.coi/config.toml` or `$COI_CONFIG`). It is stripped, with a warning, from a cloned repo's `.coi/config.toml`.

## Committing AI-Generated Code

Even with automatic protection, it is good practice to disable hooks when committing AI-generated code as an extra safety layer:

```bash
# Commit with hooks disabled
git -c core.hooksPath=/dev/null commit --no-verify -m "your message"

# Create an alias for convenience
alias gcs='git -c core.hooksPath=/dev/null commit --no-verify'
```

### Why This Extra Step?

- **Belt and suspenders**: Multiple layers of protection are better than one
- **Covers edge cases**: Protects against scenarios where protection might be disabled
- **Good habit**: Useful even outside of Coi when reviewing AI-generated code

### Full Protection Command

Disable both hooks and git attribute filters:

```bash
git -c core.hooksPath=/dev/null -c core.attributesFile=/dev/null commit --no-verify -m "msg"

# As a shell function
safe_commit() {
    git -c core.hooksPath=/dev/null -c core.attributesFile=/dev/null commit --no-verify "$@"
}
```

## Attack Vectors Coi Protects Against

### Git Hooks Injection

**Risk**: AI modifies `.git/hooks/pre-commit` to execute malicious code on your next commit.

**Protection**: `.git/hooks` is mounted read-only by default.

### Git Config core.hooksPath Bypass

**Risk**: AI sets `core.hooksPath` in `.git/config` to point to a malicious hooks directory elsewhere in the workspace.

**Protection**: `.git/config` is mounted read-only by default.

### Husky Hooks Manipulation

**Risk**: AI modifies `.husky/pre-commit` or other husky hooks.

**Protection**: `.husky` directory is mounted read-only by default.

### VS Code Task Injection

**Risk**: AI creates/modifies `.vscode/tasks.json` with tasks that auto-execute when you open the project in VS Code.

**Protection**: `.vscode` directory is mounted read-only by default.

### VS Code Settings Injection

**Risk**: AI modifies `.vscode/settings.json` to inject malicious shell arguments via `terminal.integrated.shellArgs`.

**Protection**: `.vscode` directory is mounted read-only by default.

## Git Identity Guard

Containers set `git config --global user.useConfigOnly true` during setup, which forces git to refuse commits until `user.name` and `user.email` are explicitly configured. This prevents AI tools from accidentally committing as the container's default "code" user.

Coi then seeds a real identity up front: it reads the host's global `git config --global user.name` / `user.email` (never project-local config) and writes that identity into the container's global git config. On macOS, where coi runs inside a Colima/Lima/OrbStack VM, the host is your Mac: the identity comes from the Mac's gitconfig in the shared home, falling back to the VM's (see [macOS Setup Guide](macOS-Setup-Guide#git-identity-on-macos)). This is configurable via the `[git]` block — `name` / `email` pin an explicit container identity that overrides the host git config, and `seed_host_identity = false` disables the seeding (keeping only the fail-closed guard). All these keys are trusted-scope only — they are stripped from a project's `.coi/config.toml`, so an untrusted checkout cannot choose the commit author.

**Locking the identity (`readonly = true`).** The seeded identity lives in the container's writable `~/.gitconfig`, and a config file alone loses to three override paths: rewriting the file (`git config --global`), `git -c user.name=… commit` / `git commit --author=…`, and an agent exporting its own `GIT_AUTHOR_*`. So `readonly = true` locks the identity with three layers:

- **Read-only mount** — the identity is mounted read-only at `~/.gitconfig`, so `git config --global` fails (a rename over a mount point cannot replace it — which is how `git config`'s lock-file + rename write is blocked).
- **Pinned environment** — `GIT_AUTHOR_*`/`GIT_COMMITTER_*` are set as container-level env, which takes precedence over `user.*` config and so defeats `git -c user.*`.
- **Post-commit re-stamp** — a root-owned `post-commit` hook rewrites any commit whose author or committer name/email is not the locked identity, correcting `git commit --author=…` and agent-exported `GIT_*`. Being a post-commit hook (not a verify hook), it also survives `git commit --no-verify`.

Notes:

- It locks the whole global gitconfig, so any `git config --global …` fails; use per-repo `--local` config for other settings.
- Only takes effect with a resolvable identity (explicit `name`/`email`, or a seeded host identity). If it cannot be applied, the session fails closed rather than silently falling back to a writable identity.
- Trusted-scope only, like the rest of `[git]`; default `false`.
- **Residual gaps** (documented, accepted): a repo whose local config sets `core.hooksPath` (e.g. husky) replaces the global hooks dir, so the re-stamp does not run there; and an agent that exports `GIT_CONFIG_GLOBAL` bypasses the mounted gitconfig entirely (the env layer still forces the committer). Full enforcement against a deliberately adversarial agent is not achievable in-container.

When no identity was seeded (the host has none configured, or seeding is disabled), the sandbox context file (`~/SANDBOX_CONTEXT.md`) tells the AI tool to first check whether Coi already configured `user.name`/`user.email`, and only then fall back to a priority-ordered discovery sequence:

1. **SSH agent** - parse the username from the SSH greeting, look up via platform API
2. **GitHub CLI** - `gh api user` for name/email
3. **Git log** - reuse the most recent commit's author
4. **Ask the user** - if none of the above works

This ensures commits always carry the real developer's identity, not a fabricated one.

### Clean Commit Authorship (`strip_attribution`)

AI agents commonly auto-inject attribution into commit messages — a `Co-Authored-By: <tool bot>` trailer and/or a "Generated with …" footer. Git has no setting that forbids a trailer, and per-tool opt-outs are easy to miss, so Coi enforces clean authorship tool-agnostically (on by default, `[git] strip_attribution = false` to opt out):

- A global `commit-msg` hook (`core.hooksPath` → `/etc/coi/git-hooks`, root-owned so the sandboxed agent cannot rewrite its own policy) strips matching lines from every commit message, whichever tool made the commit. It strips, never rejects — an autonomous session cannot break over a cosmetic trailer — and then delegates to the repository's own hooks, so husky/lint pre-commit flows keep working (a repo hook's rejection still rejects).
- For Claude Code the policy is additionally enforced at the source: `includeCoAuthoredBy = false` in the managed-settings tier that no in-session setting can override.
- The default patterns cover Claude/Codex/Copilot/Gemini/aider/Cursor trailers, `[bot]`/noreply identities, and "Generated with/by" footers — a human `Co-Authored-By: Jane Doe <jane@corp.example>` is untouched. `strip_attribution_patterns` replaces the list wholesale (grep -E, matched per line).
- Trusted-scope only, in both directions: a cloned repo can neither re-enable attribution you strip nor choose arbitrary line-deletion patterns for your commit messages.

**Known limitations** (accepted and pinned by an integration test): a repo whose local git config sets `core.hooksPath` — husky writes `core.hooksPath = .husky` into `.git/config` — overrides the global hook (local beats global in git's config precedence), so the strip does not run in such repos; and `git commit --no-verify` skips `commit-msg` hooks entirely. Both cases are still covered for Claude Code by the managed-settings layer.

### Protected Branches (`protected_branches`)

By default the agent can't commit on, or push to, `main` or `master`: it works on a feature branch and opens a pull request, so its changes always get reviewed. Root-owned git hooks refuse commits on a protected branch, moving the branch to a commit the remote doesn't already have (cherry-pick, merge, rebase, reset, …), and pushes to it; `git pull` keeps working. Configure or disable the list with `[git] protected_branches` in your own config — a cloned repo can't change it. The hooks catch accidents, not a determined workaround (`--no-verify`, a repo-local hooks path, and a few git commands get around them), so keep server-side branch protection on as well. Details: [Configuration → Protected Branches](Configuration#protected-branches).

## Symlink Security

Coi rejects symlinked protected paths to prevent attacks where a symlink could trick Coi into mounting arbitrary host paths as read-only (or failing to protect the real path).

- Linked git worktrees (`.git` as a file pointing at git internals outside the workspace) are supported: Coi resolves the external gitdir and common dir, mounts the common dir so git works, and re-covers its auto-executing subpaths (`hooks`, `config`, `info/attributes`, `worktrees/*/config.worktree`) with the same read-only protection a normal repo's `.git` gets — synthesizing empty read-only placeholders for absent ones
- A bidirectional-link guard rejects malicious `gitdir:` pointers (e.g. an untrusted repo pointing at `~/.ssh`): the external dir is mounted only if its `gitdir` back-pointer resolves to this workspace and it is a real object store; on any guard failure Coi fails closed (nothing mounted)
- When `.git` indirection still cannot be handled (e.g. `.git` is a symlink, or a submodule's `.git` file), Coi surfaces a setup warning instead of silently skipping git path protection, and continues protecting unrelated paths like `.vscode`
- Symlinked protected paths like `.vscode -> /etc/passwd` are rejected

## Permission Mode

By default, Coi bypasses all tool permission prompts so sessions run autonomously. For higher-security workflows, you can enable interactive mode so the tool asks before running each command:

```toml
# .coi/config.toml or ~/.coi/config.toml
[tool]
permission_mode = "interactive"
```

This affects both Claude Code and opencode - see the [Supported Tools](Supported-Tools#permission-mode) page for details on how each tool behaves in interactive mode.

## SSH Agent and Environment Variable Forwarding

Coi provides opt-in mechanisms to selectively share host resources with containers:

- **SSH agent forwarding** (`[ssh] forward_agent = true`) - Bridges the host's SSH agent socket into the container. The container can use your SSH keys for git operations without the keys themselves being copied. Disabled by default.
- **Environment variable forwarding** (`forward_env` in config) - Forwards specific host env vars by name. Values are read at session start and never stored in config.

**Security considerations:**

- Only enable SSH agent forwarding when you need git-over-SSH inside the container
- Only forward the minimum set of environment variables needed (e.g., API keys for the AI tool)
- Forwarded env vars are accessible to all processes in the container, including the AI tool
- For maximum isolation, prefer passing API keys via the tool's config file rather than env vars

## Network Isolation

In addition to path protection, Coi provides [network isolation](Network-Isolation) to prevent data exfiltration:

- **Restricted mode** (default): Blocks access to private networks (RFC1918)
- **Allowlist mode**: Only allows access to specific domains
- **Open mode**: No restrictions (use only for trusted projects)

## Real-Time Security Monitoring

In addition to passive protections, Coi includes active [Security Monitoring](Security-Monitoring) that:

- **Detects threats in real-time**: Reverse shells, credential scanning, data exfiltration
- **Responds automatically**: Pause or kill containers based on threat severity
- **Logs everything**: Audit trail in JSON Lines format for post-session review

Enable monitoring in config:

```toml
[monitoring]
enabled = true
```

See the [Security Monitoring](Security-Monitoring) page for full details.

## Summary

Coi's defense-in-depth approach:

1. **Container isolation**: AI runs in an isolated Incus container
2. **Privileged container guard**: Coi refuses to start when `security.privileged=true` is detected - this setting silently defeats all container isolation (seccomp, AppArmor, UID mapping)
3. **Security posture verification**: `coi health` checks that seccomp and AppArmor are active and warns on custom overrides (`raw.seccomp`, `raw.apparmor`)
4. **Kernel version enforcement**: Warns on host kernels below 5.15 that may lack security features for safe container isolation
5. **Path protection**: Security-sensitive paths mounted read-only
6. **Host-side immutable protection**: Protected paths locked with `chattr +i` to prevent `unshare -m` + `umount` bypass
7. **Guest API disabled**: Incus guest API (`/dev/incus`) disabled to prevent host topology leaks
8. **Git identity guard**: `user.useConfigOnly=true` prevents commits without a real identity
9. **Credential isolation**: SSH keys and env vars never exposed unless explicitly opted in
10. **Permission mode**: Optional human-in-the-loop approval for tool commands
11. **Network isolation**: Prevents unauthorized network access
12. **Security monitoring**: Real-time threat detection and automated response
13. **Sandbox context**: AI tools know their constraints via `~/SANDBOX_CONTEXT.md`
14. **Safe commit practices**: Extra protection when committing changes

These layers work together to minimize the blast radius if an AI tool is compromised or generates malicious code.

## See Also

- [Security Monitoring](Security-Monitoring) - Real-time threat detection and automated response
- [Network Isolation](Network-Isolation) - Configuring container network restrictions
- [FAQ](FAQ) - Common security questions and comparisons with other tools
- [Configuration](Configuration) - Security-related configuration options
