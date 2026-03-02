# Shared Rules (All Phases)

## Language

- **ALL prose**: Egyptian Arabic — Very concise, very comprehensive.
- **ALL technical terms**: English (function names, struct names, macro names, subsystem names).
- **Code blocks**: Original C/diff with simple concise English inline comments where helpful.
- **Never** translate function/struct/macro/file names to Arabic.

## Style

- Real-world analogies — explain like to a smart person who doesn't know kernel internals.
- Use markdown: headers, tables, code blocks (```c, ```diff, ```bash), ASCII diagrams.
- Bold key terms on first mention.
- No artificial length limits — write as much as needed to be truly understood.
- No fluff — every sentence must add value.
- RTL when mixing English with Arabic: start the line with an Arabic word or **الـ** prefix
  so the line renders RTL (e.g., **الـ** `notifier_block` بتعمل كذا وكذا).
- When showing code from the diff, show the full added/changed block — don't truncate.

## Output Format

- Return **raw markdown only** — no file paths, no wrapper text.
- Start directly with the phase header (`## Phase N: ...`).
- Phase title → `##`
- Any sub-section inside a phase → `###` or `####`
- End cleanly — no trailing commentary.
