# Security Tools

A set of skills and agents that bring structured, context-aware security practice into the development workflow. The plugin is built around a three-phase approach: establish shared vocabulary and standards (via the org's Configuration and Hardening Guide), generate a codebase-specific threat model before reviewing, and then run code reviews scoped to what actually matters for that service.

The result is that security findings are categorized — policy violations, conformance gaps, and code-level vulnerabilities — rather than produced as an undifferentiated severity list. When no threat model or CHG is present, the tools fall back to a generic OWASP/CWE-based review.

---

## Skills

| Skill | Description |
|---|---|
| `/security-threat-model` | Generate a threat model for the current codebase using the Four Question Framework. Loads the CHG to pre-populate applicable policies and standards, then interviews the developer for codebase-specific context. Produces `THREAT_MODEL.md`. See [skill README](skills/security-threat-model/README.md). |
| `/scan-dependencies` | Scans project dependencies for known vulnerabilities using Aikido (preferred) or Snyk. Suggests remediations. |

## Agents

| Agent | Description |
|---|---|
| Security Code Reviewer | Performs static analysis style security review of code changes. Loads `THREAT_MODEL.md` and the CHG to scope findings into three categories: policy violations (breach a CHG policy), conformance gaps (not using approved pattern), and vulnerabilities (code-level issues). Falls back to a generic OWASP/CWE review if no threat model is present. Runs proactively after significant features are completed. |

---

## Dependencies

**Required for `/scan-dependencies`:**
- [Aikido Local Scanner](https://app.aikido.dev/settings/integrations/localscan) (preferred) — binary or Docker image. Requires `AIKIDO_API_KEY`.
- [Snyk CLI](https://docs.snyk.io/snyk-cli) (fallback) — `npm install -g snyk`. Requires `SNYK_TOKEN`.

**Optional for `/security-threat-model`:**
- [orientation-tools plugin](../orientation-tools/) — if the `codebase-cartographer` skill has been run, the threat model skill uses `.orientation/CODEBASE_MAP.md` as a fast path for codebase exploration instead of performing its own scan. Produces a richer threat model with less interviewing required.

**Optional for all skills:**
- [virtual-ciso plugin](../virtual-ciso/) — `/vciso-chg-init` produces the Configuration and Hardening Guide that `security-threat-model` and `security-code-reviewer` use to enable policy and conformance checking. Without it, skills run in standalone vulnerability-detection mode.

---

## Configuration

### `.claude-plugins.json` (for `/scan-dependencies`)

Create a `.claude-plugins.json` file in your project root with the keys you need. Add it to `.gitignore` — it contains secrets.

```json
{
  "aikido_api_key": "AIK_CI_your_key_here",
  "snyk_token": "your_snyk_token_here"
}
```

Alternatively, set `AIKIDO_API_KEY` or `SNYK_TOKEN` as environment variables. Config file takes precedence over environment variables.

### CLAUDE.md Configuration

Add a **Security Tools** section to your project's `.claude/CLAUDE.md` to configure paths.

```markdown
## Security Tools
- security-chg-path: /path/to/Configuration and Hardening Guide.md
- threat-model-path: docs/security/THREAT_MODEL.md
```

| Key | Default | Used by |
|---|---|---|
| `security-chg-path` | _(none)_ | `/security-threat-model`, `security-code-reviewer` — path to the org's Configuration and Hardening Guide. If not set, skills run in standalone mode without policy or conformance checks. |
| `threat-model-path` | `THREAT_MODEL.md` (project root) | `/security-threat-model` — output path for the generated threat model. Overridden by `--output-path` argument if provided. |

Precedence: per-invocation argument > `CLAUDE.md` value > default.

---

## Research & References

**Anthropic defending-code-reference-harness** — the design of `security-threat-model` and `security-code-reviewer` is informed by Anthropic's published approach to LLM-assisted code security. Key principles borrowed: threat modeling as a prerequisite to scanning (not an afterthought), separate discovery and verification phases, and using the threat model to scope and filter findings rather than producing an unscoped list.
- Repo: [anthropics/defending-code-reference-harness](https://github.com/anthropics/defending-code-reference-harness)
- Blog: [Using LLMs to secure source code](https://claude.ai/blog/using-llms-to-secure-source-code)

**Shostack Four Question Framework** — the structure of `THREAT_MODEL.md` and the threat modeling interview in `/security-threat-model` follows Adam Shostack's Four Question Framework: (1) What are we working on? (2) What can go wrong? (3) What are we going to do about it? (4) Did we do a good enough job? The framework is also the basis for the vocabulary standardization built into the CHG.
- Paper: [The Four Question Framework for Threat Modeling](https://shostack.org/files/papers/The_Four_Question_Framework.pdf)
