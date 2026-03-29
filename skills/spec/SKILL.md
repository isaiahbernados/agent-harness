---
name: spec
description: "Write and review a product spec without implementing anything. Runs the Planner and Evaluator agents to produce a testable spec with automatable acceptance criteria — but stops before the Generator touches any code. Use when the user wants to plan or spec a feature first, review requirements before committing to building, or produce a spec for a human to implement. Trigger on: 'spec out X', 'write a spec for X', 'plan X without building', 'I want a spec for X', '/spec'."
user-invocable: true
---

# Spec Skill

Runs the first two phases of the harness — Planner + Evaluator spec review — without generating any code.

Use this when you want a tight, testable spec before committing to implementation. The output can feed directly into `/harness` or be handed to a human developer.

Agent prompts live in `agents/` — read the relevant file just before spawning each agent.

---

## Your Role (Team Lead)

You coordinate but don't write spec. You:
- Set up the team and tasks, spawn both agents
- Monitor progress via `TaskList` periodically
- Announce the final spec to the user when done

---

## Step 1: Create the Team

Slugify the feature name (lowercase, hyphens, max 20 chars). Call `TeamCreate`:
- `team_name`: `spec-{slug}` (e.g. `spec-dark-mode`)
- `description`: the feature description

## Step 2: Create Tasks

Call `TaskCreate` for each, passing `team_name`:

| # | Title |
|---|-------|
| 1 | Write product spec |
| 2 | Review spec for testability |

## Step 3: Spawn Both Agents

Read `agents/planner.md` and `agents/evaluator.md`. For each, replace `{TEAM_NAME}` with the actual team name and `{FEATURE_DESCRIPTION}` with the user's feature description verbatim.

Spawn both **in the same turn** using the `Agent` tool with `team_name`, `subagent_type: "general-purpose"`, and `name` set for each.

**Expected message flow:**
1. Planner writes spec → messages evaluator
2. Evaluator reviews spec → messages planner if revisions needed
3. Once all criteria approved, evaluator messages team-lead

## Step 4: After Agents Finish

When the evaluator messages you that the spec is approved:
- Read `.features/{TEAM_NAME}/FEATURE_SPEC.md` and `SPEC_REVIEW.md`
- Announce the spec to the user, including:
  - The one-line summary
  - The full list of acceptance criteria
  - A note that they can run `/harness` with the same description to implement it

### Shutdown

When done:
```
SendMessage "planner":   { "type": "shutdown_request" }
SendMessage "evaluator": { "type": "shutdown_request" }
```

---

## Settings Prerequisite

Agent Teams require this in `~/.claude/settings.json`:
```json
"env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
```

If missing, add it and ask the user to restart Claude Code.
