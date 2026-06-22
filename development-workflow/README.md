# Development Workflow

Skills and agents supporting the full development phase of the PDLC: translating a completed PRD into an implementation plan, test plan, and working code.

## PDLC position

```
PRD.md (from product-discovery plugin)
       ↓
/work-breakdown  →  PLAN.md + TEST_PLAN.md
       ↓
/build-feature-from-plan  →  code + tests
```

## Skills

| Skill | Description |
|---|---|
| `/work-breakdown <path-to-prd> [screenshots-dir]` | Translates a completed PRD into PLAN.md and TEST_PLAN.md. Spawns tech-lead, ui-architect, qa-architect, and codebase Explore agents in parallel. |
| `/qa-architect <plan-folder>` | Generates or regenerates TEST_PLAN.md for an existing PLAN.md. Standalone version of the qa-architect agent. |
| `/build-feature-from-plan <plan-folder> [phase\|all]` | Orchestrates building from a PLAN.md: delegates backend and frontend implementation to specialized agents, runs lint and tests after each phase, pauses for user validation, then runs a security scan. |

## Agents

| Agent | Tier | Used by |
|---|---|---|
| `development-workflow:tech-lead` | Planning | `/work-breakdown` |
| `development-workflow:ui-architect` | Planning | `/work-breakdown` |
| `development-workflow:qa-architect` | Planning | `/work-breakdown`, `/qa-architect` |
| `development-workflow:python-api-developer` | Build | `/build-feature-from-plan` |
| `development-workflow:vue-developer` | Build | `/build-feature-from-plan` |

If a specialized agent is not installed, skills fall back to `general-purpose`.

## Usage

### CLAUDE.md Configuration

Add a **Development Workflow** section to your project's `CLAUDE.md` to set persistent defaults — no arguments needed on every invocation.

```markdown
## Development Workflow
- plans-dir: plans/
- docs-dir: docs/
- backend-agent: development-workflow:python-api-developer
- frontend-agent: development-workflow:vue-developer
```

| Key | Default | Used by |
|---|---|---|
| `plans-dir` | `plans/` | All skills — where plan folders are located |
| `docs-dir` | `docs/` | All skills — where documentation is read from |
| `backend-agent` | `development-workflow:python-api-developer` | `/build-feature-from-plan` — swap for a different stack |
| `frontend-agent` | `development-workflow:vue-developer` | `/build-feature-from-plan` — swap for a different stack |

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

### Swapping implementation agents

To use a different backend or frontend agent (e.g., a Node.js API developer instead of Python), set the config keys:

```markdown
## Development Workflow
- backend-agent: my-plugin:node-api-developer
- frontend-agent: my-plugin:react-developer
```

The build skill will use whichever agent is configured, falling back to `general-purpose` if not available.
