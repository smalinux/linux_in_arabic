## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف وأين يقع في النواة؟

**الـ** `include/linux/of_irq.h` ينتمي إلى subsystem اسمه **Open Firmware / Flattened Device Tree (OF/FDT)**، والمسؤولون عنه Rob Herring وSaravana Kannan ضمن قائمة `OPEN FIRMWARE AND FLATTENED DEVICE TREE` في MAINTAINERS. هذا الـ subsystem يغطي كل الملفات التي تحمل البادئة `of_*`.

---

### القصة — لماذا يوجد هذا الملف أصلاً؟

تخيل أنك تبني جهازاً مدمجاً — راوتر، هاتف، لوحة Raspberry Pi. داخل هذا الجهاز يوجد عشرات المكونات: تحكم USB، شاشة، بطاقة شبكة، مستشعرات. كل واحد من هذه المكونات يحتاج أن "يصيح" على المعالج عندما يحدث شيء مهم — هذا الصياح هو **interrupt (IRQ)**.

المشكلة: كيف يعرف الـ kernel أي رقم IRQ يخص أي جهاز، وأي **interrupt controller** مسؤول عنه؟

في الأجهزة القديمة (x86) كانت الأرقام ثابتة معروفة مسبقاً. لكن في عالم المعالجات المدمجة (ARM، RISC-V، PowerPC) لا يوجد معيار — كل شركة تصنع أجهزتها بطريقة مختلفة.

الحل: **Device Tree** — ملف نصي يصف هيكل الجهاز كاملاً. يكتبه مصنّع اللوحة ويمرره إلى الـ kernel عند الإقلاع. داخله يوجد وصف دقيق: "هذا الجهاز يستخدم الـ interrupt رقم 32 من الـ controller الفلاني، وهو من نوع edge-triggered."

**الـ** `of_irq.h` هو الـ header الذي يعرّف الـ API الكامل لترجمة هذه المعلومات من الـ Device Tree إلى أرقام IRQ حقيقية يفهمها الـ kernel.

---

### المشكلة التي يحلها بالتفصيل

#### سيناريو حقيقي: لوحة ARM بها interrupt controllers متداخلة

```
Device Tree:
  gpio-controller@1000 {
      interrupt-parent = <&gic>;   /* GIC هو interrupt controller الرئيسي */
      interrupts = <0 32 4>;       /* domain=0, hwirq=32, type=IRQ_TYPE_LEVEL_HIGH */
  };

  i2c@2000 {
      interrupt-parent = <&gpio>;  /* gpio-controller هو interrupt parent */
      interrupts = <5 1>;          /* gpio pin 5, edge */
  };
```

الـ i2c يرتبط بـ gpio، والـ gpio يرتبط بـ GIC. هذا ما يسمى **interrupt hierarchy** أو **chained interrupts**. الـ `of_irq.h` يوفر الأدوات لاجتياز هذه الشجرة وترجمتها.

---

### ما الذي يعرّفه هذا الملف؟

#### 1. الـ `of_irq_init_cb_t` — callback لتهيئة interrupt controllers

```c
typedef int (*of_irq_init_cb_t)(struct device_node *, struct device_node *);
```
نوع دالة يُستخدم عند تهيئة كل interrupt controller موجود في الـ Device Tree أثناء الإقلاع.

#### 2. الـ `struct of_imap_parser` و `struct of_imap_item` — محلل interrupt-map

```c
struct of_imap_parser {
    struct device_node *node;    /* العقدة الحالية في الشجرة */
    const __be32 *imap;          /* مؤشر لبداية جدول interrupt-map */
    const __be32 *imap_end;      /* مؤشر لنهاية الجدول */
    u32 parent_offset;
};

struct of_imap_item {
    struct of_phandle_args parent_args;  /* معلومات الـ parent controller */
    u32 child_imap_count;
    u32 child_imap[16];                  /* أرقام الـ child interrupts */
};
```

الـ **`interrupt-map`** هو جدول في Device Tree يُستخدم في الأجهزة المعقدة مثل PCI bridges — يُعيّن كل interrupt من الطفل إلى interrupt مختلف في الأب.

#### 3. دوال الترجمة الأساسية

| الدالة | الغرض |
|--------|--------|
| `of_irq_parse_one()` | تقرأ `interrupts` property للجهاز رقم `index` وتعيد `of_phandle_args` |
| `of_irq_parse_raw()` | تحلل بيانات interrupt الخام من الـ Device Tree |
| `irq_of_parse_and_map()` | الدالة الأهم — تقرأ وتترجم وتعيد رقم Linux IRQ مباشرة |
| `irq_create_of_mapping()` | تنشئ mapping بين hwirq وLinux virq عبر الـ irq_domain |
| `of_irq_count()` | تعد كم interrupt يملك الجهاز |
| `of_irq_get()` | تجلب رقم IRQ بالإندكس |
| `of_irq_get_byname()` | تجلب IRQ باسمه من `interrupt-names` property |
| `of_irq_to_resource()` | تحوّل IRQ إلى `struct resource` لاستخدامه في drivers |
| `of_irq_find_parent()` | تبحث عن الـ interrupt controller الأب |
| `of_irq_init()` | تهيئ جميع interrupt controllers في الـ Device Tree |

#### 4. دوال الـ MSI (Message Signaled Interrupts)

```c
struct irq_domain *of_msi_get_domain(struct device *dev,
                                     const struct device_node *np,
                                     enum irq_domain_bus_token token);
void of_msi_configure(struct device *dev, const struct device_node *np);
u32 of_msi_xlate(struct device *dev, struct device_node **msi_np, u32 id_in);
```

الـ **MSI** آلية حديثة تستخدمها PCIe devices — بدل سلك interrupt مادي، يكتب الجهاز في عنوان ذاكرة معين فيُولّد هذا interrupt. هذه الدوال تربط الـ MSI بالـ Device Tree.

#### 5. workarounds لـ PowerPC القديمة

```c
#if defined(CONFIG_PPC32) && defined(CONFIG_PPC_PMAC)
extern unsigned int of_irq_workarounds;
extern struct device_node *of_irq_dflt_pic;
int of_irq_parse_oldworld(...);
#endif
```

أجهزة Mac القديمة (PowerMac 32-bit) كانت تستخدم Device Tree بصيغة مختلفة "oldworld" — هذا الكود يتعامل معها.

---

### كيف تسير العملية؟ — من Device Tree إلى IRQ رقم

```
Device Tree (dts file)
        |
        | of_irq_parse_one() — تقرأ "interrupts" property
        v
  of_phandle_args { np=GIC_node, args=[0,32,4] }
        |
        | irq_create_of_mapping()
        v
  irq_domain->ops->xlate() — يترجم (0,32,4) → hwirq=32, type=LEVEL_HIGH
        |
        | irq_domain_alloc_irq() — يُخصص Linux virq
        v
  Linux IRQ number (مثلاً: 45)
        |
        v
  Driver يستخدم request_irq(45, handler, ...)
```

---

### الملفات المرتبطة التي يجب معرفتها

#### الـ header الأساسي لهذا الملف
| الملف | الدور |
|-------|-------|
| `include/linux/of_irq.h` | **الملف نفسه** — API لترجمة IRQs من Device Tree |
| `drivers/of/irq.c` | التنفيذ الكامل لكل دوال `of_irq_*` |

#### الملفات المكوّنة لهذا الـ subsystem

**Core OF/Device Tree:**
| الملف | الدور |
|-------|-------|
| `include/linux/of.h` | القاموس الأساسي — struct device_node، of_phandle_args |
| `drivers/of/base.c` | قراءة properties من Device Tree |
| `drivers/of/device.c` | ربط OF nodes بـ Linux devices |
| `drivers/of/platform.c` | إنشاء platform devices من Device Tree |
| `drivers/of/address.c` | ترجمة عناوين الذاكرة من Device Tree |
| `drivers/of/property.c` | قراءة properties المتقدمة |

**IRQ Domain (يعمل مع of_irq.h مباشرة):**
| الملف | الدور |
|-------|-------|
| `include/linux/irqdomain.h` | struct irq_domain — الـ translation layer بين hwirq وvirq |
| `kernel/irq/irqdomain.c` | تنفيذ الـ irq_domain framework |
| `include/linux/irq.h` | struct irq_desc، irq_chip |

**Headers OF المتخصصة:**
| الملف | الدور |
|-------|-------|
| `include/linux/of_address.h` | ترجمة عناوين memory-mapped registers |
| `include/linux/of_platform.h` | إنشاء platform/amba devices |
| `include/linux/of_dma.h` | ربط DMA channels بالـ Device Tree |
| `include/linux/of_gpio.h` | GPIO pins من Device Tree |
| `include/linux/of_iommu.h` | IOMMU mapping من Device Tree |
| `include/linux/of_pci.h` | PCI interrupts من Device Tree |
## Phase 2: شرح الـ OF IRQ Framework

### المشكلة التي يحلها هذا الـ Subsystem

في الأنظمة المدمجة على معمارية ARM، كل SoC يحتوي على عشرات المقاطعات (interrupts) موزعة على متحكمات متعددة:

- الـ GIC (Generic Interrupt Controller) هو المتحكم الرئيسي.
- الـ GPIO controller يملك bank خاص به من الـ interrupts.
- الـ PMIC يرسل interrupts عبر I2C أو SPI.
- الـ PCIe يستخدم MSI/MSI-X.

**المشكلة الجوهرية**: كيف يعرف الـ driver أن الجهاز الخاص به يستخدم الـ IRQ رقم كذا؟ وأي متحكم interrupts يملكه؟ وما نوع الـ trigger (edge/level)?

قبل الـ Device Tree، كانت هذه المعلومات مُرمَّزة بشكل ثابت داخل الـ board file — وهو ما يعني ملفات C منفصلة لكل لوحة، وكوداً غير قابل للنقل.

الـ **OF IRQ subsystem** (`of_irq.h`) يحل هذه المشكلة بتوفير طبقة تحويل (translation layer) تقرأ معلومات الـ IRQ من الـ **Device Tree** وتحولها إلى أرقام Linux IRQ يستطيع الـ driver استخدامها مباشرة.

---

### الحل: ربط الـ Device Tree بالـ IRQ Subsystem

الـ kernel يعتمد على ثلاثة subsystems يجب فهمها معاً قبل الدخول في التفاصيل:

| Subsystem | دوره |
|---|---|
| **OF (Open Firmware / Device Tree)** | يمثل الأجهزة كشجرة نصوص، `struct device_node` هو الوحدة الأساسية |
| **IRQ Domain** (`irqdomain.h`) | يخزن الخريطة بين الـ hardware IRQ number والـ Linux virtual IRQ number |
| **OF IRQ** (`of_irq.h`) | الجسر الذي يقرأ الـ Device Tree ويُنشئ الـ IRQ Domain mappings |

الفكرة المحورية: كل `device_node` يصف متحكم interrupts يملك `struct irq_domain` خاص به. الـ OF IRQ subsystem يقرأ خاصية `interrupts` من الـ Device Tree ويترجمها عبر هذا الـ domain إلى رقم Linux يمكن تمريره لـ `request_irq()`.

---

### المثيل الواقعي: نظام الاتصالات في مستشفى

تخيل مستشفى كبيراً فيه ثلاثة أنظمة استدعاء:

- **نظام الأجراس الرئيسي** (GIC): يستقبل الإنذارات من كل الأقسام.
- **نظام غرف المرضى** (GPIO interrupt controller): كل غرفة لها زر استدعاء خاص.
- **نظام الأجهزة الطبية** (PCIe MSI): كل جهاز يرسل إشارة رقمية لنظام المراقبة.

الـ **Device Tree** هو دليل المستشفى — يقول: "غرفة 305 متصلة بزر رقم 7 في لوحة القسم الثالث، وهذه اللوحة متصلة بالمدخل 12 في النظام الرئيسي."

الـ **OF IRQ subsystem** هو موظف الاستقبال الذي يفك تشفير هذا الدليل ويقول للطاقم: "إذا أردت الاستجابة لغرفة 305، اشترك في الخط رقم 47 من نظام الإنذار المركزي."

كل جزء في المثيل يقابل مفهوماً حقيقياً:

| المثيل | المفهوم الكرنلي |
|---|---|
| دليل المستشفى | Device Tree blob (DTB) |
| غرفة المريض | `struct device_node` للجهاز |
| رقم الزر في القسم | hardware IRQ number (hwirq) |
| لوحة القسم | `struct irq_domain` للـ GPIO controller |
| النظام الرئيسي | `struct irq_domain` للـ GIC |
| الخط رقم 47 المركزي | Linux virtual IRQ number (virq) |
| موظف الاستقبال | `of_irq_parse_one()` + `irq_create_of_mapping()` |
| وصف الاتصالات في الدليل | خاصية `interrupts` في الـ DTS |

---

### المعمارية الكلية: أين يقع OF IRQ في الـ Kernel

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                        User Space                               │
 └───────────────────────────┬─────────────────────────────────────┘
                             │  syscalls
 ┌───────────────────────────▼─────────────────────────────────────┐
 │                        Kernel Space                             │
 │                                                                 │
 │  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
 │  │   Driver     │    │   Driver     │    │     Driver       │  │
 │  │ (i2c/spi/..)│    │  (gpio)      │    │   (pcie-ep)      │  │
 │  └──────┬───────┘    └──────┬───────┘    └────────┬─────────┘  │
 │         │                  │                      │            │
 │         │  irq_of_parse_and_map()                 │            │
 │         │  of_irq_get()                           │            │
 │         └──────────────────▼──────────────────────┘            │
 │                    ┌──────────────────┐                        │
 │                    │   OF IRQ Layer   │  <── of_irq.h          │
 │                    │  of_irq_parse_one│                        │
 │                    │  of_irq_init()   │                        │
 │                    │  of_msi_*()      │                        │
 │                    └────────┬─────────┘                        │
 │          ┌──────────────────┼───────────────────┐              │
 │          ▼                  ▼                   ▼              │
 │  ┌───────────────┐  ┌───────────────┐  ┌──────────────────┐   │
 │  │  irq_domain   │  │  irq_domain   │  │   irq_domain     │   │
 │  │  (GIC)        │  │  (GPIO ctrl)  │  │   (MSI parent)   │   │
 │  │  hwirq→virq   │  │  hwirq→virq   │  │   MSI vectors    │   │
 │  └───────┬───────┘  └───────┬───────┘  └────────┬─────────┘   │
 │          │                  │                   │              │
 │  ┌───────▼──────────────────▼───────────────────▼──────────┐  │
 │  │                   IRQ Core (kernel/irq/)                 │  │
 │  │           request_irq / handle_irq / irq_desc            │  │
 │  └──────────────────────────┬───────────────────────────────┘  │
 └─────────────────────────────┼───────────────────────────────────┘
                               │  Hardware IRQ lines
 ┌─────────────────────────────▼───────────────────────────────────┐
 │                        Hardware (SoC)                           │
 │  ┌────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
 │  │    GIC     │◄───│  GPIO Controller│    │   PCIe RC/EP    │  │
 │  │ (ARM GICv3)│    │  (32 IRQs)      │    │   (MSI-X)       │  │
 │  └────────────┘    └─────────────────┘    └─────────────────┘  │
 │        ▲                    ▲                      ▲            │
 │        │            ┌───────┴──────────────────────┤            │
 │  ┌─────┴──────────────────────────────────────────┐│           │
 │  │        Device Tree (فلاش/RAM)                   ││           │
 │  │  gpio0: gpio@ff780000 {                         ││           │
 │  │      interrupt-controller;                      ││           │
 │  │      #interrupt-cells = <2>;                    ││           │
 │  │      interrupts = <GIC_SPI 50 IRQ_TYPE_LEVEL>;  ││           │
 │  │  };                                             ││           │
 │  │  uart0: serial@ff190000 {                       ││           │
 │  │      interrupts = <0 12 4>;  /* GIC SPI 12 */   ││           │
 │  │  };                                             ││           │
 │  └─────────────────────────────────────────────────┘│           │
 └─────────────────────────────────────────────────────────────────┘
```

---

### التجريد المحوري: من `interrupts` في الـ DTS إلى `virq`

الـ OF IRQ subsystem يقدم **pipeline** ثلاثي المراحل:

```
DTS: interrupts = <0 12 4>
         │
         ▼ of_irq_parse_one()
  struct of_phandle_args {
      .np        = &gic_node,   /* الـ interrupt controller */
      .args      = {0, 12, 4},  /* specifier خاص بالـ GIC */
      .args_count = 3
  }
         │
         ▼ irq_create_of_mapping()
  irq_domain->ops->xlate() أو translate()
  → out_hwirq = 44  (GIC SPI 12 → hwirq 44 بعد الـ offset)
  → out_type  = IRQ_TYPE_LEVEL_HIGH
         │
         ▼ irq_create_mapping()
  virq = 67  ← هذا ما يستخدمه الـ driver مع request_irq()
```

---

### الـ Structs الأساسية وعلاقاتها

#### `struct of_phandle_args` — حزمة معلومات الـ interrupt من الـ DTS

```c
struct of_phandle_args {
    struct device_node *np;     /* مؤشر لنود الـ interrupt controller */
    int args_count;             /* عدد الـ cells في الـ specifier */
    uint32_t args[MAX_PHANDLE_ARGS]; /* القيم الخام من DTS */
};
```

هذا الـ struct هو الناتج المباشر لقراءة خاصية `interrupts` من الـ DTS. القيم فيه خام ومرتبطة بالـ hardware — لا معنى لها بدون السياق الذي يوفره الـ `irq_domain`.

#### `struct irq_domain` — قاموس الترجمة

```c
struct irq_domain {
    const struct irq_domain_ops *ops;  /* دوال الترجمة */
    struct fwnode_handle *fwnode;      /* الربط بالـ device_node */
    irq_hw_number_t hwirq_max;         /* أقصى hwirq مدعوم */
    unsigned int revmap_size;          /* حجم جدول الترجمة الخطي */
    struct irq_data *revmap[];         /* hwirq → irq_data */
    /* ... */
};
```

الـ `irq_domain` هو القلب. كل متحكم interrupts يسجّل نفسه بـ `irq_domain` يحتوي على:
- **forward map**: hwirq → virq (عبر الـ revmap).
- **ops**: دوال `xlate`/`translate` لفك تشفير الـ specifier، و`map` لإنشاء الربط.

#### `struct irq_domain_ops` — واجهة المتحكم

```c
struct irq_domain_ops {
    /* مطابقة الـ domain مع device_node من الـ DTS */
    int (*match)(struct irq_domain *d, struct device_node *node,
                 enum irq_domain_bus_token bus_token);

    /* تحويل specifier خام إلى hwirq + type */
    int (*xlate)(struct irq_domain *d, struct device_node *node,
                 const u32 *intspec, unsigned int intsize,
                 unsigned long *out_hwirq, unsigned int *out_type);

    /* النسخة الحديثة من xlate، تعمل مع irq_fwspec */
    int (*translate)(struct irq_domain *d, struct irq_fwspec *fwspec,
                     unsigned long *out_hwirq, unsigned int *out_type);

    /* إنشاء الربط بين virq و hwirq في الـ irq_desc */
    int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
};
```

#### `struct of_imap_parser` و `struct of_imap_item` — تحليل الـ interrupt-map

**الـ interrupt-map** هي خاصية DTS معقدة تُستخدم في الـ PCI bridges والـ interrupt nexus — تصف كيف تُحوَّل الـ interrupts من domain لآخر مع تعديل الـ specifier.

```c
/* يُستخدم للتكرار عبر إدخالات interrupt-map */
struct of_imap_parser {
    struct device_node *node;     /* الـ node الحالي */
    const __be32 *imap;           /* مؤشر للبيانات الخام */
    const __be32 *imap_end;       /* نهاية البيانات */
    u32 parent_offset;            /* offset الـ parent الحالي */
};

/* إدخال واحد من الـ interrupt-map */
struct of_imap_item {
    struct of_phandle_args parent_args;  /* الـ interrupt في الـ parent domain */
    u32 child_imap_count;                /* حجم الـ child specifier */
    u32 child_imap[16];                  /* قيم الـ child specifier */
};
```

مثال DTS على الـ interrupt-map في PCIe bridge:

```dts
pcie0: pcie@f8000000 {
    interrupt-map-mask = <0 0 0 7>;
    interrupt-map =
        /* child unit addr, child IRQ, parent, parent IRQ specifier */
        <0 0 0 1 &gic GIC_SPI 91 IRQ_TYPE_LEVEL_HIGH>,
        <0 0 0 2 &gic GIC_SPI 92 IRQ_TYPE_LEVEL_HIGH>,
        <0 0 0 3 &gic GIC_SPI 93 IRQ_TYPE_LEVEL_HIGH>,
        <0 0 0 4 &gic GIC_SPI 94 IRQ_TYPE_LEVEL_HIGH>;
};
```

**الـ** `for_each_of_imap_item` **macro** يُسهّل التكرار عبر هذه الإدخالات:

```c
#define for_each_of_imap_item(parser, item) \
    for (; of_imap_parser_one(parser, item);)
```

تحذير مهم في الكود: إذا خرجت من الـ loop مبكراً (`break`/`goto`/`return`)، يجب استدعاء `of_node_put(item->parent_args.np)` يدوياً لتجنب memory leak.

---

### رسم العلاقات بين الـ Structs

```
DTS Node (uart0)
   │  property: interrupts = <0 12 4>
   │  property: interrupt-parent = <&gic>
   │
   ▼
struct device_node (uart0)
   │  .fwnode → fwnode_handle
   │  .properties → struct property list
   │
   │  of_irq_parse_one(uart0_node, 0, &args)
   │
   ▼
struct of_phandle_args
   │  .np = gic_node          ← interrupt controller
   │  .args = {0, 12, 4}      ← raw specifier
   │  .args_count = 3
   │
   │  irq_create_of_mapping(&args)
   │    → irq_find_host(gic_node)
   │
   ▼
struct irq_domain (GIC domain)
   │  .ops->xlate({0,12,4}) → hwirq=44, type=IRQ_TYPE_LEVEL_HIGH
   │  .ops->map(virq, 44)   → إعداد irq_desc
   │  .revmap[44] = &irq_data
   │
   ▼
virq = 67   ← Linux IRQ number
   │
   ▼
request_irq(67, handler, 0, "uart", dev)
```

---

### الـ MSI Integration

الـ OF IRQ subsystem يدعم أيضاً الـ **MSI (Message Signaled Interrupts)** عبر ثلاث دوال رئيسية:

```c
/* البحث عن MSI domain مرتبط بـ device_node معين */
struct irq_domain *of_msi_get_domain(
    struct device *dev,
    const struct device_node *np,
    enum irq_domain_bus_token token);

/* ربط الـ device بالـ MSI domain الصحيح بناءً على الـ id */
struct irq_domain *of_msi_map_get_device_domain(
    struct device *dev,
    u32 id,
    u32 bus_token);

/* إعداد الـ MSI configuration للـ device من الـ DTS */
void of_msi_configure(struct device *dev, const struct device_node *np);

/* ترجمة MSI id من الـ device إلى الـ parent domain */
u32 of_msi_xlate(struct device *dev, struct device_node **msi_np, u32 id_in);
```

الـ `irq_domain_bus_token` يميز بين أنواع مختلفة من الـ domains:

```c
enum irq_domain_bus_token {
    DOMAIN_BUS_ANY           = 0,  /* بحث عام */
    DOMAIN_BUS_WIRED,              /* interrupts تقليدية */
    DOMAIN_BUS_GENERIC_MSI,        /* MSI عام */
    DOMAIN_BUS_PCI_MSI,            /* MSI لـ PCIe */
    DOMAIN_BUS_PLATFORM_MSI,       /* MSI للـ platform devices */
    /* ... */
};
```

---

### دورة حياة الـ OF IRQ: من Boot إلى `request_irq`

#### المرحلة 1: تهيئة الـ Interrupt Controllers

```c
void of_irq_init(const struct of_device_id *matches);
```

يُستدعى مرة واحدة أثناء الـ boot. يبحث في الـ Device Tree عن كل الـ nodes التي تملك خاصية `interrupt-controller` ويطابقها مع الـ `matches` table. يُنشئ الـ `irq_domain` لكل متحكم بترتيب هرمي (الـ root controller أولاً).

```c
/* مثال: تسجيل GIC driver */
IRQCHIP_DECLARE(gic_v3, "arm,gic-v3", gic_of_init);
/* هذا يضيف entry في __irqchip_of_table section */
/* of_irq_init() يقرأ هذا الـ section ويستدعي gic_of_init */
```

#### المرحلة 2: تحليل الـ IRQ لجهاز معين

```c
int of_irq_parse_one(struct device_node *device, int index,
                     struct of_phandle_args *out_irq);
```

يقرأ خاصية `interrupts` من الـ `device_node` ويملأ `of_phandle_args`. يأخذ بعين الاعتبار:
- الـ `interrupt-parent` (الـ controller المعني).
- الـ `#interrupt-cells` (عدد القيم في الـ specifier).
- الـ `interrupt-map` إذا وُجد (للـ bridges).

```c
/* مثال حقيقي من driver */
struct of_phandle_args irq_data;
ret = of_irq_parse_one(dev->of_node, 0, &irq_data);
/* irq_data.np → GIC node */
/* irq_data.args → {0, 12, IRQ_TYPE_LEVEL_HIGH} */
```

#### المرحلة 3: إنشاء الـ Mapping

```c
unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data);
```

يجد الـ `irq_domain` المرتبط بالـ `irq_data->np`، ثم يستدعي `ops->xlate()` لتحويل الـ specifier إلى `hwirq + type`، ثم يُنشئ mapping في الـ domain ويعيد رقم الـ `virq`.

#### الـ All-in-one function

```c
unsigned int irq_of_parse_and_map(struct device_node *node, int index);
```

تجمع المرحلتين 2 و 3 في استدعاء واحد. الدالة الأكثر استخداماً في الـ drivers:

```c
/* في probe() الـ driver */
int irq = irq_of_parse_and_map(pdev->dev.of_node, 0);
if (!irq) {
    dev_err(&pdev->dev, "failed to get IRQ\n");
    return -ENODEV;
}
ret = request_irq(irq, my_handler, 0, "my-device", priv);
```

---

### ما يملكه الـ OF IRQ وما يفوّضه للـ Drivers

| المهمة | يتكفل بها OF IRQ | يتكفل بها الـ Driver / Controller |
|---|---|---|
| قراءة `interrupts` من DTS | نعم (`of_irq_parse_one`) | لا |
| إيجاد الـ `irq_domain` الصحيح | نعم (`irq_find_host`) | لا |
| تحليل الـ `interrupt-map` | نعم (`of_imap_parser_*`) | لا |
| تحويل specifier → hwirq | لا | نعم (`ops->xlate/translate`) |
| إنشاء virq وربطه بـ hwirq | نعم (`irq_create_mapping`) | لا |
| إعداد الـ `irq_desc` وتحديد الـ handler | لا | نعم (`ops->map`) |
| برمجة الـ hardware registers | لا | نعم (`ops->activate`) |
| تسجيل handler من الـ driver | لا | Driver (`request_irq`) |

---

### نقطة دقيقة: الـ Workarounds لـ Old World PowerMac

```c
#if defined(CONFIG_PPC32) && defined(CONFIG_PPC_PMAC)
extern unsigned int of_irq_workarounds;
extern struct device_node *of_irq_dflt_pic;
int of_irq_parse_oldworld(...);
#else
#define of_irq_workarounds (0)
#define of_irq_dflt_pic (NULL)
/* stub returning -EINVAL */
#endif
```

الـ **Old World PowerMac** (iMac G3 الأولى) استخدمت Open Firmware بدون `phandle` صحيح في بعض الحالات. الفلاغات:
- `OF_IMAP_OLDWORLD_MAC`: يعني الـ `interrupt-map` بصيغة قديمة.
- `OF_IMAP_NO_PHANDLE`: الـ entries لا تحتوي على phandle صريح.

هذا الكود تاريخي بالكامل ولا أهمية له في الأنظمة الحديثة.

---

### مثال عملي: ARM SoC مع GIC و GPIO interrupt controller

```dts
/* Device Tree Source */
/ {
    gic: interrupt-controller@2c001000 {
        compatible = "arm,cortex-a9-gic";
        #interrupt-cells = <3>;       /* domain, irq, flags */
        interrupt-controller;
        reg = <0x2c001000 0x1000>;
    };

    gpio0: gpio@e0300000 {
        compatible = "myvendor,gpio";
        #interrupt-cells = <2>;       /* gpio-num, flags */
        interrupt-controller;
        /* GPIO controller نفسه متصل بالـ GIC */
        interrupts = <GIC_SPI 50 IRQ_TYPE_LEVEL_HIGH>;
        interrupt-parent = <&gic>;
    };

    uart0: serial@e0000000 {
        compatible = "myvendor,uart";
        interrupt-parent = <&gic>;
        interrupts = <GIC_SPI 12 IRQ_TYPE_LEVEL_HIGH>;
        /* GIC SPI 12 = hwirq 44 في بعض الـ GIC implementations */
    };

    sensor@0 {
        compatible = "myvendor,temp-sensor";
        interrupt-parent = <&gpio0>;
        interrupts = <5 IRQ_TYPE_EDGE_RISING>;
        /* GPIO line 5 في الـ GPIO controller */
    };
};
```

```c
/* في gpio driver probe: إنشاء irq_domain للـ GPIO controller */
static int gpio_probe(struct platform_device *pdev)
{
    struct gpio_priv *priv = /* ... */;

    /* إنشاء domain خطي لـ 32 GPIO interrupt */
    priv->irq_domain = irq_domain_add_linear(
        pdev->dev.of_node,
        32,                    /* عدد الـ interrupts */
        &gpio_irq_domain_ops,  /* تحتوي على xlate و map */
        priv
    );

    /* الـ GIC interrupt للـ GPIO controller نفسه */
    int gpio_irq = irq_of_parse_and_map(pdev->dev.of_node, 0);
    request_irq(gpio_irq, gpio_irq_handler, IRQF_SHARED, "gpio", priv);

    return 0;
}

/* في sensor driver probe */
static int sensor_probe(struct platform_device *pdev)
{
    /* هذا يمر عبر GPIO domain ثم يصل للـ GIC */
    int irq = irq_of_parse_and_map(pdev->dev.of_node, 0);
    /* الترتيب الداخلي:
     * 1. of_irq_parse_one → {gpio0_node, {5, IRQ_TYPE_EDGE_RISING}}
     * 2. irq_find_host(gpio0_node) → gpio_irq_domain
     * 3. gpio_irq_domain->ops->xlate({5, EDGE}) → hwirq=5, type=EDGE
     * 4. irq_create_mapping(gpio_domain, 5) → virq=83
     */
    request_irq(irq, sensor_handler, 0, "temp-sensor", priv);
}
```

---

### الـ `of_irq_get` vs `irq_of_parse_and_map`

```c
/* إرجاع virq أو error code سلبي — أفضل للتعامل مع الأخطاء */
int of_irq_get(struct device_node *dev, int index);

/* إرجاع virq أو 0 في حالة الخطأ — واجهة أقدم */
unsigned int irq_of_parse_and_map(struct device_node *node, int index);

/* البحث بالاسم بدل الـ index */
int of_irq_get_byname(struct device_node *dev, const char *name);
```

`of_irq_get_byname` تُستخدم عندما يملك الـ device أكثر من interrupt:

```dts
uart0: serial@e0000000 {
    interrupt-names = "rx", "tx", "error";
    interrupts = <GIC_SPI 12 4>, <GIC_SPI 13 4>, <GIC_SPI 14 4>;
};
```

```c
int rx_irq    = of_irq_get_byname(node, "rx");
int tx_irq    = of_irq_get_byname(node, "tx");
int error_irq = of_irq_get_byname(node, "error");
```

---

### خلاصة: ما يقدمه الـ OF IRQ Framework

الـ `of_irq.h` يعرّف الـ **glue layer** الذي يربط تمثيل الـ hardware في الـ Device Tree بنظام الـ IRQ في الـ kernel. بدونه، كل driver يحتاج لمعرفة بنية كل متحكم interrupts على حدة والترميز الصلب لأرقام الـ IRQ — وهو ما يجعل الكود غير قابل للنقل.

الـ abstraction المحوري هو **إخفاء تسلسل متحكمات الـ interrupts** وراء واجهة موحدة: Driver يطلب `irq_of_parse_and_map()` ويحصل على رقم يمكنه استخدامه مع `request_irq()` — بغض النظر عن عدد طبقات الـ interrupt controllers بين جهازه والـ CPU.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options

#### flags الـ OF_IMAP (workarounds لـ PowerMac 32-bit فقط)

| Flag | القيمة | الوصف |
|------|--------|-------|
| `OF_IMAP_OLDWORLD_MAC` | `0x00000001` | جهاز PowerMac قديم (Old World) يحتاج معالجة خاصة لـ interrupt-map |
| `OF_IMAP_NO_PHANDLE` | `0x00000002` | الـ interrupt-map لا تحتوي phandle — يُستخدم `of_irq_dflt_pic` بدلاً منه |

#### flags الـ irq_domain (من `irqdomain.h`)

| Flag | القيمة | الوصف |
|------|--------|-------|
| `IRQ_DOMAIN_FLAG_HIERARCHY` | bit 0 | الـ domain هرمي (parent/child) |
| `IRQ_DOMAIN_FLAG_IPI_PER_CPU` | bit 2 | IPI بـ virq منفصل لكل CPU |
| `IRQ_DOMAIN_FLAG_IPI_SINGLE` | bit 3 | IPI بـ virq واحد مشترك |
| `IRQ_DOMAIN_FLAG_MSI` | bit 4 | الـ domain يدعم MSI |
| `IRQ_DOMAIN_FLAG_ISOLATED_MSI` | bit 5 | MSI معزول (isolated) |
| `IRQ_DOMAIN_FLAG_NO_MAP` | bit 6 | لا ترجمة — virq = hwirq مباشرة |
| `IRQ_DOMAIN_FLAG_MSI_PARENT` | bit 8 | هذا الـ domain أب لـ MSI device domain |
| `IRQ_DOMAIN_FLAG_MSI_DEVICE` | bit 9 | domain خاص بجهاز MSI واحد |
| `IRQ_DOMAIN_FLAG_DESTROY_GC` | bit 10 | تحرير الـ generic chips عند الإزالة |
| `IRQ_DOMAIN_FLAG_MSI_IMMUTABLE` | bit 11 | address/data لا يتغيران عند `irq_set_affinity` |
| `IRQ_DOMAIN_FLAG_FWNODE_PARENT` | bit 12 | يتطلب مطابقة الـ parent fwnode |
| `IRQ_DOMAIN_FLAG_NONCORE` | bit 16+ | محجوز للاستخدام الداخلي |

#### flags الـ device_node (من `of.h`)

| Flag (bit index) | الوصف |
|-----------------|-------|
| `OF_DYNAMIC` (1) | الـ node مُخصص ديناميكياً (kmalloc) |
| `OF_DETACHED` (2) | منفصل عن الشجرة |
| `OF_POPULATED` (3) | تم إنشاء device له |
| `OF_POPULATED_BUS` (4) | تم إنشاء platform bus للأبناء |
| `OF_OVERLAY` (5) | مُخصص لـ overlay |
| `OF_OVERLAY_FREE_CSET` (6) | في cset جاري تحريره |

#### config options

| Config | الأثر |
|--------|-------|
| `CONFIG_OF_IRQ` | يُفعّل كل API الرئيسي — بدونه كل الدوال stubs ترجع صفراً أو -EINVAL |
| `CONFIG_PPC32 && CONFIG_PPC_PMAC` | يُفعّل `of_irq_parse_oldworld` والـ workarounds |
| `CONFIG_IRQ_DOMAIN_HIERARCHY` | يُضيف alloc/free/activate/translate/deactivate للـ domain ops |
| `CONFIG_GENERIC_MSI_IRQ` | يُفعّل MSI parent ops في الـ irq_domain |
| `CONFIG_SPARC` | يُفعّل `irq_of_parse_and_map` بتنفيذ SPARC الخاص |

---

### الـ Structs المهمة

#### 1. `struct of_imap_parser`

**الغرض:** حالة iterator لقراءة خاصية `interrupt-map` من device tree بشكل تدريجي دون allocations ديناميكية.

```c
struct of_imap_parser {
    struct device_node *node;    /* الـ node الحالي في الـ interrupt-map */
    const __be32 *imap;          /* المؤشر الحالي في بيانات الـ interrupt-map */
    const __be32 *imap_end;      /* نهاية البيانات (حدّ الـ loop) */
    u32 parent_offset;           /* offset داخل كل entry يشير إلى الـ parent */
};
```

**الاستخدام:** يُمرّر بالمؤشر إلى `of_imap_parser_init()` ثم يُستخدم في `for_each_of_imap_item()`.

---

#### 2. `struct of_imap_item`

**الغرض:** يحمل نتيجة entry واحد من الـ `interrupt-map` بعد parse — يصف الـ child specifier والـ parent interrupt specifier.

```c
struct of_imap_item {
    struct of_phandle_args parent_args; /* phandle الـ parent + args الـ interrupt */
    u32 child_imap_count;               /* عدد خلايا الـ child في هذا الـ entry */
    u32 child_imap[16];                 /* بيانات الـ child (#address-cells + #interrupt-cells) */
};
```

**ملاحظة:** الحجم 16 تقديري ثابت — يكفي في الغالب دون استخدام heap.

**العلاقة مع structs أخرى:** يحتوي `parent_args` من نوع `struct of_phandle_args` الذي يُشير بدوره إلى `struct device_node` عبر `parent_args.np`.

---

#### 3. `struct of_phandle_args` (من `of.h`)

**الغرض:** يصف مرجعاً إلى node عبر phandle مصحوباً بقائمة arguments — الناتج المباشر لـ parse عملية interrupt specifier.

```c
struct of_phandle_args {
    struct device_node *np;           /* الـ node المُشار إليه (الـ interrupt controller) */
    int args_count;                   /* عدد الـ arguments */
    uint32_t args[MAX_PHANDLE_ARGS];  /* قيم الـ arguments (#interrupt-cells خلايا) */
};
```

**يُستخدم في:** كل دوال `of_irq_parse_*` لتسليم نتيجة الـ parse، ثم يُمرّر إلى `irq_create_of_mapping()`.

---

#### 4. `struct device_node` (من `of.h`)

**الغرض:** تمثيل node واحد من شجرة الـ device tree في الذاكرة.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ node |
| `phandle` | `phandle` (u32) | معرّف فريد يُستخدم للـ cross-reference |
| `full_name` | `const char *` | المسار الكامل `/soc/interrupt-controller@...` |
| `fwnode` | `struct fwnode_handle` | واجهة موحدة مع subsystem الـ firmware node |
| `properties` | `struct property *` | قائمة مرتبطة بخصائص الـ node |
| `parent` | `struct device_node *` | الأب في الشجرة |
| `child` | `struct device_node *` | أول ابن |
| `sibling` | `struct device_node *` | الأخ التالي |
| `_flags` | `unsigned long` | bits: DYNAMIC, DETACHED, POPULATED... |

---

#### 5. `struct irq_domain` (من `irqdomain.h`)

**الغرض:** يمثّل **مجال ترجمة الـ interrupts** — يُحوّل hwirq (رقم الـ hardware interrupt) إلى virq (Linux IRQ number) والعكس.

| الحقل | الوصف |
|-------|-------|
| `link` | ربط في القائمة العالمية للـ domains |
| `name` | اسم الـ domain (debug) |
| `ops` | جدول العمليات: map/unmap/xlate/alloc... |
| `host_data` | بيانات خاصة بالـ driver |
| `flags` | `IRQ_DOMAIN_FLAG_*` |
| `mapcount` | عدد الـ mappings النشطة |
| `mutex` | قفل الـ domain (hierarchical يستخدم قفل الـ root) |
| `root` | مؤشر للـ root domain |
| `fwnode` | يربطه بـ device_node عبر `irq_domain_get_of_node()` |
| `bus_token` | نوع الـ bus (WIRED, PCI, MSI...) |
| `parent` | الـ domain الأب (في الهرمي) |
| `hwirq_max` | أقصى قيمة hwirq مدعومة |
| `revmap_size` | حجم الـ linear map |
| `revmap_tree` | radix tree للـ hwirqs خارج الـ linear map |
| `revmap[]` | مصفوفة الـ linear map (flexible array) |

---

#### 6. `struct irq_domain_ops` (من `irqdomain.h`)

**الغرض:** جدول virtual dispatch — كل interrupt controller يُوفّر implementation خاصة.

| الدالة | المهمة |
|--------|--------|
| `match` | هل هذا الـ domain يتطابق مع هذا الـ device_node؟ |
| `select` | مثل match لكن يستقبل `irq_fwspec` (أعمم) |
| `map` | إنشاء mapping بين virq وهwirq — يُستدعى مرة واحدة |
| `unmap` | إزالة الـ mapping |
| `xlate` | ترجمة DT specifier → hwirq + type |
| `translate` | مثل xlate لكن يستقبل `irq_fwspec` |
| `alloc` | (هرمي) تخصيص nr_irqs من virq |
| `free` | (هرمي) تحرير interrupts |
| `activate` | تفعيل interrupt في الـ hardware |
| `deactivate` | تعطيل interrupt في الـ hardware |
| `get_fwspec_info` | استخراج معلومات إضافية (affinity) |
| `debug_show` | عرض بيانات في debugfs |

---

#### 7. `of_irq_init_cb_t`

```c
typedef int (*of_irq_init_cb_t)(struct device_node *, struct device_node *);
```

**الغرض:** نوع callback يُستخدم داخلياً في `of_irq_init()` — الأول هو الـ node الخاص بالـ controller، والثاني هو الـ parent controller.

---

### مخططات علاقات الـ Structs

```
                    ┌─────────────────────────────┐
                    │      device_node             │
                    │  name, phandle, full_name    │
                    │  fwnode ──────────────────────┼──→ fwnode_handle
                    │  properties → property chain │
                    │  parent ──┐                  │
                    │  child  ──┤  شجرة DT         │
                    │  sibling──┘                  │
                    └──────────┬──────────────────-┘
                               │ np
                               ▼
                    ┌─────────────────────────────┐
                    │     of_phandle_args          │
                    │  np → device_node            │◀── ناتج of_irq_parse_one()
                    │  args_count                  │◀── ناتج of_irq_parse_raw()
                    │  args[MAX_PHANDLE_ARGS]      │
                    └──────────┬───────────────────┘
                               │ parent_args
                               ▼
                    ┌─────────────────────────────┐
                    │      of_imap_item            │
                    │  parent_args (of_phandle)    │
                    │  child_imap_count            │
                    │  child_imap[16]              │
                    └──────────▲───────────────────┘
                               │ يملأه
                    ┌──────────┴───────────────────┐
                    │     of_imap_parser           │
                    │  node → device_node          │
                    │  imap  (raw __be32 ptr)      │
                    │  imap_end                    │
                    │  parent_offset               │
                    └──────────────────────────────┘

                    ┌─────────────────────────────┐
                    │       irq_domain             │
                    │  fwnode ──→ device_node      │ (عبر to_of_node)
                    │  ops ─────→ irq_domain_ops  │
                    │  parent ──→ irq_domain       │ (هرمي)
                    │  root ────→ irq_domain       │
                    │  revmap[] → irq_data*        │
                    └──────────────────────────────┘
                               ▲
                    يُنشأ من   │  irq_create_of_mapping(of_phandle_args*)
                               │
                    ┌──────────┴───────────────────┐
                    │     of_phandle_args          │
                    │  (يحمل hwirq + type)         │
                    └──────────────────────────────┘
```

---

### مخطط دورة الحياة — تهيئة الـ Interrupt Controllers

```
boot (early_initcall)
        │
        ▼
of_irq_init(matches[])
        │
        ├─── for_each_matching_node(matches)
        │         │
        │         ▼
        │    of_irq_find_parent(child_node)
        │         │ يعود بـ device_node للـ parent controller
        │         ▼
        │    يُرتّب قائمة بالترتيب (parent قبل child)
        │
        ├─── لكل controller في الترتيب:
        │         │
        │         ▼
        │    cb(node, parent_node)    ← of_irq_init_cb_t
        │         │  (الـ driver يُسجّل irq_domain هنا)
        │         ▼
        │    irq_domain_instantiate()
        │         │
        │         ▼
        │    يُسجَّل الـ domain في القائمة العالمية
        │
        ▼
النظام جاهز لـ mapping الـ interrupts
```

---

### مخطط دورة الحياة — Mapping interrupt من DT إلى Linux IRQ

```
device driver probe
        │
        ▼
of_irq_get(dev_node, index)
        │
        ▼
of_irq_parse_one(dev_node, index, &irq_args)
        │  يقرأ خاصية "interrupts" أو "interrupt-map"
        │
        ├─── يبحث عن "interrupt-parent" (phandle)
        │    أو يصعد عبر of_irq_find_parent()
        │
        ├─── of_irq_parse_raw(addr, &irq_args)
        │    يُفسّر raw bytes من DT
        │
        ▼
irq_create_of_mapping(&irq_args)
        │
        ├─── of_phandle_args_to_fwspec()     ← يحوّل args → irq_fwspec
        │
        ├─── irq_find_matching_fwspec()      ← يجد الـ irq_domain المناسب
        │
        ├─── domain->ops->xlate() أو translate()
        │    يُفسّر specifier → hwirq + type
        │
        ├─── irq_create_mapping()
        │    يخصص virq جديد
        │    يُسجّله في revmap[]
        │
        ▼
يعود بـ Linux virq number
        │
        ▼
driver يستخدم virq مع request_irq()
```

---

### مخطط تدفق استدعاء `irq_of_parse_and_map()`

```
driver: irq_of_parse_and_map(node, index)
  │
  ▼
of_irq_parse_one(node, index, &out_irq)
  │
  ├─→ of_get_property(node, "interrupts", ...)
  │     │ يقرأ الـ interrupt specifier الخام
  │     ▼
  ├─→ of_irq_find_parent(node)
  │     │ يصعد في الشجرة حتى يجد "interrupt-parent"
  │     ▼  أو "interrupt-controller"
  │
  ├─→ [إذا وُجدت "interrupt-map"]:
  │     of_imap_parser_init(parser, parent_node, item)
  │       │
  │       for_each_of_imap_item(parser, item):
  │         of_imap_parser_one(parser, item)
  │           │  تُطابق child specifier وتعيد parent_args
  │           ▼
  │         تُحدّث out_irq بالـ parent_args
  │
  ▼
out_irq.np  = pointer لـ interrupt controller node
out_irq.args = [hwirq, flags, ...]
  │
  ▼
irq_create_of_mapping(&out_irq)
  │
  ├─→ irq_find_host(out_irq.np)
  │     └─→ irq_find_matching_host(node, DOMAIN_BUS_WIRED)
  │           └─→ irq_find_matching_fwnode(fwnode, token)
  │
  ├─→ domain->ops->xlate(domain, node, intspec, intsize,
  │                       &hwirq, &type)
  │     مثال: irq_domain_xlate_twocell()
  │       args[0] → hwirq
  │       args[1] → type (IRQ_TYPE_EDGE_RISING...)
  │
  ├─→ irq_create_mapping(domain, hwirq)
  │     └─→ irq_create_mapping_affinity(domain, hwirq, NULL)
  │           ├─→ domain->ops->map(domain, virq, hwirq)
  │           │     driver يُهيئ irq_desc هنا
  │           └─→ يُسجّل في domain->revmap[hwirq]
  │
  ▼
returns: virq (Linux IRQ number) — يُمرَّر لـ request_irq()
```

---

### مخطط تدفق استدعاء MSI

```
pci_device probe
  │
  ▼
of_msi_configure(dev, np)
  │  تقرأ "msi-parent" phandle من DT
  │  تُعيّن dev->msi.domain
  │
  ▼
of_msi_get_domain(dev, np, token)
  │
  ├─→ of_parse_phandle(np, "msi-parent", 0)
  │     └─→ يعود بـ device_node للـ MSI controller
  │
  ├─→ irq_find_matching_host(msi_node, token)
  │     └─→ يجد irq_domain بالـ bus_token المطلوب
  │
  ▼
returns: irq_domain* للـ MSI
  │
  ▼
of_msi_map_get_device_domain(dev, id, bus_token)
  │  يجد الـ domain المناسب بعد ترجمة الـ id
  │
  ▼
of_msi_xlate(dev, &msi_np, id_in)
  │  يترجم id الجهاز إلى id في الـ MSI domain
  │  (عبر of_map_id من of.h)
  ▼
returns: translated id
```

---

### مخطط دورة حياة `of_imap_parser` (interrupt-map iteration)

```
of_imap_parser_init(parser, parent_node, item)
  │
  │  يقرأ "interrupt-map" property من parent_node
  │  يضبط parser->imap, imap_end, parent_offset
  │
  ▼
for_each_of_imap_item(parser, item):
  │
  ├─→ of_imap_parser_one(parser, item)
  │     │
  │     ├─ تقرأ child_imap[0..child_imap_count-1] من imap
  │     ├─ تقرأ phandle للـ parent controller
  │     ├─ تحل الـ phandle → device_node (of_find_node_by_phandle)
  │     │   [تزيد refcount — يجب of_node_put() عند break]
  │     ├─ تقرأ parent interrupt specifier
  │     ├─ تملأ item->parent_args.np, args
  │     └─ تُقدّم parser->imap للـ entry التالي
  │
  └─→ تُعيد NULL عند انتهاء الـ map
  │
  ▼
[break/return مبكراً]:
  يجب استدعاء of_node_put(item->parent_args.np)
  لتجنب memory leak
```

---

### استراتيجية الـ Locking

#### قفل الـ `irq_domain->mutex`

- **ما يحميه:** عمليات الـ mapping (إنشاء/حذف الـ revmap entries)، تعديل `mapcount`، الوصول إلى `revmap[]` وـ`revmap_tree`
- **الإمساك به في:** `irq_create_mapping()`, `irq_dispose_mapping()`, `irq_domain_associate()`
- **في الهرمي:** جميع الـ domains تستخدم قفل الـ root domain (`domain->root->mutex`) — يمنع deadlock

```
irq_domain (child)
    root → irq_domain (root)
                mutex ← القفل الوحيد المستخدم للكل
```

#### قفل الـ `of_node` (RCU + kref)

- **ما يحميه:** دورة حياة الـ `device_node`
- **الآلية:** `of_node_get()` / `of_node_put()` — reference counting
- **في `of_imap_parser`:** كل `of_imap_parser_one()` تزيد refcount للـ `item->parent_args.np` — المُستخدِم مسؤول عن `of_node_put()` عند الخروج المبكر من الـ loop

#### قفل `irq_desc` العالمي

- **لا يظهر مباشرةً في `of_irq.h`**، لكن `irq_create_mapping()` تُقفل الـ `irq_desc` عبر `alloc_desc` داخلياً
- ترتيب الأقفال المقبول: `irq_domain->mutex` ثم `irq_desc lock` — العكس غير مسموح

#### ملخص ترتيب الأقفال

```
1. of_root lock (كتابة DT — نادر)
      │
      ▼
2. irq_domain->root->mutex
      │
      ▼
3. irq_desc->lock (spinlock — سياق IRQ)
```

**قاعدة:** لا يُمسك بـ `irq_domain->mutex` وأنت تحت spinlock (سياق interrupt) — الـ mutex يُمسك فقط في سياق process.
## Phase 4: شرح الـ Functions

---

### ملخص شامل — Cheatsheet

#### الـ Types والـ Structs الأساسية

| النوع / البنية | الغرض |
|---|---|
| `of_irq_init_cb_t` | Function pointer لـ callback تهيئة interrupt controller من DT |
| `struct of_imap_parser` | Iterator state لقراءة `interrupt-map` property خطوة بخطوة |
| `struct of_imap_item` | يحمل نتيجة entry واحدة من `interrupt-map` بما فيها `parent_args` و `child_imap` |
| `struct of_phandle_args` | يحمل pointer لـ `device_node` + مصفوفة args الخاصة بالـ interrupt specifier |
| `struct irq_domain` | كيان ترجمة IRQ: من HW irq number إلى Linux virq |

#### جدول الـ Functions السريع

| الدالة | المجموعة | الغرض المختصر |
|---|---|---|
| `of_irq_find_parent()` | Topology | إيجاد interrupt-parent node في DT |
| `of_irq_parse_raw()` | Parsing | تحليل raw `interrupt-map` بعد توفير العنوان |
| `of_irq_parse_one()` | Parsing | تحليل interrupt رقم N من node معين |
| `of_irq_parse_oldworld()` | Parsing (PPC legacy) | تحليل interrupts على PowerMac القديمة |
| `irq_of_parse_and_map()` | High-level | parse + map في خطوة واحدة، يُعيد virq |
| `irq_create_of_mapping()` | Mapping | تحويل `of_phandle_args` إلى Linux virq |
| `of_irq_get()` | High-level | يُعيد virq جاهز للاستخدام من index |
| `of_irq_get_byname()` | High-level | يُعيد virq بالاسم من `interrupt-names` |
| `of_irq_get_affinity()` | High-level | يُعيد `cpumask` affinity للـ interrupt |
| `of_irq_count()` | Query | عدد interrupts مُعرّفة في node |
| `of_irq_to_resource()` | Resource | ملء `struct resource` بمعلومات IRQ |
| `of_irq_to_resource_table()` | Resource | ملء مصفوفة resources بكل IRQs الـ node |
| `of_irq_init()` | Init | تهيئة interrupt controllers بالترتيب الصحيح من DT |
| `of_imap_parser_init()` | imap Iterator | تهيئة iterator لـ `interrupt-map` |
| `of_imap_parser_one()` | imap Iterator | قراءة entry واحدة من `interrupt-map` |
| `for_each_of_imap_item()` | imap Iterator | Macro للتكرار على كل entries |
| `of_msi_get_domain()` | MSI | إيجاد `irq_domain` خاص بـ MSI من DT |
| `of_msi_map_get_device_domain()` | MSI | إيجاد MSI domain بعد ترجمة الـ ID |
| `of_msi_configure()` | MSI | ضبط MSI domain على الـ device |
| `of_msi_xlate()` | MSI | ترجمة device ID إلى MSI ID صالح |

---

### المجموعة الأولى: Topology — إيجاد الـ Interrupt Parent

هذه المجموعة تُجيب على السؤال: "من هو interrupt controller المسؤول عن هذا الـ device؟" الـ DT يُعبّر عن هذه العلاقة عبر `interrupt-parent` phandle أو عبر hierarchy الـ nodes.

---

#### `of_irq_find_parent()`

```c
struct device_node *of_irq_find_parent(struct device_node *child);
```

تبدأ من `child` وتصعد في شجرة الـ DT بحثاً عن أول node يحمل `interrupt-parent` property أو يكون هو نفسه interrupt controller (يحمل `#interrupt-cells`). تتبع سلسلة الـ phandles حتى تجد controller حقيقي ليس مجرد passthrough.

**المعاملات:**
- `child`: الـ `device_node` الذي نريد معرفة interrupt parent الخاص به.

**القيمة المُعادة:** مؤشر لـ `device_node` الخاص بالـ interrupt parent مع زيادة refcount. يجب استدعاء `of_node_put()` على النتيجة عند الانتهاء. يُعيد `NULL` إذا لم يُوجد.

**تفاصيل مهمة:**
- يتحقق من الـ `#interrupt-cells` property للتأكد أن الـ node هو controller فعلي وليس مجرد bus node.
- يُستخدم داخلياً من `of_irq_parse_one()` وعدد من دوال الـ parsing الأخرى.
- **لا يأخذ lock** بل يعتمد على RCU read lock الداخلي لـ OF subsystem.

**السياق:** يُستدعى من kernel init أو driver probe context.

---

### المجموعة الثانية: Parsing — تحليل الـ Interrupt Specifiers

هذه الدوال تحوّل البيانات الخام من الـ DT (مثل `interrupts`, `interrupt-map`) إلى بنية `of_phandle_args` منظمة تصف الـ interrupt بشكل كامل: من هو الـ controller وما هي المعاملات (رقم الـ IRQ، النوع، إلخ).

---

#### `of_irq_parse_one()`

```c
int of_irq_parse_one(struct device_node *device, int index,
                     struct of_phandle_args *out_irq);
```

تقرأ الـ `interrupts` أو `interrupts-extended` property من الـ `device` node، وتستخرج الـ entry رقم `index`، ثم تتبع `interrupt-map` إذا وجدت للوصول للـ controller الحقيقي. هذه هي دالة الـ parsing الرئيسية التي يستخدمها معظم الكود.

**المعاملات:**
- `device`: الـ node الذي يحمل `interrupts` property.
- `index`: رقم الـ interrupt المطلوب (0-based).
- `out_irq`: بنية الإخراج — تُملأ بـ `np` (pointer للـ interrupt controller node) وبـ `args[]` (specifier cells).

**القيمة المُعادة:** 0 عند النجاح، قيمة سالبة عند الفشل (`-EINVAL` إذا لم توجد الـ property، `-ENOENT` إذا كان الـ index خارج النطاق).

**تفاصيل مهمة:**
- تدعم كلاً من `interrupts` البسيطة و`interrupts-extended` (phandle مدمج مع specifier).
- تستدعي `of_irq_parse_raw()` داخلياً لمعالجة الـ `interrupt-map` إذا وجدت.
- يجب استدعاء `of_node_put(out_irq->np)` عند الانتهاء.

**Pseudocode:**
```
of_irq_parse_one(device, index, out_irq):
    if "interrupts-extended" exists:
        parse phandle + args at [index] directly → out_irq
        return 0

    addr = get #address-cells of parent
    intsize = get #interrupt-cells of interrupt-parent

    read interrupts[index * intsize ... ]
    find interrupt-parent node

    call of_irq_parse_raw(addr, out_irq)
    return result
```

---

#### `of_irq_parse_raw()`

```c
int of_irq_parse_raw(const __be32 *addr, struct of_phandle_args *out_irq);
```

هذه الدالة هي قلب الـ interrupt parsing في الـ OF subsystem. تأخذ عنوان الجهاز (من `reg` property) والـ specifier الأولي المُخزّن في `out_irq`، وتُطبّق عليها آلية `interrupt-map` بشكل تكراري حتى تصل لـ controller جذري لا يحمل `interrupt-map` بنفسه.

**المعاملات:**
- `addr`: مؤشر لقيم عنوان الجهاز بصيغة big-endian (من `reg`). يُستخدم لمطابقة `interrupt-map-mask` + `interrupt-map`.
- `out_irq`: مدخل/مخرج — يحمل ابتداءً الـ specifier الأولي والـ controller المبدئي، وبعد الدالة يحمل النتيجة النهائية بعد تطبيق الـ map.

**القيمة المُعادة:** 0 عند النجاح، `-EINVAL` إذا كان format الـ `interrupt-map` غلط، `-ENOENT` إذا لم تُوجد مطابقة.

**تفاصيل مهمة:**
- تعمل بشكل تكراري (loop): كل iteration تُطبّق مرحلة mapping واحدة.
- تأخذ بعين الاعتبار `interrupt-map-mask` لإخفاء بعض bits قبل المطابقة.
- **الحالة الحرجة:** إذا كان `out_irq->np` لا يحمل `interrupt-map`، تتوقف وتُعيد النتيجة كما هي — وهذا هو الـ base case.
- تُستخدم أيضاً من الدوال التي تُعالج hierarchical interrupt domains.

**مثال تدفق DT:**
```
device: gpio-button {
    interrupts = <5 IRQ_TYPE_EDGE_FALLING>;
    interrupt-parent = <&gpio0>;
};

gpio0: gpio@... {
    interrupt-map = <5 0 &gic 0 42 IRQ_TYPE_EDGE_FALLING>;
    interrupt-map-mask = <0xf 0>;
};
```
→ `of_irq_parse_raw()` تحوّل `<5, FALLING>` عبر `gpio0` إلى `<&gic, 0, 42, FALLING>`.

---

#### `of_irq_parse_oldworld()`

```c
int of_irq_parse_oldworld(const struct device_node *device, int index,
                          struct of_phandle_args *out_irq);
```

متاحة فقط عند تفعيل `CONFIG_PPC32 && CONFIG_PPC_PMAC`. تُعالج طريقة تعريف الـ interrupts في أجهزة PowerMac القديمة (OldWorld) التي لا تستخدم `interrupt-parent` phandle الحديث. بدلاً من ذلك تعتمد على `of_irq_dflt_pic` كـ default interrupt controller وتتجاهل الـ phandle lookup.

**المعاملات:**
- `device`: الـ node الذي يحمل `AAPL,interrupts` property (الصيغة القديمة).
- `index`: رقم الـ interrupt.
- `out_irq`: بنية الإخراج.

**القيمة المُعادة:** 0 عند النجاح، `-EINVAL` على الأنظمة غير الـ OldWorld Mac.

**Workaround Flags:**
- `OF_IMAP_OLDWORLD_MAC` (0x1): تفعيل سلوك OldWorld.
- `OF_IMAP_NO_PHANDLE` (0x2): لا يُوجد phandle في الـ map، استخدم `of_irq_dflt_pic`.

---

### المجموعة الثالثة: High-Level IRQ Access — الواجهة العليا

هذه الدوال هي ما يستخدمه الـ driver مباشرة. تُخفي تفاصيل الـ parsing والـ mapping وراء واجهة بسيطة تُعيد رقم الـ Linux IRQ (virq) جاهزاً للاستخدام مع `request_irq()`.

---

#### `irq_of_parse_and_map()`

```c
unsigned int irq_of_parse_and_map(struct device_node *node, int index);
```

الدالة الأكثر استخداماً في drivers. تجمع `of_irq_parse_one()` + `irq_create_of_mapping()` في خطوة واحدة: تُحلّل الـ interrupt specifier من الـ DT ثم تُنشئ أو تجد الـ Linux virq المقابل في الـ `irq_domain` الصحيح.

**المعاملات:**
- `node`: الـ `device_node` الخاص بالجهاز.
- `index`: رقم الـ interrupt المطلوب (0 للأول).

**القيمة المُعادة:** رقم الـ Linux virq (양수) عند النجاح، 0 عند الفشل.

**تفاصيل مهمة:**
- متاحة حتى عند عدم تفعيل `CONFIG_OF_IRQ` إذا كان `CONFIG_SPARC` مُفعّلاً (SPARC لديه implementation مختلفة).
- تزيد refcount الـ virq داخلياً عبر الـ `irq_domain`.
- **لا تُستدعى من interrupt context** — يجب استدعاؤها فقط من probe أو init.

**مثال استخدام في driver:**
```c
/* في probe function */
int irq = irq_of_parse_and_map(dev->of_node, 0);
if (!irq) {
    dev_err(dev, "no IRQ found\n");
    return -EINVAL;
}
ret = devm_request_irq(dev, irq, my_handler, 0, "mydev", dev);
```

---

#### `of_irq_get()`

```c
int of_irq_get(struct device_node *dev, int index);
```

مشابهة لـ `irq_of_parse_and_map()` لكن تُعيد `int` بدلاً من `unsigned int`، مما يسمح بإعادة قيم خطأ سالبة. تستدعي `of_irq_parse_one()` ثم `irq_create_of_mapping()`.

**المعاملات:**
- `dev`: الـ `device_node`.
- `index`: رقم الـ interrupt.

**القيمة المُعادة:** رقم الـ virq عند النجاح، `-EPROBE_DEFER` إذا لم يُسجَّل الـ interrupt domain بعد (مهم جداً لـ deferred probing)، قيمة سالبة أخرى عند أخطاء أخرى.

**تفاصيل مهمة:**
- إعادة `-EPROBE_DEFER` هي الميزة الرئيسية التي تُميّزها عن `irq_of_parse_and_map()` — تُتيح لـ driver framework إعادة المحاولة لاحقاً.
- المُفضّلة في الكود الحديث على `irq_of_parse_and_map()`.

---

#### `of_irq_get_byname()`

```c
int of_irq_get_byname(struct device_node *dev, const char *name);
```

تبحث عن interrupt بالاسم في `interrupt-names` property، ثم تُحوّل الاسم إلى index وتستدعي `of_irq_get()`.

**المعاملات:**
- `dev`: الـ `device_node`.
- `name`: اسم الـ interrupt كما في `interrupt-names` (مثلاً `"rx"`, `"tx"`, `"err"`).

**القيمة المُعادة:** نفس `of_irq_get()` — virq أو قيمة خطأ سالبة.

**مثال DT:**
```dts
uart0: serial@... {
    interrupts = <4 IRQ_TYPE_LEVEL_HIGH>,
                 <5 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-names = "rx", "tx";
};
```
```c
int rx_irq = of_irq_get_byname(node, "rx"); /* يُعيد virq لـ interrupt رقم 4 */
int tx_irq = of_irq_get_byname(node, "tx"); /* يُعيد virq لـ interrupt رقم 5 */
```

---

#### `of_irq_get_affinity()`

```c
const struct cpumask *of_irq_get_affinity(struct device_node *dev, int index);
```

تستخرج CPU affinity المطلوبة للـ interrupt من الـ DT (إذا حدّدها firmware). تُستخدم في أنظمة SMP لتوزيع الـ interrupts على cores محددة كما طلبها الـ board designer.

**المعاملات:**
- `dev`: الـ `device_node`.
- `index`: رقم الـ interrupt.

**القيمة المُعادة:** مؤشر لـ `cpumask` ثابت، أو `NULL` إذا لم تُحدَّد affinity (يعني: استخدم الـ default).

**تفاصيل مهمة:**
- تعتمد على معلومات الـ affinity المُشفّرة في `irq_fwspec_info` التي تُوفّرها `irq_domain_ops::get_fwspec_info`.
- في الغالب تُعيد `NULL` على معظم platforms — الـ affinity الصريحة نادرة.

---

#### `of_irq_count()`

```c
int of_irq_count(struct device_node *dev);
```

تعدّ عدد الـ interrupts المُعرّفة في الـ node (سواء عبر `interrupts` أو `interrupts-extended`).

**المعاملات:**
- `dev`: الـ `device_node`.

**القيمة المُعادة:** عدد الـ interrupts (0 إذا لم تُوجد أي).

**الاستخدام النموذجي:**
```c
int n = of_irq_count(node);
/* تخصيص مصفوفة بحجم n ثم الـ loop */
for (int i = 0; i < n; i++)
    irqs[i] = of_irq_get(node, i);
```

---

### المجموعة الرابعة: Resource Management

تُحوّل هذه الدوال بيانات الـ IRQ من صيغة الـ OF إلى `struct resource` المستخدمة من الـ platform device layer وبقية الـ kernel subsystems.

---

#### `of_irq_to_resource()`

```c
int of_irq_to_resource(struct device_node *dev, int index,
                       struct resource *r);
```

تُحلّل الـ interrupt رقم `index` وتملأ `struct resource` بنوع `IORESOURCE_IRQ` بالمعلومات المناسبة (رقم الـ virq، flags الـ trigger type).

**المعاملات:**
- `dev`: الـ `device_node`.
- `index`: رقم الـ interrupt.
- `r`: مؤشر لـ `struct resource` التي ستُملأ.

**القيمة المُعادة:** رقم الـ virq عند النجاح (양수)، قيمة سالبة أو 0 عند الفشل.

**تفاصيل مهمة:**
- تضع اسم الـ node في `r->name` تلقائياً.
- الـ `r->flags` تحمل `IORESOURCE_IRQ` + trigger type flags مشتقة من الـ DT specifier.
- تُستخدم داخلياً من `of_device_alloc()` في `drivers/of/platform.c`.

---

#### `of_irq_to_resource_table()`

```c
int of_irq_to_resource_table(struct device_node *dev,
                             struct resource *res, int nr_irqs);
```

نسخة متعددة من `of_irq_to_resource()` — تملأ مصفوفة كاملة من الـ resources بكل الـ interrupts المُعرّفة في الـ node حتى `nr_irqs`.

**المعاملات:**
- `dev`: الـ `device_node`.
- `res`: مصفوفة `struct resource` مُخصّصة مسبقاً.
- `nr_irqs`: الحد الأقصى لعدد الـ interrupts التي ستُملأ.

**القيمة المُعادة:** عدد الـ interrupts الفعلية التي مُلئت.

---

### المجموعة الخامسة: Initialization — تهيئة Interrupt Controllers

---

#### `of_irq_init()`

```c
void of_irq_init(const struct of_device_id *matches);
```

تُهيّئ جميع الـ interrupt controllers المُعرّفة في الـ DT بالترتيب الصحيح (الـ parent قبل الـ child). هذه الدالة حاسمة جداً في early boot — تُنشئ الـ irq_domains وتُجهّز الـ hardware قبل أن تبدأ أي driver أخرى.

**المعاملات:**
- `matches`: مصفوفة `of_device_id` تحمل `compatible` strings للـ interrupt controllers المدعومة، مع `data` يُشير لـ `of_irq_init_cb_t` (دالة التهيئة).

**القيمة المُعادة:** void.

**تفاصيل مهمة:**
- تُنشئ قائمة مرتبة topologically من الـ controllers (الجذر أولاً).
- تُعيد المحاولة للـ controllers التي فشلت بسبب عدم تهيئة الـ parent بعد.
- تستدعي `cb(node, parent)` لكل controller حيث `parent` هو الـ controller الأب المُهيّأ بالفعل.
- تُستدعى مرة واحدة فقط من `start_kernel()` → `init_IRQ()` عبر platform-specific code.

**Pseudocode:**
```
of_irq_init(matches):
    // جمع كل الـ nodes المطابقة
    for_each_matching_node(node, matches):
        add to intc_desc_list with parent info

    // الترتيب الطوبولوجي والتهيئة
    while intc_desc_list not empty:
        for each desc in list where parent is initialized:
            ret = desc->irq_init_cb(desc->dev, desc->interrupt_parent)
            if ret == 0:
                mark as initialized
                move to initialized_list
            remove from intc_desc_list
```

---

### المجموعة السادسة: Mapping — تحويل Specifier إلى Linux IRQ

---

#### `irq_create_of_mapping()`

```c
unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data);
```

تأخذ `of_phandle_args` المُحلَّلة (تحمل `np` للـ controller وبيانات الـ specifier) وتُنشئ أو تجد الـ Linux virq المقابل عبر الـ `irq_domain` المُرتبط بذلك الـ controller.

**المعاملات:**
- `irq_data`: البنية المُملّأة من `of_irq_parse_one()` أو ما شابه.

**القيمة المُعادة:** Linux virq عند النجاح، 0 عند الفشل.

**تفاصيل مهمة:**
- تُحوّل `of_phandle_args` إلى `irq_fwspec` ثم تستدعي `irq_create_fwspec_mapping()`.
- تجد الـ `irq_domain` المناسب عبر `irq_find_host(irq_data->np)`.
- إذا كان الـ mapping موجوداً بالفعل، تُعيد الـ virq الموجود (idempotent).
- تستدعي `irq_domain->ops->xlate()` أو `translate()` لتحويل الـ specifier إلى HW irq number.

---

### المجموعة السابعة: imap Iterator — تكرار على `interrupt-map`

الـ `interrupt-map` property معقدة البنية وتحتاج iterator خاص لتفادي تعقيد الـ parsing اليدوي. هذه الدوال تُوفّر واجهة iterator نظيفة.

---

#### `of_imap_parser_init()`

```c
int of_imap_parser_init(struct of_imap_parser *parser,
                        struct device_node *node,
                        struct of_imap_item *item);
```

تُهيّئ الـ parser بقراءة الـ `interrupt-map` property من `node` وتجهيز الـ iterator للبدء من أول entry. تملأ الـ `item` الأولى تلقائياً ليكون جاهزاً للـ loop.

**المعاملات:**
- `parser`: بنية الـ iterator التي ستُهيّأ.
- `node`: الـ `device_node` الذي يحمل `interrupt-map`.
- `item`: بنية تُملأ بأول entry (أو تبقى فارغة إذا كانت الـ map فارغة).

**القيمة المُعادة:** 0 عند النجاح، `-EINVAL` إذا لم توجد `interrupt-map` property، `-ENOSYS` إذا لم يُفعَّل `CONFIG_OF_IRQ`.

**هيكل `struct of_imap_parser`:**
```c
struct of_imap_parser {
    struct device_node *node;   /* node يحمل interrupt-map */
    const __be32 *imap;         /* مؤشر للـ data الحالي */
    const __be32 *imap_end;     /* نهاية الـ data */
    u32 parent_offset;          /* عدد cells العنوان + interrupt للـ parent */
};
```

---

#### `of_imap_parser_one()`

```c
struct of_imap_item *of_imap_parser_one(struct of_imap_parser *parser,
                                        struct of_imap_item *item);
```

تُقرأ entry واحدة من الـ `interrupt-map` وتُحرّك الـ parser للـ entry التالية. هذه هي دالة الـ iteration الفعلية.

**المعاملات:**
- `parser`: حالة الـ iterator (تُعدَّل في كل استدعاء).
- `item`: البنية التي ستُملأ ببيانات الـ entry الحالية.

**القيمة المُعادة:** مؤشر لـ `item` عند وجود entry، `NULL` عند انتهاء الـ map.

**هيكل `struct of_imap_item`:**
```c
struct of_imap_item {
    struct of_phandle_args parent_args;  /* الـ controller الأب وبيانات الـ IRQ */
    u32 child_imap_count;                /* عدد cells الـ child (addr + irq) */
    u32 child_imap[16];                  /* بيانات الـ child (addr + irq specifier) */
};
```

**ملاحظة مهمة:** إذا خرجت من الـ loop مبكراً (`break`, `return`)، **يجب** استدعاء `of_node_put(item->parent_args.np)` يدوياً لتجنب memory leak.

---

#### `for_each_of_imap_item()` — Macro

```c
#define for_each_of_imap_item(parser, item) \
    for (; of_imap_parser_one(parser, item);)
```

Macro مريحة للتكرار على جميع entries في الـ `interrupt-map` بعد تهيئة الـ parser بـ `of_imap_parser_init()`.

**مثال استخدام:**
```c
struct of_imap_parser parser;
struct of_imap_item item;

/* تهيئة الـ parser */
ret = of_imap_parser_init(&parser, gpio_node, &item);
if (ret)
    return ret;

/* التكرار على كل entries */
for_each_of_imap_item(&parser, &item) {
    pr_info("child irq: %u, parent irq: %u\n",
            item.child_imap[1],
            item.parent_args.args[0]);
    /* of_node_put يُستدعى تلقائياً في next iteration */
}
/* عند الانتهاء الطبيعي: لا حاجة لـ of_node_put */
```

---

### المجموعة الثامنة: MSI — Message Signaled Interrupts

دوال MSI تُجسّر بين الـ OF device tree والـ MSI subsystem في الـ kernel. الـ PCIe وبعض أجهزة الـ platform تستخدم MSI بدلاً من الـ wired IRQs.

---

#### `of_msi_get_domain()`

```c
struct irq_domain *of_msi_get_domain(struct device *dev,
                                     const struct device_node *np,
                                     enum irq_domain_bus_token token);
```

تبحث في الـ DT عن الـ MSI controller المناسب للـ device وتُعيد الـ `irq_domain` المُسجَّل له. تستخدم `msi-parent` property أو تُطابق عبر الـ `token`.

**المعاملات:**
- `dev`: الـ `struct device` الخاص بالجهاز.
- `np`: الـ `device_node` المرتبط (قد يختلف عن `dev->of_node`).
- `token`: نوع الـ MSI domain المطلوب (مثلاً `DOMAIN_BUS_PCI_MSI`, `DOMAIN_BUS_PLATFORM_MSI`).

**القيمة المُعادة:** مؤشر لـ `struct irq_domain` عند النجاح، `NULL` عند الفشل.

**تفاصيل مهمة:**
- تبحث عبر `irq_find_matching_fwnode()` بعد تحويل الـ np إلى `fwnode`.
- إذا وُجد `msi-parent` property، تتبعه أولاً.
- **لا تزيد refcount** الـ domain — لا حاجة لـ put.

---

#### `of_msi_map_get_device_domain()`

```c
struct irq_domain *of_msi_map_get_device_domain(struct device *dev,
                                                u32 id,
                                                u32 bus_token);
```

أكثر تعقيداً من `of_msi_get_domain()` — تُطبّق الـ `msi-map` property لترجمة الـ device ID قبل البحث عن الـ domain. هذا ضروري في أنظمة IOMMU حيث كل device له RID (Requester ID) مختلف في مجال الـ MSI.

**المعاملات:**
- `dev`: الـ `struct device`.
- `id`: معرّف الجهاز (مثلاً PCI RID أو stream ID).
- `bus_token`: نوع الـ bus للـ domain المطلوب.

**القيمة المُعادة:** مؤشر لـ `irq_domain` أو `NULL`.

**تفاصيل مهمة:**
- تستدعي `of_map_id()` داخلياً لترجمة الـ ID عبر الـ `msi-map` table.
- تُستخدم بشكل رئيسي من الـ PCI/PCIe layer وبعض platform drivers.

---

#### `of_msi_configure()`

```c
void of_msi_configure(struct device *dev, const struct device_node *np);
```

تضبط الـ MSI domain على الـ `struct device` مباشرة عبر `dev_set_msi_domain()`. يُحوّل الـ device للاستخدام الآلي للـ MSI دون الحاجة لاستدعاءات صريحة في كل عملية.

**المعاملات:**
- `dev`: الـ `struct device` الذي سيُضبط عليه الـ MSI domain.
- `np`: الـ `device_node` المُستخدم للبحث عن الـ MSI parent.

**القيمة المُعادة:** void.

**تفاصيل مهمة:**
- تُستدعى عادةً من `of_device_alloc()` أو platform bus init code.
- بعد استدعائها، يمكن للـ driver استخدام `platform_get_irq()` أو `pci_enable_msi()` مباشرة.

---

#### `of_msi_xlate()`

```c
u32 of_msi_xlate(struct device *dev, struct device_node **msi_np, u32 id_in);
```

تُرجم الـ device ID المدخل عبر `msi-map` property للحصول على الـ ID الصحيح في نطاق الـ MSI controller. تُحدّث `*msi_np` بالـ controller node المناسب.

**المعاملات:**
- `dev`: الـ `struct device`.
- `msi_np`: مؤشر مزدوج — يُحدَّث ليُشير للـ MSI controller node.
- `id_in`: الـ device ID الأصلي (مثلاً PCI RID).

**القيمة المُعادة:** الـ ID المُترجَم في نطاق الـ MSI controller. يُعيد `id_in` بدون تغيير إذا لم توجد `msi-map` أو إذا كان `CONFIG_OF_IRQ` مُعطَّلاً.

---

### مخطط التدفق الكامل: من DT إلى `request_irq()`

```
DT property "interrupts"
        │
        ▼
of_irq_parse_one(node, index, &out_irq)
        │
        │ [يتبع interrupt-map إذا وُجدت]
        ▼
of_irq_parse_raw(addr, &out_irq)
        │
        │ out_irq.np  → interrupt controller node
        │ out_irq.args[] → HW irq specifier
        ▼
irq_create_of_mapping(&out_irq)
        │
        │ [يجد irq_domain عبر irq_find_host(out_irq.np)]
        │ [يستدعي domain->ops->xlate() لتحويل specifier → hwirq]
        │ [يُنشئ أو يجد Linux virq]
        ▼
     virq (Linux IRQ number)
        │
        ▼
devm_request_irq(dev, virq, handler, flags, name, data)
```

### مخطط علاقات الـ Structs

```
struct device_node (DT node)
    │
    │ interrupt-parent phandle
    ▼
struct device_node (interrupt controller)
    │
    │ irq_find_host()
    ▼
struct irq_domain
    │── ops->xlate()    → hwirq number
    │── ops->map()      → struct irq_desc
    └── revmap[]        → virq ↔ hwirq mapping

struct of_phandle_args
    │── np      → controller device_node
    └── args[]  → raw specifier (hwirq, type, ...)
```
## Phase 5: دليل الـ Debugging الشامل

---

### مقدمة

**الـ `of_irq`** subsystem مسؤول عن ترجمة مواصفات الـ interrupts من الـ Device Tree إلى أرقام IRQ منطقية (virtual IRQs) يستخدمها الـ kernel. المشاكل الشائعة تشمل: فشل الـ probe بسبب IRQ خاطئ، عدم إطلاق الـ interrupt، وأخطاء الـ irq_domain mapping.

---

## Software Level

### 1. مدخلات الـ debugfs

الـ kernel يعرض معلومات الـ irq_domain عبر `/sys/kernel/debug/irq/`:

```bash
# عرض كل irq_domains المسجّلة
ls /sys/kernel/debug/irq/domains/

# عرض تفاصيل domain معين (مثلاً GIC)
cat /sys/kernel/debug/irq/domains/GIC-0

# عرض كل mappings (hwirq → virq) داخل domain
cat /sys/kernel/debug/irq/domains/GIC-0/mappings

# IRQ domain tree كامل
cat /sys/kernel/debug/irq/irqs/<virq_number>
```

مثال على المخرج:
```
name:  GIC-0
 size:      0
 mapped:   64
 flags: 0x41
```
- **mapped**: عدد الـ IRQs المُرتبطة حالياً — إذا كان صفراً فالـ `irq_create_of_mapping()` لم تُنفَّذ.
- **flags**: تحتوي `IRQ_DOMAIN_FLAG_HIERARCHY` إذا كان الـ domain هرمياً.

```bash
# عرض MSI domains
ls /sys/kernel/debug/irq/domains/ | grep -i msi

# التحقق من irq_chip المرتبط بـ virq معين
cat /sys/kernel/debug/irq/irqs/42
```

---

### 2. مدخلات الـ sysfs

```bash
# قائمة كل IRQs النشطة مع عدد الـ triggers
cat /proc/interrupts

# عرض توزيع الـ IRQs على الـ CPUs
cat /proc/irq/<virq>/smp_affinity
cat /proc/irq/<virq>/smp_affinity_list

# البحث عن IRQ لـ device معين
grep "my_device" /proc/interrupts

# عرض إحصائيات الـ softirq
cat /proc/softirqs
```

مثال تفسير `/proc/interrupts`:
```
           CPU0       CPU1
 42:       1234          0   GIC-0  27 Level    uart0
```
- العمود الأول: رقم الـ virq.
- `GIC-0`: اسم الـ irq_domain.
- `27`: الـ hwirq داخل الـ controller.
- `Level`: نوع الـ trigger المُحدَّد من الـ DT.
- الصفر في CPU1 مع رقم كبير في CPU0: قد يدل على مشكلة affinity.

---

### 3. الـ ftrace — Tracepoints

```bash
# تفعيل tracepoints الخاصة بالـ irq subsystem
echo 1 > /sys/kernel/debug/tracing/events/irq/enable

# tracepoints محددة لتتبع of_irq
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable

# تتبع irq_domain mapping
echo 1 > /sys/kernel/debug/tracing/events/irqdomain/irqdomain_activate_irq/enable
echo 1 > /sys/kernel/debug/tracing/events/irqdomain/irqdomain_free_irqs/enable
echo 1 > /sys/kernel/debug/tracing/events/irqdomain/irqdomain_map/enable

# بدء الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

لتتبع دالة `of_irq_parse_one` تحديداً:
```bash
# function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo "of_irq_parse_one" > /sys/kernel/debug/tracing/set_ftrace_filter
echo "irq_create_of_mapping" >> /sys/kernel/debug/tracing/set_ftrace_filter
echo "irq_of_parse_and_map" >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

مثال مخرج ftrace:
```
kworker/0:2-42  [000] .... 1234.5678: of_irq_parse_one <-of_irq_get
kworker/0:2-42  [000] .... 1234.5679: irq_create_of_mapping <-irq_of_parse_and_map
```
إذا `of_irq_parse_one` لم تظهر أثناء probe → الـ DT لا يحتوي على `interrupts` property.

---

### 4. الـ printk و Dynamic Debug

```bash
# تفعيل dynamic debug لـ of_irq subsystem
echo "file drivers/of/irq.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/of/address.c +p" >> /sys/kernel/debug/dynamic_debug/control

# تفعيل كل debug messages في ملفات الـ OF
echo "file drivers/of/* +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع stack trace
echo "file drivers/of/irq.c +ps" > /sys/kernel/debug/dynamic_debug/control

# للـ MSI
echo "file drivers/pci/msi/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# مراقبة المخرجات
dmesg -w | grep -E "of_irq|irqdomain|irq_create"
```

لزيادة مستوى الـ loglevel أثناء الـ boot:
```bash
# في kernel command line
loglevel=8 dyndbg="file drivers/of/irq.c +p"
```

---

### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_OF_IRQ` | تفعيل الـ of_irq subsystem الأساسي |
| `CONFIG_IRQ_DOMAIN_DEBUG` | تفعيل `/sys/kernel/debug/irq/` |
| `CONFIG_GENERIC_IRQ_DEBUGFS` | debugfs مفصّل للـ IRQs |
| `CONFIG_SPARSE_IRQ` | دعم الـ sparse IRQ numbers |
| `CONFIG_IRQ_DOMAIN` | البنية الأساسية للـ irq_domain |
| `CONFIG_HARDIRQS_SW_RESEND` | إعادة إرسال الـ IRQ بالـ software |
| `CONFIG_IRQSOFF_TRACER` | تتبع فترات إيقاف الـ IRQs |
| `CONFIG_DEBUG_SHIRQ` | اختبار الـ shared IRQs |
| `CONFIG_LOCKDEP` | كشف deadlocks في الـ IRQ handlers |
| `CONFIG_DEBUG_IRQFLAGS` | التحقق من صحة IRQ flags |
| `CONFIG_PROVE_LOCKING` | إثبات صحة الـ locking |
| `CONFIG_OF_DYNAMIC` | دعم الـ dynamic DT overlays |

```bash
# التحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(OF_IRQ|IRQ_DOMAIN|GENERIC_IRQ)"
```

---

### 6. أدوات خاصة بالـ Subsystem

```bash
# عرض شجرة الـ Device Tree الكاملة
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# البحث عن nodes التي تحتوي على interrupts
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "interrupts"

# استخدام /proc/device-tree
find /proc/device-tree -name "interrupts" | while read f; do
    echo "=== $f ==="; xxd "$f"
done

# أداة device-tree compiler للتحليل
dtc -I dtb -O dts /boot/dtb/your-board.dtb | grep -A10 "interrupt"

# فحص الـ irq_domain tree
cat /sys/kernel/debug/irq/domains/*/name 2>/dev/null

# عرض الـ IRQ affinity لجميع الـ interrupts
for i in /proc/irq/*/smp_affinity_list; do
    echo "$i: $(cat $i)"
done
```

---

### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `irq: no irq domain found for /soc/uart@...` | لم يُسجَّل أي `irq_domain` لهذا الـ interrupt controller | تأكد أن driver الـ interrupt controller يستدعي `irq_domain_add_*()` قبل الـ child drivers |
| `of_irq_parse_one: parent interrupts-extended property not found` | الـ node لا يحتوي `interrupts` أو `interrupts-extended` | أضف property الـ `interrupts` للـ DTS |
| `irq_create_of_mapping: invalid irq-spec` | مواصفات الـ interrupt في الـ DT غير صحيحة (عدد الخلايا خاطئ) | طابق `#interrupt-cells` مع عدد قيم `interrupts` |
| `error -22 mapping irq` | `xlate()` callback أعادت `-EINVAL` | تحقق من قيم الـ interrupt type في الـ DTS |
| `irq_find_matching_fwnode: no fwnode` | الـ `fwnode` لم يُربط بـ irq_domain | تأكد من `irq_domain_add_linear()` بعد تهيئة الـ fwnode |
| `of_irq_find_parent: interrupt-parent not found` | الـ `interrupt-parent` phandle مكسور | تحقق من صحة الـ phandle في DTS |
| `msi: no msi domain found` | لم يُعثر على MSI domain للـ device | تأكد من تسجيل MSI domain أو استخدام `of_msi_configure()` |
| `irq: type mismatch, failed to map hwirq` | نوع الـ trigger (level/edge) غير متوافق مع الـ hardware | صحح `IRQ_TYPE_*` في DTS |
| `genirq: Flags mismatch irq` | تعارض بين flags عند `request_irq()` مقارنةً بالـ DT | استخدم `irqd_get_trigger_type()` للتأكد |
| `of_irq_parse_raw: interrupt-map property is too small` | `interrupt-map` في الـ DTS مبتورة | أكمل الـ map بالقيم الصحيحة |

---

### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في of_irq_parse_one() — للتحقق من صحة الـ output */
int of_irq_parse_one(struct device_node *device, int index,
                     struct of_phandle_args *out_irq)
{
    int rc;
    rc = /* ... */;

    /* نقطة فحص: هل الـ irq_domain موجود؟ */
    WARN_ON(!out_irq->np);

    /* نقطة فحص: هل عدد الـ args منطقي؟ */
    WARN_ON(out_irq->args_count > MAX_PHANDLE_ARGS);

    return rc;
}

/* في irq_create_of_mapping() — فحص الـ domain */
unsigned int irq_create_of_mapping(struct of_phandle_args *irq_data)
{
    struct irq_domain *domain;

    domain = irq_find_host(irq_data->np);

    /* إذا لم يُعثر على domain → dump_stack لمعرفة من استدعى */
    if (!domain) {
        dump_stack();
        pr_err("of_irq: no domain for node %pOF\n", irq_data->np);
        return 0;
    }
    /* ... */
}

/* في الـ driver عند request_irq() */
static int my_driver_probe(struct platform_device *pdev)
{
    int irq = platform_get_irq(pdev, 0);

    /* WARN إذا كان الـ IRQ سالباً (خطأ في الـ DT أو الـ mapping) */
    if (WARN_ON(irq <= 0)) {
        dev_err(&pdev->dev, "invalid IRQ %d\n", irq);
        return irq ? irq : -EINVAL;
    }
    /* ... */
}
```

**النقاط الأكثر قيمة:**
- قبل `irq_create_of_mapping()`: فحص `irq_data->np != NULL`.
- بعد `of_irq_parse_one()`: فحص `args_count` مطابق لـ `#interrupt-cells`.
- في `of_irq_init()`: بعد استدعاء كل `of_irq_init_cb_t` callback.
- في `of_imap_parser_one()`: فحص عدم تجاوز `imap_end`.

---

## Hardware Level

### 1. التحقق من توافق حالة الـ Hardware مع الـ Kernel

```bash
# قراءة حالة الـ interrupt controller (مثال: GIC)
# التحقق أن الـ hwirq مُفعَّل في GICD_ISENABLER
devmem2 0x08000100 w    # GICD_ISENABLER0 — bit يعني IRQ مُفعَّل

# التحقق من pending IRQs
devmem2 0x08000200 w    # GICD_ISPENDR0

# مقارنة مع ما يعرفه الـ kernel
cat /proc/interrupts | grep "GIC"

# التحقق من الـ trigger type المُبرمَج في الـ hardware
devmem2 0x08000C00 w    # GICD_ICFGR0 — 00=level, 10=edge
```

التطابق المطلوب:
- إذا DTS يقول `IRQ_TYPE_LEVEL_HIGH` → GICD_ICFGR يجب أن يكون `00` في الـ bit المقابل.
- إذا DTS يقول `IRQ_TYPE_EDGE_RISING` → يجب أن يكون `10`.

### 2. تقنيات الـ Register Dump

```bash
# باستخدام devmem2 (يجب تثبيته: apt install devmem2)
# قراءة سجل الـ GIC Distributor base
devmem2 0x08000000 w    # GICD_CTLR
devmem2 0x08000004 w    # GICD_TYPER — عدد الـ IRQs المدعومة

# باستخدام /dev/mem مع Python
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x08000000)
    val = struct.unpack('<I', m[0:4])[0]
    print(f'GICD_CTLR = 0x{val:08x}')
    m.close()
"

# باستخدام io utility (من busybox)
io -4 -r 0x08000000

# dump كامل لـ 256 bytes من base address
devmem2 0x08000000 && for offset in $(seq 0 4 252); do
    printf "0x%08x: " $((0x08000000 + offset))
    devmem2 $((0x08000000 + offset)) w 2>/dev/null | tail -1
done
```

**تحذير:** تعطيل `CONFIG_STRICT_DEVMEM` مطلوب للوصول لعناوين لا تخص الـ RAM. استخدم فقط في بيئة التطوير.

---

### 3. نصائح الـ Logic Analyzer / Oscilloscope

**نقاط القياس المهمة:**

```
  Device ──IRQ line──→ Interrupt Controller ──→ CPU
     │                        │
  [TP1]                    [TP2]
```

- **TP1** (بين الـ device والـ controller): قياس الـ signal المادي.
- **TP2** (مخرج الـ controller للـ CPU): التحقق من الـ trigger.

**إعدادات الـ Logic Analyzer:**

| المعلمة | القيمة الموصى بها |
|---|---|
| Sample rate | ≥ 10× أعلى تردد IRQ متوقع |
| Trigger | Rising/Falling edge حسب DTS |
| Capture depth | ≥ 10ms لالتقاط الـ burst events |
| Protocol decoder | I2C/SPI إذا كان الـ IRQ من كنترولر خارجي |

**سيناريوهات المشاكل:**

1. **Glitch على الـ IRQ line**: يسبب spurious interrupts → `cat /proc/irq/<N>/spurious`.
2. **IRQ لا يُطلَق أبداً**: تحقق بـ oscilloscope من وجود signal على الـ pin المادي.
3. **Level interrupt لا يُحرَّر**: الـ signal يبقى HIGH/LOW → الـ device لم يُقرأ status register → handler لا يُكمل.

---

### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | ما تراه في dmesg | التشخيص |
|---|---|---|
| IRQ wire غير موصول | `irq <N>: nobody cared` ثم `Disabling IRQ #N` | تحقق من الـ schematic والتوصيلات المادية |
| Shared IRQ لا يُحرَّر | `irq N: disabled; was enabled on boot` | الـ handler لا يُعيد `IRQ_HANDLED`، فحص `irq_stat` |
| Level IRQ عالق (stuck) | طوفان من `irq N: nobody cared` | مشكلة في عدم مسح interrupt flag في الـ device |
| Voltage mismatch | الـ IRQ لا يُرصد أبداً | قياس voltage بـ multimeter على GPIO pin |
| Wrong polarity | `spurious irq N` | اعكس `IRQ_TYPE_LEVEL_HIGH` إلى `IRQ_TYPE_LEVEL_LOW` |
| Clock gate للـ controller | فشل `of_irq_init` callback بـ `-EPROBE_DEFER` | تأكد من تفعيل الـ clock قبل الـ probe |

---

### 5. تصحيح الـ Device Tree

```bash
# قراءة الـ DT الحالي من الـ kernel (ما يستخدمه فعلياً)
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null > /tmp/current.dts

# البحث عن كل nodes التي تحتوي interrupts
grep -n "interrupts\b" /tmp/current.dts

# التحقق من #interrupt-cells
grep -B20 "interrupts = " /tmp/current.dts | grep "interrupt-cells"

# مقارنة عدد الـ args مع #interrupt-cells
# مثال GIC: #interrupt-cells = <3> يعني: <type hwirq flags>
# interrupts = <GIC_SPI 27 IRQ_TYPE_LEVEL_HIGH>

# التحقق من interrupt-parent
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -E "(interrupt-parent|interrupts-extended|#interrupt-cells)"

# فحص interrupt-map في bridge nodes
grep -A30 "interrupt-map\b" /tmp/current.dts

# مقارنة DT blob مع المتوقع
diff <(dtc -I dtb -O dts /boot/my-board.dtb 2>/dev/null) \
     <(dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null)
```

**قائمة تحقق الـ DT للـ Interrupts:**

```
[ ] interrupt-controller موجود في node الـ controller
[ ] #interrupt-cells محدد بالقيمة الصحيحة (1، 2، أو 3 للـ GIC)
[ ] interrupt-parent يشير للـ node الصحيح
[ ] عدد قيم interrupts = مضاعف #interrupt-cells
[ ] نوع الـ trigger صحيح (IRQ_TYPE_LEVEL_HIGH/LOW, EDGE_RISING/FALLING)
[ ] hwirq ضمن نطاق الـ controller (< GICD_TYPER.ITLinesNumber × 32)
[ ] للـ MSI: msi-parent أو msi-map موجود
```

---

## Practical Commands

### 1. أوامر جاهزة للنسخ

**تشخيص شامل لـ of_irq مرة واحدة:**

```bash
#!/bin/bash
# of_irq_debug.sh — تشخيص شامل لـ OF IRQ subsystem

echo "=== IRQ Domains ==="
ls /sys/kernel/debug/irq/domains/ 2>/dev/null || echo "debugfs not mounted"

echo -e "\n=== Active IRQs ==="
cat /proc/interrupts

echo -e "\n=== Device Tree IRQ properties ==="
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -E "(interrupts|interrupt-parent|#interrupt-cells)" | head -40

echo -e "\n=== IRQ Domain Details ==="
for domain in /sys/kernel/debug/irq/domains/*/; do
    echo "--- $(basename $domain) ---"
    cat "${domain}name" 2>/dev/null
    ls "${domain}mappings" 2>/dev/null && cat "${domain}mappings" 2>/dev/null | head -20
done

echo -e "\n=== Spurious IRQ counters ==="
for d in /proc/irq/*/spurious; do
    count=$(awk '/count/{print $2}' "$d" 2>/dev/null)
    [ "$count" != "0" ] && echo "$d: $count"
done
```

**تفعيل ftrace لـ of_irq:**
```bash
# إعداد سريع لـ tracing
mount -t debugfs none /sys/kernel/debug 2>/dev/null
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo "of_irq_parse_one irq_create_of_mapping irq_of_parse_and_map" > set_ftrace_filter
echo 1 > tracing_on
# ... reproduce the issue ...
echo 0 > tracing_on
cat trace | grep -v "^#" | head -50
```

**تفعيل dynamic debug لـ of/irq:**
```bash
mount -t debugfs none /sys/kernel/debug 2>/dev/null
echo "file drivers/of/irq.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "module irqdomain +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &
# ... trigger probe ...
kill %1
```

---

### 2. مثال على المخرجات وكيفية التفسير

**مخرج `/proc/interrupts` مع مشكلة:**
```
           CPU0       CPU1
 27:          0          0   GIC-0  59 Level    my_device
```
- العداد صفر على كلا الـ CPUs رغم وجود activity على الـ device → الـ IRQ لا يصل للـ CPU.
- الخطوة التالية: تحقق بـ oscilloscope من الـ pin المادي، ثم فحص `GICD_ISENABLER`.

**مخرج `cat /sys/kernel/debug/irq/domains/GIC-0`:**
```
name:  GIC-0
 size:      0
 mapped:    3
 flags: 0x41
  hwirq=0x3b  virq=27  [GIC_SPI 27 LEVEL]
```
- `hwirq=0x3b` = 59 decimal = GIC SPI 27 (GIC SPI يبدأ من offset 32، إذن SPI 27 = hwirq 59).
- إذا لم يظهر الـ mapping هنا → `irq_create_of_mapping()` فشلت.

**مخرج ftrace للـ mapping:**
```
kworker-42  [000] 1234.567: of_irq_parse_one: node=my_device, index=0
kworker-42  [000] 1234.568: of_irq_parse_one: intspec[0]=0 intspec[1]=27 intspec[2]=4
kworker-42  [000] 1234.569: irq_create_of_mapping: hwirq=59, type=4 → virq=27
```
- `intspec[2]=4` = `IRQ_TYPE_LEVEL_HIGH`.
- إذا لم تظهر هذه السطور → الـ `of_irq_parse_one` لم تُستدعَ = مشكلة في الـ DT.

**تشخيص `irq_domain` missing:**
```bash
$ dmesg | grep "no irq domain"
[    2.345] irq: no irq domain found for /soc/uart@ff000000 !
```
→ الـ interrupt controller الأب لم يُسجَّل بعد. الحل: تأكد أن الـ controller driver يُحمَّل أولاً (ترتيب الـ `initcall` أو `probe_defer`).

```bash
# فحص ترتيب الـ probe
dmesg | grep -E "(probe|irq_domain_add)" | head -30
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART لا يستقبل interrupts على بورد مبني على RK3562

#### العنوان
**الـ UART driver لا يستجيب على بورد صناعي مبني على RK3562**

#### السياق
شركة تصنع **industrial gateway** يعمل عليه Linux لجمع بيانات من حساسات RS-485. البورد مبني على **Rockchip RK3562**، وتم نقل DTS من مشروع سابق. عند تشغيل النظام، الـ `/dev/ttyS3` موجود لكن لا يستقبل أي بيانات رغم أن الأسلاك صحيحة.

#### المشكلة
الـ UART driver لا يحصل على **Linux IRQ** لأن `irq_of_parse_and_map()` تعيد 0، مما يجعل `request_irq()` يفشل صامتاً أو تعمل على polling mode بأداء رديء.

#### التحليل
**الـ** `irq_of_parse_and_map()` المُعرَّفة في `of_irq.h`:

```c
/* تُنفَّذ فقط إذا كان CONFIG_OF_IRQ أو CONFIG_SPARC مفعَّلاً */
extern unsigned int irq_of_parse_and_map(struct device_node *node, int index);
```

مسار التنفيذ:

```
irq_of_parse_and_map(uart_node, 0)
    └─> of_irq_parse_one(uart_node, 0, &oirq)
            └─> يقرأ خاصية "interrupts" من DTS
            └─> يستدعي of_irq_find_parent() للوصول للـ interrupt controller
            └─> يُعبئ struct of_phandle_args بـ (np, args[])
    └─> irq_create_of_mapping(&oirq)
            └─> يبحث عن irq_domain المناسب
            └─> يُنشئ mapping بين HW IRQ و Linux virq
```

الفحص الأول:

```bash
# تحقق من قراءة الـ interrupts property
cat /sys/firmware/devicetree/base/serial@fe670000/interrupts
# إذا فارغ أو غير موجود → مشكلة DTS

# تحقق من الـ irq domain المسجَّل
cat /proc/interrupts | grep uart
# إذا لا يوجد سطر → الـ mapping لم يتم
```

الفحص في dmesg:

```bash
dmesg | grep -i "irq\|uart\|serial"
# ابحث عن: "no irq domain found" أو "IRQ not found"
```

النظر في DTS المنقول — وُجد الخطأ:

```dts
/* DTS الخاطئ: interrupt-parent يشير إلى node خاطئ */
serial@fe670000 {
    compatible = "rockchip,rk3562-uart", "snps,dw-apb-uart";
    reg = <0x0 0xfe670000 0x0 0x100>;
    interrupts = <GIC_SPI 116 IRQ_TYPE_LEVEL_HIGH>;
    /* interrupt-parent مفقود! سيبحث kernel في الـ parent node */
};
```

**الـ** `of_irq_find_parent()` تصعد في شجرة الـ DT بحثاً عن `interrupt-parent`:

```c
/* تبحث عن أقرب ancestor يملك interrupt-parent */
extern struct device_node *of_irq_find_parent(struct device_node *child);
```

إذا كان الـ parent node الصحيح لا يملك `#interrupt-cells`، تفشل `of_irq_parse_one()`.

#### الحل

```dts
/* DTS الصحيح */
/ {
    gic: interrupt-controller@fd400000 {
        compatible = "arm,gic-v3";
        #interrupt-cells = <3>;
        interrupt-controller;
        /* ... */
    };

    serial@fe670000 {
        compatible = "rockchip,rk3562-uart", "snps,dw-apb-uart";
        reg = <0x0 0xfe670000 0x0 0x100>;
        interrupt-parent = <&gic>;          /* إضافة صريحة */
        interrupts = <GIC_SPI 116 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&cru SCLK_UART3>, <&cru PCLK_UART3>;
        clock-names = "baudclk", "apb_pclk";
        status = "okay";
    };
};
```

التحقق بعد الإصلاح:

```bash
# يجب أن يظهر mapping صحيح
cat /proc/interrupts | grep uart3
# 148:    0    0    0    0   GICv3 116 Level    fe670000.serial
```

#### الدرس المستفاد
`of_irq_find_parent()` تصعد في الشجرة تلقائياً، لكن عند نقل DTS بين منصات مختلفة يجب التصريح بـ `interrupt-parent` بشكل صريح لأن تركيب شجرة الـ GIC يختلف من SoC لآخر.

---

### السيناريو الثاني: SPI controller على STM32MP1 يُشغّل interrupt خاطئ

#### العنوان
**الـ SPI driver يُطلق interrupt handler للـ DMA بدلاً من الـ SPI على STM32MP1**

#### السياق
فريق يطور **IoT sensor hub** على **STM32MP157C**. البورد يحتوي SPI متصل بـ 4 حساسات IMU. بعد التشغيل، الـ SPI يعمل لفترة ثم يتجمد، والـ kernel log يُظهر `unexpected IRQ`.

#### المشكلة
الـ DTS يحتوي خاصية `interrupts` بقيمتين (SPI IRQ + DMA IRQ) لكن الـ driver يستخدم `of_irq_get(dev->of_node, 0)` بينما يجب أن يستخدم `of_irq_get_byname()`.

#### التحليل
الفرق بين الدالتين في `of_irq.h`:

```c
/* تحصل على IRQ بالترتيب (index) */
extern int of_irq_get(struct device_node *dev, int index);

/* تحصل على IRQ بالاسم من خاصية interrupt-names */
extern int of_irq_get_byname(struct device_node *dev, const char *name);
```

**الـ** DTS الخاطئ:

```dts
/* STM32MP1 SPI DTS الخاطئ */
spi2: spi@4400b000 {
    compatible = "st,stm32h7-spi";
    reg = <0x4400b000 0x400>;
    /* index 0 = SPI IRQ، index 1 = DMA IRQ */
    interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 56 IRQ_TYPE_LEVEL_HIGH>;
    /* interrupt-names مفقود! */
};
```

الـ driver يستخدم:

```c
/* في driver الخاطئ */
irq = of_irq_get(pdev->dev.of_node, 0); /* يحصل على index 0 = صحيح صدفةً */
dma_irq = of_irq_get(pdev->dev.of_node, 0); /* BUG: يجب أن يكون index 1 */
```

`of_irq_get()` تستدعي داخلياً `of_irq_parse_one()` التي تقرأ الـ `interrupts` property بالـ index المُعطى:

```c
extern int of_irq_parse_one(struct device_node *device, int index,
                             struct of_phandle_args *out_irq);
```

نتيجة: كلا الـ handler يُسجَّل على نفس الـ virq، وعند حدوث DMA interrupt يتم استدعاء SPI handler مما يُسبب corruption.

فحص عدد الـ interrupts:

```bash
# of_irq_count تعطي عدد الـ interrupts في الـ node
# يمكن محاكاتها بـ:
grep -c "interrupts" /sys/firmware/devicetree/base/spi@4400b000/interrupts
# أو:
ls /proc/irq/ | wc -l
```

#### الحل

```dts
/* DTS الصحيح */
spi2: spi@4400b000 {
    compatible = "st,stm32h7-spi";
    reg = <0x4400b000 0x400>;
    interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 56 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-names = "spi", "dma";   /* إضافة الأسماء */
    dmas = <&dmamux1 39 0x400 0x05>,
           <&dmamux1 40 0x400 0x05>;
    dma-names = "rx", "tx";
    status = "okay";
};
```

```c
/* في driver الصحيح */
irq = of_irq_get_byname(pdev->dev.of_node, "spi");
if (irq < 0) {
    /* fallback لـ index 0 للتوافق مع DTB قديم */
    irq = of_irq_get(pdev->dev.of_node, 0);
}

dma_irq = of_irq_get_byname(pdev->dev.of_node, "dma");
/* لا تستخدم dma_irq مباشرة - DMA engine يديره بنفسه */
```

```bash
# تحقق من عدد الـ interrupts المُسجَّلة
cat /proc/interrupts | grep spi
# يجب أن يظهر سطر واحد فقط للـ SPI
```

#### الدرس المستفاد
عند وجود أكثر من interrupt في node واحد، استخدم دائماً `interrupt-names` و `of_irq_get_byname()` بدلاً من الاعتماد على الترتيب. الترتيب عرضة للتغيير عند تحديث DTS.

---

### السيناريو الثالث: Android TV Box على Allwinner H616 — HDMI لا يُشغَّل HPD interrupt

#### العنوان
**الـ HDMI Hot-Plug Detection لا يعمل على Android TV Box مبني على Allwinner H616**

#### السياق
مصنع **Android TV box** يستخدم **Allwinner H616** مع Android 11. المشكلة: عند توصيل HDMI، الشاشة لا تُكتشف تلقائياً إلا بعد إعادة تشغيل. فريق BSP يُحقق في الأمر.

#### المشكلة
الـ HDMI driver يستخدم interrupt من **GPIO controller** المُنفَّذ كـ **interrupt parent** منفصل عبر `interrupt-map`، لكن الـ `of_imap_parser` لا يُفسَّر بشكل صحيح لأن `#interrupt-cells` الخاص بالـ GPIO خاطئ.

#### التحليل
الـ H616 يستخدم GPIO كـ interrupt controller ثانوي مرتبط بالـ GIC عبر interrupt-map. الـ `of_irq.h` يوفر آلية التفسير:

```c
/* struct للتنقل في جدول interrupt-map */
struct of_imap_parser {
    struct device_node *node;     /* الـ node الحالي */
    const __be32 *imap;           /* مؤشر لبداية الجدول */
    const __be32 *imap_end;       /* نهاية الجدول */
    u32 parent_offset;            /* offset للـ parent */
};

/* كل entry في الجدول */
struct of_imap_item {
    struct of_phandle_args parent_args; /* args للـ parent interrupt controller */
    u32 child_imap_count;               /* عدد cells للـ child */
    u32 child_imap[16];                 /* child interrupt specifier */
};
```

الـ DTS الخاطئ:

```dts
/* Allwinner H616 GPIO كـ interrupt controller */
pio: pinctrl@300b000 {
    compatible = "allwinner,sun50i-h616-pinctrl";
    reg = <0x0300b000 0x400>;
    interrupt-controller;
    #interrupt-cells = <3>;   /* خطأ! يجب أن يكون <2> لـ GPIO interrupts */
    gpio-controller;
    #gpio-cells = <3>;

    interrupt-parent = <&gic>;
    interrupts = <GIC_SPI 51 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 52 IRQ_TYPE_LEVEL_HIGH>,
                 /* ... */;
};

hdmi: hdmi@6000000 {
    compatible = "allwinner,sun50i-h616-dw-hdmi";
    /* ... */
    interrupt-parent = <&pio>;
    interrupts = <7 2 IRQ_TYPE_EDGE_RISING>;  /* 3 cells بسبب الخطأ أعلاه */
};
```

عند استدعاء `of_irq_parse_one()` على الـ HDMI node:

```
of_irq_parse_one(hdmi_node, 0, &oirq)
    └─> يقرأ interrupts = <7 2 IRQ_TYPE_EDGE_RISING> (3 u32s)
    └─> يجد interrupt-parent = &pio
    └─> يقرأ #interrupt-cells من pio = 3 ✓ (يتطابق عدد القيم)
    └─> of_irq_parse_raw() يُحاول ترجمتها
    └─> pio driver يتوقع فقط <bank irq_num> (2 cells)
    └─> FAILS: xlate function ترفض 3 cells
```

فحص المشكلة:

```bash
# تحقق من #interrupt-cells الفعلي
xxd /sys/firmware/devicetree/base/pinctrl@300b000/\#interrupt-cells
# يجب أن يعطي: 00 00 00 02 (القيمة 2)

# تحقق من الـ irq domain المسجَّل
cat /sys/kernel/debug/irq/domains/  2>/dev/null || \
cat /sys/kernel/debug/irqdomain_mapping 2>/dev/null

# أو:
dmesg | grep "pio\|pinctrl\|irq domain"
```

`of_irq_parse_raw()` تستخدم `of_imap_parser` إذا وُجد `interrupt-map` في الأب:

```c
/* تُفسِّر raw interrupt specifier مع مراعاة interrupt-map */
extern int of_irq_parse_raw(const __be32 *addr, struct of_phandle_args *out_irq);
```

#### الحل

```dts
/* DTS الصحيح */
pio: pinctrl@300b000 {
    compatible = "allwinner,sun50i-h616-pinctrl";
    reg = <0x0300b000 0x400>;
    interrupt-controller;
    #interrupt-cells = <2>;   /* تصحيح: bank + irq_num فقط */
    gpio-controller;
    #gpio-cells = <3>;

    interrupt-parent = <&gic>;
    interrupts = <GIC_SPI 51 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 52 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 53 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 54 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 55 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 56 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 57 IRQ_TYPE_LEVEL_HIGH>,
                 <GIC_SPI 58 IRQ_TYPE_LEVEL_HIGH>;
};

hdmi: hdmi@6000000 {
    compatible = "allwinner,sun50i-h616-dw-hdmi";
    interrupt-parent = <&pio>;
    interrupts = <7 2>;   /* تصحيح: 2 cells فقط (bank=7, irq=2) */
};
```

التحقق:

```bash
# بعد إعادة التشغيل
dmesg | grep "hdmi\|HPD"
# يجب أن يظهر: "sun6i-hdmi 6000000.hdmi: registered IRQ 143"

# اختبار HPD
udevadm monitor --property | grep HDMI
# عند توصيل كابل HDMI يجب أن يظهر event
```

#### الدرس المستفاد
قيمة `#interrupt-cells` يجب أن تتطابق تماماً مع عدد الـ u32 في خاصية `interrupts` الخاصة بكل device يستخدم هذا الـ controller كـ parent. خطأ واحد في هذه القيمة يُصمت جميع الـ interrupts الصادرة من هذا الـ controller.

---

### السيناريو الرابع: I2C على i.MX8M Plus — interrupt affinity يُسبب latency في Automotive ECU

#### العنوان
**الـ I2C interrupt يُعالَج على CPU بطيء في ECU مبني على i.MX8M Plus مما يُسبب timeout**

#### السياق
شركة automotive تطور **ECU** (Electronic Control Unit) لنظام ADAS على **NXP i.MX8M Plus**. الـ ECU يستقبل بيانات حساسات LiDAR عبر I2C بسرعة 400KHz. المشكلة: يحدث I2C timeout عشوائي تحت load عالي.

#### المشكلة
الـ i.MX8M Plus يملك Cortex-A53 cluster (4 cores) و Cortex-M7. الـ I2C interrupt يُعالَج على CPU0 الذي يحمل أيضاً workload ثقيل. الحل هو **pin the interrupt** إلى CPU3 (dedicated للـ real-time tasks).

#### التحليل
الـ `of_irq.h` يوفر دالة الـ affinity:

```c
/* تعيد cpumask المُحدَّد في DTS للـ interrupt */
extern const struct cpumask *of_irq_get_affinity(struct device_node *dev,
                                                  int index);
```

هذه الدالة تقرأ خاصية `affinity` من الـ DTS — الـ feature الحديث في kernel الذي يسمح بتحديد affinity في DTS بدلاً من `/proc/irq/N/smp_affinity`.

فحص الوضع الحالي:

```bash
# اعثر على الـ IRQ number للـ I2C
cat /proc/interrupts | grep i2c
# 56:    14523    0    0    0   GICv3  68 Level  30a30000.i2c

# تحقق من الـ affinity الحالي
cat /proc/irq/56/smp_affinity
# f (كل الـ cores) - هذه المشكلة

# تحقق من load على كل CPU
mpstat -P ALL 1 5
# CPU0: 95% - مزدحم جداً
```

في الـ driver:

```c
/* في i.MX8 I2C driver: drivers/i2c/busses/i2c-imx.c */
static int i2c_imx_probe(struct platform_device *pdev)
{
    /* ... */
    irq = platform_get_irq(pdev, 0);
    /* platform_get_irq تستدعي داخلياً irq_of_parse_and_map */

    /* affinity من DTS */
    aff = of_irq_get_affinity(pdev->dev.of_node, 0);
    if (aff) {
        /* تطبيق الـ affinity المُحدَّد في DTS */
        irq_set_affinity_hint(irq, aff);
    }
}
```

`of_irq_get_affinity()` تستخدم `of_irq_parse_one()` ثم تبحث عن معلومات الـ affinity في الـ irq_domain عبر `irq_fwspec_info`:

```c
struct irq_fwspec_info {
    unsigned long    flags;
    const struct cpumask *affinity;  /* الـ cpumask المطلوب */
};
#define IRQ_FWSPEC_INFO_AFFINITY_VALID  BIT(0)
```

#### الحل

```dts
/* i.MX8M Plus DTS - تحديد affinity للـ I2C الحرج */
&i2c3 {
    clock-frequency = <400000>;
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c3>;
    status = "okay";

    /* pin this interrupt to CPU3 (dedicated RT core) */
    affinity = <3>;   /* CPU3 فقط */

    lidar_sensor: lidar@48 {
        compatible = "vendor,lidar-sensor";
        reg = <0x48>;
    };
};
```

إذا لم يدعم الـ kernel خاصية `affinity` في DTS (kernel قديم):

```bash
# تعيين يدوي عبر sysfs (مؤقت - يُفقد عند reboot)
echo 8 > /proc/irq/56/smp_affinity  # CPU3 = bit 3 = 0b1000 = 8

# جعله دائماً عبر udev rule:
cat > /etc/udev/rules.d/99-irq-affinity.rules << 'EOF'
SUBSYSTEM=="platform", KERNEL=="30a30000.i2c", ACTION=="add", \
    RUN+="/bin/sh -c 'echo 8 > /proc/irq/$(cat /sys/bus/platform/devices/30a30000.i2c/irq)/smp_affinity'"
EOF
```

التحقق من تحسّن الـ latency:

```bash
# قياس I2C transfer latency
i2c-tools: i2cdetect -y 3
# قياس IRQ latency
cyclictest -p 95 -t 4 -n --affinity=3 -D 60
```

#### الدرس المستفاد
**الـ** `of_irq_get_affinity()` تُتيح تحديد الـ CPU affinity للـ interrupts مباشرةً من DTS في الأنظمة الحرجة زمنياً. في الـ automotive والـ industrial، تخصيص CPU لمعالجة interrupt بعينه يُقلل الـ worst-case latency بشكل قابل للقياس.

---

### السيناريو الخامس: PCIe MSI لا يعمل على AM62x في Custom Board Bring-up

#### العنوان
**الـ PCIe NVMe SSD غير مرئي بعد boot على custom board مبني على TI AM62x**

#### السياق
فريق bring-up يعمل على **custom industrial board** مبني على **TI AM62x**. البورد يحتوي M.2 slot لـ NVMe SSD متصل بـ PCIe. عند boot، الـ `lspci` لا يُظهر الـ SSD وسجلات الـ kernel تُظهر PCIe errors.

#### المشكلة
الـ AM62x يستخدم **MSI (Message Signaled Interrupts)** للـ PCIe. الـ `of_msi_get_domain()` تعيد NULL لأن الـ `msi-parent` في DTS يشير إلى node لا يملك `irq_domain` مسجَّلاً بالـ `DOMAIN_BUS_PCI_MSI` token.

#### التحليل
الدوال المتعلقة بـ MSI في `of_irq.h`:

```c
/* تحصل على الـ MSI irq_domain المناسب */
extern struct irq_domain *of_msi_get_domain(struct device *dev,
                                             const struct device_node *np,
                                             enum irq_domain_bus_token token);

/* تُعيِّن MSI parent لـ device */
extern void of_msi_configure(struct device *dev, const struct device_node *np);

/* تترجم MSI ID إلى HW IRQ */
extern u32 of_msi_xlate(struct device *dev, struct device_node **msi_np, u32 id_in);

/* تحصل على MSI domain عبر ID mapping */
extern struct irq_domain *of_msi_map_get_device_domain(struct device *dev,
                                                         u32 id,
                                                         u32 bus_token);
```

مسار التنفيذ عند init الـ PCIe device:

```
pci_device_probe()
    └─> of_msi_configure(dev, dev_of_node)
            └─> يقرأ "msi-parent" من DTS
            └─> of_msi_get_domain(dev, msi_np, DOMAIN_BUS_PCI_MSI)
                    └─> irq_find_matching_host(msi_np, DOMAIN_BUS_PCI_MSI)
                    └─> إذا لم يجد domain → يعيد NULL
            └─> dev->msi.domain = NULL  ← هنا تكمن المشكلة
    └─> pci_enable_msi() تفشل لأن domain = NULL
    └─> fallback إلى INTx أو fail كلياً
```

فحص المشكلة:

```bash
# تحقق من الـ irq domains المسجَّلة
cat /sys/kernel/debug/irq/domains/
# ابحث عن domain بـ bus_token = PCI_MSI

# أو عبر dmesg
dmesg | grep -i "msi\|pcie\|irq domain"
# ابحث عن: "PCI: no MSI domain found"

# تحقق من DTS المحمَّل
cat /sys/firmware/devicetree/base/pcie@f102000/msi-parent
```

الـ DTS الخاطئ:

```dts
/* AM62x PCIe DTS الخاطئ */
gic500: interrupt-controller@1800000 {
    compatible = "arm,gic-v3";
    #interrupt-cells = <3>;
    interrupt-controller;
    /* لم يُعلَّم كـ msi-controller! */
};

pcie0_rc: pcie@f102000 {
    compatible = "ti,am62-pcie-host";
    reg = <0x00 0x0f102000 0x00 0x1000>,
          <0x00 0x0f100000 0x00 0x400>,
          <0x01 0x00000000 0x00 0x800000>;
    msi-map = <0x0 &gic500 0x0 0x10000>;
    /* msi-parent مفقود أو يشير لـ node خاطئ */
};
```

#### الحل

```dts
/* AM62x PCIe DTS الصحيح */
gic500: interrupt-controller@1800000 {
    compatible = "arm,gic-v3";
    #interrupt-cells = <3>;
    interrupt-controller;
    msi-controller;                    /* تفعيل MSI controller */
    #msi-cells = <1>;                  /* ضروري لـ msi-map */
    reg = <0x00 0x01800000 0x00 0x10000>,
          <0x00 0x01880000 0x00 0x80000>,
          <0x00 0x01900000 0x00 0x80000>;
};

/* ITS (Interrupt Translation Service) للـ GICv3 */
gic_its: msi-controller@1820000 {
    compatible = "arm,gic-v3-its";
    msi-controller;
    #msi-cells = <1>;
    reg = <0x00 0x01820000 0x00 0x10000>;
};

pcie0_rc: pcie@f102000 {
    compatible = "ti,am62-pcie-host";
    reg = <0x00 0x0f102000 0x00 0x1000>,
          <0x00 0x0f100000 0x00 0x400>,
          <0x01 0x00000000 0x00 0x800000>;

    /* ربط MSI IDs بالـ ITS */
    msi-map = <0x0 &gic_its 0x0 0x10000>;

    ranges = <0x01000000 0x0 0x60100000 0x0 0x60100000 0x0 0x00100000>,
             <0x02000000 0x0 0x60200000 0x0 0x60200000 0x0 0x07e00000>;

    status = "okay";
};
```

في driver الـ PCIe عند الحاجة لتشخيص:

```c
/* تشخيص يدوي في driver */
static int ti_am62_pcie_probe(struct platform_device *pdev)
{
    struct irq_domain *msi_domain;
    struct device_node *np = pdev->dev.of_node;

    /* تحقق من وجود MSI domain */
    msi_domain = of_msi_get_domain(&pdev->dev, np, DOMAIN_BUS_PCI_MSI);
    if (!msi_domain) {
        /* محاولة بالـ msi-map */
        msi_domain = of_msi_map_get_device_domain(&pdev->dev,
                                                   0,  /* device ID */
                                                   DOMAIN_BUS_PCI_MSI);
    }

    if (!msi_domain) {
        dev_err(&pdev->dev, "No MSI domain found, MSI disabled\n");
        /* الاستمرار بـ INTx أو الإيقاف */
    }
    /* ... */
}
```

التحقق بعد الإصلاح:

```bash
# يجب أن يظهر الـ NVMe
lspci
# 0000:01:00.0 Non-Volatile memory controller: Samsung ...

# تحقق من MSI
lspci -vvv | grep -A5 "MSI:"
# Capabilities: [50] MSI: Enable+ Count=1/32 ...

# اختبار الـ NVMe
nvme list
# /dev/nvme0  Samsung SSD 980 ...

dmesg | grep -i "msi\|nvme\|pcie"
# [ 2.345] pci 0000:01:00.0: MSI enabled (32 vectors)
```

#### الدرس المستفاد
الـ MSI في PCIe يعتمد على سلسلة كاملة: `msi-controller` في GIC/ITS + `msi-map` في PCIe controller + الـ `irq_domain` المسجَّل بالـ token الصحيح `DOMAIN_BUS_PCI_MSI`. غياب أي حلقة في هذه السلسلة يجعل `of_msi_get_domain()` تعيد NULL وتُصمت الـ device تماماً. عند bring-up جهاز جديد، ابدأ دائماً بالتحقق من وجود الـ MSI domain قبل البحث في الـ hardware.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأكاديمي الأول لتطور نواة Linux، وفيما يلي أهم المقالات المتعلقة بـ `of_irq` و `irq_domain` و Device Tree interrupts:

| المقال | الموضوع |
|--------|---------|
| [General device tree irq domain infrastructure](https://lwn.net/Articles/440524/) | البنية التحتية الأولى لـ `irq_domain` مع Device Tree — الأساس الذي بُني عليه `of_irq_init()` |
| [IRQ-domain.txt](https://lwn.net/Articles/487684/) | توثيق `irq_domain` library: التعيين بين `hwirq` وأرقام IRQ في Linux |
| [Documentation/IRQ-domain.txt (v3.19)](https://lwn.net/Articles/625547/) | النسخة المحدّثة من التوثيق الرسمي لـ `irq_domain` |
| [A new generic IRQ layer](https://lwn.net/Articles/184750/) | مقال تاريخي أساسي عن طبقة IRQ الجنيسة التي يعتمد عليها `of_irq` |
| [Device trees II: The harder parts](https://lwn.net/Articles/573409/) | تفاصيل متقدمة عن `interrupt-map`، `interrupts-extended`، و Interrupt Nexus |
| [irqdomain: Simplify interrupt handling](https://lwn.net/Articles/856815/) | تبسيط `irqdomain` في الإصدارات الحديثة |
| [irqchip: add basic infrastructure](https://lwn.net/Articles/521798/) | إضافة بنية تحتية لـ `irqchip` تستدعي `of_irq_init()` |
| [Moving interrupts to threads](https://lwn.net/Articles/302043/) | معالجة المقاطعات في خيوط منفصلة — ذات صلة بـ `request_threaded_irq` |
| [Device tree troubles](https://lwn.net/Articles/560523/) | تحديات ومشاكل Device Tree في الكيرنل الحديث |
| [Basic ARM devicetree support](https://lwn.net/Articles/440634/) | أول دعم لـ Device Tree على ARM مع ربط `of_irq` |

---

### التوثيق الرسمي للكيرنل (`Documentation/`)

**الـ** `Documentation/` في شجرة المصدر تحتوي على أهم التوثيقات التقنية:

```
Documentation/core-api/genericirq.rst        # الطبقة الجنيسة للمقاطعات
Documentation/core-api/irq/irq-domain.rst    # irq_domain library (الأحدث)
Documentation/devicetree/usage-model.rst     # نموذج استخدام Device Tree
Documentation/devicetree/kernel-api.rst      # واجهة برمجة الكيرنل مع Device Tree
Documentation/IRQ-domain.txt                 # النسخة القديمة (مرجع تاريخي)
```

الروابط المباشرة على kernel.org:
- [irq_domain Interrupt Number Mapping Library](https://docs.kernel.org/core-api/irq/irq-domain.html)
- [Linux generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html)
- [DeviceTree Kernel API](https://www.kernel.org/doc/html/latest/devicetree/kernel-api.html)
- [Linux and the Devicetree — Usage Model](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html)
- [IRQ-domain.txt (النسخة النصية القديمة)](https://www.kernel.org/doc/Documentation/IRQ-domain.txt)

---

### Commits رئيسية في تاريخ `of_irq`

أهم النقاط التاريخية في تطور `of_irq`:

| الحدث | الوصف |
|-------|-------|
| **kernel 2.6.x (PowerPC)** | النشأة الأولى لـ `of_irq` في `arch/powerpc` |
| **kernel 3.1** | نقل `irq_domain` إلى `kernel/irq/irqdomain.c` وتعميمه |
| **kernel 3.3** | إضافة `of_irq_init()` بشكل جنيس لجميع المعماريات |
| **kernel 3.7** | دعم `irq_domain` الهرمي (hierarchical irq domains) |
| **kernel 4.x+** | إضافة دعم MSI عبر `of_msi_get_domain()` و `of_msi_configure()` |

للبحث في تاريخ الـ commits مباشرة:

```bash
# تتبع تاريخ الملف
git log --oneline -- include/linux/of_irq.h

# تتبع of_irq_init
git log --oneline --all -S "of_irq_init" -- drivers/of/irq.c

# تتبع إضافة of_msi
git log --oneline --all -S "of_msi_get_domain" -- drivers/of/irq.c
```

---

### نقاشات Mailing List

- [Linux-Kernel Archive: irq domain updates for 3.19](https://lkml.iu.edu/hypermail/linux/kernel/1412.1/01076.html) — نقاش تحديثات `irq_domain` مع تكامل `of_irq`
- [Patchwork: use irqdomain to dynamically allocate IRQ for IOAPIC](https://patchwork.kernel.org/project/linux-acpi/patch/53787F09.9000200@linux.intel.com/) — نقاش توسيع `irqdomain` لـ IOAPIC

للبحث في أرشيف LKML:
```
https://lore.kernel.org/linux-devicetree/?q=of_irq
https://lore.kernel.org/linux-arm-kernel/?q=of_irq_init
```

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
**الـ** LDD3 هو المرجع الأساسي لكتابة drivers، الفصل المرتبط:
- **الفصل 10: Interrupt Handling** — يشرح `request_irq()`, `free_irq()`, IRQ flags
- متاح مجاناً: [https://lwn.net/Kernel/LDD2/ch09.lwn](https://lwn.net/Kernel/LDD2/ch09.lwn) (LDD2) والنسخة الثالثة على O'Reilly

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 7: Interrupts and Interrupt Handlers** — يغطي بنية الـ IRQ في الكيرنل
- يشرح `irq_desc`, `irq_chip`, وآلية dispatch المقاطعات

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14: Kernel Initialization** — يتناول تهيئة المقاطعات في بيئات embedded
- يشرح Device Tree وعلاقته بتهيئة interrupt controllers

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)
- **الفصل 4: Interrupts and Exceptions** — تغطية عميقة لـ IRQ subsystem
- [Chapter 4.6 Interrupt Handling على O'Reilly](https://www.oreilly.com/library/view/understanding-the-linux/0596005652/ch04s06.html)

---

### مصادر kernelnewbies.org و elinux.org

#### kernelnewbies.org
- [Exceptions and Interrupts Handling](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling) — شرح مفصل لآلية المقاطعات في الكيرنل
- [Internals of Interrupt Handling](https://kernelnewbies.org/KernelHacking-HOWTO/Overview_of_the_Kernel_Source_Code/Internals_of_Interrupt_Handling) — الجوانب الداخلية لمعالجة IRQ
- [Details of do_IRQ() function](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling/Details_of_do_IRQ()_function) — تفاصيل دالة `do_IRQ()`

#### elinux.org
- [Device Tree Reference](https://elinux.org/Device_Tree_Reference) — مرجع شامل لخصائص Device Tree بما فيها `interrupts` و `interrupt-map`
- [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) — توضيح نقاط غامضة في Device Tree
- [Linux Drivers Device Tree Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) — دليل عملي لكتابة drivers مع Device Tree
- [Device Tree Information](https://www.elinux.org/Device_Tree_Information) — مدخل شامل لـ Device Tree
- [Device Trees](https://elinux.org/Device_Trees) — صفحة فهرسة لكل موارد Device Tree

---

### مصادر إضافية

- [How the Linux kernel handles interrupts — opensource.com](https://opensource.com/article/20/10/linux-kernel-interrupts) — شرح مبسط وعملي لآلية IRQ
- [IRQ & Device Tree على linux.org forums](https://www.linux.org/threads/irq-device-tree-tutorial-course-recommendation.43104/) — نقاش مجتمعي عن تعلم IRQ مع Device Tree

---

### مصطلحات البحث

للعثور على معلومات إضافية، استخدم هذه المصطلحات:

```
of_irq_init Linux kernel
irq_domain device tree interrupt controller
of_irq_parse_one interrupt specifier
irq_create_of_mapping hwirq
of_msi_configure MSI device tree
interrupt-map device tree nexus
irq_of_parse_and_map ARM embedded
of_irq_get_byname interrupt-names
irq_domain_add_linear of_irq
```
## Phase 8: Writing simple module

### الدالة المختارة: `of_irq_get`

**الـ** `of_irq_get()` دالة مُصدَّرة من `CONFIG_OF_IRQ` تحوّل فهرس interrupt في device tree إلى Linux virtual IRQ number. تُستدعى في كل مرة يحاول فيها driver الحصول على interrupt من عقدة DT، ما يجعلها نقطة رصد مثالية لمعرفة أي device nodes تطلب interrupts وبأي ترتيب أثناء boot أو probe.

---

### الـ kprobe Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_irq_get.c
 * Hooks of_irq_get() to log device-node name and IRQ index/result.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit       */
#include <linux/kernel.h>       /* pr_info()                               */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe()        */
#include <linux/of.h>           /* struct device_node, of_node_full_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on of_irq_get() – log DT node name and IRQ index");

/* -----------------------------------------------------------------------
 * pre_handler: يُستدعى قبل تنفيذ of_irq_get() مباشرةً.
 * الـ regs تحتوي على قيم السجلات في لحظة الاستدعاء؛
 * الـ argument الأول (dev) في x86-64 يكون في rdi، والثاني (index) في rsi.
 * ----------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: rdi = first arg (struct device_node *dev)
     *            rsi = second arg (int index)
     * On ARM64:  x0  = dev, x1 = index  (regs->regs[0], regs->regs[1])
     *
     * We use a portable helper: kprobe_get_argument() is not standard,
     * so we read directly from the ABI-defined registers.
     */
#if defined(CONFIG_X86_64)
    struct device_node *dev = (struct device_node *)regs->di;
    int index               = (int)regs->si;
#elif defined(CONFIG_ARM64)
    struct device_node *dev = (struct device_node *)regs->regs[0];
    int index               = (int)regs->regs[1];
#else
    /* Fallback – avoids compile error on other arches; data will be zero */
    struct device_node *dev = NULL;
    int index               = -1;
#endif

    /* of_node_full_name returns "/" for root, never NULL */
    pr_info("of_irq_get called: node=\"%s\" index=%d\n",
            dev ? of_node_full_name(dev) : "<null>",
            index);

    return 0; /* 0 = continue normal execution */
}

/* -----------------------------------------------------------------------
 * post_handler: يُستدعى بعد انتهاء of_irq_get() وقبل عودة النتيجة.
 * نطبع قيمة المُرجَعة (virtual IRQ number أو خطأ سالب).
 * ----------------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86-64 the return value sits in rax after the function returns.
     * On ARM64 it is in x0.
     */
#if defined(CONFIG_X86_64)
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = -1;
#endif

    pr_info("of_irq_get returned: virq=%ld%s\n",
            ret,
            ret <= 0 ? " (error/no-irq)" : " (ok)");
}

/* -----------------------------------------------------------------------
 * struct kprobe: يربط handler بالرمز المستهدف بالاسم.
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "of_irq_get",  /* kernel resolves address at register time */
    .pre_handler = handler_pre,
    .post_handler = handler_post,
};

/* -----------------------------------------------------------------------
 * module_init: نسجّل الـ kprobe؛ إذا فشل نعيد الخطأ ولا يُحمَّل الـ module.
 * ----------------------------------------------------------------------- */
static int __init kp_of_irq_init(void)
{
    int ret = register_kprobe(&kp);

    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on of_irq_get at %p\n", kp.addr);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit: نزيل الـ kprobe حتماً قبل تفريغ الـ module من الذاكرة،
 * وإلا يبقى pointer إلى كود محذوف فيسبّب kernel panic.
 * ----------------------------------------------------------------------- */
static void __exit kp_of_irq_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe on of_irq_get removed\n");
}

module_init(kp_of_irq_init);
module_exit(kp_of_irq_exit);
```

---

### شرح كل قسم

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | ماكروهات `MODULE_*` و `module_init` / `module_exit` |
| `<linux/kernel.h>` | `pr_info()` و `pr_err()` |
| `<linux/kprobes.h>` | `struct kprobe`، `register_kprobe`، `unregister_kprobe`، `pt_regs` |
| `<linux/of.h>` | `struct device_node`، `of_node_full_name()` |

**الـ** `<linux/kprobes.h>` هو القلب — يوفر الـ API الكاملة لزرع breakpoints برمجية داخل kernel دون تعديل source code.

---

#### الـ `pre_handler`

يُستدعى **قبل** تنفيذ أول تعليمة من `of_irq_get()`. يقرأ وسيطي الدالة مباشرةً من سجلات المعالج (`rdi`/`rsi` على x86-64، `x0`/`x1` على ARM64) لأن ABI يضمن وجودها هناك. **الـ** `of_node_full_name()` تُرجع المسار الكامل للعقدة مثل `/soc/interrupt-controller@1c00000` مما يسهّل تشخيص أي device يطلب interrupt.

---

#### الـ `post_handler`

يُستدعى **بعد** انتهاء `of_irq_get()` مباشرةً. قيمة المُرجَعة (virtual IRQ number الموجب، أو خطأ سالب) تكون في `rax`/`x0` وفق ABI. هذا يتيح ربط كل طلب بنتيجته في الـ dmesg.

---

#### الـ `struct kprobe`

الحقل `symbol_name` يُعفينا من حساب العنوان يدوياً؛ الـ kernel يحلّه وقت `register_kprobe()` عبر `kallsyms`. بعد التسجيل الناجح يُملأ `kp.addr` بالعنوان الفعلي الذي نطبعه للتحقق.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` تزرع int3 (x86) أو BRK (ARM64) على العنوان المستهدف. **لازم** `unregister_kprobe` في `module_exit` لاستعادة الـ byte الأصلي؛ تركه يُبقي pointer إلى كود المودول المُفرَّغ ويسبّب **kernel panic** فور استدعاء `of_irq_get` بعد `rmmod`.

---

### Makefile والاختبار

```makefile
# Makefile
obj-m += kprobe_of_irq_get.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# بناء وتحميل
make
sudo insmod kprobe_of_irq_get.ko

# مشاهدة النتائج (تظهر أثناء probe لأي device يستخدم of_irq_get)
sudo dmesg | grep "of_irq_get"

# مثال على مخرجات متوقعة:
# [   12.345678] kprobe planted on of_irq_get at ffffffffc0ab1234
# [   12.389012] of_irq_get called: node="/soc/serial@11002000" index=0
# [   12.389015] of_irq_get returned: virq=47 (ok)
# [   12.391200] of_irq_get called: node="/soc/i2c@1100f000" index=0
# [   12.391202] of_irq_get returned: virq=48 (ok)

# إزالة الـ module
sudo rmmod kprobe_of_irq_get
```

**الـ** kprobe لن تطبع شيئاً حتى يُستدعى `of_irq_get()` فعلياً؛ لرؤية النتائج حوّل bind/unbind لأي device يعتمد على DT interrupts:

```bash
# مثال: force re-probe لـ device محدد
echo "1100f000.i2c" | sudo tee /sys/bus/platform/drivers/i2c-mt65xx/unbind
echo "1100f000.i2c" | sudo tee /sys/bus/platform/drivers/i2c-mt65xx/bind
```
