---
name: debug
description: "Spawn a three-agent debugging team (Reproducer, Diagnoser, Fixer) to isolate and fix a bug. The Reproducer writes a failing test, the Diagnoser finds the root cause, and the Fixer implements a targeted fix and verifies the test passes. Use when the user reports a bug, unexpected behavior, or broken functionality. Trigger on: 'debug X', 'there's a bug where X', 'fix this bug', 'something is broken', '/debug'."
user-invocable: true
---

# Debug Skill

Orchestrates a three-agent debugging team:

1. **Reproducer** — writes a failing test that isolates the bug in the smallest possible scope
2. **Diagnoser** — reads the failing test output and traces the root cause through the source code
3. **Fixer** — implements a minimal fix targeting the root cause, verifies the reproducer test now passes, runs regression checks

The agents work sequentially via messages (Reproducer → Diagnoser → Fixer) but are all spawned at once.

Agent prompts live in `agents/` — read the relevant file just before spawning each agent.

---

## Your Role (Team Lead)

You coordinate but don't write code. You:
- Set up the team, tasks, and spawn all three agents
- Monitor progress via `TaskList` periodically
- Handle escalation if the fix fails repeatedly

---

## Step 1: Create the Team

Slugify the bug description (lowercase, hyphens, max 20 chars). Call `TeamCreate`:
- `team_name`: `debug-{slug}` (e.g. `debug-null-login`)
- `description`: the bug description

## Step 2: Create Tasks

Call `TaskCreate` for each, passing `team_name`:

| # | Title |
|---|-------|
| 1 | Reproduce the bug |
| 2 | Diagnose root cause |
| 3 | Fix and verify |

## Step 3: Spawn All Three Agents

Read `agents/reproducer.md`, `agents/diagnoser.md`, and `agents/fixer.md`. For each, replace `{TEAM_NAME}` with the actual team name and `{BUG_DESCRIPTION}` with the user's bug description verbatim.

Spawn all three **in the same turn** using the `Agent` tool with `team_name`, `subagent_type: "general-purpose"`, and `name` set for each.

**Expected message flow:**
1. Reproducer writes failing test → messages diagnoser
2. Diagnoser identifies root cause → messages fixer
3. Fixer implements fix → messages team-lead with result

## Step 4: After Agents Finish

When the fixer messages you:

- **Fixed**: announce to the user with a summary of what was broken and how it was fixed. Include the test location so they can run it.
- **Still failing**: read `DIAGNOSIS.md` and the fixer's message. Relay the root cause and remaining failure to fixer. Allow up to 2 retry cycles.

**If still failing after 2 cycles**, surface to the user:

> "After 2 attempts the fix is not holding. The reproducer test is still failing. Root cause from diagnosis: [from DIAGNOSIS.md]. Options:
> (a) **Relay new instructions** — tell me what to try and I'll pass it to the fixer
> (b) **Abort** — I'll shut the team down; artifacts are in `.debug/{TEAM_NAME}/` for reference
> (c) **Take over manually** — I'll shut the team down and you can edit the files directly"

### Shutdown

When done:
```
SendMessage "reproducer": { "type": "shutdown_request" }
SendMessage "diagnoser":  { "type": "shutdown_request" }
SendMessage "fixer":      { "type": "shutdown_request" }
```

---

## Settings Prerequisite

Agent Teams require this in `~/.claude/settings.json`:
```json
"env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
```

If missing, add it and ask the user to restart Claude Code.
