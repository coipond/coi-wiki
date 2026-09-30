# Image Management

Advanced image operations for creating and managing custom Coi images.

## List Images

```bash
# List Coi images
coi image list

# List all local images
coi image list --all

# Filter by prefix
coi image list --prefix claudeyard-

# JSON output
coi image list --format json
coi image list --json          # shorthand for --format json
```

## Publish Containers as Images

Convert a container into a reusable image:

```bash
# Publish container as image
coi image publish my-container my-custom-image

# With description
coi image publish my-container my-custom-image --description "Custom build with Python 3.11"

# Skip compression for faster publishing
coi image publish my-container my-custom-image --compression none

# Create image from container with installed tools
coi image publish coi-workspace-1 coi-rust --description "Coi with Rust toolchain"
```

## Delete Images

```bash
# Delete specific image
coi image delete my-custom-image

# Delete old image
coi image delete my-old-image
```

## Check Image Existence

```bash
# Check if image exists
coi image exists coi-default

# Use in scripts
if coi image exists my-custom-image; then
  echo "Image exists"
fi
```

## Clean Up Old Versions

```bash
# Keep only 3 most recent versions, delete older ones
coi image cleanup claudeyard-node-42- --keep 3

# Clean up all old images (keep latest only)
coi image cleanup my-project- --keep 1
```

## Creating Custom Images

### Workflow 1: Install Tools in Container

```bash
# Start a persistent session ([container] persistent = true in your
# project config or profile — as of v0.10.0 there is no --persistent flag)
coi shell

# Inside container: install your tools
sudo apt update
sudo apt install -y rust-all python3.11 golang
cargo install ripgrep
exit

# Stop the container first — exit leaves a persistent container running,
# and publish requires a stopped container
coi container stop coi-workspace-1

# Publish the stopped container as an image
coi image publish coi-workspace-1 coi-dev-full
```

Then select the custom image in config (or a profile):

```toml
# ./.coi/config.toml
[container]
image = "coi-dev-full"
```

### Workflow 2: Profile with Build Script

Create a profile directory, then declare the image and a `[container.build]` section in its config:

```bash
coi profile create my-custom
coi profile edit my-custom
```

Edit `~/.coi/profiles/my-custom/config.toml`:

```toml
[container]
image = "my-custom-image"

[container.build]
base = "coi-default"
script = "build.sh"    # resolved relative to this config.toml
```

Create `~/.coi/profiles/my-custom/build.sh`:

```bash
#!/bin/bash
apt update
apt install -y your-tools
pip install your-packages
```

Build and use:

```bash
coi build --profile my-custom
coi shell --profile my-custom
```

## Use Cases

### Team Images

Create standardized images for your team:

```bash
# Create team image with specific tools (persistent session via config)
coi shell
# Install team tools...
exit
coi image publish coi-workspace-1 team-nodejs-2024
```

Team members select it via the shared project config:

```toml
# ./.coi/config.toml (committed to the repo)
[container]
image = "team-nodejs-2024"
```

### Project-Specific Images

```bash
# Create image per project with dependencies pre-installed
coi image publish project-container myapp-v1.0
```

### Version Management

```bash
# Keep multiple versions
coi image publish coi-workspace-1 myproject-v1.0
coi image publish coi-workspace-1 myproject-v1.1

# Clean up old versions, keep last 3
coi image cleanup myproject- --keep 3
```

## Image Compression

By default, Incus compresses images with gzip when publishing. For `coi build`, the algorithm is a config key (as of v0.10.0 — the `coi build --compression` flag was removed); `coi image publish` keeps its `--compression` flag as raw plumbing:

```toml
# Global, per-project, or per-profile — like all of [container]
[container.build]
compression = "none"   # fastest, larger images — good for iteration
# compression = "xz"   # highest ratio, slowest — good for distribution
```

```bash
coi build                                           # uses [container.build] compression
coi image publish my-container my-image --compression none   # publish keeps the flag
```

**Available values:** `none`, `gzip` (default), `xz`, or any algorithm supported by Incus. The value is passed directly to `incus publish --compression`.

## Build Configuration in Project Config

You can define how to build custom images directly in `.coi/config.toml` using the `[container.build]` section:

```toml
# .coi/config.toml
[container]
image = "coi-myproject"

[container.build]
base = "coi-default"            # Base image (default: "coi-default")
script = "build.sh"             # Path to build script (relative to config file)
# commands = ["apt-get update", "apt-get install -y rustup"]  # Or inline commands
```

When `container.image` is set to a custom name and `[container.build]` is configured:

- `coi build` builds the custom image automatically

**Interactive build prompt:** When `coi shell` or `coi run` is invoked from a terminal and the required image does not exist, Coi prompts:

```text
Image 'coi-default' not found. Build it now? (~5 min) [y/N]:
```

Answering `y` triggers the build inline and continues into the session. Answering `n` (or pressing Enter) exits with an error telling you to run `coi build` manually. In non-interactive use (CI, piped scripts, `coi run` without a TTY) the prompt is skipped and the error is returned immediately.

**Script vs commands:** If both `script` and `commands` are set, `script` takes precedence. Script paths are resolved relative to the config file location.

See also [Profiles - Build Scripts](Profiles#build-scripts) for per-profile build configuration.

## Pre-installed Runtime Manager (mise)

The base `coi-default` image includes [mise](https://mise.jdx.dev) - a polyglot runtime manager for installing and managing language runtimes. The following are pre-installed:

| Tool | Notes |
|------|-------|
| **Python 3** | `python3`, `pip`, `venv` |
| **pnpm** | Fast Node.js package manager |
| **TypeScript** | `tsc` compiler |
| **tsx** | TypeScript execution without compilation |

Install additional runtimes inside your container or build script:

```bash
mise use --global go@latest      # Install Go
mise use --global ruby@3         # Install Ruby
mise use node@22                 # Install Node.js 22 (per-project)
```

Create a `.mise.toml` in your project root to pin runtime versions per-project. mise is activated in all shell sessions (interactive and non-interactive), so tools are available to AI agents automatically. Workspaces containing `mise.toml` or `.tool-versions` are automatically trusted inside the container (via `MISE_TRUSTED_CONFIG_PATHS`), so you do not need to run `mise trust` manually.

**Note:** System Node.js (v20 LTS) is retained for Claude CLI and core tooling alongside mise-managed runtimes.

## Notes

- Images are stored locally in Incus
- Publishing an image captures the complete container state
- Custom images are selected via `[container] image` in config or a profile
- Image cleanup helps manage disk space
- Use descriptive names and versions for easy identification

## Troubleshooting

### Image Not Found

**Symptom:** `coi shell` or `coi run` fails with "image not found" or prompts to build.

**Cause:** The required image (`coi-default` or a custom image) does not exist in the local Incus image store.

**Fix:**

```bash
# Build the default image
coi build

# Build a custom image for a project
coi build --profile my-profile

# Verify the image now exists
coi image exists coi-default
```

If running in a terminal, Coi prompts interactively and builds on confirmation. In non-interactive environments (CI, piped scripts), the error is returned immediately.

### Build Fails Mid-Way

**Symptom:** `coi build` starts but fails during package installation or script execution.

**Common causes and fixes:**

- **Network issue during build** - The build runs inside a container with internet access. Check your internet connection and DNS resolution. If using a restrictive firewall, try temporarily switching to open mode.
- **Full disk or storage pool** - Check available space with `coi health` or `incus storage info default`. Free up space and retry.
- **Custom build script error** - Run the script manually in a persistent container to debug:

```bash
# with [container] persistent = true in config:
coi shell
# run your build.sh steps manually and check for errors
```

### Custom Image Not Applied

**Symptom:** `coi shell` starts but the custom image's tools are not present.

**Cause:** The session used the wrong image. Either the image name in config does not match, or the custom image was not built.

**Fix:**

```bash
# Verify which image is configured
coi health

# Verify the custom image exists
coi image list

# Re-build if needed
coi build --profile your-profile
```

### Old Image After Coi Update

**Symptom:** After running `coi update`, sessions start with a stale environment (missing new tools, old mise version, etc.).

**Cause:** `coi update` updates the Coi binary but does not rebuild the container image. Image changes included in a release require a manual rebuild.

**Fix:**

```bash
coi update          # update the binary
coi build --force   # force rebuild the image
```

Release notes indicate when `coi build --force` is needed after an update.

## Best Practices

1. **Rebuild after Coi updates that include image changes** - Run `coi build --force` after updates that mention image changes in the release notes. The binary and the image are versioned independently.

2. **Use descriptive image names with versions** - Prefer `myproject-v1.2` over `myproject-latest` so you can roll back if a new image breaks something

3. **Clean up old images regularly** - Use `coi image cleanup myproject- --keep 3` to retain only the last 3 versions and prevent storage pool bloat

4. **Publish from stopped containers** - `coi image publish` captures filesystem state, not process memory. A running container's in-memory state is only present on disk if the process has flushed it. For reproducible images, stop the container first (`coi container stop <name>`) so the filesystem is in a clean, consistent state, then publish

5. **Use build scripts for team images** - Checked-in build scripts (`build.sh` in a profile directory) are reproducible and auditable. Avoid publishing ad-hoc containers as shared team images

## See Also

- [Configuration](Configuration) - Image selection and build configuration
- [Profiles](Profiles) - Per-profile image settings
- [System Health Check](System-Health-Check) - Verifying your environment after image changes
- [Updating Coi](Updating-Coi) - Keeping Coi itself up to date
