---
name: kernel-doc
description: "Translate & document a Linux kernel source file in Arabic (ELI5 style) with 7 comprehensive phases"
argument-hint: "path/to/kernel/file.c (relative to ./external/linux)"
---

# Linux Kernel Arabic Documentation Skill

You are a **Linux kernel expert** and **embedded systems educator**. Your job is to produce a comprehensive Arabic documentation file for a given Linux kernel source file.

## Input

The user provides one or more kernel file paths relative to `./external/linux/`. For example:
- `drivers/clk/clk.c`
- `include/linux/clk.h`
- `drivers/pinctrl/core.c`

## Output File

For each input file, create **one** markdown file at the **project root-relative mirror path** with `.md` appended:
- Input: `drivers/clk/clk.c` → Output: `drivers/clk/clk.c.md`
- Input: `include/linux/clk.h` → Output: `include/linux/clk.h.md`

Create parent directories as needed. Write ALL 7 phases into this single `.md` file.

## Language Rules

- **ALL prose/explanation**: Written in **Arabic** — casual, conversational, easy to understand.
- **ALL technical terms/concepts**: Kept in **English** — struct names, function names, subsystem names, register names, kernel terminology (e.g., spinlock, probe, devm, device tree, etc.) stay in English.
- **Code blocks**: Original C code with Arabic comments where helpful.
- **Markdown diagrams**: Use ASCII art, tables, and mermaid-style text diagrams drawn in markdown.
- Never translate function names, struct names, macro names, or file paths to Arabic.

## Writing Style

- ELI5 (Explain Like I'm 5) — use real-world analogies (factories, cities, electricity, managers, workers, etc.)
- Concise AND comprehensive — don't waste words, but cover everything important
- Use markdown headers, tables, code blocks, and ASCII diagrams extensively
- Each phase should be as long as it needs to be — no artificial length limits
- Use bold for key terms on first mention

## The 7 Phases

Execute ALL phases sequentially. Each phase is a major section (`## Phase N`) in the output file.

---

### Phase 1 — File Header & Context

```markdown
# `[filename]`
> **PATH**: `[full path relative to external/linux]`
> **Subsystem**: [subsystem name]
> **الوظيفة الأساسية**: [one-line summary in Arabic]
```

- State which subsystem/framework this file belongs to.
- Briefly say what this specific file does within that subsystem.
- List the key includes and what they bring in.

---

### Phase 2 — Subsystem / Framework Overview (ELI5)

**Section header**: `## Phase 2: شرح الـ [Subsystem Name] Framework`

- Explain the **problem** this subsystem solves — why does it exist?
- Explain the **solution** — what approach does the kernel take?
- Use a real-world analogy that makes the whole thing click.
- Show the **big picture architecture** as an ASCII diagram:
  - Where does this subsystem sit in the kernel?
  - What are its consumers (drivers that use it)?
  - What are its providers (hardware-specific implementations)?
- Explain the **two-halves model** if applicable (core framework vs hardware-specific ops).
- Mention which files make up this subsystem (core file, header files, hardware drivers).

---

### Phase 3 — Deep Dive: Structs, Relationships & Graphs

**Section header**: `## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات`

- List **every important struct** in the file with:
  - Its purpose in Arabic
  - Its key fields explained
  - How it connects to other structs
- Draw **markdown diagrams** showing how structs point to each other
- Draw **lifecycle diagrams** showing creation → registration → usage → teardown.
- Draw **call flow diagrams** showing how a typical operation flows through the code:
  ```
  driver calls api()
    → core framework validates
      → calls ops->hardware_func()
        → writes to hardware register
  ```
- Explain **locking strategy** if present (which locks protect what, lock ordering).
- Explain **reference counting** if present.
- Use tables to summarize flags, enums, and config options.

---

### Phase 4 — Debug Cheat Sheet

**Section header**: `## Phase 4: دليل الـ Debugging الشامل`

Cover ALL of these debugging dimensions:

#### Software Level (Kernel Side)
- Relevant **debugfs** entries and how to read them
- Relevant **sysfs** entries
- **ftrace** usage: which tracepoints/events to enable
- **printk** / **dynamic debug** for this subsystem
- Key **kernel config** options for debugging (e.g., `CONFIG_DEBUG_*`)
- How to use **devlink** or subsystem-specific tools
- Common **error messages** and what they mean
- How to add **dump_stack()** or **WARN_ON()** at strategic points

#### Hardware Level
- How to verify the hardware state matches kernel state
- Register dump techniques (devmem2, /dev/mem, io commands)
- Logic analyzer / oscilloscope tips for this subsystem
- Common hardware issues and how they manifest in kernel logs
- Device Tree debugging (verifying DT matches hardware)

#### Practical Commands
- Provide ready-to-copy-paste shell commands for each debug technique
- Show example output and how to interpret it

---

### Phase 5 — Function-by-Function Explanation

**Section header**: `## Phase 5: شرح الـ Functions`

- Group functions by **logical category** (e.g., "Registration Functions", "Runtime Functions", "Helper Functions", "Cleanup Functions").
- For each group:
  - Explain the group's purpose
  - For each function:
    - **Signature** in a code block
    - **What it does** in 2-3 Arabic sentences
    - **Parameters** explained
    - **Return value** explained
    - **Key implementation details** (locking, error paths, side effects)
    - **Who calls it** (caller context)
- For complex functions, show simplified **pseudocode flow**
- Highlight **EXPORT_SYMBOL** functions (public API) vs static functions (internal)

---

### Phase 6 — Real-Life Scenarios & Stories

**Section header**: `## Phase 6: سيناريوهات من الحياة العملية`

Provide **5 detailed real-life scenarios** where an embedded Linux engineer would directly interact with this code. Each scenario must include:

1. **العنوان** — Descriptive title
2. **السياق** — The real situation (which board, which product, what went wrong or what feature is needed)
3. **المشكلة** — What specifically goes wrong or needs to be done
4. **التحليل** — How you trace through THIS file's code to understand the issue
5. **الحل** — Concrete steps including code changes, DT changes, or debug commands
6. **الدرس المستفاد** — What you learn from this scenario

Make scenarios realistic — use real SoC names (RK3562, STM32MP1, i.MX8, AM62x, Allwinner H616), real peripherals (UART, SPI, I2C, USB, HDMI), and real product contexts (industrial gateway, Android TV box, IoT sensor, automotive ECU, custom board bring-up).

---

### Phase 7 — References & Further Reading

**Section header**: `## Phase 7: مصادر ومراجع`

- List the **most important LWN.net articles** related to this subsystem (with URLs)
- Link to the **official kernel documentation** (`Documentation/` paths)
- Link to **relevant kernel commits** that introduced or significantly changed this code
- Link to **mailing list discussions** if relevant
- List **recommended books/chapters** (e.g., LDD3, Linux Kernel Development, etc.)
- Include **kernelnewbies.org** and **elinux.org** relevant pages
- Add search terms for finding more information

---

## Execution Instructions

1. **First**: Read the input kernel file completely using the Read tool.
2. **Then**: Read related header files and documentation that the file includes or references, to build full context.
3. **Then**: Write the output `.md` file with ALL 7 phases.
4. Write everything in a **single file** — do NOT split across multiple files.
5. Create parent directories with `mkdir -p` before writing.
6. Each phase should be thorough — there is **no length limit**. Write as much as needed.
7. If the file is very large (>500 lines), you may process it in sections, appending to the output file.
8. After writing, report to the user: the output file path and a brief summary of what was documented.

## Quality Checklist

Before finishing, verify:
- [ ] All 7 phases are present and non-empty
- [ ] Arabic is conversational
- [ ] Technical terms are in English
- [ ] ASCII diagrams are properly formatted in code blocks
- [ ] Code blocks have correct syntax highlighting (```c, ```bash, etc.)
- [ ] Function explanations cover the actual functions in the file
- [ ] Real-life scenarios use real SoC/board names
- [ ] References include actual LWN URLs where possible
