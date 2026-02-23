## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `of_platform.h`** ينتمي إلى subsystem يُعرف رسمياً في MAINTAINERS بـ **"OPEN FIRMWARE AND FLATTENED DEVICE TREE"**، يُشرف عليه Rob Herring وSaravana Kannan، ويغطي كل ملفات `drivers/of/` و`include/linux/of*.h`.

---

### القصة — لماذا وُجد هذا الملف؟

تخيّل أنك تشتري لوحة تحكم كهربائية (embedded board) مثل Raspberry Pi أو board ARM صناعية. هذه اللوحة تحتوي على عشرات المكونات: UART، I2C، SPI، Ethernet، GPIO، timers... إلخ.

في عالم PC التقليدي، يكتشف الـ kernel هذه الأجهزة تلقائياً عبر **PCI bus enumeration** — الحافلة نفسها تخبر الـ kernel "أنا موجود هنا". لكن في عالم embedded، كل هذه الأجهزة **مُلحَّمة مباشرة على الـ chip** (memory-mapped)، لا يوجد بروتوكول اكتشاف تلقائي.

الحل التاريخي كان: يكتب المبرمج في الـ kernel C code يحدد يدوياً أين يجلس كل جهاز في الـ memory. النتيجة: كود مليء بـ board-specific hacks، ومستحيل صيانته.

**الحل الحديث**: **Device Tree (DT)** — ملف نصي يصف الـ hardware بشكل مستقل عن الـ kernel، يُمرَّر من الـ bootloader للـ kernel عند الإقلاع.

مثال مبسّط لـ Device Tree node:
```
uart0: serial@101f1000 {
    compatible = "arm,pl011";
    reg = <0x101f1000 0x1000>;
    interrupts = <1 0>;
};
```

الآن، السؤال: **كيف يحوّل الـ kernel هذه النصوص إلى `platform_device` حقيقية يستطيع driver تشغيلها؟**

هنا يأتي دور **`of_platform.h`**.

---

### ما الذي يفعله هذا الملف تحديداً؟

**الـ `of_platform.h`** هو الـ header الذي يُعلن عن الواجهة البرمجية المسؤولة عن **الجسر الرابط** بين:

```
Device Tree nodes  ──────────────►  Linux platform_device objects
  (in memory, parsed from .dtb)         (registered in device model)
```

بمجرد أن تمتلك `platform_device`، يعمل كل شيء بشكل اعتيادي: الـ driver يُسجَّل عبر `platform_driver`، والـ kernel يطابقه مع الجهاز، ويستدعى `probe()`.

---

### الوظائف الرئيسية المُعلَنة

| الدالة | الدور |
|--------|-------|
| `of_platform_populate()` | يمشي على شجرة الـ DT من node معيّن ويُنشئ `platform_device` لكل node مطابق |
| `of_platform_default_populate()` | نفس السابق لكن يستخدم قائمة التطابق الافتراضية للـ kernel |
| `of_platform_depopulate()` | يُزيل كل الأجهزة التي تم إنشاؤها بـ populate |
| `devm_of_platform_populate()` | نسخة managed — تُلغى تلقائياً عند تحرير الـ device |
| `of_platform_device_create()` | يُنشئ `platform_device` واحد من DT node بعينه |
| `of_platform_device_destroy()` | يُتلف `platform_device` واحد |
| `of_find_device_by_node()` | يبحث عن `platform_device` مرتبط بـ DT node محدد |
| `of_platform_bus_probe()` | النسخة القديمة (legacy) من populate، لا تزال مستخدمة |
| `of_device_alloc()` | يُخصص `platform_device` ويملأه من DT node |
| `of_device_register()` / `of_device_unregister()` | تسجيل/إلغاء تسجيل platform device |

---

### الـ `struct of_dev_auxdata` — جدول الاستثناءات

```c
struct of_dev_auxdata {
    char *compatible;       /* string to match DT node */
    resource_size_t phys_addr; /* base address to match */
    char *name;             /* override device name */
    void *platform_data;    /* legacy platform_data pointer */
};
```

هذا الـ struct هو **حل انتقالي** لمشكلة قديمة: بعض الـ subsystems في الـ kernel كانت تبحث عن الجهاز باسمه النصي (مثل `"serial8250.0"`). لكن الـ Device Tree يولّد أسماء تلقائية مختلفة. فـ `auxdata` يسمح بتجاوز هذا الاسم.

الـ kernel نفسه يحذّر: استخدام `auxdata` يُعدّ **last resort**، الأفضل دائماً قراءة البيانات مباشرة من الـ DT node.

---

### سيناريو عملي: كيف يقلع ARM board

```
[ Bootloader: U-Boot ]
        │
        │  يمرر DTB (Device Tree Blob) للـ kernel
        ▼
[ Kernel Early Init ]
        │
        │  يُفكك DTB في الذاكرة → شجرة من device_node structs
        ▼
[ machine_desc->init_machine() ]   ← للـ ARM مثلاً
        │
        │  يستدعي of_platform_populate(NULL, matches, auxdata, NULL)
        ▼
[ of_platform.c يمشي على الشجرة ]
        │
        │  لكل node يطابق matches[]
        │    → ينشئ platform_device
        │    → يُسجّله في device model
        ▼
[ Driver Matching & probe() ]
        │
        │  كل driver سجّل نفسه بـ platform_driver + of_match_table
        │  الـ kernel يطابق compatible strings
        ▼
[ Hardware يعمل ]
```

---

### الملفات ذات الصلة التي يجب معرفتها

#### الملفات الجوهرية للـ subsystem (OF/DT)

| الملف | الدور |
|-------|-------|
| `drivers/of/platform.c` | **التنفيذ الحقيقي** لكل دوال `of_platform.h` |
| `drivers/of/base.c` | عمليات التنقل في شجرة الـ DT (البحث، القراءة، التكرار) |
| `drivers/of/device.c` | ربط الـ `device_node` بـ `platform_device` وإدارة الـ uevent |
| `drivers/of/address.c` | تحويل عناوين الـ DT إلى physical addresses |
| `drivers/of/fdt.c` | فك ضغط وتحليل الـ DTB الخام من الـ bootloader |
| `drivers/of/irq.c` | ترجمة interrupt specs من الـ DT |
| `drivers/of/overlay.c` | تطبيق DP overlays ديناميكياً (للـ FPGA وغيره) |
| `drivers/of/property.c` | قراءة properties من nodes الـ DT |

#### الـ Headers الرئيسية

| الملف | الدور |
|-------|-------|
| `include/linux/of_platform.h` | **هذا الملف** — واجهة الربط بين DT وplatform bus |
| `include/linux/of.h` | التعريفات الأساسية: `device_node`, `property`, `of_phandle_args` |
| `include/linux/of_device.h` | مطابقة الـ driver بالـ device عبر `of_match_device()` |
| `include/linux/of_address.h` | واجهة ترجمة عناوين الـ DT |
| `include/linux/of_irq.h` | واجهة ترجمة interrupts من الـ DT |
| `include/linux/mod_devicetable.h` | تعريف `of_device_id` (جدول المطابقة) |

#### ملفات البنية التحتية للـ platform bus

| الملف | الدور |
|-------|-------|
| `drivers/base/platform.c` | تنفيذ الـ platform bus نفسه |
| `include/linux/platform_device.h` | تعريف `platform_device` و`platform_driver` |

---

### لماذا هذا الملف مهم؟

كل driver ARM/RISC-V/PowerPC/MIPS تقريباً على أي embedded system يمر عبر هذه الواجهة. **`of_platform_populate()`** هي النقطة التي تتحول فيها بيانات الـ hardware الجامدة المكتوبة في ملف نصي إلى كيانات حية في نظام الـ Linux device model. بدونها، يجب كتابة C code يدوي لكل board — وهو ما كان يُفعَل قبل 2008.
## Phase 2: شرح الـ OF Platform Framework

### المشكلة — لماذا يوجد هذا الـ Subsystem؟

في الأنظمة المدمجة (ARM، PowerPC، RISC-V)، لا يوجد بروتوكول قياسي كالـ PCI يسمح للـ CPU باستكشاف الـ hardware تلقائياً. الـ UART، الـ SPI controller، الـ I2C bus، الـ GPIO — كل هذه الأجهزة **memory-mapped** وعناوينها ثابتة في الـ SoC، والكرنل لا يعرفها إلا إذا أُخبر بها صراحةً.

قبل الـ Device Tree، كان الحل هو ملفات C ضخمة مثل:

```c
/* arch/arm/mach-omap2/board-omap3beagle.c */
static struct platform_device uart1_device = {
    .name = "omap-uart",
    .id   = 1,
    .resource = uart1_resources,
    ...
};
```

هذا يعني: **كل board = ملف C منفصل** في الكرنل، وكل إضافة hardware تستلزم recompile. المشكلة تفاقمت مع تزايد عدد الـ SoCs وتنوع الـ boards.

**الـ Device Tree** جاء ليفصل وصف الـ hardware عن الـ kernel source. لكن الـ Device Tree وحده مجرد بيانات — يحتاج كوداً يقرأه ويحوّله إلى `platform_device` حقيقية يفهمها الـ driver model. هذا بالضبط دور الـ **OF Platform Framework** (`of_platform.h`).

**OF = Open Firmware** — المعيار الأصلي من Sun/Apple قبل أن يتطور إلى الـ Device Tree Specification الحديث.

---

### الحل — ماذا يفعل الكرنل؟

الـ OF Platform Framework هو **الجسر** بين شجرة الـ Device Tree (بياناتها في الذاكرة كـ `device_node`) وبين الـ Linux Driver Model (الذي يعمل مع `platform_device`).

مهمته الأساسية:
1. **يمشي** على شجرة الـ Device Tree node بـ node.
2. لكل node يطابق معايير معينة، **ينشئ** `platform_device` مناسب.
3. **يسجّل** هذه الأجهزة في الـ platform bus.
4. الـ platform bus يبحث عن **driver مناسب** عبر `compatible` string.

---

### التشبيه الواقعي — مكتب التوظيف

تخيّل مصنعاً كبيراً (الـ SoC) وصل مع قائمة موظفين مكتوبة على ورق (الـ Device Tree / DTB):

```
اسم: uart-controller
عنوان مكتبه: 0x02020000
مهاراته: fsl,imx6ul-uart, fsl,imx6q-uart
```

الـ OF Platform Framework هو **مكتب الموارد البشرية (HR)**:
- يقرأ القائمة الورقية (`device_node` = الورق)
- ينشئ بطاقة هوية رسمية لكل موظف (`platform_device` = بطاقة الهوية)
- يرسله للمكتب الصحيح (يسجله في الـ platform bus)

الـ `of_device_id` هو **وصف الوظيفة** — الـ driver يقول "أنا أعمل مع من يحمل مهارة `fsl,imx6ul-uart`"، فيُربط الموظف بقسمه.

الـ `of_dev_auxdata` هو **جدول الاستثناءات** — بعض الموظفين القدامى (legacy code) لهم اسم مختلف في السجلات الرسمية عن اسمهم في القائمة الورقية، فيُصحَّح الاسم قبل التسجيل.

الخريطة الكاملة للتشبيه:

| عنصر الـ HR | المقابل في الكرنل |
|---|---|
| القائمة الورقية الواصلة مع المصنع | الـ DTB (Device Tree Blob) |
| كل سطر في القائمة | `struct device_node` |
| بطاقة الهوية الرسمية | `struct platform_device` |
| مكتب HR الذي يُنشئ البطاقات | `of_platform_populate()` |
| جدول الاستثناءات | `struct of_dev_auxdata` |
| وصف الوظيفة | `struct of_device_id` |
| قسم الشركة الذي يستقبل الموظف | الـ platform bus |
| مشرف القسم الذي يبدأ العمل | `platform_driver.probe()` |

---

### البنية الكبيرة — أين يقع هذا الـ Framework؟

```
+--------------------------------------------------+
|              User Space                          |
+--------------------------------------------------+
          |                      |
+--------------------+  +--------------------+
|   Device Drivers   |  |   Device Drivers   |
|  uart-imx.c        |  |  i2c-imx.c         |
|  (.compatible =    |  |  (.compatible =    |
|  "fsl,imx6ul-uart")|  |  "fsl,imx21-i2c")  |
+--------------------+  +--------------------+
          |                      |
+--------------------------------------------------+
|            Platform Bus (platform_bus_type)      |
|  -- يربط platform_device ↔ platform_driver --   |
+--------------------------------------------------+
          ^                      ^
          |                      |
+--------------------------------------------------+
|     OF Platform Framework  (of_platform.h)       |
|                                                  |
|  of_platform_populate()                          |
|    └── يمشي على device_node tree                 |
|    └── ينشئ platform_device لكل node مطابق      |
|    └── يسجله في platform bus                     |
+--------------------------------------------------+
          ^
          |
+--------------------------------------------------+
|         OF Core  (include/linux/of.h)            |
|   device_node tree — شجرة مبنية من الـ DTB       |
|   struct device_node { name, properties, ... }   |
+--------------------------------------------------+
          ^
          |
+--------------------------------------------------+
|    DTB (Device Tree Blob) — مُحمَّل من bootloader |
|    /chosen, /cpus, /soc/uart@02020000, ...       |
+--------------------------------------------------+
          ^
          |
+--------------------------------------------------+
|        Bootloader (U-Boot / UEFI)                |
|   يُمرّر عنوان DTB في r2 / x1 للكرنل            |
+--------------------------------------------------+
```

---

### الـ Core Abstractions — البنيات الأساسية

#### 1. `struct device_node` — تمثيل الـ DT Node

معرَّف في `include/linux/of.h`:

```c
struct device_node {
    const char        *name;         /* اسم الـ node مثل "uart" */
    phandle            phandle;      /* معرّف فريد للإشارة من nodes أخرى */
    const char        *full_name;    /* المسار الكامل مثل "/soc/uart@02020000" */
    struct fwnode_handle fwnode;     /* نقطة التكامل مع firmware node abstraction */

    struct property   *properties;  /* قائمة مرتبطة من الـ properties */
    struct device_node *parent;      /* الـ node الأب */
    struct device_node *child;       /* أول ابن */
    struct device_node *sibling;     /* الأخ التالي */
};
```

الـ `fwnode_handle` مهم: هو abstraction طبقة فوق `device_node` يسمح للكود بالعمل مع ACPI و DT بنفس الـ API. يعني من يستخدم `fwnode_handle` لا يهتم هل الـ source كان DTB أو ACPI table.

#### 2. `struct of_device_id` — جدول المطابقة

معرَّف في `include/linux/mod_devicetable.h`:

```c
struct of_device_id {
    char    name[32];         /* مطابقة بالاسم (deprecated غالباً) */
    char    type[32];         /* مطابقة بالـ type (deprecated غالباً) */
    char    compatible[128];  /* ** الحقل الأهم ** مثل "fsl,imx6ul-uart" */
    const void *data;         /* بيانات خاصة بالـ driver لكل entry */
};
```

الـ `compatible` string يتبع الصيغة: `"vendor,model"` — فمثلاً `"fsl,imx6ul-uart"` تعني: **Freescale** (الآن NXP)، controller الـ UART الموجود في **iMX6UL**.

الـ driver يُعرّف جدول المطابقة هكذا:

```c
static const struct of_device_id imx_uart_dt_ids[] = {
    { .compatible = "fsl,imx6ul-uart", .data = &imx_uart_devdata[IMX6UL_UART] },
    { .compatible = "fsl,imx6q-uart",  .data = &imx_uart_devdata[IMX6Q_UART]  },
    { /* sentinel — نهاية الجدول */ }
};
MODULE_DEVICE_TABLE(of, imx_uart_dt_ids);
```

الـ `MODULE_DEVICE_TABLE` يُخرج هذا الجدول بحيث يقرأه `udev` لتحميل الـ module تلقائياً.

#### 3. `struct platform_device` — الجهاز كما يراه الـ Driver Model

معرَّف في `include/linux/platform_device.h`:

```c
struct platform_device {
    const char   *name;          /* اسم الجهاز المستخدم للـ matching */
    int           id;            /* رقم تمييزي (-1 إذا واحد فقط) */
    struct device dev;           /* الـ generic device — قلب الـ driver model */
    u32           num_resources;
    struct resource *resource;   /* الـ IRQs والـ memory regions */
    ...
};
```

العلاقة بين `platform_device` و `device_node`:

```
platform_device
    └── dev  (struct device)
         └── of_node  (struct device_node *)  ← الرابط للـ DT
              └── properties
                   ├── compatible = "fsl,imx6ul-uart"
                   ├── reg = <0x02020000 0x4000>
                   ├── interrupts = <0 26 0x04>
                   └── clocks = <&clks IMX6UL_CLK_UART1>
```

بعد `of_platform_populate()`، يستطيع الـ driver الوصول لكل خصائص الـ DT عبر:

```c
struct device_node *np = pdev->dev.of_node;
u32 reg_val;
of_property_read_u32(np, "fifo-size", &reg_val);
```

#### 4. `struct of_dev_auxdata` — جدول التوافق مع الـ Legacy Code

```c
struct of_dev_auxdata {
    char            *compatible;   /* مطابقة النوع من الـ DT */
    resource_size_t  phys_addr;    /* مطابقة العنوان الفيزيائي */
    char            *name;         /* الاسم البديل المطلوب تعيينه */
    void            *platform_data;/* بيانات إضافية (legacy) */
};
```

**متى يُستخدم؟** حين يوجد كود قديم يبحث عن device بالاسم (مثل `"omap-uart.0"`) بدلاً من الـ `of_device_id`. الجدول يقول للـ framework: "إذا وجدت node بـ compatible هذا في العنوان هذا، سمِّ الـ platform_device باسم كذا."

**ملاحظة:** الـ comment في الكود الأصلي صريح: *"Using an auxdata lookup table should be considered a last resort"* — هو حل مؤقت للانتقال إلى الـ DT وليس ممارسة موصى بها.

---

### دورة الحياة الكاملة — من DTB إلى `probe()`

```
Boot:
  U-Boot يُمرّر DTB address
          │
          ▼
  early_init_dt_scan()
  يبني device_node tree في الذاكرة
          │
          ▼
  machine_init() / board setup
          │
          ▼
  of_platform_populate(NULL, of_default_bus_match_table, NULL, NULL)
  ┌─────────────────────────────────────────┐
  │  لكل device_node في الشجرة:            │
  │    هل يطابق جدول الـ matches؟           │
  │    نعم → of_platform_device_create()    │
  │           └── of_device_alloc()         │
  │                ├── platform_device_alloc │
  │                ├── of_address_to_resource│  ← يحوّل reg property إلى struct resource
  │                └── of_irq_to_resource    │  ← يحوّل interrupts property
  │           └── of_device_add()           │
  │                └── platform_device_add() │  ← يسجل في platform bus
  └─────────────────────────────────────────┘
          │
          ▼
  platform_bus يبحث عن driver مطابق
  (يقارن compatible string مع of_device_id جدول كل driver)
          │
          ▼
  platform_driver.probe(pdev) ← الـ driver يبدأ عمله
```

---

### الدوال الرئيسية وما تفعله

#### `of_platform_populate()`

```c
int of_platform_populate(
    struct device_node     *root,    /* NULL = root الشجرة كلها */
    const struct of_device_id *matches, /* أي nodes تهمنا */
    const struct of_dev_auxdata *lookup, /* جدول legacy overrides */
    struct device *parent            /* الـ parent device */
);
```

هذه هي **الدالة المركزية**. تمشي على الشجرة وتنشئ الأجهزة. يستدعيها الكرنل عادةً مرة واحدة عند البوت من `arch_initcall` أو من `machine_desc.init_machine`.

#### `of_platform_default_populate()`

```c
int of_platform_default_populate(
    struct device_node *root,
    const struct of_dev_auxdata *lookup,
    struct device *parent
);
```

نسخة مبسطة تستخدم `of_default_bus_match_table` الجاهزة — تشمل nodes من النوع `simple-bus`، `simple-mfd`، إلخ.

#### `devm_of_platform_populate()` — الإصدار الآمن مع managed resources

```c
int devm_of_platform_populate(struct device *dev);
```

الـ `devm_` prefix يعني: عندما يُزال الـ device (مثلاً module unload أو device hotplug)، يُنظَّف كل ما أُنشئ تلقائياً دون الحاجة لاستدعاء `devm_of_platform_depopulate()` يدوياً.

**يُستخدم** من داخل `probe()` في الـ drivers التي تُنشئ child devices (مثل MFD drivers — Multi-Function Devices):

```c
static int mfd_probe(struct platform_device *pdev)
{
    /* ينشئ child platform_devices لكل sub-node في الـ DT */
    return devm_of_platform_populate(&pdev->dev);
}
```

#### `of_find_device_by_node()`

```c
struct platform_device *of_find_device_by_node(struct device_node *np);
```

البحث العكسي: عندك `device_node`، تريد الـ `platform_device` المقابل له. مفيد في الـ cross-device references.

---

### ما يملكه الـ Framework مقابل ما يفوّضه للـ Drivers

| المسؤولية | يملكها الـ Framework | يفوّضها للـ Driver |
|---|---|---|
| مشي شجرة الـ DT | نعم | لا |
| إنشاء `platform_device` | نعم | لا |
| تحويل `reg` → `struct resource` | نعم | لا |
| تحويل `interrupts` → IRQ number | نعم (جزئياً، مع IRQ domain) | لا |
| ربط `device_node` بالـ `platform_device` | نعم | لا |
| تسجيل الجهاز في الـ bus | نعم | لا |
| قراءة خصائص الـ DT (clocks, gpios, ...) | لا | نعم — في `probe()` |
| تهيئة الـ hardware (reset, clock enable) | لا | نعم — في `probe()` |
| تعريف جدول `of_device_id` | لا | نعم |
| إدارة interrupt handlers | لا | نعم |

---

### مثال واقعي كامل — UART Controller على i.MX6UL

**الـ DTS:**
```dts
/* arch/arm/boot/dts/imx6ul.dtsi */
uart1: serial@2020000 {
    compatible = "fsl,imx6ul-uart", "fsl,imx6q-uart";
    reg = <0x02020000 0x4000>;
    interrupts = <GIC_SPI 26 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&clks IMX6UL_CLK_UART1_IPG>,
             <&clks IMX6UL_CLK_UART1_SERIAL>;
    clock-names = "ipg", "per";
    status = "disabled";
};
```

**البوت:**
```c
/* arch/arm/mach-imx/mach-imx6ul.c */
static void __init imx6ul_init_machine(void)
{
    of_platform_default_populate(NULL, NULL, NULL);
    /* الآن uart1 أصبح platform_device مسجّلاً */
}
```

**الـ Driver:**
```c
/* drivers/tty/serial/imx.c */
static const struct of_device_id imx_uart_dt_ids[] = {
    { .compatible = "fsl,imx6ul-uart",
      .data = &imx_uart_devdata[IMX6UL_UART] },
    { /* sentinel */ }
};

static struct platform_driver imx_uart_platform_driver = {
    .probe  = imx_uart_probe,
    .driver = {
        .name           = "imx-uart",
        .of_match_table = imx_uart_dt_ids,
    },
};
```

**تدفق الـ probe:**
```c
static int imx_uart_probe(struct platform_device *pdev)
{
    /* الـ framework أعطانا platform_device مع of_node مرتبط */
    struct device_node *np = pdev->dev.of_node;

    /* resource موجود مسبقاً — حوّله الـ framework من reg property */
    sport->port.mapbase = r->start;
    sport->port.membase = devm_ioremap_resource(&pdev->dev, r);

    /* IRQ موجود مسبقاً — حوّله الـ framework من interrupts property */
    sport->port.irq = platform_get_irq(pdev, 0);

    /* ما يفعله الـ driver بنفسه: يقرأ الـ clocks */
    sport->clk_ipg = devm_clk_get(&pdev->dev, "ipg");
    sport->clk_per = devm_clk_get(&pdev->dev, "per");

    return uart_add_one_port(&imx_reg, &sport->port);
}
```

---

### العلاقة مع Subsystems أخرى

- **OF Core (`include/linux/of.h`)**: الطبقة التحتية — تبني الـ `device_node` tree وتوفر دوال القراءة. الـ OF Platform تبني فوقها.
- **Platform Bus (`platform_bus_type`)**: الـ bus abstraction الذي يربط `platform_device` بـ `platform_driver`. الـ OF Platform تُسجّل الأجهزة هنا.
- **Driver Model (`struct device`, `struct bus_type`)**: الـ generic infrastructure للـ Linux device management — كل شيء يعود إليه في النهاية.
- **IRQ Domain**: يُستخدم داخلياً عند تحويل `interrupts` property إلى Linux IRQ numbers.
- **devres (Device Resources)**: نظام الـ `devm_*` — الـ `devm_of_platform_populate` يعتمد عليه لتنظيف الموارد تلقائياً.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### أعلام الـ `device_node` — `_flags`

| Flag | القيمة | المعنى |
|------|--------|--------|
| `OF_DYNAMIC` | 1 | الـ node وخصائصه مُخصَّصة بـ `kmalloc` (ديناميكية) |
| `OF_DETACHED` | 2 | الـ node منفصل عن شجرة الأجهزة |
| `OF_POPULATED` | 3 | تم إنشاء `platform_device` لهذا الـ node |
| `OF_POPULATED_BUS` | 4 | تم إنشاء bus platform لأبناء هذا الـ node |
| `OF_OVERLAY` | 5 | الـ node مخصَّص لـ overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | ضمن cset overlay قيد التحرير |

#### خيارات الـ Kconfig المؤثرة على الـ API

| Config | الأثر على `of_platform.h` |
|--------|---------------------------|
| `CONFIG_OF` | يُفعِّل `of_find_device_by_node()` |
| `CONFIG_OF_ADDRESS` | يُفعِّل كامل واجهة `of_platform_device_create/populate/depopulate` |
| `CONFIG_OF_DYNAMIC` | يُفعِّل `of_platform_register_reconfig_notifier()` ومعالجة الـ overlays |
| `CONFIG_ARM_AMBA` | يُفعِّل `of_amba_device_create()` لأجهزة `arm,primecell` |
| `CONFIG_PPC` | مسار خاص في `of_platform_default_populate_init()` لـ MacOS display |

#### ثوابت `platform_device`

| الثابت | القيمة | المعنى |
|--------|--------|--------|
| `PLATFORM_DEVID_NONE` | -1 | لا يوجد ID رقمي للجهاز |
| `PLATFORM_DEVID_AUTO` | -2 | الـ kernel يختار ID تلقائياً |

#### Compatible strings الافتراضية في `of_platform_default_populate()`

| Compatible | الغرض |
|------------|--------|
| `simple-bus` | bus بسيط يُولِّد أجهزة لأبنائه |
| `simple-mfd` | Multi-Function Device بسيط |
| `isa` | ISA bus |
| `arm,amba-bus` | AMBA bus (فقط مع `CONFIG_ARM_AMBA`) |

---

### الـ Structs الأساسية

#### 1. `struct of_dev_auxdata`

**الغرض:** جدول بحث (lookup table) يتيح تجاوز الأسماء التلقائية المولَّدة من شجرة الأجهزة عند إنشاء `platform_device`. يُستخدم كملاذ أخير عند الحاجة لأسماء محددة يعتمد عليها كود Linux القديم.

```c
struct of_dev_auxdata {
    char            *compatible;    /* compatible string للمطابقة مع device node */
    resource_size_t  phys_addr;     /* العنوان الفيزيائي للتسجيل — يُميِّز بين أجهزة متشابهة */
    char            *name;          /* الاسم المراد تعيينه لـ platform_device */
    void            *platform_data; /* بيانات خاصة بالمنصة تُمرَّر للـ driver */
};
```

**الحقول بالتفصيل:**

| الحقل | النوع | الدور |
|-------|-------|-------|
| `compatible` | `char *` | يُطابَق مع `of_device_is_compatible()` — إلزامي لكل إدخال |
| `phys_addr` | `resource_size_t` | عند وجوده، يُستخدم للتمييز بين نسختين من نفس الـ compatible |
| `name` | `char *` | يُمرَّر كـ `bus_id` لـ `of_device_alloc()` |
| `platform_data` | `void *` | يُسند إلى `dev->dev.platform_data` |

**الارتباط بـ Structs أخرى:** يُستخدم حصراً كـ array تنتهي بإدخال فارغ، ويُمرَّر إلى `of_platform_populate()` و`of_platform_bus_create()`.

---

#### 2. `struct device_node`

**الغرض:** تمثيل kernel لعقدة واحدة في شجرة أجهزة DT. هو نقطة الانطلاق لكل عمليات `of_platform`.

```c
struct device_node {
    const char       *name;        /* اسم العقدة (آخر مكوِّن في المسار) */
    phandle           phandle;     /* معرِّف رقمي فريد داخل DT */
    const char       *full_name;   /* المسار الكامل مثل /soc/uart@1c28000 */
    struct fwnode_handle fwnode;   /* واجهة firmware node — نقطة الربط مع device model */
    struct property  *properties;  /* قائمة مرتبطة بخصائص الـ node */
    struct property  *deadprops;   /* خصائص محذوفة (محتفظ بها للـ overlay rollback) */
    struct device_node *parent;    /* العقدة الأم */
    struct device_node *child;     /* أول ابن */
    struct device_node *sibling;   /* الأخ التالي (قائمة مرتبطة) */
    unsigned long    _flags;       /* OF_POPULATED, OF_DYNAMIC, ... */
    void             *data;        /* بيانات خاصة بالـ arch */
};
```

---

#### 3. `struct platform_device`

**الغرض:** تمثيل kernel لجهاز منصة (platform device). هو الناتج الرئيسي لكل دوال `of_platform_*`.

```c
struct platform_device {
    const char    *name;           /* اسم الجهاز — يُستخدم في مطابقة الـ driver */
    int            id;             /* ID رقمي، أو PLATFORM_DEVID_NONE */
    bool           id_auto;        /* true إذا اختار kernel الـ ID تلقائياً */
    struct device  dev;            /* الـ device المُضمَّن — جوهر الـ driver model */
    u64            platform_dma_mask;
    struct device_dma_parameters dma_parms;
    u32            num_resources;  /* عدد عناصر مصفوفة resource */
    struct resource *resource;     /* مصفوفة موارد (IOMEM, IRQ, ...) مستخرجة من DT */
    const struct platform_device_id *id_entry; /* إدخال جدول الـ ID إن وُجد */
    const char    *driver_override; /* إجبار مطابقة driver معين */
    struct mfd_cell *mfd_cell;     /* للأجهزة MFD */
    struct pdev_archdata archdata;  /* بيانات خاصة بالـ arch */
};
```

**الحقل الأهم:** `dev` — يحمل `dev.of_node` (مؤشر لـ `device_node`)، و`dev.parent`، و`dev.bus`، وكل بنية الـ driver model.

---

#### 4. `struct property`

**الغرض:** تمثيل خاصية واحدة داخل `device_node` (مثل `compatible`, `reg`, `interrupts`).

```c
struct property {
    char           *name;   /* اسم الخاصية */
    int             length; /* حجم القيمة بالبايت */
    void           *value;  /* قيمة الخاصية (big-endian raw bytes) */
    struct property *next;  /* الخاصية التالية في القائمة */
};
```

---

#### 5. `struct of_device_id`

**الغرض:** إدخال في جدول مطابقة (match table) — يُحدِّد أنواع `device_node` التي يتعامل معها driver أو bus معين.

```c
/* من include/linux/mod_devicetable.h */
struct of_device_id {
    char name[32];        /* مطابقة على اسم الـ node */
    char type[32];        /* مطابقة على نوع الجهاز */
    char compatible[128]; /* مطابقة على compatible string — الأكثر استخداماً */
    const void *data;     /* بيانات خاصة بالـ driver لهذا الإدخال */
};
```

---

#### 6. `struct of_reconfig_data`

**الغرض:** يُمرَّر إلى notifier عند تغيير شجرة الأجهزة ديناميكياً (overlays).

```c
struct of_reconfig_data {
    struct device_node *dn;       /* العقدة المُضافة أو المحذوفة */
    struct property    *prop;     /* الخاصية الجديدة */
    struct property    *old_prop; /* الخاصية القديمة قبل التغيير */
};
```

---

### مخططات العلاقات بين الـ Structs

#### العلاقات الأساسية

```
┌─────────────────────────────────────────────────────────────────┐
│                        device_node                              │
│  name, full_name, phandle                                       │
│  _flags: OF_POPULATED | OF_POPULATED_BUS | OF_DYNAMIC | ...     │
│  ┌──────────────┐                                               │
│  │ fwnode_handle│◄──── of_fwnode_handle(np) ────────────────┐  │
│  └──────────────┘                                            │  │
│  *properties ──► property ──► property ──► NULL              │  │
│  *parent ──► device_node (العقدة الأم)                       │  │
│  *child  ──► device_node (أول ابن)                           │  │
│  *sibling──► device_node (الأخ التالي)                       │  │
└─────────────────────────────────────────────────────────────────┘
                        │
                        │ of_device_alloc() ينشئ:
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      platform_device                            │
│  name, id                                                       │
│  num_resources, *resource ──► resource[0], resource[1], ...    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      device (dev)                       │   │
│  │  of_node ──────────────────────────────────────────────►│──►│──► device_node (نفس الـ np)
│  │  parent  ──► device (الجهاز الأم)                       │   │
│  │  bus     ──► platform_bus_type                          │   │
│  │  fwnode  ◄── يُعيَّن من fwnode_handle أعلاه             │   │
│  │  platform_data ──► void* (من of_dev_auxdata أو NULL)    │   │
│  │  coherent_dma_mask = DMA_BIT_MASK(32)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

#### علاقة `of_dev_auxdata` بعملية البحث

```
of_platform_populate(root, matches, lookup, parent)
          │
          │ lookup = مصفوفة of_dev_auxdata[]
          ▼
┌─────────────────────┐     of_dev_lookup()      ┌───────────────────────┐
│   device_node (np)  │ ──────────────────────► │  of_dev_auxdata entry │
│   compatible: "foo" │   1. طابق compatible     │  compatible: "foo"    │
│   reg: <0x1c28000>  │   2. طابق phys_addr      │  phys_addr: 0x1c28000 │
└─────────────────────┘   3. استرجع name+data    │  name: "uart0"        │
                                                  │  platform_data: &cfg  │
                                                  └───────────────────────┘
                                                          │
                                                          ▼
                                              bus_id = "uart0"
                                              platform_data = &cfg
                                              → of_platform_device_create_pdata(...)
```

---

### مخطط دورة الحياة الكاملة

```
                    ══════════════════════════════
                    PHASE 1: الإنشاء (Creation)
                    ══════════════════════════════

device tree blob (DTB)
        │
        ▼
of_unflatten_dtree()
        │
        ▼
┌────────────────┐
│  device_node   │  ← شجرة كاملة في الذاكرة
│  (الشجرة كلها) │    _flags = 0 (لم يُنشأ جهاز بعد)
└────────────────┘
        │
        │  arch_initcall_sync →
        │  of_platform_default_populate_init()
        │
        ▼
of_platform_default_populate(NULL, NULL, NULL)
        │
        ▼
of_platform_populate(root="/", match_table, lookup=NULL, parent=NULL)
        │
        ├── device_links_supplier_sync_state_pause()
        │
        ├── for_each_child_of_node(root, child):
        │       └── of_platform_bus_create(child, matches, lookup, parent, strict=true)
        │
        └── device_links_supplier_sync_state_resume()


                    ══════════════════════════════════════
                    PHASE 2: إنشاء جهاز واحد (Per-Node)
                    ══════════════════════════════════════

of_platform_bus_create(bus_node, matches, lookup, parent, strict)
        │
        ├── [تحقق] compatible موجود؟ (strict mode)
        ├── [تحقق] OF_POPULATED_BUS مُعيَّن؟ → skip
        ├── [تحقق] of_skipped_node_table? → skip
        │
        ├── of_dev_lookup(lookup, bus_node) → auxdata أو NULL
        │
        ├── [إذا arm,primecell] → of_amba_device_create() → return
        │
        ├── of_platform_device_create_pdata(bus_node, bus_id, pdata, parent)
        │       │
        │       ├── of_device_is_available(np)? لا → return NULL
        │       ├── of_node_test_and_set_flag(np, OF_POPULATED) → مُعيَّن؟ skip
        │       │
        │       ├── of_device_alloc(np, bus_id, parent)
        │       │       ├── platform_device_alloc("", PLATFORM_DEVID_NONE)
        │       │       ├── of_address_count(np) → num_reg
        │       │       ├── kcalloc(num_reg, sizeof(resource))
        │       │       ├── of_address_to_resource(np, i, res) لكل reg
        │       │       ├── device_set_node(&dev->dev, of_fwnode_handle(np))
        │       │       ├── dev->dev.parent = parent ?: &platform_bus
        │       │       └── of_device_make_bus_id() أو dev_set_name()
        │       │
        │       ├── dev->dev.bus = &platform_bus_type
        │       ├── dev->dev.platform_data = platform_data
        │       ├── of_msi_configure()
        │       │
        │       └── of_device_add(dev)
        │               ├── ofdev->name = dev_name(&ofdev->dev)
        │               ├── ofdev->id = PLATFORM_DEVID_NONE
        │               ├── set_dev_node() ← NUMA node من DT
        │               └── device_add(&ofdev->dev) ← تسجيل رسمي في kernel
        │
        ├── [إذا of_match_node(matches, bus_node)] ← هل هو bus؟
        │       └── for_each_child_of_node(bus_node, child):
        │               └── of_platform_bus_create(child, ...) ← RECURSIVE
        │
        └── of_node_set_flag(bus_node, OF_POPULATED_BUS)


                    ══════════════════════════════
                    PHASE 3: التفكيك (Teardown)
                    ══════════════════════════════

of_platform_depopulate(parent)
        │
        ├── parent->of_node مُعيَّن OF_POPULATED_BUS؟
        │
        ├── device_for_each_child_reverse(parent, of_platform_device_destroy)
        │       │  (عكسي لضمان تفكيك الأبناء قبل الآباء)
        │       │
        │       └── of_platform_device_destroy(dev)
        │               ├── OF_POPULATED مُعيَّن؟ لا → skip
        │               ├── OF_POPULATED_BUS؟ → تكرار recursive للأبناء
        │               ├── of_node_clear_flag(OF_POPULATED)
        │               ├── of_node_clear_flag(OF_POPULATED_BUS)
        │               ├── [platform bus] → platform_device_unregister()
        │               └── [amba bus]     → amba_device_unregister()
        │
        └── of_node_clear_flag(parent->of_node, OF_POPULATED_BUS)
```

---

### مخططات تدفق الاستدعاءات (Call Flow)

#### تدفق `of_platform_populate()` الكامل

```
board/arch code
  └── of_platform_default_populate(NULL, NULL, NULL)
        └── of_platform_populate(root="/", match_table, NULL, NULL)
              │
              ├── of_node_get(root)   ← زيادة refcount
              │
              ├── device_links_supplier_sync_state_pause()
              │       └── [يوقف sync_state callbacks أثناء البناء]
              │
              ├── for each child of "/":
              │     └── of_platform_bus_create(child, ...)
              │           │
              │           ├── of_dev_lookup() ← بحث في auxdata
              │           │
              │           ├── of_platform_device_create_pdata()
              │           │     ├── of_device_alloc()
              │           │     │     ├── platform_device_alloc()
              │           │     │     │     └── kzalloc(platform_device)
              │           │     │     ├── of_address_to_resource() [لكل reg]
              │           │     │     │     └── __of_address_to_resource()
              │           │     │     │           └── of_translate_address()
              │           │     │     └── device_set_node()
              │           │     │           └── dev->fwnode = &np->fwnode
              │           │     │
              │           │     └── of_device_add()
              │           │           ├── set_dev_node() [NUMA]
              │           │           └── device_add()
              │           │                 ├── bus_probe_device()
              │           │                 │     └── platform_match()
              │           │                 │           └── of_driver_match_device()
              │           │                 └── [trigger driver probe إن وُجد]
              │           │
              │           └── [إذا bus] → of_platform_bus_create(child) ← recursive
              │
              ├── device_links_supplier_sync_state_resume()
              │
              ├── of_node_set_flag(root, OF_POPULATED_BUS)
              └── of_node_put(root)  ← تخفيض refcount
```

#### تدفق `devm_of_platform_populate()` (managed version)

```
driver probe()
  └── devm_of_platform_populate(dev)
        ├── devres_alloc(devm_of_platform_populate_release, sizeof(dev*))
        │     └── [تخصيص devres resource مرتبط بعمر الجهاز]
        │
        ├── of_platform_populate(dev->of_node, NULL, NULL, dev)
        │     └── [نفس التدفق أعلاه لكن root = dev->of_node]
        │
        ├── [نجاح] *ptr = dev; devres_add(dev, ptr)
        │     └── [تسجيل cleanup تلقائي]
        │
        └── [عند unbind الـ driver تلقائياً]:
              devm_of_platform_populate_release()
                └── of_platform_depopulate(dev)
```

#### تدفق الـ Dynamic Overlay (CONFIG_OF_DYNAMIC)

```
overlay apply (e.g., via /sys/kernel/config/device-tree/overlays/)
  └── of_reconfig_notifier_call_chain(OF_RECONFIG_CHANGE_ADD, rd)
        └── of_platform_notify(nb, action, rd)
              │
              ├── of_reconfig_get_state_change() → OF_RECONFIG_CHANGE_ADD
              │
              ├── [تحقق] parent هو bus أو root؟
              ├── [تحقق] OF_POPULATED مُعيَّن؟ → skip
              │
              ├── rd->dn->fwnode.flags &= ~FWNODE_FLAG_NOT_DEVICE
              │
              ├── pdev_parent = of_find_device_by_node(parent)
              │     └── bus_find_device_by_of_node(&platform_bus_type, parent)
              │
              └── of_platform_device_create(rd->dn, NULL, &pdev_parent->dev)
                    └── of_platform_device_create_pdata(..., NULL, ...)


overlay remove:
  └── of_platform_notify(..., OF_RECONFIG_CHANGE_REMOVE, ...)
        ├── pdev = of_find_device_by_node(rd->dn)
        ├── of_platform_device_destroy(&pdev->dev, &children_left)
        └── platform_device_put(pdev)
```

---

### استراتيجية الـ Locking

#### نظرة عامة

الـ `of_platform` layer بذاتها **لا تمتلك locks خاصة بها** — تعتمد على آليات الـ locking الموجودة في طبقات أدنى:

| الطبقة | آلية الـ Lock | ما يحميه |
|--------|---------------|----------|
| `device_node._flags` | `atomic bit ops` (`test_and_set_bit`) | يمنع إنشاء جهازين لنفس الـ node بالتزامن (`OF_POPULATED`) |
| `device_add()` / `device_unregister()` | `device_lock(dev)` داخلياً | سلامة بنية الـ device في kernel |
| `of_node_get()` / `of_node_put()` | `kref` (atomic refcount) | عمر الـ `device_node` في الذاكرة |
| `devres` list | `dev->devres_lock` (spinlock) | قائمة الـ managed resources |
| `device_links` | `device_links_lock` (rwsem) | علاقات supplier/consumer بين الأجهزة |
| DT tree traversal | `of_mutex` أو `RCU` (حسب الـ build) | قراءة/تعديل شجرة الـ DT |

#### التسلسل الوقائي الحرج: `OF_POPULATED`

```c
/* في of_platform_device_create_pdata() — atomic check-and-set */
if (!of_device_is_available(np) ||
    of_node_test_and_set_flag(np, OF_POPULATED))  /* ← atomic! */
    return NULL;

/* ... إنشاء الجهاز ... */

/* عند الفشل فقط: */
err_clear_flag:
    of_node_clear_flag(np, OF_POPULATED);  /* ← إعادة الحالة */
```

هذا النمط يضمن **idempotency** — لو استُدعيت الدالة مرتين لنفس الـ node، المرة الثانية تُرجع `NULL` بأمان بدلاً من إنشاء جهاز مكرر.

#### ترتيب الـ Locks (Lock Ordering)

```
1. of_mutex / RCU          ← قراءة شجرة DT
        ↓
2. device_links_lock (rwsem) ← pause/resume sync_state
        ↓
3. device_node._flags (atomic) ← OF_POPULATED flag
        ↓
4. device_lock(dev)         ← device_add() / device_unregister()
        ↓
5. dev->devres_lock (spinlock) ← devres_add() في devm_ variants
```

**قاعدة:** لا يجب الإمساك بـ lock من مستوى أعلى أثناء طلب lock من مستوى أدنى — الترتيب أعلاه يُجنِّب الـ deadlock.

#### تأثير `device_links_supplier_sync_state_pause()`

```
of_platform_populate() يُوقف sync_state callbacks
        │
        ├── [إنشاء كل الأجهزة من DT]
        │     └── لا يُطلَق sync_state لأي supplier حتى الآن
        │
        └── device_links_supplier_sync_state_resume()
              └── الآن فقط يُرسَل sync_state لكل supplier
                  تأكَّد أن consumers كلهم مُسجَّلون
```

الهدف: ضمان أن `sync_state` callback لجهاز supplier لا يُستدعى إلا بعد تسجيل **جميع** consumer devices من DT، لتجنب الـ power management bugs.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### مجموعة: Allocation & Registration

| الدالة | الغرض | `EXPORT_SYMBOL` |
|---|---|---|
| `of_device_alloc()` | تخصيص `platform_device` من node وملء resources | `EXPORT_SYMBOL` |
| `of_device_add()` | إضافة device محضّر مسبقاً إلى bus | لا |
| `of_device_register()` | `initialize` + `add` في خطوة واحدة | `EXPORT_SYMBOL` |
| `of_device_unregister()` | إلغاء تسجيل device من bus | `EXPORT_SYMBOL` |

#### مجموعة: Device Creation

| الدالة | الغرض | `EXPORT_SYMBOL` |
|---|---|---|
| `of_platform_device_create()` | إنشاء وتسجيل `platform_device` من DT node | `EXPORT_SYMBOL` |
| `of_platform_device_create_pdata()` | نفسها مع تمرير `platform_data` (static) | لا |
| `of_amba_device_create()` | إنشاء `amba_device` لـ ARM PrimeCell | لا |

#### مجموعة: Lookup

| الدالة | الغرض | `EXPORT_SYMBOL` |
|---|---|---|
| `of_find_device_by_node()` | إيجاد `platform_device` من pointer إلى node | `EXPORT_SYMBOL` |
| `of_dev_lookup()` | البحث في جدول `of_dev_auxdata` (static) | لا |

#### مجموعة: Tree Population

| الدالة | الغرض | `EXPORT_SYMBOL` |
|---|---|---|
| `of_platform_bus_probe()` | مسح DT قديم بدون شرط `compatible` | `EXPORT_SYMBOL` |
| `of_platform_populate()` | مسح DT حديث مع شرط `compatible` | `EXPORT_SYMBOL_GPL` |
| `of_platform_default_populate()` | `populate` باستخدام match table افتراضي | `EXPORT_SYMBOL_GPL` |
| `of_platform_bus_create()` | إنشاء device لـ node وأطفاله بشكل recursive (static) | لا |

#### مجموعة: Cleanup / Depopulate

| الدالة | الغرض | `EXPORT_SYMBOL` |
|---|---|---|
| `of_platform_device_destroy()` | تدمير device واحد ومسح flags الـ node | `EXPORT_SYMBOL_GPL` |
| `of_platform_depopulate()` | إزالة جميع الأجهزة التي أنشأها `of_platform_populate` | `EXPORT_SYMBOL_GPL` |

#### مجموعة: Devres-Managed

| الدالة | الغرض | `EXPORT_SYMBOL` |
|---|---|---|
| `devm_of_platform_populate()` | `populate` مُدارة تلقائياً عند unbind الـ driver | `EXPORT_SYMBOL_GPL` |
| `devm_of_platform_depopulate()` | تحرير يدوي مبكر لـ `devm_of_platform_populate` | `EXPORT_SYMBOL_GPL` |

#### مجموعة: Dynamic OF / Notifier (CONFIG_OF_DYNAMIC)

| الدالة | الغرض |
|---|---|
| `of_platform_notify()` | معالجة أحداث إضافة/إزالة nodes من DT الحي |
| `of_platform_register_reconfig_notifier()` | تسجيل الـ notifier في subsystem الـ OF |

---

### المجموعة 1: Allocation & Low-Level Registration

**الغرض:** هذه الدوال تعمل على المستوى المنخفض — تخصيص الذاكرة للـ `platform_device`، ربطه بالـ device node، واستخراج الـ memory resources من الـ DT، ثم إضافته إلى الـ driver model.

---

#### `of_device_alloc()`

```c
struct platform_device *of_device_alloc(struct device_node *np,
                                         const char *bus_id,
                                         struct device *parent);
```

**ما تفعله:** تخصص `platform_device` فارغاً عبر `platform_device_alloc()`، تعدّ كل الـ `reg` resources من الـ node باستخدام `of_address_count()` ثم تملأها عبر `of_address_to_resource()`. تربط الـ `device_node` بالـ device عبر `device_set_node()` وتضبط الاسم إما من `bus_id` أو تولده تلقائياً عبر `of_device_make_bus_id()`.

**المعاملات:**
- `np` — الـ `device_node` المصدري من الـ DT؛ يُؤخذ منه عدد الـ reg windows والاسم.
- `bus_id` — اسم صريح للـ device على الـ bus؛ يُمرَّر `NULL` لتوليد اسم تلقائي من عنوان الـ node.
- `parent` — الـ `struct device` الأب؛ إذا كان `NULL` يُستخدَم `platform_bus` كجذر.

**القيمة المُعادة:** pointer إلى `platform_device` جديد، أو `NULL` عند فشل الـ allocation.

**تفاصيل مهمة:**
- تستدعي `of_node_get(np)` داخلياً للحفاظ على reference count صحيح للـ node.
- الـ resources تُخصَّص بـ `kcalloc` وتُسند مباشرة إلى `dev->resource`.
- لا تسجّل الـ device في الـ bus — هذا دور `of_device_add()`.
- تُستدعى من: `of_platform_device_create_pdata()` و `of_amba_device_create()`.

---

#### `of_device_add()`

```c
int of_device_add(struct platform_device *ofdev);
```

**ما تفعله:** تُعيّن `ofdev->name` من `dev_name(&ofdev->dev)` وتضبط `ofdev->id = PLATFORM_DEVID_NONE` لتجنّب ارتباك الـ platform bus في عملية الـ matching. تستدعي `of_node_to_nid()` لتحديد NUMA node المرتبط بالـ device ثم تستدعي `device_add()` لإضافته فعلياً إلى سجل الأجهزة.

**المعاملات:**
- `ofdev` — الـ `platform_device` المُهيَّأ مسبقاً بـ `of_node` صالح (يُوقف kernel بـ `BUG_ON` إذا كان NULL).

**القيمة المُعادة:** 0 عند النجاح، أو كود خطأ سالب من `device_add()`.

**تفاصيل مهمة:**
- تفترض أن `device_initialize()` قد استُدعيت مسبقاً.
- تُستخدَم مباشرة من `of_platform_device_create_pdata()` بعد الإعداد الكامل.
- الـ NUMA affinity تأتي من `of_node_to_nid()`؛ إذا أعاد `NUMA_NO_NODE` يرث الـ device node أبيه.

---

#### `of_device_register()`

```c
int of_device_register(struct platform_device *pdev);
```

**ما تفعله:** تجمع `device_initialize()` و `of_device_add()` في استدعاء واحد — مناسبة لتسجيل أجهزة يدوياً من خارج مسار populate العادي.

**المعاملات:**
- `pdev` — الـ `platform_device` المُخصَّص مسبقاً والمُهيَّأ يدوياً.

**القيمة المُعادة:** 0 نجاح، كود خطأ سالب عند الفشل.

**تفاصيل مهمة:** لا تستدعيها بعد `of_device_alloc()` لأن الأخيرة لا تستدعي `device_initialize()` — استخدم `of_platform_device_create()` بدلاً عنها في هذا المسار.

---

#### `of_device_unregister()`

```c
void of_device_unregister(struct platform_device *ofdev);
```

**ما تفعله:** تستدعي `device_unregister(&ofdev->dev)` مباشرة، مما يُطلق الـ reference ويُزيل الـ device من sysfs وقائمة الـ bus.

**المعاملات:**
- `ofdev` — الـ `platform_device` المُسجَّل مسبقاً.

**القيمة المُعادة:** void.

**تفاصيل مهمة:** لا تمسح `OF_POPULATED` flag على الـ node — هذا دور `of_platform_device_destroy()`. استخدمها فقط للأجهزة المُسجَّلة يدوياً عبر `of_device_register()`.

---

### المجموعة 2: Device Creation (Public & Internal)

**الغرض:** هذه الدوال تدمج الـ allocation والإعداد والتسجيل في مسار واحد، وتتحقق من availability الـ node وتضبط `OF_POPULATED` flag لتجنّب التكرار.

---

#### `of_platform_device_create()`

```c
struct platform_device *of_platform_device_create(struct device_node *np,
                                                    const char *bus_id,
                                                    struct device *parent);
```

**ما تفعله:** wrapper مباشر لـ `of_platform_device_create_pdata()` مع `platform_data = NULL`. تُنشئ `platform_device` كامل من الـ DT node وتُسجّله في `platform_bus`.

**المعاملات:**
- `np` — الـ DT node المُراد إنشاء device له.
- `bus_id` — الاسم على الـ bus، أو `NULL` للتوليد التلقائي.
- `parent` — الأب في device hierarchy.

**القيمة المُعادة:** pointer إلى `platform_device`، أو `NULL` إذا كان الـ node غير متاح أو مُسكَّن مسبقاً أو فشل الـ allocation.

---

#### `of_platform_device_create_pdata()` (static)

```c
static struct platform_device *of_platform_device_create_pdata(
    struct device_node *np,
    const char *bus_id,
    void *platform_data,
    struct device *parent);
```

**ما تفعله:** تتحقق أولاً من `of_device_is_available(np)` ثم تضبط `OF_POPULATED` flag atomically عبر `of_node_test_and_set_flag()` لضمان عدم التكرار. تستدعي `of_device_alloc()`، تضبط `DMA_BIT_MASK(32)` و MSI configuration، ثم تستدعي `of_device_add()`.

**pseudocode:**

```c
of_platform_device_create_pdata(np, bus_id, platform_data, parent):
    if !of_device_is_available(np) OR of_node_test_and_set_flag(np, OF_POPULATED):
        return NULL   // unavailable or already done

    dev = of_device_alloc(np, bus_id, parent)
    if !dev:
        goto err_clear_flag

    dev->dev.coherent_dma_mask = DMA_BIT_MASK(32)
    dev->dev.dma_mask = &dev->dev.coherent_dma_mask
    dev->dev.bus = &platform_bus_type
    dev->dev.platform_data = platform_data
    of_msi_configure(&dev->dev, np)

    if of_device_add(dev) != 0:
        platform_device_put(dev)
        goto err_clear_flag

    return dev

err_clear_flag:
    of_node_clear_flag(np, OF_POPULATED)
    return NULL
```

**تفاصيل مهمة:**
- `OF_POPULATED` flag تمنع إنشاء device مكرر لنفس الـ node.
- DMA mask تُضبط بـ 32-bit كـ default — الـ driver يمكنه رفعها لاحقاً.
- عند فشل `of_device_add()` يُزال الـ flag ليسمح بإعادة المحاولة.

---

#### `of_amba_device_create()` (static, CONFIG_ARM_AMBA)

```c
static struct amba_device *of_amba_device_create(struct device_node *node,
                                                   const char *bus_id,
                                                   void *platform_data,
                                                   struct device *parent);
```

**ما تفعله:** تُنشئ `amba_device` لأجهزة ARM PrimeCell (compatible = `"arm,primecell"`). تقرأ `arm,primecell-periphid` property لتحديد الـ Peripheral ID، تستخرج resource الأولى فقط عبر `of_address_to_resource()`, ثم تستدعي `amba_device_add()` مع `iomem_resource`.

**تفاصيل مهمة:**
- تُستدعى فقط من `of_platform_bus_create()` عند اكتشاف `"arm,primecell"` compatible.
- تضبط `OF_POPULATED` flag كذلك لتجنّب التكرار.
- عند `!CONFIG_ARM_AMBA` تُعيد `NULL` دائماً بدون خطأ.

---

### المجموعة 3: Lookup

**الغرض:** إيجاد `platform_device` مرتبط بـ DT node معين، أو البحث في جدول override للأسماء.

---

#### `of_find_device_by_node()`

```c
struct platform_device *of_find_device_by_node(struct device_node *np);
```

**ما تفعله:** تُجري بحثاً في `platform_bus_type` عبر `bus_find_device_by_of_node()` وتُعيد الـ `platform_device` المقابل. تزيد reference count للـ `struct device` الداخلي.

**المعاملات:**
- `np` — الـ DT node المُراد إيجاد device مرتبط به.

**القيمة المُعادة:** pointer إلى `platform_device` مع reference مرفوع، أو `NULL` إذا لم يوجد.

**تفاصيل مهمة:**
- **يجب** استدعاء `put_device()` أو `platform_device_put()` على النتيجة بعد الاستخدام لتجنّب leak.
- تُستخدَم في `of_platform_notify()` للعثور على device عند استقبال `OF_RECONFIG_CHANGE_REMOVE`.

---

#### `of_dev_lookup()` (static)

```c
static const struct of_dev_auxdata *of_dev_lookup(
    const struct of_dev_auxdata *lookup,
    struct device_node *np);
```

**ما تفعله:** تبحث في جدول `of_dev_auxdata` عن entry تطابق الـ node. المرحلة الأولى تطابق `compatible` **و** `phys_addr`. المرحلة الثانية (fallback) تطابق `compatible` فقط إذا كانت `phys_addr` و `name` كلاهما صفر/NULL.

**المعاملات:**
- `lookup` — جدول `of_dev_auxdata` منتهٍ بـ empty entry؛ يُقبَل `NULL`.
- `np` — الـ node المُراد البحث عنه.

**القيمة المُعادة:** pointer إلى entry المطابق، أو `NULL`.

**تفاصيل مهمة:** الأولوية للتطابق بـ `phys_addr` — يمنع الالتباس عند وجود أجهزة متعددة بنفس الـ compatible. تُستدعى فقط من `of_platform_bus_create()`.

---

### المجموعة 4: Tree Population

**الغرض:** المسح الـ recursive لشجرة الـ DT وإنشاء `platform_device` لكل node مؤهّل. هذه هي الدوال التي تُشغَّل عند boot لملء الـ device hierarchy.

---

#### `of_platform_bus_create()` (static)

```c
static int of_platform_bus_create(struct device_node *bus,
                                   const struct of_device_id *matches,
                                   const struct of_dev_auxdata *lookup,
                                   struct device *parent, bool strict);
```

**ما تفعله:** النواة الـ recursive للـ populate. تتحقق من وجود `compatible` property (إذا كان `strict=true`)، تتجاوز nodes الـ `of_skipped_node_table` (مثل `operating-points-v2`)، وتتجاوز nodes المُسكَّنة مسبقاً (`OF_POPULATED_BUS`). تبحث في `lookup` عبر `of_dev_lookup()` للحصول على اسم override. تُنشئ إما `amba_device` أو `platform_device`. إذا تطابق الـ node مع `matches`، تكرر العملية على أطفاله.

**pseudocode:**

```c
of_platform_bus_create(bus, matches, lookup, parent, strict):
    if strict AND !has_compatible(bus):  return 0
    if bus in of_skipped_node_table:     return 0
    if OF_POPULATED_BUS set on bus:      return 0

    auxdata = of_dev_lookup(lookup, bus)
    // get override name/platform_data

    if compatible == "arm,primecell":
        of_amba_device_create(bus, ...)
        return 0   // never recurse into primecell children

    dev = of_platform_device_create_pdata(bus, ...)
    if !dev OR !of_match_node(matches, bus):
        return 0   // leaf device, no recursion

    for each child of bus:
        rc = of_platform_bus_create(child, matches, lookup, &dev->dev, strict)
        if rc: break

    of_node_set_flag(bus, OF_POPULATED_BUS)
    return rc
```

**تفاصيل مهمة:**
- `strict=true` يُفرض من `of_platform_populate()`، `strict=false` من `of_platform_bus_probe()`.
- `OF_POPULATED_BUS` يُضبط فقط إذا كان الـ node نفسه يطابق `matches` وتم معالجة أطفاله.
- الـ recursion تتوقف عند `arm,primecell` لأن AMBA devices لا تُعامَل كـ bus.

---

#### `of_platform_bus_probe()`

```c
int of_platform_bus_probe(struct device_node *root,
                           const struct of_device_id *matches,
                           struct device *parent);
```

**ما تفعله:** واجهة قديمة (legacy) لمسح الـ DT. إذا كان `root` نفسه يطابق `matches` يُعالجه مباشرة، وإلا يُكرر على أطفاله ويُعالج من يطابق. تُستخدم `strict=false` أي لا يُشترط `compatible` property.

**المعاملات:**
- `root` — نقطة البداية؛ `NULL` يعني جذر الشجرة `/`.
- `matches` — جدول `of_device_id` للـ bus nodes المطلوب معالجتها.
- `parent` — الأب في device hierarchy.

**القيمة المُعادة:** 0 نجاح، أو كود خطأ سالب.

**تفاصيل مهمة:**
- تُطلق `of_node_put(root)` دائماً قبل العودة.
- **مهجورة** لـ new board support — استخدم `of_platform_populate()` بدلاً عنها.
- لا تدعم `of_dev_auxdata` lookup (تُمرّر `NULL` دائماً).

---

#### `of_platform_populate()`

```c
int of_platform_populate(struct device_node *root,
                          const struct of_device_id *matches,
                          const struct of_dev_auxdata *lookup,
                          struct device *parent);
```

**ما تفعله:** الواجهة الحديثة والمُفضَّلة. تُكرر على أطفال `root` المباشرين (لا على `root` نفسه) وتستدعي `of_platform_bus_create()` بـ `strict=true`. تستخدم `device_links_supplier_sync_state_pause/resume()` لتأجيل sync_state حتى اكتمال الـ populate.

**المعاملات:**
- `root` — جذر الشجرة الجزئية؛ `NULL` لمسح كامل الـ DT.
- `matches` — جدول المطابقة للـ bus nodes؛ `NULL` لمطابقة الافتراضي.
- `lookup` — جدول `of_dev_auxdata` لتجاوز الأسماء؛ يُقبَل `NULL`.
- `parent` — الأب للأجهزة المُنشَأة.

**القيمة المُعادة:** 0 نجاح، أو كود خطأ سالب.

**تفاصيل مهمة:**
- تضبط `OF_POPULATED_BUS` على `root` نفسه بعد اكتمال المسح.
- الـ `device_links_supplier_sync_state_pause()` يمنع `sync_state` callbacks من الاستدعاء قبل اكتمال كل consumers — ضروري لـ power management صحيح.
- تُطلق `of_node_put(root)` دائماً.

---

#### `of_platform_default_populate()`

```c
int of_platform_default_populate(struct device_node *root,
                                   const struct of_dev_auxdata *lookup,
                                   struct device *parent);
```

**ما تفعله:** تستدعي `of_platform_populate()` مع جدول مطابقة ثابت يتضمن: `simple-bus`، `simple-mfd`، `isa`، و `arm,amba-bus` (عند `CONFIG_ARM_AMBA`). هذه هي الـ compatibles القياسية للـ buses في الـ DT الحديث.

**تفاصيل مهمة:** تُستدعى مباشرة من `of_platform_default_populate_init()` عبر `arch_initcall_sync` عند boot لملء كامل الـ device tree.

---

### المجموعة 5: Cleanup / Depopulate

**الغرض:** إزالة الأجهزة المُنشَأة من الـ DT بشكل آمن، بما في ذلك تنظيف الـ flags على الـ nodes.

---

#### `of_platform_device_destroy()`

```c
int of_platform_device_destroy(struct device *dev, void *data);
```

**ما تفعله:** تتحقق من أن الـ device أُنشئ من الـ DT عبر `OF_POPULATED` flag. إذا كان `OF_POPULATED_BUS` مضبوطاً، تكرر التدمير على الأطفال أولاً عبر `device_for_each_child()`. تمسح الـ flags ثم تستدعي `platform_device_unregister()` أو `amba_device_unregister()` حسب نوع الـ bus.

**المعاملات:**
- `dev` — الـ `struct device` المُراد تدميره.
- `data` — غير مستخدم (يُمرَّر `NULL` أو pointer لـ `bool` في بعض السياقات).

**القيمة المُعادة:** دائماً 0 (ليُستخدَم مع `device_for_each_child()`).

**pseudocode:**

```c
of_platform_device_destroy(dev, data):
    if !dev->of_node OR !OF_POPULATED(dev->of_node):
        return 0   // not ours

    if OF_POPULATED_BUS(dev->of_node):
        device_for_each_child(dev, NULL, of_platform_device_destroy)  // recurse

    clear_flag(OF_POPULATED)
    clear_flag(OF_POPULATED_BUS)

    if dev->bus == platform_bus_type:
        platform_device_unregister(to_platform_device(dev))
    elif dev->bus == amba_bustype:
        amba_device_unregister(to_amba_device(dev))

    return 0
```

**تفاصيل مهمة:**
- الـ recursion تسير من الأطفال إلى الأب (depth-first) لأن `device_for_each_child` يُكمل قبل `unregister` الأب.
- لا تلمس الأجهزة التي ليس لها `of_node` أو لم يُضبط `OF_POPULATED` عليها — تحمي الأجهزة اليدوية.

---

#### `of_platform_depopulate()`

```c
void of_platform_depopulate(struct device *parent);
```

**ما تفعله:** تتحقق من أن `parent->of_node` مضبوط عليه `OF_POPULATED_BUS`، ثم تستدعي `device_for_each_child_reverse()` مع `of_platform_device_destroy()` لإزالة الأطفال بترتيب عكسي (LIFO)، ثم تمسح `OF_POPULATED_BUS` من الـ parent node.

**المعاملات:**
- `parent` — الـ device الأب الذي سيُزال أطفاله.

**القيمة المُعادة:** void.

**تفاصيل مهمة:**
- `device_for_each_child_reverse()` يضمن إزالة الأطفال الأحدث أولاً — يُقلل من مشاكل dependencies.
- تُستدعى من `devm_of_platform_populate_release()` تلقائياً.
- الـ `parent` نفسه لا يُزال — فقط أطفاله.

---

### المجموعة 6: Devres-Managed Lifecycle

**الغرض:** ربط دورة حياة الـ OF population بدورة حياة الـ driver عبر آلية **devres**، مما يُلغي الحاجة لاستدعاء `depopulate` يدوياً في `remove()`.

---

#### `devm_of_platform_populate()`

```c
int devm_of_platform_populate(struct device *dev);
```

**ما تفعله:** تُخصّص devres handle بحجم `sizeof(struct device *)` عبر `devres_alloc()` مع release function = `devm_of_platform_populate_release`. تستدعي `of_platform_populate(dev->of_node, NULL, NULL, dev)`. عند النجاح تُسجّل الـ handle في `devres` list الخاص بـ `dev`.

**المعاملات:**
- `dev` — الـ device الـ owner؛ `dev->of_node` يُستخدَم كـ root للـ populate.

**القيمة المُعادة:** 0 نجاح، `-EINVAL` إذا كان `dev` هو `NULL`، `-ENOMEM` عند فشل الـ allocation، أو كود خطأ من `of_platform_populate()`.

**تفاصيل مهمة:**
- عند unbind الـ driver، الـ devres framework يستدعي `devm_of_platform_populate_release()` تلقائياً والتي تستدعي `of_platform_depopulate()`.
- لا يدعم `lookup` table — للـ use cases البسيطة فقط.
- مثالية لـ MFD drivers أو composite drivers التي تُنشئ child devices.

**مثال استخدام:**

```c
static int my_mfd_probe(struct platform_device *pdev)
{
    /* populate all DT children as platform devices */
    return devm_of_platform_populate(&pdev->dev);
    /* no need for remove() — devres handles cleanup */
}
```

---

#### `devm_of_platform_depopulate()`

```c
void devm_of_platform_depopulate(struct device *dev);
```

**ما تفعله:** تستدعي `devres_release()` مع matcher function `devm_of_platform_match` للعثور على الـ devres handle المرتبط بـ `dev` وتحريره فوراً، أي تستدعي `of_platform_depopulate()` يدوياً قبل unbind الـ driver.

**المعاملات:**
- `dev` — الـ device الذي استدعى `devm_of_platform_populate()` سابقاً.

**القيمة المُعادة:** void؛ تُطلق `WARN_ON` إذا لم يوجد handle مُسجَّل.

**تفاصيل مهمة:** تُستخدَم عند الحاجة للـ depopulate قبل الـ unbind الطبيعي، مثلاً في مسار error recovery داخل `probe()`.

---

### المجموعة 7: Dynamic OF Notifier (CONFIG_OF_DYNAMIC)

**الغرض:** دعم إضافة وإزالة DT nodes في وقت التشغيل (live DT overlay) وتحويل هذه الأحداث تلقائياً إلى إنشاء/تدمير الـ platform devices.

---

#### `of_platform_notify()` (static)

```c
static int of_platform_notify(struct notifier_block *nb,
                               unsigned long action, void *arg);
```

**ما تفعله:** تُعالج أحداث `OF_RECONFIG_CHANGE_ADD` و `OF_RECONFIG_CHANGE_REMOVE` من الـ OF reconfig notifier chain.

- **ADD:** تتحقق من أن الأب هو root أو `OF_POPULATED_BUS`، تتحقق من أن الـ node غير مُسكَّن، ثم تستدعي `of_platform_device_create()` لإنشاء الـ device.
- **REMOVE:** تُعثر على الـ device عبر `of_find_device_by_node()` وتستدعي `of_platform_device_destroy()` ثم `platform_device_put()`.

**القيمة المُعادة:** `NOTIFY_OK` في الحالات العادية، `notifier_from_errno(-EINVAL)` عند فشل الإنشاء.

**تفاصيل مهمة:**
- تمسح `FWNODE_FLAG_NOT_DEVICE` من الـ fwnode قبل إنشاء الـ device لضمان صحة `fw_devlink` consumer tracking.
- لا تتعامل مع `arm,primecell` nodes ديناميكياً.

---

#### `of_platform_register_reconfig_notifier()`

```c
void of_platform_register_reconfig_notifier(void);
```

**ما تفعله:** تُسجّل `platform_of_notifier` في OF reconfig notifier chain عبر `of_reconfig_notifier_register()`.

**تفاصيل مهمة:** تُستدعى من `of_platform_default_populate_init()` ضمن `arch_initcall_sync`. `WARN_ON` تُطلق إذا فشل التسجيل.

---

### الـ Initialization Flow عند Boot

```
arch_initcall_sync
└── of_platform_default_populate_init()
    ├── device_links_supplier_sync_state_pause()
    ├── [PPC path] create display devices
    └── [non-PPC path]
        ├── populate /reserved-memory compatible nodes
        ├── populate /firmware children
        ├── create simple-framebuffer device (+ sysfb_disable)
        └── of_platform_default_populate(NULL, NULL, NULL)
            └── of_platform_populate(/, match_table, NULL, NULL)
                └── for each child of /:
                    └── of_platform_bus_create(child, ..., strict=true)
                        ├── of_platform_device_create_pdata()
                        └── [if matches] recurse into children

late_initcall_sync
└── of_platform_sync_state_init()
    └── device_links_supplier_sync_state_resume()
```

---

### الـ `struct of_dev_auxdata` والـ Macro `OF_DEV_AUXDATA`

**الـ struct:**

```c
struct of_dev_auxdata {
    char           *compatible;   // DT compatible string to match
    resource_size_t phys_addr;    // physical base address for disambiguation
    char           *name;         // override device name on bus
    void           *platform_data;// pointer injected into pdev->dev.platform_data
};
```

**الـ Macro:**

```c
#define OF_DEV_AUXDATA(_compat, _phys, _name, _pdata) \
    { .compatible = _compat, .phys_addr = _phys, \
      .name = _name, .platform_data = _pdata }
```

**متى تُستخدَم:** عند الحاجة لتعيين اسم محدد لـ device (مثلاً `"serial8250"` بدل الاسم التلقائي) أو حقن `platform_data` legacy في بيئة DT. هذا **last resort** — الطريقة الصحيحة هي قراءة البيانات مباشرة من الـ node في الـ driver.
## Phase 5: دليل الـ Debugging الشامل

> **النطاق:** subsystem الـ `of_platform` — تحويل nodes الـ Device Tree إلى `platform_device` objects.
> الملف المرجعي: `include/linux/of_platform.h`

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ **debugfs** يوفر رؤية مباشرة لشجرة الـ Device Tree الحية داخل الـ kernel.

| المسار | ما يعرضه | كيفية القراءة |
|---|---|---|
| `/sys/kernel/debug/of_reserved_mem` | reserved memory regions المُحجوزة من DT | `cat` أو `ls` |
| `/sys/kernel/debug/device-tree/` | شجرة DT كاملة node بـ node | `ls -la` ثم `cat` لكل property |
| `/sys/kernel/debug/clk/` | حالة الـ clocks المرتبطة بـ platform devices | `cat /sys/kernel/debug/clk/<name>/clk_summary` |
| `/sys/kernel/debug/regulator/` | regulators المرتبطة بالـ nodes | `cat /sys/kernel/debug/regulator/regulator_summary` |

```bash
# قراءة شجرة DT كاملة من debugfs
find /sys/kernel/debug/device-tree/ -type f | while read f; do
    echo "=== $f ==="; cat "$f" 2>/dev/null | strings; echo
done

# قراءة خصائص node معين (مثال: node اسمه "serial@ff000000")
cat /sys/kernel/debug/device-tree/soc/serial@ff000000/compatible
```

**مثال على الخرج:**
```
=== /sys/kernel/debug/device-tree/soc/serial@ff000000/compatible ===
snps,dw-apb-uart
```

---

#### 2. مدخلات الـ sysfs

الـ **sysfs** يعرض الـ platform devices بعد تسجيلها بواسطة `of_platform_populate()`.

| المسار | الوصف |
|---|---|
| `/sys/bus/platform/devices/` | قائمة بجميع الـ platform devices المسجلة |
| `/sys/bus/platform/devices/<dev>/of_node` | symlink لـ DT node الخاص بالجهاز |
| `/sys/bus/platform/devices/<dev>/driver` | symlink للـ driver المرتبط |
| `/sys/bus/platform/devices/<dev>/modalias` | الـ modalias المستخدم لتحميل الـ module |
| `/sys/bus/platform/drivers/<drv>/` | قائمة الأجهزة المرتبطة بالـ driver |
| `/sys/firmware/devicetree/base/` | شجرة DT الكاملة من منظور الـ firmware |

```bash
# فحص كل platform devices وهل لها DT node
for dev in /sys/bus/platform/devices/*; do
    node=$(readlink -f "$dev/of_node" 2>/dev/null)
    echo "$(basename $dev) → $node"
done

# التحقق من compatible string لجهاز معين
cat /sys/bus/platform/devices/ff000000.serial/of_node/compatible

# مقارنة modalias مع compatible في DT
cat /sys/bus/platform/devices/ff000000.serial/modalias
# الخرج: of:Nserial@ff000000T<NULL>Csnps,dw-apb-uart
```

---

#### 3. الـ ftrace — tracepoints وأحداث الـ OF Platform

```bash
# عرض الـ tracepoints المتعلقة بالـ device tree والـ platform
grep -r "of\|platform\|devtree" /sys/kernel/debug/tracing/available_events | head -40

# تفعيل أحداث تتبع populating الـ platform devices
echo 1 > /sys/kernel/debug/tracing/events/of/of_bus_match_disable/enable
echo 1 > /sys/kernel/debug/tracing/events/platform/platform_match/enable
echo 1 > /sys/kernel/debug/tracing/events/platform/platform_driver_register/enable
echo 1 > /sys/kernel/debug/tracing/events/device/bus_add_device/enable

# تفعيل function tracer لتتبع of_platform_populate
echo function > /sys/kernel/debug/tracing/current_tracer
echo "of_platform_populate of_platform_bus_probe of_device_alloc" \
    > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تشغيل السيناريو الذي يستدعي populate (مثلاً rebind للـ driver)
echo ff000000.serial > /sys/bus/platform/drivers/dw-apb-uart/unbind
echo ff000000.serial > /sys/bus/platform/drivers/dw-apb-uart/bind

# قراءة النتيجة
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**مثال على خرج الـ ftrace:**
```
# tracer: function
#
kworker/0:1-45    [000] ....  123.456789: of_platform_populate <-of_platform_default_populate_init
kworker/0:1-45    [000] ....  123.456800: of_device_alloc <-of_platform_device_create_pdata
kworker/0:1-45    [000] ....  123.456810: of_device_add <-of_platform_device_create_pdata
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# تفعيل dynamic debug لملفات of_platform
echo "file drivers/of/platform.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/of/device.c +p"  >> /sys/kernel/debug/dynamic_debug/control

# تفعيل لكل الـ of subsystem
echo "module of +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع timestamps وأسماء الـ functions
echo "file drivers/of/platform.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# إيقاف التفعيل لاحقاً
echo "file drivers/of/platform.c -p" > /sys/kernel/debug/dynamic_debug/control

# رؤية ما هو مفعّل حالياً
grep "of/platform" /sys/kernel/debug/dynamic_debug/control
```

**لتشغيل وقت الـ boot** — أضف إلى kernel command line:
```
dyndbg="file drivers/of/platform.c +p; file drivers/of/device.c +p"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الخيار | الوظيفة |
|---|---|
| `CONFIG_OF` | تفعيل دعم الـ Device Tree أصلاً |
| `CONFIG_OF_ADDRESS` | دعم `of_platform_populate` وإنشاء الأجهزة |
| `CONFIG_OF_DYNAMIC` | دعم التعديل الديناميكي لشجرة DT |
| `CONFIG_OF_UNITTEST` | unit tests لـ OF subsystem (مفيد للتطوير) |
| `CONFIG_OF_OVERLAY` | دعم DT overlays لـ dynamic node injection |
| `CONFIG_OF_KOBJ` | ربط DT nodes بـ kobjects لـ sysfs |
| `CONFIG_DEBUG_DRIVER` | تفعيل رسائل debug في driver core |
| `CONFIG_DEBUG_DEVRES` | تتبع managed resources (devm_*) |
| `CONFIG_PROVE_LOCKING` | كشف مشاكل الـ locking في populate |
| `CONFIG_KASAN` | كشف memory corruption في بنيات OF |
| `CONFIG_UBSAN` | كشف undefined behavior |
| `CONFIG_DYNAMIC_DEBUG` | prerequisite لـ dynamic debug |

```bash
# التحقق من الخيارات المفعلة
zcat /proc/config.gz | grep -E "CONFIG_OF|CONFIG_DEBUG_DRIVER|CONFIG_DEBUG_DEVRES"
```

---

#### 6. أدوات خاصة بالـ Subsystem

**أداة `dtc` (Device Tree Compiler):**
```bash
# تفكيك DT blob من الـ kernel إلى صيغة قابلة للقراءة
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | less

# أو من ملف dtb مباشرة
dtc -I dtb -O dts /boot/dtbs/$(uname -r)/platform.dtb | less
```

**أداة `fdtdump`:**
```bash
# تفريغ DTB header وتحليل البنية
fdtdump /boot/dtbs/$(uname -r)/platform.dtb | head -50
```

**أداة `of_node_get/put` tracing عبر kprobes:**
```bash
# تتبع reference counting لـ device_node
echo 'p:of_get of_node_get np=%di' >> /sys/kernel/debug/tracing/kprobe_events
echo 'p:of_put of_node_put np=%di' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/of_get/enable
echo 1 > /sys/kernel/debug/tracing/events/kprobes/of_put/enable
```

**سكريبت للتحقق من الـ platform devices غير المرتبطة بـ driver:**
```bash
#!/bin/bash
echo "=== Platform devices without driver ==="
for dev in /sys/bus/platform/devices/*; do
    if [ ! -L "$dev/driver" ]; then
        name=$(basename "$dev")
        compat=$(cat "$dev/of_node/compatible" 2>/dev/null | tr '\0' ' ')
        echo "UNBOUND: $name  compatible='$compat'"
    fi
done
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `of_platform_populate: NULL root` | استدعاء `of_platform_populate()` بـ root=NULL مع غياب `of_root` | التحقق من تحميل DTB بشكل صحيح — `dmesg | grep "DT"` |
| `OF: unrecognized device: ...` | الـ node موجود لكن لا يوجد driver يدعم الـ compatible | تحميل الـ module المناسب أو تفعيل `CONFIG_*` |
| `platform ff000000.serial: Driver dw-apb-uart requests probe deferral` | الـ driver يطلب موردًا غير جاهز بعد (clock, regulator) | انتظار probe deferral أو فحص ترتيب initcall |
| `of_device_add: bus 'platform': add device ff000000.serial failed` | تعارض في اسم الجهاز أو فشل sysfs creation | `dmesg -T | grep "add device"` ثم فحص `/sys/bus/platform/devices/` |
| `of_platform_device_create: alloc failed` | فشل kmalloc لـ `platform_device` | مشكلة memory — فحص `dmesg | grep "Out of memory"` |
| `of_find_device_by_node: not found` | الـ device_node لم يتحول لـ platform_device بعد | التأكد أن `of_platform_populate()` استُدعي قبل الاستخدام |
| `devm_of_platform_populate: failed` | `of_platform_populate()` أعاد خطأً في managed context | فحص قيمة الإرجاع وتتبعها مع `CONFIG_DEBUG_DEVRES` |
| `of_device_register failed` | فشل في `device_register()` | فحص dmesg للـ kobject error أو تعارض الأسماء |
| `platform: of_platform_default_populate: -19` | الـ root node غير متوفر (`-ENODEV`) | `CONFIG_OF` غير مفعّل أو DTB لم يُحمَّل |
| `ERROR: Bad of_node_put() on ... ` | خطأ في reference counting — put بدون get | تفعيل `CONFIG_OF_DYNAMIC` وتتبع kprobe |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في of_platform_device_create_pdata() — للتحقق من صحة المدخلات */
static struct platform_device *of_platform_device_create_pdata(
    struct device_node *np, const char *bus_id,
    void *platform_data, struct device *parent)
{
    struct platform_device *dev;

    /* نقطة 1: التحقق من أن الـ node لم يُحول مسبقاً */
    WARN_ON(of_find_device_by_node(np) != NULL);

    dev = of_device_alloc(np, bus_id, parent);
    if (!dev)
        return NULL;

    /* نقطة 2: التحقق من reference count الـ node */
    WARN_ON(!of_node_get(np));
    of_node_put(np);  /* compensate */

    if (of_device_add(dev) != 0) {
        /* نقطة 3: dump_stack عند فشل الإضافة لمعرفة caller */
        dump_stack();
        platform_device_put(dev);
        return NULL;
    }
    return dev;
}

/* في of_platform_populate() — للتحقق من اكتمال الـ populate */
int of_platform_populate(struct device_node *root, ...)
{
    /* نقطة 4: WARN_ON إذا استُدعيت مرتين على نفس الـ root */
    WARN_ON(root && of_find_device_by_node(root));
    ...
}
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

المبدأ الأساسي: ما يصفه DT يجب أن يتطابق مع ما هو موجود فعلياً على الـ hardware.

```bash
# فحص الـ memory regions المسجلة لـ platform device
cat /proc/iomem | grep -i "serial\|uart\|spi\|i2c"
# الخرج المتوقع:
# ff000000-ff000fff : ff000000.serial
# ff000000-ff000fff :   dw-apb-uart

# مقارنة مع ما في DT
cat /sys/bus/platform/devices/ff000000.serial/of_node/reg | hexdump -C
# يجب أن يتطابق العنوان ff000000 مع /proc/iomem

# فحص IRQ المسجل
cat /proc/interrupts | grep ff000000
# مقارنة مع DT
cat /sys/bus/platform/devices/ff000000.serial/of_node/interrupts | hexdump -C
```

---

#### 2. تقنيات الـ Register Dump

```bash
# باستخدام devmem2 (يتطلب root وCONFIG_STRICT_DEVMEM=n)
# قراءة 4 bytes من عنوان base الجهاز
devmem2 0xff000000 w
# الخرج: /dev/mem opened. Memory mapped at address 0x7f8a000000. Value at address 0xFF000000: 0x00C20100

# باستخدام /dev/mem مباشرة مع dd
dd if=/dev/mem bs=4 count=16 skip=$((0xff000000/4)) 2>/dev/null | hexdump -C

# باستخدام أداة io (من package i2c-tools أو مستقلة)
io -4 -r 0xff000000

# قراءة مجال كامل من registers
for offset in $(seq 0 4 64); do
    addr=$((0xff000000 + offset))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}')
    printf "0xff%06x = %s\n" $offset "$val"
done
```

**ملاحظة:** على أنظمة حديثة مع `CONFIG_STRICT_DEVMEM=y`، استخدم بدلاً من ذلك:
```bash
# من خلال driver مؤقت أو عبر JTAG
# أو تعطيل مؤقت: echo 0 > /proc/sys/kernel/strict_devmem (غير متاح دائماً)
```

---

#### 3. نصائح Logic Analyzer وOscilloscope

```
منهجية التحقق مع Logic Analyzer:

     SoC                    Device
      |                        |
  [UART TX] ──────────────► [RX]
  [UART RX] ◄────────────── [TX]
      |                        |
  Measure:                 Verify:
  - Baud rate              - Signal levels (3.3V/1.8V)
  - Start/Stop bits        - Termination
  - Parity                 - Cable continuity
```

| الحالة | ما تقيسه | ما تبحث عنه |
|---|---|---|
| الـ device لا يستجيب | CLK/DATA lines لـ I2C/SPI | هل هناك إشارة؟ هل الـ ACK موجود؟ |
| Probe deferral دائم | Power rails (VDD) | هل الـ regulator يوصل فعلاً؟ |
| interrupt لا يصل | IRQ line | هل المستوى يتغير عند الحدث؟ |
| بيانات خاطئة | Data bus + CLK | تزامن الـ sampling مع الـ clock edge |

```bash
# بعد قياس الـ hardware، مقارنة مع kernel driver state
cat /sys/bus/platform/devices/ff000000.serial/of_node/clock-frequency
# يجب يتطابق مع الـ oscilloscope measurement
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة الـ Hardware | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| عنوان خاطئ في DT | `ff000000.serial: iomem region not reachable` | مقارنة `reg` في DT مع map الـ SoC الفعلي |
| IRQ line لا يصل لـ GIC | `ff000000.serial: no irq found` | قياس الـ IRQ line مع logic analyzer |
| Clock لم يُفعَّل | `dw-apb-uart: unable to get clk` | `cat /sys/kernel/debug/clk/uart_clk/clk_enable_count` |
| Power domain إيقاف | `runtime suspend failed` | قياس VDD على الـ device مع multimeter |
| Reset line لا تُفرج | `device stuck in reset` | قياس RESET# line — هل ترتفع بعد boot؟ |
| عدم تطابق الـ voltage | بيانات مشوهة / CRC errors | oscilloscope على الـ data lines — voltage levels |

---

#### 5. تصحيح الـ Device Tree — التحقق من تطابقه مع الـ Hardware

**أدوات التحقق:**

```bash
# 1. قراءة DT المحمّل فعلياً (ما يراه الـ kernel)
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/live_dt.dts
less /tmp/live_dt.dts

# 2. مقارنة DTB المُدمج مع DTS المصدري
dtc -I dtb -O dts /boot/dtbs/$(uname -r)/rockchip/rk3399.dtb > /tmp/compiled_dt.dts
diff /tmp/live_dt.dts /tmp/compiled_dt.dts

# 3. التحقق من compatible string يتطابق مع driver
grep -r "of_device_id\|compatible" drivers/tty/serial/8250/8250_dw.c | head -20
# يجب أن تجد "snps,dw-apb-uart" في MODULE_DEVICE_TABLE

# 4. التحقق من وجود node في الشجرة الحية
ls /sys/firmware/devicetree/base/soc/serial@ff000000/
# إذا غاب الـ node، المشكلة في DTB وليس في الـ driver

# 5. فحص الـ status property
cat /sys/firmware/devicetree/base/soc/serial@ff000000/status
# يجب أن يكون: "okay" وليس "disabled"
```

**سكريبت تشخيص DT كامل:**
```bash
#!/bin/bash
# dt_check.sh — يتحقق من جميع platform devices ويقارن DT مع kernel state

echo "=== DT vs Kernel Platform Devices Report ==="
echo "Date: $(date)"
echo ""

# الأجهزة الموجودة في DT
echo "--- Nodes in live DT (status=okay) ---"
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | \
    grep -A2 "status = " | grep -B1 "okay" | grep "@" | tr -d ':'

echo ""
echo "--- Registered platform_devices ---"
ls /sys/bus/platform/devices/

echo ""
echo "--- Unbound devices (no driver) ---"
for dev in /sys/bus/platform/devices/*; do
    [ ! -L "$dev/driver" ] && echo "  UNBOUND: $(basename $dev)"
done
```

---

### Practical Commands

#### مجموعة الأوامر الجاهزة

**#1 — فحص شامل للحالة الأولية:**
```bash
# حالة كل platform devices
ls -la /sys/bus/platform/devices/ | wc -l  # عدد الأجهزة
ls /sys/bus/platform/devices/ | head -20   # أول 20 جهاز

# فحص DT root
ls /sys/firmware/devicetree/base/

# فحص OF subsystem messages في dmesg
dmesg -T | grep -E "OF|of_platform|platform.*probe|DT" | head -30
```

**#2 — تشخيص جهاز معين:**
```bash
DEV="ff000000.serial"  # غيّر هذا

echo "=== Device: $DEV ==="
echo "--- DT node ---"
ls /sys/bus/platform/devices/$DEV/of_node/

echo "--- compatible ---"
cat /sys/bus/platform/devices/$DEV/of_node/compatible | tr '\0' '\n'

echo "--- reg (base address) ---"
hexdump -C /sys/bus/platform/devices/$DEV/of_node/reg

echo "--- Driver bound ---"
readlink /sys/bus/platform/devices/$DEV/driver

echo "--- Resources ---"
cat /proc/iomem | grep $DEV
cat /proc/interrupts | grep $DEV
```

**#3 — تتبع of_platform_populate في الوقت الحقيقي:**
```bash
# تفعيل الـ tracing
echo nop > /sys/kernel/debug/tracing/current_tracer
echo "of_platform_populate" > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# إعادة populate (مثال: عبر module reload)
modprobe -r my_platform_driver
modprobe my_platform_driver

# قراءة النتيجة
cat /sys/kernel/debug/tracing/trace | grep of_platform
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**#4 — تفعيل dynamic debug وقراءته:**
```bash
echo "file drivers/of/platform.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/of/device.c +pflmt"   > /sys/kernel/debug/dynamic_debug/control

# مراقبة الرسائل مباشرة
dmesg -w | grep -E "of_platform|of_device"
```

**#5 — فحص DT overlay (إذا مفعّل):**
```bash
# قراءة overlays الحالية
ls /sys/kernel/config/device-tree/overlays/ 2>/dev/null

# تطبيق overlay للاختبار
mkdir /sys/kernel/config/device-tree/overlays/test
cat my_overlay.dtbo > /sys/kernel/config/device-tree/overlays/test/dtbo

# فحص النتيجة
dmesg | tail -20
ls /sys/bus/platform/devices/ | diff - <(ls /sys/bus/platform/devices/)
```

**#6 — تفريغ كامل لحالة OF subsystem:**
```bash
#!/bin/bash
# of_platform_dump.sh — تقرير شامل

OUT="/tmp/of_platform_report_$(date +%Y%m%d_%H%M%S).txt"

{
echo "=== OF Platform Debug Report ==="
echo "Kernel: $(uname -r)"
echo "Date: $(date)"
echo ""

echo "--- dmesg OF messages ---"
dmesg | grep -iE "of_platform|devicetree|DT:|fdt" | head -50
echo ""

echo "--- /proc/iomem ---"
cat /proc/iomem
echo ""

echo "--- Platform devices ---"
for dev in /sys/bus/platform/devices/*; do
    name=$(basename "$dev")
    driver=$(readlink "$dev/driver" 2>/dev/null | xargs basename 2>/dev/null || echo "NONE")
    compat=$(cat "$dev/of_node/compatible" 2>/dev/null | tr '\0' '|' || echo "no-dt-node")
    printf "%-40s driver=%-20s compatible=%s\n" "$name" "$driver" "$compat"
done
echo ""

echo "--- Kernel OF config ---"
zcat /proc/config.gz 2>/dev/null | grep -E "^CONFIG_OF" || \
    grep -E "^CONFIG_OF" /boot/config-$(uname -r) 2>/dev/null

} | tee "$OUT"

echo ""
echo "Report saved to: $OUT"
```

**تفسير الخرج المتوقع:**
```
=== OF Platform Debug Report ===
Kernel: 6.6.0-rockchip
Date: Sun Feb 22 10:00:00 UTC 2026

--- dmesg OF messages ---
[    0.000123] OF: fdt: Ignoring memory range 0x00000000 - 0x200000
[    0.451234] of_platform_populate: created ff000000.serial
[    1.234567] dw-apb-uart ff000000.serial: detected caps 00000000 should be 00000000

--- Platform devices ---
ff000000.serial                          driver=dw-apb-uart           compatible=snps,dw-apb-uart
ff110000.i2c                             driver=NONE                   compatible=rockchip,rk3399-i2c
```

**تفسير الحالات:**
- `driver=dw-apb-uart` → الجهاز مرتبط بـ driver بنجاح
- `driver=NONE` → الـ driver غير محمّل، افحص `modprobe` أو `CONFIG_*`
- `compatible=no-dt-node` → الجهاز أُنشئ يدوياً بدون DT (legacy)
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بوابة صناعية على RK3562 — فشل تهيئة I2C عند الإقلاع

#### العنوان
**of_platform_populate** لا تُسجِّل أجهزة I2C على بوابة صناعية مبنية على RK3562.

#### السياق
شركة تصنع بوابة IoT صناعية بنظام Linux مدمج على SoC **RK3562**. البوابة تقرأ بيانات من حساسات تتصل عبر **I2C**. بعد ترقية kernel من 5.15 إلى 6.6، توقفت جميع أجهزة I2C عن الظهور في `/sys/bus/i2c/`.

#### المشكلة
الـ `platform_device` الخاص بـ I2C controller لم يُنشأ أثناء الإقلاع. السجلات تُظهر:

```
[    1.234] of_platform_populate: skipping node i2c@ff400000
```

#### التحليل
**`of_platform_populate()`** في `of_platform.h` تقبل جدول `matches` من نوع `struct of_device_id`. هذا الجدول يحدد أي nodes في Device Tree يجب تسجيلها كـ `platform_device`.

```c
extern int of_platform_populate(struct device_node *root,
                const struct of_device_id *matches,  /* جدول التصفية */
                const struct of_dev_auxdata *lookup,
                struct device *parent);
```

الـ `struct of_device_id` (من `mod_devicetable.h`) لها حقل `compatible` بحجم 128 بايت:

```c
struct of_device_id {
    char name[32];
    char type[32];
    char compatible[128];  /* يجب أن يطابق compatible في DTS */
    const void *data;
};
```

السبب: جدول `matches` في كود الـ SoC-specific init تغيّر في kernel 6.6. القيمة القديمة:

```c
/* kernel 5.15 */
static const struct of_device_id rk3562_of_match[] = {
    { .compatible = "rockchip,rk3562" },
    { }
};
```

بعد الترقية أصبح الاسم:

```c
/* kernel 6.6 */
static const struct of_device_id rk3562_of_match[] = {
    { .compatible = "rockchip,rk3562-soc" },  /* تغيّر الاسم */
    { }
};
```

لكن ملف DTS للمنتج لا يزال يحتوي على:

```dts
compatible = "rockchip,rk3562";  /* القيمة القديمة */
```

لذا `of_platform_populate()` لا تجد تطابقاً وتتجاهل الـ node.

#### الحل

**الخطوة 1 — تأكيد المشكلة:**

```bash
# تحقق من compatible في DTS المُترجَم
dtc -I dtb -O dts /sys/firmware/fdt | grep -A2 "i2c@ff400000"

# تحقق من matches المستخدمة
grep -r "of_platform_populate" arch/arm64/mach-rockchip/
```

**الخطوة 2 — تعديل ملف DTS:**

```dts
/* arch/arm64/boot/dts/rockchip/rk3562-gateway.dts */
/ {
    compatible = "mycompany,rk3562-gateway", "rockchip,rk3562-soc";
    /*                                        ^^^^^^^^^^^^^^^^^^^^
     * أضف القيمة الجديدة مع الإبقاء على القديمة للتوافق */
};
```

**الخطوة 3 — أو تحديث جدول matches ليقبل كلا القيمتين:**

```c
static const struct of_device_id rk3562_of_match[] = {
    { .compatible = "rockchip,rk3562-soc" },
    { .compatible = "rockchip,rk3562" },  /* للتوافق مع DTBs القديمة */
    { }
};
```

#### الدرس المستفاد
عند ترقية kernel، راجع جداول `of_device_id` في كود init الخاص بالـ SoC. أي تغيير في `compatible` بدون تحديث DTS يُصمت `of_platform_populate()` بالكامل — لا يظهر خطأ صريح، فقط الأجهزة لا تُسجَّل.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تعارض أسماء أجهزة USB

#### العنوان
استخدام **`of_dev_auxdata`** لحل تعارض أسماء **USB** controllers على Allwinner H616.

#### السياق
منتج Android TV Box يعمل على **Allwinner H616**. يحتوي على ثلاثة USB host controllers. كود قديم في BSP يستخدم اسم الجهاز `"sunxi-ehci"` للبحث عن الـ controller الأول ضمن مكتبة Android HAL. بعد الانتقال لـ mainline kernel، تغيرت أسماء الأجهزة تلقائياً إلى `"ehci-platform.0"` وما شابه.

#### المشكلة
Android HAL تفشل في إيجاد USB controller:

```
[HAL] usb_hal: cannot find device 'sunxi-ehci'
[HAL] USB host initialization failed
```

#### التحليل
الـ `of_platform_populate()` تُولِّد أسماء الأجهزة تلقائياً من الـ DT node. لكن **`struct of_dev_auxdata`** موجودة تحديداً لهذا الغرض — تجاوز الأسماء التلقائية:

```c
/**
 * struct of_dev_auxdata - lookup table entry for device names & platform_data
 * @compatible: compatible value of node to match against node
 * @phys_addr: Start address of registers to match against node
 * @name: Name to assign for matching nodes       <-- هنا الحل
 * @platform_data: platform_data to assign for matching nodes
 */
struct of_dev_auxdata {
    char *compatible;
    resource_size_t phys_addr;  /* عنوان فيزيائي لتمييز instances متعددة */
    char *name;
    void *platform_data;
};
```

الـ `phys_addr` مهم لأن H616 لديه 3 controllers على عناوين مختلفة، فيُستخدم للتمييز بينها.

#### الحل

**تعريف جدول auxdata في كود init الخاص بالمنصة:**

```c
/* arch/arm64/mach-sunxi/mach-h616.c */
#include <linux/of_platform.h>

/* عناوين فيزيائية لـ USB controllers على H616 */
#define H616_EHCI0_BASE  0x05101000
#define H616_EHCI1_BASE  0x05200000
#define H616_EHCI2_BASE  0x05311000

static const struct of_dev_auxdata h616_auxdata_lookup[] __initconst = {
    /* أعِد تسمية أول controller فقط ليطابق ما تتوقعه Android HAL */
    OF_DEV_AUXDATA("generic-ehci", H616_EHCI0_BASE, "sunxi-ehci", NULL),
    OF_DEV_AUXDATA("generic-ehci", H616_EHCI1_BASE, "sunxi-ehci.1", NULL),
    OF_DEV_AUXDATA("generic-ehci", H616_EHCI2_BASE, "sunxi-ehci.2", NULL),
    { }  /* نهاية الجدول */
};

static void __init h616_dt_init(void)
{
    of_platform_populate(NULL,
                         of_default_bus_match_table,
                         h616_auxdata_lookup,  /* مرر الجدول هنا */
                         NULL);
}
```

**التحقق:**

```bash
ls /sys/bus/platform/devices/ | grep sunxi-ehci
# sunxi-ehci
# sunxi-ehci.1
# sunxi-ehci.2
```

#### الدرس المستفاد
الـ `of_dev_auxdata` هو آلية "الملاذ الأخير" كما يقول التعليق في الكود. لكنه ضروري حين تعتمد طبقات فوق الـ kernel (كـ Android HAL) على أسماء أجهزة ثابتة. الـ `phys_addr` في البنية يحل مشكلة التمييز بين instances متعددة من نفس الـ compatible.

---

### السيناريو 3: لوحة تطوير i.MX8 — driver لا يعمل بسبب `CONFIG_OF_ADDRESS` غير مفعَّل

#### العنوان
**`of_platform_device_create()`** تُعيد دائماً `NULL` على نظام i.MX8 مخصص.

#### السياق
فريق bring-up لوحة مخصصة مبنية على **i.MX8MP** لتطبيق medical imaging. يحاول المهندس إنشاء `platform_device` يدوياً لـ HDMI encoder خارجي يتصل عبر I2C، لكن الدالة لا تعمل.

#### المشكلة
الكود التالي يُعيد دائماً `NULL`:

```c
struct platform_device *pdev;
pdev = of_platform_device_create(encoder_node, "imx8-hdmi-enc", parent);
if (!pdev) {
    dev_err(dev, "failed to create platform device\n");
    /* يصل دائماً هنا */
}
```

لا يوجد خطأ في kernel log.

#### التحليل
النظر في `of_platform.h` يكشف الحل مباشرة:

```c
#ifdef CONFIG_OF_ADDRESS
/* Platform devices and busses creation */
extern struct platform_device *of_platform_device_create(struct device_node *np,
                                           const char *bus_id,
                                           struct device *parent);
/* ... */
#else
/* Platform devices and busses creation */
static inline struct platform_device *of_platform_device_create(struct device_node *np,
                                    const char *bus_id,
                                    struct device *parent)
{
    return NULL;  /* <-- هذا ما يُنفَّذ عند غياب CONFIG_OF_ADDRESS */
}
#endif
```

كل دوال إنشاء الأجهزة (`of_platform_device_create`، `of_platform_populate`، `devm_of_platform_populate`) محاطة بـ `#ifdef CONFIG_OF_ADDRESS`. إذا لم يكن هذا الـ config مفعلاً، **كل هذه الدوال تُعيد `NULL` أو `-ENODEV` بصمت تام**.

**التحقق:**

```bash
grep "CONFIG_OF_ADDRESS" /boot/config-$(uname -r)
# CONFIG_OF_ADDRESS is not set  <-- هنا المشكلة
```

#### الحل

**في ملف Kconfig أو defconfig الخاص باللوحة:**

```
# arch/arm64/configs/imx8mp_medical_defconfig
CONFIG_OF=y
CONFIG_OF_ADDRESS=y       # ضروري لدوال of_platform_device_create و populate
CONFIG_OF_PLATFORM=y
```

أو عبر `menuconfig`:

```bash
make menuconfig
# Device Drivers →
#   Device Tree and Open Firmware support →
#     [*] Device tree and open firmware support
#     [*]   OF Address (CONFIG_OF_ADDRESS)
```

**بعد إعادة البناء:**

```bash
grep "CONFIG_OF_ADDRESS" /boot/config-$(uname -r)
# CONFIG_OF_ADDRESS=y

dmesg | grep "imx8-hdmi-enc"
# platform imx8-hdmi-enc: registered
```

#### الدرس المستفاد
الـ `#ifdef CONFIG_OF_ADDRESS` في `of_platform.h` يعني أن نصف API هذا الملف غير موجود بدون هذا الـ config. في بيئات bring-up مع defconfig مخصص، تحقق دائماً من هذا الـ config قبل debugging أعمق. الـ stub functions تُعيد قيماً "هادئة" (`NULL`، `-ENODEV`) بدون أي رسالة خطأ.

---

### السيناريو 4: ECU سيارة على AM62x — تسريب ذاكرة عند إزالة module

#### العنوان
استخدام `devm_of_platform_populate()` بدلاً من `of_platform_populate()` لتجنب memory leak في ECU مبنية على **AM62x**.

#### السياق
شركة تصنع **ECU (Electronic Control Unit)** للسيارات على TI **AM62x**. الـ ECU يدعم تحديث الـ firmware عبر USB. المهندس كتب driver يُنشئ أجهزة فرعية عبر `of_platform_populate()` ويُزيلها يدوياً، لكن اختبارات stress لتركيب/نزع الـ module المتكرر تكشف memory leak.

#### المشكلة

```bash
# بعد 50 دورة من rmmod/insmod
cat /proc/meminfo | grep MemFree
# MemFree يتناقص باستمرار

# kernel log يُظهر:
[  100.5] kobject: 'can-controller' (0xffff...) leaked, not cleaned up properly
```

#### التحليل
الكود الأصلي يستخدم:

```c
static int am62x_ecu_probe(struct platform_device *pdev)
{
    /* ... */
    ret = of_platform_populate(pdev->dev.of_node,
                               NULL, NULL,
                               &pdev->dev);
    /* يُنشئ platform_devices فرعية لـ CAN, SPI, UART */
    return ret;
}

static int am62x_ecu_remove(struct platform_device *pdev)
{
    of_platform_depopulate(&pdev->dev);  /* يُفترض أن يُنظِّف */
    return 0;
}
```

المشكلة: إذا `probe()` فشل في مرحلة لاحقة بعد `of_platform_populate()`، الـ `remove()` لن يُستدعى تلقائياً، فتبقى الأجهزة الفرعية بدون تنظيف.

**الحل الصحيح** موجود في `of_platform.h`:

```c
extern int devm_of_platform_populate(struct device *dev);
extern void devm_of_platform_depopulate(struct device *dev);
```

**الـ `devm_`** تعني أن التنظيف يحدث تلقائياً عند `device_release`، بغض النظر عن مسار الخروج.

#### الحل

```c
static int am62x_ecu_probe(struct platform_device *pdev)
{
    int ret;

    /* إعداد CAN interface */
    ret = am62x_can_init(pdev);
    if (ret)
        return ret;

    /* استخدم devm_ بدلاً من of_platform_populate */
    ret = devm_of_platform_populate(&pdev->dev);
    if (ret) {
        dev_err(&pdev->dev, "failed to populate child devices: %d\n", ret);
        return ret;
    }
    /* لا حاجة لـ of_platform_depopulate في remove()
     * devm framework يستدعيها تلقائياً */
    return 0;
}

static int am62x_ecu_remove(struct platform_device *pdev)
{
    /* devm_of_platform_depopulate تُستدعى تلقائياً */
    /* فقط نظّف الموارد غير الـ devm */
    am62x_can_deinit(pdev);
    return 0;
}
```

**التحقق:**

```bash
# اختبار 100 دورة
for i in $(seq 100); do
    modprobe am62x-ecu
    sleep 0.5
    modprobe -r am62x-ecu
    sleep 0.5
done

# تحقق من الذاكرة
grep MemFree /proc/meminfo  # يجب أن يبقى ثابتاً
dmesg | grep -i "leaked"    # لا يجب أن يظهر شيء
```

#### الدرس المستفاد
في kernel code، كل دالة لها نظيرة `devm_` هي الخيار الأفضل دائماً في `probe()`. الـ `devm_of_platform_populate()` تضمن أن child devices تُحذف في **كل** مسار خروج — سواء من `remove()` الطبيعي، أو من `probe()` الفاشل، أو من kernel panic recovery.

---

### السيناريو 5: حساس IoT على STM32MP1 — عدم ظهور SPI device في Device Tree

#### العنوان
**`of_find_device_by_node()`** تُعيد `NULL` لـ SPI sensor على لوحة **STM32MP1**.

#### السياق
فريق يطوّر حساس بيئي صغير (temperature/humidity) مبني على **STM32MP157C**. الحساس الرئيسي يتصل عبر **SPI**. driver الحساس يحتاج للحصول على الـ `platform_device` الخاص بـ SPI controller لإعداد DMA قبل أن يبدأ القراءة.

#### المشكلة

```c
struct device_node *spi_node;
struct platform_device *spi_pdev;

spi_node = of_find_node_by_path("/soc/spi@44004000");
spi_pdev = of_find_device_by_node(spi_node);
if (!spi_pdev) {
    pr_err("SPI platform device not found!\n");
    /* يصل هنا دائماً رغم أن node موجود في DT */
}
```

#### التحليل
الدالة في `of_platform.h`:

```c
#ifdef CONFIG_OF
extern struct platform_device *of_find_device_by_node(struct device_node *np);
#else
static inline struct platform_device *of_find_device_by_node(struct device_node *np)
{
    return NULL;
}
#endif
```

الدالة تبحث في قائمة `platform_devices` المُسجَّلة عن جهاز يرتبط بهذا الـ `device_node`. ترجع `NULL` في حالتين:

1. `CONFIG_OF` غير مفعَّل (نادر على STM32MP1).
2. **الجهاز لم يُسجَّل بعد** — أي أن `of_platform_populate()` لم تُعالج هذا الـ node بعد.

التحقيق كشف أن driver الحساس يعمل كـ **built-in** ويبدأ في مرحلة `arch_initcall`، بينما `of_platform_populate()` تُستدعى لاحقاً في مرحلة `device_initcall`. أي أن الـ SPI `platform_device` لم يُنشأ بعد عند محاولة البحث عنه.

```
arch_initcall:    sensor_driver_init()  ← يبحث عن SPI device (غير موجود بعد)
device_initcall:  of_platform_populate() ← ينشئ SPI platform_device
```

#### الحل

**الخيار 1 — تأخير البحث إلى مرحلة لاحقة:**

```c
/* غيِّر مستوى init من arch_initcall إلى late_initcall */
static int __init stm32_sensor_init(void)
{
    struct device_node *spi_node;
    struct platform_device *spi_pdev;

    spi_node = of_find_node_by_path("/soc/spi@44004000");
    if (!spi_node) {
        pr_err("SPI node not found in DT\n");
        return -ENODEV;
    }

    spi_pdev = of_find_device_by_node(spi_node);
    of_node_put(spi_node);  /* أطلق المرجع دائماً */

    if (!spi_pdev) {
        pr_err("SPI platform device not registered yet\n");
        return -EPROBE_DEFER;  /* أفضل: أطلب إعادة المحاولة */
    }
    /* ... */
    return 0;
}
late_initcall(stm32_sensor_init);  /* بدلاً من arch_initcall */
```

**الخيار 2 — استخدام `-EPROBE_DEFER`:**

```c
static int stm32_sensor_probe(struct platform_device *pdev)
{
    struct device_node *spi_node;
    struct platform_device *spi_pdev;

    spi_node = of_parse_phandle(pdev->dev.of_node, "spi-bus", 0);
    spi_pdev = of_find_device_by_node(spi_node);
    of_node_put(spi_node);

    if (!spi_pdev)
        return -EPROBE_DEFER;  /* kernel سيُعيد محاولة probe() لاحقاً */

    /* ... استمر في الإعداد */
    return 0;
}
```

**الخيار 3 — التحقق من تسجيل الجهاز:**

```bash
# تأكد أن SPI platform device مُسجَّل
ls /sys/bus/platform/devices/ | grep "spi@44004000"
# spi@44004000  ← يجب أن يظهر هذا قبل استخدام of_find_device_by_node

# أو تتبع ترتيب init
dmesg | grep -E "(spi@44004000|sensor)" | head -20
```

#### الدرس المستفاد
**`of_find_device_by_node()`** تبحث فقط في الأجهزة المُسجَّلة فعلياً — وجود الـ node في DT لا يكفي. على STM32MP1 وكل ARM SoC، الأجهزة تُنشأ في `of_platform_populate()` ضمن `device_initcall`. أي driver يستخدم `of_find_device_by_node()` يجب أن يعمل في مرحلة لاحقة، أو يُعيد `-EPROBE_DEFER` ليطلب إعادة المحاولة بعد اكتمال تهيئة الأجهزة الأخرى.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

هذه أهم المقالات التقنية المتعلقة بـ `of_platform` وربط الـ Device Tree بنموذج platform device في نواة Linux:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **Platform devices and device trees** | [lwn.net/Articles/448502](https://lwn.net/Articles/448502/) | الأساس — يشرح كيف يُنشئ `of_platform_populate()` الأجهزة من شجرة الجهاز |
| **The platform device API** | [lwn.net/Articles/448499](https://lwn.net/Articles/448499/) | يفسر بنية platform_device وعلاقتها بـ of_node |
| **Refactor and enhance device tree platform registrations** | [lwn.net/Articles/433779](https://lwn.net/Articles/433779/) | تتبع إعادة هيكلة `of_platform_bus_probe()` وإدخال `of_platform_populate()` |
| **Device-tree support for ARM** | [lwn.net/Articles/367752](https://lwn.net/Articles/367752/) | الجذر التاريخي — كيف بدأ دعم DT على ARM |
| **Device tree bindings** | [lwn.net/Articles/572114](https://lwn.net/Articles/572114/) | يشرح مفهوم الـ binding وكيف يربط الـ compatible string بالـ driver |
| **Full device tree support for ARM Versatile** | [lwn.net/Articles/447918](https://lwn.net/Articles/447918/) | مثال عملي كامل على استخدام `of_platform_populate()` على لوحة ARM Versatile |
| **Device tree troubles** | [lwn.net/Articles/560523](https://lwn.net/Articles/560523/) | تحليل مشكلات DT الشائعة ومنها إشكاليات التسمية وحالات الاستخدام الخاطئ لـ `of_dev_auxdata` |
| **Device-tree schemas** | [lwn.net/Articles/771621](https://lwn.net/Articles/771621/) | التحقق الرسمي من صحة bindings باستخدام YAML schemas |
| **platform: device tree support for early platform drivers** | [lwn.net/Articles/752744](https://lwn.net/Articles/752744/) | دعم الـ early platform drivers في الأنظمة المبنية على OF |

---

### التوثيق الرسمي في النواة

**الـ** `Documentation/` paths داخل شجرة النواة:

```
Documentation/devicetree/usage-model.rst
    يشرح نموذج الاستخدام الكامل للـ Device Tree في Linux،
    بما فيه دور of_platform_populate() وآلية simple-bus.

Documentation/devicetree/bindings/
    مجلد bindings الرسمي — كل ملف .yaml يصف compatible string
    واحدة ويُلزم المطورين بتوثيق جميع الخصائص.

Documentation/driver-api/driver-model/platform.rst
    يوثق نموذج platform device الكامل المترابط مع of_platform.

Documentation/core-api/device_link.rst
    يشرح آلية device links المستخدمة مع devm_of_platform_populate()
    لضمان ترتيب probe() الصحيح.
```

روابط الإصدار الإلكتروني:

- [Linux and the Devicetree — docs.kernel.org](https://docs.kernel.org/devicetree/usage-model.html)
- [DeviceTree Kernel API — docs.kernel.org](https://docs.kernel.org/devicetree/kernel-api.html)

---

### كومِتات النواة الجوهرية

أهم commits أدخلت أو غيّرت السلوك في `of_platform.h`:

| الوصف | الدلالة |
|-------|---------|
| تحويل `of_platform_bus_probe()` إلى `of_platform_populate()` | Grant Likely — أوجد الدالة التي أصبحت المرجع الأساسي |
| إضافة `devm_of_platform_populate()` | إدخال نمط devm لإدارة دورة حياة الأجهزة تلقائياً |
| إضافة `of_platform_default_populate()` | تبسيط استدعاء populate بدون matches مخصص |
| إضافة `struct of_dev_auxdata` و ماكرو `OF_DEV_AUXDATA` | آلية طوارئ للتوافق مع البنية التحتية القديمة |

للبحث في التاريخ:
```bash
git log --oneline --all -- drivers/of/platform.c
git log --oneline --all -- include/linux/of_platform.h
```

---

### نقاشات القوائم البريدية

- **devicetree@vger.kernel.org** — القائمة البريدية الرئيسية لكل ما يخص DT:
  [lore.kernel.org/linux-devicetree](https://lore.kernel.org/r/linux-devicetree/?t=20240305202442)

- **LKML أرشيف** — البحث عن `of_platform_populate`:
  [lkml.org](https://lkml.org/) ← ابحث عن `of_platform_populate`

- نقاش عملي على kernelnewbies حول عدم استدعاء `probe()` عند overlay الـ device tree:
  [lists.kernelnewbies.org — Platform driver probe() not being called](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2019-February/019811.html)

---

### مصادر eLinux.org

| الصفحة | الرابط |
|--------|--------|
| **Device Tree Reference** | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) |
| **Linux Drivers Device Tree Guide** | [elinux.org/Linux_Drivers_Device_Tree_Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) |
| **Device Trees** — مدخل عام | [elinux.org/Device_Trees](https://elinux.org/Device_Trees) |
| **Device Tree History** | [elinux.org/Device_tree_history](https://elinux.org/Device_tree_history) |
| **Device Tree Presentations** | [elinux.org/Device_Tree_Presentations](https://elinux.org/Device_Tree_Presentations) |
| **Device Tree Mysteries** | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model — يشرح kobject/kset وبنية platform device
- **الفصل 1**: يقدم نموذج الـ bus/device/driver الذي يبني عليه of_platform
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — platform_driver، bus_type، وآلية match
- يوضح بشكل عملي كيف يربط الـ kernel الـ device بالـ driver

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15**: Device Trees — يشرح FDT، dtc، وكيف يُحوَّل DTS إلى platform_device
- **الفصل 7**: يتناول بنية النواة على ARM وكيفية تهيئة البنية التحتية لـ of_platform

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يتعمق في بنية bus_type وكيف تُسجَّل الأجهزة عبر of_platform

---

### كلمات البحث الموصى بها

للبحث عن مزيد من المعلومات استخدم هذه المصطلحات:

```
of_platform_populate
of_platform_bus_probe
devm_of_platform_populate
of_dev_auxdata
platform_device of_node
simple-bus device tree
device tree platform driver binding
of_find_device_by_node
of_device_register
CONFIG_OF_ADDRESS
device tree flattened FDT Linux
open firmware platform Linux kernel
```

**للبحث في LKML:**
```bash
# البحث في lore.kernel.org
https://lore.kernel.org/linux-devicetree/?q=of_platform_populate

# البحث في git log
git log --all --grep="of_platform_populate" -- drivers/of/
```
## Phase 8: Writing simple module

### الهدف

سنضع **kprobe** على الدالة `of_platform_populate()` المُصدَّرة من kernel، وهي الدالة المسؤولة عن إنشاء `platform_device` لكل عقدة في Device Tree تحت جذر معين. اختُيرت هذه الدالة لأنها:

- مُصدَّرة بـ `EXPORT_SYMBOL` فيمكن للـ kprobe الوصول إليها.
- تُستدعى عند boot وعند تحميل بعض drivers — مما يجعل إخراجها مفيداً لتتبع متى يُبنى الـ platform bus.
- تحمل حججاً غنية بالمعلومات: عقدة الجذر ومصفوفة التطابق والأب.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_populate.c
 *
 * Hooks of_platform_populate() to log Device Tree population events.
 * Useful for understanding when and from where DT nodes become platform_devices.
 */

#include <linux/module.h>       /* module_init, module_exit, pr_info ... */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe ...   */
#include <linux/of_platform.h>  /* of_platform_populate signature       */
#include <linux/of.h>           /* struct device_node, full_name        */
#include <linux/device.h>       /* struct device, dev_name()            */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs <example@kernel.org>");
MODULE_DESCRIPTION("kprobe on of_platform_populate — logs DT population calls");

/* ---------------------------------------------------------------
 * pre_handler: يُستدعى قبل تنفيذ of_platform_populate مباشرةً.
 * regs يحمل سجلات المعالج — منها نستخرج الحجج وفق ABI (x86-64).
 * --------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * of_platform_populate(root, matches, lookup, parent)
     *
     * x86-64 calling convention:
     *   rdi = root   (struct device_node *)
     *   rsi = matches
     *   rdx = lookup
     *   rcx = parent (struct device *)
     */
    struct device_node *root   = (struct device_node *)regs->di;
    struct device      *parent = (struct device *)regs->cx;

    /* طباعة اسم عقدة الجذر واسم الجهاز الأب إن وُجدا */
    pr_info("kprobe[of_platform_populate]: root=\"%s\" parent=\"%s\"\n",
            root   ? root->full_name      : "(null)",
            parent ? dev_name(parent)     : "(null)");

    return 0; /* 0 = استمر في تنفيذ الدالة الأصلية */
}

/* ---------------------------------------------------------------
 * post_handler: يُستدعى بعد عودة of_platform_populate بنجاح.
 * نستخدمه فقط لتأكيد انتهاء العملية (اختياري لكن مفيد للتتبع).
 * --------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("kprobe[of_platform_populate]: returned, ax=%ld\n",
            (long)regs->ax); /* ax = قيمة الإرجاع (0 = نجاح) */
}

/* ---------------------------------------------------------------
 * تعريف الـ kprobe: نحدد الدالة بالاسم — الـ kernel يحسب العنوان.
 * --------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "of_platform_populate",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ---------------------------------------------------------------
 * module_init: نسجّل الـ kprobe عند تحميل الوحدة.
 * إن فشل التسجيل (مثلاً الدالة غير مُصدَّرة) نُخبر المستخدم.
 * --------------------------------------------------------------- */
static int __init kp_ofpop_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_of_populate: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_of_populate: planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit: إلغاء تسجيل الـ kprobe ضروري قبل تفريغ الوحدة،
 * وإلا سيقفز الـ kernel إلى كود محذوف → kernel panic.
 * --------------------------------------------------------------- */
static void __exit kp_ofpop_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_of_populate: removed\n");
}

module_init(kp_ofpop_init);
module_exit(kp_ofpop_exit);
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | الماكروات الأساسية لأي وحدة kernel |
| `linux/kprobes.h` | `struct kprobe` ودوال `register/unregister_kprobe` |
| `linux/of_platform.h` | تعريف `of_platform_populate` لنعرف نوع حججها |
| `linux/of.h` | `struct device_node` وحقل `full_name` |
| `linux/device.h` | `struct device` ودالة `dev_name()` |

---

#### `handler_pre` — الـ callback قبل التنفيذ

**الـ** `pt_regs` يمثّل حالة سجلات المعالج لحظة الاعتراض. على **x86-64** تتبع الحجج ترتيب `rdi → rsi → rdx → rcx`، لذا نقرأ `regs->di` للحصول على `root` و`regs->cx` للحصول على `parent`. طباعة `full_name` تُظهر أي فرع من Device Tree يجري populate-ه.

---

#### `handler_post` — الـ callback بعد التنفيذ

يُستدعى بعد عودة الدالة الأصلية؛ **الـ** `regs->ax` يحمل قيمة الإرجاع (`0` = نجاح، سالب = خطأ). هذا مفيد للتحقق من أن `of_platform_populate` لم تفشل صامتةً.

---

#### `struct kprobe kp`

**الـ** `symbol_name` يُخبر الـ kernel بالاسم الرمزي للدالة المستهدفة — يحسب العنوان الفعلي تلقائياً عبر `kallsyms`. لا نحتاج لتحديد العنوان يدوياً.

---

#### `module_init` / `module_exit`

`register_kprobe` تزرع **breakpoint** (أو `int3` على x86) عند بداية الدالة. `unregister_kprobe` في `module_exit` تُعيد البايت الأصلي وتُزيل الربط — إهمالها يؤدي إلى **kernel panic** عند تفريغ الوحدة لأن الـ kernel سيحاول القفز إلى handler محذوف.

---

### Makefile للبناء

```makefile
obj-m += kprobe_of_populate.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل واختبار

```bash
# بناء الوحدة
make

# تحميلها
sudo insmod kprobe_of_populate.ko

# مشاهدة الإخراج (إن حدث populate بعد التحميل)
sudo dmesg | grep kprobe

# تفريغها
sudo rmmod kprobe_of_populate

# مثال على إخراج متوقع
# [  42.123456] kprobe[of_platform_populate]: planted at ffffffff81a3bc20 (of_platform_populate)
# [  42.234567] kprobe[of_platform_populate]: root="/soc" parent="platform"
# [  42.234600] kprobe[of_platform_populate]: returned, ax=0
```

> **ملاحظة:** `of_platform_populate` غالباً تُستدعى في مرحلة boot المبكرة قبل تحميل أي وحدة خارجية، لذا قد لا ترى استدعاءات جديدة بعد التحميل إلا إذا شغّلت driver يستدعيها صريحةً (مثل بعض bus controllers). يمكن اختبار ذلك بـ `echo` إلى sysfs لإعادة فحص الـ DT على أنظمة تدعمه.
