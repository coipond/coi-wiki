# Static Host Entries (`[[network.hosts]]`)

Sometimes a container needs a fixed name→address mapping that DNS will not give it: a database at `10.0.0.5`, an internal service reachable only by IP, a name that has to resolve the same way on every launch. A trusted `[[network.hosts]]` block writes those entries into the container's `/etc/hosts` at session setup — in its own coi-managed block, alongside (not clobbering) the allowlist block — and makes each address reachable in a way that matches the active network mode.

```toml
# ~/.coi/config.toml
[[network.hosts]]
ip        = "10.0.0.5"
hostnames = ["db.internal", "db"]

[[network.hosts]]
ip        = "203.0.113.10"
hostnames = ["api.internal"]
```

Each entry needs a valid IPv4 `ip` and at least one `hostname`.

## Per-Host Ports (`ports`)

An entry may carry its own `ports` to scope the firewall reachability of that one host without touching the rest of egress:

```toml
[[network.hosts]]
ip        = "192.168.1.50"
hostnames = ["redmine.susanoo.pl"]
ports     = [443]                  # reachable ONLY on 443
```

This is the piece the global `allowed_ports` cannot provide (it caps every destination, internet included), so restricted mode can open one LAN service on one port while the internet stays fully open. Empty `ports` inherits the global `allowed_ports` (else all ports). It takes effect where the mode actually filters that class of address - a private host in restricted mode, a public host in allowlist mode - and a `ports` scope that could never be enforced (a public host in restricted mode, a private host in allowlist mode, or anything in open mode) is refused at setup with an explanation rather than silently ignored. At runtime: `coi hosts add <container> <ip> <hostname> --ports 443`.

## Mode-Aware Reachability

Writing a name into `/etc/hosts` only makes it resolve — the firewall still decides whether the address is reachable. Coi reconciles the two per mode, and refuses any entry that could never be reached (so you never get a dead name), aborting before it changes `/etc/hosts` or the firewall:

| Network mode | Public IP | Private IP (RFC1918) | Metadata / link-local (169.254.0.0/16) |
|--------------|-----------|----------------------|----------------------------------------|
| `open`       | Resolves (no firewall change needed) | Resolves | Resolves |
| `restricted` | Resolves (public egress already allowed) | Resolves **+** a targeted per-address allow is punched through the RFC1918 block | **Refused** (cloud-metadata SSRF) |
| `allowlist`  | Resolves **+** the address is added to the firewall allow set | **Refused** unless `allow_local_network_access = true` (allowlist hard-blocks RFC1918, so the entry would be a dead name) | **Refused** (cloud-metadata SSRF) |

Metadata / link-local addresses (`169.254.0.0/16`, the cloud metadata endpoint) are refused in both enforcing modes regardless of `allow_local_network_access` — pointing a name there is a credential-theft SSRF vector.

## Trusted Scope Only

Because a name→IP mapping is a spoofing primitive (it could redirect `api.anthropic.com` to an attacker's box) and reachability punches a firewall hole, `[[network.hosts]]` is honored only from trusted-scope config — your `~/.coi/config.toml` or a config named by `COI_CONFIG`. A project `./.coi/config.toml`'s entries are stripped at load time with a warning:

```text
WARNING: ignoring security-downgrading "network.hosts" in project config <path>; move it to ~/.coi/config.toml or set COI_CONFIG to apply it.
```

## Adding Entries at Runtime (`coi hosts`)

The same thing can be done on an already-running container with `coi hosts`, which applies the same mode-aware reachability:

```bash
# Add an entry (one IP, one or more hostnames). Accepts a container name or alias.
coi hosts add coi-abc12345-1 10.0.0.5 db.internal cache.internal

# List the coi-managed host entries in a running container
coi hosts list coi-abc12345-1

# Remove entries by hostname
coi hosts remove coi-abc12345-1 db.internal
```

Runtime entries are session-scoped: the `/etc/hosts` line and any firewall hole opened for it are torn down when the container stops. For entries that should always be present, declare them in `[[network.hosts]]` config instead. `coi hosts add` detects the container's actual enforced mode, so it will not punch a restricted-style per-address hole through an allowlist container; adding a second hostname for an IP already present unions the names.

## See Also

- [Network Isolation](Network-Isolation) - the egress modes and firewall model these entries plug into
- [Configuration](Configuration) - full `[network]` configuration reference
- [Container Operations](Container-Operations) - managing `/etc/hosts` at runtime with `coi hosts`
- [Security Best Practices](Security-Best-Practices) - why host entries are trusted-scope only
