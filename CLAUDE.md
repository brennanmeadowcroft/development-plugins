# Claude Plugins — Developer Guide

## Documentation
All plugins require a README.  It should include details about:
* the purpose of the plugin
* the skills available
* any relevant documentation that informed the structure or content of the plugin
* any dependencies the plugin requires

The plugin README does not require installation instructions.  That is handled by the repo README.

Particular skills may require their own READMEs when the content has a specific structure or has detailed considerations. For instance, if research was used to develop a particular skill such as a security agent for a particular stack, that skill should have it's own dedicated README to outline the research.

## File Outputs / Generated Files
When a skill outputs files, the structure of those files should be maintained in a template file. The template file should be named `<output_filename>.template.<extension>`.  The structure of the output should not be in the SKILL.md.  SKILL.md should reference the template file.

## Git commit message conventions
Main commit message should follow the format:
```
<type-of-fix>(<plugin-name>): top level overview
```

Options for the type-of-fix are:
* `feat` - changes to existing functionality
* `fix` - changes meant to fix an issue to ensure existing functionality works as intended
* `chore` - small, non-functional adjustments or updating a dependency

The description should include relevant and useful detail about what work was completed.

If a Github issue was referenced in the original prompt, the commit message should include `(closes #<github_issue>)`.

## Skill Configuration Overrides

Skills that read user-configurable paths or settings follow a consistent convention: the user adds a named section to their vault's `CLAUDE.md`, and the skill reads it at startup. This avoids requiring arguments on every invocation.

### Convention

**In the skill (SKILL.md):** Document a `## Vault configuration` section listing the supported keys, their defaults, and when the skill reads them. State the precedence rule:

> Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

**In the plugin README:** Add a `### CLAUDE.md Configuration` subsection under `## Usage` with:
1. A one-sentence intro explaining what the section does
2. A copyable markdown template the user can paste into their vault's `CLAUDE.md`
3. A key/default/used-by table covering every configurable key
4. The precedence line

**Template for the README section:**

```markdown
### CLAUDE.md Configuration

Add a **<Plugin Name>** section to your vault's `CLAUDE.md` to set persistent path defaults — no arguments needed on every invocation.

\`\`\`markdown
## <Plugin Name>
- key-name: default/example value
\`\`\`

| Key        | Default          | Used by                          |
| ---------- | ---------------- | -------------------------------- |
| `key-name` | `fallback-value` | `/skill-name` — what it controls |

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.
```

**Key format inside the user's CLAUDE.md:** bullet list items under a `##`-level heading matching the plugin name, using `key: value` syntax.

---

## Skill Arguments

### `argument-hint` in frontmatter

Every skill that accepts arguments must declare them in the `argument-hint` frontmatter key. This key drives autocomplete hints and sets the canonical invocation signature. All arguments must be listed here — no undocumented arguments.

```yaml
argument-hint: "<required-arg> [--optional-flag <value>]"
```

### Argument conventions

**Named/optional arguments use `--` prefix:**
```
/skill-name --output-path <path>
/skill-name --mode fast
```

**Required positional arguments (no name prefix):**
```
/skill-name <path>
/skill-name <query>
```
Use positional arguments only when the argument is unambiguous and always required.

**Mode/subcommand as first positional argument:**
When a skill has distinct operating modes, the mode comes first, before any paths or flags:
```
/skill-name update <path> [--flag]
/skill-name init [--output-path <path>]
```

### `## Arguments` section in skill body

Every argument declared in `argument-hint` must also be documented in a `## Arguments` section in the skill body, immediately after the opening description paragraph. Format:

```markdown
## Arguments

- `--flag-name <value>` (optional) — what it controls and when to use it. Defaults to X, or the `key-name` key in `CLAUDE.md` if set.
- `<required-arg>` (required) — what it is.
```

---

## Adding a New Plugin

When a new plugin directory is created, add an entry to `.claude-plugin/marketplace.json` under the `plugins` array:

```json
{
  "name": "name-of-plugin",
  "source": "./<path-to-plugin-dir>",
  "description": "A description of what the plugin does"
}
```

The `name` must match the plugin's directory name and the `name` field in its `plugin.json`. The `source` path is relative to the repo root.

---

## Bumping Plugin Versions

Every change to a plugin requires a version bump in `<plugin-path>/.claude-plugin/plugin.json`. Use semantic versioning (`MAJOR.MINOR.PATCH`):

| Change type                                                                             | Version segment | Examples                                                        |
| --------------------------------------------------------------------------------------- | --------------- | --------------------------------------------------------------- |
| Fixes, rewording, comment-only edits (`fix`, `chore`) — no change to skill behavior     | `PATCH`         | Typo fix, reworded instruction, formatting cleanup              |
| New or improved functionality (`feat`) — skills do something new or meaningfully better | `MINOR`         | New argument, new phase, improved output format, new config key |

Bump the version as part of the same commit as the change — do not batch version bumps separately.
