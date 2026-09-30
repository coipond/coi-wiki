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
echo "$USER ALL=(ALL) NOPASSWD: /usr/sbin/nft" | sudo tee /etc/sudoers.d/coi-nft
sudo chmod 0440 /etc/sudoers.d/coi-nft
```

(If `nft` lives elsewhere on your distro, use the path from `command -v nft` in the sudoers rule — that is what `install.sh` does.)

## Key Points

- nftables plus passwordless sudo for `nft` must be available for network isolation to work
- Coi adds nftables rules to the FORWARD chain to filter container traffic
- Rules are scoped by container IP address for precise filtering
- Rules are removed when containers are stopped/deleted

## How It Works

- Coi gets the container's IP address from Incus
- nftables rules are applied per container IP, with allow rules ordered before reject rules
- Restricted mode: Allow gateway, block RFC1918, allow all else
- Allowlist mode: Allow gateway, allow specific IPs, block RFC1918, block all else
- If `nft` is unavailable and the host's FORWARD policy is DROP, Coi falls back to iptables bridge rules so container traffic can still be forwarded

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
- [Troubleshooting](Troubleshooting) - firewall rule accumulation and related issues
