---
name: plan-evaluate
description: >
  Runs the planner/evaluator loop on a PLAN.md. Spawns the document-evaluator agent
  with the plan-evaluation rubric; addresses Critical findings by spawning targeted
  planning agents; up to 3 rounds. On approval, generates ACCEPTANCE_CRITERIA.md per
  phase and materializes phase folders with PHASE_PLAN.md, TEST_PLAN_PHASE.md,
  task files, and memory.md. Use after work-breakdown to validate a plan, or
  to re-evaluate an existing plan independently. Trigger when the user says "evaluate
  the plan", "run plan eval", or invokes /plan-evaluate with a plan folder path.
argument-hint: "<plan-folder>"
---

# Plan Evaluate

Runs the planner/evaluator loop on a PLAN.md. Up to 3 rounds of critique and revision before surfacing unresolved issues to the user.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Development Workflow` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plans (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

## Arguments

- `<plan-folder>` (required) — the folder containing `PLAN.md` (e.g., `plans/010-share-with-friends`)

---

## Phase 1 — Setup

Locate these files (read them yourself — do not pass their content to sub-agents):
- `{plan-folder}/PLAN.md` — required; stop if missing
- `{plan-folder}/TEST_PLAN.md` — note if present or absent
- `{plan-folder}/discovery/PRD.md` — required for coverage evaluation; stop if missing
- `{plan-folder}/review-history.md` — read if present (prior evaluation rounds)

Locate the rubric: `rubrics/plan-evaluation-rubric.md` in the development-tools plugin directory.

If `review-history.md` does not exist, initialize it from `templates/review-history.template.md` with the feature name and document type "PLAN.md".

---

## Phase 2 — Evaluation Loop (up to 3 rounds)

Track the current round number (starting at 1, incrementing after each Planner Response).

### Per round:

**Step A — Spawn document-evaluator**

Spawn `subagent_type: "development-tools:document-evaluator"` (or `"general-purpose"` if not available).

Pass file paths — not content. The evaluator reads what it needs.

```
Files to read:
- PLAN.md: {plan-folder}/PLAN.md
- TEST_PLAN.md: {plan-folder}/TEST_PLAN.md (present: {yes|no})
- PRD.md: {plan-folder}/discovery/PRD.md
- Rubric: {rubric-path}
- Review history: {plan-folder}/review-history.md
- Feature directory: {plan-folder}
- Round: {N}

Instruction: Evaluate PLAN.md and TEST_PLAN.md against the rubric and the PRD.
Append your findings to review-history.md. If your verdict is Approved or Approved
with concerns, also generate ACCEPTANCE_CRITERIA.md for each implementation phase.
```

Wait for the evaluator to return.

**Step B — Check verdict**

- **Approved** or **Approved with concerns**: exit the loop. Go to Phase 3.
- **Needs revision** and round < 3: continue to Step C.
- **Needs revision** and round = 3: go to Phase 4 (escalation).

**Step C — Address Critical findings**

Read `{plan-folder}/review-history.md` to extract the Critical findings from this round.

For each Critical finding, determine the responsible planning agent:

| Finding type | Agent to spawn |
|---|---|
| Backend steps, endpoints, migrations, models, schemas, auth | `development-tools:tech-lead` |
| Frontend components, views, props, routes | `development-tools:ui-architect` |
| Test strategy gaps, missing scenarios, tier misassignment | `development-tools:qa-architect` |
| PRD coverage gaps (P0/P1 requirements absent) | Address inline: read the PRD and add missing steps to PLAN.md yourself |

Group findings by agent so each agent is spawned once per round.

Brief each agent with:
- Path to PLAN.md: `{plan-folder}/PLAN.md` — the agent reads it
- The specific Critical finding IDs and descriptions (verbatim from review-history.md) it must address — these are short, paste inline
- Instruction to return only the revised content for its sections, not a full rewrite
- Instruction not to change sections unrelated to its findings

After all agents return: update `{plan-folder}/PLAN.md` with the revised sections.

**Step D — Write planner response to review-history.md**

Append immediately after the evaluator section for this round:

```
### Planner Response — {YYYY-MM-DD}

#### Addressed
- [x] #{id} — {what was changed in PLAN.md, with section/step reference}

#### Not Addressed
- [ ] #{id} — {reason: deferred to implementation, disagree, or unlocks only in a later phase}
```

Increment round number and return to Step A.

---

## Phase 3 — Materialize Phase Folders (post-approval)

After approval, create a self-contained folder for each implementation phase.

**Step 1 — Parse PLAN.md phases**

Identify each implementation phase in PLAN.md. Phases are typically named "Phase 1", "Phase 2", etc. A plan with no explicit phase sections has a single phase (Phase 1 = the whole plan).

Apply the phase quality check before materializing:
1. **Visual verifiability**: each frontend phase must include at least one mounted view, not just unwired components. If a phase only produces shared components, either merge them into the phase that wires them or annotate their Tier 3 as "unit-test verified only; visual check deferred to Phase {next}."
2. **Backend-before-frontend**: no frontend phase precedes a backend phase it depends on.

**Step 2 — For each phase, create `{plan-folder}/phase-{N}/`**

**`PHASE_PLAN.md`** — extracted vertical slice for human reference:
- Context paragraph for this phase
- Backend steps for this phase (in dependency order)
- Frontend steps for this phase (in dependency order)
- Critical Scope Flags relevant to this phase
- Verification steps for this phase

Use `templates/PHASE_PLAN.template.md` as the structure reference.

**`TEST_PLAN_PHASE.md`** — test scenarios for this phase extracted from TEST_PLAN.md:
- Risk areas scoped to this phase
- Backend integration tests for this phase's endpoints
- Frontend unit tests for this phase's components

Use `templates/TEST_PLAN_PHASE.template.md` as the structure reference.

**`memory.md`** — empty file with header: `# Phase {N} Memory: {Phase Description}`

**`ACCEPTANCE_CRITERIA.md`** — already written by the document-evaluator in Phase 2. Verify it exists. If missing, re-spawn the document-evaluator with instruction to generate the criteria for all phases without re-evaluating.

**Step 3 — Generate task files**

For every numbered step in the phase (each "Step N — Title" in PHASE_PLAN.md Backend and Frontend sections), create `{plan-folder}/phase-{N}/tasks/task-{NN}-{slug}.md` following `templates/TASK.template.md`.

**Task right-sizing:** smallest unit that carries its own test cycle and worth a fresh reviewer's gate. Fold atomically-coupled steps into one task (e.g., models.py + invites.py when they share a renamed relationship). Split a step into multiple tasks if it touches more than ~5 files or has independently verifiable sub-deliverables.

**For each task file, populate:**

- **Context**: one sentence — what this task delivers
- **Files table**: exact paths from the plan step. Mark Create / Modify / Test. No "similar to above."
- **Interfaces**: what this task consumes from prior tasks (name + signature) and what it produces for later tasks (name + shape). This is the most important section — it is how a fresh agent knows what already exists.
- **Critical Scope Flags**: verbatim from PLAN.md, scoped to this step
- **Implementation notes**: exact code snippets, SQL, field definitions, or prop signatures from the plan step — copy verbatim. Do not write "see PLAN.md."
- **Test cycle**: exact command and what must be true
- **Model hint**: `haiku` for single-file mechanical transcription; `sonnet` for multi-file, auth, new complex components, or view wiring
- **Reviewer checklist**: 3–5 structural checks verifiable from the diff

**Create `{plan-folder}/phase-{N}/tasks/TASK_INDEX.md`** — checkbox list the build orchestrator ticks off as tasks complete:

```markdown
# Phase {N} Task Index: {Phase Description}

- [ ] [task-01-{slug}.md](task-01-{slug}.md) — {Task Name}
- [ ] [task-02-{slug}.md](task-02-{slug}.md) — {Task Name}
...
```

---

## Phase 4 — Escalation (max rounds reached with unresolved Critical findings)

If 3 rounds completed with Critical findings still open:

1. Read review-history.md and extract all open `[ ]` Critical findings across all rounds
2. Present to the user:
   - What was resolved (how many Critical findings addressed)
   - The unresolved Critical findings verbatim: ID, description, section reference
   - A recommendation for each: what decision is needed to unblock it
3. Ask the user to resolve them and re-invoke `/plan-evaluate` — or to explicitly accept the risk and proceed

Do not materialize phase folders until all Critical findings are resolved or explicitly accepted.

---

## Phase 5 — Report

Tell the user:
- Final verdict and number of rounds taken
- Any Important or Minor findings noted but not blocking
- The phase folders created and their paths
- The next step: `/build-feature-from-plan {plan-folder}` to begin implementation
