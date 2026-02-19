# `clk-provider.h`
> **PATH**: `include/linux/clk-provider.h`
> **Subsystem**: Common Clock Framework (CCF)
> **الوظيفة الأساسية**: الـ header الرئيسي لكتابة drivers الـ clocks — يحتوي على كل الـ structs والـ ops وطرق التسجيل اللي يحتاجها أي مبرمج يريد إضافة clock جديد للـ kernel

---

## Phase 2: شرح الـ Common Clock Framework (CCF)

### المشكلة: فوضى الـ Clocks

تخيل إنك بتبني مدينة (= SoC) فيها آلاف الأجهزة (CPU، GPU، USB، UART، SPI...). كل جهاز يحتاج كهرباء بـ"تردد" معين — يعني clock. قبل الـ CCF، كل SoC كان له كود خاص بيه لإدارة الـ clocks، مفيش كود مشترك، مفيش توحيد. النتيجة؟ فوضى تامة وكود متكرر في كل مكان.

### الحل: CCF — المدير المركزي للـ Clocks

الـ CCF (Common Clock Framework) ظهر في kernel 3.4 ليكون **المدير المركزي** لكل الـ clocks في النظام. الفكرة بسيطة:

- **Core Framework** (`drivers/clk/clk.c`): المدير العام — يعرف كيف يتعامل مع أي clock بشكل عام (enable/disable/set_rate/get_rate)
- **Hardware Drivers** (`drivers/clk/clk-xxx.c`): الفنيون المتخصصون — كل واحد يعرف تفاصيل الـ hardware بتاعه
- **`clk-provider.h`**: عقد العمل — يحدد "الواجهة" اللي لازم كل hardware driver يطبقها

### تشبيه من الحياة

الـ CCF زي شركة كهرباء:
- **المحطة المركزية** = `clk.c` (Core)
- **المولدات المختلفة** = hardware clock drivers (PLL، divider، gate...)
- **لوحة التحكم** = `clk-provider.h` (الـ interface)
- **المستهلكون** = drivers اللي تستخدم `clk_get()` و`clk_enable()`

### البنية الكبيرة (Big Picture Architecture)

```
┌─────────────────────────────────────────────────────────────┐
│                    Consumer Drivers                          │
│         (UART, SPI, I2C, USB, GPU, display, etc.)           │
│              clk_get() / clk_enable() / clk_set_rate()      │
└──────────────────────┬──────────────────────────────────────┘
                       │  Consumer API (include/linux/clk.h)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Common Clock Framework Core                     │
│                   drivers/clk/clk.c                          │
│   - Tree management (parent/child relationships)             │
│   - Rate negotiation & propagation                           │
│   - Enable/disable reference counting                        │
│   - Locking (prepare_lock mutex + enable_lock spinlock)      │
└──────────────────────┬──────────────────────────────────────┘
                       │  Provider API (include/linux/clk-provider.h)
                       │  clk_hw_register() / clk_ops callbacks
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Hardware Clock Drivers (Providers)              │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  clk_fixed   │   clk_gate   │  clk_divider │   clk_mux      │
│  _rate       │              │              │                 │
├──────────────┴──────────────┴──────────────┴────────────────┤
│         SoC-specific drivers: drivers/clk/clk-rk3562.c      │
│                              drivers/clk/clk-imx8.c         │
│                              drivers/clk/clk-stm32mp1.c     │
└──────────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Real Hardware                              │
│         PLL circuits, clock gates, dividers, muxes           │
└─────────────────────────────────────────────────────────────┘
```

### النموذج الثنائي (Two-Halves Model)

```
┌─────────────────┐         ┌─────────────────┐
│   struct clk    │◄────────│  struct clk_hw  │
│  (consumer side)│         │ (provider side) │
│  - opaque handle│         │ - links to core │
│  - per-user     │         │ - links to hw   │
└─────────────────┘         └────────┬────────┘
                                     │
                            ┌────────▼────────┐
                            │ struct clk_core │
                            │  (internal,     │
                            │   hidden in     │
                            │   clk.c)        │
                            └─────────────────┘
```

### الملفات المكونة للـ Subsystem

| الملف | الدور |
|-------|-------|
| `include/linux/clk-provider.h` | Provider API — هذا الملف |
| `include/linux/clk.h` | Consumer API |
| `drivers/clk/clk.c` | Core framework implementation |
| `drivers/clk/clk-fixed-rate.c` | Fixed rate clock |
| `drivers/clk/clk-gate.c` | Gate clock |
| `drivers/clk/clk-divider.c` | Divider clock |
| `drivers/clk/clk-mux.c` | Mux clock |
| `drivers/clk/clk-fixed-factor.c` | Fixed factor clock |
| `drivers/clk/clk-composite.c` | Composite clock |
| `drivers/clk/clk-fractional-divider.c` | Fractional divider |

---

## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### 1. `struct clk_hw` — جسر الـ Provider مع الـ Core

```c
struct clk_hw {
    struct clk_core *core;        // ← يشير للـ internal core object
    struct clk *clk;              // ← per-user handle
    const struct clk_init_data *init; // ← بيانات التهيئة (NULL بعد التسجيل)
};
```

**الغرض**: هذا الـ struct هو "الجسر" بين كود الـ hardware الخاص بك والـ core framework. كل hardware driver يضع `struct clk_hw` كأول عضو في الـ struct الخاص به:

```c
// مثال: driver يعرف gate clock
struct clk_gate {
    struct clk_hw hw;   // ← الجسر دائماً أول عضو
    void __iomem *reg;
    u8 bit_idx;
    ...
};
```

بهذه الطريقة يمكن الانتقال من `clk_hw*` لـ `clk_gate*` باستخدام `container_of()`.

---

### 2. `struct clk_ops` — الـ Virtual Table (vtable)

هذا هو **قلب الـ Provider API**. تخيله كعقد مع المدير: "أنا hardware driver، وأعدك بتنفيذ هذه الوظائف":

```c
struct clk_ops {
    // === مجموعة الـ Prepare/Unprepare (sleepable context) ===
    int  (*prepare)(struct clk_hw *hw);
    void (*unprepare)(struct clk_hw *hw);
    int  (*is_prepared)(struct clk_hw *hw);
    void (*unprepare_unused)(struct clk_hw *hw);

    // === مجموعة الـ Enable/Disable (atomic context) ===
    int  (*enable)(struct clk_hw *hw);
    void (*disable)(struct clk_hw *hw);
    int  (*is_enabled)(struct clk_hw *hw);
    void (*disable_unused)(struct clk_hw *hw);

    // === مجموعة Power Management ===
    int  (*save_context)(struct clk_hw *hw);
    void (*restore_context)(struct clk_hw *hw);

    // === مجموعة Rate ===
    unsigned long (*recalc_rate)(struct clk_hw *hw, unsigned long parent_rate);
    long          (*round_rate)(struct clk_hw *hw, unsigned long rate,
                                unsigned long *parent_rate);
    int           (*determine_rate)(struct clk_hw *hw,
                                    struct clk_rate_request *req);
    int           (*set_rate)(struct clk_hw *hw, unsigned long rate,
                               unsigned long parent_rate);
    int           (*set_rate_and_parent)(struct clk_hw *hw, ...);

    // === مجموعة Parent ===
    int (*set_parent)(struct clk_hw *hw, u8 index);
    u8  (*get_parent)(struct clk_hw *hw);

    // === مجموعة Accuracy ===
    unsigned long (*recalc_accuracy)(struct clk_hw *hw,
                                      unsigned long parent_accuracy);

    // === مجموعة Phase & Duty Cycle ===
    int (*get_phase)(struct clk_hw *hw);
    int (*set_phase)(struct clk_hw *hw, int degrees);
    int (*get_duty_cycle)(struct clk_hw *hw, struct clk_duty *duty);
    int (*set_duty_cycle)(struct clk_hw *hw, struct clk_duty *duty);

    // === مجموعة Init/Debug ===
    int  (*init)(struct clk_hw *hw);
    void (*terminate)(struct clk_hw *hw);
    void (*debug_init)(struct clk_hw *hw, struct dentry *dentry);
};
```

**الفرق المهم بين prepare/enable:**

```
┌─────────────────────────────────────────────────────────────┐
│  prepare()  — يعمل في sleepable context (يمكنه النوم)       │
│  ───────────────────────────────────────────────────────     │
│  • إعداد الـ regulators                                      │
│  • كتابة registers تحتاج delay                               │
│  • تهيئة الـ PLL قبل تشغيله                                  │
│                                                              │
│  enable()   — يعمل في atomic context (لا ينام أبداً)         │
│  ───────────────────────────────────────────────────────     │
│  • تشغيل الـ gate register                                   │
│  • spinlock محمي                                             │
│  • من interrupt handler مباشرةً                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 3. `struct clk_init_data` — بيانات التسجيل

```c
struct clk_init_data {
    const char *name;                    // اسم الـ clock (فريد في النظام)
    const struct clk_ops *ops;           // الـ vtable
    // واحد فقط من الثلاثة التالية:
    const char * const *parent_names;    // أسماء الـ parents (legacy)
    const struct clk_parent_data *parent_data; // بيانات الـ parents (حديث)
    const struct clk_hw **parent_hws;    // مؤشرات مباشرة للـ parents (الأفضل)
    u8 num_parents;                      // عدد الـ parents
    unsigned long flags;                 // CLK_SET_RATE_PARENT, إلخ
};
```

**ملاحظة مهمة**: بعد `clk_hw_register()` تصبح `init` = NULL. الـ core ينسخ البيانات ولا يحتاج للـ struct بعدها.

---

### 4. `struct clk_parent_data` — بيانات الـ Parent

```c
struct clk_parent_data {
    const struct clk_hw *hw;     // مؤشر مباشر (للـ clocks الداخلية)
    const char *fw_name;         // اسم في الـ device tree (مثل "mclk")
    const char *name;            // اسم عالمي (fallback)
    int index;                   // index في الـ DT (إذا لم يكن fw_name)
};
```

---

### 5. `struct clk_rate_request` — طلب تغيير الـ Rate

```c
struct clk_rate_request {
    struct clk_core *core;           // الـ clock المعني
    unsigned long rate;              // الـ rate المطلوب
    unsigned long min_rate;          // الحد الأدنى
    unsigned long max_rate;          // الحد الأقصى
    unsigned long best_parent_rate;  // أفضل rate للـ parent
    struct clk_hw *best_parent_hw;   // أفضل parent
};
```

---

### 6. الـ Basic Clock Types

#### `struct clk_fixed_rate` — clock بتردد ثابت

```c
struct clk_fixed_rate {
    struct clk_hw hw;
    unsigned long fixed_rate;      // التردد الثابت (مثل 24000000 = 24 MHz)
    unsigned long fixed_accuracy;  // الدقة بالـ ppb
    unsigned long flags;           // CLK_FIXED_RATE_PARENT_ACCURACY
};
```
مثال: Crystal oscillator خارجي بـ 24 MHz.

#### `struct clk_gate` — clock يمكن إيقافه/تشغيله

```c
struct clk_gate {
    struct clk_hw hw;
    void __iomem *reg;    // عنوان الـ register
    u8 bit_idx;           // رقم الـ bit في الـ register
    u8 flags;             // CLK_GATE_SET_TO_DISABLE, HIWORD_MASK, BIG_ENDIAN
    spinlock_t *lock;     // حماية الـ register (shared مع clocks أخرى)
};
```

#### `struct clk_divider` — clock بـ divider قابل للتغيير

```c
struct clk_divider {
    struct clk_hw hw;
    void __iomem *reg;
    u8 shift;                           // مكان الـ bitfield في الـ register
    u8 width;                           // حجم الـ bitfield
    u16 flags;                          // CLK_DIVIDER_ONE_BASED, POWER_OF_TWO, إلخ
    const struct clk_div_table *table;  // جدول القيم الخاصة (اختياري)
    spinlock_t *lock;
};
```

#### `struct clk_mux` — clock يختار من بين عدة parents

```c
struct clk_mux {
    struct clk_hw hw;
    void __iomem *reg;
    const u32 *table;    // جدول القيم (اختياري)
    u32 mask;            // mask الـ bitfield
    u8 shift;
    u8 flags;            // CLK_MUX_INDEX_ONE, HIWORD_MASK, إلخ
    spinlock_t *lock;
};
```

#### `struct clk_fixed_factor` — مضاعف/قاسم ثابت

```c
struct clk_fixed_factor {
    struct clk_hw hw;
    unsigned int mult;    // المضاعف
    unsigned int div;     // القاسم
    unsigned long acc;    // الدقة الثابتة
    unsigned int flags;   // CLK_FIXED_FACTOR_FIXED_ACCURACY
};
// rate = parent_rate * mult / div
```

#### `struct clk_fractional_divider` — قاسم كسري

```c
struct clk_fractional_divider {
    struct clk_hw hw;
    void __iomem *reg;
    u8 mshift, mwidth;   // numerator bitfield
    u8 nshift, nwidth;   // denominator bitfield
    u8 flags;
    void (*approximation)(...); // callback لحساب أفضل تقريب
    spinlock_t *lock;
};
// rate = parent_rate * m / n
```

#### `struct clk_composite` — مجموعة mux + divider + gate في واحد

```c
struct clk_composite {
    struct clk_hw hw;
    struct clk_ops ops;       // الـ ops المُجمَّعة (تُحسب تلقائياً)
    struct clk_hw *mux_hw;    // مؤشر لجزء الـ mux
    struct clk_hw *rate_hw;   // مؤشر لجزء الـ divider
    struct clk_hw *gate_hw;   // مؤشر لجزء الـ gate
    const struct clk_ops *mux_ops;
    const struct clk_ops *rate_ops;
    const struct clk_ops *gate_ops;
};
```

---

### علاقات الـ Structs (Diagram)

```
                    ┌─────────────────────┐
                    │   clk_init_data     │
                    │ - name              │
                    │ - ops ──────────────┼──► clk_ops (vtable)
                    │ - parent_data       │
                    │ - flags             │
                    └──────────┬──────────┘
                               │ .init (NULL بعد register)
                    ┌──────────▼──────────┐
                    │      clk_hw         │◄── يُضمَّن في clk_gate/
                    │ - core ─────────────┼──► clk_core (داخلي)    clk_divider/
                    │ - clk ──────────────┼──► struct clk (user)   clk_mux/إلخ
                    │ - init              │
                    └─────────────────────┘

Hardware Structs (كلها تحتوي clk_hw كأول عضو):
┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  clk_fixed_rate │  │    clk_gate      │  │   clk_divider    │
│ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐  │
│ │   clk_hw    │ │  │ │   clk_hw    │ │  │ │   clk_hw    │  │
│ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘  │
│ - fixed_rate    │  │ - reg           │  │ - reg            │
│ - fixed_accuracy│  │ - bit_idx       │  │ - shift, width   │
│ - flags         │  │ - flags, lock   │  │ - flags, lock    │
└─────────────────┘  └──────────────────┘  └──────────────────┘

┌──────────────────┐  ┌───────────────────────────────────────┐
│     clk_mux      │  │           clk_composite               │
│ ┌─────────────┐  │  │ ┌─────────────┐                       │
│ │   clk_hw    │  │  │ │   clk_hw    │                       │
│ └─────────────┘  │  │ └─────────────┘                       │
│ - reg, mask      │  │ - mux_hw ──► clk_mux.hw               │
│ - shift, flags   │  │ - rate_hw──► clk_divider.hw           │
│ - table, lock    │  │ - gate_hw──► clk_gate.hw              │
└──────────────────┘  └───────────────────────────────────────┘
```

### دورة حياة الـ Clock (Lifecycle)

```
1. DECLARATION (في driver probe أو __init)
   ┌─────────────────────────────────────────────────────┐
   │  static struct clk_gate my_gate = {                 │
   │      .hw.init = CLK_HW_INIT(...),                   │
   │      .reg = base + GATE_REG,                        │
   │      .bit_idx = 5,                                  │
   │  };                                                 │
   └─────────────────────────────────────────────────────┘
                          │
2. REGISTRATION            ▼
   ┌─────────────────────────────────────────────────────┐
   │  ret = clk_hw_register(dev, &my_gate.hw);           │
   │  // أو: devm_clk_hw_register() للـ auto-unregister  │
   └─────────────────────────────────────────────────────┘
                          │
3. USAGE (consumer driver)▼
   ┌─────────────────────────────────────────────────────┐
   │  clk = devm_clk_get(dev, "my-clock");               │
   │  clk_prepare_enable(clk);                           │
   │  clk_set_rate(clk, 100000000);                      │
   └─────────────────────────────────────────────────────┘
                          │
4. TEARDOWN                ▼
   ┌─────────────────────────────────────────────────────┐
   │  clk_disable_unprepare(clk);                        │
   │  clk_put(clk);                                      │
   │  clk_hw_unregister(&my_gate.hw);                    │
   │  // أو يتم تلقائياً مع devm_clk_hw_register        │
   └─────────────────────────────────────────────────────┘
```

### استراتيجية الـ Locking

الـ CCF يستخدم **قفلين منفصلين** لسببين مختلفين:

```
┌───────────────────────────────────────────────────────────┐
│  prepare_lock (mutex — يمكنه النوم)                        │
│  ───────────────────────────────────────────────────       │
│  • يحمي: clk_prepare/unprepare, clk_set_rate              │
│  • يحمي: clk_set_parent, clk_get_rate                     │
│  • يمكن الاستخدام في: process context فقط                 │
│  • الأولوية: يمنع race conditions في عمليات بطيئة          │
│                                                           │
│  enable_lock (spinlock — لا ينام)                         │
│  ───────────────────────────────────────────────────       │
│  • يحمي: clk_enable/disable                               │
│  • يمكن الاستخدام في: interrupt context                   │
│  • الأولوية: سرعة قصوى مع حماية كافية                    │
│                                                           │
│  lock الـ hardware registers (spinlock_t *lock في structs) │
│  ───────────────────────────────────────────────────       │
│  • يحمي: read-modify-write على registers مشتركة           │
│  • كل gate/divider/mux يشير لنفس الـ lock إذا كانوا       │
│    يتشاركون نفس الـ register                              │
└───────────────────────────────────────────────────────────┘
```

### جداول الـ Flags

#### CLK Flags (للـ clk_init_data.flags)

| Flag | Bit | المعنى |
|------|-----|--------|
| `CLK_SET_RATE_GATE` | 0 | لازم الـ clock يكون مُغلق أثناء تغيير الـ rate |
| `CLK_SET_PARENT_GATE` | 1 | لازم مُغلق أثناء تغيير الـ parent |
| `CLK_SET_RATE_PARENT` | 2 | انقل طلب تغيير الـ rate للـ parent |
| `CLK_IGNORE_UNUSED` | 3 | لا تغلقه حتى لو لم يستخدمه أحد |
| `CLK_GET_RATE_NOCACHE` | 6 | لا تستخدم الـ cached rate، اسأل الـ hardware دائماً |
| `CLK_SET_RATE_NO_REPARENT` | 7 | لا تغير الـ parent عند تغيير الـ rate |
| `CLK_GET_ACCURACY_NOCACHE` | 8 | لا تستخدم الـ cached accuracy |
| `CLK_RECALC_NEW_RATES` | 9 | أعد حساب الـ rates بعد الإشعارات |
| `CLK_SET_RATE_UNGATE` | 10 | يحتاج الـ clock يعمل لتغيير الـ rate |
| `CLK_IS_CRITICAL` | 11 | لا تغلقه أبداً (critical system clock) |
| `CLK_OPS_PARENT_ENABLE` | 12 | فعّل الـ parents أثناء gate/rate/reparent |
| `CLK_DUTY_CYCLE_PARENT` | 13 | مرر duty cycle calls للـ parent |

#### CLK_GATE Flags

| Flag | المعنى |
|------|--------|
| `CLK_GATE_SET_TO_DISABLE` | 1 = disable بدل 1 = enable |
| `CLK_GATE_HIWORD_MASK` | الـ mask في upper 16-bit من الـ register |
| `CLK_GATE_BIG_ENDIAN` | استخدم big-endian register access |

#### CLK_DIVIDER Flags

| Flag | المعنى |
|------|--------|
| `CLK_DIVIDER_ONE_BASED` | القيمة المقروءة = القاسم مباشرةً (ليس +1) |
| `CLK_DIVIDER_POWER_OF_TWO` | القاسم = 2^(قيمة الـ register) |
| `CLK_DIVIDER_ALLOW_ZERO` | اسمح بقيمة صفر |
| `CLK_DIVIDER_HIWORD_MASK` | mask في upper 16-bit |
| `CLK_DIVIDER_ROUND_CLOSEST` | قرّب للأقرب بدل الأعلى |
| `CLK_DIVIDER_READ_ONLY` | لا يمكن تغييره، للقراءة فقط |
| `CLK_DIVIDER_MAX_AT_ZERO` | قيمة صفر تعني أقصى قاسم (2^width) |
| `CLK_DIVIDER_BIG_ENDIAN` | big-endian register access |
| `CLK_DIVIDER_EVEN_INTEGERS` | قواسم زوجية فقط: 2*(val+1) |

#### CLK_MUX Flags

| Flag | المعنى |
|------|--------|
| `CLK_MUX_INDEX_ONE` | الـ index يبدأ من 1 لا من 0 |
| `CLK_MUX_INDEX_BIT` | الـ index هو bit واحد (power of two) |
| `CLK_MUX_HIWORD_MASK` | mask في upper 16-bit |
| `CLK_MUX_READ_ONLY` | للقراءة فقط (get_parent بس) |
| `CLK_MUX_ROUND_CLOSEST` | اختر الـ parent الأقرب للـ rate المطلوب |
| `CLK_MUX_BIG_ENDIAN` | big-endian register access |

---

## Phase 4: دليل الـ Debugging الشامل

### Software Level (Kernel Side)

#### 1. debugfs

الـ CCF يوفر شجرة كاملة في debugfs:

```bash
# عرض كل الـ clocks وحالتها
cat /sys/kernel/debug/clk/clk_summary

# مثال على الخرج:
#                             enable  prepare  protect         rate   accuracy   phase
# clock                          cnt      cnt      cnt
# ─────────────────────────────────────────────────────────────────────────────────────
# osc24M                           1        1        0     24000000          0   0
#    pll-cpu                        1        1        0    600000000          0   0
#       clk-cpu                     1        1        0    600000000          0   0
#    pll-periph0                    2        2        0    600000000          0   0
#       ahb1                        2        2        0    200000000          0   0
#          apb1                     1        1        0    100000000          0   0
#             uart0-apb              1        1        0    100000000          0   0

# عرض شجرة الـ clocks فقط
cat /sys/kernel/debug/clk/clk_dump

# عرض clock معين
ls /sys/kernel/debug/clk/pll-cpu/
# clk_accuracy  clk_duty_cycle  clk_enable_count  clk_flags
# clk_min_rate  clk_max_rate    clk_notifier_count  clk_parent
# clk_phase     clk_prepare_count  clk_rate
cat /sys/kernel/debug/clk/pll-cpu/clk_rate
# 600000000
```

#### 2. sysfs

```bash
# المعلومات الأساسية عبر sysfs (بعض السبورات فقط)
ls /sys/bus/platform/drivers/

# للـ clocks المرتبطة بـ device
cat /sys/devices/platform/soc/clock-controller/of_node/clock-names
```

#### 3. ftrace — تتبع استدعاءات الـ Clock

```bash
# تفعيل tracing لكل events الـ clk
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... قم بتشغيل التطبيق ...
cat /sys/kernel/debug/tracing/trace

# تفعيل events محددة
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_disable/enable

# مثال على الخرج:
# systemd-udevd-189   [000] .....  12.345678: clk_set_rate: uart0-apb rate=100000000
# kworker/0:1-45      [000] .....  12.346000: clk_enable: pll-periph0
```

#### 4. dynamic debug

```bash
# تفعيل debug messages للـ clk subsystem
echo "module clk +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/clk/clk.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/clk/clk-gate.c +p" > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الرسائل
dmesg -w | grep clk
```

#### 5. Kernel Config للـ Debugging

```
CONFIG_COMMON_CLK=y          # الـ CCF نفسه
CONFIG_CLK_DEBUG=y           # يفعّل debugfs entries
CONFIG_DEBUG_FS=y            # لازم لـ debugfs
CONFIG_COMMON_CLK_DEBUG=y    # معلومات إضافية
```

#### 6. رسائل الخطأ الشائعة

| الرسالة | السبب المحتمل | الحل |
|---------|---------------|------|
| `clk: couldn't get clock 'uart-clk'` | اسم الـ clock في DT غلط | تحقق من `clock-names` في DT |
| `clk: failed to prepare clock` | الـ prepare() callback أرجع error | تحقق من الـ regulator أو الـ init sequence |
| `clk: set rate failed` | الـ rate المطلوب خارج النطاق | استخدم `clk_round_rate()` أولاً |
| `WARNING: CPU: 0 PID: 1 ... clk_disable called for unprepared clock` | ترتيب استدعاء خاطئ | لازم `prepare` قبل `enable` |
| `clk: clk_core_set_parent_nolock failed` | الـ parent غير موجود أو لا يمكن تغييره | تحقق من CLK_SET_PARENT_GATE |

#### 7. إضافة WARN_ON في نقاط استراتيجية

```c
// في clk_ops->set_rate
static int my_clk_set_rate(struct clk_hw *hw, unsigned long rate,
                            unsigned long parent_rate)
{
    WARN_ON(rate == 0);
    WARN_ON(rate > MAX_SUPPORTED_RATE);
    // ...
}

// في clk_ops->enable
static int my_clk_enable(struct clk_hw *hw)
{
    struct my_clk *clk = to_my_clk(hw);
    WARN_ON(!clk->reg);  // تحقق أن الـ register mapped
    // ...
}
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel

```bash
# قراءة الـ rate من الـ kernel
cat /sys/kernel/debug/clk/my-clock/clk_rate

# قراءة الـ register مباشرة (إذا كنت تعرف العنوان)
devmem2 0x01c20000 w  # مثال لعنوان CCU على Allwinner H616

# أو باستخدام io command
io -4 -r 0x01c20000
```

#### 2. Register Dump للـ Clock Controller

```bash
# dump كل registers الـ clock controller (مثال لـ RK3562)
# ابحث عن عنوان الـ clock controller في DT
cat /proc/device-tree/clock-controller@*/reg | xxd

# استخدام devmem2 لقراءة range كامل
for addr in $(seq 0xFD7C0000 4 0xFD7C0200); do
    printf "0x%08X: " $addr
    devmem2 $addr w 2>/dev/null | grep "Value"
done
```

#### 3. Device Tree Debugging

```bash
# تحقق من الـ clock bindings في DT
dtc -I fs /proc/device-tree | grep -A5 "clock-names"

# تحقق من أن الـ clock node موجود ومتوافق
ls /proc/device-tree/clock-controller*/
cat /proc/device-tree/clock-controller*/compatible

# تحقق من الـ clock assignments
cat /proc/device-tree/uart*/clocks | xxd
cat /proc/device-tree/uart*/clock-names
```

#### 4. Logic Analyzer Tips

عند debugging مشاكل الـ clock على الـ oscilloscope:
- اقيس على الـ crystal input (OSC_IN/OSC_OUT) للتحقق من الـ reference clock
- ابحث عن الـ test points المخصصة للـ clock output في الـ schematic
- تحقق أن الـ duty cycle قريب من 50% للـ clocks الرقمية
- PLL lock time عادةً 50-200 µs — قس الوقت من enable حتى stable output

---

### Practical Commands (جاهزة للنسخ)

```bash
# === نظرة عامة سريعة ===
cat /sys/kernel/debug/clk/clk_summary | head -50

# === البحث عن clock معين ===
cat /sys/kernel/debug/clk/clk_summary | grep uart

# === عرض جميع الـ clocks المفعّلة فقط ===
awk '$2 > 0' /sys/kernel/debug/clk/clk_summary

# === تتبع تغييرات الـ rate في real-time ===
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# === عرض معلومات clock معين بالتفصيل ===
CLK="pll-cpu"
echo "Rate: $(cat /sys/kernel/debug/clk/$CLK/clk_rate)"
echo "Enable count: $(cat /sys/kernel/debug/clk/$CLK/clk_enable_count)"
echo "Prepare count: $(cat /sys/kernel/debug/clk/$CLK/clk_prepare_count)"
echo "Parent: $(cat /sys/kernel/debug/clk/$CLK/clk_parent 2>/dev/null || echo 'N/A')"
echo "Flags: $(cat /sys/kernel/debug/clk/$CLK/clk_flags)"

# === dump كامل لشجرة الـ clocks بصيغة JSON ===
cat /sys/kernel/debug/clk/clk_dump
```

---

## Phase 5: شرح الـ Functions

### المجموعة 1: وظائف تهيئة الـ Rate Request

#### `clk_hw_init_rate_request()`

```c
void clk_hw_init_rate_request(const struct clk_hw *hw,
                               struct clk_rate_request *req,
                               unsigned long rate);
```

تهيئ `struct clk_rate_request` بقيم افتراضية مناسبة قبل استخدامه في `determine_rate`. تضع الـ rate المطلوب وتملأ min/max من الحدود المسجلة للـ clock. **يجب استدعاؤها** قبل أي استخدام للـ struct.

- `hw`: مؤشر للـ clock المعني
- `req`: الـ struct المراد تهيئته
- `rate`: الـ rate المطلوب

#### `clk_hw_forward_rate_request()`

```c
void clk_hw_forward_rate_request(const struct clk_hw *core,
                                  const struct clk_rate_request *old_req,
                                  const struct clk_hw *parent,
                                  struct clk_rate_request *req,
                                  unsigned long parent_rate);
```

تنقل طلب الـ rate من clock لـ parent آخر. تُستخدم داخل `determine_rate` callback عندما تريد تفويض القرار للـ parent. تحسب الحدود المناسبة للـ parent بناءً على حدود الـ child.

---

### المجموعة 2: وظائف تسجيل Fixed Rate Clock

#### `__clk_hw_register_fixed_rate()` (الداخلية)

```c
struct clk_hw *__clk_hw_register_fixed_rate(struct device *dev,
    struct device_node *np, const char *name,
    const char *parent_name, const struct clk_hw *parent_hw,
    const struct clk_parent_data *parent_data, unsigned long flags,
    unsigned long fixed_rate, unsigned long fixed_accuracy,
    unsigned long clk_fixed_flags, bool devm);
```

الوظيفة الداخلية التي تُسجّل fixed-rate clock. لا تستدعيها مباشرةً — استخدم الـ macros بدلاً منها. معامل `devm` يحدد إن كان التسجيل سيتم إلغاؤه تلقائياً مع الـ device.

#### Macros الـ Fixed Rate (Public API)

| Macro | الاستخدام |
|-------|-----------|
| `clk_hw_register_fixed_rate(dev, name, parent_name, flags, fixed_rate)` | التسجيل الأساسي |
| `devm_clk_hw_register_fixed_rate(...)` | مع auto-unregister |
| `clk_hw_register_fixed_rate_parent_hw(...)` | الـ parent بمؤشر clk_hw |
| `clk_hw_register_fixed_rate_parent_data(...)` | الـ parent ببيانات parent_data |
| `clk_hw_register_fixed_rate_with_accuracy(...)` | مع تحديد الدقة |
| `clk_hw_register_fixed_rate_parent_accuracy(...)` | يرث دقة الـ parent |

#### `clk_unregister_fixed_rate()` / `clk_hw_unregister_fixed_rate()`

```c
void clk_unregister_fixed_rate(struct clk *clk);
void clk_hw_unregister_fixed_rate(struct clk_hw *hw);
```

يلغي تسجيل fixed-rate clock ويحرر الذاكرة. الإصدار `clk_hw_` هو الأحدث والمفضّل.

#### `of_fixed_clk_setup()`

```c
void of_fixed_clk_setup(struct device_node *np);
```

يسجّل fixed-rate clock من Device Tree node. يُستدعى تلقائياً بواسطة `CLK_OF_DECLARE` عند قراءة DT.

---

### المجموعة 3: وظائف تسجيل Gate Clock

#### `__clk_hw_register_gate()` / `__devm_clk_hw_register_gate()`

الوظيفتان الداخليتان. يفضل استخدام الـ macros:

| Macro | الاستخدام |
|-------|-----------|
| `clk_hw_register_gate(dev, name, parent_name, flags, reg, bit_idx, clk_gate_flags, lock)` | أساسي |
| `clk_hw_register_gate_parent_hw(...)` | parent بمؤشر |
| `clk_hw_register_gate_parent_data(...)` | parent ببيانات |
| `devm_clk_hw_register_gate(...)` | مع auto-unregister |
| `devm_clk_hw_register_gate_parent_hw(...)` | devm + parent_hw |
| `devm_clk_hw_register_gate_parent_data(...)` | devm + parent_data |

#### `clk_gate_is_enabled()`

```c
int clk_gate_is_enabled(struct clk_hw *hw);
```

يقرأ الـ register مباشرة للتحقق إذا كان الـ gate مفعلاً. يستخدم في `is_enabled` callback.

---

### المجموعة 4: وظائف تسجيل Divider Clock

#### Helper Functions للـ Divider (مهمة جداً)

```c
unsigned long divider_recalc_rate(struct clk_hw *hw,
    unsigned long parent_rate, unsigned int val,
    const struct clk_div_table *table,
    unsigned long flags, unsigned long width);
```
يحسب الـ rate الفعلي بناءً على قيمة الـ register. مفيد في تطبيق `recalc_rate`.

```c
long divider_round_rate_parent(struct clk_hw *hw, struct clk_hw *parent,
    unsigned long rate, unsigned long *prate,
    const struct clk_div_table *table,
    u8 width, unsigned long flags);
```
يحسب أقرب rate يمكن تحقيقه. يُحدّث `*prate` بالـ parent rate المطلوب.

```c
int divider_determine_rate(struct clk_hw *hw,
    struct clk_rate_request *req,
    const struct clk_div_table *table,
    u8 width, unsigned long flags);
```
النسخة الحديثة من `round_rate` — تعمل مع `clk_rate_request` وتدعم min/max constraints.

```c
int divider_get_val(unsigned long rate, unsigned long parent_rate,
    const struct clk_div_table *table, u8 width,
    unsigned long flags);
```
يحسب قيمة الـ register اللازمة لتحقيق الـ rate المطلوب.

---

### المجموعة 5: وظائف تسجيل Mux Clock

```c
int clk_mux_val_to_index(struct clk_hw *hw, const u32 *table,
    unsigned int flags, unsigned int val);
```
يحوّل قيمة الـ register لـ index في مصفوفة الـ parents.

```c
unsigned int clk_mux_index_to_val(const u32 *table,
    unsigned int flags, u8 index);
```
العكس — يحوّل الـ index لقيمة الـ register.

---

### المجموعة 6: وظائف التسجيل العامة

```c
int __must_check clk_hw_register(struct device *dev, struct clk_hw *hw);
int __must_check devm_clk_hw_register(struct device *dev, struct clk_hw *hw);
int __must_check of_clk_hw_register(struct device_node *node, struct clk_hw *hw);
```

| وظيفة | الاستخدام |
|-------|-----------|
| `clk_hw_register()` | تسجيل عادي — لازم `clk_hw_unregister()` يدوياً |
| `devm_clk_hw_register()` | auto-unregister مع الـ device |
| `of_clk_hw_register()` | التسجيل من DT node بدون device |

```c
void clk_hw_unregister(struct clk_hw *hw);
```
يلغي التسجيل ويحرر الموارد.

---

### المجموعة 7: Helper Functions للـ Provider

```c
const char *clk_hw_get_name(const struct clk_hw *hw);
unsigned int clk_hw_get_num_parents(const struct clk_hw *hw);
struct clk_hw *clk_hw_get_parent(const struct clk_hw *hw);
struct clk_hw *clk_hw_get_parent_by_index(const struct clk_hw *hw, unsigned int index);
int clk_hw_get_parent_index(struct clk_hw *hw);
int clk_hw_set_parent(struct clk_hw *hw, struct clk_hw *new_parent);
unsigned long clk_hw_get_rate(const struct clk_hw *hw);
unsigned long clk_hw_get_flags(const struct clk_hw *hw);
bool clk_hw_is_prepared(const struct clk_hw *hw);
bool clk_hw_is_enabled(const struct clk_hw *hw);
unsigned long clk_hw_round_rate(struct clk_hw *hw, unsigned long rate);
```

هذه الوظائف تُستخدم **داخل clk_ops callbacks** للوصول لمعلومات الـ clock الحالية.

---

### المجموعة 8: وظائف Device Tree Provider

```c
int of_clk_add_hw_provider(struct device_node *np,
    struct clk_hw *(*get)(struct of_phandle_args *clkspec, void *data),
    void *data);
int devm_of_clk_add_hw_provider(struct device *dev, ...);
void of_clk_del_provider(struct device_node *np);
```

تسجّل "مزود" الـ clock في الـ DT. بعد هذا الاستدعاء، يمكن لأي driver يشير لهذا الـ node في DT أن يحصل على الـ clock.

```c
struct clk_hw *of_clk_hw_simple_get(struct of_phandle_args *clkspec, void *data);
struct clk_hw *of_clk_hw_onecell_get(struct of_phandle_args *clkspec, void *data);
```

أشهر callback implementations:
- `of_clk_hw_simple_get`: للـ nodes التي تمتلك clock واحداً فقط
- `of_clk_hw_onecell_get`: للـ nodes التي تمتلك array من الـ clocks

---

### المجموعة 9: Macros الـ Initialization السريعة

```c
// مثال على استخدام CLK_HW_INIT
static struct clk_gate my_gate_clk = {
    .hw.init = CLK_HW_INIT(
        "uart0-gate",       // اسم الـ clock
        "apb1",             // اسم الـ parent
        &clk_gate_ops,      // الـ ops
        0                   // flags
    ),
    .reg = base + UART0_GATE_REG,
    .bit_idx = 20,
    .lock = &my_lock,
};

// مثال CLK_HW_INIT_HW (أفضل — parent بمؤشر مباشر)
static struct clk_gate my_gate_clk = {
    .hw.init = CLK_HW_INIT_HW(
        "uart0-gate",
        &apb1_clk.hw,   // مؤشر مباشر بدل اسم
        &clk_gate_ops,
        0
    ),
    // ...
};

// مثال CLK_HW_INIT_PARENTS (لـ mux بعدة parents)
static const char * const mux_parents[] = { "osc24M", "pll-cpu", "pll-periph" };
static struct clk_mux my_mux = {
    .hw.init = CLK_HW_INIT_PARENTS(
        "cpu-mux",
        mux_parents,
        &clk_mux_ops,
        0
    ),
    // ...
};
```

---

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART لا يعمل على Allwinner H616 (Android TV Box)

**العنوان**: "الـ UART صامت تماماً بعد boot الـ Android TV box"

**السياق**: تعمل على board مبنية على Allwinner H616 لصنع Android TV box. المشروع في مرحلة bring-up والـ UART console لا يعطي أي output رغم أن الـ bootloader يعمل.

**المشكلة**: `uart0` لا يعمل. لا output في dmesg، وأي محاولة لكتابة على الـ UART تُرجع error.

**التحليل**:
```bash
# الخطوة 1: تحقق من حالة الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep uart
# uart0-apb    0    0    0    100000000    0    0

# الـ enable_count = 0 !!! يعني الـ clock مش مفعّل

# الخطوة 2: تحقق من الـ DT
cat /proc/device-tree/soc/uart@5000000/clock-names
# apb — هذا صح

# الخطوة 3: تحقق من الـ parent
cat /sys/kernel/debug/clk/uart0-apb/clk_parent
# apb1 — صح

cat /sys/kernel/debug/clk/apb1/clk_enable_count
# 0 — المشكلة! الـ parent نفسه مش مفعّل
```

في ملف `clk-provider.h`، نرى أن `CLK_SET_RATE_PARENT` يعني "انقل تغيير الـ rate للـ parent". لكن المشكلة هنا في الـ enable/disable chain. نفتح الـ uart driver ونجد:

```c
// مشكلة في الـ driver: ينسى تفعيل الـ apb bus clock
uart->clk = devm_clk_get(dev, "baud");  // فقط baud clock
// ناقص: uart->apb_clk = devm_clk_get(dev, "apb");
```

**الحل**:
```c
// في uart driver:
uart->apb_clk = devm_clk_get(dev, "apb");
if (IS_ERR(uart->apb_clk))
    return PTR_ERR(uart->apb_clk);

ret = clk_prepare_enable(uart->apb_clk);
if (ret)
    return ret;
```

في الـ DT:
```dts
uart0: uart@5000000 {
    compatible = "allwinner,sun50i-h616-uart";
    clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>;
    clock-names = "apb", "baud";  /* لازم يكون الاثنين */
};
```

**الدرس**: دائماً تحقق من `clk_enable_count` لكل الـ clocks في الـ chain، ليس فقط الـ clock النهائي. الـ CCF لا يفعّل الـ parent تلقائياً إلا إذا كان `CLK_OPS_PARENT_ENABLE` مضبوطاً.

---

### السيناريو 2: SPI بطيء جداً على STM32MP1 (Industrial Gateway)

**العنوان**: "الـ SPI يعمل بـ 1 MHz بدل 10 MHz على STM32MP1"

**السياق**: تبني industrial gateway بـ STM32MP1. الـ SPI يجب أن يعمل بـ 10 MHz لقراءة sensor بيانات بسرعة كافية، لكنه يعمل بـ 1 MHz فعلياً.

**المشكلة**: `clk_set_rate(spi_clk, 10000000)` تنجح (لا error) لكن الـ rate الفعلي 1 MHz.

**التحليل**:
```bash
# تحقق من الـ rate الفعلي
cat /sys/kernel/debug/clk/spi1/clk_rate
# 1000000 — المشكلة مؤكدة

# تحقق من الـ parent
cat /sys/kernel/debug/clk/spi1/clk_parent
# ck_mco1

cat /sys/kernel/debug/clk/ck_mco1/clk_rate
# 8000000

# الـ parent معدله 8 MHz، لكن الـ SPI يحتاج 10 MHz
# المشكلة: CLK_SET_RATE_PARENT مش مضبوط!
```

نرجع لـ `clk-provider.h` ونراجع الـ flags:
```
CLK_SET_RATE_PARENT — "propagate rate change up one level"
```

```bash
# تحقق من flags الـ SPI clock
cat /sys/kernel/debug/clk/spi1/clk_flags
# 0x4 — فقط CLK_SET_RATE_PARENT ليس مضبوطاً (هو BIT(2))
# 0x4 = BIT(2) يعني CLK_SET_RATE_PARENT مضبوط في الواقع!

# إذاً المشكلة في الـ divider
cat /sys/kernel/debug/clk/spi1/clk_min_rate
# 0
cat /sys/kernel/debug/clk/spi1/clk_max_rate
# 8000000 — هنا المشكلة! مقيّد بـ 8 MHz
```

**الحل**: في الـ DTS نضيف `assigned-clock-rates` لتغيير الـ parent rate:
```dts
&spi1 {
    assigned-clocks = <&rcc SPI1_K>;
    assigned-clock-rates = <10000000>;
    assigned-clock-parents = <&rcc PLL3_Q>;  /* PLL3_Q = 100 MHz */
};
```

أو في الـ driver:
```c
// استخدم clk_set_rate_range لتحديد النطاق بشكل صحيح
ret = clk_set_rate_range(spi->clk, 0, 50000000);
ret = clk_set_rate(spi->clk, 10000000);
```

**الدرس**: `clk_set_rate()` تنجح حتى لو لم يتحقق الـ rate المطلوب — الـ CCF يُرجع أقرب rate ممكن. دائماً تحقق من الـ rate الفعلي بعد `clk_set_rate()` باستخدام `clk_get_rate()`.

---

### السيناريو 3: Display لا يظهر على i.MX8 (Custom Board)

**العنوان**: "HDMI output أسود تماماً على i.MX8MQ بعد boot"

**السياق**: custom board بـ i.MX8MQ مع HDMI display. الـ display يظهر أسود ولا يوجد signal على الـ HDMI.

**المشكلة**: الـ display pipeline تعمل لكن لا يوجد pixel clock صحيح للـ HDMI.

**التحليل**:
```bash
# تحقق من الـ pixel clock
cat /sys/kernel/debug/clk/clk_summary | grep hdmi
# hdmi_phy     0    0    0    0    0    0
# hdmi_pixel   0    0    0    0    0    0

# كلاهما بـ rate = 0 وعدد enable = 0

# تحقق من الـ PLL المسؤول
cat /sys/kernel/debug/clk/clk_summary | grep video_pll1
# video_pll1    0    0    0    594000000    0    0

# الـ PLL شغّال (rate محسوب) لكن الـ enable count = 0
```

المشكلة في `clk-provider.h` context: الـ `CLK_IS_CRITICAL` flag غير مضبوط على الـ video PLL، و الـ `CLK_IGNORE_UNUSED` غير مضبوط، فالـ kernel يغلقه كـ unused clock.

```bash
# تحقق من الـ DT
cat /proc/device-tree/hdmi*/clocks | xxd
# نجد أن الـ clock-names ناقصة "pixel" clock
```

**الحل**: في الـ DTS الخاص بالـ board:
```dts
&hdmi {
    clocks = <&clk IMX8MQ_CLK_HDMI_AXI>,
             <&clk IMX8MQ_CLK_HDMI_APB>,
             <&clk IMX8MQ_VIDEO_PLL1_REF_SEL>;
    clock-names = "iahb", "isfr", "pixel";  /* أضف "pixel" */
};
```

أو الحل السريع للـ testing:
```bash
# أجبر الـ PLL على التشغيل
echo 1 > /sys/kernel/debug/clk/video_pll1/clk_prepare_enable 2>/dev/null
# ملاحظة: هذا للـ testing فقط، ليس production
```

**الدرس**: مشاكل الـ display غالباً related بـ clock chain كاملة. ابدأ من الـ pixel clock وارجع للـ PLL خطوة بخطوة محققاً من كل enable_count.

---

### السيناريو 4: نظام يعيد التشغيل تلقائياً على AM62x (IoT Sensor)

**العنوان**: "الـ board تعيد التشغيل عشوائياً على AM62x"

**السياق**: IoT sensor node بـ TI AM62x، النظام يعيد التشغيل بشكل عشوائي كل بضع ساعات.

**المشكلة**: في الـ kernel log يظهر: `WARNING: at drivers/clk/clk.c:XXX clk_disable_unused`

**التحليل**:
```bash
dmesg | grep -i "clk\|clock" | grep -i "warn\|err\|crit"
# clk: Disabling unused clocks
# WARNING: CPU:0 clk_disable: disabling critical clock cpu-mux

# المشكلة واضحة: kernel يغلق الـ CPU clock!
```

نرجع لـ `clk-provider.h`:
```c
#define CLK_IS_CRITICAL  BIT(11)  /* do not gate, ever */
```

الـ AM62x clock driver نسي وضع `CLK_IS_CRITICAL` على الـ CPU clock:

```c
// في drivers/clk/k3/j7200-main.c (مثال)
static const struct clk_data clk_data_cpu[] = {
    [TI_CLK_CPU] = {
        .name = "cpu0-mux",
        .ops = &clk_mux_ops,
        // ناقص: .flags = CLK_IS_CRITICAL,
    },
};
```

**الحل**:
```c
// إضافة CLK_IS_CRITICAL
static const struct clk_init_data cpu_mux_init = {
    .name = "cpu0-mux",
    .ops = &clk_mux_ops,
    .parent_names = cpu_mux_parents,
    .num_parents = ARRAY_SIZE(cpu_mux_parents),
    .flags = CLK_IS_CRITICAL,  /* ← هذا يمنع الإغلاق */
};
```

أو في الـ DTS للـ workaround:
```dts
&cpu_clock_controller {
    assigned-clocks = <&k3_clks 0 0>;
    /* لا تستخدم CLK_IGNORE_UNUSED هنا — استخدم CLK_IS_CRITICAL في الـ driver */
};
```

**الدرس**: كل clock يتحكم في CPU أو memory bus لازم يكون عليه `CLK_IS_CRITICAL`. الـ `clk_disable_unused` يعمل أثناء الـ late init وقد يُغلق clocks حيوية إذا لم تكن محمية.

---

### السيناريو 5: استهلاك طاقة عالٍ على RK3562 (Battery-Powered Device)

**العنوان**: "البطارية تفرغ في 4 ساعات بدل 12 ساعة"

**السياق**: جهاز محمول بـ RK3562، يجب أن يعمل 12 ساعة لكنه يفرغ في 4 ساعات فقط.

**المشكلة**: clocks غير ضرورية تعمل بتردد عالٍ في وضع الـ idle.

**التحليل**:
```bash
# فحص جميع الـ clocks المفعّلة
awk '$2 > 0 && $4 > 100000000' /sys/kernel/debug/clk/clk_summary
# gpu-core    1    1    0    800000000    0    0
# vpu-aclk    1    1    0    600000000    0    0
# npu-clk     1    1    0    700000000    0    0

# هذه الـ clocks مفعّلة بتردد عالٍ حتى في حالة idle
```

نرجع لـ `clk-provider.h` ونرى الـ flags المتعلقة بالطاقة:
```c
CLK_IGNORE_UNUSED  BIT(3)  /* do not gate even if unused */
```

المشكلة: بعض الـ drivers تستخدم `CLK_IGNORE_UNUSED` بشكل خاطئ لـ debugging وتنسى إزالته قبل الإنتاج.

```bash
# تحقق من الـ flags
cat /sys/kernel/debug/clk/gpu-core/clk_flags
# 0x8 = BIT(3) = CLK_IGNORE_UNUSED — هذا هو السبب!
```

**الحل**:

أولاً: أزل `CLK_IGNORE_UNUSED` من الـ drivers وأضف proper runtime PM:

```c
// في gpu driver
static int gpu_runtime_suspend(struct device *dev)
{
    struct gpu_dev *gpu = dev_get_drvdata(dev);
    clk_disable_unprepare(gpu->clk);  // أغلق الـ clock في suspend
    return 0;
}

static int gpu_runtime_resume(struct device *dev)
{
    struct gpu_dev *gpu = dev_get_drvdata(dev);
    return clk_prepare_enable(gpu->clk);  // شغّل عند الحاجة
}
```

ثانياً: في الـ DTS، أضف OPP tables للـ clock scaling:
```dts
&gpu {
    operating-points-v2 = <&gpu_opp_table>;
};

gpu_opp_table: opp-table {
    opp-100000000 { opp-hz = /bits/ 64 <100000000>; };
    opp-400000000 { opp-hz = /bits/ 64 <400000000>; };
    opp-800000000 { opp-hz = /bits/ 64 <800000000>; };
};
```

**الدرس**: `CLK_IGNORE_UNUSED` مفيد فقط للـ debugging. في الإنتاج، استخدم runtime PM بشكل صحيح. أي clock بـ enable_count > 0 يستهلك طاقة حتى لو لم يستخدمه أحد فعلياً.

---

## Phase 7: مصادر ومراجع

### LWN.net Articles (الأهم)

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| The common clock framework | https://lwn.net/Articles/472998/ | المقدمة الأساسية للـ CCF |
| Clock framework updates | https://lwn.net/Articles/500461/ | التحديثات الأولى |
| Common clock framework API | https://lwn.net/Articles/523509/ | شرح الـ API بالتفصيل |
| Representing clocks in the device tree | https://lwn.net/Articles/472642/ | DT bindings للـ clocks |
| CLK_SET_RATE_PARENT and friends | https://lwn.net/Articles/563895/ | شرح الـ flags المهمة |

### التوثيق الرسمي في الـ Kernel

```
Documentation/driver-api/clk.rst          — الـ API الكاملة
Documentation/devicetree/bindings/clock/  — DT bindings لكل الـ clock controllers
Documentation/admin-guide/pm/             — Power management وعلاقته بالـ clocks
```

```bash
# قراءة التوثيق مباشرةً
cat Documentation/driver-api/clk.rst
ls Documentation/devicetree/bindings/clock/
```

### الـ Commits المهمة في تاريخ الـ CCF

| Commit | الوصف |
|--------|-------|
| `012d7a2` | أول commit للـ CCF (kernel 3.4) |
| `0f6b850` | إضافة `clk_hw` provider API |
| `fc4a05d` | إضافة `clk_parent_data` (الأسلوب الحديث للـ parents) |
| `9a34b453` | إضافة `CLK_IS_CRITICAL` flag |
| `1a9c069` | إضافة `clk_rate_request` و `determine_rate` |
| `6e5ab97` | إضافة duty cycle support |

### مناقشات الـ Mailing List

- `linux-clk@vger.kernel.org` — القائمة الرئيسية لـ CCF
- البحث في: https://lore.kernel.org/linux-clk/

### الكتب الموصى بها

| الكتاب | الفصول ذات الصلة |
|--------|-----------------|
| **Linux Device Drivers, 3rd Ed** (LDD3) | Ch. 1-3 (driver basics), لا يوجد فصل CCF لأنه كتب قبل CCF |
| **Linux Kernel Development, 3rd Ed** | Ch. 17 (device model) |
| **Embedded Linux Systems with the Yocto Project** | Ch. 7 (BSP والـ device drivers) |
| **Mastering Embedded Linux Programming** | Ch. 9 (device drivers) |

### روابط مفيدة

| الموقع | الرابط | المحتوى |
|--------|--------|---------|
| kernelnewbies.org | https://kernelnewbies.org/Linux_3.4 | تفاصيل إضافة CCF في 3.4 |
| elinux.org | https://elinux.org/images/e/ee/Clement-rmt_-_clk_framework.pdf | شرح slides لـ CCF |
| bootlin.com | https://bootlin.com/doc/training/linux-kernel/ | مواد تدريب ممتازة |
| elixir.bootlin.com | https://elixir.bootlin.com/linux/latest/source/include/linux/clk-provider.h | الكود الحالي |

### مصطلحات البحث

للبحث عن معلومات إضافية، استخدم:
- `"common clock framework" linux kernel`
- `"struct clk_ops" tutorial`
- `clk_hw_register example driver`
- `linux CCF clock provider how to write`
- `device tree clock bindings example`
- `clk_set_rate_parent propagation`

### مثال كامل: كيف تكتب Clock Driver من الصفر

```c
// مثال مبسط لـ gate clock على SoC وهمي
#include <linux/clk-provider.h>
#include <linux/module.h>
#include <linux/platform_device.h>

#define MY_CLK_GATE_REG  0x100
#define MY_CLK_GATE_BIT  5

struct my_clk_data {
    void __iomem *base;
    spinlock_t lock;
    struct clk_hw *clk_hw;
};

static int my_probe(struct platform_device *pdev)
{
    struct my_clk_data *data;
    struct resource *res;

    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    data->base = devm_ioremap_resource(&pdev->dev, res);
    if (IS_ERR(data->base))
        return PTR_ERR(data->base);

    spin_lock_init(&data->lock);

    // تسجيل gate clock
    data->clk_hw = devm_clk_hw_register_gate(
        &pdev->dev,
        "my-uart-gate",           // اسم الـ clock
        "apb-bus",                // اسم الـ parent
        0,                        // framework flags
        data->base + MY_CLK_GATE_REG,
        MY_CLK_GATE_BIT,
        0,                        // gate flags
        &data->lock               // shared register lock
    );

    if (IS_ERR(data->clk_hw))
        return PTR_ERR(data->clk_hw);

    // تسجيل كـ DT provider
    return devm_of_clk_add_hw_provider(
        &pdev->dev,
        of_clk_hw_simple_get,
        data->clk_hw
    );
}
```

---

*تم إنشاء هذا التوثيق بواسطة `/kernel-doc` skill.*
*الملف المُوثَّق: `include/linux/clk-provider.h`*
*إصدار الـ Kernel: 6.x (Common Clock Framework)*
