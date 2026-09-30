# Updating Coi

`coi update` keeps two things current: the `coi` binary and the threat-detection databases (GTFOBins + Sigma) that the monitoring daemon reads. Running it with no subcommand does both, in sequence.

```bash
coi update            # update the binary AND the detection databases
coi update core       # update the binary only
coi update patterns   # update the GTFOBins + Sigma databases only
```

| Command | Updates | Common flags |
|---------|---------|--------------|
| `coi update` | binary + detection databases | `--force` (skip binary confirmation) |
| `coi update core` | `coi` binary only | `--check`, `--force` |
| `coi update patterns` | GTFOBins + Sigma rules | `--dry-run` |

```bash
coi update core --check   # check for a newer release without installing
coi update core --force   # skip the confirmation prompt
coi update --force        # same, when running the combined update
coi update patterns --dry-run  # print the git commands without running them
```

## Updating the Binary (`coi update core`)

### How It Works

1. Queries the GitHub releases API for the latest release.
2. Compares the current version against the latest semantic version.
3. Downloads the platform-appropriate binary (e.g. `coi-linux-amd64`).
4. Verifies the SHA256 checksum against the published `checksums.txt`.
5. Atomically replaces the current binary (temp file + rename).

- **Checksum verification**: the downloaded binary is verified against its SHA256 checksum before the current binary is replaced.
- **Symlink-aware**: if `coi` is a symlink, the symlink target is resolved before replacement so existing symlinks keep working.
- **Sudo auto-escalation**: when the binary directory is not writable by the current user (e.g. `/usr/local/bin`), Coi re-executes itself with `sudo`.
- **Dev build safety**: development builds (no version tag) cannot be version-compared. `coi update core` refuses to proceed and prints a warning; pass `--force` to install the latest release anyway.

## Updating the Detection Databases (`coi update patterns`)

The combined `coi update` also refreshes the two databases the `PROC_EVENTS` monitoring daemon reads at startup:

- **GTFOBins**: the reverse-shell / privilege-escalation binary database.
- **Sigma**: the `rules/linux/process_creation/` rule subtree (pulled as a sparse, blobless checkout, ~300 KB).

On first run each repository is cloned from its configured source; on later runs it is updated with `git pull --ff-only`. Source URLs and local directories live in `~/.coi/config.toml`:

```toml
[detection]
gtfobins_source = "https://github.com/GTFOBins/GTFOBins.github.io.git"
gtfobins_dir    = "~/.coi/gtfobins"
sigma_source    = "https://github.com/SigmaHQ/sigma.git"
sigma_dir       = "~/.coi/sigma"
```

> **Restart the monitoring daemon after updating patterns** so it picks up the new rules. Use `coi update patterns --dry-run` to preview the exact `git` commands without running them.

## After Updating

Run a health check to confirm the environment is still correctly configured:

```bash
coi health
```

If the release notes mention image changes, rebuild the container image:

```bash
coi build --force
```

## Troubleshooting

### `git pull` Fails When Updating Patterns (GTFOBins or Sigma)

`coi update patterns` uses `git pull --ff-only` against the existing clone, so it fails if the local database repo has diverged, has a broken/partial checkout, or was interrupted mid-clone. The fix is to remove the affected clone and let Coi re-clone it fresh:

```bash
# For Sigma (adjust the path if you customized sigma_dir):
rm -rf ~/.coi/sigma
coi update patterns

# For GTFOBins:
rm -rf ~/.coi/gtfobins
coi update patterns
```

If the directory exists but is not a git repository, Coi reports exactly that and asks you to remove it and re-run — the same fix applies. Restart the monitoring daemon once the clone succeeds.

### `coi update core` Says "already on the latest version" When It Is Not

> **Known issue fixed in v0.11.1:** release binaries of v0.11.0 printed a doubled version prefix (`coi vv0.11.0`) and `coi update` wrongly reported "already on the latest version" even when a newer release existed. If you are on an affected binary, update once via `install.sh` (or download the release binary directly); from v0.11.1 on, version comparison is normalized and `coi --version` matches `coi version`.

## See Also

- [System Health Check](System-Health-Check) — verify the environment after updating
- [Image Management](Image-Management) — rebuilding the container image after a Coi update
- [Security Monitoring](Security-Monitoring) — the daemon that consumes the GTFOBins + Sigma databases
- [Migration Guide](Migration-Guide) — version-to-version upgrade notes
