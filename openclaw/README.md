# openclaw nono Pack

`openclaw` is a `nono` pack for running OpenClaw AI agents inside a nono security sandbox.

It ships an improved nono profile that covers all standard OpenClaw instance directories and defines a shared coordination bus for multi-agent setups.

## What It Does

This pack solves two problems:

**1. Multi-instance filesystem coverage**
The built-in `openclaw` profile only allows `~/.openclaw`. If you run a second or third agent instance (`~/.openclaw-agent1`, `~/.openclaw-agent2`), those directories are blocked. This pack's profile covers all standard instance directories so any agent variant runs without filesystem errors.

**2. Multi-agent coordination**
When multiple OpenClaw instances run inside separate nono sandboxes simultaneously, they need a way to share state without breaking isolation. This pack documents and relies on `$TMPDIR/openclaw-$UID/` as a shared coordination bus — writable by all sandboxed OpenClaw instances running as the same user.

## Coordination Bus

All sandboxed OpenClaw instances on the same machine share:

```
$TMPDIR/openclaw-$UID/
├── tasks/    ← shared task queue
├── locks/    ← file-based ownership locks (use noclobber to claim)
└── state/    ← ephemeral per-agent status
```

Agents can use this to avoid duplicating work, signal task ownership, or broadcast lightweight status — without any network calls or shared database.

## Installation

Requires nono >= 0.42.0 (`cargo install nono-cli`).

```bash
nono pull always-further/openclaw
```

## Usage

Start a main OpenClaw session:
```bash
nono run --profile openclaw -- openclaw
```

Start a named agent instance:
```bash
nono run --profile openclaw --home ~/.openclaw-agent1 -- openclaw
```

## Included Artifacts

| Artifact | Type | Purpose |
|---|---|---|
| `policy.json` | `profile` | nono sandbox profile covering all standard OpenClaw directories |
| `skills/openclaw-sandbox/SKILL.md` | `instruction` | Teaches the agent its sandbox constraints and coordination bus usage |
| `bin/nono-hook.sh` | `hook` | Injects capability context on permission denial |

## Policy Details

The profile:
- Extends `default` (inherits all standard security groups)
- Allows `~/.openclaw`, `~/.openclaw-agent1/2/3`, `~/.config/openclaw`, `~/.openclaw.json`
- Allows `$TMPDIR/openclaw-$UID` as the coordination bus
- Adds `node_runtime`, `linux_runtime_state`, `linux_sysfs_read`, `git_config` security groups
- Sets `ipc_mode: shared_memory_only` explicitly
- Network: not blocked
- Workdir: read-only
- Non-interactive

## Package Metadata

- Name: `openclaw`
- Platforms: `macos`, `linux`
- License: `Apache-2.0`
- Min nono version: `0.42.0`

## Directory Layout

```
openclaw/
├── bin/
│   └── nono-hook.sh
├── package.json
├── policy.json
├── README.md
└── skills/
    └── openclaw-sandbox/
        └── SKILL.md
```

> Note: `hooks/hooks.json` was removed in 0.1.3. Hook registration is handled directly by the `hook` artifact type in `package.json`.
