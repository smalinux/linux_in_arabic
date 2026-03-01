## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **Reset Controller Framework** في Linux kernel، وتحديداً الـ driver الخاص بـ **Amlogic Meson SoCs** — وهي شرائح المعالجات اللي بتصنعها شركة Amlogic وبتتلاقى في أجهزة Android TV boxes وبعض منتجات Google (زي Chromecast) وأجهزة embedded.

---

### الصورة الكبيرة — القصة الحقيقية

#### المشكلة اللي بيحلها

تخيل معايا إنك بتشغّل مدينة فيها مصانع كتير. كل مصنع بيشتغل باستمرار. لما مصنع يتعطّل أو يحتاج إعادة تشغيل، محتاج حد يضغط زرار **Reset** عشان يرجع لأول حالة سليمة.

في الـ SoC (System-on-Chip) زي Meson بتاع Amlogic، في جوا الشريحة الواحدة عشرات الـ **IP blocks** — يعني وحدات منفصلة زي: معالج الصوت، وحدة USB، وحدة HDMI، إلخ. كل وحدة محتاجة **reset line** خاصة بيها.

الـ CPU لما يحتاج يعمل reset لوحدة معينة، بيكتب في **register** محدد في الذاكرة — bit واحد بيوافق وحدة واحدة. لكن المشكلة إن كل SoC version ليه layout مختلف في الـ registers دي.

#### الحل — الـ Reset Controller Framework

Linux عمل **abstraction layer** اسمه Reset Controller Framework. الفكرة بسيطة:

```
Driver (USB, Audio, HDMI)
        ↓
  reset_control_assert()   ← Linux API موحّد
        ↓
  Reset Controller Driver  ← هو اللي يعرف السر
        ↓
  Hardware Register        ← كتابة bit في register
```

بدل ما كل driver يعرف عناوين الـ registers بنفسه، بيقول بس "أنا عايز reset للـ block رقم 5" والـ reset controller driver هو اللي يترجم ده لـ register حقيقي.

---

### دور الملف `reset-meson.h`

الملف ده هو الـ **shared header** أو "عقد الاتفاق" بين:
- الـ **core logic** (`reset-meson-common.c`) — اللي فيه الكود المشترك
- الـ **platform drivers** (`reset-meson.c`, `reset-meson-aux.c`) — اللي بتعرف parameters كل SoC

بيعرّف حاجتين أساسيتين:

#### 1. الـ `struct meson_reset_param` — بطاقة هوية الـ SoC

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; /* operations table: reset/assert/deassert/status */
    unsigned int reset_num;    /* كام reset line موجودة في الـ SoC ده */
    unsigned int reset_offset; /* offset الـ register بتاع الـ pulse reset */
    unsigned int level_offset; /* offset الـ register بتاع الـ level reset */
    bool level_low_reset;      /* هل 0 = reset ولا 1 = reset؟ */
};
```

كل SoC version بيملى الـ struct ده بأرقام مختلفة. مثلاً:
- **Meson 8B** و **GXBB** و **AXG**: 256 reset line، level register عند offset `0x7c`
- **Meson A1**: 96 reset line، level register عند offset `0x40`
- **Meson S4** و **C3**: 192 reset line، level register عند offset `0x40`

#### 2. الـ API Function

```c
int meson_reset_controller_register(struct device *dev,
                                    struct regmap *map,
                                    const struct meson_reset_param *param);
```

ده الـ function الوحيد اللي الـ platform drivers بتستخدمه — بيسجّل الـ reset controller مع Linux framework.

#### 3. الـ ops tables

```c
extern const struct reset_control_ops meson_reset_ops;        /* للـ SoCs العادية */
extern const struct reset_control_ops meson_reset_toggle_ops; /* للـ audio blocks */
```

---

### نوعان من الـ Reset

الملف بيدعم نوعين مختلفين من الـ reset:

| النوع | الوصف | الـ ops |
|-------|-------|---------|
| **Pulse Reset** | بتكتب 1 في register → الـ hardware بيعمل reset ويرجع لوحده | `meson_reset_ops` |
| **Level Reset** | بتبقى مسيطر: assert (hold in reset) ثم deassert (release) | `meson_reset_ops` |
| **Toggle Reset** | زي Level لكن الـ reset عبارة عن assert+deassert في sequence | `meson_reset_toggle_ops` |

الـ `level_low_reset = true` معناها إن قيمة 0 في الـ register = الـ block في reset (منطق معكوس).

---

### قصة الـ Auxiliary Bus

في بعض الـ SoCs، وحدة الصوت (Audio) بيبقى ليها reset registers جوا نفس مساحة الـ audio clock controller — مش في register منفصل. عشان كده، بدل ما يعمل platform device منفصل، الـ audio clock driver بيعمل **auxiliary device** يورّث الـ regmap الخاص بيه للـ reset driver عبر `reset-meson-aux.c`.

---

### الملفات اللي تعرفها

```
drivers/reset/amlogic/
├── reset-meson.h              ← الملف ده (الـ shared header / العقد)
├── reset-meson-common.c       ← الـ core logic المشترك (تنفيذ الـ ops)
├── reset-meson.c              ← Platform driver (من DTS عبر platform_device)
├── reset-meson-aux.c          ← Auxiliary driver (للـ audio subsystem)
├── reset-meson-audio-arb.c    ← Reset خاص بـ Audio Memory Arbiter (A113 SoCs)
├── Kconfig                    ← إعدادات الـ build
└── Makefile

include/linux/
├── reset-controller.h         ← تعريف struct reset_controller_dev و reset_control_ops
├── reset.h                    ← Consumer API (reset_control_assert/deassert)
└── regmap.h                   ← Register Map abstraction API

Documentation/
├── driver-api/reset.rst       ← شرح الـ Reset Controller Framework كامل
└── devicetree/bindings/reset/amlogic,meson-reset.yaml  ← DTS binding
```

---

### تدفق العمل من البداية للنهاية

```
Boot
 ↓
DTS: amlogic,meson-gxbb-reset
 ↓
platform_driver match → meson_reset_probe()
 ↓
ioremap() → regmap_init_mmio() → meson_reset_controller_register()
 ↓
devm_reset_controller_register() → Linux Framework يسجّل الـ controller
 ↓
USB driver: reset_control_assert(rst)
 ↓
meson_reset_assert() → regmap_update_bits() → كتابة في register
 ↓
Hardware USB block يدخل reset
```
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

أي SoC فيه عشرات الـ IP blocks — USB controller، HDMI encoder، audio DSP، Ethernet MAC، إلخ. كل block ده عنده **reset line** متصلة بـ reset controller مركزي جوه الـ SoC. لما الـ driver بتاعه يشتغل، محتاج يعمل:

1. **assert**: يحط الـ block في حالة reset (يوقفه أو يصفّيه).
2. **deassert**: يطلق الـ block من الـ reset (يشغّله).
3. **pulse**: assert ثم deassert بشكل سريع — للـ self-deasserting resets.

المشكلة إن كل SoC بيعمل ده بطريقة مختلفة:
- Amlogic Meson: بيكتب على bits في registers MMIO.
- Qualcomm: بيبعت commands على bus خاص.
- Broadcom: له registers منفصلة لكل direction.

لو كل driver يكتب كود الـ reset بنفسه، الـ codebase هيبقى فوضى. محتاج **abstraction layer** يفصل بين "مين عايز reset" و"إزاي الـ hardware بيعمل reset".

---

### الحل — إيه اللي الـ Kernel بيعمله؟

الـ kernel بيقدم الـ **Reset Controller Framework** — subsystem مركزي جوه `drivers/reset/` بيعمل:

- **Provider side**: الـ driver بتاع الـ reset controller (زي Meson) بيسجّل نفسه مع الـ framework ويقوله "أنا عندي N reset lines، وده الـ ops بتاعتهم".
- **Consumer side**: أي driver تاني (USB، GPU، إلخ) يطلب reset line بالـ ID بتاعها من الـ Device Tree، والـ framework يوصّله بالـ provider الصح.

الـ framework نفسه بيعمل: الـ lookup، الـ reference counting، والـ locking. الـ provider بس بيكتب الـ hardware ops.

---

### التشبيه الحقيقي — لوحة الكهرباء في العمارة

تخيل عمارة فيها **لوحة كهرباء مركزية** (الـ reset controller). فيها circuit breakers — كل واحد بيتحكم في شقة أو طابق معين (IP block).

| الـ Analogy | الـ Kernel Concept |
|---|---|
| لوحة الكهرباء المركزية | `reset_controller_dev` |
| كل circuit breaker | reset line بـ ID رقمي |
| الكهربائي اللي يعرف يشغّل/يوقف الـ breaker | `reset_control_ops` (provider ops) |
| الشقة اللي عايزة تقطع التيار عن نفسها | consumer driver |
| دفتر العمارة اللي فيه "شقة 5 = breaker 12" | Device Tree + `of_xlate` |
| شركة إدارة العمارة | Reset Controller Framework نفسه |

الشقة (consumer) ما بتتكلمش مع لوحة الكهرباء مباشرة — بتتكلم مع شركة الإدارة (framework). شركة الإدارة هي اللي بتبعت الكهربائي (ops) يشغّل الـ breaker المناسب.

---

### Big Picture Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │           Consumer Drivers                   │
                    │  (USB, HDMI, Audio, Ethernet drivers...)     │
                    │                                              │
                    │  reset_control_assert(rstc)                  │
                    │  reset_control_deassert(rstc)                │
                    │  reset_control_reset(rstc)                   │
                    └──────────────────┬──────────────────────────┘
                                       │  generic API
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │       Reset Controller Framework             │
                    │         (drivers/reset/core.c)              │
                    │                                             │
                    │  - Lookup by DT phandle + specifier         │
                    │  - Reference counting per reset line        │
                    │  - Shared reset handling                    │
                    │  - of_xlate: specifier → ID                 │
                    └──────────────────┬──────────────────────────┘
                                       │  calls ops->assert/deassert/reset
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │         Reset Controller Provider           │
                    │    (drivers/reset/amlogic/reset-meson.c)   │
                    │                                             │
                    │  reset_controller_dev {                     │
                    │    .ops = &meson_reset_ops                  │
                    │    .nr_resets = 256                         │
                    │    .of_node = <DT node>                     │
                    │  }                                          │
                    │                                             │
                    │  + meson_reset_param {                      │
                    │    .reset_offset = 0x0                      │
                    │    .level_offset = 0x7c                     │
                    │    .level_low_reset = true                  │
                    │  }                                          │
                    └──────────────────┬──────────────────────────┘
                                       │  regmap_write / regmap_update_bits
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │           regmap (MMIO abstraction)         │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │         Amlogic Meson SoC Hardware          │
                    │                                             │
                    │  Reset Registers:  0x00 - 0x1C  (pulse)    │
                    │  Level Registers:  0x7C - 0x9C  (latch)    │
                    │                                             │
                    │  bit N → reset line N                       │
                    └─────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ framework بيبني على فكرة واحدة: **reset line = integer ID**.

كل reset line في الـ system عندها رقم (ID). الـ provider بيقول "أنا عندي reset lines من 0 لـ N-1". الـ consumer بيطلب line بـ ID محدد. الـ framework بيربطهم.

ده بيسمح بـ:
- **Decoupling كامل**: الـ consumer driver لا يعرف إيه SoC موجود.
- **Reuse**: نفس الـ consumer driver يشتغل على أي SoC طالما الـ DT صح.
- **Safety**: الـ framework بيضمن مش اتنين يـassert نفس الـ shared reset line غلط.

---

### الـ Core Structs وعلاقتها ببعض

```
meson_reset  (private driver state)
├── param ──────────────────────► meson_reset_param
│                                 ├── reset_ops ──► reset_control_ops
│                                 │                 ├── .reset   = meson_reset_reset
│                                 │                 ├── .assert  = meson_reset_assert
│                                 │                 ├── .deassert= meson_reset_deassert
│                                 │                 └── .status  = meson_reset_status
│                                 ├── reset_num    (عدد الـ lines)
│                                 ├── reset_offset (base address للـ pulse regs)
│                                 ├── level_offset (base address للـ level regs)
│                                 └── level_low_reset (polarity flag)
│
├── map ────────────────────────► struct regmap  (MMIO abstraction)
│                                 بيخفي ناحية الـ hardware access
│
└── rcdev ──────────────────────► reset_controller_dev
                                  ├── ops ──────► (نفس reset_control_ops فوق)
                                  ├── nr_resets  (مسجّل في الـ framework)
                                  ├── of_node    (الـ DT node)
                                  ├── list       (linked في global list)
                                  └── of_xlate   (default: simple linear mapping)
```

**الـ `container_of` trick**: الـ framework بيدي الـ callbacks بـ pointer على الـ `rcdev`. الـ Meson driver بيعمل `container_of(rcdev, struct meson_reset, rcdev)` عشان يوصل لـ `param` و`map`.

---

### نوعين الـ Reset في Meson Hardware

#### 1. Self-Deasserting Reset (Pulse)

الـ hardware نفسه بيرجع الـ bit لـ 0 أوتوماتيك بعد ما الـ firmware يكتب 1. بيُكتب على `reset_offset` registers.

```c
/* write 1 to bit N → hardware generates pulse → auto-clears */
regmap_write(data->map, offset, BIT(bit));
```

ده بيتعمل في `meson_reset_reset()` — مجرد write ومع السلامة.

#### 2. Level Reset (Latch)

الـ bit بيفضل محتفظ بقيمته لحد ما الـ software يغيّره. الـ registers دي موجودة على `level_offset`.

```c
/* set or clear bit N based on assert/deassert + polarity */
regmap_update_bits(data->map, offset, BIT(bit), assert ? BIT(bit) : 0);
```

الـ polarity flag `level_low_reset` مهم: في Meson، الـ reset active هو **low** (0 = في reset، 1 = طلع من reset). الـ XOR `assert ^= data->param->level_low_reset` بيعكس المعنى:

```
level_low_reset = true:
  assert  → نريد نحط في reset → نكتب 0 → bit = 0 → true XOR true = false → BIT(bit) & 0 = 0 ✓
  deassert → نريد نطلع من reset → نكتب 1 → bit = 1 → false XOR true = true → BIT(bit) ✓
```

#### الـ Toggle Ops — للـ blocks اللي ما عندهاش pulse registers

`meson_reset_toggle_ops` بتعمل الـ pulse يدوياً: assert ثم deassert متتاليين. بيُستخدم لما الـ hardware ما عندوش الـ self-deasserting registers دي.

---

### حساب الـ Offset والـ Bit

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
{
    unsigned int stride = regmap_get_reg_stride(data->map); /* = 4 bytes */

    *offset = (id / (stride * BITS_PER_BYTE)) * stride;
    /* stride=4, BITS_PER_BYTE=8 → كل register يغطي 32 line */
    /* reset line 0-31  → offset=0x00 */
    /* reset line 32-63 → offset=0x04 */
    /* reset line 64-95 → offset=0x08 */

    *bit = id % (stride * BITS_PER_BYTE);
    /* bit position داخل الـ 32-bit register */
}
```

بعدين بيُضاف عليه `reset_offset` أو `level_offset` حسب نوع العملية.

**مثال**: reset line 40 على Meson8b:
- `stride = 4`, `BITS_PER_BYTE = 8` → bits_per_reg = 32
- `offset = (40 / 32) * 4 = 4` → register عند `reset_offset + 0x04`
- `bit = 40 % 32 = 8` → bit 8 جوه الـ register ده

---

### إزاي الـ Driver بيسجّل نفسه

```c
int meson_reset_controller_register(struct device *dev,
                                    struct regmap *map,
                                    const struct meson_reset_param *param)
{
    struct meson_reset *data;

    /* allocate managed (devm) → auto-freed on device removal */
    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);

    data->param = param;   /* hardware description */
    data->map   = map;     /* regmap handle */

    /* fill the framework-visible struct */
    data->rcdev.owner     = dev->driver->owner; /* module ref counting */
    data->rcdev.nr_resets = param->reset_num;   /* total lines */
    data->rcdev.ops       = param->reset_ops;   /* callbacks */
    data->rcdev.of_node   = dev->of_node;       /* DT matching */

    /* register with the framework — devm handles unregister on remove */
    return devm_reset_controller_register(dev, &data->rcdev);
}
```

بعد الـ register ده، الـ framework بيضيف `rcdev` في global list. أي consumer يطلب reset من الـ DT node ده، الـ framework بيلاقيه في الـ list ويستخدم الـ ops.

---

### الـ regmap Subsystem — لازم تعرفه الأول

**الـ regmap** هو abstraction layer للـ register access. بدل ما كل driver يكتب `readl()`/`writel()` مباشرة، الـ regmap بيوفر:
- **Unified API**: `regmap_read()`, `regmap_write()`, `regmap_update_bits()`.
- **Caching**: لو الـ hardware بطيء.
- **Backends**: MMIO, I2C, SPI — نفس الـ API.

في Meson، الـ regmap configured كـ MMIO مع `reg_bits=32, val_bits=32, reg_stride=4` — يعني registers متسلسلة كل 4 bytes.

---

### إيه اللي الـ Framework بيمتلكه vs إيه اللي بيفوّضه للـ Driver

| المسؤولية | مين بيعملها |
|---|---|
| Lookup الـ reset line من الـ DT specifier | **Framework** (`of_xlate`) |
| Reference counting للـ shared resets | **Framework** |
| Locking لمنع race conditions | **Framework** |
| Assert / Deassert / Pulse فعلياً على الـ hardware | **Provider Driver** (Meson) |
| حساب الـ register address والـ bit | **Provider Driver** (Meson) |
| معرفة الـ polarity (active-low/high) | **Provider Driver** (عبر `meson_reset_param`) |
| الـ register access (read/write) | **regmap** (مُفوَّض من الـ driver) |
| Device Tree matching والـ probe | **Platform Driver** infrastructure |

---

### الـ `EXPORT_SYMBOL_NS_GPL` — الـ Module Namespace

```c
EXPORT_SYMBOL_NS_GPL(meson_reset_ops, "MESON_RESET");
EXPORT_SYMBOL_NS_GPL(meson_reset_controller_register, "MESON_RESET");
```

الـ `_NS_` معناها الـ symbol ده جوه **namespace** اسمه `"MESON_RESET"`. أي module تاني عايز يستخدمه لازم يعلن:

```c
MODULE_IMPORT_NS("MESON_RESET");
```

ده بيحمي من الـ accidental use — بيخلي الـ coupling بين modules صريح وواضح في الـ code.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Config Options — Cheatsheet

| Field / Option | Type | القيمة / المعنى |
|---|---|---|
| `level_low_reset` | `bool` | `true` = reset active-low (assert = كتابة 0)، `false` = active-high |
| `reset_offset` | `unsigned int` | offset بايت لـ self-clearing reset registers |
| `level_offset` | `unsigned int` | offset بايت لـ level-sensitive assert/deassert registers |
| `reset_num` | `unsigned int` | عدد reset lines المتاحة في الـ controller |
| `reg_bits` | 32 | عرض الـ register address |
| `val_bits` | 32 | عرض الـ register value |
| `reg_stride` | 4 | المسافة بين كل register بالبايت (32-bit word) |

#### الـ compatible strings المدعومة (Device Tree)

| Compatible String | Param struct | reset_num | level_offset | level_low_reset |
|---|---|---|---|---|
| `amlogic,meson8b-reset` | `meson8b_param` | 256 | 0x7c | true |
| `amlogic,meson-gxbb-reset` | `meson8b_param` | 256 | 0x7c | true |
| `amlogic,meson-axg-reset` | `meson8b_param` | 256 | 0x7c | true |
| `amlogic,meson-a1-reset` | `meson_a1_param` | 96 | 0x40 | true |
| `amlogic,meson-s4-reset` | `meson_s4_param` | 192 | 0x40 | true |
| `amlogic,c3-reset` | `meson_s4_param` | 192 | 0x40 | true |
| `amlogic,t7-reset` | `t7_param` | 224 | 0x40 | true |

---

### الـ Structs المهمة

#### 1. `struct meson_reset_param` — الـ Header File

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; /* مؤشر لجدول العمليات */
    unsigned int reset_num;    /* عدد reset lines */
    unsigned int reset_offset; /* offset لـ pulse-reset registers */
    unsigned int level_offset; /* offset لـ level registers */
    bool level_low_reset;      /* polarity: true = active-low */
};
```

**الغرض:** يحمل الـ configuration الثابتة (compile-time) الخاصة بكل SoC variant. بيتعمل كـ `const` ومش بيتغير بعد التهيئة.

**الحقول:**
- **`reset_ops`**: بيشير لـ vtable الخاص بالـ controller — إما `meson_reset_ops` (pulse-reset) أو `meson_reset_toggle_ops` (toggle-reset).
- **`reset_num`**: بيحدد `nr_resets` في الـ `reset_controller_dev` — الـ subsystem بيستخدمها للـ bounds checking.
- **`reset_offset`**: الـ base address جوه الـ regmap لمنطقة الـ self-clearing reset bits.
- **`level_offset`**: الـ base address لمنطقة الـ level (assert/deassert) bits.
- **`level_low_reset`**: بيقلب الـ polarity — لما `true`، assert = كتابة 0، deassert = كتابة 1.

---

#### 2. `struct meson_reset` — الـ Driver Private Data

```c
struct meson_reset {
    const struct meson_reset_param *param; /* SoC-specific config */
    struct reset_controller_dev rcdev;     /* kernel reset subsystem handle */
    struct regmap *map;                    /* regmap for MMIO access */
};
```

**الغرض:** الـ runtime state للـ driver. بيتعمل بـ `devm_kzalloc` في `meson_reset_controller_register`. الـ `rcdev` embedded جواه مش pointer.

**الحقول:**
- **`param`**: pointer لـ `meson_reset_param` الثابت — read-only بعد التهيئة.
- **`rcdev`**: الـ kernel reset subsystem object — الـ core بيتعامل معاه مباشرةً.
- **`map`**: الـ regmap object — بيوفر abstraction فوق MMIO registers.

---

#### 3. `struct reset_control_ops` — الـ vtable

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

**الغرض:** الـ function pointer table اللي الـ reset subsystem بيستدعيه. الـ driver بيوفر نسختين:

| Instance | `reset` | `assert` | `deassert` | `status` |
|---|---|---|---|---|
| `meson_reset_ops` | `meson_reset_reset` (pulse) | `meson_reset_assert` | `meson_reset_deassert` | `meson_reset_status` |
| `meson_reset_toggle_ops` | `meson_reset_level_toggle` | `meson_reset_assert` | `meson_reset_deassert` | `meson_reset_status` |

---

#### 4. `struct reset_controller_dev` — الـ Kernel Reset Core

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;  /* vtable */
    struct module *owner;                  /* الـ module المالك */
    struct list_head list;                 /* في قائمة الـ controllers */
    struct list_head reset_control_head;   /* الـ controls المطلوبة */
    struct device *dev;                    /* الـ device المرتبط */
    struct device_node *of_node;           /* نود الـ device tree */
    int of_reset_n_cells;                  /* عدد cells في الـ specifier */
    int (*of_xlate)(...);                  /* ترجمة DT specifier → id */
    unsigned int nr_resets;               /* عدد الـ reset lines */
};
```

الـ driver بيملا: `ops`، `owner`، `nr_resets`، `of_node` — والباقي بيتملاه `devm_reset_controller_register`.

---

### مخطط علاقات الـ Structs

```
  ┌─────────────────────────────────────────┐
  │         struct meson_reset              │
  │  (driver private, devm allocated)       │
  │                                         │
  │  +param ──────────────────────────────► │ struct meson_reset_param (const)
  │                                         │   .reset_ops ──────────────────► struct reset_control_ops
  │  +map ────────────────────────────────► │ struct regmap
  │                                         │   (MMIO abstraction)
  │  +rcdev [embedded]                      │
  │  │  struct reset_controller_dev         │
  │  │    .ops ───────────────────────────► │ struct reset_control_ops
  │  │    .dev ───────────────────────────► │ struct device
  │  │    .of_node ───────────────────────► │ struct device_node
  │  │    .list ──────────────────────────► │ global reset_controller_list
  │  └─────────────────────────────────────┘
  └─────────────────────────────────────────┘

  container_of(rcdev, struct meson_reset, rcdev)
  ◄──────────────────────────────────────────────
  (الـ ops functions بتعمل كده عشان توصل للـ param والـ map)
```

---

### مخطط دورة حياة الـ Controller

```
  [Device Tree]                [Kernel]
       │
       │  compatible = "amlogic,meson8b-reset"
       │
       ▼
  platform_driver_probe()
       │
       ├─► devm_platform_ioremap_resource()   → void __iomem *base
       │
       ├─► device_get_match_data()            → const meson_reset_param *param
       │
       ├─► devm_regmap_init_mmio()            → struct regmap *map
       │
       └─► meson_reset_controller_register()
                │
                ├─► devm_kzalloc()            → struct meson_reset *data
                │
                ├─► data->param  = param
                ├─► data->map    = map
                ├─► data->rcdev.owner     = dev->driver->owner
                ├─► data->rcdev.nr_resets = param->reset_num
                ├─► data->rcdev.ops       = param->reset_ops
                ├─► data->rcdev.of_node   = dev->of_node
                │
                └─► devm_reset_controller_register()
                         │
                         └─► يضيف rcdev لـ global list
                             ويبدأ يقبل reset requests

  ──────────────── RUNTIME ─────────────────

  consumer calls reset_control_assert(rstc)
       │
       └─► reset_controller_dev.ops->assert(rcdev, id)
                │
                └─► meson_reset_assert() / meson_reset_deassert()
                         │
                         └─► regmap_update_bits() → MMIO write

  ──────────────── TEARDOWN ────────────────

  device_remove() أو driver_unload()
       │
       └─► devm cleanup (automatic)
                ├─► reset_controller_unregister()   (removes from list)
                ├─► regmap freed
                └─► kfree(data)
```

---

### مخطط تدفق الـ Calls

#### حالة: `reset_control_assert(rstc)`

```
  consumer driver
    └─► reset_control_assert(rstc)                  [reset/core.c]
          └─► rcdev->ops->assert(rcdev, id)
                └─► meson_reset_assert(rcdev, id)   [reset-meson-common.c]
                      └─► meson_reset_level(rcdev, id, true)
                            │
                            ├─► container_of(rcdev, struct meson_reset, rcdev)
                            │     → data
                            │
                            ├─► meson_reset_offset_and_bit(data, id, &offset, &bit)
                            │     stride = regmap_get_reg_stride(data->map)
                            │     offset = (id / (stride * 8)) * stride
                            │     bit    = id % (stride * 8)
                            │
                            ├─► offset += data->param->level_offset
                            │
                            ├─► assert ^= data->param->level_low_reset
                            │     (يقلب القيمة لو active-low)
                            │
                            └─► regmap_update_bits(map, offset, BIT(bit), val)
                                  └─► MMIO write to hardware register
```

#### حالة: `reset_control_reset(rstc)` — الـ pulse reset

```
  consumer driver
    └─► reset_control_reset(rstc)
          └─► rcdev->ops->reset(rcdev, id)
                └─► meson_reset_reset(rcdev, id)
                      │
                      ├─► meson_reset_offset_and_bit() → offset, bit
                      ├─► offset += data->param->reset_offset
                      └─► regmap_write(map, offset, BIT(bit))
                            └─► كتابة bit واحد بس → HW يعمل self-clearing pulse
```

#### حالة: `meson_reset_toggle_ops` — الـ toggle reset

```
  consumer driver
    └─► reset_control_reset(rstc)
          └─► rcdev->ops->reset → meson_reset_level_toggle(rcdev, id)
                │
                ├─► meson_reset_assert(rcdev, id)    → level write (assert)
                │     [waits for return]
                └─► meson_reset_deassert(rcdev, id)  → level write (deassert)
```

#### حالة: `reset_control_status(rstc)`

```
  consumer driver
    └─► reset_control_status(rstc)
          └─► rcdev->ops->status → meson_reset_status(rcdev, id)
                │
                ├─► meson_reset_offset_and_bit() → offset, bit
                ├─► offset += data->param->level_offset
                ├─► regmap_read(map, offset, &val)
                ├─► val = !!(BIT(bit) & val)         → 0 or 1
                └─► return val ^ data->param->level_low_reset
                      (يصحح الـ polarity للـ caller)
```

---

### استراتيجية الـ Locking

الـ driver ده **لا يحتوي على أي lock صريح** على مستوى الـ driver — وده تصميم مقصود:

| المستوى | الـ Lock | المسؤول |
|---|---|---|
| **Regmap** | internal spinlock/mutex داخل `regmap_update_bits` و `regmap_write` | الـ regmap core |
| **Reset Controller List** | `reset_list_mutex` في `reset/core.c` | الـ reset core |
| **`reset_control_head`** | محمي بـ `reset_list_mutex` | الـ reset core |

**الـ `regmap_update_bits`** — الأهم — بيعمل read-modify-write بشكل atomic داخلياً. الـ MMIO regmap بيستخدم spinlock عشان الـ reg_bits=32/val_bits=32 reads/writes تكون consistent.

**ترتيب الـ Locks (Lock Ordering):**

```
  reset_list_mutex          (reset core، global)
       ↓
  regmap internal lock      (per-regmap، local)
       ↓
  hardware register access  (MMIO، atomic)
```

مفيش حالة deadlock ممكنة لأن الـ lock hierarchy اتجاهه واحد دايماً ومفيش lock عكسي في الـ driver code نفسه.

**ملاحظة مهمة على `level_low_reset`:** الـ XOR operation في `meson_reset_level` و`meson_reset_status` هي purely computation — بتتعمل قبل أي hardware access ومش محتاجة protection.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs

| Function / Symbol | النوع | الملف | الغرض |
|---|---|---|---|
| `meson_reset_controller_register` | exported API | reset-meson-common.c | تسجيل الـ reset controller مع الـ kernel |
| `meson_reset_offset_and_bit` | static helper | reset-meson-common.c | حساب الـ register offset والـ bit index |
| `meson_reset_reset` | ops callback | reset-meson-common.c | تنفيذ self-deasserting reset |
| `meson_reset_level` | static helper | reset-meson-common.c | ضبط مستوى الـ reset line (assert/deassert) |
| `meson_reset_assert` | ops callback | reset-meson-common.c | assert الـ reset line |
| `meson_reset_deassert` | ops callback | reset-meson-common.c | deassert الـ reset line |
| `meson_reset_level_toggle` | ops callback | reset-meson-common.c | assert ثم deassert (toggle reset) |
| `meson_reset_status` | ops callback | reset-meson-common.c | قراءة حالة الـ reset line |
| `meson_reset_ops` | exported symbol | reset-meson-common.c | ops table للـ hardware reset pulse |
| `meson_reset_toggle_ops` | exported symbol | reset-meson-common.c | ops table للـ level-toggle reset |
| `meson_reset_probe` | platform driver probe | reset-meson.c | platform probe — init الـ regmap وتسجيل الـ controller |

---

### المجموعة الأولى: Registration APIs

الهدف من هذه المجموعة هو ربط الـ driver بالـ reset controller subsystem. بعد ما الـ probe يجيب الـ `regmap` والـ `param`، بيبعتهم لـ `meson_reset_controller_register` اللي بتبني الـ `meson_reset` struct وتسجله مع الـ kernel عن طريق `devm_reset_controller_register`.

---

#### `meson_reset_controller_register`

```c
int meson_reset_controller_register(struct device *dev,
                                    struct regmap *map,
                                    const struct meson_reset_param *param);
```

**بتعمل إيه:**
بتعمل `devm_kzalloc` لـ `struct meson_reset`، بتملأ الـ `rcdev` fields (owner, nr_resets, ops, of_node)، وبعدين بتفوّض لـ `devm_reset_controller_register` اللي بتضيف الـ controller للـ global list وبتربطه بالـ device tree node عشان أي consumer يلاقيه عن طريق `of_phandle`.

**Parameters:**
- `dev` — الـ `struct device*` بتاع الـ platform device، بيُستخدم للـ devm allocations وجيب الـ `of_node` والـ `driver->owner`.
- `map` — الـ `struct regmap*` المُعمل init عليه على الـ MMIO region بتاع الـ reset registers.
- `param` — `const struct meson_reset_param*` بيحتوي على الـ SoC-specific configuration (عدد الـ resets، الـ offsets، الـ polarity).

**Return value:**
`0` لو النجاح، أو error code سالب من `devm_kzalloc` أو `devm_reset_controller_register`.

**Key details:**
- الـ allocation بـ `devm_*` يعني automatically يتحرر لو الـ driver اتعمله unbind — مفيش cleanup صريح مطلوب.
- الـ `rcdev.owner` بياخد `dev->driver->owner` عشان الـ module reference counting يشتغل صح.
- الـ `EXPORT_SYMBOL_NS_GPL` مع namespace `"MESON_RESET"` يعني أي driver تاني عايز يستخدمها لازم يعمل `MODULE_IMPORT_NS("MESON_RESET")`.

**Pseudocode:**
```
meson_reset_controller_register(dev, map, param):
    data = devm_kzalloc(dev, sizeof(meson_reset))
    if !data → return -ENOMEM

    data->param  = param
    data->map    = map
    data->rcdev.owner     = dev->driver->owner
    data->rcdev.nr_resets = param->reset_num
    data->rcdev.ops       = param->reset_ops
    data->rcdev.of_node   = dev->of_node

    return devm_reset_controller_register(dev, &data->rcdev)
```

**من بيناديها:** `meson_reset_probe` في `reset-meson.c`، وكمان auxiliary drivers زي `reset-meson-aux.c`.

---

#### `meson_reset_probe`

```c
static int meson_reset_probe(struct platform_device *pdev);
```

**بتعمل إيه:**
الـ platform driver probe. بتعمل `ioremap` للـ MMIO resource الأولانية، بتجيب الـ `meson_reset_param` الـ SoC-specific من الـ device tree match data، بتعمل `regmap` فوق الـ MMIO، وبعدين بتفوّض كل حاجة لـ `meson_reset_controller_register`.

**Parameters:**
- `pdev` — الـ `struct platform_device*` اللي الـ kernel بيبعته لما يلاقي compatible match في الـ device tree.

**Return value:**
`0` لو النجاح، أو `PTR_ERR` من `devm_platform_ioremap_resource` / `devm_regmap_init_mmio`، أو `-ENODEV` لو مفيش match data.

**Key details:**
- الـ `regmap_config` بيستخدم `reg_bits=32`, `val_bits=32`, `reg_stride=4` — يعني كل register عرضه 32-bit وعنوانه aligned على 4 bytes.
- الـ `device_get_match_data` بتجيب الـ `data` pointer من الـ `of_device_id` entry المناسبة.
- كل الـ error paths بيستخدموا `dev_err_probe` أو direct return لـ `PTR_ERR` — مفيش manual cleanup.

---

### المجموعة الثانية: Helper Functions

دي الـ internal helpers اللي بتحسب الـ register address والـ bit position بناءً على الـ reset ID.

---

#### `meson_reset_offset_and_bit`

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit);
```

**بتعمل إيه:**
بتحول الـ linear reset `id` لـ register `offset` (بالـ bytes) وبت index (`bit`) جوا الـ register ده. الـ stride بتاخده من الـ regmap عشان يكون متوافق مع الـ bus width (عادةً 4 bytes = 32 bit registers).

**Parameters:**
- `data` — الـ `struct meson_reset*` اللي فيه الـ `regmap` وبالتالي الـ stride.
- `id` — الـ reset line index (linear، من 0 لـ `reset_num - 1`).
- `offset` — الـ output: byte offset من الـ base address للـ register المناسب (قبل ما يتضاف الـ `reset_offset` أو `level_offset`).
- `bit` — الـ output: bit index جوا الـ register.

**Return value:** void — بتكتب في `*offset` و `*bit`.

**الحسابات:**
```
stride     = regmap_get_reg_stride(map)   // = 4 bytes
bits_per_reg = stride * 8                 // = 32
offset = (id / bits_per_reg) * stride    // register number × bytes per reg
bit    = id % bits_per_reg               // bit position inside register
```

مثلاً: `id=40`, `stride=4`, `bits_per_reg=32`:
- `offset = (40/32)*4 = 4` → الـ register التاني (byte offset 4)
- `bit    = 40%32 = 8`    → bit 8 جوا الـ register ده

---

#### `meson_reset_level`

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert);
```

**بتعمل إيه:**
الـ core function اللي بتعمل assert أو deassert للـ reset line عن طريق الـ level register. بتـ XOR الـ `assert` مع `level_low_reset` عشان تتعامل مع الـ hardware polarity — لو الـ hardware reset active-low، الـ assert الحقيقي يعني write 0 مش 1.

**Parameters:**
- `rcdev` — الـ `struct reset_controller_dev*`، بيُستخدم مع `container_of` عشان يجيب الـ `struct meson_reset*`.
- `id` — الـ reset line index.
- `assert` — `true` = assert الـ reset، `false` = deassert.

**Return value:** نفس return value بتاع `regmap_update_bits`.

**Key details — Polarity:**
```c
assert ^= data->param->level_low_reset;
// level_low_reset=true  → active-low hardware
// assert=true  XOR true  → false → write 0 to bit  (pull line low = assert)
// assert=false XOR true  → true  → write 1 to bit  (pull line high = deassert)
```
الـ XOR ده elegant — بيتجنب الـ if/else ويتعامل مع الـ polarity inversion في سطر واحد.

**Locking:** الـ `regmap_update_bits` internally بيستخدم spin_lock بتاع الـ regmap، فمفيش locking إضافي محتاج في هذه المستوى.

---

### المجموعة التالتة: الـ Reset Control Ops Callbacks

دي الـ functions اللي بتتسجل في الـ `reset_control_ops` struct وبيناديها الـ reset controller core لما أي consumer يطلب reset operation.

---

#### `meson_reset_reset`

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                             unsigned long id);
```

**بتعمل إيه:**
بتنفذ **self-deasserting reset** — بتكتب `1` في الـ bit المناسب في الـ reset register. الـ hardware عنده dedicated reset-pulse registers (مختلفة عن الـ level registers) بتعمل reset تلقائي ومبتحتاجش deassert صريح.

**Parameters:**
- `rcdev` — الـ reset controller device.
- `id` — الـ reset line index.

**Return value:** نتيجة `regmap_write`.

**Key details:**
- بيكتب على الـ `reset_offset` (مش `level_offset`) — ده registers خاصة بالـ pulse reset.
- `regmap_write` بيكتب الـ 32-bit value كاملاً، مش bit-masked — يعني ممكن يعمل reset لـ bits تانية في نفس الـ register لو اتكتبت بـ non-zero value. لكن في العادة الـ consumer بيطلب reset لـ bit واحد بس، والـ hardware بيتعامل مع الكتابة دي كـ pulse.

**من بيناديه:** الـ reset controller core لما consumer يعمل `reset_control_reset()`.

---

#### `meson_reset_assert`

```c
static int meson_reset_assert(struct reset_controller_dev *rcdev,
                              unsigned long id);
```

**بتعمل إيه:**
Wrapper بسيط بيستدعي `meson_reset_level(rcdev, id, true)` — بيعمل assert للـ reset line وبيخليها في حالة reset.

**Parameters:** نفس `meson_reset_reset`.

**Return value:** نتيجة `meson_reset_level` → نتيجة `regmap_update_bits`.

**من بيناديه:** الـ reset core لما consumer يعمل `reset_control_assert()`، وكمان `meson_reset_level_toggle`.

---

#### `meson_reset_deassert`

```c
static int meson_reset_deassert(struct reset_controller_dev *rcdev,
                                unsigned long id);
```

**بتعمل إيه:**
Wrapper بيستدعي `meson_reset_level(rcdev, id, false)` — بيطلق الـ reset line ويخلي الـ peripheral يشتغل.

**Parameters:** نفس `meson_reset_assert`.

**Return value:** نتيجة `meson_reset_level`.

**من بيناديه:** الـ reset core لما consumer يعمل `reset_control_deassert()`، وكمان `meson_reset_level_toggle`.

---

#### `meson_reset_level_toggle`

```c
static int meson_reset_level_toggle(struct reset_controller_dev *rcdev,
                                    unsigned long id);
```

**بتعمل إيه:**
بتعمل reset عن طريق الـ level registers بدل الـ pulse registers — بتعمل `assert` الأول وبعدين `deassert`. ده البديل اللي بيتحط في `meson_reset_toggle_ops.reset` للـ SoCs اللي مبتحبش الـ direct pulse write (زي T7 اللي مليش `reset_ops` في الـ param).

**Parameters:** نفس `meson_reset_reset`.

**Return value:** `0` لو النجاح، أو error من `meson_reset_assert` لو فشل.

**Key details:**
- لو `meson_reset_assert` فشل، بيرجع فوراً — الـ deassert مش هيتعمل، والـ peripheral هيفضل في حالة reset. ده سلوك مقصود لأنه undefined behavior تعمل deassert من غير assert ناجح.

**Pseudocode:**
```
meson_reset_level_toggle(rcdev, id):
    ret = meson_reset_assert(rcdev, id)
    if ret != 0 → return ret
    return meson_reset_deassert(rcdev, id)
```

---

#### `meson_reset_status`

```c
static int meson_reset_status(struct reset_controller_dev *rcdev,
                              unsigned long id);
```

**بتعمل إيه:**
بتقرأ الـ level register وبترجع الـ logical reset status. بتـ XOR الـ raw hardware value مع `level_low_reset` عشان ترجع قيمة consistent: `1` = in reset، `0` = not in reset، بغض النظر عن الـ hardware polarity.

**Parameters:** نفس `meson_reset_reset`.

**Return value:**
- `1` — الـ peripheral في حالة reset.
- `0` — الـ peripheral شغال (out of reset).
- لا بيرجع error من `regmap_read` (الـ return value بتاعه متجاهل — limitation في الـ implementation).

**Key details:**
```c
val = !!(BIT(bit) & val);          // 1 if bit is set, 0 otherwise
return val ^ data->param->level_low_reset;
// level_low_reset=true:
//   bit=0 (line low = hardware reset) → val=0 XOR 1 = 1 (logical "in reset") ✓
//   bit=1 (line high = no reset)      → val=1 XOR 1 = 0 (logical "not in reset") ✓
```

**من بيناديه:** `reset_control_status()` من الـ consumer، أو أي code بيعمل polling على حالة الـ reset.

---

### المجموعة الرابعة: الـ Exported Ops Tables

الـ ops tables دي هي الـ interface الرسمي اللي بيتصدّر للـ modules التانية عن طريق الـ `MESON_RESET` namespace.

---

#### `meson_reset_ops`

```c
const struct reset_control_ops meson_reset_ops = {
    .reset    = meson_reset_reset,     /* pulse-based reset */
    .assert   = meson_reset_assert,    /* level assert */
    .deassert = meson_reset_deassert,  /* level deassert */
    .status   = meson_reset_status,    /* read current state */
};
EXPORT_SYMBOL_NS_GPL(meson_reset_ops, "MESON_RESET");
```

**بتعمل إيه:**
الـ ops table الرئيسية للـ SoCs اللي عندها dedicated reset-pulse registers (مثل meson8b, GXBB, AXG, A1, S4). الـ `.reset` callback بيستخدم الـ `reset_offset` registers للـ atomic pulse، والـ assert/deassert بيستخدموا الـ `level_offset` registers.

**متى بتستخدمها:** لما `meson_reset_param.reset_ops = &meson_reset_ops`.

---

#### `meson_reset_toggle_ops`

```c
const struct reset_control_ops meson_reset_toggle_ops = {
    .reset    = meson_reset_level_toggle,  /* assert + deassert via level regs */
    .assert   = meson_reset_assert,
    .deassert = meson_reset_deassert,
    .status   = meson_reset_status,
};
EXPORT_SYMBOL_NS_GPL(meson_reset_toggle_ops, "MESON_RESET");
```

**بتعمل إيه:**
نفس `meson_reset_ops` لكن الـ `.reset` callback بيعمل toggle عن طريق الـ level registers بدل الـ pulse registers. بيتستخدم مع الـ SoCs اللي مبتدعمش الـ direct pulse write زي T7 (لاحظ إن `t7_param` مليهاش `reset_ops` مضبوط في الـ code الحالي — يُرجَّح إنها بتاخد default أو `toggle_ops` من caller تاني).

---

### مقارنة بين الـ Ops Tables

| Feature | `meson_reset_ops` | `meson_reset_toggle_ops` |
|---|---|---|
| `.reset` implementation | `regmap_write` على pulse register | `assert` + `deassert` على level register |
| هل محتاج `reset_offset` regs | نعم | لا |
| هل محتاج `level_offset` regs | للـ assert/deassert/status | للـ reset + assert/deassert/status |
| مناسب لـ | meson8b, GXBB, AXG, A1, S4 | T7 وما يشابهه |

---

### الـ Data Flow العام

```
Consumer Driver
     │
     │  reset_control_reset(rst)
     ▼
Reset Controller Core (reset.c)
     │
     │  rcdev->ops->reset(rcdev, id)
     ▼
meson_reset_reset / meson_reset_level_toggle
     │
     ▼
meson_reset_offset_and_bit(id)
     │  ┌─────────────────────────────┐
     │  │  offset = (id/32) * stride  │
     │  │  bit    = id % 32           │
     │  └─────────────────────────────┘
     │
     ▼
regmap_write / regmap_update_bits
     │
     ▼
MMIO Write → Amlogic Reset Hardware Register
```

---

### ملاحظات على الـ `struct meson_reset_param`

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; /* ops table pointer */
    unsigned int reset_num;    /* total number of reset lines */
    unsigned int reset_offset; /* base offset for pulse-reset registers */
    unsigned int level_offset; /* base offset for level-reset registers */
    bool level_low_reset;      /* true if reset = active-low (bit=0 = in reset) */
};
```

| Field | meson8b | A1 | S4 | T7 |
|---|---|---|---|---|
| `reset_num` | 256 | 96 | 192 | 224 |
| `reset_offset` | 0x00 | 0x00 | 0x00 | 0x00 |
| `level_offset` | 0x7C | 0x40 | 0x40 | 0x40 |
| `level_low_reset` | true | true | true | true |
| `reset_ops` | `meson_reset_ops` | `meson_reset_ops` | `meson_reset_ops` | NULL (toggle) |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs

الـ reset controller مش بيعمل entries خاصة في debugfs بشكل مباشر، بس الـ reset framework نفسه بيوفر بعض المعلومات.

```bash
# شوف كل الـ reset controllers المسجلة في النظام
ls /sys/kernel/debug/reset/

# لو موجود، اقرأ حالة كل reset line
cat /sys/kernel/debug/reset/*/consumer_list
```

الـ `regmap` نفسه بيوفر debugfs خاص بيه:

```bash
# اقرأ كل registers الـ regmap للـ meson_reset
cat /sys/kernel/debug/regmap/*/registers

# مثال على output متوقع:
# 00000000: 00000000   <- reset register offset 0x0 (write-only pulse)
# 0000007c: ffffffff   <- level register offset 0x7c (meson8b)
```

**ملاحظة مهمة:** الـ reset registers في meson هي write-only للـ pulse reset، فقراءة `reset_offset` هترجع 0 دايمًا — المعلومة الحقيقية في `level_offset`.

---

#### 2. sysfs

```bash
# تأكد إن الـ driver اتحمل صح
ls /sys/bus/platform/drivers/meson_reset/

# شوف الـ device المرتبط
ls /sys/bus/platform/devices/ | grep reset

# مثال: amlogic,meson-gxbb-reset
cat /sys/bus/platform/devices/c1104000.reset/driver/uevent
```

```bash
# عدد الـ reset lines المتاحة (nr_resets)
# مش exposed مباشرة، بس تقدر تستنتجه من DT
cat /sys/firmware/devicetree/base/reset-controller@c1104000/compatible
```

---

#### 3. ftrace — Tracepoints/Events

```bash
# فعّل tracing لكل حاجة في subsystem الـ reset
echo 1 > /sys/kernel/debug/tracing/events/reset/enable

# شوف الـ events المتاحة
ls /sys/kernel/debug/tracing/events/reset/

# فعّل dynamic function tracing للـ meson reset functions
echo 'meson_reset_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# مثال على output:
# kworker/0:1-42 [000] .... meson_reset_assert <-reset_control_assert
# kworker/0:1-42 [000] .... meson_reset_deassert <-reset_control_deassert
```

```bash
# trace الـ regmap reads/writes (مفيد جداً لمراقبة register access)
echo 'regmap_mmio_write regmap_mmio_read' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لملف reset-meson-common.c بالكامل
echo 'file reset-meson-common.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو للـ module كله
echo 'module meson_reset +p' > /sys/kernel/debug/dynamic_debug/control

# شوف كل dynamic debug entries الفاعلة
grep meson /sys/kernel/debug/dynamic_debug/control
```

**لو محتاج تضيف printk مؤقت للكود:**

```c
static int meson_reset_assert(struct reset_controller_dev *rcdev,
                              unsigned long id)
{
    /* Temporary debug: trace every assert call with reset ID */
    dev_dbg(rcdev->dev, "assert reset line %lu\n", id);
    return meson_reset_level(rcdev, id, true);
}
```

```bash
# تأكد إن kernel log level بيطلع debug messages
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_RESET_CONTROLLER` | الـ reset framework نفسه — لازم يكون enabled |
| `CONFIG_REGMAP` | الـ regmap layer الأساسي |
| `CONFIG_REGMAP_MMIO` | MMIO backend للـ regmap |
| `CONFIG_REGMAP_DEBUGFS` | يكشف registers في debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg()` dynamically |
| `CONFIG_FTRACE` | الـ function tracing infrastructure |
| `CONFIG_FUNCTION_TRACER` | يتيح tracing للـ kernel functions |
| `CONFIG_KALLSYMS` | أسماء الـ functions في trace output |
| `CONFIG_DEBUG_KERNEL` | يفتح خيارات debug إضافية |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ regmap locks |
| `CONFIG_PM_DEBUG` | مفيد لو الـ reset مرتبط بـ power management |
| `CONFIG_OF_DYNAMIC` | debugging لـ Device Tree overlays |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'RESET_CONTROLLER|REGMAP|DYNAMIC_DEBUG'
```

---

#### 6. devlink / Tools خاصة بالـ Subsystem

الـ meson reset مش بيدعم devlink، بس في tools مفيدة:

```bash
# اقرأ معلومات الـ reset controller من الـ DT مباشرة
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "reset-controller"

# استخدم reset-gpio tool لو الـ board بيعتمد على GPIO resets
# راقب kernel log أثناء boot
dmesg | grep -i 'meson.*reset\|reset.*meson'

# شوف consumers اللي طالبين reset lines
grep -r . /sys/kernel/debug/reset/ 2>/dev/null
```

---

#### 7. جدول Common Error Messages

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `can't init regmap mmio region` | فشل `devm_regmap_init_mmio` — الـ base address مش صح أو MMIO مش accessible | تحقق من `reg` في DT، وإن الـ memory region محجوزة صح |
| `meson_reset: probe failed -ENOMEM` | `devm_kzalloc` فشل | نادر — تحقق من available memory |
| `meson_reset: probe failed -ENODEV` | `device_get_match_data` رجع NULL — الـ compatible string مش متطابق | تحقق من `compatible` في DT مقابل `meson_reset_dt_ids[]` |
| `meson_reset: probe failed -EBUSY` | الـ MMIO region محجوزة من driver تاني | تحقق من تعارض في memory map |
| `reset ID X out of range` | الـ consumer طلب reset line أكبر من `reset_num` | تحقق من `resets = <&reset_ctrl N>` في DT — N لازم < reset_num |
| `of_reset_simple_xlate: invalid reset ID` | نفس المشكلة — خطأ في DT specifier | راجع عدد cells في `#reset-cells` |
| `devm_reset_controller_register failed` | فشل التسجيل في الـ framework | نادر، راجع الـ kernel log للسبب الفعلي |

---

#### 8. Strategic Points لـ dump_stack() / WARN_ON()

**النقاط الاستراتيجية في `reset-meson-common.c`:**

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
{
    struct meson_reset *data =
        container_of(rcdev, struct meson_reset, rcdev);
    unsigned int offset, bit;

    /* WARN if reset ID exceeds declared reset_num — catches DT bugs */
    WARN_ON(id >= data->param->reset_num);

    meson_reset_offset_and_bit(data, id, &offset, &bit);
    offset += data->param->level_offset;
    assert ^= data->param->level_low_reset;

    return regmap_update_bits(data->map, offset,
                              BIT(bit), assert ? BIT(bit) : 0);
}
```

```c
int meson_reset_controller_register(struct device *dev, struct regmap *map,
                                    const struct meson_reset_param *param)
{
    /* WARN if param is missing ops — catches incomplete SoC configs */
    WARN_ON(!param->reset_ops);

    /* dump_stack here to trace who's registering the controller */
    if (unlikely(!param->reset_num)) {
        dev_err(dev, "zero reset lines configured\n");
        dump_stack();
        return -EINVAL;
    }
    ...
}
```

---

### Hardware Level

---

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

الـ meson reset controller بيستخدم **level registers** لمعرفة حالة الـ reset. اقرأ الـ level register مباشرة وقارنه بما يقوله الـ kernel:

```bash
# افرض meson8b: base = 0xc1104000, level_offset = 0x7c
# اقرأ register 0xc110407c (level register)
devmem2 0xc110407c w

# مثال output:
# Value at address 0xC110407C (0xc110407c): 0xFFFFFFFE
# الـ bit 0 = 0 → reset line 0 is asserted (level_low_reset=true)
# الـ bit 1 = 1 → reset line 1 is deasserted (active)
```

قارن النتيجة بما يقوله الـ kernel:

```bash
# شوف حالة reset line من kernel perspective
cat /sys/kernel/debug/reset/*/status 2>/dev/null
```

**جدول تفسير القيم (level_low_reset = true):**

| قيمة الـ Bit في Register | حالة الـ Reset | تفسير |
|---|---|---|
| 0 | asserted | الـ device في حالة reset |
| 1 | deasserted | الـ device يشتغل طبيعي |

---

#### 2. Register Dump Techniques

```bash
# devmem2: اقرأ reset registers لـ meson8b
# Base address: 0xc1104000 (من DT reg property)

# Reset pulse registers (write-only, قراءتهم = 0 دايمًا)
for i in 0 4 8 c 10 14 18 1c 20 24 28 2c; do
    echo -n "reset reg 0x$i: "
    devmem2 0xc110400$i w 2>/dev/null
done

# Level registers (readable, الأهم للـ debugging)
echo "=== Level Registers (offset 0x7c) ==="
for i in 0 1 2 3 4 5 6 7; do
    addr=$(printf "0xc110407%x" $((0x7c + i*4)))
    echo -n "level reg[$i] @ $addr: "
    devmem2 $addr w 2>/dev/null
done
```

```bash
# io tool (من package io أو u-boot tools)
io -4 -r 0xc1104000 32    # اقرأ 32 registers ابتداءً من base

# /dev/mem مباشرة (لو devmem2 مش متاح)
dd if=/dev/mem bs=4 count=32 skip=$((0xc1104000/4)) 2>/dev/null | hexdump -C
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو بتدبج reset line في الـ hardware:

- **الـ reset pulse width:** في `meson_reset_reset()`، الـ kernel بيكتب BIT(bit) في الـ pulse register — الـ hardware بيولد pulse أوتوماتيك. قيسه بالـ oscilloscope على الـ RST pin للـ IP block.
- **الـ setup time:** بعد deassert، في بعض الـ IP blocks محتاج تستنى قبل ما تبدأ تكلمه. وصّل probe على CLK وRST في نفس الوقت.
- **الـ level:** على Amlogic SoCs، الـ reset active-low في الغالب (`level_low_reset = true`) — يعني الـ RST pin بيروح LOW لما الـ device في reset.

```
RST Pin (active-low):
‾‾‾‾‾‾‾‾|_____|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
         ^     ^
      assert  deassert
      (write 0) (write 1)
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | ما تشوفه في الـ Kernel Log | التشخيص |
|---|---|---|
| الـ reset line مش موصولة صح في PCB | الـ consumer device يفشل في probe بعد deassert | قيس الـ RST pin بـ multimeter — لازم يروح HIGH بعد deassert |
| الـ power supply للـ IP block مش جاهز وقت deassert | device يعمل intermittent errors | راقب تتابع powerup على الـ oscilloscope |
| الـ clock مش enabled قبل deassert | device يعطي timeout errors | تأكد إن الـ clock consumer اتفعّل قبل reset deassert |
| تعارض بين اتنين drivers بيعملوا reset لنفس الـ line | `WARNING: reset control already in use` | راجع الـ DT — كل reset line لـ consumer واحد بس |
| الـ IP block محتاج وقت أطول بعد reset | device يفشل في initialization | ضيف `usleep_range()` بعد deassert في الـ consumer driver |

---

#### 5. Device Tree Debugging

```bash
# تحقق إن DT node موجود وصح
ls /sys/firmware/devicetree/base/ | grep reset

# اقرأ الـ compatible string
xxd /sys/firmware/devicetree/base/reset-controller@c1104000/compatible
# expected: amlogic,meson-gxbb-reset

# اقرأ الـ reg property (base address + size)
xxd /sys/firmware/devicetree/base/reset-controller@c1104000/reg
# expected: c1 10 40 00 00 00 04 00
# يعني base=0xc1104000, size=0x400

# تحقق من #reset-cells
xxd /sys/firmware/devicetree/base/reset-controller@c1104000/#reset-cells
# expected: 00 00 00 01 (يعني 1 cell = reset ID فقط)
```

```bash
# تحقق إن consumer موصول صح
# افرض الـ consumer هو ethernet node
grep -A 5 "resets" /sys/firmware/devicetree/base/ethernet@*/resets 2>/dev/null || \
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A2 "resets"
```

**مثال DT صح:**

```dts
/* Reset controller node */
reset: reset-controller@c1104000 {
    compatible = "amlogic,meson-gxbb-reset";
    reg = <0x0 0xc1104000 0x0 0x400>;
    #reset-cells = <1>;
};

/* Consumer node */
eth: ethernet@c9410000 {
    compatible = "amlogic,meson-gxbb-dwmac";
    resets = <&reset 40>;  /* reset line 40 */
    reset-names = "stmmaceth";
};
```

**أخطاء DT شائعة:**
- `#reset-cells = <1>` لازم يكون 1 دايمًا في meson reset
- الـ reset ID لازم < `reset_num` (256 لـ meson8b، 96 لـ A1، 192 لـ S4)
- الـ base address لازم يطابق الـ SoC datasheet بالظبط

---

### Practical Commands

---

#### جميع الـ Commands جاهزة للـ Copy/Paste

```bash
# ===== 1. تحقق سريع إن الـ driver اتحمل =====
dmesg | grep -i meson_reset
lsmod | grep meson_reset
ls /sys/bus/platform/drivers/meson_reset/

# ===== 2. اقرأ كل registers عبر regmap debugfs =====
REGMAP_PATH=$(ls /sys/kernel/debug/regmap/ | grep -i reset | head -1)
cat /sys/kernel/debug/regmap/$REGMAP_PATH/registers

# ===== 3. فعّل dynamic debug للـ driver =====
echo 'module meson_reset +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file reset-meson-common.c +pfml' > /sys/kernel/debug/dynamic_debug/control
# f=function name, p=print, m=module name, l=line number

# ===== 4. ftrace: تتبع كل reset operations =====
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 'meson_reset_* regmap_update_bits regmap_write' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
# ... شغّل الـ operation اللي بتدبجها ...
echo 0 > tracing_on
cat trace

# ===== 5. اقرأ الـ level registers مباشرة (meson8b مثال) =====
BASE=0xc1104000
LEVEL_OFF=0x7c
for i in $(seq 0 7); do
    ADDR=$(printf "0x%x" $((BASE + LEVEL_OFF + i*4)))
    echo -n "Level reg[$i] @ $ADDR = "
    devmem2 $ADDR w 2>/dev/null | grep "Value" | awk '{print $NF}'
done

# ===== 6. تحقق من الـ DT بشكل كامل =====
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null \
    | grep -A 20 "reset-controller"

# ===== 7. اعرف كل consumers للـ reset controller =====
grep -r "resets" /sys/firmware/devicetree/base/ 2>/dev/null \
    | strings | grep "reset"

# ===== 8. شوف حالة reset framework =====
cat /sys/kernel/debug/reset/*/status 2>/dev/null || \
    echo "No reset debugfs entries found (kernel may need CONFIG_RESET_CONTROLLER=y + debugfs patch)"
```

---

#### Example Output وتفسيره

**مثال ftrace output:**

```
   systemd-1    [000] .... 12.345678: meson_reset_assert <-reset_control_assert
   systemd-1    [000] .... 12.345679: meson_reset_level <-meson_reset_assert
   systemd-1    [000] .... 12.345680: regmap_update_bits <-meson_reset_level
   systemd-1    [000] .... 12.346001: meson_reset_deassert <-reset_control_deassert
   systemd-1    [000] .... 12.346002: meson_reset_level <-meson_reset_deassert
   systemd-1    [000] .... 12.346003: regmap_update_bits <-meson_reset_level
```

التفسير: systemd بيطلب reset لـ device ما. الـ assert جه قبل deassert بـ ~322 microseconds — ده الـ reset pulse الفعلي.

**مثال devmem2 output (level register):**

```
/dev/mem opened.
Memory mapped at address 0xb6f3b000.
Value at address 0xC110407C (0xb6f3b07c): 0xFFFFFFFE
```

التفسير: `0xFFFFFFFE` = كل الـ bits = 1 ما عدا bit 0. في `level_low_reset=true`:
- bit 0 = 0 → reset line 0 **asserted** (device في reset)
- bits 1-31 = 1 → reset lines 1-31 **deasserted** (devices شغّالة)
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box بـ Amlogic S905X4 — HDMI مش شغال بعد الـ suspend

#### العنوان
**HDMI output معطّل بعد الـ system resume على S905X4 Android TV Box**

#### السياق
منتج: Android TV box تجاري بـ Amlogic S905X4 (مبني على GXBB/S4 family).
الـ SoC يستخدم `amlogic,meson-s4-reset` في الـ Device Tree.
الـ `meson_s4_param` عنده `reset_num = 192` و `level_offset = 0x40` و `level_low_reset = true`.
المشكلة ظهرت بعد تفعيل الـ suspend-to-RAM feature في نسخة production.

#### المشكلة
بعد ما الجهاز يرجع من الـ suspend، شاشة HDMI تفضل سودة.
الـ HDMI encoder peripheral على الـ SoC محتاج assert/deassert صريح على الـ reset line بعد الـ resume.
الـ driver بيعمل `devm_reset_control_get` وبيعمل `reset_control_deassert()` بس مفيش حاجة بتشتغل.

#### التحليل

الـ `reset_control_deassert()` بتوصل في النهاية لـ `meson_reset_deassert()` في `reset-meson-common.c`:

```c
static int meson_reset_deassert(struct reset_controller_dev *rcdev,
                                unsigned long id)
{
    return meson_reset_level(rcdev, id, false); /* assert=false */
}
```

وفي `meson_reset_level()`:

```c
assert ^= data->param->level_low_reset;
/* level_low_reset = true, assert=false => false ^ true = true */
/* يعني بيكتب BIT(bit) في الـ register => يعمل assert مش deassert! */
return regmap_update_bits(data->map, offset,
                          BIT(bit), assert ? BIT(bit) : 0);
```

المشكلة: الـ HDMI driver غلط في تفسير الـ semantics. الـ `level_low_reset = true` معناه إن الـ bit لما يكون `0` → reset active. لما نعمل `deassert` صح، الـ XOR بيضمن إن الـ bit يتكتب `1`. ده صح.

بس لما راجع الـ DT:

```dts
/* الغلط */
resets = <&reset_ctrl RESET_HDMI_TX>;
reset-names = "hdmi";
```

**الـ reset ID اللي اتحدد في الـ DT كان غلط** — بيشاور على reset line تانية مش الـ HDMI TX. اتضح بعد مراجعة الـ S905X4 datasheet إن `RESET_HDMI_TX` هو ID رقم 34 مش 35.

الـ `meson_reset_offset_and_bit()` بتحسب:

```c
/* stride = 4 bytes, BITS_PER_BYTE = 8, stride*8 = 32 */
*offset = (id / 32) * 4;   /* register offset */
*bit    = id % 32;          /* bit position */
```

لو الـ ID غلط بـ 1، بيعمل assert/deassert على bit مختلف في نفس الـ register، والـ HDMI TX reset line مش بتتأثر.

#### الحل

**خطوة 1 — تأكيد الـ ID الصح بـ debugfs:**

```bash
# على الـ target
cat /sys/kernel/debug/reset/reset34/status
cat /sys/kernel/debug/reset/reset35/status
```

**خطوة 2 — تصحيح الـ DT:**

```dts
/* الصح */
resets = <&reset_ctrl 34>;   /* RESET_HDMI_TX الصحيح */
reset-names = "hdmi";
```

**خطوة 3 — verification:**

```bash
# بعد التعديل
echo mem > /sys/power/state
# اضغط أي زرار
dmesg | grep -i hdmi
# المفروض تشوف "hdmi_tx: reset deasserted"
```

#### الدرس المستفاد

الـ `meson_reset_offset_and_bit()` دقيقة جداً — خطأ في الـ ID بمقدار 1 بيأثر على bit واحد في نفس أو register مختلف. دايمًا راجع الـ SoC datasheet الـ reset IDs مش header files قديمة.

---

### السيناريو 2: Industrial Gateway بـ Amlogic A1 — I2C مش بيتعرف عليه بعد الـ boot

#### العنوان
**I2C controller مش شغال على Amlogic A1 industrial gateway بعد cold boot**

#### السياق
منتج: industrial IoT gateway يشتغل Linux، مبني على Amlogic A1 SoC.
الـ SoC بيستخدم `amlogic,meson-a1-reset` في الـ DT.
الـ `meson_a1_param`: `reset_num = 96`، `level_offset = 0x40`، `level_low_reset = true`.
الـ I2C bus بيتوصل بيه sensor array حرارة وضغط.
المشكلة: `i2c-meson.c` driver بيقول "Failed to reset I2C controller" عند الـ probe.

#### المشكلة

```
[    2.341] meson-i2c fdd00000.i2c: Failed to reset I2C controller: -110
```

الـ `-110` هو `ETIMEDOUT`. الـ probe فاشل ومش بيكتشف أي device.

#### التحليل

الـ I2C driver بيعمل:

```c
ret = reset_control_reset(i2c->reset);
```

ده بيوصل لـ `meson_reset_ops.reset` وهي `meson_reset_reset()`:

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                             unsigned long id)
{
    /* ... */
    offset += data->param->reset_offset; /* reset_offset = 0x0 */
    return regmap_write(data->map, offset, BIT(bit));
    /* بيكتب BIT واحد بس — self-clearing pulse reset */
}
```

الـ A1 يستخدم `reset_offset = 0x0` للـ pulse reset registers. الـ write بيعمل reset pulse وبيرجع على طول.

**المشكلة الحقيقية:** الـ reset controller node في الـ DT مش موجود أو عنده `status = "disabled"`:

```dts
/* الغلط — missing or disabled */
reset: reset-controller@fe000000 {
    compatible = "amlogic,meson-a1-reset";
    reg = <0xfe000000 0x100>;
    status = "disabled"; /* <-- ده المشكلة */
    #reset-cells = <1>;
};
```

لما الـ reset controller نفسه مش متسجل، الـ `devm_reset_controller_register` مش بيتنفذ، وأي driver بيطلب `reset_control_get` بياخد `-EPROBE_DEFER` أو timeout.

**تأكيد من الكود:**

```c
int meson_reset_controller_register(struct device *dev, struct regmap *map,
                                    const struct meson_reset_param *param)
{
    /* لو الـ probe فشل أو ما اتنفذش، مفيش rcdev متسجل */
    return devm_reset_controller_register(dev, &data->rcdev);
}
```

#### الحل

**خطوة 1 — تفعيل الـ reset controller في الـ DT:**

```dts
reset: reset-controller@fe000000 {
    compatible = "amlogic,meson-a1-reset";
    reg = <0xfe000000 0x100>;
    status = "okay"; /* fix */
    #reset-cells = <1>;
};
```

**خطوة 2 — التحقق من الـ probe:**

```bash
dmesg | grep meson_reset
# المتوقع:
# [    1.123] meson_reset fe000000.reset-controller: registered

ls /sys/bus/platform/drivers/meson_reset/
```

**خطوة 3 — اختبار I2C:**

```bash
i2cdetect -y 0
# المفروض تشوف الـ sensors
```

#### الدرس المستفاد

الـ `meson_reset_controller_register()` مش بتفشل بصوت عالٍ لو الـ DT node مش مفعّل — الـ driver مش بيتـprobe أصلاً وكل اللي يعتمد عليه بياخد `-EPROBE_DEFER`. دايماً تحقق من `dmesg | grep meson_reset` أول خطوة.

---

### السيناريو 3: Custom Board Bring-up بـ Amlogic T7 — USB3.0 مش بيشتغل

#### العنوان
**USB 3.0 host controller ما بيتعرفش على devices أثناء bring-up لـ custom T7 board**

#### السياق
منتج: custom compute board للـ ML inference يستخدم Amlogic T7 SoC.
الـ T7 يستخدم `t7_param`: `reset_num = 224`، `level_offset = 0x40`، `level_low_reset = true`.
**ملاحظة مهمة:** `t7_param` عنده `reset_ops = NULL` (مش set في الـ struct initializer).

```c
static const struct meson_reset_param t7_param = {
    .reset_num      = 224,
    .reset_offset   = 0x0,
    .level_offset   = 0x40,
    .level_low_reset = true,
    /* reset_ops غير محدد! */
};
```

الـ USB3 driver بيطلب `reset_control_reset()` وعمل kernel panic.

#### المشكلة

```
[    3.456] BUG: kernel NULL pointer dereference, address: 0000000000000000
[    3.456] Oops: 0010 [#1] SMP
[    3.456] Call Trace:
[    3.456]  reset_control_reset+0x34/0x80
[    3.456]  dwc3_core_init+0x1a0/0x400 [dwc3]
```

#### التحليل

في `meson_reset_controller_register()`:

```c
data->rcdev.ops = data->param->reset_ops;
/* t7_param.reset_ops = NULL => rcdev.ops = NULL */
```

لما الـ USB driver بيعمل `reset_control_reset()`:

```c
/* في kernel/reset/core.c */
ret = rcdev->ops->reset(rcdev, id);
/* rcdev->ops = NULL => NULL pointer dereference */
```

**السبب:** الـ T7 SoC بيستخدم `meson_reset_toggle_ops` مش `meson_reset_ops` لأن الـ hardware بتاعه مش بيدعم الـ self-clearing pulse reset registers بنفس الطريقة. الـ `toggle_ops` بتعمل assert ثم deassert يدويًا. الـ developer اللي كتب الـ DT ما حددش الـ `reset_ops` في `t7_param`.

الـ header يُعرّف الاتنين:

```c
/* في reset-meson.h */
extern const struct reset_control_ops meson_reset_ops;
extern const struct reset_control_ops meson_reset_toggle_ops;
```

الـ `t7_param` في الكود الأصلي `reset_ops` بيكون `NULL` لأن الـ T7 كان بيستخدم ops مختلفة، والكود اللي بيستخدمه كان pending upstream آن ذاك.

#### الحل

**الحل 1 — تحديد الـ ops الصح في الـ param:**

```c
static const struct meson_reset_param t7_param = {
    .reset_ops      = &meson_reset_toggle_ops, /* أضف السطر ده */
    .reset_num      = 224,
    .reset_offset   = 0x0,
    .level_offset   = 0x40,
    .level_low_reset = true,
};
```

**الحل 2 — defensive check في `meson_reset_controller_register`:**

قبل التسجيل، تأكد من إن الـ ops مش NULL:

```c
if (!param->reset_ops) {
    dev_err(dev, "reset_ops not set in param\n");
    return -EINVAL;
}
```

**التحقق:**

```bash
dmesg | grep -i usb
lsusb
# المفروض يشوف الـ USB devices
```

#### الدرس المستفاد

الـ `t7_param` في الكود الحالي ما بيحددش `reset_ops` وده intentional لأنه incomplete. أي SoC جديد يستخدم `meson_reset_h` API لازم يحدد الـ `reset_ops` صراحةً وإلا NULL dereference محتوم.

---

### السيناريو 4: Amlogic GXBB Android TV — SPI Flash مش بيتـdetect في الـ bootloader

#### العنوان
**SPI NOR Flash غير مرئي من الـ kernel على GXBB-based Android TV أثناء migration لـ mainline kernel**

#### السياق
منتج: Android TV box بـ Amlogic S905 (GXBB family).
الـ DT يستخدم `amlogic,meson-gxbb-reset` اللي بيmap لـ `meson8b_param`:
`reset_num = 256`، `reset_offset = 0x0`، `level_offset = 0x7c`، `level_low_reset = true`.
الـ SPI controller محتاج reset قبل ما يشتغل.
المشكلة ظهرت بعد migration من vendor kernel 3.14 لـ mainline 6.x.

#### المشكلة

```
[    2.100] spi-meson-spifc c1108c80.spi: failed to get reset
[    2.100] spi-meson-spifc c1108c80.spi: probe failed: -ENOENT
```

#### التحليل

الـ `-ENOENT` معناه إن الـ `reset_control_get()` مش لاقي الـ reset handle بالاسم اللي طلبه.

الـ SPI driver بيعمل:

```c
spifc->reset = devm_reset_control_get(dev, "spifc");
```

بيدور على reset بالاسم `"spifc"` في الـ DT.

الـ DT الـ vendor القديم:

```dts
/* vendor DT قديم */
spi@c1108c80 {
    compatible = "amlogic,meson-spifc";
    reg = <0xc1108c80 0x80>;
    /* مفيش resets property! */
};
```

الـ mainline DT المطلوب:

```dts
spi@c1108c80 {
    compatible = "amlogic,meson-spifc";
    reg = <0xc1108c80 0x80>;
    resets = <&reset_ctrl RESET_SPIFC>; /* أضف ده */
    reset-names = "spifc";
    clocks = <&clkc CLKID_SPIFC>;
    clock-names = "core";
};
```

الـ `reset_ctrl` هو الـ reset controller node اللي بيستخدم `meson8b_param`.

**كيف الـ subsystem بيربطها:**
لما الـ SPI driver بيعمل `devm_reset_control_get(dev, "spifc")`، الـ kernel بيبص في الـ DT على `resets` و`reset-names` properties، بيجيب الـ phandle والـ ID، وبيوصل لـ `meson_reset_controller_register` اللي سجّل الـ `rcdev` بـ `nr_resets = 256`. الـ `RESET_SPIFC` لازم يكون أقل من 256 عشان يعدّي الـ bounds check.

**التحقق من الـ ID:**

```bash
grep -r "RESET_SPIFC" include/dt-bindings/reset/
# نتيجة:
# include/dt-bindings/reset/amlogic,meson-gxbb-reset.h:#define RESET_SPIFC  24
```

الـ 24 < 256 ✓ — صح.

#### الحل

**إضافة الـ reset properties لـ SPI node في الـ DT:**

```dts
reset_ctrl: reset-controller@c1104400 {
    compatible = "amlogic,meson-gxbb-reset";
    reg = <0xc1104400 0x20>;
    #reset-cells = <1>;
};

spi@c1108c80 {
    compatible = "amlogic,meson-spifc";
    reg = <0xc1108c80 0x80>;
    resets = <&reset_ctrl 24>; /* RESET_SPIFC */
    reset-names = "spifc";
};
```

**اختبار:**

```bash
dmesg | grep spifc
# [    2.100] spi-meson-spifc c1108c80.spi: registered master spi0

ls /dev/spi*
mtdinfo
```

#### الدرس المستفاد

الـ migration من vendor kernel لـ mainline بتحتاج إعادة كتابة الـ DT من الصفر. الـ `meson8b_param` بـ `reset_num = 256` بيغطي كل الـ IDs ولكن الـ DT nodes لازم تحتوي الـ `resets` و `reset-names` properties صح، وإلا الـ driver مش بيقدر يجيب الـ reset handle.

---

### السيناريو 5: IoT Sensor Hub بـ Amlogic A1 — UART بيفشل بعد الـ suspend بسبب wrong level polarity

#### العنوان
**UART2 communication fails بعد كل suspend cycle على Amlogic A1 IoT device**

#### السياق
منتج: IoT environmental sensor hub يستخدم Amlogic A1 SoC.
الـ UART2 متوصل بـ cellular modem (Quectel EC21).
بعد كل suspend/resume cycle، الـ UART بيوقف يتكلم مع الـ modem.
`meson_a1_param`: `level_offset = 0x40`، `level_low_reset = true`.

#### المشكلة

```
[   45.231] ttyAML2: UART timeout, modem not responding
[   45.231] ec21: reset failed, communication lost
```

الـ cellular driver بيعمل assert ثم deassert للـ UART reset بعد الـ resume.
بس الـ UART بيفضل في حالة reset (asserted) بعد الـ deassert call.

#### التحليل

الكود في `meson_reset_level()`:

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
{
    /* ... */
    assert ^= data->param->level_low_reset;
    /* level_low_reset = true */

    return regmap_update_bits(data->map, offset,
                              BIT(bit), assert ? BIT(bit) : 0);
}
```

**للـ assert (reset active):**
- `assert = true`, `level_low_reset = true`
- `true ^ true = false`
- بيكتب `0` في الـ bit → ده صح لـ active-low reset

**للـ deassert (reset inactive):**
- `assert = false`, `level_low_reset = true`
- `false ^ true = true`
- بيكتب `1` في الـ bit → ده صح

**إذن الـ logic صح. المشكلة فين؟**

بعد تحليل أعمق، اتضح إن الـ cellular driver بيعمل `reset_control_assert` وبعدين بياخد `msleep(10)` ثم `reset_control_deassert`. بس في نسخة معينة من الـ driver، الـ msleep اتحذف بالغلط:

```c
/* الكود الغلط في الـ driver */
reset_control_assert(uart_reset);
/* msleep(10); -- اتحذف بالغلط */
reset_control_deassert(uart_reset);
```

الـ A1 hardware محتاج minimum reset pulse width ≥ 5μs. لما الـ assert والـ deassert بيحصلوا على طول، الـ hardware مش بيتعرف على الـ reset pulse.

**اتحقق بـ logic analyzer:**

```
UART2_RESET_L: _____|‾|___  ← pulse ضيق جداً (< 1μs)
```

**الـ `meson_reset_status()` تأكيد:**

```c
static int meson_reset_status(struct reset_controller_dev *rcdev,
                              unsigned long id)
{
    regmap_read(data->map, offset, &val);
    val = !!(BIT(bit) & val);
    return val ^ data->param->level_low_reset;
    /* level_low_reset=true: لو bit=0 → val=0 → 0^1=1 → in reset */
    /* level_low_reset=true: لو bit=1 → val=1 → 1^1=0 → out of reset */
}
```

بعد الـ "deassert"، `reset_control_status()` بيقول `0` (out of reset) ولكن الـ hardware مش قرأ الـ pulse.

#### الحل

**إصلاح الـ cellular driver:**

```c
/* الكود الصح */
ret = reset_control_assert(uart_reset);
if (ret)
    return ret;

usleep_range(10000, 20000); /* 10ms — أكتر من كافي للـ A1 */

ret = reset_control_deassert(uart_reset);
if (ret)
    return ret;
```

**أو استخدام `reset_control_reset()` اللي بتعمل pulse آمن:**

```c
/* أبسط وأضمن */
ret = reset_control_reset(uart_reset);
```

الـ `meson_reset_ops.reset` بتستخدم الـ pulse reset registers عند `reset_offset = 0x0` — بتكتب bit واحد والـ hardware بيتحكم في الـ timing.

**التحقق:**

```bash
# بعد الـ fix، اختبر 100 suspend/resume cycle
for i in $(seq 1 100); do
    echo mem > /sys/power/state
    sleep 5
    # اختبر الـ modem
    AT_CMD=$(echo "AT" | timeout 2 microcom -s 115200 /dev/ttyAML2)
    echo "Cycle $i: $AT_CMD"
done
```

**تأكيد الـ reset status:**

```bash
cat /sys/kernel/debug/reset/reset<UART2_ID>/status
# يجب يقول 0 (out of reset) بعد الـ resume
```

#### الدرس المستفاد

الـ `meson_reset_level()` بتنفذ الـ register write فوراً بدون أي timing guarantee. الـ hardware reset pulse width requirements لازم تتحقق منها في الـ SoC TRM. استخدام `reset_control_reset()` عبر `meson_reset_ops` بدل assert/deassert يدوي أأمن لأنه بيستخدم الـ hardware pulse mechanism. الـ `level_low_reset = true` في `meson_a1_param` بتعكس الـ polarity تلقائياً عبر XOR — ما تحتاجش تعكسها يدوياً في الـ driver.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/driver-api/reset.rst`](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | الـ reset controller API الرسمي — consumer interface + provider interface |
| [`docs.kernel.org/driver-api/reset.html`](https://docs.kernel.org/driver-api/reset.html) | النسخة الـ HTML من نفس التوثيق |
| `drivers/reset/amlogic/` | الـ source directory للـ Meson reset driver |
| `include/linux/reset-controller.h` | تعريف `reset_control_ops` و`reset_controller_dev` |
| `include/linux/reset.h` | الـ consumer API — `reset_control_get`, `reset_control_assert`, إلخ |

---

### مقالات LWN.net

| العنوان | الرابط |
|---------|-------|
| Add generic GPIO reset driver | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) |
| clk: en7523: reset-controller support for EN7523 SoC | [lwn.net/Articles/1039543](https://lwn.net/Articles/1039543/) |
| Driver implementer's API guide (static.lwn.net) | [static.lwn.net/kerneldoc/driver-api/index.html](https://static.lwn.net/kerneldoc/driver-api/index.html) |

---

### نقاشات الـ Mailing List

الـ reset controller subsystem أسسه **Philipp Zabel** من Pengutronix. الـ patch الأصلي اللي رفعه سنة 2013:

| الرابط | الموضوع |
|--------|---------|
| [patchwork.kernel.org — v4,2/8](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | reset: Add reset controller API (الـ patch الأصلي) |
| [lore.kernel.org — GIT PULL v2](https://lore.kernel.org/linux-arm-kernel/1365683829.4388.52.camel@pizza.hi.pengutronix.de/) | GIT PULL v2: Reset controller API |
| [lore.kernel.org — v6.2 fixes](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/) | GIT PULL: Reset controller fixes for v6.2 |

#### نقاشات Amlogic/BayLibre

| الرابط | الموضوع |
|--------|---------|
| [lore.kernel.org — PATCH 0/8](https://lore.kernel.org/lkml/20240710162526.2341399-1-jbrunet@baylibre.com/T/) | reset: amlogic: move audio reset drivers out of CCF — **Jerome Brunet 2024** |
| [lore.kernel.org — meson audio arb](https://lore.kernel.org/all/20180706143122.7612-3-jbrunet@baylibre.com/) | reset: meson: add meson audio arb driver |

---

### الـ Kernel Commits المهمة

```bash
# للبحث عن تاريخ الـ Meson reset driver في git:
git log --oneline -- drivers/reset/amlogic/
git log --oneline -- drivers/reset/reset-meson.c

# الـ commit الأصلي للـ reset controller API:
git log --oneline --all | grep -i "reset controller"
```

الـ driver انتقل من `drivers/reset/reset-meson.c` إلى `drivers/reset/amlogic/` في kernel 6.13 كجزء من إعادة هيكلة دعم Amlogic.

---

### مصادر Amlogic/BayLibre

| المصدر | الوصف |
|--------|-------|
| [baylibre.com — Improved AmLogic Support](https://baylibre.com/improved-amlogic-support-mainline-linux/) | كيف BayLibre حسّنت دعم Amlogic في الـ mainline |
| [linux-meson.com/mainlining.html](https://linux-meson.com/mainlining.html) | تقدم الـ mainlining لـ Amlogic Meson |
| [cateee.net — CONFIG_RESET_MESON](https://cateee.net/lkddb/web-lkddb/RESET_MESON.html) | Kernel Driver Database: Meson Reset Driver |

---

### موارد kernelnewbies.org و elinux.org

| المصدر | الوصف |
|--------|-------|
| [kernelnewbies.org/KernelHacking](https://kernelnewbies.org/KernelHacking) | نقطة انطلاق لكتابة drivers |
| [kernelnewbies.org/FirstKernelPatch](https://kernelnewbies.org/FirstKernelPatch) | خطوات إرسال أول patch للـ kernel |
| [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تتبع التغييرات بين إصدارات الـ kernel |
| [elinux.org/Kernel_Development](https://elinux.org/Kernel_Development) | ملخص لتطوير الـ kernel الـ embedded |

---

### الـ Regmap Framework — مصادر مساعدة

الـ `reset-meson.h` يستخدم `regmap` بشكل أساسي، فالفهم مطلوب:

| المصدر | الوصف |
|--------|-------|
| [collabora.com — Using regmaps](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) | شرح عملي لاستخدام regmap لتعميم الـ drivers |
| [opensourceforu.com — regmap](https://www.opensourceforu.com/2017/01/regmap-reducing-redundancy-linux-code/) | regmap: Reducing the Redundancy in Linux Code |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model — فهم `struct device` والـ driver model
- **الفصل 11**: Data Types in the Kernel
- متاح مجانًا على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — فهم الـ driver registration
- **الفصل 14**: The Block I/O Layer — فهم طريقة عمل الـ callback ops

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 16**: Kernel Initialization — فهم دور الـ reset في إقلاع الـ SoC
- مفيد لفهم متى وليه بيتم reset الـ IP blocks أثناء الـ boot

#### Mastering Linux Device Driver Development — John Madieu
- الفصل الخاص بالـ regmap API — مباشر ومطبق

---

### مصطلحات البحث

لو عايز تلاقي معلومات أكتر، استخدم الـ search terms دي:

```
linux reset controller subsystem
reset_controller_dev ops
devm_reset_controller_register
reset_control_ops assert deassert
amlogic meson reset driver upstream
regmap mmio reset driver
BayLibre amlogic linux kernel
drivers/reset/amlogic
CONFIG_RESET_MESON kconfig
device tree reset-cells binding
of_reset_simple_xlate
```
## Phase 8: Writing simple module

### الفكرة

الـ `meson_reset_controller_register` هي الـ exported function الوحيدة في الـ `reset-meson.h`، وبتسجّل reset controller للـ Amlogic Meson SoC. هنستخدم **kprobe** عشان نعمل hook على الدالة دي ونطبع اسم الـ device اللي بتطلبها وعدد الـ reset lines المتاحة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe hook on meson_reset_controller_register()
 * Prints device name and reset line count on every registration call.
 */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/device.h>

/* meson_reset_param is defined in the driver header; we redeclare
 * only what we need to avoid pulling in driver-private headers. */
struct meson_reset_param_probe {
	const void *reset_ops;   /* reset_control_ops pointer — we skip it */
	unsigned int reset_num;
	unsigned int reset_offset;
	unsigned int level_offset;
	bool         level_low_reset;
};

/* ---------- kprobe callback ---------- */

/*
 * pre_handler fires just before the probed instruction executes.
 * regs holds the CPU register state at that moment, which lets us
 * read the original function arguments (System V AMD64 ABI):
 *   rdi = arg0 → struct device *dev
 *   rsi = arg1 → struct regmap  *map   (not used here)
 *   rdx = arg2 → const struct meson_reset_param *param
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
	/* Retrieve arguments from registers (x86-64 ABI) */
	struct device *dev =
		(struct device *)regs->di;

	const struct meson_reset_param_probe *param =
		(const struct meson_reset_param_probe *)regs->dx;

	/* Basic sanity checks before dereferencing */
	if (!dev || !param)
		return 0;

	pr_info("meson_reset probe: device=\"%s\" reset_num=%u "
		"reset_offset=0x%x level_offset=0x%x level_low=%d\n",
		dev_name(dev),        /* e.g. "ff800000.reset-controller" */
		param->reset_num,
		param->reset_offset,
		param->level_offset,
		param->level_low_reset);

	return 0; /* 0 = let the original function run normally */
}

/* ---------- kprobe struct ---------- */

static struct kprobe kp = {
	/* Symbol name of the function we want to intercept */
	.symbol_name = "meson_reset_controller_register",
	.pre_handler = handler_pre,
};

/* ---------- init / exit ---------- */

static int __init meson_reset_probe_init(void)
{
	int ret;

	ret = register_kprobe(&kp);
	if (ret < 0) {
		pr_err("meson_reset probe: register_kprobe failed: %d\n", ret);
		return ret;
	}

	pr_info("meson_reset probe: planted at %pS\n", kp.addr);
	return 0;
}

static void __exit meson_reset_probe_exit(void)
{
	/* Must unregister before module unload to avoid a dangling handler
	 * pointer that would cause a kernel panic on the next probe hit. */
	unregister_kprobe(&kp);
	pr_info("meson_reset probe: removed\n");
}

module_init(meson_reset_probe_init);
module_exit(meson_reset_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc Demo");
MODULE_DESCRIPTION("kprobe on meson_reset_controller_register — prints reset controller registration info");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/kernel.h` | `pr_info` و macros أساسية |
| `linux/module.h` | `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/device.h` | `struct device`, `dev_name()` |

الـ `reset-meson.h` نفسه ما بنضيفوش لأننا بنشتغل من خارج شجرة الـ driver، وبدل كده بنعيد تعريف جزء بسيط من الـ `meson_reset_param` بنفس الـ layout عشان نقدر نقرأ الـ fields اللي محتاجينها.

---

#### الـ `struct meson_reset_param_probe`

بنعرّف نسخة مصغّرة من `meson_reset_param` بنفس ترتيب الـ fields عشان الـ memory layout يتطابق. ده approach آمن لأن الـ struct بسيط وما فيهوش padding surprises.

---

#### الـ `handler_pre`

الـ **pre_handler** بيتشغّل قبل أول instruction في الدالة المستهدفة مباشرةً، يعني الـ arguments لسّه موجودين في الـ registers. على الـ x86-64 ABI:
- `regs->di` = أول argument (`struct device *dev`)
- `regs->si` = تاني argument (`struct regmap *map`) — ما احتجناهوش
- `regs->dx` = تالت argument (`const struct meson_reset_param *param`)

الـ `dev_name(dev)` بيرجع اسم الـ device زي `"ff800000.reset-controller"` اللي بييجي من الـ device tree، وده مفيد جداً لتتبع أي SoC peripheral بيسجّل نفسه.

---

#### الـ `struct kprobe kp`

الـ **symbol_name** بيخلي الـ kernel يحلّ العنوان تلقائياً من الـ kallsyms، فمش محتاجين نحدد عنوان hardcoded. الـ `kp.addr` بيتملّى بعد الـ `register_kprobe` ونطبعه بـ `%pS` عشان نشوف العنوان الفعلي مع اسم الـ symbol.

---

#### الـ `module_init` و `module_exit`

الـ `register_kprobe` بيزرع **breakpoint instruction** (INT3 على x86) في الكود الحي للكيرنل. لو فشل — مثلاً الـ symbol مش موجود أو الـ CONFIG_KPROBES مش مفعّل — بيرجع error ونخرج فوراً. الـ `unregister_kprobe` في الـ exit ضروري عشان يشيل الـ breakpoint ويلغي الـ handler قبل ما الـ module يتـ unload؛ من غير كده أي reset registration جديدة هتودّي في kernel panic لأن الـ handler pointer هيبقى dangling.

---

### الـ Makefile

```makefile
obj-m += meson_reset_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تشغيل وقراءة الـ output

```bash
# تحميل الـ module
sudo insmod meson_reset_probe.ko

# متابعة الـ output (على board Amlogic أو بعد تحميل reset-meson.ko)
sudo dmesg | grep "meson_reset probe"

# مثال على output متوقع:
# [  12.345678] meson_reset probe: planted at meson_reset_controller_register+0x0/0x80 [reset_meson]
# [  12.456789] meson_reset probe: device="ff800000.reset-controller" reset_num=96 reset_offset=0x0 level_offset=0x40 level_low=0

# إزالة الـ module
sudo rmmod meson_reset_probe
```

> **ملاحظة:** الـ module ده بيشتغل على أي kernel عنده `CONFIG_KPROBES=y`. على board Amlogic مع reset-meson driver، الـ hook هيشتغل أثناء الـ boot أو لما تـ reload الـ driver.
