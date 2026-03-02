# Phase 1 — الصورة الكبيرة

## Your Role

You produce **only Phase 1** of a Linux kernel commit documentation file.

## Output Header

```markdown
## Phase 1: الصورة الكبيرة — فهم الـ commit من أوله
```

## Style

- ELI5 first — use a real-world analogy that a 5-year-old can picture.
- Then zoom out to the subsystem level.
- **ALL prose**: Egyptian Arabic.

## Steps

1. **Parse the commit message header** (subject line + body paragraph) from the diff file.
   Extract: author, date, subject, subsystem prefix (e.g., `clk:`, `mm:`, `net:`).

2. **Identify the subsystem**: look at the file paths in the stat summary. Map them to their
   kernel subsystem (e.g., `drivers/clk/` → Clock Framework, `mm/` → Memory Management).
   Check `/workspace/external/linux/MAINTAINERS` if needed to confirm the maintainer/subsystem.

3. **ELI5 Analogy section** (`### تخيل معايا...`):
   - Tell a concrete real-world story that captures exactly the bug/feature.
   - The story must map 1-to-1: what broke → what the fix does.
   - Avoid vague analogies. Be specific.

4. **Big Picture section** (`### الصورة الكبيرة`):
   - What subsystem/driver does this touch?
   - What was the existing behaviour before this commit?
   - What was broken, missing, or dangerous?
   - Why did it matter for end users / devices?

5. **The Hardware Context** (`### الـ Hardware`):
   - What chip/board/architecture is affected?
   - Where is it used in the real world (phones, TVs, servers, etc.)?
   - List the key hardware blocks involved (PLLs, clocks, memory controllers, etc.)
     with a one-line Arabic description each.

6. **Related files to know** (`### ملفات مهمة تعرفها`):
   - List the files changed (from stat) + 2-3 related files the reader should explore.
   - One line per file: path → what it does in Arabic.
