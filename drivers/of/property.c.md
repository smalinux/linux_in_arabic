## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: Open Firmware and Flattened Device Tree (OF/FDT)

الملف `drivers/of/property.c` جزء من subsystem اسمه **Open Firmware (OF)** أو **Device Tree**، وده الـ subsystem المسؤول عن إن الـ kernel يعرف الأجهزة اللي على الـ board من غير ما يحتاج يحسسها بنفسه.

---

### القصة من الأول — ليه Device Tree موجود أصلاً؟

تخيل إنك بتبني لعبة LEGO. كل قطعة وراها رقم، وعشان تعرف تركبهم صح محتاج كتيب بيقولك "القطعة دي بتتوصل في المكان ده، وبتاكل كهرباء كام".

الـ Linux kernel زي اللي بيركب اللعبة دي. بس المشكلة — على x86 (كمبيوتر عادي) فيه BIOS/UEFI يعمل **autodetection**. أما على ARM وPowerPC وRISC-V (اللي بيتحكموا في الموبايلات والـ embedded boards) — مفيش حاجة بتعمل autodetection. الـ hardware قاعد صامت، مش بيقول عن نفسه.

الحل اللي الناس اخترعته: **Device Tree** — ملف نصي (`.dts`) مكتوب فيه وصف كامل للـ hardware قبل ما تشغّل الـ kernel:

```
/ {
    uart0: serial@44e09000 {
        compatible = "ti,omap3-uart";
        reg = <0x44e09000 0x2000>;
        clocks = <&uart0_fck>;   /* بتاكل clock من هنا */
        interrupts = <72>;
    };

    camera@0 {
        port {
            endpoint {
                remote-endpoint = <&isp_ep>;  /* متوصل بالـ ISP */
            };
        };
    };
};
```

الـ bootloader بيحوّل الـ `.dts` لـ binary اسمه **DTB (Device Tree Blob)** وبيبعته للـ kernel.

---

### دور `property.c` تحديداً — الجزء الأهم

بعد ما الـ kernel يقرأ الـ DTB ويبني منه شجرة من الـ `device_node` structs في الذاكرة، جيه دور `property.c`:

> **المهمة: إزاي الـ driver يقرأ خصائص الـ node بتاعته، وإزاي الـ kernel يربط الأجهزة ببعض.**

الملف بيعمل **٣ حاجات كبيرة**:

---

#### 1. قراءة الـ Properties (البيانات الخام من الـ Device Tree)

كل `device_node` جواه list من الـ `property` structs — كل property ليها اسم وقيمة raw bytes. المشكلة: القيمة دي محتاجها تتحول لـ `u8`, `u16`, `u32`, `u64`, `string`, أو array.

```c
// driver عايز يقرأ عنوان الـ UART من الـ tree:
u32 base_addr;
of_property_read_u32(np, "reg", &base_addr);  // قرأ 0x44e09000

// driver عايز يقرأ اسم التوافق:
const char *compat;
of_property_read_string(np, "compatible", &compat);  // "ti,omap3-uart"
```

الملف بيوفر دوال لكل type: `of_property_read_u8/u16/u32/u64` وvariants للـ arrays.

---

#### 2. الـ OF Graph — ربط الأجهزة بأجهزة تانية (زي Camera Pipeline)

فيه حالة مخصوصة جداً: الأجهزة اللي بترسل data لبعض — زي **camera sensor → ISP → display**. الـ Device Tree بيوصف الروابط دي بـ **graph** — كل جهاز ليه `port` nodes، وكل port ليه `endpoint` nodes، والـ endpoints بتبص في بعض.

```
sensor {
    port@0 {
        sensor_out: endpoint {
            remote-endpoint = <&isp_in>;
        };
    };
};

isp {
    port@0 {
        isp_in: endpoint {
            remote-endpoint = <&sensor_out>;
        };
    };
    port@1 {
        isp_out: endpoint {
            remote-endpoint = <&display_in>;
        };
    };
};
```

الملف بيوفر دوال للتنقل في الـ graph ده:
- `of_graph_get_next_endpoint()` — روح من endpoint للتانية
- `of_graph_get_remote_endpoint()` — اوصل للطرف التاني من الوصلة
- `of_graph_get_remote_port_parent()` — اوصل للجهاز نفسه اللي على الطرف التاني

---

#### 3. الـ fwnode_ops — الجسر لـ Generic Firmware Layer

ده الجزء الأذكى في الملف. الـ kernel عنده abstraction اسمها **fwnode (firmware node)** بتخليه يتكلم مع الأجهزة بنفس الكود سواء الـ firmware كان **Device Tree** أو **ACPI** (اللي بيستخدمه x86).

الملف بيعرّف `of_fwnode_ops` — struct مليان function pointers — وبيوصلها بالـ Device Tree implementation:

```c
const struct fwnode_operations of_fwnode_ops = {
    .property_read_int_array = of_fwnode_property_read_int_array,
    .graph_get_next_endpoint = of_fwnode_graph_get_next_endpoint,
    .add_links               = of_fwnode_add_links,
    /* ... وغيرها */
};
```

يعني driver كتبه حد مرة للـ fwnode API، هيشتغل على Device Tree وعلى ACPI من غير ما يتغير.

---

#### 4. الـ fw_devlink — ربط الـ Consumer بالـ Supplier تلقائياً

ده الجزء الأحدث والأهم من الملف. الـ kernel عنده نظام اسمه **fw_devlink** — بيقرأ كل properties في كل node، ولو لقى property زي `clocks`, `power-domains`, `resets`, `gpio` — بيعرف إن الـ node ده "consumer" محتاج "supplier" تاني.

الملف بيعرّف table كامل من الـ parsers:

```c
DEFINE_SIMPLE_PROP(clocks,         "clocks",         "#clock-cells")
DEFINE_SIMPLE_PROP(power_domains,  "power-domains",  "#power-domain-cells")
DEFINE_SIMPLE_PROP(resets,         "resets",         "#reset-cells")
DEFINE_SIMPLE_PROP(phys,           "phys",           "#phy-cells")
DEFINE_SUFFIX_PROP(regulators,     "-supply",        NULL)
DEFINE_SUFFIX_PROP(gpio,           "-gpio",          "#gpio-cells")
/* ... أكتر من 25 binding */
```

وبيعمل `fwnode_link_add()` لكل supplier — وده بيخلي الـ kernel يضمن إن الـ supplier اتـ probe قبل الـ consumer.

---

### القصة الكاملة بالصور

```
Boot:
  bootloader
      │
      ▼
  DTB في الذاكرة
      │
      ▼
  of_fdt_unflatten_tree()  ← [drivers/of/fdt.c]
      │
      ▼
  شجرة device_nodes في الـ RAM
      │
      ▼
  property.c ← أنت هنا
  ┌──────────────────────────────────────┐
  │  1. قرا الـ properties (u32/string/…) │
  │  2. نقل في الـ graph (camera/display) │
  │  3. fwnode_ops (bridge لـ ACPI/OF)   │
  │  4. fw_devlink (consumer→supplier)   │
  └──────────────────────────────────────┘
      │
      ▼
  device drivers يتـ probe بالترتيب الصح
```

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `drivers/of/base.c` | الأساس — بناء الشجرة، البحث في الـ nodes |
| `drivers/of/fdt.c` | تحليل الـ DTB Binary وبناء الشجرة |
| `drivers/of/address.c` | قراءة `reg` properties وتحويلها لعناوين |
| `drivers/of/irq.c` | قراءة `interrupts` وتحويلها لـ IRQ numbers |
| `drivers/of/device.c` | ربط الـ device_node بـ struct device |
| `drivers/of/platform.c` | إنشاء الـ platform_device من الـ nodes |
| `drivers/of/overlay.c` | Dynamic overlays — إضافة/حذف nodes وقت التشغيل |
| `include/linux/of.h` | تعريفات `device_node`, `property`, `of_phandle_args` |
| `include/linux/of_graph.h` | تعريفات `of_endpoint` وماكروهات الـ graph iteration |
| `include/linux/property.h` | الـ generic fwnode API (المشترك بين OF وACPI) |
| `include/linux/fwnode.h` | تعريف `fwnode_handle` و`fwnode_operations` |
| `drivers/base/property.c` | تنفيذ الـ generic fwnode API من جانب الـ kernel core |

---

### ملفات الـ Subsystem (drivers/of/)

**Core:**
- `base.c` — البحث في الشجرة، traversal، الـ locking
- `property.c` — قراءة الـ properties، الـ graph، الـ fwnode_ops
- `fdt.c` — parse الـ DTB وبناء الشجرة
- `dynamic.c` — إضافة/حذف nodes dynamically
- `resolver.c` — حل phandle references في الـ overlays
- `overlay.c` — دعم الـ device tree overlays

**Hardware Interaction:**
- `address.c` — تحويل `reg` لعناوين فيزيائية
- `irq.c` — تحويل `interrupts` لـ Linux IRQ numbers
- `device.c` — ربط الـ OF node بالـ Linux device model
- `platform.c` — إنشاء platform_devices

**Misc:**
- `of_reserved_mem.c` — إدارة الـ reserved memory regions
- `of_numa.c` — NUMA topology من الـ device tree
- `kobj.c` — عرض الـ device tree في `/sys/firmware/devicetree/`
## Phase 2: شرح الـ OF Property & Graph Framework

---

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي نظام embedded، الـ kernel محتاج يعرف إيه الـ hardware الموجود على الـ board: فين الـ UART؟ بيشتغل على أنهي interrupt? محتاج أنهي clock? بيتكلم مع أنهي DMA controller?

على معالجات x86، الـ hardware بيعرّف نفسه ذاتيًا (self-describing) عبر ACPI أو PCI. أما على ARM embedded، مفيش حاجة من دي — الـ hardware صامت تمامًا.

**الحل التاريخي**: كل board كانت بيبقى ليها ملف C ضخم فيه بيانات ثابتة hardcoded في الـ kernel — أي تغيير في الـ hardware = إعادة compile للـ kernel كله. ده كان كارثة.

**الحل الصح**: **Device Tree** — ملف binary منفصل (`.dtb`) بيوصف الـ hardware بشكل declarative، بيتحمّل جمب الـ kernel من الـ bootloader، والـ kernel بيقرأه وقت الـ boot.

لكن قراءة الـ Device Tree raw مش كافية — محتاج **subsystem** متخصص يعرف يجيب property معينة، يقرأ arrays، يفك phandles، ويبني dependencies. ده هو دور `drivers/of/property.c`.

---

### الحل — الكيرنل بيعمل إيه؟

الكيرنل بيحوّل الـ DTB (binary) لـ tree من الـ `device_node` structs في الـ RAM أثناء الـ boot، وبعدين `drivers/of/property.c` بيوفر:

1. **Property Access API** — قراءة values من nodes بأنواعها (bool, u8, u16, u32, u64, string, arrays).
2. **OF Graph API** — وصف connections بين الـ hardware components (مثلاً: الـ camera sensor متوصل بأنهي CSI port على الـ SoC).
3. **fwnode_operations** — abstraction layer بيخلي بقية الكيرنل يتعامل مع DT و ACPI بنفس الـ API.
4. **fw_devlink** — بناء dependency graph بين الـ devices من خلال الـ phandles الموجودة في الـ DT.

---

### تشبيه من الواقع — مبنى إداري

تخيّل إنك موظف جديد دخلت مبنى إداري ضخم:

| عنصر حقيقي | مقابله في الكيرنل |
|---|---|
| **دليل المبنى** (كتالوج بأسماء الأقسام وأرقام الغرف) | الـ **Device Tree source** (`.dts` file) |
| **نسخة PDF مطبوعة** من الدليل | الـ **DTB** (compiled binary) |
| **مكتب الاستعلامات** اللي بيقرألك الدليل ويجيبلك المعلومة | الـ **OF Property API** (`of_property_read_*`) |
| **قسم الـ IT** اللي عارف مين بيوصّل بمين (network, phones) | الـ **OF Graph API** |
| **نظام الأذونات** اللي بيمنع قسم من التشغيل قبل ما قسم تاني يخلص setup | الـ **fw_devlink** dependency system |
| **نموذج موحد** بيتقدّم بيه لأي قسم بنفس الطريقة | الـ **fwnode_operations** (DT + ACPI abstraction) |

الدليل نفسه (DTB) مش هو اللي بيشتغل — محتاج مكتب استعلامات (OF API) يعرف كيف يفسّره ويجيب الإجابة الصح.

---

### الـ Big Picture — فين الـ subsystem ده في الكيرنل؟

```
+------------------------------------------------------------------+
|                        User Space                                |
+------------------------------------------------------------------+
                              |
+------------------------------------------------------------------+
|                    Kernel Driver Layer                           |
|  (i2c driver, gpio driver, clk driver, video driver, ...)        |
|         يستخدم: of_property_read_u32(), of_parse_phandle()      |
+------------------------------------------------------------------+
                              |
+-----------------------------+------------------------------------+
|   drivers/of/property.c     |    drivers/of/base.c              |
|   - Property Read API        |    - Node Lookup                  |
|   - OF Graph API             |    - Tree Traversal               |
|   - fwnode_operations        |    - phandle resolution           |
|   - fw_devlink / add_links   |                                   |
+-----------------------------+------------------------------------+
                              |
+------------------------------------------------------------------+
|              OF Core (drivers/of/)                               |
|   of_root → tree of device_node structs in RAM                  |
+------------------------------------------------------------------+
                              |
+------------------------------------------------------------------+
|        Boot-time DTB parsing (drivers/of/fdt.c)                 |
|        يحوّل الـ .dtb binary → device_node tree في الـ RAM      |
+------------------------------------------------------------------+
                              |
+------------------------------------------------------------------+
|   Bootloader (U-Boot) يمرر الـ DTB address في الـ r2/x1 register|
+------------------------------------------------------------------+
```

---

### الـ Core Abstractions — الـ Structs الأساسية

#### 1. `struct property`

الوحدة الأصغر في الـ DT — تمثّل property واحدة:

```c
struct property {
    char    *name;        /* اسم الـ property مثلاً "reg", "compatible" */
    int      length;      /* حجم الـ value بالـ bytes */
    void    *value;       /* الـ raw binary data (big-endian دايمًا) */
    struct property *next; /* linked list من الـ properties */
};
```

**مهم جداً**: الـ value دايمًا **big-endian** في الـ DT بغض النظر عن الـ CPU. ولذلك كل قراءة بتحتاج `be32_to_cpup()` أو `be16_to_cpup()`.

#### 2. `struct device_node`

عقدة في الـ DT tree — تمثّل hardware component أو logical node:

```c
struct device_node {
    const char   *name;         /* اسم الـ node مثلاً "uart0" */
    phandle       phandle;      /* رقم فريد بيخلي nodes تاني تشاور عليه */
    const char   *full_name;    /* المسار الكامل مثلاً "/soc/uart@1000" */
    struct fwnode_handle fwnode; /* الـ bridge لـ fwnode abstraction */

    struct property     *properties; /* linked list من الـ properties */
    struct device_node  *parent;
    struct device_node  *child;
    struct device_node  *sibling;
    unsigned long        _flags;     /* OF_DYNAMIC, OF_POPULATED, ... */
};
```

#### 3. `struct of_phandle_args`

بيُستخدم لما property بتشاور على node تاني مع arguments:

```c
/* مثال في DTS: clocks = <&clk_node 3>; */
struct of_phandle_args {
    struct device_node *np;           /* الـ node المُشار إليه */
    int                 args_count;   /* عدد الـ arguments (هنا 1) */
    uint32_t            args[MAX_PHANDLE_ARGS]; /* القيم (هنا [3]) */
};
```

#### 4. `struct of_endpoint`

بيمثّل نقطة اتصال في الـ OF Graph:

```c
struct of_endpoint {
    unsigned int port;              /* رقم الـ port */
    unsigned int id;                /* رقم الـ endpoint داخل الـ port */
    const struct device_node *local_node; /* الـ node نفسه */
};
```

#### 5. `struct supplier_bindings`

بيمثّل "نوع" من أنواع dependencies الممكنة في الـ DT:

```c
struct supplier_bindings {
    struct device_node *(*parse_prop)(struct device_node *np,
                                      const char *prop_name,
                                      int index);
    struct device_node *(*get_con_dev)(struct device_node *np);
    bool optional;
    u8   fwlink_flags;
};
```

---

### علاقة الـ Structs ببعض

```
struct device_node (e.g. /soc/i2c@1000)
│
├── full_name: "/soc/i2c@1000"
├── phandle: 0x17
├── fwnode: [fwnode_handle → of_fwnode_ops]
│
└── properties (linked list)
    │
    ├── struct property { name="compatible", value="vendor,i2c-v2" }
    │
    ├── struct property { name="reg", value=[0x1000, 0x100] (be32) }
    │
    ├── struct property { name="clocks", value=[phandle=0x5, idx=2] }
    │       ↓
    │   of_parse_phandle() يحلّها لـ:
    │   struct of_phandle_args { np=clk_node, args=[2] }
    │
    └── struct property { name="status", value="okay" }
```

---

### الـ Property Read API — كيف تشتغل؟

كل functions القراءة بتمر على نفس path:

```
of_property_read_u32(np, "reg", &val)
         ↓
of_find_property_value_of_size(np, "reg", sizeof(u32), 0, NULL)
         ↓
of_find_property(np, "reg", NULL)   ← يجيب struct property*
         ↓
validate length (min/max)
         ↓
be32_to_cpup(raw_value)             ← big-endian to CPU endian
         ↓
return val
```

**مثال عملي** — قراءة base address لـ UART:

```c
/* DTS:
 * uart0: serial@101F1000 {
 *     reg = <0x101F1000 0x1000>;
 *     clock-frequency = <14745600>;
 * };
 */

struct device_node *np = of_find_node_by_path("/uart0");
u32 base_addr, clk_freq;

/* قراءة أول u32 في الـ reg property */
of_property_read_u32_index(np, "reg", 0, &base_addr);  /* = 0x101F1000 */

/* قراءة u32 واحد */
of_property_read_u32(np, "clock-frequency", &clk_freq); /* = 14745600 */
```

**التحويل من big-endian** — لماذا ضروري؟

```c
/* الـ DTB بيخزن:  0x00 0xE1 0x00 0x00  (= 14745600 big-endian) */
/* على little-endian ARM: لو قرأنا raw سنحصل على 0x0000E100 = خطأ */
/* be32_to_cpup() بيصلح ده */
*out_value = be32_to_cpup(((__be32 *)val) + index);
```

---

### الـ OF Graph API — وصف الـ Hardware Connections

#### المشكلة

على SoC مثل i.MX8، عندك:
- Camera Sensor (OV5640) موصّل على **MIPI CSI-2 lane 0**
- Display Panel موصّل على **DSI port 1**

كيف الـ kernel يعرف هاللصقات دي بشكل generic بدون hardcoding؟

#### الحل: OF Graph

الـ DTS بيستخدم `port` و `endpoint` nodes بتشاور على بعض عبر `remote-endpoint` phandles:

```
/* DTS Example: Camera connected to ISP */
&ov5640 {                               /* Camera Sensor node */
    port {
        ov5640_out: endpoint {          /* output endpoint */
            remote-endpoint = <&isp_in>; /* يشاور على endpoint في الـ ISP */
        };
    };
};

&isp {                                  /* ISP node */
    port@0 {
        reg = <0>;
        isp_in: endpoint {              /* input endpoint */
            remote-endpoint = <&ov5640_out>; /* يشاور على الـ camera */
        };
    };
};
```

#### الهيكل الهرمي للـ Graph

```
device_node (ISP)
└── port@0  ← struct device_node (اسمه "port")
    └── endpoint@0  ← struct device_node (اسمه "endpoint")
        └── remote-endpoint = <phandle to camera's endpoint>

device_node (Camera)
└── port
    └── endpoint
        └── remote-endpoint = <phandle to ISP's endpoint>
```

بعض الـ devices بيكون ليها `ports` node وسيطة (لما بيكون فيه أكتر من group من الـ ports):

```
device_node (complex device)
└── ports           ← optional container
    ├── port@0
    │   └── endpoint@0
    └── port@1
        ├── endpoint@0
        └── endpoint@1
```

#### تسلسل الـ Graph Functions

```c
/* للوصول لـ remote device من endpoint محلي: */

of_graph_get_remote_endpoint(local_ep)
         ↓ (يحل الـ remote-endpoint phandle)
struct device_node *remote_ep

of_graph_get_port_parent(remote_ep)
         ↓ (يصعد 2 أو 3 مستويات حسب وجود "ports" node)
struct device_node *remote_device
```

الـ `of_graph_get_port_parent()` بتمشي 3 مستويات فوق لو مكان الـ "ports" node موجود، وإلا 2:

```c
/* Walk 3 levels up only if there is 'ports' node. */
for (depth = 3; depth && node; depth--) {
    node = of_get_next_parent(node);
    if (depth == 2 && !of_node_name_eq(node, "ports") ...)
        break;  /* توقف عند 2 مستويات لو مفيش "ports" */
}
```

---

### الـ fwnode_operations — الـ Abstraction Layer

**المشكلة**: الكيرنل بيدعم DT (ARM/embedded) و ACPI (x86). الدرايفر المثالي مايعرفش هو بيشتغل مع أنهي system.

**الحل**: `struct fwnode_operations` — vtable بيخلي الكيرنل يتعامل مع أي firmware interface بنفس الـ API.

```c
/* الكيرنل بيشوف فقط fwnode_handle، مش device_node أو acpi_handle */

struct fwnode_handle {
    struct fwnode_handle       *secondary;
    const struct fwnode_operations *ops;   /* ← الـ vtable */
    struct device              *dev;
    /* ... */
};
```

الـ `of_fwnode_ops` هو الـ vtable الخاصة بالـ DT:

```c
const struct fwnode_operations of_fwnode_ops = {
    .get                      = of_fwnode_get,
    .put                      = of_fwnode_put,
    .device_is_available      = of_fwnode_device_is_available,
    .property_present         = of_fwnode_property_present,
    .property_read_bool       = of_fwnode_property_read_bool,
    .property_read_int_array  = of_fwnode_property_read_int_array,
    /* ... */
    .graph_get_next_endpoint  = of_fwnode_graph_get_next_endpoint,
    .graph_get_remote_endpoint= of_fwnode_graph_get_remote_endpoint,
    .add_links                = of_fwnode_add_links,  /* fw_devlink */
    /* ... */
};
```

الكيرنل بعدين يعمل:

```c
/* Generic code (works for DT and ACPI): */
fwnode_property_read_u32(fwnode, "reg", &val);
    ↓
fwnode->ops->property_read_int_array(fwnode, "reg", sizeof(u32), &val, 1)
    ↓ (لو DT)
of_fwnode_property_read_int_array(...)
    ↓
of_property_read_u32_array(to_of_node(fwnode), ...)
```

---

### الـ fw_devlink — بناء الـ Device Dependency Graph

**المشكلة**: الـ clk driver لازم يكمّل probe قبل أي driver بيستخدمه. لكن الكيرنل بيشغّل الـ drivers بترتيب غير محدد.

**الحل**: `of_fwnode_add_links()` — بتمشي على كل properties في الـ node وتبحث عن phandles تشاور على suppliers:

```c
static int of_fwnode_add_links(struct fwnode_handle *fwnode)
{
    const struct property *p;
    struct device_node *con_np = to_of_node(fwnode);

    /* لكل property في الـ node: */
    for_each_property_of_node(con_np, p)
        of_link_property(con_np, p->name); /* هل دي property بتشاور على supplier؟ */

    return 0;
}
```

`of_link_property()` بتجرّب كل الـ `supplier_bindings` المعرّفة:

```
property "clocks" = <&clk_node 2>
      ↓
parse_clocks() ← تعرف إن "clocks" بيشاور على clock supplier
      ↓
of_link_to_phandle(consumer_node, clk_node, 0)
      ↓
fwnode_link_add(consumer_fwnode, clk_fwnode, 0)
      ↓ (الكيرنل بعدين يحوّل ده لـ device link)
device يـwait على الـ clk device يخلص probe أولاً
```

#### الـ supplier_bindings الموجودة

```c
DEFINE_SIMPLE_PROP(clocks,         "clocks",         "#clock-cells")
DEFINE_SIMPLE_PROP(iommus,         "iommus",         "#iommu-cells")
DEFINE_SIMPLE_PROP(dmas,           "dmas",           "#dma-cells")
DEFINE_SIMPLE_PROP(power_domains,  "power-domains",  "#power-domain-cells")
DEFINE_SIMPLE_PROP(phys,           "phys",           "#phy-cells")
DEFINE_SIMPLE_PROP(resets,         "resets",         "#reset-cells")
DEFINE_SIMPLE_PROP(pwms,           "pwms",           "#pwm-cells")
DEFINE_SUFFIX_PROP(regulators,     "-supply",        NULL)
DEFINE_SUFFIX_PROP(gpio,           "-gpio",          "#gpio-cells")
/* ... وكمان interrupts, interrupt-map, pinctrl-N, remote-endpoint, ... */
```

**الـ DEFINE_SIMPLE_PROP macro** بتولّد function اسمها `parse_<fname>`:

```c
/* الـ macro بيولّد: */
static struct device_node *parse_clocks(struct device_node *np,
                                         const char *prop_name, int index)
{
    return parse_prop_cells(np, prop_name, index, "clocks", "#clock-cells");
}
```

---

### إيه اللي الـ Subsystem ده بيملكه مقابل اللي بيفوّضه للدرايفر

| الـ Subsystem يملك | الدرايفر يملك |
|---|---|
| قراءة وتفسير raw binary data من الـ DT | تحديد معنى الـ values المقروءة |
| big-endian → CPU endian conversion | استخدام الـ value في تهيئة الـ hardware |
| بناء dependencies بين الـ devices | التعامل مع الـ dependency وقت الـ probe |
| تعريف هيكل الـ ports/endpoints | تفسير معنى الـ connections (camera, display...) |
| إتاحة الـ fwnode abstraction | ما بيحتاجش يعرف إنه DT أو ACPI |
| الـ reference counting للـ nodes | الحفاظ على lifetime الـ nodes اللي بيستخدمها |

---

### مثال عملي شامل — درايفر كاميرا

```c
/* من درايفر sensor مثل OV5640: */

static int ov5640_parse_dt(struct ov5640_dev *sensor)
{
    struct device_node *np = sensor->dev->of_node;
    struct device_node *ep;
    u32 rotation;
    int ret;

    /* قراءة property بسيطة */
    ret = of_property_read_u32(np, "rotation", &rotation);
    if (!ret)
        sensor->rotation = rotation;

    /* الحصول على أول endpoint في الـ port */
    ep = of_graph_get_next_endpoint(np, NULL);
    if (!ep) {
        dev_err(sensor->dev, "no endpoint found\n");
        return -EINVAL;
    }

    /* parse الـ endpoint لمعرفة port و id */
    struct of_endpoint endpoint;
    of_graph_parse_endpoint(ep, &endpoint);
    sensor->port_id = endpoint.port;

    /* الوصول للـ remote device (الـ ISP أو الـ CSI bridge) */
    struct device_node *remote = of_graph_get_remote_port_parent(ep);
    /* ... استخدام الـ remote node */

    of_node_put(ep);
    of_node_put(remote);
    return 0;
}
```

---

### ملاحظة عن الـ Reference Counting

كل `device_node` بيتم reference counting باستخدام `of_node_get()` / `of_node_put()`. الـ kernel الحديث بيستخدم `__free(device_node)` cleanup macro:

```c
/* الأسلوب القديم: */
struct device_node *node = of_get_child_by_name(parent, "ports");
if (node) {
    /* ... */
    of_node_put(node);  /* لازم تتذكر */
}

/* الأسلوب الحديث (موجود في property.c): */
struct device_node *node __free(device_node) =
    of_get_child_by_name(parent, "ports");
/* of_node_put يتعمل automatically عند خروج الـ scope */
```

---

### subsystems تانية محتاج تعرفها

- **OF Core** (`drivers/of/base.c`): بيوفر الـ `of_find_property()` و node traversal — الـ `property.c` بيعتمد عليه مباشرة.
- **fwnode** (`include/linux/fwnode.h`): الـ abstraction layer اللي بيوحّد DT و ACPI — الـ `property.c` بيعمّل الـ vtable الخاصة بالـ DT.
- **fw_devlink** (`drivers/base/core.c`): بيحوّل الـ fwnode links اللي `property.c` بيبنيها لـ actual device links بتتحكم في ترتيب الـ probe.
- **clk, gpio, regulator subsystems**: هي الـ "suppliers" اللي الـ fw_devlink بيبني dependencies عليها من خلال الـ `supplier_bindings` المعرّفة في `property.c`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### أولاً: flags الخاصة بالـ `device_node` (من `include/linux/of.h`)

| Flag | Value | المعنى |
|------|-------|---------|
| `OF_DYNAMIC` | 1 | الـ node والـ properties اتخصصت بـ `kmalloc` (ممكن تتحرر) |
| `OF_DETACHED` | 2 | الـ node اتفصلت من الـ device tree |
| `OF_POPULATED` | 3 | الـ device اتعمل بالفعل لهذا الـ node |
| `OF_POPULATED_BUS` | 4 | الـ platform bus اتعمل للـ children |
| `OF_OVERLAY` | 5 | اتخصص لأغراض الـ overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | موجود في changeset بيتحرر دلوقتي |

#### ثانياً: flags الخاصة بالـ `fwnode_handle` (من `include/linux/fwnode.h`)

| Flag | Bit | المعنى |
|------|-----|---------|
| `FWNODE_FLAG_LINKS_ADDED` | BIT(0) | الـ fwnode اتحللت links بتاعته قبل كده |
| `FWNODE_FLAG_NOT_DEVICE` | BIT(1) | الـ fwnode مش هيتحول لـ `struct device` أبداً |
| `FWNODE_FLAG_INITIALIZED` | BIT(2) | الـ hardware اتهيأ |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | BIT(3) | الـ driver محتاج الـ children يتبندوا الأول |
| `FWNODE_FLAG_BEST_EFFORT` | BIT(4) | يـ probe بدري حتى لو suppliers ناقصة |
| `FWNODE_FLAG_VISITED` | BIT(5) | اتزار أثناء cycle detection |

#### ثالثاً: fwnode link flags

| Flag | Bit | المعنى |
|------|-----|---------|
| `FWLINK_FLAG_CYCLE` | BIT(0) | الـ link ده جزء من cycle — مش هيؤخر الـ probe |
| `FWLINK_FLAG_IGNORE` | BIT(1) | تجاهل الـ link تماماً حتى في الـ cycle detection |

#### رابعاً: enum `dev_dma_attr`

| Value | المعنى |
|-------|---------|
| `DEV_DMA_NOT_SUPPORTED` | الـ device مش بيعمل DMA |
| `DEV_DMA_NON_COHERENT` | الـ DMA غير متزامن (non-coherent) |
| `DEV_DMA_COHERENT` | الـ DMA متزامن (coherent) |

#### خامساً: Config Options مؤثرة على الـ structs

| Config | التأثير |
|--------|---------|
| `CONFIG_OF_DYNAMIC` | يضيف `_flags` و `unique_id` للـ `property`، ويفعّل `of_node_get/put` الحقيقيين |
| `CONFIG_OF_KOBJ` | يضيف `struct kobject kobj` للـ `device_node` و `struct bin_attribute attr` للـ `property` |
| `CONFIG_OF_PROMTREE` | يضيف `unique_id` للـ `property` |
| `CONFIG_SPARC` | يضيف `_flags` للـ `property` + `unique_id` + `irq_trans` للـ `device_node` |
| `CONFIG_OF_IRQ` | يُفعّل parsing الـ `interrupts` و `interrupt-map` |
| `CONFIG_OF_ADDRESS` | يُفعّل `of_iomap` وحسابات DMA range |

---

### 1. الـ Structs المهمة

#### `struct property`

**الغرض:** تمثّل خاصية واحدة (property) داخل الـ device tree node. كل node ممكن يحتوي على قائمة linked list من الـ properties.

```c
struct property {
    char    *name;           /* اسم الـ property زي "compatible", "reg", إلخ */
    int      length;         /* حجم الـ value بالـ bytes */
    void    *value;          /* البيانات الخام (big-endian) */
    struct property *next;   /* التالي في الـ linked list */

    /* يظهر بس لو CONFIG_OF_DYNAMIC أو SPARC */
    unsigned long _flags;

    /* يظهر بس لو CONFIG_OF_PROMTREE */
    unsigned int unique_id;

    /* يظهر بس لو CONFIG_OF_KOBJ */
    struct bin_attribute attr;   /* sysfs binary attribute للـ property دي */
};
```

| Field | الدور |
|-------|-------|
| `name` | مفتاح البحث — بيستخدمه `of_find_property()` |
| `length` | بيتحقق منه كل functions القراءة للتأكد إن الـ size كافي |
| `value` | بيانات خام big-endian — بتتقرأ بـ `be32_to_cpup()` إلخ |
| `next` | الـ properties بتتخزن كـ singly-linked list |

---

#### `struct device_node`

**الغرض:** تمثّل node واحدة في الـ device tree — يعني جهاز أو bus أو أي entity في الـ hardware description.

```c
struct device_node {
    const char *name;               /* اسم الـ node (مش الـ full path) */
    phandle     phandle;            /* معرف رقمي فريد — بيستخدم للـ references */
    const char *full_name;          /* الـ path الكامل زي "/soc/i2c@1234" */
    struct fwnode_handle fwnode;    /* الـ firmware node handle — واجهة موحدة */

    struct property *properties;    /* linked list بالـ properties الحالية */
    struct property *deadprops;     /* properties اتشالت (كـ garbage collection) */
    struct device_node *parent;     /* الـ parent في شجرة الـ device tree */
    struct device_node *child;      /* أول child */
    struct device_node *sibling;    /* الـ sibling التالي (نفس الـ parent) */

    unsigned long _flags;           /* OF_DYNAMIC, OF_POPULATED, إلخ */
    void         *data;             /* بيانات خاصة بالـ arch */
};
```

| Field | الدور |
|-------|-------|
| `phandle` | بيستخدم في `of_parse_phandle()` — كـ pointer رقمي بين الـ nodes |
| `fwnode` | الـ abstraction layer — كل الـ drivers بيتعاملوا معاه مش مع الـ node مباشرة |
| `properties` | قائمة الـ properties الفعلية — بيبحث فيها `of_find_property()` |
| `deadprops` | properties اشتالت لكن مش اتحررت لحد ما الـ node تتحرر |

---

#### `struct of_phandle_args`

**الغرض:** نتيجة parsing الـ phandle مع الـ arguments بتاعته (زي clocks, DMA channels إلخ).

```c
struct of_phandle_args {
    struct device_node *np;             /* الـ node اللي الـ phandle بيشير إليه */
    int args_count;                     /* عدد الـ arguments */
    uint32_t args[MAX_PHANDLE_ARGS];    /* الـ arguments (MAX = 16) */
};
```

بيتستخدم مثلاً لما تقرأ:
```dts
clocks = <&clkc 15>;   /* phandle = &clkc, arg = 15 */
```

---

#### `struct of_phandle_iterator`

**الغرض:** iterator لقراءة list من الـ phandles داخل property واحدة — زي `interrupts-extended` اللي ممكن تحتوي على أكتر من phandle بأعداد مختلفة من الـ args.

```c
struct of_phandle_iterator {
    const char *cells_name;         /* اسم الـ property اللي بتحدد عدد الـ cells */
    int cell_count;                 /* عدد ثابت للـ cells (لو cells_name = NULL) */
    const struct device_node *parent;

    const __be32 *list_end;         /* نهاية الـ property كلها */
    const __be32 *phandle_end;      /* نهاية الـ phandle الحالي + args بتاعته */
    const __be32 *cur;              /* الموضع الحالي في القراءة */
    uint32_t cur_count;
    phandle phandle;
    struct device_node *node;       /* الـ node الحالية بعد resolve الـ phandle */
};
```

---

#### `struct of_endpoint`

**الغرض:** تمثّل endpoint واحد في الـ OF graph — يعني نقطة اتصال بين device وتاني.

```c
struct of_endpoint {
    unsigned int port;                      /* رقم الـ port (من الـ "reg" property) */
    unsigned int id;                        /* رقم الـ endpoint داخل الـ port */
    const struct device_node *local_node;   /* pointer للـ device_node الخاص بالـ endpoint */
};
```

---

#### `struct fwnode_handle`

**الغرض:** الـ abstraction الموحدة اللي بيتعامل معاها الـ kernel بدل الـ `device_node` مباشرة — بتشتغل مع OF, ACPI, swnode كمان.

```c
struct fwnode_handle {
    struct fwnode_handle *secondary; /* فولباك لـ fwnode تاني (زي ACPI fallback) */
    const struct fwnode_operations *ops; /* الـ vtable — ops مختلفة حسب المصدر */
    struct device *dev;              /* الـ device المرتبط (لو اتعمل) */
    struct list_head suppliers;      /* قائمة الـ fwnode links للـ suppliers */
    struct list_head consumers;      /* قائمة الـ fwnode links للـ consumers */
    u8 flags;                        /* FWNODE_FLAG_* */
};
```

---

#### `struct fwnode_operations`

**الغرض:** الـ vtable اللي بيربط الـ `fwnode_handle` بالـ implementation الفعلية (OF في حالتنا). الـ `of_fwnode_ops` المُعرَّفة في نهاية الملف هي instance واحدة منه.

| Op | الـ Implementation في الملف |
|----|------------------------------|
| `.get` / `.put` | `of_fwnode_get` / `of_fwnode_put` → `of_node_get/put` |
| `.device_is_available` | `of_fwnode_device_is_available` → `of_device_is_available` |
| `.property_present` | `of_fwnode_property_present` → `of_property_present` |
| `.property_read_bool` | `of_fwnode_property_read_bool` → `of_property_read_bool` |
| `.property_read_int_array` | `of_fwnode_property_read_int_array` → switch على elem_size |
| `.property_read_string_array` | `of_fwnode_property_read_string_array` |
| `.get_name` / `.get_name_prefix` | `of_fwnode_get_name` → `kbasename(full_name)` |
| `.get_parent` | `of_fwnode_get_parent` → `of_get_parent` |
| `.get_next_child_node` | → `of_get_next_available_child` |
| `.get_named_child_node` | → loop على available children |
| `.get_reference_args` | `of_fwnode_get_reference_args` → `of_parse_phandle_with_args` |
| `.graph_get_next_endpoint` | → `of_graph_get_next_endpoint` |
| `.graph_get_remote_endpoint` | → `of_graph_get_remote_endpoint` |
| `.graph_get_port_parent` | → `of_fwnode_graph_get_port_parent` |
| `.graph_parse_endpoint` | → `of_fwnode_graph_parse_endpoint` |
| `.iomap` | → `of_iomap` |
| `.irq_get` | → `of_irq_get` |
| `.add_links` | `of_fwnode_add_links` → `of_link_property` لكل property |

---

#### `struct supplier_bindings`

**الغرض:** يربط اسم property بدالة parsing تعرف تستخرج منها الـ supplier node. بيُستخدم في الـ `fw_devlink` لبناء dependency graph تلقائياً.

```c
struct supplier_bindings {
    /* دالة بتأخذ node + اسم property + index وترجع supplier node */
    struct device_node *(*parse_prop)(struct device_node *np,
                                      const char *prop_name, int index);

    /* لو الـ consumer node مش بيتحول لـ device — بتدور على الـ "الأب الحقيقي" */
    struct device_node *(*get_con_dev)(struct device_node *np);

    bool optional;        /* لو true: يتجاهله لو fw_devlink مش strict */
    u8 fwlink_flags;      /* FWLINK_FLAG_* — زي IGNORE للـ post-init providers */
};
```

---

#### `struct fwnode_link`

**الغرض:** يمثّل edge في الـ dependency graph بين supplier وconsumer — بيستخدمه الـ `fw_devlink`.

```c
struct fwnode_link {
    struct fwnode_handle *supplier;
    struct list_head s_hook;   /* hook في قائمة الـ consumers بتاعة الـ supplier */
    struct fwnode_handle *consumer;
    struct list_head c_hook;   /* hook في قائمة الـ suppliers بتاعة الـ consumer */
    u8 flags;                  /* FWLINK_FLAG_CYCLE أو FWLINK_FLAG_IGNORE */
};
```

---

#### `struct alias_prop` (من `of_private.h`)

**الغرض:** يمثّل entry واحدة في الـ `aliases` node — يربط اسم مختصر بـ device_node.

```c
struct alias_prop {
    struct list_head link;  /* linked في aliases_lookup list */
    const char *alias;      /* الاسم المختصر زي "serial0" */
    struct device_node *np; /* الـ node الحقيقي زي "/soc/uart@1234" */
    int id;                 /* الرقم في آخر الاسم — "serial0" → 0 */
    char stem[];            /* الجزء اللي قبل الرقم — "serial" */
};
```

---

### 2. مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                        device tree (شجرة)                        │
│                                                                   │
│   device_node (root)                                              │
│   ├── *parent = NULL                                              │
│   ├── *child ──────────────────────► device_node (child-1)       │
│   │                                  ├── *parent ◄──────────────┘│
│   │                                  ├── *sibling ──► device_node │
│   │                                  ├── *properties             │
│   │                                  │   ├── property ("compatible")│
│   │                                  │   │   └── *next            │
│   │                                  │   └── property ("reg")     │
│   │                                  │       └── *next = NULL     │
│   │                                  └── fwnode_handle            │
│   │                                      └── *ops = &of_fwnode_ops│
│   └── *properties                                                 │
│       └── property ("model") → *next → ...                        │
└─────────────────────────────────────────────────────────────────┘
```

```
fwnode_handle ─── ops ──► fwnode_operations (vtable)
     │                     ├── .get()
     │                     ├── .put()
     │                     ├── .property_read_int_array()
     │                     ├── .graph_get_next_endpoint()
     │                     └── .add_links()
     │
     ├── suppliers ◄──── fwnode_link ────► fwnode_handle (supplier)
     └── consumers ◄──── fwnode_link ────► fwnode_handle (consumer)
```

```
of_phandle_args
├── *np ───► device_node (الـ supplier المشار إليه)
├── args_count
└── args[16]

of_phandle_iterator
├── *parent ───► device_node
├── *node ─────► device_node (current resolved node)
├── *cur, *list_end, *phandle_end  (pointers داخل property->value)
└── cells_name → لو بتقرأ "#clock-cells" مثلاً
```

---

### 3. مخططات الـ OF Graph (port/endpoint)

#### بنية الـ DTS

```
device-A {
    ports {
        port@0 {                      ← device_node (port)
            reg = <0>;
            ep0: endpoint@0 {         ← device_node (endpoint)
                reg = <0>;
                remote-endpoint = <&ep1>;  ← phandle لـ device-B
            };
        };
        port@1 { ... };
    };
};

device-B {
    port@0 {
        ep1: endpoint@0 {
            remote-endpoint = <&ep0>;
    };
};
```

#### مخطط العلاقات في الذاكرة

```
device_node (device-A)
└── child: device_node (ports)
    └── child: device_node (port@0)    ← of_endpoint.port = 0
        └── child: device_node (endpoint@0)  ← of_endpoint.id = 0
            └── property "remote-endpoint" → phandle → device_node (endpoint@0 in B)

device_node (device-B)
└── child: device_node (port@0)
    └── child: device_node (endpoint@0)
        └── property "remote-endpoint" → phandle → device_node (endpoint@0 in A)
```

#### الـ `struct of_endpoint` بعد parse

```
of_endpoint {
    port       = 0             ← من reg بتاع port@0
    id         = 0             ← من reg بتاع endpoint@0
    local_node → device_node (endpoint@0 in A)
}
```

---

### 4. مخططات الـ Lifecycle

#### أ. قراءة property عادية

```
Creation (DTS compile → FDT blob)
    ↓
Kernel boot: __unflatten_device_tree()
    → يخلق device_node لكل node
    → يخلق property لكل property في الـ FDT
    → يربطهم: node->properties = linked list of properties
    ↓
Usage: of_find_property(np, "reg", NULL)
    → loop على np->properties
    → يقارن name بـ strcmp/of_prop_cmp
    → يرجع pointer للـ property
    ↓
Read: of_property_read_u32(np, "reg", &val)
    → of_find_property_value_of_size() ← validates length
    → be32_to_cpup() ← converts endianness
    ↓
Teardown (CONFIG_OF_DYNAMIC only):
    of_node_put() → refcount → 0 → of_node_release()
    → __of_prop_free() لكل property
```

#### ب. lifecycle الـ fwnode_handle

```
device_node init (of_node_init):
    fwnode_init(&node->fwnode, &of_fwnode_ops)
    ↓
Driver access via generic API:
    fwnode_property_read_u32(fwnode, "reg", &val)
    → fwnode->ops->property_read_int_array(fwnode, "reg", 4, &val, 1)
    → of_fwnode_property_read_int_array()
    → of_property_read_u32_array(to_of_node(fwnode), "reg", &val, 1)
    ↓
Teardown:
    fwnode_handle_put(fwnode)
    → fwnode->ops->put(fwnode)
    → of_fwnode_put()
    → of_node_put()
```

#### ج. lifecycle الـ fw_devlink (supplier/consumer links)

```
of_fwnode_add_links(fwnode):
    for_each_property_of_node(con_np, p):
        of_link_property(con_np, p->name)
            → loop على of_supplier_bindings[]
            → s->parse_prop(con_np, prop_name, i)
                → مثلاً parse_clocks():
                    → parse_prop_cells(np, "clocks", "#clock-cells", 0, i)
                    → of_parse_phandle_with_args()
                    → يرجع supplier node
            → of_link_to_phandle(con_dev_np, supplier_np, flags)
                → يتحقق إن الـ supplier available
                → fwnode_link_add(consumer_fwnode, supplier_fwnode, flags)
                    → يخلق fwnode_link ويضيفه للـ lists
    ↓
fw_devlink يستخدم الـ links:
    → يؤخر probe الـ consumer لحد ما الـ supplier يـ probe
```

---

### 5. مخططات تدفق الاستدعاء (Call Flow)

#### أ. قراءة array من integers

```
driver calls:
  of_property_read_u32_array(np, "reg", out, count)
    └─► of_find_property_value_of_size(np, "reg",
                min = count * sizeof(u32),
                max = 0,
                len = NULL)
          └─► of_find_property(np, "reg", NULL)
                └─► loop على np->properties (تحت devtree_lock أو rcu)
                      └─► strcmp(prop->name, "reg")
          ← returns prop->value ptr أو ERR_PTR
    └─► loop: *out++ = be32_to_cpup(val++)
    ← returns 0 أو error code
```

#### ب. traversal الـ OF graph

```
driver calls:
  for_each_endpoint_of_node(np, endpoint)
    └─► of_graph_get_next_endpoint(parent, prev)
          ├─ [prev=NULL]: of_graph_get_next_port(parent, NULL)
          │     └─► of_get_child_by_name(parent, "ports")  ← optional
          │     └─► of_get_child_by_name(parent or ports, "port")
          └─► of_graph_get_next_port_endpoint(port, prev)
                └─► of_get_next_child(port, prev)
                      [يتحقق إن اسم الـ child = "endpoint"]
          ← returns endpoint node (refcount++)
    ↓
  of_graph_parse_endpoint(endpoint, &ep)
    └─► of_get_parent(endpoint) → port_node
    └─► of_property_read_u32(port_node, "reg", &ep.port)
    └─► of_property_read_u32(endpoint, "reg", &ep.id)
    ← ep.port, ep.id, ep.local_node جاهزين
```

#### ج. الوصول للـ remote device

```
driver calls:
  of_graph_get_remote_node(local_np, port=0, endpoint=0)
    └─► of_graph_get_endpoint_by_regs(local_np, 0, 0)
          └─► for_each_endpoint_of_node(local_np, node)
                └─► of_graph_parse_endpoint(node, &ep)
                └─► check: ep.port==0 && ep.id==0
          ← returns endpoint node
    └─► of_graph_get_remote_port_parent(endpoint_node)
          └─► of_graph_get_remote_endpoint(endpoint_node)
                └─► of_parse_phandle(node, "remote-endpoint", 0)
                      └─► يقرأ الـ phandle value
                      └─► يبحث في الـ tree بالـ phandle ID
                      ← remote endpoint node
          └─► of_graph_get_port_parent(remote_ep_node)
                └─► of_node_get(node)
                └─► walk up 3 levels:
                      endpoint → port → [ports?] → device
                ← remote device_node
    └─► of_device_is_available(remote) ← يتحقق "status" property
    ← remote device_node (refcount++)
```

#### د. تدفق الـ fwnode_operations (generic path)

```
generic driver code:
  fwnode_property_read_u32(fwnode, "reg", &val)
    └─► fwnode->ops->property_read_int_array(fwnode, "reg", 4, &val, 1)
          [ops = &of_fwnode_ops]
          └─► of_fwnode_property_read_int_array()
                └─► node = to_of_node(fwnode)
                    [= container_of(fwnode, struct device_node, fwnode)]
                └─► switch(elem_size=4): of_property_read_u32_array(node,...)
                ← 0 or error
```

---

### 6. استراتيجية الـ Locking

#### الـ Locks المستخدمة

| Lock | النوع | أين معرّف | يحمي إيه |
|------|-------|-----------|-----------|
| `devtree_lock` | `raw_spinlock_t` | `of_private.h` | الـ device tree كله في حالة الـ read (بيستخدم RCU بديلاً في بعض paths) |
| `of_mutex` | `struct mutex` | `of_private.h` | العمليات الـ blocking زي الـ changeset apply وإضافة/حذف nodes |
| `of_overlay_mutex` | internal mutex | overlay code | عمليات الـ overlay |
| RCU | read-side لا يحتاج lock | kernel | traversal الـ tree في الـ read-only paths |

#### في `property.c` تحديداً

- الـ functions في الملف ده **لا تأخذ locks مباشرة** — دي مسؤولية الـ `of_find_property()` اللي بتتضمن RCU read lock داخلياً.
- `of_find_property()` بتشتغل تحت `rcu_read_lock()` في الـ paths المعتادة.
- الـ `fwnode_link_add()` المستدعاة من `of_link_to_phandle()` بتأخذ lock خاص بيها على الـ link lists.
- الـ `__free(device_node)` cleanup macro بيضمن إن `of_node_put()` يتنادى تلقائياً عند الخروج من الـ scope — ده pattern C23 `__attribute__((cleanup))`.

#### ترتيب الـ Locks (Lock Ordering)

```
of_mutex  (أعلى — outer)
    └── devtree_lock / RCU  (أدنى — inner)
```

ممنوع تمسك `of_mutex` وأنت شايل `devtree_lock` — هيعمل deadlock.

---

### ملخص سريع للـ DEFINE_SIMPLE_PROP و DEFINE_SUFFIX_PROP

الاتنين macros بيولدوا دوال `parse_*` اللي بتملأ جدول `of_supplier_bindings[]`:

```c
/* DEFINE_SIMPLE_PROP: property باسم ثابت */
DEFINE_SIMPLE_PROP(clocks, "clocks", "#clock-cells")
/* ينتج: */
static struct device_node *parse_clocks(np, prop_name, index) {
    return parse_prop_cells(np, prop_name, index, "clocks", "#clock-cells");
}

/* DEFINE_SUFFIX_PROP: property بنهاية متغيرة */
DEFINE_SUFFIX_PROP(regulators, "-supply", NULL)
/* ينتج: */
static struct device_node *parse_regulators(np, prop_name, index) {
    return parse_suffix_prop_cells(np, prop_name, index, "-supply", NULL);
}
```

الـ `of_supplier_bindings[]` يحتوي دلوقتي على 30+ binding يغطي: clocks, iommus, mboxes, io-channels, dmas, power-domains, hwlocks, phys, pwms, resets, gpios, interrupts, regulators, إلخ.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Group 1: Property Reading — Scalar & Indexed

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `of_property_read_bool` | `bool (np, propname)` | وجود property بدون قيمة |
| `of_property_count_elems_of_size` | `int (np, propname, elem_size)` | عدد عناصر property بحجم معين |
| `of_find_property_value_of_size` | `void* (np, propname, min, max, len)` | helper داخلي للـ validation |
| `of_property_read_u8_index` | `int (np, propname, index, *out)` | قراءة u8 في index محدد |
| `of_property_read_u16_index` | `int (np, propname, index, *out)` | قراءة u16 في index محدد |
| `of_property_read_u32_index` | `int (np, propname, index, *out)` | قراءة u32 في index محدد |
| `of_property_read_u64_index` | `int (np, propname, index, *out)` | قراءة u64 في index محدد |
| `of_property_read_u64` | `int (np, propname, *out)` | قراءة u64 واحدة |

#### Group 2: Property Reading — Arrays

| Function | الغرض |
|---|---|
| `of_property_read_variable_u8_array` | array من u8 بحدود min/max |
| `of_property_read_variable_u16_array` | array من u16 بحدود min/max |
| `of_property_read_variable_u32_array` | array من u32 بحدود min/max |
| `of_property_read_variable_u64_array` | array من u64 بحدود min/max |

#### Group 3: Property Reading — Strings

| Function | الغرض |
|---|---|
| `of_property_read_string` | قراءة أول string |
| `of_property_match_string` | البحث عن string وإرجاع index |
| `of_property_read_string_helper` | helper للـ string arrays مع skip |
| `of_prop_next_u32` | iterator على u32 values |
| `of_prop_next_string` | iterator على strings |

#### Group 4: OF Graph — Topology Navigation

| Function | الغرض |
|---|---|
| `of_graph_is_present` | هل في graph ports؟ |
| `of_graph_parse_endpoint` | تحليل endpoint node → `of_endpoint` |
| `of_graph_get_port_by_id` | port بـ reg id محدد |
| `of_graph_get_next_port` | iteration على ports |
| `of_graph_get_next_port_endpoint` | iteration على endpoints في port |
| `of_graph_get_next_endpoint` | iteration على كل endpoints في device |
| `of_graph_get_endpoint_by_regs` | endpoint بـ port_reg و reg |
| `of_graph_get_remote_endpoint` | remote endpoint عبر phandle |
| `of_graph_get_port_parent` | parent device من endpoint |
| `of_graph_get_remote_port_parent` | remote device node |
| `of_graph_get_remote_port` | remote port node |
| `of_graph_get_endpoint_count` | عدد endpoints في device |
| `of_graph_get_port_count` | عدد ports في device |
| `of_graph_get_remote_node` | remote device بـ port/endpoint IDs |

#### Group 5: fwnode Operations (OF → fwnode Bridge)

| Function | الغرض |
|---|---|
| `of_fwnode_get/put` | refcount management |
| `of_fwnode_device_is_available` | status check عبر fwnode |
| `of_fwnode_property_present` | وجود property عبر fwnode |
| `of_fwnode_property_read_bool` | bool property عبر fwnode |
| `of_fwnode_property_read_int_array` | int array عبر fwnode |
| `of_fwnode_property_read_string_array` | string array عبر fwnode |
| `of_fwnode_get_name` | اسم الـ node |
| `of_fwnode_get_parent` | parent node عبر fwnode |
| `of_fwnode_get_next_child_node` | next child عبر fwnode |
| `of_fwnode_get_named_child_node` | child بالاسم عبر fwnode |
| `of_fwnode_get_reference_args` | phandle args عبر fwnode |
| `of_fwnode_graph_get_next_endpoint` | graph iteration عبر fwnode |
| `of_fwnode_graph_get_remote_endpoint` | remote endpoint عبر fwnode |
| `of_fwnode_graph_get_port_parent` | port parent عبر fwnode |
| `of_fwnode_graph_parse_endpoint` | parse endpoint عبر fwnode |
| `of_fwnode_iomap` | IO memory mapping عبر fwnode |
| `of_fwnode_irq_get` | IRQ number عبر fwnode |
| `of_fwnode_add_links` | fw_devlink: ربط consumer بـ suppliers |

#### Group 6: fw_devlink — Supplier Parsing

| Function / Macro | الغرض |
|---|---|
| `parse_prop_cells` | helper: parse phandle property باسم ثابت |
| `parse_suffix_prop_cells` | helper: parse phandle property بـ suffix |
| `DEFINE_SIMPLE_PROP(fname, name, cells)` | ينشئ `parse_##fname` لـ property باسم ثابت |
| `DEFINE_SUFFIX_PROP(fname, suffix, cells)` | ينشئ `parse_##fname` لـ property بـ suffix |
| `parse_pinctrl_n` | parse pinctrl-N properties |
| `parse_gpios` | parse -gpios suffix (مع exclusion لـ nr-gpios) |
| `parse_iommu_maps` | parse iommu-map (كل 4 cells) |
| `parse_gpio_compat` | parse gpio/gpios الـ legacy |
| `parse_interrupts` | parse interrupts/interrupts-extended |
| `parse_interrupt_map` | parse interrupt-map complex binding |
| `parse_remote_endpoint` | parse remote-endpoint في graph |
| `of_link_property` | يمشي على `of_supplier_bindings` ويضيف fwnode links |
| `of_link_to_phandle` | يضيف fwnode_link لـ supplier بعد availability check |
| `of_fwnode_add_links` | entry point: يتكرر على كل properties ويستدعي `of_link_property` |

---

### Group 1: Property Reading — قراءة القيم العددية

الـ group ده هو اللب الأساسي للـ DT property access. الفكرة كلها بتتمحور حول **`of_find_property_value_of_size`** كـ internal helper بيتحقق من وجود الـ property وحجمها قبل ما يرجع pointer للـ raw data. كل functions القراءة العددية بتستدعيه وبتطبق endian conversion (big-endian في DT → native CPU).

---

#### `of_property_read_bool`

```c
bool of_property_read_bool(const struct device_node *np, const char *propname);
```

بيدور على الـ property بالاسم، ولو لقيها بيرجع `true`. بيطبع warning لو الـ property ليها value لأن الـ boolean properties المفروض تكون empty (zero-length). مفيد لـ properties زي `wakeup-source` أو `big-endian`.

- **`np`**: الـ device node اللي هندور فيه
- **`propname`**: اسم الـ property
- **Return**: `true` لو موجودة، `false` لو لأ
- **Key detail**: الـ warning مش error — الـ function بتشتغل صح حتى مع الـ non-empty properties، بس deprecated usage

---

#### `of_property_count_elems_of_size`

```c
int of_property_count_elems_of_size(const struct device_node *np,
                                    const char *propname, int elem_size);
```

بيحسب عدد العناصر في property بالقسمة `prop->length / elem_size`. مفيد جداً قبل allocate array لقراءة values.

- **`np`**: الـ device node
- **`propname`**: اسم الـ property
- **`elem_size`**: حجم العنصر الواحد (مثلاً `sizeof(u32)` = 4)
- **Return**: عدد العناصر على النجاح، `-EINVAL` لو property مش موجودة أو length مش multiple من elem_size، `-ENODATA` لو property بدون value
- **Key detail**: بيطبع `pr_err` لو الـ length مش متوافق مع elem_size قبل ما يرجع `-EINVAL`

---

#### `of_find_property_value_of_size` (static)

```c
static void *of_find_property_value_of_size(const struct device_node *np,
            const char *propname, u32 min, u32 max, size_t *len);
```

**Internal helper** بتستخدمه كل functions القراءة العددية. بيتحقق من:
1. وجود الـ property
2. وجود value (مش NULL)
3. الـ length >= min
4. لو max != 0: الـ length <= max

بيكتب الـ actual length في `*len` لو مش NULL.

- **Return**: pointer للـ raw property data، أو `ERR_PTR` بـ:
  - `-EINVAL`: property مش موجودة
  - `-ENODATA`: property بدون value
  - `-EOVERFLOW`: size خارج النطاق
- **Key detail**: بيستخدم `ERR_PTR/IS_ERR/PTR_ERR` pattern — كل الـ callers بيعملوا `IS_ERR(val)` check

---

#### `of_property_read_u8_index / u16_index / u32_index / u64_index`

```c
int of_property_read_u32_index(const struct device_node *np,
                               const char *propname,
                               u32 index, u32 *out_value);
```

بتقرأ عنصر واحد من property في index محدد (0-based). بتمرر `(index+1) * sizeof(T)` كـ min size للـ helper عشان تتأكد إن الـ property كبيرة بما يكفي للوصول للـ index ده. بعدين بتطبق endian conversion.

- **`index`**: رقم العنصر المطلوب (0-based)
- **`out_value`**: pointer للـ output، متغيرش لو في error
- **Return**: 0 على النجاح، negative error code
- **Endian**: u16 → `be16_to_cpup`, u32 → `be32_to_cpup`, u64 → `be64_to_cpup`, u8 → copy مباشر (no endian)

---

#### `of_property_read_u64`

```c
int of_property_read_u64(const struct device_node *np, const char *propname,
                         u64 *out_value);
```

بتقرأ u64 واحدة من بداية الـ property. لاحظ إنها بتستخدم `of_read_number(val, 2)` مش `be64_to_cpup` — الـ DT بيخزن u64 كـ two consecutive big-endian u32 cells.

- **Return**: 0 على النجاح، negative error code

---

### Group 2: Variable-Size Array Reading

نفس فكرة الـ indexed functions بس للـ arrays. كل function بتاخد `sz_min` و `sz_max` عشان تحدد النطاق المقبول.

#### `of_property_read_variable_u32_array` (ومثيلاتها u8/u16/u64)

```c
int of_property_read_variable_u32_array(const struct device_node *np,
                       const char *propname, u32 *out_values,
                       size_t sz_min, size_t sz_max);
```

بتستدعي `of_find_property_value_of_size` بـ `sz_min * sizeof(T)` و `sz_max * sizeof(T)`. لو `sz_max == 0` يبقى مفيش upper limit وبتقرأ `sz_min` عناصر بس. لو `sz_max > 0` بتقرأ الـ actual count `(actual_bytes / sizeof(T))`.

**Pseudocode:**
```
val = of_find_property_value_of_size(np, propname, sz_min*4, sz_max*4, &sz)
if IS_ERR(val) → return PTR_ERR(val)

if sz_max == 0:
    count = sz_min          // read minimum only
else:
    count = sz / sizeof(T)  // read all available

for count times:
    *out_values++ = be32_to_cpup(val++)

return count
```

- **Return**: عدد العناصر المقروءة على النجاح، negative على الفشل
- **Key detail**: الـ u64 variant بتقرأ كل element بـ `of_read_number(val, 2)` وبتزود الـ pointer بـ 2 cells (8 bytes)

---

### Group 3: String Properties

#### `of_property_read_string`

```c
int of_property_read_string(const struct device_node *np, const char *propname,
                            const char **out_string);
```

بترجع **pointer مباشر** لـ property data (مش copy). بتتحقق إن الـ string فيها null terminator في حدود الـ property length بـ `strnlen`.

- **Return**: 0 على النجاح، `-EINVAL` / `-ENODATA` / `-EILSEQ` (لو مفيش null terminator)
- **Key detail**: الـ empty string `""` ليها length 1 (الـ null byte)، فـ `-ENODATA` مش معناها empty string هنا

---

#### `of_property_match_string`

```c
int of_property_match_string(const struct device_node *np, const char *propname,
                             const char *string);
```

بتمشي على string list (strings متتابعة separated بـ null bytes) وبتدور على exact match. مفيد لـ properties زي `clock-names`, `power-domain-names`.

**Pseudocode:**
```
p = prop->value
end = p + prop->length
for i = 0; p < end; i++:
    l = strnlen(p, end-p) + 1
    if p+l > end → return -EILSEQ
    if strcmp(string, p) == 0 → return i   // found!
    p += l
return -ENODATA  // not found
```

- **Return**: index (0-based) على النجاح، negative على الفشل

---

#### `of_property_read_string_helper`

```c
int of_property_read_string_helper(const struct device_node *np,
                   const char *propname, const char **out_strs,
                   size_t sz, int skip);
```

**Utility helper** — لا تستخدمها مباشرة. بتقدر تعمل skip على أول N strings وبتملأ array بـ `sz` strings. لو `out_strs == NULL` بتعد عدد الـ strings الكلي. الـ macros `of_property_read_string_array` و `of_property_count_strings` بنية فوقيها.

- **`skip`**: عدد الـ strings اللي هنتجاوزها
- **`sz`**: عدد الـ strings المطلوبة
- **Return**: عدد الـ strings المقروءة فعلاً، negative error

---

#### `of_prop_next_u32` و `of_prop_next_string`

```c
const __be32 *of_prop_next_u32(const struct property *prop,
                               const __be32 *cur, u32 *pu);
const char *of_prop_next_string(const struct property *prop, const char *cur);
```

**Iterators** بتستخدم في for loops. لو `cur == NULL` بتبدأ من أول الـ property. كل call بتجيب القيمة التالية وبترجع pointer ليها (أو NULL لو خلصت).

مثال استخدام:
```c
const __be32 *cur = NULL;
u32 val;
while ((cur = of_prop_next_u32(prop, cur, &val)))
    pr_info("val = %u\n", val);
```

---

### Group 4: OF Graph — Topology Navigation

الـ OF graph binding بيصف الـ hardware connections بين devices (كاميرات، displays، audio codecs، إلخ). الـ topology:

```
device_node
└── port@0                    (port node, reg=0)
│   └── endpoint@0            (endpoint node, reg=0)
│       └── remote-endpoint → phandle to remote endpoint
└── ports                     (optional grouping node)
    └── port@1
        └── endpoint@0
```

أو بـ `in-ports` / `out-ports` للـ directional devices.

---

#### `of_graph_is_present`

```c
bool of_graph_is_present(const struct device_node *node);
```

بيتحقق إن الـ device node فيه graph connections. بيدور على `ports` child أو `port` child مباشرة. مفيد للـ drivers اللي بتدعم devices بـ graph وبدونها.

- **Return**: `true` لو في port node، `false` لو لأ
- **Key detail**: بيستخدم `__free(device_node)` cleanup — modern kernel memory management

---

#### `of_graph_parse_endpoint`

```c
int of_graph_parse_endpoint(const struct device_node *node,
                            struct of_endpoint *endpoint);
```

بيملأ `struct of_endpoint` من endpoint node. بيقرأ `reg` property من الـ port (للـ `endpoint->port`) ومن الـ endpoint نفسه (للـ `endpoint->id`). الـ caller لازم يمسك reference على `node`.

```c
struct of_endpoint {
    unsigned int port;       // من reg في parent port node
    unsigned int id;         // من reg في endpoint node نفسه
    const struct device_node *local_node;  // pointer للـ endpoint node
};
```

- **Return**: دايماً 0 (الـ reg reads اختيارية، default = 0 لو مش موجودة)
- **Key detail**: `memset(endpoint, 0, ...)` أول حاجة — safe default values

---

#### `of_graph_get_port_by_id`

```c
struct device_node *of_graph_get_port_by_id(struct device_node *parent, u32 id);
```

بيمشي على كل child nodes، بيفلتر اللي اسمها `port`، ويقارن `reg` property بالـ `id` المطلوب.

- **Return**: port node بـ incremented refcount، أو NULL — caller لازم `of_node_put()`
- **Key detail**: بيتحقق من `ports` wrapper node أولاً (تلقائياً)

---

#### `of_graph_get_next_port`

```c
struct device_node *of_graph_get_next_port(const struct device_node *parent,
                                           struct device_node *prev);
```

Iterator على الـ port nodes. لو `prev == NULL` بيرجع أول port. في كل iteration بيستدعي `of_get_next_child` ويتجاهل nodes اللي اسمها مش `port`.

- **Return**: next port node بـ incremented refcount، refcount الـ `prev` بيتـ decrement
- **Key detail**: بيتعامل مع `ports` wrapper تلقائياً في الـ `prev == NULL` case

---

#### `of_graph_get_next_port_endpoint`

```c
struct device_node *of_graph_get_next_port_endpoint(
    const struct device_node *port, struct device_node *prev);
```

Iterator على الـ endpoints داخل port واحد بس. بيطبع `WARN` لو لقى child مش اسمه `endpoint`.

- **Key detail**: مختلف عن `of_graph_get_next_endpoint` — ده محدود بـ port واحد

---

#### `of_graph_get_next_endpoint`

```c
struct device_node *of_graph_get_next_endpoint(const struct device_node *parent,
                                               struct device_node *prev);
```

Iterator شامل على **كل** endpoints في device. بيتنقل بين ports تلقائياً.

**Pseudocode:**
```
if prev == NULL:
    port = first port in parent
else:
    port = parent of prev

loop:
    endpoint = of_graph_get_next_port_endpoint(port, prev)
    if endpoint:
        of_node_put(port)
        return endpoint

    // no more endpoints in this port, try next
    prev = NULL
    port = of_graph_get_next_port(parent, port)
    if !port: return NULL
```

- **Key detail**: بيتحكم في refcounts بشكل صحيح — `of_node_put(port)` قبل ما يرجع endpoint

---

#### `of_graph_get_endpoint_by_regs`

```c
struct device_node *of_graph_get_endpoint_by_regs(
    const struct device_node *parent, int port_reg, int reg);
```

بيدور على endpoint بـ port ID و endpoint ID محددين. لو القيمة `-1` يبقى هنتجاهل المقارنة دي (wildcard).

- **Return**: endpoint node بـ incremented refcount، أو NULL
- **Key detail**: بيستخدم `for_each_endpoint_of_node` macro داخلياً

---

#### `of_graph_get_remote_endpoint`

```c
struct device_node *of_graph_get_remote_endpoint(const struct device_node *node);
```

بيرجع الـ remote endpoint بـ parse الـ `remote-endpoint` phandle property.

- **Return**: remote endpoint node بـ incremented refcount

---

#### `of_graph_get_port_parent`

```c
struct device_node *of_graph_get_port_parent(struct device_node *node);
```

بيمشي من endpoint → port → [ports|in-ports|out-ports (optional)] → device. بيعمل `of_node_get` على الـ node الأول عشان `of_get_next_parent` بيعمل `of_node_put`.

**Logic:**
```
node_get(node)          // preserve refcount
walk up to 3 levels:
    node = get_next_parent(node)
    if depth==2 and node is NOT "ports"/"in-ports"/"out-ports":
        break early (no ports wrapper)
return node             // the device node
```

- **Key detail**: بيتعامل مع depth 2 (ports wrapper) أو depth 1 (direct port under device)

---

#### `of_graph_get_remote_port_parent`

```c
struct device_node *of_graph_get_remote_port_parent(
    const struct device_node *node);
```

= `of_graph_get_port_parent(of_graph_get_remote_endpoint(node))`. يوصل للـ remote device من local endpoint مباشرة.

---

#### `of_graph_get_remote_port`

```c
struct device_node *of_graph_get_remote_port(const struct device_node *node);
```

= `of_get_next_parent(of_graph_get_remote_endpoint(node))`. بيرجع port الـ remote endpoint، مش الـ device.

---

#### `of_graph_get_endpoint_count` و `of_graph_get_port_count`

```c
unsigned int of_graph_get_endpoint_count(const struct device_node *np);
unsigned int of_graph_get_port_count(struct device_node *np);
```

بيعدوا كل endpoints / ports في device باستخدام `for_each_endpoint_of_node` و `for_each_of_graph_port`.

---

#### `of_graph_get_remote_node`

```c
struct device_node *of_graph_get_remote_node(const struct device_node *node,
                                             u32 port, u32 endpoint);
```

High-level helper: بياخد device node + port ID + endpoint ID وبيرجع الـ remote device. بيفحص `of_device_is_available` على الـ remote — لو مش available بيرجع NULL.

**Flow:**
```
endpoint_node = of_graph_get_endpoint_by_regs(node, port, endpoint)
if !endpoint_node → return NULL

remote = of_graph_get_remote_port_parent(endpoint_node)
of_node_put(endpoint_node)
if !remote → return NULL

if !of_device_is_available(remote):
    of_node_put(remote)
    return NULL

return remote
```

---

### Group 5: fwnode Operations — الجسر بين OF و Generic Firmware Node API

الـ **`fwnode_handle`** هو abstraction layer عشان kernel code يشتغل مع ACPI و DT و software nodes بنفس API. الـ `of_fwnode_ops` struct بيربط الـ generic operations بالـ OF implementations.

```c
const struct fwnode_operations of_fwnode_ops = {
    .get                      = of_fwnode_get,
    .put                      = of_fwnode_put,
    .device_is_available      = of_fwnode_device_is_available,
    .device_get_match_data    = of_fwnode_device_get_match_data,
    .device_dma_supported     = of_fwnode_device_dma_supported,
    .device_get_dma_attr      = of_fwnode_device_get_dma_attr,
    .property_present         = of_fwnode_property_present,
    .property_read_bool       = of_fwnode_property_read_bool,
    .property_read_int_array  = of_fwnode_property_read_int_array,
    .property_read_string_array = of_fwnode_property_read_string_array,
    .get_name                 = of_fwnode_get_name,
    .get_name_prefix          = of_fwnode_get_name_prefix,
    .get_parent               = of_fwnode_get_parent,
    .get_next_child_node      = of_fwnode_get_next_child_node,
    .get_named_child_node     = of_fwnode_get_named_child_node,
    .get_reference_args       = of_fwnode_get_reference_args,
    .graph_get_next_endpoint  = of_fwnode_graph_get_next_endpoint,
    .graph_get_remote_endpoint= of_fwnode_graph_get_remote_endpoint,
    .graph_get_port_parent    = of_fwnode_graph_get_port_parent,
    .graph_parse_endpoint     = of_fwnode_graph_parse_endpoint,
    .iomap                    = of_fwnode_iomap,
    .irq_get                  = of_fwnode_irq_get,
    .add_links                = of_fwnode_add_links,
};
EXPORT_SYMBOL_GPL(of_fwnode_ops);
```

كل الـ functions دي thin wrappers تقريباً: بتستخدم `to_of_node(fwnode)` للتحويل وبتستدعي الـ OF function المقابلة.

---

#### `of_fwnode_property_read_int_array`

```c
static int of_fwnode_property_read_int_array(const struct fwnode_handle *fwnode,
                 const char *propname,
                 unsigned int elem_size, void *val, size_t nval);
```

الأذكى في الـ wrappers. لو `val == NULL` بيعد العناصر. لو `val != NULL` بيقرأ. بيـ dispatch بناءً على `elem_size`:

```c
switch (elem_size) {
case sizeof(u8):  return of_property_read_u8_array(...);
case sizeof(u16): return of_property_read_u16_array(...);
case sizeof(u32): return of_property_read_u32_array(...);
case sizeof(u64): return of_property_read_u64_array(...);
}
return -ENXIO;
```

---

#### `of_fwnode_get_reference_args`

```c
static int of_fwnode_get_reference_args(const struct fwnode_handle *fwnode,
             const char *prop, const char *nargs_prop,
             unsigned int nargs, unsigned int index,
             struct fwnode_reference_args *args);
```

بيحول الـ phandle parsing من `of_parse_phandle_with_args` / `of_parse_phandle_with_fixed_args` للـ generic `fwnode_reference_args` struct. لو `nargs_prop != NULL` يستخدم dynamic args count، لو لأ يستخدم `nargs` كـ fixed count.

- **Key detail**: بيملأ `args->args[]` بـ zeros للـ slots اللي بعد `args_count` — safe للـ callers اللي بيقرأوا fixed number من slots

---

#### `of_fwnode_graph_get_port_parent`

```c
static struct fwnode_handle *of_fwnode_graph_get_port_parent(
    struct fwnode_handle *fwnode);
```

مختلف قليلاً عن `of_graph_get_port_parent` — بتاخد port node مش endpoint node. بتتحقق من اسم parent: لو `ports` بترجع grandparent، لو لأ بترجع parent مباشرة.

---

#### `of_fwnode_device_get_dma_attr`

```c
static enum dev_dma_attr of_fwnode_device_get_dma_attr(
    const struct fwnode_handle *fwnode);
```

بتستخدم `of_dma_is_coherent` لتحديد إذا الـ device كان DMA coherent أو لأ وبترجع `DEV_DMA_COHERENT` أو `DEV_DMA_NON_COHERENT`. أما `of_fwnode_device_dma_supported` بترجع `true` دايماً لأن كل OF devices بتدعم DMA.

---

### Group 6: fw_devlink — Supplier/Consumer Dependency Tracking

الـ **fw_devlink** subsystem بيبني dependency graph بين devices قبل ما يـ probe أي device، عشان يضمن إن الـ suppliers بيتـ probe قبل الـ consumers. الـ `of_fwnode_add_links` هو entry point من الـ OF side.

#### Architecture Overview

```
of_fwnode_add_links(fwnode)
    └── for_each_property_of_node(con_np, p)
            └── of_link_property(con_np, p->name)
                    └── for each binding in of_supplier_bindings[]:
                            s->parse_prop(con_np, prop_name, i) → sup_np
                            of_link_to_phandle(con_np, sup_np, flags)
                                └── fwnode_link_add(consumer, supplier, flags)
```

---

#### `parse_prop_cells` و `DEFINE_SIMPLE_PROP`

```c
static struct device_node *parse_prop_cells(struct device_node *np,
                const char *prop_name, int index,
                const char *list_name, const char *cells_name);

#define DEFINE_SIMPLE_PROP(fname, name, cells)                    \
static struct device_node *parse_##fname(struct device_node *np,  \
                    const char *prop_name, int index)             \
{                                                                  \
    return parse_prop_cells(np, prop_name, index, name, cells);   \
}
```

بيتحقق إن `prop_name == list_name`، وبعدين بيـ parse الـ phandle في `index` باستخدام `__of_parse_phandle_with_args`.

الـ macro بيولد functions زي:
```c
// DEFINE_SIMPLE_PROP(clocks, "clocks", "#clock-cells")
// ينتج:
static struct device_node *parse_clocks(struct device_node *np,
                    const char *prop_name, int index) {
    return parse_prop_cells(np, prop_name, index, "clocks", "#clock-cells");
}
```

الـ properties المغطاة: `clocks`, `interconnects`, `iommus`, `mboxes`, `io-channels`, `io-backends`, `dmas`, `power-domains`, `hwlocks`, `extcon`, `nvmem-cells`, `phys`, `wakeup-parent`, `pwms`, `resets`, `leds`, `backlight`, `panel`, `msi-parent`, `post-init-providers`, `access-controllers`, `pses`, `power-supplies`, `mmc-pwrseq`.

---

#### `parse_suffix_prop_cells` و `DEFINE_SUFFIX_PROP`

```c
static struct device_node *parse_suffix_prop_cells(struct device_node *np,
                const char *prop_name, int index,
                const char *suffix, const char *cells_name);
```

بدل من مقارنة الاسم كامل، بيتحقق إن `prop_name` **بينتهي بـ** `suffix` باستخدام `strends()`. بيتم استخدامه لـ:
- `regulators` → suffix: `-supply`
- `gpio` → suffix: `-gpio`

مثال: `vcc-supply = <&reg_3v3>` هتـ match مع suffix `-supply`.

---

#### `parse_pinctrl_n`

```c
static struct device_node *parse_pinctrl_n(struct device_node *np,
                   const char *prop_name, int index);
```

بيتحقق إن الاسم يبدأ بـ `pinctrl-` وإن الحرف التالي رقم (0-9). بيتجاهل `pinctrl-names` وغيرها من properties غير الـ numbered.

---

#### `parse_gpios`

```c
static struct device_node *parse_gpios(struct device_node *np,
               const char *prop_name, int index);
```

بيـ exclude الـ properties اللي بتنتهي بـ `,nr-gpios` وبيـ parse الباقي اللي بينتهي بـ `-gpios`.

---

#### `parse_gpio_compat`

```c
static struct device_node *parse_gpio_compat(struct device_node *np,
                 const char *prop_name, int index);
```

legacy handler لـ `gpio` و `gpios` properties. بيتجاهل nodes اللي ليها `gpio-hog` property لأن الـ parent بيوفر الـ GPIOs ليها مش هي.

---

#### `parse_interrupts`

```c
static struct device_node *parse_interrupts(struct device_node *np,
                const char *prop_name, int index);
```

بيـ handle `interrupts` و `interrupts-extended`. مـ disabled على PPC (`IS_ENABLED(CONFIG_PPC)` = skip) بسبب quirks في الـ interrupt model هناك.

---

#### `parse_interrupt_map`

```c
static struct device_node *parse_interrupt_map(struct device_node *np,
               const char *prop_name, int index);
```

أعقد parser في الـ file. الـ `interrupt-map` property بيها structure معقدة:
```
interrupt-map = <child-addr(n cells) child-irq(m cells) parent(phandle) parent-irq(k cells)>, ...
```

بيقرأ `#interrupt-cells` و `#address-cells` عشان يحدد حجم كل entry، وبيمشي على الـ entries للوصول للـ index المطلوب.

**Pseudocode:**
```
addrcells = of_bus_n_addr_cells(np)
intcells = read "#interrupt-cells"
imap = get "interrupt-map" property

for i = 0; imap + addrcells + intcells + 1 < imap_end; i++:
    imap += addrcells + intcells    // skip child addr+irq
    imap = of_irq_parse_imap_parent(imap, ...)  // parse parent + its irq cells
    if i == index: return sup_args.np
    of_node_put(sup_args.np)
```

---

#### `parse_remote_endpoint`

```c
static struct device_node *parse_remote_endpoint(struct device_node *np,
                 const char *prop_name, int index);
```

يرجع NULL لو `index > 0` — الـ endpoint connection واحدة فقط. بيستخدم `of_graph_get_remote_port_parent` مش `of_graph_get_remote_endpoint` عشان الـ dependency هي على الـ device مش الـ endpoint.

---

#### `of_link_property`

```c
static int of_link_property(struct device_node *con_np, const char *prop_name);
```

بيمشي على `of_supplier_bindings[]` array، لكل binding بيجرب `parse_prop(con_np, prop_name, i)` بـ incrementing index. لو رجعت supplier node، بيستدعي `of_link_to_phandle`. بيكمل لو فشل link ولا يوقف — `of_link_property must create links to all available supplier nodes`.

- **Key detail**: الـ `optional` flag + `!fw_devlink_is_strict()` بيخلي optional suppliers يتـ skip لو مش في strict mode

---

#### `of_link_to_phandle`

```c
static void of_link_to_phandle(struct device_node *con_np,
                               struct device_node *sup_np, u8 flags);
```

قبل ما يضيف الـ link، بيتحقق إن الـ supplier وأجداده كلهم available أو عندهم device مرتبط بيهم. لو في node مش available في الـ chain، مبيضيفش الـ link — عشان ميكونش في pending dependency على disabled hardware.

---

#### `of_fwnode_add_links`

```c
static int of_fwnode_add_links(struct fwnode_handle *fwnode);
```

Entry point من الـ fw_devlink subsystem. مـ disabled على x86 تماماً. بيتكرر على كل properties وبيستدعي `of_link_property` لكل property.

- **Caller**: الـ fw_devlink framework بيستدعيها عند registration كل fwnode
- **Key detail**: `IS_ENABLED(CONFIG_X86)` guard — x86 بيستخدم ACPI مش DT لـ device dependencies

---

### Error Codes — ملخص

| Error | المعنى في سياق property reading |
|---|---|
| `-EINVAL` | الـ property مش موجودة |
| `-ENODATA` | الـ property موجودة بدون value |
| `-EOVERFLOW` | الـ property size أصغر من المطلوب (أو أكبر من max) |
| `-EILSEQ` | string مش null-terminated داخل الـ property data |
| `-ENXIO` | elem_size غير مدعوم في `of_fwnode_property_read_int_array` |

---

### Locking Notes

الـ file كله **لا يمسك locks مباشرة**. كل الـ locking بيتعمل في `of_find_property` و `of_get_child_by_name` وغيرها من base OF functions اللي بتستخدم `of_node_lock` (spinlock) أو `devtree_lock`. الـ property.c functions آمنة للاستخدام من أي context طالما الـ base functions آمنة.

الاستثناء: `of_link_to_phandle` و `fwnode_link_add` بتستخدم internal locking داخل الـ fwnode framework.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

**الـ** `drivers/of/property.c` بيشتغل جوه subsystem الـ Device Tree، ومفيش debugfs entries خاصة بيه مباشرةً، لكن الـ OF core بيوفر:

```
/sys/kernel/debug/of_reserved_mem      # reserved memory regions من الـ DT
/sys/kernel/debug/provideinfo          # fw_devlink supply chains
/sys/kernel/debug/device_component     # component dependencies
```

قراءة الـ fwnode links المبنية بواسطة `of_fwnode_add_links()`:

```bash
# عرض كل الـ fwnode links (fw_devlink)
cat /sys/kernel/debug/provideinfo 2>/dev/null || \
  ls /sys/kernel/debug/device_component/

# عرض الـ supplier-consumer relationships
find /sys/kernel/debug -name "*.dot" 2>/dev/null
```

#### 2. sysfs — المسارات المهمة

```
/sys/firmware/devicetree/base/          # الـ DTB live tree (كل node وproperty)
/sys/firmware/fdt                       # الـ raw DTB blob
/sys/bus/platform/devices/<dev>/        # device مع of_node pointer
/sys/bus/platform/devices/<dev>/of_node # symlink للـ DT node
/sys/bus/platform/devices/<dev>/driver  # هل الـ driver اتbind؟
```

```bash
# قراءة property مباشرة من sysfs
hexdump -C /sys/firmware/devicetree/base/<node>/<property>

# مثال: قراءة compatible من root
cat /sys/firmware/devicetree/base/compatible

# فحص كل properties لـ node معين
ls -la /sys/firmware/devicetree/base/soc/serial@ff000000/

# قراءة reg property بصيغة hex
hexdump -C /sys/firmware/devicetree/base/soc/serial@ff000000/reg
```

#### 3. ftrace — الـ Tracepoints والـ Events

الـ tracepoints المتاحة في OF subsystem:

```bash
# عرض كل OF events
ls /sys/kernel/tracing/events/of/

# تفعيل tracing للـ property reads
echo 1 > /sys/kernel/tracing/events/of/enable

# trace of_property_read_* calls بتفاصيل
echo 'p:of_prop_read drivers/of/property.c:of_find_property_value_of_size np=%di propname=%si' \
  > /sys/kernel/tracing/kprobe_events

echo 1 > /sys/kernel/tracing/events/kprobes/of_prop_read/enable
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
```

```bash
# trace graph_parse_endpoint و get_remote_endpoint
echo 'p:ep_parse of_graph_parse_endpoint node=%di endpoint=%si' \
  > /sys/kernel/tracing/kprobe_events

echo 'p:ep_remote of_graph_get_remote_endpoint node=%di' \
  >> /sys/kernel/tracing/kprobe_events

echo 1 > /sys/kernel/tracing/events/kprobes/ep_parse/enable
echo 1 > /sys/kernel/tracing/events/kprobes/ep_remote/enable
```

```bash
# تتبع of_fwnode_add_links وsupplier parsing
trace-cmd record -p function_graph \
  -g of_fwnode_add_links \
  -g of_link_property \
  -g parse_prop_cells \
  sleep 2
trace-cmd report | head -100
```

#### 4. printk / Dynamic Debug

**الـ** `pr_fmt` في الملف معرف بـ `"OF: "` — كل رسائل `pr_debug()` بتظهر بالـ prefix ده.

```bash
# تفعيل dynamic debug لكل drivers/of/
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وfunction names
echo 'file drivers/of/property.c +pflm' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل OF subsystem
echo 'module of +pflm' > /sys/kernel/debug/dynamic_debug/control

# تحقق التفعيل
grep 'of/property' /sys/kernel/debug/dynamic_debug/control
```

نتيجة متوقعة بعد التفعيل في `dmesg`:

```
[   2.341] OF: graph: no port node found in /soc/display@ff900000
[   2.342] OF: comparing panel with lcd-panel
[   2.343] OF: comparing panel with panel-simple
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الفائدة |
|---|---|
| `CONFIG_OF_DYNAMIC` | يفعّل ref counting + hot-reload للـ DT nodes |
| `CONFIG_OF_UNITTEST` | unit tests للـ OF property APIs — شغّل بـ `modprobe of_unittest` |
| `CONFIG_OF_OVERLAY` | دعم overlays + debugging لتغييرات runtime |
| `CONFIG_OF_KOBJ` | يكشف كل node وproperty في sysfs |
| `CONFIG_DEBUG_OF` | فحوصات إضافية على صحة الـ DT |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في `of_node_lock` |
| `CONFIG_OF_RESOLVE` | debugging لـ phandle resolution |
| `CONFIG_DYNAMIC_DEBUG` | لازم لتفعيل `pr_debug()` في property.c |
| `CONFIG_FTRACE` | أساسي للـ function tracing |
| `CONFIG_FWNODE_MDIO` | يساعد لو بتـdebug graph مع network |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_OF|CONFIG_DEBUG_OF|CONFIG_DYNAMIC_DEBUG'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# dtc — Device Tree Compiler: فحص وتحويل الـ DTB
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | less

# fdtdump — قراءة DTB مباشرة
fdtdump /sys/firmware/fdt | grep -A5 "remote-endpoint"

# fdtget — قراءة property محددة
fdtget /sys/firmware/fdt /soc/i2c@ff020000 clocks
fdtget -t u /sys/firmware/fdt /soc/serial@ff000000 reg

# fdtput — تعديل property في الـ blob (للـ testing فقط)
fdtput -t u /tmp/test.dtb /soc/uart0 reg 0xff000000 0x1000

# of-graph tool (لو متاح)
# اعرض الـ media graph من الـ DT
media-ctl --print-topology

# fw_devlink debug
cat /sys/kernel/debug/devices_deferred   # devices متوقفة على supplier
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `OF: Read of boolean property 'X' with a value.` | property bool بس فيها value، مش standard | راجع الـ DTS — Boolean props لازم تكون فاضية `prop;` |
| `OF: size of X in node Y is not a multiple of Z` | حجم الـ property مش متطابق مع element size | الـ DTS array size غلط، تحقق بـ `fdtget -t u` |
| `OF: graph: no port node found in X` | node مفيهاش `port` child | الـ DTS ناقصة `port@0 { }` جوه الـ device node |
| `endpoint X has no parent node` | endpoint بدون port parent | الـ DT hierarchy غلط — endpoint لازم يكون جوه port |
| `non endpoint node is used (X)` | `of_graph_get_next_port_endpoint` لقت child مش اسمه endpoint | الـ DTS فيه node اسمه غلط جوه الـ port |
| `no valid endpoint (P, E) for node X` | `of_graph_get_endpoint_by_regs` مش لاقي reg match | الـ `reg` property في الـ port/endpoint غلط |
| `no valid remote node` | `remote-endpoint` phandle مكسور | الـ phandle مش بيشير لـ node موجودة |
| `not available for remote node` | الـ remote device مـdisable | الـ remote node فيه `status = "disabled"` |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

أماكن منطقية لإضافة فحوصات لو بتـdebug مشكلة معقدة:

```c
/* في of_graph_parse_endpoint() — تحقق من وجود port_node */
WARN_ON(!port_node);   /* موجود بالفعل في الكود */

/* في of_graph_get_next_port_endpoint() — تحقق من endpoint name */
WARN(!of_node_name_eq(prev, "endpoint"),
     "non endpoint node is used (%pOF)", prev);  /* موجود */

/* إضافة مقترحة عند debugging: في of_find_property_value_of_size */
/* لو property موجودة بس length = 0 وده unexpected */
if (prop->length == 0 && min > 0)
    dump_stack();  /* تتبع من طلب property فاضية */

/* في of_link_property() — لو parse_prop رجعت phandle لـ node unavailable */
WARN_ON_ONCE(phandle && !of_device_is_available(phandle));

/* في of_graph_get_remote_node() — لو remote مش available */
/* pr_debug موجود — يمكن ترقيته لـ pr_warn عند debugging */
```

---

### Hardware Level

#### 1. التحقق أن حالة الـ Hardware تطابق الـ Kernel State

الـ `of/property.c` بيقرأ من الـ DTB المُحمَّل في الـ RAM — الـ hardware state اللي محتاج تتحقق منه هو:

```bash
# 1. هل الـ DTB الـ kernel شغال بيه هو نفسه اللي على disk؟
md5sum /sys/firmware/fdt /boot/dtb-$(uname -r)/*.dtb 2>/dev/null

# 2. فين بدأ الـ DTB في الـ RAM؟
dmesg | grep -E 'OF: fdt|device tree|DTB|memory@'
# مثال ناتج: [0.000000] OF: fdt: Ignoring memory range 0x80000000 - 0x80200000

# 3. تحقق الـ chosen node — يكشف ما أرسله الـ bootloader
fdtget /sys/firmware/fdt /chosen bootargs
fdtget /sys/firmware/fdt /chosen linux,initrd-start 2>/dev/null

# 4. تحقق memory layout يطابق الـ reg properties
cat /proc/iomem | head -30
fdtget -t u /sys/firmware/fdt /memory@80000000 reg
```

#### 2. Register Dump Techniques

لـ debugging الـ MMIO registers اللي عناوينها مأخوذة من `reg` property:

```bash
# استخراج عنوان الـ register من الـ DT
REG=$(fdtget -t u /sys/firmware/fdt /soc/serial@ff000000 reg | awk '{print $1}')
SIZE=$(fdtget -t u /sys/firmware/fdt /soc/serial@ff000000 reg | awk '{print $2}')

echo "Base: 0x$(printf '%x' $REG), Size: 0x$(printf '%x' $SIZE)"

# قراءة الـ registers بـ devmem2
devmem2 0xff000000 w     # قراءة 32-bit word
devmem2 0xff000004 w
devmem2 0xff000008 w

# أو بـ /dev/mem (لو enabled)
dd if=/dev/mem bs=4 count=16 skip=$((0xff000000 / 4)) 2>/dev/null | hexdump -C

# io utility (من package i2c-tools على بعض distros)
io -4 -r 0xff000000
io -4 -r 0xff000004
```

```bash
# dump كامل لـ register range
BASE=0xff000000
SIZE=256
for i in $(seq 0 4 $((SIZE-4))); do
    ADDR=$((BASE + i))
    printf "0x%08x: " $ADDR
    devmem2 $ADDR w 2>/dev/null | grep 'Read at'
done
```

#### 3. Logic Analyzer / Oscilloscope

لـ debugging الـ graph endpoints (كاميرات، شاشات، I2S، SPI):

- **CSI/DSI camera interfaces**: استخدم LA على data lanes — تحقق `clock-lanes` و`data-lanes` في الـ DTS تطابق الـ hardware routing
- **I2C الـ phandles** (`clocks`, `resets`): قِس الـ clock frequency بالـ oscilloscope على الـ SCL line — قارن بـ `clock-frequency` property
- **GPIO properties** (`-gpios`, `-gpio` suffix): استخدم LA لمراقبة الـ pin — تأكد الـ polarity (active-low/active-high) في الـ DTS صح
- **PWM**: قِس الـ duty cycle وقارن بـ `pwm-names` و`pwms` property values

```bash
# تحقق clock-lanes في DTS يطابق hardware
fdtget /sys/firmware/fdt /soc/mipi_csi2@ff910000 clock-lanes
fdtget /sys/firmware/fdt /soc/mipi_csi2@ff910000 data-lanes
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log |
|---|---|
| الـ bootloader بعت DTB قديم | `OF: fdt: memory size ...` مش بيطابق الـ RAM الفعلي |
| `reg` property بتشير لـ address غلط | `ioremap failed for ...` أو `can't map registers` |
| Clock source مش متصل فعلاً | `clk_get failed` مع `of_clk_get_from_provider` |
| Remote endpoint hardware مش موجود | `no valid remote node` + device misbehavior |
| GPIO line متقلوبة (polarity معكوسة) | Device يشتغل عكسياً — فحص `GPIO_ACTIVE_LOW` flag في DTS |
| Power domain sequence غلط | `failed to enable power-domains` at probe |
| I2C address غلط في DTS | `i2c i2c-X: ... No such device` |

#### 5. Device Tree Debugging — التحقق أن DT يطابق الـ Hardware

```bash
# 1. عرض الـ DT كـ DTS بشري
dtc -I dtb -O dts -o /tmp/current.dts /sys/firmware/fdt 2>/dev/null
less /tmp/current.dts

# 2. مقارنة الـ DTS المنشور بالـ DT الفعلي المحمل
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/running.dts
diff /boot/my-board.dts /tmp/running.dts

# 3. فحص graph كامل (ports وendpoints)
grep -A20 "port@" /tmp/current.dts | head -80

# 4. فحص phandle consistency — كل remote-endpoint يشير لـ endpoint موجود؟
grep "remote-endpoint" /tmp/current.dts
# تحقق يدوياً أن الـ phandle references موجودة

# 5. فحص الـ overlays المطبقة
ls /sys/firmware/devicetree/overlays/ 2>/dev/null

# 6. تحقق الـ status property لكل device
fdtget /sys/firmware/fdt /soc/i2c@ff020000 status
# يجب أن يرجع: okay

# 7. اكتشاف الـ nodes التي تمت قراءتها بواسطة الـ kernel
dmesg | grep "^OF:" | sort -u
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**فحص property موجودة ولها القيمة الصح:**

```bash
# هل property موجودة؟
fdtget /sys/firmware/fdt /soc/uart@ff000000 status
# ناتج متوقع: okay

# قراءة u32 array (مثل reg)
fdtget -t u /sys/firmware/fdt /soc/uart@ff000000 reg
# ناتج: 4278190080 65536
# أي: base=0xFF000000 size=0x10000

# قراءة string list (مثل clock-names)
fdtget -t s /sys/firmware/fdt /soc/uart@ff000000 clock-names
# ناتج: uart pclk

# عدد clocks
fdtget -t u /sys/firmware/fdt /soc/uart@ff000000 clocks | wc -w
```

**تتبع الـ graph topology:**

```bash
# عرض كل endpoints في الـ system
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | \
  grep -E "(remote-endpoint|endpoint@|port@)" | head -30

# فحص remote-endpoint phandle يشير لـ node موجود
fdtget /sys/firmware/fdt /soc/camera@ff920000/port@0/endpoint@0 remote-endpoint
# ناتج: <phandle_number>

# البحث عن الـ node اللي phandle بتاعه X
PHANDLE=14   # من الأمر السابق
grep -r "phandle = <$PHANDLE>" /sys/firmware/devicetree/ 2>/dev/null || \
  dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep "phandle = <$PHANDLE>"
```

**فحص fw_devlink dependencies:**

```bash
# عرض الـ devices اللي لسه مش بدأت بسبب missing suppliers
cat /sys/kernel/debug/devices_deferred 2>/dev/null
# ناتج مثال:
# platform:ff910000.mipi-csi2 (supplier: platform:ff9f0000.clock-controller)

# فحص device link معين
ls /sys/bus/platform/devices/ff910000.mipi-csi2/consumer\:*/
```

**dynamic debug مع filter:**

```bash
# تفعيل debug للـ graph functions فقط
echo 'func of_graph* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func of_fwnode* +p' >> /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ output
dmesg -w | grep "^OF:"
```

**تشغيل OF unit tests:**

```bash
# لو CONFIG_OF_UNITTEST=m
modprobe of_unittest
dmesg | tail -30
# ناتج ناجح: "OF: OF unit test ok"
# ناتج فاشل: "OF: ERROR: unittest X failed"
```

**تتبع of_fwnode_add_links كاملاً:**

```bash
# trace-cmd مع function filter
trace-cmd record -p function \
  --func-stack \
  -l 'of_fwnode_add_links' \
  -l 'of_link_property' \
  -l 'parse_prop_cells' \
  -l 'of_link_to_phandle' \
  sleep 1

trace-cmd report 2>/dev/null | grep -E '(of_link|parse_prop|of_fwnode)' | head -50
```

**تفسير الـ output:**

```
# مثال ناتج trace-cmd:
#    kworker/0:2-34    [000]  2.341: of_fwnode_add_links: (of_fwnode_add_links)
#    kworker/0:2-34    [000]  2.341:  of_link_property: (of_link_property+0x0/0x80)
#    kworker/0:2-34    [000]  2.341:   parse_clocks: (parse_prop_cells+0x0/0x60)
#    kworker/0:2-34    [000]  2.342:   of_link_to_phandle: link added con=camera sup=clk-ctrl
#
# التفسير:
# - of_fwnode_add_links بتمشي على كل properties في الـ camera node
# - parse_clocks بتلاقي "clocks" property وبتعمل parse لها
# - of_link_to_phandle بتربط camera (consumer) بـ clk-ctrl (supplier)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Gateway صناعي على RK3562 — I2C sensor مش شايله الـ kernel

#### العنوان
**فشل probe لـ I2C temperature sensor في gateway صناعي بسبب missing `clocks` phandle**

#### السياق
شركة تعمل على industrial gateway بيشتغل على **RK3562**. الـ board بتجمع بيانات من حساسات حرارة عبر **I2C**. الـ driver الخاص بالحساس مكتوب صح وبيشتغل على boards تانية، لكن على هذا الـ board الجديد الـ sensor مش بيظهر خالص في `/sys/bus/i2c/devices`.

#### المشكلة
الـ kernel بيطلع log فيه:
```
[ 3.142] i2c i2c-2: Failed to get clock
[ 3.143] i2c 2-0048: probe failed: -EPROBE_DEFER
```
الـ device بيفضل في حالة `EPROBE_DEFER` بلا نهاية.

#### التحليل
الـ `of_fwnode_add_links()` في `property.c` بتتنادى وقت تسجيل الـ device في الـ OF graph:

```c
static int of_fwnode_add_links(struct fwnode_handle *fwnode)
{
    const struct property *p;
    struct device_node *con_np = to_of_node(fwnode);

    /* تمشي على كل property في الـ node */
    for_each_property_of_node(con_np, p)
        of_link_property(con_np, p->name);  /* تحاول تعمل link لكل supplier */

    return 0;
}
```

الـ `of_link_property()` بتمشي على `of_supplier_bindings[]` وتلاقي إن `clocks` property موجودة في الـ I2C controller node:

```c
DEFINE_SIMPLE_PROP(clocks, "clocks", "#clock-cells")
```

الـ macro ده بيعمل `parse_clocks()` اللي بتعمل:
```c
return parse_prop_cells(np, prop_name, index, "clocks", "#clock-cells");
```

الـ `parse_prop_cells()` بتستدعي `__of_parse_phandle_with_args()` — لو الـ phandle الـ `clocks` بيشاور على clock node مش `available` أو مش موجود في الـ DT، الـ link مش بتتعمل، والـ `fw_devlink` بيحط الـ device في defer.

بالـ DTS الخاص بالـ board الجديد، نسيوا يضيفوا entry لـ `assigned-clocks` في الـ I2C controller:

```dts
/* DTS خاطئ */
&i2c2 {
    status = "okay";
    /* missing: clocks = <&cru CLK_I2C2>; */
    sensor@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};
```

#### الحل
**الخطوة 1 — تأكد من الـ DT باستخدام debugfs:**
```bash
cat /sys/kernel/debug/of/base/i2c@fe5b0000/properties
# أو
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "i2c@fe5b0000"
```

**الخطوة 2 — افحص الـ fwnode links:**
```bash
cat /sys/kernel/debug/fw_devlink/links
# ابحث عن الـ consumer المعلق
```

**الخطوة 3 — صلّح الـ DTS:**
```dts
/* DTS صح */
&i2c2 {
    status = "okay";
    clocks = <&cru CLK_I2C2>;          /* supplier link يتعمل صح */
    clock-names = "i2c";
    sensor@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};
```

#### الدرس المستفاد
الـ `of_fwnode_add_links()` بتعمل **supplier links تلقائياً** لكل property معروفة زي `clocks`, `resets`, `power-domains`. لو أي supplier node مش available في الـ DT، الـ consumer بيفضل في `EPROBE_DEFER`. دايماً افحص الـ DT properties كاملة مش بس الـ driver code.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش بيشتغل بسبب graph endpoint خاطئ

#### العنوان
**فشل HDMI output على Allwinner H616 بسبب خطأ في `remote-endpoint` phandle في الـ OF graph**

#### السياق
منتج Android TV box بيستخدم **Allwinner H616**. الـ SoC بيشتغل والـ display pipeline مكون من: **DE2 (Display Engine 2) → TCON0 → HDMI**. الشاشة مش بتشتغل خالص، والـ HDMI connector مش بيديه signal.

#### المشكلة
```
[    5.231] sun8i-de2 1000000.display-engine: failed to find remote endpoint
[    5.232] sun8i-mixer 1100000.mixer: no remote endpoint
```

#### التحليل
الـ display driver بيستخدم `of_graph_get_remote_port_parent()` اللي في `property.c`:

```c
struct device_node *of_graph_get_remote_port_parent(
                       const struct device_node *node)
{
    /* الخطوة 1: اجيب الـ remote endpoint node */
    struct device_node *np __free(device_node) =
        of_graph_get_remote_endpoint(node);

    /* الخطوة 2: اجيب parent الـ port = الـ device الفعلي */
    return of_graph_get_port_parent(np);
}
```

و `of_graph_get_remote_endpoint()`:
```c
struct device_node *of_graph_get_remote_endpoint(const struct device_node *node)
{
    /* بتتبع الـ remote-endpoint phandle ببساطة */
    return of_parse_phandle(node, "remote-endpoint", 0);
}
```

`of_graph_get_port_parent()` بتمشي للـ parent بعدد معين من الـ levels:
```c
/* Walk 3 levels up only if there is 'ports' node. */
for (depth = 3; depth && node; depth--) {
    node = of_get_next_parent(node);
    if (depth == 2 && !of_node_name_eq(node, "ports") &&
        !of_node_name_eq(node, "in-ports") &&
        !of_node_name_eq(node, "out-ports"))
        break;   /* يوقف عند level 2 لو مفيش ports node */
}
```

الـ DTS الخاطئ عنده الـ `remote-endpoint` بيشاور على endpoint غلط:

```dts
/* DTS خاطئ — الـ phandle مش متطابق */
&tcon0 {
    ports {
        tcon0_out: port@1 {
            tcon0_out_hdmi: endpoint@1 {
                /* غلط: بيشاور على port بدل endpoint */
                remote-endpoint = <&hdmi_in>;  /* hdmi_in هو port مش endpoint */
            };
        };
    };
};

&hdmi {
    hdmi_in: port@0 {              /* ده port مش endpoint! */
        hdmi_in_tcon0: endpoint@0 {
            remote-endpoint = <&tcon0_out_hdmi>;
        };
    };
};
```

#### الحل
**Debug باستخدام `of_graph` functions مباشرة:**
```bash
# افحص الـ graph structure في debugfs
find /sys/kernel/debug/of -name "endpoint*" | xargs grep -l "remote-endpoint"

# استخدم ftrace لتتبع of_graph_get_remote_endpoint
echo 'of_graph_get_remote_endpoint' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
```

**الـ DTS الصح:**
```dts
&tcon0 {
    ports {
        tcon0_out: port@1 {
            #address-cells = <1>;
            #size-cells = <0>;
            tcon0_out_hdmi: endpoint@1 {
                reg = <1>;
                /* صح: بيشاور على الـ endpoint مش الـ port */
                remote-endpoint = <&hdmi_in_tcon0>;
            };
        };
    };
};

&hdmi {
    ports {
        hdmi_in: port@0 {
            #address-cells = <1>;
            #size-cells = <0>;
            hdmi_in_tcon0: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&tcon0_out_hdmi>;
            };
        };
    };
};
```

`of_graph_parse_endpoint()` بتقرأ الـ `reg` من كل port وendpoint:
```c
of_property_read_u32(port_node, "reg", &endpoint->port);
of_property_read_u32(node, "reg", &endpoint->id);
```
لو الـ `reg` ناقص، الـ port ID بيبقى 0 وبيحصل تعارض.

#### الدرس المستفاد
الـ OF graph في `property.c` بيعتمد على hierarchy صارمة: **device → ports → port@N → endpoint@M**. الـ `of_graph_get_port_parent()` بتعد levels للأعلى وبتفرق بين وجود `ports` node وعدمه. أي خطأ في الـ hierarchy أو في الـ phandle بيكسر الـ graph كله بصمت.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — SPI device مش بيشتغل بسبب `reg` size خاطئ

#### العنوان
**خطأ `-EOVERFLOW` في قراءة `reg` property لـ SPI flash على STM32MP1**

#### السياق
board IoT صغير على **STM32MP1** بيستخدم **SPI NOR flash** (W25Q64) متوصل على **SPI1**. الـ flash driver بيشتغل على نفس الكود على board تانية، لكن هنا الـ probe بيفشل فوراً.

#### المشكلة
```
[    2.871] spi_nor spi0.0: Failed to get 'reg' property
[    2.872] spi-nor: probe of spi0.0 failed with error -EOVERFLOW
```

#### التحليل
الـ SPI NOR driver بيستخدم `of_property_read_u32()` لقراءة الـ chip select:

```c
/* في drivers/spi/spi.c */
of_property_read_u32(nc, "reg", &value);
```

الـ `of_property_read_u32()` بتستدعي `of_find_property_value_of_size()`:

```c
static void *of_find_property_value_of_size(const struct device_node *np,
            const char *propname, u32 min, u32 max, size_t *len)
{
    const struct property *prop = of_find_property(np, propname, NULL);

    if (!prop)
        return ERR_PTR(-EINVAL);
    if (!prop->value)
        return ERR_PTR(-ENODATA);
    if (prop->length < min)
        return ERR_PTR(-EOVERFLOW);   /* ← هنا بيحصل الخطأ */
    if (max && prop->length > max)
        return ERR_PTR(-EOVERFLOW);
    ...
}
```

المشكلة في الـ DTS: كتبوا `reg` بـ `/bits/ 16` بدل الـ default 32-bit:

```dts
/* DTS خاطئ */
&spi1 {
    status = "okay";
    flash@0 {
        compatible = "winbond,w25q64", "jedec,spi-nor";
        reg = /bits/ 16 <0>;   /* خطأ: 2 bytes بس، المتوقع 4 bytes */
        spi-max-frequency = <50000000>;
    };
};
```

`prop->length` = 2 bytes، لكن `min` = `sizeof(u32)` = 4 bytes، فـ `prop->length < min` = true → ترجع `-EOVERFLOW`.

**للتأكد:**
```bash
# افحص الـ property length فعلياً
hexdump /sys/firmware/devicetree/base/soc/spi@5c000000/flash@0/reg
# لو طلع 2 bytes بس، المشكلة واضحة
```

**أو باستخدام `of_property_count_elems_of_size()`:**
```c
int count = of_property_count_elems_of_size(np, "reg", sizeof(u32));
/* لو رجع -EINVAL يبقى size مش multiple of 4 */
```

#### الحل
```dts
/* DTS صح */
&spi1 {
    status = "okay";
    cs-gpios = <&gpioa 15 GPIO_ACTIVE_LOW>;
    flash@0 {
        compatible = "winbond,w25q64", "jedec,spi-nor";
        reg = <0>;   /* 32-bit كما يتوقع الـ framework */
        spi-max-frequency = <50000000>;
        spi-rx-bus-width = <1>;
        spi-tx-bus-width = <1>;
    };
};
```

#### الدرس المستفاد
الـ `of_find_property_value_of_size()` صارمة جداً في التحقق من الـ size. استخدام `/bits/ 8` أو `/bits/ 16` بشكل خاطئ بيطلع `-EOVERFLOW` مش `-EINVAL`، وده بيضلل المبرمج. دايماً استخدم `hexdump` أو `dtc -I fs` لتأكيد size الـ property في الـ compiled DTB.

---

### السيناريو 4: Automotive ECU على i.MX8 — فشل قراءة string array لتهيئة CAN interfaces

#### العنوان
**فشل تهيئة CAN bus names على i.MX8 بسبب `clock-names` string غير مُنهية بـ null**

#### السياق
ECU في سيارة بيشتغل على **i.MX8QM**. الـ ECU فيه **3 CAN interfaces** (FlexCAN). الـ driver بيحتاج يقرأ `clock-names` property عشان يعمل match بين الـ clocks والـ names. في بعض الأحيان الـ boot بيتم بنجاح، وأحياناً الـ driver بيطلع error.

#### المشكلة
```
[ 1.442] flexcan 5a8d0000.flexcan: error reading clock-names: -EILSEQ
[ 1.443] flexcan: probe of 5a8d0000.flexcan failed with error -22
```

الـ error متقطع — بيحصل في بعض الـ builds ومش غيرها.

#### التحليل
الـ driver بيستخدم `of_property_read_string_array()` أو `of_property_match_string()`:

```c
int of_property_match_string(const struct device_node *np, const char *propname,
                             const char *string)
{
    const struct property *prop = of_find_property(np, propname, NULL);
    size_t l;
    int i;
    const char *p, *end;

    if (!prop)
        return -EINVAL;
    if (!prop->value)
        return -ENODATA;

    p = prop->value;
    end = p + prop->length;

    for (i = 0; p < end; i++, p += l) {
        l = strnlen(p, end - p) + 1;
        if (p + l > end)
            return -EILSEQ;   /* ← الـ string مش مُنهية بـ null قبل نهاية الـ buffer */
        ...
    }
    return -ENODATA;
}
```

المشكلة: الـ DTB كان بيتـ compile بأداة قديمة (dtc version قديمة) كانت في بعض الحالات بتنسى تضيف null terminator للـ string الأخيرة في الـ string list. الـ `prop->length` = عدد bytes بدون null، فـ `strnlen(p, end-p)` بترجع value وبعدها `p + l > end` = true.

**اكتشاف المشكلة:**
```bash
# افحص الـ raw bytes للـ property
hexdump -C /sys/firmware/devicetree/base/soc/flexcan@5a8d0000/clock-names
# لو الـ output مش منتهي بـ 00، الـ null ناقص
```

**أو باستخدام python:**
```bash
python3 -c "
data = open('/sys/firmware/devicetree/base/soc/flexcan@5a8d0000/clock-names','rb').read()
print(repr(data))
print('Last byte:', hex(data[-1]))
"
```

`of_property_read_string()` أيضاً بتكتشف نفس المشكلة:
```c
if (strnlen(prop->value, prop->length) >= prop->length)
    return -EILSEQ;   /* مفيش null terminator */
```

#### الحل
**الخطوة 1 — تحديث dtc:**
```bash
# تأكد من إصدار dtc
dtc --version
# يجب أن يكون >= 1.6.0

# إعادة compile الـ DTS
make dtbs DTC=/path/to/new/dtc
```

**الخطوة 2 — التأكد في الـ DTS:**
```dts
/* DTS صح — dtc بيضيف null تلقائياً */
&flexcan2 {
    clocks = <&clk IMX8QM_CAN2_IPG_CLK>,
             <&clk IMX8QM_CAN2_CLK>;
    clock-names = "ipg", "per";    /* dtc بيضيف \0 بعد كل string */
    status = "okay";
};
```

**الخطوة 3 — في الـ driver، handle الـ -EILSEQ صح:**
```c
ret = of_property_match_string(np, "clock-names", "per");
if (ret == -EILSEQ) {
    dev_err(dev, "Corrupted clock-names in DT, check dtc version\n");
    return ret;
}
```

#### الدرس المستفاد
الـ `of_property_match_string()` و`of_property_read_string()` بيتحققوا صراحةً من الـ null terminator. الـ `-EILSEQ` مش دايماً بيعني bug في الـ driver — ممكن يكون corruption في الـ DTB نفسه. في automotive حيث الـ build pipeline معقد ومتوزع، دايماً اعمل validation على الـ DTB نفسه.

---

### السيناريو 5: Custom Board Bring-Up على AM62x — power domains مش بتتـ enable بسبب `fw_devlink` circular dependency

#### العنوان
**deadlock في device probe على AM62x بسبب circular `power-domains` link في الـ OF graph**

#### السياق
فريق bring-up لـ custom board على **Texas Instruments AM62x** (industrial IoT). الـ board فيه custom power management IC متوصل بـ **I2C**. الـ developer أضاف `power-domains` في كل device node عشان يتحكم في الـ power. الـ boot بيتعلق لدقايق طويلة ثم بيكمل ببطء شديد.

#### المشكلة
```
[   10.001] INFO: task kworker:0:0 blocked for more than 122 seconds.
[   10.002] Possible circular dependency detected:
[   10.003] i2c-pmic -> power-domain -> i2c-pmic
```

#### التحليل
المشكلة بتبدأ في `of_fwnode_add_links()`:

```c
static int of_fwnode_add_links(struct fwnode_handle *fwnode)
{
    const struct property *p;
    struct device_node *con_np = to_of_node(fwnode);

    for_each_property_of_node(con_np, p)
        of_link_property(con_np, p->name);  /* بتمشي على كل property */

    return 0;
}
```

`of_link_property()` بتلاقي `power-domains` property وتستدعي `parse_power_domains()`:

```c
DEFINE_SIMPLE_PROP(power_domains, "power-domains", "#power-domain-cells")
```

اللي بتستدعي `parse_prop_cells()` → `__of_parse_phandle_with_args()` عشان تجيب الـ supplier node (الـ PMIC power domain controller).

في الـ DTS، الـ PMIC نفسه موجود على I2C bus، والـ I2C controller محتاج `power-domains` من الـ PMIC:

```dts
/* DTS بيخلق circular dependency */
&i2c0 {
    status = "okay";
    power-domains = <&pmic_pd 0>;  /* I2C محتاج الـ PMIC */

    pmic: pmic@48 {
        compatible = "custom,pmic";
        reg = <0x48>;
        pmic_pd: power-controller {
            #power-domain-cells = <1>;
            /* PMIC power controller */
        };
    };
};
```

الـ `fw_devlink` بيشوف:
- الـ `i2c0` محتاج `pmic_pd` (supplier)
- الـ `pmic` محتاج `i2c0` عشان يتـ probe (consumer)

فبيحصل circular dependency.

`of_link_to_phandle()` بتضيف الـ link:
```c
static void of_link_to_phandle(struct device_node *con_np,
                              struct device_node *sup_np,
                              u8 flags)
{
    struct device_node *tmp_np __free(device_node) = of_node_get(sup_np);

    /* تتحقق إن الـ supplier وأجداده available */
    while (tmp_np) {
        if (of_fwnode_handle(tmp_np)->dev)
            break;
        if (!of_device_is_available(tmp_np))
            return;
        tmp_np = of_get_next_parent(tmp_np);
    }

    fwnode_link_add(of_fwnode_handle(con_np), of_fwnode_handle(sup_np), flags);
}
```

الـ link بيتضاف بنجاح في الاتجاهين فبيحصل الـ deadlock.

**تشخيص الـ circular dependency:**
```bash
# افحص الـ fw_devlink state
cat /sys/kernel/debug/fw_devlink/links | grep -E "i2c|pmic"

# شوف الـ supplier/consumer relationships
ls /sys/bus/platform/devices/i2c0/
cat /sys/bus/platform/devices/i2c0/supplier:*/link_type 2>/dev/null

# استخدم boot parameter لـ verbose fw_devlink
# في الـ bootargs:
fw_devlink=permissive
```

#### الحل
**الحل 1 — كسر الـ circular dependency في الـ DTS:**

استخدم `power-domains` في الـ PMIC device node فقط، مش في الـ I2C controller نفسه. بدلاً من كده، استخدم `assigned-clock-parents` أو `operating-points-v2` للـ power management:

```dts
/* DTS صح — مفيش circular dependency */
&i2c0 {
    status = "okay";
    /* أزلنا power-domains من هنا */

    pmic: pmic@48 {
        compatible = "custom,pmic";
        reg = <0x48>;
        /* الـ PMIC نفسه مش محتاج power domain */
        pmic_pd: power-controller {
            #power-domain-cells = <1>;
        };
    };
};

/* devices تانية تستخدم الـ PMIC power domain */
&usb0 {
    power-domains = <&pmic_pd 1>;
};
```

**الحل 2 — استخدام `post-init-providers` لو الـ circular dependency لازمة:**

```dts
&i2c0 {
    /* بدل power-domains */
    post-init-providers = <&pmic_pd>;
};
```

الـ `parse_post_init_providers` في `of_supplier_bindings[]` بيستخدم `FWLINK_FLAG_IGNORE`:
```c
{
    .parse_prop = parse_post_init_providers,
    .fwlink_flags = FWLINK_FLAG_IGNORE,   /* مش بيعمل blocking dependency */
},
```

**الحل 3 — boot parameter مؤقت للـ debug:**
```bash
# في kernel cmdline
fw_devlink=off
# ده بيعطل fw_devlink كلياً — للـ debug فقط
```

#### الدرس المستفاد
الـ `of_fwnode_add_links()` مع `of_supplier_bindings[]` بتبني **شجرة dependencies** تلقائية من الـ DT properties. الـ `power-domains`, `clocks`, `resets` كلهم بيخلقوا blocking links عبر `fwnode_link_add()`. في بورد bring-up، ابدأ بـ `fw_devlink=permissive` عشان تشوف الـ devices بتشتغل الأول، وبعدين صلّح الـ dependencies واحدة واحدة. الـ circular dependencies خصوصاً بين الـ buses والـ power controllers هي أكتر مشكلة شائعة في AM62x و i.MX8.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

| المقال | الوصف |
|--------|-------|
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | شرح كيف بتتعامل platform devices مع device tree وازاي بيتعمل probe من DT |
| [Device tree troubles](https://lwn.net/Articles/560523/) | تحليل المشاكل العملية اللي اتواجهت في تطبيق device tree على Linux |
| [Basic ARM device tree support](https://lwn.net/Articles/423607/) | أول دعم للـ ARM في الـ device tree وكيف اتبنت البنية التحتية |
| [ELCE: Grant Likely on device trees](https://lwn.net/Articles/414016/) | عرض Grant Likely (صاحب كود الـ OF) عن فلسفة الـ device tree |
| [Device trees I: Are we having fun yet?](https://lwn.net/Articles/572692/) | نقاش عميق حول مشاكل الـ DT bindings وال `compatible` property |
| [Unified fwnode endpoint parser](https://lwn.net/Articles/737780/) | إضافة unified parser للـ fwnode endpoints بما فيها `of_graph` |
| [ACPI graph support](https://lwn.net/Articles/718184/) | كيف اتعمل دعم للـ graph topology في ACPI على غرار `of_graph` |
| [Device dependencies and deferred probing](https://lwn.net/Articles/662820/) | علاقة الـ `of_graph` بـ deferred probing لما device مش جاهز |
| [KS2009: Generic device trees](https://lwn.net/Articles/357487/) | الجلسة التاريخية اللي رسمت مستقبل الـ device tree في Linux |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في `Documentation/` داخل مصدر الـ kernel:

```
Documentation/devicetree/usage-model.rst
    شرح نموذج الاستخدام للـ device tree في Linux

Documentation/devicetree/bindings/
    كل الـ bindings الرسمية لكل subsystem (YAML schema)

Documentation/devicetree/of_unittest.rst
    شرح unit tests للـ OF subsystem

Documentation/driver-api/device_link.rst
    ربط الـ device dependencies بالـ graph

Documentation/firmware-guide/acpi/dsd/graph.rst
    الـ graph model في ACPI — مشابه تمامًا لـ of_graph
```

**الـ kernel API الرسمي:**

- [DeviceTree Kernel API — kernel.org](https://docs.kernel.org/devicetree/kernel-api.html)
- [Linux and the Devicetree — static.lwn.net](https://static.lwn.net/kerneldoc/devicetree/usage-model.html)
- [Open Firmware and Devicetree index](https://static.lwn.net/kerneldoc/devicetree/index.html)
- [Graphs — firmware-guide/acpi/dsd/graph](https://www.kernel.org/doc/html/latest/firmware-guide/acpi/dsd/graph.html)

---

### Commits مهمة في تاريخ الملف

الـ commits دي مهمة لفهم تطور `drivers/of/property.c`:

| الوصف | المصدر |
|-------|--------|
| نقل كود property من `base.c` لـ `property.c` | `git log --follow drivers/of/property.c` |
| إضافة `of_graph_*` functions | بدأت مع دعم V4L2 media graph |
| إضافة `of_property_read_string_array` | لدعم قراءة arrays من strings |
| `fwnode` abstraction فوق `of_node` | وحّدت الكود مع ACPI |

للبحث في تاريخ الـ commits:

```bash
# تاريخ الملف كله
git log --oneline drivers/of/property.c

# أول ما اتضاف of_graph
git log --oneline --all -S "of_graph_get_next_endpoint" -- drivers/of/

# مين عمل refactor من base.c
git log --oneline --diff-filter=A -- drivers/of/property.c
```

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [spinics.net — stop parsing remote-endpoint](https://www.spinics.net/lists/devicetree/msg468625.html) | نقاش عن إيقاف parse معين في of_graph properties |
| [lkml.org](https://lkml.org/) | أرشيف LKML الكامل — ابحث بـ `of_property` أو `of_graph` |
| [lore.kernel.org/devicetree](https://lore.kernel.org/devicetree/) | أرشيف مخصص لـ devicetree mailing list |

---

### مواقع eLinux.org

| الصفحة | الوصف |
|--------|-------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لكل properties الـ standard |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | شرح المشاكل الشائعة والحلول العملية |
| [Linux Drivers Device Tree Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) | دليل كتابة drivers باستخدام device tree |
| [Device Tree frowand](https://elinux.org/Device_Tree_frowand) | صفحة Frank Rowand (خبير DT) بتحليلات عميقة |
| [Device tree plumbers 2016](https://elinux.org/Device_tree_plumbers_2016_etherpad) | نقاشات Linux Plumbers Conference 2016 عن DT |

---

### kernelnewbies.org

| الصفحة | الوصف |
|--------|-------|
| [Linux 6.8 Changes](https://kernelnewbies.org/Linux_6.8) | تضمين device trees داخل الـ kernel image |
| [Linux 6.12 Changes](https://kernelnewbies.org/Linux_6.12) | آخر تحسينات OF subsystem |
| [Linux 6.13 Changes](https://kernelnewbies.org/Linux_6.13) | تغييرات OF و device tree في أحدث kernel |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تتبع كل التغييرات عبر الإصدارات |

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model
- **الفصل 12 و 13**: USB and PCI drivers — أمثلة حية على قراءة properties
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD2/ch02.lwn)

#### Linux Kernel Development — Robert Love
- **الفصل 17**: Devices and Modules
- يشرح model الـ kobject و device tree integration

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 7**: Kernel Initialization — بيشرح الـ DT boot process
- **الفصل 11**: BusyBox and the Root File System — استخدام DT في embedded systems

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- تغطية عميقة لـ OF subsystem وكيف بيشتغل مع driver model

---

### مصطلحات للبحث

للقي معلومات أكتر، ابحث بالمصطلحات دي:

```
# بحث عام
"of_property_read_u32"
"of_graph_get_next_endpoint"
"device_node property"
"Open Firmware Linux"
"fwnode_handle of_node"

# على lore.kernel.org
subject:of_property site:lore.kernel.org
subject:of_graph site:lore.kernel.org

# على Google
site:kernel.org "of_graph_get_remote_port_parent"
site:elixir.bootlin.com "of_property_read_string"
```

**الـ Bootlin Elixir Cross-Referencer:**

[elixir.bootlin.com/linux/latest/source/drivers/of/property.c](https://elixir.bootlin.com/linux/latest/source/drivers/of/property.c)

ده أحسن أداة لقراءة الكود مع كل الـ cross-references للـ callers والـ definitions.

---

### ملفات الـ Kernel المرتبطة مباشرة

```
drivers/of/property.c        ← الملف الأساسي
drivers/of/base.c            ← الـ core OF functions
drivers/of/of_graph.c        ← (تاريخيًا كان هنا، اتدمج)
include/linux/of.h           ← structs و declarations الأساسية
include/linux/of_graph.h     ← of_graph_* API declarations
include/linux/fwnode.h       ← الـ abstraction layer فوق of_node
drivers/base/property.c      ← fwnode property API implementation
```
## Phase 8: Writing simple module

### الفكرة

**`of_property_read_u32_index`** هي function مُصدَّرة بـ `EXPORT_SYMBOL_GPL` من `drivers/of/property.c`. بتقرأ قيمة u32 من property معينة في Device Tree node بالـ index. هنعمل **kprobe** عليها عشان نراقب أي driver بيحاول يقرأ من Device Tree وإيه الـ property اللي بيطلبها.

ده مفيد جداً في debugging لأنك تشوف إيه الـ properties اللي بيطلبها الـ kernel/drivers أثناء الـ boot أو الـ hotplug.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_u32.c
 * Hooks of_property_read_u32_index to log every DT property read attempt.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe, register_kprobe */
#include <linux/of.h>          /* device_node, struct property */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author");
MODULE_DESCRIPTION("kprobe on of_property_read_u32_index to trace DT property reads");

/*
 * pre_handler is called just before of_property_read_u32_index executes.
 * pt_regs holds the CPU registers at the probe point — args are in
 * the standard calling convention registers (rdi, rsi, rdx, rcx on x86_64).
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * of_property_read_u32_index(np, propname, index, out_value)
     *   arg0 (rdi) = struct device_node *np
     *   arg1 (rsi) = const char *propname
     *   arg2 (rdx) = u32 index
     *   arg3 (rcx) = u32 *out_value
     */
    const struct device_node *np =
        (const struct device_node *)regs->di;   /* 1st arg */
    const char *propname = (const char *)regs->si; /* 2nd arg */
    u32 index = (u32)regs->dx;                    /* 3rd arg */

    /*
     * np->full_name gives the DT path like /soc/i2c@ff160000
     * We check np != NULL before dereferencing to avoid crash on bad callers.
     */
    if (np && propname)
        pr_info("of_u32_probe: node='%s' prop='%s' index=%u\n",
                np->full_name ? np->full_name : "(no name)",
                propname,
                index);

    return 0; /* 0 = continue normal execution */
}

/* kprobe struct — symbol_name tells the kernel which function to hook */
static struct kprobe kp = {
    .symbol_name = "of_property_read_u32_index",
    .pre_handler = handler_pre,
};

static int __init of_probe_init(void)
{
    int ret;

    /* register_kprobe patches the first byte of the target function
     * with a breakpoint instruction (int3 on x86). */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_u32_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("of_u32_probe: planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit of_probe_exit(void)
{
    /* unregister_kprobe restores the original instruction bytes
     * and removes the handler — MUST be called on exit to avoid
     * a dangling breakpoint that crashes the kernel after rmmod. */
    unregister_kprobe(&kp);
    pr_info("of_u32_probe: removed\n");
}

module_init(of_probe_init);
module_exit(of_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي module: `MODULE_LICENSE`، `module_init`، `module_exit` |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | الـ `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/of.h` | تعريف `struct device_node` عشان نقدر نـ cast الـ register value ونوصل لـ `full_name` |

#### الـ `handler_pre`

الـ handler بيتشغّل **قبل** ما الـ function الأصلية تتنفّذ. بياخد الـ `pt_regs` اللي فيه قيم الـ registers عند نقطة الـ probe — على x86_64 الـ arguments الأولانية بتتحط في `rdi, rsi, rdx, rcx` بالترتيب حسب **System V ABI**، فبنقرأ منهم مباشرة.

بنطبع `full_name` الخاص بالـ node عشان نعرف أي device في شجرة الـ DT طالب القراءة دي، مع اسم الـ property والـ index المطلوب.

#### `struct kprobe kp`

الـ `symbol_name` بيقول للـ kernel تحط الـ probe على إيه — الـ kernel بيعمل lookup في الـ symbol table ويرجع الـ address الفعلي في `.addr` بعد الـ register. الـ `pre_handler` هو الـ callback اللي بيتنادى عند كل call.

#### `module_init` و `module_exit`

- **`register_kprobe`**: بتحط **breakpoint** (int3) في أول byte من الـ function المستهدفة. لو فشلت (مثلاً الـ symbol مش موجود أو الـ kprobes مش enabled) بترجع error code.
- **`unregister_kprobe`**: ضرورية جداً في الـ `exit` — لو نسيتها والـ module اتـ unload، الـ breakpoint هيفضل موجود في الـ memory بس الـ handler اتشال، وده هيكسر الـ kernel في أي call جاية للـ function.

---

### تشغيل الـ Module

```bash
# Build (Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod kprobe_of_u32.ko

# شوف اللي بيطلع
sudo dmesg | grep of_u32_probe

# Unload
sudo rmmod kprobe_of_u32
```

**مثال على الـ output:**

```
[ 1234.567890] of_u32_probe: planted on of_property_read_u32_index at ffffffffc0123456
[ 1234.571234] of_u32_probe: node='/soc/serial@ff180000' prop='clock-frequency' index=0
[ 1234.572100] of_u32_probe: node='/soc/i2c@ff160000' prop='#address-cells' index=0
[ 1234.573456] of_u32_probe: node='/cpus/cpu@0' prop='reg' index=0
```

كل سطر بيكشف driver اتشغّل وحاول يقرأ property من الـ Device Tree — ده useful جداً لو بتـ debug مشكلة في driver مش بيلاقي الـ clock أو الـ address بتاعه.
