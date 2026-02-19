---
name: kernel-doc
description: "Parallel Arabic documentation for Linux kernel files — 7 agents, one per phase"
argument-hint: "path/to/kernel/file.c (relative to ./external/linux)"
---

# Kernel-Doc Orchestrator (Team Leader)

You are the **orchestrator**. You do NOT write documentation yourself. You spawn 7 parallel agents and merge their output.

## Execution Steps

### Step 1 — Parse & Read

1. Parse input file paths from arguments (relative to `./external/linux/`).
2. Read each input file completely using the Read tool.
3. Read the **key header files** that the source file `#include`s (especially from `include/linux/`).
4. Read **all 7 phase instruction files** from `.claude/skills/kernel-doc/phase*.md`.
5. Read the **shared rules** from `.claude/skills/kernel-doc/common.md`.

### Step 2 — Build Context Block

Assemble a `CONTEXT` string containing:
```
FILE_PATH: <path>
FILE_CONTENT: <full source code>
HEADER_FILES: <related headers content>
```

This context is passed to every agent.

### Step 3 — Spawn 7 Parallel Agents

Use the **Task tool** to launch **7 agents simultaneously** in a single message (all in parallel). Each agent:
- **subagent_type**: `general-purpose`
- **prompt**: `COMMON_RULES + PHASE_N_INSTRUCTIONS + CONTEXT`
- **description**: `"kernel-doc phase N"`

Each agent returns its phase content as **raw markdown** (no file writing).

### Step 4 — Merge & Write

1. `mkdir -p` the output directory.
2. Concatenate all 7 phase outputs in order (Phase 1 → Phase 7) into one markdown file.
3. Output path: mirror the input path with `.md` appended.
   - `drivers/clk/clk.c` → `drivers/clk/clk.c.md`

### Step 5 — Report

Print: output file path + one-line summary per phase.

## Rules

- **NEVER** write phases yourself — always delegate to agents.
- **ALL 7 agents** must be spawned in a **single message** (parallel).
- If an input has multiple files, process each file's 7 phases in parallel.
- If an agent fails, retry that single agent — don't redo all.
