# Development Workflow

Skills and agents supporting the full development phase of the PDLC: translating a completed PRD into an implementation plan, running planner/evaluator loops to validate the plan and code, and orchestrating implementation by specialized agents.

## PDLC position

```
PRD.md (from product-discovery plugin)
       ↓
/work-breakdown  →  PLAN.md + TEST_PLAN.md
                 →  plan/evaluator loop (up to 3 rounds)
                 →  phase-N/ folders (PHASE_PLAN.md, TEST_PLAN_PHASE.md,
                                       ACCEPTANCE_CRITERIA.md, phase-status.md)
       ↓
/build-feature-from-plan
       ↓  (per phase)
  backend agent (writes tests inline)  →  code-evaluate loop
  frontend agent (writes tests inline) →  code-evaluate loop
  user validation gate (Tier 3 + Tier 4 only)
       ↓
code + tests + memory.md
```

## Skills

| Skill | Description |
|---|---|
| `/work-breakdown <path-to-prd> [screenshots-dir]` | Translates a PRD into PLAN.md and TEST_PLAN.md, runs a plan evaluation loop, and materializes phase folders. |
| `/plan-evaluate <plan-folder>` | Standalone plan evaluation loop. Re-evaluates an existing PLAN.md; addresses Critical findings via planning agents; generates ACCEPTANCE_CRITERIA.md and phase folders on approval. |
| `/code-evaluate <phase-folder> [backend\|frontend]` | Standalone code evaluation loop. Runs Tier 1 commands, spawns the code-evaluator for Tier 2 inspection, surfaces Tier 3 items for human review. |
| `/qa-architect <plan-folder>` | Generates or regenerates TEST_PLAN.md for an existing PLAN.md. |
| `/build-feature-from-plan <plan-folder> [phase\|all]` | Orchestrates building from phase folders: backend agent, code-evaluate loop, frontend agent, code-evaluate loop, user validation gate (Tier 3+4), security scan. |

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
  phase-1/
    PHASE_PLAN.md             ← extracted vertical slice for this phase
    TEST_PLAN_PHASE.md        ← test scenarios scoped to this phase
    ACCEPTANCE_CRITERIA.md    ← tiered acceptance criteria (Tier 1–4)
    phase-status.md           ← task checklist
    memory.md                 ← implementation notes (written at close)
    review-history.md         ← code evaluator dialogue for this phase
  phase-2/
    ...
```

## Acceptance criteria tiers

Each `ACCEPTANCE_CRITERIA.md` file organizes checks into four tiers:

| Tier | Who checks | What |
|---|---|---|
| 1 — Deterministic | Code-evaluator runs commands | Test suites, lint, type check, build, migration |
| 2 — Agent-verifiable | Code-evaluator reads diff/code | Endpoints exist, components created, auth applied, schema matches |
| 3 — Visual/UX | Human or browser | Layout, states, interactions, visual design |
| 4 — Product acceptance | User | Does it solve the problem; tone; edge cases not captured above |

The user validation gate surfaces only Tier 3 and Tier 4. Tiers 1 and 2 are handled by the code-evaluate loop automatically.

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

To use a different backend or frontend agent (e.g., a Node.js API developer instead of Python), set the config keys:

```markdown
## Development Workflow
- backend-agent: my-plugin:node-api-developer
- frontend-agent: my-plugin:react-developer
```

The build skill will use whichever agent is configured, falling back to `general-purpose` if not available.

### Rubrics

Evaluation rubrics are in `rubrics/`:

| File | Used by |
|---|---|
| `rubrics/plan-evaluation-rubric.md` | `document-evaluator` when evaluating PLAN.md — defines evaluation dimensions and how to generate ACCEPTANCE_CRITERIA.md |

To customize evaluation criteria for a project, copy the rubric into your project's `docs/` folder and set a `plan-evaluation-rubric` key in the `## Development Workflow` section of your project's `CLAUDE.md`.
