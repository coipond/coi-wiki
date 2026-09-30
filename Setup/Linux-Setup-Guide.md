# Linux Setup Guide

Coi's install script (`install.sh`) works across Linux distributions, but Incus itself requires some distro-specific setup. This guide covers what you need beyond the standard install.

## Universal Steps

These apply to all distributions:

```bash
# 1. Install Coi
curl -fsSL https://raw.githubusercontent.com/coipond/coi/master/install.sh | bash

# 2. Build the Coi image
coi build

# 3. Start coding
coi shell
```

The install script handles Incus initialization, idmap configuration, nftables installation (with the passwordless-sudo `nft` rule), and copy-on-write storage automatically (ZFS with btrfs fallback; ZFS is auto-installed only on apt systems, and OrbStack guests use btrfs). The sections below cover manual setup for cases where the script cannot auto-detect your environment.

## Arch Linux / CachyOS / Manjaro

Incus is available in the official Arch repositories.

### Install Incus

```bash
sudo pacman -S incus
```

### Enable the Service

```bash
sudo systemctl enable --now incus.service
```

### Add User to Groups

```bash
sudo usermod -aG incus-admin $USER
```

> **Important:** You must log out and back in for the group change to take effect. Coi runs `incus` directly and requires the `incus-admin` group to be active in your session. Alternatively, run `newgrp incus-admin` to activate the group in your current shell without logging out. You can verify with: `groups | grep incus-admin`.

Some Arch-based distributions may also require the `incus` group (in addition to `incus-admin`). If you get permission errors, try: `sudo usermod -aG incus,incus-admin $USER`.

### Initialize Incus

```bash
sudo incus admin init --auto
```

This creates the default bridge network (`incusbr0`), storage pool, and profile devices. If you need more control:

```bash
# Manual setup (equivalent to --auto)
incus network create incusbr0 ipv4.address=auto ipv4.nat=true ipv6.address=none
incus storage create default btrfs size=50GiB   # CoW driver; a `dir` pool re-unpacks the whole image on every launch (~5-6s/GB) and `coi health` warns about it
incus profile device add default root disk path=/ pool=default
incus profile device add default eth0 nic name=eth0 network=incusbr0
```

### Fix Subordinate UID/GID Mapping

Arch does not ship with subordinate UID/GID ranges for root by default. Without this, Incus cannot create unprivileged containers and you will see:

> "System doesn't have a functional idmap setup"

```bash
# Add subordinate ranges for root
echo "root:1000000:1000000000" | sudo tee -a /etc/subuid
echo "root:1000000:1000000000" | sudo tee -a /etc/subgid

# Restart Incus to pick up the changes
sudo systemctl restart incus.service
```

The Coi install script detects and offers to fix this automatically.

### Set Up Nftables

Nftables (`nft`) plus passwordless sudo for it is required for Coi's network isolation modes (`restricted`, `allowlist`). Without it, only `mode = "open"` works.

```bash
sudo pacman -S nftables

# Allow Coi to manage firewall rules (passwordless sudo for nft)
echo "$USER ALL=(ALL) NOPASSWD: $(command -v nft)" | sudo tee /etc/sudoers.d/coi-nft
sudo chmod 0440 /etc/sudoers.d/coi-nft
```

Alternatively, re-run `install.sh` — it installs nftables and creates the sudoers rule automatically.

### Verify Setup

```bash
coi health
```

## Fedora / RHEL / CentOS Stream

### Install Incus

Incus is available via the [Zabbly repository](https://github.com/zabbly/incus) on RHEL-based systems:

```bash
sudo dnf install incus
sudo systemctl enable --now incus.service
sudo usermod -aG incus-admin $USER
```

> **Important:** You must log out and back in (or run `newgrp incus-admin`) for the group change to take effect before proceeding. Verify with: `groups | grep incus-admin`.

```bash
sudo incus admin init --auto
```

### Nftables and Firewalld

Coi's network isolation needs `nft` with passwordless sudo (`nft` is already present on Fedora; the install script or the sudoers rule from the Arch section above sets up the rest).

Fedora also ships with firewalld enabled by default, which can block traffic on the Incus bridge. If containers cannot get IP addresses or reach the internet, add the bridge to the trusted zone:

```bash
sudo firewall-cmd --zone=trusted --add-interface=incusbr0 --permanent
sudo firewall-cmd --reload
```

On firewalld hosts, also stop NetworkManager from enrolling container veths into firewalld zones — leaked registrations grow the firewall ruleset quadratically with every container launched ([#695](https://github.com/coipond/coi/issues/695); the install script does this automatically as of v0.11.2, manual installs should add it):

```ini
# /etc/NetworkManager/conf.d/99-coi-unmanaged.conf
[keyfile]
unmanaged-devices+=interface-name:veth*
```

```bash
sudo systemctl reload NetworkManager
```


### Idmap

Fedora typically ships with correct `/etc/subuid` and `/etc/subgid` entries. Verify:

```bash
grep root /etc/subuid /etc/subgid
```

If empty, add the ranges as shown in the Arch section above.

## openSUSE

```bash
sudo zypper install incus
sudo systemctl enable --now incus.service
sudo usermod -aG incus-admin $USER
```

> **Important:** You must log out and back in (or run `newgrp incus-admin`) for the group change to take effect before proceeding. Verify with: `groups | grep incus-admin`.

```bash
sudo incus admin init --auto
```

Coi's network isolation needs `nft` with passwordless sudo (see the Arch section above, or re-run `install.sh`).

Firewalld is the default firewall on openSUSE and can block traffic on the Incus bridge. If containers cannot get IP addresses, add the bridge to the trusted zone:

```bash
sudo firewall-cmd --zone=trusted --add-interface=incusbr0 --permanent
sudo firewall-cmd --reload
```

On firewalld hosts, also stop NetworkManager from enrolling container veths into firewalld zones — leaked registrations grow the firewall ruleset quadratically with every container launched ([#695](https://github.com/coipond/coi/issues/695); the install script does this automatically as of v0.11.2, manual installs should add it):

```ini
# /etc/NetworkManager/conf.d/99-coi-unmanaged.conf
[keyfile]
unmanaged-devices+=interface-name:veth*
```

```bash
sudo systemctl reload NetworkManager
```


## Ubuntu / Debian

Ubuntu and Debian are the primary target for Coi. The install script handles everything automatically.

Ubuntu ships Incus 6.0.x in its main repository. Coi recommends Incus >= 6.1 from the [Zabbly repository](https://github.com/zabbly/incus) for full idmapped-mount support. Since v0.11.1 a start failing with `idmapping abilities are required but aren't supported on system` no longer aborts — Coi converts the container to `raw.idmap` and retries automatically on every start path — so an older Incus degrades gracefully rather than fatally.

```bash
sudo apt install -y incus
sudo systemctl enable --now incus.service
sudo usermod -aG incus-admin $USER
```

> **Important:** You must log out and back in (or run `newgrp incus-admin`) for the group change to take effect before proceeding. Verify with: `groups | grep incus-admin`.

```bash
sudo incus admin init --auto
```

The install script installs nftables and sets up the passwordless-sudo `nft` rule that Coi requires for network isolation. ufw (Ubuntu's default firewall) can stay enabled; `coi health` warns if ufw's FORWARD policy (DROP) conflicts with container traffic (the `ufw_conflict` check), in which case Coi adds iptables bridge rules automatically.

## Common Issues Across Distros

### "System doesn't have a functional idmap setup"

Add subordinate UID/GID ranges for root and restart Incus:

```bash
echo "root:1000000:1000000000" | sudo tee -a /etc/subuid
echo "root:1000000:1000000000" | sudo tee -a /etc/subgid
sudo systemctl restart incus.service
```

### Containers Cannot Get IP Addresses

If your distro runs firewalld (Fedora, openSUSE), ensure the Incus bridge is in the trusted zone:

```bash
sudo firewall-cmd --zone=trusted --add-interface=incusbr0 --permanent
sudo firewall-cmd --reload
```

On firewalld hosts, also stop NetworkManager from enrolling container veths into firewalld zones — leaked registrations grow the firewall ruleset quadratically with every container launched ([#695](https://github.com/coipond/coi/issues/695); the install script does this automatically as of v0.11.2, manual installs should add it):

```ini
# /etc/NetworkManager/conf.d/99-coi-unmanaged.conf
[keyfile]
unmanaged-devices+=interface-name:veth*
```

```bash
sudo systemctl reload NetworkManager
```


### "nft is not available or passwordless sudo is not configured"

Either install nftables and configure passwordless sudo for `nft` (see distro-specific sections above, or re-run `install.sh`):

```bash
echo "$USER ALL=(ALL) NOPASSWD: $(command -v nft)" | sudo tee /etc/sudoers.d/coi-nft
sudo chmod 0440 /etc/sudoers.d/coi-nft
```

Or use open network mode (add `use_sudo = false` if you want Coi to never invoke sudo):

```toml
# ~/.coi/config.toml
[network]
mode = "open"
```

### Verify Everything

```bash
coi health --verbose
```

This checks Incus setup, permissions, security posture, network configuration, and monitoring prerequisites.

## See Also

- [macOS Setup Guide](macOS-Setup-Guide) - macOS-specific installation via Colima/Lima
- [Configuration](Configuration) - Post-install configuration reference
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - Starting your first session
