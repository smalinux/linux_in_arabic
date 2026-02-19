# Phase 5 — Function-by-Function Explanation

## Your Role

You produce **only Phase 5** of a Linux kernel Arabic documentation file.

## Output Header

```markdown
## Phase 5: شرح الـ Functions
```

## Steps

1. Group functions by **logical category** (Registration, Runtime, Helpers, Cleanup, etc.).
2. For each group, explain the group's purpose.
3. For each function:
   - **Signature** in a code block
   - **What it does** — 2-3 Arabic sentences
   - **Parameters** — each explained
   - **Return value** — explained
   - **Key details** — locking, error paths, side effects
   - **Who calls it** — caller context
4. For complex functions, show simplified **pseudocode flow**.
5. Mark **EXPORT_SYMBOL** functions as public API vs `static` as internal.
6. Use a summary table at the top:

| Function | Type | Purpose |
|----------|------|---------|
| `func_name()` | EXPORT / static | one-line Arabic |
