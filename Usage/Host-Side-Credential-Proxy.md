# Host-Side Credential Proxy

Coi gives the agent a credential by putting it in the container: `forward_env`, `[[credentials]]` and `[defaults.env_commands]` all place the value where every process the agent starts can read it (see [Limits and Non-Goals](Limits-and-Non-Goals#copied-tool-credentials-are-readable-by-the-agents-processes)). To keep an API key out of the container, run a proxy on the host that holds the key and adds it to each request, for example an LLM gateway, and point the tool at the proxy. This page shows how to connect a container to one with existing config.

The parts:

- **The proxy** runs on the host and holds the real key.
- **The path** from the container to the proxy: a host address and port that Coi opens (TCP), or a host Unix socket that Coi forwards into the container.
- **The base URL** of the tool, set in `[defaults.environment]`, points at the proxy.

## Reach the Proxy Over TCP

In `restricted` and `allowlist` mode, with `allow_host_access` off (the default), the host itself is closed to the container, except for the ports of `[[network.hosts]]` and `allowed_domains` entries that name a host address (see [Container Access to Host Services](Network-Isolation#container-access-to-host-services)). Open the proxy's port on the bridge gateway, which is an address of the host. Find the gateway address of the bridge your containers use:

```bash
incus network get incusbr0 ipv4.address   # e.g. 10.47.62.1/24
```

Start the proxy so that it listens on that address (`10.47.62.1:8080` in the example below). A proxy that listens only on `127.0.0.1` is not reachable from the container. Then, in `~/.coi/config.toml`:

```toml
[[network.hosts]]
ip        = "10.47.62.1"          # the bridge gateway address
hostnames = ["llm-proxy"]
ports     = [8080]                # the proxy port

[defaults.environment]
ANTHROPIC_BASE_URL   = "http://llm-proxy:8080"
ANTHROPIC_AUTH_TOKEN = "proxy-only-token"   # a token the proxy accepts, not the provider key
```

- `[[network.hosts]]` writes `llm-proxy` into the container's `/etc/hosts` and opens port 8080 on that host address; the entry opens no other port. It works in `restricted` and `allowlist` mode.
- `[[network.hosts]]` is honored only from trusted-scope config (`~/.coi/config.toml`, `$COI_CONFIG`, or a profile in `~/.coi/profiles/`), so it cannot go in a project `.coi/config.toml` (see [Static Host Entries](Static-Host-Entries)).
- In `open` mode Coi refuses an entry with `ports` and the session does not start. If some of your sessions use `open` mode, put the example in a profile instead, `~/.coi/profiles/llm-proxy/config.toml`, with `[environment]` in place of `[defaults.environment]`, and start the proxy sessions with `coi --profile llm-proxy` (see [Profiles](Profiles)).
- The variable names come from the tool. The example uses Claude Code: `ANTHROPIC_BASE_URL` points it at the proxy, and `ANTHROPIC_AUTH_TOKEN` is the credential it sends to the proxy. Without a credential variable, Claude Code keeps sending its saved claude.ai login (see Claude Code's [Connect Claude Code to an LLM gateway](https://code.claude.com/docs/en/llm-gateway-connect)). For another tool, use its own base-URL and credential variables.
- In `allowlist` mode the container does not need the provider's domain in `allowed_domains` for model requests: the proxy on the host sends them on. Instead of `[[network.hosts]]`, an `allowed_domains` entry with the address and port (`"10.47.62.1:8080"`) also opens the proxy; the base URL then uses the address, because allowlist mode blocks DNS and no name is written to `/etc/hosts`.

## Reach the Proxy Over a Unix Socket

When the proxy listens on a Unix socket, forward the socket with `[[sockets]]`, the same mechanism Coi uses for the SSH agent. No port on the host is opened:

```toml
[[sockets]]
host      = "~/.llm-proxy/proxy.sock"   # the proxy's socket on the host
container = "/run/llm-proxy.sock"       # absolute path in the container
env       = "LLM_PROXY_SOCK"            # optional: env var set to the container path
```

- Coi only places the socket in the container and, with `env`, sets `LLM_PROXY_SOCK` to its path. The client that talks to the proxy must open the socket itself, for example `curl --unix-socket "$LLM_PROXY_SOCK" http://localhost/`.
- Claude Code takes its base URL as an `http://` or `https://` URL (`ANTHROPIC_BASE_URL`), so use the TCP path for it, and for any other tool that takes only such a URL.
- The socket in the container is owned by the `code` user with mode `0600`.
- If the host socket does not exist when the session starts, Coi prints `Socket forwarding skipped: ... not accessible` and starts the session without it. Start the proxy first.
- `[[sockets]]` entries from a project `.coi/config.toml` apply only after `coi trust` (see [Configuration](Configuration#what-you-can-set-in-coiconfigtoml)).

## Keep the Key Out of the Container

The proxy keeps the key out only if nothing else puts it in. For the key the proxy holds, remove every other way it enters the container: `forward_env`, `[[credentials]]`, `[defaults.env_commands]` and `[defaults.environment]`. Coi also copies the tool's own credential files in. For Claude Code that is `~/.claude/.credentials.json`, your claude.ai login: a credential separate from the key the proxy holds. With `ANTHROPIC_AUTH_TOKEN` set, Claude Code sends the token, not the login, but the login file is still readable in the container. The proxy does not protect it (see [Limits and Non-Goals](Limits-and-Non-Goals#copied-tool-credentials-are-readable-by-the-agents-processes)).

## Limits

- **The agent can use the proxy for the whole session.** It cannot read the key, but every request it sends through the proxy, a prompt-injected one too, goes out with the key. The proxy is the policy point: budgets, rate limits, allowed models and request logs are its job, not Coi's.
- **Coi does not ship, start or watch the proxy.** You run it on the host and keep it running.
- **Other containers can reach the proxy too.** Coi filters host access only for its own containers in `restricted` and `allowlist` mode with `allow_host_access` off. A container in `open` mode or with `allow_host_access` on, or one that Coi did not start, can reach a proxy that listens on the bridge address. Make the proxy require its own token, as `ANTHROPIC_AUTH_TOKEN` above.

## See Also

- [Network Isolation](Network-Isolation) - network modes and host access
- [Static Host Entries](Static-Host-Entries) - `[[network.hosts]]` in detail
- [Configuration](Configuration) - `[[sockets]]`, `[defaults.environment]` and the trust model
- [Limits and Non-Goals](Limits-and-Non-Goals) - what a credential in the container can do
