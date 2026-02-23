## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `include/linux/of.h`** ينتمي إلى subsystem رسمي اسمه في MAINTAINERS:
**OPEN FIRMWARE AND FLATTENED DEVICE TREE**
المشرفون: Rob Herring و Saravana Kannan — القائمة البريدية: `devicetree@vger.kernel.org`

---

### القصة — لماذا وُجد هذا الملف؟

تخيّل أنك تشتري روبوتاً جديداً. الروبوت لديه يدان، وعينان، وموتور في الصدر، ومستشعر حرارة في القدم. من يُخبر الدماغ الإلكتروني (الـ kernel) بكل هذه التفاصيل؟

في حواسيب x86 التقليدية، الـ BIOS/UEFI يعرف الأجهزة ويُخبر الـ kernel. لكن في عالم الـ embedded systems — أجهزة ARM، RISC-V، PowerPC، Raspberry Pi، أجهزة الشبكات، السيارات، الهواتف — لا يوجد BIOS. **لا أحد يُخبر الـ kernel بما هو موصول**.

الحل القديم كان أن يكتب كل مصنّع kernel code مخصصاً يعرّف الأجهزة بشكل صريح (hardcoded). النتيجة: kernel مليء بـ `board files` بالآلاف، كل ملف يصف ماكينة واحدة. كان هذا كارثة — Linus Torvalds نفسه وصف هذا بأنه "complete and utter mess".

**الحل: Device Tree.**

فكرة Device Tree مأخوذة أصلاً من **Open Firmware** — نظام firmware اخترعته Sun Microsystems وApple في التسعينيات — ثم تطوّرت لتصبح مستقلة تماماً (Flattened Device Tree / FDT).

---

### ما هو الـ Device Tree بالضبط؟

الـ **Device Tree** هو **ملف نصي يصف الأجهزة المادية** للـ kernel. يكتبه مهندس الـ hardware، ويحمله الـ bootloader (مثل U-Boot) إلى الـ kernel عند الإقلاع. الـ kernel يقرأه ويعرف:

- ما هي الـ CPUs وكم عددها وأين في الذاكرة؟
- ما هي الـ buses (I2C، SPI، UART)؟
- ما هي الأجهزة المتصلة بكل bus؟
- ما هي عناوين الذاكرة (memory-mapped registers)؟
- ما هي الـ interrupts؟
- ما هي الـ clocks والـ power domains؟

مثال لجزء من ملف Device Tree (`.dts`):

```c
// Example: Raspberry Pi UART node in Device Tree
uart0: serial@7e201000 {
    compatible = "brcm,bcm2835-aux-uart"; // driver name to bind
    reg = <0x7e201000 0x40>;              // register base address + size
    interrupts = <1 29>;                   // interrupt line
    clocks = <&aux_clk>;                   // clock reference
    status = "okay";                       // enabled
};
```

---

### دور `of.h` تحديداً

**الـ `of.h`** هو **الـ header الرئيسي** لكل الـ kernel code الذي يتعامل مع Device Tree. يُعرّف:

#### 1. الـ Data Structures الأساسية

| Structure | الدور |
|-----------|-------|
| `struct device_node` | يمثّل **node واحدة** في شجرة الأجهزة — كل جهاز هو node |
| `struct property` | تمثّل **خاصية واحدة** داخل node (مثل `compatible`, `reg`, `interrupts`) |
| `struct of_phandle_args` | تمثّل **مرجعاً** من node إلى node أخرى مع arguments |
| `struct of_phandle_iterator` | iterator للتنقل في قوائم الـ phandles |
| `struct of_changeset` | مجموعة تعديلات atomic على الشجرة (للـ overlays) |

#### 2. الـ API الكاملة للقراءة

```c
// البحث عن node
of_find_node_by_name()
of_find_node_by_path()
of_find_compatible_node()
of_find_node_by_phandle()

// قراءة الخصائص
of_property_read_u32()       // قراءة u32 واحدة
of_property_read_u32_array() // قراءة مصفوفة u32
of_property_read_string()    // قراءة string
of_property_read_bool()      // هل الخاصية موجودة؟

// التنقل في الشجرة
of_get_parent()
of_get_next_child()
of_get_next_available_child()
```

#### 3. الـ Macros للتكرار (Iteration)

```c
// مثال: المرور على كل أبناء node معينة
for_each_child_of_node(parent, child) {
    /* process each child device */
}

// مثال: المرور على كل nodes تحمل compatible string معين
for_each_compatible_node(dn, NULL, "arm,pl011") {
    /* found a PL011 UART */
}
```

#### 4. الـ phandle System — قلب التوصيلات

الـ **phandle** هو رقم فريد يُعطى لكل node، يسمح لـ nodes أخرى بالإشارة إليها. مثلاً: الـ UART تقول "أنا أستخدم الـ clock رقم 5" — الـ kernel يترجم هذا الرقم إلى الـ `device_node` الخاص بالـ clock controller.

```c
// قراءة phandle وتحليل arguments
of_parse_phandle_with_args(np, "clocks", "#clock-cells", 0, &args);
// args.np -> يشير إلى clock controller node
// args.args[0] -> رقم الـ clock داخل الـ controller
```

#### 5. الـ Dynamic Tree و Overlays

**الـ `of_changeset`** يتيح تعديل الشجرة أثناء التشغيل — هذا أساس **Device Tree Overlays** المستخدمة في Raspberry Pi لإضافة HATs وإكسسوارات دون إعادة تشغيل.

#### 6. الـ Dual Build Support

كل دالة موجودة مرتين:
- **نسخة حقيقية** عند تفعيل `CONFIG_OF` — تعمل فعلاً مع شجرة الأجهزة
- **نسخة stub** عند إيقاف `CONFIG_OF` — ترجع NULL أو 0 — تضمن compilation بلا أخطاء على أنظمة بلا DT (مثل x86)

---

### رحلة بيانات حقيقية من البداية للنهاية

```
Hardware Board
     |
     v
Bootloader (U-Boot)
  - يحمّل ملف .dtb (Device Tree Blob) إلى الذاكرة
  - يُعطي عنوانه للـ kernel كـ argument
     |
     v
Kernel Boot (of_fdt.h / fdt.c)
  - يقرأ الـ .dtb ويبني شجرة device_nodes في الذاكرة
  - of_root يشير إلى node الجذر "/"
     |
     v
Driver Initialization
  - الـ driver يبحث عن device_node بـ compatible string
  - يقرأ الخصائص: عنوان الذاكرة، الـ interrupts، الـ clocks
  - ينشئ الـ device ويربطه بالـ driver
     |
     v
Running System
  - الـ /proc/device-tree يعكس الشجرة الحية
  - Overlays يمكن إضافتها/إزالتها ديناميكياً
```

---

### الفائدة العملية

- **driver واحد يعمل على ملايين الأجهزة**: driver الـ `pl011` UART يعمل على أي SoC يُعرّف نفسه بـ `compatible = "arm,pl011"` — بغض النظر عن الشركة المصنّعة.
- **لا hardcoding**: الـ memory addresses والـ interrupts تأتي من الشجرة، لا من الـ code.
- **صيانة أسهل**: تغيير الـ hardware يتطلب تعديل ملف `.dts` فقط، لا الـ kernel code.

---

### الملفات المرتبطة — ما يجب قراءته أيضاً

#### الـ Headers الرئيسية (ضمن نفس الـ subsystem)

| الملف | الدور |
|-------|-------|
| `include/linux/of.h` | **هذا الملف** — الـ core API |
| `include/linux/of_address.h` | ترجمة عناوين الذاكرة من DT إلى physical addresses |
| `include/linux/of_irq.h` | ربط الـ interrupts المذكورة في DT بـ Linux IRQ numbers |
| `include/linux/of_platform.h` | إنشاء `platform_device` من nodes الشجرة |
| `include/linux/of_device.h` | ربط الـ device بـ DT node، قراءة match data |
| `include/linux/of_gpio.h` | قراءة GPIOs من الشجرة |
| `include/linux/of_clk.h` | دعم الـ clock providers |
| `include/linux/of_dma.h` | دعم الـ DMA controllers |
| `include/linux/of_fdt.h` | parsing الـ Flattened Device Tree blob مباشرة |
| `include/linux/of_graph.h` | العلاقات بين media nodes (cameras, displays) |
| `include/linux/of_net.h` | ربط network devices بـ DT |
| `include/linux/of_pci.h` | دعم PCI في DT |
| `include/linux/of_mdio.h` | دعم MDIO/PHY |
| `include/linux/of_reserved_mem.h` | Reserved memory regions |

#### الـ Core Implementation

| الملف | الدور |
|-------|-------|
| `drivers/of/base.c` | تنفيذ الـ core API (البحث، القراءة، التنقل) |
| `drivers/of/property.c` | تنفيذ قراءة الخصائص |
| `drivers/of/dynamic.c` | الـ dynamic tree، changesets، overlays |
| `drivers/of/overlay.c` | تطبيق وإزالة الـ DT overlays |
| `drivers/of/fdt.c` | تحويل الـ FDT blob إلى شجرة `device_node` |
| `drivers/of/address.c` | ترجمة عناوين الـ DT |
| `drivers/of/irq.c` | ربط الـ DT interrupts بـ Linux IRQ |
| `drivers/of/platform.c` | إنشاء platform devices من الشجرة |
| `drivers/of/of_private.h` | Internal structures يستخدمها الـ core فقط |

#### ملفات الـ Device Tree نفسها

| النوع | المسار | الوصف |
|-------|--------|--------|
| `.dts` | `arch/arm/boot/dts/` | ملفات المصدر للـ boards |
| `.dtsi` | نفس المسار | ملفات include مشتركة بين boards |
| `.dtb` | ناتج compilation | الـ binary المحمول للـ kernel |
| Bindings | `Documentation/devicetree/bindings/` | توثيق خصائص كل device |
## Phase 2: شرح الـ Device Tree / Open Firmware (OF) Framework

---

### المشكلة التي يحلّها هذا الـ Subsystem

في عالم الـ embedded Linux، الـ SoC (System on Chip) يحتوي على عشرات الـ peripherals المدمجة: UART، I2C، SPI، GPIO، clock controllers، interrupt controllers، وغيرها. على عكس الـ PCI الذي يدعم الـ self-discovery (الكيرنل يستطيع اكتشاف الأجهزة تلقائياً عبر config space)، هذه الأجهزة المدمجة **لا تُعرّف عن نفسها** — لا يوجد بروتوكول enumeration.

**المشكلة الكلاسيكية:** كيف يعرف الكيرنل ما هو موجود على هذا الـ SoC وكيف هو مُوصَّل؟

الحل القديم كان كتابة **board files** بلغة C مباشرةً في الكيرنل: ملفات مثل `arch/arm/mach-omap2/board-rx51.c` تصف كل قطعة hardware بـ C structs. النتيجة: آلاف الـ board files، كل board لها ملف منفصل، وكل تعديل hardware يستلزم إعادة compile الكيرنل. Linus Torvalds وصف هذا الوضع بأنه "a disaster".

---

### الحل: Device Tree

الـ **Device Tree** هو وصف للـ hardware بصيغة نصية منفصلة عن الكيرنل تُسمى **DTS (Device Tree Source)**، تُحوَّل إلى blob ثنائي يُسمى **DTB (Device Tree Blob)** يُمرَّر من الـ bootloader للكيرنل عند الإقلاع.

الـ **OF (Open Firmware) framework** — المُعرَّف في `include/linux/of.h` — هو طبقة الكيرنل المسؤولة عن:
1. **استقبال** الـ DTB من الـ bootloader وتحليله (parsing).
2. **بناء شجرة** من `struct device_node` في الذاكرة تمثّل كل node في الـ DTS.
3. **تقديم API** للـ drivers وبقية الكيرنل للاستعلام عن هذه الشجرة.
4. **ربط** الـ device nodes بالـ platform drivers المناسبة عبر خاصية `compatible`.

---

### المثيل الواقعي (Real-World Analogy)

تخيّل أن **المصنع** بنى مبنى مكاتب ضخم (الـ SoC). المبنى فيه:
- غرف (peripherals): UART، I2C، GPIO...
- أسلاك كهرباء (IRQ lines) بين الغرف
- أبواب (registers) بعناوين محددة
- لافتات على الأبواب (compatible strings)

الـ **architect** (مصمم الـ SoC) كتب **مخططات البناء** في ملف DTS — وصف شامل: "الغرفة رقم 3 (UART0) تبدأ من العنوان 0xFF010000 وطولها 0x100 بايت، تستخدم IRQ رقم 32."

الـ **bootloader** (عامل الاستقبال) يُسلّم هذه المخططات للكيرنل (المدير الجديد) عند بدء العمل.

الـ **OF framework** (قسم الموارد البشرية) يقرأ المخططات ويبني **دليلاً داخلياً** (`device_node` tree). عندما يأتي **driver** (موظف متخصص) ويقول "أنا أعمل مع UART من نوع `ns16550a`"، يتحقق قسم الموارد البشرية من الدليل ويقول: "الغرفة رقم 3 مكتوب عليها `ns16550a`، اذهب إليها."

تطابق التشبيه مع المفاهيم الفعلية:

| التشبيه | المفهوم الفعلي |
|---|---|
| مخططات البناء (DTS) | Device Tree Source file |
| الـ Blob الثنائي (DTB) | Compiled device tree binary |
| عامل الاستقبال (bootloader) | U-Boot / UEFI يُمرر DTB عبر r2/x0 |
| قسم الموارد البشرية | `of_core_init()` + OF framework |
| الدليل الداخلي | شجرة `struct device_node` في kernel memory |
| اللافتة على الباب | خاصية `compatible` في الـ DTS node |
| رقم الغرفة + طولها | خاصية `reg = <address size>` |
| سلك الكهرباء | خاصية `interrupts` |
| مرجع لغرفة أخرى | الـ `phandle` |

---

### المعمارية الكبيرة (Big Picture Architecture)

```
  ┌─────────────────────────────────────────────────────────┐
  │                   Boot Sequence                         │
  │                                                         │
  │  SoC ROM → U-Boot → passes DTB addr in CPU register    │
  └───────────────────────┬─────────────────────────────────┘
                          │ DTB blob (binary)
                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │             early_init_dt_scan() [arch/arm/...]         │
  │   Reads DTB, calls unflatten_device_tree()              │
  │   Builds struct device_node tree in kernel heap         │
  └───────────────────────┬─────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │            OF Core (drivers/of/  +  include/linux/of.h) │
  │                                                         │
  │  of_root ──► device_node [/]                            │
  │               ├── device_node [cpus]                    │
  │               │     └── device_node [cpu@0]             │
  │               ├── device_node [memory@80000000]         │
  │               ├── device_node [soc]                     │
  │               │     ├── device_node [uart@FF010000]     │
  │               │     ├── device_node [i2c@FF020000]      │
  │               │     └── device_node [gpio@FF030000]     │
  │               └── device_node [chosen]                  │
  │                                                         │
  │  Global pointers: of_root, of_chosen, of_aliases        │
  └───────────────────────┬─────────────────────────────────┘
                          │  of_platform_populate()
                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │         Platform Bus / Driver Matching Layer            │
  │                                                         │
  │  For each device_node with "compatible":                │
  │    → create platform_device                             │
  │    → match against of_device_id tables in drivers       │
  │    → call driver->probe(dev)                            │
  └───────────────────────┬─────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
  ┌────────────┐  ┌─────────────┐  ┌─────────────┐
  │ UART driver│  │  I2C driver │  │ GPIO driver │
  │  probe()   │  │  probe()    │  │  probe()    │
  │            │  │             │  │             │
  │of_property │  │of_property  │  │of_property  │
  │_read_u32() │  │_read_u32()  │  │_read_u32()  │
  └────────────┘  └─────────────┘  └─────────────┘
```

---

### الـ Core Abstraction: من DTB إلى `struct device_node`

**المفهوم المحوري:** الـ OF framework يحوّل الـ DTB الثنائي (flat structure) إلى **شجرة من الـ `struct device_node`** في ذاكرة الكيرنل، وتُصبح هذه الشجرة مرجع الكيرنل الدائم للـ hardware topology.

#### الـ `struct device_node` — العظمة الفقرية للشجرة

```c
struct device_node {
    const char *name;          /* اسم الـ node مثل "uart" */
    phandle phandle;           /* رقم فريد للإشارة لهذا الـ node من node آخر */
    const char *full_name;     /* المسار الكامل: "/soc/uart@FF010000" */
    struct fwnode_handle fwnode; /* ← نقطة التكامل مع الـ Unified Property API */

    struct property *properties; /* رأس القائمة المترابطة لكل الخصائص */
    struct property *deadprops;  /* خصائص محذوفة (للـ overlay support) */
    struct device_node *parent;  /* الـ node الأب */
    struct device_node *child;   /* أول ابن */
    struct device_node *sibling; /* الأخ التالي في نفس المستوى */

    unsigned long _flags;  /* OF_DYNAMIC, OF_POPULATED, OF_OVERLAY, إلخ */
    void *data;            /* بيانات خاصة بالـ arch */
};
```

**ملاحظة على `fwnode_handle`:** هذا الحقل مهم جداً — الـ **Unified Property API** (subsystem آخر، `include/linux/property.h`) يُجرّد الفروق بين Device Tree و ACPI و software nodes. الـ `fwnode_handle` هو "الواجهة المشتركة"، والـ `device_node` هو التنفيذ الخاص بالـ OF. الماكرو `to_of_node()` يستخدم `container_of()` للتحويل بينهما.

```c
/* كيف يعمل to_of_node(): */
#define to_of_node(__fwnode)                              \
    is_of_node(__to_of_node_fwnode) ?                     \
        container_of(__to_of_node_fwnode,                 \
                     struct device_node, fwnode) :        \
        NULL;
/* fwnode مدمج داخل device_node، container_of يحسب العنوان */
```

#### الـ `struct property` — وحدة التخزين الأساسية

```c
struct property {
    char  *name;      /* مثل: "compatible", "reg", "interrupts" */
    int    length;    /* طول الـ value بالبايت */
    void  *value;     /* البيانات الخام — big-endian دائماً */
    struct property *next; /* قائمة مترابطة بسيطة */
};
```

**لماذا big-endian؟** مواصفة Device Tree تنص على أن كل الأرقام في الـ DTB تُخزَّن بصيغة big-endian. لذلك عند قراءة `reg` أو `interrupts` يجب استخدام `be32_to_cpu()`. الدالة المساعدة `of_read_number()` تفعل هذا تلقائياً:

```c
static inline u64 of_read_number(const __be32 *cell, int size)
{
    u64 r = 0;
    for (; size--; cell++)
        r = (r << 32) | be32_to_cpu(*cell); /* تحويل من BE إلى host order */
    return r;
}
```

---

### رسم العلاقات بين الـ Structs

```
struct device_node
┌─────────────────────────────────┐
│ name: "uart"                    │
│ full_name: "/soc/uart@FF010000" │
│ phandle: 0x12                   │
│                                 │
│ fwnode ──────────────────────────────► struct fwnode_handle
│  (embedded, not a pointer)      │      { ops = &of_fwnode_ops }
│                                 │
│ properties ─────────────────────────► struct property
│                                 │     { name="compatible",
│                                 │       value="ns16550a",
│                                 │       next ─────────────► struct property
│                                 │     }                      { name="reg",
│                                 │                             value=<FF010000 100>,
│                                 │                             next ──► ... }
│ parent ──────────────────────────────► device_node [/soc]
│ child  ──────────────────────────────► NULL (leaf node)
│ sibling ─────────────────────────────► device_node [i2c@FF020000]
│ _flags: OF_POPULATED            │
└─────────────────────────────────┘
```

---

### الـ `phandle` والإشارات بين الـ Nodes

**الـ `phandle`** (اختصار physical handle) هو رقم صحيح u32 فريد يُعيَّن لكل node في الـ DTS، يُستخدم للإشارة من node إلى آخر.

مثال في DTS:
```dts
interrupt-controller@1C81000 {
    #interrupt-cells = <3>;
    interrupt-controller;
    phandle = <0x1>;    /* ← الكيرنل يُعيّن هذا */
};

uart0 {
    compatible = "ns16550a";
    reg = <0xFF010000 0x100>;
    interrupts-extended = <&intc 0 32 4>; /* ← phandle إلى intc + 3 args */
};
```

في الكيرنل، هذه الإشارة تُحلّ بـ `struct of_phandle_args`:

```c
struct of_phandle_args {
    struct device_node *np;     /* الـ node المُشار إليه (intc في المثال) */
    int args_count;             /* عدد الـ arguments (3 في المثال) */
    uint32_t args[MAX_PHANDLE_ARGS]; /* [0]=0, [1]=32, [2]=4 */
};
```

دالة `of_parse_phandle_with_args()` تقوم بالعمل:
1. تقرأ قيمة الـ phandle (0x1) من الـ property value.
2. تبحث عن الـ node صاحب هذا الـ phandle بـ `of_find_node_by_phandle(0x1)`.
3. تقرأ عدد الـ args من `#interrupt-cells` في الـ node المُشار إليه.
4. تملأ `struct of_phandle_args` وتُعيدها للـ driver.

---

### الـ `of_phandle_iterator` — iterator متقدم

عندما يكون لدى node قائمة من الـ phandles (مثل `clocks = <&clk1 0 &clk2 2>`) ولكل phandle عدد مختلف من الـ arguments، يُستخدم الـ iterator:

```c
struct of_phandle_iterator {
    const char *cells_name; /* اسم الـ property التي تحدد عدد الـ cells */
    int cell_count;         /* أو عدد ثابت إذا لم يوجد cells_name */
    const struct device_node *parent;
    const __be32 *list_end;    /* نهاية قائمة الـ phandles في الـ property */
    const __be32 *phandle_end; /* نهاية الـ phandle الحالي */
    const __be32 *cur;         /* المؤشر الحالي في القائمة */
    uint32_t cur_count;        /* عدد الـ cells للـ phandle الحالي */
    phandle phandle;           /* قيمة الـ phandle الحالية */
    struct device_node *node;  /* الـ node المقابل */
};
```

الاستخدام الفعلي:
```c
struct of_phandle_iterator it;
int err;

/* يمر على كل clock مذكور في "clocks" property */
of_for_each_phandle(&it, err, np, "clocks", "#clock-cells", -1) {
    /* it.node = device_node للـ clock controller */
    /* it.cur_count = عدد الـ args */
    /* it.args = قيم الـ args */
}
```

---

### الـ `of_device_id` ودور الـ compatible string

**الـ `struct of_device_id`** هو جسر الربط بين الـ driver والـ DT node:

```c
struct of_device_id {
    char name[32];        /* نادراً ما يُستخدم */
    char type[32];        /* نادراً ما يُستخدم */
    char compatible[128]; /* ← الأهم: مثل "fsl,imx8-uart" */
    const void *data;     /* بيانات خاصة بهذا الـ match (مثل variant info) */
};
```

كل driver يُعلن عن نفسه بجدول `of_device_id`:
```c
static const struct of_device_id imx_uart_dt_ids[] = {
    { .compatible = "fsl,imx8-uart", .data = &imx8_data },
    { .compatible = "fsl,imx6q-uart", .data = &imx6_data },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, imx_uart_dt_ids); /* يُصدَّر لـ udev/modprobe */
```

الكيرنل يُخزّن هذه الجداول في section خاصة `__of_device_id_table` في الـ ELF binary. عند تشغيل `of_platform_populate()`، لكل `device_node` مع `compatible`، تُستدعى `of_match_node()` التي تمشي على كل driver مُسجّل وتقارن الـ compatible strings.

---

### الـ Changeset: تعديل الشجرة أثناء التشغيل

**(يتطلب فهم الـ notifier chain — subsystem آخر — وهو آلية pub/sub داخل الكيرنل تُتيح لـ subsystems الاشتراك في أحداث معينة)**

الـ **`struct of_changeset`** يتيح تطبيق مجموعة من التعديلات على الشجرة atomically مع إمكانية الـ rollback:

```c
struct of_changeset {
    struct list_head entries; /* قائمة من of_changeset_entry */
};

struct of_changeset_entry {
    struct list_head node;
    unsigned long action;     /* ATTACH_NODE, ADD_PROPERTY, إلخ */
    struct device_node *np;
    struct property *prop;
    struct property *old_prop; /* لإمكانية الـ revert */
};
```

الاستخدام الرئيسي: **Device Tree Overlays** — تُستخدم في Raspberry Pi وBeagleBone لتفعيل HATs وCapes ديناميكياً دون إعادة boot:

```c
/* تطبيق overlay يضيف I2C device جديد */
of_overlay_fdt_apply(overlay_fdt, size, &ovcs_id, target_base);
/* ... الـ overlay يُعدّل الشجرة، يُنشئ platform_device جديد ... */

/* إزالة الـ overlay لاحقاً */
of_overlay_remove(&ovcs_id); /* يُعيد الشجرة لحالتها الأصلية */
```

---

### ما يملكه الـ OF Framework مقابل ما يُفوّضه للـ drivers

| ما يملكه OF Framework | ما يُفوّضه للـ drivers |
|---|---|
| بناء شجرة `device_node` من DTB | تفسير قيم `reg` property (ماذا تعني الـ addresses) |
| إدارة الـ reference counting (`of_node_get/put`) | تسجيل IRQs بناءً على `interrupts` |
| تقديم API للبحث في الشجرة | map الـ memory regions بـ `ioremap()` |
| مطابقة `compatible` strings بالـ drivers | تهيئة الـ hardware نفسه |
| قراءة الـ properties بأنواعها المختلفة | إدارة الـ clock/power lifecycle |
| إدارة الـ phandle references | التعامل مع الـ DMA |
| تصدير الشجرة عبر `/proc/device-tree` (sysfs/kobject) | اتخاذ قرارات الـ probe ordering |
| تطبيق الـ overlays وإمكانية الـ revert | — |

---

### الـ Flags وحالة الـ Node

كل `device_node` يحمل `_flags` تصف حالته الحالية:

| Flag | القيمة | المعنى |
|---|---|---|
| `OF_DYNAMIC` | 1 | الـ node مُخصَّص بـ kmalloc (يمكن تحريره) |
| `OF_DETACHED` | 2 | تم فصله عن الشجرة (في طور الحذف) |
| `OF_POPULATED` | 3 | تم إنشاء `platform_device` مقابله |
| `OF_POPULATED_BUS` | 4 | تم إنشاء bus device للـ children |
| `OF_OVERLAY` | 5 | جاء من overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | ضمن changeset قيد الحذف |

الدالة `of_node_set_flag()` تستخدم `set_bit()` الـ atomic — ضروري لأن الـ OF tree يُقرأ من سياقات متعددة (interrupt context أحياناً).

---

### مثال عملي متكامل: driver يقرأ من Device Tree

```c
/* مقتطف من driver حقيقي لـ UART */
static int my_uart_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node; /* ← device_node المقابل */
    u32 baud_rate, fifo_size;
    struct resource *res;

    /* قراءة خاصية عددية */
    if (of_property_read_u32(np, "clock-frequency", &baud_rate)) {
        dev_err(dev, "missing clock-frequency\n");
        return -EINVAL;
    }

    /* قراءة خاصية اختيارية */
    if (of_property_read_u32(np, "fifo-depth", &fifo_size))
        fifo_size = 16; /* default */

    /* التحقق من وجود خاصية boolean */
    if (of_property_read_bool(np, "hw-flow-control"))
        setup_hw_flow_control();

    /* الحصول على clock controller عبر phandle */
    struct clk *clk = of_clk_get(np, 0);
    /* of_clk_get تستخدم of_parse_phandle_with_args() داخلياً */

    /* الـ platform subsystem يقرأ "reg" ويُحوّلها لـ struct resource */
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    void __iomem *base = devm_ioremap_resource(dev, res);

    return 0;
}

static const struct of_device_id my_uart_of_ids[] = {
    { .compatible = "myvendor,myuart-v1" },
    { }
};

static struct platform_driver my_uart_driver = {
    .probe  = my_uart_probe,
    .driver = {
        .name = "my-uart",
        .of_match_table = my_uart_of_ids,
    },
};
```

**ما يحدث تحت الغطاء عند `of_property_read_u32(np, "clock-frequency", &val)`:**
1. تُستدعى `of_find_property(np, "clock-frequency", NULL)` → تمشي على قائمة `np->properties`.
2. تجد `struct property` باسم `"clock-frequency"`.
3. تقرأ `property->value` كـ `__be32` وتُحوّله بـ `be32_to_cpu()`.
4. تكتب النتيجة في `&val`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### أولاً: flags الـ `device_node` (حقل `_flags`)

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `OF_DYNAMIC` | 1 | الـ node والـ properties مخصصة بـ `kmalloc` (قابلة للتحرير) |
| `OF_DETACHED` | 2 | الـ node منفصلة عن شجرة الأجهزة الحية |
| `OF_POPULATED` | 3 | تم إنشاء الـ `struct device` المقابل بالفعل |
| `OF_POPULATED_BUS` | 4 | تم إنشاء platform bus للأبناء |
| `OF_OVERLAY` | 5 | الـ node مخصصة لـ overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | الـ node داخل changeset يُحرَّر حالياً |

#### ثانياً: flags الـ `fwnode_handle` (من `fwnode.h`)

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `FWNODE_FLAG_LINKS_ADDED` | BIT(0) | تمت معالجة الـ fwnode links |
| `FWNODE_FLAG_NOT_DEVICE` | BIT(1) | لن يُحوَّل إلى `struct device` |
| `FWNODE_FLAG_INITIALIZED` | BIT(2) | تمت تهيئة الـ hardware المقابل |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | BIT(3) | يحتاج ربط الأبناء أولاً |
| `FWNODE_FLAG_BEST_EFFORT` | BIT(4) | probe مبكر، يتجاهل بعض الـ suppliers |
| `FWNODE_FLAG_VISITED` | BIT(5) | تمت زيارته أثناء المسح |

#### ثالثاً: أكواد `OF_RECONFIG` (أحداث إعادة التهيئة الديناميكية)

| الـ Code | القيمة | الحدث |
|---|---|---|
| `OF_RECONFIG_ATTACH_NODE` | 0x0001 | إضافة node جديدة للشجرة |
| `OF_RECONFIG_DETACH_NODE` | 0x0002 | إزالة node من الشجرة |
| `OF_RECONFIG_ADD_PROPERTY` | 0x0003 | إضافة property لـ node |
| `OF_RECONFIG_REMOVE_PROPERTY` | 0x0004 | حذف property من node |
| `OF_RECONFIG_UPDATE_PROPERTY` | 0x0005 | تحديث قيمة property موجودة |

#### رابعاً: `enum of_reconfig_change`

| القيمة | المعنى |
|---|---|
| `OF_RECONFIG_NO_CHANGE` | لا تغيير |
| `OF_RECONFIG_CHANGE_ADD` | تمت إضافة عنصر |
| `OF_RECONFIG_CHANGE_REMOVE` | تمت إزالة عنصر |

#### خامساً: `enum of_overlay_notify_action`

| القيمة | الحدث |
|---|---|
| `OF_OVERLAY_INIT` | تهيئة الـ overlay object |
| `OF_OVERLAY_PRE_APPLY` | قبل تطبيق الـ overlay |
| `OF_OVERLAY_POST_APPLY` | بعد تطبيق الـ overlay |
| `OF_OVERLAY_PRE_REMOVE` | قبل إزالة الـ overlay |
| `OF_OVERLAY_POST_REMOVE` | بعد إزالة الـ overlay |

#### سادساً: خيارات الـ Kconfig الأساسية

| الـ Option | الأثر |
|---|---|
| `CONFIG_OF` | تفعيل الدعم الكامل لـ Device Tree |
| `CONFIG_OF_DYNAMIC` | دعم التعديل الديناميكي للشجرة (add/remove/update) |
| `CONFIG_OF_KOBJ` | دمج الـ nodes مع نظام `kobject`/`sysfs` |
| `CONFIG_OF_OVERLAY` | دعم تطبيق وإزالة الـ overlays |
| `CONFIG_OF_NUMA` | دعم NUMA من معلومات الـ Device Tree |
| `CONFIG_OF_PROMTREE` | دعم SPARC/OpenFirmware (يضيف `unique_id`) |

---

### الـ Structs الأساسية

#### 1. `struct property`

**الغرض:** تمثيل خاصية واحدة من خصائص الـ device node في شجرة الأجهزة. كل node تملك قائمة مرتبطة من هذه الـ properties (مثل `compatible`, `reg`, `interrupts`).

```c
struct property {
    char    *name;          /* اسم الخاصية، مثل "compatible" */
    int      length;        /* حجم البيانات بالبايت */
    void    *value;         /* البيانات الخام (big-endian للأرقام) */
    struct property *next;  /* الخاصية التالية في القائمة المرتبطة */
    /* CONFIG_OF_DYNAMIC || CONFIG_SPARC فقط: */
    unsigned long _flags;   /* flags للخاصية (dynamic, etc.) */
    /* CONFIG_OF_PROMTREE فقط: */
    unsigned int unique_id; /* معرف فريد لـ SPARC/PROM */
    /* CONFIG_OF_KOBJ فقط: */
    struct bin_attribute attr; /* لعرض الخاصية في /sys/firmware/devicetree/ */
};
```

| الحقل | الوصف |
|---|---|
| `name` | اسم الخاصية كنص |
| `length` | حجم `value` بالبايت، صفر للخاصية البوليانية |
| `value` | مؤشر للبيانات، تُقرأ بـ `of_read_number()` أو `of_property_read_*()` |
| `next` | القائمة المرتبطة لجميع properties الـ node |
| `_flags` | يُستخدم للتمييز بين الـ properties الثابتة والديناميكية |
| `attr` | يتيح قراءة الخاصية من `sysfs` كـ binary attribute |

---

#### 2. `struct device_node`

**الغرض:** العنصر الأساسي في شجرة الأجهزة. يمثل كل node (عقدة) في الـ DTB سواء كانت CPU أو bus أو جهاز طرفي. الشجرة بأكملها مبنية من هذه الـ structs المترابطة.

```c
struct device_node {
    const char     *name;         /* اسم العقدة (الجزء قبل @) */
    phandle         phandle;      /* معرف رقمي فريد لهذه العقدة في الـ DTB */
    const char     *full_name;    /* المسار الكامل: "/soc/uart@ff000000" */
    struct fwnode_handle fwnode;  /* واجهة firmware المجردة */

    struct property *properties;  /* قائمة الخصائص النشطة */
    struct property *deadprops;   /* الخصائص المحذوفة (للـ rollback) */
    struct device_node *parent;   /* العقدة الأم */
    struct device_node *child;    /* أول عقدة ابن */
    struct device_node *sibling;  /* العقدة الشقيقة التالية */

    /* CONFIG_OF_KOBJ فقط: */
    struct kobject   kobj;        /* لتمثيل العقدة في /sys/firmware/devicetree/ */

    unsigned long   _flags;       /* OF_DYNAMIC, OF_POPULATED, etc. */
    void           *data;         /* بيانات خاصة بالمنصة */

    /* CONFIG_SPARC فقط: */
    unsigned int    unique_id;
    struct of_irq_controller *irq_trans;
};
```

| الحقل | الوصف |
|---|---|
| `phandle` | رقم فريد يسمح لـ nodes أخرى بالإشارة لهذه العقدة بالاسم |
| `full_name` | المسار الكامل في الشجرة، يُستخدم في تشخيص المشاكل |
| `fwnode` | يربط الـ node بطبقة الـ firmware abstraction |
| `properties` | قائمة مرتبطة بجميع خصائص العقدة النشطة |
| `deadprops` | الخصائص المحذوفة، تُحفظ للـ rollback في الـ changesets |
| `parent/child/sibling` | بنية الشجرة الثلاثية (أب، أول ابن، الشقيق التالي) |
| `_flags` | حالة العقدة في دورة حياتها |
| `kobj` | يتيح إنشاء `/sys/firmware/devicetree/base/<path>` |

---

#### 3. `struct of_phandle_args`

**الغرض:** نتيجة تحليل إشارة phandle مع وسائطها. تُستخدم عند قراءة properties مثل `clocks = <&clk0 1 2>` حيث الـ phandle يشير لـ `clk0` والأرقام هي وسائط.

```c
struct of_phandle_args {
    struct device_node *np;              /* العقدة المُشار إليها */
    int                 args_count;      /* عدد الوسائط الفعلية */
    uint32_t            args[MAX_PHANDLE_ARGS]; /* الوسائط (حتى 16) */
};
```

**مثال عملي:** لقراءة `clocks = <&timer0 2>`:
```c
struct of_phandle_args clk_args;
of_parse_phandle_with_args(dev->of_node, "clocks", "#clock-cells", 0, &clk_args);
/* clk_args.np → &timer0 node */
/* clk_args.args[0] = 2 */
```

---

#### 4. `struct of_phandle_iterator`

**الغرض:** تيرة (iterator) للتنقل عبر قائمة من الـ phandles في property واحدة. يُستخدم بواسطة `of_for_each_phandle()`.

```c
struct of_phandle_iterator {
    const char          *cells_name;   /* اسم الخاصية التي تحدد عدد الوسائط */
    int                  cell_count;   /* عدد ثابت للوسائط (-1 = قرأ من الشجرة) */
    const struct device_node *parent;  /* العقدة الأم التي نتكرر على properties-ها */

    const __be32        *list_end;     /* نهاية القائمة الكاملة */
    const __be32        *phandle_end;  /* نهاية العنصر الحالي */

    const __be32        *cur;          /* الموضع الحالي في القائمة */
    uint32_t             cur_count;    /* عدد الوسائط للعنصر الحالي */
    phandle              phandle;      /* الـ phandle الحالي */
    struct device_node  *node;         /* العقدة المحلولة للـ phandle الحالي */
};
```

---

#### 5. `struct of_reconfig_data`

**الغرض:** يُرسل كبيانات لـ notifier chain عند تغيير الشجرة الحية. يحمل معلومات عن التغيير الذي حدث.

```c
struct of_reconfig_data {
    struct device_node *dn;        /* العقدة المتأثرة */
    struct property    *prop;      /* الخاصية الجديدة/المتغيرة */
    struct property    *old_prop;  /* الخاصية القديمة (للـ update) */
};
```

---

#### 6. `struct of_changeset_entry`

**الغرض:** يمثل عملية تغيير واحدة (atomic operation) داخل changeset. كل تعديل على الشجرة يُسجَّل كـ entry في قائمة مرتبطة لتمكين الـ rollback.

```c
struct of_changeset_entry {
    struct list_head    node;      /* ربط العنصر في قائمة الـ changeset */
    unsigned long       action;    /* نوع العملية: OF_RECONFIG_* */
    struct device_node *np;        /* العقدة المستهدفة */
    struct property    *prop;      /* الخاصية الجديدة */
    struct property    *old_prop;  /* الخاصية القديمة قبل التعديل */
};
```

---

#### 7. `struct of_changeset`

**الغرض:** مجموعة من التغييرات تُطبَّق أو تُلغى كوحدة واحدة (atomic). يُستخدم في الـ overlays لضمان consistency.

```c
struct of_changeset {
    struct list_head entries; /* قائمة بجميع of_changeset_entry */
};
```

**مثال استخدام:**
```c
struct of_changeset ocs;
of_changeset_init(&ocs);
of_changeset_add_property(&ocs, node, new_prop);
of_changeset_attach_node(&ocs, new_node);

if (of_changeset_apply(&ocs) < 0)
    of_changeset_revert(&ocs);  /* rollback كامل */

of_changeset_destroy(&ocs);
```

---

#### 8. `struct of_overlay_notify_data`

**الغرض:** بيانات الـ notifier لأحداث الـ overlay.

```c
struct of_overlay_notify_data {
    struct device_node *overlay;  /* العقدة الجذر للـ overlay */
    struct device_node *target;   /* العقدة المستهدفة في الشجرة الأصلية */
};
```

---

#### 9. `struct of_device_id` (من `mod_devicetable.h`)

**الغرض:** جدول المطابقة — يربط compatible string بالـ driver data.

```c
struct of_device_id {
    char        name[32];       /* مطابقة اسم العقدة (نادراً) */
    char        type[32];       /* مطابقة device_type (نادراً) */
    char        compatible[128]; /* "vendor,device" — الأكثر استخداماً */
    const void *data;           /* بيانات خاصة بالـ driver */
};
```

---

#### 10. `struct fwnode_handle` (من `fwnode.h`)

**الغرض:** الواجهة المجردة لجميع مصادر firmware (OF, ACPI, software nodes). الـ `device_node` يحتوي على واحدة منها.

```c
struct fwnode_handle {
    struct fwnode_handle        *secondary; /* fwnode ثانوي مرتبط */
    const struct fwnode_operations *ops;    /* جدول العمليات الافتراضية */
    struct device               *dev;       /* الجهاز المرتبط */
    struct list_head             suppliers; /* روابط الـ suppliers */
    struct list_head             consumers; /* روابط الـ consumers */
    u8                           flags;     /* FWNODE_FLAG_* */
};
```

---

### مخططات العلاقات بين الـ Structs

#### مخطط الشجرة الكاملة

```
of_root (struct device_node *)
│
├── fwnode ────────────────────────────────→ struct fwnode_handle
│                                                  │
│                                                  └── ops → of_fwnode_ops (vtable)
│
├── properties ────────────────────────────→ struct property
│                                                  │
│                                                  ├── name: "compatible"
│                                                  ├── value: "arm,cortex-a53"
│                                                  └── next ──→ struct property
│                                                                    └── next ──→ NULL
│
├── child ─────────────────────────────────→ struct device_node  (e.g. /cpus)
│         ├── parent ←─────────────────────── (يشير لـ of_root)
│         ├── sibling ──────────────────────→ struct device_node  (e.g. /memory)
│         │         └── sibling ────────────→ struct device_node  (e.g. /soc)
│         └── child ───────────────────────→ struct device_node  (e.g. /cpus/cpu@0)
│
└── kobj ──────────────────────────────────→ struct kobject
                                                   │
                                              /sys/firmware/devicetree/base/
```

#### العلاقة بين of_changeset والشجرة

```
struct of_changeset
│
└── entries (list_head)
      │
      ├── struct of_changeset_entry [0]
      │         ├── action: OF_RECONFIG_ADD_PROPERTY
      │         ├── np ────────────────────────→ struct device_node
      │         ├── prop ──────────────────────→ struct property (new)
      │         └── old_prop ─────────────────→ NULL
      │
      ├── struct of_changeset_entry [1]
      │         ├── action: OF_RECONFIG_ATTACH_NODE
      │         ├── np ────────────────────────→ struct device_node (new node)
      │         └── prop ─────────────────────→ NULL
      │
      └── struct of_changeset_entry [N]
                └── ...
```

#### العلاقة بين of_phandle_args وشجرة الأجهزة

```
device_node A  ("uart@ff000000")
│
└── properties: "clocks = <&clk0 1 0>"
                            ↑
                         phandle

of_parse_phandle_with_args()
          │
          ↓
struct of_phandle_args {
    np = &clk0_node,      ────→ device_node B ("clk0")
    args_count = 2,
    args[0] = 1,
    args[1] = 0
}
```

---

### مخطط دورة الحياة (Lifecycle)

#### دورة حياة الـ `device_node`

```
[Boot / DTB parsing]
        │
        ▼
   kzalloc() لكل node
        │
        ▼
   of_node_init(node)
   ├── kobject_init(&node->kobj, &of_node_ktype)   [CONFIG_OF_KOBJ]
   └── fwnode_init(&node->fwnode, &of_fwnode_ops)
        │
        ▼
   ربط الشجرة (parent/child/sibling)
   + ملء properties من DTB
        │
        ▼
   of_root يشير لأول عقدة
        │
        ▼
   [استخدام: of_find_node_by_*, of_get_property, ...]
        │
        ├── [CONFIG_OF_DYNAMIC] ─→ يمكن تعديل الشجرة
        │                              of_add_property()
        │                              of_remove_property()
        │                              of_attach_node()
        │                              of_detach_node()
        │
        ▼
   device driver يقرأ node
   └── of_node_get(node)   [يزيد refcount]
             │
             ▼
        driver يستخدم properties
             │
             ▼
        of_node_put(node)  [ينقص refcount]
             │
             ▼
        [إذا refcount == 0 && OF_DYNAMIC]
             └── of_node_release() → kfree()
```

#### دورة حياة الـ `of_changeset`

```
of_changeset_init(&ocs)
        │  (تهيئة قائمة entries فارغة)
        ▼
of_changeset_action() / of_changeset_add_property() / ...
        │  (تُضاف of_changeset_entry لقائمة entries)
        ▼
of_changeset_apply(&ocs)
        │
        ├── [نجاح] ─→ تُطبَّق جميع التغييرات على الشجرة الحية
        │              يُرسَل notifier لكل تغيير
        │
        └── [فشل]  ─→ of_changeset_revert() تلقائياً
                        تُلغى جميع التغييرات المطبقة
        │
        ▼
[لاحقاً] of_changeset_revert(&ocs)   [اختياري، لإلغاء overlay مثلاً]
        │
        ▼
of_changeset_destroy(&ocs)
        └── تحرير جميع of_changeset_entry والـ properties المرتبطة
```

#### دورة حياة الـ Overlay

```
[FDT blob للـ overlay]
        │
        ▼
of_overlay_fdt_apply(fdt, size, &ovcs_id, target_base)
        │
        ├── تحليل الـ FDT blob
        ├── إنشاء of_changeset داخلياً
        ├── إرسال OF_OVERLAY_PRE_APPLY لـ notifiers
        ├── of_changeset_apply()
        └── إرسال OF_OVERLAY_POST_APPLY لـ notifiers
        │
        ▼
   [الـ overlay نشط، ovcs_id يُعرِّف الـ overlay]
        │
        ▼
of_overlay_remove(&ovcs_id)
        ├── إرسال OF_OVERLAY_PRE_REMOVE
        ├── of_changeset_revert()  [يعكس التغييرات]
        └── إرسال OF_OVERLAY_POST_REMOVE
        │
        ▼
   [الشجرة عادت لحالتها الأصلية]
```

---

### مخططات تدفق الاستدعاءات (Call Flow)

#### قراءة property بسيطة

```
driver calls:
  of_property_read_u32(np, "reg", &val)
    │
    └── of_property_read_u32_array(np, "reg", &val, 1)
          │
          └── of_property_read_variable_u32_array(np, "reg", &val, 1, 0)
                │
                ├── of_find_property(np, "reg", &len)
                │     ├── raw_spin_lock(&devtree_lock)        [أخذ القفل]
                │     ├── البحث في np->properties (قائمة مرتبطة)
                │     ├── raw_spin_unlock(&devtree_lock)       [تحرير القفل]
                │     └── return: struct property *
                │
                ├── التحقق من الحجم (len >= sz * sizeof(u32))
                ├── قراءة القيم: be32_to_cpup() لكل عنصر
                └── return 0 (نجاح) أو -EINVAL/-ENODATA/-EOVERFLOW
```

#### تحليل phandle مع وسائط

```
driver calls:
  of_parse_phandle_with_args(np, "clocks", "#clock-cells", 0, &args)
    │
    └── __of_parse_phandle_with_args(np, "clocks", "#clock-cells", -1, 0, &args)
          │
          ├── of_phandle_iterator_init(&it, np, "clocks", "#clock-cells", -1)
          │     └── قراءة property "clocks" من np->properties
          │
          ├── of_phandle_iterator_next(&it)
          │     ├── قراءة phandle الأول من القائمة
          │     ├── of_find_node_by_phandle(phandle)
          │     │     └── بحث في الشجرة بـ phandle كمفتاح
          │     ├── قراءة "#clock-cells" من العقدة المُحلَّلة
          │     └── تقدم المؤشر cur بعدد الوسائط
          │
          ├── ملء args.np و args.args[]
          └── return 0
```

#### تطبيق changeset

```
of_changeset_apply(&ocs)
    │
    └── __of_changeset_apply_entries(&ocs, &last)
          │
          ├── for each entry in ocs.entries:
          │     │
          │     ├── [OF_RECONFIG_ATTACH_NODE]
          │     │     └── __of_attach_node(entry->np)
          │     │           ├── raw_spin_lock(&devtree_lock)
          │     │           ├── ربط np في الشجرة (parent->child)
          │     │           └── raw_spin_unlock(&devtree_lock)
          │     │
          │     ├── [OF_RECONFIG_ADD_PROPERTY]
          │     │     └── __of_add_property(entry->np, entry->prop)
          │     │           ├── raw_spin_lock(&devtree_lock)
          │     │           ├── إضافة prop لـ np->properties
          │     │           └── raw_spin_unlock(&devtree_lock)
          │     │
          │     └── of_reconfig_notify(entry->action, &rd)
          │           └── blocking_notifier_call_chain(...)
          │
          └── [في حالة خطأ] → __of_changeset_revert_entries(last → head)
```

#### مطابقة driver بـ device_node

```
platform_driver_probe() / of_platform_populate()
    │
    └── of_match_node(driver->of_match_table, node)
          │
          ├── for each entry in matches[]:
          │     ├── مقارنة entry.name مع node->name
          │     ├── مقارنة entry.type مع of_get_property(node, "device_type")
          │     └── of_device_is_compatible(node, entry.compatible)
          │           ├── of_get_property(node, "compatible", &len)
          │           └── strcmp() على كل string في قائمة compatible
          │
          └── return: أول of_device_id مطابق أو NULL
```

---

### استراتيجية الـ Locking

#### الأقفال المستخدمة

| القفل | النوع | ما يحميه |
|---|---|---|
| `devtree_lock` | `raw_spinlock_t` | قراءة/كتابة الشجرة (nodes وproperties) |
| قفل الـ `of_changeset` | ضمني (mutex في التطبيق) | تسلسل تطبيق الـ changesets |
| `kobj` reference counting | atomic | دورة حياة الـ node في sysfs |
| refcount في `of_node_get/put` | `kobject_get/put` (أو noop) | منع التحرير أثناء الاستخدام |

#### قواعد القفل

**الـ `devtree_lock` (raw_spinlock):**
- يُؤخذ عند أي وصول للشجرة: قراءة properties، البحث عن nodes، تعديل الشجرة.
- **raw_spinlock** (وليس spinlock عادي) لأنه يُستخدم في سياقات تعطّل الـ interrupts مبكراً.
- يجب ألا يُحتفظ به أثناء استدعاء كود يمكن أن ينام.

**ترتيب القفل (Lock Ordering):**
```
devtree_lock     ←  يُؤخذ أولاً
    └── kobj_lock (داخل kobject) ← يُؤخذ ثانياً (إن احتيج)
```

**الـ Reference Counting:**
- `of_node_get()` → `kobject_get(&node->kobj)` (مع `CONFIG_OF_KOBJ`)
- `of_node_put()` → `kobject_put(&node->kobj)` ← يستدعي `of_node_release()` عند الصفر
- قاعدة: أي دالة تُعيد `struct device_node *` تزيد الـ refcount، على المستدعي استدعاء `of_node_put()`.
- الـ DEFINE_FREE(device_node, ...) يتيح الإدارة التلقائية بـ scope-based cleanup في C.

**الـ Notifiers (CONFIG_OF_DYNAMIC):**
- `of_reconfig_notify()` تستخدم `blocking_notifier_call_chain`
- تُستدعى **خارج** `devtree_lock` لتجنب deadlock مع كود الـ notifier الذي قد يقرأ الشجرة

**مثال آمن على الاستخدام:**
```c
/* قراءة آمنة مع إدارة دورة الحياة */
struct device_node *np __free(device_node) =
    of_find_compatible_node(NULL, NULL, "arm,pl011");
if (!np)
    return -ENODEV;

/* استخدام np هنا ... */
/* of_node_put() تُستدعى تلقائياً عند الخروج من الـ scope */
```
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### الـ Data Structures الأساسية

| البنية | الغرض |
|---|---|
| `struct device_node` | العقدة الأساسية في شجرة الـ device tree |
| `struct property` | خاصية واحدة مرتبطة بعقدة (name + value) |
| `struct of_phandle_args` | نتيجة تحليل phandle مع arguments |
| `struct of_phandle_iterator` | iterator للمرور على قائمة phandles |
| `struct of_changeset` | مجموعة تعديلات atomic على الـ live tree |
| `struct of_changeset_entry` | إدخال واحد في الـ changeset log |
| `struct of_reconfig_data` | بيانات حدث reconfig لإشعار الـ notifiers |
| `struct of_overlay_notify_data` | بيانات إشعار overlay (target + overlay nodes) |

#### جدول Functions الكامل

| الدالة | الفئة | الوصف المختصر |
|---|---|---|
| `of_node_init` | Init | تهيئة عقدة جديدة (kobj + fwnode) |
| `of_node_get` | Refcount | زيادة refcount للعقدة |
| `of_node_put` | Refcount | تخفيض refcount وتحرير العقدة عند الصفر |
| `of_node_check_flag` | Flags | قراءة flag من `_flags` |
| `of_node_test_and_set_flag` | Flags | test-and-set atomic |
| `of_node_set_flag` / `of_node_clear_flag` | Flags | ضبط/مسح flag |
| `of_find_all_nodes` | Traversal | التنقل عبر كل عقد الشجرة |
| `of_find_node_by_name` | Search | بحث بالاسم |
| `of_find_node_by_type` | Search | بحث بالـ device_type |
| `of_find_compatible_node` | Search | بحث بالـ compatible string |
| `of_find_matching_node_and_match` | Search | بحث بجدول `of_device_id` |
| `of_find_node_opts_by_path` | Search | بحث بالمسار مع options |
| `of_find_node_by_phandle` | Search | بحث بقيمة phandle |
| `of_find_node_with_property` | Search | بحث بوجود property |
| `of_get_parent` | Tree Nav | الحصول على العقدة الأم |
| `of_get_next_parent` | Tree Nav | الانتقال للأعلى (يُفرج عن العقدة الحالية) |
| `of_get_next_child` | Tree Nav | الطفل التالي |
| `of_get_next_available_child` | Tree Nav | الطفل التالي غير الـ disabled |
| `of_get_next_reserved_child` | Tree Nav | الطفل المحجوز التالي |
| `of_get_child_by_name` | Tree Nav | طفل بالاسم |
| `of_get_compatible_child` | Tree Nav | طفل بالـ compatible |
| `of_find_property` | Property | البحث عن property بالاسم |
| `of_get_property` | Property | الحصول على قيمة property مباشرة |
| `of_property_read_bool` | Property Read | قراءة boolean property |
| `of_property_present` | Property | التحقق من وجود property |
| `of_property_count_elems_of_size` | Property | عدد العناصر بحجم محدد |
| `of_property_read_u8/u16/u32/u64` | Property Read | قراءة scalar |
| `of_property_read_u8/u16/u32/u64_array` | Property Read | قراءة array |
| `of_property_read_variable_uXX_array` | Property Read | قراءة array بحجم متغير |
| `of_property_read_string` | Property Read | قراءة أول string |
| `of_property_read_string_index` | Property Read | قراءة string بالفهرس |
| `of_property_read_string_array` | Property Read | قراءة مصفوفة strings |
| `of_property_count_strings` | Property Read | عدد الـ strings |
| `of_property_match_string` | Property Read | إيجاد فهرس string |
| `of_property_read_string_helper` | Property Internal | دالة مساعدة للـ strings |
| `of_device_is_compatible` | Device | التحقق من compatible |
| `of_device_compatible_match` | Device | مطابقة قائمة compatibles |
| `of_device_is_available` | Device | هل الجهاز متاح (غير disabled) |
| `of_device_is_big_endian` | Device | هل الجهاز big-endian |
| `of_match_node` | Matching | مطابقة عقدة بجدول of_device_id |
| `of_device_get_match_data` | Matching | data من جدول المطابقة |
| `of_parse_phandle` | Phandle | تحليل phandle بسيط |
| `of_parse_phandle_with_args` | Phandle | تحليل phandle مع args (cells_name) |
| `of_parse_phandle_with_fixed_args` | Phandle | تحليل phandle مع args ثابتة |
| `of_parse_phandle_with_optional_args` | Phandle | تحليل phandle مع args اختيارية |
| `of_count_phandle_with_args` | Phandle | عدد الـ phandles في القائمة |
| `of_phandle_iterator_init` | Phandle Iterator | تهيئة iterator |
| `of_phandle_iterator_next` | Phandle Iterator | الخطوة التالية |
| `of_phandle_iterator_args` | Phandle Iterator | استخراج arguments |
| `of_alias_get_id` | Alias | الحصول على رقم alias |
| `of_alias_get_highest_id` | Alias | أعلى رقم alias |
| `of_alias_from_compatible` | Alias | إنشاء alias من compatible |
| `of_n_addr_cells` | Address | عدد خلايا العنوان |
| `of_n_size_cells` | Address | عدد خلايا الحجم |
| `of_read_number` | Address | قراءة big-endian number |
| `of_map_id` | Address | تحويل ID عبر map property |
| `of_dma_get_max_cpu_address` | DMA | الحد الأقصى لعناوين CPU في DMA |
| `of_get_cpu_node` | CPU | عقدة CPU بالرقم |
| `of_cpu_device_node_get` | CPU | عقدة CPU مع رفع refcount |
| `of_cpu_node_to_id` | CPU | رقم CPU من عقدة |
| `of_get_next_cpu_node` | CPU | التنقل بين عقد CPU |
| `of_get_cpu_state_node` | CPU | عقدة حالة طاقة CPU |
| `of_get_cpu_hwid` | CPU | MPIDR / hardware ID |
| `of_machine_is_compatible` | Machine | مطابقة compatible في root node |
| `of_machine_compatible_match` | Machine | مطابقة قائمة compatibles في root |
| `of_machine_device_match` | Machine | مطابقة جدول of_device_id في root |
| `of_add_property` | Runtime Modify | إضافة property للشجرة الحية |
| `of_remove_property` | Runtime Modify | إزالة property |
| `of_update_property` | Runtime Modify | تحديث property |
| `of_attach_node` | Runtime Modify | إلحاق عقدة بالشجرة |
| `of_detach_node` | Runtime Modify | فصل عقدة |
| `of_changeset_init` | Changeset | تهيئة changeset |
| `of_changeset_destroy` | Changeset | تحرير changeset |
| `of_changeset_apply` | Changeset | تطبيق كل التعديلات |
| `of_changeset_revert` | Changeset | التراجع عن كل التعديلات |
| `of_changeset_action` | Changeset | إضافة action واحد |
| `of_reconfig_notifier_register` | Notifier | تسجيل في أحداث reconfig |
| `of_overlay_fdt_apply` | Overlay | تطبيق FDT overlay |
| `of_overlay_remove` | Overlay | إزالة overlay |
| `of_modalias` | Module | توليد modalias string |
| `of_request_module` | Module | طلب تحميل module |
| `of_console_check` | Console | التحقق من console في DT |
| `of_kexec_alloc_and_setup_fdt` | Kexec | إعداد FDT لـ kexec |
| `of_node_to_nid` | NUMA | رقم الـ NUMA node |

---

### الفئة 1: تهيئة العقد وإدارة الـ Reference Count

هذه الدوال هي نقطة البداية لدورة حياة أي `device_node`. الـ reference counting يتحكم في عمر العقدة في الذاكرة عند تفعيل `CONFIG_OF_DYNAMIC`؛ بدونه تكون العقد static وتبقى طوال عمر الـ kernel.

---

#### `of_node_init`

```c
static inline void of_node_init(struct device_node *node)
```

تهيئة عقدة device tree جديدة تم تخصيصها بـ `kzalloc`. عند تفعيل `CONFIG_OF_KOBJ` يتم تهيئة `kobject` ليسمح بتصدير العقدة عبر sysfs، ثم تهيئة `fwnode_handle` المضمنة داخلها بربطها بـ `of_fwnode_ops`. الـ refcount يُضبط على 1 ضمنياً بواسطة `kobject_init` أو `fwnode_init`.

| المعامل | الوصف |
|---|---|
| `node` | مؤشر للعقدة المُخصصة بـ `kzalloc` — يجب ألا يكون NULL |

**القيمة المُعادة:** لا شيء (void).

**تفاصيل مهمة:** يجب استدعاء `of_node_put()` عند الانتهاء من العقدة الديناميكية. العقد الغير ديناميكية (static FDT) لا تُحرَّر حتى عند وصول refcount للصفر.

**السياق:** يستدعيها كود إنشاء العقد الديناميكية مثل overlay subsystem.

---

#### `of_node_get`

```c
struct device_node *of_node_get(struct device_node *node);
```

يزيد الـ reference count للعقدة بمقدار واحد. عند تفعيل `CONFIG_OF_DYNAMIC` يستدعي `kobject_get` أو `fwnode_handle_get`. بدون `CONFIG_OF_DYNAMIC` هي no-op تُعيد المؤشر كما هو.

| المعامل | الوصف |
|---|---|
| `node` | العقدة المطلوب الاحتفاظ بها — يقبل NULL |

**القيمة المُعادة:** نفس المؤشر `node` (لتسهيل استخدامه في expressions).

**تفاصيل مهمة:** يجب مقابلة كل `of_node_get()` بـ `of_node_put()` لتجنب memory leak.

---

#### `of_node_put`

```c
void of_node_put(struct device_node *node);
```

يُخفض الـ refcount. عند وصوله للصفر يُستدعى `of_node_release()` الذي يُحرر الذاكرة فقط إذا كانت العقدة `OF_DYNAMIC`. تُعرَّف أيضاً `DEFINE_FREE(device_node, ...)` لاستخدامها مع `__free()` في cleanup.h (scoped cleanup).

| المعامل | الوصف |
|---|---|
| `node` | العقدة المراد الإفراج عنها — يقبل NULL |

**القيمة المُعادة:** لا شيء.

**تفاصيل مهمة:** لا يجوز الوصول للعقدة بعد `of_node_put()` إذا كنت المالك الأخير.

---

#### Flag Helpers

```c
int  of_node_check_flag(const struct device_node *n, unsigned long flag);
int  of_node_test_and_set_flag(struct device_node *n, unsigned long flag);
void of_node_set_flag(struct device_node *n, unsigned long flag);
void of_node_clear_flag(struct device_node *n, unsigned long flag);
```

مجموعة thin wrappers حول `test_bit` / `test_and_set_bit` / `set_bit` / `clear_bit` على حقل `n->_flags`. الـ flags المعرَّفة:

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `OF_DYNAMIC` | 1 | تم التخصيص بـ kmalloc (ديناميكي) |
| `OF_DETACHED` | 2 | فُصلت عن الشجرة |
| `OF_POPULATED` | 3 | تم إنشاء platform_device لها |
| `OF_POPULATED_BUS` | 4 | تم إنشاء platform bus للأطفال |
| `OF_OVERLAY` | 5 | جاءت من overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | ضمن changeset يجري حذفه |

**الـ Locking:** `test_and_set_bit` atomic. الباقي يستخدم bitops العادية — يُفضَّل استخدامها تحت `of_mutex` إذا تنافست threads متعددة على نفس العقدة.

---

### الفئة 2: البحث والتنقل في شجرة الـ Device Tree

الـ OF subsystem يحتفظ بشجرة ثنائية من `device_node`، متصلة عبر مؤشرات `parent/child/sibling`. جميع دوال البحث **تُعيد مؤشراً مع refcount مُزاد** — يجب الاستدعاء بـ `of_node_put()` لاحقاً.

---

#### `of_find_all_nodes` و `__of_find_all_nodes`

```c
struct device_node *of_find_all_nodes(struct device_node *prev);
struct device_node *__of_find_all_nodes(struct device_node *prev);
```

تُعيد العقدة التالية في traversal خطي كامل للشجرة. النسخة بدون `__` تأخذ `of_node_get` على النتيجة وتُطلق `of_node_put` على `prev`. النسخة الداخلية `__of_find_all_nodes` لا تُعدل الـ refcount، مخصصة لماكرو `for_each_of_allnodes`.

| المعامل | الوصف |
|---|---|
| `prev` | العقدة السابقة في التسلسل؛ `NULL` للبدء من الجذر |

**القيمة المُعادة:** العقدة التالية أو `NULL` عند نهاية الشجرة.

---

#### `of_find_node_by_name`

```c
struct device_node *of_find_node_by_name(struct device_node *from,
                                          const char *name);
```

يبحث عن أول عقدة بعد `from` يتطابق اسمها مع `name` (بـ `of_node_cmp` وهو `strcasecmp`). يُطلق `of_node_put(from)` أثناء البحث (ownership transfer).

| المعامل | الوصف |
|---|---|
| `from` | نقطة البداية؛ `NULL` للبدء من الأول — يُطلق refcount |
| `name` | الاسم المطلوب (حساس لحالة الأحرف بـ strcasecmp) |

**القيمة المُعادة:** مؤشر مع refcount مُزاد، أو `NULL`.

**ملاحظة:** استخدام `of_find_node_by_name` منتشر لكنه يبحث فقط في `node->name` لا في المسار الكامل.

---

#### `of_find_node_by_type`

```c
struct device_node *of_find_node_by_type(struct device_node *from,
                                          const char *type);
```

مثل `of_find_node_by_name` لكنه يطابق خاصية `device_type`. يُستخدم مثلاً للبحث عن عقدة `cpu` أو `memory`.

---

#### `of_find_compatible_node`

```c
struct device_node *of_find_compatible_node(struct device_node *from,
                                             const char *type,
                                             const char *compat);
```

يبحث عن عقدة تحمل `compatible` يحتوي على `compat` (بـ `strcasecmp`) وـ `device_type` يطابق `type` إذا كان غير NULL. النمط الأكثر استخداماً في driver initialization.

| المعامل | الوصف |
|---|---|
| `from` | نقطة البداية (ownership transfer)؛ `NULL` للبدء من الأول |
| `type` | نوع الجهاز (`NULL` لتجاهله) |
| `compat` | compatible string المطلوب |

**مثال عملي:**
```c
/* إيجاد عقدة GIC في الشجرة */
struct device_node *gic_node;
gic_node = of_find_compatible_node(NULL, NULL, "arm,gic-400");
if (gic_node) {
    /* استخدام العقدة */
    of_node_put(gic_node);
}
```

---

#### `of_find_matching_node_and_match`

```c
struct device_node *of_find_matching_node_and_match(
    struct device_node *from,
    const struct of_device_id *matches,
    const struct of_device_id **match);
```

يبحث بجدول `of_device_id` (نفس ما يستخدمه driver's `of_match_table`). يُعيد أيضاً المدخل المُطابق عبر `**match`.

| المعامل | الوصف |
|---|---|
| `from` | نقطة البداية (ownership transfer) |
| `matches` | جدول مُنتهٍ بـ entry فارغ |
| `match` | مؤشر لاستقبال المدخل المُطابق (يقبل NULL) |

**القيمة المُعادة:** مؤشر للعقدة المُطابقة مع refcount مُزاد، أو `NULL`.

---

#### `of_find_node_opts_by_path` و `of_find_node_by_path`

```c
struct device_node *of_find_node_opts_by_path(const char *path,
                                               const char **opts);
static inline struct device_node *of_find_node_by_path(const char *path);
```

يُحلل `path` ليجد العقدة المقابلة. يدعم مسارات كاملة مثل `/soc/uart@3f201000`، ومسارات alias مثل `serial0` (يُحلَّل عبر عقدة `/aliases`). إذا احتوى المسار على `:options` يُستخرج الجزء بعد `:` في `*opts`.

| المعامل | الوصف |
|---|---|
| `path` | مسار العقدة أو alias name |
| `opts` | مؤشر لاستقبال الجزء بعد `:` (يقبل NULL) |

**القيمة المُعادة:** العقدة أو `NULL`.

---

#### `of_find_node_by_phandle`

```c
struct device_node *of_find_node_by_phandle(phandle handle);
```

يبحث عن العقدة التي تحمل `phandle` == `handle`. يستخدم hash table داخلياً لأداء O(1). يُستدعى كثيراً من كود تحليل الـ phandles.

---

#### `of_get_parent`

```c
struct device_node *of_get_parent(const struct device_node *node);
```

يُعيد العقدة الأم مع رفع refcount. لا يأخذ ownership من `node`.

---

#### `of_get_next_parent`

```c
struct device_node *of_get_next_parent(struct device_node *node);
```

مثل `of_get_parent` لكنه يُطلق `of_node_put(node)` قبل الإعادة. يُستخدم للتنقل للأعلى في الشجرة مع cleanup تلقائي:

```c
/* مثال: صعود الشجرة حتى الجذر */
struct device_node *np = of_node_get(start);
while (np) {
    /* فحص np */
    np = of_get_next_parent(np); /* يُحرر السابق */
}
```

---

#### `of_get_next_child` و variants

```c
struct device_node *of_get_next_child(const struct device_node *node,
                                       struct device_node *prev);
struct device_node *of_get_next_child_with_prefix(const struct device_node *node,
                                                   struct device_node *prev,
                                                   const char *prefix);
struct device_node *of_get_next_available_child(const struct device_node *node,
                                                 struct device_node *prev);
struct device_node *of_get_next_reserved_child(const struct device_node *node,
                                                struct device_node *prev);
```

تُعيد الطفل التالي بعد `prev` (`NULL` للبدء). كل نسخة تُطلق `of_node_put(prev)` وتُزيد refcount النتيجة.

| النسخة | شرط الفلترة |
|---|---|
| `of_get_next_child` | كل الأطفال |
| `of_get_next_child_with_prefix` | اسم الطفل يبدأ بـ `prefix` |
| `of_get_next_available_child` | `status` ليس `"disabled"` |
| `of_get_next_reserved_child` | `status` == `"reserved"` |

---

#### `of_get_child_by_name` و `of_get_compatible_child`

```c
struct device_node *of_get_child_by_name(const struct device_node *node,
                                          const char *name);
struct device_node *of_get_compatible_child(const struct device_node *parent,
                                             const char *compatible);
struct device_node *of_get_available_child_by_name(const struct device_node *node,
                                                    const char *name);
```

بحث مباشر في أطفال عقدة واحدة فقط (غير recursive). `of_get_child_by_name` يطابق بـ `node->name`. `of_get_compatible_child` يطابق `compatible` property.

---

#### Traversal Macros

```c
/* التنقل عبر جميع الأطفال */
for_each_child_of_node(parent, child)

/* أطفال متاحون فقط */
for_each_available_child_of_node(parent, child)

/* مع scoped cleanup تلقائي */
for_each_child_of_node_scoped(parent, child)

/* أطفال بـ prefix محدد */
for_each_child_of_node_with_prefix(parent, child, prefix)

/* جميع عقد الشجرة */
for_each_of_allnodes(dn)

/* بحث بالاسم */
for_each_node_by_name(dn, name)

/* بحث بـ compatible */
for_each_compatible_node(dn, type, compatible)

/* بحث بجدول matches */
for_each_matching_node(dn, matches)

/* عقد CPU فقط */
for_each_of_cpu_node(cpu)
```

**تحذير:** ماكروهات `_scoped` تستخدم `__free(device_node)` من `cleanup.h` — تُطلق `of_node_put` تلقائياً عند خروج الـ scope. **لا** يجوز استخدام المتغير بعد الـ loop.

---

### الفئة 3: قراءة الـ Properties

هذه الفئة هي الأكثر استخداماً في driver code. جميع القيم في الـ FDT مُخزَّنة بـ big-endian؛ الدوال تُحول تلقائياً.

---

#### `of_find_property`

```c
struct property *of_find_property(const struct device_node *np,
                                   const char *name,
                                   int *lenp);
```

يبحث في قائمة `np->properties` عن property بالاسم. يُعيد مؤشراً للـ `struct property` مباشرة (لا نسخة) — البيانات تبقى ملكية الشجرة.

| المعامل | الوصف |
|---|---|
| `np` | العقدة المصدر |
| `name` | اسم الـ property (مطابقة حرفية بـ strcmp) |
| `lenp` | مؤشر لاستقبال الطول بالبايت (يقبل NULL) |

**القيمة المُعادة:** مؤشر لـ `struct property` أو `NULL`.

**الـ Locking:** تعمل تحت `of_property_read_lock` (spinlock) داخلياً في بعض الكودات، لكن الـ API العلوي يُقدم consistent view.

---

#### `of_get_property`

```c
const void *of_get_property(const struct device_node *node,
                              const char *name,
                              int *lenp);
```

اختصار لـ `of_find_property` يُعيد `prop->value` مباشرة. مناسب عند الحاجة لقيمة raw.

---

#### `of_property_read_bool`

```c
bool of_property_read_bool(const struct device_node *np, const char *propname);
```

يتحقق من وجود property (بغض النظر عن قيمتها). يُستخدم للـ boolean flags مثل `interrupt-controller` أو `dma-coherent`. يُعيد `true` إذا وُجدت الـ property حتى لو كانت فارغة (طول صفر).

---

#### `of_property_present`

```c
static inline bool of_property_present(const struct device_node *np, const char *propname);
```

wrapper حول `of_find_property` — معادل `of_property_read_bool` لكن أوضح دلالياً عند التحقق من وجود property لا قراءتها.

---

#### `of_property_count_elems_of_size`

```c
int of_property_count_elems_of_size(const struct device_node *np,
                                     const char *propname, int elem_size);
```

يُحسب عدد عناصر الـ property بافتراض أن كل عنصر بحجم `elem_size` بايت. يتحقق أن `prop->length` قابل للقسمة على `elem_size`.

**القيمة المُعادة:**
- عدد العناصر عند النجاح
- `-EINVAL` إذا لم تُوجد الـ property أو طولها لا يتوافق
- `-ENODATA` إذا كانت الـ property بدون قيمة

**الاختصارات:**
```c
of_property_count_u8_elems(np, propname)   /* elem_size = 1 */
of_property_count_u16_elems(np, propname)  /* elem_size = 2 */
of_property_count_u32_elems(np, propname)  /* elem_size = 4 */
of_property_count_u64_elems(np, propname)  /* elem_size = 8 */
```

---

#### `of_property_read_uXX_index`

```c
int of_property_read_u8_index(const struct device_node *np,
                               const char *propname, u32 index, u8 *out_value);
int of_property_read_u16_index(...);
int of_property_read_u32_index(...);
int of_property_read_u64_index(...);
```

يقرأ عنصراً واحداً بالفهرس من property. يُحوِّل من big-endian تلقائياً.

| المعامل | الوصف |
|---|---|
| `np` | العقدة المصدر |
| `propname` | اسم الـ property |
| `index` | فهرس العنصر (0-based) |
| `out_value` | مؤشر لكتابة القيمة (لا يُعدَّل عند الخطأ) |

**القيمة المُعادة:**
- `0` عند النجاح
- `-EINVAL` إذا لم تُوجد الـ property
- `-ENODATA` بدون قيمة
- `-EOVERFLOW` إذا تجاوز الفهرس الحدود

---

#### `of_property_read_variable_uXX_array`

```c
int of_property_read_variable_u32_array(const struct device_node *np,
                                         const char *propname,
                                         u32 *out_values,
                                         size_t sz_min, size_t sz_max);
```

يقرأ array بحجم متغير. يقبل أي عدد عناصر بين `sz_min` و `sz_max`. يُعيد عدد العناصر الفعلي المقروء. إذا كان `sz_max == 0` يُعامَل كـ `sz_min` (حجم ثابت).

**القيمة المُعادة:** عدد العناصر المقروءة (موجب) أو error code سالب.

---

#### `of_property_read_uXX_array` (fixed size)

```c
static inline int of_property_read_u32_array(const struct device_node *np,
                                              const char *propname,
                                              u32 *out_values, size_t sz);
```

wrapper حول `of_property_read_variable_u32_array` بـ `sz_max = 0`. يُعيد `0` أو error code. **لا يُعيد** عدد العناصر (يُحوَّل للـ 0 عند النجاح).

**مثال عملي:**
```c
u32 reg[2];
/* قراءة reg property: <base size> */
ret = of_property_read_u32_array(np, "reg", reg, 2);
if (ret) {
    dev_err(dev, "failed to read reg: %d\n", ret);
    return ret;
}
```

---

#### `of_property_read_u8/u16/u32/u64` (scalar)

```c
static inline int of_property_read_u32(const struct device_node *np,
                                        const char *propname,
                                        u32 *out_value);
```

اختصار لقراءة عنصر واحد. يستدعي `_array` مع `sz = 1`.

---

#### `of_property_read_u64`

```c
int of_property_read_u64(const struct device_node *np,
                          const char *propname, u64 *out_value);
```

قراءة قيمة 64-bit واحدة. تُعاد big-endian ← host endian.

---

#### `of_property_read_string`

```c
int of_property_read_string(const struct device_node *np,
                             const char *propname,
                             const char **out_string);
```

يقرأ أول string من property. يُعيد مؤشراً مباشراً لبيانات الـ FDT (لا نسخة). يُتحقق من أن الـ string منتهية بـ null ضمن حدود الـ property.

**القيمة المُعادة:**
- `0` عند النجاح
- `-EINVAL` إذا لم تُوجد الـ property
- `-ENODATA` بدون قيمة
- `-EILSEQ` إذا لم تكن منتهية بـ null

---

#### `of_property_read_string_index`

```c
static inline int of_property_read_string_index(const struct device_node *np,
                                                  const char *propname,
                                                  int index,
                                                  const char **output);
```

يقرأ string بفهرس محدد من property تحتوي على strings متعددة (null-separated). يستدعي `of_property_read_string_helper`.

---

#### `of_property_read_string_helper`

```c
int of_property_read_string_helper(const struct device_node *np,
                                    const char *propname,
                                    const char **out_strs, size_t sz, int index);
```

الدالة الداخلية الفعلية لتحليل multi-string properties. تتجاوز `index` strings في البداية ثم تقرأ `sz` strings. إذا كان `out_strs == NULL` تُعيد العدد الإجمالي للـ strings.

| المعامل | الوصف |
|---|---|
| `out_strs` | مصفوفة لاستقبال المؤشرات؛ NULL لحساب العدد |
| `sz` | عدد الـ strings المطلوبة |
| `index` | فهرس البداية |

---

#### `of_property_match_string`

```c
int of_property_match_string(const struct device_node *np,
                              const char *propname,
                              const char *string);
```

يبحث عن `string` في property متعددة الـ strings ويُعيد فهرسه. يُستخدم كثيراً مع `clock-names`, `reset-names`, `dma-names`.

**مثال عملي:**
```c
/* إيجاد رقم clock "apb" في clock-names */
int idx = of_property_match_string(np, "clock-names", "apb");
if (idx < 0)
    return idx;
clk = of_clk_get(np, idx);
```

---

#### `of_property_read_string_array` و `of_property_count_strings`

```c
static inline int of_property_read_string_array(const struct device_node *np,
                                                  const char *propname,
                                                  const char **out_strs, size_t sz);
static inline int of_property_count_strings(const struct device_node *np,
                                             const char *propname);
```

الأولى تقرأ حتى `sz` strings. الثانية تُعيد العدد الكلي.

---

#### `of_prop_next_u32` و `of_prop_next_string`

```c
const __be32 *of_prop_next_u32(const struct property *prop,
                                const __be32 *cur, u32 *pu);
const char *of_prop_next_string(const struct property *prop, const char *cur);
```

iterators لعناصر property. `NULL` كـ `cur` للبدء. يُستخدمان في ماكرو `of_property_for_each_u32` و `of_property_for_each_string`.

```c
/* مثال: طباعة جميع قيم property باسم "clocks" */
u32 val;
of_property_for_each_u32(np, "clocks", val)
    pr_info("clock phandle: %u\n", val);
```

---

#### Property Iteration Macros

```c
/* التنقل على properties عقدة */
for_each_property_of_node(dn, pp)

/* التنقل على u32 values */
of_property_for_each_u32(np, propname, u)

/* التنقل على strings */
of_property_for_each_string(np, propname, prop, s)
```

---

### الفئة 4: مطابقة الأجهزة

---

#### `of_device_is_compatible`

```c
int of_device_is_compatible(const struct device_node *device, const char *compat);
```

يتحقق إذا كانت `compatible` property للعقدة تحتوي على `compat`. يستخدم `of_compat_cmp` (strcasecmp افتراضياً). يُعيد درجة التطابق (integer > 0) لدعم أولوية المطابقة في قوائم compatible متعددة.

**القيمة المُعادة:** عدد موجب (درجة التطابق) أو `0` إذا لم يُطابق.

---

#### `of_device_compatible_match`

```c
int of_device_compatible_match(const struct device_node *device,
                                const char *const *compat);
```

يطابق قائمة strings (مُنتهية بـ NULL) ضد compatible property. يُعيد درجة التطابق للأفضل.

---

#### `of_device_is_available`

```c
bool of_device_is_available(const struct device_node *device);
```

يُعيد `true` إذا كانت `status` property غائبة أو `"okay"` أو `"ok"`. عقد بـ `status = "disabled"` تُعيد `false`. **يُراعى دائماً** في `of_get_next_available_child`.

---

#### `of_device_is_big_endian`

```c
bool of_device_is_big_endian(const struct device_node *device);
```

يتحقق من وجود `"big-endian"` property، أو `"native-endian"` مع CPU big-endian. يُستخدم لتحديد byte order الـ registers.

---

#### `of_match_node`

```c
const struct of_device_id *of_match_node(const struct of_device_id *matches,
                                          const struct device_node *node);
```

يُطابق عقدة بجدول `of_device_id`. يُقارن `name`, `type`, `compatible` حسب ما هو مُعرَّف في كل مدخل. يُعيد مؤشر المدخل المُطابق أو `NULL`.

**يُستخدم من:** `platform_driver` probe infrastructure، `of_platform_bus_probe()`.

---

#### `of_device_get_match_data`

```c
const void *of_device_get_match_data(const struct device *dev);
```

اختصار يُعيد حقل `data` من المدخل المُطابق في `driver->of_match_table`. مناسب لتحديد variant-specific data.

```c
/* مثال: تحديد بيانات خاصة بالـ SoC variant */
static const struct my_data soc_a_data = { .freq = 100 };
static const struct of_device_id my_ids[] = {
    { .compatible = "vendor,soc-a", .data = &soc_a_data },
    {}
};

static int my_probe(struct platform_device *pdev) {
    const struct my_data *data = of_device_get_match_data(&pdev->dev);
    /* data يُشير للـ soc_a_data */
}
```

---

#### `of_machine_is_compatible`

```c
static inline bool of_machine_is_compatible(const char *compat);
```

يُطابق `compatible` في root node (`/`) للتحقق من نوع الـ board. داخلياً يبني مصفوفة `{compat, NULL}` ويستدعي `of_machine_compatible_match`.

```c
/* مثال: تعديل سلوك على SoC محدد */
if (of_machine_is_compatible("raspberrypi,4-model-b"))
    enable_workaround();
```

---

#### `of_machine_compatible_match` و `of_machine_device_match`

```c
bool of_machine_compatible_match(const char *const *compats);
bool of_machine_device_match(const struct of_device_id *matches);
const void *of_machine_get_match_data(const struct of_device_id *matches);
```

نسخ أكثر مرونة: الأولى لقائمة strings، الثانية لجدول `of_device_id`، الثالثة تُعيد `data` من الجدول.

---

### الفئة 5: الـ Phandle — التحليل والـ Iterator

الـ **phandle** هو مؤشر رقمي (u32) يُعرِّف عقدة في الشجرة ويُستخدم للإشارة لها من عقد أخرى. هذه الآلية تُستخدم لربط المكونات (GPIO controllers, clock providers, interrupt controllers...).

---

#### `of_parse_phandle`

```c
static inline struct device_node *of_parse_phandle(const struct device_node *np,
                                                    const char *phandle_name,
                                                    int index);
```

يُحلل property تحتوي على phandle(s) ويُعيد العقدة المُشار إليها بالفهرس. يستدعي `__of_parse_phandle_with_args` مع `cell_count = 0`.

| المعامل | الوصف |
|---|---|
| `np` | العقدة التي تحتوي على الـ property |
| `phandle_name` | اسم الـ property |
| `index` | رقم الـ phandle في القائمة (0-based) |

**القيمة المُعادة:** مؤشر للعقدة مع refcount مُزاد، أو `NULL`. يجب `of_node_put()` لاحقاً.

---

#### `of_parse_phandle_with_args`

```c
static inline int of_parse_phandle_with_args(const struct device_node *np,
                                              const char *list_name,
                                              const char *cells_name,
                                              int index,
                                              struct of_phandle_args *out_args);
```

يُحلل phandle مع arguments ديناميكية. عدد الـ arguments يُستخرج من property `cells_name` في العقدة المُشار إليها (مثل `#gpio-cells`, `#clock-cells`).

| المعامل | الوصف |
|---|---|
| `list_name` | property تحتوي على قائمة `<&phandle args...>` |
| `cells_name` | اسم property عدد الـ cells مثل `"#gpio-cells"` |
| `index` | رقم الـ phandle في القائمة |
| `out_args` | مؤشر لبنية الخرج (np + args_count + args[]) |

**مثال:**
```c
/* DTS:
 * gpios = <&gpio0 5 GPIO_ACTIVE_LOW &gpio1 3 GPIO_ACTIVE_HIGH>;
 * #gpio-cells = <2>;
 */
struct of_phandle_args args;
ret = of_parse_phandle_with_args(np, "gpios", "#gpio-cells", 0, &args);
/* args.np → gpio0, args.args[0] = 5, args.args[1] = GPIO_ACTIVE_LOW */
of_node_put(args.np);
```

---

#### `of_parse_phandle_with_fixed_args`

```c
static inline int of_parse_phandle_with_fixed_args(const struct device_node *np,
                                                    const char *list_name,
                                                    int cell_count,
                                                    int index,
                                                    struct of_phandle_args *out_args);
```

نفس الوظيفة لكن عدد الـ arguments ثابت بـ `cell_count` بدلاً من قراءته من العقدة. يُستخدم عند غياب `#cells` property.

---

#### `of_parse_phandle_with_optional_args`

```c
static inline int of_parse_phandle_with_optional_args(const struct device_node *np,
                                                       const char *list_name,
                                                       const char *cells_name,
                                                       int index,
                                                       struct of_phandle_args *out_args);
```

إذا لم تُوجد `cells_name` property، يُفترض 0 arguments. مفيد للـ backward compatibility عند إضافة arguments لـ phandle موجود.

---

#### `__of_parse_phandle_with_args`

```c
int __of_parse_phandle_with_args(const struct device_node *np,
                                  const char *list_name,
                                  const char *cells_name,
                                  int cell_count,
                                  int index,
                                  struct of_phandle_args *out_args);
```

الدالة الأساسية التي تستدعيها الـ wrappers السابقة. `cell_count == -1` يعني: اقرأ عدد الـ cells من `cells_name`؛ `cell_count >= 0` يعني: استخدم هذا العدد مباشرة.

**Pseudocode Flow:**
```
parse_phandle_list(np, list_name, cells_name, cell_count, index, out_args):
    prop = of_find_property(np, list_name)
    cur = prop->value

    for each entry in list:
        phandle = be32_to_cpu(*cur++)
        node = of_find_node_by_phandle(phandle)

        if cell_count == -1:
            cells = of_get_property(node, cells_name) → n
        else:
            cells = cell_count

        if current_index == index:
            out_args->np = node
            out_args->args_count = cells
            copy cells values → out_args->args[]
            return 0

        cur += cells  /* تخطي هذا الـ entry */

    return -ENOENT
```

---

#### `of_count_phandle_with_args`

```c
int of_count_phandle_with_args(const struct device_node *np,
                                const char *list_name,
                                const char *cells_name);
```

يُعيد عدد الـ phandles في قائمة. يُستخدم قبل حلقة `of_parse_phandle_with_args`.

---

#### `of_parse_phandle_with_args_map`

```c
int of_parse_phandle_with_args_map(const struct device_node *np,
                                    const char *list_name,
                                    const char *stem_name,
                                    int index,
                                    struct of_phandle_args *out_args);
```

نسخة متقدمة تدعم **mapping** عبر properties مثل `interrupt-map`. يستخدم `stem_name` لبناء أسماء `map-mask`, `map`, `specifier-cells` وفق نمط الـ interrupt nexus.

---

#### `of_phandle_args_equal`

```c
static inline bool of_phandle_args_equal(const struct of_phandle_args *a1,
                                          const struct of_phandle_args *a2);
```

مقارنة بنيتي `of_phandle_args` (نفس العقدة، نفس العدد، نفس الـ args). يُستخدم للتحقق من التكرار.

---

#### Phandle Iterator API

```c
int of_phandle_iterator_init(struct of_phandle_iterator *it,
                              const struct device_node *np,
                              const char *list_name,
                              const char *cells_name,
                              int cell_count);

int of_phandle_iterator_next(struct of_phandle_iterator *it);

int of_phandle_iterator_args(struct of_phandle_iterator *it,
                              uint32_t *args, int size);
```

**`of_phandle_iterator_init`:** يُهيئ الـ iterator بدون المرور على أي مدخل. يُخزَّن المؤشر على بداية القائمة في `it->cur`.

**`of_phandle_iterator_next`:** يتقدم للمدخل التالي. يُطلق `of_node_put` على العقدة الحالية ويُحمِّل التالية. يُعيد `-ENOENT` عند نهاية القائمة، `-EINVAL` عند خطأ.

**`of_phandle_iterator_args`:** ينسخ arguments المدخل الحالي لمصفوفة `args` (بحد أقصى `size`). يُعيد عدد الـ args المنسوخة.

**ماكرو الاستخدام:**
```c
struct of_phandle_iterator it;
int err;

of_for_each_phandle(&it, err, np, "clocks", "#clock-cells", -1) {
    /* it.node هي العقدة الحالية */
    /* it.cur_count عدد الـ args */
    of_phandle_iterator_args(&it, args, ARRAY_SIZE(args));
}
```

---

### الفئة 6: عقد الـ CPU

---

#### `of_get_cpu_node`

```c
struct device_node *of_get_cpu_node(int cpu, unsigned int *thread);
```

يبحث عن عقدة DT المقابلة لـ logical CPU رقم `cpu`. يتحقق من `reg` property ويُقارن بـ `cpu_logical_map`. يدعم multi-threading عبر `thread`.

---

#### `of_cpu_device_node_get`

```c
struct device_node *of_cpu_device_node_get(int cpu);
```

يُعيد عقدة CPU مع refcount مُزاد. wrapper بسيط حول `of_get_cpu_node`.

---

#### `of_cpu_node_to_id`

```c
int of_cpu_node_to_id(struct device_node *np);
```

التحويل العكسي: من عقدة DT لرقم CPU منطقي. يُعيد `-ENODEV` إذا لم يُوجد CPU مقابل.

---

#### `of_get_next_cpu_node`

```c
struct device_node *of_get_next_cpu_node(struct device_node *prev);
```

يتنقل عبر عقد `cpu` في `/cpus`. يُستخدم مع ماكرو `for_each_of_cpu_node`.

---

#### `of_get_cpu_state_node`

```c
struct device_node *of_get_cpu_state_node(const struct device_node *cpu_node, int index);
```

يُعيد عقدة حالة طاقة CPU (idle-states, power-state) بالفهرس. يبحث في `cpu_node->idle-states` phandle list.

---

#### `of_get_cpu_hwid`

```c
u64 of_get_cpu_hwid(struct device_node *cpun, unsigned int thread);
```

يقرأ MPIDR أو hardware ID من `reg` property للـ CPU. يدعم threads متعددة.

---

### الفئة 7: الـ Address Cells وقراءة الأعداد

---

#### `of_read_number`

```c
static inline u64 of_read_number(const __be32 *cell, int size)
```

يقرأ عدداً big-endian بحجم `size` cells (كل cell = 4 bytes). يُجمِّع الـ cells بترتيب big-endian لتكوين u64.

```c
/* قراءة u64 من خليتين */
u64 val = of_read_number(prop_value, 2);
/* val = ((u64)cell[0] << 32) | cell[1] */
```

---

#### `of_n_addr_cells` و `of_n_size_cells`

```c
int of_n_addr_cells(struct device_node *np);
int of_n_size_cells(struct device_node *np);
```

يُعيدان قيمة `#address-cells` و `#size-cells` للعقدة الأم. إذا لم تُوجد، يصعدان في الشجرة حتى يجدا القيمة. القيمة الافتراضية: 1 لـ address cells، 0 لـ size cells.

**مهم لـ:** تحليل `reg` property في driver code.

---

#### `of_alias_get_id` و `of_alias_get_highest_id`

```c
int of_alias_get_id(const struct device_node *np, const char *stem);
int of_alias_get_highest_id(const char *stem);
```

`of_alias_get_id` يُعيد الرقم من alias مثل `serial0` → 0، `i2c2` → 2. يبحث في عقدة `/aliases`.

`of_alias_get_highest_id` يُعيد أعلى رقم مستخدم لـ stem معين.

```c
/* مثال: إيجاد رقم UART في aliases */
int id = of_alias_get_id(uart_node, "serial");
/* DTS: aliases { serial0 = &uart0; } → id = 0 */
```

---

#### `of_alias_from_compatible`

```c
int of_alias_from_compatible(const struct device_node *node, char *alias, int len);
```

ينتج alias string من compatible string (يأخذ آخر جزء بعد `/`). يُستخدم لتوليد device name.

---

#### `of_map_id`

```c
int of_map_id(const struct device_node *np, u32 id,
               const char *map_name, const char *map_mask_name,
               struct device_node **target, u32 *id_out);
```

يُحوِّل ID (مثل PCIe RID أو IOMMU stream ID) عبر `map` property. يتبع مواصفات `iommu-map`, `msi-map`. يُعيد العقدة الهدف والـ ID المُحوَّل.

---

#### `of_dma_get_max_cpu_address`

```c
phys_addr_t of_dma_get_max_cpu_address(struct device_node *np);
```

يُعيد أقصى عنوان CPU يمكن الوصول إليه عبر DMA، مستنتجاً من `dma-ranges` في الشجرة. يُستخدم لتحديد حدود CMA allocations.

---

### الفئة 8: التعديل الديناميكي — Runtime Tree Modification

هذه الدوال مشروطة بـ `CONFIG_OF_DYNAMIC` وتُعدِّل الشجرة الحية مع إصدار أحداث `OF_RECONFIG_*` للـ notifiers.

---

#### `of_add_property`

```c
int of_add_property(struct device_node *np, struct property *prop);
```

يُضيف property جديدة لعقدة حية. يأخذ `of_mutex` ويُضيف الـ property لقائمة `np->properties`، ثم يُرسل حدث `OF_RECONFIG_ADD_PROPERTY`.

**تحذير:** `prop` يجب أن يكون allocated بـ `kzalloc` ومملوكاً للشجرة بعد الاستدعاء.

---

#### `of_remove_property`

```c
int of_remove_property(struct device_node *np, struct property *prop);
```

ينقل الـ property من `np->properties` لـ `np->deadprops` (لا يُحرر الذاكرة فوراً). يُرسل `OF_RECONFIG_REMOVE_PROPERTY`.

---

#### `of_update_property`

```c
int of_update_property(struct device_node *np, struct property *newprop);
```

يستبدل property موجودة بـ `newprop`. القديمة تنتقل لـ `deadprops`. يُرسل `OF_RECONFIG_UPDATE_PROPERTY`.

---

#### `of_attach_node` و `of_detach_node`

```c
int of_attach_node(struct device_node *);
int of_detach_node(struct device_node *);
```

`of_attach_node`: يُضيف عقدة جديدة للشجرة (كطفل لـ parent المُعيَّن في العقدة). يُعيِّن `OF_DETACHED` flag ويُرسل `OF_RECONFIG_ATTACH_NODE`.

`of_detach_node`: يُزيل العقدة من الشجرة ويُعيِّن `OF_DETACHED` flag. يُرسل `OF_RECONFIG_DETACH_NODE`.

---

### الفئة 9: الـ Changeset — تعديلات Atomic

الـ **changeset** يسمح بتطبيق مجموعة من التعديلات على الشجرة بشكل atomic مع إمكانية التراجع الكامل.

```
of_changeset:
┌────────────────┐
│   entries list │ ← list_head
│  ┌──────────┐  │
│  │ action 1 │  │ (ATTACH_NODE)
│  ├──────────┤  │
│  │ action 2 │  │ (ADD_PROPERTY)
│  ├──────────┤  │
│  │ action 3 │  │ (ADD_PROPERTY)
│  └──────────┘  │
└────────────────┘
        ↓ of_changeset_apply()
   شجرة مُعدَّلة
        ↓ of_changeset_revert()
   شجرة أصلية
```

---

#### `of_changeset_init`

```c
void of_changeset_init(struct of_changeset *ocs);
```

تهيئة changeset فارغ. يُهيئ `list_head entries`.

---

#### `of_changeset_destroy`

```c
void of_changeset_destroy(struct of_changeset *ocs);
```

يُحرر كل موارد الـ changeset. يجب استدعاؤه بعد `apply` أو عند التخلي عن الـ changeset.

---

#### `of_changeset_apply`

```c
int of_changeset_apply(struct of_changeset *ocs);
```

يُطبق جميع الـ entries بالترتيب. عند فشل أي entry، يتراجع عما سبق ويُعيد error code. تحت `of_mutex`.

**Pseudocode:**
```
for each entry in ocs->entries:
    ret = of_changeset_action_apply(entry)
    if ret != 0:
        revert all previous entries
        return ret
return 0
```

---

#### `of_changeset_revert`

```c
int of_changeset_revert(struct of_changeset *ocs);
```

يتراجع عن كل التعديلات بالترتيب العكسي. يُعيد الشجرة لحالتها قبل `apply`.

---

#### `of_changeset_action`

```c
int of_changeset_action(struct of_changeset *ocs,
                         unsigned long action,
                         struct device_node *np,
                         struct property *prop);
```

يُضيف action واحداً لقائمة الـ changeset (لا يُطبقه فوراً). `action` من: `OF_RECONFIG_ATTACH_NODE`, `OF_RECONFIG_DETACH_NODE`, `OF_RECONFIG_ADD_PROPERTY`, `OF_RECONFIG_REMOVE_PROPERTY`, `OF_RECONFIG_UPDATE_PROPERTY`.

---

#### Helper Wrappers للـ Changeset

```c
/* إلحاق/فصل عقدة */
of_changeset_attach_node(ocs, np)
of_changeset_detach_node(ocs, np)

/* إضافة/إزالة/تحديث property */
of_changeset_add_property(ocs, np, prop)
of_changeset_remove_property(ocs, np, prop)
of_changeset_update_property(ocs, np, prop)

/* إنشاء عقدة جديدة */
of_changeset_create_node(ocs, parent, full_name)

/* إضافة properties بأنواع مختلفة */
of_changeset_add_prop_string(ocs, np, prop_name, str)
of_changeset_add_prop_string_array(ocs, np, prop_name, str_array, sz)
of_changeset_add_prop_u32_array(ocs, np, prop_name, array, sz)
of_changeset_add_prop_u32(ocs, np, prop_name, val)
of_changeset_add_prop_bool(ocs, np, prop_name)
of_changeset_update_prop_string(ocs, np, prop_name, str)
```

**مثال كامل لإنشاء عقدة ديناميكية:**
```c
struct of_changeset ocs;
struct device_node *new_node;

of_changeset_init(&ocs);

/* إنشاء عقدة جديدة */
new_node = of_changeset_create_node(&ocs, parent, "my-device@0");
of_changeset_add_prop_string(&ocs, new_node, "compatible", "vendor,my-dev");
of_changeset_add_prop_u32(&ocs, new_node, "reg", 0x0);
of_changeset_add_prop_bool(&ocs, new_node, "interrupt-controller");

/* تطبيق التعديلات */
ret = of_changeset_apply(&ocs);
if (ret)
    goto cleanup;

/* ... استخدام العقدة ... */

/* التراجع عند الحاجة */
of_changeset_revert(&ocs);
cleanup:
    of_changeset_destroy(&ocs);
```

---

### الفئة 10: الـ Reconfig Notifiers

---

#### `of_reconfig_notifier_register` و `of_reconfig_notifier_unregister`

```c
int of_reconfig_notifier_register(struct notifier_block *);
int of_reconfig_notifier_unregister(struct notifier_block *);
```

يُسجِّل/يُلغي تسجيل notifier لأحداث تعديل الشجرة. يُستخدم من `of_platform` لاكتشاف إضافة/إزالة العقد وإنشاء/تدمير platform devices تلقائياً.

---

#### `of_reconfig_notify`

```c
int of_reconfig_notify(unsigned long action, struct of_reconfig_data *rd);
```

يُصدر حدث reconfig لكل المُسجَّلين. `action` من `OF_RECONFIG_*` constants. `rd` يحمل العقدة والـ property المُتأثرة.

---

#### `of_reconfig_get_state_change`

```c
int of_reconfig_get_state_change(unsigned long action, struct of_reconfig_data *arg);
```

يُحوِّل action خام (`OF_RECONFIG_ATTACH_NODE`, etc.) لقيمة `of_reconfig_change` enum:
- `OF_RECONFIG_CHANGE_ADD`: إضافة عقدة/property
- `OF_RECONFIG_CHANGE_REMOVE`: إزالة
- `OF_RECONFIG_NO_CHANGE`: لا تغيير (تحديث في نفس المكان)

---

### الفئة 11: الـ Overlay

---

#### `of_overlay_fdt_apply`

```c
int of_overlay_fdt_apply(const void *overlay_fdt, u32 overlay_fdt_size,
                          int *ovcs_id, const struct device_node *target_base);
```

يُطبق FDT overlay على الشجرة الحية. يُحوِّل الـ FDT الخام لتغييرات changeset ويُطبقها. `*ovcs_id` يُستخدم لاحقاً في `of_overlay_remove`.

| المعامل | الوصف |
|---|---|
| `overlay_fdt` | بيانات FDT الخام (blob) |
| `overlay_fdt_size` | حجمها بالبايت |
| `ovcs_id` | معرّف الـ overlay للإزالة لاحقاً |
| `target_base` | عقدة أساس للتطبيق (NULL للجذر) |

**القيمة المُعادة:** `0` عند النجاح، أو error code.

---

#### `of_overlay_remove`

```c
int of_overlay_remove(int *ovcs_id);
```

يُزيل overlay بمعرِّفه. يتراجع عن التغييرات ويُحرر الموارد. يُعيِّن `*ovcs_id` إلى صفر عند النجاح.

---

#### `of_overlay_remove_all`

```c
int of_overlay_remove_all(void);
```

يُزيل جميع الـ overlays بالترتيب العكسي (LIFO).

---

#### `of_overlay_notifier_register` و `of_overlay_notifier_unregister`

```c
int of_overlay_notifier_register(struct notifier_block *nb);
int of_overlay_notifier_unregister(struct notifier_block *nb);
```

تسجيل/إلغاء تسجيل notifier لأحداث overlay. يُستدعى الـ notifier بقيم `of_overlay_notify_action`:

| القيمة | التوقيت |
|---|---|
| `OF_OVERLAY_PRE_APPLY` | قبل التطبيق |
| `OF_OVERLAY_POST_APPLY` | بعد التطبيق |
| `OF_OVERLAY_PRE_REMOVE` | قبل الإزالة |
| `OF_OVERLAY_POST_REMOVE` | بعد الإزالة |

---

### الفئة 12: Module والـ Console والـ Kexec

---

#### `of_modalias`

```c
ssize_t of_modalias(const struct device_node *np, char *str, ssize_t len);
```

يُوّلد modalias string من compatible strings للعقدة. الصيغة: `of:NnameT<type>C<compat>`. يُستخدم من `of_device_uevent()` لإرسال uevent للـ udev.

---

#### `of_request_module`

```c
int of_request_module(const struct device_node *np);
```

يستدعي `request_module()` بالـ modalias المُشتق من العقدة. يُستخدم لتحميل module تلقائياً.

---

#### `of_console_check`

```c
bool of_console_check(const struct device_node *dn, char *name, int index);
```

يتحقق إذا كانت العقدة هي console المُختار (المُعيَّن في `stdout-path` في `/chosen`). إذا طابق، يُسجِّل `name@index` كـ preferred console.

---

#### `of_kexec_alloc_and_setup_fdt`

```c
void *of_kexec_alloc_and_setup_fdt(const struct kimage *image,
                                    unsigned long initrd_load_addr,
                                    unsigned long initrd_len,
                                    const char *cmdline,
                                    size_t extra_fdt_size);
```

يُنشئ FDT معدَّلاً لاستخدامه في kexec. يُضيف `linux,initrd-start/end`، `bootargs`، وإعدادات kexec الأخرى. يُخصِّص ذاكرة لـ FDT الناتج.

---

#### `of_node_to_nid`

```c
int of_node_to_nid(struct device_node *np);
```

يُعيد رقم الـ NUMA node المقابل للعقدة (يقرأ من DT أو يحسبه من العنوان). يُعيد `NUMA_NO_NODE` عند عدم توفر معلومات NUMA.

---

### الفئة 13: الـ Declaration Macros

```c
/* إعلان handler مرتبط بـ compatible في جدول OF */
OF_DECLARE_1(table, name, compat, fn)     /* fn: void fn(struct device_node *) */
OF_DECLARE_1_RET(table, name, compat, fn) /* fn: int fn(struct device_node *) */
OF_DECLARE_2(table, name, compat, fn)     /* fn: int fn(node, node) */
```

تُنشئ `of_device_id` entry في section خاص (`__<table>_of_table`). يُستخدم من `irqchip_of_match_node()` و `clocksource_of_init()` لاستدعاء handlers تلقائياً أثناء boot.

**مثال:**
```c
static void __init my_irq_init(struct device_node *np,
                                struct device_node *parent)
{
    /* تهيئة IRQ controller */
}

IRQCHIP_DECLARE(my_irq, "vendor,my-irq", my_irq_init);
/* ينتهي إلى: OF_DECLARE_2(irqchip, my_irq, "vendor,my-irq", my_irq_init) */
```

---

### ملاحظات Locking عامة

| السياق | الـ Lock المستخدم |
|---|---|
| قراءة properties | `of_property_read_lock` (read-side, RCU-like) |
| تعديل الشجرة | `of_mutex` (mutex) |
| changeset apply/revert | `of_mutex` |
| flag operations | atomic bit ops |
| refcount | atomic (kobject/kref) |

**قاعدة عامة:** دوال `of_find_*` و `of_get_*` آمنة من أي سياق (process/interrupt) عند قراءة شجرة static. عند الشجرة الديناميكية، يجب الحذر من الـ races عند التعديل.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ Device Tree

الـ **debugfs** يوفر شجرة كاملة للـ Device Tree يمكن تصفحها مباشرة من userspace:

| المسار | الوصف | كيفية القراءة |
|--------|-------|--------------|
| `/sys/kernel/debug/of_fwnode/` | كل `fwnode_handle` مسجّل | `ls` ثم `cat` على كل ملف |
| `/sys/kernel/debug/device_component/` | تبعيات الأجهزة الـ DT-based | `cat master` أو `cat client` |
| `/sys/kernel/debug/gpio` | GPIO pins المرتبطة بنودات DT | `cat /sys/kernel/debug/gpio` |
| `/sys/kernel/debug/clk/` | شجرة الـ clocks المستخرجة من DT | `cat /sys/kernel/debug/clk/clk_summary` |
| `/sys/kernel/debug/regulator/` | regulators المعرّفة في DT | `cat /sys/kernel/debug/regulator/regulator_summary` |

```bash
# عرض كامل لشجرة الـ fwnodes
ls /sys/kernel/debug/of_fwnode/

# اقرأ خصائص node معين عبر debugfs
cat /sys/kernel/debug/clk/clk_summary
```

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ Open Firmware / Device Tree

الـ **sysfs** يعكس `device_node` و `property` مباشرةً عبر `kobject` (عند تفعيل `CONFIG_OF_KOBJ`):

| المسار | الوصف |
|--------|-------|
| `/sys/firmware/devicetree/base/` | الشجرة الكاملة للـ DTB المحمّلة |
| `/sys/firmware/devicetree/base/compatible` | compatible للـ root node |
| `/sys/firmware/devicetree/base/<node>/` | كل node وخصائصه |
| `/sys/firmware/fdt` | الـ DTB الخام (binary) كاملاً |
| `/sys/bus/platform/devices/<dev>/of_node` | symlink لنود الـ DT الخاص بالجهاز |
| `/sys/devices/.../of_node/` | خصائص الجهاز كملفات binary |

```bash
# عرض الـ compatible للجذر (root node)
cat /sys/firmware/devicetree/base/compatible

# قراءة property من node معين
xxd /sys/firmware/devicetree/base/cpus/cpu@0/clock-frequency

# تتبع of_node لجهاز platform
ls -la /sys/bus/platform/devices/fe300000.i2c/of_node/

# قراءة كل خصائص node
for f in /sys/firmware/devicetree/base/some_node/*; do
    echo "=== $(basename $f) ==="; xxd "$f" 2>/dev/null || cat "$f"; done
```

---

#### 3. الـ ftrace: Tracepoints وأحداث الـ OF

الـ **ftrace** يوفر tracepoints لمتابعة عمليات الـ Device Tree في وقت التشغيل:

```bash
# تفعيل tracing لكل أحداث of
echo 1 > /sys/kernel/debug/tracing/events/of/enable

# أو تفعيل أحداث محددة
echo 1 > /sys/kernel/debug/tracing/events/of/of_reconfig_notify/enable

# تتبع استدعاءات دوال OF الأساسية بـ function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo "of_find_node_by_path of_property_read_u32 of_parse_phandle" \
    > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# تتبع استدعاء of_changeset_apply والمتفرعات منه
echo of_changeset_apply > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace
```

أحداث مفيدة تحديداً لهذا الـ subsystem:

| Event | الغرض |
|-------|-------|
| `of_reconfig_notify` | تغيير حدث في شجرة الـ DT الحية |
| `of_overlay_fdt_apply` | تطبيق overlay جديد |
| `of_changeset_apply` | تطبيق changeset على الشجرة |

---

#### 4. الـ printk والـ Dynamic Debug

تفعيل رسائل الـ debug لنظام الـ OF:

```bash
# تفعيل dynamic debug لكل ملفات OF
echo "file drivers/of/*.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file include/linux/of.h +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ overlay فقط
echo "file drivers/of/overlay.c +pflmt" \
    > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ changeset
echo "file drivers/of/dynamic.c +p" \
    > /sys/kernel/debug/dynamic_debug/control

# رفع loglevel مؤقتاً لمشاهدة رسائل KERN_DEBUG
echo 8 > /proc/sys/kernel/printk

# مراقبة kernel log في الوقت الفعلي
dmesg -w | grep -i "OF\|devicetree\|phandle\|overlay"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_OF` | تفعيل نظام OF الأساسي |
| `CONFIG_OF_DYNAMIC` | دعم تعديل الشجرة في وقت التشغيل + refcounting |
| `CONFIG_OF_OVERLAY` | دعم تطبيق DT overlays |
| `CONFIG_OF_KOBJ` | ربط nodes بـ kobjects → ظهورها في sysfs |
| `CONFIG_OF_PROMTREE` | دعم PROM tree الأصلية |
| `CONFIG_OF_NUMA` | ربط NUMA topology بالـ DT |
| `CONFIG_OF_DEBUG` | تفعيل رسائل debug إضافية في OF core |
| `CONFIG_DEBUG_KOBJECT` | debugging لـ kobjects المرتبطة بـ device_node |
| `CONFIG_DEBUG_FS` | تفعيل debugfs لمشاهدة مدخلاته |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug لرسائل `pr_debug` في OF |
| `CONFIG_FTRACE` | تفعيل function tracing |
| `CONFIG_PROVE_LOCKING` | كشف مشاكل locking في of_mutex |
| `CONFIG_DEBUG_OBJECTS_WORK` | تتبع leaks في of_node refcount |
| `CONFIG_KASAN` | كشف memory corruption في structs OF |
| `CONFIG_UBSAN` | كشف undefined behavior في of_read_number |

```bash
# فحص الـ configs الفعّالة في kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_OF|CONFIG_DEBUG_FS|CONFIG_DYNAMIC_DEBUG"
```

---

#### 6. أدوات الـ Subsystem: dtc و fdtdump و of-cli

```bash
# تفريغ الـ DTB المحمّل كـ DTS مقروء
dtc -I dtb -O dts /sys/firmware/fdt -o /tmp/current.dts

# فحص DTB قبل التحميل
fdtdump /boot/dtb/board.dtb | head -100

# البحث عن node بـ compatible معين
dtc -I dtb -O dts /sys/firmware/fdt | grep -A5 "compatible.*my-driver"

# استخدام of-cli (إذا متوفر)
of-cli /sys/firmware/devicetree/base/

# فحص الـ phandle references
grep -r "phandle" /sys/firmware/devicetree/base/ 2>/dev/null | \
    xxd | head -20
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الإصلاح |
|-------------|--------|---------|
| `OF: ERROR: Bad cell count for /node` | عدد cells في property لا يتطابق مع `#xxx-cells` | تصحيح `#xxx-cells` في DTS |
| `OF: /node: could not get #address-cells` | غياب `#address-cells` في parent | إضافة `#address-cells = <1>` للـ parent |
| `OF: fdt: not a valid FDT magic` | الـ DTB تالف أو لم يُحمَّل | تحقق من bootloader argument |
| `of_parse_phandle: ... phandle not found` | phandle مشار إليه غير موجود | تحقق من `&label` في DTS |
| `OF: overlay: property is missing` | property مطلوبة في overlay مفقودة | أضف الـ property للـ overlay |
| `OF: changeset: apply failed` | فشل تطبيق changeset — rollback تلقائي | تحقق من dmesg لأسباب الفشل |
| `of_property_read_u32: -EINVAL` | الـ property غير موجودة | تأكد من اسم الـ property في DTS |
| `of_property_read_u32: -ENODATA` | الـ property موجودة لكن فارغة | أضف قيمة صحيحة للـ property |
| `of_property_read_u32: -EOVERFLOW` | البيانات أصغر من المتوقع | تحقق من حجم الـ array في DTS |
| `of_node_put: refcount underflow` | استدعاء `of_node_put()` زائد عن اللازم | مراجعة منطق get/put |
| `OF: overlay: target of overlay not found` | target_base غير موجود | تحقق من الـ target-path في overlay |
| `platform: deferred probe pending` | node في DT لكن dependency لم تجهز | مراجعة سلسلة الـ probe وترتيب التهيئة |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في of_parse_phandle_with_args — تحقق من صحة args_count */
if (WARN_ON(out_args->args_count > MAX_PHANDLE_ARGS)) {
    dump_stack();
    return -EINVAL;
}

/* في of_node_put — كشف refcount underflow */
if (WARN_ON(kref_read(&node->kobj.kref) == 0)) {
    dump_stack();
}

/* في of_changeset_apply — فشل غير متوقع */
ret = of_changeset_apply(&ocs);
if (WARN_ON(ret)) {
    pr_err("changeset apply failed: %d\n", ret);
    dump_stack();
}

/* في of_find_node_by_path — تحقق من initialization */
if (WARN_ON(!of_root)) {
    pr_err("OF tree not initialized\n");
    dump_stack();
}

/* تتبع تسرب of_node_get بدون of_node_put */
struct device_node *np __free(device_node) =
    of_find_node_by_path("/soc/i2c@fe300000");
if (WARN_ON(!np))
    return -ENODEV;
/* np يُحرَّر تلقائياً عند نهاية الـ scope */
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

```bash
# مقارنة base address في DTS مع التوقيع الفعلي للـ hardware
# مثال: I2C controller في Raspberry Pi
cat /sys/firmware/devicetree/base/soc/i2c@fe300000/reg | xxd
# يجب أن يعود: 0xfe300000 بصيغة big-endian

# تحقق من clock-frequency الفعلي مقابل DTS
cat /sys/firmware/devicetree/base/soc/i2c@fe300000/clock-frequency | xxd

# التحقق من interrupts
cat /sys/firmware/devicetree/base/soc/i2c@fe300000/interrupts | xxd

# مقارنة عدد الـ resources المخصصة للجهاز
cat /proc/iomem | grep -i "fe300000"
cat /proc/interrupts | grep i2c
```

---

#### 2. قراءة الـ Registers مباشرةً (Register Dump)

```bash
# قراءة register بـ devmem2 (مثال: I2C controller)
devmem2 0xfe300000 w    # قراءة كلمة 32-bit من base address
devmem2 0xfe300004 w    # السجل التالي

# استخدام /dev/mem مع xxd (يتطلب CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=16 skip=$((0xfe300000/4)) 2>/dev/null | xxd

# io utility (من package iotools)
io -4 -r 0xfe300000     # read 32-bit register

# قراءة block كامل من registers
dd if=/dev/mem of=/tmp/regs.bin bs=1 count=256 skip=$((0xfe300000)) 2>/dev/null
xxd /tmp/regs.bin

# التحقق من قيم MMIO عبر /proc/iomem أولاً
cat /proc/iomem | grep -i "i2c\|uart\|spi"
```

**تحذير:** تتطلب هذه العمليات `CONFIG_DEVMEM=y` وصلاحيات root، وقد تسبب crash إذا كانت الـ addresses خاطئة.

---

#### 3. نصائح Logic Analyzer / Oscilloscope

عند مقارنة DTS مع سلوك الـ hardware الفعلي:

```
مثال عملي: تشخيص I2C يُعرَّف في DTS بـ clock-frequency = <400000>

Oscilloscope:
  ┌─────────────────────────────────┐
  │ قس الـ SCL period على الـ scope │
  │ Period = 2.5µs → 400kHz ✓       │
  │ Period = 10µs  → 100kHz ✗        │
  └─────────────────────────────────┘

Logic Analyzer:
  - تحقق من timing بين SDA/SCL
  - قارن الـ address المُرسَل مع reg في DTS
  - راقب ACK/NACK بعد كل byte
```

| إشارة | ما تكشفه في سياق DT |
|-------|---------------------|
| SCL frequency | تطابق `clock-frequency` property |
| SPI CPOL/CPHA | تطابق `spi-cpol` و`spi-cpha` في DTS |
| UART baud | تطابق `current-speed` property |
| GPIO polarity | تطابق `GPIO_ACTIVE_LOW` flag |
| Reset GPIO timing | تطابق `reset-duration-us` property |

---

#### 4. مشاكل الـ Hardware الشائعة ومؤشراتها في الـ Kernel Log

| مشكلة الـ Hardware | النمط في dmesg | السبب المحتمل في DT |
|---------------------|----------------|---------------------|
| Base address خاطئ | `ioremap failed` أو kernel panic | `reg` property خاطئة |
| IRQ رقم خاطئ | `irq: no irq domain found` | `interrupts` property خاطئة |
| Clock مفقود | `clk_get failed: -ENOENT` | `clocks` أو `clock-names` خاطئة |
| Regulator مفقود | `regulator_get failed: -ENODEV` | `vdd-supply` property خاطئة |
| DMA channel خاطئ | `dma_request_chan failed` | `dmas` property أو `dma-names` خاطئة |
| GPIO conflict | `gpio-xxxxx: conflicting chip` | تعريف GPIO في أكثر من node |
| pinmux conflict | `pinctrl: pin xxx is already requested` | تداخل في `pinctrl-0` بين nodes |
| Power domain | `power domain init failed` | `power-domains` phandle خاطئ |

```bash
# فحص كل أخطاء الـ probe المرتبطة بـ DT
dmesg | grep -E "probe|deferred|ENODEV|EPROBE_DEFER|phandle|OF"

# رؤية كل الأجهزة التي تأجّل probing لها
cat /sys/kernel/debug/devices_deferred 2>/dev/null || \
    grep -r "deferred" /sys/kernel/debug/device_component/ 2>/dev/null
```

---

#### 5. تشخيص الـ Device Tree: التحقق من تطابق DTS مع الـ Hardware

```bash
# الخطوة 1: استخراج DTS من DTB المحمّل حالياً
dtc -I dtb -O dts -o /tmp/live.dts /sys/firmware/fdt

# الخطوة 2: مقارنة مع DTS المصدري
diff /tmp/live.dts /path/to/board.dts | head -50

# الخطوة 3: التحقق من وجود الـ node المطلوب
grep -r "compatible" /sys/firmware/devicetree/base/ 2>/dev/null | \
    strings | grep "my-driver"

# الخطوة 4: تحقق من القيم العددية بـ big-endian
# مثال: reg = <0x0 0xfe300000 0x0 0x1000>
cat /sys/firmware/devicetree/base/soc/i2c@fe300000/reg | \
    od -An -tx4 | awk '{print "0x"$1, "0x"$2, "0x"$3, "0x"$4}'

# الخطوة 5: التحقق من phandle resolution
# هل يشير phandle الـ clock إلى node موجود؟
PHANDLE=$(cat /sys/firmware/devicetree/base/soc/i2c@fe300000/clocks | \
    od -An -tu4 | awk '{print $1}')
grep -r "phandle" /sys/firmware/devicetree/base/ 2>/dev/null | \
    grep "$PHANDLE"

# الخطوة 6: تحقق من status property
cat /sys/firmware/devicetree/base/soc/i2c@fe300000/status 2>/dev/null || \
    echo "no status property (defaults to okay)"

# الخطوة 7: DT overlays المطبّقة
ls /sys/firmware/devicetree/overlays/ 2>/dev/null
```

---

### Practical Commands

#### مجموعة الأوامر الجاهزة للنسخ

##### فحص شامل لنود محدد في DT

```bash
#!/bin/bash
# فحص شامل لـ node في DT
NODE_PATH="/soc/i2c@fe300000"
BASE="/sys/firmware/devicetree/base"

echo "=== Node: $NODE_PATH ==="
echo "--- Properties ---"
for f in "$BASE/$NODE_PATH"/*; do
    name=$(basename "$f")
    echo -n "$name = "
    # محاولة قراءته كنص أولاً ثم hex
    val=$(strings "$f" 2>/dev/null)
    if [ -n "$val" ]; then
        echo "$val"
    else
        od -An -tx1 "$f" 2>/dev/null | tr -d '\n'
        echo
    fi
done

echo "--- Children ---"
ls "$BASE/$NODE_PATH/" 2>/dev/null

echo "--- Kernel device link ---"
ls -la /sys/bus/platform/devices/ | grep "fe300000"
```

##### تتبع of_parse_phandle في وقت الـ probe

```bash
# تفعيل tracing لكل دوال OF المتعلقة بالـ phandle
echo function > /sys/kernel/debug/tracing/current_tracer
cat << 'EOF' > /sys/kernel/debug/tracing/set_ftrace_filter
of_parse_phandle
__of_parse_phandle_with_args
of_phandle_iterator_next
of_find_node_by_phandle
of_count_phandle_with_args
EOF
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ driver أو أعد تحميله
modprobe my_driver

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -v "^#"
```

##### فحص refcount لـ device_node

```bash
# مراقبة of_node_get/put عبر kprobe
echo 'p:of_get of_node_get node=%di' >> /sys/kernel/debug/tracing/kprobe_events
echo 'p:of_put of_node_put node=%di' >> /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/of_get/enable
echo 1 > /sys/kernel/debug/tracing/events/kprobes/of_put/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... تشغيل الكود المشتبه به ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -E "of_get|of_put"
```

##### فحص الـ DT overlays

```bash
# تطبيق overlay وفحص النتيجة
dtoverlay my-overlay.dtbo

# التحقق من تطبيقه
dtoverlay -l

# فحص الـ nodes الجديدة
diff <(ls /sys/firmware/devicetree/base/soc/) \
     <(ls /sys/firmware/devicetree/base/soc/)

# إزالة overlay
dtoverlay -r my-overlay
```

##### مثال على مخرجات التشخيص وتفسيرها

```bash
$ dmesg | grep -E "i2c@fe300000|of_parse"
[    2.341] OF: fdt: Reserved map entry index 0 is the FDT itself
[    2.891] fe300000.i2c: supply vdd not found, using dummy regulator
[    2.892] fe300000.i2c: can't get clk 'div': -ENOENT
```

**التفسير:**
- `supply vdd not found` → خاصية `vdd-supply` مفقودة في DTS أو الـ regulator node غير مُعرَّف
- `can't get clk 'div': -ENOENT` → اسم الـ clock في `clock-names` لا يتطابق مع ما يتوقعه الـ driver، أو الـ `clocks` phandle يشير لـ node خاطئ

```bash
$ cat /sys/firmware/devicetree/base/soc/i2c@fe300000/clock-names
div   # ← الاسم موجود لكن الـ phandle خاطئ؟

$ cat /sys/firmware/devicetree/base/soc/i2c@fe300000/clocks | od -An -tu4
# تحقق أن الـ phandle يشير لـ clock controller الصحيح
```

```bash
$ cat /sys/firmware/devicetree/base/soc/i2c@fe300000/status
disabled   # ← Node موجود لكن status=disabled هو السبب!
# الحل: تغيير status إلى "okay" أو تطبيق overlay يغيّره
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART لا يظهر على بورد RK3562 صناعي

#### العنوان
**الـ UART controller موجود في الـ DTS لكن الـ driver لا يُحمَّل**

#### السياق
فريق bring-up يعمل على gateway صناعي مبني على **RK3562**. الـ serial console يعمل عبر UART2، لكن UART4 المخصص لـ RS-485 لا يُرى في `/dev/ttyS*` رغم أنه مُضاف في الـ DTS.

#### المشكلة
الـ kernel log يظهر:
```
serial 7e0a0000.serial: no matching driver found
```
والـ driver `8250_dw` لم يُحمَّل لهذه الـ instance.

#### التحليل
المهندس يفتح الـ DTS ويجد:

```dts
uart4: serial@7e0a0000 {
    compatible = "rockchip,rk3562-uart", "snps,dw-apb-uart";
    reg = <0x0 0x7e0a0000 0x0 0x100>;
    status = "okay";
};
```

ثم يتتبع كيف يقرأ الـ kernel هذا النود. الـ kernel يستخدم `of_device_is_compatible()`:

```c
/* Check if device matches any string in compatible property */
extern int of_device_is_compatible(const struct device_node *device,
                                   const char *);
```

المشكلة: الـ driver يبحث عن `"rockchip,rk3399-uart"` تحديداً عبر `of_match_node()`:

```c
/* Match device node against table of of_device_id entries */
extern const struct of_device_id *of_match_node(
    const struct of_device_id *matches,
    const struct device_node *node);
```

يفحص المهندس جدول الـ match في driver الـ `8250_dw`:
```c
static const struct of_device_id dw8250_of_match[] = {
    { .compatible = "rockchip,rk3399-uart" },
    { .compatible = "snps,dw-apb-uart" },
    { /* sentinel */ }
};
```

الـ RK3562 يستخدم string أولى `"rockchip,rk3562-uart"` غير موجودة في جدول الـ driver، لكن `"snps,dw-apb-uart"` موجودة. الفحص يمر عبر `of_device_compatible_match()` التي تمشي على كل strings الـ `compatible` property بالترتيب. المشكلة إذن ليست في الـ API، بل في أن الـ driver نفسه في kernel القديم (v5.15) لا يدعم `rk3562-uart` بعد.

#### الحل
**الخيار 1**: استخدام فقط الـ fallback compatible:

```dts
uart4: serial@7e0a0000 {
    /* Use generic dw-apb fallback only */
    compatible = "snps,dw-apb-uart";
    reg = <0x0 0x7e0a0000 0x0 0x100>;
    clocks = <&cru SCLK_UART4>, <&cru PCLK_UART4>;
    clock-names = "baudclk", "apb_pclk";
    status = "okay";
};
```

**الخيار 2**: تحديث الـ kernel أو تطبيق patch يضيف `rk3562-uart` لجدول الـ driver.

للتحقق:
```bash
# Inspect compatible property from sysfs (CONFIG_OF_KOBJ must be enabled)
cat /sys/firmware/devicetree/base/serial@7e0a0000/compatible | xxd

# Check what driver matched
ls -la /sys/bus/platform/devices/7e0a0000.serial/driver
```

#### الدرس المستفاد
الـ `compatible` property تُقرأ بالترتيب، والـ `of_match_node()` يتوقف عند أول match. احرص دائماً على وضع الـ string الأكثر تخصصاً أولاً (vendor-specific) ثم الـ fallback العام، وتحقق أن الـ driver يدعم الـ SoC الجديد قبل افتراض أن الـ DTS خاطئ.

---

### السيناريو الثاني: I2C sensor لا يُقرأ على STM32MP1 في IoT Gateway

#### العنوان
**الـ `of_parse_phandle_with_args()` يُرجع خطأً عند ربط sensor بـ I2C bus**

#### السياق
منتج IoT sensor node مبني على **STM32MP1**. يستخدم sensor مقياس ضغط **BMP280** متصل بـ I2C5. الـ driver يعمل بشكل مثالي على تصميم آخر، لكن هنا يفشل عند probe.

#### المشكلة
رسالة الـ kernel:
```
bmp280: probe of 2-0077 failed with error -22
```
الـ `-22` هو `EINVAL`، وهو ما يُرجعه `__of_parse_phandle_with_args()` عند وجود مشكلة في الـ phandle.

#### التحليل
الـ driver يحاول قراءة خاصية `vdda-supply` المربوطة بـ regulator:

```c
/*
 * Parse phandle + args from property "vdda-supply"
 * cells_name = "#supply-cells" (expected 0 cells)
 */
ret = of_parse_phandle_with_args(np, "vdda-supply",
                                 "#supply-cells", 0, &args);
```

الـ DTS الخاطئ:

```dts
/* BMP280 node */
bmp280@77 {
    compatible = "bosch,bmp280";
    reg = <0x77>;
    vdda-supply = <&reg_3v3>;  /* phandle reference */
};

/* Regulator node - MISSING #supply-cells */
reg_3v3: regulator-3v3 {
    compatible = "regulator-fixed";
    regulator-name = "3v3";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    /* #supply-cells = <0>; <-- مفقودة */
};
```

`__of_parse_phandle_with_args()` يستدعي `of_find_property()` داخلياً للبحث عن `#supply-cells` في نود الـ regulator:

```c
/* Search for property by name in node's properties linked list */
extern struct property *of_find_property(const struct device_node *np,
                                         const char *name,
                                         int *lenp);
```

الدالة تمشي على `device_node->properties` (linked list من `struct property`) وتبحث باستخدام `of_prop_cmp` وهو `strcmp`. لا تجد `#supply-cells`، فتُرجع `NULL`، فيرتد الـ parser بـ `EINVAL`.

#### الحل
إصلاح الـ DTS:

```dts
reg_3v3: regulator-3v3 {
    compatible = "regulator-fixed";
    regulator-name = "3v3";
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
    /* Supply phandle needs zero args by convention */
    #supply-cells = <0>;
};
```

للتشخيص السريع:
```bash
# Check if the property exists in live DT
cat /sys/firmware/devicetree/base/regulator-3v3/\#supply-cells 2>/dev/null || \
    echo "property missing"

# Dump all properties of the regulator node
ls /sys/firmware/devicetree/base/regulator-3v3/
```

#### الدرس المستفاد
كل phandle reference تتطلب property `#*-cells` في النود المُشار إليه. الـ `of_find_property()` يبحث بـ `strcmp` دقيقة — لا case-insensitive، لا wildcards. الخطأ `EINVAL` من `__of_parse_phandle_with_args()` دائماً يعني: إما الـ phandle غير موجود، أو `#*-cells` مفقودة أو قيمتها خاطئة.

---

### السيناريو الثالث: HDMI لا يعمل على Android TV Box مع i.MX8MQ

#### العنوان
**الـ overlay لتفعيل HDMI يُطبَّق لكن الشاشة تبقى سوداء**

#### السياق
منتج **Android TV box** مبني على **i.MX8MQ**. الفريق يستخدم device tree overlays لدعم تكوينات مختلفة (HDMI فقط، LVDS فقط، أو الاثنان). الـ overlay المخصص لتفعيل HDMI يُحمَّل بنجاح لكن الشاشة سوداء.

#### المشكلة
```
imx-drm display-subsystem: [drm] Cannot find any crtc or sizes
```

الـ HDMI node موجود لكن يبدو أن الـ driver لا يراه كـ available.

#### التحليل
الفريق يفحص الـ overlay:

```dts
/* overlay fragment targeting hdmi node */
&hdmi {
    status = "okay";
    ddc-i2c-bus = <&i2c2>;
};
```

يفحصون الـ base DTS:
```dts
hdmi: hdmi@32fd8000 {
    compatible = "fsl,imx8mq-hdmi";
    /* ... */
    status = "disabled";
};
```

المشكلة: الـ overlay يُطبَّق عبر `of_overlay_fdt_apply()`:

```c
int of_overlay_fdt_apply(const void *overlay_fdt, u32 overlay_fdt_size,
                         int *ovcs_id, const struct device_node *target_base);
```

بعد التطبيق، يجب أن يصبح `status = "okay"`. لكن الـ driver يفحص الـ availability عبر:

```c
/* Returns true only if status is "okay" or "ok" or absent */
extern bool of_device_is_available(const struct device_node *device);
```

هذه الدالة تستخدم `of_get_property()`:

```c
extern const void *of_get_property(const struct device_node *node,
                                   const char *name,
                                   int *lenp);
```

المشكلة الحقيقية: الـ overlay يستهدف `&hdmi` لكن الـ label `hdmi` في الـ base DTS مُعرَّف تحت `&dcss` block كـ sub-node، والـ overlay يستهدف النود الخطأ. الـ `of_node_check_flag()` يظهر أن النود يحمل `OF_POPULATED` (flag 3) — أي أن الـ driver قد probe الـ node القديم قبل تطبيق الـ overlay.

```c
/* Check if OF_POPULATED flag is set — device already created */
static inline int of_node_check_flag(const struct device_node *n,
                                     unsigned long flag)
{
    return test_bit(flag, &n->_flags);
}
```

#### الحل
تطبيق الـ overlay قبل تهيئة الـ platform devices، أو استخدام `of_changeset` للتعديل الصحيح:

```c
struct of_changeset ocs;
of_changeset_init(&ocs);

/* Update status property to "okay" */
of_changeset_update_prop_string(&ocs, hdmi_node, "status", "okay");

ret = of_changeset_apply(&ocs);
if (ret)
    of_changeset_destroy(&ocs);
```

للتشخيص:
```bash
# Check current status after overlay
cat /sys/firmware/devicetree/base/hdmi@32fd8000/status

# Check flags — populated flag means driver already ran
cat /sys/kernel/debug/devicetree/of_node_flags 2>/dev/null

# Use dtc to decompile live DT and inspect
dtc -I fs /sys/firmware/devicetree/base/ 2>/dev/null | grep -A5 "hdmi@"
```

#### الدرس المستفاد
الـ overlays تعمل بشكل صحيح فقط إذا طُبِّقت قبل أن يُنشئ الـ kernel الـ platform devices من الـ nodes المعنية. بعد `OF_POPULATED`، تغيير الـ `status` لا يُعيد الـ probe تلقائياً — تحتاج لـ `bus_rescan` أو إعادة bind. استخدم `of_changeset` مع notifier لضمان التسلسل الصحيح.

---

### السيناريو الرابع: SPI Flash لا يُكتشف على Allwinner H616 في Custom Board

#### العنوان
**الـ `of_get_next_available_child()` يتخطى النود رغم أن `status` صحيح**

#### السياق
custom board للـ **Allwinner H616** يستخدم SPI NOR flash خارجي (W25Q128) متصل بـ SPI0. المهندس يكتب driver مخصص يمشي على الـ child nodes لـ SPI controller ليجد الـ flash.

#### المشكلة
الـ driver لا يجد الـ flash node رغم وجوده في الـ DTS. لا رسائل خطأ، فقط صمت.

#### التحليل
الـ driver يستخدم:

```c
/* Walk available children of SPI controller */
struct device_node *child;
for_each_available_child_of_node(spi_node, child) {
    if (of_device_is_compatible(child, "jedec,spi-nor"))
        /* found it */;
}
```

الماكرو `for_each_available_child_of_node` يستدعي `of_get_next_available_child()` في كل iteration:

```c
extern struct device_node *of_get_next_available_child(
    const struct device_node *node, struct device_node *prev);
```

هذه الدالة داخلياً تستدعي `of_device_is_available()` على كل child، والتي تفحص خاصية `status`:

```c
extern bool of_device_is_available(const struct device_node *device);
```

المهندس يفحص الـ DTS:

```dts
spi0: spi@5010000 {
    compatible = "allwinner,sun50i-h616-spi";
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
        status = "OK";  /* <-- خطأ إملائي */
    };
};
```

`of_device_is_available()` تقبل فقط `"okay"` أو `"ok"` (بأحرف صغيرة). الـ `"OK"` بأحرف كبيرة غير مقبول! الـ comparison تعتمد على `strcmp` (case-sensitive) داخل `of_get_property()`.

للتحقق يستخدم:

```c
/* Check what of_get_property returns for "status" */
const char *status = of_get_property(child, "status", NULL);
pr_info("flash status: '%s'\n", status ?: "not set");
```

#### الحل
```dts
flash@0 {
    compatible = "jedec,spi-nor";
    reg = <0>;
    spi-max-frequency = <50000000>;
    status = "okay";  /* must be lowercase */
};
```

للتشخيص من الـ shell:
```bash
# Check status value with hexdump to see exact bytes
xxd /sys/firmware/devicetree/base/spi@5010000/flash@0/status

# Use for_each_child (not available) to list all children regardless of status
ls /sys/firmware/devicetree/base/spi@5010000/

# For drivers using for_each_child_of_node vs for_each_available_child_of_node
# the difference is this availability check
```

#### الدرس المستفاد
الـ `for_each_available_child_of_node` يستخدم `of_device_is_available()` التي تفحص `status` بـ `strcmp` حساسة للحالة. القيمة `"OK"` لا تعمل — فقط `"okay"` أو `"ok"`. استخدم `for_each_child_of_node` إذا أردت المشي على كل الـ nodes بغض النظر عن الـ `status`.

---

### السيناريو الخامس: CAN Bus لا يُهيَّأ على AM62x في Automotive ECU

#### العنوان
**الـ `of_property_read_u32_array()` يُرجع `-EOVERFLOW` على خاصية `bitrate-switch-ranges`**

#### السياق
وحدة **Automotive ECU** مبنية على **TI AM62x** تستخدم CAN-FD controller. الـ driver يقرأ خاصية مخصصة `bitrate-switch-ranges` من الـ DTS لتحديد نطاقات السرعة المدعومة. الـ driver يعمل على منصة أخرى لكنه يفشل هنا.

#### المشكلة
```
mcan 4000000.can: can't read bitrate-switch-ranges property: -75
```
الـ `-75` هو `EOVERFLOW`.

#### التحليل
الـ driver يفعل:

```c
/* Read array of u32 values for bitrate ranges */
u32 ranges[4];  /* driver expects exactly 4 values */
ret = of_property_read_u32_array(np, "bitrate-switch-ranges",
                                 ranges, 4);
if (ret)
    dev_err(dev, "can't read bitrate-switch-ranges property: %d\n", ret);
```

الدالة `of_property_read_u32_array()` هي wrapper حول:

```c
static inline int of_property_read_u32_array(const struct device_node *np,
                                             const char *propname,
                                             u32 *out_values, size_t sz)
{
    int ret = of_property_read_variable_u32_array(np, propname, out_values,
                                                  sz, 0);
    if (ret >= 0)
        return 0;
    else
        return ret;
}
```

الدالة تستدعي `of_property_read_variable_u32_array()` بـ `sz_min = sz` و `sz_max = 0`. إذا كانت الـ property تحتوي على عدد عناصر **أقل** من `sz`، تُرجع `EOVERFLOW`.

المهندس يفحص الـ DTS:

```dts
can0: can@4000000 {
    compatible = "ti,am6252-mcan";
    /* ... */
    /* This board only supports 2 ranges, not 4 */
    bitrate-switch-ranges = <500000 2000000>;  /* 2 values فقط */
};
```

الـ driver يتوقع 4 قيم، لكن الـ DTS يحتوي على 2 فقط. لذلك `of_property_read_variable_u32_array()` تجد أن `property->length / sizeof(u32) = 2 < sz_min = 4`، فتُرجع `-EOVERFLOW`.

المهندس يتحقق باستخدام:

```c
/* Count actual number of u32 elements first */
int count = of_property_count_u32_elems(np, "bitrate-switch-ranges");
dev_info(dev, "bitrate-switch-ranges has %d elements\n", count);
```

حيث `of_property_count_u32_elems()` تستدعي:

```c
static inline int of_property_count_u32_elems(const struct device_node *np,
                                              const char *propname)
{
    return of_property_count_elems_of_size(np, propname, sizeof(u32));
}
```

#### الحل
**الخيار 1**: تصحيح الـ DTS ليحتوي على 4 قيم:

```dts
/* Provide all 4 required bitrate range values */
bitrate-switch-ranges = <250000 500000 1000000 2000000>;
```

**الخيار 2**: تعديل الـ driver ليستخدم `of_property_read_variable_u32_array()` مع `sz_min`:

```c
u32 ranges[4];
/* Accept 2 to 4 values — flexible reading */
ret = of_property_read_variable_u32_array(np, "bitrate-switch-ranges",
                                          ranges, 2, 4);
if (ret < 0) {
    dev_err(dev, "failed: %d\n", ret);
    return ret;
}
/* ret now holds actual number of elements read */
num_ranges = ret;
```

للتشخيص:
```bash
# Count u32 elements in property
python3 -c "
import struct
data = open('/sys/firmware/devicetree/base/can@4000000/bitrate-switch-ranges','rb').read()
print(f'elements: {len(data)//4}, values: {struct.unpack(\">\" + \"I\"*(len(data)//4), data)}')
"

# Or using fdtget
fdtget /boot/my-am62x.dtb /can@4000000 bitrate-switch-ranges
```

#### الدرس المستفاد
الفرق بين `of_property_read_u32_array()` (تطلب عدداً ثابتاً) و `of_property_read_variable_u32_array()` (تقبل نطاقاً) حاسم. استخدم دائماً `of_property_count_u32_elems()` أولاً إذا كان عدد العناصر قد يتغير بين الـ boards، ثم استخدم الـ variable variant مع حدود مناسبة لتجنب `EOVERFLOW` أو `ENODATA`.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ** LWN.net هو المصدر الأكاديمي الأول لمتابعة تطور kernel Linux، وما يلي أبرز المقالات المتعلقة بـ Device Tree:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| ELCE: Grant Likely on device trees | [lwn.net/Articles/414016](https://lwn.net/Articles/414016/) | مقدمة شاملة من مصمم النظام نفسه |
| Platform devices and device trees | [lwn.net/Articles/448502](https://lwn.net/Articles/448502/) | ربط `platform_device` بـ DT nodes |
| Device tree troubles | [lwn.net/Articles/560523](https://lwn.net/Articles/560523/) | تحليل مشكلات التبني وصراعات ARM |
| Device trees I: Are we having fun yet? | [lwn.net/Articles/572692](https://lwn.net/Articles/572692/) | تشريح بنية DTS وحالات الاستخدام |
| Device trees II: The harder parts | [lwn.net/Articles/574852](https://lwn.net/Articles/574852/) | صعوبات abstraction والـ binding |
| Device tree overlays | [lwn.net/Articles/616859](https://lwn.net/Articles/616859/) | التعديل الديناميكي للـ DT وقت التشغيل |
| An alternative device-tree source language | [lwn.net/Articles/730217](https://lwn.net/Articles/730217/) | مقترح YAML-DT كبديل لـ DTS |
| Device-tree support for ARM | [lwn.net/Articles/367752](https://lwn.net/Articles/367752/) | بداية إضافة DT لمعمارية ARM |
| General device tree irq domain infrastructure | [lwn.net/Articles/440524](https://lwn.net/Articles/440524/) | بنية IRQ domain داخل DT |
| Device tree on x86, part v4 | [lwn.net/Articles/429318](https://lwn.net/Articles/429318/) | تجربة نقل DT لمعمارية x86 |

---

### التوثيق الرسمي للـ kernel

**الـ** `Documentation/devicetree/` هي المرجع الرسمي الموجود داخل شجرة الـ kernel:

```
Documentation/devicetree/
├── usage-model.rst          # نموذج الاستخدام الكامل — نقطة البداية المثالية
├── bindings/                # مئات الـ binding specs لكل نوع جهاز
│   ├── gpio/
│   ├── i2c/
│   ├── spi/
│   └── ...
├── of_unittest.rst          # كيفية تشغيل اختبارات of_selftest
└── kernel-api.rst           # توثيق API على مستوى kernel
```

**الـ** online version على docs.kernel.org:

- [Open Firmware and Devicetree — docs.kernel.org](https://docs.kernel.org/devicetree/index.html)
- [Linux and the Devicetree (usage model)](https://docs.kernel.org/devicetree/usage-model.html)
- [OF Unittest documentation](https://www.kernel.org/doc/html/latest/devicetree/of_unittest.html)

**الـ** header الأساسي المُوثَّق:
- `include/linux/of.h` — واجهة برمجية كاملة مع تعليقات kernel-doc
- `include/linux/of_gpio.h` — GPIO من DT
- `include/linux/of_irq.h` — IRQ من DT
- `include/linux/of_address.h` — ترجمة العناوين

---

### Commits مهمة في تاريخ of.h

**الـ** commits التالية شكّلت المسار التاريخي للملف:

```bash
# البداية: نقل OF من PowerPC إلى kernel مشترك (2005-2007)
git log --oneline --follow include/linux/of.h | tail -20

# أبرز الـ commits التاريخية:
# - إضافة of_find_node_by_path()
# - إضافة of_get_property()
# - إضافة of_match_device() لربط drivers
# - إضافة fwnode_handle لتوحيد ACPI وDT
```

**الـ** commits يمكن البحث عنها في:
- [kernel.org git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/of.h)

---

### نقاشات Mailing List

**الـ** mailing lists الأساسية للنقاشات التاريخية:

| القائمة | الموضوع |
|---------|---------|
| `linux-kernel@vger.kernel.org` | نقاشات عامة حول DT API |
| `devicetree@vger.kernel.org` | القائمة المخصصة للـ Device Tree |
| `linux-arm-kernel@lists.infradead.org` | نقاشات ARM وDT integration |

**أبرز نقاش تاريخي:** رفض Russell King الأولي لإضافة FDT لـ ARM، الموثق في أرشيفات `linux-arm-kernel` عام 2009 — يُظهر كيف تحولت الحاجة العملية إلى قبول المجتمع.

للبحث في الأرشيفات:
- [lore.kernel.org/devicetree](https://lore.kernel.org/devicetree/)
- [lore.kernel.org/linux-arm-kernel](https://lore.kernel.org/linux-arm-kernel/)

---

### صفحات eLinux.org

**الـ** eLinux.org يُركّز على embedded Linux ويحتوي أغنى مجموعة عملية حول DT:

| الصفحة | الرابط | المحتوى |
|--------|--------|---------|
| Device Tree Reference | [elinux.org/Device_Tree_Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل للصيغة والـ bindings |
| Device Tree Mysteries | [elinux.org/Device_Tree_Mysteries](https://elinux.org/Device_Tree_Mysteries) | حل المشكلات الشائعة |
| Device tree history | [elinux.org/Device_tree_history](https://elinux.org/Device_tree_history) | التاريخ من OpenFirmware حتى ARM |
| Device Tree unittest | [elinux.org/Device_Tree_unittest](https://elinux.org/Device_Tree_unittest) | اختبار الـ OF API |
| EBC Device Trees | [elinux.org/EBC_Device_Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على BeagleBone |
| Capemgr | [elinux.org/Capemgr](https://elinux.org/Capemgr) | إدارة overlays ديناميكياً |

---

### صفحات kernelnewbies.org

**الـ** kernelnewbies.org يوثّق التغييرات بين إصدارات الـ kernel:

| الصفحة | الرابط | ما تجده |
|--------|--------|---------|
| Linux 3.0 Driver Architecture | [kernelnewbies.org/Linux_3.0_DriverArch](https://kernelnewbies.org/Linux_3.0_DriverArch) | أول DT support لعدة SoCs |
| Linux 3.11 Drivers | [kernelnewbies.org/Linux_3.11-DriversArch](https://kernelnewbies.org/Linux_3.11-DriversArch) | توسعة DT bindings |
| Linux 3.15 Drivers | [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) | تغييرات API |
| Linux 4.0 Drivers | [kernelnewbies.org/Linux_4.0-DriversArch](https://kernelnewbies.org/Linux_4.0-DriversArch) | تحسينات overlay |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 14 — The Linux Device Model
- **ملاحظة:** الكتاب قديم (kernel 2.6) لكنه يشرح أسس `kobject` و`sysfs` التي يعتمد عليها `of_node`
- [متاح مجاناً على lwn.net](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم:** Chapter 17 — Devices and Modules
- يشرح `platform_device`، `bus_type`، وكيف يندمج DT مع نموذج الأجهزة
- ISBN: 978-0672329463

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المهم:** Chapter 16 — Device Trees
- أفضل كتاب مخصص لشرح DTS syntax وكيفية كتابة bindings من الصفر
- ISBN: 978-0137017836

#### Mastering Embedded Linux Programming — Chris Simmonds
- يغطي DT overlays والتطبيقات العملية على Raspberry Pi وBeagleBone

---

### المعيار الرسمي: Devicetree Specification

**الـ** [Devicetree Specification](https://www.devicetree.org/specifications/) هو المرجع المعياري الذي يُعرّف:
- بنية FDT (Flattened Device Tree) الثنائية
- قواعد `phandle`، `reg`، `interrupts`، `clocks`
- المتطلبات الأساسية لأي binding جديد

---

### أدوات مساعدة

```bash
# تحويل DTB → DTS قابل للقراءة
dtc -I dtb -O dts -o output.dts input.dtb

# فحص صحة DTS
dtc -I dts -O dtb -o /dev/null input.dts

# فحص binding مقابل schema
dt-validate -p Documentation/devicetree/bindings/ my-device.dts

# عرض DT الحالي للنظام الحي
ls /proc/device-tree/
cat /proc/device-tree/compatible

# عرض من sysfs
ls /sys/firmware/devicetree/base/
```

---

### مصطلحات البحث (Search Terms)

للعثور على مزيد من المعلومات، استخدم:

```
"device tree" "of_find_node_by_name" linux kernel
"struct device_node" linux kernel driver
"of_match_table" platform_driver
"fwnode_handle" ACPI device tree unification
"devicetree bindings" yaml schema linux
dtc compiler device tree compiler linux
"flattened device tree" FDT boot protocol
"OF selftest" linux kernel testing
device tree overlay linux runtime
"of_phandle_args" interrupt specifier linux
```

---

### خلاصة المصادر حسب الأولوية

```
المستوى المبتدئ:
  1. elinux.org/Device_Tree_Reference
  2. docs.kernel.org/devicetree/usage-model.html
  3. lwn.net/Articles/448502/ (Platform devices and device trees)

المستوى المتوسط:
  4. lwn.net/Articles/572692/ (Device trees I)
  5. lwn.net/Articles/574852/ (Device trees II)
  6. Embedded Linux Primer — فصل Device Trees

المستوى المتقدم:
  7. Devicetree Specification (devicetree.org)
  8. Documentation/devicetree/bindings/ (داخل kernel source)
  9. lore.kernel.org/devicetree/ (نقاشات patches الحالية)
  10. git log include/linux/of.h (تاريخ التغييرات)
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `of_reconfig_notifier` هو آلية notifier مُصدَّرة من `include/linux/of.h` تُتيح لأي module الاستماع لأحداث تعديل الـ Device Tree في وقت التشغيل — مثل إضافة node، حذف node، أو تحديث property. نستخدمها هنا لطباعة معلومات كل حدث يطرأ على شجرة الأجهزة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * of_reconfig_watcher.c
 *
 * Kernel module that hooks into the OF (Open Firmware / Device Tree)
 * reconfig notifier chain to observe live DT changes at runtime.
 *
 * Relevant API from include/linux/of.h:
 *   of_reconfig_notifier_register()
 *   of_reconfig_notifier_unregister()
 *   of_reconfig_get_state_change()
 *   struct of_reconfig_data  { .dn, .prop, .old_prop }
 */

#include <linux/module.h>       /* module_init/exit, MODULE_* macros         */
#include <linux/kernel.h>       /* pr_info, pr_err                            */
#include <linux/notifier.h>     /* notifier_block, NOTIFY_OK                 */
#include <linux/of.h>           /* of_reconfig_notifier_*, of_reconfig_data  */

/* -----------------------------------------------------------------------
 * action_name() — helper: map numeric action to readable string
 * ----------------------------------------------------------------------- */
static const char *action_name(unsigned long action)
{
    switch (action) {
    case OF_RECONFIG_ATTACH_NODE:     return "ATTACH_NODE";
    case OF_RECONFIG_DETACH_NODE:     return "DETACH_NODE";
    case OF_RECONFIG_ADD_PROPERTY:    return "ADD_PROPERTY";
    case OF_RECONFIG_REMOVE_PROPERTY: return "REMOVE_PROPERTY";
    case OF_RECONFIG_UPDATE_PROPERTY: return "UPDATE_PROPERTY";
    default:                          return "UNKNOWN";
    }
}

/* -----------------------------------------------------------------------
 * of_reconfig_callback() — الـ callback الرئيسي
 *
 * يُستدعى في كل مرة تتغير فيها شجرة الأجهزة عبر of_changeset_apply().
 *
 * @nb     : مؤشر لـ notifier_block المُسجَّل (يمكن الوصول منه للبنية الأم)
 * @action : رمز الحدث (OF_RECONFIG_*)
 * @data   : مؤشر لـ struct of_reconfig_data يحمل الـ node والـ property
 * ----------------------------------------------------------------------- */
static int of_reconfig_callback(struct notifier_block *nb,
                                unsigned long action,
                                void *data)
{
    /* data is always of_reconfig_data* when coming from of_reconfig_notify */
    struct of_reconfig_data *rd = (struct of_reconfig_data *)data;

    /* Guard against NULL — some callers may pass NULL during teardown */
    if (!rd || !rd->dn)
        return NOTIFY_OK;

    pr_info("of_watcher: action=%-20s node=%s\n",
            action_name(action),
            rd->dn->full_name ? rd->dn->full_name : "(unnamed)");

    /* For property-level actions, print the property name too */
    if (rd->prop) {
        pr_info("of_watcher:   new_prop=%s (len=%d)\n",
                rd->prop->name ? rd->prop->name : "?",
                rd->prop->length);
    }

    /* For UPDATE, show the old property name for cross-reference */
    if (action == OF_RECONFIG_UPDATE_PROPERTY && rd->old_prop) {
        pr_info("of_watcher:   old_prop=%s (len=%d)\n",
                rd->old_prop->name ? rd->old_prop->name : "?",
                rd->old_prop->length);
    }

    /*
     * NOTIFY_OK tells the notifier chain to continue calling
     * the next registered listener (if any).
     */
    return NOTIFY_OK;
}

/* -----------------------------------------------------------------------
 * Static notifier_block — ربط الـ callback بالـ chain
 *
 * .notifier_call : الدالة التي ستُستدعى عند كل حدث
 * .priority      : الأولوية (0 = افتراضي، أعلى = يُستدعى أولاً)
 * ----------------------------------------------------------------------- */
static struct notifier_block of_reconfig_nb = {
    .notifier_call = of_reconfig_callback,
    .priority      = 0,
};

/* -----------------------------------------------------------------------
 * of_watcher_init() — تسجيل الـ notifier عند تحميل الـ module
 * ----------------------------------------------------------------------- */
static int __init of_watcher_init(void)
{
    int ret;

    /*
     * of_reconfig_notifier_register() adds our notifier_block to the
     * of_reconfig_chain (a blocking_notifier_head defined in drivers/of/base.c).
     * Requires CONFIG_OF_DYNAMIC=y on the target kernel.
     */
    ret = of_reconfig_notifier_register(&of_reconfig_nb);
    if (ret) {
        pr_err("of_watcher: failed to register notifier (%d)\n", ret);
        return ret;
    }

    pr_info("of_watcher: registered — watching Device Tree reconfigurations\n");
    return 0;
}

/* -----------------------------------------------------------------------
 * of_watcher_exit() — إلغاء التسجيل عند إزالة الـ module
 *
 * إلغاء التسجيل ضروري: بدونه سيحاول الكيرنل استدعاء callback
 * في ذاكرة تم تحريرها → kernel oops أو panic.
 * ----------------------------------------------------------------------- */
static void __exit of_watcher_exit(void)
{
    of_reconfig_notifier_unregister(&of_reconfig_nb);
    pr_info("of_watcher: unregistered — goodbye\n");
}

module_init(of_watcher_init);
module_exit(of_watcher_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("OF Watcher Example");
MODULE_DESCRIPTION("Monitor Device Tree runtime reconfigurations via of_reconfig notifier");
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | ماكروات `module_init` / `module_exit` / `MODULE_*` |
| `<linux/kernel.h>` | `pr_info` / `pr_err` |
| `<linux/notifier.h>` | تعريف `struct notifier_block` و `NOTIFY_OK` |
| `<linux/of.h>` | `of_reconfig_notifier_register/unregister`، `struct of_reconfig_data`، ثوابت `OF_RECONFIG_*` |

**الـ** `<linux/notifier.h>` ضروري بشكل صريح لأن `of.h` يُعلّن `struct notifier_block` فقط كـ forward declaration ولا يُضمّن الـ header كاملاً.

---

#### الـ callback — `of_reconfig_callback`

```
nb     → مؤشر للـ notifier_block (نادراً ما نستخدمه مباشرةً هنا)
action → رمز عددي: OF_RECONFIG_ATTACH_NODE / ADD_PROPERTY / ...
data   → void* يُشير فعلياً لـ struct of_reconfig_data*
```

**الـ** `struct of_reconfig_data` يحمل ثلاثة حقول أساسية:

```
.dn       → الـ device_node المتأثر (الاسم الكامل في .full_name)
.prop     → الـ property الجديدة (للإضافة أو التحديث)
.old_prop → الـ property القديمة (للتحديث فقط)
```

يطبع الـ callback اسم العملية واسم الـ node، ثم اسم الـ property إن وُجدت — معلومات كافية لمتابعة overlay أو hotplug DT.

---

#### الـ notifier_block

```c
static struct notifier_block of_reconfig_nb = {
    .notifier_call = of_reconfig_callback,
    .priority      = 0,
};
```

**الـ** `.priority = 0` يجعل الـ module يُستدعى بترتيب التسجيل. رفع القيمة يجعله يُستدعى أولاً قبل باقي المستمعين على نفس الـ chain.

---

#### `module_init` و `module_exit`

- **`of_reconfig_notifier_register`**: تُضيف الـ `notifier_block` إلى الـ `of_reconfig_chain` الداخلية (blocking notifier). تعتمد على `CONFIG_OF_DYNAMIC=y`.
- **`of_reconfig_notifier_unregister`**: تُزيله من الـ chain. إهمال هذه الخطوة يُسبب use-after-free عند أول حدث DT بعد `rmmod`.

---

### كيفية الاختبار

```bash
# بناء الـ module (Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod of_reconfig_watcher.ko

# تطبيق DT overlay لتوليد حدث (مثال على Raspberry Pi / BeagleBone)
sudo dtoverlay some_overlay.dtbo

# مشاهدة مخرجات الـ notifier
dmesg | grep of_watcher

# إزالة الـ module
sudo rmmod of_reconfig_watcher
```

**مثال مخرجات متوقعة:**

```
[  123.456789] of_watcher: registered — watching Device Tree reconfigurations
[  130.112233] of_watcher: action=ATTACH_NODE          node=/soc/i2c@fe804000/sensor@48
[  130.112290] of_watcher: action=ADD_PROPERTY         node=/soc/i2c@fe804000/sensor@48
[  130.112290] of_watcher:   new_prop=compatible (len=12)
```

---

### ملاحظات الأمان والتوافق

- الـ module يعمل فقط على أنظمة بها `CONFIG_OF_DYNAMIC=y` (مثل ARM SoCs التي تدعم DT overlays).
- الـ callback يُنفَّذ في سياق sleeping (blocking notifier) — آمن لاستخدام `pr_info` وأي عمليات لا تتطلب atomic context.
- يجب دائماً التحقق من `rd && rd->dn` قبل dereference لتفادي NULL pointer في حالات edge cases أثناء التهيئة.
