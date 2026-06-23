# Development Plugins

A personal coding harness built on top of Claude Code, covering the full arc of software development — from discovery through planning, building, and review before shipping.

Each phase produces an artifact that feeds the next. The outputs of discovery become the inputs to planning; the approved plan becomes the brief for each implementing agent.

---

## Core Workflow

```
/product-discovery
       ↓
   PRD.md
       ↓
/work-breakdown
       ↓
   PLAN.md + phase-N/ folders (each with PHASE_PLAN.md + ACCEPTANCE_CRITERIA.md)
       ↓
/build-feature-from-plan
       ↓
   code + tests, validated by phase
```

### Discover

Turn a GitHub issue or idea into a PRD with UX specs and a recommended technical approach.

| Skill                                   | When to use                                                                                              |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `/product-discovery [issue-number ...]` | Full discovery: interviews you, writes the PRD, consults UX strategist and system architect in parallel. |
| `/feature-enhancement [issue-number]`   | Lightweight version for enhancements where the need is already well understood.                          |

---

### Plan

Translate the PRD into a concrete, phase-based implementation plan. The plan goes through an evaluator loop — up to 3 rounds of critique and revision — before any code is written. On approval, each phase gets its own folder with everything an implementing agent needs.

| Skill                           | When to use                                                                                                                                                                       |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/work-breakdown <path-to-prd>` | The main entry point. Spawns tech-lead, ui-architect, and qa-architect to produce `PLAN.md` + `TEST_PLAN.md`, runs the plan evaluation loop, and materializes `phase-N/` folders. |
| `/plan-evaluate <plan-folder>`  | Re-runs the evaluation loop on an existing plan after changes.                                                                                                                    |
| `/qa-architect <plan-folder>`   | Regenerates `TEST_PLAN.md` without rerunning the full breakdown.                                                                                                                  |

Each phase folder contains a `ACCEPTANCE_CRITERIA.md` that the evaluator derives from the plan — organized into four tiers so the code evaluator handles what's deterministic and you only review what requires human judgment:

| Tier                   | Who checks it                                                      |
| ---------------------- | ------------------------------------------------------------------ |
| 1 — Commands           | Code evaluator (tests pass, lint, build, migration)                |
| 2 — Code structure     | Code evaluator (endpoints exist, components created, auth applied) |
| 3 — Visual / UX        | You, in the browser                                                |
| 4 — Product acceptance | You                                                                |

---

### Build

Implement each phase. Backend and frontend agents write their own tests (TDD). A code evaluator reviews each component automatically. You see only what requires human judgment — Tier 3 and Tier 4.

| Skill                                                 | When to use                                                                                                               |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `/build-feature-from-plan <plan-folder> [phase\|all]` | Main entry point. Runs backend, code-evaluate, frontend, code-evaluate, then pauses for your Tier 3 + 4 review per phase. |
| `/code-evaluate <phase-folder> [backend\|frontend]`   | Re-evaluate a component without rerunning the full build.                                                                 |

---

### Review

After the feature is on a branch.

| Skill                    | When to use                                                             |
| ------------------------ | ----------------------------------------------------------------------- |
| `/code-review`           | Reviews the current diff for correctness, reuse, and efficiency issues. |
| `/scan-dependencies`     | Scans for known dependency vulnerabilities.                             |
| `/security-threat-model` | Generates a threat model scoped to this codebase and feature.           |

---

## Supplementary Tools

These can be used at any point in the workflow — not tied to a specific phase.

| Skill                    | Plugin                                           | What it does                                                                                                                           |
| ------------------------ | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `/codebase-orientation`  | [orientation-tools](orientation-tools/README.md) | Full orientation brief: architecture, key files, patterns, and gotchas. Useful before planning or when joining an unfamiliar codebase. |
| `/codebase-health`       | [orientation-tools](orientation-tools/README.md) | Risk profile and tech debt assessment before large changes.                                                                            |
| `/codebase-cartographer` | [orientation-tools](orientation-tools/README.md) | Navigable codebase map scoped to a specific task.                                                                                      |
| Vue architecture review  | [vue-tools](vue-tools/README.md)                 | Component design, state management guidance, and feature organization for Vue.js projects.                                             |

---

## Installation

### Add the marketplace

```bash
claude plugin marketplace add /path/to/development-plugins
```

### Install plugins

```bash
claude plugin install development-tools@bmeadowcroft-development-plugins
claude plugin install product-discovery@bmeadowcroft-development-plugins
claude plugin install orientation-tools@bmeadowcroft-development-plugins
claude plugin install security-tools@bmeadowcroft-development-plugins
claude plugin install vue-tools@bmeadowcroft-development-plugins
```

### Update after changes

```bash
git pull
claude plugin update development-tools@bmeadowcroft-development-plugins
```
