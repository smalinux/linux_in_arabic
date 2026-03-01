## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **Reset Controller Framework** في الـ Linux kernel — وده subsystem مستقل بيتكلم عنه الـ MAINTAINERS في السطر:

```
RESET CONTROLLER FRAMEWORK
M: Philipp Zabel <p.zabel@pengutronix.de>
F: drivers/reset/
F: include/linux/reset-controller.h
F: include/linux/reset.h
```

والـ driver ده خاص بـ chips من شركة **Amlogic** (بتعمل SoCs في Android TV boxes وغيرها)، بيشتغل تحت مظلة الـ Reset Controller Framework.

---

### القصة الكبيرة — إيه المشكلة اللي بتتحل؟

تخيل إنك بتشغّل جهاز فيه شريحة معالج (SoC) زي الـ Amlogic Meson — وده الـ chip اللي بتلاقيه في TV boxes كتير زي Xiaomi Mi Box وAmlogic S905. الـ SoC ده جوّاه عشرات الـ "كتل" (blocks) — USB، HDMI، Ethernet، Audio، GPU، إلخ.

كل "كتلة" من دول محتاجة في بعض الأوقات تتعمل لها **reset** — يعني بنقولها "ابدأي من الأول" أو "اتوقفي". ده بيحصل مثلاً:
- لما الـ driver بيتحمّل لأول مرة
- لما في hang أو error
- لما بنحاول نوفّر استهلاك الطاقة

المشكلة إن كل chip عنده registers مختلفة في addresses مختلفة، وبعض الـ resets بتشتغل بـ "pulse" (اضغط وافرج)، وبعضها بيشتغل بـ "level" (ثبّت القيمة). يعني محتاجين كود مرن يتعامل مع الاختلافات دي.

---

### الحل — الـ Reset Controller Framework

الـ Linux اخترع framework موحّد اسمه **Reset Controller Framework**. الفكرة إن:
1. كل driver لـ hardware جديد بيسجّل نفسه كـ "reset controller"
2. أي driver تاني (USB, GPU, إلخ) يقدر يطلب reset لنفسه بكل بساطة بـ `reset_control_deassert()` أو `reset_control_reset()` من غير ما يعرف أي تفاصيل hardware

يعني الـ USB driver مش محتاج يعرف إن الـ reset register موجود على address كذا وبيت كذا — ده شغل الـ reset controller driver.

---

### دور الملف `reset-meson-common.c`

الملف ده هو **القلب المشترك** لكل درايفرات reset الخاصة بـ Amlogic Meson. بدل ما كل chip تكتب نفس الكود من الأول، الملف ده بيقدّم:

**1. حساب الـ Register والـ Bit**
كل reset line ليها رقم (ID). الدالة `meson_reset_offset_and_bit()` بتحوّل الـ ID ده لـ register offset + bit number:

```c
/* convert reset ID → which register + which bit inside it */
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
{
    unsigned int stride = regmap_get_reg_stride(data->map);
    *offset = (id / (stride * BITS_PER_BYTE)) * stride;
    *bit = id % (stride * BITS_PER_BYTE);
}
```

يعني لو الـ registers عرضها 32 bit، الـ reset ID رقم 33 يبقى في الـ register التاني، bit رقم 1.

**2. نوعين من الـ Reset**

الـ Amlogic chips عندها طريقتين:

| النوع | الوصف | الـ ops المستخدمة |
|-------|--------|-------------------|
| **Pulse reset** | بنكتب bit واحد في register مخصص فيحصل reset أوتوماتيك | `meson_reset_ops` |
| **Level/Toggle reset** | بنـ assert الـ bit ثم نـ deassert يدوياً | `meson_reset_toggle_ops` |

**3. الدالة الرئيسية `meson_reset_controller_register()`**
دي هي اللي بتعمل كل حاجة — بتاخد parameters من الـ driver الخاص بكل chip وبتسجّل الـ reset controller مع الـ kernel:

```c
int meson_reset_controller_register(struct device *dev, struct regmap *map,
                                    const struct meson_reset_param *param)
{
    /* allocate + fill meson_reset struct */
    /* register with kernel's reset framework */
    return devm_reset_controller_register(dev, &data->rcdev);
}
```

---

### السيناريو الكامل من الأول للآخر

```
Device Tree (YAML)
      │
      ▼
reset-meson.c  ← يُحدّد: هل ده meson8b أم A1 أم S4?
  (Platform driver)        ↓ يبعت param + regmap
      │
      ▼
reset-meson-common.c  ← يسجّل reset controller واحد مع الـ kernel
      │
      ▼
Reset Controller Framework  ← يخزّن الـ controller في قائمة
      │
      ▼
USB/GPU/Audio drivers  ← يطلبوا reset بكل بساطة
```

---

### الـ `meson_reset_param` — قلب التكوين

الـ struct ده في الـ header `reset-meson.h` هو اللي بيفرّق بين كل chip وتانية:

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; /* pulse or toggle? */
    unsigned int reset_num;    /* how many reset lines? */
    unsigned int reset_offset; /* where are pulse-reset registers? */
    unsigned int level_offset; /* where are level-reset registers? */
    bool level_low_reset;      /* is reset active when bit=0? */
};
```

مثلاً الـ `meson8b` عنده 256 reset line، بينما الـ `meson-a1` عنده 96 بس.

---

### الملفات المرتبطة اللي لازم تعرفها

**Core الـ subsystem (الملفات اللي بنتكلم عنها):**

| الملف | الدور |
|-------|-------|
| `drivers/reset/amlogic/reset-meson-common.c` | **الملف ده** — القلب المشترك |
| `drivers/reset/amlogic/reset-meson.h` | الـ structs والـ API المشتركة |
| `drivers/reset/amlogic/reset-meson.c` | Platform driver — يدعم meson8b, A1, S4, T7 |
| `drivers/reset/amlogic/reset-meson-aux.c` | Auxiliary driver — reset لـ Audio subsystem |
| `drivers/reset/amlogic/reset-meson-audio-arb.c` | Audio memory arbiter reset |

**الـ Reset Framework العام:**

| الملف | الدور |
|-------|-------|
| `include/linux/reset-controller.h` | تعريف `reset_controller_dev` و `reset_control_ops` |
| `include/linux/reset.h` | الـ API اللي بيستخدمها كل driver يطلب reset |
| `drivers/reset/core.c` | تنفيذ الـ framework نفسه |

**الـ Device Tree Bindings:**

| الملف | الدور |
|-------|-------|
| `Documentation/devicetree/bindings/reset/amlogic,meson-reset.yaml` | وصف الـ hardware في الـ DT |

---

### ليه الكود انفصل لملف "common"؟

في البداية كان كل حاجة في ملف واحد. لما جاء الـ Audio subsystem محتاج reset controllers جوّاه عن طريق **Auxiliary Bus** (مش Platform Bus)، لازم يُعاد استخدام نفس الـ logic بدون copy-paste. الحل كان نقل الـ core logic لملف `reset-meson-common.c` وتـ export الدوال بـ `EXPORT_SYMBOL_NS_GPL` تحت namespace اسمه `"MESON_RESET"` — وبكده أي driver تاني يـ import الـ namespace ده يقدر يستخدم نفس الكود.
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

في أي SoC حديث زي Amlogic Meson، في عشرات الـ IP blocks — USB controller، Ethernet MAC، Video decoder، Audio engine، إلخ. كل block ده محتاج **reset line** — سلك واحد بيخلي الـ hardware يرجع لحالته الابتدائية.

المشكلة إن كل **driver** محتاج reset لـ IP بتاعه، بس الـ reset lines دي مش بتتحكم فيها الـ driver نفسه — بتتحكم فيها **وحدة مركزية** جوه الـ SoC بتسمى Reset Controller (أو Reset Manager). الوحدة دي بيبقى ليها register map في الـ MMIO بيتحكم في كل الـ reset lines بـ bits.

**السيناريو من غير Framework:**

```
usb_driver → يعمل read/write مباشر على register عنوان 0xC1104440
eth_driver → يعمل read/write مباشر على register عنوان 0xC1104444
audio_driver → يعمل read/write مباشر على register عنوان 0xC1104444
```

النتيجة:
- **كل driver** محتاج يعرف العنوان الفيزيائي للـ register.
- **race condition**: لو `eth_driver` و`audio_driver` وصلوا على نفس الـ register في نفس الوقت (لأن bits مختلفة في نفس الـ register)، هيكسروا بعض.
- **تكرار كود** في كل driver.
- **coupling** بين الـ driver وعنوان hardware معين.

---

### الحل — نهج الـ Kernel

الـ Kernel حل المشكلة بـ **Reset Controller Framework** اللي موجود في `drivers/reset/`. الفكرة:

1. **Provider**: الـ SoC vendor بيكتب driver واحد بيعرف كل الـ reset registers. بيسجل نفسه في الـ Framework كـ `reset_controller_dev`.
2. **Consumer**: أي driver محتاج reset بياخده عن طريق الـ Framework باستخدام API زي `reset_control_get()` — من غير ما يعرف أي register أو bit.
3. **Device Tree**: الـ DT هو اللي بيربط Consumer بـ Provider بـ phandle.

---

### التشبيه الواقعي — لوحة الكهرباء في مبنى

تخيل مبنى فيه **لوحة قواطع كهرباء (Distribution Board)** في غرفة الكهربائي.

| عنصر في التشبيه | المقابل في الـ Kernel |
|---|---|
| لوحة القواطع نفسها | `reset_controller_dev` — الـ Provider |
| كل قاطع (circuit breaker) | reset line واحدة بـ ID معين |
| الكهربائي اللي بيشغل/يوقف القواطع | `reset_control_ops` — الـ ops table |
| الشقة/المكتب اللي بيطلب إيقاف الكهرباء | الـ Consumer Driver |
| رقم الشقة في الطلب | `id` اللي بيتبعت في كل op call |
| مدير المبنى اللي بيوصّل الطلبات | Reset Controller Framework Core |
| عقد إيجار بيحدد أي قاطع لأي شقة | Device Tree phandle + `of_xlate` |
| غرفة الكهربائي مقفولة (مش محتاج تدخلها) | الـ regmap abstraction عن الـ registers |

**التفصيل**: الشقة (Consumer) مش بتكلم الكهربائي مباشرة ومش بتعرف رقم القاطع الفيزيائي. بتقول لمدير المبنى (Framework) "أنا شقة 5B، عايز أوقف الكهرباء". المدير بيرجع للعقد (DT) يشوف إن شقة 5B متربطة بقاطع رقم 12، وبيروح للكهربائي (ops) يقوله يوقف 12. الكهربائي عنده مفاتيح (registers) بيتحكم فيها.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Device Tree (.dts)                        │
│                                                                   │
│  reset: reset-controller@c1104440 {                              │
│      compatible = "amlogic,meson-gxbb-reset";                    │
│      #reset-cells = <1>;                                         │
│  };                                                               │
│                                                                   │
│  usb@c9000000 {                                                   │
│      resets = <&reset RESET_USB>;    ← phandle + ID              │
│  };                                                               │
└───────────────────────┬─────────────────────────────────────────┘
                        │  of_xlate() → translates phandle to ID
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│              Reset Controller Framework Core                      │
│          (drivers/reset/core.c — kernel generic layer)           │
│                                                                   │
│  - يحتفظ بـ linked list من كل الـ reset_controller_dev           │
│  - بيوفر API للـ consumers: reset_control_get/assert/deassert    │
│  - بيعمل reference counting على الـ reset controls              │
│  - بيمنع conflicts بين consumers                                 │
└──────────┬──────────────────────────────┬───────────────────────┘
           │ calls ops->assert/deassert    │ calls ops->reset
           ▼                              ▼
┌──────────────────────────────────────────────────────────────────┐
│              Provider: Meson Reset Driver                         │
│    (drivers/reset/amlogic/reset-meson-common.c)                  │
│                                                                   │
│  struct meson_reset {                                            │
│      struct reset_controller_dev rcdev;  ← يسجّل في الـ core    │
│      struct regmap *map;                 ← abstraction للـ regs  │
│      const struct meson_reset_param *param; ← hardware config   │
│  }                                                               │
│                                                                   │
│  meson_reset_ops / meson_reset_toggle_ops                        │
└──────────┬───────────────────────────────────────────────────────┘
           │ regmap_write / regmap_update_bits
           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Regmap Abstraction Layer                        │
│                  (include/linux/regmap.h)                         │
│   - يخبي تفاصيل الـ MMIO access (locking, caching, endianness)   │
└──────────┬───────────────────────────────────────────────────────┘
           │ raw MMIO read/write
           ▼
┌──────────────────────────────────────────────────────────────────┐
│              Amlogic Meson SoC Hardware                           │
│         Reset Controller Register Bank                            │
│                                                                   │
│  Offset 0x00: [bit31]...[bit0]  → resets 31..0                   │
│  Offset 0x04: [bit31]...[bit0]  → resets 63..32                  │
│  ...                                                              │
│  Level Offset 0x7C: assert/deassert with persistent state        │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      Consumers                                    │
│   usb_driver    eth_driver    audio_driver    video_driver        │
│       │              │              │               │             │
│       └──────────────┴──────────────┴───────────────┘           │
│              كلهم بيتكلموا مع الـ Framework Core فقط             │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ Framework بيعمل **فصل كامل** بين:

- **"أنا عايز أعمل reset لـ USB"** — ده شغل الـ Consumer، بيعبّر عنه بـ `id` رقمي مجرد.
- **"الـ reset لـ USB يعني أكتب bit 5 في register 0x04"** — ده شغل الـ Provider، مخبّي تماماً.

الـ abstraction المحوري هو struct تانين:

**أول واحد — `struct reset_control_ops`:**

```c
struct reset_control_ops {
    /* one-shot pulse: assert ثم deassert أوتوماتيك (hardware-driven) */
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);

    /* persistent: يثبّت الـ reset line في حالة assert (active) */
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);

    /* persistent: يرفع الـ reset line (خروج من الـ reset) */
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);

    /* بيرجع الحالة الحالية: 1 = in reset, 0 = normal */
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

الـ `id` هنا هو رقم مجرد بيمثل **رقم الـ reset line** — الـ Provider هو اللي بيحوّله لـ register offset + bit number.

**تاني واحد — `struct reset_controller_dev`:**

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;    /* الـ vtable */
    struct module *owner;                   /* ownership للـ module */
    struct list_head list;                  /* في الـ global list */
    struct list_head reset_control_head;    /* الـ consumers المتصلين */
    struct device *dev;                     /* الـ device model */
    struct device_node *of_node;            /* الـ DT node (للـ lookup) */
    int (*of_xlate)(...);                   /* DT phandle → ID */
    unsigned int nr_resets;                 /* عدد الـ reset lines */
};
```

---

### علاقة الـ Structs ببعض

```
struct meson_reset
┌─────────────────────────────────────────┐
│  param ──────────────────────────────►  struct meson_reset_param    │
│                                         ┌────────────────────────┐  │
│                                         │ reset_ops → ops table  │  │
│                                         │ reset_num = 256        │  │
│                                         │ reset_offset = 0x00    │  │
│                                         │ level_offset = 0x7C    │  │
│                                         │ level_low_reset = true │  │
│                                         └────────────────────────┘  │
│                                                                      │
│  map ────────────────────────────────►  struct regmap               │
│                                         (MMIO abstraction)          │
│                                                                      │
│  rcdev ──────────────────────────────►  struct reset_controller_dev │
│   ┌──────────────────────────────────   ┌────────────────────────┐  │
│   │  ops ──────────────────────────────►│ reset_control_ops      │  │
│   │  nr_resets = 256                    │ .reset = meson_reset_  │  │
│   │  of_node → DT node                 │ .assert = ...          │  │
│   │  list → global rcdev list          │ .deassert = ...        │  │
│   └─────────────────────────────────── │ .status = ...          │  │
│                                         └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**container_of trick**: لما الـ Framework Core بيستدعي `ops->assert(rcdev, id)`، الـ Provider بيعمل:

```c
struct meson_reset *data =
    container_of(rcdev, struct meson_reset, rcdev);
```

بيوصل للـ `meson_reset` الأم من الـ `rcdev` المدفون جواه — ده pattern أساسي في الـ kernel لعمل OOP بـ C.

---

### ليه فيه اتنين ops tables؟

```c
/* للـ hardware اللي عنده dedicated reset pulse register */
const struct reset_control_ops meson_reset_ops = {
    .reset    = meson_reset_reset,        /* write single bit → pulse */
    .assert   = meson_reset_assert,
    .deassert = meson_reset_deassert,
    .status   = meson_reset_status,
};

/* للـ hardware اللي مش عنده pulse register — بيعمل toggle يدوي */
const struct reset_control_ops meson_reset_toggle_ops = {
    .reset    = meson_reset_level_toggle, /* assert ثم deassert بـ software */
    .assert   = meson_reset_assert,
    .deassert = meson_reset_deassert,
    .status   = meson_reset_status,
};
```

بعض الـ Meson SoCs عندها register خاص بيعمل **self-clearing pulse** لما تكتب فيه — بمجرد الكتابة الـ hardware بيعمل assert/deassert وحده. ده `meson_reset_ops.reset`.

تانية بعض الـ SoCs مش عندها الـ feature دي — لازم الـ software يعمل assert يدوياً ثم deassert. ده `meson_reset_toggle_ops.reset` = `meson_reset_level_toggle`.

---

### حساب الـ Register Offset والـ Bit

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                        unsigned long id,
                                        unsigned int *offset,
                                        unsigned int *bit)
{
    unsigned int stride = regmap_get_reg_stride(data->map);
    /* stride = حجم كل register بالـ bytes، عادة 4 (32-bit registers) */

    *offset = (id / (stride * BITS_PER_BYTE)) * stride;
    /* مثال: id=40, stride=4, BITS_PER_BYTE=8
       offset = (40 / 32) * 4 = 1 * 4 = 4 → Register الثاني */

    *bit = id % (stride * BITS_PER_BYTE);
    /* bit = 40 % 32 = 8 → Bit رقم 8 في الـ register ده */
}
```

يعني الـ reset lines بتتوزع على registers بالتسلسل:
- Reset 0-31 → Register offset 0
- Reset 32-63 → Register offset 4
- Reset 64-95 → Register offset 8
- وهكذا...

---

### الـ level_low_reset — Polarity Inversion

```c
assert ^= data->param->level_low_reset;
```

بعض الـ hardware بيكون **active-low reset**: يعني عشان تعمل assert للـ reset، لازم تكتب **0** مش 1. الـ flag ده بيعكس المنطق تلقائياً:

| `level_low_reset` | عايز assert | اللي بيتكتب |
|---|---|---|
| false | true | BIT(bit) = 1 |
| false | false | 0 |
| true | true → false بعد XOR | 0 |
| true | false → true بعد XOR | BIT(bit) = 1 |

الـ Framework Consumer مش محتاج يعرف أي حاجة عن الـ polarity دي.

---

### ملكية الـ Framework — ايه بيمتلكه وايه بيفوّضه؟

| المهمة | مين المسؤول؟ |
|---|---|
| تسجيل الـ reset controller في الـ global list | **Framework Core** (`devm_reset_controller_register`) |
| الـ reference counting على الـ reset controls | **Framework Core** |
| التأكد إن Consumer واحد بس يـassert | **Framework Core** |
| ربط Consumer بـ Provider عن طريق الـ DT | **Framework Core** + `of_xlate` |
| معرفة الـ register offset وعمل الـ bit manipulation | **Provider (Meson Driver)** |
| معرفة الـ polarity (active-high/low) | **Provider (Meson Driver)** |
| الـ MMIO access اللي فيه locking | **Regmap Layer** |
| تحديد أي reset ID يخص أي IP block | **Device Tree** |
| طلب الـ reset والانتظار | **Consumer Driver** |

---

### الـ devm_ Pattern

```c
data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
...
return devm_reset_controller_register(dev, &data->rcdev);
```

الـ `devm_` prefix معناه **device-managed resource**. لما الـ device بيتشال (unbind أو error)، الـ kernel بيحرر الـ memory ويـunregister الـ controller أوتوماتيك من غير ما الـ driver يكتب cleanup code يدوي. ده بيلغي الـ `.remove` callback في أغلب الحالات.

---

### الـ EXPORT_SYMBOL_NS_GPL — الـ Module Namespace

```c
EXPORT_SYMBOL_NS_GPL(meson_reset_ops, "MESON_RESET");
EXPORT_SYMBOL_NS_GPL(meson_reset_controller_register, "MESON_RESET");
```

الـ `NS` هنا = **Namespace**. الـ symbols مش export عامة — بس الـ modules اللي بتعلن:

```c
MODULE_IMPORT_NS("MESON_RESET");
```

هي اللي تقدر تستخدمها. ده بيمنع أي driver عشوائي من استخدام الـ Meson-specific API — بيفرض نوع من الـ encapsulation على مستوى الـ kernel modules.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

| العنصر | النوع | القيم / المعنى |
|--------|-------|----------------|
| `level_low_reset` | `bool` في `meson_reset_param` | `true` = الـ reset active-low (البت `0` يعني reset)، `false` = active-high |
| `CONFIG_RESET_CONTROLLER` | Kconfig option | لو مش enabled، كل دوال الـ reset controller بتبقى stubs فارغة |
| `GFP_KERNEL` | allocation flag | يُستخدم في `devm_kzalloc` — يسمح بالـ sleep أثناء التخصيص |
| `BIT(bit)` | macro | يحوّل رقم البت لـ bitmask: `1 << bit` |
| `BITS_PER_BYTE` | constant | = 8، يُستخدم في حساب الـ offset |
| `EXPORT_SYMBOL_NS_GPL` | macro | يصدّر الرمز للـ namespace المسمى `"MESON_RESET"` بشرط GPL |
| `MODULE_IMPORT_NS` | macro | يعلن إن الـ module ده بيستورد نفس الـ namespace |

---

### كل الـ Structs المهمة

#### 1. `struct meson_reset`

**الغرض:** الـ private state الخاص بـ driver الـ reset في Amlogic Meson. بيربط بين الـ hardware parameters والـ regmap والـ reset controller المسجّل في الـ kernel.

```c
struct meson_reset {
    const struct meson_reset_param *param;  // hardware config (offsets, polarity, count)
    struct reset_controller_dev rcdev;      // embedded reset controller — الـ kernel يشوفه
    struct regmap *map;                     // abstracted register access
};
```

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `param` | `const struct meson_reset_param *` | بوينتر للـ config الثابت الخاص بكل chip variant |
| `rcdev` | `struct reset_controller_dev` | الـ struct اللي الـ reset core framework بيشتغل بيه — مـ embedded مش بوينتر |
| `map` | `struct regmap *` | الـ regmap اللي بيعمل read/write على الـ hardware registers |

**الاتصال بالـ structs التانية:**
- `param` ← `meson_reset_param`
- `rcdev` ← `reset_controller_dev` (kernel core)
- `map` ← `regmap` (regmap framework)

---

#### 2. `struct meson_reset_param`

**الغرض:** الـ hardware description الخاصة بكل chip variant (مثلاً GX، AXG، SM1). كل chip driver بيعرّف instance ثابت منها.

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; // الـ ops table (toggle أو standard)
    unsigned int reset_num;    // عدد الـ reset lines المدعومة
    unsigned int reset_offset; // base offset لـ registers الـ pulse reset
    unsigned int level_offset; // base offset لـ registers الـ level reset
    bool level_low_reset;      // هل الـ assert بيكون بـ 0 ولا 1؟
};
```

| الفيلد | الشرح |
|--------|-------|
| `reset_ops` | بيشاور على `meson_reset_ops` أو `meson_reset_toggle_ops` |
| `reset_num` | بيتحط في `rcdev.nr_resets` |
| `reset_offset` | الـ base address للـ pulse reset registers (write-to-reset) |
| `level_offset` | الـ base address للـ level-sensitive reset registers |
| `level_low_reset` | لو `true`، الـ XOR في `meson_reset_level` و`meson_reset_status` بيقلب المنطق |

---

#### 3. `struct reset_controller_dev`

**الغرض:** الـ struct الأساسي في الـ kernel reset framework. الـ Meson driver بيعبّيه وبيسجّله.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;       // function pointers للـ assert/deassert/reset/status
    struct module *owner;                      // الـ module اللي سجّله
    struct list_head list;                     // بيتحط في قايمة عالمية للـ controllers
    struct list_head reset_control_head;       // قايمة الـ consumers اللي طلبوا reset lines
    struct device *dev;                        // الـ device المرتبط بيه
    struct device_node *of_node;               // الـ DT node — consumers بيبحثوا بيه
    const struct of_phandle_args *of_args;     // بديل لـ of_node في حالة GPIO resets
    int of_reset_n_cells;                      // عدد خلايا الـ specifier في DT
    int (*of_xlate)(...);                      // تحويل DT specifier لـ ID رقمي
    unsigned int nr_resets;                    // عدد الـ reset lines
};
```

---

#### 4. `struct reset_control_ops`

**الغرض:** جدول الـ function pointers اللي الـ reset framework بيستدعيها. الـ Meson driver بيعبّي نسختين منها.

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

**النسختين في الـ Meson driver:**

| الـ ops table | `reset` | `assert` | `deassert` | `status` |
|---------------|---------|----------|------------|---------|
| `meson_reset_ops` | `meson_reset_reset` (pulse) | `meson_reset_assert` | `meson_reset_deassert` | `meson_reset_status` |
| `meson_reset_toggle_ops` | `meson_reset_level_toggle` | `meson_reset_assert` | `meson_reset_deassert` | `meson_reset_status` |

الفرق: `meson_reset_ops` بيستخدم الـ pulse reset registers مباشرة، أما `meson_reset_toggle_ops` فبيعمل assert ثم deassert يدوياً على الـ level registers.

---

### مخطط علاقات الـ Structs (ASCII)

```
Platform Driver (e.g., reset-meson-gx.c)
          │
          │ passes: dev, regmap, meson_reset_param*
          ▼
meson_reset_controller_register()
          │
          │ allocates (devm)
          ▼
┌─────────────────────────────────────┐
│         struct meson_reset          │
│                                     │
│  param ──────────────────────────► struct meson_reset_param  │
│  │         reset_ops ─────────────► struct reset_control_ops │
│  │         reset_num                                          │
│  │         reset_offset                                       │
│  │         level_offset                                       │
│  │         level_low_reset                                    │
│                                                               │
│  map ────────────────────────────► struct regmap             │
│  │                (regmap framework → hardware registers)     │
│                                                               │
│  rcdev ◄── embedded ────────────► struct reset_controller_dev│
│             ops ───────────────► struct reset_control_ops    │
│             owner (module)                                    │
│             nr_resets                                         │
│             of_node ──────────► struct device_node (DT)      │
│             list ─────────────► global rcdev list (kernel)   │
│             reset_control_head► consumer reset_control list  │
└─────────────────────────────────────┘
          │
          │ devm_reset_controller_register()
          ▼
     Reset Framework Core
     (consumers call reset_control_reset(), etc.)
```

---

### مخطط دورة الحياة (Lifecycle)

```
[Boot / Module Init]
        │
        ▼
Platform driver probe()
        │
        ├─ regmap_init()  ──────────────────► struct regmap (جاهز)
        │
        ├─ meson_reset_controller_register(dev, map, param)
        │       │
        │       ├─ devm_kzalloc() ─────────► struct meson_reset (مخصّص)
        │       │
        │       ├─ data->param    = param   (hardware config)
        │       ├─ data->map      = map     (regmap handle)
        │       ├─ rcdev.owner    = driver module
        │       ├─ rcdev.nr_resets= param->reset_num
        │       ├─ rcdev.ops      = param->reset_ops
        │       ├─ rcdev.of_node  = dev->of_node
        │       │
        │       └─ devm_reset_controller_register(dev, &rcdev)
        │               │
        │               └─ يضيف الـ rcdev للـ global list
        │                  consumers دلوقتي يقدروا يطلبوا reset lines
        │
        ▼
[Runtime — Consumer يطلب reset]
        │
        ├─ reset_control_reset(rstc)   ──► ops->reset()    (pulse)
        ├─ reset_control_assert(rstc)  ──► ops->assert()   (level high/low)
        ├─ reset_control_deassert(rstc)──► ops->deassert() (level release)
        └─ reset_control_status(rstc)  ──► ops->status()   (read state)
        │
        ▼
[Driver Unload / Device Remove]
        │
        └─ devm cleanup تلقائياً:
               - devm_reset_controller_register unregisters rcdev
               - devm_kzalloc يحرر struct meson_reset
               (مفيش teardown يدوي محتاجه الـ driver)
```

---

### مخطط Call Flow

#### A. الـ Pulse Reset (`meson_reset_reset`)

```
consumer: reset_control_reset(rstc)
  → reset framework: rcdev->ops->reset(rcdev, id)
    → meson_reset_reset(rcdev, id)
      → container_of(rcdev, struct meson_reset, rcdev)  [يجيب data]
        → meson_reset_offset_and_bit(data, id, &offset, &bit)
          → regmap_get_reg_stride(data->map)             [يجيب stride]
          → offset = (id / (stride*8)) * stride
          → bit    = id % (stride*8)
        → offset += data->param->reset_offset            [يضيف base]
        → regmap_write(data->map, offset, BIT(bit))
          → hardware register write (pulse reset)
```

#### B. الـ Level Assert (`meson_reset_assert`)

```
consumer: reset_control_assert(rstc)
  → reset framework: rcdev->ops->assert(rcdev, id)
    → meson_reset_assert(rcdev, id)
      → meson_reset_level(rcdev, id, assert=true)
        → container_of(rcdev, struct meson_reset, rcdev)
          → meson_reset_offset_and_bit(data, id, &offset, &bit)
          → offset += data->param->level_offset
          → assert ^= data->param->level_low_reset       [XOR لقلب المنطق لو active-low]
          → regmap_update_bits(map, offset, BIT(bit), assert ? BIT(bit) : 0)
            → hardware register RMW (read-modify-write)
```

#### C. الـ Level Toggle (`meson_reset_level_toggle`)

```
consumer: reset_control_reset(rstc)   [مع meson_reset_toggle_ops]
  → reset framework: rcdev->ops->reset(rcdev, id)
    → meson_reset_level_toggle(rcdev, id)
      → meson_reset_assert(rcdev, id)    [assert=true  → level HIGH أو LOW حسب polarity]
        → meson_reset_level(..., true)
          → regmap_update_bits(...)      [يسيت البت]
      → meson_reset_deassert(rcdev, id) [assert=false → level released]
        → meson_reset_level(..., false)
          → regmap_update_bits(...)      [يكلير البت]
```

#### D. قراءة الحالة (`meson_reset_status`)

```
consumer: reset_control_status(rstc)
  → reset framework: rcdev->ops->status(rcdev, id)
    → meson_reset_status(rcdev, id)
      → container_of(rcdev, struct meson_reset, rcdev)
        → meson_reset_offset_and_bit(data, id, &offset, &bit)
        → offset += data->param->level_offset
        → regmap_read(map, offset, &val)          [يقرأ الـ register]
        → val = !!(BIT(bit) & val)                [يعزل البت المطلوب → 0 أو 1]
        → return val ^ data->param->level_low_reset  [XOR لتصحيح الـ polarity]
          → 1 = في reset، 0 = مش في reset (بغض النظر عن الـ polarity)
```

---

### حساب الـ Offset والـ Bit

**المعادلة الجوهرية:**

```
stride = regmap_get_reg_stride(map)   // عادةً 4 bytes (32-bit registers)

offset = (id / (stride * 8)) * stride
bit    = id % (stride * 8)
```

**مثال عملي** (stride = 4، أي registers 32-bit):

| Reset ID | offset (bytes) | bit |
|----------|---------------|-----|
| 0        | 0x00          | 0   |
| 7        | 0x00          | 7   |
| 31       | 0x00          | 31  |
| 32       | 0x04          | 0   |
| 63       | 0x04          | 31  |
| 64       | 0x08          | 0   |

بعدين بيتضاف `reset_offset` أو `level_offset` كـ base لنقل كل ده للمنطقة الصح في الـ address space.

---

### استراتيجية الـ Locking

الـ driver ده **مفيش فيه explicit locks** خاصة بيه — وده مقصود.

| المستوى | المسؤول عن الـ locking | الآلية |
|---------|----------------------|--------|
| **Regmap** | `regmap` framework نفسه | كل `regmap_read/write/update_bits` بيتم تحت lock داخلي في الـ regmap (عادةً `spinlock` أو `mutex` حسب إعداد الـ regmap) |
| **Reset framework** | `reset_controller_dev` | الـ framework بيحمي الـ `reset_control_head` list بـ `mutex` داخلي |
| **`meson_reset` struct** | Read-only بعد الـ probe | الـ `param` و`map` بيتكتبوا مرة واحدة في `meson_reset_controller_register` وبعدين read-only، مفيش حاجة تتغير |
| **الـ `assert ^= level_low_reset`** | No lock needed | عملية على variable محلي على الـ stack — thread-safe بطبيعتها |

**ترتيب الـ locking** (lock ordering) من الأعلى للأسفل:

```
reset framework mutex (reset_control)
        │
        ▼
regmap internal lock (spinlock/mutex)
        │
        ▼
hardware register access (atomic بالنسبة للـ CPU)
```

مفيش خطر deadlock لأن الـ driver نفسه مبيمسكش أي lock خاص بيه — كل حاجة متوكلة على الـ frameworks اللي تحته.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs (Cheatsheet)

#### Category: Helpers

| Function | Signature Summary | الغرض |
|---|---|---|
| `meson_reset_offset_and_bit` | `(data, id, *offset, *bit)` | حساب الـ register offset والـ bit index من الـ reset ID |

#### Category: Reset Operations (Ops Callbacks)

| Function | الغرض | نوع الـ Reset |
|---|---|---|
| `meson_reset_reset` | Self-deasserting pulse reset عبر write-only register | Pulse (level-agnostic) |
| `meson_reset_level` | Assert أو deassert خط الـ reset عبر read-modify-write | Level-based |
| `meson_reset_assert` | Wrapper يستدعي `meson_reset_level(..., true)` | Level assert |
| `meson_reset_deassert` | Wrapper يستدعي `meson_reset_level(..., false)` | Level deassert |
| `meson_reset_level_toggle` | Assert ثم deassert تسلسلياً | Level toggle |
| `meson_reset_status` | قراءة حالة الـ reset line الحالية | Status query |

#### Category: Registration

| Function | الغرض |
|---|---|
| `meson_reset_controller_register` | تخصيص وتسجيل الـ reset controller مع الـ kernel |

#### Category: Exported Ops Tables

| Symbol | الغرض |
|---|---|
| `meson_reset_ops` | الـ ops table للـ hardware ذو الـ dedicated pulse reset register |
| `meson_reset_toggle_ops` | الـ ops table للـ hardware الذي ينفذ الـ reset بـ assert/deassert تسلسلي |

---

### Group 1: Helpers — حساب الـ Address والـ Bit

هذه المجموعة مسؤولة عن ترجمة الـ linear reset ID إلى عنوان register فعلي وموضع bit داخله، مع مراعاة الـ register stride الذي يحدده الـ regmap backend (8-bit أو 16-bit أو 32-bit).

---

#### `meson_reset_offset_and_bit`

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
```

**ما تفعله:**
الدالة تحسب الـ byte offset داخل الـ register map والـ bit position اللي يقابل الـ reset line المحددة بالـ `id`. بتستخدم الـ `regmap_get_reg_stride` عشان تحترم اتساق الـ register width للـ bus (AXI/APB) سواء كان 8 أو 32 bit.

**المعاملات:**
- `data` — pointer للـ `meson_reset` private struct اللي فيه الـ `regmap` والـ `param`.
- `id` — رقم الـ reset line (linear index من 0).
- `offset` — output: الـ byte offset للـ register اللي فيه الـ bit ده.
- `bit` — output: رقم الـ bit داخل الـ register.

**القيمة المُرجعة:** `void` — النتيجة في `*offset` و `*bit`.

**التفاصيل:**

```
stride = regmap_get_reg_stride(map)   // e.g., 4 for 32-bit registers
bits_per_reg = stride * 8             // e.g., 32

offset = (id / bits_per_reg) * stride // which register
bit    = id % bits_per_reg             // which bit within that register
```

مثال: لو `id = 35` وـ `stride = 4` (32-bit regs):
- `bits_per_reg = 32`
- `offset = (35/32)*4 = 4` → الـ register التاني
- `bit = 35%32 = 3`

الـ **stride** مهم لأن الـ regmap يعامل الـ offset كـ byte address، فلازم نضرب في الـ stride عشان منقفزش بـ byte واحدة بين registers عرضها 4 bytes.

**من يستدعيها:** كل الـ ops callbacks (`meson_reset_reset`, `meson_reset_level`, `meson_reset_status`) — دايماً أول خطوة.

---

### Group 2: Reset Operations Callbacks

الـ ops callbacks هي اللي بتنفذ الـ reset فعلياً على الـ hardware. الـ kernel reset framework بيستدعيها من خلال `struct reset_control_ops`. في Meson، في نموذجين للـ reset:

1. **Pulse reset** — register مخصص للـ reset، الكتابة عليه بـ `1` على الـ bit بتعمل pulse تلقائي.
2. **Level reset** — register مشترك بيتحكم في assert/deassert بشكل صريح.

---

#### `meson_reset_reset`

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                             unsigned long id)
```

**ما تفعله:**
بتنفذ self-deasserting reset عن طريق كتابة `1` على الـ bit المقابل للـ reset line في الـ dedicated reset register. الـ hardware بينهي الـ reset pulse تلقائياً بعد عدد محدد من الـ clock cycles، ما في حاجة لـ deassert صريح.

**المعاملات:**
- `rcdev` — pointer للـ `reset_controller_dev`، بيتحول لـ `meson_reset` عبر `container_of`.
- `id` — رقم الـ reset line.

**القيمة المُرجعة:** `0` عند النجاح، أو error code سلبي من `regmap_write`.

**التفاصيل:**
- بتستخدم `regmap_write` مش `regmap_update_bits` — ده **write-only register** وكتابة `BIT(bit)` بتعمل pulse فقط على الـ line المحددة.
- الـ `offset` بيتضاف عليه `param->reset_offset` عشان الـ reset register في بداية معينة في الـ register map.
- **تحذير:** ده مش thread-safe على مستوى الـ register كله لو bits مختلفة لـ resets مختلفة في نفس الوقت — لكن الـ hardware صمّمه كـ write-to-pulse فالـ bits مستقلة.

**Pseudocode:**
```
offset, bit = meson_reset_offset_and_bit(id)
offset += param->reset_offset
regmap_write(map, offset, BIT(bit))
```

**من يستدعيها:** الـ reset framework عبر `reset_control_reset()` من drivers أخرى.

---

#### `meson_reset_level`

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
```

**ما تفعله:**
الدالة الأساسية اللي بتتحكم في level-based reset. بتعمل read-modify-write على الـ level register عشان تضبط أو تمسح الـ bit المقابل لـ reset line معينة. بتراعي الـ polarity: لو الـ hardware بيعمل reset على `LOW` (أي `level_low_reset = true`)، بتعكس منطق الـ assert.

**المعاملات:**
- `rcdev` — الـ reset controller device.
- `id` — رقم الـ reset line.
- `assert` — `true` لـ assert (تفعيل الـ reset)، `false` لـ deassert.

**القيمة المُرجعة:** `0` عند النجاح أو error code من `regmap_update_bits`.

**التفاصيل:**
- الـ **XOR** مع `level_low_reset`:
  ```
  assert ^= data->param->level_low_reset;
  ```
  لو `level_low_reset = true` وـ `assert = true`، النتيجة `false` → الـ bit بيتمسح (0) عشان يعمل reset على LOW.
  لو `level_low_reset = false` وـ `assert = true`، النتيجة `true` → الـ bit بيتضبط (1).

- `regmap_update_bits` آمن لأنه بيعمل read-modify-write مع الـ locking اللي بيوفره الـ regmap framework.

**Pseudocode:**
```
offset, bit = meson_reset_offset_and_bit(id)
offset += param->level_offset

assert = assert XOR level_low_reset

mask  = BIT(bit)
value = assert ? BIT(bit) : 0
regmap_update_bits(map, offset, mask, value)
```

**من يستدعيها:** `meson_reset_assert` وـ `meson_reset_deassert` — مش بتتسمى مباشرة من الـ framework.

---

#### `meson_reset_assert`

```c
static int meson_reset_assert(struct reset_controller_dev *rcdev,
                              unsigned long id)
```

**ما تفعله:**
Thin wrapper بيستدعي `meson_reset_level` بـ `assert = true`. تفعيل الـ reset (put device in reset state).

**المعاملات:**
- `rcdev` — الـ reset controller.
- `id` — رقم الـ reset line.

**القيمة المُرجعة:** نفس قيمة `meson_reset_level`.

**من يستدعيها:** الـ framework عبر `reset_control_assert()`.

---

#### `meson_reset_deassert`

```c
static int meson_reset_deassert(struct reset_controller_dev *rcdev,
                                unsigned long id)
```

**ما تفعله:**
Thin wrapper بيستدعي `meson_reset_level` بـ `assert = false`. تحرير الـ device من حالة الـ reset.

**المعاملات:**
- `rcdev` — الـ reset controller.
- `id` — رقم الـ reset line.

**القيمة المُرجعة:** نفس قيمة `meson_reset_level`.

**من يستدعيها:** الـ framework عبر `reset_control_deassert()`.

---

#### `meson_reset_level_toggle`

```c
static int meson_reset_level_toggle(struct reset_controller_dev *rcdev,
                                    unsigned long id)
```

**ما تفعله:**
بتنفذ reset sequence كاملة على hardware ملوش dedicated pulse register: بتعمل assert ثم deassert بالترتيب. لو الـ assert فشل، بيرجع الـ error فوراً وما بيكملش.

**المعاملات:**
- `rcdev` — الـ reset controller.
- `id` — رقم الـ reset line.

**القيمة المُرجعة:** `0` عند النجاح، أو error code من أول عملية فاشلة.

**التفاصيل:**
- ده هو الـ `.reset` callback في `meson_reset_toggle_ops`.
- ما فيش delay بين الـ assert والـ deassert — الـ hardware والـ regmap latency كافيين في Amlogic SoCs.
- لو `assert` نجحت وـ `deassert` فشلت، الـ device هيفضل في حالة reset — ده error path خطير لكنه نادر جداً.

**Pseudocode:**
```
ret = meson_reset_assert(rcdev, id)
if ret != 0:
    return ret
return meson_reset_deassert(rcdev, id)
```

**من يستدعيها:** الـ framework عبر `reset_control_reset()` لما الـ controller مسجّل بـ `meson_reset_toggle_ops`.

---

#### `meson_reset_status`

```c
static int meson_reset_status(struct reset_controller_dev *rcdev,
                              unsigned long id)
```

**ما تفعله:**
بتقرأ الحالة الحالية لخط الـ reset من الـ level register وبترجعها كـ `1` (asserted) أو `0` (deasserted)، مع مراعاة الـ polarity عبر XOR مع `level_low_reset`.

**المعاملات:**
- `rcdev` — الـ reset controller.
- `id` — رقم الـ reset line.

**القيمة المُرجعة:**
- `1` → الـ reset مفعّل (asserted).
- `0` → الـ reset غير مفعّل (deasserted).
- (ما بترجعش error codes من الـ `regmap_read` — ده limitation.)

**التفاصيل:**
- `!!` بتضمن إن الـ result دايماً `0` أو `1` بغض النظر عن الـ bit position.
- XOR مع `level_low_reset` بيعكس المنطق: لو الـ bit = 0 وـ `level_low_reset = true`، يعني الـ reset asserted.
- **ملاحظة:** `regmap_read` بيرجع error لكن الدالة ما بتتحققش منه — الـ `val` هيفضل `0` لو فشل القراءة.

**Pseudocode:**
```
offset, bit = meson_reset_offset_and_bit(id)
offset += param->level_offset

regmap_read(map, offset, &val)
val = !!(val & BIT(bit))       // normalize to 0 or 1
return val XOR level_low_reset  // adjust for polarity
```

**من يستدعيها:** الـ framework عبر `reset_control_status()`.

---

### Group 3: Registration

#### `meson_reset_controller_register`

```c
int meson_reset_controller_register(struct device *dev,
                                    struct regmap *map,
                                    const struct meson_reset_param *param)
```

**ما تفعله:**
الدالة الوحيدة المُصدَّرة للـ platform drivers. بتخصص الـ `meson_reset` private struct، بتملأ الـ `reset_controller_dev`، وبتسجله مع الـ kernel reset framework. كل التخصيصات بـ `devm_` فالتنظيف تلقائي عند الـ device unbind.

**المعاملات:**
- `dev` — الـ platform device، بيُستخدم للـ devm allocation والـ of_node والـ driver owner.
- `map` — الـ regmap instance اللي بيمثل الـ register space للـ reset controller.
- `param` — الـ hardware-specific parameters (offsets، عدد الـ resets، polarity، ops table).

**القيمة المُرجعة:** `0` عند النجاح، `-ENOMEM` لو فشل الـ allocation، أو error code من `devm_reset_controller_register`.

**التفاصيل:**

```c
// حقول الـ rcdev اللي بتتملأ:
data->rcdev.owner     = dev->driver->owner;   // module ownership
data->rcdev.nr_resets = param->reset_num;     // عدد الـ reset lines
data->rcdev.ops       = param->reset_ops;     // meson_reset_ops أو meson_reset_toggle_ops
data->rcdev.of_node   = dev->of_node;         // للـ DT xlate
```

- الـ `of_xlate` مش بتتضبط صراحة → الـ framework بيستخدم `of_reset_simple_xlate` الـ default اللي بيفسر الـ DT specifier كـ index مباشر.
- `devm_reset_controller_register` بيضيف الـ rcdev لـ global list في الـ reset framework وبيعمل register لكل الـ `nr_resets` lines.
- لما الـ device بيتشال (remove/unbind)، الـ devm framework تلقائياً بيستدعي `reset_controller_unregister`.

**Pseudocode:**
```
data = devm_kzalloc(dev, sizeof(*data))
if !data: return -ENOMEM

data->param = param
data->map   = map
rcdev.owner     = dev->driver->owner
rcdev.nr_resets = param->reset_num
rcdev.ops       = param->reset_ops
rcdev.of_node   = dev->of_node

return devm_reset_controller_register(dev, &data->rcdev)
```

**من يستدعيها:** الـ platform-specific probe functions في ملفات زي `reset-meson-axg.c`, `reset-meson-gx.c` وغيرها.

---

### Group 4: Exported Ops Tables

#### `meson_reset_ops` و `meson_reset_toggle_ops`

```c
const struct reset_control_ops meson_reset_ops = {
    .reset    = meson_reset_reset,
    .assert   = meson_reset_assert,
    .deassert = meson_reset_deassert,
    .status   = meson_reset_status,
};

const struct reset_control_ops meson_reset_toggle_ops = {
    .reset    = meson_reset_level_toggle,
    .assert   = meson_reset_assert,
    .deassert = meson_reset_deassert,
    .status   = meson_reset_status,
};
```

الفرق الوحيد بين الاتنين هو الـ `.reset` callback:

| | `meson_reset_ops` | `meson_reset_toggle_ops` |
|---|---|---|
| `.reset` | `meson_reset_reset` (pulse via dedicated reg) | `meson_reset_level_toggle` (assert+deassert) |
| `.assert` | `meson_reset_assert` | `meson_reset_assert` |
| `.deassert` | `meson_reset_deassert` | `meson_reset_deassert` |
| `.status` | `meson_reset_status` | `meson_reset_status` |
| متى تستخدم | Hardware عنده `reset_offset` register مستقل | Hardware بيستخدم `level_offset` فقط |

كلاهما مُصدَّران بـ `EXPORT_SYMBOL_NS_GPL` داخل الـ namespace `"MESON_RESET"` — الـ platform drivers اللي تستخدمهم لازم تعمل `MODULE_IMPORT_NS("MESON_RESET")`.

---

### الـ Data Flow الكامل

```
Platform Driver probe()
        │
        ▼
meson_reset_controller_register(dev, map, param)
        │
        ├─► devm_kzalloc → struct meson_reset { param, map, rcdev }
        │
        └─► devm_reset_controller_register
                │
                └─► kernel reset framework list
                            │
                    Consumer calls reset_control_reset(rstc)
                            │
                            ▼
                    rcdev->ops->reset(rcdev, id)
                            │
                   ┌────────┴────────┐
                   ▼                 ▼
          meson_reset_reset    meson_reset_level_toggle
          (write BIT to        (assert then deassert
           reset register)      level register)
                   │                 │
                   └────────┬────────┘
                            ▼
                meson_reset_offset_and_bit
                (id → offset + bit)
                            │
                            ▼
                   regmap_write / regmap_update_bits
                   (actual hardware register access)
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده هو `reset-meson-common.c` — اللي بيوفر الـ core logic لـ reset controller على chips الـ Amlogic Meson. بيستخدم `regmap` للوصول للـ registers وبيسجّل نفسه كـ `reset_controller_dev` في الـ kernel. الـ debugging هنا بيتمحور حوالين ٣ محاور: هل الـ reset بينفّذ صح؟ هل الـ register بيتكتب صح؟ وهل الـ Device Tree صح؟

---

### Software Level

#### 1. debugfs entries

الـ reset subsystem بيعمل entries في `/sys/kernel/debug/reset/`:

```bash
# اعرض كل الـ reset controllers المسجّلين
ls /sys/kernel/debug/reset/

# اقرأ حالة reset معيّن (بيتطلب CONFIG_RESET_CONTROLLER=y)
cat /sys/kernel/debug/reset/meson_reset/reset0
```

الـ kernel reset subsystem بيعمل entries زي:
```
/sys/kernel/debug/reset/<controller-name>/
    ├── reset<N>        # حالة كل reset line
```

كمان ممكن تستخدم:
```bash
# اعرض الـ regmap registers مباشرة
ls /sys/kernel/debug/regmap/
# هتلاقي entry باسم الـ device زي:
ls /sys/kernel/debug/regmap/meson-reset*/
cat /sys/kernel/debug/regmap/meson-reset*/registers
```

الـ output بيبقى:
```
00: 00000000   # reset register عند offset 0x0
7c: ffffffff   # level register (meson8b) — كل الـ bits deasserted
```

---

#### 2. sysfs entries

```bash
# تحقق إن الـ driver اتحمّل صح
ls /sys/bus/platform/drivers/meson_reset/
# هتلاقي symlink للـ device زي:
# ff800000.reset-controller -> ../../../../devices/platform/ff800000.reset-controller

# تحقق من الـ of_node
cat /sys/bus/platform/devices/*/of_node/compatible

# عدد الـ resets المتاحة
cat /sys/class/reset-controller/*/nr_resets
# مثال output: 256
```

---

#### 3. ftrace — tracepoints وأهم events

```bash
# فعّل الـ tracing على regmap operations
cd /sys/kernel/debug/tracing

echo 1 > tracing_on
echo "regmap_reg_write regmap_reg_read" > set_event

# أو فعّل كل الـ regmap events
echo "regmap:*" > set_event
echo 1 > tracing_on

# شغّل الـ reset (مثلاً reset line 5)
# ثم اقرأ الـ trace
cat trace
```

مثال output:
```
          bash-1234  [000]  ...1  regmap_reg_write: meson-reset 0x00000000 = 0x00000020
          bash-1234  [000]  ...1  regmap_reg_write: meson-reset 0x0000007c = 0x00000020
```

**الـ `0x20` = `BIT(5)` يعني reset line 5 اتعمله assert.**

```bash
# تتبّع دوال الـ driver نفسه
echo "meson_reset_*" > set_ftrace_filter
echo function > current_tracer
cat trace
```

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug للـ driver
echo "module meson_reset +p" > /sys/kernel/debug/dynamic_debug/control
echo "module reset_controller +p" > /sys/kernel/debug/dynamic_debug/control

# أو بالـ file path
echo "file drivers/reset/amlogic/reset-meson-common.c +p" \
    > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug للـ regmap
echo "module regmap +p" > /sys/kernel/debug/dynamic_debug/control
```

الـ output في `dmesg`:
```
[  12.345678] meson_reset meson-reset: registering reset controller with 256 resets
```

---

#### 5. Kernel Config options للـ debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_RESET_CONTROLLER=y` | لازم يكون مفعّل أصلاً |
| `CONFIG_DEBUG_FS=y` | يفتح الـ debugfs entries |
| `CONFIG_REGMAP_DEBUGFS=y` | يظهر registers في debugfs |
| `CONFIG_DYNAMIC_DEBUG=y` | يفعّل `pr_debug()` runtime |
| `CONFIG_KALLSYMS=y` | يظهر رموز الدوال في stack traces |
| `CONFIG_DEBUG_KERNEL=y` | umbrella option للـ debug |
| `CONFIG_PROVE_LOCKING=y` | يكشف مشاكل الـ locking |
| `CONFIG_OF_UNITTEST=y` | يفعّل اختبارات الـ Device Tree |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "RESET_CONTROLLER|REGMAP_DEBUGFS|DYNAMIC_DEBUG"
```

---

#### 6. devlink وأدوات subsystem-specific

الـ driver ده مش بيستخدم devlink مباشرة، لكن في أدوات مفيدة:

```bash
# استخدم reset-id من DT لاختبار reset يدوي
# (لو عندك access من driver آخر)
cat /proc/device-tree/*/resets

# اعرض كل الـ reset consumers
grep -r "resets" /proc/device-tree/ 2>/dev/null

# أو استخدم dtc لقراءة الـ DT المُحمَّل
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "reset-controller"

# تحقق من الـ module مُحمَّل
lsmod | grep meson_reset
modinfo meson_reset
```

---

#### 7. رسائل الخطأ الشائعة

| رسالة في dmesg | المعنى | الحل |
|----------------|--------|-------|
| `can't init regmap mmio region` | فشل `devm_regmap_init_mmio` — عادةً الـ base address مش متاح | تحقق من الـ DT resource و`reg` property |
| `meson_reset: probe failed: -ENOMEM` | فشل `devm_kzalloc` — ذاكرة ناقصة | نادر، تحقق من `dmesg | grep oom` |
| `meson_reset: probe failed: -ENODEV` | `device_get_match_data` رجع NULL — الـ compatible string غلط | تحقق من الـ DT compatible مقابل `meson_reset_dt_ids` |
| `meson_reset: probe failed: -ENOENT` | مشكلة في resource — offset خاطئ في DT | تحقق من `reg` property في DT |
| `regmap: error -EIO writing to 0x00` | فشل كتابة الـ register — مشكلة hardware | تحقق من الـ clock للـ reset block |
| `reset controller not found` (من consumer) | الـ reset controller لم يُسجَّل بعد | تأكد من الـ probe order أو استخدم `deferred_probe` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

النقاط الأكثر فائدة في الـ code:

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                             unsigned long id)
{
    struct meson_reset *data =
        container_of(rcdev, struct meson_reset, rcdev);
    unsigned int offset, bit;

    /* نقطة 1: تحقق إن الـ id في الـ range الصح */
    WARN_ON(id >= data->rcdev.nr_resets);

    meson_reset_offset_and_bit(data, id, &offset, &bit);
    offset += data->param->reset_offset;

    /* نقطة 2: تحقق من نتيجة الـ write */
    int ret = regmap_write(data->map, offset, BIT(bit));
    WARN_ON(ret);
    return ret;
}

static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
{
    struct meson_reset *data =
        container_of(rcdev, struct meson_reset, rcdev);
    unsigned int offset, bit;

    /* نقطة 3: تحقق إن level_offset متاح */
    WARN_ON(!data->param->level_offset && !data->param->reset_offset);

    meson_reset_offset_and_bit(data, id, &offset, &bit);
    offset += data->param->level_offset;
    assert ^= data->param->level_low_reset;

    return regmap_update_bits(data->map, offset,
                              BIT(bit), assert ? BIT(bit) : 0);
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

```bash
# اقرأ الـ level register مباشرة وقارنه بالـ kernel state

# مثال meson8b: level_offset = 0x7c، base address من DT
RESET_BASE=$(cat /proc/device-tree/reset-controller@*/reg | \
    od -An -tx4 | awk '{print $1}')
echo "Base: 0x$RESET_BASE"

# اقرأ الـ reset register (offset 0x0)
devmem2 0x$((16#$RESET_BASE + 0x00)) w

# اقرأ الـ level register (offset 0x7c لـ meson8b)
devmem2 0x$((16#$RESET_BASE + 0x7c)) w
```

ثم قارن بالـ debugfs:
```bash
cat /sys/kernel/debug/regmap/meson-reset*/registers
```

---

#### 2. Register Dump باستخدام devmem2 و /dev/mem

```bash
# تثبيت devmem2 لو مش موجود
apt-get install devmem2
# أو compile من المصدر

# مثال: dump كل الـ reset registers على GXBB (base = 0xc8100000)
BASE=0xc8100000

# reset registers: offset 0x0 إلى 0x1c (8 registers × 4 bytes = 32 bytes)
for offset in 00 04 08 0c 10 14 18 1c; do
    ADDR=$((BASE + 16#$offset))
    printf "REG[0x%02x] = " $((16#$offset))
    devmem2 $(printf "0x%x" $ADDR) w 2>/dev/null | grep "Value at"
done

# level registers: offset 0x7c إلى 0x98 (meson8b)
for offset in 7c 80 84 88 8c 90 94 98; do
    ADDR=$((BASE + 16#$offset))
    printf "LEV[0x%02x] = " $((16#$offset))
    devmem2 $(printf "0x%x" $ADDR) w 2>/dev/null | grep "Value at"
done
```

مثال output مفسَّر:
```
REG[0x00] = 0x00000000  # reset register — يُكتب فيه ثم يُرجع لصفر تلقائياً
LEV[0x7c] = 0xffffffff  # كل الـ bits = 1 → كل الـ lines deasserted (level_low_reset=true)
LEV[0x80] = 0xfffffffe  # bit 0 = 0 → reset line 32 لا تزال في reset
```

```bash
# بديل باستخدام /dev/mem مع io tool
io -4 -r 0xc8100000  # اقرأ 4 bytes
```

---

#### 3. Logic Analyzer / Oscilloscope

الـ reset lines على Amlogic Meson عادةً internal للـ SoC وما بتوصلش لـ pins خارجية. لكن لو في reset line بتاخد peripheral خارجي:

- **الـ logic analyzer**: راقب الـ reset pin على الـ peripheral، المدة بين assert وdeassert لازم تكون كافية (راجع datasheet الـ peripheral).
- **الـ oscilloscope**: قيس الـ voltage levels:
  - Active-low reset: الـ line لازم توصل `0V` وقت الـ assert.
  - Active-high reset: الـ line لازم توصل `VCC` وقت الـ assert.
- **نقطة المراقبة**: بين الـ SoC reset pin والـ peripheral reset pin.
- **timing**: راقب إن الـ deassert بييجي بعد الـ clock stable للـ peripheral.

```
SoC Reset Pin ──────┐
                    │  assert (low)
    ________________│‾‾‾‾‾‾‾‾‾‾‾‾
                    │← min pulse width (راجع datasheet)
```

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في dmesg | السبب |
|---------|---------------|-------|
| الـ clock للـ reset block مش مفعّل | `regmap: error -EIO writing` | الـ reset controller نفسه يحتاج clock يكون مفعّل أولاً |
| Base address غلط في DT | `can't init regmap mmio region` + `-ENXIO` | الـ `reg` property في DT مش بيطابق الـ hardware |
| Consumer driver بيطلب reset قبل الـ probe | `(____ptrval____): reset controller not ready` ثم `-EPROBE_DEFER` | الـ probe order — الـ reset controller لازم يتسجّل أول |
| Reset line ID تعدّى الـ nr_resets | `reset: invalid reset index` | الـ consumer بيطلب reset ID أكبر من `reset_num` |
| Level register لا يستجيب | الـ bit بيفضل deasserted بعد `assert()` | `level_low_reset` قيمته غلط في الـ param |

---

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT node موجود ومُحمَّل
ls /proc/device-tree/ | grep reset

# اعرض الـ compatible string
cat /proc/device-tree/reset-controller*/compatible
# المتوقع: amlogic,meson-gxbb-reset أو amlogic,meson8b-reset إلخ

# تحقق من الـ reg property (base address وsize)
hexdump -C /proc/device-tree/reset-controller*/reg
# مثال output:
# 00000000  c8 10 00 00 00 00 01 00  |........|
# يعني base=0xc8100000، size=0x100

# تحقق من الـ #reset-cells (لازم = 1)
hexdump -C /proc/device-tree/reset-controller*/'#reset-cells'
# المتوقع: 00 00 00 01

# تحقق من الـ consumers
grep -r "resets = " /proc/device-tree/ 2>/dev/null | head -20

# قارن الـ DT المُحمَّل بالـ source
dtc -I fs /proc/device-tree 2>/dev/null | grep -B2 -A10 "reset-controller"
```

مثال DT صحيح:
```dts
reset: reset-controller@c8100000 {
    compatible = "amlogic,meson-gxbb-reset";
    reg = <0x0 0xc8100000 0x0 0x100>;
    #reset-cells = <1>;
};
```

تحقق إن الـ compatible يطابق أحد القيم في `meson_reset_dt_ids`:
```
amlogic,meson8b-reset
amlogic,meson-gxbb-reset
amlogic,meson-axg-reset
amlogic,meson-a1-reset
amlogic,meson-s4-reset
amlogic,c3-reset
amlogic,t7-reset
```

---

### Practical Commands

#### Session كامل للـ debugging من الصفر

```bash
# ═══════════════════════════════════════
# 1. تحقق إن الـ module محمّل
# ═══════════════════════════════════════
lsmod | grep meson_reset

# لو مش محمّل:
modprobe meson_reset

# ═══════════════════════════════════════
# 2. تحقق من الـ probe
# ═══════════════════════════════════════
dmesg | grep -i "meson.reset\|reset.*meson" | tail -20

# ═══════════════════════════════════════
# 3. اعرض الـ regmap registers
# ═══════════════════════════════════════
# ابحث عن الـ regmap entry
ls /sys/kernel/debug/regmap/
# ثم
cat /sys/kernel/debug/regmap/<device-name>/registers

# ═══════════════════════════════════════
# 4. فعّل dynamic debug وراقب الـ reset operations
# ═══════════════════════════════════════
echo "file drivers/reset/amlogic/reset-meson-common.c +pflmt" \
    > /sys/kernel/debug/dynamic_debug/control

# ═══════════════════════════════════════
# 5. فعّل regmap tracing
# ═══════════════════════════════════════
echo "regmap:regmap_reg_write regmap:regmap_reg_read" \
    > /sys/kernel/debug/tracing/set_event
echo 1 > /sys/kernel/debug/tracing/tracing_on

# ═══════════════════════════════════════
# 6. اعمل reset يدوي على line معيّنة (عن طريق consumer)
# ═══════════════════════════════════════
# مثلاً: أعد probe لـ device معيّن
echo "ff800000.usb" > /sys/bus/platform/drivers/dwc3/unbind
echo "ff800000.usb" > /sys/bus/platform/drivers/dwc3/bind

# ═══════════════════════════════════════
# 7. اقرأ الـ trace
# ═══════════════════════════════════════
cat /sys/kernel/debug/tracing/trace | grep regmap
```

#### تفسير الـ trace output

```
# السطر ده يعني:
dwc3-1234  [000]  regmap_reg_write: meson-reset 0x00000000 = 0x00000040
#                                   ^controller  ^reset-reg  ^BIT(6) → reset line 6

dwc3-1234  [000]  regmap_reg_write: meson-reset 0x0000007c = 0x00000040
#                                   ^controller  ^level-reg  ^assert line 6

dwc3-1234  [000]  regmap_reg_write: meson-reset 0x0000007c = 0x00000000
#                                   ^controller  ^level-reg  ^deassert line 6 (level_low_reset)
```

#### script لفحص حالة reset line معيّنة

```bash
#!/bin/bash
# check_reset.sh <reset_id> <base_address> <level_offset>
# مثال: ./check_reset.sh 6 0xc8100000 0x7c

ID=$1
BASE=$2
LEV_OFF=$3

LEVEL_ADDR=$((BASE + LEV_OFF))
REG_IDX=$((ID / 32))
BIT_IDX=$((ID % 32))
REG_ADDR=$((LEVEL_ADDR + REG_IDX * 4))

VAL=$(devmem2 $(printf "0x%x" $REG_ADDR) w 2>/dev/null | \
      awk '/Value at/ {print $NF}')
BIT_VAL=$(( (16#${VAL#0x} >> BIT_IDX) & 1 ))

echo "Reset line $ID:"
echo "  Register addr: $(printf '0x%x' $REG_ADDR)"
echo "  Register value: $VAL"
echo "  Bit $BIT_IDX = $BIT_VAL"
echo "  Status (level_low_reset=true): $([ $BIT_VAL -eq 0 ] && echo 'ASSERTED' || echo 'DEASSERTED')"
```

مثال output:
```
Reset line 6:
  Register addr: 0xc810007c
  Register value: 0xffffffbf
  Bit 6 = 0
  Status (level_low_reset=true): ASSERTED
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box على Amlogic S905X — HDMI مش شغال بعد resume من suspend

#### العنوان
**HDMI output** مش بترجع بعد `suspend/resume` على TV box بيستخدم Amlogic S905X

#### السياق
شركة بتعمل **Android TV box** بشرائح Amlogic S905X. المنتج وصل للـ production وبدأت شكاوى من users إن الـ HDMI بيتقطع بعد ما الجهاز يدخل في `suspend` وييجي `resume`. الشاشة بتفضل سودة ومحتاج reboot كامل.

#### المشكلة
الـ `HDMI controller` جوه الـ S905X محتاج reset صريح أثناء الـ `resume` عشان يرجع يشتغل صح. الـ driver بيعمل `deassert` للـ reset بس من غير ما يعمل `assert` الأول — يعني مش بيعمل full toggle.

#### التحليل
الـ driver بيستخدم `reset_control_deassert()` بس أثناء `resume`:

```c
/* في hdmi driver أثناء resume */
reset_control_deassert(hdmi_reset); /* بس من غير assert قبلها */
```

لما بنبص على `reset-meson-common.c` بنلاقي إن `meson_reset_deassert` بتعمل:

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
{
    /* ... */
    offset += data->param->level_offset; /* بتروح لـ level register */
    assert ^= data->param->level_low_reset; /* بتعكس لو reset active-low */

    return regmap_update_bits(data->map, offset,
                              BIT(bit), assert ? BIT(bit) : 0);
    /* لو assert=false → بتكتب 0 في الـ bit → deassert فقط */
}
```

المشكلة إن الـ `level register` ممكن يكون فعلاً `deasserted` من قبل الـ suspend، فالكتابة مش بتحدث أي تغيير فعلي في الـ hardware — الـ `HDMI PHY` فضل stuck.

لو استخدمنا `meson_reset_toggle_ops` بدل `meson_reset_ops`، الـ `.reset` callback هيكون:

```c
static int meson_reset_level_toggle(struct reset_controller_dev *rcdev,
                                    unsigned long id)
{
    int ret;

    ret = meson_reset_assert(rcdev, id);   /* assert أولاً */
    if (ret)
        return ret;

    return meson_reset_deassert(rcdev, id); /* deassert تاني */
}
```

ده بيضمن إن الـ hardware يشوف `assert → deassert` وحدة واضحة.

#### الحل
تغيير الـ HDMI driver عشان يستخدم `reset_control_reset()` بدل `reset_control_deassert()` وقت الـ resume، أو التأكد إن الـ reset controller المسجّل بيستخدم `meson_reset_toggle_ops`:

```c
/* في DT: reset-controller node */
&reset_controller {
    /* الـ driver المسؤول يختار meson_reset_toggle_ops */
};

/* في hdmi resume code */
ret = reset_control_reset(hdmi_reset); /* assert ثم deassert تلقائياً */
if (ret)
    dev_err(dev, "HDMI reset failed: %d\n", ret);
```

للتحقق:

```bash
# شوف الـ reset status قبل وبعد resume
cat /sys/kernel/debug/reset/*/status
# أو
devmem2 0xc1104404 w  # قراءة reset level register على S905X
```

#### الدرس المستفاد
`meson_reset_ops` و`meson_reset_toggle_ops` مش نفس الشيء. لو الـ peripheral محتاج full `assert→deassert` cycle عشان يتـ initialize صح، لازم تستخدم `reset_control_reset()` مع `meson_reset_toggle_ops` مش بس `deassert`.

---

### السيناريو 2: Industrial Gateway على Amlogic A311D — Ethernet مش بيشتغل بعد driver load

#### العنوان
**Ethernet** على A311D بيفشل بعد `insmod` الـ driver بسبب reset offset غلط في الـ platform data

#### السياق
فريق هندسي بيعمل **industrial IoT gateway** على Amlogic A311D. الـ board بيحتاج Gigabit Ethernet. بعد ما عملوا bring-up للـ kernel، الـ `eth0` بيظهر في `ip link` بس مش بيقدر يعمل `link up` خالص حتى لو الكابل موصول.

#### المشكلة
الـ `reset_offset` في الـ `meson_reset_param` للـ Ethernet reset controller غلط — بيشاور على register لا علاقتله بالـ Ethernet بالمرة.

#### التحليل
في `reset-meson-common.c` عند حساب الـ offset:

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
{
    unsigned int stride = regmap_get_reg_stride(data->map);

    *offset = (id / (stride * BITS_PER_BYTE)) * stride;
    *bit    = id % (stride * BITS_PER_BYTE);
}
```

ثم في `meson_reset_reset`:

```c
offset += data->param->reset_offset; /* ← هنا المشكلة */
return regmap_write(data->map, offset, BIT(bit));
```

لو `reset_offset` غلط في الـ `meson_reset_param` بتاع الـ platform driver، الـ `regmap_write` هتحط BIT في register تاني خالص. على A311D، الـ Ethernet reset bit موجود في `RESET1_REGISTER` مش `RESET0_REGISTER`.

نفترض إن الـ param بتقول:

```c
/* غلط */
static const struct meson_reset_param a311d_param = {
    .reset_ops    = &meson_reset_ops,
    .reset_num    = 256,
    .reset_offset = 0x00,  /* ← بيشاور على RESET0 */
    .level_offset = 0x40,
    .level_low_reset = false,
};
```

لكن Ethernet reset id = 40 مثلاً، يعني:
- `stride` = 4 bytes (32-bit registers)
- `offset` = (40 / 32) * 4 = 4 bytes → RESET1
- لكن `reset_offset = 0x00` → النتيجة النهائية 0x04 ✓

لو الـ offset في الـ param بيشاور على base address غلط:

```c
.reset_offset = 0x10,  /* غلط: بيحط الـ write في 0x14 بدل 0x04 */
```

يبقى الـ Ethernet hardware مش بياخد reset خالص.

#### الحل
```bash
# تحقق من الـ reset registers على A311D بـ devmem2
devmem2 0xFF800000 w  # RESET0
devmem2 0xFF800004 w  # RESET1
devmem2 0xFF800008 w  # RESET2

# قارن بـ Amlogic A311D TRM - Ethernet reset في RESET2[15]
# يعني reset_offset المفروض يبقى 0x08
```

تعديل الـ platform data:

```c
/* صح */
static const struct meson_reset_param a311d_eth_param = {
    .reset_ops    = &meson_reset_ops,
    .reset_num    = 256,
    .reset_offset = 0x00,  /* base: kernel بيحسب offset من id */
    .level_offset = 0x40,
    .level_low_reset = false,
};
```

ثم اعمل:

```bash
# تحقق بعد التصحيح
echo 1 > /sys/class/net/eth0/device/reset
ip link show eth0
```

#### الدرس المستفاد
الـ `reset_offset` في `meson_reset_param` هو الـ **base address** للـ reset registers داخل الـ regmap — الـ kernel بيحسب الـ per-bit offset تلقائياً من الـ `id`. لازم تتحقق من TRM إن الـ base صح، وإلا الـ writes هتروح في أماكن عشوائية.

---

### السيناريو 3: Custom Board Bring-up على Amlogic S905X3 — USB مش بيشتغل خالص

#### العنوان
**USB 3.0** مش بيشتغل على custom board بسبب `level_low_reset` غلط في الـ param

#### السياق
فريق hardware بيعمل **custom media player board** على Amlogic S905X3. الـ USB 3.0 controller مبيشوفش أي devices. الـ kernel log بيقول `xhci-hcd` اتـ probe صح، بس `lsusb` فاضي تماماً.

#### المشكلة
الـ USB reset line على هذا الـ SoC **active-low** — يعني عشان ترفع الـ reset (deassert)، محتاج تكتب `0` في الـ bit. الـ `level_low_reset = false` في الـ param معناه الـ driver هيكتب `1` عشان يعمل deassert — وده في الحقيقة بيخلي الـ USB فاضل في reset دايماً.

#### التحليل
في `meson_reset_level`:

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
{
    /* ... */
    assert ^= data->param->level_low_reset;
    /*
     * لو level_low_reset = false:
     *   deassert (assert=false): false ^ false = false → كتابة 0 → normal
     *
     * لو level_low_reset = true (active-low):
     *   deassert (assert=false): false ^ true = true → كتابة 1
     *   assert   (assert=true):  true  ^ true = false → كتابة 0
     *
     * الـ XOR بيعكس المعنى لو الـ hardware active-low
     */

    return regmap_update_bits(data->map, offset,
                              BIT(bit), assert ? BIT(bit) : 0);
}
```

وكمان `meson_reset_status` بيستخدم نفس المنطق:

```c
val = !!(BIT(bit) & val);
return val ^ data->param->level_low_reset;
/* لو level_low_reset=true وقرأنا 1 → 1^1=0 → يعني deasserted (شغال) */
```

لو الـ param غلط:

```c
/* غلط: USB على S905X3 active-low */
static const struct meson_reset_param s905x3_usb_param = {
    .level_low_reset = false, /* ← المشكلة */
};
```

الـ `meson_reset_deassert` هيكتب `0` في الـ bit بدل `1`، وده يخلي الـ USB controller فاضل في assert (reset active).

#### الحل
```c
/* صح */
static const struct meson_reset_param s905x3_usb_param = {
    .reset_ops       = &meson_reset_ops,
    .reset_num       = 256,
    .reset_offset    = 0x00,
    .level_offset    = 0x40,
    .level_low_reset = true,  /* ← USB reset active-low على S905X3 */
};
```

للتحقق قبل وبعد:

```bash
# قرأ الـ USB reset level register
devmem2 0xFF800040 w

# لو bit X = 0 بعد deassert → صح (active-low deasserted بـ 0)
# انظر لـ TRM الـ USB reset bit

# تحقق من status
cat /sys/kernel/debug/reset/*/status
```

#### الدرس المستفاد
الـ `level_low_reset` flag مش تفصيلة صغيرة — هي بتعكس **كل منطق الـ assert/deassert**. لازم تراجع TRM للـ SoC عشان تعرف كل peripheral reset polarity صح قبل ما تكتب الـ platform data.

---

### السيناريو 4: IoT Sensor Hub على Amlogic S905W — Kernel Panic أثناء الـ boot بسبب regmap stride غلط

#### العنوان
**Kernel panic** أثناء أول reset operation بسبب حساب offset غلط نتيجة stride خاطئ

#### السياق
فريق embedded بيعمل **IoT sensor hub** رخيص على Amlogic S905W (نسخة مقتصدة من S905X). بعد ما عملوا port للـ kernel، الـ system بيعمل panic بعد ثواني من الـ boot أثناء ما peripheral معين بيحاول يعمل reset.

#### المشكلة
الـ regmap اتـ configure بـ `reg_stride = 1` (byte access) بدل `reg_stride = 4` (32-bit word access) زي ما المفروض لـ Amlogic reset registers. ده بيخلي حساب الـ offset غلط جداً.

#### التحليل
في `meson_reset_offset_and_bit`:

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
{
    unsigned int stride = regmap_get_reg_stride(data->map);
    /*
     * لو stride = 4 (صح):
     *   id=0..31  → offset=0,  bit=0..31
     *   id=32..63 → offset=4,  bit=0..31
     *
     * لو stride = 1 (غلط):
     *   id=0..7   → offset=0,  bit=0..7
     *   id=8..15  → offset=1,  bit=0..7
     *   id=32     → offset=4,  bit=0
     *   ← الـ offset بيتحسب بالبايت مش بالـ word!
     */

    *offset = (id / (stride * BITS_PER_BYTE)) * stride;
    *bit    = id % (stride * BITS_PER_BYTE);
}
```

لو `stride=1`:
- `id=64` → `offset = (64/8)*1 = 8` bytes من الـ base
- `bit = 64%8 = 0`

لو `stride=4` (صح):
- `id=64` → `offset = (64/32)*4 = 8` bytes من الـ base
- `bit = 64%32 = 0`

في الحالة دي النتيجة اتفقت مصادفة، بس:
- `id=10` مع `stride=1`: `offset=1`, `bit=2` — بيكتب في byte register
- `id=10` مع `stride=4`: `offset=0`, `bit=10` — بيكتب في word register

الـ regmap_write على address تحت المسموح بيه ممكن يطلع `BUS_ERROR` وده يعمل panic.

#### الحل
```c
/* في platform driver بتاع S905W */
static const struct regmap_config s905w_reset_regmap_config = {
    .reg_bits   = 32,
    .val_bits   = 32,
    .reg_stride = 4,  /* ← لازم 4 bytes لـ 32-bit registers */
};

/* وقت إنشاء الـ regmap */
map = devm_regmap_init_mmio(dev, base, &s905w_reset_regmap_config);
```

للـ debug:

```bash
# تحقق من الـ stride الفعلي
cat /sys/kernel/debug/regmap/*/range
# أو
dmesg | grep -i "regmap\|reset" | head -40
```

#### الدرس المستفاد
`regmap_get_reg_stride()` بيأثر مباشرة على كل حساب في `meson_reset_offset_and_bit`. الـ stride لازم يتطابق مع الـ hardware register width. دايماً verify الـ regmap config من TRM قبل الـ bring-up.

---

### السيناريو 5: Automotive ECU على Amlogic A311D — SPI Flash مش بيتقرأ بسبب reset مش اتحرّر

#### العنوان
**SPI NOR Flash** على automotive ECU بيفشل في الـ probe لأن reset ما اتحررش بعد power-on

#### السياق
شركة automotive بتعمل **ECU (Electronic Control Unit)** للسيارات بيستخدم Amlogic A311D. الـ ECU بيستخدم SPI NOR flash لتخزين firmware. بعد power-on، الـ kernel بيفشل في الـ probe للـ SPI flash بـ `spi-nor: probe timeout`. الـ scope بيُريّ إن الـ SPI CLK مش بيشتغل.

#### المشكلة
الـ SPI controller محتاج `deassert` للـ reset قبل ما يشتغل. الـ driver بيستخدم `devm_reset_control_get_optional()` وبيفضل يشتغل حتى لو الـ reset handle فاضي `NULL` — ومن غير deassert، الـ SPI controller فاضل في reset.

#### التحليل
الـ reset controller اتسجّل صح عبر `meson_reset_controller_register`:

```c
int meson_reset_controller_register(struct device *dev, struct regmap *map,
                                    const struct meson_reset_param *param)
{
    struct meson_reset *data;

    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->param = param;
    data->map   = map;
    data->rcdev.owner    = dev->driver->owner;
    data->rcdev.nr_resets = param->reset_num;
    data->rcdev.ops      = data->param->reset_ops;
    data->rcdev.of_node  = dev->of_node; /* ← ربط بالـ DT node */

    return devm_reset_controller_register(dev, &data->rcdev);
}
```

لو `dev->of_node` NULL أو الـ DT للـ SPI مش بيشاور على الـ reset controller صح:

```dts
/* غلط - مفيش resets property */
&spi0 {
    status = "okay";
    /* مفيش resets = <&reset_controller RESET_SPI0>; */
};
```

يبقى `devm_reset_control_get_optional()` بيرجع NULL والـ driver مش بيعمل deassert. الـ SPI controller فاضل في reset من الـ power-on default state.

عشان نتأكد: `meson_reset_status` بيقرأ الـ level register:

```c
static int meson_reset_status(struct reset_controller_dev *rcdev,
                              unsigned long id)
{
    /* ... */
    regmap_read(data->map, offset, &val);
    val = !!(BIT(bit) & val);
    return val ^ data->param->level_low_reset;
    /* لو بيرجع 1 → الـ peripheral لسه في reset */
}
```

#### الحل
تصحيح الـ DT:

```dts
/* صح */
&spi0 {
    status = "okay";
    resets = <&reset_controller RESET_SPI0>;
    reset-names = "spi";
};
```

وفي الـ SPI driver:

```c
/* تأكيد إن الـ deassert بيتعمل */
rst = devm_reset_control_get_optional_exclusive(dev, "spi");
if (IS_ERR(rst))
    return PTR_ERR(rst);

if (rst) {
    ret = reset_control_deassert(rst);
    if (ret) {
        dev_err(dev, "failed to deassert reset: %d\n", ret);
        return ret;
    }
}
```

للتحقق:

```bash
# تحقق من reset status قبل وبعد الـ deassert
cat /sys/kernel/debug/reset/*/status

# تحقق من DT
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "spi0"

# لو SPI controller في reset:
# الـ CLK مش بيشتغل → مش هتشوف نبضات على الـ scope
```

#### الدرس المستفاد
`devm_reset_controller_register` بيربط الـ reset controller بالـ `of_node` — لو الـ DT مش فيه `resets` property على الـ peripheral node، الـ consumer driver مش هيلاقي الـ reset خالص. الـ DT و الـ driver لازم يكونوا متزامنين. دايماً verify الـ reset status في الـ `debugfs` أثناء الـ bring-up.
## Phase 7: مصادر ومراجع

---

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط |
|--------|--------|
| **Reset Controller API** — الوثيقة الرئيسية | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) |
| **Reset Controller API** (kernel.org RST source) | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) |
| **Device Tree Binding** لـ Amlogic Meson Reset | [amlogic,meson-reset.yaml](https://www.kernel.org/doc/Documentation/devicetree/bindings/reset/amlogic,meson-reset.yaml) |
| **Driver implementer's API guide** | [docs.kernel.org/driver-api/index.html](https://docs.kernel.org/driver-api/index.html) |

---

### مقالات LWN.net ذات الصلة

الـ **LWN.net** هو المرجع الأهم لمتابعة تطور الـ kernel — الروابط دي حقيقية ومعادة من نتائج البحث:

| المقال | الرابط |
|--------|--------|
| **reset: Add generic GPIO reset driver** — أول GPIO-based reset controller | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) |
| **regmap: Generic I2C and SPI register map library** — الـ patch الأصلي لـ regmap | [lwn.net/Articles/451789](https://lwn.net/Articles/451789/) |
| **Generic I2C and SPI register map library** — discussion thread | [lwn.net/Articles/448436](https://lwn.net/Articles/448436/) |
| **regmap: introduce fast_io busses** — spinlock optimization | [lwn.net/Articles/490704](https://lwn.net/Articles/490704/) |
| **clk: en7523: reset-controller support for EN7523 SoC** | [lwn.net/Articles/1039543](https://lwn.net/Articles/1039543/) |
| **PCI: Expose and manage PCI device reset** | [lwn.net/Articles/866578](https://lwn.net/Articles/866578/) |

---

### Patchwork — المناقشات والـ Patches الأصلية

الـ **Patchwork** هو الأرشيف الرسمي للـ patches المرسلة على الـ mailing lists:

| الـ Patch | الرابط |
|-----------|--------|
| **[v4,2/8] reset: Add reset controller API** — الـ patch الأصلي لـ Philipp Zabel | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) |
| **[v4,2/2] reset: Add GPIO support to reset controller framework** | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/) |
| **[v3,3/3] reset: add support for Meson-A1 SoC Reset Controller** | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/1569738255-3941-4-git-send-email-xingyu.chen@amlogic.com/) |
| **[v2] reset: meson: add level reset support for GX SoC family** — Neil Armstrong | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1508235558-24860-1-git-send-email-narmstrong@baylibre.com/) |

---

### Lore.kernel.org — مناقشات الـ Mailing List

الـ **lore.kernel.org** هو الأرشيف الرسمي الجديد لكل الـ mailing lists:

| الموضوع | الرابط |
|---------|--------|
| **[PATCH 0/8] reset: amlogic: move audio reset drivers out of CCF** — Jerome Brunet, 2024 | [lore.kernel.org](https://lore.kernel.org/lkml/20240710162526.2341399-1-jbrunet@baylibre.com/T/) |
| **[GIT PULL] Reset controller fixes for v6.2** — Philipp Zabel | [lore.kernel.org](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/) |
| **[GIT PULL v2] Reset controller API** — الـ pull request الأصلي | [lore.kernel.org](https://lore.kernel.org/linux-arm-kernel/1365683829.4388.52.camel@pizza.hi.pengutronix.de/) |
| **الـ LKML الرئيسي** — أرشيف شامل | [lkml.org](https://lkml.org/) |

---

### Git — الـ Commits المهمة في تاريخ الـ Driver

للبحث عن commits خاصة بالـ Meson reset driver في مستودع الـ kernel الرسمي:

```bash
# تتبع تاريخ الملف الأصلي
git log --oneline drivers/reset/amlogic/reset-meson-common.c
git log --oneline drivers/reset/reset-meson.c

# البحث في المستودع الرسمي online
# https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/reset/amlogic
```

الـ **kernel.org git browser** المباشر:
- [git.kernel.org — drivers/reset/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/reset)
- [git.kernel.org — drivers/reset/amlogic/](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/reset/amlogic)

---

### مصادر تقنية إضافية

#### Regmap Framework
الـ **regmap** هو الطبقة التجريدية اللي بيستخدمها driver الـ Meson Reset للوصول للـ registers:

| المصدر | الرابط |
|--------|--------|
| **Using regmaps to make Linux drivers more generic** — Collabora blog | [collabora.com](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) |
| **MFD, regmap, syscon** — Embedded Linux Conference slides | [linuxfoundation.org (PDF)](http://events17.linuxfoundation.org/sites/events/files/slides/belloni-mfd-regmap-syscon_0.pdf) |

#### BayLibre و Amlogic Mainlining
| المصدر | الرابط |
|--------|--------|
| **How We Improved AmLogic Support in Mainline Linux** — BayLibre blog | [baylibre.com](https://baylibre.com/improved-amlogic-support-mainline-linux/) |
| **Amlogic Mainline Linux** — Embedded Recipes 2017 (Neil Armstrong) | [freedesktop.org](https://people.freedesktop.org/~narmstrong/erecipes-2017-amlogic/) |
| **Kernel mainlining progress** — linux-meson.com | [linux-meson.com](https://linux-meson.com/mainlining.html) |

---

### الكتب المرجعية الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
الكتاب المجاني المتاح online — الأكثر استخداماً لتعلم كتابة الـ drivers:

- **الفصل 14** — The Linux Device Model: فاهم `device`, `driver`, `bus`, و `class`
- **الفصل 3** — Char Drivers: الأساس قبل ما تتعلم أي subsystem
- التحميل المجاني: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17** — Devices and Modules: شرح `module_init`, `EXPORT_SYMBOL`, وربط الـ drivers بالـ kernel
- **الفصل 14** — The Block I/O Layer: فاهم الـ subsystem architecture pattern المتكرر في كل الـ frameworks

#### Essential Linux Device Drivers — Sreekrishnan Venkateswaran
- **الفصل 6** — Serial Drivers: نموذج لكيفية بناء الـ `ops` struct وتسجيل الـ controller
- بيفسر الـ `container_of` macro بشكل عملي ممتاز

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 15** — Kernel Initialization: بيفسر كيف الـ SoC components بتتهيأ ومتى بيحصل الـ reset
- مناسب لفهم السياق الـ hardware لأي reset controller

#### Linux Device Drivers Development — John Madieu (Packt, 2017)
- فصل خاص عن **regmap**: practical examples بـ I2C وـ SPI وـ MMIO
- فصل عن **Device Tree** binding وتسجيل الـ platform drivers
- المصدر الأحدث اللي بيغطي الـ APIs الحديثة

---

### مصادر الـ Kernel Source المباشرة

الملفات دي في الـ kernel source هي أهم مرجع:

```
# Consumer API — كيف الـ drivers بتطلب reset
include/linux/reset.h

# Provider API — كيف الـ reset controllers بتسجل نفسها
include/linux/reset-controller.h

# التوثيق الرسمي
Documentation/driver-api/reset.rst

# Driver بسيط للدراسة (GPIO-based)
drivers/reset/reset-gpio.c

# الـ Meson driver نفسه
drivers/reset/amlogic/reset-meson-common.c
drivers/reset/amlogic/reset-meson.h

# Simple reset controller — نموذج شائع في الـ embedded
drivers/reset/reset-simple.c
```

---

### مصطلحات البحث — للعثور على مزيد من المعلومات

استخدم المصطلحات دي في بحثك:

```
# للبحث في الـ kernel mailing list
site:lore.kernel.org "reset-controller"
site:lore.kernel.org "reset_controller_dev"
site:lore.kernel.org amlogic meson reset

# للبحث عن مقالات تقنية
"reset_control_ops" linux kernel driver
"devm_reset_controller_register" example
"regmap_update_bits" reset driver linux

# للبحث عن الـ Device Tree binding
"amlogic,meson-reset" devicetree
linux reset controller devicetree binding example

# للبحث عن الـ patches
patchwork "reset: meson"
lore.kernel.org "reset: amlogic"
```

---

### ملخص أهم المراجع بالترتيب

| الأولوية | المرجع | السبب |
|----------|--------|--------|
| **1** | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | التوثيق الرسمي الكامل للـ API |
| **2** | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) | أول GPIO reset driver — نموذج للتعلم |
| **3** | [lwn.net/Articles/451789](https://lwn.net/Articles/451789/) | فهم regmap اللي بيستخدمه الـ driver |
| **4** | [lore.kernel.org — reset amlogic 2024](https://lore.kernel.org/lkml/20240710162526.2341399-1-jbrunet@baylibre.com/T/) | آخر تطور في الـ Meson reset subsystem |
| **5** | [patchwork — reset API v4](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | الـ patch الأصلي للـ framework من Philipp Zabel |
| **6** | LDD3 — الفصل 14 | فهم الـ device model الأساسي |
| **7** | [collabora.com — regmap guide](https://www.collabora.com/news-and-blog/blog/2020/05/27/using-regmaps-to-make-linux-drivers-more-generic/) | شرح عملي للـ regmap بـ examples |
## Phase 8: Writing simple module

### الفكرة

**`meson_reset_controller_register`** هي الدالة المصدَّرة الأهم في الملف — بتُسجَّل كل مرة درايفر Amlogic Meson يلف نفسه كـ reset controller. هنـ kprobe على الدالة دي مش هيأثر على الـ hardware، وهيطبع لنا معلومات مفيدة عن كل reset controller بيتسجل في النظام.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on meson_reset_controller_register
 *
 * Prints device name and number of resets every time
 * an Amlogic Meson reset controller is registered.
 */

/* (1) Includes */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/device.h>
#include <linux/reset-controller.h>

/* forward declaration of the param struct used by meson */
struct meson_reset_param {
    const struct reset_control_ops *reset_ops;
    unsigned int reset_num;
    unsigned int reset_offset;
    unsigned int level_offset;
    bool level_low_reset;
};

/*
 * (2) pre-handler: called right before meson_reset_controller_register executes.
 *
 * Prototype of the hooked function:
 *   int meson_reset_controller_register(struct device *dev,
 *                                        struct regmap *map,
 *                                        const struct meson_reset_param *param);
 *
 * pt_regs on arm64: x0=dev, x1=map, x2=param
 * pt_regs on x86-64: di=dev, si=map, dx=param
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#if defined(CONFIG_X86_64)
    struct device             *dev   = (struct device *)regs->di;
    struct meson_reset_param  *param = (struct meson_reset_param *)regs->dx;
#elif defined(CONFIG_ARM64)
    struct device             *dev   = (struct device *)regs->regs[0];
    struct meson_reset_param  *param = (struct meson_reset_param *)regs->regs[2];
#else
    /* unsupported arch — just log that we hit the probe */
    pr_info("meson_reset_probe: triggered (arch unknown)\n");
    return 0;
#endif

    /* safety: both pointers must be valid kernel addresses */
    if (!dev || !param)
        return 0;

    pr_info("meson_reset_probe: registering reset controller\n");
    pr_info("  device    : %s\n", dev_name(dev));
    pr_info("  nr_resets : %u\n", param->reset_num);
    pr_info("  reset_off : 0x%x  level_off: 0x%x\n",
            param->reset_offset, param->level_offset);
    pr_info("  low_reset : %s\n",
            param->level_low_reset ? "yes (active-low)" : "no (active-high)");

    return 0; /* 0 = let the real function run normally */
}

/* (3) kprobe descriptor */
static struct kprobe kp = {
    .symbol_name = "meson_reset_controller_register",
    .pre_handler = handler_pre,
};

/* (4) module_init */
static int __init meson_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("meson_reset_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("meson_reset_probe: kprobe planted at %p\n", kp.addr);
    return 0;
}

/* (5) module_exit */
static void __exit meson_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("meson_reset_probe: kprobe removed\n");
}

module_init(meson_probe_init);
module_exit(meson_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc example");
MODULE_DESCRIPTION("kprobe on meson_reset_controller_register");
```

---

### شرح كل جزء

#### (1) الـ Includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل API الـ kprobes |
| `linux/device.h` | عشان نستخدم `dev_name()` ونقرأ اسم الـ device |
| `linux/reset-controller.h` | بيجيب `struct reset_control_ops` اللي محتاجينه في `meson_reset_param` |

الـ `meson_reset_param` مش exported في header عام، فعرّفناه يدويًا بنفس ترتيب الحقول الأصلي عشان الـ casting يشتغل صح.

#### (2) الـ pre_handler

**الـ `pt_regs`** بيحتوي على الـ registers وقت الدخول للدالة — على x86-64 الأرغومنتات في `di/si/dx`، وعلى arm64 في `regs[0]/[1]/[2]`. احنا بناخد `dev` (arg0) و`param` (arg2) ونطبع منهم اسم الـ device وعدد الـ resets وoffsets الـ registers.

الـ `return 0` مهم جدًا — لو رجعنا قيمة غير صفر الـ kprobe framework هيمنع تنفيذ الدالة الأصلية، وده مش اللي عايزينه هنا.

#### (3) الـ `struct kprobe`

بنحدد `symbol_name` بدل عنوان صريح عشان الـ kernel بيحل الاسم لعنوان وقت الـ register، ومش لازم نعرف عنوان compile-time. لو الـ symbol مش موجود (مثلًا الـ driver مش loaded)، الـ `register_kprobe` هيفشل وبنطبع الـ error.

#### (4) `module_init`

بيستدعي `register_kprobe` ويطبع الـ address اللي انزرع فيه الـ probe. لو فشل الـ registration (مثلًا الـ symbol مش exported أو CONFIG_KPROBES=n)، بنرجع الـ error مباشرة.

#### (5) `module_exit`

**لازم** نعمل `unregister_kprobe` في الـ exit عشان لو شلنا الـ module من غير ما نشيل الـ probe، أي reset controller بيتسجل بعد كده هيجمب على كود اتشال من الذاكرة ويعمل kernel panic. الـ unregister بيضمن إن الـ hook اتنزع بشكل آمن قبل ما الـ module يتشال.

---

### Makefile للتجميع

```makefile
obj-m += meson_reset_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod meson_reset_probe.ko

# مشاهدة الـ output
sudo dmesg | grep meson_reset_probe

# إزالة الـ module
sudo rmmod meson_reset_probe
```

> **ملاحظة:** الـ module ده مفيد على بورد Amlogic فعلي (مثل Odroid-C4 أو AML-S905X3) أو في QEMU مع device tree بيحتوي على reset controller من نوع Meson. على x86 عادي الـ symbol مش موجود فـ`register_kprobe` هيرجع `-ENOENT`.
