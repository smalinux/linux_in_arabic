# Phase 2 — المشكلة والـ Root Cause

## Your Role

You produce **only Phase 2** of a Linux kernel commit documentation file.

## Output Header

```markdown
## Phase 2: المشكلة — ليه الكود القديم كان بيتعطل؟
```

## Style

- Think like a detective reconstructing the crime scene.
- Show the **before** state clearly, then explain exactly why it broke.
- **ALL prose**: Egyptian Arabic.

## Steps

1. **قبل الـ commit — الكود القديم** (`### الكود قبل التعديل`):
   - Use `git show <parent>:<file>` (via Bash) to read the relevant sections of the file
     **before** this commit was applied.
   - Show the key old code in a ```c block.
   - Annotate with English inline comments explaining what each part did.

2. **شجرة الـ clocks / structs / flow** (`### الـ Architecture قبل الـ fix`):
   - Draw an ASCII diagram of the clock tree / call chain / data flow as it existed before.
   - Label every node with its real name from the code.
   - Show where the chain broke or the dangerous operation happened.

3. **الـ Root Cause** (`### ليه كان بيحصل crash / lockup / bug؟`):
   - Step by step, trace exactly what sequence of events triggered the bug.
   - Use numbered steps, be precise.
   - Name the exact register, function, or race condition responsible.
   - If hardware behaviour is involved (e.g. PLL glitch), explain what the hardware
     does at the electrical/digital level that caused the problem.

4. **تأثير المشكلة على المستخدم** (`### إيه اللي كان بيحصل للمستخدم؟`):
   - What did the user / device experience? (freeze, crash, wrong frequency, corruption?)
   - Was it deterministic or intermittent?
   - Was there a workaround before this fix?
