## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **Common Clock Framework (CCF)** — الـ subsystem الرسمي في Linux kernel اللي بيتحكم في كل الـ clocks الجوه الـ SoC. المسؤولين عنه: Michael Turquette و Stephen Boyd، والـ mailing list هو `linux-clk@vger.kernel.org`.

---

### قبل أي كود — إيه هو الـ Clock أصلاً؟

تخيل الـ CPU زي عامل في مصنع. العامل ده محتاج **إيقاع** (طبّالة) يشتغل على إيقاعها — يمشي بسرعة لما الطبّالة تضرب بسرعة، ويمشي ببطء لما تبطّأ. الـ **clock signal** هو الطبّالة دي — نبضات منتظمة بتقول للدوائر الإلكترونية "روح خطوة".

كل جهاز في الـ SoC (CPU, GPU, USB, UART, I2C, ...) محتاج **تردد معين** عشان يشتغل صح:
- الـ CPU ممكن يحتاج 1.8 GHz
- الـ UART محتاج 48 MHz
- الـ RTC محتاج 32.768 kHz

---

### القصة: من فين بيجي الـ Clock؟

```
[Crystal Oscillator 24MHz]  ← مصدر ثابت خارجي
        |
        ↓
    [PLL]  ← بيضرّب التردد لـ 1GHz+
        |
   ┌────┴────┐
   ↓         ↓
[Divider]  [Gate]  ← بيقسّم أو بيقطع الـ clock
   |
   ↓
[CPU / Peripheral]
```

في أسفل الشجرة دي، في مصدر **ثابت لا يتغير** — زي الـ crystal oscillator أو الـ oscillator داخل الـ SoC. ده اللي بيمثّله ملف `clk-fixed-rate.c`.

---

### إيه هو `clk-fixed-rate.c` بالظبط؟

ده تطبيق لأبسط نوع من الـ clocks في الـ kernel: **fixed-rate clock** — ساعة تردّدها **ثابت لا يتغير** أبداً. مفيش `set_rate`، مفيش `gate`، مفيش `mux`. بس رقم واحد: "أنا بشتغل على X Hz، خلاص."

**الـ fixed_rate clock بيعمل إيه؟**
- بيرجّع تردده الثابت لما أي حد يسأله
- بيرجّع accuracy (دقة التردد بـ ppb) لو اتحددت
- مش ممكن تقفله (no gate) ومش ممكن تغيّر تردده (no set_rate)

---

### ليه ده مهم؟ مثال حقيقي

**Crystal oscillator** على الـ PCB بيشتغل على 24 MHz. الـ kernel محتاج يعرف ده عشان يبني فوقيه كل الـ PLL tree.

في الـ Device Tree:
```dts
clk_24mhz: oscillator {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;  /* 24 MHz */
};
```

الـ kernel لما يقرأ الـ DT، بيستدعي `of_fixed_clk_setup()` اللي موجودة في الملف ده، بتعمل `clk_hw_register_fixed_rate_with_accuracy()` وبتسجّل الـ clock في الـ CCF بـ تردد 24 MHz ثابت للأبد.

بعد كده أي PLL أو divider جوه الـ SoC ممكن يقول "أنا parent-ي هو clk_24mhz" — ويبني تردده بناءً عليه.

---

### الـ struct الأساسي

```c
struct clk_fixed_rate {
    struct clk_hw hw;          /* handle للـ CCF framework */
    unsigned long fixed_rate;  /* التردد الثابت بـ Hz */
    unsigned long fixed_accuracy; /* الدقة بـ ppb */
    unsigned long flags;       /* CLK_FIXED_RATE_PARENT_ACCURACY */
};
```

الـ `clk_hw` هو الـ "جواز سفر" اللي بيربط الـ hardware-specific struct بالـ common framework.

---

### الـ ops — بساطة مطلقة

```c
const struct clk_ops clk_fixed_rate_ops = {
    .recalc_rate     = clk_fixed_rate_recalc_rate,     /* ارجع fixed_rate */
    .recalc_accuracy = clk_fixed_rate_recalc_accuracy, /* ارجع fixed_accuracy */
    /* مفيش enable, disable, set_rate, set_parent... */
};
```

مقارنةً بالـ `struct clk_ops` الكاملة اللي فيها 15+ callback، ده أبسط تطبيق ممكن.

---

### مسارات التسجيل

الملف فيه **3 طرق** لتسجيل الـ fixed-rate clock:

| الطريقة | الاستخدام |
|---------|-----------|
| `clk_hw_register_fixed_rate()` | driver عادي بيسجّل يدوياً |
| `devm_clk_hw_register_fixed_rate()` | نفسه بس مع **devres** (auto cleanup) |
| `of_fixed_clk_setup()` / `CLK_OF_DECLARE` | من الـ Device Tree مباشرة في early boot |

الـ `__clk_hw_register_fixed_rate()` هي الـ core function اللي الكل بيستدعيها — بتعمل alloc، تعبئ الـ struct، وتسجّل في الـ CCF.

---

### الـ devres pattern

```c
/* لو devm=true، بيستخدم devres_alloc بدل kzalloc */
/* لما الـ device يتشال، الـ framework بيستدعي تلقائياً: */
static void devm_clk_hw_register_fixed_rate_release(struct device *dev, void *res)
{
    clk_hw_unregister(&fix->hw);
    /* الـ kfree هيعمله devres تلقائياً */
}
```

ده بيحل مشكلة الـ memory leak لو الـ driver اتشال.

---

### الـ OF (Device Tree) Integration

```c
CLK_OF_DECLARE(fixed_clk, "fixed-clock", of_fixed_clk_setup);
```

السطر ده بيقول للـ kernel: "لما تلاقي node بـ `compatible = "fixed-clock"` في الـ DT، استدعي `of_fixed_clk_setup`."

في early boot، الـ `of_fixed_clk_setup` بتشتغل مباشرة. لو الـ system بيستخدم platform driver model، الـ `of_fixed_clk_probe` بتشتغل بدلاً منها.

---

### الملفات المهمة اللي المفروض تعرفها

| الملف | الدور |
|-------|-------|
| `drivers/clk/clk.c` | **قلب الـ CCF** — الـ framework نفسه، تسجيل الـ clocks وإدارة الـ tree |
| `drivers/clk/clk-fixed-rate.c` | **الملف ده** — أبسط نوع clock |
| `drivers/clk/clk-fixed-factor.c` | clock ثابت النسبة من الـ parent (مش fixed-rate) |
| `drivers/clk/clk-divider.c` | clock بيقسّم تردد الـ parent بمقسوم قابل للضبط |
| `drivers/clk/clk-gate.c` | clock يمكن تشغيله أو إيقافه فقط |
| `drivers/clk/clk-mux.c` | clock بيختار من بين عدة parents |
| `drivers/clk/clk-fixed-mmio.c` | fixed-rate بس التردد بيتقرأ من MMIO register |
| `drivers/clk/clk-fixed-rate_test.c` | unit tests للملف ده |
| `include/linux/clk-provider.h` | التعريفات: `struct clk_hw`, `struct clk_ops`, `struct clk_fixed_rate` |
| `include/linux/clk.h` | الـ consumer API: `clk_get`, `clk_enable`, `clk_set_rate` |
| `Documentation/devicetree/bindings/clock/fixed-clock.yaml` | الـ DT binding لـ `fixed-clock` |
## Phase 2: شرح الـ Common Clock Framework (CCF)

### المشكلة — ليه الـ Subsystem ده موجود؟

في أي SoC حديث زي i.MX8 أو RK3588، في المئات من الـ clocks — كل peripheral بياخد clock: الـ UART، الـ SPI، الـ USB، الـ GPU، الـ DDR. قبل الـ CCF (قبل kernel 3.4)، كل platform بتعمل clock management بطريقتها الخاصة — كود مكرر، inconsistent API، ومفيش طريقة موحدة تعمل `clk_set_rate()` أو `clk_enable()` بدون ما تعرف التفاصيل الداخلية للـ SoC.

**المشكلة الجوهرية**:
- كل driver عايز يعمل `clk_enable()` بدون ما يعرف هو clock oscillator ولا PLL ولا divider.
- الـ clock tree معقدة — clock A مش ممكن يتغير rate إلا لو parent B يتغير الأول.
- محتاج نعرف كل consumer بياخد clock منين عشان نعمل power gating صح.

---

### الحل — الـ Kernel اتخد إيه؟

الـ **Common Clock Framework (CCF)** بيوفر:

1. **Unified consumer API** — أي driver يكلم `clk_get()`, `clk_enable()`, `clk_set_rate()` بدون ما يهتم بالـ hardware.
2. **Provider abstraction** — كل نوع clock (fixed, gate, divider, mux, PLL) بيعمل implement لـ `struct clk_ops`.
3. **Clock tree management** — الـ framework بيبني شجرة تبعيات بين الـ clocks ويتحكم في propagation.
4. **Reference counting** — `clk_enable/disable` بيستخدم reference count عشان يعرف امتى يطفي الـ clock فعلاً.

---

### الـ Real-World Analogy — شبكة الكهرباء

تخيل شبكة كهرباء في مدينة:

| مفهوم الكهرباء | مقابله في CCF |
|---|---|
| محطة توليد (power plant) | Crystal oscillator — مصدر الـ clock الأساسي |
| محولات كهرباء (transformers) | PLLs و dividers — بيغيروا الـ frequency |
| مفاتيح رئيسية (circuit breakers) | Gate clocks — بتفتح/تقفل الـ clock |
| اختيار مصدر (switch between sources) | Mux clocks — بتختار من أي parent |
| كابل ثابت بجهد ثابت | **Fixed-rate clock** — مش ممكن تغير rate وبيوصل نفس الـ frequency دايمًا |
| عداد الكهرباء (meter) | Consumer driver — بيستخدم الـ clock |
| شركة الكهرباء (grid manager) | CCF core — بيدير الشجرة كلها |

الـ **fixed-rate clock** زي كابل كهرباء مباشر من المحطة — مفيش transformer، مفيش switch، الجهد ثابت دايمًا. السبورة (CCF) بتعرف إنه موجود وبتديه للـ consumers، بس مش بتحاول تغيره.

الـ mapping مش surface level:
- لما `clk_enable()` بتتعمل على fixed clock → CCF بيتأكد إن الـ parent (لو موجود) enabled الأول، تماماً زي ما مش تقدر تشغل المصنع إلا لو الشبكة الرئيسية شغالة.
- الـ `recalc_rate()` على fixed clock → بترجع القيمة المخزنة مباشرة، زي ما عداد الكهرباء بيقرأ جهد ثابت مش بيحسبه.
- الـ `accuracy` بـ ppb → زي دقة الـ voltage regulation: ±50ppm يعني في 50MHz clock هتلاقي انحراف ≈2500Hz.

---

### الـ Big Picture Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           Consumer Drivers               │
                    │  (UART driver, SPI driver, USB driver)  │
                    └──────────────┬──────────────────────────┘
                                   │ clk_get(), clk_enable(),
                                   │ clk_set_rate(), clk_put()
                    ┌──────────────▼──────────────────────────┐
                    │         Consumer API Layer               │
                    │         include/linux/clk.h              │
                    │   (struct clk — per-user handle)         │
                    └──────────────┬──────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────┐
                    │       CCF Core  (drivers/clk/clk.c)      │
                    │  - Clock tree (linked list/hashtable)    │
                    │  - Reference counting                    │
                    │  - Rate propagation                      │
                    │  - prepare_lock / enable_lock            │
                    │  - debugfs (/sys/kernel/debug/clk/)      │
                    └──┬──────┬──────┬──────┬─────────────────┘
                       │      │      │      │
              ┌────────▼─┐ ┌──▼───┐ ┌▼───┐ ┌▼──────────┐
              │Fixed Rate│ │ Gate │ │ Mux│ │  Divider  │
              │clk-fixed │ │clk-  │ │clk-│ │  clk-     │
              │-rate.c   │ │gate.c│ │mux │ │  divider.c│
              └────────┬─┘ └──┬───┘ └┬───┘ └┬──────────┘
                       │      │      │      │
                       └──────┴──────┴──────┘
                                   │  struct clk_hw + struct clk_ops
                    ┌──────────────▼──────────────────────────┐
                    │       Platform Clock Drivers             │
                    │  (drivers/clk/imx/, clk/rockchip/, etc) │
                    │   يعملوا register لـ clock tree          │
                    └──────────────┬──────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────┐
                    │           Hardware (SoC)                 │
                    │  Crystal OSC → PLL → Dividers → Gates   │
                    └─────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ CCF بيبني كل حاجة حواليه **ثلاث structs محورية**:

```
struct clk_fixed_rate          struct clk_hw           struct clk_core
┌──────────────────┐          ┌─────────────┐         ┌──────────────────┐
│ hw  ◄────────────┼──────────► core ────────┼────────► name             │
│ fixed_rate       │          │ clk         │         │ ops              │
│ fixed_accuracy   │          │ init ───────┼──┐      │ rate             │
│ flags            │          └─────────────┘  │      │ parent           │
└──────────────────┘                           │      │ children list    │
   Hardware-specific                     ┌─────▼────┐ │ enable_count     │
   data (private)                        │clk_init_ │ │ prepare_count    │
                                         │data      │ └──────────────────┘
                                         │ name     │    CCF internal state
                                         │ ops ─────┼──► struct clk_ops
                                         │ flags    │    (vtable)
                                         │ parents  │
                                         └──────────┘
                                           Shared init data
                                           (freed after register)
```

**الـ `container_of` trick** هو القلب:

```c
#define to_clk_fixed_rate(_hw) container_of(_hw, struct clk_fixed_rate, hw)
```

الـ CCF بيمرر `struct clk_hw *hw` لكل callback، والـ driver بيرجع للـ parent struct اللي فيه الـ hardware-specific data. ده نفس الـ pattern اللي بيستخدمه الـ kobject subsystem والـ device model.

---

### الـ Fixed-Rate Clock — أبسط تطبيق ممكن

الـ `clk-fixed-rate.c` بيمثل الـ **leaf node الأبسط** في الـ clock tree — clock مصدره إما crystal خارجي أو oscillator داخلي، وقيمته ثابتة مش ممكن تتغير.

**الـ `struct clk_ops` المُنفَّذة**:

```c
const struct clk_ops clk_fixed_rate_ops = {
    .recalc_rate      = clk_fixed_rate_recalc_rate,      /* يرجع fixed_rate مباشرة */
    .recalc_accuracy  = clk_fixed_rate_recalc_accuracy,  /* يرجع fixed_accuracy أو من parent */
};
```

**ليه مفيش `.enable` / `.disable` / `.set_rate`؟**

- `.enable` / `.disable`: مش موجودين → الـ CCF بيتعامل مع الـ enable/disable عن طريق propagation للـ parent. الـ fixed clock نفسه مش بيتطفش — الـ hardware اللي ورا الـ crystal مش قابل للـ gate بشكل عام.
- `.set_rate`: غير موجود → لو حد استدعى `clk_set_rate()` على fixed clock، الـ CCF بيرفضها لأنه مفيش `CLK_SET_RATE_PARENT` flag وإمكانية `round_rate` أو `determine_rate`.
- `.round_rate` / `.determine_rate`: مش موجودين → يعني الـ CCF مش هيقدر يعمل rate negotiation، وده صح لأن الـ rate ثابت.

---

### مسار التسجيل الكامل

```c
/* مثال: تسجيل 24MHz crystal في imx8 platform */
clk_hw_register_fixed_rate(dev, "osc_24m", NULL, 0, 24000000);

/* اللي بيحصل داخلياً في __clk_hw_register_fixed_rate() */
```

```
  kzalloc(struct clk_fixed_rate)
         │
         ▼
  fill clk_init_data:
    .name = "osc_24m"
    .ops  = &clk_fixed_rate_ops
    .num_parents = 0   (root clock, no parent)
         │
         ▼
  fixed->fixed_rate = 24000000
  fixed->hw.init = &init
         │
         ▼
  clk_hw_register(dev, &fixed->hw)
         │
         ▼  [inside CCF core - clk.c]
  alloc struct clk_core
  build clock tree node
  hash by name ("osc_24m")
  hw->init = NULL  (freed reference, no longer needed)
         │
         ▼
  clk_core stored in global clock tree
  accessible via clk_get("osc_24m")
```

---

### الـ Device Tree Integration

الـ fixed-rate clock مدعوم مباشرة من الـ DT بدون أي platform-specific code:

```dts
/* في device tree */
osc_24m: clock-osc-24m {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;
    clock-accuracy = <50000>;   /* 50000 ppb = 50 ppm */
    clock-output-names = "osc_24m";
};

/* consumer */
uart0: serial@30860000 {
    clocks = <&osc_24m>;
};
```

الـ kernel بيعمل الآتي:
1. `CLK_OF_DECLARE(fixed_clk, "fixed-clock", of_fixed_clk_setup)` → يسجل handler لـ `compatible = "fixed-clock"` أثناء الـ early boot.
2. `of_fixed_clk_setup()` → تقرأ `clock-frequency` و `clock-accuracy` من الـ DT وتسجل الـ clock.
3. `of_clk_add_hw_provider()` → تربط الـ DT node بالـ `clk_hw` عشان الـ consumers يقدروا يعملوا `clk_get()` باستخدام الـ phandle.

**ليه في `probe()` وفي `of_fixed_clk_setup()` الاتنين؟**

الـ comment في الكود صريح:
```c
/*
 * This function is not executed when of_fixed_clk_setup
 * succeeded.
 */
```

الـ `CLK_OF_DECLARE` بيشتغل في الـ early boot (قبل الـ driver subsystem). الـ `platform_driver` بيشتغل لو الـ early setup فشل أو في سيناريوهات معينة. ده **defensive design** — ضمان إن الـ clock متسجلش مرتين.

---

### الـ `devm` Pattern والـ Resource Management

```c
/* الفرق بين الاتنين */

/* 1. Manual management */
hw = clk_hw_register_fixed_rate(dev, "my_clk", NULL, 0, 26000000);
/* لازم تعمل في remove(): */
clk_hw_unregister_fixed_rate(hw);  /* بيعمل kfree للـ struct */

/* 2. Managed (devm) */
hw = devm_clk_hw_register_fixed_rate(dev, "my_clk", NULL, 0, 26000000);
/* الـ devres framework بيعمل unregister + kfree تلقائياً لما الـ device يتحل */
```

الـ `devm` بيستخدم `devres_alloc()` — بيحجز الـ `struct clk_fixed_rate` نفسه كـ devres resource مع callback `devm_clk_hw_register_fixed_rate_release`. لما الـ device يتحل، الـ devres code بيعمل `clk_hw_unregister()` ثم `kfree()`. **ملاحظة مهمة**: الـ release callback بيعمل `clk_hw_unregister()` وليس `clk_hw_unregister_fixed_rate()`، لأن الأخير بيعمل `kfree()` وده سيعمل double-free مع الـ devres.

---

### إيه اللي الـ CCF بيملكه vs إيه اللي بيفوضه للـ Driver

| المسؤولية | CCF Core (clk.c) | Driver (clk-fixed-rate.c) |
|---|---|---|
| Clock tree بناء وصيانة | نعم | لا |
| Reference counting (enable/prepare) | نعم | لا |
| Rate propagation عبر الشجرة | نعم | لا |
| Locking (prepare_lock, enable_lock) | نعم | لا |
| debugfs entries | نعم (generic) | اختياري (.debug_init) |
| قراءة الـ rate من الـ hardware | لا | **نعم** (.recalc_rate) |
| تغيير الـ rate في الـ hardware | لا | **نعم** (.set_rate) |
| Enable/Disable الـ hardware gate | لا | **نعم** (.enable/.disable) |
| تحديد الـ accuracy | لا | **نعم** (.recalc_accuracy) |
| الـ hardware-specific state | لا | **نعم** (struct clk_fixed_rate) |

الـ fixed-rate clock بيفوض **أقل حاجة ممكنة** — بس الـ `recalc_rate` والـ `recalc_accuracy`، لأنه hardware أبسط حاجة موجودة: مفيش gate، مفيش mux، مفيش divider.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### الـ CLK Framework Flags (عامة على كل الـ clocks)

| Flag | Bit | المعنى |
|------|-----|--------|
| `CLK_SET_RATE_GATE` | BIT(0) | لازم الـ clock يتوقف وقت تغيير الـ rate |
| `CLK_SET_PARENT_GATE` | BIT(1) | لازم يتوقف وقت تغيير الـ parent |
| `CLK_SET_RATE_PARENT` | BIT(2) | ارفع طلب تغيير الـ rate للـ parent |
| `CLK_IGNORE_UNUSED` | BIT(3) | متوقفوش حتى لو مش متستخدم |
| `CLK_GET_RATE_NOCACHE` | BIT(6) | اسأل الـ HW دايماً، متستخدمش الـ cache |
| `CLK_SET_RATE_NO_REPARENT` | BIT(7) | متعملش reparent عند تغيير الـ rate |
| `CLK_GET_ACCURACY_NOCACHE` | BIT(8) | اسأل الـ HW عن الـ accuracy بدون cache |
| `CLK_RECALC_NEW_RATES` | BIT(9) | أعد حساب الـ rates بعد الـ notifications |
| `CLK_SET_RATE_UNGATE` | BIT(10) | الـ clock لازم يشتغل عشان يتغير الـ rate |
| `CLK_IS_CRITICAL` | BIT(11) | متوقفوش أبداً |
| `CLK_OPS_PARENT_ENABLE` | BIT(12) | فعّل الـ parents وقت الـ gate/ungate |
| `CLK_DUTY_CYCLE_PARENT` | BIT(13) | ابعت طلب الـ duty cycle للـ parent |

#### الـ CLK_FIXED_RATE Flags (خاص بالـ fixed-rate clock)

| Flag | Bit | المعنى |
|------|-----|--------|
| `CLK_FIXED_RATE_PARENT_ACCURACY` | BIT(0) | استخدم accuracy الـ parent بدل `fixed_accuracy` |

#### الـ CONFIG Options المؤثرة

| Config | الأثر |
|--------|-------|
| `CONFIG_OF` | يفعّل كود الـ Device Tree (الـ `_of_fixed_clk_setup`، الـ probe/remove، الـ `CLK_OF_DECLARE`) |

---

### الـ Structs الأساسية

#### 1. `struct clk_fixed_rate`

**الهدف:** الـ struct الرئيسي اللي يمثل الـ fixed-rate clock — كل بياناته ثابتة في الـ init وما بتتغيرش بعد كده.

```c
struct clk_fixed_rate {
    struct clk_hw hw;            // نقطة الوصل مع الـ CCF (Common Clock Framework)
    unsigned long fixed_rate;    // التردد الثابت بالـ Hz
    unsigned long fixed_accuracy;// الدقة بالـ ppb (parts per billion)
    unsigned long flags;         // flags خاصة بالـ fixed-rate (CLK_FIXED_RATE_*)
};
```

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `hw` | `struct clk_hw` | الـ bridge بين الـ driver code والـ CCF |
| `fixed_rate` | `unsigned long` | الـ rate الوحيد اللي بيرجعه `recalc_rate` دايماً |
| `fixed_accuracy` | `unsigned long` | الدقة بالـ ppb، يتجاهلها لو `CLK_FIXED_RATE_PARENT_ACCURACY` مفعّل |
| `flags` | `unsigned long` | بس `CLK_FIXED_RATE_PARENT_ACCURACY` مستخدم حالياً |

**العلاقات:** الـ `hw` field هو المدخل الوحيد للـ CCF — كل الـ ops بتاخد `struct clk_hw *hw` وبتعمل `container_of` ترجع للـ `clk_fixed_rate`.

---

#### 2. `struct clk_hw`

**الهدف:** الـ handle اللي يربط الـ hardware-specific struct بالـ CCF. بيكون مضمّن (embedded) جوا كل `struct clk_foo`.

```c
struct clk_hw {
    struct clk_core *core;           // الـ CCF internal representation
    struct clk *clk;                 // الـ per-user clock handle
    const struct clk_init_data *init;// بيانات الـ init — بتتمسح بعد التسجيل
};
```

| Field | الوظيفة |
|-------|---------|
| `core` | بيملاه الـ CCF بعد `clk_hw_register` — فيه كل الـ tree logic |
| `clk` | الـ handle اللي بيطلع للـ consumer drivers |
| `init` | مؤقت — بيُقرأ وقت التسجيل وبعدين بيتحوّل لـ NULL |

---

#### 3. `struct clk_init_data`

**الهدف:** بيانات الـ init المشتركة بين الـ provider والـ CCF — بس مؤقتة وقت التسجيل.

```c
struct clk_init_data {
    const char              *name;         // اسم الـ clock في الـ tree
    const struct clk_ops    *ops;          // الـ callbacks — هنا clk_fixed_rate_ops
    const char * const      *parent_names; // أسماء الـ parents (string-based، legacy)
    const struct clk_parent_data *parent_data; // parent data (modern)
    const struct clk_hw     **parent_hws;  // pointers مباشرة للـ parents (internal)
    u8                       num_parents;  // 0 أو 1 في حالة الـ fixed-rate
    unsigned long            flags;        // الـ CLK_* flags العامة
};
```

**ملاحظة مهمة:** بس واحد من `parent_names`/`parent_data`/`parent_hws` يُستخدم في نفس الوقت.

---

#### 4. `struct clk_ops`

**الهدف:** جدول الـ function pointers — بيحدد إيه اللي الـ clock ده يعرف يعمله.

الـ `clk_fixed_rate_ops` بيملي بس:

```c
const struct clk_ops clk_fixed_rate_ops = {
    .recalc_rate     = clk_fixed_rate_recalc_rate,
    .recalc_accuracy = clk_fixed_rate_recalc_accuracy,
};
```

| Op المفعّل | الوظيفة |
|-----------|---------|
| `recalc_rate` | يرجع `fixed_rate` من غير ما يسأل الـ HW |
| `recalc_accuracy` | يرجع `fixed_accuracy` أو accuracy الـ parent |

كل الـ ops التانية (enable، disable، set_rate، set_parent...) = NULL — الـ CCF بيتعامل مع الـ NULL بشكل آمن.

---

#### 5. `struct clk_parent_data`

**الهدف:** وصف الـ parent clock بطريقة flexible — بدل الاعتماد على الـ string names بس.

```c
struct clk_parent_data {
    const struct clk_hw *hw;       // pointer مباشر (أسرع وأضمن)
    const char          *fw_name;  // اسم في الـ firmware (DT)
    const char          *name;     // اسم global fallback
    int                  index;    // index في الـ DT clocks property
};
```

---

### مخطط العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────┐
│                  struct clk_fixed_rate                  │
│                                                         │
│  ┌──────────────────┐   fixed_rate: unsigned long       │
│  │   struct clk_hw  │   fixed_accuracy: unsigned long   │
│  │                  │   flags: unsigned long            │
│  │  core ──────────────────────────────────────────┐   │
│  │  clk  ──────────────────────────────────────┐   │   │
│  │  init ──────────────────────────────────┐   │   │   │
│  └──────────────────┘                      │   │   │   │
└───────────────────────────────────────────│───│───│───┘
                                            │   │   │
                    ┌───────────────────────┘   │   │
                    ▼                           │   │
          ┌──────────────────┐                 │   │
          │ struct clk_init  │ (مؤقت)          │   │
          │ _data            │                 │   │
          │                  │                 │   │
          │  name            │                 │   │
          │  ops ────────────────────────────────────┐
          │  parent_names    │                 │   │  │
          │  parent_data     │                 │   │  │
          │  parent_hws      │                 │   │  │
          │  num_parents     │                 │   │  │
          │  flags           │                 │   │  │
          └──────────────────┘                 │   │  │
                                               │   │  │
                    ┌──────────────────────────┘   │  │
                    ▼                              │  │
          ┌──────────────────┐                    │  │
          │   struct clk     │ (per-user handle)  │  │
          │   (opaque)       │◄───────────────────┘  │
          └──────────────────┘                       │
                                                     │
          ┌──────────────────────────────────────────┘
          ▼
  ┌──────────────────┐
  │   struct clk_ops │
  │                  │
  │  .recalc_rate ───────► clk_fixed_rate_recalc_rate()
  │  .recalc_accu ───────► clk_fixed_rate_recalc_accuracy()
  │  (باقي الـ ops = NULL)│
  └──────────────────┘

          ┌──────────────────┐
          │  struct clk_core │ (داخل الـ CCF)
          │  (opaque للـ    │
          │   driver)        │
          │  يُملأ بواسطة   │
          │  clk_hw_register │
          └──────────────────┘
```

---

### مخطط دورة الحياة (Lifecycle)

#### مسار التسجيل العادي (non-devm)

```
                    Driver/Platform Code
                           │
                           ▼
           clk_hw_register_fixed_rate(dev, name, parent, flags, rate)
                           │
                           ▼
           __clk_hw_register_fixed_rate(dev, np, ..., devm=false)
                           │
                    ┌──────┴──────┐
                    │  kzalloc()  │  ← تخصيص struct clk_fixed_rate
                    └──────┬──────┘
                           │
                    ┌──────┴──────────────────────┐
                    │  تهيئة struct clk_init_data  │
                    │  name, ops, parent, flags    │
                    └──────┬──────────────────────┘
                           │
                    ┌──────┴──────────────────────┐
                    │  تعبئة struct clk_fixed_rate │
                    │  fixed_rate, fixed_accuracy  │
                    │  flags, hw.init = &init      │
                    └──────┬──────────────────────┘
                           │
                 ┌─────────┴──────────┐
                 │ dev أو np موجود؟   │
                 └─────────┬──────────┘
                    ┌──────┴──────┐
              dev   │             │   np بس (DT)
              ▼                   ▼
    clk_hw_register()    of_clk_hw_register()
              │                   │
              └──────┬────────────┘
                     │
                     ▼
          CCF يقرأ hw.init ويملأ hw.core
          hw.init يُحوّل لـ NULL
                     │
                     ▼
              يرجع struct clk_hw *
                     │
                     ▼
              الـ clock جاهز للاستخدام
```

#### مسار الـ devm (auto-cleanup)

```
           devm_clk_hw_register_fixed_rate(dev, ...)
                           │
                           ▼
           __clk_hw_register_fixed_rate(..., devm=true)
                           │
                    devres_alloc(devm_clk_hw_register_fixed_rate_release,
                                 sizeof(clk_fixed_rate), GFP_KERNEL)
                           │
                    [تهيئة وتسجيل زي فوق]
                           │
                    devres_add(dev, fixed)
                           │
                           ▼
              عند device_unregister أو driver_detach:
                           │
                    devm_clk_hw_register_fixed_rate_release()
                           │
                    clk_hw_unregister(&fix->hw)
                           │
                    devres framework → kfree(fixed)
```

#### مسار الـ Device Tree (CONFIG_OF)

```
          DT node: compatible = "fixed-clock"
                     clock-frequency = <24000000>
                     clock-accuracy  = <100>
                           │
           ┌───────────────┴───────────────────┐
           │ Early boot (CLK_OF_DECLARE)        │ Platform driver probe
           │ of_fixed_clk_setup(node)           │ of_fixed_clk_probe(pdev)
           └───────────────┬───────────────────┘
                           │
                    _of_fixed_clk_setup(node)
                           │
                    of_property_read_u32("clock-frequency")
                    of_property_read_u32("clock-accuracy")
                    of_property_read_string("clock-output-names")
                           │
                    clk_hw_register_fixed_rate_with_accuracy()
                           │
                    of_clk_add_hw_provider(node, of_clk_hw_simple_get, hw)
                           │
                    الـ consumers قادرين يعملوا clk_get() عبر الـ DT
```

#### مسار التدمير (Teardown)

```
    ┌──────────────────────────────────────────────────────┐
    │                  خيارات الـ Unregister               │
    └──────────────────────────────────────────────────────┘

    [1] clk_hw_unregister_fixed_rate(hw)
              │
              ├── clk_hw_unregister(hw)    ← ينزل الـ clock من الـ CCF tree
              └── kfree(fixed)             ← يحرر الـ struct

    [2] clk_unregister_fixed_rate(clk)    [legacy API]
              │
              ├── __clk_get_hw(clk)       ← يجيب الـ hw من الـ clk
              ├── clk_unregister(clk)     ← ينزله من الـ CCF
              └── kfree(to_clk_fixed_rate(hw))

    [3] devm auto-release
              │
              └── devm_clk_hw_register_fixed_rate_release()
                      │
                      └── clk_hw_unregister(&fix->hw)
                          [بس unregister — الـ kfree بيعمله devres]
```

---

### مخطط الـ Call Flow

#### سيناريو: consumer بيسأل عن الـ rate

```
consumer driver
    │
    └── clk_get_rate(clk)
              │
              ▼
        CCF: clk_core_get_rate(core)
              │
              ├── لو في cache وCLK_GET_RATE_NOCACHE مش مفعّل → رجّع الـ cache
              │
              └── clk_core_recalc_rate(core, parent_rate)
                        │
                        ▼
                  clk_ops->recalc_rate(hw, parent_rate)
                        │
                        ▼
                  clk_fixed_rate_recalc_rate(hw, parent_rate)
                        │
                        ▼
                  return to_clk_fixed_rate(hw)->fixed_rate
                        │
                        ▼  [مثلاً 24000000 Hz]
                  ترجع للـ consumer
```

#### سيناريو: consumer بيسأل عن الـ accuracy

```
consumer driver
    │
    └── clk_get_accuracy(clk)
              │
              ▼
        CCF → clk_ops->recalc_accuracy(hw, parent_accuracy)
              │
              ▼
        clk_fixed_rate_recalc_accuracy(hw, parent_accuracy)
              │
         ┌────┴────────────────────────────────┐
         │ CLK_FIXED_RATE_PARENT_ACCURACY مفعّل؟│
         └────┬────────────────────────────────┘
         نعم  │                          لا
              ▼                          ▼
     return parent_accuracy    return fixed->fixed_accuracy
```

#### سيناريو: تسجيل عبر `__clk_hw_register_fixed_rate`

```
driver code
    │
    └── clk_hw_register_fixed_rate(dev, "osc24m", NULL, 0, 24000000)
          │  [macro يوسّع لـ __clk_hw_register_fixed_rate]
          │
          ▼
    __clk_hw_register_fixed_rate(dev=dev, np=NULL, name="osc24m",
                                  parent_name=NULL, ..., rate=24000000,
                                  accuracy=0, clk_fixed_flags=0, devm=false)
          │
          ├── kzalloc(sizeof(struct clk_fixed_rate), GFP_KERNEL)
          │
          ├── init.name    = "osc24m"
          │   init.ops     = &clk_fixed_rate_ops
          │   init.flags   = 0
          │   init.num_parents = 0
          │
          ├── fixed->fixed_rate    = 24000000
          │   fixed->fixed_accuracy = 0
          │   fixed->flags         = 0
          │   fixed->hw.init       = &init
          │
          ├── dev != NULL → clk_hw_register(dev, &fixed->hw)
          │         │
          │         ▼
          │   CCF: يقرأ hw->init، يبني struct clk_core
          │        يضيف الـ clock للـ global tree
          │        يحوّل hw->init لـ NULL
          │
          └── return &fixed->hw
```

#### سيناريو: طلب set_rate (يفشل بشكل طبيعي)

```
consumer
    │
    └── clk_set_rate(clk, 48000000)
              │
              ▼
        CCF: يدور على clk_ops->set_rate
              │
              ▼
        clk_fixed_rate_ops.set_rate = NULL
              │
              ▼
        CCF: لا يُنفّذ set_rate — يرجع -EINVAL أو يتجاهل حسب السياق
              │
              ▼
        الـ rate فاضل 24000000 Hz
```

---

### استراتيجية الـ Locking

الـ `clk-fixed-rate.c` نفسه **مافيهوش أي lock صريح** لأن:

1. البيانات ثابتة بعد الـ init — `fixed_rate` و`fixed_accuracy` و`flags` ما بتتغيرش أبداً بعد `clk_hw_register`.
2. كل الـ locking بيحصل في الـ CCF (`drivers/clk/clk.c`).

#### الـ Locks في الـ CCF اللي بتحمي الـ fixed-rate clock

| Lock | النوع | بيحمي إيه | متى بيتمسك |
|------|-------|-----------|------------|
| `prepare_lock` | `mutex` | الـ prepare/unprepare count، الـ clock tree structure | `clk_prepare()`، `clk_unprepare()` |
| `enable_lock` | `spinlock` | الـ enable/disable count | `clk_enable()`، `clk_disable()` (atomic context) |
| `clk_list_lock` | `spinlock` | القائمة العامة للـ clocks المسجلة | وقت التسجيل/الإلغاء |

#### ترتيب الـ Locking (Lock Ordering)

```
prepare_lock (mutex)
    └── enable_lock (spinlock)
```

القاعدة: ممتنعش `enable_lock` وأنت بتنتظر `prepare_lock` — الأول أبطأ (sleepable)، الثاني atomic.

#### ليه الـ fixed-rate آمن بدون locks؟

```
struct clk_fixed_rate بعد التسجيل:
┌─────────────────────────────────────────────┐
│  fixed_rate     → قراءة فقط (read-only)     │
│  fixed_accuracy → قراءة فقط (read-only)     │
│  flags          → قراءة فقط (read-only)     │
│  hw.core        → يتحكم فيه CCF             │
│  hw.clk         → يتحكم فيه CCF             │
│  hw.init        → NULL بعد التسجيل          │
└─────────────────────────────────────────────┘

كل القراءات concurrent آمنة — ما فيش كتابة بعد الـ register.
```

#### الـ devres والـ Thread Safety

الـ `devres_alloc` / `devres_add` / `devres_free` — كلهم thread-safe من جهة الـ devres framework نفسه. الـ driver مش محتاج يعمل حاجة.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### جدول كل الـ Functions والـ APIs

| الـ Function / الـ Macro | النوع | الغرض |
|---|---|---|
| `clk_fixed_rate_recalc_rate` | `static` ops callback | ترجع الـ `fixed_rate` بدون أي حساب |
| `clk_fixed_rate_recalc_accuracy` | `static` ops callback | ترجع الـ accuracy (ثابتة أو من الـ parent) |
| `clk_fixed_rate_ops` | `const struct clk_ops` | جدول الـ ops للـ fixed-rate clock |
| `devm_clk_hw_register_fixed_rate_release` | `static` devres destructor | cleanup callback لحذف الـ clock عند الـ device release |
| `__clk_hw_register_fixed_rate` | `exported` core allocator | الـ function الأساسية اللي بتعمل alloc وتسجيل للـ clock |
| `clk_register_fixed_rate` | `exported` legacy API | wrapper قديم بيرجع `struct clk *` بدل `clk_hw *` |
| `clk_unregister_fixed_rate` | `exported` legacy cleanup | إلغاء تسجيل clock قديم من النوع `struct clk *` |
| `clk_hw_unregister_fixed_rate` | `exported` modern cleanup | إلغاء تسجيل clock حديث من النوع `struct clk_hw *` |
| `_of_fixed_clk_setup` | `static` OF helper | قراءة الـ DT node وتسجيل الـ clock وإضافته كـ provider |
| `of_fixed_clk_setup` | `__init` OF callback | entry point لوقت الـ boot عبر `CLK_OF_DECLARE` |
| `of_fixed_clk_probe` | `platform_driver` probe | entry point لوقت runtime عبر الـ platform driver |
| `of_fixed_clk_remove` | `platform_driver` remove | cleanup عند إزالة الـ platform device |
| `clk_hw_register_fixed_rate` | macro | wrapper مبسّط فوق `__clk_hw_register_fixed_rate` |
| `devm_clk_hw_register_fixed_rate` | macro | نفس السابق لكن بـ devres lifecycle |

---

### المجموعة 1: الـ ops Callbacks

هذه الـ callbacks هي قلب تعامل الـ CCF (Common Clock Framework) مع الـ clock. الـ `clk_fixed_rate_ops` يعرّف فقط `recalc_rate` و`recalc_accuracy` لأن الـ fixed-rate clock لا تحتاج `enable`/`disable`/`set_rate` — فكل هذه الـ ops تُورَث من الـ parent أو تُتجاهل.

---

#### `clk_fixed_rate_recalc_rate`

```c
static unsigned long clk_fixed_rate_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    return to_clk_fixed_rate(hw)->fixed_rate;
}
```

**الـ CCF بيسألها:** ما هو rate الـ clock دي دلوقتي؟ الإجابة دايمًا ثابتة — بترجع `fixed_rate` مباشرةً من الـ struct بدون أي قراءة من الـ hardware أو حسابات. الـ `parent_rate` بيتجاهل تمامًا لأن الـ clock مش بتشتق rate-ها من الـ parent.

**Parameters:**
- `hw` — pointer للـ `clk_hw` المدمج جوا `clk_fixed_rate`. بيُستخدم مع `to_clk_fixed_rate()` للوصول للـ container struct.
- `parent_rate` — الـ rate الخاص بالـ parent. مُتجاهَل هنا تمامًا.

**Return:** `unsigned long` — قيمة `fixed_rate` الثابتة بالـ Hz.

**Key details:**
- لا locking داخلي — الـ CCF بيمسك الـ `prepare_mutex` قبل ما يستدعيها.
- لا side effects خالص — read-only access على الـ struct field.
- بيتكلمها الـ CCF core من `clk_recalc_rates()` أو `clk_core_get_rate_nolock()`.

---

#### `clk_fixed_rate_recalc_accuracy`

```c
static unsigned long clk_fixed_rate_recalc_accuracy(struct clk_hw *hw,
        unsigned long parent_accuracy)
{
    struct clk_fixed_rate *fixed = to_clk_fixed_rate(hw);

    if (fixed->flags & CLK_FIXED_RATE_PARENT_ACCURACY)
        return parent_accuracy;

    return fixed->fixed_accuracy;
}
```

بتحدد الـ accuracy بتاع الـ clock بالـ ppb (parts per billion). لو الـ flag `CLK_FIXED_RATE_PARENT_ACCURACY` مضبوط، بترجع accuracy الـ parent (مفيد لو الـ clock source خارجي والـ accuracy بتاعه أحسن). لو مفيش flag، بترجع `fixed_accuracy` الثابتة.

**Parameters:**
- `hw` — pointer للـ `clk_hw`.
- `parent_accuracy` — الـ accuracy بتاع الـ parent clock بالـ ppb.

**Return:** `unsigned long` — الـ accuracy بالـ ppb.

**Key details:**
- الـ flag `CLK_FIXED_RATE_PARENT_ACCURACY` هو flag خاص بالـ `clk_fixed_rate` (مش CCF flags)، متعرّفش في `linux/clk-provider.h` كـ `BIT(0)` من `clk_fixed_rate.flags`.
- الـ CCF بيستدعيها من `clk_core_get_accuracy_recalc()`.

---

#### `clk_fixed_rate_ops`

```c
const struct clk_ops clk_fixed_rate_ops = {
    .recalc_rate      = clk_fixed_rate_recalc_rate,
    .recalc_accuracy  = clk_fixed_rate_recalc_accuracy,
};
EXPORT_SYMBOL_GPL(clk_fixed_rate_ops);
```

جدول الـ operations الخاص بالـ fixed-rate clock. بيحتوي فقط على ops الـ rate والـ accuracy لأن:
- **لا enable/disable**: الـ fixed clock مش بتتوقف — الـ CCF بيـpass الـ enable للـ parent.
- **لا set_rate / round_rate**: الـ rate ثابت ومش قابل للتغيير.
- **لا set_parent**: الـ parent ثابت.

مُصدَّر بـ `EXPORT_SYMBOL_GPL` عشان الـ drivers التانية ممكن تعيد استخدامه لو عايزة تعمل clock بنفس السلوك.

---

### المجموعة 2: التسجيل (Registration)

هذه المجموعة مسؤولة عن allocate وتهيئة وتسجيل الـ `clk_fixed_rate` struct في الـ CCF.

---

#### `__clk_hw_register_fixed_rate` — الـ Core Allocator

```c
struct clk_hw *__clk_hw_register_fixed_rate(struct device *dev,
        struct device_node *np, const char *name,
        const char *parent_name, const struct clk_hw *parent_hw,
        const struct clk_parent_data *parent_data, unsigned long flags,
        unsigned long fixed_rate, unsigned long fixed_accuracy,
        unsigned long clk_fixed_flags, bool devm)
```

هي الـ function الحقيقية اللي بتعمل كل شغل التسجيل. كل الـ macros والـ wrappers بتوصل في النهاية ليها. بتقوم بـ alloc الـ `clk_fixed_rate` struct، وملء الـ `clk_init_data`، ثم استدعاء `clk_hw_register()` أو `of_clk_hw_register()` حسب السياق.

**Parameters:**
- `dev` — الـ `struct device` المالك للـ clock. لو `NULL` وفيه `np`، بيستخدم `of_clk_hw_register`.
- `np` — الـ `device_node` من الـ DT. بيُستخدم لو مفيش `dev`.
- `name` — اسم الـ clock في الـ CCF tree.
- `parent_name` — اسم الـ parent كـ string (legacy method).
- `parent_hw` — pointer مباشر للـ parent `clk_hw` (modern method).
- `parent_data` — struct بيحتوي معلومات الـ parent بطريقة أكثر مرونة.
- `flags` — CCF-level flags زي `CLK_SET_RATE_PARENT`.
- `fixed_rate` — الـ rate الثابت بالـ Hz.
- `fixed_accuracy` — الـ accuracy الثابتة بالـ ppb.
- `clk_fixed_flags` — flags خاصة بالـ `clk_fixed_rate` زي `CLK_FIXED_RATE_PARENT_ACCURACY`.
- `devm` — `true` لو المطلوب ربط الـ lifetime بالـ device.

**Return:** `struct clk_hw *` — pointer للـ `clk_hw` المسجل، أو `ERR_PTR()` في حالة الفشل.

**Pseudocode flow:**

```
if devm:
    fixed = devres_alloc(release_fn, sizeof(*fixed))
else:
    fixed = kzalloc(sizeof(*fixed))

if !fixed → return ERR_PTR(-ENOMEM)

fill init: name, ops=clk_fixed_rate_ops, flags, parents
fill fixed: fixed_rate, fixed_accuracy, clk_fixed_flags

hw = &fixed->hw
hw->init = &init

if dev OR !np:
    ret = clk_hw_register(dev, hw)
else:
    ret = of_clk_hw_register(np, hw)

if ret:
    if devm: devres_free(fixed)
    else: kfree(fixed)
    return ERR_PTR(ret)

if devm:
    devres_add(dev, fixed)   ← ربط الـ lifetime

return hw
```

**Key details:**
- الـ `init` struct بيتعمل على الـ stack ومش بيتحفظ بعد التسجيل — الـ CCF بيـcopy كل اللي يحتاجه.
- الثلاث طرق لتحديد الـ parent (`parent_name`, `parent_hw`, `parent_data`) بتُستخدم بشكل حصري — فقط واحد في المرة.
- لو `devm=true`، الـ `devres_add` بيضمن إن `devm_clk_hw_register_fixed_rate_release` هيتكلم تلقائيًا لما الـ device يتحذف.

---

#### `clk_register_fixed_rate` — Legacy Wrapper

```c
struct clk *clk_register_fixed_rate(struct device *dev, const char *name,
        const char *parent_name, unsigned long flags,
        unsigned long fixed_rate)
```

API قديم بيرجع `struct clk *` بدل `struct clk_hw *`. الـ new drivers المفروض تستخدم `clk_hw_register_fixed_rate` بدلًا منه. بيستدعي `clk_hw_register_fixed_rate_with_accuracy` مع `accuracy=0`.

**Parameters:**
- `dev` — الـ device المالك.
- `name` — اسم الـ clock.
- `parent_name` — اسم الـ parent كـ string.
- `flags` — CCF flags.
- `fixed_rate` — الـ rate بالـ Hz.

**Return:** `struct clk *` أو `ERR_PTR()`.

**Key details:**
- بيستخدم `ERR_CAST(hw)` لتحويل الـ error من `clk_hw *` لـ `clk *` بدون فقدان الـ error code.
- الوصول لـ `hw->clk` بيتطلب إن الـ CCF يكون سجّل الـ clock بنجاح أولًا.
- مُصدَّر بـ `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_hw_register_fixed_rate_release` — Devres Destructor

```c
static void devm_clk_hw_register_fixed_rate_release(struct device *dev, void *res)
{
    struct clk_fixed_rate *fix = res;

    clk_hw_unregister(&fix->hw);
}
```

هذا هو الـ cleanup callback اللي بيتسجّل مع الـ devres framework. لما الـ device يتحذف، الـ devres بيستدعي الـ function دي تلقائيًا.

**لماذا لا يستدعي `clk_hw_unregister_fixed_rate`؟** لأن `clk_hw_unregister_fixed_rate` بيعمل `kfree(fixed)` بعد الـ unregister، بس في الحالة دي الـ devres framework هو اللي هيعمل الـ `kfree` تلقائيًا على الـ `res` pointer — فلو اتعملت `kfree` يدويًا برضو، هيحصل double free.

**Key details:**
- `res` هنا هو نفس الـ `fixed` pointer اللي اتعمله `devres_alloc`.
- بيستدعي فقط `clk_hw_unregister` ويسيب الـ `kfree` للـ devres infrastructure.

---

### المجموعة 3: إلغاء التسجيل (Cleanup / Unregistration)

---

#### `clk_unregister_fixed_rate` — Legacy Unregister

```c
void clk_unregister_fixed_rate(struct clk *clk)
{
    struct clk_hw *hw;

    hw = __clk_get_hw(clk);
    if (!hw)
        return;

    clk_unregister(clk);
    kfree(to_clk_fixed_rate(hw));
}
EXPORT_SYMBOL_GPL(clk_unregister_fixed_rate);
```

المقابل لـ `clk_register_fixed_rate`. بيستخرج الـ `clk_hw` من الـ `struct clk`، بيلغي تسجيله من الـ CCF، ثم بيحرر الـ `clk_fixed_rate` struct.

**Parameters:**
- `clk` — pointer للـ `struct clk` المراد حذفه.

**Key details:**
- `__clk_get_hw` دالة internal تترجم من `struct clk *` لـ `struct clk_hw *`.
- الترتيب مهم: `clk_unregister` أولًا ثم `kfree` — العكس سيؤدي لـ use-after-free.
- بيتكلمه الـ driver اللي سجّل الـ clock بـ `clk_register_fixed_rate`.

---

#### `clk_hw_unregister_fixed_rate` — Modern Unregister

```c
void clk_hw_unregister_fixed_rate(struct clk_hw *hw)
{
    struct clk_fixed_rate *fixed;

    fixed = to_clk_fixed_rate(hw);

    clk_hw_unregister(hw);
    kfree(fixed);
}
EXPORT_SYMBOL_GPL(clk_hw_unregister_fixed_rate);
```

المقابل الحديث للـ `clk_hw_register_fixed_rate`. بيلغي تسجيل الـ clock من الـ CCF ويحرر الذاكرة.

**Parameters:**
- `hw` — pointer للـ `clk_hw` المراد حذفه.

**Key details:**
- **لا يُستخدم مع devm-registered clocks** — الـ devres release callback بيستدعي `clk_hw_unregister` فقط بدون `kfree`، وبيسيب الـ `kfree` للـ devres، تحاشيًا لـ double free.
- الـ `to_clk_fixed_rate(hw)` لازم يتسيف قبل `clk_hw_unregister` لأن الـ CCF ممكن يكون بيمشي على الـ hw في أي وقت.

---

### المجموعة 4: الـ Device Tree / OF Integration

هذه المجموعة بتربط الـ fixed-rate clock بالـ Device Tree، وبتدعم سيناريوهين مختلفين: وقت الـ boot المبكر (early boot)، ووقت الـ runtime عبر الـ platform driver.

---

#### `_of_fixed_clk_setup` — الـ Core DT Setup Helper

```c
static struct clk_hw *_of_fixed_clk_setup(struct device_node *node)
{
    struct clk_hw *hw;
    const char *clk_name = node->name;
    u32 rate;
    u32 accuracy = 0;
    int ret;

    if (of_property_read_u32(node, "clock-frequency", &rate))
        return ERR_PTR(-EIO);

    of_property_read_u32(node, "clock-accuracy", &accuracy);

    of_property_read_string(node, "clock-output-names", &clk_name);

    hw = clk_hw_register_fixed_rate_with_accuracy(NULL, clk_name, NULL,
                                                  0, rate, accuracy);
    if (IS_ERR(hw))
        return hw;

    ret = of_clk_add_hw_provider(node, of_clk_hw_simple_get, hw);
    if (ret) {
        clk_hw_unregister_fixed_rate(hw);
        return ERR_PTR(ret);
    }

    return hw;
}
```

بتقرأ معلومات الـ clock من الـ DT node وتسجّله في الـ CCF، ثم بتضيفه كـ clock provider عشان الـ nodes التانية في الـ DT تقدر ترجع عليه بـ `clocks = <&my_clock>`. لو `clock-output-names` مش موجودة، بيستخدم اسم الـ node نفسه.

**Parameters:**
- `node` — الـ `device_node` اللي بيمثل الـ fixed-clock في الـ DT.

**Return:** `struct clk_hw *` أو `ERR_PTR()`.

**مثال DT:**
```dts
osc: oscillator {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;   /* 24 MHz */
    clock-accuracy = <1000>;        /* 1000 ppb */
    clock-output-names = "osc24m";
};
```

**Key details:**
- `clock-frequency` إلزامية — لو مش موجودة بترجع `-EIO`.
- `clock-accuracy` اختيارية — الـ default هو `0` (perfect clock).
- `of_clk_add_hw_provider` بيربط الـ `node` بالـ `hw` عشان الـ lookups من الـ DT تشتغل.
- لو `of_clk_add_hw_provider` فشل، بتعمل cleanup للـ hw المسجّل قبل الرجوع بـ error.

---

#### `of_fixed_clk_setup` — Early Boot Entry Point

```c
void __init of_fixed_clk_setup(struct device_node *node)
{
    _of_fixed_clk_setup(node);
}
CLK_OF_DECLARE(fixed_clk, "fixed-clock", of_fixed_clk_setup);
```

**الـ `__init` marker** معناه إن الـ function دي بتشتغل فقط أثناء الـ kernel init وبيتحرر الـ memory بتاعها بعدين. **الـ `CLK_OF_DECLARE` macro** بيسجّل الـ function في section خاص في الـ binary (`__clk_of_table`) عشان الـ OF subsystem يعرف إنه لو لقى node بالـ compatible `"fixed-clock"` في وقت مبكر من الـ boot، يستدعي `of_fixed_clk_setup`.

**Key details:**
- الـ `CLK_OF_DECLARE` بيخلي الـ DT node يتعامل معاه قبل حتى ما الـ platform driver framework يقوم — ده ضروري للـ clocks اللي بيحتاجها الـ kernel نفسه في وقت الـ boot.
- الـ function بترمي الـ return value لأن الـ early init مش محتاج يتعامل مع الـ error بشكل صريح.

---

#### `of_fixed_clk_probe` — Runtime Platform Driver Probe

```c
static int of_fixed_clk_probe(struct platform_device *pdev)
{
    struct clk_hw *hw;

    /*
     * This function is not executed when of_fixed_clk_setup
     * succeeded.
     */
    hw = _of_fixed_clk_setup(pdev->dev.of_node);
    if (IS_ERR(hw))
        return PTR_ERR(hw);

    platform_set_drvdata(pdev, hw);

    return 0;
}
```

ده الـ probe بتاع الـ platform driver اللي بيتكلم لو `of_fixed_clk_setup` **لم تنجح** في وقت الـ boot المبكر (يعني لو الـ DT node اتعالج بعد الـ init phase). بيخزّن الـ `hw` في `pdev->driver_data` عشان `of_fixed_clk_remove` يقدر يوصله.

**Parameters:**
- `pdev` — الـ `platform_device` المقابل للـ DT node.

**Return:** `0` عند النجاح، أو error code سالب.

**Key details:**
- التعليق الموجود في الـ code بيوضح إن الـ probe ده مش بيشتغل لو `CLK_OF_DECLARE` نجح أصلًا.
- `platform_set_drvdata(pdev, hw)` بيخزّن الـ `hw` عشان `remove` يقدر يجيبه بـ `platform_get_drvdata`.

---

#### `of_fixed_clk_remove` — Platform Driver Remove

```c
static void of_fixed_clk_remove(struct platform_device *pdev)
{
    struct clk_hw *hw = platform_get_drvdata(pdev);

    of_clk_del_provider(pdev->dev.of_node);
    clk_hw_unregister_fixed_rate(hw);
}
```

بتعمل cleanup كامل للـ clock عند إزالة الـ device. بتحذف الـ provider أولًا من الـ OF registry ثم بتلغي تسجيل الـ clock وتحرر الذاكرة.

**Parameters:**
- `pdev` — الـ platform device المراد إزالته.

**Key details:**
- الترتيب مهم: `of_clk_del_provider` لازم يجي قبل `clk_hw_unregister_fixed_rate` عشان نضمن إن محدش قادر يطلب reference جديدة للـ clock بعدما اتحذف.
- الـ `of_fixed_clk_driver` مسجّل بـ `builtin_platform_driver` يعني هو جزء من الـ kernel مش module منفصل.

---

### المجموعة 5: الـ Macros والـ Static Data

#### جدول الـ Macros

| الـ Macro | يُوسّع إلى | الغرض |
|---|---|---|
| `to_clk_fixed_rate(_hw)` | `container_of(_hw, struct clk_fixed_rate, hw)` | الوصول من `clk_hw *` للـ `clk_fixed_rate *` |
| `clk_hw_register_fixed_rate(dev, name, pname, flags, rate)` | `__clk_hw_register_fixed_rate(..., false)` | تسجيل بدون devres، بدون accuracy |
| `devm_clk_hw_register_fixed_rate(dev, name, pname, flags, rate)` | `__clk_hw_register_fixed_rate(..., true)` | تسجيل مع devres، بدون accuracy |

#### `of_fixed_clk_ids` و `of_fixed_clk_driver`

```c
static const struct of_device_id of_fixed_clk_ids[] = {
    { .compatible = "fixed-clock" },
    { }
};

static struct platform_driver of_fixed_clk_driver = {
    .driver = {
        .name = "of_fixed_clk",
        .of_match_table = of_fixed_clk_ids,
    },
    .probe  = of_fixed_clk_probe,
    .remove = of_fixed_clk_remove,
};
builtin_platform_driver(of_fixed_clk_driver);
```

الـ driver ده مسجّل عشان يتعامل مع الـ `"fixed-clock"` nodes اللي فاتتها الـ early init. الـ `builtin_platform_driver` بيخليه يتسجّل أوتوماتيكيًا في `device_initcall()` level.

---

### تدفق التسجيل الكامل — Flow Diagram

```
DT Boot:
  OF subsystem scans DT
       │
       ▼
  finds compatible="fixed-clock"
       │
       ├─── Early boot (CLK_OF_DECLARE matched) ──►  of_fixed_clk_setup()
       │                                                      │
       │                                               _of_fixed_clk_setup()
       │                                                      │
       └─── Late boot (platform driver matched) ──► of_fixed_clk_probe()
                                                          │
                                                   _of_fixed_clk_setup()
                                                          │
                                                          ▼
                                          clk_hw_register_fixed_rate_with_accuracy()
                                                          │
                                                          ▼
                                          __clk_hw_register_fixed_rate()
                                                          │
                                            ┌─────────────┴──────────────┐
                                            │                            │
                                       devm=true                    devm=false
                                            │                            │
                                      devres_alloc()               kzalloc()
                                            │                            │
                                            └─────────────┬──────────────┘
                                                          │
                                                    fill init_data
                                                    fill clk_fixed_rate
                                                          │
                                            ┌─────────────┴──────────────┐
                                            │                            │
                                       dev || !np                  !dev && np
                                            │                            │
                                   clk_hw_register()         of_clk_hw_register()
                                            │                            │
                                            └─────────────┬──────────────┘
                                                          │
                                                    if devm:
                                                   devres_add()
                                                          │
                                                   return clk_hw *
```

---

### ملاحظات إضافية للـ Expert

1. **الـ `clk_fixed_rate.flags` vs CCF flags**: فيه نوعين من الـ flags هنا — الـ CCF-level flags في `clk_init_data.flags` (زي `CLK_SET_RATE_PARENT`)، والـ fixed-rate specific flags في `clk_fixed_rate.flags` (زي `CLK_FIXED_RATE_PARENT_ACCURACY`). الخلط بينهم bug شائع.

2. **الـ `of_clk_hw_register` vs `clk_hw_register`**: الأولى بتربط الـ clock بالـ `device_node` بدل الـ `device`، ومهمة للـ clocks اللي بتتسجل قبل ما الـ `struct device` يتخلق للـ DT node.

3. **الـ double-registration protection**: الـ comment في الـ source بيوضح صريحًا إن `of_fixed_clk_probe` مش هيشتغل لو `of_fixed_clk_setup` نجح — ده لأن الـ kernel بعد ما الـ `CLK_OF_DECLARE` handler ينجح، بيـmark الـ node على إنه اتعالج.

4. **الـ `__init` vs `builtin_platform_driver`**: الـ `of_fixed_clk_setup` بيشتغل في `arch_initcall` level أو أبكر. الـ platform driver بيشتغل في `device_initcall`. ده فرق مهم في ترتيب الـ initialization.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بـ clk-fixed-rate

الـ **CCF (Common Clock Framework)** بيعمل تلقائياً directory لكل clock مسجّل تحت:

```
/sys/kernel/debug/clk/<clock-name>/
```

الملفات الجوهرية داخل الـ directory دي:

| الملف | المحتوى | كيفية القراءة |
|---|---|---|
| `clk_rate` | الـ frequency الحالية بالـ Hz | `cat /sys/kernel/debug/clk/<name>/clk_rate` |
| `clk_accuracy` | الدقة بالـ ppb | `cat /sys/kernel/debug/clk/<name>/clk_accuracy` |
| `clk_flags` | الـ flags المضبوطة على الـ clock | `cat /sys/kernel/debug/clk/<name>/clk_flags` |
| `clk_enable_count` | عدد مرات الـ enable | `cat /sys/kernel/debug/clk/<name>/clk_enable_count` |
| `clk_prepare_count` | عدد مرات الـ prepare | `cat /sys/kernel/debug/clk/<name>/clk_prepare_count` |
| `clk_notifier_count` | عدد الـ notifiers المسجّلين | `cat /sys/kernel/debug/clk/<name>/clk_notifier_count` |

لعرض شجرة كل الـ clocks دفعة واحدة:

```bash
# عرض شجرة الـ clocks مع الـ rates
cat /sys/kernel/debug/clk/clk_summary

# مثال على output:
#                                  enable  prepare  protect                                duty
# clock                          count    count    count        rate   accuracy phase  cycle
# --------------------------------------------------------------------------------------------
# xtal                               1        1        0    24000000          0     0  50000
#    fixed_clk_100mhz               0        0        0   100000000          0     0  50000
```

```bash
# عرض الشجرة بتنسيق هرمي
cat /sys/kernel/debug/clk/clk_orphan_summary

# dump كامل بشكل مفصّل
cat /sys/kernel/debug/clk/clk_dump
```

الـ `clk_dump` بيطلع JSON يشمل كل clock مع كل خصائصه — مفيد جداً للـ scripting.

---

#### 2. مدخلات الـ sysfs المتعلقة بـ fixed-rate clocks

الـ fixed-rate clock نفسه مش بيظهر كـ device مستقل في الغالب، لكن لو اتسجّل عبر `platform_driver` (زي `of_fixed_clk_driver`)، هتلاقيه هنا:

```bash
# إيجاد الـ platform device الخاص بالـ fixed clock
ls /sys/bus/platform/devices/ | grep fixed_clk

# مثال
ls /sys/bus/platform/devices/of_fixed_clk/
```

للـ consumers اللي بيستخدموا الـ clock:

```bash
# مين بيستخدم الـ clock ده؟
grep -r "clock-names" /sys/firmware/devicetree/base/
# أو
find /sys/devices -name "of_node" -exec ls {} \;
```

---

#### 3. الـ ftrace — الـ tracepoints والـ events

الـ CCF بيحتوي على tracepoints جاهزة تحت `clk` subsystem:

```bash
# عرض كل events متاحة للـ clk subsystem
ls /sys/kernel/tracing/events/clk/

# Output المتوقع:
# clk_disable          clk_disable_complete
# clk_enable           clk_enable_complete
# clk_prepare          clk_prepare_complete
# clk_set_rate         clk_set_rate_complete
# clk_set_parent       clk_set_parent_complete
# clk_unprepare        clk_unprepare_complete
```

بما إن الـ fixed-rate clock مش بيدعم `set_rate`، الـ events الأهم هنا هي الـ `recalc_rate` اللي بتتنفذ عند الـ registration وعند أي طلب للـ rate. فعّل الـ events دي:

```bash
# تفعيل كل events الـ clk subsystem
echo 1 > /sys/kernel/tracing/events/clk/enable

# أو بشكل انتقائي — تابع طلبات enable/disable فقط
echo 1 > /sys/kernel/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_disable/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_prepare/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# نفّذ أي عملية على النظام تستدعي الـ clock

# اقرأ النتائج
cat /sys/kernel/tracing/trace

# مثال output:
# systemd-1     [000] ....   0.123456: clk_enable: xtal
# kernel        [000] ....   0.123458: clk_prepare: fixed_clk_100mhz
```

لتابع `clk_hw_register_fixed_rate` نفسها عبر **function tracer**:

```bash
echo function > /sys/kernel/tracing/current_tracer
echo '__clk_hw_register_fixed_rate' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
# أعد تحميل الـ driver أو boot الجهاز
cat /sys/kernel/tracing/trace
```

---

#### 4. الـ printk والـ dynamic debug

الـ CCF مش بيستخدم `pr_debug` كتير في ملف `clk-fixed-rate.c` نفسه، لكن الـ core framework في `drivers/clk/clk.c` عنده **dynamic debug** messages وفيرة.

تفعيل dynamic debug لكل الـ clk subsystem:

```bash
# تفعيل كل debug messages في drivers/clk/
echo 'file drivers/clk/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو تحديداً ملف clk-fixed-rate.c
echo 'file drivers/clk/clk-fixed-rate.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو تحديداً الـ core framework
echo 'file drivers/clk/clk.c +p' > /sys/kernel/debug/dynamic_debug/control

# التحقق من الإعدادات
grep 'clk' /sys/kernel/debug/dynamic_debug/control
```

لو حبيت **printk level** أكتر تفصيلاً عبر kernel cmdline:

```
# في /etc/default/grub أضف:
GRUB_CMDLINE_LINUX="... loglevel=8 dyndbg=\"file drivers/clk/* +p\""
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config Option | الوصف | متى تفعّلها |
|---|---|---|
| `CONFIG_COMMON_CLK` | الـ CCF نفسه — لازم يكون enabled | دايماً |
| `CONFIG_CLK_DEBUG` | يفعّل الـ debugfs entries للـ clocks | أثناء الـ development |
| `CONFIG_DEBUG_FS` | لازم مفعّل عشان debugfs يشتغل | أثناء الـ development |
| `CONFIG_KALLSYMS` | يخلي stack traces أوضح بكتير | أثناء الـ development |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ `pr_debug` dynamic control | أثناء الـ development |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في الـ prepare_lock | لو في شكّ في locking issues |
| `CONFIG_DEBUG_OBJECTS_WORK` | يتتبع الـ work objects | لو في use-after-free |
| `CONFIG_KASAN` | يكتشف memory corruption | لو في crashes غريبة |
| `CONFIG_OF_UNITTEST` | اختبارات وحدة لـ Device Tree | للتحقق من DT parsing |

لتفعيل `CONFIG_CLK_DEBUG` في `.config`:

```bash
# في kernel build directory
./scripts/config --enable CONFIG_CLK_DEBUG
./scripts/config --enable CONFIG_DEBUG_FS
make oldconfig
```

---

#### 6. الـ devlink وأدوات الـ CCF المتخصصة

الـ clk subsystem مش بيستخدم `devlink` مباشرة، لكن في أدوات userspace مخصصة:

**أداة `clk_summary` المدمجة:**
```bash
# أفضل overview للحالة الكاملة
cat /sys/kernel/debug/clk/clk_summary
```

**أداة `clk_dump` للـ JSON output:**
```bash
cat /sys/kernel/debug/clk/clk_dump | python3 -m json.tool | grep -A5 "fixed"
```

**أداة `clk-utils` (إن وجدت في distro):**
```bash
# بعض الـ distros بيوفروا clk-utils
clk list           # عرض كل الـ clocks
clk rate <name>    # قراءة rate لـ clock معين
```

**عبر `devmem2` لقراءة registers يدوياً (راجع Hardware Level):**
```bash
# مثال: قراءة register من base address معروف
devmem2 0xFE000000 w
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الإصلاح |
|---|---|---|
| `clk: couldn't get clock 'fixed_clk_100mhz'` | الـ consumer طلب clock بالاسم ومش موجود | تأكد من اسم الـ clock في DT يطابق `clock-output-names` |
| `clk: failed to register fixed rate clock` | فشل `clk_hw_register` — عادةً اسم مكرر أو NULL name | تأكد من `name` مش NULL ومش متسجّل قبل كده |
| `fixed-rate clock without clock-frequency property` | DT node ليه `compatible = "fixed-clock"` لكن بدون `clock-frequency` | أضف `clock-frequency = <...>;` في الـ DT node |
| `of_clk_add_hw_provider failed` | فشل تسجيل الـ provider في the OF tree | تحقق من الـ node مش orphan وعنده parent صح |
| `-ENOMEM` عند `kzalloc` | ذاكرة ناقصة | نادر جداً في fixed-rate — راجع memory pressure |
| `WARNING: CPU: ... clk_hw_unregister` مع refcount != 0 | unregister وفي consumers لسه بيستخدموه | نظّم الـ shutdown order، استخدم `devm_` variants |
| `clk: clk_prepare_enable failed for fixed_clk` | الـ parent مش متاح | تأكد الـ parent clock متسجّل قبل الـ child |
| `-EIO` من `_of_fixed_clk_setup` | `of_property_read_u32` على `clock-frequency` فشل | الـ property غلط أو missing في DTS |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

الأماكن المنطقية لإضافة assertions أثناء الـ debugging:

```c
/* في __clk_hw_register_fixed_rate — تحقق من المدخلات */
struct clk_hw *__clk_hw_register_fixed_rate(...)
{
    /* تحقق أن الـ rate مش صفر */
    WARN_ON(fixed_rate == 0);

    /* تحقق أن مش اتديلنا dev وnp في نفس الوقت */
    WARN_ON(dev && np);

    fixed = kzalloc(...);
    /* تحقق من الـ allocation */
    if (!fixed) {
        dump_stack(); /* فين الـ memory راحت؟ */
        return ERR_PTR(-ENOMEM);
    }
    ...
}
```

```c
/* في clk_fixed_rate_recalc_rate — تحقق أن الـ rate محفوظ صح */
static unsigned long clk_fixed_rate_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    struct clk_fixed_rate *fixed = to_clk_fixed_rate(hw);

    /* لو الـ rate بقى صفر بعد الـ registration — مشكلة */
    WARN_ON_ONCE(fixed->fixed_rate == 0);

    return fixed->fixed_rate;
}
```

```c
/* في devm_clk_hw_register_fixed_rate_release — تحقق من الـ cleanup */
static void devm_clk_hw_register_fixed_rate_release(struct device *dev, void *res)
{
    struct clk_fixed_rate *fix = res;

    /* تحقق أن الـ hw pointer صالح */
    WARN_ON(!fix);

    clk_hw_unregister(&fix->hw);
}
```

---

### Hardware Level

---

#### 1. التحقق أن حالة الـ Hardware تطابق حالة الـ Kernel

الـ fixed-rate clock مصدره إما **crystal oscillator** خارجي أو **PLL** أو **oscillator** مدمج داخل الـ SoC. الخطوات:

```
Kernel State                    Hardware State
-----------                     --------------
clk_rate = 24000000 Hz   ←→    Crystal يشتغل على 24 MHz؟
clk_enable_count = 1     ←→    Enable line/power rail مفعّلة؟
clk_prepare_count = 1    ←→    الـ parent clock شغّال؟
```

**خطوات التحقق العملي:**

1. اقرأ الـ rate من debugfs وقارنه بالـ datasheet
2. تحقق من voltage rails للـ oscillator (عادةً VDD_OSC أو VDDIO)
3. لو الـ clock source هو crystal — قِس الـ frequency بـ oscilloscope على الـ crystal pins

---

#### 2. تقنيات الـ Register Dump

الـ fixed-rate clock نفسه مش بيقرأ من registers (لأن الـ rate ثابت في الكود)، لكن لو في شك في الـ hardware:

**باستخدام `devmem2`:**
```bash
# تثبيت الأداة
apt-get install devmem2
# أو
yum install devmem2

# قراءة 32-bit word من عنوان معين (مثال: clock controller base)
devmem2 0xFF760000 w

# قراءة byte
devmem2 0xFF760000 b

# مثال على Rockchip RK3399 — CRU base = 0xFF760000
# قراءة PPLL_CON0
devmem2 0xFF760000 w
# Output: /dev/mem opened. Memory mapped at address 0x7f8b430000.
# Value at address 0xFF760000 (0x7f8b430000): 0x00232400
```

**باستخدام `/dev/mem` مباشرة مع Python:**
```python
#!/usr/bin/env python3
import mmap, struct

# قراءة من /dev/mem مع alignment صحيح
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096,
                  mmap.MAP_SHARED, mmap.PROT_READ,
                  offset=0xFF760000)
    value = struct.unpack('<I', m.read(4))[0]
    print(f'Register value: 0x{value:08X}')
    m.close()
```

**لو `CONFIG_STRICT_DEVMEM=y` مفعّل:**
```bash
# استخدم /proc/iomem عشان تعرف الـ addresses المسموح بيها
cat /proc/iomem | grep -i clk

# أو استخدم kernel module بدل /dev/mem
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**نقاط القياس الجوهرية:**

```
Crystal Oscillator
      │
      ├── XTAL_IN  ←── قِس هنا: لازم تلاقي sine wave بالـ frequency الصح
      │
      └── XTAL_OUT ←── قِس هنا: square wave بعد الـ inverter الداخلي
```

**إعدادات الـ Oscilloscope:**

| الإعداد | القيمة المناسبة |
|---|---|
| Time/div | 1/frequency × 10 (مثلاً لـ 24MHz: ~40ns/div) |
| Voltage/div | حسب VDD (عادةً 1.8V أو 3.3V) |
| Trigger | Rising edge على الـ XTAL_OUT |
| Coupling | DC |

**باستخدام Logic Analyzer:**
```
Channel 0 → XTAL_OUT pin
Sample Rate → minimum 10× الـ frequency المتوقعة
             (لـ 24MHz: sample على 240MHz على الأقل)
```

**أشياء لازم تشوفها:**
- الـ frequency بالضبط تطابق الـ `clock-frequency` في DTS
- مفيش noise أو ringing زايد على الـ edges
- الـ duty cycle قريب من 50%
- الـ rise/fall time مش أبطأ من المتوقع (يدل على capacitive loading)

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | الأعراض في الـ Kernel Log | التشخيص |
|---|---|---|
| Crystal مش شغّال | `clk: couldn't get clock 'xtal'` + system يهنج مبكراً | قِس على الـ XTAL pins بـ oscilloscope |
| voltage rail منقطعة للـ oscillator | Kernel يـ boot جزئياً وبعدين يتوقف | قِس VDD_OSC بـ multimeter |
| اسم clock في DTS غلط | `clk: failed to register clock 'wrong_name'` | قارن `clock-output-names` بين DTS والـ consumer |
| `clock-frequency` مش محدد | `-EIO returned from _of_fixed_clk_setup` | أضف `clock-frequency` للـ DTS node |
| مشكلة في الـ crystal loading capacitors | Clock unstable، أحياناً بيشتغل وأحياناً لأ | قِس بـ oscilloscope وراجع الـ BOM |
| clock مش مربوط صح في الـ DTS | `device X: failed to get clk` | راجع `clocks` و `clock-names` properties |

---

#### 5. الـ Device Tree Debugging

**التحقق من الـ DT يطابق الـ Hardware:**

```bash
# عرض الـ DT المحمّل فعلياً في الـ runtime
ls /sys/firmware/devicetree/base/

# البحث عن كل nodes من نوع fixed-clock
find /sys/firmware/devicetree/base/ -name "compatible" \
  -exec grep -l "fixed-clock" {} \;

# قراءة clock-frequency من DT node مباشرة
hexdump -C /sys/firmware/devicetree/base/clocks/osc24m/clock-frequency
# Output (مثال 24MHz = 0x016E3600):
# 00000000  01 6e 36 00                                       |.n6.|

# قراءة أوضح
od -A x -t x4z /sys/firmware/devicetree/base/clocks/osc24m/clock-frequency
```

**مقارنة الـ DT source بالـ runtime:**

```bash
# فكّ ضغط الـ DTB المحمّل
dtc -I fs /sys/firmware/devicetree/base > /tmp/current.dts 2>/dev/null

# ابحث عن fixed-clock nodes
grep -A10 'fixed-clock' /tmp/current.dts

# مثال output متوقع:
# osc24m: osc24m {
#     compatible = "fixed-clock";
#     #clock-cells = <0>;
#     clock-frequency = <24000000>;
#     clock-output-names = "osc24m";
# };
```

**التحقق من الـ provider مسجّل صح:**

```bash
# تأكد أن الـ clock provider اتسجّل
cat /sys/kernel/debug/clk/clk_summary | grep -i osc

# تأكد أن الـ OF provider موجود
# (مش في sysfs مباشرة، لكن لو الـ consumer اشتغل = الـ provider شغّال)
dmesg | grep -i 'fixed.*clk\|clk.*fixed\|clock-frequency'
```

**أمثلة DTS صحيحة وغلط:**

```dts
/* صح */
osc24m: osc24m {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;        /* 24 MHz */
    clock-accuracy = <50000>;           /* 50 ppm = 50000 ppb */
    clock-output-names = "osc24m";
};

/* غلط — ناقص clock-frequency */
bad_osc: bad_osc {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    /* clock-frequency مش موجود → -EIO */
};

/* غلط — #clock-cells ناقص */
bad_osc2: bad_osc2 {
    compatible = "fixed-clock";
    clock-frequency = <24000000>;
    /* #clock-cells مش موجود → consumers مش هيعرفوا يحصلوا عليه */
};
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**1. فحص شامل سريع:**
```bash
#!/bin/bash
# quick_clk_debug.sh — فحص سريع لحالة الـ fixed-rate clocks

echo "=== Clock Summary ==="
cat /sys/kernel/debug/clk/clk_summary | grep -E "(clock|clk|osc|xtal)" -i

echo ""
echo "=== Fixed Clocks في DT ==="
find /sys/firmware/devicetree/base -name "compatible" | while read f; do
    if grep -q "fixed-clock" "$f" 2>/dev/null; then
        dir=$(dirname "$f")
        echo "Node: $dir"
        echo -n "  clock-frequency: "
        if [ -f "$dir/clock-frequency" ]; then
            od -A n -t u4 "$dir/clock-frequency" | tr -d ' '
            echo " Hz"
        else
            echo "MISSING!"
        fi
    fi
done

echo ""
echo "=== Kernel Messages المتعلقة بالـ Clocks ==="
dmesg | grep -i 'clk\|clock\|fixed' | tail -30
```

**2. تفعيل الـ ftrace للـ clocks:**
```bash
#!/bin/bash
# enable_clk_trace.sh

cd /sys/kernel/tracing

# صفّي الـ trace السابق
echo > trace

# فعّل events الـ clk
echo 1 > events/clk/enable

# ابدأ
echo 1 > tracing_on
echo "Tracing enabled — اضغط Enter لإيقاف وقراءة النتائج"
read

echo 0 > tracing_on
cat trace | grep -i 'fixed\|osc\|xtal' | head -50
```

**3. التحقق من DT node لـ fixed clock بالتفصيل:**
```bash
#!/bin/bash
# check_fixed_clk_dt.sh <node-path>
# مثال: ./check_fixed_clk_dt.sh /sys/firmware/devicetree/base/clocks/osc24m

NODE="${1:-/sys/firmware/devicetree/base/clocks}"

for dir in $NODE/*/; do
    comp="$dir/compatible"
    if [ -f "$comp" ] && grep -q "fixed-clock" "$comp" 2>/dev/null; then
        echo "=== $(basename $dir) ==="
        echo -n "  compatible: "; cat "$comp"; echo
        if [ -f "$dir/clock-frequency" ]; then
            freq=$(od -A n -t u4 "$dir/clock-frequency" | tr -d ' \n')
            echo "  clock-frequency: $freq Hz ($(echo "scale=6; $freq/1000000" | bc) MHz)"
        else
            echo "  clock-frequency: *** MISSING ***"
        fi
        if [ -f "$dir/clock-accuracy" ]; then
            acc=$(od -A n -t u4 "$dir/clock-accuracy" | tr -d ' \n')
            echo "  clock-accuracy: $acc ppb"
        fi
        if [ -f "$dir/clock-output-names" ]; then
            echo -n "  clock-output-names: "; cat "$dir/clock-output-names"; echo
        fi
    fi
done
```

**4. مقارنة الـ DT frequency بالـ Kernel:**
```bash
#!/bin/bash
# compare_dt_vs_kernel.sh
# يقارن الـ rate في DT مع اللي الـ kernel شايفه فعلاً

echo "Clock Name | DT Freq (Hz) | Kernel Rate (Hz) | Match?"
echo "-----------|--------------|------------------|-------"

find /sys/firmware/devicetree/base -name "compatible" | while read f; do
    if grep -q "fixed-clock" "$f" 2>/dev/null; then
        dir=$(dirname "$f")
        name=$(basename "$dir")

        # DT frequency
        if [ -f "$dir/clock-frequency" ]; then
            dt_freq=$(od -A n -t u4 "$dir/clock-frequency" | tr -d ' \n')
        else
            dt_freq="N/A"
        fi

        # clock-output-names (الاسم الفعلي)
        if [ -f "$dir/clock-output-names" ]; then
            clk_name=$(cat "$dir/clock-output-names" | tr -d '\0')
        else
            clk_name="$name"
        fi

        # Kernel rate
        kernel_rate_file="/sys/kernel/debug/clk/$clk_name/clk_rate"
        if [ -f "$kernel_rate_file" ]; then
            kernel_rate=$(cat "$kernel_rate_file")
        else
            kernel_rate="NOT_FOUND"
        fi

        # مقارنة
        if [ "$dt_freq" = "$kernel_rate" ]; then
            match="YES"
        else
            match="*** NO ***"
        fi

        echo "$clk_name | $dt_freq | $kernel_rate | $match"
    fi
done
```

**5. تفعيل dynamic debug للـ CCF أثناء الـ runtime:**
```bash
# تفعيل كل debug messages في clk subsystem
echo 'file drivers/clk/clk.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/clk/clk-fixed-rate.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ output في real-time
dmesg -w | grep -i clk
```

**6. استخدام `perf` لتتبع استدعاءات الـ clock:**
```bash
# تتبع كل استدعاءات clk_get و clk_prepare_enable
perf trace -e 'clk:*' sleep 5

# أو بـ perf record
perf record -e 'clk:clk_enable,clk:clk_disable' -a sleep 10
perf script | grep -i fixed
```

**مثال output متوقع من `clk_summary` على Raspberry Pi:**
```
                                 enable  prepare  protect                                duty
clock                          count    count    count        rate   accuracy phase  cycle
--------------------------------------------------------------------------------------------
 osc                               5        5        0    19200000          0     0  50000
    clk_pwm                        0        0        0    19200000          0     0  50000
    fixed_clk                      1        1        0   100000000          0     0  50000
```

**تفسير الـ output:**
- `enable count = 1` → الـ clock شغّال ومطلوب من consumer واحد
- `prepare count = 1` → الـ parent متحضّر
- `rate = 100000000` → 100 MHz كما هو محدد في الـ DTS
- `accuracy = 0` → مش محدد في الـ DTS (يعني `fixed_accuracy = 0`)
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على بورد RK3562 — Industrial Gateway

#### العنوان
**الـ UART بيطلع بيانات غلط على gateway صناعي بسبب `clock-frequency` غلط في DT**

#### السياق
شركة بتبني industrial gateway على RK3562. الـ UART0 بيتكلم مع sensor خارجي على 115200 baud. البورد اتعمل bring-up وكل حاجة شغالة ماشياً، لحد ما الـ sensor بدأ يبعت garbage data.

#### المشكلة
الـ UART driver بيحسب الـ baud rate divisor بناءً على الـ clock rate. لو الـ clock rate اللي الـ framework شايفه غلط، الـ divisor بيتحسب غلط وبيطلع framing errors على الـ serial line.

الـ DT فيه:

```dts
/* في board DTS */
uart0_clk: uart0-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;  /* غلط — المفروض 48000000 */
};
```

#### التحليل
لما الكيرنل يبوت، `CLK_OF_DECLARE` بيسجل `of_fixed_clk_setup` كـ handler لـ `"fixed-clock"`:

```c
CLK_OF_DECLARE(fixed_clk, "fixed-clock", of_fixed_clk_setup);
```

الـ `of_fixed_clk_setup` بتكال early في boot وبتستدعي `_of_fixed_clk_setup`:

```c
static struct clk_hw *_of_fixed_clk_setup(struct device_node *node)
{
    u32 rate;
    /* بتقرأ clock-frequency من DT */
    if (of_property_read_u32(node, "clock-frequency", &rate))
        return ERR_PTR(-EIO);
    /* ...
       rate هنا = 24000000 بدل 48000000
    */
    hw = clk_hw_register_fixed_rate_with_accuracy(NULL, clk_name, NULL,
                                                   0, rate, accuracy);
```

الـ `fixed_rate` اتحط 24 MHz في `struct clk_fixed_rate`. وأي caller بيعمل `clk_get_rate()` هيجيله:

```c
static unsigned long clk_fixed_rate_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    return to_clk_fixed_rate(hw)->fixed_rate; /* يرجع 24000000 دايماً */
}
```

الـ UART driver بيحسب: `divisor = rate / (16 * baud)` → `24000000 / (16 * 115200) = 13` بدل `26`. النتيجة: الـ baud الفعلي = `24000000 / (16 * 13) ≈ 115384` — فرق كافي يعمل framing errors مع بعض sensors.

#### الحل

**1. صحح الـ DT:**

```dts
uart0_clk: uart0-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <48000000>;  /* القيمة الصح */
};
```

**2. تحقق بعد البوت:**

```bash
# شوف الـ clock rate اللي الـ framework شايفه
cat /sys/kernel/debug/clk/uart0-clk/clk_rate

# أو عن طريق clk_summary
cat /sys/kernel/debug/clk/clk_summary | grep uart
```

**3. لو مش قادر تعدل DT وعايز تتحقق runtime:**

```bash
# شوف الـ node في sysfs
find /sys/firmware/devicetree -name "uart0-clk" -exec cat {}/clock-frequency \; | od -An -tu4
```

#### الدرس المستفاد
**الـ `clk-fixed-rate` لا يتحقق من أي حاجة** — هو بيثق في القيمة اللي في DT بشكل كامل. أي خطأ في `clock-frequency` بيتنقل مباشرة لأي driver بياخد الـ rate من الـ framework. لازم تتحقق من الـ datasheet والـ schematic قبل ما تكتب الـ DT.

---

### السيناريو 2: SPI flash مش بتتعرف على AM62x — IoT Sensor Board

#### العنوان
**الـ SPI controller بيرفض يشتغل بعد bring-up على AM62x بسبب غياب `clock-frequency` في DT**

#### السياق
فريق بيعمل bring-up لـ IoT sensor board على TI AM62x. الـ SPI0 متوصل بـ W25Q64 flash. الـ driver بيلود من غير مشاكل ظاهرة، لكن أي محاولة للـ read/write بتفشل.

#### المشكلة
الـ SPI clock node في DT منسي مش بيتقرأ صح:

```dts
spi0_ref_clk: spi0-ref-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    /* clock-frequency منسية خالص! */
};
```

#### التحليل
في `_of_fixed_clk_setup`:

```c
if (of_property_read_u32(node, "clock-frequency", &rate))
    return ERR_PTR(-EIO);  /* هيرجع -EIO لو الـ property مش موجودة */
```

الـ function بترجع `ERR_PTR(-EIO)` فوراً. الـ `of_fixed_clk_setup` (اللي بتتكال early) بتتجاهل الـ return value:

```c
void __init of_fixed_clk_setup(struct device_node *node)
{
    _of_fixed_clk_setup(node);  /* الـ error بيتحط في الهواء */
}
```

الـ clock مش بيتسجل خالص. الـ SPI driver لما يحاول `clk_get()` هيجيب `-ENOENT` أو `NULL`، وبالتالي التهيئة بتفشل بشكل صامت أو بيتجاهل الـ clock ويشتغل بـ default rate خاطئة.

**تشخيص:**

```bash
# شوف kernel log أول حاجة
dmesg | grep -i "fixed-clock\|spi0\|clk"

# Expected output لو الـ node فشل:
# clk: failed to register clk for spi0-ref-clk: -5

# شوف لو الـ clock موجود في clk tree
cat /sys/kernel/debug/clk/clk_summary | grep spi0
```

#### الحل

```dts
spi0_ref_clk: spi0-ref-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <100000000>;  /* 100 MHz حسب AM62x datasheet */
};
```

**تأكيد:**

```bash
cat /sys/kernel/debug/clk/spi0-ref-clk/clk_rate
# Output: 100000000
```

#### الدرس المستفاد
الـ `of_fixed_clk_setup` بتتجاهل الـ error من `_of_fixed_clk_setup` — الـ -EIO بيروح في الهواء لأن الـ function هي `void`. لازم دايماً تراجع `dmesg` بعد البوت وتتأكد إن كل الـ fixed clocks اتسجلت صح قبل ما تبدأ تـ debug الـ peripherals.

---

### السيناريو 3: HDMI مش بيطلع صورة على Android TV Box — Allwinner H616

#### العنوان
**شاشة سودا على Android TV box بسبب `clock-accuracy` بيخلي الـ HDMI PLL يرفض الـ parent**

#### السياق
شركة بتبني Android TV box على Allwinner H616. الـ HDMI بيشتغل تمام على بعض الشاشات ومش شغال على تانية. بعد تحقيق، الفارق هو إن الشاشات الصعبة بتـ negotiate HDMI 2.0 وبتطلب clock accuracy أحسن.

#### المشكلة
الـ 24 MHz crystal clock معرّف في DT بـ accuracy مش مضبوطة:

```dts
osc24M: osc24M-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;
    clock-accuracy = <50000>;  /* 50000 ppb = 50 ppm — أكبر من المطلوب */
};
```

الـ HDMI PLL driver بيتحقق من accuracy الـ parent قبل ما يقبله كـ source لـ pixel clock.

#### التحليل
لما الـ HDMI driver يسأل عن accuracy:

```c
static unsigned long clk_fixed_rate_recalc_accuracy(struct clk_hw *hw,
        unsigned long parent_accuracy)
{
    struct clk_fixed_rate *fixed = to_clk_fixed_rate(hw);

    if (fixed->flags & CLK_FIXED_RATE_PARENT_ACCURACY)
        return parent_accuracy;

    return fixed->fixed_accuracy;  /* هيرجع 50000 ppb */
}
```

الـ `fixed_accuracy` اتملت من DT في `_of_fixed_clk_setup`:

```c
of_property_read_u32(node, "clock-accuracy", &accuracy);
/* accuracy = 50000 */

hw = clk_hw_register_fixed_rate_with_accuracy(NULL, clk_name, NULL,
                                               0, rate, accuracy);
/* fixed->fixed_accuracy = 50000 */
```

الـ HDMI PLL driver بيعمل `clk_get_accuracy()` على الـ parent ولو لاقى > 20000 ppb بيرفض يستخدمه لـ high-res pixel clocks.

**تشخيص:**

```bash
# شوف الـ accuracy المسجلة
cat /sys/kernel/debug/clk/osc24M-clk/clk_accuracy
# Output: 50000

# شوف كل الـ clocks وأي منها بـ high accuracy
cat /sys/kernel/debug/clk/clk_summary
```

#### الحل

**لو الـ crystal فعلاً ±20 ppm:**

```dts
osc24M: osc24M-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <24000000>;
    clock-accuracy = <20000>;  /* 20 ppm — حسب crystal datasheet */
};
```

**لو عايز الـ PLL يحسب accuracy من الـ parent بدل ما تحدده ثابت:**

```c
/* في الـ driver اللي بيسجل الـ clock */
hw = __clk_hw_register_fixed_rate(dev, NULL, "osc24M", NULL, NULL,
                                   NULL, 0, 24000000, 0,
                                   CLK_FIXED_RATE_PARENT_ACCURACY, /* flags */
                                   false);
```

لما `CLK_FIXED_RATE_PARENT_ACCURACY` يتبعت، الـ `recalc_accuracy` بيرجع `parent_accuracy` بدل `fixed->fixed_accuracy`.

#### الدرس المستفاد
الـ `clock-accuracy` في ppb مش ppm — 50000 ppb = 50 ppm. لازم تراجع crystal datasheet وتحط القيمة الصح. كمان الـ `CLK_FIXED_RATE_PARENT_ACCURACY` flag موجود للحالات اللي الـ accuracy بتيجي من الـ parent وملقتش فيها قيمة fixed.

---

### السيناريو 4: Memory Leak عند Driver Unbind على i.MX8 — Automotive ECU

#### العنوان
**memory leak في كل مرة الـ ECU بيعمل hot-unplug لـ I2C controller على i.MX8**

#### السياق
automotive ECU بيشتغل على i.MX8M. الـ system محتاج يعمل dynamic disable/enable لـ I2C controller بأكمله (driver unbind/bind) كجزء من power management flow. بعد 100 cycle، الـ kernel memory بيبدأ يقل.

#### المشكلة
الـ I2C driver بيستخدم `clk_register_fixed_rate` (الـ API القديم مش `devm`) لكن بعدين بيعمل `clk_unregister` بس من غير ما يعمل `clk_unregister_fixed_rate`.

```c
/* كود الـ I2C driver (غلط) */
static int imx8_i2c_probe(struct platform_device *pdev)
{
    priv->ref_clk = clk_register_fixed_rate(&pdev->dev, "i2c-ref",
                                             NULL, 0, 66000000);
    /* ... */
}

static int imx8_i2c_remove(struct platform_device *pdev)
{
    clk_unregister(priv->ref_clk);  /* غلط! */
    /* المفروض: clk_unregister_fixed_rate(priv->ref_clk) */
}
```

#### التحليل
`clk_register_fixed_rate` بتكال `__clk_hw_register_fixed_rate` مع `devm=false`:

```c
/* المسار: clk_register_fixed_rate → __clk_hw_register_fixed_rate */
fixed = kzalloc(sizeof(*fixed), GFP_KERNEL);  /* heap allocation */
/* ... */
/* fixed->hw.clk بيتسجل في الـ framework */
```

الـ `struct clk_fixed_rate` اتحجز بـ `kzalloc`. لما بيتعمل remove:

```c
/* clk_unregister بتحرر الـ struct clk الداخلي بس */
/* الـ struct clk_fixed_rate اللي اتعمله kzalloc مش بيتحرر! */
```

المفروض يُستخدم:

```c
void clk_unregister_fixed_rate(struct clk *clk)
{
    struct clk_hw *hw;
    hw = __clk_get_hw(clk);
    if (!hw)
        return;
    clk_unregister(clk);
    kfree(to_clk_fixed_rate(hw));  /* ← هنا بيحرر الـ struct clك_fixed_rate */
}
```

**تشخيص:**

```bash
# شوف الـ memory leak
cat /proc/meminfo | grep MemFree
# اعمل 10 bind/unbind cycles
for i in $(seq 1 10); do
    echo "imx8-i2c" > /sys/bus/platform/drivers/imx8-i2c/unbind
    echo "imx8-i2c.0" > /sys/bus/platform/drivers/imx8-i2c/bind
done
cat /proc/meminfo | grep MemFree
# لو الـ MemFree نقص → leak موجود

# كمان ممكن تستخدم kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الحل

```c
static void imx8_i2c_remove(struct platform_device *pdev)
{
    /* استخدم الـ pair الصح */
    clk_unregister_fixed_rate(priv->ref_clk);  /* بيعمل kfree للـ struct */
}
```

**أو أحسن من كده: استخدم `devm_clk_hw_register_fixed_rate`** — الـ `devres` بيتكلف بالتحرير تلقائياً لما الـ device يتـ removed:

```c
/* في probe */
hw = devm_clk_hw_register_fixed_rate(&pdev->dev, "i2c-ref", NULL, 0, 66000000);

/* في remove: مش محتاج تعمل حاجة — devres بتشيل */
```

الـ `devm_clk_hw_register_fixed_rate_release` بيتكال تلقائياً:

```c
static void devm_clk_hw_register_fixed_rate_release(struct device *dev, void *res)
{
    struct clk_fixed_rate *fix = res;
    clk_hw_unregister(&fix->hw);
    /* devres بعدين بتعمل kfree(res) */
}
```

#### الدرس المستفاد
كل `clk_register_fixed_rate` لازم يقابله `clk_unregister_fixed_rate` مش `clk_unregister`. والأحسن: استخدم `devm_*` variants دايماً في الـ platform drivers عشان تتجنب هذا النوع من الأخطاء.

---

### السيناريو 5: I2C Sensor بطيء جداً على STM32MP1 — Custom Bring-Up Board

#### العنوان
**الـ I2C transactions بطيئة ×4 على STM32MP1 بسبب `clock-output-names` بيخلي الـ driver يـ bind على الغلط**

#### السياق
فريق bring-up بيشتغل على custom board على STM32MP1. فيه crystal خارجي 48 MHz للـ I2C high-speed. الـ I2C transactions بتشتغل لكن بطيئة جداً — بيمشوا بـ 100 kHz بدل 400 kHz.

#### المشكلة
فيه clock node تاني في DT اسمه "hse-clk" بـ 12 MHz، والـ new crystal node فيه:

```dts
hse_ext: hse-ext-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <48000000>;
    /* clock-output-names مش موجود */
};
```

والـ I2C node بيـ reference:

```dts
&i2c4 {
    clocks = <&hse_ext>;
    clock-names = "hse-clk";  /* ده اسم الـ property مش اسم الـ clock */
};
```

#### التحليل
في `_of_fixed_clk_setup`:

```c
static struct clk_hw *_of_fixed_clk_setup(struct device_node *node)
{
    const char *clk_name = node->name;  /* default: اسم الـ node في DT */
    /* ... */
    of_property_read_string(node, "clock-output-names", &clk_name);
    /* لو "clock-output-names" مش موجود، clk_name يفضل = node->name */
```

الـ `node->name` للـ node `hse-ext-clk` هو `"hse-ext-clk"`. لكن الـ I2C driver بيـ lookup بالاسم `"hse-clk"` (من الـ old 12 MHz clock). النتيجة: الـ framework بيديه الـ clock القديم 12 MHz بدل 48 MHz الجديد.

**تشخيص:**

```bash
# شوف إيه الـ clock rate اللي الـ I2C driver بياخده فعلاً
cat /sys/kernel/debug/clk/clk_summary | grep i2c4

# شوف كل الـ registered fixed clocks وأسمائها
cat /sys/kernel/debug/clk/clk_summary | grep -A2 "hse"

# شوف الـ I2C speed فعلياً
i2cdetect -l
# أو
cat /sys/bus/i2c/devices/i2c-4/*/name
```

#### الحل

**أضف `clock-output-names` في DT عشان تتحكم في اسم الـ clock المسجل:**

```dts
hse_ext: hse-ext-clk {
    compatible = "fixed-clock";
    #clock-cells = <0>;
    clock-frequency = <48000000>;
    clock-output-names = "hse-clk";  /* ← ده اللي بيتسجل في الـ framework */
};
```

دلوقتي `_of_fixed_clk_setup` هتقرأ `"hse-clk"` من `clock-output-names` وتسجل الـ clock بيه:

```c
of_property_read_string(node, "clock-output-names", &clk_name);
/* clk_name = "hse-clk" */

hw = clk_hw_register_fixed_rate_with_accuracy(NULL, clk_name, NULL,
                                               0, rate, accuracy);
/* registered as "hse-clk" @ 48 MHz */
```

**تأكيد:**

```bash
cat /sys/kernel/debug/clk/hse-clk/clk_rate
# Output: 48000000

# I2C speed دلوقتي صح
i2cdetect -F 4
# Should show: functionality: 400kbps Fast-mode
```

#### الدرس المستفاد
الـ `clock-output-names` property في `"fixed-clock"` DT node مش decorative — هو بالظبط الاسم اللي الـ framework بيسجل الـ clock بيه وبيـ lookup بيه. لو الـ drivers بتـ reference الـ clock بالاسم (مش بالـ phandle وبس)، لازم `clock-output-names` يطابق الاسم المتوقع.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي اتكتبت عن الـ Common Clock Framework وبتغطي الـ fixed-rate clock جوّاه:

| المقال | الوصف |
|--------|--------|
| [A common clock framework](https://lwn.net/Articles/472998/) | أول مقال بيشرح الـ framework لما اتقدم للدمج — بيتكلم عن الـ `clk_hw_ops` وتصميم الـ API |
| [common clk framework](https://lwn.net/Articles/486841/) | مقال أعمق بيشرح الـ `clk_hw`، الـ `clk_ops`، وإزاي الـ struct clk مقسوم عن الـ hardware |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | الـ documentation الرسمية اللي اتنشرت مع الـ framework — بتشرح إزاي تكتب driver جديد |
| [Common struct clk implementation, v5](https://lwn.net/Articles/392902/) | نسخة مبكرة من الـ proposal — بتوضح التطور من `DEFINE_CLK_FIXED` لحد الـ API الحالية |
| [Common struct clk implementation, v10](https://lwn.net/Articles/421677/) | النسخة اللي فيها Jeremy Kerr أضاف الـ fixed-rate support الأساسي |
| [clk: implement clock rate protection mechanism](https://lwn.net/Articles/740484/) | بيتكلم عن الـ `CLK_FIXED_RATE_PARENT_ACCURACY` وحماية الـ rate |

---

### التوثيق الرسمي للـ kernel

#### داخل شجرة المصدر

**الـ `clk-fixed-rate.c`** نفسه جزء من:

```
drivers/clk/clk-fixed-rate.c         ← الملف المدروس
include/linux/clk-provider.h          ← تعريف struct clk_fixed_rate وكل الـ macros
Documentation/driver-api/clk.rst      ← توثيق الـ Common Clock Framework
Documentation/devicetree/bindings/clock/fixed-clock.yaml   ← الـ DT binding الرسمي
Documentation/devicetree/bindings/clock/clock-bindings.txt ← قواعد عامة للـ clock bindings
```

#### الـ kernel.org documentation

- [Common Clock Framework — docs.kernel.org](https://docs.kernel.org/driver-api/clk.html)
- [fixed-clock DT binding](https://www.kernel.org/doc/Documentation/devicetree/bindings/clock/fixed-clock.txt)
- [fixed-factor-clock DT binding](https://www.kernel.org/doc/Documentation/devicetree/bindings/clock/fixed-factor-clock.txt)
- [clock-bindings.txt (general)](https://www.kernel.org/doc/Documentation/clk.txt)

---

### Commits مهمة في تاريخ الملف

الـ framework اتدمج في kernel 3.4 (مايو 2012). أهم الـ commits:

| الـ patch | الوصف |
|-----------|--------|
| [PATCH v7 2/3: clk: introduce the common clock framework](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) | الـ commit الأصلي لإدخال الـ framework — بيتضمن الـ fixed-rate implementation |
| [PATCH v7 3/3: clk: basic clock hardware types](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00027.html) | الـ patch اللي أضاف الـ `clk_fixed_rate_ops` و`clk_register_fixed_rate` |
| [Re: PATCH v6 3/3: clk: basic clock hardware types](https://lkml.iu.edu/hypermail/linux/kernel/1203.1/03756.html) | نقاش مهم في الـ mailing list حول تصميم الـ basic types |
| [clk: Register clkdev after setup of fixed-rate clocks](https://patchwork.kernel.org/patch/9370703/) | patch لاحق بيصلح ترتيب التسجيل |

الـ [GitHub history](https://github.com/torvalds/linux/blob/master/drivers/clk/clk-fixed-rate.c) للملف بيكشف كل التغييرات من البداية لحد دلوقتي.

---

### نقاشات الـ mailing list

- [LKML: PATCH v7 — common clock framework introduction](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) — النقاش الأصلي بين Jeremy Kerr وMike Turquette ومجتمع الـ kernel
- [LKML: basic clock hardware types discussion](https://lkml.iu.edu/hypermail/linux/kernel/1203.1/03756.html) — بيتناقشوا في تصميم الـ `struct clk_fixed_rate` وفصل الـ ops
- [Patchwork: clk-fixed-rate clkdev fix](https://patchwork.kernel.org/patch/9370703/)

---

### مصادر eLinux.org

| المرجع | الوصف |
|--------|--------|
| [Common Clock Framework BoF (PDF)](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | عرض تقديمي من مؤتمر ELC عن الـ framework |
| [Generic Linux CLK Framework — CELF 2009 (PDF)](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | الجذور التاريخية للـ clock framework قبل الـ Common CLK |
| [Linux clock management framework — ELC 2007 (PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | أقدم عرض عن إدارة الـ clocks في Linux |

---

### مصادر kernelnewbies.org

الـ kernelnewbies.org بيذكر الـ Common Clock Framework في changelog كتير من الإصدارات:

- [Linux 5.9 Changelog](https://kernelnewbies.org/Linux_5.9) — تحسينات في الـ CLK framework
- [Linux 6.7 Changelog](https://kernelnewbies.org/Linux_6.7) — تحديثات في الـ COMMON CLK

للمبتدئين، أفضل نقطة بداية هي [kernelnewbies.org](https://kernelnewbies.org) مع البحث عن `"common clock"`.

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: بيتكلم عن الـ hardware support بشكل عام
- الكتاب مجاني على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- ملاحظة: الـ Common CLK Framework أحدث من LDD3، فالفصول مش بتغطيه مباشرة — لازم تجمع بين الكتاب والـ kernel docs

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Memory Management — مهم لفهم `kzalloc` و`devres`
- **الفصل 14**: The Block I/O Layer — بيشرح مفهوم الـ `container_of` pattern
- بيدي خلفية قوية عن الـ kernel internals اللي الـ clock framework بيستخدمها

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 13**: بيتكلم عن الـ board support وإعداد الـ clocks في الـ embedded systems
- بيشرح إزاي الـ fixed-rate clocks بتتعرف في الـ Device Tree

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- الفصل الخاص بالـ platform drivers مفيد لفهم `platform_driver` و`of_device_id`

---

### عروض تقديمية من مؤتمرات ELC

| العرض | المصدر |
|-------|--------|
| [Common Clock Framework: How to Use It — Gregory Clement, Free Electrons (ELCE 2013)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | أفضل عرض عملي للـ framework — بيشرح `clk_hw_register_fixed_rate` بالتفصيل |

---

### مصطلحات البحث

لو عايز تلاقي معلومات أكتر، استخدم الـ search terms دي:

```
linux kernel common clock framework
clk_hw_register_fixed_rate
struct clk_fixed_rate
CLK_OF_DECLARE fixed-clock
of_clk_add_hw_provider
clk-provider.h clk_ops
devres clock driver linux
linux device tree clock-frequency binding
CONFIG_COMMON_CLK
clk_fixed_rate_recalc_rate
```

على المواقع دي:
- `lwn.net` — للمقالات التقنية العميقة
- `lore.kernel.org` — لأرشيف الـ mailing list الكامل
- `elixir.bootlin.com` — للـ cross-reference في كود الـ kernel
- `patchwork.kernel.org` — لتتبع الـ patches والتغييرات

---

### روابط سريعة للـ cross-reference

- [elixir.bootlin.com — clk-fixed-rate.c](https://elixir.bootlin.com/linux/latest/source/drivers/clk/clk-fixed-rate.c) — شوف كل الملفات اللي بتستخدم الـ functions دي
- [elixir.bootlin.com — struct clk_fixed_rate](https://elixir.bootlin.com/linux/latest/ident/clk_fixed_rate) — كل الأماكن اللي بيتعرف فيها الـ struct
- [GitHub — torvalds/linux clk-fixed-rate.c](https://github.com/torvalds/linux/blob/master/drivers/clk/clk-fixed-rate.c) — الـ history الكامل للـ commits
## Phase 8: Writing simple module

### الفكرة

هنعمل module بيستخدم **kprobe** يعمل hook على الدالة `__clk_hw_register_fixed_rate` اللي هي الدالة الأساسية اللي بتتسجل فيها كل fixed-rate clocks في الـ kernel. كل مرة أي driver يسجل clock ثابت، الـ kprobe بتاعنا بيطبع اسم الـ clock والـ rate بتاعته — ده مفيد جداً لو عايز تشوف إيه الـ clocks اللي بتتسجل أثناء الـ boot.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_fixed_clk.c
 * Traces every fixed-rate clock registration via __clk_hw_register_fixed_rate
 */

/* kprobes API */
#include <linux/kprobes.h>
/* pr_info, printk */
#include <linux/kernel.h>
/* module_init / module_exit macros */
#include <linux/module.h>
/* clk_hw, clk_fixed_rate structs */
#include <linux/clk-provider.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Trace fixed-rate clock registrations using kprobe");

/*
 * pre_handler — runs just BEFORE __clk_hw_register_fixed_rate executes.
 *
 * Prototype of the hooked function:
 *   struct clk_hw *__clk_hw_register_fixed_rate(
 *       struct device *dev,
 *       struct device_node *np,
 *       const char *name,          <- regs: arg2 on x86-64 (rdx)
 *       const char *parent_name,
 *       const struct clk_hw *parent_hw,
 *       const struct clk_parent_data *parent_data,
 *       unsigned long flags,
 *       unsigned long fixed_rate,  <- regs: arg7 on x86-64 (r9 + stack)
 *       unsigned long fixed_accuracy,
 *       unsigned long clk_fixed_flags,
 *       bool devm)
 *
 * بنستخدم regs_get_kernel_argument() عشان نجيب الـ arguments بشكل portable
 * بدل ما نبص على registers بشكل manual.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* arg index 2 => name (0-based: dev=0, np=1, name=2) */
    const char *name = (const char *)regs_get_kernel_argument(regs, 2);

    /* arg index 7 => fixed_rate */
    unsigned long fixed_rate = (unsigned long)regs_get_kernel_argument(regs, 7);

    /* arg index 8 => fixed_accuracy */
    unsigned long fixed_accuracy = (unsigned long)regs_get_kernel_argument(regs, 8);

    /* arg index 0 => dev pointer, just check if it's NULL */
    void *dev = (void *)regs_get_kernel_argument(regs, 0);

    /* print meaningful info about the clock being registered */
    pr_info("kprobe_fixed_clk: registering clock \"%s\" "
            "rate=%lu Hz accuracy=%lu ppb dev=%s\n",
            name ? name : "(null)",
            fixed_rate,
            fixed_accuracy,
            dev ? "yes" : "no(OF)");

    /* return 0 = let the original function execute normally */
    return 0;
}

/* kprobe struct — symbol_name بدل address عشان أوضح وأسهل */
static struct kprobe kp = {
    .symbol_name = "__clk_hw_register_fixed_rate",
    .pre_handler = handler_pre,
};

static int __init kprobe_fixed_clk_init(void)
{
    int ret;

    /* register the kprobe with the kernel */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_fixed_clk: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_fixed_clk: planted kprobe on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit kprobe_fixed_clk_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتشال من الذاكرة،
     * لو سبنا الـ kprobe فعّال والـ handler اتشال هيحصل kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("kprobe_fixed_clk: kprobe removed\n");
}

module_init(kprobe_fixed_clk_init);
module_exit(kprobe_fixed_clk_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `linux/kernel.h` | بيجيب `pr_info` و `pr_err` |
| `linux/module.h` | لازم لأي module — بيعرّف `module_init/exit` |
| `linux/clk-provider.h` | بيعرّف `struct clk_hw` و `struct clk_fixed_rate` لو احتجناهم |

**الـ** `linux/clk-provider.h` مش مطلوب مباشرةً هنا لأننا بنشتغل على الـ raw registers، بس بيوضح context الـ types المستخدمة.

---

#### الـ `handler_pre`

الدالة بتتشغل **قبل** تنفيذ `__clk_hw_register_fixed_rate`، يعني الـ arguments لسه موجودة في الـ registers ولم يتغيروا. بنستخدم `regs_get_kernel_argument(regs, N)` عشان الكود يشتغل على x86-64 وARM64 وغيرهم بدون تعديل — ده abstraction من الـ kernel نفسه.

الـ `return 0` مهم — لو رجعنا قيمة غير صفر الـ kprobe framework هيعتبرها خطأ.

---

#### اختيار الـ Symbol

اخترنا `__clk_hw_register_fixed_rate` (الـ double underscore) مش الـ macro `clk_hw_register_fixed_rate`، لأن الـ macro مجرد wrapper بيستدعي الدالة الحقيقية — والـ kprobe بيحتاج **اسم دالة real** موجودة في الـ kernel symbol table.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe`**: بيحط breakpoint على أول instruction في الدالة المستهدفة. لو الدالة `inline`ت أو `notrace`ت هيرجع error — هنا `__clk_hw_register_fixed_rate` exported وليست `notrace`، يعني آمنة.
- **`unregister_kprobe`**: إجباري في الـ `exit` — لو الـ module اتـ`rmmod` والـ kprobe لسه active، أي استدعاء بعدها هيلاقي handler code مش موجود في الذاكرة ويعمل panic.

---

### Makefile

```makefile
obj-m += kprobe_fixed_clk.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميل
sudo insmod kprobe_fixed_clk.ko

# شوف الـ output (لو في clocks بتتسجل بعد التحميل)
sudo dmesg | grep kprobe_fixed_clk

# لو عايز تشغل مثال: probe على clock موجود عن طريق load driver
# أو على embedded system أثناء الـ boot

# إزالة الـ module
sudo rmmod kprobe_fixed_clk
```

**مثال output متوقع على embedded board:**

```
[    2.341] kprobe_fixed_clk: planted kprobe on __clk_hw_register_fixed_rate @ ffffffff81234abc
[    2.512] kprobe_fixed_clk: registering clock "osc24M" rate=24000000 Hz accuracy=0 ppb dev=no(OF)
[    2.513] kprobe_fixed_clk: registering clock "osc32k" rate=32768 Hz accuracy=50000 ppb dev=no(OF)
[    2.701] kprobe_fixed_clk: registering clock "pll-cpu" rate=1008000000 Hz accuracy=0 ppb dev=yes
```

---

### ملاحظة على الـ CONFIG

لازم الـ kernel يكون مبني بـ:
```
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENTS=y   # optional, for ftrace integration
```

تتأكد بـ:
```bash
grep CONFIG_KPROBES /boot/config-$(uname -r)
```
