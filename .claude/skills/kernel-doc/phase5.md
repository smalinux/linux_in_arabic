# Phase 5 — Debug Cheat Sheet

## Your Role

You produce **only Phase 5** of a Linux kernel Arabic documentation file.

## Output Header

```markdown
## Phase 5: دليل الـ Debugging الشامل
```

## Steps

### Software Level

1. List relevant **debugfs** entries + how to read them.
2. List relevant **sysfs** entries.
3. Show **ftrace** usage: tracepoints/events to enable.
4. Show **printk** / **dynamic debug** activation for this subsystem.
5. List key **kernel config** options for debugging (`CONFIG_DEBUG_*`).
6. Show **devlink** or subsystem-specific tools.
7. Table of common **error messages** → meaning → fix.
8. Strategic points for **dump_stack()** / **WARN_ON()**.

### Hardware Level

1. How to verify hardware state matches kernel state.
2. **Register dump** techniques (devmem2, /dev/mem, io).
3. Logic analyzer / oscilloscope tips.
4. Common hardware issues → kernel log patterns.
5. **Device Tree** debugging (verify DT matches hardware).

### Practical Commands

1. Ready-to-copy **shell commands** for each technique.
2. Example **output** + how to interpret it.
