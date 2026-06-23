# Plan Evaluation Rubric

This rubric guides the document-evaluator agent when reviewing a PLAN.md and associated TEST_PLAN.md. Apply it to identify gaps before implementation begins.

## What to read before evaluating

Read all of these before forming any opinion:
- The PLAN.md being evaluated (full content)
- The PRD.md from the discovery folder (to verify coverage)
- TEST_PLAN.md if it exists (to evaluate test strategy completeness)
- Any review-history.md from prior rounds (to verify claimed fixes before surfacing new findings)

---

## Evaluation dimensions

### 1. PRD coverage

Check PLAN.md against the PRD's Feature Requirements section:

- **P0 requirements**: Every P0 feature must have explicit implementation steps. Flag any P0 feature with no corresponding plan step.
- **P1 requirements**: Must be either implemented or explicitly deferred with rationale. Flag P1 features that are silently absent.
- **P2 requirements**: Should be acknowledged as out of scope if not included. Silent absence is not a finding; acknowledged absence is.
- **Scope creep**: Flag plan steps that have no traceable PRD requirement, unless they are clearly prerequisite infrastructure.

| Severity | Condition |
|---|---|
| Critical | Any P0 requirement is absent without explanation |
| Important | Any P1 requirement is absent without explanation |
| Minor | P2 scope is not acknowledged |

---

### 2. Phase decomposition

Each implementation phase must be a self-contained vertical slice:

- Does each phase have a Backend section and a Frontend section? (A phase may be backend-only or frontend-only if the slice is genuinely vertical and intentional.)
- Can each phase be implemented independently without blocking on another in-progress phase? Flag hidden or circular dependencies.
- Is each phase's scope achievable by a single agent in one focused session? A phase with more than ~10 backend steps or ~8 frontend components is likely too large.
- Are phases ordered correctly? Phase N must not depend on work completed in Phase N+1.

| Severity | Condition |
|---|---|
| Critical | Phases have circular dependencies or a phase is so large it cannot be implemented by one agent |
| Important | Dependencies between phases are implicit rather than stated; phase size is borderline |
| Minor | Phase naming or ordering could be clearer |

---

### 3. Implementation completeness (within each phase)

**Backend section**: verify dependency order and completeness:
- Migrations before model changes; models before schemas; schemas before routers
- Every new endpoint has: HTTP method, path, auth requirement, response schema, and edge cases
- Auth requirements are explicit on every endpoint — not assumed or inherited
- Required env vars are listed if new ones are introduced
- Migration safety is addressed: rollback strategy, backfill concerns if applicable

**Frontend section**: verify completeness:
- New components are named with props and emits specified
- Modified components describe what changes (the delta), not just "update X"
- New API method additions are listed
- Route changes are explicit when applicable

| Severity | Condition |
|---|---|
| Critical | Missing auth on an endpoint; migration with no rollback; steps that reference a model or component not yet created in that phase |
| Important | Steps vague enough that an implementing agent would need to make architectural decisions that should be resolved now |
| Minor | Wording is unclear but intent is recoverable from context |

---

### 4. Test strategy completeness

Review TEST_PLAN.md against the PRD and plan:

- Does it cover the critical paths for every P0 feature?
- Are error and edge cases represented, not just the happy path?
- Are test scenarios specific enough for an implementing agent to write tests? ("Test that search returns empty list when no results match" is specific. "Test search" is not.)
- Are test tiers correctly assigned: unit, integration, contract/E2E?

| Severity | Condition |
|---|---|
| Critical | No tests planned for a P0 critical path |
| Important | Test scenarios cover only happy paths with no error or edge cases |
| Minor | A non-critical path has thin coverage |

---

### 5. Critical scope flags

The plan must include a Critical Scope Flags section listing the highest-risk items:

- Are security boundaries (auth checks, input validation, CORS) flagged?
- Are breaking changes to existing behavior flagged?
- Are external dependencies (third-party APIs, env vars, infrastructure) flagged?
- Are the flags specific enough to act on, or generic?

| Severity | Condition |
|---|---|
| Important | Critical Scope Flags section is missing or empty when the plan clearly has high-risk items |
| Minor | Flags are present but could be more specific |

---

## Acceptance criteria generation

When your verdict is **Approved** or **Approved with concerns**, generate an `ACCEPTANCE_CRITERIA.md` file for each implementation phase. Write each file to `{feature-dir}/phase-{N}/ACCEPTANCE_CRITERIA.md`. Create the directory if it does not exist.

Derive the criteria directly from the phase content. Every criterion must trace to a plan step or PRD requirement.

---

### Tier 1 — Deterministic (run a command, check the result)

List every shell command that must exit 0. Be specific — target the minimal test suite for this phase's work rather than the entire suite when possible.

Examples to derive from the plan:
- `pytest tests/api/test_{module}.py -v` — from backend test steps
- `ruff check app/` — backend lint
- `alembic upgrade head && alembic downgrade -1` — migration safety
- `npx tsc --noEmit` — frontend type check
- `npm run test:unit -- src/components/{ComponentName}.spec.ts` — frontend unit tests
- `npm run build` — frontend build

---

### Tier 2 — Agent-verifiable (evaluator inspects code)

List specific structural checks an agent can verify by reading files or the diff. Be concrete — name the file, endpoint, component, or field.

Examples:
- "`app/routers/trips.py` exports `GET /api/v1/trips` with auth middleware applied"
- "ORM model `Trip` includes columns: `id`, `name`, `date`, `price`, `user_id`"
- "Component `TripCard.vue` accepts props: `trip: Trip`, `onSelect: () => void`"
- "API method `fetchTrips(params: TripSearchParams)` added to `src/api/trips.ts`"
- "Response schema `TripListResponse` includes `items: Trip[]` and `total: number`"

---

### Tier 3 — Visual/UX (browser or human verification)

List what to verify visually or through browser interaction. Reference the UX notes or PRD Design Specs section. Be specific about what to see, not just "UI looks right."

Examples:
- "Trip list renders with name, formatted date, and price per card — in that vertical order"
- "Empty state shows 'No trips yet' message and a call-to-action button when list is empty"
- "Search input clears its value on route navigation away from the list"
- "Loading state shows skeleton cards (not blank space) while the API call is in flight"

---

### Tier 4 — Product acceptance (human judgment required)

List the product-level questions only a human can answer. Keep this short — if Tiers 1–3 are thorough, Tier 4 should be a small set of genuine judgment calls.

Examples:
- "Does the trip listing flow feel appropriate for booking use cases?"
- "Is the error state message appropriate in tone — not alarming, but clear?"
- "Are there edge cases from the PRD not covered by the above checks?"

---

## Verdict definitions

| Verdict | Meaning |
|---|---|
| **Approved** | No Critical findings. At most Minor findings. Plan is ready for implementation. |
| **Approved with concerns** | No Critical findings. Important findings noted but implementation can proceed. Concerns should be tracked. |
| **Needs revision** | One or more Critical findings present. Do not proceed to implementation. |
