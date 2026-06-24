---
name: build-feature-from-plan
description: >
  Orchestrates building a feature from a PLAN.md and its phase folders. For each
  phase: dispatches one agent per task (implementation + reviewer gate), then pauses
  for user validation when a phase has Tier 3 visual items. After all phases, runs a
  security scan. Trigger when the user says "build the feature", "implement the plan",
  "start building", or invokes /build-feature-from-plan with a plan folder path.
argument-hint: "<plan-folder> [phase-number|all]"
---

# Build Feature From Plan

You are the orchestrator. You dispatch one implementer agent per task and one reviewer agent per task, tracking progress in a durable ledger. You keep your own context lean: large artifacts (diffs, reviewer findings) move as file paths; dispatch prompts are short.

## Core principles

- **Task file = brief**: each task file (`task-NN-*.md`) is already self-contained. Pass its path to the implementer — do not copy its content inline or write a separate brief file.
- **Durable ledger**: the ledger (`.build-ledger.md`) is the orchestrator's sole source of truth for resumption. After compaction, trust the ledger and `git log` — not your own recollection.
- **Lean dispatch prompts**: pass file paths, not content. The only things inline in a dispatch prompt are short operational instructions (3–5 lines). Everything else is a file reference.
- **Model per task**: every Agent spawn specifies `model` explicitly. An omitted model silently inherits your session's most expensive model.
- **Continuous execution**: do not pause between tasks. User gates happen at phase-level milestones only, and only when the phase has Tier 3 visual items.

## Configuration

Read from the project's `CLAUDE.md` at startup:

1. Look for a `## Development Workflow` section
2. Read (use defaults if not present):
   - `backend-agent` → agent for backend tasks (default: `development-tools:python-api-developer`)
   - `frontend-agent` → agent for frontend tasks (default: `development-tools:vue-developer`)

Fall back to `general-purpose` if a specified agent is unavailable.

## Arguments

- `<plan-folder>` (required) — folder containing `PLAN.md` and phase subfolders
- `[phase-number|all]` (optional) — build only `phase-{N}`, or all phases in order. Default: `all`

---

## Model selection

Read the task file's **Model** line and use it. Specify `model` on every Agent spawn.

| Model | When | Roles |
|---|---|---|
| `haiku` | Single file, mechanical transcription, complete spec, prop-only changes | Implementer |
| `sonnet` | Multi-file, auth logic, new complex components, view wiring | Implementer, reviewer, fix agent |
| `opus` | Final security scan | Security scan only |

Reviewer agents are always `sonnet`. Fix agents match the implementer or step up one tier if the fix requires more judgment than the original implementation.

---

## Phase 0 — Setup

### Step 0 — Check the progress ledger

```bash
cat {plan-folder}/.build-ledger.md 2>/dev/null
```

If the ledger exists, skip all tasks already marked `DONE`. Resume at the first incomplete task. After compaction, trust the ledger and `git log --oneline` over your own recollection.

If no ledger, create `{plan-folder}/.build-ledger.md`:
```
# Build Ledger: {plan-name}
Started: {YYYY-MM-DD}
Branch: (set after checkout)
Base SHA: (set after checkout)

## Tasks
```

### Step 1 — Permission setup

If this is the first build in this project, recommend the user run `/fewer-permission-prompts` before proceeding. The task loop runs many git commands and test suites. Pre-allowlisting prevents constant prompts.

### Step 2 — Read PLAN.md (once)

Read `{plan-folder}/PLAN.md`. Extract and keep in working memory:
- Plan name (for commit messages)
- Phase numbers and descriptions
- Critical Scope Flags (verbatim)

Do not re-read PLAN.md after this step.

### Step 3 — Identify target phases and tasks

- `phase-number` specified: only `{plan-folder}/phase-{N}/`
- `all`: all `phase-{N}/` directories in numeric order

For each target phase, read `phase-{N}/tasks/TASK_INDEX.md` to get the ordered task list.

If task files don't exist (plan predates this skill version): treat the whole phase as one task using `PHASE_PLAN.md` as the spec, and note this to the user.

Verify for each phase:
- `PHASE_PLAN.md` exists
- `ACCEPTANCE_CRITERIA.md` exists (if missing, tell user to run `/plan-evaluate {plan-folder}` first)
- `tasks/TASK_INDEX.md` exists

### Step 4 — Baseline + branch

Run the test baseline (from `CLAUDE.md`). If tests fail, ask the user whether to fix them or proceed with known failures noted.

```bash
git checkout main && git pull && git checkout -b feature/{plan-name}
```

Record in the ledger:
```
Branch: feature/{plan-name}
Base SHA: {git rev-parse HEAD}
```

---

## Phase 1 — Execute Tasks

For each target phase, execute every task in `TASK_INDEX.md` order. **Check the ledger before each task — skip tasks already marked DONE.**

### Per-task loop (Steps A–E)

#### Step A — Record pre-task SHA

```bash
git rev-parse HEAD
```

Store as `{task}-base-sha`. This is the diff base for the reviewer.

#### Step B — Dispatch the implementer

Read the task file's **Model** line. Determine agent type from the task path (tasks in the Backend section → `{backend-agent}`; Frontend section → `{frontend-agent}`).

Dispatch prompt (keep to 5 lines — no content pasted inline):
```
Read {phase-folder}/tasks/{task-file}.md — it is your complete spec.
Implement everything listed. Write or update the tests in the Test cycle section.
Commit using the exact message in the task file.
Write a one-paragraph result to {phase-folder}/tasks/{task-slug}-result.md:
  what was implemented, tests run (N passed / M failed), commit SHA, any deviations.
Return only: status (DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED), commit SHA, test summary.
```

Wait for the compact return.

**Handle status:**
- **DONE**: proceed to Step C
- **DONE_WITH_CONCERNS**: read the result file; if the concern is about correctness or scope, treat as Critical and go to Step D; if it's an observation, note in ledger and proceed to Step C
- **NEEDS_CONTEXT**: provide the missing context and re-dispatch (same model)
- **BLOCKED**: escalate to user (see Escalation note)

#### Step C — Reviewer gate

Generate the diff:
```bash
git diff {task-base-sha}..HEAD > /tmp/{task-slug}.diff
```

Dispatch prompt for `development-tools:code-evaluator` with `model: "sonnet"`:
```
Spec: {phase-folder}/tasks/{task-file}.md (read the Reviewer checklist section)
Implementer result: {phase-folder}/tasks/{task-slug}-result.md
Diff: /tmp/{task-slug}.diff
Phase Critical Scope Flags: {paste the flags inline — they are short}
Write your findings to {phase-folder}/tasks/{task-slug}-review.md
Return: verdict (Approved | Needs fixes), one-line summary of findings.
```

**Check verdict:**
- **Approved** or **Approved with concerns**: go to Step E
- **Needs fixes**: go to Step D

#### Step D — Fix loop (up to 3 rounds)

Read Critical and Important findings from `{phase-folder}/tasks/{task-slug}-review.md`.

Dispatch the fix agent (same agent type as implementer, model `sonnet` or up one tier if needed):
```
Fix findings in {phase-folder}/tasks/{task-slug}-review.md (Critical and Important only).
Spec: {phase-folder}/tasks/{task-file}.md
Fix only what is listed — do not refactor unrelated code.
Run the task's test cycle. Confirm it passes.
Append fix summary to {phase-folder}/tasks/{task-slug}-result.md.
Return: status, new commit SHA, test summary.
```

After the fix agent returns, regenerate the diff (`/tmp/{task-slug}.diff`) and re-dispatch the reviewer (Step C). Max 3 reviewer rounds per task.

If 3 rounds complete with Critical findings still open: escalate to user (see Escalation note).

#### Step E — Update ledger and TASK_INDEX

Append to `.build-ledger.md`:
```
- {task-slug}: DONE (commit {SHA}, {N} tests passed, reviewer: Approved)
```

Update `{phase-folder}/tasks/TASK_INDEX.md`: check the box for this task.

---

## Phase 2 — User Validation Gate (per phase)

After all tasks in a phase are complete, read `{phase-folder}/ACCEPTANCE_CRITERIA.md`:

- **Phase has Tier 3 items (visual/UX)**: pause for user validation
- **Phase has only Tier 1/2 items** (backend-only phases): continue automatically — do not pause

**When pausing, present:**

1. What was built — one paragraph summarizing the phase (from task result files, not re-reading source)
2. **Tier 3 checklist** (verbatim from ACCEPTANCE_CRITERIA.md) — steps to verify in the browser or app
3. **Tier 4 checklist** (verbatim from ACCEPTANCE_CRITERIA.md) — product acceptance questions
4. Test summary (aggregate from ledger entries for this phase)
5. Any DONE_WITH_CONCERNS notes from the ledger

**Do not proceed until the user explicitly confirms.**

If the user reports issues:
1. Identify which task(s) are affected
2. Dispatch the fix agent for each, re-run the reviewer gate (Steps C/D)
3. Update the ledger and TASK_INDEX, then re-present the Tier 3 checklist

Once confirmed, append to the ledger:
```
- Phase {N}: User Confirmed
```

---

## Phase 3 — Repeat for subsequent phases

Continue Phases 1–2 for each remaining phase. Each task implementer starts fresh. The only shared state is the ledger and git history.

---

## Phase 4 — Security Scan

Once all phases are confirmed, run the security scan with `model: "opus"`.

- **Critical / high**: spawn fix agent (`sonnet`), re-scan
- **Medium**: present to user with recommendation
- **Low / informational**: report only

---

## Phase 5 — Close Out

1. Run the full verification checklist from `PLAN.md`. Fix any failures.

2. For each phase, update `{phase-folder}/memory.md` with non-obvious decisions, deviations from the plan, and anything a future engineer would need to know.

3. Clean up temp diff files:
   ```bash
   rm -f /tmp/{plan-name}-*.diff
   ```
   Result and review files in `tasks/` stay — they are the permanent record of what was built and why.

4. Report to the user:
   - What was built (by phase, with commit ranges from the ledger)
   - Any deferred items or known limitations
   - Branch name and suggested PR title: `[{plan-name}] {feature description}`

---

## Escalation note

If a task's fix loop reaches 3 rounds with Critical findings still open:

1. Surface the open findings verbatim (from the review file), what was attempted, and options: fix manually, adjust the task spec, or proceed accepting the risk
2. If the user fixes manually: re-run the reviewer gate (Step C) before marking the task done
3. Do not proceed to the next task until the current task is resolved or explicitly accepted
