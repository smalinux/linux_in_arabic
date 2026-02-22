# Shared Rules (All Phases)

## Language

- **ALL prose**: Arabic — Very concise, very comprehensive.
- **ALL technical terms**: English.
- **Code blocks**: Original C with Simple concise English comments where helpful.
- **Never** translate function/struct/macro names to Arabic.

## Style

- Real-world examples.
- Use markdown: headers, tables, code blocks (```c, ```bash), ASCII diagrams.
- Bold key terms on first mention.
- No artificial length limits — write as much as needed.
- No fluff — every sentence must add value.
- RTL when mixing English with Arabic, for example starts the line with **الـ** to make the line RTL specially when first word in the sentence is English.

## Output Format

- Return **raw markdown only** — no file paths, no wrapper text.
- Start directly+ with the phase header (`## Phase N: ...`).
- Phase title → `##`
- Any sub-section inside a phase → `###` or `####`
- End cleanly — no trailing commentary.
