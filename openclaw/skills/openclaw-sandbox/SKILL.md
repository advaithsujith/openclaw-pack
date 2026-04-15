---
name: openclaw-sandbox
description: Understands nono security sandbox constraints for OpenClaw agents. Use when running inside a nono sandbox, when operations fail with permission errors, or when coordinating with other sandboxed OpenClaw instances.
---

# nono Sandbox Awareness for OpenClaw

You are running inside a **nono security sandbox**. nono enforces OS-level capability restrictions using Landlock (Linux) and Seatbelt (macOS). These are kernel-enforced boundaries — they cannot be bypassed from within the session.

## What the sandbox allows

Your active profile (`openclaw`) grants access to:

- **Your instance directory**: `~/.openclaw`, `~/.openclaw-agent1`, `~/.openclaw-agent2`, or `~/.openclaw-agent3` depending on which instance you are
- **Shared config**: `~/.config/openclaw`
- **Root config file**: `~/.openclaw.json`
- **Coordination bus**: `$TMPDIR/openclaw-$UID/` — shared across all OpenClaw sandbox instances on this machine
- **Network**: not blocked — outbound connections are allowed
- **Working directory**: read access only

Everything else is blocked at the kernel level. There is no escalation path from inside the sandbox.

## Multi-agent coordination bus

When multiple OpenClaw instances run inside separate nono sandboxes simultaneously, they share one writable space: `$TMPDIR/openclaw-$UID/`.

This is the **coordination bus**. Use it to:

- Avoid duplicating work across agents
- Signal task ownership or completion
- Share lightweight state between instances

### Conventions

```
$TMPDIR/openclaw-$UID/
├── tasks/       ← task queue: each agent writes/reads task files here
├── locks/       ← file-based locks: agent claims a task by creating a lock file
└── state/       ← shared ephemeral state: status, progress, results
```

To claim a task:
```bash
COORD="$TMPDIR/openclaw-$UID"
mkdir -p "$COORD/locks"
# Try to create a lock file — fails if another agent already owns it
(set -C; echo "$$" > "$COORD/locks/task-name.lock") 2>/dev/null && echo "claimed" || echo "already taken"
```

To broadcast state:
```bash
echo '{"status":"working","task":"summarize-emails"}' > "$TMPDIR/openclaw-$UID/state/agent1.json"
```

These files are temporary — they live only for the lifetime of the current boot session.

## When an operation is denied

If a file read, write, edit, or shell command fails with a permission error (EPERM, EACCES, "Operation not permitted", "Permission denied"):

1. **Do NOT retry** the same operation or attempt alternative paths
2. **Do NOT try workarounds** — sudo, chmod, copying files to allowed locations, or asking the user to run commands from another terminal
3. **Run `nono why`** to get the exact diagnosis:

```bash
nono why --path /path/that/failed --op read 2>/dev/null
```

This tells you the reason, the blocking rule, and the suggested fix.

## Helping the user fix it

After diagnosing with `nono why`, offer two options:

1. **Quick fix** — restart the session with the path explicitly allowed:
   ```
   nono run --allow /path/to/needed -- openclaw
   ```

2. **Persistent fix** — add the path to a custom nono profile. Run `nono profile guide` to get the schema, write a profile to `~/.config/nono/profiles/<name>.json`, then start with:
   ```
   nono run --profile <name> -- openclaw
   ```

## Checking current capabilities

If `NONO_CAP_FILE` is set, it points to a JSON file listing exactly what this sandbox can access:

```bash
cat "$NONO_CAP_FILE" | jq .
```

Fields:
- `fs`: array of filesystem capabilities (`path`, `resolved`, `access`)
- `net_blocked`: `true` if network is blocked
