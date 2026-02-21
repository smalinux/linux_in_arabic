## Phase 1: الصورة الكبيرة

# `sh_clk.h`

> **PATH**: `include/linux/sh_clk.h`
> **Subsystem**: SuperH (SH) Clock Framework / Clock Management
> **الوظيفة الأساسية**: تعريف هياكل البيانات وواجهات إدارة الـ clock لمعالجات SuperH (SH) من Renesas

---

### ما هي القصة الكبيرة؟

تخيل أن عندك مصنع فيه آلات كتير. كل آلة تحتاج كهرباء تشتغل بيها. لو شغّلت كل الآلات في نفس الوقت، الكهرباء راح تنتهي وتغلى الفلوس. الحل؟ تعمل "مشرف طاقة" ذكي يقدر:
- يشغّل الآلة لما تحتاجها بس.
- يضبط سرعتها (أسرع = أكل طاقة أكثر).
- يطفيها لما ما حد يحتاجها.

هذا بالضبط ما يعمله الـ **clock framework** في معالجات **SuperH (SH)** من Renesas. الـ `sh_clk.h` هو "دستور المشرف" — فيه تعريف كل شيء يحتاجه النظام عشان يتحكم في الساعات (clocks) اللي تشغّل كل وحدة في الشريحة.

---

### ما هي الـ Clock ولماذا نحتاج إدارتها؟

الـ **clock** في الشريحة (SoC) عبارة عن إشارة كهربائية تنبض بتردد معين (مثلاً 33 MHz أو 200 MHz). كل وحدة في الشريحة (USB، UART، GPU، RAM controller...) تحتاج هذه النبضة عشان "تعرف متى تشتغل".

المشكلة:
- الشريحة فيها عشرات أو مئات الـ clocks.
- بعض الـ clocks مترابطة — لو عدّلت الأب (parent clock)، الأولاد (child clocks) تتأثر.
- بعض الوحدات تحتاج إيقاف الـ clock عشان توفر طاقة (Power Management).
- بعضها يحتاج تغيير التردد ديناميكياً (Dynamic Voltage and Frequency Scaling - DVFS).

الـ `sh_clk.h` يحدد الإطار الكامل لحل هذه المشكلة على معالجات SuperH.

---

### ما الذي يوفره هذا الـ Header؟

#### 1. الـ `struct clk` — نواة كل شيء

هذا الـ struct هو "بطاقة هوية" كل clock في النظام. كل clock عنده:

```c
struct clk {
    struct list_head    node;          /* ربط الـ clock بقائمة كل الـ clocks */
    struct clk         *parent;        /* من أين يأتي هذا الـ clock؟ */
    struct clk        **parent_table;  /* قائمة بـ parents محتملين (للاختيار) */
    struct sh_clk_ops  *ops;           /* العمليات المتاحة (enable/disable/recalc) */

    struct list_head    children;      /* الـ clocks الأولاد */
    int                 usecount;      /* كم مستخدم يحتاجه الآن؟ */

    unsigned long       rate;          /* التردد الحالي بالـ Hz */
    unsigned long       flags;         /* خصائص إضافية */

    void __iomem       *enable_reg;    /* عنوان السجل (register) لتشغيل/إيقاف الـ clock */
    unsigned int        enable_bit;    /* رقم الـ bit داخل الـ register */
    unsigned int        div_mask;      /* قناع القسمة */
};
```

فكّر في كل `clk` كأنه محطة كهرباء صغيرة — عندها مصدر (parent)، وزبائن (children)، وكنترول لتشغيلها وإيقافها.

#### 2. الـ `struct sh_clk_ops` — وظائف كل Clock

```c
struct sh_clk_ops {
    int (*enable)(struct clk *clk);              /* شغّل الـ clock */
    void (*disable)(struct clk *clk);            /* أطفئ الـ clock */
    unsigned long (*recalc)(struct clk *clk);    /* احسب التردد الحالي */
    int (*set_rate)(struct clk *clk, unsigned long rate); /* اضبط التردد */
    int (*set_parent)(struct clk *clk, struct clk *parent); /* غيّر المصدر */
    long (*round_rate)(struct clk *clk, unsigned long rate); /* قرّب التردد لأقرب قيمة مدعومة */
};
```

هذا نمط الـ **vtable** (مثل الـ polymorphism في C++) — كل نوع clock يعرّف وظائفه الخاصة، لكن الواجهة موحّدة.

#### 3. أنواع الـ Clocks المدعومة

**الـ MSTP (Module Stop) Clocks:**
هذه الـ clocks تتحكم في تشغيل/إيقاف وحدات كاملة. مثلاً: "أوقف وحدة USB" = اكتب في سجل MSTP.

```c
#define SH_CLK_MSTP32(_p, _r, _b, _f)  \
    SH_CLK_MSTP(_p, _r, _b, 0, _f | CLK_ENABLE_REG_32BIT)
```

الـ macro يملأ `struct clk` بالقيم الصحيحة: أين الـ register، أي bit، وهل هو 8/16/32 bit.

**الـ DIV4 Clocks:**
تأخذ clock أسرع وتقسمه على 1، 2، 4، 8،... حتى تحصل على تردد أبطأ. تُستخدم مثلاً لتوليد clock لـ CPU من oscillator خارجي.

```c
#define SH_CLK_DIV4(_parent, _reg, _shift, _div_bitmap, _flags)
```

**الـ DIV6 Clocks:**
مثل DIV4 لكن يقسم على قيم حتى 64. يدعم أيضاً اختيار الـ parent من جدول.

**الـ FSIDIV Clocks:**
متخصصة لواجهة الصوت **FSI (Fifo-buffered Serial Interface)** — تحتاج mapping خاص لسجلاتها.

#### 4. جداول الترددات للـ cpufreq

```c
struct clk_div_mult_table {
    unsigned int *divisors;     /* قائمة القواسم المتاحة */
    unsigned int nr_divisors;
    unsigned int *multipliers;  /* قائمة المضاعفات */
    unsigned int nr_multipliers;
};
```

هذا يسمح لنظام **cpufreq** (تحكم ديناميكي في تردد المعالج) أن يعرف ما هي الترددات الممكنة لكل clock، وذلك بتوليد جدول `cpufreq_frequency_table` تلقائياً.

#### 5. الـ `clk_mapping` — تعيين الذاكرة

```c
struct clk_mapping {
    phys_addr_t      phys;    /* العنوان الفيزيائي للـ register */
    void __iomem    *base;    /* العنوان الافتراضي بعد iomap */
    unsigned long    len;     /* حجم المنطقة */
    struct kref      ref;     /* reference counting */
};
```

لأن عدة clocks قد تشارك نفس الـ register block، الـ `kref` يضمن أن الـ mapping لا يُحذف قبل أن تنتهي كل الـ clocks من استخدامه.

---

### قصة: رحلة تشغيل وحدة USB

تخيل أن driver يريد يشغّل وحدة USB على SH7724:

1. الـ driver يستدعي `clk_get()` للحصول على مؤشر الـ clock.
2. يستدعي `clk_enable()` — هذا يزيد الـ `usecount` ويكتب في سجل MSTP.
3. الـ CPG (Clock Pulse Generator) يوقف إشارة "module stop" → الوحدة تبدأ تشتغل.
4. عند الانتهاء: `clk_disable()` → إذا الـ usecount وصل صفر، تُطفأ الوحدة وتُوفّر طاقة.

كل هذا مُجرّد (abstracted) خلف `sh_clk_ops` و macros الـ MSTP.

---

### الـ Flags المهمة

| Flag | المعنى |
|---|---|
| `CLK_ENABLE_ON_INIT` | شغّل الـ clock عند الإقلاع فوراً |
| `CLK_ENABLE_REG_32BIT` | الـ register حجمه 32 bit |
| `CLK_ENABLE_REG_16BIT` | الـ register حجمه 16 bit |
| `CLK_ENABLE_REG_8BIT` | الـ register حجمه 8 bit |
| `CLK_MASK_DIV_ON_DISABLE` | عند الإيقاف، اكتب قيمة القسمة القصوى في الـ register |

---

### دوال الـ API العامة

```c
/* إدارة دورة حياة الـ clock */
int clk_register(struct clk *);
void clk_unregister(struct clk *);

/* إعادة حساب الترددات في شجرة الـ clocks */
void recalculate_root_clocks(void);
void propagate_rate(struct clk *);

/* تغيير الـ parent */
int clk_reparent(struct clk *child, struct clk *parent);

/* تسجيل مجموعات الـ clocks */
int sh_clk_mstp_register(struct clk *clks, int nr);
int sh_clk_div4_register(struct clk *clks, int nr, struct clk_div4_table *table);
int sh_clk_div6_register(struct clk *clks, int nr);
int sh_clk_fsidiv_register(struct clk *clks, int nr);
```

---

### الملفات المرتبطة — ما يجب أن تعرفه

**الـ Core Implementation:**
- `drivers/sh/clk/core.c` — تنفيذ كل دوال الـ framework الأساسية (enable/disable/recalc/register)
- `drivers/sh/clk/cpg.c` — تنفيذ MSTP و DIV4 و DIV6 و FSIDIV

**الـ Platform Clock Definitions (لكل CPU):**
- `arch/sh/kernel/cpu/sh4a/clock-sh7724.c` — تعريف كل clocks لشريحة SH7724
- `arch/sh/kernel/cpu/sh4a/clock-sh7723.c` — لشريحة SH7723
- `arch/sh/kernel/cpu/sh4a/clock-sh7722.c` — لشريحة SH7722

**الـ Headers المكملة:**
- `include/linux/clk.h` — واجهة الـ clock العامة لـ Linux (المستوى الأعلى)
- `include/asm/clock.h` — تعريفات architecture-specific لـ SH

**الـ Consumers:**
- `drivers/sh/pm_runtime.c` — Power Management Runtime يستخدم الـ clock framework
- `drivers/cpufreq/sh-cpufreq.c` — التحكم الديناميكي في تردد المعالج

---

### هيكل ملفات الـ Subsystem

```
include/linux/sh_clk.h          ← هذا الملف (الـ header)
drivers/sh/clk/
    core.c                       ← قلب الـ framework
    cpg.c                        ← تنفيذ MSTP/DIV4/DIV6/FSIDIV
arch/sh/kernel/cpu/sh4a/
    clock-sh7722.c               ← clock definitions لـ SH7722
    clock-sh7723.c               ← clock definitions لـ SH7723
    clock-sh7724.c               ← clock definitions لـ SH7724
drivers/sh/pm_runtime.c          ← Power Management
drivers/cpufreq/sh-cpufreq.c    ← CPU frequency scaling
```

---

### خلاصة

**الـ `sh_clk.h`** هو الـ header المركزي لإطار إدارة الـ clocks الخاص بمعالجات **SuperH** من Renesas. يوفر:
- تعريف موحّد لـ `struct clk` يمثّل أي clock في الشريحة
- نظام ops قابل للتوسعة لتوحيد واجهة enable/disable/recalc
- macros جاهزة لتعريف MSTP وDIV4 وDIV6 وFSIDIV clocks بأقل كود ممكن
- تكامل مع `cpufreq` لدعم DVFS

هذا الـ framework يسبق الـ **Common Clock Framework** الحديث في Linux (الموجود في `include/linux/clk-provider.h`)، وهو مصمم خصيصاً لمعالجات SH التي كانت شائعة في الأجهزة المدمجة اليابانية مطلع الألفية.
## Phase 2: شرح الـ SH Clock (sh_clk) Framework

---

### المشكلة: ليه أصلاً محتاجين clock framework خاص بـ SuperH؟

عشان تفهم المشكلة كويس، لازم تفهم الـ hardware أول.

الـ **SuperH (SH)** هي معالجات embedded يابانية من Renesas، بتتحكم فيها في أجهزة زي كاميرات، سيارات، وأجهزة صناعية. المشكلة مش في الـ clock نفسه — المشكلة في إن الـ SH chips عندها **نظام توزيع clock معقد جداً** اسمه الـ **CPG (Clock Pulse Generator)**، وده النظام اللي بيولد كل الـ clock signals الجوية داخل الـ chip.

الـ CPG بيعمل الآتي:
- بياخد clock خارجي واحد (الـ **crystal** أو الـ **PLL output**)
- بيمرره على سلسلة من الـ **dividers** (قواسم)
- كل قسمة بتطلع clock جديد لـ subsystem معين (CPU، bus، peripheral، USB، etc.)
- وعندك **MSTP (Module Stop)** registers بتقدر تقفل أي peripheral كليًا لتوفير الطاقة

المشكلة الكبيرة: الـ Linux kernel القديم ما كانش عنده طريقة موحدة للتعامل مع الـ SH clocks. كل driver كان بيتعامل مع الـ registers بشكل مباشر — **فوضى كاملة**:
- لو عندك device بتستخدم USB و serial في نفس الوقت، مين يقفل الـ clock؟
- لو غلطت في الـ register address، الـ chip هيعمل freeze.
- مش عارف مين فتح الـ clock ومين لسه محتاجه.

الحل اللي جاء به `sh_clk.h` هو إنه عمل **abstraction layer كامل** فوق الـ CPG hardware.

---

### الحل: الـ sh_clk Framework

الفكرة المحورية هي: **كل clock في النظام = object (`struct clk`) بيتتعامل معاه بـ reference counting وبـ operations table.**

بدل ما كل driver يكتب في الـ registers مباشرة، بيطلب الـ clock بالاسم، والـ framework بيتكفل بـ:
1. تشغيل/إيقاف الـ clock الصح في الـ register الصح
2. حساب الـ usecount عشان ميقفلش clock حد لسه بيستخدمه
3. حساب وتحديث الـ rate لكل clock لما الـ parent يتغير
4. بناء شجرة الـ clocks (parent → children) ونشر التغييرات

---

### التشبيه من الحياة الحقيقية

تخيل **شبكة مياه في عمارة كبيرة**:

```
[خزان سطح — المصدر الأصلي]
         |
    [عداد رئيسي]
    /           \
[خط أدوار 1-5]  [خط أدوار 6-10]
   /    \              |
[شقة A] [شقة B]    [شقة C]
```

- الـ **crystal oscillator** = خزان السطح (المصدر الأصلي)
- الـ **PLL** = مضخة الضغط (بتضاعف التردد)
- الـ **divider** = محبس التقسيم (بيخفض التدفق/التردد)
- الـ **MSTP** = صنبور الشقة (تفتحه للشقة اللي محتاجة مية بس)
- الـ **usecount** = عداد مين فاتح الصنبور، ما تقفلوش إلا لما كلهم يقولوا "خلصت"

لو شقة B مش محتاجة مية، تقفل صنبورها بس — ما تقطعش الخط كله. ولو شقة A وشقة B كلهم فاتحين، لما A يقفل، الخط لازم يفضل شغال عشان B لسه محتاجه.

---

### البنية الكبيرة — أين يقع الـ sh_clk في الـ kernel؟

```
┌─────────────────────────────────────────────────────────┐
│                   User Space / Apps                     │
└────────────────────────┬────────────────────────────────┘
                         │ /sys/kernel/debug/clk  (debugfs)
┌────────────────────────▼────────────────────────────────┐
│              Generic Kernel Subsystems                  │
│   (USB, Serial, I2C, SPI, display drivers, etc.)        │
│   يستخدموا:  clk_get()  clk_enable()  clk_set_rate()   │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│            Generic CLK API  (linux/clk.h)               │
│        الواجهة المشتركة لكل الـ platforms               │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│         SH Clock Framework  (linux/sh_clk.h)            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  MSTP Clocks │  │  DIV4 Clocks │  │  DIV6 Clocks │  │
│  │ (gate on/off)│  │(÷ 1,2,4,...) │  │(÷ 1..63)     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
│         └─────────────────┴──────────────────┘          │
│                           │                             │
│              ┌────────────▼────────────┐                │
│              │   struct clk tree       │                │
│              │  (parent/child links)   │                │
│              └────────────┬────────────┘                │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│           SH CPG Hardware  (Clock Pulse Generator)       │
│  ┌──────┐  ┌──────┐  ┌──────────┐  ┌─────────────────┐ │
│  │XTAL  │→ │ PLL  │→ │ Dividers │→ │  MSTP Registers │ │
│  │ osc  │  │      │  │(FRQCR,   │  │  (MSTPCR0-3)    │ │
│  └──────┘  └──────┘  │ etc.)    │  └─────────────────┘ │
│                       └──────────┘                      │
└─────────────────────────────────────────────────────────┘
```

---

### الـ Structs الأساسية وكيف ترتبط ببعض

#### 1. `struct clk` — قلب الـ framework

```c
struct clk {
    struct list_head    node;         /* رابط في القائمة العالمية للـ clocks */
    struct clk         *parent;       /* الـ clock الأب (مصدره) */
    struct clk        **parent_table; /* جدول الآباء المحتملين (للـ mux clocks) */
    unsigned short      parent_num;   /* عدد الآباء في الجدول */
    unsigned char       src_shift;    /* موضع bits الـ source في الـ register */
    unsigned char       src_width;    /* عرض bits الـ source في الـ register */
    struct sh_clk_ops  *ops;          /* جدول العمليات (enable/disable/recalc) */

    struct list_head    children;     /* قائمة الأبناء */
    struct list_head    sibling;      /* رابطه في قائمة أبناء أبيه */

    int                 usecount;     /* كم واحد فاتح الـ clock ده */

    unsigned long       rate;         /* التردد الحالي بالـ Hz */
    unsigned long       flags;        /* CLK_ENABLE_ON_INIT, CLK_ENABLE_REG_32BIT, إلخ */

    void __iomem       *enable_reg;   /* عنوان الـ register اللي بنكتب فيه */
    void __iomem       *status_reg;   /* register التحقق من الحالة (بعض SH chips) */
    unsigned int        enable_bit;   /* رقم الـ bit داخل الـ register */
    void __iomem       *mapped_reg;   /* الـ register بعد الـ mapping الفعلي */

    unsigned int        div_mask;     /* mask الـ divider bits */
    unsigned long       arch_flags;   /* bitmap للـ divider values المسموح بيها */
    void               *priv;         /* بيانات خاصة بكل نوع clock */
    struct clk_mapping *mapping;      /* معلومات الـ memory-mapped registers */
    struct cpufreq_frequency_table *freq_table; /* جدول الترددات المتاحة */
    unsigned int        nr_freqs;     /* عدد الترددات في الجدول */
};
```

#### 2. `struct sh_clk_ops` — جدول العمليات (الـ vtable)

```c
struct sh_clk_ops {
    void  (*init)(struct clk *clk);                        /* تهيئة أولية */
    int   (*enable)(struct clk *clk);                      /* تشغيل الـ clock */
    void  (*disable)(struct clk *clk);                     /* إيقاف الـ clock */
    unsigned long (*recalc)(struct clk *clk);              /* إعادة حساب التردد */
    int   (*set_rate)(struct clk *clk, unsigned long rate); /* تغيير التردد */
    int   (*set_parent)(struct clk *clk, struct clk *parent); /* تغيير المصدر */
    long  (*round_rate)(struct clk *clk, unsigned long rate); /* تقريب التردد */
};
```

ده نفس مبدأ الـ **vtable في C++** — بس في C. كل نوع clock (MSTP, DIV4, DIV6) بيملا الـ ops بطريقته الخاصة، لكن الكود العلوي بيتعامل معاهم بنفس الطريقة.

#### 3. `struct clk_mapping` — إدارة الـ MMIO

```c
struct clk_mapping {
    phys_addr_t   phys;   /* العنوان الفيزيائي للـ register */
    void __iomem *base;   /* العنوان الـ virtual بعد ioremap */
    unsigned long len;    /* حجم المنطقة المـ mapped */
    struct kref   ref;    /* reference count — متى نعمل iounmap؟ */
};
```

**الـ `kref` هنا ذكي جداً:** أكتر من clock ممكن يستخدم نفس الـ register block. بدل ما كل clock يعمل `ioremap` لوحده (تضييع في الـ memory)، الـ framework بيعمل `ioremap` مرة واحدة، ويحسب المستخدمين بـ `kref`. لما آخر واحد يخلص، بيعمل `iounmap` تلقائي.

---

### شجرة الـ Clocks — كيف ترتبط الـ structs

```
                    ┌─────────────────────┐
                    │  struct clk (PLL)   │
                    │  rate = 1000 MHz    │
                    │  parent = NULL      │
                    │  children ──────────┼──┐
                    └─────────────────────┘  │
                                             │
          ┌──────────────────────────────────┘
          │
          ▼
┌──────────────────────┐         ┌──────────────────────┐
│ struct clk (DIV4)    │         │ struct clk (DIV4)    │
│ rate = 500 MHz       │         │ rate = 250 MHz       │
│ parent = PLL clock   │         │ parent = PLL clock   │
│ div_mask = 0xF       │         │ div_mask = 0xF       │
│ children ────────────┼──┐      │ children ────────────┼──┐
│ sibling ─────────────┼──┼──────┼◄ sibling             │  │
└──────────────────────┘  │      └──────────────────────┘  │
                          │                                 │
          ┌───────────────┘           ┌─────────────────────┘
          ▼                           ▼
┌──────────────────────┐   ┌──────────────────────────┐
│ struct clk (MSTP)    │   │ struct clk (MSTP)        │
│ "USB clock"          │   │ "I2C clock"              │
│ rate = 500 MHz       │   │ rate = 250 MHz           │
│ enable_reg = 0xFF...│   │ enable_reg = 0xFF...     │
│ enable_bit = 14      │   │ enable_bit = 9           │
│ usecount = 2         │   │ usecount = 0             │
│ ops->enable() ───────┼──►│ [يكتب bit في MSTPCR]    │
└──────────────────────┘   └──────────────────────────┘
```

**ملاحظة مهمة على الـ `sibling`:** في الـ kernel، الـ children مش array — دي linked list. كل child عنده `sibling` يربطه بالـ child التاني اللي عنده نفس الأب. ده بيسمح بإضافة/حذف clocks ديناميكياً.

---

### أنواع الـ Clocks في الـ SH Framework

#### النوع الأول: MSTP (Module Stop)

**الـ MSTP** هو أبسط نوع — زي مفتاح كهرباء. bit = 0 → الـ module شغال، bit = 1 → الـ module متوقف.

```c
/* تعريف clock لـ USB controller */
#define SH_CLK_MSTP32(_p, _r, _b, _f)
    SH_CLK_MSTP(_p, _r, _b, 0, _f | CLK_ENABLE_REG_32BIT)

/* مثال عملي: */
static struct clk mstp_clks[] = {
    /* USB clock: أبوه الـ peripheral clock، register MSTPCR2، bit 14 */
    [MSTP214] = SH_CLK_MSTP32(&p_clk, MSTPCR2, 14, 0),
    /* I2C clock: نفس الـ register، bit 9 */
    [MSTP209] = SH_CLK_MSTP32(&p_clk, MSTPCR2,  9, 0),
};
```

**الـ `status_reg`** في `SH_CLK_MSTP32_STS`: بعض الـ SH chips (زي R-Car) عندها register منفصل للتحقق إن الـ module فعلاً اتوقف — مش بس كتبت في الـ enable register. الكود بيعمل polling على الـ status bit عشان يتأكد.

```c
/* مع status register */
#define SH_CLK_MSTP32_STS(_p, _r, _b, _s, _f)
    SH_CLK_MSTP(_p, _r, _b, _s, _f | CLK_ENABLE_REG_32BIT)
```

#### النوع الثاني: DIV4 Clocks

الـ **DIV4** بيقسم تردد الـ parent على واحدة من القيم: 1، 2، 4، 6، 8، 12، 16، 18، 24، 32، 36، 48. الـ `arch_flags` هو **bitmap** بيحدد أنهي قيم مسموح بيها.

```c
#define SH_CLK_DIV4(_parent, _reg, _shift, _div_bitmap, _flags)
{
    .parent      = _parent,
    .enable_reg  = (void __iomem *)_reg,
    .enable_bit  = _shift,       /* موضع الـ field في الـ register */
    .arch_flags  = _div_bitmap,  /* bitmap: bit N = قيمة القسمة N مسموح بيها */
    .div_mask    = SH_CLK_DIV4_MSK, /* 0xF = 4 bits */
    .flags       = _flags,
}
```

الـ `SH_CLK_DIV4_MSK` = `(1 << 4) - 1` = `0xF` — يعني 4 bits عشان تشيل 16 قيمة مختلفة.

#### النوع الثالث: DIV6 Clocks

الـ **DIV6** أكتر مرونة — بيقسم من 1 لـ 64. الـ `SH_CLK_DIV6_MSK` = `(1 << 6) - 1` = `0x3F` = 6 bits.

التميز المهم في DIV6 هو الـ **parent mux** — ممكن يبدل مصدره بين عدة clocks:

```c
#define SH_CLK_DIV6_EXT(_reg, _flags, _parents, _num_parents, _src_shift, _src_width)
{
    .enable_reg   = (void __iomem *)_reg,
    .flags        = _flags | CLK_MASK_DIV_ON_DISABLE,
    .div_mask     = SH_CLK_DIV6_MSK,
    .parent_table = _parents,     /* مصفوفة الآباء المحتملين */
    .parent_num   = _num_parents, /* عددهم */
    .src_shift    = _src_shift,   /* موضع bits الاختيار في الـ register */
    .src_width    = _src_width,   /* عرض bits الاختيار */
}
```

الـ `CLK_MASK_DIV_ON_DISABLE` يعني: لما تعطل الـ clock، صفر الـ divider bits كمان — بعض SH chips بتشترط كده.

#### النوع الرابع: FSIDIV (FSI Fractional Divider)

ده خاص جداً بالـ **FSI (FIFO-buffered Serial Interface)** — واجهة صوت. ده divider كسري (fractional) لأن الصوت محتاج ترددات دقيقة جداً (44100 Hz، 48000 Hz). الـ `.enable_reg` في التعريف بيتحول لـ `.mapping` بعد الـ register من `sh_clk_fsidiv_register()`.

---

### جدول الـ Flags وأهميتها

| الـ Flag | القيمة | المعنى |
|----------|--------|--------|
| `CLK_ENABLE_ON_INIT` | `BIT(0)` | شغل الـ clock أثناء boot حتى لو ما طلبه حد |
| `CLK_ENABLE_REG_32BIT` | `BIT(1)` | الـ register بـ 32-bit access |
| `CLK_ENABLE_REG_16BIT` | `BIT(2)` | الـ register بـ 16-bit access |
| `CLK_ENABLE_REG_8BIT` | `BIT(3)` | الـ register بـ 8-bit access |
| `CLK_MASK_DIV_ON_DISABLE` | `BIT(4)` | صفر الـ divider لما تعطل |

الـ access size مهم جداً — بعض الـ registers على SH chips ما بتقبلش 32-bit write، لازم 16-bit. لو غلطت، الـ write بيتجاهل أو الـ chip بيعمل undefined behavior.

---

### الـ Rate Table — كيف تحسب الـ DIV4 Rate

```c
struct clk_div_mult_table {
    unsigned int *divisors;    /* مصفوفة قيم القسمة: {1, 2, 4, 8, ...} */
    unsigned int  nr_divisors; /* عددها */
    unsigned int *multipliers; /* مصفوفة معاملات الضرب (للـ PLL) */
    unsigned int  nr_multipliers;
};
```

الـ `clk_rate_table_build()` بتبني جدول بكل الترددات الممكنة من ضرب parent_rate في كل multiplier وقسمته على كل divisor. النتيجة بتتخزن في `cpufreq_frequency_table` — نفس الجدول اللي الـ cpufreq framework بيستخدمه لتغيير سرعة الـ CPU!

ده الربط الذكي بين الـ **clock management** والـ **power management (cpufreq)** — لما الـ CPU عايز يغير تردده، بيمشي على الجدول ده ويختار أقرب تردد متاح.

---

### الـ Rate Propagation — نشر التغيير في الشجرة

ده من أهم مفاهيم الـ framework. لما تغير تردد clock:

```
propagate_rate(parent_clk)
        │
        ├─► للكل child في children:
        │       child->rate = child->ops->recalc(child)
        │       propagate_rate(child)  ← تكرار recursive
        │
        └─► لو child عنده ops->recalc = followparent_recalc:
                child->rate = child->parent->rate
                (يورث التردد من أبيه مباشرة)
```

يعني لو غيرت الـ PLL rate، التغيير بينزل تلقائياً لكل الـ clocks اللي تحته في الشجرة. ده زي لما تغير ضغط المياه في الخزان الرئيسي — كل الشقق بتتأثر.

---

### الـ Clock Lookup — CLKDEV Macros

```c
#define CLKDEV_CON_ID(_id, _clk)         { .con_id = _id, .clk = _clk }
#define CLKDEV_DEV_ID(_id, _clk)         { .dev_id = _id, .clk = _clk }
#define CLKDEV_ICK_ID(_cid, _did, _clk)  { .con_id = _cid, .dev_id = _did, .clk = _clk }
```

الـ driver لما بيطلب `clk_get(dev, "usb")`:
1. الـ kernel بيدور في جدول الـ lookup
2. بيطابق الـ `dev_id` (اسم الـ device) والـ `con_id` (اسم الـ connection)
3. بيرجع الـ `struct clk *` المقابل

ده بيفصل تعريف الـ clocks (في board file) عن استخدامها (في drivers) — الـ driver مش محتاج يعرف أي hardware موجود، بس يطلب clock بالاسم.

---

### الصورة الكاملة: من الـ crystal للـ driver

```
Crystal (33 MHz)
     │
     ▼
┌──────────┐
│  PLL × N │  → مثلاً × 30 = 1000 MHz
└────┬─────┘
     │
     ├──► DIV4 clock (÷4 = 250 MHz) ──► CPU bus clock
     │         └── MSTP: DMA engine
     │         └── MSTP: Memory controller
     │
     ├──► DIV4 clock (÷8 = 125 MHz) ──► Peripheral clock (P clock)
     │         └── MSTP: UART  ─────────────────► serial driver
     │         └── MSTP: I2C   ─────────────────► i2c driver
     │         └── MSTP: SPI   ─────────────────► spi driver
     │
     └──► DIV6 clock (÷2..64) ──► USB clock (60 MHz fixed)
               └── MSTP: USB controller ────────► usb driver

كل driver:
  clk = clk_get(dev, "fck");   // اطلب clock بالاسم
  clk_enable(clk);              // شغله (ويزيد usecount)
  ...
  clk_disable(clk);             // خلصت منه (يقلل usecount)
  clk_put(clk);                 // حرر المرجع
```

---

### الـ `usecount` — قلب الـ Clock Gating

```c
int usecount; /* في struct clk */
```

الـ usecount بيشتغل كالآتي:
- `clk_enable(clk)`: يزيد usecount، لو بقى 1 → يشغل الـ hardware فعلاً
- `clk_disable(clk)`: يقلل usecount، لو وصل 0 → يوقف الـ hardware فعلاً

**ليه ده مهم؟** لأن أكتر من driver ممكن يشترك في نفس الـ clock. مثلاً الـ DMA engine والـ UART ممكن يستخدموا نفس الـ peripheral clock. لو الـ UART شغّل الـ clock وبعدين DMA اشتغل كمان، usecount = 2. لو UART وقف، usecount يبقى 1، والـ clock يفضل شغال عشان DMA لسه محتاجه.

هذا هو الـ **reference counting** المطبق على الـ hardware — نفس مبدأ الـ `kref` في الـ `struct clk_mapping`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Macros — Cheatsheet

#### الـ CLK Flags (حقل `flags` في `struct clk`)

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `CLK_ENABLE_ON_INIT` | `BIT(0)` | شغّل الـ clock تلقائيًا عند بدء التشغيل |
| `CLK_ENABLE_REG_32BIT` | `BIT(1)` | الـ register يُقرأ/يُكتب بـ 32 bit (الافتراضي) |
| `CLK_ENABLE_REG_16BIT` | `BIT(2)` | الـ register يُقرأ/يُكتب بـ 16 bit |
| `CLK_ENABLE_REG_8BIT` | `BIT(3)` | الـ register يُقرأ/يُكتب بـ 8 bit |
| `CLK_MASK_DIV_ON_DISABLE` | `BIT(4)` | عند إيقاف الـ clock، اكتب قيمة الـ divider في الـ register |
| `CLK_ENABLE_REG_MASK` | `BIT(1)\|BIT(2)\|BIT(3)` | mask لتحديد حجم الـ access |

#### الـ Divider Masks

| الـ Macro | الحساب | القيمة |
|---|---|---|
| `SH_CLK_DIV_MSK(4)` | `(1<<4)-1` | `0xF` — 4 bits للـ DIV4 |
| `SH_CLK_DIV4_MSK` | نفس السابق | `0xF` |
| `SH_CLK_DIV_MSK(6)` | `(1<<6)-1` | `0x3F` — 6 bits للـ DIV6 |
| `SH_CLK_DIV6_MSK` | نفس السابق | `0x3F` |

#### الـ Rate Change Notifier Events (من `linux/clk.h`)

| الـ Event | القيمة | المعنى |
|---|---|---|
| `PRE_RATE_CHANGE` | `BIT(0)` | قبل تغيير الـ rate — أوقف عملك فورًا |
| `POST_RATE_CHANGE` | `BIT(1)` | بعد نجاح التغيير |
| `ABORT_RATE_CHANGE` | `BIT(2)` | فشل التغيير، عُد لحالتك السابقة |

#### الـ Clock Definition Macros — Cheatsheet

| الـ Macro | نوع الـ Clock | ملاحظة |
|---|---|---|
| `SH_CLK_MSTP32(p,r,b,f)` | MSTP بـ 32bit | لا يوجد status register |
| `SH_CLK_MSTP32_STS(p,r,b,s,f)` | MSTP بـ 32bit + status | فيه status register |
| `SH_CLK_MSTP16(p,r,b,f)` | MSTP بـ 16bit | |
| `SH_CLK_MSTP8(p,r,b,f)` | MSTP بـ 8bit | |
| `SH_CLK_DIV4(parent,reg,shift,bitmap,flags)` | قسمة 4-bit | `div_mask = 0xF` |
| `SH_CLK_DIV6(parent,reg,flags)` | قسمة 6-bit + أب واحد | `CLK_MASK_DIV_ON_DISABLE` تلقائي |
| `SH_CLK_DIV6_EXT(reg,flags,parents,n,shift,width)` | DIV6 + اختيار الأب | عدة آباء مُحتملين |
| `SH_CLK_FSIDIV(reg,parent)` | FSI divider خاص | الـ `enable_reg` يتحول لـ `mapping` |

#### الـ CLKDEV Registration Macros

| الـ Macro | متى تستخدمه |
|---|---|
| `CLKDEV_CON_ID(id, clk)` | ربط الـ clock باسم connection فقط |
| `CLKDEV_DEV_ID(id, clk)` | ربط الـ clock باسم device فقط |
| `CLKDEV_ICK_ID(cid, did, clk)` | ربط باسم connection + اسم device معًا |

---

### 1. الـ Structs المهمة

#### `struct clk_mapping`

**الغرض:** تخزين معلومات الـ memory-mapped register الخاص بالـ clock — يعني العنوان الفيزيائي (`phys`) ومكانه في الذاكرة الافتراضية (`base`) وحجمه. والأهم: يستخدم **الـ `kref`** للـ reference counting، بحيث لو أكثر من clock يشارك نفس الـ register region، ما نحتاج نعمل `ioremap` مرتين.

تخيّله كـ "مستأجر مشترك" — أول واحد يحجز الشقة، والباقين يدفعون إيجارًا مشتركًا، وآخر واحد يخرج يُلغي الحجز.

```c
struct clk_mapping {
    phys_addr_t   phys;   /* العنوان الفيزيائي للـ register */
    void __iomem *base;   /* العنوان الافتراضي بعد ioremap */
    unsigned long len;    /* حجم المنطقة بالـ bytes */
    struct kref   ref;    /* عدد من يستخدمون هذا الـ mapping */
};
```

| الحقل | النوع | الوظيفة |
|---|---|---|
| `phys` | `phys_addr_t` | العنوان الحقيقي في ذاكرة الـ hardware |
| `base` | `void __iomem *` | المؤشر للـ virtual address بعد `ioremap` |
| `len` | `unsigned long` | حجم المنطقة |
| `ref` | `struct kref` | عداد مشتركين — اخر واحد يُطفئ الضوء |

**علاقته بالآخرين:** الـ `struct clk` يحمل مؤشر `*mapping` إليه. عند تسجيل FSI clock، الـ `enable_reg` يتحول إلى pointer لهذا الـ mapping.

---

#### `struct sh_clk_ops`

**الغرض:** جدول العمليات (vtable) — يعني بدل ما تكتب كود مختلف لكل نوع clock، كل نوع يعبئ هذا الجدول بدواله الخاصة، والـ core ينادي منه. مثل "قائمة الموظفين مع مهاراتهم" — الـ core بس يقول "من يعرف `enable`؟" ويروح ينادي الصح.

```c
struct sh_clk_ops {
#ifdef CONFIG_SH_CLK_CPG_LEGACY
    void (*init)(struct clk *clk);           /* تهيئة أولية (legacy فقط) */
#endif
    int  (*enable)(struct clk *clk);         /* تشغيل الـ clock */
    void (*disable)(struct clk *clk);        /* إيقاف الـ clock */
    unsigned long (*recalc)(struct clk *clk); /* احسب الـ rate الحالي */
    int  (*set_rate)(struct clk *clk,
                     unsigned long rate);    /* اضبط الـ rate */
    int  (*set_parent)(struct clk *clk,
                       struct clk *parent); /* غيّر الأب */
    long (*round_rate)(struct clk *clk,
                       unsigned long rate); /* قرّب الـ rate لأقرب قيمة ممكنة */
};
```

| الدالة | متى تُنادى | ماذا تفعل |
|---|---|---|
| `init` | مرة واحدة (legacy) | تهيئة hardware أولية |
| `enable` | عند `clk_enable()` | تشغيل الـ clock |
| `disable` | عند `clk_disable()` | إيقاف الـ clock |
| `recalc` | عند `propagate_rate()` | إعادة حساب الـ rate من الـ hardware |
| `set_rate` | عند `clk_set_rate()` | كتابة الـ rate الجديد للـ hardware |
| `set_parent` | عند `clk_set_parent()` | تغيير مصدر الـ clock |
| `round_rate` | عند `clk_round_rate()` | "لو طلبت X Hz، هتاخد كام فعلاً؟" |

**علاقته بالآخرين:** الـ `struct clk` يحمل مؤشر `*ops` لهذا الـ struct. كل نوع clock (MSTP, DIV4, DIV6) يعبئ نسخته الخاصة منه.

---

#### `struct clk` — القلب النابض

**الغرض:** يمثّل clock واحد في النظام. كل oscillator أو PLL أو divider أو gate له كائن من هذا النوع. يعيش في linked list عالمية، ومرتبط بأبيه وأبنائه.

تخيّله كـ "موزّع كهرباء" — يأخذ تيار من أبيه، يعدّله، ويوزّعه على أبنائه.

```c
struct clk {
    /* --- الربط بقائمة الـ clocks العالمية --- */
    struct list_head   node;          /* حلقة في القائمة العالمية */

    /* --- شجرة الأب/الأبناء --- */
    struct clk        *parent;        /* الأب الحالي */
    struct clk       **parent_table;  /* قائمة آباء محتملين (DIV6_EXT) */
    unsigned short     parent_num;    /* عدد الآباء في الجدول */
    unsigned char      src_shift;     /* موضع bits الاختيار في الـ register */
    unsigned char      src_width;     /* عدد bits الاختيار */
    struct sh_clk_ops *ops;           /* جدول العمليات */

    struct list_head   children;      /* رأس قائمة الأبناء */
    struct list_head   sibling;       /* حلقة في قائمة إخوته */

    /* --- الاستخدام والـ Rate --- */
    int            usecount;          /* كم جهاز يستخدمني الآن */
    unsigned long  rate;              /* الـ rate الحالي بالـ Hz */
    unsigned long  flags;             /* CLK_ENABLE_ON_INIT وغيرها */

    /* --- الـ Hardware Registers --- */
    void __iomem  *enable_reg;        /* register الـ enable/disable */
    void __iomem  *status_reg;        /* register للتحقق من الحالة (اختياري) */
    unsigned int   enable_bit;        /* رقم الـ bit داخل enable_reg */
    void __iomem  *mapped_reg;        /* الـ register الفعلي بعد التحويل */

    /* --- الـ Divider --- */
    unsigned int   div_mask;          /* mask لبيتات الـ divider (0xF أو 0x3F) */
    unsigned long  arch_flags;        /* bitmap للـ divisors المتاحة */

    /* --- البيانات الإضافية --- */
    void                          *priv;       /* بيانات خاصة بالـ driver */
    struct clk_mapping            *mapping;    /* الـ register mapping المشترك */
    struct cpufreq_frequency_table *freq_table; /* جدول الـ frequencies المدعومة */
    unsigned int                   nr_freqs;   /* عدد الـ entries في الجدول */
};
```

| الحقل | وظيفته بالعامية |
|---|---|
| `node` | "شنطتي في القطار العالمي" — يربطني بقائمة كل الـ clocks |
| `parent` | "أبوي اللي يعطيني التيار" |
| `parent_table` | "قائمة آبائي المحتملين" (للـ mux clocks) |
| `usecount` | "كم جهاز يحتاجني؟" — لو صفر، ممكن أنام |
| `rate` | "سرعتي الحالية بالـ Hz" |
| `enable_reg` + `enable_bit` | "مفتاح الإضاءة — الـ register والـ bit" |
| `status_reg` | "هل أنا فعلاً شغال؟" — بعض الـ hardware يتأخر |
| `div_mask` | "القناع اللي يحدد bits قسمتي" |
| `arch_flags` | "bitmap بالقسمات المسموح بها" |
| `mapping` | "مؤشر لمعلومات الـ ioremap المشتركة" |
| `freq_table` | "جدول بالسرعات المدعومة لـ cpufreq" |

---

#### `struct clk_div_mult_table`

**الغرض:** جدول يحتوي على كل قيم الـ dividers والـ multipliers الممكنة لـ clock معين. يُستخدم لبناء جدول الـ frequencies وإيجاد أقرب rate.

```c
struct clk_div_mult_table {
    unsigned int *divisors;       /* مصفوفة قيم القسمة [1, 2, 4, 8 ...] */
    unsigned int  nr_divisors;    /* عدد عناصر المصفوفة */
    unsigned int *multipliers;    /* مصفوفة قيم الضرب (اختياري) */
    unsigned int  nr_multipliers; /* عدد عناصر مصفوفة الضرب */
};
```

| الحقل | الوظيفة |
|---|---|
| `divisors` | مصفوفة أرقام القسمة الممكنة |
| `nr_divisors` | حجم المصفوفة |
| `multipliers` | مصفوفة أرقام الضرب (لبعض الـ PLLs) |
| `nr_multipliers` | حجمها |

**علاقته بالآخرين:** يُستخدم داخل `struct clk_div_table` ويُمرّر لـ `clk_rate_table_build()`.

---

#### `struct clk_div_table` (المعروف أيضًا بـ `clk_div4_table`)

**الغرض:** جدول التحكم في الـ DIV4 clocks. يجمع بين جدول القسمة/الضرب وـ callback اختياري يُنادى عند تغيير الـ rate (مثلاً لتشغيل sequence معين في الـ hardware).

```c
struct clk_div_table {
    struct clk_div_mult_table *div_mult_table; /* جدول القيم الممكنة */
    void (*kick)(struct clk *clk);             /* callback عند تغيير الـ rate */
};
/* المُرادف: */
#define clk_div4_table clk_div_table
```

| الحقل | الوظيفة |
|---|---|
| `div_mult_table` | مؤشر للجدول الفعلي بالقيم |
| `kick` | callback يُنادى بعد تغيير الـ rate (لبعض الـ SoCs) |

---

### 2. مخطط علاقات الـ Structs

```
                    ┌─────────────────────────────────────────────┐
                    │              struct clk                      │
                    │                                             │
                    │  node ────────────────► [global clk list]  │
                    │  parent ──────────────► struct clk          │
                    │  parent_table ─────────► struct clk *[]     │
                    │  ops ──────────────────► struct sh_clk_ops  │
                    │  mapping ──────────────► struct clk_mapping │
                    │  freq_table ───────────► cpufreq_freq_table │
                    │  children ─────────────► [list of children] │
                    │  sibling ──────────────► [sibling in list]  │
                    └─────────────────────────────────────────────┘
                              │                    │
                    ┌─────────▼──────┐   ┌────────▼────────────────┐
                    │ struct         │   │ struct clk_mapping       │
                    │ sh_clk_ops     │   │                         │
                    │                │   │  phys ────► HW address  │
                    │  enable()      │   │  base ────► ioremap ptr │
                    │  disable()     │   │  len                    │
                    │  recalc()      │   │  ref ─────► kref        │
                    │  set_rate()    │   │  (shared by N clocks)   │
                    │  set_parent()  │   └─────────────────────────┘
                    │  round_rate()  │
                    └────────────────┘

           ┌─────────────────────────────────────────┐
           │          struct clk_div_table            │
           │  div_mult_table ──► struct clk_div_mult_table │
           │                      ┌──────────────────────┐ │
           │                      │ divisors[]           │ │
           │                      │ nr_divisors          │ │
           │                      │ multipliers[]        │ │
           │                      │ nr_multipliers       │ │
           │                      └──────────────────────┘ │
           │  kick() ──► callback fn                   │
           └─────────────────────────────────────────┘
```

#### شجرة الـ Parent/Child (مثال واقعي)

```
PLL_clock (root)
    │
    ├── DIV4_clock_A  (parent = PLL, children list)
    │       │
    │       ├── MSTP_clock_1  (sibling of MSTP_2)
    │       └── MSTP_clock_2  (sibling of MSTP_1)
    │
    └── DIV6_clock_B  (parent = PLL)
            │
            └── MSTP_clock_3
```

كل `sibling` هو حلقة في قائمة `children` الخاصة بالأب — يعني الأب عنده `children` كـ `list_head`، وكل ابن يربط `sibling` الخاص به بهذه القائمة.

---

### 3. دورة حياة الـ Clock — من الولادة للممات

```
[تعريف Clock في الـ board file]
        │
        │  SH_CLK_MSTP32(&parent, REG, BIT, FLAGS)
        │  ← تعبئة struct clk بالقيم الأولية
        ▼
[التسجيل في الـ kernel]
        │
        │  sh_clk_mstp_register(clks, nr)
        │    └─► clk_register(clk)   ← لكل clock
        │           ├─► إضافة لقائمة عالمية (node)
        │           ├─► ربط بالأب (parent)
        │           ├─► إضافة لقائمة أبناء الأب (sibling → children)
        │           └─► recalc() لحساب الـ rate الأولي
        ▼
[الاستخدام من الـ Driver]
        │
        │  clk = clk_get(dev, "my_clock")
        │    └─► البحث في قائمة clkdev
        │
        │  clk_enable(clk)
        │    ├─► usecount++
        │    ├─► لو usecount == 1: enable الأب أولاً
        │    └─► ops->enable(clk) ← كتابة الـ register
        │
        │  [... الـ driver يعمل ...]
        │
        │  clk_disable(clk)
        │    ├─► usecount--
        │    ├─► لو usecount == 0: ops->disable(clk)
        │    └─► disable الأب لو usecount الأب = 0
        │
        │  clk_put(clk)
        ▼
[إلغاء التسجيل]
        │
        │  clk_unregister(clk)
        │    ├─► إزالة من القائمة العالمية
        │    ├─► إزالة من قائمة أبناء الأب
        │    └─► تحرير الموارد
        ▼
    [انتهى]
```

#### دورة حياة الـ `clk_mapping` (مع الـ kref)

```
أول clock يحتاج mapping:
    ioremap(phys, len) → base
    kref_init(&mapping->ref)  ← ref = 1

clock ثانٍ يشارك نفس الـ region:
    kref_get(&mapping->ref)   ← ref = 2

clock أول يُلغى:
    kref_put(&mapping->ref, release_fn)  ← ref = 1
    (لا يُحرر الـ mapping)

clock ثاني يُلغى:
    kref_put(&mapping->ref, release_fn)  ← ref = 0
    → release_fn() تُنادى تلقائيًا
    → iounmap(mapping->base)
    → kfree(mapping)
```

---

### 4. مخططات تدفق العمليات

#### 4.1 تدفق `clk_enable()`

```
driver calls clk_enable(clk)
    │
    ├─► هل usecount > 0؟
    │       نعم: usecount++, return 0  (ما في حاجة نشغّل)
    │       لا: تابع
    │
    ├─► هل للـ clock أب؟
    │       نعم: clk_enable(parent)  ← recursive
    │
    ├─► ops->enable(clk)
    │       ├─► (MSTP) اكتب 0 على enable_bit في enable_reg
    │       │         بحجم الـ access (8/16/32 bit)
    │       └─► تحقق من status_reg لو موجود
    │
    ├─► usecount++
    └─► return 0
```

#### 4.2 تدفق `clk_set_rate()`

```
driver calls clk_set_rate(clk, new_rate)
    │
    ├─► ops->round_rate(clk, new_rate)
    │       └─► clk_rate_table_round() أو clk_rate_div_range_round()
    │               └─► البحث في freq_table أو حساب أقرب divisor
    │               └─► return actual_rate
    │
    ├─► ops->set_rate(clk, actual_rate)
    │       ├─► احسب قيمة الـ divisor من actual_rate
    │       ├─► اكتب القيمة في enable_reg (مع enable_bit كـ shift)
    │       └─► لو `kick` موجود في div_table: kick(clk)
    │
    ├─► clk->rate = actual_rate
    │
    └─► propagate_rate(clk)
            └─► لكل child في children:
                    child->ops->recalc(child)
                    child->rate = new_child_rate
                    propagate_rate(child)  ← recursive
```

#### 4.3 تدفق `sh_clk_mstp_register()`

```
sh_clk_mstp_register(clks[], nr)
    │
    ├─► for i in 0..nr:
    │       clk = &clks[i]
    │       clk->ops = &sh_clk_mstp_clk_ops  ← تعيين الـ vtable
    │       clk_register(clk)
    │           ├─► list_add(&clk->node, &clock_list)
    │           ├─► clk_reparent(clk, clk->parent)
    │           │       ├─► list_add(&clk->sibling, &parent->children)
    │           │       └─► clk->parent = parent
    │           └─► followparent_recalc(clk) أو ops->recalc(clk)
    │
    └─► lو CLK_ENABLE_ON_INIT في flags:
            clk_enable_init_clocks()
                └─► for each clock with flag: clk_enable(clk)
```

#### 4.4 تدفق `clk_set_parent()` للـ DIV6_EXT

```
driver calls clk_set_parent(clk, new_parent)
    │
    ├─► ops->set_parent(clk, new_parent)
    │       ├─► ابحث عن new_parent في parent_table[]
    │       ├─► احسب قيمة bits = index في الجدول
    │       ├─► اكتب bits في enable_reg عند src_shift بعرض src_width
    │       └─► return 0
    │
    ├─► clk_reparent(clk, new_parent)
    │       ├─► أزل clk->sibling من قائمة أبنه القديم
    │       ├─► أضف clk->sibling لقائمة أبناء new_parent
    │       └─► clk->parent = new_parent
    │
    └─► propagate_rate(clk)
            └─► أعد حساب الـ rate من الأب الجديد
```

---

### 5. استراتيجية الـ Locking

نظام الـ SH clock يعتمد على **قفل عالمي واحد** (clock_lock) يحمي كل عمليات الـ clock. هذا نهج بسيط مقارنة بنظام الـ CCF الحديث.

#### جدول الـ Locks

| ما الذي يُحمى | الـ Lock المستخدم | نوع العملية |
|---|---|---|
| قائمة `clock_list` العالمية | `clock_list_sem` (rwlock) | read عند بحث, write عند register |
| `usecount` لكل clock | `enable_lock` (spinlock) | كل `enable`/`disable` |
| `clk->rate` و `clk->parent` | `clock_lock` (mutex) | `set_rate`, `set_parent` |
| `clk_mapping->ref` | `kref` الداخلي (atomic) | أي عملية على الـ mapping |

#### ترتيب القفل (Lock Ordering)

```
clock_lock (mutex)          ← الأعلى — حجب لعمليات set_rate/set_parent
    └─► enable_lock (spinlock)  ← الأدنى — حجب لعمليات enable/disable
            └─► hardware register access
```

**قاعدة مهمة:** لا تمسك `enable_lock` وبعدين تحاول تاخذ `clock_lock` — هذا deadlock مضمون. الترتيب دائمًا من الأعلى للأسفل.

#### ملاحظة على `clk_enable()` من atomic context

**الـ `ops->enable()`** في SH يكتب الـ registers مباشرة، وهذا ممكن من atomic context (interrupt handlers) لأنه لا يستخدم `mutex`. لكن `clk_set_rate()` و`clk_set_parent()` يستخدمان `mutex`، فـ **ممنوع** نداؤهما من interrupt context.

#### الـ `kref` في `clk_mapping`

الـ `kref` في حد ذاته atomic — يستخدم `refcount_t` الذي مبني على atomic operations. لا يحتاج قفل خارجي لـ `kref_get/put`. لكن القرار "هل أعمل ioremap جديد أم أشارك الموجود" يحتاج قفل خارجي لتجنب race condition.
## Phase 4: شرح الـ Functions

### جدول الملخص

| Function | Type | Purpose |
|----------|------|---------|
| `followparent_recalc()` | extern | يحسب الـ rate للـ clock من الـ parent مباشرةً |
| `recalculate_root_clocks()` | extern | يعيد حساب الـ rate لكل الـ root clocks |
| `propagate_rate()` | extern | ينشر تغيير الـ rate من الـ clock لكل أبنائه |
| `clk_reparent()` | extern | يغير الـ parent clock لـ clock معين |
| `clk_register()` | extern | يسجل الـ clock في النظام |
| `clk_unregister()` | extern | يحذف الـ clock من النظام |
| `clk_enable_init_clocks()` | extern | يشغّل كل الـ clocks التي عليها `CLK_ENABLE_ON_INIT` |
| `clk_rate_table_build()` | extern | يبني جدول الـ frequencies الممكنة للـ clock |
| `clk_rate_table_round()` | extern | يجد أقرب rate موجود في الجدول |
| `clk_rate_table_find()` | extern | يجد index الـ entry المناسب في جدول الـ frequencies |
| `clk_rate_div_range_round()` | extern | يحسب أقرب rate ممكن باستخدام نطاق من القسمة |
| `clk_rate_mult_range_round()` | extern | يحسب أقرب rate ممكن باستخدام نطاق من الضرب |
| `sh_clk_mstp_register()` | extern | يسجل مجموعة MSTP clocks |
| `sh_clk_mstp32_register()` | static inline / deprecated | wrapper قديم لـ `sh_clk_mstp_register()` |
| `sh_clk_div4_register()` | extern | يسجل مجموعة DIV4 clocks |
| `sh_clk_div4_enable_register()` | extern | يسجل DIV4 clocks مع دعم الـ enable/disable |
| `sh_clk_div4_reparent_register()` | extern | يسجل DIV4 clocks مع دعم تغيير الـ parent |
| `sh_clk_div6_register()` | extern | يسجل مجموعة DIV6 clocks |
| `sh_clk_div6_reparent_register()` | extern | يسجل DIV6 clocks مع دعم تغيير الـ parent |
| `sh_clk_fsidiv_register()` | extern | يسجل FSI divider clocks |

---

## المجموعة الأولى: Core Clock Lifecycle — التسجيل والحذف

### فكرة المجموعة

تخيل الـ clock tree زي شجرة عائلة. كل clock له أب (parent)، وممكن يكون له أبناء (children). قبل ما أي clock يشتغل، لازم يتسجل في النظام — زي ما الموظف يسجل دخوله في الشركة. هذه المجموعة هي نقطة الدخول والخروج لكل clock في نظام SH (SuperH).

---

### `clk_register()`

```c
int clk_register(struct clk *clk);
```

**ما يفعله:**
يضيف الـ `clk` في القائمة العالمية للـ clocks في الكيرنل. بيربطه بالـ parent المحدد في `clk->parent`، ويضيف الـ clock كـ child في قائمة أبناء الـ parent. كمان بيستدعي `recalc` عشان يحسب الـ rate الحالي للـ clock لحظة التسجيل.

**البارامترات:**
- `clk`: مؤشر لـ `struct clk` المراد تسجيله، لازم يكون مملوء بالحقول المطلوبة (على الأقل `ops` أو `parent`).

**القيمة المُرجَعة:**
- `0` عند النجاح.
- قيمة سالبة (errno) عند الفشل، مثلاً لو الـ `clk` هو `NULL`.

**تفاصيل مهمة:**
- يُستدعى هذا عادةً من كود الـ board setup أو الـ platform driver أثناء الـ boot.
- الـ locking: يحتاج قفل داخلي على قائمة الـ clocks (عادةً `spinlock` أو `mutex` حسب السياق في `clk.c`).
- بعد التسجيل، الـ clock يصبح مرئياً للنظام ويمكن لأي driver يطلبه.

**من يستدعيه:**
يُستدعى من دوال التسجيل المتخصصة مثل `sh_clk_mstp_register()` و `sh_clk_div4_register()` وغيرها، أو مباشرةً من كود platform الـ SH.

---

### `clk_unregister()`

```c
void clk_unregister(struct clk *clk);
```

**ما يفعله:**
عكس `clk_register()` تماماً — يشيل الـ clock من القائمة العالمية، ويفصله عن الـ parent، ويزيله من قائمة الأبناء. زي ما الموظف يسجل خروجه ويعطي شارته.

**البارامترات:**
- `clk`: الـ clock المراد إلغاء تسجيله.

**القيمة المُرجَعة:**
- لا شيء (`void`).

**تفاصيل مهمة:**
- لا يعمل `free` للذاكرة — المسؤولية على المستدعي.
- يجب أن يكون الـ `usecount` صفراً قبل الاستدعاء (لا يوجد مستخدمين).
- خطير لو استُدعي والـ clock لازال مستخدماً.

**من يستدعيه:**
كود platform عند إيقاف تشغيل وحدة أو أثناء الـ module unload.

---

### `clk_enable_init_clocks()`

```c
void clk_enable_init_clocks(void);
```

**ما يفعله:**
يمر على كل الـ clocks المسجلة ويشغّل أي clock عليه flag الـ `CLK_ENABLE_ON_INIT`. هذه الـ clocks هي التي يحتاج النظام أن تكون شغّالة من أول لحظة — زي الكهرباء الرئيسية للمبنى قبل ما تشتغل أي غرفة.

**البارامترات:**
- لا توجد.

**القيمة المُرجَعة:**
- لا شيء (`void`).

**تفاصيل مهمة:**
- يُستدعى مرة واحدة فقط أثناء الـ boot، بعد ما تتسجل كل الـ clocks.
- الـ clocks التي عليها `CLK_ENABLE_ON_INIT` عادةً هي clocks حيوية للـ CPU أو الـ bus.

**من يستدعيه:**
كود تهيئة الـ SH clock من `arch/sh/` أثناء الـ `__init`.

---

## المجموعة الثانية: Rate Propagation — نشر التغييرات في الـ Clock Tree

### فكرة المجموعة

الـ clocks مترابطة مثل خراطيم المياه. لو غيّرت ضغط المياه في الأنبوب الرئيسي (الـ parent)، كل الأنابيب الفرعية (الـ children) تتأثر. هذه الدوال مسؤولة عن حساب ونشر تغييرات الـ rate عبر الشجرة.

---

### `followparent_recalc()`

```c
unsigned long followparent_recalc(struct clk *clk);
```

**ما يفعله:**
أبسط دالة `recalc` ممكنة — ببساطة ترجع الـ rate الخاص بالـ parent مباشرةً بدون أي حساب. تُستخدم للـ clocks التي تعمل بنفس تردد أبيها (pass-through clocks).

**البارامترات:**
- `clk`: الـ clock المراد حساب rate-ه — يُستخدم فقط للوصول لـ `clk->parent`.

**القيمة المُرجَعة:**
- الـ `rate` الحالي للـ parent (بالـ Hz).
- `0` لو الـ `clk->parent` هو `NULL`.

**تفاصيل مهمة:**
- هذه الدالة عادةً تُوضع في `sh_clk_ops.recalc` للـ clocks التي ليس لديها divisor أو multiplier خاص بها.
- آمنة جداً لأنها لا تلمس أي hardware register.

**من يستدعيه:**
الـ clock framework يستدعيها عبر `clk->ops->recalc` عند الحاجة لمعرفة الـ rate.

---

### `recalculate_root_clocks()`

```c
void recalculate_root_clocks(void);
```

**ما يفعله:**
يمر على كل الـ clocks التي ليس لها parent (الـ root clocks)، ويستدعي `ops->recalc` لكل واحد منهم ليحدّث قيمة الـ rate المحفوظة. ثم يستدعي `propagate_rate()` لكل root لنشر التحديث لأبنائهم.

**البارامترات:**
- لا توجد.

**القيمة المُرجَعة:**
- لا شيء (`void`).

**تفاصيل مهمة:**
- يُستدعى عادةً عند بداية التشغيل أو بعد تغيير كبير في الـ PLL settings.
- عملية مكلفة نسبياً لأنها تمر على كل الشجرة.

**من يستدعيه:**
كود الـ platform init في `arch/sh/` أثناء الـ boot.

---

### `propagate_rate()`

```c
void propagate_rate(struct clk *clk);
```

**ما يفعله:**
يأخذ clock واحد ويحدّث rate كل أبنائه (children) بشكل recursive. لكل child، يستدعي `ops->recalc` لإعادة حساب rate-ه بناءً على الـ parent الجديد، ثم يكمل لأبناء الـ child وهكذا... زي موجة تنتشر في بحيرة.

```
propagate_rate(parent)
    └─> foreach child of parent:
            child->rate = child->ops->recalc(child)
            propagate_rate(child)   ← recursive
```

**البارامترات:**
- `clk`: الـ clock الأب الذي بدأ منه التغيير — يُستخدم كنقطة انطلاق لنشر التحديث.

**القيمة المُرجَعة:**
- لا شيء (`void`).

**تفاصيل مهمة:**
- استدعاء recursive — يمكن أن يكون عميقاً جداً لو الشجرة كبيرة.
- لا يغير الـ hardware، فقط يحدّث القيم المحفوظة في الـ structs.
- يُستدعى بعد أي `set_rate` ناجح على أي clock.

**من يستدعيه:**
الـ clock framework داخلياً بعد أي تغيير في الـ rate، وأيضاً من `recalculate_root_clocks()`.

---

### `clk_reparent()`

```c
int clk_reparent(struct clk *child, struct clk *parent);
```

**ما يفعله:**
يغير أبا الـ `child` من الـ parent الحالي إلى الـ `parent` الجديد. عملياً: يشيل الـ child من قائمة أبناء الـ parent القديم ويضيفه في قائمة أبناء الـ parent الجديد، ويحدث `child->parent`.

**البارامترات:**
- `child`: الـ clock المراد تغيير أبيه.
- `parent`: الـ parent الجديد المراد الانتقال إليه.

**القيمة المُرجَعة:**
- `0` عند النجاح.
- قيمة سالبة عند الفشل.

**تفاصيل مهمة:**
- بعد `clk_reparent()`، يجب استدعاء `propagate_rate()` لتحديث الـ rate.
- الـ locking ضروري قبل الاستدعاء لأن العملية تعدّل قوائم مشتركة.
- يُستخدم داخلياً من قبل `ops->set_parent` في الـ div4/div6 clocks.

**من يستدعيه:**
الـ `sh_clk_div4_reparent_register()` و `sh_clk_div6_reparent_register()` عند تغيير الـ parent.

---

## المجموعة الثالثة: Rate Table Helpers — حساب الترددات الممكنة

### فكرة المجموعة

الـ SH clocks تعمل بقسمة أو ضرب تردد الـ parent. مش كل قيمة ممكنة — فيه جدول بالقيم المسموح بها. هذه الدوال تساعد في بناء هذا الجدول والبحث فيه. تخيلها زي قائمة أسعار في مطعم — مش تطلب أي سعر، تختار من الموجود.

---

### `clk_rate_table_build()`

```c
void clk_rate_table_build(struct clk *clk,
                          struct cpufreq_frequency_table *freq_table,
                          int nr_freqs,
                          struct clk_div_mult_table *src_table,
                          unsigned long *bitmap);
```

**ما يفعله:**
يملأ الجدول `freq_table` بكل الترددات الممكنة للـ clock. يحسب كل تردد بضرب الـ rate الحالي للـ parent في كل مضاعف (multiplier) ثم يقسمه على كل قاسم (divisor) موجود في الـ `src_table`. لو وُجد `bitmap`، يتجاهل الـ entries الممثّلة ببتات `0` فيه.

**البارامترات:**
- `clk`: الـ clock المراد بناء جدوله — يُستخدم للوصول لـ rate الـ parent.
- `freq_table`: مؤشر لمصفوفة `cpufreq_frequency_table` ستُملأ بالنتائج.
- `nr_freqs`: عدد الـ entries في الـ `freq_table`.
- `src_table`: الجدول المصدر الذي يحتوي على قوائم الـ divisors والـ multipliers.
- `bitmap`: mask اختياري يحدد أي الـ entries صالحة (`NULL` = كل شيء صالح).

**القيمة المُرجَعة:**
- لا شيء (`void`).

**تفاصيل مهمة:**
- يُملأ الجدول بترتيب تصاعدي للـ frequency.
- الـ `cpufreq_frequency_table` هي بنية مشتركة مع الـ cpufreq subsystem.
- نهاية الجدول تُعلَّم بـ `CPUFREQ_TABLE_END`.

**من يستدعيه:**
يُستدعى أثناء تسجيل الـ div clocks، عادةً من الدوال الداخلية في `drivers/sh/clk/`.

---

### `clk_rate_table_round()`

```c
long clk_rate_table_round(struct clk *clk,
                          struct cpufreq_frequency_table *freq_table,
                          unsigned long rate);
```

**ما يفعله:**
يبحث في الجدول `freq_table` عن أقرب تردد ممكن للـ `rate` المطلوب. "أقرب" هنا يعني الأقل تردداً أو المساوي للمطلوب (round down). زي ما تطلب 100 كيلومتر وأقرب محطة على 95 كيلومتر — هذه هي اللي ترجعها.

**البارامترات:**
- `clk`: الـ clock (يُستخدم للسياق).
- `freq_table`: الجدول المبني مسبقاً من `clk_rate_table_build()`.
- `rate`: التردد المطلوب بالـ Hz.

**القيمة المُرجَعة:**
- أقرب تردد متاح (بالـ Hz) كـ `long`.
- قيمة سالبة (errno) لو لم يُوجد أي تردد مناسب.

**تفاصيل مهمة:**
- هذه الدالة تُستخدم كـ implementation لـ `sh_clk_ops.round_rate`.
- لا تغير أي hardware — فقط حساب.

**من يستدعيه:**
الـ clock framework عند استدعاء `clk_round_rate()` من الـ drivers.

---

### `clk_rate_table_find()`

```c
int clk_rate_table_find(struct clk *clk,
                        struct cpufreq_frequency_table *freq_table,
                        unsigned long rate);
```

**ما يفعله:**
يبحث في الجدول عن الـ entry الذي يمثّل الـ `rate` المطلوب ويرجع الـ index الخاص به. مختلف عن `clk_rate_table_round()` لأنه يرجع الـ index مش القيمة، وهذا الـ index يُستخدم لاحقاً للكتابة في الـ hardware register.

**البارامترات:**
- `clk`: الـ clock (للسياق).
- `freq_table`: جدول الترددات.
- `rate`: التردد المطلوب بالضبط.

**القيمة المُرجَعة:**
- الـ index (موجب أو صفر) للـ entry المناسب.
- قيمة سالبة لو لم يُوجد تطابق.

**تفاصيل مهمة:**
- البحث يتوقع تطابقاً تاماً للـ `rate`، لذا عادةً يُستدعى بعد `clk_rate_table_round()`.
- الـ index المُرجَع يُحمَّل مباشرةً في الـ register الخاص بالـ clock divider.

**من يستدعيه:**
الـ `set_rate` implementation في الـ div clocks.

---

### `clk_rate_div_range_round()`

```c
long clk_rate_div_range_round(struct clk *clk,
                              unsigned int div_min,
                              unsigned int div_max,
                              unsigned long rate);
```

**ما يفعله:**
يحاول إيجاد أفضل قاسم (divisor) بين `div_min` و `div_max` يعطي أقرب تردد للـ `rate` المطلوب. بدلاً من جدول جاهز، يحسب التردد لكل قاسم ممكن في النطاق ويختار الأنسب.

**البارامترات:**
- `clk`: الـ clock — يُستخدم للوصول لـ rate الـ parent.
- `div_min`: أصغر قيمة قسمة مسموحة.
- `div_max`: أكبر قيمة قسمة مسموحة.
- `rate`: التردد المطلوب.

**القيمة المُرجَعة:**
- أقرب تردد ممكن بالـ Hz (قيمة موجبة).
- قيمة سالبة عند الخطأ.

**الـ Pseudocode:**

```c
best_rate = 0;
for (div = div_min; div <= div_max; div++) {
    candidate = parent_rate / div;
    if (candidate <= rate && candidate > best_rate)
        best_rate = candidate;
}
return best_rate;
```

**من يستدعيه:**
الـ `round_rate` implementation للـ clocks التي تعمل بنطاق قسمة مستمر.

---

### `clk_rate_mult_range_round()`

```c
long clk_rate_mult_range_round(struct clk *clk,
                               unsigned int mult_min,
                               unsigned int mult_max,
                               unsigned long rate);
```

**ما يفعله:**
نفس فكرة `clk_rate_div_range_round()` لكن باستخدام الضرب بدل القسمة. يجد أفضل مضاعف بين `mult_min` و `mult_max` يعطي أقرب تردد للمطلوب.

**البارامترات:**
- `clk`: الـ clock.
- `mult_min`: أصغر مضاعف مسموح.
- `mult_max`: أكبر مضاعف مسموح.
- `rate`: التردد المطلوب.

**القيمة المُرجَعة:**
- أقرب تردد ممكن بالـ Hz.
- قيمة سالبة عند الخطأ.

**من يستدعيه:**
الـ `round_rate` implementation للـ PLL أو clocks التي تضرب الـ rate.

---

## المجموعة الرابعة: MSTP Clock Registration — تسجيل أقفال الطاقة

### فكرة المجموعة

الـ **MSTP** (Module Stop) هو آلية في معالجات SH/RCar تسمح بإيقاف الـ clock عن وحدات معينة (peripherals) لتوفير الطاقة — زي مفتاح الكهرباء لكل غرفة في البيت. كل وحدة (UART، SPI، USB، إلخ) لها bit في register معين، لو الـ bit = 1 فالـ clock موقوف.

---

### `sh_clk_mstp_register()`

```c
int sh_clk_mstp_register(struct clk *clks, int nr);
```

**ما يفعله:**
يسجّل مصفوفة من الـ MSTP clocks دفعة واحدة. كل clock في المصفوفة يمثّل bit واحد في أحد الـ MSTP registers. يستدعي `clk_register()` لكل clock في المصفوفة ويربطه بالـ ops المناسبة للـ MSTP (enable/disable عبر bit manipulation).

**البارامترات:**
- `clks`: مؤشر لأول عنصر في مصفوفة `struct clk`، كل عنصر يصف MSTP clock واحد.
- `nr`: عدد الـ clocks في المصفوفة.

**القيمة المُرجَعة:**
- `0` لو تسجّلت كلها بنجاح.
- قيمة سالبة (errno) عند أول فشل.

**تفاصيل مهمة:**
- كل `struct clk` في المصفوفة لازم يكون مهيأ بالـ macro `SH_CLK_MSTP32` أو مشابهاته.
- الـ `enable_reg` يشير للـ MSTP register، والـ `enable_bit` هو رقم الـ bit.
- الـ `status_reg` اختياري — لو موجود يُستخدم للتحقق من توقف الـ clock فعلاً.
- لو الـ `flags` يحتوي على `CLK_ENABLE_REG_32BIT` يتعامل مع الـ register كـ 32-bit، وهكذا للـ 16bit و 8bit.

**مثال على الاستخدام مع الـ Macro:**

```c
/* تعريف MSTP clock لوحدة UART0 */
static struct clk uart0_clk = SH_CLK_MSTP32(
    &p_clk,        /* الـ parent clock */
    MSTPCR0,       /* عنوان الـ MSTP register */
    20,            /* رقم الـ bit */
    0              /* flags */
);

/* تسجيل مصفوفة من الـ clocks */
static struct clk *mstp_clks[] = { &uart0_clk, ... };
sh_clk_mstp_register(mstp_clks, ARRAY_SIZE(mstp_clks));
```

**من يستدعيه:**
كود الـ platform في `arch/sh/` أو `arch/arm/mach-*/` للـ RCar خلال الـ `__init`.

---

### `sh_clk_mstp32_register()` (deprecated)

```c
static inline int __deprecated sh_clk_mstp32_register(struct clk *clks, int nr);
```

**ما يفعله:**
wrapper قديم يستدعي `sh_clk_mstp_register()` مباشرةً بدون أي فرق في السلوك. موجود فقط للتوافق مع الكود القديم (backward compatibility).

**تفاصيل مهمة:**
- المُعلِّم `__deprecated` يجعل المترجم يعطي تحذيراً عند استخدامه.
- لا تستخدمه في كود جديد — استخدم `sh_clk_mstp_register()` مباشرةً.

---

## المجموعة الخامسة: DIV4 Clock Registration

### فكرة المجموعة

الـ **DIV4** clocks هي clocks تأخذ تردد الـ parent وتقسمه على واحدة من 16 قيمة ممكنة (4 bits للاختيار). شائعة جداً في معالجات SH للـ CPU clock وغيره. المميز فيها أن التقسيمات المسموح بها قابلة للتخصيص عبر الـ `arch_flags` bitmap.

---

### `sh_clk_div4_register()`

```c
int sh_clk_div4_register(struct clk *clks, int nr,
                         struct clk_div4_table *table);
```

**ما يفعله:**
يسجّل مصفوفة من الـ DIV4 clocks. كل clock يعمل بقسمة الـ parent على واحدة من القيم المحددة في الـ `table`. هذا الإصدار البسيط: لا يدعم الـ enable/disable المستقل ولا تغيير الـ parent.

**البارامترات:**
- `clks`: مصفوفة الـ DIV4 clocks (كل واحد مهيأ بـ `SH_CLK_DIV4`).
- `nr`: عدد الـ clocks.
- `table`: مؤشر لـ `struct clk_div_table` الذي يحتوي على `div_mult_table` وـ `kick` function.

**القيمة المُرجَعة:**
- `0` عند النجاح، قيمة سالبة عند الفشل.

**تفاصيل مهمة:**
- الـ `table->kick` هي دالة تُستدعى بعد تغيير الـ rate لتطبيق الإعداد على الـ hardware (مثل كتابة في register خاص).
- الـ `arch_flags` في كل `struct clk` هو bitmap يحدد أي من الـ 16 تقسيمة مسموح باستخدامها.

---

### `sh_clk_div4_enable_register()`

```c
int sh_clk_div4_enable_register(struct clk *clks, int nr,
                                struct clk_div4_table *table);
```

**ما يفعله:**
نفس `sh_clk_div4_register()` لكن يضيف دعم الـ enable/disable لكل clock بشكل مستقل. يُستخدم للـ clocks التي يمكن إيقافها بالكامل عند عدم الحاجة إليها.

**البارامترات:**
- نفس بارامترات `sh_clk_div4_register()`.

**القيمة المُرجَعة:**
- `0` عند النجاح، قيمة سالبة عند الفشل.

**من يستدعيه:**
كود الـ platform لـ SH/RCar عند تسجيل clocks التي تحتاج power gating.

---

### `sh_clk_div4_reparent_register()`

```c
int sh_clk_div4_reparent_register(struct clk *clks, int nr,
                                  struct clk_div4_table *table);
```

**ما يفعله:**
نفس `sh_clk_div4_register()` لكن يضيف دعم تغيير الـ parent clock ديناميكياً. مناسب للـ clocks التي يمكن توصيلها بأكثر من مصدر (مثل PLL-A أو PLL-B).

**البارامترات:**
- نفس بارامترات `sh_clk_div4_register()`.
- الـ `clk->parent_table` و `clk->parent_num` يجب أن يكونا محددين في كل `struct clk`.

**القيمة المُرجَعة:**
- `0` عند النجاح، قيمة سالبة عند الفشل.

---

## المجموعة السادسة: DIV6 Clock Registration

### فكرة المجموعة

الـ **DIV6** clocks تشبه الـ DIV4 لكن أعقد — تستخدم 6 bits للتقسيم (64 قيمة ممكنة) وغالباً لديها القدرة على اختيار الـ parent من مصادر متعددة. يدعم أيضاً الـ `CLK_MASK_DIV_ON_DISABLE` الذي يعني: عند الإيقاف، أكتب القيمة القصوى للـ divisor بدل ما توقف الـ clock كلياً.

---

### `sh_clk_div6_register()`

```c
int sh_clk_div6_register(struct clk *clks, int nr);
```

**ما يفعله:**
يسجّل مصفوفة من الـ DIV6 clocks ذات الـ parent الثابت. كل clock له parent واحد محدد ومسبق.

**البارامترات:**
- `clks`: مصفوفة الـ DIV6 clocks (كل واحد مهيأ بـ `SH_CLK_DIV6`).
- `nr`: عدد الـ clocks.

**القيمة المُرجَعة:**
- `0` عند النجاح، قيمة سالبة عند الفشل.

**تفاصيل مهمة:**
- الـ `enable_bit` هو `0` دائماً لأن الـ DIV6 لا يستخدم bit بعينه للـ enable — بدلاً من ذلك يكتب القيمة القصوى في الـ divisor field (بسبب `CLK_MASK_DIV_ON_DISABLE`).
- الـ `div_mask` يساوي `SH_CLK_DIV6_MSK` = `(1<<6)-1` = `0x3F`.

---

### `sh_clk_div6_reparent_register()`

```c
int sh_clk_div6_reparent_register(struct clk *clks, int nr);
```

**ما يفعله:**
يسجّل الـ DIV6 clocks مع دعم اختيار الـ parent ديناميكياً. كل clock لديه جدول `parent_table` بالـ parents الممكنين، والاختيار يتم عبر حقول `src_shift` و `src_width` في الـ register.

**البارامترات:**
- `clks`: مصفوفة الـ DIV6 clocks (كل واحد مهيأ بـ `SH_CLK_DIV6_EXT`).
- `nr`: عدد الـ clocks.

**القيمة المُرجَعة:**
- `0` عند النجاح، قيمة سالبة عند الفشل.

**تفاصيل مهمة:**
- الـ `SH_CLK_DIV6_EXT` macro يحدد `src_shift` و `src_width` اللتان تصفان موقع حقل اختيار الـ parent في الـ register.
- الـ `parent_table` هو مصفوفة من مؤشرات `struct clk*`، وطولها `parent_num`.

**مقارنة DIV4 vs DIV6:**

| الخاصية | DIV4 | DIV6 |
|---------|------|------|
| عدد bits للتقسيم | 4 bits | 6 bits |
| عدد قيم التقسيم | 16 | 64 |
| الـ divisor mask | `SH_CLK_DIV4_MSK` (0xF) | `SH_CLK_DIV6_MSK` (0x3F) |
| اختيار الـ parent | عبر `reparent` variant | مدمج في `EXT` variant |
| الـ disable | يوقف الـ clock | يكتب max divisor |

---

## المجموعة السابعة: FSI Divider Clock Registration

### فكرة المجموعة

الـ **FSI** (Feature Serial Interface أو Flexible Sound Interface) هو محيط صوتي موجود في بعض معالجات SH/RCar. الـ `fsidiv` clocks خاصة بهذا المحيط ولها آلية تقسيم مختلفة عن DIV4/DIV6 — تستخدم register خاص بالـ FSI نفسه بدل الـ CPG.

---

### `sh_clk_fsidiv_register()`

```c
int sh_clk_fsidiv_register(struct clk *clks, int nr);
```

**ما يفعله:**
يسجّل الـ FSI divider clocks. الميزة الفريدة: يُحوّل الـ `enable_reg` (الذي يحمل عنوان فيزيائي) إلى `clk->mapping` عن طريق عمل `ioremap` للـ register. يستخدم الـ `kref` في `clk_mapping` لإدارة مشاركة الـ mapping بين عدة clocks.

**البارامترات:**
- `clks`: مصفوفة الـ FSI clocks (كل واحد مهيأ بـ `SH_CLK_FSIDIV`).
- `nr`: عدد الـ clocks.

**القيمة المُرجَعة:**
- `0` عند النجاح.
- قيمة سالبة (errno) لو فشل الـ `ioremap` أو التسجيل.

**تفاصيل مهمة:**
- بعد الاستدعاء، الـ `clk->enable_reg` لم يعد يشير للعنوان الفيزيائي — بدلاً من ذلك، يُستخدم `clk->mapped_reg` للوصول الفعلي.
- الـ `clk_mapping` يحتوي على `kref` للـ reference counting — لو clock-ان يشتركان في نفس الـ register، الـ `ioremap` يحصل مرة واحدة فقط.
- هذا الـ clock نوع خاص جداً ومرتبط بالـ FSI hardware فقط.

**الـ Pseudocode للمنطق الداخلي:**

```c
for (i = 0; i < nr; i++) {
    phys = (phys_addr_t)clks[i].enable_reg;

    /* ابحث لو في mapping موجود لنفس العنوان */
    mapping = find_existing_mapping(phys);
    if (!mapping) {
        /* أنشئ mapping جديد */
        mapping = create_mapping(phys, ...);
        mapping->base = ioremap(phys, len);
    }
    kref_get(&mapping->ref);  /* زيّد الـ reference count */

    clks[i].mapping = mapping;
    clk_register(&clks[i]);
}
```

**من يستدعيه:**
كود platform الخاص بالـ FSI في `arch/sh/` أو `sound/soc/sh/`.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنشتغل عليه هو **SH Clock Framework** — نظام إدارة الـ clock الخاص بمعالجات SuperH (SH) في Linux kernel. الكود موجود في `include/linux/sh_clk.h` ويتحكم في:
- الـ **MSTP clocks** (Module Stop clocks — تفعيل/تعطيل وحدات الـ SoC)
- الـ **DIV4/DIV6 clocks** (قسمة تردد الـ clock على 4 أو 6)
- الـ **FSIDIV clocks** (قسمة خاصة لوحدة FSI الصوتية)
- الـ **clk_mapping** (ربط عناوين فيزيائية بـ virtual memory للـ registers)

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ SH clock framework بيستخدم الـ **debugfs** عشان يعرض حالة كل الـ clocks:

```bash
# Mount الـ debugfs لو مش مـ mounted
mount -t debugfs none /sys/kernel/debug

# عرض شجرة الـ clocks (لو kernel بيستخدم common clk framework)
ls /sys/kernel/debug/clk/

# قراءة تردد clock معين
cat /sys/kernel/debug/clk/<clock_name>/clk_rate

# معرفة كم مرة تم تفعيل الـ clock (enable count)
cat /sys/kernel/debug/clk/<clock_name>/clk_enable_count

# معرفة الـ parent الحالي للـ clock
cat /sys/kernel/debug/clk/<clock_name>/clk_parent

# عرض كل الـ clocks وترددها دفعة واحدة
cat /sys/kernel/debug/clk/clk_summary
```

مثال على output من `clk_summary`:
```
                                 enable  prepare  protect                                duty
   clock                          cnt    cnt      cnt        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 extal                              1      1        0    20000000          0     0  50000
   pll1                             1      1        0   800000000          0     0  50000
     div4_clk                       2      2        0   200000000          0     0  50000
       mstp_clk_uart0               1      1        0   200000000          0     0  50000
```

الـ `enable cnt` هو نفسه `usecount` في `struct clk` — لو وصل صفر والـ clock لسه شغال، فيه مشكلة.

#### 2. مدخلات الـ sysfs

```bash
# عرض الـ clocks المرتبطة بجهاز معين
ls /sys/bus/platform/devices/<device_name>/

# فحص حالة الـ clock للـ device
cat /sys/bus/platform/devices/<device_name>/driver/

# لو بتستخدم cpufreq مع الـ clocks (freq_table في struct clk)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# عرض كل الـ clock events المتاحة
ls /sys/kernel/debug/tracing/events/clk/

# تفعيل كل أحداث الـ clock
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_disable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_parent/enable

# تشغيل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تشغيل الـ application أو الـ driver اللي تريد تـ trace فيه
# ثم قراءة النتائج
cat /sys/kernel/debug/tracing/trace

# مثال على استخدام function_graph tracer لتتبع دوال sh_clk
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo "sh_clk_*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

مثال على output من `clk_set_rate` event:
```
    kworker-123   [000] ....  1234.567890: clk_set_rate: div4_clk 200000000
    kworker-123   [000] ....  1234.567891: clk_set_rate_complete: div4_clk 200000000
```

#### 4. الـ printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لملفات الـ SH clock
echo "file drivers/sh/clk*.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/sh/clk-cpg.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لكل الـ clock subsystem
echo "module sh_clk +p" > /sys/kernel/debug/dynamic_debug/control

# عرض القواعد الفعّالة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep clk

# تغيير مستوى الـ printk لمشاهدة رسائل debug في kernel log
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk
```

في الكود، الـ `printk` المناسبة هي:
```c
pr_debug("sh_clk: clock %s rate=%lu usecount=%d\n",
         clk->name, clk->rate, clk->usecount);
```

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الوظيفة | متى تفعّله؟ |
|---|---|---|
| `CONFIG_SH_CLK_CPG_LEGACY` | يفعّل الـ `init` callback القديم في `sh_clk_ops` | لو تـ debug على hardware قديم SH |
| `CONFIG_COMMON_CLK` | يفعّل الـ common clock framework مع debugfs كامل | أساسي للـ debugging |
| `CONFIG_CLK_DEBUG` | يضيف معلومات إضافية في debugfs | مهم جداً |
| `CONFIG_DEBUG_FS` | يفعّل debugfs كله | شرط أساسي |
| `CONFIG_DEBUG_KERNEL` | يفعّل كل الـ kernel debug options | للتطوير فقط |
| `CONFIG_KALLSYMS` | يعرض أسماء الدوال في stack traces | مهم لقراءة call stacks |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ clock locks | لو في مشاكل locking |
| `CONFIG_DEBUG_SHIRQ` | يـ debug مشاكل الـ shared IRQ | لو الـ clock مرتبط بـ IRQ |
| `CONFIG_CPUFREQ_DEBUG` | يفعّل debug الـ cpufreq (مرتبط بـ freq_table في struct clk) | لمشاكل الـ frequency scaling |

```bash
# التحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(COMMON_CLK|CLK_DEBUG|DEBUG_FS|SH_CLK)"
# أو
cat /boot/config-$(uname -r) | grep -E "CONFIG_(COMMON_CLK|CLK_DEBUG)"
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# clk_summary — أهم أداة للـ SH clocks
cat /sys/kernel/debug/clk/clk_summary

# clk_dump — نسخة JSON من شجرة الـ clocks
cat /sys/kernel/debug/clk/clk_dump

# لو شغّال على SuperH hardware وفيه tool خاصة بـ CPG
# فحص حالة MSTP registers عبر /proc
cat /proc/clocks  # لو kernel قديم بيدعم هذا

# استخدام devmem2 لقراءة MSTP registers مباشرة (راجع Hardware Level)
# قراءة cpufreq frequency table المرتبطة بالـ clocks
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `sh_clk: unable to register clock` | فشل `clk_register()` — غالباً لأن الـ clock موجود أصلاً أو في memory allocation | تحقق من `usecount` وتأكد من عدم التسجيل المزدوج |
| `sh_clk: invalid clock rate` | `set_rate` طلب تردد خارج نطاق الـ `freq_table` | تحقق من `clk_rate_table_round()` ومن قيم الـ `divisors` في `clk_div_mult_table` |
| `clk: Enabling CLK_IS_CRITICAL clock` | محاولة تعطيل clock حيوي | لا تعطّل هذا الـ clock أبداً — راجع `CLK_ENABLE_ON_INIT` flag |
| `clk: Failed to reparent` | فشل `clk_reparent()` — الـ parent المطلوب غير موجود في `parent_table` | تحقق من `parent_num` وأن الـ parent مسجّل |
| `sh_clk: failed to map clock register` | فشل `ioremap` لعنوان `phys` في `clk_mapping` | تحقق من صحة العنوان الفيزيائي وأنه ضمن نطاق الـ SoC |
| `clk: orphan clock` | clock تم تسجيله قبل الـ parent | غيّر ترتيب التسجيل في `sh_clk_mstp_register` أو `sh_clk_div4_register` |
| `sh_clk: divide by zero` | `div_mask = 0` أو الـ divisor صفر في `clk_div_mult_table` | تحقق من `SH_CLK_DIV4_MSK` / `SH_CLK_DIV6_MSK` وقيم الـ divisors |
| `genirq: Flags mismatch` (مرتبط بـ clock IRQ) | الـ clock interrupt مشترك بشكل خاطئ | راجع `enable_bit` وتحقق من عدم التعارض مع devices أخرى |
| `WARNING: CPU: X PID: Y at drivers/clk/clk.c` | `WARN_ON` انفجر داخل الـ clock framework | اقرأ الـ stack trace — غالباً `usecount` سالب أو enable/disable غير متوازن |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

الأماكن الذهبية لوضع `WARN_ON()` و `dump_stack()` في كود الـ SH clock:

```c
/* في دالة enable — تحقق أن usecount لا يتجاوز حد معقول */
int sh_clk_enable(struct clk *clk)
{
    WARN_ON(clk->usecount < 0);  /* usecount سالب = enable/disable غير متوازن */

    if (clk->ops && clk->ops->enable) {
        WARN_ON(!clk->enable_reg);  /* لازم يكون فيه register */
        return clk->ops->enable(clk);
    }
    return 0;
}

/* في clk_reparent — تحقق أن الـ parent في الـ parent_table */
int clk_reparent(struct clk *child, struct clk *parent)
{
    WARN_ON(!child || !parent);
    WARN_ON(child == parent);  /* clock لا يكون parent لنفسه */
    /* ... */
}

/* في clk_rate_table_find — تحقق أن الـ rate موجود */
int clk_rate_table_find(struct clk *clk,
                         struct cpufreq_frequency_table *freq_table,
                         unsigned long rate)
{
    WARN_ON(!freq_table);
    WARN_ON(clk->nr_freqs == 0);
    /* ... */
}

/* في clk_mapping — تحقق الـ reference count */
void clk_mapping_release(struct kref *kref)
{
    struct clk_mapping *mapping = container_of(kref, struct clk_mapping, ref);
    WARN_ON(!mapping->base);  /* لو base = NULL معناه double-release */
    iounmap(mapping->base);
}
```

---

### Hardware Level

#### 1. التحقق أن حالة الـ Hardware تطابق حالة الـ Kernel

الفكرة: الـ kernel بيحتفظ بـ `usecount` و `rate` في `struct clk` — هل يطابقون الـ hardware فعلاً؟

```bash
# الخطوة 1: اقرأ rate الـ clock من kernel
cat /sys/kernel/debug/clk/div4_clk/clk_rate
# مثال: 200000000

# الخطوة 2: احسب التردد يدوياً من الـ register
# لو FRQCR register عند 0xFFC80000 وفيه div field
devmem2 0xFFC80000 w
# اقرأ القيمة وطبّق الـ div_mask

# الخطوة 3: قارن النتيجتين
# لو مختلفتين = kernel cache قديم أو hardware تغيّر بدون علم الـ kernel

# الخطوة 4: تحقق من MSTP register — هل الـ bit صح؟
# مثلاً MSTPCR0 عند 0xE6150130
devmem2 0xE6150130 w
# Bit 0 = 0 → module شغال، Bit = 1 → module متوقف
```

#### 2. قراءة الـ Registers مباشرة (Register Dump)

**أدوات الـ Register Dump:**

```bash
# تثبيت devmem2 لو مش موجود
apt-get install devmem2
# أو build من source: https://github.com/VCTLabs/devmem2

# قراءة MSTP Control Register (32-bit) — MSTPCR0 مثلاً
devmem2 0xE6150130 w
# Output: Value at address 0xE6150130 (0xE6150130): 0x000000FF
# كل bit = module واحد، 1 = متوقف، 0 = شغال

# قراءة Frequency Control Register (FRQCR) للـ DIV4 clocks
devmem2 0xFFC80000 w

# قراءة عبر /dev/mem مباشرة (Python)
python3 -c "
import mmap, struct
with open('/dev/mem', 'r+b') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xE6150000 & ~0xFFF)
    offset = 0xE6150130 & 0xFFF
    val = struct.unpack('<I', m[offset:offset+4])[0]
    print(f'MSTPCR0 = 0x{val:08X}')
    # طباعة كل bit
    for i in range(32):
        state = 'STOPPED' if (val >> i) & 1 else 'RUNNING'
        print(f'  Bit {i:2d}: {state}')
"

# استخدام io utility (من package iotools)
io -4 0xE6150130  # قراءة 32-bit register

# قراءة باستخدام dd من /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xE6150130 / 4)) 2>/dev/null | xxd
```

**مقارنة قيم الـ Register مع kernel:**

```bash
# Script يقارن حالة MSTP register مع kernel
#!/bin/bash
MSTPCR0=0xE6150130

# قراءة من hardware
HW_VAL=$(devmem2 $MSTPCR0 w | grep "Value" | awk '{print $NF}')
echo "Hardware MSTPCR0: $HW_VAL"

# قراءة من kernel debugfs
echo "Kernel clock states:"
for clk in /sys/kernel/debug/clk/*/; do
    name=$(basename $clk)
    rate=$(cat $clk/clk_rate 2>/dev/null)
    enabled=$(cat $clk/clk_enable_count 2>/dev/null)
    echo "  $name: rate=$rate, enabled=$enabled"
done
```

#### 3. نصائح Logic Analyzer والـ Oscilloscope

**Clock Signal Verification:**

للـ SH SoC clocks، الـ clock signals ممكن تكون على pins خارجية أو internal:

```
الـ EXTAL pin (crystal input):
┌──┐  ┌──┐  ┌──┐  ┌──┐  ┌──┐
│  │  │  │  │  │  │  │  │  │
┘  └──┘  └──┘  └──┘  └──┘  └
↑ قيس التردد هنا بالـ oscilloscope
```

**نقاط القياس المهمة:**

| النقطة | ما تقيسه | القيمة المتوقعة |
|---|---|---|
| EXTAL pin | تردد الـ crystal المرجعي | 20 MHz (نموذجي) |
| CKIO pin | الـ bus clock الخارجي | EXTAL / divider |
| AUDCK pin | clock وحدة الصوت (FSI) | قيمة SH_CLK_FSIDIV |
| GPIO clock output | لو الـ SoC يدعم clock output | حسب الإعداد |

**إعداد Logic Analyzer:**
```
1. وصّل Channel 1 على EXTAL
2. وصّل Channel 2 على الـ clock output المشتبه فيه
3. اضبط Trigger على rising edge
4. قيس:
   - التردد (يجب يطابق clk_rate في debugfs)
   - الـ duty cycle (يجب قريب من 50%)
   - الـ jitter (يجب أقل من 1% من الـ period)
5. لو الـ clock متقطع (glitchy) عند تغيير الـ rate = مشكلة في CLK_MASK_DIV_ON_DISABLE
```

#### 4. مشاكل Hardware شائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في dmesg | السبب الجذري |
|---|---|---|
| Crystal غير مستقر | `clk: rate mismatch after set_rate` + `WARN_ON` متكرر | EXTAL frequency خطأ في DT أو crystal معيوب |
| MSTP register لا يستجيب | `sh_clk: timeout waiting for clock stop` | Power domain مش شغال أو bus clock متوقف |
| DIV4 divisor خاطئ | `clk: requested rate not achievable` متكرر | `_div_bitmap` خاطئ في `SH_CLK_DIV4` macro أو `divisors[]` غلط |
| clk_mapping فشل | `ioremap failed for clock register` | عنوان فيزيائي خارج نطاق الـ SoC أو محجوز لـ device آخر |
| Parent clock متوقف وabruptly | `BUG: scheduling while atomic` عند `clk_enable` | parent لم يُفعَّل قبل الـ child — خلل في ترتيب `usecount` |
| FSIDIV overflow | `sh_clk_fsidiv: invalid divider value` | قيمة القسمة أكبر من `div_mask` |
| Power glitch على enable_reg | جهاز يتوقف عشوائياً | الـ `enable_reg` في power domain مختلف — تحقق بالـ oscilloscope |

#### 5. Device Tree Debugging — تحقق أن الـ DT يطابق الـ Hardware

**السيناريو:** `enable_reg` في `struct clk` مشتق من DT node — لازم يطابق العنوان الفيزيائي الحقيقي.

```bash
# عرض الـ DT المُحمّل حالياً
ls /sys/firmware/devicetree/base/

# البحث عن clock nodes في الـ DT
find /sys/firmware/devicetree/base/ -name "compatible" | xargs grep -l "renesas,cpg-clocks" 2>/dev/null

# قراءة reg property للـ clock controller node
# مثلاً لو الـ node اسمه cpg
cat /sys/firmware/devicetree/base/cpg/reg | xxd
# الـ output هيكون العنوان الفيزيائي والحجم

# مقارنة مع ما في DTS source
# في ملف DTS:
# cpg: clock-controller@e6150000 {
#     reg = <0xe6150000 0x1000>;
# };

# التحقق باستخدام dtc
dtc -I fs /sys/firmware/devicetree/base/ -O dts 2>/dev/null | grep -A5 "cpg"

# التحقق أن الـ clock names في DT تطابق ما في kernel
cat /sys/firmware/devicetree/base/cpg/clock-names 2>/dev/null | tr '\0' '\n'

# فحص assigned-clocks لو في DT
find /sys/firmware/devicetree/base/ -name "assigned-clocks" | head -5
```

**أخطاء DT شائعة مع الـ SH clocks:**

```bash
# الخطأ: العنوان في DT خاطئ
# dmesg يقول: "sh_clk: failed to map 0xE6150000"
# الحل: تحقق من SoC datasheet وقارن مع reg في DT node

# الخطأ: clock-frequency غلط (extal)
# dmesg يقول: "clk: rate mismatch"
# الحل:
grep -r "clock-frequency" /sys/firmware/devicetree/base/ 2>/dev/null | head

# التحقق من #clock-cells
cat /sys/firmware/devicetree/base/cpg/#clock-cells 2>/dev/null | xxd
# يجب أن يكون 1 (يعني index-based clocks)
```

---

### Practical Commands

#### 1. Script شامل للـ Diagnosis السريع

```bash
#!/bin/bash
# sh_clk_debug.sh — فحص سريع لحالة SH clocks

echo "=== SH Clock Framework Debug Report ==="
echo "Date: $(date)"
echo ""

echo "--- Clock Summary ---"
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null || echo "debugfs not available"
echo ""

echo "--- Clock Dump (JSON) ---"
cat /sys/kernel/debug/clk/clk_dump 2>/dev/null | python3 -m json.tool 2>/dev/null | head -50
echo ""

echo "--- Orphan Clocks ---"
cat /sys/kernel/debug/clk/clk_orphan_summary 2>/dev/null || echo "No orphan summary available"
echo ""

echo "--- Recent Clock-related Kernel Messages ---"
dmesg | grep -iE "(clk|clock|mstp|cpg|frqcr)" | tail -30
echo ""

echo "--- CPUFreq (linked to clock freq_table) ---"
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies 2>/dev/null
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq 2>/dev/null
echo ""
```

```bash
# تشغيل السكريبت
chmod +x sh_clk_debug.sh
./sh_clk_debug.sh 2>&1 | tee /tmp/clk_debug_$(date +%Y%m%d_%H%M%S).log
```

#### 2. تفعيل الـ Clock Tracing خطوة بخطوة

```bash
#!/bin/bash
# تفعيل tracing للـ clock events

TRACE_DIR=/sys/kernel/debug/tracing

# وقف أي tracing قديم
echo 0 > $TRACE_DIR/tracing_on

# مسح الـ buffer القديم
echo > $TRACE_DIR/trace

# اختيار الـ tracer
echo nop > $TRACE_DIR/current_tracer

# تفعيل clock events
for event in clk_enable clk_enable_complete clk_disable clk_disable_complete \
             clk_set_rate clk_set_rate_complete clk_set_parent clk_set_parent_complete; do
    if [ -d "$TRACE_DIR/events/clk/$event" ]; then
        echo 1 > $TRACE_DIR/events/clk/$event/enable
        echo "Enabled: $event"
    fi
done

# بدء الـ tracing
echo 1 > $TRACE_DIR/tracing_on
echo "Tracing started. Press Ctrl+C to stop."

# انتظر interrupt
trap 'echo 0 > $TRACE_DIR/tracing_on; echo "Tracing stopped."' INT
sleep 30

# عرض النتائج
echo ""
echo "=== Trace Results ==="
cat $TRACE_DIR/trace | grep -v "^#" | head -100
```

#### 3. قراءة MSTP Registers وتفسيرها

```bash
#!/bin/bash
# قراءة MSTP registers لـ Renesas SH/R-Car SoCs
# عدّل العناوين حسب الـ SoC الخاص بك

# R-Car H2/M2 MSTP register addresses
declare -A MSTP_REGS=(
    ["SMSTPCR0"]="0xE6150130"
    ["SMSTPCR1"]="0xE6150134"
    ["SMSTPCR2"]="0xE6150138"
    ["SMSTPCR3"]="0xE615013C"
    ["SMSTPCR4"]="0xE6150140"
    ["SMSTPCR7"]="0xE615014C"
    ["SMSTPCR8"]="0xE6150990"
    ["SMSTPCR9"]="0xE6150994"
)

echo "=== MSTP Register Status ==="
for name in "${!MSTP_REGS[@]}"; do
    addr=${MSTP_REGS[$name]}
    val=$(devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}')
    if [ -n "$val" ]; then
        echo ""
        echo "$name ($addr) = $val"
        # تحويل hex لـ binary وعرض الـ bits المُوقفة
        dec_val=$((val))
        for i in $(seq 31 -1 0); do
            bit=$(( (dec_val >> i) & 1 ))
            if [ $bit -eq 1 ]; then
                echo "  Bit $i: STOPPED (1)"
            fi
        done
    else
        echo "$name ($addr): cannot read (no devmem2 or permission denied)"
    fi
done
```

**تفسير الـ output:**
```
=== MSTP Register Status ===
SMSTPCR0 (0xE6150130) = 0x00200000
  Bit 21: STOPPED (1)
```
يعني: Module رقم 21 في SMSTPCR0 متوقف — لو driver ذاك الـ module بيشكي، هنا السبب.

#### 4. فحص clk_mapping والـ ioremap

```bash
# لو بتشك في مشاكل الـ clk_mapping (struct clk_mapping في sh_clk.h)
# افحص الـ iomem allocations
cat /proc/iomem | grep -iE "(clk|cpg|mstp)"

# فحص الـ virtual mappings
cat /proc/vmallocinfo | grep ioremap | grep -E "(E615|FFC8)" | head -20

# فحص reference count للـ mappings (kref في struct clk_mapping)
# لا يوجد interface مباشر، لكن يمكن trace عبر ftrace
echo "clk_mapping*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep clk_mapping
```

#### 5. تشخيص مشاكل DIV4/DIV6

```bash
# فحص الـ clocks المبنية على DIV4
cat /sys/kernel/debug/clk/clk_summary | grep -E "(div4|div6|DIV)"

# قراءة FRQCR register (Frequency Control Register) لـ SH7734 مثلاً
# عنوان FRQCR = 0xFFCA0010 (يختلف حسب الـ SoC)
devmem2 0xFFCA0010 w
# Output مثال: 0x00004D20
# Bits [3:0] = DIV4 field → القيمة = 2 → divisor = 4 (من جدول الـ divisors)

# مقارنة مع kernel
cat /sys/kernel/debug/clk/div4_clk/clk_rate
# يجب أن يساوي: parent_rate / divisor

# فحص DIV6 register
# SH_CLK_DIV6_MSK = 0x3F (6 bits)
# Register bits [5:0] = divisor - 1
devmem2 <DIV6_REG_ADDR> w
```

#### 6. تفسير dmesg عند مشاكل الـ Clock

```bash
# فلترة رسائل الـ clock من dmesg مع timestamp
dmesg -T | grep -iE "(clk|clock|mstp|cpg)" | grep -vE "(systemd|NetworkManager)"

# البحث عن WARNINGs المتعلقة بالـ clocks
dmesg | grep -E "WARNING|BUG|WARN_ON" | grep -A3 -B3 "clk"

# مراقبة الـ kernel messages في real-time أثناء تجربة الـ clock
dmesg -w | grep -iE "(clk|mstp)" &
CLK_PID=$!
# ... شغّل الـ driver أو الـ test ...
kill $CLK_PID
```

**مثال على output ناجح عند تسجيل الـ clocks:**
```
[    1.234567] sh_clk: registering MSTP clocks (12 clocks)
[    1.234890] sh_clk: clk uart0 enabled, rate=66666666
[    1.235100] sh_clk: clk i2c0 enabled, rate=66666666
[    1.235300] sh_clk: div4 clock pll1_div2, rate=400000000
```

**مثال على output مشكلة:**
```
[    1.234567] sh_clk: failed to register clock div4_clk: -EINVAL
[    1.234890] WARNING: CPU: 0 PID: 1 at drivers/sh/clk.c:234 sh_clk_div4_register+0x4c/0x80
[    1.234900] sh_clk: div_bitmap 0x0 invalid for div4 clock
```
**التفسير:** `_div_bitmap` في ماكرو `SH_CLK_DIV4` صفر — يعني ما فيه أي divisor مسموح فيه.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: جهاز IoT على SH7734 — الـ UART صامت تماماً بعد الـ boot

#### العنوان
الـ UART لا يشتغل على بورد صناعي يعمل بـ SH7734 بعد تشغيل الـ kernel

#### السياق
شركة تصنع **industrial gateway** صغير للـ SCADA systems. البورد يشتغل بـ **Renesas SH7734** (SuperH). المهندس يحاول تشغيل console على الـ UART0 لكن ما يطلع أي شيء — لا boot messages، لا shell، لا حتى garbage characters.

#### المشكلة
الـ UART driver يُحمَّل بدون أخطاء في الـ kernel log (اللي تم التقاطه عبر JTAG)، لكن الـ hardware لا يرسل أو يستقبل. الـ oscilloscope على TX pin يُظهر الـ line stuck high (idle) — يعني ما في أي transmission إطلاقاً.

#### التحليل
المهندس يتتبع المشكلة عبر `sh_clk.h`:

1. الـ UART يحتاج clock من الـ MSTP (Module Stop) register. لو الـ clock مش مفعّل، الـ peripheral يصير في حالة reset.

2. في `sh_clk.h` الـ macro `SH_CLK_MSTP32` يُعرَّف هكذا:
```c
#define SH_CLK_MSTP32(_p, _r, _b, _f) \
    SH_CLK_MSTP(_p, _r, _b, 0, _f | CLK_ENABLE_REG_32BIT)
```
يعني الـ access للـ enable register لازم يكون 32-bit. لو أحد استخدم `SH_CLK_MSTP8` بالغلط، الـ write للـ register هيكون 8-bit وهيكتب في العنوان الغلط على SH7734 لأن الـ byte ordering مختلف.

3. المهندس يفحص تعريف الـ UART clock في ملف platform:
```c
/* الغلط — استخدام 8-bit access على register يحتاج 32-bit */
static struct clk mstp_clks[] = {
    [MSTP_UART0] = SH_CLK_MSTP8(&p_clk, MSTPCR0, 20, 0),
};
```

4. الـ struct `clk` في `sh_clk.h` عنده field اسمه `flags`:
```c
unsigned long flags;
```
وعند التسجيل عبر `sh_clk_mstp_register`، الـ framework يقرأ الـ `CLK_ENABLE_REG_MASK` من الـ flags عشان يعرف حجم الـ access:
```c
#define CLK_ENABLE_REG_MASK (CLK_ENABLE_REG_32BIT | \
                             CLK_ENABLE_REG_16BIT | \
                             CLK_ENABLE_REG_8BIT)
```
لو `CLK_ENABLE_REG_8BIT` set، الـ driver هيعمل `iowrite8` بدل `iowrite32`، وعلى SH7734 الـ MSTPCR0 هو 32-bit only register — الـ 8-bit write مش هيشتغل صح.

#### الحل
```c
/* الصح — استخدام SH_CLK_MSTP32 */
static struct clk mstp_clks[] = {
    [MSTP_UART0] = SH_CLK_MSTP32(&p_clk, MSTPCR0, 20, 0),
};
```

للتأكد، المهندس يفحص الـ register مباشرة عبر JTAG:
```bash
# قراءة MSTPCR0 عبر /dev/mem أو JTAG
# لو bit 20 لسا set = الـ UART module stopped
devmem 0xFFC80014 32  # عنوان MSTPCR0 على SH7734
```

لو الـ bit لسا 1، يعني الـ clock ما اتفعّل، وده تأكيد للمشكلة.

#### الدرس المستفاد
**الـ access size في MSTP مهم جداً.** الـ macros `SH_CLK_MSTP8` و`SH_CLK_MSTP16` و`SH_CLK_MSTP32` مش مجرد style — هي بتحدد كيف الـ framework يكتب للـ register. على SuperH، معظم الـ MSTP registers هي 32-bit، لازم تستخدم `SH_CLK_MSTP32` أو `SH_CLK_MSTP32_STS` دايماً إلا لو الـ datasheet بيقول غير كده صراحةً.

---

### السيناريو 2: R-Mobile A1 — الـ display يتجمد عند تغيير الـ frequency

#### العنوان
الـ HDMI output يتقطع ويتجمد على لوحة تعمل بـ Renesas R-Mobile A1 عند تغيير الـ CPU frequency

#### السياق
منتج **Android TV box** يستخدم **Renesas R-Mobile A1** (نفس architecture الـ SuperH المستخدم في mobile). الجهاز يشغل فيديو 1080p عبر **HDMI**. المستخدم لما يفتح تطبيق ثقيل والـ CPU يعمل **cpufreq scaling**، الـ HDMI يتجمد لثانية أو اثنتين أو أحياناً يقطع بالكامل.

#### المشكلة
الـ HDMI controller يعتمد على clock مشتق من نفس PLL اللي بيتغير عند الـ frequency scaling. المشكلة مش في الـ HDMI driver نفسه — في طريقة تعريف الـ clock tree وكيف الـ `propagate_rate` بيشتغل.

#### التحليل
في `sh_clk.h`:
```c
void propagate_rate(struct clk *);
```

لما الـ CPU clock يتغير، الـ `propagate_rate` بيتمشى على كل الـ children اللي في `struct clk`:
```c
struct list_head    children;
struct list_head    sibling;  /* node for children */
```

الـ HDMI clock كان معرّف كـ child مباشر للـ PLL بدون أي buffer أو divider مستقل:
```c
/* clock tree خاطئ */
static struct clk hdmi_clk = {
    .parent = &pll_clk,    /* مباشرة من PLL */
    .ops    = &hdmi_clk_ops,
};
```

لما PLL rate تتغير، `propagate_rate` بيستدعي `recalc` على كل child بما فيهم `hdmi_clk`. لو الـ `recalc` في `sh_clk_ops` ما بيعمل لا شيء أو بيرجّع قيمة غلط:
```c
struct sh_clk_ops {
    unsigned long (*recalc)(struct clk *clk);
    /* ... */
};
```

الـ HDMI hardware بيشوف clock مختلف مش متزامن مع الـ PLL output الجديد، فالـ pixel clock بيصير wrong وتحدث desynchronization.

المشكلة التانية: الـ `CLK_MASK_DIV_ON_DISABLE` flag مش متحطوط على الـ DIV6 clock اللي بيغذّي الـ HDMI:
```c
#define CLK_MASK_DIV_ON_DISABLE BIT(4)
```
هذا الـ flag المفروض يضمن إن الـ divider يرجع لأكبر قيمة ممكنة (تقسيم أكتر) وقت الـ disable عشان ما يكون في glitch.

#### الحل
1. إضافة intermediate divider مستقل للـ HDMI:
```c
/* استخدام DIV6 مع CLK_MASK_DIV_ON_DISABLE */
static struct clk hdmi_clk = SH_CLK_DIV6(&pll_clk, HDMI_DIV_REG,
                                          CLK_ENABLE_ON_INIT);
/* SH_CLK_DIV6 يضيف CLK_MASK_DIV_ON_DISABLE تلقائياً */
```

2. تسجيل notifier على الـ PLL clock لتجميد الـ HDMI output قبل التغيير:
```c
/* في الـ HDMI driver */
static int hdmi_clk_notifier(struct notifier_block *nb,
                              unsigned long event, void *data)
{
    if (event == PRE_RATE_CHANGE)
        hdmi_disable_output(); /* تجميد مؤقت */
    else if (event == POST_RATE_CHANGE)
        hdmi_reconfigure_and_enable(); /* إعادة تهيئة */
    return NOTIFY_OK;
}
```

#### الدرس المستفاد
**الـ `propagate_rate` بيأثر على كل الـ children.** أي clock حساس للتوقيت زي HDMI أو MIPI-DSI لازم يكون له divider مستقل (DIV6 مع `CLK_MASK_DIV_ON_DISABLE`) وnotifier يحميه وقت التغيير. هذا الـ flag في `sh_clk.h` مش تفصيلة — هو ضمان عدم حدوث glitches.

---

### السيناريو 3: SH7372 — kernel panic عند تسجيل MSTP clocks

#### العنوان
الـ kernel يعمل panic أثناء الـ boot على SH7372 عند استدعاء `sh_clk_mstp_register`

#### السياق
مهندس يعمل **board bring-up** لبورد جديد مبني على **Renesas SH7372** (استخدم في أجهزة مثل Sony Tablet). هو بيحاول إضافة دعم لـ I2C controller جديد متصل بمحسسات حرارة لنظام **automotive ECU**.

#### المشكلة
الـ kernel يعمل panic بـ:
```
BUG: unable to handle kernel NULL pointer dereference at 00000008
EIP: sh_clk_mstp_register+0x3c
```

#### التحليل
المهندس يفتح `sh_clk.h` ويشوف تعريف `SH_CLK_MSTP`:
```c
#define SH_CLK_MSTP(_parent, _enable_reg, _enable_bit, _status_reg, _flags) \
{                                       \
    .parent     = _parent,              \
    .enable_reg = (void __iomem *)_enable_reg, \
    .enable_bit = _enable_bit,          \
    .status_reg = _status_reg,          \
    .flags      = _flags,               \
}
```

المهندس عرّف الـ I2C clock هكذا:
```c
static struct clk mstp_clks[MSTP_NR] = {
    [MSTP_I2C] = SH_CLK_MSTP32(&p_clk, MSTPCR1, 16, 0),
};
```

المشكلة: الـ `p_clk` لم يُسجَّل بعد! الـ field `.parent` في `struct clk` هو pointer:
```c
struct clk {
    struct list_head    node;
    struct clk          *parent;    /* <-- هذا pointer */
    /* ... */
};
```

لما `sh_clk_mstp_register` بتحاول تتحقق من الـ parent clock (عشان تحسب الـ rate)، الـ `p_clk` يكون struct موجود في BSS لكن ما اتسجلش في الـ clock framework بعد — يعني `p_clk.node` فاضية والـ `p_clk.rate` صفر.

الـ framework بيعمل `followparent_recalc`:
```c
unsigned long followparent_recalc(struct clk *);
```
اللي بيحاول يقرأ `clk->parent->rate` — وهنا الـ panic لأن الـ parent نفسه ما اتسجلش والـ linked list فيه داتا garbage.

#### الحل
ترتيب التسجيل مهم جداً — الـ parent لازم يتسجل أول:
```c
/* في arch/sh/kernel/cpu/sh4a/clock-sh7372.c */
static int __init arch_clk_init(void)
{
    int ret = 0;

    /* أولاً: تسجيل الـ root clocks */
    ret = clk_register(&r_clk);
    ret |= clk_register(&p_clk);   /* <-- p_clk يتسجل هنا */
    ret |= clk_register(&b_clk);

    /* ثانياً: تسجيل الـ MSTP clocks اللي تعتمد على p_clk */
    ret |= sh_clk_mstp_register(mstp_clks, ARRAY_SIZE(mstp_clks));

    return ret;
}
```

للتأكد من الترتيب الصح:
```bash
# بعد boot ناجح، شيك على الـ clock tree
cat /sys/kernel/debug/clk/clk_summary
# أو
cat /proc/clocks  # على kernels قديمة
```

#### الدرس المستفاد
**الـ clock registration order حرفي.** الـ `struct clk` في `sh_clk.h` بيستخدم raw pointers للـ parent — مش handles أو strings. لازم كل parent يكون مسجلاً في الـ framework قبل ما تسجّل أي child بيعتمد عليه. ده مختلف عن الـ Common Clock Framework الحديث اللي بيدعم late binding.

---

### السيناريو 4: SH7786 — SPI لا يتجاوز 1MHz على بورد صناعي

#### العنوان
الـ SPI controller على SH7786 يرفض العمل فوق 1MHz رغم إن الـ spec يدعم 25MHz

#### السياق
**industrial gateway** يستخدم **Renesas SH7786** للتواصل مع flash chip خارجي عبر **SPI**. الـ flash يدعم 25MHz، لكن الـ driver دايماً يشتغل بـ 1MHz أو أقل. ده بيخلي قراءة الـ firmware updates بطيئة جداً — 30 ثانية بدل ثانيتين.

#### المشكلة
الـ `clk_rate_table_round` بيرجع قيمة أقل بكتير من المطلوب.

#### التحليل
في `sh_clk.h` الـ function:
```c
long clk_rate_table_round(struct clk *clk,
                          struct cpufreq_frequency_table *freq_table,
                          unsigned long rate);
```

هذه الـ function بتبحث في الـ `freq_table` عن أقرب rate أقل من أو يساوي المطلوب.

الـ SPI clock معرّف كـ DIV4:
```c
static struct clk spi_clk = SH_CLK_DIV4(&p_clk, FRQCR, 12,
                                          0x0F,   /* div_bitmap: divisors 2,4,8,16 فقط */
                                          0);
```

الـ `arch_flags` = `0x0F` يعني الـ `div_bitmap` بيسمح بـ divisors معينة فقط. الـ `SH_CLK_DIV4_MSK`:
```c
#define SH_CLK_DIV4_MSK  SH_CLK_DIV_MSK(4)  /* = (1<<4)-1 = 0xF */
```

الـ `div_mask` = `0xF` يعني 4 bits = divisors من 1 لـ 16.

المشكلة: الـ `div_bitmap` كان `0x0F` بالهيكس يعني bits 0-3 set — ده يعني divisors رقم 1,2,4,8 (بالـ 0-index). لكن على SH7786 الـ actual divisors هي {2, 4, 8, 16, 32, 64} — فـ bit 0 يعني ÷2 مش ÷1.

لما الـ parent clock = 66MHz:
- ÷2 = 33MHz (مش في الـ bitmap الصح)
- ÷4 = 16.5MHz
- أقرب لـ 25MHz هو 16.5MHz

لكن الـ `clk_rate_table_build` بنى الـ `freq_table` بشكل غلط لأن الـ `div_bitmap` مش متطابق مع الـ hardware:
```c
void clk_rate_table_build(struct clk *clk,
                          struct cpufreq_frequency_table *freq_table,
                          int nr_freqs,
                          struct clk_div_mult_table *src_table,
                          unsigned long *bitmap);
```

الـ `bitmap` المرسول كان يمثل divisors مختلفة عن الـ hardware.

#### الحل
تصحيح الـ `div_bitmap` ليطابق الـ hardware divisors الفعلية:
```c
/* الـ SH7786 SPI clock: divisors المتاحة هي 2,4,8,16 */
/* bit 0 = ÷2, bit 1 = ÷4, bit 2 = ÷8, bit 3 = ÷16 */
static struct clk spi_clk = SH_CLK_DIV4(&p_clk, FRQCR, 12,
                                          DIV4_SPI_STD,  /* bitmap محسوب صح */
                                          0);
```

للتشخيص:
```bash
# فحص الـ rate الفعلي
cat /sys/kernel/debug/clk/spi_clk/clk_rate

# طلب rate معين وشيك على اللي اتحدد فعلاً
# في الـ driver code
pr_info("SPI requested rate: %lu, actual rate: %lu\n",
        requested, clk_get_rate(spi_clk));
```

#### الدرس المستفاد
**الـ `div_bitmap` في DIV4 clocks لازم يطابق الـ hardware بالضبط.** كل bit في الـ bitmap يمثل divisor محدد حسب الـ `clk_div_mult_table`. خطأ في الـ bitmap يخلي `clk_rate_table_build` يبني freq_table غلط، و`clk_rate_table_round` يختار دايماً قيمة أقل من المطلوب. دايماً قارن الـ bitmap مع الـ hardware manual قبل ما تكتب الـ clock definitions.

---

### السيناريو 5: SH7734 — memory leak بعد hot-plug لـ USB device

#### العنوان
الـ system memory بيتآكل تدريجياً على SH7734 كل ما يتعمل plug/unplug لـ USB device

#### السياق
**industrial panel PC** يشتغل بـ **Renesas SH7734** بيستخدمه مصنع كـ operator interface. البورد عنده **USB host port** للـ barcode scanners. بعد أسبوع من الشغل، الـ system بيبدأ يتباطأ، وبعد أسبوعين يموت بـ OOM (out of memory).

#### المشكلة
كل عملية USB plug/unplug بتعمل memory leak صغير. بعد آلاف العمليات، الـ memory بتخلص.

#### التحليل
المهندس يفتح `sh_clk.h` ويلاحظ الـ struct:
```c
struct clk_mapping {
    phys_addr_t      phys;
    void __iomem     *base;
    unsigned long    len;
    struct kref      ref;   /* <-- reference counting */
};
```

وفي `struct clk`:
```c
struct clk_mapping   *mapping;
```

الـ `clk_mapping` بيستخدم `kref` للـ reference counting. الفكرة إن لما clock متعدد بيشاركوا نفس الـ memory-mapped region، بدل ما كل واحد يعمل `ioremap` لحاله، بيتشاركوا نفس الـ `clk_mapping`.

المشكلة في الـ USB clock driver:
```c
/* لما USB device يتعمل probe */
static int usb_clk_enable(struct clk *clk)
{
    /* هنا كان في كود يعمل ioremap جديد في كل مرة */
    clk->mapping = kzalloc(sizeof(*clk->mapping), GFP_KERNEL);
    if (!clk->mapping) return -ENOMEM;

    clk->mapping->base = ioremap(USB_CLK_REG, 4);
    clk->mapping->phys = USB_CLK_REG;
    clk->mapping->len  = 4;
    kref_init(&clk->mapping->ref);
    /* ... */
}
```

لكن في الـ disable:
```c
static void usb_clk_disable(struct clk *clk)
{
    /* نسي يعمل kref_put أو iounmap */
    clk_write_reg(0, clk);  /* بس disable الـ bit */
}
```

لأن `kref_put` ما اتاستدعاش، الـ `clk_mapping` ما اتحرّرش، و`iounmap` ما اتاستدعاش، فكل plug/unplug بيعمل leak لـ:
- `sizeof(struct clk_mapping)` = عدة bytes
- `ioremap` region (مش kernel memory مباشرة، لكن VMAs بتتراكم)

بعد وقت، الـ virtual address space بيمتلي والـ `ioremap` calls بتفشل.

#### الحل
تصحيح الـ enable/disable pair ليستخدم الـ `kref` صح:
```c
static void usb_clk_mapping_release(struct kref *kref)
{
    struct clk_mapping *mapping = container_of(kref,
                                               struct clk_mapping, ref);
    iounmap(mapping->base);
    kfree(mapping);
}

static int usb_clk_enable(struct clk *clk)
{
    /* تحقق لو الـ mapping موجود بالفعل */
    if (!clk->mapping) {
        clk->mapping = kzalloc(sizeof(*clk->mapping), GFP_KERNEL);
        if (!clk->mapping) return -ENOMEM;

        clk->mapping->base = ioremap(USB_CLK_REG, 4);
        clk->mapping->phys = USB_CLK_REG;
        clk->mapping->len  = 4;
        kref_init(&clk->mapping->ref); /* ref = 1 */
    } else {
        kref_get(&clk->mapping->ref);  /* زيادة ref */
    }
    /* تفعيل الـ clock bit */
    return 0;
}

static void usb_clk_disable(struct clk *clk)
{
    /* تعطيل الـ clock bit أولاً */
    clk_write_reg(0, clk);

    /* تحرير الـ mapping لو مفيش references تانية */
    if (clk->mapping)
        kref_put(&clk->mapping->ref, usb_clk_mapping_release);
}
```

للتشخيص:
```bash
# مراقبة الـ vmalloc/ioremap usage
cat /proc/vmallocinfo | grep -c ioremap

# مراقبة memory قبل وبعد plug/unplug
watch -n 1 'cat /proc/meminfo | grep -E "MemFree|VmallocUsed"'

# كل plug/unplug لازم ما يغيّر الأرقام بشكل تراكمي
```

#### الدرس المستفاد
**الـ `kref` في `clk_mapping` مش decoration — هو ضمان عدم الـ leak.** كل `kref_init` أو `kref_get` لازم يقابله `kref_put` مع release function صح. في الـ SuperH clock framework، الـ `clk_mapping` مصمم للمشاركة بين clocks متعددة تشاركوا نفس الـ physical register region. استخدامه بشكل فردي لكل clock بدون مشاركة هو هدر، والنسيان في الـ release هو leak مضمون. راجع دايماً إن لكل path بيعمل `enable` في `sh_clk_ops` يوجد مقابله `disable` يحرر نفس الموارد بالضبط.
## Phase 7: مصادر ومراجع

---

### مقدمة سريعة

الـ `sh_clk.h` هو جزء من نظام إدارة الـ clock الخاص بمعالجات **SuperH (SH)** من Renesas. هذا النظام سابق للـ **Common Clock Framework** اللي دخل الـ kernel في الإصدار 3.4. المصادر التالية تساعدك تفهم السياق التاريخي والتقني لهذا الـ subsystem.

---

### أولاً: مقالات LWN.net

| المقالة | الوصف | الرابط |
|---------|-------|--------|
| **A common clock framework** | المقالة الأساسية اللي شرحت فيها فكرة الـ Common Clock Framework قبل ما يُدمج في الـ kernel — مهمة لفهم ليش كان الـ sh_clk framework موجود كبديل مؤقت | [lwn.net/Articles/472998](https://lwn.net/Articles/472998/) |
| **common clk framework** (متابعة) | متابعة للنقاش حول تصميم الـ framework وكيفية توحيد كل الـ arch-specific clock code | [lwn.net/Articles/486841](https://lwn.net/Articles/486841/) |
| **Documentation/clk.txt** | نقاش حول التوثيق الرسمي للـ framework وكيف يُستخدم من الـ driver developers | [lwn.net/Articles/489668](https://lwn.net/Articles/489668/) |
| **Resurrecting the SuperH architecture** | مقالة تتكلم عن إحياء معمارية SuperH ومشروع J2 الـ open-source core — تعطيك سياق تاريخي عن SH في الـ kernel | [lwn.net/Articles/647636](https://lwn.net/Articles/647636/) |
| **The Common Clk Framework** (توثيق رسمي على LWN) | نسخة من التوثيق الرسمي للـ kernel clock API على LWN | [static.lwn.net/kerneldoc/driver-api/clk.html](https://static.lwn.net/kerneldoc/driver-api/clk.html) |

---

### ثانياً: التوثيق الرسمي للـ Kernel

هذه المسارات تشير لملفات داخل شجرة الـ kernel source:

```
Documentation/driver-api/clk.rst          # الـ Common Clock Framework API الرسمي
Documentation/arch/sh/                    # توثيق معمارية SuperH
Documentation/devicetree/bindings/clock/renesas,cpg-mstp-clocks.txt  # bindings خاصة بـ MSTP
```

الـ **الروابط الإلكترونية** للتوثيق الرسمي:

- [The Common Clk Framework — docs.kernel.org](https://docs.kernel.org/driver-api/clk.html) — المرجع الأساسي لكل الـ clock API في الـ kernel الحديث
- [SuperH Interfaces Guide — docs.kernel.org](https://docs.kernel.org/arch/sh/index.html) — دليل الـ interfaces الخاص بمعمارية SH
- [Feature status on SH architecture](https://docs.kernel.org/arch/sh/features.html) — حالة الميزات المدعومة على SH
- [renesas,cpg-mstp-clocks.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/clock/renesas,cpg-mstp-clocks.txt) — Device Tree bindings للـ MSTP clocks

---

### ثالثاً: مناقشات الـ Mailing List وتاريخ الـ Commits

#### أهم الـ Patchwork Links:

| الرقم | الموضوع | الرابط |
|-------|---------|--------|
| sh 2.6.31-rc1 | أول pull request للـ SH updates يتضمن تطوير الـ clock framework | [spinics.net/lists/linux-sh/msg03170.html](https://www.spinics.net/lists/linux-sh/msg03170.html) |
| MSTP wakeup | تعديل على الـ MSTP clocks لدعم الـ wakeup sources أثناء system suspend | [patchwork.kernel.org — mstp wakeup](https://patchwork.kernel.org/project/linux-renesas-soc/patch/1510234022-29442-2-git-send-email-geert+renesas@glider.be/) |
| R-Car CPG/MSSR | إضافة driver جديد للـ Clock Pulse Generator على R-Car Gen3 | [patchwork.kernel.org — R8A7791](https://patchwork.kernel.org/patch/9698795/) |
| sh-sci interface clock | إزالة الـ interface clock الخاص بـ serial driver كجزء من ترحيل الـ sh_clk | [patchwork.kernel.org — sh-sci](https://patchwork.kernel.org/project/linux-sh/patch/1450119456-964-2-git-send-email-geert+renesas@glider.be/) |

#### البحث عن commits مباشرة:

```bash
# البحث في تاريخ الـ kernel عن sh_clk
git log --oneline -- include/linux/sh_clk.h
git log --oneline -- drivers/sh/clk.c
git log --oneline -- drivers/clk/renesas/
```

---

### رابعاً: kernelnewbies.org

الـ **الصفحة المرتبطة بالـ SuperH clock** على kernelnewbies تظهر في سياق kernel 2.6.31 اللي كانت نقطة تحول مهمة:

- [Linux 2.6.31 — kernelnewbies.org](https://kernelnewbies.org/Linux_2_6_31) — هذا الإصدار أضاف:
  - دعم SuperH MTU2 Timer driver
  - دعم SuperH TMU Timer driver
  - دعم CMT clockevents لمعالج sh7724

الـ [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) يتتبع كل التغييرات الكبيرة في كل إصدار kernel.

---

### خامساً: elinux.org

- [Linux Kernel Resources — elinux.org](https://elinux.org/Linux_Kernel_Resources) — صفحة تجمع موارد الـ kernel مرتبة حسب الـ architecture، تحت قسم Architecture Sites تجد إشارة لـ SuperH

> ملاحظة: elinux.org لا تحتوي على صفحات مخصصة لـ sh_clk بشكل مباشر، لكن [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) هي نقطة انطلاق جيدة للبحث عن موارد الـ embedded Linux.

---

### سادساً: الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: Linux Device Model — يشرح كيف تُسجَّل الـ devices والـ clocks في الـ kernel
- **الفصل 1**: مقدمة عن الـ kernel وكيفية تنظيمه
- متاح مجاناً على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 11**: Timers and Time Management — يشرح الـ clock sources وعلاقتها بالـ timekeeping
- **الفصل 17**: Devices and Modules — يشرح تسجيل الـ devices والـ clocks
- مهم لفهم الفرق بين الـ **clock source** والـ **clock event** وإدارة الـ power

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 15**: Embedded Linux on Modern Processors — يتكلم عن إدارة الـ clock والـ power في أنظمة الـ embedded
- يغطي مفاهيم الـ clock gating وكيف تؤثر على الـ power consumption

#### Programming Embedded Systems — Michael Barr
- يشرح مفاهيم الـ oscillator والـ PLL والـ clock divider من منظور hardware
- مفيد لفهم ما يجري داخل الـ CPG (Clock Pulse Generator) الخاص بـ Renesas

---

### سابعاً: مصادر إضافية متخصصة

#### توثيق الـ Bootlin (Formerly Free Electrons)
- [Common Clock Framework — ELCE 2013 (PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) — شرح عملي ممتاز لكيفية استخدام الـ Common Clock Framework

#### توثيق Renesas الرسمي
- ابحث عن **User's Manual** لكل معالج SH من [renesas.com](https://www.renesas.com) — تحت قسم "Clock Pulse Generator (CPG)"
- الـ datasheet يشرح كيف تعمل الـ MSTP bits فعلياً على الـ hardware

#### Kernel Source Code (المرجع الأكيد)
```
include/linux/sh_clk.h          # الملف الرئيسي للـ data structures
drivers/sh/clk.c                # التنفيذ الأساسي للـ sh_clk framework
drivers/sh/clk-cpg.c            # الـ CPG legacy clock handling
drivers/clk/renesas/            # الـ modern clock drivers لمعالجات Renesas
arch/sh/kernel/cpu/             # إعداد الـ clocks لكل نوع معالج SH
```

---

### ثامناً: كلمات البحث المفيدة

استخدم هذه الـ search terms للبحث عن معلومات إضافية:

```
"sh_clk" linux kernel
"SuperH clock framework" linux
"SH_CLK_MSTP" linux kernel
"CPG MSTP" renesas linux
"clk_div4_table" SuperH
"sh_clk_ops" linux
"CLK_ENABLE_ON_INIT" SuperH
"sh_clk_fsidiv" linux kernel
"clk_reparent" SuperH linux
linux-sh mailing list clock
renesas clock gating linux driver
```

للبحث في الـ LKML (Linux Kernel Mailing List):
- [lore.kernel.org/linux-sh/](https://lore.kernel.org/linux-sh/) — أرشيف قائمة بريد SuperH الرسمية
- [lore.kernel.org/linux-clk/](https://lore.kernel.org/linux-clk/) — أرشيف قائمة بريد الـ clock framework

---

### ملخص المصادر الأهم

| الأولوية | المصدر | السبب |
|----------|--------|-------|
| ⭐⭐⭐ | [docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html) | التوثيق الرسمي للـ Common Clock Framework |
| ⭐⭐⭐ | [docs.kernel.org/arch/sh/index.html](https://docs.kernel.org/arch/sh/index.html) | دليل SuperH الرسمي في الـ kernel |
| ⭐⭐⭐ | [lwn.net/Articles/472998](https://lwn.net/Articles/472998/) | فهم لماذا جاء الـ CCF ليحل محل أنظمة مثل sh_clk |
| ⭐⭐ | [lwn.net/Articles/647636](https://lwn.net/Articles/647636/) | السياق التاريخي لمعمارية SuperH في Linux |
| ⭐⭐ | [kernelnewbies.org/Linux_2_6_31](https://kernelnewbies.org/Linux_2_6_31) | نقطة تحول مهمة في تطوير sh_clk |
| ⭐⭐ | [kernel.org/doc/Documentation/devicetree/bindings/clock/renesas,cpg-mstp-clocks.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/clock/renesas,cpg-mstp-clocks.txt) | DT bindings للـ MSTP clocks |
| ⭐ | [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) | موارد عامة للـ embedded Linux |
| ⭐ | LDD3 الفصل 14 | فهم الـ device model المرتبط بالـ clocks |
## Phase 8: Writing simple module

### الفكرة العامة

الـ `sh_clk.h` بيعرّف نظام الـ clock الخاص بمعالجات SuperH. فيه دوال exported زي `clk_register()` و`clk_unregister()` اللي بتتسجل فيها أي clock جديدة في النظام. فكرتنا هي نعمل **kprobe** على `clk_register` عشان نشوف كل مرة أي كود في الكيرنل بيحاول يسجل clock جديدة — نطبع معلوماتها زي الـ rate والـ flags والـ usecount.

تخيل إنك واقف عند باب محطة القطر وكل ما حد يدخل تكتب اسمه وميعاده — هو ده بالضبط الـ kprobe.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * sh_clk_register_probe.c
 *
 * kprobe على clk_register() من sh_clk
 * بيطبع معلومات الـ clock اللي بتتسجل
 */

#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/sh_clk.h>      /* struct clk */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe على clk_register لمراقبة تسجيل sh_clk clocks");

/*
 * هنا بنعرّف الـ kprobe struct.
 * الـ .symbol_name بيقول للكيرنل: اربط الـ probe على رأس الدالة دي.
 */
static struct kprobe kp = {
    .symbol_name = "clk_register",
};

/*
 * الـ pre_handler: بيتنادى لحظة ما الـ CPU وصل لأول تعليمة في clk_register،
 * قبل ما الدالة الأصلية تشتغل.
 *
 * @p   : مؤشر للـ kprobe نفسه (مش محتاجينه هنا لكن الـ signature يستلزمه)
 * @regs: سجلات الـ CPU وقت الـ probe — منها بنجيب الـ argument الأول
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * الـ argument الأول لـ clk_register(struct clk *clk) موجود في:
     *   - x86_64  : regs->di
     *   - ARM64   : regs->regs[0]
     *   - ARM32   : regs->ARM_r0
     * بنستخدم regs->di هنا كمثال لـ x86_64.
     * لو شغّلته على ARM غيّرها لـ regs->regs[0].
     */
    struct clk *clk = (struct clk *)regs->di;

    /* تأكد إن المؤشر مش NULL قبل ما تقرأ منه */
    if (!clk) {
        pr_info("[sh_clk_probe] clk_register نودي بـ NULL pointer!\n");
        return 0;
    }

    /*
     * نطبع أهم حقول الـ struct clk:
     *  - rate    : تردد الـ clock بالـ Hz
     *  - flags   : خصائص زي CLK_ENABLE_ON_INIT
     *  - usecount: كام حاجة شغّالة بتعتمد عليه دلوقتي
     */
    pr_info("[sh_clk_probe] clk_register() اتنادت!\n");
    pr_info("[sh_clk_probe]   rate     = %lu Hz\n",  clk->rate);
    pr_info("[sh_clk_probe]   flags    = 0x%lx\n",   clk->flags);
    pr_info("[sh_clk_probe]   usecount = %d\n",       clk->usecount);

    /* لو فيه parent نطبع عنوانه كمرجع */
    if (clk->parent)
        pr_info("[sh_clk_probe]   parent   = %px\n", clk->parent);
    else
        pr_info("[sh_clk_probe]   parent   = (root clock, no parent)\n");

    return 0; /* لازم يرجع 0 — أي قيمة تانية بتخلي الكيرنل يعمل fault */
}

/*
 * module_init: بيتنادى لما تعمل insmod
 * بنسجّل الـ kprobe هنا.
 */
static int __init sh_clk_probe_init(void)
{
    int ret;

    kp.pre_handler = handler_pre; /* ربط الـ callback بالـ kprobe */

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[sh_clk_probe] فشل تسجيل الـ kprobe: %d\n", ret);
        return ret;
    }

    pr_info("[sh_clk_probe] الـ kprobe اتسجّل على '%s' عند العنوان %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/*
 * module_exit: بيتنادى لما تعمل rmmod
 * لازم نشيل الـ kprobe عشان ما نسيبش pointer معلّق على كود اتشال.
 */
static void __exit sh_clk_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[sh_clk_probe] الـ kprobe اتشال من '%s'\n", kp.symbol_name);
}

module_init(sh_clk_probe_init);
module_exit(sh_clk_probe_exit);
```

---

### Makefile للـ module

```makefile
obj-m += sh_clk_register_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | ليه موجود؟ |
|---|---|
| `linux/kernel.h` | عشان `pr_info` و`pr_err` |
| `linux/module.h` | عشان `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | عشان `struct kprobe` و`register_kprobe` و`unregister_kprobe` |
| `linux/sh_clk.h` | عشان `struct clk` اللي بنقرأ منه الـ rate والـ flags |

الـ includes دي زي أدوات النجار — كل أداة ليها شغلة واحدة بالظبط.

---

#### الـ `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "clk_register",
};
```

ده بيقول للكيرنل: "اربط الـ probe على الدالة اللي اسمها `clk_register`". الكيرنل هيدور على العنوان الحقيقي في الـ kallsyms جوّاه وحده.

---

#### الـ `handler_pre`

ده القلب. بيتنادى قبل ما `clk_register` تشتغل بالفعل.

**الـ `regs->di`** على x86_64 هو أول argument للدالة — وده بالظبط مؤشر الـ `struct clk`. بنعمل cast وبنقرأ منه `rate` و`flags` و`usecount` ونطبعهم بـ `pr_info`.

اللي بنطبعه مفيد جداً في الـ debugging: تخيل إنك شايف clock اتسجّلت بـ rate = 0 — يبقى فيه مشكلة في الـ initialization.

---

#### الـ `module_init` والـ `module_exit`

- الـ `init` بيستدعي `register_kprobe` — ده زي "اتكيّف عند الباب وابدأ المراقبة".
- الـ `exit` بيستدعي `unregister_kprobe` — لازم نعمل ده قبل ما الـ module يتشال من الذاكرة، لأن لو سبنا الـ kprobe مسجّل والكود بتاعنا اتشال، الكيرنل هيـcrash لما الـ probe يتنادى.

---

### كيفية التجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod sh_clk_register_probe.ko

# شوف الـ logs
sudo dmesg | grep sh_clk_probe

# لو عندك كود بيسجّل clock تقدر تحرّكه وتشوف الـ output

# إزالة الـ module
sudo rmmod sh_clk_register_probe
sudo dmesg | grep sh_clk_probe
```

**ملحوظة مهمة:** الدالة `clk_register` موجودة في `drivers/sh/clk.c` وهي خاصة بمعالجات SuperH. لو شغّلت على kernel مش فيه CONFIG_SH_CLK_CPG أو مشابه، الـ `register_kprobe` هترجع `-ENOENT` لأن الـ symbol مش موجود في الـ kallsyms. في الحالة دي تقدر تستبدل `clk_register` بأي symbol exported تاني زي `clk_rate_table_build`.
