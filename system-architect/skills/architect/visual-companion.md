# Visual Companion Guide

A lightweight static HTML renderer for side-by-side comparisons during architecture sessions. Use it when seeing two or three options next to each other would be clearer than reading them sequentially in the terminal.

## When to Use

**Use the visual companion** when the question is visual — meaning the answer is genuinely clearer shown than described:

- Comparing two interface definitions or config shapes side by side
- Showing what the same call sequence looks like under two different designs
- Placing two architecture diagrams next to each other for direct comparison
- Showing a before/after for a proposed change

**Stay in the terminal** when the content is text or tabular:

- Clarifying questions
- Tradeoff lists or pros/cons
- Choosing between named options described in words
- Anything where a sentence or table is the right format

A topic that involves code is not automatically a visual question. "Should we use events or direct calls?" is a conceptual question — use the terminal. "Here's what the event handler interface looks like vs. the direct call interface" — that's visual, use the companion.

## Offering It

Do NOT offer the visual companion upfront. Wait until a comparison would genuinely benefit from side-by-side layout. When that moment comes, offer it as its own message:

> "This comparison might be easier to read side by side — I can write an HTML file you can open in your browser. It'll show the two options with syntax highlighting and diagrams rendered. Want me to?"

Wait for the user's response. If they decline, continue in the terminal and don't offer again unless they raise it.

## How to Write the File

When the user accepts, write a self-contained HTML file using `templates/visual-companion.template.html` as the base. Fill in:

- The title and topic label at the top
- Each option column: name, description, code block(s) and/or mermaid diagram, tradeoffs
- Use as many columns as there are options (2 is standard; 3 is the max before it gets cramped)

Write the file to:

```
<output-path>/topics/<topic>/visual/YYYY-MM-DD-<short-label>.html
```

Then tell the user:

> "Written to `<path>`. Open it in your browser — diagrams will render automatically. Let me know what you think."

Do not wait for them to open it before your next message. Continue the conversation.

## Per-Comparison Decision

Even after the user has accepted the companion, decide for each comparison whether to use it. Not every option-comparison needs an HTML file — short snippets of 5 lines or less are readable inline. Use the file when:

- Either option has more than ~10 lines of code
- You're comparing mermaid diagrams (they don't render in the terminal)
- There are three or more things to compare at once

## File Naming

Use semantic names with a short label for the comparison:

- `2026-06-25-interface-comparison.html`
- `2026-06-25-routing-approaches.html`
- `2026-06-25-pipeline-options.html`

Never reuse filenames for different comparisons. Each comparison gets its own file so the user has a record.

## CDN Dependencies

The template uses:

- **mermaid.js** (via jsDelivr CDN) — renders `<pre class="mermaid">` blocks automatically on load
- **highlight.js** (via cdnjs CDN) — syntax highlights `<code>` blocks automatically on load

These require an internet connection. Both are loaded from well-maintained CDNs and add no server-side dependencies to the plugin.
