---
name: plan-evaluate
description: >
  Runs the planner/evaluator loop on a PLAN.md. Spawns the document-evaluator agent
  with the plan-evaluation rubric; addresses Critical findings by spawning targeted
  planning agents; up to 3 rounds. On approval, generates ACCEPTANCE_CRITERIA.md per
  phase and materializes phase folders with PHASE_PLAN.md, TEST_PLAN_PHASE.md,
  phase-status.md, and memory.md. Use after work-breakdown to validate a plan, or
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

Read the following in parallel:
- `{plan-folder}/PLAN.md` — required; stop if missing
- `{plan-folder}/TEST_PLAN.md` — include if present
- `{plan-folder}/discovery/PRD.md` — required for coverage evaluation; stop if missing
- `{plan-folder}/review-history.md` — read if present (prior evaluation rounds)

Locate the rubric: `rubrics/plan-evaluation-rubric.md` in the development-tools plugin directory. This path is relative to the plugin's root.

If `review-history.md` does not exist, initialize it from `review-history.template.md` with the feature name and document type "PLAN.md".

---

## Phase 2 — Evaluation Loop (up to 3 rounds)

Track the current round number (starting at 1, incrementing after each Planner Response).

### Per round:

**Step A — Spawn document-evaluator**

Spawn `subagent_type: "development-tools:document-evaluator"` (or `"general-purpose"` if not available).

Provide:
- Full content of `{plan-folder}/PLAN.md`
- Full content of `{plan-folder}/TEST_PLAN.md` (if present)
- Full content of `{plan-folder}/discovery/PRD.md`
- The rubric file path
- The review history file path: `{plan-folder}/review-history.md`
- The current round number
- The feature directory path: `{plan-folder}` (for writing ACCEPTANCE_CRITERIA.md files when approved)
- Instruction: "Evaluate the PLAN.md and TEST_PLAN.md against the rubric and the PRD. Append your findings to review-history.md in the format described in your instructions. If your verdict is Approved or Approved with concerns, also generate ACCEPTANCE_CRITERIA.md for each implementation phase as instructed in the rubric."

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
| PRD coverage gaps (P0/P1 requirements absent) | Address inline by reading the PRD and adding the missing steps to PLAN.md |

For each agent needed: spawn it with the relevant finding(s). Group findings by agent so each agent is only spawned once per round.

Brief each agent with:
- The full current content of `{plan-folder}/PLAN.md`
- The specific Critical finding IDs and descriptions (verbatim from review-history.md) this agent must address
- Instruction to return the updated content for the relevant sections of PLAN.md only — not a full rewrite
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

**Step 2 — For each phase, create the folder structure**

Create `{plan-folder}/phase-{N}/` and write the following files:

**`phase-{N}/PHASE_PLAN.md`**
Extract from PLAN.md:
- The Context section (or a phase-specific context paragraph)
- The Backend section steps for this phase
- The Frontend section steps for this phase
- The Critical Scope Flags relevant to this phase
- The Verification steps for this phase

Use `PHASE_PLAN.template.md` as the structure reference.

**`phase-{N}/TEST_PLAN_PHASE.md`**
If TEST_PLAN.md exists, extract the test scenarios relevant to this phase:
- Risk areas scoped to this phase's work
- Backend integration tests (Tier 2) for this phase's endpoints
- Frontend unit tests (Tier 1) for this phase's components

Use `TEST_PLAN_PHASE.template.md` as the structure reference.

**`phase-{N}/phase-status.md`**
Generate a task checklist from the phase's steps. Use `phase-status.template.md` as the structure. Leave all checkboxes unchecked.

**`phase-{N}/memory.md`**
Create an empty file with a single header: `# Phase {N} Memory: {Phase Description}`

**`phase-{N}/ACCEPTANCE_CRITERIA.md`**
This file was already written by the document-evaluator in Phase 2. If it is missing, re-spawn the document-evaluator with instruction to generate the criteria for all phases without re-evaluating (verdict is already Approved).

---

## Phase 4 — Escalation (max rounds reached with unresolved Critical findings)

If 3 rounds completed with Critical findings still open:

1. Read review-history.md and extract all open `[ ]` Critical findings across all rounds
2. Present to the user:
   - A summary of what was resolved (how many Critical findings addressed)
   - The unresolved Critical findings verbatim: ID, description, section reference
   - A recommendation for each: what decision is needed to unblock it
3. Ask the user to resolve them, then re-invoke `/plan-evaluate` — or to explicitly accept the risk and proceed

Do not materialize phase folders until all Critical findings are resolved or explicitly accepted by the user.

---

## Phase 5 — Report

Tell the user:
- Final verdict and number of rounds taken
- Any Important or Minor findings noted but not blocking
- The phase folders created and their paths
- The next step: `/build-feature-from-plan {plan-folder}` to begin implementation
