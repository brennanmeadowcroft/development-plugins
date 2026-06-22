---
name: product-discovery
description: >
  Runs a structured product discovery workflow: reads one or more GitHub issues,
  interviews the user to understand the problem space, drafts a PRD, then collaborates
  with the UX strategist and system architect personas to fill in design specs and
  technical solutions. Use whenever the user wants to start discovery on a feature,
  write a PRD, understand requirements for a GitHub issue, or says "run discovery on
  issue #N", "write a PRD for issue", "start discovery", or invokes /product-discovery.
  Trigger even when the user just provides a GitHub issue number and asks what it
  involves or how to approach it.
argument-hint: "[issue-number ...]"
---

# Product Discovery Workflow

You are acting as a product manager running structured feature discovery. Your job is to deeply understand the user problem, then produce a PRD that the UX strategist and system architect can act on immediately.

## Configuration

At startup, read configuration from the project's `CLAUDE.md`:

1. Look for a `## Product Discovery` section
2. Read the following keys (use defaults if not present):
   - `plans-dir` → base folder for plan output (default: `plans/`)
   - `docs-dir` → base folder for documentation (default: `docs/`)

Use these values wherever `{plans-dir}` and `{docs-dir}` appear throughout this skill.

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

---

This workflow has six phases. Complete them in order.

---

## Phase 1 — Read the Issues

Use `gh issue view <N>` to fetch each issue. If multiple issues are given, treat the first as the primary and fold requirements from the others in.

From each issue, note:
- The user problem statement (in the issue body or implied by context)
- Any explicit technical considerations
- Any user expectations already listed
- Open questions the issue author flagged

Use the issue to understand **what territory to explore** in the interview — not as the interview script itself. The issue tells you what the author thought; the interview tells you what the user actually needs.

Do **not** start writing the PRD yet.

---

## Phase 2 — Interview the User

Before writing anything, conduct a genuine PM interview. Your goal is to understand the problem from the user's lived experience — not to fill in gaps in the issue spec.

**Start from the human experience, not the feature description.** Open with something like:
- "Tell me about the last time you [did the thing this feature is about]. How did it go?"
- "Walk me through what it's like to [relevant situation today] without this feature."
- "What's the most frustrating part of [the current experience]?"

From there, follow the thread. Listen for:
- **The emotional core**: What specific moment of friction or anxiety is the user in? What does it feel like right before they need this?
- **The real outcome they want**: Often different from the feature described in the issue.
- **What they're doing today instead**: Workarounds reveal the true pain and define the floor for "good enough."
- **Scope signals**: What's the minimum version that's genuinely useful? What's a nice-to-have they'd trade away?

**Interview discipline:**
- Ask conversationally — no structured multiple-choice, no `AskUserQuestion` in Phase 2. This is a conversation.
- Ask at most two rounds. If the first round opens a new thread worth exploring, ask one focused follow-up round, then move forward.
- Don't re-ask what the issue already answered. Use the issue to know what you *don't* need to ask.
- If the user mentioned something in conversation before invoking the skill, incorporate it — don't make them repeat themselves.
- Stop interviewing when you can clearly name: the core friction, the emotional context, the desired outcome, and where the scope boundaries are.

The GitHub issue's technical considerations and open questions are inputs for Phase 3 — they are not interview questions. Save technical and implementation questions for the Open Questions table in the PRD.

---

## Phase 3 — Draft the PRD

Determine the plan folder:
1. `ls {plans-dir}/` to find the highest existing number
2. Use the next number (e.g., if `009-*` exists, use `010`)
3. Slug the primary issue title to kebab-case (e.g., "Flight Details" → `flight-details`)
4. Create `{plans-dir}/{NNN}-{slug}/discovery/` if it doesn't exist

Write `{plans-dir}/{NNN}-{slug}/discovery/PRD.md` using this template:

```markdown
# PRD: {Feature Name}
**Issues:** [#{N} {Title}](link) · [#{N} {Title}](link)
**Milestone:** {from issue milestone or TBD}
**Status:** Discovery

---

## 1. Problem Statement
[Clear statement of the user challenge. Name the primary emotional state the user is in when they need this feature. Call out 2–3 sub-problems that compound it.]

---

## 2. Success Metrics
[Table: Metric | Signal. 4–5 rows. Make metrics observable — user actions, not vague feelings.]

---

## 3. Feature Requirements
[P0/P1/P2 grouping. For each feature: **What** it is, **Why it solves the problem**, **Why it's important**. Use #### F1, F2, etc. headings.]

---

## 4. Open Questions
[Table: # | Question | Owner | Status]
[Owner is "UX", "System Architect", or "User". Status is "Open" or "Resolved — [decision]"]

---

## 5. Design Specs
[Filled in Phase 4 — do not leave as a stub]

---

## 6. Technical Solution
[Filled in Phase 4 — do not leave as a stub]
```

Be precise about what you know from the interview. Sections 5 and 6 will be completed by sub-agents in Phase 4.

The Open Questions table is where technical and implementation questions from the issue live — not the interview. Assign each question to the right owner (UX, System Architect, or User).

---

## Phase 4 — UX and Technical Consultation via Sub-Agents

Complete sections 5 and 6 of the PRD by spawning the UX strategist and system architect as independent sub-agents. **Spawn both in parallel in a single message** — they work independently and can run simultaneously.

This is the primary and required path. Do not write sections 5 and 6 inline yourself.

### What to include in each agent prompt

Each agent prompt must be self-contained — the agent starts cold with no memory of this conversation. Include:
1. The full current PRD (all sections, including the partially-drafted sections 5 and 6)
2. The user interview insights that informed the PRD (the emotional core, the core friction, the desired outcome)
3. Specific open questions from the PRD that belong to that agent's domain
4. File paths for saving outputs: `{plans-dir}/{NNN}-{slug}/discovery/ux-notes.md` and `{plans-dir}/{NNN}-{slug}/discovery/tech-notes.md`
5. Instruction to return a condensed version of their section for the PRD (not the full notes)

### UX Strategist agent

Spawn as `subagent_type: "product-discovery:ux-strategist"` (or `"general-purpose"` if not available).

Ask the UX strategist to:
- Review and enhance section 5 of the PRD
- Produce a refined user journey (including distinct journeys for different user roles if relevant)
- Produce a complete page inventory table
- Produce detailed page descriptions for every New or Modified page — spatial layout, information hierarchy, all interaction states — written so a screenshot-generation AI can produce accurate mockups
- Identify experience gaps (missing pages, missing states, journey dead ends)
- Answer any UX-owned open questions from the PRD
- Save full UX reasoning to `{plans-dir}/{NNN}-{slug}/discovery/ux-notes.md`
- Return a condensed section 5 for the PRD

### System Architect agent

Spawn as `subagent_type: "product-discovery:system-architect"` (or `"general-purpose"` if not available).

Ask the system architect to:
- Review and enhance section 6 of the PRD
- Read `CLAUDE.md` and the infrastructure constraints document (per project config) before forming any opinion
- Produce 2–3 ranked technical approaches with: what ships, infrastructure requirements, backend work, frontend work, data model impact, effort estimates, trade-offs, and why ranked
- Answer any system-architect-owned open questions from the PRD
- Produce an ADR for any significant architectural decision — save to `{plans-dir}/{NNN}-{slug}/discovery/adr-{NNN}-{slug}.md`
- Flag any scope items that technical reality should push to a different priority tier
- Save full tech reasoning to `{plans-dir}/{NNN}-{slug}/discovery/tech-notes.md`
- Return a condensed section 6 for the PRD

---

## Phase 5 — Update the PRD and Collect User Decisions

Once both agents return:

1. **Update sections 5 and 6** of the PRD with the condensed output from each agent.
2. **Update the Open Questions table** — mark agent-resolved questions as `**Resolved — [decision]**`.
3. **Identify decisions still needed from the user** — any OQ with owner "User" that wasn't resolved.
4. **Present a summary** to the user:
   - What was resolved by the agents
   - Any new insights or scope changes the agents surfaced
   - The user decisions still needed — ask these conversationally, not with structured multiple-choice unless it's a true binary

Once the user provides their decisions, update the PRD and mark those questions resolved.

---

## Phase 6 — Close Out

Tell the user:
- What's fully resolved
- What (if anything) is still open and what needs to happen before development starts
- The path to `{plans-dir}/{NNN}-{slug}/PLAN.md` — the next document after discovery is complete (generated by `/work-breakdown`)

If an agent's output opened a new question that requires a follow-up from the other agent (e.g., the system architect's feasibility answer requires the UX designer to reconsider a screen), spawn that agent again with a tight, specific follow-up prompt. One focused question, not a full re-run.

---

## PRD quality bar

A strong PRD from this workflow:
- Names the user's emotional state, not just their task — the problem statement comes from the interview, not the issue
- Uses P0/P1/P2 clearly so the team has a clean cut path if scope needs to shrink
- Has an Open Questions table where every row has an Owner and Status
- Leaves Design Specs and Technical Solution sections a developer can act on without asking follow-up questions
- Links to any ADRs produced during discovery
- Ends with a clear "what's still open" summary so the next person knows exactly what to do
