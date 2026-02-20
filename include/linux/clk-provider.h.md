## Phase 1: الصورة الكبيرة

# `clk-provider.h`

> **PATH**: `include/linux/clk-provider.h`
> **Subsystem**: Common Clock Framework (CCF)
> **الوظيفة الأساسية**: الـ header الرئيسي اللي يستخدمه كل **كاتب درايفر clock** عشان يعرّف clock جديد ويسجّله في الـ kernel

---

### ما هي المشكلة اللي يحلّها هذا الملف؟

تخيّل إنك بتبني مصنع ضخم فيه مئات الآلات. كل آلة تحتاج كهرباء بجهد وتردد معيّن — بعضها يحتاج 5V وبعضها 12V وبعض الآلات تشتغل بنص السرعة أو ضعف السرعة.

في المعالجات (SoC)، الـ **clock** هو نبضة كهربائية منتظمة — يعني نفس فكرة التردد (Hz). كل جزء من الشريحة (CPU، GPU، USB، display، camera...) يحتاج clock بتردد معيّن عشان يشتغل صح.

المشكلة: كل شركة (Qualcomm، MediaTek، Samsung، Raspberry Pi...) عندها **hardware clock مختلف تمامًا**. لو كل واحد كتب كود الـ clock من الصفر بطريقته، الـ kernel سيبقى فوضى.

الحل: الـ kernel اخترع **Common Clock Framework** — إطار موحّد. وهذا الملف هو **عقد التعاقد** بين الإطار وكل درايفر clock.

---

### القصة الكاملة: من يكتب الـ clock ومن يستخدمه؟

```
┌─────────────────────────────────────────────────────┐
│                   مستخدم الـ clock                  │
│         (درايفر USB، GPU، Camera، إلخ)              │
│                  يستخدم: clk.h                       │
└────────────────────┬────────────────────────────────┘
                     │  clk_get() / clk_enable() / clk_set_rate()
                     ▼
┌─────────────────────────────────────────────────────┐
│           Common Clock Framework (CCF)              │
│              drivers/clk/clk.c                       │
│   يدير الشجرة، يتحقق من الـ constraints             │
└────────────────────┬────────────────────────────────┘
                     │  يستدعي struct clk_ops
                     ▼
┌─────────────────────────────────────────────────────┐
│           كاتب درايفر الـ clock                     │
│   (يمثّل هاردوير Qualcomm / RPi / إلخ)              │
│              يستخدم: clk-provider.h  ← نحن هنا      │
└─────────────────────────────────────────────────────┘
```

الملف هذا هو **واجهة الطبقة الثانية** — اللي يراها كاتب الـ clock driver فقط، مش المستخدم العادي.

---

### ما الذي يوفّره هذا الملف؟

#### 1. الـ `struct clk_ops` — عقد العمل مع الـ clock

تخيّل إنك توظّف موظف وتعطيه قائمة مهام لازم يقدر يعملها. الـ `struct clk_ops` هي هذه القائمة:

```c
struct clk_ops {
    /* هل الـ clock جاهز للعمل؟ */
    int     (*prepare)(struct clk_hw *hw);
    void    (*unprepare)(struct clk_hw *hw);

    /* شغّل / أوقف الـ clock */
    int     (*enable)(struct clk_hw *hw);
    void    (*disable)(struct clk_hw *hw);

    /* احسب التردد الفعلي بناءً على الـ parent */
    unsigned long (*recalc_rate)(struct clk_hw *hw, unsigned long parent_rate);

    /* لو طلبت تردد معيّن، إيش أقرب قيمة ممكنة؟ */
    long    (*round_rate)(struct clk_hw *hw, unsigned long rate, unsigned long *parent_rate);

    /* غيّر التردد فعليًا في الهاردوير */
    int     (*set_rate)(struct clk_hw *hw, unsigned long rate, unsigned long parent_rate);

    /* من هو الـ parent الحالي؟ */
    u8      (*get_parent)(struct clk_hw *hw);
    int     (*set_parent)(struct clk_hw *hw, u8 index);

    /* duty cycle — نسبة HIGH إلى LOW في الموجة */
    int     (*get_duty_cycle)(struct clk_hw *hw, struct clk_duty *duty);
    int     (*set_duty_cycle)(struct clk_hw *hw, struct clk_duty *duty);

    /* ... وغيرها */
};
```

الـ driver يملأ هذه الـ callbacks، والـ CCF يستدعيها عند الحاجة. لو الـ driver ما ملأ callback معين، الـ CCF يتجاهله بأمان.

---

#### 2. الـ `struct clk_hw` — هوية الـ clock في الـ kernel

```c
struct clk_hw {
    struct clk_core *core;  /* البيانات الداخلية للـ CCF */
    struct clk      *clk;   /* النسخة العامة للمستخدمين */
    const struct clk_init_data *init; /* بيانات التهيئة */
};
```

الـ `clk_hw` هو **بطاقة هوية الـ clock**. كل clock driver يدمج هذا الـ struct داخل الـ struct الخاص به:

```c
/* مثال: clock بمعدل ثابت */
struct clk_fixed_rate {
    struct clk_hw hw;       /* لازم يكون أول شيء أو تستخدم to_clk_fixed_rate() */
    unsigned long fixed_rate;
    unsigned long fixed_accuracy;
    unsigned long clk_latency_ns;
};
```

---

#### 3. الـ `struct clk_init_data` — ورقة التسجيل

```c
struct clk_init_data {
    const char *name;                        /* اسم الـ clock */
    const struct clk_ops *ops;               /* قائمة المهام */
    const struct clk_parent_data *parent_data; /* من هم الـ parents المحتملون */
    u8 num_parents;                          /* عددهم */
    unsigned long flags;                     /* سلوكيات خاصة */
};
```

---

#### 4. الـ Flags — قواعد السلوك

| Flag | المعنى |
|------|--------|
| `CLK_SET_RATE_GATE` | لازم توقف الـ clock قبل ما تغيّر تردده |
| `CLK_SET_RATE_PARENT` | لو ما قدرت تحقق التردد، اطلب من الـ parent يغيّر تردده |
| `CLK_IGNORE_UNUSED` | لا توقف هذا الـ clock حتى لو ما أحد يستخدمه |
| `CLK_IS_CRITICAL` | هذا الـ clock حيوي — لا توقفه أبدًا (مثل clock الـ CPU الأساسي) |
| `CLK_GET_RATE_NOCACHE` | لا تثق في الـ cache — اسأل الهاردوير مباشرة |
| `CLK_SET_RATE_UNGATE` | الـ clock لازم يكون شغّال عشان تقدر تغيّر تردده |

---

#### 5. الـ `struct clk_rate_request` — طلب التردد

```c
struct clk_rate_request {
    unsigned long rate;           /* التردد المطلوب */
    unsigned long min_rate;       /* الحد الأدنى المقبول */
    unsigned long max_rate;       /* الحد الأقصى المقبول */
    unsigned long best_parent_rate; /* أفضل تردد يقدر الـ parent يوفّره */
    struct clk_hw *best_parent_hw;  /* أفضل parent اخترناه */
};
```

تخيّل إنك تطلب من مصنع كهرباء "أحتاج 100MHz". المصنع يرد: "أقدر أعطيك 98MHz أو 104MHz، أيهما تفضل؟" هذا الـ struct هو ورقة التفاوض.

---

#### 6. أنواع الـ clocks الجاهزة (Building Blocks)

الـ CCF يوفّر أنواعًا جاهزة تقدر تستخدمها مباشرة:

```
clk_fixed_rate    → clock ثابت لا يتغيّر (مثل crystal خارجي 24MHz)
clk_gate          → clock يُشعَل ويُطفأ فقط (ON/OFF)
clk_divider       → clock مقسوم (parent_rate / N)
clk_mux           → تختار من بين عدة مصادر
clk_fixed_factor  → (parent_rate * M) / N
clk_fractional_divider → قسمة كسرية دقيقة
clk_multiplier    → مضاعف
clk_composite     → تجميع mux + divider + gate في clock واحد
```

تخيّل إنها **قطع LEGO** — تبني منها شجرة الـ clocks الخاصة بشريحتك.

---

#### 7. ربط الـ Device Tree بالـ clocks

```c
/* تسجيل مزود clocks في الـ device tree */
of_clk_add_hw_provider(np, of_clk_hw_onecell_get, clk_data);

/* ماكرو التعريف التلقائي عند boot */
CLK_OF_DECLARE(my_clks, "vendor,my-clk", my_clk_init);
```

لما الـ kernel يقرأ الـ device tree ويلاقي `compatible = "vendor,my-clk"` يستدعي `my_clk_init` تلقائيًا.

---

### شجرة الـ clocks: كيف تبدو؟

```
Crystal 24MHz (clk_fixed_rate)
    │
    ├── PLL0: 1.2GHz (clk_fixed_factor × 50)
    │       │
    │       ├── CPU clock: 600MHz (clk_divider ÷ 2)
    │       └── GPU clock: 400MHz (clk_divider ÷ 3)
    │
    ├── PLL1: 480MHz (clk_fixed_factor × 20)
    │       │
    │       └── USB clock: 480MHz (clk_gate)
    │
    └── OSC: 24MHz مباشر
            │
            └── RTC clock: 32KHz (clk_divider ÷ 750)
```

كل `clk_hw` هو عقدة في هذه الشجرة. الـ CCF يدير العلاقات بينها ويضمن إن لو غيّرت تردد clock، يُحدَّث كل ما يعتمد عليه.

---

### الملفات المرتبطة

#### الملفات الأساسية في الـ CCF

| الملف | الدور |
|-------|-------|
| `include/linux/clk-provider.h` | **نحن هنا** — واجهة كاتب الـ driver |
| `include/linux/clk.h` | واجهة مستخدم الـ clock (clk_get, clk_enable...) |
| `drivers/clk/clk.c` | قلب الـ CCF — المنطق الكامل للإدارة |
| `drivers/clk/clkdev.c` | ربط الـ clocks بالـ devices |
| `include/linux/of_clk.h` | دعم الـ Device Tree للـ clocks |

#### أنواع الـ clocks الجاهزة

| الملف | النوع |
|-------|-------|
| `drivers/clk/clk-fixed-rate.c` | `clk_fixed_rate` |
| `drivers/clk/clk-gate.c` | `clk_gate` |
| `drivers/clk/clk-divider.c` | `clk_divider` |
| `drivers/clk/clk-mux.c` | `clk_mux` |
| `drivers/clk/clk-fixed-factor.c` | `clk_fixed_factor` |
| `drivers/clk/clk-composite.c` | `clk_composite` |
| `drivers/clk/clk-fractional-divider.c` | `clk_fractional_divider` |

#### أمثلة على درايفرات حقيقية

| الملف | الشريحة |
|-------|---------|
| `drivers/clk/clk-bcm2835.c` | Raspberry Pi |
| `drivers/clk/qcom/` | Qualcomm Snapdragon |
| `drivers/clk/samsung/` | Samsung Exynos |
| `drivers/clk/mediatek/` | MediaTek |
| `drivers/clk/sunxi-ng/` | Allwinner |

---

### ملخص: لماذا هذا الملف مهم؟

هذا الملف هو **الجسر** بين الهاردوير الفعلي والـ kernel. بدونه، كل شركة ستكتب إدارة الـ clock بطريقتها الخاصة، ولن يوجد أي توحيد. بفضله:

- كل clock driver يتبع نفس **العقد** (`struct clk_ops`)
- الـ CCF يقدر يدير شجرة كاملة من الـ clocks بشكل مركزي
- إدارة الطاقة تعمل تلقائيًا — الـ kernel يوقف الـ clocks غير المستخدمة
- الـ device tree يصف الـ clocks بشكل منفصل عن الكود

---

## Phase 2: شرح الـ Common Clock Framework

---

### المشكلة — ليش يوجد هذا الـ Subsystem أصلاً؟

تخيل إنك تبني نظام تشغيل لجهاز فيه معالج، وشاشة، وـ Wi-Fi، وـ USB، وـ camera. كل واحد من هذه الأجزاء يحتاج **clock** — يعني إشارة كهربائية تنبض بتردد معين (مثلاً 100 MHz) علشان يعرف متى "يتحرك" وينفذ عملياته.

المشكلة القديمة كانت هكذا:

- كل **driver** يكتب الكود الخاص فيه لإدارة الـ clock
- لو عندك Qualcomm، Samsung، NXP — كل شركة تكتب كود مختلف تماماً
- **لا يوجد تنسيق** بين الـ drivers — لو الـ USB يريد يشغل الـ clock والـ camera تريد توقفه في نفس الوقت، ماذا يحدث؟
- **لا يوجد power management** مركزي — الـ clock يبقى شغال حتى لو لا أحد يستخدمه، وهذا يهدر طاقة كثيرة
- **لا يوجد debugging** — مستحيل تعرف من يستخدم أي clock

هذه الفوضى كانت موجودة فعلاً في الـ Linux kernel قبل عام 2012. كل SoC (System on Chip) كان عنده كود مختلف، مكرر، وصعب الصيانة.

---

### الحل — الـ Common Clock Framework (CCF)

الـ **CCF** جاء سنة 2012 (Linux 3.4) كـ unified framework يحل هذه المشاكل. الفكرة الأساسية:

**فصل المسؤوليات إلى ثلاث طبقات:**

```
┌─────────────────────────────────────────────────────┐
│                   Consumer API                       │
│     (clk_get, clk_enable, clk_set_rate, ...)        │
│           ← الـ drivers العادية تستخدم هذا          │
├─────────────────────────────────────────────────────┤
│              CCF Core (clk.c)                        │
│     (tree management, locking, ref-counting,        │
│      rate propagation, power management)             │
│           ← المحرك الداخلي، لا أحد يلمسه           │
├─────────────────────────────────────────────────────┤
│                Provider API                          │
│     (clk_ops, clk_hw, clk_hw_register_*, ...)       │
│           ← كود الـ hardware الخاص بكل SoC          │
└─────────────────────────────────────────────────────┘
```

الـ **Consumer** (مثلاً driver الـ USB) لا يعرف شيئاً عن الـ hardware. هو فقط يقول "أريد clock بـ 480 MHz". الـ CCF Core يأخذ هذا الطلب، يتفاوض مع شجرة الـ clocks، ويوجه الأمر للـ **Provider** الصحيح (كود الـ Qualcomm مثلاً).

---

### التشبيه الحقيقي — شبكة الكهرباء

تخيل **شبكة كهرباء** في مدينة:

| الكهرباء | الـ CCF |
|----------|---------|
| محطة توليد الكهرباء | الـ Crystal Oscillator (المصدر الأساسي، مثلاً 24 MHz) |
| محولات الجهد (Transformers) | الـ PLL (يرفع 24 MHz إلى 1 GHz) |
| خطوط التوزيع | الـ clock tree |
| مفاتيح القطع (Circuit Breakers) | الـ clock gates |
| مقسمات الجهد | الـ clock dividers |
| العداد (الميزان) | الـ reference counting |
| شركة الكهرباء | الـ CCF Core |
| المصانع والمنازل | الـ consumer drivers |

المصنع (الـ USB driver) لا يهمه من أين تأتي الكهرباء — هو فقط يوصل القابس. شركة الكهرباء (CCF) تدير كل شيء: توليد، توزيع، فصل التيار عند عدم الاستخدام.

---

### الـ Clock Tree — المفهوم الأهم

الـ clocks في أي SoC ليست مستقلة — هي **شجرة هرمية**. كل clock له **parent** يأخذ منه تردده ويعدله:

```
                    [Crystal 24 MHz]
                          │
                    ┌─────┴─────┐
                    │   PLL_A   │  ← يضرب × 40 = 960 MHz
                    └─────┬─────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
        [DIV ÷2]      [DIV ÷4]    [MUX]──── [Crystal 24MHz]
        480 MHz        240 MHz         │
             │            │        [USB PHY Clock]
        [GATE]        [GATE]
          │               │
       [USB3]         [SDMMC]
```

هذه الشجرة تعني:
- لو غيرت الـ PLL_A، تتأثر **كل** الـ clocks تحته تلقائياً
- لو أوقفت الـ USB3، يجب إيقاف الـ GATE فقط — لكن لا تلمس الـ PLL_A لأن الـ SDMMC لا يزال يستخدمه
- **Reference counting**: الـ PLL_A يبقى شغالاً طالما يوجد أي consumer واحد فعال تحته

---

### المفاهيم الأساسية العميقة

#### 1. الـ `struct clk_hw` — جسر بين الـ Core والـ Hardware

```c
struct clk_hw {
    struct clk_core *core;   /* ← يشير لـ CCF Core internal struct */
    struct clk *clk;         /* ← الـ handle الذي يعطى للـ consumers */
    const struct clk_init_data *init; /* ← بيانات التهيئة */
};
```

الـ `clk_hw` هو **الحلقة الوسيطة**. عندما يكتب مطور driver الـ hardware struct الخاص فيه، يضع `clk_hw` كأول عضو:

```c
/* مثال: clock ثابت التردد */
struct clk_fixed_rate {
    struct clk_hw   hw;        /* ← يجب أن يكون أولاً */
    unsigned long   fixed_rate;
    unsigned long   fixed_accuracy;
    unsigned long   flags;
};
```

هذه التقنية (وضع struct داخل struct) تسمح للـ CCF باستخدام `container_of()` للوصول للـ hardware struct من الـ `clk_hw` pointer:

```
┌──────────────────────────────────┐
│      clk_fixed_rate              │
│  ┌─────────────────────────┐     │
│  │       clk_hw            │ ←── CCF يعرف هذا الجزء فقط
│  │   core* ──────────────► clk_core (internal) │
│  └─────────────────────────┘     │
│   fixed_rate = 24000000          │
│   fixed_accuracy = 0             │
└──────────────────────────────────┘
         ↑
   container_of(hw, struct clk_fixed_rate, hw)
   يعطيك العنوان الكامل للـ struct
```

---

#### 2. الـ `struct clk_ops` — عقد العمل

هذا الـ struct هو **الواجهة** بين الـ CCF Core وأي hardware clock. الـ CCF لا يهمه نوع الـ hardware — هو فقط يستدعي هذه الـ callbacks:

**لماذا prepare و enable منفصلان؟**

هذا سؤال مهم جداً. التمييز جاء لسبب عملي:

- **`prepare`**: عملية **قد تنام** (sleep) — مثلاً انتظار PLL يستقر (lock) يأخذ مئات الـ microseconds. لذلك لا يمكن استدعاؤه في السياقات التي لا يمكن فيها النوم (interrupt context, atomic context).
- **`enable`**: عملية **سريعة وذرية** — فقط كتابة bit واحد في register. يمكن استدعاؤه حتى في interrupt context.

```
زمن                    التسلسل الصحيح:
  │
  ├─► clk_prepare()   ← قد ينام، يهيئ الـ PLL، ينتظر الاستقرار
  │
  ├─► clk_enable()    ← سريع، يفتح الـ gate فقط، لا ينام
  │
  ├─► [استخدام الـ clock]
  │
  ├─► clk_disable()   ← سريع، يغلق الـ gate
  │
  └─► clk_unprepare() ← يوقف الـ PLL، قد ينام
```

---

#### 3. الـ `struct clk_rate_request` — التفاوض على التردد

عندما يطلب driver "أريد 100 MHz"، الأمر ليس بسيطاً. الـ hardware له قيود. الـ CCF يستخدم هذا الـ struct كـ "ورقة تفاوض":

```
Parent: 96 MHz أو 120 MHz
تريد: 100 MHz

التفاوض:
  ┌─ جرب Parent=96MHz:  96 ÷ 1 = 96 MHz  (بعيد)
  │                     96 ÷ 2 = 48 MHz  (أبعد)
  │
  └─ جرب Parent=120MHz: 120 ÷ 1 = 120 MHz (قريب)
                        120 ÷ 2 = 60 MHz  (بعيد)

أفضل نتيجة: 120 MHz (أقرب لـ 100 MHz مع القيود)
```

---

#### 4. الـ CLK Flags — سلوك الـ Clock

| Flag | المعنى |
|------|--------|
| `CLK_SET_RATE_GATE` | يجب إيقاف الـ clock قبل تغيير تردده |
| `CLK_SET_RATE_PARENT` | لو طُلب تغيير التردد، غير تردد الـ parent أيضاً |
| `CLK_IGNORE_UNUSED` | لا توقف هذا الـ clock حتى لو لا أحد يستخدمه |
| `CLK_IS_CRITICAL` | هذا الـ clock حيوي، لا توقفه أبداً |
| `CLK_GET_RATE_NOCACHE` | لا تستخدم الـ cache، اقرأ التردد من الـ hardware مباشرة |
| `CLK_SET_RATE_UNGATE` | فتح الـ gate تلقائياً عند تغيير التردد |
| `CLK_OPS_PARENT_ENABLE` | شغّل الـ parent قبل استدعاء أي ops على هذا الـ clock |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CONSUMER SPACE                                │
│                                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │USB Driver│  │GPU Driver│  │CAM Driver│  │I2C Driver│  ...      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘           │
│       │              │              │              │                 │
│  clk_get/enable/set_rate/put (Consumer API - <linux/clk.h>)         │
└───────┼──────────────┼──────────────┼──────────────┼────────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     CCF CORE  (kernel/clk/clk.c)                    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Clock Tree (Radix Tree)                   │   │
│  │  clk_core ──── clk_core ──── clk_core ──── clk_core        │   │
│  │  (Crystal)     (PLL_A)       (DIV÷2)       (USB_GATE)      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│  المسؤوليات: Reference counting, Rate propagation, Locking         │
└─────────────────────────────────────────────────────────────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    PROVIDER API  (<linux/clk-provider.h>)            │
│   struct clk_hw + struct clk_ops + clk_hw_register_*()             │
└─────────────────────────────────────────────────────────────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  HARDWARE-SPECIFIC CLOCK DRIVERS                     │
│  Qualcomm (qcom) │ Samsung (clk-s3) │ NXP (imx) │ Allwinner (sunxi)│
└─────────────────────────────────────────────────────────────────────┘
```

---

### الـ Locking Model — لماذا قفلان؟

#### الـ `prepare_lock` (mutex)
يحمي: `prepare()`, `unprepare()`, `set_rate()`, `set_parent()` — هذه العمليات **قد تنام**.

#### الـ `enable_lock` (spinlock)
يحمي: `enable()`, `disable()` — هذه العمليات **لا تنام أبداً**، وقد تُستدعى من interrupt context.

```
Context         prepare_lock     enable_lock
─────────────   ────────────     ───────────
Process ctx  →  mutex ✓          spinlock ✓
Softirq ctx  →  FORBIDDEN ✗      spinlock ✓
Hardirq ctx  →  FORBIDDEN ✗      spinlock ✓
```

---

### الـ Reference Counting — ضمان عدم إيقاف Clock مشترك

```
USB Driver:    clk_prepare_enable(usb_clk)  → PLL.prepare_count = 1
Camera Driver: clk_prepare_enable(cam_clk)  → PLL.prepare_count = 2
I2C Driver:    clk_prepare_enable(i2c_clk)  → PLL.prepare_count = 3

USB Driver:    clk_disable_unprepare(usb_clk) → PLL.prepare_count = 2
                                               ← PLL لا يزال شغالاً!

I2C Driver:    clk_disable_unprepare(i2c_clk)  → PLL.prepare_count = 0
                                               ← الآن يُوقف PLL فعلاً
```

---

## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ CLK Flags — Cheatsheet

| Flag | Bit | المعنى |
|---|---|---|
| `CLK_SET_RATE_GATE` | 0 | لازم الـ clock يتوقف عشان تغير الـ rate |
| `CLK_SET_PARENT_GATE` | 1 | لازم الـ clock يتوقف عشان تغير الـ parent |
| `CLK_SET_RATE_PARENT` | 2 | لو مش قادر تحقق الـ rate المطلوب، اطلب من الـ parent يغير rate-ه |
| `CLK_IGNORE_UNUSED` | 3 | حتى لو محدش بيستخدمه، متوقفوش |
| `CLK_GET_RATE_NOCACHE` | 6 | دايماً اقرأ الـ rate من الـ hardware مباشرة |
| `CLK_SET_RATE_NO_REPARENT` | 7 | لما تغير الـ rate، ماتغيرش الـ parent |
| `CLK_GET_ACCURACY_NOCACHE` | 8 | نفس فكرة `NOCACHE` بس للـ accuracy |
| `CLK_RECALC_NEW_RATES` | 9 | بعد أي تغيير، أعد حساب الـ rates لكل الـ children |
| `CLK_SET_RATE_UNGATE` | 10 | فتح الـ gate تلقائياً لما تغير الـ rate |
| `CLK_IS_CRITICAL` | 11 | الـ clock ده حيوي للنظام — ممنوع يتوقف أبداً |
| `CLK_OPS_PARENT_ENABLE` | 12 | الـ parent لازم يكون enabled قبل ما تعمل أي ops |
| `CLK_DUTY_CYCLE_PARENT` | 13 | خد بيانات الـ duty cycle من الـ parent |

---

### الـ CLK_GATE Flags

| Flag | Bit | المعنى |
|---|---|---|
| `CLK_GATE_SET_TO_DISABLE` | 0 | اكتب 1 في الـ register عشان تـ disable (عكس المنطق الطبيعي) |
| `CLK_GATE_HIWORD_MASK` | 1 | استخدم الـ high 16 bits كـ write-enable mask |
| `CLK_GATE_BIG_ENDIAN` | 2 | الـ register بيستخدم big-endian byte order |

---

### الـ CLK_DIVIDER Flags

| Flag | Bit | المعنى |
|---|---|---|
| `CLK_DIVIDER_ONE_BASED` | 0 | القيمة 1 في الـ register = قسمة على 1 |
| `CLK_DIVIDER_POWER_OF_TWO` | 1 | القيمة في الـ register هي الأس |
| `CLK_DIVIDER_ALLOW_ZERO` | 2 | سماح بالقيمة صفر في الـ register |
| `CLK_DIVIDER_HIWORD_MASK` | 3 | استخدم الـ high 16 bits كـ write mask |
| `CLK_DIVIDER_ROUND_CLOSEST` | 4 | قرّب للأقرب مش للأصغر |
| `CLK_DIVIDER_READ_ONLY` | 5 | الـ divider للقراءة بس |
| `CLK_DIVIDER_MAX_AT_ZERO` | 6 | القيمة صفر تعني أكبر قسمة ممكنة |
| `CLK_DIVIDER_BIG_ENDIAN` | 7 | big-endian access |
| `CLK_DIVIDER_EVEN_INTEGERS` | 8 | القسمة بالأرقام الزوجية بس |

---

### الـ CLK_MUX Flags

| Flag | Bit | المعنى |
|---|---|---|
| `CLK_MUX_INDEX_ONE` | 0 | الـ index يبدأ من 1 مش من 0 |
| `CLK_MUX_INDEX_BIT` | 1 | الـ index هو رقم الـ bit مش قيمته |
| `CLK_MUX_HIWORD_MASK` | 2 | استخدم الـ high word كـ write mask |
| `CLK_MUX_READ_ONLY` | 3 | للقراءة بس |
| `CLK_MUX_ROUND_CLOSEST` | 4 | اختار أقرب rate ممكن |
| `CLK_MUX_BIG_ENDIAN` | 5 | big-endian access |

---

### كل الـ Structs المهمة

#### `struct clk_hw` — البطاقة الشخصية للـ Clock

```c
struct clk_hw {
    struct clk_core          *core;  // الـ core اللي بيديره kernel
    struct clk               *clk;   // الـ handle اللي بيستخدمه consumers
    const struct clk_init_data *init; // بيانات التسجيل — بتتبقى null بعد التسجيل
};
```

#### `struct clk_init_data` — استمارة التسجيل

```c
struct clk_init_data {
    const char               *name;         // اسم الـ clock — لازم unique في النظام
    const struct clk_ops     *ops;          // قائمة الوظائف
    const char * const       *parent_names; // أسماء الـ parents (الطريقة القديمة)
    const struct clk_parent_data *parent_data; // بيانات الـ parents (الطريقة الجديدة)
    const struct clk_hw     **parent_hws;   // مؤشرات مباشرة للـ parents (الأسرع)
    u8                        num_parents;  // عدد الـ parents
    unsigned long             flags;        // الـ CLK_* flags
};
```

#### `struct clk_ops` — قائمة الوظائف (Virtual Table)

تحتوي على 23 callback function تُحدّد سلوك الـ clock: prepare/unprepare، enable/disable، recalc_rate، round_rate، determine_rate، set_rate، get_parent/set_parent، get_phase/set_phase، get_duty_cycle/set_duty_cycle، init، terminate، debug_init.

**الفرق بين `prepare` و `enable`:**
- **`prepare`**: non-atomic، ممكن يـ sleep — للعمليات التي تأخذ وقتاً (انتظار PLL)
- **`enable`**: atomic، لا ينام — لكتابة register واحد

---

### مخطط علاقات الـ Structs

#### مخطط "الوراثة" بالـ Embedding

```
clk_fixed_rate              clk_gate
┌──────────────┐           ┌──────────────┐
│ clk_hw hw   │           │ clk_hw hw   │
│─────────────│           │─────────────│
│ fixed_rate  │           │ *reg        │
│ accuracy    │           │ bit_idx     │
│ flags       │           │ flags       │
└──────────────┘           │ *lock       │
                           └──────────────┘

clk_divider                clk_mux
┌──────────────┐           ┌──────────────┐
│ clk_hw hw   │           │ clk_hw hw   │
│─────────────│           │─────────────│
│ *reg        │           │ *reg        │
│ shift       │           │ *table      │
│ width       │           │ mask        │
│ flags       │           │ shift       │
│ *table      │           │ flags       │
│ *lock       │           │ *lock       │
└──────────────┘           └──────────────┘

clk_composite — بيجمع الكل:
┌────────────────────────────────────┐
│ clk_hw hw                         │
│ ops (مجمّعة)                      │
│ *mux_hw  ──────────────▶ clk_mux  │
│ *rate_hw ──────────────▶ clk_divider
│ *gate_hw ──────────────▶ clk_gate │
└────────────────────────────────────┘
```

---

### دورة الحياة — Lifecycle

```
Driver Code → clk_hw_register() → kernel allocates clk_core
    → يربط clk_hw.core → clk_core
    → يضيفه لشجرة الـ clocks
    → ops->init(hw) إن وجدت
    → Clock جاهز ✓

Consumer:
    clk_get() → clk_prepare() → clk_enable() → [استخدام] → clk_disable() → clk_unprepare() → clk_put()

إلغاء التسجيل:
    clk_hw_unregister() → ops->terminate() → يحرر clk_core
```

---

### مخطط تدفق تغيير الـ Rate

```
clk_set_rate(clk, 100MHz)
    │
    ├── 1. validate: هل CLK_SET_RATE_GATE؟
    ├── 2. ops->determine_rate(hw, &req)  أو  ops->round_rate()
    │       └── لو CLK_SET_RATE_PARENT: اطلب من الـ parent
    ├── 3. ops->set_rate(hw, rate, parent_rate)
    └── 4. لو CLK_RECALC_NEW_RATES: أعد حساب rates كل الـ children
```

---

### استراتيجية الـ Locking

```
prepare_lock (mutex) — يحمي:
    • clk_prepare/unprepare
    • clk_set_rate / clk_set_parent
    • تعديل شجرة الـ clocks

spinlock_t *lock (في كل struct) — يحمي:
    • القراءة والكتابة في الـ MMIO registers
    • read-modify-write للـ bits

Lock Ordering (لا تعكسه!):
    prepare_lock (mutex)
        └──▶ spinlock (داخل ops->enable/disable)
```

---

## Phase 4: شرح الـ Functions

---

## جدول الملخص

| Function | Type | Purpose |
|----------|------|---------|
| `clk_hw_init_rate_request()` | static inline | تهيئة طلب rate جديد لـ clock |
| `clk_hw_forward_rate_request()` | EXPORT | تمرير طلب rate من clock لـ parent |
| `__clk_hw_register_fixed_rate()` | EXPORT | تسجيل fixed-rate clock (الدالة الأساسية) |
| `clk_register_fixed_rate()` | EXPORT | تسجيل fixed-rate clock (legacy API) |
| `clk_hw_register_fixed_rate` | macro | تسجيل fixed-rate clock بـ name فقط |
| `devm_clk_hw_register_fixed_rate` | macro | تسجيل fixed-rate clock مع devres |
| `clk_hw_unregister_fixed_rate()` | EXPORT | إلغاء تسجيل fixed-rate clock |
| `of_fixed_clk_setup()` | EXPORT | إعداد fixed clock من device tree |
| `__clk_hw_register_gate()` | EXPORT | تسجيل gate clock (الدالة الأساسية) |
| `__devm_clk_hw_register_gate()` | EXPORT | تسجيل gate clock مع devres |
| `clk_register_gate()` | EXPORT | تسجيل gate clock (legacy API) |
| `clk_hw_register_gate` | macro | تسجيل gate clock بـ name |
| `devm_clk_hw_register_gate` | macro | تسجيل gate clock مع devres |
| `clk_gate_is_enabled()` | EXPORT | التحقق إذا كان gate clock مفعّل |
| `clk_gate_restore_context()` | EXPORT | استعادة حالة gate clock بعد suspend |
| `__clk_hw_register_divider()` | EXPORT | تسجيل divider clock (الدالة الأساسية) |
| `divider_recalc_rate()` | EXPORT | حساب rate الفعلي من قيمة register |
| `divider_round_rate_parent()` | EXPORT | تقريب rate مع اختيار parent |
| `divider_determine_rate()` | EXPORT | تحديد أفضل rate لـ divider |
| `divider_get_val()` | EXPORT | تحويل rate لقيمة register |
| `__clk_hw_register_mux()` | EXPORT | تسجيل mux clock (الدالة الأساسية) |
| `clk_mux_val_to_index()` | EXPORT | تحويل قيمة register لـ index |
| `clk_mux_index_to_val()` | EXPORT | تحويل index لقيمة register |
| `clk_hw_register_fixed_factor()` | EXPORT | تسجيل fixed-factor clock (hw API) |
| `clk_register_fractional_divider()` | EXPORT | تسجيل fractional divider clock |
| `clk_hw_register_composite()` | EXPORT | تسجيل composite clock (hw API) |
| `clk_hw_register()` | EXPORT | تسجيل clock عبر hw pointer |
| `devm_clk_hw_register()` | EXPORT | تسجيل clock مع devres (hw API) |
| `of_clk_hw_register()` | EXPORT | تسجيل clock من device tree node |
| `clk_hw_get_rate()` | static inline | إرجاع rate الـ clock الحالي |
| `clk_hw_get_parent()` | static inline | إرجاع hw الـ parent الحالي |
| `clk_hw_get_flags()` | static inline | إرجاع flags الـ clock |
| `clk_hw_is_prepared()` | static inline | التحقق إذا كان clock في حالة prepared |
| `clk_hw_is_enabled()` | static inline | التحقق إذا كان clock مفعّل |
| `clk_hw_reparent()` | EXPORT | تغيير parent للـ clock |
| `clk_hw_set_rate_range()` | EXPORT | تحديد حدود الـ rate |
| `clk_hw_round_rate()` | EXPORT | تقريب rate لأقرب قيمة مدعومة |
| `of_clk_add_hw_provider()` | EXPORT | تسجيل hw clock provider |
| `devm_of_clk_add_hw_provider()` | EXPORT | تسجيل hw provider مع devres |
| `of_clk_del_provider()` | EXPORT | إلغاء تسجيل clock provider |
| `of_clk_hw_onecell_get()` | EXPORT | إرجاع hw من جدول بـ index |
| `of_clk_parent_fill()` | EXPORT | ملء قائمة أسماء الـ parents من DT |
| `of_clk_detect_critical()` | EXPORT | اكتشاف critical clocks من DT |
| `CLK_OF_DECLARE` | macro | تسجيل clock driver لـ device tree |
| `CLK_HW_INIT` | macro | تهيئة clk_init_data بـ parent name |
| `CLK_HW_INIT_HW` | macro | تهيئة clk_init_data بـ parent hw |
| `CLK_HW_INIT_NO_PARENT` | macro | تهيئة clk_init_data بدون parent |
| `CLK_FIXED_FACTOR` | macro | تعريف fixed-factor clock statically |

---

### الفئة الأولى: Rate Request Helpers

#### `clk_hw_init_rate_request()`
تهيّئ `struct clk_rate_request` بقيم ابتدائية صحيحة. تقرأ `rate_range` من الـ clock وتضعها في الـ request، بحيث أي دالة تعالج الطلب لاحقاً تعرف الحدود الدنيا والقصوى.

#### `clk_hw_forward_rate_request()`
تأخذ طلب rate موجود وتحوّله لطلب جديد موجّه للـ parent clock. تحترم `CLK_SET_RATE_PARENT` flag — لو الـ clock مش مسموح له بتغيير rate الـ parent، ترجع خطأ.

---

### الفئة الثانية: Fixed-Rate Clock

**الـ fixed-rate clock** هو مصدر clock ثابت لا يتغيّر — مثل crystal oscillator.

#### `__clk_hw_register_fixed_rate()`
الدالة الأم الداخلية. تخصص ذاكرة لـ `struct clk_fixed_rate`، تملأ حقولها، وتسجّلها. كل الـ macros تستدعي هذه الدالة.

**تختار واحدة فقط من:** `parent_name`, `parent_hw`, أو `parent_data`.

**Macros المتاحة:**
```c
clk_hw_register_fixed_rate(dev, name, parent_name, flags, fixed_rate)
clk_hw_register_fixed_rate_parent_hw(dev, name, parent_hw, flags, fixed_rate)
clk_hw_register_fixed_rate_with_accuracy(dev, name, parent_name, flags, fixed_rate, accuracy)
devm_clk_hw_register_fixed_rate(dev, name, parent_name, flags, fixed_rate)
clk_hw_register_fixed_rate_parent_accuracy(dev, name, parent_data, flags, fixed_rate)
```

**القاعدة:** مؤشر hw مباشر → `parent_hw`. parent في driver آخر → `parent_name` أو `parent_data`.

---

### الفئة الثالثة: Gate Clock

**الـ gate clock** هو مفتاح تشغيل/إيقاف بسيط — لا يغيّر الـ frequency، فقط يقطع أو يوصّل.

#### `__clk_hw_register_gate()`
تنشئ gate clock مرتبطاً بـ bit محدد في register MMIO. عند enable يُكتب `1`، عند disable يُكتب `0`. الـ `lock` ضروري لحماية read-modify-write.

**Macros:** `clk_hw_register_gate`, `clk_hw_register_gate_parent_hw`, `clk_hw_register_gate_parent_data`, `devm_clk_hw_register_gate*`

#### `clk_gate_is_enabled()`
تقرأ قيمة الـ bit من الـ register مع مراعاة `CLK_GATE_SET_TO_DISABLE` flag.

#### `clk_gate_restore_context()`
تُعيد كتابة حالة الـ gate الصحيحة بعد الخروج من suspend. تُستدعى من `clk_ops.restore_context`.

---

### الفئة الرابعة: Divider Clock

**الـ divider clock** — قسمة تردد. `output_rate = parent_rate / divider_value`.

#### `__clk_hw_register_divider()`
تُنشئ `struct clk_divider` وتسجّلها. تدعم dividers بجدول قيم أو بحساب رياضي مباشر.

**Macros:** `clk_hw_register_divider`, `clk_hw_register_divider_parent_hw`, `clk_hw_register_divider_table`, `devm_clk_hw_register_divider*`

#### `divider_recalc_rate(hw, parent_rate, val, table, flags, width)`
تحسب الـ rate الفعلي من القيمة في الـ register. تُستخدم في `clk_ops.recalc_rate`.

#### `divider_round_rate_parent(hw, parent, rate, *prate, table, shift, width, flags, clk_flags)`
تُحدّد أقرب rate يمكن تحقيقه. تُغيّر `*prate` إذا كان مسموحاً بتغيير rate الـ parent.

#### `divider_get_val(rate, parent_rate, table, width, flags)`
عكس `divider_recalc_rate` — تحوّل rate مطلوب لقيمة الـ register.

---

### الفئة الخامسة: Mux Clock

**الـ mux clock** — محوّل يختار بين عدة مصادر clock.

#### `__clk_hw_register_mux()`
تُنشئ mux clock يختار بين `num_parents` مصادر. القيمة في الـ register تحدد أي parent مختار.

**Macros:** `clk_hw_register_mux`, `clk_hw_register_mux_hws`, `clk_hw_register_mux_parent_data`, `clk_hw_register_mux_table`, `devm_clk_hw_register_mux*`

#### `clk_mux_val_to_index()` و `clk_mux_index_to_val()`
تحويل بين قيمة الـ register وـ index الـ parent. مفيدة مع الـ `table` المخصصة.

---

### الفئة السادسة: Fixed-Factor Clock

**output_rate = (parent_rate × mult) / div** — بدون hardware register.

```c
clk_hw_register_fixed_factor(dev, name, parent_name, flags, mult, div)
clk_hw_register_fixed_factor_fwname(dev, np, name, fw_name, flags, mult, div)
clk_hw_register_fixed_factor_parent_hw(dev, name, parent_hw, flags, mult, div)
devm_clk_hw_register_fixed_factor*(...)
```

---

### الفئة السابعة: Fractional Divider

**output = parent_rate × (M / N)** — قسمة كسرية دقيقة.

```c
clk_hw_register_fractional_divider(dev, name, parent_name, flags, reg,
                                    mshift, mwidth, nshift, nwidth,
                                    clk_divider_flags, lock)
```

---

### الفئة الثامنة: Composite Clock

يجمع mux + rate + gate في clock واحد. كل مكوّن اختياري (يمكن NULL).

```c
clk_hw_register_composite(dev, name, parent_names, num_parents,
                           mux_hw, mux_ops,
                           rate_hw, rate_ops,
                           gate_hw, gate_ops,
                           flags)
clk_hw_register_composite_pdata(...)   // مع parent_data
devm_clk_hw_register_composite_pdata(...)  // مع devres
```

---

### الفئة التاسعة: Core Registration

```c
int clk_hw_register(struct device *dev, struct clk_hw *hw);
int devm_clk_hw_register(struct device *dev, struct clk_hw *hw);
int of_clk_hw_register(struct device_node *node, struct clk_hw *hw);
void clk_hw_unregister(struct clk_hw *hw);
```

**`clk_hw_register()`** — الإصدار الحديث المُفضَّل. ترجع `0` أو كود خطأ. الـ `devm_*` تُلغي التسجيل تلقائياً.

**`of_clk_hw_register()`** — للـ clocks المُعرَّفة مبكراً في boot بدون `struct device`.

---

### الفئة العاشرة: Helper Functions

```c
// معلومات أساسية
const char *clk_hw_get_name(const struct clk_hw *hw);          // اسم الـ clock
struct device *clk_hw_get_dev(const struct clk_hw *hw);        // الـ device المالك
struct device_node *clk_hw_get_of_node(const struct clk_hw *hw); // DT node

// معلومات الـ parent
unsigned int clk_hw_get_num_parents(const struct clk_hw *hw);
struct clk_hw *clk_hw_get_parent(const struct clk_hw *hw);
struct clk_hw *clk_hw_get_parent_by_index(const struct clk_hw *hw, unsigned int index);
int clk_hw_get_parent_index(struct clk_hw *hw);
int clk_hw_set_parent(struct clk_hw *hw, struct clk_hw *new_parent);

// معلومات الـ rate
unsigned long clk_hw_get_rate(const struct clk_hw *hw);
unsigned long clk_hw_get_flags(const struct clk_hw *hw);
bool clk_hw_is_prepared(const struct clk_hw *hw);
bool clk_hw_is_enabled(const struct clk_hw *hw);
long clk_hw_round_rate(struct clk_hw *hw, unsigned long rate);

// rate range
void clk_hw_get_rate_range(struct clk_hw *hw, unsigned long *min, unsigned long *max);
int clk_hw_set_rate_range(struct clk_hw *hw, unsigned long min, unsigned long max);
```

---

### الفئة الحادية عشرة: OF Provider Functions

```c
// تسجيل provider
int of_clk_add_hw_provider(struct device_node *np,
                            struct clk_hw *(*get)(struct of_phandle_args*, void*),
                            void *data);
int devm_of_clk_add_hw_provider(struct device *dev, ...);
void of_clk_del_provider(struct device_node *np);

// callbacks جاهزة
struct clk_hw *of_clk_hw_simple_get(...);    // لـ provider بـ clock واحد
struct clk_hw *of_clk_hw_onecell_get(...);   // لـ provider بمصفوفة clocks

// مساعدات DT
int of_clk_parent_fill(struct device_node *np, const char **parents, unsigned int size);
int of_clk_detect_critical(struct device_node *np, int index, unsigned long *flags);
```

---

### الفئة الثانية عشرة: Macros

**تسجيل مبكر في boot:**
```c
CLK_OF_DECLARE(name, compat, fn)          // مبكر جداً (قبل device model)
CLK_OF_DECLARE_DRIVER(name, compat, fn)   // كـ driver عادي
```

**init macros:**

| Macro | عدد الـ Parents | طريقة تحديد الـ Parent |
|-------|----------------|------------------------|
| `CLK_HW_INIT` | واحد | string name |
| `CLK_HW_INIT_HW` | واحد | hw pointer |
| `CLK_HW_INIT_FW_NAME` | واحد | firmware name |
| `CLK_HW_INIT_PARENTS` | متعدد | مصفوفة strings |
| `CLK_HW_INIT_PARENTS_HW` | متعدد | مصفوفة hw pointers |
| `CLK_HW_INIT_PARENTS_DATA` | متعدد | مصفوفة parent_data |
| `CLK_HW_INIT_NO_PARENT` | صفر | — |

**Static init macros:**
```c
CLK_FIXED_FACTOR(_struct, _name, _parent, _div, _mult, _flags)
CLK_FIXED_FACTOR_HW(_struct, _name, _parent, _div, _mult, _flags)
CLK_FIXED_FACTOR_FW_NAME(_struct, _name, _fw_name, _div, _mult, _flags)
```

---

### خلاصة: كيف تختار API المناسب

```
هل تكتب driver جديد؟
    ├── نعم → استخدم hw API (clk_hw_*)
    └── لا → يمكن الإبقاء على legacy API

كيف تعرف الـ parent؟
    ├── مؤشر hw مباشر → parent_hw
    ├── اسم string → parent_name
    └── DT firmware name → fw_name / parent_data

هل تحتاج devres؟
    ├── نعم → devm_* variants
    └── لا → النسخة العادية

نوع الـ clock:
    ├── تردد ثابت → fixed_rate
    ├── مفتاح on/off → gate
    ├── قسمة صحيحة → divider
    ├── قسمة كسرية → fractional_divider
    ├── ضرب وقسمة ثابتان → fixed_factor
    ├── اختيار مصدر → mux
    └── تجميع الكل → composite
```

---

## Phase 5: دليل الـ Debugging الشامل

---

### مقدمة سريعة

تخيل إنك مدير مصنع كبير وفجأة الإنتاج وقف. ماذا تعمل؟ تروح تفتش — السجلات، الماكينات، الأسلاك. الـ debugging في الـ CCF هو نفس الشيء تماماً.

---

## Software Level

### 1. الـ debugfs — مدخل البيانات الذهبي

```
/sys/kernel/debug/clk/
├── clk_summary          ← ملخص كل الـ clocks دفعة واحدة
├── clk_dump             ← نفس المعلومات بصيغة JSON
├── clk_orphan_summary   ← الـ clocks اللي ما لقت parent
├── pll1/
│   ├── clk_rate         ← معدل التردد الحالي بالـ Hz
│   ├── clk_enable_count ← كم مرة اتطلب تفعيله
│   ├── clk_prepare_count
│   ├── clk_protect_count
│   └── clk_flags
```

```bash
# اقرأ الملخص الكامل
cat /sys/kernel/debug/clk/clk_summary

# مثال على الـ output:
#                           enable  prepare  protect
# clock                     count   count    count    rate
# -----------------------------------------------------------
# osc24M                        1       1        0   24000000
#    pll-cpux                   1       1        0  816000000
#       cpux                    1       1        0  816000000
```

**فهم الأعمدة:**

| العمود | المعنى |
|--------|---------|
| `enable_cnt` | عدد المستهلكين اللي طلبوا `clk_enable()` |
| `prepare_cnt` | عدد المستهلكين اللي طلبوا `clk_prepare()` |
| `rate` | التردد الحالي بالـ Hz |

```bash
# الـ orphan clocks — خطر خفي
cat /sys/kernel/debug/clk/clk_orphan_summary
# إذا ظهر أي clock هنا → مشكلة في DT أو registration
```

---

### 2. الـ ftrace — تتبع الأحداث لحظة بلحظة

**قائمة الـ tracepoints المتاحة:** `clk_enable`, `clk_disable`, `clk_set_rate`, `clk_set_parent`, `clk_set_phase`, `clk_set_duty_cycle`, وجميعها مع `_complete` variant.

```bash
# تفعيل كل الـ clk events
cd /sys/kernel/tracing
echo 1 > events/clk/enable
echo 1 > tracing_on

# شاهد النتائج
cat trace
# <idle>-0  [000] 45.123: clk_set_rate: pll1 600000000
# modprobe-234 [001] 46.234: clk_enable: uart0_clk

# فلتر على clock معين
echo 'name == "pll1"' > events/clk/clk_set_rate/filter

# باستخدام trace-cmd (أسهل)
trace-cmd record -e 'clk:*' sleep 5
trace-cmd report | grep clk
```

---

### 3. الـ Dynamic Debug

```bash
# تفعيل debugging لـ clk.c
echo 'file clk.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل ملفات الـ clk
echo 'file drivers/clk/* +p' > /sys/kernel/debug/dynamic_debug/control

# في الـ kernel cmdline
dyndbg="file drivers/clk/clk.c +pflmt"
```

---

### 4. خيارات الـ Kernel Config

| الـ config option | الوصف |
|------------------|-------|
| `CONFIG_COMMON_CLK` | يفعّل الـ CCF نفسه |
| `CONFIG_CLK_DEBUG` | يفعّل الـ debugfs للـ CCF |
| `CONFIG_DEBUG_FS` | لازم للـ clk_summary |
| `CONFIG_TRACING` | للـ ftrace |
| `CONFIG_LOCKDEP` | يكشف deadlocks |
| `CONFIG_PROVE_LOCKING` | يثبت صحة الـ locking |

---

### 5. جدول رسائل الخطأ الشائعة

| رمز الخطأ | المعنى | الحل |
|-----------|--------|------|
| `-ENOENT` | الـ clock غير موجود | تحقق من الاسم في الـ DT |
| `-EINVAL` | معدل تردد غلط | تأكد إن الـ rate ضمن الـ min/max |
| `-EBUSY` | الـ clock محمي من التغيير | أطلق الـ protection |
| `-EPERM` | gate مغلق يمنع التغيير | افتح الـ gate أولاً |
| `-ERANGE` | الـ rate خارج النطاق | استخدم `clk_round_rate()` |
| `-EPROBE_DEFER` | الـ provider لم يُسجَّل بعد | طبيعي — الكيرنل يعيد المحاولة |

---

## Hardware Level

### 1. قراءة الـ Registers مباشرة

```bash
# باستخدام devmem2
devmem2 0xFE000000 w      # قراءة word (32-bit)
devmem2 0xFE000000 w 0x12345678  # كتابة (خطر!)

# باستخدام io utility
io -4 0xFE000000
```

### 2. Logic Analyzer وـ Oscilloscope

```
ما تراه        → ماذا يعني
─────────────────────────────────
لا إشارة       → الـ clock مطفأ أو مشكلة في gate
تردد خاطئ      → الـ divider مُبرمج غلط
إشارة مشوهة   → مشكلة في الـ trace impedance
duty cycle غير 50% → مشكلة في الـ phase/duty
```

### 3. الـ Device Tree Debugging

```bash
# اقرأ الـ DT المُحمَّل فعلاً
dtc -I fs /proc/device-tree 2>/dev/null > /tmp/current.dts
grep -n 'clocks\|clock-names\|clock-frequency' /tmp/current.dts
```

---

## الأوامر العملية — نسخ وتشغيل مباشرة

```bash
#!/bin/bash
# quick_clk_check.sh
echo "=== Clock Tree Summary ==="
cat /sys/kernel/debug/clk/clk_summary

echo "=== Orphan Clocks (BAD if any!) ==="
cat /sys/kernel/debug/clk/clk_orphan_summary

echo "=== Recent Clock Errors ==="
dmesg | grep -E 'clk.*error|clk.*fail|clk.*timeout' | tail -20
```

```bash
# تفعيل الـ debugging الكامل
mount -t debugfs none /sys/kernel/debug
mount -t tracefs none /sys/kernel/tracing
echo 'file drivers/clk/clk.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 1 > /sys/kernel/tracing/events/clk/enable
echo 1 > /sys/kernel/tracing/tracing_on
```

---

### منهجية الـ Debugging

```
المشكلة: clock لا يعمل كما يجب

الخطوة 1: clk_summary → هل الـ clock موجود في الشجرة؟
    لا → مشكلة DT أو registration
    نعم ↓

الخطوة 2: هل الـ rate صحيح؟
    لا → مشكلة في .recalc_rate() أو الـ registers
    نعم ↓

الخطوة 3: هل enable_count > 0؟
    لا → شخص ما لم يستدعِ clk_enable()
    نعم ↓

الخطوة 4: ftrace → هل الأوامر تصل للـ driver؟
    لا → مشكلة في الـ consumer code
    نعم ↓

الخطوة 5: devmem2 → هل الـ register يطابق الـ kernel state؟
    لا → مشكلة في الـ ops
    نعم ↓

الخطوة 6: oscilloscope على الـ pin → hardware مشكلة؟
```

---

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART صامت على بورد RK3562 صناعي

#### العنوان
**Gateway صناعي لا يطبع أي شيء على Serial Console عند الإقلاع**

#### السياق
شركة تصنع **industrial gateway** مبني على **Rockchip RK3562**. المهندس يعمل على **bring-up** أول نسخة من البورد، وعند تشغيل الجهاز لأول مرة لا يظهر أي شيء على الـ serial console.

#### المشكلة
```
clk_uart2: failed to set rate 1500000 Hz
clk_uart2: prepare failed: -EBUSY
```

#### التحليل
الـ `round_rate` callback مكتوب غلط — يرجع دائماً قيمة أعلى من اللي الـ hardware يقدر يوصلها، فالـ CCF يحاول يسيّت rate مستحيل.

```
clk_set_rate()
    └── clk_calc_new_rates()
         └── ops->round_rate()   ← يرجع قيمة خاطئة!
    └── clk_change_rate()
         └── ops->set_rate()     ← يفشل
```

#### الحل

```bash
# تشخيص
cat /sys/kernel/debug/clk/clk_summary | grep uart
```

```c
// إصلاح round_rate
static long rk3562_uart_clk_round_rate(struct clk_hw *hw,
                                        unsigned long rate,
                                        unsigned long *parent_rate)
{
    unsigned long best_rate = 0;
    unsigned int div;

    for (div = 1; div <= 64; div++) {
        unsigned long r = *parent_rate / div;
        if (r <= rate && r > best_rate)
            best_rate = r;
    }
    return best_rate; // ارجع rate حقيقي وليس خيالي
}
```

#### الدرس المستفاد
الـ `round_rate` callback هو **عقد** بين الـ driver والـ CCF. لو رجّع قيمة لا يمكن تحقيقها، الـ CCF يفشل في صمت. دايماً اختبر الـ `clk_summary` أول شيء.

---

### السيناريو الثاني: شاشة HDMI تومض على Android TV Box بـ Allwinner H616

#### العنوان
**صورة HDMI تومض كل 30 ثانية على TV Box في الإنتاج**

#### المشكلة
الـ Android power manager بيفعّل **runtime PM** كل فترة ويحاول يغلق clocks غير ضرورية:

```
Runtime PM suspend
    └── clk_disable_unused()
         └── إذا refcount == 0 → يغلق الـ clock
              └── HDMI clock OFF → الشاشة تومض!
```

#### الحل

```c
// الحل السريع: أضف CLK_IS_CRITICAL
static CLK_FIXED_FACTOR(tcon_tv_clk,
    "tcon-tv", "pll-video0",
    1, 1,
    CLK_IS_CRITICAL);  // لا تقفله أبداً

// أو: اجعل الـ driver يحتفظ بـ reference
ret = clk_prepare_enable(hdmi->pixel_clk);
// لا تعمل clk_disable — ابق ماسكه
```

```bash
# تحقق
cat /sys/kernel/debug/clk/tcon-tv/clk_enable_count
# يجب أن يكون >= 1 دايماً
```

#### الدرس المستفاد
الـ `CLK_IS_CRITICAL` مش "optional decoration" — هو الفرق بين شاشة مستقرة وشاشة تومض.

---

### السيناريو الثالث: SPI Flash بطيء على STM32MP1 في IoT Device

#### العنوان
**قراءة SPI Flash بسرعة 1 MHz بدلاً من 50 MHz**

#### المشكلة
```bash
dmesg | grep spi
# spi0: setup: can't set 50000000 Hz, using 1000000 Hz
```

الـ PLL4_R output كان **16 MHz** بدلاً من **200 MHz** المفروض. مع divider power-of-two: 16÷1=16 MHz (لكن هذا أقل من 50 MHz) فاختار الـ CCF أبطأ قيمة.

#### الحل

```dts
&rcc {
    assigned-clocks = <&rcc PLL4_R>;
    assigned-clock-rates = <200000000>;
};
```

```bash
# تحقق
cat /sys/kernel/debug/clk/clk_summary | grep spi
# spi1_k    1    1    0    50000000    ← ممتاز!
```

#### الدرس المستفاد
مشاكل الـ clock غالباً مش في الـ driver نفسه بل في الـ clock tree configuration في الـ DT.

---

### السيناريو الرابع: USB لا يعمل على i.MX8 بعد Deep Sleep

#### العنوان
**USB لا يُكتشف على i.MX8M في ECU للسيارات بعد Deep Sleep**

#### المشكلة
```bash
dmesg | grep usb
# usb usb1-port1: cannot reset (err = -110)
```

الـ resume function نسي تفعيل PHY clock:

```c
// الكود الغلط
static int xhci_imx8_resume(struct device *dev)
{
    clk_prepare_enable(priv->core_clk);  // صح
    // نسي phy_ref_clk!
    xhci_resume(hcd, 0);  // USB لا يستجيب
}
```

#### الحل

```c
static int xhci_imx8_resume(struct device *dev)
{
    clk_prepare_enable(priv->core_clk);
    clk_prepare_enable(priv->phy_ref_clk);  // الإضافة
    clk_prepare_enable(priv->ahb_clk);
    return xhci_resume(hcd, 0);
}
```

#### الدرس المستفاد
في suspend/resume cycle، الـ clocks لا تعود تلقائياً. كل driver مسؤول عن إعادة تفعيل **كل** الـ clocks.

---

### السيناريو الخامس: I2C يفشل على AM62x أثناء Board Bring-up

#### العنوان
**I2C لا يعمل أبداً على custom board بـ TI AM62x رغم صح DTS**

#### المشكلة
```bash
cat /sys/kernel/debug/clk/clk_summary | grep i2c
# main_i2c0_fclk    0    0    0    0    0    0    0
#                   ↑ disabled و rate = 0!
```

الـ `compatible` string في الـ DTS مختلف عن الـ `CLK_OF_DECLARE`:

```c
// في driver:
CLK_OF_DECLARE(ti_k3_clk, "ti,k3-clk", ti_k3_clk_probe);

// في DTS:
compatible = "ti,am625-clk";  // ← خطأ! مختلف
```

#### الحل

```dts
clock-controller@400000 {
    compatible = "ti,am625-clk", "ti,k3-clk";  // أضف fallback
};
```

```bash
# بعد الإصلاح
cat /sys/kernel/debug/clk/clk_summary | grep i2c
# main_i2c0_fclk    1    1    0    48000000  ← ✓

i2cdetect -y 0  # الآن يعمل!
```

#### الدرس المستفاد
الـ `CLK_OF_DECLARE` و`of_clk_add_hw_provider` هما الجسر بين الـ clock driver والـ DT. أي خطأ في الـ `compatible` string — حتى حرف واحد — يعني إن الـ clock controller غير موجود.

---

## Phase 7: مصادر ومراجع

---

### توثيق الـ Kernel الرسمي

| الملف | الوصف |
|---|---|
| [`Documentation/driver-api/clk.rst`](https://docs.kernel.org/driver-api/clk.html) | التوثيق الرسمي الأحدث للـ CCF |
| `Documentation/devicetree/bindings/clock/` | كيف تربط الـ clocks بالـ Device Tree |
| `drivers/clk/clk.c` | قلب الـ CCF — الكود الفعلي |
| `include/linux/clk-provider.h` | هذا الملف |
| `include/linux/clk.h` | واجهة الـ consumer |

---

### مقالات LWN.net

**[A common clock framework — LWN.net (2012)](https://lwn.net/Articles/472998/)**
> أول مقال تفصيلي يشرح الفكرة من صفر. **ابدأ من هنا.**

**[common clk framework — LWN.net (2012)](https://lwn.net/Articles/486841/)**
> تغطية للـ patch series الذي دخل الـ kernel رسميًا.

**[Add a generic struct clk — LWN.net (2011)](https://lwn.net/Articles/460193/)**
> العمل الأولي لـ Jeremy Kerr قبل ما دخل Turquette. مهم لفهم التاريخ.

---

### الـ Mailing List والـ Patches التاريخية

**[PATCH v7 2/3: clk: introduce the common clock framework](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html)**
> الـ patch الذي أنشأ `clk-provider.h` من Mike Turquette في مارس 2012.

**[lore.kernel.org/linux-clk](https://lore.kernel.org/linux-clk/)**
> الـ mailing list الرسمي — كل patch وكل نقاش منذ 2013.

---

### عروض المؤتمرات

**[Bootlin — Common clock framework: how to use it (ELC 2013)](https://bootlin.com/pub/conferences/2013/elc/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf)**
> من أفضل الـ presentations العملية. أمثلة كاملة على كتابة `clk_ops`.

**[eLinux.org — Common Clock Framework](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf)**
> Slides من Birds-of-a-Feather session من الفريق نفسه.

---

### الكود المرجعي في الـ Kernel

```bash
# أبسط مثال
drivers/clk/clk-fixed-rate.c

# gate, mux, divider
drivers/clk/clk-gate.c
drivers/clk/clk-mux.c
drivers/clk/clk-divider.c

# قلب الـ framework
drivers/clk/clk.c
```

**[include/linux/clk-provider.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/clk-provider.h)**

---

### الكتب المرجعية

- **Linux Device Drivers, 3rd Edition (LDD3)** — [متاح مجانًا](https://lwn.net/Kernel/LDD3/)
- **Linux Kernel Development, 3rd Edition** — Robert Love
- **Embedded Linux Primer, 2nd Edition** — Christopher Hallinan

---

### كيف تبحث بنفسك

```bash
# على lore.kernel.org
https://lore.kernel.org/linux-clk/

# على git.kernel.org
git log --follow -p include/linux/clk-provider.h
git log --all --grep="clk: introduce"

# على Elixir (قراءة الكود بروابط)
https://elixir.bootlin.com/linux/latest/source/include/linux/clk-provider.h
```

**مصطلحات البحث:**
```
"common clock framework" linux kernel
"clk_hw" linux driver implementation
"clk_ops" linux custom clock
CCF linux ARM SoC clock driver
```

---

### ملخص سريع للمصادر حسب هدفك

| إذا كنت تريد... | اذهب إلى |
|---|---|
| تفهم الفكرة من البداية | [LWN.net 472998](https://lwn.net/Articles/472998/) |
| تكتب clock driver جديد | [docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html) |
| تفهم لماذا التصميم هكذا | [PATCH v7 على LKML](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) |
| تتابع أحدث التغييرات | [lore.kernel.org/linux-clk](https://lore.kernel.org/linux-clk/) |
| تقرأ كود مثالي | `drivers/clk/clk-gate.c` + `drivers/clk/clk-fixed-rate.c` |

---

## Phase 8: Writing simple module

### ماذا سنبني؟

سنكتب **kernel module** كامل يستخدم **kprobe** لاعتراض دالة `clk_hw_register` — يعني كل مرة يسجّل أي **clock** نفسه في النظام، موديولنا يعرف ويطبع اسمه وـ**flags** بتاعته.

ليه هذا مفيد؟ لأن عند الـ**boot**، المئات من الـ**clocks** تسجّل نفسها. هذا الموديول يعطيك عين داخل هذه العملية بدون ما تلمس الـ**kernel source** أو تعيد الـ**compile**.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * clk_spy.c — kprobe على clk_hw_register
 *
 * يعترض كل clock يسجّل نفسه في النظام
 * ويطبع اسمه وـflags بتاعته في kernel log
 */

/* ① الـ includes الأساسية */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info, pr_err */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/clk-provider.h> /* struct clk_hw, struct clk_init_data */
#include <linux/ptrace.h>       /* struct pt_regs */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("مبرمج عربي <dev@example.com>");
MODULE_DESCRIPTION("يتجسس على clk_hw_register باستخدام kprobe");

/* ② مساعد لاستخراج clk_hw من رجيستر المعالج
 *
 * توقيع الدالة: int clk_hw_register(struct device *dev, struct clk_hw *hw)
 * x86_64: الأول=RDI, الثاني=RSI
 * ARM64:  الأول=x0,  الثاني=x1
 */
static struct clk_hw *get_hw_from_regs(struct pt_regs *regs)
{
#if defined(CONFIG_X86_64)
    return (struct clk_hw *)regs->si;
#elif defined(CONFIG_ARM64)
    return (struct clk_hw *)regs->regs[1];
#else
    return NULL;
#endif
}

/* ③ الـ pre_handler — يُستدعى قبل تنفيذ clk_hw_register */
static int clk_spy_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    struct clk_hw *hw;
    const struct clk_init_data *init;
    const char *clk_name = "(unknown)";
    unsigned long flags = 0;

    hw = get_hw_from_regs(regs);
    if (!hw)
        return 0;

    init = hw->init;
    if (init) {
        if (init->name)
            clk_name = init->name;
        flags = init->flags;
    }

    pr_info("clk_spy: [+] clock registered → name=%-30s flags=0x%08lx\n",
            clk_name, flags);

    /* نفسّر الـ flags الشائعة من clk-provider.h */
    if (flags & CLK_SET_RATE_PARENT)
        pr_info("clk_spy:      ↳ CLK_SET_RATE_PARENT\n");
    if (flags & CLK_IGNORE_UNUSED)
        pr_info("clk_spy:      ↳ CLK_IGNORE_UNUSED\n");
    if (flags & CLK_IS_CRITICAL)
        pr_info("clk_spy:      ↳ CLK_IS_CRITICAL\n");
    if (flags & CLK_GET_RATE_NOCACHE)
        pr_info("clk_spy:      ↳ CLK_GET_RATE_NOCACHE\n");

    return 0;
}

/* ④ تعريف الـ kprobe */
static struct kprobe clk_spy_kp = {
    .symbol_name = "clk_hw_register",
    .pre_handler = clk_spy_pre_handler,
};

/* ⑤ module_init */
static int __init clk_spy_init(void)
{
    int ret;

    ret = register_kprobe(&clk_spy_kp);
    if (ret < 0) {
        pr_err("clk_spy: فشل تسجيل الـkprobe: خطأ %d\n", ret);
        return ret;
    }

    pr_info("clk_spy: تم التحميل — يراقب clk_hw_register عند العنوان %p\n",
            clk_spy_kp.addr);
    return 0;
}

/* ⑥ module_exit — unregister ضروري لمنع kernel panic */
static void __exit clk_spy_exit(void)
{
    unregister_kprobe(&clk_spy_kp);
    pr_info("clk_spy: تمت الإزالة\n");
}

module_init(clk_spy_init);
module_exit(clk_spy_exit);
```

---

### شرح كل قسم

#### ① الـ includes — لماذا كل واحد؟

| الـ include | السبب |
|---|---|
| `linux/module.h` | بدونه لا يُعرَّف `module_init` ولا `MODULE_LICENSE` |
| `linux/kprobes.h` | يعرّف `struct kprobe` و`register_kprobe` |
| `linux/clk-provider.h` | لقراءة `struct clk_hw` والـ`CLK_*` flags |
| `linux/ptrace.h` | `struct pt_regs` — يحمل قيم الـ CPU registers |

---

#### ② استخراج الوسيطات من الـ registers

الـ **calling convention** يختلف بحسب المعمارية:
```
x86_64:  الأول = RDI,  الثاني = RSI
ARM64:   الأول = x0,   الثاني = x1
```

نحن نريد الوسيط الثاني `hw`، لذلك نقرأ `RSI` أو `x1`.

---

#### ③ الـ pre_handler — قلب الموديول

```
clk_hw_register() يُستدعى
        │
        ▼
   [kprobe يعترض]
        │
        ▼
  clk_spy_pre_handler()   ← نحن هنا، نطبع البيانات
        │
        ▼
  clk_hw_register() تكمل طبيعياً
```

نرجع `0` دايماً لأن أي قيمة أخرى قد تسبب مشاكل.

---

#### ④ كيف يعمل الـ kprobe؟

```
register_kprobe() يجد عنوان clk_hw_register في الـ symbol table
         │
         ▼
يستبدل أول byte بـ INT3 (x86) أو BRK (ARM64)
         │
         ▼
كل مرة يصل CPU لهذا العنوان → trap → pre_handler → يكمل
```

لا **recompile** ولا **reboot** — تعديل مباشر في الذاكرة!

---

#### ⑥ لماذا unregister ضروري في exit؟

```
بعد rmmod بدون unregister:
  clk_hw_register → [INT3 trap] → يقفز لعنوان في ذاكرة محذوفة → 💥 PANIC
```

---

### الـ Makefile

```makefile
obj-m += clk_spy.o

KDIR := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### كيف تشغّله؟

```bash
# بناء الموديول
make

# تحميله
sudo insmod clk_spy.ko

# مشاهدة الـ log
dmesg | grep clk_spy

# مثال على الـ output:
# [2.341] clk_spy: تم التحميل — يراقب clk_hw_register
# [2.389] clk_spy: [+] clock registered → name=apll0             flags=0x00000000
# [2.392] clk_spy: [+] clock registered → name=sys_clk           flags=0x00000024
# [2.392] clk_spy:      ↳ CLK_IS_CRITICAL
# [2.392] clk_spy:      ↳ CLK_GET_RATE_NOCACHE

# إزالة الموديول بأمان
sudo rmmod clk_spy
```

---

### تحذيرات مهمة

| الموقف | الخطر | الحل |
|---|---|---|
| قراءة `hw->init->name` بدون تحقق | NULL pointer dereference | `if (init && init->name)` |
| عدم `unregister_kprobe` في exit | kernel panic عند rmmod | دائماً `unregister` في exit |
| استخدام `regs->si` على ARM64 | قراءة بيانات خاطئة | `#ifdef` للمعمارية |
| طباعة كثيرة جداً | يبطّئ الـ boot | استخدم `pr_debug` + `dyndbg` |
