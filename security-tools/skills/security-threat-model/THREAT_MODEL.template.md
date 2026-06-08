# Threat Model: [Service Name]

> **Generated:** YYYY-MM-DD
> **Framework:** Four Question Framework (Shostack)
> **CHG:** [date of CHG used, or "not configured — standalone model"]
> **Stack:** [detected stacks]
> **Review cadence:** [from interview]

---

## 1. What Are We Working On?

### Service Overview
[What it does, who uses it, why it exists]

### Deployment Context
[Cloud provider, region, internet-facing vs. internal, containerized or not]

### Data Handled
[Data types and asset tier classification if CHG defines tiers]

### Trust Boundaries
| Component | Trust Level | Notes |
|---|---|---|
| [External users / callers] | Untrusted | ... |
| [Internal services] | [zone] | ... |
| [Databases / storage] | Trusted | ... |

### Entry Points
[Where untrusted input enters the system — HTTP routes, queue consumers, file uploads, webhooks, etc.]

---

## 2. What Can Go Wrong?

### Applicable Threat Actors
[Named personas from org vocabulary, or plain descriptions if standalone. For each: what they want from this service specifically.]

### Threat Scenarios
| Scenario | Actor | Target | Likelihood | Impact |
|---|---|---|---|---|
| [Description] | [actor] | [data or function] | High/Med/Low | High/Med/Low |

### Explicitly Out of Scope
[What the scan should not flag or deprioritize — attacks that aren't realistic for this service]

---

## 3. What Are We Going to Do About It?

### Controls in Place
[Existing security measures — auth, validation, secret management, logging, etc.]

### Policy Status
[Only present if CHG was loaded. For each applicable policy from CHG Section 2:]

| Policy | Status | Notes |
|---|---|---|
| [Policy name] | Met / Gap / Unverified | ... |

### Standards Conformance
[Only present if CHG was loaded. For each applicable standard from CHG Section 3:]

| Domain | Approved Pattern | Status | Notes |
|---|---|---|---|
| [e.g., Auth] | [library/pattern] | Conformant / Gap / TBD | ... |

### Open Items
[Security gaps identified — these are the priority focus for the scan]

---

## 4. Did We Do a Good Enough Job?

### Validation Approach
[How the team will verify mitigations are working]

### Review Triggers
[Conditions that should prompt re-running this threat model]

---

## Scan Guidance

> Consumed by `security-code-reviewer` to scope and categorize findings.

### Service Classification
- **Type:** [internet-facing | internal | data-processing | mixed]
- **Data tiers:** [applicable tiers, or sensitivity description if no CHG]
- **Trust zones crossed:** [list]

### Policy Checks
[For each applicable policy: what a violation looks like in this codebase's code]

### Conformance Checks
[For each applicable standard: the approved pattern and what non-conformance looks like in this codebase's stack]

### Priority Focus Areas
[Top 2–3 areas most likely to have issues based on the threat model above. Be specific — name the files, routes, or patterns to examine.]

### Deprioritize
[What the scan can skip or treat as lower priority — attacks off the threat model, areas already well-controlled, out-of-scope components]
