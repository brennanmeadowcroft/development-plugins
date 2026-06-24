# Development Workflow

Skills and agents supporting the full development phase of the PDLC: translating a completed PRD into an implementation plan, running planner/evaluator loops to validate the plan and code, and orchestrating implementation by specialized agents.

## PDLC position

```
PRD.md (from product-discovery plugin)
       ↓
/work-breakdown  →  PLAN.md + TEST_PLAN.md
                 →  plan/evaluator loop (up to 3 rounds)
                 →  phase-N/ folders
                       PHASE_PLAN.md, TEST_PLAN_PHASE.md,
                       ACCEPTANCE_CRITERIA.md, memory.md,
                       tasks/TASK_INDEX.md, tasks/task-NN-*.md
       ↓
/build-feature-from-plan
       ↓  (per task, in order)
  implementer agent  →  reviewer gate  →  fix loop (up to 3 rounds)
       ↓  (per phase, when Tier 3 items exist)
  user validation gate (Tier 3 + Tier 4)
       ↓
  security scan
       ↓
code + tests + .build-ledger.md + memory.md
```

## Skills

| Skill | Description |
|---|---|
| `/work-breakdown <path-to-prd> [screenshots-dir]` | Translates a PRD into PLAN.md and TEST_PLAN.md, runs a plan evaluation loop, and materializes phase folders with task files. |
| `/plan-evaluate <plan-folder>` | Standalone plan evaluation loop. Re-evaluates an existing PLAN.md; addresses Critical findings via planning agents; generates ACCEPTANCE_CRITERIA.md, phase folders, and task files on approval. |
| `/code-evaluate <phase-folder> [backend\|frontend]` | Standalone code evaluation loop. Runs Tier 1 commands, spawns the code-evaluator for Tier 2 inspection, surfaces Tier 3 items for human review. |
| `/qa-architect <plan-folder>` | Generates or regenerates TEST_PLAN.md for an existing PLAN.md. |
| `/build-feature-from-plan <plan-folder> [phase\|all]` | Orchestrates building from task files: one implementer agent per task, one reviewer gate per task, user validation gate when a phase has Tier 3 items, security scan at the end. |

## Agents

| Agent | Tier | Used by |
|---|---|---|
| `development-tools:tech-lead` | Planning | `/work-breakdown`, `/plan-evaluate` |
| `development-tools:ui-architect` | Planning | `/work-breakdown`, `/plan-evaluate` |
| `development-tools:qa-architect` | Planning | `/work-breakdown`, `/qa-architect` |
| `development-tools:document-evaluator` | Evaluation | `/work-breakdown`, `/plan-evaluate` |
| `development-tools:code-evaluator` | Evaluation | `/build-feature-from-plan`, `/code-evaluate` |
| `development-tools:python-api-developer` | Build | `/build-feature-from-plan` |
| `development-tools:vue-developer` | Build | `/build-feature-from-plan` |

If a specialized agent is not installed, skills fall back to `general-purpose`.

## Phase folder structure

After `/work-breakdown` or `/plan-evaluate` completes, each implementation phase has a self-contained folder:

```
plans/010-{slug}/
  PLAN.md                     ← full plan with all phases
  TEST_PLAN.md                ← full test strategy
  review-history.md           ← plan evaluator dialogue
  .build-ledger.md            ← durable task progress (written during build)
  phase-1/
    PHASE_PLAN.md             ← extracted vertical slice for this phase (human reference)
    TEST_PLAN_PHASE.md        ← test scenarios scoped to this phase
    ACCEPTANCE_CRITERIA.md    ← tiered acceptance criteria (Tier 1–4) for user gate
    memory.md                 ← implementation notes (written at close)
    review-history.md         ← code evaluator dialogue for this phase
    tasks/
      TASK_INDEX.md           ← checkbox list; ticked off as each task completes
      task-01-{slug}.md       ← self-contained brief: files, interfaces, test cycle
      task-01-{slug}-result.md   ← implementer writes this (commit SHA, tests, deviations)
      task-01-{slug}-review.md   ← reviewer writes this (verdict, findings)
      task-02-{slug}.md
      ...
  phase-2/
    ...
```

## Task files

Each `task-NN-{slug}.md` is the complete brief for a single implementer agent. It contains:

- **Files**: exact paths to create, modify, and test — no "similar to above"
- **Interfaces**: what the task consumes from prior tasks (exact names + signatures) and what it produces for later tasks — this is how a fresh agent knows what already exists without reading the whole codebase
- **Implementation notes**: exact code snippets, SQL, field definitions, or prop signatures copied verbatim from the plan
- **Test cycle**: exact command and expected outcome
- **Model hint**: `haiku` for mechanical single-file transcription; `sonnet` for multi-file coordination, auth logic, or complex components
- **Reviewer checklist**: 3–5 structural checks verifiable from the diff

The `TASK_INDEX.md` lists all tasks with checkboxes. The build orchestrator ticks them off as each task completes. The `.build-ledger.md` is the machine-readable counterpart — it survives context compaction and is the source of truth for resumption after an interrupted build.

## Acceptance criteria tiers

Each `ACCEPTANCE_CRITERIA.md` organizes checks into four tiers:

| Tier | Who checks | What |
|---|---|---|
| 1 — Deterministic | Code-evaluator runs commands | Test suites, lint, type check, build, migration |
| 2 — Agent-verifiable | Code-evaluator reads diff/code | Endpoints exist, components created, auth applied, schema matches |
| 3 — Visual/UX | Human or browser | Layout, states, interactions, visual design |
| 4 — Product acceptance | User | Does it solve the problem; tone; edge cases not captured above |

The user validation gate surfaces Tier 3 and Tier 4 — and only for phases that have Tier 3 items. Backend-only phases (all pytest) continue automatically without pausing.

## Build context management

`/build-feature-from-plan` is designed to run long features without exhausting the orchestrator's context:

- **One agent per task**: each implementer and reviewer starts with a fresh context scoped to a single task file
- **File handoffs**: large artifacts (diffs, Tier 1 output, reviewer findings) move as file paths — not pasted inline into dispatch prompts
- **Durable ledger**: `.build-ledger.md` tracks completed tasks so the build can resume after context compaction or interruption
- **Dynamic model selection**: the `haiku`/`sonnet` model hint in each task file controls which model is used per task — mechanical work uses cheaper/faster models; integration work uses stronger models

Before starting a long build, run `/fewer-permission-prompts` to pre-allowlist common git and test commands and reduce interruptions.

## Phase quality rules

When `/work-breakdown` or `/plan-evaluate` materializes phase folders, it enforces:

1. **Backend-first**: backend phases always precede the frontend phases that depend on them
2. **Visual verifiability**: each frontend phase includes at least one mounted view, not just unwired shared components. Components that aren't wired into a view in the same phase either move to the phase that wires them, or their Tier 3 is annotated "unit-test verified only; visual check deferred to Phase N"
3. **Size**: phases targeting more than ~15 tasks are flagged for splitting

## Usage

### CLAUDE.md Configuration

Add a **Development Workflow** section to your project's `CLAUDE.md` to set persistent defaults — no arguments needed on every invocation.

```markdown
## Development Workflow
- plans-dir: plans/
- docs-dir: docs/
- backend-agent: development-tools:python-api-developer
- frontend-agent: development-tools:vue-developer
```

| Key | Default | Used by |
|---|---|---|
| `plans-dir` | `plans/` | All skills — where plan folders are located |
| `docs-dir` | `docs/` | All skills — where documentation is read from |
| `backend-agent` | `development-tools:python-api-developer` | `/build-feature-from-plan` — swap for a different stack |
| `frontend-agent` | `development-tools:vue-developer` | `/build-feature-from-plan` — swap for a different stack |

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

### Swapping implementation agents

To use a different backend or frontend agent (e.g., a Node.js API developer instead of Python):

```markdown
## Development Workflow
- backend-agent: my-plugin:node-api-developer
- frontend-agent: my-plugin:react-developer
```

The build skill falls back to `general-purpose` if a specified agent is not available.

### Rubrics

Evaluation rubrics are in `rubrics/`:

| File | Used by |
|---|---|
| `rubrics/plan-evaluation-rubric.md` | `document-evaluator` when evaluating PLAN.md — defines evaluation dimensions and how to generate ACCEPTANCE_CRITERIA.md |

To customize evaluation criteria for a project, copy the rubric into your project's `docs/` folder and set a `plan-evaluation-rubric` key in the `## Development Workflow` section of your project's `CLAUDE.md`.

### Templates

Output file templates live in `templates/`:

| Template | Generated by | Output |
|---|---|---|
| `templates/PHASE_PLAN.template.md` | `/work-breakdown`, `/plan-evaluate` | `phase-N/PHASE_PLAN.md` |
| `templates/TEST_PLAN_PHASE.template.md` | `/work-breakdown`, `/plan-evaluate` | `phase-N/TEST_PLAN_PHASE.md` |
| `templates/ACCEPTANCE_CRITERIA.template.md` | `document-evaluator` agent | `phase-N/ACCEPTANCE_CRITERIA.md` |
| `templates/TASK.template.md` | `/work-breakdown`, `/plan-evaluate` | `phase-N/tasks/task-NN-*.md` |
| `templates/review-history.template.md` | `/plan-evaluate`, `/code-evaluate` | `review-history.md` |
