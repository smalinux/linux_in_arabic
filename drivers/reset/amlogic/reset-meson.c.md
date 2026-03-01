## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **ARM/Amlogic Meson SoC support** subsystem في الـ Linux kernel — وبالتحديد من **Reset Controller framework** الخاص بـ SoCs شركة Amlogic (شركة صينية بتعمل chips للـ TV boxes والـ embedded devices).

---

### ليه الـ Reset Controller موجود أصلاً؟

تخيل إن عندك جهاز كمبيوتر وفيه USB controller اتعلق — إيه اللي بتعمله؟ بتعمله restart طبعاً. نفس الفكرة بالظبط في الـ SoC (System on Chip).

الـ **SoC** زي Amlogic Meson فيه عشرات أو مئات الـ **IP blocks** (وحدات) جوا الـ chip الواحدة: GPU، USB، HDMI، Audio، Ethernet، وغيرها كتير. كل واحدة من دول ممكن تتعلق أو تحتاج reset عشان تشتغل صح.

الـ **Reset Controller** هو الجهاز اللي بيتحكم في كل الأزرار دي — بيعرف: "reset وحدة رقم X" يعني امسح بت معين في register معين في الـ hardware.

---

### القصة من الأول

```
+------------------+         +-------------------+         +------------------+
|  Driver محتاج   |  يطلب   |  Reset Framework  |  يستخدم |  Reset Controller|
|  reset لـ USB   | ------> |  (kernel core)    | ------> |  (هذا الـ driver)|
+------------------+         +-------------------+         +------------------+
                                                                     |
                                                            يكتب في register
                                                            في الـ hardware
                                                                     |
                                                            +------------------+
                                                            |  Amlogic Meson   |
                                                            |  SoC Hardware    |
                                                            +------------------+
```

الـ **kernel's reset framework** بيعمل abstraction layer عام — أي driver ممكن يطلب reset لأي block بدون ما يعرف تفاصيل الـ hardware. الـ **reset-meson.c** هو اللي بيكمل الصورة: بيعرف الـ kernel "عايز تعمل reset لـ block رقم 5 في Meson? اكتب في الـ register ده، الـ bit ده".

---

### ازاي الـ Hardware بيشتغل؟

في الـ Amlogic Meson SoCs، الـ reset registers منظمة بشكل بسيط:

- عندك **مجموعة registers** في address معين في الـ memory.
- كل register فيه **32 bit**.
- كل **bit** = reset line واحدة (يعني بتتحكم في IP block واحد).
- عشان تعمل reset لـ block: تكتب `1` في الـ bit المناسب (أو `0` حسب الـ chip).
- عشان تـ "assert" (تمسك في reset): تكتب قيمة معينة وتسيبها.
- عشان تـ "deassert" (تطلق من reset): تكتب القيمة التانية.

الفرق بين **pulse reset** (لحظي) و**level reset** (مستمر):
- **Pulse**: اكتب الـ bit وارجع تاني — الـ hardware بيعمل reset لحظة ويرجع لوحده.
- **Level**: أنت اللي بتتحكم — assert (ابدأ reset) وبعدين deassert (خلص reset).

---

### هدف الـ File ده تحديداً

الـ `reset-meson.c` هو الـ **platform driver** الرئيسي — وظيفته:

1. **يتعرف** على الـ chip من الـ Device Tree (باستخدام `.compatible` strings).
2. **يقرأ** الـ parameters الخاصة بكل chip (عدد الـ resets، أوفست الـ registers).
3. **يعمل** `regmap` على منطقة الـ memory الخاصة بالـ reset registers.
4. **يسجل** الـ reset controller مع الـ kernel.

الـ file بنفسه خفيف — الـ logic الحقيقية موجودة في `reset-meson-common.c`.

---

### الـ SoC Variants المدعومة

| Compatible String | الـ Param | عدد الـ Resets |
|---|---|---|
| `amlogic,meson8b-reset` | `meson8b_param` | 256 |
| `amlogic,meson-gxbb-reset` | `meson8b_param` | 256 |
| `amlogic,meson-axg-reset` | `meson8b_param` | 256 |
| `amlogic,meson-a1-reset` | `meson_a1_param` | 96 |
| `amlogic,meson-s4-reset` | `meson_s4_param` | 192 |
| `amlogic,c3-reset` | `meson_s4_param` | 192 |
| `amlogic,t7-reset` | `t7_param` | 224 |

---

### الفرق بين الـ Chips

الـ `level_low_reset = true` معناها إن الـ reset يحصل لما الـ bit يبقى `0` مش `1` — يعني **active low**. الـ driver بيتعامل مع الفرق ده في `reset-meson-common.c` بـ XOR بسيط.

الـ `t7_param` ملهاش `reset_ops` صريحة — يعني بتستخدم `NULL` اللي الـ `meson_reset_controller_register` هيتعامل معاه بشكل مختلف (محتملاً بيستخدم `meson_reset_toggle_ops`).

---

### الـ Files المكونة للـ Subsystem ده

```
drivers/reset/amlogic/
├── reset-meson.c          ← الـ file ده: platform driver + chip params
├── reset-meson-common.c   ← الـ core logic: reset/assert/deassert/status
├── reset-meson.h          ← الـ shared header: struct meson_reset_param
├── reset-meson-aux.c      ← auxiliary bus variant للـ chips الجديدة
├── reset-meson-audio-arb.c← reset controller خاص بـ Audio Memory Arbiter
├── Kconfig                ← build config
└── Makefile               ← build rules

include/linux/
└── reset-controller.h     ← الـ kernel framework API: reset_control_ops، reset_controller_dev
```

### الـ Files المرتبطة اللي المفروض تعرفها

| الـ File | الأهمية |
|---|---|
| `include/linux/reset-controller.h` | الـ API الأساسي للـ Reset Controller framework |
| `include/linux/reset.h` | الـ consumer API (الـ drivers اللي بتطلب reset) |
| `include/linux/regmap.h` | الـ abstraction للـ register access |
| `drivers/reset/core.c` | الـ core implementation للـ reset framework |
| `arch/arm64/boot/dts/amlogic/` | الـ Device Tree files اللي بتعرف الـ reset controller |
| `Documentation/devicetree/bindings/reset/amlogic,meson-reset.yaml` | الـ DT binding spec |

---

### الـ Subsystem في الـ MAINTAINERS

الـ file ده تابع لـ **ARM/Amlogic Meson SoC support** subsystem اللي بيتكلم فيه Neil Armstrong و Kevin Hilman (BayLibre) — وده مش موجود صراحةً في الـ MAINTAINERS تحت entry منفصل للـ reset، بس يندرج تحت الـ SoC support العام والـ Reset Controller framework العام.
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ Framework ده موجود؟

في أي SoC زي Amlogic Meson، في عشرات الـ IP blocks — USB, HDMI, Ethernet, Audio DSP, إلخ. كل block عنده **reset line** متصلة بـ register في الـ reset controller hardware. لما تيجي تـ probe driver معين، محتاج:

1. تـ assert الـ reset (تحط الـ block في حالة reset)
2. تستنى
3. تـ deassert (ترجعه يشتغل من الأول)

**المشكلة:** لو كل driver اتكلم مع الـ hardware directly، هتحصل:
- **تعارض**: اتنين drivers يكتبوا على نفس الـ register في نفس الوقت
- **تكرار كود**: كل driver هيعمل نفس logic حساب offset وbit
- **عدم abstraction**: Driver مش المفروض يعرف عنوان الـ register الـ physical، ده hardware detail

**الـ kernel محتاج layer بيعزل consumers (الـ drivers اللي محتاجة reset) عن providers (الـ hardware reset controller).**

---

### الحل — نهج الـ Kernel

الـ kernel عنده **Reset Controller Framework** جوه `drivers/reset/` بيقدم:

| Abstraction | المعنى |
|---|---|
| **reset_controller_dev** | الـ hardware controller نفسه (provider) |
| **reset_control_ops** | الـ vtable اللي فيه assert/deassert/reset/status |
| **reset_control** | handle يمسكه الـ consumer driver |

الـ consumer بيطلب reset line بـ `devm_reset_control_get()` وبيشتغل على الـ handle من غير ما يعرف أي حاجة عن الـ hardware.

---

### الـ Analogy — Circuit Breaker Panel

تخيل لوحة كهربائية (distribution board) في عمارة:

| الـ Analogy | الـ Kernel Equivalent |
|---|---|
| لوحة الـ circuit breakers كلها | `reset_controller_dev` (الـ Amlogic reset hardware) |
| كل breaker (circuit) بالرقم | reset ID (من 0 لـ 255 مثلاً) |
| الكهربائي اللي بيحدد كل breaker بيخدم إيه | `of_xlate` — بتترجم من DT phandle لـ ID |
| الشقة اللي بتطلب تقطع التيار عنها | consumer driver بيعمل `reset_control_assert()` |
| شركة الكهرباء (العامل التنفيذي) | `reset_control_ops` — `assert`, `deassert`, `reset` |
| اللوحة مش بتعرف الأجهزة جوه الشقة | الـ framework مش بيعرف إيه الـ driver اللي بيستخدمه |
| ممنوع يتحكم في breaker غير مختص | الـ framework بيعمل locking وref-counting |

الفرق المهم: الـ `reset_controller_dev` هو اللوحة كلها، والـ ID هو رقم الـ breaker. الـ framework بيضمن إن مفيش حد يعمل race على نفس الـ breaker.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │                   Consumer Drivers                       │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
  │  │  USB DWC │  │  HDMI    │  │  Ethernet│  ...         │
  │  │  Driver  │  │  Driver  │  │  Driver  │              │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
  │       │              │              │                    │
  │       └──────────────┴──────────────┘                   │
  │                      │  reset_control_get()             │
  │                      │  reset_control_assert()          │
  │                      │  reset_control_deassert()        │
  └──────────────────────┼──────────────────────────────────┘
                         │
  ┌──────────────────────▼──────────────────────────────────┐
  │              Reset Controller Framework                  │
  │          (drivers/reset/reset.c — core layer)            │
  │                                                          │
  │   reset_controller_list (global linked list of rcdevs)  │
  │   locking, refcounting, of_xlate dispatch               │
  └──────────────────────┬──────────────────────────────────┘
                         │  calls ops->assert() / ops->deassert()
  ┌──────────────────────▼──────────────────────────────────┐
  │              Provider: Amlogic Meson Driver              │
  │         (reset-meson.c + reset-meson-common.c)           │
  │                                                          │
  │  meson_reset (struct)                                    │
  │  ┌──────────────────┐  ┌─────────────────────────────┐  │
  │  │ reset_controller │  │  meson_reset_param          │  │
  │  │ _dev (rcdev)     │  │  - reset_num                │  │
  │  │ - ops            │  │  - reset_offset             │  │
  │  │ - nr_resets      │  │  - level_offset             │  │
  │  │ - of_node        │  │  - level_low_reset          │  │
  │  └──────────────────┘  └─────────────────────────────┘  │
  │                │                                         │
  │                ▼  regmap_write / regmap_update_bits      │
  └────────────────┬────────────────────────────────────────┘
                   │
  ┌────────────────▼────────────────────────────────────────┐
  │              regmap (MMIO abstraction layer)             │
  │  يعرف: reg_bits=32, val_bits=32, reg_stride=4           │
  └────────────────┬────────────────────────────────────────┘
                   │  ioremap + raw register access
  ┌────────────────▼────────────────────────────────────────┐
  │         Amlogic Meson Reset Hardware Registers           │
  │                                                          │
  │  base + reset_offset:  0x00, 0x04, 0x08, ... (pulse)   │
  │  base + level_offset:  0x7C, 0x80, ... (assert/deassert)│
  │                                                          │
  │  كل register = 32 bit = 32 reset line                   │
  │  Reset ID 0  → reg[0], bit 0                            │
  │  Reset ID 31 → reg[0], bit 31                           │
  │  Reset ID 32 → reg[4], bit 0   (stride = 4 bytes)       │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction

**الفكرة المحورية:** كل reset controller هو مجرد provider بيعمل implement لـ `reset_control_ops`، وبيسجل نفسه عند الـ framework كـ `reset_controller_dev` بعدد معروف من الـ resets (`nr_resets`). الـ framework الـ core هو الوسيط الوحيد.

#### الـ struct المحورية: `reset_controller_dev`

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops; /* vtable: assert/deassert/reset/status */
    struct module *owner;
    struct list_head list;               /* في global list عند الـ framework */
    struct list_head reset_control_head; /* الـ consumers اللي طالبين resets */
    struct device *dev;
    struct device_node *of_node;         /* ربط بـ Device Tree */
    int of_reset_n_cells;                /* عدد cells في الـ phandle */
    int (*of_xlate)(...);                /* ترجمة DT specifier → ID */
    unsigned int nr_resets;              /* عدد الـ reset lines */
};
```

#### الـ vtable: `reset_control_ops`

```c
struct reset_control_ops {
    /* pulse reset — self-deasserting */
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);

    /* level control */
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);

    /* query state */
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

---

### الـ Meson-Specific Structs — كيف بتتوصل ببعض

```
  struct meson_reset
  ┌────────────────────────────────────────────┐
  │  param ──────────────────────────────────► struct meson_reset_param
  │                                             ┌──────────────────────┐
  │                                             │ reset_ops (vtable)   │
  │                                             │ reset_num            │
  │                                             │ reset_offset         │
  │                                             │ level_offset         │
  │                                             │ level_low_reset      │
  │                                             └──────────────────────┘
  │
  │  map ────────────────────────────────────► struct regmap
  │                                             (MMIO abstraction)
  │
  │  rcdev ──────────────────────────────────► struct reset_controller_dev
  │  (embedded — container_of() trick)          (registered with framework)
  └────────────────────────────────────────────┘
```

**الـ `container_of()` trick**: الـ framework بيمرر pointer على الـ `rcdev`، والـ driver بيرجع للـ `meson_reset` الأب بـ:

```c
struct meson_reset *data =
    container_of(rcdev, struct meson_reset, rcdev);
```

ده pattern أساسي في الـ kernel لأن الـ framework مش عارف الـ private data.

---

### حساب الـ Register Offset والـ Bit

الـ logic في `meson_reset_offset_and_bit()`:

```c
/* stride = 4 bytes (32-bit registers) */
/* BITS_PER_BYTE = 8                   */
/* stride * BITS_PER_BYTE = 32         */

*offset = (id / 32) * 4;   /* كل 32 reset → register جديد */
*bit    =  id % 32;         /* الـ bit جوه الـ register */
```

**مثال:**
- Reset ID 0   → offset=0x00, bit=0
- Reset ID 31  → offset=0x00, bit=31
- Reset ID 32  → offset=0x04, bit=0
- Reset ID 63  → offset=0x04, bit=31
- Reset ID 64  → offset=0x08, bit=0

ثم بيضاف `reset_offset` أو `level_offset` من الـ param حسب نوع العملية.

---

### نوعان من الـ Reset في Meson

#### 1. Pulse Reset (Self-deasserting)

```c
/* كتابة BIT(bit) على الـ reset register → يعمل pulse تلقائي */
regmap_write(data->map, offset + reset_offset, BIT(bit));
```

الـ hardware بيعمل pulse وبيرجع لوحده. المستخدم: `meson_reset_ops.reset`.

#### 2. Level Reset (Manual assert/deassert)

```c
/* assert: set or clear الـ bit حسب polarity */
assert ^= data->param->level_low_reset;
regmap_update_bits(data->map, offset + level_offset,
                   BIT(bit), assert ? BIT(bit) : 0);
```

**الـ `level_low_reset = true`** معناها إن الـ hardware بيكون في reset لما الـ bit يبقى 0 (active-low). الـ XOR (`^=`) بيعمل polarity inversion أوتوماتيك.

| `level_low_reset` | assert في الـ software | القيمة المكتوبة للـ hardware |
|---|---|---|
| false | true | bit = 1 |
| true  | true | bit = 0 (invert) |

---

### الـ Chip Variants — إزاي الـ Driver بيدعم SoCs مختلفة

```c
static const struct meson_reset_param meson8b_param = {
    .reset_ops      = &meson_reset_ops,
    .reset_num      = 256,        /* 256 reset line */
    .reset_offset   = 0x0,
    .level_offset   = 0x7c,       /* مختلف عن A1 */
    .level_low_reset = true,
};

static const struct meson_reset_param meson_a1_param = {
    .reset_ops      = &meson_reset_ops,
    .reset_num      = 96,         /* أقل reset lines */
    .reset_offset   = 0x0,
    .level_offset   = 0x40,       /* مختلف */
    .level_low_reset = true,
};
```

الـ `of_device_id` table بتربط كل compatible string بالـ param المناسب:

```c
{ .compatible = "amlogic,meson8b-reset",    .data = &meson8b_param },
{ .compatible = "amlogic,meson-a1-reset",   .data = &meson_a1_param },
{ .compatible = "amlogic,t7-reset",         .data = &t7_param },
```

`device_get_match_data()` بترجع الـ `.data` pointer بناءً على الـ DT compatible. **ده الـ probe-time polymorphism** — نفس الكود بيشتغل على hardware مختلف.

---

### الـ regmap Layer — ليه مش Raw ioremap؟

الـ **regmap** (بيحتاج فهم `drivers/base/regmap/`) هو abstraction فوق الـ MMIO بيقدم:

- **Caching**: ممكن يخزن قيم الـ registers لو الـ hardware يسمح
- **Locking**: thread-safe access للـ registers
- **`reg_stride = 4`**: بيعرف إن كل register offset بيزيد بـ 4 bytes (word-aligned 32-bit)
- **Portability**: نفس الـ API ممكن يشتغل مع I2C أو SPI بدل MMIO من غير ما يتغير الـ driver

```c
static const struct regmap_config regmap_config = {
    .reg_bits   = 32,   /* عرض الـ address */
    .val_bits   = 32,   /* عرض الـ data */
    .reg_stride = 4,    /* كل register بيبعد 4 bytes عن اللي قبله */
};

map = devm_regmap_init_mmio(dev, base, &regmap_config);
```

---

### ما بيمتلكه الـ Framework vs ما بيفوضه للـ Driver

| المسؤولية | Framework (core) | Driver (Meson) |
|---|---|---|
| لوكيشن الـ consumer handles | ✓ |  |
| Locking وref-counting | ✓ |  |
| ترجمة DT phandle → ID | ✓ (default xlate) |  |
| حساب register offset | | ✓ |
| Polarity (active-low/high) | | ✓ |
| نوع العملية (pulse vs level) | | ✓ |
| الوصول الفعلي للـ hardware | | ✓ (عبر regmap) |
| دعم chip variants | | ✓ (عبر param struct) |
| إعلان عدد الـ resets | يستقبله | يوفره (`nr_resets`) |

---

### الـ Module Namespace: `MESON_RESET`

```c
EXPORT_SYMBOL_NS_GPL(meson_reset_ops, "MESON_RESET");
EXPORT_SYMBOL_NS_GPL(meson_reset_controller_register, "MESON_RESET");
MODULE_IMPORT_NS("MESON_RESET");
```

الـ **`EXPORT_SYMBOL_NS_GPL`** بيصدّر الـ symbol في namespace معين. أي module تاني (زي `reset-meson-audio-arb.c`) محتاج يعمل `MODULE_IMPORT_NS("MESON_RESET")` عشان يقدر يستخدم الـ symbols دي. ده بيمنع استخدام الـ internal API من خارج الـ Meson subsystem من غير إذن صريح.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Config Options والـ Parameters المهمة

| Parameter | meson8b | meson_a1 | meson_s4 | t7 |
|-----------|---------|----------|----------|----|
| `reset_num` | 256 | 96 | 192 | 224 |
| `reset_offset` | 0x00 | 0x00 | 0x00 | 0x00 |
| `level_offset` | 0x7C | 0x40 | 0x40 | 0x40 |
| `level_low_reset` | true | true | true | true |
| `reset_ops` | `meson_reset_ops` | `meson_reset_ops` | `meson_reset_ops` | nil (no ops) |

> **ملحوظة:** الـ `t7_param` مش بيحدد `reset_ops` — يعني بيبقى `NULL`، ودا قصد أو bug محتمل.

---

| Flag / Bool | المعنى |
|---|---|
| `level_low_reset = true` | الـ assert (reset active) بيتم بكتابة `0` على البت مش `1` |
| `level_low_reset = false` | الـ assert بيتم بكتابة `1` على البت |

---

### الـ Structs الأساسية

---

#### `struct meson_reset_param` — الـ SoC Configuration Template

**الغرض:** بتحدد خصائص الـ reset controller لكل SoC. بتتعرف كـ `const static` — يعني read-only في الـ rodata.

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; // pointer للـ ops المناسبة للـ SoC دا
    unsigned int reset_num;                    // عدد الـ reset lines الكلي
    unsigned int reset_offset;                 // offset الـ self-clearing reset registers
    unsigned int level_offset;                 // offset الـ level (assert/deassert) registers
    bool level_low_reset;                      // هل assert = write 0 ولا write 1؟
};
```

- **بتتوصل بـ:** `meson_reset` عن طريق `data->param`
- **بتتوصل بـ:** `reset_control_ops` عن طريق `reset_ops`
- **بتيجي من:** الـ device tree matching عن طريق `of_device_id.data`

---

#### `struct meson_reset` — الـ Runtime Instance (في reset-meson-common.c)

**الغرض:** الـ state الفعلي للـ driver في الـ runtime — بيتعمل `devm_kzalloc` في الـ probe.

```c
struct meson_reset {
    const struct meson_reset_param *param; // pointer للـ SoC params (read-only)
    struct reset_controller_dev rcdev;     // الـ core reset framework struct (embedded!)
    struct regmap *map;                    // الـ regmap للوصول للـ MMIO registers
};
```

**ملحوظة مهمة:** الـ `rcdev` بيتعمل **embed** جوا `meson_reset` مش pointer. ده بيخلي الـ `container_of()` يشتغل — بتبدأ من `rcdev*` وترجع `meson_reset*`.

---

#### `struct reset_controller_dev` — الـ Kernel Reset Framework Interface

**الغرض:** الـ struct اللي بيعرّف الـ reset controller للـ kernel reset framework.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;      // الـ callbacks (reset/assert/deassert/status)
    struct module *owner;                      // الـ module المالك
    struct list_head list;                     // entry في القائمة العالمية للـ controllers
    struct list_head reset_control_head;       // قائمة الـ consumers اللي طالبين reset lines
    struct device *dev;                        // الـ device المرتبط
    struct device_node *of_node;               // الـ DT node كـ phandle target
    const struct of_phandle_args *of_args;     // للـ GPIO-based reset controllers
    int of_reset_n_cells;                      // عدد الـ cells في الـ reset specifier
    int (*of_xlate)(...);                      // translation: DT specifier → reset ID
    unsigned int nr_resets;                    // عدد الـ reset lines المتاحة
};
```

---

#### `struct reset_control_ops` — الـ Operations Table

**الغرض:** جدول الـ function pointers اللي بيربط الـ kernel reset API بالـ hardware.

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);    // pulse reset
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);   // assert (hold in reset)
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id); // deassert (release)
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);   // قراءة الحالة
};
```

الـ driver بيوفر نوعين:
- **`meson_reset_ops`** — بيستخدم write مباشر على الـ self-clearing register للـ `reset()`
- **`meson_reset_toggle_ops`** — بيعمل assert ثم deassert يدوياً (للـ SoCs اللي مافيهاش self-clearing)

---

#### `struct regmap_config` — إعداد الـ MMIO Regmap

```c
static const struct regmap_config regmap_config = {
    .reg_bits   = 32, // عنوان الـ register بـ 32-bit
    .val_bits   = 32, // القيمة بـ 32-bit
    .reg_stride = 4,  // كل register على حدة = 4 bytes (word-aligned)
};
```

---

### مخطط علاقات الـ Structs (ASCII)

```
  ┌─────────────────────────────────────────────────────────┐
  │              meson_reset  (heap, per-device)            │
  │                                                         │
  │  ┌─────────────────────┐   ┌──────────────────────────┐│
  │  │  *param             │──▶│  meson_reset_param        ││
  │  │  (const, rodata)    │   │  (reset_num, offsets,     ││
  │  └─────────────────────┘   │   level_low_reset,        ││
  │                            │   *reset_ops ──────────┐  ││
  │  ┌─────────────────────┐   └──────────────────────┘ │  ││
  │  │  *map               │──▶  regmap                  │  ││
  │  │  (MMIO abstraction) │     (wraps ioremap base)    │  ││
  │  └─────────────────────┘                             │  ││
  │                                                      │  ││
  │  ┌────────────────────────────────────────────────┐  │  ││
  │  │  rcdev  [embedded reset_controller_dev]        │  │  ││
  │  │                                                │  │  ││
  │  │   *ops ◀──────────────────────────────────────┘  │  ││
  │  │   *dev ──▶ platform_device.dev                    │  ││
  │  │   *of_node ──▶ device_node (DT)                   │  ││
  │  │   nr_resets = param->reset_num                    │  ││
  │  │   list ──▶ [global reset controller list]         │  ││
  │  └────────────────────────────────────────────────┘  │  ││
  └─────────────────────────────────────────────────────────┘

        reset_control_ops (rodata)
        ┌──────────────────────────────┐
        │ .reset    = meson_reset_reset│
        │ .assert   = meson_reset_assert│
        │ .deassert = meson_reset_deassert│
        │ .status   = meson_reset_status│
        └──────────────────────────────┘
```

---

### مخطط الـ Lifecycle: من الـ probe للـ teardown

```
  kernel boot / module load
         │
         ▼
  platform_driver_register()
  [via module_platform_driver macro]
         │
         ▼ (DT match found)
  meson_reset_probe(pdev)
         │
         ├─► devm_platform_ioremap_resource()
         │       maps the MMIO range → void __iomem *base
         │
         ├─► device_get_match_data()
         │       يجيب الـ meson_reset_param* من of_device_id.data
         │
         ├─► devm_regmap_init_mmio()
         │       بيلف الـ base في regmap abstraction layer
         │
         └─► meson_reset_controller_register(dev, map, param)
                 │
                 ├─► devm_kzalloc(sizeof(meson_reset))
                 │       allocates meson_reset on heap
                 │
                 ├─► data->param = param
                 ├─► data->map   = map
                 ├─► data->rcdev.owner     = dev->driver->owner
                 ├─► data->rcdev.nr_resets = param->reset_num
                 ├─► data->rcdev.ops       = param->reset_ops
                 ├─► data->rcdev.of_node   = dev->of_node
                 │
                 └─► devm_reset_controller_register(&data->rcdev)
                         يضيف الـ rcdev للقائمة العالمية
                         الـ consumers يقدروا يطلبوا reset lines دلوقتي

  ─────────────── runtime ───────────────
  consumer calls reset_control_reset(rstc)
         │
         ▼ (kernel reset framework)
  rcdev->ops->reset(rcdev, id)
         │
         ▼
  [hardware gets reset]

  ─────────────── teardown ───────────────
  device unbind / module unload
         │
         ▼
  devm cleanup (LIFO order):
    1. devm_reset_controller_register cleanup
       → يشيل الـ rcdev من القائمة العالمية
    2. devm_kzalloc cleanup
       → يحرر الـ meson_reset struct
    3. devm_regmap_init_mmio cleanup
       → يحرر الـ regmap
    4. devm_platform_ioremap_resource cleanup
       → يعمل iounmap للـ MMIO region
```

---

### مخطط الـ Call Flow: reset operation كاملة

```
consumer driver
  │
  │  reset_control_reset(rstc)
  ▼
[kernel reset framework core]
  │  يجيب الـ rcdev من الـ rstc
  │  بيتحقق إن id < nr_resets
  │
  │  rcdev->ops->reset(rcdev, id)
  ▼
meson_reset_reset(rcdev, id)
  │
  │  container_of(rcdev, struct meson_reset, rcdev)
  │  → يجيب الـ meson_reset* (data)
  │
  │  meson_reset_offset_and_bit(data, id, &offset, &bit)
  │    stride = regmap_get_reg_stride(map)  [= 4]
  │    offset = (id / 32) * 4
  │    bit    = id % 32
  │
  │  offset += data->param->reset_offset  [usually 0x0]
  │
  │  regmap_write(map, offset, BIT(bit))
  ▼
[regmap layer]
  │  writel(BIT(bit), base + offset)
  ▼
[hardware register]
  SoC reset register gets written
  → self-clearing pulse → peripheral is reset
```

---

### مخطط الـ Call Flow: assert/deassert (level reset)

```
consumer calls reset_control_assert(rstc)
  │
  ▼
meson_reset_assert(rcdev, id)
  │
  └─► meson_reset_level(rcdev, id, assert=true)
          │
          ├─► meson_reset_offset_and_bit() → offset, bit
          │
          ├─► offset += param->level_offset  [0x40 or 0x7c]
          │
          ├─► assert ^= param->level_low_reset
          │     [level_low_reset=true → assert becomes false → write 0]
          │     [يعني: لو السيستم active-low، نكتب 0 عشان نعمل assert]
          │
          └─► regmap_update_bits(map, offset, BIT(bit),
                                 assert ? BIT(bit) : 0)
                → بيعدل البت المطلوب بس، مش بيكتب الـ register كله


consumer calls reset_control_deassert(rstc)
  │
  ▼
meson_reset_deassert(rcdev, id)
  │
  └─► meson_reset_level(rcdev, id, assert=false)
          │  [نفس المنطق بس assert=false → بيكتب BIT(bit)]
          ▼
        regmap_update_bits(...)
```

---

### مخطط الـ Call Flow: status query

```
consumer calls reset_control_status(rstc)
  │
  ▼
meson_reset_status(rcdev, id)
  │
  ├─► meson_reset_offset_and_bit() → offset, bit
  ├─► offset += param->level_offset
  ├─► regmap_read(map, offset, &val)
  │     val = raw register value
  │
  ├─► val = !!(BIT(bit) & val)
  │     [يعني: هل البت المطلوب set؟ → 0 أو 1]
  │
  └─► return val ^ param->level_low_reset
        [لو active-low: 1 في الـ register = not-in-reset → نرجع 0]
        [لو active-high: 1 في الـ register = in-reset → نرجع 1]
```

---

### حساب الـ Register Offset والـ Bit

الـ reset lines بتتوزع على registers بـ 32-bit، كل register بيغطي 32 line:

```
id=0  → register offset=0x00, bit=0
id=1  → register offset=0x00, bit=1
...
id=31 → register offset=0x00, bit=31
id=32 → register offset=0x04, bit=0
id=63 → register offset=0x04, bit=31
id=64 → register offset=0x08, bit=0
...
```

الـ formula:
```
stride  = 4  (bytes per register)
offset  = (id / 32) * 4
bit     = id % 32
```

لو `reset_offset = 0x00`:
- reset register للـ id=33 → `0x00 + 0x04 = 0x04`, bit=1

لو `level_offset = 0x40`:
- level register للـ id=33 → `0x40 + 0x04 = 0x44`, bit=1

---

### استراتيجية الـ Locking

الـ driver **مش بيعمل أي locking يدوي**. الـ locking بيتم على مستويين خارجيين:

| المستوى | الـ Mechanism | من يتحكم فيه |
|---|---|---|
| **Regmap internal lock** | الـ `regmap` بيستخدم `spinlock` أو `mutex` داخلي على حسب الـ config | الـ regmap framework تلقائياً |
| **Reset framework lock** | الـ reset core بيحمي الـ `reset_control_head` list | الـ kernel reset framework |

**الـ `regmap_update_bits`** آمن concurrently لأن الـ regmap بيعمل read-modify-write تحت lock داخلي. ده مهم جداً لأن `assert`/`deassert` بيعملوا `update_bits` مش `write` كامل — بيحمي الـ bits التانية في نفس الـ register.

**الـ `meson_reset_reset`** بيستخدم `regmap_write` مش `regmap_update_bits` — ده لأن الـ register ده self-clearing pulse register، كل bit فيه مستقل ومش محتاج read-modify-write.

> **لا يوجد** `spinlock` أو `mutex` مُعرَّف في `meson_reset` نفسه — كل الحماية جاية من الـ regmap layer والـ reset framework.

---

### مخطط علاقة الـ of_device_id بالـ Params

```
Device Tree node
  compatible = "amlogic,meson-gxbb-reset"
          │
          ▼
  meson_reset_dt_ids[]
  ┌────────────────────────────────────────────┐
  │ "amlogic,meson8b-reset"   → &meson8b_param │
  │ "amlogic,meson-gxbb-reset"→ &meson8b_param │
  │ "amlogic,meson-axg-reset" → &meson8b_param │
  │ "amlogic,meson-a1-reset"  → &meson_a1_param│
  │ "amlogic,meson-s4-reset"  → &meson_s4_param│
  │ "amlogic,c3-reset"        → &meson_s4_param│
  │ "amlogic,t7-reset"        → &t7_param      │
  └────────────────────────────────────────────┘
          │
          ▼ device_get_match_data()
  const struct meson_reset_param *param
          │
          ▼
  مبعوت لـ meson_reset_controller_register()
```

**ملحوظة:** meson8b و gxbb و axg كلهم بيستخدموا نفس الـ param — نفس الـ layout للـ registers.
## Phase 4: شرح الـ Functions

### ملخص الـ Functions والـ APIs — Cheatsheet

#### Registration & Probe

| Function | File | الغرض |
|---|---|---|
| `meson_reset_probe` | reset-meson.c | Entry point للـ platform driver، بيعمل map للـ MMIO وبيسجّل الـ controller |
| `meson_reset_controller_register` | reset-meson-common.c | بيخصص الـ `meson_reset` struct وبيسجله مع الـ reset core |

#### Core Reset Ops

| Function | الـ callback اللي بتملأه |
|---|---|
| `meson_reset_reset` | `reset_control_ops.reset` — pulse reset |
| `meson_reset_assert` | `reset_control_ops.assert` — level assert |
| `meson_reset_deassert` | `reset_control_ops.deassert` — level deassert |
| `meson_reset_status` | `reset_control_ops.status` — اقرأ الحالة |
| `meson_reset_level_toggle` | `reset_control_ops.reset` في ops التانية (toggle variant) |

#### Helpers

| Function | الغرض |
|---|---|
| `meson_reset_offset_and_bit` | يحسب register offset والـ bit number من الـ reset id |
| `meson_reset_level` | kernel-internal، بتعمل assert أو deassert مع مراعاة الـ polarity |

#### Exported Ops Tables

| Symbol | الـ namespace | الاستخدام |
|---|---|---|
| `meson_reset_ops` | `MESON_RESET` | للـ SoCs اللي عندها self-deasserting reset hardware |
| `meson_reset_toggle_ops` | `MESON_RESET` | للـ SoCs اللي محتاجة software toggle في الـ reset |

---

### المجموعة الأولى: Probe & Registration

الهدف من المجموعة دي إنها تربط الـ hardware description اللي جاية من الـ Device Tree بالـ reset core framework. الـ probe بتعمل map للـ physical registers، وبعدين `meson_reset_controller_register` بيربط كل حاجة مع بعض في struct واحد.

---

#### `meson_reset_probe`

```c
static int meson_reset_probe(struct platform_device *pdev)
```

**الـ function دي شغلتها إيه؟**
الـ platform core بتناديها لما بيلاقي device بيتطابق مع أي compatible string في `meson_reset_dt_ids`. بتعمل ioremap للـ MMIO region الأولى، بتجيب الـ `meson_reset_param` المناسبة للـ SoC من الـ match data، وبتبني regmap فوق الـ MMIO وبعدين بتفوّض لـ `meson_reset_controller_register`.

**الـ Parameters:**
- `pdev` — الـ `platform_device` اللي الـ core عمل allocate ليه وعمرّ بياناته من الـ DT node.

**الـ Return Value:**
- `0` في حالة النجاح.
- قيمة سالبة من `PTR_ERR` لو فشل الـ ioremap أو الـ regmap init.
- `PTR_ERR` من `meson_reset_controller_register` لو فشل التسجيل.
- `-ENODEV` لو `device_get_match_data` رجعت NULL (مش المفروض يحصل لو الـ table صح).

**Key Details:**
- كل الـ allocations بتستخدم `devm_*` — يعني الـ cleanup بيحصل أوتوماتيك لما الـ device يتشال أو الـ probe يفشل.
- الـ `regmap_config` ثابتة: 32-bit registers، stride = 4 bytes — مناسبة لـ MMIO word-aligned registers.
- الـ `dev_err_probe` في حالة فشل الـ regmap بيطبع error message وبيتعامل صح مع حالة `-EPROBE_DEFER`.

**Pseudocode Flow:**
```
meson_reset_probe(pdev):
    base = devm_platform_ioremap_resource(pdev, 0)
    if ERR → return error

    param = device_get_match_data(dev)
    if NULL → return -ENODEV

    map = devm_regmap_init_mmio(dev, base, &regmap_config)
    if ERR → dev_err_probe + return error

    return meson_reset_controller_register(dev, map, param)
```

---

#### `meson_reset_controller_register`

```c
int meson_reset_controller_register(struct device *dev,
                                    struct regmap *map,
                                    const struct meson_reset_param *param)
```

**الـ function دي شغلتها إيه؟**
بتعمل `devm_kzalloc` لـ `struct meson_reset` اللي بتجمع الـ `regmap` والـ `param` والـ `reset_controller_dev` في struct واحد. بعدين بتملّأ حقول الـ `rcdev` وبتسجله مع الـ reset core عن طريق `devm_reset_controller_register`.

**الـ Parameters:**
- `dev` — الـ `struct device` بتاع الـ platform device (مستخدم للـ devm allocations والـ of_node).
- `map` — الـ regmap instance اللي فوق الـ MMIO.
- `param` — الـ SoC-specific configuration (عدد الـ resets، offsets، polarity).

**الـ Return Value:**
- `0` في حالة النجاح.
- `-ENOMEM` لو فشل الـ kzalloc.
- error من `devm_reset_controller_register` لو الـ core رفض التسجيل.

**Key Details:**
- الـ `rcdev.owner` بيتحط من `dev->driver->owner` — ده مهم لضمان صح الـ module reference counting.
- الـ `rcdev.of_node` بياخد قيمته من `dev->of_node` — ده اللي بيخلي الـ consumer يقدر يعمل `of_reset_simple_xlate` على الـ phandle.
- لو `param->reset_ops` بيت NULL (زي في `t7_param` اللي مش عندها `reset_ops`)، الـ framework هيعمل NULL dereference — ده bug محتمل في `t7_param`.
- الـ function دي exported بالـ `EXPORT_SYMBOL_NS_GPL` تحت namespace `MESON_RESET` — يعني الـ drivers التانية (زي `reset-meson-aux`) تقدر تستخدمها بشرط `MODULE_IMPORT_NS`.

**Pseudocode Flow:**
```
meson_reset_controller_register(dev, map, param):
    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL)
    if !data → return -ENOMEM

    data->param = param
    data->map   = map
    data->rcdev.owner     = dev->driver->owner
    data->rcdev.nr_resets = param->reset_num
    data->rcdev.ops       = param->reset_ops
    data->rcdev.of_node   = dev->of_node

    return devm_reset_controller_register(dev, &data->rcdev)
```

---

### المجموعة التانية: Helpers

---

#### `meson_reset_offset_and_bit`

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                        unsigned long id,
                                        unsigned int *offset,
                                        unsigned int *bit)
```

**الـ function دي شغلتها إيه؟**
بتحوّل الـ `id` اللي هو رقم الـ reset line (0-based) لـ register offset وبت number داخل الـ register ده. الـ calculation بتراعي الـ register stride اللي جاية من الـ regmap config (في الحالة دي 4 bytes).

**الـ Parameters:**
- `data` — الـ `meson_reset` instance اللي فيه الـ regmap وبالتالي الـ stride.
- `id` — رقم الـ reset line الـ consumer طلبه.
- `offset` — output: الـ byte offset للـ register (قبل إضافة الـ base offset زي `reset_offset` أو `level_offset`).
- `bit` — output: رقم الـ bit داخل الـ register (0..31).

**الـ Return Value:** void — بتكتب النتيجة في الـ pointers.

**Key Details:**
- الـ `stride = regmap_get_reg_stride(data->map)` — في حالتنا 4 bytes يعني 32 bit per register.
- المعادلة: `offset = (id / 32) * 4`، `bit = id % 32`.
- الـ caller هو المسؤول عن إضافة `param->reset_offset` أو `param->level_offset` على الـ offset الناتج.
- لا locking هنا — الـ regmap نفسه بيتولى الـ thread safety.

**مثال عملي:**
```
id = 37, stride = 4
stride * BITS_PER_BYTE = 32
offset = (37 / 32) * 4 = 1 * 4 = 4   → register at byte 4
bit    = 37 % 32        = 5           → bit 5
```

---

#### `meson_reset_level`

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                              unsigned long id, bool assert)
```

**الـ function دي شغلتها إيه؟**
الـ internal shared implementation للـ assert والـ deassert. بتحسب الـ offset والـ bit، وبعدين بتطبّق XOR مع `level_low_reset` لمعرفة القيمة الصح اللي لازم تتكتب على الـ hardware (active-low vs active-high polarity).

**الـ Parameters:**
- `rcdev` — الـ reset controller dev، بيتحول لـ `meson_reset` بـ `container_of`.
- `id` — رقم الـ reset line.
- `assert` — `true` للـ assert، `false` للـ deassert (من منظور الـ software).

**الـ Return Value:**
- `0` في حالة النجاح.
- error code من `regmap_update_bits` لو في مشكلة I/O.

**Key Details:**
- `assert ^= data->param->level_low_reset` — الـ XOR ده جوهري: لو `level_low_reset = true` (الحالة السائدة في كل الـ Meson SoCs)، فـ assert الـ software (active) = كتابة 0 على الـ hardware bit.
- `regmap_update_bits` بيضمن read-modify-write atomically على مستوى الـ regmap lock — بس مش hardware-atomic على الـ bus.
- الـ offset بيتضاف عليه `level_offset` (مش `reset_offset`) — لأن الـ level registers مختلفة عن الـ pulse-reset registers.

**Polarity Logic:**

```
assert (software) = true, level_low_reset = true:
    assert ^= true  →  assert = false
    val = 0  →  BIT(bit) cleared  →  hardware pulled LOW  →  device in reset ✓

deassert (software) = false, level_low_reset = true:
    assert ^= true  →  assert = true
    val = BIT(bit)  →  hardware HIGH  →  device out of reset ✓
```

---

### المجموعة التالتة: Reset Control Ops

---

#### `meson_reset_reset`

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                              unsigned long id)
```

**الـ function دي شغلتها إيه؟**
بتعمل **pulse reset** — بتكتب `1` في الـ bit المقابل للـ reset line في الـ pulse-reset register. الـ hardware نفسه بيعمل self-deassert بعد كده بدون تدخل من الـ software. ده الـ `.reset` callback في `meson_reset_ops`.

**الـ Parameters:**
- `rcdev` — الـ reset controller dev.
- `id` — رقم الـ reset line.

**الـ Return Value:**
- `0` في حالة النجاح من `regmap_write`.
- error code لو في مشكلة I/O.

**Key Details:**
- بتستخدم `regmap_write` (مش `regmap_update_bits`) — يعني بتكتب الـ register كله بـ single bit set. ده مقبول لأن كتابة الـ register ده بيعمل pulse على الـ bit المكتوب بس (الـ hardware design بيتكلم عن write-1-to-pulse).
- `offset += data->param->reset_offset` — الـ pulse registers في base مختلف عن الـ level registers.
- الـ function دي مش موجودة في `meson_reset_toggle_ops` — `t7_param` بتستخدم `meson_reset_level_toggle` كـ `.reset` بدلاً منها.

---

#### `meson_reset_assert`

```c
static int meson_reset_assert(struct reset_controller_dev *rcdev,
                               unsigned long id)
```

**الـ function دي شغلتها إيه؟**
الـ `.assert` callback — بتنادي `meson_reset_level(rcdev, id, true)` يعني بتطلب الـ hardware يفضل في حالة reset.

**Parameters & Return:** نفس `meson_reset_level` — راجع الشرح فوق.

**Who calls it:** الـ reset core لما consumer ينادي `reset_control_assert()`.

---

#### `meson_reset_deassert`

```c
static int meson_reset_deassert(struct reset_controller_dev *rcdev,
                                 unsigned long id)
```

**الـ function دي شغلتها إيه؟**
الـ `.deassert` callback — بتنادي `meson_reset_level(rcdev, id, false)` يعني بتطلق الـ device من الـ reset.

**Parameters & Return:** نفس `meson_reset_level`.

**Who calls it:** الـ reset core لما consumer ينادي `reset_control_deassert()`.

---

#### `meson_reset_status`

```c
static int meson_reset_status(struct reset_controller_dev *rcdev,
                               unsigned long id)
```

**الـ function دي شغلتها إيه؟**
بتقرأ الـ level register وبترجع الحالة الحالية للـ reset line بعد تطبيق الـ polarity correction. ترجع `1` لو الـ device في reset، `0` لو خارج reset.

**الـ Parameters:**
- `rcdev` — الـ reset controller dev.
- `id` — رقم الـ reset line.

**الـ Return Value:**
- `1` — الـ device في reset (asserted).
- `0` — الـ device خارج reset (deasserted).
- بيتجاهل error من `regmap_read` (ده design issue بسيط).

**Key Details:**
- `val = !!(BIT(bit) & val)` — بتعمل normalize للقيمة لـ 0 أو 1.
- `return val ^ data->param->level_low_reset` — الـ XOR نفس الـ polarity trick اللي في `meson_reset_level`، بس هنا في الاتجاه العكسي (read بدل write).
- **لو `level_low_reset = true`**: bit = 0 يعني في reset → XOR → ترجع 1. منطقي.
- **Who calls it:** الـ reset core لما consumer ينادي `reset_control_status()`.

---

#### `meson_reset_level_toggle`

```c
static int meson_reset_level_toggle(struct reset_controller_dev *rcdev,
                                     unsigned long id)
```

**الـ function دي شغلتها إيه؟**
ده الـ `.reset` callback في `meson_reset_toggle_ops`. بدل الـ hardware pulse، بيعمل software toggle — assert ثم deassert على التوالي. ده للـ SoCs اللي مش عندها dedicated pulse-reset register (زي السيناريو اللي `t7_param` ممكن تتعامل معاه).

**الـ Parameters:**
- `rcdev` — الـ reset controller dev.
- `id` — رقم الـ reset line.

**الـ Return Value:**
- `0` في حالة النجاح.
- error code من `meson_reset_assert` لو فشل.

**Key Details:**
- لو `assert` نجح وبعدين `deassert` فشل، الـ device هيفضل في reset — ده error path محتاج متابعة من الـ consumer.
- مافيش delay بين الـ assert والـ deassert — الـ hardware لازم يقدر يتعامل مع ده. في الـ SoCs الحديثة عادة كده.

**Pseudocode Flow:**
```
meson_reset_level_toggle(rcdev, id):
    ret = meson_reset_assert(rcdev, id)
    if ret → return ret

    return meson_reset_deassert(rcdev, id)
```

---

### المجموعة الرابعة: Static Data & Module Boilerplate

---

#### الـ `meson_reset_param` instances

الـ structs دي read-only compile-time configuration، محدش بيناديها كـ functions لكنها جوهرية لفهم الـ driver.

| Instance | `reset_num` | `reset_offset` | `level_offset` | `level_low_reset` | `reset_ops` |
|---|---|---|---|---|---|
| `meson8b_param` | 256 | 0x000 | 0x07C | true | `meson_reset_ops` |
| `meson_a1_param` | 96 | 0x000 | 0x040 | true | `meson_reset_ops` |
| `meson_s4_param` | 192 | 0x000 | 0x040 | true | `meson_reset_ops` |
| `t7_param` | 224 | 0x000 | 0x040 | true | **NULL** ⚠️ |

**ملاحظة على `t7_param`:** الـ `reset_ops` فيها NULL — يعني لو `meson_reset_controller_register` اتنادى بيها، هيتحط `rcdev.ops = NULL`. الـ reset core مش لازم يcrash فوراً لكن أي محاولة استخدام الـ reset line هتسبب NULL pointer dereference. محتمل إن الـ `t7_param` مقصود تستخدم مع `meson_reset_toggle_ops` من driver تاني أو إنها incomplete implementation.

---

#### `meson_reset_dt_ids`

```c
static const struct of_device_id meson_reset_dt_ids[]
```

الـ table دي بتربط كل `compatible` string بالـ `meson_reset_param` المناسبة. الـ `device_get_match_data()` في الـ probe بتجيب الـ `.data` pointer منها مباشرة.

| Compatible | الـ Param |
|---|---|
| `amlogic,meson8b-reset` | `meson8b_param` |
| `amlogic,meson-gxbb-reset` | `meson8b_param` |
| `amlogic,meson-axg-reset` | `meson8b_param` |
| `amlogic,meson-a1-reset` | `meson_a1_param` |
| `amlogic,meson-s4-reset` | `meson_s4_param` |
| `amlogic,c3-reset` | `meson_s4_param` |
| `amlogic,t7-reset` | `t7_param` |

---

#### `module_platform_driver`

```c
module_platform_driver(meson_reset_driver);
```

الـ macro دي بتوّلد `module_init` و`module_exit` تلقائياً. الـ `module_init` بتنادي `platform_driver_register` والـ `module_exit` بتنادي `platform_driver_unregister`. مافيش أي setup خاص عند الـ load/unload غير ده.

---

### ملخص تدفق الـ Execution الكامل

```
insmod  →  platform_driver_register(meson_reset_driver)
             ↓
         kernel matches DT node "amlogic,meson8b-reset"
             ↓
         meson_reset_probe(pdev)
           ├─ devm_platform_ioremap_resource  →  base (MMIO virtual addr)
           ├─ device_get_match_data           →  param (e.g. &meson8b_param)
           ├─ devm_regmap_init_mmio           →  map
           └─ meson_reset_controller_register(dev, map, param)
                ├─ devm_kzalloc              →  data
                ├─ fill data->rcdev fields
                └─ devm_reset_controller_register  →  rcdev added to global list

consumer driver calls reset_control_assert(rstc):
    → reset core looks up rcdev from rstc
    → rcdev->ops->assert(rcdev, id)
    → meson_reset_assert(rcdev, id)
    → meson_reset_level(rcdev, id, true)
    → meson_reset_offset_and_bit(data, id, &offset, &bit)
    → regmap_update_bits(map, offset + level_offset, BIT(bit), 0)
       (0 because level_low_reset=true → assert=true XOR true = false → val=0)
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده بيتحكم في **Amlogic Meson Reset Controller** — بيكتب على registers عبر **regmap/MMIO** عشان يعمل assert/deassert/reset لـ hardware blocks مختلفة. الـ debugging بيتركز على: صح ما يقراش الـ register، والـ DT يتطابق مع الـ hardware، والـ reset line تتصرف صح.

---

### Software Level

#### 1. debugfs

الـ regmap بيعمل entries تلقائي في debugfs لو `CONFIG_REGMAP` مفعّل.

```bash
# اعرف الـ mount point
mount | grep debugfs
# عادةً:
mount -t debugfs none /sys/kernel/debug

# اوصل لـ regmap dump للـ meson_reset
ls /sys/kernel/debug/regmap/
# هتلاقي entry زي: fd000000.reset  (العنوان بيتغير حسب الـ SoC)

# اقرأ كل الـ registers الـ regmap شايفها
cat /sys/kernel/debug/regmap/fd000000.reset/registers
```

**تفسير الـ output:**

```
00: 00000000   # reset_offset=0x00: كتابة BIT(n) هنا بتطلق pulse reset
7c: ffffffff   # level_offset=0x7c (meson8b): كل الـ bits deasserted (high=deasserted لأن level_low_reset=true)
40: ffffffff   # level_offset=0x40 (a1/s4): نفس المنطق
```

- لو bit = 0 في level register وـ `level_low_reset=true` → الـ block في حالة reset (asserted).
- لو bit = 1 → الـ block شغال (deasserted).

```bash
# اتحقق إن الـ regmap registers اتكتبت بعد probe
cat /sys/kernel/debug/regmap/*/name
```

#### 2. sysfs

الـ reset controller بيظهر نفسه كـ platform device:

```bash
# اعرف الـ device
ls /sys/bus/platform/devices/ | grep reset

# مثال:
ls /sys/bus/platform/devices/fd000000.reset/
# هتلاقي: driver  driver_override  modalias  of_node  power  subsystem  uevent

# تأكد إن الـ driver اتبند صح
cat /sys/bus/platform/devices/fd000000.reset/driver
# المفروض يرجع: ../../../../../../bus/platform/drivers/meson_reset

# اعرف الـ of_node
ls /sys/bus/platform/devices/fd000000.reset/of_node
```

الـ **reset consumers** بيظهروا في:
```bash
# كل device بتطلب reset control بيبان هنا
ls /sys/kernel/debug/reset/
# لو CONFIG_RESET_CONTROLLER=y وـ CONFIG_DEBUG_FS=y
```

#### 3. ftrace

```bash
# فعّل tracing على probe function
cd /sys/kernel/debug/tracing

# تتبع function calls جوا الـ driver
echo 'meson_reset_*' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# شغّل العملية اللي عايز تتتبعها (مثلاً reset consumer)
# بعدين اقرأ الـ trace
cat trace

echo 0 > tracing_on
echo nop > current_tracer
```

**events مفيدة:**

```bash
# تتبع regmap reads/writes (بيكشف القيم الفعلية المكتوبة في الـ registers)
echo 1 > events/regmap/regmap_reg_write/enable
echo 1 > events/regmap/regmap_reg_read/enable
echo 1 > tracing_on

cat trace | grep -E "fd000000"
```

**مثال output:**

```
kworker/0:1-42    [000] ....  123.456: regmap_reg_write: fd000000.reset map=fd000000 reg=7c val=ffffffff
ethphy-0         [000] ....  123.457: regmap_reg_write: fd000000.reset map=fd000000 reg=7c val=fffffffe
```

السطر التاني بيقول: bit 0 في level_offset اتعمله clear → block[0] اتعمله assert reset.

#### 4. printk / dynamic debug

الـ driver ما فيهوش `dev_dbg` كتير، لكن الـ regmap core فيه:

```bash
# فعّل dynamic debug لـ regmap
echo 'module regmap +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module reset_controller +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ platform driver بالاسم
echo 'file drivers/reset/amlogic/reset-meson.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/reset/amlogic/reset-meson-common.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# الـ flags: p=print f=func l=line m=module t=thread
```

لو احتجت تضيف debug مؤقت، النقاط المهمة في `reset-meson-common.c`:

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                             unsigned long id)
{
    // ...
    meson_reset_offset_and_bit(data, id, &offset, &bit);
    offset += data->param->reset_offset;

    dev_dbg(rcdev->dev, "pulse reset id=%lu reg=0x%x bit=%u\n",
            id, offset, bit);  // <-- نقطة مفيدة

    return regmap_write(data->map, offset, BIT(bit));
}
```

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_RESET_CONTROLLER=y` | الـ reset framework الأساسي |
| `CONFIG_REGMAP=y` | الـ regmap core |
| `CONFIG_REGMAP_MMIO=y` | MMIO backend للـ regmap |
| `CONFIG_DEBUG_FS=y` | يفتح /sys/kernel/debug |
| `CONFIG_REGMAP_DEBUGFS=y` | يعمل dump لـ registers في debugfs |
| `CONFIG_DYNAMIC_DEBUG=y` | يفعّل dynamic debug |
| `CONFIG_FTRACE=y` | function tracing |
| `CONFIG_KALLSYMS=y` | أسماء الـ symbols في stack trace |
| `CONFIG_OF=y` | Device Tree parsing |
| `CONFIG_PROVE_LOCKING=y` | يكشف deadlocks في reset ops |
| `CONFIG_PM_DEBUG=y` | لو فيه مشاكل في suspend/resume |

```bash
# تأكد الـ configs مفعّلة
zcat /proc/config.gz | grep -E 'RESET_CONTROLLER|REGMAP|DEBUG_FS'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# اعرف كل الـ reset controllers المسجّلة في الـ kernel
cat /sys/kernel/debug/reset/
# لو الـ file موجود — بيظهر كل rcdev

# اعرف عدد الـ resets المسجّلة
grep -r meson_reset /sys/bus/platform/drivers/

# استخدم devlink لو الـ device مرتبط بـ netdev
# (مش مباشر هنا لكن Ethernet PHY ممكن يستخدم reset)
devlink dev show

# اعرف الـ consumers بتوع reset معين
# (من خلال الـ DT)
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "resets"
```

#### 7. جدول Error Messages

| رسالة في dmesg | المعنى | الحل |
|----------------|--------|-------|
| `can't init regmap mmio region` | فشل `devm_regmap_init_mmio` | تحقق من الـ resource في DT (reg property)، تأكد الـ address map صح |
| `meson_reset: probe of fd000000.reset failed with error -22` | `-ENODEV` من `device_get_match_data` | الـ compatible string في DT مش موجود في `meson_reset_dt_ids` |
| `meson_reset: probe of fd000000.reset failed with error -12` | `-ENOMEM` في `devm_kzalloc` | نادر — memory ضيقة |
| `meson_reset: probe of fd000000.reset failed with error -16` | `-EBUSY` من resource claim | resource اتحجزت من driver تاني |
| `reset controller not ready, waiting` | الـ consumer طلب reset قبل ما الـ controller يتسجل | تأكد من `probe ordering` أو استخدم `deferred probe` |
| `reset_controller_register failed` | فشل تسجيل الـ rcdev | تحقق من `nr_resets > 0` وـ `ops != NULL` |

#### 8. نقاط WARN_ON و dump_stack

النقاط الاستراتيجية لو احتجت تـ debug مشكلة حسابية في offset:

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
{
    unsigned int stride = regmap_get_reg_stride(data->map);

    // تأكد إن الـ id في النطاق الصح
    WARN_ON(id >= data->param->reset_num);

    *offset = (id / (stride * BITS_PER_BYTE)) * stride;
    *bit = id % (stride * BITS_PER_BYTE);
}
```

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
{
    // ...
    // تحقق إن الـ level_offset محدد
    WARN_ON(!data->param->level_offset && assert);
    // ...
}
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel

الـ kernel بيحسب الـ register offset من الـ id:

```
offset = (id / 32) * 4   (لأن stride=4, BITS_PER_BYTE=8)
bit    = id % 32
```

**مثال:** id=5 → offset=0, bit=5

```bash
# اقرأ level register يدوياً وقارن بـ kernel status
# meson8b: level_offset=0x7c, base=0xc1104000
# اقرأ الـ register الأول (bits 0-31)
devmem2 0xc110407c w
# لو bit 5 = 0 مع level_low_reset=true → block 5 في حالة reset
```

```bash
# للـ S4/A1: level_offset=0x40
devmem2 <base_addr+0x40> w
```

#### 2. Register Dump Techniques

```bash
# devmem2 (يحتاج تثبيت)
apt-get install devmem2
# أو
busybox devmem <addr> w

# اقرأ reset_offset (pulse registers) - meson8b
BASE=0xc1104000
devmem2 $((BASE + 0x00)) w   # reset bits 0-31
devmem2 $((BASE + 0x04)) w   # reset bits 32-63
# ...حتى 256 bits = 8 registers

# اقرأ level_offset (assert/deassert state) - meson8b
devmem2 $((BASE + 0x7c)) w   # level bits 0-31
devmem2 $((BASE + 0x80)) w   # level bits 32-63

# للـ A1/S4: BASE يختلف حسب الـ SoC datasheet
# level_offset=0x40
devmem2 $((BASE + 0x40)) w
```

```bash
# لو devmem2 مش متاح، استخدم /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xc110407c / 4)) | xxd

# أو io utility من busybox
io -4 -r 0xc110407c
```

**جدول تفسير القيم:**

| قيمة level register | level_low_reset=true | الحالة |
|---------------------|----------------------|--------|
| bit=1 | true | deasserted (block شغال) |
| bit=0 | true | asserted (block في reset) |
| bit=1 | false | asserted (block في reset) |
| bit=0 | false | deasserted (block شغال) |

#### 3. Logic Analyzer / Oscilloscope

لو عندك access لـ reset signals على الـ board:

- **Reset pulse** (من `meson_reset_reset`): بتشوف pulse قصيرة active-low على السيگنال — عرضها بيتحدد بـ hardware، مش software.
- **Level reset** (من `meson_reset_assert`): السيگنال بيفضل low لحد ما `deassert` يتنادى.

```
Assert:    ‾‾‾‾|_____________________|‾‾‾‾
                ↑ assert               ↑ deassert

Pulse:     ‾‾‾‾|__|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
                ↑ brief pulse (one write cycle)
```

**نصائح عملية:**
- اربط probe على reset pin للـ component المشكوك فيه.
- trigger على falling edge.
- قيس عرض الـ pulse — لو صغير جداً، الـ hardware ممكن يتجاهله.
- لو الـ signal ما بيروحش low خالص → الـ register address غلط أو الـ clock مش شغّال.

#### 4. Hardware Issues → Kernel Log Patterns

| مشكلة hardware | pattern في dmesg |
|----------------|-----------------|
| الـ reset controller clock مش مفعّل | probe يفشل بـ `-ENODEV` أو الـ regmap writes تتجاهل |
| عنوان الـ base خاطئ في DT | `can't init regmap mmio region` أو kernel panic في probe |
| block مش بيخرج من reset | الـ consumer driver يعمل timeout أو يرجع `-ETIMEDOUT` |
| polarity عكوسة (level_low_reset خاطئة) | الـ block شغّال بدري أوي أو ما بيشتغلش خالص |
| reset pulse قصير جداً | الـ hardware بيشتغل بس بيقع بعد كده بدون سبب واضح |
| تعارض بين درايفرين على نفس الـ register | `regmap_update_bits` ما بيشتغلش صح — لازم locking |

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT node بتاع الـ reset controller
dtc -I fs /sys/firmware/devicetree/base -O dts | grep -A20 "reset-controller"

# أو مباشرة
ls /sys/firmware/devicetree/base/soc/reset-controller@*/
cat /sys/firmware/devicetree/base/soc/reset-controller@fd000000/compatible
# المفروض يطبع: amlogic,meson-s4-reset (أو الـ compatible المناسب)

cat /sys/firmware/devicetree/base/soc/reset-controller@fd000000/reg
# بيطبع الـ base address كـ big-endian bytes
```

**تحقق من الـ DT صح:**

```bash
# تأكد إن الـ compatible موجود في الـ driver
grep "amlogic,meson-s4-reset" /sys/firmware/devicetree/base/soc/*/compatible

# تحقق من الـ consumers بيشيروا لـ reset controller صح
dtc -I fs /sys/firmware/devicetree/base -O dts | grep -B2 -A5 "resets = "
```

**DT صحيح لـ S4:**

```dts
reset: reset-controller@fd000000 {
    compatible = "amlogic,meson-s4-reset";
    reg = <0x0 0xfd000000 0x0 0x100>;
    #reset-cells = <1>;
};

/* consumer example */
ethernet: ethernet@... {
    resets = <&reset 40>;  /* reset id=40 */
    reset-names = "mac";
};
```

**لو `#reset-cells = <1>` مش موجود** → الـ consumer هيفشل في `of_reset_simple_xlate`.

---

### Practical Commands

#### Shell Commands جاهزة للنسخ

```bash
#!/bin/bash
# === Meson Reset Controller Debug Script ===

RESET_BASE="fd000000"  # غيّر حسب الـ SoC

echo "=== 1. Check driver loaded ==="
lsmod | grep meson_reset
ls /sys/bus/platform/drivers/meson_reset/

echo "=== 2. Check device bound ==="
ls -la /sys/bus/platform/devices/${RESET_BASE}.reset/driver

echo "=== 3. Regmap register dump ==="
cat /sys/kernel/debug/regmap/${RESET_BASE}.reset/registers 2>/dev/null || \
    echo "debugfs not available - check CONFIG_REGMAP_DEBUGFS"

echo "=== 4. DT compatible check ==="
cat /sys/firmware/devicetree/base/soc/reset-controller@${RESET_BASE}/compatible 2>/dev/null | strings

echo "=== 5. Enable regmap tracing ==="
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_read/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "Tracing enabled - run your test then check:"
echo "  cat /sys/kernel/debug/tracing/trace | grep ${RESET_BASE}"

echo "=== 6. Check reset consumers ==="
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B3 -A2 "resets = "
```

```bash
# تحقق من حالة reset معين (مثلاً id=40 على S4)
# BASE_ADDR من الـ DT reg property
BASE_ADDR=0xfd000000
LEVEL_OFFSET=0x40   # لـ S4
RESET_ID=40

REG_OFFSET=$(( (RESET_ID / 32) * 4 ))
BIT_NUM=$(( RESET_ID % 32 ))
ADDR=$(( BASE_ADDR + LEVEL_OFFSET + REG_OFFSET ))

printf "Reading level register for reset %d:\n" $RESET_ID
printf "  Address: 0x%x\n" $ADDR
printf "  Bit: %d\n" $BIT_NUM

VAL=$(devmem2 $(printf "0x%x" $ADDR) w 2>/dev/null | grep "Value" | awk '{print $NF}')
echo "  Register value: $VAL"

# فسّر الـ bit
BIT_VAL=$(( (0x${VAL#0x} >> BIT_NUM) & 1 ))
if [ $BIT_VAL -eq 1 ]; then
    echo "  Status: DEASSERTED (block running) - level_low_reset=true"
else
    echo "  Status: ASSERTED (block in reset) - level_low_reset=true"
fi
```

```bash
# فعّل dynamic debug لـ reset framework كله
echo 'file drivers/reset/* +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'module meson_reset +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep reset
```

```bash
# ftrace: تتبع كل calls لـ meson_reset functions
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo 'meson_reset_*' > set_ftrace_filter
cat set_ftrace_filter  # تأكد الـ functions اتضافت
echo function > current_tracer
echo 1 > tracing_on

# شغّل الـ consumer (مثلاً modprobe للـ ethernet driver)
# بعدين:
cat trace | head -50
echo 0 > tracing_on
echo nop > current_tracer
echo > set_ftrace_filter
```

#### تفسير الـ Output

**regmap register dump مثال (meson8b, 256 resets):**

```
$ cat /sys/kernel/debug/regmap/c1104000.reset/registers
00: 00000000   # pulse reset reg[0]  - write-only, reads 0
04: 00000000   # pulse reset reg[1]
...
7c: ffffffff   # level reg[0]: bits 0-31 كلهم 1 = جميع الـ blocks deasserted
80: ffffffef   # level reg[1]: bit 4 = 0 → block[36] في حالة reset
84: ffffffff
...
```

الـ block[36]: offset = (36/32)*4 = 4, bit = 36%32 = 4. في register 0x80، bit 4 = 0 مع level_low_reset=true → asserted.

**ftrace output:**

```
# tracer: function
#
#    TASK-PID    CPU#  IRQS-OFF  TIMESTAMP  FUNCTION
          eth0-523   [001]    d...  1234.567: meson_reset_deassert <-reset_control_deassert
          eth0-523   [001]    d...  1234.568: meson_reset_level <-meson_reset_deassert
          eth0-523   [001]    d...  1234.569: meson_reset_offset_and_bit <-meson_reset_level
```

الـ sequence ده طبيعي — الـ ethernet driver بيطلب deassert للـ MAC reset.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box بـ Amlogic S905X3 — الـ HDMI مش شغال بعد Suspend/Resume

#### العنوان
**HDMI output** مش بترجع بعد الـ suspend على TV box بـ Amlogic S905X3 (مبني على **meson-sm1** يستخدم `meson8b_param`).

#### السياق
شركة بتعمل Android TV box للسوق الأفريقي، الـ SoC هو Amlogic S905X3 (SM1 family). المنتج بيعمل كويس في التشغيل العادي، بس بعد ما الجهاز يدخل suspend ويصحى، شاشة الـ HDMI بتفضل سودا — مفيش صورة خالص.

#### المشكلة
الـ HDMI controller محتاج reset صريح بعد كل resume cycle عشان يرجع يشتغل. الـ `reset-meson.c` سجّل الـ reset controller باستخدام `meson8b_param` اللي فيها:
- `level_low_reset = true` → يعني الـ assert بيكون بـ clear البت (write 0)
- `level_offset = 0x7c` → ده عنوان رجستر الـ level reset

المشكلة إن الـ HDMI driver بيعمل `reset_control_reset()` (اللي بتستدعي `.reset` op) بدل ما يعمل assert ثم deassert بشكل صريح، وده بيتسبب في إن الـ reset pulse مش بيكون كافي.

#### التحليل
لما بنتبع الكود في `reset-meson.c`:

```c
// meson_reset_probe بتشيل الـ param من DT
param = device_get_match_data(dev);   // هيجيب meson8b_param
// ...
return meson_reset_controller_register(dev, map, param);
// الـ rcdev.nr_resets = 256 (meson8b_param.reset_num)
```

الـ `meson_reset_ops` (معرّف في الـ core) بتوفر `.reset` و `.assert` و `.deassert`. لما الـ HDMI driver بيعمل `reset_control_reset()`:
1. بتكتب 1 في رجستر `reset_offset` → pulse assert
2. **بس** الـ level registers في `level_offset = 0x7c` مش اتغيرتش

الرجستر في `0x7c` هو اللي بيتحكم في الـ persistent reset level. بعد الـ resume، الـ hardware بيقرأ من الـ level register مش من الـ pulse register، فالـ HDMI بيفضل في حالة reset.

```
MMIO Map (meson8b_param):
offset 0x00: Pulse Reset Registers  (reset_offset)
offset 0x7c: Level Reset Registers  (level_offset)

level_low_reset = true:
  assert   → clear bit in 0x7c (write 0 = hold in reset)
  deassert → set bit in 0x7c   (write 1 = release from reset)
```

#### الحل
الـ HDMI driver لازم يستخدم assert/deassert بشكل صريح:

```c
/* في hdmi driver resume callback */
ret = reset_control_assert(hdmi->reset);
if (ret)
    return ret;

/* تأخير بسيط عشان الـ hardware يستقر */
udelay(10);

ret = reset_control_deassert(hdmi->reset);
if (ret)
    return ret;
```

وفي الـ DT لازم تتأكد إن الـ HDMI node بيشاور على الـ reset controller صح:

```dts
hdmi_tx: hdmi-tx@ff600000 {
    compatible = "amlogic,meson-sm1-dw-hdmi";
    resets = <&reset_ctrl RESET_HDMITX>;
    reset-names = "hdmitx";
    /* ... */
};
```

#### الدرس المستفاد
على SoCs اللي بتستخدم `level_low_reset = true`، الـ pulse reset (`.reset` op) وحده مش كافي بعد power event. لازم دايمًا تستخدم `.assert` + `.deassert` عشان تضمن إن الـ level register اتغير فعلًا.

---

### السيناريو 2: Industrial Gateway بـ Amlogic A1 — الـ SPI Flash مش بيتعرف عليه

#### العنوان
**SPI NOR Flash** مش بيظهر على industrial gateway بعد kernel upgrade، رغم إن الـ hardware ماتغيرش.

#### السياق
منتج industrial gateway بيستخدم Amlogic A1 SoC للـ IoT applications. الـ SPI flash بيخزن configuration data حرجة. بعد upgrade من kernel 5.15 لـ 6.6، الـ flash مش بيتعرف عليه خالص في الـ boot.

#### المشكلة
الـ `meson_a1_param` معرّفة كالآتي في الـ driver:

```c
static const struct meson_reset_param meson_a1_param = {
    .reset_ops    = &meson_reset_ops,
    .reset_num    = 96,       // 96 reset line بس
    .reset_offset = 0x0,
    .level_offset = 0x40,
    .level_low_reset = true,
};
```

الـ `reset_num = 96` يعني الـ driver بيرفض أي reset ID أكبر من 95. الـ kernel الجديد بدّل رقم الـ SPI reset line من 94 لـ 96 في الـ DT بتاع الـ upstream، وده بيخلي الـ `of_xlate` يرجع error.

#### التحليل
في `meson_reset_probe`:

```c
param = device_get_match_data(dev);
// param->reset_num = 96

return meson_reset_controller_register(dev, map, param);
// بيعمل: rcdev->nr_resets = param->reset_num = 96
```

لما الـ SPI driver بيطلب reset رقم 96:

```c
// في reset core (reset-simple.c أو reset-meson core):
if (id >= rcdev->nr_resets)
    return -EINVAL;  // ← هنا بيفشل
```

الـ kernel log بيظهر:

```
[    2.341] spi-nor spi0.0: failed to get reset: -EINVAL
[    2.342] spi-nor: probe of spi0.0 failed with error -22
```

#### الحل
**أولًا:** تحقق من الـ DT المستخدم:

```bash
# على الجهاز
cat /proc/device-tree/soc/spi@fe010000/resets
# أو
dtc -I fs /proc/device-tree | grep -A5 "spi@fe"
```

**ثانيًا:** إذا الـ reset ID اتغير في الـ upstream DT، رجّع الـ DT القديم أو استخدم الـ ID الصح:

```dts
/* DT صح للـ A1 */
spifc: spi@fe010000 {
    resets = <&reset_ctrl 94>;  /* ← الرقم الصح للـ A1 */
};
```

**ثالثًا:** لو فعلًا الـ hardware محتاج reset 96 (خارج الـ 96 line)، الـ `meson_a1_param` محتاج تعديل في الـ driver — لكن ده unlikely لأن الـ A1 عنده 96 reset line فقط.

```bash
# debug: اطبع الـ reset controller info
cat /sys/kernel/debug/reset/meson_reset/
```

#### الدرس المستفاد
الـ `reset_num` في `meson_reset_param` هو حد صارم — أي ID تعداه بيعمل `-EINVAL`. لما بتعمل kernel upgrade، اتأكد إن الـ DT بتاعك متزامن مع الـ param table في الـ driver، خصوصًا لو الـ upstream غيّر numbering.

---

### السيناريو 3: Custom Board Bring-up بـ Amlogic S4 — الـ USB3 مش شغال خالص

#### العنوان
**USB 3.0 port** مش بيشتغل على custom board بـ Amlogic S4 SoC رغم إن الـ DT صح.

#### السياق
شركة electronics بتعمل custom development board على Amlogic S4 (مستخدم في TV applications). المهندس بدأ الـ board bring-up وعمل DT كامل، لكن الـ USB3 port مش بيظهر ومفيش devices بتتعرف عليها.

#### المشكلة
الـ `meson_s4_param` معرّفة:

```c
static const struct meson_reset_param meson_s4_param = {
    .reset_ops    = &meson_reset_ops,
    .reset_num    = 192,
    .reset_offset = 0x0,
    .level_offset = 0x40,     // ← مختلف عن meson8b (0x7c)
    .level_low_reset = true,
};
```

المهندس copy-paste الـ DT من board تانية بـ GXBB (اللي بتستخدم `meson8b_param`) ونسي يغيّر الـ compatible string.

#### التحليل
الـ DT المغلوط:

```dts
/* غلط — compatible خاطئ */
reset: reset-controller@ffd01000 {
    compatible = "amlogic,meson-gxbb-reset";  // ← غلط!
    reg = <0x0 0xffd01000 0x0 0x100>;
    #reset-cells = <1>;
};
```

لما `meson_reset_probe` بتشتغل:

```c
param = device_get_match_data(dev);
// بيجيب meson8b_param بدل meson_s4_param
// level_offset = 0x7c بدل 0x40 ← هنا المشكلة
```

لما الـ USB3 PHY driver بيعمل deassert:
- بيكتب في offset `0x7c` (عنوان خاطئ للـ S4)
- الـ S4 hardware بيتجاهل الكتابة دي أو بتعمل side effects
- الـ USB3 PHY بيفضل في reset

```bash
# على الجهاز، شوف الـ reset controller اللي اتسجل
cat /sys/bus/platform/devices/ffd01000.reset-controller/of_node/compatible
# النتيجة: amlogic,meson-gxbb-reset  ← المشكلة واضحة
```

#### الحل
صحح الـ DT:

```dts
reset: reset-controller@ffd01000 {
    compatible = "amlogic,meson-s4-reset";  // ← الصح
    reg = <0x0 0xffd01000 0x0 0x100>;
    #reset-cells = <1>;
};
```

للتحقق بعد الإصلاح:

```bash
# تأكد إن الـ regmap بيكتب في العنوان الصح
echo 'file reset-meson.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | grep meson_reset
```

#### الدرس المستفاد
الـ `compatible` string في الـ DT مش مجرد اسم — هو اللي بيحدد أي `meson_reset_param` بيتستخدم، وبالتالي بيحدد `level_offset` الصحيح. خطأ بسيط في الـ compatible على board جديدة يقدر يعطّل peripheral كامل.

---

### السيناريو 4: Production Line Failure — Amlogic T7 بـ Automotive ECU

#### العنوان
**Automotive ECU** بـ Amlogic T7 بيفشل في الـ POST (Power-On Self Test) على خط الإنتاج بنسبة 30%.

#### السياق
شركة automotive بتنتج ECU module بـ Amlogic T7 SoC للتحكم في نظام infotainment. على خط الإنتاج، 30% من الوحدات بتفشل في POST — التشخيص الأولي بيشاور على مشكلة في الـ CAN bus controller اللي محتاج reset في البداية.

#### المشكلة
الـ `t7_param` معرّفة بدون `reset_ops`:

```c
static const struct meson_reset_param t7_param = {
    .reset_num      = 224,
    .reset_offset   = 0x0,
    .level_offset   = 0x40,
    .level_low_reset = true,
    // reset_ops مش محددة! → NULL
};
```

مقارنةً بباقي الـ params:

```c
static const struct meson_reset_param meson8b_param = {
    .reset_ops = &meson_reset_ops,   // ← موجودة
    // ...
};
```

لما الـ CAN driver بيطلب `.assert`:

```c
// في reset core
if (!rcdev->ops->assert)
    return -ENOTSUPP;
```

بس لأن `t7_param.reset_ops = NULL`، الـ `meson_reset_controller_register` (في الـ core) بيستخدم `meson_reset_toggle_ops` بدلًا — اللي ممكن يكون ليه سلوك مختلف على الـ T7.

#### التحليل
```c
// في meson_reset_probe:
param = device_get_match_data(dev);  // t7_param
// param->reset_ops = NULL

return meson_reset_controller_register(dev, map, param);
```

الـ `meson_reset_controller_register` في الـ core بتشيك:

```c
// في reset-meson-core (غير موضح هنا لكن منطقي):
if (!param->reset_ops)
    rcdev->ops = &meson_reset_toggle_ops;  // fallback
else
    rcdev->ops = param->reset_ops;
```

الـ `meson_reset_toggle_ops` بتعمل toggle بدل assert/deassert مستقلين. على بعض الـ production boards، الـ CAN controller محتاج hold time محدد في الـ assert state — الـ toggle سريع بيكون مش كافي.

```bash
# اشوف الـ ops المستخدمة
cat /sys/kernel/debug/reset/*/ops
# أو
dmesg | grep -i "reset\|can"
```

#### الحل
**أولًا:** اعرف إذا الـ T7 محتاج `meson_reset_ops` أو `meson_reset_toggle_ops`:

```bash
# اقرأ الـ T7 TRM لعنوان الـ reset registers
# وتأكد إن الـ pulse registers موجودة (reset_offset = 0x0)
```

**ثانيًا:** إذا الـ T7 بيدعم الـ standard ops، الـ fix في الـ driver:

```c
/* في reset-meson.c */
static const struct meson_reset_param t7_param = {
    .reset_ops      = &meson_reset_ops,  // ← أضفها
    .reset_num      = 224,
    .reset_offset   = 0x0,
    .level_offset   = 0x40,
    .level_low_reset = true,
};
```

**ثالثًا:** في الـ CAN driver، زود الـ assert hold time:

```c
reset_control_assert(can->reset);
usleep_range(100, 200);   // hold لمدة كافية
reset_control_deassert(can->reset);
```

#### الدرس المستفاد
الـ `t7_param` في الكود الحالي مش بيحدد `reset_ops` — ده design choice أو bug محتاج مراجعة مع الـ T7 TRM. على automotive applications اللي فيها timing requirements صارمة، الـ reset op behavior لازم يكون deterministic ومتوثق.

---

### السيناريو 5: Amlogic S905Y2 Smart Display — الـ I2C بيتعلق بعد kernel panic ويحتاج hard reboot

#### العنوان
**I2C controller** على Amlogic S905Y2 (GXLX2 family, يستخدم `meson8b_param`) بيفضل stuck في bus-busy state بعد kernel panic ومحتاج hard power cycle.

#### السياق
شركة بتصنع smart display بـ Amlogic S905Y2 للـ retail signage. المنتج بيشتغل 24/7 وعنده watchdog بيعمل reset تلقائي لما يحصل panic. بعد الـ watchdog reset، الـ I2C bus بيفضل في busy state ومش بيقدر يتكلم مع الـ touch controller.

#### المشكلة
الـ watchdog reset بيعمل software reset للـ CPU لكن مش بيعمل hardware reset للـ I2C controller. الـ I2C controller بيفضل في نص transaction من قبل الـ panic — الـ SDA line بيفضل low (bus-busy condition).

الـ `meson8b_param`:
```c
static const struct meson_reset_param meson8b_param = {
    .reset_ops    = &meson_reset_ops,
    .reset_num    = 256,
    .reset_offset = 0x0,
    .level_offset = 0x7c,
    .level_low_reset = true,
};
```

الـ I2C driver مش بيعمل reset في الـ probe بعد watchdog restart، وبالتالي الـ controller بيبدأ في حالة غير نظيفة.

#### التحليل
عند الـ boot بعد watchdog reset:

```c
// meson_reset_probe بتشتغل عادي
base = devm_platform_ioremap_resource(pdev, 0);
param = device_get_match_data(dev);  // meson8b_param
map = devm_regmap_init_mmio(dev, base, &regmap_config);
meson_reset_controller_register(dev, map, param);
// الـ controller اتسجل — كل حاجة تمام
```

المشكلة مش في `reset-meson.c` نفسه — الـ driver اشتغل صح. المشكلة إن الـ I2C driver مش بيستغل الـ reset controller في الـ probe:

```c
// i2c driver probe (المفروض يكون كده):
reset = devm_reset_control_get(dev, "i2c");
reset_control_reset(reset);  // ← مش موجود في بعض drivers
```

الـ level register في `0x7c` لو اتعمل له assert (bit clear) وبعدين deassert (bit set)، الـ I2C controller هيبدأ من نظيف.

```bash
# تشخيص: شوف حالة الـ I2C bus
i2cdetect -y 0  # هيرجع error لو busy

# شوف الـ reset controller متاح؟
ls /sys/kernel/debug/reset/
cat /sys/kernel/debug/reset/meson_reset/reset0

# جرب manual reset (لو debugfs مدعوم)
```

#### الحل

**أولًا:** في الـ I2C DT node، أضف الـ reset:

```dts
i2c0: i2c@ffd1f000 {
    compatible = "amlogic,meson-axg-i2c";
    resets = <&reset_ctrl RESET_I2C_M_A>;
    reset-names = "i2c";
    /* ... */
};
```

**ثانيًا:** في الـ I2C driver probe، ضيف reset sequence:

```c
static int meson_i2c_probe(struct platform_device *pdev)
{
    struct reset_control *rst;

    rst = devm_reset_control_get_optional(dev, "i2c");
    if (!IS_ERR_OR_NULL(rst)) {
        reset_control_assert(rst);
        udelay(5);
        reset_control_deassert(rst);
    }
    /* باقي الـ probe */
}
```

**ثالثًا:** للـ production fix السريع، استخدم sysfs:

```bash
# manual reset عبر الـ debugfs (للـ testing فقط)
echo 1 > /sys/kernel/debug/reset/meson_reset/assert42
echo 0 > /sys/kernel/debug/reset/meson_reset/assert42
```

**رابعًا:** اضمن إن الـ watchdog reset بيعمل full SoC reset:

```bash
# في u-boot أو الـ bootloader
# تأكد إن RESET_TYPE = full system reset مش CPU-only
```

#### الدرس المستفاد
الـ `reset-meson.c` بيوفر الـ infrastructure الصح — `meson8b_param` بيدعم 256 reset line والـ level registers في `0x7c` قادرة تعمل clean hardware reset. المشكلة دايمًا في الـ consumer drivers اللي مش بتستغل الـ reset controller في الـ probe. على 24/7 products، كل driver لازم يعمل hardware reset في الـ probe عشان يضمن clean state بعد أي watchdog event.
## Phase 7: مصادر ومراجع

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **Reset controller API** | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | التوثيق الرسمي لـ consumer و driver API |
| **reset.rst** | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | المصدر الخام بصيغة RST |
| **DT Bindings: reset** | [kernel.org/doc/Documentation/devicetree/bindings/reset/reset.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/reset/reset.txt) | كيفية وصف reset controllers في Device Tree |
| **DT Bindings Directory** | [github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/reset](https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/reset) | كل ملفات bindings الخاصة بالـ reset |

الـ `Documentation/` paths المهمة جوّا الـ kernel source:

```
Documentation/driver-api/reset.rst
Documentation/devicetree/bindings/reset/reset.txt
Documentation/devicetree/bindings/reset/amlogic,meson-reset.yaml
include/linux/reset-controller.h
include/linux/reset.h
drivers/reset/
```

---

### مقالات LWN.net

#### **reset: Add generic GPIO reset driver**
- **URL**: [lwn.net/Articles/585145/](https://lwn.net/Articles/585145/)
- الـ Philipp Zabel أضاف دعم GPIO كـ reset source جوّا الـ reset controller framework — نفس البنية اللي بيستخدمها driver الـ Meson.

#### **Introduction of STM32MP13 RCC driver (Reset Clock Controller)**
- **URL**: [lwn.net/Articles/888111/](https://lwn.net/Articles/888111/)
- مثال عملي على driver بيجمع reset و clock controller في نفس الوقت — مفيد لفهم التصميم المشترك.

#### **clk: en7523: reset-controller support for EN7523 SoC**
- **URL**: [lwn.net/Articles/1039543/](https://lwn.net/Articles/1039543/)
- مثال حديث على إضافة reset-controller support لـ SoC جديد، نفس الأسلوب اللي بيتبعه `reset-meson.c`.

#### **regmap: Generic I2C and SPI register map library**
- **URL**: [lwn.net/Articles/451789/](https://lwn.net/Articles/451789/)
- الـ `reset-meson.c` بيستخدم `devm_regmap_init_mmio` — الفهم الصح للـ regmap ضروري هنا.

#### **PCI: Expose and manage PCI device reset**
- **URL**: [lwn.net/Articles/866578/](https://lwn.net/Articles/866578/)
- مقال عن تعامل الـ kernel مع reset signals بوجه عام من منظور مختلف.

---

### مناقشات الـ Mailing List

#### إضافة الـ reset controller API الأصلية — Philipp Zabel
- **Patchwork**: [patchwork.kernel.org — reset: Add reset controller API (v4)](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/)
- الـ patch الأصلي اللي عرّف `struct reset_controller_dev` و `struct reset_control_ops`.

#### إضافة GPIO support للـ reset framework
- **Patchwork**: [patchwork.kernel.org — reset: Add GPIO support](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/)

#### إضافة دعم Meson-A1 SoC
- **Patchwork**: [patchwork.kernel.org — reset: add Meson-A1 support (v3)](https://patchwork.kernel.org/project/linux-amlogic/cover/1569738255-3941-1-git-send-email-xingyu.chen@amlogic.com/)
- الـ patch اللي عرّف `struct meson_reset_param` الموجود في الكود الحالي.

#### إضافة level reset لـ GX SoC family
- **Patchwork**: [patchwork.kernel.org — reset: meson: add level reset support for GX](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1508235558-24860-1-git-send-email-narmstrong@baylibre.com/)
- نارمسترونج أضاف `level_offset` و `level_low_reset` للـ Meson8b/GXBB.

#### GIT PULL — Reset controller fixes v6.2
- **lore.kernel.org**: [lore.kernel.org/all/20230104095855](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/)
- تصحيحات مهمة على الـ framework من المينتينر.

---

### Commits مهمة في الـ Kernel

الـ commits دي أسّست وطوّرت `reset-meson.c`:

```bash
# ابحث عن تاريخ الـ driver كامل
git log --oneline -- drivers/reset/reset-meson.c
git log --oneline -- drivers/reset/amlogic/

# الـ commit اللي عرّف الـ reset controller framework
git log --oneline --all | grep -i "reset: add reset controller"

# كل اللي خص amlogic reset
git log --oneline --all --grep="amlogic.*reset\|reset.*meson" --regexp-ignore-case
```

الـ commits الجوهرية حسب التاريخ:

| الـ Commit | الوصف |
|------------|-------|
| `a0236527` (تقريبي) | إضافة `reset-meson.c` الأصلي من Carlo Caione |
| patch من 2017-10 | إضافة level reset لـ GX — Neil Armstrong |
| patch من 2019-09 | إضافة `meson_reset_param` لـ Meson-A1 — Xingyu Chen |
| إعادة هيكلة حديثة | نقل الكود لـ `drivers/reset/amlogic/` وفصل الـ common code |

---

### مصادر خاصة بـ BayLibre وأمالوجيك

- **BayLibre Blog**: [baylibre.com/improved-amlogic-support-mainline-linux](https://baylibre.com/improved-amlogic-support-mainline-linux/)
  قصة إضافة Amlogic Meson لـ mainline من وجهة نظر BayLibre.

- **Linux for Amlogic Meson**: [linux-meson.com](https://linux-meson.com/)
  توثيق مجتمعي لتقدم الـ mainlining لكل SoC Amlogic.

- **Embedded Recipes 2017 — Neil Armstrong**: [people.freedesktop.org/~narmstrong/erecipes-2017-amlogic](https://people.freedesktop.org/~narmstrong/erecipes-2017-amlogic/)
  عرض تقديمي عن Amlogic Meson في mainline Linux.

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- المؤلفون: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **مجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- الفصول المهمة:
  - Chapter 3: Char Drivers — فهم `platform_driver`
  - Chapter 9: Communicating with Hardware — `ioremap`, `readl/writel`
  - Chapter 14: The Linux Device Model — `device_get_match_data`, devres

#### Linux Kernel Development — Robert Love (3rd Edition)
- Chapter 17: Devices and Modules — كيف بيشتغل `module_platform_driver`
- Chapter 18: Debugging — فهم `dev_err_probe`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- Chapter 15: Kernel Initialization — فهم الـ platform و DT boot flow
- Chapter 16: Kernel Build System — كيف بيتبنى الـ driver

#### Linux Device Driver Development — John Madieu (Packt, 2nd Edition)
- Chapter 2: Regmap API — شرح مفصل لـ `devm_regmap_init_mmio`
- Chapter 12: Reset controller framework

---

### مصطلحات البحث

لو عاوز تلاقي أكتر، استخدم المصطلحات دي:

```
# بحث عام
linux kernel reset controller driver
linux reset_controller_dev ops
linux devm_reset_controller_register

# بحث خاص بـ Amlogic
amlogic meson reset controller linux kernel
reset-meson.c BayLibre Neil Armstrong
amlogic meson8b gxbb axg reset driver

# بحث في الـ mailing lists
lore.kernel.org reset controller amlogic
patchwork linux-amlogic reset meson

# بحث في الـ kernel source
# على Elixir Cross Referencer:
# https://elixir.bootlin.com/linux/latest/source/drivers/reset
```

**الـ Elixir Cross Referencer** مفيد جداً للتنقل في الكود:
- [elixir.bootlin.com/linux/latest/source/drivers/reset/amlogic](https://elixir.bootlin.com/linux/latest/source/drivers/reset/amlogic)
- [elixir.bootlin.com/linux/latest/source/include/linux/reset-controller.h](https://elixir.bootlin.com/linux/latest/source/include/linux/reset-controller.h)

---

### ملخص المراجع الأهم

```
الأولوية الأولى  → docs.kernel.org/driver-api/reset.html
الأولوية الثانية → lwn.net/Articles/585145/ (GPIO reset driver)
الأولوية الثالثة → patchwork: meson_reset_param patch (Meson-A1)
الأولوية الرابعة → lwn.net/Articles/451789/ (regmap)
الأولوية الخامسة → linux-meson.com (Amlogic mainlining)
```
## Phase 8: Writing simple module

### الفكرة

**`meson_reset_probe`** هي الدالة اللي بتتشغل لما kernel يلاقي device بيطابق أي compatible string في جدول `meson_reset_dt_ids`. هنستخدم **kprobe** نعلقه على `meson_reset_probe` عشان نطبع اسم الـ device اللي اتـ probe وعدد الـ resets المتاحة — ده مفيد لو عايز تتأكد إن الـ reset controller اتسجل بالشكل الصح.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: hook meson_reset_probe to log device name and reset count
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/platform_device.h>

/* kprobe pre-handler: called just before meson_reset_probe executes */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86_64: first arg (struct platform_device *pdev) is in rdi.
     * On ARM64:  first arg is in x0.
     * kprobes gives us pt_regs — we pull the first argument from it.
     */
#if defined(CONFIG_X86_64)
    struct platform_device *pdev = (struct platform_device *)regs->di;
#elif defined(CONFIG_ARM64)
    struct platform_device *pdev = (struct platform_device *)regs->regs[0];
#else
    /* fallback: skip logging on unsupported arch */
    pr_info("meson_reset_probe: kprobe hit (arch not supported for arg extraction)\n");
    return 0;
#endif

    /* guard against null — should never happen but be safe */
    if (!pdev) {
        pr_info("meson_reset_probe: pdev is NULL\n");
        return 0;
    }

    pr_info("meson_reset_probe: probing device [%s] (id=%d)\n",
            pdev->name ? pdev->name : "<noname>",
            pdev->id);

    return 0; /* 0 = let the original function run normally */
}

/* the kprobe descriptor */
static struct kprobe kp = {
    .symbol_name = "meson_reset_probe",
    .pre_handler = handler_pre,
};

static int __init meson_reset_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("meson_reset_kprobe: failed to register kprobe: %d\n", ret);
        return ret;
    }

    pr_info("meson_reset_kprobe: kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit meson_reset_kprobe_exit(void)
{
    /* must unregister before module unloads — otherwise kernel crashes */
    unregister_kprobe(&kp);
    pr_info("meson_reset_kprobe: kprobe removed\n");
}

module_init(meson_reset_kprobe_init);
module_exit(meson_reset_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on meson_reset_probe to log Amlogic reset controller probing");
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/platform_device.h>
```

**`kprobes.h`** بيجيب تعريف `struct kprobe` و `register_kprobe` / `unregister_kprobe`.
**`platform_device.h`** محتاجينه عشان نـ cast الـ argument الأول من `pt_regs` لـ `struct platform_device *` بشكل صحيح.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**`pt_regs`** بيحتوي على حالة الـ registers لحظة وصول الـ kprobe — منه بنستخرج الـ argument الأول للدالة اللي اتـ hook عليها.
الـ `#ifdef` ضروري لأن اسم الـ register بيختلف بين architectures (مثلاً `regs->di` على x86_64 مقابل `regs->regs[0]` على ARM64).

---

#### استخراج `pdev->name`

```c
pr_info("meson_reset_probe: probing device [%s] (id=%d)\n",
        pdev->name ? pdev->name : "<noname>", pdev->id);
```

**`pdev->name`** بيكون اسم الـ driver زي `"meson_reset"`، و**`pdev->id`** بيكون الـ instance number.
ده بيديك confirmation إن الـ reset controller بدأ probe وإيه الـ device اللي اتعمله.

---

#### `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "meson_reset_probe",
    .pre_handler = handler_pre,
};
```

**`.symbol_name`** بيخلي kernel يحل العنوان تلقائياً من الـ kallsyms بدل ما نحدد عنوان يدوي.
**`.pre_handler`** بيتشغل *قبل* الدالة الأصلية — مناسب لو بس عايزين نراقب بدون تغيير السلوك.

---

#### `module_init`

```c
ret = register_kprobe(&kp);
```

**`register_kprobe`** بتزرع breakpoint في الـ kernel code للدالة المحددة.
لو رجعت error (مثلاً الدالة `inline` أو مش exported في kallsyms) الـ module بيرجع الـ error ومبيحملش.

---

#### `module_exit`

```c
unregister_kprobe(&kp);
```

**لازم** نشيل الـ kprobe قبل ما الـ module يتـ unload — لو متشلناش، الـ callback هيحاول يشتغل بعد ما الـ code اتشال من الـ memory وده kernel panic فوري.

---

### Makefile للتجميع

```makefile
obj-m += meson_reset_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ module

```bash
# تحميل الـ module
sudo insmod meson_reset_kprobe.ko

# شوف الـ kprobe اتسجل
sudo dmesg | grep meson_reset_kprobe

# لو الـ reset controller اتـ probe (مثلاً بعد modprobe reset-meson)
# هتشوف رسالة زي:
# meson_reset_kprobe: kprobe planted at meson_reset_probe (0xffff...)
# meson_reset_probe: probing device [meson_reset] (id=-1)

# تفريغ الـ module
sudo rmmod meson_reset_kprobe
sudo dmesg | grep meson_reset_kprobe
# meson_reset_kprobe: kprobe removed
```

---

### ملاحظة على الـ CONFIG

الـ kprobes محتاج:

| Config | القيمة |
|--------|--------|
| `CONFIG_KPROBES` | `y` |
| `CONFIG_KALLSYMS` | `y` |
| `CONFIG_KALLSYMS_ALL` | `y` (preferred) |
| `CONFIG_DEBUG_INFO` | `y` (optional but helps) |

لو `meson_reset_probe` كانت `static` وفي kernel مش بيـ export الـ static symbols في kallsyms، الـ `register_kprobe` هترجع `-ENOENT` — والحل إما تضيف `EXPORT_SYMBOL` في الـ driver الأصلي أو تستخدم عنوان يدوي من `/proc/kallsyms`.
