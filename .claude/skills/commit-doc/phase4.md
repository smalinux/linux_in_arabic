# Phase 4 — الخلاصة والـ Timeline

## Your Role

You produce **only Phase 4** of a Linux kernel commit documentation file.

## Output Header

```markdown
## Phase 4: الخلاصة — الصورة الكاملة
```

## Style

- Summary phase: synthesize everything. No new deep dives.
- Heavy use of tables, timelines, and diagrams.
- **ALL prose**: Egyptian Arabic.

## Steps

1. **الـ Timeline — خطوة بخطوة** (`### الـ Timeline بعد الـ fix`):
   - Draw a step-by-step ASCII timeline of the **fixed** sequence of operations.
   - Label every step with the real function/event name.
   - Show what changed compared to the broken old flow.
   - Example format:
     ```
     cpufreq requests 1 GHz
           │
           ▼
     clk_set_rate(sys_pll)
           │
           ├─ PRE_RATE_CHANGE → notifier fires
           │       └─ cpu_clk → XTAL (24 MHz, safe)
           │          udelay(100 µs)
           │
           ├─ PLL registers reprogrammed (glitchy — harmless now)
           │
           └─ POST_RATE_CHANGE → notifier fires
                   └─ cpu_clk → cpu_scale_out_sel (new freq)
     ```

2. **جدول الملخص** (`### جدول الملخص`):
   Build a markdown table with these exact columns:
   | الجانب | التفاصيل |
   Rows must include:
   - المشكلة اللي اتحلت
   - الـ Root Cause
   - الـ Mechanism المستخدم
   - الـ Files اللي اتغيرت
   - الـ Kernel APIs المستخدمة
   - الـ Hardware المتأثر
   - هل هو حل مؤقت؟ (FIXME?)
   - تاريخ الـ commit والـ author

3. **الـ Signed-off Chain** (`### الـ Review Chain`):
   - List every `Signed-off-by` and `Reviewed-by` from the commit message.
   - One line per person: name → دوره في الـ review (author / reviewer / maintainer).
   - Explain in Arabic what `Signed-off-by` and `Reviewed-by` mean in the Linux kernel process.

4. **إيه اللي تقراه بعد كده** (`### للفضوليين — اقرأ كمان`):
   - List 3-5 related commits, files, or kernel docs the reader should explore next.
   - Each entry: file or concept name → جملة واحدة بالعربي تقول ليه مهم.
   - If the commit message contains a mailing list link, mention it.

5. **جملة واحدة تلخص الـ commit** (`### الـ commit في جملة واحدة`):
   - Write a single Arabic sentence that captures the entire commit.
   - Format: bold, centered with `> **...**`
