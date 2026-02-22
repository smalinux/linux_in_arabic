# Phase 2 — Subsystem / Framework Overview

## Your Role

You produce **only Phase 2** of a Linux kernel Arabic documentation file.
You are writing for an **embedded Linux engineer** who understands C, ARM hardware,
and basic kernel concepts — but needs to reach **kernel subsystem expert level**.

## Output Header

```markdown
## Phase 2: شرح الـ [Subsystem Name] Framework
```

## Steps

1. Explain the **problem** this subsystem solves — why does it exist?
2. Explain the **solution** — what approach does the kernel take?
3. Provide a **real-world analogy** that makes it click.
4. Draw the **big picture architecture** as ASCII diagram, examples:
   - Where does this subsystem sit in the kernel?
   - Consumers (drivers that use it)
   - Providers (hardware-specific implementations)
5. One **real-world analogy** — but go beyond surface level, map every part of the
  analogy to the actual kernel concept
6. The **core abstraction** this subsystem introduces — what is the central idea?
7. What does this subsystem **own** vs. What does it **delegate** to drivers?

## Style

1. Explain & go deeper than **Phase 1**.
2. No ELI5 — assume the reader knows C and basic kernel structure
3. Explain all hard concepts in great details.
4. Draw graphs to make sense how each struct fit together.
5. Draw whenever needed.
6. If a concept requires understanding another subsystem first, **name it explicitly**
  and give a one-line explanation before continuing
