## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

**الـ Reset Controller Framework** — موجود في `drivers/reset/` وبيتبعه المُشرف `Philipp Zabel`. الملف ده جزء من الـ Amlogic Meson SoC support اللي بيتبعه `Neil Armstrong` و `Kevin Hilman` من BayLibre، وبيخص تحديداً جزء الـ audio subsystem في SoC شرائح Amlogic.

---

### المشكلة اللي بيحلها الملف — القصة الكاملة

تخيل معايا إن عندك شريحة Amlogic Meson في جهاز Android أو TV Box. الشريحة دي جوّاها حاجات كتير بتشتغل مع بعض: CPU، GPU، audio، video، وغيرهم. كل جزء من دول محتاج **reset** — يعني لو حصل مشكلة أو عند الـ boot، لازم تقدر تقول للـ audio processor مثلاً "اصحى من الأول" أو "ابقى واقف لحد ما أقولك".

الـ **Reset Controller** هو الجهة المسؤولة عن إدارة الـ reset lines دي. في Meson SoCs، في reset controller رئيسي بيتسجل عن طريق Device Tree (`amlogic,meson8b-reset` وأخواتها). ده الـ driver العادي في `reset-meson.c`.

**المشكلة الجديدة:** شرائح Amlogic زي G12A و SM1 و A1 بيستخدموا audio clock controller (`axg-audio-clkc`) — وده الـ clock driver الخاص بالصوت. المشكلة إن جوّا الـ audio clock controller في **registers للـ reset الخاصة بالـ audio** — مش في الـ reset controller الرئيسي!

يعني الـ audio components (زي DAC، ADC، I2S interfaces) محتاجة reset، لكن registers الـ reset دي موجودة جوّا memory map بتاع الـ audio clock controller، مش جوّا الـ main reset controller.

**السؤال:** إزاي الـ reset framework يوصل للـ registers دي؟

**الحل:** الـ audio clock driver بيعمل **Auxiliary Device** — يعني بيقول للـ kernel "أنا عندي sub-device اسمه reset controller، أسجّله على الـ auxiliary bus". الـ file اللي بندرسه `reset-meson-aux.c` هو الـ driver اللي بيلتقط الـ auxiliary device ده ويسجّل reset controller كامل منه.

---

### الـ Auxiliary Bus — ببساطة

الـ **Auxiliary Bus** فكرة بسيطة: لو عندك driver كبير (زي audio clock controller) وجوّاه وظيفة تانية مستقلة (زي reset controller)، بدل ما تحشر الكود كله في driver واحد، الـ parent driver بيعمل "virtual device" على الـ auxiliary bus، وبييجي driver تاني صغير يتعامل مع الوظيفة دي بشكل منفصل.

```
axg-audio-clkc (parent driver / audio clock controller)
    |
    |--- registers الـ audio clock
    |--- registers الـ audio reset  <--- هنا المشكلة
    |
    +---> ينشئ auxiliary device اسمه "axg-audio-clkc.rst-g12a"
              |
              V
         reset-meson-aux.c driver يلتقطه
              |
              V
         يسجّل reset controller كامل في الـ reset framework
```

---

### ليه الملف ده مهم؟

| السبب | الشرح |
|-------|-------|
| **الفصل بين المخاوف** | الـ audio clock driver مش شغله يكون reset controller، فالملف ده بيفصل المسؤوليات |
| **إعادة استخدام الكود** | بيستخدم نفس الـ core functions من `reset-meson-common.c` |
| **دعم أجيال متعددة** | يدعم A1، G12A، SM1 — كل واحد بـ parameters مختلفة |
| **التكامل مع الـ framework** | أي driver في الكيرنل يقدر يعمل `reset_control_get()` ويـ reset audio component بدون ما يعرف التفاصيل |

---

### بنية الملف وكيف يشتغل

```c
// كل SoC عنده parameters مختلفة للـ audio reset
static const struct meson_reset_param meson_g12a_audio_param = {
    .reset_ops   = &meson_reset_toggle_ops, // toggle: assert ثم deassert
    .reset_num   = 26,                       // 26 reset line في audio
    .level_offset = 0x24,                    // offset الـ register في الـ regmap
};
```

الـ `probe` function بسيطة جداً:
1. بتاخد الـ `regmap` من الـ parent device (الـ audio clock controller) — ده اللي بيديه وصول لنفس الـ registers
2. بتسجّل reset controller بالـ parameters الخاصة بالـ SoC

```c
static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
{
    // param: عدد الـ resets والـ offsets الخاصة بكل SoC
    const struct meson_reset_param *param = (void *)(id->driver_data);

    // اجيب الـ regmap من الـ parent (audio clock controller)
    map = dev_get_regmap(adev->dev.parent, NULL);

    // سجّل reset controller — الكود الفعلي في reset-meson-common.c
    return meson_reset_controller_register(&adev->dev, map, param);
}
```

---

### الفرق بين `toggle_ops` و `reset_ops`

الـ audio reset بيستخدم `meson_reset_toggle_ops` مش `meson_reset_ops`:

| النوع | الآلية | متى يُستخدم |
|-------|--------|-------------|
| `meson_reset_ops` | يكتب bit واحد في register الـ "pulse reset" | الـ main SoC reset controller |
| `meson_reset_toggle_ops` | يعمل assert ثم deassert للـ level register | الـ audio reset (لأن مفيش pulse register) |

---

### الـ Device IDs والـ SoCs المدعومة

```c
static const struct auxiliary_device_id meson_reset_aux_ids[] = {
    { .name = "a1-audio-clkc.rst-a1",     ... }, // Amlogic A1
    { .name = "a1-audio-clkc.rst-a1-vad", ... }, // A1 Voice Activity Detection
    { .name = "axg-audio-clkc.rst-g12a",  ... }, // G12A (S905X2, S905Y2, S905D2)
    { .name = "axg-audio-clkc.rst-sm1",   ... }, // SM1 (S905X3)
};
```

**الـ name format:** `<parent_driver_name>.<auxiliary_device_name>` — ده هو الـ matching mechanism على الـ auxiliary bus.

---

### الصورة الكاملة للـ Subsystem

```
drivers/reset/amlogic/
├── reset-meson.c          ← الـ main platform driver (Device Tree based)
├── reset-meson-aux.c      ← الملف الحالي (Auxiliary Bus based, للـ audio)
├── reset-meson-common.c   ← الـ core logic المشتركة (assert/deassert/toggle)
├── reset-meson-audio-arb.c← reset controller خاص بالـ audio memory arbiter
└── reset-meson.h          ← الـ shared structs والـ API

include/linux/
├── reset-controller.h     ← تعريف struct reset_controller_dev و reset_control_ops
├── reset.h                ← الـ consumer API (reset_control_get, reset_control_reset)
└── auxiliary_bus.h        ← تعريف auxiliary_device و auxiliary_driver

drivers/clk/meson/
└── axg-audio.c            ← الـ parent driver اللي بينشئ الـ auxiliary device
```

---

### ملفات مهمة يجب معرفتها

| الملف | الدور |
|-------|-------|
| `drivers/reset/amlogic/reset-meson-common.c` | الـ core: فيه منطق الـ assert/deassert/toggle الفعلي |
| `drivers/reset/amlogic/reset-meson.h` | الـ `meson_reset_param` struct والـ exported symbols |
| `drivers/reset/amlogic/reset-meson.c` | الـ platform driver للـ main reset controller |
| `drivers/clk/meson/axg-audio.c` | الـ parent اللي بيخلق الـ auxiliary device |
| `include/linux/auxiliary_bus.h` | الـ auxiliary bus infrastructure |
| `include/linux/reset-controller.h` | الـ reset controller framework API |
| `Documentation/driver-api/reset.rst` | التوثيق الرسمي للـ reset framework |
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة اللي بيحلها الـ Subsystem ده

في أي SoC حديث زي Amlogic Meson، كل peripheral (audio engine، USB controller، GPU، إلخ) محتاج يتعمله **hardware reset** — يعني إشارة كهربية بتضغط على الـ peripheral وتخليه يرجع لحالته الابتدائية. المشكلة إن:

- الـ reset lines بتبقى مبعثرة في registers كتير في أماكن مختلفة في الـ SoC.
- كل driver كان بيعمل bit manipulation مباشرة على الـ registers، من غير أي تنسيق.
- لو جهازين حاولوا يعملوا reset لنفس الـ block في نفس الوقت → race condition.
- الـ Device Tree مش عارف يعبّر عن "مين بيتحكم في reset مين" بشكل موحد.

---

### الحل اللي بيقدمه الـ Reset Controller Framework

الـ kernel قدّم **Reset Controller Framework** كـ abstraction layer موحد:

1. الـ **reset controller** هو الـ hardware block اللي بيتحكم في مجموعة reset lines.
2. كل reset line ليها **ID** فريد جوا الـ controller.
3. الـ consumer drivers (زي audio driver) بتطلب الـ reset بالاسم من الـ Device Tree، من غير ما تعرف أي register أو أي bit.
4. الـ framework نفسه بيعمل locking وبيضمن إن الـ access آمن.

---

### التشبيه بالواقع — موزع الكهرباء في مصنع

تخيل مصنع فيه لوحة توزيع كهرباء مركزية (الـ reset controller). اللوحة دي فيها صفوف من الـ circuit breakers، كل واحد بيخدم ماكينة معينة (peripheral).

| الواقع | الـ Kernel Concept |
|--------|--------------------|
| لوحة التوزيع | `reset_controller_dev` |
| رقم الـ circuit breaker | reset ID (`unsigned long id`) |
| ضغط الـ breaker وترجيعه | `reset()` operation — assert ثم deassert |
| تثبيت الـ breaker في وضع إيقاف | `assert()` |
| تشغيل الـ breaker | `deassert()` |
| مهندس الصيانة اللي عارف مكان كل breaker | الـ driver اللي عنده `reset_control_ops` |
| العامل اللي بيطلب وقف ماكينة بالاسم | الـ consumer driver |
| سجل المصنع اللي فيه "ماكينة X → breaker رقم 5" | الـ Device Tree node + `of_xlate` |

الـ consumer (العامل) مش محتاج يعرف مكان الـ breaker — بيقول بس "إيقاف ماكينة X"، والـ framework يترجمها.

---

### الـ Big Picture Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                          │
│   (audio-driver, USB-driver, GPU-driver, ...)                    │
│                                                                  │
│   reset_control_reset(rstc);   ← API موحدة                       │
│   reset_control_assert(rstc);                                    │
│   reset_control_deassert(rstc);                                  │
└────────────────────────┬─────────────────────────────────────────┘
                         │  reset_control handle
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│               Reset Controller Framework (kernel/core)           │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  reset_controller_dev                                   │   │
│   │  ├── ops  ──────────────────────────► reset_control_ops│   │
│   │  │                                    ├── reset()       │   │
│   │  │                                    ├── assert()      │   │
│   │  │                                    └── deassert()    │   │
│   │  ├── nr_resets  (عدد الـ reset lines)                   │   │
│   │  ├── of_node    (Device Tree node)                      │   │
│   │  └── of_xlate   (DT specifier → ID)                    │   │
│   └─────────────────────────────────────────────────────────┘   │
└────────────────────────┬─────────────────────────────────────────┘
                         │  يستدعي ops->reset(rcdev, id)
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│              Platform-Specific Reset Driver                      │
│         (مثلاً: reset-meson-aux.c / reset-meson.c)              │
│                                                                  │
│   meson_reset_toggle_ops                                         │
│   ├── reset()  → كتابة على regmap register                       │
│   ├── assert() → set bit في register                             │
│   └── deassert() → clear bit في register                         │
│                                                                  │
│   ┌────────────────────────────┐                                 │
│   │  meson_reset_param         │                                 │
│   │  ├── reset_ops             │                                 │
│   │  ├── reset_num (عدد lines) │                                 │
│   │  └── level_offset (reg)    │                                 │
│   └────────────────────────────┘                                 │
└────────────────────────┬─────────────────────────────────────────┘
                         │  regmap_read / regmap_write
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Hardware Registers (MMIO)                     │
│            Reset Control Registers على الـ SoC                   │
└──────────────────────────────────────────────────────────────────┘
```

---

### الـ Auxiliary Bus — طبقة إضافية جوا الصورة

الـ file ده بالتحديد بيستخدم **Auxiliary Bus** مش الـ Platform Bus العادي. ده مهم نفهمه:

#### ليه Auxiliary Bus؟

في Amlogic، الـ audio clock controller (axg-audio-clkc) هو parent driver ضخم. هو اللي بيتحكم في الـ clocks والـ resets للـ audio subsystem في نفس الـ register space.

المشكلة: الـ audio clock driver والـ audio reset driver بيشاركوا نفس الـ `regmap`. لو اتسجلوا كـ platform devices مستقلين → مش هيقدروا يشاركوا الـ regmap بسهولة.

الحل: الـ audio clock driver بيعمل **auxiliary device** جديد ويحطه على الـ **auxiliary bus**. الـ `reset-meson-aux` driver بعدين بيعمل probe على الـ auxiliary device ده، ويجيب الـ regmap من الـ parent مباشرة:

```c
map = dev_get_regmap(adev->dev.parent, NULL);
```

```
┌──────────────────────────────────────┐
│   axg-audio-clkc  (Parent Driver)    │  ← يمتلك الـ regmap
│   ┌──────────────────────────────┐   │
│   │  auxiliary_device            │   │  ← يسجّل child device
│   │  name: "axg-audio-clkc.rst-g12a" │
│   └──────────────┬───────────────┘   │
└──────────────────┼───────────────────┘
                   │  Auxiliary Bus
                   ▼
┌──────────────────────────────────────┐
│   reset-meson-aux  (هذا الـ file)    │  ← يعمل probe على الـ child
│   probe():                           │
│   ├── يجيب regmap من parent          │
│   └── يسجّل reset controller        │
└──────────────────────────────────────┘
```

#### الـ Auxiliary Bus بإيجاز

**الـ Auxiliary Bus** هو bus مبني جوا الـ kernel بيسمح لـ driver واحد إنه يخلق sub-devices افتراضية. مش موجود في الـ hardware فعلاً، بس بيسمح بـ driver splitting بدون ما تخسر الـ shared state (زي الـ regmap هنا).

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية هي **فصل "مين يطلب الـ reset" عن "إزاي يتعمل الـ reset"**.

```
Consumer (audio driver)            Provider (meson reset driver)
       │                                       │
       │  reset_control_reset(rstc)            │
       │ ─────────────────────────────────────►│
       │                                       │
       │                    ops->reset(rcdev, id)
       │                                       │  bit manipulation
       │                                       │  على الـ register
```

الـ consumer مش محتاج يعرف:
- أي register بالظبط.
- أي bit جوا الـ register.
- إزاي الـ reset شغال (toggle أو level).
- مين اللي بيعمل locking.

---

### الـ Struct Relationships — العلاقات بين الـ Structs

```
meson_reset_param
┌────────────────────────┐
│ reset_ops ─────────────┼──► meson_reset_toggle_ops (reset_control_ops)
│ reset_num = 26         │    ┌────────────────────────────────────────┐
│ level_offset = 0x24    │    │ .reset   = meson_reset_toggle          │
└────────────────────────┘    │ .assert  = NULL                        │
           │                  │ .deassert = NULL                       │
           │ تُمرر إلى        └────────────────────────────────────────┘
           ▼
meson_reset_controller_register(dev, map, param)
           │
           │ تبني وتسجّل
           ▼
reset_controller_dev
┌────────────────────────────────────────────┐
│ ops      ──────────────────────────────────┼──► reset_control_ops
│ dev      ──────────────────────────────────┼──► struct device (auxiliary_device.dev)
│ nr_resets = 26                             │
│ of_node  ──────────────────────────────────┼──► DT node
└────────────────────────────────────────────┘

auxiliary_device_id
┌─────────────────────────────────────────────┐
│ name = "axg-audio-clkc.rst-g12a"           │
│ driver_data ───────────────────────────────►│ &meson_g12a_audio_param
└─────────────────────────────────────────────┘

auxiliary_driver
┌──────────────────────────────────┐
│ probe    = meson_reset_aux_probe │
│ id_table = meson_reset_aux_ids  │
└──────────────────────────────────┘
```

---

### إيه اللي الـ Framework بيمتلكه vs اللي بيفوضه للـ Driver

| المسؤولية | الـ Framework (kernel core) | الـ Driver (reset-meson-aux) |
|-----------|----------------------------|------------------------------|
| تخزين قائمة الـ reset controllers | ✓ | |
| الـ API للـ consumers (`reset_control_get`) | ✓ | |
| الـ locking عند الاستخدام | ✓ | |
| ترجمة DT specifier لـ ID (`of_xlate`) | ✓ (default) أو يفوضها | |
| معرفة الـ register address | | ✓ |
| bit manipulation على الـ register | | ✓ |
| تعريف عدد الـ reset lines | | ✓ (عبر `reset_num`) |
| الحصول على الـ regmap من الـ parent | | ✓ |
| ربط نفسه بالـ auxiliary device | | ✓ |

---

### flow الـ Probe بالتفصيل

```c
// 1. الـ kernel يلاقي auxiliary_device باسم "axg-audio-clkc.rst-g12a"
// 2. يطابقه مع meson_reset_aux_ids
// 3. يستدعي:

static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
{
    // استخراج الـ param من id->driver_data (pointer لـ meson_reset_param)
    const struct meson_reset_param *param =
        (const struct meson_reset_param *)(id->driver_data);

    // جيب الـ regmap من الـ parent driver (audio clock controller)
    // الـ parent هو اللي سجّل الـ auxiliary_device ده
    struct regmap *map = dev_get_regmap(adev->dev.parent, NULL);
    if (!map)
        return -EINVAL;  // الـ parent لسه مجهزش الـ regmap

    // سجّل الـ reset controller مع الـ framework
    // الـ framework دلوقتي يعرف: في X reset lines، هنا الـ ops بتاعتهم
    return meson_reset_controller_register(&adev->dev, map, param);
}
```

نقطة جوهرية: الـ `driver_data` في الـ `auxiliary_device_id` بيتستخدم كـ type-punned pointer على `meson_reset_param`. ده pattern شائع في الـ kernel بيخلي كل entry في الـ id_table تشير لـ config مختلف للـ hardware.

---

### الـ meson_reset_toggle_ops — ليه "toggle"؟

الـ Amlogic audio reset مش بيعمل assert/deassert منفصلين. بدلاً من كده، الـ reset بيتعمل بـ **toggle**: بيكتب 1 في الـ bit، وبعدين يكتب 0 — كل ده في عملية واحدة `reset()`. ولهيك الـ ops اسمه `toggle_ops` ومفيهوش `assert` أو `deassert` منفصلين.

ده بيعكس الـ hardware behavior — الـ reset register في الـ audio block مش بيحتفظ بـ state، بس بيستجيب للـ pulse.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Config Options — Cheatsheet

| الـ Symbol | النوع | القيمة / المعنى |
|---|---|---|
| `CONFIG_RESET_CONTROLLER` | Kconfig | لازم يكون enabled عشان الـ reset subsystem يشتغل |
| `MODULE_IMPORT_NS("MESON_RESET")` | namespace | بيعلن إن الـ module بيستورد symbols من namespace اسمها `MESON_RESET` |
| `MODULE_DEVICE_TABLE(auxiliary, ...)` | macro | بيولد جدول الـ IDs في الـ module بحيث الـ udev/kernel يقدر يعمل match |
| `module_auxiliary_driver(...)` | macro | بيعمل `module_init` و`module_exit` تلقائياً عن طريق `auxiliary_driver_register/unregister` |
| `kernel_ulong_t` | type | pointer-sized integer بيُستخدم لتخزين الـ `driver_data` بشكل portable |
| `level_low_reset` | bool field | لو `true` الـ reset active-low (0 = reset فعّال)، لو `false` active-high |

---

**الـ Devices المدعومة (من `meson_reset_aux_ids`):**

| الـ name | الـ SoC | `reset_num` | `level_offset` |
|---|---|---|---|
| `a1-audio-clkc.rst-a1` | A1 | 32 | `0x28` |
| `a1-audio-clkc.rst-a1-vad` | A1 (VAD) | 6 | `0x08` |
| `axg-audio-clkc.rst-g12a` | G12A | 26 | `0x24` |
| `axg-audio-clkc.rst-sm1` | SM1 | 39 | `0x28` |

---

### 1. الـ Structs المهمة

---

#### `struct meson_reset_param`
**الغرض:** بيحتوي على الـ configuration الخاصة بكل SoC — عدد الـ resets، الـ offset في الـ register map، ونوع الـ ops.

```c
struct meson_reset_param {
    const struct reset_control_ops *reset_ops; /* pointer للـ ops الخاصة بالـ toggle أو level */
    unsigned int reset_num;                    /* عدد الـ reset lines المتاحة */
    unsigned int reset_offset;                 /* offset للـ assert/deassert registers */
    unsigned int level_offset;                 /* offset لـ register الـ level reset */
    bool level_low_reset;                      /* active-low ولا active-high */
};
```

**الاتصالات:**
- بيُشير لـ `reset_control_ops` (إما `meson_reset_ops` أو `meson_reset_toggle_ops`)
- بيُمرَّر لـ `meson_reset_controller_register()` عشان يبني الـ `reset_controller_dev`
- بيُخزَّن في `auxiliary_device_id.driver_data` كـ `kernel_ulong_t` وبيتعمله cast جوه `probe`

---

#### `struct auxiliary_device_id`
**الغرض:** بيعرّف الـ match بين الـ auxiliary device والـ driver. الـ kernel بيقارن الـ `name` مع اسم الـ device على الـ bus.

```c
struct auxiliary_device_id {
    char name[...];          /* اسم الـ match مثلاً "a1-audio-clkc.rst-a1" */
    kernel_ulong_t driver_data; /* pointer مخزّن كـ integer — بيشاور على meson_reset_param */
};
```

**الاتصالات:**
- الـ `driver_data` بيحمل عنوان `meson_reset_param` المناسب لكل device
- الـ `id_table` في `auxiliary_driver` بيُشير لمصفوفة من هذا الـ struct

---

#### `struct auxiliary_driver`
**الغرض:** تعريف الـ driver على الـ auxiliary bus — مشابه لـ `platform_driver` بس للـ auxiliary bus.

```c
struct auxiliary_driver {
    int (*probe)(struct auxiliary_device *auxdev,
                 const struct auxiliary_device_id *id); /* entry point */
    void (*remove)(struct auxiliary_device *auxdev);
    void (*shutdown)(struct auxiliary_device *auxdev);
    int (*suspend)(struct auxiliary_device *auxdev, pm_message_t state);
    int (*resume)(struct auxiliary_device *auxdev);
    const char *name;
    struct device_driver driver;          /* الـ core driver struct */
    const struct auxiliary_device_id *id_table; /* جدول الـ IDs المدعومة */
};
```

**في الـ driver ده:**
```c
static struct auxiliary_driver meson_reset_aux_driver = {
    .probe    = meson_reset_aux_probe,
    .id_table = meson_reset_aux_ids,
    /* name مش محدد — بيتاخد من اسم الـ module تلقائياً */
};
```

**الاتصالات:**
- `.driver` بيشيله الـ bus core في قائمة الـ drivers المسجلين
- `.id_table` بيشاور على `meson_reset_aux_ids[]`
- `.probe` بيستقبل `auxiliary_device` ويجيب منه الـ `regmap` والـ `param`

---

#### `struct auxiliary_device`
**الغرض:** بيمثل sub-device على الـ auxiliary bus. الـ parent هو الـ audio clock driver (axg-audio-clkc أو a1-audio-clkc).

```c
struct auxiliary_device {
    struct device dev;     /* الـ device الأساسي — بيحمل الـ parent pointer */
    const char *name;      /* اسم الـ sub-device (الجزء بعد الـ modname) */
    u32 id;
    struct {
        struct xarray irqs;
        struct mutex lock;
        bool irq_dir_exists;
    } sysfs;
};
```

**الاتصالات:**
- `adev->dev.parent` بيشاور على الـ parent device (audio clock controller)
- `dev_get_regmap(adev->dev.parent, NULL)` بيجيب الـ `regmap` من الـ parent

---

#### `struct reset_control_ops`
**الغرض:** الـ vtable بتاع الـ reset controller — بيحدد العمليات المتاحة.

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

**في الـ driver ده:** بيُستخدم `meson_reset_toggle_ops` فقط (مش `meson_reset_ops`) — ده بيعمل toggle pulse على الـ reset line بدل assert/deassert منفصلين.

---

#### `struct reset_controller_dev`
**الغرض:** بيمثل الـ reset controller داخل الـ kernel reset subsystem. بيُنشأ جوه `meson_reset_controller_register()`.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;        /* الـ vtable */
    struct module *owner;
    struct list_head list;                      /* مربوط في قائمة عالمية */
    struct list_head reset_control_head;        /* قائمة الـ consumers */
    struct device *dev;                         /* الـ device المرتبط */
    struct device_node *of_node;
    const struct of_phandle_args *of_args;
    int of_reset_n_cells;
    int (*of_xlate)(...);
    unsigned int nr_resets;                     /* عدد الـ reset lines = reset_num */
};
```

---

#### `struct regmap`
**الغرض:** abstraction layer للـ register access. بيُجلب من الـ parent device ويُمرَّر للـ reset controller لتشغيل الـ register read/write.

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
  auxiliary_device_id[]
  ┌──────────────────────────────┐
  │ name: "a1-audio-clkc.rst-a1" │
  │ driver_data ─────────────────┼──► meson_reset_param
  └──────────────────────────────┘    ┌───────────────────────┐
                                      │ reset_ops ─────────────┼──► reset_control_ops
                                      │ reset_num = 32         │    ┌─────────────────┐
                                      │ level_offset = 0x28    │    │ .reset()        │
                                      └───────────────────────┘    │ .assert()       │
                                                                    │ .deassert()     │
                                                                    │ .status()       │
                                                                    └─────────────────┘

  auxiliary_driver
  ┌─────────────────────────────┐
  │ .probe = meson_reset_aux_   │
  │          probe              │
  │ .id_table ──────────────────┼──► auxiliary_device_id[]
  │ .driver (device_driver)     │
  └─────────────────────────────┘

  auxiliary_device (من الـ parent driver)
  ┌──────────────────────────────┐
  │ .dev                         │
  │   .parent ───────────────────┼──► parent device (audio-clkc)
  │ .name = "rst-a1"             │         │
  └──────────────────────────────┘         │ dev_get_regmap()
                                           ▼
                                      regmap
                                      ┌──────────────┐
                                      │ register I/O │
                                      └──────┬───────┘
                                             │ مُمرَّر لـ
                                             ▼
                                      reset_controller_dev
                                      ┌──────────────────────┐
                                      │ .ops ─────────────────┼──► reset_control_ops
                                      │ .nr_resets = 32       │
                                      │ .dev                  │
                                      │ .list (عالمي)         │
                                      └──────────────────────┘
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
[kernel boot / module load]
        │
        ▼
module_auxiliary_driver(meson_reset_aux_driver)
        │
        ├─► auxiliary_driver_register()
        │         │
        │         ▼
        │   الـ bus core يسجّل الـ driver ويدور على devices مطابقة
        │
        ▼
[auxiliary bus يجد device اسمه "a1-audio-clkc.rst-a1"]
        │
        ▼
meson_reset_aux_probe(adev, id)
        │
        ├─► cast id->driver_data → meson_reset_param*
        │
        ├─► dev_get_regmap(adev->dev.parent, NULL)
        │         │
        │         ▼
        │   يجيب الـ regmap من الـ parent audio clock driver
        │
        ├─► meson_reset_controller_register(&adev->dev, map, param)
        │         │
        │         ▼
        │   ينشئ reset_controller_dev
        │   يسجله في الـ reset subsystem
        │   (devm — بيتحرر تلقائياً عند remove)
        │
        ▼
[الـ driver شغّال — consumers يقدروا يطلبوا reset]

[module unload / device removed]
        │
        ▼
devm cleanup → reset_controller_unregister()
        │
        ▼
auxiliary_driver_unregister()
        │
        ▼
[انتهى]
```

---

### 4. مخطط Call Flow

#### عند الـ Probe:

```
kernel auxiliary bus core
  → meson_reset_aux_probe(adev, id)
      → (cast) id->driver_data → const struct meson_reset_param *param
      → dev_get_regmap(adev->dev.parent, NULL)
            → الـ parent (audio-clkc driver) رجّع regmap
      → meson_reset_controller_register(&adev->dev, map, param)
            → [في reset-meson.c]
              → devm_kzalloc() لـ reset_controller_dev
              → rcdev->ops    = param->reset_ops      (= &meson_reset_toggle_ops)
              → rcdev->nr_resets = param->reset_num   (= 32 مثلاً)
              → devm_reset_controller_register(dev, rcdev)
                    → reset_controller_register(rcdev)
                          → إضافة rcdev لـ قائمة عالمية
                          → return 0
```

#### عند طلب Reset من consumer:

```
consumer driver
  → reset_control_reset(rstc)
      → reset subsystem core
            → rcdev->ops->reset(rcdev, id)
                  → meson_reset_toggle_ops.reset()
                        → regmap_write() [assert]
                        → regmap_write() [deassert]
                              → hardware audio reset register
```

---

### 5. استراتيجية الـ Locking

الـ driver ده **بسيط جداً** من ناحية الـ locking — ما فيش locks صريحة في الـ file ده نفسه. كل الـ locking موجود في الـ layers التحتانية:

| الـ Layer | الـ Lock | بيحمي إيه |
|---|---|---|
| **auxiliary bus core** | `device_lock` (embedded في `struct device`) | الـ probe/remove serialization |
| **reset subsystem** | mutex داخلي في الـ core | قائمة `reset_controller_dev` العالمية عند register/unregister |
| **regmap** | spinlock أو mutex (حسب الـ config) | الـ register read/write operations |
| **sysfs (auxiliary_device)** | `adev->sysfs.lock` (mutex) | IRQ sysfs entries |

**ترتيب الـ Locks (Lock Ordering):**
```
device_lock (bus level)
    └─► reset subsystem mutex
            └─► regmap lock (spinlock/mutex)
                    └─► hardware register access
```

**ملاحظة مهمة:** الـ `dev_get_regmap()` في `probe` تُستدعى بعد ما الـ bus يمسك `device_lock` — ده بيضمن إن الـ parent device موجود ومش بيتحرر في نفس الوقت. الـ `devm_*` functions بتضمن إن الـ cleanup بيحصل بالترتيب الصح عند الـ remove.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Function Cheatsheet

| Function | File | Category | الغرض |
|---|---|---|---|
| `meson_reset_aux_probe` | `reset-meson-aux.c` | Registration | Entry point للـ auxiliary driver |
| `meson_reset_controller_register` | `reset-meson-common.c` | Registration | يسجّل الـ reset controller مع الـ kernel |
| `meson_reset_offset_and_bit` | `reset-meson-common.c` | Helper | يحسب الـ register offset والـ bit position |
| `meson_reset_reset` | `reset-meson-common.c` | Runtime | تنفيذ pulse reset (write-only) |
| `meson_reset_assert` | `reset-meson-common.c` | Runtime | تثبيت الـ reset (assert) |
| `meson_reset_deassert` | `reset-meson-common.c` | Runtime | رفع الـ reset (deassert) |
| `meson_reset_level` | `reset-meson-common.c` | Helper | الـ common logic للـ assert/deassert |
| `meson_reset_level_toggle` | `reset-meson-common.c` | Runtime | assert ثم deassert فوراً |
| `meson_reset_status` | `reset-meson-common.c` | Runtime | قراءة الحالة الحالية للـ reset line |
| `dev_get_regmap` | kernel API | Helper | جلب الـ regmap من الـ parent device |

---

### Group 1: Registration

هذه الـ functions مسؤولة عن ربط الـ driver بالـ auxiliary bus وتسجيل الـ reset controller مع الـ reset subsystem. التسلسل: `module_init` → `auxiliary_driver_register` → `probe` → `meson_reset_controller_register` → `devm_reset_controller_register`.

---

#### `meson_reset_aux_probe`

```c
static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
```

**الـ function** دي هي الـ probe callback للـ auxiliary driver. بتجيب الـ `meson_reset_param` المرتبطة بالـ device من الـ `id->driver_data`، وبعدين بتجيب الـ `regmap` من الـ parent device عن طريق `dev_get_regmap`، وأخيراً بتسجّل الـ reset controller.

**Parameters:**
- `adev` — الـ `struct auxiliary_device` اللي اتعمل من الـ parent (audio clock controller). بيحتوي على `dev.parent` اللي منه بنجيب الـ `regmap`.
- `id` — الـ matched entry من `meson_reset_aux_ids`. الـ `driver_data` فيها pointer للـ `meson_reset_param` الصح.

**Return:** `0` عند النجاح، أو error code سلبي (`-EINVAL` لو مفيش regmap، أو الـ error اللي بيرجعه `meson_reset_controller_register`).

**Key details:**
- مفيش explicit locking هنا — الـ auxiliary bus framework بيضمن إن الـ probe بيتكلّم بشكل serialized.
- الـ `regmap` مش بتتعمل هنا — هي موجودة بالفعل في الـ parent (audio clock controller) وبتُشارك عن طريق `dev_get_regmap(adev->dev.parent, NULL)`.
- لو `map == NULL`، بيرجع `-EINVAL` مباشرةً — مفيش logging لأن الـ parent المفروض يكون حاطط الـ regmap بشكل صحيح.
- كل الـ allocation بيتعمل بـ `devm_*` في `meson_reset_controller_register`، فالـ cleanup تلقائي عند الـ remove.

**Caller context:** الـ auxiliary bus core بيستدعيها من process context لما بيلاقي match بين الـ device والـ driver.

**Pseudocode:**
```
probe(adev, id):
    param = id->driver_data           // cast to meson_reset_param*
    map   = dev_get_regmap(adev->dev.parent, NULL)
    if !map → return -EINVAL
    return meson_reset_controller_register(&adev->dev, map, param)
```

---

#### `meson_reset_controller_register`

```c
int meson_reset_controller_register(struct device *dev, struct regmap *map,
                                    const struct meson_reset_param *param)
```

**الـ function** دي هي الـ core registration routine المشتركة بين الـ platform driver والـ auxiliary driver. بتعمل `devm_kzalloc` لـ `struct meson_reset`، بتملأ الـ `rcdev` fields، وبتسجّل الـ controller مع الـ reset subsystem.

**Parameters:**
- `dev` — الـ device struct (في حالتنا `&adev->dev`). بيُستخدم للـ devm allocation والـ `of_node`.
- `map` — الـ `regmap` الـ shared مع الـ parent. كل الـ register access بيمر منه.
- `param` — الـ immutable config (عدد الـ resets، الـ offsets، الـ ops).

**Return:** `0` عند النجاح، أو `-ENOMEM` لو فشل الـ allocation، أو الـ error من `devm_reset_controller_register`.

**Key details:**
- الـ `rcdev.of_node` بيُعيَّن لـ `dev->of_node` — في الـ auxiliary case دي بتكون `NULL` لأن الـ auxiliary devices مش موجودة في الـ DT مباشرةً، لكن ده مقبول لأن الـ consumers بيحصلوا على الـ reset handle بطرق تانية (مثلاً `of_reset_control_get` من الـ parent node).
- الـ `devm_reset_controller_register` بيضمن إن الـ unregister بيحصل تلقائياً عند الـ device removal.
- الـ `rcdev.owner = dev->driver->owner` بيمنع الـ module من الـ unload وفيه users.

**Caller context:** Process context فقط (الـ probe path).

```
meson_reset_controller_register(dev, map, param):
    data = devm_kzalloc(dev, sizeof(*data))
    if !data → return -ENOMEM

    data->param           = param
    data->map             = map
    data->rcdev.owner     = dev->driver->owner
    data->rcdev.nr_resets = param->reset_num
    data->rcdev.ops       = param->reset_ops
    data->rcdev.of_node   = dev->of_node

    return devm_reset_controller_register(dev, &data->rcdev)
```

---

### Group 2: Runtime Reset Operations

الـ functions دي هي الـ actual hardware operations اللي بيطلبها الـ consumers عبر الـ reset subsystem API. بتشتغل كلها من خلال الـ `reset_control_ops` المسجّلة في الـ `rcdev`.

في الـ auxiliary driver، الـ `param->reset_ops` دايماً بيكون `&meson_reset_toggle_ops` — يعني الـ `reset` operation بيعمل assert+deassert وملوش write-pulse hardware register زي الـ platform driver.

---

#### `meson_reset_offset_and_bit`

```c
static void meson_reset_offset_and_bit(struct meson_reset *data,
                                       unsigned long id,
                                       unsigned int *offset,
                                       unsigned int *bit)
```

**الـ helper** دي بتحوّل الـ linear reset ID لـ register offset وbit number. الـ stride بتجيبه من الـ regmap نفسه (عادةً 4 bytes = 32 bit) فالحساب بيكون: `offset = (id / 32) * 4` و `bit = id % 32`.

**Parameters:**
- `data` — الـ private driver data اللي فيها الـ `regmap`.
- `id` — الـ reset line ID (من 0 لـ `reset_num - 1`).
- `offset` — output: byte offset من بداية الـ register bank.
- `bit` — output: bit position داخل الـ register.

**Return:** void — النتيجة في الـ output pointers.

**Key details:** الـ offset ده relative — الـ caller بيضيف عليه `reset_offset` أو `level_offset` حسب العملية. ده يخلّي الـ helper generic لكلا النوعين.

---

#### `meson_reset_level`

```c
static int meson_reset_level(struct reset_controller_dev *rcdev,
                             unsigned long id, bool assert)
```

**الـ common backend** لكل من `meson_reset_assert` و`meson_reset_deassert`. بيحسب الـ offset، وبيطبّق الـ `level_low_reset` polarity XOR، وبيعمل `regmap_update_bits` لتعديل بت واحد بس من غير ما يأثر على الباقي.

**Parameters:**
- `rcdev` — pointer للـ `reset_controller_dev`، الـ function بتعمل `container_of` للوصول لـ `struct meson_reset`.
- `id` — رقم الـ reset line.
- `assert` — `true` يعني put in reset، `false` يعني release from reset.

**Return:** return value الـ `regmap_update_bits` (0 أو error).

**Key details:**
- `assert ^= data->param->level_low_reset` — الـ XOR ده بيعكس الـ polarity لو الـ hardware بيستخدم active-low reset. في الـ audio auxiliary driver، `level_low_reset = false`، فمفيش inversion.
- `regmap_update_bits` بيعمل read-modify-write atomically (مع locking جوّا الـ regmap).

---

#### `meson_reset_assert`

```c
static int meson_reset_assert(struct reset_controller_dev *rcdev,
                              unsigned long id)
```

بتضع الـ reset line في حالة assert (in-reset). بتستدعي `meson_reset_level(rcdev, id, true)`.

**Caller:** الـ reset subsystem لما يُستدعى `reset_control_assert()` من الـ consumer driver.

---

#### `meson_reset_deassert`

```c
static int meson_reset_deassert(struct reset_controller_dev *rcdev,
                                unsigned long id)
```

بترفع الـ reset line (release from reset). بتستدعي `meson_reset_level(rcdev, id, false)`.

**Caller:** الـ reset subsystem لما يُستدعى `reset_control_deassert()`.

---

#### `meson_reset_reset`

```c
static int meson_reset_reset(struct reset_controller_dev *rcdev,
                             unsigned long id)
```

**الـ function** دي خاصة بـ `meson_reset_ops` (الـ platform driver). بتكتب BIT واحد في الـ `reset_offset` register — الـ hardware بيتعامل مع ده كـ self-clearing pulse reset. **مش موجودة في الـ `meson_reset_toggle_ops`** المستخدمة في الـ auxiliary driver.

**Return:** return value الـ `regmap_write`.

**Key details:** الـ write هنا write-only على register مختلف عن الـ `level_offset`. الـ bit بيتكلّم hardware وبيرجع لـ 0 لوحده.

---

#### `meson_reset_level_toggle`

```c
static int meson_reset_level_toggle(struct reset_controller_dev *rcdev,
                                    unsigned long id)
```

**الـ function** دي هي الـ `reset` op في `meson_reset_toggle_ops` — اللي بيستخدمها الـ auxiliary driver. بتعمل `assert` وبعدين `deassert` على نفس الـ level register. ده مناسب للـ audio IPs اللي مش عندها hardware pulse-reset register.

**Return:** `0` عند النجاح، أو error من أول `assert` (لو فشل، الـ deassert مش بيتكلّم).

```
level_toggle(rcdev, id):
    ret = meson_reset_assert(rcdev, id)
    if ret → return ret
    return meson_reset_deassert(rcdev, id)
```

---

#### `meson_reset_status`

```c
static int meson_reset_status(struct reset_controller_dev *rcdev,
                              unsigned long id)
```

**بتقرأ** الحالة الحالية للـ reset line من الـ `level_offset` register. بتأخذ الـ bit value وبتطبّق الـ `level_low_reset` XOR لإرجاع المعنى الصحيح: `1` = in reset, `0` = released.

**Return:** `1` لو الـ reset مفعّل، `0` لو مرفوع، أو error من الـ `regmap_read` (في الـ regmap implementation، الـ read errors نادرة لـ MMIO).

**Key details:** الـ `regmap_read` في الـ auxiliary case بيقرأ من الـ parent device's regmap — فالـ concurrent access مع الـ audio clock driver ممكن يحصل، والـ regmap locking بيحمي ده.

---

### Group 3: Static Data (Device Tables & Params)

| Struct | Reset Num | Level Offset | الجهاز |
|---|---|---|---|
| `meson_a1_audio_param` | 32 | `0x28` | A1 — audio main |
| `meson_a1_audio_vad_param` | 6 | `0x08` | A1 — VAD (Voice Activity Detection) |
| `meson_g12a_audio_param` | 26 | `0x24` | G12A / AXG audio |
| `meson_sm1_audio_param` | 39 | `0x28` | SM1 audio |

كل الـ params بتستخدم `meson_reset_toggle_ops` لأن الـ audio register bank مش عنده dedicated pulse-reset registers. الـ `level_low_reset` مش محدودة (default = `false`) يعني active-high assert.

الـ `meson_reset_aux_ids` table بيربط الـ auxiliary device name (اللي بيتكوّن من `KBUILD_MODNAME.devname`) بالـ param الصح عبر `driver_data`. الـ `MODULE_DEVICE_TABLE(auxiliary, ...)` بيولّد الـ modalias اللي بيخلّي الـ udev يلود الـ module تلقائياً.

---

### Group 4: Module Boilerplate

#### `module_auxiliary_driver`

```c
module_auxiliary_driver(meson_reset_aux_driver);
```

**الـ macro** دي بتتوسّع لـ `module_init` و`module_exit` اللي بيستدعوا `auxiliary_driver_register` و`auxiliary_driver_unregister`. بتوفّر الـ boilerplate لـ drivers بسيطة مفيهاش حاجة خاصة في الـ init/exit.

**الـ namespace:** الـ `MODULE_IMPORT_NS("MESON_RESET")` بيسمح لـ module إنه يستخدم الـ symbols المصدّرة بـ `EXPORT_SYMBOL_NS_GPL(..., "MESON_RESET")` من `reset-meson-common.c` — وهي `meson_reset_toggle_ops` و`meson_reset_controller_register`.
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده هو `reset-meson-aux` — بيشتغل على الـ **Auxiliary Bus** وبيوفر reset controller للـ audio subsystem في chips الـ Amlogic (A1, G12A, SM1). الـ debugging بتاعته بيتوزع على مستويين: software (kernel internals) وhardware (registers فعلية).

---

### Software Level

#### 1. Debugfs Entries

الـ reset framework بيعمل entries تحت `/sys/kernel/debug/reset/`:

```bash
# اتأكد إن debugfs متعمل
mount | grep debugfs
# لو مش متعمل:
mount -t debugfs none /sys/kernel/debug

# اعرض كل الـ reset controllers المتسجلين
ls /sys/kernel/debug/reset/

# اقرأ state الـ reset lines لأي controller
cat /sys/kernel/debug/reset/<controller-name>/reset0
cat /sys/kernel/debug/reset/<controller-name>/reset1
```

**مثال على output:**
```
reset0: asserted
reset1: deasserted
reset2: deasserted
```

الـ `asserted` يعني الـ block لسه في reset — لازم `deasserted` عشان يشتغل.

الـ auxiliary bus نفسه معندوش debugfs خاص بيه، لكن كل device بيظهر في `/sys/bus/auxiliary/devices/`.

---

#### 2. Sysfs Entries

```bash
# اعرض كل الـ auxiliary devices المتسجلين
ls /sys/bus/auxiliary/devices/

# الـ devices المتوقعين من الـ driver ده
ls /sys/bus/auxiliary/devices/ | grep -E "a1-audio-clkc|axg-audio-clkc"

# مثال على اسم device:
# a1-audio-clkc.rst-a1.0
# axg-audio-clkc.rst-g12a.0

# اتحقق من الـ driver اللي bind عليه
cat /sys/bus/auxiliary/devices/axg-audio-clkc.rst-g12a.0/driver

# اعرض الـ parent
cat /sys/bus/auxiliary/devices/axg-audio-clkc.rst-g12a.0/../uevent

# اتحقق من reset controller المرتبط
ls /sys/bus/platform/devices/*/reset_controller/
```

---

#### 3. Ftrace — Tracepoints وEvents

```bash
# اتأكد من وجود tracepoints الـ reset subsystem
ls /sys/kernel/debug/tracing/events/reset/

# فعّل كل events الـ reset
echo 1 > /sys/kernel/debug/tracing/events/reset/enable

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_reset/enable
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_assert/enable
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_deassert/enable
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_status/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# تتبع الـ probe function بالـ function_graph tracer
echo meson_reset_aux_probe > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الـ module ...
cat /sys/kernel/debug/tracing/trace
```

**مثال output متوقع:**
```
     kworker-123   [000] ....  1234.5: reset_control_deassert: id=5
     kworker-123   [000] ....  1234.6: reset_control_assert:   id=5
```

---

#### 4. Printk و Dynamic Debug

الـ driver نفسه معندوش `dev_dbg` calls صريحة، لكن الـ reset framework والـ regmap بيعملوا logging:

```bash
# فعّل dynamic debug للـ reset subsystem كله
echo 'file drivers/reset/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ meson reset driver تحديداً
echo 'file drivers/reset/amlogic/reset-meson-aux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/reset/amlogic/reset-meson.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ regmap (بيغطي قراءات وكتابات الـ registers)
echo 'file drivers/base/regmap/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ auxiliary bus
echo 'file drivers/base/auxiliary.c +p' > /sys/kernel/debug/dynamic_debug/control

# اقرأ الـ kernel log
dmesg -w | grep -E "meson|reset|auxiliary|regmap"
```

**عشان تشوف الـ regmap writes على الـ reset registers:**
```bash
echo 'file drivers/base/regmap/regmap.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | grep regmap
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_RESET_CONTROLLER` | تفعيل الـ reset framework الأساسي (لازم يكون `y`) |
| `CONFIG_DEBUG_FS` | تفعيل debugfs عشان تشوف `/sys/kernel/debug/reset/` |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug messages |
| `CONFIG_REGMAP_DEBUG` | debug messages من الـ regmap layer |
| `CONFIG_AUXILIARY_BUS` | تفعيل الـ auxiliary bus نفسه |
| `CONFIG_PM_DEBUG` | تتبع power management transitions |
| `CONFIG_KALLSYMS_ALL` | أسماء كاملة في stack traces |
| `CONFIG_PROVE_LOCKING` | كشف lock ordering problems |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | كشف sleep in atomic context |

```bash
# اتحقق من الـ configs الحالية
zcat /proc/config.gz | grep -E "CONFIG_RESET|CONFIG_REGMAP|CONFIG_AUXILIARY|CONFIG_DEBUG_FS"
```

---

#### 6. Devlink وأدوات الـ Subsystem

الـ driver ده مش بيستخدم devlink، لكن في أدوات مفيدة:

```bash
# اعرض كل الـ reset controllers المتسجلين عن طريق sysfs
find /sys -name "nr_resets" 2>/dev/null

# اتحقق من الـ module نفسه
lsmod | grep meson_reset
modinfo meson_reset_aux

# اعرض ربط الـ driver بالـ device
ls -la /sys/bus/auxiliary/devices/axg-audio-clkc.rst-g12a.0/driver

# فرض unbind/rebind للـ driver (اختبار الـ probe)
echo "axg-audio-clkc.rst-g12a.0" > /sys/bus/auxiliary/drivers/meson_reset_aux/unbind
echo "axg-audio-clkc.rst-g12a.0" > /sys/bus/auxiliary/drivers/meson_reset_aux/bind

# اعرض الـ regmap لكل device
ls /sys/kernel/debug/regmap/
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة في dmesg | السبب | الحل |
|---|---|---|
| `meson_reset_aux: probe of ... failed with error -EINVAL` | `dev_get_regmap()` رجع `NULL` — الـ parent driver مش عامل regmap | اتأكد إن الـ parent audio clock driver بيعمل `devm_regmap_init()` |
| `auxiliary: failed to create auxiliary device` | مشكلة في تسجيل الـ device من الـ parent | اتحقق من الـ parent driver logs |
| `meson_reset_aux: no driver found for auxiliary device` | اسم الـ device في `id_table` مش متطابق مع الـ device المسجل | قارن `auxiliary_device_id.name` مع اسم الـ device في الـ bus |
| `reset: ... already in use` | محاولة استخدام نفس الـ reset line من أكتر من consumer بدون `shared` flag | استخدم `reset_control_get_shared()` في الـ consumer |
| `regmap: ... out of range` | الـ reset ID أكبر من `reset_num` | اتحقق من صحة `reset_num` في `meson_reset_param` |
| `meson_reset_controller_register failed` | فشل `devm_reset_controller_register()` — غالباً memory issue | اتحقق من `dmesg` قبل السطر ده |
| `No regmap found for device` | الـ parent device مش عامل regmap | اتحقق إن audio clock driver بيستخدم `regmap_init_mmio` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
{
    const struct meson_reset_param *param =
        (const struct meson_reset_param *)(id->driver_data);
    struct regmap *map;

    /* نقطة 1: تحقق إن الـ param مش NULL */
    if (WARN_ON(!param)) {
        dump_stack();
        return -EINVAL;
    }

    map = dev_get_regmap(adev->dev.parent, NULL);

    /* نقطة 2: الأهم — لو map = NULL معناه الـ parent مش جاهز */
    if (WARN_ON(!map)) {
        dev_err(&adev->dev, "parent regmap not found — parent not ready?\n");
        dump_stack();
        return -EINVAL;
    }

    /* نقطة 3: تحقق من صحة reset_num */
    WARN_ON(param->reset_num == 0);

    return meson_reset_controller_register(&adev->dev, map, param);
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تتطابق مع الـ Kernel State

الـ reset registers موجودة داخل الـ audio clock controller MMIO block. الـ `level_offset` هو الـ offset الفعلي للـ reset level register:

| SoC | `level_offset` | `reset_num` |
|---|---|---|
| A1 Audio | `0x28` | 32 |
| A1 Audio VAD | `0x08` | 6 |
| G12A Audio | `0x24` | 26 |
| SM1 Audio | `0x28` | 39 |

للتحقق من الـ reset state الفعلي في الـ hardware:

```bash
# أولاً: اعرف الـ base address للـ audio clock controller
cat /proc/iomem | grep -i "audio"
# مثال: ff642000-ff642fff : axg-audio-clkc

# اقرأ الـ reset level register مباشرة (G12A مثلاً)
# base = 0xff642000, level_offset = 0x24
devmem2 0xff642024 w
```

**تفسير النتيجة:**
- كل bit يمثل reset line واحد
- `bit = 1` → deasserted (شغال)
- `bit = 0` → asserted (في reset)

هذا يعتمد على `level_low_reset` في الـ params — لو `true`، الـ polarity معكوس.

---

#### 2. Register Dump Techniques

```bash
# طريقة 1: devmem2 (الأشهر)
# لازم تثبته: apt install devmem2
devmem2 <physical_address> w

# طريقة 2: /dev/mem مع Python
python3 -c "
import mmap, struct
PAGE_SIZE = 4096
BASE = 0xff642000  # audio clkc base
OFFSET = 0x24      # G12A level_offset

with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), PAGE_SIZE,
                    mmap.MAP_SHARED, mmap.PROT_READ,
                    offset=BASE & ~(PAGE_SIZE-1))
    val = struct.unpack('<I', mem[(BASE & (PAGE_SIZE-1)) + OFFSET:
                                  (BASE & (PAGE_SIZE-1)) + OFFSET + 4])[0]
    print(f'Reset Level Register: 0x{val:08X}')
    for i in range(32):
        state = 'DEASSERTED' if (val >> i) & 1 else 'ASSERTED'
        print(f'  reset[{i:2d}]: {state}')
"

# طريقة 3: io utility (من package: ioport)
io -4 0xff642024

# طريقة 4: regmap debugfs (الأفضل لو kernel supports it)
ls /sys/kernel/debug/regmap/
# اختار الـ regmap المرتبط بالـ audio clkc
cat /sys/kernel/debug/regmap/ff642000.audio-clkc/registers
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ reset signals في Amlogic SoCs بتكون internal — مش exposed على pins خارجية مباشرة. لكن:

```
تقنيات عملية:
──────────────

1. راقب الـ I2S / SPDIF output:
   - لو الـ audio codec شغال لكن مفيش output →
     غالباً الـ audio DMA block لسه في reset
   - ابحث عن BCLK/LRCLK/SDATA على الـ oscilloscope

2. راقب الـ MCLK (Master Clock):
   - لو الـ MCLK موجود لكن الـ audio مش شغال →
     راجع الـ reset state للـ I2S block

3. طريقة indirect:
   - فعّل audio DMA transfer
   - راقب الـ IRQ line للـ audio DMA
   - لو مفيش IRQ → الـ block لسه في reset

Trigger Setup (Logic Analyzer):
   Channel 1: BCLK (expected: 3.072 MHz for 48kHz audio)
   Channel 2: LRCLK (expected: 48 kHz)
   Trigger: Rising edge on BCLK
   Sample Rate: >= 10x BCLK = 30 MHz minimum
```

---

#### 4. Hardware Issues الشائعة → Kernel Log Patterns

| مشكلة hardware | Pattern في dmesg | التشخيص |
|---|---|---|
| الـ audio block مش بيخرج صوت رغم إن الـ driver شغال | `meson-axg-sound: ... FIFO underrun` | الـ audio DMA block لسه في reset — اتحقق من reset ID |
| الـ parent clock driver مش بيشتغل | `axg-audio-clkc: probe deferred` | الـ reset-aux driver هيتأخر هو كمان (probe defer cascade) |
| تعارض في regmap access | `regmap: ... cache sync failed` | أكتر من consumer بيكتب على نفس الـ register في نفس الوقت |
| الـ reset مش بيحصل toggle صح | `meson_reset: timeout waiting for reset` | مشكلة في clock الـ audio subsystem نفسه |
| الـ SoC مش بيوقف من power management | `meson-reset: failed to assert reset` | الـ clock gate للـ audio block متقفل |

---

#### 5. Device Tree Debugging

الـ driver ده بيتعمل instantiation من الـ audio clock controller كـ **auxiliary device** — مش من DT مباشرة. لكن الـ parent بيتعمل من DT:

```bash
# اتحقق من الـ DT node للـ parent audio clock controller
find /proc/device-tree -name "audio-clkc*" -o -name "*axg*audio*" 2>/dev/null
ls /proc/device-tree/soc/bus*/audio-clkc*/

# اقرأ الـ compatible string
cat /proc/device-tree/soc/bus/audio-clkc@*/compatible | tr '\0' '\n'
# المتوقع: amlogic,axg-audio-clkc أو amlogic,a1-audio-clkc

# اتحقق من الـ reg (base address)
hexdump -C /proc/device-tree/soc/bus/audio-clkc@*/reg
# المتوقع: FF 64 20 00 00 00 10 00  (0xff642000, size 0x1000)

# اتحقق من الـ resets المرتبطة بالـ audio clkc نفسه
cat /proc/device-tree/soc/bus/audio-clkc@*/resets 2>/dev/null | hexdump -C

# تحقق إن الـ DT node موجود وصح
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "audio-clkc"
```

**DT مثال متوقع للـ G12A:**
```dts
audio_clkc: clock-controller@ff642000 {
    compatible = "amlogic,g12a-audio-clkc";
    reg = <0x0 0xff642000 0x0 0x1000>;
    clocks = <&clkc CLKID_AUDIO>;
    clock-names = "pclk";
    #clock-cells = <1>;
    #reset-cells = <1>;
};
```

لو الـ `reg` غلط أو الـ `compatible` مش متطابق مع الـ driver، الـ parent مش هيتعمل، وبالتالي الـ auxiliary device مش هيتنشأ، والـ reset controller مش هيتسجل.

---

### Practical Commands

#### سيناريو 1: التحقق من إن الـ driver اتحمل صح

```bash
# تحقق من وجود الـ module
lsmod | grep meson_reset
# المتوقع:
# meson_reset_aux        16384  0
# meson_reset            16384  1 meson_reset_aux

# تحقق من تسجيل الـ auxiliary devices
ls /sys/bus/auxiliary/devices/ | grep "rst"
# المتوقع:
# axg-audio-clkc.rst-g12a.0
# axg-audio-clkc.rst-sm1.0

# تحقق من ربط الـ driver
readlink /sys/bus/auxiliary/devices/axg-audio-clkc.rst-g12a.0/driver
# المتوقع:
# ../../../bus/auxiliary/drivers/meson_reset_aux
```

#### سيناريو 2: مشكلة probe — الـ driver مش بيشتغل

```bash
# اتحقق من الـ kernel log
dmesg | grep -E "meson.reset|auxiliary|audio.clkc" | tail -30

# فعّل dynamic debug وأعد تحميل الـ module
echo 'file drivers/reset/amlogic/* +p' > /sys/kernel/debug/dynamic_debug/control
modprobe -r meson_reset_aux && modprobe meson_reset_aux
dmesg | tail -20

# اتحقق من وجود الـ parent regmap
ls /sys/kernel/debug/regmap/ | grep audio
# لو مفيش → الـ parent driver مش عامل regmap → هيفشل بـ -EINVAL
```

#### سيناريو 3: قراءة حالة الـ reset registers مباشرة

```bash
#!/bin/bash
# script لقراءة كل الـ reset states لـ G12A audio

BASE=0xff642000  # base address من /proc/iomem
OFFSET=0x24      # level_offset للـ G12A
ADDR=$((BASE + OFFSET))

echo "Reading reset level register at 0x$(printf '%08X' $ADDR)"
VAL=$(devmem2 $ADDR w 2>/dev/null | grep "Value:" | awk '{print $NF}')
echo "Raw value: $VAL"

# فك الـ bits
VAL_DEC=$((16#${VAL#0x}))
for i in $(seq 0 25); do
    STATE=$(( (VAL_DEC >> i) & 1 ))
    [ $STATE -eq 1 ] && S="DEASSERTED" || S="ASSERTED (in reset)"
    printf "  reset[%2d]: %s\n" $i "$S"
done
```

#### سيناريو 4: مراقبة الـ reset operations لحظة بلحظة

```bash
# فعّل ftrace مع filter على الـ reset functions
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo "function" > /sys/kernel/debug/tracing/current_tracer
echo "meson_reset*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية اللي عايز تتتبعها (مثلاً تشغيل audio)
aplay /dev/zero -d 1 -f S16_LE -r 48000 -c 2 &

# اقرأ الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -50
```

**مثال output:**
```
# tracer: function
#
          aplay-1234  [001] .... 5678.90: meson_reset_toggle_ops+0x10/0x40
          aplay-1234  [001] .... 5678.91: meson_reset_controller_register+0x0/0x80
```

#### سيناريو 5: التحقق الكامل من الـ auxiliary bus chain

```bash
#!/bin/bash
echo "=== Auxiliary Bus Reset Devices ==="
for dev in /sys/bus/auxiliary/devices/*rst*; do
    name=$(basename $dev)
    drv=$(readlink $dev/driver 2>/dev/null | xargs basename 2>/dev/null || echo "UNBOUND")
    parent=$(cat $dev/uevent | grep DEVPATH | cut -d= -f2)
    echo "Device: $name"
    echo "  Driver:  $drv"
    echo "  Parent:  $(dirname $parent)"
    echo "---"
done

echo ""
echo "=== Reset Controllers ==="
find /sys -name "nr_resets" 2>/dev/null | while read f; do
    echo "Controller: $(dirname $f)"
    echo "  nr_resets: $(cat $f)"
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Audio لا يشتغل على Android TV Box بـ S905X3 (SM1)

#### العنوان
**الـ HDMI Audio ميشتغلش بعد boot على TV Box بـ Amlogic S905X3**

#### السياق
شركة بتعمل Android TV box بـ **Amlogic S905X3 (SM1 family)**. الـ BSP مبني على Linux 6.6. بعد ما الجهاز اشتغل، الـ HDMI video ماشي تمام بس الـ audio مش بيطلع خالص — لا HDMI audio ولا jack audio.

#### المشكلة
الـ audio subsystem مش بيشتغل. الـ `aplay -l` بيرجع قائمة فاضة. الـ kernel log فيه:

```
axg-audio-clkc: failed to register reset controller
meson-reset-aux: probe of axg-audio-clkc.rst-sm1.0 failed with error -22
```

#### التحليل
الـ flow من الكود هو:

```c
/* الـ probe بيتكلم على اسم axg-audio-clkc.rst-sm1 */
static const struct auxiliary_device_id meson_reset_aux_ids[] = {
    ...
    {
        .name = "axg-audio-clkc.rst-sm1",
        .driver_data = (kernel_ulong_t)&meson_sm1_audio_param,
    }, {}
};
```

الـ `meson_reset_aux_probe` بيعمل التالي:

```c
static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
{
    const struct meson_reset_param *param =
        (const struct meson_reset_param *)(id->driver_data);
    struct regmap *map;

    /* بيجيب الـ regmap من الـ parent device */
    map = dev_get_regmap(adev->dev.parent, NULL);
    if (!map)
        return -EINVAL;  /* هنا بيحصل الـ -22 = EINVAL */

    return meson_reset_controller_register(&adev->dev, map, param);
}
```

المشكلة: `dev_get_regmap(adev->dev.parent, NULL)` بترجع `NULL`. ده معناه إن الـ **parent driver** (الـ `axg-audio-clkc`) مش عامل register للـ regmap صح أو إن الـ `auxiliary_device` اتسجل قبل ما الـ parent يخلص الـ init.

الـ `meson_sm1_audio_param` فيها:
```c
static const struct meson_reset_param meson_sm1_audio_param = {
    .reset_ops   = &meson_reset_toggle_ops,
    .reset_num   = 39,       /* 39 reset line */
    .level_offset = 0x28,    /* register offset */
};
```

لو الـ regmap مش موجود، مفيش طريقة نوصل لـ reset registers — وبالتالي الـ audio hardware فضل في حالة reset.

#### الحل

**خطوة 1 — تأكد إن الـ parent بيعمل regmap_init:**

```bash
# شوف الـ parent driver
ls /sys/bus/auxiliary/devices/axg-audio-clkc.rst-sm1.0/
cat /sys/bus/auxiliary/devices/axg-audio-clkc.rst-sm1.0/../uevent

# تأكد إن الـ regmap موجود على الـ parent
ls /sys/kernel/debug/regmap/
```

**خطوة 2 — لو الـ parent مش بيعمل regmap:**

ابحث في driver الـ `axg-audio-clkc` عن `regmap_init` أو `devm_regmap_init_mmio` — لازم يكون موجود قبل ما يعمل `auxiliary_device_add`.

**خطوة 3 — لو في race condition:**

ضيف `deferred probe` — الـ `-EINVAL` من `dev_get_regmap` ممكن يتحول لـ `-EPROBE_DEFER` في الـ parent driver لو الـ init order غلط:

```c
map = dev_get_regmap(adev->dev.parent, NULL);
if (!map)
    return -EPROBE_DEFER; /* اتأخر وجرب تاني */
```

#### الدرس المستفاد
**الـ** `dev_get_regmap` بترجع `NULL` بصمت لو الـ parent مش عامل register للـ regmap. المكان الصح للـ debug هو الـ parent driver مش الـ reset driver. دايمًا check إن الـ parent عمل `devm_regmap_init_*` قبل ما يعمل `auxiliary_device_add`.

---

### السيناريو 2: Bring-up بورد custom بـ Amlogic A1 — الـ VAD مش بيشتغل

#### العنوان
**Voice Activity Detection (VAD) مش بيشتغل على IoT sensor device بـ Amlogic A1**

#### السياق
فريق بيعمل **IoT smart speaker** بـ **Amlogic A1** chip. الجهاز محتاج **VAD** (Voice Activity Detection) عشان يصحى من الـ sleep على صوت. الـ A1 عنده audio clock controller منفصل عن الـ VAD audio.

#### المشكلة
الـ VAD microphone مش بيشتغل خالص. الـ `dmesg` فيه:

```
a1-audio-clkc rst-a1-vad: reset_control_reset failed: -6
```

**الـ -6 = ENXIO** — معناه الـ reset ID خارج الـ range.

#### التحليل
الـ driver عنده `param` لـ VAD منفصل:

```c
static const struct meson_reset_param meson_a1_audio_vad_param = {
    .reset_ops   = &meson_reset_toggle_ops,
    .reset_num   = 6,       /* 6 reset lines فقط */
    .level_offset = 0x8,
};
```

الـ `id_table` بيطابق:
```c
{
    .name = "a1-audio-clkc.rst-a1-vad",
    .driver_data = (kernel_ulong_t)&meson_a1_audio_vad_param,
},
```

المشكلة إن الكود الـ userspace أو الـ audio driver بيطلب **reset ID = 7** بس `reset_num = 6` يعني المسموح بيه من 0 لـ 5 فقط. الـ `meson_reset_controller_register` بيسجل controller بـ `nr_resets = param->reset_num` وأي طلب بـ ID >= 6 بيفشل.

```
reset_num = 6  →  valid IDs: 0, 1, 2, 3, 4, 5
requested ID = 7  →  ENXIO (out of range)
```

ده ممكن يحصل لو الـ DT أو الـ audio driver بيحاول يـ reset الـ VAD FIFO بـ index غلط.

#### الحل

**خطوة 1 — تأكد من الـ reset IDs المستخدمة:**

```bash
# شوف الـ reset consumer في الـ DT
grep -r "resets" /proc/device-tree/ --include="*.dts" 2>/dev/null
# أو
dtc -I fs /proc/device-tree | grep -A5 "vad"
```

**خطوة 2 — تأكد من الـ A1 datasheet:**

راجع الـ A1 TRM (Technical Reference Manual) — كم reset line عندها الـ VAD block؟ لو أكتر من 6، الـ `reset_num` في الكود غلط.

**خطوة 3 — لو الـ reset_num غلط:**

```c
/* لو الـ datasheet بيقول 8 resets مثلًا */
static const struct meson_reset_param meson_a1_audio_vad_param = {
    .reset_ops    = &meson_reset_toggle_ops,
    .reset_num    = 8,   /* تصحيح القيمة */
    .level_offset = 0x8,
};
```

**خطوة 4 — debug بسيط:**

```bash
# شوف الـ reset controllers المسجلة
cat /sys/kernel/debug/reset/reset_controllers
# أو
ls /sys/kernel/debug/reset/
```

#### الدرس المستفاد
**الـ** `reset_num` في `meson_reset_param` هو الـ guard الوحيد ضد out-of-bounds access على الـ reset registers. لازم تتأكد منه مع الـ datasheet قبل bring-up. الـ ENXIO هو الـ error الأوضح وبيوديك على طول للـ `reset_num`.

---

### السيناريو 3: Industrial Gateway بـ Amlogic G12A — الـ audio بيـ crash بعد suspend/resume

#### العنوان
**الـ audio بيـ crash بعد suspend/resume cycle على industrial gateway بـ Amlogic G12A**

#### السياق
شركة بتعمل **industrial IoT gateway** بـ **Amlogic G12A** — الجهاز بيتكلم مع sensors عبر الـ audio ADC interface. الـ gateway بيدخل الـ suspend بعد فترة idle وبعد الـ resume الـ audio subsystem بيـ hang أو بيطلع garbage data.

#### المشكلة
بعد أول `echo mem > /sys/power/state` والـ resume:

```
WARNING: CPU: 0 PID: 234 at drivers/reset/reset.c:175 reset_control_reset+0x...
axg-audio-clkc: reset controller used after device removed
```

أو الأبسط: الـ audio بيشتغل عشوائي ومش بيرجع clean state.

#### التحليل
الـ G12A بيستخدم:
```c
static const struct meson_reset_param meson_g12a_audio_param = {
    .reset_ops    = &meson_reset_toggle_ops,  /* toggle — pulse reset */
    .reset_num    = 26,
    .level_offset = 0x24,
};
```

الـ `meson_reset_toggle_ops` بتعمل **pulse reset** — بتكتب 1 في الـ bit وبترجعه 0. ده مش level-based reset.

المشكلة: عند الـ resume، الـ audio hardware محتاج يتعمل reset صريح. لكن:

1. الـ `meson_reset_aux_driver` مفيش فيه `remove` callback — يعني لو الـ parent driver اتـ unbind، الـ reset controller فضل معلق.
2. الـ auxiliary bus بيحل الـ `probe` عند الـ resume بس لو الـ parent ده نفسه اتـ re-probe.

```c
static struct auxiliary_driver meson_reset_aux_driver = {
    .probe    = meson_reset_aux_probe,
    /* مفيش .remove ولا .suspend ولا .resume */
    .id_table = meson_reset_aux_ids,
};
```

لو الـ `axg-audio-clkc` parent بيعمل `regmap` جديد عند الـ resume بس الـ reset controller لسه شايل الـ pointer للـ regmap القديم — بيحصل use-after-free أو stale pointer.

#### الحل

**خطوة 1 — تأكد إن الـ regmap بيتعمل re-init عند الـ resume:**

الـ `meson_reset_controller_register` بيحفظ الـ `map` pointer جوا الـ reset controller data. لو الـ parent عمل `devm_regmap_init_mmio` جديد عند الـ resume، المفروض نفس الـ pointer — بس لو re-allocated ده مشكلة.

```bash
# تتبع الـ regmap addresses
echo 'p:regmap_init devm_regmap_init_mmio' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable
echo mem > /sys/power/state
cat /sys/kernel/debug/tracing/trace
```

**خطوة 2 — تأكد إن الـ audio hardware بيتعمل reset صريح عند الـ resume:**

```bash
# بعد resume مباشرة، شوف state الـ reset lines
cat /sys/kernel/debug/reset/*/state 2>/dev/null
```

**خطوة 3 — الحل الصح:**

الـ parent audio clock driver لازم يعمل explicit reset عند الـ resume قبل ما يشغل الـ clocks:

```c
/* في الـ audio clock driver resume */
reset_control_reset(audio_rst);  /* pulse reset لكل الـ 26 line */
```

#### الدرس المستفاد
**الـ** `module_auxiliary_driver` بيعمل register/unregister تلقائي، بس مفيش lifecycle management للـ suspend/resume في الـ reset aux driver نفسه. المسؤولية على الـ parent audio driver إنه يعمل reset صريح للـ hardware عند كل resume cycle.

---

### السيناريو 4: الـ driver مش بيـ load على Amlogic S905D3 (SM1) — اسم الـ auxiliary device غلط

#### العنوان
**الـ** `reset-meson-aux` **مش بيـ load على S905D3 رغم إن الـ hardware SM1**

#### السياق
فريق bring-up بيشتغل على **Amlogic S905D3** للـ automotive entertainment system. الـ S905D3 هو SM1 family. الـ audio مش شغال والـ dmesg مش فيه أي رسالة من `reset-meson-aux`.

#### المشكلة
الـ driver مش بيـ probe خالص. الـ `lsmod` بيظهر `reset_meson_aux` محمل بس مفيش device بيـ bind بيه:

```bash
$ ls /sys/bus/auxiliary/drivers/reset_meson_aux/
# فاضة تمامًا
```

#### التحليل
الـ matching على الـ auxiliary bus بيعتمد على **اسم دقيق**:

```c
static const struct auxiliary_device_id meson_reset_aux_ids[] = {
    { .name = "a1-audio-clkc.rst-a1",      ... },
    { .name = "a1-audio-clkc.rst-a1-vad",  ... },
    { .name = "axg-audio-clkc.rst-g12a",   ... },
    { .name = "axg-audio-clkc.rst-sm1",    ... },  /* هذا هو SM1 */
    {}
};
```

الاسم `axg-audio-clkc.rst-sm1` لازم **يطابق بالظبط** الاسم اللي الـ parent audio clock driver بيعمل `auxiliary_device_add` بيه.

الـ S905D3 بيستخدم نفس `axg-audio-clkc` driver بس لو الـ driver register الـ device بـ اسم مختلف — مثلًا `axg-audio-clkc.rst-s905d3` — مفيش match.

```bash
# شوف الـ auxiliary devices المسجلة فعلًا
ls /sys/bus/auxiliary/devices/
# مثلًا:
axg-audio-clkc.rst-s905d3.0  <-- اسم مختلف!
```

الـ probe callback:
```c
static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
```
مش بيتكلم خالص لأن الـ bus ما لقاش match في الـ `id_table`.

#### الحل

**خطوة 1 — شوف الاسم الفعلي للـ auxiliary device:**

```bash
ls /sys/bus/auxiliary/devices/
# أو
udevadm info /sys/bus/auxiliary/devices/*/
```

**خطوة 2 — لو الاسم مختلف، حدد مين بيجيب الاسم ده:**

الاسم بييجي من الـ parent driver لما بيعمل:
```c
/* في axg-audio-clkc driver */
adev->name = "rst-sm1";  /* ده بيتجمع مع KBUILD_MODNAME */
```

**خطوة 3 — الحل:**

إما:
- صلح الـ `adev->name` في الـ audio clock driver عشان يطابق الـ id_table
- أو أضف الـ entry الجديد في الـ `meson_reset_aux_ids`:

```c
{
    .name = "axg-audio-clkc.rst-s905d3",
    .driver_data = (kernel_ulong_t)&meson_sm1_audio_param, /* نفس الـ params */
},
```

#### الدرس المستفاد
**الـ auxiliary bus matching** حساس جدًا للأسماء. أي chip جديد أو variant لازم يكون عنده entry صريح في `meson_reset_aux_ids` حتى لو الـ hardware parameters نفسها. `MODULE_DEVICE_TABLE(auxiliary, meson_reset_aux_ids)` بيساعد الـ udev يعمل autoload — بس لو الاسم غلط، مفيش autoload.

---

### السيناريو 5: Production Line — الـ audio بيشتغل على بعض الوحدات ومش بيشتغل على غيرها (Amlogic G12A)

#### العنوان
**الـ audio يشتغل على 70% من الوحدات في الـ production line — مشكلة timing في الـ init**

#### السياق
مصنع بينتج **Android TV boxes** بـ **Amlogic G12A**. في الـ production testing، 30% من الوحدات بتفشل في اختبار الـ audio. المشكلة مش consistent — نفس الـ firmware، نفس الـ hardware.

#### المشكلة
الـ dmesg على الوحدات الفاشلة:

```
meson-reset-aux axg-audio-clkc.rst-g12a.0: probe deferred
meson-reset-aux axg-audio-clkc.rst-g12a.0: probe deferred
...
(بعد كذا محاولة)
meson-reset-aux axg-audio-clkc.rst-g12a.0: probe failed with error -22
```

#### التحليل
الـ `meson_reset_aux_probe` بيعتمد على:

```c
map = dev_get_regmap(adev->dev.parent, NULL);
if (!map)
    return -EINVAL;
```

الـ `-EINVAL` مش بيـ trigger الـ deferred probe mechanism في Linux — الـ deferred probe بيشتغل بس مع `-EPROBE_DEFER`.

يعني:
1. لو الـ parent `axg-audio-clkc` مش خلص الـ `regmap` init لما الـ auxiliary device اتـ probe
2. الـ `dev_get_regmap` بترجع `NULL`
3. الـ driver بيرجع `-EINVAL` — **نهائي، مش deferred**
4. الـ device فضل unbound للأبد

الـ inconsistency في الـ production جاية من **boot time variation** — بعض الوحدات بيبوت أبطأ شوية (DRAM timing، voltage ramp) فالـ parent مش بيخلص قبل الـ child.

```
Timeline على وحدة شغالة:
t=0ms:  axg-audio-clkc probe starts
t=5ms:  regmap initialized
t=6ms:  auxiliary_device_add(rst-g12a)
t=7ms:  meson_reset_aux_probe → dev_get_regmap → OK ✓

Timeline على وحدة فاشلة:
t=0ms:  axg-audio-clkc probe starts
t=4ms:  auxiliary_device_add(rst-g12a)  ← قبل الـ regmap!
t=5ms:  meson_reset_aux_probe → dev_get_regmap → NULL → -EINVAL ✗
t=6ms:  regmap initialized (بس فات الأوان)
```

#### الحل

**خطوة 1 — في الـ parent audio clock driver:**

لازم `auxiliary_device_add` يتعمل **بعد** `devm_regmap_init_mmio`:

```c
/* في axg-audio-clkc probe */
map = devm_regmap_init_mmio(dev, base, &axg_audio_regmap_cfg);
if (IS_ERR(map))
    return PTR_ERR(map);

/* FIRST: register regmap */
/* THEN: add auxiliary devices */
ret = auxiliary_device_add(&clkc->rst_dev);
```

**خطوة 2 — fix مؤقت في الـ reset aux driver:**

```c
static int meson_reset_aux_probe(struct auxiliary_device *adev,
                                 const struct auxiliary_device_id *id)
{
    const struct meson_reset_param *param =
        (const struct meson_reset_param *)(id->driver_data);
    struct regmap *map;

    map = dev_get_regmap(adev->dev.parent, NULL);
    if (!map)
        return -EPROBE_DEFER; /* بدل -EINVAL: اسمح بـ retry */

    return meson_reset_controller_register(&adev->dev, map, param);
}
```

**خطوة 3 — تحقق من الـ fix في production:**

```bash
# على وحدة بعد الـ fix
dmesg | grep -E "meson-reset-aux|axg-audio"
# المفروض تشوف probe succeeded مرة واحدة بدون defer
```

**خطوة 4 — production test script:**

```bash
#!/bin/bash
# اختبر الـ reset controller
if [ -d /sys/bus/auxiliary/drivers/reset_meson_aux ]; then
    ls /sys/bus/auxiliary/drivers/reset_meson_aux/ | grep rst-g12a
    echo "PASS: reset controller bound"
else
    echo "FAIL: reset controller not found"
    exit 1
fi
```

#### الدرس المستفاد
**الـ** `-EINVAL` من `dev_get_regmap` يعامله الـ kernel كـ fatal error — مش بيعمل retry. في أي driver بيعتمد على resource من الـ parent، لازم تفكر: هل الـ parent ضامن إن الـ resource موجودة قبل ما يعمل `auxiliary_device_add`؟ لو مش ضامن، الـ `-EPROBE_DEFER` هو الـ safety net. الـ production failures الـ intermittent دايمًا بتوديك لـ race conditions في الـ init order.
## Phase 7: مصادر ومراجع

---

### توثيق الـ Kernel الرسمي

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **Reset controller API** | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | التوثيق الرسمي الكامل لـ reset controller framework — consumer API وdriver interface |
| **Reset controller RST source** | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | ملف RST الأصلي للتوثيق |
| **Driver implementer's API guide** | [docs.kernel.org/driver-api/index.html](https://docs.kernel.org/driver-api/index.html) | دليل شامل لكل driver subsystem بما فيه reset |
| **Auxiliary Bus documentation** | [docs.kernel.org/driver-api/auxiliary_bus.html](https://docs.kernel.org/driver-api/auxiliary_bus.html) | توثيق الـ auxiliary bus المستخدم في الدرايفر |

---

### مقالات LWN.net

الـ**LWN.net** هو المرجع التاريخي الأول لمتابعة تطور الـ kernel subsystems.

#### Reset Controller

- **[reset: Add generic GPIO reset driver](https://lwn.net/Articles/585145/)**
  مقالة بتناول إضافة دعم الـ GPIO للـ reset controller framework — نفس الـ framework اللي بيستخدمه `reset-meson-aux.c`.

- **[Kernel development (reset controller introduction)](https://lwn.net/Articles/549089/)**
  مرجع للفترة اللي اتقدم فيها الـ reset controller API الأول من Philipp Zabel.

#### Auxiliary Bus

- **[Managing multifunction devices with the auxiliary bus](https://lwn.net/Articles/840416/)**
  مقالة أساسية شارحة الـ auxiliary bus اللي اتضاف في kernel 5.11 — ده هو الـ bus اللي `reset-meson-aux.c` بيشتغل عليه. بتشرح ليه اتعمل وإزاي الـ parent driver بيسجّل الـ `auxiliary_device` والـ child driver بيعمل probe ليه.

- **[A fresh look at the kernel's device model](https://lwn.net/Articles/645810/)**
  شرح معمّق لنموذج الأجهزة في الـ kernel — مفيد لفهم إزاي الـ `struct device` وراثة الـ regmap من الـ parent.

#### Regmap

- **[regmap: Generic I2C and SPI register map library](https://lwn.net/Articles/451789/)**
  المقالة الأصلية اللي قدّمت الـ regmap API — الـ `reset-meson-aux.c` بيجيب الـ `regmap` من الـ parent عبر `dev_get_regmap()`.

- **[regmap: introduce fast_io busses, and use a spinlock for them](https://lwn.net/Articles/490704/)**
  تطوير الـ regmap لدعم الـ fast I/O مع spinlock — مهم لفهم الـ locking model في الـ regmap المستخدم مع الـ MMIO registers.

---

### نقاشات الـ Mailing List

#### الـ Patch Series الأصلي للدرايفر

- **[PATCH 0/8] reset: amlogic: move audio reset drivers out of CCF**
  [lore.kernel.org/lkml/20240710162526.2341399-1-jbrunet@baylibre.com/T/](https://lore.kernel.org/lkml/20240710162526.2341399-1-jbrunet@baylibre.com/T/)
  الـ patch series اللي أنشأ `reset-meson-aux.c` — Jerome Brunet فصل الـ reset logic من الـ audio clock controller وحطها في درايفر منفصل على الـ auxiliary bus. ده المرجع الأهم لفهم سبب وجود الدرايفر.

- **[PATCH] clk: amlogic: axg-audio: use the auxiliary reset driver**
  [lkml.org/lkml/2024/7/19/310](https://lkml.org/lkml/2024/7/19/310)
  الـ companion patch اللي خلّى الـ clock driver يستخدم الـ reset auxiliary driver الجديد.

#### Reset Controller Framework — النقاشات التأسيسية

- **[v4,2/8] reset: Add reset controller API — Philipp Zabel**
  [patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/)
  الـ patch الأصلي اللي أضاف الـ `reset_controller_dev` وبدأ بيه الـ framework كله.

- **[v4,2/2] reset: Add GPIO support to reset controller framework**
  [patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/)
  إضافة دعم GPIO للـ framework — بتوضح إزاي الـ framework اتمدّد.

- **[GIT PULL] Reset controller fixes for v6.2 — Philipp Zabel**
  [lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/)
  نموذج على صيانة الـ subsystem وإزاي الـ maintainer بيشتغل.

- **[PATCH 2/2] reset: meson: add meson audio arb driver — Jerome Brunet**
  [lore.kernel.org/all/20180706143122.7612-3-jbrunet@baylibre.com/](https://lore.kernel.org/all/20180706143122.7612-3-jbrunet@baylibre.com/)
  السلف المباشر لـ `reset-meson-aux.c` — أول reset driver خاص بالـ audio على Meson.

---

### ملفات الـ Kernel Source ذات الصلة

```
drivers/reset/
├── core.c                          # تطبيق الـ reset controller framework
├── reset-meson.c                   # الدرايفر الأساسي لـ Meson platform reset
├── amlogic/
│   ├── reset-meson.h               # الـ shared header (meson_reset_param, ops)
│   ├── reset-meson-aux.c           # الدرايفر موضوع الدراسة
│   └── reset-meson-level.c         # level-based reset implementation
include/linux/
├── reset-controller.h              # واجهة الـ reset controller provider
└── reset.h                         # واجهة الـ reset consumer
Documentation/driver-api/reset.rst  # التوثيق الرسمي
```

---

### كتب مرجعية

#### Linux Device Drivers (LDD3)

**Linux Device Drivers, 3rd Edition** — Corbet, Rubini, Kroah-Hartman
- الكتاب مجاني: [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- الـ Chapter 14 (The Linux Device Model): لفهم `struct device`، الـ bus model، وإزاي الـ parent/child relationship بتشتغل — وده أساسي لفهم الـ auxiliary bus.
- الـ Chapter 3 (Char Drivers) + Chapter 17 (Network Drivers): لأمثلة على الـ probe pattern المستخدم في كل driver.

#### Linux Kernel Development — Robert Love

**Linux Kernel Development, 3rd Edition** — Robert Love
- الفصل الخاص بالـ Device Model وـ Kobjects: بيشرح الـ `struct kobject`، الـ `struct device`، وبنية الـ sysfs.
- مفيد لفهم إزاي الـ `auxiliary_device` بيرث من `struct device`.

#### Embedded Linux Primer — Christopher Hallinan

**Embedded Linux Primer, 2nd Edition** — Christopher Hallinan
- الفصول الخاصة بالـ Board Support Package (BSP) وإزاي الـ SoC drivers بتتكاتف.
- مهم لفهم الـ context اللي بيشتغل فيه `reset-meson-aux.c` — SoC بيه audio subsystem بيحتاج reset مستقل.

---

### مواقع مرجعية إضافية

- **Linux Meson Project** — [linux-meson.com/mainlining.html](https://linux-meson.com/mainlining.html)
  بتتبع حالة الـ mainlining لكل Amlogic SoC feature — مفيد لمعرفة إيه اللي اتدعم وإيه اللي لسه.

- **Linux Kernel Driver DataBase — CONFIG_RESET_MESON** — [cateee.net/lkddb/web-lkddb/RESET_MESON.html](https://cateee.net/lkddb/web-lkddb/RESET_MESON.html)
  بيوضح الـ Kconfig dependencies والـ SoCs المدعومة.

- **Kernel lore search** — [lore.kernel.org](https://lore.kernel.org)
  للبحث في كل نقاشات الـ mailing list حول أي subsystem.

---

### Search Terms مفيدة

لو عايز تتعمق أكتر، ابحث بالـ terms دي:

```
# بحث عام في الـ reset framework
"reset_controller_dev" site:lore.kernel.org
"devm_reset_controller_register" linux kernel
"reset_control_ops" linux driver example

# خاص بـ Amlogic/Meson
"reset-meson-aux" linux kernel
"meson_reset_toggle_ops" site:lore.kernel.org
"jbrunet@baylibre.com" reset amlogic

# الـ auxiliary bus
"module_auxiliary_driver" linux kernel example
"auxiliary_device_id" driver linux
"dev_get_regmap" parent auxiliary bus

# الـ regmap مع الـ reset
"regmap_update_bits" reset controller driver
"dev_get_regmap" parent device driver
```
## Phase 8: Writing simple module

### الهدف

هنعمل module بيستخدم **kprobe** عشان يراقب دخول `meson_reset_aux_probe` — وهي الـ probe function اللي بتتشغل لما driver بيتطابق مع Amlogic Meson reset auxiliary device على الـ auxiliary bus. ده بيخلينا نشوف اسم الـ device واسم الـ driver اللي بيتعمله probe في الـ kernel log.

---

### الـ Module Code

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on meson_reset_aux_probe:
 * Logs device name and driver name each time an Amlogic
 * Meson reset auxiliary device is probed.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* kprobe struct, register/unregister_kprobe */
#include <linux/auxiliary_bus.h> /* struct auxiliary_device, struct auxiliary_device_id */
#include <linux/printk.h>      /* pr_info, pr_err */

/*
 * Signature of the target function:
 *   static int meson_reset_aux_probe(struct auxiliary_device *adev,
 *                                    const struct auxiliary_device_id *id);
 *
 * The kprobe pre-handler receives a pt_regs* and we extract arguments
 * from the ABI-defined registers (x86-64: rdi=arg1, rsi=arg2).
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs->di -> first arg  : struct auxiliary_device *adev
     * regs->si -> second arg : const struct auxiliary_device_id *id
     *
     * استخدمنا pt_regs عشان نوصل للـ arguments اللي اتبعتت للـ function
     * قبل ما تتنفذ، وده بيشتغل على x86-64 بناءً على System V ABI.
     */
    struct auxiliary_device        *adev =
            (struct auxiliary_device *)regs->di;
    const struct auxiliary_device_id *id =
            (const struct auxiliary_device_id *)regs->si;

    /*
     * dev_name بترجع الـ kobject name اللي بيظهر في sysfs.
     * id->name بيحتوي على الـ match string زي "axg-audio-clkc.rst-g12a".
     */
    pr_info("[meson_reset_aux_kprobe] probe called: dev=%s  id.name=%s\n",
            dev_name(&adev->dev),
            id->name);

    /* إرجاع 0 يعني "كمّل تنفيذ الـ function الأصلية" */
    return 0;
}

/* post_handler بيشتغل بعد ما الـ function تتنفذ — مش محتاجينه هنا */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* intentionally empty — نحن بس عايزين نشوف اللحظة اللي بيتعمل فيها probe */
}

/*
 * struct kprobe بيحدد على إيه هنضع الـ hook وإيه الـ callbacks.
 * الـ symbol_name هو اسم الـ function في الـ kernel symbol table.
 */
static struct kprobe kp = {
    .symbol_name = "meson_reset_aux_probe",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init meson_aux_probe_kp_init(void)
{
    int ret;

    /*
     * register_kprobe بيقول للـ kernel: "حط breakpoint على
     * meson_reset_aux_probe واستدعي handlers بتاعتنا".
     * لو الـ symbol مش موجود أو CONFIG_KPROBES مش enabled هيرجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[meson_reset_aux_kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[meson_reset_aux_kprobe] planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit meson_aux_probe_kp_exit(void)
{
    /*
     * unregister_kprobe لازم يتعمل في exit عشان نرجع الـ instruction
     * الأصلية ونمنع أي use-after-free لو الـ module اتـ unload
     * وفي probe لسه بتتشغل.
     */
    unregister_kprobe(&kp);
    pr_info("[meson_reset_aux_kprobe] removed from %s\n", kp.symbol_name);
}

module_init(meson_aux_probe_kp_init);
module_exit(meson_aux_probe_kp_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on meson_reset_aux_probe to log Amlogic reset device binding");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | يعرّف `struct kprobe` وفنكشنز الـ register/unregister |
| `linux/auxiliary_bus.h` | بيديك `struct auxiliary_device` و`struct auxiliary_device_id` عشان تقدر تفك الـ pointers |
| `linux/printk.h` | `pr_info` / `pr_err` للـ kernel log |

---

#### الـ `handler_pre` — ليه؟

**الـ pre_handler** بيتشغل قبل ما الـ function الأصلية تاخد الـ control، وده بالظبط اللي محتاجينه عشان نشوف الـ arguments اللي بتتبعت ليها.
بنسحب `adev` من `regs->di` و`id` من `regs->si` بناءً على **System V AMD64 ABI** اللي بيقول إن أول argument في `%rdi` والتاني في `%rsi`.

---

#### الـ `struct kprobe`

الحقل `symbol_name` بيخلي الـ kernel يحل الـ address وقت الـ `register_kprobe` بدل ما إحنا نحدده يدوياً، وده أأمن وأوضح.

---

#### الـ `module_init` / `module_exit`

- **init**: بيسجّل الـ kprobe — لو فشل بيرجع الـ error code فوراً.
- **exit**: لازم `unregister_kprobe` يتعمل **دايماً** في الـ exit، لأن لو الـ module اتـ unload من غيره، الـ callback pointer هيبقى dangling pointer وهيعمل kernel panic.

---

### Makefile

```makefile
obj-m += meson_aux_probe_kp.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# تحميل الـ module
sudo insmod meson_aux_probe_kp.ko

# اتحقق إن الـ kprobe اتسجّل
sudo dmesg | grep meson_reset_aux_kprobe

# لو عندك Amlogic board أو تقدر تعمل modprobe لـ reset-meson-aux:
sudo modprobe reset-meson-aux
# هتشوف في dmesg سطر زي:
# [meson_reset_aux_kprobe] probe called: dev=axg-audio-clkc.rst-g12a.0  id.name=axg-audio-clkc.rst-g12a

# إزالة الـ module
sudo rmmod meson_aux_probe_kp
```

---

### ملاحظة على الـ Architecture

الكود بيستخدم `regs->di` و`regs->si` اللي بيشتغل على **x86-64** بس. لو المنصة ARM64، الـ arguments بتيجي من `regs->regs[0]` و`regs->regs[1]`. عشان تعمل كود portable، ممكن تستبدل الـ kprobe بـ **tracepoint** أو **fprobe** اللي بيعطوك الـ arguments مباشرةً بدون اعتماد على الـ register layout.
