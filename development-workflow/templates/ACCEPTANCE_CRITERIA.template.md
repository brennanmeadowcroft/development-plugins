# Acceptance Criteria: Phase {N} — {Phase Description}

**Feature:** {Feature Name}
**Phase:** {N}

---

## Tier 1 — Deterministic

Run each command. All must exit 0 before the phase is considered complete.

| Command | What it validates |
|---|---|
| `{command}` | {description} |

---

## Tier 2 — Agent-Verifiable

The code evaluator verifies these by reading the implementation directly.

- [ ] {Specific structural check — name the file, endpoint, component, or field}
- [ ] {Another specific check}

---

## Tier 3 — Visual / UX

Verify these in a browser or by inspecting the rendered output.

- [ ] {What to see and where — specific layout, state, or interaction}
- [ ] {Another visual check}

---

## Tier 4 — Product Acceptance

Human judgment required. The user validation gate surfaces these.

- [ ] {Product-level question or acceptance check}
