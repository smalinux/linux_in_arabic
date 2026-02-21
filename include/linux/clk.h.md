## Phase 1: الصورة الكبيرة

# `clk.h`

> **PATH**: `include/linux/clk.h`
> **Subsystem**: Clock Framework / CCF (Common Clock Framework)
> **الوظيفة الأساسية**: واجهة المستخدم (Consumer API) لنظام إدارة الساعات في Linux kernel — تتيح لأي driver طلب ساعة، تشغيلها، إيقافها، وتغيير ترددها.

---

### ما هي المشكلة اللي هذا الملف يحلها؟

تخيل إنك بتبني جهاز إلكتروني — موبايل مثلاً. جوّاه عشرات المكوّنات: CPU، GPU، كاميرا، USB، Wi-Fi، شاشة... كل واحد منهم يحتاج **ساعة (clock)** — يعني إشارة كهربائية بتدق بتردد معيّن — عشان يشتغل.

المشكلة؟ الساعات مش منفصلة. هي متداخلة زي **شجرة**:

```
[Crystal / Oscillator]
        |
      [PLL]          ← يضاعف التردد
     /     \
  [Divider] [Mux]   ← يقسّم أو يختار مصدر
     |         |
  [CPU CLK] [USB CLK]
```

لو driver الكاميرا طلب تغيير تردد ساعته، ممكن يأثر على USB بدون ما يدري. ولو 3 drivers كلهم شغّالوا نفس الساعة وواحد فيهم قال "اطفيها"، هل تطفيها فعلاً؟ لأ — لازم الكل يوافق.

**الـ CCF (Common Clock Framework)** جاء عشان يحل هذه الفوضى. وملف `clk.h` هو **الواجهة اللي أي driver يستخدمها** للتعامل مع هذا النظام — بدون ما يحتاج يعرف تفاصيل الـ hardware.

---

### القصة الكاملة بطريقة ELI5

فكر في **clk.h** كمنيو مطعم. أنت (الـ driver) ما تحتاج تعرف كيف يطبخون — بس تطلب من المنيو.

المنيو يحتوي على:

**1. طلب الساعة (Get)**
```c
struct clk *clk_get(struct device *dev, const char *id);
// "أعطني ساعة اسمها uart_clk لهذا الجهاز"
```
زي ما تطلب طاولة في المطعم — الـ kernel يبحث لك عن الساعة الصح بناءً على جهازك واسم الاتصال.

**2. التحضير والتشغيل (Prepare + Enable)**
```c
clk_prepare(clk);   // تجهيز — ممكن ينام (قد يحتاج وقت)
clk_enable(clk);    // تشغيل — سريع، ممكن من atomic context
// أو مجمّع:
clk_prepare_enable(clk);
```
لماذا خطوتين؟ لأن بعض الـ hardware تحتاج وقت للتجهيز (إيقاظ PLL مثلاً) — هذا لا يصلح من interrupt context. لكن التشغيل الفعلي (flip a bit) سريع ويصلح من أي مكان.

**3. التحكم في التردد (Rate)**
```c
clk_get_rate(clk);                    // كم تردده الحالي؟
clk_round_rate(clk, rate);            // لو طلبت X، سأعطيك Y فعلاً
clk_set_rate(clk, rate);              // غيّر التردد
clk_set_rate_range(clk, min, max);    // حدد نطاق مسموح
```

**4. الإشعار عند تغيير التردد (Notifiers)**
```c
clk_notifier_register(clk, &nb);
```
لو ساعة USB ستتغير، الـ driver يسجّل نفسه عشان يعرف مسبقاً (PRE_RATE_CHANGE) ويستعد، وبعدها (POST_RATE_CHANGE) يأكد الأمر.

**5. الـ Bulk APIs — طلب أكثر من ساعة دفعة واحدة**
```c
struct clk_bulk_data clks[] = {
    { .id = "axi" },
    { .id = "apb" },
    { .id = "ref" },
};
clk_bulk_get(dev, 3, clks);
clk_bulk_prepare_enable(3, clks);
```
بدل ما تطلب كل ساعة لوحدها — تطلبهم كلهم مرة وحدة. سهولة وأقل كود.

**6. الـ devm variants — إدارة تلقائية**
```c
devm_clk_get(dev, "uart");
// الساعة ستتحرر تلقائياً لما الـ device يُفصل
```
**الـ devm** يعني "device-managed" — مثل smart pointer — ما تحتاج تتذكر تحرر الساعة يدوياً.

---

### الفرق بين Consumer و Provider

| الجانب | الملف | الدور |
|--------|-------|-------|
| **Consumer API** | `include/linux/clk.h` | الـ driver اللي *يستخدم* الساعة |
| **Provider API** | `include/linux/clk-provider.h` | الـ driver اللي *يوفّر* الساعة (PLL driver مثلاً) |
| **Core implementation** | `drivers/clk/clk.c` | المحرك الداخلي للنظام |
| **Device lookup** | `drivers/clk/clkdev.c` | ربط الأجهزة بالساعات |

`clk.h` هو **واجهة المستخدم فقط** — ما فيه أي تفاصيل hardware.

---

### هيكل الـ API — مجموعات منطقية

```
clk.h APIs
├── Lifecycle
│   ├── clk_get / clk_put
│   ├── devm_clk_get / devm_clk_put
│   └── of_clk_get (من Device Tree)
│
├── Enable/Disable (atomic-safe)
│   ├── clk_enable / clk_disable
│   └── clk_bulk_enable / clk_bulk_disable
│
├── Prepare/Unprepare (sleepable)
│   ├── clk_prepare / clk_unprepare
│   └── clk_bulk_prepare / clk_bulk_unprepare
│
├── Combined helpers
│   ├── clk_prepare_enable / clk_disable_unprepare
│   └── clk_bulk_prepare_enable / clk_bulk_disable_unprepare
│
├── Rate Control
│   ├── clk_get_rate / clk_set_rate
│   ├── clk_round_rate
│   ├── clk_set_rate_range / clk_set_min_rate / clk_set_max_rate
│   ├── clk_set_rate_exclusive / clk_rate_exclusive_get/put
│   └── clk_drop_range
│
├── Topology
│   ├── clk_get_parent / clk_set_parent
│   ├── clk_has_parent / clk_is_match
│   └── clk_get_sys
│
├── Signal Properties
│   ├── clk_get_phase / clk_set_phase
│   ├── clk_get_scaled_duty_cycle / clk_set_duty_cycle
│   └── clk_get_accuracy
│
├── Notifications
│   └── clk_notifier_register / clk_notifier_unregister
│
└── Power Management
    ├── clk_save_context
    └── clk_restore_context
```

---

### الـ Structs الأساسية في الملف

**`struct clk_notifier`** — يربط ساعة بقائمة مستمعين:
```c
struct clk_notifier {
    struct clk              *clk;           // أي ساعة
    struct srcu_notifier_head notifier_head; // قائمة المستمعين
    struct list_head        node;           // ربط في القائمة العامة
};
```

**`struct clk_notifier_data`** — البيانات اللي تُرسل للمستمع لما يتغير التردد:
```c
struct clk_notifier_data {
    struct clk    *clk;       // الساعة المتغيّرة
    unsigned long old_rate;   // التردد القديم
    unsigned long new_rate;   // التردد الجديد
};
```

**`struct clk_bulk_data`** — لطلب أكثر من ساعة:
```c
struct clk_bulk_data {
    const char *id;   // اسم الساعة
    struct clk *clk;  // الـ pointer بعد الحصول عليها
};
```

---

### Compile-time Guards — متى يُفعَّل كل شيء؟

الملف يستخدم 3 guards:

| Guard | ماذا يحمي |
|-------|-----------|
| `CONFIG_COMMON_CLK` | notifiers، phase، duty cycle، accuracy، save/restore context |
| `CONFIG_HAVE_CLK_PREPARE` | clk_prepare / clk_unprepare |
| `CONFIG_HAVE_CLK` | كل الـ get/put/enable/disable/rate APIs |

لو الـ config مش مفعّل، الـ functions تصبح `static inline` ترجع قيمة dummy (0 أو NULL) — بدون أي overhead.

---

### الملفات المرتبطة — ما يجب أن تعرفه

**Core Framework:**
- `include/linux/clk-provider.h` — واجهة صانع الساعة (من يكتب PLL driver أو divider driver)
- `drivers/clk/clk.c` — التنفيذ الكامل للـ CCF، الشجرة، الـ locking، الـ rate propagation
- `drivers/clk/clkdev.c` — lookup table: من يربط `(device, id)` بـ `struct clk`
- `drivers/clk/clk-devres.c` — تنفيذ كل `devm_clk_*` functions

**Bulk operations:**
- `drivers/clk/clk-bulk.c` — تنفيذ `clk_bulk_*` functions

**Device Tree integration:**
- `include/linux/of_clk.h` — `of_clk_get`, `of_clk_get_by_name`
- `drivers/clk/clk-conf.c` — تطبيق إعدادات الساعة من Device Tree

**Hardware drivers (أمثلة):**
- `drivers/clk/clk-fixed-rate.c` — ساعة بتردد ثابت
- `drivers/clk/clk-divider.c` — قسمة التردد
- `drivers/clk/clk-mux.c` — اختيار مصدر
- `drivers/clk/clk-gate.c` — تشغيل/إيقاف
- `drivers/clk/clk-pll.c` (في مجلدات platforms) — ضرب التردد

**Platform-specific:**
- `drivers/clk/*/` — مجلدات خاصة بكل معالج (Qualcomm, MediaTek, Rockchip, ...)

---

### ملخص — لماذا هذا الملف مهم؟

`clk.h` هو **عقد التعامل** بين أي driver وبين نظام إدارة الساعات. بدونه، كل driver كان لازم يكتب كود hardware خاص به — فوضى كاملة. بفضل هذا الـ header، أي driver في أي معالج في العالم يقدر يقول:

```c
clk = devm_clk_get_enabled(&pdev->dev, "uart");
// وبكذا الساعة شغّالة وجاهزة
```

بدون ما يعرف شيئاً عن الـ PLL أو الـ divider أو أي حديد تحته.
## Phase 2: شرح الـ Clock Framework (CCF)

---

### لماذا يوجد هذا الـ Framework أصلاً؟ — المشكلة

تخيل إنك بتبني موبايل مثلاً Qualcomm Snapdragon. جوّاه في الـ SoC عندك:
- CPU cores تشتغل على 2.8 GHz
- GPU تشتغل على 840 MHz
- USB controller تحتاج 480 MHz
- UART للـ debug تحتاج 115200 baud (مشتقة من 24 MHz)
- Camera ISP تحتاج 400 MHz
- DDR memory controller تحتاج 1066 MHz
- Bluetooth/WiFi تحتاج 40 MHz reference clock
- وعشرات غيرهم...

كل هذه الأجهزة تحتاج **clock signals** — نبضات كهربية بترددات محددة — عشان تشتغل. بدون الـ clock، الـ digital circuit ما بتعرف "امتى تحرك بياناتها".

**قبل الـ CCF**، كل SoC vendor كان يكتب clock management code خاص بيه. النتيجة؟

- **كود مكرر** في كل مكان: كل ARM SoC (Samsung Exynos, TI OMAP, NXP i.MX, Qualcomm MSM) عنده نفس المنطق مكتوب بشكل مختلف
- **لا يوجد abstraction**: الـ driver بيتكلم مباشرة مع الـ hardware registers
- **لا يوجد reference counting**: لو driver A و driver B يستخدمون نفس الـ clock، ومين يطفيها؟
- **لا يوجد tree traversal**: لو عايز تغير rate لـ clock، محتاج تعرف يدوياً إيه الـ clocks المتأثرة
- **لا يوجد power management**: ما في طريقة موحدة لمعرفة إيه الـ clocks مش محتاجها وتطفيها

---

### الحل — الـ Common Clock Framework (CCF)

**الـ CCF** أُدخل للـ kernel في 2012 بواسطة Mike Turquette من Linaro. الفكرة الأساسية:

> كل الـ clocks في أي SoC هي في الحقيقة **شجرة (tree)** — كل clock ليها parent تاخد منه الـ signal وتعمل عليه عمليات (تقسيم، ضرب، تشغيل/إيقاف). نبني abstraction موحد يمثل هذه الشجرة ونوفر API موحد للـ consumers.

الـ CCF يقسم العالم لجزئين:

| الطرف | اسمه في الكود | دوره |
|-------|--------------|------|
| من يوفر الـ clock (hardware) | **Provider / clk_hw** | يسجّل الـ clock في النظام |
| من يستهلك الـ clock (driver) | **Consumer** | يطلب الـ clock بـ `clk_get()` |
| النظام اللي بينظم كل شيء | **CCF core** (`clk.c`) | يربط الاثنين، يدير الـ tree |

---

### تشبيه من الواقع — شبكة الكهرباء

تخيل **شبكة توزيع الكهرباء**:

```
[محطة توليد كهرباء = Crystal Oscillator / PLL]
         |
    [محول كهرباء كبير = PLL Divider]
    /           \
[محول صغير]  [محول صغير]
   |                |
[بيوت]          [مصانع]
```

- **محطة التوليد** = الـ oscillator الأساسي (مثلاً 24 MHz crystal)
- **المحولات** = الـ PLLs والـ dividers اللي بتعمل multiply وdivide
- **البيوت والمصانع** = الـ drivers (USB, Camera, CPU)

لما بيت (driver) يحتاج كهرباء (clock)، ما بيروحش المحطة مباشرة — بيتصل بشبكة التوزيع اللي بتوديله الكهرباء (CCF).

لما المحطة تغير الجهد (rate change)، كل من على الشبكة يتأثر — CCF بيدير هذا التأثير أوتوماتيكياً.

لما ما في أحد يستخدم فرع (لا يوجد consumer)، CCF يقدر يفصل ذلك الفرع لتوفير الطاقة (**clock gating**).

---

### المفاهيم الأساسية بعمق

#### 1. الـ Clock Rate
الـ **rate** هو عدد النبضات في الثانية، بالـ Hz. مثلاً:
- 24,000,000 Hz = 24 MHz (crystal oscillator نموذجي)
- 2,800,000,000 Hz = 2.8 GHz (CPU)

```c
unsigned long clk_get_rate(struct clk *clk); /* بيرجع الـ rate بالـ Hz */
int clk_set_rate(struct clk *clk, unsigned long rate);
long clk_round_rate(struct clk *clk, unsigned long rate); /* شوف ايش ممكن فعلاً */
```

**لماذا `clk_round_rate`؟** لأن الـ hardware ما تقدر تعطيك أي rate تطلبه — بتقدر بس تقسم بأعداد صحيحة. لو عندك 24 MHz وعايز 10 MHz، أقرب شيء ممكن هو 8 MHz (24/3) أو 12 MHz (24/2).

#### 2. الـ Clock Gating (التشغيل والإيقاف)
**الـ clock gating** هو ببساطة فتح أو إغلاق مرور نبضات الـ clock. في الـ hardware هو gate circuit:

```
clock_in ──┐
           AND ──── clock_out
enable ────┘
```

لما `enable = 1`، الـ clock يعدي. لما `enable = 0`، الـ output دايماً صفر = جهاز متوقف.

في الـ CCF عندنا **مرحلتين** للتشغيل — هذا مهم جداً:

```
clk_prepare()   ← slow path, يقدر ينام (sleep) — يجهّز الـ clock (مثلاً يشغّل PLL ويستنى يستقر)
clk_enable()    ← fast path, atomic — يفتح الـ gate فعلياً

clk_disable()   ← fast path, atomic
clk_unprepare() ← slow path
```

**لماذا مرحلتان؟** لأن بعض الـ clocks (خصوصاً PLLs) تحتاج وقت lock time قد يكون ميلي ثانية — ما ينفع تستنى في الـ atomic context (interrupt handler). فـ:
- `clk_prepare()` = تشغّل الـ PLL وتستنى في سياق يقدر ينام
- `clk_enable()` = تفتح الـ gate بسرعة في الـ atomic context

الدالة المريحة:
```c
/* تجمع prepare + enable في خطوة واحدة (non-atomic context فقط) */
static inline int clk_prepare_enable(struct clk *clk)

/* تجمع disable + unprepare */
static inline void clk_disable_unprepare(struct clk *clk)
```

#### 3. الـ Reference Counting
الـ CCF يحتفظ بعداد لكل clock:

```
struct clk_core {
    unsigned int enable_count;   /* كم consumer يستخدمونها الآن */
    unsigned int prepare_count;  /* كم consumer عاملين prepare */
    ...
}
```

لما `enable_count` يوصل صفر، الـ CCF يطفي الـ clock أوتوماتيكياً. هذا يحل مشكلة "مين يطفي الـ shared clock".

#### 4. الـ Parent و الـ Clock Tree
كل clock (ما عدا الجذور مثل الـ crystal) لها **parent** — المصدر اللي تاخد منه الـ signal وتعالجه.

```
struct clk_core {
    struct clk_core  *parent;        /* الـ parent الحالي */
    struct clk_parent_map *parents;  /* قائمة الـ parents الممكنة (للـ MUX) */
    u8 num_parents;                  /* عدد الـ parents الممكنة */
    struct hlist_head children;      /* الـ clocks اللي هي children لهذه */
}
```

```c
struct clk *clk_get_parent(struct clk *clk);
int clk_set_parent(struct clk *clk, struct clk *parent); /* للـ MUX clocks */
bool clk_has_parent(const struct clk *clk, const struct clk *parent);
```

#### 5. الـ PLL (Phase-Locked Loop)
الـ **PLL** هو الـ building block الأهم. يأخذ input clock ويضاعفه:

```
input 24 MHz → [PLL: ×100] → 2400 MHz → [Divider: /2] → 1200 MHz CPU

الـ PLL داخلياً:
Reference (24MHz) → [Phase Detector] → [VCO] → output
                          ↑                        |
                    [Divider /N] ←──────────────────┘
```

- **VCO** (Voltage Controlled Oscillator): مذبذب تردده يتحكم فيه الجهد
- **Phase Detector**: يقارن الـ reference مع feedback
- **Lock**: لما الـ output/N = reference بالضبط

الـ PLL يحتاج **lock time** (عادة 50-500 microseconds) حتى يستقر على الـ target frequency — ولهذا السبب تحديداً الـ `clk_prepare()` تنفصل عن `clk_enable()`.

#### 6. الـ Divider
الأبسط: يأخذ الـ input ويقسم:

```
input 1200 MHz → [Divider /4] → 300 MHz
```

في الـ hardware غالباً register فيه القيمة:
```
REG[3:0] = 0x4  →  output = input / 4
```

#### 7. الـ MUX
يختار من أي source يجي الـ clock:

```
src_A (PLL1: 1200 MHz) ──┐
src_B (PLL2:  800 MHz) ──┤ MUX ──→ output
src_C (OSC:    24 MHz) ──┘
        ↑
    register bit يختار
```

هذا مهم للـ power management: لما تريد تخفف الاستهلاك، تحوّل لـ source أبطأ (24 MHz) وتطفي الـ PLL.

#### 8. الـ Notifier System
لما rate الـ clock يتغير، بعض الـ drivers محتاجين يعلمون مسبقاً عشان يتكيفوا (مثلاً UART محتاج يُعيد حساب الـ baud rate divisor):

```c
/* ثلاثة أنواع من الـ notifications */
#define PRE_RATE_CHANGE   BIT(0)  /* قبل التغيير — قرر إذا تقبل أو ترفض */
#define POST_RATE_CHANGE  BIT(1)  /* بعد التغيير بنجاح */
#define ABORT_RATE_CHANGE BIT(2)  /* فشل التغيير — ارجع لحالتك الأصلية */

int clk_notifier_register(struct clk *clk, struct notifier_block *nb);
```

```
struct clk_notifier {
    struct clk              *clk;           /* الـ clock المراقب */
    struct srcu_notifier_head notifier_head; /* قائمة الـ callbacks */
    struct list_head         node;
};

struct clk_notifier_data {
    struct clk    *clk;
    unsigned long  old_rate;  /* الـ rate قبل التغيير */
    unsigned long  new_rate;  /* الـ rate بعد التغيير */
};
```

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                                  │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  USB Driver  │  │  GPU Driver  │  │ Camera Driver│  ← Consumers │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                  │                       │
│         │  clk_get()      │  clk_get()       │  clk_get()           │
│         │  clk_enable()   │  clk_set_rate()  │  clk_prepare_enable()│
│         ▼                 ▼                  ▼                       │
│  ┌─────────────────────────────────────────────────────┐            │
│  │              CCF Consumer API  (clk.h)               │            │
│  │  clk_get / clk_put / clk_enable / clk_disable       │            │
│  │  clk_set_rate / clk_get_rate / clk_set_parent       │            │
│  │  clk_prepare / clk_unprepare / clk_round_rate       │            │
│  └─────────────────────────┬───────────────────────────┘            │
│                             │                                        │
│  ┌─────────────────────────▼───────────────────────────┐            │
│  │              CCF Core  (drivers/clk/clk.c)           │            │
│  │                                                      │            │
│  │  ┌─────────────────────────────────────────────┐    │            │
│  │  │              clk_core (internal struct)      │    │            │
│  │  │  name, rate, parent, children, enable_count │    │            │
│  │  │  ops, flags, min_rate, max_rate, phase      │    │            │
│  │  └─────────────────────────────────────────────┘    │            │
│  │                                                      │            │
│  │  • Clock Tree traversal                              │            │
│  │  • Reference counting (enable_count, prepare_count) │            │
│  │  • Rate propagation up/down the tree                │            │
│  │  • Orphan clock resolution                          │            │
│  │  • clk_hashtable (lookup by name)                   │            │
│  │  • clk_root_list / clk_orphan_list                  │            │
│  │  • Notifier chain management                        │            │
│  │  • debugfs integration                              │            │
│  └─────────────────────────┬───────────────────────────┘            │
│                             │                                        │
│  ┌─────────────────────────▼───────────────────────────┐            │
│  │           Provider API  (clk-provider.h)             │            │
│  │                                                      │            │
│  │  struct clk_ops {                                    │            │
│  │    .prepare / .unprepare                            │            │
│  │    .enable  / .disable                              │            │
│  │    .set_rate / .round_rate / .recalc_rate           │            │
│  │    .set_parent / .get_parent                        │            │
│  │  }                                                   │            │
│  └─────────────────────────┬───────────────────────────┘            │
│                             │  clk_hw_register()                    │
│  ┌──────────────────────────▼──────────────────────────┐            │
│  │            Clock Providers (Hardware Implementations) │            │
│  │                                                      │            │
│  │  ┌───────────┐ ┌──────────┐ ┌────────┐ ┌────────┐  │            │
│  │  │clk-fixed  │ │clk-divid.│ │clk-mux │ │clk-gate│  │  ← Generic│
│  │  └───────────┘ └──────────┘ └────────┘ └────────┘  │            │
│  │                                                      │            │
│  │  ┌──────────────────────────────────────────────┐   │            │
│  │  │  SoC-specific drivers (e.g. clk-qcom.c)      │   │  ← Vendor  │
│  │  │  يسجّل الـ PLLs والـ clocks الخاصة بالـ SoC  │   │            │
│  │  └──────────────────────────────────────────────┘   │            │
│  └──────────────────────────┬───────────────────────────┘           │
└─────────────────────────────┼───────────────────────────────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Real Hardware     │
                    │  (PLL, Divider,     │
                    │   Gate registers)   │
                    └────────────────────┘
```

---

### شجرة الـ Clocks — مثال واقعي

هذا مثال مبسّط يشابه ما تجده في SoC مثل i.MX8 أو Snapdragon:

```
                [24 MHz Crystal OSC]          ← clk_root_list
                         │
              ┌──────────┼──────────┐
              │          │          │
          [PLL_CPU]  [PLL_BUS]  [PLL_USB]    ← Fixed-Rate PLLs
         1200 MHz    800 MHz     480 MHz
              │          │          │
           [/2]        [/4]        [/1]       ← Dividers
          600 MHz    200 MHz    480 MHz
              │          │          │
          [CPU MUX]──────┘          │         ← MUX (يختار src)
              │       [BUS_CLK]     │
          [CPU_CLK]   200 MHz   [USB_CLK]
          600 MHz         │        480 MHz
              │      ┌────┴────┐       │
           [CPU]  [UART_CLK][SPI_CLK] [USB]   ← Consumers
                  200 MHz   100 MHz
                     │
                  [UART Driver]
```

في الكود يمثّل هذا هيكل بيانات:

```
clk_root_list:
  └─ osc_24m (rate=24MHz, parent=NULL, children=[pll_cpu, pll_bus, pll_usb])

clk_hashtable:
  "pll_cpu" → clk_core { rate=1200MHz, parent=osc_24m, children=[div2] }
  "div2"    → clk_core { rate=600MHz,  parent=pll_cpu, children=[cpu_mux] }
  "cpu_mux" → clk_core { rate=600MHz,  parent=div2,    children=[cpu_clk] }
  "cpu_clk" → clk_core { rate=600MHz,  parent=cpu_mux, enable_count=1 }
  ...
```

---

### العلاقة بين الـ Structs

هذا الرسم يوضح كيف تتربط الـ structs ببعض:

```
┌─────────────────────────────────────────────────────────────┐
│  struct clk  (الـ consumer's handle — في clk.c)             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  core ──────────────────────────────────────────┐   │    │
│  │  dev                                            │   │    │
│  │  dev_id (اسم الـ device)                        │   │    │
│  │  con_id (اسم الـ connection "uart", "spi")      │   │    │
│  │  min_rate, max_rate  (قيود هذا الـ consumer)    │   │    │
│  │  exclusive_count                                │   │    │
│  └─────────────────────────────────────────────────┘   │    │
└──────────────────────────────────────┬──────────────────┘    │
                                       │ يشير لـ               │
                                       ▼                        │
┌─────────────────────────────────────────────────────────────┐│
│  struct clk_core  (الـ actual clock — في clk.c)              ││
│  ┌──────────────────────────────────────────────────────┐   ││
│  │  name         "pll_cpu"                              │   ││
│  │  ops ─────────────────────────────────────────────┐ │   ││
│  │  hw ──────────────────────────────────────────┐   │ │   ││
│  │  parent ──────────────────────────────────┐   │   │ │   ││
│  │  parents[]  (قائمة الـ parents الممكنة)   │   │   │ │   ││
│  │  num_parents                              │   │   │ │   ││
│  │  rate         1200000000UL               │   │   │ │   ││
│  │  enable_count 2                           │   │   │ │   ││
│  │  prepare_count 2                          │   │   │ │   ││
│  │  flags        CLK_SET_RATE_PARENT         │   │   │ │   ││
│  │  children     ─────────────────────────────────│─►[div2]││
│  │  clks (قائمة الـ consumer handles)        │   │   │ │   ││
│  └──────────────────────────────────────────┘│  │   │ │   ││
└─────────────────────────────────────┬────────┘  │   │ │   ││
              يشير لـ parent's clk_core│           │   │ │   ││
                                      ▼            │   │ │   ││
                               [osc_24m clk_core]  │   │ │   ││
                                                   │   │ │   ││
                                                   ▼   │ │   ││
┌─────────────────────────────────────────────────┐│   │ │   ││
│  struct clk_hw  (رابط الـ provider بالـ CCF)     ││   │ │   ││
│  ┌──────────────────────────────────────────┐  ││   │ │   ││
│  │  core ─────────────────────────────────── │──┘│   │ │   ││
│  │  clk (consumer handle خاص بالـ provider) │  │   │ │   ││
│  └──────────────────────────────────────────┘  │   │ │   ││
└─────────────────────────────────────────────────┘   │ │   ││
                                                      ▼ │   ││
┌──────────────────────────────────────────────────────┐│   ││
│  struct clk_ops  (الـ vtable — وظائف الـ hardware)   ││   ││
│  ┌──────────────────────────────────────────────────┐ ││   ││
│  │  .recalc_rate()  ← احسب الـ rate من الـ hardware │ ││   ││
│  │  .round_rate()   ← أقرب rate ممكن               │ ││   ││
│  │  .set_rate()     ← اكتب في الـ registers         │ ││   ││
│  │  .prepare()      ← شغّل الـ PLL واستنى lock      │ ││   ││
│  │  .unprepare()    ← أوقف الـ PLL                  │ ││   ││
│  │  .enable()       ← افتح الـ gate                 │ ││   ││
│  │  .disable()      ← أقفل الـ gate                 │ ││   ││
│  │  .set_parent()   ← غيّر الـ MUX                  │ ││   ││
│  │  .get_parent()   ← اقرأ الـ MUX الحالي           │ ││   ││
│  └──────────────────────────────────────────────────┘ ││   ││
└───────────────────────────────────────────────────────┘│   ││
                                                         │   ││
└────────────────────────────────────────────────────────┘   ││
                                                              ││
└─────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

---

### الـ Locking Strategy في الـ CCF Core

في `clk.c` تجد مباشرة في أوله:

```c
static DEFINE_SPINLOCK(enable_lock);   /* للـ enable/disable — atomic safe */
static DEFINE_MUTEX(prepare_lock);     /* للـ prepare/rate change — قد ينام */
```

**لماذا spinlock للـ enable؟** لأن `clk_enable()` يُستدعى من الـ atomic context (interrupt handlers). الـ spinlock لا ينام.

**لماذا mutex للـ prepare؟** لأن `clk_prepare()` قد يحتاج ينتظر الـ PLL يستقر — ينام — فيحتاج mutex.

هذا الفصل بين سياقي الـ locking هو أحد أذكى قرارات التصميم في الـ CCF.

---

### الـ Orphan Clocks

عند boot، قد يُسجَّل clock قبل أن يُسجَّل parent-ه (ترتيب init غير محدد):

```c
static HLIST_HEAD(clk_root_list);    /* clocks عندها parent */
static HLIST_HEAD(clk_orphan_list);  /* clocks ما لقت parent-ها بعد */
```

عندما يُسجَّل clock جديد، الـ CCF يمشي على الـ orphan list ويتحقق: "هل أحد اليتامى ينتظر هذا الـ clock كـ parent؟" — إذا نعم، يُعيد تنظيم الشجرة أوتوماتيكياً.

---

### الـ Bulk API — للـ Devices اللي تحتاج عدة Clocks

بعض الـ devices محتاجة عدة clocks في نفس الوقت (مثلاً camera تحتاج core clock وpixel clock وbus clock):

```c
struct clk_bulk_data clks[] = {
    { .id = "core" },   /* اسم الـ clock في الـ device tree */
    { .id = "pixel" },
    { .id = "bus" },
};

/* بدل ما تكتب clk_get ثلاث مرات */
clk_bulk_get(dev, ARRAY_SIZE(clks), clks);
clk_bulk_prepare_enable(ARRAY_SIZE(clks), clks);

/* وعند الانتهاء */
clk_bulk_disable_unprepare(ARRAY_SIZE(clks), clks);
clk_bulk_put(ARRAY_SIZE(clks), clks);
```

---

### الـ devm_ Variants — إدارة تلقائية للموارد

أي function بتبدأ بـ `devm_` تعني "device-managed" — الـ resource يُحرَّر أوتوماتيكياً لما الـ device تُفصل (unbind):

```c
/* بدون devm — المبرمج مسؤول عن التنظيف */
struct clk *clk = clk_get(dev, "uart");
/* ... لازم تتذكر clk_put(clk) في .remove() */

/* مع devm — kernel يتكفل بالتنظيف */
struct clk *clk = devm_clk_get(dev, "uart");
/* لما الـ device تُفصل، الـ clk_put يُستدعى أوتوماتيكياً */
```

---

### الـ Rate Range و Exclusive Control

أحياناً driver لا يريد rate محددة بس يريد range مقبولة:

```c
/* يضمن إن الـ clock تشتغل بين 100 MHz و 500 MHz */
clk_set_rate_range(clk, 100000000UL, 500000000UL);

/* يتخلى عن القيود */
clk_drop_range(clk); /* = clk_set_rate_range(clk, 0, ULONG_MAX) */
```

وأحياناً driver يريد **السيطرة الكاملة** على الـ rate — لا أحد غيره يغيرها:

```c
/* احجز حق التحكم المنفرد */
clk_rate_exclusive_get(clk);

/* الآن لو أي driver آخر حاول clk_set_rate() — سيفشل */

/* حرر السيطرة */
clk_rate_exclusive_put(clk);
```

هذا مهم مثلاً لـ audio subsystem: لما تشغّل موسيقى بـ 48 kHz sample rate، الـ audio clock يجب يبقى ثابتاً بالضبط — أي تغيير حتى من driver آخر سيخرب الصوت.

---

### الـ Phase و Duty Cycle

الـ CCF لا يدير الـ rate فقط، يدير أيضاً:

**الـ Phase**: زاوية الإزاحة بين clock-ين (بالدرجات 0-359):
```c
clk_set_phase(clk, 90);  /* أزح الـ clock بمقدار 90 درجة */
clk_get_phase(clk);       /* اقرأ الـ phase الحالي */
```

مفيد للـ DDR memory interfaces حيث الـ data strobe يحتاج يكون out-of-phase مع الـ clock.

**الـ Duty Cycle**: نسبة وقت الـ HIGH مقارنة بالـ LOW:
```c
/* ضبط duty cycle على 25% (1/4) */
clk_set_duty_cycle(clk, 1, 4);
clk_get_scaled_duty_cycle(clk, 100); /* يرجع 25 */
```

---

### كيف يتعرف الـ Consumer على الـ Clock الصحيح؟

الربط يتم عبر **Device Tree** (DT):

```
/* في الـ .dts file */
uart0: serial@40001000 {
    compatible = "vendor,uart";
    clocks = <&clk_ctrl UART_CLK>;  /* pointer للـ clock provider */
    clock-names = "uart";           /* الاسم اللي الـ driver يستخدمه */
};
```

```c
/* في الـ driver */
struct clk *clk = devm_clk_get(dev, "uart");
/* الـ CCF يمشي على الـ DT → يلاقي <&clk_ctrl UART_CLK> → يطلب من clk_ctrl → يرجع الـ clock handle */
```

الدوال:
```c
struct clk *of_clk_get(struct device_node *np, int index);
struct clk *of_clk_get_by_name(struct device_node *np, const char *name);
struct clk *of_clk_get_from_provider(struct of_phandle_args *clkspec);
```

---

### ملخص تصميمي

| المبدأ | كيف يطبقه CCF |
|--------|--------------|
| **Abstraction** | `struct clk` يخفي كل تعقيدات الـ hardware |
| **Separation of concerns** | Consumer API / CCF Core / Provider API طبقات منفصلة |
| **Reference counting** | `enable_count` و `prepare_count` يمنعان إيقاف clock مستخدم |
| **Tree management** | Rate propagation و orphan resolution تلقائيان |
| **Power efficiency** | `clk_disable_unused()` يطفي كل clock بدون consumer |
| **Safety** | الـ Notifiers تتيح للـ consumers رفض تغيير الـ rate |
| **Resource management** | `devm_` variants تمنع resource leaks |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Config Options — Cheatsheet

#### الـ Notifier Event Types (من `clk.h`)

| الـ Flag | القيمة | متى يُطلَق؟ |
|---|---|---|
| `PRE_RATE_CHANGE` | `BIT(0)` | قبل تغيير الـ rate — لازم الـ driver يوقف أي عملية متأثرة |
| `POST_RATE_CHANGE` | `BIT(1)` | بعد نجاح تغيير الـ rate |
| `ABORT_RATE_CHANGE` | `BIT(2)` | لو فشل التغيير بعد ما أُرسل `PRE_RATE_CHANGE` |

#### الـ CLK Flags (من `clk-provider.h`)

| الـ Flag | الـ Bit | المعنى بالعربي |
|---|---|---|
| `CLK_SET_RATE_GATE` | 0 | لازم الـ clock يكون مقفول (gated) أثناء تغيير الـ rate |
| `CLK_SET_PARENT_GATE` | 1 | لازم يكون gated أثناء تغيير الـ parent |
| `CLK_SET_RATE_PARENT` | 2 | لو طلبت تغيير rate، ارفع الطلب للـ parent |
| `CLK_IGNORE_UNUSED` | 3 | لا توقفه حتى لو ما فيه أحد يستخدمه |
| `CLK_GET_RATE_NOCACHE` | 6 | اقرأ الـ rate من الـ hardware مباشرة، تجاهل الـ cache |
| `CLK_SET_RATE_NO_REPARENT` | 7 | عند تغيير الـ rate لا تغير الـ parent تلقائياً |
| `CLK_GET_ACCURACY_NOCACHE` | 8 | اقرأ الـ accuracy من الـ hardware مباشرة |
| `CLK_RECALC_NEW_RATES` | 9 | أعد حساب الـ rates بعد إرسال الـ notifications |
| `CLK_SET_RATE_UNGATE` | 10 | الـ clock يحتاج يشتغل (ungate) عشان تقدر تغير الـ rate |
| `CLK_IS_CRITICAL` | 11 | لا توقف هذا الـ clock أبداً — حيوي للنظام |
| `CLK_OPS_PARENT_ENABLE` | 12 | فعّل الـ parent قبل عمليات gate/ungate/set_rate/reparent |
| `CLK_DUTY_CYCLE_PARENT` | 13 | مرّر عمليات duty cycle للـ parent |

#### الـ Kconfig Options

| الـ Option | المعنى |
|---|---|
| `CONFIG_COMMON_CLK` | يفعّل الـ Common Clock Framework — بدونه كل دوال الـ notifier والـ phase تصير no-op |
| `CONFIG_HAVE_CLK_PREPARE` | يفعّل `clk_prepare/unprepare` — بدونه يصيرون stubs تنادي `might_sleep()` فقط |
| `CONFIG_HAVE_CLK` | يفعّل `clk_get/put/enable/disable` وكل الـ API الأساسي |
| `CONFIG_OF` | يفعّل الـ Device Tree integration لاستخراج الـ clocks من الـ DT |

---

### 1. الـ Structs المهمة

---

#### `struct clk`

هذا هو الـ **handle** اللي يحصل عليه الـ driver/consumer — زي مفتاح السيارة. كل consumer عنده نسخته الخاصة من هذا الـ handle، لكنهم كلهم يشيرون لنفس الـ hardware. التعريف الداخلي ليس في `clk.h` (هو opaque للـ consumers) — يعيش في `drivers/clk/clk.c`.

**الغرض:** يفصل الـ consumer API عن الـ internal implementation. كل `struct clk` مرتبط بـ `struct clk_core` واحد يمثل الـ hardware الفعلي.

**الحقول الرئيسية** (من التعريف الداخلي):
- مؤشر لـ `struct clk_core` — الـ hardware الحقيقي
- `min_rate` و `max_rate` — قيود هذا الـ consumer تحديداً
- مؤشر للـ `struct device` المالك

---

#### `struct clk_notifier`

```c
struct clk_notifier {
    struct clk              *clk;           /* الـ clock المراقَب */
    struct srcu_notifier_head notifier_head; /* قائمة الـ callbacks */
    struct list_head         node;          /* ربطه في القائمة العامة */
};
```

**الغرض:** تخيّل لوحة إعلانات لكل clock — أي driver يريد أن "يستمع" لتغييرات الـ rate يضع بطاقته هنا. عند تغيير الـ rate، الكيرنل يمشي القائمة وينادي كل callback.

**العلاقات:**
- واحد لكل `struct clk` محتاج notification
- يحتوي على `srcu_notifier_head` — يستخدم الـ **SRCU (Sleepable RCU)** عشان الـ callbacks ممكن تنام
- مرتبط في linked list عالمية يديرها الـ CCF

---

#### `struct clk_notifier_data`

```c
struct clk_notifier_data {
    struct clk    *clk;      /* الـ clock اللي تغير */
    unsigned long  old_rate; /* الـ rate القديم */
    unsigned long  new_rate; /* الـ rate الجديد */
};
```

**الغرض:** الرسالة اللي تُرسَل مع كل notification — زي SMS يقول "كنت على 100MHz، راح أصير 200MHz". يُمرَّر لكل callback كـ `void *data`.

**ملاحظة خاصة:** في `POST_RATE_CHANGE`، الـ `old_rate` و `new_rate` يكونون نفس القيمة (التحسين الداخلي).

---

#### `struct clk_bulk_data`

```c
struct clk_bulk_data {
    const char *id;  /* اسم الـ clock مثل "apb", "axi" */
    struct clk *clk; /* الـ handle بعد الـ lookup */
};
```

**الغرض:** بدل ما تطلب كل clock على حدة، تبني مصفوفة من هذا الـ struct وتطلب الكل بنداء واحد. زي قائمة تسوق — تعطيها للسوبرماركت (الكيرنل) مرة وحدة.

**مثال استخدام:**
```c
/* مثال: driver يحتاج 3 clocks */
static struct clk_bulk_data my_clks[] = {
    { .id = "core" },
    { .id = "bus"  },
    { .id = "ref"  },
};
/* نداء واحد يجلب الثلاثة */
ret = devm_clk_bulk_get(dev, ARRAY_SIZE(my_clks), my_clks);
```

---

#### `struct clk_rate_request` (من `clk-provider.h`)

```c
struct clk_rate_request {
    struct clk_core *core;            /* الـ clock المطلوب منه */
    unsigned long    rate;            /* الـ rate المطلوب */
    unsigned long    min_rate;        /* الحد الأدنى المسموح */
    unsigned long    max_rate;        /* الحد الأقصى المسموح */
    unsigned long    best_parent_rate;/* أفضل rate يقدر يوفره الـ parent */
    struct clk_hw   *best_parent_hw; /* أفضل parent مناسب */
};
```

**الغرض:** رسالة تفاوض بين الـ consumer والـ hardware. الـ consumer يقول "أريد 200MHz"، الـ framework يملأ `min_rate/max_rate` من القيود، ثم ينادي `determine_rate` على الـ provider ليقول "أقدر أعطيك 196MHz من هذا الـ parent".

---

#### `struct clk_duty` (من `clk-provider.h`)

```c
struct clk_duty {
    unsigned int num; /* البسط مثلاً 1 */
    unsigned int den; /* المقام مثلاً 2 — يعني 50% duty cycle */
};
```

**الغرض:** يصف نسبة وقت الـ HIGH إلى وقت الـ LOW في إشارة الـ clock. `num/den = 1/2` يعني 50%.

---

### 2. مخططات علاقات الـ Structs

```
                        ┌─────────────────────────────────────────────────┐
                        │              Consumer Side (clk.h)              │
                        └─────────────────────────────────────────────────┘

  ┌──────────────┐         ┌──────────────────┐        ┌──────────────────┐
  │struct device │────────▶│   struct clk     │───────▶│ struct clk_core  │
  │  (consumer)  │  owns   │  (opaque handle) │ points │  (internal, one  │
  └──────────────┘         │  - min_rate      │   to   │   per hardware)  │
                           │  - max_rate      │        │  - rate          │
                           │  - *clk_core ───────────▶│  - enable_count  │
                           └──────────────────┘        │  - prepare_count │
                                    │                  │  - flags         │
                                    │ used in          │  - *parent ─────────┐
                           ┌────────▼─────────┐        │  - *clk_hw ─────────┼─▶ struct clk_hw
                           │struct clk_notifier│       │  - children list │  │
                           │  - *clk          │        └──────────────────┘  │
                           │  - notifier_head │                              │
                           │  - node (list) ──┼──▶ global notifier list      │
                           └──────────────────┘                              │
                                    │                                        │
                           ┌────────▼──────────────┐                        │
                           │ struct clk_notifier_   │                        │
                           │        data            │                        │
                           │  - *clk               │                        │
                           │  - old_rate           │                        │
                           │  - new_rate           │                        │
                           └───────────────────────┘                        │
                                                                             │
                        ┌────────────────────────────────────────────────── ┘
                        │           Provider Side (clk-provider.h)
                        ▼
              ┌─────────────────┐         ┌──────────────────────┐
              │  struct clk_hw  │────────▶│   struct clk_ops     │
              │  (provider view)│  .ops   │  - prepare/unprepare │
              │  - init         │         │  - enable/disable    │
              │  - clk ────────────────▶  │  - recalc_rate       │
              └─────────────────┘  back  │  - round_rate        │
                                   ref   │  - determine_rate    │
                                         │  - set_rate          │
                                         │  - set_parent        │
                                         │  - get_parent        │
                                         │  - get_phase         │
                                         │  - set_phase         │
                                         │  - get_duty_cycle    │
                                         │  - set_duty_cycle    │
                                         └──────────────────────┘

              ┌─────────────────────────────────────────────┐
              │          struct clk_rate_request            │
              │  - *core ─────────────────▶ struct clk_core │
              │  - rate (requested)                         │
              │  - min_rate / max_rate (constraints)        │
              │  - best_parent_rate                         │
              │  - *best_parent_hw ────▶ struct clk_hw      │
              └─────────────────────────────────────────────┘

              ┌──────────────────────────────────────────────────┐
              │              struct clk_bulk_data []             │
              │  [0] { .id="core", .clk ──▶ struct clk }        │
              │  [1] { .id="bus",  .clk ──▶ struct clk }        │
              │  [2] { .id="ref",  .clk ──▶ struct clk }        │
              └──────────────────────────────────────────────────┘
```

---

### 3. مخططات الـ Lifecycle

#### دورة حياة الـ Clock — من الطلب للتحرير

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                        CLOCK CONSUMER LIFECYCLE                        │
  └─────────────────────────────────────────────────────────────────────────┘

  [1] الاستحصال (Get)
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_get(dev, "name")                               │
  │    ──▶ يبحث في قائمة الـ providers                  │
  │    ──▶ يخلق struct clk جديد (per-consumer handle)   │
  │    ──▶ يربطه بـ struct clk_core الموجود              │
  │    ◀── يرجع struct clk *                             │
  │                                                      │
  │  أو بالـ Device Tree:                               │
  │  of_clk_get(np, index)                              │
  │    ──▶ يقرأ phandle من الـ DT                       │
  │    ──▶ يجد الـ provider المسجل                      │
  │    ◀── يرجع struct clk *                             │
  └──────────────────────────────────────────────────────┘
                          │
                          ▼
  [2] التحضير (Prepare) — يجوز ينام
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_prepare(clk)                                   │
  │    ──▶ يزيد prepare_count على clk_core              │
  │    ──▶ ينادي clk_ops.prepare()                      │
  │    ──▶ يجهز الـ PLLs، يخصص الـ resources            │
  │    (ممكن ينام — لا تناديه من atomic context)        │
  │                                                      │
  └──────────────────────────────────────────────────────┘
                          │
                          ▼
  [3] التفعيل (Enable) — لا ينام
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_enable(clk)                                    │
  │    ──▶ يزيد enable_count على clk_core               │
  │    ──▶ لو أول enable: ينادي clk_ops.enable()        │
  │    ──▶ يضمن وجود إشارة clock صالحة                  │
  │    (ممكن يُنادى من atomic context)                  │
  │                                                      │
  │  أو بنداء مدمج:                                     │
  │  clk_prepare_enable(clk)  ─▶ prepare ثم enable      │
  └──────────────────────────────────────────────────────┘
                          │
                          ▼
  [4] الاستخدام (Usage)
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_get_rate(clk)     ─▶ اقرأ الـ rate الحالي      │
  │  clk_set_rate(clk, r)  ─▶ غير الـ rate               │
  │  clk_round_rate(clk,r) ─▶ استفسر بدون تغيير         │
  │  clk_set_parent(clk,p) ─▶ غير مصدر الـ clock        │
  │  clk_get_phase(clk)    ─▶ اقرأ الـ phase            │
  │  clk_set_phase(clk,d)  ─▶ اضبط الـ phase            │
  │                                                      │
  └──────────────────────────────────────────────────────┘
                          │
                          ▼
  [5] التعطيل (Disable) — لا ينام
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_disable(clk)                                   │
  │    ──▶ يقلل enable_count                            │
  │    ──▶ لو وصل صفر: ينادي clk_ops.disable()          │
  │    (ممكن يُنادى من atomic context)                  │
  │                                                      │
  └──────────────────────────────────────────────────────┘
                          │
                          ▼
  [6] إلغاء التحضير (Unprepare) — يجوز ينام
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_unprepare(clk)                                 │
  │    ──▶ يقلل prepare_count                           │
  │    ──▶ لو وصل صفر: ينادي clk_ops.unprepare()        │
  │    (لا تناديه من atomic context)                    │
  │                                                      │
  │  أو بنداء مدمج:                                     │
  │  clk_disable_unprepare(clk) ─▶ disable ثم unprepare │
  └──────────────────────────────────────────────────────┘
                          │
                          ▼
  [7] التحرير (Put)
  ┌──────────────────────────────────────────────────────┐
  │                                                      │
  │  clk_put(clk)                                       │
  │    ──▶ يحرر الـ struct clk (الـ handle)              │
  │    ──▶ لا يحذف clk_core (مشترك بين consumers)       │
  │                                                      │
  │  أو تلقائياً مع devm:                              │
  │  devm_clk_get ──▶ يُحرَّر عند unbind الـ device     │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

---

#### دورة حياة الـ Notifier

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                    NOTIFIER LIFECYCLE                            │
  └──────────────────────────────────────────────────────────────────┘

  التسجيل:
  clk_notifier_register(clk, nb)
    ──▶ هل يوجد struct clk_notifier لهذا الـ clock؟
        لا  ──▶ اخلق struct clk_notifier جديد
                 أضفه للقائمة العامة
        نعم ──▶ استخدم الموجود
    ──▶ أضف notifier_block للـ srcu_notifier_head

  عند تغيير الـ rate (set_rate):
    ──▶ ابحث عن struct clk_notifier للـ clock هذا
    ──▶ أنشئ struct clk_notifier_data { old, new }
    ──▶ أرسل PRE_RATE_CHANGE لكل المسجلين
    ──▶ نفذ تغيير الـ rate في الـ hardware
        نجح  ──▶ أرسل POST_RATE_CHANGE
        فشل  ──▶ أرسل ABORT_RATE_CHANGE

  إلغاء التسجيل:
  clk_notifier_unregister(clk, nb)
    ──▶ أزل nb من srcu_notifier_head
    ──▶ لو القائمة فارغة: احذف struct clk_notifier
```

---

### 4. مخططات تدفق الـ Calls

#### تدفق `clk_set_rate`

```
  Driver ──▶ clk_set_rate(clk, rate)
                │
                ▼
         [acquire prepare_lock]  ← mutex، يجوز ينام
                │
                ▼
         clk_core_set_rate_nolock(core, rate)
                │
                ├──▶ clk_calc_new_rates()
                │        │
                │        ├──▶ core->ops->determine_rate()  ← أو round_rate
                │        │      └── يملأ clk_rate_request
                │        │
                │        └──▶ تكرر صعوداً للـ parent لو CLK_SET_RATE_PARENT
                │
                ├──▶ clk_notify(PRE_RATE_CHANGE)
                │        └── srcu_notifier_call_chain()
                │              ──▶ كل callback مسجل
                │
                ├──▶ clk_change_rate()
                │        ├──▶ core->ops->set_rate(core, rate, parent_rate)
                │        └──▶ تحديث core->rate
                │
                ├──▶ clk_notify(POST_RATE_CHANGE)
                │        └── إشعار كل المسجلين بالـ rate الجديد
                │
                └──▶ [release prepare_lock]
```

#### تدفق `clk_get` عبر Device Tree

```
  Driver ──▶ devm_clk_get(dev, "uart")
                │
                ▼
         __clk_get(dev, "uart")
                │
                ▼
         of_clk_get_by_name(dev->of_node, "uart")
                │
                ▼
         of_clk_get_from_provider(clkspec)
                │
                ├──▶ يبحث في قائمة الـ of_clk_providers
                │        (كل provider مسجل بـ of_clk_add_provider)
                │
                ├──▶ provider->get(clkspec, provider->data)
                │        └── ينادي الـ callback الخاص بالـ provider
                │              (مثلاً: of_fixed_clk_get)
                │
                └──▶ يرجع struct clk *
                          │
                          ▼
                   devm_add_action(dev, clk_put, clk)
                   [يسجل cleanup تلقائي عند unbind]
```

#### تدفق `clk_prepare_enable` (المدمج)

```
  clk_prepare_enable(clk)
         │
         ├──[1]─▶ clk_prepare(clk)
         │              │
         │        [acquire prepare_mutex]
         │              │
         │        core->ops->prepare(core)  ← ممكن ينام
         │              │
         │        [release prepare_mutex]
         │
         └──[2]─▶ clk_enable(clk)
                        │
                  [acquire enable_spinlock]  ← IRQ-safe
                        │
                  core->ops->enable(core)   ← لا ينام أبداً
                        │
                  [release enable_spinlock]
```

---

### 5. استراتيجية الـ Locking

الـ Clock Framework يستخدم **طبقتين من الـ locks** — هذا مهم جداً لأن بعض العمليات تحتاج تنام وبعضها في atomic context:

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                       LOCKING HIERARCHY                             │
  └─────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  prepare_lock  (mutex — يجوز ينام)                              │
  │                                                                  │
  │  يحمي:                                                          │
  │  • prepare_count على كل clk_core                               │
  │  • عمليات clk_ops: prepare, unprepare, set_rate, set_parent     │
  │  • قراءة وكتابة الـ rate (clk_core->rate)                      │
  │  • تغيير شجرة الـ parents                                       │
  │  • الـ notifier calls (PRE/POST/ABORT)                          │
  │                                                                  │
  │  من يمسكه:                                                      │
  │  clk_prepare, clk_unprepare, clk_set_rate, clk_set_parent,      │
  │  clk_round_rate, clk_get_rate (في بعض الحالات)                  │
  └──────────────────────────────────────────────────────────────────┘
                              │
                              │ (يجب مسك prepare_lock قبل enable_lock)
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  enable_lock  (spinlock — لا ينام، IRQ-safe)                    │
  │                                                                  │
  │  يحمي:                                                          │
  │  • enable_count على كل clk_core                                │
  │  • عمليات clk_ops: enable, disable                              │
  │                                                                  │
  │  من يمسكه:                                                      │
  │  clk_enable, clk_disable — ممكن يُنادَوا من interrupt handler   │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │  SRCU (Sleepable RCU) — للـ notifiers                           │
  │                                                                  │
  │  يحمي:                                                          │
  │  • srcu_notifier_head داخل struct clk_notifier                  │
  │  • يسمح للـ callbacks بالنوم (blocking notifiers)               │
  │  • يسمح بالقراءة المتزامنة مع الكتابة بأمان                    │
  └──────────────────────────────────────────────────────────────────┘
```

#### ترتيب الـ Locking (لتجنب Deadlock)

```
  القاعدة الذهبية:
  prepare_lock (mutex) ──▶ ثم ──▶ enable_lock (spinlock)
  لا تعكس الترتيب أبداً!

  صح:
  mutex_lock(prepare_lock)
    spin_lock(enable_lock)
      ...
    spin_unlock(enable_lock)
  mutex_unlock(prepare_lock)

  خطأ (deadlock):
  spin_lock(enable_lock)
    mutex_lock(prepare_lock)  ← خطر! mutex ممكن ينام، spinlock لا يسمح
```

#### جدول ملخص من يستطيع استدعاء ماذا

| العملية | الـ Lock المستخدم | من atomic context؟ | يجوز ينام؟ |
|---|---|---|---|
| `clk_prepare` | `prepare_mutex` | لا | نعم |
| `clk_unprepare` | `prepare_mutex` | لا | نعم |
| `clk_enable` | `enable_spinlock` | نعم | لا |
| `clk_disable` | `enable_spinlock` | نعم | لا |
| `clk_set_rate` | `prepare_mutex` | لا | نعم |
| `clk_set_parent` | `prepare_mutex` | لا | نعم |
| `clk_get_rate` | `prepare_mutex` (أحياناً) | لا | نعم |
| notifier callbacks | SRCU | لا | نعم |
| `clk_get` / `clk_put` | لا interrupt context | لا | نعم |

#### تعدد الـ Consumers وعدادات الـ Reference

```
  Consumer A         Consumer B         Consumer C
      │                  │                  │
      │ clk_enable()     │ clk_enable()     │ (لم يفعّل)
      ▼                  ▼                  ▼
  ┌───────────────────────────────────────────────────┐
  │  struct clk_core                                  │
  │  enable_count = 2  ← (A + B فعّلا)               │
  │  prepare_count = 2                                │
  │                                                   │
  │  الـ hardware يشتغل طالما enable_count > 0        │
  └───────────────────────────────────────────────────┘

  A ──▶ clk_disable()  ──▶ enable_count = 1  (لا يزال شغّال)
  B ──▶ clk_disable()  ──▶ enable_count = 0  (يوقف الـ hardware)
```
## Phase 4: شرح الـ Functions

---

### جدول الـ Functions

| Function | Type | Purpose |
|----------|------|---------|
| `clk_notifier_register()` | EXPORT | تسجيل callback يُنادى عند تغيير rate الساعة |
| `clk_notifier_unregister()` | EXPORT | إلغاء تسجيل callback الـ notifier |
| `devm_clk_notifier_register()` | EXPORT | نسخة managed من تسجيل الـ notifier |
| `clk_get_accuracy()` | EXPORT | جلب دقة الساعة بوحدة ppb |
| `clk_set_phase()` | EXPORT | ضبط إزاحة الـ phase لإشارة الساعة |
| `clk_get_phase()` | EXPORT | قراءة إزاحة الـ phase الحالية |
| `clk_set_duty_cycle()` | EXPORT | ضبط نسبة الـ duty cycle للساعة |
| `clk_get_scaled_duty_cycle()` | EXPORT | قراءة نسبة الـ duty cycle مضروبة في scale |
| `clk_is_match()` | EXPORT | مقارنة مؤشرَين للساعة هل يشيران لنفس الـ hardware |
| `clk_rate_exclusive_get()` | EXPORT | حجز التحكم الحصري في rate الساعة |
| `devm_clk_rate_exclusive_get()` | EXPORT | نسخة managed من حجز التحكم الحصري |
| `clk_rate_exclusive_put()` | EXPORT | تحرير التحكم الحصري في rate الساعة |
| `clk_save_context()` | EXPORT | حفظ سياق رجستيرات الساعات قبل قطع الطاقة |
| `clk_restore_context()` | EXPORT | استعادة سياق رجستيرات الساعات بعد التشغيل |
| `clk_prepare()` | EXPORT | تحضير الساعة قبل التشغيل (قد ينام) |
| `clk_unprepare()` | EXPORT | إلغاء تحضير الساعة |
| `clk_bulk_prepare()` | EXPORT | تحضير مجموعة ساعات دفعة واحدة |
| `clk_bulk_unprepare()` | EXPORT | إلغاء تحضير مجموعة ساعات |
| `clk_is_enabled_when_prepared()` | EXPORT | هل الـ prepare يُشغّل الساعة تلقائياً؟ |
| `clk_get()` | EXPORT | الحصول على مرجع لساعة معينة |
| `clk_bulk_get()` | EXPORT | الحصول على مجموعة ساعات دفعة واحدة |
| `clk_bulk_get_all()` | EXPORT | الحصول على كل الساعات المتاحة للجهاز |
| `clk_bulk_get_optional()` | EXPORT | جلب ساعات متعددة مع تحمّل الغياب |
| `devm_clk_bulk_get()` | EXPORT | جلب ساعات متعددة بإدارة تلقائية |
| `devm_clk_bulk_get_optional()` | EXPORT | جلب ساعات اختيارية متعددة بإدارة تلقائية |
| `devm_clk_bulk_get_optional_enable()` | EXPORT | جلب وتشغيل ساعات اختيارية بإدارة تلقائية |
| `devm_clk_bulk_get_all()` | EXPORT | جلب كل الساعات بإدارة تلقائية |
| `devm_clk_bulk_get_all_enabled()` | EXPORT | جلب وتشغيل كل الساعات بإدارة تلقائية |
| `devm_clk_get()` | EXPORT | جلب ساعة بإدارة تلقائية عند فك ربط الجهاز |
| `devm_clk_get_prepared()` | EXPORT | جلب ساعة وتحضيرها بإدارة تلقائية |
| `devm_clk_get_enabled()` | EXPORT | جلب ساعة وتشغيلها بإدارة تلقائية |
| `devm_clk_get_optional()` | EXPORT | جلب ساعة اختيارية بإدارة تلقائية |
| `devm_clk_get_optional_prepared()` | EXPORT | جلب ساعة اختيارية وتحضيرها بإدارة تلقائية |
| `devm_clk_get_optional_enabled()` | EXPORT | جلب ساعة اختيارية وتشغيلها بإدارة تلقائية |
| `devm_clk_get_optional_enabled_with_rate()` | EXPORT | جلب ساعة اختيارية وضبط rate وتشغيلها |
| `devm_get_clk_from_child()` | EXPORT | جلب ساعة من node ابن في شجرة الـ device tree |
| `clk_enable()` | EXPORT | تشغيل الساعة (من atomic context) |
| `clk_bulk_enable()` | EXPORT | تشغيل مجموعة ساعات دفعة واحدة |
| `clk_disable()` | EXPORT | إيقاف الساعة (من atomic context) |
| `clk_bulk_disable()` | EXPORT | إيقاف مجموعة ساعات دفعة واحدة |
| `clk_get_rate()` | EXPORT | قراءة تردد الساعة الحالي بالـ Hz |
| `clk_put()` | EXPORT | تحرير مرجع الساعة |
| `clk_bulk_put()` | EXPORT | تحرير مجموعة مراجع ساعات |
| `clk_bulk_put_all()` | EXPORT | تحرير كل مراجع ساعات المجموعة |
| `devm_clk_put()` | EXPORT | تحرير ساعة managed يدوياً |
| `clk_round_rate()` | EXPORT | تقريب rate مطلوب لأقرب rate ممكنة |
| `clk_set_rate()` | EXPORT | ضبط تردد الساعة |
| `clk_set_rate_exclusive()` | EXPORT | ضبط rate وحجز التحكم الحصري في آن واحد |
| `clk_has_parent()` | EXPORT | التحقق هل ساعة ممكن تكون parent لأخرى |
| `clk_set_rate_range()` | EXPORT | تحديد نطاق مسموح به لتردد الساعة |
| `clk_set_min_rate()` | EXPORT | تحديد أدنى تردد مسموح للساعة |
| `clk_set_max_rate()` | EXPORT | تحديد أعلى تردد مسموح للساعة |
| `clk_set_parent()` | EXPORT | تغيير مصدر الساعة الأب |
| `clk_get_parent()` | EXPORT | الحصول على مرجع ساعة الأب |
| `clk_get_sys()` | EXPORT | جلب ساعة باسم الجهاز بدل الـ struct device |
| `clk_prepare_enable()` | static inline | تحضير وتشغيل الساعة في خطوة واحدة |
| `clk_disable_unprepare()` | static inline | إيقاف وإلغاء تحضير الساعة في خطوة واحدة |
| `clk_bulk_prepare_enable()` | static inline | تحضير وتشغيل مجموعة ساعات دفعة واحدة |
| `clk_bulk_disable_unprepare()` | static inline | إيقاف وإلغاء تحضير مجموعة ساعات |
| `clk_drop_range()` | static inline | إزالة أي نطاق rate محدد مسبقاً |
| `clk_get_optional()` | static inline | جلب ساعة مع تحمّل غيابها (تُرجع NULL بدل خطأ) |
| `of_clk_get()` | EXPORT | جلب ساعة من device tree بالـ index |
| `of_clk_get_by_name()` | EXPORT | جلب ساعة من device tree بالاسم |
| `of_clk_get_from_provider()` | EXPORT | جلب ساعة من provider عبر phandle args |

---

### المجموعة الأولى: تسجيل وإلغاء الـ Notifiers

الـ **notifier** في لينكس هو نظام يسمح لأي كود يهتم بحدث معين (زي تغيير تردد ساعة) إنه يسجّل callback يُنادى لما الحدث يحصل. فكّره زي الاشتراك في نشرة إخبارية — أنت تسجّل اهتمامك، والنظام ينادي عليك لما في أخبار جديدة.

الـ clock framework يستخدم ثلاث لحظات للإشعار:
- **PRE_RATE_CHANGE**: قبل ما تتغير السرعة — "هنغير السرعة، استعد"
- **POST_RATE_CHANGE**: بعد ما تغيّرت السرعة بنجاح — "خلاص اتغيرت"
- **ABORT_RATE_CHANGE**: فشل التغيير — "ارجع لوضعك الأصلي"

---

#### `clk_notifier_register()`

```c
int clk_notifier_register(struct clk *clk, struct notifier_block *nb);
```

بتسجّل callback يُنادى لما rate الساعة `clk` على وشك تتغير أو اتغيرت. النظام يحتفظ بقائمة من الـ notifiers لكل ساعة، وأول مرة تسجّل على ساعة معينة بيُنشئ entry جديدة في هذه القائمة.

**البارامترات:**
- `clk`: الساعة اللي عايز تراقب تغيير rate فيها.
- `nb`: الـ `notifier_block` اللي فيه الـ callback function، ده اللي بيتنادى لما الحدث يوقع.

**القيمة المُرجَعة:** `0` في حالة النجاح، أو قيمة سالبة تمثل رقم الخطأ.

**تفاصيل مهمة:**
- لازم تتوازن كل `clk_notifier_register()` بـ `clk_notifier_unregister()` مقابلها.
- الـ callback لازم يُرجع `NOTIFY_DONE`، `NOTIFY_OK`، `NOTIFY_STOP`، أو `NOTIFY_BAD` حسب الحالة.
- يُستخدم داخلياً الـ `srcu_notifier_head` اللي يسمح بـ sleeping في الـ callbacks.
- **من يناديها:** أي driver يهتم بتغييرات rate ساعة معينة.

---

#### `clk_notifier_unregister()`

```c
int clk_notifier_unregister(struct clk *clk, struct notifier_block *nb);
```

تُلغي تسجيل callback سبق تسجيله بـ `clk_notifier_register()`. لو كان ده آخر notifier على هذه الساعة، النظام يُزيل الـ entry الخاصة بها من القائمة الداخلية.

**البارامترات:**
- `clk`: الساعة المسجَّل عليها.
- `nb`: نفس الـ `notifier_block` اللي اتسجّل.

**القيمة المُرجَعة:** `0` في حالة النجاح، أو قيمة سالبة في حالة الخطأ.

**تفاصيل مهمة:** يجب استدعاؤها قبل تحرير الـ driver أو قبل أي `clk_put()`.

---

#### `devm_clk_notifier_register()`

```c
int devm_clk_notifier_register(struct device *dev, struct clk *clk,
                                struct notifier_block *nb);
```

نسخة **managed** من `clk_notifier_register()`. "Managed" تعني إن النظام تلقائياً يُلغي التسجيل لما الـ `device` يُفصَل (unbound) من الـ bus، من غير ما تحتاج تستدعي `clk_notifier_unregister()` يدوياً.

**البارامترات:**
- `dev`: الجهاز اللي بيملك هذا الـ notifier وبيُربط بعمره به.
- `clk`: الساعة المراد مراقبتها.
- `nb`: الـ `notifier_block` بالـ callback.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

---

### المجموعة الثانية: خصائص الإشارة — Phase و Duty Cycle و Accuracy

هذه المجموعة تتعامل مع **خصائص إشارة الساعة** نفسها، مش بس سرعتها. فكر في الساعة كموجة مربعة — الـ rate هو سرعتها، لكن في أيضاً زاوية الموجة (phase) ونسبة الوقت اللي تكون فيه "عالية" (duty cycle).

---

#### `clk_get_accuracy()`

```c
long clk_get_accuracy(struct clk *clk);
```

بتُرجع دقة مصدر الساعة بوحدة **ppb (parts per billion)**. الدقة تعني: بكم هرتز قد يختلف التردد الفعلي عن التردد المطلوب؟ قيمة `0` تعني ساعة مثالية.

**البارامترات:**
- `clk`: مصدر الساعة المراد معرفة دقته.

**القيمة المُرجَعة:** قيمة الدقة بوحدة ppb (موجبة)، أو `0` لو الساعة مثالية، أو `-ENOTSUPP` لو الـ feature مش مدعومة.

**مثال:** لو عندنا ساعة 100 MHz ودقتها 50 ppb، يعني التردد الفعلي ممكن يكون 100,000,000 ± 5 Hz.

---

#### `clk_set_phase()`

```c
int clk_set_phase(struct clk *clk, int degrees);
```

بتضبط إزاحة الـ **phase** (الطور) لإشارة الساعة بالدرجات. متخيّلش الساعة كرقم بس، هي موجة — والـ phase هو نقطة البداية في هذه الموجة. مفيد لما تحتاج ساعتين متزامنتين بفارق زمني معين.

**البارامترات:**
- `clk`: إشارة الساعة المراد ضبطها.
- `degrees`: الإزاحة بالدرجات (0–359).

**القيمة المُرجَعة:** `0` في حالة النجاح، أو `errno` سالب.

---

#### `clk_get_phase()`

```c
int clk_get_phase(struct clk *clk);
```

بتُرجع إزاحة الـ phase الحالية لإشارة الساعة بالدرجات.

**البارامترات:**
- `clk`: إشارة الساعة المراد قراءة phase فيها.

**القيمة المُرجَعة:** عدد الدرجات (موجب)، أو `errno` سالب في حالة الخطأ.

---

#### `clk_set_duty_cycle()`

```c
int clk_set_duty_cycle(struct clk *clk, unsigned int num, unsigned int den);
```

بتضبط **نسبة الـ duty cycle** لإشارة الساعة. الـ duty cycle هو النسبة اللي تكون فيها الإشارة "عالية" (HIGH) من كل دورة. مثلاً `num=1, den=2` تعني 50% (نصف الوقت HIGH ونصف الوقت LOW).

**البارامترات:**
- `clk`: إشارة الساعة.
- `num`: البسط (numerator) لنسبة الـ duty cycle.
- `den`: المقام (denominator) لنسبة الـ duty cycle.

**القيمة المُرجَعة:** `0` في حالة النجاح، أو `errno` سالب.

**تفاصيل:** `num` يجب أن يكون أصغر من أو يساوي `den`. النسبة `num/den` تمثل وقت الـ HIGH مقسوماً على دورة كاملة.

---

#### `clk_get_scaled_duty_cycle()`

```c
int clk_get_scaled_duty_cycle(struct clk *clk, unsigned int scale);
```

بتُرجع نسبة الـ duty cycle الحالية مضروبة في `scale` عشان تكون عدد صحيح (لأن الكمبيوتر ما بيحب الكسور). مثلاً لو الـ duty cycle هو 50% و`scale=100`، تُرجع 50.

**البارامترات:**
- `clk`: إشارة الساعة.
- `scale`: العامل الضاربي لتحويل النسبة لعدد صحيح.

**القيمة المُرجَعة:** قيمة الـ duty cycle × scale، أو `errno` سالب في حالة الخطأ.

---

#### `clk_is_match()`

```c
bool clk_is_match(const struct clk *p, const struct clk *q);
```

بتتحقق هل مؤشران لساعتين `p` و `q` يشيران لـ **نفس الـ hardware clock**. في لينكس، ممكن يكون عندك أكثر من `struct clk *` يمثل نفس الساعة الفيزيائية (لأن كل consumer بياخد مرجعه الخاص)، وهذه الدالة تتحقق من الهوية الداخلية الحقيقية (`struct clk_core`).

**البارامترات:**
- `p`: المؤشر الأول للساعة.
- `q`: المؤشر الثاني للساعة.

**القيمة المُرجَعة:** `true` لو يشيران لنفس الـ hardware، `false` غير ذلك. ملاحظة: مؤشران `NULL` يُعتبران متطابقين (`true`).

---

### المجموعة الثالثة: التحكم الحصري في الـ Rate

أحياناً driver معين لازم **يضمن** إن تردد ساعة ما يتغيرش من أي طرف تاني أثناء عمله. ده زي قفل الأبواب أثناء الطبخ — ما تريد أحد يدخل ويغيير درجة الحرارة.

---

#### `clk_rate_exclusive_get()`

```c
int clk_rate_exclusive_get(struct clk *clk);
```

بتحجز **التحكم الحصري** في rate الساعة. بعد استدعاء هذه الدالة، لا يستطيع أي consumer آخر تغيير rate هذه الساعة (حتى بشكل غير مباشر عبر تغيير parent).

**البارامترات:**
- `clk`: الساعة المراد حجز التحكم فيها.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل مهمة:**
- **لا يُستدعى من atomic context** — قد ينام.
- لو نُودي عليها أكثر من مرة (حتى من نفس الـ driver)، الـ rate يُقفل بشكل أعمق وما يمكن فكه إلا بعد نفس عدد الاستدعاءات لـ `clk_rate_exclusive_put()`.
- لازم تتوازن كل استدعاء بـ `clk_rate_exclusive_put()` مقابله.
- **من يناديها:** drivers الـ real-time أو اللي تحتاج rate ثابتة مضمونة أثناء عمليات حساسة.

---

#### `devm_clk_rate_exclusive_get()`

```c
int devm_clk_rate_exclusive_get(struct device *dev, struct clk *clk);
```

نسخة **managed** من `clk_rate_exclusive_get()`. بتستدعي `clk_rate_exclusive_get()` وبتسجّل cleanup handler يستدعي `clk_rate_exclusive_put()` تلقائياً لما الـ device يُفصَل.

**البارامترات:**
- `dev`: الجهاز اللي بيملك هذا الحجز.
- `clk`: الساعة المراد حجز التحكم فيها.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

---

#### `clk_rate_exclusive_put()`

```c
void clk_rate_exclusive_put(struct clk *clk);
```

بتُحرّر التحكم الحصري اللي تم حجزه بـ `clk_rate_exclusive_get()`. بعد ما الـ exclusive counter يوصل لصفر، يُسمح مجدداً لـ consumers آخرين بتغيير rate الساعة.

**البارامترات:**
- `clk`: الساعة المراد تحرير التحكم فيها.

**تفاصيل مهمة:** **لا يُستدعى من atomic context**. لازم يتوازن مع عدد مقابل من `clk_rate_exclusive_get()`.

---

### المجموعة الرابعة: حفظ واستعادة السياق (Power Management)

---

#### `clk_save_context()`

```c
int clk_save_context(void);
```

بتحفظ محتويات رجستيرات الـ clock hardware قبل ما الجهاز يدخل في حالة power-off عميقة (زي الـ suspend). في هذه الحالة، الرجستيرات قد تُفقد قيمها.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل مهمة:**
- تُستدعى من عمق كود الـ suspend، لذا **لا يلزم locking** (الـ system في حالة متسلسلة).
- **من يناديها:** كود الـ suspend في الـ platform أو الـ SoC.

---

#### `clk_restore_context()`

```c
void clk_restore_context(void);
```

بتستعيد محتويات رجستيرات الـ clock hardware بعد العودة من حالة power-off. تُستدعى والساعات كلها مشغّلة.

**تفاصيل مهمة:** تُستدعى من عمق كود الـ resume، **لا يلزم locking**، الـ system في حالة متسلسلة.

---

### المجموعة الخامسة: التحضير والإلغاء (Prepare/Unprepare)

الـ **prepare** في لينكس هو خطوة مسبقة للتشغيل تسمح للـ clock framework بعمل عمليات قد تستغرق وقتاً أو تحتاج للنوم (sleeping) — زي تفعيل PLL أو انتظار استقرار التردد. بعدها يأتي `clk_enable()` وهو عملية سريعة يمكن من atomic context.

فكّرها كـ "تسخين الفرن" (prepare) ثم "وضع الطبق" (enable).

---

#### `clk_prepare()`

```c
int clk_prepare(struct clk *clk);
```

بتُحضّر الساعة للتشغيل. هذه الخطوة قد تتضمن عمليات بطيئة زي تشغيل PLL وانتظار استقراره. يجب استدعاؤها قبل `clk_enable()`.

**البارامترات:**
- `clk`: الساعة المراد تحضيرها (NULL يُقبل ويُعامل كـ no-op).

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل مهمة:**
- **لا يُستدعى من atomic context** — قد ينام.
- لازم تتوازن كل `clk_prepare()` بـ `clk_unprepare()`.
- متاحة فقط لو `CONFIG_HAVE_CLK_PREPARE` مُفعّل، غير ذلك no-op.

---

#### `clk_unprepare()`

```c
void clk_unprepare(struct clk *clk);
```

تُلغي تحضير الساعة، عكس `clk_prepare()`. يجب استدعاؤها بعد `clk_disable()`.

**البارامترات:**
- `clk`: الساعة المراد إلغاء تحضيرها.

**تفاصيل مهمة:** **لا يُستدعى من atomic context** — قد ينام.

---

#### `clk_bulk_prepare()`

```c
int __must_check clk_bulk_prepare(int num_clks,
                                   const struct clk_bulk_data *clks);
```

بتُحضّر مجموعة ساعات دفعة واحدة. لو فشل تحضير أي ساعة، بتُلغي تحضير كل الساعات اللي نجحت قبلها (rollback) وتُرجع الخطأ.

**البارامترات:**
- `num_clks`: عدد الساعات في المصفوفة.
- `clks`: مصفوفة من `clk_bulk_data` (كل عنصر يحمل اسم الساعة ومؤشرها).

**القيمة المُرجَعة:** `0` إذا نجح الكل، `errno` سالب إذا فشل أحدها.

---

#### `clk_bulk_unprepare()`

```c
void clk_bulk_unprepare(int num_clks, const struct clk_bulk_data *clks);
```

تُلغي تحضير مجموعة ساعات دفعة واحدة.

**البارامترات:**
- `num_clks`: عدد الساعات.
- `clks`: مصفوفة الساعات.

---

#### `clk_is_enabled_when_prepared()`

```c
bool clk_is_enabled_when_prepared(struct clk *clk);
```

بتُجيب على السؤال: هل `clk_prepare()` تُشغّل الساعة تلقائياً؟ في بعض الـ platforms، الـ prepare والـ enable هما نفس الشيء عملياً.

**البارامترات:**
- `clk`: الساعة المراد السؤال عنها.

**القيمة المُرجَعة:** `true` لو الـ prepare يُشغّل الساعة أيضاً، `false` غير ذلك.

**تفاصيل:** مهمة لكود إدارة الطاقة — لو `true`، فـ `clk_disable()` وحدها ما تكفي لإيقاف الساعة فعلاً، لازم `clk_unprepare()` أيضاً.

---

### المجموعة السادسة: الحصول على الساعات وتحريرها (Get/Put)

هذه المجموعة مسؤولة عن **دورة حياة** المرجع (reference) للساعة: تبدأ بـ `clk_get()` تنتهي بـ `clk_put()`. بينهما الـ driver يملك مؤشراً صالحاً للعمل مع الساعة.

---

#### `clk_get()`

```c
struct clk *clk_get(struct device *dev, const char *id);
```

بتبحث عن ساعة معينة وتُرجع مرجعاً لها. الـ framework يستخدم `dev` و `id` معاً لتحديد الساعة المطلوبة — ممكن نفس `id` يُشير لساعات مختلفة لأجهزة مختلفة.

**البارامترات:**
- `dev`: الجهاز الطالب للساعة (الـ consumer).
- `id`: اسم الساعة المطلوبة (كـ string مثل `"uart_clk"`).

**القيمة المُرجَعة:** مؤشر `struct clk *` صالح في حالة النجاح، أو `IS_ERR()` بـ `errno` في حالة الخطأ.

**تفاصيل مهمة:**
- **لا تُستدعى من interrupt context**.
- الـ driver يفترض إن الساعة غير مُشغّلة بعد الحصول عليها.
- متاحة فقط مع `CONFIG_HAVE_CLK`.

---

#### `clk_bulk_get()`

```c
int __must_check clk_bulk_get(struct device *dev, int num_clks,
                               struct clk_bulk_data *clks);
```

بتحصل على مجموعة ساعات دفعة واحدة. لو فشل الحصول على أي ساعة، بتُحرّر كل الساعات اللي نجحت قبلها (rollback تلقائي).

**البارامترات:**
- `dev`: الجهاز الطالب.
- `num_clks`: عدد الساعات المطلوبة.
- `clks`: مصفوفة `clk_bulk_data` مُهيّأة بالأسماء مسبقاً، والـ framework يملأ حقل `clk` لكل عنصر.

**القيمة المُرجَعة:** `0` في حالة النجاح الكلي، `errno` سالب إذا فشل أي منها.

**سياق الاستدعاء:** لا يُستدعى من interrupt context.

---

#### `clk_bulk_get_all()`

```c
int __must_check clk_bulk_get_all(struct device *dev,
                                   struct clk_bulk_data **clks);
```

بتحصل على **كل** الساعات المرتبطة بالجهاز في مرة واحدة، من غير ما تعرف عددها مسبقاً. النظام يُخصص المصفوفة تلقائياً.

**البارامترات:**
- `dev`: الجهاز الطالب.
- `clks`: مؤشر لمؤشر — النظام يُخصص مصفوفة `clk_bulk_data` ويُعيد عنوانها هنا.

**القيمة المُرجَعة:** عدد موجب = عدد الساعات المُحصّلة، `0` = لا يوجد ساعات، قيمة سالبة = خطأ.

---

#### `clk_bulk_get_optional()`

```c
int __must_check clk_bulk_get_optional(struct device *dev, int num_clks,
                                        struct clk_bulk_data *clks);
```

مثل `clk_bulk_get()` لكن **تتحمّل غياب أي ساعة**. لو ساعة معينة غير موجودة (لا يوجد provider)، بدل ما تُرجع `-ENOENT`، تضع `NULL` في حقل `clk` وتكمل.

**القيمة المُرجَعة:** `0` في حالة النجاح (حتى لو بعض الساعات غائبة)، `errno` سالب في حالة خطأ حقيقي.

---

#### `clk_put()`

```c
void clk_put(struct clk *clk);
```

بتُحرّر مرجع ساعة حُصل عليه بـ `clk_get()` أو `clk_get_sys()`.

**البارامترات:**
- `clk`: الساعة المراد تحريرها.

**تفاصيل مهمة:**
- **لا تُستدعى من interrupt context**.
- يجب أن تكون كل `clk_enable()` قد قابلها `clk_disable()` قبل استدعاء هذه الدالة.

---

#### `clk_bulk_put()`

```c
void clk_bulk_put(int num_clks, struct clk_bulk_data *clks);
```

بتُحرّر مجموعة مراجع ساعات تم الحصول عليها بـ `clk_bulk_get()`.

**البارامترات:**
- `num_clks`: عدد الساعات في المصفوفة.
- `clks`: مصفوفة الساعات.

---

#### `clk_bulk_put_all()`

```c
void clk_bulk_put_all(int num_clks, struct clk_bulk_data *clks);
```

مثل `clk_bulk_put()` لكن مخصصة لمجموعات حُصل عليها بـ `clk_bulk_get_all()`.

---

#### `devm_clk_put()`

```c
void devm_clk_put(struct device *dev, struct clk *clk);
```

تُحرّر يدوياً ساعة managed (حُصل عليها بـ `devm_clk_get()`) قبل موعد تحريرها التلقائي عند فصل الجهاز.

**البارامترات:**
- `dev`: الجهاز المالك.
- `clk`: الساعة المراد تحريرها.

---

### المجموعة السابعة: الـ Managed Get (devm_clk_get_*)

الـ **devm** (device-managed) functions تُضيف طبقة من الأمان — تتكفّل بتحرير الموارد تلقائياً لما الـ device يُفصَل. هذا يُقلّل من bugs الـ resource leak.

تخيّلها كأنك استأجرت أدوات من محل واتفقت معهم إنهم يُرسلوا يستردّونها لما ينتهي عقدك، من غير ما تحتاج تتذكر تُرجعها بنفسك.

---

#### `devm_clk_get()`

```c
struct clk *devm_clk_get(struct device *dev, const char *id);
```

مثل `clk_get()` لكن الـ clock يُحرَّر تلقائياً لما الـ device يُفصَل.

**البارامترات:**
- `dev`: الجهاز الطالب.
- `id`: اسم الساعة.

**القيمة المُرجَعة:** `struct clk *` صالح أو `IS_ERR()`.

**سياق الاستدعاء:** قد ينام (`May sleep`).

---

#### `devm_clk_get_prepared()`

```c
struct clk *devm_clk_get_prepared(struct device *dev, const char *id);
```

`devm_clk_get()` + `clk_prepare()` في خطوة واحدة. الساعة المُرجَعة (لو صالحة) تكون **محضّرة** لكن **غير مُشغّلة**. لما الجهاز يُفصَل، النظام يستدعي `clk_unprepare()` و `clk_put()` تلقائياً.

**القيمة المُرجَعة:** `struct clk *` محضّرة أو `IS_ERR()`.

---

#### `devm_clk_get_enabled()`

```c
struct clk *devm_clk_get_enabled(struct device *dev, const char *id);
```

`devm_clk_get()` + `clk_prepare_enable()` في خطوة واحدة. الساعة المُرجَعة تكون **محضّرة ومُشغّلة**. لما الجهاز يُفصَل، النظام يستدعي `clk_disable_unprepare()` و `clk_put()` تلقائياً.

**القيمة المُرجَعة:** `struct clk *` مُشغّلة أو `IS_ERR()`.

---

#### `devm_clk_get_optional()`

```c
struct clk *devm_clk_get_optional(struct device *dev, const char *id);
```

مثل `devm_clk_get()` لكن لو ما وُجدت الساعة (خطأ `-ENOENT`)، تُرجع `NULL` بدل الخطأ. الـ `NULL` يعمل كـ dummy clock — كل العمليات عليه تنجح بدون أثر.

**القيمة المُرجَعة:** `struct clk *` صالحة، أو `NULL` لو غير موجودة، أو `IS_ERR()` لخطأ حقيقي.

---

#### `devm_clk_get_optional_prepared()`

```c
struct clk *devm_clk_get_optional_prepared(struct device *dev, const char *id);
```

`devm_clk_get_optional()` + `clk_prepare()`. لو الساعة موجودة، تكون محضّرة. لو غير موجودة، تُرجع `NULL`.

---

#### `devm_clk_get_optional_enabled()`

```c
struct clk *devm_clk_get_optional_enabled(struct device *dev, const char *id);
```

`devm_clk_get_optional()` + `clk_prepare_enable()`. لو الساعة موجودة، تكون محضّرة ومُشغّلة. لو غير موجودة، تُرجع `NULL`.

---

#### `devm_clk_get_optional_enabled_with_rate()`

```c
struct clk *devm_clk_get_optional_enabled_with_rate(struct device *dev,
                                                     const char *id,
                                                     unsigned long rate);
```

أشمل دالة في هذه المجموعة — تجمع: `devm_clk_get_optional()` + `clk_set_rate()` + `clk_prepare_enable()` في خطوة واحدة. مثالية لـ drivers اللي تحتاج ساعة بتردد محدد وتريدها جاهزة فوراً.

**البارامترات:**
- `dev`: الجهاز الطالب.
- `id`: اسم الساعة.
- `rate`: التردد المطلوب بالـ Hz.

**القيمة المُرجَعة:** `struct clk *` مضبوطة ومُشغّلة، أو `NULL` لو غير موجودة، أو `IS_ERR()`.

**التدفق الداخلي المبسّط:**
```c
/* pseudocode داخلي */
clk = devm_clk_get_optional(dev, id);
if (IS_ERR_OR_NULL(clk)) return clk;

ret = clk_set_rate(clk, rate);
if (ret) { devm_clk_put(dev, clk); return ERR_PTR(ret); }

ret = clk_prepare_enable(clk);
if (ret) { devm_clk_put(dev, clk); return ERR_PTR(ret); }

/* تسجيل cleanup تلقائي للـ disable + unprepare + put */
return clk;
```

---

#### `devm_get_clk_from_child()`

```c
struct clk *devm_get_clk_from_child(struct device *dev,
                                     struct device_node *np,
                                     const char *con_id);
```

بتحصل على ساعة من **node ابن** في شجرة الـ device tree، بدل من الجهاز الرئيسي نفسه. مفيد لـ devices اللي ساعاتها مُعرَّفة في sub-nodes في الـ DT.

**البارامترات:**
- `dev`: الجهاز المالك (للـ devm management).
- `np`: الـ `device_node` الابن اللي فيه تعريف الساعة.
- `con_id`: اسم الـ connection ID للساعة في ذاك الـ node.

**القيمة المُرجَعة:** `struct clk *` أو `IS_ERR()`.

---

### المجموعة الثامنة: تشغيل وإيقاف الساعات (Enable/Disable)

بعد الـ prepare، الخطوة الثانية هي **enable** — وهي عملية سريعة يُمكن استدعاؤها من atomic context (زي interrupt handler). الـ enable لا ينام ولا يُسبّب تأخيراً طويلاً.

الـ clock framework يحتفظ بـ **enable count** — لو عندك أكثر من consumer للساعة، الساعة تظل مُشغّلة لحين يُوقفها آخر consumer.

---

#### `clk_enable()`

```c
int clk_enable(struct clk *clk);
```

بتُبلّغ النظام إن هذا الـ consumer يحتاج الساعة مُشغّلة الآن. تزيد الـ enable count بمقدار 1.

**البارامترات:**
- `clk`: الساعة المراد تشغيلها.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل مهمة:**
- **يُمكن استدعاؤها من atomic context**.
- لازم تُستدعى بعد `clk_prepare()` (إلا في platforms بدون منفصلة).
- لازم تتوازن كل `clk_enable()` بـ `clk_disable()`.

---

#### `clk_bulk_enable()`

```c
int __must_check clk_bulk_enable(int num_clks,
                                  const struct clk_bulk_data *clks);
```

بتُشغّل مجموعة ساعات دفعة واحدة. لو فشل تشغيل أي ساعة، بتوقف وتُرجع الخطأ.

**البارامترات:**
- `num_clks`: عدد الساعات.
- `clks`: مصفوفة الساعات (المحضّرة مسبقاً).

**القيمة المُرجَعة:** `0` في حالة نجاح الكل، `errno` سالب إذا فشل أي منها.

---

#### `clk_disable()`

```c
void clk_disable(struct clk *clk);
```

بتُبلّغ النظام إن هذا الـ consumer لم يعد يحتاج الساعة. تُخفّض الـ enable count بمقدار 1. لو وصل الـ count لصفر وما في consumers أخرى، الساعة قد تُوقَف.

**البارامترات:**
- `clk`: الساعة المراد إيقافها.

**تفاصيل مهمة:**
- **يُمكن استدعاؤها من atomic context**.
- لو الساعة مشتركة بين drivers متعددة، الساعة لا تُوقَف إلا لما كل الـ consumers يستدعوا `clk_disable()`.

---

#### `clk_bulk_disable()`

```c
void clk_bulk_disable(int num_clks, const struct clk_bulk_data *clks);
```

بتوقف مجموعة ساعات دفعة واحدة.

**البارامترات:**
- `num_clks`: عدد الساعات.
- `clks`: مصفوفة الساعات.

---

#### `clk_get_rate()`

```c
unsigned long clk_get_rate(struct clk *clk);
```

بتُرجع التردد الحالي للساعة بالـ **Hz**. هذه القيمة صالحة بعد ما تكون الساعة مُشغّلة.

**البارامترات:**
- `clk`: الساعة المراد معرفة ترددها.

**القيمة المُرجَعة:** التردد بالـ Hz (unsigned long)، أو `0` لو الساعة غير صالحة أو مش مدعوم.

---

### المجموعة التاسعة: التحكم في Rate و Parent

---

#### `clk_round_rate()`

```c
long clk_round_rate(struct clk *clk, unsigned long rate);
```

بتُجيب على السؤال: "لو طلبت من الساعة تردد `rate`، إيه أقرب تردد ممكن تُعطيه فعلاً؟" من غير ما تُغيّر أي شيء في الـ hardware.

مفيد جداً لـ drivers اللي عايزة تتحقق من Rate قبل ضبطه.

**البارامترات:**
- `clk`: الساعة.
- `rate`: التردد المطلوب بالـ Hz.

**القيمة المُرجَعة:** أقرب تردد ممكن (موجب)، أو `errno` سالب في حالة الخطأ.

---

#### `clk_set_rate()`

```c
int clk_set_rate(struct clk *clk, unsigned long rate);
```

بتضبط تردد الساعة للقيمة المطلوبة (أو أقرب قيمة ممكنة). التغيير يبدأ من أعلى الشجرة (الـ parent المتأثر) وينتشر لأسفل.

**البارامترات:**
- `clk`: الساعة المراد تغيير ترددها.
- `rate`: التردد المطلوب بالـ Hz.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل مهمة:** يُطلق إشعارات `PRE_RATE_CHANGE` قبل التغيير و `POST_RATE_CHANGE` أو `ABORT_RATE_CHANGE` بعده.

**التدفق الداخلي المبسّط:**
```c
/* pseudocode لتغيير rate */
// 1. احسب الـ rate الجديد لكل الـ clocks المتأثرة
// 2. أرسل PRE_RATE_CHANGE لكل الـ notifiers
// 3. إذا رجع أحد NOTIFY_BAD → أرسل ABORT_RATE_CHANGE وارجع بخطأ
// 4. اكتب الـ rate الجديد للـ hardware
// 5. أرسل POST_RATE_CHANGE
```

---

#### `clk_set_rate_exclusive()`

```c
int clk_set_rate_exclusive(struct clk *clk, unsigned long rate);
```

تجمع `clk_set_rate()` + `clk_rate_exclusive_get()` في استدعاء واحد **atomic** (بمعنى إنهما يتمان كعملية واحدة لا تتجزأ من الـ locking perspective). يمنع race condition بين ضبط الـ rate وحجز التحكم فيه.

**البارامترات:**
- `clk`: الساعة.
- `rate`: التردد المطلوب.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل مهمة:** لازم يتوازن بـ `clk_rate_exclusive_put()`.

---

#### `clk_has_parent()`

```c
bool clk_has_parent(const struct clk *clk, const struct clk *parent);
```

بتتحقق هل ساعة `parent` يمكن أن تكون **parent محتملة** لساعة `clk` (أي هل هي ضمن قائمة الـ parents المتاحة لـ `clk`)، من غير ما تُغيّر الـ parent الفعلي.

**البارامترات:**
- `clk`: الساعة المراد التحقق من parents لها.
- `parent`: الساعة المُرشّحة لتكون parent.

**القيمة المُرجَعة:** `true` لو هي parent محتملة، `false` غير ذلك.

---

#### `clk_set_rate_range()`

```c
int clk_set_rate_range(struct clk *clk, unsigned long min, unsigned long max);
```

بتُحدد **نطاقاً** مسموحاً به للتردد — الـ framework سيرفض أي `clk_set_rate()` خارج هذا النطاق، ويُحاول دائماً ابقاء الـ rate بين `min` و `max`.

**البارامترات:**
- `clk`: الساعة.
- `min`: أدنى تردد مسموح به (inclusive) بالـ Hz.
- `max`: أعلى تردد مسموح به (inclusive) بالـ Hz.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

---

#### `clk_set_min_rate()`

```c
int clk_set_min_rate(struct clk *clk, unsigned long rate);
```

تُحدد أدنى تردد مسموح به فقط (بدون تغيير الأعلى). اختصار لـ `clk_set_rate_range()` مع الإبقاء على الـ max الحالي.

**البارامترات:**
- `clk`: الساعة.
- `rate`: أدنى تردد مسموح (inclusive) بالـ Hz.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب.

---

#### `clk_set_max_rate()`

```c
int clk_set_max_rate(struct clk *clk, unsigned long rate);
```

تُحدد أعلى تردد مسموح به فقط. اختصار لـ `clk_set_rate_range()` مع الإبقاء على الـ min الحالي.

**البارامترات:**
- `clk`: الساعة.
- `rate`: أعلى تردد مسموح (inclusive) بالـ Hz.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب.

---

#### `clk_set_parent()`

```c
int clk_set_parent(struct clk *clk, struct clk *parent);
```

بتُغيّر مصدر الساعة — أي تُحدد من أي ساعة أخرى تستقي `clk` ترددها. هذا يُؤثر على التردد النهائي لـ `clk` وكل الـ clocks اللي تحتها في الشجرة.

**البارامترات:**
- `clk`: الساعة المراد تغيير parent لها.
- `parent`: الساعة الجديدة المراد استخدامها كـ parent.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

**تفاصيل:** يُطلق إشعارات الـ notifier (`PRE/POST_RATE_CHANGE`) لأن تغيير الـ parent غالباً يُغيّر التردد.

---

#### `clk_get_parent()`

```c
struct clk *clk_get_parent(struct clk *clk);
```

بتُرجع مرجع ساعة الـ parent الحالية.

**البارامترات:**
- `clk`: الساعة المراد معرفة parent لها.

**القيمة المُرجَعة:** `struct clk *` لساعة الـ parent، أو `IS_ERR()` في حالة الخطأ.

**تفاصيل:** المرجع المُرجَع **لا** يحتاج `clk_put()` — هو مرجع داخلي مؤقت.

---

#### `clk_get_sys()`

```c
struct clk *clk_get_sys(const char *dev_id, const char *con_id);
```

مثل `clk_get()` لكن بتأخذ **اسم الجهاز كـ string** بدل مؤشر `struct device *`. مفيد لحالات الـ early init أو لما ما تكون عندك مؤشر للـ device.

**البارامترات:**
- `dev_id`: اسم الجهاز كـ string.
- `con_id`: اسم الـ connection ID للساعة.

**القيمة المُرجَعة:** `struct clk *` أو `IS_ERR()`.

**سياق الاستدعاء:** لا يُستدعى من interrupt context.

---

### المجموعة العاشرة: الـ Helper Inlines — الدمج والتبسيط

هذه الـ functions هي `static inline` — مُبنيّة فوق الـ primitives السابقة لتوفير patterns شائعة الاستخدام.

---

#### `clk_prepare_enable()`

```c
static inline int clk_prepare_enable(struct clk *clk)
{
    int ret;
    ret = clk_prepare(clk);
    if (ret)
        return ret;
    ret = clk_enable(clk);
    if (ret)
        clk_unprepare(clk);
    return ret;
}
```

تجمع `clk_prepare()` + `clk_enable()` في خطوة واحدة. أكثر دالة استخداماً في drivers الـ Linux — لما تريد تُشغّل ساعة في non-atomic context ببساطة.

**التدفق:**
1. تستدعي `clk_prepare()` — لو فشل، تُرجع الخطأ.
2. تستدعي `clk_enable()` — لو فشل، تستدعي `clk_unprepare()` للـ cleanup ثم تُرجع الخطأ.
3. لو كلاهما نجح، تُرجع `0`.

**ملاحظة:** تُستدعى فقط من **non-atomic context** لأن `clk_prepare()` قد تنام.

---

#### `clk_disable_unprepare()`

```c
static inline void clk_disable_unprepare(struct clk *clk)
{
    clk_disable(clk);
    clk_unprepare(clk);
}
```

عكس `clk_prepare_enable()` — تُوقف وتُلغي التحضير في خطوة واحدة. دائماً يجب أن تُستدعى من **non-atomic context**.

**ترتيب الاستدعاء مهم:** `clk_disable()` أولاً ثم `clk_unprepare()` — عكس الترتيب خطأ.

---

#### `clk_bulk_prepare_enable()`

```c
static inline int __must_check
clk_bulk_prepare_enable(int num_clks, const struct clk_bulk_data *clks)
```

تُحضّر وتُشغّل مجموعة ساعات دفعة واحدة. rollback تلقائي لو فشل الـ enable بعد الـ prepare.

**البارامترات:**
- `num_clks`: عدد الساعات.
- `clks`: مصفوفة الساعات (المُحصَّل عليها مسبقاً).

**القيمة المُرجَعة:** `0` في حالة النجاح الكلي، `errno` سالب في حالة أي فشل.

---

#### `clk_bulk_disable_unprepare()`

```c
static inline void clk_bulk_disable_unprepare(int num_clks,
                                               const struct clk_bulk_data *clks)
```

تُوقف وتُلغي تحضير مجموعة ساعات دفعة واحدة. عكس `clk_bulk_prepare_enable()`.

---

#### `clk_drop_range()`

```c
static inline int clk_drop_range(struct clk *clk)
{
    return clk_set_rate_range(clk, 0, ULONG_MAX);
}
```

بتُزيل أي نطاق rate محدد مسبقاً بـ `clk_set_rate_range()`. تعمل بضبط الـ min=0 والـ max=ULONG_MAX، يعني "أي rate مسموح".

**البارامترات:**
- `clk`: الساعة المراد إزالة القيود عنها.

**القيمة المُرجَعة:** `0` في حالة النجاح، `errno` سالب في حالة الخطأ.

---

#### `clk_get_optional()`

```c
static inline struct clk *clk_get_optional(struct device *dev, const char *id)
{
    struct clk *clk = clk_get(dev, id);
    if (clk == ERR_PTR(-ENOENT))
        return NULL;
    return clk;
}
```

مثل `clk_get()` لكن تتعامل مع حالة "الساعة غير موجودة" بإرجاع `NULL` بدل `-ENOENT`. مفيد لـ drivers اللي بعض الساعات اختيارية فيها — لو الساعة مش في الـ device tree ما يكون خطأ.

**المنطق الداخلي:** لو الساعة غير موجودة (`-ENOENT`)، تُرجع `NULL`. أي خطأ آخر يُعاد كـ `IS_ERR()`.

---

### المجموعة الحادية عشرة: الـ Device Tree Integration (of_clk_*)

هذه الدوال مخصصة للـ platforms اللي تستخدم **Open Firmware / Device Tree** لوصف الـ hardware. تحتاج `CONFIG_OF` و `CONFIG_COMMON_CLK`.

---

#### `of_clk_get()`

```c
struct clk *of_clk_get(struct device_node *np, int index);
```

بتجلب ساعة من قائمة الساعات المُعرَّفة في `clocks` property للـ device tree node، باستخدام الـ **index** (الترتيب في القائمة).

**مثال في DT:**
```
clocks = <&clk_ref>, <&clk_pll>;
/* index 0 → clk_ref, index 1 → clk_pll */
```

**البارامترات:**
- `np`: الـ `device_node` اللي فيه `clocks` property.
- `index`: رقم الساعة في القائمة (يبدأ من 0).

**القيمة المُرجَعة:** `struct clk *` أو `IS_ERR(-ENOENT)` لو الـ DT/CLK غير مُفعَّل.

---

#### `of_clk_get_by_name()`

```c
struct clk *of_clk_get_by_name(struct device_node *np, const char *name);
```

مثل `of_clk_get()` لكن بالاسم بدل الـ index. تبحث في `clock-names` property عن الاسم وترجع الساعة المقابلة.

**مثال في DT:**
```
clocks = <&clk_ref>, <&clk_pll>;
clock-names = "ref", "pll";
/* name "pll" → index 1 → clk_pll */
```

**البارامترات:**
- `np`: الـ `device_node`.
- `name`: اسم الساعة كما في `clock-names` property.

**القيمة المُرجَعة:** `struct clk *` أو `IS_ERR()`.

---

#### `of_clk_get_from_provider()`

```c
struct clk *of_clk_get_from_provider(struct of_phandle_args *clkspec);
```

دالة منخفضة المستوى — بتجلب ساعة مباشرة من قائمة الـ clock providers المُسجَّلين، باستخدام `phandle args` المُفسَّرة من الـ DT. عادةً لا تُستدعى مباشرة من الـ drivers بل من قِبَل الـ framework نفسه.

**البارامترات:**
- `clkspec`: بنية تحمل الـ phandle ومعاملاته كما تم تفسيرها من الـ DT.

**القيمة المُرجَعة:** `struct clk *` أو `IS_ERR()`.

---

### ملخص دورة حياة الساعة الكاملة

```
┌─────────────────────────────────────────────────┐
│              دورة حياة الساعة                   │
│                                                 │
│  1. clk_get() / devm_clk_get()                 │
│     ↓ (الحصول على مرجع)                        │
│  2. clk_prepare()                               │
│     ↓ (تحضير - قد ينام، non-atomic)            │
│  3. clk_enable()                                │
│     ↓ (تشغيل - سريع، atomic ok)               │
│  4. [استخدام الساعة...]                        │
│     ↓                                           │
│  5. clk_disable()                               │
│     ↓ (إيقاف - سريع، atomic ok)               │
│  6. clk_unprepare()                             │
│     ↓ (إلغاء التحضير - قد ينام)               │
│  7. clk_put()                                   │
│     (تحرير المرجع)                             │
└─────────────────────────────────────────────────┘

الاختصار: clk_prepare_enable() = (2) + (3)
           clk_disable_unprepare() = (5) + (6)
           devm_clk_get_enabled() = (1) + (2) + (3) تلقائياً
```

### جدول المقارنة: العادي vs Managed vs Optional

| النمط | الدالة | تُحرَّر تلقائياً؟ | تتحمّل الغياب؟ |
|--------|--------|:-----------------:|:---------------:|
| العادي | `clk_get()` | لا | لا |
| Optional | `clk_get_optional()` | لا | نعم (NULL) |
| Managed | `devm_clk_get()` | نعم | لا |
| Managed+Optional | `devm_clk_get_optional()` | نعم | نعم (NULL) |
| Managed+Enabled | `devm_clk_get_enabled()` | نعم | لا |
| Managed+Opt+Enabled | `devm_clk_get_optional_enabled()` | نعم | نعم (NULL) |
| All-in-one | `devm_clk_get_optional_enabled_with_rate()` | نعم | نعم (NULL) |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs — كيف تقرأها

**الـ debugfs** هو زي لوحة عدادات السيارة — يعطيك نظرة داخلية على حالة الـ clock tree في الـ kernel بدون ما تحتاج تفتح غطاء المحرك.

أول خطوة: تأكد إن الـ debugfs متوصل:

```bash
# وصّل الـ debugfs لو مش موصول
mount -t debugfs debugfs /sys/kernel/debug

# تأكد إنه موجود
ls /sys/kernel/debug/clk/
```

##### المدخلات الأساسية:

| المدخل | المسار | وش يعطيك |
|--------|--------|-----------|
| `clk_summary` | `/sys/kernel/debug/clk/clk_summary` | شجرة كاملة لكل الـ clocks مع الـ rate والـ enable count |
| `clk_dump` | `/sys/kernel/debug/clk/clk_dump` | نفس المعلومات بصيغة JSON قابلة للـ parse |
| لكل clock مجلد خاص | `/sys/kernel/debug/clk/<clock_name>/` | معلومات clock بعينه |

##### مجلد كل clock يحتوي على:

```
/sys/kernel/debug/clk/<اسم_الـ_clock>/
├── clk_accuracy          ← الدقة بالـ ppb (parts per billion)
├── clk_duty_cycle        ← نسبة الـ duty cycle
├── clk_enable_count      ← عدد مرات clk_enable() بدون clk_disable()
├── clk_flags             ← الـ flags الداخلية
├── clk_min_rate          ← أقل rate مسموح
├── clk_max_rate          ← أعلى rate مسموح
├── clk_notifier_count    ← عدد الـ notifiers المسجلين
├── clk_parent            ← اسم الـ parent clock
├── clk_phase             ← الـ phase shift بالدرجات
├── clk_prepare_count     ← عدد مرات clk_prepare() بدون clk_unprepare()
└── clk_rate              ← الـ rate الحالي بالـ Hz
```

##### قراءة الـ clk_summary:

```bash
# اقرأ ملخص كل الـ clocks
cat /sys/kernel/debug/clk/clk_summary
```

**مثال على الـ output:**

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
---------------------------------------------------------------------------------------------
 osc24M                             1       1        0    24000000          0     0  50000
    pll-cpu                         1       1        0   816000000          0     0  50000
       cpu                          1       1        0   816000000          0     0  50000
    pll-audio                       0       0        0   786432000          0     0  50000
       audio-codec                  0       0        0    24576000          0     0  50000
```

**كيف تفهم الـ output:**
- **enable cnt**: لو هذا الرقم 0 والـ driver يشتكي — الـ clock مش enabled. لازم `clk_enable()` يتسمى.
- **prepare cnt**: لو 0 والـ enable cnt > 0 — مشكلة، مستحيل تعمل enable بدون prepare في معظم الأنظمة.
- **rate = 0**: إشارة خطر — الـ clock مش شغال أو في حالة خطأ.

##### قراءة الـ clk_dump (JSON):

```bash
# اقرأ بصيغة JSON وجمّلها
cat /sys/kernel/debug/clk/clk_dump | python3 -m json.tool | head -100
```

##### قراءة معلومات clock محدد:

```bash
CLOCK="pll-cpu"

# الـ rate الحالي
cat /sys/kernel/debug/clk/${CLOCK}/clk_rate

# عدد مرات الـ enable
cat /sys/kernel/debug/clk/${CLOCK}/clk_enable_count

# الـ parent
cat /sys/kernel/debug/clk/${CLOCK}/clk_parent

# الـ phase
cat /sys/kernel/debug/clk/${CLOCK}/clk_phase

# الـ accuracy بالـ ppb
cat /sys/kernel/debug/clk/${CLOCK}/clk_accuracy

# الـ duty cycle
cat /sys/kernel/debug/clk/${CLOCK}/clk_duty_cycle

# الـ notifiers المسجلين
cat /sys/kernel/debug/clk/${CLOCK}/clk_notifier_count
```

---

#### 2. مدخلات الـ sysfs

الـ **sysfs** يعطيك معلومات أقل تفصيلاً من الـ debugfs لكن متاحة دايماً حتى في production:

```bash
# تصفح devices وشوف الـ clocks المرتبطة
ls /sys/bus/platform/devices/
ls /sys/devices/platform/<device_name>/

# الـ power management يأثر على الـ clock state
cat /sys/devices/platform/<device_name>/power/runtime_status
cat /sys/devices/platform/<device_name>/power/runtime_suspended_time
cat /sys/devices/platform/<device_name>/power/runtime_active_time
```

##### للـ devices اللي فيها clocks في الـ Device Tree:

```bash
# شوف الـ clock-names المرتبطة بـ device
cat /sys/devices/platform/<device>/of_node/clock-names 2>/dev/null || \
  grep -r "clock-names" /proc/device-tree/<device_path>/
```

---

#### 3. الـ ftrace — تتبع أحداث الـ clock

**الـ ftrace** هو كاميرا مراقبة داخل الـ kernel — يسجل كل عملية على الـ clocks في الوقت الفعلي.

##### تفعيل الـ tracepoints:

```bash
# روح لمجلد الـ tracing
cd /sys/kernel/debug/tracing

# شوف كل الـ events المتعلقة بالـ clocks
ls events/clk/
```

**قائمة الـ events المهمة:**

| Event | وش يسجل |
|-------|---------|
| `clk_enable` | كل مرة يتسمى `clk_enable()` |
| `clk_enable_complete` | اكتمال الـ enable |
| `clk_disable` | كل مرة يتسمى `clk_disable()` |
| `clk_disable_complete` | اكتمال الـ disable |
| `clk_prepare` | كل مرة يتسمى `clk_prepare()` |
| `clk_prepare_complete` | اكتمال الـ prepare |
| `clk_unprepare` | كل مرة يتسمى `clk_unprepare()` |
| `clk_set_rate` | كل محاولة تغيير الـ rate |
| `clk_set_rate_complete` | اكتمال تغيير الـ rate |
| `clk_set_parent` | تغيير الـ parent |
| `clk_set_parent_complete` | اكتمال تغيير الـ parent |
| `clk_set_phase` | تغيير الـ phase |
| `clk_set_duty_cycle` | تغيير الـ duty cycle |

##### تفعيل الـ tracing خطوة بخطوة:

```bash
# خطوة 1: فعّل كل أحداث الـ clk
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# أو فعّل حدث بعينه
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable

# خطوة 2: امسح الـ trace القديم
echo > /sys/kernel/debug/tracing/trace

# خطوة 3: فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# خطوة 4: اعمل العملية اللي تريد تتتبعها
# مثلاً شغّل device معين أو driver

# خطوة 5: اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# خطوة 6: أوقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

##### سكريبت متكامل للـ tracing:

```bash
#!/bin/bash
TRACE_DIR=/sys/kernel/debug/tracing

# امسح القديم
echo > ${TRACE_DIR}/trace

# فعّل كل أحداث الـ clock
echo 1 > ${TRACE_DIR}/events/clk/enable

# فعّل الـ tracing
echo 1 > ${TRACE_DIR}/tracing_on

echo "Tracing enabled. Run your test now. Press ENTER to stop..."
read

# أوقف
echo 0 > ${TRACE_DIR}/tracing_on
echo 0 > ${TRACE_DIR}/events/clk/enable

# احفظ النتيجة
cat ${TRACE_DIR}/trace > /tmp/clk_trace_$(date +%Y%m%d_%H%M%S).txt
echo "Saved to /tmp/clk_trace_*.txt"
```

##### مثال على output الـ ftrace:

```
     systemd-1     [000] ....  1234.567890: clk_enable: pll-cpu
     systemd-1     [000] ....  1234.567891: clk_enable_complete: pll-cpu
     kworker/0:1   [000] ....  1235.100000: clk_set_rate: pll-cpu rate=1008000000
     kworker/0:1   [000] ....  1235.100050: clk_set_rate_complete: pll-cpu rate=1008000000
```

**كيف تقرأه:** اسم العملية — رقم الـ CPU — الوقت بالثواني — نوع الحدث — اسم الـ clock — التفاصيل.

---

#### 4. الـ printk والـ dynamic debug

##### تفعيل رسائل الـ debug للـ clock subsystem:

```bash
# الطريقة 1: dynamic debug — فعّل رسائل debug لملف معين
echo "file drivers/clk/clk.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/clk/clk-bulk.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل ملفات الـ clk
echo "file drivers/clk/* +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ module معين
echo "module clk-sunxi +p" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ filters الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep clk
```

##### تغيير مستوى الـ printk:

```bash
# الطريقة 2: رفع مستوى الـ kernel log
# (الأرقام من 0 إلى 8، الـ 8 هو الأعلى تفصيلاً)
echo 8 > /proc/sys/kernel/printk

# أو عبر dmesg
dmesg -n 8

# شوف الـ log في الوقت الفعلي
dmesg -w | grep -i clk
```

##### في وقت الـ boot — kernel boot params:

```
# في grub أو bootloader args:
loglevel=8 dyndbg="file drivers/clk/* +p"
```

---

#### 5. خيارات الـ Kernel Config للـ debugging

| الـ Config | الوصف | متى تفعّله |
|-----------|-------|-----------|
| `CONFIG_COMMON_CLK` | الـ framework الأساسي للـ clocks | لازم يكون مفعّل دايماً |
| `CONFIG_HAVE_CLK` | تفعيل الـ CLK API للـ platform | لازم يكون مفعّل |
| `CONFIG_HAVE_CLK_PREPARE` | دعم `clk_prepare()`/`clk_unprepare()` | في معظم الـ platforms الحديثة |
| `CONFIG_CLK_DEBUG` | يفعّل مجلد `/sys/kernel/debug/clk/` | أساسي للـ debugging |
| `CONFIG_DEBUG_FS` | يفعّل الـ debugfs كله | لازم لكل debugging |
| `CONFIG_TRACING` | الـ ftrace infrastructure | لازم لتتبع الأحداث |
| `CONFIG_CLK_FRACTIONAL_DIVIDER` | debug لـ fractional dividers | لو عندك هذا النوع |
| `CONFIG_CLK_COMPOSITE` | debug لـ composite clocks | لو عندك composite |
| `CONFIG_COMMON_CLK_SCPI` | debug لـ SCPI clocks | للـ ARM SCPI |
| `CONFIG_PM_CLK` | debug لـ clock management في الـ PM | لمشاكل الـ suspend/resume |
| `CONFIG_DEBUG_SPINLOCK` | يكشف مشاكل الـ locking في الـ clock ops | لمشاكل الـ concurrency |

##### للتحقق من الـ config الحالي:

```bash
# شوف الـ config المفعّل في الـ kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_(COMMON_CLK|HAVE_CLK|CLK_DEBUG|DEBUG_FS)"

# أو
cat /boot/config-$(uname -r) | grep -E "CONFIG_(COMMON_CLK|HAVE_CLK|CLK_DEBUG)"
```

---

#### 6. أدوات خاصة بالـ subsystem

##### الـ clk_summary كـ script:

```bash
#!/bin/bash
# سكريبت شامل لفحص حالة الـ clocks

echo "=== Clock Tree Summary ==="
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null || echo "debugfs not mounted"

echo ""
echo "=== Clocks with enable_count > 0 ==="
for clk_dir in /sys/kernel/debug/clk/*/; do
    clk_name=$(basename "$clk_dir")
    enable_count=$(cat "$clk_dir/clk_enable_count" 2>/dev/null)
    if [ -n "$enable_count" ] && [ "$enable_count" -gt 0 ] 2>/dev/null; then
        rate=$(cat "$clk_dir/clk_rate" 2>/dev/null)
        echo "  ${clk_name}: enabled=${enable_count}, rate=${rate} Hz"
    fi
done

echo ""
echo "=== Clocks with notifiers ==="
for clk_dir in /sys/kernel/debug/clk/*/; do
    clk_name=$(basename "$clk_dir")
    notif=$(cat "$clk_dir/clk_notifier_count" 2>/dev/null)
    if [ -n "$notif" ] && [ "$notif" -gt 0 ] 2>/dev/null; then
        echo "  ${clk_name}: notifiers=${notif}"
    fi
done
```

##### فحص الـ notifiers المسجلين:

```bash
# شوف كل الـ clocks اللي فيها notifiers
for f in /sys/kernel/debug/clk/*/clk_notifier_count; do
    count=$(cat $f 2>/dev/null)
    [ "$count" -gt 0 ] 2>/dev/null && echo "$f: $count"
done
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|------|
| `clk: couldn't get clock <name>` | ما قدر يجيب الـ clock المطلوب | تحقق إن الـ clock معرّف في الـ DT أو registered في الـ kernel |
| `-ENOENT` من `clk_get()` | الـ clock مش موجود في الـ system | تحقق من `clock-names` في الـ DT وتطابقها مع الـ driver |
| `-EPROBE_DEFER` من `clk_get()` | الـ clock provider ما تسجّل بعد | طبيعي — انتظر الـ driver ييجي مرة ثانية بعد ما الـ provider يجهز |
| `clk: failed to enable clock <name>` | فشل `clk_enable()` | تحقق إن `clk_prepare()` اتسمى أولاً |
| `WARNING: CPU: 0 PID: 0 at kernel/time/... clk_enable` | حاولت `clk_enable()` في atomic context | استخدم `clk_prepare_enable()` خارج الـ atomic context |
| `clk: <name> already prepared` | الـ prepare count غير متوازن | تحقق إن `clk_unprepare()` يتسمى عدد مرات مساوية |
| `clk: <name> already enabled` | الـ enable count غير متوازن | تحقق إن `clk_disable()` يتسمى عدد مرات مساوية |
| `clk: <name>: rate change failed` | فشل `clk_set_rate()` | تحقق من `clk_round_rate()` أولاً وتأكد الـ rate في النطاق المسموح |
| `clk: <name>: set rate on clock with exclusive rate` | محاولة تغيير rate محمي بـ exclusive | لازم `clk_rate_exclusive_put()` أولاً |
| `could not get clock <name>: -EINVAL` | الـ clock معرّف لكن المعامل غلط | تحقق من اسم الـ clock بالضبط كما في الـ DT |
| `clk: <name>: failed to set parent` | فشل `clk_set_parent()` | الـ parent المطلوب مش في قائمة الـ possible parents |
| `clk: orphan clock <name>` | الـ clock ما لقى الـ parent عند التسجيل | الـ parent ما تسجّل بعد أو اسمه غلط |
| `clk: <name>: rate 0 is invalid` | محاولة set_rate بقيمة 0 | تحقق من الـ rate الممرر |
| `prepare_count: 0 but enable_count: N` | خلل في الـ reference counting | bug في الـ driver — enable بدون prepare |

---

#### 8. أماكن استراتيجية لـ dump_stack() وWARN_ON()

زي ما تحط نقطة توقف في كود C عادي، تقدر تحط `WARN_ON()` و`dump_stack()` في نقاط حرجة لتشخيص المشاكل:

```c
/* في callback الـ notifier — تحقق إن البيانات صحيحة */
static int my_clk_notifier_cb(struct notifier_block *nb,
                               unsigned long event, void *data)
{
    struct clk_notifier_data *ndata = data;

    /* تحقق إن الـ clock مش NULL */
    if (WARN_ON(!ndata->clk)) {
        return NOTIFY_BAD;
    }

    /* تحقق إن الـ rate جديد منطقي */
    if (WARN_ON(ndata->new_rate == 0 && event == POST_RATE_CHANGE)) {
        pr_err("Clock %s went to 0 Hz!\n",
               __clk_get_name(ndata->clk));
        dump_stack(); /* اطبع الـ call stack */
        return NOTIFY_BAD;
    }

    /* تحقق من عدم تجاوز النطاق */
    WARN_ON(ndata->new_rate > MAX_EXPECTED_RATE);

    return NOTIFY_OK;
}
```

```c
/* في driver probe — تحقق من الـ clock قبل الاستخدام */
static int my_driver_probe(struct platform_device *pdev)
{
    struct clk *clk;
    unsigned long rate;

    clk = devm_clk_get(&pdev->dev, "my-clock");
    if (WARN_ON(IS_ERR(clk))) {
        /* dump_stack() هنا يساعد تشوف مين استدعى الـ probe */
        dump_stack();
        return PTR_ERR(clk);
    }

    rate = clk_get_rate(clk);
    /* تحقق إن الـ rate في النطاق المتوقع */
    WARN_ON(rate < MIN_RATE || rate > MAX_RATE);

    return 0;
}
```

**النقاط الاستراتيجية للـ WARN_ON:**
1. بعد `clk_get()` مباشرة — تحقق من الـ error
2. قبل `clk_set_rate()` — تحقق إن الـ rate منطقي
3. في `PRE_RATE_CHANGE` notifier — تحقق من الـ new_rate
4. في `clk_prepare()` — تحقق من الـ context (not atomic)
5. عند `clk_put()` — تحقق إن الـ enable count = 0

---

### Hardware Level

---

#### 1. التحقق من حالة الـ hardware تطابق حالة الـ kernel

الـ kernel يحتفظ بنسخته من حالة الـ clocks في الذاكرة. لكن هل الـ hardware فعلاً بيقول نفس الكلام؟

```
kernel state (clk_summary)  ←→  hardware registers  ←→  actual signal
```

##### خطوات التحقق:

```bash
# خطوة 1: اقرأ الـ rate من الـ kernel
cat /sys/kernel/debug/clk/pll-cpu/clk_rate
# مثلاً يعطيك: 816000000

# خطوة 2: اقرأ الـ register المقابل مباشرة من الـ hardware
# (يختلف العنوان حسب الـ SoC — راجع datasheet)
devmem2 0x01c20000 w   # مثال لـ Allwinner H3 PLL_CPUX_CTRL_REG

# خطوة 3: احسب الـ rate من الـ register يدوياً وقارن مع الـ kernel
```

---

#### 2. تقنيات قراءة الـ registers مباشرة

##### باستخدام devmem2:

```bash
# تثبيت devmem2 لو مش موجود
apt-get install devmem2  # Debian/Ubuntu
# أو
busybox devmem <address>

# قراءة register واحد (32-bit word)
devmem2 0x01C20000 w

# قراءة عدة registers متتالية
for offset in 0 4 8 12 16 20; do
    addr=$(printf "0x%08X" $((0x01C20000 + offset)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value at" | awk '{print $NF}')
    echo "  ${addr}: ${val}"
done
```

##### باستخدام /dev/mem:

```bash
# قراءة مباشرة من /dev/mem باستخدام dd
# قراءة 64 بايت ابتداءً من العنوان 0x01C20000
dd if=/dev/mem bs=4 count=16 skip=$((0x01C20000/4)) 2>/dev/null | xxd

# أو باستخدام python
python3 -c "
import mmap, struct, os

fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
base = 0x01C20000
size = 0x100

m = mmap.mmap(fd, size, mmap.MAP_SHARED, mmap.PROT_READ, offset=base)
for i in range(0, size, 4):
    val = struct.unpack('<I', m[i:i+4])[0]
    print(f'  0x{base+i:08X}: 0x{val:08X}  ({val:032b})')
m.close()
os.close(fd)
"
```

##### باستخدام io utility:

```bash
# io utility موجود في بعض الـ embedded systems
io -4 -r 0x01C20000    # قراءة 32-bit
io -4 0x01C20000 0x100 # قراءة range
```

**ملاحظة مهمة:** الـ `/dev/mem` يحتاج `CONFIG_DEVMEM=y` في الـ kernel config وصلاحيات root. في بعض الأنظمة يحتاج `CONFIG_STRICT_DEVMEM=n`.

---

#### 3. الـ Oscilloscope وعداد التردد — نصائح عملية

##### ما تحتاجه:

- **Oscilloscope** أو **Logic Analyzer** مع دعم تحليل التردد
- **Frequency Counter** دقيق (مثل Siglent SDM3055 أو أي عداد USB)
- **Test points** على الـ PCB عند خطوط الـ clock

##### التحقق من خروج الـ clock فعلياً:

```
SoC Pin → Test Point → Oscilloscope Probe
```

**نقاط يجب فحصها:**
1. **Crystal oscillator output** (عادةً 24 MHz أو 25 MHz أو 32.768 kHz)
2. **PLL output** (تتحقق منه بالـ scope على pinout الـ SoC)
3. **Clock to peripherals** (UART, SPI, I2C clock lines)

##### نصائح عملية مع الـ Oscilloscope:

```
للـ clock السريع (> 100 MHz):
  - استخدم probe بـ 10x attenuation مع compensation صحيح
  - قصّر السلك بين الـ probe tip والـ ground clip قدر الإمكان
  - Bandwidth لا تقل عن 200 MHz للقياسات الدقيقة

للـ clock البطيء (< 10 MHz):
  - 1x probe كافي
  - تحقق من الـ duty cycle (يجب أن يكون 50% للـ clocks العادية)
  - ابحث عن الـ jitter الزائد

للتحقق من الـ phase:
  - استخدم 2-channel scope
  - Channel 1: reference clock
  - Channel 2: derived clock
  - قيس الفرق الزمني بين الـ rising edges
```

##### ربط القراءة بالـ kernel:

```bash
# الـ kernel يقول الـ PLL يعطي 816 MHz
cat /sys/kernel/debug/clk/pll-cpu/clk_rate
# → 816000000

# تأكد بالـ scope: ابحث عن نبضة بتردد 816 MHz
# إذا الـ scope يقول 408 MHz → مشكلة في الـ divider settings
# إذا الـ scope يقول 0 / لا إشارة → الـ PLL مش locked أو مطفي
```

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | ما يظهر في dmesg | التحقق |
|---------|-----------------|--------|
| الـ Crystal مش شغال | `clk: failed to get rate for osc` أو rate = 0 | قيس الـ crystal بالـ scope على الـ OSC pins |
| الـ PLL ما اقفل (not locked) | `pll: timeout waiting for lock` | تحقق من supply voltage للـ PLL و external components |
| clock مش وصل للـ peripheral | `device failed to probe: -ETIMEDOUT` بعد clock ops | تحقق من وصول الإشارة فعلياً للـ chip |
| الـ duty cycle غلط | مشاكل في استقبال الـ SPI/I2C/UART | قيس الـ duty cycle بالـ scope، يجب 50% |
| noise على خط الـ clock | intermittent device errors وـ CRC errors | تحقق من الـ decoupling capacitors والـ ground plane |
| wrong parent | rate مختلف عن المتوقع تماماً | قارن `clk_parent` في debugfs مع الـ DT |

---

#### 5. الـ Device Tree Debugging — تحقق من الـ clocks

الـ **Device Tree** هو الخريطة اللي تخبر الـ kernel بالـ hardware — لو الخريطة غلط، كل شيء سيفشل.

##### التحقق من الـ DT في الـ runtime:

```bash
# شوف الـ DT المحمّل فعلاً في الـ kernel
ls /proc/device-tree/
ls /proc/device-tree/clocks/

# اقرأ الـ clock node
cat /proc/device-tree/clocks/osc24M/compatible
cat /proc/device-tree/clocks/osc24M/clock-frequency

# شوف device معين وعلاقته بالـ clocks
ls /proc/device-tree/soc/uart@1C28000/
cat /proc/device-tree/soc/uart@1C28000/clock-names
# → apb1_uart0

# تحقق من الـ phandle للـ clock
hexdump -C /proc/device-tree/soc/uart@1C28000/clocks
```

##### فحص الـ DT source مقابل الـ hardware:

```bash
# فك الـ DTB إلى DTS للمراجعة
dtc -I dtb -O dts /proc/device-tree > /tmp/current_dt.dts

# ابحث عن الـ clock nodes
grep -A 10 "clocks {" /tmp/current_dt.dts | head -50

# ابحث عن device معين وعلاقته بالـ clocks
grep -B 2 -A 15 "uart@1C28000" /tmp/current_dt.dts
```

##### مثال DT صحيح:

```dts
/* مثال: UART يأخذ clock من APB */
uart0: serial@01c28000 {
    compatible = "snps,dw-apb-uart";
    reg = <0x01c28000 0x400>;
    /* clock-names يجب أن يطابق ما يطلبه الـ driver بالضبط */
    clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>;
    clock-names = "apb", "uart";
    /* لو clock-names غلط → -ENOENT عند clk_get() */
};
```

##### أخطاء DT شائعة:

```bash
# خطأ 1: clock-names لا تطابق ما يطلبه الـ driver
# الـ driver يطلب: devm_clk_get(dev, "apb")
# الـ DT يقول: clock-names = "apb_clk"  ← خطأ!
# النتيجة: -ENOENT

# كيف تشخّص؟
dmesg | grep "ENOENT\|couldn't get clock\|failed to get"

# خطأ 2: الـ phandle غلط (يشير لـ node غير موجود)
# النتيجة: -EPROBE_DEFER أو -EINVAL

# خطأ 3: ناسي تعريف الـ #clock-cells في الـ provider
# النتيجة: of_clk_get() يفشل
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

##### فحص شامل سريع:

```bash
#!/bin/bash
# ==========================================
# سكريبت تشخيص شامل للـ Clock Subsystem
# ==========================================

echo "════════════════════════════════════════"
echo "      CLK Subsystem Debug Report"
echo "════════════════════════════════════════"
echo ""

# 1. تأكد من وجود debugfs
if ! mountpoint -q /sys/kernel/debug; then
    echo "[!] debugfs not mounted. Mounting..."
    mount -t debugfs debugfs /sys/kernel/debug
fi

# 2. شجرة الـ clocks الكاملة
echo "--- Clock Tree ---"
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null || echo "N/A"
echo ""

# 3. الـ clocks اللي enabled
echo "--- Enabled Clocks ---"
for d in /sys/kernel/debug/clk/*/; do
    name=$(basename "$d")
    ec=$(cat "$d/clk_enable_count" 2>/dev/null || echo 0)
    if [ "$ec" -gt 0 ] 2>/dev/null; then
        rate=$(cat "$d/clk_rate" 2>/dev/null || echo "?")
        pc=$(cat "$d/clk_prepare_count" 2>/dev/null || echo "?")
        echo "  ✓ ${name}: rate=${rate}Hz, enable=${ec}, prepare=${pc}"
    fi
done
echo ""

# 4. آخر رسائل الـ kernel المتعلقة بالـ clocks
echo "--- Recent CLK kernel messages ---"
dmesg | grep -iE "clk|clock" | tail -20
echo ""

echo "════════════════════════════════════════"
echo "Done."
```

##### فحص clock محدد بالتفصيل:

```bash
#!/bin/bash
CLOCK="${1:-pll-cpu}"  # مرر اسم الـ clock كـ argument أو يستخدم pll-cpu
D="/sys/kernel/debug/clk/${CLOCK}"

if [ ! -d "$D" ]; then
    echo "Clock '${CLOCK}' not found in debugfs"
    echo "Available clocks:"
    ls /sys/kernel/debug/clk/
    exit 1
fi

echo "=== Detailed info for: ${CLOCK} ==="
echo ""
printf "  %-20s %s\n" "Rate (Hz):"         "$(cat $D/clk_rate 2>/dev/null)"
printf "  %-20s %s\n" "Parent:"             "$(cat $D/clk_parent 2>/dev/null)"
printf "  %-20s %s\n" "Enable count:"       "$(cat $D/clk_enable_count 2>/dev/null)"
printf "  %-20s %s\n" "Prepare count:"      "$(cat $D/clk_prepare_count 2>/dev/null)"
printf "  %-20s %s\n" "Phase (degrees):"   "$(cat $D/clk_phase 2>/dev/null)"
printf "  %-20s %s\n" "Duty cycle:"         "$(cat $D/clk_duty_cycle 2>/dev/null)"
printf "  %-20s %s\n" "Accuracy (ppb):"     "$(cat $D/clk_accuracy 2>/dev/null)"
printf "  %-20s %s\n" "Min rate:"           "$(cat $D/clk_min_rate 2>/dev/null)"
printf "  %-20s %s\n" "Max rate:"           "$(cat $D/clk_max_rate 2>/dev/null)"
printf "  %-20s %s\n" "Notifier count:"     "$(cat $D/clk_notifier_count 2>/dev/null)"
printf "  %-20s %s\n" "Flags:"              "$(cat $D/clk_flags 2>/dev/null)"
```

##### تفعيل كل الـ tracing دفعة واحدة:

```bash
#!/bin/bash
T=/sys/kernel/debug/tracing

# فعّل كل أحداث الـ clk
for event in clk_enable clk_enable_complete \
             clk_disable clk_disable_complete \
             clk_prepare clk_prepare_complete \
             clk_unprepare clk_unprepare_complete \
             clk_set_rate clk_set_rate_complete \
             clk_set_parent clk_set_parent_complete \
             clk_set_phase clk_set_duty_cycle; do
    f="${T}/events/clk/${event}/enable"
    [ -f "$f" ] && echo 1 > "$f" && echo "Enabled: $event"
done

echo > ${T}/trace
echo 1 > ${T}/tracing_on
echo "Tracing all CLK events. Ctrl+C to stop and save..."

trap 'echo 0 > ${T}/tracing_on
      OUT="/tmp/clk_trace_$(date +%Y%m%d_%H%M%S).txt"
      cp ${T}/trace $OUT
      echo ""
      echo "Saved to: $OUT"
      echo "Total events: $(wc -l < $OUT)"
      exit 0' INT

cat ${T}/trace_pipe
```

##### قراءة registers مع تحليل تلقائي:

```bash
#!/bin/bash
# مثال لـ Allwinner H3/H5 CCU registers
# عدّل العناوين حسب الـ SoC الخاص بك

CCU_BASE=0x01C20000

declare -A REGS=(
    ["PLL_CPUX_CTRL"]="0x000"
    ["PLL_AUDIO_CTRL"]="0x008"
    ["PLL_VIDEO_CTRL"]="0x010"
    ["PLL_VE_CTRL"]="0x018"
    ["PLL_DDR_CTRL"]="0x020"
    ["PLL_PERIPH0_CTRL"]="0x028"
    ["AHB1_APB1_DIV"]="0x054"
    ["APB2_DIV"]="0x058"
    ["CPUX_AXI_CFG"]="0x050"
)

echo "=== CCU Register Dump ==="
for name in "${!REGS[@]}"; do
    offset="${REGS[$name]}"
    addr=$(printf "0x%08X" $((CCU_BASE + 16#${offset#0x})))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value at" | awk '{print $NF}')
    echo "  ${name} (${addr}): ${val}"
done
```

##### فحص مشكلة EPROBE_DEFER:

```bash
# لو driver يعاني من EPROBE_DEFER متكرر
dmesg | grep -E "deferred|EPROBE_DEFER" | tail -20

# شوف الـ deferred devices
cat /sys/kernel/debug/devices_deferred 2>/dev/null || \
  ls /sys/kernel/debug/ | grep defer
```

##### فحص الـ duty cycle لـ clock معين:

```bash
# اقرأ الـ duty cycle من debugfs (يرجع كـ numerator/denominator)
cat /sys/kernel/debug/clk/your-clock/clk_duty_cycle

# يجب أن يكون قريب من 50000/100000 للـ clocks العادية
# أي 50%
```

##### سكريبت مقارنة kernel vs hardware للـ rate:

```bash
#!/bin/bash
# قارن الـ rate اللي يقوله الـ kernel مع حسابك اليدوي من الـ register

CLOCK="pll-cpu"
KERNEL_RATE=$(cat /sys/kernel/debug/clk/${CLOCK}/clk_rate)

# اقرأ الـ PLL register (هذا مثال - عدّل العنوان لـ SoC خاصتك)
PLL_REG=$(devmem2 0x01C20000 w 2>/dev/null | grep "Value at" | awk '{print $NF}')

echo "Kernel reports: ${KERNEL_RATE} Hz"
echo "Raw PLL register: ${PLL_REG}"
echo ""
echo "Manual calculation (edit formula for your SoC):"
echo "  P = bits[15:8], N = bits[7:0], M = bits[19:16]"
echo "  Rate = (OSC_24M * (N+1)) / ((M+1) * (2^P))"
```

---

#### تفسير النتائج — أمثلة

##### مثال 1: clock enabled_count غير متوازن

```
clock               enable_cnt  prepare_cnt
spi-clk                  3           3
```

**التفسير:** ٣ drivers (أو نفس الـ driver ٣ مرات) طلبوا الـ enable. الـ clock ما يطفأ إلا لما الـ 3 يسووا `clk_disable()`.

##### مثال 2: اكتشاف clock مش شغال بالـ ftrace

```
kworker/0:2   [001]  5678.901234: clk_set_rate: uart0-clk rate=115200
kworker/0:2   [001]  5678.901300: clk_set_rate_complete: uart0-clk rate=0
```

**التفسير:** الـ rate المطلوب 115200 لكن النتيجة 0! هذا يعني الـ `.set_rate()` operation في الـ clock driver فشل. افحص الـ driver code.

##### مثال 3: mismatch بين kernel وhardware

```bash
cat /sys/kernel/debug/clk/pll-cpu/clk_rate
# → 816000000

devmem2 0x01C20000 w
# → 0x82001900

# حساب يدوي:
# N = bits[7:0] = 0x19 = 25 → N+1 = 26
# الـ rate = 24MHz * 26 = 624 MHz ← مختلف!
```

**التفسير:** الـ kernel محسوب الـ rate غلط لأن الـ `.recalc_rate()` في الـ clock driver عنده bug في تفسير الـ register fields. راجع الـ datasheet للـ SoC.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART صامت على Gateway صناعي بـ RK3562

#### العنوان
**الـ UART لا يشتغل بعد suspend/resume على gateway صناعي**

#### السياق
شركة تبني industrial gateway بمعالج **RK3562** يجمع بيانات من حساسات عبر **UART** بـ 115200 baud. الجهاز يدخل في **suspend** كل ساعة لتوفير الطاقة، ثم يرجع بـ wakeup مبرمج. المشكلة ظهرت في الإنتاج: بعد أول suspend/resume، الـ UART يصبح صامتاً تماماً، لا يستقبل ولا يرسل.

#### المشكلة
الـ UART driver يستخدم `clk_prepare_enable()` في `probe()` مرة واحدة، وما فيه أي تعامل مع `clk_save_context()` و `clk_restore_context()`. بعد الـ resume، سجلات الـ clock controller على الـ RK3562 رجعت لقيمها الافتراضية، لكن الـ driver ما أعاد تفعيل الـ clock.

#### التحليل
في `clk.h`، عندنا:

```c
/* في clk.h — دالتان مهمتان لإدارة السياق */
int clk_save_context(void);   /* يحفظ سجلات الـ clock قبل poweroff */
void clk_restore_context(void); /* يستعيدها بعد resume */
```

الكود ده بيوضح:
```
/* التدفق الخاطئ */
probe():
    clk_prepare_enable(uart_clk)  ← OK

suspend():
    /* لا شيء! الـ clock سجلاته راحت */

resume():
    /* الـ clock hardware reset، لكن ما حد استعاده */
    /* UART يحاول يشتغل بدون clock → صمت تام */
```

الكود الصحيح يجب أن يعتمد على `clk_save_context()` / `clk_restore_context()` اللي بتتعامل مع حالة انقطاع كامل في السجلات، أو يستخدم `clk_disable_unprepare()` / `clk_prepare_enable()` في حلقة suspend/resume:

```c
/* clk_prepare_enable: ينفذ خطوتين بترتيب صحيح */
static inline int clk_prepare_enable(struct clk *clk)
{
    int ret;
    ret = clk_prepare(clk);   /* prepare أولاً — قد ينام */
    if (ret)
        return ret;
    ret = clk_enable(clk);    /* enable ثانياً — atomic */
    if (ret)
        clk_unprepare(clk);   /* rollback لو فشل */
    return ret;
}

/* clk_disable_unprepare: عكس الترتيب */
static inline void clk_disable_unprepare(struct clk *clk)
{
    clk_disable(clk);    /* disable أولاً */
    clk_unprepare(clk);  /* unprepare ثانياً */
}
```

#### الحل

```c
/* في uart_driver.c — إضافة suspend/resume handlers */

static int rk3562_uart_suspend(struct device *dev)
{
    struct uart_priv *priv = dev_get_drvdata(dev);

    /* أوقف الـ clock بالترتيب الصحيح */
    clk_disable_unprepare(priv->uart_clk);
    return 0;
}

static int rk3562_uart_resume(struct device *dev)
{
    struct uart_priv *priv = dev_get_drvdata(dev);
    int ret;

    /* أعد تفعيل الـ clock بالترتيب الصحيح */
    ret = clk_prepare_enable(priv->uart_clk);
    if (ret) {
        dev_err(dev, "فشل في إعادة تفعيل uart_clk: %d\n", ret);
        return ret;
    }
    return 0;
}

static DEFINE_SIMPLE_DEV_PM_OPS(rk3562_uart_pm_ops,
    rk3562_uart_suspend, rk3562_uart_resume);
```

للتشخيص السريع:
```bash
# تحقق من حالة الـ clocks بعد resume
cat /sys/kernel/debug/clk/clk_summary | grep uart

# راقب الـ clock enable count
cat /sys/kernel/debug/clk/uart_clk/clk_enable_count
```

#### الدرس المستفاد
**الـ `clk_prepare_enable()` في `probe()` وحده لا يكفي.** أي driver يعمل على hardware قد يفقد سجلاته في suspend يجب أن ينفذ `clk_disable_unprepare()` في suspend و `clk_prepare_enable()` في resume. الـ `clk_save_context()` و `clk_restore_context()` مخصصان لطبقة أعمق (platform-level) وليس للـ drivers الفردية.

---

### السيناريو الثاني: تشويش في بيانات SPI على لوحة IoT بـ STM32MP1

#### العنوان
**بيانات SPI مشوشة بعد تغيير تردد الـ CPU على حساس IoT**

#### السياق
لوحة IoT مبنية على **STM32MP1** تقرأ بيانات من حساس ضغط عبر **SPI** بسرعة 10 MHz. النظام يستخدم **cpufreq** لتغيير تردد الـ CPU حسب الحمل. في الإنتاج، المهندسون لاحظوا أن قراءات الحساس تصبح عشوائية عند ارتفاع حمل المعالج وتغيير تردده.

#### المشكلة
الـ SPI clock مشتق من الـ CPU PLL (أو parent مشترك). لما يتغير تردد الـ CPU، يتغير معه تردد الـ SPI بشكل غير متوقع، مما يكسر الاتصال مع الحساس. الـ driver لم يستخدم `clk_notifier_register()` ليكون على علم بالتغيير.

#### التحليل
الـ header يعرّف نظام الـ notifiers للـ clock:

```c
/* المعرّفات في clk.h */
#define PRE_RATE_CHANGE   BIT(0)  /* قبل التغيير  */
#define POST_RATE_CHANGE  BIT(1)  /* بعد التغيير  */
#define ABORT_RATE_CHANGE BIT(2)  /* لو التغيير فشل */

/* struct تحمل بيانات التغيير */
struct clk_notifier_data {
    struct clk    *clk;
    unsigned long  old_rate;  /* التردد القديم */
    unsigned long  new_rate;  /* التردد الجديد */
};

/* الدالة للتسجيل */
int clk_notifier_register(struct clk *clk, struct notifier_block *nb);
```

التدفق الحالي الخاطئ:
```
cpufreq يطلب رفع تردد CPU
    ↓
clk framework يغير PLL
    ↓
تردد SPI يتغير تلقائياً (مشتق منه)
    ↓
SPI driver لا يعلم ← بيانات مشوشة!
```

#### الحل

```c
/* في spi_sensor_driver.c */

static int stm32mp1_spi_clk_notify(struct notifier_block *nb,
                                    unsigned long action, void *data)
{
    struct clk_notifier_data *cnd = data;
    struct spi_priv *priv = container_of(nb, struct spi_priv, clk_nb);

    switch (action) {
    case PRE_RATE_CHANGE:
        /* قبل التغيير: أوقف أي transaction جارية */
        dev_dbg(priv->dev, "تردد الـ SPI سيتغير من %lu إلى %lu Hz\n",
                cnd->old_rate, cnd->new_rate);
        mutex_lock(&priv->transfer_lock);
        priv->clock_changing = true;
        /* انتظر انتهاء أي DMA transfer جارية */
        complete_all(&priv->transfer_done);
        break;

    case POST_RATE_CHANGE:
        /* بعد التغيير: أعد حساب divisor وفتح الباب */
        priv->actual_spi_freq = cnd->new_rate / priv->clk_div;
        dev_dbg(priv->dev, "تردد الـ SPI الجديد: %lu Hz\n",
                priv->actual_spi_freq);
        priv->clock_changing = false;
        mutex_unlock(&priv->transfer_lock);
        break;

    case ABORT_RATE_CHANGE:
        /* التغيير ألغي — أعد الأمور لحالتها */
        priv->clock_changing = false;
        mutex_unlock(&priv->transfer_lock);
        break;
    }

    return NOTIFY_DONE;
}

static int stm32mp1_spi_probe(struct platform_device *pdev)
{
    struct spi_priv *priv;
    int ret;

    /* ... باقي الـ probe ... */

    priv->spi_clk = devm_clk_get(&pdev->dev, "spi");
    if (IS_ERR(priv->spi_clk))
        return PTR_ERR(priv->spi_clk);

    /* سجّل الـ notifier لتلقي إشعارات تغيير التردد */
    priv->clk_nb.notifier_call = stm32mp1_spi_clk_notify;
    ret = devm_clk_notifier_register(&pdev->dev, priv->spi_clk, &priv->clk_nb);
    if (ret) {
        dev_err(&pdev->dev, "فشل تسجيل clk notifier: %d\n", ret);
        return ret;
    }

    return 0;
}
```

#### الدرس المستفاد
عندما يكون الـ clock مشتركاً بين أجهزة ذات حساسية للتردد (مثل SPI, I2C, UART)، يجب تسجيل `clk_notifier_register()` لتلقي إشعار `PRE_RATE_CHANGE` وإيقاف العمليات قبل حدوث التغيير. الـ `devm_clk_notifier_register()` أفضل لأنه يلغي التسجيل تلقائياً عند إزالة الـ driver.

---

### السيناريو الثالث: HDMI لا يظهر صورة على Android TV Box بـ Allwinner H616

#### العنوان
**الـ HDMI يبقى أسود على TV Box بسبب خطأ في clk_set_rate_exclusive**

#### السياق
شركة تبني **Android TV Box** بمعالج **Allwinner H616**. الجهاز يدعم **HDMI** بدقة 4K@60Hz. المشكلة ظهرت عند تشغيل تطبيقين يطلبان HDMI في نفس الوقت (مثل Miracast + DisplayPort tunneling): الشاشة تصبح سوداء ويظهر في dmesg خطأ `-EBUSY`.

#### المشكلة
Driver الـ HDMI استخدم `clk_set_rate_exclusive()` لحجز تردد الـ HDMI PLL وضمان عدم تغييره، لكنه لم يستدعِ `clk_rate_exclusive_put()` في حالة الخطأ، مما أدى إلى lock دائم على الـ clock.

#### التحليل

```c
/* من clk.h — الفرق بين الدالتين */

/* هذه تضبط التردد وتحجز الـ clock لمستخدم واحد فقط */
int clk_set_rate_exclusive(struct clk *clk, unsigned long rate);

/* هذه تحرر الحجز — يجب استدعاؤها لكل clk_set_rate_exclusive ناجح */
void clk_rate_exclusive_put(struct clk *clk);

/* ملاحظة في التوثيق:
 * "If exclusivity is claimed more than once on clock, even by the same driver,
 *  the rate effectively gets locked as exclusivity can't be preempted."
 */
```

الكود الخاطئ:
```c
static int h616_hdmi_enable(struct hdmi_priv *priv)
{
    int ret;

    /* ضبط تردد HDMI PLL وحجزه */
    ret = clk_set_rate_exclusive(priv->hdmi_pll, 594000000); /* 594 MHz لـ 4K60 */
    if (ret) {
        dev_err(priv->dev, "فشل ضبط HDMI PLL: %d\n", ret);
        return ret;
        /* ← BUG: لو clk_set_rate_exclusive نجح جزئياً ثم فشل لسبب آخر،
           الـ exclusive lock قد يكون محجوزاً بدون إفراج */
    }

    ret = hdmi_phy_enable(priv);  /* لو فشل هذا */
    if (ret) {
        /* ← BUG: نسي clk_rate_exclusive_put! */
        return ret;
    }

    return 0;
}
```

#### الحل

```c
static int h616_hdmi_enable(struct hdmi_priv *priv)
{
    int ret;

    /* ضبط التردد وحجز الـ clock */
    ret = clk_set_rate_exclusive(priv->hdmi_pll, 594000000UL);
    if (ret) {
        dev_err(priv->dev, "فشل ضبط HDMI PLL: %d\n", ret);
        return ret;
    }
    /* من هنا يجب ضمان استدعاء clk_rate_exclusive_put عند أي خطأ */

    ret = hdmi_phy_enable(priv);
    if (ret) {
        dev_err(priv->dev, "فشل تفعيل HDMI PHY: %d\n", ret);
        clk_rate_exclusive_put(priv->hdmi_pll); /* ← الإصلاح */
        return ret;
    }

    ret = hdmi_controller_start(priv);
    if (ret) {
        hdmi_phy_disable(priv);
        clk_rate_exclusive_put(priv->hdmi_pll); /* ← الإصلاح */
        return ret;
    }

    priv->exclusive_held = true; /* تذكر أنك حاجز */
    return 0;
}

static void h616_hdmi_disable(struct hdmi_priv *priv)
{
    hdmi_controller_stop(priv);
    hdmi_phy_disable(priv);

    if (priv->exclusive_held) {
        clk_rate_exclusive_put(priv->hdmi_pll);
        priv->exclusive_held = false;
    }
}
```

للتشخيص:
```bash
# تحقق من exclusive count على الـ clock
cat /sys/kernel/debug/clk/hdmi_pll/clk_exclusive_count

# تحقق من التردد الحالي
cat /sys/kernel/debug/clk/hdmi_pll/clk_rate
```

#### الدرس المستفاد
`clk_set_rate_exclusive()` و `clk_rate_exclusive_put()` يجب أن يكونا **متوازنين دائماً** مثل `mutex_lock/unlock` تماماً. أي مسار خطأ يجب أن يستدعي `clk_rate_exclusive_put()`. استخدم `devm_clk_rate_exclusive_get()` كلما أمكن لأنه يتعامل مع التنظيف تلقائياً.

---

### السيناريو الرابع: كاميرا MIPI-CSI تتجمد على لوحة AM62x

#### العنوان
**الـ MIPI-CSI camera تتجمد بعد 30 ثانية بسبب clk_bulk_enable غير مكتمل**

#### السياق
شركة تبني نظام رؤية حاسوبية لتطبيق **automotive** على معالج **TI AM62x**. الكاميرا تتصل عبر **MIPI-CSI-2** وتُغذي بيانات لـ ISP. المشكلة: الكاميرا تشتغل لمدة 30 ثانية ثم تتجمد الصورة، والنظام يسجل timeout في الـ ISP DMA.

#### المشكلة
Driver الـ CSI يستخدم `clk_bulk_data` لإدارة 3 clocks: `csi_clk`, `isp_clk`, `dphy_clk`. المشكل أن `clk_bulk_enable()` يُرجع قيمة يجب التحقق منها (`__must_check`)، لكن الكود يتجاهلها.

#### التحليل

```c
/* من clk.h */
int __must_check clk_bulk_enable(int num_clks,
                                  const struct clk_bulk_data *clks);
```

الـ `__must_check` يعني المترجم يحذرك إذا تجاهلت القيمة المُرجعة. لكن لو أسكتّ التحذيرات أو فاتك التحذير:

```c
/* الكود الخاطئ */
static int am62x_csi_start_streaming(struct csi_priv *priv)
{
    /* ← BUG: تجاهل قيمة الإرجاع */
    clk_bulk_enable(priv->num_clks, priv->clks);

    /* الـ ISP DMA يبدأ بدون التحقق */
    return start_dma_transfer(priv);
}
```

ماذا يحدث؟ لو `dphy_clk` (ثالث clock في المصفوفة) فشل في الـ enable:
- `clk_bulk_enable` تُرجع خطأ ويُوقف الـ clocks التي فعّلها
- الـ ISP DMA يبدأ بدون DPHY clock → يشتغل 30 ثانية من الـ cache/buffer ثم يتجمد

```c
/* من clk.h — clk_bulk_prepare_enable تجمع الخطوتين */
static inline int __must_check
clk_bulk_prepare_enable(int num_clks, const struct clk_bulk_data *clks)
{
    int ret;
    ret = clk_bulk_prepare(num_clks, clks);
    if (ret)
        return ret;
    ret = clk_bulk_enable(num_clks, clks);
    if (ret)
        clk_bulk_unprepare(num_clks, clks); /* rollback تلقائي */
    return ret;
}
```

#### الحل

```c
/* في am62x_csi_driver.c */

/* تعريف الـ clocks المطلوبة */
static const char * const am62x_csi_clk_ids[] = {
    "csi_clk", "isp_clk", "dphy_clk"
};

static int am62x_csi_probe(struct platform_device *pdev)
{
    struct csi_priv *priv;
    int ret, i;

    priv->num_clks = ARRAY_SIZE(am62x_csi_clk_ids);
    priv->clks = devm_kcalloc(&pdev->dev, priv->num_clks,
                               sizeof(*priv->clks), GFP_KERNEL);

    /* ملء الـ IDs */
    for (i = 0; i < priv->num_clks; i++)
        priv->clks[i].id = am62x_csi_clk_ids[i];

    /* احصل على كل الـ clocks دفعة واحدة */
    ret = devm_clk_bulk_get(&pdev->dev, priv->num_clks, priv->clks);
    if (ret) {
        dev_err(&pdev->dev, "فشل الحصول على CSI clocks: %d\n", ret);
        return ret;
    }

    return 0;
}

static int am62x_csi_start_streaming(struct csi_priv *priv)
{
    int ret;

    /* ← الإصلاح: تحقق دائماً من القيمة المُرجعة */
    ret = clk_bulk_prepare_enable(priv->num_clks, priv->clks);
    if (ret) {
        dev_err(priv->dev, "فشل تفعيل CSI clocks: %d\n", ret);
        return ret;
    }

    ret = start_dma_transfer(priv);
    if (ret) {
        clk_bulk_disable_unprepare(priv->num_clks, priv->clks);
        return ret;
    }

    return 0;
}

static void am62x_csi_stop_streaming(struct csi_priv *priv)
{
    stop_dma_transfer(priv);
    /* clk_bulk_disable_unprepare عكس clk_bulk_prepare_enable */
    clk_bulk_disable_unprepare(priv->num_clks, priv->clks);
}
```

للتشخيص:
```bash
# تحقق من حالة كل clock على حدة
for clk in csi_clk isp_clk dphy_clk; do
    echo "=== $clk ==="
    cat /sys/kernel/debug/clk/$clk/clk_enable_count
    cat /sys/kernel/debug/clk/$clk/clk_rate
done
```

#### الدرس المستفاد
`__must_check` موجودة لسبب وجيه — لا تتجاهلها أبداً. `clk_bulk_prepare_enable()` تُسهّل الإدارة لأنها تعمل rollback تلقائي عند الفشل. دائماً استخدم `devm_clk_bulk_get()` بدلاً من `clk_bulk_get()` لتجنب memory leaks عند إزالة الـ driver.

---

### السيناريو الخامس: I2C بطيء جداً على ECU السيارة بـ i.MX8

#### العنوان
**الـ I2C يعمل بـ 100 KHz بدل 400 KHz على ECU يعمل بـ NXP i.MX8**

#### السياق
شركة تصنع **automotive ECU** بمعالج **NXP i.MX8** يتواصل مع عدة chips عبر **I2C** بسرعة Fast-Mode (400 KHz). في بيئة التطوير كل شيء كان يشتغل، لكن في النماذج الأولية الإنتاجية، الـ I2C يعمل بـ 100 KHz فقط. هذا يسبب تأخيراً في قراءة بيانات الـ sensors مما يؤثر على زمن استجابة نظام السلامة.

#### المشكلة
الـ Device Tree للنموذج الإنتاجي يستخدم مصدر clock مختلف لـ I2C. الـ driver يطلب تردداً محدداً لكن `clk_round_rate()` تُرجع قيمة مختلفة عما يتوقعه، والـ driver لا يتحقق من الفرق.

#### التحليل

```c
/* من clk.h */
long clk_round_rate(struct clk *clk, unsigned long rate);
/*
 * تُجيب على سؤال: "لو طلبت @rate، ما التردد الفعلي الذي سأحصل عليه؟"
 * بدون أي تغيير في الـ hardware
 */

int clk_set_rate(struct clk *clk, unsigned long rate);
/* تضبط التردد فعلياً */

unsigned long clk_get_rate(struct clk *clk);
/* تقرأ التردد الحالي */
```

السيناريو خطوة بخطوة:

```
/* في Device Tree للنموذج الإنتاجي */
&i2c1 {
    clocks = <&clk_root_i2c>;  /* مصدر مختلف عن بيئة التطوير */
    clock-frequency = <400000>;
};

/* الـ driver يطلب 400 KHz */
clk_set_rate(i2c_clk, 400000);

/* لكن مصدر الـ clock في النموذج الإنتاجي له divisors مختلفة */
/* clk_round_rate() سيُرجع 100000 (100 KHz) لأن أقرب قيمة ممكنة هي هذه */

/* النتيجة: clk_set_rate تنجح (تضبط أقرب قيمة ممكنة = 100 KHz) */
/* الـ driver لا يلاحظ الفرق! */
```

الكود الخاطئ:
```c
static int imx8_i2c_set_clock(struct i2c_priv *priv, unsigned long target_rate)
{
    /* ← BUG: لا تحقق من القيمة الفعلية */
    return clk_set_rate(priv->i2c_clk, target_rate);
}
```

#### الحل

```c
static int imx8_i2c_set_clock(struct i2c_priv *priv, unsigned long target_rate)
{
    long actual_rate;
    int ret;

    /* أولاً: اسأل عن التردد الفعلي الذي سنحصل عليه */
    actual_rate = clk_round_rate(priv->i2c_clk, target_rate);
    if (actual_rate < 0) {
        dev_err(priv->dev, "clk_round_rate فشل: %ld\n", actual_rate);
        return actual_rate;
    }

    /* تحقق من الفرق — قبول هامش 5% كحد أقصى */
    if (abs((long)target_rate - actual_rate) > (target_rate / 20)) {
        dev_warn(priv->dev,
            "I2C: طلبت %lu Hz لكن الممكن فقط %ld Hz — تحقق من الـ DT\n",
            target_rate, actual_rate);
        /*
         * في السيارات، هذا خطأ حرج وليس تحذيراً فقط
         * لأن البروتوكول يتوقع timing محدداً
         */
        return -EINVAL;
    }

    /* الآن اضبط التردد */
    ret = clk_set_rate(priv->i2c_clk, target_rate);
    if (ret) {
        dev_err(priv->dev, "clk_set_rate فشل: %d\n", ret);
        return ret;
    }

    /* تحقق نهائي من التردد الفعلي */
    actual_rate = clk_get_rate(priv->i2c_clk);
    dev_info(priv->dev, "I2C clock: طلبت %lu Hz، حصلت على %ld Hz\n",
             target_rate, actual_rate);

    return 0;
}

/* وفي probe() — تحقق من مصدر الـ clock */
static int imx8_i2c_probe(struct platform_device *pdev)
{
    struct i2c_priv *priv;
    struct clk *parent;
    unsigned long parent_rate;

    priv->i2c_clk = devm_clk_get(&pdev->dev, "per");
    if (IS_ERR(priv->i2c_clk))
        return PTR_ERR(priv->i2c_clk);

    /* تحقق من الـ parent clock لفهم القيود */
    parent = clk_get_parent(priv->i2c_clk);
    if (parent) {
        parent_rate = clk_get_rate(parent);
        dev_info(&pdev->dev, "I2C parent clock: %lu Hz\n", parent_rate);

        /* تحقق أن الـ parent يدعم 400KHz I2C */
        if (parent_rate < 8000000UL) { /* يجب أن يكون 8x على الأقل */
            dev_err(&pdev->dev,
                "I2C parent clock منخفض جداً (%lu Hz) لـ 400KHz I2C\n",
                parent_rate);
            return -EINVAL;
        }
    }

    return 0;
}
```

لتشخيص المشكلة في الميدان:
```bash
# قارن التردد المطلوب والفعلي
cat /sys/kernel/debug/clk/imx8mp_i2c1_root_clk/clk_rate

# تحقق من شجرة الـ clocks وأين المشكلة
cat /sys/kernel/debug/clk/clk_summary | grep -A5 i2c

# استخدم i2cdetect للتحقق من السرعة الفعلية
i2cdetect -F 1  # يُظهر الـ features المدعومة

# قياس timing فعلي بـ oscilloscope logic analyzer
# أو باستخدام:
i2ctransfer -y 1 w1@0x50 0x00 r1
# وقس الوقت بين SCL pulses
```

#### الدرس المستفاد
`clk_round_rate()` هي أداة تشخيص قيّمة يجب استخدامها **قبل** `clk_set_rate()` للتحقق مما ستحصل عليه فعلاً. في تطبيقات الـ automotive والـ safety-critical، الفرق بين التردد المطلوب والفعلي يجب أن يكون خطأ حرجاً وليس مجرد تحذير. أيضاً، `clk_get_parent()` مفيدة في مرحلة الـ bring-up لفهم شجرة الـ clocks والتحقق من مصادرها في الـ DT.
## Phase 7: مصادر ومراجع

---

### مقدمة سريعة

كل اللي اتعلمناه عن الـ Common Clock Framework وملف `include/linux/clk.h` مبني على مصادر حقيقية من داخل مجتمع الـ Linux kernel. هنا بنجمع أهم المصادر اللي تقدر ترجع إليها لتعمق أكثر.

---

### مقالات LWN.net الأساسية

**الـ LWN.net** هو أهم مصدر صحفي وتقني للـ Linux kernel — كل تغيير كبير فيه مقال يشرحه.

| المقال | الوصف |
|--------|--------|
| [A common clock framework](https://lwn.net/Articles/472998/) | المقال الأصلي اللي شرح فكرة الـ CCF لأول مرة — يشرح ليش احتاجوا إطار موحد بدل كل سوب-سيستم يعمل كلوك-فريمووورك خاص بيه |
| [common clk framework](https://lwn.net/Articles/486841/) | مقال تقني أعمق يشرح `struct clk` الموحدة والـ API اللي بتجمع عمليات مختلفة من منصات مختلفة |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | أرشيف LWN لملف التوثيق الرسمي اللي كتبه Mike Turquette — شرح الـ API للمبرمجين |
| [Add a generic struct clk](https://lwn.net/Articles/460193/) | نقاش مبكر في الميلينغ ليست عن ضرورة توحيد `struct clk` عبر كل المنصات |
| [clk: add LP8788 clock driver](https://lwn.net/Articles/545908/) | مثال عملي على كتابة كلوك درايفر بسيط باستخدام الـ CCF |
| [Renesas R-Car Gen2 Common Clock Framework support](https://lwn.net/Articles/574312/) | مثال على تبني الـ CCF في معالج متكامل حقيقي |
| [Create common DPLL/clock configuration API](https://lwn.net/Articles/920248/) | مقال حديث عن تطوير الـ CCF ليشمل الـ DPLL |

---

### التوثيق الرسمي للـ Kernel

**الـ kernel.org docs** هي المرجع الرسمي — مكتوبة من قِبل المطورين أنفسهم.

#### الصفحة الرئيسية للـ CCF:

```
https://docs.kernel.org/driver-api/clk.html
```

هذه هي نسخة الـ RST المحولة من الملف `Documentation/driver-api/clk.rst` — تشرح:
- الفلسفة خلف الـ `prepare/enable` split
- كيفية كتابة `clk_ops`
- تفاصيل الـ clock tree

#### نسخة أرشيفية قديمة (مفيدة لفهم التطور):

```
https://www.kernel.org/doc/Documentation/clk.txt
```

#### ملف المصدر المباشر على GitHub:

```
https://github.com/torvalds/linux/blob/master/include/linux/clk.h
```

هنا تشوف الكود الحالي بالضبط مع تاريخ كل تغيير.

#### توثيق KUnit API للـ clock testing:

```
https://docs.kernel.org/dev-tools/kunit/api/clk.html
```

---

### ملفات المصدر الأساسية في الـ Kernel

لو عندك نسخة من الكيرنل، هذه هي أهم الملفات اللي تقرأها بالترتيب:

```
include/linux/clk.h          ← واجهة المستخدم (Consumer API)
include/linux/clk-provider.h ← واجهة صانع الكلوك (Provider API)
drivers/clk/clk.c            ← التنفيذ الأساسي للـ CCF
drivers/clk/clk-fixed-rate.c ← مثال بسيط على كلوك ثابت
drivers/clk/clk-divider.c    ← مثال على clock divider
drivers/clk/clk-mux.c        ← مثال على clock mux
Documentation/driver-api/clk.rst ← التوثيق الرسمي
```

---

### Commits مهمة في تاريخ الـ CCF

هذه هي اللحظات المفصلية في تاريخ الـ Common Clock Framework:

#### الـ Commit الأصلي (Linux 3.4، مايو 2012):

المؤلف: **Mike Turquette** من Linaro — يمكنك تتبعه بالبحث في:

```bash
git log --oneline --all | grep -i "common clock"
git log --oneline v3.3..v3.4 -- drivers/clk/
```

أو مباشرة عبر:
```
https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/clk
```

#### أهم patch series على مرحلة التقديم:

- [GIT PULL Integrator common clock - Mike Turquette](https://lore.kernel.org/linux-arm-kernel/20120707001310.GA1230@gmail.com/)

#### مثال على patches التحويل من منصات قديمة:

- [PATCH v9: ARM davinci: convert to common clock framework](https://lore.kernel.org/lkml/20180427001745.4116-7-david@lechnology.com/t/)
- [PATCH: MIPS TXx9 Convert to Common Clock Framework](https://lore.kernel.org/all/1471541667-30689-4-git-send-email-geert@linux-m68k.org/)
- [PATCH v6: clk exynos4/5 migrate to common clock framework](https://lore.kernel.org/all/5133468C.4010001@gmail.com/T/)

---

### نقاشات Mailing List المهمة

**الـ lore.kernel.org** هو الأرشيف الرسمي لكل الميلينغ ليست.

#### للبحث في نقاشات الـ CCF:

```
https://lore.kernel.org/linux-clk/
```

هذا هو الميلينغ ليست المخصص لكل شيء متعلق بالـ clock subsystem.

#### patch مهمة تاريخياً:

```
https://lists.infradead.org/pipermail/linux-arm-kernel/2011-November/072767.html
```

هذا الـ patch أضاف `clk_prepare_enable()` و `clk_disable_unprepare()` كـ helper functions — اللي بتستخدمهم كثيراً في الكود اليومي.

---

### موارد eLinux.org

**الـ eLinux.org** هو ويكي مجتمع الـ Embedded Linux — فيه مواد عملية ومحاضرات:

| المصدر | الوصف |
|--------|--------|
| [Generic Linux CLK Framework - CELF 2009](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | عرض تقديمي من مؤتمر CELF 2009 — يشرح الأفكار الأولى للإطار الموحد |
| [Linux clock management framework - ELC 2007](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | محاضرة قديمة تشرح إدارة الكلوكات قبل الـ CCF — مفيدة لفهم "ليش احتجنا CCF" |
| [Common Clock Framework BoFs](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | تلخيص جلسة Birds of a Feather عن الـ CCF في مؤتمر |
| [High Resolution Timers](https://elinux.org/High_Resolution_Timers) | مرتبط بالكلوكات — يشرح الـ hrtimer framework |
| [Kernel Timer Systems](https://elinux.org/index.php/Kernel_Timer_Systems) | نظرة عامة على أنظمة الـ timing في الكيرنل |

---

### موارد Bootlin (مواد مؤتمرات ممتازة)

**Bootlin** (سابقاً Free Electrons) تنشر كل مواد مؤتمراتها مجاناً:

```
https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/
```

```
https://bootlin.com/pub/conferences/2013/elc/common-clock-framework-how-to-use-it/
```

هذه المواد من 2013 تشرح الـ CCF بطريقة عملية لكتابة الـ drivers.

---

### كتب مرجعية مهمة

#### Linux Device Drivers, 3rd Edition (LDD3)

**الكتاب المقدس** لكتابة كيرنل درايفرز — متاح مجاناً:

```
https://lwn.net/Kernel/LDD3/
```

- **الفصل المرتبط**: Chapter 1 (Device Driver Concepts) و Chapter 14 (The Linux Device Model)
- الكتاب قديم شوية (2005) لكن الأساسيات لا تتغير
- الـ CCF ما كان موجوداً وقت كتابته لكن مفاهيم الـ device model أساسية لفهمه

#### Linux Kernel Development, 3rd Edition — Robert Love

هذا الكتاب أحدث وأشمل لفهم الكيرنل بشكل عام:
- **الفصل المرتبط**: Chapter 17 (Devices and Modules)
- يشرح كيف تتكامل الـ subsystems مع بعضها
- مفيد لفهم ليش الـ CCF مصمم بهالطريقة

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan

للمطور اللي يشتغل على embedded systems:
- **الفصل المرتبط**: Chapter 15 (Kernel Initialization) و Chapter 16 (Kernel Debugging Techniques)
- يشرح كيف تُهيَّأ الـ clocks أثناء boot في الأنظمة المدمجة
- مفيد جداً لفهم الـ Device Tree وعلاقته بالكلوكات

---

### مصادر إضافية متخصصة

#### وثيقة Device Tree للكلوكات:

```
https://www.kernel.org/doc/Documentation/devicetree/bindings/clock/
```

هذا المجلد فيه كل الـ bindings الخاصة بالكلوكات في الـ Device Tree.

#### Infradead mirror للتوثيق (نسخة تاريخية مفيدة):

```
https://www.infradead.org/~mchehab/rst_conversion/driver-api/clk.html
```

---

### مصطلحات البحث الفعّالة

لو حابب تبحث عن معلومات أكثر، استخدم هذه الـ search terms:

#### للبحث العام:

```
linux kernel common clock framework CCF
linux clk_prepare clk_enable difference
linux clock consumer provider API
linux clk_ops implementation
linux clock tree debugfs
```

#### للبحث في الكود:

```bash
# البحث في الكيرنل مباشرة
grep -r "clk_prepare" drivers/ --include="*.c" | head -20
grep -r "clk_ops" include/ --include="*.h"
find drivers/clk -name "*.c" | xargs grep "clk_register"
```

#### للبحث في الـ mailing list:

```
site:lore.kernel.org "common clock framework"
site:lore.kernel.org linux-clk clk_prepare
site:lwn.net "clock framework" linux kernel
```

#### للبحث في patches:

```
site:lore.kernel.org "[PATCH] clk:"
site:lore.kernel.org linux-clk "clk_bulk"
```

---

### جدول ملخص المصادر

| النوع | المصدر | الأولوية |
|-------|--------|----------|
| توثيق رسمي | [docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html) | عالية جداً |
| مقال تأسيسي | [LWN: A common clock framework](https://lwn.net/Articles/472998/) | عالية جداً |
| مقال تقني | [LWN: common clk framework](https://lwn.net/Articles/486841/) | عالية |
| توثيق أصلي | [LWN: Documentation/clk.txt](https://lwn.net/Articles/489668/) | عالية |
| كود مصدري | [GitHub: include/linux/clk.h](https://github.com/torvalds/linux/blob/master/include/linux/clk.h) | عالية |
| مؤتمر | [Bootlin ELCE 2013](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/) | متوسطة |
| ميلينغ ليست | [lore.kernel.org/linux-clk](https://lore.kernel.org/linux-clk/) | متوسطة |
| كتاب | LDD3 - Linux Device Drivers 3rd Ed. | أساسية |
| كتاب | Linux Kernel Development - Robert Love | أساسية |
| embedded | Embedded Linux Primer - Hallinan | للـ embedded |
| eLinux PDF | [CCF CELF 2009](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | تاريخية |
| eLinux BoFs | [Common Clock Framework BoFs](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | تاريخية |
## Phase 8: Writing simple module

### الفكرة العامة

تخيّل إن الـ kernel عنده "راديو داخلي" — كل مرة أي جهاز يطلب تغيير تردد ساعته (clock rate)، الـ kernel ينادي كل من "اشترك" في هذا الراديو. هذا بالضبط ما يفعله **`clk_notifier`**: هو نظام اشتراكات يخبرك قبل وبعد كل تغيير في سرعة الـ clock.

سنكتب module يشترك في أحداث تغيير الـ rate لأول clock موجود في النظام ويطبع المعلومات في كل مرة يتغير فيها الـ rate.

---

### الـ Hook المختار: `clk_notifier_register` / `clk_notifier_unregister`

اخترنا **نظام الـ clk_notifier** لأنه:
- مُصدَّر (exported) ومتاح لأي module
- يعطينا بيانات ثرية: الاسم القديم والجديد للـ rate
- آمن تماماً — لا يعدّل سلوك النظام، فقط يراقب

الـ events الثلاثة التي سنراقبها:

| Event | متى يُطلَق |
|---|---|
| `PRE_RATE_CHANGE` | قبل التغيير مباشرة |
| `POST_RATE_CHANGE` | بعد نجاح التغيير |
| `ABORT_RATE_CHANGE` | لو فشل التغيير |

---

### الكود الكامل للـ Module

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * clk_rate_watcher.c
 *
 * module يراقب تغييرات الـ clock rate عبر نظام الـ clk_notifier
 * ويطبع المعلومات في كل حدث تغيير.
 */

/* ---- الـ includes ---- */
#include <linux/module.h>      /* module_init, module_exit, MODULE_* macros */
#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/clk.h>         /* struct clk, clk_notifier_register, clk_notifier_data */
#include <linux/notifier.h>    /* struct notifier_block, NOTIFY_OK, NOTIFY_DONE */
#include <linux/clk-provider.h>/* clk_hw_get_name — للحصول على اسم الـ clock */
#include <linux/of.h>          /* of_find_node_by_path */
#include <linux/of_clk.h>      /* of_clk_get */
#include <linux/err.h>         /* IS_ERR */

/* ---- الـ module metadata ---- */
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner <learner@example.com>");
MODULE_DESCRIPTION("يراقب تغييرات clock rate عبر clk_notifier ويطبعها في dmesg");
MODULE_VERSION("1.0");

/* ---- متغيرات عامة ---- */

/*
 * مؤشر للـ clock الذي سنراقبه.
 * نحصل عليه في module_init ونحرره في module_exit.
 */
static struct clk *watched_clk;

/*
 * الـ notifier_block هو "بطاقة الاشتراك" —
 * بها مؤشر لدالة الـ callback التي ستُنادى عند كل حدث.
 */
static struct notifier_block clk_rate_nb;

/* ---- دالة الـ Callback ---- */

/**
 * clk_rate_notifier_cb - تُستدعى من الـ kernel عند كل حدث تغيير في الـ rate
 * @nb:    مؤشر لنفس الـ notifier_block الذي سجّلناه
 * @event: نوع الحدث: PRE_RATE_CHANGE / POST_RATE_CHANGE / ABORT_RATE_CHANGE
 * @data:  مؤشر لـ struct clk_notifier_data يحمل old_rate و new_rate
 *
 * يجب أن تُعيد NOTIFY_OK أو NOTIFY_DONE في كل الحالات.
 * إعادة NOTIFY_BAD في PRE_RATE_CHANGE تعني "أنا أرفض هذا التغيير".
 */
static int clk_rate_notifier_cb(struct notifier_block *nb,
                                 unsigned long event,
                                 void *data)
{
    /*
     * نحوّل الـ void* إلى النوع الصحيح struct clk_notifier_data
     * الذي يحمل الـ old_rate والـ new_rate والمؤشر للـ clock نفسه.
     */
    struct clk_notifier_data *ndata = (struct clk_notifier_data *)data;

    /* نختار نص الحدث لطباعة مقروءة */
    const char *event_name;

    switch (event) {
    case PRE_RATE_CHANGE:
        /*
         * هذا الحدث يأتي قبل التغيير — الـ old_rate هو الحالي،
         * والـ new_rate هو المطلوب. لو أعدنا NOTIFY_BAD هنا
         * يتوقف التغيير (نحن هنا مراقبون فقط فنعيد NOTIFY_OK).
         */
        event_name = "PRE_RATE_CHANGE";
        pr_info("[clk_watcher] %s: clock سيتغير من %lu Hz إلى %lu Hz\n",
                event_name,
                ndata->old_rate,
                ndata->new_rate);
        break;

    case POST_RATE_CHANGE:
        /*
         * التغيير نجح — في هذه المرحلة old_rate وnew_rate
         * كلاهما يساوي الـ rate الجديد (تفصيل تنفيذي في الـ kernel).
         * نطبع old_rate من الـ struct لأنه يحمل القيمة السابقة.
         */
        event_name = "POST_RATE_CHANGE";
        pr_info("[clk_watcher] %s: clock تغيّر بنجاح — rate الجديد: %lu Hz\n",
                event_name,
                ndata->new_rate);
        break;

    case ABORT_RATE_CHANGE:
        /*
         * فشل التغيير بعد إرسال PRE_RATE_CHANGE —
         * يُبلَّغ كل المشتركين بـ ABORT لإلغاء أي تجهيزات فعلوها.
         */
        event_name = "ABORT_RATE_CHANGE";
        pr_warn("[clk_watcher] %s: فشل تغيير الـ clock! العودة إلى %lu Hz\n",
                event_name,
                ndata->old_rate);
        break;

    default:
        pr_warn("[clk_watcher] حدث غير معروف: %lu\n", event);
        break;
    }

    /* NOTIFY_OK = "استلمت وموافق، تابع إشعار البقية" */
    return NOTIFY_OK;
}

/* ---- module_init ---- */

/**
 * clk_watcher_init - نقطة دخول الـ module
 *
 * نحصل هنا على مرجع للـ clock الأول من Device Tree،
 * ثم نسجّل الـ notifier عليه.
 */
static int __init clk_watcher_init(void)
{
    int ret;
    struct device_node *np;

    pr_info("[clk_watcher] بدء تحميل الـ module\n");

    /*
     * نبحث عن أول node من نوع "clock" في الـ Device Tree.
     * هذا النهج يعمل على معظم البيئات المضمّنة (embedded).
     * في بيئة x86 عادية، قد لا يوجد DT clock وسيفشل —
     * في هذه الحالة يمكن استخدام clk_get(NULL, "specific-clk-name").
     */
    np = of_find_node_by_type(NULL, "clock");
    if (!np) {
        /*
         * لو ما فيه DT، نحاول clk_get_sys كـ fallback.
         * هنا نطبع تحذير ونخرج بأمان.
         */
        pr_warn("[clk_watcher] لم يُعثر على clock node في Device Tree\n");
        pr_warn("[clk_watcher] يرجى تعديل init لاستخدام clk_get() مع اسم محدد\n");
        return -ENODEV;
    }

    /*
     * of_clk_get: نأخذ المؤشر الفعلي للـ clock من الـ DT node.
     * index=0 يعني أول clock مُعرَّف في هذا الـ node.
     */
    watched_clk = of_clk_get(np, 0);
    of_node_put(np); /* نُحرر الـ node بعد الانتهاء منه */

    if (IS_ERR(watched_clk)) {
        pr_err("[clk_watcher] فشل الحصول على الـ clock: %ld\n",
               PTR_ERR(watched_clk));
        return PTR_ERR(watched_clk);
    }

    pr_info("[clk_watcher] تم الحصول على الـ clock — rate الحالي: %lu Hz\n",
            clk_get_rate(watched_clk));

    /*
     * نُعبئ الـ notifier_block:
     * .notifier_call = دالة الـ callback
     * .priority = 0 (أولوية عادية، كلما كانت أعلى كلما نودي أولاً)
     */
    clk_rate_nb.notifier_call = clk_rate_notifier_cb;
    clk_rate_nb.priority      = 0;

    /*
     * clk_notifier_register: نُسجّل اشتراكنا —
     * من الآن فصاعداً كل تغيير في rate هذا الـ clock
     * سيُنادي clk_rate_notifier_cb تلقائياً.
     */
    ret = clk_notifier_register(watched_clk, &clk_rate_nb);
    if (ret) {
        pr_err("[clk_watcher] فشل تسجيل الـ notifier: %d\n", ret);
        clk_put(watched_clk); /* نُحرر الـ clock لو فشلنا */
        return ret;
    }

    pr_info("[clk_watcher] الـ notifier مُسجَّل بنجاح — الآن نراقب تغييرات الـ rate\n");
    return 0;
}

/* ---- module_exit ---- */

/**
 * clk_watcher_exit - تنظيف عند إزالة الـ module
 *
 * قاعدة ذهبية: كل ما سجّلناه يجب أن نُلغي تسجيله،
 * وكل ما حصلنا عليه يجب أن نُحرره — وإلا memory leak.
 */
static void __exit clk_watcher_exit(void)
{
    /*
     * clk_notifier_unregister: نفصل الـ callback عن الـ clock.
     * لو نسينا هذا وأُزيل الـ module، سيحاول الـ kernel استدعاء
     * كود غير موجود → kernel panic مضمون.
     */
    clk_notifier_unregister(watched_clk, &clk_rate_nb);
    pr_info("[clk_watcher] تم إلغاء تسجيل الـ notifier\n");

    /*
     * clk_put: نُحرر المرجع الذي أخذناه بـ of_clk_get.
     * الـ kernel يحتفظ بعداد مراجع (reference count) للـ clocks —
     * clk_put يُنقص العداد، ولو وصل صفر يُحرر الموارد.
     */
    clk_put(watched_clk);
    pr_info("[clk_watcher] تم تحرير الـ clock — الـ module خرج نظيفاً\n");
}

module_init(clk_watcher_init);
module_exit(clk_watcher_exit);
```

---

### شرح كل قسم

#### الـ includes

```c
#include <linux/clk.h>
#include <linux/notifier.h>
#include <linux/clk-provider.h>
```

الـ `clk.h` يُعرّف `struct clk_notifier_data` والـ API كاملاً، والـ `notifier.h` يُعرّف `struct notifier_block` ومعرّفات الـ `NOTIFY_*`. بدونها الـ compiler لن يعرف أي شيء عن نظام الـ clock أو الـ notifiers.

---

#### الـ `struct clk_notifier_data`

```c
struct clk_notifier_data {
    struct clk    *clk;       /* المؤشر للـ clock */
    unsigned long  old_rate;  /* الـ rate قبل التغيير */
    unsigned long  new_rate;  /* الـ rate بعد التغيير */
};
```

هذا الـ struct هو "الرسالة" التي يُرسلها الـ kernel لكل مشترك — مثل SMS يحمل "كنت على قناة X، والآن انتقلت إلى قناة Y". نحن نقرأ منها فقط ولا نعدّلها.

---

#### الـ `notifier_block`

```c
clk_rate_nb.notifier_call = clk_rate_notifier_cb;
clk_rate_nb.priority      = 0;
```

الـ `notifier_block` هو "بطاقة عضويتنا" في نظام الإشعارات. الـ `.priority` يحدد الترتيب — من أعلى أولوية إلى أدنى. اخترنا `0` (متوسط) لأننا مراقبون فقط ولا نريد التدخل في الترتيب.

---

#### لماذا `clk_get_rate` في الـ init؟

```c
pr_info("[clk_watcher] rate الحالي: %lu Hz\n", clk_get_rate(watched_clk));
```

لنطبع نقطة البداية — هكذا عند قراءة الـ `dmesg` نعرف من أين بدأ الـ clock قبل أي تغيير.

---

#### لماذا `NOTIFY_OK` وليس `NOTIFY_DONE`؟

- **`NOTIFY_OK`** = "استلمت الرسالة، وأريد الـ kernel أن يُكمل إخبار باقي المشتركين"
- **`NOTIFY_DONE`** = "استلمت، لكن توقف عن إخبار البقية" (يوقف السلسلة)

نحن مراقبون فقط فنستخدم `NOTIFY_OK` لضمان عدم تعطيل باقي المشتركين.

---

#### لماذا `clk_notifier_unregister` في الـ exit؟

```c
clk_notifier_unregister(watched_clk, &clk_rate_nb);
clk_put(watched_clk);
```

تخيّل إنك اشتركت في خدمة بريد، ثم انتقلت من المنزل دون أن تُلغي الاشتراك — ستصل الرسائل إلى عنوان لا أحد فيه. بالمثل: لو أُزيل الـ module دون إلغاء الـ notifier، سيُنادي الـ kernel عنواناً في الذاكرة لم يعد موجوداً → **kernel panic**. الـ `clk_put` بدوره يُعيد المرجع لمنع تسريب الموارد.

---

### كيف تبني وتختبر

#### الـ Makefile

```makefile
obj-m += clk_rate_watcher.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

#### البناء والتحميل

```bash
# بناء الـ module
make

# تحميله (يحتاج kernel مع CONFIG_COMMON_CLK=y وجهاز فيه DT clocks)
sudo insmod clk_rate_watcher.ko

# مراقبة الـ output في real-time
sudo dmesg -wH | grep clk_watcher

# إزالة الـ module بشكل نظيف
sudo rmmod clk_rate_watcher
```

#### مثال لـ output متوقع

```
[clk_watcher] بدء تحميل الـ module
[clk_watcher] تم الحصول على الـ clock — rate الحالي: 1000000000 Hz
[clk_watcher] الـ notifier مُسجَّل بنجاح — الآن نراقب تغييرات الـ rate
...
[clk_watcher] PRE_RATE_CHANGE: clock سيتغير من 1000000000 Hz إلى 800000000 Hz
[clk_watcher] POST_RATE_CHANGE: clock تغيّر بنجاح — rate الجديد: 800000000 Hz
```

---

### ملاحظة على بيئة x86

على x86 بدون Device Tree، غيّر دالة الـ init لاستخدام:

```c
/* استبدل "your-clock-name" باسم clock حقيقي من /sys/kernel/debug/clk/clk_summary */
watched_clk = clk_get(NULL, "your-clock-name");
if (IS_ERR(watched_clk)) { ... }
```

على أنظمة ARM/RISC-V المضمّنة (Raspberry Pi، BeagleBone، إلخ) كود الـ `of_clk_get` يعمل مباشرة.
