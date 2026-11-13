# Network Isolation

Coi provides network isolation to protect your host and private networks from container access.

**Requirements:** Network isolation (restricted/allowlist modes) requires nftables (`nft`) plus the passwordless-sudo rule that `install.sh` creates (`/etc/sudoers.d/coi-nft`). Coi uses nftables rules to filter container traffic in the FORWARD chain, and traffic to the host itself in an input chain. If nftables or passwordless sudo is not available, set `[network] mode = "open"` in config (optionally with `use_sudo = false` to disable sudo entirely), or install nftables and configure the sudoers rule.

## Network Modes

### Restricted Mode (Default)

Blocks local networks, allows internet:

```bash
coi shell  # Default behavior
```

- Blocks: RFC1918 private networks (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- Blocks: Cloud metadata endpoints (169.254.0.0/16)
- Allows: All public internet (npm, pypi, GitHub, APIs, etc.)
- Blocks: Services on the host itself, except DHCP, ping and DNS to the gateway, unless `allow_host_access` is set (see [Container Access to Host Services](#container-access-to-host-services))
- IPv6 egress blocked host-side (plus disabled inside the container as root-reversible defense-in-depth), so it cannot bypass the IPv4 firewall rules
- A container's rules are installed (and replaced, e.g. when attaching to a running container) as one atomic firewall transaction, so it is never left briefly unfiltered
- A blocked IPv4 TCP connection fails at once with "connection refused" (a TCP reset); other blocked IPv4 traffic gets ICMP port-unreachable. Allowlist mode rejects the same way. Blocked IPv6 traffic is dropped, with no reply

### Allowlist Mode

Only specific domains allowed:

```toml
[network]
mode = "allowlist"
```

- Requires configuration with `allowed_domains` list
- **TTL-aware DNS refresh**: Domain IPs are re-resolved based on actual DNS TTL values, not a fixed interval. Domains with short TTLs (e.g., CDNs, cloud services) refresh more frequently, preventing allowed domains from becoming unreachable when their IPs rotate. The `refresh_interval_minutes` config acts as a maximum cap; the actual interval is the minimum TTL across all resolved domains (with a 60-second floor to prevent excessive queries)
- Always blocks RFC1918 private networks
- Blocks services on the host itself, except DHCP and ping to the gateway, unless `allow_host_access` is set (see [Container Access to Host Services](#container-access-to-host-services))
- IP caching for DNS failure resilience

### Open Mode

No restrictions (trusted projects only):

```toml
[network]
mode = "open"
```

## Running Without Sudo (`use_sudo = false`)

Restricted/allowlist enforcement requires Coi to run `nft` via passwordless
sudo (the installer adds `/etc/sudoers.d/coi-nft`). If you decline that sudoers
rule, set:

```toml
[network]
mode = "open"
use_sudo = false
```

With `use_sudo = false`, Coi never invokes `sudo` for network operations:

- **`open` mode** runs normally (no privileged rules are needed).
- **`restricted` / `allowlist`** modes fail fast with a clear error — they
  cannot be enforced without sudo, and Coi does not silently downgrade to
  open (fail-closed). Switch to `mode = "open"` to proceed.
- **`coi health`** reports the nft check as OK in open mode and as a
  warning (not a hard failure) if you pair `use_sudo = false` with a mode
  that needs nft — so a no-sudoers setup gets a clean health report.
- **nft network monitoring** is skipped (it requires `sudo nft`).

The setting can only make Coi do less — it never weakens isolation — so it is
safe to set from any config scope. It is the supported way to run Coi with no
sudoers modification at all.

## Configuration

```toml
# ~/.coi/config.toml
[network]
mode = "restricted"  # restricted | open | allowlist

# Allowlist mode configuration
# Supports both domain names and raw IPv4 addresses
allowed_domains = [
    "registry.npmjs.org",  # npm package registry
    "api.anthropic.com",   # Claude API
    "platform.claude.com", # Claude Platform
]
refresh_interval_minutes = 30  # Maximum IP refresh interval; actual interval uses DNS TTL if shorter (0 to disable)
```

### Important for Allowlist Mode

- **Gateway handling**: Coi automatically detects the Incus bridge gateway IP. In allowlist mode the firewall accepts it only for DHCP and a rate-limited ping, plus the ports of `[[network.hosts]]` and `allowed_domains` entries that name it, unless `allow_host_access` is set (see [Container Access to Host Services](#container-access-to-host-services)); DNS to it is blocked like all DNS. The security monitor does not flag the gateway's DNS and DHCP ports (see [Security Monitoring](Security-Monitoring#network-level-threats-nftables)). You do not need to add it manually.
- **No DNS servers needed**: allowlist mode blocks all DNS from the container. Coi resolves each allowed hostname on the host and writes the addresses into the container's `/etc/hosts`, so listing `8.8.8.8`/`1.1.1.1` buys nothing. If the container runs systemd-resolved, Coi stops it until the container's next boot, so `resolvectl` does not work: names in `/etc/hosts` still resolve, and a lookup of any other name fails at once. Coi also puts `files [!SUCCESS=return]` in front of the `hosts:` modules in the container's `/etc/nsswitch.conf` (or adds a `hosts:` line if there is none), so a name not in `/etc/hosts` fails without a DNS query. When a running container that an allowlist session left this way is set up in restricted or open mode, Coi starts systemd-resolved again and removes the prefix. On a container with systemd, `coi-nss-restore.service` also removes the prefix at the next boot. Changes that packages make to the `hosts:` line meanwhile (for example `libnss-mdns`, `libnss-myhostname`) are kept.
- **Firewall rule ordering**: port-scoped accepts for private `[[network.hosts]]` entries → DNS reject → gateway DHCP accept → RFC1918 + 169.254.0.0/16 reject (or LAN accept with `allow_local_network_access`) → port-scoped accepts for allowlisted addresses → rate-limited ICMP echo → default reject. Traffic to the host itself goes through the input chain instead: port-scoped accepts for `[[network.hosts]]` host-address entries → DNS reject → replies on existing connections → gateway DHCP accept → rate-limited ping to the gateway → port-scoped accepts for `allowed_domains` host-address entries → reject. In both chains, each reject has a TCP reset rule ahead of it.
- Supports both domain names (`github.com`) and raw IPv4 addresses (`8.8.8.8`)
- Subdomains must be listed explicitly (`github.com` ≠ `api.github.com`)
- Domains behind CDNs may have many IPs that change frequently
- DNS failures use cached IPs from previous successful resolution

## Egress Hardening: DNS Pinning and Port Scoping

Beyond the coarse mode choice, three composable controls narrow what a container can reach. `dns_servers` and `allowed_ports` are trusted-scope only (honored from `~/.coi/config.toml` / `$COI_CONFIG`, stripped from a project `./.coi/config.toml`); `allowed_domains` entries, including their `:ports` suffixes, are honored from project config too.

### DNS Resolver Pinning (`dns_servers`)

In restricted mode, pin the resolvers a container may reach on port 53, so a compromised container cannot bypass your resolver by talking straight to a public one (e.g. `8.8.8.8`):

```toml
[network]
mode        = "restricted"
dns_servers = ["192.168.1.2"]   # e.g. your Pi-hole
```

Coi accepts `:53` only to the listed IPv4 addresses and rejects every other off-box DNS query. The bridge's own resolver (the container's normal DHCP-provided DNS) travels a different path and is left untouched, so ordinary resolution keeps working with no `resolv.conf` changes. A pinned LAN resolver stays reachable on `:53` even when private networks are otherwise blocked - but on port 53 only.

- IPv4 addresses only.
- **Not valid in allowlist mode** (which blocks all DNS by design and resolves via `/etc/hosts`); Coi fails closed if you set both.
- **Caveat:** pinning `:53` only bites when port 443 is also constrained (allowlist mode, or `allowed_ports`), otherwise malware can still tunnel DNS-over-HTTPS on 443.

### Egress Port Allowlist (`allowed_ports`)

Restrict which destination ports the container may reach. In restricted mode this caps the otherwise-open internet egress; in allowlist mode it further constrains the allowlisted hosts. Everything else is rejected (ICMP echo still works, so `ping`/health checks are fine):

```toml
[network]
mode          = "restricted"
allowed_ports = [80, 443]   # web only; blocks SSH (22), DB ports, IoT panels, ...
```

This turns "installed a malicious package that now scans the LAN" into a far smaller problem: even reachable hosts are reachable only on the ports they legitimately need. The cap applies to the LAN too - even with `allow_local_network_access = true`, the local network is reachable only on these ports, so enabling local access does not silently reopen SSH/DB ports. Bridge-provided DNS is unaffected; add `53` if the container resolves via an off-box resolver.

### Per-Destination Ports (`allowed_domains` with `:ports`)

`allowed_ports` applies one port set to every allowlisted host. When destinations legitimately need different ports, scope each `allowed_domains` entry individually with a `:ports` suffix - a single port, a comma list, or a `lo-hi` range:

```toml
[network]
mode = "allowlist"
allowed_domains = [
    "github.com:443",            # git/HTTPS only
    "registry.npmjs.org:80,443", # a port list
    "192.168.1.50:5000",         # the NAS web UI, and nothing else on it
    "10.0.0.0/8:22",             # SSH into the lab subnet, but only SSH
    "svc.internal:8000-8100",    # a port range
    "api.anthropic.com",         # no port -> inherits allowed_ports (else all)
]
```

Each destination is then reachable only on its own ports. An entry with no `:ports` inherits the global `allowed_ports` (else all ports), so existing allowlists behave unchanged. IPv4 only; a malformed port fails the session closed at startup.

> **Note:** With [Security Monitoring](Security-Monitoring) enabled, connections to common C2/backdoor ports (1234, 4444, 5555, 6666, 6667, 6697, 8080, 12345, 31337) are flagged even when the firewall allows them, so avoid those ports for allowlisted services.

### The "Internet Open, LAN Only redmine:443" Recipe

Combined with per-host `ports` on `[[network.hosts]]` (see [Static Host Entries](Static-Host-Entries)), restricted mode can express "all internet open, and on the LAN reach only `redmine.susanoo.pl:443`, resolved by my Pi-hole":

```toml
[network]
mode        = "restricted"
dns_servers = ["192.168.1.2"]

[[network.hosts]]
ip        = "192.168.1.50"
hostnames = ["redmine.susanoo.pl"]
ports     = [443]
```

The allowlist-mode version - "only GitHub and Anthropic on the internet, and on the LAN only `redmine.susanoo.pl:443`" - needs no `dns_servers` (allowlist mode resolves names on the host and blocks DNS from the container), and `ports` is required for the LAN host:

```toml
[network]
mode            = "allowlist"
allowed_domains = ["github.com", "api.github.com", "api.anthropic.com"]

[[network.hosts]]
ip        = "192.168.1.50"
hostnames = ["redmine.susanoo.pl"]
ports     = [443]
```

See [Static Host Entries - A LAN Service in Allowlist Mode](Static-Host-Entries#a-lan-service-in-allowlist-mode).

## Container Access to Host Services

In `restricted` and `allowlist` mode, Coi also filters traffic from the container to the host itself: the bridge gateway and the host's other IPv4 addresses. The host accepts from the container only:

- Replies on connections the host opened
- DHCP (UDP 67) to the gateway
- Ping to the gateway, rate-limited
- DNS (UDP/TCP 53) to the gateway, in `restricted` mode only (allowlist mode blocks all DNS)
- The ports of `[[network.hosts]]` and `allowed_domains` entries that name a host address

A database or dev server that listens on the host is therefore not reachable from the container. `open` mode does not filter this traffic. If Coi cannot detect the gateway (restricted mode then starts with a warning), DHCP and DNS are accepted to any host address.

To reach one host service, list the host address with its ports. In allowlist mode it can be an `allowed_domains` entry:

```toml
[network]
mode            = "allowlist"
allowed_domains = ["github.com", "192.168.1.10:5432"]   # 192.168.1.10 is an address of this host
```

In either mode it can be a `[[network.hosts]]` entry with `ports` (see [Static Host Entries](Static-Host-Entries)). An entry with no ports uses the global `allowed_ports`; with neither, the address stays closed and Coi prints a warning. In allowlist mode with `allow_local_network_access` off (the default), a private `[[network.hosts]]` address with neither is refused at setup with an explanation instead. In allowlist mode these entries never open ports 53 and 853 on the host. An `allowed_domains` name that resolves to a host address also stays closed, and Coi prints a warning: list the address itself.

To let the container reach every service on the host, set `allow_host_access` in trusted-scope config (`~/.coi/config.toml` or `$COI_CONFIG`):

```toml
[network]
allow_host_access = true   # default false
```

With `allow_host_access` on, allowlist mode still blocks DNS (port 53) to the host. From a project `./.coi/config.toml` or a project profile it is ignored with a warning. When it is on in restricted or allowlist mode, the Network row of the [startup grants](Container-Lifecycle-and-Sessions#effective-grants-at-startup) shows `host services allowed (allow_host_access)`.

## Host Access to Container Services

### Accessing Services from the Host

By default, Coi allows the host machine to access services running in containers. In restricted and allowlist mode the input chain accepts the container's replies on connections the host opened; in open mode Coi adds no input rules. Connections the container opens to the host are filtered as described in [Container Access to Host Services](#container-access-to-host-services).

For example, if a web server runs on port 3000 in the container:

```bash
# Inside container: Puma/Rails server listening on 0.0.0.0:3000
# From host: Access via container IP
curl http://<container-ip>:3000
```

### Publishing Ports on the Host's localhost

Reaching services via `<container-ip>:<port>` works, but requires looking up the IP and does not tell the agent which ports you can reach. The `[ports]` config section (v0.10.1) publishes container ports at `localhost:<port>` on the host via Incus proxy devices — and because that uses the userspace forkproxy (`bind=host`), NOT NAT rules, the nftables isolation described on this page is completely untouched: the container gains no new outbound capability. See [Port Publishing](Port-Publishing).

### Allowing Access from Entire Local Network

For development environments where you want machines on your local network to access container services (e.g., accessing containers via tmux from multiple machines), add this to your config:

```toml
[network]
allow_local_network_access = true  # Allow all RFC1918, not just gateway
```

> **Warning:** When `allow_local_network_access = true`, ALL RFC1918 private network traffic is allowed - there is no RFC1918 blocking at all. A compromised AI tool can reach any machine on your local network, including internal services, NAS devices, and other development machines. Only enable this in fully trusted environments where cross-machine access is genuinely required.

**Default behavior:** Only the host (gateway IP) can access container services. Other machines on your local network cannot, even if they are on the same subnet.

### Troubleshooting Container Access

If you see "Connection refused" when trying to access container services:

1. Verify container service is listening: `coi container exec <name> -- netstat -tlnp`
2. Check container IP: `coi list` (shows IPv4 for running containers)
3. Ensure firewall allows traffic to the bridge network

## Static Host Entries (`[[network.hosts]]`)

Need a fixed name→address mapping (a DB at `10.0.0.5`, an internal service reachable only by IP) that resolves the same way on every launch — and is reachable in a way that matches the active network mode?

**→ See [Static Host Entries](Static-Host-Entries)** for the `[[network.hosts]]` config, per-host `ports`, the per-mode reachability table, trusted-scope rules, and the runtime `coi hosts` command.

## nftables Setup

Restricted/allowlist modes need nftables plus a passwordless-sudo rule for `nft`. If you hit "nft is not available or passwordless sudo is not configured":

**→ See [nftables Setup](nftables-Setup)** for the quick open-mode workaround, the install + sudoers steps, how the FORWARD-chain rules work, and orphaned-rule cleanup.

## Best Practices

1. **Use `restricted` mode by default** - It is the right balance for most work: full internet access for packages and APIs, no access to your internal network. Only switch away from it when you have a specific reason.

2. **Use `allowlist` mode for untrusted codebases** - When working in a repo you did not write or have not fully audited, constrain egress to only the services the project legitimately needs (npm, PyPI, GitHub, the project's own API). This limits what a malicious or prompted AI tool can contact.

3. **Do not add DNS servers to the allowlist** - allowlist mode resolves names on the host and blocks container DNS, so public resolvers like `8.8.8.8` and `1.1.1.1` are unnecessary.

4. **Do not use `allow_local_network_access = true` without understanding the risk** - This disables all RFC1918 blocking. It is appropriate when running services on other machines in your network that the AI needs to reach, but it removes the lateral movement protection.

5. **Combine with security monitoring** - Network isolation is a preventive control; [Security Monitoring](Security-Monitoring) is a detective control. Use both together. Monitoring logs network events even in open mode.

6. **Clean up orphaned rules periodically** - If containers exit abnormally, nftables rules can accumulate. Run `coi clean --orphans` or add a cron job to keep rules tidy.

## See Also

- [Security Best Practices](Security-Best-Practices) - Recommended network settings for different risk levels
- [Security Monitoring](Security-Monitoring) - How network threats are detected and responded to
- [Configuration](Configuration) - Full network configuration reference
- [Profiles](Profiles) - Per-profile network mode settings
- [Port Publishing](Port-Publishing) - Reaching container services at `localhost:<port>` without weakening isolation
- [Static Host Entries](Static-Host-Entries) - custom `/etc/hosts` name→IP mappings with mode-aware firewall reachability
- [nftables Setup](nftables-Setup) - installing nftables + the sudoers rule for restricted/allowlist modes
