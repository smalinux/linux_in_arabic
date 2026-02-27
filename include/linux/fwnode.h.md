## Phase 1: الصورة الكبيرة ببساطة

### المشكلة اللي بيحلها الـ fwnode

تخيل عندك جهاز كمبيوتر. الـ kernel لازم يعرف: "فين الـ I2C controller؟ وعنده كام interrupt؟ وبيشتغل بأي clock frequency؟" — يعني لازم يعرف كل التفاصيل التقنية لكل قطعة hardware قبل ما يشغلها.

المشكلة: في Linux بيتعامل مع نوعين مختلفين من الأنظمة:

- **أنظمة مبنية على Device Tree (DT)**: زي الـ ARM boards والـ embedded systems — بتوصف الـ hardware في ملف نصي بصيغة DT.
- **أنظمة مبنية على ACPI**: زي أي لابتوب أو PC — الـ BIOS/UEFI بيوفر الـ ACPI tables اللي بتوصف الـ hardware.

المشكلة الكبيرة: الـ driver بتاع مثلاً chip I2C معين المفروض يشتغل على ARM وعلى x86 بدون ما يهتم هو نفسه بالفرق ده. مش معقول كل driver يكتب كود مرتين — مرة يقرأ من DT ومرة من ACPI.

**الحل: الـ fwnode (firmware node)** — طبقة تجريد موحدة فوق الاتنين.

---

### القصة بالكامل

#### بدون fwnode (القديم):

```
Driver I2C
    ├── #ifdef CONFIG_OF  → يقرأ من Device Tree
    └── #ifdef CONFIG_ACPI → يقرأ من ACPI
```

كل driver فيه شرط وكود مكرر. الصيانة جحيم.

#### بعد fwnode (الحديث):

```
Driver I2C
    └── fwnode_property_read_u32("clock-frequency", &freq)
              ↓
        fwnode_handle  ← pointer موحد
              ↓
        ┌────────────┐
        │  OF (DT)   │  أو  │  ACPI  │  أو  │  Software Node  │
        └────────────┘       └────────┘       └─────────────────┘
```

الـ driver بيكلم الـ `fwnode_handle` فقط، وهو بيحول للـ backend الصح تلقائياً عبر الـ **vtable** (جدول دوال افتراضية).

---

### الـ fwnode.h بالتحديد — إيه اللي فيه؟

الـ `fwnode.h` هو الـ **header الأساسي** اللي بيعرّف:

#### 1. الـ `struct fwnode_handle` — القلب

```c
struct fwnode_handle {
    struct fwnode_handle *secondary; /* fallback لو الـ primary فشل */
    const struct fwnode_operations *ops; /* الـ vtable */
    struct device *dev;              /* الـ device المرتبط بيه */
    struct list_head suppliers;      /* من بيوفرله resources */
    struct list_head consumers;      /* مين بيستهلك resources منه */
    u8 flags;
};
```

ده الـ **handle** — زي pointer بس موحد. كل node في الـ firmware (DT node أو ACPI device) ليه `fwnode_handle` خاص بيه.

#### 2. الـ `struct fwnode_operations` — الـ vtable

```c
struct fwnode_operations {
    struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
    void (*put)(struct fwnode_handle *fwnode);
    bool (*property_present)(const struct fwnode_handle *, const char *);
    int  (*property_read_int_array)(...);
    /* graph traversal, child iteration, references ... */
};
```

ده جدول function pointers — زي الـ virtual functions في C++. كل backend (OF / ACPI / swnode) بيملي الجدول ده بالتنفيذ الخاص بيه.

#### 3. الـ `struct fwnode_link` — الـ dependency graph

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  /* اللي بيوفر resource */
    struct fwnode_handle *consumer;  /* اللي بيستهلك */
    u8 flags;
};
```

الـ kernel بيبني graph من العلاقات دي عشان يعرف يشغّل الـ devices بالترتيب الصح — الـ supplier الأول، بعدين الـ consumer.

#### 4. الـ `struct fwnode_endpoint` — توصيل الكاميرات والـ media

```c
struct fwnode_endpoint {
    unsigned int port;  /* رقم الـ port */
    unsigned int id;    /* رقم الـ endpoint */
    const struct fwnode_handle *local_fwnode;
};
```

ده بيوصف الـ graph اللي الـ media subsystem بيستخدمه لتوصيل الكاميرا بالـ ISP مثلاً.

#### 5. الـ Macros للاستدعاء الآمن

```c
/* بيتأكد إن الـ fwnode مش NULL وإن الـ op موجودة */
#define fwnode_call_int_op(fwnode, op, ...)  \
    (fwnode_has_op(fwnode, op) ?             \
     (fwnode)->ops->op(fwnode, ##__VA_ARGS__) : -ENXIO)
```

بدل ما كل حد يعمل null check يدوي.

---

### تخيل العملية بمثال حقيقي

**مثال**: driver بتاع sensor temperature على I2C يقرأ الـ `clock-frequency`.

```
1. الـ Device Tree (ARM):
   i2c@ff160000 {
       clock-frequency = <400000>;
       temp-sensor@48 { ... };
   };

2. ACPI (x86):
   Device (TMP) {
       Method (_CRS) { ... }
       Name (_DSD, Package {
           "clock-frequency", 400000
       })
   }

3. الـ Driver (نفس الكود على الاتنين):
   fwnode_property_read_u32(fwnode, "clock-frequency", &freq);
         ↓
   fwnode_call_int_op(fwnode, property_read_int_array, ...)
         ↓
   لو OF backend → of_property_read_u32()
   لو ACPI backend → acpi_dev_get_property()
   لو swnode backend → software_node_get_reference_args()
```

---

### الـ fwnode flags — حالات الـ node

| Flag | المعنى |
|------|---------|
| `FWNODE_FLAG_LINKS_ADDED` | اتعمل parse للـ dependency links بتاعته |
| `FWNODE_FLAG_NOT_DEVICE` | مش مرتبط بـ struct device (node وصفي فقط) |
| `FWNODE_FLAG_INITIALIZED` | الـ hardware اتهيأ |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | محتاج أولاده يتشغلوا الأول |
| `FWNODE_FLAG_BEST_EFFORT` | يحاول يتشغل حتى لو في suppliers مفيهاش driver |
| `FWNODE_FLAG_VISITED` | اتعمله visit أثناء الـ cycle detection |

---

### الـ secondary fwnode — ال fallback الذكي

الـ `secondary` في `fwnode_handle` بيخلي الـ kernel يربط اتنين مع بعض:

```
DT fwnode → secondary → ACPI fwnode
```

مثلاً لابتوب بيستخدم DT للـ clock لكن ACPI للـ interrupt — الـ kernel يقدر يدور في الاتنين بشفافية تامة.

---

### الـ Subsystems المرتبطة

الـ `fwnode.h` ينتمي لـ **3 subsystems** في نفس الوقت حسب MAINTAINERS:

| Subsystem | الملفات الأساسية |
|-----------|-----------------|
| **ACPI** | `drivers/acpi/`, `include/acpi/`, `include/linux/acpi.h` |
| **Driver Core** | `drivers/base/`, `include/linux/device.h`, `include/linux/property.h` |
| **Software Nodes & Device Properties** | `drivers/base/property.c`, `drivers/base/swnode.c`, `include/linux/property.h` |

---

### الملفات اللي لازم تعرفها

#### الـ Core (القلب):

| الملف | الدور |
|-------|-------|
| `include/linux/fwnode.h` | الـ types والـ vtable الأساسية (الملف ده) |
| `include/linux/property.h` | الـ API العالي المستوى للـ drivers |
| `drivers/base/property.c` | تنفيذ الـ unified property API |
| `drivers/base/swnode.c` | الـ Software Node backend |

#### الـ Backends:

| الملف | الـ Backend |
|-------|------------|
| `drivers/of/property.c` | Device Tree backend |
| `drivers/acpi/property.c` | ACPI backend |
| `drivers/acpi/device_pm.c` | ACPI power management عبر fwnode |

#### الـ Headers المصاحبة:

| الملف | الدور |
|-------|-------|
| `include/linux/of.h` | Device Tree API |
| `include/linux/acpi.h` | ACPI API |
| `include/linux/bits.h` | BIT() macro للـ flags |

---

### الخلاصة

الـ `fwnode.h` هو **عقد التجريد** اللي بيفصل الـ drivers عن طريقة وصف الـ hardware. بدونه كل driver كان هيكتب كوده مرتين — مرة لـ DT ومرة لـ ACPI. بيه، الـ driver بيكلم واجهة واحدة، والـ kernel يختار الـ backend الصح تلقائياً.
## Phase 2: شرح الـ fwnode (Firmware Node) Framework

### المشكلة — ليه الـ fwnode موجود أصلاً؟

الـ Linux kernel بيشتغل على hardware بتُعرَّف بطرق مختلفة جداً:

- على ARM Embedded: الـ hardware متعرفش عن نفسه — محتاج **Device Tree** (DT) يوصف كل device وconnections بتاعته.
- على x86 PC: الـ BIOS/UEFI بيوفر **ACPI** tables بتحتوي على نفس المعلومات دي.
- في Testing أو Virtual Environments: في **software_node** — بتعرف الـ hardware في الـ C code نفسه.

المشكلة الكبيرة: كل source دي لها format مختلف تماماً، والـ kernel drivers كانوا لازم يعرفوا هما بيشتغلوا على DT ولا ACPI ولا غيره، وبعدين يكتبوا code مختلف لكل حالة.

النتيجة: code duplication ضخم، وأي driver محتاج يدعم أكتر من platform لازم يكون فيه `#ifdef CONFIG_OF` و `#ifdef CONFIG_ACPI` في كل حتة.

---

### الحل — الـ fwnode abstraction layer

الـ kernel حل المشكلة دي بـ **abstraction layer** اسمه **fwnode (Firmware Node)**:

بدل ما كل driver يتكلم مع DT أو ACPI مباشرة، كل driver بيتكلم مع `fwnode_handle` — وده pointer abstract بيخفي وراه التفاصيل كلها.

```
Driver Code                fwnode API              Firmware Source
───────────               ──────────              ───────────────
device_property_read_u32() ──────────────────────► OF (Device Tree)
                                                 ► ACPI
                                                 ► software_node
```

الـ driver مش محتاج يعرف إيه الـ source — الـ fwnode هو اللي بيعرف.

---

### الـ Real-World Analogy — الترجمان الفوري

تخيل إنك في مؤتمر دولي وعندك:
- متحدث بيتكلم عربي (Device Tree)
- متحدث بيتكلم إنجليزي (ACPI)
- متحدث بيتكلم فرنساوي (software_node)

بدل ما كل واحد في الجمهور يتعلم اللغات الثلاثة، في **ترجمان فوري** (الـ fwnode framework) بيستمع لأي لغة وبيترجمها لنفس الـ interface.

الجمهور (الـ driver) بيسأل سؤال واحد: "إيه الـ interrupt number؟"
الترجمان بيروح للمتحدث المناسب ويجيب الإجابة.

التمثيل الكامل:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| الترجمان نفسه | `struct fwnode_handle` |
| طريقة تواصل الترجمان مع كل متحدث | `struct fwnode_operations` (vtable) |
| السؤال اللي بيسأله الجمهور | `device_property_read_*()` API |
| المتحدث العربي | OF/DT backend (`of_fwnode_ops`) |
| المتحدث الإنجليزي | ACPI backend (`acpi_fwnode_ops`) |
| المتحدث الفرنساوي | `software_node_ops` |
| جمهور المؤتمر | الـ drivers اللي بتستخدم الـ API |
| الحوار اللي في الـ microphone | الـ properties (مثلاً `clock-frequency`) |
| مترجم احتياطي تاني | الـ `secondary` fwnode (fallback chain) |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kernel Drivers                           │
│  i2c-driver.c   spi-driver.c   gpio-driver.c   clk-driver.c    │
└──────────────────────────┬──────────────────────────────────────┘
                           │  device_property_read_u32()
                           │  fwnode_get_named_child_node()
                           │  fwnode_graph_get_next_endpoint()
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Property API  (drivers/base/property.c)            │
│           Generic wrappers تستدعي fwnode_call_*_op()           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                  fwnode_handle  +  fwnode_operations             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ fwnode_handle                                            │   │
│  │  ├── ops ──────────────────────────────────────────────► │   │
│  │  ├── secondary ──► (fallback fwnode_handle)              │   │
│  │  ├── dev ──────────────────────────────────────────────► │   │
│  │  ├── suppliers (list_head)                               │   │
│  │  ├── consumers (list_head)                               │   │
│  │  └── flags (u8)                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────┬──────────────────┬──────────────────┬──────────────┘
             │                  │                  │
             ▼                  ▼                  ▼
┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐
│  OF / DT Layer  │  │   ACPI Layer     │  │  software_node      │
│  of_fwnode_ops  │  │ acpi_fwnode_ops  │  │  software_node_ops  │
│                 │  │                  │  │                     │
│ drivers/of/     │  │ drivers/acpi/    │  │ drivers/base/       │
│ property.c      │  │ property.c       │  │ swnode.c            │
└────────┬────────┘  └────────┬─────────┘  └──────────┬──────────┘
         │                   │                        │
         ▼                   ▼                        ▼
  Device Tree Blob      ACPI Tables              C structs in
  (DTB in memory)       (DSDT/SSDT)              driver code
```

---

### الـ Core Abstraction — الـ vtable بتاع الـ fwnode

الـ central idea هي الـ **`struct fwnode_operations`** — ده vtable خالص زي الـ virtual functions في C++، لكن implemented manually في C.

```c
struct fwnode_operations {
    /* lifecycle */
    struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
    void (*put)(struct fwnode_handle *fwnode);

    /* device status */
    bool (*device_is_available)(const struct fwnode_handle *fwnode);
    const void *(*device_get_match_data)(const struct fwnode_handle *fwnode,
                                         const struct device *dev);
    bool (*device_dma_supported)(const struct fwnode_handle *fwnode);
    enum dev_dma_attr (*device_get_dma_attr)(const struct fwnode_handle *fwnode);

    /* property access */
    bool (*property_present)(const struct fwnode_handle *fwnode,
                             const char *propname);
    bool (*property_read_bool)(const struct fwnode_handle *fwnode,
                               const char *propname);
    int (*property_read_int_array)(...);
    int (*property_read_string_array)(...);

    /* tree traversal */
    const char *(*get_name)(const struct fwnode_handle *fwnode);
    const char *(*get_name_prefix)(const struct fwnode_handle *fwnode);
    struct fwnode_handle *(*get_parent)(const struct fwnode_handle *fwnode);
    struct fwnode_handle *(*get_next_child_node)(...);
    struct fwnode_handle *(*get_named_child_node)(...);
    int (*get_reference_args)(...);

    /* graph (media pipeline) */
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);

    /* hardware access */
    void __iomem *(*iomap)(struct fwnode_handle *fwnode, int index);
    int (*irq_get)(const struct fwnode_handle *fwnode, unsigned int index);

    /* devlink */
    int (*add_links)(struct fwnode_handle *fwnode);
};
```

كل backend (OF, ACPI, swnode) بيعمل instance من الـ struct ده وبيملي الـ function pointers اللي بيعرفها.

---

### الـ fwnode_handle بالتفصيل

```c
struct fwnode_handle {
    struct fwnode_handle *secondary;   // fallback لو الـ op مش موجود
    const struct fwnode_operations *ops; // vtable — مين أنا؟
    struct device *dev;                // الـ device المرتبط (بعد probe)
    struct list_head suppliers;        // مين بيوفرلي resources
    struct list_head consumers;        // مين بياخد مني resources
    u8 flags;                          // FWNODE_FLAG_*
};
```

#### الـ `secondary` Field — الـ Fallback Chain

ده من أهم الـ features: لو الـ DT node عنده بعض الـ properties والـ ACPI node عنده properties تانية، ممكن تربطهم في chain:

```
fwnode_handle (OF) ──secondary──► fwnode_handle (ACPI) ──secondary──► NULL
```

لما يُستدعى operation مش موجود في الأول، الـ framework تقدر تنزل للـ secondary تلقائياً. ده بيسمح بـ hybrid configurations.

#### الـ `suppliers` و `consumers` Lists — الـ fw_devlink System

الـ firmware بيعرف dependencies بين الـ devices، مثلاً: الـ I2C controller محتاج الـ clock يشتغل الأول قبل ما هو يشتغل.

الـ `fwnode_link` بيمثل العلاقة دي:

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  // مين بيوفر
    struct list_head s_hook;         // hook في قائمة الـ suppliers
    struct fwnode_handle *consumer;  // مين بياخد
    struct list_head c_hook;         // hook في قائمة الـ consumers
    u8 flags;                        // FWLINK_FLAG_CYCLE | FWLINK_FLAG_IGNORE
};
```

الـ diagram بتاع الـ lists:

```
fwnode_handle (clk)                    fwnode_handle (i2c)
├── consumers list_head ◄──────────────── s_hook (في fwnode_link)
│                                         c_hook (في fwnode_link)
│                                          └───────────────────► suppliers list_head
└── ...
```

الـ kernel driver core بيستخدم الـ links دي عشان يضمن إن الـ probe order صح — الـ supplier يتـ probe قبل الـ consumer.

---

### الـ Flags بالتفصيل

```c
#define FWNODE_FLAG_LINKS_ADDED          BIT(0)  // تم parse الـ DT/ACPI وإضافة الـ links
#define FWNODE_FLAG_NOT_DEVICE           BIT(1)  // هذا الـ node مش هيبقى device (مثلاً pin group)
#define FWNODE_FLAG_INITIALIZED          BIT(2)  // الـ hardware اتـ initialize
#define FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD BIT(3) // لازم الـ children يتـ probe الأول
#define FWNODE_FLAG_BEST_EFFORT          BIT(4)  // probe حتى لو في suppliers ناقصين
#define FWNODE_FLAG_VISITED              BIT(5)  // اتـ visit أثناء cycle detection
```

---

### الـ Graph Endpoint API — لـ Media Pipelines

في الـ embedded systems، خصوصاً في الـ camera pipelines، في ports وendpoints بتربط الـ devices ببعض:

```
Camera Sensor ──port0──endpoint──► ──endpoint──port0── CSI-2 Controller
                                  (fwnode_link)
```

الـ `struct fwnode_endpoint` بيعبر عن طرف الوصلة دي:

```c
struct fwnode_endpoint {
    unsigned int port;           // رقم الـ port على الـ device
    unsigned int id;             // رقم الـ endpoint على الـ port
    const struct fwnode_handle *local_fwnode; // مين أنا
};
```

الـ software_node بيستخدم naming conventions محددة للـ ports والـ endpoints:

```c
#define SWNODE_GRAPH_PORT_NAME_FMT      "port@%u"       // e.g., "port@0"
#define SWNODE_GRAPH_ENDPOINT_NAME_FMT  "endpoint@%u"   // e.g., "endpoint@0"
```

---

### الـ Reference Args — الـ phandle مع Arguments

في الـ Device Tree، ممكن node تشير لـ node تانية وتبعت معاها arguments، مثلاً:

```dts
clocks = <&clk_node 2 3>;  // reference لـ clk_node مع args: 2, 3
```

الـ `struct fwnode_reference_args` بيحمل الـ reference دي:

```c
struct fwnode_reference_args {
    struct fwnode_handle *fwnode;           // الـ node المُشار إليها
    unsigned int nargs;                     // عدد الـ args
    u64 args[NR_FWNODE_REFERENCE_ARGS];    // max 16 argument
};
```

---

### الـ Dispatch Macros — قلب الـ Framework

الـ framework بيستخدم 4 macros أساسية عشان يعمل dispatch للـ operations:

```c
// بيتحقق إن الـ fwnode صالح وعنده الـ operation
#define fwnode_has_op(fwnode, op) \
    (!IS_ERR_OR_NULL(fwnode) && (fwnode)->ops && (fwnode)->ops->op)

// للـ ops اللي بترجع int — بترجع -EINVAL لو fwnode NULL، -ENXIO لو op مش موجود
#define fwnode_call_int_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ##__VA_ARGS__) : \
     (IS_ERR_OR_NULL(fwnode) ? -EINVAL : -ENXIO))

// للـ ops اللي بترجع bool
#define fwnode_call_bool_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ##__VA_ARGS__) : false)

// للـ ops اللي بترجع pointer
#define fwnode_call_ptr_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ##__VA_ARGS__) : NULL)

// للـ ops اللي مش بترجع حاجة
#define fwnode_call_void_op(fwnode, op, ...) \
    do { \
        if (fwnode_has_op(fwnode, op)) \
            (fwnode)->ops->op(fwnode, ##__VA_ARGS__); \
    } while (false)
```

الـ macros دي بتضمن إن:
1. الـ fwnode مش NULL ومش error pointer (عن طريق `IS_ERR_OR_NULL`).
2. الـ ops pointer موجود.
3. الـ operation المحدد موجود في الـ vtable.
4. لو أي حاجة ناقصة، بترجع default value مناسب.

---

### كيفية إنشاء fwnode جديد (من وجهة نظر Backend Developer)

```c
// 1. عرّف الـ vtable
static const struct fwnode_operations my_fwnode_ops = {
    .get                   = my_get,
    .put                   = my_put,
    .property_present      = my_property_present,
    .property_read_int_array = my_property_read_int_array,
    // ... باقي الـ ops
};

// 2. الـ struct الخاص بيك بيحتوي على fwnode_handle
struct my_node {
    struct fwnode_handle fwnode;  // لازم يكون embedded, مش pointer
    // ... بقية الـ data
};

// 3. إنشاء الـ fwnode
struct my_node *node = kzalloc(sizeof(*node), GFP_KERNEL);
fwnode_init(&node->fwnode, &my_fwnode_ops);
```

الـ `fwnode_init()` بتعمل:
```c
static inline void fwnode_init(struct fwnode_handle *fwnode,
                               const struct fwnode_operations *ops)
{
    fwnode->ops = ops;
    INIT_LIST_HEAD(&fwnode->consumers);  // قائمة فاضية
    INIT_LIST_HEAD(&fwnode->suppliers);  // قائمة فاضية
}
```

---

### الـ DMA Attributes

الـ enum ده بيعبر عن قدرة الـ device على الـ DMA:

```c
enum dev_dma_attr {
    DEV_DMA_NOT_SUPPORTED,  // الـ device مش بيعمل DMA أصلاً
    DEV_DMA_NON_COHERENT,   // DMA بدون cache coherency (محتاج explicit flush/invalidate)
    DEV_DMA_COHERENT,       // DMA مع hardware cache coherency
};
```

الـ ACPI بيقول للـ kernel الـ attribute ده عن طريق الـ `_CCA` method في الـ ACPI tables. الـ DT بيعمله عن طريق الـ `dma-coherent` property.

---

### الـ fwnode Framework بيملك إيه وبيفوّض إيه؟

| الـ fwnode Framework يملك | بيفوّض للـ Backend |
|---|---|
| الـ `fwnode_handle` struct definition | كيفية تخزين الـ properties |
| الـ dispatch macros | كيفية قراءة الـ properties من الـ DT أو ACPI |
| الـ fwnode_link management | كيفية تعريف الـ dependencies |
| الـ graph endpoint structs | كيفية parse الـ endpoint info |
| الـ `fwnode_operations` interface | كل الـ implementation |
| الـ lifecycle (init, get, put) | متى تُحذف الـ resources |
| الـ flag definitions | معنى الـ flags في الـ context بتاعه |

---

### الـ fw_devlink — الـ Probe Ordering System

**الـ fw_devlink** هو subsystem مرتبط بشكل وثيق بالـ fwnode. المهمة بتاعته: ضمان إن الـ driver بتاع الـ supplier يتـ probe قبل الـ consumer.

مثال: `i2c@40003000` محتاج `clk@40000000`:

```
fwnode(clk) ◄──────── fwnode_link ──────────► fwnode(i2c)
    │           supplier          consumer         │
    │                                              │
    ▼                                              ▼
device(clk)  [probe أولاً]             device(i2c)  [probe بعدين]
```

الـ kernel بيبني الـ graph ده من الـ firmware description (DT أو ACPI) بعدين بيستخدمه عشان يرتب الـ probe.

الـ `fw_devlink_is_strict()` بتحدد لو الـ kernel لازم يـ defer probe الـ consumer لحد ما الـ supplier يتـ probe — أو يـ probe على طول (best effort).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums — Cheatsheet

#### الـ `FWNODE_FLAG_*` — flags الخاصة بـ `fwnode_handle.flags`

| Flag | Bit | المعنى |
|------|-----|--------|
| `FWNODE_FLAG_LINKS_ADDED` | `BIT(0)` | الـ fwnode اتعمله parse وأُضيفت links الـ suppliers بتاعته |
| `FWNODE_FLAG_NOT_DEVICE` | `BIT(1)` | الـ fwnode مش هيتحول لـ `struct device` أبداً (مثال: nodes وصفية فقط) |
| `FWNODE_FLAG_INITIALIZED` | `BIT(2)` | الـ hardware المقابل للـ fwnode اتعمله initialize |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | `BIT(3)` | الـ driver محتاج الـ child devices تكون bound قبل ما يقدر يعمل probe |
| `FWNODE_FLAG_BEST_EFFORT` | `BIT(4)` | يعمل probe بدري حتى لو في suppliers ناقصة — يتجاهل suppliers من غير drivers |
| `FWNODE_FLAG_VISITED` | `BIT(5)` | اتزار خلال cycle detection في الـ devlink graph |

#### الـ `FWLINK_FLAG_*` — flags الخاصة بـ `fwnode_link.flags`

| Flag | Bit | المعنى |
|------|-----|--------|
| `FWLINK_FLAG_CYCLE` | `BIT(0)` | الـ link ده جزء من cycle — متعملش defer للـ probe |
| `FWLINK_FLAG_IGNORE` | `BIT(1)` | تجاهل الـ link ده خالص حتى في cycle detection |

#### الـ `enum dev_dma_attr`

| Value | المعنى |
|-------|--------|
| `DEV_DMA_NOT_SUPPORTED` | الـ device مش بيدعم DMA خالص |
| `DEV_DMA_NON_COHERENT` | بيدعم DMA بس cache غير متزامن — محتاج sync manual |
| `DEV_DMA_COHERENT` | بيدعم DMA مع cache coherency كامل |

#### الـ Macros المهمة

| Macro | الغرض |
|-------|--------|
| `NR_FWNODE_REFERENCE_ARGS` | الحد الأقصى لعدد الـ args في reference — قيمته 16 |
| `SWNODE_GRAPH_PORT_NAME_FMT` | format اسم الـ port في software nodes: `"port@%u"` |
| `SWNODE_GRAPH_ENDPOINT_NAME_FMT` | format اسم الـ endpoint: `"endpoint@%u"` |
| `fwnode_has_op(fwnode, op)` | يتحقق إن الـ fwnode مش NULL وإن الـ op موجودة في الـ ops table |
| `fwnode_call_int_op(...)` | يستدعي op وترجع int، أو `-EINVAL`/`-ENXIO` لو مفيش op |
| `fwnode_call_bool_op(...)` | يستدعي op وترجع bool، أو `false` لو مفيش op |
| `fwnode_call_ptr_op(...)` | يستدعي op وترجع pointer، أو `NULL` لو مفيش op |
| `fwnode_call_void_op(...)` | يستدعي op من غير ما يهتم بالـ return value |

---

### 1. الـ Structs المهمة

---

#### `struct fwnode_handle`

**الغرض:** ده الـ "handle" العام اللي بيمثل أي firmware node — سواء كان من ACPI أو Device Tree أو software node. ده الـ base object اللي كل الـ firmware backends بتوسعه.

```c
struct fwnode_handle {
    struct fwnode_handle *secondary;   /* fallback node لو الأولاني مش كافي */
    const struct fwnode_operations *ops; /* vtable — بتتحكم في كل العمليات */

    /* بيستخدمه device links فقط */
    struct device *dev;                /* الـ device المرتبط بالـ node ده */
    struct list_head suppliers;        /* قائمة الـ fwnode_link اللي الـ node ده consumer فيها */
    struct list_head consumers;        /* قائمة الـ fwnode_link اللي الـ node ده supplier فيها */
    u8 flags;                          /* FWNODE_FLAG_* */
};
```

| Field | النوع | الشرح |
|-------|-------|--------|
| `secondary` | `struct fwnode_handle *` | لو الـ primary node مش قادر يجاوب على سؤال، الـ kernel بيروح للـ secondary — مثال: ACPI + swnode معاً |
| `ops` | `const struct fwnode_operations *` | الـ vtable — كل backend بيحط الـ function pointers بتاعته هنا |
| `dev` | `struct device *` | ربط مع الـ device object لما الـ node يتحول لـ device — `NULL` غير كده |
| `suppliers` | `struct list_head` | رأس قائمة الـ `fwnode_link` nodes اللي الـ fwnode ده بياخد منها (consumer) |
| `consumers` | `struct list_head` | رأس قائمة الـ `fwnode_link` nodes اللي بتاخد من الـ fwnode ده (supplier) |
| `flags` | `u8` | بتات حالة الـ node — الـ `FWNODE_FLAG_*` |

**الارتباط بالـ structs التانية:** بيشير لـ `fwnode_operations` وبيشير له `fwnode_link` من طرفين.

---

#### `struct fwnode_operations`

**الغرض:** الـ **vtable** — جدول الـ function pointers اللي كل firmware backend (ACPI, DT, swnode) بيملاه بالـ implementations الخاصة بيه. ده اللي بيخلي الـ kernel يشتغل مع أي firmware بنفس الـ API.

```c
struct fwnode_operations {
    /* Reference counting */
    struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
    void (*put)(struct fwnode_handle *fwnode);

    /* Device availability & metadata */
    bool (*device_is_available)(const struct fwnode_handle *fwnode);
    const void *(*device_get_match_data)(...);
    bool (*device_dma_supported)(...);
    enum dev_dma_attr (*device_get_dma_attr)(...);

    /* Property access */
    bool (*property_present)(...);
    bool (*property_read_bool)(...);
    int (*property_read_int_array)(...);
    int (*property_read_string_array)(...);

    /* Node navigation */
    const char *(*get_name)(...);
    const char *(*get_name_prefix)(...);
    struct fwnode_handle *(*get_parent)(...);
    struct fwnode_handle *(*get_next_child_node)(...);
    struct fwnode_handle *(*get_named_child_node)(...);

    /* References */
    int (*get_reference_args)(...);

    /* Graph (ports & endpoints) */
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);

    /* Hardware access */
    void __iomem *(*iomap)(struct fwnode_handle *fwnode, int index);
    int (*irq_get)(...);

    /* Dependency management */
    int (*add_links)(struct fwnode_handle *fwnode);
};
```

| Function Pointer | الغرض المختصر |
|------------------|---------------|
| `get` / `put` | reference counting — حماية الـ node من الحذف |
| `device_is_available` | هل الـ node متاح في الـ firmware؟ (مش `disabled`) |
| `device_get_match_data` | بيجيب الـ driver match data من الـ firmware table |
| `device_dma_supported` | هل الـ device يدعم DMA؟ |
| `device_get_dma_attr` | نوع الـ DMA coherency |
| `property_present` | هل property موجودة؟ |
| `property_read_bool` | اقرأ property boolean |
| `property_read_int_array` | اقرأ array من integers |
| `property_read_string_array` | اقرأ array من strings |
| `get_name` / `get_name_prefix` | اسم الـ node للـ logging والـ debug |
| `get_parent` | الـ parent node في شجرة الـ firmware |
| `get_next_child_node` | iteration على الـ children |
| `get_named_child_node` | جيب child باسم معين |
| `get_reference_args` | حل reference property وجيب الـ args معاها |
| `graph_get_next_endpoint` | iteration على الـ endpoints في الـ graph |
| `graph_get_remote_endpoint` | الطرف الآخر من connection |
| `graph_get_port_parent` | الـ device اللي فيه الـ port |
| `graph_parse_endpoint` | اقرأ port id وendpoint id |
| `iomap` | عمل map لـ MMIO region |
| `irq_get` | جيب رقم الـ IRQ |
| `add_links` | أنشئ dependency links لـ suppliers |

---

#### `struct fwnode_link`

**الغرض:** بيمثل **علاقة dependency** بين fwnode تاني — supplier وconsumer. الـ kernel بيستخدمه عشان يضمن إن الـ supplier يعمل probe قبل الـ consumer.

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  /* الـ node اللي بيوفر الـ resource */
    struct list_head s_hook;         /* hook في قائمة consumers الخاصة بالـ supplier */
    struct fwnode_handle *consumer;  /* الـ node اللي محتاج الـ resource */
    struct list_head c_hook;         /* hook في قائمة suppliers الخاصة بالـ consumer */
    u8 flags;                        /* FWLINK_FLAG_* */
};
```

| Field | الشرح |
|-------|--------|
| `supplier` | مؤشر للـ fwnode اللي بيوفر resource — مثلاً: clock أو regulator |
| `s_hook` | entry في `supplier->consumers` list |
| `consumer` | مؤشر للـ fwnode اللي محتاج الـ resource |
| `c_hook` | entry في `consumer->suppliers` list |
| `flags` | `FWLINK_FLAG_CYCLE` أو `FWLINK_FLAG_IGNORE` |

---

#### `struct fwnode_endpoint`

**الغرض:** بيمثل **نقطة اتصال** في graph الـ firmware — زي connector في دائرة كاميرا أو display pipeline. كل endpoint عنده port id وendpoint id.

```c
struct fwnode_endpoint {
    unsigned int port;                       /* رقم الـ port في الـ device */
    unsigned int id;                         /* رقم الـ endpoint داخل الـ port */
    const struct fwnode_handle *local_fwnode; /* الـ node المحلي */
};
```

مثال واقعي: كاميرا (port 0, endpoint 0) متوصلة بـ ISP (port 1, endpoint 0).

---

#### `struct fwnode_reference_args`

**الغرض:** لما property في الـ firmware بتشير لـ node تانية وبتعدي معاها arguments — زي `clocks = <&clk0 1 0>` في Device Tree، الـ node هنا هي `clk0` والـ args هي `[1, 0]`.

```c
struct fwnode_reference_args {
    struct fwnode_handle *fwnode;          /* الـ node المُشار إليه */
    unsigned int nargs;                    /* عدد الـ args الفعلية */
    u64 args[NR_FWNODE_REFERENCE_ARGS];   /* الـ args — حد أقصى 16 */
};
```

---

### 2. مخطط العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────┐
│                  fwnode_handle                      │
│                                                     │
│  secondary ──────────────────────────────────────►  fwnode_handle
│                                                     │  (fallback)
│  ops ────────────────────────────────────────────►  fwnode_operations
│                                                     │  (vtable: get/put/
│  dev ────────────────────────────────────────────►  struct device       │   property_*/graph_*)
│                                                     │
│  suppliers ◄──── list_head ◄──── fwnode_link.c_hook │
│                                      │              │
│  consumers ◄──── list_head ◄──── fwnode_link.s_hook │
│                                                     │
└─────────────────────────────────────────────────────┘
                         ▲               ▲
                         │               │
              fwnode_link.consumer    fwnode_link.supplier
                         │               │
              ┌──────────┴───────────────┴──────────┐
              │           fwnode_link                │
              │  supplier ─────────────────────────► fwnode_handle (supplier)
              │  consumer ─────────────────────────► fwnode_handle (consumer)
              │  s_hook  (في قائمة supplier->consumers)            │
              │  c_hook  (في قائمة consumer->suppliers)            │
              │  flags                                              │
              └─────────────────────────────────────────────────────┘

fwnode_reference_args:
              ┌──────────────────────────────────────┐
              │       fwnode_reference_args           │
              │  fwnode ───────────────────────────► fwnode_handle
              │  nargs  = 2                           │
              │  args   = [1, 0, ...]                 │
              └──────────────────────────────────────┘

fwnode_endpoint:
              ┌──────────────────────────────────────┐
              │         fwnode_endpoint               │
              │  port         = 0                     │
              │  id           = 0                     │
              │  local_fwnode ─────────────────────► fwnode_handle
              └──────────────────────────────────────┘
```

---

#### مخطط الـ Supplier/Consumer Graph بالتفصيل

```
  fwnode_handle (clock supplier)          fwnode_handle (uart consumer)
  ┌──────────────────────────┐            ┌──────────────────────────┐
  │  consumers               │            │  suppliers               │
  │  [list_head] ◄──────────────────────► [list_head]               │
  │                          │  fwnode_link                         │
  │                          │  ┌────────────────────────────────┐  │
  │                          │  │ supplier ──► clock fwnode_handle│  │
  │                          │  │ s_hook   ── في clock->consumers │  │
  │                          │  │ consumer ──► uart fwnode_handle │  │
  │                          │  │ c_hook   ── في uart->suppliers  │  │
  │                          │  └────────────────────────────────┘  │
  └──────────────────────────┘            └──────────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

#### lifecycle الـ `fwnode_handle`

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                         CREATION                                        │
  │                                                                         │
  │  Backend allocates fwnode (e.g. acpi_device, of_node, swnode)          │
  │           │                                                             │
  │           ▼                                                             │
  │  fwnode_init(fwnode, &backend_ops)                                      │
  │    → ops = &backend_ops                                                 │
  │    → INIT_LIST_HEAD(&fwnode->consumers)                                 │
  │    → INIT_LIST_HEAD(&fwnode->suppliers)                                 │
  │           │                                                             │
  │           ▼                                                             │
  │                         REGISTRATION                                    │
  │                                                                         │
  │  device_add() → device->fwnode = fwnode                                │
  │               → fwnode->dev = device                                   │
  │           │                                                             │
  │           ▼                                                             │
  │                      LINK CREATION                                      │
  │                                                                         │
  │  fwnode->ops->add_links(fwnode)                                         │
  │    → يحدد كل الـ suppliers من الـ firmware                              │
  │    → fwnode_link_add(consumer, supplier, flags)                         │
  │    → FWNODE_FLAG_LINKS_ADDED يتعمل set                                  │
  │           │                                                             │
  │           ▼                                                             │
  │                         PROBE ORDER                                     │
  │                                                                         │
  │  fw_devlink يتحقق إن كل suppliers عملوا probe أول                       │
  │    → لو في cycle: FWLINK_FLAG_CYCLE يتعمل set → متعملش defer           │
  │           │                                                             │
  │           ▼                                                             │
  │                       HARDWARE INIT                                     │
  │                                                                         │
  │  driver->probe() ينجح                                                   │
  │    → fwnode_dev_initialized(fwnode, true)                               │
  │    → FWNODE_FLAG_INITIALIZED يتعمل set                                  │
  │           │                                                             │
  │           ▼                                                             │
  │                         TEARDOWN                                        │
  │                                                                         │
  │  device_del() أو driver unload                                          │
  │    → fwnode_dev_initialized(fwnode, false)                              │
  │    → fwnode_links_purge(fwnode) — يمسح كل الـ links                    │
  │    → fw_devlink_purge_absent_suppliers(fwnode)                          │
  │    → fwnode->ops->put(fwnode)                                           │
  └─────────────────────────────────────────────────────────────────────────┘
```

---

#### lifecycle الـ `fwnode_link`

```
  fwnode_link_add(consumer, supplier, flags)
         │
         ▼
  alloc fwnode_link
  link->supplier = supplier
  link->consumer = consumer
  link->flags = flags
         │
         ├── list_add(&link->s_hook, &supplier->consumers)
         └── list_add(&link->c_hook, &consumer->suppliers)
         │
         ▼
   [fw_devlink checks this link during probe ordering]
         │
         ▼
  fwnode_links_purge(fwnode)
         │
         ├── iterates supplier->consumers → removes + kfrees links
         └── iterates consumer->suppliers → removes + kfrees links
```

---

### 4. مخططات الـ Call Flow

#### قراءة property من driver

```
driver calls: fwnode_property_read_u32(fwnode, "clock-frequency", &val)
  │
  ▼
property.h wrapper
  → fwnode_call_int_op(fwnode, property_read_int_array, ...)
      │
      ├── fwnode_has_op(fwnode, property_read_int_array)?
      │     → !IS_ERR_OR_NULL(fwnode) && fwnode->ops && fwnode->ops->property_read_int_array
      │
      ├── YES → fwnode->ops->property_read_int_array(fwnode, "clock-frequency", 4, &val, 1)
      │           │
      │           ├── [Device Tree backend] → of_property_read_u32_array()
      │           │     → reads from DTB in memory
      │           │
      │           └── [ACPI backend] → acpi_dev_get_property()
      │                 → reads from ACPI namespace
      │
      └── NO (ops==NULL) → return -ENXIO
                           (IS_ERR_OR_NULL) → return -EINVAL
```

#### graph endpoint traversal

```
driver calls: fwnode_graph_get_next_endpoint(fwnode, NULL)
  │
  ▼
fwnode_call_ptr_op(fwnode, graph_get_next_endpoint, NULL)
  │
  ▼
fwnode->ops->graph_get_next_endpoint(fwnode, NULL)
  │
  ├── [DT] → of_graph_get_next_endpoint()
  │     → يدور على child nodes باسم "port@*" ثم "endpoint@*"
  │
  └── returns fwnode_handle للـ endpoint
         │
         ▼
  fwnode_graph_parse_endpoint(endpoint_fwnode, &ep_struct)
    → fwnode->ops->graph_parse_endpoint(fwnode, &ep_struct)
    → ep_struct.port = رقم الـ port
    → ep_struct.id   = رقم الـ endpoint
         │
         ▼
  fwnode_graph_get_remote_endpoint(endpoint_fwnode)
    → fwnode->ops->graph_get_remote_endpoint(fwnode)
    → يرجع فيه الطرف الآخر من الكابل
```

#### dependency probe ordering

```
fw_devlink_create_devlink() [في drivers/base/]
  │
  ▼
fwnode->ops->add_links(fwnode)
  │
  ├── يمشي على كل references في الـ firmware node
  │     مثال: "clocks", "power-domains", "regulators"
  │
  ▼
fwnode_link_add(consumer_fwnode, supplier_fwnode, 0)
  │
  ├── يخلق fwnode_link object
  └── يضيفه في lists الطرفين
         │
         ▼
  really_probe() في drivers/base/dd.c
    → بيتحقق: هل كل fwnode_link في consumer->suppliers
              الـ supplier بتاعها عمل probe وفيها FWNODE_FLAG_INITIALIZED؟
    → لو لأ → defer probe
    → لو آه → كمل
```

---

### 5. استراتيجية الـ Locking

الـ `fwnode.h` نفسه **مش بيعرّف locks صريحة** — لكن الـ locking بيحصل في الطبقات اللي فوقيه:

#### جدول الـ Locking

| Data | الـ Lock المستخدم | مكانه |
|------|-------------------|--------|
| `fwnode_handle.flags` | `device_links_write_lock()` — وهو `device_links_lock` (rwsem) | `drivers/base/core.c` |
| `fwnode_handle.suppliers` list | `device_links_write_lock()` | `drivers/base/core.c` |
| `fwnode_handle.consumers` list | `device_links_write_lock()` | `drivers/base/core.c` |
| `fwnode_link` allocation/free | `device_links_write_lock()` | `drivers/base/core.c` |
| قراءة properties (ops calls) | بدون lock في الغالب — الـ properties immutable بعد الـ init | — |
| الـ `fwnode_handle.dev` pointer | `device_lock(dev)` لو محتاج access متزامن | `drivers/base/core.c` |

#### ترتيب الـ Locking (Lock Ordering)

```
device_links_write_lock()   [rwsem - outer]
    └── device_lock(dev)    [mutex - inner]
```

**قاعدة:** مينفعش تمسك `device_lock` وبعدين تاخد `device_links_write_lock` — ده هيعمل deadlock. الترتيب دايماً: device_links أولاً ثم device_lock.

#### الـ `fwnode_dev_initialized()` و `fwnode_init()` — thread safety

- **`fwnode_init()`**: بيتعمل call مرة واحدة أثناء الـ initialization — قبل ما أي thread تاني يشوف الـ object، فمش محتاج lock.
- **`fwnode_dev_initialized()`**: بيعدل `flags` — المفروض يتعمل call من سياق الـ device probe/remove اللي محمي بـ `device_lock`.
- الـ `IS_ERR_OR_NULL` check في `fwnode_dev_initialized()` بيحمي من الـ NULL dereference لو الـ fwnode مش موجود.

#### الـ `secondary` pointer

الـ `fwnode_handle.secondary` بيتعمل set مرة واحدة وبيبقى ثابت — فالقراءة منه مش محتاجة lock. الـ `fwnode_call_*` macros بيتبعوا chain الـ secondary أوتوماتيك من خلال الـ ops.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs — Cheatsheet

#### الـ Inline Functions

| Function | النوع | الغرض |
|---|---|---|
| `fwnode_init()` | `static inline` | تهيئة `fwnode_handle` بربط الـ ops وتهيئة الـ lists |
| `fwnode_dev_initialized()` | `static inline` | set/clear الـ `FWNODE_FLAG_INITIALIZED` flag |

#### الـ Exported Functions

| Function | النوع | الغرض |
|---|---|---|
| `fwnode_link_add()` | exported | إنشاء `fwnode_link` بين supplier وconsumer |
| `fwnode_links_purge()` | exported | حذف كل الـ links المرتبطة بـ fwnode |
| `fw_devlink_purge_absent_suppliers()` | exported | تنظيف links الـ suppliers الغير موجودة |
| `fw_devlink_is_strict()` | exported | يرجع true لو الـ fw_devlink mode هو strict |

#### الـ Dispatch Macros (vtable call helpers)

| Macro | يرجع | الغرض |
|---|---|---|
| `fwnode_has_op(fwnode, op)` | `bool` | check إن الـ op موجود في الـ vtable |
| `fwnode_call_int_op(fwnode, op, ...)` | `int` | استدعاء op يرجع int |
| `fwnode_call_bool_op(fwnode, op, ...)` | `bool` | استدعاء op يرجع bool |
| `fwnode_call_ptr_op(fwnode, op, ...)` | `ptr` | استدعاء op يرجع pointer |
| `fwnode_call_void_op(fwnode, op, ...)` | `void` | استدعاء op لا يرجع قيمة |

#### الـ fwnode_operations vtable pointers

| Op | Signature مختصرة | الغرض |
|---|---|---|
| `get` | `fwnode_handle *(*)(fwnode_handle *)` | ref-count زيادة |
| `put` | `void (*)(fwnode_handle *)` | ref-count نقصان |
| `device_is_available` | `bool (*)(fwnode_handle *)` | هل الـ device متاح في الـ firmware |
| `device_get_match_data` | `const void *(*)(fwnode_handle *, device *)` | driver match data |
| `device_dma_supported` | `bool (*)(fwnode_handle *)` | هل الـ device يدعم DMA |
| `device_get_dma_attr` | `enum dev_dma_attr (*)(fwnode_handle *)` | نوع الـ DMA coherency |
| `property_present` | `bool (*)(fwnode_handle *, propname)` | هل property موجودة |
| `property_read_bool` | `bool (*)(fwnode_handle *, propname)` | قراءة boolean property |
| `property_read_int_array` | `int (*)(fwnode_handle *, propname, elem_size, val, nval)` | قراءة integer array |
| `property_read_string_array` | `int (*)(fwnode_handle *, propname, val, nval)` | قراءة string array |
| `get_name` | `const char *(*)(fwnode_handle *)` | اسم الـ node |
| `get_name_prefix` | `const char *(*)(fwnode_handle *)` | prefix للطباعة |
| `get_parent` | `fwnode_handle *(*)(fwnode_handle *)` | الـ parent node |
| `get_next_child_node` | `fwnode_handle *(*)(fwnode_handle *, child)` | iteration على الـ children |
| `get_named_child_node` | `fwnode_handle *(*)(fwnode_handle *, name)` | child باسم معين |
| `get_reference_args` | `int (*)(fwnode_handle *, prop, nargs_prop, nargs, index, args)` | resolve phandle reference |
| `graph_get_next_endpoint` | `fwnode_handle *(*)(fwnode_handle *, prev)` | iteration على الـ endpoints |
| `graph_get_remote_endpoint` | `fwnode_handle *(*)(fwnode_handle *)` | الـ remote endpoint المقابل |
| `graph_get_port_parent` | `fwnode_handle *(*)(fwnode_handle *)` | parent الـ port node |
| `graph_parse_endpoint` | `int (*)(fwnode_handle *, fwnode_endpoint *)` | parse port/id من endpoint |
| `iomap` | `void __iomem *(*)(fwnode_handle *, index)` | map MMIO resource |
| `irq_get` | `int (*)(fwnode_handle *, index)` | get Linux IRQ number |
| `add_links` | `int (*)(fwnode_handle *)` | بناء الـ fwnode links للـ suppliers |

---

### Group 1: Initialization — تهيئة الـ fwnode

الـ group ده مسؤول عن تهيئة الـ `fwnode_handle` قبل ما يتسجّل في النظام. كل firmware backend (OF، ACPI، swnode) بيستخدمه لربط الـ ops وتحضير الـ linked lists اللي بتتحكم في الـ device links.

---

#### `fwnode_init`

```c
static inline void fwnode_init(struct fwnode_handle *fwnode,
                               const struct fwnode_operations *ops)
```

**بتعمل إيه:**
بتربط الـ `ops` vtable بالـ `fwnode_handle` وبتهيّئ الـ `suppliers` و`consumers` lists باستخدام `INIT_LIST_HEAD`. من غير الاستدعاء ده، أي محاولة إنشاء device link هتفشل لأن الـ lists مش initialized.

**Parameters:**
- `fwnode` — الـ handle اللي هيتهيّأ، بيكون embedded جوه struct أكبر زي `of_node` أو `acpi_device`.
- `ops` — pointer على الـ `fwnode_operations` vtable الخاصة بالـ firmware backend.

**Return:** void

**Key details:**
- لا يوجد locking — بيتّستدعى في سياق initialization قبل ما الـ node يبقى visible للنظام.
- بعد الاستدعاء، الـ `fwnode->dev` بيفضل `NULL` وبيتّعبّى بعدين من `device_add()`.
- الـ `flags` field بيفضل zero — كل الـ flags ماشية على default غير set.

**Who calls it:**
بيتّستدعى من firmware backends عند تهيئة node جديد — مثلاً `of_node_init()` في OF، أو `acpi_init_device_object()` في ACPI، أو `software_node_register()`.

---

#### `fwnode_dev_initialized`

```c
static inline void fwnode_dev_initialized(struct fwnode_handle *fwnode,
                                          bool initialized)
```

**بتعمل إيه:**
بتعدّل الـ `FWNODE_FLAG_INITIALIZED` flag في الـ `fwnode->flags`. الـ flag ده بيشير إن الـ hardware المقابل للـ node اتّهيأ فعلاً. الـ device core بيستخدمه عشان يعرف امتى يبدأ يبني الـ supplier/consumer ordering.

**Parameters:**
- `fwnode` — الـ handle المستهدف. لو `IS_ERR_OR_NULL` بيرجع فوراً بدون crash.
- `initialized` — `true` يعمل set للـ flag، `false` يعمل clear.

**Return:** void

**Key details:**
- الـ guard على `IS_ERR_OR_NULL` مهم جداً — بعض الـ paths بتمرر NULL fwnode وده expected.
- الـ flag ده بيأثر على الـ `fw_devlink` engine: node مش initialized ممكن يسبب defer للـ probe.
- مفيش locking هنا — المفروض يتّستدعى من سياق synchronous زي `probe()` أو `remove()`.

**Who calls it:**
`really_probe()` في `drivers/base/dd.c` بيستدعيه بـ `true` بعد `probe()` ينجح، وبـ `false` بعد `remove()`.

---

### Group 2: Dispatch Macros — الـ vtable Call Layer

الـ group ده بيوفر abstraction layer موحدة فوق الـ `fwnode_operations` vtable. بدلاً من كل caller يعمل check يدوي على الـ pointer، الـ macros دي بتتكلف كل الـ NULL/ERR checking وبترجع قيم افتراضية مناسبة.

---

#### `fwnode_has_op`

```c
#define fwnode_has_op(fwnode, op) \
    (!IS_ERR_OR_NULL(fwnode) && (fwnode)->ops && (fwnode)->ops->op)
```

**بيعمل إيه:**
بيتحقق إن الـ `fwnode` مش NULL أو error pointer، وإن الـ `ops` vtable موجود، وإن الـ op المحدد مش NULL. الثلاثة conditions لازم تتحقق.

**Parameters:**
- `fwnode` — الـ handle المستهدف.
- `op` — اسم الـ field في `fwnode_operations` (بدون `->` أو `()`).

**Return:** `bool` — الـ result من الـ compound condition.

**Key details:**
- المفروض يتّستخدم كـ guard قبل الـ dispatch، أو بشكل مباشر لو الكود محتاج يعمل early-exit.
- الـ `IS_ERR_OR_NULL` بيحمي من الـ error pointer patterns اللي شايعة في الـ OF/ACPI code.

---

#### `fwnode_call_int_op`

```c
#define fwnode_call_int_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : \
     (IS_ERR_OR_NULL(fwnode) ? -EINVAL : -ENXIO))
```

**بيعمل إيه:**
لو الـ op موجود بيستدعيه ويرجع قيمته. لو الـ `fwnode` نفسه invalid بيرجع `-EINVAL`. لو الـ `fwnode` valid بس الـ op مش مُنفَّذ بيرجع `-ENXIO` — ده تمييز مهم جداً.

**Parameters:**
- `fwnode` — الـ handle.
- `op` — اسم الـ operation في الـ vtable.
- `...` — arguments إضافية بتتمرر للـ op مباشرة (بعد الـ fwnode الأول).

**Return:** `int` — قيمة الـ op، أو `-EINVAL`/`-ENXIO` لو مفيش op.

**Key details:**
- الفرق بين `-EINVAL` و`-ENXIO`: الأول فشل الـ handle نفسه، التاني الـ backend مش بيدعم الـ operation دي.
- الـ `## __VA_ARGS__` مع الـ `##` بيحل مشكلة trailing comma لو مفيش arguments إضافية.

**Pseudocode flow:**
```
fwnode_call_int_op(fwnode, property_read_int_array, name, 4, buf, n):
  if fwnode_has_op(fwnode, property_read_int_array):
    return fwnode->ops->property_read_int_array(fwnode, name, 4, buf, n)
  elif IS_ERR_OR_NULL(fwnode):
    return -EINVAL
  else:
    return -ENXIO   /* backend doesn't implement this op */
```

---

#### `fwnode_call_bool_op`

```c
#define fwnode_call_bool_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : false)
```

**بيعمل إيه:**
بيستدعي op يرجع `bool`، ولو مش موجود يرجع `false` كـ safe default.

**Return:** `bool`.

**Key details:**
- الـ `false` default هنا منطقي — "هل property موجودة؟" الجواب الافتراضي هو "لا".
- بيتّستخدم مع ops زي `device_is_available`، `device_dma_supported`، `property_present`، `property_read_bool`.

---

#### `fwnode_call_ptr_op`

```c
#define fwnode_call_ptr_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : NULL)
```

**بيعمل إيه:**
بيستدعي op يرجع pointer، ولو مش موجود يرجع `NULL`.

**Return:** `pointer` أو `NULL`.

**Key details:**
- الـ NULL return بيتوافق مع convention الـ iteration: `get_next_child_node` بترجع NULL لو مفيش children.
- بيتّستخدم مع: `get_parent`، `get_next_child_node`، `get_named_child_node`، `graph_get_next_endpoint`، `graph_get_remote_endpoint`، `graph_get_port_parent`.

---

#### `fwnode_call_void_op`

```c
#define fwnode_call_void_op(fwnode, op, ...) \
    do { \
        if (fwnode_has_op(fwnode, op)) \
            (fwnode)->ops->op(fwnode, ## __VA_ARGS__); \
    } while (false)
```

**بيعمل إيه:**
بيستدعي op لا يرجع قيمة (مثل `put`)، مع الـ do-while(false) pattern للـ macro hygiene.

**Return:** void.

**Key details:**
- الـ `do { } while (false)` ضروري عشان المـ macro يتصرف صح مع `if/else` بدون braces.
- بيتّستخدم مع: `put`.

---

### Group 3: fwnode_operations vtable — شرح كل Op

الـ vtable دي هي قلب الـ firmware node abstraction. كل firmware backend (OF، ACPI، swnode) بينفّذ subset منها. الـ ops اللي مش موجودة بترجع قيم افتراضية عبر الـ dispatch macros.

#### Lifecycle Ops: `get` و`put`

```c
struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
void (*put)(struct fwnode_handle *fwnode);
```

**بيعملوا إيه:**
بيديروا الـ reference counting على الـ fwnode object. `get` بيزود الـ refcount ويرجع نفس الـ pointer. `put` بينقص الـ refcount وممكن يحرر الـ memory.

**Key details:**
- الـ OF implementation: `of_node_get()` / `of_node_put()` — بيتبعوا الـ `kobject` refcount.
- الـ ACPI implementation: بيزودوا/بينقصوا الـ `acpi_handle` reference.
- الـ swnode implementation: لا يعمل حاجة (static nodes).
- لازم كل iterator يعمل `put` على الـ node اللي جابه لو مش هيتابع الـ iteration.

---

#### Availability Op: `device_is_available`

```c
bool (*device_is_available)(const struct fwnode_handle *fwnode);
```

**بيعمل إيه:**
بيتحقق من الـ firmware description إن الـ device ده enabled. في الـ DT بيشوف الـ `status` property: `"okay"` أو absent يعني available. في ACPI بيشوف `_STA` method.

**Return:** `true` لو available، `false` لو disabled أو mismatch.

**Key details:**
- بيتّستدعى من `of_device_is_available()` و`acpi_device_is_present()`.
- الـ probe path بيقف لو الـ device مش available.

---

#### Match Data Op: `device_get_match_data`

```c
const void *(*device_get_match_data)(const struct fwnode_handle *fwnode,
                                     const struct device *dev);
```

**بيعمل إيه:**
بيرجع الـ driver-specific data المرتبطة بالـ compatible string أو الـ ACPI ID. الـ OF implementation بتبحث في الـ `of_device_id` table وترجع الـ `data` field. بيُستخدم في drivers لقراءة chip-specific configuration من `of_match_table`.

**Parameters:**
- `fwnode` — الـ device node.
- `dev` — الـ struct device المقابل، محتاجه عشان يوصل لـ driver's `of_match_table`.

**Return:** `const void *` — pointer للـ match data، أو `NULL` لو مفيش match.

---

#### DMA Ops: `device_dma_supported` و`device_get_dma_attr`

```c
bool (*device_dma_supported)(const struct fwnode_handle *fwnode);
enum dev_dma_attr (*device_get_dma_attr)(const struct fwnode_handle *fwnode);
```

**بيعملوا إيه:**
الأول بيتحقق إن الـ device capable of DMA. التاني بيرجع نوع الـ coherency:

| قيمة | معنى |
|---|---|
| `DEV_DMA_NOT_SUPPORTED` | الـ device مش بيدعم DMA |
| `DEV_DMA_NON_COHERENT` | DMA بدون hardware coherency — محتاج cache ops |
| `DEV_DMA_COHERENT` | DMA مع hardware cache coherency |

**Key details:**
- الـ `of_dma_is_coherent()` بيقرأ الـ `dma-coherent` property من الـ DT.
- النتيجة بتأثر على الـ `dma_map_ops` اللي بيتّستخدم مع الـ device.

---

#### Property Ops

```c
bool (*property_present)(const struct fwnode_handle *fwnode,
                         const char *propname);

bool (*property_read_bool)(const struct fwnode_handle *fwnode,
                           const char *propname);

int (*property_read_int_array)(const struct fwnode_handle *fwnode,
                               const char *propname,
                               unsigned int elem_size, void *val,
                               size_t nval);

int (*property_read_string_array)(const struct fwnode_handle *fwnode_handle,
                                  const char *propname, const char **val,
                                  size_t nval);
```

**بيعملوا إيه:**
- `property_present`: بيتحقق وجود property بالاسم بس — مش بيقرأ قيمتها.
- `property_read_bool`: بيقرأ property boolean — في الـ DT property موجودة = true، absent = false.
- `property_read_int_array`: بيقرأ array من integers. `elem_size` بيحدد حجم كل عنصر (1, 2, 4, أو 8 bytes). لو `val == NULL` بيرجع عدد العناصر.
- `property_read_string_array`: بيقرأ array من strings. لو `val == NULL` بيرجع عدد الـ strings.

**Parameters (property_read_int_array):**
- `propname` — اسم الـ property.
- `elem_size` — حجم كل عنصر بالـ bytes (sizeof(u8/u16/u32/u64)).
- `val` — buffer للنتيجة، أو `NULL` لاستعلام العدد فقط.
- `nval` — عدد العناصر المطلوبة.

**Return:** `0` لو نجح، عدد العناصر لو `val == NULL`، قيمة سالبة للخطأ.

**Key details:**
- الـ `elem_size` abstraction مهم جداً — بيسمح لنفس الـ op تقرأ `u8` و`u32` و`u64` arrays.
- في الـ OF implementation، `__of_property_read_u32_index()` وإخواتها بتنفّذ الـ parsing.
- Thread safety: الـ OF nodes محمية بـ `of_node_lock` — الـ ops دي بتاخد الـ lock داخلياً.

---

#### Naming Ops: `get_name` و`get_name_prefix`

```c
const char *(*get_name)(const struct fwnode_handle *fwnode);
const char *(*get_name_prefix)(const struct fwnode_handle *fwnode);
```

**بيعملوا إيه:**
`get_name` بيرجع اسم الـ node في الـ firmware tree (مثلاً `"i2c@3c000000"`). `get_name_prefix` بيرجع prefix للـ print (مثلاً `"/"` في الـ OF أو `"\"` في الـ ACPI).

**Return:** `const char *` — string ثابتة، مش محتاج تـ free.

**Key details:**
- الـ strings دي owned بالـ firmware backend — مش mutable ومش بتتحرر.
- بيتّستخدموا في `dev_err()` messages وفي sysfs naming.

---

#### Tree Traversal Ops: `get_parent`، `get_next_child_node`، `get_named_child_node`

```c
struct fwnode_handle *(*get_parent)(const struct fwnode_handle *fwnode);

struct fwnode_handle *
(*get_next_child_node)(const struct fwnode_handle *fwnode,
                       struct fwnode_handle *child);

struct fwnode_handle *
(*get_named_child_node)(const struct fwnode_handle *fwnode,
                        const char *name);
```

**بيعملوا إيه:**
- `get_parent`: بيرجع parent node مع ref bump.
- `get_next_child_node`: بيعمل iteration على الـ children. لو `child == NULL` بيبدأ من الأول. كل مرة بيبقى الـ caller مسؤول عن الـ `put` على الـ `child` السابق.
- `get_named_child_node`: بيدور على child باسم محدد مع ref bump.

**Return:** `fwnode_handle *` أو `NULL` في نهاية الـ iteration.

**Key details:**
- الـ pattern الصح للـ iteration:
  ```c
  child = NULL;
  while ((child = fwnode_call_ptr_op(parent, get_next_child_node, child)))
      process(child);
  /* last child already put by next call returning NULL */
  ```
- في الـ OF implementation، `of_get_next_available_child()` بتتخطى الـ disabled nodes.

---

#### Reference Op: `get_reference_args`

```c
int (*get_reference_args)(const struct fwnode_handle *fwnode,
                          const char *prop,
                          const char *nargs_prop,
                          unsigned int nargs,
                          unsigned int index,
                          struct fwnode_reference_args *args);
```

**بتعمل إيه:**
بتـ resolve reference property زي `clocks`، `interrupts`، `dmas` في الـ DT، أو الـ ACPI equivalent. بترجع الـ fwnode المُشار إليه والـ arguments المصاحبة (زي `clock-cells`، `interrupt-cells`).

**Parameters:**
- `prop` — اسم الـ property اللي فيها الـ phandle (مثلاً `"clocks"`).
- `nargs_prop` — اسم الـ property اللي بتحدد عدد الـ args (مثلاً `"#clock-cells"`). ممكن تبقى `NULL`.
- `nargs` — عدد الـ args المتوقعة لو `nargs_prop == NULL`.
- `index` — index الـ reference لو في multiple references في نفس الـ property.
- `args` — struct للنتيجة.

**Return:** `0` لو نجح، قيمة سالبة للخطأ.

**Key details:**
- `struct fwnode_reference_args` بيحتوي على `fwnode` مع ref bump + array من `u64` args (max `NR_FWNODE_REFERENCE_ARGS = 16`).
- الـ caller مسؤول عن `fwnode_handle_put(args.fwnode)` بعد الاستخدام.
- بيتّستدعى من `fwnode_property_get_reference_args()` في `drivers/base/property.c`.

**Pseudocode flow:**
```
get_reference_args(fwnode, "clocks", "#clock-cells", 0, 0, &args):
  1. قراءة property "clocks" من الـ firmware node
  2. get phandle من index 0
  3. resolve phandle → target fwnode_handle
  4. قراءة "#clock-cells" من الـ target node → nargs
  5. قراءة nargs u32 values من الـ property بعد الـ phandle
  6. ملء args->fwnode + args->args[] + args->nargs
  7. return 0
```

---

#### Graph Ops: `graph_get_next_endpoint`، `graph_get_remote_endpoint`، `graph_get_port_parent`، `graph_parse_endpoint`

```c
struct fwnode_handle *
(*graph_get_next_endpoint)(const struct fwnode_handle *fwnode,
                           struct fwnode_handle *prev);

struct fwnode_handle *
(*graph_get_remote_endpoint)(const struct fwnode_handle *fwnode);

struct fwnode_handle *
(*graph_get_port_parent)(struct fwnode_handle *fwnode);

int (*graph_parse_endpoint)(const struct fwnode_handle *fwnode,
                            struct fwnode_endpoint *endpoint);
```

**بيعملوا إيه:**
الـ ops دي بتدير الـ **OF graph** (V4L2 media graph / camera pipeline). الـ topology:

```
device_node
  └── port@0
        └── endpoint@0  ←→  endpoint@1
                              └── port@1
                                    └── device_node (remote)
```

- `graph_get_next_endpoint`: iteration على كل الـ endpoints في الـ device (عبر كل الـ ports).
- `graph_get_remote_endpoint`: من local endpoint يرجع الـ endpoint المقابل على الـ remote device.
- `graph_get_port_parent`: من port node يرجع الـ device node الأب.
- `graph_parse_endpoint`: يقرأ `port` و`id` من endpoint node ويملأ `struct fwnode_endpoint`.

**Parameters (graph_parse_endpoint):**
- `fwnode` — الـ endpoint node.
- `endpoint` — struct النتيجة: `{ .port, .id, .local_fwnode }`.

**Return:** `0` أو قيمة سالبة للخطأ.

**Key details:**
- الـ `SWNODE_GRAPH_PORT_NAME_FMT = "port@%u"` و`SWNODE_GRAPH_ENDPOINT_NAME_FMT = "endpoint@%u"` هي الـ naming convention الإلزامية للـ software nodes.
- الـ OF implementation: `of_graph_get_next_endpoint()` بتمشي عبر الـ ports tree.
- بيتّستخدموا كتير في الـ V4L2، CSI، MIPI DSI، sound card drivers.

---

#### Resource Ops: `iomap` و`irq_get`

```c
void __iomem *(*iomap)(struct fwnode_handle *fwnode, int index);
int (*irq_get)(const struct fwnode_handle *fwnode, unsigned int index);
```

**بيعملوا إيه:**
- `iomap`: بيعمل map لـ MMIO register range بالـ index من الـ firmware description وبيرجع `void __iomem *`. في الـ PCI fwnodes بيستدعي `pci_iomap_range()`.
- `irq_get`: بيرجع Linux virtual IRQ number من الـ firmware IRQ description بالـ index. في الـ OF بيستدعي `of_irq_get()`.

**Parameters:**
- `index` — رقم الـ resource (اللي الـ driver طالبه من 0 إلى N-1).

**Return:**
- `iomap`: `void __iomem *` أو `IOMEM_ERR_PTR(errno)` للخطأ.
- `irq_get`: Linux IRQ number (양수) أو قيمة سالبة للخطأ.

**Key details:**
- الـ `iomap` op موجودة أساساً للـ PCI fwnodes — platform devices عادةً بتستخدم `platform_get_resource()` مباشرة.
- `irq_get` بيتّستخدم من `fwnode_irq_get()` في `drivers/base/property.c`.

---

#### Links Op: `add_links`

```c
int (*add_links)(struct fwnode_handle *fwnode);
```

**بتعمل إيه:**
بتقرأ الـ firmware description وبتنشئ `fwnode_link` objects بين الـ consumer (fwnode الحالي) وكل الـ suppliers المذكورين (مثلاً `clocks`، `power-domains`، `iommus`، `pinctrl`). الـ device core بيستخدم الـ links دي لبناء الـ probe ordering.

**Parameters:**
- `fwnode` — الـ consumer node اللي هيتحلل suppliers بتاعه.

**Return:** `0` لو نجح، قيمة سالبة للخطأ (مثلاً لو supplier node مش موجود بعد).

**Key details:**
- بيتّستدعى من `fwnode_link_add()` context أثناء device registration.
- الـ FWNODE_FLAG_LINKS_ADDED بيتّسيت بعد نجاحه عشان منيتعملش add مرتين.
- الـ FWNODE_FLAG_BEST_EFFORT بيأثر على السلوك: مش هيـ defer الـ probe لو supplier مش موجود وعنده driver.

**Pseudocode flow:**
```
add_links(consumer_fwnode):
  for each property in {clocks, power-domains, iommus, pinctrl-N, ...}:
      for each phandle reference in property:
          supplier = resolve_phandle(ref)
          if supplier found:
              fwnode_link_add(consumer_fwnode, supplier, 0)
          else:
              record missing supplier
  set FWNODE_FLAG_LINKS_ADDED
  return 0 or -ENODEV
```

---

### Group 4: Link Management — إدارة الـ fwnode_links

الـ group ده هو الـ runtime management للـ `fwnode_link` objects اللي بتحدد الـ supplier/consumer relationships بين الـ nodes. الـ `fw_devlink` engine بيبني عليها الـ device links الحقيقية.

---

#### `fwnode_link_add`

```c
int fwnode_link_add(struct fwnode_handle *con, struct fwnode_handle *sup,
                    u8 flags);
```

**بتعمل إيه:**
بتنشئ `struct fwnode_link` جديدة وبتربط الـ consumer node بالـ supplier node. الـ link بتتضاف في قائمتين: `sup->consumers` و`con->suppliers`. لو الـ link موجودة بالفعل بتـ update الـ flags وترجع `0`.

**Parameters:**
- `con` — الـ consumer fwnode (الـ device اللي محتاج الـ supplier).
- `sup` — الـ supplier fwnode (الـ device اللي بيوفر الـ resource).
- `flags` — مجموعة من:
  - `FWLINK_FLAG_CYCLE (BIT(0))`: الـ link جزء من cycle — مش هنـ defer الـ probe بسببه.
  - `FWLINK_FLAG_IGNORE (BIT(1))`: تجاهل الـ link تماماً حتى في الـ cycle detection.

**Return:** `0` لو نجح (أو موجود)، قيمة سالبة للخطأ (مثلاً `-ENOMEM`).

**Key details:**
- بيتّستدعى من `add_links` op (أو من `fw_devlink` code مباشرة).
- الـ locking: محتاج يتّستدعى مع الـ `fwnode_links_lock` مأخوذ — راجع `drivers/base/core.c`.
- الـ link هتتحول لـ `device_link` حقيقية بعدين لما الـ struct devices تتّسجل.

---

#### `fwnode_links_purge`

```c
void fwnode_links_purge(struct fwnode_handle *fwnode);
```

**بتعمل إيه:**
بتحذف كل الـ `fwnode_link` objects اللي فيها الـ fwnode ده سواء كـ supplier أو كـ consumer. بتمشي على الـ `fwnode->suppliers` list وعلى الـ `fwnode->consumers` list وبتحرر كل `fwnode_link` بـ `kfree`.

**Parameters:**
- `fwnode` — الـ node اللي هتتحذف links بتاعته كلها.

**Return:** void

**Key details:**
- بيتّستدعى أثناء فصل الـ fwnode عن النظام — مثلاً في `fwnode_handle_put()` لما الـ refcount يوصل صفر.
- بعد الاستدعاء، الـ `suppliers` و`consumers` lists بتبقى empty lists صح (مش dangling).
- خطر: لو فيه thread تاني بيتمشى على الـ lists في نفس الوقت، لازم يكون في locking مناسب.

---

#### `fw_devlink_purge_absent_suppliers`

```c
void fw_devlink_purge_absent_suppliers(struct fwnode_handle *fwnode);
```

**بتعمل إيه:**
بتمشي على الـ consumers list بتاعة الـ fwnode وبتحذف الـ links اللي الـ supplier بتاعها مش موجود كـ struct device (يعني موجود كـ firmware node بس مش registered كـ device). ده بيمنع الـ infinite probe deferral للـ devices اللي بتتكلم على hardware غير موجود.

**Parameters:**
- `fwnode` — الـ supplier fwnode.

**Return:** void

**Key details:**
- بيتّستدعى لما `fw_devlink` يلاقي supplier node موجود في الـ DT/ACPI بس مش بيتحول لـ device.
- بيشوف الـ `FWNODE_FLAG_NOT_DEVICE` flag كمؤشر إن الـ node مش هيبقى device أبداً.
- الـ use case: nodes زي `cpus`، `memory`، `reserved-memory` موجودة في الـ DT بس مش devices حقيقية.

---

#### `fw_devlink_is_strict`

```c
bool fw_devlink_is_strict(void);
```

**بتعمل إيه:**
بترجع `true` لو الـ `fw_devlink` kernel parameter مضبوط على `"strict"` أو أعلى. في الـ strict mode، كل الـ `fwnode_link` references بتتحول لـ `device_link` إلزامية — يعني الـ consumer مش هيتـ probe إلا بعد الـ supplier.

**Parameters:** لا يوجد.

**Return:** `bool`.

**Key details:**
- الـ default في معظم kernel configs هو `"permissive"` أو `"off"`.
- الـ strict mode مفيد للـ embedded systems اللي عايزة probe ordering صارم.
- بيتّستدعى من `fw_devlink` logic في `drivers/base/core.c` لقرار إنشاء الـ device links.
- الـ FWNODE_FLAG_BEST_EFFORT بيـ override الـ strict mode للـ nodes اللي محتاجة تـ probe early.

---

### ملاحظة ختامية: العلاقة بين الـ Groups

```
firmware source (DT/ACPI/swnode)
         │
         ▼
  fwnode_handle ←─── fwnode_init()
         │               └── sets ops vtable
         │               └── INIT_LIST_HEAD(suppliers/consumers)
         │
         ├──[ops vtable]──► property_read_*  (قراءة configuration)
         │                ► graph_get_*      (media pipeline)
         │                ► get_reference_args (phandle resolution)
         │                ► add_links        (build dependency graph)
         │
         ▼
  fwnode_link_add()  ──► fwnode_link{con→sup}
         │
         ▼
  fw_devlink engine  ──► device_link (runtime PM + probe ordering)
         │
         └── fwnode_links_purge()  (cleanup on remove)
```
## Phase 5: دليل الـ Debugging الشامل

الـ `fwnode` subsystem هو الطبقة اللي بتوحد الوصول لـ firmware metadata سواء جاية من **Device Tree (OF)**، **ACPI**، أو **software_node**. المشاكل بتظهر غالباً في probe failures، missing properties، أو broken device links.

---

### Software Level

#### 1. debugfs Entries

الـ debugfs بيكشف معلومات الـ device links والـ fwnode graph:

```bash
# Mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل device links (supplier → consumer)
ls /sys/kernel/debug/device_component/
cat /sys/kernel/debug/device_component/<driver_name>

# شوف الـ fw_devlink graph كامل
cat /sys/kernel/debug/fw_devlink/graph 2>/dev/null

# فحص الـ deferred probes
cat /sys/kernel/debug/devices_deferred
```

**تفسير الـ output:**

```
/sys/kernel/debug/devices_deferred
platform:my_device    # اسم الـ device اللي اتأخر probe-ه
  Driver: my_driver   # الـ driver المسؤول
  Waiting for: supplier_device  # اللي بينتظره
```

الـ `fw_devlink/graph` بيفرق بين links بـ `CYCLE` flag (مش هيسبب defer) وlinks عادية.

---

#### 2. sysfs Entries

```bash
# شوف الـ fwnode المرتبطة بـ device معين
ls -la /sys/bus/platform/devices/<device_name>/

# الـ of_node symlink (لو DT)
ls -la /sys/bus/platform/devices/<device>/of_node
# → يشاور على /sys/firmware/devicetree/base/...

# شوف كل properties الـ DT node
ls /sys/firmware/devicetree/base/<path_to_node>/
cat /sys/firmware/devicetree/base/<path_to_node>/compatible

# فحص الـ supplier/consumer links
ls /sys/bus/platform/devices/<device>/supplier:*/
ls /sys/bus/platform/devices/<device>/consumer:*/

# شوف status الـ device link
cat /sys/bus/platform/devices/<device>/supplier:<supplier>/auto_remove_on
cat /sys/bus/platform/devices/<device>/supplier:<supplier>/status
```

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# Enable function tracing للـ fwnode subsystem
cd /sys/kernel/debug/tracing

# تتبع كل دوال الـ property reading
echo "fwnode_property_read*" > set_ftrace_filter
echo "device_property_read*" >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
# ... شغّل الـ driver ...
cat trace

# استخدم trace events للـ device links
echo 1 > events/devlink/enable
echo 1 > tracing_on
# ثم اقرأ الـ trace
cat trace | grep devlink

# تتبع دوال الـ probe path مع الـ fwnode
echo "really_probe" > set_ftrace_filter
echo "driver_probe_device" >> set_ftrace_filter
echo "fwnode_link_add" >> set_ftrace_filter
echo function_graph > current_tracer
echo 1 > tracing_on
```

**مثال output مفيد:**

```
 1) | really_probe() {
 1) | fwnode_link_add() {    /* supplier link being added */
 1)   0.312 us    |   }
 1) + 15.231 us   | }
```

---

#### 4. printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل ملفات الـ fwnode subsystem
echo "file drivers/base/property.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/fwnode.c +p" >> /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/core.c +p" >> /sys/kernel/debug/dynamic_debug/control

# أو فعّل بـ module name (لو driver معين)
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل الـ fw_devlink debug messages
echo "file drivers/base/devlink.c +pmfl" > /sys/kernel/debug/dynamic_debug/control
# p = printk, m = module name, f = function, l = line

# شوف إيه اللي enabled حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep "property\|fwnode\|devlink"

# من kernel cmdline (early boot)
# أضف للـ bootargs:
dyndbg="file drivers/base/property.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_DRIVER` | يفعّل pr_debug في drivers/base كلها |
| `CONFIG_ACPI_DEBUG` | يطبع ACPI namespace و method eval |
| `CONFIG_OF_DYNAMIC` | يسمح بتعديل DT في runtime (testing) |
| `CONFIG_PROVE_LOCKING` | يكتشف locking bugs في الـ fwnode ops |
| `CONFIG_DEBUG_DEVRES` | يتبع managed resources (devm_*) |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل pr_debug الـ selective |
| `CONFIG_SOFTLOCKUP_DETECTOR` | يكتشف lockups أثناء الـ probe |
| `CONFIG_FWNODE_MDIO` | debugging الـ MDIO fwnode graph |
| `CONFIG_OF_UNITTEST` | unit tests لـ OF/fwnode layer |
| `CONFIG_ACPI_TABLE_UPGRADE` | تحديث ACPI tables في runtime |

```bash
# تأكد إن الـ config options موجودة
zcat /proc/config.gz | grep -E "CONFIG_DEBUG_DRIVER|CONFIG_OF_DYNAMIC|CONFIG_DYNAMIC_DEBUG"
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# أداة dtc لفحص الـ Device Tree
dtc -I fs /sys/firmware/devicetree/base > /tmp/current_dt.dts
# ثم ابحث عن الـ node اللي عندك مشكلة فيه
grep -A 20 "my_device" /tmp/current_dt.dts

# فحص ACPI tables
acpidump > /tmp/acpi.dat
acpixtract -a /tmp/acpi.dat
iasl -d DSDT.dat   # يطلع DSDT.dsl قابل للقراءة
grep -i "my_device" DSDT.dsl

# devlink tool (iproute2) — مش نفس الـ kernel fw_devlink
# لكن لفحص الـ fwnode لـ network devices:
devlink dev show
devlink dev info pci/0000:01:00.0

# فحص الـ software_node
ls /sys/kernel/debug/software_nodes/ 2>/dev/null

# سكريبت لطباعة كل fwnode properties لـ device
for f in /sys/firmware/devicetree/base/soc/my_device/*; do
    echo -n "$(basename $f): "
    xxd $f 2>/dev/null || cat $f 2>/dev/null
done
```

---

#### 7. رسائل الـ Error الشائعة → المعنى → الحل

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `probe deferral - supplier X not ready` | الـ consumer بيحاول يعمل probe قبل الـ supplier | تأكد إن الـ supplier device/driver اتحمل، فحص `devices_deferred` |
| `fwnode_property_read_u32: -EINVAL` | الـ property موجودة لكن نوعها غلط | تأكد من نوع الـ property في DTS (u32 مش string) |
| `fwnode_property_read_u32: -ENODATA` | الـ property مش موجودة خالص | تأكد من اسم الـ property في DTS/ACPI |
| `fwnode_handle is NULL` | الـ device مش مربوط بـ fwnode | تأكد إن `dev->fwnode` اتضبط صح في platform data |
| `ERROR: Bad cell-index for <node>` | الـ phandle reference فيها index برا الـ range | فحص عدد الـ `#xxx-cells` في الـ parent node |
| `not creating device link ... no supplier found` | الـ `fw_devlink` مش لاقي الـ supplier node | اسم الـ supplier في DT مش متطابق مع الـ device المسجل |
| `graph_get_remote_endpoint: NULL` | endpoint الـ graph مش متوصل لـ remote | فحص `remote-endpoint` phandle في DTS |
| `-ENXIO from fwnode_call_int_op` | الـ fwnode مش NULL لكن الـ op نفسها مش موجودة | الـ fwnode_operations struct ناقصها الـ function pointer دي |
| `supplier X prevented from binding` | `FWNODE_FLAG_BEST_EFFORT` مش set والـ supplier مش جاهز | إما تضيف الـ flag أو تصلح ترتيب الـ probe |
| `cycle detected in fwnode link` | دورة في الـ supplier/consumer graph | فحص الـ DT، ممكن تضيف `FWLINK_FLAG_IGNORE` بحذر |

---

#### 8. مواضع `dump_stack()` و`WARN_ON()`

أماكن استراتيجية تحط فيها debug assertions:

```c
/* في fwnode_init() — تتأكد إن الـ ops مش NULL */
static inline void fwnode_init(struct fwnode_handle *fwnode,
                               const struct fwnode_operations *ops)
{
    WARN_ON(!ops);          /* لازم يكون عنده ops */
    fwnode->ops = ops;
    INIT_LIST_HEAD(&fwnode->consumers);
    INIT_LIST_HEAD(&fwnode->suppliers);
}

/* في fwnode_link_add() — تتأكد من عدم وجود cycles */
int fwnode_link_add(struct fwnode_handle *con, struct fwnode_handle *sup,
                    u8 flags)
{
    WARN_ON(con == sup);    /* الـ device مش هيكون supplier لنفسه */
    /* ... */
}

/* في driver probe — تتأكد من الـ fwnode قبل ما تستخدمه */
static int my_driver_probe(struct platform_device *pdev)
{
    struct fwnode_handle *fwnode = dev_fwnode(&pdev->dev);

    if (WARN_ON(!fwnode)) {
        /* dump_stack هنا يساعد تعرف مين استدعى الـ probe */
        dump_stack();
        return -ENODEV;
    }
    /* ... */
}

/* لو بتنفذ fwnode_operations — تتأكد من valid state */
static int mydrv_fwnode_property_read_int_array(
    const struct fwnode_handle *fwnode,
    const char *propname,
    unsigned int elem_size, void *val, size_t nval)
{
    struct mydrv_data *data = container_of(fwnode, struct mydrv_data, fwnode);
    WARN_ON(!data->initialized);  /* الـ hardware لازم يكون initialized */
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من إن حالة الـ Hardware بتطابق حالة الـ Kernel

```bash
# تأكد من إن الـ device ظهر للـ kernel
dmesg | grep "my_device\|my_driver"

# تحقق من الـ FWNODE_FLAG_INITIALIZED
# من داخل kernel (بـ printk مؤقت):
# pr_info("flags: 0x%x\n", fwnode->flags);

# فحص إن الـ device available (status = "okay" في DT)
cat /sys/firmware/devicetree/base/soc/my_device/status
# يجيب: okay

# تحقق من device availability عبر sysfs
cat /sys/bus/platform/devices/my_device/of_node/status

# تأكد من الـ DMA coherency
cat /sys/bus/platform/devices/my_device/dma_coherent 2>/dev/null
```

---

#### 2. Register Dump Techniques

```bash
# devmem2: اقرأ register بعنوان فيزيائي
# تثبيت: apt install devmem2 أو compile من source
devmem2 0xFE200000 w    # اقرأ 32-bit word من العنوان ده

# /dev/mem (لو CONFIG_DEVMEM=y والـ strict mode مش enabled)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE200000 / 4)) 2>/dev/null | xxd

# io utility (من package iotools)
io -4 0xFE200000         # قراءة 32-bit

# لو MMIO عبر /proc/iomem، اعرف العناوين أولاً
cat /proc/iomem | grep "my_device\|my_controller"
# ثم استخدم devmem2 بالعنوان الصح

# لـ PCIe devices، اقرأ config space
lspci -vvv -s 01:00.0
setpci -s 01:00.0 0x04.w   # قراءة Command register
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ I2C fwnode graph:**
- راقب SDA/SCL على الـ oscilloscope
- الـ START condition: SDA ينزل وهو SCL عالي
- لو الـ driver قرأ property `clock-frequency` غلط → هتشوف timing مختلف
- تأكد إن الـ `reg` property في DTS بتطابق الـ hardware address على الـ bus

**للـ SPI:**
- راقب CS، SCLK، MOSI، MISO
- لو `spi-max-frequency` property اتقرأت غلط → هتشوف clock سريع/بطيء أكتر من اللازم

**للـ GPIO-based fwnode:**
- استخدم `gpioinfo` و`gpioget` لقبل ما تشيل الـ driver
- قارن قراءة GPIO مع الـ logic analyzer

```bash
# فحص GPIO assignments
gpioinfo gpiochip0
# شوف إيه الـ pin المرتبط بالـ fwnode
cat /sys/firmware/devicetree/base/my_device/gpios | xxd
```

---

#### 4. Hardware Issues الشائعة → Kernel Log Patterns

| المشكلة الـ hardware | الـ pattern في kernel log |
|---|---|
| الـ device مش موجود على البورد أو مش متوصل | `probe of my_device failed with error -ENODEV` |
| الـ IRQ line مش متوصل | `IRQ X: nobody cared` أو `nobody cared (try booting with irqpoll)` |
| الـ power rail مش شغال | `timeout waiting for device to become ready` |
| الـ clock source مش configured | `could not get clk: -ENOENT` |
| عنوان I2C غلط في DTS | `i2c i2c-0: no device found at 0xXX` |
| الـ reg property بتشاور على عنوان برا الـ parent range | `translation of ... failed` أو `bad address` |
| شورت في الـ SDA line | `i2c transfer timed out` بشكل متكرر |

---

#### 5. Device Tree Debugging

```bash
# اقرأ الـ DTB الـ current المستخدم من الـ kernel
dtc -I fs /sys/firmware/devicetree/base -O dts -o /tmp/live_dt.dts 2>/dev/null
less /tmp/live_dt.dts

# قارن مع الـ DTB اللي compile-ت منه
dtc -I dtb -O dts my_board.dtb -o /tmp/compiled_dt.dts
diff /tmp/live_dt.dts /tmp/compiled_dt.dts

# تحقق من الـ phandle references محلولة صح
# كل &label في DTS بيتحول لـ phandle number
grep -A5 "my_device" /tmp/live_dt.dts

# تحقق من الـ #address-cells و#size-cells
cat /sys/firmware/devicetree/base/soc/\#address-cells | xxd
# لازم تطابق عدد الـ values في reg property

# فحص الـ interrupts property
cat /sys/firmware/devicetree/base/my_device/interrupts | xxd
# عدد الـ bytes = عدد الـ interrupts × #interrupt-cells × 4

# تحقق من الـ overlays (لو بتستخدم)
ls /sys/firmware/devicetree/overlays/ 2>/dev/null

# Tool متخصصة: fdtget لقراءة property معينة
fdtget /boot/my_board.dtb /soc/my_device compatible
fdtget /boot/my_board.dtb /soc/my_device reg
```

**تحقق إن الـ `status = "okay"` موجودة:**

```bash
fdtget /boot/my_board.dtb /soc/my_device status
# لازم يرجع: okay
# لو رجع: disabled → الـ device مش هيتعمل probe
```

---

### Practical Commands — جاهزة للـ Copy/Paste

#### فحص سريع لأي device مش بيعمل probe

```bash
#!/bin/bash
DEVICE="my_device_name"  # غير ده

echo "=== Deferred probes ==="
cat /sys/kernel/debug/devices_deferred | grep -A3 "$DEVICE"

echo "=== dmesg for device ==="
dmesg | grep -i "$DEVICE" | tail -30

echo "=== Device links (suppliers) ==="
ls /sys/bus/platform/devices/$DEVICE/supplier:* 2>/dev/null

echo "=== fwnode flags (via DT) ==="
cat /sys/firmware/devicetree/base/$(ls -la /sys/bus/platform/devices/$DEVICE/of_node | awk '{print $NF}' | sed 's|.*base/||')/status 2>/dev/null

echo "=== Properties ==="
ls /sys/firmware/devicetree/base/$(readlink /sys/bus/platform/devices/$DEVICE/of_node | sed 's|.*base/||')/ 2>/dev/null
```

#### تفعيل الـ fw_devlink verbose logging

```bash
# من cmdline (أضف للـ grub أو U-Boot):
fw_devlink=on

# أو اكتب في sysfs (runtime):
echo on > /sys/bus/platform/fw_devlink 2>/dev/null

# شوف الـ current mode
cat /proc/cmdline | grep fw_devlink
```

#### تتبع كل property reads لـ driver معين

```bash
# Enable function graph tracing
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function_graph > current_tracer
echo "fwnode_property_read*:mod:my_driver" > set_ftrace_filter 2>/dev/null || \
echo "fwnode_property_read*" > set_ftrace_filter
echo 1 > tracing_on

# حمّل الـ module
modprobe my_driver

echo 0 > tracing_on
cat trace | head -100
```

**مثال الـ output:**

```
 0) | fwnode_property_read_u32_array() {
 0) | fwnode_call_int_op() {
 0) | of_fwnode_property_read_int_array() {
 0)   2.140 us    |     of_find_property();
 0)   0.871 us    |     of_read_number();
 0)   4.820 us    |   }
 0)   5.903 us    |  }
 0)   7.102 us    | }
```

**التفسير:** وقت التنفيذ الكلي 7µs، الـ property اتقرأت من DT بنجاح عبر `of_fwnode_property_read_int_array`.

#### فحص الـ fwnode_link graph يدوياً

```bash
# الـ suppliers والـ consumers لكل device
for dev in /sys/bus/platform/devices/*/; do
    name=$(basename $dev)
    sup_count=$(ls $dev/supplier:* 2>/dev/null | wc -l)
    con_count=$(ls $dev/consumer:* 2>/dev/null | wc -l)
    if [ "$sup_count" -gt 0 ] || [ "$con_count" -gt 0 ]; then
        echo "$name: suppliers=$sup_count consumers=$con_count"
    fi
done
```

#### استخدام WARN_ON لكشف NULL fwnode في runtime

```bash
# أضف kernel param لـ panic on WARN (للـ CI/testing)
echo 1 > /proc/sys/kernel/panic_on_warn

# أو من cmdline:
# panic_on_warn=1
```

#### قراءة ACPI _DSD properties (مكافئة DT properties)

```bash
# _DSD = Device-Specific Data → بيحتوي على fwnode properties في ACPI
# باستخدام acpi_dbg tool
acpi_dbg -e "\_SB.MY_DEV._DSD"

# أو من sysfs (ACPI devices)
ls /sys/firmware/acpi/devices/
cat /sys/bus/acpi/devices/MYDEV0001\:00/description 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — الـ I2C sensor مش بيـprobe

#### العنوان
**الـ fwnode supplier link بيمنع الـ I2C temperature sensor من الـ probe على gateway صناعي**

#### السياق
فريق بيشتغل على industrial gateway بيستخدم **Texas Instruments AM62x** SoC. الـ gateway بيقرأ بيانات من sensor حرارة عبر **I2C** (TMP117)، ومتوصل بـ HDMI encoder خارجي. الـ board bring-up بدأ حديثاً، والـ DT كُتب من الصفر.

#### المشكلة
عند boot، الـ TMP117 driver بيديك:

```
[ 4.123456] i2c i2c-1: Failed to get supply 'vdd': -EPROBE_DEFER
[ 4.123789] tmp117 1-0048: probe deferred
```

الـ probe بيتأجل للأبد — حتى بعد ما الـ regulator driver اتحمل بنجاح. الـ sensor مش بيظهر في `/sys/bus/i2c/devices/`.

#### التحليل

الـ kernel بيستخدم `fwnode_link_add()` عشان يبني قائمة `suppliers` جوه `fwnode_handle`:

```c
/* fwnode.h */
struct fwnode_handle {
    struct fwnode_handle *secondary;
    const struct fwnode_operations *ops;
    struct device *dev;
    struct list_head suppliers;   /* كل fwnode_link بيبان هنا */
    struct list_head consumers;
    u8 flags;
};
```

الـ `fw_devlink` subsystem بيمشي على suppliers list دي قبل ما يسمح بـ probe. لو في `fwnode_link` موجود في القائمة ومش marked كـ `FWLINK_FLAG_CYCLE` ولا `FWLINK_FLAG_IGNORE`:

```c
/* fwnode.h */
struct fwnode_link {
    struct fwnode_handle *supplier;
    struct list_head s_hook;      /* hook في قائمة supplier */
    struct fwnode_handle *consumer;
    struct list_head c_hook;      /* hook في قائمة consumer */
    u8 flags;
};
```

المشكلة: في الـ DT، الـ sensor node فيه `pinctrl-0` بيشاور على `pinctrl` node اللي `status = "disabled"`. الـ `fw_devlink` بيشوفه supplier غير initialized (الـ `FWNODE_FLAG_INITIALIZED` مش set)، فبيدخل في defer loop لا نهاية ليها.

`fwnode_dev_initialized()` المفروض يتعمل من الـ pinctrl driver لما يخلص bind، بس لما الـ node disabled، مفيش driver بيـbind ومفيش حد بيعمل:

```c
/* fwnode.h */
static inline void fwnode_dev_initialized(struct fwnode_handle *fwnode,
                                          bool initialized)
{
    if (IS_ERR_OR_NULL(fwnode))
        return;
    if (initialized)
        fwnode->flags |= FWNODE_FLAG_INITIALIZED;
    else
        fwnode->flags &= ~FWNODE_FLAG_INITIALIZED;
}
```

فالـ supplier فضل `FWNODE_FLAG_INITIALIZED = 0` وفالـ consumer (TMP117) عالق في defer.

#### الحل

**خيار 1 — DT fix:** شيل الـ `pinctrl-0` من الـ sensor node لو هو مش محتاجه فعلاً:

```dts
/* قبل */
&i2c1 {
    tmp117@48 {
        compatible = "ti,tmp117";
        reg = <0x48>;
        pinctrl-names = "default";
        pinctrl-0 = <&i2c1_pins>;  /* هنا المشكلة */
        vdd-supply = <&vdd_3v3>;
    };
};

/* بعد */
&i2c1 {
    tmp117@48 {
        compatible = "ti,tmp117";
        reg = <0x48>;
        /* شلنا pinctrl من الـ sensor نفسه، الـ I2C controller هو اللي بيـhandle الـ pinmux */
        vdd-supply = <&vdd_3v3>;
    };
};
```

**خيار 2 — fw_devlink=off للتشخيص:**

```bash
# أضف للـ kernel cmdline مؤقتاً
fw_devlink=off
```

**تحقق من السبب:**

```bash
# شوف suppliers list للـ fwnode
cat /sys/kernel/debug/devices_deferred
# أو
grep -r "tmp117" /sys/kernel/debug/supply_map 2>/dev/null
```

#### الدرس المستفاد
**الـ `suppliers` list في `fwnode_handle` بتتبني تلقائياً من أي property بيشاور على node تاني.** حتى `pinctrl-0` على node داخل I2C بيخلق `fwnode_link`. لو الـ supplier disabled ومفيش driver بيـcall `fwnode_dev_initialized()`، الـ consumer هيفضل في defer loop. دايماً اتحقق من `/sys/kernel/debug/devices_deferred` وشوف مين الـ supplier المُعلَّق.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI مش بيشتغل بسبب graph endpoint

#### العنوان
**الـ `graph_get_next_endpoint` بيرجع NULL وشاشة الـ HDMI فارغة على TV box**

#### السياق
منتج Android TV box بيستخدم **Allwinner H616**. الـ display stack: DE2 (Display Engine) → HDMI encoder → connector. المطور ورث DT من board قديم (H6) وعدّل عليه. بعد أول boot، HDMI مش بيديك صورة خالص.

#### المشكلة

```
[ 3.456] sun8i-de2 1000000.display-engine: failed to bind all components
[ 3.457] sun8i-hdmi 6000000.hdmi: endpoint 0 not found
```

#### التحليل

الـ display driver بيستخدم `fwnode_operations.graph_get_next_endpoint` عشان يمشي على الـ graph. الـ `fwnode_operations` struct في `fwnode.h`:

```c
struct fwnode_operations {
    /* ... */
    struct fwnode_handle *
    (*graph_get_next_endpoint)(const struct fwnode_handle *fwnode,
                               struct fwnode_handle *prev);

    struct fwnode_handle *
    (*graph_get_remote_endpoint)(const struct fwnode_handle *fwnode);

    struct fwnode_handle *
    (*graph_get_port_parent)(struct fwnode_handle *fwnode);

    int (*graph_parse_endpoint)(const struct fwnode_handle *fwnode,
                                struct fwnode_endpoint *endpoint);
};
```

الـ macro اللي بيتنادى:

```c
#define fwnode_call_ptr_op(fwnode, op, ...)     \
    (fwnode_has_op(fwnode, op) ?                \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : NULL)
```

لو `graph_get_next_endpoint` بيرجع NULL، يعني إما:
- الـ `fwnode_has_op` بيرجع false (الـ ops مش set)
- أو الـ OF graph parser مش لاقي الـ port/endpoint nodes

الـ `fwnode_endpoint` struct:

```c
struct fwnode_endpoint {
    unsigned int port;   /* رقم الـ port */
    unsigned int id;     /* رقم الـ endpoint */
    const struct fwnode_handle *local_fwnode;
};
```

المشكلة في الـ DT: المطور نسي الـ naming convention المطلوب:

```dts
/* غلط — اسم عشوائي */
de2_out: port {
    de2_out_ep: endpoint {
        remote-endpoint = <&hdmi_in_ep>;
    };
};

/* صح — لازم يتبع SWNODE_GRAPH_PORT_NAME_FMT */
ports {
    port@0 {                    /* "port@%u" */
        #address-cells = <1>;
        #size-cells = <0>;
        de2_out_ep: endpoint@0 { /* "endpoint@%u" */
            reg = <0>;
            remote-endpoint = <&hdmi_in_ep>;
        };
    };
};
```

الـ `fwnode.h` بيعرّف الـ naming:

```c
#define SWNODE_GRAPH_PORT_NAME_FMT      "port@%u"
#define SWNODE_GRAPH_ENDPOINT_NAME_FMT  "endpoint@%u"
```

لما الـ OF backend بيدور على endpoints، بيدور على nodes باسم `port@N`. لو الاسم `port` بدون `@N`، الـ `graph_get_next_endpoint` مش بيلاقيه.

#### الحل

صحح الـ DT:

```dts
&de2 {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;

        port@0 {
            reg = <0>;
            #address-cells = <1>;
            #size-cells = <0>;
            de2_out: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&hdmi_in>;
            };
        };
    };
};

&hdmi {
    ports {
        #address-cells = <1>;
        #size-cells = <0>;

        port@0 {
            reg = <0>;
            #address-cells = <1>;
            #size-cells = <0>;
            hdmi_in: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&de2_out>;
            };
        };
    };
};
```

**تشخيص:**

```bash
# تحقق إن الـ graph links اتبنت صح
cat /sys/kernel/debug/of_graph
# أو
dtc -I fs /proc/device-tree | grep -A5 "port@"
```

#### الدرس المستفاد
**الـ `SWNODE_GRAPH_PORT_NAME_FMT` و `SWNODE_GRAPH_ENDPOINT_NAME_FMT` مش مجرد اقتراح — دول convention إجباري.** الـ `graph_get_next_endpoint` في `fwnode_operations` بيمشي على الـ node names بالـ pattern ده. اسم غلط = الـ graph مش بيتبني = display stack كامل بيوقع.

---

### السيناريو الثالث: IoT Sensor Node على STM32MP1 — الـ SPI DMA مش شغال

#### العنوان
**الـ `device_dma_supported` بيرجع false على STM32MP1 وبيعطل الـ SPI DMA transfers**

#### السياق
IoT sensor node بيستخدم **STM32MP1** كـ host processor. الـ sensor بيتكلم عبر **SPI** بـ data rate عالي (20 MHz). المطور عايز يستخدم DMA عشان يخفف الـ CPU load. لكن الـ SPI driver مش بيعمل DMA transfers وبيرجع للـ PIO mode.

#### المشكلة

```
[ 2.891] spi-stm32 44004000.spi: DMA not available, using PIO mode
[ 2.892] spi-stm32 44004000.spi: registered master spi0
```

الـ CPU utilization بيوصل لـ 80% عند full throughput بدل ما يبقى 5%.

#### التحليل

الـ SPI driver بيتحقق من DMA support عبر:

```c
/* fwnode_operations */
bool (*device_dma_supported)(const struct fwnode_handle *fwnode);
enum dev_dma_attr (*device_get_dma_attr)(const struct fwnode_handle *fwnode);
```

والـ enum:

```c
enum dev_dma_attr {
    DEV_DMA_NOT_SUPPORTED,
    DEV_DMA_NON_COHERENT,
    DEV_DMA_COHERENT,
};
```

الـ macro اللي بيـcall الـ op:

```c
#define fwnode_call_bool_op(fwnode, op, ...)    \
    (fwnode_has_op(fwnode, op) ?                \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : false)
```

لو `device_dma_supported` مش موجود في الـ ops، بيرجع `false` تلقائياً. الـ OF backend بيـimplements الـ op دي، لكن بتقرا من الـ DT property `dma-coherent` أو من معلومات الـ IOMMU.

المشكلة: المطور نسى يحط `dma-ranges` في الـ DT لـ SoC دي:

```dts
/* غلط — مفيش dma-ranges */
soc {
    #address-cells = <1>;
    #size-cells = <1>;
    /* مفيش dma-ranges! */

    spi@44004000 {
        compatible = "st,stm32h7-spi";
        reg = <0x44004000 0x400>;
        dmas = <&dmamux1 37 0x400 0x05>,
               <&dmamux1 38 0x400 0x05>;
        dma-names = "rx", "tx";
    };
};
```

بدون `dma-ranges`، الـ OF fwnode backend بيعتبر الـ device مش accessible من DMA، فالـ `device_dma_supported` بيرجع false.

#### الحل

```dts
soc {
    #address-cells = <1>;
    #size-cells = <1>;
    /* أضف dma-ranges عشان يعرف الـ kernel إن الـ DMA accessible */
    dma-ranges = <0x00000000 0x00000000 0xffffffff>;

    spi@44004000 {
        compatible = "st,stm32h7-spi";
        reg = <0x44004000 0x400>;
        dmas = <&dmamux1 37 0x400 0x05>,
               <&dmamux1 38 0x400 0x05>;
        dma-names = "rx", "tx";
        /* لو الـ SoC non-coherent DMA */
        /* مفيش dma-coherent property هنا = NON_COHERENT */
    };
};
```

**تحقق:**

```bash
# شوف الـ DMA attr للـ device
cat /sys/bus/platform/devices/44004000.spi/dma_coherent_mask
# أو من kernel log مع dynamic debug
echo "module spi_stm32 +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep -i dma
```

**كود تشخيص في driver:**

```c
/* الـ driver بيقرأ الـ DMA attr عبر fwnode */
if (!fwnode_call_bool_op(fwnode, device_dma_supported)) {
    dev_warn(dev, "DMA not available, using PIO mode\n");
    /* فالـ driver بيـfallback لـ PIO */
}
```

#### الدرس المستفاد
**الـ `device_dma_supported` في `fwnode_operations` بيتحكم في أساسي جداً — هل الـ device يقدر يستخدم DMA.** الـ `fwnode_call_bool_op` بيرجع `false` لو الـ op مش موجود أو لو الـ DT مش فيه `dma-ranges` صح. على STM32MP1 و SoCs زيها، `dma-ranges` في الـ soc node حاجة أساسية مش اختيارية.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — Probe Defer Cycle بيعمل Deadlock

#### العنوان
**الـ `FWLINK_FLAG_CYCLE` وضع غلط بيخلي ECU يـboot بدون CAN interface**

#### السياق
**Automotive ECU** بيستخدم **NXP i.MX8** للـ ADAS processing. فيه CAN controller (FlexCAN) بيعتمد على clock provider، وأحياناً الـ clock provider نفسه بيرجع للـ CAN عبر power domain. النتيجة: circular dependency في الـ `fwnode_link` graph.

#### المشكلة

```
[ 5.234] fsl-flexcan 30c00000.can: probe deferred due to supplier 30390000.clock-controller
[ 5.235] fsl-flexcan 30c00000.can: ... which is waiting on 30c00000.can (cycle detected)
[ 5.240] WARNING: circular dependency in fwnode link graph
```

الـ CAN interface مش بيظهر خالص حتى بعد full boot. الـ vehicle network معطل.

#### التحليل

الـ kernel لما يشوف cycle في الـ `fwnode_link` graph، بيـmark الـ links كـ `FWLINK_FLAG_CYCLE`:

```c
/* fwnode.h */
/*
 * CYCLE: The fwnode link is part of a cycle. Don't defer probe.
 * IGNORE: Completely ignore this link, even during cycle detection.
 */
#define FWLINK_FLAG_CYCLE   BIT(0)
#define FWLINK_FLAG_IGNORE  BIT(1)

struct fwnode_link {
    struct fwnode_handle *supplier;
    struct list_head s_hook;
    struct fwnode_handle *consumer;
    struct list_head c_hook;
    u8 flags;           /* هنا بيتحط FWLINK_FLAG_CYCLE */
};
```

لما `flags` = `FWLINK_FLAG_CYCLE`، المفروض الـ probe ordering يتجاهل الـ link ده ويسمح لأي طرف يـprobe أول. بس في الحالة دي، الـ DT عنده مشكلة: الـ power domain node فيه property بتشاور على الـ CAN node بدون سبب حقيقي — بقايا من copy-paste من board تاني.

الـ `fw_devlink` subsystem بيبني الـ `fwnode_link` عبر `fwnode_link_add()`:

```c
/* fwnode.h */
int fwnode_link_add(struct fwnode_handle *con,
                    struct fwnode_handle *sup,
                    u8 flags);
```

وبيـpurge الـ links اللي مش محتاجها بـ:

```c
void fwnode_links_purge(struct fwnode_handle *fwnode);
void fw_devlink_purge_absent_suppliers(struct fwnode_handle *fwnode);
```

الـ cycle detection algorithm بيمشي على الـ `suppliers` و `consumers` lists في كل `fwnode_handle` وبيلاقي الـ cycle. بس في الحالة دي، الـ cycle حقيقي من الـ DT بسبب property زيادة.

**تحديد الـ cycle:**

```bash
# شوف الـ fwnode graph
cat /sys/kernel/debug/devices_deferred
# شوف الـ supplier links
ls /sys/kernel/debug/device_link/
```

#### الحل

**شيل الـ property الزيادة من الـ DT:**

```dts
/* قبل — الـ power domain بيشاور على CAN بالغلط */
power-domains-ctrl@30390000 {
    compatible = "fsl,imx8mq-gpc";
    reg = <0x30390000 0x10000>;
    /* دي property غلط — بتخلق link من PDC للـ CAN */
    fsl,imx-src = <&can1>;   /* شيلها */
};

/* بعد — شلنا الـ property الغلط */
power-domains-ctrl@30390000 {
    compatible = "fsl,imx8mq-gpc";
    reg = <0x30390000 0x10000>;
    /* مفيش reference للـ CAN هنا */
};
```

**لو الـ cycle حقيقي وصعب تشيله، استخدم `FWLINK_FLAG_IGNORE` من الـ driver:**

```c
/* في الـ driver اللي بيخلق الـ link يدوياً */
fwnode_link_add(consumer_fwnode, supplier_fwnode,
                FWLINK_FLAG_IGNORE);  /* تجاهل الـ link ده في cycle detection */
```

**تأكيد بعد الحل:**

```bash
# تأكد إن الـ CAN probe نجح
ip link show can0
# أو
dmesg | grep flexcan
```

#### الدرس المستفاد
**الـ `fwnode_link` graph بيتبني من أي property في الـ DT بتشاور على node تانية.** حتى property زيادة أو غلط بتعمل cycle. الـ `FWLINK_FLAG_CYCLE` بيتحط أوتوماتيكي بس مش دايماً بيحل المشكلة. الحل الصح: شيل الـ dependency الغلط من الـ DT. في automotive systems، الـ boot time حرج — cycle في الـ fwnode graph يعني interface critical زي CAN مش بيـprobe.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — Software Node مش بيشتغل مع I2C Camera

#### العنوان
**استخدام `software_node` مع camera ISP على RK3562 بيفشل لأن الـ `fwnode_init` مش اتعمل صح**

#### السياق
فريق bring-up بيشتغل على custom board بيستخدم **Rockchip RK3562**. الـ board فيه camera module MIPI-CSI متوصل على ISP. الـ camera driver بيستخدم **software_node** (مش DT) عشان الـ board مش ليها DT بعد — الـ driver بيـregister الـ node يدوياً من الـ driver code. الـ ISP مش بيـdetect الـ camera.

#### المشكلة

```
[ 6.123] rkisp1 fdfe0000.isp: failed to find remote endpoint
[ 6.124] rkisp1 fdfe0000.isp: graph endpoint not found: -ENOENT
[ 6.125] rkisp1 fdfe0000.isp: probe failed: -ENOENT
```

#### التحليل

الـ software_node system بيستخدم `struct fwnode_handle` مع custom `fwnode_operations`. عشان الـ fwnode يشتغل صح، لازم يتعمله `fwnode_init()`:

```c
/* fwnode.h */
static inline void fwnode_init(struct fwnode_handle *fwnode,
                               const struct fwnode_operations *ops)
{
    fwnode->ops = ops;
    INIT_LIST_HEAD(&fwnode->consumers);  /* لازم تتعمل */
    INIT_LIST_HEAD(&fwnode->suppliers);  /* لازم تتعمل */
}
```

المطور عمل كده:

```c
/* كود المطور الغلط */
static struct fwnode_handle cam_fwnode = {
    .ops = &software_node_ops,
    /* نسي يعمل INIT_LIST_HEAD للـ consumers و suppliers */
};

/* نتيجة: الـ consumers.next و suppliers.next = NULL */
/* لما الـ fw_devlink يحاول يمشي على القوائم دول → kernel panic أو سلوك غريب */
```

الـ `fwnode_has_op` بيتحقق:

```c
#define fwnode_has_op(fwnode, op) \
    (!IS_ERR_OR_NULL(fwnode) && (fwnode)->ops && (fwnode)->ops->op)
```

الـ ops موجودة، فالـ macro بيرجع true. لكن لما الـ `add_links` callback بيتعمل:

```c
int (*add_links)(struct fwnode_handle *fwnode);
```

بيحاول يمشي على الـ suppliers list اللي هي uninitialized (مش اتعملها `INIT_LIST_HEAD`). الـ linked list code بيـaccess `suppliers.next` وبيلاقي garbage pointer.

في نفس الوقت، الـ graph endpoint للـ camera محتاج الـ `port@0/endpoint@0` structure، والـ naming convention:

```c
/* لازم تتبع الـ macros دي */
#define SWNODE_GRAPH_PORT_NAME_FMT      "port@%u"
#define SWNODE_GRAPH_ENDPOINT_NAME_FMT  "endpoint@%u"
```

المطور استخدم اسم `"port"` بدل `"port@0"` في الـ software_node properties.

#### الحل

**أولاً — صحح الـ fwnode_init:**

```c
/* camera driver — صح */
static struct fwnode_handle cam_fwnode;

static int camera_probe(struct i2c_client *client)
{
    /* لازم تستخدم fwnode_init قبل أي حاجة تانية */
    fwnode_init(&cam_fwnode, &software_node_ops);

    /* دلوقتي الـ consumers و suppliers lists initialized */
    /* ... باقي الـ probe code */
}
```

**ثانياً — صحح الـ graph node names:**

```c
/* غلط */
static const struct software_node cam_port_node = {
    .name = "port",          /* غلط */
    .parent = &cam_node,
};

static const struct software_node cam_ep_node = {
    .name = "endpoint",      /* غلط */
    .parent = &cam_port_node,
};

/* صح */
static const struct software_node cam_port_node = {
    .name = "port@0",        /* يتبع SWNODE_GRAPH_PORT_NAME_FMT */
    .parent = &cam_node,
};

static const struct software_node cam_ep_node = {
    .name = "endpoint@0",    /* يتبع SWNODE_GRAPH_ENDPOINT_NAME_FMT */
    .parent = &cam_port_node,
};
```

**ثالثاً — verify:**

```bash
# شوف الـ software nodes المسجلة
ls /sys/bus/platform/drivers/rkisp1/
# أو
cat /sys/kernel/debug/software_nodes
# تأكد إن الـ camera فيه fwnode مع graph
v4l2-ctl --list-devices
```

**تحقق من الـ link بين الـ ISP و camera:**

```bash
media-ctl -p -d /dev/media0
# المفروض يظهر link بين camera sensor و ISP
```

#### الدرس المستفاد
**`fwnode_init()` مش اختياري — هي اللي بتعمل `INIT_LIST_HEAD` للـ `consumers` و `suppliers` lists.** بدونها، أي كود بيمشي على القوائم دول هيوقع في undefined behavior أو kernel panic. في board bring-up بدون DT، الـ software_node system قوي جداً بس محتاج `fwnode_init()` صح وناميق convention صح (`port@N`, `endpoint@N`) عشان الـ graph API تشتغل.

## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات على LWN.net اللي بتغطي الـ fwnode framework وكل اللي يتعلق بيه:

| المقال | الوصف |
|--------|-------|
| [device property: Introducing software nodes](https://lwn.net/Articles/770825/) | بيشرح إزاي الـ `software_node` (swnode) اتضافت كنوع مستقل من الـ fwnode، بديل عن الـ `property_set` |
| [Software fwnode references](https://lwn.net/Articles/789099/) | بيغطي إضافة دعم الـ reference arguments للـ software nodes، وازاي تشاور على node تاني من جوا الـ swnode |
| [ACPI graph support](https://lwn.net/Articles/718184/) | بيشرح دعم الـ fwnode graph endpoints في ACPI وازاي اتربطت بنفس الـ ops اللي بتستخدمها الـ OF |
| [Unified fwnode endpoint parser](https://lwn.net/Articles/737780/) | بيغطي توحيد الـ endpoint parser في الـ v4l2_fwnode وربطه بالـ fwnode_operations |
| [introduce fwnode in the I2C subsystem](https://lwn.net/Articles/889236/) | مثال عملي على إزاي subsystem كامل (I2C) بدأ يستخدم الـ fwnode بدل ما يتعامل مع الـ OF و ACPI بشكل منفصل |
| [gpiolib: Two new helpers and way toward fwnode](https://lwn.net/Articles/889987/) | بيوضح إزاي الـ gpiolib بدأت تتحول من الـ `of_node` للـ fwnode API |
| [Add support for software nodes to gpiolib](https://lwn.net/Articles/914223/) | تكملة للموضوع: إضافة swnode support كاملة للـ gpiolib |
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | خلفية مهمة عن العلاقة بين الـ platform devices والـ device tree، اللي الـ fwnode جه يوحّدها مع ACPI |

---

### التوثيق الرسمي في الـ kernel

#### ملفات `Documentation/` المباشرة

```
Documentation/driver-api/driver-model/
Documentation/firmware-guide/acpi/dsd/graph.rst
Documentation/firmware-guide/acpi/dsd/properties-rules.rst
Documentation/firmware-guide/acpi/enumeration.rst
Documentation/devicetree/usage-model.rst
```

**الـ fwnode graph documentation (ACPI DSD):**
- [Graphs — The Linux Kernel documentation](https://www.kernel.org/doc/html/v5.2/firmware-guide/acpi/dsd/graph.html) — بيشرح إزاي الـ graph endpoints بتشتغل مع ACPI _DSD وإزاي الـ fwnode بيوفر abstraction موحدة
- [ACPI Based Device Enumeration](https://docs.kernel.org/firmware-guide/acpi/enumeration.html) — بيغطي إزاي الـ ACPI devices بتتـenumerate وبتتعرف على الـ fwnode بتاعها

**الـ V4L2 fwnode kAPI:**
- [V4L2 fwnode kAPI](https://docs.kernel.org/driver-api/media/v4l2-fwnode.html) — مثال حي على استخدام الـ fwnode_operations في subsystem حقيقي

**الـ driver infrastructure:**
- [Device drivers infrastructure](https://docs.kernel.org/driver-api/infrastructure.html) — بيشمل الـ `fwnode_handle`, `device_property_*`, `fwnode_property_*` APIs

---

### Kernel Source Files المهمة

الملفات دي هي اللي بتـimplement الـ fwnode framework فعلاً:

```
include/linux/fwnode.h          ← الـ header الأساسي (الملف اللي بندرسه)
include/linux/property.h        ← الـ public API للـ device properties
drivers/base/property.c         ← implementation الـ device_property_* و fwnode_property_*
drivers/base/swnode.c           ← software nodes implementation
drivers/base/core.c             ← ربط الـ fwnode بالـ struct device
```

---

### Kernel Commits المهمة

#### الـ commit الأصلي — إدخال الـ fwnode

الـ fwnode framework اتقدّم أول ما Rafael J. Wysocki كتبه في 2015 كجزء من توحيد الـ device property API:

- **[PATCH] driver core: Implement device property accessors through fwnode ones**
  - [linux.kernel.narkive.com](https://linux.kernel.narkive.com/pNOj9vdt/patch-driver-core-implement-device-property-accessors-through-fwnode-ones)
  - ده الـ patch اللي حوّل الـ `device_property_*()` functions عشان تستخدم الـ `fwnode_property_*()` جواها

- **[PATCH 04/20] device property: Add operations struct for fwnode operations**
  - [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-acpi/patch/1487869276-25244-5-git-send-email-sakari.ailus@linux.intel.com/)
  - ده الـ patch اللي أدخل الـ `struct fwnode_operations` كـ vtable موحدة

- **[PATCH] device property: Fallback to secondary fwnode if primary misses the property**
  - [linux.kernel.narkive.com](https://linux.kernel.narkive.com/XO6y722J/patch-v1-08-13-device-property-fallback-to-secondary-fwnode-if-primary-misses-the-property)
  - بيوضح فلسفة الـ primary/secondary fwnode chain

#### الـ fw_devlink improvements

- **[PATCH v1 0/9] fw_devlink improvements** — Saravana Kannan (Google)
  - [lore.kernel.org](https://lore.kernel.org/linux-devicetree/CAGETcx_nVXbHzZ3+_aR4SZtSnSBU=Rfp8Qm2jOs7zGZRaH_88A@mail.gmail.com/T/)
  - بيغطي تحسينات الـ `fw_devlink` وعلاقتها بالـ `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` و `FWNODE_FLAG_BEST_EFFORT`

- **[PATCH v4 3/8] driver core: Add fw_devlink.strict kernel param**
  - [mail-archive.com](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2468715.html)

---

### Mailing List Discussions

مناقشات مهمة على الـ LKML وقوائم البريد ذات الصلة:

| الموضوع | الرابط |
|---------|-------|
| `set_secondary_fwnode()` export discussion | [lists.openwall.net](https://lists.openwall.net/linux-kernel/2020/04/08/743) |
| Fix secondary fwnode handling in `set_primary_fwnode()` | [lore.kernel.org/patchwork](https://lore.kernel.org/patchwork/patch/1298392) |
| Do not overwrite secondary fwnode with NULL | [mail-archive.com](https://www.mail-archive.com/linux-i2c@vger.kernel.org/msg22196.html) |
| `fw_devlink` debug command line parameter | [lore.kernel.org/all](https://lore.kernel.org/all/20210914043928.4066136-6-saravanak@google.com/) |
| `fwnode_device_is_available()` introduction | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-acpi/patch/1496741861-8240-5-git-send-email-sakari.ailus@linux.intel.com/) |

**الـ LKML archive:** [lkml.org](https://lkml.org/) — ابحث بـ `fwnode` أو `fwnode_handle` أو `fw_devlink`

---

### كتب مُوصى بيها

#### Linux Device Drivers (LDD3) — Corbet, Rubini, Kroah-Hartman
الكتاب ده قديم شوية ومش بيغطي الـ fwnode API مباشرة، لكن:
- **Chapter 14: The Linux Device Model** — أساس لفهم الـ `struct device`, `kobject`, وازاي الـ device model بيشتغل
- الـ fwnode هو extension طبيعي للـ device model اللي LDD3 بيشرحه
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17: Devices and Modules** — بيغطي الـ device model والـ sysfs
- بيبني فهم الأساس اللي الـ fwnode بيتبنى عليه
- الـ fwnode نفسه أحدث من الكتاب ده، بس الخلفية ضرورية

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 15: Bootstrapping Linux** — بيغطي إزاي الـ firmware (DT, ACPI) بتـpass الـ platform info للـ kernel
- مهم كخلفية لفهم ليه الـ fwnode abstraction ضرورية في البيئات الـ embedded

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- بيغطي الـ driver model بعمق أكبر من LDD3
- مفيد لفهم الـ device lifecycle اللي الـ fwnode بيتعامل معاه

---

### KernelNewbies

الـ KernelNewbies بتتبع التغييرات في كل kernel release. ابحث فيها عن التغييرات المتعلقة بالـ fwnode في:

- [Linux_6.14 — KernelNewbies](https://kernelnewbies.org/Linux_6.14) — بيذكر `v4l: fwnode: Add support for CSI-2 C-PHY line orders` وتغييرات تانية في الـ fwnode
- [Linux_6.15 — KernelNewbies](https://kernelnewbies.org/Linux_6.15)
- [Linux_6.13 — KernelNewbies](https://kernelnewbies.org/Linux_6.13)
- [LinuxChanges — KernelNewbies](https://kernelnewbies.org/LinuxChanges) — للـ releases الأقدم
- [Documents — KernelNewbies](https://kernelnewbies.org/Documents) — فيه شرح لكتير من الـ kernel concepts

---

### eLinux.org

الـ eLinux.org مش عندها صفحة مخصصة للـ fwnode، لكن الموارد دي مفيدة كخلفية:

- [eLinux.org Device Tree Reference](https://elinux.org/Device_Tree_Reference) — الـ DT هو أحد الـ fwnode backends
- [eLinux.org Device Tree Usage](https://elinux.org/Device_Tree_Usage) — إزاي الـ DT properties بتتقرأ، نفس الـ pattern اللي الـ fwnode بيوحّده

---

### Search Terms للبحث عن مزيد من المعلومات

لو حابب تعمق أكتر، استخدم الـ search terms دي:

```
# للبحث في الـ LKML
fwnode_handle linux kernel
fwnode_operations vtable
fw_devlink probe ordering
software_node swnode kernel
fwnode_graph_get_next_endpoint

# للبحث في git log
git log --oneline -- include/linux/fwnode.h
git log --oneline -- drivers/base/property.c
git log --oneline -- drivers/base/swnode.c

# للبحث في kernel source
git grep "fwnode_operations" -- include/
git grep "fwnode_has_op" -- drivers/

# في LWN.net
site:lwn.net fwnode
site:lwn.net "device property" kernel
site:lwn.net "software node" kernel

# في patchwork
site:patchwork.kernel.org fwnode
```

---

### ملخص سريع للمصادر حسب الأولوية

| الأولوية | المصدر | ليه؟ |
|----------|--------|-------|
| 1 | `include/linux/fwnode.h` + `include/linux/property.h` | الـ source نفسه هو أدق توثيق |
| 2 | [LWN: Introducing software nodes](https://lwn.net/Articles/770825/) | بيشرح الـ design decisions |
| 3 | [LWN: Software fwnode references](https://lwn.net/Articles/789099/) | بيكمل الصورة |
| 4 | [Kernel docs: ACPI DSD Graphs](https://www.kernel.org/doc/html/v5.2/firmware-guide/acpi/dsd/graph.html) | مثال عملي على الـ fwnode graph API |
| 5 | [LDD3 Chapter 14](https://lwn.net/Kernel/LDD3/) | الأساس النظري للـ device model |
| 6 | [fw_devlink improvements](https://lore.kernel.org/linux-devicetree/CAGETcx_nVXbHzZ3+_aR4SZtSnSBU=Rfp8Qm2jOs7zGZRaH_88A@mail.gmail.com/T/) | لفهم الـ FWNODE_FLAG_* flags |
## Phase 8: Writing simple module

### الهدف

**الـ** `fwnode_link_add()` هي function مُصدَّرة من firmware node subsystem بتُضاف link بين consumer و supplier node. هنستخدم **kprobe** عشان نعمل hook عليها ونطبع اسم الـ fwnode اللي بيتعمله link لحظة ما بيتنادى عليها.

ده بيخلينا نشوف إزاي الـ kernel بيبني dependency graph بين الـ firmware nodes وقت bootup أو device registration.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * fwnode_link_add kprobe tracer
 *
 * Hooks fwnode_link_add() to log every firmware node link
 * being created — shows consumer/supplier relationships.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/fwnode.h>       /* fwnode_handle, fwnode_call_ptr_op */
#include <linux/device.h>       /* struct device */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("Trace fwnode_link_add() calls via kprobe");

/* ------------------------------------------------------------------ */
/*  helper: safely get the name of an fwnode (may be NULL)             */
/* ------------------------------------------------------------------ */
static const char *safe_fwnode_name(const struct fwnode_handle *fwnode)
{
    if (!fwnode || !fwnode->ops || !fwnode->ops->get_name)
        return "<unknown>";
    return fwnode->ops->get_name(fwnode);
}

/* ------------------------------------------------------------------ */
/*  pre-handler: runs BEFORE fwnode_link_add executes                  */
/*  pt_regs holds the CPU registers at the probe site                  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Calling convention on x86-64:
     *   arg0 (rdi) = con  — consumer fwnode_handle *
     *   arg1 (rsi) = sup  — supplier fwnode_handle *
     *   arg2 (rdx) = flags (u8)
     *
     * On ARM64 use regs->regs[0], regs->regs[1], regs->regs[2].
     */
#ifdef CONFIG_X86_64
    struct fwnode_handle *con = (struct fwnode_handle *)regs->di;
    struct fwnode_handle *sup = (struct fwnode_handle *)regs->si;
    u8 flags                  = (u8)regs->dx;
#elif defined(CONFIG_ARM64)
    struct fwnode_handle *con = (struct fwnode_handle *)regs->regs[0];
    struct fwnode_handle *sup = (struct fwnode_handle *)regs->regs[1];
    u8 flags                  = (u8)regs->regs[2];
#else
    /* Fallback: skip on unsupported arch */
    return 0;
#endif

    pr_info("fwnode_link_add: consumer=[%s] supplier=[%s] flags=0x%02x\n",
            safe_fwnode_name(con),
            safe_fwnode_name(sup),
            flags);

    return 0; /* 0 = let the probed function continue normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor — points to the symbol we want to hook           */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "fwnode_link_add",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module_init: register the kprobe                                   */
/* ------------------------------------------------------------------ */
static int __init fwnode_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("fwnode_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("fwnode_probe: planted kprobe on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit: MUST unregister — otherwise the probe stays active    */
/*  after the module text is gone → guaranteed kernel panic            */
/* ------------------------------------------------------------------ */
static void __exit fwnode_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("fwnode_probe: removed kprobe from %s\n", kp.symbol_name);
}

module_init(fwnode_probe_init);
module_exit(fwnode_probe_exit);
```

---

### Makefile

```makefile
obj-m += fwnode_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/fwnode.h>
```

**الـ** `kprobes.h` بيجيب `struct kprobe` و `register_kprobe()` / `unregister_kprobe()`.
**الـ** `fwnode.h` بيجيب `struct fwnode_handle` و `fwnode_operations` اللي محتاجينهم عشان نقرأ اسم الـ node.

---

#### الـ `safe_fwnode_name()`

```c
static const char *safe_fwnode_name(const struct fwnode_handle *fwnode)
```

الـ fwnode مش لازم يكون عنده `ops->get_name` — بعض الـ nodes بدائية أو بتتعمل في مرحلة مبكرة. الـ NULL check ده بيحمي من kernel panic جوّا الـ probe handler.

---

#### الـ `handler_pre()`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ kprobe بيوقف التنفيذ قبل ما `fwnode_link_add` تشتغل وبيديك الـ `pt_regs` — يعني registers الـ CPU اللي فيها الـ arguments. إحنا بنقرأ `rdi` و `rsi` على x86-64 (أو `regs[0]` و `regs[1]` على ARM64) عشان نعرف مين consumer ومين supplier.

---

#### الـ `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "fwnode_link_add",
    .pre_handler = handler_pre,
};
```

**الـ** `.symbol_name` بيخلي الـ kernel يحل الـ address تلقائياً من الـ kallsyms — مش محتاج تكتب address يدوي.
**الـ** `.pre_handler` هي الـ callback اللي بتتنادى عليها قبل ما الـ function الأصلية تشتغل.

---

#### الـ `module_init`

```c
ret = register_kprobe(&kp);
```

`register_kprobe` بتزرع breakpoint عند بداية `fwnode_link_add`. لو رجعت قيمة سالبة يبقى إما الـ symbol مش موجود في الـ kernel (مش compiled) أو الـ CONFIG_KPROBES مش enabled.

---

#### الـ `module_exit`

```c
unregister_kprobe(&kp);
```

لازم تعمل unregister في الـ exit وإلا لما الـ module يتشال من الـ memory، الـ breakpoint هيظل بيشاور على كود اتمسح → **kernel panic** فوري. ده مش اختياري.

---

### تشغيل وقراءة الـ output

```bash
# بناء الـ module
make

# تحميله
sudo insmod fwnode_probe.ko

# مشاهدة الـ log (بيظهر لما device جديد بيتـ probe)
sudo dmesg -w | grep fwnode_link_add

# تفريغه
sudo rmmod fwnode_probe
```

مثال على الـ output لو جهاز USB اتوصل:

```
fwnode_probe: planted kprobe on fwnode_link_add @ ffffffff81a3c220
fwnode_probe: fwnode_link_add: consumer=[/i2c@0/sensor@48] supplier=[/clocks/osc24M] flags=0x00
fwnode_probe: fwnode_link_add: consumer=[/i2c@0/sensor@48] supplier=[/regulators/vdd] flags=0x00
```

ده بيوضح إن الـ sensor محتاج الـ clock والـ regulator قبل ما يتـ probe — ده بالظبط الـ dependency graph اللي الـ `fw_devlink` بيعمله.

---

### ملاحظات مهمة

| نقطة | تفصيل |
|------|--------|
| `CONFIG_KPROBES=y` | لازم يكون enabled في الـ kernel config |
| `CONFIG_KALLSYMS=y` | عشان الـ `.symbol_name` يشتغل |
| الـ handler بيشتغل في interrupt context | ممنوع تعمل sleep أو تاخد non-spinlock mutex |
| الـ `regs` layout معتمد على architecture | الكود عنده `#ifdef` للـ x86-64 والـ ARM64 |
