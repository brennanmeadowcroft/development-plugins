# security-threat-model

Generates a `THREAT_MODEL.md` for the current codebase using the Four Question Framework. The threat model is the scoping document that `security-code-reviewer` reads before doing any analysis — it tells the scanner which threat actors are realistic, what data is in scope, which CHG policies apply, and what to deprioritize.

Running this skill before a code review is what enables the three-category finding structure (policy violations, conformance gaps, vulnerabilities). Without it, the reviewer falls back to a generic OWASP/CWE scan with no org context.

---

## The Four Question Framework

The skill structures both its interview and its output around Adam Shostack's Four Question Framework for threat modeling:

1. **What are we working on?** — System decomposition: architecture, components, data flows, trust boundaries, entry points.
2. **What can go wrong?** — Threat identification: which threat actors are realistic, what would they target, what are the highest-consequence failure modes.
3. **What are we going to do about it?** — Controls inventory and gap identification: what's already in place, what CHG policies and standards apply, what's missing.
4. **Did we do a good enough job?** — Validation and review cadence: how controls will be verified and when the threat model should be revisited.

The framework was chosen because it produces threat models that are actionable at the code level (not just architecture), and because it uses the same vocabulary defined in the CHG — ensuring findings from the code review can be traced back to the org-level threat model.

Reference: [The Four Question Framework for Threat Modeling](https://shostack.org/files/papers/The_Four_Question_Framework.pdf)

---

## CHG Integration

When the Configuration and Hardening Guide is configured via `security-chg-path` in `CLAUDE.md`, the skill:

- Loads **Section 1 (Vocabulary)** to use org-standard threat actor names, asset tiers, trust zone labels, and severity definitions throughout the interview and output
- Pre-populates **Section 2 (Policies)** applicable to this service type into the threat model's Policy Status table
- Pre-populates **Section 3 (Standards)** applicable to this stack into the Standards Conformance table

These pre-populated tables are what enables the code reviewer to produce policy violation and conformance gap findings — it reads them directly from the threat model's Scan Guidance section.

If no CHG is configured, the skill runs in standalone mode: the threat model is generated from first principles using the interview, and the code reviewer produces vulnerability findings only.

---

## Codebase Exploration

The skill follows a fast-path / fallback approach for exploring the codebase before the interview:

1. **orientation-tools fast path** — reads `.orientation/CODEBASE_MAP.md` and `.orientation/ORIENTATION.md` if the [orientation-tools plugin](../../../orientation-tools/) has been run. These files contain synthesized data flows, external boundaries, and module dependencies from a multi-agent deep scan — substantially richer than what a single-pass file read produces.
2. **Architecture docs** — checks `CLAUDE.md` for a docs path pointer, then looks for `ARCHITECTURE.md` in the project root and common doc directories.
3. **Fallback scan** — reads dependency manifests and infrastructure config to detect the stack and surface entry points when neither of the above is available.

Running `/codebase-cartographer` before `/security-threat-model` on a new or complex codebase produces a significantly more accurate threat model with fewer interview questions needed.

---

## THREAT_MODEL.md Structure

The output file follows the template in `THREAT_MODEL.template.md` alongside this skill. Key sections:

| Section | Purpose |
|---|---|
| Questions 1–4 | The four-question threat model, populated from exploration + interview |
| Policy Status | Table of applicable CHG policies and their current status (met / gap / unverified) |
| Standards Conformance | Table of applicable CHG standards and their current conformance status |
| **Scan Guidance** | Machine-readable section consumed by `security-code-reviewer` |

The **Scan Guidance** section is the most important for downstream tooling. It contains:
- Service classification (internet-facing, internal, data-processing)
- Policy checks — what a violation looks like in this codebase's code
- Conformance checks — what non-conformance looks like for this stack
- Priority focus areas — specific files, routes, or patterns the scanner should start with
- Deprioritize list — what to skip or treat as low priority

Keeping the Scan Guidance section accurate and specific is what makes the difference between a focused, high-signal code review and a noisy one.

---

## Arguments

| Argument | Required | Description |
|---|---|---|
| `--output-path <path>` | No | Directory or full path to write `THREAT_MODEL.md`. Defaults to project root or `threat-model-path` in `CLAUDE.md`. |
