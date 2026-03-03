## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **Reset Controller Framework** في Linux kernel، وبالتحديد هو الـ early-boot driver الخاص بـ SoCs الشركة الصينية **Allwinner** (المعروفة كمان بـ sunxi).

---

### الصورة الكبيرة: ليه محتاجين Reset Controller أصلاً؟

تخيل إنك شغال على كمبيوتر وفيه USB controller، Wi-Fi chip، وـ HDMI encoder — كلهم شغالين جوه نفس الشريحة (SoC). لما الـ kernel يبدأ، بعض الـ hardware components دي ممكن تكون في حالة غلط أو undefined من الـ power-on. محتاج **ترستها** — تبعتلها إشارة "reset" تخليها ترجع لحالة معروفة وابتدي من أول وجديد.

الـ **Reset Controller** هو اللي بيتحكم في الإشارات دي. في الـ Allwinner SoCs، فيه registers معينة — كل bit فيها بيتحكم في reset جهاز معين. تكتب `0` في الـ bit → الجهاز اتعمله reset (active-low). تكتب `1` → الجهاز شغال بشكل طبيعي.

---

### القصة: مشكلة الـ Chicken-and-Egg

في الـ Allwinner sun6i-A31 SoC، الـ clock subsystem (اللي بيتحكم في سرعة كل جزء من الشريحة) محتاج الـ reset controller يشتغل **الأول** — قبل ما الـ kernel ينهي booting ويبدأ يسجل الـ platform drivers بالطريقة العادية.

المشكلة: الـ reset driver العادي بيتسجل عبر `platform_driver` → يعني محتاج الـ kernel ييجي على device وـ يشغله بعدين. لكن لو الـ clock controller نفسه محتاج reset قبل كده، هتدخل في **Chicken-and-Egg problem**:

```
Clock controller يحتاج reset ← Reset controller لسه مش شغال
Reset controller يحتاج driver يتسجل ← Driver يحتاج kernel يكون جاهز
← دايرة مفرغة!
```

**الحل:** ملف `reset-sunxi.c` موجود عشان يكسر الدايرة دي — بيعمل **early initialization** للـ reset controller قبل ما يبدأ الـ normal driver registration.

---

### إيه اللي بيعمله الملف ده بالظبط؟

الملف بيعمل حاجة واحدة بس: **يسجل الـ reset controller للـ AHB1 bus في الـ sun6i-A31 بدري جداً في الـ boot**.

#### الـ Flow:

```
Boot يبدأ
    ↓
arch/arm/mach-sunxi/sunxi.c → sun6i_timer_init()
    ↓
sun6i_reset_init()  ← هنا بييجي reset-sunxi.c
    ↓
بيدور على device tree nodes بالـ compatible = "allwinner,sun6i-a31-ahb1-reset"
    ↓
لكل node: بيعمل sunxi_reset_init(np)
    ↓
    ├── kzalloc: يحجز memory للـ reset_simple_data
    ├── of_address_to_resource: يقرأ عنوان الـ registers من الـ device tree
    ├── request_mem_region: يحجز الـ memory region
    ├── ioremap: يعمل map للـ physical registers في virtual memory
    ├── spin_lock_init: يجهز lock للـ thread safety
    ├── active_low = true: bits 0 تعني reset (Allwinner convention)
    └── reset_controller_register: يسجل الـ controller في الـ framework
    ↓
دلوقتي الـ clock controllers تقدر تعمل reset لنفسها
    ↓
باقي الـ boot يكمل بشكل طبيعي
```

---

### ليه الـ `active_low = true`؟

في الـ Allwinner hardware، الـ reset logic معكوسة:
- **Bit = 0** → الجهاز في حالة reset (متوقف، محجوز)
- **Bit = 1** → الجهاز شغال عادي

ده اللي بنسميه **active-low reset**: لما بتطلق الـ reset، بتكتب `0` مش `1`. الـ `reset_simple.c` بيتعامل مع الموضوع ده عبر الـ `active_low` flag في الـ read-modify-write operation.

---

### الفرق بين الملف ده وـ `reset-simple.c`

| الجانب | `reset-sunxi.c` | `reset-simple.c` |
|---|---|---|
| **وقت التسجيل** | early boot (قبل `platform_driver`) | بعد ما الـ kernel يجهز |
| **الآلية** | `of_address_to_resource` + `ioremap` يدوي | `devm_platform_get_and_ioremap_resource` |
| **الهدف** | sun6i-A31 AHB1 reset فقط | أي simple reset controller |
| **الـ ops** | `reset_simple_ops` (مشتركة) | `reset_simple_ops` (نفس الـ ops) |
| **المناسبة** | early init لكسر الـ chicken-and-egg | normal driver registration |

يعني `reset-sunxi.c` هو wrapper خفيف جداً حوالين `reset-simple.c` — بس وظيفته الأساسية إنه يشتغل **بدري** قبل أي حد تاني.

---

### الملفات المرتبطة اللي المفروض تعرفها

#### Core Framework
- **`drivers/reset/core.c`** — قلب الـ Reset Controller Framework. يحتوي على `reset_controller_register()` وكل الـ API اللي الـ consumers بيستخدموها.
- **`include/linux/reset-controller.h`** — بيعرّف `struct reset_controller_dev` وـ `struct reset_control_ops`.

#### Simple Reset Implementation
- **`drivers/reset/reset-simple.c`** — بيحتوي على الـ `reset_simple_ops` (assert/deassert/status). الـ `reset-sunxi.c` بيستخدم نفس الـ ops دي.
- **`include/linux/reset/reset-simple.h`** — بيعرّف `struct reset_simple_data`.

#### Sunxi-specific
- **`include/linux/reset/sunxi.h`** — بيعلن عن `sun6i_reset_init()`.
- **`arch/arm/mach-sunxi/sunxi.c`** — نقطة الاستدعاء الأولى. هنا بيتنادى `sun6i_reset_init()` في `sun6i_timer_init()` وده اللي بيحصل early في الـ boot.

#### Device Tree
- **`arch/arm/boot/dts/allwinner/sun6i-a31.dtsi`** — بيحتوي على الـ node بـ compatible `"allwinner,sun6i-a31-ahb1-reset"` اللي الـ driver بيدور عليه.

---

### ملخص في جملة واحدة

**الـ `reset-sunxi.c`** هو early-boot shim صغير بيسجل الـ reset controller للـ Allwinner sun6i-A31 **قبل** ما الـ kernel يكمل initialization — عشان يكسر الـ dependency cycle بين الـ clock subsystem والـ reset controller، وبعدين كل الـ sun6i/sun8i boards تقدر تشتغل بشكل طبيعي.
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي SoC زي Allwinner A31، في عشرات الـ peripherals — USB، Ethernet، I2C، SPI، إلخ. كل peripheral محتاج إنه يتعمله **hardware reset** (يتخنق ويتصحى) عشان يشتغل صح. الـ reset بيتتحكم فيه عن طريق **register bits** في control block مخصوص داخل الـ SoC.

**المشكلة الأولى:** كل driver بيحتاج reset كان بيعمل direct memory access للـ registers ده بنفسه — مفيش abstraction. يعني كل driver كان لازم يعرف العنوان الفيزيائي، ويعمل read-modify-write بنفسه، ويعمل لocking بنفسه.

**المشكلة التانية:** بعض الـ peripherals بتشارك نفس الـ reset line — لو driver واحد عمل assert والتاني عمل deassert، الجهاز هيتلخبط.

**المشكلة التلاتة:** في بعض الـ SoCs، الـ reset controller نفسه لازم يتعمل initialize **قبل** ما الـ device model يشتغل خالص — يعني قبل ما تقدر تسجل device driver عادي.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ kernel بيقدم **Reset Controller Framework** — طبقة وسيطة بين:
- **الـ consumers**: الـ drivers اللي محتاجة تعمل reset لجهاز معين
- **الـ providers**: الـ drivers اللي بتتحكم في الـ reset hardware نفسه

الـ framework بيوحّد الـ API، بيعمل reference counting، وبيحل مشكلة الـ shared reset lines.

---

### الـ Big Picture — فين الـ subsystem ده في الـ kernel؟

```
+---------------------------------------------------------------+
|                     Consumer Drivers                          |
|   (USB driver, Ethernet driver, I2C driver, ...)              |
|              devm_reset_control_get()                         |
|              reset_control_assert/deassert()                  |
+-----------------------------+---------------------------------+
                              |
                              v
+---------------------------------------------------------------+
|              Reset Controller Framework (core)                |
|      drivers/reset/core.c                                     |
|                                                               |
|  - Global list of reset_controller_dev                        |
|  - Reference counting for shared lines                        |
|  - OF (Device Tree) translation via of_xlate                  |
|  - devm_ resource management                                  |
+-----------------------------+---------------------------------+
                              |
               +--------------+--------------+
               |                             |
               v                             v
+-----------------------------+  +-----------------------------+
|  reset-simple.c             |  |  reset-sunxi.c (early init) |
|  (generic bit-bang driver)  |  |  (sun6i AHB1 — early boot)  |
|  reset_simple_ops           |  |  sunxi_reset_init()          |
+-----------------------------+  +-----------------------------+
               |                             |
               v                             v
+---------------------------------------------------------------+
|                    Hardware Registers (MMIO)                  |
|   Allwinner SoC Reset Control Registers                       |
|   [bit 0 = reset line 0, bit 1 = reset line 1, ...]          |
+---------------------------------------------------------------+
```

---

### الـ Core Abstraction — الفكرة المركزية

الـ framework بيبني على فكرة واحدة بسيطة:

> **كل reset controller هو provider له عدد من الـ reset lines، كل line ليها ID عددي. الـ consumer بيطلب line بالاسم من الـ Device Tree، والـ framework بيترجمها لـ ID وبيوصّلها للـ provider الصح.**

الـ central struct هي `reset_controller_dev`:

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;  /* assert, deassert, reset, status */
    struct module *owner;
    struct list_head list;                /* linked في global list */
    struct list_head reset_control_head;  /* الـ consumers المتصلين */
    struct device *dev;
    struct device_node *of_node;          /* DT node للـ matching */
    int of_reset_n_cells;
    int (*of_xlate)(...);                 /* DT specifier → reset ID */
    unsigned int nr_resets;               /* عدد الـ reset lines */
};
```

---

### الـ reset_simple_data — اللي بيستخدمه sunxi

الـ `reset_simple_data` هو wrapper حول `reset_controller_dev` بيضيف الـ hardware-specific details:

```c
struct reset_simple_data {
    spinlock_t          lock;       /* حماية الـ read-modify-write */
    void __iomem        *membase;   /* base address للـ MMIO registers */
    struct reset_controller_dev rcdev; /* الـ core struct — embedded */
    bool                active_low; /* 0 = reset active, 1 = running (sunxi) */
    bool                status_active_low;
    unsigned int        reset_us;   /* minimum assert time بالـ microseconds */
};
```

**لماذا `active_low = true` في sunxi؟**
في Allwinner SoCs، الـ convention هو إن الـ bit = 0 يعني "الجهاز في حالة reset"، والـ bit = 1 يعني "الجهاز شغال". ده عكس الـ convention العادي — ومن هنا الـ `active_low`.

---

### العلاقة بين الـ Structs

```
reset_simple_data
+----------------------------------+
| lock (spinlock_t)                |
| membase (void __iomem *)         |
| active_low = true                |
| rcdev ──────────────────────────►+
|    .ops = &reset_simple_ops      |  ← الـ assert/deassert logic
|    .owner = THIS_MODULE          |
|    .nr_resets = size * 8         |  ← كل بت = reset line واحدة
|    .of_node = np                 |  ← ربط بالـ Device Tree
|    .list ──────────────────────► global reset controller list (في core.c)
+----------------------------------+
```

الـ `nr_resets = size * 8` منطقي جداً: لو الـ reset register block حجمه 4 bytes، يبقى في 32 reset line (كل bit = line).

---

### الـ reset_control_ops — الـ callbacks

```c
struct reset_control_ops {
    int (*reset)(rcdev, id);    /* assert + deassert in one shot (self-deasserting) */
    int (*assert)(rcdev, id);   /* اجبر الجهاز يدخل reset */
    int (*deassert)(rcdev, id); /* اطلق الجهاز من الـ reset */
    int (*status)(rcdev, id);   /* اقرأ حالة الـ reset line */
};
```

الـ `reset_simple_ops` هي implementation جاهزة تشتغل مع أي hardware بيستخدم bit-per-line model — وده اللي بيستخدمه sunxi.

---

### Early Init — المشكلة الخاصة بـ sunxi

**الـ Device Model** في Linux بيشتغل بعد ما الـ kernel يعمل initialize لكتير من subsystems. لكن بعض الـ peripherals (زي AHB1 reset controller في A31) لازمة **قبل** كل ده.

```c
/* الـ driver مش بيستخدم platform_device/platform_driver */
/* بيشتغل مباشرة من __init */
void __init sun6i_reset_init(void)
{
    struct device_node *np;

    /* امشي على كل node في DT يطابق "allwinner,sun6i-a31-ahb1-reset" */
    for_each_matching_node(np, sunxi_early_reset_dt_ids)
        sunxi_reset_init(np);  /* سجل الـ controller يدوياً */
}
```

ده بيخلي الـ reset controller متاح بدري — قبل ما أي driver تاني يطلبه.

---

### الـ sunxi_reset_init بالتفصيل

```c
static int sunxi_reset_init(struct device_node *np)
{
    struct reset_simple_data *data;
    struct resource res;
    resource_size_t size;

    /* 1. allocate الـ driver data */
    data = kzalloc(sizeof(*data), GFP_KERNEL);

    /* 2. اقرأ الـ MMIO range من الـ Device Tree */
    of_address_to_resource(np, 0, &res);
    size = resource_size(&res);

    /* 3. احجز الـ memory region (منع conflicts مع drivers تانية) */
    request_mem_region(res.start, size, np->name);

    /* 4. عمل ioremap — تحويل physical address لـ virtual address */
    data->membase = ioremap(res.start, size);

    /* 5. init الـ spinlock */
    spin_lock_init(&data->lock);

    /* 6. ابنِ الـ rcdev */
    data->rcdev.owner = THIS_MODULE;
    data->rcdev.nr_resets = size * 8;      /* كل بت = reset line */
    data->rcdev.ops = &reset_simple_ops;   /* الـ generic ops */
    data->rcdev.of_node = np;
    data->active_low = true;               /* Allwinner convention */

    /* 7. سجّل الـ controller في الـ framework */
    return reset_controller_register(&data->rcdev);
}
```

**الـ ioremap**: لازم تفهم إن الـ ARM processor بيشتغل بـ virtual addresses. الـ `ioremap` بتعمل mapping للـ physical MMIO address في الـ virtual address space عشان الـ CPU يقدر يوصله. ده مش مجرد casting — ده kernel virtual memory management فعلي.

---

### الـ Device Tree — كيف الـ consumer بيوصل للـ provider

في الـ DT:
```
/* Provider */
ahb1_rst: reset@01c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;  /* 12 bytes = 96 reset lines */
    #reset-cells = <1>;
};

/* Consumer */
uart0: serial@01c28000 {
    resets = <&ahb1_rst 16>;  /* reset line 16 */
};
```

الـ framework بيترجم `<&ahb1_rst 16>` لـ: "استخدم الـ `reset_controller_dev` المرتبط بـ `ahb1_rst` node، وادي الـ ops الـ ID = 16".

---

### الـ Framework بيملك إيه وبيفوّض إيه؟

| المسؤولية | الـ Framework (core) | الـ Driver (sunxi/simple) |
|---|---|---|
| OF parsing وربط consumer بـ provider | الـ framework | - |
| Reference counting للـ shared lines | الـ framework | - |
| Global list للـ controllers | الـ framework | - |
| MMIO mapping | - | Driver |
| Bit manipulation (assert/deassert) | - | `reset_simple_ops` |
| Active-low/high logic | - | Driver |
| Early boot initialization | - | `sun6i_reset_init` |
| Spinlock أثناء register access | - | Driver |

---

### تشبيه واقعي — الـ Circuit Breaker Panel

تخيل لوحة الكهرباء الرئيسية في عمارة:

| عنصر في التشبيه | المقابل في الـ Kernel |
|---|---|
| لوحة الكهرباء الرئيسية | `reset_controller_dev` |
| كل circuit breaker | reset line واحدة (bit واحدة) |
| رقم الـ breaker | reset ID |
| السكرتيرة في الاستقبال اللي بتوصّل الطلبات | Reset Controller Framework (core) |
| الفني اللي بيقطع/يوصل الكهرباء فعلاً | `reset_simple_ops` |
| عميل بيطلب "وقف الكهرباء عن الشقة 16" | Consumer driver بيعمل `reset_control_assert` |
| السكرتيرة بتترجم "شقة 16" لـ "breaker رقم 16" | `of_xlate` في الـ framework |
| لو اتنين عملاء بيشاركوا نفس الـ breaker | Shared reset line مع reference counting |
| الطوارئ — قطع الكهرباء قبل افتتاح العمارة | `sun6i_reset_init` — early init قبل الـ device model |

السكرتيرة (الـ framework) مش عارفة إزاي تقطع الكهرباء — دي وظيفة الفني (الـ driver). بس هي اللي بتعرف مين طلب، مين بيشارك مع مين، وبتمنع الـ conflicts.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

| اسم | نوعه | قيمته / تأثيره |
|-----|------|----------------|
| `active_low` | `bool` في `reset_simple_data` | لو `true`: بيـclear الـ bit عشان يـassert الـ reset (السلوك الـ Sunxi الافتراضي) |
| `status_active_low` | `bool` في `reset_simple_data` | لو `true`: الـ bit بيتقرى `0` لما الـ reset يكون مـasserted |
| `reset_us` | `unsigned int` في `reset_simple_data` | أقل delay (microseconds) بين assert وdeassert — `0` يعني مش مدعوم |
| `CONFIG_RESET_CONTROLLER` | Kconfig | لو مش معمول enable، كل دوال الـ register بتبقى inline stubs بترجع `0` |
| `GFP_KERNEL` | flag للـ `kzalloc` | allocation عادي من الـ kernel heap، ممكن ينام |
| `__initconst` | attribute | الـ data بتتحط في الـ `.init.rodata` section وبتتحرر بعد الـ boot |
| `__init` | attribute | الكود بيتحط في `.init.text` وبيتحرر بعد الـ boot |

**الـ compatible strings المدعومة (early init فقط):**

| Compatible String | الـ SoC | الـ Bus |
|-------------------|---------|---------|
| `allwinner,sun6i-a31-ahb1-reset` | Allwinner A31 | AHB1 |

---

### 1. الـ Structs المهمة

#### `struct reset_simple_data`
**الغرض:** الـ main data structure للـ driver — بتجمع كل اللي يحتاجه الـ simple reset controller في مكان واحد.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `lock` | `spinlock_t` | بيحمي الـ read-modify-write على الـ MMIO registers |
| `membase` | `void __iomem *` | الـ base address للـ memory-mapped registers بعد الـ ioremap |
| `rcdev` | `struct reset_controller_dev` | الـ core reset controller object — **embedded** مش pointer |
| `active_low` | `bool` | `true` للـ Sunxi: clear bit = assert reset |
| `status_active_low` | `bool` | مش بيتستخدم في الـ sunxi-reset.c مباشرةً لكن موجود في الـ struct |
| `reset_us` | `unsigned int` | مش بيتـset في الـ sunxi-reset.c (بيفضل `0`) |

**العلاقة مع باقي الـ structs:**
- الـ `rcdev` embedded جوا `reset_simple_data` — الـ kernel بيشتغل مع `rcdev` بس، والـ driver يـrecover الـ `reset_simple_data` بـ `container_of`.
- الـ `rcdev.ops` بيشاور على `reset_simple_ops` — الـ ops اللي فيها الـ assert/deassert الفعلي.
- الـ `rcdev.of_node` بيشاور على الـ Device Tree node اللي جاي منها الـ controller.

---

#### `struct reset_controller_dev`
**الغرض:** الـ core kernel structure اللي بتمثل أي reset controller — الـ reset core بيشتغل معاها.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `ops` | `const struct reset_control_ops *` | الـ vtable: reset, assert, deassert, status |
| `owner` | `struct module *` | الـ kernel module اللي بيمتلك الـ controller |
| `list` | `struct list_head` | الـ controller بيتضاف في global list جوا الـ reset core |
| `reset_control_head` | `struct list_head` | list بالـ reset controls اللي تم request منه |
| `dev` | `struct device *` | الـ device المرتبطة (مش بيتـset في الـ early init path) |
| `of_node` | `struct device_node *` | الـ DT node — الـ consumers بيستخدموه للـ lookup |
| `of_args` | `const struct of_phandle_args *` | للـ GPIO-based controllers (مش مستخدم هنا) |
| `of_reset_n_cells` | `int` | عدد الـ cells في الـ reset specifier |
| `of_xlate` | function pointer | بتحول الـ DT specifier لـ reset ID (الـ default هو `of_reset_simple_xlate`) |
| `nr_resets` | `unsigned int` | عدد الـ reset lines = `size * 8` (كل byte = 8 bits = 8 resets) |

---

#### `struct reset_control_ops`
**الغرض:** الـ vtable (operations table) اللي الـ reset core بيستدعيها لما consumer يطلب reset.

| الـ Callback | الـ Signature | الوظيفة |
|--------------|--------------|---------|
| `reset` | `(rcdev, id) → int` | self-deasserting reset دفعة واحدة |
| `assert` | `(rcdev, id) → int` | يـassert الـ reset line يدوياً |
| `deassert` | `(rcdev, id) → int` | يـdeassert الـ reset line يدوياً |
| `status` | `(rcdev, id) → int` | يقرى الحالة الحالية للـ reset line |

في الـ Sunxi: الـ `reset_simple_ops` (معرّفة في `reset-simple.c`) هي اللي بتـimplements الـ callbacks دي.

---

#### `struct resource`
**الغرض:** بيمثل الـ memory region اللي الـ controller بيحتاجها (من الـ DT).

| الـ Field | الوصف |
|-----------|-------|
| `start` | الـ physical base address |
| `end` | الـ physical end address |
| `name` | اسم الـ region (من `np->name`) |

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
+----------------------------------+
|      reset_simple_data           |
|                                  |
|  lock: spinlock_t                |
|  membase: void __iomem *─────────────► [MMIO Registers]
|  active_low: true                |
|  status_active_low: bool         |
|  reset_us: uint                  |
|                                  |
|  +----------------------------+  |
|  |  reset_controller_dev     |  |
|  |  (rcdev) [embedded]       |  |
|  |                            |  |
|  |  ops ──────────────────────────► reset_simple_ops
|  |  owner: THIS_MODULE        |  |   .assert()
|  |  nr_resets: size*8         |  |   .deassert()
|  |  of_node ──────────────────────► device_node (DT)
|  |  list ──────────────────────────► [global rcdev list]
|  |  reset_control_head ────────────► [per-consumer list]
|  |  dev: NULL (early init)    |  |
|  +----------------------------+  |
+----------------------------------+

        container_of(rcdev, reset_simple_data, rcdev)
        ◄─────────────────────────────────────────────
        (الـ reset_simple_ops بتستخدمها عشان توصل للـ lock والـ membase)
```

---

### 3. مخطط الـ Lifecycle

```
BOOT TIME (early __init)
│
▼
sun6i_reset_init()
│  يلف على كل DT node بـ compatible "allwinner,sun6i-a31-ahb1-reset"
│
▼
sunxi_reset_init(np)
│
├─► kzalloc(reset_simple_data)        [CREATION]
│       ↓ فشل → -ENOMEM
│
├─► of_address_to_resource(np)        [DT PARSE]
│       ↓ فشل → goto err_alloc
│
├─► request_mem_region(start, size)   [RESOURCE CLAIM]
│       ↓ فشل → -EBUSY → goto err_alloc
│
├─► ioremap(start, size)              [MAP TO VIRTUAL]
│       → data->membase
│       ↓ فشل → -ENOMEM → goto err_alloc
│
├─► spin_lock_init(&data->lock)       [LOCK INIT]
│
├─► rcdev.nr_resets = size * 8        [CONFIG]
│   rcdev.ops = &reset_simple_ops
│   rcdev.of_node = np
│   rcdev.owner = THIS_MODULE
│   data->active_low = true
│
├─► reset_controller_register(rcdev)  [REGISTRATION]
│       → بيضيف في global list
│       → consumers ممكن يبدأوا يعملوا lookup
│
▼
SYSTEM RUNNING
│  consumers بيعملوا reset_control_get() → reset_assert() / reset_deassert()
│
▼
TEARDOWN
   مش موجود في الـ early init path!
   (الـ .init sections بتتحرر لكن الـ data نفسها بتفضل في الـ memory)
   الـ reset_controller_unregister() مش بتتعمل call هنا.

err_alloc:
   kfree(data) ← بس لو الـ ioremap اتعمل، الـ membase بيـleak!
   (ده bug موجود في الـ code الأصلي)
```

---

### 4. مخطط الـ Call Flow

#### 4.1 الـ Early Init Flow

```
start_kernel()  [init/main.c]
  └─► ... early init hooks ...
        └─► sun6i_reset_init()        [reset/reset-sunxi.c]
              └─► for_each_matching_node(np, sunxi_early_reset_dt_ids)
                    └─► sunxi_reset_init(np)
                          ├─► kzalloc()
                          ├─► of_address_to_resource(np, 0, &res)
                          │     └─► [يقرى reg property من الـ DT]
                          ├─► request_mem_region(res.start, size, np->name)
                          │     └─► [يحجز الـ physical memory range]
                          ├─► ioremap(res.start, size)
                          │     └─► [يرسم على الـ virtual address space]
                          ├─► spin_lock_init(&data->lock)
                          ├─► [يملا حقول الـ rcdev]
                          └─► reset_controller_register(&data->rcdev)
                                └─► [يضيف في global rcdev list]
                                └─► [consumers ممكن يبدأوا يشتغلوا]
```

#### 4.2 الـ Consumer Reset Flow (بعد الـ registration)

```
consumer driver calls reset_control_assert(rstc)
  └─► reset_core: يجيب الـ rcdev من الـ rstc
        └─► rcdev->ops->assert(rcdev, id)
              └─► reset_simple_ops.assert(rcdev, id)
                    └─► container_of(rcdev, reset_simple_data, rcdev)
                          └─► spin_lock_irqsave(&data->lock, flags)
                                └─► reg = readl(data->membase + reg_offset)
                                      └─► [active_low=true] reg &= ~BIT(bit_offset)
                                            └─► writel(reg, data->membase + reg_offset)
                                                  └─► spin_unlock_irqrestore(&data->lock, flags)
```

---

### 5. استراتيجية الـ Locking

#### الـ Lock المستخدم: `spinlock_t lock` في `reset_simple_data`

| السؤال | الإجابة |
|--------|---------|
| ما اللي بيحميه؟ | الـ read-modify-write cycles على الـ MMIO registers |
| ليه spinlock وmutex? | الـ reset ops ممكن تتعمل call من interrupt context |
| الـ variant المستخدم | `spin_lock_irqsave` / `spin_unlock_irqrestore` في `reset_simple_ops` (في reset-simple.c) |
| الـ init | `spin_lock_init(&data->lock)` في `sunxi_reset_init` |

#### ليه `irqsave`؟

```
لو interrupt جه أثناء read-modify-write:
  read reg   ← interrupt handler بيعمل assert على reset تاني
    modify
      write  ← الـ interrupt handler's write اتـoverwrite!

الحل: spin_lock_irqsave بيعمل disable للـ interrupts على الـ CPU الحالي
أثناء الـ critical section
```

#### ترتيب الـ Locks (Lock Ordering)

الـ driver ده بسيط — فيه lock واحد بس (`data->lock`). مفيش نested locking، مفيش احتمال deadlock من ناحية الـ driver نفسه.

الـ reset core (في `reset-controller.c`) ممكن يمسك lock خاص بيه قبل ما يستدعي الـ ops — الـ `data->lock` بيتمسك جوا الـ ops بس، فالترتيب دايماً:

```
reset_core_lock (إن وجد)
  └─► data->lock  (spinlock, irqsave)
```

مفيش عكس لحد دلوقتي في الـ codebase ده.
## Phase 4: شرح الـ Functions

### ملخص الـ Functions والـ APIs

| Function / API | النوع | الغرض |
|---|---|---|
| `sunxi_reset_init` | `static int` | تهيئة reset controller واحد من الـ device tree node |
| `sun6i_reset_init` | `void __init` | early-boot entry point — تتمشى على كل matching nodes وتستدعي `sunxi_reset_init` |
| `kzalloc` | kernel API | تخصيص ذاكرة مصفّرة لـ `reset_simple_data` |
| `of_address_to_resource` | OF API | تحويل `reg` property في الـ DT لـ `struct resource` |
| `resource_size` | kernel macro | حساب حجم الـ MMIO region |
| `request_mem_region` | kernel API | حجز الـ physical memory range في `/proc/iomem` |
| `ioremap` | kernel API | map الـ physical registers لـ virtual address |
| `spin_lock_init` | kernel macro | تهيئة الـ spinlock لحماية read-modify-write |
| `reset_controller_register` | reset API | تسجيل الـ `reset_controller_dev` في الـ reset subsystem |
| `for_each_matching_node` | OF macro | iteration على كل DT nodes اللي تطابق الـ `of_device_id` table |

---

### ### التصنيف المنطقي للـ Functions

```
┌─────────────────────────────────────────────┐
│  Early-Boot Entry                           │
│    sun6i_reset_init()                       │
│         │                                   │
│         ▼ (for each matching DT node)       │
│  Core Initialization                        │
│    sunxi_reset_init()                       │
│      ├── kzalloc()                          │
│      ├── of_address_to_resource()           │
│      ├── request_mem_region()               │
│      ├── ioremap()                          │
│      ├── spin_lock_init()                   │
│      └── reset_controller_register()        │
└─────────────────────────────────────────────┘
```

---

### ### المجموعة الأولى: Early Initialization

الهدف من المجموعة دي هو تشغيل الـ reset controllers في أبكر وقت ممكن خلال الـ boot — قبل ما الـ driver model يتجهّز. في الـ Allwinner sun6i (A31)، الـ AHB1 reset controller لازم يتسجّل early لأن أجهزة تانية كتير بتحتاجه عشان تتهيّأ.

---

#### `sun6i_reset_init`

```c
void __init sun6i_reset_init(void)
```

**ما بتعمله:**
الـ entry point الوحيد اللي بيتكلّم من الـ early boot code. بتعمل iteration على كل الـ device tree nodes اللي compatible مع `"allwinner,sun6i-a31-ahb1-reset"` وبتستدعي `sunxi_reset_init` لكل واحدة. الغرض هو تسجيل الـ reset controllers قبل ما أي device driver يتشال.

**Parameters:** لا يوجد — بتقرأ الـ DT مباشرة.

**Return value:** `void` — أي error في التهيئة بيترجعه الـ `sunxi_reset_init` ولكن بيتجاهل هنا (early boot tradeoff).

**Key details:**
- الـ `__init` annotation معناها الكود ده بيتحذف من الذاكرة بعد انتهاء الـ initialization.
- الـ `__initconst` على `sunxi_early_reset_dt_ids` بيخلّي الـ table في `.init.rodata` section — تتحذف برضو بعد الـ boot.
- بيتستدعى من `arch/arm/mach-sunxi/` أو من `early_initcall` — **مش** من الـ platform driver model.
- لو فشل `sunxi_reset_init` لأحد الـ nodes، الـ loop بتكمّل للـ node الجاي.

**Who calls it:** الـ SoC-specific machine init code أو `early_initcall`.

---

### ### المجموعة التانية: Core Initialization

---

#### `sunxi_reset_init`

```c
static int sunxi_reset_init(struct device_node *np)
```

**ما بتعمله:**
الـ function دي هي قلب الـ driver — بتعمل كل خطوات تهيئة الـ reset controller لـ node واحدة من الـ DT. بتخصص الـ `reset_simple_data`، بتعمل map للـ MMIO registers، بتهيّئ الـ spinlock، وبتسجّل الـ controller مع الـ reset subsystem.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node للـ reset controller — مصدر كل المعلومات (reg, compatible, name) |

**Return value:**
- `0` عند النجاح.
- `-ENOMEM` لو فشل `kzalloc` أو `ioremap`.
- `-EBUSY` لو الـ memory region محجوزة لـ driver تاني.
- أي error code من `of_address_to_resource` لو الـ `reg` property غلط.

**Key details:**

1. **Memory allocation** — بتستخدم `kzalloc(GFP_KERNEL)` عشان الـ context هنا ليس atomic (early boot لكن قبل scheduling). الـ zero-fill مهم لأن `reset_simple_data` فيها fields زي `status_active_low` و`reset_us` مش بتتعمل-set صريح.

2. **MMIO mapping** — `of_address_to_resource` بتحوّل الـ `reg` property لـ `struct resource` فيها الـ physical base address والـ size. بعدين `request_mem_region` بتحجز الـ range في `iomem_resource` tree عشان تمنع conflict. لو المنطقة محجوزة، بتفشل بـ `-EBUSY` وبتعمل `goto err_alloc`.

3. **nr_resets calculation** — بيحسبها كـ `size * 8` (عدد البايتات × 8 = عدد الـ bits). كل bit في الـ register هو reset line مستقلة. ده يعني driver مش محتاج يعرف التفاصيل — بيشتغل generic.

4. **active_low = true** — في Allwinner hardware، الـ reset active-low يعني: لمّا الـ bit = 0 → الجهاز في حالة reset. لمّا الـ bit = 1 → الجهاز يشتغل عادي. الـ `reset_simple_ops` بيأخذ ده في الاعتبار لما بيعمل assert/deassert.

5. **لا يوجد `iounmap` أو `release_mem_region`** في الـ error path بعد `ioremap` — لأن لو وصلنا لـ `err_alloc` من `ioremap`، معناها الـ `request_mem_region` نجحت، فلازم يتعمل release. لكن في الكود الحالي **ده bug محتمل** في الـ error path: لو `ioremap` فشلت، الـ memory region محجوزة ومش بتتحرر. ده tradeoff قبول في early-boot code لأن الـ system بيـ panic أصلاً.

6. **لا cleanup function** — الـ driver ده مش بيدعم `remove` لأن الـ early reset controllers بتفضل موجودة طول عمر الـ kernel.

**Pseudocode flow:**

```
sunxi_reset_init(np):
    data = kzalloc(sizeof(reset_simple_data))
    if !data → return -ENOMEM

    ret = of_address_to_resource(np, index=0, &res)
    if ret → goto err_alloc

    size = resource_size(&res)   // res.end - res.start + 1

    if !request_mem_region(res.start, size, np->name):
        ret = -EBUSY
        goto err_alloc

    data->membase = ioremap(res.start, size)
    if !data->membase:
        ret = -ENOMEM
        goto err_alloc          // BUG: mem_region not released here

    spin_lock_init(&data->lock)

    data->rcdev.owner    = THIS_MODULE
    data->rcdev.nr_resets = size * 8
    data->rcdev.ops      = &reset_simple_ops   // from reset-simple driver
    data->rcdev.of_node  = np
    data->active_low     = true

    return reset_controller_register(&data->rcdev)

err_alloc:
    kfree(data)
    return ret
```

**Who calls it:** فقط `sun6i_reset_init` — مش مكشوف للـ driver model.

---

### ### الـ Data Structures المستخدمة

#### `struct reset_simple_data`

```c
struct reset_simple_data {
    spinlock_t              lock;           /* protects RMW on registers */
    void __iomem            *membase;       /* virtual base of MMIO region */
    struct reset_controller_dev rcdev;      /* embedded — passed to reset subsystem */
    bool                    active_low;     /* 0=assert, 1=deassert (Allwinner style) */
    bool                    status_active_low;
    unsigned int            reset_us;       /* min delay between assert/deassert */
};
```

الـ `rcdev` embedded مش pointer — ده pattern شائع في الـ kernel عشان يسمح بـ `container_of` من الـ ops callbacks.

#### `struct reset_controller_dev`

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;   /* assert/deassert/reset/status */
    struct module          *owner;
    struct list_head        list;          /* linked into global reset controller list */
    struct list_head        reset_control_head;
    struct device          *dev;           /* NULL هنا لأنه early init */
    struct device_node     *of_node;       /* DT node للـ phandle lookup */
    int                     of_reset_n_cells;
    int (*of_xlate)(...);                  /* default: of_reset_simple_xlate */
    unsigned int            nr_resets;
};
```

**ملاحظة مهمة:** في الـ early initialization، الـ `rcdev.dev` = `NULL` لأن مفيش `struct device` متاح. الـ `of_node` بيعوّض عن ده للـ DT lookups.

---

### ### الـ APIs الخارجية المستخدمة

#### `of_address_to_resource`

```c
int of_address_to_resource(struct device_node *dev, int index,
                           struct resource *r);
```

بتقرأ الـ `reg` property من الـ DT node وبتملّأ `struct resource` بالـ physical address والـ size. الـ `index` = 0 يعني أول `reg` entry. بترجع 0 عند النجاح أو error code لو الـ property مش موجودة أو غلطانة.

---

#### `request_mem_region`

```c
struct resource *request_mem_region(resource_size_t start,
                                    resource_size_t n,
                                    const char *name);
```

بتسجّل memory region في `/proc/iomem` وبتمنع أي driver تاني من حجز نفس الـ range. بترجع `NULL` لو المنطقة محجوزة مسبقاً (و الكود بيعامل ده كـ `-EBUSY`). الـ `name` = `np->name` بيظهر في `/proc/iomem`.

---

#### `reset_controller_register`

```c
int reset_controller_register(struct reset_controller_dev *rcdev);
```

بتضيف الـ `rcdev` لـ global list في الـ reset subsystem. بعد ما بترجع، أي consumer ممكن يعمل `reset_control_get` ويلاقي الـ controller ده عن طريق الـ `of_node` phandle. بتضبط الـ `of_xlate` لـ `of_reset_simple_xlate` لو كانت NULL. بترجع 0 عند النجاح.

---

### ### الـ `sunxi_early_reset_dt_ids` Table

```c
static const struct of_device_id sunxi_early_reset_dt_ids[] __initconst = {
    { .compatible = "allwinner,sun6i-a31-ahb1-reset", },
    { /* sentinel */ },
};
```

الـ table دي بتحدد أي DT nodes بتتهيّأ في الـ early boot. في الوقت الحالي فيها entry واحدة بس للـ A31 AHB1 reset controller. الـ `for_each_matching_node` بتمشي عليها وبتعمل iteration على الـ DT كله دورة واحدة.

الفرق بين الـ early registration دي والـ platform driver registration:

| الجانب | Early (sunxi_reset_init) | Platform Driver |
|---|---|---|
| التوقيت | قبل الـ driver model | بعد الـ device enumeration |
| الـ `rcdev.dev` | NULL | pointer لـ `struct device` |
| الـ cleanup | لا يوجد | `reset_controller_unregister` |
| الـ memory management | manual `kzalloc/kfree` | devm_* |
| الاستخدام | AHB1 reset فقط | باقي الـ Allwinner controllers |
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده هو `reset-sunxi.c` — بيعمل early initialization لـ reset controller على Allwinner SoCs (تحديداً sun6i/A31). الـ driver بسيط جداً: بياخد memory-mapped register range من الـ Device Tree، وبيسجّل `reset_controller_dev` باستخدام `reset_simple_ops` مع `active_low = true`.

---

### Software Level

#### 1. debugfs

الـ reset subsystem بيعرض معلومات عبر `debugfs`:

```bash
# mount debugfs لو مش موجود
mount -t debugfs debugfs /sys/kernel/debug

# شوف كل الـ reset controllers المسجّلين
cat /sys/kernel/debug/reset/reset_controllers

# شوف الـ resets اللي متطلبها consumers دلوقتي
cat /sys/kernel/debug/reset/reset_controls
```

**مثال على output:**

```
reset_controllers:
  rcdev[0]: sun6i-a31-ahb1-reset (nr_resets=96, of_node=ahb1-reset@1c202000)
    reset_controls:
      id=12 (asserted=0, triggered=1, shared=0)
      id=17 (asserted=1, triggered=0, shared=1)
```

- `asserted=1` يعني الـ device لسّه في reset.
- `nr_resets = size * 8` — الـ driver بيحسبها من حجم الـ memory region.

---

#### 2. sysfs

الـ driver ده `early init` — بيتسجّل قبل الـ platform device model، فمفيش `sysfs` device node مباشر ليه. لكن ممكن تتحقق من الـ memory region:

```bash
# شوف الـ memory regions المحجوزة
cat /proc/iomem | grep -i reset

# مثال output:
# 01c202000-01c2020ff : ahb1-reset@1c202000
```

لو المنطقة دي مش ظاهرة، يبقى `request_mem_region` فشل أو الـ DT entry غلط.

---

#### 3. ftrace — Tracepoints

```bash
# فعّل tracing للـ reset subsystem
cd /sys/kernel/debug/tracing

# Enable reset events
echo 1 > events/reset/enable
echo 1 > tracing_on

# أو بشكل أدق
echo 1 > events/reset/reset_assert/enable
echo 1 > events/reset/reset_deassert/enable
echo 1 > events/reset/reset_reset/enable

# اقرأ الـ trace
cat trace
```

**مثال output:**

```
kworker/0:1-42    [000] ....   12.345678: reset_assert: id=17
kworker/0:1-42    [000] ....   12.345700: reset_deassert: id=17
```

لو شايف `reset_assert` بس من غير `reset_deassert` → الـ device عالق في reset.

```bash
# Function tracer على الـ sunxi_reset_init
echo function > current_tracer
echo sunxi_reset_init > set_ftrace_filter
echo 1 > tracing_on
# شغّل البورد وشوف
cat trace
```

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug للـ reset subsystem كله
echo "file reset-sunxi.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file reset-simple.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file reset-controller.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ reset module
echo "module reset_sunxi +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep reset
```

**في kernel cmdline (للـ early boot debugging):**

```
dyndbg="file reset-sunxi.c +p; file reset-simple.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_RESET_CONTROLLER` | لازم يكون `=y` أصلاً عشان الـ driver يشتغل |
| `CONFIG_DEBUG_KERNEL` | يفتح debugging options عامة |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف bugs في `spin_lock` اللي الـ driver بيستخدمه |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكتشف لو حد نادى `spin_lock` في context ممنوع |
| `CONFIG_PROVE_LOCKING` | lockdep — يكتشف deadlocks في الـ spinlock |
| `CONFIG_DEBUG_MEMORY_INIT` | يكتشف مشاكل في `kzalloc` |
| `CONFIG_KASAN` | يكتشف memory corruption بعد `kzalloc` |
| `CONFIG_OF` | لازم يكون `=y` عشان `of_address_to_resource` تشتغل |
| `CONFIG_IOREMAP_PROT` | بيساعد في debug الـ ioremap issues |
| `CONFIG_EARLY_PRINTK` | لو محتاج `printk` قبل console init |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(RESET|DEBUG_SPIN|PROVE_LOCK|KASAN)"
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# عرض كل الـ reset controllers عبر سطر الأوامر
ls -la /sys/bus/platform/drivers/ | grep reset

# لو الـ driver اتسجّل كـ platform device (مش الحالة هنا بس للمقارنة)
# devlink مش مناسب هنا — الـ driver ده pure reset controller بدون netdev

# استخدم reset-gpio أو reset-simple tools للتحقق
cat /proc/iomem | grep -A1 "ahb1-reset"

# شوف كل الـ memory regions المحجوزة
cat /proc/iomem
```

---

#### 7. جدول Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `of_address_to_resource failed` | الـ DT node مش عنده `reg` property | اتحقق من `.dtsi` وتأكد من `reg = <addr size>` |
| `unable to request mem region` (`-EBUSY`) | منطقة الـ memory محجوزة مسبقاً من driver تاني | تحقق من `cat /proc/iomem` — فيه conflict |
| `ioremap failed` | الـ kernel مش قادر يعمل virtual mapping للعنوان | تأكد إن الـ physical address صح ومش خارج النطاق |
| `kzalloc failed` (`-ENOMEM`) | ما فيش memory كافية | تحقق من `dmesg | grep oom` — ممكن OOM killer اشتغل |
| `reset_controller_register failed` | مشكلة في تسجيل الـ rcdev | تحقق إن `CONFIG_RESET_CONTROLLER=y` |
| `reset ID X out of range` | consumer طلب reset ID أكبر من `nr_resets` | الـ DT specifier للـ consumer بيشاور على index خاطئ |
| `reset already in use` | نفس الـ reset متطلوب من أكتر من consumer بدون `shared` | استخدم `reset-names` صح في الـ consumer DT |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
static int sunxi_reset_init(struct device_node *np)
{
    struct reset_simple_data *data;
    struct resource res;
    resource_size_t size;
    int ret;

    data = kzalloc(sizeof(*data), GFP_KERNEL);
    if (!data) {
        /* WARN هنا مفيد لأن early init فشل — ممكن يأثر على كل الـ system */
        WARN(1, "sunxi-reset: failed to alloc data for %s\n", np->name);
        return -ENOMEM;
    }

    ret = of_address_to_resource(np, 0, &res);
    if (ret) {
        /* dump_stack هنا لو عايز تعرف من نادى sun6i_reset_init */
        dump_stack();
        goto err_alloc;
    }

    size = resource_size(&res);

    /* WARN لو الـ size مش متوقع — مثلاً 0 أو أكبر من 0x1000 */
    WARN_ON(size == 0 || size > SZ_4K);

    if (!request_mem_region(res.start, size, np->name)) {
        /* هنا مهم تعرف مين حاجز الـ region */
        pr_err("sunxi-reset: mem region %pa busy, dumping iomem:\n",
               &res.start);
        /* dump_stack لمعرفة call chain */
        dump_stack();
        ret = -EBUSY;
        goto err_alloc;
    }

    data->membase = ioremap(res.start, size);
    if (!data->membase) {
        WARN(1, "sunxi-reset: ioremap failed for %pa\n", &res.start);
        ret = -ENOMEM;
        goto err_alloc;
    }

    /* WARN_ON لو nr_resets = 0 — يعني size = 0 وده مش منطقي */
    WARN_ON(data->rcdev.nr_resets == 0);

    return reset_controller_register(&data->rcdev);

err_alloc:
    kfree(data);
    return ret;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State مطابق للـ Kernel State

الـ driver بيستخدم `active_low = true`، يعني:
- **bit = 0** → الـ device في reset (asserted)
- **bit = 1** → الـ device شغّال (deasserted)

```bash
# اقرأ الـ AHB1 reset register مباشرة
# عنوان الـ AHB1 reset على sun6i/A31: 0x01C202000

# باستخدام devmem2
devmem2 0x01C202000 w   # اقرأ الـ word الأول (resets 0-31)
devmem2 0x01C202004 w   # resets 32-63
devmem2 0x01C202008 w   # resets 64-95

# مثال output:
# Value at address 0x01C202000 (0xb6f58000): 0xFFFFFFE7
# bit N = 0 يعني reset N لسّه asserted
# bit N = 1 يعني reset N تم deassert
```

قارن الـ value دي بالـ `reset_controls` في `debugfs` — لو الـ kernel قال إن reset 3 deasserted بس الـ register bit 3 = 0، في bug في الـ `active_low` logic.

---

#### 2. Register Dump Techniques

```bash
# باستخدام /dev/mem (لو CONFIG_STRICT_DEVMEM=n)
dd if=/dev/mem bs=4 count=24 skip=$((0x01C202000/4)) 2>/dev/null | xxd

# باستخدام io utility (من package io أو busybox)
io -4 -r 0x01C202000

# باستخدام devmem2 (أسهل)
for offset in 0 4 8 12 16 20; do
    addr=$((0x01C202000 + offset))
    printf "Offset +%02d: " $offset
    devmem2 $(printf "0x%X" $addr) w 2>/dev/null | grep "Value"
done
```

**تفسير النتيجة:**

```
Offset +00: Value: 0xFFFFFFE7
# Binary: 1111 1111 1111 1111 1111 1111 1110 0111
# bit 3 = 0 → reset 3 asserted (device في reset)
# bit 4 = 0 → reset 4 asserted
# كل الـ bits التانية = 1 → deasserted (شغّالة)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو عندك وصول للـ hardware pins:

- **نقاط القياس**: شوف الـ reset line الفعلية للـ peripheral اللي بتـ debug فيه (مش الـ register) — ابحث في الـ schematic على الـ RST# pin.
- **التوقيت المتوقع**: بعد boot، الـ `sun6i_reset_init` بيتنادى في `__init` → يعني الـ reset deassert بيحصل بدري جداً.
- **الـ active_low**: على الـ oscilloscope، الـ reset line المفروض تبدأ LOW (asserted) وترفع لـ HIGH بعد الـ init.
- **Logic levels**: تأكد إن الـ VDD صح — الـ A31 بيشتغل على 1.8V I/O في بعض الـ pins.

```
الـ reset line timeline:
  ____                    _______________________
      |__________________| deassert بعد init
  RESET ASSERTED (LOW)   NORMAL OPERATION (HIGH)
  ^boot                  ^sun6i_reset_init بيتنادى
```

---

#### 4. Hardware Issues الشائعة → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ dmesg | التشخيص |
|---|---|---|
| الـ reg address في DT غلط | `of_address_to_resource failed` | راجع `reg` property في `.dtsi` |
| الـ memory region مش في أي memory map | `ioremap failed` | تحقق من الـ SoC memory map في الـ datasheet |
| الـ clock للـ AHB1 bus مش اتفعّل | consumer devices فشلت رغم إن reset OK | تحقق من الـ clock controller قبل الـ reset controller |
| مشكلة في VDD للـ peripheral | reset deassert OK بس device مش بيشتغل | قيس الـ voltage على الـ peripheral power rail |
| short circuit على الـ reset line | الـ register value بيتكتب بس ما بيتغيرش | فحص الـ PCB للـ shorts |

---

#### 5. Device Tree Debugging

```bash
# اتحقق إن الـ DT node اتعرف صح
ls /proc/device-tree/ | grep reset
cat /proc/device-tree/ahb1-reset@1c202000/compatible
# المفروض يطلع: allwinner,sun6i-a31-ahb1-reset

# اتحقق من الـ reg property
xxd /proc/device-tree/ahb1-reset@1c202000/reg
# المفروض تشوف العنوان والحجم encoded كـ big-endian 32-bit values
# مثال: 01 c2 02 00 00 00 01 00
# يعني start=0x01C202000, size=0x100 (256 bytes = 2048 resets bits)

# اتحقق إن الـ phandle صح في الـ consumer
cat /proc/device-tree/soc/usb@1c14000/resets
xxd /proc/device-tree/soc/usb@1c14000/resets

# اتحقق من الـ DT compiled version
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "ahb1-reset"
```

**DT entry الصح لـ sun6i:**

```dts
ahb1_rst: reset@1c202000 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202000 0x100>;  /* 256 bytes = 2048 reset bits */
    #reset-cells = <1>;
};

/* consumer example */
usb_host0: usb@1c14000 {
    resets = <&ahb1_rst 22>;   /* AHB1 reset ID 22 */
    reset-names = "ahb";
};
```

لو `#reset-cells = <1>` مش موجودة، الـ consumer مش هيقدر يطلب الـ reset ويطلع error في `dmesg`.

---

### Practical Commands

#### Shell Commands جاهزة للنسخ

```bash
# ============================================================
# 1. تحقق إن الـ driver اشتغل في الـ early boot
# ============================================================
dmesg | grep -i "sunxi\|sun6i\|ahb1-reset\|reset"

# ============================================================
# 2. شوف كل الـ reset controllers المسجّلين
# ============================================================
cat /sys/kernel/debug/reset/reset_controllers 2>/dev/null || \
    echo "debugfs مش mounted أو CONFIG_RESET_CONTROLLER=n"

# ============================================================
# 3. شوف الـ memory region المحجوزة للـ driver
# ============================================================
grep -i "ahb1-reset\|1c202" /proc/iomem

# ============================================================
# 4. اقرأ الـ reset registers مباشرة
# ============================================================
BASE=0x01C202000
for i in 0 4 8; do
    ADDR=$((BASE + i))
    echo -n "Reg offset +$i (resets $((i*8))-$(((i+1)*8-1))): "
    devmem2 $(printf "0x%08X" $ADDR) w 2>&1 | grep -oP '0x[0-9A-Fa-f]+'
done

# ============================================================
# 5. فعّل ftrace للـ reset events
# ============================================================
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/reset/enable
echo 1 > tracing_on
# ... شغّل العملية اللي بتـ debug فيها ...
cat trace | grep reset
echo 0 > tracing_on

# ============================================================
# 6. فعّل dynamic debug
# ============================================================
echo "file reset-sunxi.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file reset-simple.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &   # شوف الـ output في real time

# ============================================================
# 7. اتحقق من الـ DT node
# ============================================================
find /proc/device-tree -name "compatible" -exec sh -c \
    'val=$(cat "$1" 2>/dev/null); echo "$1: $val"' _ {} \; 2>/dev/null | \
    grep -i "allwinner\|sun6i"

# ============================================================
# 8. شوف أي resets asserted (bit = 0 في active_low mode)
# ============================================================
python3 -c "
import struct
# قرأ القيمة من devmem2 قبل كده
regs = [0xFFFFFFE7, 0xFFFFFFFF, 0xFFFFFFFF]  # استبدلها بالقيم الحقيقية
for reg_idx, val in enumerate(regs):
    for bit in range(32):
        if not (val >> bit & 1):  # active_low: 0 = asserted
            reset_id = reg_idx * 32 + bit
            print(f'Reset ID {reset_id} is ASSERTED (device in reset)')
"
```

**مثال على output وتفسيره:**

```
# dmesg output عند الـ boot الناجح:
[    0.000123] sunxi-reset: registered 2048 resets for ahb1-reset@1c202000

# dmesg output عند فشل الـ DT:
[    0.000123] sunxi-reset: of_address_to_resource failed: -22
# التفسير: -22 = -EINVAL → الـ reg property مش موجود أو format غلط

# dmesg output عند conflict:
[    0.000200] sunxi-reset: can't request mem region 01c202000-01c2020ff
# التفسير: driver تاني حاجز نفس المنطقة قبله

# ftrace output:
kworker/u4:0-89  [001] d...  5.123456: reset_deassert: rcdev=ahb1-reset@1c202000 id=22
# التفسير: الـ USB reset (id=22) اتعمل deassert في الوقت الصح
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بورد Allwinner H616 — Android TV Box — الـ USB مش شغال من أول boot

#### العنوان
**الـ USB controller مش بيشتغل على TV box بـ H616 بسبب reset line مش متحررة**

#### السياق
شركة بتبني Android TV box بـ **Allwinner H616** SoC. البورد بيبوت، الـ Android بيطلع على الشاشة، بس الـ USB host ports مش بتشتغل خالص — لا mouse، لا keyboard، لا USB storage. الـ `dmesg` بيظهر الـ USB driver اتسجل بس الـ devices مش بتتعرف.

#### المشكلة
الـ USB controller محتاج reset صح قبل ما يشتغل. الـ reset controller الخاص بالـ AHB1 bus (اللي الـ USB بيعدي عليه) لازم يتسجل **early** قبل ما الـ USB driver يحاول يشتغل. لو `sun6i_reset_init()` ماتشتغلتش في الوقت الصح، الـ reset line فاضلة في حالة assert (بسبب `active_low = true`)، وده بيخلي الـ USB controller محجوز في reset.

#### التحليل
في `reset-sunxi.c`:

```c
static const struct of_device_id sunxi_early_reset_dt_ids[] __initconst = {
    { .compatible = "allwinner,sun6i-a31-ahb1-reset", },
    { /* sentinel */ },
};

void __init sun6i_reset_init(void)
{
    struct device_node *np;

    for_each_matching_node(np, sunxi_early_reset_dt_ids)
        sunxi_reset_init(np);  // بيسجل الـ reset controller early
}
```

جوه `sunxi_reset_init()`:

```c
data->active_low = true;
// يعني bit=0 → reset مضغوط، bit=1 → reset متحرر
// الـ USB محتاج deassert (bit=1) عشان يشتغل
```

الـ `nr_resets` بيتحسب كده:

```c
data->rcdev.nr_resets = size * 8;
// size = resource_size(&res) — حجم الـ register block بالبايت
// كل bit = reset line واحدة
```

لو الـ DT node مش موجود أو الـ compatible string غلط، `for_each_matching_node` مش هتلاقي حاجة، و`sun6i_reset_init()` هترجع من غير ما تعمل حاجة.

#### الحل

**أولاً: اتحقق إن الـ DT node موجود وصح:**

```bash
# على البورد
grep -r "ahb1-reset" /proc/device-tree/ 2>/dev/null
# أو
dtc -I fs /proc/device-tree | grep -A5 "ahb1-reset"
```

**ثانياً: DT المفروض يكون كده:**

```dts
ahb1_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;
    #reset-cells = <1>;
};
```

**ثالثاً: تأكد إن `sun6i_reset_init()` اتكلمت من `arch/arm/mach-sunxi/` أو من `arch/arm64/` early init:**

```c
// في mach init أو early_initcall
void __init sunxi_dt_init(void)
{
    sun6i_reset_init();  // لازم يكون هنا قبل أي driver
    ...
}
```

**رابعاً: اتحقق من الـ reset state:**

```bash
# قرا الـ register مباشرة (AHB1 reset base على H616)
devmem 0x01c202c0 32
# لو القيمة 0x00000000 → كل الـ resets مضغوطة (active_low=true)
# المفروض تشوف bits مضروبة للـ USB
```

#### الدرس المستفاد
الـ `__init` و`__initconst` مش مجرد تحسين — ده بيحدد **متى** بالظبط الكود ده بيشتغل. أي reset controller محتاج `early` registration لازم يتعمله init صريح قبل الـ driver model يشتغل، مش يعتمد على `platform_driver` عادي.

---

### السيناريو 2: Allwinner A31 — Industrial Gateway — الـ I2C بيطلع `-EPROBE_DEFER` بشكل متكرر

#### العنوان
**الـ I2C sensor driver بيفضل يرجع `-EPROBE_DEFER` على gateway بـ sun6i A31**

#### السياق
شركة بتبني **industrial IoT gateway** بـ Allwinner A31 (sun6i). البورد بيشغل custom Yocto image. الـ I2C temperature sensor (TMP102) مش بيشتغل. الـ `dmesg` بيظهر:

```
tmp102 0-0048: probe deferred
tmp102 0-0048: probe deferred
tmp102 0-0048: probe deferred
```

ومش بيتحل حتى لو انتظرت.

#### المشكلة
الـ I2C controller نفسه محتاج reset قبل ما يشتغل. الـ reset node في الـ DT بيشاور على `ahb1_rst` — اللي محتاج `sun6i_reset_init()` عشان يكون موجود في الـ reset controller list. لو الـ `sunxi_reset_init()` اتعملت بس فشلت في الـ `request_mem_region`:

```c
if (!request_mem_region(res.start, size, np->name)) {
    ret = -EBUSY;
    goto err_alloc;  // بيعمل kfree وبيرجع -EBUSY
}
```

الـ reset controller مش هيتسجل، والـ I2C driver هيفضل يرجع `-EPROBE_DEFER` للأبد.

#### التحليل

لما `sunxi_reset_init()` بتاخد الـ resource:

```c
ret = of_address_to_resource(np, 0, &res);
if (ret)
    goto err_alloc;  // فشل في قراءة الـ reg من DT

size = resource_size(&res);
if (!request_mem_region(res.start, size, np->name)) {
    ret = -EBUSY;   // حد تاني حجز المنطقة دي
    goto err_alloc;
}
```

في الحالتين، الـ `data` بيتعمله `kfree` بس **مافيش** `iounmap` لأن `data->membase` ما اتعملش ioremap لسه. ده design صح — بس المشكلة إن الـ reset controller ماتسجلش خالص.

لو الـ region اتحجزت قبل كده (مثلاً bootloader عمل setup ومفيش `release_mem_region`)، الـ `request_mem_region` هيفشل.

#### الحل

**أولاً: اكتشف مين حاجز الـ region:**

```bash
cat /proc/iomem | grep -i "ahb1\|01c202"
# لو شايف المنطقة محجوزة بـ اسم تاني → فيه conflict
```

**ثانياً: اتحقق من الـ DT — ممكن في node تاني بنفس الـ reg:**

```bash
dtc -I fs /proc/device-tree 2>/dev/null | grep -B2 -A8 "01c202c0"
```

**ثالثاً: لو الـ bootloader هو المشكلة، أضف `release_mem_region` في bootloader أو خلي الـ kernel يعمل override:**

```bash
# kernel cmdline
memmap=0xc@0x01c202c0  # بيحجز المنطقة للـ kernel من البداية
```

**رابعاً: تأكد الـ reset controller اتسجل:**

```bash
ls /sys/bus/platform/drivers/reset-simple/
# لو مش موجود → الـ early init فشلت
dmesg | grep -i "reset\|sunxi"
```

#### الدرس المستفاد
الـ `request_mem_region` في `sunxi_reset_init()` ده check مهم — بيمنع إن driver تاني يشتغل على نفس الـ registers. بس لو فشل، لازم تعرف السبب من `/proc/iomem` قبل ما تحاول أي حل تاني.

---

### السيناريو 3: Allwinner H3 — IoT Sensor Node — الـ SPI flash مش بيشتغل بعد warm reset

#### العنوان
**الـ SPI NOR flash بيفشل بعد `reboot` بدون power cycle على H3 board**

#### السياق
**IoT sensor node** بـ Allwinner H3 بيستخدم SPI NOR flash (W25Q128) كـ data storage. بعد `reboot` الـ Linux، الـ SPI flash بيظهر في الـ dmesg بس أي read/write operation بتفشل بـ timeout. الـ power cycle (فصل وتوصيل) بيحل المشكلة. ده بيحصل في production بشكل عشوائي.

#### المشكلة
الـ SPI controller على H3 محتاج reset صح في كل boot. في الـ warm reboot، الـ `sun6i_reset_init()` بتشتغل تاني، بس الـ `request_mem_region` ممكن تفشل لأن الـ region اتحجزت من الـ boot الأول وما اتحررتش (لأن `release_mem_region` مش موجودة في الـ init path):

```c
// sunxi_reset_init() — مفيش cleanup function
// الـ data المتخصصة بـ kzalloc مش ليها devm أو unregister path
if (!request_mem_region(res.start, size, np->name)) {
    ret = -EBUSY;
    goto err_alloc;  // بيرجع error، الـ SPI reset controller ماتسجلش
}
```

لأن الـ function دي `__init`، الكود بيتشال من الـ memory بعد أول boot — بس الـ region المحجوزة فاضلة محجوزة. في warm reboot، الـ init code بيشتغل تاني وبيلاقي الـ region busy.

#### التحليل

```c
data->membase = ioremap(res.start, size);
if (!data->membase) {
    ret = -ENOMEM;
    goto err_alloc;
}
// لاحظ: لو ioremap نجحت بس request_mem_region فشلت → leak
// الترتيب في الكود: request_mem_region الأول، ioremap التاني
// يعني لو request_mem_region فشل → ioremap ماحصلتش → no leak
```

الـ flow الصح:
1. `kzalloc` → نجح
2. `of_address_to_resource` → نجح
3. `request_mem_region` → **فشل** في warm reboot بـ `-EBUSY`
4. `goto err_alloc` → `kfree(data)` بس
5. الـ SPI controller مش هيلاقي reset controller → probe fail

#### الحل

**أولاً: في الـ DT، استخدم `no-map` للـ reset region عشان تمنع conflict:**

```dts
/* مش حل حقيقي للـ request_mem_region بس بيوضح الـ region */
```

**ثانياً: الحل الحقيقي — استخدم `CONFIG_RESET_SIMPLE` driver بدل الـ early init للـ H3:**

```dts
// H3 ممكن يستخدم reset-simple driver بدل sun6i early init
ahb_rst: reset-controller@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;
    #reset-cells = <1>;
};
```

**ثالثاً: debug الـ warm reboot issue:**

```bash
# قبل reboot
cat /proc/iomem | grep 01c202

# بعد reboot في early dmesg
dmesg | grep -E "reset|EBUSY|sunxi"

# اتحقق من الـ SPI reset state
devmem2 0x01c0681c  # SPI0 reset register على H3
```

**رابعاً: workaround في الـ bootloader (U-Boot):**

```bash
# في U-Boot، قبل ما تبوت الـ kernel، حرر الـ SPI reset
mw.l 0x01c202c4 0xffffffff  # deassert كل الـ APB2 resets
```

#### الدرس المستفاد
الـ `__init` functions مع `request_mem_region` من غير corresponding `release_mem_region` ممكن تعمل مشاكل في warm reboot scenarios. السيناريو ده بيوضح ليه الـ early init path مختلف عن الـ platform_driver path اللي فيه `devm_` APIs بتتعمل cleanup أوتوماتيك.

---

### السيناريو 4: Allwinner H616 — Custom Board Bring-up — الـ HDMI مش بيطلع صورة

#### العنوان
**الـ HDMI مش بيشتغل على custom H616 board بسبب خطأ في حساب `nr_resets`**

#### السياق
Engineer بيعمل **custom board bring-up** لـ smart display بـ Allwinner H616. الـ HDMI port موجود فيزيائياً، الـ cable صح، الـ monitor بيقول "no signal". الـ `dmesg` بيظهر الـ HDMI driver اتسجل بس `drm` subsystem بيطلع error في الـ reset phase.

#### المشكلة
الـ engineer غلط في الـ `reg` property في الـ DT لـ AHB reset controller — كتب size غلط. ده خلى `nr_resets` يتحسب بقيمة أصغر من اللازم:

```c
size = resource_size(&res);
// لو size = 4 بدل 12 (0xc)
// nr_resets = 4 * 8 = 32 بدل 12 * 8 = 96
data->rcdev.nr_resets = size * 8;
```

الـ HDMI reset line ID ممكن يكون رقم 64 مثلاً — وده خارج الـ `nr_resets = 32`، فالـ reset framework برفض الطلب.

#### التحليل

في `sunxi_reset_init()`:

```c
ret = of_address_to_resource(np, 0, &res);
// res.start و res.end بييجوا من DT reg property
// مثال DT غلط:
// reg = <0x01c202c0 0x4>;  ← المفروض 0xc
size = resource_size(&res);
// resource_size = end - start + 1 = 0x4 = 4 bytes
data->rcdev.nr_resets = size * 8;  // = 32 فقط
```

لما الـ HDMI driver يطلب reset رقم 64:

```
reset_controller: reset ID 64 >= nr_resets 32
→ -EINVAL
→ HDMI probe fail
```

#### الحل

**أولاً: اتحقق من الـ DT الصح للـ H616:**

```bash
# قرا الـ reg property من الـ device tree
hexdump -C /proc/device-tree/ahb1-clock-reset@*/reg 2>/dev/null
# المفروض تشوف: 01 c2 02 c0 00 00 00 0c
#               (start=0x01c202c0, size=0x0c = 12 bytes)
```

**ثانياً: صح الـ DT:**

```dts
// غلط
ahb1_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0x4>;  /* خطأ! */
    #reset-cells = <1>;
};

// صح
ahb1_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;  /* 12 bytes = 96 reset lines */
    #reset-cells = <1>;
};
```

**ثالثاً: اتحقق من عدد الـ resets المتاحة:**

```bash
# بعد boot صح
cat /sys/kernel/debug/reset/*/nr_resets 2>/dev/null
# أو
dmesg | grep "reset-controller"
```

**رابعاً: اتحقق من الـ HDMI reset ID في الـ DT:**

```dts
// في الـ HDMI node
resets = <&ahb1_rst 64>;  // ID 64 → لازم nr_resets > 64
```

#### الدرس المستفاد
الـ `nr_resets = size * 8` ده formula بسيطة بس حساسة جداً لصح الـ `reg` property في الـ DT. أي خطأ في الـ size بيقطع كل الـ reset lines اللي ID-ها أكبر من القيمة المحسوبة. دايماً verify الـ reg size من الـ SoC datasheet مش من board schematic بس.

---

### السيناريو 5: Allwinner A64 — Tablet Bring-up — الـ spinlock corruption في multi-core boot

#### العنوان
**kernel panic بـ spinlock corruption على A64 quad-core tablet أثناء early boot**

#### السياق
Engineer بيعمل bring-up لـ **Android tablet** بـ Allwinner A64 (quad-core ARM Cortex-A53). البورد بيبوت على single core بشكل كامل، بس لما بيشغل SMP (كل الـ cores) بيحصل kernel panic في early boot:

```
BUG: spinlock bad magic on CPU#2
spin_bug() called in sunxi_reset_init
```

#### المشكلة
الـ `sun6i_reset_init()` معلمة `__init` وبتشتغل من `start_kernel()` على CPU0 بس. بس في بعض الـ configurations، الـ secondary cores بيبدأوا يشتغلوا قبل ما تخلص الـ init path. الـ `spin_lock_init()` في `sunxi_reset_init()` لازم تخلص قبل أي core تاني يحاول يوصل للـ reset controller:

```c
spin_lock_init(&data->lock);
// لو CPU1 حاول يعمل reset قبل الـ spin_lock_init تخلص → corruption
```

أو الـ scenario التاني: الـ `kzalloc` مع `GFP_KERNEL` في early init بيحتاج الـ memory allocator يكون جاهز — لو اتكلمت قبل `mm_init()` → panic.

#### التحليل

```c
static int sunxi_reset_init(struct device_node *np)
{
    struct reset_simple_data *data;
    ...
    data = kzalloc(sizeof(*data), GFP_KERNEL);  // ← محتاج mm جاهز
    if (!data)
        return -ENOMEM;
    ...
    spin_lock_init(&data->lock);  // ← لازم يخلص قبل SMP
    ...
    data->rcdev.ops = &reset_simple_ops;
    // reset_simple_ops بتستخدم data->lock في كل assert/deassert
    return reset_controller_register(&data->rcdev);
}
```

الـ `reset_simple_ops` (في reset-simple.c) بتعمل:
```c
// في كل assert/deassert:
spin_lock_irqsave(&data->lock, flags);
// لو lock ماتعملش init → undefined behavior
```

#### الحل

**أولاً: تأكد إن `sun6i_reset_init()` بتتكلم بعد `mm_init()`:**

```bash
# قرا الـ call trace في الـ panic
dmesg | grep -A20 "spin_bug\|BUG: spinlock"
# شوف الـ call stack — هل بييجي من postcore_initcall أو من arch init؟
```

**ثانياً: في الـ arch init code، تأكد الترتيب:**

```c
// arch/arm64/kernel/setup.c أو مشابه
void __init setup_arch(char **cmdline_p)
{
    ...
    // mm_init() لازم يجي أول
    mm_init();
    ...
    // بعدين فقط
    sun6i_reset_init();
}
```

**ثالثاً: لو الـ panic بييجي من SMP:**

```bash
# عطل SMP مؤقتاً وشوف لو الـ panic اتحل
# في kernel cmdline:
maxcpus=1 nr_cpus=1

# لو اتحل → مشكلة race condition في early init
```

**رابعاً: اتحقق من الـ SMP bring-up order:**

```bash
dmesg | grep -E "CPU[0-9]|smp|reset"
# شوف متى بالظبط الـ secondary cores بيبدأوا relative لـ reset init
```

**خامساً: الحل الأضمن — move الـ early reset init لـ `arch_initcall`:**

```c
// بدل استدعاء مباشر في setup_arch
static int __init sunxi_early_reset_setup(void)
{
    sun6i_reset_init();
    return 0;
}
arch_initcall(sunxi_early_reset_setup);
// arch_initcall بييجي بعد mm_init وقبل SMP bring-up في معظم الـ configs
```

#### الدرس المستفاد
الـ `__init` code على Sunxi SoCs مع SMP محتاج attention خاص للـ ordering. الـ `spin_lock_init()` في `sunxi_reset_init()` ده مش مجرد initialization روتيني — ده يحمي الـ shared reset registers من race conditions في multi-core environment. أي تغيير في الـ init order في arch code ممكن يكسر ده بشكل صامت.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المرجع | الرابط |
|--------|--------|
| **Reset controller API** — التوثيق الرئيسي للـ subsystem | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) |
| **ARM Allwinner SoCs** — صفحة دعم sunxi في الـ kernel | [docs.kernel.org/arch/arm/sunxi.html](https://docs.kernel.org/arch/arm/sunxi.html) |
| **Device Tree Bindings** — ملفات الـ bindings لـ reset controllers | [github.com/torvalds/linux — Documentation/devicetree/bindings/reset](https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/reset) |
| **Reset controller API** — النسخة القديمة من kernel.org | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) |

---

### مقالات LWN.net

**الـ** LWN.net هو أهم مصدر لمتابعة تطور الـ kernel — الروابط دي بتغطي تاريخ الـ reset framework:

| المقال | الرابط |
|--------|--------|
| **reset: Add generic GPIO reset driver** — نقطة تحول في تعميم الـ framework | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) |
| **ARM: sunxi: Introduce Allwinner H3 support** — إدخال دعم H3 مع reset controller | [lwn.net/Articles/644670](https://lwn.net/Articles/644670/) |
| **ARM: sunxi: Introduce Allwinner A23 (sun8i) support** — توسيع دعم sunxi | [lwn.net/Articles/602527](https://lwn.net/Articles/602527/) |
| **Expose and manage PCI device reset** — مقال عن reset management بشكل عام | [lwn.net/Articles/857548](https://lwn.net/Articles/857548/) |

---

### مناقشات Mailing List وـ Patchwork

**الـ** patchwork بيوثق رحلة كل patch من الـ RFC للـ merge — مفيد جداً لفهم قرارات التصميم:

| المناقشة | الرابط |
|----------|--------|
| **reset: sunxi: Add compatible for Allwinner H3 bus resets** (Jens Kuske) | [patchwork.kernel.org — patch/1445444428](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1445444428-4652-1-git-send-email-jenskuske@gmail.com/) |
| **reset: sunxi: Add Allwinner H3 bus resets** (v4) | [patchwork.ozlabs.org — patch/536747](https://patchwork.ozlabs.org/patch/536747/) |
| **ARM: sun6i: Add SMP support for the Allwinner A31** (Maxime Ripard) | [patchwork.kernel.org — patch/1383471013](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1383471013-21695-3-git-send-email-maxime.ripard@free-electrons.com/) |
| **linux-sunxi Mailing List** — قائمة بريدية لمجتمع Allwinner | [linux-sunxi.org/Mailing\_list](https://linux-sunxi.org/Mailing_list) |
| **linux-sunxi Google Group** — أرشيف النقاشات | [groups.google.com/g/linux-sunxi](https://groups.google.com/g/linux-sunxi) |

---

### مصادر مجتمع linux-sunxi

**الـ** linux-sunxi community هو اللي قاد mainlining جهاز Allwinner في الـ kernel الرسمي:

| المصدر | الرابط |
|--------|--------|
| **Linux Kernel** — صفحة kernel في wiki الـ sunxi | [linux-sunxi.org/Linux\_Kernel](https://linux-sunxi.org/Linux_Kernel) |
| **Linux mainlining effort** — تتبع تقدم إدخال الـ drivers | [linux-sunxi.org/Linux\_mainlining\_effort](https://linux-sunxi.org/Linux_mainlining_effort) |
| **Mainline Kernel Howto** — دليل عملي للمطورين | [linux-sunxi.org/Mainline\_Kernel\_Howto](https://linux-sunxi.org/Mainline_Kernel_Howto) |
| **linux-sunxi GitHub** — الـ source التاريخي قبل mainlining | [github.com/linux-sunxi/linux-sunxi](https://github.com/linux-sunxi/linux-sunxi) |

---

### ملفات الـ Source Code المهمة في الـ Kernel

الملفات دي هي القلب اللي لازم تقراه لفهم الـ subsystem كامل:

```
drivers/reset/reset-sunxi.c          ← الـ driver الرئيسي (الملف اللي بندرسه)
drivers/reset/reset-simple.c         ← الـ generic simple reset ops
drivers/reset/core.c                 ← قلب الـ reset framework
include/linux/reset-controller.h     ← struct reset_controller_dev وـ ops
include/linux/reset/reset-simple.h   ← struct reset_simple_data
include/linux/reset.h                ← الـ consumer API
Documentation/driver-api/reset.rst   ← التوثيق الرسمي
Documentation/devicetree/bindings/reset/allwinner,sunxi-clock-reset.yaml
```

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
**الـ** LDD3 متاحة مجاناً — الفصول دي الأهم لفهم الـ reset driver:

| الفصل | الموضوع |
|-------|---------|
| **Chapter 3** — Char Drivers | فهم `file_operations` وهيكل الـ drivers |
| **Chapter 9** — Communicating with Hardware | `ioremap`, `ioread32`, `iowrite32` |
| **Chapter 14** — The Linux Device Model | `platform_device`, `device_node` |

> متاحة على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
| الفصل | الموضوع |
|-------|---------|
| **Chapter 11** — Timers and Time Management | فهم `spinlock_t` وـ locking |
| **Chapter 9** — An Introduction to Kernel Synchronization | `spin_lock_irqsave` |
| **Chapter 17** — Devices and Modules | `platform_driver`, `of_device_id` |

#### Embedded Linux Primer — Christopher Hallinan
- **Chapter 15** — Debugging Embedded Linux — تشخيص مشاكل الـ reset في الـ embedded systems
- **Chapter 16** — Kernel Debugging Techniques — استخدام `dev_dbg` ودوات الـ debugging

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 6** — Device Drivers — شرح تفصيلي لـ `platform_device` و `of_match_table`

---

### أدوات للبحث وإيجاد المزيد

#### الـ Search Terms المفيدة

```
# للبحث في kernel source:
git log --oneline --all -- drivers/reset/
git log --all --grep="sunxi reset"
git log --all --grep="reset-controller"

# للبحث في LWN:
site:lwn.net "reset controller" linux
site:lwn.net allwinner sunxi

# للبحث في patchwork:
site:patchwork.kernel.org "reset: sunxi"
site:patchwork.ozlabs.org "reset sunxi"

# للبحث العام:
"reset_controller_register" site:elixir.bootlin.com
"reset_simple_ops" site:elixir.bootlin.com
```

#### أدوات Online للتصفح

| الأداة | الاستخدام | الرابط |
|--------|-----------|--------|
| **Elixir Cross-Referencer** | تصفح الـ kernel source مع cross-references | [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/drivers/reset) |
| **git.kernel.org** | الـ git history الرسمي | [git.kernel.org/torvalds/linux](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git) |
| **lore.kernel.org** | أرشيف الـ mailing lists | [lore.kernel.org/linux-arm-kernel](https://lore.kernel.org/linux-arm-kernel/) |
| **LKDDB** | قاعدة بيانات الـ kernel drivers | [cateee.net/lkddb — RESET\_SIMPLE](https://cateee.net/lkddb/web-lkddb/RESET_SIMPLE.html) |

---

### ملخص رحلة الـ reset-sunxi.c

```
2013  ← Maxime Ripard يكتب أول نسخة لـ reset-sunxi.c لدعم sun6i A31
         └─ الـ driver بيستخدم early init لأن AHB1 reset محتاج قبل
            ما يتشغل الـ device model

2015  ← Jens Kuske يضيف compatible لـ Allwinner H3
         └─ [patchwork.kernel.org/patch/1445444428]

2016+ ← الـ simple reset framework يتعمم
         └─ reset-sunxi.c يبقى wrapper بسيط فوق reset-simple.c
            بدل ما كان يحتوي الـ logic كله جواه
```

**الـ** driver ده مثال ممتاز على مبدأ الـ kernel: كود صغير وواضح، يعمل حاجة واحدة كويس.
## Phase 8: Writing simple module

### الفكرة

الـ `reset_controller_register()` هي أهم function في الملف — بتسجّل الـ reset controller في الـ kernel. هنستخدم **kprobe** عشان نـ hook عليها ونطبع معلومات عن كل controller بيتسجّل: عدد الـ resets، والـ device tree node اللي جاي منها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe hook on reset_controller_register()
 * Prints info about every reset controller registered in the system.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/reset-controller.h>   /* struct reset_controller_dev */
#include <linux/of.h>                 /* struct device_node, of_node_full_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Docs Demo");
MODULE_DESCRIPTION("kprobe on reset_controller_register to log reset controllers");

/* ------------------------------------------------------------------ */
/* pre_handler — بيتشغّل مباشرةً قبل ما reset_controller_register تنفّذ */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86_64 the first argument lives in rdi.
     * On ARM64 it lives in x0.
     * kprobes gives us pt_regs so we read the right register.
     */
#if defined(CONFIG_X86_64)
    struct reset_controller_dev *rcdev =
        (struct reset_controller_dev *)regs->di;
#elif defined(CONFIG_ARM64)
    struct reset_controller_dev *rcdev =
        (struct reset_controller_dev *)regs->regs[0];
#else
    /* Unsupported arch — bail silently */
    return 0;
#endif

    if (!rcdev)
        return 0;

    /* Print the DT node name if available, otherwise "no-dt-node" */
    pr_info("[sunxi-reset-probe] reset_controller_register called:\n");
    pr_info("  nr_resets  = %u\n", rcdev->nr_resets);
    pr_info("  of_node    = %s\n",
            rcdev->of_node ? of_node_full_name(rcdev->of_node)
                           : "no-dt-node");
    pr_info("  owner      = %s\n",
            rcdev->owner ? rcdev->owner->name : "no-owner");

    return 0; /* 0 = let the real function execute normally */
}

/* ------------------------------------------------------------------ */
/* struct kprobe — بتحدد إيه الـ function اللي هنـ hook عليها          */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "reset_controller_register",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init — بيسجّل الـ kprobe عند تحميل الـ module               */
/* ------------------------------------------------------------------ */
static int __init reset_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[sunxi-reset-probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[sunxi-reset-probe] kprobe planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit — بيشيل الـ kprobe عند إزالة الـ module عشان يمنع crash */
/* ------------------------------------------------------------------ */
static void __exit reset_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[sunxi-reset-probe] kprobe removed from %s\n",
            kp.symbol_name);
}

module_init(reset_probe_init);
module_exit(reset_probe_exit);
```

---

### Makefile

```makefile
obj-m += reset_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | يعرّف `struct kprobe` و`register_kprobe` / `unregister_kprobe` |
| `linux/reset-controller.h` | يعرّف `struct reset_controller_dev` اللي هنـ cast إليها argument الأول |
| `linux/of.h` | يوفر `of_node_full_name()` عشان نطبع اسم الـ DT node بشكل مقروء |

#### الـ `handler_pre`

الـ kprobe بيوقف التنفيذ لحظة قبل ما الـ function الأصلية تشتغل. بنقرأ الـ `pt_regs` عشان نجيب البـ argument الأول (`rcdev`) — وده بيختلف بين الـ architectures (x86 بيحطه في `rdi`، ARM64 في `x0`). بعدين بنطبع عدد الـ resets واسم الـ DT node واسم الـ module اللي بيسجّل الـ controller.

#### الـ `struct kprobe`

الـ `symbol_name` بتقول للـ kernel على أي رمز نحط الـ probe، والـ `pre_handler` هو الـ callback اللي هيتشغّل. الـ kernel بيحل الـ symbol لعنوان وقت الـ registration.

#### الـ `module_init` / `module_exit`

الـ `register_kprobe` بتزرع الـ breakpoint في الـ kernel. لازم نعمل `unregister_kprobe` في الـ exit عشان لو الـ module اتشال والـ probe لسه موجودة، أي استدعاء لـ `reset_controller_register` بعد كده هيتسبب في **kernel panic** لأن الـ handler بتاعنا اتشال من الذاكرة.

---

### تشغيل تجريبي

```bash
# بناء الـ module
make

# تحميله
sudo insmod reset_probe.ko

# نشوف اللوج — لو في driver بيسجّل reset controller هتشوف المعلومات
sudo dmesg | grep sunxi-reset-probe

# إزالته
sudo rmmod reset_probe
```

مثال على الـ output المتوقع:

```
[sunxi-reset-probe] kprobe planted at ffffffffc0123456 (reset_controller_register)
[sunxi-reset-probe] reset_controller_register called:
  nr_resets  = 64
  of_node    = /soc/reset-controller@1c202c0
  owner      = reset-simple
```
