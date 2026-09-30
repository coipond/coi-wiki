# Headless Orchestration (`coi tool spec`)

`coi tool spec` is a non-executing command that prints the exact launch
command + environment for the profile's tool inside an existing container.
An external orchestrator then runs that command through its own container
exec + tmux — owning the terminal, streaming, input, and lifecycle — instead of
coi launching and driving the tool itself.

This is the thin seam for building automation on top of coi: coi already knows
how to turn a profile into the right argv for any supported tool (session id,
resume, model, permission), and `coi tool spec` is how you get that out without
executing it. Adding a new tool stays a coi-side change with zero orchestrator
work.

> For scripting an interactive session (send a prompt, capture output),
> you usually want [Tmux Automation](Tmux-Automation) instead. Reach for
> `coi tool spec` when an external driver needs to own the tool's runtime.

# Fire-and-Forget Prompt Runs (`coi run --prompt…`)

For cron and other unattended automation, `coi run` can execute the AI agent
against a predefined prompt, run it to completion, and exit with the agent's
own status code — no external orchestrator, no TTY, no interaction:

```bash
coi run --prompt "Update all dependencies and open a PR"   # inline prompt text
coi run --prompt-file ./nightly.md                         # prompt from a host file
coi run --prompt-name nightly-maintenance                  # a named prompt (see below)
```

The three flags are mutually exclusive, and none can be combined with a
positional command (`coi run --prompt … -- <cmd>` is a usage error) — a headless
prompt run is the command.

**Requirements**

- **`[tool] permission_mode = "bypass"`** — a headless run has no terminal to
  approve tool use, so bypass mode is required (it is the default; see
  [Configuration](Configuration)).
- **Tool support** — currently supported for the `claude` tool.

## Named Prompts — the `[prompts]` Table

Instead of repeating prompt text, register reusable prompts in a `[prompts]`
table and invoke them by name with `--prompt-name`. Each entry is either an
inline string or a `{ file = "…" }` table whose path resolves relative to
the config file:

```toml
# ~/.coi/config.toml
[prompts]
tidy                = "Run the formatter and linter, commit any fixes."
nightly-maintenance = { file = "prompts/nightly.md" }   # relative to this config
```

`[prompts]` is also available in [profiles](Profiles) and inherits like the rest
of a profile's config.

> **Trusted-scope only.** `[prompts]` is honored only from trusted-scope
> config (`~/.coi/config.toml` / `$COI_CONFIG`). A `[prompts]` table in an
> untrusted project `./.coi/config.toml` (or a project-supplied profile) is
> stripped at load — an untrusted repo cannot inject prompts that run with
> bypass permissions.

## Cron Example

Because the run exits with the agent's status code, it drops straight into cron:

```cron
# Nightly maintenance at 03:00, logged for review
0 3 * * * cd ~/project && coi run --prompt-name nightly-maintenance >> ~/coi-nightly.log 2>&1
```

# `coi tool spec` — Launch Spec for an External Orchestrator

## Synopsis

```bash
coi tool spec --container <ctr> --session-id <id> --prompt-file <host> \
    [--system-prompt-file <host>] \
    [--continue[=<id>] | --resume-id <id> | --resume] \
    --json
```

## What It Prints

With `--json`, it emits a `command` (argv) and a tool-derived `env`:

```json
{
  "command": ["claude","--verbose","--permission-mode","bypassPermissions",
              "--session-id","<id>","\"$(cat /home/code/.coi/runs/<id>.prompt)\""],
  "env": { "ANTHROPIC_MODEL": "opus", "CLAUDE_CODE_EFFORT_LEVEL": "high" }
}
```

- **`command`** — the exact argv to run in the container, built from the profile's
  tool, so it is correct for any tool (claude/codex/opencode/pi/omp). The
  prompt is embedded via a staged-file `"$(cat …)"` reference, so arbitrary prompt
  text (quotes, newlines, `$()`) cannot corrupt the command. Run it through a shell
  (e.g. `bash -c "<command>"`) so the substitution expands.
- **`env`** — tool-derived environment only (model / effort). Secrets and
  auth stay with the caller — the orchestrator adds its own `--env` (e.g.
  `CLAUDE_CODE_OAUTH_TOKEN`) when it execs. No auth is baked into the spec.

### Prompt Delivery for Tools That Cannot Embed It

Tools that cannot put the initial prompt in argv (e.g. opencode) get a `prompt`
field instead — the in-container path to the staged prompt file — so the
orchestrator delivers it out-of-band after launch (e.g. `tmux load-buffer <path>`
+ `paste-buffer`, or [`coi tmux send`](Tmux-Automation)).

## Flags

| Flag | Description |
|------|-------------|
| `--container <ctr>` | **Required.** Existing container to build the launch spec for. |
| `--session-id <id>` | **Required.** Session id for the launch. Validated as a safe token (letters, digits, `.`, `_`, `-`; starts alphanumeric; max 64) since it becomes a filename component and is joined into the shell-run command. |
| `--prompt-file <host>` | Host file whose contents are staged as the tool's initial prompt. |
| `--system-prompt-file <host>` | Host file staged as the tool's system prompt (only for tools that support one, e.g. Claude's `--append-system-prompt`; codex/pi/omp reject it). |
| `--json` | Print the spec as JSON (the machine-readable form orchestrators consume). |

### Resume Strategies (at Most One)

| Flag | Meaning |
|------|---------|
| `--continue[=<id>]` | **Discover-or-fresh.** Looks up coi's **host-side** session store (`~/.coi/…`) via `DiscoverSessionID`; resumes if found, else starts fresh. Bare `--continue` targets `--session-id`. Correct when *coi* owns session persistence. |
| `--resume-id <id>` | **Assert, don't discover.** Builds a resume command for that exact id **verbatim** — no host-side lookup. For an orchestrator that owns session state itself (it restored the tool's state dir into the container and assigned the id). |
| `--resume-latest` | **Resume-latest**, no id. Resumes the most recent conversation the tool can find. |

`--continue` / `--resume-id` / `--resume-latest` are mutually exclusive (a
usage error otherwise). Each renders in the tool's own vocabulary — e.g.
`--resume-id` becomes claude `--resume <id>`, codex `resume <id>`, opencode
`--session <id>`; pi/omp resume the latest conversation.

> **No bare `--resume` on `coi tool spec`.** That name belongs to
> coi's root `--resume` (resume a coi session by id), which is inherited here but
> not read by this command. To avoid silently swallowing it, `coi tool spec
> --resume` is rejected with an error pointing at `--resume-latest` (no id) or
> `--resume-id <id>` (a specific id).

## Example: Orchestrator-Owned Resume

```bash
# The orchestrator has already restored the archived state dir into <ctr>.
coi tool spec --container <ctr> \
    --session-id <new-run-id> \
    --resume-id <prior-run-id> \
    --prompt-file <host-prompt> --json
# → { "command": ["claude","--verbose","--permission-mode","bypassPermissions",
#                 "--resume","<prior-run-id>","\"$(cat …/<new-run-id>.prompt)\""],
#     "env": { "ANTHROPIC_MODEL": "opus" } }
```

The orchestrator then runs that argv in its own `coi container exec -t <ctr>
--env … -- bash -c "<command>"` inside a tmux it controls, captured to its own
stream, with input via `send-keys` — adding its own auth `--env`.

## Why Not Have coi Drive the Tool?

Having coi own the runtime (launch the tool in a coi-side tmux and hand back
handles) meant an external driver had to mirror it — tmux-in-tmux streaming lag,
auth not re-applied on a reused container, and no usable terminal input. `coi
tool spec` inverts that: coi computes, the driver runs. It reuses the driver's
already-working terminal path and stays a thin, tool-agnostic layer.

## See Also

- [Tmux Automation](Tmux-Automation) - `coi tmux send` / `capture`, which compose with `tool spec` for out-of-band prompt delivery and I/O
- [Container Operations](Container-Operations) - `coi container exec` and low-level container control
- [Supported Tools](Supported-Tools) - The per-tool command shapes `tool spec` renders
- [Profiles](Profiles) - Bundle tool + model + limits an orchestrator selects with `--profile`
