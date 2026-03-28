---
name: harness
description: "Spin up a three-agent team (Planner, Evaluator, Generator) to plan AND implement a software feature end-to-end. Use this skill whenever the user wants to build, implement, add, spec out, or design any feature — even if they don't say 'harness' explicitly. Trigger on: 'build a feature', 'implement [X]', 'add [X] to the app', 'let's build [X]', 'help me design [X]', 'plan [X]', 'I want [X] in my app', '/harness'. Also use for any 'make the app do X' request where X requires writing new code."
user-invocable: true
---

# Agent Harness

Orchestrates a three-agent team:

1. **Planner** — expands the user's brief into a product spec with automatable acceptance criteria
2. **Evaluator** — reviews the spec for testability *before* generation, then grades the final implementation
3. **Generator** — implements the feature against the approved spec, using git checkpoints throughout

The Evaluator acts twice: first as a spec reviewer, then as a QA tester. This catches untestable criteria early — before the Generator wastes time implementing them.

Agent prompts live in `agents/` — read the relevant file just before spawning each agent.

---

## Your Role (Team Lead)

You coordinate but don't write spec or code. You:
- Set up the team, tasks, and spawn all three agents
- Monitor progress via `TaskList` periodically — don't just wait for messages
- Route messages between agents when needed
- Synthesize the final output

---

## Step 1: Create the Team

Slugify the feature name (lowercase, hyphens, max 20 chars). Call `TeamCreate`:
- `team_name`: `feat-{slug}` (e.g. `feat-dark-mode`)
- `description`: the feature description

## Step 2: Create Tasks

Call `TaskCreate` for each, passing `team_name`:

| # | Title |
|---|-------|
| 1 | Write product spec |
| 2 | Review spec for testability |
| 3 | Implement feature |
| 4 | Evaluate and grade |

## Step 3: Spawn All Three Agents

Read `agents/planner.md`, `agents/generator.md`, and `agents/evaluator.md`. For each, replace `{TEAM_NAME}` with the actual team name and `{FEATURE_DESCRIPTION}` with the user's feature description verbatim — then use the file contents as that agent's prompt.

Spawn all three **in the same turn** using the `Agent` tool with `team_name`, `subagent_type: "general-purpose"`, and `name` set for each.

**Expected message flow:**
1. Planner writes spec → messages evaluator (spec review) AND generator (generator waits)
2. Evaluator reviews spec → messages planner if revisions needed, then approves → messages generator
3. Generator implements → messages evaluator
4. Evaluator grades → messages team-lead with verdict

## Step 4: After Agents Finish

Once the evaluator messages you with the final verdict:

- **PASS**: write `.features/{TEAM_NAME}/FEATURE_PLAN.md` (see format below), announce to the user.
- **FAIL**: read `EVALUATION.md` to extract the root cause hypotheses, then relay *both* the failed criteria AND the root cause analysis to `"generator"`. Ask generator to fix and re-submit. The evaluator will do another full pass. Allow up to 2 re-evaluation cycles.

**If still failing after 2 cycles**, surface the situation to the user with a clear choice:

> "After 2 attempts the evaluator is still failing on: [specific criteria]. Root cause: [from EVALUATION.md]. Options:
> (a) **Relay new instructions** — tell me what to adjust and I'll pass it to the generator
> (b) **Abort and restart** — I'll shut the team down; re-invoke the skill with a refined description (the prior attempt will be in `.features/{TEAM_NAME}/` for context)
> (c) **Take over manually** — I'll shut the team down and you can edit the files directly"

Wait for the user's choice before acting.

### Observability

Between spawning agents and receiving the final verdict, periodically call `TaskList` (with `team_name`) to check each agent's progress. If an agent appears stuck (in_progress for a long time with no messages), proactively check in with a message.

### Shutdown

When done:
```
SendMessage "planner":   { "type": "shutdown_request" }
SendMessage "generator": { "type": "shutdown_request" }
SendMessage "evaluator": { "type": "shutdown_request" }
```

---

## FEATURE_PLAN.md Format

Written to `.features/{TEAM_NAME}/FEATURE_PLAN.md`:

```markdown
# Feature: {Feature Name}

## Summary
{What was built}

## Result
Verdict: PASS / FAIL
{2-3 sentence summary from evaluator}

## Product Spec
{Contents of FEATURE_SPEC.md}

## Spec Review
{Contents of SPEC_REVIEW.md}

## Evaluation
{Contents of EVALUATION.md}
```

---

## Settings Prerequisite

Agent Teams require this in `~/.claude/settings.json`:
```json
"env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
```

If missing, add it and ask the user to restart Claude Code.
