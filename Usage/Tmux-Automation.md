# Tmux Automation

Interact with running AI coding sessions for automation workflows.

> Building an external orchestrator that owns the tool's runtime (its own
> terminal, streaming, and lifecycle)? See [Headless Orchestration](Headless-Orchestration)
> (`coi tool spec`) — it prints the tool's launch command + env for you to run,
> and composes with the `coi tmux send` / `capture` commands below for
> out-of-band prompt delivery and I/O.

## List Sessions

```bash
# List all active tmux sessions
coi tmux list
```

## Send Commands

Send commands or prompts to a running AI session:

```bash
# Send a prompt to the AI
coi tmux send coi-abc12345-1 "write a hello world script"

# Send an exit command
coi tmux send coi-abc12345-1 "/exit"

# Send multiple commands
coi tmux send coi-abc12345-1 "run tests"
coi tmux send coi-abc12345-1 "fix any failures"
```

## Capture Output

Capture the current visible output from a session:

```bash
# Capture output from specific session
coi tmux capture coi-abc12345-1
```

## Automation Use Cases

### Batch Processing

Send prompts in sequence and poll for completion before moving on:

```bash
#!/bin/bash

# Helper: wait until the session output stops changing
wait_for_idle() {
  local session="$1"
  local prev=""
  while true; do
    current=$(coi tmux capture "$session")
    [ "$current" = "$prev" ] && break
    prev="$current"
    sleep 3
  done
}

for prompt in "create tests" "document code" "optimize performance"; do
  coi tmux send my-session "$prompt"
  wait_for_idle my-session
  coi tmux capture my-session > "output-${prompt// /-}.txt"
done
```

> **Polling, Not Sleep**
> Fixed `sleep` durations are unreliable — tasks complete at different speeds. The `wait_for_idle` helper above polls until output stabilizes. For AI tools that print a prompt marker when ready (e.g., `>`), grep for that marker instead of comparing full output.

### CI/CD Integration

```bash
#!/bin/bash
# Start a persistent session, run an AI task, capture results
# (persistence comes from config: [container] persistent = true)

# Launch and wait for the session to be ready
coi shell
SESSION=$(coi list --running --format=json | jq -r '.active_containers[0].name')

# Send task and wait for completion
coi tmux send "$SESSION" "analyze code for security issues"

# Poll until output stabilises (AI tool returned to prompt)
prev=""
while true; do
  current=$(coi tmux capture "$SESSION")
  [ "$current" = "$prev" ] && break
  prev="$current"
  sleep 5
done

coi tmux capture "$SESSION" > security-report.txt
coi shutdown "$SESSION"
```

### Monitoring

```bash
# Periodically check AI session output
while true; do
  coi tmux capture my-session | grep -i "error\|warning" && notify-send "Issue detected"
  sleep 300
done
```

### Unattended Tasks

```bash
# Queue multiple AI tasks to run overnight
# (persistence via config: [container] persistent = true)
coi shell --slot 1 &
sleep 5
coi tmux send coi-workspace-1 "refactor module A"
coi tmux send coi-workspace-1 "refactor module B"
coi tmux send coi-workspace-1 "refactor module C"
coi tmux send coi-workspace-1 "/exit"
```

## Notes

- Sessions use tmux internally
- Standard tmux commands work after attaching with `coi attach`
- Use container name (from `coi list`) to target specific sessions
- Commands are sent as-is to the AI tool's input
- Capture shows the current terminal buffer (visible output only)

## See Also

- [Headless Orchestration](Headless-Orchestration) - `coi tool spec`: drive any tool from an external orchestrator
- [Container Operations](Container-Operations) - Full container management command reference
- [Container Lifecycle and Sessions](Container-Lifecycle-and-Sessions) - How sessions and tmux windows relate
