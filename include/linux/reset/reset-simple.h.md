## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: Reset Controller Framework

الملف ده جزء من الـ **Reset Controller Framework** في Linux kernel، والـ maintainer بتاعه هو Philipp Zabel من Pengutronix.

---

### الفكرة من الأول: إيه هو الـ Reset في الـ Hardware؟

تخيل إنك عندك جهاز كمبيوتر وعلق — إيه اللي بتعمله؟ بتضغط زرار الـ Reset. نفس الفكرة بالظبط بتحصل جوا الـ SoC (System on Chip).

الـ SoC بيكون فيه عشرات الـ IP blocks — USB controller، Ethernet، SPI، I2C، GPU، إلخ. كل واحد فيهم ممكن يحتاج يتعمله reset عشان:
- يبدأ من حالة معروفة (known state) بعد الـ power-on
- يتعافى من حالة error
- يتوقف عن الشغل لما مش محتاجينه (power saving)

---

### القصة: الـ Reset Line ببساطة

```
┌─────────────────────────────────────────────────────┐
│                      SoC                            │
│                                                     │
│  ┌──────────────────┐         ┌─────────────────┐  │
│  │  Reset Controller │──RST0──▶│   USB Controller │  │
│  │  (Register Bank)  │──RST1──▶│   SPI Controller │  │
│  │                  │──RST2──▶│   I2C Controller │  │
│  │  Bit 0 → RST0   │──RST3──▶│   Ethernet MAC  │  │
│  │  Bit 1 → RST1   │         └─────────────────┘  │
│  │  Bit 2 → RST2   │                               │
│  └──────────────────┘                               │
│           ▲                                         │
│    CPU يكتب فيه                                     │
└─────────────────────────────────────────────────────┘
```

الـ Reset Controller ده مجرد **register (أو مجموعة registers)**، كل bit فيها بيتحكم في reset line واحدة. الـ CPU لما يكتب `1` في bit معين — الـ IP block المقابل له بيدخل في reset. لما يكتب `0` — بيتحرر من الـ reset ويبدأ يشتغل.

---

### المشكلة: كل SoC عمل اللعبة دي بطريقته

بعض الـ SoCs بيعمل reset لما الـ bit = 1 (**active high**).
بعضها بيعمل reset لما الـ bit = 0 (**active low**).
بعضها بيقرأ الـ status بطريقة عكسية.

فكان كل vendor بيكتب driver خاص بيه — تكرار ومشاكل maintenance.

---

### الحل: `reset-simple` — الـ Generic Driver

الـ **reset-simple** جه كحل موحد لكل الـ reset controllers اللي بتشتغل بنفس الفكرة البسيطة دي:
> "عندي register/s، كل bit بيتحكم في reset line، الفرق بس هل active-high ولا active-low"

بدل ما كل SoC يكتب driver من الأول، بيستخدم `reset-simple` ويقوله فقط:
- فين الـ base address؟
- active high ولا active low؟
- فيه delay لازم بين assert وdeassert؟

---

### الـ Header File: `reset-simple.h` — دوره إيه؟

الملف ده هو الـ **public interface** اللي بيعرّف:

**1. الـ `struct reset_simple_data`** — البيانات الأساسية للـ driver:

```c
struct reset_simple_data {
    spinlock_t          lock;           /* حماية الـ register من race conditions */
    void __iomem        *membase;       /* عنوان الـ register في memory */
    struct reset_controller_dev rcdev;  /* بيربطه بالـ framework */
    bool                active_low;     /* reset = bit صفر؟ */
    bool                status_active_low; /* قراءة الـ status معكوسة؟ */
    unsigned int        reset_us;       /* delay بين assert وdeassert بالـ microseconds */
};
```

**2. الـ `reset_simple_ops`** — الـ operations الجاهزة:
- `assert` → ضع الـ IP block في reset
- `deassert` → حرره من الـ reset
- `reset` → assert ثم deassert (reset كامل)
- `status` → هل دلوقتي في reset؟

الـ header ده مهم لأن **drivers تانية** بتستخدمه لما بتبني reset controller أكتر تعقيداً فوق نفس الفكرة البسيطة دي — بدل ما تكتب الـ ops من الأول.

---

### مثال حقيقي: Allwinner A31 SoC

الـ Allwinner A31 عنده register في العنوان `0x01C202D0` مثلاً، كل bit فيه بيـ reset IP block. لكن عنده **active_low = true** — يعني لازم تكتب `0` عشان تعمل reset، و`1` عشان تحرره.

بدون `reset-simple`، كانوا هيكتبوا driver كامل. بعد `reset-simple`، بيقولوا بس في Device Tree:

```
compatible = "allwinner,sun6i-a31-clock-reset";
```

وخلاص — الـ framework بيعرف يتعامل معاه.

---

### الـ `active_low` وإيه معناها بالظبط

```
Active High (العادي):
  Bit = 1  →  Reset مفعّل  (الجهاز واقف)
  Bit = 0  →  Reset مش مفعّل (الجهاز شغال)

Active Low (معكوس):
  Bit = 0  →  Reset مفعّل  (الجهاز واقف)
  Bit = 1  →  Reset مش مفعّل (الجهاز شغال)
```

لاحظ: `active_low` بيتكلم عن قيمة الـ bit في الـ register، مش عن الـ voltage على الـ pin الفعلي.

---

### الـ `status_active_low` — ليه موجودة منفصلة؟

بعض الـ SoCs بعمل hardware design غريب شوية: الـ bit اللي بتكتبه عشان تعمل reset مختلف عن الـ bit اللي بتقراه عشان تعرف هل في reset؟

فـ `status_active_low` بتعرفك: "لو قريت الـ bit وطلع `0` — معناه الجهاز في reset؟" — لو اتجاوبت بـ yes فـ `status_active_low = true`.

---

### الـ `reset_us` — ليه في delay؟

بعض الـ IP blocks مش بتعمل reset فوري — لازم الـ reset signal يفضل active لفترة (microseconds) عشان الـ internal state يتمسح. لو مكتبش delay كافي، الـ reset ممكن ميتمش بشكل صح.

لو `reset_us = 0` — معناه "مش عارفين" ومش بدعموا الـ atomic reset operation (بس assert/deassert منفصلين مدعومين).

---

### SoCs اللي بتستخدم reset-simple

| SoC / Vendor | Compatible String | ملاحظة |
|---|---|---|
| Allwinner A31 | `allwinner,sun6i-a31-clock-reset` | active_low |
| Intel SOCFPGA | `altr,stratix10-rst-mgr` | offset + status_active_low |
| STM32 | `st,stm32-rcc` | active_high |
| Aspeed AST2400/2500/2600 | `aspeed,astXXXX-lpc-reset` | active_high |
| Synopsys DW | `snps,dw-high-reset` / `snps,dw-low-reset` | الاتنين |
| Sophgo CV1800B/SG2042 | `sophgo,*-reset` | active_low |
| ZTE ZX296718 | `zte,zx296718-reset` | active_low |

---

### الملفات المهمة في الـ Subsystem

#### Core Framework
| الملف | الدور |
|---|---|
| `drivers/reset/core.c` | قلب الـ framework — بيدير الـ reset controls ويوفر الـ API للـ consumers |
| `include/linux/reset-controller.h` | تعريف `struct reset_controller_dev` و`struct reset_control_ops` |
| `include/linux/reset.h` | الـ API اللي بيستخدمه أي driver يريد يعمل reset لجهاز تاني |

#### Simple Reset Driver
| الملف | الدور |
|---|---|
| `include/linux/reset/reset-simple.h` | **الملف اللي بندرسه** — interface الـ simple reset |
| `drivers/reset/reset-simple.c` | التنفيذ الفعلي للـ simple reset ops |

#### Hardware-Specific Drivers (بيستخدموا reset-simple أو بيورثوا منه)
| الملف | الـ SoC |
|---|---|
| `drivers/reset/reset-sunxi.c` | Allwinner SoCs (أكبر من simple) |
| `drivers/reset/reset-socfpga.c` | Intel SOCFPGA |
| `drivers/reset/reset-ath79.c` | Qualcomm Atheros AR7xxx |
| `drivers/reset/reset-berlin.c` | Marvell Berlin |
| `drivers/reset/reset-zynq.c` | Xilinx Zynq |
| `drivers/reset/starfive/` | StarFive JH71x0 |
| `drivers/reset/amlogic/` | Amlogic SoCs |

#### Device Tree Bindings
| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/reset/` | مواصفات الـ DT لكل reset controller |
| `Documentation/driver-api/reset.rst` | توثيق الـ API للـ kernel developers |

---

### الصورة الكاملة: من الـ Device Tree للـ Hardware

```
Device Tree
    │
    │  compatible = "allwinner,sun6i-a31-clock-reset"
    │
    ▼
reset-simple driver (reset-simple.c)
    │  يقرأ الـ devdata (active_low, reg_offset, nr_resets)
    │  يعمل devm_reset_controller_register()
    ▼
Reset Controller Framework (core.c)
    │  بيسجل الـ controller ويوفره للـ consumers
    ▼
Consumer Driver (مثلاً USB driver)
    │  reset_control_get() → يجيب الـ reset handle
    │  reset_control_assert() → يعمل reset
    │  reset_control_deassert() → يحرر الجهاز
    ▼
reset_simple_ops
    │  assert/deassert → read-modify-write على الـ register
    ▼
Hardware Register
    │  Bit يتغير → Reset Line تتغير
    ▼
IP Block يدخل/يخرج من Reset
```
## Phase 2: شرح الـ Reset Framework

---

### المشكلة — ليه الـ Reset Framework موجود أصلاً؟

في أي SoC حديث (زي Allwinner، Rockchip، NXP i.MX)، عندك عشرات الـ IP blocks — USB controller، UART، Ethernet MAC، GPU، إلخ. كل واحد فيهم محتاج **reset line** — سلك واحد أو bit واحد في register بيتحكم في إعادة تشغيل الـ IP block من الصفر.

المشكلة كانت إن كل driver كان بيعمل اللي يجيله:
- بيقرأ base address من الـ device tree بنفسه
- بيعمل `ioremap` بنفسه
- بيكتب على الـ register بنفسه بدون أي تنسيق مع حد تاني

النتيجة: **code duplication ضخم**، ولو اتنين drivers شاركوا نفس الـ reset register (وده بيحصل!) ممكن يتعاركوا على نفس الـ bit ويكسروا بعض.

---

### الحل — الـ Reset Framework بيعمل إيه؟

الـ kernel قرر يعمل **abstraction layer** واضح بين:

- **الـ consumer**: الـ driver اللي *محتاج* يعمل reset لجهاز معين (مثلاً USB driver محتاج يعمل reset للـ USB PHY).
- **الـ provider** (اسمه reset controller): الـ driver اللي *يعرف عنوان الـ register* الفعلي وبيعمل العملية الفعلية.

الـ framework بيشتغل زي وسيط — الـ consumer بيطلب reset بالاسم/الـ ID، والـ framework بيوجّه الطلب للـ provider الصح.

---

### الـ Big Picture — أين يجلس الـ Reset Framework في الـ Kernel؟

```
+------------------------------------------------------------------+
|                    Consumer Drivers                              |
|  (USB driver, Ethernet driver, GPU driver, ...)                  |
|                                                                  |
|   reset_control_assert(rstc);                                    |
|   reset_control_deassert(rstc);                                  |
|   reset_control_reset(rstc);                                     |
+-----------------------------+------------------------------------+
                              |
                              |  opaque handle: struct reset_control *
                              v
+------------------------------------------------------------------+
|                   RESET FRAMEWORK CORE                           |
|              drivers/reset/core.c                                |
|                                                                  |
|  - يحتفظ بـ list of registered reset_controller_dev              |
|  - يترجم الـ DT phandle + specifier → controller + ID           |
|  - يدير الـ shared resets (reference counting)                   |
|  - يفوّض الـ ops للـ provider                                    |
+-------+-----------------------------+----------------------------+
        |                             |
        v                             v
+---------------+           +------------------+
| reset_simple  |           |  Other Providers |
| (generic drv) |           | (per-SoC custom) |
|               |           |                  |
| Allwinner,    |           | Tegra, MediaTek,  |
| ASPEED,       |           | Qualcomm, ...    |
| Berlios, ...  |           |                  |
+-------+-------+           +------------------+
        |
        v
+---------------------------+
|   Hardware Register       |
|   (MMIO bit in SoC)       |
|                           |
|  bit=1 → device in reset  |
|  bit=0 → device running   |
+---------------------------+
```

---

### Analogy عميقة — شبكة الكهرباء في مبنى

تخيل مبنى فيه **لوحة كهربائية مركزية** (Distribution Panel):

| عنصر في الـ Analogy | المقابل في الـ Kernel |
|---|---|
| لوحة الكهربائية كلها | `struct reset_controller_dev` |
| كل **circuit breaker** (قاطع) في اللوحة | reset line واحدة — تُعرَّف بـ `id` |
| **الكهربائي** اللي بيفتح/يقفل القاطع | الـ provider driver (reset-simple أو غيره) |
| **الساكن** في الشقة محتاج يقطع كهرباء الغرفة | الـ consumer driver |
| **شركة الكهرباء** اللي بتنظم الطلبات | الـ Reset Framework Core |
| **رقم الشقة + رقم الغرفة** | الـ DT phandle + specifier |

الـ consumer (الساكن) مش محتاج يعرف في أي لوحة القاطع ولا كيف يوصله — بس يطلب "قطع كهرباء غرفة نوم الشقة 5". شركة الكهرباء (الـ framework) تترجم الطلب وتبعته للكهربائي الصح اللي يعرف يفتح القاطع الفعلي.

لو اتنين شقق بتشارك نفس القاطع (shared reset) — الـ framework بيتأكد إن القاطع ما يتقفلش إلا لما **كل الشقق** مش محتاجاه، عن طريق **reference counting**.

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction المحوري هو إن الـ framework يفصل **من يطلب** عن **كيف تتنفذ**، عن طريق ثلاث طبقات:

1. **`struct reset_control *`** — الـ handle اللي الـ consumer يمسكه، opaque تماماً.
2. **`struct reset_controller_dev`** — التجريد اللي الـ provider يسجل نفسه بيه.
3. **`struct reset_control_ops`** — الـ vtable اللي الـ provider بيملأه بالتنفيذ الفعلي.

---

### الـ Structs والعلاقة بينهم

```
struct reset_simple_data          (reset-simple provider's private data)
┌─────────────────────────────────────────────────┐
│  spinlock_t        lock          ← يحمي read-modify-write على الـ register  │
│  void __iomem     *membase       ← base address للـ register block           │
│  bool              active_low    ← منطق التأكيد (0=assert أم 1=assert)       │
│  bool              status_active_low ← منطق القراءة                          │
│  unsigned int      reset_us      ← حد أدنى للـ delay بين assert و deassert   │
│                                                                               │
│  struct reset_controller_dev  rcdev   ← embedded! مش pointer               │
│  ┌──────────────────────────────────────────────┐                            │
│  │  const struct reset_control_ops  *ops         │ ← يشاور على reset_simple_ops │
│  │  struct module                   *owner        │                           │
│  │  struct list_head                 list         │ ← ربط في global list      │
│  │  struct list_head  reset_control_head          │ ← list of active handles  │
│  │  struct device                   *dev          │                           │
│  │  struct device_node              *of_node      │ ← ربط بالـ DT node        │
│  │  int                    of_reset_n_cells       │ ← كام cell في الـ specifier│
│  │  int (*of_xlate)(rcdev, phandle_args)          │ ← يحوّل DT args → ID     │
│  │  unsigned int           nr_resets              │ ← عدد الـ reset lines      │
│  └──────────────────────────────────────────────┘                            │
└─────────────────────────────────────────────────┘

              ↑ container_of() يرجع لـ reset_simple_data من rcdev
```

```
struct reset_control_ops      (vtable — الـ provider بيملأها)
┌──────────────────────────────────────────────────────────┐
│  int (*reset)   (rcdev, id)  ← assert ثم deassert تلقائياً  │
│  int (*assert)  (rcdev, id)  ← يحط الجهاز في reset          │
│  int (*deassert)(rcdev, id)  ← يطلق الجهاز من reset         │
│  int (*status)  (rcdev, id)  ← هل الجهاز في reset دلوقتي؟   │
└──────────────────────────────────────────────────────────┘

reset_simple_ops (المعلنة في reset-simple.h كـ extern const)
→ بتنفذ الـ 4 callbacks دول بناءً على الـ bit في الـ MMIO register
```

---

### كيف يربط الـ Framework بين Consumer و Provider — خطوة خطوة

```
Device Tree:
  uart0 {
      resets = <&ccu RST_BUS_UART0>;   ← phandle + specifier
  };

  ccu: reset-controller@1c20000 {      ← الـ provider node
      #reset-cells = <1>;
  };

1. الـ consumer driver يطلب:
   rstc = devm_reset_control_get(&pdev->dev, "uart");

2. الـ framework يقرأ DT:
   - يلاقي phandle → يشاور على الـ reset_controller_dev المسجل
   - يستدعي of_xlate(rcdev, args) → يطلع ID = RST_BUS_UART0

3. يرجع struct reset_control * مرتبط بـ (rcdev, id)

4. الـ consumer يعمل:
   reset_control_assert(rstc)
   → framework يستدعي rcdev->ops->assert(rcdev, id)
   → reset_simple_ops.assert() تكتب الـ bit في membase
```

---

### الـ active_low و status_active_low — ليه الاتنين مختلفين؟

**مفهوم مهم**: بعض الـ SoCs بيصمموا الـ register بشكل معكوس.

- **`active_low = false`** (الأكثر شيوعاً): كتابة `1` في الـ bit = assert reset (الجهاز متوقف).
- **`active_low = true`**: كتابة `0` في الـ bit = assert reset (الجهاز متوقف).

**`status_active_low`** منفصل لأن أحياناً الـ hardware بيقرأ الـ status بمنطق مختلف عن الـ control:
- ممكن تكتب `1` لتأكيد الـ reset، بس لما تقرأ الـ register تلاقي `0` وهو في reset.

ده بيحصل في بعض الـ Allwinner SoCs حيث الـ write register والـ read register مختلفين في الـ polarity.

---

### الـ reset_us — ليه محتاج delay؟

```
Timeline:
  t=0   assert()   →  bit written, device enters reset
  t=?               → hardware needs minimum time to latch reset
  t=reset_us        deassert() → bit cleared, device starts boot

  إذا reset_us = 0 → reset() operation غير مدعوم
  (الـ framework يرفض الطلب بـ -ENOTSUPP)
```

بعض الـ IP blocks بيحتاج الـ reset signal يفضل active لفترة معينة عشان الـ internal state machine تتصفر صح. بدون الـ delay ده، الجهاز ممكن يطلع من الـ reset بـ state غير محددة.

---

### ما الذي يملكه الـ Framework مقابل ما يفوّضه

| المسؤولية | الـ Reset Framework Core | الـ Provider Driver |
|---|---|---|
| البحث في الـ DT عن الـ provider | نعم | لا |
| ترجمة phandle → controller | نعم | لا |
| Reference counting للـ shared resets | نعم | لا |
| تسجيل الـ controller في الـ global list | نعم (reset_controller_register) | لا |
| كتابة/قراءة الـ MMIO register | لا | **نعم** |
| معرفة الـ active_low polarity | لا | **نعم** |
| تحديد كم reset line موجودة | لا | **نعم** (nr_resets) |
| الـ delay بين assert/deassert | لا | **نعم** (reset_us) |

---

### الـ reset-simple كـ Generic Driver

**`reset-simple`** هو provider جاهز للـ SoCs اللي بيتبع نمط بسيط:
- register block متسلسل
- كل bit في الـ register = reset line واحدة
- كل الـ lines بنفس الـ polarity

بدل ما كل SoC vendor يكتب نفس الـ code، بيستخدموا `reset_simple_ops` مباشرةً ومجرد بيملأوا `reset_simple_data` بالـ base address والـ polarity.

```c
/* مثال: Allwinner H3 reset controller */
static int sun8i_h3_reset_probe(struct platform_device *pdev)
{
    struct reset_simple_data *data;

    data = devm_kzalloc(&pdev->dev, sizeof(*data), GFP_KERNEL);

    data->membase    = devm_platform_ioremap_resource(pdev, 0);
    data->active_low = false;          /* bit=1 means in reset */
    data->reset_us   = 0;             /* self-deasserting not supported */

    data->rcdev.ops      = &reset_simple_ops; /* الـ generic ops */
    data->rcdev.nr_resets = 64;
    data->rcdev.of_node  = pdev->dev.of_node;

    return devm_reset_controller_register(&pdev->dev, &data->rcdev);
}
```

ده بيوضح قوة الـ pattern: الـ vendor مكتبش أي reset logic — كل اللي عمله إنه حدد **أين** الـ register و**ما** منطق الـ polarity.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### Config Options

| Option | المعنى |
|--------|--------|
| `CONFIG_RESET_CONTROLLER` | لو مش enabled، كل دوال التسجيل بتتحول لـ stubs فاضية بترجع 0 |

#### الـ Boolean Flags في `reset_simple_data`

| Field | القيمة | التأثير |
|-------|--------|---------|
| `active_low` | `false` (default) | بيتعمل set للبت عشان assert الـ reset |
| `active_low` | `true` | بيتعمل clear للبت عشان assert الـ reset |
| `status_active_low` | `false` (default) | البت بيكون set لما الـ reset يكون asserted |
| `status_active_low` | `true` | البت بيكون cleared لما الـ reset يكون asserted |

#### الـ Devdata الثابتة (compile-time constants)

| Instance | `reg_offset` | `nr_resets` | `active_low` | `status_active_low` |
|----------|-------------|-------------|-------------|---------------------|
| `reset_simple_socfpga` | `0x20` | `256` (8 banks × 32) | `false` | `true` |
| `reset_simple_active_low` | `0` | من حجم الـ resource | `true` | `true` |
| (default / no devdata) | `0` | من حجم الـ resource | `false` | `false` |

#### الـ Compatible Strings المدعومة

| Compatible String | Devdata |
|-------------------|---------|
| `altr,stratix10-rst-mgr` | `reset_simple_socfpga` |
| `st,stm32-rcc` | none |
| `allwinner,sun6i-a31-clock-reset` | `reset_simple_active_low` |
| `zte,zx296718-reset` | `reset_simple_active_low` |
| `aspeed,ast2400/2500/2600-lpc-reset` | none |
| `bitmain,bm1880-reset` | `reset_simple_active_low` |
| `brcm,bcm4908-misc-pcie-reset` | `reset_simple_active_low` |
| `snps,dw-high-reset` | none |
| `snps,dw-low-reset` | `reset_simple_active_low` |
| `sophgo,cv1800b-reset` | `reset_simple_active_low` |
| `sophgo,sg2042-reset` | `reset_simple_active_low` |

---

### كل الـ Structs المهمة

---

#### 1. `struct reset_simple_data`

**الغرض:** الـ driver data الرئيسية اللي بتمثل controller واحد بسيط — بتشيل كل المعلومات اللازمة عشان تعمل assert/deassert لأي reset line بالـ bit-banging على الـ MMIO registers.

| Field | النوع | الشرح |
|-------|-------|-------|
| `lock` | `spinlock_t` | بيحمي الـ read-modify-write على الـ registers — لازم يكون spinlock مش mutex لأن الـ ops ممكن تتكلم من interrupt context |
| `membase` | `void __iomem *` | الـ base address بعد الـ ioremap — بيشاور على أول reset register بعد أي `reg_offset` |
| `rcdev` | `struct reset_controller_dev` | الـ base class — مُضمَّن جوه الـ struct عشان نعمل `container_of` |
| `active_low` | `bool` | بيحدد اتجاه الـ polarity لعملية الـ assert |
| `status_active_low` | `bool` | بيحدد اتجاه الـ polarity لقراءة الحالة |
| `reset_us` | `unsigned int` | الـ minimum delay بالـ microseconds بين الـ assert والـ deassert — لو 0 الـ `reset()` op مش supported |

**العلاقة بالـ structs التانية:** الـ `rcdev` field مُضمَّن (embedded) مش pointer — ده بيخلي الـ `container_of` macro يعدّي بسهولة من `rcdev*` للـ `reset_simple_data*`.

---

#### 2. `struct reset_controller_dev`

**الغرض:** الـ base class لأي reset controller في الكيرنل — الـ reset core subsystem بيشتغل معاه مباشرة.

| Field | النوع | الشرح |
|-------|-------|-------|
| `ops` | `const struct reset_control_ops *` | الـ function pointers للـ assert/deassert/reset/status |
| `owner` | `struct module *` | الـ module اللي سجّل الـ controller — بيُستخدم لـ reference counting |
| `list` | `struct list_head` | بيربط الـ controller بالـ global list في الـ reset core |
| `reset_control_head` | `struct list_head` | list بالـ reset controls اللي اتطلبت من الـ consumers |
| `dev` | `struct device *` | الـ device اللي بيمثله |
| `of_node` | `struct device_node *` | الـ DT node — بيُستخدم عشان الـ consumers يلاقوا الـ controller من الـ DT |
| `of_args` | `const struct of_phandle_args *` | للـ GPIO-based reset controllers |
| `of_reset_n_cells` | `int` | عدد الـ cells في الـ reset specifier |
| `of_xlate` | function pointer | بيترجم الـ DT specifier لـ reset ID — لو NULL يستخدم الـ default |
| `nr_resets` | `unsigned int` | عدد الـ reset lines المتاحة |

---

#### 3. `struct reset_control_ops`

**الغرض:** الـ vtable — بيفصل الـ reset core عن الـ driver implementation.

| Op | الـ Signature | متى بيتكلم |
|----|--------------|-----------|
| `reset` | `(rcdev, id) → int` | لما consumer يطلب self-deasserting reset كامل |
| `assert` | `(rcdev, id) → int` | لما consumer يطلب يحط الـ peripheral في reset |
| `deassert` | `(rcdev, id) → int` | لما consumer يطلب يطلع الـ peripheral من reset |
| `status` | `(rcdev, id) → int` | لما consumer يطلب يعرف هل الـ peripheral في reset ولا لأ |

**في الـ `reset_simple_ops`:** كل الـ 4 ops متضافة — `reset` بيستخدم `assert` + delay + `deassert`.

---

#### 4. `struct reset_simple_devdata`

**الغرض:** read-only compile-time configuration لكل compatible string — بتتحدد من الـ `of_device_id` table ومش بتتخزن في الـ runtime data.

| Field | الشرح |
|-------|-------|
| `reg_offset` | المسافة بين الـ base resource والأول reset register |
| `nr_resets` | لو > 0 بيـ override الحساب التلقائي من حجم الـ resource |
| `active_low` | بيتنقل لـ `reset_simple_data.active_low` |
| `status_active_low` | بيتنقل لـ `reset_simple_data.status_active_low` |

---

### مخطط العلاقات بين الـ Structs

```
                         ┌─────────────────────────────────────┐
                         │        reset_simple_data             │
                         │                                      │
                         │  lock: spinlock_t                    │
                         │  membase: void __iomem *──────────► MMIO registers
                         │  active_low: bool                    │
                         │  status_active_low: bool             │
                         │  reset_us: unsigned int              │
                         │                                      │
                         │  ┌───────────────────────────────┐  │
                         │  │    reset_controller_dev (rcdev)│  │
                         │  │                               │  │
                         │  │  ops ──────────────────────────┼──┼──► reset_simple_ops
                         │  │  owner ────────────────────────┼──┼──► THIS_MODULE
                         │  │  list ─────────────────────────┼──┼──► global reset controllers list
                         │  │  reset_control_head ───────────┼──┼──► reset_control list (consumers)
                         │  │  dev ──────────────────────────┼──┼──► platform_device.dev
                         │  │  of_node ──────────────────────┼──┼──► device_node (DT)
                         │  │  nr_resets: unsigned int       │  │
                         │  └───────────────────────────────┘  │
                         └─────────────────────────────────────┘
                                          ▲
                                          │ container_of(rcdev, reset_simple_data, rcdev)
                                          │
                         ┌────────────────┴──────────────────┐
                         │         reset_control_ops          │
                         │  .assert  = reset_simple_assert   │
                         │  .deassert= reset_simple_deassert │
                         │  .reset   = reset_simple_reset    │
                         │  .status  = reset_simple_status   │
                         └───────────────────────────────────┘
```

---

### مخطط الـ Lifecycle: من الـ probe للـ teardown

```
  Device Tree
      │
      │  compatible = "allwinner,sun6i-a31-clock-reset"
      ▼
  platform_driver_register(reset_simple_driver)
      │
      ▼
  reset_simple_probe(pdev)
      │
      ├─► of_device_get_match_data()  ──── يجيب الـ devdata الثابتة
      │
      ├─► devm_kzalloc()  ──────────────── يحجز reset_simple_data
      │
      ├─► devm_platform_get_and_ioremap_resource()  ── يعمل ioremap للـ MMIO
      │
      ├─► spin_lock_init(&data->lock)
      │
      ├─► data->rcdev.ops = &reset_simple_ops
      ├─► data->rcdev.nr_resets = resource_size * 8
      ├─► data->rcdev.of_node = dev->of_node
      │
      ├─► [لو في devdata] ينقل reg_offset, nr_resets, active_low, status_active_low
      │
      ├─► data->membase += reg_offset  ──── يضبط الـ base pointer
      │
      └─► devm_reset_controller_register()
              │
              └──► reset core يضيف الـ rcdev للـ global list
                        │
                        ▼
              ┌─── RUNTIME ──────────────────────┐
              │  consumer calls reset_control_*() │
              │  reset core يلاقي الـ rcdev       │
              │  بيكلم ops->assert/deassert/status│
              └──────────────────────────────────┘
                        │
                        ▼
              device_remove / driver_unregister
                        │
                        └──► devm cleanup:
                               ├─► reset_controller_unregister()  (يشيله من الـ list)
                               └─► iounmap + kfree  (تلقائي من devm)
```

---

### مخطط الـ Call Flow: كيف بتشتغل الـ ops

#### `assert` / `deassert` flow

```
consumer code
    │
    │  reset_control_assert(rstc)
    ▼
reset core (drivers/reset/core.c)
    │
    │  يتحقق من الـ exclusive/shared ownership
    │  يجيب rcdev من rstc->rcdev
    │
    │  rcdev->ops->assert(rcdev, id)
    ▼
reset_simple_assert(rcdev, id)
    │
    └──► reset_simple_update(rcdev, id, assert=true)
              │
              ├─► to_reset_simple_data(rcdev)
              │       └─► container_of(rcdev, reset_simple_data, rcdev)
              │
              ├─► bank  = id / 32   (أي register؟)
              ├─► offset= id % 32   (أي bit؟)
              │
              ├─► spin_lock_irqsave(&data->lock, flags)
              │
              ├─► reg = readl(membase + bank*4)
              │
              ├─► [assert=true XOR active_low=false] → reg |= BIT(offset)
              │   [assert=true XOR active_low=true]  → reg &= ~BIT(offset)
              │
              ├─► writel(reg, membase + bank*4)
              │
              └─► spin_unlock_irqrestore(&data->lock, flags)
```

#### `reset` flow (self-deasserting)

```
consumer code
    │
    │  reset_control_reset(rstc)
    ▼
reset core
    │
    │  rcdev->ops->reset(rcdev, id)
    ▼
reset_simple_reset(rcdev, id)
    │
    ├─► [data->reset_us == 0] → return -ENOTSUPP
    │
    ├─► reset_simple_assert(rcdev, id)   ← يعمل assert
    │
    ├─► usleep_range(reset_us, reset_us * 2)  ← ينتظر
    │
    └─► reset_simple_deassert(rcdev, id)  ← يعمل deassert
```

#### `status` flow

```
consumer code
    │
    │  reset_control_status(rstc)
    ▼
reset core
    │
    │  rcdev->ops->status(rcdev, id)
    ▼
reset_simple_status(rcdev, id)
    │
    ├─► bank  = id / 32
    ├─► offset= id % 32
    │
    ├─► reg = readl(membase + bank*4)   ← read-only, لا يحتاج lock
    │
    ├─► bit_val = (reg >> offset) & 1
    │
    └─► return !(bit_val) ^ !(status_active_low)
         ┌────────────────────────────────────────────────┐
         │ status_active_low=false: 1=asserted, 0=deasserted │
         │ status_active_low=true:  0=asserted, 1=deasserted │
         └────────────────────────────────────────────────┘
```

---

### الـ Bit Addressing: كيف بيتحسب الـ Bank والـ Offset

```
reset ID (0-based)
        │
        ├──► bank   = id / 32   →  اختيار الـ register (كل register 32 bit)
        └──► offset = id % 32   →  اختيار الـ bit جوه الـ register

مثال: id = 37
    bank   = 37 / 32 = 1  →  address = membase + 1*4 = membase + 0x04
    offset = 37 % 32 = 5  →  BIT(5) = 0x20

membase:
  ┌──────────┬──────────┬──────────┬──────────┐
  │ reg[0]   │ reg[1]   │ reg[2]   │ reg[3]   │
  │ IDs 0-31 │ IDs 32-63│IDs 64-95 │IDs 96-127│
  └──────────┴──────────┴──────────┴──────────┘
  +0x00       +0x04       +0x08       +0x0C
```

---

### استراتيجية الـ Locking

#### ليه spinlock مش mutex؟

الـ reset ops بيمكن اتنادوا من contexts مختلفة — منها interrupt context. الـ mutex مش safe في interrupt context، فالـ spinlock هو الاختيار الوحيد.

#### إيه اللي بيحميه الـ lock؟

| العملية | محتاج lock؟ | السبب |
|---------|------------|-------|
| `assert` | نعم | read-modify-write على الـ MMIO register |
| `deassert` | نعم | read-modify-write على الـ MMIO register |
| `reset` | لا (مباشرة) | بيكلم assert/deassert اللي هما اللي بياخدوا الـ lock |
| `status` | لا | read-only — atomic على ARMv7+ وx86 |

#### الـ Lock Ordering

```
مفيش lock ordering معقد هنا — lock واحد بس (data->lock) بيحمي
كل الـ register access. مفيش nested locking.

spin_lock_irqsave → يحفظ الـ IRQ state ويعطل الـ interrupts
    └─► readl / writel
spin_unlock_irqrestore → يرجع الـ IRQ state
```

#### ليه `irqsave` وليس مجرد `spin_lock`؟

لو interrupt جه وهو جوه الـ critical section وحاول يعمل reset من interrupt handler — هيحصل deadlock. الـ `irqsave` بيمنع ده بإنه يعطل الـ local interrupts طول فترة الـ lock.

---

### الـ `container_of` Pattern

الـ `rcdev` مُضمَّن جوه `reset_simple_data` — ده بيخلي الـ kernel يعدّي بسهولة من الـ base class للـ derived class:

```c
/* الـ reset core بيكلم الـ ops بـ rcdev pointer */
rcdev->ops->assert(rcdev, id);

/* جوه الـ assert function: */
static inline struct reset_simple_data *
to_reset_simple_data(struct reset_controller_dev *rcdev)
{
    /* بيحسب عنوان الـ reset_simple_data من عنوان الـ rcdev field جوه */
    return container_of(rcdev, struct reset_simple_data, rcdev);
}
```

```
الـ memory layout:
┌─────────────────────────────────────────┐
│         reset_simple_data               │
│  offset 0:  lock (spinlock_t)           │◄── &data
│  offset X:  membase (void __iomem *)    │
│  offset Y:  rcdev (reset_controller_dev)│◄── &data->rcdev  (ده اللي بياخده الـ core)
│  offset Z:  active_low (bool)           │
│  offset Z+1: status_active_low (bool)   │
│  offset W:  reset_us (unsigned int)     │
└─────────────────────────────────────────┘

container_of(ptr=&data->rcdev, type=reset_simple_data, member=rcdev)
    = (reset_simple_data *)((char *)ptr - offsetof(reset_simple_data, rcdev))
    = &data  ✓
```

---

### خلاصة: الـ Design Decisions

| القرار | السبب |
|--------|-------|
| Embedding بدل pointer لـ `rcdev` | بيوفر allocation إضافية ويخلي `container_of` أبسط |
| `devm_*` APIs | بيضمن cleanup تلقائي لو الـ probe فشل أو الـ device اتفصل |
| `spinlock` مش `mutex` | safety في interrupt context |
| `irqsave` مش `irq` | الـ caller ممكن يكون جه من interrupt context نفسه |
| `status` بدون lock | read-only operation — atomic على المعالجات الحديثة |
| `reset_us = 0` يعني unsupported | بيخلي الـ driver explicit إنه مش عارف الـ timing |
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs (Cheatsheet)

#### مجموعات الـ Functions

| Category | Function | Exported? | Purpose |
|----------|----------|-----------|---------|
| **Core Helper** | `to_reset_simple_data()` | لأ (static inline) | تحويل `rcdev` → `reset_simple_data` |
| **Core Engine** | `reset_simple_update()` | لأ (static) | read-modify-write على الـ register |
| **Reset Ops** | `reset_simple_assert()` | لأ (static) | تفعيل الـ reset line |
| **Reset Ops** | `reset_simple_deassert()` | لأ (static) | إلغاء الـ reset line |
| **Reset Ops** | `reset_simple_reset()` | لأ (static) | assert + delay + deassert دفعة واحدة |
| **Reset Ops** | `reset_simple_status()` | لأ (static) | قراءة حالة الـ reset line |
| **Probe/Init** | `reset_simple_probe()` | لأ (static) | تسجيل الـ controller مع الـ reset subsystem |
| **Exported Data** | `reset_simple_ops` | نعم (`EXPORT_SYMBOL_GPL`) | الـ ops table اللي بتستخدمها الـ SoC drivers |

---

### المجموعة 1: Core Helper

الـ helper دي بتوفر الـ type-safe access من الـ generic `reset_controller_dev` pointer للـ `reset_simple_data` الكامل اللي بيحتوي على الـ lock والـ membase وبقية الـ state.

---

#### `to_reset_simple_data()`

```c
static inline struct reset_simple_data *
to_reset_simple_data(struct reset_controller_dev *rcdev)
{
    return container_of(rcdev, struct reset_simple_data, rcdev);
}
```

**بتعمل إيه:**
الـ `reset_control_ops` callbacks بتوصل بـ `rcdev` pointer بس، فالـ helper دي بتستخدم `container_of` عشان ترجع الـ `reset_simple_data` الأكبر اللي بيحتوي على الـ `rcdev` field جوّاه. الفكرة نفس الـ pattern المعتاد في الـ kernel للـ object embedding.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rcdev` | `struct reset_controller_dev *` | الـ pointer اللي بييجي من الـ reset core callbacks |

**Return:** `struct reset_simple_data *` — الـ parent struct.

**Key details:** inline function — صفر overhead في الـ runtime. لو `rcdev` مش embedded صح في `reset_simple_data`، الـ result undefined behavior.

---

### المجموعة 2: Core Engine

الـ engine function الوحيدة `reset_simple_update()` هي اللي بتعمل الشغل الفعلي — كل الـ ops التانية مجرد wrappers عليها. الفكرة إن الـ reset controllers البسيطة بتشتغل كلها بنفس الـ pattern: bit في register = reset line واحدة.

---

#### `reset_simple_update()`

```c
static int reset_simple_update(struct reset_controller_dev *rcdev,
                               unsigned long id, bool assert)
```

**بتعمل إيه:**
بتحسب الـ register address والـ bit offset من الـ `id`، بعدين بتعمل read-modify-write مع حماية بـ `spinlock_irqsave`. بتأخذ في الاعتبار الـ `active_low` flag عشان تعرف هتـset الـ bit أو تـclear.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rcdev` | `struct reset_controller_dev *` | الـ controller device |
| `id` | `unsigned long` | رقم الـ reset line (0-based، flat index عبر كل الـ banks) |
| `assert` | `bool` | `true` = assert reset، `false` = deassert |

**Return:** `0` دايمًا — الـ register write مافيهاش error path على الـ MMIO البسيط.

**Key details — الـ bit addressing:**

```
bank   = id / 32      /* رقم الـ register (كل register 32 bit) */
offset = id % 32      /* رقم الـ bit جوّا الـ register */
addr   = membase + (bank * 4)
```

**الـ active_low logic:**

```c
/* XOR بين assert والـ active_low يحدد هنـset ولا هنـclear */
if (assert ^ data->active_low)
    reg |= BIT(offset);   /* set the bit */
else
    reg &= ~BIT(offset);  /* clear the bit */
```

| `assert` | `active_low` | XOR result | Action |
|----------|--------------|------------|--------|
| true | false | true | set bit (assert = high) |
| true | true | false | clear bit (assert = low) |
| false | false | false | clear bit (deassert = low) |
| false | true | true | set bit (deassert = high) |

**Locking:** `spin_lock_irqsave` / `spin_unlock_irqrestore` — آمن من الـ IRQ context لأن بعض الـ reset lines ممكن تتعمل من interrupt handler.

**Caller context:** يتنادى من `reset_simple_assert()` و`reset_simple_deassert()` بس.

**Pseudocode flow:**

```
reset_simple_update(rcdev, id, assert):
    data = to_reset_simple_data(rcdev)
    bank   = id / 32
    offset = id % 32

    spin_lock_irqsave(&data->lock)
        reg = readl(data->membase + bank*4)
        if (assert XOR active_low):
            reg |= BIT(offset)
        else:
            reg &= ~BIT(offset)
        writel(reg, data->membase + bank*4)
    spin_unlock_irqrestore(&data->lock)

    return 0
```

---

### المجموعة 3: Reset Control Ops

الأربع functions دول هم الـ `.ops` callbacks اللي الـ reset core بيستدعيهم. بيتشاركوا نفس الـ signature pattern: `(rcdev, id)`. بيتسجلوا في `reset_simple_ops` table.

---

#### `reset_simple_assert()`

```c
static int reset_simple_assert(struct reset_controller_dev *rcdev,
                               unsigned long id)
```

**بتعمل إيه:**
wrapper بسيط على `reset_simple_update()` بـ `assert=true`. الـ reset core بتنادي عليها لما consumer يطلب `reset_control_assert()`.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rcdev` | `struct reset_controller_dev *` | الـ controller |
| `id` | `unsigned long` | رقم الـ reset line |

**Return:** نفس return من `reset_simple_update()` — دايمًا `0`.

**Caller context:** الـ reset core من process context أو atomic context حسب الـ consumer.

---

#### `reset_simple_deassert()`

```c
static int reset_simple_deassert(struct reset_controller_dev *rcdev,
                                 unsigned long id)
```

**بتعمل إيه:**
wrapper على `reset_simple_update()` بـ `assert=false`. بتطلق الـ reset line وتخلي الـ device يشتغل.

**Parameters:** نفس `reset_simple_assert()`.

**Return:** `0` دايمًا.

---

#### `reset_simple_reset()`

```c
static int reset_simple_reset(struct reset_controller_dev *rcdev,
                              unsigned long id)
```

**بتعمل إيه:**
بتعمل الـ full reset cycle: assert → delay → deassert في خطوة واحدة. الـ delay بيتحدد من `data->reset_us`. لو الـ `reset_us` بصفر يعني الـ driver مش عارف الـ timing المطلوب ويرجع `-ENOTSUPP`.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rcdev` | `struct reset_controller_dev *` | الـ controller |
| `id` | `unsigned long` | رقم الـ reset line |

**Return:**
- `-ENOTSUPP` لو `reset_us == 0` (مش معروف الـ minimum delay)
- الـ error من `reset_simple_assert()` لو فشل
- `0` على النجاح

**Key details:**

```c
if (!data->reset_us)
    return -ENOTSUPP;          /* الـ consumer لازم يعمل assert/deassert يدوي */

ret = reset_simple_assert(rcdev, id);
if (ret)
    return ret;

usleep_range(data->reset_us, data->reset_us * 2);  /* sleeping delay */

return reset_simple_deassert(rcdev, id);
```

**الـ `usleep_range`:** بتنام بين `reset_us` و`reset_us * 2` microseconds — مينفعش تتنادى من atomic/interrupt context. الـ consumers اللي محتاجين atomic reset لازم يستخدموا `assert`/`deassert` منفصلين.

**Caller context:** process context فقط بسبب `usleep_range`.

**Pseudocode flow:**

```
reset_simple_reset(rcdev, id):
    data = to_reset_simple_data(rcdev)

    if data->reset_us == 0:
        return -ENOTSUPP

    ret = reset_simple_assert(rcdev, id)
    if ret != 0:
        return ret

    sleep(reset_us .. reset_us*2 microseconds)

    return reset_simple_deassert(rcdev, id)
```

---

#### `reset_simple_status()`

```c
static int reset_simple_status(struct reset_controller_dev *rcdev,
                               unsigned long id)
```

**بتعمل إيه:**
بتقرأ الـ register وبتحدد هل الـ reset line هي في حالة asserted أم لأ. بتأخذ في الاعتبار `status_active_low` لأن بعض الـ controllers بتقرأ الـ bit معكوس عن حالة الـ reset الفعلية.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `rcdev` | `struct reset_controller_dev *` | الـ controller |
| `id` | `unsigned long` | رقم الـ reset line |

**Return:**
- `1` = الـ reset في حالة asserted (الـ device في reset)
- `0` = الـ reset في حالة deasserted (الـ device شغال)

**Key details — الـ status logic:**

```c
reg = readl(data->membase + (bank * reg_width));
return !(reg & BIT(offset)) ^ !data->status_active_low;
```

**Truth table:**

| bit value | `status_active_low` | `!(bit)` | `!status_active_low` | XOR = result |
|-----------|---------------------|----------|----------------------|--------------|
| 1 (set) | false | 0 | 1 | 1 → asserted |
| 0 (clear) | false | 1 | 1 | 0 → deasserted |
| 1 (set) | true | 0 | 0 | 0 → deasserted |
| 0 (clear) | true | 1 | 0 | 1 → asserted |

**Locking:** مفيش lock — الـ `readl` atomic على الـ 32-bit aligned registers، ومش فيه read-modify-write هنا.

---

### المجموعة 4: الـ ops Table (Exported Data)

```c
const struct reset_control_ops reset_simple_ops = {
    .assert   = reset_simple_assert,
    .deassert = reset_simple_deassert,
    .reset    = reset_simple_reset,
    .status   = reset_simple_status,
};
EXPORT_SYMBOL_GPL(reset_simple_ops);
```

**بتعمل إيه:**
الـ `reset_simple_ops` هي الـ vtable الكاملة للـ simple reset controller. متاحة بـ `EXPORT_SYMBOL_GPL` عشان الـ SoC-specific drivers اللي بتستخدم نفس الـ bit-bang pattern تقدر تستخدمها مباشرة من غير ما تكتب الـ ops دي من جديد.

**الـ drivers اللي بتستخدم `reset_simple_ops` مباشرة:**
- Allwinner (sun4i, sun6i reset controllers)
- Aspeed LPC reset
- STM32 RCC
- DesignWare high/low reset

---

### المجموعة 5: Probe / Registration

الـ `reset_simple_probe()` هي نقطة الدخول من الـ platform driver infrastructure. بتعمل كل الـ setup اللازم وبتسجّل الـ controller مع الـ reset core.

---

#### `reset_simple_probe()`

```c
static int reset_simple_probe(struct platform_device *pdev)
```

**بتعمل إيه:**
بتخصص الـ `reset_simple_data` struct، بتعمل ioremap للـ register space، بتهيئ الـ spinlock والـ rcdev fields، وبتسجل الـ controller مع الـ reset core بـ `devm_reset_controller_register()`. بتقرأ الـ per-compatible configuration من `of_device_get_match_data()`.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `pdev` | `struct platform_device *` | الـ platform device من الـ OF/DT matching |

**Return:**
- `0` على النجاح
- `-ENOMEM` لو `devm_kzalloc` فشل
- `PTR_ERR(membase)` لو الـ ioremap فشل
- error من `devm_reset_controller_register()` لو التسجيل فشل

**Key details:**

```c
/* 1. الـ devdata بتيجي من الـ of_device_id .data field */
devdata = of_device_get_match_data(dev);

/* 2. تخصيص الـ driver state بـ devm */
data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);

/* 3. ioremap الـ resource الأولى */
membase = devm_platform_get_and_ioremap_resource(pdev, 0, &res);

/* 4. تهيئة الـ rcdev */
data->rcdev.nr_resets = resource_size(res) * BITS_PER_BYTE; /* default */
data->rcdev.ops       = &reset_simple_ops;
data->rcdev.of_node   = dev->of_node;

/* 5. override من الـ devdata لو موجود */
if (devdata) {
    reg_offset              = devdata->reg_offset;
    data->rcdev.nr_resets   = devdata->nr_resets ?: data->rcdev.nr_resets;
    data->active_low        = devdata->active_low;
    data->status_active_low = devdata->status_active_low;
}

/* 6. offset الـ membase لو في reg_offset */
data->membase += reg_offset;

/* 7. تسجيل مع الـ reset core */
return devm_reset_controller_register(dev, &data->rcdev);
```

**الـ `nr_resets` حساب:**
الـ default بيحسب عدد الـ resets من حجم الـ MMIO resource بالـ bits (`resource_size * 8`). لو الـ `devdata->nr_resets` موجود بيـoverride ده. مثلاً SOCFPGA بيحدد `8 banks * 32 = 256 resets` بغض النظر عن حجم الـ resource.

**الـ devm functions:** كل الـ allocations بـ `devm_*` — لو الـ probe نجح والـ device اتفصل بعدين، الـ cleanup بتحصل automatically. مفيش `.remove` function محتاج.

**Caller context:** `__init` / probe context من الـ kernel init أو platform bus matching.

**Pseudocode flow:**

```
reset_simple_probe(pdev):
    devdata = of_device_get_match_data(dev)

    data = devm_kzalloc(...)        /* fail → -ENOMEM */
    membase = devm_platform_get_and_ioremap_resource(...)  /* fail → PTR_ERR */

    spin_lock_init(&data->lock)
    data->membase          = membase
    data->rcdev.owner      = THIS_MODULE
    data->rcdev.nr_resets  = resource_size(res) * 8
    data->rcdev.ops        = &reset_simple_ops
    data->rcdev.of_node    = dev->of_node

    if devdata != NULL:
        apply reg_offset, nr_resets override, active_low, status_active_low

    data->membase += reg_offset

    return devm_reset_controller_register(dev, &data->rcdev)
```

---

### ASCII: علاقة الـ Functions ببعض

```
platform_driver.probe
        │
        ▼
reset_simple_probe()
  ├─ devm_kzalloc()             → reset_simple_data allocation
  ├─ devm_platform_get_and_ioremap_resource()
  ├─ spin_lock_init()
  ├─ populate rcdev fields
  └─ devm_reset_controller_register()
                │
                ▼
         reset core registers rcdev
                │
         (later, consumer calls reset_control_assert/deassert/reset/status)
                │
                ▼
         reset_simple_ops vtable
          ├─ .assert   → reset_simple_assert()
          │                  └─ reset_simple_update(id, assert=true)
          │                        ├─ spin_lock_irqsave
          │                        ├─ readl → modify → writel
          │                        └─ spin_unlock_irqrestore
          ├─ .deassert → reset_simple_deassert()
          │                  └─ reset_simple_update(id, assert=false)
          ├─ .reset    → reset_simple_reset()
          │                  ├─ reset_simple_assert()
          │                  ├─ usleep_range(reset_us, reset_us*2)
          │                  └─ reset_simple_deassert()
          └─ .status   → reset_simple_status()
                             ├─ readl (no lock needed)
                             └─ return bit_value XOR status_active_low logic
```

---

### الـ Compatible Strings والـ Devdata Configuration

| Compatible String | `active_low` | `status_active_low` | `reg_offset` | `nr_resets` |
|-------------------|--------------|---------------------|--------------|-------------|
| `altr,stratix10-rst-mgr` | false | true | 0x20 | 256 (8×32) |
| `st,stm32-rcc` | false | false | 0 | من resource |
| `allwinner,sun6i-a31-clock-reset` | true | true | 0 | من resource |
| `aspeed,ast2400/2500/2600-lpc-reset` | false | false | 0 | من resource |
| `snps,dw-high-reset` | false | false | 0 | من resource |
| `snps,dw-low-reset` | true | true | 0 | من resource |
| `sophgo,cv1800b-reset` | true | true | 0 | من resource |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ **reset controller subsystem** بيسجّل نفسه في `/sys/kernel/debug/` لو كان `CONFIG_DEBUG_FS=y`.

```bash
# اعرض كل الـ reset controllers المسجّلة
ls /sys/kernel/debug/reset/

# اقرأ حالة كل reset line واحدة واحدة (لو الـ driver دعم status)
cat /sys/kernel/debug/reset/<controller-name>/reset<N>
```

الـ `reset-simple` driver نفسه ما بيضيفش entries خاصة بيه في debugfs، لكن الـ framework الأساسي بيعرض:

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/reset/` | directory لكل `reset_controller_dev` |
| `/sys/kernel/debug/reset/<name>/reset<N>` | عدد مرات assert/deassert لكل line |

```bash
# مثال: Allwinner A31
ls /sys/kernel/debug/reset/
# reset0  reset1  reset2 ...

cat /sys/kernel/debug/reset/reset0
# 1  (assertd مرة واحدة)
```

---

#### 2. مدخلات الـ sysfs

```bash
# اعرض الـ reset controller device في sysfs
ls /sys/bus/platform/devices/ | grep reset

# مثال: STM32 RCC
ls /sys/bus/platform/devices/40021000.rcc/

# اقرأ الـ driver اللي bind عليه
cat /sys/bus/platform/devices/40021000.rcc/driver_override
cat /sys/bus/platform/devices/40021000.rcc/uevent
```

الـ **`nr_resets`** مش مكشوفة مباشرة في sysfs، لكن ممكن تشوفها عن طريق:

```bash
# عدد الـ resets = resource_size * 8 bits
cat /proc/iomem | grep -i reset
# 40021000-400213ff : 40021000.rcc   →  1024 bytes = 8192 bits = 8192 reset lines
```

---

#### 3. الـ ftrace — Tracepoints/Events

الـ reset subsystem عنده tracepoints جاهزة:

```bash
# فعّل الـ tracing للـ reset events
mount -t tracefs nodev /sys/kernel/tracing

# اعرض كل الـ reset events المتاحة
ls /sys/kernel/tracing/events/reset/

# فعّل كل reset events
echo 1 > /sys/kernel/tracing/events/reset/enable

# أو events محددة
echo 1 > /sys/kernel/tracing/events/reset/reset_control_assert/enable
echo 1 > /sys/kernel/tracing/events/reset/reset_control_deassert/enable
echo 1 > /sys/kernel/tracing/events/reset/reset_control_reset/enable
echo 1 > /sys/kernel/tracing/events/reset/reset_control_acquire/enable
echo 1 > /sys/kernel/tracing/events/reset/reset_control_release/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/tracing/trace
```

مثال على output:

```
# tracer: nop
#
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
           systemd-1     [000] .....  12.345678: reset_control_deassert: id=5
           systemd-1     [000] .....  12.345700: reset_control_deassert: id=5 ret=0
```

**تتبع الـ `reset_simple_update` بالـ function tracer:**

```bash
echo function > /sys/kernel/tracing/current_tracer
echo reset_simple_update > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
# ... اعمل العملية اللي عايز تتتبعها ...
cat /sys/kernel/tracing/trace
echo 0 > /sys/kernel/tracing/tracing_on
echo nop > /sys/kernel/tracing/current_tracer
```

---

#### 4. الـ printk / Dynamic Debug

**الـ `reset-simple` driver** مش بيستخدم `dev_dbg` كتير، لكن ممكن تفعّل dynamic debug للـ reset framework كله:

```bash
# فعّل كل debug messages في reset subsystem
echo 'file drivers/reset/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو لملف معين
echo 'file drivers/reset/reset-simple.c +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إن الـ rule اتضاف
grep reset /sys/kernel/debug/dynamic_debug/control
```

**عن طريق kernel cmdline:**

```
dyndbg="file drivers/reset/reset-simple.c +p"
```

**الـ `dev_err` و `dev_warn` بتظهر دايمًا** — شوف `/var/log/kern.log` أو `dmesg`.

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config | الوظيفة |
|------------|---------|
| `CONFIG_RESET_CONTROLLER` | يفعّل الـ reset controller framework أساسًا |
| `CONFIG_DEBUG_FS` | يفعّل debugfs لعرض معلومات الـ controllers |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg` / `pr_debug` runtime |
| `CONFIG_TRACING` | يفعّل الـ ftrace infrastructure |
| `CONFIG_LOCKDEP` | يكتشف مشاكل الـ spinlock في `reset_simple_update` |
| `CONFIG_LOCK_STAT` | إحصائيات الـ spinlock contention |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف double-lock وغيرها |
| `CONFIG_PROVE_LOCKING` | proof-of-locking للـ `data->lock` |
| `CONFIG_OF_UNITTEST` | اختبار الـ Device Tree parsing |
| `CONFIG_DEBUG_OBJECTS` | تتبع lifetime الـ objects |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E 'RESET|DEBUG_FS|DYNAMIC_DEBUG|LOCKDEP'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ `reset-simple` driver** بيُسجَّل كـ platform driver، فممكن تستخدم:

```bash
# اعرض كل الـ reset controllers المسجّلة في الـ kernel
ls /sys/bus/platform/drivers/simple-reset/

# bind/unbind يدوي للـ driver (للـ testing)
echo "40021000.rcc" > /sys/bus/platform/drivers/simple-reset/bind
echo "40021000.rcc" > /sys/bus/platform/drivers/simple-reset/unbind

# اعرض معلومات الـ device
cat /sys/bus/platform/devices/40021000.rcc/uevent

# تحقق من الـ compatible string اللي match
cat /sys/bus/platform/devices/40021000.rcc/of_node/compatible
```

**أداة `devm` لتتبع الـ resources:**

```bash
# اعرض الـ managed resources للـ device
cat /sys/kernel/debug/devices_deferred
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|------|
| `simple-reset: probe of <dev> failed with error -ENOMEM` | فشل `devm_kzalloc` — الذاكرة نفدت | فحص الـ RAM، قلّل الـ memory pressure |
| `simple-reset: probe of <dev> failed with error -ENODEV` | فشل `devm_platform_get_and_ioremap_resource` — مفيش `reg` في DT | صحّح الـ Device Tree node |
| `simple-reset: probe of <dev> failed with error -EBUSY` | الـ memory region محجوز بـ driver تاني | فحص `/proc/iomem` لمعرفة مين حاجز العنوان |
| `reset_simple_reset: -ENOTSUPP` | **`reset_us == 0`** — الـ `reset()` op غير مدعومة | اضبط `reset_us` في الـ driver أو استخدم assert+deassert يدوي |
| `(none)` — device يفضل في reset | الـ `active_low` مش صح — الـ bit logic معكوسة | فحص الـ datasheet وصحّح `active_low`/`status_active_low` |
| `of_reset_simple_xlate: invalid reset id` | الـ consumer بيطلب reset ID أكبر من `nr_resets` | فحص DTS للـ consumer وعدد الـ resets في الـ controller |
| `Unable to handle kernel paging request` في `reset_simple_update` | `membase` غلط أو NULL | تحقق من `devm_platform_get_and_ioremap_resource` |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` / `WARN_ON()`

```c
static int reset_simple_update(struct reset_controller_dev *rcdev,
                               unsigned long id, bool assert)
{
    struct reset_simple_data *data = to_reset_simple_data(rcdev);
    int reg_width = sizeof(u32);
    int bank = id / (reg_width * BITS_PER_BYTE);
    int offset = id % (reg_width * BITS_PER_BYTE);
    unsigned long flags;
    u32 reg;

    /* [DEBUG] تحقق إن الـ id في النطاق الصح */
    WARN_ON(id >= rcdev->nr_resets);

    /* [DEBUG] تحقق إن membase مش NULL */
    WARN_ON(!data->membase);

    spin_lock_irqsave(&data->lock, flags);

    reg = readl(data->membase + (bank * reg_width));

    /* [DEBUG] طباعة قيمة الـ register قبل وبعد التعديل */
    pr_debug("reset_simple: bank=%d offset=%d reg_before=0x%08x assert=%d active_low=%d\n",
             bank, offset, reg, assert, data->active_low);

    if (assert ^ data->active_low)
        reg |= BIT(offset);
    else
        reg &= ~BIT(offset);

    writel(reg, data->membase + (bank * reg_width));

    pr_debug("reset_simple: reg_after=0x%08x\n", reg);

    spin_unlock_irqrestore(&data->lock, flags);

    return 0;
}

static int reset_simple_reset(struct reset_controller_dev *rcdev,
                              unsigned long id)
{
    struct reset_simple_data *data = to_reset_simple_data(rcdev);

    /* [DEBUG] لو reset_us == 0 يعني reset() مش مدعوم */
    if (!data->reset_us) {
        pr_warn("reset_simple: reset() called but reset_us=0, returning -ENOTSUPP\n");
        dump_stack();  /* عشان تعرف مين اللي بيستدعي reset() */
        return -ENOTSUPP;
    }
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

الـ `reset_simple_status()` بتقرأ الـ register مباشرة وترجع حالة الـ reset line. للتحقق:

```bash
# 1. اقرأ حالة الـ reset من الـ kernel
cat /sys/kernel/debug/reset/reset5   # مثال: reset line رقم 5

# 2. اقرأ نفس الـ register من الـ hardware مباشرة
# bank = 5 / 32 = 0  →  register offset = 0x00
# bit  = 5 % 32 = 5
devmem2 0x40021000 w   # اقرأ الـ 32-bit register عند base address

# 3. قارن البيتة رقم 5 في النتيجة مع ما أرجعته reset_simple_status
```

**المعادلة:**
```
reg_addr = membase + (id / 32) * 4
bit      = id % 32
value    = (reg >> bit) & 1

# لو active_low=false:  reset_asserted = (value == 1)
# لو active_low=true:   reset_asserted = (value == 0)
```

---

#### 2. تقنيات الـ Register Dump

```bash
# باستخدام devmem2 (لازم تثبته: apt install devmem2)
devmem2 0x40021000 w   # قراءة 32-bit word

# باستخدام /dev/mem (لازم CONFIG_DEVMEM=y و CONFIG_STRICT_DEVMEM=n)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x40021000)
    val = struct.unpack('<I', m.read(4))[0]
    print(f'Register: 0x{val:08x}')
    for i in range(32):
        print(f'  bit[{i:2d}] = {(val >> i) & 1}')
    m.close()
"

# باستخدام io utility (من package edbg أو busybox)
io -4 -r 0x40021000

# dump كل الـ registers في range الـ controller
BASE=0x40021000
for offset in 0 4 8 12 16 20; do
    printf "offset 0x%02x: 0x%08x\n" $offset $(devmem2 $((BASE + offset)) w 2>/dev/null | awk '/Value/{print $NF}')
done
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

```
الـ Reset Line Timing:
─────────────────────

assert:    _____
                |____________________________  (active-low: خط ينزل)
deassert:                                   |‾‾‾‾‾

قِس:
├── T_assert:   الوقت من كتابة الـ register للبداية الفعلية للـ reset
├── T_reset:    مدة بقاء الـ reset مضغوط (لازم >= reset_us)
└── T_deassert: الوقت من كتابة الـ register للارتفاع الفعلي للـ signal
```

**نقاط القياس:**
- شبّك الـ probe على الـ reset pin للـ chip المستهدف
- شغّل الـ kernel وراقب الـ signal لما `reset_simple_reset()` بتُنفَّذ
- تأكد إن `T_reset >= reset_us` المضبوطة في الـ driver
- لو الـ signal ما بيتغيرش → الـ clock للـ register block ممكن يكون معطّل

**ملاحظات:**
- بعض الـ SoCs عندها **clock gating** للـ reset controller نفسه — تأكد إن الـ clock شغّالة قبل تفسير الـ register reads
- الـ write-to-register ممكن يكون **buffered** (write buffer) — استخدم `readl()` بعد `writel()` للـ flush

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Kernel Log | التفسير |
|---------|-------------------|---------|
| الـ reset line مش شغّالة | Device يهنج بعد probe، مفيش error | `active_low` غلط — الـ bit بيتمسح بدل ما يتضبط |
| الـ device مش بيتـ reset صح | `reset_simple_reset: -ENOTSUPP` | `reset_us=0`، أو DT مش بيحدد الـ delay |
| تعارض في الـ memory map | `probe of X failed with error -EBUSY` | عنوان الـ register مضبوط غلط في DT |
| الـ bit يتغير لكن الـ device ما بيتأثرش | مفيش log — المشكلة hardware | الـ reset pin مش موصّل، أو الـ PCB ناقصه pull-up/pull-down |
| read-modify-write race | kernel panic / data corruption | spinlock مش مفعّل، أو interrupt handler بيعدّل نفس الـ register |

---

#### 5. تـ Debugging الـ Device Tree

```bash
# اعرض الـ DT node للـ reset controller
cat /proc/device-tree/soc/reset-controller@40021000/compatible
cat /proc/device-tree/soc/reset-controller@40021000/reg

# أو بشكل أوضح
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "reset-controller"

# تحقق من الـ resets property في الـ consumer
dtc -I fs /proc/device-tree 2>/dev/null | grep -B 5 -A 5 "resets ="

# تحقق إن الـ driver وجد الـ node
dmesg | grep -i "simple-reset\|reset-simple"
```

**مثال DTS صح لـ Allwinner A31:**

```dts
/* في الـ SoC dtsi */
rst: reset-controller@1c202d8 {
    compatible = "allwinner,sun6i-a31-clock-reset";
    reg = <0x01c202d8 0xb>;  /* 11 bytes = 88 bits = 88 reset lines */
    #reset-cells = <1>;
};

/* في الـ board dts */
uart0: serial@1c28000 {
    compatible = "snps,dw-apb-uart";
    reg = <0x01c28000 0x400>;
    resets = <&rst 16>;      /* reset line رقم 16 */
};
```

**أخطاء DT شائعة:**

```bash
# لو الـ #reset-cells مش موجود
dmesg | grep "missing #reset-cells"

# لو الـ reset ID أكبر من nr_resets
dmesg | grep "invalid reset"

# تحقق من nr_resets المحسوبة
# nr_resets = resource_size(reg) * 8
# reg = <base, size>  →  size * 8 = max reset id
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ

**1. تفعيل كل الـ reset tracing:**

```bash
#!/bin/bash
# reset-trace.sh — تفعيل الـ tracing للـ reset subsystem

TRACE=/sys/kernel/tracing

# reset الـ tracer
echo nop > $TRACE/current_tracer
echo 0 > $TRACE/tracing_on

# فعّل reset events
for event in reset_control_assert reset_control_deassert \
             reset_control_reset reset_control_acquire reset_control_release; do
    echo 1 > $TRACE/events/reset/$event/enable 2>/dev/null && \
        echo "Enabled: $event" || echo "Not found: $event"
done

# فعّل function tracing للـ reset_simple functions
echo function > $TRACE/current_tracer
echo 'reset_simple_*' > $TRACE/set_ftrace_filter

echo 1 > $TRACE/tracing_on
echo "Tracing active. Press Ctrl+C to stop."
trap "echo 0 > $TRACE/tracing_on; cat $TRACE/trace" INT
sleep 30
```

**2. قراءة حالة كل الـ reset lines:**

```bash
#!/bin/bash
# dump-resets.sh — اعرض حالة كل الـ reset lines

RESET_DEBUG=/sys/kernel/debug/reset

if [ ! -d "$RESET_DEBUG" ]; then
    echo "debugfs not mounted or CONFIG_DEBUG_FS not enabled"
    exit 1
fi

for ctrl in $RESET_DEBUG/*/; do
    echo "=== Controller: $(basename $ctrl) ==="
    for line in $ctrl/reset*; do
        id=$(basename $line | sed 's/reset//')
        count=$(cat $line 2>/dev/null)
        [ "$count" != "0" ] && echo "  reset${id}: used ${count} times"
    done
done
```

**3. قراءة الـ registers مباشرة بـ Python:**

```bash
python3 << 'EOF'
import mmap, struct, sys

BASE_ADDR = 0x40021000   # غيّر لـ base address الصح
NUM_REGS  = 4            # عدد الـ 32-bit registers

try:
    with open('/dev/mem', 'rb') as f:
        m = mmap.mmap(f.fileno(), NUM_REGS * 4,
                      offset=BASE_ADDR & ~0xFFF)
        page_off = BASE_ADDR & 0xFFF

        print(f"Reset Controller Registers @ 0x{BASE_ADDR:08X}")
        print("-" * 50)
        for i in range(NUM_REGS):
            m.seek(page_off + i * 4)
            val = struct.unpack('<I', m.read(4))[0]
            print(f"  Reg[{i}] offset=0x{i*4:02X}: 0x{val:08X}")
            for bit in range(32):
                if val & (1 << bit):
                    reset_id = i * 32 + bit
                    print(f"    → reset[{reset_id:3d}] (bit {bit}) is SET")
        m.close()
except PermissionError:
    print("Need root + CONFIG_STRICT_DEVMEM=n")
    sys.exit(1)
EOF
```

**4. التحقق من الـ dynamic debug:**

```bash
# فعّل debug للـ reset-simple driver
echo 'file drivers/reset/reset-simple.c +pflmt' \
    > /sys/kernel/debug/dynamic_debug/control

# شوف اللي تم تفعيله
grep reset-simple /sys/kernel/debug/dynamic_debug/control

# راقب الـ output
dmesg -w | grep reset_simple
```

**5. فحص الـ spinlock stats:**

```bash
# لو CONFIG_LOCK_STAT=y
cat /proc/lock_stat | grep reset

# مثال output:
# class name         con-bounces  contentions  waittime-min  waittime-max
# reset_simple_data:lock:0         0            0.00          0.00
```

**6. اختبار assert/deassert يدوي عبر الـ kernel sysfs (لو مدعوم):**

```bash
# مفيش interface مباشر، لكن ممكن تكتب module صغير للـ testing:
# أو تستخدم الـ consumer driver (مثلاً: force bind ثم unbind)
echo "40021000.rcc" > /sys/bus/platform/drivers/simple-reset/unbind
sleep 1
echo "40021000.rcc" > /sys/bus/platform/drivers/simple-reset/bind
dmesg | tail -20
```

**7. مراقبة الـ dmesg أثناء الـ boot للـ reset controller:**

```bash
# فلتر رسائل الـ reset أثناء الـ boot
dmesg | grep -Ei "reset|rst" | grep -v "irq\|interrupt\|timer"

# مثال output طبيعي:
# [    1.234567] simple-reset simple-reset: 88 resets registered
# [    1.234700] sunxi-mmc 1c0f000.mmc: reset control found

# مثال output مع خطأ:
# [    1.234567] simple-reset: probe of 1c202d8.reset-controller failed with error -ENODEV
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على Allwinner H616 — Android TV Box

#### العنوان
**الـ UART console صامت تماماً بعد boot على TV box بـ Allwinner H616**

#### السياق
شركة بتعمل Android TV box رخيص بـ **Allwinner H616** SoC. المنتج في مرحلة bring-up. المهندس شغّل الـ kernel لأول مرة، الـ bootloader اشتغل وطلع serial output، بعدين لما الـ kernel اتحمّل — صمت تام. مفيش أي output على الـ UART console.

#### المشكلة
الـ UART driver اتحمّل، الـ device node اتعمله probe بنجاح، بس مفيش حرف واحد بيطلع. الـ engineer عمل `dmesg` من ADB بعد ما الـ Android boot — شاف إن الـ ttyS0 registered، بس الـ console مش شغال.

#### التحليل
الـ Allwinner H616 reset controller بيستخدم `reset_simple_data` مع **`active_low = true`** (زي ما ظاهر في `reset_simple_dt_ids` للـ `allwinner,sun6i-a31-clock-reset`).

```c
static const struct reset_simple_devdata reset_simple_active_low = {
    .active_low = true,
    .status_active_low = true,
};
```

الـ `reset_simple_update` بيعمل XOR بين `assert` و `active_low`:

```c
if (assert ^ data->active_low)
    reg |= BIT(offset);   // set the bit
else
    reg &= ~BIT(offset);  // clear the bit
```

لو `active_low = true` والـ driver عمل `assert` (reset active):
- `assert=true ^ active_low=true = false` → بيعمل **clear** للـ bit → ده صح

لو الـ engineer حط `active_low = false` بالغلط في الـ DT binding الجديد أو في custom driver:
- `assert=true ^ active_low=false = true` → بيعمل **set** للـ bit → ده عكس المطلوب → الـ UART فاضل في reset state

الـ UART peripheral محتاج deassert عشان يشتغل، بس لو الـ polarity غلط، الـ reset line فاضل active (asserted) حتى بعد ما الـ driver عمل deassert.

فحص الـ status:

```c
return !(reg & BIT(offset)) ^ !data->status_active_low;
```

لو `status_active_low` غلط كمان، الـ `reset_simple_status` هيقول "deasserted" وهو في الحقيقة asserted — الـ driver مش هيحاول يعمل retry.

#### الحل

**أولاً: فحص الـ DT node:**

```bash
# على الـ target
cat /proc/device-tree/clock-controller@3001000/compatible
# المفروض: allwinner,sun6i-a31-clock-reset
```

**ثانياً: التأكد من الـ binding الصح في DTS:**

```dts
/* غلط — مش بيحط active_low */
reset: reset-controller@3001000 {
    compatible = "snps,dw-high-reset";  /* wrong compatible */
    reg = <0x3001000 0x4>;
    #reset-cells = <1>;
};

/* صح */
reset: reset-controller@3001000 {
    compatible = "allwinner,sun6i-a31-clock-reset";
    reg = <0x3001000 0x4>;
    #reset-cells = <1>;
};
```

**ثالثاً: قراءة الـ register مباشرة:**

```bash
# devmem2 أو /dev/mem
devmem2 0x3001000 w
# لو الـ UART reset bit == 0 (active low, should be 1 for deasserted)
# يبقى الـ reset لسه active
```

**رابعاً: لو بتعمل custom driver بتستخدم `reset_simple_data` مباشرة:**

```c
data->active_low = true;        /* H616 clears bit to assert */
data->status_active_low = true; /* H616 reads 0 while asserted */
```

#### الدرس المستفاد
الـ `active_low` في `reset_simple_data` مش بس flag جمالي — ده بيعكس كل منطق الـ assert/deassert. لازم تراجع datasheet الـ SoC وتشوف هل الـ reset register بيتعمله **set** أو **clear** عشان يفعّل الـ reset. الـ compatible string في الـ DT هي اللي بتحدد الـ `active_low` تلقائياً — متحطش compatible خاطئة.

---

### السيناريو الثاني: SPI Flash مش بيتعرف عليه في Industrial Gateway على AM62x

#### العنوان
**الـ SPI NOR flash مش بيظهر على industrial gateway بـ TI AM62x**

#### السياق
شركة بتعمل industrial IoT gateway بـ **TI AM62x** SoC. الـ gateway بيتوصل بـ SPI NOR flash خارجي لتخزين config. الـ board اشتغل، بس `mtdinfo` بيقول مفيش أي MTD devices.

#### المشكلة
الـ `spi-nor` driver بيعتمد على إن الـ SPI controller يكون خرج من reset قبل ما يعمل probe. لو الـ reset controller مش شغال صح، الـ SPI controller هيفضل في reset وأي attempt للـ probe هيفشل بصمت أو بـ timeout.

#### التحليل

الـ AM62x custom reset driver ورث `reset_simple_ops`:

```c
/* في driver الـ AM62x custom */
data->rcdev.ops = &reset_simple_ops;
```

الـ `reset_simple_reset` بتشتغل كده:

```c
static int reset_simple_reset(struct reset_controller_dev *rcdev,
                              unsigned long id)
{
    struct reset_simple_data *data = to_reset_simple_data(rcdev);
    int ret;

    if (!data->reset_us)
        return -ENOTSUPP;  /* ← هنا المشكلة */

    ret = reset_simple_assert(rcdev, id);
    ...
    usleep_range(data->reset_us, data->reset_us * 2);
    return reset_simple_deassert(rcdev, id);
}
```

المهندس نسي يحط `reset_us` في الـ `reset_simple_data`. القيمة فضلت 0. أي consumer بيستدعي `reset_control_reset()` بدل `reset_control_deassert()` هيرجع `ENOTSUPP`.

الـ SPI controller driver بيعمل:

```c
/* في spi controller driver */
ret = reset_control_reset(spi->rst);
if (ret && ret != -ENOTSUPP)
    return ret;
/* لو ENOTSUPP بيكمل بدون reset — الـ peripheral فاضل في reset! */
```

بعض الـ drivers بتتجاهل `ENOTSUPP` وبتكمل، فالـ SPI controller بيكمل probe بدون ما يتعمله reset صح. النتيجة: الـ SPI controller في حالة unknown state، الـ flash مش بيتعرف عليه.

#### الحل

**في الـ platform data أو الـ driver initialization:**

```c
/* لازم تحط reset_us بالقيمة الصح من datasheet */
data->reset_us = 10; /* AM62x SPI reset: minimum 10 µs */
```

**أو استخدم assert/deassert بدل reset:**

```c
/* في الـ SPI controller driver */
reset_control_assert(spi->rst);
udelay(10);
reset_control_deassert(spi->rst);
```

**فحص من command line:**

```bash
# شوف الـ reset controller registered
cat /sys/kernel/debug/reset/*/reset_count 2>/dev/null

# جرب تعمل reset يدوي للـ SPI
echo 1 > /sys/kernel/debug/reset/spi0-reset/reset

# تحقق من MTD بعدين
mtdinfo -a
```

**أو فحص الـ dmesg:**

```bash
dmesg | grep -i "spi\|reset\|ENOTSUPP"
# هتلاقي: "reset_control_reset returned -95"
```

#### الدرس المستفاد
الـ `reset_us = 0` في `reset_simple_data` معناها إن `reset_simple_reset()` هترجع `ENOTSUPP` دايماً. لازم تحدد الـ minimum reset pulse width من الـ datasheet وتحطه في `reset_us`. لو الـ consumer بيستخدم `reset_control_reset()` وانت ناسي الـ `reset_us`، هتتعذب في debugging.

---

### السيناريو الثالث: I2C Sensor يتعلق عند boot على STM32MP1 ECU

#### العنوان
**الـ I2C temperature sensor بيعلّق الـ boot على automotive ECU بـ STM32MP1**

#### السياق
شركة بتعمل automotive ECU بـ **STM32MP157** لتطبيق HVAC control. الـ ECU فيه I2C temperature sensor. بعض الـ boards (مش كلها) بتعلق في boot لمدة 30 ثانية قبل ما تكمل. المشكلة intermittent ومش قابلة للـ reproduce بسهولة.

#### المشكلة
الـ I2C controller بتاع الـ STM32MP1 محتاج reset صريح عند boot. لو الـ reset controller register اتقرأ وانكتب في نفس الوقت من thread تاني (أو IRQ)، ممكن يحصل **race condition** على الـ read-modify-write cycle.

#### التحليل

الـ `reset_simple_update` بتستخدم `spin_lock_irqsave`:

```c
static int reset_simple_update(struct reset_controller_dev *rcdev,
                               unsigned long id, bool assert)
{
    struct reset_simple_data *data = to_reset_simple_data(rcdev);
    int reg_width = sizeof(u32);
    int bank = id / (reg_width * BITS_PER_BYTE);
    int offset = id % (reg_width * BITS_PER_BYTE);
    unsigned long flags;
    u32 reg;

    spin_lock_irqsave(&data->lock, flags);  /* ← protect RMW */

    reg = readl(data->membase + (bank * reg_width));
    if (assert ^ data->active_low)
        reg |= BIT(offset);
    else
        reg &= ~BIT(offset);
    writel(reg, data->membase + (bank * reg_width));

    spin_unlock_irqrestore(&data->lock, flags);

    return 0;
}
```

ده صح ومحمي. بس المشكلة مختلفة:

الـ STM32MP1 بيستخدم `st,stm32-rcc` كـ compatible:

```c
{ .compatible = "st,stm32-rcc", },
/* مفيش devdata → active_low = false, status_active_low = false */
```

الـ I2C driver بيعمل:

```c
ret = reset_control_deassert(i2c->rst);
```

بس مفيش `reset_us` متحدد (القيمة 0 لأن `reset_simple_reset` مش متسمّاه هنا). الـ driver بيعمل deassert مباشرة بدون delay. على بعض الـ boards، الـ power supply بيكون بطيء شوية، والـ I2C controller محتاج وقت أطول بعد deassert عشان يكون جاهز.

الـ I2C driver لو حاول يكتب في الـ register قبل ما الـ controller يخلص stable → controller response غير محدد → kernel ينتظر ACK → timeout بعد 30 ثانية.

#### الحل

**الحل الأول: إضافة delay صريح في الـ I2C platform driver بعد deassert:**

```c
reset_control_deassert(i2c->rst);
usleep_range(100, 200); /* wait for controller to stabilize */
```

**الحل الثاني: لو بتعمل custom board وعندك flexibility في الـ DT، استخدم compatible مع `reset_us`:**

بما إن `st,stm32-rcc` مش بيدعم `reset_us` في الـ devdata، ممكن تعمل custom reset node منفصل بيستخدم `reset_simple_data` مباشرة في driver مخصص:

```c
data->reset_us = 100; /* 100 µs for I2C controller on this board */
```

**الحل الثالث: debugging من الـ field:**

```bash
# افحص الـ I2C reset register مباشرة
devmem2 0x40023800 w  # RCC base على STM32MP1

# شوف الـ I2C controller status
dmesg | grep "i2c\|reset\|timeout"

# جرب تزيد الـ timeout في الـ I2C driver مؤقتاً
echo 50000 > /sys/module/i2c_stm32f7/parameters/timeout_ms
```

#### الدرس المستفاد
الـ `reset_simple_data` بتوفر الـ locking الصح (`spin_lock_irqsave`)، بس مش بتوفر أي delay بعد deassert لو `reset_us = 0`. الـ consumer driver هو المسؤول عن أي stabilization delay. المشاكل الـ intermittent دي غالباً timing-related وبتتضح في boards بـ power supply بطيء.

---

### السيناريو الرابع: USB مش بيشتغل على i.MX8M — Custom Board Bring-up

#### العنوان
**USB host controller مش بيشوف أي devices على i.MX8M custom board**

#### السياق
فريق bring-up بيشتغل على custom board بـ **i.MX8M Plus**. الـ board عندها USB-A port للـ host mode. الـ schematic صح، الـ power supply للـ VBUS شغال، بس `lsusb` مش بيطلع أي device — حتى لو وصّلوا USB stick.

#### المشكلة
الـ USB controller محتاج إن الـ reset line يتعمله deassert. الـ engineer عمل custom reset driver بيستخدم `reset_simple_data`، بس حسب الـ USB controller specification الخاص بـ i.MX8M، الـ reset register bit بيكون **active_low** — يعني **clear = assert**. المهندس حط `active_low = false` بالغلط.

#### التحليل

الـ `reset_simple_update` بتحسب الـ bank و offset بالطريقة دي:

```c
int reg_width = sizeof(u32);                      // 4 bytes
int bank = id / (reg_width * BITS_PER_BYTE);      // id / 32
int offset = id % (reg_width * BITS_PER_BYTE);    // id % 32
```

لو الـ USB reset ID = 2:
- `bank = 2 / 32 = 0` → register offset = 0
- `offset = 2 % 32 = 2` → bit 2

لو `active_low = false` (غلط):
```c
/* assert USB reset */
assert=true ^ active_low=false = true → reg |= BIT(2)  /* sets bit */
/* deassert USB reset */
assert=false ^ active_low=false = false → reg &= ~BIT(2) /* clears bit */
```

بس الـ hardware بيعمل العكس (active low):
- bit=1 → reset deasserted (USB شغال)
- bit=0 → reset asserted (USB في reset)

إذن لما الـ driver بيعمل deassert → بيعمل clear للـ bit → ده في الحقيقة assert على الـ hardware → الـ USB فاضل في reset.

**التحقق من الـ status:**

```c
return !(reg & BIT(offset)) ^ !data->status_active_low;
```

لو `status_active_low = false` و الـ bit = 0 (asserted في reality):
- `!(0) ^ !false = 1 ^ 0 = 1` → driver بيقول "asserted" ← ده صح!

يعني الـ status بيقول asserted — ده clue إن في مشكلة polarity.

#### الحل

**تصحيح الـ `active_low` في الـ driver:**

```c
/* الإعداد الغلط */
data->active_low = false;
data->status_active_low = false;

/* الإعداد الصح للـ i.MX8M USB reset (active low) */
data->active_low = true;
data->status_active_low = true;
```

**أو لو بتستخدم DT compatible من موجود:**

```dts
usbotg1: usb@38100000 {
    compatible = "fsl,imx8mp-dwc3";
    resets = <&reset_controller USB_RESET_ID>;
    ...
};

reset_controller: reset@30390000 {
    compatible = "snps,dw-low-reset";  /* active_low = true */
    reg = <0x30390000 0x4>;
    #reset-cells = <1>;
};
```

**تحقق من الـ register مباشرة:**

```bash
# اقرأ الـ reset register
devmem2 0x30390000 w

# لو الـ bit 2 = 0 → USB في reset (active low)
# لو الـ bit 2 = 1 → USB شغال

# كمان
cat /sys/kernel/debug/usb/devices
dmesg | grep -i "xhci\|dwc3\|reset"
```

#### الدرس المستفاد
الفرق بين `active_low = true` و `active_low = false` بيقلب كل منطق الـ assert/deassert. دايماً راجع الـ SoC datasheet وشوف الـ reset register description: لو بيقول "write 0 to assert reset" → `active_low = true`. الـ `reset_simple_status` ممكن يساعدك تكتشف المشكلة بسرعة.

---

### السيناريو الخامس: HDMI مش بيطلع صورة على RK3562 Media Player

#### العنوان
**HDMI output أسود تماماً على RK3562 media player بعد suspend/resume**

#### السياق
شركة بتعمل Android media player بـ **Rockchip RK3562**. الجهاز بيشتغل كويس أول ما يتشغّل. بس بعد الـ suspend وبعدين الـ resume، الـ HDMI بيرجع أسود. اليوزر لازم يعمل reboot عشان يرجع الصورة.

#### المشكلة
الـ HDMI controller (VOP + HDMI TX) محتاج reset صريح عند resume. الـ suspend/resume sequence في الـ driver بيعمل `reset_control_reset()`. بس الـ `reset_simple_data` اللي بيستخدمها الـ Rockchip custom reset driver مش فيها `reset_us` محددة.

#### التحليل

عند الـ resume، الـ HDMI driver بيعمل:

```c
/* في الـ HDMI resume callback */
ret = reset_control_reset(hdmi->rst);
if (ret == -ENOTSUPP) {
    /* fallback: manual assert/deassert */
    reset_control_assert(hdmi->rst);
    udelay(50);
    reset_control_deassert(hdmi->rst);
} else if (ret) {
    dev_err(dev, "reset failed: %d\n", ret);
    return ret;
}
```

الـ `reset_simple_reset` بتشوف:

```c
if (!data->reset_us)
    return -ENOTSUPP;
```

الـ `reset_us = 0` → بترجع `ENOTSUPP` → الـ driver بيعمل fallback manual.

الـ fallback بيعمل assert ثم `udelay(50)` ثم deassert — ده المفروض يكفي. بس في الـ RK3562 HDMI، الـ reset bit ده shared مع الـ VOP controller في نفس الـ register!

```c
/* bank = 0, offset = 5 for HDMI, offset = 4 for VOP */
int bank = id / 32;   // both in bank 0
int offset = id % 32; // HDMI = bit 5, VOP = bit 4
```

لما الـ HDMI driver بيعمل:
```c
spin_lock_irqsave(&data->lock, flags);
reg = readl(data->membase + 0);  // reads both bits
reg |= BIT(5);                    // sets HDMI reset bit
writel(reg, data->membase + 0);   // writes back — VOP bit unchanged
spin_unlock_irqrestore(&data->lock, flags);
```

الـ spinlock بيحمي من race بين الـ HDMI و VOP threads — ده صح. بس المشكلة إن الـ VOP driver مش بيعمل reset عند resume (بيفترض إن الـ hardware لسه في حالته). فلما الـ HDMI reset بيخلص وبيعمل deassert، الـ VOP registers اتمسحت بسبب الـ shared power domain reset.

#### الحل

**الحل الأول: إضافة `reset_us` الصح في الـ driver:**

```c
/* في الـ Rockchip custom reset driver */
data->reset_us = 50; /* 50 µs: minimum for RK3562 HDMI+VOP reset */
```

ده بيخلي `reset_simple_reset` تشتغل بشكل صح بدل ما الـ driver يعمل fallback.

**الحل الثاني: الـ VOP driver لازم كمان يعمل reset عند resume:**

```c
/* في الـ VOP resume callback */
reset_control_assert(vop->rst);
udelay(50);
reset_control_deassert(vop->rst);
/* ثم أعد initialize الـ VOP registers */
rk3562_vop_init_hw(vop);
```

**الحل الثالث: استخدام shared reset:**

```dts
/* في الـ DTS */
vop: vop@30000000 {
    resets = <&cru SRST_VOP>, <&cru SRST_HDMI>;
    reset-names = "vop", "hdmi";
};
```

وتستخدم `reset_control_get_exclusive` بدل shared access.

**Debugging:**

```bash
# اتحقق من الـ HDMI status بعد resume
dmesg | grep -i "hdmi\|vop\|resume\|reset" | tail -50

# اقرأ الـ reset register
devmem2 0xFD7C0000 w  # RK3562 CRU base (example)

# شوف الـ reset count
cat /sys/kernel/debug/reset/*/reset_count

# فحص HDMI link
cat /sys/class/drm/card0-HDMI-A-1/status
```

#### الدرس المستفاد
الـ `spin_lock_irqsave` في `reset_simple_update` بيحمي الـ read-modify-write من race conditions بين الـ consumers. بس الحماية دي لـ register level فقط — مش بتحمي من الـ side effects على shared peripherals. لما يكون فيه reset registers مشتركة بين أكتر من peripheral، لازم كل الـ consumers يعيدوا initialize بعد أي reset، مش بس الحالة المتضررة واحدة.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|--------|
| [Reset controller API — docs.kernel.org](https://docs.kernel.org/driver-api/reset.html) | التوثيق الرسمي الكامل لـ reset controller API: consumer API، provider API، وشرح `reset_control_ops` |
| [reset.rst — kernel.org](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | نفس التوثيق بصيغة RST الخام من `Documentation/driver-api/reset.rst` |
| [Device Tree Reset Bindings — kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/reset/reset.txt) | كيفية تعريف reset providers وconsumers في Device Tree |
| [ARM Allwinner SoCs — docs.kernel.org](https://docs.kernel.org/arch/arm/sunxi.html) | توثيق رسمي لـ Allwinner/sunxi SoCs اللي كانت الأساس الأول لـ `reset-simple` |
| [Device Tree Bindings Directory — GitHub](https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/reset) | كل ملفات bindings الخاصة بالـ reset controllers في الـ kernel |

### مقالات LWN.net

| المقال | الوصف |
|--------|--------|
| [Add generic GPIO reset driver — LWN.net](https://lwn.net/Articles/585145/) | patch يضيف GPIO reset driver فوق reset controller framework — مثال عملي على تمديد الـ framework |
| [STM32MP13 RCC driver (Reset Clock Controller) — LWN.net](https://lwn.net/Articles/888111/) | إدخال driver لـ STM32MP13 يجمع بين reset وclock في controller واحد |
| [Add Arria10 System Manager Reset Controller — LWN.net](https://lwn.net/Articles/714705/) | reset controller لـ Intel Arria10 FPGA SoC — مثال على استخدام الـ framework في بيئة FPGA |
| [Add Mobileye EyeQ system controller support (clk, reset, pinctrl) — LWN.net](https://lwn.net/Articles/979170/) | دمج reset مع clock وpinctrl في system controller موحد |
| [clk: en7523: reset-controller support for EN7523 SoC — LWN.net](https://lwn.net/Articles/1039543/) | إضافة reset-controller لـ EN7523 RISC-V SoC — مثال حديث على استخدام الـ framework |
| [PCI: Expose and manage PCI device reset — LWN.net](https://lwn.net/Articles/866578/) | نقاش حول تعريض reset control لـ userspace عبر sysfs |

### Patchwork — النقاشات والـ Patches المهمة

**الـ framework نفسه:**

| الـ Patch | الوصف |
|-----------|--------|
| [Reset controller API — v4,2/8 — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | الـ patch الأصلي اللي أدخل reset controller API — بقلم **Philipp Zabel** من Pengutronix |
| [GIT PULL v2: Reset controller API — lore.kernel.org](https://lore.kernel.org/linux-arm-kernel/1365683829.4388.52.camel@pizza.hi.pengutronix.de/) | طلب الـ pull الرسمي للـ reset controller framework |
| [GIT PULL: Reset controller fixes for v6.2 — lore.kernel.org](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/) | آخر pull requests لـ Philipp Zabel كـ maintainer |
| [GPIO support for reset controller framework — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/) | إضافة GPIO support للـ framework |
| [Reset controller updates for v5.19 — Patchwork](https://patchwork.kernel.org/project/linux-soc/patch/20220503160057.46625-1-p.zabel@pengutronix.de/) | آخر التحديثات قبيل v5.19 |

**الـ reset-simple بالتحديد:**

| الـ Patch | الوصف |
|-----------|--------|
| [Allwinner A31 Reset Code — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1398265476-29373-3-git-send-email-maxime.ripard@free-electrons.com/) | الـ patch الأصلي من **Maxime Ripard** اللي أسس عليه الـ `reset-simple` |
| [reset: ti-rstctrl: use the reset-simple driver — Patchwork](https://patchwork.kernel.org/patch/10167365/) | مثال على كيفية إعادة استخدام `reset-simple` في SoC مختلف (TI) |
| [STM32 reset driver — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/4648059.qaGUNaKlSl@wuerfel/) | إضافة STM32 reset driver مبني على نفس المفهوم |

### مصادر كود المصدر المرجعية

| الملف | الوصف |
|-------|--------|
| `include/linux/reset/reset-simple.h` | الهيدر موضوع الدراسة — يعرّف `struct reset_simple_data` و`reset_simple_ops` |
| `drivers/reset/reset-simple.c` | التطبيق الكامل لـ ops: assert، deassert، status، reset |
| `include/linux/reset-controller.h` | تعريف `struct reset_controller_dev` و`struct reset_control_ops` |
| `include/linux/reset.h` | واجهة الـ consumer: `reset_control_get`، `reset_control_assert`، إلخ |
| `drivers/reset/` | مجلد كل reset controller drivers في الـ kernel |

للوصول للكود مباشرة:

```bash
# بتشوف الهيدر
less include/linux/reset/reset-simple.h

# بتشوف التطبيق
less drivers/reset/reset-simple.c

# بتتتبع تاريخ الملف
git log --oneline drivers/reset/reset-simple.c

# بتشوف كل الـ drivers اللي بتستخدم reset-simple
grep -r "reset_simple_ops\|reset-simple" drivers/ --include="*.c" -l
```

### أدوات Kconfig والـ Config Database

| المصدر | الوصف |
|--------|--------|
| [CONFIG_RESET_SIMPLE — cateee.net](https://cateee.net/lkddb/web-lkddb/RESET_SIMPLE.html) | قاعدة بيانات الـ Kconfig للـ `RESET_SIMPLE` — بتشوف فيه أي SoCs بتستخدمه |
| [CONFIG_RESET_CONTROLLER — cateee.net](https://cateee.net/lkddb/web-lkddb/RESET_CONTROLLER.html) | قاعدة بيانات الـ Kconfig للـ `RESET_CONTROLLER` الأساسي |
| [config_reset_controller — kernelconfig.io](https://www.kernelconfig.io/config_reset_controller) | واجهة بديلة لاستكشاف dependencies الـ Kconfig |

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
كتاب مجاني على النت — [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

الفصول ذات الصلة:
- **Chapter 9: Communicating with Hardware** — MMIO، `ioread`/`iowrite`، `void __iomem *` — اللي هو أساس `membase` في `reset_simple_data`
- **Chapter 5: Concurrency and Race Conditions** — الـ `spinlock_t` والـ `spin_lock_irqsave` المستخدمة في read-modify-write

#### Linux Kernel Development — Robert Love (3rd Edition)
الفصول ذات الصلة:
- **Chapter 10: Kernel Synchronization Methods** — `spinlock_t`، متى تستخدمه مقارنة بـ mutex
- **Chapter 17: Devices and Modules** — بنية الـ device drivers وكيفية تسجيل الـ subsystem

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
الفصول ذات الصلة:
- **Chapter 16: Kernel Debugging Techniques** — debugging أثناء تطوير reset drivers
- **Chapter 15: Embedded Linux Porting** — porting kernel لـ SoC جديد وإضافة reset controller له

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 18: Page and Buffer Cache** — مش مباشر، لكن يشرح كيف الـ kernel يدير الـ I/O regions

### مصادر تكميلية

| المصدر | الوصف |
|--------|--------|
| [LKML.org — Linux Kernel Mailing List Archive](https://lkml.org/) | أرشيف كامل لـ LKML — ابحث عن `reset-simple` أو `reset_simple_ops` |
| [lore.kernel.org](https://lore.kernel.org/) | الأرشيف الرسمي الحديث لكل mailing lists — أفضل من LKML |
| [Bootlin — Mali & Allwinner Blog](https://bootlin.com/blog/mali-opengl-support-on-allwinner-platforms-with-mainline-linux/) | Bootlin (free-electrons سابقاً) — الشركة اللي من ضمن موظفيها Maxime Ripard صاحب الـ reset-simple |
| [ZynqMP Reset-controller Driver — Xilinx Wiki](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842455/ZynqMP+Linux+Reset-controller+Driver) | مثال عملي كامل لـ reset controller على Xilinx Zynq |
| [STM32 Reset Overview — ST Wiki](https://wiki.st.com/stm32mpu/wiki/Reset_overview) | شرح مفصل لـ reset architecture على STM32 — مفيد لفهم السياق الـ hardware |

### كلمات البحث الموصى بها

للبحث في LKML / lore.kernel.org:

```
reset-simple.h
reset_simple_ops
reset_simple_data
reset_control_ops assert deassert
devm_reset_controller_register
reset-controller framework Philipp Zabel
Allwinner reset controller Maxime Ripard
```

للبحث في Google / DuckDuckGo:

```
linux kernel reset controller subsystem driver implementation
"reset_simple_ops" site:elixir.bootlin.com
"struct reset_simple_data" linux kernel
linux SoC reset controller active_low spinlock MMIO
reset-controller.h reset_control_ops kernel
```

للبحث في Elixir Cross Reference:

- [elixir.bootlin.com/linux/latest/source/include/linux/reset/reset-simple.h](https://elixir.bootlin.com/linux/latest/source/include/linux/reset/reset-simple.h) — الهيدر مع cross-references كاملة
- [elixir.bootlin.com/linux/latest/source/drivers/reset/reset-simple.c](https://elixir.bootlin.com/linux/latest/source/drivers/reset/reset-simple.c) — التطبيق مع كل الـ callers
## Phase 8: Writing simple module

### الهدف

**الـ** `reset_simple_ops` هو الـ symbol الوحيد المُصدَّر من ملف `reset-simple.c` عبر `EXPORT_SYMBOL_GPL`. هو struct من النوع `reset_control_ops` بيحوي pointers للـ callbacks الأربعة: `assert`, `deassert`, `reset`, `status`. هنستخدم **kprobe** على الدالة `reset_simple_update` — وهي القلب اللي تنادي عليه كل من `assert` و `deassert` — عشان نشوف كل عملية reset بتحصل على أي bank وأي bit.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on reset_simple_update:
 * Logs every assert/deassert that goes through the simple reset controller.
 *
 * reset_simple_update(rcdev, id, assert) is the core helper called by both
 * reset_simple_assert() and reset_simple_deassert(). Hooking it gives us
 * visibility into every bit-level reset operation on the MMIO register bank.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/reset-controller.h>       /* struct reset_controller_dev   */
#include <linux/reset/reset-simple.h>     /* struct reset_simple_data      */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Mohamed <example@example.com>");
MODULE_DESCRIPTION("kprobe on reset_simple_update to trace simple-reset ops");

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler                                                  */
/* ------------------------------------------------------------------ */

/*
 * reset_simple_update signature:
 *   static int reset_simple_update(struct reset_controller_dev *rcdev,
 *                                  unsigned long id, bool assert);
 *
 * On x86-64 the arguments arrive in:
 *   regs->di  => rcdev  (first arg)
 *   regs->si  => id     (second arg)
 *   regs->dx  => assert (third arg, bool promoted to int)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct reset_controller_dev *rcdev;
    struct reset_simple_data    *data;
    unsigned long id;
    bool do_assert;

    /* Extract arguments from CPU registers (x86-64 System V ABI) */
    rcdev     = (struct reset_controller_dev *)regs->di;
    id        = (unsigned long)regs->si;
    do_assert = (bool)regs->dx;

    /* Walk from rcdev back to the containing reset_simple_data */
    data = container_of(rcdev, struct reset_simple_data, rcdev);

    /*
     * Compute which 32-bit register bank and which bit inside it
     * will be touched — same arithmetic used inside reset_simple_update.
     */
    {
        int reg_width = sizeof(u32);               /* 4 bytes            */
        int bank      = id / (reg_width * 8);      /* register index     */
        int offset    = id % (reg_width * 8);      /* bit position 0-31  */

        pr_info("reset-simple-probe: %s  id=%lu  bank=%d  bit=%d  "
                "active_low=%d  membase=%px\n",
                do_assert ? "ASSERT  " : "DEASSERT",
                id, bank, offset,
                data->active_low,
                data->membase);
    }

    /* Return 0: let the original function execute normally */
    return 0;
}

/* ------------------------------------------------------------------ */
/*  kprobe struct                                                        */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    /* Target the internal helper, not the exported ops wrapper */
    .symbol_name = "reset_simple_update",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module_init / module_exit                                           */
/* ------------------------------------------------------------------ */

static int __init reset_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("reset-simple-probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("reset-simple-probe: kprobe planted on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit reset_probe_exit(void)
{
    /* Must unregister before module memory is freed to avoid kernel crash */
    unregister_kprobe(&kp);
    pr_info("reset-simple-probe: kprobe removed\n");
}

module_init(reset_probe_init);
module_exit(reset_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | بيعرّف `struct kprobe` وكل الـ API بتاعه |
| `<linux/reset-controller.h>` | بيعرّف `struct reset_controller_dev` اللي بنعدّي عليها في الـ callback |
| `<linux/reset/reset-simple.h>` | بيعرّف `struct reset_simple_data` اللي فيها `lock`, `membase`, `active_low` |

**الـ** `reset-controller.h` ضروري عشان نعرف شكل الـ `rcdev` ونقدر نعمل `container_of` منه للـ `reset_simple_data`.

---

#### الـ `handler_pre`

الـ callback ده بيتنادي عليه الكيرنل **قبل** ما `reset_simple_update` تتنفّذ فعلاً، وبيعدّيله pointer للـ `pt_regs` اللي فيها حالة الـ registers اللحظة دي.

```c
rcdev     = (struct reset_controller_dev *)regs->di;
id        = (unsigned long)regs->si;
do_assert = (bool)regs->dx;
```

**الـ** System V AMD64 ABI بيحط أول ٣ arguments في `rdi`, `rsi`, `rdx` على التوالي — فبنسحبهم من الـ `pt_regs` مباشرةً بدون أي تعديل.

```c
data = container_of(rcdev, struct reset_simple_data, rcdev);
```

**الـ** `rcdev` مش standalone struct — هو **مدفون جوا** `reset_simple_data`. `container_of` بتحسب offset الـ field وترجعلنا pointer للـ parent struct عشان نوصل لـ `membase` و `active_low`.

```c
int bank   = id / (reg_width * 8);
int offset = id % (reg_width * 8);
```

الـ reset line ID هو flat index في bitmap. كل register عرضه 32-bit، فـ `bank` هو رقم الـ register و `offset` هو رقم البت جواه — نفس الحساب اللي بتعمله `reset_simple_update` أصلاً.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "reset_simple_update",
    .pre_handler = handler_pre,
};
```

بنحدد الـ symbol بالاسم بدل العنوان عشان الكيرنل يحل الـ address وقت الـ `register_kprobe`. **الـ** `pre_handler` بيتنادي عليه قبل تنفيذ أول instruction في الدالة.

> **ملاحظة:** `reset_simple_update` دالة `static` داخل الـ module — الكيرنل بيعرفها في الـ kallsyms لو الكيرنل اتبنى مع `CONFIG_KALLSYMS_ALL=y`. لو مش موجودة، الـ `register_kprobe` هيرجع `-ENOENT` وبنطبع الـ error بوضوح.

---

#### الـ `module_init`

```c
ret = register_kprobe(&kp);
```

بيزرع **breakpoint** (INT3 على x86) على أول byte من `reset_simple_update`. لو فشل — مثلاً الرمز مش موجود أو الـ kprobes متعطّلة — بنرجع الـ error code ومش بيكمل الـ init.

---

#### الـ `module_exit`

```c
unregister_kprobe(&kp);
```

لازم يتنادي عليه قبل ما الـ module يتفكك من الذاكرة. لو اتركنا الـ kprobe مسجّل والـ handler pointer بقى dangling، الكيرنل هيـ panic أول ما أي reset operation تحصل. `unregister_kprobe` بيشيل الـ breakpoint ويضمن مفيش handler شغّال في الجو.

---

### Makefile للتجربة

```makefile
obj-m := reset_simple_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# Build and load
make
sudo insmod reset_simple_probe.ko

# Watch the trace (trigger a reset from another driver or via sysfs)
sudo dmesg -w | grep reset-simple-probe

# Unload
sudo rmmod reset_simple_probe
```

---

### مثال على الـ output

```
[  142.331204] reset-simple-probe: kprobe planted on reset_simple_update @ ffffffffc0a12340
[  143.005812] reset-simple-probe: ASSERT   id=5   bank=0  bit=5  active_low=0  membase=ffff888102a00000
[  143.006100] reset-simple-probe: DEASSERT id=5   bank=0  bit=5  active_low=0  membase=ffff888102a00000
```

يعني: reset line رقم 5 (البت 5 في الريجستر الأول) اتعمل لها assert ثم deassert — ده الـ pulse اللي بيعمله `reset_simple_reset` لما `reset_us > 0`.
