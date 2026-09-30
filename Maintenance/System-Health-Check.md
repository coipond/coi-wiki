# System Health Check

Use `coi health` to diagnose setup issues and verify your environment is correctly configured.

## Usage

```bash
# Basic health check
coi health

# JSON output for scripting/automation
coi health --format json
coi health --json                # shorthand for --format json

# Verbose output with additional checks
coi health --verbose
```

## Example Output

```text
Code on Incus Health Check
==========================

SYSTEM:
  [OK]   Operating system  : Ubuntu 24.04.4 LTS (amd64)
  [OK]   Kernel version    : Kernel 6.17.0-19-generic (>= 5.15)
  [OK]   Kernel build age  : Kernel built 42 days ago
  [OK]   Kernel mitigations: CPU mitigations enabled (no disable flags on the kernel command line)
  [OK]   Distro support    : ubuntu 24.04 supported until 2029-05-31

CRITICAL:
  [OK]   Incus             : Running (version 6.21)
  [OK]   Permissions       : User in incus-admin group
  [OK]   Default image     : coi-default (fingerprint: 3c946d37f573)
  [OK]   Image age         : 2 days old
  [OK]   Privileged check  : Default profile uses unprivileged containers
  [OK]   Security posture  : Full isolation - unprivileged containers with seccomp and AppArmor

NETWORKING:
  [OK]   Network bridge    : incusbr0 (10.128.178.1/24)
  [OK]   IP forwarding     : Enabled
  [OK]   nft firewall      : nft available, Incus bridge NAT enabled (restricted mode available)
  [OK]   Bridge forward    : Bridge incusbr0 forwarding OK
  [OK]   Docker FORWARD    : Docker running, FORWARD policy is not DROP

MONITORING:
  [OK]   nftables          : Available and configured
  [OK]   systemd journal   : Access granted
  [OK]   libsystemd        : Installed

STORAGE:
  [OK]   Coi directory     : ~/.coi (writable)
  [OK]   Sessions dir      : ~/.coi/sessions-claude (writable)
  [OK]   Disk space        : 455.0 GB available
  [OK]   Incus storage pools: default (zfs): 26.1 GiB free of 50.0 GiB (48% used)

CONFIGURATION:
  [OK]   Config loaded     : ~/.coi/config.toml
  [OK]   Network mode      : restricted
  [OK]   Tool              : claude

STATUS:
  [OK]   Containers        : 1 running
  [OK]   Saved sessions    : 12 session(s)
  [OK]   Orphaned resources: No orphaned resources

OTHER:
  [OK]   Monitoring config : Enabled with auto_pause=true
  [OK]   Audit Log Dir     : ~/.coi/audit (writable)
  [OK]   Cgroup Availability: Cgroup v2 is available with controllers
  [OK]   Container Connectivity: DNS and HTTP working (status 200)
  [OK]   Network Restriction: Restricted mode working (external OK, private networks blocked)

STATUS: HEALTHY
All 27 checks passed
```

**Note:** The MONITORING section (nftables, systemd journal, libsystemd) only appears when `monitoring.nft.enabled = true` in your config. The `--verbose` flag adds optional checks like DNS resolution, passwordless sudo, and process monitoring capability.

## Exit Codes

- `0` = healthy (all checks pass)
- `1` = degraded (warnings but functional)
- `2` = unhealthy (critical failures)

## What Is Checked

| Category | Checks |
|----------|--------|
| **System** | OS info and Colima/Lima detection, kernel version (warns if < 5.15), kernel build age (warns when built >6 months ago — distribution lag means unpatched kernel bugs), kernel mitigations (warns when CPU side-channel mitigations are disabled on the kernel command line, e.g. `mitigations=off`), distro support (warns past or within 6 months of the distribution's standard-support EOL) |
| **Critical** | Incus availability and version (>= 6.1 required, 6.7+ recommended as an advisory note), group permissions, default image, image age, privileged profile detection (`security.privileged=true`), security posture (seccomp + AppArmor verification) |
| **Networking** | Network bridge, IP forwarding, nft (mode-aware, with masquerade check), bridge forward rules (warns if FORWARD policy is DROP without bridge ACCEPT rules), iptables sudo, ufw conflict (warns if ufw's FORWARD DROP could block container traffic), Docker FORWARD policy (warns if Docker sets FORWARD to DROP), firewalld veth bloat (warns when dead veth registrations accumulate in firewalld zones — the ruleset grows quadratically; see [Troubleshooting](Troubleshooting#firewall-ruleset-grows-huge-firewalld--networkmanager)) |
| **Storage** | Coi directory, sessions directory, disk space (warns if <5GB), Incus storage pools (per-pool stats and **driver**; warns if <5GB free or >80% used, fails if <2GB free or >90% used; **warns on `dir`-driver pools** — every launch re-unpacks the whole image, ~5-6s per unpacked GB; recreate the pool with zfs/btrfs, e.g. by re-running `install.sh`) |
| **Configuration** | Config files, network mode, tool |
| **Status** | Running containers, saved sessions, orphaned resources (veth interfaces, nft rules/chains, iptables bridge rules) |
| **Monitoring** | nftables availability (when NFT monitoring enabled), systemd-journal access, libsystemd, monitoring config, audit log directory, cgroup availability |
| **Container Networking** | Container connectivity (DNS + HTTP to external host), network restriction verification (confirms restricted mode blocks private networks) |
| **Optional** | DNS resolution, passwordless sudo, process monitoring capability (with `--verbose`) |

### Security Posture Details

The Security posture check inspects the default Incus profile to verify container isolation:

| Status | Meaning |
|--------|---------|
| **OK** (full isolation) | Unprivileged + seccomp enabled + AppArmor available |
| **OK** (seccomp-only) | Unprivileged + seccomp enabled + AppArmor not available (macOS/Lima) |
| **Warning** | `raw.seccomp` or `raw.apparmor` overridden on default profile - custom settings should be verified |
| **Failed** | `security.privileged=true` - all isolation disabled (seccomp and AppArmor inactive) |

JSON output (`--format json`) includes detailed fields: `seccomp`, `apparmor`, `privileged`, `raw_seccomp_override`, `raw_apparmor_override`.

### Version Checks

Coi enforces minimum versions for critical dependencies:

- **Incus >= 6.1** - Required for UID mapping support. `coi health` warns; `coi shell`/`coi run`/`coi build` fail with an actionable error and a link to install from the Zabbly repository.
- **nftables >= 0.9.0** - Required for network monitoring features. `coi health` warns if below minimum.
- **Kernel >= 5.15** - Recommended for security features. `coi health` and CLI commands warn on stderr but do not block.

## JSON Output

```bash
coi health --format json
```

Returns structured JSON with all check results, suitable for scripting and CI integration:

```json
{
  "status": "healthy",
  "timestamp": "2026-03-29T12:00:00Z",
  "checks": {
    "security_posture": {
      "name": "security_posture",
      "status": "ok",
      "message": "Full isolation - unprivileged containers with seccomp and AppArmor",
      "details": {
        "seccomp": "enabled (default)",
        "apparmor": "enabled (default)",
        "privileged": false,
        "raw_seccomp_override": false,
        "raw_apparmor_override": false
      }
    }
  },
  "summary": {
    "total": 27,
    "passed": 27,
    "warnings": 0,
    "failed": 0
  }
}
```

## Notes

**Colima/Lima detection:** When running inside a Colima or Lima VM, the health check automatically detects this and shows `[colima]` in the OS info. If `nft` is not available or passwordless sudo is not configured, the nft check provides Colima-specific guidance (set `mode = "open"`). AppArmor is not available in Lima VMs, so the security posture check reports seccomp-only isolation.

**Privileged profile detection:** If `security.privileged=true` is set on the default Incus profile, both the `privileged_profile` and `security_posture` checks fail. Fix with: `incus profile unset default security.privileged`


## Updating Coi

Updating `coi` (and its GTFOBins + Sigma detection databases) has moved to its own page: **[Updating Coi](Updating-Coi)**.

```bash
coi update          # update the binary AND the detection databases
coi update core --check  # check for a newer release without installing
```

After updating, re-run `coi health` (this page) to confirm the environment is still correctly configured, and `coi build --force` if the release notes mention image changes.

## See Also

- [Updating Coi](Updating-Coi) - Updating the binary and detection databases
- [Troubleshooting](Troubleshooting) - Detailed diagnosis steps for specific issues
- [Image Management](Image-Management) - Rebuilding images to fix environment issues
- [Security Monitoring](Security-Monitoring) - Verifying monitoring is active and healthy
