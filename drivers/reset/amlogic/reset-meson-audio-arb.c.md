## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من subsystemين متداخلين:
- **RESET CONTROLLER FRAMEWORK** — الـ framework العام في الـ kernel اللي بيدير عمليات reset للـ hardware.
- **ARM/Amlogic Meson SoC Sound Drivers** — لأن Jerome Brunet (نفس الـ author) هو maintainer الـ sound على Amlogic، والـ file ده بيخدم الـ audio DMA engines تحديداً.

---

### الـ Big Picture — إيه اللي بيحصل؟

#### تخيل السيناريو ده

عندك شقة فيها 6 غرف (كل غرفة هي audio channel)، وكل الغرف بتشتغل على نفس خط الكهرباء (الـ memory bus). لو فتحت كل الغرف دلوقتي من غير ترتيب، هيحصل عندك حمل زيادة على الخط وممكن يتعطل. فمحتاج **حارس بوابة** (arbiter) يتحكم مين يدخل ومين لأ.

ده بالظبط اللي بتعمله الـ file دي.

---

### الـ Audio Memory Arbiter — إيه ده؟

في شرائح **Amlogic AXG وSM1** (زي A113X وS905X3)، الـ audio subsystem عنده عدة engines للـ DMA:

- **TODDR** (To DDR) — engines بتكتب صوت من الـ hardware إلى الـ RAM (زي capture من ميكروفون).
- **FRDDR** (From DDR) — engines بتقرا صوت من الـ RAM وتبعته للـ hardware (زي playback في سماعة).

كل engine ده محتاج يوصل للـ memory (DDR) عشان يعمل شغله. لكن المشكلة إن الوصول للـ memory مش مجاني — في **arbiter** (حكم) بيتحكم مين يوصل ومين لأ.

الـ **ARB_GENERAL_BIT** (bit 31) هو زرار التشغيل الرئيسي للـ arbiter ده. لازم يتفعّل الأول قبل ما أي channel يشتغل.

---

### الـ Goal — إيه اللي بيعمله الـ file ده؟

الـ file ده بيعمل **reset controller** خاص بالـ audio memory arbiter. يعني:

1. **بيسجّل** نفسه في الـ Linux Reset Controller Framework.
2. **بيوفّر** لكل audio engine القدرة إنه يعمل assert/deassert لـ reset line بتاعته.
3. **بيتحكم** في bit واحد في register واحد بالـ memory-mapped I/O عشان يفعّل أو يوقف كل channel.
4. **بيحمي** الوصول المتزامن للـ register بـ spinlock عشان ما يحصلش race condition لو اتنين حاولوا يعدّلوا في نفس الوقت.

---

### ليه الـ Reset هنا مش "Reset" عادي؟

في الـ kernel، الـ "reset" مش معناها دايماً إنك بتعمل restart للـ hardware. أحياناً معناها إنك **بتفعّل أو بتوقف** جزء من الـ hardware بشكل controlled. هنا:

- **Assert** (reset فعّال) = بتوقف الـ engine ده، بتمنعه من الوصول للـ memory.
- **Deassert** (reset رفعته) = بتفتح الباب للـ engine ده ويدخل على الـ memory.

ده معمول بالـ bit manipulation على register واحد:
```c
// assert = clear the bit (stop access)
val &= ~BIT(arb->reset_bits[id]);

// deassert = set the bit (allow access)
val |= BIT(arb->reset_bits[id]);
```

---

### الـ AXG vs SM1 — إيه الفرق؟

| SoC | TODDR channels | FRDDR channels | Total |
|-----|---------------|----------------|-------|
| AXG (A113X) | A, B, C | A, B, C | 6 |
| SM1 (S905X3) | A, B, C, D | A, B, C, D | 8 |

الـ SM1 عنده 8 channels بدل 6، وده بيتعامل معاه بـ match data منفصلة:
```c
static const unsigned int sm1_audio_arb_reset_bits[] = {
    [AXG_ARB_TODDR_A] = 0,
    ...
    [AXG_ARB_TODDR_D] = 3,  // extra channel
    [AXG_ARB_FRDDR_D] = 7,  // extra channel
};
```

---

### قصة الـ Probe — إيه اللي بيحصل لما الـ driver يشتغل؟

```
Device Tree
    |
    ↓
meson_audio_arb_probe()
    |
    ├── جيب الـ clock وفعّله (بدون clock، الـ arbiter مش شغال)
    |
    ├── map الـ register في الـ memory
    |
    ├── اكتب BIT(31) في الـ register (فعّل الـ ARB_GENERAL_BIT)
    |   (كل الـ channels لسه مقفولة، بس الـ arbiter نفسه صحي)
    |
    └── سجّل نفسك كـ reset controller
            |
            ↓
    الـ audio engines تقدر دلوقتي تعمل reset_control_deassert()
    عشان تفتح وصولها للـ memory
```

---

### ليه الـ Spinlock مهم هنا؟

لأن الـ register واحد بيتحكم في كل الـ channels. لو channel A بيعمل deassert وفي نفس الوقت channel B بيعمل assert، ممكن يحصل race condition وتتفقد بيانات. الـ spinlock بيضمن إن كل read-modify-write عملية atomically.

```c
spin_lock(&arb->lock);
val = readl(arb->regs);      // قرا الحالة الحالية
val |= BIT(reset_bits[id]);  // عدّل الـ bit المطلوب
writel(val, arb->regs);      // اكتب الكل مرة واحدة
spin_unlock(&arb->lock);
```

---

### الـ Files المرتبطة اللي لازم تعرفها

**الـ Core — Reset Framework:**
- `include/linux/reset-controller.h` — تعريف `reset_controller_dev` و`reset_control_ops`.
- `include/linux/reset.h` — الـ API اللي بيستخدمه الـ consumer (مش الـ provider).
- `drivers/reset/core.c` — قلب الـ reset framework.

**الـ Amlogic Reset Family:**
- `drivers/reset/amlogic/reset-meson.c` — الـ main Amlogic reset controller (للـ SoC-wide resets).
- `drivers/reset/amlogic/reset-meson-common.c` — shared logic بين الـ Amlogic reset drivers.
- `drivers/reset/amlogic/reset-meson-aux.c` — auxiliary reset controller.
- `drivers/reset/amlogic/reset-meson.h` — shared header للـ Amlogic reset drivers.

**الـ DT Bindings:**
- `include/dt-bindings/reset/amlogic,meson-axg-audio-arb.h` — تعريف الـ reset IDs (AXG_ARB_TODDR_A إلخ).

**الـ Audio Consumers (اللي بيستخدموا الـ reset ده):**
- `sound/soc/meson/axg-toddr.c` — الـ TODDR DMA engine driver.
- `sound/soc/meson/axg-frddr.c` — الـ FRDDR DMA engine driver.
- `sound/soc/meson/axg-fifo.c` — shared FIFO logic للـ audio DMA.

**الـ Device Tree:**
- `arch/arm64/boot/dts/amlogic/` — مكان تعريف `amlogic,meson-axg-audio-arb` node في الـ DTS.

---

### ملخص بجملة واحدة

الـ file ده هو الـ **حارس بوابة** للـ audio DMA في شرائح Amlogic AXG/SM1 — بيتحكم في مين من الـ audio channels يقدر يوصل للـ memory، ومنيمپليمنته كـ reset controller عشان الـ audio drivers يقدروا يفتحوا/يغلقوا الوصول ده بطريقة standard وآمنة.
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

أي hardware block جوه الـ SoC محتاج في لحظة معينة إنك "تعيد تشغيله" — أي تحطه في حالة reset ثم تطلقه. الـ reset مش بس بـ power cycle؛ هو control signal جوه الـ SoC بيوقف الـ block ويرجعه لـ known state.

المشكلة القديمة كانت: كل driver كان بيكتب الـ reset logic بنفسه — بيـmmap الـ register، بيعمل bit manipulation، وبيدير الـ timing بنفسه. النتيجة:

- **code duplication** في كل driver
- مفيش **shared ownership** لو اتنين drivers بيتحكموا في نفس الـ reset line
- مفيش **abstraction** للـ consumers — اللي عايز يعمل reset لازم يعرف عنوان الـ register وأي bit

الـ kernel محتاج layer موحدة تفصل بين:
- **الـ provider**: اللي عنده الـ registers وبيعمل الـ assert/deassert فعلاً
- **الـ consumer**: اللي بيقول "أنا عايز reset لـ block رقم X"

---

### الحل — الـ Reset Controller Framework

الـ kernel حل المشكلة بـ **Reset Controller Subsystem** (موجود في `drivers/reset/`). الفكرة:

1. كل hardware "مزود للـ reset" (زي الـ audio arbiter في ملفنا) بيسجل نفسه كـ **reset controller** عن طريق `reset_controller_dev`.
2. الـ consumer (driver تاني عايز يعمل reset لـ block معين) بيستخدم API زي `reset_control_get()` و`reset_control_assert()`.
3. الـ framework بيترجم الـ ID من الـ device tree لـ bit position حقيقي في الـ register — من غير ما الـ consumer يعرف أي حاجة عن الـ hardware.

---

### Analogy حقيقية — لوحة المفاتيح الكهربائية في المصنع

تخيل مصنع فيه آلات كتير. فيه **لوحة تحكم مركزية** فيها مفاتيح — كل مفتاح بيوقف آلة معينة ويعيد تشغيلها (reset).

| العنصر في الـ analogy | المقابل في الـ kernel |
|---|---|
| لوحة التحكم المركزية (مع الـ register) | `reset_controller_dev` — الـ provider |
| كل مفتاح (bit في الـ register) | reset line واحدة (reset ID) |
| المشرف اللي بيضغط المفتاح | الـ Reset Controller Framework |
| الآلة اللي محتاجة reset | الـ consumer driver |
| رقم الآلة في كتيب التعليمات | الـ ID في الـ device tree |
| كتيب التعليمات نفسه | الـ DT bindings (`amlogic,meson-axg-audio-arb.h`) |

المشرف (الـ framework) هو اللي بيترجم "آلة رقم 3" لـ "المفتاح الفيزيائي على اللوحة". الـ consumer (المهندس) مش محتاج يعرف اللوحة نفسها.

---

### الـ Big Picture — فين الـ subsystem ده في الـ kernel؟

```
┌─────────────────────────────────────────────────────────────┐
│                     Consumer Drivers                        │
│   (audio DMA driver, codec driver, etc.)                    │
│                                                             │
│   reset_control_assert(rstc);                               │
│   reset_control_deassert(rstc);                             │
└────────────────────────┬────────────────────────────────────┘
                         │  Generic Reset API
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Reset Controller Core (kernel/reset.c)         │
│                                                             │
│  - reset_control_get()   → بيبحث عن الـ controller المناسب │
│  - of_xlate()            → بيترجم DT phandle → reset ID    │
│  - reference counting    → shared resets بين أكتر من driver│
└────────────────────────┬────────────────────────────────────┘
                         │  reset_control_ops callbacks
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         Reset Controller Provider Driver                    │
│    (reset-meson-audio-arb.c  ← ملفنا)                      │
│                                                             │
│   .assert()   → بيعمل bit clear في الـ register            │
│   .deassert() → بيعمل bit set في الـ register              │
│   .status()   → بيقرأ الـ bit                              │
└────────────────────────┬────────────────────────────────────┘
                         │  MMIO (readl/writel)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│           Hardware: Audio ARB Register                      │
│                                                             │
│   Bit 31 (ARB_GENERAL): enable الـ arbiter نفسه            │
│   Bit 0-6: enable كل DMA endpoint على حدا                  │
│                                                             │
│   Amlogic AXG / SM1 SoC Audio Subsystem                    │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — ما هو الـ struct المحوري؟

الـ abstraction الأساسية هي `reset_controller_dev`. ده الـ struct اللي بيمثل الـ "مزود للـ reset" من وجهة نظر الـ framework.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops; /* الـ callbacks: assert/deassert/status */
    struct module *owner;
    struct list_head list;               /* بيتحط في global list */
    struct list_head reset_control_head; /* الـ consumers اللي اخدوا handle منه */
    struct device *dev;
    struct device_node *of_node;         /* بيربطه بالـ DT node */
    int of_reset_n_cells;
    int (*of_xlate)(...);                /* DT specifier → reset ID */
    unsigned int nr_resets;             /* عدد الـ reset lines */
};
```

الـ driver بتاعنا بيـembed الـ `reset_controller_dev` جوه `meson_audio_arb_data` — وده الـ pattern المعتاد:

```c
struct meson_audio_arb_data {
    struct reset_controller_dev rstc;   /* لازم يكون الأول أو يتعامل معه بـ container_of */
    void __iomem *regs;                 /* الـ MMIO base address للـ register */
    struct clk *clk;                    /* الـ clock اللي الـ arbiter محتاجه يشتغل */
    const unsigned int *reset_bits;     /* جدول: reset_id → bit_position */
    spinlock_t lock;                    /* حماية الـ read-modify-write */
};
```

#### العلاقة بين الـ structs

```
meson_audio_arb_data
┌─────────────────────────────────┐
│  rstc: reset_controller_dev     │◄── الـ framework بيشوفه بس ده
│  ┌───────────────────────────┐  │
│  │  ops → rstc_ops           │  │
│  │  of_node                  │  │
│  │  nr_resets                │  │
│  └───────────────────────────┘  │
│  regs: void __iomem*            │◄── الـ hardware register
│  clk:  struct clk*             │◄── الـ clock dependency
│  reset_bits: unsigned int[]    │◄── [reset_id] = bit_number
│  lock: spinlock_t              │◄── concurrency protection
└─────────────────────────────────┘

container_of(rcdev, struct meson_audio_arb_data, rstc)
     ↑ بيوصل من الـ rcdev pointer للـ parent struct
```

---

### الـ Reset Line Mapping — ازاي الـ IDs بتتترجم؟

الـ DT bindings بتعرف IDs من 0 لـ 7:

```c
/* من amlogic,meson-axg-audio-arb.h */
#define AXG_ARB_TODDR_A  0   /* To-DDR endpoint A: بيكتب في الـ RAM */
#define AXG_ARB_TODDR_B  1
#define AXG_ARB_TODDR_C  2
#define AXG_ARB_FRDDR_A  3   /* From-DDR endpoint A: بيقرأ من الـ RAM */
/* ... */
```

> **TODDR** = "To DDR" = DMA بيكتب في الـ RAM (capture)
> **FRDDR** = "From DDR" = DMA بيقرأ من الـ RAM (playback)

لكن الـ IDs دي مش هي الـ bit positions في الـ register مباشرة. في AXG:

```c
static const unsigned int axg_audio_arb_reset_bits[] = {
    [AXG_ARB_TODDR_A] = 0,   /* reset_id=0 → bit 0 */
    [AXG_ARB_TODDR_B] = 1,   /* reset_id=1 → bit 1 */
    [AXG_ARB_TODDR_C] = 2,   /* reset_id=2 → bit 2 */
    [AXG_ARB_FRDDR_A] = 4,   /* reset_id=3 → bit 4  ← gap! */
    [AXG_ARB_FRDDR_B] = 5,
    [AXG_ARB_FRDDR_C] = 6,
};
```

لاحظ الـ gap: الـ ID=3 (FRDDR_A) بيتحول لـ bit 4 مش bit 3 — لأن الـ hardware register فيه bit 3 مش مخصص أو محجوز.

في SM1 أضافوا TODDR_D و FRDDR_D:

```c
static const unsigned int sm1_audio_arb_reset_bits[] = {
    /* نفس AXG ... */
    [AXG_ARB_TODDR_D] = 3,   /* bit 3 اللي كان محجوز في AXG */
    [AXG_ARB_FRDDR_D] = 7,
};
```

ده مثال حقيقي على الـ **hardware-specific mapping** اللي الـ framework بيعزله عن الـ consumer.

---

### الـ Assert/Deassert Logic — ليه معكوس؟

```c
static int meson_audio_arb_update(struct reset_controller_dev *rcdev,
                                  unsigned long id, bool assert)
{
    u32 val;
    struct meson_audio_arb_data *arb =
        container_of(rcdev, struct meson_audio_arb_data, rstc);

    spin_lock(&arb->lock);
    val = readl(arb->regs);       /* read current register */

    if (assert)
        val &= ~BIT(arb->reset_bits[id]);  /* assert = clear the bit */
    else
        val |= BIT(arb->reset_bits[id]);   /* deassert = set the bit */

    writel(val, arb->regs);
    spin_unlock(&arb->lock);

    return 0;
}
```

الـ logic معكوس لأن الـ register ده **enable register** مش **reset register**:
- `bit = 1` → الـ DMA endpoint **enabled** (خارج من الـ reset)
- `bit = 0` → الـ DMA endpoint **disabled / in reset**

يعني "assert reset" = "clear the enable bit"، و"deassert reset" = "set the enable bit".

الـ status function بتعمل نفس الـ inversion:

```c
static int meson_audio_arb_status(struct reset_controller_dev *rcdev,
                                  unsigned long id)
{
    u32 val = readl(arb->regs);
    return !(val & BIT(arb->reset_bits[id])); /* 1 = in reset (bit cleared) */
}
```

---

### الـ Probe — ازاي الـ driver بيبدأ؟

```c
static int meson_audio_arb_probe(struct platform_device *pdev)
{
    /* 1. جيب الـ match data (AXG or SM1 table) من الـ of_device_id */
    data = of_device_get_match_data(dev);

    /* 2. allocate الـ private struct */
    arb = devm_kzalloc(dev, sizeof(*arb), GFP_KERNEL);

    /* 3. enable الـ clock — الـ arbiter محتاج clock عشان register access تشتغل */
    arb->clk = devm_clk_get_enabled(dev, NULL);

    /* 4. map الـ register */
    arb->regs = devm_platform_ioremap_resource(pdev, 0);

    /* 5. ابدأ spinlock */
    spin_lock_init(&arb->lock);

    /* 6. وصّل الـ reset_controller_dev بالـ match data */
    arb->reset_bits    = data->reset_bits;
    arb->rstc.nr_resets = data->reset_num;
    arb->rstc.ops      = &meson_audio_arb_rstc_ops;
    arb->rstc.of_node  = dev->of_node;
    arb->rstc.owner    = THIS_MODULE;

    /*
     * 7. اكتب الـ GENERAL bit (bit 31) — ده بيفتح الـ arbiter نفسه
     *    لكن كل الـ DMA endpoints لسه disabled (bits 0-7 = 0)
     *    يعني initial state: arbiter شغال بس مفيش حد بيمشي DMA
     */
    writel(BIT(ARB_GENERAL_BIT), arb->regs);

    /* 8. سجّل الـ controller في الـ framework */
    ret = devm_reset_controller_register(dev, &arb->rstc);
}
```

**الـ Clock Dependency**: الـ arbiter ده بيحتاج clock قبل ما يقدر يوصل للـ registers. ده بيربط الـ Reset subsystem بالـ **Clock subsystem** (CCF - Common Clock Framework). الـ CCF هو الـ subsystem اللي بيدير كل الـ clocks في الـ SoC.

---

### الـ Framework بيمتلك إيه؟ وبيفوّض إيه للـ driver؟

| المسؤولية | الـ Framework (reset core) | الـ Provider Driver (ملفنا) |
|---|---|---|
| تسجيل الـ controller في global list | نعم | لا |
| ترجمة DT phandle → reset ID | نعم (of_xlate) | لا |
| reference counting للـ shared resets | نعم | لا |
| الـ assert/deassert الفعلي على الـ hardware | لا | نعم |
| معرفة أي bit في أي register | لا | نعم |
| الـ clock management | لا | نعم |
| الـ spinlock (حماية الـ register) | لا | نعم |
| الـ remove / cleanup | لا | نعم |

الـ framework هو الـ "وسيط الذكي" — بيعرف مين عايز إيه ومين يقدر يديه. لكن "إزاي تدي الـ reset فعلاً" ده شغل الـ provider driver.

---

### ملخص الـ Data Flow

```
Device Tree:
  audio-controller {
      resets = <&audio_arb AXG_ARB_FRDDR_A>;
  };
          │
          │  reset_control_get(dev, NULL)
          ▼
  Reset Core: of_xlate(AXG_ARB_FRDDR_A=3) → id=3
          │
          │  ops->assert(rcdev, id=3)
          ▼
  meson_audio_arb_assert(rcdev, 3)
          │
          │  reset_bits[3] = 4  (AXG mapping)
          ▼
  val &= ~BIT(4)   →  writel(val, arb->regs)
          │
          ▼
  Hardware Register: bit 4 cleared → FRDDR_A in reset
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Macros والـ Constants والـ Reset IDs

#### Reset Line IDs — من الـ DT-Binding Header

| Macro | القيمة (DT index) | المعنى |
|---|---|---|
| `AXG_ARB_TODDR_A` | 0 | To-DDR channel A (capture) |
| `AXG_ARB_TODDR_B` | 1 | To-DDR channel B |
| `AXG_ARB_TODDR_C` | 2 | To-DDR channel C |
| `AXG_ARB_FRDDR_A` | 3 | From-DDR channel A (playback) |
| `AXG_ARB_FRDDR_B` | 4 | From-DDR channel B |
| `AXG_ARB_FRDDR_C` | 5 | From-DDR channel C |
| `AXG_ARB_TODDR_D` | 6 | To-DDR channel D (SM1 only) |
| `AXG_ARB_FRDDR_D` | 7 | From-DDR channel D (SM1 only) |

> **الـ DT index** (العمود الأول) هو رقم الـ reset في الـ device tree — مش نفس الـ bit في الـ register.

#### Bit Mapping داخل الـ Hardware Register

الجدول ده بيوضح إزاي الـ DT index بيتحول لـ bit رقم كام في الـ register الفعلي:

| DT Reset ID | AXG Bit | SM1 Bit |
|---|---|---|
| `AXG_ARB_TODDR_A` (0) | bit 0 | bit 0 |
| `AXG_ARB_TODDR_B` (1) | bit 1 | bit 1 |
| `AXG_ARB_TODDR_C` (2) | bit 2 | bit 2 |
| `AXG_ARB_TODDR_D` (6) | — | bit 3 |
| `AXG_ARB_FRDDR_A` (3) | bit 4 | bit 4 |
| `AXG_ARB_FRDDR_B` (4) | bit 5 | bit 5 |
| `AXG_ARB_FRDDR_C` (5) | bit 6 | bit 6 |
| `AXG_ARB_FRDDR_D` (7) | — | bit 7 |
| General Enable | bit 31 | bit 31 |

#### الـ Macro الوحيد في الـ Driver نفسه

| Macro | القيمة | المعنى |
|---|---|---|
| `ARB_GENERAL_BIT` | 31 | الـ bit اللي بيفعّل الـ arbiter نفسه في الـ register |

> الـ `ARB_GENERAL_BIT` بيتحط أول ما الـ driver بيعمل probe — من غيره كل الـ channels مش هتشتغل حتى لو عملنا deassert ليها.

#### Reset Semantics — Active Low Logic

| الحالة | قيمة الـ bit في الـ register | معنى للـ channel |
|---|---|---|
| assert (reset فعّال) | **0** | الـ channel متوقف — memory access محجوب |
| deassert (reset منتهي) | **1** | الـ channel شغّال — memory access مفتوح |

الـ driver بيعكس المنطق: `assert → val &= ~BIT(...)` و `deassert → val |= BIT(...)`.

---

### الـ Structs المهمة

#### 1. `struct meson_audio_arb_data`

**الغرض**: الـ struct الرئيسي للـ driver instance — بيحتوي على كل state خاص بالـ device.

```c
struct meson_audio_arb_data {
    struct reset_controller_dev rstc;   // embedded reset controller
    void __iomem *regs;                 // MMIO base address of arbiter register
    struct clk *clk;                    // clock gate — must be enabled before any access
    const unsigned int *reset_bits;     // pointer to bit-map array (axg or sm1)
    spinlock_t lock;                    // protects read-modify-write on regs
};
```

| الـ Field | النوع | الدور |
|---|---|---|
| `rstc` | `struct reset_controller_dev` | الـ embedded struct اللي بيسجّل الـ driver في الـ reset framework — بيتم الوصول له بـ `container_of` |
| `regs` | `void __iomem *` | بوابة الـ MMIO للـ register الوحيد اللي بيتحكم في كل الـ channels |
| `clk` | `struct clk *` | الـ clock اللي لازم يكون شغّال عشان الـ arbiter يشتغل — بيتفتح في الـ probe بـ `devm_clk_get_enabled` |
| `reset_bits` | `const unsigned int *` | مصفوفة بتحوّل الـ DT reset ID لـ bit رقم في الـ register — بتتحدد من الـ match data |
| `lock` | `spinlock_t` | بتحمي الـ read-modify-write على الـ register من الـ race conditions |

**الربط بالـ Structs التانية**: الـ `rstc` field هو نقطة الربط — الـ reset framework بيشتغل مع `reset_controller_dev*` والـ driver بيرجع لـ `meson_audio_arb_data` باستخدام `container_of(rcdev, struct meson_audio_arb_data, rstc)`.

---

#### 2. `struct meson_audio_arb_match_data`

**الغرض**: بيانات ثابتة خاصة بكل SoC — بتتحدد من الـ `of_device_id.data`.

```c
struct meson_audio_arb_match_data {
    const unsigned int *reset_bits;  // bit-map array for this SoC variant
    unsigned int reset_num;          // total number of reset lines supported
};
```

| الـ Field | الدور |
|---|---|
| `reset_bits` | مؤشر على المصفوفة اللي بتربط كل DT index بـ bit position في الـ hardware register |
| `reset_num` | عدد الـ reset lines — بيتحط في `rstc.nr_resets` |

الـ instances الثابتة:
- **`axg_audio_arb_match`**: لـ AXG — 6 channels فقط
- **`sm1_audio_arb_match`**: لـ SM1 — 8 channels (زاد TODDR_D و FRDDR_D)

---

#### 3. `struct reset_controller_dev` (من الـ framework)

**الغرض**: الـ struct القياسي اللي بيمثّل أي reset controller في الـ kernel.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;          // callbacks (assert/deassert/status)
    struct module *owner;                          // THIS_MODULE
    struct list_head list;                         // linked into global reset controller list
    struct list_head reset_control_head;           // list of active reset_control handles
    struct device *dev;                            // back-pointer to device
    struct device_node *of_node;                   // DT node for xlate lookups
    const struct of_phandle_args *of_args;         // used for GPIO-based reset controllers
    int of_reset_n_cells;                          // cells in DT specifier
    int (*of_xlate)(...);                          // ID translation (default = simple)
    unsigned int nr_resets;                        // max reset ID
};
```

الـ driver بيملى منه: `ops`, `owner`, `of_node`, `nr_resets` فقط — الباقي بيتملى من الـ framework في `devm_reset_controller_register`.

---

#### 4. `struct reset_control_ops` (الـ ops table)

**الغرض**: جدول الـ callbacks اللي بيربط الـ framework بالـ driver.

```c
static const struct reset_control_ops meson_audio_arb_rstc_ops = {
    .assert   = meson_audio_arb_assert,    // clear bit → disable channel
    .deassert = meson_audio_arb_deassert,  // set bit → enable channel
    .status   = meson_audio_arb_status,    // read bit → 1 if in reset
};
```

| الـ Callback | الدور | ملاحظة |
|---|---|---|
| `.assert` | بيوقف الـ channel (clear the bit) | بيستدعي `meson_audio_arb_update(..., true)` |
| `.deassert` | بيفتح الـ channel (set the bit) | بيستدعي `meson_audio_arb_update(..., false)` |
| `.status` | بيرجّع حالة الـ reset | 1 = in reset, 0 = deasserted |
| `.reset` | غير موجود | الـ arbiter مش بيدعم self-deasserting reset |

---

### مخطط العلاقات بين الـ Structs (ASCII)

```
platform_device
    │
    │  pdev->dev.driver_data
    ▼
┌────────────────────────────────────────────┐
│         meson_audio_arb_data               │
│                                            │
│  ┌─────────────────────────────────┐       │
│  │   reset_controller_dev  (rstc)  │◄──────┼── container_of() يرجع لنا arb
│  │   .ops ──────────────────────────────────────► reset_control_ops
│  │   .owner = THIS_MODULE          │       │         .assert
│  │   .of_node ──────────────────────────────────► device_node (DT)
│  │   .nr_resets = N               │       │         .deassert
│  └─────────────────────────────────┘       │         .status
│                                            │
│  .regs ──────────────────────────────────────────► MMIO Register (32-bit)
│                                            │        [31][...][7][6][5][4][3][2][1][0]
│  .clk ───────────────────────────────────────────► struct clk (clock gate)
│                                            │
│  .reset_bits ────────────────────────────────────► axg_audio_arb_reset_bits[]
│                                            │        or sm1_audio_arb_reset_bits[]
│  .lock (spinlock_t)                        │        (index = DT reset ID → value = bit pos)
└────────────────────────────────────────────┘

of_device_id.data ────────────────────────────────► meson_audio_arb_match_data
                                                      .reset_bits (→ same arrays above)
                                                      .reset_num
```

---

### مخطط الـ Lifecycle — من الـ probe للـ remove

```
kernel boots / DT match found
         │
         ▼
meson_audio_arb_probe()
    │
    ├─ of_device_get_match_data()
    │       └─ يجيب pointer على axg_audio_arb_match أو sm1_audio_arb_match
    │
    ├─ devm_kzalloc() ──────────────► ينشئ meson_audio_arb_data على الـ heap
    │
    ├─ platform_set_drvdata() ──────► يربط arb بـ pdev
    │
    ├─ devm_clk_get_enabled() ──────► يفتح الـ clock (يفشل لو مش موجود)
    │
    ├─ devm_platform_ioremap_resource() ► يعمل map للـ MMIO register
    │
    ├─ spin_lock_init() ────────────► يهيّئ الـ spinlock
    │
    ├─ يملى arb->rstc fields
    │
    ├─ writel(BIT(31), arb->regs) ──► يفعّل الـ ARB_GENERAL_BIT
    │       (كل الـ channels لسه في reset)
    │
    └─ devm_reset_controller_register()
            └─ يسجّل arb->rstc في الـ global reset controller list
                      │
                      ▼
            [Driver Ready — consumers يقدروا يطلبوا reset]
                      │
         ┌────────────┴────────────┐
         │                         │
    consumer calls            consumer calls
    reset_control_assert()    reset_control_deassert()
         │                         │
         ▼                         ▼
    meson_audio_arb_assert()  meson_audio_arb_deassert()
         │                         │
         └──────────┬──────────────┘
                    ▼
          meson_audio_arb_update()
            spin_lock → readl → modify bit → writel → spin_unlock
                    │
                    ▼
            Hardware Register updated
                    │
         [يفضل شغّال لحد ما يحصل unbind/rmmod]
                    │
                    ▼
meson_audio_arb_remove()
    ├─ spin_lock → writel(0, regs) → spin_unlock
    │       └─ يطفّي كل الـ bits (كل الـ channels ترجع reset)
    └─ [devm cleanup: iounmap, clk_disable, kfree — automatically]
```

---

### مخطط الـ Call Flow — سيناريو deassert

```
Audio subsystem driver
    └─ reset_control_deassert(rstc_handle)
            │
            ▼
    [Reset Framework Core]
    reset_control_deassert()
        └─ rcdev->ops->deassert(rcdev, id)
                │
                ▼
        meson_audio_arb_deassert(rcdev, id)
            └─ meson_audio_arb_update(rcdev, id, false)
                    │
                    ├─ container_of(rcdev, meson_audio_arb_data, rstc) → arb
                    │
                    ├─ spin_lock(&arb->lock)
                    │
                    ├─ val = readl(arb->regs)          // قراءة الـ register الحالي
                    │
                    ├─ bit_pos = arb->reset_bits[id]   // ترجمة ID → bit
                    │
                    ├─ val |= BIT(bit_pos)             // set the bit (deassert = enable)
                    │
                    ├─ writel(val, arb->regs)          // كتابة في الـ hardware
                    │
                    └─ spin_unlock(&arb->lock)
                            │
                            ▼
                    Channel الصوت بقى يقدر يوصل للـ DDR memory
```

#### مخطط assert (العكس)

```
reset_control_assert(rstc_handle)
    └─ rcdev->ops->assert(rcdev, id)
            └─ meson_audio_arb_assert(rcdev, id)
                    └─ meson_audio_arb_update(rcdev, id, true)
                            ├─ spin_lock
                            ├─ val = readl(regs)
                            ├─ val &= ~BIT(reset_bits[id])   // clear the bit (disable)
                            ├─ writel(val, regs)
                            └─ spin_unlock
```

#### مخطط status query

```
reset_control_status(rstc_handle)
    └─ rcdev->ops->status(rcdev, id)
            └─ meson_audio_arb_status(rcdev, id)
                    ├─ val = readl(arb->regs)          // بدون lock (read-only)
                    ├─ bit = val & BIT(reset_bits[id])
                    └─ return !(bit)
                            // 0 = bit set   = deasserted = channel active
                            // 1 = bit clear = asserted   = channel in reset
```

> **ملاحظة**: الـ `status` مش بيعمل `spin_lock` لأنه read-only من register واحد — القراءة atomicبالطبيعة على ARM.

---

### استراتيجية الـ Locking

#### الـ Lock المستخدم

الـ driver فيه **spinlock واحد بس** — `arb->lock` — وده مقصود لأن الـ hardware عبارة عن **register واحد فقط**.

```
┌─────────────────────────────────────────────────────┐
│                   arb->lock (spinlock_t)             │
│                                                      │
│  يحمي: arb->regs (read-modify-write sequence)       │
│                                                      │
│  من يمسكه:                                          │
│    - meson_audio_arb_update() → assert + deassert   │
│    - meson_audio_arb_remove() → writel(0)            │
│                                                      │
│  لا يمسكه:                                          │
│    - meson_audio_arb_status() → read-only           │
└─────────────────────────────────────────────────────┘
```

#### ليه spinlock وملاش mutex؟

| السبب | التفسير |
|---|---|
| الـ reset callbacks ممكن يتنادوا من interrupt context | الـ mutex مش safe في interrupt context |
| العملية بسيطة وسريعة جداً (read-modify-write) | الـ spinlock مناسب للـ critical sections القصيرة |
| الـ register واحد — مفيش blocking I/O | مفيش سبب للـ sleep |

#### سيناريو الـ Race

```
CPU0                              CPU1
────────────────                  ────────────────
deassert(FRDDR_A)                 deassert(TODDR_A)
  spin_lock(&arb->lock) ✓           spin_lock(&arb->lock) ← ينتظر
  val = readl(regs)     = 0x80000000
  val |= BIT(4)         = 0x80000010
  writel(val, regs)
  spin_unlock(&arb->lock)           spin_lock(&arb->lock) ✓
                                    val = readl(regs) = 0x80000010
                                    val |= BIT(0)     = 0x80000011
                                    writel(val, regs)
                                    spin_unlock
```

من غير الـ lock، الـ CPU1 ممكن يقرأ القيمة القديمة ويفقد تغيير الـ CPU0.

#### ترتيب الـ Locks (Lock Ordering)

الـ driver مش بيمسك أي lock تاني غير `arb->lock` — مفيش احتمال deadlock من الداخل.

الـ lock الخارجي الوحيد اللي الـ framework بيمسكه قبل ما يوصل للـ ops هو `reset_control->acquired_mutex` — لكن ده في مستوى أعلى ومش بيتداخل مع الـ spinlock الداخلي.

```
Lock ordering (من الأعلى للأدنى):
  [reset framework internal mutex]  ← framework level
          ↓
  [arb->lock (spinlock)]            ← driver level
          ↓
  [MMIO register access]            ← hardware level
```

---

### ملخص تصميمي

الـ driver ده من أبسط الـ reset controllers في الـ kernel — **register واحد، spinlock واحد، ops ثلاثة**. البساطة دي مقصودة لأن الـ audio arbiter نفسه hardware بسيط: كل channel بيتحكم فيه بـ bit واحد في register واحد، والـ general enable bit (31) هو المفتاح الرئيسي اللي لو اتشال يقطع كل الـ channels مرة واحدة.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs

#### جدول Functions الأساسية

| Function | Category | Caller Context |
|---|---|---|
| `meson_audio_arb_probe` | Registration | kernel → `platform_driver.probe` |
| `meson_audio_arb_remove` | Cleanup | kernel → `platform_driver.remove` |
| `meson_audio_arb_update` | Runtime Core | internal only |
| `meson_audio_arb_assert` | Runtime Ops | reset subsystem |
| `meson_audio_arb_deassert` | Runtime Ops | reset subsystem |
| `meson_audio_arb_status` | Runtime Ops | reset subsystem |

#### جدول الـ Kernel APIs المستخدمة

| API | الـ Subsystem | الغرض |
|---|---|---|
| `devm_kzalloc` | devres | allocate zeroed memory managed by device |
| `devm_clk_get_enabled` | clk | get + prepare + enable clock, devres-managed |
| `devm_platform_ioremap_resource` | platform | map MMIO resource, devres-managed |
| `devm_reset_controller_register` | reset | register reset controller, devres-managed |
| `of_device_get_match_data` | OF | pull match data pointer from DT compatible |
| `platform_get_drvdata` | platform | retrieve driver private data |
| `platform_set_drvdata` | platform | store driver private data |
| `spin_lock_init` | locking | init spinlock |
| `spin_lock / spin_unlock` | locking | protect register R-M-W |
| `readl / writel` | MMIO | 32-bit register read/write |
| `container_of` | kernel macro | recover outer struct from embedded member |
| `BIT(n)` | bitops | produce a bitmask with bit n set |
| `dev_err_probe` | error reporting | log error + return errno atomically |
| `module_platform_driver` | module | register/unregister platform driver via macro |

---

### Category 1: Registration

هدف هذه المجموعة هو تهيئة الـ hardware وتسجيل الـ reset controller مع kernel حتى يقدر الـ consumer drivers يستخدموا الـ resets عن طريق `reset_control_get`.

---

#### `meson_audio_arb_probe`

```c
static int meson_audio_arb_probe(struct platform_device *pdev)
```

**الوظيفة:**
الـ entry point للـ driver — بيتبعت من الـ platform bus لما الـ kernel يلاقي DT node بـ compatible string متطابق. بيعمل allocate لـ `meson_audio_arb_data`، بيحصل على الـ clock والـ MMIO region، بيكتب الـ `ARB_GENERAL_BIT` للـ register عشان يفتح الـ arbiter نفسه (لكن يخلي كل الـ DMA interfaces في reset)، وأخيراً بيسجل الـ reset controller.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `pdev` | `struct platform_device *` | الـ platform device اللي الـ kernel بنى من DT |

**الـ Return Value:**
- `0` → success
- `-EINVAL` → ما فيش match data في الـ DT
- `-ENOMEM` → فشل الـ allocation
- `PTR_ERR(arb->clk)` → فشل الـ clock get/enable
- `PTR_ERR(arb->regs)` → فشل الـ ioremap
- error code من `devm_reset_controller_register` لو فشل التسجيل

**تفاصيل مهمة:**
- كل الـ resources بتتاخد بـ `devm_*` → automatic cleanup لو الـ probe فشل في أي خطوة.
- الـ `devm_clk_get_enabled` بتعمل `clk_get` + `clk_prepare` + `clk_enable` في خطوة واحدة — الـ clock لازم تشتغل عشان يقدر يكتب للـ registers.
- بكتابة `BIT(ARB_GENERAL_BIT)` (bit 31) على الـ register، الـ arbiter hardware يكون active لكن كل الـ DMA channels (bits 0-7) في reset — ده الـ safe initial state.
- لو `devm_reset_controller_register` فشل، الكود بيكال `meson_audio_arb_remove` يدوياً عشان يعمل disable للـ arbiter قبل ما الـ devres يعمل cleanup للباقي.

**Pseudocode Flow:**

```
probe(pdev):
  data = of_device_get_match_data(dev)       // get SoC-specific reset_bits table
  if !data → return -EINVAL

  arb = devm_kzalloc(dev, sizeof(*arb))      // allocate driver state
  if !arb → return -ENOMEM
  platform_set_drvdata(pdev, arb)

  arb->clk = devm_clk_get_enabled(dev, NULL) // get + enable bus clock
  if error → return dev_err_probe(...)

  arb->regs = devm_platform_ioremap_resource(pdev, 0)  // map register bank
  if error → return PTR_ERR(...)

  spin_lock_init(&arb->lock)
  arb->reset_bits = data->reset_bits         // point to SoC bit-map table
  arb->rstc.nr_resets = data->reset_num
  arb->rstc.ops = &meson_audio_arb_rstc_ops
  arb->rstc.of_node = dev->of_node
  arb->rstc.owner = THIS_MODULE

  writel(BIT(31), arb->regs)                // enable arbiter, all DMA in reset

  ret = devm_reset_controller_register(dev, &arb->rstc)
  if ret:
    meson_audio_arb_remove(pdev)            // disable arbiter before devres cleanup
  return ret
```

---

### Category 2: Cleanup

---

#### `meson_audio_arb_remove`

```c
static void meson_audio_arb_remove(struct platform_device *pdev)
```

**الوظيفة:**
بيعمل disable لكل الـ DMA interfaces عن طريق كتابة `0` على الـ arbiter register. ده بيعمل أثر جانبي مهم: بيكتب على bit 31 (الـ `ARB_GENERAL_BIT`) بصفر كمان، يعني الـ arbiter نفسه بيتعطل والـ hardware بيبطل يستقبل أي DMA requests. الـ devres بعدين بتعمل cleanup للـ clock والـ MMIO تلقائياً.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `pdev` | `struct platform_device *` | الـ platform device |

**الـ Return Value:** `void`

**تفاصيل مهمة:**
- بيتاخد الـ `arb` pointer من `platform_get_drvdata` — لازم يكون اتحط مسبقاً في الـ probe.
- الـ spinlock ضروري هنا عشان الـ `writel(0)` لازم تكون atomic بالنسبة لأي `meson_audio_arb_update` جاري في نفس الوقت.
- بيتكال كمان manually من `probe` لو `devm_reset_controller_register` فشل — يعني ده مش cleanup-only function.

---

### Category 3: Runtime Core

---

#### `meson_audio_arb_update`

```c
static int meson_audio_arb_update(struct reset_controller_dev *rcdev,
                                  unsigned long id, bool assert)
```

**الوظيفة:**
الـ internal workhorse — بتنفذ الـ read-modify-write على الـ arbiter register بشكل thread-safe. لما `assert = true`، بتمسح الـ bit المقابل للـ DMA channel (active-low reset logic). لما `assert = false`، بتحط الـ bit (enable channel).

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `rcdev` | `struct reset_controller_dev *` | pointer للـ reset controller، بيتستخدم مع `container_of` للوصول لـ `arb` |
| `id` | `unsigned long` | reset ID — index في جدول `reset_bits[]` |
| `assert` | `bool` | `true` = put in reset، `false` = release from reset |

**الـ Return Value:**
- `0` دايماً — الـ register write مفيهاش failure path.

**تفاصيل مهمة:**
- الـ `container_of(rcdev, struct meson_audio_arb_data, rstc)` بيرجّع الـ outer struct من الـ embedded `rstc` member — pattern أساسي في kernel OOP.
- الـ spinlock بيحمي الـ read-modify-write sequence كاملة عشان يمنع الـ race condition لو اتنين consumer حاولوا يغيروا channels مختلفة في نفس الوقت.
- الـ hardware بيعمل **active-low reset**: bit = 0 → channel في reset، bit = 1 → channel شغال.
- `arb->reset_bits[id]` بيعطي رقم الـ bit الفعلي في الـ register — ده بيفصل الـ logical reset ID عن الـ physical bit position.

**الـ Active-Low Reset Logic:**

```
assert (put in reset):
  val &= ~BIT(reset_bits[id])   // clear bit → channel disabled/reset

deassert (release from reset):
  val |= BIT(reset_bits[id])    // set bit → channel enabled/active
```

**Pseudocode Flow:**

```
update(rcdev, id, assert):
  arb = container_of(rcdev, meson_audio_arb_data, rstc)

  spin_lock(&arb->lock)
    val = readl(arb->regs)           // read current state of ALL channels
    if assert:
      val &= ~BIT(arb->reset_bits[id])  // clear = assert reset
    else:
      val |= BIT(arb->reset_bits[id])   // set = deassert reset
    writel(val, arb->regs)           // write back (preserves other channels)
  spin_unlock(&arb->lock)

  return 0
```

---

### Category 4: Runtime Ops (Reset Controller Callbacks)

هذه المجموعة هي الـ vtable الفعلي اللي الـ reset subsystem بيكال فيه. كل function هي thin wrapper حول `meson_audio_arb_update` أو `meson_audio_arb_status`. الـ reset subsystem core بيكالها من consumer code زي `reset_control_assert(rst)`.

---

#### `meson_audio_arb_assert`

```c
static int meson_audio_arb_assert(struct reset_controller_dev *rcdev,
                                  unsigned long id)
```

**الوظيفة:**
بتعمل assert للـ reset على الـ DMA channel المحدد — يعني بتوقف الـ channel وتحطه في حالة reset. مجرد wrapper بتكال `meson_audio_arb_update(rcdev, id, true)`.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `rcdev` | `struct reset_controller_dev *` | الـ reset controller |
| `id` | `unsigned long` | رقم الـ reset line (مثلاً `AXG_ARB_FRDDR_A`) |

**الـ Return Value:** نفس return value من `meson_audio_arb_update` — دايماً `0`.

**Who calls it:** الـ reset subsystem core عند `reset_control_assert(rst)` من الـ consumer driver.

---

#### `meson_audio_arb_deassert`

```c
static int meson_audio_arb_deassert(struct reset_controller_dev *rcdev,
                                    unsigned long id)
```

**الوظيفة:**
بتعمل deassert للـ reset — يعني بتحرر الـ DMA channel من حالة الـ reset وتخليه يشتغل. مجرد wrapper بتكال `meson_audio_arb_update(rcdev, id, false)`.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `rcdev` | `struct reset_controller_dev *` | الـ reset controller |
| `id` | `unsigned long` | رقم الـ reset line |

**الـ Return Value:** دايماً `0`.

**Who calls it:** الـ reset subsystem core عند `reset_control_deassert(rst)` أو `reset_control_reset(rst)` من الـ consumer driver.

---

#### `meson_audio_arb_status`

```c
static int meson_audio_arb_status(struct reset_controller_dev *rcdev,
                                  unsigned long id)
```

**الوظيفة:**
بتقرأ الـ register وبترجع حالة الـ reset للـ channel المحدد — بدون أي locking لأنها read-only operation على single 32-bit register (atomic على الـ ARM/AArch64). بترجع `1` لو الـ channel في reset، `0` لو شغال.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `rcdev` | `struct reset_controller_dev *` | الـ reset controller |
| `id` | `unsigned long` | رقم الـ reset line |

**الـ Return Value:**
- `1` → channel في reset (bit = 0، active-low logic → `!(val & BIT(...))` = true = 1)
- `0` → channel شغال (bit = 1)

**تفاصيل مهمة:**
- مفيش spinlock هنا — الـ `readl` على 32-bit aligned register أتومي على الـ ARM architecture.
- الـ `!` في `return !(val & BIT(...))` بتعكس الـ active-low hardware logic لتتطابق مع الـ kernel convention اللي بيعتبر `1 = in reset`.
- الـ reset subsystem core بيكالها عند `reset_control_status(rst)`.

---

### الـ `reset_control_ops` Vtable

```c
static const struct reset_control_ops meson_audio_arb_rstc_ops = {
    .assert   = meson_audio_arb_assert,
    .deassert = meson_audio_arb_deassert,
    .status   = meson_audio_arb_status,
};
```

الـ `.reset` callback مش موجود — الـ driver بيدعم فقط الـ explicit assert/deassert model. ده منطقي لأن الـ audio DMA channels محتاجة تتحكم فيها بشكل مستقل وما في حاجة لـ self-deasserting reset pulse.

---

### الـ Data Flow الكامل

```
Consumer Driver
  reset_control_deassert(rst)
        │
        ▼
  Reset Subsystem Core
  (kernel/reset/core.c)
        │ calls ops->deassert(rcdev, id)
        ▼
  meson_audio_arb_deassert(rcdev, id)
        │ calls update(..., false)
        ▼
  meson_audio_arb_update(rcdev, id, false)
        │
        ├─ container_of → arb
        ├─ spin_lock
        ├─ readl(arb->regs)          ← single 32-bit arbiter register
        ├─ val |= BIT(reset_bits[id]) ← set bit = enable DMA channel
        ├─ writel(val, arb->regs)
        └─ spin_unlock
```

---

### ملاحظات على الـ Locking Strategy

| Scenario | Locking |
|---|---|
| `assert` / `deassert` | `spin_lock` / `spin_unlock` — ضروري لـ RMW atomicity |
| `status` | بدون lock — read-only، atomic على ARM |
| `remove` | `spin_lock` — write `0` لازم تكون exclusive |
| `probe` | بدون lock — الـ driver لسه ما اتسجلش، مفيش concurrency |

الـ spinlock اختيار صح لأن الـ reset ops ممكن تتكال من أي context (including IRQ) في بعض الـ drivers — وإن كان الـ audio DMA consumers عادةً process context.
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده بسيط جداً — register واحد بس بيتحكم في الـ memory arbiter الخاص بالـ audio subsystem في Amlogic AXG/SM1. كل الـ reset lines هي bits في نفس الـ register. الـ debugging بالتالي بيتمحور حول:
- قراءة الـ register الواحد ده
- التحقق من الـ clock اللي بيغذيه
- التأكد إن الـ Device Tree صح

---

### Software Level

#### 1. debugfs

الـ reset subsystem بيكشف معلومات عن كل الـ reset controllers المسجلة:

```bash
# اعرض كل الـ reset controllers المتاحة
ls /sys/kernel/debug/reset/

# لو الـ kernel اتبنى بـ CONFIG_RESET_CONTROLLER
cat /sys/kernel/debug/reset/meson-audio-arb-reset/
```

> **ملاحظة**: الـ driver ده مش بيسجل debugfs entries خاصة بيه. الـ entries الموجودة هي اللي بتديها الـ reset core نفسها.

```bash
# شوف الـ reset controls المطلوبة حالياً
cat /sys/kernel/debug/reset/*/requested_resets 2>/dev/null
```

#### 2. sysfs

```bash
# التأكد إن الـ driver اتحمل صح
ls /sys/bus/platform/drivers/meson-audio-arb-reset/

# شوف الـ device المرتبطة بالـ driver
ls /sys/bus/platform/devices/ | grep audio-arb

# معلومات الـ clock المرتبطة بالـ device
cat /sys/kernel/debug/clk/*/clk_enable_count 2>/dev/null | head -20

# شوف state الـ clock اللي بيستخدمها الـ arbiter (اسمه عادةً pclk أو clk_audio)
grep -r "audio" /sys/kernel/debug/clk/ 2>/dev/null
```

#### 3. ftrace — Tracepoints

```bash
# فعّل الـ tracing على reset subsystem
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تابع كل الـ events المتعلقة بالـ reset
echo 'reset:*' > /sys/kernel/debug/tracing/set_event

# أو على مستوى function tracing للـ driver بالتحديد
echo meson_audio_arb_update > /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_audio_arb_status >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer

# ابدأ التسجيل
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**تفسير الـ output:**
```
# بتشوف كل مرة assert أو deassert اتعمل على أي reset line
meson_audio_arb_update() { val = readl(); val &= ~BIT(x); writel(val); }
```

#### 4. printk / dynamic debug

```bash
# فعّل الـ dynamic debug لكل ملفات الـ audio reset driver
echo 'file reset-meson-audio-arb.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّله بـ kernel cmdline
# dyndbg="file reset-meson-audio-arb.c +p"

# شوف الـ messages في dmesg
dmesg | grep -i "audio.*arb\|arb.*reset\|meson-audio-arb"
```

> **تنبيه**: الـ driver الحالي ما فيهوش `dev_dbg()` calls كتير، بس الـ `dev_err_probe()` في الـ probe هيظهر لو في مشكلة في الـ clock.

#### 5. Kernel Config Options

| Option | الوظيفة |
|--------|---------|
| `CONFIG_RESET_CONTROLLER` | الـ reset subsystem الأساسي — لازم يكون enabled |
| `CONFIG_DEBUG_FS` | عشان تشتغل مع `/sys/kernel/debug` |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg()` dynamically |
| `CONFIG_LOCKDEP` | كشف مشاكل الـ spinlock (مهم عشان الـ driver بيستخدم `spin_lock`) |
| `CONFIG_PROVE_LOCKING` | تحليل أعمق لمشاكل الـ locking |
| `CONFIG_DEBUG_SPINLOCK` | كشف spinlock bugs |
| `CONFIG_CLKDEV_LOOKUP` | debugging الـ clock lookup |
| `CONFIG_COMMON_CLK_DEBUG` | معلومات الـ clock في debugfs |
| `CONFIG_OF` | الـ Device Tree parsing — ضروري للـ driver |

#### 6. devlink / أدوات خاصة بالـ subsystem

الـ driver ده بيستخدم الـ **platform bus** العادي، ما فيش devlink. الأدوات المفيدة:

```bash
# شوف كل الـ platform devices
cat /proc/iomem | grep -i audio

# تحقق من الـ resource الـ MMIO المخصص للـ arbiter
cat /proc/iomem | grep -A2 "audio-arb"

# شوف الـ clocks باستخدام clk tool (لو متاح)
cat /sys/kernel/debug/clk/clk_summary | grep -i audio
```

#### 7. رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|------|
| `failed to get clock` | الـ clock المطلوب في الـ DT مش موجود أو فشل `devm_clk_get_enabled()` | تحقق من الـ DT: هل `clocks` property موجودة وصح؟ |
| `failed to register arb reset controller` | `devm_reset_controller_register()` فشل | نادر — ممكن memory issue أو تعارض في الـ of_node |
| `meson-audio-arb-reset: probe failed` | الـ probe رجع error | اتبع الـ error code اللي قبلها في dmesg |
| `-EINVAL` في الـ probe | `of_device_get_match_data()` رجع NULL | الـ compatible string في الـ DT مش متطابقة مع اللي في الـ driver |
| `-ENOMEM` | فشل `devm_kzalloc()` | نادر جداً — مشكلة في الـ system memory |
| `could not get clocks` (من clk framework) | الـ clock provider مش ready | تحقق من الـ clock driver اتحمل قبل الـ audio arb (deferred probe ممكن يحل) |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
static int meson_audio_arb_update(struct reset_controller_dev *rcdev,
                                  unsigned long id, bool assert)
{
    u32 val;
    struct meson_audio_arb_data *arb =
        container_of(rcdev, struct meson_audio_arb_data, rstc);

    /* نقطة مهمة: تحقق إن الـ id مش خارج الحدود */
    WARN_ON(id >= rcdev->nr_resets);

    spin_lock(&arb->lock);
    val = readl(arb->regs);

    /* debugging: اطبع قيمة الـ register قبل وبعد */
    pr_debug("arb: before update id=%lu assert=%d val=0x%08x\n", id, assert, val);

    if (assert)
        val &= ~BIT(arb->reset_bits[id]);
    else
        val |= BIT(arb->reset_bits[id]);

    pr_debug("arb: after  update id=%lu assert=%d val=0x%08x\n", id, assert, val);

    writel(val, arb->regs);
    spin_unlock(&arb->lock);

    return 0;
}
```

**نقطة تانية مهمة في الـ probe:**

```c
/* بعد writel(BIT(ARB_GENERAL_BIT), arb->regs) */
pr_debug("arb: initial register state = 0x%08x\n", readl(arb->regs));
WARN_ON(!(readl(arb->regs) & BIT(ARB_GENERAL_BIT)));
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware متطابقة مع الـ Kernel

الـ arbiter register واحد بس. كل bit بيمثل enable/disable لـ memory interface معين:

| Bit | الـ Interface (AXG) | الـ Interface (SM1) |
|-----|-------------------|-------------------|
| 0   | TODDR_A           | TODDR_A           |
| 1   | TODDR_B           | TODDR_B           |
| 2   | TODDR_C           | TODDR_C           |
| 3   | —                 | TODDR_D           |
| 4   | FRDDR_A           | FRDDR_A           |
| 5   | FRDDR_B           | FRDDR_B           |
| 6   | FRDDR_C           | FRDDR_C           |
| 7   | —                 | FRDDR_D           |
| 31  | ARB_GENERAL       | ARB_GENERAL       |

**القاعدة**: bit = 1 معناها الـ interface شغال (deasserted). bit = 0 معناها reset (asserted).

```bash
# اقرأ الـ register مباشرة من الـ kernel
# أول خطوة: اعرف العنوان من device tree أو /proc/iomem
cat /proc/iomem | grep -i "audio-arb"
```

#### 2. Register Dump باستخدام devmem2 / io

```bash
# مثال: لو الـ register عنده 0xFF632000 (عنوان افتراضي — اتحقق من DT)
# استخدم devmem2 (لازم تبقى root)
devmem2 0xFF632000 w

# أو باستخدام busybox devmem
devmem 0xFF632000

# تفسير النتيجة:
# 0x80000000 = فقط الـ ARB_GENERAL bit شغال، كل الـ DMA interfaces متوقفة
# 0x80000077 = ARB_GENERAL + TODDR_A/B/C + FRDDR_A/B/C كلهم شغالين (AXG)
# 0x800000FF = كل الـ interfaces شغالة (SM1)
```

```bash
# طريقة بديلة باستخدام /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0xFF632000)
    val = struct.unpack('<I', mm.read(4))[0]
    print(f'ARB Register = 0x{val:08X}')
    mm.close()
"
```

**تفسير القراءة:**

```
ARB Register = 0x80000031
→ bit 31 = 1  ← ARB_GENERAL شغال (ضروري)
→ bit  5 = 1  ← FRDDR_B شغال
→ bit  4 = 1  ← FRDDR_A شغال
→ bit  0 = 1  ← TODDR_A شغال
→ باقي الـ bits = 0 ← interfaces تانية في reset
```

#### 3. Logic Analyzer / Oscilloscope

الـ arbiter ده بيتحكم في الـ **AXI/AHB bus access** للـ audio DMA. لو عندك أدوات hardware:

- **Logic Analyzer**: راقب الـ AXI bus signals الخاصة بالـ TODDR/FRDDR
- المشكلة الشائعة هي إن الـ DMA يطلب access بس الـ arbiter ما فتحوش ← هتشوف requests بدون responses
- **JTAG debugger**: باستخدام OpenOCD، ممكن تقرأ الـ register مباشرة من الـ SoC

```
# OpenOCD command لقراءة الـ register
mdw 0xFF632000
```

#### 4. Hardware Issues الشائعة والـ Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | السبب |
|--------------------|--------------------------|-------|
| الـ clock مش شغال | `failed to get clock` → probe fails | الـ audio pll مش initialized |
| الـ register address غلط | kernel panic / data abort | DT address مش متطابق مع الـ SoC |
| الـ ARB_GENERAL bit مش اتبعت | audio DMA يفشل بـ timeout | بعد writel الأولى، الـ bit ما اتكتبش |
| الـ TODDR/FRDDR مش deasserted | audio capture/playback silent | consumer ما طلبش deassert |
| الـ spinlock deadlock | kernel hang / lockdep warning | context خاطئ بيعمل lock |

```bash
# دور على hardware-related errors في الـ log
dmesg | grep -E "abort|fault|timeout|audio.*dma|dma.*audio"
```

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT node موجود وصح
find /proc/device-tree -name "*audio-arb*" -o -name "*audio_arb*" 2>/dev/null

# اقرأ الـ compatible string
cat /proc/device-tree/audio-controller@*/reset-controller@*/compatible 2>/dev/null | xxd

# تحقق من الـ reg property (عنوان الـ register)
cat /proc/device-tree/audio-controller@*/reset-controller@*/reg | xxd

# تحقق من الـ clocks property
ls /proc/device-tree/audio-controller@*/reset-controller@*/

# استخدم dtc لتحويل الـ DTB للـ DTS وافحصه
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 20 "audio-arb"
```

**الـ DT المتوقع للـ AXG:**

```dts
audio_arb: reset-controller@280 {
    compatible = "amlogic,meson-axg-audio-arb";
    reg = <0x0 0x280 0x0 0x4>;   /* register واحد بس */
    clocks = <&clkc_audio AUD_CLKID_DDR_ARB>;
    #reset-cells = <1>;
};
```

**نقاط التحقق:**
- `reg` لازم يشاور على register واحد (size = 4 bytes)
- `clocks` لازم موجود وصح — ده اللي الـ driver بيطلبه بـ `devm_clk_get_enabled(dev, NULL)`
- `#reset-cells = <1>` ضروري عشان الـ consumers يقدروا يطلبوا reset

```bash
# تحقق إن الـ consumers بيستخدموا الـ reset صح
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -B2 -A5 "resets.*ARB\|audio-arb"
```

---

### Practical Commands

#### مجموعة شاملة للـ Debugging

```bash
#!/bin/bash
# === Audio ARB Reset Driver Debug Script ===

echo "=== 1. Driver Load Status ==="
lsmod | grep -i reset || echo "built-in (not module)"
ls /sys/bus/platform/drivers/meson-audio-arb-reset/ 2>/dev/null || echo "Driver not found"

echo ""
echo "=== 2. dmesg — audio arb messages ==="
dmesg | grep -i "audio.*arb\|arb.*audio\|meson-audio-arb" | tail -20

echo ""
echo "=== 3. Clock Status ==="
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null | grep -i "ddr_arb\|audio_arb" || \
  echo "Check CONFIG_COMMON_CLK_DEBUG"

echo ""
echo "=== 4. MMIO Region ==="
cat /proc/iomem | grep -i "audio\|arb" | head -10

echo ""
echo "=== 5. Reset Controller Info ==="
ls /sys/kernel/debug/reset/ 2>/dev/null

echo ""
echo "=== 6. Device Tree Node ==="
find /proc/device-tree -name "*audio-arb*" 2>/dev/null | while read f; do
  echo "Found: $f"
  cat "$f/compatible" 2>/dev/null | tr '\0' '\n'
done
```

```bash
# تفعيل ftrace خطوة بخطوة
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo function > /sys/kernel/debug/tracing/current_tracer
echo meson_audio_arb_update    > /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_audio_arb_status   >> /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_audio_arb_assert   >> /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_audio_arb_deassert >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل audio stream هنا أو اعمل reset من consumer...

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**مثال output من ftrace:**

```
# TASK-PID   CPU#  ||||   TIMESTAMP  FUNCTION
aplay-1234   [001] ....  1234.5678: meson_audio_arb_deassert <-reset_control_deassert
aplay-1234   [001] ....  1234.5679: meson_audio_arb_update <-meson_audio_arb_deassert
```

**التفسير**: `aplay` طلب deassert على reset line — ده معناه إنه بيحاول يفتح الـ FRDDR interface عشان يشتغل. لو مش شايف الـ calls دي، الـ consumer مش بيطلب الـ reset صح.

```bash
# اقرأ الـ register مباشرة (غيّر العنوان حسب الـ SoC)
# AXG: عادةً في audio subsystem base + 0x280
ARB_BASE=$(cat /proc/iomem | grep -i "audio" | head -1 | awk '{print $1}' | cut -d- -f1)
echo "Audio base: 0x$ARB_BASE"
devmem2 $((16#$ARB_BASE + 0x280)) w 2>/dev/null || echo "Install devmem2: apt install devmem2"

# dynamic debug
echo 'file reset-meson-audio-arb.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
dmesg -w &   # راقب في background
# شغّل الـ audio
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Android TV Box بـ Amlogic S905X3 — الصوت مش شغال من الأساس

#### العنوان
Audio bring-up فاشل على S905X3 TV box — لا صوت HDMI ولا analog

#### السياق
شركة بتعمل Android TV box بـ **Amlogic S905X3** (SM1 family). المنتج وصل مرحلة الـ DVT وبدأوا يشغلوا Android. كل حاجة تمام — video، network، إلخ — بس الصوت مش بيطلع خالص، لا من HDMI ولا من الـ 3.5mm jack.

#### المشكلة
الـ `dmesg` بيظهر:

```
meson-audio-arb-reset: failed to get clock
```

الـ driver اتحمل بس فشل في الـ `probe`. وبالتالي كل audio DMA controllers (FRDDR/TODDR) مش قادرين يعملوا reset — لأن الـ reset controller نفسه مش موجود.

#### التحليل
في `meson_audio_arb_probe`:

```c
arb->clk = devm_clk_get_enabled(dev, NULL);
if (IS_ERR(arb->clk))
    return dev_err_probe(dev, PTR_ERR(arb->clk),
                         "failed to get clock\n");
```

الـ driver محتاج clock قبل أي حاجة تانية. لو الـ DT node بتاع `audio-arb` مش مرتبط بـ clock صح، أو الـ clock provider لسه مش probe، هيفشل.

السبب الجذري: الـ DT node بتاع الـ `audio-arb` على SM1 لازم يستخدم compatible `"amlogic,meson-sm1-audio-arb"` مش الـ AXG. الفريق كان copy-paste من board قديمة AXG.

```dts
/* غلط */
audio_arb: reset-controller@...{
    compatible = "amlogic,meson-axg-audio-arb";
    ...
};

/* صح للـ S905X3 */
audio_arb: reset-controller@...{
    compatible = "amlogic,meson-sm1-audio-arb";
    clocks = <&clkc_audio AUD_CLKID_DDR_ARB>;
    clock-names = "ddr_arb";
    ...
};
```

لما الـ compatible غلط، الـ `of_device_get_match_data` بيرجع `NULL` وبيحصل:

```c
data = of_device_get_match_data(dev);
if (!data)
    return -EINVAL;
```

#### الحل
1. صحح الـ compatible في الـ board DTS.
2. تأكد إن الـ `clkc_audio` node موجود وشغال.
3. تحقق من ترتيب الـ probe بـ `deferred probe` mechanism:

```bash
# شوف إيه اللي فاشل في الـ probe
cat /sys/kernel/debug/devices_deferred

# تحقق من الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep ddr_arb
```

#### الدرس المستفاد
الـ `devm_clk_get_enabled` هي أول checkpoint في الـ driver. أي فشل فيها بيوقف كل الـ audio subsystem. لما تعمل porting من AXG لـ SM1، غيّر الـ compatible قبل أي حاجة تانية — الاتنين شبه بصريًا بس بيعتمدوا على `match_data` مختلف.

---

### السيناريو الثاني: Industrial Gateway بـ Amlogic A113D — Audio Capture بيتوقف بشكل عشوائي

#### العنوان
TODDR_A بيتفرز وسط الـ recording في industrial gateway

#### السياق
gateway صناعي بيستخدم **Amlogic A113D** (AXG family) لـ voice capture من microphone array. البروتوكول بيعمل multi-channel audio recording بشكل continuous. بعد ساعات من الشغل، الـ audio stream بيتوقف فجأة ومحتاج reboot.

#### المشكلة
بعد التحقيق، الـ race condition بتحصل في الـ `meson_audio_arb_update`:

```c
static int meson_audio_arb_update(struct reset_controller_dev *rcdev,
                                  unsigned long id, bool assert)
{
    u32 val;
    struct meson_audio_arb_data *arb =
        container_of(rcdev, struct meson_audio_arb_data, rstc);

    spin_lock(&arb->lock);       /* صح — موجودة */
    val = readl(arb->regs);

    if (assert)
        val &= ~BIT(arb->reset_bits[id]);
    else
        val |= BIT(arb->reset_bits[id]);

    writel(val, arb->regs);
    spin_unlock(&arb->lock);

    return 0;
}
```

الـ spinlock موجود وصح. المشكلة مش هنا. المهندس راح يفحص الـ `meson_audio_arb_status`:

```c
static int meson_audio_arb_status(struct reset_controller_dev *rcdev,
                                  unsigned long id)
{
    u32 val;
    struct meson_audio_arb_data *arb =
        container_of(rcdev, struct meson_audio_arb_data, rstc);

    val = readl(arb->regs);      /* قراءة بدون lock! */

    return !(val & BIT(arb->reset_bits[id]));
}
```

الـ `status` بيقرأ الـ register بدون `spin_lock`. لو حصل concurrent assert/deassert من thread تاني، القراءة ممكن تاخد قيمة inconsistent على بعض architectures.

#### التحليل
الـ arbiter register بيتحكم في كل الـ TODDR/FRDDR channels في register واحد. أي write ناقص (partial) أو قراءة في وسط write بيعمل corruption للـ channel state.

في الـ production code، لو audio middleware بيعمل poll على الـ status في loop سريعة مع assert/deassert متوازي، الـ race بيظهر بعد آلاف الـ iterations.

```
Thread A (assert TODDR_A):         Thread B (status check):
  spin_lock()
  val = readl()                      val = readl()  ← بيقرأ قيمة قديمة
  val &= ~BIT(0)
  writel(val)
  spin_unlock()
                                     return !(val & BIT(0))  ← نتيجة غلط
```

#### الحل
المهندس يضيف الـ lock في `status` (لو قدر يعدل الـ upstream code):

```c
static int meson_audio_arb_status(struct reset_controller_dev *rcdev,
                                  unsigned long id)
{
    u32 val;
    unsigned long flags;
    struct meson_audio_arb_data *arb =
        container_of(rcdev, struct meson_audio_arb_data, rstc);

    spin_lock_irqsave(&arb->lock, flags);
    val = readl(arb->regs);
    spin_unlock_irqrestore(&arb->lock, flags);

    return !(val & BIT(arb->reset_bits[id]));
}
```

أو على مستوى الـ userspace middleware: قلل polling rate وأضف retry logic.

```bash
# debug: شوف قيمة الـ register مباشرة
devmem2 0xFF642000 w  # عنوان الـ ARB register على AXG
```

#### الدرس المستفاد
الـ `status` callback في reset controllers غالبًا بيتعمله read-only وبيبقى unlocked — بس لو الـ hardware register مشترك مع write path، الـ lock ضروري حتى في القراءة. في embedded audio systems الـ concurrent access حقيقي وليس theoretical.

---

### السيناريو الثالث: Custom Board Bring-up بـ Amlogic S905X3 — الـ ARB_GENERAL_BIT مش اتبعت

#### العنوان
كل الـ audio DMA channels فاشلة حتى بعد deassert — SMC/ARB general enable منسي

#### السياق
مهندس BSP بيعمل bring-up لـ custom board بـ **S905X3** لتطبيق audio streaming. الـ driver probe نجح، لكن لما أي audio application بتحاول تشتغل، الـ DMA بيتوقف فورًا.

#### المشكلة
المهندس فحص كل حاجة — clocks، resets، DT — وكل حاجة تمام. لكن لقى إن الـ ARB register قيمته `0x00000000` بدل `0x80000000`.

#### التحليل
في `meson_audio_arb_probe` في نهايته:

```c
/*
 * Enable general :
 * In the initial state, all memory interfaces are disabled
 * and the general bit is on
 */
writel(BIT(ARB_GENERAL_BIT), arb->regs);
```

الـ `ARB_GENERAL_BIT` هو bit 31. هذا الـ bit هو الـ master enable للـ arbiter كله. بدونه، حتى لو عملت deassert لأي FRDDR/TODDR، الـ DMA access للـ DDR هيتمنع hardware.

الـ flow الصح:

```
probe():
  1. get clock         → enable DDR ARB clock
  2. ioremap           → access register
  3. writel(BIT(31))   → enable ARB_GENERAL (master enable)
  4. register rstc     → expose reset lines to consumers

consumer (FRDDR_A):
  reset_control_deassert() → val |= BIT(4) → DMA path open
```

المهندس كان يعتقد إن `writel(BIT(31))` ده initialization routine بس — ما أدركش إنه هو نفسه الـ "power on" للـ arbiter كله. لما أعاد قراءة الكود لقى إن الـ `meson_audio_arb_remove` بيعمل:

```c
static void meson_audio_arb_remove(struct platform_device *pdev)
{
    struct meson_audio_arb_data *arb = platform_get_drvdata(pdev);

    /* Disable all access */
    spin_lock(&arb->lock);
    writel(0, arb->regs);     /* بيمسح حتى ARB_GENERAL_BIT */
    spin_unlock(&arb->lock);
}
```

كان عنده leftover من bring-up قديم بيعمل manual `remove` call بدون `probe` — وده خلى الـ register يفضل `0`.

#### الحل

```bash
# تحقق من قيمة الـ register مباشرة
devmem2 0xFF660000 w
# المتوقع: 0x80000000 (bit 31 فقط بعد probe مباشرة)
# بعد deassert FRDDR_A: 0x80000010 (bit 31 + bit 4)

# force reset للـ driver
echo "meson-audio-arb-reset" > /sys/bus/platform/drivers/meson-audio-arb-reset/unbind
echo "meson-audio-arb-reset" > /sys/bus/platform/drivers/meson-audio-arb-reset/bind
```

#### الدرس المستفاد
الـ `ARB_GENERAL_BIT` (bit 31) هو master gate للـ DDR access. مش مجرد initialization — هو شرط ضروري لأي DMA audio. تأكد دايمًا من قيمته بـ `devmem2` في أول خطوة من الـ audio bring-up على أي Amlogic platform.

---

### السيناريو الرابع: Android TV Box بـ S905X3 — الـ FRDDR_D مش بيعمل reset صح

#### العنوان
الـ SM1-specific channels (TODDR_D / FRDDR_D) بيعملوا unexpected behavior

#### السياق
TV box بـ **Amlogic S905X3** بيستخدم 4 FRDDR channels لـ multi-zone audio (living room، bedroom، إلخ). الـ channels من A لـ C شغالة، بس الـ FRDDR_D بيعمل glitch بعد resume من suspend.

#### المشكلة
بعد suspend/resume، الـ FRDDR_D بيطلع صوت distorted. الـ channels التانية تمام.

#### التحليل
المهندس قارن بين الـ two match tables:

```c
/* AXG — 6 channels فقط */
static const unsigned int axg_audio_arb_reset_bits[] = {
    [AXG_ARB_TODDR_A] = 0,
    [AXG_ARB_TODDR_B] = 1,
    [AXG_ARB_TODDR_C] = 2,
    [AXG_ARB_FRDDR_A] = 4,
    [AXG_ARB_FRDDR_B] = 5,
    [AXG_ARB_FRDDR_C] = 6,
};

/* SM1 — 8 channels، بيضيف TODDR_D و FRDDR_D */
static const unsigned int sm1_audio_arb_reset_bits[] = {
    [AXG_ARB_TODDR_A] = 0,
    [AXG_ARB_TODDR_B] = 1,
    [AXG_ARB_TODDR_C] = 2,
    [AXG_ARB_FRDDR_A] = 4,
    [AXG_ARB_FRDDR_B] = 5,
    [AXG_ARB_FRDDR_C] = 6,
    [AXG_ARB_TODDR_D] = 3,   /* bit 3 */
    [AXG_ARB_FRDDR_D] = 7,   /* bit 7 */
};
```

لاحظ: الـ `AXG_ARB_FRDDR_A` في الـ DT header هو index 3، لكن في `sm1_audio_arb_reset_bits` array، الـ index 3 هو `AXG_ARB_TODDR_D` وليس FRDDR_A.

الـ array indexed بـ enum values من الـ header:
```c
#define AXG_ARB_FRDDR_A  3   /* from dt-bindings header */
```

لكن في الـ sm1 array:
```c
[AXG_ARB_FRDDR_A] = 4,   /* bit 4 in hardware */
[AXG_ARB_TODDR_D] = 3,   /* bit 3 in hardware — same index slot as... */
```

الـ `AXG_ARB_FRDDR_A` (index=3 في الـ enum) → bit 4 في الـ hardware.
الـ `AXG_ARB_TODDR_D` (index=6 في الـ enum) → bit 3 في الـ hardware.

لو الـ audio driver بعت reset لـ `AXG_ARB_FRDDR_D` (index=7)، الـ lookup:
```c
val |= BIT(arb->reset_bits[7]);  /* = BIT(7) — صح */
```

المشكلة اتضح إنها في الـ suspend/resume path: الـ audio ASoC driver كان بيعمل assert للـ FRDDR_D بس بينسى يعمل deassert بعد resume. الـ bug مش في الـ arbiter نفسه بل في الـ consumer.

```bash
# تحقق من حالة الـ reset lines بعد resume
cat /sys/kernel/debug/reset/*/  # لو debugfs مفعل
```

#### الحل
في الـ ASoC FRDDR driver، تأكد من sequence الـ resume:

```c
/* في .resume callback */
reset_control_deassert(frddr->reset);  /* لازم يحصل قبل enable */
clk_enable(frddr->clk);
```

#### الدرس المستفاد
الـ SM1 زاد channels جديدة (TODDR_D, FRDDR_D) بـ bits مختلفة عن الـ AXG. تأكد دايمًا من الـ suspend/resume sequence لكل channel بشكل مستقل — الـ arbiter نفسه بيخزن state في hardware register وبيتأثر بالـ power transitions.

---

### السيناريو الخامس: IoT Audio Device بـ Amlogic A113X — الـ module unload بيعمل kernel panic

#### العنوان
`rmmod` لـ audio driver بيسبب kernel panic بسبب orphaned DMA access

#### السياق
IoT device بـ **Amlogic A113X** (AXG) بيشغل voice assistant. الفريق بيعمل field update للـ firmware — بيعمل `rmmod` للـ audio modules ثم `insmod` للنسخة الجديدة. في بعض الأحيان بيحصل kernel panic أثناء الـ `rmmod`.

#### المشكلة
الـ panic trace بيشير لـ access في DMA engine بعد ما الـ arbiter اتشال:

```
BUG: unable to handle kernel paging request at ...
RIP: meson_frddr_dma_interrupt+0x...
```

#### التحليل
ترتيب الـ `meson_audio_arb_remove` هو:

```c
static void meson_audio_arb_remove(struct platform_device *pdev)
{
    struct meson_audio_arb_data *arb = platform_get_drvdata(pdev);

    /* Disable all access */
    spin_lock(&arb->lock);
    writel(0, arb->regs);    /* بيكتب zero — بيوقف الـ DDR access كله */
    spin_unlock(&arb->lock);
}
```

الـ `writel(0, arb->regs)` بيمسح الـ `ARB_GENERAL_BIT` وكل الـ channel bits دفعة واحدة. لو في الـ وقت ده FRDDR_A لسه في نص DMA transaction، الـ hardware بيقطع الـ DDR access فجأة وبيسبب memory fault.

الـ `devm_reset_controller_register` بيعمل unregister تلقائيًا لما الـ device بيتشال — بس الـ consumers (FRDDR drivers) ممكن يكونوا لسه مش اتشالوا.

الـ dependency chain المفروضة:

```
FRDDR_A driver
  └── depends on → audio_arb reset controller
        └── depends on → clkc_audio clock
```

لو الـ `rmmod` sequence مش محترم الـ dependency order، الـ arbiter بيتشال وhardware consumers لسه شغالين.

#### الحل

```bash
# الترتيب الصح للـ rmmod
rmmod snd_soc_meson_axg_frddr   # أول: أوقف الـ consumers
rmmod snd_soc_meson_axg_toddr
rmmod snd_soc_meson_audio_arb    # تاني: أوقف الـ arbiter
```

في الـ DT، وضح الـ dependency بشكل صريح:

```dts
frddr_a: audio-controller@... {
    resets = <&arb AXG_ARB_FRDDR_A>;
    /* الـ kernel reset framework هيضمن الـ ordering */
};
```

وتأكد إن الـ FRDDR driver بيعمل `reset_control_assert` قبل ما يرجع من الـ `remove` callback:

```c
static int meson_frddr_remove(struct platform_device *pdev)
{
    struct meson_frddr *frddr = platform_get_drvdata(pdev);

    reset_control_assert(frddr->reset);   /* أوقف الـ DMA path أولًا */
    /* ثم أوقف باقي الـ resources */
    return 0;
}
```

#### الدرس المستفاد
الـ `writel(0, arb->regs)` في `remove` هو hard shutdown لكل الـ audio DDR access. أي consumer لسه شغال في الوقت ده هيعمل fault. الـ reset framework dependencies في الـ DT مهمة جدًا — وترتيب الـ `rmmod` في field updates لازم يكون documented ومختبر بشكل صريح.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| Reset controller API | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | الـ API الكامل للـ reset controller subsystem — consumer و provider interface |
| Reset controller RST source | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | المصدر الخام للتوثيق |
| DT bindings — audio arb | [kernel.org/doc/Documentation/devicetree/bindings/reset/amlogic,meson-axg-audio-arb.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/reset/amlogic,meson-axg-audio-arb.txt) | الـ Device Tree bindings الرسمية للـ `reset-meson-audio-arb` |
| Bus-Independent Device Accesses | [docs.kernel.org/driver-api/device-io.html](https://docs.kernel.org/driver-api/device-io.html) | `readl` / `writel` و MMIO patterns |
| Locking in the kernel | [docs.kernel.org/locking/spinlocks.html](https://docs.kernel.org/locking/spinlocks.html) | `spin_lock` / `spin_unlock` والـ read-modify-write protection |
| Devres — Managed Device Resource | [kernel.org/doc/html/latest/driver-api/driver-model/devres.html](https://www.kernel.org/doc/html/latest/driver-api/driver-model/devres.html) | `devm_*` API بالكامل بما فيه `devm_reset_controller_register` |
| DRM/Meson VPU documentation | [docs.kernel.org/gpu/meson.html](https://docs.kernel.org/gpu/meson.html) | سياق أوسع لمنظومة Amlogic Meson داخل الـ kernel |

---

### مقالات LWN.net

| المقال | الرابط |
|--------|--------|
| Generic GPIO reset driver (patch discussion) | [lwn.net/Articles/585145/](https://lwn.net/Articles/585145/) |
| Reset controller support for EN7523 SoC | [lwn.net/Articles/1039543/](https://lwn.net/Articles/1039543/) |
| Broadcom STB RESCAL reset controller | [lwn.net/Articles/807076/](https://lwn.net/Articles/807076/) |
| Kernel development — locking article | [lwn.net/Articles/198557/](https://lwn.net/Articles/198557/) |

> **ملحوظة:** مفيش مقال LWN.net مخصص بالكامل لـ `reset-meson-audio-arb`، لكن المقالات فوق بتغطي الـ reset controller framework اللي الـ driver ده بيبني عليه.

---

### نقاشات الـ Mailing List والـ Patches

#### الـ Patch الأصلي — تقديم الـ Driver (2018)

**الـ patch series** اللي Jerome Brunet قدّمه في يوليو 2018 بيعرّف الـ driver لأول مرة:

- [PATCH 0/2 — reset: meson: add audio arb controller](https://lore.kernel.org/all/20180706143122.7612-1-jbrunet@baylibre.com/T/)
- [PATCH 2/2 — reset: meson: add meson audio arb driver](https://lore.kernel.org/all/20180706143122.7612-3-jbrunet@baylibre.com/)
- [v2, 1/2 — reset: meson: add dt-bindings for meson-axg audio arb (Patchwork)](https://patchwork.ozlabs.org/project/devicetree-bindings/patch/20180720152633.6227-2-jbrunet@baylibre.com/)

#### تطور الـ Driver — نقل لـ Reset Tree (2024)

في 2024 Jerome Brunet قدّم patch series ينقل الـ audio reset drivers من الـ CCF (Common Clock Framework) لـ reset tree باستخدام الـ auxiliary bus:

- [PATCH 0/8 — reset: amlogic: move audio reset drivers out of CCF](https://lore.kernel.org/lkml/20240710162526.2341399-1-jbrunet@baylibre.com/T/)
- [LKML — clk: amlogic: axg-audio: use the auxiliary reset driver](https://lkml.org/lkml/2024/7/19/310)

#### الـ Reset Controller Framework الأصلي

- [GIT PULL v2 — Reset controller API — Philipp Zabel (2013)](https://lore.kernel.org/linux-arm-kernel/1365683829.4388.52.camel@pizza.hi.pengutronix.de/)
- [v4,2/8 — reset: Add reset controller API — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/)
- [PATCH 1/7 — reset: add devm_reset_controller_register API — LKML](https://lkml.org/lkml/2016/5/1/56)

#### الـ Shared Reset Control

- [v2,1/3 — phy: amlogic: phy-meson-gxl-usb2: fix shared reset controller use](https://patchwork.kernel.org/project/linux-amlogic/patch/20201201190100.17831-2-aouledameur@baylibre.com/)

---

### مصادر Amlogic / Meson المتخصصة

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| Linux for Amlogic Meson (الموقع الرسمي) | [linux-meson.com](https://linux-meson.com/) | دليل شامل لـ Amlogic SoCs في الـ mainline kernel |
| Kernel mainlining progress | [linux-meson.com/mainlining.html](https://linux-meson.com/mainlining.html) | تتبع حالة الـ drivers في الـ upstream |
| Hardware Support | [linux-meson.com/hardware.html](https://linux-meson.com/hardware.html) | قائمة الـ SoCs المدعومة بما فيها AXG و SM1 |
| Amlogic — eLinux.org | [elinux.org/Amlogic](https://elinux.org/Amlogic) | مرجع مجتمعي لـ Amlogic SoC ecosystem |
| AML Products — eLinux.org | [elinux.org/AML_Products](https://elinux.org/AML_Products) | قائمة منتجات Amlogic المدعومة |
| Elixir — reset-meson-audio-arb.c source | [elixir.bootlin.com/linux/latest/source/drivers/reset/reset-meson-audio-arb.c](https://elixir.bootlin.com/linux/latest/source/drivers/reset/reset-meson-audio-arb.c) | الكود المصدري مع cross-reference كامل |
| cateee.net — CONFIG_RESET_MESON_AUDIO_ARB | [cateee.net/lkddb/web-lkddb/RESET_MESON_AUDIO_ARB.html](https://cateee.net/lkddb/web-lkddb/RESET_MESON_AUDIO_ARB.html) | معلومات الـ Kconfig والـ kernel versions |
| DT bindings (Android kernel mirror) | [android.googlesource.com — amlogic,meson-axg-audio-arb.txt](https://android.googlesource.com/kernel/common/+/refs/heads/android-gs-raviole-5.10-android12-d1/Documentation/devicetree/bindings/reset/amlogic,meson-axg-audio-arb.txt) | نسخة من الـ bindings في Android kernel |
| mjmwired DT bindings mirror | [mjmwired.net — amlogic,meson-axg-audio-arb.txt](https://mjmwired.net/kernel/Documentation/devicetree/bindings/reset/amlogic,meson-axg-audio-arb.txt) | مرجع إضافي للـ bindings |
| BayLibre — Amlogic mainline article | [baylibre.com/improved-amlogic-support-mainline-linux/](https://baylibre.com/improved-amlogic-support-mainline-linux/) | مقال من الشركة صاحبة Jerome Brunet عن جهودهم في الـ upstream |

---

### الكتب الموصى بها

#### Linux Device Drivers (LDD3) — Rubini, Corbet, Kroah-Hartman

الـ chapters الأهم لفهم الـ driver ده:

| الـ Chapter | الموضوع ذو الصلة |
|-------------|-----------------|
| Chapter 3 — Char Drivers | `file_operations` pattern المشابه لـ `reset_control_ops` |
| Chapter 9 — Communicating with Hardware | `readl` / `writel` و MMIO |
| Chapter 5 — Concurrency and Race Conditions | `spinlock` و read-modify-write protection |
| Chapter 14 — The Linux Device Model | `platform_driver`، `of_match_table`، `container_of` |

> متاح مجاناً على [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

| الـ Chapter | الموضوع |
|-------------|---------|
| Chapter 7 — Interrupts and Interrupt Handlers | سياق الـ `spin_lock` |
| Chapter 10 — Kernel Synchronization Methods | `spinlock_t`، `spin_lock_init`، الـ race conditions |
| Chapter 17 — Devices and Modules | `module_platform_driver`، `MODULE_*` macros |

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **Chapter 15 — Kernel Initialization** — فهم الـ SoC boot sequence وعلاقته بالـ reset controllers
- **Chapter 16 — Embedding Linux** — الـ Device Tree و `compatible` strings في بيئات embedded

---

### قراءة إضافية — مقالات متخصصة

| الموضوع | الرابط |
|---------|--------|
| Spinlock tutorial — embetronicx | [embetronicx.com — spinlock in linux kernel](https://embetronicx.com/tutorials/linux/device-drivers/spinlock-in-linux-kernel-1/) |
| Read-Write Spinlock — embetronicx | [embetronicx.com — read write spinlock](https://embetronicx.com/tutorials/linux/device-drivers/read-write-spinlock/) |
| Devres — managed resource allocation (PDF) | [haifux.org/lectures/323/haifux-devres.pdf](http://www.haifux.org/lectures/323/haifux-devres.pdf) |
| STM32 Reset overview | [wiki.st.com/stm32mpu/wiki/Reset_overview](https://wiki.st.com/stm32mpu/wiki/Reset_overview) |
| ZynqMP Reset Controller Driver | [xilinx-wiki — ZynqMP Reset Controller](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842455/Zynqmp+Linux+Reset-controller+Driver) |

---

### مسارات الـ Source Code المهمة في الـ Kernel

```
drivers/reset/reset-meson-audio-arb.c       ← الـ driver نفسه
drivers/reset/core.c                         ← الـ reset framework core
drivers/reset/reset-simple.c                 ← مثال أبسط لـ reset controller
include/linux/reset-controller.h             ← struct reset_controller_dev و reset_control_ops
include/linux/reset.h                        ← الـ consumer API
include/dt-bindings/reset/amlogic,meson-axg-audio-arb.h  ← تعريف الـ reset IDs
Documentation/driver-api/reset.rst           ← التوثيق الرسمي
Documentation/devicetree/bindings/reset/amlogic,meson-axg-audio-arb.txt  ← الـ DT bindings
```

---

### مصطلحات البحث

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
linux kernel reset controller subsystem
reset_controller_dev reset_control_ops
devm_reset_controller_register
amlogic meson axg audio arbiter DDR
meson-audio-arb reset driver
TODDR FRDDR audio DMA linux
container_of embedded struct pattern linux driver
spin_lock read-modify-write MMIO linux
of_device_get_match_data platform driver
devm_clk_get_enabled linux driver
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `meson_audio_arb_update` هي الدالة المحورية اللي بتعمل assert/deassert لأي reset line في الـ audio arbiter. هنعمل **kprobe** عليها عشان نشوف كل مرة بيتعمل فيها reset لـ audio DMA channel — مين طلبه وإيه الـ ID والـ assert flag.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on meson_audio_arb_update()
 * Prints reset ID and assert/deassert direction every time an
 * Amlogic audio-arbiter reset line is toggled.
 */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/reset-controller.h>   /* struct reset_controller_dev */

/* ------------------------------------------------------------------ */
/*  kprobe handler — fires BEFORE meson_audio_arb_update() executes   */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention:
     *   rdi = arg0 → struct reset_controller_dev *rcdev
     *   rsi = arg1 → unsigned long id
     *   rdx = arg2 → bool assert  (0 = deassert, 1 = assert)
     *
     * On ARM64 the same args live in x0, x1, x2.
     * We use regs_get_kernel_argument() which is arch-agnostic.
     */
    unsigned long id     = regs_get_kernel_argument(regs, 1);
    unsigned long assert = regs_get_kernel_argument(regs, 2);

    pr_info("[arb-probe] reset line %lu → %s\n",
            id, assert ? "ASSERT (disabled)" : "DEASSERT (enabled)");

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe struct                                                       */
/* ------------------------------------------------------------------ */
static struct kprobe arb_kp = {
    .symbol_name = "meson_audio_arb_update", /* function to hook      */
    .pre_handler = handler_pre,               /* our callback          */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init arb_probe_init(void)
{
    int ret;

    ret = register_kprobe(&arb_kp);
    if (ret < 0) {
        pr_err("[arb-probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[arb-probe] planted on %s @ %px\n",
            arb_kp.symbol_name, arb_kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit arb_probe_exit(void)
{
    unregister_kprobe(&arb_kp);
    pr_info("[arb-probe] removed\n");
}

module_init(arb_probe_init);
module_exit(arb_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc Example");
MODULE_DESCRIPTION("kprobe on meson_audio_arb_update to trace audio reset toggles");
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/reset-controller.h>
```

**الـ** `kprobes.h` بيجيب `struct kprobe` وكل الـ API اللازم للـ hook.
**الـ** `reset-controller.h` مش ضروري وقت الـ hook نفسه بس بيوضح نوع الـ `rcdev` لو احتجنا نوصله لاحقاً.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

بيتفعّل **قبل** ما الـ function الأصلية تشتغل، فممكن نشوف الـ arguments النظيفة قبل ما تتغير.
**الـ** `regs_get_kernel_argument()` بتجيب الـ argument رقم N بطريقة portable على x86-64 وARM64 من غير ما نكتب كود arch-specific.

---

#### قراءة الـ Arguments

```c
unsigned long id     = regs_get_kernel_argument(regs, 1); /* reset line number */
unsigned long assert = regs_get_kernel_argument(regs, 2); /* 1=assert 0=deassert */
```

الـ argument رقم 0 هو `rcdev` — مش محتاجينه هنا.
الـ `id` بيحدد أنهي DMA channel (TODDR_A, FRDDR_B, إلخ) والـ `assert` بيقول هل بيتعطّل ولا بيتفعّل.

---

#### الـ `struct kprobe`

```c
static struct kprobe arb_kp = {
    .symbol_name = "meson_audio_arb_update",
    .pre_handler = handler_pre,
};
```

الكيرنل بيحوّل الـ `symbol_name` لعنوان حقيقي وقت الـ `register_kprobe`.
لو الدالة `inline` أو محمية بـ `NOKPROBE_SYMBOL` هتفشل الـ registration وده المتوقع.

---

#### الـ `module_init`

```c
ret = register_kprobe(&arb_kp);
```

بيحط **breakpoint** برمجي على أول instruction في `meson_audio_arb_update`.
لو الـ return سالب معناه الدالة مش موجودة أو محمية — بنطبع الـ error ونرجع.

---

#### الـ `module_exit`

```c
unregister_kprobe(&arb_kp);
```

إزالة الـ kprobe **إلزامية** في الـ exit عشان لو مزلناش هيفضل في الكيرنل ويشير لكود اتفشخ — وده kernel panic مضمون.

---

### Makefile للتجربة

```makefile
obj-m += arb_kprobe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod arb_kprobe.ko

# مشاهدة الـ output
sudo dmesg -w | grep arb-probe

# إزالة الـ module
sudo rmmod arb_kprobe
```

---

### مثال على الـ Output المتوقع

```
[arb-probe] planted on meson_audio_arb_update @ ffffffffc08a1230
[arb-probe] reset line 4 → DEASSERT (enabled)
[arb-probe] reset line 4 → ASSERT (disabled)
[arb-probe] reset line 0 → DEASSERT (enabled)
[arb-probe] removed
```

كل سطر بيمثل audio DMA channel بيتفعّل أو بيتعطّل — ده مفيد جداً في debugging مشاكل الـ underrun/overrun في الـ audio pipeline على Amlogic SoCs.
