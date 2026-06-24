---
name: work-breakdown
description: >
  Generates a PLAN.md and TEST_PLAN.md work breakdown for a feature that has completed
  product discovery. Reads the PRD, tech-notes, and ux-notes from the discovery folder;
  spawns tech-lead, ui-architect, qa-architect, and codebase Explore agents in parallel;
  asks any blocking architecture questions; then writes PLAN.md and TEST_PLAN.md to the
  feature's plan folder. Use after discovery is complete and the PRD exists. Trigger when
  the user says "generate the work breakdown", "create a plan", "write PLAN.md", or
  invokes /work-breakdown with a path to a PRD.
argument-hint: "<path-to-prd> [screenshots-dir]"
---

# Work Breakdown Workflow

You are generating an implementation plan (PLAN.md) and test plan (TEST_PLAN.md) for a feature that has completed product discovery. Your job is to translate the PRD, UX notes, and tech notes into a concrete, developer-ready work breakdown.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Development Workflow` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plan output (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)

Use these values wherever `{plans-dir}` and `{docs-dir}` appear.

## Arguments

- `<path-to-prd>` (required) — path to the PRD.md (e.g., `plans/010-share-with-friends/discovery/PRD.md`)
- `[screenshots-dir]` (optional) — path to screenshots directory for the ui-architect agent

---

This workflow has five phases. Complete them in order.

---

## Phase 1 — Read the Source Documents

From the PRD path, derive:
- **Discovery dir**: the directory containing the PRD (e.g., `plans/010-share-with-friends/discovery/`)
- **Feature dir**: the parent of the discovery dir (e.g., `plans/010-share-with-friends/`)
- **Plan output path**: `{feature-dir}/PLAN.md`
- **Test plan output path**: `{feature-dir}/TEST_PLAN.md`

Read all three documents in parallel:
- `{discovery-dir}/PRD.md`
- `{discovery-dir}/tech-notes.md` (if it exists)
- `{discovery-dir}/ux-notes.md` (if it exists)

From these documents, extract:
- The feature's problem statement and goals
- The P0/P1/P2 feature requirements
- The recommended technical approach (from tech-notes)
- The page inventory and component changes (from UX notes and PRD Design Specs section)
- All critical scope flags and risk callouts
- The backend files that will change (from tech-notes)
- The frontend components and views that will change (from UX notes and PRD Design Specs section)
- Any resolved open questions that affect implementation

---

## Phase 2 — Parallel Agent Consultation

Spawn the following four agents **in parallel in a single message** — they work independently.

### Agent A: tech-lead

Spawn as `subagent_type: "development-workflow:tech-lead"` (or `"general-purpose"` if not available).

The tech-lead must:
1. Read `CLAUDE.md` and any referenced patterns docs before forming opinions
2. Review the PRD's Technical Solution section and tech-notes
3. Produce the implementation plan for the Backend section of PLAN.md:
   - Steps in dependency order (migrations before models, models before schemas, schemas before routers)
   - Every new or modified file with a one-line description of what changes
   - New endpoints table (method, path, auth requirement, description, edge cases)
   - Critical scope flags: auth checks, migration safety, env vars, potential regressions
4. Produce an ADR for any significant architectural decision not already captured in the discovery ADRs
5. Save detailed reasoning to `{feature-dir}/tech-planning-notes.md` if needed
6. Return: the complete Backend section for PLAN.md + any new ADR paths

Provide the agent with:
- Full text of the PRD
- Full text of tech-notes.md
- The feature directory path for saving outputs
- Instruction to focus only on the backend implementation plan, not frontend

### Agent B: ui-architect

Spawn as `subagent_type: "development-workflow:ui-architect"` (or `"general-purpose"` if not available).

The ui-architect must:
1. Read `CLAUDE.md` and any referenced frontend patterns/design docs
2. Read the existing shared components most relevant to this work
3. If screenshots are provided, examine them
4. For every UI change in the PRD, enumerate:
   - Which existing components to reuse (with exact props)
   - What new components need to be built (name, location, props, emits)
   - What prop additions or removals are needed on modified components
   - Any dead code or duplication to clean up as part of this work
   - Any gaps in the UX description that will block implementation
5. Return: a structured breakdown organized by component/screen for the Frontend section of PLAN.md

Provide the agent with:
- Full text of the PRD Design Specs section
- Full text of ux-notes.md
- The screenshots directory path if provided
- Instruction to return a structured breakdown (not code) organized by component/screen

### Agent C: qa-architect

Spawn as `subagent_type: "development-workflow:qa-architect"` (or `"general-purpose"` if not available).

The qa-architect must:
1. Read `CLAUDE.md` and any test strategy doc
2. Read the PRD to understand feature requirements and critical paths
3. Produce a complete TEST_PLAN.md for the feature, written to `{feature-dir}/TEST_PLAN.md`
4. Return: a summary (test counts by tier, top risk areas, any notable gaps)

Provide the agent with:
- Full text of the PRD
- The feature plan path: `{feature-dir}`
- The `plans-dir` and `docs-dir` config values
- Instruction to write TEST_PLAN.md and return a summary

### Agent D: Codebase Explore

Spawn as `subagent_type: "Explore"` with search breadth `"thorough"`.

Ask the agent to read the backend files that will change based on the tech-notes:
- Exact function signatures for any helper being replaced or extracted
- Current ORM model relationships relevant to new tables
- Current schema field lists for responses being extended
- The migration numbering sequence to confirm the next migration number
- Test fixture helpers (from the test config) to understand the test pattern

Provide the agent with:
- The list of specific files and line ranges to read (from tech-notes "Files That Will Change" section)
- The critical functions to focus on (name them explicitly)

---

## Phase 3 — Ask Clarifying Questions

After all four agents return, review their findings for decision points that would significantly change the implementation. Ask the user about genuine architectural choices — not cosmetic preferences.

Good reasons to ask:
- Two valid approaches exist that produce meaningfully different structures (use `AskUserQuestion` with visual previews)
- A required dependency (route, component, endpoint) does not exist and building it is significant scope
- An agent found an ambiguity in the PRD that has not been resolved

Do not ask about:
- Styling preferences that follow directly from design conventions
- Implementation details clear from the tech-notes
- Anything the PRD already resolved

Ask at most 2 questions before proceeding.

---

## Phase 4 — Write PLAN.md

Write the plan to `{feature-dir}/PLAN.md`. The plan must be:
- Scannable at a glance (tables for file changes, clear step headings)
- Detailed enough to execute without re-reading the PRD
- Broken into **Backend** and **Frontend** sections
- Faithful to the language used in the PRD — if the PRD uses a specific term for an entity, the plan uses that same term everywhere

### Plan structure

**Context section**: One paragraph explaining the user problem this solves and the intended outcome. Then list the delivery phases if applicable.

**Backend section**: Numbered steps in dependency order:
1. Environment / config changes (new env vars, config files)
2. Database migration (with exact table definitions, backfill logic)
3. ORM model additions and relationship changes
4. Schema/type additions and modifications
5. Shared dependency helpers (extracted or new)
6. Router changes (which routers, what changes, auth requirements)
7. New endpoints (table: method, path, auth requirement, description, edge cases)
8. Any critical bug fixes required as part of this work

**Frontend section**: Numbered steps in dependency order:
1. Design system additions (new tokens if any)
2. Dead code cleanup (duplicate components to delete)
3. New shared components (name, location, props, emits, layout description)
4. Modified shared components (what changes, what props are added/removed)
5. New feature-local components (name, location, props, layout, states)
6. New views / routes (route config, layout pattern, all interaction states)
7. Modified views (specific additions — describe the delta, not the full view)
8. API method additions
9. Store type additions
10. Router guard changes (if any)

### Phase splitting rules

The structure of phases determines what agents can verify, what the user can validate, and how much context each implementation agent needs. Apply these rules when deciding what goes in each phase:

**Backend-first.** If the feature has both backend and frontend work, backend phases must come first. A frontend phase that calls new endpoints cannot be tested until those endpoints exist.

**One frontend phase = one visual slice.** Each frontend phase must include everything needed to visually verify it end-to-end. This means: the shared components that are new AND the views that wire them together, in the same phase. Never split "build shared components" and "wire them into views" across separate phases — the components cannot be meaningfully reviewed in isolation. A Tier 3 checklist of "check how the component looks" is unanswerable if the component isn't rendered anywhere.

**Corollary: no stub-only phases.** If a set of frontend steps produces components that aren't mounted in any view until a later phase, two options:
  - Move the components to the same phase as their consuming views, OR
  - Write those steps' Tier 3 as "verified by unit tests only; visual check deferred to Phase X" — do not ask the user to visually verify unwired components

**Phase size target.** Aim for phases that touch ~15–20 files at most. If a single phase would touch significantly more, split it along a natural seam (e.g., by feature area or by screen).

**Recommended structure for mixed backend/frontend features:**
1. Phase 1: All backend changes + backend tests (fully testable via pytest/jest without UI)
2. Phase 2: All frontend changes, or a first frontend slice — include shared components AND the feature views that use them so the full flow is visually verifiable end-to-end
3. Phase 3+: Additional frontend slices (if too large for one phase), each equally complete and verifiable

**Deferred phases.** Phase 3 (or later) for out-of-scope work: one brief section listing what it delivers, why it's deferred, and what must exist before it can build.

---

**Critical Scope Flags**: A numbered list of the highest-risk items from all agents — security holes, silent bugs, missing dependencies, env vars that must be set in production.

**Critical Files table**: Every file that changes → one-line description. Mark "New" or "Delete" for files being created or removed.

**Verification section**: Numbered steps for validating end-to-end:
1. Migration command — confirms migration applies cleanly
2. Backend test suite command
3. Frontend test suite command
4. Manual test walkthrough — golden path + key failure states
5. Build command — production image builds successfully

---

## Phase 5 — Plan Evaluation Loop

After writing PLAN.md and TEST_PLAN.md, run the planner/evaluator loop to validate the plan before materializing phase folders. This loop runs inline — do not ask the user to invoke `/plan-evaluate` separately.

Locate the rubric: `rubrics/plan-evaluation-rubric.md` in the development-tools plugin directory.

Initialize `{feature-dir}/review-history.md` from `templates/review-history.template.md` with the feature name and document type "PLAN.md".

### Per round (up to 3):

**Step A — Spawn document-evaluator**

Spawn `subagent_type: "development-tools:document-evaluator"` (or `"general-purpose"` if not available).

Pass file paths — not content. The evaluator reads what it needs.

```
Files to read:
- PLAN.md: {feature-dir}/PLAN.md
- TEST_PLAN.md: {feature-dir}/TEST_PLAN.md (present: {yes|no})
- PRD.md: {discovery-dir}/PRD.md
- Rubric: {rubric-path}
- Review history: {feature-dir}/review-history.md
- Feature directory: {feature-dir}
- Round: {N}

Instruction: Evaluate PLAN.md and TEST_PLAN.md against the rubric and the PRD.
Append your findings to review-history.md. If your verdict is Approved or Approved
with concerns, also generate ACCEPTANCE_CRITERIA.md for each implementation phase.
```

Wait for the evaluator to return.

**Step B — Check verdict**

- **Approved** or **Approved with concerns**: exit the loop. Go to Phase 6.
- **Needs revision** and round < 3: continue to Step C.
- **Needs revision** and round = 3: go to Phase 7 (escalation).

**Step C — Address Critical findings**

Read `{feature-dir}/review-history.md` and extract the Critical findings from this round.

Determine the responsible planning agent per finding type:

| Finding type | Agent to spawn |
|---|---|
| Backend steps, endpoints, migrations, models, schemas, auth | `development-tools:tech-lead` |
| Frontend components, views, props, routes | `development-tools:ui-architect` |
| Test strategy gaps, missing scenarios | `development-tools:qa-architect` |
| PRD coverage gaps (P0/P1 requirements absent) | Address inline: read the PRD and add the missing steps to PLAN.md |

Group findings by agent. Brief each agent with:
- Path to PLAN.md: `{feature-dir}/PLAN.md` — the agent reads it
- The specific Critical finding IDs and descriptions (verbatim from review-history.md) it must address — these are short, paste inline
- Instruction to return only the revised content for its sections — not a full rewrite
- Instruction not to change sections unrelated to its findings

After agents return, update PLAN.md with the revised sections.

**Step D — Write planner response**

Append to `{feature-dir}/review-history.md`:

```
### Planner Response — {YYYY-MM-DD}

#### Addressed
- [x] #{id} — {what was changed, with section/step reference}

#### Not Addressed
- [ ] #{id} — {reason}
```

Increment round and return to Step A.

---

## Phase 6 — Materialize Phase Folders

After the plan is approved, create a self-contained folder for each implementation phase with task-level breakdowns so each implementing agent gets precisely what it needs.

**Step 1 — Identify phases**

Parse PLAN.md. Phases are sections named "Phase 1", "Phase 2", etc. A plan with no explicit phase sections is treated as a single Phase 1.

**Step 1.5 — Apply phase quality check**

Before materializing folders, verify each phase passes the splitting rules from Phase 4:

1. **Visual verifiability check**: For each frontend phase, confirm it includes at least one mounted view or route (not just shared components in isolation). If a phase only produces shared components with no consuming view, flag it: either merge the components into the next phase that wires them, or annotate those steps' Tier 3 as "unit-test verified only; visual check deferred to Phase {next}."

2. **Backend-before-frontend check**: No frontend phase appears before the backend phase it depends on.

3. **Size check**: Count tasks in scope per phase. If any phase has more than ~15 distinct tasks, recommend splitting it to the user before materializing.

If any check fails, revise the phase breakdown in PLAN.md before continuing. Do not materialize folders for a phase that will produce unverifiable Tier 3 items.

**Step 2 — For each phase, create `{feature-dir}/phase-{N}/`** with these files:

**`PHASE_PLAN.md`** — extracted vertical slice for human reference. Include:
- Context paragraph for this phase
- Backend steps for this phase (in dependency order)
- Frontend steps for this phase (in dependency order)
- Critical Scope Flags relevant to this phase
- Verification steps for this phase

Use `templates/PHASE_PLAN.template.md` as the structural reference.

**`TEST_PLAN_PHASE.md`** — test scenarios for this phase extracted from TEST_PLAN.md. Include risk areas, backend integration tests, and frontend unit tests scoped to this phase's work. Use `templates/TEST_PLAN_PHASE.template.md` as the structural reference.

**`memory.md`** — empty file with header: `# Phase {N} Memory: {Phase Description}`

**`ACCEPTANCE_CRITERIA.md`** — already written by the document-evaluator in Phase 5. Verify it exists. If missing, re-spawn the document-evaluator with instruction to generate the criteria without re-evaluating.

**Step 3 — Generate task files for each phase**

For every numbered step in the phase (each "Step N — Title" in PHASE_PLAN.md Backend and Frontend sections), create a corresponding task file at `{feature-dir}/phase-{N}/tasks/task-{NN}-{slug}.md` following `templates/TASK.template.md`.

**Task right-sizing:** A task is the smallest unit that carries its own test cycle and is worth a fresh reviewer's gate. Fold tightly-coupled steps into one task if a reviewer could not meaningfully approve one while rejecting its neighbor (e.g., models.py + invites.py when they share a renamed relationship that must ship atomically). Split a step into multiple tasks if it touches more than ~5 files or has independently verifiable sub-deliverables.

**For each task file, populate:**

- **Context**: one sentence from the plan step — what this task delivers and why
- **Files table**: copy exact paths from the plan step's description. Mark Create/Modify/Test. Be exact — no "similar to above."
- **Interfaces section**: this is the most critical part. Trace the dependency chain:
  - *Consumes*: what does this task need from earlier tasks in this phase or prior phases? Name the function, type, endpoint, or field and the task that produces it. If the consuming agent won't see it in their files, describe the signature.
  - *Produces*: what does this task make available for later tasks? Name it and give its shape (function signature, TypeScript type, API response schema, etc.).
- **Critical Scope Flags**: copy any flags from PLAN.md that apply specifically to this step
- **Implementation notes**: copy exact code snippets, SQL, field definitions, or prop signatures verbatim from the plan step. Do not write "see PLAN.md."
- **Test cycle**: the exact command(s) to run and what must be true. For backend tasks, name the specific test functions if they're in TEST_PLAN_PHASE.md. For frontend tasks, name the spec file.
- **Model hint**: `haiku` for mechanical transcription (single file, complete spec provided, prop addition only); `sonnet` for multi-file coordination, auth logic, new complex components, or view wiring
- **Reviewer checklist**: 3–5 structural checks the reviewer can verify from the diff

**Also create `{feature-dir}/phase-{N}/tasks/TASK_INDEX.md`** — a checkbox list of all tasks in execution order. The build orchestrator updates these boxes as tasks complete; this is the human-visible progress view.

```markdown
# Phase {N} Task Index: {Phase Description}

- [ ] [task-01-{slug}.md](task-01-{slug}.md) — {Task Name}
- [ ] [task-02-{slug}.md](task-02-{slug}.md) — {Task Name}
...
```

Tasks execute in the listed order. If a task has no Interfaces dependency on the immediately prior task, note the earliest it can run — but implement in order by default.

---

## Phase 7 — Escalation (max rounds reached)

If 3 evaluation rounds completed with Critical findings still open:

1. Extract all open `[ ]` Critical findings from `{feature-dir}/review-history.md`
2. Present to the user:
   - What was resolved vs. what remains open
   - Unresolved Critical findings verbatim with section references
   - A recommendation for each: what decision is needed to unblock it
3. Ask the user to resolve them and then re-invoke `/plan-evaluate {feature-dir}`, or to accept the risk and proceed

Do not materialize phase folders until all Critical findings are resolved or explicitly accepted.

---

## Phase 8 — Confirm and Close Out

Tell the user:
- The paths to PLAN.md and TEST_PLAN.md
- The evaluation result (approved in N rounds, any noted concerns)
- The phase folders created and their paths
- A one-paragraph summary: how many phases, backend steps per phase, frontend steps per phase, what's new vs. modified
- Any open questions raised but not resolved (should be rare after the evaluation loop)
- Next step: `/build-feature-from-plan {feature-dir}` to begin implementation
