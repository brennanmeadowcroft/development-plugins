---
name: security-threat-model
description: >
  Generate a threat model for the current codebase using the Four Question Framework. Loads the org's Configuration and Hardening Guide (CHG) to pre-populate applicable policies and standards, then interviews the developer to fill in codebase-specific context. Produces a THREAT_MODEL.md consumed by security-code-reviewer. Run when starting a new project, before a major launch, or when the architecture changes significantly.
allowed-tools: Read, Write, Bash
argument-hint: '[--output-path <path>]'
---

# Security Threat Model

You are generating a threat model for this codebase using the Four Question Framework. The output is a `THREAT_MODEL.md` file that scopes and focuses security reviews — it tells the scanner what matters for this specific project, what policies apply, and what to deprioritize.

Work methodically: explore the codebase first, load the CHG if available, then conduct a focused interview. Don't ask about things you can determine from the code.

## Arguments

- `--output-path <path>` (optional) — directory or full file path to write `THREAT_MODEL.md`. If a directory is given, the file is named `THREAT_MODEL.md` inside it. Defaults to the project root, or the `threat-model-path` key in `CLAUDE.md` if set.

---

## Phase 0: Resolve Paths, Load CHG, Check for Existing Threat Model

**1. Resolve the output path**

Check for a `--output-path` argument. If provided, use it as the output directory (append `THREAT_MODEL.md` if it doesn't end in `.md`).

If no argument, check `.claude/CLAUDE.md` (project-level) for a `## Security Tools` section with a `threat-model-path` key.

Default if neither is set: `THREAT_MODEL.md` in the project root.

**2. Find the CHG**

Check `.claude/CLAUDE.md` for a `## Security Tools` section with a `security-chg-path` key. If a path is configured and the file exists, read it. If not configured or not found, proceed without the CHG — flag this at the end as a gap.

**3. Check for an existing threat model**

If a `THREAT_MODEL.md` already exists at the resolved output path, read it. Offer to update specific sections rather than regenerating from scratch.

**4. Find the template**

Locate `THREAT_MODEL.template.md` alongside this skill file. Use `find` to locate the skill directory if needed:

```bash
find ~/.claude -name "THREAT_MODEL.template.md" 2>/dev/null | head -1
```

Read the template — it defines the output structure. Do not modify the template; use it only as a reference for the document you will write.

---

## Phase 1: Explore the Codebase

Before asking anything, learn as much as possible from the project. Follow this fast-path / fallback approach:

**Step 1 — Check for orientation-tools output (fast path)**

Look for `.orientation/CODEBASE_MAP.md` and `.orientation/ORIENTATION.md`. If found, read both. These are produced by the orientation-tools plugin and contain synthesized data flows, external boundaries, entry points, and module dependencies — exactly what this skill needs. Skip Steps 2 and 3 if the map is present and recent.

If `.orientation/` doesn't exist, note at the end that running `/codebase-cartographer` before re-running this skill would produce a richer threat model.

**Step 2 — Locate architecture documentation**

Check `.claude/CLAUDE.md` for any key pointing to architecture or documentation files (look for keys like `docs-folder`, `architecture-doc`, `docs-path`, or similar). If found, read the referenced file or list that folder.

If no CLAUDE.md pointer exists, check these locations in order:

1. `ARCHITECTURE.md` in the project root
2. `docs/ARCHITECTURE.md`
3. `docs/architecture.md`
4. Any `.md` file in `docs/` or `documentation/` whose name suggests architecture or design

**Step 3 — Fallback scan (only if Steps 1–2 gave insufficient context)**

```bash
find . -maxdepth 3 \
  -not -path '*/node_modules/*' \
  -not -path '*/.git/*' \
  -not -path '*/dist/*' \
  -not -path '*/.next/*' \
  -not -path '*/build/*' \
  | sort
```

Read dependency manifests to identify the stack: `package.json`, `go.mod`, `requirements.txt`, `Cargo.toml`, `pom.xml`, `composer.json`, `Gemfile`.

Read infrastructure and config signals: `.env.example`, `config/`, `terraform/`, `*.tf`, `docker-compose.yml`, `k8s/`, `Dockerfile`.

**Extract from whichever sources you read:**

- **Stack and languages** in use
- **Service type signals**: HTTP routes, queue consumers, data storage, internet-facing exposure
- **External dependencies**: databases, third-party APIs, cloud services
- **Entry points**: where untrusted input enters the system
- **Trust boundaries**: where this service ends and external systems begin

Brief the developer in one short paragraph on what you found and how you found it, then move to the interview.

---

## Phase 2: Interview

Use the Four Question Framework. Be direct — this is a working session. Adapt based on what the codebase already told you. If something is obvious from the code, state it and ask for confirmation rather than asking from scratch.

Use the org vocabulary from CHG Section 1 (threat actors, asset tiers, trust zones, severity labels) if the CHG was loaded. If not, use plain language.

### Question 1: What Are We Working On?

Cover what the code couldn't tell you:

- **Purpose**: What does this service do? Who are its users (internal team, external customers, automated systems)?
- **Data**: What sensitive data does it handle or store? If the CHG defines asset tiers, which tier does the data fall into?
- **Deployment**: How is this deployed — cloud provider, region, self-hosted? Internet-facing, internal-only, or both?
- **Trust boundaries**: What systems does this talk to? Which are trusted, which are untrusted or external?
- **Authentication**: Who can call this? Is it behind auth? What kind?

### Question 2: What Can Go Wrong?

Draw on what you know about the codebase's attack surface. Present candidate threats and ask the developer to confirm, correct, or add. Use CHG threat actor vocabulary if available.

Cover:

- **Realistic threat actors for this service**: Which threat actors would target this specifically, and what would they be after?
- **Highest-consequence failures**: What's the worst realistic outcome? Data exfiltration, service disruption, privilege escalation, financial loss?
- **Known weak spots**: Areas the developer is already concerned about? Features built under time pressure? Third-party integrations not yet reviewed?
- **What's explicitly off the threat model**: What attacks aren't realistic for this service, so the scanner doesn't waste time on them?

### Question 3: What Are We Going to Do About It?

Inventory what's already in place, then identify gaps.

- **Existing controls**: What security measures are already implemented? (Auth, input validation, secret management, logging)
- **CHG conformance check** (if CHG loaded): Walk through applicable standards from CHG Section 3 for this stack. For each, ask whether the project is using the approved pattern. Note gaps.
- **CHG policy check** (if CHG loaded): Walk through applicable policies from CHG Section 2 for this service type. Confirm which are met, which need verification.
- **Known gaps**: What security work is known to be incomplete or deferred?

### Question 4: Did We Do a Good Enough Job?

- **Review cadence**: When should this threat model be revisited? (After major architecture changes, before each release, quarterly?)
- **Validation**: How will the team verify that the mitigations identified here are actually working?

---

## Phase 3: Generate THREAT_MODEL.md

Use the structure from `THREAT_MODEL.template.md`. Present the outline and ask for confirmation before writing.

Populate every section from what you learned in the exploration and interview phases. Omit the Policy Status and Standards Conformance tables if no CHG was loaded. Fill the Scan Guidance section carefully — this is what `security-code-reviewer` reads to scope its work.

Write the file to the resolved output path.

---

## Completion

After writing:

1. **File written** — confirm the full path
2. **CHG gap** — if no CHG was available, flag it and note that running `/vciso-chg-init` would enable policy and conformance checks in future threat models
3. **Open items** — list any policy gaps or conformance gaps surfaced during the interview
4. **Next step** — suggest running `security-code-reviewer` using this threat model as context
