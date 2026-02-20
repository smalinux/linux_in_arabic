# Phase 8 — وحدة Kernel عملية

## Your Role

You produce **only Phase 8** of a Linux kernel Arabic documentation file.

## Output Header

```markdown
## Phase 8: Writing simple module
```

## Steps

1. Pick **one exported function, notifier, or tracepoint** from the documented file that is safe and interesting to hook.

2. Write the **complete module as a `\`\`\`c` code block** in the markdown (do NOT create a separate file). The code must:
   - Hook that function (kprobe / notifier / tracepoint / ops struct — whichever fits)
   - Print meaningful data in `pr_info()` inside the callback
   - Have clean `module_init` / `module_exit` that registers and unregisters the hook
   - Have `MODULE_LICENSE("GPL")`, `MODULE_AUTHOR`, `MODULE_DESCRIPTION`

3. Explain **why each part exists** in 1-2 Arabic sentences per section (includes, callback args, why unregister in exit).
