# Port Publishing

Every Coi container has its own IP, so a service running inside was always reachable at `<container-ip>:<port>` — but nothing gave you `localhost` semantics, and nothing told the agent which ports you can actually reach. The `[ports]` config section (v0.10.1, #558) fixes both: it publishes container TCP ports on the host via Incus proxy devices, exports the mapping to the session environment, and instructs the agent (through the sandbox context file) to bind its servers to ports you can open directly.

Typical use case: you ask a coding agent to build a web app, it starts a dev server, and you open `http://localhost:23410` on your machine — no `incus list` for IPs, no manual forwarding.

## Quick Start

```toml
# ~/.coi/config.toml (or a profile, or a repo's .coi/config.toml)
[ports]
pool = 3
```

Every session now publishes 3 identity-mapped ports: the host port and the container port are the SAME number. The sandbox context file tells the agent: *"when you start a web server or any service the user should access, bind it to one of these ports and tell the user the localhost URL."* The agent binds e.g. `--port 23410`, you open `http://localhost:23410`. The numbers are exported inside the container as `COI_PORTS` (space-separated) and printed in the session startup log.

## The Two Forms

### `pool = N` — Free Ports for Anything the Agent Starts

```toml
[ports]
pool = 3   # 0-10 identity-mapped ports per session
```

Best for agent-driven work where you do not know in advance what will run. Identity mapping means there is only ONE number to communicate — the agent binds it, you open it.

### `[[ports.map]]` — Named, Fixed Container Ports

```toml
[[ports.map]]
name = "web"        # required, unique; drives COI_PORT_WEB and the device name
container = 3000    # required: the TCP port INSIDE the container (1-65535)

[[ports.map]]
name = "postgres"
container = 5432
host = 15432        # optional: pin the host port (otherwise auto-allocated)
# listen = "0.0.0.0" # optional: host listen address (default 127.0.0.1 — loopback only)
```

Best for services with well-known container ports (databases, a dev server config you control). Each entry is exported as `COI_PORT_<NAME>` (name upper-cased, non-alphanumerics → `_`), so the agent and your scripts can find the host port: `postgres` above is reachable at `localhost:$COI_PORT_POSTGRES`.

The `name` is load-bearing: it becomes the env var, the Incus device name (`coi-port-<name>`), the entry's line in the sandbox context file, and the identifier in trust warnings and errors. Names that would collapse to the same device or env var (`web_1` vs `web-1`) are rejected at preflight rather than silently overwriting each other.

## How Allocation Works

Auto-allocated ports (the pool, and map entries without `host =`) are deterministic with no coordination state:

```text
host port = 20000 + (identity-hash % 400) * 100 + (slot - 1) * 10 + index
# identity-hash = workspace-path hash, or the session_name hash when set
```

- Each workspace gets a stable 100-port neighborhood (derived from the same hash container names use).
- Each slot gets a 10-port block inside it — parallel sessions of one workspace never collide.
- Different workspaces sharing a profile get distinct ports by construction.

At most 10 auto-allocated ports per slot (pool + unpinned map entries); pin `host =` explicitly if you need more. Port publishing requires the slot to be within 1–10 (auto-allocated slots always are).

### The Preflight

Before any container is created, Coi bind-probes every planned port on the host:

- A taken pinned port is a hard error — the session aborts with `host port 15432 (ports.map "postgres") is already in use — free it or change the pin`, and no container is launched. A pin means pin.
- A busy auto/pool port is silently skipped forward within the slot's 10-port block. The real numbers are announced every session via the env vars, the startup log, and the context file, so exactness is a nicety, not a contract.

## Seeing Published Ports

`coi list` shows each container's published ports, with zero extra Incus calls:

```text
Active Containers:
------------------
  coi-a3f2-1 (persistent)
    Status: Running
    Ports: 23410, 23411, web:23412->3000, postgres:15432->5432
    ...
```

Pool ports render as the bare identity-mapped number; named entries as `name:host->container` (with the listen address prefixed when it is not loopback, e.g. `lan:0.0.0.0:8080->3000`). In `--format json` the same data is the `published_ports` array. For a Running container these are live host listeners; a stopped persistent container shows its last session's publications (re-resolved at the next start).

## Inside the Container

| What | Where |
|------|-------|
| Pool port numbers | `COI_PORTS` env var (space-separated), e.g. `23410 23411 23412` |
| Named entry host ports | `COI_PORT_<NAME>` env vars, e.g. `COI_PORT_WEB=23412` |
| Agent instructions | `~/SANDBOX_CONTEXT.md` "Published Ports" section |

The proxy device connects to container-localhost, so a dev server bound to `127.0.0.1` inside the container is still reachable — the agent can bind `0.0.0.0` or `127.0.0.1`, both work.

## Security Model

- **Loopback by default.** Published ports listen on the host's `127.0.0.1`. A map entry can opt into LAN exposure with `listen = "0.0.0.0"` (or a specific interface IP); the pool is always loopback-only.
- **Untrusted project configs are gated.** A cloned repo's `.coi/config.toml` declaring `[ports]` must be approved with `coi trust` first (same per-source fingerprint as out-of-workspace mounts, forwarded sockets, and ad-hoc credentials, and re-approval is required if the config later changes). Without approval the entries are dropped with a warning. The attack this blocks: a malicious repo squatting a well-known localhost port — e.g. a fake postgres on `127.0.0.1:5432` capturing credentials from your host's own local connections. If a repo's untrusted `pool` overrode a pool you configured yourself in trusted scope, your own value is restored, not dropped to zero.
- **Isolation-neutral.** Publishing uses Incus's userspace `forkproxy` (`bind=host`), NOT NAT rules — Coi's [nftables network isolation](Network-Isolation) is untouched, and the container gains no new outbound capability.

## Persistent Containers

Port devices are re-resolved and re-published on every session, including reuse of a persistent container: stale devices from the previous session are stripped before the preflight, so pinned ports keep working across sessions, pool numbers stay stable (no drift), and entries you removed from config — or revoked with `coi untrust` — stop being published. A stopped container's ports are not bound on the host (the forkproxy only runs while the container does).

## Profiles

`[ports]` works in profiles like every other section — and because allocation is per workspace and slot, several instances launched from the SAME profile (or several parallel slots of one workspace) get distinct, non-conflicting ports automatically:

```toml
# ~/.coi/profiles/webdev/config.toml
[ports]
pool = 3

[[ports.map]]
name = "web"
container = 3000
```

When the same map entry `name` appears in multiple scopes (global config, profile, project config), the later scope replaces the earlier entry rather than duplicating it.

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `port preflight failed: host port N ... is already in use` | Another process holds a **pinned** port. Free it, change the pin, or drop `host =` to let Coi auto-allocate. |
| `no free port left in this slot's block` | All 10 ports of this slot's block are busy on the host (rare; usually another app squatting the 2xxxx range). Reduce the pool or pin entries elsewhere. |
| Port listed in `COI_PORTS` but `localhost:<port>` refuses | Nothing inside bound that port yet — check the agent actually started the server, and on the number from `COI_PORTS` (see also `coi list`). |
| `[ports] declares N auto-allocated ports; at most 10 per slot` | pool + unpinned map entries exceed the block. Pin `host =` for some entries or shrink the pool. |
| `ports.map "x" collides with "y"` | Two names sanitize to the same device/env name (`web_1` vs `web-1`). Rename one. |
| Warning: `ignoring untrusted port ...` | The `[ports]` section came from a repo's `.coi/config.toml` you haven't approved. Run `coi trust` in the workspace. |

## See Also

- [Configuration](Configuration) — full config reference and the trust model for project configs
- [Network Isolation](Network-Isolation) — why publishing does not weaken egress rules
- [Profiles](Profiles) — bundling `[ports]` into reusable setups
