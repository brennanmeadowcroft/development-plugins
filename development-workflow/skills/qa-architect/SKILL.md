---
name: qa-architect
description: >
  Generates a TEST_PLAN.md for a feature plan that already exists. Spawns the qa-architect
  agent against an existing PLAN.md and writes the test plan alongside it. Use when you have
  a PLAN.md and want a test plan generated (or regenerated) without running the full
  work-breakdown. Trigger when the user says "create a test plan", "generate test plan for",
  or invokes /qa-architect with a plan folder path.
argument-hint: "<plan-folder>"
---

# QA Architect

This skill generates a test plan for an existing feature plan by spawning the qa-architect agent.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Development Workflow` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plans (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)

## Arguments

- `<plan-folder>` (required) — the folder containing `PLAN.md` (e.g., `plans/010-share-with-friends`)

## Steps

1. Verify `{plan-folder}/PLAN.md` exists. If not, tell the user and stop.

2. Spawn the qa-architect agent as `subagent_type: "development-workflow:qa-architect"` (or `"general-purpose"` if not available).

   Provide the agent with:
   - Full content of `{plan-folder}/PLAN.md`
   - The feature plan folder path: `{plan-folder}`
   - The `plans-dir` and `docs-dir` config values
   - Instruction to write `{plan-folder}/TEST_PLAN.md`

3. Report back to the user:
   - The path to the generated TEST_PLAN.md
   - Test counts by tier (unit, integration, contract, E2E)
   - The top 2–3 risk areas identified
   - Any notable gaps or deferred items
