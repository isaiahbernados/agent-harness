---
name: iterate
description: "Resume a stopped or failed harness run from where it left off. Reads the existing .features/{name}/ artifacts and re-enters the loop at the right phase. Use when a harness run was interrupted, the generator never finished, or the evaluator returned a FAIL and you want to retry. Trigger on: 'resume feat-X', 'continue the harness run for X', 'retry the harness', '/iterate', '/iterate {team-name}'."
user-invocable: true
---

# Iterate Skill

Resumes an in-progress or failed harness run without starting from scratch. Reads `.features/{name}/` to determine the resume point, then re-enters the agent loop at the right phase.

Agent prompts live in `agents/` — read the relevant file just before spawning each agent.

---

## Your Role (Team Lead)

1. Determine which run to resume
2. Read its artifacts to understand the current state
3. Spawn the right agents to continue
4. Handle the verdict the same way as `/harness`

---

## Step 1: Identify the Run

**If a team name was provided** (e.g. `/iterate feat-dark-mode`): use it directly.

**If no team name was provided**: list all `.features/*/` directories that do NOT have a `FEATURE_PLAN.md` file. Present them to the user and ask which to resume.

## Step 2: Determine Resume Point

Read the contents of `.features/{TEAM_NAME}/`:

| State | What to do |
|-------|-----------|
| No `FEATURE_SPEC.md` | Tell user there's nothing to resume — run `/harness` instead |
| Has `FEATURE_PLAN.md` with `Verdict: PASS` | Tell user this run already passed — show them the plan |
| Has `FEATURE_SPEC.md` + `SPEC_REVIEW.md`, no `EVALUATION.md` | Generator was interrupted before finishing → spawn Generator only |
| Has `FEATURE_SPEC.md` but no `SPEC_REVIEW.md` | Spec review never finished → tell user to run `/spec` or `/harness` again |
| Has `EVALUATION.md` with `Verdict: FAIL` | Evaluation failed → spawn Generator (with eval context) + Evaluator |
| Has `FEATURE_PLAN.md` with `Verdict: FAIL` | Previous run concluded as failed → spawn Generator + Evaluator for a fresh retry |

## Step 3: Spawn Agents

Read the relevant agent files. Replace `{TEAM_NAME}` with the team name and `{FEATURE_DESCRIPTION}` with the original feature description (read from `FEATURE_SPEC.md`'s one-line summary or problem statement if not otherwise available).

**For Generator-only resume** (spec done, generator interrupted):
- Spawn Generator. It will read `CHECKPOINT.md` if present and continue from there.
- The Evaluator will be messaged by the Generator when it finishes — you don't need to spawn it separately in this case. Actually, spawn the Evaluator too so it's ready to receive the message.

**For Generator + Evaluator resume** (evaluation failed):
- Spawn both in the same turn.
- Generator reads `EVALUATION.md` for root cause context before writing any code.

## Step 4: Handle the Verdict

Same as `/harness`:
- **PASS**: write `.features/{TEAM_NAME}/FEATURE_PLAN.md`, announce to user
- **FAIL**: relay root cause to generator, allow up to 2 more retry cycles
- **Still failing after 2 cycles**: surface to user with 3-option escalation

### Shutdown

When done:
```
SendMessage "generator": { "type": "shutdown_request" }
SendMessage "evaluator": { "type": "shutdown_request" }
```

---

## Settings Prerequisite

Agent Teams require this in `~/.claude/settings.json`:
```json
"env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
```
