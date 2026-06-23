---
name: code-evaluate
description: >
  Evaluates a completed backend or frontend phase implementation against the phase
  plan and acceptance criteria. Runs Tier 1 commands directly, spawns the code-evaluator
  agent for Tier 2 inspection, and surfaces Tier 3 items for user review. Up to 3 rounds
  with a fix loop before escalating to the user. Also called internally by
  build-feature-from-plan after each component implementation. Trigger when the user
  says "evaluate the code", "run code eval", or invokes /code-evaluate with a phase
  folder and component.
argument-hint: "<phase-folder> [backend|frontend]"
---

# Code Evaluate

Runs the code evaluation loop for a completed backend or frontend phase component.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Development Workflow` section
2. Read:
   - `backend-agent` → agent for backend fixes (default: `development-tools:python-api-developer`)
   - `frontend-agent` → agent for frontend fixes (default: `development-tools:vue-developer`)

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

## Arguments

- `<phase-folder>` (required) — the phase folder containing `PHASE_PLAN.md` (e.g., `plans/010-share-with-friends/phase-1`)
- `[backend|frontend]` (optional) — which component to evaluate. Default: `backend`, then `frontend` sequentially.

---

## Phase 1 — Setup

Read in parallel:
- `{phase-folder}/PHASE_PLAN.md` — required; stop if missing
- `{phase-folder}/ACCEPTANCE_CRITERIA.md` — required; if missing, tell the user to run `/plan-evaluate` first
- `{phase-folder}/review-history.md` — read if present

Capture the current HEAD SHA: `git rev-parse HEAD`

Determine the base SHA for the diff:
- Read `{phase-folder}/phase-status.md` for the recorded implementation start SHA if available
- Otherwise: use `git log --oneline -10` to identify the commit before this component's implementation began, and use that SHA
- Record the base SHA for use across all rounds

Initialize `{phase-folder}/review-history.md` from the template if it does not exist.

---

## Phase 2 — Evaluation Loop (up to 3 rounds per component)

Evaluate components in order: backend first if both are in scope, then frontend.

### Per component, per round:

**Step A — Run Tier 1 commands**

Read the Tier 1 section of `{phase-folder}/ACCEPTANCE_CRITERIA.md`. Run every command listed:
- Capture stdout, stderr, and exit code for each
- Do not stop on failure — run all commands and collect all results
- Record results as a structured list: command, exit code, output (truncated to relevant portion if long)

**Step B — Generate diff**

Run: `git diff {base-sha}..HEAD > /tmp/phase-{N}-{component}-diff.txt`

Use the base SHA captured in Phase 1 so the diff always covers the full implementation including any fixes from prior rounds.

**Step C — Spawn code-evaluator**

Spawn `subagent_type: "development-tools:code-evaluator"` (or `"general-purpose"` if not available).

Provide:
- Full content of `{phase-folder}/PHASE_PLAN.md`
- Full content of `{phase-folder}/ACCEPTANCE_CRITERIA.md`
- Tier 1 results (each command, its exit code, and relevant output)
- Base SHA: `{base-sha}`
- HEAD SHA: current HEAD
- Diff file path: `/tmp/phase-{N}-{component}-diff.txt`
- The current round number
- Review history file path: `{phase-folder}/review-history.md`
- The component being evaluated: backend or frontend

Wait for the evaluator to return.

**Step D — Check verdict**

- **Approved** or **Approved with concerns**: exit the component loop. Record the Tier 3 checklist from the evaluator. Move to the next component (or Phase 3 if all components done).
- **Needs fixes** and round < 3: continue to Step E.
- **Needs fixes** and round = 3: go to Phase 4 (escalation for this component).

**Step E — Dispatch fix agent**

Read the Critical findings from `{phase-folder}/review-history.md` for this round.

Spawn the relevant implementation agent:
- Backend findings → `{backend-agent}`
- Frontend findings → `{frontend-agent}`

Brief the agent with:
- The full content of `{phase-folder}/PHASE_PLAN.md`
- The full content of `{phase-folder}/ACCEPTANCE_CRITERIA.md`
- The Critical and Important finding IDs and descriptions (verbatim from review-history.md)
- Instruction: fix only what is listed — do not refactor unrelated code
- Instruction: write or fix tests as needed based on the findings
- Instruction to commit when complete with message: `[{plan-name}] Fix {component} evaluation findings (round {N})`

Wait for the agent to complete. Increment round number and return to Step A.

---

## Phase 3 — Report

After all components pass evaluation (or max rounds reached and escalated):

**Tell the user:**

1. **Tier 1 + 2 status**: All checks passed / N issues resolved after M rounds
2. **Tier 3 checklist** (verbatim from ACCEPTANCE_CRITERIA.md Tier 3): what to verify manually in the browser or app
3. Any Important findings noted but not blocking (with their IDs for reference)
4. Next step: the user should work through the Tier 3 checklist, then confirm for the Tier 4 product acceptance step

**Update `{phase-folder}/phase-status.md`**:
- Mark backend and frontend Tier 1 and Tier 2 evaluation checkboxes as complete

---

## Phase 4 — Escalation (max rounds reached with unresolved Critical findings)

If 3 rounds completed for a component with Critical findings still open:

1. Extract all open `[ ]` Critical findings from `{phase-folder}/review-history.md`
2. Present to the user:
   - The unresolved Critical findings verbatim (ID, description, file:line)
   - What the fix agent attempted (from the round history)
   - A recommendation: what the user should do manually or what decision is needed
3. Ask the user to resolve them, then re-invoke `/code-evaluate {phase-folder} {component}` to continue
