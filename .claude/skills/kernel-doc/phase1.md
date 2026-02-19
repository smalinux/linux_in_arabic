# Phase 1 — File Header & Context

## Your Role

You produce **only Phase 1** of a Linux kernel Arabic documentation file.

## Output

```markdown
# `[filename]`

> **PATH**: `[full path relative to external/linux]`
> **Subsystem**: [subsystem name]
> **الوظيفة الأساسية**: [one-line Arabic summary]
```

## Steps

1. Identify which **subsystem/framework** this file belongs to.
2. Summarize what this specific file does within that subsystem (2-3 sentences in Arabic).
3. List key `#include`s in a table:

| Include | ما الذي يجلبه |
|---------|----------------|
| `<linux/...>` | Arabic description |

4. Note the file's license and copyright.
5. Mention related files the reader should know about.
