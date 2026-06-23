---
name: build-feature-from-plan
description: >
  Orchestrates building a feature from a PLAN.md and its phase folders. For each
  phase: spawns the backend agent (which writes tests inline), runs the code-evaluate
  loop, then spawns the frontend agent (which writes tests inline), runs the
  code-evaluate loop, then pauses for user validation of Tier 3 + Tier 4 items.
  After all phases, runs a security scan. Trigger when the user says "build the
  feature", "implement the plan", "start building", or invokes
  /build-feature-from-plan with a plan folder path.
argument-hint: "<plan-folder> [phase-number|all]"
---

# Build Feature From Plan

You are the orchestrator for building a feature from a PLAN.md and its phase folders. You delegate implementation to specialized agents, run evaluation loops after each component, and coordinate user validation before closing out.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Development Workflow` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plans (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)
   - `backend-agent` → agent for backend implementation (default: `development-tools:python-api-developer`)
   - `frontend-agent` → agent for frontend implementation (default: `development-tools:vue-developer`)

If a specified agent is not available, fall back to `general-purpose`.

## Arguments

- `<plan-folder>` (required) — the folder containing `PLAN.md` and phase subfolders (e.g., `plans/010-share-with-friends`)
- `[phase-number|all]` (optional) — the phase to build (e.g., `1`), or `all` to build all phases sequentially. Default: `all`

---

## Phase 0 — Setup

1. Read `{plan-folder}/PLAN.md` and note:
   - The plan name (e.g., `010-share-with-friends`) — used in commit messages
   - The implementation phases and their folders
   - The Critical Scope Flags — treat these as non-negotiable throughout

2. Identify the target phase folder(s):
   - If `phase-number` is specified: `{plan-folder}/phase-{N}/`
   - If `all`: all `phase-{N}/` subdirectories in dependency order

3. For each target phase, verify these files exist:
   - `PHASE_PLAN.md` — required
   - `ACCEPTANCE_CRITERIA.md` — required; if missing, tell the user to run `/plan-evaluate {plan-folder}` first
   - `TEST_PLAN_PHASE.md` — include if present

4. Run the test baseline before touching any code (use project commands from `CLAUDE.md`):
   - Backend test suite
   - Frontend test suite

   If tests fail, stop and ask the user whether to fix the failures first or proceed (marking known-broken tests as skipped per project convention).

5. Unless the user explicitly provides a branch name or says to work on the current branch, create a feature branch from `main`:
   ```bash
   git checkout main && git pull && git checkout -b feature/{plan-name}
   ```

6. Record the pre-implementation HEAD SHA for use as the diff base in evaluation:
   ```bash
   git rev-parse HEAD
   ```
   Write it to `{phase-folder}/phase-status.md` under a "Implementation base SHA" note.

---

## Phase 1 — Backend Implementation + Evaluation

Repeat for each target phase in order.

### Step A — Spawn backend agent

Spawn `{backend-agent}` to implement the backend component.

Brief the agent with:
- Full content of `{phase-folder}/PHASE_PLAN.md`
- Full content of `{phase-folder}/TEST_PLAN_PHASE.md` (if present)
- Full content of `{phase-folder}/ACCEPTANCE_CRITERIA.md`
- Instruction: implement **only the Backend section** steps, in the order listed
- Instruction: **write tests alongside the implementation** (TDD) — the test scenarios in TEST_PLAN_PHASE.md define what to cover; write them as you implement each step
- Instruction: tests must satisfy the Tier 1 commands listed in ACCEPTANCE_CRITERIA.md
- The Critical Scope Flags for this phase — treat these as non-negotiable
- Instruction to commit when done: `[{plan-name}] Phase {N} backend implementation`
- The content of `{phase-folder}/memory.md` if it contains prior implementation notes

Wait for the backend agent to complete.

### Step B — Run code evaluation loop

Run the evaluation inline (up to 3 rounds):

**Round setup**: read `{phase-folder}/ACCEPTANCE_CRITERIA.md` Tier 1 commands. Run each command and capture exit code + output.

**Spawn code-evaluator** (`development-tools:code-evaluator` or `general-purpose`):
- PHASE_PLAN.md content
- ACCEPTANCE_CRITERIA.md content
- Tier 1 results
- Base SHA (from Phase 0 Step 6), HEAD SHA
- Diff: `git diff {base-sha}..HEAD`
- Round number
- Review history: `{phase-folder}/review-history.md`
- Component: `backend`

If verdict is **Needs fixes**:
- Read Critical findings from `{phase-folder}/review-history.md`
- Spawn `{backend-agent}` with the findings and instruction to fix only what is listed
- Agent commits: `[{plan-name}] Phase {N} backend evaluation fixes (round {N})`
- Re-run evaluation (max 3 rounds total)

If max rounds exceeded with open Critical findings: surface to user (see escalation note at end of skill).

Mark backend evaluation checkboxes complete in `{phase-folder}/phase-status.md`.

---

## Phase 2 — Frontend Implementation + Evaluation

### Step A — Spawn frontend agent

Spawn `{frontend-agent}` to implement the frontend component.

Brief the agent with:
- Full content of `{phase-folder}/PHASE_PLAN.md`
- Full content of `{phase-folder}/TEST_PLAN_PHASE.md` (if present)
- Full content of `{phase-folder}/ACCEPTANCE_CRITERIA.md`
- Instruction: implement **only the Frontend section** steps, in the order listed
- Instruction: **write tests alongside the implementation** (TDD) — the test scenarios in TEST_PLAN_PHASE.md define what to cover; write them as you implement each component
- Instruction: tests must satisfy the Tier 1 commands listed in ACCEPTANCE_CRITERIA.md
- The Critical Scope Flags for this phase
- Instruction to commit when done: `[{plan-name}] Phase {N} frontend implementation`
- The content of `{phase-folder}/memory.md` if it contains prior implementation notes

Wait for the frontend agent to complete.

### Step B — Run code evaluation loop

Same structure as Phase 1 Step B, with component: `frontend` and agent: `{frontend-agent}`.

Mark frontend evaluation checkboxes complete in `{phase-folder}/phase-status.md`.

---

## Phase 3 — User Validation Gate

**Stop here and present to the user:**

1. What was built — a one-paragraph summary of the backend and frontend work completed for this phase
2. **Tier 3 checklist** (verbatim from `{phase-folder}/ACCEPTANCE_CRITERIA.md` Tier 3) — what to verify manually in the browser or app. Present each item as a numbered step.
3. **Tier 4 checklist** (verbatim from `{phase-folder}/ACCEPTANCE_CRITERIA.md` Tier 4) — product acceptance questions for the user to answer
4. Test suite status (pass/fail counts from the last evaluation round)
5. Any Important findings noted but not blocking (for awareness)

**Ask the user to:**
- Work through the Tier 3 checklist and report any visual/UX issues
- Answer the Tier 4 product acceptance questions
- Explicitly confirm when ready to proceed

**Do not proceed until the user explicitly confirms.**

If the user reports issues or requests changes:
1. Determine whether the change is backend, frontend, or both
2. Spawn the relevant agent with a precise description of what to change and why
3. Run the relevant Tier 1 commands and re-run the code-evaluate loop
4. Present the Tier 3 checklist again and ask the user to re-validate
5. Repeat until the user explicitly confirms

Once confirmed, mark Tier 3 and Tier 4 checkboxes complete in `{phase-folder}/phase-status.md`.

---

## Repeat Phases 1–3 for each subsequent phase

After the user validates Phase 1, proceed to Phase 1–3 for Phase 2 (if it exists), and so on.

Each phase is fully validated before the next phase begins.

---

## Phase 4 — Security Scan

Once all phases are confirmed by the user, invoke the `/scan` skill (or the project's security scan command) on the changed files.

If the scan reports issues:
- **Critical or high severity**: spawn the relevant agent to fix. Re-run the scan after fixes.
- **Medium severity**: present to the user with a recommendation and let them decide.
- **Low / informational**: report to the user; do not block.

---

## Phase 5 — Close Out

1. Run the full verification checklist from `PLAN.md` one final time. If any step fails, fix it before closing out.

2. For each phase folder, update `{phase-folder}/memory.md` with:
   - Non-obvious implementation decisions made during this build
   - Deviations from the plan and why
   - Challenges encountered and how they were resolved
   - Anything a future engineer would need to know that is not obvious from the code

   Do not document routine changes — focus on the surprising, non-obvious, and risk-mitigating.

3. Mark all remaining tasks complete in each `{phase-folder}/phase-status.md`.

4. Report to the user:
   - What was built and committed (by phase)
   - Any deferred items or known limitations
   - The branch name and suggested PR title: `[{plan-name}] {feature description}`

---

## Escalation note

If the code evaluation loop reaches 3 rounds with Critical findings still open for any component, stop and present the situation to the user:
- The unresolved findings verbatim
- What the fix agent attempted
- Options: fix manually, adjust the plan, or proceed accepting the risk

Do not skip to the next phase until the current phase's evaluation is resolved or explicitly accepted.
