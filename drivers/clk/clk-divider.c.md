## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

**الـ `clk-divider.c`** جزء من **Common Clock Framework (CCF)** — الـ subsystem الرسمي في Linux kernel المسؤول عن إدارة كل الـ clocks في النظام. موجود في `MAINTAINERS` تحت اسم **COMMON CLK FRAMEWORK**، ومسؤول عنه Michael Turquette و Stephen Boyd، والميلينج ليست بتاعته `linux-clk@vger.kernel.org`.

---

### الصورة الكبيرة — قبل أي كود

#### المشكلة اللي بيحلها

تخيل عندك معالج بيشتغل على 1000 MHz (جيجاهرتز). ده المصدر (الـ **parent clock**). بس في الـ SoC (System on Chip) عندك عشرات الـ devices التانية — USB محتاج 48 MHz، UART محتاج 1.8432 MHz، SPI محتاج 50 MHz.

إزاي تطلع الـ frequencies دي من مصدر واحد بيدي 1000 MHz؟

الإجابة: **بتقسم الـ frequency**.

1000 ÷ 20 = 50 MHz للـ SPI.
1000 ÷ 20.8 ≈ 48 MHz للـ USB.

الـ hardware عنده **register** فيه عدد صغير — لما بتكتب فيه 20 مثلاً، الـ clock بيطلع بسرعة الأب مقسومة على 20. ده بالظبط اللي بيتحكم فيه `clk-divider.c`.

#### القصة بالكامل

تخيل الـ SoC زي مبنى فيه مولد كهرباء رئيسي (الـ PLL) بيولد تردد عالي — مثلاً 1 GHz. في المبنى ده أوضة كثيرة (USB، SPI، I2C، GPU...) كل أوضة محتاجة كهرباء بجهد مختلف.

الـ **clock divider** زي محول كهرباء (transformer) بيأخذ التردد العالي ويقسمه على رقم معين، فتطلع تردد أقل يناسب الجهاز التاني.

الـ kernel لازم:
1. يعرف إيه التردد الحالي؟ ← قرأ الـ register وحسب.
2. يشوف لو حد طلب تردد معين — إيه أقرب قيمة ممكنة؟ ← بيجرب كل الـ dividers الممكنة.
3. يكتب الـ divider الصح في الـ register.

`clk-divider.c` بيعمل الثلاث حاجات دي.

---

### ليه مهم؟

- **توفير الطاقة**: لما جهاز مش شغال، ممكن نبطئ الـ clock بتاعه عن طريق زيادة الـ divider ← أقل استهلاك للطاقة.
- **التوافق مع الـ hardware**: كل SoC عنده طريقة تانية لترميز الـ divider في الـ register (بعضهم بيحسبه `val+1`، بعضهم `2^val`، بعضهم جدول lookup خاص).
- **المرونة**: نفس الكود بيخدم آلاف الـ SoCs المختلفة بفضل الـ flags والـ tables.

---

### الهدف من `clk-divider.c` تحديداً

الملف ده بيوفر **تطبيق generic قابل لإعادة الاستخدام** لـ clock divider — يعني أي driver لأي SoC ممكن يستخدمه بدل ما يكتب كوده من الصفر.

بيوفر:
- قراءة الـ divider الحالي من الـ register وحساب التردد الفعلي (`recalc_rate`).
- إيجاد أفضل divider لتردد مطلوب (`determine_rate` / `round_rate`).
- كتابة الـ divider الجديد في الـ register (`set_rate`).
- تسجيل الـ clock في الـ CCF (`clk_hw_register_divider`).
- دعم الـ `devm_` لتحرير الموارد تلقائياً لما الـ device بيتحذف.

---

### أنواع الـ Divider المدعومة

| الـ Flag | المعنى | مثال |
|---|---|---|
| `CLK_DIVIDER_ONE_BASED` | القيمة في الـ register = الـ divider مباشرة | val=4 → ÷4 |
| default (بدون flag) | الـ divider = val + 1 | val=3 → ÷4 |
| `CLK_DIVIDER_POWER_OF_TWO` | الـ divider = 2^val | val=3 → ÷8 |
| `CLK_DIVIDER_EVEN_INTEGERS` | الـ divider = 2*(val+1) | val=2 → ÷6 |
| `CLK_DIVIDER_MAX_AT_ZERO` | val=0 يعني أكبر divider ممكن | حسب الـ width |
| `CLK_DIVIDER_READ_ONLY` | الـ divider ثابت من الـ bootloader، لا يتغير | — |
| `CLK_DIVIDER_ROUND_CLOSEST` | اختار أقرب تردد (مش دايماً الأقل) | — |
| جدول `clk_div_table` | lookup table: val معين → div معين | أي تناسب عشوائي |

---

### الـ Clock Tree — الصورة العامة

```
[Crystal Oscillator / PLL]  ← مصدر التردد
         |
         v
   [clk-divider]  ← clk-divider.c يتحكم هنا
    (÷2, ÷4, ÷8 ...)
         |
         v
   [clk-gate]   ← يفتح/يقفل الـ clock (clk-gate.c)
         |
         v
   [Device: USB / SPI / I2C ...]
```

الـ CCF بيبني **clock tree** من هذه الـ blocks. `clk-divider.c` هو الـ block المسؤول عن تقليل التردد.

---

### الملفات المهمة في الـ Subsystem

#### الـ Core

| الملف | الدور |
|---|---|
| `drivers/clk/clk.c` | قلب الـ CCF — يدير الـ clock tree كله |
| `drivers/clk/clk-divider.c` | هذا الملف — generic divider |
| `drivers/clk/clk-gate.c` | فتح وإغلاق الـ clock |
| `drivers/clk/clk-mux.c` | اختيار مصدر الـ clock (مين الأب) |
| `drivers/clk/clk-fixed-rate.c` | clock بتردد ثابت لا يتغير |
| `drivers/clk/clk-fixed-factor.c` | divider وmultiplier ثابتان |
| `drivers/clk/clk-composite.c` | يجمع gate + divider + mux في block واحد |
| `drivers/clk/clk-multiplier.c` | عكس الـ divider — يضاعف التردد |
| `drivers/clk/clk-fractional-divider.c` | قسمة كسرية (numerator/denominator) |

#### الـ Headers

| الملف | الدور |
|---|---|
| `include/linux/clk-provider.h` | كل الـ structs والـ ops للـ clock providers — `struct clk_divider`، `struct clk_ops`، flags |
| `include/linux/clk.h` | الـ API للـ clock consumers (الـ devices اللي بتستخدم الـ clocks) |
| `include/linux/clkdev.h` | ربط الـ clocks بالـ devices |

#### مثال على Hardware Drivers تستخدم `clk-divider.c`

| الملف | الـ SoC |
|---|---|
| `drivers/clk/bcm/clk-bcm2835.c` | Raspberry Pi (BCM2835) |
| `drivers/clk/imx/` | NXP i.MX series |
| `drivers/clk/rockchip/` | Rockchip RK3xxx |
| `drivers/clk/samsung/` | Samsung Exynos |
| `drivers/clk/baikal-t1/clk-ccu-div.c` | Baikal-T1 — مثال مباشر لـ divider مخصص |

---

### ملاحظة للقارئ

قبل تقرأ `clk-divider.c` لازم تفهم:
1. **`include/linux/clk-provider.h`** — تعريف `struct clk_divider`، `struct clk_ops`، كل الـ flags.
2. **`drivers/clk/clk.c`** — كيف الـ CCF يدير الـ clock tree ويستدعي الـ ops.
3. **`drivers/clk/clk-gate.c`** و **`drivers/clk/clk-mux.c`** — الـ blocks التانية في نفس الـ tree.
## Phase 2: شرح الـ Clock Divider (CCF) Framework

---

### المشكلة اللي بيحلها الـ subsystem ده

في أي SoC حديث فيه مئات الـ clocks — الـ CPU محتاج 1.2 GHz، الـ UART محتاج 48 MHz، الـ SDIO محتاج 200 MHz، الـ DSP محتاج 600 MHz. الـ hardware مش بيولد كل الـ frequencies دي من صفر، بدل كده بيبدأ من مصدر واحد (oscillator خارجي مثلاً 24 MHz أو PLL بيطلع 1.2 GHz) وبعدين بيقسم التردد ده على أرقام صحيحة عشان يوصل للـ peripherals.

المشكلة إن كل SoC عنده register مختلف لحساب القسمة — بعضهم بيحسب `div = register_value + 1`، وبعضهم `div = 2^register_value`، وبعضهم بيبص في lookup table. لو كل driver كتب logic التحويل ده بنفسه، الكود اتكرر مئات المرات وبقى صعب الصيانة.

**الـ Common Clock Framework (CCF)** — وهو الـ subsystem الأساسي اللي الـ clk-divider بيشتغل جوه — ظهر في kernel 3.4 عشان يعمل abstraction layer موحدة لكل أنواع الـ clocks. الـ `clk-divider.c` هو جزء من الـ CCF بيتخصص تحديداً في النوع اللي بيعمل **قسمة عدد صحيح على التردد**.

---

### الحل — المقاربة اللي بيتبعها الـ kernel

الـ CCF بيعرّف interface موحد (`struct clk_ops`) — كل clock implementation بتعمل struct بـ function pointers جوه. الـ framework بيتعامل مع أي clock عن طريق الـ interface ده بدون ما يعرف التفاصيل التحتانية.

الـ `clk-divider.c` بيوفر:
- **Generic implementation** لـ 3 ops: `recalc_rate` + `determine_rate` + `set_rate`.
- دعم لـ **8 أنماط hardware مختلفة** من خلال flags — نفس الكود بيخدم Rockchip و Allwinner و NXP بدون تعديل.
- **Helper functions مُصدَّرة** (`EXPORT_SYMBOL_GPL`) عشان drivers تانية تبني فوقيها.

---

### تشبيه من الواقع — صندوق التروس

تخيل صندوق تروس في عربية:

| العربية | الـ kernel |
|---|---|
| المحرك (engine) | الـ PLL أو الـ crystal oscillator |
| صندوق التروس | `struct clk_divider` |
| رقم الترس الحالي | قيمة الـ hardware register |
| النسبة (gear ratio) | قيمة الـ divider (مثلاً ÷4) |
| عجلات الدريف | الـ peripheral clock consumer |
| سرعة الدوران الخارجة | `output_rate = parent_rate / div` |
| ECU (كمبيوتر العربية) | الـ CCF core (`drivers/clk/clk.c`) |
| دليل التروس (manual) | `struct clk_div_table` |

لما driver بيطلب تردد معين:
1. الـ ECU (CCF core) بيسأل صندوق التروس (divider) "ايه أحسن ترس عشان توصل للسرعة دي؟"
2. صندوق التروس بيحسب ايه الـ divider المناسب (`determine_rate`).
3. لو المحرك نفسه ممكن يتغير سرعته (CLK_SET_RATE_PARENT)، صندوق التروس بيسأله "ايه أحسن سرعة ممكنة منك؟"
4. بعدين بيكتب الرقم في الـ register (`set_rate`).

الفرق المهم عن التشبيه السطحي: في العربية الترس ثابت في وقت الشغل — في الـ kernel ممكن تغير الـ divider في الـ runtime وانت ماشي، والـ CCF بيضمن ترتيب الخطوات (prepare → enable → set_rate) عشان الـ hardware ما يتكسرش.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Consumer Drivers                              │
│   (USB driver, UART driver, GPU driver, etc.)                   │
│             clk_set_rate() / clk_get_rate()                     │
└────────────────────────┬────────────────────────────────────────┘
                         │  Public clk API (include/linux/clk.h)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              CCF Core  (drivers/clk/clk.c)                      │
│  - Clock tree management        - Rate caching                  │
│  - Prepare/enable ref counting  - Notifiers                     │
│  - debugfs (/sys/kernel/debug/clk/)                             │
└──────┬───────────────┬──────────────────┬───────────────────────┘
       │               │                  │
       ▼               ▼                  ▼
┌──────────┐   ┌──────────────┐   ┌──────────────┐
│clk-fixed │   │ clk-divider  │   │  clk-mux     │
│  -rate.c │   │    .c        │   │  .c          │
│          │   │ (هو ده موضوعنا)│  │              │
└──────────┘   └──────┬───────┘   └──────────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
   ┌─────────────────┐ ┌────────────────────┐
   │ clk_divider_ops │ │ clk_divider_ro_ops │
   │ (read+write)    │ │ (read-only)        │
   └────────┬────────┘ └────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────┐
│         Platform Clock Drivers                │
│  (drivers/clk/rockchip/, clk/sunxi/, etc.)   │
│  بيستخدموا clk_hw_register_divider()         │
│  أو بيعملوا clk_divider_ops جوه clk_ops بتاعتهم│
└───────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────────┐
│              Hardware Registers                │
│   void __iomem *reg  ← MMIO address           │
│   shift=4, width=3 → bits [6:4] في الـ reg    │
└───────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية هي فصل **الـ register encoding** عن **الـ actual divisor**.

في الـ hardware ممكن:
- الـ register value `0` يعني divide by 1 (one-based)
- الـ register value `0` يعني divide by `2^width` (max at zero)
- الـ register value `2` يعني divide by 4 (power of two: `2^2`)
- الـ register value `3` يعني divide by 6 (even integers: `2*(3+1)`)
- الـ register value `3` يعني divide by 5 (lookup table: val=3 → div=5)

الـ `clk-divider.c` بيعمل translation layer:

```
register_value ←──→ actual_divisor
     val       ←──→ div
```

الـ functions `_get_div()` و `_get_val()` هما قلب الـ framework — بيترجموا في الاتجاهين بناءً على الـ flags.

---

### الـ Key Structs وعلاقتها ببعض

```
struct clk_divider                struct clk_div_table
┌──────────────────┐              ┌───────────────────┐
│ struct clk_hw hw │◄──────┐      │ unsigned int val  │
│ void __iomem *reg│      │      │ unsigned int div  │
│ u8 shift         │      │      └───────────────────┘
│ u8 width         │      │        (array, ends with div=0)
│ u16 flags        │      │
│ clk_div_table *table─────────►  [val=0,div=2], [val=1,div=4], ...
│ spinlock_t *lock │      │
└──────────────────┘      │
         │                │
         │  container_of  │
         ▼                │
  struct clk_hw           │
  ┌──────────────┐        │
  │ clk_core *core│       │
  │ clk *clk     │        │
  │ clk_init_data*init────┘ (during registration only)
  └──────────────┘


struct clk_ops (clk_divider_ops)
┌──────────────────────────────────────┐
│ recalc_rate → clk_divider_recalc_rate│
│ determine_rate → clk_divider_determine_rate│
│ set_rate → clk_divider_set_rate      │
│ (enable/disable = NULL → delegate)  │
└──────────────────────────────────────┘


struct clk_rate_request
┌──────────────────────────────┐
│ unsigned long rate           │  ← الـ target rate المطلوب
│ unsigned long min_rate       │
│ unsigned long max_rate       │
│ unsigned long best_parent_rate│ ← الـ CCF بيملأها
│ struct clk_hw *best_parent_hw│ ← الـ parent الأنسب
└──────────────────────────────┘
```

---

### الـ Flag System — أنماط الـ Hardware المختلفة

كل flag بيمثل نوع hardware مختلف:

```c
// Default (no flags): div = register_value + 1
// reg=0 → div=1, reg=1 → div=2, reg=7 → div=8

#define CLK_DIVIDER_ONE_BASED     BIT(0)
// div = register_value (raw)
// reg=1 → div=1, reg=7 → div=7, reg=0 → invalid (unless ALLOW_ZERO)

#define CLK_DIVIDER_POWER_OF_TWO  BIT(1)
// div = 2^register_value
// reg=0 → div=1, reg=1 → div=2, reg=3 → div=8

#define CLK_DIVIDER_ALLOW_ZERO    BIT(2)
// يسمح بـ div=0 (bypass mode في بعض الـ hardware)

#define CLK_DIVIDER_HIWORD_MASK   BIT(3)
// Rockchip-style: الـ bits العلوية (16-31) mask للـ bits السفلية (0-15)
// بيكتب mask+value في register write واحدة

#define CLK_DIVIDER_ROUND_CLOSEST BIT(4)
// بدل ما يعمل ceiling لأقرب divider أعلى، يروح للأقرب (فوق أو تحت)

#define CLK_DIVIDER_READ_ONLY     BIT(5)
// الـ divider اتحدد في البوت أو الـ firmware، مش مسموح للـ kernel يغيره

#define CLK_DIVIDER_MAX_AT_ZERO   BIT(6)
// reg=0 → div = 2^width (القيمة الأكبر مش الأصغر)

#define CLK_DIVIDER_BIG_ENDIAN    BIT(7)
// يستخدم ioread32be/iowrite32be بدل readl/writel

#define CLK_DIVIDER_EVEN_INTEGERS BIT(8)
// div = 2*(register_value + 1)
// reg=0 → div=2, reg=1 → div=4, reg=2 → div=6
```

---

### الـ Rate Calculation Flow — خطوة بخطوة

#### 1. `recalc_rate` — قراءة التردد الحالي من الـ hardware

```c
static unsigned long clk_divider_recalc_rate(struct clk_hw *hw,
        unsigned long parent_rate)
{
    struct clk_divider *divider = to_clk_divider(hw); // container_of

    // اقرأ الـ register وعزل الـ bits المطلوبة
    val = clk_div_readl(divider) >> divider->shift;
    val &= clk_div_mask(divider->width); // mask = (1<<width)-1

    // حوّل الـ register value لـ actual divisor
    return divider_recalc_rate(hw, parent_rate, val,
                               divider->table, divider->flags, divider->width);
}
```

**الـ `to_clk_divider`** بيستخدم `container_of` — لأن الـ `clk_hw` هو أول field في `clk_divider`، فالـ pointer بينفع يتحول لـ pointer على الـ struct الأكبر. ده pattern أساسي في الـ kernel للـ OOP بالـ C.

#### 2. `determine_rate` — إيه أحسن تردد ممكن؟

```c
static int clk_divider_determine_rate(struct clk_hw *hw,
                                      struct clk_rate_request *req)
{
    // لو read-only: مش هنغير الـ divider، احسب التردد الحالي بس
    if (divider->flags & CLK_DIVIDER_READ_ONLY) {
        val = clk_div_readl(divider) >> divider->shift;
        return divider_ro_determine_rate(...);
    }

    // لو قابل للتغيير: ابحث عن أحسن divider
    return divider_determine_rate(hw, req, ...);
}
```

الـ `clk_divider_bestdiv()` هي أعقد function في الملف — بتعمل:

1. لو `CLK_SET_RATE_PARENT` مش مضبوط: الـ parent rate ثابتة، يدور على أقرب divider بس.
2. لو `CLK_SET_RATE_PARENT` مضبوط: بيعمل loop على كل الـ dividers الممكنة، ولكل divider بيسأل الـ parent "ايه أحسن rate ممكنة لو أنا محتاج `rate * div`؟"

```
target = 100 MHz, parent_rate يمكن يتغير

div=1: parent يقدر يعمل 100 MHz → output = 100 MHz ✓
div=2: parent يقدر يعمل 200 MHz → output = 100 MHz ✓
div=3: parent يقدر يعمل 300 MHz → output = 100 MHz ✓
...
بيختار اللي بيديه أدق نتيجة (أقرب لـ target)
```

#### 3. `set_rate` — الكتابة الفعلية في الـ register

```c
static int clk_divider_set_rate(struct clk_hw *hw, unsigned long rate,
                                unsigned long parent_rate)
{
    // احسب الـ register value المناسب
    value = divider_get_val(rate, parent_rate, ...);

    // خد الـ spinlock (atomic context safe)
    if (divider->lock)
        spin_lock_irqsave(divider->lock, flags);

    if (divider->flags & CLK_DIVIDER_HIWORD_MASK) {
        // Rockchip-style: mask في الـ upper 16-bit
        val = clk_div_mask(divider->width) << (divider->shift + 16);
    } else {
        // Standard: اقرأ-عدّل-اكتب (RMW)
        val = clk_div_readl(divider);
        val &= ~(clk_div_mask(divider->width) << divider->shift);
    }
    val |= (u32)value << divider->shift;
    clk_div_writel(divider, val);

    spin_unlock_irqrestore(divider->lock, flags);
    return 0;
}
```

**الـ HIWORD_MASK pattern**: بعض الـ hardware (Rockchip خصوصاً) بيسمح بـ atomic write للـ register بدل read-modify-write — الـ upper 16 bits بتحدد الـ bits اللي هتتغير، والـ lower 16 bits بتحدد القيم. ده بيمنع race conditions بدون spinlock في بعض الحالات.

---

### شرح الـ Register Bit Field

```
32-bit register مثال (width=3, shift=4):

  Bit: 31 ... 16 15 ... 7  6  5  4  3 ... 0
       ┌────────┬─────────┬──┬──┬──┬────────┐
       │  mask  │   ...   │  divider  │ ...  │
       │(hiword)│         │ bits[6:4] │      │
       └────────┴─────────┴──┴──┴──┴────────┘

clk_div_mask(3) = (1<<3)-1 = 0b111 = 7

val = readl(reg) >> 4;   // shift right by 4
val &= 0b111;            // mask to 3 bits → values 0..7
```

---

### مثال عملي — Rockchip RK3399

```c
// من drivers/clk/rockchip/clk-rk3399.c
RK3399_DIV_FLAGS(PCLK_PERILP0, "pclk_perilp0", "hclk_perilp0",
    CLK_SET_RATE_PARENT,
    RK3399_CLKSEL_CON(23), 0, 3,           /* shift=0, width=3 */
    CLK_DIVIDER_HIWORD_MASK, NULL);
```

ده بيعمل divider بـ:
- `reg = RK3399_CLKSEL_CON(23)` (MMIO address)
- `shift = 0`, `width = 3` → bits [2:0]، أقصى divider = 8
- `CLK_DIVIDER_HIWORD_MASK` → الـ upper 16-bit بتحدد الـ mask
- `CLK_SET_RATE_PARENT` → لو محتاج rate مختلفة، يطلب من الـ parent يتغير

---

### ما بيملكه الـ Framework مقابل ما بيفوّضه

#### الـ `clk-divider.c` بيملك:
- حساب الـ `best divider` للـ target rate.
- الـ RMW logic للـ register (read-modify-write).
- الـ flag interpretation (ONE_BASED, POWER_OF_TWO, etc.).
- الـ locking (spinlock) أثناء الكتابة.
- الـ HIWORD_MASK write optimization.
- الـ endianness handling (BE vs LE).

#### بيفوّض للـ platform driver:
- تحديد عنوان الـ register (`void __iomem *reg`).
- تحديد الـ `shift` و `width` المناسبين للـ SoC.
- اختيار الـ flags المناسبة للـ hardware.
- توفير الـ `clk_div_table` لو الـ hardware بيستخدم lookup.
- توفير الـ `spinlock` المشترك مع registers تانية في نفس الـ address.
- الـ enable/disable/prepare (مفيش في الـ divider ops — بيتوارث من الـ parent).

---

### الـ Two ops Variants

```c
// للـ clocks القابلة للتعديل
const struct clk_ops clk_divider_ops = {
    .recalc_rate    = clk_divider_recalc_rate,
    .determine_rate = clk_divider_determine_rate,
    .set_rate       = clk_divider_set_rate,
};

// للـ clocks اللي الـ firmware حددها (read-only)
const struct clk_ops clk_divider_ro_ops = {
    .recalc_rate    = clk_divider_recalc_rate,    // نفسه
    .determine_rate = clk_divider_determine_rate, // بيرجع القيمة الحالية بس
    // .set_rate = NULL → الـ CCF core مش هيحاول يغيرها
};
```

الـ `ro` (read-only) variant مهمة لـ clocks اتحددت من الـ bootloader أو الـ firmware ومش المفروض الـ kernel يلمسها (مثلاً الـ DRAM clock في بعض الأنظمة).

---

### الـ devm Pattern للـ Resource Management

```c
struct clk_hw *__devm_clk_hw_register_divider(struct device *dev, ...)
{
    // خصص pointer يتسجل في الـ devres list بتاع الـ device
    ptr = devres_alloc(devm_clk_hw_release_divider, sizeof(*ptr), GFP_KERNEL);

    hw = __clk_hw_register_divider(...);

    if (!IS_ERR(hw)) {
        *ptr = hw;
        devres_add(dev, ptr);  // الـ kernel هيـ unregister تلقائياً عند device removal
    }
    return hw;
}
```

**الـ devres subsystem**: لما `device_unregister()` بيتكلم، الـ kernel بيمشي على list الـ resources المرتبطة بالـ device ده وبيـ cleanup كل حاجة — مش محتاج الـ driver يتذكر يـ unregister يدوياً. ده بيمنع resource leaks في الـ error paths.

---

### سبب فصل `_get_div` عن `_get_val`

الفصل ده ضروري لأن الـ mapping مش دايماً bijective (one-to-one في الاتجاهين):

- `_get_div(table, val, flags, width)` → يحوّل register value لـ actual divisor
- `_get_val(table, div, flags, width)` → يحوّل actual divisor لـ register value

في الـ `CLK_DIVIDER_MAX_AT_ZERO`:
- `val=0` → `div = 2^width` (مش div=0)
- `div = 2^width` → `val=0` (الـ zero كاد استثنائي)

لو استخدمنا function واحدة، هنحتاج condition في كل اتجاه — الفصل بيوضح النية ويسهل الـ testing.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums — Cheatsheet

#### CLK Framework Flags (في `clk-provider.h`) — بتأثر على الـ core framework

| Flag | Value | المعنى |
|------|-------|---------|
| `CLK_SET_RATE_GATE` | BIT(0) | لازم تعمل gate للـ clock قبل ما تغير الـ rate |
| `CLK_SET_PARENT_GATE` | BIT(1) | لازم تعمل gate قبل ما تغير الـ parent |
| `CLK_SET_RATE_PARENT` | BIT(2) | ارفع طلب تغيير الـ rate للـ parent |
| `CLK_IGNORE_UNUSED` | BIT(3) | متعملش gate حتى لو مش بيستخدمها حد |
| `CLK_GET_RATE_NOCACHE` | BIT(6) | دايما اقرا الـ rate من الـ hardware مباشرة |
| `CLK_SET_RATE_NO_REPARENT` | BIT(7) | متغيرش الـ parent لما تغير الـ rate |
| `CLK_RECALC_NEW_RATES` | BIT(9) | اعيد حساب الـ rates بعد أي notification |
| `CLK_SET_RATE_UNGATE` | BIT(10) | الـ clock محتاج يشتغل عشان تعمل set_rate |
| `CLK_IS_CRITICAL` | BIT(11) | ممنوع تعمله gate أبدا |
| `CLK_OPS_PARENT_ENABLE` | BIT(12) | الـ parent محتاج يكون enabled أثناء gate/ungate/set_rate |
| `CLK_DUTY_CYCLE_PARENT` | BIT(13) | فوّض حسابات الـ duty cycle للـ parent |

#### CLK_DIVIDER Flags — خاصة بالـ divider

| Flag | Value | تأثيره على تحويل الـ register value لـ divider |
|------|-------|------------------------------------------------|
| `CLK_DIVIDER_ONE_BASED` | BIT(0) | الـ div = raw register value (مش +1) |
| `CLK_DIVIDER_POWER_OF_TWO` | BIT(1) | الـ div = 2^val |
| `CLK_DIVIDER_ALLOW_ZERO` | BIT(2) | اسمح بـ divider = 0 (bypass) |
| `CLK_DIVIDER_HIWORD_MASK` | BIT(3) | الـ upper 16-bit فيهم الـ mask، الـ lower 16-bit فيهم الـ value |
| `CLK_DIVIDER_ROUND_CLOSEST` | BIT(4) | قرّب لأقرب قيمة مش لفوق |
| `CLK_DIVIDER_READ_ONLY` | BIT(5) | الـ divider read-only — البرنامج ميقدرش يغيره |
| `CLK_DIVIDER_MAX_AT_ZERO` | BIT(6) | لما val=0 يبقى div = 2^width |
| `CLK_DIVIDER_BIG_ENDIAN` | BIT(7) | استخدم `ioread32be` / `iowrite32be` |
| `CLK_DIVIDER_EVEN_INTEGERS` | BIT(8) | الـ div = 2*(val+1) — أرقام زوجية بس |

#### جدول تحويل val ↔ div حسب الـ flag

| Flag مضبوط | val → div | div → val |
|------------|-----------|-----------|
| DEFAULT (لا شيء) | `val + 1` | `div - 1` |
| `ONE_BASED` | `val` | `div` |
| `POWER_OF_TWO` | `1 << val` | `ffs(div)` |
| `MAX_AT_ZERO` | `val ? val : mask+1` | `div==mask+1 ? 0 : div` |
| `EVEN_INTEGERS` | `2 * (val + 1)` | `(div >> 1) - 1` |
| `table` | lookup بالـ val | lookup بالـ div |

---

### الـ Structs المهمة

#### 1. `struct clk_div_table`

```c
struct clk_div_table {
    unsigned int val;   /* القيمة اللي بتتكتب في الـ register */
    unsigned int div;   /* الـ divider الحقيقي المقابل */
};
```

**الغرض:** mapping صريح بين register values والـ dividers الحقيقية — بتستخدمه الـ hardware اللي ما بتتبعش أي pattern رياضي منتظم. مثلا Rockchip أو Allwinner chips.

**الاستخدام:** array منتهية بـ entry فيها `div = 0` كـ sentinel.

```c
/* مثال: divider يدعم فقط 1, 2, 4, 8 */
static const struct clk_div_table my_div_table[] = {
    { .val = 0, .div = 1 },
    { .val = 1, .div = 2 },
    { .val = 2, .div = 4 },
    { .val = 3, .div = 8 },
    { }  /* sentinel: div=0 يعني نهاية الـ array */
};
```

---

#### 2. `struct clk_divider`

```c
struct clk_divider {
    struct clk_hw            hw;      /* الـ handle اللي بيربط الـ framework بالـ hardware */
    void __iomem            *reg;     /* عنوان الـ MMIO register */
    u8                       shift;   /* بداية الـ bitfield جوه الـ register */
    u8                       width;   /* عدد الـ bits في الـ bitfield */
    u16                      flags;   /* CLK_DIVIDER_* flags */
    const struct clk_div_table *table;/* جدول التحويل، أو NULL لو بيستخدم formula */
    spinlock_t              *lock;    /* يحمي كتابة الـ register */
};
```

| Field | الوظيفة |
|-------|---------|
| `hw` | **لازم يكون أول field** — الـ `container_of` بيعتمد عليه |
| `reg` | عنوان الـ register — بيتقرأ/بيتكتب بـ `readl`/`writel` |
| `shift` | لو الـ divider في bits 8-11 مثلاً، يبقى `shift = 8` |
| `width` | عدد bits الـ divider field، `clk_div_mask(width) = (1<<width)-1` |
| `flags` | بيحدد طريقة التحويل والـ endianness وغيره |
| `table` | لو `NULL` يستخدم الـ formula، غير كده يعمل lookup |
| `lock` | `spinlock_t*` — ممكن يكون `NULL` لو الـ hardware protected بطريقة تانية |

---

#### 3. `struct clk_hw`

```c
struct clk_hw {
    struct clk_core *core;            /* pointer للـ internal core object */
    struct clk      *clk;             /* الـ consumer-facing handle */
    const struct clk_init_data *init; /* بيتبقى NULL بعد التسجيل */
};
```

**الغرض:** الجسر بين الـ hardware-specific struct (زي `clk_divider`) والـ CCF (Common Clock Framework). الـ `clk_hw` بيكون مضمّن (embedded) جوه `clk_divider.hw` والـ `to_clk_divider` بيرجع من الـ `clk_hw` للـ `clk_divider` بـ `container_of`.

---

#### 4. `struct clk_init_data`

```c
struct clk_init_data {
    const char              *name;
    const struct clk_ops    *ops;
    const char * const      *parent_names;  /* \
    const struct clk_parent_data *parent_data; /*  واحد بس من الثلاثة */
    const struct clk_hw    **parent_hws;    /* /
    u8                       num_parents;
    unsigned long            flags;
};
```

**الغرض:** بيتملي مؤقتاً في `__clk_hw_register_divider` على الـ stack، وبيتمرر للـ framework عبر `hw->init`. بعد `clk_hw_register` الـ pointer بيتصفّر.

---

#### 5. `struct clk_rate_request`

```c
struct clk_rate_request {
    struct clk_core *core;
    unsigned long    rate;             /* الـ rate المطلوب — بيتعدّل */
    unsigned long    min_rate;
    unsigned long    max_rate;
    unsigned long    best_parent_rate; /* أفضل rate للـ parent */
    struct clk_hw   *best_parent_hw;  /* أفضل parent */
};
```

**الغرض:** بيتمرر في `determine_rate` callback. الـ divider بيعدّل `rate` و`best_parent_rate` بعد ما يحسب أفضل divisor.

---

#### 6. `struct clk_ops`

الـ vtable الأساسية. الـ divider بيستخدم 3 منها فقط:

```c
const struct clk_ops clk_divider_ops = {
    .recalc_rate    = clk_divider_recalc_rate,
    .determine_rate = clk_divider_determine_rate,
    .set_rate       = clk_divider_set_rate,
};

/* النسخة الـ read-only: بدون set_rate */
const struct clk_ops clk_divider_ro_ops = {
    .recalc_rate    = clk_divider_recalc_rate,
    .determine_rate = clk_divider_determine_rate,
};
```

---

### مخطط العلاقات بين الـ Structs (ASCII)

```
┌─────────────────────────────────────────────────────────────────┐
│                      struct clk_divider                         │
│                                                                 │
│  ┌──────────────┐  ←── embedded (first field)                  │
│  │ struct clk_hw│──────────────────────────────────────────┐   │
│  │   .core ─────┼──► struct clk_core (CCF internal)        │   │
│  │   .clk  ─────┼──► struct clk (consumer handle)          │   │
│  │   .init ─────┼──► struct clk_init_data (temp, stack)    │   │
│  └──────────────┘                                           │   │
│                                                             │   │
│  .reg  ──────────────► void __iomem * (hardware register)  │   │
│  .shift / .width ────► bitfield position/size               │   │
│  .flags ─────────────► CLK_DIVIDER_* bitmask                │   │
│  .lock  ──────────────► spinlock_t * (shared or NULL)       │   │
│  .table ──────────────► struct clk_div_table[]              │   │
│                          { val, div } entries               │   │
│                          terminated by div=0                │   │
└─────────────────────────────────────────────────────────────────┘
        │
        │ to_clk_divider(hw) = container_of(hw, clk_divider, hw)
        │
        ▼
┌──────────────────┐       ┌─────────────────────┐
│  struct clk_hw   │◄──────│  struct clk_divider  │
│  (embedded)      │       │  .hw (first field)   │
└──────────────────┘       └─────────────────────┘
        │
        ▼
┌──────────────────┐
│ struct clk_core  │  (CCF الـ internal — في clk.c)
│  .rate           │
│  .parent ────────┼──► struct clk_core (parent)
│  .children list  │
│  .ops ───────────┼──► struct clk_ops (vtable)
└──────────────────┘

┌────────────────────────────────────────┐
│         struct clk_init_data           │
│  .name                                 │
│  .ops    ────────────► clk_divider_ops │
│  .flags                                │
│  .parent_names / parent_hws            │
│  .num_parents                          │
└────────────────────────────────────────┘
  (بيتعمل على الـ stack في register function
   وبيتصفّر بعد clk_hw_register)

┌────────────────────────────────────────┐
│       struct clk_rate_request          │
│  .rate          ← مطلوب من الـ consumer│
│  .best_parent_rate ← يتعدّل بالـ algo  │
│  .best_parent_hw  ← أفضل parent       │
└────────────────────────────────────────┘
  (يتمرر في determine_rate callback)
```

---

### دورة حياة الـ Divider Clock (Lifecycle Diagram)

```
ALLOCATION & INIT
─────────────────
driver calls __clk_hw_register_divider(dev, np, name, ...)
    │
    ├─► validate CLK_DIVIDER_HIWORD_MASK: width+shift <= 16
    │
    ├─► kzalloc(sizeof(struct clk_divider))
    │     → div->reg, shift, width, flags, lock, table = args
    │     → div->hw.init = &init   (stack-allocated clk_init_data)
    │
    ├─► set init.ops:
    │     CLK_DIVIDER_READ_ONLY? → clk_divider_ro_ops
    │                            → clk_divider_ops
    │
    └─► clk_hw_register(dev, &div->hw)
              │
              ├─► CCF reads hw->init
              ├─► allocates struct clk_core internally
              ├─► sets hw->core, hw->clk
              └─► hw->init = NULL  (cleared after registration)

REGISTRATION COMPLETE
─────────────────────
Consumer can now call: clk_get(), clk_set_rate(), etc.

USAGE
─────
clk_set_rate(clk, rate)
    └─► clk_divider_determine_rate(hw, req)
            └─► clk_divider_set_rate(hw, rate, parent_rate)

TEARDOWN
────────
clk_hw_unregister_divider(hw)
    ├─► clk_hw_unregister(hw)   (CCF removes from tree)
    └─► kfree(div)              (free the struct clk_divider)

    أو (devm variant):
device_release / driver_detach
    └─► devm_clk_hw_release_divider(dev, res)
              └─► clk_hw_unregister_divider(hw)
```

---

### مخطط تدفق الـ Calls — Rate Calculation

#### recalc_rate (قراءة الـ rate الحالي من الـ hardware)

```
consumer: clk_get_rate(clk)
  └─► CCF: clk_core_get_rate_recalc()
        └─► ops->recalc_rate(hw, parent_rate)
              └─► clk_divider_recalc_rate(hw, parent_rate)
                    │
                    ├─► clk_div_readl(divider)
                    │     ├─► CLK_DIVIDER_BIG_ENDIAN? ioread32be(reg)
                    │     └─► else: readl(reg)
                    │
                    ├─► val = (raw_reg >> shift) & clk_div_mask(width)
                    │
                    └─► divider_recalc_rate(hw, parent_rate, val, table, flags, width)
                              │
                              ├─► _get_div(table, val, flags, width)
                              │     ├─► ONE_BASED?     return val
                              │     ├─► POW_OF_TWO?    return 1 << val
                              │     ├─► MAX_AT_ZERO?   return val ? val : mask+1
                              │     ├─► EVEN_INTEGERS? return 2*(val+1)
                              │     ├─► table?         _get_table_div(table, val)
                              │     └─► default:       return val + 1
                              │
                              └─► return DIV_ROUND_UP_ULL(parent_rate, div)
```

#### determine_rate (اختيار أفضل divider لـ rate مطلوب)

```
consumer: clk_set_rate(clk, target_rate)
  └─► CCF: clk_core_determine_rate_nolock()
        └─► ops->determine_rate(hw, req)
              └─► clk_divider_determine_rate(hw, req)
                    │
                    ├─► CLK_DIVIDER_READ_ONLY?
                    │     ├─► قرأ val الحالي من الـ register
                    │     └─► divider_ro_determine_rate(hw, req, table, width, flags, val)
                    │               └─► _get_div() → div ثابت
                    │               └─► CLK_SET_RATE_PARENT?
                    │                     └─► clk_hw_round_rate(parent, rate*div)
                    │
                    └─► divider_determine_rate(hw, req, table, width, flags)
                              └─► clk_divider_bestdiv(hw, parent_hw, rate, &best_parent_rate, ...)
                                    │
                                    ├─► _get_maxdiv(table, width, flags)
                                    │
                                    ├─► CLK_SET_RATE_PARENT NOT set?
                                    │     └─► _div_round(table, parent_rate, rate, flags)
                                    │           ├─► ROUND_CLOSEST? _div_round_closest()
                                    │           └─► else: _div_round_up()
                                    │
                                    └─► CLK_SET_RATE_PARENT set?
                                          └─► loop على كل div من 1..maxdiv
                                                ├─► clk_hw_round_rate(parent, rate*i)
                                                ├─► now = parent_rate / i
                                                └─► _is_best_div(rate, now, best, flags)?
                                                      └─► update bestdiv, best_parent_rate
```

#### set_rate (كتابة الـ divider في الـ hardware)

```
CCF: ops->set_rate(hw, rate, parent_rate)
  └─► clk_divider_set_rate(hw, rate, parent_rate)
        │
        ├─► divider_get_val(rate, parent_rate, table, width, flags)
        │     ├─► div = DIV_ROUND_UP_ULL(parent_rate, rate)
        │     ├─► _is_valid_div(table, div, flags)?  → -EINVAL لو مش valid
        │     └─► _get_val(table, div, flags, width) → register value
        │
        ├─► lock:
        │     divider->lock? spin_lock_irqsave(lock, flags)
        │                  : __acquire(lock)   (annotation فقط)
        │
        ├─► CLK_DIVIDER_HIWORD_MASK?
        │     val = mask << (shift + 16)        (upper 16-bit = write-enable mask)
        │   else:
        │     val = readl(reg)
        │     val &= ~(mask << shift)           (clear الـ bitfield)
        │
        ├─► val |= value << shift               (set القيمة الجديدة)
        ├─► clk_div_writel(divider, val)
        │     ├─► BIG_ENDIAN? iowrite32be(val, reg)
        │     └─► else: writel(val, reg)
        │
        └─► unlock:
              divider->lock? spin_unlock_irqrestore(lock, flags)
                           : __release(lock)
```

---

### استراتيجية الـ Locking

#### المشكلة
الـ `clk_divider_set_rate` بيعمل read-modify-write على الـ register:
```
read reg → clear bits → set new bits → write reg
```
لو thread تاني كتب في نفس الوقت → **race condition** → corruption.

#### الحل: `spinlock_t *lock`

```
┌─────────────────────────────────────────────────────┐
│               Lock Strategy                         │
│                                                     │
│  divider->lock != NULL:                             │
│    spin_lock_irqsave(lock, flags)                   │
│      → read-modify-write register                   │
│    spin_unlock_irqrestore(lock, flags)              │
│                                                     │
│  divider->lock == NULL:                             │
│    __acquire() / __release()  (annotation فقط)     │
│    → hardware بيعمل atomic write بدون lock         │
│      مثال: HIWORD_MASK registers (write-once)       │
└─────────────────────────────────────────────────────┘
```

#### ليه `irqsave`؟
الـ `clk_set_rate` ممكن يتنادى من context عادي أو من interrupt handler في بعض الحالات. الـ `spin_lock_irqsave` بيضمن:
1. مفيش thread تاني يقدر يمسك نفس الـ lock.
2. الـ interrupts متوقفة على الـ CPU الحالية أثناء الكتابة.

#### الـ Lock Ownership
الـ `lock` pointer مش ملك الـ divider — بيتشاركه ممكن مع كل الـ clocks اللي في نفس الـ register bank. الـ driver اللي بيسجّل الـ clock هو اللي بيملك الـ `spinlock_t` وبيمرره لكل `clk_hw_register_divider` في نفس الـ bank.

```
┌──── SoC Clock Controller ──────────────────────────┐
│                                                    │
│   spinlock_t clk_lock;  ← shared lock             │
│                                                    │
│   clk_hw_register_divider(..., &clk_lock)  → div1 │
│   clk_hw_register_divider(..., &clk_lock)  → div2 │
│   clk_hw_register_divider(..., &clk_lock)  → div3 │
│                                                    │
│   كل الـ dividers في نفس الـ bank بيشتركوا في     │
│   نفس الـ lock عشان الـ register قد يكون واحد     │
└────────────────────────────────────────────────────┘
```

#### ترتيب الـ Locks (Lock Ordering)
الـ CCF بيستخدم `prepare_mutex` (sleepable) للعمليات اللي بتنام، و`enable_lock` (spinlock) للعمليات الـ atomic. الـ `divider->lock` بييجي في أدنى مستوى:

```
prepare_mutex  (CCF level — sleepable)
    └─► enable_lock  (CCF level — spinlock, IRQ-safe)
            └─► divider->lock  (hardware level — spinlock, irqsave)
```

لا يجوز أبدا تمسك `divider->lock` وبعدين تحاول تمسك `prepare_mutex` — ده هيعمل deadlock.

---

### ملخص العلاقات النهائية

```
clk_divider
    ├── clk_hw  ←→  clk_core (CCF)
    │                   └──► clk_ops (vtable)
    │                             ├── recalc_rate
    │                             ├── determine_rate
    │                             └── set_rate
    ├── void __iomem *reg   (MMIO)
    ├── clk_div_table[]     (optional lookup table)
    └── spinlock_t *lock    (shared with sibling clocks)

clk_rate_request
    ├── rate              (target / result)
    ├── best_parent_rate  (يتعدّل في determine_rate)
    └── best_parent_hw    (أفضل parent clock)
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Category 1: Register I/O Helpers

| Function | Signature Summary | Purpose |
|---|---|---|
| `clk_div_readl` | `(struct clk_divider*) → u32` | قراءة الـ register مع دعم big-endian |
| `clk_div_writel` | `(struct clk_divider*, u32)` | كتابة الـ register مع دعم big-endian |

#### Category 2: Div/Val Conversion Helpers

| Function | Purpose |
|---|---|
| `_get_table_maxdiv` | أكبر divisor في الـ table |
| `_get_table_mindiv` | أصغر divisor في الـ table |
| `_get_maxdiv` | أكبر divisor بناءً على الـ flags |
| `_get_table_div` | تحويل register val → div عبر table |
| `_get_div` | تحويل val → div حسب flags أو table |
| `_get_table_val` | تحويل div → register val عبر table |
| `_get_val` | تحويل div → register val حسب flags |

#### Category 3: Rate Rounding Helpers

| Function | Purpose |
|---|---|
| `_is_valid_table_div` | التحقق أن الـ div موجود في الـ table |
| `_is_valid_div` | التحقق أن الـ div صالح للـ flags المعطاة |
| `_round_up_table` | أقرب div أكبر أو يساوي في الـ table |
| `_round_down_table` | أقرب div أصغر أو يساوي في الـ table |
| `_div_round_up` | حساب div للـ ceiling |
| `_div_round_closest` | حساب div للأقرب |
| `_div_round` | dispatcher للـ rounding حسب flags |
| `_is_best_div` | مقارنة div جديد بأفضل div سابق |
| `_next_div` | الـ div التالي في التسلسل |
| `clk_divider_bestdiv` | البحث عن أفضل div ممكن |

#### Category 4: Exported Rate API

| Function | Exported | Purpose |
|---|---|---|
| `divider_recalc_rate` | `EXPORT_SYMBOL_GPL` | حساب الـ rate الفعلي من الـ val المقروء |
| `divider_determine_rate` | `EXPORT_SYMBOL_GPL` | تحديد الـ rate المطلوب مع إمكانية تغيير parent |
| `divider_ro_determine_rate` | `EXPORT_SYMBOL_GPL` | نفسه لكن الـ div read-only |
| `divider_round_rate_parent` | `EXPORT_SYMBOL_GPL` | wrapper قديم لـ `divider_determine_rate` |
| `divider_ro_round_rate_parent` | `EXPORT_SYMBOL_GPL` | wrapper قديم لـ `divider_ro_determine_rate` |
| `divider_get_val` | `EXPORT_SYMBOL_GPL` | تحويل rate إلى register value |

#### Category 5: clk_ops Callbacks (static)

| Function | clk_ops slot | Purpose |
|---|---|---|
| `clk_divider_recalc_rate` | `.recalc_rate` | قراءة الـ register وحساب الـ rate |
| `clk_divider_determine_rate` | `.determine_rate` | dispatcher للـ RO وغير RO |
| `clk_divider_set_rate` | `.set_rate` | كتابة الـ div value في الـ register |

#### Category 6: Registration & Cleanup

| Function | Exported | Purpose |
|---|---|---|
| `__clk_hw_register_divider` | `EXPORT_SYMBOL_GPL` | التسجيل الفعلي لـ clk_divider |
| `__devm_clk_hw_register_divider` | `EXPORT_SYMBOL_GPL` | نسخة devres مع auto-cleanup |
| `clk_register_divider_table` | `EXPORT_SYMBOL_GPL` | legacy API بـ `struct clk*` |
| `clk_unregister_divider` | `EXPORT_SYMBOL_GPL` | legacy unregister |
| `clk_hw_unregister_divider` | `EXPORT_SYMBOL_GPL` | unregister وتحرير الـ memory |
| `devm_clk_hw_release_divider` | internal devres | cleanup callback تلقائي |

---

### Category 1: Register I/O Helpers

الـ `clk_divider` بيعمل read/write على MMIO register، لازم يدعم little-endian وbig-endian حسب الـ `CLK_DIVIDER_BIG_ENDIAN` flag.

---

#### `clk_div_readl`

```c
static inline u32 clk_div_readl(struct clk_divider *divider)
```

**بتعمل إيه:** بتقرأ الـ 32-bit register اللي فيه الـ divider value. لو الـ flag `CLK_DIVIDER_BIG_ENDIAN` متضبط بتستخدم `ioread32be`، غير كده `readl` العادية.

**Parameters:**
- `divider` — pointer للـ `struct clk_divider` اللي فيه عنوان الـ register والـ flags

**Return:** الـ raw u32 من الـ register

**Key details:** كل reads على الـ MMIO يحتاجوا `readl`/`ioread32be` عشان الـ memory barriers الصح. الفرق بينهم هو byte-swap.

---

#### `clk_div_writel`

```c
static inline void clk_div_writel(struct clk_divider *divider, u32 val)
```

**بتعمل إيه:** بتكتب الـ val في الـ register مع مراعاة الـ endianness تماماً زي الـ `clk_div_readl`.

**Parameters:**
- `divider` — الـ clk_divider descriptor
- `val` — الـ 32-bit value اللي هيتكتب

**Return:** void

**Key details:** لازم يتعمل وهو شايل الـ spinlock لو الـ caller هو `clk_divider_set_rate`.

---

### Category 2: Div/Val Conversion Helpers

الـ divider register ممكن يعمل بأكتر من encoding. مثلاً:
- `CLK_DIVIDER_ONE_BASED`: الـ register value = الـ div value مباشرة (÷1 = reg 1)
- الـ default: div = reg + 1 (÷1 = reg 0)
- `CLK_DIVIDER_POWER_OF_TWO`: div = 1 << reg
- `CLK_DIVIDER_EVEN_INTEGERS`: div = 2 × (reg + 1)
- `CLK_DIVIDER_MAX_AT_ZERO`: reg=0 يعني أكبر div ممكن
- Table-based: lookup table صريحة

---

#### `_get_table_maxdiv`

```c
static unsigned int _get_table_maxdiv(const struct clk_div_table *table, u8 width)
```

**بتعمل إيه:** بتمشي على الـ `clk_div_table` وترجع أكبر `div` موجود فيها بشرط أن الـ `val` يناسب الـ register width.

**Parameters:**
- `table` — array من `{val, div}` أخرها `{0, 0}`
- `width` — عدد bits في الـ divider field

**Return:** أكبر div صالح، أو 0 لو الـ table فاضية

---

#### `_get_table_mindiv`

```c
static unsigned int _get_table_mindiv(const struct clk_div_table *table)
```

**بتعمل إيه:** بترجع أصغر div في الـ table، بتبدأ بـ `UINT_MAX` وبتقارن.

**Parameters:**
- `table` — الـ div table

**Return:** أصغر div

---

#### `_get_maxdiv`

```c
static unsigned int _get_maxdiv(const struct clk_div_table *table, u8 width,
                                unsigned long flags)
```

**بتعمل إيه:** dispatcher بيحسب أكبر divisor ممكن حسب الـ encoding المستخدم. لو `ONE_BASED` يرجع `mask`، لو `POWER_OF_TWO` يرجع `1 << mask`، لو `EVEN_INTEGERS` يرجع `2*(mask+1)`، لو table يستخدم `_get_table_maxdiv`، غير كده `mask + 1`.

**Parameters:**
- `table` — optional lookup table
- `width` — عرض الـ bit field
- `flags` — `CLK_DIVIDER_*` flags

**Return:** الـ maximum divisor

---

#### `_get_div`

```c
static unsigned int _get_div(const struct clk_div_table *table,
                             unsigned int val, unsigned long flags, u8 width)
```

**بتعمل إيه:** بتحول الـ raw register value لـ divisor حقيقي حسب الـ encoding. ده الـ core conversion function.

**Parameters:**
- `table` — lookup table (optional)
- `val` — الـ raw value من الـ register بعد mask وshift
- `flags` — الـ CLK_DIVIDER flags
- `width` — عرض الـ field (للـ MAX_AT_ZERO)

**Return:** الـ divisor الفعلي

**Logic:**
```
if ONE_BASED:    return val
if POW2:         return 1 << val
if MAX_AT_ZERO:  return val ? val : mask+1
if EVEN:         return 2*(val+1)
if table:        lookup table
default:         return val + 1
```

---

#### `_get_val`

```c
static unsigned int _get_val(const struct clk_div_table *table,
                             unsigned int div, unsigned long flags, u8 width)
```

**بتعمل إيه:** العكس تماماً من `_get_div` — بتحول divisor لـ register value.

**Parameters:**
- `table` — lookup table
- `div` — الـ divisor المطلوب
- `flags` — الـ CLK_DIVIDER flags
- `width` — للـ MAX_AT_ZERO

**Return:** الـ value اللي يتكتب في الـ register

---

#### `_get_table_div` و `_get_table_val`

```c
static unsigned int _get_table_div(const struct clk_div_table *table, unsigned int val)
static unsigned int _get_table_val(const struct clk_div_table *table, unsigned int div)
```

**بيعملوا إيه:** linear search في الـ table — الأول يبحث بالـ val ويرجع الـ div، والتاني العكس. يرجعوا 0 لو مش موجود.

---

### Category 3: Rate Rounding Helpers

لما الـ CCF بيطلب rate جديد، الـ divider لازم يحسب أقرب divisor يحقق الـ rate المطلوب. فيه طريقتين: ceiling (الأعلى) أو closest (الأقرب).

---

#### `_is_valid_div`

```c
static bool _is_valid_div(const struct clk_div_table *table, unsigned int div,
                          unsigned long flags)
```

**بتعمل إيه:** بتتحقق أن الـ div صالح للـ hardware. لو `POWER_OF_TWO` لازم يكون power of 2. لو table لازم يكون فيها. غير كده كل قيمة صالحة.

---

#### `_round_up_table` و `_round_down_table`

```c
static int _round_up_table(const struct clk_div_table *table, int div)
static int _round_down_table(const struct clk_div_table *table, int div)
```

**بيعملوا إيه:** بيدوروا في الـ table على أقرب div أكبر-أو-يساوي (up) أو أصغر-أو-يساوي (down) من الـ div المطلوب.

**Key details:** `_round_up_table` بيبدأ بـ `INT_MAX` ويضيق، و`_round_down_table` بيبدأ بـ `mindiv`.

---

#### `_div_round_up`

```c
static int _div_round_up(const struct clk_div_table *table,
                         unsigned long parent_rate, unsigned long rate,
                         unsigned long flags)
```

**بتعمل إيه:** بتحسب الـ div بطريقة ceiling: `div = ceil(parent_rate / rate)` ثم بتقربه لأقرب قيمة صالحة (power of 2 أو table).

**Return:** الـ div ceiling

---

#### `_div_round_closest`

```c
static int _div_round_closest(const struct clk_div_table *table,
                              unsigned long parent_rate, unsigned long rate,
                              unsigned long flags)
```

**بتعمل إيه:** بتحسب div فوق وتحت ثم بتختار الأقرب للـ rate المطلوب عن طريق مقارنة `|rate - up_rate|` بـ `|rate - down_rate|`.

**Pseudocode:**
```
up   = ceil(parent_rate / rate)  → snap to valid
down = floor(parent_rate / rate) → snap to valid
up_rate   = ceil(parent_rate / up)
down_rate = ceil(parent_rate / down)
return (rate - up_rate) <= (down_rate - rate) ? up : down
```

---

#### `_div_round`

```c
static int _div_round(const struct clk_div_table *table,
                      unsigned long parent_rate, unsigned long rate,
                      unsigned long flags)
```

**بتعمل إيه:** dispatcher بسيط — لو `CLK_DIVIDER_ROUND_CLOSEST` يستدعي `_div_round_closest`، غير كده `_div_round_up`.

---

#### `_is_best_div`

```c
static bool _is_best_div(unsigned long rate, unsigned long now,
                         unsigned long best, unsigned long flags)
```

**بتعمل إيه:** بتحدد لو الـ `now` أفضل من الـ `best` الحالي. لو `ROUND_CLOSEST` بتستخدم absolute difference، غير كده بتتأكد أن `now <= rate` و `now > best` (ceiling behavior).

---

#### `_next_div`

```c
static int _next_div(const struct clk_div_table *table, int div,
                     unsigned long flags)
```

**بتعمل إيه:** بترجع الـ div التالي في التسلسل. لو `POWER_OF_TWO` بتعمل `roundup_pow_of_two(div+1)`، لو table بتستخدم `_round_up_table`، غير كده `div+1`.

**Who calls it:** `clk_divider_bestdiv` في الـ loop بتاعته.

---

#### `clk_divider_bestdiv`

```c
static int clk_divider_bestdiv(struct clk_hw *hw, struct clk_hw *parent,
                               unsigned long rate,
                               unsigned long *best_parent_rate,
                               const struct clk_div_table *table, u8 width,
                               unsigned long flags)
```

**بتعمل إيه:** ده قلب الـ rate determination. بتدور على أفضل div يحقق الـ rate المطلوب. لو الـ clock مش بيسمح بتغيير الـ parent rate (`CLK_SET_RATE_PARENT` مش موضوع) بترجع على طول نتيجة `_div_round`. لو مسموح بتغيير الـ parent rate، بتعمل exhaustive search على كل div ممكن.

**Parameters:**
- `hw` — الـ clock hardware handle
- `parent` — الـ parent hw (للـ rounding)
- `rate` — الـ target rate
- `best_parent_rate` — input/output: الـ parent rate الحالي والأفضل
- `table`, `width`, `flags` — خصائص الـ divider

**Return:** الـ best divisor

**Pseudocode:**
```
if rate == 0: rate = 1
maxdiv = _get_maxdiv(...)

if !CLK_SET_RATE_PARENT:
    bestdiv = _div_round(parent_rate, rate)
    clamp(bestdiv, 1, maxdiv)
    return bestdiv

maxdiv = min(ULONG_MAX/rate, maxdiv)  /* overflow guard */

for i = first_valid_div; i <= maxdiv; i = _next_div(i):
    if rate * i == saved_parent_rate:
        return i   /* perfect match, no parent change needed */
    parent_rate = clk_hw_round_rate(parent, rate * i)
    now = ceil(parent_rate / i)
    if _is_best_div(rate, now, best):
        bestdiv = i; best = now
        *best_parent_rate = parent_rate

if !bestdiv:
    bestdiv = maxdiv
    *best_parent_rate = clk_hw_round_rate(parent, 1)

return bestdiv
```

**Key details:** الـ overflow guard `min(ULONG_MAX/rate, maxdiv)` مهمة جداً لأن الـ loop بتحسب `rate * i`. الحالة المثالية لما `rate * i == saved_parent_rate` بترجع فوراً بدون تغيير parent.

---

### Category 4: Exported Rate API

الـ functions دي exported للـ drivers التانية أو الـ clock providers التانية اللي بتبني dividers بمنطق مختلف.

---

#### `divider_recalc_rate`

```c
unsigned long divider_recalc_rate(struct clk_hw *hw, unsigned long parent_rate,
                                  unsigned int val,
                                  const struct clk_div_table *table,
                                  unsigned long flags, unsigned long width)
```

**بتعمل إيه:** بتحسب الـ output rate من الـ parent rate والـ raw register value. بتستخدم `_get_div` للتحويل ثم `DIV_ROUND_UP_ULL` للقسمة. لو الـ div جه صفر وmش `CLK_DIVIDER_ALLOW_ZERO` بتطبع WARN وترجع `parent_rate`.

**Parameters:**
- `hw` — للـ logging فقط (اسم الـ clock)
- `parent_rate` — الـ parent clock rate بالـ Hz
- `val` — الـ raw masked value من الـ register
- `table`, `flags`, `width` — خصائص الـ divider

**Return:** الـ output rate بالـ Hz

**Key details:** بتستخدم `DIV_ROUND_UP_ULL` مش division عادية عشان الـ rate بالـ Hz ممكن يتجاوز 32-bit. الـ div=0 مع `ALLOW_ZERO` معناه bypass (no division).

**Who calls it:** `clk_divider_recalc_rate` (الـ ops callback) وكمان الـ drivers التانية زي `clk-divider` variants.

---

#### `clk_divider_recalc_rate`

```c
static unsigned long clk_divider_recalc_rate(struct clk_hw *hw,
                                             unsigned long parent_rate)
```

**بتعمل إيه:** الـ `.recalc_rate` ops callback. بتقرأ الـ register باستخدام `clk_div_readl`، بتعمل shift وmask للـ field، ثم بتمرر للـ `divider_recalc_rate`.

**Who calls it:** الـ CCF core أثناء `clk_get_rate()` أو أي cache invalidation.

---

#### `divider_determine_rate`

```c
int divider_determine_rate(struct clk_hw *hw, struct clk_rate_request *req,
                           const struct clk_div_table *table, u8 width,
                           unsigned long flags)
```

**بتعمل إيه:** بتستدعي `clk_divider_bestdiv` لتحديد أفضل div وبتحدث `req->rate` بالـ rate الفعلي اللي هيتحقق. دي النقطة اللي بيعرف فيها الـ framework الـ rate الحقيقي قبل الـ set.

**Parameters:**
- `hw` — الـ clock
- `req` — `struct clk_rate_request` بيتضمن `rate`, `best_parent_rate`, `best_parent_hw`
- `table`, `width`, `flags` — خصائص الـ divider

**Return:** 0 دايماً (no error path)

**Key details:** بيعدل `req->rate` و`req->best_parent_rate` in-place. الـ `clk_rate_request` بيتحول بين الـ CCF layers.

---

#### `divider_ro_determine_rate`

```c
int divider_ro_determine_rate(struct clk_hw *hw, struct clk_rate_request *req,
                              const struct clk_div_table *table, u8 width,
                              unsigned long flags, unsigned int val)
```

**بتعمل إيه:** للـ read-only dividers — الـ div مش ممكن يتغير، بس لو `CLK_SET_RATE_PARENT` متضبط بيقدر يطلب من الـ parent rate جديد يحقق الـ rate المطلوب.

**Parameters:**
- `val` — الـ fixed register value (مش هيتغير)
- باقي الـ parameters زي `divider_determine_rate`

**Return:** 0 أو `-EINVAL` لو `CLK_SET_RATE_PARENT` ومفيش parent

**Key details:** بيحسب `req->best_parent_rate = round_rate(parent, rate * div)` ثم `req->rate = ceil(best_parent_rate / div)`.

---

#### `divider_round_rate_parent`

```c
long divider_round_rate_parent(struct clk_hw *hw, struct clk_hw *parent,
                               unsigned long rate, unsigned long *prate,
                               const struct clk_div_table *table,
                               u8 width, unsigned long flags)
```

**بتعمل إيه:** wrapper قديم (legacy) فوق `divider_determine_rate` بيستخدم الـ old-style API (بيرجع `long` مش بيعبي `req`). بيبني `clk_rate_request` داخلياً ثم بيستخدمه.

**Return:** الـ achievable rate أو error code سالب

---

#### `divider_ro_round_rate_parent`

```c
long divider_ro_round_rate_parent(struct clk_hw *hw, struct clk_hw *parent,
                                  unsigned long rate, unsigned long *prate,
                                  const struct clk_div_table *table, u8 width,
                                  unsigned long flags, unsigned int val)
```

**بتعمل إيه:** نفس `divider_round_rate_parent` بس للـ read-only variant، wrapper فوق `divider_ro_determine_rate`.

---

#### `divider_get_val`

```c
int divider_get_val(unsigned long rate, unsigned long parent_rate,
                    const struct clk_div_table *table, u8 width,
                    unsigned long flags)
```

**بتعمل إيه:** بتحول rate مطلوب لـ register value جاهز للكتابة. بتحسب `div = ceil(parent_rate / rate)` ثم بتتحقق من صحته ثم بتحوله لـ register val وبتعمل clamp بالـ mask.

**Parameters:**
- `rate` — الـ target output rate
- `parent_rate` — الـ parent rate الثابت
- `table`, `width`, `flags` — خصائص الـ divider

**Return:** الـ register value (non-negative) أو `-EINVAL` لو الـ div مش valid

**Key details:** ده الـ function اللي بيستخدمه `clk_divider_set_rate` مباشرة. الـ `min_t` في الـخير بيضمن إن الـ value مش بيتجاوز الـ mask.

---

### Category 5: clk_ops Callbacks

الـ CCF بيستدعي الـ callbacks دي من الـ `struct clk_ops` المسجلة. مش بيتم استدعاؤها مباشرة من الـ driver.

---

#### `clk_divider_determine_rate`

```c
static int clk_divider_determine_rate(struct clk_hw *hw,
                                      struct clk_rate_request *req)
```

**بتعمل إيه:** dispatcher — لو الـ divider `READ_ONLY` بتقرأ الـ current val من الـ register وبتستدعي `divider_ro_determine_rate`. غير كده `divider_determine_rate` العادية.

**Who calls it:** الـ CCF core في `clk_core_determine_rate_nolock()`.

---

#### `clk_divider_set_rate`

```c
static int clk_divider_set_rate(struct clk_hw *hw, unsigned long rate,
                                unsigned long parent_rate)
```

**بتعمل إيه:** بتحسب الـ register value بـ `divider_get_val` ثم بتكتبه في الـ register. بتاخد الـ spinlock لو موجود. بتتعامل مع `CLK_DIVIDER_HIWORD_MASK` بطريقة خاصة — مش بتقرأ الـ register الأول بل بتكتب الـ mask في الـ high 16-bit مباشرة.

**Locking:** `spin_lock_irqsave` / `spin_unlock_irqrestore` لو `divider->lock != NULL`. لو NULL بيستخدم `__acquire`/`__release` كـ annotation للـ sparse فقط (بدون locking فعلي).

**HIWORD_MASK handling:**
```c
/* لو HIWORD_MASK: الـ high 16-bit بيحمل الـ write-enable mask */
val = mask << (shift + 16);   /* no read-modify-write needed */
val |= value << shift;
writel(val);
```

**Key details:** لو `divider_get_val` رجع error سالب، بترجع الـ error مباشرة بدون لمس الـ register.

---

### الـ clk_ops Tables

```c
const struct clk_ops clk_divider_ops = {
    .recalc_rate    = clk_divider_recalc_rate,
    .determine_rate = clk_divider_determine_rate,
    .set_rate       = clk_divider_set_rate,
};

const struct clk_ops clk_divider_ro_ops = {
    .recalc_rate    = clk_divider_recalc_rate,
    .determine_rate = clk_divider_determine_rate,
    /* لا .set_rate — read-only */
};
```

الـ `clk_divider_ro_ops` بيفتقر لـ `.set_rate` لأن الـ register مش بيتكتب. الـ determination بيستخدم `divider_ro_determine_rate` داخلياً عبر الـ `clk_divider_determine_rate` dispatcher.

---

### Category 6: Registration & Cleanup

---

#### `__clk_hw_register_divider`

```c
struct clk_hw *__clk_hw_register_divider(struct device *dev,
        struct device_node *np, const char *name,
        const char *parent_name, const struct clk_hw *parent_hw,
        const struct clk_parent_data *parent_data, unsigned long flags,
        void __iomem *reg, u8 shift, u8 width,
        unsigned long clk_divider_flags,
        const struct clk_div_table *table, spinlock_t *lock)
```

**بتعمل إيه:** ده الـ core registration function. بتخصص `struct clk_divider` بـ `kzalloc`، بتملأ الـ `clk_init_data`، وبتستدعي `clk_hw_register`. لو `HIWORD_MASK` وwidth+shift > 16 بترجع `-EINVAL` بدون allocation.

**Parameters:**
- `dev` — الـ device للـ sysfs والـ devres
- `np` — device node للـ OF-based registration
- `name` — اسم الـ clock في الـ CCF
- `parent_name` / `parent_hw` / `parent_data` — مصدر الـ parent (واحد بس بيتحدد)
- `flags` — الـ CLK_* framework flags
- `reg` — عنوان الـ MMIO register
- `shift` — مكان الـ bit field
- `width` — عدد bits
- `clk_divider_flags` — الـ CLK_DIVIDER_* flags
- `table` — optional lookup table
- `lock` — spinlock مشترك مع registers تانية

**Return:** `struct clk_hw*` أو `ERR_PTR(-ENOMEM / -EINVAL / error)`

**Error paths:**
1. `HIWORD_MASK` مع `width + shift > 16` → `-EINVAL`
2. `kzalloc` فشل → `-ENOMEM`
3. `clk_hw_register` فشل → `kfree(div)` وreturn error

**Pseudocode:**
```
if HIWORD_MASK && width+shift > 16: return ERR_PTR(-EINVAL)
div = kzalloc(sizeof(*div))
if !div: return ERR_PTR(-ENOMEM)
init = { name, ops, flags, parent... }
div->{ reg, shift, width, flags, lock, table, hw.init }
ret = clk_hw_register(dev, &div->hw)
if ret: kfree(div); return ERR_PTR(ret)
return &div->hw
```

---

#### `__devm_clk_hw_register_divider`

```c
struct clk_hw *__devm_clk_hw_register_divider(struct device *dev, ...)
```

**بتعمل إيه:** نسخة managed من `__clk_hw_register_divider`. بتخصص `devres` pointer بـ `devres_alloc` لحفظ الـ `clk_hw*`، ولما الـ device بيتحذف الـ `devm_clk_hw_release_divider` بيتعمل call تلقائي.

**Key details:** لو الـ registration فشل بتعمل `devres_free(ptr)` عشان ما تحصلش leak.

---

#### `devm_clk_hw_release_divider`

```c
static void devm_clk_hw_release_divider(struct device *dev, void *res)
```

**بتعمل إيه:** الـ devres cleanup callback. بتستدعي `clk_hw_unregister_divider` على الـ `clk_hw*` المخزن في `res`.

**Who calls it:** الـ devres framework تلقائياً أثناء `device_del()` أو `driver_remove()`.

---

#### `clk_register_divider_table`

```c
struct clk *clk_register_divider_table(struct device *dev, const char *name,
        const char *parent_name, unsigned long flags,
        void __iomem *reg, u8 shift, u8 width,
        unsigned long clk_divider_flags,
        const struct clk_div_table *table, spinlock_t *lock)
```

**بتعمل إيه:** legacy API بيرجع `struct clk*` (مش `struct clk_hw*`). بتستدعي `__clk_hw_register_divider` وبترجع `hw->clk`. الـ drivers الجديدة لازم تستخدم `clk_hw_register_divider*` macros بدلاً منها.

**Return:** `struct clk*` أو `ERR_PTR`

---

#### `clk_unregister_divider`

```c
void clk_unregister_divider(struct clk *clk)
```

**بتعمل إيه:** legacy unregister. بتجيب الـ `clk_hw` من الـ `clk`، بتعمل `to_clk_divider`، بتستدعي `clk_unregister` وبتعمل `kfree` للـ `div`.

---

#### `clk_hw_unregister_divider`

```c
void clk_hw_unregister_divider(struct clk_hw *hw)
```

**بتعمل إيه:** الـ modern unregister. بتعمل `to_clk_divider(hw)`، بتستدعي `clk_hw_unregister(hw)`، ثم `kfree(div)`. دي اللي بتستدعيها `devm_clk_hw_release_divider`.

**Key details:** الترتيب مهم — لازم `clk_hw_unregister` الأول قبل `kfree` عشان الـ CCF core ممكن يكون لسه شايل pointer للـ `hw`.

---

### تدفق العمل الكامل — ASCII Flow

```
Driver init:
   __clk_hw_register_divider()
        │
        ├─ kzalloc(struct clk_divider)
        ├─ fill clk_init_data
        └─ clk_hw_register(dev, hw)
                │
                └─ CCF يضيف الـ clock للـ tree

Rate Query (clk_get_rate):
   CCF core
    └─ clk_divider_recalc_rate()
            │
            ├─ clk_div_readl()     [read register]
            ├─ shift + mask
            └─ divider_recalc_rate()
                    │
                    ├─ _get_div()  [val → divisor]
                    └─ DIV_ROUND_UP_ULL(parent_rate, div)

Rate Change (clk_set_rate):
   CCF core
    ├─ clk_divider_determine_rate()
    │       │
    │       ├─ [RO?] divider_ro_determine_rate()
    │       └─ [RW]  divider_determine_rate()
    │                   └─ clk_divider_bestdiv()
    │                           └─ exhaustive div search
    │
    └─ clk_divider_set_rate()
            │
            ├─ divider_get_val()   [rate → register val]
            ├─ spin_lock_irqsave()
            ├─ clk_div_readl() + mask [or HIWORD direct]
            ├─ clk_div_writel()
            └─ spin_unlock_irqrestore()
```

---

### جدول الـ CLK_DIVIDER Flags وتأثيرها

| Flag | Bit | تأثيره على `_get_div` | تأثيره على `_get_val` |
|---|---|---|---|
| `CLK_DIVIDER_ONE_BASED` | 0 | `val` مباشرة | `div` مباشرة |
| `CLK_DIVIDER_POWER_OF_TWO` | 1 | `1 << val` | `__ffs(div)` |
| `CLK_DIVIDER_ALLOW_ZERO` | 2 | يسمح div=0 (bypass) | — |
| `CLK_DIVIDER_HIWORD_MASK` | 3 | — | mask في high 16-bit |
| `CLK_DIVIDER_ROUND_CLOSEST` | 4 | — | يؤثر على الـ rounding |
| `CLK_DIVIDER_READ_ONLY` | 5 | — | يمنع `.set_rate` |
| `CLK_DIVIDER_MAX_AT_ZERO` | 6 | `val==0 → mask+1` | `div==mask+1 → 0` |
| `CLK_DIVIDER_BIG_ENDIAN` | 7 | يستخدم `ioread32be` | `iowrite32be` |
| `CLK_DIVIDER_EVEN_INTEGERS` | 8 | `2*(val+1)` | `(div>>1)-1` |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ clk-divider

الـ **Common Clock Framework (CCF)** بيعمل mount لـ debugfs تحت `/sys/kernel/debug/clk/`.
كل clock مسجّل بيطلع ليه directory خاص بيه فيه الـ entries دي:

```
/sys/kernel/debug/clk/<clock_name>/
├── clk_rate          ← الـ rate الحالي بالـ Hz
├── clk_min_rate      ← أدنى rate مسموح
├── clk_max_rate      ← أعلى rate مسموح
├── clk_accuracy      ← الدقة بالـ ppb
├── clk_phase         ← الـ phase بالدرجات
├── clk_flags         ← الـ flags الـ framework
├── clk_enable_count  ← عدد المرات اللي اتعمل فيها enable
├── clk_prepare_count ← عدد المرات اللي اتعمل فيها prepare
├── clk_protect_count ← عدد المرات اللي اتعمل فيها protect
├── clk_duty_cycle    ← نسبة الـ duty cycle
└── clk_summary       ← ملخص شامل لكل شيء
```

**إزاي تقرأهم:**

```bash
# اقرأ الـ rate الحالي للـ divider clock اللي اسمه "uart_clk_div"
cat /sys/kernel/debug/clk/uart_clk_div/clk_rate

# اطبع ملخص كامل للشجرة كلها من root
cat /sys/kernel/debug/clk/clk_summary

# شوف الـ flags بتاعة الـ clock عشان تعرف CLK_DIVIDER_READ_ONLY أو غيره
cat /sys/kernel/debug/clk/uart_clk_div/clk_flags
```

**مثال على output الـ clk_summary:**

```
                                 enable  prepare  protect                                duty
   clock                          count    count    count        rate   accuracy phase  cycle
---------------------------------------------------------------------------------------------
 uart_parent                          1        1        0   400000000          0     0  50000
    uart_clk_div                      1        1        0    50000000          0     0  50000
```

تفسير الـ output: الـ `uart_clk_div` بيقسم الـ `uart_parent` (400 MHz) على 8 ويطلع 50 MHz.

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ clock

الـ sysfs بيوفر معلومات عن الـ clock من ناحية الـ device:

```
/sys/bus/platform/devices/<device>/driver/
/sys/class/clk/                         ← لو kernel فيه CONFIG_COMMON_CLK_SYSFS
```

**الأهم للـ divider هو الـ device-specific sysfs:**

```bash
# شوف الـ clock اللي بيستخدمها الـ device
ls /sys/bus/platform/devices/fe300000.uart/

# فى بعض الـ SoCs بيظهر:
cat /sys/bus/platform/devices/fe300000.uart/clocks
```

---

#### 3. استخدام الـ ftrace

الـ **ftrace** بيوفر tracepoints جاهزة للـ CCF:

```bash
# اعرض كل الـ tracepoints المتاحة للـ clk
ls /sys/kernel/debug/tracing/events/clk/
```

```
clk_disable
clk_disable_complete
clk_enable
clk_enable_complete
clk_prepare
clk_prepare_complete
clk_set_rate
clk_set_rate_complete
clk_set_parent
clk_set_parent_complete
clk_set_phase
clk_set_phase_complete
```

**تتبع عمليات الـ set_rate على الـ divider:**

```bash
# فعّل الـ tracing لعملية set_rate
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate_complete/enable

# فعّل الـ tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية اللي بتستدعي set_rate
# مثلاً restart الـ UART
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
# TASK-PID   CPU#  IRQS-OFF  NEED-RESCHED  HARDIRQ/SOFTIRQ  PREEMPT-DEPTH  TIMESTAMP  FUNCTION
          <idle>-0   [000]   ...1    12.345678: clk_set_rate: uart_clk_div 50000000
          <idle>-0   [000]   ...1    12.345699: clk_set_rate_complete: uart_clk_div 50000000
```

**تتبع الـ recalc_rate لمعرفة إيه اللي بيحسب الـ rate:**

```bash
# استخدم function_graph tracer على دوال الـ divider
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo "clk_divider_recalc_rate divider_recalc_rate" > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. تفعيل الـ printk والـ dynamic debug

الـ **dynamic debug** بيخليك تفعّل رسائل `pr_debug()` و`dev_dbg()` بدون إعادة compile:

```bash
# فعّل كل رسائل الـ debug في ملف clk-divider.c
echo "file drivers/clk/clk-divider.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل كل رسائل debug للـ CCF subsystem كلها
echo "module clk +p" > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep clk-divider
```

**لو عايز تضيف رسائل مؤقتة في الـ kernel:**

```bash
# غيّر مستوى الـ printk المؤقت (بدون reboot)
echo 8 > /proc/sys/kernel/printk    # اطبع كل المستويات

# أو من command line أثناء boot:
# loglevel=8 في kernel cmdline
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_COMMON_CLK` | الـ CCF نفسه - لازم يكون enabled |
| `CONFIG_CLK_DEBUG` | يفعّل الـ debugfs entries للـ clocks |
| `CONFIG_COMMON_CLK_DEBUG` | يضيف debug info إضافية للـ CCF |
| `CONFIG_DEBUG_KERNEL` | يفعّل debugging عام في الـ kernel |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug |
| `CONFIG_TRACING` | يفعّل الـ ftrace infrastructure |
| `CONFIG_EVENT_TRACING` | يفعّل الـ tracepoints |
| `CONFIG_LOCKDEP` | يكتشف مشاكل الـ locking في الـ spinlock |
| `CONFIG_PROVE_LOCKING` | يثبت صحة الـ locking |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف مشاكل الـ spinlock (مهم لـ divider->lock) |
| `CONFIG_KCSAN` | يكتشف الـ data races |
| `CONFIG_KASAN` | يكتشف memory bugs في الـ kzalloc/kfree |

**لتشغيل kernel مع كل الـ debug options:**

```bash
# في .config
CONFIG_COMMON_CLK=y
CONFIG_CLK_DEBUG=y
CONFIG_DYNAMIC_DEBUG=y
CONFIG_EVENT_TRACING=y
CONFIG_DEBUG_SPINLOCK=y
```

---

#### 6. أدوات خاصة بالـ subsystem

**الـ clk tool (userspace):**

```bash
# مش موجود افتراضياً لكن ممكن تستخدم:
# على Android/embedded:
cat /sys/kernel/debug/clk/clk_summary

# على بعض الـ distros:
# busybox devmem لقراءة الـ registers مباشرة
```

**الـ devlink:** مش مرتبط مباشرة بالـ clock divider.

**أداة clk_test (kernel selftests):**

```bash
# تشغيل الـ selftests الخاصة بالـ CCF
cd tools/testing/selftests/drivers/clk
make
./clk_test
```

---

#### 7. رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `Zero divisor and CLK_DIVIDER_ALLOW_ZERO not set` | الـ register بيحتوي على قيمة 0 كـ divisor وده مش مسموح | اضف `CLK_DIVIDER_ALLOW_ZERO` للـ flags أو تحقق من قيمة الـ register بعد reset |
| `divider value exceeds LOWORD field` | الـ width+shift بتتجاوز الـ 16 bit في وضع `CLK_DIVIDER_HIWORD_MASK` | راجع قيم الـ width وshift في الـ registration |
| `clk_hw_register` returns `-ENOMEM` | فشل الـ kzalloc عند تسجيل الـ divider | مشكلة في الـ memory - شوف OOM killer في الـ dmesg |
| `clk_hw_register` returns `-EINVAL` | اسم الـ clock مكرر أو parent غير موجود | تأكد من uniqueness الاسم وصحة اسم الـ parent |
| `clk_set_rate` returns `-EINVAL` | الـ div المطلوب مش موجود في الـ table أو مش valid | تحقق من `_is_valid_div()` وقيم الـ clk_div_table |
| `clk_set_rate` returns `-EPERM` | الـ clock معمله `CLK_DIVIDER_READ_ONLY` | الـ clock read-only بالتعريف - استخدم clock تاني |
| `clk_set_rate` returns `0` لكن الـ rate غلط | الـ rounding أعطى نتيجة غير متوقعة | فعّل `CLK_DIVIDER_ROUND_CLOSEST` أو راجع الـ parent rate |
| رسالة `WARNING` في `divider_recalc_rate` | الـ div = 0 ومش مسموح | غلط في الـ HW أو الـ register ما اتكتبش صح |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

**أماكن مناسبة لإضافة assertions أثناء الـ debugging:**

```c
static int clk_divider_set_rate(struct clk_hw *hw, unsigned long rate,
                                unsigned long parent_rate)
{
    struct clk_divider *divider = to_clk_divider(hw);
    int value;

    value = divider_get_val(rate, parent_rate, divider->table,
                            divider->width, divider->flags);

    /* Assert: value يجب أن يكون valid */
    WARN_ON(value < 0);

    /* Assert: الـ register pointer لازم يكون mapped */
    WARN_ON(!divider->reg);

    /* Assert: الـ width لازم يكون > 0 */
    WARN_ON(divider->width == 0);

    if (value < 0)
        return value;

    /* ... بقية الكود */
}
```

```c
unsigned long divider_recalc_rate(struct clk_hw *hw, unsigned long parent_rate,
                                  unsigned int val,
                                  const struct clk_div_table *table,
                                  unsigned long flags, unsigned long width)
{
    unsigned int div;

    div = _get_div(table, val, flags, width);

    /* هنا WARN_ON موجودة أصلاً في الكود */
    WARN(!(flags & CLK_DIVIDER_ALLOW_ZERO),
         "%s: Zero divisor and CLK_DIVIDER_ALLOW_ZERO not set\n",
         clk_hw_get_name(hw));

    /* نقطة استراتيجية إضافية: تحقق من overflow */
    WARN_ON(parent_rate > ULONG_MAX / (div ? div : 1));
}
```

**لـ dump_stack() ضيفها في `__clk_hw_register_divider` عند الـ error:**

```c
ret = clk_hw_register(dev, hw);
if (ret) {
    pr_err("clk_divider: failed to register '%s': %d\n", name, ret);
    dump_stack();   /* يطبع الـ call stack كاملاً */
    kfree(div);
    hw = ERR_PTR(ret);
}
```

---

### Hardware Level

---

#### 1. التحقق من أن حالة الـ Hardware تطابق حالة الـ Kernel

الفكرة: اقرأ الـ register مباشرة وقارنه بما يعتقده الـ kernel.

```bash
# الخطوة 1: اعرف الـ rate اللي بيقوله الـ kernel
cat /sys/kernel/debug/clk/uart_clk_div/clk_rate
# مثلاً: 50000000

# الخطوة 2: اعرف الـ parent rate
cat /sys/kernel/debug/clk/uart_parent/clk_rate
# مثلاً: 400000000

# الخطوة 3: احسب الـ divider المتوقع
# div = parent / rate = 400000000 / 50000000 = 8

# الخطوة 4: اقرأ الـ register الفعلي
# افترض الـ register عند العنوان 0xFE300010 والـ shift=0 والـ width=4
devmem2 0xFE300010 w
# لو القيمة = 7 في وضع "val+1" → div = 8 ✓ (صح)
# لو القيمة = 8 في وضع ONE_BASED → div = 8 ✓ (صح)
# لو القيمة = 3 في وضع POWER_OF_TWO → div = 2^3 = 8 ✓ (صح)
```

**Script للتحقق الآلي:**

```bash
#!/bin/bash
CLK_NAME="uart_clk_div"
REG_ADDR="0xFE300010"
SHIFT=0
WIDTH=4
MASK=$(( (1 << WIDTH) - 1 ))

KERNEL_RATE=$(cat /sys/kernel/debug/clk/${CLK_NAME}/clk_rate)
PARENT_RATE=$(cat /sys/kernel/debug/clk/uart_parent/clk_rate)

REG_VAL=$(devmem2 ${REG_ADDR} w | grep "Read at" | awk '{print $NF}')
RAW_DIV=$(( (REG_VAL >> SHIFT) & MASK ))
HW_DIV=$(( RAW_DIV + 1 ))   # افتراض: val+1 mode

EXPECTED_RATE=$(( PARENT_RATE / HW_DIV ))

echo "Kernel rate:   ${KERNEL_RATE} Hz"
echo "Expected rate: ${EXPECTED_RATE} Hz"
echo "HW divisor:    ${HW_DIV}"

if [ "${KERNEL_RATE}" -eq "${EXPECTED_RATE}" ]; then
    echo "STATUS: OK - HW matches kernel"
else
    echo "STATUS: MISMATCH! Kernel thinks rate is different from HW"
fi
```

---

#### 2. تقنيات الـ Register Dump

**باستخدام devmem2 (الأشهر):**

```bash
# تثبيت devmem2
apt-get install devmem2
# أو بناؤها من المصدر

# قراءة register بالـ word (32-bit)
devmem2 0xFE300010 w

# كتابة قيمة للـ register (احترس!)
devmem2 0xFE300010 w 0x00000007
```

**باستخدام /dev/mem مباشرة:**

```bash
# قراءة 4 bytes من عنوان 0xFE300010
dd if=/dev/mem bs=4 count=1 skip=$((0xFE300010/4)) 2>/dev/null | xxd
```

**باستخدام io utility:**

```bash
# متوفرة في بعض الـ embedded distros
io -4 0xFE300010       # قراءة 32-bit
io -4 -w 0xFE300010 7  # كتابة 32-bit
```

**باستخدام /proc/iomem لمعرفة عناوين الـ registers:**

```bash
# ابحث عن الـ clock controller في الـ memory map
cat /proc/iomem | grep -i "clock\|clk\|cmu\|pll"
```

**مثال على output:**

```
fe300000-fe3fffff : fe300000.clock-controller
  fe300000-fe3fffff : clk-cru
```

**ثم dump الـ region كاملة:**

```bash
# dump أول 256 bytes من الـ clock controller
devmem2 0xfe300000 w  # reg 0x00
devmem2 0xfe300004 w  # reg 0x04
devmem2 0xfe300008 w  # reg 0x08
devmem2 0xfe300010 w  # reg 0x10 (مثلاً: UART divider)
```

**أو باستخدام loop:**

```bash
#!/bin/bash
BASE=0xfe300000
for offset in $(seq 0 4 64); do
    addr=$(printf "0x%08x" $((BASE + offset)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Read at" | awk '{print $NF}')
    printf "Reg[0x%03x] = %s\n" $offset "$val"
done
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**الـ Oscilloscope:**

- **قِس الـ clock output** على الـ pin المناسب (مثلاً UART_CLK) وقارن بالـ rate المتوقع من الـ kernel.
- **التحقق من الـ glitches**: أثناء استدعاء `clk_set_rate()` من الـ kernel، المفروض مش يكون في glitch على الـ clock إلا لو الـ hardware بيعمل ungated rate change.
- **قياس الـ duty cycle**: يجب أن يكون قريباً من 50% للـ divider العادي.

```
         parent_clk (400MHz)
         ┌┐┌┐┌┐┌┐┌┐┌┐┌┐┌┐
         ││││││││││││││││││
──────── ┘└┘└┘└┘└┘└┘└┘└┘└

         output_clk (div=8, 50MHz)
         ┌───────┐       ┌───────
         │       │       │
─────────┘       └───────┘
         ←  20ns →
```

**الـ Logic Analyzer:**

- صِّل على الـ clock output line.
- اضبط الـ trigger على rising edge.
- قِس الـ frequency باستخدام الـ measurement tool.
- لو الـ divider مش شغال صح، هتلاقي frequency مختلفة عن المتوقع.

**نقاط القياس المهمة:**

| نقطة القياس | ما يجب أن تجده |
|---|---|
| Parent clock pin | الـ rate الأصلي قبل القسمة |
| Divider output pin | Parent rate / div value |
| Power rails | Stable voltage, no droops أثناء rate change |
| Reset line | لا توجد glitch أثناء `clk_set_rate` |

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| مشكلة الـ HW | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| الـ register مش accessible (bad mapping) | `Unable to handle kernel paging request` أو `BUG: unable to handle kernel NULL pointer dereference` | الـ `divider->reg` بيشاور على عنوان غلط |
| القسمة بصفر (register = 0 بعد reset) | `WARN_ON: Zero divisor and CLK_DIVIDER_ALLOW_ZERO not set` | الـ HW بيخرج من reset وقيمة الـ divider = 0 |
| الـ clock مش stable بعد set_rate | `clk_set_rate_complete: <name> <rate>` بدون error لكن الـ device بيتعطل | الـ HW بيحتاج وقت settle بعد تغيير الـ divider |
| مشكلة الـ concurrency في الـ register | `BUG: spinlock recursion on CPU` | الـ `divider->lock` مش متشارك صح مع registers تانية |
| الـ divider value فوق الـ max | الـ rate المحسوب أقل من المتوقع بكتير | الـ width مش محسوبة صح في الـ DT أو الـ driver |
| الـ big-endian/little-endian mismatch | الـ rate = قيمة عشوائية | `CLK_DIVIDER_BIG_ENDIAN` flag مش مضافة رغم أن الـ SoC big-endian |

---

#### 5. تشخيص الـ Device Tree

**التحقق من أن الـ DT يطابق الـ HW:**

```bash
# اعرض الـ DT المُفعَّل حالياً
cat /proc/device-tree/clocks/clock-controller@fe300000/compatible
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A20 "clock-controller"
```

**مثال على DT node للـ divider clock:**

```dts
/* في الـ SoC DTS */
clock_controller: clk@fe300000 {
    compatible = "vendor,soc-cru";
    reg = <0xfe300000 0x10000>;
    #clock-cells = <1>;
};

/* clock consumer */
uart0: serial@fe600000 {
    compatible = "snps,dw-apb-uart";
    clocks = <&clock_controller CLK_UART0>;
    clock-names = "baudclk";
};
```

**التحقق من صحة الـ DT:**

```bash
# تحقق من أن الـ reg في الـ DT يطابق الـ datasheet
cat /proc/device-tree/clocks/clock-controller@fe300000/reg | xxd

# تحقق من أن الـ #clock-cells صح
cat /proc/device-tree/clocks/clock-controller@fe300000/\#clock-cells | xxd

# شوف الـ clocks اللي بيستخدمها الـ UART
cat /proc/device-tree/serial@fe600000/clocks | xxd
```

**لو الـ DT مش بيطابق الـ HW:**

```bash
# فحص الـ DT overlay المُطبَّق
ls /sys/kernel/config/device-tree/overlays/

# مقارنة الـ DT المُفسَّر بالـ hardware datasheet
dtc -I fs /proc/device-tree -O dts 2>/dev/null > /tmp/current_dts.txt
diff /tmp/current_dts.txt /tmp/expected_dts.txt
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**الأوامر الأساسية للتشخيص:**

```bash
# 1. عرض شجرة الـ clocks كاملة مع الـ rates
cat /sys/kernel/debug/clk/clk_summary

# 2. البحث عن clock محدد باسمه
grep "uart" /sys/kernel/debug/clk/clk_summary

# 3. عرض rate لـ clock محدد
CLK="uart_clk_div"
cat /sys/kernel/debug/clk/${CLK}/clk_rate

# 4. عرض كل معلومات clock واحد
for f in /sys/kernel/debug/clk/${CLK}/*; do
    echo "=== $(basename $f) ==="; cat $f 2>/dev/null; echo
done

# 5. تفعيل كل tracing للـ CCF
for evt in clk_set_rate clk_set_rate_complete clk_enable clk_disable; do
    echo 1 > /sys/kernel/debug/tracing/events/clk/${evt}/enable
done
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 6. قراءة الـ trace بعد العملية
cat /sys/kernel/debug/tracing/trace

# 7. إيقاف الـ tracing وتنظيف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace

# 8. تفعيل dynamic debug للـ clk-divider
echo "file drivers/clk/clk-divider.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w  # تابع الـ output

# 9. فحص kernel messages المتعلقة بالـ clocks
dmesg | grep -i "clk\|clock\|divider\|pll"

# 10. قراءة register مباشرة (غيّر العنوان حسب الـ SoC)
devmem2 0xFE300010 w
```

**فحص شامل لمشكلة rate خاطئ:**

```bash
#!/bin/bash
# clock_debug.sh - تشخيص شامل لـ clk-divider
CLK="${1:-uart_clk_div}"

echo "=== Clock Debug for: ${CLK} ==="
echo ""

echo "--- Kernel reported values ---"
echo "Rate:          $(cat /sys/kernel/debug/clk/${CLK}/clk_rate 2>/dev/null) Hz"
echo "Min rate:      $(cat /sys/kernel/debug/clk/${CLK}/clk_min_rate 2>/dev/null) Hz"
echo "Max rate:      $(cat /sys/kernel/debug/clk/${CLK}/clk_max_rate 2>/dev/null) Hz"
echo "Enable count:  $(cat /sys/kernel/debug/clk/${CLK}/clk_enable_count 2>/dev/null)"
echo "Prepare count: $(cat /sys/kernel/debug/clk/${CLK}/clk_prepare_count 2>/dev/null)"
echo "Flags:         $(cat /sys/kernel/debug/clk/${CLK}/clk_flags 2>/dev/null)"

echo ""
echo "--- Clock tree context ---"
grep -A5 "${CLK}" /sys/kernel/debug/clk/clk_summary 2>/dev/null

echo ""
echo "--- Recent kernel messages ---"
dmesg | grep -i "${CLK}" | tail -20
```

**تشغيله:**

```bash
chmod +x clock_debug.sh
./clock_debug.sh uart_clk_div
```

**مثال على output وتفسيره:**

```
=== Clock Debug for: uart_clk_div ===

--- Kernel reported values ---
Rate:          50000000 Hz         ← 50 MHz - الـ rate الحالي
Min rate:      1000000 Hz          ← 1 MHz - أدنى rate
Max rate:      400000000 Hz        ← 400 MHz - أعلى rate (= parent)
Enable count:  1                   ← مفعّل مرة واحدة (طبيعي)
Prepare count: 1                   ← مُحضَّر مرة واحدة (طبيعي)
Flags:         0x00000000          ← لا يوجد CLK_SET_RATE_PARENT

--- Clock tree context ---
                                 enable  prepare  protect     rate
   uart_parent                       1        1        0  400000000
      uart_clk_div                   1        1        0   50000000
← الـ divider = 8 (400M / 50M)

--- Recent kernel messages ---
[    2.345678] clk: uart_clk_div registered, rate=50000000
```

**تفسير enable_count=0 مع rate صح:**

```bash
# لو enable_count=0 لكن الـ rate موجود، ده طبيعي
# الـ clock registered لكن مش متطلوب من أي device حالياً
# الـ rate المعروض هو آخر rate محسوب، مش بالضرورة active
```

**فحص مشكلة الـ zero divisor:**

```bash
# فعّل الـ dynamic debug عشان تشوف الـ WARN
echo "file drivers/clk/clk-divider.c +p" > /sys/kernel/debug/dynamic_debug/control
echo 8 > /proc/sys/kernel/printk
# ثم trigger الـ clk_set_rate
dmesg | grep "Zero divisor"
# لو ظهر: "Zero divisor and CLK_DIVIDER_ALLOW_ZERO not set"
# → الحل: ضيف CLK_DIVIDER_ALLOW_ZERO في الـ flags أو اصلح الـ HW reset value
```

**فحص مشكلة الـ HIWORD_MASK:**

```bash
# لو ظهر في dmesg:
# "divider value exceeds LOWORD field"
# → تحقق من قيم الـ shift و width في الـ driver أو DTS
# المعادلة: width + shift يجب أن يكون <= 16
echo "width + shift must be <= 16 for HIWORD_MASK mode"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART بتيجي بسرعة غلط على Industrial Gateway بـ RK3562

#### العنوان
**الـ baud rate بيجي دايمًا أقل من المطلوب على UART2 في gateway صناعي**

#### السياق
شركة بتبني industrial gateway بالـ RK3562 عشان تربط معدات Modbus RTU بشبكة Ethernet. الـ UART2 محتاج يشتغل بـ 115200 baud. البورد شغال بـ production لكن الـ slaves مش بتستجاوبش بانتظام.

#### المشكلة
الـ UART2 فعليًا شغال بـ ~114285 baud بدل 115200. الفرق صغير لكن كافي يعمل framing errors مع بعض devices دقيقة في التوقيت. المهندس ما لاقى المشكلة من أول لأن الـ oscilloscope ما اتستخدمش.

```bash
# المهندس لاحظ في dmesg:
[   12.443] serial: 8250: tty0: 1 input overrun(s)
```

#### التحليل
الـ UART clock جاي من `uart_clk` اللي هو output من divider. لما المهندس فحص:

```bash
cat /sys/kernel/debug/clk/uart2_clk/clk_rate
# نتيجة: 114285714
```

راح لكود الـ divider. الـ parent clock هو PLL عند 800 MHz، والـ divider مسجل بـ `width=4` و `shift=0` وبدون flags خاصة يعني **default mode** — الـ `_get_div` بترجع `val + 1`:

```c
/* من _get_div() — الـ default case */
return val + 1;   /* val=6 → div=7 → 800MHz/7 = 114.28MHz */
```

المهندس طلب 115200 * 16 = ~1.8432 MHz للـ UART oversampling. الـ `clk_divider_bestdiv` اختار divisor=7 لأنه الأقرب لـ ceiling، لكن الـ `_div_round_up` جاب:

```c
int div = DIV_ROUND_UP_ULL((u64)800000000, 115200 * 16);
/* = DIV_ROUND_UP(800000000, 1843200) = 434 */
```

المشكلة أن الـ PLL source مش 800 MHz بالظبط ولا الـ table بتسمح بكل divisor، فالأقرب هو 435 → 800M/435 = 1,839,080 Hz ≠ 1,843,200 Hz.

#### الحل
الحل كان تغيير الـ PLL إلى قيمة قابلة للقسمة بشكل نظيف، أو استخدام `CLK_DIVIDER_ROUND_CLOSEST` في registration:

```c
/* في ملف clk-rk3562.c عند تسجيل الـ uart clock divider */
clk_hw_register_divider(dev, "uart2_div", "cpll",
    CLK_SET_RATE_PARENT,
    base + RK3562_UART2_CON,
    0, 4,
    CLK_DIVIDER_ROUND_CLOSEST,  /* أضف هذا */
    &rk3562_lock);
```

لأن `_div_round_closest` بتحسب:

```c
/* من _div_round_closest() */
up_rate   = DIV_ROUND_UP_ULL(parent_rate, up);
down_rate = DIV_ROUND_UP_ULL(parent_rate, down);
return (rate - up_rate) <= (down_rate - rate) ? up : down;
/* بتختار الأقرب لـ 1843200 Hz فعليًا */
```

أو الأسهل: تغيير الـ PLL إلى 1,474,560,000 Hz (مضاعف نظيف لـ 115200 × 16 = 1,843,200).

#### الدرس المستفاد
الـ `CLK_DIVIDER_ROUND_CLOSEST` مش luxury — هو ضروري لأي clock بيخدم protocol حساس للـ baud rate زي UART/SPI. الـ default `_div_round_up` دايمًا بيدي frequency أقل من المطلوب.

---

### السيناريو الثاني: SPI Flash مش بيتعرف عليه عند boot على STM32MP1

#### العنوان
**الـ SPI1 clock بيتجاوز الـ 50 MHz المسموح بيه للـ Flash، البورد مش بتبوت**

#### السياق
منتج IoT sensor بيستخدم STM32MP1 مع W25Q128 SPI NOR Flash. الـ flash datasheet بتحدد max 50 MHz. عند board bring-up الجديد، الـ u-boot بيتقل وبيعمل hang أحيانًا عند قراءة الـ flash.

#### المشكلة
الـ SPI clock بيتسجل كـ read-only divider (`CLK_DIVIDER_READ_ONLY`) لأن الـ bootloader هو اللي بيضبطه. الـ kernel بيقرا القيمة ومش بيغيرها. لكن الـ bootloader نفسه بيضبط divisor غلط.

```bash
# في kernel dmesg
[    2.1] spi-nor spi0.0: unrecognized JEDEC id bytes: ff ff ff
```

#### التحليل
الـ `clk_divider_determine_rate` لما شاف الـ `CLK_DIVIDER_READ_ONLY` flag:

```c
static int clk_divider_determine_rate(struct clk_hw *hw,
                                      struct clk_rate_request *req)
{
    struct clk_divider *divider = to_clk_divider(hw);

    /* if read only, just return current value */
    if (divider->flags & CLK_DIVIDER_READ_ONLY) {
        u32 val;
        val = clk_div_readl(divider) >> divider->shift;
        val &= clk_div_mask(divider->width);
        return divider_ro_determine_rate(hw, req, divider->table,
                                         divider->width,
                                         divider->flags, val);
    }
    /* ... */
}
```

الـ `val` اللي اتقرأ من الـ register كان 1 (يعني divisor = 2 بالـ default encoding)، فالـ clock = 200 MHz / 2 = 100 MHz. الـ SPI flash انهار.

```bash
# تأكيد
cat /sys/kernel/debug/clk/spi1_ker_ck/clk_rate
# 100000000  ← 100 MHz ! أعلى من الـ 50 MHz المسموح
```

المشكلة في الـ bootloader كان بيكتب val=1 بدل val=3 (÷4 = 50 MHz):

```c
/* divider_recalc_rate — default mode */
div = _get_div(table, val, flags, width);
/* val=1, no flags → return val+1 = 2 → 200MHz/2 = 100MHz */
```

#### الحل
المهندس عدّل الـ bootloader SPL عشان يكتب val=3:

```c
/* في stm32mp1 SPL clock init */
/* نريد 200MHz / 4 = 50MHz → val = div - 1 = 3 */
writel((readl(RCC_SPI1CKSELR) & ~GENMASK(3,0)) | 3,
       RCC_SPI1DIVR);
```

وكتأكيد، أضاف assertion في kernel driver:

```c
/* بعد probe، تأكد الـ rate */
rate = clk_get_rate(spi_clk);
if (rate > 50000000) {
    dev_err(dev, "SPI clock %lu Hz exceeds flash limit!\n", rate);
    return -EINVAL;
}
```

#### الدرس المستفاد
الـ `CLK_DIVIDER_READ_ONLY` معناه الـ kernel واثق في الـ bootloader. لازم تتحقق يدويًا إن الـ bootloader ضبط الـ divider صح قبل ما تعتمد على الـ flag ده. استخدم `clk_get_rate()` في الـ driver `probe` كـ sanity check.

---

### السيناريو الثالث: HDMI مش بيشتغل على Android TV Box بـ Allwinner H616

#### العنوان
**الـ HDMI بيدي صورة على 1080p بس مش على 4K بسبب divider table ناقصة**

#### السياق
شركة صينية بتعمل Android TV box بالـ Allwinner H616. الـ HDMI بيشتغل تمام على 1080p/60Hz بس لما المستخدم بيختار 4K/30Hz من الـ settings، الشاشة بتبقى سودا.

#### المشكلة
الـ HDMI clock على H616 محتاج ~297 MHz لـ 4K/30Hz. الـ `clk_div_table` المسجلة في الـ driver ناقصها المدخل ده.

#### التحليل
الـ HDMI controller بيطلب rate عن طريق `clk_set_rate()`. الـ CCF بيستدعي `clk_divider_determine_rate` → `clk_divider_bestdiv`. جوا `clk_divider_bestdiv`:

```c
for (i = _next_div(table, 0, flags); i <= maxdiv;
                     i = _next_div(table, i, flags)) {
    parent_rate = clk_hw_round_rate(parent, rate * i);
    now = DIV_ROUND_UP_ULL((u64)parent_rate, i);
    if (_is_best_div(rate, now, best, flags)) {
        bestdiv = i;
        best = now;
        *best_parent_rate = parent_rate;
    }
}
```

الـ `_next_div` مع الـ table بتستدعي `_round_up_table` اللي بتدور على أقرب قيمة أكبر من `div` في الجدول. لو الجدول مش فيه القيمة المناسبة، بترجع `INT_MAX` وبكده الـ loop ما بيلاقيش divisor مناسب:

```c
static int _round_up_table(const struct clk_div_table *table, int div)
{
    int up = INT_MAX;
    for (clkt = table; clkt->div; clkt++) {
        if (clkt->div == div) return clkt->div;
        else if (clkt->div < div) continue;
        if ((clkt->div - div) < (up - div))
            up = clkt->div;
    }
    return up;  /* INT_MAX لو مفيش قيمة مناسبة */
}
```

فالـ `bestdiv` بيبقى 0 ويدخل الـ fallback:

```c
if (!bestdiv) {
    bestdiv = _get_maxdiv(table, width, flags);
    *best_parent_rate = clk_hw_round_rate(parent, 1);
}
```

الـ maxdiv في الجدول كان 4، فبيضبط الـ PLL على أدنى rate ممكن، والنتيجة rate غلط تمامًا.

```bash
# تشخيص
cat /sys/kernel/debug/clk/hdmi_tmds_clk/clk_rate
# 148500000  ← 1080p60 rate، مش 297MHz اللي محتاجاه
```

#### الحل
إضافة المدخل الناقص في الـ table:

```c
/* في drivers/clk/sunxi-ng/ccu-h616.c أو الملف المشابه */
static const struct clk_div_table h616_hdmi_div_table[] = {
    { .val = 0, .div = 1 },
    { .val = 1, .div = 2 },
    { .val = 2, .div = 4 },
    { .val = 3, .div = 8 },  /* ← المدخل الناقص للـ 4K */
    { /* sentinel */ }
};
```

مع ضبط الـ PLL parent ليوصل 2376 MHz عشان ÷8 = 297 MHz.

#### الدرس المستفاد
الـ `clk_div_table` لازم تغطي **كل** الـ divisors المدعومة في الـ hardware. مدخل ناقص بيخلي `_round_up_table` ترجع `INT_MAX` وبكده الـ clock بيفشل صامتًا بدل ما يرجع error واضح.

---

### السيناريو الرابع: I2C بيعمل lockup على Automotive ECU بـ i.MX8

#### العنوان
**الـ I2C3 بيتعلق تحت load عالي بسبب race condition في `clk_divider_set_rate`**

#### السياق
ECU خاص بنظام كاميرا rear-view على سيارة بيستخدم i.MX8QM. الـ I2C3 بيشيل كاميرات GMSL. تحت load عالي (كل الكاميرات شغالة + CPU load عالي)، الـ I2C controller بيعمل lockup ومحتاج reset.

#### المشكلة
الـ driver بيعمل `clk_set_rate` على الـ I2C clock من context غير مناسب، والـ spinlock المشترك مع clock تاني بيعمل deadlock تحت شروط معينة.

#### التحليل
في `clk_divider_set_rate`:

```c
static int clk_divider_set_rate(struct clk_hw *hw, unsigned long rate,
                                unsigned long parent_rate)
{
    struct clk_divider *divider = to_clk_divider(hw);
    int value;
    unsigned long flags = 0;
    u32 val;

    value = divider_get_val(rate, parent_rate, divider->table,
                            divider->width, divider->flags);
    if (value < 0)
        return value;

    if (divider->lock)
        spin_lock_irqsave(divider->lock, flags);  /* ← هنا المشكلة */
    else
        __acquire(divider->lock);
    /* ... */
    if (divider->lock)
        spin_unlock_irqrestore(divider->lock, flags);
}
```

الـ `spinlock_t *lock` ده shared بين عدة clocks في نفس الـ CCM block. لو clock تاني في interrupt handler حاول يعمل `clk_set_rate` في نفس الوقت، بيحصل priority inversion مع الـ I2C driver اللي شغال في process context.

المشكلة ظهرت لأن الـ I2C driver كان بيعمل `clk_set_rate` من داخل `i2c_transfer` اللي ممكن يتنادى من kernel thread بيشيل IRQ ثاني.

```bash
# من kernel trace
[  145.332] BUG: spinlock lockup suspected on CPU#2
[  145.332] lock: imx8_clk_lock+0x0
[  145.332] caller: clk_divider_set_rate+0x44
```

#### الحل
المهندس ثبّت الـ I2C clock rate مرة واحدة في الـ probe وما عادش بيغيره:

```c
/* في driver I2C probe */
ret = clk_set_rate(i2c_clk, I2C_CLK_RATE);
if (ret) {
    dev_err(dev, "failed to set I2C clk rate: %d\n", ret);
    return ret;
}
/* بعدها: اعمل prepare_enable بس، ما تعملش set_rate تاني */
ret = clk_prepare_enable(i2c_clk);
```

وعلى مستوى الـ clock، استخدم `CLK_DIVIDER_READ_ONLY` بعد الـ init لمنع أي تغيير لاحق:

```c
/* register I2C clock as read-only after initial setup */
divider->flags |= CLK_DIVIDER_READ_ONLY;
```

#### الدرس المستفاد
الـ `spinlock` في `clk_divider_set_rate` بيحمي الـ register access لكن مش بيحل مشاكل الـ locking order بين drivers. الـ I2C clock في production system المفروض يتثبت مرة واحدة، مش يتغير dynamically.

---

### السيناريو الخامس: USB Host مش شغال على AM62x بسبب CLK_DIVIDER_HIWORD_MASK غلط

#### العنوان
**USB devices مش بتتعرف عليها على AM62x LP SK board بعد تعديل الـ Device Tree**

#### السياق
مهندس BSP بيعمل custom board بالـ AM62x LP لتطبيق industrial HMI. عمل copy من الـ reference DT وعدّل عليه. بعد التعديل، USB host ports مش بيشوفوا أي device.

#### المشكلة
المهندس عدّل الـ `assigned-clock-rates` في الـ DT عشان يغير USB clock، لكن الـ register بتستخدم `CLK_DIVIDER_HIWORD_MASK` وهو مش عارف.

#### التحليل
الـ `CLK_DIVIDER_HIWORD_MASK` معناه إن الـ 16 bit العليا من الـ register هي الـ write mask للـ 16 bit السفلى. لما `clk_divider_set_rate` بيشتغل:

```c
if (divider->flags & CLK_DIVIDER_HIWORD_MASK) {
    /* الـ mask بيتكتب في الـ high word */
    val = clk_div_mask(divider->width) << (divider->shift + 16);
} else {
    /* read-modify-write عادي */
    val = clk_div_readl(divider);
    val &= ~(clk_div_mask(divider->width) << divider->shift);
}
val |= (u32)value << divider->shift;
clk_div_writel(divider, val);
```

المهندس في driver مش كاتب `CLK_DIVIDER_HIWORD_MASK`، فالكود بيعمل read-modify-write عادي. لكن الـ hardware على AM62x بيفسر الـ write مختلف — بيشوف الـ high bits كـ mask، وبما إنها zero في الـ write العادي، بيعني "لا تغير شيء" — فالـ divider مش بيتغير.

```bash
# قبل وبعد set_rate
devmem2 0x04040000 w   # قراءة USB clock reg
# 0x00030002  ← نفس القيمة! الـ write ما اشتغلش
```

المهندس أيضًا تجاوز الـ validation في `__clk_hw_register_divider`:

```c
if (clk_divider_flags & CLK_DIVIDER_HIWORD_MASK) {
    if (width + shift > 16) {
        pr_warn("divider value exceeds LOWORD field\n");
        return ERR_PTR(-EINVAL);
    }
}
```

الـ validation دي بتشتغل لو الـ flag اتضبط، بس المهندس مش ضابطه أصلًا.

```bash
# التحقق من الـ clk debugfs
cat /sys/kernel/debug/clk/usb_clk/clk_rate
# 60000000  ← المطلوب 120MHz للـ USB 2.0 High Speed
```

#### الحل
تصحيح الـ clock registration في الـ platform driver:

```c
/* في drivers/clk/ti/clk-am62x.c أو مشابه */
hw = clk_hw_register_divider(dev, "usb_clkdiv",
    "hsdivider_clkout3",
    0,                          /* framework flags */
    usb_ctrl_base + USB_CLKDIV_REG,
    0,   /* shift */
    4,   /* width */
    CLK_DIVIDER_HIWORD_MASK |   /* ← الـ flag الناقص */
    CLK_DIVIDER_ONE_BASED,
    &am62x_clk_lock);
```

وللتحقق بعد الإصلاح:

```bash
# كتابة الـ rate وتحقق
echo 120000000 > /sys/kernel/debug/clk/usb_clk/clk_rate
devmem2 0x04040000 w
# 0x00FF0001  ← high byte = mask، low byte = divider value
```

#### الدرس المستفاد
الـ `CLK_DIVIDER_HIWORD_MASK` مش مجرد optimization — هو **ضروري** للـ hardware اللي بيستخدم الـ shadow-mask mechanism في الـ clock registers (شائع في TI وRockchip SoCs). بدونه، الـ write بيتجاهله الـ hardware صامتًا. دايمًا راجع الـ TRM للـ SoC قبل ما تكتب الـ clock flags.
## Phase 7: مصادر ومراجع

### المصادر الرسمية للـ Kernel

#### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| `Documentation/driver-api/clk.rst` | التوثيق الرسمي للـ Common Clock Framework — يشرح الـ API كامل والـ ops |
| `include/linux/clk-provider.h` | Header الرئيسي — فيه تعريف `struct clk_divider` و`struct clk_div_table` وكل الـ flags |
| `drivers/clk/clk-divider.c` | الملف الأساسي موضوع الدراسة |
| `drivers/clk/clk.c` | نواة الـ framework — فيه `clk_hw_register` و`clk_hw_round_rate` اللي بيتعملوا call من الـ divider |

الـ online version للتوثيق الرسمي:
- [The Common Clk Framework — Linux Kernel Documentation](https://docs.kernel.org/driver-api/clk.html)
- [kernel.org — Documentation/clk.txt (النسخة القديمة)](https://www.kernel.org/doc/Documentation/clk.txt)

---

### مقالات LWN.net

الـ LWN هو المرجع الأهم لفهم التاريخ التصميمي للـ framework:

| المقالة | الأهمية |
|---------|---------|
| [A common clock framework](https://lwn.net/Articles/472998/) | أول مقالة ناقشت الـ framework قبل ما يتدمج — فيها الـ design rationale الأصلي |
| [common clk framework](https://lwn.net/Articles/486841/) | مقالة LWN الأساسية اللي غطت دمج الـ framework في kernel 3.4 — لازم تُقرأ |
| [clk: dt: bindings for mux, divider & gate clocks](https://lwn.net/Articles/555242/) | غطت إضافة Device Tree bindings للـ divider والـ gate والـ mux clocks الأساسية |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | نقاش حول التوثيق الرسمي للـ framework |
| [clk: add si5351 i2c common clock driver](https://lwn.net/Articles/543320/) | مثال حقيقي لـ driver بيستخدم الـ clk-divider مع hardware فعلي |
| [The Common Clk Framework — static.lwn.net](https://static.lwn.net/kerneldoc/driver-api/clk.html) | نسخة LWN من التوثيق الرسمي |

---

### نقاشات الـ Mailing List والـ Patches المهمة

دي أبرز الـ patches اللي شكّلت الـ `clk-divider.c` زي ما هو دلوقتي:

#### الإضافة الأولى — Kernel 3.4 (2012)

- [PATCH v7 3/3: clk: basic clock hardware types](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00027.html) — Mike Turquette أضاف الـ `clk-divider.c` لأول مرة مع الـ `clk-gate.c` و`clk-mux.c`

#### Patches مهمة بعد كده

| الـ Patch | الأهمية |
|----------|---------|
| [clk: divider: replace bitfield width with mask](https://patchwork.kernel.org/project/linux-rockchip/patch/1416438943-11429-2-git-send-email-james.hogan@imgtec.com/) | غيّر الـ internal representation من width لـ mask |
| [clk: divider: Make generic for usage elsewhere](https://patchwork.kernel.org/project/linux-arm-msm/patch/1416442245-21967-3-git-send-email-sboyd@codeaurora.org/) | Stephen Boyd فصل الـ generic logic عن الـ register I/O — فكرة `divider_recalc_rate` و`divider_get_val` كـ EXPORT_SYMBOL_GPL |
| [clk: divider: read-only divider can propagate rate change](https://patchwork.kernel.org/project/linux-clk/patch/20180105170959.17266-2-jbrunet@baylibre.com/) | Jerome Brunet أضاف دعم `CLK_SET_RATE_PARENT` للـ read-only divider — ده اللي عمل `divider_ro_determine_rate` |
| [clk: divider: Switch from .round_rate to .determine_rate](https://patchwork.kernel.org/project/linux-clk/patch/20210627223959.188139-3-martin.blumenstingl@googlemail.com/) | التحول الأهم — استبدال `.round_rate` بـ `.determine_rate` |
| [PATCH: clk: divider: Introduce CLK_DIVIDER_ALLOW_ZERO flag](https://linux-kernel.vger.kernel.narkive.com/jtMyS6ZP/patch-clk-divider-introduce-clk-divider-allow-zero-flag) | إضافة الـ flag اللي بيمنع الـ WARN لما الـ divisor = 0 |
| [clk: clk-divider: add CLK_DIVIDER_ZERO_GATE support](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1531921023-18497-2-git-send-email-aisheng.dong@nxp.com/) | Dong Aisheng أضاف `CLK_DIVIDER_ZERO_GATE` |
| [clk: divider: fix rate calculation for fractional rates](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20131106161911.GA16735@n2100.arm.linux.org.uk/) | إصلاح حساب الـ rate للكسور — استخدام `DIV_ROUND_UP_ULL` |
| [PATCH: clk: clk-divider: round divider to valid value when table is used](https://lkml.iu.edu/hypermail/linux/kernel/1401.1/01133.html) | إصلاح الـ table lookup logic |
| [clk: dt: bindings for mux, divider & gate clocks (v2)](https://linux.kernel.narkive.com/OV6yp6s3/patch-v2-0-5-clk-dt-bindings-for-mux-divider-gate-clocks) | إضافة الـ DT bindings للـ basic clocks |

---

### الـ Commits الأساسية على Git

للبحث في تاريخ الملف مباشرة:

```bash
# تاريخ كامل للملف
git log --oneline drivers/clk/clk-divider.c

# أول commit أضاف الملف
git log --follow --diff-filter=A -- drivers/clk/clk-divider.c

# مقارنة النسخة الحالية بنسخة قديمة
git diff v3.4..v6.8 -- drivers/clk/clk-divider.c
```

**الـ commit الأصلي** اللي أضاف الملف:
- SHA: `b2476490ef11` — "clk: basic clock hardware types" — Mike Turquette، مارس 2012

---

### موارد elinux.org

| المورد | الرابط |
|--------|--------|
| Linux Clock Management Framework (ELC 2007 PDF) | [elinux.org — ELC 2007 Linux clock fmw](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) |
| Timing API Specification | [elinux.org — Timing API Specification](https://elinux.org/Timing_API_Specification) |

---

### مصادر Bootlin (Embedded Linux)

- [Common Clock Framework — How to Use It (ELCE 2013 PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) — شرح عملي ممتاز من Thomas Petazzoni و Maxime Ripard

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 3** — Character Drivers (فيه فهم عام للـ kernel APIs)
- الكتاب ده قديم (2005) وما بيغطيش الـ Common Clock Framework لأنه أضيف في 2012، لكنه لا يزال مرجع للمبادئ العامة
- [متاح مجاناً على kernel.org](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 11** — Timers and Time Management — قريب من مفهوم الـ clock management
- **الفصل 17** — Memory Management — مهم لفهم `kzalloc` و`devres_alloc` اللي بيتستخدموا في الـ divider
- مرجع ممتاز للـ kernel internals بشكل عام

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16** — Kernel Initialization — بيناقش الـ clock setup في الـ boot process
- بيغطي الـ platform-specific clock configuration اللي الـ clk-divider بيكون جزء منها

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 6** — Device Drivers — بيغطي تسجيل الـ drivers والـ device model اللي الـ devres فيه

---

### kernelnewbies.org

الـ kernelnewbies ما عندوش صفحة مخصصة للـ clk-divider، لكن في موارد مفيدة:

| المورد | الرابط |
|--------|--------|
| Linux 4.9 Changelog (فيه تحديثات على الـ clk subsystem) | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) |
| KernelGlossary — لفهم المصطلحات | [kernelnewbies.org/KernelGlossary](https://kernelnewbies.org/KernelGlossary) |
| KernelHacking — نقطة البداية للمساهمة | [kernelnewbies.org/KernelHacking](https://kernelnewbies.org/KernelHacking) |

---

### الـ Source Code References المباشرة

```
drivers/clk/clk-divider.c      # الملف الرئيسي
drivers/clk/clk.c              # نواة الـ framework
drivers/clk/clk-composite.c   # بيجمع divider + gate + mux مع بعض
drivers/clk/clk-fixed-factor.c # نوع تاني من الـ divider (fixed ratio)
include/linux/clk-provider.h   # كل التعريفات والـ macros
include/linux/clk.h            # الـ consumer API (clk_get, clk_set_rate, إلخ)
Documentation/driver-api/clk.rst
Documentation/devicetree/bindings/clock/clock-bindings.txt
Documentation/devicetree/bindings/clock/fixed-factor-clock.yaml
```

---

### Search Terms للبحث عن معلومات أكثر

لو حابب تعمق أكتر، دي أفضل الـ search terms:

```
linux kernel "clk_divider" "CLK_SET_RATE_PARENT"
linux "struct clk_divider" site:elixir.bootlin.com
linux clk framework "determine_rate" "round_rate" migration
linux "clk_hw_register_divider" device tree binding
linux kernel clock subsystem "divider_recalc_rate" EXPORT_SYMBOL_GPL
site:patchwork.kernel.org "clk: divider"
linux-clk@vger.kernel.org mailing list archive
linux kernel "CLK_DIVIDER_ROUND_CLOSEST" implementation
linux clk divider "HIWORD_MASK" Rockchip
linux clk divider "EVEN_INTEGERS" implementation
```

#### أدوات مفيدة للبحث في الـ Source Code

| الأداة | الرابط |
|--------|--------|
| Elixir Cross Referencer (لقراءة الـ source مع الـ cross-references) | [elixir.bootlin.com/linux/latest/source/drivers/clk/clk-divider.c](https://elixir.bootlin.com/linux/latest/source/drivers/clk/clk-divider.c) |
| kernel.googlesource.com | [clk-divider.c على googlesource](https://kernel.googlesource.com/pub/scm/linux/kernel/git/mark/linux/+/arch-timer/updates/drivers/clk/clk-divider.c) |
| Linux Kernel Driver DataBase | [cateee.net — CONFIG_COMMON_CLK](https://cateee.net/lkddb/web-lkddb/COMMON_CLK.html) |
| Patchwork linux-clk | [patchwork.kernel.org/project/linux-clk](https://patchwork.kernel.org/project/linux-clk/) |
## Phase 8: Writing simple module

### الفكرة

**`divider_recalc_rate`** هي function مُصدَّرة بـ `EXPORT_SYMBOL_GPL` من ملف `clk-divider.c`، وشغلتها إنها تحسب الـ rate الفعلية للـ divider clock بناءً على قيمة الـ register والـ parent rate. كل مرة أي driver يطلب إعادة حساب الـ clock rate، بتتنادي. ده بيخليها نقطة ممتازة للـ kprobe — نشوف مين بيطلب recalc وبأي parent rate وإيه الـ divider المحسوب.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on divider_recalc_rate()
 * Logs: clock name, parent_rate, raw register val, computed rate
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/clk-provider.h>  /* struct clk_hw, clk_hw_get_name() */

/* ──────────────────────────────────────────────
 * pre-handler: يتنادى قبل ما divider_recalc_rate تبدأ
 * ────────────────────────────────────────────── */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Calling convention x86-64:
     *   rdi = hw          (struct clk_hw *)
     *   rsi = parent_rate (unsigned long)
     *   rdx = val         (unsigned int)
     *
     * On arm64 replace regs->di/si/dx with regs->regs[0/1/2]
     */
    struct clk_hw *hw          = (struct clk_hw *)regs->di;
    unsigned long  parent_rate = (unsigned long)regs->si;
    unsigned int   val         = (unsigned int)regs->dx;
    const char    *name        = "(unknown)";

    /* clk_hw_get_name() safe to call here — returns const char* */
    if (hw)
        name = clk_hw_get_name(hw);

    pr_info("clk-divider probe: clk=%s parent_rate=%lu reg_val=%u\n",
            name, parent_rate, val);

    return 0; /* 0 = let the original function continue normally */
}

/* ──────────────────────────────────────────────
 * post-handler: يتنادى بعد ما الـ function ترجع
 * ────────────────────────────────────────────── */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * rax holds the return value (computed rate in Hz)
     * نطبعها عشان نشوف الـ divider أثّر عليها إزاي
     */
    unsigned long computed_rate = (unsigned long)regs->ax;

    pr_info("clk-divider probe: => computed_rate=%lu Hz\n", computed_rate);
}

/* ──────────────────────────────────────────────
 * kprobe struct — بنحدد فيه اسم الـ symbol اللي هنعمله hook
 * ────────────────────────────────────────────── */
static struct kprobe kp = {
    .symbol_name = "divider_recalc_rate",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ──────────────────────────────────────────────
 * module_init: بنسجل الـ kprobe
 * ────────────────────────────────────────────── */
static int __init clk_divider_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("clk-divider probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("clk-divider probe: planted at %p\n", kp.addr);
    return 0;
}

/* ──────────────────────────────────────────────
 * module_exit: بنشيل الـ kprobe لازم نعمل كده
 * عشان لو الـ module اتشال والـ kprobe لسه شايل،
 * الـ kernel هيحاول ينفذ handler مش موجود → kernel panic
 * ────────────────────────────────────────────── */
static void __exit clk_divider_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("clk-divider probe: removed\n");
}

module_init(clk_divider_probe_init);
module_exit(clk_divider_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on divider_recalc_rate to trace clock divider recalculation");
```

---

### Makefile للـ Module

```makefile
obj-m += clk_divider_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل
make
sudo insmod clk_divider_probe.ko
sudo dmesg | grep "clk-divider probe"

# لو عايز تشوف أكتر نشاط، شغّل أي عملية بتغير الـ clock rate زي:
# sudo cpupower frequency-set -f 1GHz   (على x86 مع cpufreq)

# إزالة الـ module
sudo rmmod clk_divider_probe
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` / `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info`, `pr_err` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/clk-provider.h` | `struct clk_hw`, `clk_hw_get_name()` عشان نطبع اسم الـ clock |

#### الـ `handler_pre`

**الـ pre-handler** بيتنادى قبل تنفيذ الـ function الأصلية مباشرةً، ده بيخلينا نشوف القيم اللي دخلت الـ function (الـ arguments) زي `parent_rate` و`val` اللي هو قيمة الـ register الخام. بنستخدم `regs->di/si/dx` على x86-64 لأنها الـ registers اللي بيمررها الـ calling convention لأول ٣ arguments.

#### الـ `handler_post`

**الـ post-handler** بيتنادى بعد ما الـ function ترجع وقبل ما الـ caller يستكمل، والـ return value بيكون في `rax` على x86-64. ده مهم عشان نشوف الـ rate المحسوبة النهائية مش بس الـ input.

#### الـ `kprobe` struct

بنحدد فيه `symbol_name` بالاسم الـ text بدل العنوان الـ numeric — الـ kernel بيعمل resolve للعنوان وقت الـ registration. لو الـ symbol مش موجود أو مش مُصدَّر، `register_kprobe` بترجع error.

#### الـ `module_init`

بنسجل الـ kprobe هنا وبنطبع العنوان اللي وصله الـ kernel بعد الـ resolve، ده مفيد للـ debugging ونتأكد إن الـ hook اتحط صح.

#### الـ `module_exit`

**لازم** نعمل `unregister_kprobe` هنا. لو شلنا الـ module من غير ما نشيل الـ kprobe، الـ kernel هيفضل عنده pointer لـ handler code مش موجودة في الميموري → kernel panic فوري في أول recalc بعد كده.
