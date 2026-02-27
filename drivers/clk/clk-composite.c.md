## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `clk-composite.c`** ينتمي لـ **Common Clock Framework (CCF)** — الـ subsystem المركزي في Linux kernel اللي بيتحكم في كل الـ clocks الخاصة بالـ hardware. موقعه: `drivers/clk/`.

---

### المشكلة اللي بيحلها

تخيل إنك بتبني راديو. الراديو ده عنده 3 أجزاء:

1. **مفتاح الاختيار** — تختار المحطة (مصدر الإشارة).
2. **مقبض الصوت** — تحدد سرعة/تردد الإشارة.
3. **زر التشغيل/الإيقاف** — تشغل أو توقف الإشارة.

في الـ SoC (System on Chip)، كل clock بيعمل نفس الكلام:
- **مفتاح الاختيار** = **mux** — يختار من فين ييجي الـ clock (من كذا مصدر).
- **مقبض الصوت** = **divider/PLL** — يحدد الـ frequency.
- **زر التشغيل** = **gate** — يشغل أو يوقف الـ clock.

المشكلة: لو كل clock بيشيل الـ 3 أجزاء دي كـ structs منفصلة، هتحتاج تكتب كود كتير وتكرر نفسك. وبعض الـ clocks بتاخد جزء أو اتنين بس — مش الـ 3.

**الحل = `clk_composite`** — هو زي كبسولة بتضم الـ 3 أجزاء (أو أي جزء منهم) في clock واحد بيتعامل معاه CCF بشكل موحد.

---

### الـ Big Picture

```
                  ┌─────────────────────────────────┐
                  │         clk_composite            │
                  │                                  │
  [parent_A] ──┐  │  ┌──────┐   ┌────────┐  ┌──── ┐ │
  [parent_B] ──┼──┼─►│ mux  ├──►│divider ├─►│gate ├─┼──► output clock
  [parent_C] ──┘  │  └──────┘   └────────┘  └──── ┘ │
                  │  mux_hw     rate_hw      gate_hw  │
                  └─────────────────────────────────┘
```

**الـ `clk-composite.c` بيعمل إيه بالظبط؟**

بيأخذ ثلاث أجزاء مستقلة (mux + rate/divider + gate) — كل جزء عنده `clk_hw` و `clk_ops` خاصين بيه — وبيدمجهم في clock واحد بيقدم **واجهة موحدة** لـ CCF.

لما CCF بيسأل الـ composite clock "ايه الـ parent بتاعك؟" — الـ composite بيحول السؤال للـ `mux_hw`. لما بيسأل "ايه الـ rate؟" — بيحول للـ `rate_hw`. لما بيسأل "هو شغال؟" — بيحول للـ `gate_hw`.

ده pattern اسمه **Delegation** — الـ composite مش بينفذ حاجة بنفسه، هو بس **وسيط ذكي** يوجه كل سؤال للجزء الصح.

---

### ليه ده مهم؟

لو اشتغلت على أي SoC — Qualcomm Snapdragon، Rockchip، NVIDIA Tegra، Allwinner، إلخ — هتلاقي الـ clock tree بتاعه مليان clock كل واحد فيهم عنده mux + divider + gate. بدل ما كل driver يكتب الكود ده من الأول، بيستخدم `clk_hw_register_composite()` ويبعت الأجزاء — والـ framework يتكفل بالباقي.

---

### الـ determine_rate ذكي

أذكى جزء في الملف هو `clk_composite_determine_rate()`. لما حد بيطلب frequency معينة:

1. بيلف على كل الـ parents المتاحة للـ mux.
2. لكل parent بيسأل الـ rate_hw: "لو أنا شغال من ده، أقرب rate تقدر تديهولي إيه؟"
3. بيختار الـ parent اللي بيدي أقرب rate للمطلوب.
4. لو flag الـ `CLK_SET_RATE_NO_REPARENT` موجود — مش هيغير الـ parent خالص، هيحسب بس من الـ parent الحالي.

---

### سيناريو حقيقي

تخيل شريحة NVIDIA Tegra. فيها clock خاص بـ display controller:
- بيختار من كذا مصدر (PLL_D, PLL_D2, إلخ) عشان يدي الـ pixel clock الصح.
- بعد الاختيار، بيقسم الـ frequency لحاجة مناسبة للشاشة.
- وعنده gate يقفله لو الشاشة اتقفلت توفيرًا للطاقة.

الـ Tegra driver بيعمل `clk_hw_register_composite()` ويبعت:
- `mux_hw` = الـ mux struct الخاص بـ Tegra
- `rate_hw` = الـ divider struct
- `gate_hw` = الـ gate struct

وCCF بيتعامل مع الـ display clock ده كـ clock عادي يقدر يتحكم فيه.

---

### الـ devm Variant

الملف كمان بيوفر `devm_clk_hw_register_composite_pdata()` — ده بيضيف الـ clock على الـ device resource management، يعني لما الـ device اتفصلت، الـ clock بيتشاله أوتوماتيك من غير ما الـ driver يعمل `free` يدوي.

---

### الـ set_rate_and_parent

لما محتاج تغير الـ rate والـ parent في نفس الوقت، الكود بيكون ذكي في الترتيب:
- لو الـ rate الحالي **أكبر** من المطلوب → اخفض الـ rate الأول *ثم* غير الـ parent (عشان متعديش على الـ device بـ rate عالي وهو على parent جديد).
- لو الـ rate الحالي **أقل أو مساوي** → غير الـ parent الأول *ثم* اضبط الـ rate.

ده لحماية الـ hardware من تجاوز الحدود القصوى أثناء التبديل.

---

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `drivers/clk/clk-composite.c` | **الملف الحالي** — تنفيذ الـ composite clock |
| `drivers/clk/clk.c` | قلب الـ CCF — الـ core framework |
| `drivers/clk/clk-mux.c` | تنفيذ الـ mux clock (يُستخدم كـ mux_hw) |
| `drivers/clk/clk-divider.c` | تنفيذ الـ divider clock (يُستخدم كـ rate_hw) |
| `drivers/clk/clk-gate.c` | تنفيذ الـ gate clock (يُستخدم كـ gate_hw) |
| `drivers/clk/clk-fixed-rate.c` | الـ fixed-rate clock |
| `drivers/clk/clk-fixed-factor.c` | الـ fixed-factor divider/multiplier |
| `include/linux/clk-provider.h` | **الهيدر الرئيسي** — تعريف `struct clk_composite`، `struct clk_ops`، `struct clk_hw` |
| `include/linux/clk.h` | واجهة المستهلكين (consumers) — `clk_get_rate()`، `clk_enable()` |

### ملفات الـ HW Drivers اللي بتستخدم الـ composite

- `drivers/clk/tegra/` — NVIDIA Tegra
- `drivers/clk/rockchip/` — Rockchip SoCs
- `drivers/clk/samsung/` — Samsung Exynos
- `drivers/clk/imx/` — NXP i.MX
- `drivers/clk/qcom/` — Qualcomm

---

### ملخص سريع

**الـ `clk-composite.c`** = لبنة بناء أساسية في CCF بتسمح لأي SoC driver يدمج الـ mux + rate + gate في clock واحد موحد، بدل تكرار نفس الـ boilerplate في كل driver. هو مش بينفذ حاجة بنفسه — هو وسيط بيحول كل عملية للجزء المسؤول عنها بالضبط.
## Phase 2: شرح الـ Common Clock Framework (CCF) و الـ clk-composite

---

### المشكلة — ليه الـ CCF موجود أصلاً؟

في أي SoC حديث — خد مثلاً i.MX8 أو Rockchip RK3588 — فيه مئات الـ clocks. كل clock عبارة عن إشارة تردد بتغذّي peripheral معينة: USB، UART، GPU، DDR، إلخ. قبل الـ CCF، كل vendor كان بيعمل implementation خاص بيه من الصفر. النتيجة؟

- كود متكرر بالمئات في كل driver.
- مفيش policy موحدة لـ enable/disable أو rate negotiation.
- consumer مش عارف يطلب rate من hardware بشكل portable.
- power management بيتعمل manually وبيتكسر بسهولة.

**الـ CCF** اتضاف في kernel 3.4 عشان يحل ده بـ abstraction موحدة لكل الـ clocks على كل الـ platforms.

---

### الحل — الـ CCF بيعمل إيه؟

الـ CCF بيعمل حاجتين أساسيتين:

1. **Generic clock tree**: شجرة hierarchical من الـ clocks — كل clock ليه parent وممكن يبقى parent لـ clock تاني. الـ rate بتتحسب من فوق لتحت.
2. **Standard ops interface**: كل clock hardware بيعمل implement لـ `struct clk_ops` — مجموعة function pointers موحدة. الـ framework بيكلم الـ ops دي من غير ما يعرف التفاصيل الـ hardware-specific.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Consumer Drivers                             │
  │  (USB driver, UART driver, GPU driver, ...)                     │
  │           clk_enable() / clk_set_rate() / clk_get_rate()        │
  └─────────────────────┬───────────────────────────────────────────┘
                        │   Consumer API  (include/linux/clk.h)
  ┌─────────────────────▼───────────────────────────────────────────┐
  │              Common Clock Framework (CCF)                       │
  │         drivers/clk/clk.c  +  include/linux/clk-provider.h     │
  │                                                                 │
  │   - Clock tree management (parent/child relationships)          │
  │   - Reference counting (enable/disable)                         │
  │   - Rate propagation & caching                                  │
  │   - Debugfs  (/sys/kernel/debug/clk/)                           │
  └────┬──────────────┬──────────────┬────────────────┬────────────┘
       │              │              │                │
       ▼              ▼              ▼                ▼
  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌─────────────────┐
  │clk_fixed│  │ clk_mux  │  │clk_divider│  │  clk_composite  │  ← ملف دراستنا
  │  _rate  │  │          │  │           │  │  (mux+rate+gate) │
  └─────────┘  └──────────┘  └───────────┘  └─────────────────┘
       │              │              │                │
       └──────────────┴──────────────┴────────────────┘
                             │
              Provider API  (include/linux/clk-provider.h)
                             │
  ┌──────────────────────────▼──────────────────────────────────────┐
  │              Platform Clock Drivers                             │
  │  drivers/clk/imx/   drivers/clk/rockchip/   drivers/clk/qcom/  │
  │  (register clk_hw instances, describe the hardware clock tree)  │
  └─────────────────────────────────────────────────────────────────┘
                             │
  ┌──────────────────────────▼──────────────────────────────────────┐
  │                      Real Hardware                              │
  │              SoC Clock Controller (PLL, CCM, CRU)              │
  └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions في الـ CCF

قبل ما نتكلم عن الـ composite، لازم تفهم الـ building blocks الأساسية:

#### `struct clk_hw` — نقطة الربط بين الـ framework والـ hardware

```c
struct clk_hw {
    struct clk_core *core;   /* pointer للـ CCF internal state */
    struct clk *clk;         /* الـ consumer-facing handle */
    const struct clk_init_data *init; /* بيتفرغ بعد registration */
};
```

الـ `clk_hw` مش بيوصف الـ hardware مباشرةً — هو بيتضمّن جوا الـ driver-specific struct. مثلاً:

```c
struct clk_divider {
    struct clk_hw hw;   /* embedded — container_of بيوصّلك للـ divider */
    void __iomem *reg;
    u8 shift;
    u8 width;
    spinlock_t *lock;
};
```

#### `struct clk_ops` — الـ vtable بتاع كل clock

```c
struct clk_ops {
    /* Gate ops */
    int   (*enable)(struct clk_hw *hw);
    void  (*disable)(struct clk_hw *hw);
    int   (*is_enabled)(struct clk_hw *hw);

    /* Rate ops */
    unsigned long (*recalc_rate)(struct clk_hw *hw, unsigned long parent_rate);
    long          (*round_rate)(struct clk_hw *hw, unsigned long rate, unsigned long *prate);
    int           (*determine_rate)(struct clk_hw *hw, struct clk_rate_request *req);
    int           (*set_rate)(struct clk_hw *hw, unsigned long rate, unsigned long parent_rate);

    /* Mux ops */
    int  (*set_parent)(struct clk_hw *hw, u8 index);
    u8   (*get_parent)(struct clk_hw *hw);

    /* Combined */
    int  (*set_rate_and_parent)(struct clk_hw *hw, unsigned long rate,
                                unsigned long parent_rate, u8 index);
    /* ... */
};
```

---

### المشكلة الثانية — ليه الـ clk-composite موجود؟

في الـ SoC الواحد، كتير من الـ clocks مش "simple" — هي فعلياً **ثلاث وظائف في وحدة واحدة**:

```
              ┌───────┐
PLL_A ───────►│       │       ┌──────────┐     ┌──────┐
              │  MUX  ├──────►│ DIVIDER  ├────►│ GATE ├──► peripheral_clk
PLL_B ───────►│       │       └──────────┘     └──────┘
              └───────┘
      اختار المصدر    اقسم التردد         افتح/قفل الـ clock
```

**الـ Mux** بيختار المصدر (parent).
**الـ Rate/Divider** بيضبط التردد.
**الـ Gate** بيفتح أو بيقفل الـ clock.

الـ CCF عنده implementations جاهزة لكل واحد منهم (`clk_mux`, `clk_divider`, `clk_gate`). المشكلة إن كل peripheral_clk في الـ SoC لو محتاجتلو الثلاثة مع بعض، هتحتاج register 3 clocks منفصلين وتربطهم. ده **code duplication** ومش practical لما عندك 200 clock في الـ SoC.

**الحل**: `clk_composite` — يجمع الثلاثة في clock واحد يبقى من وجهة نظر الـ consumer وحدة واحدة.

---

### الـ Analogy الحقيقي — محطة كهرباء متكاملة

تخيل محطة توزيع كهرباء في مدينة:

| الجزء في الواقع | المقابل في الـ composite |
|---|---|
| **مصادر الطاقة** (طاقة شمسية، بترول، شبكة وطنية) | **الـ parent clocks** (PLL_A, PLL_B, OSC) |
| **لوحة التوزيع** — بتختار المصدر | **الـ mux** — `set_parent` / `get_parent` |
| **المحول (Transformer)** — بيغير الجهد | **الـ divider/rate** — `set_rate` / `recalc_rate` |
| **القاطع الرئيسي (Circuit Breaker)** — فتح/قفل | **الـ gate** — `enable` / `disable` |
| **مهندس التشغيل** بيتحكم في الثلاثة | **الـ clk_composite** — بيفوّض لكل جزء |
| **الشبكة الوطنية** تعرف بس "أرسل طاقة للمدينة" | **الـ CCF** — بيتكلم مع الـ composite كـ unit واحدة |

الـ consumer (driver) بيقول "عايز 100 MHz" — الـ composite بيقرر: أنا هعدّل الـ divider ولو محتاج هغير المصدر كمان، وبعدين هفتح الـ gate. المحطة الكاملة بتشتغل كـ black box واحدة.

---

### الـ `struct clk_composite` — التفاصيل

```c
struct clk_composite {
    struct clk_hw   hw;         /* embedded handle — ده اللي بيتسجل في الـ CCF */
    struct clk_ops  ops;        /* الـ ops بتتبني dynamically أثناء الـ registration */

    struct clk_hw   *mux_hw;    /* pointer للـ mux sub-clock */
    struct clk_hw   *rate_hw;   /* pointer للـ rate/divider sub-clock */
    struct clk_hw   *gate_hw;   /* pointer للـ gate sub-clock */

    const struct clk_ops *mux_ops;   /* الـ ops بتاعت الـ mux */
    const struct clk_ops *rate_ops;  /* الـ ops بتاعت الـ rate */
    const struct clk_ops *gate_ops;  /* الـ ops بتاعت الـ gate */
};
```

```
    Memory Layout:
    ┌──────────────────────────────────────────┐
    │  struct clk_composite                    │
    │  ┌────────────────┐                      │
    │  │  struct clk_hw │◄─── CCF يشوف بس ده  │
    │  │    hw          │                      │
    │  └────────────────┘                      │
    │  struct clk_ops ops  ◄── بتتملي dynamically │
    │                                          │
    │  *mux_hw  ──────────────────────────────►│ struct clk_mux { clk_hw; reg; ... }
    │  *rate_hw ──────────────────────────────►│ struct clk_divider { clk_hw; reg; ... }
    │  *gate_hw ──────────────────────────────►│ struct clk_gate { clk_hw; reg; bit; ... }
    │                                          │
    │  *mux_ops  ─► clk_mux_ops               │
    │  *rate_ops ─► clk_divider_ops           │
    │  *gate_ops ─► clk_gate_ops              │
    └──────────────────────────────────────────┘
```

---

### الـ Dynamic Ops Building — القلب بتاع الـ composite

لما بتعمل register لـ composite، الـ `__clk_hw_register_composite` مش بتستخدم fixed ops struct — هي بتبني الـ `clk_composite_ops` dynamically بناءً على أيه الـ sub-clocks اللي اتديتلها:

```c
/* من drivers/clk/clk-composite.c */
clk_composite_ops = &composite->ops;  /* ops موجودة جوا الـ composite struct نفسها */

if (mux_hw && mux_ops) {
    /* لو فيه mux، wire الـ get/set_parent */
    composite->mux_hw = mux_hw;
    composite->mux_ops = mux_ops;
    clk_composite_ops->get_parent = clk_composite_get_parent;
    if (mux_ops->set_parent)
        clk_composite_ops->set_parent = clk_composite_set_parent;
    if (mux_ops->determine_rate)
        clk_composite_ops->determine_rate = clk_composite_determine_rate;
}

if (rate_hw && rate_ops) {
    /* لو فيه rate hw، wire الـ recalc/round/set_rate */
    clk_composite_ops->recalc_rate = clk_composite_recalc_rate;
    if (rate_ops->determine_rate)
        clk_composite_ops->determine_rate = clk_composite_determine_rate;
    else if (rate_ops->round_rate)
        clk_composite_ops->round_rate = clk_composite_round_rate;
    if (rate_ops->set_rate)
        clk_composite_ops->set_rate = clk_composite_set_rate;
    composite->rate_hw = rate_hw;
    composite->rate_ops = rate_ops;
}

if (mux_hw && mux_ops && rate_hw && rate_ops) {
    /* لو الاتنين موجودين، ممكن نعمل atomic set_rate_and_parent */
    if (mux_ops->set_parent && rate_ops->set_rate)
        clk_composite_ops->set_rate_and_parent = clk_composite_set_rate_and_parent;
}

if (gate_hw && gate_ops) {
    /* wire الـ enable/disable/is_enabled */
    composite->gate_hw = gate_hw;
    composite->gate_ops = gate_ops;
    clk_composite_ops->is_enabled = clk_composite_is_enabled;
    clk_composite_ops->enable = clk_composite_enable;
    clk_composite_ops->disable = clk_composite_disable;
}
```

الفكرة الجوهرية هنا: الـ `struct clk_ops` بتاع الـ composite مش ثابتة في الـ compile time — هي بتتحدد في الـ runtime بناءً على الـ sub-clocks المتاحة.

---

### الـ Delegation Pattern — كيف بتحصل الـ Forwarding

كل function في الـ composite بتعمل نفس الحاجة: تاخد الـ `clk_hw` بتاع الـ composite، تعمل `container_of` للـ `clk_composite`، تجيب الـ sub-hw المناسب، تعمل `__clk_hw_set_clk` عشان تربطه بالـ clock الأصلي، وتعدّي الـ call.

```c
/* مثال: clk_composite_get_parent */
static u8 clk_composite_get_parent(struct clk_hw *hw)
{
    /* 1. من الـ clk_hw وصّلنا للـ clk_composite كامل */
    struct clk_composite *composite = to_clk_composite(hw);

    const struct clk_ops *mux_ops = composite->mux_ops;
    struct clk_hw *mux_hw = composite->mux_hw;

    /* 2. ربط الـ mux_hw بنفس الـ clk بتاع الـ composite */
    __clk_hw_set_clk(mux_hw, hw);

    /* 3. فوّض للـ mux الحقيقي */
    return mux_ops->get_parent(mux_hw);
}
```

الـ `__clk_hw_set_clk(mux_hw, hw)` مهمة: بتعمل `mux_hw->clk = hw->clk` عشان الـ sub-hw يبقى متربط بنفس الـ `struct clk` instance. لو متعملتش ده، الـ sub-hw هيشاور على clock غلط في الشجرة.

---

### الـ `determine_rate` — الجزء الأذكى

الـ `clk_composite_determine_rate` هي أكثر function معقدة في الملف. هدفها: إيه أقرب rate ممكن تتحقق، ومن أنهي parent؟

```
  طلب: "عايز 100 MHz"
         │
         ▼
  فيه mux وrate معاً؟
         │
    ┌────┴─────┐
   نعم         لأ ─► فوّض لـ rate فقط أو mux فقط
    │
    ▼
  CLK_SET_RATE_NO_REPARENT set؟
    │
  ┌─┴──┐
 نعم    لأ ─► loop على كل الـ parents
  │
  ▼
استخدم الـ parent الحالي فقط
وجرّب تضبط الـ rate عليه
```

في الـ loop على الـ parents:

```c
for (i = 0; i < clk_hw_get_num_parents(mux_hw); i++) {
    parent = clk_hw_get_parent_by_index(mux_hw, i);
    if (!parent) continue;

    /* جرّب مع الـ parent ده — إيه أقرب rate ممكنة؟ */
    clk_hw_forward_rate_request(hw, req, parent, &tmp_req, req->rate);
    ret = clk_composite_determine_rate_for_parent(rate_hw, &tmp_req,
                                                  parent, rate_ops);
    if (ret) continue;

    /* احسب الفرق عن الـ target */
    rate_diff = abs(req->rate - tmp_req.rate);

    /* اتذكر أحسن نتيجة */
    if (best_rate_diff > rate_diff) {
        req->best_parent_hw = parent;
        req->best_parent_rate = tmp_req.best_parent_rate;
        best_rate_diff = rate_diff;
        best_rate = tmp_req.rate;
    }

    /* لو exact match، وقّف على طول */
    if (!rate_diff) return 0;
}
```

النتيجة: الـ framework بيعرف مين الـ parent الأمثل وإيه الـ rate الأقرب قبل ما يبدأ يعمل أي hardware changes.

---

### الـ `set_rate_and_parent` — الأمان في التسلسل

لما محتاج تغير الـ parent والـ rate في نفس الوقت، فيه مشكلة: لو غيّرت الـ parent الأول وبعدين الـ rate، ممكن يكون في لحظة معينة الـ clock بيدي rate أعلى من المطلوب ويضر بالـ peripheral. الحل؟

```c
static int clk_composite_set_rate_and_parent(struct clk_hw *hw,
                                             unsigned long rate,
                                             unsigned long parent_rate,
                                             u8 index)
{
    /* احسب الـ rate الحالية */
    temp_rate = rate_ops->recalc_rate(rate_hw, parent_rate);

    if (temp_rate > rate) {
        /* الـ rate الحالية أعلى من المطلوبة
         * اخفّض الـ rate الأول قبل ما تغير الـ parent
         * عشان متعداش الـ target في أي لحظة */
        rate_ops->set_rate(rate_hw, rate, parent_rate);
        mux_ops->set_parent(mux_hw, index);
    } else {
        /* الـ rate الحالية أقل — غيّر الـ parent الأول
         * وبعدين اضبط الـ rate */
        mux_ops->set_parent(mux_hw, index);
        rate_ops->set_rate(rate_hw, rate, parent_rate);
    }

    return 0;
}
```

ده "glitch-minimizing" — بيضمن إن الـ clock مش هيعدّي الـ target rate في أي نقطة أثناء التغيير.

---

### الـ devm — الـ Managed Registration

الـ `devm_clk_hw_register_composite_pdata` بتستخدم الـ **devres** subsystem. الـ devres بيخزّن pointer للـ resource ويربطه بالـ `struct device`. لما الـ device بتتفصل، الـ kernel بيكلم `devm_clk_hw_release_composite` أوتوماتيك.

```c
static void devm_clk_hw_release_composite(struct device *dev, void *res)
{
    /* res هنا هو الـ ptr اللي اتخزن في الـ devres */
    clk_hw_unregister_composite(*(struct clk_hw **)res);
}
```

```
  Device lifecycle:
  probe() ──► devm_clk_hw_register_composite_pdata()
                   │
                   └─► devres_alloc() + devres_add()
                            ↕ (linked to device)
  remove() ──► devres_release_all()
                   │
                   └─► devm_clk_hw_release_composite()
                            └─► clk_hw_unregister_composite()
                                     └─► clk_hw_unregister() + kfree()
```

---

### ملخص — إيه اللي بيملكه الـ CCF وإيه اللي بيفوّضه

| المسؤولية | الـ CCF يملكها | الـ clk_composite يفوّضها للـ sub-clocks |
|---|---|---|
| clock tree traversal | CCF | — |
| reference counting | CCF | — |
| rate caching | CCF | — |
| debugfs | CCF | — |
| parent selection logic | `clk_composite_determine_rate` | `rate_ops->determine_rate` أو `round_rate` |
| hardware mux register | — | `mux_ops->set_parent` / `get_parent` |
| hardware divider register | — | `rate_ops->set_rate` / `recalc_rate` |
| hardware gate register | — | `gate_ops->enable` / `disable` |
| sequencing (rate vs parent order) | `clk_composite_set_rate_and_parent` | — |
| memory management | `kzalloc` / `kfree` في الـ composite | — |

---

### مثال حقيقي — Rockchip CRU

الـ Rockchip clock drivers في `drivers/clk/rockchip/` بتستخدم الـ `clk_composite` بكثافة. كل peripheral clock في RK3588 بيتعمل register بـ `clk_hw_register_composite` مع:

- **mux_hw**: instance من `clk_mux` بيشاور على CRU_CLKSELn register.
- **rate_hw**: instance من `clk_divider` بيقسّم التردد.
- **gate_hw**: instance من `clk_gate` بيتحكم في CRU_GATE register.

الـ consumer driver بيشوف clock واحد اسمه مثلاً `"clk_uart2"` ويعمل `clk_set_rate(clk, 115200*16)` — والـ composite بيتكفل بكل التفاصيل.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### CLK Flags (من `include/linux/clk-provider.h`)

| Flag | Value | المعنى |
|------|-------|---------|
| `CLK_SET_RATE_GATE` | `BIT(0)` | لازم الـ clock يتعطّل وقت تغيير الـ rate |
| `CLK_SET_RATE_PARENT` | `BIT(2)` | ارفع طلب تغيير الـ rate للـ parent |
| `CLK_SET_RATE_NO_REPARENT` | `BIT(7)` | متعملش reparent وقت تغيير الـ rate — استخدم الـ parent الحالي بس |
| `CLK_SET_RATE_UNGATE` | `BIT(10)` | الـ clock محتاج يشتغل عشان يعرف يتغير الـ rate |

> **ملاحظة مهمة:** الـ flag `CLK_SET_RATE_NO_REPARENT` له تأثير مباشر داخل `clk_composite_determine_rate()` — لو موجود يتجاهل loop الـ parents ويشتغل على الـ parent الحالي بس.

---

### الـ Structs المهمة

#### 1. `struct clk_hw`

**الغرض:** الـ handle اللي بيربط الـ common clock framework بالـ hardware-specific structure. كل driver بيحط `clk_hw` جوه الـ struct الخاص بيه.

| Field | Type | الوظيفة |
|-------|------|---------|
| `core` | `struct clk_core *` | pointer للـ internal core اللي الـ framework بيديره |
| `clk` | `struct clk *` | الـ per-user handle اللي بيتبعت للـ consumers |
| `init` | `const struct clk_init_data *` | init data — بيتحول لـ NULL بعد الـ register |

---

#### 2. `struct clk_init_data`

**الغرض:** بيانات التهيئة اللي بتتحرك بين الـ driver والـ framework وقت الـ registration.

| Field | Type | الوظيفة |
|-------|------|---------|
| `name` | `const char *` | اسم الـ clock في النظام |
| `ops` | `const struct clk_ops *` | الـ callbacks الخاصة بالـ clock دا |
| `parent_names` | `const char * const *` | أسماء الـ parents (الطريقة القديمة) |
| `parent_data` | `const struct clk_parent_data *` | بيانات الـ parents (الطريقة الجديدة) |
| `parent_hws` | `const struct clk_hw **` | pointers مباشرة للـ parents (أسرع) |
| `num_parents` | `u8` | عدد الـ parents المحتملين |
| `flags` | `unsigned long` | الـ CLK_* flags اللي فوق |

> بس **واحد بس** من الـ parent_names/parent_data/parent_hws بيتستخدم في نفس الوقت.

---

#### 3. `struct clk_ops`

**الغرض:** جدول الـ callbacks اللي الـ framework بيستدعيه على كل clock. الـ composite بيبني نسخته الخاصة منه وقت الـ registration.

| Callback | الوظيفة في الـ Composite |
|----------|--------------------------|
| `get_parent` | يفوّض لـ `mux_ops->get_parent` |
| `set_parent` | يفوّض لـ `mux_ops->set_parent` |
| `determine_rate` | يعمل loop على الـ parents أو يفوّض لـ rate/mux |
| `round_rate` | يفوّض لـ `rate_ops->round_rate` |
| `recalc_rate` | يفوّض لـ `rate_ops->recalc_rate` |
| `set_rate` | يفوّض لـ `rate_ops->set_rate` |
| `set_rate_and_parent` | يدي الـ rate و index مع بعض — atomic |
| `is_enabled` | يفوّض لـ `gate_ops->is_enabled` |
| `enable` | يفوّض لـ `gate_ops->enable` |
| `disable` | يفوّض لـ `gate_ops->disable` |

---

#### 4. `struct clk_parent_data`

**الغرض:** وصف الـ parent بطريقة flexible — بيدعم الـ DT fw_name والاسم الصريح والـ hw pointer.

| Field | Type | الوظيفة |
|-------|------|---------|
| `hw` | `const struct clk_hw *` | pointer مباشر للـ parent (لو internal) |
| `fw_name` | `const char *` | اسم الـ parent في الـ Device Tree |
| `name` | `const char *` | الاسم العالمي كـ fallback |
| `index` | `int` | index محلي عند الـ provider |

---

#### 5. `struct clk_rate_request`

**الغرض:** وعاء بيتنقل بين الـ framework والـ driver وقت التفاوض على الـ rate المناسب.

| Field | Type | الوظيفة |
|-------|------|---------|
| `core` | `struct clk_core *` | الـ clock المتأثر بالطلب |
| `rate` | `unsigned long` | الـ rate المطلوب — بيتعدّل وقت التفاوض |
| `min_rate` | `unsigned long` | الحد الأدنى المسموح بيه |
| `max_rate` | `unsigned long` | الحد الأقصى المسموح بيه |
| `best_parent_rate` | `unsigned long` | أفضل rate قادر يديه الـ parent |
| `best_parent_hw` | `struct clk_hw *` | أفضل parent وصلنا ليه |

---

#### 6. `struct clk_composite` ← الـ struct الأساسي في الملف

**الغرض:** clock مُركَّب (aggregate) بيجمع جوّه ثلاث sub-clocks مستقلين: mux + rate + gate. الـ composite بيبني `clk_ops` خاص بيه ديناميكيًا بناءً على أيه sub-clocks اتحطت.

| Field | Type | الوظيفة |
|-------|------|---------|
| `hw` | `struct clk_hw` | الـ handle الخاص بالـ composite نفسه في الـ framework |
| `ops` | `struct clk_ops` | الـ ops table اللي بيتبني وقت الـ registration (mutable) |
| `mux_hw` | `struct clk_hw *` | pointer لـ sub-clock الـ mux (اختيار الـ parent) |
| `rate_hw` | `struct clk_hw *` | pointer لـ sub-clock الـ rate (divider/PLL) |
| `gate_hw` | `struct clk_hw *` | pointer لـ sub-clock الـ gate (تشغيل/إيقاف) |
| `mux_ops` | `const struct clk_ops *` | الـ ops الخاصة بالـ mux sub-clock |
| `rate_ops` | `const struct clk_ops *` | الـ ops الخاصة بالـ rate sub-clock |
| `gate_ops` | `const struct clk_ops *` | الـ ops الخاصة بالـ gate sub-clock |

> الـ `to_clk_composite(_hw)` macro بيعمل `container_of` من `clk_hw` للـ `clk_composite`.

---

### رسم علاقات الـ Structs

```
                    ┌──────────────────────────────────────────────┐
                    │           struct clk_composite               │
                    │                                              │
                    │  ┌─────────────┐   ┌──────────────────────┐ │
                    │  │ struct clk_hw│   │  struct clk_ops ops  │ │
                    │  │    hw       │   │  (built dynamically) │ │
                    │  │  .core ─────┼───┼──► struct clk_core   │ │
                    │  │  .clk  ─────┼───┼──► struct clk        │ │
                    │  │  .init ─────┼───┼──► clk_init_data     │ │
                    │  └─────────────┘   └──────────────────────┘ │
                    │                                              │
                    │  *mux_hw ──────────────► struct clk_hw (mux)│
                    │  *rate_hw ─────────────► struct clk_hw (div)│
                    │  *gate_hw ─────────────► struct clk_hw (gate│
                    │                                              │
                    │  *mux_ops ─────────────► struct clk_ops     │
                    │  *rate_ops ────────────► struct clk_ops     │
                    │  *gate_ops ────────────► struct clk_ops     │
                    └──────────────────────────────────────────────┘

  struct clk_hw (mux/rate/gate)        struct clk_init_data
  ┌─────────────────────┐              ┌─────────────────────────┐
  │ .core ──────────────┼──────────►   │ .name                   │
  │ .clk (shared) ◄─────┼──shared──    │ .ops                    │
  │ .init = NULL        │  با composite│ .parent_names /         │
  └─────────────────────┘              │   parent_data /         │
                                       │   parent_hws            │
                                       │ .num_parents            │
                                       │ .flags                  │
                                       └─────────────────────────┘

  struct clk_rate_request
  ┌──────────────────────────────────┐
  │ .core ───► struct clk_core       │
  │ .rate          (requested)       │
  │ .min_rate / .max_rate            │
  │ .best_parent_rate                │
  │ .best_parent_hw ──► clk_hw       │
  └──────────────────────────────────┘
```

---

### رسم دورة حياة الـ Composite Clock

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  CREATION                                                       │
  │                                                                 │
  │  driver_probe()                                                 │
  │    │                                                            │
  │    ├─► allocate mux_hw  (e.g., clk_hw_register_mux)            │
  │    ├─► allocate rate_hw (e.g., clk_hw_register_divider)        │
  │    └─► allocate gate_hw (e.g., clk_hw_register_gate)           │
  └─────────────────────────────────────────────────────────────────┘
                      │
                      ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  REGISTRATION                                                   │
  │                                                                 │
  │  clk_hw_register_composite() / clk_hw_register_composite_pdata │
  │    │                                                            │
  │    └─► __clk_hw_register_composite()                           │
  │           │                                                     │
  │           ├─ kzalloc(struct clk_composite)                     │
  │           ├─ fill clk_init_data                                 │
  │           ├─ build clk_composite_ops table dynamically         │
  │           │    ├─ mux_hw?  → add get/set_parent, determine_rate │
  │           │    ├─ rate_hw? → add recalc, round, determine, set  │
  │           │    │             + set_rate_and_parent if both      │
  │           │    └─ gate_hw? → add is_enabled, enable, disable    │
  │           ├─ clk_hw_register(dev, hw)  ← يسجّل في الـ framework │
  │           └─ share .clk pointer: mux_hw/rate_hw/gate_hw        │
  └─────────────────────────────────────────────────────────────────┘
                      │
                      ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  USAGE (consumer calls clk_set_rate / clk_enable ...)          │
  │                                                                 │
  │  framework calls composite ops                                  │
  │    ├─► clk_composite_determine_rate() → يفاوض مع الـ parents   │
  │    ├─► clk_composite_set_rate_and_parent() → atomic            │
  │    ├─► clk_composite_enable() → gate                           │
  │    └─► clk_composite_recalc_rate() → rate                      │
  └─────────────────────────────────────────────────────────────────┘
                      │
                      ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  TEARDOWN                                                       │
  │                                                                 │
  │  clk_hw_unregister_composite(hw)                               │
  │    ├─ clk_hw_unregister(hw)   ← يشيل من الـ framework          │
  │    └─ kfree(composite)        ← يحرر الـ memory               │
  │                                                                 │
  │  أو تلقائيًا لو استخدمت devm_clk_hw_register_composite_pdata:  │
  │    device_release → devm_clk_hw_release_composite()            │
  └─────────────────────────────────────────────────────────────────┘
```

---

### رسم Call Flow

#### تغيير الـ Rate مع إمكانية Reparent

```
consumer: clk_set_rate(clk, target_rate)
  │
  └─► CCF core: calls composite ops->determine_rate(hw, req)
        │
        └─► clk_composite_determine_rate(hw, req)
              │
              ├─[rate_hw + rate_ops + mux_hw + mux_ops->set_parent]─►
              │   │
              │   ├─[CLK_SET_RATE_NO_REPARENT set?]─►
              │   │    parent = clk_hw_get_parent(mux_hw)
              │   │    clk_hw_forward_rate_request(hw, req, parent, &tmp)
              │   │    clk_composite_determine_rate_for_parent(rate_hw, &tmp, parent, rate_ops)
              │   │      └─► rate_ops->determine_rate(rate_hw, req)
              │   │           OR rate_ops->round_rate(rate_hw, rate, &prate)
              │   │    req->rate = tmp.rate; return 0
              │   │
              │   └─[reparent allowed]─►
              │        for each parent in mux_hw:
              │          clk_hw_forward_rate_request(hw, req, parent, &tmp, rate)
              │          clk_composite_determine_rate_for_parent(rate_hw, &tmp, parent, rate_ops)
              │          track best_rate_diff → update req->best_parent_hw
              │        req->rate = best_rate; return 0
              │
              ├─[only rate_hw + rate_ops->determine_rate]─►
              │   __clk_hw_set_clk(rate_hw, hw)
              │   rate_ops->determine_rate(rate_hw, req)
              │
              └─[only mux_hw + mux_ops->determine_rate]─►
                  __clk_hw_set_clk(mux_hw, hw)
                  mux_ops->determine_rate(mux_hw, req)
```

#### تطبيق الـ Rate بعد الاتفاق (set_rate_and_parent)

```
CCF core: calls ops->set_rate_and_parent(hw, rate, parent_rate, index)
  │
  └─► clk_composite_set_rate_and_parent(hw, rate, parent_rate, index)
        │
        ├─ __clk_hw_set_clk(rate_hw, hw)   ← share clk/core context
        ├─ __clk_hw_set_clk(mux_hw, hw)
        │
        ├─ temp_rate = rate_ops->recalc_rate(rate_hw, parent_rate)
        │
        ├─[temp_rate > rate] → خفّض الـ rate الأول عشان متعلّيش
        │    rate_ops->set_rate(rate_hw, rate, parent_rate)
        │    mux_ops->set_parent(mux_hw, index)
        │
        └─[temp_rate <= rate] → غيّر الـ parent الأول عشان ما تلقطش glitch
             mux_ops->set_parent(mux_hw, index)
             rate_ops->set_rate(rate_hw, rate, parent_rate)
```

#### Enable / Disable

```
consumer: clk_enable(clk)
  └─► CCF: ops->enable(hw)
        └─► clk_composite_enable(hw)
              ├─ composite = to_clk_composite(hw)
              ├─ __clk_hw_set_clk(gate_hw, hw)   ← sync context
              └─ gate_ops->enable(gate_hw)
                    └─► hardware register write (e.g., clk_gate_ops)

consumer: clk_disable(clk)
  └─► CCF: ops->disable(hw)
        └─► clk_composite_disable(hw)
              ├─ __clk_hw_set_clk(gate_hw, hw)
              └─ gate_ops->disable(gate_hw)
```

#### الـ `__clk_hw_set_clk` — ليه مهم؟

```c
static inline void __clk_hw_set_clk(struct clk_hw *dst, struct clk_hw *src)
{
    dst->clk  = src->clk;   // share the per-user handle
    dst->core = src->core;  // share the core context
}
```

الـ sub-clock (mux/rate/gate) مش مسجّل بشكل مستقل في الـ framework — هو بس struct بيشتغل لما الـ composite ييجي يستخدمه. الـ `__clk_hw_set_clk` بيوصّله بـ context الـ composite عشان الـ ops بتاعته تشتغل صح.

---

### بناء الـ `clk_composite_ops` الديناميكي

الـ ops table مش ثابتة — بتتبنى بناءً على أيه sub-clocks اتحطت:

```
  mux_hw + mux_ops provided?
    ├─ mux_ops->get_parent   → ops.get_parent    = clk_composite_get_parent   [مطلوب]
    ├─ mux_ops->set_parent?  → ops.set_parent    = clk_composite_set_parent
    └─ mux_ops->determine_rate? → ops.determine_rate = clk_composite_determine_rate

  rate_hw + rate_ops provided?
    ├─ rate_ops->recalc_rate → ops.recalc_rate   = clk_composite_recalc_rate  [مطلوب]
    ├─ rate_ops->determine_rate? → ops.determine_rate = clk_composite_determine_rate
    ├─ rate_ops->round_rate? → ops.round_rate    = clk_composite_round_rate
    └─ rate_ops->set_rate + (determine_rate OR round_rate)?
                           → ops.set_rate        = clk_composite_set_rate

  mux_hw + rate_hw + mux_ops->set_parent + rate_ops->set_rate?
    └─ ops.set_rate_and_parent = clk_composite_set_rate_and_parent

  gate_hw + gate_ops provided?
    ├─ gate_ops->is_enabled  → ops.is_enabled    = clk_composite_is_enabled   [مطلوب]
    ├─ gate_ops->enable      → ops.enable        = clk_composite_enable       [مطلوب]
    └─ gate_ops->disable     → ops.disable       = clk_composite_disable      [مطلوب]
```

---

### استراتيجية الـ Locking

الـ `clk-composite.c` **مش بيعمل locking مباشر** — الـ locking كله بيتعمل في الـ Common Clock Framework (CCF) core قبل ما تتاستدعي الـ ops callbacks.

| Lock | بيحميه | المكان |
|------|--------|--------|
| `prepare_lock` (mutex) | كل عمليات `prepare/unprepare` | CCF core — `clk_prepare()` |
| `enable_lock` (spinlock) | كل عمليات `enable/disable` | CCF core — `clk_enable()` |
| `clk_rate_lock` (rwsem) | قراءة/كتابة الـ rate tree | CCF core — `clk_set_rate()` |

**ترتيب الـ Locking داخل الـ Composite:**

```
clk_set_rate(clk, rate)           ← consumer
  └─ CCF acquires prepare_lock
     └─ CCF acquires clk_rate_lock (write)
        └─ calls ops->determine_rate   ← no extra lock needed
           └─ calls ops->set_rate_and_parent
              ├─ rate_ops->set_rate    ← hardware write (spinlock inside gate/div driver)
              └─ mux_ops->set_parent  ← hardware write
```

> **قاعدة مهمة:** الـ `enable/disable` بيتم من spinlock context — الـ callbacks دي **لازم** ما تنامش ومش تحجز mutex.
> الـ `prepare/unprepare` بيتم من mutex context — ممكن ينام.

**الـ ordering في `set_rate_and_parent`:**

المنطق بيمنع الـ glitch:
- لو الـ rate الحالي **أكبر** من المطلوب → اخفّض الـ rate الأول **قبل** ما تغير الـ parent (عشان متعلّيش عن الـ target لحظيًا)
- لو الـ rate الحالي **أصغر** → غيّر الـ parent الأول **قبل** الـ rate (عشان الـ parent الجديد يكون جاهز)

---

### الـ devm Variant والـ Resource Management

```
devm_clk_hw_register_composite_pdata(dev, ...)
  │
  ├─ devres_alloc(devm_clk_hw_release_composite, sizeof(clk_hw*), GFP_KERNEL)
  │    └─ allocates a ptr to store the clk_hw pointer
  │
  ├─ __clk_hw_register_composite(...)
  │
  ├─[success] devres_add(dev, ptr)   ← ربط الـ resource بالـ device
  └─[failure]  devres_free(ptr)

  on device_release():
    └─ devm_clk_hw_release_composite(dev, res)
         └─ clk_hw_unregister_composite(*(clk_hw **)res)
               ├─ clk_hw_unregister(hw)
               └─ kfree(composite)
```

الـ `devm_` variant بتضمن إن الـ composite clock يتحرر تلقائيًا لما الـ device يتشال — مفيش حاجة للـ driver يعمل cleanup يدوي.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### Registration Functions

| Function | Export | الغرض |
|---|---|---|
| `__clk_hw_register_composite()` | static | الـ core logic لتسجيل composite clock |
| `clk_hw_register_composite()` | `EXPORT_SYMBOL_GPL` | تسجيل composite باستخدام `parent_names` |
| `clk_hw_register_composite_pdata()` | internal | تسجيل composite باستخدام `clk_parent_data` |
| `clk_register_composite()` | `EXPORT_SYMBOL_GPL` | wrapper قديم يرجع `struct clk *` |
| `clk_register_composite_pdata()` | internal | wrapper قديم مع `pdata` |
| `__devm_clk_hw_register_composite()` | static | devm core logic |
| `devm_clk_hw_register_composite_pdata()` | exported | devm-managed registration |

#### Cleanup Functions

| Function | Export | الغرض |
|---|---|---|
| `clk_unregister_composite()` | exported | unregister + free للـ `struct clk *` API |
| `clk_hw_unregister_composite()` | `EXPORT_SYMBOL_GPL` | unregister + free للـ `clk_hw` API |
| `devm_clk_hw_release_composite()` | static (devres cb) | devres callback للـ auto cleanup |

#### MUX Ops Delegation

| Function | الغرض |
|---|---|
| `clk_composite_get_parent()` | delegate لـ `mux_ops->get_parent` |
| `clk_composite_set_parent()` | delegate لـ `mux_ops->set_parent` |

#### Rate Ops Delegation

| Function | الغرض |
|---|---|
| `clk_composite_recalc_rate()` | delegate لـ `rate_ops->recalc_rate` |
| `clk_composite_determine_rate()` | main rate negotiation مع parent sweeping |
| `clk_composite_determine_rate_for_parent()` | helper — يحسب best rate لـ parent واحد |
| `clk_composite_round_rate()` | delegate لـ `rate_ops->round_rate` |
| `clk_composite_set_rate()` | delegate لـ `rate_ops->set_rate` |
| `clk_composite_set_rate_and_parent()` | atomic: يغير rate + parent بالترتيب الصح |

#### Gate Ops Delegation

| Function | الغرض |
|---|---|
| `clk_composite_is_enabled()` | delegate لـ `gate_ops->is_enabled` |
| `clk_composite_enable()` | delegate لـ `gate_ops->enable` |
| `clk_composite_disable()` | delegate لـ `gate_ops->disable` |

---

### Group 1: MUX Delegation Functions

الـ composite clock بتستخدم pattern ثابت مع كل delegate function:
1. استخرج الـ `clk_composite` من الـ `clk_hw` عبر `to_clk_composite()`.
2. اعمل `__clk_hw_set_clk(sub_hw, hw)` — يعني خلّي الـ sub-hw يشير لنفس `core` و `clk` زي الـ composite.
3. استدعي الـ sub-ops مباشرة.

ده ضروري لأن الـ sub-HW structs مش متسجلين مستقلين في الـ CCF — محتاجين يقفلوا على الـ `clk_core` الصحيح عشان الـ tree traversal يشتغل.

---

#### `clk_composite_get_parent`

```c
static u8 clk_composite_get_parent(struct clk_hw *hw)
```

بتـ delegate الـ call لـ `mux_ops->get_parent` بعد ما تـ sync الـ `mux_hw` مع الـ composite `clk_core`. بترجع index الـ parent الحالي. بتتحال من الـ CCF أثناء clk_core traversal أو عند query الـ tree.

- **`hw`**: الـ `clk_hw` embedded في `struct clk_composite`.
- **Return**: `u8` — index الـ parent الحالي من وجهة نظر الـ mux sub-clock.
- **Key details**: لازم `mux_ops->get_parent` يكون موجود — الـ registration code يتحقق منها. لا locking هنا، الـ CCF بيـ hold الـ `prepare_lock` قبل ما يستدعيها.

---

#### `clk_composite_set_parent`

```c
static int clk_composite_set_parent(struct clk_hw *hw, u8 index)
```

بتـ delegate لـ `mux_ops->set_parent`. بتتسجل فقط لو `mux_ops->set_parent` موجود. الـ CCF بيستدعيها أثناء `clk_set_parent()` أو أثناء rate reparenting.

- **`hw`**: composite clock handle.
- **`index`**: رقم الـ parent المطلوب.
- **Return**: `0` نجاح، قيمة سالبة خطأ — نفس ما يرجعه الـ underlying mux op.
- **Key details**: لازم يتسبقها `clk_composite_determine_rate` عشان الـ CCF يحدد الـ `best_parent_hw`.

---

### Group 2: Rate Delegation Functions

---

#### `clk_composite_recalc_rate`

```c
static unsigned long clk_composite_recalc_rate(struct clk_hw *hw,
                                               unsigned long parent_rate)
```

بتـ sync الـ `rate_hw` وبترجع الـ recalculated rate من الـ rate sub-clock. الـ CCF بيستدعيها لما يحتاج يعرف الـ actual output rate بعد ما يمشي في الـ clock tree.

- **`hw`**: composite clock handle.
- **`parent_rate`**: معدل الـ parent اللي بيجي من الـ CCF.
- **Return**: `unsigned long` — الـ output rate المحسوبة.
- **Key details**: لازم `rate_ops->recalc_rate` موجودة — validated في registration. لا side effects.

---

#### `clk_composite_determine_rate_for_parent`

```c
static int clk_composite_determine_rate_for_parent(struct clk_hw *rate_hw,
                                                   struct clk_rate_request *req,
                                                   struct clk_hw *parent_hw,
                                                   const struct clk_ops *rate_ops)
```

**Helper function** — بتحسب أحسن rate ممكن بالنسبة لـ parent محدد. بتحط `best_parent_hw` و `best_parent_rate` في الـ `req` وبعدين بتجرب `determine_rate` الأول، لو مش موجودة بتستخدم `round_rate`.

```
req->best_parent_hw  = parent_hw
req->best_parent_rate = clk_hw_get_rate(parent_hw)

if rate_ops->determine_rate:
    return rate_ops->determine_rate(rate_hw, req)
else:
    rate = rate_ops->round_rate(rate_hw, req->rate, &req->best_parent_rate)
    if rate < 0: return error
    req->rate = rate
    return 0
```

- **`rate_hw`**: الـ rate sub-clock handle (بعد `__clk_hw_set_clk`).
- **`req`**: الـ rate request struct — بيتعدل in-place.
- **`parent_hw`**: الـ parent المراد تجربته.
- **`rate_ops`**: الـ ops الخاصة بالـ rate sub-clock.
- **Return**: `0` نجاح أو error code.
- **Key details**: بتـ mutate الـ `req` مباشرة. المستدعي (`clk_composite_determine_rate`) بيعمل copy مؤقت `tmp_req` عشان يحمي الأصل.

---

#### `clk_composite_determine_rate`

```c
static int clk_composite_determine_rate(struct clk_hw *hw,
                                        struct clk_rate_request *req)
```

دي أهم function في الملف — بتقرر الـ best achievable rate وأحسن parent. فيها 4 سيناريوهات:

**السيناريو 1**: rate_hw + rate_ops + mux_hw + mux_ops->set_parent كلهم موجودين:
- لو `CLK_SET_RATE_NO_REPARENT` معيّنة: بتجرب الـ parent الحالي بس.
- غير كده: بتـ sweep على كل الـ parents وبتختار اللي أقرب للـ requested rate.

**السيناريو 2**: rate_hw فقط مع `determine_rate`: delegate مباشر.

**السيناريو 3**: mux_hw فقط مع `determine_rate`: delegate للـ mux.

**السيناريو 4**: لا mux ولا rate: error.

```
if rate_hw && mux_hw && mux can reparent:
    if CLK_SET_RATE_NO_REPARENT:
        try current parent only
        return best rate found

    best_rate_diff = ULONG_MAX
    for each parent i in mux_hw:
        clone req → tmp_req
        call clk_composite_determine_rate_for_parent(...)
        diff = |req->rate - tmp_req.rate|
        if diff < best_rate_diff:
            update best_{hw, rate, rate_diff}
        if diff == 0: return early (exact match)

    req->rate = best_rate
    return 0

elif rate_hw && rate_ops->determine_rate:
    delegate to rate sub-clock

elif mux_hw && mux_ops->determine_rate:
    delegate to mux sub-clock

else:
    return -EINVAL
```

- **`hw`**: composite handle.
- **`req`**: الـ rate request — بيتعدل بالـ best result.
- **Return**: `0` نجاح أو error.
- **Key details**: الـ parent sweep هو الجزء الأهم — بيضمن إن الـ composite يقدر يستغل كل parents الـ mux عشان يوصل لأقرب rate للمطلوب. الـ `tmp_req` بيحمي الـ original request من التعديل أثناء التجربة.

---

#### `clk_composite_round_rate`

```c
static long clk_composite_round_rate(struct clk_hw *hw, unsigned long rate,
                                     unsigned long *prate)
```

بتـ delegate لـ `rate_ops->round_rate`. بتُسجَّل فقط لو `determine_rate` مش موجودة لكن `round_rate` موجودة. الـ CCF بيستدعيها أثناء `clk_round_rate()`.

- **`hw`**: composite handle.
- **`rate`**: الـ requested rate.
- **`*prate`**: الـ parent rate — يمكن يتعدل لو الـ round_rate op اقترح parent rate مختلف.
- **Return**: `long` — الـ achievable rate. قيمة سالبة = error.

---

#### `clk_composite_set_rate`

```c
static int clk_composite_set_rate(struct clk_hw *hw, unsigned long rate,
                                  unsigned long parent_rate)
```

بتـ delegate لـ `rate_ops->set_rate`. بتُسجَّل فقط لو `set_rate` موجودة مع `round_rate` أو `determine_rate`. الـ CCF بيستدعيها بعد `clk_composite_determine_rate` وبعد تعديل الـ parent لو لزم.

- **`hw`**: composite handle.
- **`rate`**: الـ target rate.
- **`parent_rate`**: معدل الـ parent الجديد.
- **Return**: `0` نجاح أو error.

---

#### `clk_composite_set_rate_and_parent`

```c
static int clk_composite_set_rate_and_parent(struct clk_hw *hw,
                                             unsigned long rate,
                                             unsigned long parent_rate,
                                             u8 index)
```

بتغير الـ rate والـ parent في نفس الـ atomic operation. الفكرة: عشان نتجنب glitch، لازم نرتب العمليات صح. لو الـ current rate أعلى من الـ target — اخفض الـ rate الأول عشان نتجنب overshoot، وبعدين غير الـ parent. لو العكس — غير الـ parent الأول عشان الـ divider يستفيد من الـ rate الأعلى.

```
current_rate = rate_ops->recalc_rate(rate_hw, parent_rate)

if current_rate > target_rate:
    // اخفض الـ divider أول عشان لا نتعدى الـ budget
    rate_ops->set_rate(rate_hw, rate, parent_rate)
    mux_ops->set_parent(mux_hw, index)
else:
    // غير الـ parent أول عشان نستفيد من الـ rate الأعلى
    mux_ops->set_parent(mux_hw, index)
    rate_ops->set_rate(rate_hw, rate, parent_rate)
```

- **`hw`**: composite handle.
- **`rate`**: الـ target output rate.
- **`parent_rate`**: معدل الـ parent الجديد.
- **`index`**: index الـ parent المطلوب في الـ mux.
- **Return**: دايمًا `0` — الـ underlying ops بتـ handle errors بنفسها.
- **Key details**: ده الـ set_rate_and_parent callback اللي الـ CCF بيستدعيه لما يقدر يغير الاتنين مع بعض. بيتسجل فقط لو `mux_ops->set_parent` و `rate_ops->set_rate` كلاهما موجودان. لا locking داخلي — الـ CCF بيـ hold الـ `prepare_lock`.

---

### Group 3: Gate Delegation Functions

---

#### `clk_composite_is_enabled`

```c
static int clk_composite_is_enabled(struct clk_hw *hw)
```

بتـ delegate لـ `gate_ops->is_enabled`. بتسأل الـ gate sub-clock عن حالته.

- **Return**: `1` enabled, `0` disabled.
- **Key details**: لازم `gate_ops->is_enabled` موجودة — validated في registration.

---

#### `clk_composite_enable`

```c
static int clk_composite_enable(struct clk_hw *hw)
```

بتفتح الـ gate عبر `gate_ops->enable`. الـ CCF بيستدعيها من atomic context (بعد `prepare`). لا يجوز sleep داخلها.

- **Return**: `0` نجاح أو error.

---

#### `clk_composite_disable`

```c
static void clk_composite_disable(struct clk_hw *hw)
```

بتقفل الـ gate عبر `gate_ops->disable`. بتتحال من atomic context.

- **Return**: void.

---

### Group 4: Registration Functions

---

#### `__clk_hw_register_composite` (Core Registration)

```c
static struct clk_hw *__clk_hw_register_composite(
    struct device *dev,
    const char *name,
    const char * const *parent_names,
    const struct clk_parent_data *pdata,
    int num_parents,
    struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
    struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
    struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
    unsigned long flags)
```

ده الـ core function اللي كل الـ registration wrappers بتستدعيه. بيبني الـ composite clock من الصفر:

```
composite = kzalloc(sizeof(*composite))
init.name = name; init.flags = flags
init.parent_names = parent_names OR init.parent_data = pdata
hw = &composite->hw
clk_composite_ops = &composite->ops

// MUX setup
if mux_hw && mux_ops:
    validate mux_ops->get_parent exists
    composite->mux_hw = mux_hw
    composite->mux_ops = mux_ops
    clk_composite_ops->get_parent = clk_composite_get_parent
    if mux_ops->set_parent:   ops->set_parent = ...
    if mux_ops->determine_rate: ops->determine_rate = ...

// Rate setup
if rate_hw && rate_ops:
    validate rate_ops->recalc_rate exists
    ops->recalc_rate = clk_composite_recalc_rate
    if determine_rate:  ops->determine_rate = ...
    elif round_rate:    ops->round_rate = ...
    if set_rate && (determine_rate || round_rate):
        ops->set_rate = clk_composite_set_rate

// set_rate_and_parent
if mux && rate && mux->set_parent && rate->set_rate:
    ops->set_rate_and_parent = clk_composite_set_rate_and_parent

// Gate setup
if gate_hw && gate_ops:
    validate is_enabled + enable + disable all exist
    ops->is_enabled = ...
    ops->enable = ...
    ops->disable = ...

init.ops = clk_composite_ops
composite->hw.init = &init
ret = clk_hw_register(dev, hw)

// After registration: sync sub-hw clk pointers
if mux_hw:  mux_hw->clk = hw->clk
if rate_hw: rate_hw->clk = hw->clk
if gate_hw: gate_hw->clk = hw->clk

return hw
```

- **`dev`**: الـ device owner — يستخدم لـ devres وclock lookup.
- **`name`**: اسم الـ clock في الـ CCF global namespace.
- **`parent_names`**: array of parent name strings — legacy API. يكون `NULL` لو `pdata` مستخدم.
- **`pdata`**: `clk_parent_data` array — الـ modern API اللي بيدعم DT phandles. يكون `NULL` لو `parent_names` مستخدم.
- **`num_parents`**: عدد الـ parents.
- **`mux_hw` / `mux_ops`**: الـ mux sub-clock handle وops. يكون `NULL` لو الـ composite مش محتاج mux.
- **`rate_hw` / `rate_ops`**: الـ rate (divider/PLL) sub-clock. يكون `NULL` لو rate ثابت.
- **`gate_hw` / `gate_ops`**: الـ gate sub-clock. يكون `NULL` لو مفيش gate.
- **`flags`**: CCF flags زي `CLK_SET_RATE_PARENT`, `CLK_SET_RATE_NO_REPARENT`.
- **Return**: `struct clk_hw *` — valid pointer أو `ERR_PTR(-EINVAL)` لو validation فشل أو `ERR_PTR(-ENOMEM)`.
- **Key details**:
  - الـ `composite->ops` هو embedded struct — مش static، يعني كل composite عنده ops table خاصة بيه بيتبنى dynamically.
  - بعد `clk_hw_register()`، الـ `init` struct بيتجاهل (الـ CCF بيـ null الـ `hw->init`).
  - الـ sub-hw clk pointer sync ضروري عشان `__clk_hw_set_clk` تشتغل صح.
  - لازم تكون الـ three sub-clocks (mux, rate, gate) كلها optional — يعني composite ممكن يكون مجرد gate بدون mux أو rate.

---

#### `clk_hw_register_composite`

```c
struct clk_hw *clk_hw_register_composite(struct device *dev, const char *name,
    const char * const *parent_names, int num_parents,
    struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
    struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
    struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
    unsigned long flags)
```

الـ public API الأساسية للـ modern `clk_hw` style. بتستدعي `__clk_hw_register_composite` مع `parent_names` و `NULL` لـ `pdata`.

- **Return**: `struct clk_hw *` أو `ERR_PTR`.
- **EXPORT_SYMBOL_GPL** — متاحة لكل الـ drivers.
- **Who calls it**: SoC clock drivers اللي بتستخدم legacy string parent names.

---

#### `clk_hw_register_composite_pdata`

```c
struct clk_hw *clk_hw_register_composite_pdata(struct device *dev,
    const char *name,
    const struct clk_parent_data *parent_data, int num_parents,
    struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
    struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
    struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
    unsigned long flags)
```

نفس `clk_hw_register_composite` لكن بتستخدم `clk_parent_data` الـ modern API اللي بتدعم DT `fw_name` و direct `clk_hw` pointers.

- **Return**: `struct clk_hw *` أو `ERR_PTR`.
- **Who calls it**: Drivers اللي بتستخدم CCF provider API الجديد مع DT integration.

---

#### `clk_register_composite`

```c
struct clk *clk_register_composite(struct device *dev, const char *name,
    const char * const *parent_names, int num_parents,
    struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
    struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
    struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
    unsigned long flags)
```

الـ legacy wrapper اللي بيرجع `struct clk *` بدل `struct clk_hw *`. بيستدعي `clk_hw_register_composite` وبعدين بيرجع `hw->clk`.

- **Return**: `struct clk *` أو `ERR_PTR`.
- **EXPORT_SYMBOL_GPL**.
- **Key details**: الـ `struct clk *` API قديمة — الـ preferred الجديد هو `clk_hw` API. موجودة للتوافق مع drivers قديمة.

---

#### `clk_register_composite_pdata`

```c
struct clk *clk_register_composite_pdata(struct device *dev, const char *name,
    const struct clk_parent_data *parent_data, int num_parents, ...)
```

نفس السابق لكن مع `parent_data`. مش معملها `EXPORT_SYMBOL_GPL` في الكود الحالي.

---

### Group 5: Cleanup Functions

---

#### `clk_unregister_composite`

```c
void clk_unregister_composite(struct clk *clk)
```

بتـ unregister الـ composite clock المسجل بـ legacy `struct clk *` API. بتجيب الـ `clk_hw` من الـ `clk` عبر `__clk_get_hw()` وبعدين بتحوّل لـ `clk_composite` وبتـ free الـ memory.

- **`clk`**: الـ clock المراد حذفه.
- **Key details**: لو `hw` = `NULL` بترجع silently. ترتيب العمليات: `clk_unregister(clk)` الأول — ده بيحذف من الـ CCF tree — وبعدين `kfree(composite)`. لا يجوز عكس الترتيب.

---

#### `clk_hw_unregister_composite`

```c
void clk_hw_unregister_composite(struct clk_hw *hw)
```

الـ counterpart لـ `clk_hw_register_composite`. بتـ unregister وبتـ free الـ `struct clk_composite`.

- **`hw`**: الـ clock handle.
- **EXPORT_SYMBOL_GPL**.
- **Key details**: الـ sub-HW structs (mux, rate, gate) مش بتتـ free هنا — ده مسؤولية الـ caller. الـ composite بس هو اللي بيتـ free.

---

#### `devm_clk_hw_release_composite`

```c
static void devm_clk_hw_release_composite(struct device *dev, void *res)
```

الـ devres release callback. بتتحال automatically لما الـ device يتـ detach أو يتـ remove. بتـ dereference الـ `res` pointer (اللي هو `struct clk_hw **`) وبتستدعي `clk_hw_unregister_composite`.

- **`dev`**: الـ device — مش مستخدم directly.
- **`res`**: الـ resource pointer اللي `devres_alloc` حجزه.
- **Who calls it**: الـ devres subsystem تلقائيًا.

---

#### `__devm_clk_hw_register_composite`

```c
static struct clk_hw *__devm_clk_hw_register_composite(
    struct device *dev,
    const char *name,
    const char * const *parent_names,
    const struct clk_parent_data *pdata,
    int num_parents, ...)
```

الـ core لـ devm registration. بتحجز `devres` resource أولًا، بتستدعي `__clk_hw_register_composite`، لو نجح بتربط الـ `clk_hw *` بالـ resource ليتم `clk_hw_unregister_composite` تلقائيًا وقت device removal.

```
ptr = devres_alloc(devm_clk_hw_release_composite, sizeof(*ptr))
if !ptr: return ERR_PTR(-ENOMEM)

hw = __clk_hw_register_composite(...)

if !IS_ERR(hw):
    *ptr = hw
    devres_add(dev, ptr)   // register for auto-cleanup
else:
    devres_free(ptr)        // cleanup on failure

return hw
```

- **Return**: `struct clk_hw *` أو `ERR_PTR`.
- **Key details**: الـ `devres_alloc` بيحجز `sizeof(struct clk_hw *)` — يعني pointer لـ pointer. ده pattern standard لكل `devm_*` CLK functions.

---

#### `devm_clk_hw_register_composite_pdata`

```c
struct clk_hw *devm_clk_hw_register_composite_pdata(struct device *dev,
    const char *name,
    const struct clk_parent_data *parent_data, int num_parents,
    struct clk_hw *mux_hw, const struct clk_ops *mux_ops,
    struct clk_hw *rate_hw, const struct clk_ops *rate_ops,
    struct clk_hw *gate_hw, const struct clk_ops *gate_ops,
    unsigned long flags)
```

الـ public devm interface — بتستدعي `__devm_clk_hw_register_composite` مع `parent_data`. الـ clock بيتـ unregister تلقائيًا عند `device_del` / `driver_remove`.

- **Return**: `struct clk_hw *` أو `ERR_PTR`.
- **Who calls it**: Drivers اللي عايزة automatic lifecycle management بدون manual cleanup.

---

### الـ Architecture Pattern — رسم توضيحي

```
  clk_composite (registered in CCF)
  ┌──────────────────────────────────────────┐
  │  struct clk_hw hw                        │
  │  struct clk_ops ops  ← built dynamically │
  │                                          │
  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
  │  │ mux_hw   │  │ rate_hw  │  │gate_hw │ │
  │  │ mux_ops  │  │ rate_ops │  │gate_ops│ │
  │  └──────────┘  └──────────┘  └────────┘ │
  └──────────────────────────────────────────┘
           │              │            │
           ▼              ▼            ▼
    clk_mux        clk_divider    clk_gate
    (not reg.)     (not reg.)     (not reg.)
    ─ get_parent   ─ recalc_rate  ─ is_enabled
    ─ set_parent   ─ round_rate   ─ enable
                   ─ determine_r  ─ disable
                   ─ set_rate
```

الـ sub-clocks مش مسجلين مستقلين في الـ CCF — الـ composite بيـ own الـ registration وبيـ delegate الـ ops ليهم. الـ `__clk_hw_set_clk()` بتـ sync الـ `core` و `clk` pointers عشان الـ CCF locking وـ tree traversal يشتغلوا على الـ composite وليس على الـ sub-clocks.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المدخلات المهمة

الـ **Common Clock Framework (CCF)** بيعمل mount لـ debugfs تحت `/sys/kernel/debug/clk/`.
كل clock composite بيظهر كـ directory باسمه فيه الملفات دي:

| الملف | المحتوى | أمر القراءة |
|-------|---------|------------|
| `clk_rate` | الـ rate الحالي بالـ Hz | `cat /sys/kernel/debug/clk/<name>/clk_rate` |
| `clk_enable_count` | عدد مرات الـ enable | `cat /sys/kernel/debug/clk/<name>/clk_enable_count` |
| `clk_prepare_count` | عدد مرات الـ prepare | `cat /sys/kernel/debug/clk/<name>/clk_prepare_count` |
| `clk_flags` | الـ flags المضبوطة | `cat /sys/kernel/debug/clk/<name>/clk_flags` |
| `clk_parent` | اسم الـ parent الحالي | `cat /sys/kernel/debug/clk/<name>/clk_parent` |
| `clk_possible_parents` | كل الـ parents المتاحة | `cat /sys/kernel/debug/clk/<name>/clk_possible_parents` |
| `clk_duty_cycle` | duty cycle لو بيدعمه | `cat /sys/kernel/debug/clk/<name>/clk_duty_cycle` |

**عشان تشوف شجرة كل الـ clocks:**

```bash
# شجرة كاملة بالـ rates
cat /sys/kernel/debug/clk/clk_summary

# شجرة بالـ hierarchy
cat /sys/kernel/debug/clk/clk_dump
```

**مثال على output من `clk_summary`:**

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 osc24M                             1       1        0    24000000          0     0  50000
   pll-cpu                          1       1        0   816000000          0     0  50000
     cpu-composite                  1       1        0   816000000          0     0  50000
```

الـ `cpu-composite` ده مثال على `clk_composite` — الـ enable_cnt=1 يعني هو enabled وشغال.

---

#### 2. sysfs — المدخلات المهمة

```bash
# لو الـ clock مرتبط بـ device
ls /sys/bus/platform/devices/<device>/clocks/

# قراءة الـ rate من sysfs (لو متاحة)
cat /sys/class/clk/<name>/clk_rate

# الـ consumers اللي بيستخدموا الـ clock
cat /sys/kernel/debug/clk/<composite-name>/clk_enable_count
```

---

#### 3. ftrace — الـ Tracepoints المهمة

الـ **CCF** بيدعم tracepoints جاهزة تحت `clk`:

```bash
# عرض كل الـ tracepoints المتاحة للـ clk subsystem
ls /sys/kernel/debug/tracing/events/clk/

# الـ events الأهم لـ clk_composite:
# clk_set_rate        — لما بيتعمل set_rate
# clk_set_rate_complete
# clk_set_parent      — لما بيتغير الـ parent (mux)
# clk_set_parent_complete
# clk_enable          — عند الـ enable (gate)
# clk_disable         — عند الـ disable
# clk_prepare
# clk_unprepare
```

**تفعيل tracing لـ composite clock معين:**

```bash
# تفعيل كل events الـ clk
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# أو تفعيل event محددة
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_parent/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable

# فلترة على clock بالاسم
echo 'name == "cpu-composite"' > /sys/kernel/debug/tracing/events/clk/clk_set_rate/filter

# تشغيل الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# قراءة النتيجة
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
     kworker/0:1-23    [000]  1234.567: clk_set_rate: cpu-composite 816000000
     kworker/0:1-23    [000]  1234.568: clk_set_parent: cpu-composite 1
     kworker/0:1-23    [000]  1234.569: clk_set_rate_complete: cpu-composite 816000000
```

السطر التاني `clk_set_parent` يعني الـ composite غيّر الـ mux لـ parent index=1 — ده normal لو `CLK_SET_RATE_NO_REPARENT` مش مضبوط.

---

#### 4. printk / dynamic debug

**تفعيل dynamic debug للـ clk subsystem:**

```bash
# تفعيل كل رسائل الـ debug في drivers/clk/
echo "file drivers/clk/* +p" > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل ملف clk-composite.c بالتحديد
echo "file drivers/clk/clk-composite.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وأسماء الـ functions
echo "file drivers/clk/clk-composite.c +pflm" > /sys/kernel/debug/dynamic_debug/control

# تفعيل في core CCF
echo "file drivers/clk/clk.c +p" > /sys/kernel/debug/dynamic_debug/control
```

**الـ error الوحيد اللي بيطبعه `clk-composite.c` مباشرةً:**

```c
pr_err("clk: clk_composite_determine_rate function called, but no mux or rate callback set!\n");
```

ده بيظهر لو الـ composite اتسجل من غير mux_ops و rate_ops — خطأ في الـ registration.

---

#### 5. Kernel Config Options للـ Debugging

| الـ Option | الغرض |
|-----------|-------|
| `CONFIG_COMMON_CLK` | الـ CCF الأساسية — لازم تكون enabled |
| `CONFIG_CLK_DEBUG` | يفعّل debugfs entries للـ clocks |
| `CONFIG_DEBUG_FS` | لازم لـ `/sys/kernel/debug/clk/` |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في clock locks |
| `CONFIG_DEBUG_OBJECTS` | يتحقق من صحة الـ clock objects |
| `CONFIG_KASAN` | يكتشف heap overflows في `kzalloc(composite)` |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` dynamically |
| `CONFIG_TRACING` | لازم لـ ftrace events |
| `CONFIG_CLK_QCOM` أو غيره | platform-specific composite clocks |

```bash
# التحقق من الـ config الحالي
grep -E "CONFIG_(COMMON_CLK|CLK_DEBUG|DEBUG_FS|DYNAMIC_DEBUG)" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الأداة الأهم: `clk_summary` + `clk_dump`**

```bash
# JSON-like dump لكل الـ clock tree
cat /sys/kernel/debug/clk/clk_dump | python3 -m json.tool | grep -A5 "composite"

# تتبع مشاكل الـ orphan clocks
grep -i "orphan" /sys/kernel/debug/clk/clk_summary
```

**أداة `devlink` مش مباشرة للـ clocks، لكن ممكن تستخدم:**

```bash
# لو الـ composite clock جزء من network driver
devlink dev info pci/0000:01:00.0

# أداة clk-test لو موجودة في kernel selftests
cd tools/testing/selftests/clk && make && ./clk_test
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|-------|
| `clk: clk_composite_determine_rate function called, but no mux or rate callback set!` | الـ composite اتسجل بدون mux_ops ولا rate_ops مع وجود طلب determine_rate | تأكد من تمرير mux_hw+mux_ops أو rate_hw+rate_ops صح |
| `%s: missing round_rate op is required` | الـ rate_ops بيحتوي على `set_rate` بس من غير `round_rate` أو `determine_rate` | أضف `round_rate` أو `determine_rate` في rate_ops |
| `clk_hw_register failed` | فشل تسجيل الـ composite في الـ CCF | راجع `dmesg` لتفاصيل أكتر، غالباً اسم متكرر أو parent مش موجود |
| `clk: couldn't get parent clock` | الـ parent_names فيها اسم مش متسجل | تأكد من ترتيب تسجيل الـ clocks (الـ parents الأول) |
| `-EINVAL` from `clk_hw_register_composite` | mux_ops بدون `get_parent`، أو gate_ops ناقصة `is_enabled`/`enable`/`disable` | راجع أن الـ ops struct مكتمل |
| `WARNING: ... WARN(1, ...)` | `set_rate` موجود بدون `round_rate`/`determine_rate` | اكتمل الـ ops |
| `clk: failed to reparent` | فشل `set_parent` على الـ mux | راجع الـ hardware register access في الـ mux driver |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في __clk_hw_register_composite — بعد clk_hw_register */
ret = clk_hw_register(dev, hw);
if (ret) {
    pr_err("composite: register failed for '%s': %d\n", name, ret);
    /* هنا ضيف dump_stack() لو محتاج تعرف من طلب الـ register */
    dump_stack();
    hw = ERR_PTR(ret);
    goto err;
}

/* في clk_composite_determine_rate — قبل الـ pr_err */
WARN_ON(!mux_ops && !rate_ops);
pr_err("...");

/* في clk_composite_set_rate_and_parent — تحقق من الـ rate order */
WARN_ON(rate_ops->recalc_rate == NULL);

/* في clk_composite_get_parent — تحقق من null */
WARN_ON(!mux_ops->get_parent);
```

**نقاط مقترحة للـ WARN_ON في vetting:**

```c
/* تأكد إن mux_hw->clk اتضبط صح بعد الـ register */
WARN_ON(composite->mux_hw && !composite->mux_hw->clk);
WARN_ON(composite->rate_hw && !composite->rate_hw->clk);
WARN_ON(composite->gate_hw && !composite->gate_hw->clk);
```

---

### Hardware Level

#### 1. التحقق من تطابق الـ Hardware State مع الـ Kernel State

الـ `clk_composite` بيجمع 3 أجزاء hardware:

```
+------------------------------------------+
|           clk_composite                  |
|  +--------+  +---------+  +---------+   |
|  | mux_hw |  | rate_hw |  | gate_hw |   |
|  | (MUX)  |  | (DIV)   |  | (GATE)  |   |
|  +---+----+  +----+----+  +----+----+   |
+------|------------|------------|----------+
       v            v            v
   HW registers  HW registers  HW registers
```

```bash
# قراءة الـ rate الحالي من kernel
cat /sys/kernel/debug/clk/<composite-name>/clk_rate

# مقارنة بالـ oscilloscope أو hardware register مباشرة
# افرض الـ clock على GPIO output عبر الـ clock testing pin
# أو استخدم devmem2 لقراءة الـ clock control register مباشرة
```

**مثال عملي — Rockchip composite clock:**

```bash
# قراءة الـ clock rate من kernel
cat /sys/kernel/debug/clk/clk_gpll/clk_rate
# output: 1200000000

# قراءة الـ register مباشرة (GPLL control على RK3399 مثلاً)
devmem2 0xFF760000 w   # CRU base + PLL offset
# قارن الـ bits مع الـ datasheet
```

---

#### 2. Register Dump Techniques

```bash
# تثبيت devmem2
apt install devmem2
# أو
git clone https://github.com/hackndev/devmem2 && cd devmem2 && make

# قراءة register واحد (32-bit)
devmem2 0xFF760000 w

# قراءة عدة registers متتالية عبر /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x100, offset=0xFF760000)
    for i in range(0, 0x40, 4):
        val = struct.unpack('<I', m[i:i+4])[0]
        print(f'0x{0xFF760000+i:08X}: 0x{val:08X}')
"

# استخدام io utility (من package i2c-tools أو busybox)
io -4 -r 0xFF760000
```

**لو الـ /dev/mem محظور بسبب `CONFIG_STRICT_DEVMEM`:**

```bash
# استخدم kernel module بدل /dev/mem
# أو فعّل CONFIG_STRICT_DEVMEM=n في kernel .config
# أو استخدم crash utility مع vmcore
crash /usr/lib/debug/boot/vmlinux /proc/kcore
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**لو عندك Saleae أو بروب oscilloscope:**

```
1. ابحث في الـ datasheet عن clock test output pin (بعض الـ SoCs زي Allwinner
   بيدعم CLK_OUT pins)

2. فعّل الـ clock output على الـ pin المناسب:
```

```bash
# Allwinner مثلاً — تفعيل CLK_OUT0
echo 1 > /sys/class/gpio/gpio<N>/value   # pin specific to SoC
```

```
3. اضبط الـ oscilloscope:
   - Timebase: 1/rate × 10  (مثلاً لـ 100MHz → 10ns/div)
   - Trigger: Rising edge
   - قيس الـ frequency بالـ cursor

4. Logic Analyzer (Saleae):
   - Sample rate: >= 4× clock frequency
   - استخدم "Clock" decoder
   - قارن النتيجة مع clk_rate في debugfs
```

**مشاكل شائعة تشوفها على الـ scope:**

| ما تشوفه على الـ Scope | التفسير |
|------------------------|---------|
| لا توجد إشارة | الـ gate مغلق (disabled) أو الـ parent مش شغال |
| frequency أقل من المتوقع | الـ divider (rate_hw) مضبوط على قيمة عالية |
| jitter عالي | noise على الـ power supply أو مشكلة في الـ PLL |
| frequency صح بس unstable | التحويل جارٍ (clk_set_rate in progress) |

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | ما يظهر في dmesg | التشخيص |
|---------|-----------------|---------|
| الـ mux مش بيتحول | `clk: failed to reparent X to Y` | راجع الـ set_parent في mux driver وتحقق من الـ register |
| الـ divider بيرفض الـ rate | `round_rate returned negative` | الـ rate المطلوب أقل من minimum أو أكبر من maximum الـ divider |
| الـ gate مش بيـ enable | timeout في الـ driver بعد clk_enable | راجع الـ is_enabled — ممكن hardware busy flag |
| الـ parent مش متسجل | `clk: orphan clock <name>` | الـ parent_names غلط أو الـ parent اتسجل بعد الـ child |
| OOM عند الـ registration | `kzalloc failed` في composite alloc | ذاكرة ناقصة — نادر، راجع `/proc/meminfo` |

---

#### 5. Device Tree Debugging

```bash
# عرض الـ DT node للـ clock controller
cat /proc/device-tree/clock-controller@FF760000/compatible
# output: rockchip,rk3399-cru

# التحقق من الـ clocks property في device node
fdtget /boot/dtb/<board>.dtb /ethernet@FF800000 clocks

# مقارنة بالـ DT compiled
dtc -I fs /proc/device-tree > /tmp/current_dt.dts
grep -A5 "composite" /tmp/current_dt.dts
```

**التحقق من صحة الـ DT:**

```bash
# لازم الـ clock-cells يتطابق مع عدد الـ args في المستهلك
# في clock provider:
#   clock-cells = <1>;  ← index-based (single argument)
# في clock consumer:
#   clocks = <&cru SCLK_UART0>;  ← يمرر index

# لو في خطأ في الـ clock-cells:
# dmesg بيظهر: "clk: invalid clock specifier size for ..."

# التحقق من تطابق الـ clock indices
grep -r "SCLK_UART0" include/dt-bindings/clock/
```

**مثال DT صح لـ composite clock:**

```dts
cru: clock-controller@ff760000 {
    compatible = "rockchip,rk3399-cru";
    reg = <0x0 0xff760000 0x0 0x1000>;
    #clock-cells = <1>;
    assigned-clocks = <&cru ACLK_PERIHP>;
    assigned-clock-rates = <150000000>;
};

uart0: serial@ff180000 {
    clocks = <&cru SCLK_UART0>, <&cru PCLK_UART0>;
    clock-names = "baudclk", "apb_pclk";
};
```

لو `assigned-clock-rates` مش شغال — راجع إن الـ composite's rate_ops بيدعم `set_rate`.

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

```bash
# ===== 1. فحص شامل للـ clock tree =====
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null || \
    echo "debugfs غير mounted — جرب: mount -t debugfs none /sys/kernel/debug"

# ===== 2. إيجاد composite clock بالاسم =====
CLK_NAME="cpu-composite"
echo "Rate:    $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_rate)"
echo "Enabled: $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_enable_count)"
echo "Parent:  $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_parent)"
echo "Parents: $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_possible_parents)"
echo "Flags:   $(cat /sys/kernel/debug/clk/$CLK_NAME/clk_flags)"

# ===== 3. تفعيل ftrace لكل clock events =====
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ workload هنا
cat /sys/kernel/debug/tracing/trace | grep -E "(set_rate|set_parent|enable|disable)"

# ===== 4. تفعيل dynamic debug لـ clk-composite =====
echo "file drivers/clk/clk-composite.c +pflm" > \
    /sys/kernel/debug/dynamic_debug/control
# تأكد من التفعيل
grep "clk-composite" /sys/kernel/debug/dynamic_debug/control

# ===== 5. مراقبة dmesg أثناء عمل الـ clock =====
dmesg -w | grep -iE "(clk|clock|composite|pll|mux|divider|gate)"

# ===== 6. فحص الـ orphan clocks =====
grep -v "^$" /sys/kernel/debug/clk/clk_summary | awk '$4 == 0 {print "ORPHAN?", $1}'

# ===== 7. قياس الـ clock rate change time عبر ftrace =====
CLK="cpu-composite"
echo "name == \"$CLK\"" > /sys/kernel/debug/tracing/events/clk/clk_set_rate/filter
echo "name == \"$CLK\"" > /sys/kernel/debug/tracing/events/clk/clk_set_rate_complete/filter
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate_complete/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ workload
cat /sys/kernel/debug/tracing/trace
# ابحث عن الفرق بين timestamp of clk_set_rate و clk_set_rate_complete

# ===== 8. إيجاد كل الـ composite clocks في النظام =====
for d in /sys/kernel/debug/clk/*/; do
    clk=$(basename "$d")
    parents=$(cat "$d/clk_possible_parents" 2>/dev/null)
    rate=$(cat "$d/clk_rate" 2>/dev/null)
    # composite عادة بيبقى ليه أكتر من parent واحد (mux)
    count=$(echo $parents | wc -w)
    [ "$count" -gt 1 ] && echo "Possible composite: $clk (parents: $parents, rate: $rate)"
done
```

#### تفسير الـ Output

```
# من clk_summary:
#
#                          enable  prepare  protect
#   clock                    cnt     cnt      cnt        rate
# ---------------------------------------------------------------
#   cpu-composite              1       1        0   816000000
#     ↑                        ↑       ↑                ↑
#   الاسم              enable_count  prepare_count    الـ rate
#
# enable_count=1  → الـ gate مفتوح (enabled)
# prepare_count=1 → الـ clock prepared (يقدر يتـ enable)
# rate=816000000  → 816 MHz

# من ftrace:
#
#   kworker-23 [000] 1234.000: clk_set_rate: cpu-composite 1200000000
#                                                ↑              ↑
#                                             اسم الـ clk    الـ rate الجديد
#   kworker-23 [000] 1234.002: clk_set_parent: cpu-composite 2
#                                                                ↑
#                                                         index الـ parent الجديد
#   kworker-23 [000] 1234.003: clk_set_rate_complete: cpu-composite 1200000000
#                                                                      ↑
#                                                           الـ rate اللي اتضبط فعلاً
#
# الفرق 1234.003 - 1234.000 = 3ms → وقت الـ rate change
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على بورد RK3562 — Industrial Gateway

#### العنوان
**الـ UART2 واقف عند bring-up على gateway صناعي بناءً على RK3562**

#### السياق
فريق bring-up بيشتغل على gateway صناعي بيستخدم RK3562. البورد محتاج UART2 يشتغل بـ 115200 baud للتواصل مع وحدة Modbus RTU. الـ kernel بيبوت، لكن الـ UART مش بيرد خالص.

#### المشكلة
الـ `clk_set_rate()` بترجع صح (`0`) لكن فعلياً الـ UART clock مش بيتغير. الـ baud rate على الـ oscilloscope بيظهر قيمة عشوائية مش `115200 * 16 = 1.8432 MHz`.

#### التحليل
الـ UART clock على RK3562 بيتعمل register كـ `clk_composite`. اللي بيحصل:

```c
/* clk_composite_determine_rate — اللي بيتكال لما نطلب rate معين */
static int clk_composite_determine_rate(struct clk_hw *hw,
                                        struct clk_rate_request *req)
{
    /* لو في rate_hw + rate_ops بس مفيش mux_ops->set_parent */
    } else if (rate_hw && rate_ops && rate_ops->determine_rate) {
        __clk_hw_set_clk(rate_hw, hw);
        return rate_ops->determine_rate(rate_hw, req);  /* <-- بيتكال */
    }
```

المشكلة إن الـ composite clock اتعمل register بـ flag `CLK_SET_RATE_NO_REPARENT`. لما بتطلب rate، الكود بيدخل هنا:

```c
if (clk_hw_get_flags(hw) & CLK_SET_RATE_NO_REPARENT) {
    parent = clk_hw_get_parent(mux_hw);
    /* بياخد الـ parent الحالي فقط — مش بيدور على أحسن parent */
    clk_hw_forward_rate_request(hw, req, parent, &tmp_req, req->rate);
    ret = clk_composite_determine_rate_for_parent(rate_hw,
                                                  &tmp_req,
                                                  parent,
                                                  rate_ops);
```

الـ parent الحالي مش قادر يوفر الـ rate المطلوبة، لكن الكود مش بيجرب باقي الـ parents لأن الـ flag منع ده.

#### الحل
**1 — شيل الـ flag الغلط من driver الـ RK3562:**

```c
/* في drivers/clk/rockchip/clk-rk3562.c */
/* قبل: */
COMPOSITE(SCLK_UART2, "clk_uart2", mux_parents, CLK_SET_RATE_NO_REPARENT, ...),

/* بعد: */
COMPOSITE(SCLK_UART2, "clk_uart2", mux_parents, 0, ...),
```

**2 — تأكد بـ debugfs:**

```bash
# اقرأ الـ rate الحقيقية
cat /sys/kernel/debug/clk/clk_uart2/clk_rate

# اقرأ الـ parent الحالي
cat /sys/kernel/debug/clk/clk_uart2/clk_parent

# شوف كل الـ parents المتاحة
ls /sys/kernel/debug/clk/ | grep uart
```

**3 — بعد التصحيح، اطلب الـ rate صح:**

```bash
# من userspace للتجربة
echo 1843200 > /sys/kernel/debug/clk/clk_uart2/clk_rate
```

#### الدرس المستفاد
**الـ `CLK_SET_RATE_NO_REPARENT`** بيمنع `clk_composite_determine_rate` من إنه يلف على كل الـ parents في الـ loop (السطر 107). لو الـ parent الحالي مش قادر يوفر الـ rate، الطلب بيفشل بصمت أو بيرجع أقرب rate مش صح. دايماً افحص الـ flags قبل ما تفترض إن الـ composite clock هيدور على أحسن parent.

---

### السيناريو 2: HDMI مش بيشتغل بـ 4K على Android TV Box بناءً على Allwinner H616

#### العنوان
**الـ HDMI عالق عند 1080p بدل 4K@30fps على TV Box بناءً على H616**

#### السياق
شركة بتعمل Android TV box بـ Allwinner H616. العميل بيشكي إن الجهاز مش بيعرض 4K رغم إن الـ HDMI 2.0 مدعوم. الـ display driver بيطلب `594 MHz` للـ TCON clock لكن الجهاز بيرجع `297 MHz`.

#### المشكلة
الـ `clk_set_rate_and_parent()` مش بتشتغل صح. البورد بيوصل لـ `clk_composite_set_rate_and_parent` لكن ترتيب العمليات بيسبب glitch.

#### التحليل
```c
static int clk_composite_set_rate_and_parent(struct clk_hw *hw,
                                             unsigned long rate,
                                             unsigned long parent_rate,
                                             u8 index)
{
    unsigned long temp_rate;

    __clk_hw_set_clk(rate_hw, hw);
    __clk_hw_set_clk(mux_hw, hw);

    /* بيحسب الـ rate الحالية */
    temp_rate = rate_ops->recalc_rate(rate_hw, parent_rate);

    if (temp_rate > rate) {
        /* لو الـ rate الحالية أكبر من المطلوبة:
           أول حاجة قلل الـ rate، بعدين غير الـ parent */
        rate_ops->set_rate(rate_hw, rate, parent_rate);
        mux_ops->set_parent(mux_hw, index);
    } else {
        /* لو الـ rate الحالية أصغر من المطلوبة:
           أول حاجة غير الـ parent، بعدين ارفع الـ rate */
        mux_ops->set_parent(mux_hw, index);
        rate_ops->set_rate(rate_hw, rate, parent_rate);
    }
```

المشكلة إن الـ `rate_ops->recalc_rate(rate_hw, parent_rate)` بتاخد `parent_rate` كـ parameter، لكن الـ `parent_rate` ده هو الـ rate بتاع الـ parent الجديد (اللي هيتغير)، مش الـ parent الحالي. في H616، الـ TCON مرتبط بـ PLL_VIDEO0 و PLL_VIDEO1، والـ framework بيبعت `parent_rate` للـ parent الجديد قبل ما الـ mux يتغير فعلياً.

النتيجة: `temp_rate` بيتحسب غلط، فالترتيب اللي الكود بياخده بيسبب لحظة إن الـ divider والـ mux يكونوا في حالة inconsistent، فالـ HDMI PHY بيخسر الـ lock.

#### الحل
**1 — اقرأ الـ rate الحالية من الـ hardware مباشرة:**

```c
/* في driver الـ H616، قبل ما تسجل الـ composite clock، تأكد إن
   rate_ops->recalc_rate بتقرأ من الـ hardware مش من الـ cache */
static unsigned long sun50i_h616_tcon_recalc_rate(struct clk_hw *hw,
                                                   unsigned long parent_rate)
{
    /* قرأ الـ divider من الـ register مباشرة */
    u32 reg = readl(clk->base + CLK_TCON_REG);
    u32 div = (reg >> 0) & 0xF;
    return parent_rate / (div + 1);
}
```

**2 — استخدم الـ debugfs لتتبع ترتيب العمليات:**

```bash
# فعّل الـ clock tracing
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
cat /sys/kernel/debug/tracing/trace
```

**3 — تحقق من الـ parent المختار:**

```bash
cat /sys/kernel/debug/clk/tcon_top_tcon0/clk_parent
cat /sys/kernel/debug/clk/tcon_top_tcon0/clk_rate
```

#### الدرس المستفاد
**الـ `clk_composite_set_rate_and_parent`** بيعمل decision بناءً على `recalc_rate` اللي ممكن تاخد `parent_rate` للـ parent الجديد مش الحالي. لو الـ hardware حساس لـ glitches زي الـ HDMI PHY، لازم الـ rate_ops->recalc_rate تقرأ الـ hardware مباشرة، وممكن تحتاج تعمل workaround بـ safe intermediate rate.

---

### السيناريو 3: I2C بيتعطل تحت load على STM32MP1 — IoT Sensor Hub

#### العنوان
**الـ I2C3 بيدي `EREMOTEIO` عشوائياً على بورد STM32MP1 في sensor hub**

#### السياق
جهاز IoT بيجمع بيانات من 8 sensors عن طريق I2C3 على STM32MP1. تحت load عالي (CPU بيشتغل 100%)، بعض الـ I2C transactions بتفشل بـ `-EREMOTEIO`. المشكلة مش منتظمة — بتحصل مرة كل دقايق.

#### المشكلة
الـ I2C clock بيتغير في اللحظة الغلط. الـ `clk_composite_determine_rate` بيختار parent مختلف لما الـ CPU تحت load بسبب DVFS (Dynamic Voltage and Frequency Scaling)، فالـ I2C clock بيتغير وسط transaction.

#### التحليل
الـ STM32MP1 عنده I2C clock كـ composite clock بـ mux بين `HSI` و `CSI` و `PLL4_R`:

```c
/* لما الـ DVFS بيغير parent الـ PLL، الـ composite determine_rate
   بيلف على كل الـ parents */
for (i = 0; i < clk_hw_get_num_parents(mux_hw); i++) {
    struct clk_rate_request tmp_req;

    parent = clk_hw_get_parent_by_index(mux_hw, i);
    if (!parent)
        continue;

    clk_hw_forward_rate_request(hw, req, parent, &tmp_req, req->rate);
    ret = clk_composite_determine_rate_for_parent(rate_hw,
                                                  &tmp_req,
                                                  parent,
                                                  rate_ops);
    if (ret)
        continue;

    /* بيختار أحسن parent */
    if (!rate_diff || !req->best_parent_hw
                   || best_rate_diff > rate_diff) {
        req->best_parent_hw = parent;  /* <-- ممكن يغير الـ parent */
```

لما الـ DVFS بيطلب rate جديدة للـ CPU، الـ framework بيعيد حساب rates الـ clocks المرتبطة. لو الـ I2C composite clock مش معلوم عليه `CLK_SET_RATE_NO_REPARENT`، الـ framework ممكن يغير الـ parent وسط I2C transaction.

#### الحل
**1 — ثبّت الـ I2C clock parent:**

```c
/* في stm32mp1_clk.c */
/* قبل: */
STM32_COMPOSITE_NODIV(I2C3_K, "i2c3_k", 0,
                      i2c35_src, ARRAY_SIZE(i2c35_src), ...),

/* بعد — أضف CLK_SET_RATE_NO_REPARENT: */
STM32_COMPOSITE_NODIV(I2C3_K, "i2c3_k",
                      CLK_SET_RATE_NO_REPARENT,
                      i2c35_src, ARRAY_SIZE(i2c35_src), ...),
```

**2 — أو ثبّت الـ parent من الـ DT:**

```dts
/* في board DTS */
&i2c3 {
    clocks = <&rcc I2C3_K>;
    assigned-clocks = <&rcc I2C3_K>;
    assigned-clock-parents = <&rcc HSI_K>; /* HSI ثابت، مش DVFS */
    assigned-clock-rates = <100000>;        /* 100 kHz */
};
```

**3 — تحقق من الـ parent الحالي وأكد إنه ثابت:**

```bash
# قبل load
cat /sys/kernel/debug/clk/i2c3_k/clk_parent

# شغّل load
stress-ng --cpu 4 --timeout 60s &

# بعد load
cat /sys/kernel/debug/clk/i2c3_k/clk_parent
# لازم يكون نفس الـ parent
```

#### الدرس المستفاد
**الـ `clk_composite_determine_rate`** بيلف على كل الـ parents ويختار أحسنهم لو مفيش `CLK_SET_RATE_NO_REPARENT`. ده ممكن يعمل reparent غير متوقع وسط I2C/SPI transaction. الـ peripherals اللي بتعمل transactions طويلة لازم clock بتاعها يكون بـ `CLK_SET_RATE_NO_REPARENT` أو parent ثابت من الـ DT.

---

### السيناريو 4: SPI Flash مش بيتعرف عليه على i.MX8 — Automotive ECU

#### العنوان
**فشل `spi_nor_probe` على i.MX8M Plus في ECU بسبب composite clock غلط**

#### السياق
ECU للسيارة بيستخدم i.MX8M Plus. الـ SPI NOR flash (W25Q256) مش بيتعرف عليه وقت الـ boot. الـ kernel log بيظهر:

```
spi-nor spi0.0: unrecognized JEDEC id bytes: ff ff ff
spi_imx 30820000.spi: spi transfer failed: -110
```

#### المشكلة
الـ ECSPI1 clock مش متضبط صح. الـ `clk_hw_register_composite` اتكال بـ `gate_ops` ناقصة.

#### التحليل
في `__clk_hw_register_composite`، في validation صارمة على الـ gate_ops:

```c
if (gate_hw && gate_ops) {
    if (!gate_ops->is_enabled || !gate_ops->enable ||
        !gate_ops->disable) {
        hw = ERR_PTR(-EINVAL);
        goto err;
    }

    composite->gate_hw = gate_hw;
    composite->gate_ops = gate_ops;
    clk_composite_ops->is_enabled = clk_composite_is_enabled;
    clk_composite_ops->enable = clk_composite_enable;
    clk_composite_ops->disable = clk_composite_disable;
}
```

لو الـ `gate_ops` ناقصة واحدة من الـ 3 callbacks، الـ register بيفشل بـ `-EINVAL`. لكن في حالتنا، المشكلة مختلفة: الـ driver الـ custom اللي عمله فريق الـ ECU عمل `gate_hw` و`gate_ops` صح، لكن نسي إن الـ gate بيتكنترل بـ CCM register مختلف.

لما الـ `clk_composite_enable` بيتكال:

```c
static int clk_composite_enable(struct clk_hw *hw)
{
    struct clk_composite *composite = to_clk_composite(hw);
    const struct clk_ops *gate_ops = composite->gate_ops;
    struct clk_hw *gate_hw = composite->gate_hw;

    __clk_hw_set_clk(gate_hw, hw);  /* بيربط الـ gate_hw بالـ composite */
    return gate_ops->enable(gate_hw);  /* بيكتب في الـ register الغلط */
}
```

الـ `__clk_hw_set_clk` بيعمل `gate_hw->clk = hw->clk`، فالـ gate بيستخدم الـ `clk` بتاع الـ composite مش الـ gate نفسه. لو الـ gate_hw بيستخدم base address بيتحسب من `hw->clk`، النتيجة هتكون كتابة في address غلط.

لكن المشكلة الأساسية إن فريق الـ ECU عرّف الـ composite clock بدون `gate_hw` بالغلط (مرروا `NULL`)، فالـ gate مش بيتكال أصلاً، والـ ECSPI clock مش بيتفعل:

```c
/* الكود الغلط في custom i.MX8 driver */
hw = clk_hw_register_composite(dev, "ecspi1_root_clk",
        ecspi_sels, ARRAY_SIZE(ecspi_sels),
        mux_hw, &clk_mux_ops,
        div_hw, &clk_divider_ops,
        NULL, NULL,  /* <-- gate_hw و gate_ops = NULL */
        CLK_SET_RATE_NO_REPARENT);
```

لأن `gate_hw` = `NULL`، مفيش `clk_composite_ops->enable` اتسابت، فالـ CCF بيفترض إن الـ clock دايماً enabled — لكن فعلياً الـ hardware gate مقفل.

#### الحل
**1 — صحّح تسجيل الـ composite clock:**

```c
/* صح: مرّر الـ gate_hw الحقيقي */
hw = clk_hw_register_composite(dev, "ecspi1_root_clk",
        ecspi_sels, ARRAY_SIZE(ecspi_sels),
        mux_hw, &clk_mux_ops,
        div_hw, &clk_divider_ops,
        gate_hw, &clk_gate_ops,  /* <-- السطر المهم */
        CLK_SET_RATE_NO_REPARENT);
```

**2 — تحقق بـ debugfs:**

```bash
# شوف إذا كان الـ enable_count صح
cat /sys/kernel/debug/clk/ecspi1_root_clk/clk_enable_count

# شوف الـ is_enabled
cat /sys/kernel/debug/clk/ecspi1_root_clk/clk_enable_count

# قرأ الـ rate الحالية
cat /sys/kernel/debug/clk/ecspi1_root_clk/clk_rate
```

**3 — تأكد من الـ CCM register مباشرة:**

```bash
# اقرأ CCM CCGR register المسؤول عن ECSPI1
devmem2 0x30384050 w
```

#### الدرس المستفاد
**لما بتمرر `NULL` لـ `gate_hw`**، الـ `__clk_hw_register_composite` مش بيسيب `enable/disable/is_enabled` في الـ ops، فالـ CCF بيعتقد إن الـ clock دايماً on. الـ peripheral مش بيشتغل لأن الـ hardware gate مش بيتفتح. دايماً اتأكد إنك بتمرر الـ 3 components (mux, rate, gate) صح وإن كل callback موجودة قبل ما تسجل الـ composite.

---

### السيناريو 5: USB مش بيشوف Devices على AM62x — Custom Board Bring-up

#### العنوان
**الـ USB3.0 SuperSpeed مش شغال على custom board بناءً على AM62x**

#### السياق
فريق bring-up بيشتغل على custom board بناءً على Texas Instruments AM62x. الـ USB2.0 شغال تمام، لكن الـ USB3.0 SuperSpeed مش بيشوف أي device. الـ lsusb بيظهر الـ xHCI controller لكن مفيش devices.

#### المشكلة
الـ USB3 pipe clock (اللي بيغذي الـ USB3 PHY) مش بيوصل للـ rate الصح. الـ SuperSpeed بيحتاج `5 Gbps` على الـ SerDes، وده بيحتاج `250 MHz` reference clock للـ PHY.

#### التحليل
الـ AM62x عنده `usb3_phy_clk` كـ composite clock بـ mux بين `HSDIV4_16FFT_MAIN_0_HSDIVOUT8_CLK` و `CLK_EXT_REFCLK`. الـ `determine_rate` بيلف على الاتنين:

```c
for (i = 0; i < clk_hw_get_num_parents(mux_hw); i++) {
    struct clk_rate_request tmp_req;

    parent = clk_hw_get_parent_by_index(mux_hw, i);
    if (!parent)
        continue;

    clk_hw_forward_rate_request(hw, req, parent, &tmp_req, req->rate);
    ret = clk_composite_determine_rate_for_parent(rate_hw,
                                                  &tmp_req,
                                                  parent,
                                                  rate_ops);
    if (ret)
        continue;

    if (req->rate >= tmp_req.rate)
        rate_diff = req->rate - tmp_req.rate;
    else
        rate_diff = tmp_req.rate - req->rate;

    if (!rate_diff || !req->best_parent_hw
                   || best_rate_diff > rate_diff) {
        req->best_parent_hw = parent;
        req->best_parent_rate = tmp_req.best_parent_rate;
        best_rate_diff = rate_diff;
        best_rate = tmp_req.rate;
    }
```

المشكلة إن الـ `CLK_EXT_REFCLK` مش موجود على الـ custom board (الـ pad مش متوصل)، لكن الـ DT بيعرّفه كـ parent. الـ `clk_hw_get_parent_by_index` بيرجع `NULL` لأن الـ external clock مش registered، فالكود بيعمل `continue` ويعدي.

لكن المشكلة الأعمق: الـ `HSDIV4_16FFT_MAIN_0_HSDIVOUT8_CLK` ممكن يوفر `250 MHz` بس عايز الـ PLL الأصلي يبقى configured بشكل معين. الـ `determine_rate_for_parent` بيحسب:

```c
static int clk_composite_determine_rate_for_parent(struct clk_hw *rate_hw,
                                                   struct clk_rate_request *req,
                                                   struct clk_hw *parent_hw,
                                                   const struct clk_ops *rate_ops)
{
    req->best_parent_hw = parent_hw;
    req->best_parent_rate = clk_hw_get_rate(parent_hw);  /* بياخد الـ current rate */

    if (rate_ops->determine_rate)
        return rate_ops->determine_rate(rate_hw, req);

    rate = rate_ops->round_rate(rate_hw, req->rate,
                                &req->best_parent_rate);
```

الـ `clk_hw_get_rate(parent_hw)` بيرجع `0` لأن الـ PLL مش initialized بعد — الـ bring-up بيحصل قبل ما `clk_set_rate_ungate` يشتغل. فالـ divider بيحسب على أساس parent rate = 0، والـ round_rate بيرجع 0.

#### الحل
**1 — تأكد إن الـ parent PLLs initialized قبل USB:**

```dts
/* في board DTS */
&usbss0 {
    assigned-clocks = <&k3_clks 151 0>;
    assigned-clock-parents = <&k3_clks 151 2>; /* HSDIV4 مش EXT */
    assigned-clock-rates = <250000000>;
};
```

**2 — تحقق من initialization order بـ debugfs:**

```bash
# تأكد إن الـ PLL شغال قبل USB probe
cat /sys/kernel/debug/clk/hsdiv4_16fft_main_0_hsdivout8_clk/clk_rate
# لازم يرجع 250000000

# شوف الـ USB3 PHY clock
cat /sys/kernel/debug/clk/usb3_phy_clk/clk_rate
cat /sys/kernel/debug/clk/usb3_phy_clk/clk_parent
```

**3 — لو الـ bring-up لسه مكتمل، force الـ rate يدوياً:**

```bash
# force enable الـ parent PLL
echo 250000000 > /sys/kernel/debug/clk/hsdiv4_16fft_main_0_hsdivout8_clk/clk_rate

# بعدين reset الـ USB controller
echo 1 > /sys/bus/platform/drivers/xhci-hcd/3100000.usb/remove
echo 3100000.usb > /sys/bus/platform/drivers/xhci-hcd/bind
```

**4 — تأكد من الـ SuperSpeed link:**

```bash
lsusb -t
# المفروض تشوف SuperSpeed (5000M) مش HighSpeed (480M)
```

#### الدرس المستفاد
**الـ `clk_composite_determine_rate_for_parent`** بتستخدم `clk_hw_get_rate(parent_hw)` اللي ممكن يرجع `0` لو الـ parent مش initialized بعد. لازم تتأكد إن ترتيب الـ clock initialization في الـ DT صح، وإن الـ parent PLLs موجودة وشغالة قبل الـ peripherals اللي بتعتمد عليها. مشاكل الـ USB3 PHY كتير بيجي من clock rate = 0 مش من hardware fault.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [A common clock framework](https://lwn.net/Articles/472998/) | أول مقال بيشرح تصميم الـ CCF — بيغطي الـ `struct clk_ops`، الـ mux/rate/gate، وفلسفة الـ framework |
| [common clk framework](https://lwn.net/Articles/486841/) | إعلان feature freeze للـ CCF مع تفاصيل التغييرات النهائية قبل الدمج |
| [Common struct clk implementation, v5](https://lwn.net/Articles/392902/) | النسخة المبكرة من المقترح الأصلي — مفيد لفهم التطور التاريخي للـ `struct clk` الموحد |
| [clk: implement clock rate protection mechanism](https://lwn.net/Articles/740484/) | بيتكلم عن إعادة هيكلة الـ `round_rate` و`determine_rate` callbacks — مباشرة متعلق بـ `clk-composite.c` |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | النص الأصلي للـ documentation اللي انبعت على المينلست مع الـ framework |

---

### الـ Official Kernel Documentation

```
Documentation/driver-api/clk.rst
```
الصفحة الرسمية للـ Common Clk Framework — بتشرح الـ provider API، الـ `clk_ops`، وكيفية تسجيل الـ clocks.

- **Online:** [https://docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html)

```
include/linux/clk-provider.h
```
الهيدر الأساسي — بيحتوي على `struct clk_composite`، `struct clk_ops`، `struct clk_hw`، وكل الـ macros زي `to_clk_composite()`.

```
drivers/clk/clk.c
```
الـ core implementation للـ CCF — بيوضح كيف الـ framework بتستدعي الـ composite ops.

```
drivers/clk/clk-composite.c
```
الملف موضوع الدراسة — بيجمع mux + rate + gate في clock واحد.

---

### Kernel Commits المهمة

| الـ Commit | الوصف |
|-----------|-------|
| [patchwork: \[V2\] clk: Add composite clock type](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20130206.081048.71241785637713947.hdoyu@nvidia.com/) | الـ patch الأصلي اللي بعته NVIDIA (Prashant Gaikwad) في فبراير 2013 لإضافة الـ `clk-composite` |
| [barebox port: CLK: Add support for composite clock](http://lists.infradead.org/pipermail/barebox/2015-March/022622.html) | نقل الـ composite clock لـ barebox — بيوضح مدى قابلية الكود للنقل |

**للبحث في git عن تاريخ الملف:**
```bash
git log --follow drivers/clk/clk-composite.c
git log --oneline --follow drivers/clk/clk-composite.c | head -20
```

---

### مناقشات الـ Mailing List

| الرابط | الوصف |
|--------|-------|
| [LKML: \[PATCH v7 2/3\] clk: introduce the common clock framework](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) | الـ patch الرسمي اللي بيعرّف الـ CCF — السياق التاريخي لـ `clk-composite` |
| [bootlin ELCE 2013: Common clock framework how to use it](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | شرح عملي من Free Electrons (bootlin) لمؤتمر ELCE 2013 — بيغطي composite clocks تفصيليًا |

**للبحث في الـ LKML مباشرة:**
```
https://lore.kernel.org/linux-clk/?q=clk-composite
https://lore.kernel.org/linux-clk/?q=clk_hw_register_composite
```

---

### elinux.org

| الرابط | الوصف |
|--------|-------|
| [State of the common struct clk (BoFs PDF)](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | عرض BoF بيغطي تصميم الـ CCF وقرارات الـ API |
| [Generic Linux CLK Framework — CELF 2009 (PDF)](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | العرض التاريخي اللي سبق الـ CCF — بيوضح المشكلة اللي جاء الـ CCF يحلها |

---

### الكتب الموصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الكتاب مجاني:** [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- **الفصل المناسب:** Chapter 1 (Device Driver Architecture) — للخلفية العامة عن الـ kernel subsystems
- ملحوظة: LDD3 قديمة ومش بتغطي الـ CCF بشكل مباشر لأنه اتضاف بعد إصدارها، لكن بتشرح مبادئ الـ device model اللي بيبني عليها الـ CCF.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول المفيدة:**
  - Chapter 17 (Devices and Modules) — بيشرح الـ device model وكيف بتتسجل الـ drivers
  - Chapter 13 (The Virtual Filesystem) — نفس نمط الـ ops struct بيتكرر في الـ `clk_ops`
- الكتاب بيعطي فهم عميق لـ patterns زي الـ `struct clk_ops` (مماثل للـ `file_operations`)

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المناسب:** Chapter 15 (Embedded Linux Applications) والفصول اللي بتتكلم عن Power Management
- بيشرح أهمية الـ clock gating في الـ embedded systems وعلاقته بالـ power consumption

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds (3rd Edition)
- أحدث كتاب عملي للـ embedded Linux — بيغطي الـ device tree والـ clock bindings بشكل عملي

---

### kernelnewbies.org — تغييرات الـ Kernel المتعلقة بالـ CCF

الـ CCF بيتذكر في release notes كتير — ابحث في:

| الرابط | ما يحتويه |
|--------|-----------|
| [Linux_5.9 - Kernel Newbies](https://kernelnewbies.org/Linux_5.9) | تغييرات الـ clock subsystem في 5.9 |
| [Linux_6.7 - Kernel Newbies](https://kernelnewbies.org/Linux_6.7) | آخر تحديثات الـ clk framework |
| [Linux_6.12 - Kernel Newbies](https://kernelnewbies.org/Linux_6.12) | تغييرات حديثة في الـ clock drivers |

للوصول لكل تغييرات الـ clk subsystem عبر الإصدارات:
```
https://kernelnewbies.org/LinuxChanges
```
ثم ابحث عن كلمة "clock" أو "clk" في كل صفحة إصدار.

---

### مصادر إضافية مفيدة

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| STM32MPU Wiki | [Clock overview](https://wiki.st.com/stm32mpu/wiki/Clock_overview) | مثال حقيقي لاستخدام الـ CCF في SoC من ST |
| kernel.org doc | [clk.txt القديم](https://www.kernel.org/doc/Documentation/clk.txt) | النسخة النصية القديمة من الـ documentation |
| infradead mirror | [CCF docs (kernel 5.2)](https://www.infradead.org/~mchehab/rst_conversion/driver-api/clk.html) | ميرور مع تنسيق RST محول |
| cateee.net | [CONFIG_COMMON_CLK](https://cateee.net/lkddb/web-lkddb/COMMON_CLK.html) | قائمة بكل الـ drivers اللي بتعتمد على الـ CCF |
| bootlin elixir | [clk-composite.c cross-reference](https://elixir.bootlin.com/linux/latest/source/drivers/clk/clk-composite.c) | تصفح الكود مع cross-references تفاعلية |
| Android googlesource | [clk-composite.c (msm)](https://android.googlesource.com/kernel/msm.git/+/android-5.1.0_r0.7/drivers/clk/clk-composite.c) | نسخة قديمة للمقارنة التاريخية |

---

### Search Terms للبحث عن معلومات إضافية

```
linux kernel common clock framework CCF clk_ops
clk_composite mux rate gate linux kernel
clk_hw_register_composite linux driver
CLK_SET_RATE_NO_REPARENT linux kernel
devres clk_hw composite driver
linux clock subsystem NVIDIA tegra CCF
struct clk_composite linux kernel internals
determine_rate round_rate difference linux clk
clk_hw_forward_rate_request linux kernel
```

**للبحث في الـ kernel mailing list مباشرة:**
```bash
# على lore.kernel.org
https://lore.kernel.org/linux-clk/

# على LKML
https://lkml.org/lkml/search?q=clk+composite
```
## Phase 8: Writing simple module

### الفكرة

**`clk_hw_register_composite`** هي الـ exported function الأهم في الملف — بتسجّل composite clock جديدة في الـ CCF (Common Clock Framework). هنعمل kprobe عليها عشان نتتبع كل مرة بيتسجّل فيها composite clock في النظام — اسمه، عدد الـ parents، والـ flags المستخدمة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on clk_hw_register_composite:
 * Logs clock name, num_parents, and flags on each registration.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/clk-provider.h> /* struct clk_hw, struct clk_ops */

/*
 * clk_hw_register_composite signature:
 *
 *   struct clk_hw *clk_hw_register_composite(
 *       struct device *dev,
 *       const char *name,
 *       const char * const *parent_names,
 *       int num_parents,
 *       struct clk_hw *mux_hw,   const struct clk_ops *mux_ops,
 *       struct clk_hw *rate_hw,  const struct clk_ops *rate_ops,
 *       struct clk_hw *gate_hw,  const struct clk_ops *gate_ops,
 *       unsigned long flags);
 *
 * On x86-64 the first 6 args are in: rdi, rsi, rdx, rcx, r8, r9
 * Remaining args are on the stack.
 * arg[0]=dev  arg[1]=name  arg[2]=parent_names  arg[3]=num_parents
 * arg[4]=mux_hw  arg[5]=mux_ops ...
 */

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* arg1 = name (const char *), passed in RSI on x86-64 */
    const char *name        = (const char *)regs->si;

    /* arg3 = num_parents (int), passed in RCX on x86-64 */
    int         num_parents = (int)regs->cx;

    /*
     * arg10 = flags (unsigned long) — 10th argument, lives on the stack.
     * After the return address (8 bytes) come the stack args starting at
     * rsp+8.  Args 7-onwards (0-indexed from 6) are at rsp+8, rsp+16, ...
     * arg[6]=gate_hw  arg[7]=gate_ops  arg[8]=flags  → rsp + (3*8) = rsp+24
     * But inside the pre-handler the stack pointer already points past the
     * call frame, so we read via kernel_stack_pointer() helper offset.
     *
     * Simpler: use regs_get_kernel_argument() which handles arch differences.
     */
    unsigned long flags = regs_get_kernel_argument(regs, 10);

    /* Defensive: name might be NULL in error paths */
    if (!name)
        name = "(null)";

    pr_info("clk_composite_kprobe: registering clock \"%s\""
            " | num_parents=%d | flags=0x%lx\n",
            name, num_parents, flags);

    /*
     * Print which sub-clocks are present based on non-NULL ops pointers.
     * arg[5]=mux_ops (R9), arg[7]=rate_ops (stack arg 1 = rsp+8),
     * arg[9]=gate_ops (stack arg 3 = rsp+24).
     */
    pr_info("clk_composite_kprobe:   mux_ops=%s  rate_ops=%s  gate_ops=%s\n",
            regs->r9              ? "yes" : "no",   /* mux_ops  */
            regs_get_kernel_argument(regs, 7) ? "yes" : "no",  /* rate_ops */
            regs_get_kernel_argument(regs, 9) ? "yes" : "no"); /* gate_ops */

    return 0; /* 0 = continue normal execution, do NOT skip the function */
}

/* kprobe descriptor — one per hooked symbol */
static struct kprobe kp = {
    .symbol_name = "clk_hw_register_composite",
    .pre_handler = handler_pre,
};

static int __init clk_composite_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("clk_composite_kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("clk_composite_kprobe: hooked clk_hw_register_composite at %p\n",
            kp.addr);
    return 0;
}

static void __exit clk_composite_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("clk_composite_kprobe: unhooked clk_hw_register_composite\n");
}

module_init(clk_composite_probe_init);
module_exit(clk_composite_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on clk_hw_register_composite to trace composite clock registration");
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه محتاجه |
|--------|-----------|
| `linux/module.h` | الـ macros الأساسية للـ module (init/exit/license) |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `linux/clk-provider.h` | تعريفات `clk_hw` و`clk_ops` للـ type annotations |

---

#### الـ `handler_pre` — الـ Callback

**الـ `pre_handler`** بيتنفّذ قبل ما الـ CPU ينفّذ أول instruction في `clk_hw_register_composite`. الـ `struct pt_regs *regs` بيحتوي على حالة كل الـ registers لحظة الاستدعاء.

```
x86-64 calling convention:
  arg0 (dev)          → RDI  → regs->di
  arg1 (name)         → RSI  → regs->si     ✓ استخدمناه
  arg2 (parent_names) → RDX  → regs->dx
  arg3 (num_parents)  → RCX  → regs->cx     ✓ استخدمناه
  arg4 (mux_hw)       → R8   → regs->r8
  arg5 (mux_ops)      → R9   → regs->r9     ✓ استخدمناه
  arg6+ (على الـ stack) → regs_get_kernel_argument(regs, N)
```

**الـ `regs_get_kernel_argument(regs, N)`** بتجيب الـ argument رقم N بشكل portable بدون ما تحسب الـ stack offsets يدويًا.

الـ `return 0` مهم — لو رجعنا قيمة غير صفر، الـ kprobe بيعمل jmp للـ post_handler ويتخطى الـ function الأصلية، وده مش اللي عايزينه هنا.

---

#### الـ `struct kprobe kp`

**الـ `symbol_name`** بيخلي الـ kernel يحل العنوان وقت الـ `register_kprobe` من الـ kallsyms — مش محتاج تحدد العنوان يدويًا. الـ `.pre_handler` هو الـ callback اللي بيتشغّل قبل كل استدعاء للـ function.

---

#### الـ `module_init` و `module_exit`

في الـ `init`، بنسجّل الـ kprobe — لو فشل (مثلاً الـ symbol مش موجود أو الـ kernel مش مبني بـ `CONFIG_KPROBES=y`) بنرجع الـ error فورًا.

في الـ `exit`، **لازم** نعمل `unregister_kprobe` قبل ما الـ module يتفرغ من الذاكرة، لأن الـ kprobe بيحتوي على pointer للـ `handler_pre` — لو الـ module اتشال والـ kprobe فاضل مسجّل، أي استدعاء للـ function هيؤدي لـ kernel panic بسبب invalid function pointer.

---

### الـ Makefile

```makefile
obj-m += clk_composite_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ Module

```bash
# بناء وتحميل
make
sudo insmod clk_composite_kprobe.ko

# مراقبة الـ log (في terminal ثاني)
sudo dmesg -w | grep clk_composite_kprobe

# لو عايز تجبر registration — load أي driver بيستخدم composite clock
# مثلاً على Raspberry Pi:
sudo modprobe clk-bcm2835

# إزالة الـ module
sudo rmmod clk_composite_kprobe
```

**مثال output متوقع:**

```
clk_composite_kprobe: hooked clk_hw_register_composite at ffffffffc0123456
clk_composite_kprobe: registering clock "uart0_clk" | num_parents=2 | flags=0x0
clk_composite_kprobe:   mux_ops=yes  rate_ops=yes  gate_ops=yes
clk_composite_kprobe: registering clock "emmc_clk"  | num_parents=4 | flags=0x2
clk_composite_kprobe:   mux_ops=yes  rate_ops=yes  gate_ops=no
```

> **ملاحظة:** `CONFIG_KPROBES=y` و`CONFIG_KALLSYMS=y` لازم يكونوا enabled في الـ kernel config. تحقق بـ `grep CONFIG_KPROBES /boot/config-$(uname -r)`.
