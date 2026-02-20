---
name: kernel-doc
description: "Parallel Arabic documentation for Linux kernel files — multiple agents, one per phase"
argument-hint: "path/to/kernel/file.c (relative to ./external/linux)"
---

# Kernel-Doc Orchestrator (Team Leader)

You are the **orchestrator**. You do NOT write documentation yourself.
You spawn multiple parallel agents, one per phase, each writing its output to a temp file.
Then You then merge with `cat`.

> **PERMISSION REQUIREMENT**: This skill requires agents to have `Write` and `Bash` tool access.
> If agents report they cannot write files, ask the user to allow Write/Bash permissions for subagents.

---

## Execution Steps

### Step 1 — Parse Input

1. Parse input file paths from arguments (relative to `./external/linux/`).
2. Do **NOT** read the source files or headers yourself. Agents will read what they need.
3. Read **all phase instruction files** from `.claude/skills/kernel-doc/phase*.md`.
4. Read the **shared rules** from `.claude/skills/kernel-doc/common.md`.
   - Store these as `COMMON_RULES` and `PHASE_N_INSTRUCTIONS` strings.

### Step 2 — Prepare Temp Directory

```bash
export KDOC_TMP=$(mktemp -d /tmp/kernel-doc-XXXXXX) && echo $KDOC_TMP
```

Save the printed path for use in agent prompts.

### Step 3 — Spawn N Parallel Agents

Use the **Task tool** to launch **all N agents simultaneously** in a **single message** (parallel).

Each agent gets:
- **subagent_type**: `general-purpose`
- **description**: `"kernel-doc phase N"`
- **prompt** must contain:

```
COMMON_RULES:
<paste common.md content>

PHASE_INSTRUCTIONS:
<paste phase-N.md content>

SOURCE_FILE_PATH: external/linux/<path>

YOUR TASK:
1. Read the source file at SOURCE_FILE_PATH using the Read tool.
2. Read any key headers it #includes from include/linux/ that define structs/types used in the file (max 3 headers, skip if over 300 lines each).
3. Produce your phase output following PHASE_INSTRUCTIONS and COMMON_RULES.
4. Write your output to: <KDOC_TMP>/phase-N.md using the Write tool.
5. Reply with only: "phase-N done" — do NOT return the markdown content as text.
```

> **Why agents read files themselves**: This avoids embedding the source file 8× in agent prompts, saving thousands of input tokens and keeping the orchestrator context small.

### Step 4 — Merge & Write

After all agents confirm done:

1. `mkdir -p` the output directory (mirroring the input path structure).
2. Concatenate:
   ```bash
   cat <KDOC_TMP>/phase-1.md \
       <KDOC_TMP>/phase-2.md \
       <KDOC_TMP>/phase-N.md \
       > <output-path>
   ```
3. Output path: mirror the input path with `.md` appended.
   - `drivers/clk/clk.c` → `drivers/clk/clk.c.md`
4. Clean up: `rm -rf <KDOC_TMP>`

### Step 5 — Fallback (if agents lack Write permission)

If agents return their content as text instead of writing to files:
1. Extract each agent's markdown content.
2. Run ONE bash command that writes all phases to the output file at once:
   ```bash
   cat > <output-path> << 'ENDOFDOC'
   <phase1 content>
   <phase2 content>
   ...
   ENDOFDOC
   ```
3. This keeps the merge in a single tool call rather than 8 separate writes.

### Step 6 — Report

Print: output file path + one-line summary per phase.

---

## Rules

- **NEVER** write phases yourself — always delegate to agents.
- **ALL N agents** must be spawned in a **single message** (parallel).
- **NEVER embed source file content in agent prompts** — pass the path only.
- If an agent fails, retry that single agent — don't redo all.
- Agents must **write to temp files** and return only short confirmations.
