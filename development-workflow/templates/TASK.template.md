# Task {N}: {Task Name}

**Phase:** {phase number}
**Files:** {N} file(s)
**Model:** haiku | sonnet
**Commit:** `[{plan-name}] {commit message}`

> This is the complete brief for the implementing agent. The agent should not need to read
> PLAN.md or PHASE_PLAN.md — everything is here.

---

## Context

{One sentence: what this task delivers and why it matters in the sequence.}

---

## Files

| Action | Path | Change |
|---|---|---|
| **Create** | `{exact/path/to/new-file.py}` | {what this file is} |
| **Modify** | `{exact/path/to/existing.py}` | {what changes} |
| **Test** | `{exact/path/to/test_file.py}` | {what's covered} |

---

## Interfaces

**Consumes from prior tasks:**
- `{FunctionName / TypeName / endpoint path}` from Task {M} — {what it is, its signature or shape}

**Produces for later tasks:**
- `{FunctionName / TypeName / endpoint path}` — `{signature or shape}` — used by Task {M}

> If this is the first task, "Consumes" is empty. If no later task depends on this task's output, "Produces" is empty.

---

## Critical Scope Flags

{Flags from PLAN.md Critical Scope Flags that apply specifically to this task. Omit this section if none apply.}

---

## Implementation notes

{Exact code snippets, SQL, field definitions, or prop signatures from PLAN.md that the agent needs — copy verbatim from the plan. Do not write "see PLAN.md." If the plan specifies exact code, put it here.}

---

## Test cycle

After implementing, run:

```bash
{exact test command}
```

Expected: {N} tests pass. Relevant assertions: {what must be true}.

{For frontend tasks with no server: `{npm run test -- --testPathPattern=ComponentName}`}
{For tasks with no test file (e.g., migration-only): `{migrate command}` — confirm the table change is present.}

---

## Reviewer checklist

The task reviewer verifies these from the diff:

- [ ] {Specific structural check — name the file, line, or behaviour}
- [ ] {Another structural check}
- [ ] {Critical Scope Flag adherence if applicable}
