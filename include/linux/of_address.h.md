## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem؟

الـ `of_address.h` ينتمي إلى **Open Firmware / Device Tree (OF)** subsystem في Linux kernel. هذا الـ subsystem مسؤول عن قراءة وصف الأجهزة من ملف يُسمى **Device Tree Blob (DTB)** — وهو ملف ثنائي يصف للـ kernel أي الأجهزة موجودة على اللوحة وأين تقع في الذاكرة.

---

### القصة بالتشبيه

تخيّل أنك وصلت إلى مدينة جديدة تماماً ولا تعرف شيئاً عن تخطيطها. معك خريطة (هي الـ Device Tree) تقول لك:

> "المصنع A يقع في الحي رقم 0x02000000، ويحتاج مساحة 4KB."
> "جسر PCI يربط الحي الرئيسي بالحي الجانبي، والعناوين في الحي الجانبي تُترجَم هكذا..."

مشكلتك الآن: **الخريطة مكتوبة بلغة الحي الجانبي** (عناوين الجهاز)، لكنك تحتاج العناوين بلغة نظام التشغيل (عناوين CPU). هنا يأتي دور `of_address.h` — إنه **قاموس الترجمة**.

---

### المشكلة التي يحلّها

في المعالجات المدمجة (ARM, RISC-V, PowerPC)، لا يوجد BIOS يخبر الـ kernel أين توجد الأجهزة. بدلاً من ذلك، يصف المصنّع اللوحة في ملف **Device Tree Source (DTS)**، يُحوَّل إلى **DTB** ويُمرَّر للـ kernel عند الإقلاع.

كل جهاز في الـ DTB له خاصية `reg` تحتوي على **عنوانه وحجمه**. لكن:

- عناوين الأجهزة ليست دائماً عناوين CPU مباشرة.
- الـ PCI bus لها نظام عناوين خاص بها (32-bit أو 64-bit أو I/O).
- الـ DMA قد يرى الذاكرة من نافذة مختلفة عن CPU.
- الجسور (bridges) تُضيف offset أو تحوّل النطاقات.

**الحل:** خاصية `ranges` في الـ DTB تصف كيفية ترجمة العناوين من مستوى إلى آخر. الـ `of_address.h` يوفر API لقراءة هذه الخصائص وتطبيق سلسلة الترجمة كاملة.

---

### ماذا يفعل هذا الملف تحديداً؟

الـ `include/linux/of_address.h` هو **الـ header العام** لوحدة ترجمة العناوين في الـ OF subsystem. يُعرّف:

#### 1. Structs أساسية

| الـ Struct | الدور |
|---|---|
| `struct of_pci_range_parser` | **آلة قراءة** خاصية `ranges` أو `dma-ranges` من الـ DTB خطوة بخطوة |
| `struct of_pci_range` | يمثّل **نطاق عنوان واحد**: عنوان الجهاز، عنوان CPU، الحجم، الأعلام |

```c
struct of_pci_range {
    union {
        u64 pci_addr;    /* عنوان الجهاز (PCI/bus side) */
        u64 bus_addr;
    };
    u64 cpu_addr;        /* العنوان المقابل من جانب CPU */
    u64 parent_bus_addr; /* عنوان الـ parent bus */
    u64 size;            /* حجم النطاق */
    u32 flags;           /* نوع المورد: MEM, IO, ... */
};
```

#### 2. دوال الترجمة الرئيسية

| الدالة | ماذا تفعل |
|---|---|
| `of_translate_address()` | تُحوّل عنوان جهاز من الـ DTB إلى عنوان CPU فعلي |
| `of_translate_dma_address()` | نفس الشيء لكن من منظور الـ DMA |
| `of_address_to_resource()` | تحوّل خاصية `reg` مباشرة إلى `struct resource` جاهزة للاستخدام |
| `of_iomap()` | تُترجم العنوان **وتعمل** `ioremap` تلقائياً — يمكن الكتابة/القراءة مباشرة |
| `of_io_request_and_map()` | مثل `of_iomap` لكن تحجز المورد أيضاً (`request_mem_region`) |

#### 3. دوال قراءة الـ PCI ranges

```c
/* يُهيّئ parser ليقرأ خاصية "ranges" من node */
of_pci_range_parser_init(parser, node);

/* يقرأ نطاقاً واحداً في كل مرة */
for_each_of_range(parser, range) {
    /* range->cpu_addr هو عنوان CPU */
    /* range->pci_addr هو عنوان PCI */
    /* range->size هو الحجم */
}
```

#### 4. Compile-time Stubs

عندما `CONFIG_OF_ADDRESS` غير مفعّل (على بعض المعماريات)، كل الدوال تُعيد أخطاء آمنة (`-ENOSYS`, `NULL`, `OF_BAD_ADDR`) دون crash.

---

### مثال حقيقي: كيف يستخدمه driver؟

```c
/* Driver يريد الوصول إلى سجلات جهازه */
static int my_driver_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    void __iomem *base;

    /* of_iomap تقرأ خاصية reg[0] من DTB وتُترجم العنوان وتعمل ioremap */
    base = of_iomap(np, 0);
    if (!base)
        return -ENOMEM;

    /* الآن يمكن القراءة/الكتابة مباشرة */
    writel(0x1, base + REG_CTRL);
    return 0;
}
```

---

### سلسلة الترجمة الكاملة (Address Translation Chain)

```
DTB Node (reg = <0x1000 0x100>)
         |
         v
  of_get_address()         ← يجلب الـ raw bytes من خاصية reg
         |
         v
  of_translate_address()   ← يمشي شجرة الـ DTB للأعلى
         |                    ويطبّق كل "ranges" على الطريق
         v
  عنوان CPU الفعلي (مثلاً 0xFE001000)
         |
         v
  ioremap()                ← يربط العنوان الفيزيائي بالـ virtual address
         |
         v
  void __iomem *base       ← جاهز للاستخدام من الـ driver
```

---

### ملفات مرتبطة يجب معرفتها

#### الملفات الأساسية (Core)

| الملف | الدور |
|---|---|
| `drivers/of/address.c` | التنفيذ الكامل لكل دوال `of_address.h` |
| `drivers/of/base.c` | قراءة الـ DTB، البحث عن nodes وproperties |
| `drivers/of/fdt.c` | تحليل ملف DTB الثنائي عند الإقلاع |
| `drivers/of/fdt_address.c` | ترجمة العناوين في مرحلة الإقلاع المبكرة (early boot) |

#### Headers المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/of.h` | تعريف `struct device_node`، `struct property` — الأساس |
| `include/linux/of_address.h` | **هذا الملف** — API ترجمة العناوين |
| `include/linux/of_device.h` | ربط الـ DTB node بالـ platform_device |
| `include/linux/of_platform.h` | إنشاء devices تلقائياً من الـ DTB |
| `include/linux/of_dma.h` | ترجمة DMA ranges |
| `include/linux/of_irq.h` | ترجمة مقاطعات الـ DTB |
| `include/linux/ioport.h` | تعريف `struct resource` التي تُملأ بالعناوين |

#### مثال DTB يوضّح المشكلة

```dts
/* في ملف .dts */
soc {
    ranges = <0x0 0xfe000000 0x1000000>; /* bus 0x0 → CPU 0xfe000000 */

    uart0: serial@1000 {
        reg = <0x1000 0x100>; /* عنوان bus = 0x1000 */
        /* of_translate_address سيحوّله → CPU: 0xfe001000 */
    };
};
```

---

### ملخص الهدف

**الـ `of_address.h` هو واجهة ترجمة العناوين في عالم Device Tree** — يحوّل العناوين المجرّدة المكتوبة في الـ DTB إلى عناوين CPU حقيقية قابلة للاستخدام، مع دعم كامل للـ PCI، DMA، والجسور متعددة المستويات.
## Phase 2: شرح الـ OF Address Translation Framework

---

### المشكلة التي يحلّها هذا الـ Subsystem

في أنظمة embedded — خصوصاً ARM SoCs — كل peripheral له عنوان في فضاء الذاكرة (memory-mapped I/O). لكن السؤال الجوهري: **من أين يعرف الـ kernel هذا العنوان؟**

في x86 يوجد BIOS/UEFI و PCI enumeration — الجهاز يُعرِّف نفسه. لكن في ARM embedded boards:

- الـ hardware لا يُعرِّف نفسه تلقائياً.
- العناوين ثابتة hardcoded في الـ SoC.
- قد توجد bus bridges متعددة (AHB → APB → peripheral) كل منها يُضيف offset.
- الـ DMA قد يرى عناوين مختلفة تماماً عمّا يراه الـ CPU.

المشكلة إذن ثلاثية الأبعاد:

| البُعد | التحدي |
|--------|--------|
| **مصدر العنوان** | من أين نقرأ عنوان الـ peripheral؟ |
| **ترجمة العنوان** | كيف نحوّل عنوان الـ bus إلى عنوان يفهمه الـ CPU؟ |
| **توحيد التمثيل** | كيف نعطي الـ driver بنية موحّدة بصرف النظر عن نوع الـ bus؟ |

---

### الحل: OF Address Translation

**الـ OF (Open Firmware / Device Tree)** هو الآلية القياسية في Linux لوصف الـ hardware statically. الـ DTB (Device Tree Blob) يحتوي على شجرة من الـ nodes، كل node يصف peripheral واحداً وعناوينه.

الـ `of_address` subsystem يتولى:

1. **قراءة** خاصية `reg` من الـ Device Tree node.
2. **ترجمة** العنوان عبر سلسلة الـ buses (باستخدام خاصية `ranges`).
3. **تحويل** الناتج إلى `struct resource` الذي يفهمه باقي الـ kernel.
4. **توفير** `ioremap` مباشر عبر `of_iomap()`.

---

### التشابه الواقعي: نظام المواصلات

تخيّل شبكة نقل عام من ثلاث مستويات: مترو → حافلة → شارع محلي.

```
المسافر يعرف وجهته كـ: "محطة 5 في الخط الأصفر"
الـ GPS يحوّلها إلى إحداثيات حقيقية: (30.0444° N, 31.2357° E)
```

| عنصر المواصلات | المقابل في الـ kernel |
|----------------|----------------------|
| رقم المحطة في الخط | عنوان الـ peripheral في الـ `reg` property (bus address) |
| الخط نفسه (أصفر/أحمر) | الـ bus node في Device Tree |
| جدول التحويل بين الخطوط | خاصية `ranges` في الـ DT |
| الإحداثيات الجغرافية الحقيقية | الـ CPU physical address بعد الترجمة |
| خريطة المدينة الكاملة | `struct resource` النهائي |
| السائق الذي يصل وجهته | الـ driver الذي يستدعي `of_iomap()` |

الـ `ranges` property تماماً كجدول تحويل خطوط المواصلات: تقول "عنوان 0x1000 في الـ child bus يقابل عنوان 0x40001000 في الـ parent bus". وإذا كان هناك bridge آخر فوق، يتكرر التحويل مرة أخرى.

---

### المعمارية الكاملة

```
┌─────────────────────────────────────────────────────────────────┐
│                        Device Tree Blob                         │
│                                                                 │
│  / {                                                            │
│    #address-cells = <1>;                                        │
│    #size-cells = <1>;                                           │
│                                                                 │
│    soc {                                                        │
│      compatible = "simple-bus";                                 │
│      #address-cells = <1>;                                      │
│      #size-cells = <1>;                                         │
│      ranges = <0x0  0x40000000  0x10000000>;                    │
│      /*  child_addr  parent_addr   size  */                     │
│                                                                 │
│      uart0: serial@1000 {                                       │
│        reg = <0x1000  0x100>;                                   │
│        /*    bus_addr  size  */                                  │
│      };                                                         │
│    };                                                           │
│  };                                                             │
└────────────────────────┬────────────────────────────────────────┘
                         │  of_address_to_resource() / of_iomap()
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   OF Address Subsystem                          │
│                                                                 │
│  ┌─────────────────┐    ┌──────────────────┐                   │
│  │ of_get_address()│    │of_translate_addr()│                   │
│  │                 │───▶│                   │                   │
│  │ reads `reg`     │    │ walks `ranges`    │                   │
│  │ property as raw │    │ chain upward to   │                   │
│  │ __be32 array    │    │ root node         │                   │
│  └─────────────────┘    └────────┬─────────┘                   │
│                                  │                              │
│                    CPU phys addr: 0x40001000                    │
│                                  │                              │
│  ┌───────────────────────────────▼──────────────────────────┐  │
│  │         of_address_to_resource()                         │  │
│  │  fills struct resource { .start=0x40001000, .end=... }   │  │
│  └───────────────────────────────┬──────────────────────────┘  │
└──────────────────────────────────│──────────────────────────────┘
                                   │
           ┌───────────────────────┴───────────────────┐
           │                                           │
           ▼                                           ▼
  ┌─────────────────┐                      ┌─────────────────────┐
  │   of_iomap()    │                      │  Driver uses        │
  │                 │                      │  struct resource    │
  │ ioremap(phys,   │                      │  via               │
  │   size)         │                      │  platform_get_      │
  │                 │                      │  resource()         │
  │ returns void    │                      │                     │
  │ __iomem *       │                      └─────────────────────┘
  └─────────────────┘
```

---

### الـ Core Abstraction: Address Space Hierarchy

الفكرة المركزية في هذا الـ subsystem هي أن **كل bus node في الـ Device Tree يمثّل فضاء عناوين مستقل**، وخاصية `ranges` هي **دالة تحويل (mapping function)** بين فضاء العناوين الابن وفضاء العناوين الأب.

#### بنية الـ `ranges` property

```
ranges = <child-addr  parent-addr  size>
         [#child-addr-cells words] [#parent-addr-cells words] [#size-cells words]
```

مثال حقيقي من SoC يحتوي على AXI bridge:

```dts
/ {
    #address-cells = <2>;   /* 64-bit addressing at root */
    #size-cells = <2>;

    axi_bus: axi@0 {
        #address-cells = <1>;
        #size-cells = <1>;
        ranges = <0x00000000  0x00000000 0x40000000  0x40000000>;
        /*        child(32b)   parent_hi  parent_lo   size       */

        apb_bus: apb@20000000 {
            #address-cells = <1>;
            #size-cells = <1>;
            ranges = <0x0  0x20000000  0x10000000>;

            spi0: spi@3000 {
                reg = <0x3000  0x100>;
            };
        };
    };
};
```

الـ kernel يحوّل `spi0` بخطوتين:

```
spi0 في APB:   0x00003000
+ APB ranges:  0x00003000 + 0x20000000 = 0x20003000 (في AXI space)
+ AXI ranges:  0x20003000 + 0x00000000 = 0x20003000 (في CPU physical space)
```

---

### البنى الأساسية

#### `struct of_pci_range_parser`

```c
struct of_pci_range_parser {
    struct device_node *node;   /* الـ DT node الذي يحتوي على ranges */
    const struct of_bus *bus;   /* ops خاصة بنوع الـ bus (PCI/generic) */
    const __be32 *range;        /* مؤشر للـ entry الحالي في ranges array */
    const __be32 *end;          /* نهاية الـ array */
    int na;                     /* #address-cells للـ child */
    int ns;                     /* #size-cells */
    int pna;                    /* #address-cells للـ parent */
    bool dma;                   /* هل نقرأ dma-ranges بدل ranges؟ */
};
```

هذا الـ struct هو **حالة iterator** — يسمح بالمشي عبر entries الـ `ranges` property واحداً تلو الآخر دون تخصيص ذاكرة إضافية.

```
ranges array في الذاكرة:
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│ ca0  │ ca1  │ pa0  │ pa1  │ s0   │ ca0  │ ca1  │ pa0  │ ...  │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
 ▲                                   ▲
 range (current)                     range + (na+pna+ns) = next entry
```

#### `struct of_pci_range`

```c
struct of_pci_range {
    union {
        u64 pci_addr;      /* العنوان في فضاء PCI */
        u64 bus_addr;      /* أو العنوان في فضاء أي bus آخر */
    };
    u64 cpu_addr;          /* العنوان بعد الترجمة لفضاء الـ CPU */
    u64 parent_bus_addr;   /* العنوان في الـ parent bus */
    u64 size;              /* حجم النطاق */
    u32 flags;             /* IORESOURCE_MEM / IORESOURCE_IO / ... */
};
```

هذه البنية هي **الناتج المُوحَّد** لعملية parsing — بغض النظر عن نوع الـ bus.

#### `struct resource` (من `ioport.h`)

```c
struct resource {
    resource_size_t start;   /* بداية النطاق في فضاء الـ CPU */
    resource_size_t end;     /* نهايته */
    const char *name;
    unsigned long flags;     /* IORESOURCE_MEM, IORESOURCE_IO, ... */
    unsigned long desc;
    struct resource *parent, *sibling, *child;  /* شجرة الـ resources */
};
```

**الـ resource subsystem** (مستقل عن OF) هو الـ registry المركزي للـ kernel لتتبع من يملك أي نطاق عناوين، ويمنع التعارض بين الـ drivers.

---

### علاقة البنى ببعضها

```
Device Tree Property "reg":
  raw __be32 array
         │
         │  of_get_address()
         ▼
  raw address + size (still in bus space)
         │
         │  of_translate_address()   ← يمشي شجرة الـ nodes للأعلى
         ▼                             مطبّقاً كل ranges في الطريق
  u64 cpu_addr  (physical address)
         │
         │  of_address_to_resource()
         ▼
  struct resource {
    .start = cpu_addr,
    .end   = cpu_addr + size - 1,
    .flags = IORESOURCE_MEM,
  }
         │
         ├──▶  of_iomap()  →  ioremap()  →  void __iomem*
         │
         └──▶  platform device core  →  driver استخدام
```

---

### الـ DMA Address Space Problem

هذا أحد أدق جوانب الـ subsystem. الـ DMA controller قد يرى عناوين مختلفة عن الـ CPU — خصوصاً عند وجود IOMMU أو address remapping في الـ interconnect.

```
CPU يرى RAM في:       0x80000000
DMA controller يرى:   0x00000000  (بعد bridge يطرح 0x80000000)
```

لذلك الـ DT يحتوي على `dma-ranges` منفصلة:

```dts
dma-ranges = <0x00000000  0x80000000  0x40000000>;
/*            DMA address  CPU address   size      */
```

الـ subsystem يوفّر:

```c
/* ترجمة عنوان من فضاء الـ DMA إلى فضاء الـ CPU */
u64 of_translate_dma_address(struct device_node *dev, const __be32 *in_addr);

/* استخراج نطاق DMA كامل مع start و length */
const __be32 *of_translate_dma_region(struct device_node *dev,
                                       const __be32 *addr,
                                       phys_addr_t *start,
                                       size_t *length);
```

---

### ما يملكه الـ Subsystem مقابل ما يُفوّضه

| المسؤولية | المالك |
|-----------|--------|
| قراءة `reg` و `ranges` من الـ DTB | OF Address Subsystem |
| المشي عبر bus hierarchy للترجمة | OF Address Subsystem |
| تحويل الناتج إلى `struct resource` | OF Address Subsystem |
| استدعاء `ioremap()` | OF Address Subsystem (عبر `of_iomap`) |
| تسجيل الـ resource في شجرة kernel resources | Resource Subsystem |
| تحديد نوع الـ bus وعدد الـ cells | الـ DT نفسه (`#address-cells`) |
| PCI-specific address encoding | OF PCI Subsystem (يستخدم نفس البنى) |
| IOMMU address translation | IOMMU Subsystem (طبقة منفصلة) |
| فحص `dma-coherent` property | OF Address (`of_dma_is_coherent`) |

---

### الـ API الكاملة: من البسيط للمتقدم

#### المستوى الأول: الـ driver العادي

```c
/* الأبسط: اقرأ الـ reg[index] وارجع pointer مباشر للـ MMIO */
void __iomem *base = of_iomap(node, 0);
if (!base)
    return -ENOMEM;

/* أو مع حجز الـ resource (request_mem_region داخلياً) */
void __iomem *base = of_io_request_and_map(node, 0, "my-driver");
```

#### المستوى الثاني: تحويل عنوان + struct resource

```c
struct resource res;
int ret = of_address_to_resource(node, 0, &res);
if (ret)
    return ret;
/* الآن res.start هو الـ CPU physical address */
/* res.flags يحتوي IORESOURCE_MEM */
```

#### المستوى الثالث: قراءة raw مع ترجمة يدوية

```c
u64 size;
unsigned int flags;
const __be32 *addr = of_get_address(node, 0, &size, &flags);
if (!addr)
    return -EINVAL;

u64 cpu_addr = of_translate_address(node, addr);
if (cpu_addr == OF_BAD_ADDR)
    return -EINVAL;
```

#### المستوى الرابع: iterator للـ ranges (PCI/complex buses)

```c
struct of_pci_range_parser parser;
struct of_pci_range range;

/* تهيئة الـ parser على الـ ranges property */
of_pci_range_parser_init(&parser, node);

/* المشي عبر كل نطاق */
for_each_of_range(&parser, &range) {
    struct resource res;
    of_pci_range_to_resource(&range, node, &res);
    /* res.start = CPU address للنطاق */
    /* range.bus_addr = العنوان في فضاء الـ bus */
    /* range.size = الحجم */
}
```

---

### حساب عدد الـ Entries: `of_range_count()`

```c
static inline int of_range_count(const struct of_range_parser *parser)
{
    if (!parser || !parser->node || !parser->range ||
        parser->range == parser->end)
        return 0;
    /* كل entry حجمه = na + pna + ns كلمات __be32 */
    return (parser->end - parser->range) / (parser->na + parser->pna + parser->ns);
}
```

هذا الحساب بسيط لكن دقيق: بما أن حجم كل entry يختلف بحسب `#address-cells` و `#size-cells` الخاصة بكل bus، لا يمكن استخدام sizeof ثابت — لذا يُحسب ديناميكياً من الـ parser state.

> **تحذير**: استدعاء `of_range_count()` داخل أو بعد الـ `for_each_of_range()` يعطي نتائج غير دقيقة لأن `parser->range` يتقدّم مع كل iteration.

---

### الـ CONFIG Guards

```c
#ifdef CONFIG_OF_ADDRESS
    /* الـ API الكامل متاح */
    extern u64 of_translate_address(...);
    extern int of_address_to_resource(...);
    ...
#else
    /* stub functions ترجع -ENOSYS أو OF_BAD_ADDR */
    /* تسمح بالبناء على أنظمة بدون Device Tree */
#endif
```

هذا النمط شائع في الـ kernel: الـ subsystem يوفّر **stubs** كاملة عند تعطيل الـ feature، مما يُبقي الـ drivers portable دون `#ifdef` في كودها.

---

### مثال واقعي: UART Driver على ARM SoC

```dts
/* في الـ Device Tree */
soc {
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0x0  0xFE000000  0x01000000>;

    uart0: serial@7e215040 {
        compatible = "brcm,bcm2835-aux-uart";
        reg = <0x7e215040 0x40>;
        /* bus addr: 0x7e215040, translated: 0xFE215040 */
    };
};
```

```c
/* في الـ driver */
static int bcm2835_aux_serial_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;

    /* of_iomap تقرأ reg[0]، تترجم العنوان، تستدعي ioremap */
    void __iomem *regs = of_iomap(np, 0);
    /* regs يشير إلى 0xFE215040 بعد ioremap */

    /* أو بشكل أكثر وضوحاً */
    struct resource res;
    of_address_to_resource(np, 0, &res);
    /* res.start = 0xFE215040 */
    /* res.end   = 0xFE21507F */
    /* res.flags = IORESOURCE_MEM */
}
```

---

### خلاصة: لماذا هذا الـ Subsystem ضروري؟

بدون `of_address`:

- كل driver يحتاج لقراءة raw `__be32` values يدوياً.
- كل driver يحتاج لتنفيذ منطق ترجمة العناوين عبر الـ bus hierarchy.
- لا توحيد بين PCI، AHB، APB، وباقي الـ buses.
- الـ DMA address space منفصل تماماً بدون معالجة موحّدة.

مع `of_address`:

- الـ driver يستدعي `of_iomap(node, 0)` ويعمل.
- الترجمة الكاملة عبر أي عدد من الـ bridges تحدث تلقائياً.
- الناتج دائماً `struct resource` موحّد أو `void __iomem*` جاهز للاستخدام.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags، الـ Enums، وخيارات الـ Config — Cheatsheet

#### أعلام الـ `IORESOURCE_*` (من `ioport.h`) — تُستخدم في `of_pci_range.flags`

| Flag | القيمة | المعنى |
|---|---|---|
| `IORESOURCE_IO` | `0x100` | منفذ I/O (PCI/ISA) |
| `IORESOURCE_MEM` | `0x200` | ذاكرة MMIO |
| `IORESOURCE_REG` | `0x300` | إزاحة register |
| `IORESOURCE_IRQ` | `0x400` | خط مقاطعة |
| `IORESOURCE_DMA` | `0x800` | قناة DMA |
| `IORESOURCE_BUS` | `0x1000` | مورد bus |
| `IORESOURCE_PREFETCH` | `0x2000` | قابل للـ prefetch (PCI MEM) |
| `IORESOURCE_MEM_64` | `0x100000` | عنوان 64-bit |
| `IORESOURCE_WINDOW` | `0x200000` | bridge window |
| `IORESOURCE_UNSET` | `0x20000000` | لم يُعيَّن عنوان بعد |
| `IORESOURCE_BUSY` | `0x80000000` | محجوز من driver |

#### أعلام `device_node._flags` (من `of.h`)

| Flag / Bit | المعنى |
|---|---|
| `OF_DYNAMIC` (bit 1) | الـ node مُخصَّص بـ kmalloc |
| `OF_DETACHED` (bit 2) | منفصل عن شجرة الأجهزة |
| `OF_POPULATED` (bit 3) | جهاز مُنشأ بالفعل |
| `OF_POPULATED_BUS` (bit 4) | platform bus مُنشأ للأبناء |
| `OF_OVERLAY` (bit 5) | مُخصَّص لـ overlay |

#### خيارات الـ Kconfig

| Option | الأثر على `of_address.h` |
|---|---|
| `CONFIG_OF_ADDRESS` | يُفعِّل جميع دوال الترجمة الحقيقية (`of_translate_address`, `of_iomap`, ...) |
| `CONFIG_OF` | يُفعِّل `of_address_to_resource` و`of_iomap` |

---

### 1. الـ Structs الأساسية

#### 1.1 `struct of_pci_range_parser`

**الغرض:** يحتفظ بحالة المُكرِّر (iterator state) أثناء المسح التسلسلي لخاصية `ranges` أو `dma-ranges` في شجرة الأجهزة (Device Tree).

```c
struct of_pci_range_parser {
    struct device_node *node;   /* العقدة التي تحوي خاصية ranges */
    const struct of_bus *bus;   /* عمليات البوص (translate/count_cells) */
    const __be32 *range;        /* مؤشر للإدخال الحالي في المصفوفة */
    const __be32 *end;          /* نهاية المصفوفة (حدّ التوقف) */
    int na;                     /* #address-cells للجهاز الفرعي */
    int ns;                     /* #size-cells للجهاز الفرعي */
    int pna;                    /* #address-cells للعقدة الأم */
    bool dma;                   /* true = يقرأ dma-ranges بدل ranges */
};
#define of_range_parser of_pci_range_parser  /* alias عام */
```

| الحقل | النوع | الشرح |
|---|---|---|
| `node` | `struct device_node *` | العقدة المصدر |
| `bus` | `const struct of_bus *` | عمليات البوص الداخلية (static, غير مُصدَّرة) |
| `range` | `const __be32 *` | يتقدم خلية بخلية عند كل `parser_one()` |
| `end` | `const __be32 *` | يُحسَب من حجم الخاصية |
| `na`, `ns`, `pna` | `int` | أحجام الحقول بالخلايا (كل خلية = 4 bytes) |
| `dma` | `bool` | يُحدد أي خاصية تُقرأ |

---

#### 1.2 `struct of_pci_range`

**الغرض:** يمثّل إدخالاً واحداً مُفكَّكاً (decoded) من خاصية `ranges`، يصف نافذة عنوان مع التعيين بين فضاء عناوين الجهاز وفضاء عناوين الـ CPU.

```c
struct of_pci_range {
    union {
        u64 pci_addr;       /* عنوان PCI (للبوصات من نوع PCI) */
        u64 bus_addr;       /* عنوان البوص الفرعي (generic) */
    };
    u64 cpu_addr;           /* العنوان المقابل في فضاء CPU */
    u64 parent_bus_addr;    /* عنوان البوص الأم (للترجمة المتعددة المستويات) */
    u64 size;               /* حجم النافذة بالبايت */
    u32 flags;              /* IORESOURCE_MEM / IORESOURCE_IO / ... */
};
#define of_range of_pci_range  /* alias عام */
```

| الحقل | الشرح |
|---|---|
| `pci_addr` / `bus_addr` | الطرف الأول من التعيين (جانب الجهاز) |
| `cpu_addr` | العنوان الفيزيائي للـ CPU بعد الترجمة الكاملة |
| `parent_bus_addr` | يُستخدم داخلياً لدعم الترجمة متعددة المستويات (bridge → bridge → CPU) |
| `size` | حجم النطاق |
| `flags` | نوع المورد (`IORESOURCE_MEM`, `IORESOURCE_IO`, إلخ) |

---

#### 1.3 `struct device_node` (من `of.h`)

**الغرض:** يمثّل عقدة في شجرة الأجهزة المُحمَّلة في الذاكرة.

```c
struct device_node {
    const char      *name;          /* اسم العقدة */
    phandle          phandle;       /* معرّف للإشارة من عقد أخرى */
    const char      *full_name;     /* المسار الكامل */
    struct fwnode_handle fwnode;    /* واجهة firmware node العامة */
    struct property *properties;   /* قائمة الخصائص */
    struct property *deadprops;    /* خصائص محذوفة (لا تزال في الذاكرة) */
    struct device_node *parent;    /* العقدة الأم */
    struct device_node *child;     /* أول ابن */
    struct device_node *sibling;   /* الأخ التالي */
    unsigned long    _flags;        /* OF_DYNAMIC, OF_POPULATED, ... */
    void            *data;          /* بيانات خاصة بالمعمارية */
};
```

---

#### 1.4 `struct property` (من `of.h`)

**الغرض:** يمثّل خاصية واحدة (مثل `reg`, `ranges`) داخل عقدة.

```c
struct property {
    char            *name;    /* اسم الخاصية */
    int              length;  /* الحجم بالبايت */
    void            *value;   /* البيانات الخام (big-endian) */
    struct property *next;    /* الخاصية التالية في القائمة */
};
```

---

#### 1.5 `struct resource` (من `ioport.h`)

**الغرض:** يمثّل مورد نظام (I/O port أو MMIO range أو IRQ) بعد تحويله من تنسيق Device Tree.

```c
struct resource {
    resource_size_t  start;    /* بداية النطاق (شاملة) */
    resource_size_t  end;      /* نهاية النطاق (شاملة) */
    const char      *name;     /* اسم المورد للعرض */
    unsigned long    flags;    /* IORESOURCE_MEM / IO / IRQ / ... */
    unsigned long    desc;     /* IORES_DESC_* للبحث في iomem */
    struct resource *parent;   /* في الشجرة الهرمية للموارد */
    struct resource *sibling;  /* المورد التالي في نفس المستوى */
    struct resource *child;    /* أول مورد فرعي */
};
```

---

### 2. مخططات العلاقات بين الـ Structs

#### 2.1 العلاقة العامة

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                        device_node                               │
  │  name, full_name, phandle, _flags                                │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
  │  │  properties  │  │    parent    │  │    child     │           │
  │  │  (linked     │  │  (device_    │  │  (device_    │           │
  │  │   list of    │  │   node *)    │  │   node *)    │           │
  │  │   property)  │  └──────────────┘  └──────────────┘           │
  │  └──────┬───────┘                                                │
  └─────────┼────────────────────────────────────────────────────────┘
            │
            ▼
  ┌──────────────────────┐
  │      property        │
  │  name="ranges"       │
  │  length=N bytes      │
  │  value=[ __be32 ... ]│◄──── يقرأه parser مباشرةً
  │  *next               │
  └──────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │                   of_pci_range_parser                            │
  │  *node ──────────────────────────────────► device_node           │
  │  *bus  (internal of_bus ops, static)                             │
  │  *range ─► يشير داخل property.value                              │
  │  *end   ─► property.value + property.length                      │
  │  na, ns, pna, dma                                                │
  └───────────────────────────┬──────────────────────────────────────┘
                              │ of_pci_range_parser_one()
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                     of_pci_range                                 │
  │  bus_addr / pci_addr                                             │
  │  cpu_addr                                                        │
  │  parent_bus_addr                                                 │
  │  size                                                            │
  │  flags                                                           │
  └───────────────────────────┬──────────────────────────────────────┘
                              │ of_pci_range_to_resource()
                              │ of_address_to_resource()
                              ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                       struct resource                            │
  │  start = cpu_addr                                                │
  │  end   = cpu_addr + size - 1                                     │
  │  flags = IORESOURCE_MEM / IO                                     │
  │  *parent, *sibling, *child (شجرة iomem_resource)                │
  └──────────────────────────────────────────────────────────────────┘
```

#### 2.2 شجرة `device_node` (مثال: SoC مع PCI bridge)

```
  of_root (/)
     │
     ├── soc@f0000000          [device_node]
     │     │  properties: [#address-cells=1, #size-cells=1, ranges=...]
     │     │
     │     └── pci@e0000000   [device_node]
     │           properties: [#address-cells=3, #size-cells=2,
     │                         ranges=<0x2000000 0 0x10000000
     │                                 0xe0000000 0 0x10000000>]
     │
     └── memory@80000000       [device_node]
```

---

### 3. دورة حياة الـ Range Parser

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  CREATION / INIT                                                │
  │                                                                 │
  │  of_pci_range_parser_init(parser, node)                         │
  │    │                                                            │
  │    ├── يبحث في node->properties عن "ranges"                    │
  │    ├── يحسب na = of_n_addr_cells(node->parent)                 │
  │    │          ns = of_n_size_cells(node)                       │
  │    │          pna = of_n_addr_cells(node->parent->parent)      │
  │    ├── يضبط parser->range = property->value                    │
  │    └── يضبط parser->end   = value + length/4                  │
  │                                                                 │
  │  of_pci_dma_range_parser_init(parser, node)                     │
  │    └── نفس الخطوات لكن يبحث عن "dma-ranges" ويضبط dma=true   │
  └───────────────────────┬─────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  ITERATION / USAGE                                              │
  │                                                                 │
  │  for_each_of_range(parser, range)                               │
  │    │                                                            │
  │    └── of_pci_range_parser_one(parser, range)                   │
  │          │                                                      │
  │          ├── يقرأ (na + pna + ns) خلايا من parser->range      │
  │          ├── يفك تشفير bus_addr, cpu_addr, size, flags         │
  │          ├── يترجم cpu_addr عبر سلسلة of_translate_address()  │
  │          ├── يُقدِّم parser->range += (na + pna + ns)          │
  │          └── يُعيد range (أو NULL إذا وصل نهاية المصفوفة)    │
  └───────────────────────┬─────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  TEARDOWN                                                       │
  │                                                                 │
  │  لا يوجد تحرير صريح — parser على الـ stack                     │
  │  of_node_put(node) إذا أُخذت reference على العقدة              │
  └─────────────────────────────────────────────────────────────────┘
```

---

### 4. مخططات تدفق الاستدعاءات

#### 4.1 `of_address_to_resource()` — تحويل `reg` إلى `struct resource`

```
  driver / platform code
    │
    └── of_address_to_resource(node, index, &res)
          │
          ├── __of_get_address(node, index, -1, &size, &flags)
          │     │
          │     ├── of_get_property(node, "reg", &len)
          │     │     └── يجد خاصية "reg" في node->properties
          │     │
          │     └── يُعيد مؤشر للإدخال [index] في مصفوفة reg
          │
          ├── of_translate_address(node, addr_cells_ptr)
          │     │
          │     ├── يصعد شجرة device_node (child → parent → ...)
          │     ├── لكل مستوى: يبحث عن "ranges"
          │     ├── يُطابق عنوان الإدخال مع نوافذ ranges
          │     └── يُضيف offset الترجمة حتى يصل للـ CPU space
          │
          └── يملأ res->start, res->end, res->flags, res->name
```

#### 4.2 `of_iomap()` — تعيين MMIO مباشرةً

```
  driver->probe()
    │
    └── of_iomap(node, index)
          │
          ├── of_address_to_resource(node, index, &res)
          │     └── [تدفق الاستدعاءات السابق]
          │
          └── ioremap(res.start, resource_size(&res))
                │
                └── يُعيد void __iomem * للاستخدام في readl/writel
```

#### 4.3 `of_io_request_and_map()` — حجز + تعيين

```
  driver->probe()
    │
    └── of_io_request_and_map(node, index, "my-device")
          │
          ├── of_address_to_resource(node, index, &res)
          │
          ├── request_mem_region(res.start, size, name)
          │     └── يحجز النطاق في iomem_resource tree
          │           (يمنع التعارض مع driverات أخرى)
          │
          └── ioremap(res.start, size)
                └── يُعيد void __iomem *
```

#### 4.4 `for_each_of_range` — المسح الكامل لـ ranges

```
  pci host driver
    │
    ├── of_pci_range_parser_init(&parser, node)
    │
    └── for_each_of_range(&parser, &range)
          │
          └── of_pci_range_parser_one(&parser, &range)
                │
                ├── يُفكِّك إدخال ranges (3+2+2 خلايا مثلاً)
                ├── يستدعي bus->translate() لترجمة العنوان
                └── يملأ:
                      range.bus_addr  ← عنوان PCI space
                      range.cpu_addr  ← عنوان CPU space (مُترجَم)
                      range.size      ← حجم النافذة
                      range.flags     ← IORESOURCE_MEM/IO

                ثم بعد الحلقة:
                of_pci_range_to_resource(&range, node, &res)
                  └── pci_register_io_range() أو
                      insert_resource(&iomem_resource, &res)
```

#### 4.5 `of_translate_dma_address()` — ترجمة DMA

```
  dma subsystem / IOMMU
    │
    └── of_translate_dma_address(dev_node, dma_addr_cells)
          │
          ├── يصعد الشجرة بحثاً عن "dma-ranges"
          ├── يُطابق العنوان مع نوافذ dma-ranges
          └── يُعيد العنوان الفيزيائي للـ CPU (u64)
                أو OF_BAD_ADDR عند الفشل
```

---

### 5. استراتيجية الـ Locking

**الـ `of_address.h` نفسه لا يُعرِّف locks** — يعتمد على الـ locking المُعرَّف في `of.h` وتطبيقات الـ core.

#### 5.1 الـ Locks الرئيسية في OF Address Layer

| Lock | النوع | يحمي ماذا |
|---|---|---|
| `of_mutex` (في `of/base.c`) | `mutex` | قراءة/كتابة خصائص العقد (`property->value`) |
| `devtree_lock` (SPARC) | `spinlock` | الوصول للـ device tree على SPARC |
| `iomem_resource` tree lock | `rwlock` (داخل `resource.c`) | إضافة/حذف موارد من شجرة iomem |

#### 5.2 قواعد الـ Locking عند استخدام `of_address.h`

```
  قراءة فقط (read-only path):
  ─────────────────────────────
  of_address_to_resource()
  of_translate_address()
  of_get_address()
  __of_get_address()
    └── يستدعون of_get_property() داخلياً
          └── يأخذ of_mutex (read lock) ضمنياً

  لا يحتاج الـ caller لـ lock صريح.

  كتابة (request path):
  ─────────────────────
  of_io_request_and_map()
    └── request_mem_region()
          └── يأخذ resource_lock (rwlock write) داخلياً
                ← يجب عدم استدعائه من atomic context
```

#### 5.3 ترتيب الـ Locks (Lock Ordering)

```
  [الأعلى أولاً — يجب الأخذ بهذا الترتيب]

  1. of_mutex            (لحماية device_tree properties)
       ↓
  2. resource_lock       (لحماية iomem_resource tree)
       ↓
  3. ioremap internal    (لا يُشترط lock من الـ caller)

  تحذير: لا تأخذ of_mutex وأنت تحمل resource_lock — عكس الترتيب يُسبب deadlock.
```

#### 5.4 مثال تطبيقي

```c
/* آمن: قراءة ثم حجز المورد */
int my_driver_probe(struct platform_device *pdev)
{
    struct resource res;

    /* 1. قراءة العنوان من DT — يأخذ of_mutex داخلياً */
    if (of_address_to_resource(pdev->dev.of_node, 0, &res))
        return -ENODEV;

    /* 2. حجز المورد — يأخذ resource_lock داخلياً */
    base = of_io_request_and_map(pdev->dev.of_node, 0, "my-driver");
    if (IS_ERR(base))
        return PTR_ERR(base);

    return 0;
}
```

---

### ملخص العلاقات النهائية

```
  Device Tree Binary (DTB)
         │
         │ boot: unflatten_device_tree()
         ▼
  device_node tree (in RAM)
    └── properties (linked list: name + raw __be32 data)
         │
         │ of_address_to_resource() / of_translate_address()
         ▼
  of_pci_range_parser  ──reads──►  property.value (ranges/__be32 array)
         │
         │ of_pci_range_parser_one()
         ▼
  of_pci_range  (decoded: bus_addr, cpu_addr, size, flags)
         │
         │ of_pci_range_to_resource() / of_address_to_resource()
         ▼
  struct resource  (start, end, flags)
         │
         │ ioremap() / request_mem_region()
         ▼
  void __iomem *  →  readl / writel  →  Hardware Registers
```
## Phase 4: شرح الـ Functions

---

### ملخص شامل — Cheatsheet

#### جدول الـ Functions الرئيسية

| Function | Category | Returns | Purpose |
|---|---|---|---|
| `of_translate_address()` | Address Translation | `u64` | ترجمة عنوان bus إلى عنوان CPU |
| `of_translate_dma_address()` | DMA Translation | `u64` | ترجمة عنوان DMA من device-space إلى CPU-space |
| `of_translate_dma_region()` | DMA Translation | `const __be32 *` | ترجمة منطقة DMA كاملة وإرجاع start/length |
| `of_address_to_resource()` | Resource Mapping | `int` | تحويل عنوان OF node إلى `struct resource` |
| `of_iomap()` | MMIO Mapping | `void __iomem *` | قراءة reg[index] وعمل `ioremap` مباشرة |
| `of_io_request_and_map()` | MMIO Mapping | `void __iomem *` | مثل `of_iomap` لكن يحجز المنطقة بـ `request_mem_region` |
| `__of_get_address()` | Low-level Helpers | `const __be32 *` | قراءة raw address من `reg` أو PCI BAR |
| `of_get_address()` | Low-level Helpers | `const __be32 *` | wrapper لـ `__of_get_address` بـ index |
| `of_get_pci_address()` | Low-level Helpers | `const __be32 *` | wrapper لـ `__of_get_address` بـ BAR number |
| `of_property_read_reg()` | Low-level Helpers | `int` | قراءة addr+size من `reg` property بصورة portable |
| `of_address_count()` | Low-level Helpers | `int` | حساب عدد عناوين الـ reg[] |
| `of_pci_range_parser_init()` | Range Parsing | `int` | تهيئة parser لـ `ranges` property |
| `of_pci_dma_range_parser_init()` | Range Parsing | `int` | تهيئة parser لـ `dma-ranges` property |
| `of_pci_range_parser_one()` | Range Parsing | `struct of_pci_range *` | قراءة entry واحدة من الـ ranges |
| `of_range_count()` | Range Parsing | `int` | حساب عدد entries في الـ ranges |
| `of_pci_address_to_resource()` | PCI Helpers | `int` | تحويل PCI BAR إلى `struct resource` |
| `of_pci_range_to_resource()` | PCI Helpers | `int` | تحويل `of_pci_range` إلى `struct resource` |
| `of_range_to_resource()` | Generic Helpers | `int` | تحويل ranges[index] إلى `struct resource` |
| `of_dma_is_coherent()` | DMA Attributes | `bool` | فحص إذا كان الـ device يدعم DMA coherency |

#### جدول الـ Macros / Aliases

| Macro | Expands To | Purpose |
|---|---|---|
| `of_range` | `of_pci_range` | Generic alias |
| `of_range_parser` | `of_pci_range_parser` | Generic alias |
| `of_range_parser_init` | `of_pci_range_parser_init` | Generic alias |
| `for_each_of_pci_range` | loop macro | Iterator على كل range |
| `for_each_of_range` | `for_each_of_pci_range` | Generic alias |

---

### Category 1: Address Translation

هذه المجموعة مسؤولة عن ترجمة العناوين من فضاء الـ device/bus إلى فضاء الـ CPU، وهو جوهر عمل الـ OF address subsystem. الـ DT يصف العناوين بصيغة relative لكل bus، وهذه الـ functions تتبع سلسلة الـ `ranges` properties صعوداً في الـ device tree حتى تصل إلى عنوان CPU مطلق.

---

#### `of_translate_address()`

```c
extern u64 of_translate_address(struct device_node *np, const __be32 *addr);
```

تترجم عنواناً مخزناً بصيغة big-endian cells داخل الـ DT إلى عنوان فيزيائي مطلق يفهمه الـ CPU. تعتمد على traversal لسلسلة `ranges` properties من الـ node الحالي صعوداً حتى الـ root، وكل مستوى يُجري offset translation.

**Parameters:**
- `np` — الـ `device_node` الذي يحتوي على الـ address المراد ترجمته.
- `addr` — مؤشر لـ array من `__be32` يمثل العنوان بصيغة bus-specific (عدد الـ cells يُحدَّد من `#address-cells`).

**Return:**
- الـ `u64` يمثل العنوان الفيزيائي على الـ CPU، أو `OF_BAD_ADDR` (`(u64)-1`) عند الفشل.

**Key details:**
- تعمل فقط عند تفعيل `CONFIG_OF_ADDRESS`؛ وإلا ترجع `OF_BAD_ADDR`.
- لا تأخذ أي lock بنفسها — تعتمد على `of_node_get/put` في المسار.
- تتعامل مع bus types مختلفة (simple-bus، PCI) عبر الـ `of_bus` abstraction.

**Who calls it:** `of_address_to_resource()`، وكود ترجمة عناوين PCI، وأي driver يحتاج raw address translation.

---

#### `of_translate_dma_address()`

```c
extern u64 of_translate_dma_address(struct device_node *dev,
                                    const __be32 *in_addr);
```

مشابهة لـ `of_translate_address()` لكنها تعمل على الـ `dma-ranges` بدلاً من `ranges`. تترجم عنوان DMA كما يراه الـ device إلى عنوان يراه الـ CPU في الـ memory map.

**Parameters:**
- `dev` — الـ node الخاص بالـ device المُصدِر لعمليات DMA.
- `in_addr` — العنوان بصيغة big-endian cells كما يراه الـ device على الـ bus.

**Return:**
- `u64` للعنوان المُترجَم، أو `OF_BAD_ADDR` عند الفشل.

**Key details:**
- متاحة دائماً بغض النظر عن `CONFIG_OF_ADDRESS` لأن DMA translation مطلوبة أحياناً حتى بدون full address support.
- تستخدمها الـ IOMMU و DMA mapping layers.

---

#### `of_translate_dma_region()`

```c
extern const __be32 *of_translate_dma_region(struct device_node *dev,
                                             const __be32 *addr,
                                             phys_addr_t *start,
                                             size_t *length);
```

تترجم region كاملة من الـ `dma-ranges` — تُعيد معها عنوان البداية والحجم في آنٍ واحد.

**Parameters:**
- `dev` — الـ device node.
- `addr` — مؤشر لبداية الـ range entry في الـ property.
- `start` — output، العنوان الفيزيائي للبداية على الـ CPU.
- `length` — output، حجم المنطقة بالـ bytes.

**Return:**
- مؤشر للـ `__be32` التالي في الـ property (للسماح بالـ iteration)، أو `NULL` عند الفشل.

**Who calls it:** الـ DMA mapping setup في bus drivers، وخاصة عند بناء الـ DMA aperture.

---

### Category 2: Resource Mapping

هذه المجموعة تُجسِّر عالم الـ Device Tree مع الـ `struct resource` المستخدمة في كل Linux kernel driver. تقرأ الـ `reg` property، تُترجم العنوان، وتملأ الـ resource structure جاهزة للاستخدام.

---

#### `of_address_to_resource()`

```c
extern int of_address_to_resource(struct device_node *dev, int index,
                                  struct resource *r);
```

تقرأ الـ entry رقم `index` من `reg` property للـ node، تُترجم عنوانها إلى CPU physical address، وتملأ الـ `struct resource` كاملةً بالـ start/end/flags.

**Parameters:**
- `dev` — الـ device node الذي يحتوي على `reg`.
- `index` — رقم الـ entry (0-based) في الـ `reg` array.
- `r` — output، الـ resource المراد ملؤها.

**Return:**
- `0` عند النجاح، error code سالب عند الفشل (`-EINVAL` عند عدم وجود `reg` أو فشل الترجمة).

**Key details:**
- متاحة تحت `CONFIG_OF` (وليس فقط `CONFIG_OF_ADDRESS`) مما يجعلها الأكثر استخداماً.
- تضبط `r->flags` بناءً على نوع الـ bus (MMIO → `IORESOURCE_MEM`, I/O → `IORESOURCE_IO`).
- كود فالسك الشائع: نسيان فحص القيمة المُعادة — الـ driver يُكمل بـ resource فارغة.

**Pseudocode:**
```c
// simplified flow
of_address_to_resource(dev, index, r):
    addr = __of_get_address(dev, index, -1, &size, &flags)
    if !addr: return -EINVAL
    cpu_addr = of_translate_address(dev, addr)
    if cpu_addr == OF_BAD_ADDR: return -EINVAL
    r->start = cpu_addr
    r->end   = cpu_addr + size - 1
    r->flags = flags
    r->name  = dev->full_name
    return 0
```

---

#### `of_iomap()`

```c
void __iomem *of_iomap(struct device_node *node, int index);
```

الـ convenience function الأكثر استخداماً في drivers: تقرأ `reg[index]`، تُترجم العنوان، ثم تستدعي `ioremap()` مباشرة. لا تحجز المنطقة في kernel resource tree.

**Parameters:**
- `node` — الـ device node.
- `index` — رقم الـ reg entry.

**Return:**
- مؤشر `void __iomem *` جاهز للـ `readl/writel`، أو `NULL` عند الفشل.

**Key details:**
- تعمل تحت `CONFIG_OF`.
- لا تستدعي `request_mem_region()` — لا يظهر الـ mapping في `/proc/iomem`.
- يجب مقابلتها بـ `iounmap()` عند cleanup.
- الأسرع للـ bring-up، لكن `of_io_request_and_map()` هي الأصح للـ production.

**Who calls it:** غالبية platform drivers في `probe()` callback.

---

#### `of_io_request_and_map()`

```c
void __iomem *of_io_request_and_map(struct device_node *device,
                                    int index, const char *name);
```

تجمع `request_mem_region()` + `ioremap()` في استدعاء واحد. تحجز المنطقة في الـ kernel resource tree وتُنشئ الـ mapping.

**Parameters:**
- `device` — الـ device node.
- `index` — رقم الـ reg entry.
- `name` — اسم يُسجَّل في `/proc/iomem` (عادةً اسم الـ driver أو الـ peripheral).

**Return:**
- `void __iomem *` عند النجاح، أو `IOMEM_ERR_PTR(-errno)` عند الفشل — يجب فحصه بـ `IS_ERR()`.

**Key details:**
- عند الفشل في الـ `request`، ترجع `IOMEM_ERR_PTR(-EBUSY)` إذا كانت المنطقة محجوزة.
- تحت `CONFIG_OF_ADDRESS` فقط (الـ fallback ترجع `-EINVAL`).
- الأنسب للـ production drivers — تضمن عدم تعارض الـ address regions.
- مقابلها في الـ cleanup: `iounmap()` ثم `release_mem_region()` — أو استخدم `devm_of_iomap()` بدلاً منها.

---

### Category 3: Low-level Address Helpers

هذه الـ functions تُتيح الوصول المباشر للبيانات الخام في `reg` property، قبل أي ترجمة. مفيدة لـ bus drivers والكود الذي يحتاج raw cell values.

---

#### `__of_get_address()`

```c
extern const __be32 *__of_get_address(struct device_node *dev, int index,
                                      int bar_no, u64 *size,
                                      unsigned int *flags);
```

الـ low-level accessor للـ `reg` property. تُحدِّد `#address-cells` و`#size-cells` من الـ parent node، ثم تُعيد مؤشراً للـ cells التي تمثل الـ entry المطلوبة.

**Parameters:**
- `dev` — الـ device node.
- `index` — رقم الـ entry بـ index عادي (`bar_no = -1` في هذه الحالة).
- `bar_no` — رقم PCI BAR (`index = -1` في هذه الحالة). أحدهما فقط مُستخدَم.
- `size` — output، حجم المنطقة بالـ bytes.
- `flags` — output، الـ `IORESOURCE_*` flags المقابلة لنوع الـ bus.

**Return:**
- مؤشر `const __be32 *` لأول cell في الـ address، أو `NULL` عند الفشل.

**Key details:**
- المؤشر المُعاد يشير مباشرة داخل الـ DT blob — لا copy، لا allocation.
- تُحوِّل bus-specific flags (مثل PCI space bits) إلى `IORESOURCE_*` flags.
- تستدعيها `of_get_address()` و`of_get_pci_address()` كـ thin wrappers.

---

#### `of_get_address()`

```c
static inline const __be32 *of_get_address(struct device_node *dev, int index,
                                           u64 *size, unsigned int *flags)
{
    return __of_get_address(dev, index, -1, size, flags);
}
```

Wrapper مُبسَّط لقراءة عنوان بـ index للـ non-PCI devices.

**Parameters:**
- `dev` — الـ device node.
- `index` — رقم الـ entry في `reg`.
- `size` — output الحجم.
- `flags` — output الـ flags.

**Return:** مؤشر raw address cells أو `NULL`.

---

#### `of_get_pci_address()`

```c
static inline const __be32 *of_get_pci_address(struct device_node *dev,
                                               int bar_no,
                                               u64 *size, unsigned int *flags)
{
    return __of_get_address(dev, -1, bar_no, size, flags);
}
```

Wrapper لقراءة عنوان PCI BAR المحدد.

**Parameters:**
- `dev` — الـ PCI device node.
- `bar_no` — رقم الـ BAR (0–5).
- `size`، `flags` — كما سبق.

**Return:** مؤشر raw address cells أو `NULL`.

---

#### `of_property_read_reg()`

```c
int of_property_read_reg(struct device_node *np, int idx, u64 *addr, u64 *size);
```

قراءة portable لـ `reg[idx]` — تُعيد الـ addr والـ size مباشرةً كـ `u64` بدون الحاجة لتعامل الـ caller مع `#address-cells` و`#size-cells`.

**Parameters:**
- `np` — الـ device node.
- `idx` — رقم الـ entry.
- `addr` — output، عنوان الـ reg entry (كما يظهر في DT، قبل ترجمة bus).
- `size` — output، حجم المنطقة.

**Return:**
- `0` عند النجاح، أو error code سالب (`-ENOENT`، `-EINVAL`).

**Key details:**
- أبسط من `__of_get_address()` للـ drivers التي لا تحتاج raw cells.
- لا تُترجم العنوان — ترجع القيمة كما هي في الـ DT.
- تحت `CONFIG_OF_ADDRESS`؛ الـ fallback ترجع `-ENOSYS`.

---

#### `of_address_count()`

```c
static inline int of_address_count(struct device_node *np)
{
    struct resource res;
    int count = 0;
    while (of_address_to_resource(np, count, &res) == 0)
        count++;
    return count;
}
```

تعدّ عدد الـ address entries في `reg` property عبر iteration حتى يفشل `of_address_to_resource()`.

**Return:** عدد الـ entries الصالحة (يمكن أن يكون 0).

**Key details:**
- تعتمد على `of_address_to_resource()` — غير مباشرة وتُكرر الـ parsing.
- مفيدة لـ drivers التي تحتاج معرفة عدد الـ regions ديناميكياً قبل allocation.

---

### Category 4: Range Parsing

الـ `ranges` و`dma-ranges` properties تصف كيف تُترجَم عناوين الـ child bus إلى عناوين الـ parent bus. هذه المجموعة تُوفر iterator-based API لقراءة هذه الـ ranges entry بـ entry.

---

#### `of_pci_range_parser_init()`

```c
extern int of_pci_range_parser_init(struct of_pci_range_parser *parser,
                                    struct device_node *node);
```

تُهيئ الـ `of_pci_range_parser` لقراءة الـ `ranges` property من الـ node المحدد. تضبط الـ `na/ns/pna` fields بناءً على `#address-cells` من الـ parent والـ child.

**Parameters:**
- `parser` — مؤشر لـ parser state يجب أن يكون valid.
- `node` — الـ device node الذي يحتوي على `ranges`.

**Return:**
- `0` عند النجاح، أو error سالب (`-ENOENT` إذا لم تكن `ranges` موجودة).

**Key details:**
- `of_range_parser_init` هو alias مُعرَّف بـ `#define`.
- الـ parser state يُحتفظ به بين الاستدعاءات لـ `of_pci_range_parser_one()`.
- بعد init يُستخدم عادةً مباشرةً في `for_each_of_range()`.

---

#### `of_pci_dma_range_parser_init()`

```c
extern int of_pci_dma_range_parser_init(struct of_pci_range_parser *parser,
                                        struct device_node *node);
```

مطابقة لـ `of_pci_range_parser_init()` لكنها تستهدف `dma-ranges` property بدلاً من `ranges`. تضبط `parser->dma = true`.

**Parameters:** نفس `of_pci_range_parser_init()`.

**Return:** `0` أو error سالب.

**Who calls it:** الـ PCI host controller drivers وكل bus driver يحتاج بناء DMA aperture.

---

#### `of_pci_range_parser_one()`

```c
extern struct of_pci_range *of_pci_range_parser_one(
                                struct of_pci_range_parser *parser,
                                struct of_pci_range *range);
```

تقرأ الـ entry الحالية من الـ parser وتُقدِّم المؤشر للـ entry التالية. هي الـ engine خلف `for_each_of_range()`.

**Parameters:**
- `parser` — الـ parser state (مُعدَّل in-place لكل استدعاء).
- `range` — output، الـ `of_pci_range` المراد ملؤه.

**Return:**
- مؤشر `range` ذاته عند النجاح (non-NULL)، أو `NULL` عند انتهاء الـ entries.

**Key details:**
- تملأ `range->pci_addr` (أو `bus_addr`)، `range->cpu_addr`، `range->size`، `range->flags`.
- تُقدِّم `parser->range` بمقدار `(na + pna + ns)` cells لكل استدعاء.
- الـ `cpu_addr` يمر بـ `of_translate_address()` داخلياً.

**Pseudocode للـ iterator:**
```c
// for_each_of_range expands to:
for (; of_pci_range_parser_one(&parser, &range); ) {
    // range.cpu_addr, range.size, range.flags جاهزة
    process_range(&range);
}
```

---

#### `of_range_count()`

```c
static inline int of_range_count(const struct of_range_parser *parser)
{
    if (!parser || !parser->node || !parser->range ||
        parser->range == parser->end)
        return 0;
    return (parser->end - parser->range) / (parser->na + parser->pna + parser->ns);
}
```

تحسب عدد الـ entries المتبقية في الـ parser بقسمة المسافة بين `range` و`end` على حجم الـ entry الواحدة.

**Parameters:**
- `parser` — الـ parser state بعد init.

**Return:** عدد الـ entries، أو `0` إذا كان الـ parser فارغاً أو invalid.

**Key details:**
- كما ينبه التعليق في الـ header: استدعاؤها أثناء أو بعد `for_each_of_range()` سيُعطي نتيجة غير صحيحة لأن `parser->range` يتقدم.
- مفيدة فقط قبل بدء الـ iteration لمعرفة الحجم مسبقاً (مثلاً لـ allocation).

---

### Category 5: PCI & Generic Resource Helpers

تتعامل مع تحويل الـ PCI addresses والـ ranges إلى `struct resource` الجاهزة للتسجيل في الـ kernel resource tree.

---

#### `of_pci_address_to_resource()`

```c
extern int of_pci_address_to_resource(struct device_node *dev, int bar,
                                      struct resource *r);
```

تقرأ PCI BAR رقم `bar` من الـ `reg` property لـ PCI device، تُترجم العنوان، وتملأ الـ `struct resource`.

**Parameters:**
- `dev` — الـ PCI device node.
- `bar` — رقم الـ BAR (0–5).
- `r` — output resource.

**Return:**
- `0` عند النجاح، أو error سالب.

**Key details:**
- تستدعي `__of_get_address()` مع `bar_no` بدلاً من `index`.
- تُفرِّق بين 32-bit و64-bit BAR عبر الـ flags المُستخرجة.

---

#### `of_pci_range_to_resource()`

```c
extern int of_pci_range_to_resource(const struct of_pci_range *range,
                                    const struct device_node *np,
                                    struct resource *res);
```

تُحوِّل `struct of_pci_range` المُملوءة مسبقاً بـ `of_pci_range_parser_one()` إلى `struct resource`.

**Parameters:**
- `range` — الـ range المُحوَّلة (input).
- `np` — الـ device node (للـ naming فقط).
- `res` — output resource.

**Return:**
- `0` عند النجاح، `IORESOURCE_UNSET` بـ flag عند عنوان غير صالح.

**Who calls it:** الـ PCI host bridge drivers بعد `for_each_of_range()` loop لبناء الـ bus windows.

---

#### `of_range_to_resource()`

```c
extern int of_range_to_resource(struct device_node *np, int index,
                                struct resource *res);
```

API مباشرة للحصول على `struct resource` من `ranges[index]` دون الحاجة لإنشاء parser يدوياً.

**Parameters:**
- `np` — الـ device node الذي يحتوي على `ranges`.
- `index` — رقم الـ range entry.
- `res` — output resource.

**Return:**
- `0` عند النجاح، أو error سالب.

**Key details:**
- تُبسِّط الاستخدام الشائع: init parser + one iteration + convert → في استدعاء واحد.

---

### Category 6: DMA Attributes

---

#### `of_dma_is_coherent()`

```c
extern bool of_dma_is_coherent(struct device_node *np);
```

تفحص إذا كان الـ device أو أحد أسلافه يحمل الـ property `dma-coherent` في الـ DT، مما يعني أن الـ DMA يعمل بـ cache coherency تلقائياً دون الحاجة لـ explicit cache operations.

**Parameters:**
- `np` — الـ device node.

**Return:**
- `true` إذا وُجدت `dma-coherent` property في المسار من `np` إلى الـ root، `false` إذا لم تُوجد.

**Key details:**
- تمشي في سلسلة الـ parent nodes حتى تجد الـ property أو تصل الـ root.
- تُستخدم في بناء الـ `dma_ops` للـ device — تُحدد إذا كان `dma_sync_*` operations مطلوبة.
- تحت `CONFIG_OF_ADDRESS`؛ الـ fallback ترجع `false`.
- في الأنظمة HW-coherent كبعض ARM64 SOCs قد لا تكون هذه الـ property ضرورية.

**Who calls it:** `of_dma_configure()` في DMA mapping framework، وأي bus driver يُقرر الـ DMA coherency policy.

---

### ملاحظات معمارية

```
Device Tree Node
     │
     │  reg = <addr size>
     ▼
__of_get_address()          ← raw cells من DT blob
     │
     ▼
of_translate_address()      ← traversal سلسلة ranges صعوداً
     │
     ▼
CPU Physical Address
     │
     ├──► of_address_to_resource()   → struct resource (kernel resource tree)
     │
     └──► of_iomap() / of_io_request_and_map()  → void __iomem * (MMIO mapping)
```

الـ `ranges` property هي الجسر بين مستويات الـ bus hierarchy. كل مستوى يُضيف offset. الـ OF address subsystem يُؤتمت هذا التحويل الهرمي بشكل كامل.
## Phase 5: دليل الـ Debugging الشامل

### المقدمة

**الـ `of_address.h`** يُعرِّف واجهات ترجمة العناوين في **Device Tree** — من عنوان الجهاز (device address) إلى عنوان المعالج (CPU address)، ومعالجة **`reg`** و**`ranges`** و**`dma-ranges`**. أي خطأ في الترجمة ينتج عنه `NULL` pointer أو وصول خاطئ للذاكرة أو فشل في الـ DMA.

---

## الـ Debugging على المستوى البرمجي (Software Level)

### 1. مدخلات الـ debugfs

الـ `of_address` لا يملك مدخلات `debugfs` مستقلة، لكن الـ subsystems المرتبطة تُوفّرها:

```bash
# عرض شجرة الـ Device Tree المفسَّرة بالكامل
ls /sys/kernel/debug/devicetree/

# قراءة محتوى node معين
cat /sys/kernel/debug/devicetree/base/soc/serial@1000000/reg

# عرض resource tree الكاملة (مهم لـ of_address_to_resource)
cat /sys/kernel/debug/iomap
```

مثال على المخرجات:
```
0000000001000000-0000000001000fff : serial@1000000
  0000000001000000-0000000001000fff : serial@1000000
```
التفسير: السطر الأول مدخل iomem، الثاني تأكيد الحجز. إذا غاب السطر الثاني → `request_mem_region` فشل.

---

### 2. مدخلات الـ sysfs

```bash
# عرض resources المخصصة لجهاز معين (PCI وغيره)
cat /sys/bus/platform/devices/1000000.serial/resources

# للـ PCI devices - ranges المترجَمة
cat /sys/bus/pci/devices/0000:00:00.0/resource

# فحص DMA coherency
cat /sys/bus/platform/devices/<dev>/dma_coherent
```

| الملف | المحتوى | كيفية التفسير |
|-------|---------|---------------|
| `resources` | start/end/flags | `flags=0x200` → `IORESOURCE_MEM` |
| `resource` (PCI) | BARs المترجَمة | عمود 3 = الـ flags |
| `dma_coherent` | 0 أو 1 | نتيجة `of_dma_is_coherent()` |

---

### 3. الـ ftrace — Tracepoints والـ Events

```bash
# تفعيل tracing لكل دوال of_address
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تتبع دالة of_translate_address
echo 'of_translate_address' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تتبع of_iomap وكل ما يستدعيها
echo 'of_iomap' > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer

# قراءة النتائج
cat /sys/kernel/debug/tracing/trace | grep -E 'of_(translate|iomap|address)'
```

مثال على مخرجات `function_graph`:
```
 1)               |  of_iomap() {
 1)               |    of_address_to_resource() {
 1)               |      __of_get_address() {
 1)   0.412 us    |        of_get_property();
 1)   1.203 us    |      }
 1)               |      of_translate_address() {
 1)   2.891 us    |        __of_translate_address();
 1)   3.102 us    |      }
 1)   4.998 us    |    }
 1)   5.234 us    |    ioremap();
 1)   6.100 us    |  }
```
**إذا توقف التتبع عند `__of_get_address` دون إكمال** → خاصية `reg` غائبة أو تالفة.

---

### 4. الـ printk والـ dynamic debug

```bash
# تفعيل dynamic debug لملفات of_address
echo 'file drivers/of/address.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/pci.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل رسائل OF subsystem
echo 'module of +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع معلومات إضافية (function name + line number)
echo 'file drivers/of/address.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# إيقاف التفعيل
echo 'file drivers/of/address.c -p' > /sys/kernel/debug/dynamic_debug/control
```

في بداية الـ boot قبل تفعيل `dynamic_debug`:
```bash
# أضف لسطر الـ kernel command line
dyndbg="file drivers/of/address.c +p"
# أو لتفعيل الكل
of.dyndbg="+p"
```

---

### 5. خيارات الـ Kernel Config للـ Debugging

| الـ Config | الوصف | متى تستخدمه |
|------------|--------|-------------|
| `CONFIG_OF_ADDRESS` | تفعيل الـ subsystem أصلاً | مطلوب دائماً |
| `CONFIG_DEBUG_DEVRES` | تتبع تسريبات `devm_*` | عند شك في `of_io_request_and_map` |
| `CONFIG_GENERIC_BUG` | تفعيل `BUG()`/`WARN()` | مطلوب للـ WARN_ON |
| `CONFIG_DEBUG_VM` | فحص صحة العناوين الافتراضية | عند `ioremap` خاطئ |
| `CONFIG_OF_UNITTEST` | تشغيل unit tests لـ OF | للتحقق من صحة الترجمة |
| `CONFIG_KASAN` | كشف الوصول الخاطئ للذاكرة | عند crash في `of_translate_address` |
| `CONFIG_DEBUG_RESOURCE` | فحص تعارض الـ resources | عند `request_mem_region` fails |
| `CONFIG_IOMMU_DEBUGFS` | debugfs للـ IOMMU/DMA | عند مشاكل DMA ranges |

```bash
# التحقق من تفعيل الخيارات في kernel الحالي
grep -E 'CONFIG_(OF_ADDRESS|DEBUG_DEVRES|OF_UNITTEST)' /boot/config-$(uname -r)
```

---

### 6. أدوات خاصة بالـ Subsystem

#### الـ dtc وفحص الـ DTS مسبقاً

```bash
# فحص صحة ملف DTS وترجمة الـ ranges يدوياً
dtc -I dts -O dtb -o /tmp/test.dtb myboard.dts 2>&1

# تحويل DTB إلى DTS مقروء من kernel المشغَّل
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | \
    grep -A 20 'serial@'
```

#### فحص **`/proc/iomem`**

```bash
# عرض كل resources المحجوزة — مرتبة هرمياً
cat /proc/iomem

# البحث عن device بعنوان معروف
grep '01000000' /proc/iomem
```

مثال:
```
01000000-01000fff : serial@1000000
  01000000-01000fff : serial@1000000
```

#### أداة **`fdtdump`** لفحص الـ DTB

```bash
# تفريغ DTB المستخدَم حالياً
cp /sys/firmware/fdt /tmp/current.dtb
fdtdump /tmp/current.dtb | grep -A 10 'serial@'
```

#### **`devmem2`** للتحقق من الترجمة

```bash
# قراءة من عنوان تم الحصول عليه بـ of_address_to_resource
devmem2 0x01000000 b   # قراءة byte واحد
# المخرج: Value at address 0x01000000 (0xc3456000): 0x60
```

---

### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|---------|------|
| `OF: Bad cell count for ...` | قيم `#address-cells` أو `#size-cells` خاطئة | راجع الـ DTS وتأكد من تطابق القيم مع الـ parent |
| `OF: no matching bus translation` | لا يوجد `ranges` في الـ parent node | أضف `ranges;` في الـ parent أو حدد القيم الصحيحة |
| `of_iomap: failed to iomap ...` | العنوان خارج نطاق الذاكرة أو محجوز | افحص `/proc/iomem` وتأكد من عدم التعارض |
| `can't map ... region` | `request_mem_region` فشل بسبب تعارض | ابحث في `/proc/iomem` عمن حجز العنوان |
| `OF: Bad address count for` | عدد entries في `reg` أقل من الـ `index` المطلوب | زد عدد entries في `reg` في الـ DTS |
| `DMA: mapping failed` | `of_translate_dma_address` أعاد `OF_BAD_ADDR` | افحص `dma-ranges` في الـ DTS |
| `of_translate_address: Bad cell count` | تنسيق `ranges` خاطئ | احسب الحجم: `(na + pna + ns) * 4` bytes |
| `no 'ranges' property` | الـ bus لا يملك ترجمة عناوين | أضف `ranges;` (passthrough) أو عرّف الترجمة |
| `Invalid device address` | العنوان يساوي `OF_BAD_ADDR (0xFFFFFFFFFFFFFFFF)` | راجع تسلسل `ranges` من الجذر حتى الجهاز |

---

### 8. نقاط استراتيجية لـ `dump_stack()` و`WARN_ON()`

```c
/* في driver يستخدم of_iomap — فحص النتيجة */
base = of_iomap(np, 0);
if (WARN_ON(!base)) {
    /* سيطبع stack trace + رسالة تحذير تلقائياً */
    return -ENOMEM;
}

/* عند ترجمة عنوان DMA */
dma_addr = of_translate_dma_address(dev->of_node, reg);
if (WARN_ON(dma_addr == OF_BAD_ADDR)) {
    dev_err(dev, "DMA address translation failed\n");
    dump_stack(); /* لمعرفة من استدعى هذا المسار */
    return -EINVAL;
}

/* فحص صحة الـ parser قبل الدوران */
ret = of_pci_range_parser_init(&parser, np);
if (WARN_ON_ONCE(ret)) {
    dev_err(dev, "ranges parser init failed: %d\n", ret);
    return ret;
}

/* فحص عدد العناوين المتوقع */
count = of_address_count(np);
WARN(count < 2, "device %s: expected >=2 reg entries, got %d\n",
     np->full_name, count);
```

**الأماكن الأكثر أهمية لـ `dump_stack()`:**
- عند إعادة `OF_BAD_ADDR` من أي دالة ترجمة
- عند فشل `of_io_request_and_map` (لمعرفة أي driver طلب العنوان)
- عند `of_range_count() == 0` في حين يتوقع الكود وجود entries

---

## الـ Debugging على المستوى الصلب (Hardware Level)

### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

```bash
# قارن عنوان الـ hardware الفعلي مع ما يراه الـ kernel
# 1. احصل على العنوان من الـ DTS
grep -r 'serial@' /sys/firmware/devicetree/base/ 2>/dev/null | head -5

# 2. احصل على العنوان المترجَم من kernel
cat /proc/iomem | grep serial

# 3. قارن بمعلومات الـ datasheet
# العنوان في الـ DTS يجب أن يكون عنوان الـ bus (قبل الترجمة)
# العنوان في /proc/iomem يجب أن يكون CPU physical address (بعد الترجمة)

# مثال: SoC بـ AXI bus عند 0x4000_0000
# DTS:     reg = <0x0 0x1000>;  (عنوان bus)
# ranges:  <0x0 0x0 0x40000000 0x10000000>;
# /proc/iomem: 40001000-40001fff  (CPU address)
```

### 2. تقنيات الـ Register Dump

```bash
# قراءة register بعنوان فيزيائي مباشر (يتطلب CONFIG_STRICT_DEVMEM=n أو CAP_SYS_RAWIO)
devmem2 0x40001000 w    # قراءة 32-bit word
devmem2 0x40001000 b    # قراءة byte
devmem2 0x40001000 h    # قراءة 16-bit

# باستخدام /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x40001000 / 4)) 2>/dev/null | xxd

# قراءة كتلة من الـ registers (مثلاً 256 byte)
dd if=/dev/mem bs=1 count=256 skip=$((0x40001000)) 2>/dev/null | xxd

# باستخدام io utility (من package iotools)
io -4 0x40001000   # قراءة 32-bit
io -4 -r 0x40001000 16  # قراءة 16 registers
```

**مقارنة العنوان الفيزيائي مع ما يقرأه الـ driver:**
```bash
# احصل على العنوان الذي حجزه الـ driver
cat /proc/iomem | grep 'my-driver-name'
# ثم اقرأ مباشرة من نفس العنوان بـ devmem2
```

---

### 3. نصائح Logic Analyzer والـ Oscilloscope

عند مشاكل `of_iomap` أو وصول خاطئ للـ bus:

**نقاط القياس على الـ AXI/AHB bus:**
```
CPU ──────────────────── AXI Bus ──────────────────── Peripheral
     AWADDR[31:0]                    AWADDR[31:0]
     AWVALID/AWREADY                 AWVALID/AWREADY
     WDATA[31:0]                     WDATA[31:0]
     RDATA[31:0]                     RDATA[31:0]
     BRESP[1:0]                      BRESP[1:0]
```

**ما تبحث عنه:**
- `BRESP = 2'b10` → SLVERR: الجهاز موجود لكن يرفض الطلب (عنوان خاطئ أو الجهاز معطّل)
- `BRESP = 2'b11` → DECERR: عنوان خارج نطاق أي decoder (خطأ في الترجمة)
- غياب `AWREADY` → interconnect لا يعرف العنوان
- `RDATA = 0xDEADBEEF` أو `0xFFFFFFFF` → عادةً ما يعني bus error

**إعداد Logic Analyzer:**
```
Trigger: AWADDR == <expected_address>
Capture: AWVALID, AWREADY, AWADDR, WDATA, RDATA, BRESP
Sample rate: >= 2x bus clock frequency
```

---

### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط log الـ kernel | السبب المحتمل |
|---------|-------------------|---------------|
| عنوان خاطئ في الـ DTS | `OF: no matching bus translation` | `ranges` لا يغطي العنوان المطلوب |
| قطعة الـ clock معطّلة | `ioremap` ينجح لكن قراءة الـ register تعطي `0xFFFFFFFF` | الجهاز لا يستجيب — افعّل الـ clock |
| مشكلة power domain | `of_iomap` ينجح ثم crash عند الوصول | الـ power domain لم يُفعَّل قبل الوصول |
| عنوان PCI BAR خاطئ | `pci_resource_start` لا يتطابق مع الـ DTS | `of_pci_range_parser_init` يحتاج `ranges` صحيحة |
| DMA يكتب خارج الـ buffer | سايلنت corruption أو `IOMMU fault` | `dma-ranges` لا يعكس قيود الـ DMA hardware |
| `#address-cells=<2>` على 32-bit SoC | `OF: Bad cell count` | الـ DTS يفترض 64-bit لكن الـ hardware 32-bit |

---

### 5. الـ Device Tree Debugging — التحقق من تطابق الـ DT مع الـ Hardware

#### فحص `ranges` property

```bash
# عرض ranges الـ raw بالـ hex
hexdump -C /sys/firmware/devicetree/base/soc/ranges 2>/dev/null
# أو
cat /sys/firmware/devicetree/base/soc/ranges | xxd

# مثال مخرجات (ranges بـ #address-cells=1, parent #address-cells=1, #size-cells=1):
# 00000000: 0000 0000 4000 0000 1000 0000  ....@...........
# التفسير: child_addr=0x00000000, parent_addr=0x40000000, size=0x10000000
```

#### التحقق من `reg` property

```bash
# قراءة reg لـ device معين
hexdump -C /sys/firmware/devicetree/base/soc/serial@1000000/reg
# 00000000: 0100 0000 0000 1000  ........
# التفسير: addr=0x01000000, size=0x00001000 (مع #address-cells=1, #size-cells=1)
```

#### أداة **`of_property_read_reg`** — اختبار برمجي

```c
/* في module مؤقت للاختبار */
u64 addr, size;
int ret;

ret = of_property_read_reg(np, 0, &addr, &size);
if (ret) {
    pr_err("of_property_read_reg failed: %d\n", ret);
} else {
    pr_info("reg[0]: addr=0x%llx, size=0x%llx\n", addr, size);
    /* قارن هذا مع ما في الـ datasheet */
}
```

#### فحص `dma-ranges`

```bash
# هل يوجد dma-ranges في الـ node؟
ls /sys/firmware/devicetree/base/soc/dma-ranges 2>/dev/null && \
    hexdump -C /sys/firmware/devicetree/base/soc/dma-ranges || \
    echo "No dma-ranges — DMA uses 1:1 mapping"

# فحص of_dma_is_coherent
# إذا كانت النتيجة 0 لكن الـ hardware coherent → مشكلة في DTS
# ابحث عن: dma-coherent; في الـ node
grep -r 'dma-coherent' /sys/firmware/devicetree/base/ 2>/dev/null | head -5
```

---

## الأوامر العملية الجاهزة للنسخ

### السيناريو 1: `of_iomap` يُعيد `NULL`

```bash
# الخطوة 1: تحقق من وجود node في الـ DT
ls /sys/firmware/devicetree/base/ | grep -i 'serial\|uart'

# الخطوة 2: تحقق من reg property
hexdump -C /sys/firmware/devicetree/base/soc/serial@1000000/reg 2>/dev/null

# الخطوة 3: تحقق من وجود ranges في الـ parent
hexdump -C /sys/firmware/devicetree/base/soc/ranges 2>/dev/null
ls /sys/firmware/devicetree/base/soc/ | grep ranges

# الخطوة 4: تحقق من /proc/iomem
cat /proc/iomem | grep '01000'

# الخطوة 5: فعّل dynamic debug
echo 'file drivers/of/address.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# ثم أعد تشغيل الـ driver وراقب dmesg
dmesg -w | grep -E 'OF:|of_'
```

---

### السيناريو 2: فشل ترجمة DMA address

```bash
# الخطوة 1: تحقق من dma-ranges
cat /sys/firmware/devicetree/base/soc/dma-ranges 2>/dev/null | xxd || \
    echo "WARNING: No dma-ranges found"

# الخطوة 2: فعّل IOMMU debug (إن وجد)
cat /sys/kernel/debug/iommu/devices 2>/dev/null | grep <device_name>

# الخطوة 3: تتبع دوال الترجمة
echo 'of_translate_dma_address of_translate_dma_region' > \
    /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ driver
cat /sys/kernel/debug/tracing/trace
```

---

### السيناريو 3: فحص PCI ranges

```bash
# الخطوة 1: عرض الـ DTS لـ PCI controller
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | \
    awk '/pcie@/{found=1} found{print; if(/\};/){found=0}}'

# الخطوة 2: تحقق من resources المترجَمة
cat /proc/iomem | grep -A 5 'PCI\|pci'

# الخطوة 3: فحص BAR specific
lspci -v -s 00:00.0 | grep 'Memory\|I/O'

# الخطوة 4: فحص of_pci_range_parser يدوياً عبر module
# أو استخدم:
cat /sys/bus/pci/devices/0000:00:00.0/resource
```

---

### السيناريو 4: تحليل شامل لـ node معين

```bash
#!/bin/bash
# سكريبت شامل لتحليل DT node
NODE_PATH="/sys/firmware/devicetree/base/soc/serial@1000000"

echo "=== Node Properties ==="
for f in "$NODE_PATH"/*; do
    name=$(basename "$f")
    echo -n "$name: "
    if file "$f" | grep -q 'ASCII'; then
        cat "$f"
    else
        hexdump -C "$f" | head -3
    fi
    echo
done

echo "=== Translated Address in /proc/iomem ==="
# استخراج العنوان من reg property
ADDR=$(hexdump -e '1/4 "%08x\n"' "$NODE_PATH/reg" 2>/dev/null | head -1)
cat /proc/iomem | grep "$ADDR" 2>/dev/null || echo "NOT FOUND in iomem"

echo "=== Parent ranges ==="
hexdump -C "$(dirname $NODE_PATH)/ranges" 2>/dev/null || echo "No ranges"
```

---

### نموذج مخرجات وكيفية تفسيرها

```
# مخرجات of_address debug مع dynamic_debug مفعّل:
[    2.345678] OF: of_translate_address: bus is 'simple-bus'
[    2.345679] OF: of_translate_address: source addr (00000000 01000000)
[    2.345680] OF: of_translate_address: ranges, nr_cells=3
[    2.345681] OF: of_translate_address: match: ( 00000000 40000000 10000000)
[    2.345682] OF: of_translate_address: found ( 00000000 41000000 00001000)
```

**التفسير:**
- السطر 1: الـ bus نوعه `simple-bus` — يدعم `ranges`
- السطر 2: العنوان المدخل من `reg` property = `0x01000000`
- السطر 3: الـ `ranges` تحتوي 3 خلايا لكل entry
- السطر 4: وجد تطابقاً — child `0x0` يُترجَم لـ parent `0x40000000` بحجم `0x10000000`
- السطر 5: العنوان المُترجَم النهائي = `0x41000000` (CPU physical address)

```
# مخرجات فاشلة — OF_BAD_ADDR:
[    2.345683] OF: of_translate_address: no ranges, cannot translate
# السبب: الـ parent node لا يملك ranges property
# الحل: أضف ranges; (للـ passthrough) أو ranges = <child parent size>;
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: بوابة صناعية على RK3562 — فشل `of_iomap` بصمت

#### العنوان
**فشل تعيين عنوان UART في بوابة صناعية على RK3562**

#### السياق
شركة تبني بوابة IoT صناعية تعتمد على معالج **RK3562** من Rockchip. الجهاز يحتوي على UART4 مخصص لاتصال RS-485 مع أجهزة قياس ميداني. المنتج في مرحلة bring-up الأولى، والمهندس يكتب driver مخصص للتحكم بالـ UART.

#### المشكلة
الـ driver يستدعي `of_iomap(np, 0)` لكنه يحصل على `NULL` في كل مرة، مما يتسبب في `NULL pointer dereference` عند محاولة الكتابة على السجلات.

```c
static int rs485_probe(struct platform_device *pdev)
{
    void __iomem *base;

    /* returns NULL — driver crashes here */
    base = of_iomap(pdev->dev.of_node, 0);
    if (!base) {
        dev_err(&pdev->dev, "of_iomap failed\n");
        return -ENOMEM;
    }
    writel(0x1, base + UART_LCR); /* crash if base is NULL */
    ...
}
```

#### التحليل
**الـ `of_iomap`** تعتمد داخليًا على `of_address_to_resource` ثم `ioremap`. السلسلة هي:

```
of_iomap(node, 0)
  └─► of_address_to_resource(node, 0, &res)
        └─► of_translate_address(node, addr_cells_raw)
              └─► تمشي شجرة DT صعودًا بحثًا عن "ranges"
```

المهندس فتح ملف DT وجد:

```dts
/* خطأ: لا يوجد #address-cells و #size-cells في العقدة الأب */
soc {
    uart4: serial@fe670000 {
        compatible = "rockchip,rk3562-uart";
        /* reg موجود لكن بدون #address-cells في الأب */
        reg = <0x0 0xfe670000 0x0 0x100>;
    };
};
```

**الجذر الحقيقي**: الـ `soc` node تفتقر إلى `#address-cells = <2>` و `#size-cells = <2>`، لذا عندما تحاول `of_translate_address` تفسير الـ `reg` property بـ **na** و **ns** خاطئين، تُرجع `OF_BAD_ADDR` وبالتالي `of_address_to_resource` تفشل وتُرجع `-EINVAL`، وبالتالي `of_iomap` تُرجع `NULL`.

#### الحل

**تصحيح DT:**
```dts
soc {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    uart4: serial@fe670000 {
        compatible = "rockchip,rk3562-uart", "snps,dw-apb-uart";
        reg = <0x0 0xfe670000 0x0 0x100>;
        interrupts = <GIC_SPI 118 IRQ_TYPE_LEVEL_HIGH>;
        clocks = <&cru SCLK_UART4>, <&cru PCLK_UART4>;
        clock-names = "baudclk", "apb_pclk";
        status = "okay";
    };
};
```

**أمر debug للتحقق قبل التعديل:**
```bash
# تحقق من عدد address-cells في الأب
fdtget /boot/dtb.dtb /soc '#address-cells'
fdtget /boot/dtb.dtb /soc '#size-cells'

# تحقق من ترجمة العنوان
cat /sys/bus/platform/devices/fe670000.serial/resources
```

#### الدرس المستفاد
**الـ `of_iomap`** تصمت عند الفشل وتُرجع `NULL` دون تفاصيل. يجب دائمًا استخدام `of_io_request_and_map` بدلًا منها لأنها تُسجّل الخطأ وتُظهر السبب. كذلك يجب التحقق من `#address-cells` و `#size-cells` في كل عقدة أب عند bring-up جهاز جديد.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — DMA يكتب على ذاكرة خاطئة

#### العنوان
**`dma-ranges` مفقود يتسبب في تلف البيانات في TV Box على H616**

#### السياق
منتج Android TV Box يعتمد على معالج **Allwinner H616**. المنتج يدعم تشغيل 4K عبر HDMI. أثناء اختبارات الجودة، يظهر تلف عشوائي في الصورة عند تشغيل محتوى 4K مع نقل USB في الوقت ذاته.

#### المشكلة
محرك USB xHCI يستخدم DMA لنقل البيانات، لكن العناوين المحسوبة خاطئة. الـ DMA يكتب بيانات USB في منطقة ذاكرة لا تعود للـ buffer الصحيح.

#### التحليل
الكود في kernel يستدعي `of_translate_dma_address` لتحويل عنوان DMA من فضاء الجهاز إلى فضاء CPU:

```c
/* من kernel/drivers/usb/host/xhci-plat.c (مبسط) */
u64 dma_base = of_translate_dma_address(np, dma_addr_be32);
```

**الـ `of_translate_dma_address`** تعمل بنفس آلية `of_translate_address` لكنها تبحث في `dma-ranges` بدلًا من `ranges`. إذا لم تجد `dma-ranges`، تفترض أن عنوان DMA = عنوان CPU (mapping مباشر 1:1).

على H616، الـ USB controller عند عنوان فيزيائي `0x05200000`، وعنوان DMA الخاص به **ليس** identically mapped مع عنوان CPU بسبب IOMMU. الـ DT الخاطئ:

```dts
/* خطأ: لا يوجد dma-ranges */
usb3: usb@5200000 {
    compatible = "snps,dwc3";
    reg = <0x05200000 0x10000>;
    /* dma-ranges غائبة ! */
};
```

بدون `dma-ranges`، تُرجع `of_translate_dma_address` العنوان كما هو، فيحدث تلف الذاكرة.

**التحقق من المشكلة:**
```bash
# هل يوجد dma-ranges في عقدة USB؟
dtc -I fs /proc/device-tree | grep -A5 "usb@5200000"

# فحص عناوين DMA المستخدمة فعلًا
cat /sys/kernel/debug/dma-api/dump 2>/dev/null | grep usb
```

#### الحل

**تصحيح DT بإضافة `dma-ranges`:**
```dts
bus@5000000 {
    compatible = "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges = <0x05000000 0x05000000 0x02000000>;
    /* DMA offset: CPU addr = DMA addr + 0x40000000 على H616 */
    dma-ranges = <0x00000000 0x40000000 0xc0000000>;

    usb3: usb@5200000 {
        compatible = "snps,dwc3";
        reg = <0x05200000 0x10000>;
        dr_mode = "host";
        snps,dis_u2_susphy_quirk;
    };
};
```

**التحقق بعد التصحيح:**
```bash
# of_dma_is_coherent يعكس صحة الإعداد
cat /sys/bus/platform/devices/5200000.usb/of_node/dma-coherent
```

#### الدرس المستفاد
**الـ `of_translate_dma_address`** تفترض mapping مباشر إذا لم تجد `dma-ranges` — صمت خطير. في أي SoC يحتوي على IOMMU أو offset بين فضاء DMA وفضاء CPU، يجب التحقق من وجود `dma-ranges` صريحة في كل bus node.

---

### السيناريو الثالث: بورد STM32MP1 — `of_address_to_resource` يُرجع عنوانًا خاطئًا لـ SPI

#### العنوان
**تداخل resources بين SPI1 وSPI2 على STM32MP1 بسبب خطأ في index**

#### السياق
مهندس يعمل على بورد مخصص يعتمد على **STM32MP157** من STMicroelectronics. البورد تحتوي على مستشعرات IMU متعددة تتصل عبر SPI. المهندس يضيف driver لقراءة مستشعر ICM-42688 عبر SPI1.

#### المشكلة
الـ driver يقرأ بيانات خاطئة تمامًا. عند تتبع المشكلة، يتضح أن الـ driver يُعيّن registers SPI2 بدلًا من SPI1.

#### التحليل
الكود المشكل:

```c
static int icm42688_probe(struct spi_device *spi)
{
    struct resource res;
    int ret;

    /* خطأ: index=1 يشير للـ reg الثاني، لكن المقصود هو الأول */
    ret = of_address_to_resource(spi->dev.of_node, 1, &res);
    if (ret)
        return ret;

    priv->base = ioremap(res.start, resource_size(&res));
    ...
}
```

دالة `of_address_to_resource` تستدعي `__of_get_address(dev, index, -1, &size, &flags)` حيث `index` هو رقم إدخال الـ `reg` property. القيمة `1` تشير للإدخال الثاني (zero-indexed).

الـ DT للجهاز:
```dts
&spi1 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;

    icm42688@0 {
        compatible = "invensense,icm42688";
        reg = <0>;  /* SPI CS0 — هذا الـ reg الوحيد، index=0 */
        spi-max-frequency = <10000000>;
    };
};
```

عقدة SPI device لا تحتوي على `reg` متعدد. استخدام `index=1` يتجاوز نهاية الـ property ويرث بيانات من الذاكرة المجاورة، محدثًا undefined behavior.

**فحص سريع:**
```bash
# كم عدد reg entries في العقدة؟
fdtget /boot/stm32mp157-myboard.dtb \
  /soc/spi@40004000/icm42688@0 reg | wc -w

# استخدم of_address_count للتحقق البرمجي
```

```c
/* التحقق البرمجي الآمن */
int count = of_address_count(spi->dev.of_node);
dev_info(&spi->dev, "reg entries: %d\n", count);
```

#### الحل

```c
/* الإصلاح: استخدام index=0 للـ reg الأول والوحيد */
ret = of_address_to_resource(spi->dev.of_node, 0, &res);
```

أو الأفضل، استخدام `of_property_read_reg` الذي يتحقق من الحدود:
```c
u64 addr, size;
ret = of_property_read_reg(spi->dev.of_node, 0, &addr, &size);
if (ret) {
    dev_err(&spi->dev, "failed to read reg[0]: %d\n", ret);
    return ret;
}
```

#### الدرس المستفاد
**الـ `of_address_to_resource`** لا تتحقق من صحة الـ index مقابل حجم الـ `reg` property. استخدام `of_address_count` قبل الوصول، أو `of_property_read_reg` الذي يتضمن فحصًا داخليًا، يمنع هذا النوع من الأخطاء الصامتة.

---

### السيناريو الرابع: بورد AM62x — PCIe لا يُكتشف بسبب `ranges` ناقصة

#### العنوان
**عدم اكتشاف جهاز NVMe على AM62x بسبب غياب `ranges` في عقدة PCIe**

#### السياق
**Texas Instruments AM625** يُستخدم في بوابة صناعية تحتاج تخزين محلي عالي السرعة. المهندس يُضيف SSD NVMe عبر PCIe Gen2. الـ kernel يُشغّل driver `pcie-j721e` لكن الجهاز لا يظهر في `lspci`.

#### المشكلة
الـ PCIe driver يستخدم `of_pci_range_parser_init` و `for_each_of_range` لاستخراج نطاقات الذاكرة والـ I/O التي يُمررها للجسر (bridge). إذا فشل التحليل، لا تُخصص موارد للأجهزة المتصلة.

#### التحليل
```c
/* من drivers/pci/controller/cadence/pcie-cadence-host.c (مبسط) */
static int cdns_pcie_host_setup(struct cdns_pcie_rc *rc)
{
    struct of_pci_range_parser parser;
    struct of_pci_range range;
    struct device_node *np = dev->of_node;

    /* تهيئة parser من "ranges" property */
    if (of_pci_range_parser_init(&parser, np)) {
        dev_err(dev, "missing ranges property\n");
        return -EINVAL;
    }

    /* تكرار على كل range لتعيين موارد */
    for_each_of_pci_range(&parser, &range) {
        /* تحويل pci_addr → cpu_addr */
        of_pci_range_to_resource(&range, np, &res);
        ...
    }
}
```

**الـ `of_pci_range_parser_init`** تبحث عن property اسمها `ranges` في الـ node. إذا لم تجدها أو كانت فارغة، تُرجع خطأ.

الـ DT المعطوب:
```dts
pcie0: pcie@f102000 {
    compatible = "ti,am625-pcie-host";
    reg = <0x00 0x0f102000 0x00 0x1000>,
          <0x00 0x0f100000 0x00 0x400>;
    reg-names = "intd_cfg", "user_cfg";
    /* خطأ فادح: ranges غائبة! */
    #address-cells = <3>;
    #size-cells = <2>;
    device_type = "pci";
};
```

**فحص المشكلة:**
```bash
# هل توجد ranges؟
fdtget /boot/am625-gateway.dtb /pcie@f102000 ranges

# فحص parser من الداخل عبر dynamic_debug
echo "file pcie-cadence-host.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep -i "pcie\|ranges"

# of_range_count يُعيد 0 إذا لم تجد parser أي entries
```

#### الحل

**تصحيح DT بإضافة `ranges` صحيحة لـ AM62x:**
```dts
pcie0: pcie@f102000 {
    compatible = "ti,am625-pcie-host";
    reg = <0x00 0x0f102000 0x00 0x1000>,
          <0x00 0x0f100000 0x00 0x400>;
    reg-names = "intd_cfg", "user_cfg";
    #address-cells = <3>;
    #size-cells = <2>;
    device_type = "pci";
    /*
     * ranges format:
     * <pci-space pci-addr-hi pci-addr-lo cpu-addr-hi cpu-addr-lo size-hi size-lo>
     * 0x02000000 = 32-bit memory space
     */
    ranges = <0x02000000 0x0 0x60000000  0x0 0x60000000  0x0 0x08000000>,
             <0x01000000 0x0 0x00001000  0x0 0x68000000  0x0 0x00010000>;
    bus-range = <0x0 0xff>;
    ...
};
```

**التحقق بعد التصحيح:**
```bash
lspci -tv
cat /proc/iomem | grep -i pci
# يجب أن تظهر نطاقات PCIe المخصصة
```

#### الدرس المستفاد
**الـ `of_pci_range_parser_init`** و `for_each_of_range` هي الآلية الوحيدة التي يعرف بها الـ kernel كيف يُرسم فضاء PCIe على فضاء CPU. غياب `ranges` يُصمت الكشف عن كل الأجهزة المتصلة بالـ PCIe bus. يجب دومًا التحقق من `of_range_count` بعد التهيئة للتأكد من وجود entries فعلية.

---

### السيناريو الخامس: ECU سيارة على i.MX8QM — `of_dma_is_coherent` يُرجع قيمة خاطئة يتسبب في deadlock

#### العنوان
**deadlock في نظام AUTOSAR على i.MX8QM بسبب خطأ في إعداد DMA coherency**

#### السياق
نظام **Automotive ECU** يعتمد على **NXP i.MX8QM** لتشغيل AUTOSAR Classic على Cortex-R واجهة مع Linux على Cortex-A. أحد الـ subsystems يستخدم I2C للتواصل مع مستشعرات درجة الحرارة والضغط. أثناء اختبارات load عالي، يتجمد النظام بشكل عشوائي بعد 2-3 ساعات من التشغيل.

#### المشكلة
الـ I2C DMA driver يتصرف بشكل خاطئ مع cache. في الأنظمة التي يكون فيها DMA coherent (مشارك Cache)، لا داعي لـ `cache flush` يدوي. لكن إذا أخطأ الـ driver في تحديد هذا، يحدث إما:
- **double flush**: هدر وقت.
- **no flush عند الحاجة**: قراءة بيانات stale من Cache بدلًا من الـ RAM، مما يُسبب تلف البيانات والـ deadlock في بروتوكولات synchronization.

#### التحليل
الـ driver يستدعي `of_dma_is_coherent`:

```c
/* من drivers/i2c/busses/i2c-imx.c (مبسط) */
static int i2c_imx_probe(struct platform_device *pdev)
{
    struct imx_i2c_struct *i2c_imx;
    bool dma_coherent;

    /* هل الـ DMA coherent مع CPU cache? */
    dma_coherent = of_dma_is_coherent(pdev->dev.of_node);

    if (dma_coherent) {
        /* لا حاجة لـ sync manual */
        i2c_imx->dma_ops = &coherent_dma_ops;
    } else {
        /* يجب عمل cache flush قبل/بعد كل DMA transfer */
        i2c_imx->dma_ops = &non_coherent_dma_ops;
    }
}
```

**الـ `of_dma_is_coherent`** تبحث عن property `dma-coherent` في العقدة أو أسلافها. إذا وجدتها، تُرجع `true`.

على i.MX8QM، الـ I2C controller ليس DMA-coherent بشكل افتراضي — يجب إضافة `dma-coherent` فقط إذا أُضيف SMMU/IOMMU في pipeline. الـ DT الخاطئ:

```dts
/* خطأ: dma-coherent في soc node تشمل كل الأجهزة */
soc@0 {
    compatible = "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;
    dma-coherent; /* ← هذا يجعل of_dma_is_coherent ترجع true لكل child */
    ranges;

    i2c0: i2c@56246000 {
        compatible = "fsl,imx8qm-lpi2c";
        reg = <0x56246000 0x1000>;
        /* I2C هذا ليس DMA-coherent فعلًا */
    };
};
```

**الـ `of_dma_is_coherent`** تُعيد `true` لأن الـ parent (`soc@0`) يحمل `dma-coherent`. الـ driver يتجاهل الـ cache flush، فتقرأ الـ CPU بيانات stale.

**أدوات debug:**
```bash
# هل العقدة أو أسلافها تحمل dma-coherent؟
fdtget -t s /boot/imx8qm-ecu.dtb /soc/i2c@56246000 dma-coherent 2>&1
fdtget -t s /boot/imx8qm-ecu.dtb /soc dma-coherent 2>&1

# فحص من الـ sysfs
cat /sys/bus/platform/devices/56246000.i2c/of_node/dma-coherent 2>/dev/null \
  || echo "not coherent"

# فحص coherency من الـ DMA API
cat /sys/kernel/debug/dma-api/dma-coherent 2>/dev/null
```

#### الحل

**إزالة `dma-coherent` من `soc` node وإضافتها فقط للأجهزة التي تدعمها فعلًا:**

```dts
soc@0 {
    compatible = "simple-bus";
    #address-cells = <1>;
    #size-cells = <1>;
    /* حُذفت dma-coherent من هنا */
    ranges;

    /* GPU يدعم DMA coherency عبر SMMU */
    gpu: gpu@53000000 {
        compatible = "vivante,gc";
        reg = <0x53000000 0x10000>;
        dma-coherent; /* ← صحيح: GPU + SMMU = coherent */
    };

    /* I2C لا يدعمها */
    i2c0: i2c@56246000 {
        compatible = "fsl,imx8qm-lpi2c";
        reg = <0x56246000 0x1000>;
        /* بدون dma-coherent — of_dma_is_coherent ستُرجع false */
    };
};
```

**فحص برمجي للتأكد:**
```c
/* في init code أو في probe للتحقق */
if (of_dma_is_coherent(pdev->dev.of_node))
    dev_info(&pdev->dev, "DMA: coherent mode\n");
else
    dev_info(&pdev->dev, "DMA: non-coherent, cache sync required\n");
```

#### الدرس المستفاد
**الـ `of_dma_is_coherent`** ترث الـ property من أي عقدة أب في الشجرة — وضع `dma-coherent` في عقدة `soc` يُطبّقها على جميع الأجهزة الفرعية حتى التي لا تدعمها. في الأنظمة المدمجة الحرجة كـ ECU، خطأ الـ cache coherency لا يُسبب crash فوريًا بل deadlock متأخر يصعب تشخيصه. يجب دومًا إضافة `dma-coherent` بشكل انتقائي لكل جهاز وليس للـ bus الكاملة.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأول لمتابعة تطور kernel Linux، وفيما يلي أهم المقالات المتعلقة بـ `of_address` وترجمة العناوين في device tree:

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| Platform devices and device trees | [lwn.net/Articles/448502](https://lwn.net/Articles/448502/) | يشرح كيف تتكامل platform devices مع device tree وكيف تُستخدم `of_address_to_resource()` |
| Device trees I: Are we having fun yet? | [lwn.net/Articles/572692](https://lwn.net/Articles/572692/) | تحليل عميق لمشاكل التصميم في device tree ونظام عناوين الـ `reg` و `ranges` |
| Device tree overlays | [lwn.net/Articles/616859](https://lwn.net/Articles/616859/) | يغطي الـ overlays الديناميكية وعلاقتها بترجمة العناوين |
| of: Add generic device tree DMA helpers | [lwn.net/Articles/487197](https://lwn.net/Articles/487197/) | يتناول إضافة دوال `of_translate_dma_address()` وآلية `dma-ranges` |
| Dynamic devices and static configuration | [lwn.net/Articles/435894](https://lwn.net/Articles/435894/) | يناقش كيف يحل device tree مشكلة التهيئة الثابتة للعناوين |
| Device tree troubles | [lwn.net/Articles/560523](https://lwn.net/Articles/560523/) | نقاش مهم حول مشاكل تنفيذ device tree وترجمة العناوين على architectures مختلفة |
| ioremap() and memremap() | [lwn.net/Articles/653585](https://lwn.net/Articles/653585/) | يشرح آليات mapping الذاكرة التي تعتمد عليها `of_iomap()` |
| Managed device resources | [lwn.net/Articles/215861](https://lwn.net/Articles/215861/) | يغطي devres API الذي يُستخدم مع `of_io_request_and_map()` |
| ELCE: Grant Likely on device trees | [lwn.net/Articles/414016](https://lwn.net/Articles/414016/) | محاضرة المطوّر الرئيسي لـ device tree توضح فلسفة تصميم نظام العناوين |
| CMA & device tree, once again | [lwn.net/Articles/605348](https://lwn.net/Articles/605348/) | يناقش reserved memory في device tree وعلاقتها بـ DMA address translation |

---

### توثيق kernel الرسمي

الملفات التالية موجودة داخل شجرة المصدر `Documentation/`:

```
Documentation/devicetree/usage-model.rst
    — النموذج العام لاستخدام device tree، يشمل #address-cells و #size-cells

Documentation/devicetree/bindings/
    — bindings رسمية لكل جهاز، مع أمثلة على خاصية reg و ranges

Documentation/core-api/dma-api.rst
    — واجهة DMA الكاملة، تشرح علاقة of_translate_dma_address() بالـ DMA mapping

Documentation/driver-api/device-io.rst
    — يشرح ioremap وما يعتمد عليه of_iomap()

Documentation/driver-api/driver-model/devres.rst
    — الـ managed resources المستخدمة مع of_io_request_and_map()
```

**الـ** kernel documentation أون-لاين:
- [docs.kernel.org/devicetree/usage-model.html](https://docs.kernel.org/devicetree/usage-model.html)
- [docs.kernel.org/devicetree/kernel-api.html](https://docs.kernel.org/devicetree/kernel-api.html)

---

### ملفات المصدر الرئيسية في kernel

الكود الفعلي لدوال `of_address.h` موزّع على:

```
include/linux/of_address.h      — الـ header الذي ندرسه (declarations + inline functions)
drivers/of/address.c            — التنفيذ الكامل لof_translate_address(), of_iomap(), ...
drivers/of/base.c               — دوال مساعدة مشتركة لـ device tree
```

للاطلاع على تاريخ التغييرات:

```bash
# تاريخ ملف التنفيذ الكامل
git log --oneline -- drivers/of/address.c

# أهم commit أضاف دعم dma-ranges
git log --oneline --all --grep="dma-ranges"

# البحث عن of_pci_range
git log --oneline --all --grep="of_pci_range"
```

**الـ** commits الأبرز التي شكّلت هذا الـ subsystem:

| الوصف | الملاحظة |
|-------|---------|
| إضافة `of_translate_address()` الأصلية | Grant Likely، مشروع PowerPC الأصلي |
| نقل `drivers/of/address.c` من arch/powerpc | توحيد الكود لجميع architectures |
| إضافة `of_pci_range_parser` | دعم PCI host bridges |
| إضافة `of_translate_dma_region()` | دعم DMA ranges المجزأة |

يمكن البحث عن الـ commits بالرابط:
`https://github.com/torvalds/linux/commits/master/drivers/of/address.c`

---

### نقاشات mailing list

- **[PATCH] Device Tree on ARM platform** — [lwn.net/Articles/334826](https://lwn.net/Articles/334826/) — النقاش الأصلي لنقل device tree إلى ARM، يتضمن مناقشة ترجمة العناوين
- **address translation for PCIe-to-localbus bridge** — [lists.infradead.org](http://lists.infradead.org/pipermail/linux-arm-kernel/2013-November/209484.html) — نقاش تقني حول `of_pci_range_parser` على linux-arm-kernel
- **ranges property in PCI RC node** — [lists.kernelnewbies.org](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2023-February/022848.html) — شرح عملي لخاصية `ranges` في PCIe root complex
- **What does OS do with ranges in PCI RC node** — [lists.kernelnewbies.org](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2022-November/022774.html) — توضيح كيف يبني الـ kernel خريطة العناوين من `ranges`

للبحث المباشر في الـ mailing lists:
- [lore.kernel.org/devicetree](https://lore.kernel.org/devicetree/) — الأرشيف الرسمي لقائمة devicetree
- [lore.kernel.org/linux-arm-kernel](https://lore.kernel.org/linux-arm-kernel/) — نقاشات ARM مع device tree

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)

```
الفصل الثامن: Allocating Memory
    — يشرح ioremap() الذي تبنى عليه of_iomap()

الفصل التاسع: Communicating with Hardware
    — يغطي I/O ports وI/O memory وعلاقتها بـ resource management

متوفر مجاناً: https://lwn.net/Kernel/LDD3/
```

#### Linux Kernel Development — Robert Love (3rd Edition)

```
الفصل الثاني عشر: Memory Management
    — أساس فهم physical/virtual address translation

الفصل التاسع عشر: Portability
    — يناقش big-endian (__be32) وأهميتها في of_address
```

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

```
الفصل السادس: System Initialization
    — يشرح كيف يقرأ bootloader وkernel الـ device tree

الفصل التاسع: File Systems
    — يتضمن أمثلة عملية على device tree bindings مع عناوين حقيقية
```

#### مراجع إضافية مفيدة

- **Mastering Embedded Linux Programming** — Frank Vasquez & Chris Simmonds
  - يغطي device tree بشكل مكثف مع أمثلة عملية على `reg` و `ranges`

- **Professional Linux Kernel Architecture** — Wolfgang Mauerer
  - تحليل عميق لـ memory management وكيف يندمج معه device tree

---

### مصادر eLinux.org

| الصفحة | الرابط | المحتوى |
|--------|--------|---------|
| Device Tree Reference | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لخصائص DTS بما فيها reg وranges وdma-ranges |
| Device Tree Mysteries | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) | يحل غموض ترجمة العناوين وحسابات #address-cells |
| Linux Drivers Device Tree Guide | [elinux.org/Linux_Drivers_Device_Tree_Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) | دليل عملي لكتابة drivers تستخدم of_address API |
| Device Trees | [elinux.org/Device_Trees](https://elinux.org/Device_Trees) | نقطة البداية لكل موارد device tree على elinux |

---

### مصادر kernelnewbies.org

- [kernelnewbies.org/Linux_3.18-DriversArch](https://kernelnewbies.org/Linux_3.18-DriversArch) — تغييرات of_address في kernel 3.18
- [kernelnewbies.org/Linux_4.0-DriversArch](https://kernelnewbies.org/Linux_4.0-DriversArch) — تغييرات of_address في kernel 4.0
- [kernelnewbies.org/Linux_4.4-DriversArch](https://kernelnewbies.org/Linux_4.4-DriversArch) — إضافة of_dma_is_coherent وتحسينات of_pci_range
- [kernelnewbies.org/KernelGlossary](https://kernelnewbies.org/KernelGlossary) — مسرد المصطلحات: device tree، DMA، MMIO، iomap

---

### مصادر خارجية أخرى

- **Device Tree for Dummies** — Thomas Petazzoni (Free Electrons):
  [events.static.linuxfound.org/slides/petazzoni-device-tree-dummies.pdf](https://events.static.linuxfound.org/sites/events/files/slides/petazzoni-device-tree-dummies.pdf)
  — أفضل مقدمة عملية، تشرح #address-cells و ranges بأمثلة مرئية

- **Devicetree Specification** (ذات الصلة بـ of_address):
  [devicetree-specification.readthedocs.io](https://devicetree-specification.readthedocs.io/en/stable/)
  - الفصل 2.3: Node Names — تسمية العقد بالعناوين
  - الفصل 2.3.8: `#address-cells` و `#size-cells`
  - الفصل 2.3.9: خاصية `reg`
  - الفصل 2.5: ترجمة العناوين عبر `ranges`

---

### كلمات البحث للاستزادة

للبحث في Google أو lore.kernel.org أو GitHub:

```
# للبحث عن التنفيذ الكامل
"drivers/of/address.c" linux kernel

# لفهم ترجمة العناوين
"of_translate_address" OR "of_address_to_resource" device tree kernel

# لفهم PCI ranges
"of_pci_range_parser" linux kernel

# لفهم DMA translation
"dma-ranges" "of_translate_dma_address" linux kernel

# لنقاشات mailing list
site:lore.kernel.org "of_address" OR "dma-ranges" device tree

# للـ commits ذات الصلة
site:github.com torvalds/linux "of_pci_range" OR "of_translate_dma"

# للـ bindings الرسمية
site:kernel.org Documentation devicetree bindings ranges
```
## Phase 8: Writing simple module

### الدالة المختارة للـ hook: `of_address_to_resource`

**الـ `of_address_to_resource`** هي الدالة الأكثر استخداماً في `of_address.h` — تُحوِّل عنوان `reg` من Device Tree إلى `struct resource` جاهز للاستخدام من قِبَل drivers. كل driver يستدعيها عند الـ probe لمعرفة نطاق الذاكرة أو I/O الخاص بجهازه، مما يجعل الـ kprobe عليها ذا قيمة تشخيصية عالية.

---

### الكود الكامل للـ module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * of_addr_kprobe.c
 *
 * Attaches a kprobe to of_address_to_resource() and logs
 * which device-tree node + resource index is being resolved,
 * along with the resulting physical address range when the
 * function returns successfully.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit       */
#include <linux/kernel.h>      /* pr_info, pr_err                         */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, ...     */
#include <linux/of.h>          /* struct device_node, of_node_full_name() */
#include <linux/ioport.h>      /* struct resource, IORESOURCE_MEM         */
#include <linux/stringify.h>   /* __stringify                             */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs <example@kernel.org>");
MODULE_DESCRIPTION("kprobe on of_address_to_resource() to trace DT address resolution");

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler: called just BEFORE of_address_to_resource()   */
/* ------------------------------------------------------------------ */

/*
 * The prototype of the hooked function is:
 *   int of_address_to_resource(struct device_node *dev,
 *                              int index,
 *                              struct resource *r);
 *
 * On x86_64  : args in RDI, RSI, RDX  → regs->di, regs->si, regs->dx
 * On ARM64   : args in X0,  X1,  X2   → regs->regs[0..2]
 * We use regs_get_kernel_argument() which is arch-agnostic since 5.8.
 */
static int pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    struct device_node *dn;
    int                 index;

    /* Extract first two arguments regardless of architecture */
    dn    = (struct device_node *)regs_get_kernel_argument(regs, 0);
    index = (int)               regs_get_kernel_argument(regs, 1);

    if (dn)
        pr_info("[of_addr_kprobe] resolving: node=%-32s  reg-index=%d\n",
                of_node_full_name(dn), index);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/*  kretprobe handler: called just AFTER of_address_to_resource()     */
/* ------------------------------------------------------------------ */

/*
 * We use a kretprobe so we can also log the *result* — the physical
 * address range that the kernel resolved from the DT "reg" property.
 */
struct res_data {
    struct resource *rp; /* pointer to the caller's struct resource */
};

static int ret_entry(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    struct res_data *d = (struct res_data *)ri->data;

    /* Third argument is the pointer to struct resource */
    d->rp = (struct resource *)regs_get_kernel_argument(regs, 2);
    return 0;
}

static int ret_handler(struct kretprobe_instance *ri, struct pt_regs *regs)
{
    struct res_data *d   = (struct res_data *)ri->data;
    long             ret = regs_return_value(regs); /* return value of hooked fn */

    if (ret == 0 && d->rp) {
        /* Success — log the resolved physical resource */
        pr_info("[of_addr_kprobe] resolved : name=%-24s  start=0x%llx  end=0x%llx  flags=0x%lx\n",
                d->rp->name ? d->rp->name : "(null)",
                (unsigned long long)d->rp->start,
                (unsigned long long)d->rp->end,
                d->rp->flags);
    } else if (ret != 0) {
        pr_info("[of_addr_kprobe] failed   : ret=%ld\n", ret);
    }

    return 0;
}

/* ------------------------------------------------------------------ */
/*  kprobe instance (pre-handler only, for the entry log)             */
/* ------------------------------------------------------------------ */

static struct kprobe entry_kp = {
    .symbol_name = "of_address_to_resource",
    .pre_handler = pre_handler,
};

/* ------------------------------------------------------------------ */
/*  kretprobe instance (return-value + resource log)                  */
/* ------------------------------------------------------------------ */

static struct kretprobe ret_kp = {
    .kp.symbol_name = "of_address_to_resource",
    .entry_handler  = ret_entry,   /* save args before call  */
    .handler        = ret_handler, /* inspect result after   */
    .data_size      = sizeof(struct res_data),
    .maxactive      = 20, /* max concurrent instances (20 CPUs probing at once) */
};

/* ------------------------------------------------------------------ */
/*  module_init / module_exit                                         */
/* ------------------------------------------------------------------ */

static int __init of_addr_probe_init(void)
{
    int ret;

    /* Register the entry kprobe first */
    ret = register_kprobe(&entry_kp);
    if (ret < 0) {
        pr_err("[of_addr_kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    /* Register the return kprobe */
    ret = register_kretprobe(&ret_kp);
    if (ret < 0) {
        pr_err("[of_addr_kprobe] register_kretprobe failed: %d\n", ret);
        unregister_kprobe(&entry_kp); /* clean up what we already registered */
        return ret;
    }

    pr_info("[of_addr_kprobe] loaded — watching of_address_to_resource()\n");
    return 0;
}

static void __exit of_addr_probe_exit(void)
{
    /*
     * Must unregister both probes before the module's text is removed.
     * Failure to do so causes the kernel to call into freed memory.
     */
    unregister_kretprobe(&ret_kp);
    unregister_kprobe(&entry_kp);
    pr_info("[of_addr_kprobe] unloaded\n");
}

module_init(of_addr_probe_init);
module_exit(of_addr_probe_exit);
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يُعرِّف `struct kprobe`، `struct kretprobe`، وكل دوال التسجيل والإلغاء |
| `linux/of.h` | يُعرِّف `struct device_node` و`of_node_full_name()` لطباعة اسم العقدة |
| `linux/ioport.h` | يُعرِّف `struct resource` وماكروهات `IORESOURCE_*` لتفسير الـ flags |

---

#### الـ `pre_handler`

**الـ `regs_get_kernel_argument(regs, N)`** دالة portable تسحب الـ argument رقم N من السجلات بغض النظر عن المعمارية (x86_64، ARM64، RISC-V). نطبع اسم عقدة DT ورقم الـ `reg` المطلوب قبل تنفيذ الدالة الأصلية لمعرفة *من* يطلب الترجمة.

---

#### الـ `ret_entry` و`ret_handler`

الـ `ret_entry` يُحفظ مؤشر `struct resource` من الـ argument الثالث في `ri->data` قبل دخول الدالة — لأن بعد العودة لا تُضمن قيم السجلات. الـ `ret_handler` يُقرأ نتيجة الدالة عبر `regs_return_value()` ثم يطبع `start`/`end`/`flags` من الـ resource إذا نجحت العملية.

---

#### الـ `maxactive = 20`

**الـ `maxactive`** يحدد عدد الاستدعاءات المتزامنة التي يمكن للـ kretprobe تتبعها دفعة واحدة. في أنظمة SMP مع devices كثيرة تجري الـ probe بالتوازي، قيمة صغيرة كـ 1 ستفقد بعض الأحداث.

---

#### `module_init` / `module_exit`

يسجِّل الـ `init` الـ kprobe ثم الـ kretprobe بالترتيب، ويتراجع نظيفاً إن فشل أي منهما. الـ `exit` يُلغي كليهما بالترتيب العكسي قبل أن يُزيل الـ kernel نص الموديول من الذاكرة — الإخفاق في ذلك يؤدي إلى **kernel panic** من الاستدعاء إلى عنوان محرَّر.

---

### ملف `Makefile` للبناء

```makefile
obj-m += of_addr_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الموديول
make
sudo insmod of_addr_kprobe.ko

# مراقبة الـ log في الوقت الفعلي
sudo dmesg -w | grep of_addr_kprobe

# إزالة الموديول
sudo rmmod of_addr_kprobe
```

---

### مثال على المخرجات المتوقعة

```
[of_addr_kprobe] loaded — watching of_address_to_resource()
[of_addr_kprobe] resolving: node=/soc/serial@9000000              reg-index=0
[of_addr_kprobe] resolved : name=9000000.serial              start=0x9000000  end=0x90000ff  flags=0x200
[of_addr_kprobe] resolving: node=/soc/ethernet@4000000            reg-index=0
[of_addr_kprobe] resolved : name=4000000.ethernet            start=0x4000000  end=0x4003fff  flags=0x200
```

**الـ `flags=0x200`** تعني `IORESOURCE_MEM` — أي منطقة ذاكرة ممتدة تم تعريفها في خاصية `reg` بـ Device Tree.
