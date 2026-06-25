# System Architect Plugin

An interactive system architecture sounding board. Use it when you have an idea or a problem and need a thinking partner to work through design options — one that will push back on over-engineering, explore alternatives, and help you land on the simplest approach that actually solves the problem.

Produces three types of documentation to support ongoing conversations across multiple sessions.

## Skills

### `/architect`

Starts a brainstorming session on a system design problem. Works conversationally: asks clarifying questions one at a time, proposes 2-3 approaches with tradeoffs, pushes back on unnecessary complexity, embeds mermaid diagrams in design docs, and writes documentation at the end of every session.

Examples of what this is good for:
- Making an existing system more pluggable so it's easier to extend
- Designing self-service deployment systems for non-technical users
- Approaches to API versioning given an existing architecture
- Self-service data pipeline and data platform design

## Usage

```
/architect
/architect --topic api-versioning
/architect --resume docs/architecture/topics/plugin-system/notes.md
/architect --context services/ingest/README.md
/architect --topic api-versioning --context docs/architecture/topics/current-api/notes.md
```

### CLAUDE.md Configuration

Add a **System Architect** section to your project's `CLAUDE.md` to configure where design docs are written — no argument needed on every invocation.

```markdown
## System Architect
- output-path: docs/architecture/
```

| Key | Default | Used by |
|-----|---------|---------|
| `output-path` | `docs/architecture/` | `/architect` — base path for topics, decisions, and session logs |

Precedence: per-invocation argument > `CLAUDE.md` value > hardcoded default.

## Output Files

Every session writes a session log, regardless of whether the design is complete. Design docs are only written once the user approves.

| Type | Path | When written |
|------|------|--------------|
| **Session log** | `<output-path>/sessions/YYYY-MM-DD-<topic>.md` | Every session — appended, not overwritten |
| **Topic doc** | `<output-path>/<topic>.md` | Open explorations with no final decision |
| **ADR** | `<output-path>/decisions/YYYY-MM-DD-<title>.md` | Decisions that have been made and committed to |

The skill asks at the start of each session whether you're converging on a decision (→ ADR) or exploring the space (→ topic doc). Topic docs are updated in place across sessions; ADRs are immutable once written.

## Agents

### `architect-evaluator`

Stress-tests a proposed design from three angles:

1. **Complexity** — is this the simplest thing that solves the problem? What could be cut?
2. **Failure modes** — where is this likely to break under realistic conditions?
3. **Assumptions** — what does the design rely on being true that might not be?

Returns a structured "Hold or Revisit" verdict. Spawned by `/architect` when you accept the offer to run a stress test. Not intended for direct invocation.

## Design Philosophy

**Simple first.** The goal is the smallest design that solves the stated problem. Complexity requires justification; simplicity does not. A new service has to be built, deployed, monitored, and operated. A new abstraction has to be understood and maintained by everyone who touches it.

**One question at a time.** The skill won't stack questions. Each turn has one question so you can think clearly about each decision.

**Pushback is the point.** The skill will challenge over-engineered approaches directly. If the pushback is wrong, say so — that's useful too. The conversation should be adversarial in the right way.

**Documentation by default.** Every session leaves a trace. Session logs mean you can always pick up where you left off, even weeks later.

## Mermaid Diagrams

Diagrams are embedded directly in design docs as mermaid code blocks, renderable in any IDE or markdown tool that supports mermaid (VS Code, JetBrains, Obsidian, GitHub, etc.). No server or additional tooling required.

The skill uses diagrams when spatial or temporal relationships benefit from visualization — architecture overviews, request flows, state machines, data models. Text questions stay as text.
