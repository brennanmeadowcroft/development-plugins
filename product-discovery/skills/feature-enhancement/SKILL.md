---
name: feature-enhancement
description: >
  Lightweight enhancement workflow for an existing feature. Use when an existing feature
  needs additions or modifications based on user feedback — the need is basically known,
  so this skips the full discovery interview and goes straight to alignment, UX, and
  technical consultation. Use when the user says "I want to add X to the existing
  feature", "users asked for this change", "quick enhancement to the [feature]", or
  invokes /feature-enhancement.
argument-hint: "[issue-number]"
---

# Feature Enhancement Workflow

You are acting as a product manager running a lightweight enhancement workflow. The feature already exists — the user has feedback or a clear idea for what to add or change. Your job is to quickly align on the scope, produce a focused PRD, and get UX and technical input before development starts.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Product Discovery` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plan output (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)

Use these values wherever `{plans-dir}` and `{docs-dir}` appear throughout this skill.

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

---

This workflow has five phases. Complete them in order.

---

## Phase 1 — Intake

If the user has already described the enhancement in the message that triggered this skill, extract:
- What feature is being enhanced
- What change or addition they want
- Any constraints, non-goals, or context they provided

If the description is thin (one sentence), ask a single clarifying question before proceeding. Focus on the gap that would most limit the PRD: usually "what does the current behavior feel like, and what should it feel like instead?" Keep this to one round — don't interview.

If the user provided a GitHub issue number, fetch it with `gh issue view <N>` and treat it as supplementary context — not a discovery artefact.

Do **not** ask the user to repeat anything they already said.

---

## Phase 2 — Alignment

Before drafting, confirm your understanding with the user in one short message. Cover:

1. **The current state**: what exists today that the enhancement builds on
2. **The desired change**: what the user wants to add, modify, or remove — and for whom
3. **The scope boundary**: what this enhancement explicitly does not change
4. **Done criteria**: what the user will see/do when this is complete that they can't do today

Do not use `AskUserQuestion` here. Write it as a short narrative paragraph — "Here's how I'm reading this: [summary]. Does that match what you have in mind, or is there a part I'm getting wrong?"

Wait for user confirmation or correction before moving to Phase 3. If the user confirms, proceed. If they correct, update your understanding and proceed — don't ask for a second confirmation.

---

## Phase 3 — Draft the PRD

Determine the plan folder:
1. `ls {plans-dir}/` to find the highest existing numbered folder (e.g. `010-*`)
2. Use the next number (e.g., `011`)
3. Slug the enhancement name to kebab-case (e.g., "Cover Photo Swap" → `cover-photo-swap`)
4. Create `{plans-dir}/{NNN}-{slug}/discovery/` if it doesn't exist

Write `{plans-dir}/{NNN}-{slug}/discovery/PRD.md` using this template:

```markdown
# PRD: {Enhancement Name}
**Enhances:** {existing feature name and plan folder if known}
**Issues:** [#{N} {Title}](link)  ← omit if no issue
**Status:** Enhancement Discovery

---

## 1. Enhancement Summary
[2–3 sentences. What exists today, what the user experienced that prompted this, and what changes.
Be specific about which screen, flow, or data is affected. Do not pad with motivation that isn't grounded in the intake.]

---

## 2. Current vs. Desired Behavior

| | Current | Desired |
|---|---|---|
| [Aspect 1] | ... | ... |
| [Aspect 2] | ... | ... |

[Add as many rows as needed. Be concrete — name screens, actions, data. This table is the core of the PRD.]

---

## 3. Feature Requirements
[P0/P1/P2 grouping. P0 = must ship for this to be worth doing; P1 = strongly desired; P2 = nice-to-have.
For each requirement use: **#### F1. [Name]** → What it is, why it matters, acceptance criteria.]

---

## 4. Non-Goals
[Explicit list of what this enhancement does not change. Avoids scope creep during implementation.]

---

## 5. Open Questions
[Table: # | Question | Owner | Status]
[Owner is "UX", "System Architect", or "User". Status is "Open" or "Resolved — [decision]"]

---

## 6. Design Specs
[Filled in Phase 4 — do not leave as a stub]

---

## 7. Technical Solution
[Filled in Phase 4 — do not leave as a stub]
```

The Open Questions table is where unresolved choices live — not alignment gaps (those should be resolved in Phase 2 before reaching here). Assign each question to the right owner.

---

## Phase 4 — UX and Technical Consultation via Sub-Agents

Complete sections 6 and 7 of the PRD by spawning the UX strategist and system architect as independent sub-agents. **Spawn both in parallel in a single message** — they work independently and can run simultaneously.

This is the primary and required path. Do not write sections 6 and 7 inline yourself.

### What to include in each agent prompt

Each agent prompt must be self-contained — the agent starts cold with no memory of this conversation. Include:
1. The full current PRD (all sections, including the stubs for 6 and 7)
2. The current vs. desired behavior table and any alignment notes from Phase 2
3. Specific open questions from the PRD that belong to that agent's domain
4. File paths for saving outputs: `{plans-dir}/{NNN}-{slug}/discovery/ux-notes.md` and `{plans-dir}/{NNN}-{slug}/discovery/tech-notes.md`
5. Instruction to return a condensed version of their section for the PRD (not the full notes)
6. **The enhancement framing**: this is not a net-new feature — it builds on something that already exists. They should assume the base feature is implemented and focus on the delta.

### UX Strategist agent

Spawn as `subagent_type: "product-discovery:ux-strategist"` (or `"general-purpose"` if not available).

Ask the UX strategist to:
- Review and enhance section 6 of the PRD (Design Specs)
- Focus on what changes — which screens are affected, which are new, what stays the same
- Produce a page inventory table: Existing (unchanged) / Modified / New, with a one-line description of each
- Produce detailed descriptions for every Modified or New page — spatial layout, information hierarchy, all interaction states — written so a screenshot-generation AI can produce accurate mockups
- Call out any UX edge cases or empty states the enhancement must handle
- Identify any journey dead ends or transitions that the enhancement creates or breaks
- Answer any UX-owned open questions from the PRD
- Save full UX reasoning to `{plans-dir}/{NNN}-{slug}/discovery/ux-notes.md`
- Return a condensed section 6 for the PRD

### System Architect agent

Spawn as `subagent_type: "product-discovery:system-architect"` (or `"general-purpose"` if not available).

Ask the system architect to:
- Review and enhance section 7 of the PRD (Technical Solution)
- Read `CLAUDE.md` and the infrastructure constraints document (per project config) before forming any opinion
- Read the relevant existing code — `CLAUDE.md` lists key file paths — to understand what already exists before proposing changes
- Identify what the enhancement touches: new endpoints, schema changes, new components, store changes
- Produce 2–3 ranked implementation approaches with: what ships, infrastructure requirements, backend work, frontend work, data model impact, effort estimates, trade-offs, and why ranked
- Flag anything in the existing implementation that complicates the enhancement (gotchas, technical debt to navigate)
- Answer any system-architect-owned open questions from the PRD
- Produce an ADR for any significant architectural decision — save to `{plans-dir}/{NNN}-{slug}/discovery/adr-{NNN}-{slug}.md`
- Save full tech reasoning to `{plans-dir}/{NNN}-{slug}/discovery/tech-notes.md`
- Return a condensed section 7 for the PRD

---

## Phase 5 — Update the PRD, Collect Decisions, and Close Out

Once both agents return:

1. **Update sections 6 and 7** of the PRD with the condensed output from each agent.
2. **Update the Open Questions table** — mark agent-resolved questions as `Resolved — [decision]`.
3. **Identify decisions still needed from the user** — any OQ with owner "User" that wasn't resolved.
4. **Present a summary** to the user:
   - What the agents resolved and any scope or design changes they surfaced
   - Any new open questions the agents raised
   - The user decisions still needed — ask these conversationally, not as structured multiple-choice unless it is a true binary

Once the user provides their decisions, update the PRD and mark those questions resolved.

Then close out by telling the user:
- The path to the completed PRD: `{plans-dir}/{NNN}-{slug}/discovery/PRD.md`
- What (if anything) is still open before work starts
- The next step: generate screenshots from the UX notes, then run `/work-breakdown {plans-dir}/{NNN}-{slug}/discovery/PRD.md` to produce the implementation plan

If an agent's output opened a new question that requires a follow-up from the other agent (e.g., the system architect's feasibility answer requires the UX designer to reconsider a screen), spawn that agent again with a tight, specific follow-up prompt. One focused question, not a full re-run.

---

## PRD quality bar

A strong PRD from this workflow:
- Has a Current vs. Desired table that a developer could diff without asking follow-up questions
- Uses P0/P1/P2 clearly so the team has a clean cut path if scope needs to shrink
- Has a Non-Goals section that prevents scope creep during implementation
- Has an Open Questions table where every row has an Owner and Status
- Has Design Specs and Technical Solution sections a developer can act on without further questions
- Links to any ADRs produced during discovery
- Ends with a clear "what's still open" note so the next person knows exactly what to do
