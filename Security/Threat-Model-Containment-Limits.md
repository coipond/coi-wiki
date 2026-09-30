# Threat Model: Containment Limits

Coi isolates AI coding agents in unprivileged Incus system containers. That boundary is strong against the common failure modes (credential theft, workspace tampering, network exfiltration) — but it is a shared-kernel boundary, and what that means deserves an honest account.

## The Shared-Kernel Residual Risk

Every Coi container runs on the host's kernel. A kernel vulnerability reachable from inside the container defeats every container-level control at once — protected paths, secret masks, network rules, monitoring.

This is not hypothetical. Trail of Bits' ["VMs won't contain cyber-capable agents"](https://blog.trailofbits.com/2026/08/26/vms-wont-contain-cyber-capable-agents/) (Aug 2026) demonstrated a cyber-capable model escaping full QEMU/KVM virtual machines — a stronger boundary than containers — three times over, by autonomously discovering and chaining kernel and hypervisor bugs across ~12-hour runs. Two of their findings shape Coi's guidance:

1. **Attack surface is the escape surface.** Every kernel feature reachable from inside the sandbox is a potential way out. Disable what you do not use.
2. **Distribution lag is a vulnerability.** Patches land upstream faster than stable distributions backport them. An "old but supported" kernel means months of publicly-known, unpatched bugs.

## What `reduce_kernel_surface` Does — and Does Not

```toml
[security]
reduce_kernel_surface = true   # trusted scope only (~/.coi/config.toml)
```

When enabled (it is on in the built-in `hardened` profile):

- **Docker/nesting support is turned off** — `security.nesting`, the mknod/setxattr syscall interceptions, and the unprivileged low-port sysctl are all removed. Docker-in-container will not work; that is the point.
- **High-risk syscall families are denied** via `security.syscalls.deny`: `io_uring_setup/enter/register`, `bpf`, `userfaultfd`, `keyctl/add_key/request_key`. These are the families behind the majority of recent Linux privilege-escalation and container-escape chains. Denied syscalls return `EPERM`; normal agent work is unaffected (Node/libuv detects the io_uring denial and falls back to its thread pool — the same set Docker's default seccomp profile has blocked for years).

The policy is applied fail-closed: it is set before the container's first boot, re-converged whenever a persistent container restarts, and verified against the effective (profile-inherited) Incus config. If an attached Incus profile pins a key the policy needs off — the classic Docker-in-Incus recipe of `security.nesting=true` on the default profile — Coi refuses to launch rather than silently booting with the hardening defeated; the error names the pinned key and the remedy.

### Strict Tier: Also Deny `perf_event_open`

```toml
[security]
reduce_kernel_surface_strict = true   # implies reduce_kernel_surface
```

The strict tier adds `perf_event_open` — another long-standing kernel-LPE vector — to the deny list. It is a separate opt-in because, unlike the base families (which have transparent in-container fallbacks), denying `perf_event_open` removes real capability: kernel-level profiling (`perf`, the JVM async-profiler's perf mode) stops working. It still degrades gracefully — the syscall returns `EPERM`, it does not crash the process — and ordinary agent work is unaffected (git, Node `--prof`, py-spy, and Go pprof do not use it). Enabling strict implies the base tier.

If you only want Docker off without the deny list:

```toml
[container]
docker = false    # honored from any scope (it only tightens)
```

**What it does NOT do:** it does not make the kernel boundary safe against a determined, cyber-capable agent. It shrinks the reachable surface; hundreds of syscalls remain, and any one kernel 0-day among them is game over for containment. Treat it as risk reduction, not risk elimination.

An untrusted (project-scoped) `.coi/config.toml` can set `docker = false` but can never set `docker = true` or flip `reduce_kernel_surface` — a cloned repo must not widen the kernel surface your trusted config narrowed.

## Freshness: Run `coi health`

`coi health` now reports distribution-lag signals:

- **Kernel build age** — warns when the running kernel was built more than 6 months ago.
- **Kernel mitigations** — warns when CPU side-channel mitigations are disabled on the kernel command line (`mitigations=off` or per-vulnerability disables like `nopti`, `mds=off`). The Trail of Bits escape host ran exactly `mitigations=off` — on a shared kernel, those flags hand sandboxed code hardware privilege-escalation surface that no amount of patching can fix.
- **Distro support** — warns when the host distribution is past (or within 6 months of) its standard-support end.
- **Incus version** — adds an advisory note below the recommended release (stays OK, never flips the exit code; hard minimum stays 6.1).
- **Image age** — warns when the coi base image is over 30 days old; rebuild with `coi build --force`.

Practical guidance:

- Keep the host kernel current — it is the isolation boundary. Prefer distributions/kernels that track upstream closely (Ubuntu HWE kernels, current releases) over aging LTS kernels.
- Update Incus from the [Zabbly repository](https://github.com/zabbly/incus) rather than distribution backports.
- Rebuild the base image regularly and prefer ephemeral containers (the default): every session starts pristine, so nothing an agent planted survives.
- Set `[limits.runtime] max_duration` for unattended runs — the Trail of Bits escapes took 12+ hours of uninterrupted autonomy. The `hardened` profile caps sessions at 4h by default.

## When a Container Is Not Enough

If your threat model includes a frontier-capability agent actively trying to escape — running untrusted agents on hardware that also holds valuable secrets, or agent workloads specifically aimed at security research — a shared-kernel container is the wrong tool regardless of hardening. Use:

- a dedicated VM with a minimal device surface (the article found Firecracker-class microVMs held where QEMU fell), or
- a dedicated/disposable host with nothing else on it.

Coi's sweet spot is containing misbehavior — prompt-injected agents, supply-chain-compromised tooling, overeager automation — with excellent ergonomics. For containing adversaries, put a hardware or hypervisor boundary underneath it.
