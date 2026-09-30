# macOS Setup Guide

Coi can run on macOS by using Incus inside a [Colima](https://github.com/abiosoft/colima), [Lima](https://github.com/lima-vm/lima), or [OrbStack](https://orbstack.dev) Linux VM. All three provide Linux VMs on macOS that can run Incus.

**Automatic Environment Detection**: Coi automatically detects when running inside a Colima, Lima, or OrbStack VM and adjusts its UID-mapping configuration accordingly. No manual configuration needed — including when your Mac user's UID is not 1000. (Colima/Lima handling accurate as of v0.10.0; OrbStack as of v0.11.1.)

## How It Works

Coi has two independent mechanisms that make workspace ownership come out right on a Mac VM, both automatic:

1. **The filesystem check (v0.11.1)** - Before starting a container, Coi `statfs`'s the host source of every disk device it will mount — the workspace, any configured `[[mount]]`s, and a git worktree's external git dir. A source on a FUSE-family filesystem (what OrbStack's macOS share reports) or 9p (Lima's `mountType: 9p`) cannot be relied on for Incus's idmapped (`shift=true`) mounts, so Coi skips shift for that container and applies a `raw.idmap` UID mapping instead — decided before first boot, no error, no config. A path that does not exist yet is judged by its nearest existing ancestor (that is the filesystem it would be created on).
2. **VM detection (Colima/Lima)** - Coi also detects Colima/Lima guests (virtiofs mounts in `/proc/mounts`, the `lima` user). These VMs map UIDs at the VM level themselves, so Coi disables shifting and lets the VM's own mapping stand.
3. **Mismatched UIDs are remapped automatically** - When the VM-side user's UID differs from the container's `code` user (e.g. your Mac's primary user is UID 501), Coi applies a `raw.idmap` mapping at container creation, before first boot — so workspace files stay writable on both sides with no config.
4. **A reactive backstop underneath** - If a container start still fails with `idmapping abilities are required but aren't supported on system` (a kernel/filesystem that cannot do idmapped mounts and that the checks above did not catch), Coi converts the container to `raw.idmap` and retries automatically — on every start path, including reused persistent containers.

> **Note:** If you added the old `[incus] code_uid = <your uid>` workaround from before v0.10.0, you can remove it — the `raw.idmap` handling made it unnecessary.
>
> This environment shape is continuously verified in CI: every PR runs a lima-smoke lane (Coi inside a Lima VM, virtiofs mount, non-1000 guest UID).

## Network Mode on macOS

Network isolation modes (`restricted`, `allowlist`) work inside Colima/Lima/OrbStack VMs (accurate as of v0.10.0): they are enforced with nftables, and `nft` is a standard Incus dependency on common VM images — no extra setup. Verified on an OrbStack Ubuntu 24.04 VM with Incus from the Zabbly repo: `restricted` mode blocks private networks while still allowing internet out.

> Older versions of this guide said these modes require firewalld and recommended `mode = "open"` — that predates Coi's migration to nftables and is no longer true. Prefer keeping `restricted` (the safer default) unless it demonstrably fails on your image.

To confirm enforcement works on your VM image (kernel netfilter modules can vary by base image):

```bash
coi health   # the "Network restriction" check must pass
```

If — and only if — that check fails on your image, fall back to open mode:

```toml
# ~/.coi/config.toml
[network]
mode = "open"
```

## AWS Bedrock on macOS/Colima

If you are using Claude via AWS Bedrock, Coi automatically validates your setup when running in Colima/Lima and prevents startup with helpful error messages if anything is misconfigured.

### Common Issues and Fixes

**Issue 1: Dual .aws Directory Locations**

Colima creates two `.aws` locations that can get out of sync:

- `/home/lima/.aws/` - Lima's VM home (where `~` expands inside Colima)
- `/Users/<yourname>/.aws/` - macOS home (mounted via virtiofs)

**Solution:** Pick one location and be consistent:

1. **Recommended:** Use macOS path `/Users/<yourname>/.aws/`
2. Run `aws sso login` on your Mac (not inside Colima)
3. Ensure it is mounted to containers (see below)

**Issue 2: Restrictive Permissions on SSO Cache**

AWS SSO creates cache files with permissions `-rw-------` (600), which become unreadable inside containers.

**Solution:** After running `aws sso login`, fix permissions:

```bash
chmod 644 ~/.aws/sso/cache/*.json
```

**Issue 3: .aws Not Mounted**

The container needs access to your AWS credentials.

**Solution:** Add to your config:

```toml
# ~/.coi/config.toml
[[mounts.default]]
host = "~/.aws"
container = "/home/code/.aws"
```

### Complete Bedrock Setup Example

1. Configure Bedrock in `~/.claude/settings.json`:

```json
{
  "anthropic": {
    "apiProvider": "bedrock",
    "bedrock": {
      "region": "us-west-2",
      "profile": "default"
    }
  }
}
```

2. Set up AWS SSO (on macOS, not in Colima):

```bash
aws configure sso
aws sso login
chmod 644 ~/.aws/sso/cache/*.json
```

3. Configure mount in `~/.coi/config.toml`:

```toml
[[mounts.default]]
host = "/Users/<yourname>/.aws"
container = "/home/code/.aws"
```

4. Launch Coi:

```bash
colima ssh
coi shell
```

Coi validates your setup and provides clear error messages if anything is missing or misconfigured.

## Claude Auth on macOS (Keychain)

If `coi shell` drops you into a brand-new Claude Code session — the theme/onboarding prompt and a login flow — even though you are already signed in on your Mac, this is expected and fixable. Two things combine on macOS:

1. **coi runs inside the Linux VM.** `coi` is installed and runs inside the Colima/Lima/OrbStack guest, so from its point of view "home" is the VM's home (e.g. `/home/lima`), not your Mac home. Your Mac home is shared into the VM under `/Users`. As of Coi v0.12+, coi automatically reads your tool config from the shared Mac home (`/Users/<you>/.claude`) when the VM home has none — so settings and the `~/.claude.json` onboarding state come across on their own.

2. **The login token is in the macOS Keychain, not a file.** Claude Code on macOS stores its OAuth token in the login Keychain (item `Claude Code-credentials`), not in `~/.claude/.credentials.json` (the Linux location coi seeds from). coi runs in the Linux VM and cannot read the Mac Keychain, so there is no credential file to bring across — hence the login flow.

When coi detects this situation it prints a hint with the exact command. To fix it, materialize the token into a file on your Mac, then start coi again:

```bash
# Run these on the Mac (Terminal.app), NOT inside Colima/Lima.
# Confirm the Keychain item exists (name can vary by Claude Code version):
security find-generic-password -s "Claude Code-credentials"
# If the name differs: security dump-keychain | grep -i claude

# Write it to the file coi seeds from:
security find-generic-password -s "Claude Code-credentials" -w > ~/.claude/.credentials.json
head -c1 ~/.claude/.credentials.json   # sanity: should print "{"
```

Then run `coi shell` again. coi picks up `~/.claude/.credentials.json` (plus your settings and `~/.claude.json`) from the shared Mac home and you land in an authenticated session.

> The token refreshes periodically, so if you get logged out again, re-run the `security … > ~/.claude/.credentials.json` command.

### Alternative: Use an API Key

If you would rather not touch the Keychain, forward an API key instead — this sidesteps OAuth entirely:

```bash
# ~/.coi/config.toml
[env]
forward_env = ["ANTHROPIC_API_KEY"]
```

Then `export ANTHROPIC_API_KEY=…` on the Mac (or inside the VM) before starting coi. API-key usage bills separately from a Pro/Max subscription, so subscription users usually prefer the Keychain approach above.

### Older Layouts / Whole-`/Users` Shares

If the automatic shared-home pickup does not fire (e.g. a custom mount that shares all of `/Users`, or a non-standard home path), point coi at the file explicitly with a credential entry:

```toml
# ~/.coi/config.toml (inside the VM)
[[credentials]]
host = "/Users/<you>/.claude/.credentials.json"
container = "/home/code/.claude/.credentials.json"
mode = "0600"

[[credentials]]
host = "/Users/<you>/.claude.json"
container = "/home/code/.claude.json"
```

## Setup Instructions

> **Note: Ubuntu Template Assumed**
> The commands below use `apt` and assume the default Colima/Lima Ubuntu template. If you started Colima with a different distro (e.g., `colima start --image debian`), substitute your distro's package manager. The Ubuntu template is recommended for Coi — it is the most tested and matches the base image Coi builds on.

```bash
# Install Colima (example)
brew install colima

# Start Colima with the Ubuntu template (default) and sufficient resources
colima start --cpu 4 --memory 8 --disk 50

# SSH into the VM
colima ssh

# Inside the VM, install Incus from the Zabbly repository
# (Ubuntu's own repo ships Incus 6.0.x, below Coi's required >= 6.1;
#  follow the repo setup at https://github.com/zabbly/incus, then:)
sudo apt update && sudo apt install -y incus
sudo incus admin init --auto
sudo usermod -aG incus-admin $USER
newgrp incus-admin

# Install Coi
curl -fsSL https://raw.githubusercontent.com/coipond/coi/master/install.sh | bash

# Build image and start a session
coi build
coi shell
```

## OrbStack

[OrbStack](https://orbstack.dev) is a popular fast alternative to Colima/Lima and is fully supported as of v0.11.1. Create an Ubuntu machine, install Incus in it, and run Coi inside — your macOS files are available in the machine under `/mnt/mac`.

```bash
# On the Mac
brew install orbstack
orb create ubuntu:24.04 coi-vm
orb shell -m coi-vm     # or: ssh coi-vm@orb

# Inside the machine, install Incus from the Zabbly repository
# (follow https://github.com/zabbly/incus, then:)
sudo apt update && sudo apt install -y incus
sudo incus admin init --auto
sudo usermod -aG incus-admin $USER
newgrp incus-admin

# Install Coi, build the image, start a session
curl -fsSL https://raw.githubusercontent.com/coipond/coi/master/install.sh | bash
coi build
coi shell
```

### How Coi Handles OrbStack's macOS Share

Unlike Colima/Lima, OrbStack's VM does not map UIDs at the VM level, and its macOS share (a FUSE filesystem — `fuseblk`) cannot be relied on for Incus's idmapped (`shift=true`) mounts:

- On OrbStack versions before 2.2.2, asking for a shift mount on the share failed the container start outright.
- On OrbStack 2.2.2 and later, the share accepts idmapped mounts but maps ownership wrongly — the container starts clean and every write into `/workspace` fails with `Permission denied`, with nothing explaining why. (Root cause is upstream: [orbstack/orbstack#2530](https://github.com/orbstack/orbstack/issues/2530).)

As of v0.11.1 Coi handles both cases automatically: it detects the FUSE-backed source before start and uses a `raw.idmap` UID mapping instead of shift — for the workspace, any `[[mount]]`s from the share, and a git worktree whose main repository lives on the share. No `disable_shift` configuration is needed anymore (it remains available as a manual override). Persistent containers created by an older Coi are healed on their next reuse: the mapping is re-decided and creation-time shift devices are converted.

### Storage on OrbStack

The OrbStack VM kernel has no ZFS tooling, so Coi's installer sets up a btrfs storage pool instead (still copy-on-write, still fast). If your pool was created before that (e.g. by a plain `incus admin init`), it may be on the slow `dir` driver — every session then re-unpacks the whole container image (~5-6s per unpacked GB). `coi health` warns about this; the fix is recreating the pool with btrfs (re-running `install.sh` sets one up).

### Keep Your Workspace Where You Like

Both locations work: a workspace on the guest's own disk (fastest) or on the macOS share under `/mnt/mac` (convenient for editing from the Mac side). Coi decides the right UID mapping per container based on where the workspace and mounts actually live.

## Manual Override

Rarely needed as of v0.11.1: the filesystem check covers the OrbStack/Colima/Lima shares this used to be hand-set for, and even when the check misses, a start failing on idmapped mounts auto-recovers via the `raw.idmap` fallback. Keep it for a source the checks clear but that still cannot do idmapped mounts — symptoms are the start error `idmapping abilities are required but aren't supported on system` without recovery, or a workspace that mounts but is unwritable:

```toml
# ~/.coi/config.toml
[incus]
disable_shift = true
```

## See Also

- [Linux Setup Guide](Linux-Setup-Guide) - Linux-specific installation
- [Configuration](Configuration) - Post-install configuration reference
- [Network Isolation](Network-Isolation) - Network mode limitations on macOS
