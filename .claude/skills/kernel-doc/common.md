# Shared Rules (All Phases)

## Language

- **ALL prose**: Arabic — casual, ELI5, conversational.
- **ALL technical terms**: English — struct names, function names, macros, subsystem names, kernel terms (spinlock, probe, devm, device tree).
- **Code blocks**: Original C with Arabic comments where helpful.
- **Never** translate function/struct/macro names to Arabic.

## Style

- ELI5 — real-world analogies (factories, electricity, managers, workers).
- Use markdown: headers, tables, code blocks (```c, ```bash), ASCII diagrams.
- Bold key terms on first mention.
- No artificial length limits — write as much as needed.
- No fluff — every sentence must add value.

## Output Format

- Return **raw markdown only** — no file paths, no wrapper text.
- Start directly with the phase header (`## Phase N: ...`).
- End cleanly — no trailing commentary.
