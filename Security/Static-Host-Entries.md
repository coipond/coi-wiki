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

This is the piece the global `allowed_ports` cannot provide (it caps every destination, internet included): restricted mode can open one LAN service on one port while the internet stays fully open, and allowlist mode can reach one LAN service while everything else stays closed. Empty `ports` inherits the global `allowed_ports` (else all ports).

It takes effect where the mode actually filters that class of address - a private host in restricted or allowlist mode, a public host in allowlist mode. A `ports` scope that could never be enforced is refused at setup with an explanation rather than silently ignored: a public host in restricted mode (already reachable on every port), a private host in allowlist mode with `allow_local_network_access = true` (the whole LAN is already open on the global `allowed_ports`), or anything in open mode. At runtime: `coi hosts add <container> <ip> <hostname> --ports 443`.

### A LAN Service in Allowlist Mode

Allowlist mode blocks the LAN, so a private address is only opened on explicit ports: set `ports` on the entry (or a global `allowed_ports`). Without either, the entry is refused - an allowlist container never gets an all-ports hole to a LAN machine.

```toml
[network]
mode            = "allowlist"
allowed_domains = ["github.com", "api.anthropic.com"]

[[network.hosts]]
ip        = "10.50.0.100"
hostnames = ["redmine.susanoo.pl"]
ports     = [443]                  # required for a LAN host in allowlist mode
```

The container reaches `redmine.susanoo.pl:443` and nothing else on that machine or the rest of the LAN, and you manage no firewall rules yourself. The name resolves from `/etc/hosts`, as every allowlisted name does: allowlist mode blocks all DNS from the container, so the LAN host's DNS ports are refused (`53`, and `853` for DNS-over-TLS) even if it is also your resolver. The host itself still resolves names normally.

## Mode-Aware Reachability

Writing a name into `/etc/hosts` only makes it resolve — the firewall still decides whether the address is reachable. Coi reconciles the two per mode, and refuses any entry that could never be reached (so you never get a dead name), aborting before it changes `/etc/hosts` or the firewall:

| Network mode | Public IP | Private IP (RFC1918) | Metadata / link-local (169.254.0.0/16) |
|--------------|-----------|----------------------|----------------------------------------|
| `open`       | Resolves (no firewall change needed) | Resolves | Resolves |
| `restricted` | Resolves (public egress already allowed) | Resolves **+** a targeted per-address allow is punched through the RFC1918 block | **Refused** (cloud-metadata SSRF) |
| `allowlist`  | Resolves **+** the address is added to the firewall allow set (scoped to its ports) | Resolves **+** a targeted allow on the entry's `ports` (required) is punched through the RFC1918 block; with `allow_local_network_access = true` the LAN is already open, so no extra rule | **Refused** (cloud-metadata SSRF) |

Metadata / link-local addresses (`169.254.0.0/16`, the cloud metadata endpoint) are refused in both enforcing modes regardless of `allow_local_network_access` — pointing a name there is a credential-theft SSRF vector.

## Security Monitoring

With [security monitoring](Security-Monitoring) on, a connection to a host entry on one of its permitted ports is not reported as a private-network threat and does not pause the container - in restricted and allowlist mode alike, including entries added at runtime with `coi hosts add`. The monitors check against the container's live firewall rules, so the same holds for a LAN resolver pinned with `dns_servers`. A connection to any other port on that host is still blocked and still flagged, and the other checks (such as known attack ports) still apply to the host.

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

Runtime entries are session-scoped: the `/etc/hosts` line and any firewall hole opened for it are torn down when the container stops. For entries that should always be present, declare them in `[[network.hosts]]` config instead. `coi hosts add` detects the container's actual enforced mode and whether it was started with `allow_local_network_access`, from its live firewall rules rather than from the config of whoever runs the command - so it will not punch a restricted-style per-address hole through an allowlist container, and in allowlist mode a LAN host needs `--ports` just as in config. Adding a second hostname for an IP already present unions the names.

## See Also

- [Network Isolation](Network-Isolation) - the egress modes and firewall model these entries plug into
- [Configuration](Configuration) - full `[network]` configuration reference
- [Container Operations](Container-Operations) - managing `/etc/hosts` at runtime with `coi hosts`
- [Security Best Practices](Security-Best-Practices) - why host entries are trusted-scope only
