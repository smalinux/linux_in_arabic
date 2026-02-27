## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem اللي ينتمي له الملف ده؟

**الـ `clk-multiplier.c`** جزء من **Common CLK Framework** — الإطار العام لإدارة الـ clocks في Linux kernel. المسؤولون عنه هما Michael Turquette و Stephen Boyd، والملفات بتاعته موجودة في:

- `drivers/clk/`
- `include/linux/clk-provider.h`
- `include/linux/clk.h`

---

### قصة المشكلة — تخيل معايا

تخيل إنك عندك **مصنع كبير** فيه ماكينات كتير. كل ماكينة محتاجة تشتغل بـ **سرعة معينة** (frequency). فيه **مصدر طاقة واحد رئيسي** (oscillator) بيدي تردد ثابت — مثلاً 100 MHz.

المشكلة: أنا عايز ماكينة تشتغل بـ **300 MHz** — ثلاث أضعاف المصدر. إيه الحل؟

**الحل هو الـ Clock Multiplier** — زي ما بضرب في جير، بضرب تردد الـ clock الأصلي في رقم معين عشان أوصل للتردد اللي أنا محتاجه.

```
[Parent Clock: 100 MHz] → [×3 Multiplier] → [Output: 300 MHz]
```

ده بالظبط اللي بيعمله `clk-multiplier.c` — **بيوفر driver عام لأي hardware register بيحتوي على معامل ضرب للـ clock**.

---

### الـ Big Picture بشكل أعمق

#### الـ Common CLK Framework — السياق الكامل

الـ Linux kernel فيه آلاف الـ chips — كل chip عندها طريقة مختلفة للتحكم في الـ clocks. قبل الـ Common CLK Framework (اتضاف في 2012)، كل platform كانت بتعمل كود خاص بيها من الصفر. النتيجة: **كود متكرر ومشتت**.

الحل: Framework موحد فيه:
1. **Core layer** (`clk.c`) — يتعامل مع الـ clock tree، الـ enable/disable، الـ rate propagation.
2. **Building blocks** — drivers صغيرة لكل نوع من أنواع الـ clock:

| الملف | الوظيفة |
|-------|---------|
| `clk-fixed-rate.c` | clock بتردد ثابت |
| `clk-gate.c` | clock ممكن تفتح/تقفل (enable/disable) |
| `clk-divider.c` | clock بتقسم التردد (÷N) |
| `clk-mux.c` | clock بتختار من أكتر من مصدر |
| `clk-fixed-factor.c` | multiplier + divider بنسبة ثابتة |
| **`clk-multiplier.c`** | **clock بتضرب التردد (×N) من register** |

---

### ماذا يفعل `clk-multiplier.c` تحديداً؟

الملف ده بيوفر **clock type عام** اسمه `clk_multiplier` — أي hardware عنده **register** فيه بت-فيلد بيحدد عامل الضرب، تقدر تستخدم الـ driver ده مباشرةً.

#### الـ struct الأساسية:

```c
struct clk_multiplier {
    struct clk_hw   hw;      /* ربط بالـ common framework */
    void __iomem   *reg;     /* عنوان الـ register في الـ hardware */
    u8              shift;   /* بداية الـ bit field جوه الـ register */
    u8              width;   /* عدد البتات اللي بتمثل الـ multiplier */
    u8              flags;   /* خيارات إضافية */
    spinlock_t     *lock;    /* حماية من الـ race conditions */
};
```

#### مثال واقعي — تخيل الـ register:

```
Register (32-bit):
Bits: [31 ... 8 | 7  6  5  4  3  2  1  0]
                  ^                      ^
               bit 7                  bit 0

لو shift=4, width=3:
Bits [6:4] = 011 (binary) = 3 → التردد = parent_rate × 3
```

---

### الـ Operations الثلاث اللي بيعملها الملف

#### 1. `clk_multiplier_recalc_rate` — احسب التردد الحالي
بتقرأ الـ register، بتعزل الـ bits بتاعة الـ multiplier، وبتضرب في الـ parent rate.

```c
val = clk_mult_readl(mult) >> mult->shift;   /* shift للـ field */
val &= GENMASK(mult->width - 1, 0);          /* عزل الـ bits */
return parent_rate * val;                     /* الناتج */
```

#### 2. `clk_multiplier_determine_rate` — حدد أفضل multiplier لتردد معين
لو طلبت تردد معين، الـ function دي بتحسب أنسب قيمة للـ multiplier وأفضل parent rate ممكن.

المنطق في `__bestmult`:
- لو مش مسموح تغير الـ parent → قسّم التردد المطلوب على الـ parent الحالي وخذ الأقرب.
- لو مسموح تغير الـ parent → جرّب كل قيم الـ multiplier الممكنة وابحث عن أفضل تركيبة.

#### 3. `clk_multiplier_set_rate` — اكتب الـ multiplier في الـ register
بتحسب الـ factor المناسب، بتعمل read-modify-write على الـ register مع حماية الـ spinlock.

```c
val = clk_mult_readl(mult);
val &= ~GENMASK(mult->width + mult->shift - 1, mult->shift);  /* امسح الـ bits القديمة */
val |= factor << mult->shift;                                   /* اكتب الـ value الجديدة */
clk_mult_writel(mult, val);
```

---

### الـ Flags المتاحة

| الـ Flag | المعنى |
|---------|--------|
| `CLK_MULTIPLIER_ZERO_BYPASS` | لو الـ register = 0، اعتبره = 1 (ماتوقفش) |
| `CLK_MULTIPLIER_ROUND_CLOSEST` | تقريب للأقرب بدل تقريب للأسفل |
| `CLK_MULTIPLIER_BIG_ENDIAN` | اقرأ/اكتب الـ register بـ big-endian |

---

### الصورة الكاملة في الـ Clock Tree

```
[Crystal Oscillator: 24 MHz]
          |
          ▼
   [clk-fixed-rate]        ← مصدر ثابت
          |
          ▼
   [clk-multiplier ×N]     ← ده اللي في ملفنا
          |
          ▼
   [clk-divider ÷M]        ← للتضبيط الدقيق
          |
          ▼
   [clk-gate]              ← enable/disable
          |
          ▼
   [Peripheral: USB, UART, GPU...]
```

---

### الملفات اللي المفروض تعرفها

#### Core Framework:
- **`drivers/clk/clk.c`** — قلب الـ framework، بيدير الـ clock tree كلها
- **`include/linux/clk-provider.h`** — تعريف `clk_hw`، `clk_ops`، `clk_multiplier`، وكل الـ structs الأساسية
- **`include/linux/clk.h`** — الـ API اللي بيستخدمها الـ consumer (الـ driver اللي محتاج الـ clock)

#### Building Blocks المشابهة:
- **`drivers/clk/clk-divider.c`** — نفس الفكرة بس قسمة بدل ضرب
- **`drivers/clk/clk-fixed-factor.c`** — multiplier + divider بنسبة ثابتة في الكود (مش في register)
- **`drivers/clk/clk-gate.c`** — تشغيل/إيقاف الـ clock
- **`drivers/clk/clk-mux.c`** — اختيار الـ parent clock
- **`drivers/clk/clk-composite.c`** — بيجمع mux + divider + gate في clock واحدة

#### Hardware Drivers اللي بتستخدم الـ multiplier:
- أي `drivers/clk/<vendor>/` — كل vendor بيعمل driver خاص بيه بيستخدم الـ building blocks دي
## Phase 2: شرح الـ Common Clock Framework (CCF) وـ clk-multiplier

---

### المشكلة اللي الـ Subsystem بيحلها

في أي SoC (System-on-Chip) حديث — زي Allwinner H3 أو STM32MP1 — في عشرات الـ clocks:
- Clock للـ CPU
- Clock للـ USB
- Clock للـ SDMMC
- Clock للـ UART
- إلخ...

كل clock ممكن يتعمل فيه:
- **Enable / Disable** — تشغيل أو إيقاف
- **Rate Change** — تغيير التردد
- **Parent Selection** — اختيار مصدره (PLL1 ولا PLL2 ولا OSC)

قبل الـ CCF (Common Clock Framework)، كل platform كانت بتكتب clock driver منفصل تماماً. المشكلة:

| المشكلة | التأثير |
|---|---|
| Code duplication ضخم | كل vendor بيعيد اختراع العجلة |
| لا يوجد tree topology موحد | صعب تتبع dependencies |
| لا يوجد reference counting | Clock ممكن يتوقف وfيه driver شغال عليه |
| لا توجد واجهة موحدة للـ consumers | كل driver بيكلم hardware مباشرة |

---

### الحل — Common Clock Framework

الـ CCF اتقدم في kernel 3.4 وحل المشكلة بثلاث أفكار أساسية:

1. **Unified consumer API** — أي driver بيستخدم `clk_get()`, `clk_enable()`, `clk_set_rate()` بغض النظر عن الـ hardware.
2. **Clock tree** — الـ clocks منظمة في شجرة بـ parent/child relationships. تغيير parent rate بيتشال تلقائياً للـ children.
3. **Provider abstraction** — كل نوع clock (gate, divider, mux, multiplier) بيوفر `struct clk_ops` وبس، والـ framework بيتولى الباقي.

---

### التشبيه الحقيقي — شركة كهرباء

تخيل شبكة كهرباء في مدينة كبيرة:

```
[محطة توليد رئيسية] ← هذه هي الـ Crystal Oscillator (24 MHz fixed)
        |
   [محطة تحويل PLL] ← ترفع التردد (مثلاً x40 = 960 MHz)
        |
  ┌─────┴──────┐
[حي A]      [حي B]   ← هذه هي الـ Clock Domains
  |              |
[بيت 1]     [مصنع]   ← هذه هي الـ Peripheral Drivers
```

| عنصر الكهرباء | مقابله في الـ CCF |
|---|---|
| محطة توليد (fixed) | `clk_fixed_rate` — crystal oscillator |
| محطة تحويل (ترفع الجهد بمعامل) | `clk_multiplier` — موضوع ملفنا |
| محول كهربائي (يقسم الجهد) | `clk_divider` |
| مفتاح كهرباء (يقطع/يوصل) | `clk_gate` |
| أسلاك التوزيع (تختار مصدر) | `clk_mux` |
| عداد الكهرباء (reference count) | `clk_core.enable_count` + `prepare_count` |
| شركة الكهرباء (المدير) | `drivers/clk/clk.c` — قلب الـ CCF |
| المستهلك (الأجهزة المنزلية) | أي driver يعمل `clk_get()` |

الجزء الأعمق في التشبيه:
- لما بيت 1 يطلب زيادة جهد → شركة الكهرباء بتتفاوض مع محطة التحويل → وهي ممكن تطلب من محطة التوليد تغيير خامها. هذا هو بالضبط `CLK_SET_RATE_PARENT` في الـ CCF.
- لما تفصل كهرباء حي A، حي B ما يتأثرش. هذا هو الـ enable reference counting.
- شركة الكهرباء مش بتهتم إذا المفتاح ميكانيكي أو رقمي — بس تعطيه أمر قطع/وصل. هذا هو مبدأ الـ `clk_ops` abstraction.

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Consumer Drivers                       │
│  (USB driver, MMC driver, UART driver, GPU driver ...)   │
│                                                         │
│   clk_get()  clk_enable()  clk_set_rate()  clk_put()   │
└───────────────────────┬─────────────────────────────────┘
                        │  Consumer API (include/linux/clk.h)
                        ▼
┌─────────────────────────────────────────────────────────┐
│              Common Clock Framework Core                 │
│              (drivers/clk/clk.c)                        │
│                                                         │
│  • Clock tree management (parent/child)                 │
│  • Reference counting (prepare_count, enable_count)     │
│  • Rate negotiation & propagation                       │
│  • Debugfs (/sys/kernel/debug/clk/)                     │
│  • Notifier chains                                      │
└──────┬──────────────┬──────────────┬────────────────────┘
       │              │              │
       ▼              ▼              ▼
  clk_gate       clk_divider   clk_multiplier   clk_mux
  (gate.c)       (divider.c)   (clk-multiplier.c) (mux.c)
       │              │              │              │
       └──────────────┴──────────────┴──────────────┘
                        │
                        │  struct clk_ops (callbacks)
                        ▼
┌─────────────────────────────────────────────────────────┐
│           Platform / SoC Clock Drivers                  │
│                                                         │
│   drivers/clk/sunxi-ng/    (Allwinner)                  │
│   drivers/clk/samsung/     (Exynos)                     │
│   drivers/clk/stm32/       (STMicro)                    │
│   drivers/clk/imx/         (NXP i.MX)                  │
│                                                         │
│   كل منهم يستخدم clk_multiplier_ops أو يكتب ops خاصة  │
└──────────────────────────────┬──────────────────────────┘
                               │
                               ▼
                      ┌────────────────┐
                      │  Real Hardware │
                      │  (SoC Registers│
                      │   via MMIO)    │
                      └────────────────┘
```

---

### الـ Core Abstraction — ثلاثي الـ CCF

الـ CCF يقوم على ثلاث structs محورية. لازم تفهمهم كويس عشان تفهم الـ multiplier:

#### 1. `struct clk_hw` — نقطة الربط

```c
struct clk_hw {
    struct clk_core *core;   /* pointer للـ framework-managed state */
    struct clk      *clk;    /* per-user handle (للـ consumers) */
    const struct clk_init_data *init; /* metadata عند التسجيل */
};
```

الـ `clk_hw` دائماً موجود كـ **أول member** في أي clock struct خاصة (زي `clk_multiplier`). ده بيخلي الـ `container_of()` macro يشتغل بكفاءة:

```c
/* من include/linux/clk-provider.h */
#define to_clk_multiplier(_hw) \
    container_of(_hw, struct clk_multiplier, hw)
```

يعني الـ framework يعدي `clk_hw *` للـ callback، والـ callback يحوله لـ `clk_multiplier *` في سطر واحد.

#### 2. `struct clk_ops` — عقد التنفيذ

```c
struct clk_ops {
    /* Life cycle */
    int   (*prepare)(struct clk_hw *);
    void  (*unprepare)(struct clk_hw *);
    int   (*enable)(struct clk_hw *);
    void  (*disable)(struct clk_hw *);

    /* Rate management */
    unsigned long (*recalc_rate)(struct clk_hw *, unsigned long parent_rate);
    int           (*determine_rate)(struct clk_hw *, struct clk_rate_request *);
    int           (*set_rate)(struct clk_hw *, unsigned long rate,
                              unsigned long parent_rate);

    /* Parent management */
    int  (*set_parent)(struct clk_hw *, u8 index);
    u8   (*get_parent)(struct clk_hw *);
    /* ... + phase, duty cycle, debug ... */
};
```

الـ `clk_multiplier_ops` في ملفنا بتنفذ 3 callbacks بس:

```c
const struct clk_ops clk_multiplier_ops = {
    .recalc_rate    = clk_multiplier_recalc_rate,
    .determine_rate = clk_multiplier_determine_rate,
    .set_rate       = clk_multiplier_set_rate,
};
```

#### 3. `struct clk_multiplier` — الـ Hardware Descriptor

```c
struct clk_multiplier {
    struct clk_hw   hw;       /* MUST be first — enables container_of */
    void __iomem   *reg;      /* عنوان الـ register في MMIO space */
    u8              shift;    /* بداية الـ bit field داخل الـ register */
    u8              width;    /* عدد الـ bits */
    u8              flags;    /* CLK_MULTIPLIER_* flags */
    spinlock_t     *lock;     /* حماية read-modify-write من الـ IRQ */
};
```

مثال حقيقي: في Allwinner H3، الـ PLL Audio clock له register بيبدأ من bit 8 وعرضه 5 bits للـ multiplier factor:

```
Register 0x01C20008:
 31      16 15    13 12     8  7       0
┌──────────┬────────┬────────┬─────────┐
│ reserved │  post  │ factor │  p bits │
│          │  div   │  N (5b)│         │
└──────────┴────────┴────────┴─────────┘
                     ↑
               shift=8, width=5
               maxmult = (1<<5)-1 = 31
               output = parent_rate * N
```

---

### رحلة `recalc_rate` — كيف يقرأ الـ Framework التردد الحالي

```
Consumer calls: clk_get_rate(clk)
        │
        ▼
CCF core: clk_recalc_rates(core)
        │
        ▼
يستدعي: core->ops->recalc_rate(hw, parent_rate)
        │
        ▼
clk_multiplier_recalc_rate(hw, parent_rate):
```

```c
static unsigned long clk_multiplier_recalc_rate(struct clk_hw *hw,
                                                 unsigned long parent_rate)
{
    struct clk_multiplier *mult = to_clk_multiplier(hw);
    unsigned long val;

    /* قراءة الـ register (big endian أو little endian) */
    val = clk_mult_readl(mult) >> mult->shift;

    /* عزل الـ bit field بالـ mask */
    val &= GENMASK(mult->width - 1, 0);

    /* لو الـ multiplier = 0 والـ ZERO_BYPASS مفعّل → اعتبره bypass (×1) */
    if (!val && mult->flags & CLK_MULTIPLIER_ZERO_BYPASS)
        val = 1;

    return parent_rate * val;  /* التردد الحقيقي */
}
```

مثال: `parent_rate = 24 MHz`, register يحتوي `N=40` في bits [12:8]:
- `val = (reg >> 8) & 0x1F = 40`
- `return 24_000_000 * 40 = 960_000_000` → 960 MHz

---

### رحلة `determine_rate` — التفاوض على أفضل تردد

هذه أعقد خطوة في الـ multiplier. الـ consumer يطلب تردداً معيناً، والـ framework بيسأل:
"ما أقرب تردد فعلي تقدر تحققه؟ وما أفضل parent_rate محتاجه؟"

```c
static int clk_multiplier_determine_rate(struct clk_hw *hw,
                                          struct clk_rate_request *req)
{
    struct clk_multiplier *mult = to_clk_multiplier(hw);

    /* __bestmult يجرب كل قيم الـ multiplier من 1 لـ maxmult */
    unsigned long factor = __bestmult(hw, req->rate,
                                      &req->best_parent_rate,
                                      mult->width, mult->flags);

    /* الناتج = أفضل parent_rate × أفضل multiplier */
    req->rate = req->best_parent_rate * factor;
    return 0;
}
```

**الـ `__bestmult` بتشتغل بسيناريوهين:**

**السيناريو 1: `CLK_SET_RATE_PARENT` مش مفعّل**
الـ parent_rate ثابت، نحسب المضاعف المناسب مباشرة:
```c
bestmult = rate / orig_parent_rate;
/* نتحقق: لا تجاوز الـ maxmult، لا نصل للـ zero */
```

**السيناريو 2: `CLK_SET_RATE_PARENT` مفعّل**
الـ framework يقدر يطلب من الـ parent يغير تردده! هنا بتحصل خوارزمية بحث:

```
for i from 1 to maxmult:
    1. لو rate == parent_rate × i بالظبط → perfect match, return i
    2. اسأل parent: لو طلبت منك (rate / i) تقدر تعطيني ايه؟
       parent_rate = clk_hw_round_rate(parent, rate / i)
    3. احسب: current_rate = parent_rate × i
    4. لو current_rate أفضل من best_rate → خزّن i و parent_rate
```

```c
for (i = 1; i < maxmult; i++) {
    /* السيناريو المثالي: لا نغير الـ parent */
    if (rate == orig_parent_rate * i) {
        *best_parent_rate = orig_parent_rate;
        return i;
    }

    /* نسأل الـ parent عن أقرب rate لـ (rate/i) */
    parent_rate = clk_hw_round_rate(clk_hw_get_parent(hw), rate / i);
    current_rate = parent_rate * i;

    if (__is_best_rate(rate, current_rate, best_rate, flags)) {
        bestmult = i;
        best_rate = current_rate;
        *best_parent_rate = parent_rate;
    }
}
```

الـ `__is_best_rate` بتحدد "الأفضل" حسب الـ flags:
- **بدون `ROUND_CLOSEST`**: أفضل rate = أقرب rate من الأسفل بدون تجاوز الهدف
- **مع `ROUND_CLOSEST`**: أفضل rate = أقل فرق مطلق `abs(rate - new)`

```c
static bool __is_best_rate(unsigned long rate, unsigned long new,
                             unsigned long best, unsigned long flags)
{
    if (flags & CLK_MULTIPLIER_ROUND_CLOSEST)
        return abs(rate - new) < abs(rate - best);  /* أقل خطأ */

    return new >= rate && new < best;   /* أكبر من الهدف وأصغر من best */
}
```

---

### رحلة `set_rate` — الكتابة على الـ Register

```c
static int clk_multiplier_set_rate(struct clk_hw *hw, unsigned long rate,
                                    unsigned long parent_rate)
{
    struct clk_multiplier *mult = to_clk_multiplier(hw);
    unsigned long factor = __get_mult(mult, rate, parent_rate);
    unsigned long flags = 0;
    unsigned long val;

    /* اقفل الـ spinlock لحماية read-modify-write من الـ IRQ */
    if (mult->lock)
        spin_lock_irqsave(mult->lock, flags);
    else
        __acquire(mult->lock);  /* للـ static analysis فقط */

    val = clk_mult_readl(mult);

    /* امسح الـ bits الخاصة بالـ multiplier فقط */
    val &= ~GENMASK(mult->width + mult->shift - 1, mult->shift);

    /* اكتب القيمة الجديدة في مكانها */
    val |= factor << mult->shift;
    clk_mult_writel(mult, val);

    if (mult->lock)
        spin_unlock_irqrestore(mult->lock, flags);
    else
        __release(mult->lock);

    return 0;
}
```

مثال على الـ bit manipulation:
```
width=5, shift=8, factor=40 (0x28)

GENMASK(5+8-1, 8) = GENMASK(12, 8) = 0x1F00

قبل: reg = 0xAB_FF_00 (قيمة عشوائية)
بعد المسح: reg & ~0x1F00 = 0xAB_E0_00
بعد الكتابة: 0xAB_E0_00 | (40 << 8) = 0xAB_E0_00 | 0x2800 = 0xAB_E8_00
```

---

### العلاقة بين الـ Structs — مخطط شامل

```
Consumer Driver
    │
    │  clk_set_rate(clk, 960_000_000)
    ▼
┌──────────────────────────────────────────┐
│  struct clk  (per-user handle)           │
│  └─→ struct clk_core                    │
│       ├─ rate: 960_000_000              │
│       ├─ parent: → clk_core (PLL_in)    │
│       ├─ ops: → clk_multiplier_ops      │
│       └─ hw: → clk_hw ─────────────┐   │
└─────────────────────────────────────│───┘
                                      │  container_of
                                      ▼
                          ┌────────────────────────┐
                          │  struct clk_multiplier  │
                          │  ├─ hw  ← (أول member) │
                          │  ├─ reg: 0xF000_1008   │
                          │  ├─ shift: 8            │
                          │  ├─ width: 5            │
                          │  ├─ flags: ROUND_CLOSEST│
                          │  └─ lock: &pll_lock     │
                          └──────────┬──────────────┘
                                     │  ioread32 / iowrite32
                                     ▼
                          ┌────────────────────────┐
                          │  Hardware Register      │
                          │  0xF000_1008            │
                          │  bits[12:8] = N factor  │
                          └────────────────────────┘
```

---

### ماذا يمتلك الـ CCF وماذا يُفوّض للـ Driver

| المسؤولية | الـ CCF Core | الـ clk_multiplier Driver |
|---|---|---|
| Clock tree traversal | نعم — يتتبع parent/child | لا |
| Reference counting | نعم — `enable_count`, `prepare_count` | لا |
| Rate caching | نعم — `clk_core.rate` | لا |
| Notifier chains | نعم — يبلّغ consumers بالتغيير | لا |
| Debugfs entries | نعم — `/sys/kernel/debug/clk/` | لا |
| Register access | لا | **نعم — `readl/writel` مباشرة** |
| Bit field manipulation | لا | **نعم — shift/width/mask** |
| Endianness handling | لا | **نعم — `CLK_MULTIPLIER_BIG_ENDIAN`** |
| Rate calculation | لا | **نعم — `recalc_rate`** |
| Best rate negotiation | لا | **نعم — `determine_rate` + `__bestmult`** |
| Locking strategy | لا (يوفر prepare_lock) | **نعم — spinlock للـ IRQ context** |

---

### نقطة مهمة: الـ `__acquire` و `__release`

```c
if (mult->lock)
    spin_lock_irqsave(mult->lock, flags);
else
    __acquire(mult->lock);  /* ← مش lock حقيقي! */
```

الـ `__acquire()` و `__release()` هم **annotations للـ sparse** (أداة static analysis للـ kernel)، مش locks حقيقية. لما مفيش lock مطلوب (الـ driver تأكد إن الوصول آمن)، بيستخدمهم عشان sparse ما يشكيش من "unbalanced locking".

---

### مثال تطبيقي — Allwinner H3 PLL1

```c
/* من drivers/clk/sunxi-ng/ccu-sun8i-h3.c (مبسّط) */
static struct clk_multiplier pll_cpu_mult = {
    .hw.init = CLK_HW_INIT("pll-cpu",
                            "osc24M",          /* parent: 24 MHz crystal */
                            &clk_multiplier_ops,
                            0),
    .reg   = base + 0x000,    /* PLL_CPUX_CTRL_REG */
    .shift = 8,
    .width = 5,
    .lock  = &pll_lock,
};

/*
 * لما kernel تطلب 1008 MHz للـ CPU:
 * determine_rate: best factor = 42, best_parent_rate = 24 MHz
 *                 actual_rate = 24 × 42 = 1008 MHz
 * set_rate: اكتب 42 في bits[12:8] من PLL_CPUX_CTRL_REG
 */
```

هذا بالضبط ما بيحصل عند boot لما الـ kernel تضبط تردد الـ CPU.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags & Enums — Cheatsheet

#### الـ `clk_multiplier` flags (حقل `flags` في `struct clk_multiplier`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_MULTIPLIER_ZERO_BYPASS` | `BIT(0)` | لو قيمة الـ multiplier في الـ register صفر، تعامل معاها كـ bypass (رجّع parent rate بدون تغيير بدل ما تضرب في صفر) |
| `CLK_MULTIPLIER_ROUND_CLOSEST` | `BIT(1)` | اختار أقرب multiplier integer للقيمة المطلوبة (بدل الـ floor) |
| `CLK_MULTIPLIER_BIG_ENDIAN` | `BIT(2)` | استخدم `ioread32be`/`iowrite32be` بدل `readl`/`writel` عشان الـ hardware big-endian |

#### الـ CLK framework flags (مُستخدمة من `clk_hw_get_flags`)

| Flag | المعنى في السياق ده |
|------|---------------------|
| `CLK_SET_RATE_PARENT` | لو موجودة، الـ `__bestmult` ممكن يطلب تغيير rate الـ parent نفسه عشان يوصل للـ rate المطلوب |

---

### الـ Structs الأساسية

#### 1. `struct clk_multiplier`

**الغرض:** الـ struct الرئيسي اللي بيمثل clock بيضرب frequency الـ parent في multiplier قابل للتعديل عبر register في الـ hardware.

```c
struct clk_multiplier {
    struct clk_hw  hw;       /* embedded handle — ربط بالـ CCF */
    void __iomem  *reg;      /* عنوان الـ MMIO register اللي فيه الـ multiplier */
    u8             shift;    /* ابتداءً من أنهي bit داخل الـ register */
    u8             width;    /* عدد الـ bits الخاصة بالـ multiplier field */
    u8             flags;    /* CLK_MULTIPLIER_* flags */
    spinlock_t    *lock;     /* pointer لـ spinlock خارجي (ممكن NULL) */
};
```

| الحقل | النوع | الدور |
|-------|-------|-------|
| `hw` | `struct clk_hw` | الـ embedded handle — نقطة الاتصال مع الـ Common Clock Framework |
| `reg` | `void __iomem *` | الـ MMIO register اللي بيحتوي على الـ multiplier field |
| `shift` | `u8` | عدد الـ bits اللي نشيل منها الـ multiplier field داخل الـ register |
| `width` | `u8` | حجم الـ multiplier field بالـ bits — الـ max value = `(1 << width) - 1` |
| `flags` | `u8` | bitfield من `CLK_MULTIPLIER_*` flags |
| `lock` | `spinlock_t *` | لو مش NULL بيُستخدم في `set_rate` لحماية الـ read-modify-write |

**الـ macro للوصول:**
```c
#define to_clk_multiplier(_hw) container_of(_hw, struct clk_multiplier, hw)
```
بيرجع `struct clk_multiplier *` من `struct clk_hw *` عبر pointer arithmetic.

---

#### 2. `struct clk_hw`

**الغرض:** الـ handle العام اللي بيربط أي hardware-specific clock struct بالـ Common Clock Framework (CCF).

```c
struct clk_hw {
    struct clk_core          *core;  /* pointer للـ CCF internal state */
    struct clk               *clk;   /* per-user clk handle */
    const struct clk_init_data *init; /* init data — NULL بعد التسجيل */
};
```

| الحقل | الدور |
|-------|-------|
| `core` | الـ CCF بيملأه بعد `clk_hw_register` — مش بيتلمسه الـ driver |
| `clk` | per-user handle — الـ CCF بيديه للـ consumers |
| `init` | بيتحدد قبل التسجيل، بيشمل الاسم والـ ops والـ parents |

---

#### 3. `struct clk_ops`

**الغرض:** جدول الـ function pointers اللي الـ CCF بيستدعيه على الـ clock. الـ `clk_multiplier` بيملأ 3 منهم فقط.

```c
const struct clk_ops clk_multiplier_ops = {
    .recalc_rate    = clk_multiplier_recalc_rate,   /* اقرأ register، احسب الـ rate الحالي */
    .determine_rate = clk_multiplier_determine_rate, /* ابحث عن أحسن multiplier لـ rate مطلوب */
    .set_rate       = clk_multiplier_set_rate,       /* اكتب الـ multiplier في الـ register */
};
```

الـ callbacks التانية (`prepare`, `enable`, `set_parent`, إلخ) مش موجودة هنا — الـ CCF بيتعامل معاها بـ defaults.

---

#### 4. `struct clk_rate_request`

**الغرض:** بيُمرَّر لـ `determine_rate` وبيحمل الـ constraints والـ result.

```c
struct clk_rate_request {
    struct clk_core *core;            /* الـ clock نفسه */
    unsigned long    rate;            /* الـ rate المطلوب (input) / الـ rate المتحقق (output) */
    unsigned long    min_rate;        /* الحد الأدنى المسموح */
    unsigned long    max_rate;        /* الحد الأقصى المسموح */
    unsigned long    best_parent_rate;/* أحسن rate للـ parent (input/output) */
    struct clk_hw   *best_parent_hw;  /* أحسن parent clock */
};
```

---

### علاقات الـ Structs — ASCII Diagram

```
  ┌──────────────────────────────────────────────────────┐
  │              struct clk_multiplier                   │
  │                                                      │
  │  ┌─────────────────┐   void __iomem *reg ────────────┼──► MMIO Register
  │  │  struct clk_hw  │                                 │    [31      shift+width-1 .. shift .. 0]
  │  │  ─────────────  │   u8 shift                      │         └─── multiplier field ───┘
  │  │  *core ─────────┼──► struct clk_core (CCF)        │
  │  │  *clk  ─────────┼──► struct clk (consumer handle) │   spinlock_t *lock ─────────────────► external spinlock
  │  │  *init ─────────┼──► struct clk_init_data         │                                        (أو NULL)
  │  │                 │        └─► *ops ────────────────┼──► clk_multiplier_ops
  │  └─────────────────┘            └─► name, parents   │
  └──────────────────────────────────────────────────────┘

  clk_multiplier_ops:
  ┌──────────────────────────┐
  │ .recalc_rate             │──► clk_multiplier_recalc_rate()
  │ .determine_rate          │──► clk_multiplier_determine_rate()
  │ .set_rate                │──► clk_multiplier_set_rate()
  └──────────────────────────┘
```

---

### Lifecycle Diagram — من التسجيل للاستخدام

```
  [Driver / Platform Code]
          │
          │  kzalloc(sizeof(*mult))  أو static allocation
          ▼
  ┌─────────────────────────────────────┐
  │  mult->reg   = ioremap(...)         │
  │  mult->shift = X                    │  ← إعداد الـ struct
  │  mult->width = Y                    │
  │  mult->flags = CLK_MULTIPLIER_*    │
  │  mult->lock  = &some_spinlock       │
  │  mult->hw.init = &clk_init_data     │
  │    .ops     = &clk_multiplier_ops  │
  │    .name    = "my-mult-clk"         │
  │    .parents = [...]                 │
  └─────────────────────────────────────┘
          │
          │  clk_hw_register(dev, &mult->hw)
          ▼
  ┌─────────────────────────────────────┐
  │  CCF allocates struct clk_core      │
  │  CCF fills mult->hw.core            │  ← بعد التسجيل
  │  mult->hw.init = NULL               │
  │  Clock visible to system            │
  └─────────────────────────────────────┘
          │
          │  Consumer: clk_set_rate(clk, rate)
          ▼
  ┌─────────────────────────────────────┐
  │  CCF calls .determine_rate()        │  ← ابحث عن أحسن multiplier
  │  CCF calls .set_rate()              │  ← اكتب في الـ register
  └─────────────────────────────────────┘
          │
          │  clk_hw_unregister(&mult->hw)
          ▼
  ┌─────────────────────────────────────┐
  │  CCF frees struct clk_core          │  ← تنظيف
  │  Driver frees mult (kfree)          │
  └─────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### `recalc_rate` — حساب الـ rate الحالي

```
  CCF needs current rate
    └─► clk_multiplier_recalc_rate(hw, parent_rate)
          │
          ├─► to_clk_multiplier(hw)          — recover struct clk_multiplier*
          │
          ├─► clk_mult_readl(mult)            — قرأ الـ register
          │       ├─ [BIG_ENDIAN?] ioread32be(mult->reg)
          │       └─ [else]        readl(mult->reg)
          │
          ├─► val >>= mult->shift             — shift down
          ├─► val &= GENMASK(width-1, 0)     — mask الـ field
          │
          ├─► [val==0 && ZERO_BYPASS?] val = 1   — bypass: treat 0 as 1×
          │
          └─► return parent_rate * val        — الـ rate الفعلي
```

#### `determine_rate` — إيجاد أحسن multiplier

```
  CCF: clk_set_rate(clk, target_rate)
    └─► clk_multiplier_determine_rate(hw, req)
          │
          ├─► to_clk_multiplier(hw)
          │
          └─► __bestmult(hw, req->rate, &req->best_parent_rate, width, flags)
                │
                ├─[CLK_SET_RATE_PARENT NOT set]──────────────────────────────┐
                │   bestmult = rate / orig_parent_rate                        │
                │   clamp to [1..maxmult]  (ZERO_BYPASS guard)               │
                │   return bestmult  ◄───────────────────────────────────────┘
                │
                └─[CLK_SET_RATE_PARENT set] loop i = 1..maxmult-1
                      │
                      ├─[rate == orig_parent_rate * i] perfect match → return i
                      │
                      └─ clk_hw_round_rate(parent, rate/i) → parent_rate
                             current_rate = parent_rate * i
                             __is_best_rate(rate, current_rate, best_rate, flags)?
                               ├─[ROUND_CLOSEST] abs(rate-new) < abs(rate-best)
                               └─[else]          new >= rate && new < best
                             update bestmult, best_rate, best_parent_rate
          │
          req->rate = req->best_parent_rate * factor
          return 0
```

#### `set_rate` — كتابة الـ multiplier في الـ register

```
  CCF: apply new rate
    └─► clk_multiplier_set_rate(hw, rate, parent_rate)
          │
          ├─► to_clk_multiplier(hw)
          │
          ├─► __get_mult(mult, rate, parent_rate)
          │       ├─[ROUND_CLOSEST] DIV_ROUND_CLOSEST(rate, parent_rate)
          │       └─[else]          rate / parent_rate
          │
          ├─► [mult->lock != NULL] spin_lock_irqsave(lock, flags)
          │   [mult->lock == NULL] __acquire(lock)   — sparse annotation only
          │
          ├─► clk_mult_readl(mult)                   — read current register value
          ├─► val &= ~GENMASK(width+shift-1, shift) — clear the multiplier field
          ├─► val |= factor << shift                 — set new multiplier
          ├─► clk_mult_writel(mult, val)             — write back
          │
          └─► [mult->lock != NULL] spin_unlock_irqrestore(lock, flags)
              [mult->lock == NULL] __release(lock)

          return 0
```

---

### Locking Strategy

#### ما اللي بيحتاج lock وما اللي ما بيحتاجش

| Operation | Lock مطلوب؟ | السبب |
|-----------|-------------|-------|
| `recalc_rate` | لا | read-only — الـ CCF نفسه بيحمي بـ `prepare_mutex` |
| `determine_rate` | لا | حسابات بس، ما فيش write للـ hardware |
| `set_rate` | نعم (`spinlock`) | read-modify-write على الـ MMIO register — لازم atomic |

#### الـ Locking في `set_rate` بالتفصيل

```
  [CPU A: set_rate]                    [CPU B: set_rate]
  spin_lock_irqsave(&lock, flags)
    readl(reg)      ← critical section
    modify val
    writel(reg, val)
  spin_unlock_irqrestore(&lock, flags)
                                       spin_lock_irqsave(&lock, flags)
                                         readl(reg)   ← safe now
                                         ...
                                       spin_unlock_irqrestore(...)
```

**نقاط مهمة:**
- الـ `spinlock_t *lock` ده **pointer لـ lock خارجي** — مش الـ `clk_multiplier` نفسه اللي بيمتلكه. ده بيسمح لكذا clock يشاركوا نفس الـ lock لو بيتحكموا في نفس الـ register.
- لو `mult->lock == NULL` يبقى الـ driver بيقول "أنا ضامن التزامن بطريقة تانية" — الـ `__acquire`/`__release` مجرد annotations لـ sparse (الـ static analysis tool) ومش بتعمل حاجة real في runtime.
- الـ `irqsave` variant مهمة عشان `set_rate` ممكن يتنادى من contexts مختلفة.

#### Lock Ordering

الملف ده ما عندوش lock ordering معقد — في حين إن الـ CCF نفسه عنده ترتيب:

```
  prepare_mutex  (sleepable, يحمي prepare/unprepare)
      └─► enable_lock (spinlock, يحمي enable/disable)
              └─► mult->lock (spinlock, يحمي register write في set_rate)
```

الـ `mult->lock` دايمًا بيتاخد من جوا `set_rate` اللي بيتنادى من CCF core اللي هو بالفعل شايل الـ locks فوقيه — فمفيش خطر deadlock طالما الـ driver ما بياخدش نفس الـ lock في interrupt handler بصورة عكسية.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Functions الكاملة في الملف

| Function | النوع | الغرض |
|---|---|---|
| `clk_mult_readl` | Helper (static inline) | قراءة الـ register بـ endianness صح |
| `clk_mult_writel` | Helper (static inline) | كتابة الـ register بـ endianness صح |
| `__get_mult` | Helper (static) | حساب الـ multiplier من rate/parent_rate |
| `__is_best_rate` | Helper (static) | مقارنة rate جديد بـ best حالي |
| `__bestmult` | Helper (static) | البحث عن أفضل multiplier مع/بدون تغيير الـ parent |
| `clk_multiplier_recalc_rate` | clk_ops callback | إعادة حساب الـ rate الحالي من الـ hardware |
| `clk_multiplier_determine_rate` | clk_ops callback | تحديد أقرب rate ممكن قبل الضبط |
| `clk_multiplier_set_rate` | clk_ops callback | كتابة الـ multiplier في الـ register |

#### الـ Structs والـ Macros الأساسية

| Symbol | المصدر | الدور |
|---|---|---|
| `struct clk_multiplier` | `include/linux/clk-provider.h` | الـ main struct للـ driver |
| `to_clk_multiplier(_hw)` | `include/linux/clk-provider.h` | `container_of` wrapper للـ `clk_hw` |
| `CLK_MULTIPLIER_ZERO_BYPASS` | `BIT(0)` | معاملة الـ zero multiplier كـ bypass |
| `CLK_MULTIPLIER_ROUND_CLOSEST` | `BIT(1)` | تقريب للأقرب بدل التقريب للأسفل |
| `CLK_MULTIPLIER_BIG_ENDIAN` | `BIT(2)` | استخدام Big Endian I/O |
| `clk_multiplier_ops` | exported | الـ `clk_ops` struct الجاهز للتسجيل |

---

### المجموعة الأولى: الـ I/O Helpers

الغرض من المجموعة دي إنها تعزل كل كود الـ register access في مكان واحد وتعالج مشكلة الـ endianness بشكل شفاف. أي حاجة تقرأ أو تكتب الـ hardware register بتمر من هنا، وبكده لو فيه SoC بـ Big Endian bus (زي بعض MIPS أو PowerPC platforms) بيشتغل من غير ما يغير أي منطق تاني.

---

#### `clk_mult_readl`

```c
static inline u32 clk_mult_readl(struct clk_multiplier *mult)
```

**الـ function دي** بتقرأ الـ 32-bit register اللي فيها الـ multiplier field. لو الـ flag `CLK_MULTIPLIER_BIG_ENDIAN` متسته، بتستخدم `ioread32be` عشان تعمل byte-swap تلقائي؛ غير كده بتستخدم `readl` العادي اللي بيعمل Little Endian access على ARM وغيره.

**Parameters:**
- `mult` — مؤشر لـ `struct clk_multiplier`، منه بنجيب `mult->reg` (الـ MMIO address) و`mult->flags`.

**Return:**
- `u32` — قيمة الـ register كاملة (raw، قبل masking أو shifting).

**Key details:**
- `static inline` → الـ compiler بيـ inline الـ function ومفيش function call overhead.
- الـ `ioread32be` / `readl` بيضمنوا memory barrier صح على الـ architectures اللي تحتاجه.
- الـ caller دايماً هو `clk_multiplier_recalc_rate` أو `clk_multiplier_set_rate`.

---

#### `clk_mult_writel`

```c
static inline void clk_mult_writel(struct clk_multiplier *mult, u32 val)
```

**الـ function دي** بتكتب قيمة الـ 32-bit لـ register. نفس منطق الـ endianness: `iowrite32be` لو Big Endian، `writel` لو غيره.

**Parameters:**
- `mult` — الـ `struct clk_multiplier`.
- `val` — القيمة الكاملة المراد كتابتها في الـ register (بعد ما بيتعمل read-modify-write في الـ caller).

**Return:** void.

**Key details:**
- الـ caller (`clk_multiplier_set_rate`) بيعمل read-modify-write كامل قبل ما يكال الـ function دي.
- لازم الـ spinlock يكون مسكوك وقت الكتابة (الـ caller مسؤول عن ده).

---

### المجموعة التانية: الـ Rate Calculation Helpers

الـ helpers دول بيعملوا الـ math وراء حساب الـ multiplier. الـ design فصل منطق الـ "إيه أحسن multiplier؟" عن منطق الـ "اكتب في الـ hardware" عشان يسهل الـ testing والـ reuse.

---

#### `__get_mult`

```c
static unsigned long __get_mult(struct clk_multiplier *mult,
                                unsigned long rate,
                                unsigned long parent_rate)
```

**الـ function دي** بتحسب قيمة الـ multiplier المطلوبة لتحقيق الـ `rate` المطلوب بناءً على `parent_rate`. لو الـ flag `CLK_MULTIPLIER_ROUND_CLOSEST` متسته، بتستخدم `DIV_ROUND_CLOSEST` عشان يقرب للأقرب؛ غير كده بتعمل integer division عادي (تقريب للأسفل).

**Parameters:**
- `mult` — الـ struct عشان نعرف الـ flags.
- `rate` — الـ target output rate المطلوب.
- `parent_rate` — الـ rate الحالي للـ parent clock.

**Return:**
- `unsigned long` — قيمة الـ multiplier (N) بحيث `output ≈ parent_rate * N`.

**Key details:**
- الـ function دي بتُستخدم فقط في `clk_multiplier_set_rate`، يعني بعد ما الـ framework قرر الـ final rate.
- مفيش bounds checking هنا، الـ caller (`clk_multiplier_set_rate`) هو اللي بيكتب القيمة في الـ register والـ hardware بيكون bounded بـ `width` bits.

---

#### `__is_best_rate`

```c
static bool __is_best_rate(unsigned long rate, unsigned long new,
                           unsigned long best, unsigned long flags)
```

**الـ function دي** بتقارن `new` (rate محسوب جديد) بـ `best` (أحسن rate لقيناه لحد دلوقتي) وبتقول هل `new` أحسن. المنطق بيتغير على حسب الـ flags: لو `CLK_MULTIPLIER_ROUND_CLOSEST` → الأقرب لـ `rate` المطلوب هو الأحسن؛ غير كده → الأحسن هو الأكبر من أو يساوي `rate` والأصغر من `best` (يعني floor بدون تجاوز).

**Parameters:**
- `rate` — الـ target rate المطلوب.
- `new` — الـ rate الجديد المراد تقييمه.
- `best` — أحسن rate لقيناه لحد دلوقتي.
- `flags` — الـ `clk_multiplier->flags`.

**Return:**
- `true` لو `new` أحسن من `best`، `false` غير كده.

**Key details:**
- دي pure helper، مفيش side effects.
- الـ `abs()` هنا هو macro الـ kernel اللي بيتعامل مع `unsigned long` difference صح.
- بيُستخدم حصرياً جوه `__bestmult`.

---

#### `__bestmult`

```c
static unsigned long __bestmult(struct clk_hw *hw, unsigned long rate,
                                unsigned long *best_parent_rate,
                                u8 width, unsigned long flags)
```

**دي أهم function في الملف** من ناحية التعقيد. مهمتها البحث عن أفضل قيمة multiplier (من 1 لـ `maxmult`) اللي تحقق الـ target rate، مع إمكانية تغيير الـ parent rate (لو `CLK_SET_RATE_PARENT` متسته في الـ clk flags).

**Parameters:**
- `hw` — الـ `clk_hw` handle لتفعيل `clk_hw_get_flags` و`clk_hw_round_rate`.
- `rate` — الـ target output rate.
- `best_parent_rate` — مؤشر لـ parent rate: دخل = الـ parent rate الحالي، خرج = أحسن parent rate لقيناه.
- `width` — عدد bits الـ multiplier field (يحدد `maxmult = (1 << width) - 1`).
- `flags` — الـ `clk_multiplier->flags` (للـ rounding strategy).

**Return:**
- `unsigned long` — قيمة الـ multiplier الأمثل.

**Key details:**
- لو `CLK_SET_RATE_PARENT` مش متسته: مفيش iteration، بيحسب مباشرة `rate / orig_parent_rate` مع clamping.
- لو `CLK_SET_RATE_PARENT` متسته: بيعمل loop من 1 لـ `maxmult - 1` ولكل `i` بيسأل الـ parent عن أحسن rate عنده لـ `rate / i` عن طريق `clk_hw_round_rate`، وبيختار الـ combination الأفضل.
- الـ early exit: لو لقى exact match بدون تغيير الـ parent، بيرجع فوراً.
- الـ locking: مفيش locking هنا، الـ caller (CCF framework) بيمسك `prepare_mutex` وقت الـ `determine_rate`.

**Pseudocode Flow:**

```
__bestmult(hw, rate, best_parent_rate, width, flags):
    maxmult = (1 << width) - 1
    orig_parent_rate = *best_parent_rate

    if NOT CLK_SET_RATE_PARENT in clk_hw_get_flags(hw):
        bestmult = rate / orig_parent_rate
        clamp bestmult to [1, maxmult]  // avoid 0 and overflow
        return bestmult

    // CLK_SET_RATE_PARENT case: search all multipliers
    best_rate = ULONG_MAX
    for i in [1, maxmult):
        if rate == orig_parent_rate * i:
            *best_parent_rate = orig_parent_rate
            return i  // perfect match, no parent change needed

        parent_rate = clk_hw_round_rate(parent, rate / i)
        current_rate = parent_rate * i

        if __is_best_rate(rate, current_rate, best_rate, flags):
            bestmult = i
            best_rate = current_rate
            *best_parent_rate = parent_rate

    return bestmult
```

---

### المجموعة التالتة: الـ clk_ops Callbacks

الـ callbacks دول هي الـ interface الرسمي بين الـ driver والـ **Common Clock Framework (CCF)**. بيتسجلوا في `clk_multiplier_ops` struct.

---

#### `clk_multiplier_recalc_rate`

```c
static unsigned long clk_multiplier_recalc_rate(struct clk_hw *hw,
                                                unsigned long parent_rate)
```

**الـ function دي** بتقرأ الـ multiplier الحالي من الـ hardware وبتحسب الـ output rate الفعلي. بتمسك الـ register، بتعمل shift وmask عشان تاخد الـ field بالضبط، وبعدين بتضرب في الـ parent rate.

**Parameters:**
- `hw` — الـ `clk_hw`، بيتحوله لـ `clk_multiplier` عن طريق `to_clk_multiplier`.
- `parent_rate` — الـ rate الحالي للـ parent clock (بيجي من الـ CCF).

**Return:**
- `unsigned long` — الـ output rate الحالي الفعلي = `parent_rate * multiplier_value`.

**Key details:**
- لو `val == 0` و`CLK_MULTIPLIER_ZERO_BYPASS` متسته: الـ val بيتعامل كـ 1 (bypass)، يعني الـ output = parent_rate.
- لو `val == 0` وبدون الـ flag: الـ output rate بيكون 0 (clock stopped effectively).
- مفيش locking: الـ CCF بيضمن إن الـ `prepare_mutex` مسكوك عند الـ call.
- الـ GENMASK(mult->width - 1, 0) بيعمل mask صح لأي width من 1 لـ 8 bits.
- **الـ caller:** الـ CCF framework في `clk_recalc_rates()` و`__clk_recalc_rates()`.

---

#### `clk_multiplier_determine_rate`

```c
static int clk_multiplier_determine_rate(struct clk_hw *hw,
                                         struct clk_rate_request *req)
```

**الـ function دي** هي الـ `determine_rate` callback، بتحسب أقرب rate ممكن يحققه الـ clock بدون ما تلمس الـ hardware. بتستدعي `__bestmult` عشان تلاقي أحسن multiplier وبتكتب النتيجة في الـ `req` struct.

**Parameters:**
- `hw` — الـ `clk_hw` handle.
- `req` — مؤشر لـ `struct clk_rate_request` (الـ CCF بيملاه):
  - `req->rate` — الـ target rate المطلوب (input/output).
  - `req->best_parent_rate` — الـ parent rate المقترح (input/output).

**Return:**
- `0` دايماً (الـ function مش بترفض requests).

**Key details:**
- الـ function بتعدل `req->rate` و`req->best_parent_rate` in-place.
- مفيش I/O هنا خالص، pure calculation.
- الـ CCF بيستخدم النتيجة في `clk_core_determine_round_nolock` ثم بيقرر هل يكال `set_rate` أم لا.
- **الـ caller:** CCF في سياق `clk_set_rate()` أو `clk_round_rate()`.

---

#### `clk_multiplier_set_rate`

```c
static int clk_multiplier_set_rate(struct clk_hw *hw, unsigned long rate,
                                   unsigned long parent_rate)
```

**الـ function دي** هي اللي بتلمس الـ hardware فعلياً. بتحسب الـ multiplier المطلوب، بتمسك الـ spinlock لو موجود، وبتعمل atomic read-modify-write على الـ register.

**Parameters:**
- `hw` — الـ `clk_hw` handle.
- `rate` — الـ target rate المطلوب (اللي الـ CCF قرره بعد `determine_rate`).
- `parent_rate` — الـ parent rate الفعلي بعد أي تغيير.

**Return:**
- `0` دايماً.

**Key details:**

**الـ Locking:**
- لو `mult->lock` مش NULL: بيستخدم `spin_lock_irqsave` / `spin_unlock_irqrestore` → آمن من interrupt context.
- لو `mult->lock` هو NULL: بيستخدم `__acquire` / `__release` (sparse annotations فقط، مفيش locking حقيقي) → الـ caller مسؤول عن الحماية.

**الـ Read-Modify-Write:**
```c
val = clk_mult_readl(mult);                            // قرأ الـ register
val &= ~GENMASK(mult->width + mult->shift - 1,         // امسح bits الـ field
                mult->shift);
val |= factor << mult->shift;                          // ضع القيمة الجديدة
clk_mult_writel(mult, val);                            // اكتب
```

- الـ mask بيحافظ على باقي bits الـ register (shared register pattern).
- **الـ caller:** CCF في `clk_core_set_rate_nolock` بعد ما يتأكد من الـ rate صح.

**Pseudocode Flow:**

```
clk_multiplier_set_rate(hw, rate, parent_rate):
    factor = __get_mult(mult, rate, parent_rate)

    if mult->lock:
        spin_lock_irqsave(mult->lock, flags)

    val = clk_mult_readl(mult)
    // clear the multiplier field
    val &= ~GENMASK(width + shift - 1, shift)
    // set new multiplier
    val |= factor << shift
    clk_mult_writel(mult, val)

    if mult->lock:
        spin_unlock_irqrestore(mult->lock, flags)

    return 0
```

---

### المجموعة الرابعة: الـ Exported Ops Table

#### `clk_multiplier_ops`

```c
const struct clk_ops clk_multiplier_ops = {
    .recalc_rate    = clk_multiplier_recalc_rate,
    .determine_rate = clk_multiplier_determine_rate,
    .set_rate       = clk_multiplier_set_rate,
};
EXPORT_SYMBOL_GPL(clk_multiplier_ops);
```

الـ struct ده هو الـ public API الوحيد في الملف. الـ driver code في أي SoC بيعمل `struct clk_hw_init_data` ويحط `&clk_multiplier_ops` فيه. الـ `EXPORT_SYMBOL_GPL` معناه إن الـ struct ده متاح للـ modules بس بشرط GPL license.

**الـ ops المُعرَّفة:**
| Op | الـ Callback |
|---|---|
| `.recalc_rate` | `clk_multiplier_recalc_rate` |
| `.determine_rate` | `clk_multiplier_determine_rate` |
| `.set_rate` | `clk_multiplier_set_rate` |

**الـ ops الغائبة ومقصودة:**
| Op | السبب |
|---|---|
| `.enable` / `.disable` | الـ multiplier clock مش gate |
| `.set_parent` / `.get_parent` | single parent، الـ CCF بيتعامل معاه |
| `.round_rate` | الـ `determine_rate` بيغنى عنه في الـ CCF الجديد |

---

### ملاحظة: تدفق البيانات الكامل

```
clk_set_rate(clk, target_rate)
        │
        ▼
clk_multiplier_determine_rate(hw, req)
   └── __bestmult(hw, rate, &parent_rate, width, flags)
           ├── [no CLK_SET_RATE_PARENT] → rate / parent_rate
           └── [CLK_SET_RATE_PARENT]   → loop + clk_hw_round_rate(parent, rate/i)
        │
        ▼ (CCF يحدد parent rate ويطبقه)
        │
clk_multiplier_set_rate(hw, rate, parent_rate)
   ├── __get_mult(mult, rate, parent_rate) → factor
   ├── spin_lock_irqsave (if lock)
   ├── read-modify-write register
   └── spin_unlock_irqrestore (if lock)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — Entries وإزاي تقراها

**الـ** Common Clock Framework (CCF) بيعمل entries تلقائي في `/sys/kernel/debug/clk/` لكل clock مسجل.

```bash
# اعرف اسم الـ clock الـ multiplier من debugfs
ls /sys/kernel/debug/clk/

# اقرأ الـ rate الفعلية (بتستدعي recalc_rate داخليا)
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_rate

# اقرأ عدد الـ prepare/enable references
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_prepare_count
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_enable_count

# اعرف الـ parent الحالي
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_parent

# اقرأ الـ flags المضبوطة على الـ clock
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_flags

# اعرض شجرة الـ clocks كاملة مع الـ rates
cat /sys/kernel/debug/clk/clk_summary

# عرض مفصل — يظهر كل clock وأبناؤه والـ rate والـ enable count
cat /sys/kernel/debug/clk/clk_dump
```

**تفسير `clk_summary`:**

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
---------------------------------------------------------------------------------------------
 osc24M                             2       2        0    24000000          0     0  50000
   mult_clk                         1       1        0   240000000          0     0  50000
```

**الـ** `mult_clk` rate = `parent_rate * multiplier_val_from_reg`. لو الـ rate صفر → multiplier register قيمته 0 ومافيش `CLK_MULTIPLIER_ZERO_BYPASS`.

---

#### 2. sysfs — Entries ذات الصلة

```bash
# لو الـ clock مربوط بـ device، الـ sysfs بيظهر فيه
ls /sys/bus/platform/devices/<dev_name>/

# الـ clock consumers اللي بيستخدموا الـ multiplier
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_prepare_count

# اقرأ الـ rate من الـ consumer device (لو مسجلة)
# مثلا لو في subsystem بيعرض clk_rate كـ sysfs attribute
cat /sys/class/<subsystem>/<dev>/clk_rate
```

---

#### 3. ftrace — Tracepoints والـ Events

**الـ** CCF بيوفر tracepoints جاهزة في `drivers/clk/clk.c`.

```bash
# فعّل الـ tracing للـ clock framework
cd /sys/kernel/debug/tracing

# اعرف الـ events المتاحة للـ clk
grep -r 'clk' available_events

# فعّل كل events الـ clk
echo 1 > events/clk/enable

# أو بشكل انتقائي:
echo 1 > events/clk/clk_set_rate/enable
echo 1 > events/clk/clk_set_rate_complete/enable
echo 1 > events/clk/clk_enable/enable
echo 1 > events/clk/clk_disable/enable
echo 1 > events/clk/clk_prepare/enable

# ابدأ الـ trace
echo 1 > tracing_on

# اعمل العملية اللي بتشتبه فيها (set rate مثلا)
echo 0 > tracing_on

# اقرأ النتيجة
cat trace
```

**مثال على output:**

```
   systemd-1      [000]  1234.567890: clk_set_rate: mult_clk 240000000
   systemd-1      [000]  1234.567891: clk_set_rate_complete: mult_clk 240000000
```

لو شفت `clk_set_rate` بدون `clk_set_rate_complete` → في deadlock محتمل أو خطأ في `set_rate`.

```bash
# trace function calls داخل الـ clk multiplier نفسه
echo clk_multiplier_set_rate > set_ftrace_filter
echo clk_multiplier_recalc_rate >> set_ftrace_filter
echo clk_multiplier_determine_rate >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
```

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug للـ clk subsystem (لو في pr_debug فيه)
echo 'file drivers/clk/clk-multiplier.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل debug messages في مجلد الـ clk
echo 'file drivers/clk/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إن الـ activation اتعمل
cat /sys/kernel/debug/dynamic_debug/control | grep clk-multiplier

# اعرض الـ kernel log في real-time
dmesg -w | grep -i 'clk\|mult'

# زوّد الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk
```

**الـ** `clk-multiplier.c` مافيهوش `pr_debug` calls حالياً — لكن لو أضفت `pr_debug` للـ `clk_multiplier_set_rate` يظهر هنا.

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_COMMON_CLK` | تفعيل CCF الأساسي — لازم يكون مفعل |
| `CONFIG_CLK_DEBUG` | تفعيل debugfs entries للـ clocks |
| `CONFIG_DEBUG_FS` | لازم لأي debugfs access |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `+p` في dynamic_debug |
| `CONFIG_PROVE_LOCKING` | يكتشف lockdep violations في `mult->lock` |
| `CONFIG_DEBUG_SPINLOCK` | يتتبع spinlock usage — مهم للـ `spin_lock_irqsave` في `set_rate` |
| `CONFIG_LOCKUP_DETECTOR` | يكتشف الـ deadlocks المحتملة |
| `CONFIG_KALLSYMS` | يعرض أسماء الـ functions في stack traces |
| `CONFIG_FRAME_POINTER` | stack traces أدق |
| `CONFIG_DEBUG_KERNEL` | يفعّل مجموعة debug options دفعة واحدة |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_CLK_DEBUG|CONFIG_COMMON_CLK|CONFIG_PROVE_LOCK'
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# clk_summary — أهم أداة
cat /sys/kernel/debug/clk/clk_summary

# اعرف كل الـ clocks اللي rate = 0 (مشكلة محتملة)
awk '$6 == 0 {print $1, $6}' /sys/kernel/debug/clk/clk_summary

# استخدم clk_dump لـ JSON-like output
cat /sys/kernel/debug/clk/clk_dump

# أداة userspace: clk-tools (لو موجودة في distro)
# أو اكتب script بسيط:
for clk in /sys/kernel/debug/clk/*/; do
    name=$(basename $clk)
    rate=$(cat $clk/clk_rate 2>/dev/null)
    echo "$name: $rate Hz"
done
```

**الـ** `devlink` مش ذات صلة مباشرة بالـ clk-multiplier. الـ subsystem-specific tool هو `clk_summary` وقراءة الـ register مباشرة.

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `clk: couldn't get parent clock` | الـ parent clock مش متاح عند التسجيل | تأكد إن الـ parent registered قبل الـ multiplier |
| `clk: failed to reparent` | فشل تغيير الـ parent | تحقق من `CLK_SET_RATE_PARENT` flag |
| `clk: rate 0 on <mult_clk>` | multiplier=0 ومافيش `ZERO_BYPASS` | اضبط الـ register أو فعّل `CLK_MULTIPLIER_ZERO_BYPASS` |
| `BUG: spinlock bad magic` | corruption في `mult->lock` | تأكد إن `spinlock_t` initialized بـ `spin_lock_init` |
| `WARNING: CPU: X PID: Y ... enable_lock` | `set_rate` اتنادى من atomic context غلط | راجع الـ caller — `set_rate` لازم من sleepable context |
| `clk: <mult_clk> already disabled` | enable/disable mismatch | راجع الـ consumers وتأكد من balanced enable/disable |
| `clk: <mult_clk> No parents available` | parent لم يُسجل | راجع ترتيب `clk_register` في الـ driver |
| `ioread32be: invalid address` | الـ `reg` pointer خاطئ أو unmapped | تحقق من `ioremap` ومن الـ BIG_ENDIAN flag |

---

#### 8. نقاط استراتيجية لـ dump_stack() / WARN_ON()

```c
static int clk_multiplier_set_rate(struct clk_hw *hw, unsigned long rate,
                                   unsigned long parent_rate)
{
    struct clk_multiplier *mult = to_clk_multiplier(hw);
    unsigned long factor = __get_mult(mult, rate, parent_rate);

    /* WARN لو الـ factor طلع صفر وMZERO_BYPASS مش مفعل */
    WARN_ON(factor == 0 && !(mult->flags & CLK_MULTIPLIER_ZERO_BYPASS));

    /* WARN لو الـ rate المطلوب أكبر من اللي يقدر يحققه الـ register */
    WARN_ON(factor > ((1 << mult->width) - 1));

    /* WARN لو الـ reg pointer فيه مشكلة */
    WARN_ON(!mult->reg);

    if (mult->lock)
        spin_lock_irqsave(mult->lock, flags);
    else
        __acquire(mult->lock);

    val = clk_mult_readl(mult);
    val &= ~GENMASK(mult->width + mult->shift - 1, mult->shift);
    val |= factor << mult->shift;
    clk_mult_writel(mult, val);

    /* تحقق بعد الكتابة — readback للتحقق من الـ hardware */
    WARN_ON((clk_mult_readl(mult) >> mult->shift &
             GENMASK(mult->width - 1, 0)) != factor);

    if (mult->lock)
        spin_unlock_irqrestore(mult->lock, flags);
    else
        __release(mult->lock);

    return 0;
}
```

نقاط `dump_stack()` المفيدة:
- عند اكتشاف `factor == 0` في production system
- عند read-back mismatch بعد الكتابة للـ register
- في `clk_multiplier_recalc_rate` لو `parent_rate == 0`

---

### Hardware Level

#### 1. مطابقة حالة الـ Hardware مع حالة الـ Kernel

الـ `clk_multiplier` بيقرأ/يكتب register واحد بـ bit field محدد بـ `shift` و `width`.

```bash
# اقرأ الـ rate اللي الـ kernel يعتقده
cat /sys/kernel/debug/clk/<mult_clk_name>/clk_rate

# احسب المضاعف يدوياً:
# rate = parent_rate * multiplier
# multiplier = rate / parent_rate

# مثال: لو parent = 24MHz والـ rate = 240MHz → multiplier = 10

# اقرأ قيمة الـ parent rate
cat /sys/kernel/debug/clk/<parent_clk_name>/clk_rate
```

---

#### 2. Register Dump — قراءة الـ Register مباشرة

```bash
# devmem2 — يقرأ register بعنوان فيزيائي مباشر
# مثال: لو الـ reg عنده physical address 0x01C20000

# قراءة 32-bit register
devmem2 0x01C20000 w

# أو باستخدام /dev/mem مع python
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x01C20000 & ~0xFFF)
    offset = 0x01C20000 & 0xFFF
    val = struct.unpack('<I', m[offset:offset+4])[0]
    print(f'Register value: 0x{val:08X}')
    # استخرج الـ multiplier field: مثلا shift=0, width=4
    shift, width = 0, 4
    mask = (1 << width) - 1
    mult_val = (val >> shift) & mask
    print(f'Multiplier field: {mult_val}')
    m.close()
"

# أو بـ io utility (من package i2c-tools أو busybox)
io -4 -r 0x01C20000
```

**مطابقة القيمة:**

```
Register raw: 0x0000000A  (= 10 decimal)
shift=0, width=4 → multiplier = (0xA >> 0) & 0xF = 10
parent_rate = 24,000,000 Hz
expected_rate = 24,000,000 * 10 = 240,000,000 Hz

kernel reported rate: 240000000 Hz ✓
```

لو في فرق → إما الـ `shift`/`width` غلط في `struct clk_multiplier`، أو الـ endianness مش مضبوط (`CLK_MULTIPLIER_BIG_ENDIAN`).

---

#### 3. Logic Analyzer / Oscilloscope

```
الهدف: التحقق إن الـ clock output فعلياً تغير بعد set_rate

نقاط القياس:
┌─────────────────────────────────────────────────────────┐
│  Oscilloscope Setup:                                     │
│                                                          │
│  CH1 → Clock output pin (الـ signal بعد الـ multiplier)  │
│  CH2 → Reference clock (parent clock pin)               │
│                                                          │
│  Expected: CH1 freq = N × CH2 freq                      │
│  N = multiplier value                                    │
└─────────────────────────────────────────────────────────┘

نصائح:
- استخدم x10 probe لتقليل الـ capacitive loading على signal lines
- الـ trigger على CH2 (parent) لتثبيت الصورة
- قس الفرق بين CH1 وCH2 بـ FFT mode للتأكد من الـ harmonics
- لو في jitter كبير → مشكلة في الـ PLL أو الـ power supply noise

Logic Analyzer:
- اربط على الـ register bus (لو SoC بيدعم JTAG/ETM)
- استخدم OpenOCD لـ live register reading بدون إيقاف الـ CPU
```

```bash
# OpenOCD — قراءة register live
openocd -f board/my_board.cfg &
telnet localhost 4444
# في telnet:
# mdw phys 0x01C20000     (memory display word, physical address)
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | ما يظهر في dmesg | التحقق |
|---|---|---|
| Power supply للـ clock domain منخفض | rate غلط، clock unstable، system hangs عشوائية | قس VDD بـ multimeter أثناء load |
| Register access لـ wrong address | `Unhandled fault: external abort` أو `BUG: unable to handle kernel paging` | تحقق من physical address في DT مقارنة بـ datasheet |
| Endianness خاطئ (`BIG_ENDIAN` flag) | rate يظهر قيمة غريبة جداً مثلاً `0xA000000` بدل `0xA` | اقرأ الـ register بـ devmem2 وقارن |
| Clock glitch عند set_rate | kernel panics أو data corruption في consumers | أضف `CLK_SET_RATE_GATE` flag |
| Multiplier overflow (`factor > maxmult`) | rate أقل من المطلوب بكثير | تحقق من `width` في struct وقارن بـ datasheet |
| Lock contention في IRQ context | `BUG: scheduling while atomic` | تأكد إن `mult->lock` مش NULL |

---

#### 5. Device Tree Debugging

**الـ** `clk_multiplier` بياخد `reg`, `shift`, `width`, `flags` من الـ driver مباشرة — مش من DT في الغالب — لكن الـ parent يُحدد من DT.

```bash
# تحقق من الـ DT المحمل فعلياً (compiled)
cat /sys/firmware/devicetree/base/__symbols__/<node_name>

# اقرأ الـ clock node من DT
fdtdump /sys/firmware/fdt | grep -A 20 'multiplier'

# أو باستخدام dtc
dtc -I fs /sys/firmware/devicetree/base/ -O dts 2>/dev/null | grep -A 30 'clk-mult'

# تحقق من الـ clock-names و clocks properties في consumer devices
cat /proc/device-tree/<consumer-node>/clocks | xxd
cat /proc/device-tree/<consumer-node>/clock-names

# تحقق من الـ parent clock binding
ls /sys/kernel/debug/clk/<parent_clock>/
cat /sys/kernel/debug/clk/<parent_clock>/clk_rate
```

**أشياء تتحقق منها في DT مقارنة بـ Datasheet:**

```
✓ physical register address صح ؟
✓ الـ clock-output-names يطابق ما يتوقعه الـ consumer ؟
✓ الـ parent clock صح في clocks = <&parent_clk> ؟
✓ الـ shift وwidth بتتطابق مع bit fields في الـ datasheet ؟
✓ CLK_MULTIPLIER_BIG_ENDIAN محتاجها SoC ده ؟
```

---

### Practical Commands

#### سيناريو 1: التحقق من قيمة الـ Multiplier الفعلية

```bash
#!/bin/bash
# تحقق شامل من حالة clk_multiplier

CLK_NAME="my_mult_clk"  # غير الاسم حسب نظامك
CLK_PATH="/sys/kernel/debug/clk/$CLK_NAME"

echo "=== Clock Multiplier Debug Report ==="
echo "Clock Name: $CLK_NAME"
echo ""

if [ -d "$CLK_PATH" ]; then
    echo "Rate:          $(cat $CLK_PATH/clk_rate) Hz"
    echo "Prepare count: $(cat $CLK_PATH/clk_prepare_count)"
    echo "Enable count:  $(cat $CLK_PATH/clk_enable_count)"
    echo "Flags:         $(cat $CLK_PATH/clk_flags)"
    echo "Parent:        $(cat $CLK_PATH/clk_parent 2>/dev/null || echo 'N/A')"

    PARENT=$(cat $CLK_PATH/clk_parent 2>/dev/null)
    if [ -n "$PARENT" ] && [ -d "/sys/kernel/debug/clk/$PARENT" ]; then
        PARENT_RATE=$(cat /sys/kernel/debug/clk/$PARENT/clk_rate)
        RATE=$(cat $CLK_PATH/clk_rate)
        if [ "$PARENT_RATE" -gt 0 ]; then
            MULT=$(( RATE / PARENT_RATE ))
            echo "Calculated multiplier: $MULT (rate/parent_rate)"
        fi
    fi
else
    echo "ERROR: Clock '$CLK_NAME' not found in debugfs!"
    echo "Available clocks:"
    ls /sys/kernel/debug/clk/ | head -20
fi
```

#### سيناريو 2: تفعيل ftrace لمراقبة set_rate

```bash
#!/bin/bash
# تفعيل tracing لكل عمليات الـ clk
cd /sys/kernel/debug/tracing

# reset
echo nop > current_tracer
echo 0 > tracing_on
echo > trace

# فعّل events
echo 1 > events/clk/clk_set_rate/enable
echo 1 > events/clk/clk_set_rate_complete/enable
echo 1 > events/clk/clk_enable/enable
echo 1 > events/clk/clk_disable/enable

# ابدأ
echo 1 > tracing_on
echo "Tracing started. Press Enter to stop..."
read

echo 0 > tracing_on

# اعرض النتيجة مفلترة على multiplier clock
cat trace | grep -E 'mult|clk_set_rate'
```

**مثال output وتفسيره:**

```
# task-pid [cpu] timestamp: event: details
   bash-1234  [000] 12345.678: clk_set_rate: my_mult_clk 240000000
   bash-1234  [000] 12345.679: clk_set_rate_complete: my_mult_clk 240000000
```

لو شفت `clk_set_rate` بدون `clk_set_rate_complete` في نفس الـ PID → خطأ في `set_rate` أو deadlock.

#### سيناريو 3: مقارنة Register Value مع kernel state

```bash
#!/bin/bash
# مثال على SoC بـ multiplier register على 0x01C20000
# shift=0, width=4

PHYS_ADDR=0x01C20000
SHIFT=0
WIDTH=4
CLK_NAME="my_mult_clk"

KERNEL_RATE=$(cat /sys/kernel/debug/clk/$CLK_NAME/clk_rate)
PARENT_RATE=$(cat /sys/kernel/debug/clk/$(cat /sys/kernel/debug/clk/$CLK_NAME/clk_parent)/clk_rate)

# اقرأ الـ register
RAW_VAL=$(devmem2 $PHYS_ADDR w 2>/dev/null | grep "Read" | awk '{print $NF}')
MASK=$(( (1 << WIDTH) - 1 ))
HW_MULT=$(( (RAW_VAL >> SHIFT) & MASK ))
HW_RATE=$(( PARENT_RATE * HW_MULT ))

echo "Kernel reported rate: $KERNEL_RATE Hz"
echo "HW register value:    0x$(printf '%08X' $RAW_VAL)"
echo "HW multiplier field:  $HW_MULT"
echo "HW calculated rate:   $HW_RATE Hz"

if [ "$KERNEL_RATE" -eq "$HW_RATE" ]; then
    echo "STATUS: OK — Kernel state matches hardware"
else
    echo "STATUS: MISMATCH — possible stale cache or wrong shift/width"
fi
```

#### سيناريو 4: dynamic debug activation

```bash
# فعّل debug messages للـ clk multiplier
echo 'file drivers/clk/clk-multiplier.c +pflmt' \
    > /sys/kernel/debug/dynamic_debug/control

# +p = print, +f = function name, +l = line number, +m = module name, +t = thread id

# تحقق
grep 'clk-multiplier' /sys/kernel/debug/dynamic_debug/control

# اراقب
dmesg -wH | grep clk
```

#### سيناريو 5: اكتشاف lock issues

```bash
# لو مشغل kernel بـ CONFIG_PROVE_LOCKING
dmesg | grep -E 'DEADLOCK|lock|WARNING.*lock'

# اعرض lockdep stats
cat /proc/lockdep_stats

# لو الـ system بيتجمد عند set_rate، اعمل sysrq
echo t > /proc/sysrq-trigger   # dump all threads
dmesg | grep -A 10 'mult_clk'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ UART بيشتغل ببطء غير متوقع

#### العنوان
**تردد UART خاطئ بسبب multiplier قيمته صفر**

#### السياق
شركة بتبني industrial gateway على RK3562، الـ gateway بيتوصل بـ Modbus RTU devices عن طريق RS-485 (UART). الـ baud rate المطلوب 115200. بعد boot الجهاز، الـ UART شغال لكن بيحصل framing errors كتير، والـ communication بتفشل بشكل متقطع.

#### المشكلة
الـ UART controller بيشتغل بـ baud rate مش 115200 بالظبط. الـ engineer لاحظ إن الـ clock source للـ UART بيدي rate غلط عن طريق `cat /sys/kernel/debug/clk/clk_summary`.

```
uart_mclk        0    0    0   0          0        0    0  50000000
```

الـ expected rate كانت `100000000` (100 MHz) والمفروض الـ UART divider يقسمها لـ 115200. لكن الـ rate طلع 50 MHz بس.

#### التحليل
الـ clock tree عند الـ RK3562:

```
pll_gpll (1200 MHz)
    └── gpll_div2 (600 MHz)  [clk-divider]
            └── uart_mult    [clk-multiplier]  <-- المشكلة هنا
                    └── uart_mclk
```

الـ `uart_mult` بيستخدم `clk_multiplier_ops`. لما الـ kernel بيبدأ وبيقرأ الـ multiplier من الـ register:

```c
// clk_multiplier_recalc_rate
val = clk_mult_readl(mult) >> mult->shift;
val &= GENMASK(mult->width - 1, 0);

if (!val && mult->flags & CLK_MULTIPLIER_ZERO_BYPASS)
    val = 1;

return parent_rate * val;
```

الـ register value بعد reset = `0`. لو `CLK_MULTIPLIER_ZERO_BYPASS` مش set، الـ function بترجع `parent_rate * 0 = 0`، وده بيخلي الـ CCF framework يستخدم آخر cached value أو يحسب بناءً على قيمة غلط.

لو `CLK_MULTIPLIER_ZERO_BYPASS` **set**، الـ `val` بيتحول لـ 1، فالـ rate بتبقى = `parent_rate * 1`. لو الـ `parent_rate` هو 50 MHz، النتيجة صح بالنسبالها، لكن الـ expected كانت ضرب في 2 لتطلع 100 MHz.

الـ DT node للـ multiplier:

```dts
// خطأ — مش محدد ضرب في 2 بعد reset
uart_mult: uart-mult {
    compatible = "fixed-factor-clock";
    clocks = <&gpll_div2>;
    clock-mult = <1>;  // <-- ده المشكلة
    clock-div = <1>;
};
```

أو لو استخدموا `clk_multiplier` بدل fixed-factor، الـ hardware register لازم يتكتب فيه قيمة `2` أثناء الـ initialization ومش بيحصل.

#### الحل

**خطوة 1**: تأكيد المشكلة:

```bash
# قراءة الـ multiplier register مباشرة
devmem2 0xFD7C0080 w   # عنوان CRU register للـ uart_mult على RK3562

# أو عن طريق debugfs
cat /sys/kernel/debug/clk/uart_mult/clk_rate
cat /sys/kernel/debug/clk/uart_mult/clk_parent_rate
```

**خطوة 2**: تعديل الـ driver initialization لكتابة قيمة صح:

```c
// في driver initialization أو في .init callback
static int uart_mult_init(struct clk_hw *hw)
{
    struct clk_multiplier *mult = to_clk_multiplier(hw);
    unsigned long val;

    val = clk_mult_readl(mult);
    val &= ~GENMASK(mult->width + mult->shift - 1, mult->shift);
    val |= (2 << mult->shift);  /* set multiplier = 2 */
    clk_mult_writel(mult, val);

    return 0;
}
```

**خطوة 3**: أو تصحيح الـ DT:

```dts
uart_mult: uart-mult@fd7c0080 {
    compatible = "rockchip,rk3562-clk-multiplier";
    reg = <0x0 0xfd7c0080 0x0 0x4>;
    clocks = <&gpll_div2>;
    #clock-cells = <0>;
    rockchip,shift = <0>;
    rockchip,width = <3>;
    /* لازم تضمن إن الـ hardware يتكتب فيه 2 عند boot */
};
```

#### الدرس المستفاد
لما الـ `clk_multiplier` register قيمته صفر بعد reset، لازم تتأكد من:
1. إن `CLK_MULTIPLIER_ZERO_BYPASS` set لو الـ zero معناه bypass مش zero output.
2. إن الـ bootloader أو الـ `.init` callback بيكتب القيمة الصح في الـ register قبل ما الـ kernel يقرأها.
3. مراجعة `clk_summary` دايمًا بعد boot على أي board جديد.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش بتطلع صورة بـ 4K

#### العنوان
**الـ HDMI pixel clock بيتحسب غلط بسبب `CLK_SET_RATE_PARENT` مش مضبوط**

#### السياق
manufacturer بيبني Android TV box على Allwinner H616. الجهاز المفروض يدعم 4K@30fps عن طريق HDMI. بعد تشغيل Android، الـ 1080p شغال تمام، لكن لما بيحاول يشغل 4K الشاشة بتتقطع أو مش بتشتغل خالص.

#### المشكلة
الـ pixel clock للـ HDMI عند 4K@30fps المفروض يكون ~297 MHz. الـ clock tree:

```
pll_video0 (variable)
    └── hdmi_pll_mult  [clk-multiplier, width=4]
            └── tcon_tv_clk
                    └── hdmi pixel clock
```

الـ engineer شاف:

```bash
cat /sys/kernel/debug/clk/hdmi_pll_mult/clk_rate
# Output: 148500000  (148.5 MHz بدل 297 MHz)
```

#### التحليل
في `__bestmult`:

```c
if (!(clk_hw_get_flags(hw) & CLK_SET_RATE_PARENT)) {
    bestmult = rate / orig_parent_rate;
    // لو parent_rate = 148.5 MHz و rate = 297 MHz
    // bestmult = 297000000 / 148500000 = 2   ✓

    if (bestmult > maxmult)
        bestmult = maxmult;

    return bestmult;  // بيرجع 2، بس بدون تعديل الـ parent
}
```

المشكلة إن `CLK_SET_RATE_PARENT` **مش set** في الـ clock flags. يعني الـ `__bestmult` بتحسب المضاعف بناءً على الـ parent rate الحالية فقط. لو الـ `pll_video0` مش على الـ rate الصح (148.5 MHz)، الناتج غلط.

للـ 4K، الـ pixel clock = 297 MHz. الـ `pll_video0` لازم يكون على 148.5 MHz، والـ multiplier = 2. لكن لو الـ `pll_video0` كان على 148 MHz (قيمة قريبة بس مش دقيقة)، النتيجة:

```
297 / 148 = 2 (integer division)
actual rate = 148 * 2 = 296 MHz
```

فرق 1 MHz كافي يخلي الـ HDMI sink يرفض الـ signal.

لو `CLK_SET_RATE_PARENT` كان set، الـ loop في `__bestmult` كان هيجرب:

```c
for (i = 1; i < maxmult; i++) {
    if (rate == orig_parent_rate * i) {
        *best_parent_rate = orig_parent_rate;
        return i;  // perfect match
    }

    parent_rate = clk_hw_round_rate(clk_hw_get_parent(hw), rate / i);
    // لـ i=2: rate/2 = 148.5 MHz
    // parent يقدر يعمل 148.5 MHz بالظبط؟ لو آه، perfect match
    current_rate = parent_rate * i;

    if (__is_best_rate(rate, current_rate, best_rate, flags)) {
        bestmult = i;
        best_rate = current_rate;
        *best_parent_rate = parent_rate;  // يطلب من parent 148.5 MHz
    }
}
```

#### الحل

**تعديل الـ clock registration** في الـ Allwinner H616 clock driver:

```c
// في drivers/clk/sunxi-ng/ccu-h616.c (مثال)
static struct ccu_mult hdmi_pll_mult_clk = {
    .mult   = _SUNXI_CCU_MULT(0, 4),
    .common = {
        .reg        = 0x0510,
        .hw.init    = CLK_HW_INIT("hdmi-pll-mult",
                                   "pll-video0",
                                   &ccu_mult_ops,
                                   CLK_SET_RATE_PARENT),  /* مهم جداً */
    },
};
```

**تحقق بعد الإصلاح**:

```bash
# اطلب 297 MHz
echo 297000000 > /sys/kernel/debug/clk/hdmi_pll_mult/clk_rate

# تحقق
cat /sys/kernel/debug/clk/hdmi_pll_mult/clk_rate
# Expected: 297000000

cat /sys/kernel/debug/clk/pll_video0/clk_rate
# Expected: 148500000
```

#### الدرس المستفاد
الـ `CLK_SET_RATE_PARENT` flag مش optional لو الـ clock multiplier محتاج يطلب من parent rate معينة. بدونه، الـ `__bestmult` بتحسب فقط بناءً على الـ parent rate الحالية، وأي خطأ في الـ parent بيتضاعف في الـ output. دايمًا فكر: "هل الـ multiplier لازم يغير الـ parent؟"

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — الـ SPI data corruption عند high speed

#### العنوان
**تحريف بيانات SPI بسبب `CLK_MULTIPLIER_ROUND_CLOSEST` بتعمل overshoot في الـ clock**

#### السياق
شركة بتبني IoT environmental sensor board على STM32MP1. الـ board بتقرأ high-speed ADC بيانات عن طريق SPI بـ 50 MHz. الـ ADC (ADS8688) عنده maximum SPI clock = 50 MHz بالظبط. بعد bring-up، القراءات جاية غلط أحيانًا، وبيبان فيها bit errors.

#### المشكلة
الـ engineer قاس الـ SPI clock على oscilloscope: 51.2 MHz — أكبر من الـ maximum للـ ADC.

```bash
cat /sys/kernel/debug/clk/spi2_k/clk_rate
# Output: 51200000
```

#### التحليل
الـ SPI kernel driver طلب `clk_set_rate(spi_clk, 50000000)`. الـ clock tree:

```
hse_ck (24 MHz)
    └── pll3_r (512 MHz)  [PLL]
            └── spi_mult  [clk-multiplier, CLK_MULTIPLIER_ROUND_CLOSEST, width=4]
                    └── spi2_k
```

عند `clk_multiplier_determine_rate`:

```c
unsigned long factor = __bestmult(hw, req->rate, &req->best_parent_rate,
                                   mult->width, mult->flags);
req->rate = req->best_parent_rate * factor;
```

في `__bestmult`، لما `CLK_SET_RATE_PARENT` set:

```c
for (i = 1; i < maxmult; i++) {
    parent_rate = clk_hw_round_rate(clk_hw_get_parent(hw), rate / i);
    // rate = 50 MHz, i=1: parent_rate = round(50 MHz) → maybe 51.2 MHz
    current_rate = parent_rate * i;  // = 51.2 MHz
    // ...
}
```

في `__is_best_rate` مع `CLK_MULTIPLIER_ROUND_CLOSEST`:

```c
if (flags & CLK_MULTIPLIER_ROUND_CLOSEST)
    return abs(rate - new) < abs(rate - best);
    // abs(50M - 51.2M) = 1.2M  vs  abs(50M - 50M) = 0
    // بيختار الأقرب، لكن لو مفيش option بالظبط 50M
    // ممكن 51.2M يكون الأقرب من options تانية زي 48 MHz
```

المشكلة إن `CLK_MULTIPLIER_ROUND_CLOSEST` سمحت بـ rate أعلى من المطلوب، بينما السلوك الافتراضي (بدون flag) في `__is_best_rate`:

```c
return new >= rate && new < best;
// يشترط: new >= rate (مش أقل)، لكن ممكن يختار أعلى
```

كلاهما ممكن يديك قيمة أعلى. المشكلة الأساسية إن الـ PLL مش قادر يعمل بالظبط 50 MHz، وبيعطي 51.2 MHz كأقرب قيمة.

#### الحل

**خطوة 1**: اعرف إيه القيم الممكنة للـ parent PLL:

```bash
# جرب قيم مختلفة
for rate in 48000000 49000000 50000000 51000000; do
    echo $rate > /sys/kernel/debug/clk/pll3_r/clk_rate
    cat /sys/kernel/debug/clk/pll3_r/clk_rate
done
```

**خطوة 2**: لو الـ PLL مش بيدي 50 MHz بالظبط، حدد أعلى قيمة أقل منه (49.152 MHz مثلاً):

```c
// في الـ SPI driver، اطلب rate أقل بهامش أمان
#define SPI_ADC_MAX_RATE    50000000UL
#define SPI_RATE_MARGIN     200000UL   /* 200 kHz هامش */

clk_set_rate(spi_clk, SPI_ADC_MAX_RATE - SPI_RATE_MARGIN);
```

**خطوة 3**: أو ازل `CLK_MULTIPLIER_ROUND_CLOSEST` من الـ multiplier flags وضيف `CLK_MULTIPLIER_ROUND_CLOSEST` بس للـ clocks اللي ينفع تتجاوز الـ limit:

```c
// في STM32MP1 clock driver
static struct clk_multiplier spi_mult = {
    .reg   = ...,
    .shift = 4,
    .width = 4,
    .flags = 0,  /* بدون ROUND_CLOSEST — نختار دايمًا أقل أو يساوي */
    .lock  = &spi_mult_lock,
};
```

**تحقق**:

```bash
cat /sys/kernel/debug/clk/spi2_k/clk_rate
# Expected: <= 50000000
```

#### الدرس المستفاد
الـ `CLK_MULTIPLIER_ROUND_CLOSEST` خطر لـ devices عندها maximum frequency hard limit. استخدمها بس لو الـ device تقدر تشتغل فوق وتحت الـ target بشكل متساوي (زي audio sample rates). للـ ADC أو peripheral بـ maximum spec، اتركها disabled عشان الـ `__is_best_rate` الـ default يختار دايمًا rate أقل من أو يساوي المطلوب.

---

### السيناريو 4: Automotive ECU على i.MX8 — نظام الـ CAN Bus بيتوقف بعد warm reset

#### العنوان
**الـ CAN clock بيرجع صفر بعد warm reset بسبب مفيش `CLK_MULTIPLIER_ZERO_BYPASS`**

#### السياق
شركة automotive بتبني ECU على NXP i.MX8MP. الـ ECU بيتحكم في body electronics عن طريق CAN bus. الـ system بيعمل warm reset كل ما يحصل fault. بعد الـ reset، الـ CAN بيشتغل تاني، لكن في 1 من كل 10 resets الـ CAN بيفضل صامت والـ ECU بيحتاج hard power cycle.

#### المشكلة
لما الـ warm reset بيحصل، الـ CAN controller بيحاول يتواصل لكن الـ bus مفيش activity. الـ engineer لاحظ:

```bash
# بعد warm reset مباشرة
cat /sys/kernel/debug/clk/can1_root/clk_rate
# Output: 0
```

#### التحليل
الـ clock tree:

```
sys_pll1_800m
    └── can_mult  [clk-multiplier, shift=8, width=3, NO ZERO_BYPASS]
            └── can1_root
```

أثناء warm reset، بعض الـ CRU registers بترجع لـ reset values. الـ `can_mult` register بعد reset = `0` في الـ bit field.

في `clk_multiplier_recalc_rate`:

```c
val = clk_mult_readl(mult) >> mult->shift;
val &= GENMASK(mult->width - 1, 0);
// val = 0 بعد reset

if (!val && mult->flags & CLK_MULTIPLIER_ZERO_BYPASS)
    val = 1;
// CLK_MULTIPLIER_ZERO_BYPASS مش set!
// val يفضل 0

return parent_rate * val;  // parent_rate * 0 = 0
```

الـ CCF بيخزن الـ rate = 0. لما الـ CAN driver بيتعمل probe تاني بعد reset، بيطلب `clk_enable` وبيشوف rate = 0 (valid من ناحية الـ framework)، فمش بيعمل error لكن الـ hardware مش بيشتغل.

الـ race condition: أحيانًا الـ bootloader بيكتب القيمة الصح في الـ register قبل ما الـ kernel يقرأها (الحالة العادية)، وأحيانًا الـ kernel بيقرأ قبل ما الـ bootloader يكمل (الحالة الغلط).

#### الحل

**الإصلاح الأساسي** — ضيف `CLK_MULTIPLIER_ZERO_BYPASS`:

```c
// في drivers/clk/imx/clk-imx8mp.c
static const struct clk_mult_data can_mult_data = {
    .reg    = 0x8880,
    .shift  = 8,
    .width  = 3,
    .flags  = CLK_MULTIPLIER_ZERO_BYPASS,  /* صفر = bypass = parent rate */
};
```

لو `CLK_MULTIPLIER_ZERO_BYPASS` set:

```c
if (!val && mult->flags & CLK_MULTIPLIER_ZERO_BYPASS)
    val = 1;  // zero معناه bypass، مش zero output
return parent_rate * 1;  // = parent_rate ✓
```

**أو إصلاح بديل** — اضمن bootloader يكتب القيمة:

```
# في U-Boot للـ i.MX8MP
mw.l 0x30388880 0x00000200  # set can_mult = 1 (shift=8, value=1<<8)
```

**تحقق بعد الإصلاح**:

```bash
# محاكاة الـ zero value
devmem2 0x30388880 w 0x00000000

# الـ clock rate لازم يكون parent_rate مش صفر
cat /sys/kernel/debug/clk/can1_root/clk_rate
# Expected: 800000000 (sys_pll1_800m) أو القيمة المبرمجة
```

#### الدرس المستفاد
في الـ automotive و safety-critical systems، أي clock بيتحكم في communication bus لازم يكون `CLK_MULTIPLIER_ZERO_BYPASS` set لو الـ hardware register ممكن يرجع صفر بعد reset. الـ "silent failure" (clock rate = 0 بدون error) أخطر من الـ crash لأنه بيخلي الـ system يبان شغال وهو فعليًا مش بيتواصل.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ I2C بتيجي بـ rate غلط على big-endian MMIO

#### العنوان
**قراءة register بـ wrong endianness بتديك multiplier قيمة عشوائية**

#### السياق
engineer بيعمل bring-up لـ custom industrial board على TI AM62x. الـ board فيها FPGA بتعمل clock generation عن طريق AXI-mapped registers. الـ FPGA بيستخدم big-endian MMIO. الـ I2C للـ board sensors المفروض يشتغل بـ 400 kHz (Fast Mode). بعد boot، الـ I2C بيشتغل لكن بـ rate عشوائية — كل boot rate مختلف.

#### المشكلة
```bash
# Boot 1
cat /sys/kernel/debug/clk/fpga_i2c_mult/clk_rate
# Output: 134217728000  (قيمة ضخمة جداً — غلط واضح)

# Boot 2
cat /sys/kernel/debug/clk/fpga_i2c_mult/clk_rate
# Output: 2097152000
```

الـ I2C بيبعت pulses بـ frequencies غريبة، والـ sensors مش بتستجاوب.

#### التحليل
في `clk_mult_readl`:

```c
static inline u32 clk_mult_readl(struct clk_multiplier *mult)
{
    if (mult->flags & CLK_MULTIPLIER_BIG_ENDIAN)
        return ioread32be(mult->reg);

    return readl(mult->reg);  // little-endian بالـ default
}
```

الـ FPGA بيكتب الـ multiplier value بـ big-endian. لو الـ `CLK_MULTIPLIER_BIG_ENDIAN` flag **مش set**، الـ `readl` بيقرأ الـ bytes بترتيب غلط.

مثال: الـ FPGA كتب multiplier = 4 (0x00000004 في big-endian):

```
MMIO bytes: [0x00][0x00][0x00][0x04]  <- big-endian في memory
readl()     بيقرأ: 0x04000000         <- little-endian interpretation
```

القيمة بعد shift وmask:

```c
val = 0x04000000 >> mult->shift;  // لو shift=0 → val = 0x04000000 = 67108864
val &= GENMASK(mult->width - 1, 0);
// لو width=8 → val &= 0xFF → val = 0x00
// لو width=32 → val = 67108864
```

الـ result مش 4 وده بيخلي `parent_rate * val` قيمة ضخمة أو صفر.

في `clk_mult_writel` نفس المشكلة:

```c
static inline void clk_mult_writel(struct clk_multiplier *mult, u32 val)
{
    if (mult->flags & CLK_MULTIPLIER_BIG_ENDIAN)
        iowrite32be(val, mult->reg);
    else
        writel(val, mult->reg);  // بيكتب بـ wrong endianness
}
```

يعني لما `clk_multiplier_set_rate` بيحاول يكتب قيمة جديدة، بيكتبها بـ little-endian في register بيتوقع big-endian — فالـ FPGA بيقرأ garbage.

#### الحل

**خطوة 1**: تأكيد المشكلة بقراءة raw register:

```bash
# قرأ 4 bytes من عنوان الـ FPGA I2C mult register
devmem2 0xA0010200 w
# لو النتيجة 0x04000000 وانت متوقع 0x00000004 — confirmed endianness issue
```

**خطوة 2**: تعديل الـ DT أو الـ platform driver لضيف الـ flag:

في الـ DT:
```dts
fpga_i2c_mult: i2c-mult@a0010200 {
    compatible = "custom,am62x-fpga-clk-mult";
    reg = <0x0 0xa0010200 0x0 0x4>;
    clocks = <&fpga_base_clk>;
    #clock-cells = <0>;
    big-endian;  /* triggers CLK_MULTIPLIER_BIG_ENDIAN */
};
```

في الـ driver:
```c
static int fpga_clk_mult_probe(struct platform_device *pdev)
{
    struct clk_multiplier *mult;
    // ...

    if (of_property_read_bool(pdev->dev.of_node, "big-endian"))
        mult->flags |= CLK_MULTIPLIER_BIG_ENDIAN;

    // register clk...
}
```

**خطوة 3**: تحقق بعد الإصلاح:

```bash
cat /sys/kernel/debug/clk/fpga_i2c_mult/clk_rate
# Expected: القيمة الصح (مثلاً 20000000 إذا FPGA كتب mult=2 وparent=10MHz)

# اختبر الـ I2C
i2cdetect -y 1
# يلزم يظهر الـ sensor addresses
```

**خطوة 4**: تأكد من `set_rate` كمان صح:

```bash
# اطلب rate معينة وتحقق إن الـ register اتكتب صح
echo 40000000 > /sys/kernel/debug/clk/fpga_i2c_mult/clk_rate
devmem2 0xA0010200 w
# bytes المكتوبة لازم تكون big-endian
```

#### الدرس المستفاد
الـ `CLK_MULTIPLIER_BIG_ENDIAN` flag مش مجرد تفصيلة — هو الفرق بين قيمة صح وقيمة عشوائية. لما بتتعامل مع FPGA أو custom IP cores، دايمًا تحقق من الـ endianness بـ `devmem2` قبل ما تثق في أي clock rate من الـ `clk_summary`. الـ symptom (قيمة كبيرة جداً أو عشوائية) هو علامة كلاسيكية لـ endianness mismatch في register access.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقالة | الوصف |
|---------|-------|
| [A common clock framework](https://lwn.net/Articles/472998/) | أول مقالة تشرح فكرة الـ Common Clock Framework (CCF) وسبب إنشائه لتوحيد واجهات الـ clock في الـ kernel |
| [common clk framework](https://lwn.net/Articles/486841/) | نقاش النسخة الأحدث من الـ CCF وتطور الـ API قبل دخوله للـ mainline |
| [Common struct clk implementation, v5](https://lwn.net/Articles/392902/) | مراحل التطور المبكرة لـ `struct clk` الموحدة والـ ops المرتبطة بها |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | نقاش قائم حول التوثيق الرسمي للـ CCF في مرحلته الأولى |
| [clk: implement clock rate protection mechanism](https://lwn.net/Articles/740484/) | يشرح آلية حماية الـ rate وعلاقتها بـ `clk_rate_request` المستخدمة في `determine_rate` |

---

### التوثيق الرسمي في الـ kernel

**الملفات الأساسية:**

```
Documentation/driver-api/clk.rst          ← التوثيق الرسمي للـ CCF (RST)
include/linux/clk-provider.h              ← تعريف struct clk_multiplier وكل الـ ops
drivers/clk/clk-multiplier.c             ← الملف موضوع الدراسة
drivers/clk/clk.c                         ← قلب الـ CCF وتنسيق الـ clk tree
```

**الرابط الإلكتروني للتوثيق:**
- [The Common Clk Framework — Kernel Docs](https://docs.kernel.org/driver-api/clk.html)
- [Documentation/clk.txt (kernel.org)](https://www.kernel.org/doc/Documentation/clk.txt)

---

### Commits المهمة في تاريخ الملف

الـ commits دي مرتبة من الأقدم للأحدث — كل واحدة غيّرت سلوك الملف بشكل مباشر:

| الـ commit hash | الوصف |
|----------------|-------|
| `f2e0a53271a4` | **الـ commit الأصلي** — أضاف `clk-multiplier.c` سنة 2015 بقلم Maxime Ripard لتوحيد تطبيقات الـ multiplier المتكررة |
| `25f77a3aa4cb` | إصلاح حالة الـ overflow/underflow لما `CLK_SET_RATE_PARENT` مش موجود |
| `9427b71a8505` | إضافة دعم صريح للـ big-endian عبر `CLK_MULTIPLIER_BIG_ENDIAN` |
| `5fd9c05c846d` | نقل الـ macro `to_clk_multiplier()` لـ `clk-provider.h` |
| `acba7855dda0` | حذف `clk_register_multiplier()` و`clk_unregister_multiplier()` (نظافة API) |
| `5834fd75e623` | استبدال `clk_readl/clk_writel` بـ `readl/writel` العادية |
| `62e59c4e69b3` | إزالة `io.h` من `clk-provider.h` |
| `e1bd55e5a567` | إضافة SPDX license header |
| `772e2dc59c9c` | **آخر تغيير مهم** — تحويل `round_rate()` إلى `determine_rate()` لأن الأولى deprecated |

للاطلاع على الـ commits مباشرة:
```bash
# داخل مصدر الـ kernel
git log --oneline --follow drivers/clk/clk-multiplier.c
git show f2e0a53271a4
```

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [linux-clk archive — lore.kernel.org](https://lore.kernel.org/linux-clk/) | الـ mailing list الرسمية للـ clock subsystem — فيها كل الـ patches والنقاشات |
| [Patchwork: clk sunxi-ng A23/A33 CCU](https://patchwork.kernel.org/project/linux-clk/patch/20160906121837.7517-8-maxime.ripard@free-electrons.com/) | مثال على استخدام Maxime Ripard للـ multiplier في منظومة Allwinner |
| [Patchwork: clk_set_rate_parent for CPU clock](https://patchwork.kernel.org/project/linux-clk/patch/20170721161935.18411-1-maxime.ripard@free-electrons.com/) | نقاش `CLK_SET_RATE_PARENT` الذي يتحكم في سلوك `__bestmult()` |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل المناسب:** Chapter 1 (An Introduction to Device Drivers) + Chapter 9 (Communicating with Hardware)
- **الأهمية:** يشرح `ioread32be`، `readl`، `writel` والـ memory-mapped I/O المستخدمة في `clk_mult_readl/writel`
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 10: Kernel Synchronization Methods — يشرح `spin_lock_irqsave` المستخدم في `clk_multiplier_set_rate`
  - Chapter 17: Devices and Modules — سياق الـ driver model
- **الناشر:** Addison-Wesley, ISBN: 978-0672329463

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل ذو الصلة:** Chapter 14: Kernel Debugging Techniques, وChapter 16: Embedded Linux Platform Specifics
- يشرح الـ clock trees في السياق العملي للـ embedded SoCs (ARM, MIPS)
- **الناشر:** Prentice Hall, ISBN: 978-0137016709

---

### موارد kernelnewbies.org

| الرابط | الأهمية |
|--------|---------|
| [Linux 6.7 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.7) | يذكر تحديثات الـ "Common CLK Framework" كـ subsystem |
| [Linux 6.9 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.9) | تحديثات حديثة للـ clk subsystem |
| [Linux 6.12 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.12) | أحدث تغييرات في منظومة الـ clock controllers |

---

### موارد eLinux.org

| الرابط | الأهمية |
|--------|---------|
| [Linux Clock Management Framework (PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | عرض ELC 2007 — نظرة تاريخية على الـ clock framework قبل CCF |
| [Kernel Timer Systems — elinux.org](https://elinux.org/index.php/Kernel_Timer_Systems) | سياق عام للـ timing وclocking في الـ kernel |

---

### مصادر إضافية مفيدة

| المصدر | الرابط |
|--------|--------|
| Common Clock Framework — Bootlin (ELCE 2013 PDF) | [bootlin.com](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) |
| clk-provider.h على GitHub | [torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/clk-provider.h) |
| clk-multiplier.c على GitHub | [torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/clk/clk-multiplier.c) |
| STM32MPU Clock Overview | [wiki.st.com](https://wiki.st.com/stm32mpu/wiki/Clock_overview) |

---

### كلمات البحث للاستزادة

```
linux clk_multiplier_ops kernel
linux CCF common clock framework clk_ops
linux "determine_rate" clk multiplier
linux CLK_SET_RATE_PARENT propagation
linux clk-provider.h struct clk_multiplier
linux GENMASK bitfield clock register
linux ioread32be big endian mmio
linux spin_lock_irqsave clock driver
Maxime Ripard clk multiplier free-electrons
linux clk composite multiplier divider
site:lore.kernel.org linux-clk clk-multiplier
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `clk_multiplier_set_rate` اللي بتتغير فيها الـ multiplier value جوه الـ hardware register. ده أنسب نقطة لأنها بتتنفذ كل ما حاجة في النظام طلبت تغيير frequency لأي clock بيستخدم `clk_multiplier_ops`.

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on clk_multiplier_set_rate — prints rate & parent_rate on every call
 * Tested on Linux 6.x with CONFIG_KPROBES=y and CONFIG_CLK_MULTIPLIER=y
 */

#include <linux/kernel.h>      /* pr_info, KERN_INFO */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/clk-provider.h>/* struct clk_hw, to_clk_multiplier */
#include <linux/clk.h>         /* clk_hw_get_name */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-arabic");
MODULE_DESCRIPTION("kprobe on clk_multiplier_set_rate to trace multiplier changes");

/* ------------------------------------------------------------------ */
/*  pre-handler: بيتنفذ قبل ما الدالة الأصلية تبدأ                    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Linux x86-64 calling convention:
     *   rdi = 1st arg  → struct clk_hw *hw
     *   rsi = 2nd arg  → unsigned long rate
     *   rdx = 3rd arg  → unsigned long parent_rate
     *
     * على ARM64: x0, x1, x2
     * نستخدم regs_get_kernel_argument() اللي portable
     */
    struct clk_hw *hw          = (struct clk_hw *)regs_get_kernel_argument(regs, 0);
    unsigned long  rate        = (unsigned long)  regs_get_kernel_argument(regs, 1);
    unsigned long  parent_rate = (unsigned long)  regs_get_kernel_argument(regs, 2);

    const char *clk_name = "(unknown)";

    /* clk_hw_get_name بترجع اسم الـ clock من struct clk_core */
    if (hw)
        clk_name = clk_hw_get_name(hw);

    pr_info("[clk_mult_probe] clock='%s'  rate=%lu Hz  parent_rate=%lu Hz  mult=%lu\n",
            clk_name,
            rate,
            parent_rate,
            (parent_rate > 0) ? (rate / parent_rate) : 0UL);

    return 0; /* 0 = استمر في تنفيذ الدالة الأصلية */
}

/* ------------------------------------------------------------------ */
/*  struct kprobe — بيحدد الدالة المستهدفة والـ callback              */
/* ------------------------------------------------------------------ */
static struct kprobe clk_mult_kp = {
    .symbol_name = "clk_multiplier_set_rate", /* اسم الـ symbol في kernel */
    .pre_handler = handler_pre,               /* يتنفذ قبل الدالة */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init clk_mult_probe_init(void)
{
    int ret;

    ret = register_kprobe(&clk_mult_kp);
    if (ret < 0) {
        pr_err("[clk_mult_probe] register_kprobe failed: %d\n", ret);
        /*
         * ممكن يفشل لو:
         *   - الـ symbol مش موجود (CONFIG_CLK_MULTIPLIER مش enabled)
         *   - الـ kernel مش compiled بـ CONFIG_KPROBES=y
         *   - الدالة inline أو في قائمة blacklist
         */
        return ret;
    }

    pr_info("[clk_mult_probe] planted on %s @ %px\n",
            clk_mult_kp.symbol_name,
            clk_mult_kp.addr); /* .addr بيتملى تلقائياً بعد register */

    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit clk_mult_probe_exit(void)
{
    unregister_kprobe(&clk_mult_kp);
    /*
     * لازم نعمل unregister قبل ما الـ module يتـ unload،
     * عشان لو جه call للدالة بعد كده والـ handler مش موجود → kernel panic
     */
    pr_info("[clk_mult_probe] removed\n");
}

module_init(clk_mult_probe_init);
module_exit(clk_mult_probe_exit);
```

### Makefile للبناء

```makefile
# ضع الـ Makefile في نفس الـ directory مع clk_mult_probe.c
obj-m += clk_mult_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
make
sudo insmod clk_mult_probe.ko
# شغّل أي حاجة بتغير clock frequency مثلاً: cpufreq أو عن طريق /sys/kernel/debug/clk/
sudo dmesg | grep clk_mult_probe
sudo rmmod clk_mult_probe
```

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيعرّف `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/clk-provider.h` | بيعرّف `struct clk_hw`، `to_clk_multiplier`، و `clk_hw_get_name` |
| `linux/module.h` | ماكروهات الـ module الأساسية زي `module_init`، `MODULE_LICENSE` |

**الـ** `linux/kprobes.h` هو اللب الأساسي، من غيره مفيش طريقة نعمل hook على أي دالة في الـ kernel بدون تعديل الكود الأصلي.

#### الـ `handler_pre` والـ arguments

بنستخدم `regs_get_kernel_argument(regs, N)` بدل ما نقرأ الـ registers مباشرة، عشان الدالة دي portable على x86-64 وARM64 في نفس الوقت ومش بتحتاج `#ifdef`.

**الـ** `pr_info` بيطبع اسم الـ clock والـ rate المطلوبة والـ parent_rate، ومنهم بنحسب الـ multiplier المتوقع — ده بيخلينا نشوف أي clock في النظام بتتغير frequency-ها ومين بيطلب كام.

#### الـ `struct kprobe`

**الـ** `.symbol_name` بيقول للـ kernel يدور على الدالة دي في الـ symbol table ويحسب عنوانها تلقائياً، أحسن من hardcode address لأن الـ address بيتغير بين kernel builds لو KASLR شغّال.

#### `module_init` و `module_exit`

**الـ** `register_kprobe` بيحط breakpoint افتراضي على أول instruction في الدالة المستهدفة؛ لو الـ registration فشل بنرجع الـ error فوراً ومبنخليش الـ module يفضل loaded بدون hook فعّال.

**الـ** `unregister_kprobe` في الـ exit ضروري جداً — الـ kernel بيحذف كود الـ module من الذاكرة لما يتـ unload، فلو جه request لـ `clk_multiplier_set_rate` بعد كده والـ handler pointer لسه شايل على memory محذوفة هيحصل use-after-free وـ kernel panic.
