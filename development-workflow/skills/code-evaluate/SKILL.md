---
name: code-evaluate
description: >
  Evaluates a completed backend or frontend phase implementation against the phase
  plan and acceptance criteria. Runs Tier 1 commands, spawns the code-evaluator
  agent for Tier 2 inspection, and surfaces Tier 3 items for user review. Up to 3
  rounds with a fix loop before escalating to the user. Trigger when the user says
  "evaluate the code", "run code eval", or invokes /code-evaluate with a phase
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

Verify these files exist:
- `{phase-folder}/PHASE_PLAN.md` — required; stop if missing
- `{phase-folder}/ACCEPTANCE_CRITERIA.md` — required; if missing, tell the user to run `/plan-evaluate` first

Capture the current HEAD SHA: `git rev-parse HEAD`

Determine the base SHA for the diff using `git log --oneline -20` to identify the commit before this component's implementation began. Record it — use the same base SHA across all rounds so the diff always covers the full implementation.

Initialize `{phase-folder}/review-history.md` from `templates/review-history.template.md` if it does not exist.

---

## Phase 2 — Evaluation Loop (up to 3 rounds per component)

Evaluate components in order: backend first if both are in scope, then frontend.

### Per component, per round:

**Step A — Run Tier 1 commands**

Read `{phase-folder}/ACCEPTANCE_CRITERIA.md` for the Tier 1 command list. Run every command:
- Do not stop on failure — run all commands and collect all results
- Capture all output to a file: `git diff {base-sha}..HEAD > /tmp/{phase-slug}-tier1-r{round}.txt` — use a separate file per round so prior rounds' output stays accessible

Actually capture Tier 1 output:
```bash
{tier-1-command} > /tmp/{phase-slug}-{component}-tier1-r{round}.txt 2>&1; echo "Exit: $?" >> /tmp/{phase-slug}-{component}-tier1-r{round}.txt
```

Run each command and capture to the same file. Record exit codes.

**Step B — Generate diff**

```bash
git diff {base-sha}..HEAD > /tmp/{phase-slug}-{component}-r{round}.diff
```

**Step C — Spawn code-evaluator**

Spawn `subagent_type: "development-tools:code-evaluator"` (or `"general-purpose"` if not available) with `model: "sonnet"`.

Pass file paths — not content. The evaluator reads what it needs.

```
Files to read:
- Phase plan: {phase-folder}/PHASE_PLAN.md
- Acceptance criteria: {phase-folder}/ACCEPTANCE_CRITERIA.md
- Tier 1 results file: /tmp/{phase-slug}-{component}-tier1-r{round}.txt
- Diff file: /tmp/{phase-slug}-{component}-r{round}.diff
- Review history: {phase-folder}/review-history.md

Context:
- Base SHA: {base-sha}
- HEAD SHA: {current HEAD}
- Round: {N}
- Component: {backend | frontend}
```

Wait for the evaluator to return.

**Step D — Check verdict**

- **Approved** or **Approved with concerns**: exit the component loop. Record the Tier 3 checklist from the evaluator. Move to the next component (or Phase 3 if all components done).
- **Needs fixes** and round < 3: continue to Step E.
- **Needs fixes** and round = 3: go to Phase 4 (escalation for this component).

**Step E — Dispatch fix agent**

Read the Critical findings from `{phase-folder}/review-history.md` for this round.

Spawn the relevant implementation agent with `model: "sonnet"`:
- Backend findings → `{backend-agent}`
- Frontend findings → `{frontend-agent}`

Pass file paths — not content:
```
Files to read:
- Phase plan: {phase-folder}/PHASE_PLAN.md
- Acceptance criteria: {phase-folder}/ACCEPTANCE_CRITERIA.md

Fix these Critical and Important findings (paste verbatim — they are short):
{findings from review-history.md}

Instruction: fix only what is listed. Do not refactor unrelated code.
Instruction: write or fix tests as needed.
Commit when complete: [{plan-name}] Fix {component} evaluation findings (round {N})
```

Wait for the agent to complete. Increment round number and return to Step A.

---

## Phase 3 — Report

After all components pass evaluation (or max rounds reached and escalated):

**Tell the user:**

1. **Tier 1 + 2 status**: All checks passed / N issues resolved after M rounds
2. **Tier 3 checklist** (verbatim from ACCEPTANCE_CRITERIA.md Tier 3): what to verify manually in the browser or app
3. Any Important findings noted but not blocking (with their IDs for reference)
4. Next step: the user should work through the Tier 3 checklist

---

## Phase 4 — Escalation (max rounds reached with unresolved Critical findings)

If 3 rounds completed for a component with Critical findings still open:

1. Extract all open `[ ]` Critical findings from `{phase-folder}/review-history.md`
2. Present to the user:
   - The unresolved Critical findings verbatim (ID, description, file:line)
   - What the fix agent attempted (from the round history)
   - A recommendation: what the user should do manually or what decision is needed
3. Ask the user to resolve them, then re-invoke `/code-evaluate {phase-folder} {component}` to continue
