# nftables Setup

Network isolation (restricted/allowlist modes) requires nftables. If you see the error "nft is not available or passwordless sudo is not configured", you have two options.

> This page covers the egress-isolation firewall (the FORWARD-chain rules that enforce network modes). The separate ruleset used by the security monitor is documented in [nftables Monitoring Internals](nftables-Monitoring-Internals).

## Option 1: Use Open Network Mode (Quick Fix)

```toml
# ~/.coi/config.toml
[network]
mode = "open"
```

This disables egress filtering but allows you to work immediately. If you also want Coi to never invoke `sudo` for network operations, add `use_sudo = false` (see [Running without sudo](Network-Isolation#running-without-sudo-use_sudo--false)).

## Option 2: Install and Configure nftables (Recommended)

nftables provides the FORWARD chain filtering needed for network isolation. Re-running `install.sh` sets this up automatically, or do it manually:

```bash
# 1. Install nftables (Ubuntu/Debian)
sudo apt install nftables

# 2. Allow Coi to manage firewall rules (passwordless sudo for nft)
coi health --fix
```

`coi health --fix` writes `/etc/sudoers.d/coi-nft` safely: the rule names you by numeric UID and is checked with `visudo` before it is installed (see [System Health Check](System-Health-Check#fixing-problems---fix)). To do the same by hand:

```bash
rule="#$(id -u) ALL=(ALL) NOPASSWD: $(command -v nft || echo /usr/sbin/nft)"
tmp=$(mktemp) && echo "$rule" > "$tmp" \
  && sudo visudo -cf "$tmp" && sudo install -m 0440 "$tmp" /etc/sudoers.d/coi-nft
rm -f "$tmp"
```

Always validate with `visudo` first: a syntax error in `/etc/sudoers.d` disables `sudo` for the whole system, including the command you would need to fix it.

## Key Points

- nftables plus passwordless sudo for `nft` must be available for network isolation to work
- Coi adds nftables rules to its own `ip coi` table (a forward-hook chain, plus an input chain for traffic to the host itself, which holds the allowlist DNS block) to filter container traffic
- Rules are scoped by container IP address for precise filtering
- Rules are removed when containers are stopped/deleted

## How It Works

- Coi gets the container's IP address from Incus
- nftables rules are applied per container IP and evaluated in order (first match wins)
- Restricted mode: Allow gateway (forward chain), block RFC1918, allow all else
- Allowlist mode: Block all DNS, allow gateway DHCP only, block RFC1918 and 169.254.0.0/16, allow the allowlisted IPs on their ports, block all else
- Traffic to the host itself (restricted and allowlist mode, unless `allow_host_access` is set): allow replies, DHCP, ping and (restricted mode only) DNS to the gateway, allow the ports of `[[network.hosts]]` and `allowed_domains` entries that name a host address, block all else; see [Container Access to Host Services](Network-Isolation#container-access-to-host-services)
- In open mode, if `nft` is unavailable and the host's FORWARD policy is DROP, Coi falls back to iptables bridge rules so container traffic can still be forwarded (restricted/allowlist always require nft)

## Bridge Forwarding Rules at Launch

Before `coi shell`, `coi run` or `coi build` starts a container, and in restricted or allowlist mode after `coi container launch` or `coi container start` starts one, Coi checks that the host's FORWARD chain lets the Incus bridge's traffic through, so the container can get an IP over DHCP. It does this once per command. What it does depends on what sudo allows:

- **sudo allows `iptables`:** Coi reads the chain with `iptables -S FORWARD` and adds its two `coi-bridge-forward` ACCEPT rules (traffic in from and out to the bridge) when they are missing. It adds them even when the chain accepts the traffic now, so a DROP policy set later (for example by Docker or ufw) does not cut containers off. It then prints a line that starts with `Added iptables FORWARD rules for <bridge>`.
- **sudo refuses `iptables`** (for example, only the `coi-nft` rule is in place), and iptables is the nf_tables variant or not installed: Coi reads the chain with `nft list chain ip filter FORWARD` and adds no rules. If the chain accepts the bridge's traffic, or there is no such chain, Coi changes nothing and prints nothing. If the chain drops the traffic, Coi prints one warning.
- **sudo refuses `iptables` and iptables is the legacy variant**, or Coi cannot read the chain: Coi prints one warning that it could not inspect the FORWARD chain, and adds no rules.
- **`[network] use_sudo = false`:** Coi runs no sudo for this check and changes nothing.

For the warnings and what to do about them, see [Troubleshooting](Troubleshooting#warning-could-not-ensure-bridge-forwarding-rules).

## Automatic Cleanup

Coi automatically manages firewall resources to prevent accumulation of stale configurations:

- **Cleanup on all termination paths**: Firewall rules are cleaned up during normal exit, `coi shutdown`, `coi kill`, and when the security responder auto-kills a container
- **Orphaned resource detection**: `coi clean --orphans` scans for and removes:
  - Orphaned veth interfaces (no master bridge)
  - Orphaned nft rules (for non-existent container IPs)
  - Orphaned IPv6 block rules (for containers no longer running)
  - Orphaned nftables monitoring rules and chains (when nftables monitoring is enabled)
  - Orphaned iptables bridge rules (coi-bridge-forward rules with no Coi containers running)

This prevents nftables from accumulating stale rules over time, which could otherwise cause configuration bloat and potential conflicts.

**Manual cleanup:**
```bash
# Clean up all orphaned resources (veths and stale nft/iptables rules)
coi clean --orphans

# Dry run to see what would be cleaned
coi clean --orphans --dry-run
```

## See Also

- [Network Isolation](Network-Isolation) - the network modes these rules enforce
- [nftables Monitoring Internals](nftables-Monitoring-Internals) - the separate monitoring ruleset and chains
- [System Health Check](System-Health-Check) - verifies `nft` and the sudoers rule are configured
- [Troubleshooting](Troubleshooting) - firewall rule accumulation, the bridge forwarding warning and related issues
