---
name: build-feature-from-plan
description: >
  Orchestrates building a feature from a PLAN.md. Delegates backend work to the
  backend agent and frontend work to the frontend agent. Runs lint after each agent,
  spawns test-writing agents with test_gaps.md context, pauses for user validation,
  then runs a security scan. Trigger when the user says "build the feature",
  "implement the plan", "start building", or invokes /build-feature-from-plan
  with a plan folder path.
argument-hint: "<plan-folder> [phase-number|all]"
---

# Build Feature From Plan

You are the orchestrator for building a feature from a PLAN.md. You delegate implementation to specialized subagents, enforce quality gates, and coordinate user validation before shipping.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Development Workflow` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plans (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)
   - `backend-agent` → agent to use for backend implementation (default: `development-workflow:python-api-developer`)
   - `frontend-agent` → agent to use for frontend implementation (default: `development-workflow:vue-developer`)

If `backend-agent` or `frontend-agent` is not specified, use the defaults. If the specified agent is not available, fall back to `general-purpose`.

## Arguments

- `<plan-folder>` (required) — the folder containing `PLAN.md` (e.g., `plans/010-share-with-friends`)
- `[phase-number|all]` (optional) — phase to build (e.g., `1`), or `all` to build all phases. Default: `all`

---

## Phase 0 — Setup

1. Read `{plan-folder}/PLAN.md` in full. Note:
   - The plan name (used in commit messages, e.g., `010-share-with-friends`)
   - The backend steps and frontend steps for the target phase(s)
   - The Critical Scope Flags section — these are the highest-risk items
   - The Verification section — this becomes your final checklist

2. Check whether `{plan-folder}/TEST_PLAN.md` exists. If it does, read it and note:
   - The test files it references and their P0/P1 priorities
   - The Risk Areas section

3. Note the path `{plan-folder}/test_gaps.md` — you will pass it to the test-writing agents later. It may not exist yet; that is fine.

4. Run the test baseline before touching any code (use project commands from `CLAUDE.md`):
   - Backend test suite
   - Frontend test suite

   If tests fail, stop and ask the user whether to fix the failures first or proceed (marking known-broken tests as skipped per project convention).

5. Unless the user explicitly provides a branch name or says to work on the current branch, create a feature branch from `main`:
   ```bash
   git checkout main && git pull && git checkout -b feature/{plan-name}
   ```

6. Write a task list to `{plan-folder}/phase-status.md` broken into:
   - Backend tasks (from the Backend section of PLAN.md)
   - Frontend tasks (from the Frontend section of PLAN.md)
   - Test tasks (from TEST_PLAN.md P0/P1 items, if it exists)
   - QA validation (pending user sign-off)

---

## Phase 1 — Backend Implementation

Spawn the backend agent (`{backend-agent}`) to implement all backend steps for the target phase(s).

Brief the agent with:
- The full content of `{plan-folder}/PLAN.md`
- Instruction to implement **only the Backend section** steps, in the order listed
- The Critical Scope Flags from the plan — instruct the agent to treat these as non-negotiable
- Instruction **not** to write tests (tests come in a later phase)
- Instruction to commit when done using the format: `[{plan-name}] Backend implementation`
- The content of `{plan-folder}/memory.md` if it exists (prior implementation notes)
- The `plans-dir` config value for writing test_gaps.md if needed

Wait for the backend agent to complete.

After the backend agent finishes:

1. Run the backend lint command (from `CLAUDE.md`). If lint fails, spawn the backend agent again with the full lint output and instruct it to fix all violations. Repeat until lint is clean.

2. Run the backend test suite. If tests that were passing before this phase now fail, spawn the backend agent to fix the regressions, providing the full test output. Repeat until green.

3. Update `{plan-folder}/phase-status.md` to mark backend tasks complete.

---

## Phase 2 — Frontend Implementation

Spawn the frontend agent (`{frontend-agent}`) to implement all frontend steps for the target phase(s).

Brief the agent with:
- The full content of `{plan-folder}/PLAN.md`
- Instruction to implement **only the Frontend section** steps, in the order listed
- Any screenshots referenced in the plan (provide the screenshots directory path if noted in the plan)
- Instruction **not** to write tests (tests come in a later phase)
- Instruction to commit when done using the format: `[{plan-name}] Frontend implementation`
- The content of `{plan-folder}/memory.md` if it exists
- The `plans-dir` config value for writing test_gaps.md if needed

Wait for the frontend agent to complete.

After the frontend agent finishes:

1. Run the frontend lint commands (from `CLAUDE.md`). If lint fails, spawn the frontend agent again with the full lint output and instruct it to fix all violations. Repeat until lint is clean.

2. Run the frontend test suite. If tests that were passing before this phase now fail, spawn the frontend agent to fix the regressions. Repeat until green.

3. Update `{plan-folder}/phase-status.md` to mark frontend tasks complete.

---

## Phase 3 — Test Implementation

Spawn two agents **in parallel in a single message**:

### Agent A: Backend Tests

Spawn the backend agent (`{backend-agent}`) to write backend tests.

Brief the agent with:
- The Tier 2 (Integration Tests) and Tier 3 (Contract Tests) sections of `{plan-folder}/TEST_PLAN.md`
- The Risk Areas section from the test plan
- Instruction: "Implement only **P0 and P1** tests. Skip P2 and P3 entirely."
- Context about test_gaps.md: "Read `{plan-folder}/test_gaps.md` if it exists — it contains implementation details discovered during development that may alter the test plan."
- Instruction to commit when done: `[{plan-name}] Backend tests`

### Agent B: Frontend Tests

Spawn the frontend agent (`{frontend-agent}`) to write frontend tests.

Brief the agent with:
- The Tier 1 (Unit Tests) section of `{plan-folder}/TEST_PLAN.md`
- Instruction: "Implement only **P0 and P1** tests. Skip P2 and P3 entirely."
- Context about test_gaps.md: "Read `{plan-folder}/test_gaps.md` if it exists."
- Instruction to commit when done: `[{plan-name}] Frontend tests`

Wait for both agents to complete.

After both test agents finish:

1. Run lint for both backend and frontend. Fix any violations by spawning the relevant agent with the lint output.

2. Run both test suites. If any tests fail:
   - Backend failures → spawn the backend agent with the full test output to fix them
   - Frontend failures → spawn the frontend agent with the full test output to fix them
   Repeat until both suites are green.

3. Update `{plan-folder}/phase-status.md` to mark test tasks complete.

---

## Phase 4 — User Validation Gate

**Stop here and present to the user:**

1. What was built — a one-paragraph summary of the backend, frontend, and test work completed
2. How to validate it — reproduce the Verification steps from `PLAN.md` verbatim
3. Current test suite status (pass/fail counts from the last run)
4. Any items skipped or deferred

**Ask the user to validate and report back. Do not proceed until the user explicitly confirms.**

If the user reports issues or requests changes:

1. Determine whether each change is backend, frontend, or both.
2. Spawn the relevant agent with a precise description of what needs to change and why.
3. After completion, run lint and the relevant test suite; fix any failures.
4. If tests need updating as a result of changes, spawn the appropriate test agent (P0/P1 only).
5. Ask the user to validate again. Repeat until the user explicitly confirms.

---

## Phase 5 — Security Scan

Once the user confirms the feature is working, invoke the `/scan` skill (or the project's security scan command) to run a static analysis scan on the changed files.

If the scan reports issues:
- **Critical or high severity**: spawn the relevant agent to fix. Re-run the scan after fixes.
- **Medium severity**: present to the user with a recommendation and let them decide.
- **Low / informational**: report to the user; do not block on these.

---

## Phase 6 — Close Out

1. Run the full verification checklist from `PLAN.md` one final time.

   If any step fails, fix it before closing out.

2. Update `{plan-folder}/memory.md` with:
   - Non-obvious implementation decisions made during this build
   - Any deviations from the plan and why
   - Challenges encountered and how they were resolved
   - Anything a future engineer would need to know that is not obvious from the code

   Do **not** document routine changes — focus on the surprising, non-obvious, and risk-mitigating decisions.

3. Update `{plan-folder}/phase-status.md` to mark all tasks complete.

4. Report back to the user:
   - What was built and committed
   - Any deferred items or known limitations
   - The branch name and suggested PR title: `[{plan-name}] {feature description}`
