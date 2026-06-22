# Product Discovery

Structured product discovery and feature enhancement workflows that produce complete PRDs — including UX design specs and technical solutions — ready for development planning.

## Skills

### `/product-discovery [issue-number ...]`

Runs a full structured discovery workflow: reads one or more GitHub issues, interviews the user about the problem, drafts a PRD, then consults the UX strategist and system architect agents in parallel to fill in design specs and technical solution sections.

**Output:** `{plans-dir}/{NNN}-{slug}/discovery/PRD.md`, `ux-notes.md`, `tech-notes.md`, and any ADRs. The completed PRD is the input to `/work-breakdown`.

### `/feature-enhancement [issue-number]`

Lightweight discovery for enhancements to existing features. Skips the full interview phase — goes straight to alignment, then UX and technical consultation. Best when the need is already understood and scope is narrow.

**Output:** Same structure as `/product-discovery`, scoped to the delta.

## Agents

Both skills spawn these agents during the consultation phase:

| Agent | Role |
|---|---|
| `product-discovery:ux-strategist` | Maps user journeys, identifies screens, describes layouts for screenshot generation |
| `product-discovery:system-architect` | Evaluates technical feasibility, proposes ranked approaches, produces ADRs |

If a specialized agent is not installed, the skill falls back to `general-purpose`.

## Usage

### CLAUDE.md Configuration

Add a **Product Discovery** section to your project's `CLAUDE.md` to set persistent path defaults — no arguments needed on every invocation.

```markdown
## Product Discovery
- plans-dir: plans/
- docs-dir: docs/
- infrastructure-doc: docs/INFRASTRUCTURE.md
```

| Key | Default | Used by |
|---|---|---|
| `plans-dir` | `plans/` | Both skills — where plan folders are created |
| `docs-dir` | `docs/` | Both skills — where documentation is read from |
| `infrastructure-doc` | `docs/INFRASTRUCTURE.md` | `system-architect` — hard deployment constraints |

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

### Infrastructure constraints

The `system-architect` agent reads an infrastructure constraints document before proposing any solution. This ensures recommendations stay within your actual deployment environment.

Create `docs/INFRASTRUCTURE.md` (or set `infrastructure-doc` in CLAUDE.md) with content like:

```markdown
# Infrastructure Constraints
- Deployment: self-hosted (home server)
- No cloud services — all components must be self-hostable
- Available: Docker, PostgreSQL, Redis, nginx
- Storage: local filesystem only
```

If this file does not exist, the system-architect will ask about deployment constraints before proposing any approach that depends on specific infrastructure.

## PDLC position

This plugin covers the **Discovery** phase:

```
GitHub issue / idea
       ↓
/product-discovery  or  /feature-enhancement
       ↓
PRD.md (with UX specs + technical solution)
       ↓
/work-breakdown  (development-workflow plugin)
       ↓
PLAN.md + TEST_PLAN.md
```
