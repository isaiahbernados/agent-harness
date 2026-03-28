# agent-harness

A Claude Code plugin that spins up a **three-agent team** to plan and implement software features end-to-end — automatically.

Invoke it with `/harness` and describe what you want to build. Three specialized agents take it from there: one writes a product spec, one reviews it for testability, and one implements it. The evaluator then tests the result and sends it back for fixes if anything fails.

## How it works

```
You ──► /harness "add dark mode to the settings page"
              │
              ▼
        ┌─────────────────────────────────────────────┐
        │                  Team Lead                   │
        │  (coordinates agents, monitors progress)     │
        └──────┬──────────────────────────────────────┘
               │ spawns
       ┌───────┼───────────────┐
       ▼       ▼               ▼
   Planner  Evaluator      Generator
      │         │               │
      │  writes │  reviews      │
      │  spec ──► spec ─────────► implements
      │                         │
      │              grades ◄───┘
      │                │
      └────────────────┘
             │
             ▼
     PASS → FEATURE_PLAN.md
     FAIL → Generator retries (up to 2x)
```

### The agents

**Planner** reads your codebase, writes a product spec with concrete acceptance criteria, and iterates with the Evaluator until every criterion is automatable and unambiguous.

**Evaluator** runs twice:
1. Before generation — reviews the spec for testability, flags vague criteria, forces rewrites
2. After generation — tests the implementation against each criterion using browser MCP tools, CLI, or API calls; writes a graded evaluation with root-cause analysis for failures

**Generator** waits for spec approval, implements the feature criterion-by-criterion with checkpoint commits, self-evaluates before handing off, and uses the Evaluator's root-cause analysis to fix failures strategically (not symptomatically).

### Output artifacts

All output lands in `.features/{team-name}/` in your project:

| File | Written by | Contents |
|------|-----------|----------|
| `FEATURE_SPEC.md` | Planner | Product spec + acceptance criteria |
| `SPEC_REVIEW.md` | Evaluator | Testability review + testing notes |
| `CHECKPOINT.md` | Generator | Progress log + key decisions |
| `EVALUATION.md` | Evaluator | Graded criteria + root causes |
| `FEATURE_PLAN.md` | Team Lead | Final summary of everything |

## Prerequisites

Agent Teams must be enabled in `~/.claude/settings.json`:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

If it's missing, Claude will add it and prompt you to restart.

## Installation

```bash
claude plugin install https://github.com/isaiahbernados/agent-harness
```

Then restart Claude Code. The `/harness` skill will be available in any project.

## Usage

```
/harness add a search bar that filters the product list in real time
```

```
/harness build a CSV export button for the reports table
```

```
/harness implement OAuth login with GitHub
```

You can also trigger it naturally — the skill activates on requests like "build X", "implement X", "add X to the app", or "let's make X".

### Retry behavior

If the Evaluator finds failures after the first implementation pass, the Generator gets the failed criteria plus root-cause analysis and retries. This happens automatically up to 2 times. If it's still failing after that, Claude will surface the situation and ask how you'd like to proceed.

## Why a spec-first loop?

LLMs tend to write code that *looks* right but fails on edge cases the prompt didn't mention. By forcing a written spec through a testability gate before a single line of code is written, the harness:

- Catches vague requirements early (before the Generator wastes time on them)
- Gives the Generator a precise, graded checklist to work from
- Gives the Evaluator objective criteria to test against — no subjective judgment calls
- Produces a paper trail (`.features/`) that survives context resets

## License

MIT
