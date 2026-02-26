## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `clk-fixed-factor.c` جزء من **Common Clock Framework (CCF)** — الـ subsystem اللي بيتحكم في كل الـ clocks جوه الـ Linux kernel. الـ maintainers هما Michael Turquette و Stephen Boyd، والـ mailing list هو `linux-clk@vger.kernel.org`.

---

### الـ Clock Framework — ليه موجود أصلاً؟

تخيل إنك بتصمم موبايل فيه:
- **CPU** بيشتغل على 1.8 GHz
- **USB controller** بيحتاج 60 MHz
- **UART** بيحتاج 48 MHz
- **Camera sensor** بيحتاج 24 MHz

مصدر الساعة الأصلي (الـ **crystal oscillator**) بيدي مثلاً 24 MHz بس. إزاي توصل للـ 1.8 GHz؟ وإزاي توصل للـ 48 MHz؟

الإجابة: عن طريق **شجرة من الـ clocks** — كل clock بييجي من clock أعلى منه (الـ **parent**) وبيضرب أو بيقسم الـ frequency بنسبة ثابتة أو متغيرة.

```
crystal (24 MHz)
    └─── PLL (×75 = 1800 MHz)  ← CPU clock
    └─── /2 (12 MHz)
    └─── /1 ×2 (48 MHz)        ← UART clock
    └─── /1 (24 MHz)           ← Camera clock
```

الـ **Common Clock Framework** هو الـ kernel subsystem اللي بينظم الشجرة دي، وبيوفر API موحد لكل الـ drivers عشان يطلبوا الـ clock اللي محتاجينه بدون ما يعرفوا التفاصيل الـ hardware.

---

### الـ Fixed-Factor Clock — ما هو بالضبط؟

الـ **fixed-factor clock** هو أبسط نوع clock في الشجرة دي. هو مجرد:

```
output_rate = parent_rate * mult / div
```

حيث `mult` و `div` **ثابتين — مش ممكن تغيرهم بالـ software**. مفيش register بتكتب فيه. مفيش زر بتضغطه. النسبة محددة من تصميم الـ hardware.

**مثال واقعي:** على SoC زي Raspberry Pi، فيه clock مصدره الـ PLL بيدي 1 GHz. الـ USB controller محتاج 500 MHz بالظبط. الحل: fixed-factor clock بـ `mult=1, div=2`. ده بيقسم الـ 1 GHz على 2 ويدي 500 MHz ثابت. مش محتاج أكتر من كده.

---

### القصة الكاملة — المشكلة اللي بيحلها الملف ده

**المشكلة:**
كل SoC عنده عشرات أو مئات الـ clocks. الجزء الأكبر منها مجرد تقسيم أو ضرب ثابت من clock أعلى. لو كل driver كتب الـ math ده بنفسه، هيبقى في **code duplication** هائل وهيصعب إدارة شجرة الـ clocks.

**الحل اللي بيقدمه `clk-fixed-factor.c`:**
بيوفر **نوع clock جاهز ومعياري** أي driver أو SoC يقدر يستخدمه بكل بساطة. بدل ما تكتب logic، بتقول بس: "عايز clock اسمه `usb_clk`، أبوه `pll_clk`، نسبته `mult=1, div=4`" — وخلاص.

---

### الصورة الكاملة — تدفق البيانات

```
Device Tree (DTS)                  Kernel Boot
─────────────────                  ────────────
fixed-factor-clock {               CLK_OF_DECLARE
  clock-mult = <1>;    ──────────► of_fixed_factor_clk_setup()
  clock-div  = <2>;                    │
  clocks = <&pll>;                     │
}                                      ▼
                               __clk_hw_register_fixed_factor()
                                       │
                                       ▼
                               struct clk_fixed_factor {
                                   mult = 1
                                   div  = 2
                                   hw   → clk_fixed_factor_ops
                               }
                                       │
                                       ▼
                               clk_hw_register()  ← يسجل في CCF
                                       │
                               ─────────────────────────────
                               أي driver يطلب الـ clock:
                               clk_get_rate(usb_clk)
                               └─► recalc_rate: parent_rate * 1 / 2
```

---

### الـ ops الأربعة — قلب الملف

| **Operation** | **الدالة** | **الهدف** |
|---|---|---|
| `recalc_rate` | `clk_factor_recalc_rate` | يحسب الـ rate الفعلية: `parent * mult / div` |
| `determine_rate` | `clk_factor_determine_rate` | يقول للـ CCF "أقرب rate ممكنة لما طلب" |
| `set_rate` | `clk_factor_set_rate` | بيرجع 0 (نجاح وهمي) لأن النسبة ثابتة |
| `recalc_accuracy` | `clk_factor_recalc_accuracy` | بيحسب دقة الـ clock بالـ ppb |

---

### طرق الـ Registration — المرونة في الاستخدام

الملف بيوفر أكتر من طريقة لتسجيل الـ clock عشان يناسب كل السيناريوهات:

| **الدالة** | **متى تستخدمها** |
|---|---|
| `clk_hw_register_fixed_factor` | التسجيل العادي باسم الـ parent |
| `clk_hw_register_fixed_factor_index` | الـ parent من DT بـ index |
| `clk_hw_register_fixed_factor_fwname` | الـ parent من firmware name |
| `clk_hw_register_fixed_factor_with_accuracy_fwname` | نفسه + دقة ثابتة |
| `clk_hw_register_fixed_factor_parent_hw` | الـ parent كـ pointer مباشر |
| `devm_*` variants | نفس الأنواع بس بـ devres (automatic cleanup) |

**الـ `devm_` prefix** معناه إن الـ kernel هيتولى تحرير الـ memory تلقائياً لما الـ device يتشال — مش محتاج تعمل `free` يدوي.

---

### الـ Device Tree Integration

```dts
/* مثال من device tree */
div_clk: clock-divider {
    compatible = "fixed-factor-clock";
    clocks = <&pll_clk>;    /* الـ parent */
    clock-mult = <1>;
    clock-div  = <4>;
    clock-output-names = "div_clk";
};
```

الـ `CLK_OF_DECLARE(fixed_factor_clk, "fixed-factor-clock", of_fixed_factor_clk_setup)` في آخر الملف بيقول للـ kernel: "لما تلاقي `compatible = "fixed-factor-clock"` في الـ DT، اتصل بالـ function دي عشان تعمل setup للـ clock."

---

### الـ Files المهمة اللي المقروء يعرفها

```
drivers/clk/
├── clk.c                    ← قلب الـ CCF — الـ core logic كله هنا
├── clk-fixed-factor.c       ← الملف ده — fixed multiplier/divider
├── clk-fixed-rate.c         ← clock بـ rate ثابت بالكامل (مش محتاج parent)
├── clk-divider.c            ← divider قابل للتغيير (variable)
├── clk-multiplier.c         ← multiplier قابل للتغيير
├── clk-mux.c                ← اختيار بين أكتر من parent
├── clk-gate.c               ← تشغيل وإيقاف الـ clock
├── clk-composite.c          ← تركيب عدة عمليات في clock واحد
└── clk-devres.c             ← devres helpers للتسجيل

include/linux/
├── clk-provider.h           ← struct clk_fixed_factor + كل الـ APIs
├── clk.h                    ← واجهة الـ consumer (الـ drivers اللي بتستخدم الـ clocks)
└── of_clk.h                 ← دوال الـ Device Tree integration

Documentation/devicetree/bindings/clock/
└── fixed-factor-clock.yaml  ← مواصفات الـ DT binding
```

---

### ملخص الفكرة في جملة واحدة

**الـ `clk-fixed-factor.c`** بيوفر أبسط building block في شجرة الـ clocks: clock بيأخذ rate أبوه ويضربه أو يقسمه بنسبة ثابتة من الـ hardware — مش محتاج register، مش محتاج ضبط، بس حساب رياضي بسيط يتكرر في عشرات الـ SoCs فعملوا له module مستقل ومعياري.
## Phase 2: شرح الـ Common Clock Framework (CCF) وموقع الـ clk-fixed-factor فيه

### المشكلة اللي بيحلها الـ Subsystem ده

في أي SoC حديث (زي Allwinner A64 أو i.MX8 أو Rockchip RK3399) فيه عشرات لو مش مئات الـ clocks. كل clock ممكن:

- يتولّد من crystal oscillator خارجي
- يتضاعف عن طريق PLL
- يتقسم على divisor ثابت أو متغير
- يتقفّل (gate) لتوفير الطاقة

المشكلة القديمة: كل driver كان بيتعامل مع الـ hardware registers بتاعت الـ clock بنفسه. النتيجة:

- **كود متكرر** في كل driver
- **race conditions** لما أكتر من driver بيحاول يغير نفس الـ clock
- **power waste** لأن مفيش تتبع مركزي لمين شغّال الـ clock ومين لأ
- **صعوبة الـ dependency tracking** — لو قفّلت clock مشترك، هتكسر 3 drivers

### الحل — الـ Common Clock Framework

الكيرنل جاب الـ **CCF (Common Clock Framework)** في الـ 3.4 وعرّف abstraction موحدة:

1. **tree of clocks** — كل clock عارف parent-ه، والـ framework بيمشي الشجرة لما محتاج يغير rate أو يقفل clock
2. **reference counting** — الـ clock مش بيتقفل غير لما آخر consumer يعمل له `clk_disable`
3. **ops table** — كل نوع clock بيقدم callbacks ثابتة، والـ framework يستخدمهم بشكل موحد
4. **provider/consumer split** — الـ driver اللي بيعرّف الـ clock (provider) منفصل تماماً عن الـ driver اللي بيستخدمه (consumer)

### الـ Analogy — شبكة المياه

فكّر في شبكة مياه مدينة:

| مفهوم المياه | مقابله في CCF |
|---|---|
| محطة المعالجة (المصدر) | crystal oscillator — `clk_fixed_rate` |
| محطة رفع الضغط (PLL) | PLL clock — تضاعف الـ frequency |
| صمام تقليل ضغط | divider clock — `clk_divider` أو `clk_fixed_factor` |
| صنبور في الشقة | consumer driver يعمل `clk_enable` |
| عداد المياه المركزي | CCF reference counting |
| خريطة الأنابيب | clock tree في الـ CCF |

**الـ `clk-fixed-factor` بالتحديد** هو الصمام اللي نسبة الضغط فيه ثابتة من المصنع — مش ممكن تغيرها، بس الكيرنل محتاج يعرف عنها عشان يحسب الـ rates الصح.

لما consumer يطلب `clk_get_rate("uart_clk")`:
1. الـ CCF بيسأل `uart_clk` عن rate-ه (`recalc_rate`)
2. الـ `uart_clk` بيسأل parent-ه عن rate-ه
3. لو أبوه `clk_fixed_factor` بـ mult=1 و div=2، هيرجّع `parent_rate / 2`
4. الـ CCF يجمّع النتيجة ويرجّعها للـ consumer

### الصورة الكبيرة — معمارية الـ CCF

```
+------------------------------------------------------------------+
|                      Consumer Drivers                            |
|   uart driver       spi driver        usb driver                 |
|   clk_enable()      clk_set_rate()    clk_get_rate()             |
+---------------------------+----------------------------------+---+
                            |  clk_* API (include/linux/clk.h)
+---------------------------v----------------------------------+---+
|                                                                  |
|              Common Clock Framework (CCF) Core                   |
|              drivers/clk/clk.c                                   |
|                                                                  |
|   - clock tree management (parent/child relationships)           |
|   - reference counting (prepare_count, enable_count)            |
|   - rate propagation up/down the tree                           |
|   - debugfs (/sys/kernel/debug/clk/)                            |
|   - OF (device tree) clock provider resolution                  |
|                                                                  |
+---+-------------+-------------+-------------+------------------+-+
    |             |             |             |
    v             v             v             v
+-------+   +---------+   +--------+   +----------+
| fixed |   | fixed   |   | gate   |   | divider  |
| rate  |   | factor  |   | clock  |   | clock    |
|       |   |(THIS    |   |        |   |          |
|clk-   |   |FILE)    |   |clk-    |   |clk-      |
|fixed- |   |clk-     |   |gate.c  |   |divider.c |
|rate.c |   |fixed-   |   |        |   |          |
+-------+   |factor.c |   +--------+   +----------+
            +---------+
                |
     implements clk_ops:
     .recalc_rate
     .determine_rate
     .set_rate (nop)
     .recalc_accuracy
```

```
                    Clock Tree Example (i.MX8)
                    ==========================

    [24 MHz XTAL OSC]          <- clk_fixed_rate
           |
    [PLL1 = 800 MHz]           <- platform-specific PLL clock
           |
    [AHB_CLK = 400 MHz]        <- clk_fixed_factor (mult=1, div=2)
           |
    [IPG_CLK = 200 MHz]        <- clk_fixed_factor (mult=1, div=2)
           |
    [UART_CLK_ROOT]            <- clk_gate (can be disabled)
           |
    [uart1_clk]                <- consumer: uart driver
```

### الـ Core Abstraction — ثلاث طبقات

الـ CCF بيفصل بين ثلاث طبقات:

#### 1. الـ `struct clk` — واجهة الـ consumer

```c
// ما بيشوفهوش الـ driver writer عادةً
// بيتعامل معاه عن طريق clk_get(), clk_enable(), etc.
struct clk;
```

**الـ `struct clk`** هو الـ handle اللي الـ consumer driver بياخده من `clk_get()`. ده per-consumer — يعني لو تلاتة drivers استخدموا نفس الـ clock، فيه تلات instances من `struct clk` بس clock واحد في الـ hardware.

#### 2. الـ `struct clk_core` — التمثيل الداخلي

الـ CCF core بيحتفظ بـ `struct clk_core` واحدة لكل clock، بتحتوي على:
- اسم الـ clock
- الـ parent و الـ children
- الـ cached rate
- الـ prepare/enable reference counts
- pointer للـ `clk_hw`

#### 3. الـ `struct clk_hw` — واجهة الـ provider

```c
struct clk_hw {
    struct clk_core *core;   // back-pointer للـ CCF core
    struct clk      *clk;    // الـ default consumer handle
    const struct clk_init_data *init;  // بيتصفّر بعد التسجيل
};
```

**الـ `struct clk_hw`** هو الجسر بين الـ CCF وبين الـ hardware-specific struct. دايماً بيكون **أول field** في أي hardware-specific struct زي `struct clk_fixed_factor`.

#### ربط الثلاث طبقات

```
Consumer Driver          CCF Core              Provider Driver
+------------+        +----------+           +------------------+
|            |        |          |           | struct clk_      |
| struct clk |<------>| struct   |<--------->| fixed_factor {   |
| (per user) |        | clk_core |           |   struct clk_hw  |
|            |        |          |           |   hw;     <------+--+
+------------+        +----------+           |   unsigned mult; |  |
                           |                 |   unsigned div;  |  |
                           |                 |   unsigned long  |  |
                           |                 |   acc;           |  |
                           |                 |   unsigned flags;|  |
                           |                 | }                |  |
                           |                 +------------------+  |
                           |                                       |
                           +--- clk_core->hw ----------------------+
                                points to &fix->hw
```

### الـ `struct clk_fixed_factor` بالتفصيل

```c
struct clk_fixed_factor {
    struct clk_hw   hw;       // MUST be first — used by container_of
    unsigned int    mult;     // multiplier (البسط)
    unsigned int    div;      // divider (المقام)
    unsigned long   acc;      // accuracy in ppb (parts per billion)
    unsigned int    flags;    // CLK_FIXED_FACTOR_FIXED_ACCURACY
};
```

المعادلة الأساسية:

```
output_rate = (parent_rate * mult) / div
```

**مثال حقيقي:** SoC عنده PLL بيدي 1 GHz، ومحتاج clock للـ AHB bus بـ 250 MHz:

```c
// في driver الـ SoC
static CLK_FIXED_FACTOR(ahb_clk, "ahb", "pll1", 4, 1, 0);
//                                              div mult flags
// ahb_rate = pll1_rate * 1 / 4 = 1GHz / 4 = 250 MHz
```

### الـ `struct clk_ops` للـ fixed-factor

```c
const struct clk_ops clk_fixed_factor_ops = {
    .determine_rate  = clk_factor_determine_rate,
    .set_rate        = clk_factor_set_rate,        // nop دايماً
    .recalc_rate     = clk_factor_recalc_rate,
    .recalc_accuracy = clk_factor_recalc_accuracy,
};
```

#### لماذا `.set_rate` موجود لو هو nop؟

لأن الـ CCF بيفرق بين نوعين من الـ clocks:

- **clocks بدون `.set_rate`**: الـ CCF مش هيسمح بأي محاولة لتغيير الـ rate
- **clocks بـ `.set_rate`**: الـ CCF هيسمح بالمحاولة، والـ driver بيقول "تمام" حتى لو ما عملش حاجة

الـ `clk_fixed_factor` بيوفر `.set_rate` لأنه بيدعم `CLK_SET_RATE_PARENT` — يعني لو حد طلب rate مختلف، ممكن الـ clock يطلب من parent-ه يغيّر rate-ه هو بدل ما يرفض الطلب. الـ `.determine_rate` بيحسب الـ best_parent_rate المناسبة، وبعدين `.set_rate` بيرجّع 0 لأن التغيير الفعلي حصل في الـ parent.

#### الـ `clk_factor_recalc_rate` — ليه `do_div`؟

```c
static unsigned long clk_factor_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    struct clk_fixed_factor *fix = to_clk_fixed_factor(hw);
    unsigned long long int rate;

    // بنعمل cast لـ u64 قبل الضرب عشان نتجنب overflow
    // مثلاً: 1GHz * 4 = 4,000,000,000 > ULONG_MAX على 32-bit
    rate = (unsigned long long int)parent_rate * fix->mult;
    do_div(rate, fix->div);  // do_div آمن على 32-bit و 64-bit
    return (unsigned long)rate;
}
```

**الـ `do_div` macro** — ده مهم: على ARM 32-bit، الـ hardware مش بيعمل 64-bit division مباشرة. الـ `do_div` بيستخدم الـ `__udivdi3` من `libgcc` أو routine خاصة في الكيرنل تناسب الـ architecture.

#### الـ `clk_factor_determine_rate` — propagation للـ parent

```c
static int clk_factor_determine_rate(struct clk_hw *hw,
                     struct clk_rate_request *req)
{
    struct clk_fixed_factor *fix = to_clk_fixed_factor(hw);

    if (clk_hw_get_flags(hw) & CLK_SET_RATE_PARENT) {
        unsigned long best_parent;

        // عايزين rate معين → إيه الـ parent rate المطلوبة؟
        // rate = parent * mult / div
        // parent = rate * div / mult
        best_parent = (req->rate / fix->mult) * fix->div;

        // اسأل الـ parent: أقرب rate تقدر تديه لـ best_parent هو إيه؟
        req->best_parent_rate = clk_hw_round_rate(
            clk_hw_get_parent(hw), best_parent);
    }

    // الـ rate الفعلي اللي هنديه = parent_rate * mult / div
    req->rate = (req->best_parent_rate / fix->div) * fix->mult;

    return 0;
}
```

**الـ `struct clk_rate_request`** — ده الـ struct اللي الـ CCF بيستخدمه لتمرير القيود بين الـ clocks في الشجرة:

```c
struct clk_rate_request {
    struct clk_core *core;
    unsigned long    rate;             // الـ rate المطلوب / المحسوب
    unsigned long    min_rate;         // الحد الأدنى
    unsigned long    max_rate;         // الحد الأعلى
    unsigned long    best_parent_rate; // الـ parent rate الأنسب
    struct clk_hw   *best_parent_hw;  // الـ parent الأنسب
};
```

### طرق التسجيل — من يسجّل ومتى

الملف بيوفر **تسع دوال تسجيل** مختلفة. الاختلاف الجوهري في:

1. **طريقة تحديد الـ parent**
2. **إدارة الـ memory** (manual vs devres)

```
                    طرق تحديد الـ Parent
                    ====================

clk_hw_register_fixed_factor()          ← باسم نصي (string name)
      |
      | parent_name = "pll1"

clk_hw_register_fixed_factor_parent_hw() ← بـ pointer مباشر
      |
      | parent_hw = &pll1_hw

clk_hw_register_fixed_factor_index()    ← بـ DT index
      |
      | index = 0 → first clock in 'clocks' property

clk_hw_register_fixed_factor_fwname()   ← بـ firmware name من DT
      |
      | fw_name = "pll" → matches clock-names property in DT
```

#### Manual vs Devres

| دالة | memory management |
|---|---|
| `clk_hw_register_fixed_factor()` | `kfree()` يدوي عند الإزالة |
| `devm_clk_hw_register_fixed_factor()` | الـ kernel بيعمل `kfree` تلقائياً عند `device_remove` |

الـ **devres** (device resource management) — subsystem تاني محتاج تفهمه: بيربط الـ resources بالـ device lifecycle، لما الـ device بيتشال من النظام، الـ kernel تلقائياً بيعمل cleanup لكل الـ resources المسجلة.

```c
// devres path
if (devm)
    fix = devres_alloc(
        devm_clk_hw_register_fixed_factor_release, // cleanup function
        sizeof(*fix),
        GFP_KERNEL
    );
// ...
if (devm)
    devres_add(dev, fix); // ربط الـ resource بالـ device
```

### الـ Device Tree Integration

```c
// في DTS:
ahb_clk: ahb-clk {
    compatible = "fixed-factor-clock";
    clocks = <&pll1>;        // parent
    clock-mult = <1>;
    clock-div = <4>;
    #clock-cells = <0>;
};
```

الـ kernel بيقرأ ده في وقت مبكر جداً عن طريق:

```c
CLK_OF_DECLARE(fixed_factor_clk, "fixed-factor-clock",
        of_fixed_factor_clk_setup);
```

**`CLK_OF_DECLARE`** — ده macro بيحط الـ callback في section خاص في الـ binary (`__clk_of_table`) والـ CCF بيمشي فيه أثناء `of_clk_init()` قبل أي `initcall` عادي. ده ضروري لأن بعض الـ clocks لازم تكون جاهزة قبل حتى ما الـ bus drivers تشتغل.

**ليه فيه `platform_driver` كمان؟**

لو الـ `CLK_OF_DECLARE` فشل (مثلاً لأن الـ module مش موجود وقت الـ early init)، الـ `OF_POPULATED` flag بيتمسح وبيسمح للـ `platform_driver` يحاول تاني في وقت لاحق من الـ boot.

```
                Boot Sequence
                =============

early_init
    |
    +---> of_clk_init()
              |
              +---> CLK_OF_DECLARE callbacks   ← أول محاولة
                        |
                   نجح؟ ← نعم → خلاص
                        |
                       لأ → of_node_clear_flag(OF_POPULATED)
                              |
                              v
subsys_initcall
    |
    +---> platform_driver_register()
              |
              +---> of_fixed_factor_clk_probe()  ← تاني محاولة
```

### الـ CCF بيمتلك إيه وبيفوّض إيه؟

#### ما يمتلكه الـ CCF:

- **clock tree** وكل الـ parent/child relationships
- **reference counting** للـ prepare/enable
- **rate caching** وإبلاغ الـ consumers لما الـ rate يتغير (`clk_notifier`)
- **debugfs** interface (`/sys/kernel/debug/clk/`)
- **OF resolution** — ترجمة الـ phandles لـ clk_hw pointers

#### ما يفوّضه للـ provider (clk-fixed-factor في حالتنا):

- **حساب الـ rate الفعلي** (`recalc_rate`) — الـ CCF ما بيعرفش الـ mult/div
- **تحديد الـ best achievable rate** (`determine_rate`)
- **حساب الـ accuracy** (`recalc_accuracy`)
- **أي interaction مع hardware registers** (في حالة الـ fixed-factor مفيش، بس في الـ gate/divider فيه)

### رسم العلاقات بين الـ Structs

```
__clk_hw_register_fixed_factor()
             |
             | kmalloc أو devres_alloc
             v
+------------------------------------+
|     struct clk_fixed_factor        |
|                                    |
|  +-----------------------------+   |
|  |      struct clk_hw hw       |   |
|  |                             |   |
|  | +------+  +-----------+     |   |
|  | | core |->| clk_core  |     |   |
|  | |      |  |           |     |   |
|  | |      |  | name      |     |   |
|  | |      |  | rate      |     |   |
|  | |      |  | ref_count |     |   |
|  | |      |  | parent----+---> parent clk_core
|  | |      |  | children  |     |   |
|  | +------+  +-----------+     |   |
|  |                             |   |
|  |  init --> clk_init_data     |   |  (بيتصفر بعد clk_hw_register)
|  |           .name             |   |
|  |           .ops = &clk_      |   |
|  |              fixed_factor_  |   |
|  |              ops            |   |
|  |           .parent_names/   |   |
|  |            parent_data/    |   |
|  |            parent_hws      |   |
|  +-----------------------------+   |
|                                    |
|  mult = 1                          |
|  div  = 4                          |
|  acc  = 0                          |
|  flags= 0                          |
+------------------------------------+

to_clk_fixed_factor(hw) = container_of(hw, struct clk_fixed_factor, hw)
```

الـ `container_of` macro بيحسب عنوان الـ `struct clk_fixed_factor` من عنوان الـ `hw` field داخله — ده ممكن لأن `hw` دايماً أول field في الـ struct.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### CLK Flags (الـ `flags` بتاعة `clk_init_data`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_SET_RATE_GATE` | `BIT(0)` | لازم يتقفل الـ clock وقت تغيير الـ rate |
| `CLK_SET_PARENT_GATE` | `BIT(1)` | لازم يتقفل وقت تغيير الـ parent |
| `CLK_SET_RATE_PARENT` | `BIT(2)` | يبعّت تغيير الـ rate للـ parent فوق |
| `CLK_IGNORE_UNUSED` | `BIT(3)` | ميقفلوش حتى لو مش مستخدم |
| `CLK_GET_RATE_NOCACHE` | `BIT(6)` | ميستخدمش الـ cached rate |
| `CLK_IS_CRITICAL` | `BIT(11)` | ميتقفلش أبدًا |
| `CLK_OPS_PARENT_ENABLE` | `BIT(12)` | الـ parents محتاجة تتفعّل وقت gate/set_rate |
| `CLK_DUTY_CYCLE_PARENT` | `BIT(13)` | يعدّي duty cycle للـ parent |

#### CLK_FIXED_FACTOR Flags (الـ `flags` بتاعة `clk_fixed_factor` نفسها)

| Flag | Value | المعنى |
|------|-------|--------|
| `CLK_FIXED_FACTOR_FIXED_ACCURACY` | `BIT(0)` | يستخدم الـ `acc` المحدد بدل دقة الـ parent |

#### Macros للـ Static Declaration

| Macro | الاستخدام |
|-------|-----------|
| `CLK_FIXED_FACTOR` | يعرّف struct ثابت بـ parent name string |
| `CLK_FIXED_FACTOR_HW` | نفس الأول بس بـ `clk_hw *` pointer للـ parent |
| `CLK_FIXED_FACTOR_HWS` | نفس الأول بس بـ array من الـ `clk_hw *` |
| `CLK_FIXED_FACTOR_FW_NAME` | يستخدم اسم الـ parent من الـ firmware |
| `to_clk_fixed_factor(_hw)` | يحوّل من `clk_hw *` لـ `clk_fixed_factor *` باستخدام `container_of` |

---

### الـ Structs المهمة

#### 1. `struct clk_fixed_factor`

**الغرض:** الـ struct الأساسي للملف ده — بيمثّل clock بنسبة ثابتة (mult/div) من الـ parent.

```c
struct clk_fixed_factor {
    struct clk_hw   hw;      /* embed — الربط بالـ CCF */
    unsigned int    mult;    /* مضاعف الـ rate */
    unsigned int    div;     /* مقسوم عليه الـ rate */
    unsigned long   acc;     /* الدقة بالـ ppb لو FIXED_ACCURACY */
    unsigned int    flags;   /* CLK_FIXED_FACTOR_* flags */
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `hw` | `struct clk_hw` | الجسر بين الـ driver والـ CCF — **أول field دايمًا** |
| `mult` | `unsigned int` | بيتضرب في rate الـ parent |
| `div` | `unsigned int` | بينقسم عليه الناتج — لازم > 0 |
| `acc` | `unsigned long` | دقة ثابتة بالـ ppb (parts per billion) |
| `flags` | `unsigned int` | يتحكم في سلوك الـ accuracy |

**الصيغة:** `rate = parent_rate * mult / div`

---

#### 2. `struct clk_hw`

**الغرض:** الـ handle الرابط بين كود الـ hardware-specific وإطار عمل الـ CCF (Common Clock Framework).

```c
struct clk_hw {
    struct clk_core         *core;  /* pointer للـ core الداخلي */
    struct clk              *clk;   /* الـ consumer handle */
    const struct clk_init_data *init; /* بيتصفّر بعد التسجيل */
};
```

| Field | الدور |
|-------|-------|
| `core` | يشير للـ `clk_core` الداخلي في الـ CCF |
| `clk` | الـ per-user handle اللي بيتعرّضه للـ consumers |
| `init` | بيانات التهيئة — بتتشيل بعد `clk_hw_register()` |

---

#### 3. `struct clk_init_data`

**الغرض:** بيانات التهيئة المشتركة بين الـ provider والـ CCF — بتتملى مرة واحدة وقت التسجيل.

```c
struct clk_init_data {
    const char              *name;          /* اسم الـ clock */
    const struct clk_ops    *ops;           /* الـ callbacks */
    /* واحد بس من التالت دول */
    const char * const      *parent_names;  /* أسماء string للـ parents */
    const struct clk_parent_data *parent_data; /* بيانات parents */
    const struct clk_hw     **parent_hws;   /* pointers مباشرة */
    u8                      num_parents;    /* عدد الـ parents */
    unsigned long           flags;          /* CLK_* flags */
};
```

---

#### 4. `struct clk_ops`

**الغرض:** جدول الـ virtual functions — الـ CCF بيكلّم الـ hardware من خلاله. الـ `clk_fixed_factor` بيملا 4 منهم بس:

| Op | الـ Fixed Factor Implementation | الملاحظة |
|----|----------------------------------|----------|
| `determine_rate` | `clk_factor_determine_rate` | يحسب أنسب rate بيتطلبها الـ parent |
| `set_rate` | `clk_factor_set_rate` | يرجّع 0 دايمًا (nop) |
| `recalc_rate` | `clk_factor_recalc_rate` | `parent * mult / div` |
| `recalc_accuracy` | `clk_factor_recalc_accuracy` | يرجّع `acc` الثابت أو accuracy الـ parent |
| باقي الـ ops | `NULL` | مش محتاجها |

---

#### 5. `struct clk_parent_data`

**الغرض:** بيصف الـ parent بطريقة مرنة — يدعم الـ DT index أو الـ fw_name أو الـ pointer المباشر.

```c
struct clk_parent_data {
    const struct clk_hw *hw;       /* pointer مباشر للـ parent */
    const char          *fw_name;  /* اسم الـ parent في الـ firmware */
    const char          *name;     /* اسم global fallback */
    int                 index;     /* index في الـ DT clocks property */
};
```

---

#### 6. `struct clk_rate_request`

**الغرض:** بيحمل الـ constraints وقت حساب الـ rate — يتبادل بين الـ CCF والـ ops.

```c
struct clk_rate_request {
    struct clk_core *core;              /* الـ clock المتأثر */
    unsigned long    rate;              /* الـ rate المطلوب / المحسوب */
    unsigned long    min_rate;          /* الحد الأدنى */
    unsigned long    max_rate;          /* الحد الأقصى */
    unsigned long    best_parent_rate;  /* أفضل rate للـ parent */
    struct clk_hw   *best_parent_hw;    /* أنسب parent */
};
```

---

### رسم علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────┐
│                  struct clk_fixed_factor                 │
│                                                         │
│  ┌──────────────────┐  mult: unsigned int               │
│  │  struct clk_hw   │  div:  unsigned int               │
│  │                  │  acc:  unsigned long               │
│  │  core ──────────────────────────────────────────────►│struct clk_core (CCF internal)
│  │  clk  ──────────────────────────────────────────────►│struct clk (consumer handle)
│  │  init ──────┐   │  flags: unsigned int               │
│  └─────────────│───┘                                    │
└────────────────│────────────────────────────────────────┘
                 │
                 ▼
    ┌────────────────────────────┐
    │    struct clk_init_data    │
    │                            │
    │  name: "cpu-pll-div2"      │
    │  ops ──────────────────────►  const struct clk_ops (clk_fixed_factor_ops)
    │  parent_data ──────────────►  struct clk_parent_data
    │  num_parents: 1            │     .index = 0  /  .fw_name = "pll"
    │  flags: CLK_SET_RATE_PARENT│
    └────────────────────────────┘

    ┌──────────────────────────────────────────────┐
    │            clk_fixed_factor_ops              │
    │                                              │
    │  .determine_rate = clk_factor_determine_rate │
    │  .set_rate       = clk_factor_set_rate       │
    │  .recalc_rate    = clk_factor_recalc_rate    │
    │  .recalc_accuracy= clk_factor_recalc_accuracy│
    └──────────────────────────────────────────────┘

    ┌──────────────────────────────────────────┐
    │          struct clk_rate_request         │
    │                                          │
    │  rate             ◄── consumer request   │
    │  best_parent_rate ──► parent negotiation │
    │  best_parent_hw ─────► parent clk_hw *  │
    └──────────────────────────────────────────┘
```

---

### Lifecycle Diagram

```
CREATION
────────
  kmalloc(clk_fixed_factor)          [أو devres_alloc لو devm]
       │
       ▼
  fix->mult = mult
  fix->div  = div
  fix->hw.init = &init
  fix->acc  = acc
  fix->flags = fixflags
       │
       ▼
  init.name = name
  init.ops  = &clk_fixed_factor_ops
  init.parent_* = (name | hw | data)
  init.num_parents = 1

REGISTRATION
─────────────
       │
       ├── (dev != NULL)  →  clk_hw_register(dev, hw)
       │                        └► CCF يسجّل ويملا hw->core
       │
       └── (np != NULL)   →  of_clk_hw_register(np, hw)
                               └► CCF يسجّل من الـ DT node

       [لو devm]: devres_add(dev, fix)
                    └► cleanup تلقائي وقت remove

OF PROVIDER (CONFIG_OF)
────────────────────────
  of_clk_add_hw_provider(node, of_clk_hw_simple_get, hw)
       └► يسمح للـ DT consumers يأخدوا الـ clock

USAGE
──────
  consumer calls clk_get()  →  CCF يلاقي hw->core
  consumer calls clk_set_rate(wanted_rate)
       └► CCF يكلّم determine_rate → set_rate → recalc_rate

  consumer calls clk_get_rate()
       └► CCF يكلّم recalc_rate(parent_rate)
            └► return (parent_rate * mult) / div

TEARDOWN
─────────
  [legacy]   clk_unregister_fixed_factor(clk)
               └► clk_unregister(clk)
               └► kfree(fix)

  [hw API]   clk_hw_unregister_fixed_factor(hw)
               └► clk_hw_unregister(hw)
               └► kfree(fix)

  [devm]     device remove → devres_release()
               └► devm_clk_hw_register_fixed_factor_release()
                    └► clk_hw_unregister(&fix->hw)
                    └► devres code يعمل kfree(fix)

  [OF driver] of_fixed_factor_clk_remove()
               └► of_clk_del_provider(node)
               └► clk_hw_unregister_fixed_factor(clk)
```

---

### Call Flow Diagrams

#### حساب الـ Rate (recalc_rate)

```
consumer: clk_get_rate(clk)
  └► CCF: clk_core_get_rate_nolock()
       └► clk_recalc_rate(core, parent_rate)
            └► ops->recalc_rate(hw, parent_rate)
                 └► clk_factor_recalc_rate(hw, parent_rate)
                      │
                      ├── fix = to_clk_fixed_factor(hw)
                      ├── rate = (u64)parent_rate * fix->mult
                      ├── do_div(rate, fix->div)
                      └── return (unsigned long)rate
```

#### طلب تغيير الـ Rate (determine_rate)

```
consumer: clk_set_rate(clk, rate)
  └► CCF: clk_core_determine_round_nolock()
       └► ops->determine_rate(hw, req)
            └► clk_factor_determine_rate(hw, req)
                 │
                 ├── [لو CLK_SET_RATE_PARENT]:
                 │     best_parent = (req->rate / mult) * div
                 │     req->best_parent_rate =
                 │         clk_hw_round_rate(parent_hw, best_parent)
                 │
                 └── req->rate = (req->best_parent_rate / div) * mult
                       [الـ CCF يكمّل ويبعّت للـ parent لو لزم]
```

#### الـ set_rate (nop)

```
CCF: ops->set_rate(hw, rate, parent_rate)
  └► clk_factor_set_rate(hw, rate, parent_rate)
       └── return 0   /* دايمًا — determine_rate ضمن الـ nop */
```

**السبب:** الـ `determine_rate` بتحسب rate بيتوافق مع الـ mult/div الثابتة، فـ `set_rate` مش محتاج يعمل أي حاجة في الـ hardware.

#### حساب الـ Accuracy (recalc_accuracy)

```
CCF: ops->recalc_accuracy(hw, parent_accuracy)
  └► clk_factor_recalc_accuracy(hw, parent_accuracy)
       │
       ├── [لو CLK_FIXED_FACTOR_FIXED_ACCURACY]:
       │     return fix->acc   /* قيمة ثابتة بالـ ppb */
       │
       └── return parent_accuracy  /* يورّث دقة الـ parent */
```

#### التسجيل من الـ DT

```
kernel boot: CLK_OF_DECLARE()
  └► of_fixed_factor_clk_setup(node)
       └► _of_fixed_factor_clk_setup(node)
            ├── of_property_read_u32(node, "clock-div", &div)
            ├── of_property_read_u32(node, "clock-mult", &mult)
            ├── of_property_read_string(node, "clock-output-names", &name)
            └── __clk_hw_register_fixed_factor(NULL, node, ...)
                 └── of_clk_hw_register(np, hw)
                 └── of_clk_add_hw_provider(node, ...)

[لو setup فشل]:
platform driver probe: of_fixed_factor_clk_probe(pdev)
  └── of_node_clear_flag(node, OF_POPULATED)  [في setup]
  └── _of_fixed_factor_clk_setup(pdev->dev.of_node)
  └── platform_set_drvdata(pdev, clk)
```

#### تسجيل باستخدام الـ devm API

```
driver probe:
  devm_clk_hw_register_fixed_factor(dev, name, parent, flags, mult, div)
    └── __clk_hw_register_fixed_factor(..., devm=true)
         ├── devres_alloc(release_fn, sizeof(*fix), GFP_KERNEL)
         ├── [fill struct]
         ├── clk_hw_register(dev, hw)
         └── devres_add(dev, fix)
                └── [مربوط بالـ device lifetime]

driver remove (تلقائي):
  devres_release_all(dev)
    └── devm_clk_hw_register_fixed_factor_release(dev, fix)
         └── clk_hw_unregister(&fix->hw)
         [devres code يعمل kfree(fix) بعدها]
```

---

### استراتيجية الـ Locking

الـ `clk-fixed-factor.c` **مش بيمسك أي lock بنفسه** — ده تصميم مقصود في الـ CCF.

| المستوى | الـ Lock | من يمسكه |
|---------|----------|----------|
| prepare/unprepare | `prepare_lock` (mutex) | الـ CCF نفسه قبل ما يكلّم الـ ops |
| enable/disable | spinlock داخلي في الـ CCF | الـ CCF نفسه |
| rate change | `prepare_lock` | الـ CCF يمسكه طول tree traversal |
| devres | الـ device's devres list lock | `devres_add` / `devres_release` داخليًا |

**نتيجة:** كل الـ ops في `clk_fixed_factor` (determine_rate, set_rate, recalc_rate, recalc_accuracy) بيتنفّذوا وهم محميين بالـ `prepare_lock` بتاع الـ CCF — مش محتاجين locks إضافية لأن:

1. الـ `mult` والـ `div` بيتكتبوا مرة واحدة وقت الإنشاء ومش بيتغيروا.
2. الحسابات read-only بحتة.
3. مفيش shared mutable state.

**ترتيب الـ Locking (في الـ CCF):**
```
prepare_lock (mutex)
  └── enable_lock (spinlock)  [داخل نطاق prepare_lock]
```
الـ fixed factor مش محتاج enable_lock لأنه مش بيعمل gate.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs (Cheatsheet)

#### مجموعة الـ clk_ops Callbacks

| Function | النوع | الغرض |
|---|---|---|
| `clk_factor_recalc_rate` | `static` | يحسب rate الـ clock من rate الـ parent |
| `clk_factor_determine_rate` | `static` | يحدد أفضل rate ممكنة مع أفضل parent rate |
| `clk_factor_set_rate` | `static` | nop — دايمًا ترجع 0 |
| `clk_factor_recalc_accuracy` | `static` | يحسب accuracy للـ clock |

#### مجموعة التسجيل (Registration)

| Function | devm? | Parent Method | Export |
|---|---|---|---|
| `__clk_hw_register_fixed_factor` | internal | كل الطرق | لا |
| `clk_hw_register_fixed_factor` | لا | by name | `EXPORT_SYMBOL_GPL` |
| `clk_hw_register_fixed_factor_parent_hw` | لا | by `clk_hw*` | `EXPORT_SYMBOL_GPL` |
| `clk_hw_register_fixed_factor_fwname` | لا | by fw_name | `EXPORT_SYMBOL_GPL` |
| `clk_hw_register_fixed_factor_index` | لا | by DT index | `EXPORT_SYMBOL_GPL` |
| `clk_hw_register_fixed_factor_with_accuracy_fwname` | لا | fw_name + acc | `EXPORT_SYMBOL_GPL` |
| `clk_register_fixed_factor` | لا | by name (legacy `clk*`) | `EXPORT_SYMBOL_GPL` |
| `devm_clk_hw_register_fixed_factor` | نعم | by name | `EXPORT_SYMBOL_GPL` |
| `devm_clk_hw_register_fixed_factor_index` | نعم | by DT index | `EXPORT_SYMBOL_GPL` |
| `devm_clk_hw_register_fixed_factor_parent_hw` | نعم | by `clk_hw*` | `EXPORT_SYMBOL_GPL` |
| `devm_clk_hw_register_fixed_factor_fwname` | نعم | fw_name | `EXPORT_SYMBOL_GPL` |
| `devm_clk_hw_register_fixed_factor_with_accuracy_fwname` | نعم | fw_name + acc | `EXPORT_SYMBOL_GPL` |

#### مجموعة الـ Cleanup

| Function | الغرض |
|---|---|
| `clk_unregister_fixed_factor` | unregister + kfree (legacy `clk*` API) |
| `clk_hw_unregister_fixed_factor` | unregister + kfree (modern `clk_hw*` API) |
| `devm_clk_hw_register_fixed_factor_release` | devres release callback — unregister بدون kfree |

#### مجموعة الـ OF/DT

| Function | الغرض |
|---|---|
| `_of_fixed_factor_clk_setup` | يقرأ DT node ويسجّل الـ clock |
| `of_fixed_factor_clk_setup` | `__init` wrapper لـ `CLK_OF_DECLARE` |
| `of_fixed_factor_clk_probe` | `platform_driver` probe |
| `of_fixed_factor_clk_remove` | `platform_driver` remove |

---

### المجموعة الأولى: الـ clk_ops Callbacks

الـ `clk_fixed_factor_ops` هو الـ vtable اللي بيربط الـ CCF (Common Clock Framework) بالـ implementation. الـ fixed factor clock مش بتعمل gate وملهاش hardware register — كل شغلها في الحسابات بس.

```c
const struct clk_ops clk_fixed_factor_ops = {
    .determine_rate    = clk_factor_determine_rate,
    .set_rate          = clk_factor_set_rate,
    .recalc_rate       = clk_factor_recalc_rate,
    .recalc_accuracy   = clk_factor_recalc_accuracy,
};
EXPORT_SYMBOL_GPL(clk_fixed_factor_ops);
```

لاحظ إن مفيش `.prepare` / `.enable` / `.set_parent` — الـ CCF بيتعامل مع دول عن طريق الـ parent مباشرة.

---

#### `clk_factor_recalc_rate`

```c
static unsigned long clk_factor_recalc_rate(struct clk_hw *hw,
                                             unsigned long parent_rate)
```

**الـ CCF بيستدعيها** لما يحتاج يعرف الـ rate الحالية للـ clock. بتحسب `rate = parent_rate * mult / div` باستخدام `do_div` لتجنب overflow في الـ 64-bit division.

| Parameter | الوصف |
|---|---|
| `hw` | الـ `clk_hw` embedded في `clk_fixed_factor` |
| `parent_rate` | الـ rate الحالية للـ parent clock بالـ Hz |

**Return:** الـ rate المحسوبة بالـ Hz كـ `unsigned long`.

**Key details:**
- الـ `rate` اتعملتلها cast لـ `unsigned long long` قبل الضرب عشان تتجنب overflow لما `parent_rate * mult` بيتخطى حد الـ 32-bit.
- `do_div(a, b)` بتعمل in-place division وبترجع الـ remainder — الناتج بيتحط في `rate` نفسها.
- مفيش locking هنا — الـ CCF نفسه بيمسك `enable_lock` أو `prepare_lock` حسب السياق قبل ما يستدعي الـ callback.

**Pseudocode:**
```
rate_u64 = (u64)parent_rate * fix->mult
rate_u64 /= fix->div          // do_div in-place
return (unsigned long)rate_u64
```

---

#### `clk_factor_determine_rate`

```c
static int clk_factor_determine_rate(struct clk_hw *hw,
                                      struct clk_rate_request *req)
```

الـ CCF بيستدعيها في مرحلة الـ rate negotiation قبل `set_rate`. الدور إنها تملا `req->rate` و`req->best_parent_rate` بأفضل قيم ممكنة بناءً على الـ constraints.

| Parameter | الوصف |
|---|---|
| `hw` | handle للـ clock الحالي |
| `req` | struct بيحمل الـ `rate` المطلوبة وبيتملى بالـ `best_parent_rate` |

**Return:** دايمًا `0` — مفيش error paths.

**Key details:**
- لو الـ flag `CLK_SET_RATE_PARENT` موجود: بيحسب الـ parent rate المثالي كـ `best_parent = (req->rate / mult) * div`، وبعدين بيستدعي `clk_hw_round_rate` على الـ parent الفعلي عشان يجيب أقرب rate ممكنة.
- في الحالتين بيحسب الـ `req->rate` النهائية من `best_parent_rate` الفعلية: `(best_parent_rate / div) * mult`.
- الحسابات integer truncation — مش round، مش ceil.
- **بدون locking** — same as `recalc_rate`.

**Pseudocode:**
```
if CLK_SET_RATE_PARENT in flags:
    ideal_parent = (req->rate / mult) * div
    req->best_parent_rate = clk_hw_round_rate(parent_hw, ideal_parent)

req->rate = (req->best_parent_rate / div) * mult
return 0
```

---

#### `clk_factor_set_rate`

```c
static int clk_factor_set_rate(struct clk_hw *hw, unsigned long rate,
                                unsigned long parent_rate)
```

الـ CCF بيستدعيها بعد `determine_rate` لتطبيق الـ rate. بما إن الـ fixed factor clock مش بتغير hardware settings، الـ function دي nop خالص.

| Parameter | الوصف |
|---|---|
| `hw` | handle للـ clock |
| `rate` | الـ rate المطلوب (ignored) |
| `parent_rate` | rate الـ parent (ignored) |

**Return:** دايمًا `0`.

**Key details:**
- لازم تتوجد في الـ ops عشان الـ CCF ميرفضش الـ `set_rate` call طالما `determine_rate` موجودة.
- الـ `determine_rate` بتضمن إن الـ rate المطلوبة هي نفس اللي الـ clock هيديها — فلأ في حاجة تتعمل هنا.

---

#### `clk_factor_recalc_accuracy`

```c
static unsigned long clk_factor_recalc_accuracy(struct clk_hw *hw,
                                                  unsigned long parent_accuracy)
```

الـ CCF بيستدعيها لحساب دقة (accuracy) الـ clock — بالـ ppb (parts per billion). لو الـ `CLK_FIXED_FACTOR_FIXED_ACCURACY` flag اتسيت، بيرجع القيمة الـ hardcoded في `fix->acc`، غير كده بيورّث accuracy الـ parent.

| Parameter | الوصف |
|---|---|
| `hw` | handle للـ clock |
| `parent_accuracy` | accuracy الـ parent بالـ ppb |

**Return:** الـ accuracy بالـ ppb كـ `unsigned long`.

**Key details:**
- الـ `fix->flags` هنا هو flags خاصة بالـ `clk_fixed_factor` struct (مش الـ CCF flags) — معرّف بالـ `CLK_FIXED_FACTOR_FIXED_ACCURACY BIT(0)`.
- لو الـ flag مش موجود، الـ clock بيورّث accuracy الـ parent بدون تعديل — ده السلوك الافتراضي.

---

### المجموعة الثانية: الـ Core Internal Registration

#### `__clk_hw_register_fixed_factor`

```c
static struct clk_hw *
__clk_hw_register_fixed_factor(struct device *dev, struct device_node *np,
        const char *name, const char *parent_name,
        const struct clk_hw *parent_hw, const struct clk_parent_data *pdata,
        unsigned long flags, unsigned int mult, unsigned int div,
        unsigned long acc, unsigned int fixflags, bool devm)
```

ده الـ internal workhorse اللي كل الـ public registration functions بتتصل بيه. بيعمل alloc للـ `clk_fixed_factor`، بيملا الـ `clk_init_data`، وبيستدعي الـ register function المناسبة.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المسؤول عن الـ clock (ممكن NULL لو DT-only) |
| `np` | الـ device_node لما الـ register بيتم من DT بدون device |
| `name` | اسم الـ clock في الـ clock tree |
| `parent_name` | اسم الـ parent كـ string (nullable) |
| `parent_hw` | pointer مباشر للـ parent `clk_hw` (nullable) |
| `pdata` | `clk_parent_data` للـ DT-aware parent lookup (nullable) |
| `flags` | الـ CCF flags زي `CLK_SET_RATE_PARENT` |
| `mult` | مضاعف الـ rate |
| `div` | قاسم الـ rate |
| `acc` | الـ accuracy الـ hardcoded (لو `CLK_FIXED_FACTOR_FIXED_ACCURACY` |
| `fixflags` | flags خاصة بالـ `clk_fixed_factor` struct |
| `devm` | `true` لو المطلوب devres-managed allocation |

**Return:** `struct clk_hw *` أو `ERR_PTR` في حالة الفشل.

**Key details:**
- لو `devm=true` و`dev=NULL`: بيرجع `ERR_PTR(-EINVAL)` فورًا.
- الـ allocation: `devres_alloc` للـ devm path، `kmalloc` للـ non-devm.
- بس واحدة من الثلاثة (`parent_name`، `parent_hw`، `pdata`) بتتسيت في `clk_init_data` — priority بالترتيب.
- لو `dev` موجود: بيستخدم `clk_hw_register(dev, hw)`.
- لو مفيش `dev`: بيستخدم `of_clk_hw_register(np, hw)`.
- في حالة فشل الـ register: بيعمل `devres_free` أو `kfree` حسب الـ path.
- في حالة نجاح الـ devm: بيستدعي `devres_add(dev, fix)` عشان يربط الـ cleanup بحياة الـ device.

**Pseudocode:**
```
if devm && !dev: return ERR_PTR(-EINVAL)

if devm:
    fix = devres_alloc(release_fn, sizeof(*fix), GFP_KERNEL)
else:
    fix = kmalloc(sizeof(*fix), GFP_KERNEL)

if !fix: return ERR_PTR(-ENOMEM)

fix->mult = mult; fix->div = div; fix->acc = acc; fix->flags = fixflags
init.name = name; init.ops = &clk_fixed_factor_ops; init.flags = flags
init.num_parents = 1

if parent_name:   init.parent_names = &parent_name
elif parent_hw:   init.parent_hws   = &parent_hw
else:             init.parent_data  = pdata

if dev:   ret = clk_hw_register(dev, hw)
else:     ret = of_clk_hw_register(np, hw)

if ret:
    if devm: devres_free(fix)
    else:    kfree(fix)
    return ERR_PTR(ret)

if devm: devres_add(dev, fix)
return hw
```

---

### المجموعة الثالثة: Public Registration APIs

كل الـ functions دي thin wrappers على `__clk_hw_register_fixed_factor`. الفرق بينهم في طريقة تحديد الـ parent وفي الـ devm behavior.

---

#### `clk_hw_register_fixed_factor`

```c
struct clk_hw *clk_hw_register_fixed_factor(struct device *dev,
        const char *name, const char *parent_name, unsigned long flags,
        unsigned int mult, unsigned int div)
```

الـ API الأساسي للتسجيل بـ parent name كـ string. مناسب لما الـ parent اسمه globally unique في الـ clock tree.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المسجِّل (للـ devres وللـ clk_hw_register) |
| `name` | اسم الـ clock الجديد |
| `parent_name` | اسم الـ parent clock كـ string |
| `flags` | CCF flags |
| `mult` | المضاعف |
| `div` | القاسم |

**Return:** `clk_hw *` أو `ERR_PTR`.

**Who calls it:** drivers بتعرف اسم الـ parent statically في compile time.

---

#### `clk_hw_register_fixed_factor_parent_hw`

```c
struct clk_hw *clk_hw_register_fixed_factor_parent_hw(struct device *dev,
        const char *name, const struct clk_hw *parent_hw,
        unsigned long flags, unsigned int mult, unsigned int div)
```

التسجيل بـ pointer مباشر للـ parent `clk_hw`. أفضل من الـ name-based لما الـ parent والـ child في نفس الـ driver — بيتجنب string lookup ويضمن type safety.

**Key detail:** بيسيت `pdata.index = -1` عشان يمنع الـ CCF من البحث بالـ DT index، والـ parent_hw هو اللي بيتحدد.

---

#### `clk_hw_register_fixed_factor_fwname`

```c
struct clk_hw *clk_hw_register_fixed_factor_fwname(struct device *dev,
        struct device_node *np, const char *name, const char *fw_name,
        unsigned long flags, unsigned int mult, unsigned int div)
```

التسجيل بـ `fw_name` — اسم الـ parent زي ما هو محدد في الـ DT `clock-names` property. ده الأسلوب الـ DT-aware الحديث — بيخلي الـ driver مش محتاج يعرف الاسم الـ global للـ parent.

---

#### `clk_hw_register_fixed_factor_index`

```c
struct clk_hw *clk_hw_register_fixed_factor_index(struct device *dev,
        const char *name, unsigned int index, unsigned long flags,
        unsigned int mult, unsigned int div)
```

التسجيل بـ phandle index في الـ `clocks` property في الـ DT — `index=0` يعني أول clock في الـ list.

---

#### `clk_hw_register_fixed_factor_with_accuracy_fwname`

```c
struct clk_hw *clk_hw_register_fixed_factor_with_accuracy_fwname(struct device *dev,
        struct device_node *np, const char *name, const char *fw_name,
        unsigned long flags, unsigned int mult, unsigned int div,
        unsigned long acc)
```

نفس `clk_hw_register_fixed_factor_fwname` لكن بيسيت `fix->acc` و`CLK_FIXED_FACTOR_FIXED_ACCURACY` — مناسب للـ clocks اللي عندها accuracy معروفة ومستقلة عن الـ parent.

---

#### `clk_register_fixed_factor` (Legacy API)

```c
struct clk *clk_register_fixed_factor(struct device *dev, const char *name,
        const char *parent_name, unsigned long flags,
        unsigned int mult, unsigned int div)
```

الـ legacy API اللي بترجع `struct clk *` بدل `struct clk_hw *`. بتستدعي `clk_hw_register_fixed_factor` وبترجع `hw->clk`.

**Key detail:** الـ `struct clk *` API اتعملتله deprecation لصالح `struct clk_hw *`. الكود الجديد لازم يستخدم `clk_hw_*` variants.

---

### المجموعة الرابعة: الـ devm Registration APIs

الـ `devm_*` variants بتفرق في إنها بتستخدم `devres_alloc` + `devres_add` بدل `kmalloc`. الـ cleanup بيحصل automatically لما الـ device بتتعمل release — مش محتاج manual unregister في الـ error paths.

#### `devm_clk_hw_register_fixed_factor`

```c
struct clk_hw *devm_clk_hw_register_fixed_factor(struct device *dev,
        const char *name, const char *parent_name, unsigned long flags,
        unsigned int mult, unsigned int div)
```

نفس `clk_hw_register_fixed_factor` لكن بـ automatic cleanup.

#### `devm_clk_hw_register_fixed_factor_index`

```c
struct clk_hw *devm_clk_hw_register_fixed_factor_index(struct device *dev,
        const char *name, unsigned int index, unsigned long flags,
        unsigned int mult, unsigned int div)
```

#### `devm_clk_hw_register_fixed_factor_parent_hw`

```c
struct clk_hw *devm_clk_hw_register_fixed_factor_parent_hw(struct device *dev,
        const char *name, const struct clk_hw *parent_hw,
        unsigned long flags, unsigned int mult, unsigned int div)
```

#### `devm_clk_hw_register_fixed_factor_fwname`

```c
struct clk_hw *devm_clk_hw_register_fixed_factor_fwname(struct device *dev,
        struct device_node *np, const char *name, const char *fw_name,
        unsigned long flags, unsigned int mult, unsigned int div)
```

#### `devm_clk_hw_register_fixed_factor_with_accuracy_fwname`

```c
struct clk_hw *devm_clk_hw_register_fixed_factor_with_accuracy_fwname(
        struct device *dev, struct device_node *np, const char *name,
        const char *fw_name, unsigned long flags, unsigned int mult,
        unsigned int div, unsigned long acc)
```

كل الـ devm variants بتمرر `devm=true` لـ `__clk_hw_register_fixed_factor`.

---

### المجموعة الخامسة: الـ Cleanup

#### `devm_clk_hw_register_fixed_factor_release`

```c
static void devm_clk_hw_register_fixed_factor_release(struct device *dev, void *res)
```

الـ devres release callback — الـ devres framework بيستدعيها تلقائيًا لما الـ device بتتـrelease.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device (unused) |
| `res` | pointer للـ `clk_fixed_factor` struct |

**Key detail مهم جدًا:** بتستدعي `clk_hw_unregister` مباشرة بدل `clk_hw_unregister_fixed_factor` — عشان إن الأخيرة بتعمل `kfree` على الـ struct، وده هيعمل double free لأن الـ devres code هو اللي هيعمل `kfree` بعد ما الـ callback تخلص.

---

#### `clk_hw_unregister_fixed_factor`

```c
void clk_hw_unregister_fixed_factor(struct clk_hw *hw)
```

الـ cleanup الأساسي للـ non-devm registrations. بيعمل unregister من الـ CCF وبعدين بيعمل `kfree` للـ `clk_fixed_factor` struct.

| Parameter | الوصف |
|---|---|
| `hw` | الـ `clk_hw` المضمّن في `clk_fixed_factor` |

**Who calls it:** drivers في الـ remove/error paths، وكمان `_of_fixed_factor_clk_setup` في حالة فشل `of_clk_add_hw_provider`.

**Key detail:** الترتيب مهم — أول unregister (عشان الـ CCF يوقف استخدام الـ struct)، بعدين kfree.

---

#### `clk_unregister_fixed_factor` (Legacy)

```c
void clk_unregister_fixed_factor(struct clk *clk)
```

الـ legacy cleanup المقابل لـ `clk_register_fixed_factor`. بيجيب الـ `clk_hw` من الـ `clk` بـ `__clk_get_hw`، بيعمل `clk_unregister` بعدين `kfree`.

---

### المجموعة السادسة: الـ OF/DT Setup

#### `_of_fixed_factor_clk_setup`

```c
static struct clk_hw *_of_fixed_factor_clk_setup(struct device_node *node)
```

الـ internal function اللي بتعمل parse لـ DT node وبتسجل الـ clock. بتتاستدعى من اثنين: `of_fixed_factor_clk_setup` (early init) و`of_fixed_factor_clk_probe` (late probe).

| Parameter | الوصف |
|---|---|
| `node` | الـ device_node اللي فيه `clock-div` و`clock-mult` properties |

**Return:** `clk_hw *` أو `ERR_PTR`.

**Flow:**
```
1. of_property_read_u32(node, "clock-div", &div)
   → فشل: return ERR_PTR(-EIO)

2. of_property_read_u32(node, "clock-mult", &mult)
   → فشل: return ERR_PTR(-EIO)

3. of_property_read_string(node, "clock-output-names", &clk_name)
   → اختياري، fallback لـ node->name

4. __clk_hw_register_fixed_factor(NULL, node, clk_name, ...)
   → فشل: of_node_clear_flag(node, OF_POPULATED) ثم return ERR_CAST

5. of_clk_add_hw_provider(node, of_clk_hw_simple_get, hw)
   → فشل: clk_hw_unregister_fixed_factor(hw) ثم return ERR_PTR
```

**Key detail:** لو التسجيل فشل، بتعمل clear لـ `OF_POPULATED` flag عشان تسمح لـ probe function إنها تحاول تاني لاحقًا — مهم جدًا في سيناريوهات الـ deferred probe.

---

#### `of_fixed_factor_clk_setup`

```c
void __init of_fixed_factor_clk_setup(struct device_node *node)
```

الـ `__init` entry point المسجّل بـ `CLK_OF_DECLARE`. بيتنادى early في الـ boot قبل ما الـ platform drivers تتسجل.

**Who calls it:** الـ OF framework في مرحلة `of_clk_init()`.

```c
CLK_OF_DECLARE(fixed_factor_clk, "fixed-factor-clock", of_fixed_factor_clk_setup);
```

ده بيسجّل الـ function كـ handler لكل DT nodes اللي compatible بـ `"fixed-factor-clock"`.

---

#### `of_fixed_factor_clk_probe`

```c
static int of_fixed_factor_clk_probe(struct platform_device *pdev)
```

الـ platform driver probe — بيتنادى لو الـ early `of_fixed_factor_clk_setup` فشل (مثلًا بسبب `deferred probe` لـ parent clock).

**Flow:**
```
clk = _of_fixed_factor_clk_setup(pdev->dev.of_node)
if IS_ERR(clk): return PTR_ERR(clk)
platform_set_drvdata(pdev, clk)   // احفظ الـ hw* للـ remove
return 0
```

---

#### `of_fixed_factor_clk_remove`

```c
static void of_fixed_factor_clk_remove(struct platform_device *pdev)
```

الـ platform driver cleanup.

**Flow:**
```
clk = platform_get_drvdata(pdev)
of_clk_del_provider(pdev->dev.of_node)   // ازيل من الـ OF provider list
clk_hw_unregister_fixed_factor(clk)      // unregister + kfree
```

**Key detail:** لازم `of_clk_del_provider` يتعمل قبل `clk_hw_unregister_fixed_factor` عشان تمنع الـ CCF من استخدام الـ hw pointer بعد ما يتحرر.

---

### العلاقة بين الـ Functions (Architecture Diagram)

```
                    ┌─────────────────────────────────────────────┐
                    │         Public Registration APIs            │
                    │                                             │
                    │  clk_hw_register_fixed_factor               │
                    │  clk_hw_register_fixed_factor_parent_hw     │
                    │  clk_hw_register_fixed_factor_fwname        │
                    │  clk_hw_register_fixed_factor_index         │
                    │  clk_hw_register_fixed_factor_with_accuracy │
                    │  devm_clk_hw_register_fixed_factor_*        │
                    └──────────────────────┬──────────────────────┘
                                           │ all call
                                           ▼
                    ┌─────────────────────────────────────────────┐
                    │    __clk_hw_register_fixed_factor()         │
                    │   (alloc + fill init_data + register)       │
                    └──────────┬──────────────────┬──────────────┘
                               │                  │
                     dev != NULL               dev == NULL
                               │                  │
                    clk_hw_register()   of_clk_hw_register()
                               │                  │
                               └────────┬─────────┘
                                        │
                                        ▼
                              CCF Clock Tree
                                        │
                              ┌─────────┴──────────┐
                              │  clk_fixed_factor_ops│
                              │  .recalc_rate        │
                              │  .determine_rate     │
                              │  .set_rate (nop)     │
                              │  .recalc_accuracy    │
                              └─────────────────────┘
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ fixed-factor clock

الـ **CCF (Common Clock Framework)** بيعمل تلقائيًا directory لكل clock مسجّل تحت:

```
/sys/kernel/debug/clk/<clock-name>/
```

المدخلات المهمة جوّاه:

| المدخل | المعنى | أمر القراءة |
|--------|---------|-------------|
| `clk_rate` | الـ rate الفعلي بالـ Hz | `cat /sys/kernel/debug/clk/<name>/clk_rate` |
| `clk_parent` | اسم الـ parent | `cat /sys/kernel/debug/clk/<name>/clk_parent` |
| `clk_prepare_count` | عدد مرات الـ prepare | `cat /sys/kernel/debug/clk/<name>/clk_prepare_count` |
| `clk_enable_count` | عدد مرات الـ enable | `cat /sys/kernel/debug/clk/<name>/clk_enable_count` |
| `clk_flags` | الـ flags المفعّلة | `cat /sys/kernel/debug/clk/<name>/clk_flags` |
| `clk_accuracy` | الدقة بالـ ppb | `cat /sys/kernel/debug/clk/<name>/clk_accuracy` |

لمشاهدة شجرة الـ clocks كاملة:

```bash
# شجرة كاملة بالـ rates
cat /sys/kernel/debug/clk/clk_summary

# شجرة مضغوطة
cat /sys/kernel/debug/clk/clk_dump
```

مثال على output من `clk_summary`:

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 osc24M                             1       1        0    24000000          0     0  50000
    pll_cpu                         1       1        0   816000000          0     0  50000
       cpu_div1                     1       1        0   408000000          0     0  50000
```

**الـ fixed-factor** بيظهر هنا بـ rate = parent_rate * mult / div، لو الرقم غلط → ابدأ من هنا.

#### 2. مدخلات الـ sysfs

الـ sysfs للـ clocks بيكون على مستوى الـ device مش الـ clock مباشرة:

```bash
# لو الـ driver جه من DT (platform_device)
ls /sys/bus/platform/devices/of_fixed_factor_clk*/

# تحقق من الـ device tree node اللي ربط بيه
cat /sys/bus/platform/devices/<dev>/of_node/clock-mult
cat /sys/bus/platform/devices/<dev>/of_node/clock-div
```

لو الـ clock متسجّل كـ provider:

```bash
# شوف كل الـ clocks اللي بيستخدم الـ fixed-factor كـ parent
grep -r "fixed-factor-clock" /sys/firmware/devicetree/base/ 2>/dev/null
```

#### 3. الـ ftrace — tracepoints وأحداث الـ clocks

**الـ CCF بيوفر tracepoints** جاهزة تحت `clk` subsystem:

```bash
# اعرض كل الـ events المتاحة
ls /sys/kernel/debug/tracing/events/clk/

# الـ events المهمة للـ fixed-factor
cat /sys/kernel/debug/tracing/events/clk/clk_set_rate/format
cat /sys/kernel/debug/tracing/events/clk/clk_set_rate_complete/format
```

تفعيل الـ tracing لمتابعة rate changes:

```bash
# فعّل الـ events المطلوبة
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate_complete/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_prepare/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ workload (أو فقط انتظر)

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

مثال output:

```
     kworker/0:1-45    [000] ....  1234.567890: clk_set_rate: cpu_div1 rate=204000000
     kworker/0:1-45    [000] ....  1234.567920: clk_set_rate_complete: cpu_div1 ret=0
```

**الـ fixed-factor** مش بيعمل `set_rate` حقيقي (الـ `clk_factor_set_rate` بترجع 0 دايمًا)، فلو شفت `clk_set_rate` عليه → الـ parent هو اللي اتغيّر فعلًا (لو `CLK_SET_RATE_PARENT` مفعّل).

function graph tracing لمتابعة `recalc_rate`:

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo clk_factor_recalc_rate > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

#### 4. الـ printk والـ dynamic debug

تفعيل الـ dynamic debug لـ clk subsystem:

```bash
# فعّل كل messages في drivers/clk/
echo "file drivers/clk/clk-fixed-factor.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ clk subsystem
echo "module clk_fixed_factor +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ control
grep "clk-fixed-factor" /sys/kernel/debug/dynamic_debug/control
```

لو عندك kernel مبني بنفسك، ممكن تضيف `pr_debug()` مؤقتًا:

```c
static unsigned long clk_factor_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    struct clk_fixed_factor *fix = to_clk_fixed_factor(hw);
    unsigned long long int rate;

    rate = (unsigned long long int)parent_rate * fix->mult;
    do_div(rate, fix->div);

    /* debug: print mult/div/result */
    pr_debug("%s: parent=%lu mult=%u div=%u rate=%lu\n",
             clk_hw_get_name(hw), parent_rate,
             fix->mult, fix->div, (unsigned long)rate);

    return (unsigned long)rate;
}
```

وتفعيله بـ:

```bash
echo "file drivers/clk/clk-fixed-factor.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w
```

#### 5. الـ Kernel Config Options للـ debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_COMMON_CLK` | الأساس، لازم يكون enabled |
| `CONFIG_CLKDEV_LOOKUP` | تمكين lookup الـ clocks بالاسم |
| `CONFIG_CLK_DEBUG` | تفعيل debugfs entries للـ clocks |
| `CONFIG_DEBUG_FS` | لازم لـ debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | للـ pr_debug() activation |
| `CONFIG_TRACING` | للـ ftrace |
| `CONFIG_FUNCTION_TRACER` | لـ function-level tracing |
| `CONFIG_KALLSYMS` | لتظهر أسماء الـ functions في traces |
| `CONFIG_DEBUG_OBJECTS` | كشف use-after-free في الـ objects |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(COMMON_CLK|CLK_DEBUG|DEBUG_FS|DYNAMIC_DEBUG)"
```

#### 6. أدوات خاصة بالـ subsystem

**أداة `clk_summary`** (أهم أداة):

```bash
# طباعة ملخص شامل بالـ rates والـ enable counts
cat /sys/kernel/debug/clk/clk_summary | column -t
```

**سكريبت Python لتحليل clk_summary:**

```bash
# فلتر الـ clocks اللي rate = 0 (مشكلة محتملة)
awk 'NR>3 && $6==0 {print}' /sys/kernel/debug/clk/clk_summary
```

**أداة `of_dump_addr`** للـ DT:

```bash
# dump كل الـ fixed-factor nodes من الـ DT
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "fixed-factor-clock" {} \; 2>/dev/null
```

**لو عندك `devmem2` وعارف الـ PLL registers:**

```bash
# مثال على قراءة register
devmem2 0x01C20004 w  # عنوان افتراضي، حسب الـ SoC
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|---------|-------|
| `Fixed factor clock <name> must have a clock-div property` | الـ DT node ناقصه `clock-div` | أضف `clock-div = <N>;` في الـ DT |
| `Fixed factor clock <name> must have a clock-mult property` | الـ DT node ناقصه `clock-mult` | أضف `clock-mult = <N>;` في الـ DT |
| `clk_hw_register: failed to register clock <name>` | فشل التسجيل (اسم مكرر أو parent مش موجود) | تحقق من uniqueness الاسم واكتمال الـ parent |
| `clk: couldn't get clock <name>` | الـ consumer مش لاقي الـ clock | تحقق من الـ DT `clocks` phandle |
| `-EINVAL` من `__clk_hw_register_fixed_factor` | `devm=true` بدون `dev` | مش مفروض يحصل، مشكلة في الـ driver caller |
| `-ENOMEM` | فشل الـ kmalloc/devres_alloc | نقص ذاكرة، راجع `/proc/meminfo` |
| `-EIO` | `of_property_read_u32` فشل | الـ DT مكتوب غلط، تحقق من `dtc -I dtb -O dts` |
| `of_clk_add_hw_provider: failed` | فشل تسجيل الـ clock كـ provider | parent node مش موجود أو مش prepared |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
static unsigned long clk_factor_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    struct clk_fixed_factor *fix = to_clk_fixed_factor(hw);
    unsigned long long int rate;

    /* WARN لو div = 0 → division by zero */
    if (WARN_ON(fix->div == 0))
        return 0;

    /* WARN لو mult = 0 → rate هيبقى صفر دايمًا */
    WARN_ON(fix->mult == 0);

    rate = (unsigned long long int)parent_rate * fix->mult;
    do_div(rate, fix->div);
    return (unsigned long)rate;
}

static struct clk_hw *
__clk_hw_register_fixed_factor(struct device *dev, struct device_node *np,
        ...)
{
    /* WARN لو اتسجل بدون اسم */
    if (WARN_ON(!name))
        return ERR_PTR(-EINVAL);

    /* WARN لو div = 0 وقت التسجيل */
    if (WARN_ON(div == 0))
        return ERR_PTR(-EINVAL);
    ...
}
```

نقطة `dump_stack()` مفيدة في `devm_clk_hw_register_fixed_factor_release` لو الـ unregister اتعمل في وقت غير متوقع:

```c
static void devm_clk_hw_register_fixed_factor_release(struct device *dev, void *res)
{
    struct clk_fixed_factor *fix = res;
    pr_debug("releasing fixed-factor clock, called from:\n");
    dump_stack();  /* مؤقت للـ debugging فقط */
    clk_hw_unregister(&fix->hw);
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state بيطابق الـ kernel state

الـ **fixed-factor clock** مافيش ليه registers حقيقية — هو مجرد حساب رياضي. لكن الـ parent بتاعه (عادةً PLL أو oscillator) عنده registers.

خطوات التحقق:
1. اقرأ rate الـ parent من debugfs
2. احسب يدويًا: `rate = parent_rate * mult / div`
3. قارن بـ `cat /sys/kernel/debug/clk/<fixed-factor-name>/clk_rate`

```bash
# مثال: parent = 24MHz, mult=2, div=1 → expected = 48MHz
PARENT=$(cat /sys/kernel/debug/clk/osc24M/clk_rate)
MULT=2
DIV=1
echo "Expected: $(( PARENT * MULT / DIV )) Hz"
cat /sys/kernel/debug/clk/my_fixed_factor/clk_rate
```

#### 2. تقنيات الـ Register Dump

لما الـ parent هو PLL وعايز تتحقق منه مباشرة:

```bash
# باستخدام devmem2 (لازم root + CONFIG_STRICT_DEVMEM=n)
devmem2 <PLL_BASE_ADDR> w

# مثال على Allwinner A64 - PLL_CPUX
devmem2 0x01C20000 w

# باستخدام /dev/mem
python3 -c "
import mmap, struct, os
fd = os.open('/dev/mem', os.O_RDONLY)
m = mmap.mmap(fd, 4096, offset=0x01C20000)
val = struct.unpack('<I', m[:4])[0]
print(hex(val))
m.close()
os.close(fd)
"
```

لو `CONFIG_STRICT_DEVMEM=y`، استخدم **io** من package `busybox`:

```bash
io -4 -r 0x01C20000
```

أو عمل kernel module بسيط يقرأ الـ register بـ `ioremap`.

#### 3. Logic Analyzer / Oscilloscope

الـ fixed-factor نفسه مش بيغيّر إشارة فيزيائية — هو فقط حساب. لكن لو شاك في الـ parent:

**نقاط القياس:**
- قيس تردد الـ signal عند pin الـ parent (XTAL/PLL output)
- قارن بـ `cat /sys/kernel/debug/clk/<parent>/clk_rate`

**Logic Analyzer tips:**
```
- ضبّط الـ trigger على rising edge
- sample rate لازم ≥ 10× الـ clock المتوقع
- مثال: clock 48MHz → sample rate ≥ 480MSa/s
- استخدم FFT mode لو التردد مش واضح
```

**Oscilloscope:**
```
- استخدم probe ×10 لتقليل الـ capacitive loading
- لو الـ signal رديء → احتمال مشكلة في الـ termination أو trace length
```

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ kernel log

| المشكلة الفيزيائية | نمط الـ log | التحقق |
|--------------------|-------------|---------|
| XTAL مش بيشتغل | `clk: couldn't get clock osc24M` | قيس تردد الـ XTAL بالـ oscilloscope |
| PLL unstable | `clk: failed to set rate` أو rate خاطئ في clk_summary | تحقق من الـ power supply لوحدة الـ PLL (VDD_PLL) |
| clock tree مش initialized | `clk_get: osc 24MHz: Invalid argument` | تحقق من ترتيب الـ initialization في الـ DT |
| الـ parent مش enabled قبل الـ child | `prepare_enable failed` | شوف `clk_prepare_count` للـ parent |
| voltage منخفض على الـ SoC | rate صح بس جهاز unstable | قيس VDD_CORE |

#### 5. الـ Device Tree Debugging

تحقق إن الـ DT بيطابق الـ hardware:

```bash
# dump الـ compiled DTB كـ text مقروء
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A10 "fixed-factor-clock"
```

مثال DT صح:

```dts
cpu_div: cpu-div {
    compatible = "fixed-factor-clock";
    clocks = <&pll_cpu>;        /* parent clock phandle */
    clock-div = <2>;            /* مقسوم على 2 */
    clock-mult = <1>;           /* مضروب في 1 */
    #clock-cells = <0>;
    clock-output-names = "cpu-div";
};
```

أخطاء شائعة في الـ DT:

```bash
# تحقق إن clock-div موجود
fdtget /sys/firmware/fdt /clocks/cpu-div clock-div

# تحقق إن الـ parent phandle صح
fdtget /sys/firmware/fdt /clocks/cpu-div clocks
```

تحقق من الـ OF flags:

```bash
# شوف الـ nodes اللي اتعمل لها OF_POPULATED
cat /sys/kernel/debug/clk/clk_summary | grep cpu-div
```

---

### Practical Commands

#### أوامر جاهزة للـ Copy-Paste

```bash
# === 1. عرض كل الـ clocks وحالتها ===
cat /sys/kernel/debug/clk/clk_summary

# === 2. البحث عن clock معين ===
grep "my_clock_name" /sys/kernel/debug/clk/clk_summary

# === 3. قراءة rate clock معين ===
cat /sys/kernel/debug/clk/cpu_div/clk_rate

# === 4. قراءة parent الـ clock ===
cat /sys/kernel/debug/clk/cpu_div/clk_parent

# === 5. قراءة flags ===
cat /sys/kernel/debug/clk/cpu_div/clk_flags

# === 6. تفعيل dynamic debug للـ clk-fixed-factor ===
echo "file drivers/clk/clk-fixed-factor.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &

# === 7. trace كل rate changes ===
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
cat /sys/kernel/debug/tracing/trace | grep -i "fixed\|div\|mult"
echo 0 > /sys/kernel/debug/tracing/tracing_on

# === 8. تحقق من الـ DT nodes ===
find /sys/firmware/devicetree/base -name "compatible" | \
  xargs grep -l "fixed-factor-clock" 2>/dev/null | \
  while read f; do
    dir=$(dirname "$f")
    echo "=== $dir ==="
    cat "$dir/clock-mult" 2>/dev/null | xxd
    cat "$dir/clock-div" 2>/dev/null | xxd
  done

# === 9. حساب الـ rate اليدوي ومقارنته ===
CLK_NAME="cpu_div"
PARENT_RATE=$(cat /sys/kernel/debug/clk/$(cat /sys/kernel/debug/clk/$CLK_NAME/clk_parent)/clk_rate 2>/dev/null)
ACTUAL_RATE=$(cat /sys/kernel/debug/clk/$CLK_NAME/clk_rate)
echo "Parent rate: $PARENT_RATE"
echo "Actual rate: $ACTUAL_RATE"

# === 10. مشاهدة kernel messages المتعلقة بالـ clk ===
dmesg | grep -iE "clk|clock|fixed.factor" | tail -50

# === 11. تحقق من الـ enable/prepare counts ===
for f in /sys/kernel/debug/clk/*/clk_enable_count; do
  cnt=$(cat "$f")
  name=$(echo "$f" | cut -d/ -f6)
  [ "$cnt" -gt 0 ] && echo "$name: enable_count=$cnt"
done
```

#### تفسير الـ Output

**مثال clk_summary output:**

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 osc24M                             2       2        0    24000000          0     0  50000
    pll_cpu                         1       1        0   816000000          0     0  50000
       cpu_div                      1       1        0   408000000          0     0  50000
```

**كيف تقرأه:**
- `enable_cnt=0` وـ`prepare_cnt=0` للـ clock اللي المفروض يشتغل = **مشكلة** (consumer مش عامل `clk_prepare_enable`)
- `rate=0` = الـ parent مش enabled أو الـ `recalc_rate` فشل
- `rate` ≠ `parent_rate * mult / div` = **bug في الـ mult أو div** أو overflow في الحساب

**مثال trace output:**

```
   systemd-1      [000] ....  123.456: clk_set_rate: cpu_div rate=204000000
   systemd-1      [000] ....  123.457: clk_set_rate_complete: cpu_div ret=0
```

- `ret=0` = نجح، الـ rate اتغيّر عند الـ parent (لو `CLK_SET_RATE_PARENT` set)
- `ret=-EINVAL` = الـ rate مش ممكن (نادر في fixed-factor لأن `set_rate` دايمًا ترجع 0)
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على Industrial Gateway بـ RK3562

#### العنوان
**الـ UART بيديه baud rate غلط على Industrial Gateway بـ RK3562**

#### السياق
شركة بتبني industrial gateway بـ RK3562 للتواصل مع sensors عن طريق RS-485. الـ UART المربوط بـ RS-485 transceiver محتاج 115200 bps بالظبط. الـ SoC بيوفر `uart_clk` اللي مصدره `cpll` عن طريق fixed-factor clock اسمها `cpll_uart` بـ `mult=1, div=6`.

#### المشكلة
بعد boot، الـ UART بيشتغل بـ baud rate مش صح — القراءة من الـ sensors بتيجي corrupted. الـ engineer يعمل:

```bash
cat /sys/kernel/debug/clk/cpll_uart/clk_rate
# Output: 99999999   (بدل 100000000)
```

الفرق صغير بس كافي يكسر الـ UART framing مع devices تانية.

#### التحليل
الـ `clk_factor_recalc_rate` في الملف بتحسب:

```c
static unsigned long clk_factor_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    struct clk_fixed_factor *fix = to_clk_fixed_factor(hw);
    unsigned long long int rate;

    rate = (unsigned long long int)parent_rate * fix->mult;
    do_div(rate, fix->div);
    return (unsigned long)rate;
}
```

الـ `cpll` rate اتحدد في DT كـ 600000000 Hz. لكن عند `do_div(600000000 * 1, 6)` النتيجة صح = 100000000.

المشكلة مش في الـ fixed-factor نفسها — المشكلة أن الـ `cpll` parent rate اتسجل غلط في DT كـ 599999994 Hz (قيمة قديمة من vendor BSP). لما `do_div` بتقسم على 6 النتيجة بتيجي `99999999`.

الـ `clk_factor_determine_rate` كمان بتحسب:

```c
req->rate = (req->best_parent_rate / fix->div) * fix->mult;
```

لو الـ `best_parent_rate` مش مضروب في `div` بالظبط، التقريب هيفضل يرجع قيمة غلط.

#### الحل

**أولاً:** تصحيح قيمة الـ `cpll` في DT:

```dts
/* في rk3562.dtsi */
cpll: clock-cpll {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <600000000>; /* صحح من 599999994 */
};
```

**ثانياً:** تأكيد بعد boot:

```bash
cat /sys/kernel/debug/clk/cpll/clk_rate
# يجب: 600000000

cat /sys/kernel/debug/clk/cpll_uart/clk_rate
# يجب: 100000000
```

**ثالثاً:** لو الـ PLL فعلاً مش بيديك 600 MHz بالظبط وعندك drift، استخدم `clk_hw_register_fixed_factor_with_accuracy_fwname` وحدد accuracy معقولة:

```c
hw = clk_hw_register_fixed_factor_with_accuracy_fwname(
    dev, np, "cpll_uart", "cpll",
    0, 1, 6,
    500000 /* 500 ppb accuracy */
);
```

ده بيخلي الـ CCF يعرف الـ clock مش perfect وبياخد ده في الاعتبار.

#### الدرس المستفاد
الـ `clk_factor_recalc_rate` بتحسب بـ integer division — أي خطأ في الـ parent rate بينتشر للـ children. دايماً verify الـ parent rate في debugfs قبل تعديل الـ child dividers.

---

### السيناريو الثاني: SPI Flash مش بيشتغل على STM32MP1 IoT Sensor

#### العنوان
**الـ SPI clock بيتجاوز الـ max frequency المسموح بيها على STM32MP1**

#### السياق
IoT sensor node بـ STM32MP1 بيستخدم SPI flash (W25Q128) بأقصى سرعة 104 MHz. الـ engineer عامل fixed-factor clock `spi_clk` بـ `mult=1, div=2` من `pll4_r` اللي قيمته 209 MHz.

#### المشكلة
الـ SPI flash بيتعمله corruption أحياناً وبتحصل read errors. الـ engineer بيشك في الـ clock frequency.

```bash
cat /sys/kernel/debug/clk/spi_clk/clk_rate
# Output: 104500000   (أكتر من 104 MHz!)
```

#### التحليل
الـ `pll4_r` في الـ STM32MP1 بيديه 209 MHz مش 208 MHz. الـ fixed-factor clock بتحسب:

```c
rate = (unsigned long long int)209000000 * 1;
do_div(rate, 2);
// result = 104500000 Hz
```

الناتج 104.5 MHz — بيتجاوز الـ W25Q128 limit بـ 500 KHz.

الـ `clk_factor_determine_rate` لو اتنادت بـ target = 104 MHz:

```c
best_parent = (req->rate / fix->mult) * fix->div;
// best_parent = (104000000 / 1) * 2 = 208000000

req->best_parent_rate = clk_hw_round_rate(parent_hw, 208000000);
// لو الـ PLL مش بيقدر يديك 208 بالظبط — بيديك 209

req->rate = (209000000 / 2) * 1 = 104500000;
```

الـ framework حسب rate وعدّلها لأعلى لأن مفيش `CLK_SET_RATE_PARENT` في الـ flags.

#### الحل

**الحل الأول:** تغيير الـ `div` لـ 3 وقبول rate أقل (69.6 MHz):

```dts
spi_clk: spi-clock {
    compatible = "fixed-factor-clock";
    clocks = <&pll4_r>;
    #clock-cells = <0>;
    clock-div = <3>;   /* بدل 2 */
    clock-mult = <1>;
};
```

**الحل الثاني (الأفضل):** تفعيل `CLK_SET_RATE_PARENT` وترك الـ framework يضبط الـ PLL:

```c
hw = clk_hw_register_fixed_factor(dev, "spi_clk", "pll4_r",
    CLK_SET_RATE_PARENT, /* بيسمح بتعديل الـ parent */
    1, 2);
```

لما `CLK_SET_RATE_PARENT` موجود، `clk_factor_determine_rate` بتحسب:

```c
best_parent = (104000000 / 1) * 2 = 208000000;
req->best_parent_rate = clk_hw_round_rate(parent, 208000000);
// PLL بيحاول يديك 208 MHz فعلاً
req->rate = (208000000 / 2) * 1 = 104000000; // exactly!
```

#### الدرس المستفاد
الـ fixed-factor clock مش بتعدل الـ parent rate من تلقاء نفسها — محتاج `CLK_SET_RATE_PARENT` صراحةً. وإلا النتيجة هي أقرب rate ممكنة من الـ parent الحالي حتى لو تجاوزت الـ limit.

---

### السيناريو الثالث: HDMI مش بيشتغل على Android TV Box بـ Allwinner H616

#### العنوان
**الـ HDMI output مش بيظهر على Android TV Box بـ Allwinner H616**

#### السياق
Android TV box بـ Allwinner H616 مش بيشوفه التلفزيون على HDMI. الـ display pipeline محتاج `hdmi_clk` بـ 148.5 MHz عشان 1080p@60Hz. الـ clock مشتق من `video_pll` عن طريق fixed-factor بـ `mult=1, div=4`.

#### المشكلة
الـ board بتـ boot بس الشاشة سودا. الـ logs بتقول:

```
[    3.241] sunxi-drm: HDMI clock rate mismatch: got 148000000, need 148500000
```

#### التحليل
الـ `video_pll` اتضبط على 594 MHz. الـ fixed-factor:

```c
rate = (unsigned long long int)594000000 * 1;
do_div(rate, 4);
// result = 148500000 Hz  ✓
```

ده صح. إذن المشكلة في مكان تاني — الـ DT الخاص بالـ fixed-factor مكتوب غلط:

```dts
/* الغلط في DT */
hdmi_clk: hdmi-clock {
    compatible = "fixed-factor-clock";
    clocks = <&video_pll>;
    #clock-cells = <0>;
    clock-div = <4>;
    clock-mult = <0>;  /* BUG: mult = 0 ! */
};
```

الـ `_of_fixed_factor_clk_setup` بتقرأ:

```c
if (of_property_read_u32(node, "clock-mult", &mult)) {
    pr_err("... must have a clock-mult property\n");
    return ERR_PTR(-EIO);
}
```

القراءة نجحت (القيمة موجودة) بس القيمة = 0، فـ `clk_factor_recalc_rate` بتحسب:

```c
rate = 594000000ULL * 0;  // = 0
do_div(rate, 4);
// result = 0 Hz
```

الـ clock بيتسجل بـ rate = 0 وبالتالي HDMI pipeline بيفشل.

#### الحل

**تصحيح الـ DT:**

```dts
hdmi_clk: hdmi-clock {
    compatible = "fixed-factor-clock";
    clocks = <&video_pll>;
    #clock-cells = <0>;
    clock-div = <4>;
    clock-mult = <1>;  /* الصح */
};
```

**Verification:**

```bash
cat /sys/kernel/debug/clk/hdmi_clk/clk_rate
# يجب: 148500000
```

**نصيحة للـ driver developers:** لو بتستخدم الـ API مباشرةً في C، أضف validation:

```c
if (mult == 0) {
    dev_err(dev, "fixed-factor mult cannot be zero!\n");
    return ERR_PTR(-EINVAL);
}
hw = clk_hw_register_fixed_factor(dev, "hdmi_clk", "video_pll", 0, mult, div);
```

#### الدرس المستفاد
الـ kernel مش بيعمل validate إن `mult != 0` — الـ `_of_fixed_factor_clk_setup` بتقبل أي قيمة. دايماً تأكد من الـ DT properties يدوياً لو الـ clock rate جت 0 في debugfs.

---

### السيناريو الرابع: I2C Timeout على Custom Board بـ i.MX8M Plus

#### العنوان
**الـ I2C transactions بتـ timeout بعد board bring-up على i.MX8M Plus**

#### السياق
custom board للـ automotive ECU بيستخدم i.MX8M Plus. الـ I2C بيتكلم مع PMIC وـ IMU sensors. بعد bring-up مباشرةً، الـ I2C بتبدأ timeouts وبيـ log:

```
i2c_imx 30a20000.i2c: timeout in wait_bus_not_busy
```

#### المشكلة
الـ engineer بيفحص الـ clock tree ولاقى إن `i2c3_clk` مش متسجل صح.

```bash
cat /sys/kernel/debug/clk/i2c3_clk/clk_rate
# Output: 0
```

#### التحليل
الـ custom clock driver بيستخدم `devm_clk_hw_register_fixed_factor_index` عشان يربط الـ I2C clock بـ parent من DT index:

```c
hw = devm_clk_hw_register_fixed_factor_index(dev, "i2c3_clk",
    2,      /* index في clocks property */
    0,      /* flags */
    1, 4);  /* mult=1, div=4 */
```

الكود بيبني `clk_parent_data`:

```c
/* من الملف */
struct clk_hw *devm_clk_hw_register_fixed_factor_index(struct device *dev,
        const char *name, unsigned int index, unsigned long flags,
        unsigned int mult, unsigned int div)
{
    const struct clk_parent_data pdata = { .index = index };

    return __clk_hw_register_fixed_factor(dev, NULL, name, NULL, NULL, &pdata,
                                          flags, mult, div, 0, 0, true);
}
```

الـ index = 2 بس الـ DT الخاص بالـ device عنده بس entry واحدة في `clocks`:

```dts
/* الغلط */
&i2c3 {
    clocks = <&clk IMX8MP_CLK_I2C3>;  /* index 0 فقط */
};
```

الـ `__clk_hw_register_fixed_factor` بتمرر الـ `pdata` لـ `clk_hw_register` اللي بيفشل لأن الـ index 2 مش موجود. الـ `ret` بتيجي error والـ `hw = ERR_PTR(ret)` — بس الكود اللي فوق مش بيتحقق من الـ IS_ERR وبيحاول يستخدم الـ clock.

#### الحل

**أولاً:** تصحيح الـ index في الكود:

```c
/* index = 0 عشان أول وبس clock في DT */
hw = devm_clk_hw_register_fixed_factor_index(dev, "i2c3_clk",
    0, 0, 1, 4);
if (IS_ERR(hw)) {
    dev_err(dev, "failed to register i2c3_clk: %ld\n", PTR_ERR(hw));
    return PTR_ERR(hw);
}
```

**ثانياً:** أو استخدم `devm_clk_hw_register_fixed_factor_fwname` بـ fw_name بدل index عشان أوضح:

```c
hw = devm_clk_hw_register_fixed_factor_fwname(dev, dev->of_node,
    "i2c3_clk", "i2c3_root_clk",
    0, 1, 4);
```

مع DT:

```dts
&custom_clk_ctrl {
    clocks = <&clk IMX8MP_CLK_I2C3_ROOT>;
    clock-names = "i2c3_root_clk";
};
```

**Debug:**

```bash
# تأكيد الـ parent صح
cat /sys/kernel/debug/clk/i2c3_clk/clk_parent
cat /sys/kernel/debug/clk/i2c3_clk/clk_rate
```

#### الدرس المستفاد
الـ `devm_clk_hw_register_fixed_factor_index` بتستخدم `clk_parent_data.index` اللي بيتوافق مع ترتيب الـ `clocks` property في DT — مش global index. دايماً فضل `fw_name` على `index` في custom drivers عشان أقل عرضة للأخطاء.

---

### السيناريو الخامس: Memory Controller Instability على AM62x Industrial Gateway

#### العنوان
**الـ DDR memory controller بيعمل crashes متقطعة على AM62x عند load كبير**

#### السياق
industrial gateway بـ TI AM62x بيجمع بيانات من 8 Modbus sensors بالتوازي. بيعمل crash عشوائي تحت load مع:

```
Unhandled fault: external abort on non-linefetch
```

الـ engineer بيشك في الـ DDR clock stability.

#### المشكلة
الـ vendor BSP بيستخدم custom fixed-factor clock `ddr_clk` مشتق من `ddrpll` بـ `mult=1, div=2`. الـ `ddr_clk` المفروض يديه 800 MHz.

```bash
cat /sys/kernel/debug/clk/ddr_clk/clk_rate
# Output: 800000000  — يبدو صح!
```

بس الـ crashes بتستمر.

#### التحليل
الـ engineer بيفحص الـ accuracy:

```bash
cat /sys/kernel/debug/clk/ddr_clk/clk_accuracy
# Output: 0  (يعني "perfect" — مش صح لـ PLL حقيقي)

cat /sys/kernel/debug/clk/ddrpll/clk_accuracy
# Output: 300  (300 ppb)
```

الـ `clk_factor_recalc_accuracy` في الملف:

```c
static unsigned long clk_factor_recalc_accuracy(struct clk_hw *hw,
                        unsigned long parent_accuracy)
{
    struct clk_fixed_factor *fix = to_clk_fixed_factor(hw);

    if (fix->flags & CLK_FIXED_FACTOR_FIXED_ACCURACY)
        return fix->acc;

    return parent_accuracy;
}
```

الـ `ddr_clk` مسجل بـ:

```c
/* الكود في vendor BSP */
hw = clk_hw_register_fixed_factor(dev, "ddr_clk", "ddrpll",
    0, 1, 2);
/* مفيش CLK_FIXED_FACTOR_FIXED_ACCURACY */
```

إذن `clk_factor_recalc_accuracy` المفروض يرجع `parent_accuracy = 300 ppb` — بس الـ debugfs بيقول 0.

اتضح إن الـ vendor BSP بيستخدم نسخة قديمة من الـ kernel مفيهاش `clk_factor_recalc_accuracy` كـ op، فالـ accuracy بتتحسب كـ 0 في الـ CCF framework بدون الـ callback.

الحقيقي إن الـ ddrpll عنده 300 ppb drift — يعني 800 MHz ± 240 Hz. ده مش هو سبب الـ crash بالضرورة.

الـ engineer بيتعمق أكتر ويلاقي المشكلة الحقيقية: الـ `CLK_SET_RATE_PARENT` مش موجود، فلما الـ system بيحاول يعمل `clk_set_rate` على `ddr_clk` بقيمة 800 MHz، الـ `clk_factor_determine_rate` بتحسب:

```c
/* CLK_SET_RATE_PARENT مش set */
req->rate = (req->best_parent_rate / fix->div) * fix->mult;
// best_parent_rate = current ddrpll rate = 1600000000
// req->rate = (1600000000 / 2) * 1 = 800000000 — صح

// بس لو ddrpll اتغير (مثلاً من thermal throttle) لـ 1400000000
// req->rate = (1400000000 / 2) * 1 = 700000000
```

الـ AM62x thermal management كانت بتخفض الـ `ddrpll` تحت load عالي — والـ `ddr_clk` بيتبعه تلقائياً عن طريق `clk_factor_recalc_rate`، بس الـ DDR controller محتاج re-training عند تغيير الـ frequency.

#### الحل

**أولاً:** منع thermal throttle من تغيير الـ ddrpll:

```dts
&ddrpll {
    clock-frequency = <1600000000>;
    assigned-clocks = <&ddrpll>;
    assigned-clock-rates = <1600000000>;
};
```

**ثانياً:** تسجيل الـ `ddr_clk` بـ `CLK_IS_CRITICAL` عشان الـ CCF ميعملش عليه operations غير متوقعة:

```c
hw = clk_hw_register_fixed_factor(dev, "ddr_clk", "ddrpll",
    CLK_IS_CRITICAL, /* لا تحاول تغيير الـ rate */
    1, 2);
```

**ثالثاً:** لتتبع الـ accuracy بشكل صح مع النسخ الجديدة من الـ kernel:

```c
hw = clk_hw_register_fixed_factor_with_accuracy_fwname(dev, np,
    "ddr_clk", "ddrpll",
    CLK_IS_CRITICAL,
    1, 2,
    300 /* 300 ppb — نفس الـ PLL */
);
```

**Debug commands:**

```bash
# مراقبة الـ ddrpll rate
watch -n 1 'cat /sys/kernel/debug/clk/ddrpll/clk_rate'

# فحص الـ thermal state
cat /sys/class/thermal/thermal_zone0/temp
cat /sys/class/thermal/thermal_zone0/policy
```

#### الدرس المستفاد
الـ fixed-factor clock بتعكس أي تغيير في الـ parent rate تلقائياً عن طريق `clk_factor_recalc_rate` — ده خطر على clocks زي DDR اللي محتاجة hardware re-training عند تغيير الـ frequency. استخدم `CLK_IS_CRITICAL` أو ثبت الـ parent PLL عشان تمنع التغيير غير المتوقع.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [A common clock framework](https://lwn.net/Articles/472998/) | أول تغطية شاملة لـ common clock framework — بيشرح التصميم العام وعمليات `clk_set_rate()` وشجرة الـ clocks |
| [common clk framework (v2)](https://lwn.net/Articles/486841/) | تحسينات الـ framework: معالجة الـ orphan clocks والـ multi-parent clocks |
| [Common struct clk implementation, v5](https://lwn.net/Articles/392902/) | الإصدار الأولي من النقاشات اللي أدّت لتوحيد `struct clk` عبر معمارية ARM |
| [clk: implement clock rate protection mechanism](https://lwn.net/Articles/740484/) | إضافة آلية حماية الـ rate — مرتبط بكيفية تعامل `clk_fixed_factor` مع `CLK_SET_RATE_PARENT` |
| [clk: Add kunit tests for fixed rate and parent data](https://lwn.net/Articles/976983/) | إضافة KUnit tests للـ fixed rate clocks — مباشرة متعلق بـ `clk-fixed-factor` |
| [Binding and driver for gated-fixed-clocks](https://lwn.net/Articles/987538/) | إضافة نوع جديد `gated-fixed-clock` — يستخدم نفس مفاهيم الـ `clk-fixed-factor` |

---

### التوثيق الرسمي داخل الـ kernel

**الـ driver API documentation:**

```
Documentation/driver-api/clk.rst
```

الملف ده هو المرجع الرسمي لـ Common Clock Framework — بيشرح:
- الـ `struct clk_ops` وكل callback
- الفرق بين `clk_hw_register_*` و `clk_register_*`
- آليات الـ `determine_rate` / `round_rate`

**الـ Device Tree binding:**

```
Documentation/devicetree/bindings/clock/fixed-factor-clock.yaml
```

YAML schema للـ `compatible = "fixed-factor-clock"` — بيحدد الـ required properties: `clock-div` و `clock-mult`.

**الـ clock bindings العامة:**

```
Documentation/devicetree/bindings/clock/clock-bindings.txt
```

---

### Kernel Commits المهمة

| الـ commit | التاريخ | الوصف |
|------------|---------|-------|
| [`f0948f59dbc8`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0948f59dbc8) | 2012-05-03 | **الإضافة الأولى** — Sascha Hauer أضاف `clk-fixed-factor.c` للـ kernel |
| [`79b16641efab`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=79b16641efab) | 2013-04-12 | إضافة Device Tree binding لـ `fixed-factor-clock` |
| [`ecbf3f1795fd`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ecbf3f1795fd) | — | تحويل لـ `clk_parent_data` بدل الـ parent name — السماح للـ framework باختيار الـ parent |
| [`6ebd5247ad2a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6ebd5247ad2a) | — | إضافة `clk_hw_register_fixed_factor_parent_hw()` — تسجيل بـ `clk_hw` pointer |
| [`ff773fd21999`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ff773fd21999) | 2024-02-21 | إضافة دعم `CLK_FIXED_FACTOR_FIXED_ACCURACY` و `recalc_accuracy` |
| [`582de809b052`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=582de809b052) | 2025-08-11 | إضافة `determine_rate()` ops بدل الـ deprecated `round_rate()` |
| [`e0c26569d3ad`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e0c26569d3ad) | 2025-08-11 | حذف `round_rate()` نهائياً من `clk_fixed_factor_ops` |
| [`50ce6826a48f`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=50ce6826a48f) | — | إصلاح double free في الـ `devm` resource management |

---

### مناقشات Mailing List

- **[lore.kernel.org/lkml](https://lore.kernel.org/lkml/)** — الأرشيف الرسمي لكل النقاشات
- **[Patchwork: clk DT fixed-factor-clock binding](https://lore.kernel.org/patchwork/patch/372891/)** — نقاش إضافة الـ DT binding لـ `fixed-factor-clock`
- **[lore.kernel.org/linux-clk](https://lore.kernel.org/linux-clk/)** — قائمة بريدية مخصصة للـ clock subsystem — هنا بيتناقش أي تعديل على `clk-fixed-factor.c`

---

### مصادر eLinux.org

| المصدر | الوصف |
|--------|-------|
| [Generic Linux CLK Framework — CELF 2009 (PDF)](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | عرض من مؤتمر CELF 2009 عن الـ generic clock framework قبل توحيده |
| [Linux clock management framework — ELC 2007 (PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | الجذور التاريخية لإدارة الـ clocks في Linux Embedded |
| [Common Clock Framework BoFs (PDF)](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | نقاش Birds-of-a-Feather عن الـ Common Clock Framework |

---

### مصادر Bootlin / Elixir

| المصدر | الوصف |
|--------|-------|
| [clk-fixed-factor.c — Elixir Cross Referencer](https://elixir.bootlin.com/linux/latest/source/drivers/clk/clk-fixed-factor.c) | تصفح الـ source code مع cross-reference لكل symbol |
| [of_fixed_factor_clk_setup identifier](https://elixir.bootlin.com/linux/latest/ident/of_fixed_factor_clk_setup) | كل الأماكن اللي بتستخدم `of_fixed_factor_clk_setup` في الـ kernel |
| [clk_register_fixed_factor identifier](https://elixir.bootlin.com/linux/latest/ident/clk_register_fixed_factor) | كل استخدامات `clk_register_fixed_factor` عبر الـ kernel |
| [Common Clock Framework — Bootlin ELCE 2013 (PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | شرح عملي للـ framework من Gregory Clement |

---

### الكتب الموصى بها

| الكتاب | الفصل / الموضوع ذو الصلة |
|--------|--------------------------|
| **Linux Device Drivers, 3rd Edition (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 1 (Introduction), Chapter 14 (The Linux Device Model) — الأساس لفهم `struct device` و `devres` اللي بيستخدمهم الـ `clk-fixed-factor` |
| **Linux Kernel Development, 3rd Edition** — Robert Love | Chapter 17 (Devices and Modules) — بيشرح الـ platform_driver وآلية الـ `of_match_table` |
| **Embedded Linux Primer, 2nd Edition** — Christopher Hallinan | Chapter 15 (Debugging Embedded Linux Applications/Kernel) + أقسام الـ Device Tree — مهم لفهم كيف بيتسجل `fixed-factor-clock` من الـ DTS |
| **Mastering Embedded Linux Programming** — Frank Vasquez, Chris Simmonds | الأقسام المتعلقة بالـ clock management وتوليف الـ SoC |

---

### الـ Kernel Documentation الرسمي أونلاين

| الرابط | الوصف |
|--------|-------|
| [The Common Clk Framework — docs.kernel.org](https://docs.kernel.org/driver-api/clk.html) | التوثيق الرسمي الكامل لـ Common Clock Framework |
| [fixed-factor-clock.txt — kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/clock/fixed-factor-clock.txt) | الـ DT binding القديم (النسخة الجديدة YAML) |
| [clk.txt — kernel.org](https://www.kernel.org/doc/Documentation/clk.txt) | النسخة القديمة من توثيق الـ framework |

---

### مصطلحات البحث

عشان تلاقي مزيد من المعلومات، استخدم المصطلحات دي:

```
linux kernel clk_fixed_factor
linux common clock framework CCF driver
CLK_SET_RATE_PARENT fixed factor
clk_hw_register_fixed_factor devm
of_fixed_factor_clk_setup device tree
linux clk determine_rate round_rate
devres clk_hw_register
linux clock divider multiplier fixed
```

**الـ kernel source paths المهمة:**

```
drivers/clk/clk-fixed-factor.c          # الملف الرئيسي
include/linux/clk-provider.h            # struct clk_fixed_factor, clk_fixed_factor_ops
Documentation/driver-api/clk.rst        # التوثيق الرسمي
Documentation/devicetree/bindings/clock/fixed-factor-clock.yaml  # DT schema
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على `clk_factor_recalc_rate` — الدالة اللي بتحسب rate الـ fixed-factor clock من parent rate. دي safe لأنها read-only، مش بتغير state، وبتتكلم كتير وقت booting وكمان وقت أي driver بيعمل `clk_get_rate()`. هنطبع اسم الـ clock، الـ parent rate، و الـ rate المحسوب بعد تطبيق mult/div.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_fixed_factor.c
 *
 * Hooks clk_factor_recalc_rate() to log every fixed-factor
 * clock rate recalculation: name, parent_rate, and computed rate.
 */

#include <linux/module.h>      /* module_init / module_exit / MODULE_* */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, ...   */
#include <linux/clk-provider.h>/* struct clk_hw, to_clk_fixed_factor   */
#include <linux/printk.h>      /* pr_info                               */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on clk_factor_recalc_rate to trace fixed-factor clocks");

/* ------------------------------------------------------------------ */
/*  pre-handler: called just before clk_factor_recalc_rate executes   */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first arg (hw) is in RDI, second (parent_rate) in RSI.
     * On arm64:  first arg in x0, second in x1.
     * We use regs_get_kernel_argument() which is arch-agnostic.
     */
    struct clk_hw *hw = (struct clk_hw *)regs_get_kernel_argument(regs, 0);
    unsigned long parent_rate = regs_get_kernel_argument(regs, 1);

    struct clk_fixed_factor *fix;
    unsigned long long computed;
    const char *clk_name = "(unknown)";

    if (!hw)
        return 0;

    /* clk_hw_get_name() is safe to call here — no locks needed */
    clk_name = clk_hw_get_name(hw);

    /* cast hw → clk_fixed_factor to read mult/div */
    fix = to_clk_fixed_factor(hw);

    /* replicate the kernel formula: rate = parent * mult / div */
    computed = (unsigned long long)parent_rate * fix->mult;
    do_div(computed, fix->div);

    pr_info("[ff_kprobe] clock=%-24s  parent=%lu Hz  mult=%u  div=%u  => %llu Hz\n",
            clk_name, parent_rate, fix->mult, fix->div, computed);

    return 0; /* 0 = let the real function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                  */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    /*
     * We probe the *static* (non-exported) helper.
     * clk_factor_recalc_rate is not exported, but kprobes can reach
     * any kernel symbol present in /proc/kallsyms.
     */
    .symbol_name = "clk_factor_recalc_rate",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module init / exit                                                 */
/* ------------------------------------------------------------------ */
static int __init ff_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[ff_kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[ff_kprobe] planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit ff_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[ff_kprobe] removed from %s\n", kp.symbol_name);
}

module_init(ff_kprobe_init);
module_exit(ff_kprobe_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_fixed_factor.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod kprobe_fixed_factor.ko

# شوف الـ output
sudo dmesg -w | grep ff_kprobe

# لو حابب تجبر recalculation — اقرأ rate أي clock
cat /sys/kernel/debug/clk/*/clk_rate   # أو استخدم clk_summary

# إزالة الـ module
sudo rmmod kprobe_fixed_factor
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | يعرّف `struct kprobe` وكل API الـ registration |
| `linux/clk-provider.h` | يعرّف `struct clk_hw`، `struct clk_fixed_factor`، و `to_clk_fixed_factor()` |
| `linux/printk.h` | `pr_info()` / `pr_err()` |

الـ `linux/module.h` ضروري لأي kernel module — بيوفر `module_init`, `module_exit`, `MODULE_LICENSE`, إلخ.

#### الـ `handler_pre`

الـ pre-handler بيتنفذ قبل الدالة المستهدفة بالظبط، فـ arguments لسه موجودة في الـ registers. بنستخدم `regs_get_kernel_argument()` بدل الـ access المباشر لـ `regs->di` أو `regs->regs[0]` عشان الكود يبقى portable على x86 وARM على حد سواء. بعد ما جبنا الـ `clk_hw`، استخدمنا `to_clk_fixed_factor()` — وهو container_of macro — نقرأ منه `mult` و `div` ونحسب الـ rate بنفس formula الـ kernel عشان نتحقق من الـ output.

#### الـ `return 0` في الـ handler

القيمة صفر بتقول للـ kprobe infrastructure "خلي الدالة الأصلية تكمل طبيعي". لو رجعنا قيمة غير صفر، الـ kernel ممكن يـ abort التنفيذ — مش مطلوب هنا.

#### الـ `kp.symbol_name`

بدل ما نحط العنوان الـ hardcoded، بنحط اسم الـ symbol وبنسيب للـ kprobe infrastructure إنه يبحث في `kallsyms` وقت الـ `register_kprobe`. الدالة `clk_factor_recalc_rate` مش exported بـ `EXPORT_SYMBOL_GPL`، لكن طالما هي موجودة في `kallsyms` (مش inlined)، الـ kprobe يوصلها.

#### `module_init` و `module_exit`

الـ `init` بيسجل الـ kprobe — لو فشل بيرجع الـ error مباشرةً ومش بيكمل. الـ `exit` لازم دايماً يعمل `unregister_kprobe()` عشان يشيل الـ breakpoint من الكود ويحرر الـ resources؛ لو نسينا ده، الـ handler هيتنفذ حتى بعد ما الـ module اتشال من الـ memory وده kernel panic مضمون.

---

### مثال على الـ output

```
[ff_kprobe] planted at clk_factor_recalc_rate (ffffffffc0a1b230)
[ff_kprobe] clock=apb_clk                    parent=100000000 Hz  mult=1  div=4  => 25000000 Hz
[ff_kprobe] clock=uart_clk                   parent=100000000 Hz  mult=1  div=8  => 12500000 Hz
[ff_kprobe] clock=mmc_clk                    parent=200000000 Hz  mult=1  div=2  => 100000000 Hz
[ff_kprobe] removed from clk_factor_recalc_rate
```

الـ output ده بيكشف فعلياً الـ clock tree — كل fixed-factor node بالـ parent rate والـ output rate — وده مفيد جداً في debugging مشاكل الـ timing على SoCs.
