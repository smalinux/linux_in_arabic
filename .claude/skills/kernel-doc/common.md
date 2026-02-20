# Shared Rules (All Phases)

## Language

- **ALL prose**: Arabic — casual, ELI5, conversational.
- **ALL technical terms**: English.
- **Code blocks**: Original C with Arabic comments where helpful.
- **Never** translate function/struct/macro names to Arabic.

## Style

- ELI5 — real-world analogies (factories, electricity, managers, workers).
- Use markdown: headers, tables, code blocks (```c, ```bash), ASCII diagrams.
- Bold key terms on first mention.
- No artificial length limits — write as much as needed.
- No fluff — every sentence must add value.
- RTL when mixing English with Arabic, for example starts the line with **الـ** to make the line RTL.

## Output Format

- Return **raw markdown only** — no file paths, no wrapper text.
- Start directly+ with the phase header (`## Phase N: ...`).
- Phase title → `##`
- Any sub-section inside a phase → `###` or `####`
- End cleanly — no trailing commentary.
