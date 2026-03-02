---
name: commit-doc
description: "Parallel Arabic documentation for Linux kernel commits — multiple agents, one per phase"
argument-hint: "<commit-hash>"
---

# Commit-Doc Orchestrator

You are the **orchestrator**. You do NOT write documentation yourself.
You spawn 4 parallel agents, one per phase, each writing its output to a temp file.
Then you merge them with `cat`.

> **PERMISSION REQUIREMENT**: This skill requires agents to have `Write` and `Bash` tool access.
> If agents report they cannot write files, ask the user to allow Write/Bash permissions for subagents.

---

## Execution Steps

### Step 1 — Parse Input

1. Extract the **commit hash** from the skill argument.
2. Read **all phase instruction files** from `.claude/skills/commit-doc/phase*.md`.
3. Read **shared rules** from `.claude/skills/commit-doc/common.md`.
   - Store as `COMMON_RULES` and `PHASE_N_INSTRUCTIONS` strings.

### Step 2 — Capture Commit Data

Run these commands inside `/workspace/external/linux`:

```bash
export CDOC_TMP=$(mktemp -d /tmp/commit-doc-XXXXXX) && echo $CDOC_TMP

cd /workspace/external/linux

# Full diff + message
git show <hash> > $CDOC_TMP/commit.diff

# Stat summary (files changed, insertions, deletions)
git show <hash> --stat > $CDOC_TMP/commit.stat

# Parent commit hash (for before/after file reads)
git show <hash> --no-patch --format="%P" > $CDOC_TMP/parent.txt
```

Save `$CDOC_TMP` path for use in agent prompts.

### Step 3 — Spawn 4 Parallel Agents

Use the **Agent tool** to launch **all 4 agents simultaneously** in a **single message** (parallel).

Each agent gets:
- **subagent_type**: `general-purpose`
- **description**: `"commit-doc phase N"`
- **prompt** must contain:

```
COMMON_RULES:
<paste common.md content>

PHASE_INSTRUCTIONS:
<paste phase-N.md content>

COMMIT_HASH: <hash>
LINUX_REPO: /workspace/external/linux
CDOC_TMP: <CDOC_TMP path>

YOUR TASK:
1. Read the full commit diff from: <CDOC_TMP>/commit.diff
2. Read the stat summary from: <CDOC_TMP>/commit.stat
3. If your phase requires reading source files, check out lines from the changed
   files in /workspace/external/linux using the Read tool (the working tree
   reflects HEAD, use `git show <parent>:<file>` via Bash for the before-state).
4. Produce your phase output following PHASE_INSTRUCTIONS and COMMON_RULES.
5. Write your output to: <CDOC_TMP>/phase-N.md using the Write tool.
6. Reply with only: "phase-N done" — do NOT return the markdown content as text.
```

### Step 4 — Merge & Write

After all 4 agents confirm done:

1. `mkdir -p /workspace/commits`
2. Concatenate:
   ```bash
   cat <CDOC_TMP>/phase-1.md \
       <CDOC_TMP>/phase-2.md \
       <CDOC_TMP>/phase-3.md \
       <CDOC_TMP>/phase-4.md \
       > /workspace/commits/<hash>.md
   ```
3. Clean up: `rm -rf <CDOC_TMP>`

### Step 5 — Fallback (if agents lack Write permission)

If agents return their content as text instead of writing to files:
1. Extract each agent's markdown content.
2. Run ONE bash command that writes all phases to the output file at once:
   ```bash
   cat > /workspace/commits/<hash>.md << 'ENDOFDOC'
   <phase1 content>
   <phase2 content>
   <phase3 content>
   <phase4 content>
   ENDOFDOC
   ```

### Step 6 — Report

Print: output file path + one-line summary per phase.

---

## Rules

- **NEVER** write phases yourself — always delegate to agents.
- **ALL 4 agents** must be spawned in a **single message** (parallel).
- **NEVER embed commit diff content in agent prompts** — pass the temp file paths only.
- If an agent fails, retry that single agent — don't redo all.
- Agents must **write to temp files** and return only short confirmations.
