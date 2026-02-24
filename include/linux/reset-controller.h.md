## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: Reset Controller Framework

الـ file ده جزء من **Reset Controller Framework** اللي موجود في الـ kernel تحت مسؤولية Philipp Zabel من Pengutronix.

---

### ليه أصلاً في حاجة اسمها Reset؟

تخيل إنك عندك USB controller جوه SoC (شريحة متكاملة). الـ USB controller ده ممكن يتوقف أو يتعلق بسبب bug أو error. عشان تصحّحه، محتاج "تعمله reset" — يعني تقفل الكهرباء عنه لحظة وتفتحها تاني، زي ما بتعمل restart للكمبيوتر بالضغط على الزرار.

المشكلة إن في الـ SoC، مفيش زرار. بدل كده في **reset lines** — وهي خطوط كهربائية جوه الشريحة بتتحكم في كل hardware block لوحده. لما تعمل هذا الخط `assert` (يعني تشيله HIGH أو LOW حسب الـ design)، الـ block بيتعمل reset وبيوقف. لما تعمل `deassert`، بيرجع شغال من أول.

---

### القصة الكاملة

**المشكلة**: في SoC واحد ممكن يكون فيه 50 أو 100 hardware block مختلف (USB, UART, SPI, GPU, DSP, …)، وكل واحد فيهم عنده reset line خاصة بيه. مين المسؤول عن التحكم في الـ reset lines دي؟

في الغالب في hardware خاصة بتتحكم في كل الـ reset lines دي في مكان واحد — بنسميها **Reset Controller**. ممكن تكون جزء من الـ Clock Controller أو SysCon أو chip مستقل.

**الحل اللي الـ framework بيقدمه**: بدل ما كل driver يتكلم مع الـ reset hardware بشكل مباشر ويكون الكود متفرق في كل مكان، الـ framework بيعمل طبقة وسطى:

```
┌─────────────────────────────────────────────┐
│           Consumer Driver                   │
│  (USB driver, SPI driver, GPU driver, …)    │
│         يطلب reset بـ reset_control_reset() │
└──────────────────┬──────────────────────────┘
                   │  reset.h API
┌──────────────────▼──────────────────────────┐
│        Reset Controller Framework           │
│            drivers/reset/core.c             │
│   يدير القائمة ويوزع الطلبات               │
└──────────────────┬──────────────────────────┘
                   │  reset-controller.h API
┌──────────────────▼──────────────────────────┐
│        Reset Controller Driver              │
│  (مثلاً: reset-imx7.c, reset-ath79.c, …)   │
│   يعرف يتكلم مع الـ hardware الحقيقي       │
└─────────────────────────────────────────────┘
```

---

### دور الـ File: `reset-controller.h`

الـ file ده هو **واجهة الـ Provider** — يعني بيحدد الـ interface اللي أي reset controller driver محتاج يشتغل بيه عشان يسجّل نفسه في الـ framework.

فيه حاجتين أساسيتين:

#### 1. `struct reset_control_ops` — الـ Operations/Callbacks

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

ده الـ "عقد" بين الـ framework والـ driver. كل reset controller driver بيملي الـ struct ده بـ function pointers خاصة بيه:

| الـ Operation | المعنى |
|---|---|
| `reset` | reset سريع تلقائي (assert + wait + deassert) |
| `assert` | وقّف الـ hardware (حطّه في reset) |
| `deassert` | فك الـ reset (خلّيه يشتغل) |
| `status` | اسأل: هل هو في reset دلوقتي ولا لأ؟ |

#### 2. `struct reset_controller_dev` — الـ Reset Controller Device

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;  /* الـ callbacks */
    struct module *owner;                  /* مين صاحب الـ driver */
    struct list_head list;                 /* عشان يتضاف في global list */
    struct list_head reset_control_head;   /* قائمة الـ consumers بتوعه */
    struct device *dev;                    /* الـ device في نظام الـ drivers */
    struct device_node *of_node;           /* نود الـ device tree */
    const struct of_phandle_args *of_args; /* لو reset-gpios */
    int of_reset_n_cells;                  /* عدد cells في الـ specifier */
    int (*of_xlate)(...);                  /* ترجمة device tree ID → reset ID */
    unsigned int nr_resets;                /* كام reset line عنده */
};
```

الـ driver بيملي الـ struct ده ويسجّله في الـ framework بـ `reset_controller_register()`.

---

### قصة عملية: USB Controller على i.MX7

1. الـ device tree بيقول: "الـ USB controller ده reset line رقم 5 في الـ `reset-imx7` controller".
2. الـ `reset-imx7.c` driver بيملي `reset_controller_dev` بـ `ops` بتكتب في registers الـ SoC.
3. الـ USB driver بيطلب `reset_control_get()` — الـ framework بيبحث في قائمته ويلاقي الـ `reset-imx7` controller.
4. الـ USB driver بيعمل `reset_control_reset()` — الـ framework بينادي `ops->reset()` اللي بيكتب في الـ hardware register.
5. الـ USB controller اتعمله reset وبيشتغل صح.

---

### الفرق بين assert/deassert و reset

- **`assert`** + **`deassert`**: للـ hardware اللي محتاج تتحكم فيه يدوياً — زي الـ PHY اللي لازم يفضل في reset وانت بتهيّئ حاجات تانية.
- **`reset`** (self-deasserting): للـ hardware اللي بيعمل reset بنفسه سريع — بس تقوله "اعمل reset" وهو يكمّل.

---

### الـ `of_xlate` Function

الـ device tree بيحدد الـ reset line بـ specifier (رقم أو أكتر). الـ `of_xlate` بيترجم الـ specifier ده لـ ID داخلي يفهمه الـ driver. لو الـ controller simple، بيستخدم `of_reset_simple_xlate` الـ default اللي بترجع الرقم مباشرة.

---

### الـ `devm_` Variant

```c
int devm_reset_controller_register(struct device *dev,
                                   struct reset_controller_dev *rcdev);
```

الـ `devm_` prefix معناها "device-managed" — يعني لما الـ device يتشال (unbind/remove)، الـ framework بيعمل unregister تلقائياً بدون ما الـ driver يحتاج يعمل cleanup يدوي.

---

### الـ Files المهمة اللي لازم تعرفها

| الـ File | الدور |
|---|---|
| `include/linux/reset-controller.h` | **الـ file ده** — واجهة الـ provider (driver side) |
| `include/linux/reset.h` | واجهة الـ consumer (من بيطلب الـ reset) |
| `drivers/reset/core.c` | قلب الـ framework — بينفّذ كل الـ logic |
| `drivers/reset/reset-simple.c` | helper لـ controllers بسيطة بـ memory-mapped registers |
| `drivers/reset/reset-gpio.c` | reset controller بيستخدم GPIO pins |
| `drivers/reset/reset-imx7.c` | مثال على driver لـ i.MX7 SoC |
| `drivers/reset/reset-ath79.c` | مثال على driver لـ Atheros MIPS SoCs |
| `include/linux/reset/` | headers إضافية (مثل reset-simple.h) |
| `include/dt-bindings/reset/` | constants للـ device tree specifiers |
| `Documentation/driver-api/reset.rst` | التوثيق الرسمي للـ framework |
| `Documentation/devicetree/bindings/reset/` | Device tree bindings |

---

### ملخص

**الـ `reset-controller.h`** بيحدد الـ interface اللي أي hardware reset controller driver لازم يشتغل بيه — بيسجّل نفسه بـ struct فيه:
- الـ **ops** (إيه اللي يقدر يعمله: assert/deassert/reset/status)
- معلومات الـ **device tree** (عشان الـ consumers يلاقوه بالاسم أو الـ phandle)
- عدد الـ **reset lines** اللي بيوفّرها

من غير الـ file ده، مفيش طريقة موحدة تكتب reset controller driver — كل vendor كان هيعمل interface خاص بيه وكان التوحيد مستحيل.
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي SoC حديث (زي i.MX8، RK3399، أو أي Allwinner) فيه عشرات الـ IP blocks — USB controller، Ethernet MAC، GPU، DSP، إلخ. كل واحد فيهم بيحتاج **reset line** خاصة بيه: سلك واحد (أو أكتر) بيتحكم في إن الـ block يبقى في reset state أو يشتغل.

المشكلة القديمة كانت: كل driver بيعمل ده بطريقته الخاصة — بيكتب مباشرة على register معين في clock/reset block الخاص بالـ SoC. النتيجة:
- **تكرار كود** في كل driver لكل SoC.
- **تشابك** بين driver الـ consumer وتفاصيل الـ hardware للـ SoC.
- مستحيل تشغّل نفس driver الـ USB مثلاً على SoC مختلف من غير تعديل.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ kernel قسّم المسألة لطبقتين بوضوح:

| الطبقة | المسؤولية |
|--------|-----------|
| **Reset Controller Driver** | بيعرف SoC architecture، بيكتب على الـ registers الصح |
| **Reset Consumer Driver** | بيطلب reset بالاسم/الـ ID من غير ما يعرف أي hardware |

الـ **Reset Controller Framework** هو الوسيط — بيوفر API موحد للـ consumers، وبيسجّل الـ providers (controllers).

---

### تشبيه من الواقع — مع mapping كامل

تخيل **لوحة كهرباء مركزية** في مصنع كبير:

```
[الـ breaker panel]           ←→   reset_controller_dev
[كل circuit breaker]          ←→   reset line (واحد من nr_resets)
[رقم الـ breaker]             ←→   unsigned long id
[الكهربائي اللي بيشغّل/يوقف] ←→   reset_control_ops (assert/deassert/reset)
[الآلة اللي بتطلب كهرباء]    ←→   consumer driver (USB, Ethernet, ...)
[بطاقة الـ identification]    ←→   device tree phandle + of_xlate
[مفتاح واحد shared بين آلتين] ←→   RESET_CONTROL_SHARED
[مفتاح حصري لآلة وحدة]       ←→   RESET_CONTROL_EXCLUSIVE
```

الكهربائي (الـ ops) بيعرف الـ hardware — الآلة (consumer) بس بتقول "أنا عايزة أشتغل".

---

### الـ Big Picture Architecture

```
╔══════════════════════════════════════════════════════════╗
║                     Consumer Drivers                     ║
║  [USB Driver]  [Ethernet Driver]  [GPU Driver]  [...]    ║
╚══════════════════════╤═══════════════════════════════════╝
                       │  reset_control_assert(rstc)
                       │  reset_control_deassert(rstc)
                       │  reset_control_reset(rstc)
                       ▼
╔══════════════════════════════════════════════════════════╗
║              Reset Controller Framework                  ║
║  ┌─────────────────────────────────────────────────┐    ║
║  │  reset_control (per consumer handle)            │    ║
║  │   - flags (shared/exclusive/optional)           │    ║
║  │   - refcount                                    │    ║
║  │   - ptr → reset_controller_dev                 │    ║
║  └─────────────────────┬───────────────────────────┘    ║
║                        │                                 ║
║  ┌─────────────────────▼───────────────────────────┐    ║
║  │  reset_controller_dev                           │    ║
║  │   - ops → reset_control_ops                    │    ║
║  │   - of_node / of_args                          │    ║
║  │   - of_xlate()  (DT → ID translation)          │    ║
║  │   - nr_resets                                   │    ║
║  │   - list  (linked in global list)               │    ║
║  │   - reset_control_head (list of consumers)      │    ║
║  └─────────────────────┬───────────────────────────┘    ║
║                        │  calls ops->assert(rcdev, id)  ║
╚════════════════════════╪═════════════════════════════════╝
                         ▼
╔══════════════════════════════════════════════════════════╗
║              Reset Controller Drivers (Providers)        ║
║                                                          ║
║  [imx-src.c]  [rockchip-reset.c]  [sunxi-reset.c] [...] ║
║                                                          ║
║  كل واحد بيعمل:                                          ║
║    - register reset_controller_dev                      ║
║    - implement assert/deassert/reset/status             ║
║    - writel/readl على الـ SoC reset registers           ║
╚══════════════════════════════════════════════════════════╝
                         ▲
                         │
╔════════════════════════╧═════════════════════════════════╗
║              Device Tree                                 ║
║  consumer:                                               ║
║    resets = <&rst RST_USB0>;   /* phandle + specifier */ ║
║  provider:                                               ║
║    rst: reset-controller { ... #reset-cells = <1>; }    ║
╚══════════════════════════════════════════════════════════╝
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ framework بيبني على **فصل الهوية عن التنفيذ**:

1. الـ consumer driver بيحصل على `struct reset_control *` — مجرد handle مجرد.
2. الـ framework بيعمل lookup في الـ device tree عشان يعرف أي `reset_controller_dev` المسؤول.
3. الـ `of_xlate` بيترجم الـ DT specifier (زي `<&rst RST_USB0>`) لـ `unsigned long id` يفهمه الـ controller.
4. الـ `reset_control_ops` بينفّذ العمل الفعلي على الـ hardware.

---

### تفصيل الـ Structs وعلاقتها

```
reset_controller_dev
┌─────────────────────────────────────────────┐
│ ops ──────────────────────────────────────► reset_control_ops
│                                              ┌──────────────┐
│                                              │ reset(id)    │
│                                              │ assert(id)   │
│                                              │ deassert(id) │
│                                              │ status(id)   │
│                                              └──────────────┘
│ owner   → struct module*  (for refcounting)
│ dev     → struct device*  (للـ devm lifecycle)
│ of_node → struct device_node*  (الـ DT node)
│ of_args → struct of_phandle_args*  (لـ GPIO-based resets)
│ of_reset_n_cells → عدد الـ cells في الـ DT specifier
│ of_xlate(rcdev, reset_spec) → int id
│ nr_resets → عدد الـ reset lines الكلي
│
│ list ──────────────────────────────────────► global reset_controller_list
│                                              (linked list of all providers)
│
│ reset_control_head ────────────────────────► list of reset_control handles
│                                              (كل consumer أخد handle منه)
└─────────────────────────────────────────────┘
```

```
reset_control  (private struct في reset.c — مش مكشوفة للـ consumer)
┌─────────────────────────────────────────────┐
│ rcdev  ──► reset_controller_dev              │
│ id         (الـ line index جوا الـ controller)│
│ refcnt     (shared controls محتاجة refcount) │
│ acquired   (exclusive ownership flag)        │
│ flags      (shared / exclusive / optional)   │
│ list  ─────► reset_control_head في rcdev     │
└─────────────────────────────────────────────┘
```

---

### الـ of_xlate — ترجمة الـ Device Tree

الـ DT consumer بيحدد reset زي كده:

```dts
/* في الـ consumer node */
resets = <&cru SRST_USB2OTG0>;

/* في الـ provider node */
cru: clock-reset-controller@ff760000 {
    #reset-cells = <1>;   /* ← of_reset_n_cells = 1 */
};
```

الـ framework بياخد `<&cru SRST_USB2OTG0>` ويعمل `of_phandle_args`:
- `args[0] = SRST_USB2OTG0` (مثلاً = 135)

الدالة `of_xlate` بتترجمها لـ `id = 135` — وده اللي بيتبعت لـ `ops->assert(rcdev, 135)`.

الـ default implementation هي `of_reset_simple_xlate` اللي بترجع `args[0]` مباشرة. الـ drivers الأكتر تعقيداً ممكن تعمل custom `of_xlate`.

---

### Registration API

```c
/* provider بيسجّل نفسه */
int reset_controller_register(struct reset_controller_dev *rcdev);
void reset_controller_unregister(struct reset_controller_dev *rcdev);

/* نسخة مبنية على devm — بتعمل unregister أوتوماتيك لما الـ device يتشال */
int devm_reset_controller_register(struct device *dev,
                                   struct reset_controller_dev *rcdev);
```

الـ `devm_*` variant بتربط الـ lifecycle بالـ `struct device` — لما الـ platform device يتـ remove، الـ framework بيعمل `reset_controller_unregister` أوتوماتيك. ده بيعتمد على الـ **devres subsystem** (device resource management).

---

### مثال عملي — Rockchip Reset Controller

```c
/* drivers/reset/reset-rockchip.c (مبسّط) */

struct rockchip_reset {
    struct reset_controller_dev rcdev;  /* embedded struct */
    void __iomem *base;                 /* mapped register base */
};

static int rockchip_reset_assert(struct reset_controller_dev *rcdev,
                                  unsigned long id)
{
    struct rockchip_reset *rc = container_of(rcdev,
                                              struct rockchip_reset, rcdev);
    /* كل reset line ليها bank وbit position */
    int bank = id / 16;
    int offset = id % 16;

    /* CRU_SOFTRSTn registers: upper 16 bits = write mask */
    writel(BIT(offset) | (BIT(offset) << 16),
           rc->base + bank * 4);
    return 0;
}

static int rockchip_reset_deassert(struct reset_controller_dev *rcdev,
                                    unsigned long id)
{
    struct rockchip_reset *rc = container_of(rcdev,
                                              struct rockchip_reset, rcdev);
    int bank = id / 16;
    int offset = id % 16;

    /* clear the bit, mask still set */
    writel(BIT(offset) << 16,
           rc->base + bank * 4);
    return 0;
}

static const struct reset_control_ops rockchip_reset_ops = {
    .assert   = rockchip_reset_assert,
    .deassert = rockchip_reset_deassert,
    /* لا يوجد .reset لأن الـ hardware مش self-deasserting */
};

static int rockchip_reset_probe(struct platform_device *pdev)
{
    struct rockchip_reset *rc;
    struct resource *res;

    rc = devm_kzalloc(&pdev->dev, sizeof(*rc), GFP_KERNEL);

    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    rc->base = devm_ioremap_resource(&pdev->dev, res);

    rc->rcdev.ops        = &rockchip_reset_ops;
    rc->rcdev.owner      = THIS_MODULE;
    rc->rcdev.dev        = &pdev->dev;
    rc->rcdev.of_node    = pdev->dev.of_node;
    rc->rcdev.nr_resets  = 256;  /* عدد الـ reset lines في الـ SoC */

    /* التسجيل في الـ framework */
    return devm_reset_controller_register(&pdev->dev, &rc->rcdev);
}
```

---

### Shared vs. Exclusive Resets — مشكلة حقيقية

لما IP block اتنين بيشتركوا في نفس الـ reset line (زي USB2 و USB3 في بعض الـ SoCs):

```
         ┌──────────────────┐
         │   reset line X   │
         └────────┬─────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   [USB2 Driver]       [USB3 Driver]
```

لو USB2 عمل `assert` والـ USB3 لسه شغال — كارثة. الـ framework بيحل ده بالـ `RESET_CONTROL_SHARED`:
- بيحتفظ بـ **assert_count** لكل shared reset.
- الـ reset بيتعمل assert فعلاً بس لما **كل** الـ consumers طلبوا assert.
- الـ deassert بيحصل لما أول واحد يطلبه.

---

### الـ Framework بيملك إيه vs. بيفوّض إيه؟

| الـ Framework يملكه | الـ Driver يفوّض إليه |
|---------------------|----------------------|
| global list of controllers | `ops->assert/deassert/reset/status` |
| lookup من DT → rcdev | `of_xlate` (اختياري، فيه default) |
| إدارة الـ shared/exclusive semantics | الكتابة على الـ hardware registers |
| refcounting لـ reset_control handles | معرفة عدد الـ cells في الـ DT |
| devm lifecycle integration | أي hardware quirk |
| الـ bulk API للـ consumers | — |

---

### Subsystems ذات صلة

- **الـ Clock Framework (CCF)**: في معظم SoCs، الـ reset block موجود جوا نفس unit الخاص بالـ clock (CRU/CCU). الـ reset controller driver ممكن يتشارك نفس الـ register space مع الـ clock driver.
- **الـ Device Tree / OF subsystem**: الـ framework بيعتمد عليه بالكامل لتوصيل الـ consumer بالـ provider عبر الـ `phandle`.
- **الـ devres subsystem**: بيمكّن الـ `devm_reset_controller_register` تعمل cleanup أوتوماتيك.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Config Options & Enums — Cheatsheet

| الرمز | النوع | القيمة / المعنى |
|---|---|---|
| `CONFIG_RESET_CONTROLLER` | Kconfig | لو مفعّل، الـ API كلها موجودة؛ لو مش مفعّل، كل الدوال بتبقى inline stubs بترجع 0 |
| `shared` (في `reset_control`) | bool | 1 = أكتر من consumer ممكن يمسك نفس الـ reset line |
| `acquired` (في `reset_control`) | bool | 1 = في حد واخد exclusive ownership على الـ line دي |
| `array` (في `reset_control`) | bool | 1 = الـ object ده في الحقيقة `reset_control_array` |
| `of_reset_n_cells` | int | عدد الـ cells في الـ DT specifier (default = 1 لو مفيش `of_xlate` custom) |
| `nr_resets` | unsigned int | أقصى عدد reset lines يقدر الـ controller يوفّرها |
| `deassert_count` | atomic_t | counter للـ deassert calls على الـ shared reset |
| `triggered_count` | atomic_t | counter للـ reset calls (للـ shared فقط، قيمته 0 أو 1) |
| `GFP_KERNEL` | flag | يُستخدم في `devres_alloc` جوه `devm_reset_controller_register` |

---

### 1. الـ Structs المهمة

#### `struct reset_control_ops`

**الغرض:** جدول الـ function pointers اللي الـ driver بيملّيه — هو الـ "interface" بين الـ reset core framework والـ hardware driver.

| الـ Field | النوع | الوظيفة |
|---|---|---|
| `reset` | `int (*)(rcdev, id)` | self-deasserting reset: يعمل assert ثم deassert تلقائيًا |
| `assert` | `int (*)(rcdev, id)` | يحط الـ line في حالة reset (يضغطها) |
| `deassert` | `int (*)(rcdev, id)` | يفك الـ reset (يخلّي الجهاز يشتغل) |
| `status` | `int (*)(rcdev, id)` | يقرأ الحالة الحالية للـ line (asserted أو لا) |

- كل الـ ops اختيارية — الـ core بيتحقق من وجودها قبل ما يستدعيها.
- الـ `id` هو الرقم اللي طلع من `of_xlate` أو من الـ consumer لما طلب الـ reset line.

**ارتباطه بالباقي:** `reset_controller_dev` بيشاور عليه بـ `ops`.

---

#### `struct reset_controller_dev`

**الغرض:** يمثّل الـ reset controller نفسه — زي ما `struct clk_hw` بيمثّل الـ clock provider. واحد من دول بيتسجّل لكل IP block أو SoC subsystem عنده reset lines.

| الـ Field | النوع | الوظيفة |
|---|---|---|
| `ops` | `const struct reset_control_ops *` | الـ callbacks الخاصة بالـ hardware |
| `owner` | `struct module *` | يضمن إن الـ module ميتفرجش وفيه حد شايل reset |
| `list` | `struct list_head` | الـ node اللي بييجي في الـ global `reset_controller_list` |
| `reset_control_head` | `struct list_head` | رأس قايمة كل الـ `reset_control` objects اللي اتطلبت من الـ controller ده |
| `dev` | `struct device *` | الـ device المقابل في الـ driver model |
| `of_node` | `struct device_node *` | الـ DT node اللي الـ consumers بيشاوروا عليه كـ phandle |
| `of_args` | `const struct of_phandle_args *` | بديل لـ `of_node` في حالة reset-gpio controllers |
| `of_reset_n_cells` | int | عدد الـ args في الـ DT specifier |
| `of_xlate` | `int (*)(rcdev, reset_spec)` | بترجم DT args لـ id رقمي؛ default = `of_reset_simple_xlate` |
| `nr_resets` | unsigned int | سقف الـ IDs المقبولة |

**ملاحظة مهمة:** `of_node` و`of_args` مش ممكن يكونوا موجودين مع بعض — `reset_controller_register` بترفض ده وترجع `-EINVAL`.

---

#### `struct reset_control` (معرّف في `core.c`)

**الغرض:** يمثّل الـ handle اللي الـ consumer driver بياخده لما يطلب reset line معينة. مش جزء من الـ header العام لكنه محوري في الفهم.

| الـ Field | النوع | الوظيفة |
|---|---|---|
| `rcdev` | `struct reset_controller_dev *` | بيرجع للـ controller اللي منه الـ line دي |
| `list` | `struct list_head` | node في `rcdev->reset_control_head` |
| `id` | unsigned int | رقم الـ reset line جوّه الـ controller |
| `refcnt` | `struct kref` | عدد الـ get calls — لما يوصل صفر يتحذف |
| `acquired` | bool | exclusive ownership flag |
| `shared` | bool | هل ممكن أكتر من consumer يمسكه؟ |
| `array` | bool | لو true الـ object فعليًا `reset_control_array` |
| `deassert_count` | `atomic_t` | للـ shared: كام waiter عامل deassert |
| `triggered_count` | `atomic_t` | للـ shared: اتعمل reset فعلًا؟ |

---

#### `struct reset_control_array` (معرّف في `core.c`)

**الغرض:** يخلّي الـ consumer يتعامل مع مجموعة reset lines كـ unit واحدة.

| الـ Field | النوع | الوظيفة |
|---|---|---|
| `base` | `struct reset_control` | أول field — بيخلّي `container_of` يشتغل، وبيورّث الـ API كله |
| `num_rstcs` | unsigned int | عدد الـ lines في الـ array |
| `rstc[]` | `struct reset_control *[]` | الـ pointers للـ individual reset controls |

---

#### `struct reset_gpio_lookup` (معرّف في `core.c`)

**الغرض:** جدول lookup للـ reset lines اللي بتيجي عن طريق GPIO بدل ما تيجي من reset controller حقيقي.

| الـ Field | النوع | الوظيفة |
|---|---|---|
| `of_args` | `struct of_phandle_args` | الـ phandle مع الـ GPIO number |
| `swnode` | `struct fwnode_handle *` | software node بيشاور على الـ GPIO provider |
| `list` | `struct list_head` | node في `reset_gpio_lookup_list` |

---

### 2. مخططات العلاقات بين الـ Structs

```
                   reset_controller_list (global)
                   ┌──────────────────────────────────────┐
                   │  LIST_HEAD(reset_controller_list)    │
                   └──────────┬───────────────────────────┘
                              │
              ┌───────────────▼───────────────┐
              │    reset_controller_dev        │
              │  ┌──────────────────────────┐  │
              │  │ ops ──────────────────────┼──┼──► reset_control_ops
              │  │                          │  │      { reset, assert,
              │  │ owner ────────────────────┼──┼──► struct module      deassert, status }
              │  │                          │  │
              │  │ dev ──────────────────────┼──┼──► struct device
              │  │                          │  │
              │  │ of_node ──────────────────┼──┼──► struct device_node
              │  │                          │  │
              │  │ of_args ──────────────────┼──┼──► struct of_phandle_args
              │  │                          │  │
              │  │ list  ◄────────────────── ┼──┼── node in reset_controller_list
              │  │                          │  │
              │  │ reset_control_head ───────┼──┼──► ┐
              │  └──────────────────────────┘  │     │
              └───────────────────────────────┘     │
                                                    │
              ┌─────────────────────────────────────▼───┐
              │          reset_control                   │
              │  ┌─────────────────────────────────────┐ │
              │  │ rcdev ◄── back-pointer to rcdev      │ │
              │  │ list  ◄── node in reset_control_head │ │
              │  │ id        (reset line number)        │ │
              │  │ refcnt    (kref)                     │ │
              │  │ shared / acquired / array            │ │
              │  │ deassert_count / triggered_count     │ │
              │  └─────────────────────────────────────┘ │
              └─────────────────────────────────────────┘
                        │ (لو array == true)
                        ▼
              ┌─────────────────────────────────────────┐
              │        reset_control_array               │
              │  base  (embedded reset_control)          │
              │  num_rstcs                               │
              │  rstc[0] ──► reset_control               │
              │  rstc[1] ──► reset_control               │
              │  rstc[N] ──► reset_control               │
              └─────────────────────────────────────────┘
```

---

### 3. دورة حياة الـ Reset Controller

```
┌─────────────────────────────────────────────────────────────────┐
│                        DRIVER PROBE                             │
│                                                                 │
│  1. الـ driver يـ allocate ويـ fill reset_controller_dev:       │
│     rcdev->ops     = &my_reset_ops;                             │
│     rcdev->owner   = THIS_MODULE;                               │
│     rcdev->dev     = &pdev->dev;                                │
│     rcdev->of_node = pdev->dev.of_node;                         │
│     rcdev->nr_resets = MY_RESET_COUNT;                          │
│                                                                 │
│  2. يستدعي:                                                     │
│     devm_reset_controller_register(dev, rcdev)                  │
│           │                                                     │
│           ▼                                                     │
│     reset_controller_register(rcdev)                            │
│       - يتحقق: of_node XOR of_args                              │
│       - لو مفيش of_xlate → يحط of_reset_simple_xlate            │
│       - INIT_LIST_HEAD(&rcdev->reset_control_head)              │
│       - mutex_lock(&reset_list_mutex)                           │
│       - list_add(&rcdev->list, &reset_controller_list)          │
│       - mutex_unlock(&reset_list_mutex)                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     RUNTIME USAGE                               │
│                                                                 │
│  consumer driver:                                               │
│    rstc = devm_reset_control_get(dev, "my-reset")               │
│      → of_xlate يترجم DT args لـ id                             │
│      → ينشئ reset_control مرتبط بالـ rcdev                      │
│      → يضيفه لـ rcdev->reset_control_head                       │
│                                                                 │
│    reset_control_assert(rstc)                                   │
│      → rcdev->ops->assert(rcdev, rstc->id)                      │
│                                                                 │
│    reset_control_deassert(rstc)                                  │
│      → rcdev->ops->deassert(rcdev, rstc->id)                    │
│                                                                 │
│    reset_control_reset(rstc)   [self-deasserting]               │
│      → rcdev->ops->reset(rcdev, rstc->id)                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DRIVER REMOVE                             │
│                                                                 │
│  devm cleanup يستدعي تلقائيًا:                                   │
│    devm_reset_controller_release(dev, res)                      │
│      → reset_controller_unregister(rcdev)                       │
│           - mutex_lock(&reset_list_mutex)                       │
│           - list_del(&rcdev->list)                              │
│           - mutex_unlock(&reset_list_mutex)                     │
│                                                                 │
│  الـ rcdev بعد كده مش موجود في الـ global list                   │
└─────────────────────────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### 4.1 تسجيل الـ Controller (devm path)

```
driver probe
  └─► devm_reset_controller_register(dev, rcdev)
        ├─► devres_alloc(devm_reset_controller_release, ...)
        │     └── ينشئ devres entry مع pointer للـ release function
        │
        ├─► reset_controller_register(rcdev)
        │     ├── validates: !(of_node && of_args)
        │     ├── if !of_xlate:
        │     │     rcdev->of_reset_n_cells = 1
        │     │     rcdev->of_xlate = of_reset_simple_xlate
        │     ├── INIT_LIST_HEAD(&rcdev->reset_control_head)
        │     ├── mutex_lock(&reset_list_mutex)
        │     ├── list_add(&rcdev->list, &reset_controller_list)
        │     └── mutex_unlock(&reset_list_mutex)
        │
        ├─► *rcdevp = rcdev
        └─► devres_add(dev, rcdevp)
              └── يربط الـ release callback بالـ device lifecycle
```

#### 4.2 تنفيذ reset على الـ hardware

```
consumer driver
  └─► reset_control_reset(rstc)
        ├── if rstc->array:
        │     rstc_to_array(rstc) → reset_control_array_reset(array)
        │       └── for each rstc[i]: reset_control_reset(rstc[i])
        │
        ├── if rstc->shared:
        │     atomic_inc(&rstc->triggered_count)
        │     [logic ensures only first caller triggers hardware]
        │
        └── rcdev->ops->reset(rcdev, rstc->id)
              └── hardware: toggles reset register bit
```

#### 4.3 ترجمة الـ DT specifier (of_xlate)

```
consumer DT:
  resets = <&my_rcc 42>;

core يقرأ phandle args:
  reset_spec->args[0] = 42

  of_xlate(rcdev, reset_spec):
    [default: of_reset_simple_xlate]
      ├── if args[0] >= rcdev->nr_resets → return -EINVAL
      └── return args[0]   → id = 42

  reset_control->id = 42

  عند الاستخدام:
    rcdev->ops->assert(rcdev, 42)
      → driver يكتب في الـ register اللي بيتحكم في bit 42
```

---

### 5. استراتيجية الـ Locking

#### الـ Mutexes الموجودة في `core.c`

| الـ Mutex | يحمي إيه؟ | متى يتأخد؟ |
|---|---|---|
| `reset_list_mutex` | الـ global `reset_controller_list` | في `register` و`unregister` فقط |
| `reset_gpio_lookup_mutex` | الـ `reset_gpio_lookup_list` | عند إضافة أو بحث GPIO lookups |

#### ترتيب الـ Locking (Lock Ordering)

```
reset_list_mutex
  └── [مش بياخد أي lock تاني جوّاه]

reset_gpio_lookup_mutex
  └── [مستقل تمامًا عن reset_list_mutex]
```

لا يوجد nested locking بين الـ mutex اتنين — ده بيمنع الـ deadlock.

#### الـ Atomic Operations

- **`deassert_count`** و**`triggered_count`** في `reset_control` هما `atomic_t` — بيتعملوا `atomic_inc` / `atomic_dec` بدون mutex لأن العملية atomic في نفسها.
- الـ `kref` (refcnt) في `reset_control` بيستخدم `kref_get` / `kref_put` اللي هما atomic بطبيعتهم.

#### ملاحظة على الـ Shared Resets

الـ shared reset بيحتاج synchronization إضافية:

```
Thread A: reset_control_deassert(shared_rstc)
  → atomic_inc(&rstc->deassert_count)   [الآن = 1]
  → ops->deassert(rcdev, id)            [hardware يشتغل]

Thread B: reset_control_deassert(shared_rstc)
  → atomic_inc(&rstc->deassert_count)   [الآن = 2]
  → [مش بيكلّم hardware تاني لأن count > 1]

Thread A: reset_control_assert(shared_rstc)
  → atomic_dec(&rstc->deassert_count)   [الآن = 1]
  → [مش بيعمل assert لأن Thread B لسه محتاجه]

Thread B: reset_control_assert(shared_rstc)
  → atomic_dec(&rstc->deassert_count)   [الآن = 0]
  → ops->assert(rcdev, id)              [hardware يتوقف]
```

الـ reference counting ده بيضمن إن الـ hardware ميتعملوش assert غير لما كل الـ consumers يخلصوا.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Structs / Types

| الـ Type | الـ Purpose |
|---|---|
| `struct reset_control_ops` | vtable الـ callbacks اللي الـ reset controller driver بيimplement-ها |
| `struct reset_controller_dev` | الـ object الأساسي اللي بيمثل reset controller واحد في الـ system |
| `struct reset_control` | (في core.c) الـ handle اللي الـ consumer بياخده لـ reset line معين |
| `struct reset_control_array` | array من reset controls بتتعامل معاها كوحدة واحدة |

#### Registration API

| الـ Function | الـ Context | الـ Purpose |
|---|---|---|
| `reset_controller_register()` | driver probe | يسجل الـ rcdev في الـ global list |
| `reset_controller_unregister()` | driver remove | يشيل الـ rcdev من الـ global list |
| `devm_reset_controller_register()` | driver probe (managed) | نفس التسجيل بس automatically يعمل unregister على driver detach |

#### Ops Callbacks (بيimplementها الـ Driver)

| الـ Callback | متى يتحط | الـ Behavior |
|---|---|---|
| `ops->reset` | self-deasserting resets | assert + deassert in one shot |
| `ops->assert` | إذا الـ hardware بيدعم manual assert | يوقف الـ reset line في حالة assert |
| `ops->deassert` | إذا الـ hardware بيدعم manual deassert | يطلق الـ reset line |
| `ops->status` | اختياري | يرجع الـ state الحالي للـ reset line |

#### Internal Helpers (في core.c)

| الـ Function | الـ Purpose |
|---|---|
| `of_reset_simple_xlate()` | default translation من DT specifier لـ reset ID |
| `rcdev_name()` | يجيب اسم الـ rcdev للـ logging |
| `devm_reset_controller_release()` | devres release callback |

---

### Category 1: الـ Ops Callbacks — `struct reset_control_ops`

الـ `reset_control_ops` هو الـ vtable اللي الـ driver بيملاه. الـ reset controller framework بيشوف الـ hardware من خلال الـ 4 function pointers دول. مش كل الـ callbacks إجبارية — الـ driver لازم يimplementها بناءً على capabilities الـ hardware.

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

---

#### `ops->reset`

```c
int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
```

**بيعمل إيه:** بيعمل pulse كامل على الـ reset line — assert وبعدين deassert تلقائياً. ده الـ callback الخاص بـ **self-deasserting resets** اللي الـ hardware فيها بيعمل deassert لوحده أو اللي محتاجين pulse مش level.

**Parameters:**
- `rcdev` — الـ reset controller device object اللي بيملكه الـ driver
- `id` — الـ reset line index (بيجي من `of_xlate` بعد ما اتترجم من الـ DT)

**Return:** `0` نجح، قيمة سالبة `errno` في حالة خطأ.

**Key Details:**
- لو الـ `ops->assert` و `ops->deassert` موجودين بس `ops->reset` مش موجود، الـ framework في `reset_control_reset()` بيعمل assert ثم deassert يدوياً كـ fallback.
- مش محتاج locking في مستوى الـ ops — الـ framework بيتعامل مع الـ shared/exclusive semantics قبل ما يوصل للـ callback.
- الـ caller context: `reset_control_reset()` في `drivers/reset/core.c`.

---

#### `ops->assert`

```c
int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
```

**بيعمل إيه:** بيحط الـ reset line في حالة **assert** (أي الـ device في state الـ reset). الـ device مش هيتحرك لحد ما يتعمله deassert.

**Parameters:**
- `rcdev` — الـ reset controller device
- `id` — رقم الـ reset line

**Return:** `0` نجح، error code سالب.

**Key Details:**
- الـ framework بيكون بيtrack الـ `deassert_count` عبر `atomic_t` في `struct reset_control` قبل ما يوصل للـ callback ده. ما بيستدعيش `ops->assert` غير لما الـ count يوصل لـ zero.
- الـ exclusive reset control: الـ ops بتتنادى مباشرة. الـ shared reset: الـ framework بيدير الـ reference counting.
- الـ caller: `reset_control_assert()`.

---

#### `ops->deassert`

```c
int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
```

**بيعمل إيه:** بيطلق الـ reset line — الـ device يرجع يشتغل. ده بيتنادى بعد ما الـ `deassert_count` يبقى أكبر من zero لأول مرة (في الـ shared resets).

**Parameters:**
- `rcdev` — الـ reset controller device
- `id` — رقم الـ reset line

**Return:** `0` نجح، error code سالب.

**Key Details:**
- لازم الـ driver يضمن إن الـ deassert operation آمنة ومش هتأثر على devices تانية شايرة نفس الـ bus/clock domain.
- الـ caller: `reset_control_deassert()`.

---

#### `ops->status`

```c
int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
```

**بيعمل إيه:** بيرجع الـ state الحالي للـ reset line — هل في حالة assert ولا deassert. اختياري ومش كل الـ hardware بيدعمه.

**Parameters:**
- `rcdev` — الـ reset controller device
- `id` — رقم الـ reset line

**Return:** `1` لو في reset (asserted)، `0` لو مش في reset، قيمة سالبة لو خطأ.

**Key Details:**
- الـ caller: `reset_control_status()`.
- لو الـ op مش موجودة (NULL)، الـ framework بيرجع `-ENOTSUPP`.

---

### Category 2: الـ Registration Functions

الـ group ده هو الـ public API اللي الـ reset controller driver بيستخدمه في الـ `probe()` و `remove()`. الهدف: يضيف أو يشيل الـ `reset_controller_dev` من الـ global list اللي بتستخدمها الـ consumer drivers للـ lookup.

الـ global state في الـ framework:
```c
static DEFINE_MUTEX(reset_list_mutex);        /* يحمي reset_controller_list */
static LIST_HEAD(reset_controller_list);      /* list كل الـ registered rcdevs */
```

---

#### `reset_controller_register`

```c
int reset_controller_register(struct reset_controller_dev *rcdev);
```

**بيعمل إيه:** بيسجل الـ `rcdev` في الـ kernel-wide reset controller list حتى الـ consumer drivers يقدروا يلاقوه ويستخدموا الـ reset lines بتاعته. بيعمل default setup لو الـ `of_xlate` مش موجود.

**Parameters:**
- `rcdev` — pointer لـ `reset_controller_dev` موجود ومهيأ من الـ driver. لازم الـ `ops` و `nr_resets` يكونوا set قبل الاستدعاء.

**Return:** `0` نجح، `-EINVAL` لو الـ `of_node` و `of_args` الاتنين معاً موجودين (التناقض ده غلط).

**Key Details:**

```
reset_controller_register():
  if (rcdev->of_node && rcdev->of_args) → return -EINVAL  // conflict
  if (!rcdev->of_xlate):
      rcdev->of_reset_n_cells = 1
      rcdev->of_xlate = of_reset_simple_xlate  // default 1:1 mapping
  INIT_LIST_HEAD(&rcdev->reset_control_head)   // init consumer list
  mutex_lock(&reset_list_mutex)
  list_add(&rcdev->list, &reset_controller_list)
  mutex_unlock(&reset_list_mutex)
  return 0
```

- بيحمي الإضافة بـ `reset_list_mutex` — الـ locking مطلوب لأن أي consumer driver ممكن يكون بيعمل lookup في نفس الوقت.
- بيعمل `INIT_LIST_HEAD` على `reset_control_head` اللي هو الـ list الداخلية للـ handles اللي الـ consumers أخدوها.
- الـ `of_reset_n_cells = 1` معناها DT specifier بيتكون من cell واحد بس (الـ ID).
- الـ caller: driver `probe()` function.
- مصدر: `EXPORT_SYMBOL_GPL` — متاح للـ modules.

---

#### `reset_controller_unregister`

```c
void reset_controller_unregister(struct reset_controller_dev *rcdev);
```

**بيعمل إيه:** بيشيل الـ `rcdev` من الـ global list. بعد ما يتشال، الـ consumers الجديدة مش هتلاقيه، بس الـ handles الموجودة قبل كده لسه valid لحد ما يتعمل لها `reset_control_put()`.

**Parameters:**
- `rcdev` — pointer للـ controller اللي هيتشال. لازم يكون نفس الـ pointer اللي اتسجل قبل كده.

**Return:** void.

**Key Details:**
- بيعمل `list_del` تحت `reset_list_mutex`.
- **لا يحرر الـ memory** — ده مسؤولية الـ driver نفسه بعد الاستدعاء.
- لو في consumers لسه شايلة handles وبيحاولوا يعملوا reset operations بعد الـ unregister، ممكن يحصل use-after-free إلا لو الـ driver ضمن انه cleanup الـ consumers الأول.
- الـ caller: driver `remove()` أو `devm_reset_controller_release()`.
- مصدر: `EXPORT_SYMBOL_GPL`.

---

#### `devm_reset_controller_register`

```c
int devm_reset_controller_register(struct device *dev,
                                   struct reset_controller_dev *rcdev);
```

**بيعمل إيه:** نسخة managed من `reset_controller_register()`. بيستخدم `devres` framework عشان automatically يستدعي `reset_controller_unregister()` لما الـ device يتdetach أو الـ driver يتremove. بيخلي الـ error handling في الـ probe أبسط بكتير.

**Parameters:**
- `dev` — الـ `struct device*` الخاص بالـ driver (بيربط الـ cleanup بيه).
- `rcdev` — الـ reset controller object المهيأ.

**Return:** `0` نجح، `-ENOMEM` لو فشل الـ `devres_alloc`، أو error من `reset_controller_register()`.

**Key Details:**

```
devm_reset_controller_register():
  rcdevp = devres_alloc(devm_reset_controller_release, sizeof(*rcdevp), GFP_KERNEL)
  if (!rcdevp) → return -ENOMEM
  ret = reset_controller_register(rcdev)
  if (ret):
      devres_free(rcdevp)
      return ret
  *rcdevp = rcdev
  devres_add(dev, rcdevp)
  return 0
```

- بيخزن pointer للـ `rcdev` في الـ devres resource نفسه عشان الـ release callback (`devm_reset_controller_release`) يعرف إيه اللي يشيله.
- الـ `devm_reset_controller_release` ببساطة بتنادي `reset_controller_unregister(*(struct reset_controller_dev **)res)`.
- الـ caller: driver `probe()` — الأفضل استخدامه بدل `reset_controller_register()` في الـ modern drivers.
- مصدر: `EXPORT_SYMBOL_GPL`.

---

### Category 3: الـ Default Translation Helper

#### `of_reset_simple_xlate` (static)

```c
static int of_reset_simple_xlate(struct reset_controller_dev *rcdev,
                                 const struct of_phandle_args *reset_spec);
```

**بيعمل إيه:** بيترجم الـ DT phandle specifier لـ reset ID عددي. ده الـ default لو الـ driver ما حددش `of_xlate` خاص بيه. بيفترض mapping مباشر 1:1 بين الـ DT argument والـ reset line index.

**Parameters:**
- `rcdev` — الـ controller (بستخدمه يتحقق من الـ bounds).
- `reset_spec` — الـ parsed phandle args من الـ DT. الـ `reset_spec->args[0]` هو الـ reset index.

**Return:** الـ ID العددي للـ reset line (>= 0) لو نجح، `-EINVAL` لو الـ index أكبر من أو يساوي `nr_resets`.

**Key Details:**
- بسيطة جداً: `return reset_spec->args[0]` بعد bounds check.
- الـ driver اللي عنده custom specifier (زي GPIO number + flags) لازم يحط `of_xlate` خاص بيه و`of_reset_n_cells` أكبر من 1.
- بتتحط automatically في `reset_controller_register()` لو `of_xlate` كانت NULL.
- الـ caller: كود الـ lookup الداخلي في `core.c` لما بيعمل parse لـ DT.

---

#### `rcdev_name` (static)

```c
static const char *rcdev_name(struct reset_controller_dev *rcdev);
```

**بيعمل إيه:** بيرجع اسم مناسب للـ `rcdev` للاستخدام في الـ kernel log messages. بيجرب 3 مصادر بالترتيب: الـ device name، الـ OF node name، الـ of_args node name.

**Parameters:**
- `rcdev` — الـ controller object.

**Return:** string ثابت أو NULL لو ماعنداش أي اسم.

**Key Details:**
- Helper داخلي فقط، مش exported.
- الـ priority: `dev` > `of_node` > `of_args->np`.

---

### Category 4: الـ `struct reset_controller_dev` — شرح الـ Fields

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;     /* vtable الـ hardware operations */
    struct module *owner;                    /* الـ kernel module صاحب الـ driver */
    struct list_head list;                   /* entry في reset_controller_list */
    struct list_head reset_control_head;     /* list الـ handles اللي الـ consumers أخدوها */
    struct device *dev;                      /* الـ device model object */
    struct device_node *of_node;             /* DT node (phandle target) */
    const struct of_phandle_args *of_args;   /* للـ GPIO-based reset controllers */
    int of_reset_n_cells;                    /* عدد الـ cells في الـ DT specifier */
    int (*of_xlate)(...);                    /* ترجمة DT specifier → reset ID */
    unsigned int nr_resets;                  /* عدد الـ reset lines المتاحة */
};
```

| الـ Field | الإجبارية | الشرح |
|---|---|---|
| `ops` | إجباري | لازم يكون set قبل التسجيل |
| `owner` | مستحسن | `THIS_MODULE` عشان الـ refcount صح |
| `list` | framework | بيتملاه `reset_controller_register()` |
| `reset_control_head` | framework | بيتملاه `reset_controller_register()` |
| `dev` | مستحسن | لو موجود بيُستخدم في الـ logging والـ devm |
| `of_node` | أو `of_args` | لازم واحد منهم موجود للـ DT lookup |
| `of_args` | أو `of_node` | خاص بالـ GPIO-based controllers |
| `of_reset_n_cells` | اختياري | الـ default 1 لو `of_xlate` مش موجود |
| `of_xlate` | اختياري | الـ default هو `of_reset_simple_xlate` |
| `nr_resets` | إجباري | لازم يعكس العدد الفعلي للـ hardware lines |

**ملاحظة مهمة على `of_node` vs `of_args`:**
- `of_node`: الأغلب — الـ reset controller عنده DT node خاص بيه كـ phandle target.
- `of_args`: الـ GPIO-based reset controllers اللي بتكون embedded جوه DT property لـ device تاني (مش controller node مستقل). لو الاتنين موجودين مع بعض، `reset_controller_register()` بترفض بـ `-EINVAL`.

---

### الـ Data Flow — كيف يتكامل كل حاجة

```
Driver Probe:
  1. يملا struct reset_controller_dev:
       .ops       = &my_reset_ops
       .owner     = THIS_MODULE
       .dev       = &pdev->dev
       .of_node   = dev->of_node
       .nr_resets = MY_RESET_COUNT
  2. يستدعي devm_reset_controller_register(dev, rcdev)
       └─ يعمل devres_alloc
       └─ يستدعي reset_controller_register(rcdev)
              └─ of_xlate = of_reset_simple_xlate  (if not set)
              └─ INIT_LIST_HEAD(&rcdev->reset_control_head)
              └─ list_add(&rcdev->list, &reset_controller_list)

Consumer Driver:
  reset_control_get(dev, "name")
    └─ يعمل DT lookup → يلاقي الـ rcdev
    └─ يستدعي rcdev->of_xlate() → يجيب الـ ID
    └─ يعمل alloc struct reset_control
    └─ يضيفه في rcdev->reset_control_head

  reset_control_reset(rstc)
    └─ يستدعي rcdev->ops->reset(rcdev, rstc->id)

Driver Remove (devres path):
  devm_reset_controller_release()
    └─ reset_controller_unregister(rcdev)
         └─ list_del(&rcdev->list)
```

---

### الـ `CONFIG_RESET_CONTROLLER` Guard

الـ header بيحط الـ 3 registration functions جوه:
```c
#if IS_ENABLED(CONFIG_RESET_CONTROLLER)
    /* real implementations */
#else
    /* stub inlines ترجع 0 */
#endif
```

الـ stubs بترجع `0` (success) عشان الـ consumer code يشتغل على platforms مش عندها reset controller subsystem من غير ما تحتاج `#ifdef` في كل مكان. ده الـ pattern القياسي في kernel optional subsystems.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs Entries

الـ **reset controller** بيعرض معلوماته جوه `/sys/kernel/debug/` لو kernel متبني بـ `CONFIG_DEBUG_FS=y`.

```bash
# اعرض كل الـ reset controllers المسجلين
ls /sys/kernel/debug/reset/

# مثال على مسار controller معين
cat /sys/kernel/debug/reset/<reset-name>/

# لو الـ driver بيستخدم reset-gpio، دور على entries الـ gpio
ls /sys/kernel/debug/gpio
cat /sys/kernel/debug/gpio
```

بعض الـ SoC vendors (مثلاً Rockchip، NXP) بيضيفوا custom debugfs entries:

```bash
# Rockchip CRU reset debug
ls /sys/kernel/debug/clk/
# لمّا الـ reset مرتبط بالـ clock controller

# عرض معلومات الـ device manager (devm resources)
cat /sys/kernel/debug/devices_deferred
```

| Entry | المعنى |
|-------|---------|
| `/sys/kernel/debug/gpio` | حالة GPIO-based resets (asserted/deasserted) |
| `/sys/kernel/debug/clk/` | عالـ SoCs اللي الـ reset بيتحكم فيه الـ clock controller |
| `/sys/kernel/debug/reset/` | لو vendor driver بيسجّل entries خاصة |

---

#### 2. sysfs Entries

```bash
# ابحث عن device بيستخدم reset
ls /sys/bus/platform/devices/

# اعرض resets المرتبطة بـ device معين
cat /sys/devices/platform/<device-name>/of_node/resets

# اعرض معلومات الـ driver
cat /sys/bus/platform/devices/<device>/driver/module/refcnt

# عدد الـ reset controls المطلوبة من controller معين
# (مش standard entry — بتختلف حسب الـ vendor)
ls /sys/devices/platform/<reset-controller-dev>/
```

---

#### 3. ftrace — Tracepoints & Events

الـ reset controller subsystem عنده tracepoints جوه `drivers/reset/`:

```bash
# فعّل الـ function tracer على دوال الـ reset subsystem
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تابع كل دوال reset_controller
echo 'reset_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer

# أو استخدم function_graph لرؤية call stack
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'reset_controller_register' > /sys/kernel/debug/tracing/set_graph_function

# شغّل الـ tracing وتفرج على النتيجة
cat /sys/kernel/debug/tracing/trace

# بعد خلاص
echo nop > /sys/kernel/debug/tracing/current_tracer
echo > /sys/kernel/debug/tracing/set_ftrace_filter
```

الدوال الأهم للـ trace:

```
reset_controller_register
reset_controller_unregister
devm_reset_controller_register
reset_control_reset
reset_control_assert
reset_control_deassert
reset_control_status
of_reset_simple_xlate
```

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لكل ملفات الـ reset subsystem
echo 'file drivers/reset/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ module معين
echo 'module reset-simple +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه الـ debug messages المفعّلة
grep 'reset' /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel log
dmesg -w | grep -i reset

# أو بمستوى أعلى
dmesg --level=debug | grep reset
```

لو عايز تضيف printk مؤقت في driver تحت الـ debugging:

```c
/* في دالة assert مثلاً */
int my_reset_assert(struct reset_controller_dev *rcdev, unsigned long id)
{
    dev_dbg(rcdev->dev, "asserting reset id=%lu\n", id);
    /* ... */
}
```

---

#### 5. Kernel Config Options للـ Debugging

| CONFIG | الوصف |
|--------|--------|
| `CONFIG_RESET_CONTROLLER=y` | الـ subsystem الأساسي — لازم يتفعّل |
| `CONFIG_DEBUG_FS=y` | يفتح `/sys/kernel/debug/` |
| `CONFIG_DYNAMIC_DEBUG=y` | يفعّل `dev_dbg()` runtime |
| `CONFIG_DEBUG_LIST=y` | يتحقق من corruption في الـ `list_head` |
| `CONFIG_LIST_HARDENED=y` | حماية إضافية للـ linked lists |
| `CONFIG_PROVE_LOCKING=y` | يكتشف deadlocks في الـ mutex/spinlock |
| `CONFIG_DEBUG_DEVRES=y` | تتبع الـ `devm_*` resources |
| `CONFIG_OF_DYNAMIC=y` | دعم Device Tree dynamic |
| `CONFIG_GPIOLIB_IRQCHIP=y` | لو reset مبني على GPIO |
| `CONFIG_DEBUG_GPIO=y` | debug للـ GPIO-based resets |
| `CONFIG_KALLSYMS=y` | يظهر function names في stack traces |
| `CONFIG_KASAN=y` | كتشاف memory bugs في الـ driver |

```bash
# تحقق من الـ config الحالي
grep -E 'CONFIG_RESET|CONFIG_DEBUG_LIST|CONFIG_DEBUG_DEVRES' /boot/config-$(uname -r)
```

---

#### 6. Subsystem-Specific Tools

```bash
# اعرض كل الـ platform devices (reset controllers عادةً platform devices)
ls /sys/bus/platform/devices/ | grep -i reset

# اعرض الـ OF node الخاص بـ reset controller
ls /sys/devices/platform/<rcdev-name>/of_node/

# اعرض الـ resets property في الـ device tree من sysfs
cat /sys/devices/platform/<consumer-device>/of_node/resets

# lsmod لتحقق من تحميل الـ module
lsmod | grep reset

# modinfo لمعرفة معلومات الـ module
modinfo reset-simple

# تحقق من binding صحيح عبر of_platform
cat /sys/bus/platform/devices/<device>/driver_override
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `reset controller not found` | الـ `reset-controller` phandle في DT غلط أو الـ driver مش loaded | تحقق من DT وتأكد الـ module متحمّل |
| `reset: -EPROBE_DEFER` | الـ reset controller لسه ما اتسجّلش وقت probe الـ consumer | طبيعي — الـ kernel هيعيد الـ probe تاني |
| `reset: -ENOENT` | الـ reset line مش موجود في DT | راجع `resets` property في DTS |
| `reset: -EBUSY` | حاول تاخد exclusive reset وهو مشغول | شيل الـ `RESET_CONTROL_EXCLUSIVE` أو استنى |
| `reset: -ENOTSUPP` | الـ `CONFIG_RESET_CONTROLLER=n` | فعّل الـ config |
| `reset: failed to get reset` | فشل الـ `__reset_control_get()` | راجع الـ DT phandle وتأكد الـ rcdev متسجّل |
| `devm_reset_controller_register failed` | مشكلة في تسجيل الـ controller | راجع الـ ops وتأكد الـ dev valid |
| `list_add corruption` | الـ `list_head` في `reset_control_head` اتخرب | فعّل `CONFIG_DEBUG_LIST` |
| `of_reset_simple_xlate: invalid specifier` | عدد cells في DT مش متطابق مع `of_reset_n_cells` | طابق `#reset-cells` في DTS مع الـ driver |
| `WARNING: refcount_t: addition on 0` | عملت reset بعد ما الـ controller اتفك | راجع lifecycle الـ `reset_controller_unregister` |

---

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

```c
int reset_controller_register(struct reset_controller_dev *rcdev)
{
    /* تحقق إن الـ ops مش NULL قبل التسجيل */
    WARN_ON(!rcdev->ops);
    WARN_ON(!rcdev->dev);
    WARN_ON(rcdev->nr_resets == 0);
    /* ... */
}

int my_reset_assert(struct reset_controller_dev *rcdev, unsigned long id)
{
    /* تحقق إن الـ id في النطاق */
    if (WARN_ON(id >= rcdev->nr_resets)) {
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}

/* في of_xlate — تحقق من صحة الـ specifier */
int my_of_xlate(struct reset_controller_dev *rcdev,
                const struct of_phandle_args *reset_spec)
{
    if (WARN_ON(reset_spec->args_count != rcdev->of_reset_n_cells))
        return -EINVAL;
    /* ... */
}
```

النقاط الاستراتيجية:

| النقطة | السبب |
|--------|--------|
| بداية `reset_controller_register()` | تحقق إن الـ `ops` و `dev` و `nr_resets` valid |
| بداية `of_xlate()` | تحقق إن عدد الـ args صح |
| بداية `assert/deassert/reset` ops | تحقق إن الـ id في النطاق |
| عند `reset_controller_unregister()` | تحقق إن مفيش consumers لسه مشغلين |

---

### Hardware Level

---

#### 1. التحقق إن حالة الـ Hardware تطابق الـ Kernel State

```bash
# اقرأ حالة الـ reset من kernel (لو الـ .status op متعمل)
# من داخل driver أو عبر debugfs custom entry

# لو reset مبني على GPIO
gpioget <gpiochip> <offset>
# القيمة 1 = deasserted (يشتغل)، 0 = asserted (في reset)

# قارن مع ما يقوله الـ kernel
cat /sys/kernel/debug/gpio | grep -A2 'reset'

# تحقق من حالة الـ device بعد reset
dmesg | grep -i 'reset\|probe\|deferred'
```

---

#### 2. Register Dump Techniques

```bash
# اقرأ register الـ reset control مباشرةً من الـ hardware
# لازم تعرف العنوان الفيزيائي من DTS أو Datasheet

# باستخدام devmem2
devmem2 0x<phys_addr> w

# مثال: لو reset controller عند عنوان 0xFF750000 على Rockchip
devmem2 0xFF750000 w   # اقرأ reset register الأول

# باستخدام /dev/mem (لازم root)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xFF750000 & ~0xFFF)
    offset = 0xFF750000 & 0xFFF
    val = struct.unpack('<I', m[offset:offset+4])[0]
    print(hex(val))
"

# باستخدام io tool (من package devmem2 أو busybox)
io -4 -r 0xFF750000
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو الـ reset line GPIO:

```
- وصّل probe على الـ reset pin الخارجي للـ IC
- ابحث عن:
  * Pulse width: لازم تكون أطول من minimum reset pulse الـ datasheet (غالباً 1–100 µs)
  * Polarity: active-low ولا active-high — تطابق مع DTS (reset-active = <1>)
  * Timing: الـ reset لازم يتحرر بعد power-on بوقت كافي

ASCII: Reset Sequence على Logic Analyzer
─────────────────────────────────────────
RST_N  ─┐          ┌────────────────────
         └──────────┘
         |<- assert->|<-- deassert -->
         |  (low)    |   (device runs)
─────────────────────────────────────────
```

لو الـ reset مبني على bus (I2C/SPI register):
- استخدم protocol analyzer على الـ bus
- تحقق إن الـ write للـ reset bit وصل فعلاً

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| مشكلة HW | Pattern في الـ Kernel Log |
|-----------|--------------------------|
| الـ reset line مش متوصل (floating) | `device probe failed` + `timeout` في الـ consumer driver |
| polarity معكوسة في DTS | Device مش بيتحرك / بيفضل في reset دايماً |
| Minimum pulse width مش كافية | Device unstable بعد probe ، random resets |
| Power sequencing خاطئ | `reset: -EPROBE_DEFER` متكرر مع timeout في النهاية |
| Reset controller نفسه مش بيشتغل | `reset controller not found` أو `of_parse_phandle failed` |
| GPIO mux خاطئ | `gpio: pin mux conflict` أو gpio request fails |

---

#### 5. Device Tree Debugging

```bash
# اتحقق إن الـ DT متحمّل صح
ls /sys/firmware/devicetree/base/

# ابحث عن reset controller node
find /sys/firmware/devicetree/base -name 'reset-controller*'

# اقرأ #reset-cells
cat /sys/firmware/devicetree/base/<path-to-rcdev>/#reset-cells | xxd

# اقرأ resets property من consumer device
cat /sys/firmware/devicetree/base/<path-to-consumer>/resets | xxd
# كل 4 bytes = phandle، كل الـ args بعدها حسب #reset-cells

# استخدم dtc لتحويل الـ DTB المحمّل لـ readable format
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 'reset'

# تحقق من الـ phandle references
grep -r 'resets' /sys/firmware/devicetree/base/ 2>/dev/null
```

تحقق إن:
- `#reset-cells` في rcdev node يساوي `of_reset_n_cells` في الـ driver
- الـ phandle في consumer's `resets` بيشاور على rcdev الصح
- الـ `reset-names` تطابق ما بيطلبه الـ consumer بـ `reset_control_get(dev, "name")`

---

### Practical Commands

---

#### مجموعة 1: فحص التسجيل والحالة

```bash
#!/bin/bash
# === Reset Controller Quick Health Check ===

echo "=== Loaded Reset Modules ==="
lsmod | grep -i reset

echo "=== Reset-related Platform Devices ==="
ls /sys/bus/platform/devices/ | grep -iE 'reset|cru|src|prcm'

echo "=== Kernel Log (reset) ==="
dmesg | grep -i 'reset' | tail -30

echo "=== GPIO State (reset lines) ==="
cat /sys/kernel/debug/gpio 2>/dev/null | grep -i reset

echo "=== Deferred Probes ==="
cat /sys/kernel/debug/devices_deferred 2>/dev/null
```

**مثال output:**
```
=== Loaded Reset Modules ===
reset_simple          16384  0
reset_rockchip        20480  0

=== Kernel Log (reset) ===
[    1.234] reset: reset controller [ff750000.cru] registered with 408 resets
[    1.567] dwc3 ff600000.dwc3: assigning shared reset: usb3
[    1.890] mmc0: reset deasserted
```

**تفسير:**
- `registered with 408 resets` → الـ controller اشتغل صح وعنده 408 reset line
- `deasserted` → الـ device خرج من reset وجاهز

---

#### مجموعة 2: ftrace لتتبع الـ assert/deassert

```bash
#!/bin/bash
# === Trace reset assert/deassert calls ===

TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

echo 'reset_control_assert
reset_control_deassert
reset_control_reset
reset_control_status' > $TRACE/set_ftrace_filter

echo function > $TRACE/current_tracer
echo 1 > $TRACE/tracing_on

# شغّل العملية اللي عايز تتبعها (مثلاً probe device)
sleep 5

echo 0 > $TRACE/tracing_on
cat $TRACE/trace

# cleanup
echo nop > $TRACE/current_tracer
echo > $TRACE/set_ftrace_filter
```

**مثال output:**
```
# tracer: function
#
           <...>-142   [001] .... 12.345: reset_control_assert <-dwc3_core_init
           <...>-142   [001] .... 12.346: reset_control_deassert <-dwc3_core_init
```

**تفسير:**
- الـ caller هو `dwc3_core_init` — يعني الـ USB controller بيعمل reset لنفسه عند الـ init

---

#### مجموعة 3: Dynamic Debug

```bash
# فعّل debug لكل ملفات الـ reset subsystem
echo 'file drivers/reset/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه المفعّل
grep 'drivers/reset' /sys/kernel/debug/dynamic_debug/control | grep '=p'

# شوف النتيجة
dmesg -w
# أو
journalctl -k -f | grep reset
```

---

#### مجموعة 4: فحص DT بالتفصيل

```bash
#!/bin/bash
# === DT Reset Verification ===

echo "=== Reset Controllers in DT ==="
find /sys/firmware/devicetree/base -name '#reset-cells' | while read f; do
    dir=$(dirname $f)
    name=$(cat $dir/name 2>/dev/null || echo "unknown")
    cells=$(xxd $f | awk '{print $2}')
    echo "  Node: $dir | name: $name | #reset-cells: 0x$cells"
done

echo ""
echo "=== Consumers with 'resets' property ==="
find /sys/firmware/devicetree/base -name 'resets' | head -20
```

**مثال output:**
```
=== Reset Controllers in DT ==="
  Node: /sys/firmware/devicetree/base/cru | name: cru | #reset-cells: 0x00000001

=== Consumers with 'resets' property ==="
/sys/firmware/devicetree/base/usb@ff600000/resets
/sys/firmware/devicetree/base/mmc@ff500000/resets
```

**تفسير:**
- `#reset-cells: 0x00000001` → كل reset specifier فيه argument واحد (الـ id)
- لو `#reset-cells` مش متطابق مع الـ driver's `of_reset_n_cells` → هيتعطل الـ `of_xlate`

---

#### مجموعة 5: devmem2 Register Inspection

```bash
# مثال على Rockchip RK3399 CRU reset registers
# Reset registers تبدأ من offset 0x400 في الـ CRU

BASE=0xFF760000  # RK3399 CRU base

# اقرأ أول 8 reset registers
for i in 0 4 8 12 16 20 24 28; do
    addr=$(printf "0x%X" $((BASE + 0x400 + i)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value at" | awk '{print $NF}')
    echo "SOFTRST_REG$((i/4)): $addr = $val"
done
```

**مثال output:**
```
SOFTRST_REG0: 0xFF760400 = 0x00000000   # all deasserted
SOFTRST_REG1: 0xFF760404 = 0x00000002   # bit1 asserted = some IP in reset
```

**تفسير:**
- bit = 1 → الـ IP في حالة reset (asserted)
- bit = 0 → الـ IP شتغل (deasserted)
- طابق الـ bit positions مع الـ datasheet والـ driver's `of_xlate` mapping
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ USB Controller مش بيشتغل بعد الـ suspend

#### العنوان
**USB hub مش بيرجع بعد system suspend/resume على industrial gateway**

#### السياق
شركة بتبني industrial IoT gateway بـ RK3562. الجهاز بيشتغل Linux 6.6، وبيدخل suspend بعد فترة خمول. المشكلة ظهرت في production: بعد أول resume، الـ USB hub مش بيشتغل، والـ connected sensors بتختفي من `/sys/bus/usb/devices/`.

#### المشكلة
الـ USB controller عنده reset line خاصة بيه في الـ Device Tree. بعد الـ resume، الـ driver بيحاول `reset_control_deassert()` لكن الـ reset controller نفسه لسه مش registered — لأن probe order خلى الـ USB driver يـ probe قبل الـ reset controller driver.

#### التحليل
الـ reset subsystem يشتغل كالآتي:

```c
/* consumer يطلب reset control */
reset_control_deassert(rstc);
  /* بيدور في قائمة reset_controller_list */
  /* لو الـ rcdev مش موجود بعد → يرجع -EPROBE_DEFER */
```

الـ `reset_controller_dev` بيتضاف للـ global list لما بـ:
```c
int reset_controller_register(struct reset_controller_dev *rcdev);
```
وبيتشال بـ:
```c
void reset_controller_unregister(struct reset_controller_dev *rcdev);
```

لو الـ USB driver probe اشتغل وطلب reset قبل ما الـ CRU (Clock/Reset Unit) driver يـ register الـ `rcdev`، الـ lookup بيفشل. لكن المشكلة الفعلية هنا في الـ resume path: الـ CRU driver بيعمل `reset_controller_unregister()` في الـ suspend ثم `reset_controller_register()` تاني في الـ resume — وفي اللحظة دي الـ USB driver بيحاول deassert قبل ما الـ register يكتمل.

تحقق من الـ `of_node` في الـ `reset_controller_dev`:
```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;
    struct module *owner;
    struct list_head list;           /* يتضاف للـ global list */
    struct list_head reset_control_head;
    struct device *dev;
    struct device_node *of_node;     /* ده اللي بيتطابق مع الـ DT phandle */
    /* ... */
    unsigned int nr_resets;
};
```

الـ `of_xlate` بتترجم الـ DT specifier لـ `id` يتبعت للـ ops:
```c
int (*of_xlate)(struct reset_controller_dev *rcdev,
                const struct of_phandle_args *reset_spec);
```

لو الـ rcdev مش في الـ list، الـ xlate مش بتتكلم أصلاً.

#### الحل
الـ suspend/resume path في الـ CRU driver لازم **ميعملش** unregister/register. الصح هو إن الـ ops نفسها تتعامل مع الـ power state:

```c
/* في الـ CRU driver — suspend */
static int rk3562_cru_suspend(struct device *dev)
{
    /* احفظ الـ registers بس، متعملش unregister */
    rk3562_cru_save_context(cru);
    return 0;
}

/* resume */
static int rk3562_cru_resume(struct device *dev)
{
    rk3562_cru_restore_context(cru);
    /* الـ rcdev لسه registered، مفيش مشكلة */
    return 0;
}
```

للتشخيص:
```bash
# شوف الـ reset controllers المسجلة
cat /sys/kernel/debug/reset/reset_controllers

# trace الـ probe order
dmesg | grep -E "(reset|cru|usb)" | head -40

# تحقق من الـ defer list
cat /sys/kernel/debug/devices_deferred
```

#### الدرس المستفاد
الـ `reset_controller_register/unregister` ليست للـ power management — هي للـ probe/remove فقط. الـ suspend/resume لازم يبقى transparent للـ reset subsystem.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش بيظهر عند cold boot

#### العنوان
**الـ HDMI output فاضي على TV box بعد cold boot، بيشتغل بس بعد reboot**

#### السياق
منتج Android TV box رخيص بـ Allwinner H616، Android 12. المستخدمين بيشتكوا إن الشاشة بتبقى سودا عند أول تشغيل، لكن لو عملوا reboot من الـ settings، الـ HDMI بيشتغل عادي. المشكلة بتحصل في ~30% من الـ units.

#### المشكلة
الـ HDMI controller محتاج reset assert ثم deassert عشان يـ initialize صح. الـ driver بيستخدم `reset_control_ops::reset()` اللي المفروض تعمل self-deasserting reset. لكن الـ H616 CRU driver بيـ register الـ `rcdev` بـ `nr_resets` غلط في بعض الـ variants.

#### التحليل
الـ `reset_control_ops` فيها:
```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

الـ `id` جاي من الـ `of_xlate`:
```c
int (*of_xlate)(struct reset_controller_dev *rcdev,
                const struct of_phandle_args *reset_spec);
```

الـ default `of_reset_simple_xlate` بتعمل:
```c
if (reset_spec->args[0] >= rcdev->nr_resets)
    return -EINVAL;
return reset_spec->args[0];
```

لو الـ HDMI reset index في الـ DT هو `64`، والـ driver سجّل `nr_resets = 64` (بدل `65` مثلاً)، الـ xlate بترجع `-EINVAL`. الـ consumer بيتعامل مع الـ error كـ optional reset وبيكمل من غير reset — وعلى cold boot الـ hardware state بيكون undefined.

بعد الـ reboot، الـ bootloader أو الـ kernel نفسه بيعمل full reset للـ SoC، فالـ HDMI controller بيـ initialize من state نظيف.

```bash
# على الـ board — شوف الـ reset line
cat /sys/kernel/debug/reset/reset_controllers
# ابحث عن الـ H616 CRU entry ورقم الـ nr_resets

# شوف error في الـ dmesg
dmesg | grep -i "reset.*invalid\|EINVAL.*reset\|hdmi.*reset"
```

#### الحل
في الـ H616 CRU driver:

```c
/* قبل */
rcdev->nr_resets = H616_RST_NUMBER;  /* كانت 64 بدل 65 */

/* بعد — تأكد من الـ enum */
rcdev->nr_resets = H616_RST_NUMBER;  /* لازم = آخر index + 1 */
```

وفي الـ Device Tree، تأكد إن الـ HDMI node بيستخدم `resets-names` وبيتعامل مع الـ error:

```dts
hdmi: hdmi@6000000 {
    resets = <&ccu RST_BUS_HDMI>, <&ccu RST_BUS_HDMI_SUB>;
    reset-names = "hdmi", "sub";
};
```

في الـ driver:
```c
/* لو الـ reset required مش optional */
rstc = devm_reset_control_get(dev, "hdmi");
if (IS_ERR(rstc))
    return dev_err_probe(dev, PTR_ERR(rstc), "failed to get reset\n");
```

#### الدرس المستفاد
الـ `nr_resets` لازم يكون = آخر valid reset ID + 1. أي off-by-one هنا بيخلي آخر reset line invisible للـ subsystem. والـ consumer لازم يعامل الـ reset كـ required مش optional لو الـ hardware محتاجه للـ init.

---

### السيناريو 3: STM32MP1 — I2C على custom board بيعمل hang بعد NACK

#### العنوان
**الـ I2C bus بيتعلق permanently بعد slave device بيعمل NACK على STM32MP157**

#### السياق
custom industrial board بـ STM32MP157C، Linux 5.15. في القلب فيه I2C connected temperature sensors. لما sensor بيكون absent أو buggy وبيعمل NACK، الـ I2C bus بيتعلق، والـ kernel بيـ log: `i2c i2c-1: controller timed out`. بعد كده الـ I2C bus مش بيشتغل خالص لحد ما الـ system يـ reboot.

#### المشكلة
الـ STM32 I2C controller محتاج hardware reset عشان يخرج من الـ locked state. الـ driver لازم يستخدم `reset_control_ops::reset()` أو `assert/deassert` sequence. المشكلة إن الـ driver بيتعامل مع الـ timeout لكن مش بيعمل hardware reset — بيعمل بس software register reset.

#### التحليل
الـ STM32 I2C driver بيحتفظ بـ `reset_controller_dev` عن طريق الـ consumer API، لكن الـ provider (STM32 RCC driver) بيسجل الـ rcdev كالآتي:

```c
/* في الـ STM32 RCC driver */
struct reset_controller_dev rcdev = {
    .ops = &stm32_reset_ops,
    .of_node = np,
    .nr_resets = STM32MP1_LAST_RESET,
    .of_reset_n_cells = 1,
    /* of_xlate = default simple xlate */
};
reset_controller_register(&rcdev);
```

الـ `stm32_reset_ops` عندها:
```c
static const struct reset_control_ops stm32_reset_ops = {
    .assert   = stm32_reset_assert,
    .deassert = stm32_reset_deassert,
    /* reset = NULL ← مفيش self-deasserting reset! */
};
```

لما الـ I2C driver بيحاول:
```c
reset_control_reset(i2c_rstc);
/* بيتحول لـ:
   rcdev->ops->reset(rcdev, id)
   لو ops->reset == NULL → يرجع -ENOTSUPP
*/
```

الـ driver بيـ ignore الـ error ويكمل، والـ hardware لسه locked.

#### الحل
في الـ I2C timeout handler:

```c
static int stm32_i2c_wait_free_bus(struct stm32_i2c_dev *i2c_dev)
{
    /* حاول software recovery أولاً */
    if (i2c_recover_bus(&i2c_dev->adap) == 0)
        return 0;

    /* لو فشل، hardware reset */
    if (i2c_dev->rst) {
        reset_control_assert(i2c_dev->rst);
        udelay(2);
        reset_control_deassert(i2c_dev->rst);
        /* reinitialize الـ controller */
        stm32_i2c_hw_init(i2c_dev);
    }
    return 0;
}
```

تأكد من الـ DT:
```dts
i2c1: i2c@40012000 {
    resets = <&rcc I2C1_R>;
    reset-names = "i2c";
};
```

للتشخيص:
```bash
# شوف حالة الـ reset line
cat /sys/kernel/debug/reset/reset_controllers

# تحقق من الـ I2C bus state
i2cdetect -y 1

# kernel log
dmesg | grep -E "i2c.*timeout|i2c.*reset"
```

#### الدرس المستفاد
لما `reset_control_ops::reset` بيكون `NULL`، الـ `reset_control_reset()` بيرجع `-ENOTSUPP`. الـ recovery path لازم يستخدم `assert` + `deassert` explicitly، ومش يعتمد على الـ `reset()` shortcut.

---

### السيناريو 4: i.MX8MM — Automotive ECU، SPI Flash مش بيتعرف عليه

#### العنوان
**الـ SPI NOR Flash مش بيظهر في `/dev` على automotive ECU بـ i.MX8M Mini**

#### السياق
شركة automotive بتبني ECU بـ i.MX8M Mini للـ infotainment. الـ SPI NOR Flash (W25Q128) بيتوصل عن طريق ECSPI2. على بعض الـ boards، الـ flash مش بيظهر في `/dev/mtd*` خالص. على boards تانية نفس الـ revision، بيشتغل عادي. الـ SPI signals بتبان صح على الـ oscilloscope.

#### المشكلة
الـ ECSPI controller على i.MX8M عنده reset line منفصلة. الـ CCF (Common Clock Framework) + Reset driver لازم يـ register الـ rcdev مع الصح `of_args` أو `of_node`. على بعض الـ boards، الـ bootloader بيعمل partial init للـ ECSPI ويسيبه في state غريبة، والـ Linux driver مش بيعمل hardware reset قبل الـ init.

#### التحليل
الـ `reset_controller_dev` عنده field مهم:

```c
struct reset_controller_dev {
    /* ... */
    struct device_node *of_node;     /* الـ primary lookup key */
    const struct of_phandle_args *of_args; /* للـ GPIO-based reset */
    /* ... */
};
```

الـ consumer lookup بيشتغل على الـ `of_node` للـ match:

```c
/* في reset.c — consumer يطلب reset */
/* بيدور في rcdev->list على الـ rcdev اللي of_node بتاعه
   بتطابق الـ phandle في DT الـ consumer */
```

لو الـ `imx8mm-clk` driver (اللي بيـ register الـ reset controller) اتـ probe بـ `of_node` غلط — مثلاً بسبب DT alias conflict — الـ ECSPI driver مش هيلاقي الـ rcdev ويـ probe defer أو يكمل من غير reset.

تحقق:
```bash
# شوف الـ registered reset controllers وأل of_node بتاع كل واحد
cat /sys/kernel/debug/reset/reset_controllers

# شوف الـ deferred devices
cat /sys/kernel/debug/devices_deferred

# kernel log
dmesg | grep -E "ecspi|spi.*reset|imx.*reset"
```

#### الحل
في الـ ECSPI driver، الـ init sequence لازم يكون:

```c
static int spi_imx_probe(struct platform_device *pdev)
{
    struct spi_imx_data *spi_imx;

    /* احصل على الـ reset */
    spi_imx->rst = devm_reset_control_get_optional_exclusive(&pdev->dev, NULL);
    if (IS_ERR(spi_imx->rst))
        return dev_err_probe(&pdev->dev, PTR_ERR(spi_imx->rst),
                             "failed to get reset\n");

    /* اعمل hardware reset دايماً قبل الـ init */
    if (spi_imx->rst) {
        reset_control_assert(spi_imx->rst);
        usleep_range(10, 20);
        reset_control_deassert(spi_imx->rst);
        usleep_range(10, 20);  /* انتظر الـ controller يـ stabilize */
    }

    /* باقي الـ init */
}
```

الـ DT لازم يكون صح:
```dts
ecspi2: spi@30830000 {
    #address-cells = <1>;
    #size-cells = <0>;
    compatible = "fsl,imx8mm-ecspi", "fsl,imx51-ecspi";
    reg = <0x30830000 0x10000>;
    resets = <&src IMX8MQ_RESET_ECSPI2_RESET>;
};
```

وتأكد إن `imx8mm-clk` node و`src` node عندهم الـ `of_node` الصح في الـ DTS — مفيش duplicate phandles.

#### الدرس المستفاد
الـ `of_node` في `reset_controller_dev` هو الـ primary matching key. أي DT misconfiguration (duplicate nodes, wrong phandle) بتخلي الـ consumer مش يلاقي الـ rcdev. دايماً اعمل hardware reset explicit في الـ probe حتى لو الـ bootloader "المفروض" عمله.

---

### السيناريو 5: AM62x — IoT Sensor Node، الـ Ethernet مش بيشتغل بعد driver module reload

#### العنوان
**الـ Ethernet interface بتختفي بعد `rmmod/insmod` على AM62x IoT gateway**

#### السياق
IoT sensor aggregator بـ TI AM625 (AM62x family)، Debian Linux. الـ ops team بتعمل hot-reload للـ cpsw_new Ethernet driver كجزء من الـ OTA update procedure. بعد الـ reload، الـ interface `eth0` مش بتظهر. `dmesg` بيقول `cpsw: failed to get reset`.

#### المشكلة
الـ Ethernet controller محتاج reset controller اللي بيكون جزء من TI AM62x CTRL module. لما الـ `rmmod cpsw_new` بيحصل، الـ consumer release بيتحصل عادي. لكن لما الـ `insmod` بييجي، الـ probe بيفشل لأن الـ reset controller driver نفسه عمل `devm_reset_controller_register()` — وده مشكلته إنه مرتبط بـ device lifetime، مش module lifetime.

#### التحليل
```c
/* الـ AM62x reset provider بيستخدم */
int devm_reset_controller_register(struct device *dev,
                                   struct reset_controller_dev *rcdev);
```

الـ `devm_` version بتعمل auto-unregister لما الـ `dev` يـ detach. لو الـ reset provider device نفسه عمل unbind/rebind (بسبب الـ OTA)، الـ rcdev بيتشال من الـ list. لو الـ provider عمل rebind قبل الـ Ethernet driver probe بالكامل، ممكن يكون fine. لكن لو في race condition أو الـ provider مش rebind تلقائياً، الـ consumer بيلاقي الـ list فاضي.

الـ `reset_controller_head` في `reset_controller_dev`:
```c
struct list_head reset_control_head;  /* قائمة الـ consumers الحاليين */
```

ده بيخلي الـ unregister آمن — مفيش dangling pointers — لكن بيخلي الـ consumers الجدد مش يلاقوا الـ provider.

```bash
# شوف الـ reset controllers بعد الـ reload
cat /sys/kernel/debug/reset/reset_controllers

# شوف لو الـ provider محتاج rebind
ls /sys/bus/platform/drivers/ti-syscon-reset/

# جرب rebind الـ provider
echo "some-addr.reset-ctrl" > /sys/bus/platform/drivers/ti-syscon-reset/bind
```

#### الحل
الـ OTA procedure لازم تأخذ في الاعتبار الـ dependency order:

```bash
#!/bin/bash
# OTA hot-reload script

# 1. أوقف الـ consumer أولاً
rmmod cpsw_new

# 2. لو الـ provider اتأثر، unbind/rebind
echo "8000000.reset-ctrl" > /sys/bus/platform/drivers/ti-syscon-reset/unbind
sleep 0.1
echo "8000000.reset-ctrl" > /sys/bus/platform/drivers/ti-syscon-reset/bind
sleep 0.1

# 3. reload الـ consumer
modprobe cpsw_new

# 4. تحقق
ip link show eth0
```

أو الأفضل: استخدم `reset_controller_register()` (بدون devm) في الـ provider لو بتحتاج إن الـ registration تفضل حتى لو الـ device اتـ unbound بشكل temporary — وتعمل unregister يدوياً في الـ `remove()`:

```c
/* في الـ provider driver */
static int ti_reset_probe(struct platform_device *pdev)
{
    /* ... */
    return reset_controller_register(&priv->rcdev);  /* not devm */
}

static int ti_reset_remove(struct platform_device *pdev)
{
    reset_controller_unregister(&priv->rcdev);
    return 0;
}
```

#### الدرس المستفاد
الـ `devm_reset_controller_register()` مناسبة للـ simple cases، لكن في بيئات الـ hot-reload والـ OTA، استخدم `reset_controller_register()` اليدوية مع explicit `unregister` في الـ `remove`. الـ dependency order بين الـ reset provider والـ consumer لازم تكون explicit في الـ OTA scripts.
## Phase 7: مصادر ومراجع

---

### التوثيق الرسمي للـ Kernel

| المصدر | الرابط |
|--------|--------|
| **Reset controller API** — التوثيق الرسمي الكامل للـ subsystem | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) |
| نسخة `.rst` خام من kernel.org | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) |
| نسخة v5.18 من التوثيق | [kernel.org/doc/html/v5.18/driver-api/reset.html](https://www.kernel.org/doc/html/v5.18/driver-api/reset.html) |
| **Device Tree bindings** للـ reset controllers | [github.com — Documentation/devicetree/bindings/reset](https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/reset) |
| الـ header الرئيسي `reset-controller.h` على GitHub | [github.com — include/linux/reset-controller.h](https://github.com/torvalds/linux/blob/master/include/linux/reset-controller.h) |
| الـ consumer API header `reset.h` على GitHub | [github.com — include/linux/reset.h](https://github.com/torvalds/linux/blob/master/include/linux/reset.h) |

---

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأساسي لمتابعة تطور الـ kernel — الروابط دي كلها حقيقية من نتائج البحث:

| المقالة | الرابط |
|---------|--------|
| **Add generic GPIO reset driver** — إضافة driver للـ GPIO-based reset lines للـ framework | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) |
| **Add support for MSM's mmio clock/reset controller** — دمج الـ reset controller مع الـ clock framework في Qualcomm | [lwn.net/Articles/580579](https://lwn.net/Articles/580579/) |
| **Introduction of STM32MP13 RCC driver (Reset Clock Controller)** — مثال حقيقي على RCC driver | [lwn.net/Articles/888111](https://lwn.net/Articles/888111/) |
| **Add Mobileye EyeQ system controller support (clk, reset, pinctrl)** — تطبيق متكامل | [lwn.net/Articles/979170](https://lwn.net/Articles/979170/) |
| **clk: en7523: reset-controller support for EN7523 SoC** — أحدث مثال على إضافة reset support لـ SoC | [lwn.net/Articles/1039543](https://lwn.net/Articles/1039543/) |
| **Add Arria10 System Manager Reset Controller** | [lwn.net/Articles/714705](https://lwn.net/Articles/714705/) |

---

### Patchwork ومناقشات الـ Mailing List

**الـ** patchwork و lore هما الأرشيف الرسمي لكل patches الـ kernel:

| الموضوع | الرابط |
|---------|--------|
| **[v4,2/8] reset: Add reset controller API** — الـ patch الأصلي اللي أضاف الـ subsystem (Philipp Zabel, 2013) | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) |
| **[GIT PULL v2] Reset controller API** — أول git pull request للـ subsystem | [lore.kernel.org](https://lore.kernel.org/linux-arm-kernel/1365683829.4388.52.camel@pizza.hi.pengutronix.de/) |
| **[v4,2/2] reset: Add GPIO support to reset controller framework** | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/) |
| **[GIT PULL] Reset controller fixes for v6.2** — Philipp Zabel | [lore.kernel.org](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/) |
| **Why do we need reset_control_get_optional()?** — نقاش مهم عن الـ optional resets | [groups.google.com/fa.linux.kernel](https://groups.google.com/g/fa.linux.kernel/c/9jG5RFPcEZU) |
| **How to initialize hardware state with the shared reset line?** — نقاش عن الـ shared reset | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/1806.0/02774.html) |

---

### ملاحظة عن kernelnewbies.org و elinux.org

**الـ** kernelnewbies.org و elinux.org ما عندهمش صفحات مخصصة للـ reset controller subsystem لحد دلوقتي. بدل كده:

- **الـ** kernelnewbies.org مفيد لمتابعة الـ kernel changelogs — ابحث عن "reset" في صفحة كل إصدار: `https://kernelnewbies.org/LinuxVersions`
- **الـ** [Xilinx/AMD Wiki](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842455/ZynqMP+Linux+Reset-controller+Driver) عنده توثيق عملي جيد للـ ZynqMP reset controller كمثال SoC حقيقي
- **الـ** [STM32 MPU Wiki](https://wiki.st.com/stm32mpu/wiki/Reset_overview) عنده شرح وافر للـ reset controller في STM32MP

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
**الـ** LDD3 متاح مجانًا — الفصول ذات الصلة:

| الفصل | الموضوع |
|-------|---------|
| Chapter 14: The Linux Device Model | الـ `struct device`، الـ driver model، والـ resource management |
| Chapter 3: Char Drivers | أساسيات كتابة drivers ودورة probe/remove |

- رابط التحميل المجاني: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
| الفصل | الموضوع |
|-------|---------|
| Chapter 17: Devices and Modules | الـ device model وتسجيل الـ drivers |
| Chapter 13: The Virtual Filesystem | فهم الـ abstraction layers |

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
| الفصل | الموضوع |
|-------|---------|
| Chapter 16: Kernel Debugging | أدوات تتبع المشاكل في reset drivers |
| Chapter 7: Bootloader Fundamentals | فهم reset sequence عند الـ boot |

---

### مسارات الكود المهمة في الـ Kernel Source

```
drivers/reset/              ← كل implementations الـ reset controllers
include/linux/reset.h       ← consumer API (reset_control_get, assert, deassert)
include/linux/reset-controller.h  ← provider API (الملف اللي بندرسه)
Documentation/driver-api/reset.rst ← التوثيق الرسمي
Documentation/devicetree/bindings/reset/ ← DT bindings لكل controller
```

---

### Search Terms للبحث عن مزيد من المعلومات

للبحث في الـ mailing list archives على `lore.kernel.org`:

```
reset_controller_register site:lore.kernel.org
reset_control_ops assert deassert site:lore.kernel.org
of_reset_simple_xlate device tree
```

للبحث العام:

```
linux kernel "reset_controller_dev" driver example
linux "devm_reset_controller_register" SoC driver
linux reset controller "nr_resets" "of_xlate" implementation
linux reset subsystem shared exclusive reset control
linux "CONFIG_RESET_CONTROLLER" Kconfig driver
```

للبحث في الـ kernel source مباشرة على Elixir Cross Referencer:

- [elixir.bootlin.com — reset-controller.h](https://elixir.bootlin.com/linux/latest/source/include/linux/reset-controller.h)
- [elixir.bootlin.com — drivers/reset/](https://elixir.bootlin.com/linux/latest/source/drivers/reset)
- **الـ** Elixir بيخليك تشوف كل مكان اتاستخدم فيه `reset_controller_register` أو `reset_control_ops` في الـ kernel كله

---

### Commits مهمة في تاريخ الـ Subsystem

| التغيير | الوصف |
|---------|-------|
| أول إضافة للـ reset controller API | Philipp Zabel — Linux 3.9 (2013) |
| إضافة GPIO reset driver | Philipp Zabel — Linux 3.16 (2014) |
| `reset_control_get_optional()` | جعل الـ optional resets آمنة مع NULL |
| إضافة shared reset controls | دعم أكثر من device تشارك نفس reset line |
| `devm_reset_controller_register()` | إضافة managed registration |

للبحث في الـ git log بنفسك:

```bash
git log --oneline drivers/reset/ include/linux/reset*.h
git log --oneline --follow -- include/linux/reset-controller.h
```
## Phase 8: Writing simple module

### الفكرة

**`reset_controller_register()`** هي الفانكشن الأكتر إثارة في الفايل ده — كل reset controller driver بيناديها عشان يسجّل نفسه في الـ kernel. بـ kprobe على الفانكشن دي هنقدر نشوف كل driver بيسجّل reset controller جديد: اسم الـ device، عدد الـ resets، والـ ops المتاحة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_reset_ctrl.c
 *
 * Hooks reset_controller_register() and logs every reset controller
 * that gets registered during boot or at module-load time.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit       */
#include <linux/kernel.h>       /* pr_info                                 */
#include <linux/kprobes.h>      /* kprobe API                              */
#include <linux/reset-controller.h> /* struct reset_controller_dev         */
#include <linux/device.h>       /* dev_name()                              */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on reset_controller_register — log every reset controller");

/* ------------------------------------------------------------------ */
/*  الـ pre-handler: بيتنادى قبل ما reset_controller_register تتنفذ   */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument lands in RDI.
     * On ARM64 it lands in x0. The kprobe core gives us pt_regs,
     * so we read the first syscall-convention register.
     */
#if defined(CONFIG_X86_64)
    struct reset_controller_dev *rcdev =
        (struct reset_controller_dev *)regs->di;
#elif defined(CONFIG_ARM64)
    struct reset_controller_dev *rcdev =
        (struct reset_controller_dev *)regs->regs[0];
#else
    /* Unsupported arch — skip safely */
    return 0;
#endif

    /* Guard against a NULL pointer before we dereference anything */
    if (!rcdev)
        return 0;

    pr_info("[reset_ctrl_probe] registering controller: dev=%s  "
            "nr_resets=%u  "
            "has_reset=%s  has_assert=%s  has_deassert=%s  has_status=%s\n",
            /* dev_name is NULL-safe when dev itself is valid */
            rcdev->dev ? dev_name(rcdev->dev) : "(no dev)",
            rcdev->nr_resets,
            (rcdev->ops && rcdev->ops->reset)    ? "yes" : "no",
            (rcdev->ops && rcdev->ops->assert)   ? "yes" : "no",
            (rcdev->ops && rcdev->ops->deassert) ? "yes" : "no",
            (rcdev->ops && rcdev->ops->status)   ? "yes" : "no");

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/*  تعريف الـ kprobe struct                                            */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "reset_controller_register", /* الـ symbol اللي هنـ hook عليه */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module_init                                                        */
/* ------------------------------------------------------------------ */
static int __init reset_ctrl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[reset_ctrl_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[reset_ctrl_probe] kprobe planted at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                        */
/* ------------------------------------------------------------------ */
static void __exit reset_ctrl_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[reset_ctrl_probe] kprobe removed\n");
}

module_init(reset_ctrl_probe_init);
module_exit(reset_ctrl_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | ماكروهات `module_init` / `module_exit` و `MODULE_LICENSE` |
| `<linux/kprobes.h>` | الـ `struct kprobe` وفانكشنز `register_kprobe` / `unregister_kprobe` |
| `<linux/reset-controller.h>` | تعريف `struct reset_controller_dev` اللي بنـ cast إليه الـ argument |
| `<linux/device.h>` | فانكشن `dev_name()` عشان نطبع اسم الـ device بشكل readable |

**الـ `<linux/reset-controller.h>`** ضروري عشان بدونه مش هنعرف نوصل لـ `rcdev->nr_resets` أو `rcdev->ops` بدون undefined behavior.

---

#### الـ `handler_pre` — سبب اختيار `pre` مش `post`

الـ **pre-handler** بيتنادى *قبل* ما الفانكشن الأصلية تتنفذ، يعني الـ `rcdev` لسه متسجلش — ده بالظبط اللي عايزينه عشان نلوّق معلومات الـ controller *قبل* أي تعديلات داخلية تحصل. الـ post-handler كان هيكون مفيد لو احتجنا نلوّق الـ return value بس مش ضروري هنا.

---

#### قراءة الـ Argument من `pt_regs`

الـ kprobe بتديلنا `struct pt_regs` لا الـ arguments مباشرةً. على **x86-64** الـ argument الأول بيكون في `regs->di` (register RDI)، وعلى **ARM64** في `regs->regs[0]`. الـ `#if` الـ preprocessor ده بيخلي الكود portable من غير ما يـ break على أي architecture.

---

#### الـ `pr_info` — المعلومات المطبوعة

| Field | السبب |
|-------|-------|
| `dev_name(rcdev->dev)` | اسم الـ device في sysfs — بيسهّل تحديد الـ driver |
| `nr_resets` | عدد الـ reset lines اللي الـ controller بيديرها |
| `has_reset/assert/deassert/status` | بنكشف أي ops الـ driver implementها فعلاً |

الـ **null-checks** على `rcdev->dev` و`rcdev->ops` ضرورية لأن بعض الـ controllers بيتسجلوا قبل ما يـ bind لـ device.

---

#### `module_init` و `module_exit`

**`register_kprobe`** بتزرع الـ breakpoint في الـ kernel text segment وبتربط الـ handler بيه. لو فشلت (مثلاً الـ symbol مش exported أو الـ kprobes مش enabled) بنرجع الـ error فوراً.

**`unregister_kprobe`** في الـ `exit` ضرورية جداً — لو ما عملناهاش والـ module اتـ unload، الـ kernel هيفضل يجري كود handler اتمسح من الـ memory وده بيسبب **kernel panic**. ده مش اختياري.

---

### طريقة التجربة

```bash
# في مجلد الـ module
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# حمّل الـ module
sudo insmod kprobe_reset_ctrl.ko

# شوف الـ output (لو في controllers بيتسجلوا بعد الـ load)
sudo dmesg | grep reset_ctrl_probe

# لو عايز تجبر تسجيل: load أي reset controller driver
# مثلاً على Raspberry Pi:
sudo modprobe reset-raspberrypi

# امسح الـ module
sudo rmmod kprobe_reset_ctrl
```

**ملاحظة:** غالباً الـ controllers بتتسجل وقت الـ boot قبل ما تـ load الـ module. عشان تتقدر تشوف نتيجة حية، شغّل `modprobe` لـ driver بيستخدم reset framework بعد تحميل الـ kprobe module.
