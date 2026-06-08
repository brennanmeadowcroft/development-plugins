# claude-plugins

A personal coding harness built on top of Claude Code, covering the full arc of the software development lifecycle — from first contact with an unfamiliar codebase through planning, building, and security review before shipping.

Commercial AI coding tools are general-purpose. This plugin collection layers structured, opinionated workflows on top of them: each plugin addresses a specific phase of the SDLC, and the outputs of earlier phases feed into later ones.

## Plugins

These plugins cover the software development lifecycle — the build phase of the broader product development lifecycle.

### Orientation

- [orientation-tools](orientation-tools/README.md) — maps an unfamiliar codebase: structure, architecture, patterns, and risk profile before making changes.

### Architecture & Design

- [vue-tools](vue-tools/README.md) — Vue.js architecture guidance for component design, state management, and multi-team feature organization.
- [security-tools](security-tools/README.md) — generates a threat model scoped to this codebase before significant features are built.

### Implementation

- [development-tools](development-workflow/README.md) — executes a plan document phase-by-phase on the platform server.

### Review & Verification

- [security-tools](security-tools/README.md) — reviews completed code against the threat model; scans dependencies for known vulnerabilities.

---

## Installation

### 1. Add the marketplace

**Claude Code**
```
/plugin marketplace add brennanmeadowcroft/development-plugins
```

**Claude Desktop**
1. Click the "+" on a new chat
2. "Connectors" > "Manage Connectors"
3. "Browse Plugins" in the sidebar
4. Click the "Personal" tab
5. Click the "+"
6. Choose add from Github
7. Add `brennanmeadowcroft/development-plugins` as the repo

**Manual configuration**

You can also add the marketplace directly in `~/.claude/settings.json`:
```json
{
  "extraKnownMarketplaces": {
    "bmeadowcroft-plugins": {
      "source": { "source": "github", "repo": "brennanmeadowcroft/development-plugins" }
    }
  }
}
```

### 2. Install a plugin

**Claude Code**
```
/plugin install <plugin-name>@bmeadowcroft-development-plugins
```

For example:
```
/plugin install orientation-tools@bmeadowcroft-development-plugins
```

## Local development

To test a plugin without pushing to GitHub, add your local clone as a marketplace:

```
/plugin marketplace add /path/to/claude-plugins
```

Then install from it:

```
/plugin install <plugin-name>@bmeadowcroft-development-plugins
```

The marketplace name `bmeadowcroft-development-plugins` comes from the `name` field in `.claude-plugin/marketplace.json`. Changes to skill and agent files are picked up on the next session — no reinstall needed.

