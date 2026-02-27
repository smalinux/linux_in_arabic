## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `include/linux/property.h`** ينتمي لـ subsystem اسمه **SOFTWARE NODES AND DEVICE PROPERTIES** (وهو جزء من **DRIVER CORE**)، maintainers هم Andy Shevchenko وفريق Intel. الملفات الأساسية للـ subsystem:

| الملف | الدور |
|-------|-------|
| `include/linux/property.h` | الـ public API header — موضوع شرحنا |
| `include/linux/fwnode.h` | الـ low-level types: `fwnode_handle`, `fwnode_operations` |
| `drivers/base/property.c` | الـ implementation (~1482 سطر) |
| `drivers/base/swnode.c` | الـ software nodes implementation (~1144 سطر) |

---

### المشكلة اللي بيحلها الملف ده — القصة الكاملة

تخيل إنك بتصمم kernel driver لـ sensor حرارة. الـ sensor ده موجود في أجهزة مختلفة:
- في Raspberry Pi: معلوماته موجودة في **Device Tree** (ملف `.dts`)
- في لاب توب Intel: معلوماته موجودة في **ACPI tables** (firmware من BIOS)
- في جهاز embedded مفيهوش firmware خالص: معلوماته مكتوبة **في الكود مباشرةً** (software node)

السؤال: ازاي تكتب driver واحد يشتغل مع الثلاث حالات دول من غير ما تحط `#ifdef` في كل حتة؟

```
قبل property.h:
  driver-for-DT.c  →  of_property_read_u32(np, "clock-freq", &val)
  driver-for-ACPI.c → acpi_dev_get_property(adev, "clock-freq", ...)
  driver-for-X86.c  → قراءة من table خاصة...

بعد property.h:
  driver.c (واحد بس) → device_property_read_u32(dev, "clock-freq", &val)
```

**الـ `property.h`** هو الـ "مترجم الموحد" — بيخلي أي driver يتكلم لغة واحدة مع أي مصدر معلومات firmware.

---

### الصورة الكبيرة — ELI5

فكّر في الموضوع زي كده:

**الـ Device Tree / ACPI / Software Node = بطاقة هوية الجهاز**

زي ما البطاقة الشخصية بتقول: "الاسم: أحمد، السن: 30، العنوان: القاهرة" — كمان كل hardware device بيحتاج "بطاقة هوية" بتقول:
- "الـ clock speed بتاعتي: 400MHz"
- "الـ interrupt رقم: 5"
- "الـ GPIO اللي بيتحكم فيّ: رقم 17"
- "أنا compatible مع: `vendor,my-sensor-v2`"

المشكلة إن في Linux في **أكتر من نوع بطاقة هوية**:

```
┌─────────────────────────────────────────────────┐
│              Hardware Description Sources        │
│                                                  │
│  ┌──────────────┐  ┌──────────┐  ┌───────────┐  │
│  │  Device Tree │  │   ACPI   │  │  Software │  │
│  │  (ARM/RISC-V)│  │  (x86)   │  │   Node    │  │
│  └──────┬───────┘  └────┬─────┘  └─────┬─────┘  │
│         │               │              │         │
│         └───────────────┼──────────────┘         │
│                         │                        │
│                  ┌──────▼──────┐                 │
│                  │fwnode_handle│  ← الوسيط الموحد │
│                  └──────┬──────┘                 │
│                         │                        │
│              ┌──────────▼──────────┐             │
│              │   property.h API    │             │
│              │  device_property_*  │             │
│              │  fwnode_property_*  │             │
│              └─────────────────────┘             │
└─────────────────────────────────────────────────┘
```

**الـ `fwnode_handle`** هو الـ "بطاقة هوية موحدة" — abstraction layer بيمثّل أي مصدر firmware بصرف النظر عن نوعه.

**الـ `property.h`** هو الـ API اللي بيتيح للـ driver يقرأ من أي بطاقة هوية بنفس الكود.

---

### ليه ده مفيد؟

**بدونه:** كل driver لازم يعرف هو شغّال على ARM مع DT ولا x86 مع ACPI ولا embedded بدون firmware، ويكتب كود مختلف لكل حالة.

**بيه:** Driver بسيط زي ده بيشتغل في كل مكان:

```c
static int my_sensor_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    u32 clock_freq;
    const char *label;

    /* نفس الكود على DT و ACPI و software node */
    if (device_property_read_u32(dev, "clock-frequency", &clock_freq))
        clock_freq = 1000000; /* default */

    if (!device_property_read_string(dev, "label", &label))
        dev_info(dev, "Sensor label: %s\n", label);

    return 0;
}
```

---

### محتوى الملف — الـ 4 طبقات

#### 1. قراءة الـ Properties من الـ Device مباشرةً
```c
/* قراءة قيمة u32 من أي firmware source */
device_property_read_u32(dev, "clock-freq", &val);

/* قراءة string */
device_property_read_string(dev, "label", &str);

/* التحقق من وجود property */
device_property_present(dev, "some-feature");

/* قراءة bool */
device_property_read_bool(dev, "enabled");
```

#### 2. قراءة الـ Properties من الـ `fwnode_handle` مباشرةً
نفس الـ API بس على مستوى الـ `fwnode` بدل الـ `device` — مفيد لما تتعامل مع child nodes أو external fwnodes:
```c
fwnode_property_read_u32(fwnode, "reg", &val);
fwnode_property_match_string(fwnode, "compatible", "vendor,chip");
```

#### 3. التنقل في شجرة الـ Nodes (Traversal)
الـ hardware مش flat — فيه parent/child relationships. مثلاً: `SoC → I2C bus → sensor`:
```c
/* التنقل بين الأبناء */
device_for_each_child_node(dev, child) {
    fwnode_property_read_u32(child, "reg", &addr);
}

/* الوصول لـ parent */
struct fwnode_handle *parent = fwnode_get_parent(fwnode);

/* البحث عن child باسم محدد */
struct fwnode_handle *child = device_get_named_child_node(dev, "ports");
```

#### 4. الـ Fwnode Graph — ربط الأجهزة ببعض
لما يكون فيه data path بين devices (مثلاً: camera → ISP → display)، بيستخدموا **fwnode graph**:
```c
/* الـ camera تعرف الـ ISP اللي بتتكلم معاه */
struct fwnode_handle *remote = fwnode_graph_get_remote_endpoint(ep);
```

#### 5. الـ Software Nodes — لما مفيش Firmware
لما الـ hardware مفهوش DT ولا ACPI (أو الـ firmware ناقص)، الـ driver نفسه بيعرّف الـ properties في الكود:
```c
/* تعريف properties في الكود */
static const struct property_entry my_device_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("label", "my-sensor"),
    PROPERTY_ENTRY_BOOL("some-feature"),
    { }  /* sentinel */
};

static const struct software_node my_swnode = {
    .name = "my-device",
    .properties = my_device_props,
};

/* ربطها بالـ device */
device_add_software_node(dev, &my_swnode);
```

---

### الـ `struct property_entry` — كيف بتُخزّن الـ Property

```c
struct property_entry {
    const char *name;       /* اسم الـ property */
    size_t length;          /* حجم الـ value بالبايت */
    bool is_inline;         /* هل الـ value صغيرة تتخزن inline؟ */
    enum dev_prop_type type; /* U8, U16, U32, U64, STRING, أو REF */
    union {
        const void *pointer; /* لو الـ data كبيرة → pointer */
        union {
            u8  u8_data[8];
            u16 u16_data[4];
            u32 u32_data[2];
            u64 u64_data[1];
            const char *str[...];
        } value;             /* لو الـ data صغيرة → inline */
    };
};
```

الـ `is_inline = true` معناها الـ value اتخزنت جوه الـ struct نفسه (optimization لتجنب memory allocation إضافي للقيم الصغيرة).

---

### الـ Architecture الكاملة — كيف بيشتغل كل ده

```
┌──────────────────────────────────────────────────────────┐
│                   Driver Code                            │
│  device_property_read_u32(dev, "reg", &val)              │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│              drivers/base/property.c                     │
│  1. استخرج dev->fwnode                                   │
│  2. نادي fwnode->ops->property_read_int_array(...)       │
└───────────┬───────────────────────────────────┬──────────┘
            │                                   │
            ▼                                   ▼
┌───────────────────────┐         ┌─────────────────────────┐
│   OF (Device Tree)    │         │         ACPI            │
│   of_fwnode_ops       │         │    acpi_fwnode_ops       │
│  (drivers/of/...)     │         │  (drivers/acpi/...)      │
└───────────────────────┘         └─────────────────────────┘
            │                                   │
            ▼                                   ▼
    يقرأ من DTB                      يقرأ من ACPI tables
    في الـ RAM                        في الـ firmware
```

---

### الملفات المرتبطة اللي المبرمج يعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/fwnode.h` | الـ low-level types والـ `fwnode_operations` vtable |
| `include/linux/of.h` | الـ Device Tree specific API |
| `include/linux/acpi.h` | الـ ACPI specific API |
| `drivers/base/property.c` | الـ implementation للـ unified API |
| `drivers/base/swnode.c` | الـ software nodes implementation |
| `drivers/of/property.c` | الـ DT backend للـ fwnode ops |
| `drivers/acpi/property.c` | الـ ACPI backend للـ fwnode ops |
| `include/linux/of_graph.h` | الـ DT graph API (كاميرات، media) |
| `Documentation/driver-api/device_link.rst` | توثيق الـ device links |
## Phase 2: شرح الـ Unified Device Property / fwnode Framework

---

### المشكلة اللي بيحلها الـ Framework ده

**الـ Linux kernel** بيشتغل على hardware بتتوصفله بطرق مختلفة جداً:

- **Device Tree (DT)**: المستخدمة في ARM/RISC-V embedded — ملفات `.dts` بتوصف الـ hardware كـ tree من nodes بـ properties.
- **ACPI**: المستخدمة في x86/ARM servers — tables بتيجي من firmware بتوصف الـ hardware بـ namespace موحد.
- **Software Nodes**: مش فيه firmware خالص — الـ driver أو الـ board file بتعرّف الـ properties في الـ C code مباشرة.

**المشكلة**: كل source ليها API مختلف. لو driver عايز يقرأ property اسمها `clock-frequency`:
- في DT بيستخدم `of_property_read_u32(np, "clock-frequency", &val)`
- في ACPI بيستخدم شيء تاني خالص
- النتيجة: driver مكتوب مرتين أو مليان `#ifdef CONFIG_OF` و `#ifdef CONFIG_ACPI`

ده code duplication بيخلي الـ driver fragile ومش portable.

---

### الحل: Unified Device Property Interface

الـ kernel حل المشكلة دي بـ abstraction layer اسمه **firmware node (fwnode)**.

الفكرة:
1. كل مصدر وصف hardware (DT، ACPI، software) بيوفر `struct fwnode_operations` — table من function pointers.
2. كل node في الـ tree (DT node، ACPI device، software node) بتتمثل بـ `struct fwnode_handle`.
3. الـ driver بيتكلم مع الـ `fwnode_handle` بس، مش مع DT أو ACPI مباشرة.
4. الـ `property.h` هو الـ API اللي بيستخدمه الـ driver — generic وموحد.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Code                              │
│  device_property_read_u32(dev, "reg", &val)                    │
│  fwnode_property_present(fwnode, "compatible")                  │
└────────────────────┬────────────────────────────────────────────┘
                     │  calls into
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              Unified Property API  (property.h / property.c)    │
│                                                                 │
│   device_property_*()  ──►  dev_fwnode(dev)  ──►  fwnode_*()  │
└────────────────────┬────────────────────────────────────────────┘
                     │  dispatches via fwnode->ops->property_*()
                     ▼
        ┌────────────┴──────────────┐──────────────────┐
        │                           │                  │
        ▼                           ▼                  ▼
┌──────────────┐           ┌──────────────┐   ┌──────────────────┐
│  OF/DT Layer │           │  ACPI Layer  │   │  Software Nodes  │
│  (of.c)      │           │  (acpi.c)    │   │  (swnode.c)      │
│              │           │              │   │                  │
│ fwnode_ops   │           │ fwnode_ops   │   │ fwnode_ops       │
│ backed by    │           │ backed by    │   │ backed by        │
│ device_node  │           │ acpi_device  │   │ property_entry[] │
└──────┬───────┘           └──────┬───────┘   └────────┬─────────┘
       │                          │                     │
       ▼                          ▼                     ▼
  .dts / dtb files          ACPI BIOS Tables       C code arrays
  (Boot-time loaded)        (UEFI firmware)        (compile-time)
```

**الـ dev_fwnode(dev)** هو الـ macro اللي بيجيب الـ `fwnode_handle` المرتبط بالـ `struct device`.

---

### التشبيه الحقيقي — Hardware Store مع Translators

تخيل إن عندك مهندس (الـ driver) بيشتغل على مشروع، وبيحتاج يعرف "ما هو الـ voltage للـ component ده؟"

المشكلة: الـ datasheet ممكن يكون:
- **باليابانية** (Device Tree)
- **بالألمانية** (ACPI)
- **مكتوب بخط يدك** في نوتة (Software Node)

الحل: توظف **مترجم موحد** (الـ fwnode API) بيتكلم مع كل مصدر ويرجع لك الإجابة بالإنجليزي.

| التشبيه | الـ Kernel Concept |
|---------|-------------------|
| المهندس (الـ driver) | أي kernel driver محتاج يقرأ config |
| السؤال عن الـ voltage | `device_property_read_u32(dev, "voltage", &v)` |
| المترجم الموحد | `fwnode_property_read_u32()` |
| الـ datasheet باليابانية | Device Tree node (`.dts`) |
| الـ datasheet بالألمانية | ACPI namespace object |
| النوتة المكتوبة بخط يدك | `software_node` مع `property_entry[]` |
| الـ translator's desk | `struct fwnode_handle` |
| قواعد ترجمة كل لغة | `struct fwnode_operations` |
| عنوان الـ desk في الشركة | `dev->fwnode` pointer في `struct device` |

---

### الـ Core Abstraction: الـ `fwnode_handle`

**الـ `struct fwnode_handle`** هو حجر الأساس في الـ framework كله. ده مش بيحتوي على data — هو handle بس، زي الـ file descriptor في POSIX:

```c
struct fwnode_handle {
    struct fwnode_handle *secondary;   /* fallback fwnode (e.g. DT + swnode together) */
    const struct fwnode_operations *ops; /* vtable — who handles operations */

    /* used by device links subsystem */
    struct device *dev;
    struct list_head suppliers;
    struct list_head consumers;
    u8 flags;
};
```

**الـ `secondary`** ده مهم: ممكن device عندها DT node (الـ primary) وفي نفس الوقت software node (الـ secondary) بيضيف properties ناقصة. الـ framework بيسأل الـ primary الأول، لو مش لاقي يسأل الـ secondary.

---

### الـ vtable: `struct fwnode_operations`

ده الـ interface اللي لازم كل backend (DT، ACPI، swnode) يـimplement:

```c
struct fwnode_operations {
    /* lifecycle */
    struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
    void (*put)(struct fwnode_handle *fwnode);

    /* device status */
    bool (*device_is_available)(const struct fwnode_handle *fwnode);
    const void *(*device_get_match_data)(...);
    bool (*device_dma_supported)(...);
    enum dev_dma_attr (*device_get_dma_attr)(...);

    /* property access */
    bool (*property_present)(const struct fwnode_handle *, const char *propname);
    bool (*property_read_bool)(...);
    int (*property_read_int_array)(...);
    int (*property_read_string_array)(...);

    /* tree traversal */
    const char *(*get_name)(...);
    struct fwnode_handle *(*get_parent)(...);
    struct fwnode_handle *(*get_next_child_node)(...);
    struct fwnode_handle *(*get_named_child_node)(...);

    /* references */
    int (*get_reference_args)(...);

    /* graph (media/camera pipelines) */
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);

    /* misc */
    void __iomem *(*iomap)(...);
    int (*irq_get)(...);
    int (*add_links)(struct fwnode_handle *fwnode); /* for devlinks */
};
```

الـ dispatch بيتم عبر macros زي:
```c
#define fwnode_call_int_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ?            \
     (fwnode)->ops->op(fwnode, ##__VA_ARGS__) : -ENXIO)
```

لو الـ backend مش implement الـ operation ده، بيرجع `-ENXIO` تلقائي.

---

### الـ Property Types والـ `struct property_entry`

**الـ `enum dev_prop_type`** بيحدد أنواع الـ properties الممكنة:

```c
enum dev_prop_type {
    DEV_PROP_U8,
    DEV_PROP_U16,
    DEV_PROP_U32,
    DEV_PROP_U64,
    DEV_PROP_STRING,
    DEV_PROP_REF,    /* reference to another fwnode — زي phandles في DT */
};
```

**الـ `struct property_entry`** هو الـ in-memory representation لـ property واحدة (مستخدم في software nodes):

```c
struct property_entry {
    const char *name;
    size_t length;
    bool is_inline;           /* true = value stored in struct itself */
    enum dev_prop_type type;
    union {
        const void *pointer;  /* when is_inline == false */
        union {
            u8  u8_data[8];
            u16 u16_data[4];
            u32 u32_data[2];
            u64 u64_data[1];
            const char *str[1]; /* on 64-bit: sizeof(u64)/sizeof(char*) = 1 */
        } value;              /* when is_inline == true */
    };
};
```

**تحسين مهم**: الـ `is_inline` بيخلي values صغيرة (u8, u16, u32, u64 فردية) متخزنة مباشرة في الـ struct من غير heap allocation. ده بيقلل fragmentation لأن الـ property_entry arrays بتكون static في الـ driver.

```
property_entry (is_inline = true):
┌──────────┬────────┬───────────┬──────┬────────────────────────┐
│  name *  │ length │ is_inline │ type │  value.u32_data[0]=42  │
└──────────┴────────┴───────────┴──────┴────────────────────────┘

property_entry (is_inline = false):
┌──────────┬────────┬───────────┬──────┬────────────────────────┐
│  name *  │ length │   false   │ type │  pointer → u32 array[] │
└──────────┴────────┴───────────┴──────┴────────────────────────┘
                                              │
                                              ▼
                                        { 100, 200, 300, 400 }
```

---

### الـ Software Nodes — لما مفيش Firmware

**الـ `struct software_node`** بيسمح لـ driver أو platform code يوصف hardware من C code خالص:

```c
struct software_node {
    const char *name;
    const struct software_node *parent;    /* tree structure */
    const struct property_entry *properties;
};
```

مثال حقيقي: I2C device مش عنده DT node (legacy x86 board):

```c
/* define the properties */
static const struct property_entry my_sensor_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("compatible", "bosch,bme280"),
    PROPERTY_ENTRY_BOOL("wakeup-source"),
    { }  /* sentinel */
};

/* define the software node */
static const struct software_node my_sensor_swnode = {
    .name       = "bme280-sensor",
    .properties = my_sensor_props,
};

/* register it and attach to device */
int init_sensor(struct device *dev)
{
    return device_add_software_node(dev, &my_sensor_swnode);
}
```

بعد كده الـ driver بيقرأ بنفس الـ API:

```c
u32 freq;
device_property_read_u32(dev, "clock-frequency", &freq); /* works! */
```

---

### الـ Reference Properties والـ `software_node_ref_args`

الـ DT بيدعم مفهوم **phandles** — property بيشير لـ node تاني في الـ tree. مثال:

```dts
clocks = <&clk_apb 3>;  /* reference to clk_apb node, arg=3 */
```

الـ equivalent في software nodes هو `struct software_node_ref_args`:

```c
struct software_node_ref_args {
    const struct software_node *swnode;  /* target node */
    struct fwnode_handle *fwnode;        /* or a raw fwnode */
    unsigned int nargs;
    u64 args[NR_FWNODE_REFERENCE_ARGS]; /* up to 16 extra args */
};
```

الـ macro `SOFTWARE_NODE_REFERENCE` بيستخدم `_Generic` يـtype-check الـ reference وقت الـ compilation:

```c
#define SOFTWARE_NODE_REFERENCE(_ref_, ...)     \
(const struct software_node_ref_args) {         \
    .swnode = _Generic(_ref_,                   \
        const struct software_node *: _ref_,    \
        struct software_node *: _ref_,          \
        default: NULL),                         \
    .fwnode = _Generic(_ref_,                   \
        struct fwnode_handle *: _ref_,          \
        default: NULL),                         \
    .nargs = COUNT_ARGS(__VA_ARGS__),           \
    .args = { __VA_ARGS__ },                    \
}
```

---

### الـ Fwnode Graph — Camera / Media Pipelines

الـ framework بيدعم **graph topology** بين devices عبر مفهوم الـ **ports** و **endpoints**.

مثال: camera sensor متصل بـ ISP (Image Signal Processor):

```
sensor node                    ISP node
┌──────────────┐              ┌──────────────┐
│  port@0      │              │  port@0      │
│  endpoint@0  │◄────────────►│  endpoint@0  │
│  (TX)        │ remote-endpoint  (RX)       │
└──────────────┘              └──────────────┘
```

الـ API بيسمح traverse الـ graph:

```c
/* get the remote endpoint connected to a local endpoint */
struct fwnode_handle *remote =
    fwnode_graph_get_remote_endpoint(local_endpoint);

/* get the device that owns the remote endpoint */
struct fwnode_handle *remote_dev =
    fwnode_graph_get_remote_port_parent(local_endpoint);

/* iterate all endpoints of a device */
fwnode_graph_for_each_endpoint(dev_fwnode(dev), ep) {
    /* process each endpoint */
}
```

---

### الـ Two-Level API: `device_*` vs `fwnode_*`

الـ framework بيوفر مستويين من الـ API:

| **الـ `device_*` API** | **الـ `fwnode_*` API** |
|------------------------|------------------------|
| `device_property_read_u32(dev, ...)` | `fwnode_property_read_u32(fwnode, ...)` |
| بياخد `struct device *` | بياخد `struct fwnode_handle *` |
| بيعمل `dev_fwnode(dev)` داخلياً | مباشر على الـ fwnode |
| للـ drivers اللي عندهم `struct device` | لما بتتعامل مع child nodes أو references |

**الـ `dev_fwnode(dev)`** هو macro بيستخدم `_Generic` لـ const-correctness:

```c
#define dev_fwnode(dev)                          \
    _Generic((dev),                              \
        const struct device *: __dev_fwnode_const, \
        struct device *: __dev_fwnode)(dev)
```

---

### الـ Scoped Iteration — Memory Safety

الـ framework بيدعم **scoped iteration** مستخدماً الـ cleanup mechanism (`__free`):

```c
/* classic iteration — requires manual fwnode_handle_put() on break */
fwnode_for_each_child_node(parent, child) {
    if (condition)
        break; /* MUST call fwnode_handle_put(child) here! */
}

/* scoped iteration — automatic cleanup via DEFINE_FREE */
fwnode_for_each_child_node_scoped(parent, child) {
    if (condition)
        break; /* child is auto-released when scope exits */
}
```

الـ `DEFINE_FREE` هو macro من الـ `cleanup.h` subsystem بيـregister destructor للـ variable:

```c
DEFINE_FREE(fwnode_handle, struct fwnode_handle *, fwnode_handle_put(_T))
```

ده بيمنع reference leaks لما الـ loop بيـbreak مبكر.

---

### ما الذي يـown الـ Framework vs ما بيـdelegate للـ Backends

```
┌──────────────────────────────────────────────────────────────┐
│              ما الـ Framework بيـOWN                         │
├──────────────────────────────────────────────────────────────┤
│ • الـ unified API surface (device_property_*, fwnode_*)      │
│ • الـ dispatch logic (fwnode_call_*_op macros)              │
│ • الـ secondary fwnode fallback logic                        │
│ • الـ graph traversal wrappers                              │
│ • الـ software_node implementation كاملة                    │
│ • الـ property_entry inline/pointer storage logic           │
│ • الـ scoped iteration + DEFINE_FREE                        │
│ • الـ connection finding (fwnode_connection_find_match)      │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│              ما بيـDelegate للـ Backends                     │
├──────────────────────────────────────────────────────────────┤
│ • parsing الـ DT binary blob ← OF subsystem                  │
│ • parsing الـ ACPI tables ← ACPI subsystem                  │
│ • reference counting semantics (get/put) ← كل backend بـalgo│
│ • تحديد لو device "available" ← backend بيقرر              │
│ • الـ IRQ mapping من property index ← platform-specific     │
│ • الـ iomap للـ memory regions ← backend-specific           │
│ • إنشاء فـdevlinks (add_links) ← backend بيعرف topology    │
└──────────────────────────────────────────────────────────────┘
```

---

### كيفية ربط كل الـ Structs ببعض — Big Picture

```
struct device
├── fwnode: *fwnode_handle  ──────────────────────────────────┐
└── ...                                                        │
                                                               ▼
                                             struct fwnode_handle
                                             ├── secondary: *fwnode_handle (optional)
                                             ├── ops: *fwnode_operations
                                             │       ├── get / put
                                             │       ├── property_present
                                             │       ├── property_read_int_array
                                             │       ├── property_read_string_array
                                             │       ├── get_parent
                                             │       ├── get_next_child_node
                                             │       ├── graph_get_next_endpoint
                                             │       └── add_links
                                             ├── dev: *device (backref)
                                             ├── suppliers: list_head
                                             ├── consumers: list_head
                                             └── flags: u8

If backend == software_node:
    fwnode_handle is embedded inside:
    struct swnode {
        struct kobject kobj;
        struct fwnode_handle fwnode;  ◄── embedded here
        const struct software_node *node;
        struct swnode *parent;
        struct list_head children;
        /* ... */
    }
    and software_node contains:
    struct software_node
    ├── name: "my-device"
    ├── parent: *software_node
    └── properties: *property_entry[]
                    ├── { "clock-frequency", 4, true, U32, {.u32_data[0]=400000} }
                    ├── { "compatible",      8, true, STR, {.str[0]="bosch,bme280"} }
                    └── { NULL }  ← sentinel
```

---

### مثال Driver حقيقي: I2C Touchscreen

```c
/* في board file أو platform init code */
static const struct property_entry ts_props[] = {
    PROPERTY_ENTRY_U32("touchscreen-size-x", 1080),
    PROPERTY_ENTRY_U32("touchscreen-size-y", 1920),
    PROPERTY_ENTRY_BOOL("touchscreen-inverted-x"),
    PROPERTY_ENTRY_STRING("compatible", "edt,edt-ft5406"),
    { }
};

static const struct software_node ts_swnode = {
    .name       = "ts@38",
    .properties = ts_props,
};

/* في الـ driver probe() */
static int ts_probe(struct i2c_client *client)
{
    struct device *dev = &client->dev;
    u32 width, height;
    bool inv_x;

    /* يقرأ بـ unified API — مش مهم مصدر الـ properties */
    if (device_property_read_u32(dev, "touchscreen-size-x", &width))
        width = 800; /* default */

    if (device_property_read_u32(dev, "touchscreen-size-y", &height))
        height = 600;

    inv_x = device_property_read_bool(dev, "touchscreen-inverted-x");

    dev_info(dev, "TS: %dx%d, inv_x=%d\n", width, height, inv_x);
    return 0;
}
```

نفس الـ driver بيشتغل مع DT، ACPI، أو software_node من غير أي تغيير.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### `enum dev_prop_type` — أنواع الـ properties

| القيمة | المعنى |
|---|---|
| `DEV_PROP_U8` | integer 8-bit |
| `DEV_PROP_U16` | integer 16-bit |
| `DEV_PROP_U32` | integer 32-bit (الأشيع في الـ DT) |
| `DEV_PROP_U64` | integer 64-bit |
| `DEV_PROP_STRING` | **الـ** `const char *` |
| `DEV_PROP_REF` | reference لـ node تاني (`software_node_ref_args`) |

#### `enum dev_dma_attr` — قدرات الـ DMA

| القيمة | المعنى |
|---|---|
| `DEV_DMA_NOT_SUPPORTED` | الجهاز مش بيدعم DMA خالص |
| `DEV_DMA_NON_COHERENT` | DMA موجود بس محتاج sync يدوي |
| `DEV_DMA_COHERENT` | DMA coherent — الـ cache متزامن أوتوماتيك |

#### `FWNODE_FLAG_*` — flags الـ fwnode_handle

| الـ Flag | البت | المعنى |
|---|---|---|
| `FWNODE_FLAG_LINKS_ADDED` | `BIT(0)` | تم parse الـ fwnode links بالفعل |
| `FWNODE_FLAG_NOT_DEVICE` | `BIT(1)` | الـ node دي مش هتتحول لـ `struct device` |
| `FWNODE_FLAG_INITIALIZED` | `BIT(2)` | الـ hardware اتهيأ |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | `BIT(3)` | يحتاج الـ child devices تتـ bind الأول |
| `FWNODE_FLAG_BEST_EFFORT` | `BIT(4)` | يعدي لو الـ suppliers ناقصين |
| `FWNODE_FLAG_VISITED` | `BIT(5)` | اتزار أثناء الـ traversal (داخلي) |

#### `FWLINK_FLAG_*` — flags الـ fwnode_link

| الـ Flag | البت | المعنى |
|---|---|---|
| `FWLINK_FLAG_CYCLE` | `BIT(0)` | الـ link جزء من cycle — متـ defer الـ probe |
| `FWLINK_FLAG_IGNORE` | `BIT(1)` | تجاهل الـ link تماماً حتى في cycle detection |

#### `FWNODE_GRAPH_*` — flags بحث الـ graph

| الـ Flag | البت | المعنى |
|---|---|---|
| `FWNODE_GRAPH_ENDPOINT_NEXT` | `BIT(0)` | لو مفيش exact match، خد أقرب ID أكبر |
| `FWNODE_GRAPH_DEVICE_DISABLED` | `BIT(1)` | ضم الـ endpoints اللي جهازها disabled |

#### الـ Config Options المؤثرة

| الـ Option | التأثير |
|---|---|
| `CONFIG_CPU_BIG_ENDIAN` | يأثر على `fwnode_device_is_big_endian()` — لو مفعل، الـ `native-endian` property تعني BE |

#### الـ Macros المساعدة — cheatsheet

| الـ Macro | الاستخدام |
|---|---|
| `PROPERTY_ENTRY_U8(name, val)` | inline scalar u8 |
| `PROPERTY_ENTRY_U32(name, val)` | inline scalar u32 (الأكثر شيوعاً) |
| `PROPERTY_ENTRY_STRING(name, val)` | inline string |
| `PROPERTY_ENTRY_BOOL(name)` | boolean — length=0, is_inline=true |
| `PROPERTY_ENTRY_U32_ARRAY(name, arr)` | array بـ pointer خارجي |
| `PROPERTY_ENTRY_REF(name, ref, ...)` | reference لـ software_node مع args |
| `SOFTWARE_NODE_REFERENCE(ref, ...)` | يبني `software_node_ref_args` inline |
| `SOFTWARE_NODE(name, props, parent)` | يبني `software_node` بسرعة |
| `NR_FWNODE_REFERENCE_ARGS` | `16` — أقصى عدد args في reference |

---

### 1. الـ Structs المهمة

#### `struct fwnode_handle`

**الغرض:** الـ handle الأساسي لأي firmware node — سواء OF (Device Tree)، ACPI، أو software. كل node في الـ tree بتتمثل بـ `fwnode_handle`.

```c
struct fwnode_handle {
    struct fwnode_handle *secondary; // فلبك لـ node تاني (مثلاً ACPI fallback لـ DT)
    const struct fwnode_operations *ops; // الـ vtable — محدد نوع الـ backend
    struct device *dev;       // الـ device المربوط بالـ node (لو موجود)
    struct list_head suppliers; // الـ nodes اللي الـ node دي تعتمد عليها
    struct list_head consumers; // الـ nodes اللي بتعتمد على الـ node دي
    u8 flags;                 // FWNODE_FLAG_* بالـ bits
};
```

**الارتباطات:**
- بتشاور على `fwnode_operations` (الـ vtable)
- مربوطة بـ `struct device` عن طريق `dev`
- بتشكل tree عن طريق `get_parent` / `get_next_child_node` في الـ ops

---

#### `struct fwnode_operations`

**الغرض:** الـ vtable (virtual function table) — بتفصل الـ API العام عن الـ backend implementation (OF/ACPI/swnode).

```c
struct fwnode_operations {
    struct fwnode_handle *(*get)(struct fwnode_handle *);   // ref-count up
    void (*put)(struct fwnode_handle *);                    // ref-count down
    bool (*device_is_available)(...);    // هل الجهاز enabled في الـ firmware؟
    const void *(*device_get_match_data)(...); // match data من الـ id_table
    bool (*device_dma_supported)(...);
    enum dev_dma_attr (*device_get_dma_attr)(...);
    bool (*property_present)(...);        // هل الـ property موجودة؟
    bool (*property_read_bool)(...);
    int (*property_read_int_array)(...);  // قراءة integer array (core API)
    int (*property_read_string_array)(...);
    const char *(*get_name)(...);
    const char *(*get_name_prefix)(...);
    struct fwnode_handle *(*get_parent)(...);
    struct fwnode_handle *(*get_next_child_node)(...);
    struct fwnode_handle *(*get_named_child_node)(...);
    int (*get_reference_args)(...);       // resolve property reference مع args
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);
    void __iomem *(*iomap)(...);          // map مساحة الـ registers
    int (*irq_get)(...);
    int (*add_links)(...);                // يبني dependency links
};
```

**الارتباطات:** كل `fwnode_handle` بيشاور على instance واحد من الـ ops (مشترك بين كل nodes من نفس النوع).

---

#### `struct property_entry`

**الغرض:** تمثيل واحدة property في الـ software node — سواء كانت قيمة inline صغيرة أو pointer لبيانات خارجية.

```c
struct property_entry {
    const char *name;     // اسم الـ property (مثلاً "clock-frequency")
    size_t length;        // حجم البيانات بالـ bytes
    bool is_inline;       // true = القيمة محفوظة مباشرة في الـ union
    enum dev_prop_type type; // نوع البيانات (U8/U16/U32/U64/STRING/REF)
    union {
        const void *pointer;  // pointer للبيانات لو مش inline
        union {
            u8  u8_data[8];   // inline: 8 قيم u8
            u16 u16_data[4];  // inline: 4 قيم u16
            u32 u32_data[2];  // inline: 2 قيم u32
            u64 u64_data[1];  // inline: قيمة u64 واحدة
            const char *str[1]; // inline: string pointer واحد
        } value;
    };
};
```

**ملحوظة مهمة:** الـ inline union كلها بحجم 8 bytes (حجم `u64`) — ده تصميم مقصود عشان توفر allocation لـ scalars صغيرة.

**الارتباطات:**
- array منها بتتحمل في `software_node.properties`
- بتستخدم `software_node_ref_args` لما `type == DEV_PROP_REF`

---

#### `struct software_node`

**الغرض:** node افتراضية بالكامل في الـ kernel — بتوفر firmware description لما الـ hardware مش معرّف في ACPI أو Device Tree (مثلاً platform devices قديمة أو USB devices محتاجة properties إضافية).

```c
struct software_node {
    const char *name;                      // اسم الـ node
    const struct software_node *parent;    // الـ parent في الـ tree
    const struct property_entry *properties; // array من الـ properties (NULL-terminated)
};
```

**الارتباطات:**
- بتتحول لـ `fwnode_handle` عن طريق `software_node_fwnode()`
- الـ properties بتاعتها بتتقرأ عن طريق نفس `fwnode_property_*` API
- ممكن تتربط بـ `struct device` عن طريق `device_add_software_node()`

---

#### `struct software_node_ref_args`

**الغرض:** property من نوع reference — بتشاور على node تانية مع arguments إضافية (مثل `clocks = <&clk0 0>` في DT).

```c
struct software_node_ref_args {
    const struct software_node *swnode; // reference لـ software_node
    struct fwnode_handle *fwnode;       // أو reference مباشر لـ fwnode
    unsigned int nargs;                 // عدد الـ args
    u64 args[NR_FWNODE_REFERENCE_ARGS]; // الـ args (max 16)
};
```

**الارتباطات:**
- بتتخزن كـ `pointer` في `property_entry` لما `type == DEV_PROP_REF`
- بتتقرأ عن طريق `fwnode_property_get_reference_args()`

---

#### `struct fwnode_link`

**الغرض:** dependency link بين node supplier وnode consumer — الـ kernel بيستخدمها عشان يتأكد إن الـ supplier اتـ probe قبل الـ consumer.

```c
struct fwnode_link {
    struct fwnode_handle *supplier; // الـ node اللي بتوفر (مثلاً clock provider)
    struct list_head s_hook;        // hook في قائمة suppliers.consumers
    struct fwnode_handle *consumer; // الـ node اللي بتستهلك
    struct list_head c_hook;        // hook في قائمة consumers.suppliers
    u8 flags;                       // FWLINK_FLAG_*
};
```

---

#### `struct fwnode_endpoint`

**الغرض:** يمثل endpoint في الـ fwnode graph — نظام الـ OF graph للتعبير عن الاتصالات بين الأجهزة (مثلاً camera sensor → ISP).

```c
struct fwnode_endpoint {
    unsigned int port;                    // رقم الـ port
    unsigned int id;                      // رقم الـ endpoint في الـ port
    const struct fwnode_handle *local_fwnode; // الـ fwnode الخاص بالـ endpoint ده
};
```

---

#### `struct fwnode_reference_args`

**الغرض:** نتيجة `get_reference_args` — بتحتوي على الـ fwnode المُشار إليه مع الـ integer arguments.

```c
struct fwnode_reference_args {
    struct fwnode_handle *fwnode; // الـ node المُشار إليها
    unsigned int nargs;           // عدد الـ args الفعلية
    u64 args[NR_FWNODE_REFERENCE_ARGS]; // القيم (max 16)
};
```

---

### 2. مخطط علاقات الـ Structs

```
                    ┌──────────────────────────────┐
                    │       struct device           │
                    │  (drivers/base/core.c)        │
                    │                               │
                    │  fwnode ──────────────────────┼──┐
                    └──────────────────────────────┘  │
                                                       │ dev_fwnode(dev)
                                                       ▼
┌────────────────────────────────────────────────────────────────┐
│                   struct fwnode_handle                          │
│                                                                 │
│  secondary ──────────────────────────────────► fwnode_handle   │
│             (fallback, e.g. ACPI for OF node)  (another type)  │
│                                                                 │
│  ops ────────────────────────────────────────► fwnode_operations│
│                                                 (vtable)        │
│                                                                 │
│  dev ◄──────────────────────────────────────── struct device   │
│                                                                 │
│  suppliers ──────► list_head ◄──── fwnode_link.c_hook          │
│  consumers ──────► list_head ◄──── fwnode_link.s_hook          │
│  flags (u8)                                                     │
└────────────────────────────────────────────────────────────────┘
          ▲                    ▲
          │ parent/child       │ software backend
          │                    │
┌─────────────────┐   ┌────────────────────────────────┐
│  fwnode_handle  │   │      struct software_node      │
│  (OF/ACPI node) │   │                                │
└─────────────────┘   │  name                          │
                      │  parent ────────────────────►  software_node
                      │  properties ────────────────►  property_entry[]
                      └────────────────────────────────┘
                                        │
                                        ▼
                      ┌────────────────────────────────┐
                      │     struct property_entry      │
                      │                                │
                      │  name (string)                 │
                      │  length                        │
                      │  is_inline (bool)              │
                      │  type (dev_prop_type)          │
                      │  union {                       │
                      │    pointer ────────────────►   │ (array data / ref_args)
                      │    value { u8/u16/u32/u64/str }│
                      │  }                             │
                      └────────────────────────────────┘
                                (لما type == DEV_PROP_REF)
                                        │ pointer
                                        ▼
                      ┌────────────────────────────────┐
                      │  struct software_node_ref_args │
                      │                                │
                      │  swnode ──────────────────────►software_node
                      │  fwnode ──────────────────────►fwnode_handle
                      │  nargs                         │
                      │  args[16]                      │
                      └────────────────────────────────┘


فـ الـ fwnode_link:
┌─────────────────┐     fwnode_link      ┌─────────────────┐
│   supplier      │◄──────────────────── │   consumer      │
│  fwnode_handle  │  s_hook ←→ c_hook   │  fwnode_handle  │
└─────────────────┘                      └─────────────────┘
```

---

### 3. دورة حياة الـ Structs

#### الـ software_node — من التعريف للاستخدام للحذف

```
[1] تعريف static (compile-time)
    ───────────────────────────
    static const struct property_entry props[] = {
        PROPERTY_ENTRY_U32("clock-frequency", 400000),
        PROPERTY_ENTRY_STRING("compatible", "vendor,chip"),
        { }  /* sentinel */
    };

    static const struct software_node node = {
        .name = "my-device",
        .properties = props,
    };

         │
         ▼
[2] التسجيل في الـ kernel
    ───────────────────────
    software_node_register(&node);
    /* أو لمجموعة: software_node_register_node_group(group) */

    داخلياً:
    - بيخصص swnode (software_node private struct)
    - بيبني fwnode_handle مدمج
    - بيضيفه للـ global swnode list

         │
         ▼
[3] الربط بالـ device
    ──────────────────
    device_add_software_node(dev, &node);
    /* أو: device_create_managed_software_node(dev, props, parent) */

    - بيربط fwnode_handle بـ dev->fwnode

         │
         ▼
[4] الاستخدام من الـ driver
    ──────────────────────
    device_property_read_u32(dev, "clock-frequency", &val);
    /* يمشي: dev → fwnode → ops->property_read_int_array */

         │
         ▼
[5] الحذف
    ──────
    device_remove_software_node(dev);
    /* أو: fwnode_remove_software_node(fwnode) */
    /* أو: software_node_unregister(&node) */

    - بيفصل الـ fwnode عن الـ device
    - بيحذف الـ swnode من الـ list
    - الـ ref-count لازم يوصل صفر قبل الـ free الفعلي
```

#### الـ fwnode_handle reference counting

```
fwnode_handle_get(fwnode)  →  ref++
fwnode_handle_put(fwnode)  →  ref--  →  (لو وصل 0) → ops->put() → free

في الـ iteration loops:
    fwnode_for_each_child_node(parent, child) {
        /* child got via get_next_child_node = implicit get */
        if (done) {
            fwnode_handle_put(child);  /* لازم لو break */
            break;
        }
    }
    /* loop طبيعي = الـ last iteration بترجع NULL = مفيش put مطلوب */

    الـ _scoped variants بتعمل __free(fwnode_handle) أوتوماتيك:
    fwnode_for_each_child_node_scoped(parent, child) {
        /* child بيتـ put أوتوماتيك عند الخروج من الـ scope */
    }
```

---

### 4. مخططات تدفق الـ API

#### قراءة property من device

```
driver:
  device_property_read_u32(dev, "reg", &val)
    │
    ├─► dev_fwnode(dev)
    │     └─► _Generic(dev):
    │           const dev* → __dev_fwnode_const(dev)
    │           dev*       → __dev_fwnode(dev)
    │         يرجع dev->fwnode (أو dev->of_node/acpi_node)
    │
    └─► device_property_read_u32_array(dev, "reg", &val, 1)
          │
          └─► fwnode_property_read_u32_array(fwnode, "reg", &val, 1)
                │
                └─► fwnode_call_int_op(fwnode, property_read_int_array,
                                        "reg", sizeof(u32), &val, 1)
                      │
                      ├─► [OF backend]
                      │     of_fwnode_ops.property_read_int_array()
                      │       └─► of_property_read_u32_array()
                      │             └─► reads from DTB blob
                      │
                      ├─► [ACPI backend]
                      │     acpi_fwnode_ops.property_read_int_array()
                      │       └─► acpi_dev_prop_read()
                      │
                      └─► [swnode backend]
                            swnode_ops.property_read_int_array()
                              └─► يبحث في property_entry array
                                    └─► بيرجع القيمة من value.u32_data
```

#### إضافة software_node لجهاز موجود

```
driver init:
  device_add_software_node(dev, &my_node)
    │
    ├─► software_node_fwnode(&my_node)
    │     └─► يرجع الـ fwnode_handle المدمج في الـ swnode
    │
    ├─► set_secondary_fwnode(dev, swnode_fwnode)
    │     └─► dev->fwnode->secondary = swnode_fwnode
    │         (أو dev->fwnode = swnode_fwnode لو مفيش fwnode أصلي)
    │
    └─► device_add_software_node → returns 0 on success

الـ secondary chain:
  dev->fwnode (OF/ACPI node)
       └─► secondary → swnode fwnode
                           └─► secondary → NULL

لما ops تتدعى وتفشل على الأول، بتـ fallback على الـ secondary.
```

#### بناء graph والاتصال بين الأجهزة

```
[camera sensor DT node]          [ISP DT node]
  port@0/endpoint@0    ────────►   port@0/endpoint@0
    remote-endpoint = &isp_ep         remote-endpoint = &cam_ep

driver يقرأ الاتصال:
  fwnode_graph_get_next_endpoint(dev_fwnode(sensor_dev), NULL)
    └─► يرجع fwnode لـ endpoint@0
          │
          fwnode_graph_get_remote_endpoint(ep_fwnode)
            └─► يتبع remote-endpoint property
                  └─► يرجع fwnode لـ ISP endpoint
                        │
                        fwnode_graph_get_port_parent(remote_ep)
                          └─► يرجع fwnode لـ ISP device نفسه

fwnode_graph_parse_endpoint(ep_fwnode, &endpoint)
  └─► يملا: endpoint.port = 0
             endpoint.id   = 0
             endpoint.local_fwnode = ep_fwnode
```

#### البحث عن connection بالـ con_id

```
driver:
  device_connection_find_match(dev, "i2c", data, my_match_fn)
    │
    └─► fwnode_connection_find_match(dev_fwnode(dev), "i2c", data, fn)
          │
          ├─► يبحث في الـ fwnode properties عن "i2c-bus" أو مشابه
          │
          └─► فلو لقى match:
                fn(matched_fwnode, "i2c", data)
                  └─► driver-specific: مثلاً يرجع i2c_adapter*
```

---

### 5. استراتيجية الـ Locking

#### مفيش lock صريح في `property.h` — إيه الحكمة؟

الـ `property.h` API مبنية على مبدأ **read-mostly** — الـ properties بتتقرأ كتير وبتتكتب نادراً (أو مش بتتكتب خالص بعد الـ registration).

#### الـ locking حسب كل layer:

| الـ Layer | الـ Lock المستخدم | المحمي |
|---|---|---|
| **OF (Device Tree)** | `of_mutex` / `devtree_lock` (spinlock) | الـ DTB tree traversal |
| **ACPI** | `acpi_device_lock` (mutex) | الـ ACPI namespace |
| **swnode** | `swnode_root_ids` (ida) + `kobj_lock` | التسجيل والـ id allocation |
| **fwnode links** | `fwnode_links_lock` (mutex في device.c) | قوائم `suppliers`/`consumers` |
| **فـ الـ consumers/suppliers lists** | RCU أو device_links_lock | الـ devlink traversal |

#### قاعدة الـ refcount:

```
الـ fwnode_handle بيستخدم atomic refcount (عبر ops->get/ops->put)

لازم:
  - كل call لـ get_next_child_node / get_parent / graph_get_*
    بترجع fwnode بـ refcount مرفوع
  - لازم تعمل fwnode_handle_put() لما تخلص
  - الـ _scoped() macros بتعمل ده أوتوماتيك بـ __free(fwnode_handle)

ترتيب الـ locking (لو احتجت تمسك أكتر من lock):
  device_links_lock → fwnode_links_lock → swnode internal lock

لا تعمل:
  swnode internal lock → device_links_lock  ← deadlock!
```

#### الـ software_node registration — thread safety:

```c
/* software_node_register() محمية بـ: */
static DEFINE_MUTEX(software_nodes_lock);

/* بتحمي: */
/* - إضافة/حذف nodes من الـ global list */
/* - الـ parent/child relationships */

/* بعد الـ registration، الـ properties read-only → لا تحتاج lock */
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Group 1: Device Property Read (via `struct device`)

| Function | الغرض |
|---|---|
| `device_property_present` | هل الـ property موجودة؟ |
| `device_property_read_bool` | قراءة boolean property |
| `device_property_read_u8/u16/u32/u64` | قراءة قيمة scalar واحدة |
| `device_property_read_u8/u16/u32/u64_array` | قراءة array من القيم |
| `device_property_read_string` | قراءة string واحدة |
| `device_property_read_string_array` | قراءة array من الـ strings |
| `device_property_match_string` | البحث عن string في property |
| `device_property_match_property_string` | مقارنة property بـ array ثابت |
| `device_property_count_u8/u16/u32/u64` | عدد العناصر في property |
| `device_property_string_array_count` | عدد الـ strings في property |

#### Group 2: fwnode Property Read (via `fwnode_handle`)

| Function | الغرض |
|---|---|
| `fwnode_property_present` | هل الـ property موجودة؟ |
| `fwnode_property_read_bool` | قراءة boolean |
| `fwnode_property_read_u8/u16/u32/u64` | قراءة scalar واحدة |
| `fwnode_property_read_u8/u16/u32/u64_array` | قراءة array |
| `fwnode_property_read_string` | قراءة string |
| `fwnode_property_read_string_array` | قراءة string array |
| `fwnode_property_match_string` | بحث عن string |
| `fwnode_property_match_property_string` | مقارنة property بـ array |
| `fwnode_property_count_u8/u16/u32/u64` | عد العناصر |
| `fwnode_property_string_array_count` | عد الـ strings |

#### Group 3: fwnode Navigation

| Function | الغرض |
|---|---|
| `dev_fwnode` (macro) | الحصول على `fwnode_handle` من `device` |
| `fwnode_get_parent` | الـ parent node |
| `fwnode_get_next_parent` | التنقل للـ parent مع drop المرجع الحالي |
| `fwnode_count_parents` | عدد الـ parents |
| `fwnode_get_nth_parent` | الـ parent على عمق معين |
| `fwnode_get_next_child_node` | التنقل بين الـ children |
| `fwnode_get_next_available_child_node` | children المتاحة فقط |
| `fwnode_get_named_child_node` | child باسم معين |
| `fwnode_get_child_node_count` | عدد الـ children |
| `fwnode_get_named_child_node_count` | عدد الـ children بنفس الاسم |
| `fwnode_get_name` | اسم الـ node |
| `fwnode_get_name_prefix` | prefix الاسم (للطباعة) |
| `fwnode_name_eq` | مقارنة الاسم |

#### Group 4: fwnode Reference Management

| Function | الغرض |
|---|---|
| `fwnode_handle_get` | زيادة الـ refcount |
| `fwnode_handle_put` | تقليل الـ refcount |
| `fwnode_property_get_reference_args` | قراءة reference مع arguments |
| `fwnode_find_reference` | البحث عن reference بالـ index |

#### Group 5: fwnode Graph (V4L2 / media topology)

| Function | الغرض |
|---|---|
| `fwnode_graph_get_next_endpoint` | التنقل بين الـ endpoints |
| `fwnode_graph_get_port_parent` | الـ device الذي يحتوي الـ port |
| `fwnode_graph_get_remote_port_parent` | الـ device الطرف الآخر |
| `fwnode_graph_get_remote_port` | الـ port الطرف الآخر |
| `fwnode_graph_get_remote_endpoint` | الـ endpoint الطرف الآخر |
| `fwnode_graph_get_endpoint_by_id` | endpoint بـ port/id محددين |
| `fwnode_graph_get_endpoint_count` | عدد الـ endpoints |
| `fwnode_graph_parse_endpoint` | parse بيانات الـ endpoint |
| `fwnode_graph_is_endpoint` (inline) | هل الـ node هو endpoint؟ |

#### Group 6: Device Capabilities

| Function | الغرض |
|---|---|
| `fwnode_device_is_available` | هل الـ device متاح (status = okay)؟ |
| `fwnode_device_is_big_endian` (inline) | هل الـ device big-endian؟ |
| `fwnode_device_is_compatible` (inline) | هل compatible يطابق؟ |
| `device_is_big_endian` (inline) | wrapper لـ device |
| `device_is_compatible` (inline) | wrapper لـ device |
| `device_dma_supported` | هل DMA مدعومة؟ |
| `device_get_dma_attr` | نوع الـ DMA coherency |
| `device_get_match_data` | بيانات الـ driver match |
| `device_get_phy_mode` | PHY interface mode من الـ device |
| `fwnode_get_phy_mode` | PHY interface mode من الـ fwnode |
| `fwnode_iomap` | iomap للـ fwnode |
| `fwnode_irq_get` | IRQ number بالـ index |
| `fwnode_irq_get_byname` | IRQ number بالاسم |

#### Group 7: Connection Matching

| Function | الغرض |
|---|---|
| `fwnode_connection_find_match` | إيجاد أول match لـ connection |
| `device_connection_find_match` (inline) | wrapper لـ device |
| `fwnode_connection_find_matches` | إيجاد كل الـ matches |

#### Group 8: Software Nodes

| Function | الغرض |
|---|---|
| `is_software_node` | هل الـ fwnode هو software node؟ |
| `to_software_node` | cast من fwnode لـ software_node |
| `software_node_fwnode` | الحصول على fwnode من software_node |
| `software_node_find_by_name` | البحث عن software node باسم |
| `software_node_register` | تسجيل node واحد |
| `software_node_unregister` | إلغاء تسجيل node واحد |
| `software_node_register_node_group` | تسجيل مجموعة nodes |
| `software_node_unregister_node_group` | إلغاء تسجيل مجموعة |
| `fwnode_create_software_node` | إنشاء node ديناميكي |
| `fwnode_remove_software_node` | حذف node ديناميكي |
| `device_add_software_node` | ربط node بـ device |
| `device_remove_software_node` | فك ربط الـ node من الـ device |
| `device_create_managed_software_node` | إنشاء node مُدار بالـ devres |

#### Group 9: Property Entry Macros & Helpers

| Macro / Function | الغرض |
|---|---|
| `PROPERTY_ENTRY_U8/U16/U32/U64` | تعريف scalar property inline |
| `PROPERTY_ENTRY_STRING` | تعريف string property inline |
| `PROPERTY_ENTRY_BOOL` | تعريف boolean property |
| `PROPERTY_ENTRY_U8/U16/U32/U64_ARRAY` | تعريف array property |
| `PROPERTY_ENTRY_STRING_ARRAY` | تعريف string array property |
| `PROPERTY_ENTRY_REF` | تعريف reference property |
| `PROPERTY_ENTRY_REF_ARRAY` | تعريف reference array |
| `SOFTWARE_NODE_REFERENCE` | بناء `software_node_ref_args` |
| `property_entries_dup` | نسخ مصفوفة properties |
| `property_entries_free` | تحرير مصفوفة properties |

---

### Group 1: Device Property Read

**الغرض:** الـ `device_property_*` functions هي الـ high-level API التي يستخدمها الـ driver مباشرة. كلها تعمل كـ thin wrappers فوق نظيراتها الـ `fwnode_property_*`، وتحصل على الـ `fwnode_handle` من الـ `device` عبر `dev_fwnode(dev)` داخليًا.

---

#### `dev_fwnode` (macro)

```c
#define dev_fwnode(dev)   \
    _Generic((dev),       \
        const struct device *: __dev_fwnode_const, \
        struct device *: __dev_fwnode)(dev)
```

**الـ macro** يستخدم C11 `_Generic` لاختيار الـ const-correct version تلقائيًا. بيرجع `fwnode_handle *` أو `const fwnode_handle *` بناءً على نوع `dev`. النقطة الجوهرية هنا هي أن كل الـ property API يبدأ من هنا — الـ fwnode هو الـ abstraction layer اللي يعرف إزاي يتكلم مع DT أو ACPI أو software nodes.

---

#### `device_property_present`

```c
bool device_property_present(const struct device *dev, const char *propname);
```

بتبحث إذا كانت الـ property موجودة في الـ firmware description للـ device. بترجع `true` إذا الـ property موجودة بصرف النظر عن قيمتها. بتُستخدم قبل القراءة لتجنب error handling في بعض الحالات.

- **`dev`**: الـ device المراد الاستعلام عنه.
- **`propname`**: اسم الـ property (مثلاً `"interrupts"`, `"clock-frequency"`).
- **Return**: `true` إذا موجودة، `false` لو مش موجودة أو الـ fwnode نفسه NULL.
- **Caller context**: أي context، لا locking خاص، الـ fwnode backend يدير الـ locking الداخلي.

---

#### `device_property_read_bool`

```c
bool device_property_read_bool(const struct device *dev, const char *propname);
```

بتقرأ boolean property — يعني property موجودها وجودها هو القيمة نفسها (مثل `"big-endian"` في DT). بترجع `true` لو الـ property موجودة وقيمتها true، وإلا `false`.

- **Return**: الـ boolean value مباشرة.
- **Key detail**: في DT، الـ boolean property بتكون مجرد present/absent. في ACPI، ممكن تكون integer بقيمة 1/0.

---

#### `device_property_read_u8_array` / `u16` / `u32` / `u64`

```c
int device_property_read_u8_array(const struct device *dev,
                                  const char *propname,
                                  u8 *val, size_t nval);
```

بتقرأ array من integer values. لو `val == NULL` و `nval == 0`، بترجع عدد العناصر المتاحة بدل ما تقرأ (discovery mode). لو `val != NULL`، بتملي الـ buffer.

- **`val`**: buffer لاستقبال القيم، أو NULL لمعرفة العدد.
- **`nval`**: عدد العناصر المطلوبة، أو 0 للاستعلام.
- **Return**: 0 عند النجاح، عدد العناصر المتاحة لو `val == NULL`، أو error code سلبي (`-EINVAL`, `-ENODATA`, `-EOVERFLOW`).
- **Error paths**: `-EOVERFLOW` لو الـ buffer أصغر من عدد العناصر، `-ENODATA` لو الـ property غير موجودة.

---

#### `device_property_read_u8` / `u16` / `u32` / `u64` (inline wrappers)

```c
static inline int device_property_read_u32(const struct device *dev,
                                           const char *propname, u32 *val)
{
    return device_property_read_u32_array(dev, propname, val, 1);
}
```

الـ scalar versions مجرد wrappers تستدعي الـ array version بـ `nval=1`. الـ kernel prefer استخدام هذه بدلاً من الـ array version مباشرة للـ single-value properties.

---

#### `device_property_count_u8` / `u16` / `u32` / `u64` (inline)

```c
static inline int device_property_count_u32(const struct device *dev,
                                            const char *propname)
{
    return device_property_read_u32_array(dev, propname, NULL, 0);
}
```

بتستعمل الـ discovery mode (val=NULL, nval=0) لمعرفة عدد العناصر. بترجع العدد كـ positive int أو error code سلبي. مفيدة قبل الـ `kmalloc` لتخصيص buffer بالحجم الصح.

---

#### `device_property_read_string`

```c
int device_property_read_string(const struct device *dev,
                                const char *propname,
                                const char **val);
```

بتقرأ أول string من الـ property وبتحط pointer إليها في `*val`. الـ pointer بيشاور على الـ string داخل الـ firmware data مباشرة — مش copied — لذا لا تحرر الـ pointer.

- **Return**: 0 عند النجاح أو error سلبي.
- **Key detail**: الـ string lifetime مرتبط بالـ fwnode نفسه.

---

#### `device_property_match_string`

```c
int device_property_match_string(const struct device *dev,
                                 const char *propname,
                                 const char *string);
```

بتبحث عن `string` في كل عناصر string array property. بترجع الـ index (0-based) لو لقتها، أو error سلبي (`-ENODATA`, `-ENOENT`).

- **Real-world**: مستخدمة في `of_property_match_string` style lookups لـ clock names أو PHY modes.

---

#### `device_property_match_property_string` (inline)

```c
static inline int device_property_match_property_string(
    const struct device *dev, const char *propname,
    const char * const *array, size_t n);
```

بتقرأ الـ property كـ string وبتبحث فيها في array ثابت موجود في الـ driver code. عكس `match_string` اللي بيبحث في الـ firmware data، هنا الـ driver هو اللي عنده قائمة الـ valid values.

---

### Group 2: fwnode Property Read

**الغرض:** نفس الـ Group 1 تمامًا لكن بتشتغل على `fwnode_handle *` مباشرة بدل `struct device *`. دي الـ low-level API اللي الـ device variants تستدعيها. الـ fwnode backend (DT, ACPI, swnode) بيوفر الـ `fwnode_operations` vtable اللي بتنفذ الـ actual reads.

**pseudocode flow للـ fwnode_property_read_u32_array:**

```
fwnode_property_read_u32_array(fwnode, propname, val, nval):
    if fwnode has secondary:
        try primary fwnode first
        if failed, try secondary fwnode
    call fwnode->ops->property_read_int_array(fwnode, propname,
                                              sizeof(u32), val, nval)
    return result
```

الـ secondary fwnode مهم جدًا — بيسمح بالـ fallback chain (مثلاً ACPI + software node كـ overlay).

---

#### `fwnode_property_present`

```c
bool fwnode_property_present(const struct fwnode_handle *fwnode,
                             const char *propname);
```

بتستدعي `fwnode_call_bool_op(fwnode, property_present, propname)`. لو الـ fwnode عنده secondary، بتجرب الاتنين. بترجع `true` لو الـ property موجودة في أي منهم.

---

#### `fwnode_property_read_u32_array`

```c
int fwnode_property_read_u32_array(const struct fwnode_handle *fwnode,
                                   const char *propname, u32 *val,
                                   size_t nval);
```

الـ workhorse الأساسي. بتمرر `elem_size = sizeof(u32)` للـ vtable op. الـ backend (مثلاً `of_fwnode_ops`) عارف إزاي يفسر الـ binary data بناءً على الـ element size. لو `nval=0 && val=NULL`، بيرجع عدد العناصر.

---

### Group 3: fwnode Navigation

**الغرض:** الـ fwnode graph هو شجرة nodes — كل node ممكن يكون له parent وأبناء. الـ navigation functions دي بتسمح بالتنقل في هذه الشجرة بطريقة آمنة مع إدارة الـ refcounting.

---

#### `fwnode_get_parent`

```c
struct fwnode_handle *fwnode_get_parent(const struct fwnode_handle *fwnode);
```

بترجع reference جديدة للـ parent node (refcount مزوّد). المستدعي مسؤول عن استدعاء `fwnode_handle_put()` على النتيجة.

- **Return**: pointer للـ parent أو NULL لو الـ node هو الـ root.

---

#### `fwnode_get_next_parent`

```c
struct fwnode_handle *fwnode_get_next_parent(struct fwnode_handle *fwnode);
```

**مختلفة عن `get_parent`!** — بتـ drop الـ reference على `fwnode` المُمرَّر وبترجع reference على الـ parent. مصممة للاستخدام في iterator patterns. الـ macro `fwnode_for_each_parent_node` تعتمد عليها.

- **Side effect**: تحرير الـ `fwnode` المُمرَّر (drop refcount).
- **Key detail**: لو المستدعي عنده reference على الـ node الحالي، لازم `get` قبل ما يمرره لهذه الـ function.

---

#### `fwnode_get_nth_parent`

```c
struct fwnode_handle *fwnode_get_nth_parent(struct fwnode_handle *fwn,
                                            unsigned int depth);
```

بترجع الـ ancestor على عمق `depth` في الشجرة. `depth=0` بيرجع الـ node نفسه، `depth=1` بيرجع الـ parent، إلخ. بتستخدم داخليًا `fwnode_get_next_parent` في loop.

---

#### `fwnode_get_next_child_node`

```c
struct fwnode_handle *fwnode_get_next_child_node(
    const struct fwnode_handle *fwnode, struct fwnode_handle *child);
```

Iterator على الـ children. لو `child == NULL`، بترجع أول child. لو `child != NULL`، بترجع الـ next child بعده وبتـ drop reference على `child`. مبنية للاستخدام في `fwnode_for_each_child_node`.

---

#### `fwnode_get_next_available_child_node`

```c
struct fwnode_handle *fwnode_get_next_available_child_node(
    const struct fwnode_handle *fwnode, struct fwnode_handle *child);
```

نفس `get_next_child_node` لكن بتتخطى الـ nodes اللي `status != "okay"` أو مش available. مفيدة لـ drivers اللي تهمهم الـ hardware الـ active فقط.

---

#### `fwnode_get_named_child_node`

```c
struct fwnode_handle *fwnode_get_named_child_node(
    const struct fwnode_handle *fwnode, const char *childname);
```

بتبحث عن child بالاسم مباشرة. بترجع reference جديدة (refcount++) أو NULL.

---

#### Iterator Macros

```c
/* تنقل على كل الـ children */
#define fwnode_for_each_child_node(fwnode, child)
/* تنقل على الـ children المتاحة فقط */
#define fwnode_for_each_available_child_node(fwnode, child)
/* تنقل على children باسم محدد */
#define fwnode_for_each_named_child_node(fwnode, child, name)

/* scoped versions — تحرر الـ child تلقائيًا بنهاية الـ scope */
#define fwnode_for_each_child_node_scoped(fwnode, child)
#define fwnode_for_each_available_child_node_scoped(fwnode, child)
```

الـ **scoped** versions تستخدم `__free(fwnode_handle)` attribute (من `DEFINE_FREE` macro) لضمان تحرير الـ reference تلقائيًا لو الـ loop خرج بـ `break` أو `return`. دي أكثر أمانًا وتمنع الـ reference leaks.

**مهم:** في الـ non-scoped versions، لو خرجت من الـ loop بـ `break`، لازم تستدعي `fwnode_handle_put(child)` يدويًا.

---

#### `fwnode_count_parents` / `fwnode_get_child_node_count` / `fwnode_get_named_child_node_count`

```c
unsigned int fwnode_count_parents(const struct fwnode_handle *fwn);
unsigned int fwnode_get_child_node_count(const struct fwnode_handle *fwnode);
unsigned int fwnode_get_named_child_node_count(const struct fwnode_handle *fwnode,
                                               const char *name);
```

helper functions للحساب. بتتنقل في الشجرة وبتعد بدون تعديل. `get_named_child_node_count` مفيدة لمعرفة عدد الـ sub-nodes بنفس الاسم (مثلاً كل الـ `"port@N"` nodes).

---

### Group 4: fwnode Reference Management

**الغرض:** الـ fwnode هو reference-counted object. أي function بترجع fwnode pointer بترجع reference جديدة (+1 refcount). المستدعي ملزم بتحرير هذه الـ reference.

---

#### `fwnode_handle_get`

```c
struct fwnode_handle *fwnode_handle_get(struct fwnode_handle *fwnode);
```

بتزيد الـ refcount. بتستدعي `fwnode_call_ptr_op(fwnode, get)`. بترجع الـ fwnode نفسه (يسهل الـ chaining). آمنة على NULL.

---

#### `fwnode_handle_put` (inline)

```c
static inline void fwnode_handle_put(struct fwnode_handle *fwnode)
{
    fwnode_call_void_op(fwnode, put);
}
```

بتنقص الـ refcount. لو وصل صفر، الـ backend بيحرر الـ memory. الأكثر استخدامًا في الـ kernel. آمنة على NULL بسبب `fwnode_call_void_op` اللي بيتحقق من NULL.

```c
/* DEFINE_FREE تجعل التنظيف التلقائي ممكنًا */
DEFINE_FREE(fwnode_handle, struct fwnode_handle *, fwnode_handle_put(_T))
```

هذا يُسجّل cleanup function للاستخدام مع `__free(fwnode_handle)` في الـ scoped macros.

---

#### `fwnode_property_get_reference_args`

```c
int fwnode_property_get_reference_args(const struct fwnode_handle *fwnode,
                                       const char *prop,
                                       const char *nargs_prop,
                                       unsigned int nargs,
                                       unsigned int index,
                                       struct fwnode_reference_args *args);
```

بتقرأ reference property (مثل `clocks`, `interrupts`, `gpios`) وبتملي `fwnode_reference_args` بالـ fwnode المُشار إليه والـ arguments الإضافية.

- **`prop`**: اسم الـ property اللي فيها الـ references (مثل `"clocks"`).
- **`nargs_prop`**: اسم الـ property اللي بتحدد عدد الـ args (مثل `"#clock-cells"`) أو NULL.
- **`nargs`**: عدد ثابت من الـ args لو `nargs_prop == NULL`.
- **`index`**: index الـ reference المطلوبة (0-based) لو الـ property فيها أكثر من reference.
- **`args`**: struct يُملي بالنتيجة — `args->fwnode` هو الـ referenced node (برجع مع refcount++).
- **Return**: 0 أو error سلبي.
- **Caller**: المستدعي مسؤول عن `fwnode_handle_put(args->fwnode)`.

---

#### `fwnode_find_reference`

```c
struct fwnode_handle *fwnode_find_reference(const struct fwnode_handle *fwnode,
                                            const char *name,
                                            unsigned int index);
```

بتبحث عن reference property بالاسم والـ index. أبسط من `get_reference_args` لأنها ما بتجيبش الـ args. بترجع fwnode مع refcount++ أو ERR_PTR.

---

### Group 5: fwnode Graph (Media / V4L2 Topology)

**الغرض:** الـ fwnode graph API بتوفر abstraction للـ endpoint-based hardware topology المستخدمة في V4L2, MIPI CSI, DisplayPort وغيرها. كل device بتكون ليها `ports` وكل port بيكون فيه `endpoints` تتصل بـ endpoints في devices تانية.

```
                 [ Device A ]                    [ Device B ]
          port@0/endpoint@0  ◄───────────►  port@0/endpoint@0
```

---

#### `fwnode_graph_get_next_endpoint`

```c
struct fwnode_handle *fwnode_graph_get_next_endpoint(
    const struct fwnode_handle *fwnode, struct fwnode_handle *prev);
```

Iterator على endpoints. `prev == NULL` يبدأ من أول endpoint. بتـ drop reference على `prev` وبترجع reference على الـ next. بتتنقل عبر كل الـ ports وكل الـ endpoints.

---

#### `fwnode_graph_get_port_parent`

```c
struct fwnode_handle *fwnode_graph_get_port_parent(
    const struct fwnode_handle *fwnode);
```

بترجع الـ device fwnode اللي يحتوي الـ port. `fwnode` هنا هو endpoint أو port node.

---

#### `fwnode_graph_get_remote_port_parent`

```c
struct fwnode_handle *fwnode_graph_get_remote_port_parent(
    const struct fwnode_handle *fwnode);
```

من endpoint محلي، بتجيب الـ device الـ remote (الطرف الآخر من الوصلة). هذا هو الـ most commonly used function في media drivers لإيجاد الـ connected component.

---

#### `fwnode_graph_get_remote_endpoint`

```c
struct fwnode_handle *fwnode_graph_get_remote_endpoint(
    const struct fwnode_handle *fwnode);
```

من endpoint محلي، بترجع الـ remote endpoint (مش الـ device، الـ endpoint نفسه). بيستخدمها الـ driver لقراءة properties من الـ remote endpoint مباشرة.

---

#### `fwnode_graph_get_endpoint_by_id`

```c
struct fwnode_handle *fwnode_graph_get_endpoint_by_id(
    const struct fwnode_handle *fwnode,
    u32 port, u32 endpoint, unsigned long flags);
```

بتبحث عن endpoint بـ `port` number وـ `endpoint` ID محددين.

- **`flags`**: `FWNODE_GRAPH_ENDPOINT_NEXT` للبحث عن أقرب endpoint أكبر من الـ ID المطلوب. `FWNODE_GRAPH_DEVICE_DISABLED` لقبول endpoints من devices disabled.
- **Real-world**: بيستخدمها driver لإيجاد endpoint معين من DT node.

---

#### `fwnode_graph_parse_endpoint`

```c
int fwnode_graph_parse_endpoint(const struct fwnode_handle *fwnode,
                                struct fwnode_endpoint *endpoint);
```

بتملي `struct fwnode_endpoint` بالـ `port` number والـ `id`. بتستدعي الـ backend op مباشرة. بيستخدمها الـ graph framework داخليًا.

---

#### `fwnode_graph_is_endpoint` (inline)

```c
static inline bool fwnode_graph_is_endpoint(const struct fwnode_handle *fwnode)
{
    return fwnode_property_present(fwnode, "remote-endpoint");
}
```

بتتحقق إذا كان الـ node هو endpoint بالبحث عن property `"remote-endpoint"`. بسيطة وفعالة.

---

### Group 6: Device Capabilities

---

#### `fwnode_device_is_available`

```c
bool fwnode_device_is_available(const struct fwnode_handle *fwnode);
```

بتتحقق من الـ `status` property. في DT، `status = "okay"` أو غياب الـ status يعني available. في ACPI، بتتحقق من الـ `_STA` method. مستخدمة داخليًا في `get_next_available_child_node`.

---

#### `fwnode_device_is_big_endian` (inline)

```c
static inline bool fwnode_device_is_big_endian(const struct fwnode_handle *fwnode)
{
    if (fwnode_property_present(fwnode, "big-endian"))
        return true;
    if (IS_ENABLED(CONFIG_CPU_BIG_ENDIAN) &&
        fwnode_property_present(fwnode, "native-endian"))
        return true;
    return false;
}
```

بتتحقق من اتنين حالات: `"big-endian"` يعني BE دايمًا. `"native-endian"` يعني BE بس لو الـ CPU نفسه BE. الـ driver بيستخدمها لاختيار `ioread32be` vs `readl`.

---

#### `device_dma_supported` / `device_get_dma_attr`

```c
bool device_dma_supported(const struct device *dev);
enum dev_dma_attr device_get_dma_attr(const struct device *dev);
```

بتستعلم عن DMA capabilities من الـ firmware. `get_dma_attr` بترجع واحد من:
- `DEV_DMA_NOT_SUPPORTED`
- `DEV_DMA_NON_COHERENT`
- `DEV_DMA_COHERENT`

---

#### `device_get_match_data`

```c
const void *device_get_match_data(const struct device *dev);
```

بترجع الـ `data` field من الـ matching entry في `of_device_id` أو `acpi_device_id`. بتستخدمها الـ drivers بدل ما يعملوا custom matching. بيرجع NULL لو ما فيش match.

---

#### `device_get_phy_mode` / `fwnode_get_phy_mode`

```c
int device_get_phy_mode(struct device *dev);
int fwnode_get_phy_mode(const struct fwnode_handle *fwnode);
```

بتقرأ `"phy-mode"` أو `"phy-connection-type"` property وبترجع `phy_interface_t` enum value. بتستخدمها الـ Ethernet/network drivers لتحديد الـ PHY interface (RGMII, SGMII, MII, إلخ). بترجع error سلبي لو الـ mode مش معروف.

---

#### `fwnode_iomap`

```c
void __iomem *fwnode_iomap(struct fwnode_handle *fwnode, int index);
```

بتعمل iomap للـ memory region الـ `index` المحددة في الـ fwnode. بتستدعي الـ backend op. بترجع NULL لو الـ op مش مدعومة. بتستخدمها devices في ACPI world بشكل رئيسي.

---

#### `fwnode_irq_get` / `fwnode_irq_get_byname`

```c
int fwnode_irq_get(const struct fwnode_handle *fwnode, unsigned int index);
int fwnode_irq_get_byname(const struct fwnode_handle *fwnode, const char *name);
```

بيجيبوا الـ Linux virtual IRQ number. `by_index` عن طريق رقم ترتيبي. `byname` بيبحث في `"interrupt-names"` property. بيرجعوا الـ IRQ number (양موجب) أو error سلبي.

---

### Group 7: Connection Matching

**الغرض:** الـ connection API بيسمح للـ drivers بالبحث عن connections لـ device مع devices تانية (مثلاً sensor متصل بـ camera controller) بطريقة firmware-agnostic.

---

#### `fwnode_connection_find_match`

```c
void *fwnode_connection_find_match(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match);
```

بتبحث عن أول connection تطابق الـ `con_id` وبتستدعي الـ `match` callback على كل candidate. الـ `match` function بتاخد `(fwnode, id, data)` وبترجع non-NULL لو لقت match.

```c
typedef void *(*devcon_match_fn_t)(const struct fwnode_handle *fwnode,
                                   const char *id, void *data);
```

- **`con_id`**: connection identifier (مثل `"i2c"`, `"spi"`).
- **`data`**: arbitrary data يتمرر للـ match callback.
- **Return**: نتيجة الـ match callback أو NULL.

---

#### `fwnode_connection_find_matches`

```c
int fwnode_connection_find_matches(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match,
                                   void **matches, unsigned int matches_len);
```

نفس السابقة لكن بتجمع كل الـ matches في array. `matches` buffer بيُملي بنتائج الـ match callbacks. `matches_len` هو حجم الـ buffer.

- **Return**: عدد الـ matches الفعلية أو error سلبي.

---

### Group 8: Software Nodes

**الغرض:** الـ software nodes بتحل مشكلة الـ hardware اللي ما عندهاش firmware description كاملة. بدل ما تحتاج DT node أو ACPI table، الـ driver أو platform code بيعرّف properties في الـ C code مباشرة كـ static data أو dynamic allocation، وبيضيفها على الـ device.

```
struct software_node {
    const char *name;
    const struct software_node *parent;   /* شجرة nodes */
    const struct property_entry *properties;
};
```

---

#### `is_software_node` / `to_software_node` / `software_node_fwnode`

```c
bool is_software_node(const struct fwnode_handle *fwnode);
const struct software_node *to_software_node(const struct fwnode_handle *fwnode);
struct fwnode_handle *software_node_fwnode(const struct software_node *node);
```

الـ type checking والـ casting utilities. `to_software_node` بيستخدم `container_of` داخليًا. `software_node_fwnode` بيرجع الـ embedded fwnode_handle.

---

#### `software_node_find_by_name`

```c
const struct software_node *software_node_find_by_name(
    const struct software_node *parent, const char *name);
```

بتبحث في الـ registered software nodes عن node باسم معين تحت parent معين. `parent == NULL` بيبحث في الـ root level.

---

#### `software_node_register` / `software_node_unregister`

```c
int software_node_register(const struct software_node *node);
void software_node_unregister(const struct software_node *node);
```

بيسجلوا/بيلغوا تسجيل node واحد في الـ global software node registry. الـ node لازم يكون `static` أو مضمون إنه موجود طول فترة التسجيل. يُستخدم للـ nodes الـ static.

---

#### `software_node_register_node_group` / `software_node_unregister_node_group`

```c
int software_node_register_node_group(const struct software_node * const *node_group);
void software_node_unregister_node_group(const struct software_node * const *node_group);
```

بيسجلوا/بيلغوا مجموعة كاملة من الـ nodes في ضربة واحدة. الـ `node_group` هو NULL-terminated array من pointers. الـ register بيتوقف على أول error وبيـ unregister اللي سجله قبله.

---

#### `fwnode_create_software_node`

```c
struct fwnode_handle *fwnode_create_software_node(
    const struct property_entry *properties,
    const struct fwnode_handle *parent);
```

بتخلق software node ديناميكي (مع `kmalloc` داخليًا). مفيدة لما تحتاج تنشئ node في runtime بدون static struct. الـ properties بتتنسخ (`property_entries_dup` داخليًا).

- **Return**: fwnode مع refcount=1 أو ERR_PTR.
- **Caller**: لازم يستدعي `fwnode_remove_software_node` لما يخلص.

---

#### `fwnode_remove_software_node`

```c
void fwnode_remove_software_node(struct fwnode_handle *fwnode);
```

بتحرر الـ dynamically created software node. بتستدعي `fwnode_handle_put` اللي لما الـ refcount يوصل صفر بتعمل `kfree`.

---

#### `device_add_software_node`

```c
int device_add_software_node(struct device *dev,
                             const struct software_node *node);
```

بتربط software node بـ device. بتحط الـ fwnode كـ secondary للـ existing device fwnode. بعدها، الـ `device_property_*` calls على هذا الـ device هتلاقي properties من الـ software node.

- **Locking**: يُستدعى قبل `device_add()` عادةً.
- **Return**: 0 أو error سلبي.

---

#### `device_remove_software_node`

```c
void device_remove_software_node(struct device *dev);
```

بتفك ربط الـ software node من الـ device وبتنزل الـ reference. لازم يتستدعى في cleanup.

---

#### `device_create_managed_software_node`

```c
int device_create_managed_software_node(struct device *dev,
                                        const struct property_entry *properties,
                                        const struct software_node *parent);
```

بتجمع `fwnode_create_software_node` + `device_add_software_node` في خطوة واحدة، مع **devres management**. يعني لما الـ device بتـ unbind أو بتتحذف، الـ node بيتحرر تلقائيًا. الأسهل والأكثر أمانًا في الـ drivers الحديثة.

---

### Group 9: Property Entry Macros

**الغرض:** الـ `struct property_entry` بيمثل property واحدة في الـ software node. الـ macros بتسهّل إنشاء الـ static arrays من هذه الـ properties.

```c
struct property_entry {
    const char *name;
    size_t length;
    bool is_inline;          /* القيمة مخزنة inline في الـ struct نفسه */
    enum dev_prop_type type;
    union {
        const void *pointer; /* للـ arrays والـ external data */
        union {
            u8  u8_data[8];
            u16 u16_data[4];
            u32 u32_data[2];
            u64 u64_data[1];
            const char *str[...];
        } value;             /* للـ inline scalars */
    };
};
```

---

#### Scalar Inline Macros

```c
PROPERTY_ENTRY_U8(_name_, _val_)
PROPERTY_ENTRY_U16(_name_, _val_)
PROPERTY_ENTRY_U32(_name_, _val_)
PROPERTY_ENTRY_U64(_name_, _val_)
PROPERTY_ENTRY_STRING(_name_, _val_)
```

بيخزنوا القيمة مباشرة في الـ `value` union (is_inline=true). ما فيش dynamic allocation. مثال:

```c
static const struct property_entry my_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("label", "my-device"),
    PROPERTY_ENTRY_BOOL("wakeup-source"),
    { }  /* sentinel */
};
```

---

#### Boolean Macro

```c
#define PROPERTY_ENTRY_BOOL(_name_)     \
(struct property_entry) {               \
    .name = _name_,                     \
    .is_inline = true,                  \
}
```

بـ `is_inline=true` وبدون type محدد، الـ framework بيعاملها كـ boolean present property.

---

#### Array Macros

```c
PROPERTY_ENTRY_U32_ARRAY(_name_, _val_)        /* حجم تلقائي بـ ARRAY_SIZE */
PROPERTY_ENTRY_U32_ARRAY_LEN(_name_, _val_, _len_)  /* حجم يدوي */
```

بيستخدموا `pointer` field بدل `value` — بيشاوروا على الـ array الخارجية. الـ array لازم تكون static أو موجودة طول عمر الـ property. `property_entries_dup` بتعمل deep copy لو احتجت.

---

#### Reference Macros

```c
#define PROPERTY_ENTRY_REF(_name_, _ref_, ...)
#define PROPERTY_ENTRY_REF_ARRAY(_name_, _val_)
```

بيخلقوا reference property تشاور على `software_node` أو `fwnode_handle`. بيستخدم `SOFTWARE_NODE_REFERENCE` macro داخليًا.

```c
/* مثال: device-a بيتوصل بـ device-b */
static const struct software_node dev_b_node = { .name = "device-b", ... };
static const struct property_entry dev_a_props[] = {
    PROPERTY_ENTRY_REF("connected-to", &dev_b_node),
    { }
};
```

---

#### `property_entries_dup` / `property_entries_free`

```c
struct property_entry *property_entries_dup(const struct property_entry *properties);
void property_entries_free(const struct property_entry *properties);
```

`property_entries_dup` بتعمل deep copy للـ array — بتنسخ الـ names والـ data والـ string arrays كلها. بترجع pointer للـ copy الجديدة أو ERR_PTR.

`property_entries_free` بتحرر الـ copy الناتجة عن `dup`. **لا تستدعيها على static arrays.**

- **Caller**: `fwnode_create_software_node` بتستدعي `dup` داخليًا، و`fwnode_remove_software_node` بتستدعي `free` تلقائيًا.

---

### ملاحظات Architectural مهمة

1. **الـ vtable pattern**: كل الـ fwnode operations بتمر بـ `fwnode_operations` vtable. الـ `fwnode_call_*_op` macros بتتحقق من NULL fwnode وغياب الـ op قبل الاستدعاء.

2. **الـ secondary fwnode**: الـ `fwnode_handle.secondary` بيسمح بـ chaining — لو الـ primary فشل في البحث عن property، بيجرب الـ secondary. هذا هو الآلية اللي بتخلي software nodes تعمل كـ overlay فوق DT/ACPI nodes.

3. **الـ refcounting**: كل function بترجع `fwnode_handle *` بترجع reference جديدة (+1). المستدعي لازم يعمل `fwnode_handle_put` في كل code paths. الـ scoped macros بتتعامل مع هذا تلقائيًا.

4. **الـ NULL safety**: الـ `fwnode_call_*_op` macros آمنة على NULL fwnode — بترجع error مناسب بدل crash.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بندرسه هو **Unified Device Property Interface** — الكود في `include/linux/property.h` وتنفيذه في `drivers/base/property.c` و`drivers/base/swnode.c`. ده الـ layer اللي بيوحد الوصول للـ device properties سواء من Device Tree أو ACPI أو Software Nodes.

---

### Software Level

#### 1. مداخل الـ debugfs

الـ property subsystem نفسه مش ليه debugfs entries مخصصة، بس الـ fwnode graph والـ software nodes ممكن نتتبعهم من خلال:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف device links المرتبطة بالـ fwnode
ls /sys/kernel/debug/device_component/

# firmware node graph (لو driver بيستخدم graph API)
ls /sys/kernel/debug/
```

| debugfs Entry | الـ Content | طريقة القراءة |
|---|---|---|
| `/sys/kernel/debug/device_component/` | device component matching | `cat /sys/kernel/debug/device_component/<dev>` |
| `/sys/kernel/debug/acpi/` | ACPI namespace nodes (ACPI backend) | `cat /sys/kernel/debug/acpi/<path>/properties` |
| `/sys/kernel/debug/of_dump` | OF/DT node dump (DT backend) | يستلزم `CONFIG_OF_DEBUG` |

#### 2. مداخل الـ sysfs

```bash
# إقرأ الـ firmware node path لأي device
cat /sys/devices/<path>/firmware_node/path 2>/dev/null

# DMA support
cat /sys/devices/<path>/dma_mask_bits 2>/dev/null

# properties من OF backend
ls /sys/firmware/devicetree/base/<node>/
cat /sys/firmware/devicetree/base/<node>/<prop_name>

# ACPI backend
ls /sys/firmware/acpi/tables/
cat /sys/bus/acpi/devices/<HID>/properties 2>/dev/null

# software nodes المسجلة
ls /sys/firmware/swnode/ 2>/dev/null
```

**مثال عملي** — اقرأ property من DT node مباشرة:

```bash
# لو عندك node اسمه /soc/i2c@ff160000 وعنده property clock-frequency
xxd /sys/firmware/devicetree/base/soc/i2c@ff160000/clock-frequency
# output: 00000000: 0006 1a80  ← big-endian u32 = 400000 Hz = 400 KHz
```

#### 3. الـ ftrace — Tracepoints وEvents

الـ property API مش ليها tracepoints built-in، بس نقدر نتتبع الـ function calls مباشرة:

```bash
# شغّل function tracer على device_property_read_u32
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo 'device_property_read_u32_array' > set_ftrace_filter
echo 'fwnode_property_read_u32_array' >> set_ftrace_filter
echo 'fwnode_property_present' >> set_ftrace_filter
echo 'software_node_register' >> set_ftrace_filter
echo 1 > tracing_on
# شغّل الـ driver
echo 0 > tracing_on
cat trace
```

```bash
# تتبع كل الـ fwnode operations باستخدام function_graph
echo function_graph > current_tracer
echo 'fwnode_*' > set_ftrace_filter
echo 'device_property_*' >> set_ftrace_filter
echo 1 > tracing_on
modprobe <your_driver>
echo 0 > tracing_on
cat trace | head -100
```

**مثال على output متوقع:**

```
# tracer: function_graph
 0) | fwnode_property_present() {
 0) | fwnode_call_bool_op() {
 0)   0.312 us    | of_fwnode_property_present();
 0)   1.105 us    | }
 0)   1.890 us    | }
```

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لـ property subsystem
echo 'file drivers/base/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/swnode.c +p' >> /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل drivers/base
echo 'file drivers/base/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي مفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep property

# لو محتاج OF/DT debug
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/base.c +p' >> /sys/kernel/debug/dynamic_debug/control

# ACPI property debug
echo 'file drivers/acpi/property.c +p' > /sys/kernel/debug/dynamic_debug/control
```

**من الـ kernel command line** لتفعيل debug بدري في الـ boot:

```bash
# في /etc/default/grub أو kernel cmdline
dyndbg="file drivers/base/property.c +p; file drivers/base/swnode.c +p"
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض | متى تفعّله |
|---|---|---|
| `CONFIG_OF` | دعم Device Tree | دايماً على embedded |
| `CONFIG_OF_DYNAMIC` | إضافة/حذف DT nodes runtime | debugging dynamic DT |
| `CONFIG_ACPI_DEBUG` | ACPI debug messages | ACPI property issues |
| `CONFIG_ACPI_DEBUGGER` | ACPI interactive debugger | ACPI namespace problems |
| `CONFIG_DEBUG_DRIVER` | driver core debug messages | device/fwnode binding |
| `CONFIG_PROVE_LOCKING` | detect locking bugs | reference count races |
| `CONFIG_KASAN` | detect use-after-free | fwnode_handle_put bugs |
| `CONFIG_UBSAN` | detect undefined behavior | property type mismatches |
| `CONFIG_DEBUG_OBJECTS` | track object lifecycles | fwnode reference leaks |
| `CONFIG_DEBUG_OBJECTS_WORK` | track work_struct objects | deferred probe issues |
| `CONFIG_SOFTIRQ_REENTRANT` | reentrancy checks | — |
| `CONFIG_FWNODE_MDIO` | MDIO fwnode support | networking property debug |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(OF|ACPI_DEBUG|DEBUG_DRIVER|KASAN)'
# أو
grep -E 'CONFIG_(OF|ACPI_DEBUG|DEBUG_DRIVER)' /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

**أداة `dtc` لقراءة الـ Device Tree:**

```bash
# فك compile الـ DTB الحالي
dtc -I dtb -O dts -o /tmp/current.dts /sys/firmware/fdt 2>/dev/null
# أو
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | head -200

# اقرأ node معين
cat /sys/firmware/devicetree/base/compatible | tr '\0' '\n'
```

**أداة `acpidump` للـ ACPI:**

```bash
# dump كل ACPI tables
acpidump -o /tmp/acpi.dat

# استخرج الـ DSDT
acpidump -n DSDT | acpixtract -a
iasl -d dsdt.dat
# ابحث في الـ DSDT المفكوك عن properties
grep -i 'DSD\|_DSD' dsdt.dsl
```

**فحص الـ software nodes:**

```bash
# شوف الـ software nodes المسجلة عبر sysfs
find /sys -name "software_node" -type d 2>/dev/null
# أو ابحث في dmesg
dmesg | grep -i 'software.node\|swnode'
```

**الـ `devlink` مش مرتبط بالـ property API مباشرة، بس لو بتعمل debug لـ net device properties:**

```bash
devlink dev info
devlink dev param show
```

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `fwnode_property_read ... -EINVAL` | الـ fwnode نفسه NULL أو invalid | تحقق من `dev_fwnode(dev)` إنه مش NULL قبل الاستخدام |
| `fwnode_property_read ... -ENXIO` | الـ backend مش عنده الـ op ده | الـ fwnode type مش بيدعم العملية دي |
| `fwnode_property_read ... -ENODATA` | الـ property موجودة بس فاضية | تحقق من DT/ACPI/swnode تعريف الـ property |
| `fwnode_property_read ... -EPROTO` | type mismatch — قرأت u32 من string | غيّر الـ function للنوع الصح |
| `fwnode_property_read ... -EOVERFLOW` | الـ array أصغر من عدد العناصر | زوّد `nval` أو استخدم `count` أولاً |
| `software_node_register: -EEXIST` | الـ node مسجّل قبل كده | تأكد من unregister قبل re-register |
| `device_add_software_node: -EBUSY` | الـ device عنده fwnode بالفعل | استخدم `device_remove_software_node` الأول |
| `ERROR: Bad cell-index ...` | الـ phandle reference index غلط | تحقق من `nargs_prop` في DT |
| `fwnode_graph_get_endpoint_by_id: invalid endpoint` | port/endpoint IDs مش موجودين | تحقق من الـ DT graph bindings |
| `fwnode_handle reference count leak` | `fwnode_handle_put` مش اتعمل | استخدم `fwnode_for_each_*_scoped` أو `__free(fwnode_handle)` |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في driver يقرأ property — تحقق قبل الاستخدام */
static int my_driver_probe(struct platform_device *pdev)
{
    struct fwnode_handle *fwnode = dev_fwnode(&pdev->dev);

    /* نقطة 1: تحقق من وجود الـ fwnode */
    if (WARN_ON(!fwnode)) {
        dev_err(&pdev->dev, "no firmware node!\n");
        return -EINVAL;
    }

    u32 val;
    int ret = device_property_read_u32(&pdev->dev, "clock-frequency", &val);

    /* نقطة 2: log القيمة المقروءة للتحقق */
    dev_dbg(&pdev->dev, "clock-frequency = %u (ret=%d)\n", val, ret);

    /* نقطة 3: تحقق من child nodes */
    struct fwnode_handle *child;
    unsigned int count = fwnode_get_child_node_count(fwnode);
    dev_dbg(&pdev->dev, "child node count = %u\n", count);

    fwnode_for_each_child_node(fwnode, child) {
        /* نقطة 4: تحقق من الـ child اسمه إيه */
        dev_dbg(&pdev->dev, "child: %s available=%d\n",
                fwnode_get_name(child),
                fwnode_device_is_available(child));

        /* نقطة 5: WARN لو الـ child مش available وإنت متوقع يكون */
        WARN_ON(!fwnode_device_is_available(child));
    }
    return 0;
}

/* نقطة 6: في cleanup — تحقق من reference count */
static void my_cleanup(struct fwnode_handle *fwnode)
{
    WARN_ON(IS_ERR_OR_NULL(fwnode));
    fwnode_handle_put(fwnode);  /* لازم يتعمل دايماً */
}
```

**نقاط dump_stack() المفيدة:**

```c
/* لو فيه software_node register/unregister race */
int ret = software_node_register(node);
if (ret) {
    pr_err("swnode register failed: %d\n", ret);
    dump_stack();  /* يكشف من اللي استدعى الـ register */
}

/* لو fwnode_graph endpoint مش لاقي remote */
struct fwnode_handle *remote = fwnode_graph_get_remote_endpoint(ep);
if (!remote) {
    dev_warn(dev, "no remote endpoint for port %u\n", ep_data.port);
    dump_stack();  /* لتتبع من طلب الـ endpoint */
}
```

---

### Hardware Level

#### 1. التحقق من إن الـ Hardware State بيتطابق مع الـ Kernel State

الـ property API بيوفر الـ metadata بس، المشكلة دايماً بين القيمة اللي في الـ DT/ACPI وإيه اللي على الـ hardware فعلاً.

```bash
# تحقق من compatible string يتطابق مع اللي على الـ silicon
cat /sys/firmware/devicetree/base/<node>/compatible | tr '\0' '\n'
# قارن بالـ datasheet part number

# تحقق من reg property (base address) يتطابق مع الـ hardware manual
xxd /sys/firmware/devicetree/base/<node>/reg
# output: big-endian address + size

# تحقق إن الـ clock-frequency property بتتطابق مع إيه اللي الـ hardware بيشتغل عليه
cat /sys/firmware/devicetree/base/<node>/clock-frequency | od -An -tx4
```

**مثال: I2C controller** — قارن الـ property مع الـ hardware:

```bash
# من DT
xxd /sys/firmware/devicetree/base/soc/i2c@ff160000/clock-frequency
# من kernel (ما تم read فعلاً)
dmesg | grep -i 'i2c.*ff160000\|ff160000.*i2c'
# من hardware register (SCL frequency divider register)
devmem2 0xff160014 w  # مثلاً CLKDIV register
```

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 — قراءة register واحد
devmem2 <phys_addr> [b|h|w|q]

# مثال: اقرأ 4 registers بدءاً من 0xFF160000
for offset in 0 4 8 c; do
    addr=$((0xFF160000 + 0x$offset))
    printf "0x%08X: " $addr
    devmem2 $addr w 2>/dev/null | grep -o '0x[0-9A-Fa-f]*$'
done

# /dev/mem مع dd (لـ range كامل)
dd if=/dev/mem bs=4 count=64 skip=$((0xFF160000/4)) 2>/dev/null | xxd

# io utility (من package ioport)
io -4 -r 0xFF160000

# من داخل kernel module — ioread32 بعد ioremap
void __iomem *base = ioremap(0xFF160000, 0x1000);
dev_info(dev, "REG0=0x%08x\n", ioread32(base));
iounmap(base);
```

**مثال على استخدام `fwnode_iomap`:**

```c
/* الـ API نفسه بيوفر iomap مع الـ fwnode */
void __iomem *regs = fwnode_iomap(fwnode, 0);  /* index 0 = أول reg range */
if (!regs) {
    dev_err(dev, "iomap failed — check 'reg' property in DT\n");
    return -ENOMEM;
}
u32 val = ioread32(regs);
dev_info(dev, "first register = 0x%08x\n", val);
iounmap(regs);
```

#### 3. Logic Analyzer وOscilloscope

**للـ I2C devices** اللي بتستخدم property API:

```
Oscilloscope tips:
─────────────────
- قيس الـ SCL frequency وقارنها بـ "clock-frequency" property
- لو الـ property بتقول 400KHz والـ scope بيقيس 100KHz → غلط في الـ divider register
- ابحث عن clock stretching مفرط → يدل على mismatch في timing properties

Logic Analyzer (مثلاً Saleae):
──────────────────────────────
- الـ "interrupts" property في DT → تأكد الـ IRQ line بيطلع على الـ trigger المحدد (edge/level)
- فعّل الـ protocol decoder المناسب (I2C/SPI/UART) وقارن الـ timing بالـ properties
```

**ASCII diagram — مقارنة property مع hardware:**

```
DT property: clock-frequency = <400000>
                │
                ▼
    ┌─────────────────────┐
    │  driver reads val   │  device_property_read_u32(dev,"clock-frequency",&hz)
    │  hz = 400000        │
    └────────┬────────────┘
             │ calculates divider
             ▼
    ┌─────────────────────┐
    │  hardware register  │  CLKDIV = APB_CLK / (5 * desired_freq)
    │  write to HW        │
    └────────┬────────────┘
             │
             ▼
    ┌─────────────────────┐
    │   oscilloscope      │  measure actual SCL frequency
    │   measure SCL       │  should match 400KHz ± tolerance
    └─────────────────────┘
```

#### 4. Common Hardware Issues وكيف تظهر في الـ Kernel Log

| المشكلة الـ Hardware | الـ Kernel Log Pattern | الـ Root Cause |
|---|---|---|
| الـ `reg` property خطأ (address غلط) | `ioremap: invalid phys address` أو `fwnode_iomap failed` | الـ DT/ACPI reg range مش بيوافق الـ hardware manual |
| الـ IRQ number خطأ | `irq: type mismatch, failed to map` | `interrupts` property في DT غلط |
| الـ `compatible` مش متطابق مع الـ driver | `Driver 'xxx' was not added` أو device مش probe | نسخة خاطئة من `compatible` string |
| `status = "disabled"` في DT بالغلط | Device مش بيظهر في `/sys/bus/` | فعّل الـ node بـ `status = "okay"` |
| مشكلة endianness — property big-endian بس driver بيقرأ كـ LE | قيم عشوائية خاطئة | استخدم `fwnode_device_is_big_endian()` واضبط الـ accessors |
| phandle reference لـ non-existent node | `fwnode_property_get_reference_args: -EINVAL` | الـ node المشار إليه مش موجود في DT |
| clock-frequency خارج نطاق الـ hardware | `clk: set rate failed` في الـ clk driver | راجع الـ hardware limits في الـ datasheet |

#### 5. Device Tree Debugging

```bash
# 1. تحقق إن الـ DTB اتحمل صح
ls -la /sys/firmware/fdt  # لازم يكون موجود
md5sum /sys/firmware/fdt   # قارن مع الـ dtb file

# 2. فك compile الـ DTB الحي وقارنه بالـ source
dtc -I dtb -O dts /sys/firmware/fdt -o /tmp/live.dts 2>/dev/null
diff /tmp/live.dts /path/to/source.dts | head -50

# 3. تحقق من overlays المطبّقة
ls /sys/firmware/devicetree/overlays/ 2>/dev/null

# 4. تحقق من node معين موجود وخصائصه صح
# الـ 'status' property
cat /sys/firmware/devicetree/base/<path>/status 2>/dev/null || echo "no status (ok by default)"

# 5. اقرأ كل properties لـ node
for f in /sys/firmware/devicetree/base/<node>/*; do
    name=$(basename $f)
    printf "%-30s: " "$name"
    xxd $f 2>/dev/null | head -1 || echo "(dir)"
done

# 6. تحقق من compatible يتطابق مع driver table
cat /sys/firmware/devicetree/base/<node>/compatible | tr '\0' '\n'
# قارن بالـ of_device_id table في source الـ driver

# 7. تحقق من phandle references
xxd /sys/firmware/devicetree/base/<node>/clocks  # يرجع phandle (u32 big-endian)
# قيمة الـ phandle ابحث عنها في
grep -r "phandle" /sys/firmware/devicetree/base/ 2>/dev/null | head

# 8. لو بتستخدم software_node بدل DT — تحقق من التسجيل
dmesg | grep -E 'swnode|software.node'
```

**مثال تحقق من `fwnode_device_is_available`:**

```bash
# لو device مش بيظهر رغم وجود الـ node في DT
python3 -c "
import os
node = '/sys/firmware/devicetree/base/soc/i2c@ff160000'
status_file = os.path.join(node, 'status')
if os.path.exists(status_file):
    val = open(status_file,'rb').read().rstrip(b'\x00').decode()
    print(f'status = {repr(val)}')
else:
    print('no status property → device is available (okay by default)')
"
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. اقرأ كل properties لأي device عبر sysfs:**

```bash
#!/bin/bash
# usage: ./read_props.sh <device_path>
# example: ./read_props.sh /sys/firmware/devicetree/base/soc/i2c@ff160000
NODE="${1:-/sys/firmware/devicetree/base}"
for f in "$NODE"/*; do
    [ -f "$f" ] || continue
    name=$(basename "$f")
    size=$(wc -c < "$f")
    printf "%-35s [%3d bytes]: " "$name" "$size"
    if [ "$size" -le 64 ]; then
        xxd "$f" | head -2
    else
        xxd "$f" | head -1
        echo "  ..."
    fi
done
```

**2. فعّل كل dynamic debug للـ property subsystem:**

```bash
#!/bin/bash
set -e
DBG=/sys/kernel/debug/dynamic_debug/control
for f in \
    "file drivers/base/property.c +p" \
    "file drivers/base/swnode.c +p" \
    "file drivers/of/property.c +p" \
    "file drivers/acpi/property.c +p" \
    "file drivers/base/core.c +p"
do
    echo "$f" > "$DBG" 2>/dev/null && echo "OK: $f" || echo "SKIP: $f"
done
echo "Dynamic debug enabled. Watch: dmesg -w"
```

**3. ftrace لـ property API:**

```bash
#!/bin/bash
TDIR=/sys/kernel/debug/tracing
echo 0 > $TDIR/tracing_on
echo function_graph > $TDIR/current_tracer
cat > $TDIR/set_ftrace_filter << 'EOF'
device_property_read_u32_array
device_property_read_string
device_property_present
fwnode_property_present
fwnode_property_read_u32_array
software_node_register
software_node_unregister
device_add_software_node
device_remove_software_node
fwnode_graph_get_next_endpoint
fwnode_graph_parse_endpoint
EOF
echo 1 > $TDIR/tracing_on
echo "Tracing active. Run: cat $TDIR/trace_pipe"
echo "Stop with: echo 0 > $TDIR/tracing_on"
```

**4. تحقق من software nodes المسجلة:**

```bash
# كل kernel objects المرتبطة بـ swnode
ls /sys/kernel/ | grep -i node

# دور على software node في dmesg
dmesg | grep -iE 'swnode|software.node|property.*registered'

# لو عندك device اسمه مثلاً "touchscreen"
DEVPATH=$(find /sys/devices -name "firmware_node" -type d 2>/dev/null | head -5)
for p in $DEVPATH; do
    echo "=== $p ==="
    cat "$p/path" 2>/dev/null || ls "$p/" 2>/dev/null
done
```

**5. تحقق من endpoint graph:**

```bash
# ابحث عن كل endpoints في DT
find /sys/firmware/devicetree/base -name "endpoint*" -type d 2>/dev/null | while read ep; do
    echo "=== $ep ==="
    # اقرأ remote-endpoint phandle
    re="$ep/remote-endpoint"
    if [ -f "$re" ]; then
        phandle=$(xxd "$re" | awk '{print $2$3}' | head -1)
        echo "  remote-endpoint phandle: 0x$phandle"
    fi
    # اقرأ port وendpoint IDs
    for prop in reg bus-type; do
        pf="$ep/$prop"
        [ -f "$pf" ] && printf "  %-20s: " "$prop" && xxd "$pf" | head -1
    done
done
```

**6. تحقق من DMA support:**

```bash
# لأي device
for dev in /sys/devices/platform/*; do
    name=$(basename "$dev")
    dma=$(cat "$dev/dma_mask_bits" 2>/dev/null)
    [ -n "$dma" ] && echo "$name: dma_mask_bits=$dma"
done
```

#### مثال على Output وكيف تفسره

**Output من `dmesg` بعد تفعيل dynamic debug:**

```
[   12.345678] property: device_property_read_u32 dev=ff160000.i2c propname="clock-frequency" nval=1
[   12.345701] swnode: software_node_register: node "touchscreen-params"
[   12.345712] property: fwnode_property_present fwnode=(of) prop="big-endian" → false
[   12.345780] property: fwnode_property_read_u32_array: prop="interrupts" ret=0 val[0]=45
```

**تفسير الـ output:**

```
ff160000.i2c        → اسم الـ platform device (address.type)
clock-frequency     → اسم الـ property اللي اتقرأت
nval=1              → طلب قراءة عنصر واحد
ret=0               → نجح
val[0]=45           → الـ IRQ number اللي اتقرأ
(of)                → الـ backend هو OF/Device Tree
→ false             → الـ property "big-endian" مش موجودة = little-endian
```

**Output من ftrace:**

```
 1)               |  device_property_read_u32_array() {
 1)               |    fwnode_property_read_u32_array() {
 1)               |      of_fwnode_property_read_int_array() {
 1)   0.541 us    |        of_find_property();
 1)   0.102 us    |        of_property_read_u32_index();
 1)   1.876 us    |      }
 1)   2.934 us    |    }
 1)   3.712 us    |  }
```

**تفسير:** الـ `device_property_read_u32_array` استدعت الـ OF backend مباشرة، ووجدت الـ property في `0.541 µs`. لو `of_find_property` بترجع NULL وإنت شايل الـ `-ENODATA` → الـ property مش موجودة في الـ DT node ده.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — I2C Sensor مش بيتشاف

#### العنوان
**الـ I2C temperature sensor مش بيظهر على نظام Industrial Gateway بسبب غلطة في `device_property_read_u32`**

#### السياق
شركة بتعمل industrial gateway على RK3562 للـ factory automation. الـ gateway بتقرأ بيانات حرارة من sensor خارجي متوصل على I2C2. الـ DT موجود وصح، لكن الـ driver بيفشل في الـ probe ومش بيقرأ الـ `clock-frequency`.

#### المشكلة
الـ driver بيعمل:
```c
u32 freq;
ret = device_property_read_u32(dev, "clock-frequency", &freq);
if (ret) {
    dev_err(dev, "failed to read clock-frequency: %d\n", ret);
    return ret;
}
```
الـ log بيظهر:
```
failed to read clock-frequency: -22
```
الـ `-22` ده `EINVAL` — مش `ENOENT`.

#### التحليل
**الـ `device_property_read_u32`** في `property.h` هي inline تستدعي:
```c
static inline int device_property_read_u32(const struct device *dev,
                                           const char *propname, u32 *val)
{
    return device_property_read_u32_array(dev, propname, val, 1);
}
```
اللي بيروح لـ `fwnode_property_read_u32_array` عبر `dev_fwnode(dev)`.

المشكلة إن الـ DT node فيه:
```dts
clock-frequency = <0x61A80>;  /* 400000 صح */
```
لكن الـ node اتعمله `status = "disabled"` في الـ overlay بتاع production، فـ `fwnode_device_is_available()` بترجع `false`، والـ kernel مش بيربط الـ fwnode بالـ device صح، فالـ `dev_fwnode(dev)` بيرجع `NULL` أو secondary fwnode فاضي.

لما `dev_fwnode` يرجع `NULL`:
```c
#define dev_fwnode(dev)   \
    _Generic((dev),       \
             const struct device *: __dev_fwnode_const, \
             struct device *: __dev_fwnode)(dev)
```
وجوا `fwnode_call_int_op`:
```c
#define fwnode_call_int_op(fwnode, op, ...)    \
    (fwnode_has_op(fwnode, op) ?               \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : \
     (IS_ERR_OR_NULL(fwnode) ? -EINVAL : -ENXIO))
```
لما `fwnode` يكون `NULL`، `IS_ERR_OR_NULL` بترجع `true` → `-EINVAL`.

#### الحل
```bash
# تأكد إن الـ node enabled في الـ DT
grep -r "clock-frequency\|status" arch/arm64/boot/dts/rockchip/rk3562-gateway.dts
```

في الـ DT:
```dts
&i2c2 {
    status = "okay";

    temp_sensor: tmp102@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
        status = "okay";   /* كان "disabled" */
        clock-frequency = <400000>;
    };
};
```

أو لو الـ driver نفسه محتاج يتحقق:
```c
/* تأكد إن الـ fwnode موجود قبل القراءة */
if (!dev_fwnode(dev)) {
    dev_warn(dev, "no fwnode, using default freq\n");
    freq = 100000; /* default */
} else {
    ret = device_property_read_u32(dev, "clock-frequency", &freq);
    if (ret)
        freq = 100000;
}
```

#### الدرس المستفاد
**الـ `-EINVAL` من `device_property_read_*` مش معناها إن الـ property مش موجودة** — ممكن تعني إن الـ `fwnode` نفسه `NULL` بسبب disabled node. دايمًا افحص `dev_fwnode(dev)` أولًا وتأكد إن الـ `status = "okay"`.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش شغال بسبب `fwnode_graph`

#### العنوان
**الـ HDMI output مش بيظهر على Android TV Box بسبب غلطة في `fwnode_graph_get_remote_endpoint`**

#### السياق
مصنع بيعمل Android TV Box رخيص على Allwinner H616. الـ display pipeline: DE2 → TCON → HDMI. كل حاجة متوصلة في DT عبر `ports/port/endpoint`. لكن بعد update في الـ DT، الشاشة وقفت تشتغل تمامًا.

#### المشكلة
الـ HDMI driver بيعمل:
```c
struct fwnode_handle *remote;
remote = fwnode_graph_get_remote_endpoint(dev_fwnode(dev));
if (!remote) {
    dev_err(dev, "no remote endpoint found\n");
    return -ENODEV;
}
```
بيطلع في الـ dmesg:
```
sunxi-hdmi: no remote endpoint found
```

#### التحليل
**الـ `fwnode_graph_get_remote_endpoint`** معرفة في `property.h`:
```c
struct fwnode_handle *fwnode_graph_get_remote_endpoint(
    const struct fwnode_handle *fwnode);
```

الـ `fwnode_graph_is_endpoint` بتشوف:
```c
static inline bool fwnode_graph_is_endpoint(const struct fwnode_handle *fwnode)
{
    return fwnode_property_present(fwnode, "remote-endpoint");
}
```

المشكلة: المطور اللي عدّل الـ DT نسي يضيف `remote-endpoint` في الـ endpoint الخاص بـ HDMI:

```dts
/* الـ DT الغلط */
&hdmi {
    ports {
        port@0 {
            hdmi_in: endpoint {
                /* remote-endpoint ناقص! */
            };
        };
    };
};
```

بدون `remote-endpoint`، الـ `fwnode_graph_is_endpoint` بترجع `false`، والـ graph traversal بيفشل.

**الـ `fwnode_graph_get_next_endpoint`** بتتحرك على endpoints اللي عندها `remote-endpoint` property، فلما تيجي تعدي الـ HDMI endpoint مش بتلاقيه.

#### الحل
```dts
/* الـ DT الصح */
&tcon_top {
    ports {
        port@1 {
            tcon_out_hdmi: endpoint@1 {
                reg = <1>;
                remote-endpoint = <&hdmi_in>;
            };
        };
    };
};

&hdmi {
    ports {
        port@0 {
            hdmi_in: endpoint {
                remote-endpoint = <&tcon_out_hdmi>;  /* ده اللي كان ناقص */
            };
        };
    };
};
```

تحقق بعد الإصلاح:
```bash
# تأكد إن الـ graph links صح
cat /sys/kernel/debug/of_graph
# أو
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "hdmi"
```

#### الدرس المستفاد
**الـ `fwnode_graph_*` functions** كلها تعتمد على وجود `remote-endpoint` property عشان تعرف إن الـ node ده endpoint. غياب الـ property = الـ graph مش بيشوف الـ node خالص، مش بس مش بيتربط.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — Software Node بعد Board Bring-up

#### العنوان
**إضافة SPI sensor بدون DT عن طريق `software_node` على STM32MP157 custom board**

#### السياق
فريق bring-up بيشتغل على custom board معتمدة على STM32MP157. الـ PCB فيها SPI accelerometer (ADXL345) متوصل على SPI3، لكن الـ DT الأصلي مش فيه أي mention لهذا الـ sensor عشان الـ board كانت prototype. محدش عايز يعدل الـ DT في هذه المرحلة، فالفريق قرر يستخدم `software_node`.

#### المشكلة
المطور عمل software_node غلط وجه `-ENOENT` لما حاول يقرأ الـ properties.

#### التحليل
الكود الغلط:
```c
static const struct property_entry adxl345_props[] = {
    PROPERTY_ENTRY_U32("spi-max-frequency", 5000000),
    PROPERTY_ENTRY_BOOL("spi-cpol"),
    /* نسي الإنهاء بـ {} */
};

static const struct software_node adxl345_node = {
    .name = "adxl345",
    .properties = adxl345_props,
};
```

**الـ `struct property_entry`** معرّفة في `property.h`:
```c
struct property_entry {
    const char *name;
    size_t length;
    bool is_inline;
    enum dev_prop_type type;
    union {
        const void *pointer;
        union {
            u8 u8_data[sizeof(u64) / sizeof(u8)];
            /* ... */
        } value;
    };
};
```

الـ array بيتمشى عليها بالـ loop وبيوقف لما `name == NULL`. بدون الـ terminator `{}` في الآخر، الـ loop بتاخد garbage memory كـ name → `ENOENT` أو kernel crash.

أيضًا، `PROPERTY_ENTRY_BOOL` بتعمل:
```c
#define PROPERTY_ENTRY_BOOL(_name_)  \
(struct property_entry) {            \
    .name = _name_,                  \
    .is_inline = true,               \
}
```
`length = 0`، `type` = 0 = `DEV_PROP_U8` — ده مقصود، الـ bool بتتعرف بـ `is_inline=true` و`length=0`.

#### الحل
```c
static const struct property_entry adxl345_props[] = {
    PROPERTY_ENTRY_U32("spi-max-frequency", 5000000),
    PROPERTY_ENTRY_BOOL("spi-cpol"),
    { }  /* terminator — ضروري */
};

static const struct software_node adxl345_node = {
    .name = "adxl345",
    .properties = adxl345_props,
};

/* في الـ module init */
static int __init adxl345_swnode_init(void)
{
    int ret;

    ret = software_node_register(&adxl345_node);
    if (ret) {
        pr_err("failed to register adxl345 swnode: %d\n", ret);
        return ret;
    }

    ret = device_add_software_node(&spi_dev->dev, &adxl345_node);
    if (ret)
        software_node_unregister(&adxl345_node);

    return ret;
}
```

تحقق:
```bash
ls /sys/bus/platform/devices/ | grep adxl
cat /sys/bus/spi/devices/spi0.0/properties/spi-max-frequency
```

#### الدرس المستفاد
**الـ `property_entry` array** لازم تنتهي بـ `{ }` زي بالظبط `struct file_operations`. والـ `PROPERTY_ENTRY_BOOL` مش بتحتاج value — وجود الـ name كفاية كما في DT. الـ `software_node` مثالية لـ prototype boards أو لما الـ hardware description تيجي من firmware خارجي.

---

### السيناريو 4: Automotive ECU على i.MX8QM — `device_is_big_endian` بيعمل data corruption

#### العنوان
**بيانات CAN bus غلط على i.MX8QM automotive ECU بسبب سوء فهم `device_is_big_endian`**

#### السياق
شركة automotive بتطور ECU على NXP i.MX8QM. الـ ECU بتتكلم مع CAN transceiver خارجي عبر SPI. الـ SPI driver بيقرأ registers من الـ transceiver. الداتا اللي بتيجي بتكون corrupted بشكل ثابت — قيم معكوسة byte-order.

#### المشكلة
المطور شاف إن الـ datasheet بيقول إن الـ transceiver بيستخدم Big-Endian registers، فأضاف `big-endian` للـ DT node وعمل:

```c
u32 reg_val;
spi_read(spi, &reg_val, sizeof(reg_val));

if (device_is_big_endian(&spi->dev))
    reg_val = be32_to_cpu(reg_val);
else
    reg_val = le32_to_cpu(reg_val);
```

الداتا لسه غلط.

#### التحليل
**الـ `device_is_big_endian`** معرّفة في `property.h`:
```c
static inline bool device_is_big_endian(const struct device *dev)
{
    return fwnode_device_is_big_endian(dev_fwnode(dev));
}
```

اللي بترجع لـ:
```c
static inline bool fwnode_device_is_big_endian(const struct fwnode_handle *fwnode)
{
    if (fwnode_property_present(fwnode, "big-endian"))
        return true;
    if (IS_ENABLED(CONFIG_CPU_BIG_ENDIAN) &&
        fwnode_property_present(fwnode, "native-endian"))
        return true;
    return false;
}
```

المشكلة إن `device_is_big_endian` بتقول لك إن **registers الـ device** Big-Endian — ده معيار الـ Linux device tree لـ memory-mapped registers. لكن الـ SPI transaction نفسها، الـ CPU (i.MX8QM = Little-Endian ARM64) بيبعتها كـ LE. الـ `be32_to_cpu` على LE CPU بتعمل byte-swap، لكن المطور عمل swap مرتين!

الـ transceiver بيستقبل data بـ SPI bit order واضح في الـ datasheet (MSB first). الـ SPI controller بيتحكم في ده بـ `spi-lsb-first` property، مش `big-endian`.

الكود الصح:
```c
/* الـ SPI data جاية صح — المشكلة في كيفية التفسير فقط */
u32 reg_val;
spi_read(spi, &reg_val, sizeof(reg_val));
/* SPI بيجيب BE data في buffer، نحوّلها للـ CPU endianness */
reg_val = be32_to_cpu(*((__be32 *)&reg_val));
```

#### الحل
احذف `big-endian` من DT node الـ SPI device لأنها مش معناها إيه تتوقع في SPI transactions:

```dts
&ecspi2 {
    status = "okay";

    can_transceiver: mcp2518fd@0 {
        compatible = "microchip,mcp2518fd";
        reg = <0>;
        spi-max-frequency = <20000000>;
        /* احذف big-endian — مش مناسبة هنا */
        /* الـ driver هو المسؤول عن be32_to_cpu للـ register values */
    };
};
```

وفي الـ driver:
```c
/* قرا Register كـ Big-Endian صح */
static int mcp251xfd_reg_read(struct spi_device *spi, u16 addr, u32 *val)
{
    __be32 be_val;
    /* ... SPI transfer ... */
    *val = be32_to_cpup(&be_val);  /* conversion واحدة بس */
    return 0;
}
```

#### الدرس المستفاد
**`device_is_big_endian`** مخصوص لـ **MMIO registers** — بتقول `ioread32be` vs `readl`. بالنسبة لـ SPI/I2C protocols، الـ byte order بيتحكم فيها الـ protocol نفسه والـ driver. استخدام `big-endian` في DT node لـ SPI device غلط semantically وبيسبب confusion.

---

### السيناريو 5: AM62x Custom Board — `fwnode_property_get_reference_args` لـ Multi-PHY Setup

#### العنوان
**RGMII Ethernet مش بيشتغل على AM62x industrial board بسبب غلطة في `phy-handle` reference**

#### السياق
فريق bring-up بيشتغل على custom industrial board معتمدة على TI AM62x. الـ board فيها اتنين Ethernet ports، كل واحدة متوصلة بـ PHY مختلفة عبر RGMII. الـ driver بيستخدم `fwnode_property_get_reference_args` عشان يجيب الـ PHY handle من DT.

#### المشكلة
الـ first port بيشتغل تمام. الـ second port بيفشل في probe مع:
```
am65-cpsw-nuss: Failed to get phy node: -2
```
الـ `-2` ده `ENOENT`.

#### التحليل
**الـ `fwnode_property_get_reference_args`** معرّفة في `property.h`:
```c
int fwnode_property_get_reference_args(const struct fwnode_handle *fwnode,
                                       const char *prop,
                                       const char *nargs_prop,
                                       unsigned int nargs,
                                       unsigned int index,
                                       struct fwnode_reference_args *args);
```

الـ driver بيعمل:
```c
struct fwnode_reference_args phy_args;
ret = fwnode_property_get_reference_args(dev_fwnode(dev),
                                         "phy-handle",
                                         NULL,  /* nargs_prop */
                                         0,     /* nargs */
                                         0,     /* index */
                                         &phy_args);
```

المشكلة في الـ DT:

```dts
/* الـ DT الغلط */
&cpsw3g {
    pinctrl-names = "default";
    pinctrl-0 = <&main_rgmii1_pins_default>;

    cpsw3g_mdio: mdio@f00 {
        #address-cells = <1>;
        #size-cells = <0>;

        cpsw3g_phy0: ethernet-phy@0 {
            reg = <0>;
        };
        cpsw3g_phy1: ethernet-phy@1 {
            reg = <1>;
        };
    };

    /* port@1 صح */
    cpsw_port1: port@1 {
        phy-handle = <&cpsw3g_phy0>;
        phy-mode = "rgmii-rxid";
    };

    /* port@2 فيه غلطة */
    cpsw_port2: port@2 {
        phy-handle = <&cpsw3g_phy1>;
        phy-mode = "rgmii-rxid";
        status = "disabled";  /* ده هو المشكلة */
    };
};
```

الـ `fwnode_device_is_available` بتشوف الـ `status` property:
```c
bool fwnode_device_is_available(const struct fwnode_handle *fwnode);
```

لما الـ `status = "disabled"`، بعض الـ of_graph traversal functions مش بتمشي على disabled nodes. لكن المشكلة هنا أعمق: الـ driver بيعمل `device_for_each_child_node` عشان يلاقي الـ ports:

```c
device_for_each_child_node(dev, port_fwnode) {
    /* بيعدي الـ disabled nodes أوتوماتيك */
}
```

الـ `device_for_each_child_node` بيستخدم `device_get_next_child_node` اللي بيستدعي `fwnode_get_next_child_node` — بعض الـ implementations بتعدي الـ disabled children بسبب OF core behavior.

#### الحل

```dts
/* الـ DT الصح */
cpsw_port2: port@2 {
    reg = <2>;
    label = "port2";
    phy-handle = <&cpsw3g_phy1>;
    phy-mode = "rgmii-rxid";
    status = "okay";  /* غيّرنا من disabled لـ okay */
    ti,mac-only;
};
```

لو الـ driver محتاج يتعامل مع disabled ports بشكل explicit:
```c
/* استخدم fwnode_for_each_child_node مش device_for_each_child_node */
fwnode_for_each_child_node(dev_fwnode(dev), port_fwnode) {
    /* بيمشي على كل children حتى الـ disabled */
    if (!fwnode_device_is_available(port_fwnode)) {
        dev_dbg(dev, "port %s is disabled, skipping\n",
                fwnode_get_name(port_fwnode));
        continue;
    }
    /* process port */
}
```

أو لو عايز تعدد كل الـ child nodes بما فيها الـ disabled:
```c
unsigned int total = fwnode_get_child_node_count(dev_fwnode(dev));
unsigned int available = 0;

fwnode_for_each_available_child_node(dev_fwnode(dev), child) {
    available++;
    fwnode_handle_put(child);
}

dev_info(dev, "total ports: %u, available: %u\n", total, available);
```

تأكد من الـ DTS باستخدام:
```bash
# على الـ target
cat /sys/firmware/devicetree/base/bus@f0000/ethernet@8000000/port@2/status
# لازم يطلع "okay"

# أو
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "port@2"
```

#### الدرس المستفاد
**`device_for_each_child_node`** و **`fwnode_for_each_available_child_node`** بيتعديان الـ `disabled` nodes أوتوماتيكيًا. لو عايز تعدد الـ disabled nodes كمان، استخدم **`fwnode_for_each_child_node`** واتحقق من الـ availability يدويًا بـ `fwnode_device_is_available`. فهم الفرق بين الـ macros الثلاثة حيوي لـ multi-port drivers.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي تطور الـ unified device property interface والـ fwnode API في الـ Linux kernel:

| المقال | الوصف |
|--------|-------|
| [Add ACPI _DSD and unified device properties support](https://lwn.net/Articles/614319/) | أول إعلان للـ patchset اللي أدخل الـ unified property API في kernel 3.19، بقلم Rafael Wysocki و Mika Westerberg |
| [Add ACPI _DSD and unified device properties support (earlier)](https://lwn.net/Articles/612062/) | النسخة الأقدم من نفس الـ patchset — بيوضح المناقشات الأولية حول التصميم |
| [device property: Introducing software nodes](https://lwn.net/Articles/770825/) | إدخال الـ `software_node` كـ fwnode type مستقل بديلاً لـ `property_set` |
| [Software fwnode references](https://lwn.net/Articles/789099/) | إضافة دعم الـ reference properties في الـ software nodes عبر `fwnode_property_get_reference_args()` |
| [ACPI graph support](https://lwn.net/Articles/718184/) | الـ patches اللي أضافت `fwnode_graph_*` API لدعم الـ port/endpoint topology في ACPI و DT |
| [Unified fwnode endpoint parser](https://lwn.net/Articles/737780/) | دمج الـ endpoint parser بين ACPI و OF لإنتاج API موحد لـ V4L2 والـ media subsystem |
| [introduce fwnode in the I2C subsystem](https://lwn.net/Articles/889236/) | تبني الـ I2C subsystem للـ fwnode API بدل الاعتماد المباشر على OF أو ACPI |
| [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/) | نظرة شاملة على الـ device model بما فيها الـ property framework |

---

### التوثيق الرسمي في الـ kernel

#### ملفات `Documentation/` ذات الصلة المباشرة

```
Documentation/driver-api/driver-model/driver.rst
Documentation/firmware-guide/acpi/DSD-properties-rules.rst
Documentation/firmware-guide/acpi/enumeration.rst
Documentation/firmware-guide/acpi/dsd/graph.rst
Documentation/firmware-guide/acpi/dsd/phy.rst
Documentation/devicetree/bindings/
```

**الـ** `DSD-properties-rules.rst` هو المرجع الأساسي لفهم قواعد استخدام الـ `_DSD` properties مع ACPI — الـ source الأصلي للـ `DEV_PROP_*` types.

روابط الـ docs الرسمية:

- [_DSD Device Properties Usage Rules](https://docs.kernel.org/firmware-guide/acpi/DSD-properties-rules.html) — قواعد استخدام الـ `_DSD` object في ACPI 5.1
- [ACPI Based Device Enumeration](https://www.kernel.org/doc/html/latest/firmware-guide/acpi/enumeration.html) — كيف يشتغل الـ `PRP0001` مع `compatible` property
- [Graphs — ACPI DSD](https://docs.kernel.org/firmware-guide/acpi/dsd/graph.html) — توثيق الـ `fwnode_graph_*` API من ناحية ACPI
- [Device drivers infrastructure](https://static.lwn.net/kerneldoc/driver-api/infrastructure.html) — الـ driver-api الكاملة بما فيها `driver_find_device_by_fwnode()`
- [V4L2 fwnode kAPI](https://www.kernel.org/doc/html/latest/driver-api/media/v4l2-fwnode.html) — مثال حي على استخدام الـ fwnode API في subsystem حقيقي

#### الملفات الأساسية في الـ source tree

```
include/linux/property.h       ← الـ header الرئيسي (هذا الملف)
include/linux/fwnode.h         ← تعريف struct fwnode_handle و fwnode_operations
drivers/base/property.c        ← implementation الـ device_property_* functions
drivers/base/swnode.c          ← implementation الـ software_node
drivers/base/core.c            ← ربط الـ fwnode بالـ struct device
```

---

### commits مهمة في تاريخ الـ API

| الحدث | المرجع |
|-------|--------|
| إدخال الـ unified device property interface (kernel 3.19) | [PATCH v6 02/12 — Rafael Wysocki على lore.kernel.org](https://lore.kernel.org/lkml/2127128.VT1Iq03xz1@vostro.rjw.lan/) |
| شرح المحاضرة الأصلية لتصميم الـ API | [Unified Device Properties Interface — Linux Foundation slides (PDF)](https://events.static.linuxfound.org/sites/events/files/slides/unified_properties_API_0.pdf) |
| دعم GPIO مع الـ unified property interface في Linux 3.19 | [gpio: Support for unified device properties interface](https://www.systutorials.com/linux-kernels/541953/gpio-support-for-unified-device-properties-interface-linux-3-19/) |

---

### مناقشات الـ mailing list

| الموضوع | الرابط |
|---------|--------|
| Fallback to secondary fwnode if primary misses property | [linux.kernel.narkive.com](https://linux.kernel.narkive.com/XO6y722J/patch-v1-08-13-device-property-fallback-to-secondary-fwnode-if-primary-misses-the-property) |
| إضافة `fwnode_get_name()` للـ fwnode framework | [mail-archive.com — PATCH v3 04/10](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2091140.html) |
| Device property improvements + `%pfw` format specifier | [mail-archive.com — PATCH v9 00/12](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2123755.html) |
| إضافة `fwnode_graph_*` لـ software_node | [mail-archive.com — software_node graph support](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2312505.html) |
| كيفية inject fwnode/oftree/acpi data من platform driver | [mail-archive.com — How to inject fwnode](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2023615.html) |
| استخدام fwnode API للـ node names في vsprintf | [mail-archive.com — PATCH v4 08/11](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2093902.html) |

الأرشيف الكامل للـ LKML: [lkml.org](https://lkml.org/) — ابحث بـ "device property fwnode" أو "software_node" أو "property_entry".

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصول ذات الصلة:** Chapter 14 — *The Linux Device Model*
- بيشرح الـ `kobject`، الـ `bus_type`، وكيف اتبنى الـ device model اللي فوقه اتبنى الـ fwnode
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

> ملاحظة: LDD3 قديم (kernel 2.6) ومش بيغطي الـ `fwnode` API مباشرةً، لكنه أساس لفهم الـ device model.

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل ذو الصلة:** Chapter 17 — *Devices and Modules*
- بيغطي الـ device model والـ sysfs، وهي البنية التحتية اللي بيشتغل فوقها الـ property interface

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصول ذات الصلة:** Chapter 7 — *Bootloaders*، Chapter 11 — *BusyBox*، وChapter 15 — *Debugging Embedded Linux*
- بيغطي الـ Device Tree من ناحية عملية لأنظمة embedded، وهو السياق الأصلي للـ fwnode في DT

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي الـ driver model بعمق أكبر من LDD3، مع تفاصيل الـ `kobject` و `device` structures

---

### مصادر eLinux.org

| الصفحة | الوصف |
|--------|-------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لصيغة الـ DT ومفهوم الـ properties كـ key-value pairs |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | شرح عملي لإشكاليات الـ DT properties وكيف تُفسَّر |
| [Linux Drivers Device Tree Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) | دليل الـ drivers في استخدام DT properties — مثال حي على ما يعتمد عليه الـ fwnode DT backend |
| [Device Tree frowand](https://elinux.org/Device_Tree_frowand) | أدوات تحليل DT properties والـ access patterns |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | روابط موارد تطوير الـ kernel بشكل عام |

---

### kernelnewbies.org

- [LinuxChanges](https://kernelnewbies.org/LinuxChanges) — تتبع التغييرات في كل إصدار kernel، ابحث عن "device property" أو "fwnode" في إصدارات 3.19، 4.x، 5.x
- [Drivers](https://kernelnewbies.org/Drivers) — نقطة بداية لكتابة drivers تستخدم الـ property API
- [KernelGlossary](https://kernelnewbies.org/KernelGlossary) — تعريفات المصطلحات بما فيها DT و ACPI
- [Documents](https://kernelnewbies.org/Documents) — فهرس التوثيق المتاح

---

### أدوات البحث في الـ source code

```bash
# البحث عن كل المواضع اللي بتستخدم device_property_read_u32
grep -r "device_property_read_u32" drivers/ --include="*.c" | head -30

# عرض implementation الـ software_node كاملة
less drivers/base/swnode.c

# تتبع تاريخ الـ property.h في git
git log --oneline include/linux/property.h

# البحث عن الـ commit الأصلي للـ API
git log --oneline --all | grep -i "unified device property"

# مشاهدة كل الـ symbols المصدّرة من property.c
grep "EXPORT_SYMBOL" drivers/base/property.c
```

---

### كلمات البحث الموصى بها

للبحث في Google أو DuckDuckGo أو lore.kernel.org:

```
linux kernel fwnode_handle device property
linux kernel software_node property_entry
linux kernel unified device properties ACPI DT
Rafael Wysocki Mika Westerberg device property interface
linux kernel fwnode_graph endpoint port
linux kernel device_property_read_u32 driver example
linux kernel ACPI _DSD properties driver
linux kernel swnode software node registration
linux kernel property_entries_dup software node
linux kernel fwnode_connection_find_match
```

للبحث في Elixir Cross Reference (أفضل أداة للـ kernel source):

- [https://elixir.bootlin.com/linux/latest/source/include/linux/property.h](https://elixir.bootlin.com/linux/latest/source/include/linux/property.h)
- [https://elixir.bootlin.com/linux/latest/source/drivers/base/property.c](https://elixir.bootlin.com/linux/latest/source/drivers/base/property.c)
- [https://elixir.bootlin.com/linux/latest/source/drivers/base/swnode.c](https://elixir.bootlin.com/linux/latest/source/drivers/base/swnode.c)
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `device_property_present` — دالة exported مهمة بتتسأل "هل الـ property دي موجودة في الـ firmware node بتاع الـ device ده؟". بتتاخد في كتير من الـ drivers أثناء الـ probe لتحديد capabilities الجهاز. الـ hook هيطبع اسم الـ device واسم الـ property اللي بيتسأل عنها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_dev_prop.c
 * Trace every call to device_property_present():
 * prints the device name and the property being queried.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/kernel.h>       /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on device_property_present — log property queries");

/*
 * device_property_present() signature:
 *   bool device_property_present(const struct device *dev, const char *propname);
 *
 * On x86-64:  rdi = dev,  rsi = propname
 * On arm64:   x0  = dev,  x1  = propname
 * We read them from pt_regs via regs_get_kernel_argument().
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* First argument: pointer to struct device */
    const struct device *dev =
        (const struct device *)regs_get_kernel_argument(regs, 0);

    /* Second argument: property name string */
    const char *propname =
        (const char *)regs_get_kernel_argument(regs, 1);

    if (!dev || !propname)
        return 0;

    /* dev_name() returns the kobject name — safe to call in probe context */
    pr_info("device_property_present: dev=\"%s\"  prop=\"%s\"\n",
            dev_name(dev), propname);

    return 0; /* 0 = let the original function run normally */
}

/* kprobe descriptor — one per probed symbol */
static struct kprobe kp = {
    .symbol_name = "device_property_present",
    .pre_handler = handler_pre,
};

static int __init kp_devprop_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit kp_devprop_exit(void)
{
    /* Must unregister before the module is unloaded;
     * otherwise the pre_handler pointer becomes a dangling pointer
     * inside the kernel's kprobe table → instant panic on next hit. */
    unregister_kprobe(&kp);
    pr_info("kprobe removed from %s\n", kp.symbol_name);
}

module_init(kp_devprop_init);
module_exit(kp_devprop_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_dev_prop.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | ليه موجود |
|--------|-----------|
| `linux/module.h` | ماكروهات `module_init` / `module_exit` و `MODULE_LICENSE` |
| `linux/kprobes.h` | **`struct kprobe`** و `register_kprobe` / `unregister_kprobe` |
| `linux/device.h` | **`struct device`** و `dev_name()` |
| `linux/kernel.h` | `pr_info` / `pr_err` |

#### الـ `handler_pre`

الـ kprobe بيشغّل الـ `pre_handler` قبل ما تتنفذ الـ instruction الأولى في الدالة المستهدفة، يعني الـ arguments لسه موجودة في الـ registers. **`regs_get_kernel_argument(regs, N)`** دي helper محمولة بتجيب الـ N-th argument من الـ `pt_regs` بغض النظر عن الـ architecture (x86-64 أو arm64). بناخد `dev` و`propname` منها ونطبعهم.

الـ return value لازم يكون `0` عشان الـ kernel يكمل تنفيذ الدالة الأصلية — لو رجّعنا `1` هيـ skip الدالة وده خطر.

#### الـ `struct kprobe`

**`symbol_name`** بيقول للـ kprobe subsystem يحل عنوان الدالة بالاسم وقت الـ registration. `addr` بييجي مملّي أوتوماتيك.

#### الـ `module_init`

بنسجّل الـ kprobe فيه. لو فشل (مثلاً الدالة inlined أو `CONFIG_KPROBES=n`) بنرجّع الـ error لـ `insmod` فيطبعها وما يحمّلش الـ module.

#### الـ `module_exit`

الـ `unregister_kprobe` ضروري قبل تفريغ الـ module من الذاكرة. لو ما عملناهوش، الـ kernel هيفضل يستدعي `handler_pre` اللي رمزه اتمسح → **kernel panic**. ده الـ cleanup الأساسي لأي kprobe.

---

### تشغيل وقراءة النتيجة

```bash
# بناء الموديول
make

# تحميله
sudo insmod kprobe_dev_prop.ko

# تحميل أي driver يعمل probe لجهاز (مثال: USB أو I2C)
# أو بس اشوف الـ log مباشرة
sudo dmesg | grep device_property_present

# مثال على output:
# [  42.301] kprobe planted at device_property_present (ffffffff81a3bc20)
# [  42.415] device_property_present: dev="0000:00:1f.3"  prop="snps,reset-gpio"
# [  42.416] device_property_present: dev="i2c-0"         prop="clock-frequency"

# إزالة الموديول
sudo rmmod kprobe_dev_prop
```

**الـ output** بيكشف أي driver بيسأل عن أي property، مفيد لـ debugging مشاكل الـ Device Tree أو ACPI حيث property مش موجودة فالـ driver بيفشل في الـ probe صامتاً.
