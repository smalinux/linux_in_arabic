## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `fwnode.h`** ينتمي لـ subsystem اسمه **SOFTWARE NODES AND DEVICE PROPERTIES** — وهو جزء من **Driver Core** في Linux kernel. بتلاقيه مذكور في MAINTAINERS تحت عنوانين: **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS** و **SOFTWARE NODES AND DEVICE PROPERTIES**.

---

### القصة من الأول

تخيل إنك بتشتري laptop جديد. جوّاه عشرات الـ hardware components: Bluetooth chip, Wi-Fi card, camera, sensors. كل component ده محتاج driver. الـ driver محتاج يعرف معلومات عن الـ hardware: إيه الـ interrupts بتاعته؟ بيتكلم على أي bus? هل بيدعم DMA? إيه الـ GPIO pins المربوطة بيه؟

الـ Linux بيشتغل على ملايين الأجهزة المختلفة. في عالم الـ embedded و ARM، البيانات دي بتيجي من **Device Tree** — ملف نصي بيوصف الـ hardware. في عالم الـ x86 و UEFI، البيانات دي بتيجي من **ACPI** — جداول بيكتبها الـ BIOS/firmware. في حالات تانية (testing، hotplug، platforms غريبة)، البيانات دي ممكن تيجي من **Software Nodes** — بيانات بتتعمل في memory جوّا الكود نفسه.

المشكلة الكبيرة: كل source من دول بيتكلم بلغة مختلفة. الـ driver اللي بيقرأ temperature sensor محتاج يعرف: هل البيانات جاية من Device Tree ولا ACPI ولا Software Node? لو كان كل driver لازم يتعامل مع الفروق دي بنفسه، هيبقى في كود مكرر ومعقد في كل مكان.

---

### الحل: الـ `fwnode_handle` — الـ Universal Translator

**الـ `fwnode.h`** بيعرّف **abstraction layer** — طبقة وسطى موحّدة. أي مصدر بيانات firmware (DT, ACPI, swnode) بيحوّل نفسه لـ `fwnode_handle`. الـ driver بعدين بيتكلم مع `fwnode_handle` بس، من غير ما يحتاج يعرف هو جاي من فين.

```
┌─────────────┐   ┌──────────────┐   ┌──────────────────┐
│ Device Tree │   │     ACPI     │   │  Software Nodes  │
│  (OF nodes) │   │  (DSDT/SSDT) │   │  (in-memory)     │
└──────┬──────┘   └──────┬───────┘   └────────┬─────────┘
       │                 │                     │
       ▼                 ▼                     ▼
┌──────────────────────────────────────────────────────┐
│              fwnode_handle  (الواجهة الموحّدة)        │
│         struct fwnode_handle + fwnode_operations     │
└──────────────────────────────────────────────────────┘
                          │
                          ▼
               ┌──────────────────┐
               │   Device Driver  │
               │  (لا يهمه المصدر) │
               └──────────────────┘
```

---

### إيه اللي في الملف بالظبط؟

#### 1. **`struct fwnode_handle`** — الـ Handle نفسه

ده الـ object الأساسي. بمثابة "بطاقة تعريف" لأي firmware node:

```c
struct fwnode_handle {
    struct fwnode_handle *secondary; /* fallback لو الـ primary مش لاقي المعلومة */
    const struct fwnode_operations *ops; /* الـ vtable - مين بيعمل إيه */
    struct device *dev;              /* الـ device المرتبط بيه */
    struct list_head suppliers;      /* مين بيوفرّله resources */
    struct list_head consumers;      /* مين بياخد منه resources */
    u8 flags;
};
```

الفكرة الذكية هنا: الـ `secondary` — لو device عنده بيانات في ACPI بس بعض البيانات ناقصة، ممكن يعمل chain مع swnode كـ fallback.

#### 2. **`struct fwnode_operations`** — الـ Virtual Table

ده الـ vtable بتاع الـ fwnode. كل implementation (OF, ACPI, swnode) بتعبّي الـ function pointers دي:

```c
struct fwnode_operations {
    struct fwnode_handle *(*get)(...);           /* reference counting */
    void (*put)(...);
    bool (*device_is_available)(...);            /* هل الـ device موجود ومتاح؟ */
    bool (*property_present)(...);               /* هل الـ property موجودة؟ */
    bool (*property_read_bool)(...);             /* اقرأ قيمة boolean */
    int (*property_read_int_array)(...);         /* اقرأ array من integers */
    int (*property_read_string_array)(...);      /* اقرأ array من strings */
    struct fwnode_handle *(*get_parent)(...);    /* الـ parent node */
    struct fwnode_handle *(*get_next_child_node)(...); /* iterate على الأبناء */
    /* ... graph endpoints, IRQs, MMIO, links ... */
    int (*add_links)(...);                       /* ابني dependency links */
};
```

#### 3. **`struct fwnode_link`** — الـ Dependency Graph

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  /* اللي بيوفّر (مثلاً: clock, regulator) */
    struct fwnode_handle *consumer;  /* اللي بياخد */
    u8 flags;
};
```

ده بيخلّي الكيرنل يعرف "الـ device A مش هيشتغل قبل الـ device B". مهم جداً لـ **probe ordering** — إيه اللي لازم يتحمّل قبل إيه.

#### 4. **`struct fwnode_endpoint`** — الـ Media Graph

```c
struct fwnode_endpoint {
    unsigned int port;  /* رقم البورت (مثلاً: camera port 0) */
    unsigned int id;    /* رقم الـ endpoint */
    const struct fwnode_handle *local_fwnode;
};
```

ده بيوصف ازاي الـ hardware components مربوطة ببعض في graph — مهم لـ camera pipelines و media subsystem.

#### 5. **الـ Macros** — الـ Safe Dispatch

```c
/* لو الـ fwnode مش عارف يعمل العملية، يرجع error مناسب */
#define fwnode_call_int_op(fwnode, op, ...)  \
    (fwnode_has_op(fwnode, op) ?             \
     (fwnode)->ops->op(fwnode, ...) : -ENXIO)
```

الـ macros دي بتعمل safe dispatch — لو الـ backend مش implement الـ operation، مش هيحصل crash.

---

### الـ Flags: حالة الـ Node

| Flag | المعنى |
|---|---|
| `FWNODE_FLAG_LINKS_ADDED` | الـ node اتفحص وتم بناء الـ dependency links بتاعته |
| `FWNODE_FLAG_NOT_DEVICE` | الـ node ده مش هيتحوّل لـ `struct device` |
| `FWNODE_FLAG_INITIALIZED` | الـ hardware اتعمّل initialize |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | محتاج الأبناء يتبندوا الأول قبل ما هو يقدر يشتغل |
| `FWNODE_FLAG_BEST_EFFORT` | يحاول يـ probe حتى لو في suppliers ناقصين |

---

### قصة حقيقية: Camera على ARM Device

تخيل Raspberry Pi عندها camera module مربوط. الـ Device Tree بيقول:

```
camera: camera@0 {
    compatible = "sony,imx219";
    port {
        cam_endpoint: endpoint {
            remote-endpoint = <&csi_endpoint>;
        };
    };
};
```

1. الـ OF subsystem بيقرأ الـ DT ويعمل `fwnode_handle` لـ `camera@0`.
2. الـ camera driver بيستخدم `fwnode_property_read_string(fwnode, "compatible", ...)` — مش بيفرق معاه إنه DT ولا ACPI.
3. الـ `graph_get_next_endpoint` بيمشي على الـ ports والـ endpoints.
4. الـ `fwnode_link` بيعمل dependency بين الـ camera والـ CSI controller — الـ CSI controller لازم يتـ probe الأول.

نفس الكود هيشتغل على Windows laptop بـ ACPI بدون أي تغيير في الـ driver!

---

### الملفات المكوّنة للـ Subsystem

#### الـ Core Headers

| الملف | الوظيفة |
|---|---|
| `include/linux/fwnode.h` | **الملف ده** — الـ low-level types والـ operations vtable |
| `include/linux/property.h` | الـ high-level API للـ drivers (ما يستخدمه الـ driver فعلاً) |

#### الـ Core Implementations

| الملف | الوظيفة |
|---|---|
| `drivers/base/property.c` | الـ unified property API implementation |
| `drivers/base/swnode.c` | الـ Software Nodes backend (in-memory firmware nodes) |

#### الـ Backend Implementations

| الملف | الوظيفة |
|---|---|
| `drivers/of/property.c` | Device Tree (OF) backend — بيـ implement الـ fwnode_operations لـ DT |
| `drivers/acpi/property.c` | ACPI backend — بيـ implement الـ fwnode_operations لـ ACPI |

#### الـ Related Subsystems

| الملف | الوظيفة |
|---|---|
| `drivers/media/v4l2-core/v4l2-fwnode.c` | الـ V4L2 camera graph parsing باستخدام fwnode |
| `include/media/v4l2-fwnode.h` | الـ V4L2 fwnode API |
| `drivers/net/mdio/fwnode_mdio.c` | الـ MDIO (network PHY) fwnode support |
| `include/linux/acpi.h` | الـ ACPI definitions المرتبطة |
## Phase 2: شرح الـ fwnode (Firmware Node) Framework

### المشكلة — ليه الـ fwnode موجود أصلاً؟

في الـ embedded Linux، الـ kernel لازم يعرف الـ hardware الموجود على الـ board — مين موجود، إيه خصائصه، وإزاي مرتبط بالأجزاء التانية. المشكلة إن في طرق مختلفة تماماً لوصف الـ hardware دا:

| الطريقة | بيتستخدم فين |
|---------|-------------|
| **Device Tree (OF)** | ARM، RISC-V، embedded boards |
| **ACPI** | x86، ARM servers |
| **Software Nodes (swnode)** | بدون firmware، بالكود مباشرةً |

كل واحدة من دول ليها structures مختلفة، APIs مختلفة، وطريقة تنظيم مختلفة. من غير abstraction layer موحد، كل driver هيتعامل مع الـ 3 طرق دول منفصلاً — كود متكرر، وـ driver مش portable.

---

### الحل — الـ fwnode Abstraction Layer

الـ kernel حل المشكلة دي بإنه عمل **abstraction layer** واحد اسمه `fwnode` (firmware node). الفكرة هي:

- كل node في الـ firmware description (سواء DT node أو ACPI device أو software node) بيتمثل بـ `struct fwnode_handle`.
- كل نوع firmware بيوفر **operations table** (vtable) تفيذ الـ operations دي بطريقته الخاصة.
- الـ driver بيتكلم مع الـ `fwnode_handle` فقط — مش عارف إيه الـ backend.

النتيجة: driver واحد يشتغل على ARM مع Device Tree، وعلى server مع ACPI، من غير ما يتغير سطر واحد.

---

### الـ Real-World Analogy — أعمق من السطح

تخيل إنك بتكتب تطبيق يقرأ ملفات — بس في أنواع مختلفة: PDF، Word، وPlain Text. بدل ما تكتب كود منفصل لكل نوع، بتعمل `interface Document` فيه:
- `read_page(n)`
- `get_title()`
- `get_properties()`

وكل نوع ملف بيعمل implementation خاصة بيه.

**الآن نربط الـ analogy بالـ kernel بالتفصيل:**

| الـ Analogy | الـ Kernel Concept |
|-------------|-------------------|
| الـ `Document` interface | `struct fwnode_handle` + `struct fwnode_operations` |
| ملف PDF | Device Tree node (`struct device_node` + OF backend) |
| ملف Word | ACPI device object (`struct acpi_device` + ACPI backend) |
| ملف Plain Text | Software Node (`struct software_node`) |
| الـ `read_page(n)` method | `property_read_int_array` operation |
| الكود اللي بيقرأ الملف | Driver بيستخدم `fwnode_property_read_u32()` |
| الـ document library | `drivers/base/property.c` |
| الـ plugin system | `fwnode_ops` registration |

لما الـ driver بيطلب `fwnode_property_read_u32(fwnode, "reg", &val)` — مش بيعرف إذا كان بيقرأ من DT أو ACPI. الـ vtable بتوجه الطلب للـ backend الصح.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Driver Code                             │
│  fwnode_property_read_u32(fwnode, "clock-frequency", &v)    │
└────────────────────────┬────────────────────────────────────┘
                         │ calls
                         ▼
┌─────────────────────────────────────────────────────────────┐
│             fwnode Abstraction Layer                        │
│         (drivers/base/property.c)                           │
│                                                             │
│   fwnode_call_int_op(fwnode, property_read_int_array, ...)  │
│         ──► fwnode->ops->property_read_int_array(...)       │
└────┬──────────────────┬──────────────────┬──────────────────┘
     │                  │                  │
     ▼                  ▼                  ▼
┌─────────┐      ┌─────────────┐    ┌────────────────┐
│  OF/DT  │      │    ACPI     │    │ Software Node  │
│ backend │      │   backend   │    │   backend      │
│         │      │             │    │                │
│of_fwnode│      │acpi_fwnode  │    │swnode_fwnode   │
│  _ops   │      │    _ops     │    │    _ops        │
└────┬────┘      └──────┬──────┘    └───────┬────────┘
     │                  │                   │
     ▼                  ▼                   ▼
┌──────────┐   ┌──────────────┐   ┌──────────────────┐
│  struct  │   │ struct acpi  │   │ struct software  │
│device_   │   │  _device     │   │   _node          │
│  node    │   │              │   │                  │
│(DT tree) │   │(ACPI tables) │   │ (in-kernel data) │
└──────────┘   └──────────────┘   └──────────────────┘
```

**الـ fwnode مش بيحل محل الـ DT أو ACPI** — هو بس layer فوقيهم بيوحد الـ access.

---

### الـ Core Abstractions — الـ Structs المهمة

#### 1. `struct fwnode_handle` — القلب

```c
struct fwnode_handle {
    struct fwnode_handle *secondary; /* fallback node لو مش لاقي property */
    const struct fwnode_operations *ops; /* الـ vtable */
    struct device *dev;   /* الـ device المرتبط (لو موجود) */
    struct list_head suppliers; /* fwnode links: المورّدين */
    struct list_head consumers; /* fwnode links: المستهلكين */
    u8 flags;             /* FWNODE_FLAG_* */
};
```

الـ `secondary` مهم جداً: لو الـ primary fwnode (مثلاً ACPI node) مش عارف يجيب property معينة، الـ kernel بيجرب الـ `secondary` (ممكن يكون software node بيكمّل الـ ACPI data). ده بيخلي الـ fwnode system **composable**.

#### 2. `struct fwnode_operations` — الـ vtable

```c
struct fwnode_operations {
    /* lifecycle */
    struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
    void (*put)(struct fwnode_handle *fwnode);

    /* device availability & matching */
    bool (*device_is_available)(const struct fwnode_handle *fwnode);
    const void *(*device_get_match_data)(...);
    bool (*device_dma_supported)(...);
    enum dev_dma_attr (*device_get_dma_attr)(...);

    /* property access */
    bool (*property_present)(...);
    bool (*property_read_bool)(...);
    int (*property_read_int_array)(...);
    int (*property_read_string_array)(...);

    /* tree navigation */
    const char *(*get_name)(...);
    const char *(*get_name_prefix)(...);
    struct fwnode_handle *(*get_parent)(...);
    struct fwnode_handle *(*get_next_child_node)(...);
    struct fwnode_handle *(*get_named_child_node)(...);

    /* cross-references */
    int (*get_reference_args)(...);

    /* graph (V4L2, media) */
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);

    /* hardware access */
    void __iomem *(*iomap)(...);
    int (*irq_get)(...);

    /* dependency tracking */
    int (*add_links)(struct fwnode_handle *fwnode);
};
```

الـ vtable مقسمة لـ 6 مجموعات وظيفية — مش مجرد قائمة functions عشوائية.

#### 3. `struct fwnode_link` — الـ Dependency Graph

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  /* الـ node اللي بيوفر resource */
    struct list_head s_hook;         /* hook في قايمة الـ supplier */
    struct fwnode_handle *consumer;  /* الـ node اللي بيستهلك */
    struct list_head c_hook;         /* hook في قايمة الـ consumer */
    u8 flags;                        /* FWLINK_FLAG_CYCLE | IGNORE */
};
```

الـ `fwnode_link` بيمثل علاقة dependency بين اتنين nodes — على سبيل المثال: الـ I2C controller هو supplier للـ sensor اللي عليه. الـ kernel بيستخدم العلاقات دي في **device probe ordering** — مش هيشغّل الـ consumer قبل ما الـ supplier يبقى جاهز.

```
fwnode_handle (I2C controller)
    suppliers list: []
    consumers list: ──► fwnode_link ──► fwnode_handle (Sensor)
                                            suppliers list: ──► fwnode_link (back)
```

#### 4. `struct fwnode_endpoint` — الـ Graph Topology

```c
struct fwnode_endpoint {
    unsigned int port;           /* رقم الـ port */
    unsigned int id;             /* رقم الـ endpoint في الـ port */
    const struct fwnode_handle *local_fwnode;
};
```

الـ **fwnode graph** — مفهوم مختلف عن الـ fwnode tree — بيصف الروابط الفيزيائية بين components. بيتستخدم في:
- الـ **V4L2 / Media subsystem**: كاميرا ◄──► ISP ◄──► Display
- الـ **MIPI CSI/DSI** connections
- الـ **Audio routing**

كل device ليه ports، وكل port فيه endpoints، وكل endpoint بيتوصل بـ endpoint ريموت في device تاني.

```
Camera Node                    ISP Node
  port@0                         port@0
    endpoint@0 ──────────────► endpoint@0
```

---

### الـ Dispatch Macros — إزاي الـ Call بيتوزع

الـ fwnode مش بيدي function calls مباشرة — بيستخدم macros بتتحقق الأول:

```c
/* لو الـ op موجود ومش NULL — ينفذ، غير كده يرجع error code */
#define fwnode_call_int_op(fwnode, op, ...)         \
    (fwnode_has_op(fwnode, op) ?                    \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) :    \
     (IS_ERR_OR_NULL(fwnode) ? -EINVAL : -ENXIO))

/* لو الـ op موجود — ينفذ، غير كده يرجع NULL */
#define fwnode_call_ptr_op(fwnode, op, ...)         \
    (fwnode_has_op(fwnode, op) ?                    \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : NULL)
```

الـ `fwnode_has_op` بيتحقق من 3 حاجات في خطوة واحدة:
1. الـ `fwnode` مش NULL ومش error pointer
2. الـ `fwnode->ops` موجود
3. الـ op المحدد نفسه مش NULL

ده بيحمي من NULL dereference في حالة backend مش عامل implement لكل الـ ops.

---

### الـ fwnode Flags — حالة الـ Node

```c
#define FWNODE_FLAG_LINKS_ADDED         BIT(0) /* تم parse الـ links بتاعته */
#define FWNODE_FLAG_NOT_DEVICE          BIT(1) /* مش هيتحول لـ struct device */
#define FWNODE_FLAG_INITIALIZED         BIT(2) /* الـ hardware اتـ initialized */
#define FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD BIT(3) /* lazm الـ children يتـ probe أول */
#define FWNODE_FLAG_BEST_EFFORT         BIT(4) /* يكمل حتى لو suppliers ناقصة */
#define FWNODE_FLAG_VISITED             BIT(5) /* اتزار في cycle detection */
```

**الـ `FWNODE_FLAG_NOT_DEVICE`** مهم: مش كل fwnode بيتحول لـ device — بعضهم nodes وصفية زي `pinctrl` groups أو `clocks`.

**الـ `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD`** بيتستخدم في devices زي الـ MFD (Multi-Function Device) اللي محتاجة الـ child devices يتبانوا الأول عشان تشتغل.

---

### العلاقة بين الـ Structs

```
                    ┌──────────────────────────┐
                    │    fwnode_handle         │
                    │  ┌──────────────────┐    │
                    │  │ ops ─────────────┼────┼──► fwnode_operations (vtable)
                    │  │ secondary ───────┼────┼──► fwnode_handle (fallback)
                    │  │ dev ─────────────┼────┼──► struct device
                    │  │ suppliers (list) │    │
                    │  │ consumers (list) │    │
                    │  │ flags            │    │
                    │  └──────────────────┘    │
                    └──────────────────────────┘
                              │ embedded in
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │device_node   │  │acpi_device   │  │software_node │
    │  .fwnode     │  │  .fwnode     │  │  .fwnode     │
    │  (OF/DT)     │  │  (ACPI)      │  │  (swnode)    │
    └──────────────┘  └──────────────┘  └──────────────┘
```

الـ `fwnode_handle` مش struct مستقل في الغالب — بيتعمله **embed** جوه الـ backend-specific struct. يعني `struct device_node` اللي هو الـ DT node فيه `struct fwnode_handle fwnode` جواه، والـ kernel بيعمل `container_of` يرجع للـ parent struct.

---

### إيه اللي الـ fwnode Framework بيمتلكه؟

| الـ fwnode يمتلك | الـ fwnode يفوّض |
|-----------------|-----------------|
| الـ unified API للـ properties | قراءة الـ DT/ACPI data الفعلية |
| الـ dependency tracking (fwnode_link) | تحديد الـ probe order للـ drivers |
| الـ graph topology (ports/endpoints) | الـ media pipeline routing (V4L2) |
| الـ lifecycle (get/put) | الـ reference counting implementation |
| الـ dispatch macros | الـ NULL/error checking logic |
| الـ secondary chaining | الـ fallback property lookup |

---

### ملاحظة على الـ Related Subsystems

- **الـ OF (Open Firmware) subsystem**: `include/linux/of.h` — بيوفر الـ DT backend للـ fwnode.
- **الـ ACPI subsystem**: `include/linux/acpi.h` — بيوفر الـ ACPI backend.
- **الـ device links subsystem**: مختلف عن الـ fwnode links — الـ fwnode links بيكونوا في مرحلة الـ firmware parsing قبل ما تتعمل `struct device`، وبعدين بيتحولوا لـ device links.
- **الـ property API**: `include/linux/property.h` — هو الـ public API اللي drivers بيستخدموه فوق الـ fwnode layer.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums — Cheatsheet

#### `FWNODE_FLAG_*` — flags الـ `fwnode_handle`

| Flag | Bit | المعنى |
|------|-----|--------|
| `FWNODE_FLAG_LINKS_ADDED` | 0 | الـ fwnode اتفرز مسبقاً عشان يضيف fwnode links |
| `FWNODE_FLAG_NOT_DEVICE` | 1 | الـ fwnode مش هيتحول لـ `struct device` أبداً |
| `FWNODE_FLAG_INITIALIZED` | 2 | الـ hardware المقابل للـ fwnode اتعمل له initialize |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | 3 | الـ driver محتاج الـ child devices تكون bound قبل ما هو يـ probe |
| `FWNODE_FLAG_BEST_EFFORT` | 4 | يـ probe بدري حتى لو في suppliers ناقصين — بس يرتب بس مع اللي عندهم drivers |
| `FWNODE_FLAG_VISITED` | 5 | اتزار أثناء cycle detection |

#### `FWLINK_FLAG_*` — flags الـ `fwnode_link`

| Flag | Bit | المعنى |
|------|-----|--------|
| `FWLINK_FLAG_CYCLE` | 0 | الـ link ده جزء من cycle — متعملش defer للـ probe |
| `FWLINK_FLAG_IGNORE` | 1 | تجاهل الـ link ده خالص حتى في cycle detection |

#### `enum dev_dma_attr`

| Value | المعنى |
|-------|--------|
| `DEV_DMA_NOT_SUPPORTED` | الجهاز مش بيدعم DMA |
| `DEV_DMA_NON_COHERENT` | الجهاز يعمل DMA لكن بدون cache coherency |
| `DEV_DMA_COHERENT` | الجهاز يعمل DMA مع cache coherency كاملة |

#### Macros مهمة

| Macro | الغرض |
|-------|--------|
| `NR_FWNODE_REFERENCE_ARGS` | الحد الأقصى للـ integer args في `fwnode_reference_args` = 16 |
| `SWNODE_GRAPH_PORT_NAME_FMT` | format اسم الـ port في software nodes = `"port@%u"` |
| `SWNODE_GRAPH_ENDPOINT_NAME_FMT` | format اسم الـ endpoint = `"endpoint@%u"` |
| `fwnode_has_op(fwnode, op)` | يتحقق إن الـ fwnode صحيح وعنده الـ op دي |
| `fwnode_call_int_op(...)` | يستدعي op وترجع int، أو `-EINVAL`/`-ENXIO` لو مفيش |
| `fwnode_call_bool_op(...)` | يستدعي op وترجع bool، أو `false` لو مفيش |
| `fwnode_call_ptr_op(...)` | يستدعي op وترجع pointer، أو `NULL` لو مفيش |
| `fwnode_call_void_op(...)` | يستدعي op بدون return value |

---

### الـ Structs المهمة

---

#### 1. `struct fwnode_handle`

**الغرض:** ده الـ base object لأي firmware node — سواء كان ACPI node، DT node، أو software node. كل الـ firmware topology بتبدأ منه.

```c
struct fwnode_handle {
    struct fwnode_handle *secondary;       // fallback fwnode لو الأول مجاوبش
    const struct fwnode_operations *ops;   // vtable: كل العمليات المتاحة
    struct device *dev;                    // الـ device المرتبط (for devlinks only)
    struct list_head suppliers;            // list من fwnode_link اللي بتوفر resources
    struct list_head consumers;            // list من fwnode_link اللي بتستهلك resources
    u8 flags;                              // FWNODE_FLAG_* bitmask
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `secondary` | `*fwnode_handle` | لو الـ ops الأولانية فشلت، يحاول مع الـ secondary (مثلاً ACPI + DT معاً) |
| `ops` | `*fwnode_operations` | الـ vtable — كل الـ backends بتحدد الـ ops بتاعتهم هنا |
| `dev` | `*device` | بيتحدد بس لما الـ fwnode يتربط بـ `struct device` |
| `suppliers` | `list_head` | رأس list للـ `fwnode_link` اللي فيها الـ fwnode ده consumer |
| `consumers` | `list_head` | رأس list للـ `fwnode_link` اللي فيها الـ fwnode ده supplier |
| `flags` | `u8` | bitmask من `FWNODE_FLAG_*` |

---

#### 2. `struct fwnode_operations`

**الغرض:** الـ vtable اللي كل firmware backend (ACPI، OF/DT، swnode) بيملّيه. بيوفر abstraction layer موحد للتعامل مع أي نوع firmware node.

| Function Pointer | الدور |
|-----------------|-------|
| `get` / `put` | reference counting |
| `device_is_available` | هل الـ device متاح في firmware؟ |
| `device_get_match_data` | جيب الـ driver match data |
| `device_dma_supported` | هل بيدعم DMA؟ |
| `device_get_dma_attr` | جيب نوع الـ DMA coherency |
| `property_present` | هل الـ property موجودة؟ |
| `property_read_bool` | اقرأ boolean property |
| `property_read_int_array` | اقرأ array من integers |
| `property_read_string_array` | اقرأ array من strings |
| `get_name` | اسم الـ node |
| `get_name_prefix` | prefix للطباعة |
| `get_parent` | الـ parent node |
| `get_next_child_node` | iteration على الـ children |
| `get_named_child_node` | child باسم معين |
| `get_reference_args` | اقرأ reference property مع args |
| `graph_get_next_endpoint` | iteration على الـ endpoints في graph |
| `graph_get_remote_endpoint` | الـ endpoint الـ remote المقابل |
| `graph_get_port_parent` | الـ parent للـ port node |
| `graph_parse_endpoint` | parse port/endpoint IDs |
| `iomap` | map MMIO region |
| `irq_get` | جيب رقم الـ IRQ |
| `add_links` | ابني fwnode links لكل الـ suppliers |

---

#### 3. `struct fwnode_link`

**الغرض:** يمثل **dependency link** بين firmware node supplier وآخر consumer. بيُستخدم في الـ device probe ordering.

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  // الـ node اللي بيوفر resource
    struct list_head s_hook;         // hook في قائمة consumers الخاصة بالـ supplier
    struct fwnode_handle *consumer;  // الـ node اللي بيستهلك
    struct list_head c_hook;         // hook في قائمة suppliers الخاصة بالـ consumer
    u8 flags;                        // FWLINK_FLAG_* bitmask
};
```

| Field | الدور |
|-------|-------|
| `supplier` | pointer للـ `fwnode_handle` المُوفِّر |
| `s_hook` | بيربط الـ link في `supplier->consumers` list |
| `consumer` | pointer للـ `fwnode_handle` المستهلك |
| `c_hook` | بيربط الـ link في `consumer->suppliers` list |
| `flags` | CYCLE أو IGNORE |

---

#### 4. `struct fwnode_endpoint`

**الغرض:** بيمثل **graph endpoint** في firmware graph — يُستخدم في تمثيل connections بين hardware blocks (زي camera sensor ↔ ISP).

```c
struct fwnode_endpoint {
    unsigned int port;                      // رقم الـ port
    unsigned int id;                        // رقم الـ endpoint جوه الـ port
    const struct fwnode_handle *local_fwnode; // الـ fwnode الخاص بالـ endpoint ده
};
```

---

#### 5. `struct fwnode_reference_args`

**الغرض:** لما property بتشير لـ fwnode تانية وبتحمل arguments معها (زي GPIO specifiers في DT).

```c
struct fwnode_reference_args {
    struct fwnode_handle *fwnode;       // الـ node المُشار إليه
    unsigned int nargs;                 // عدد الـ args الصحيحة
    u64 args[NR_FWNODE_REFERENCE_ARGS]; // القيم (max 16)
};
```

---

### رسم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                      fwnode_handle (A)                          │
│  ops ──────────────────────────────────► fwnode_operations      │
│  secondary ──────────────────────────► fwnode_handle (B)        │
│  dev ────────────────────────────────► struct device            │
│  consumers ──┐                                                   │
│  suppliers ──┤                                                   │
└──────────────┼──────────────────────────────────────────────────┘
               │
               │  (via list_head hooks)
               ▼
┌──────────────────────────────────────────────────────────────────┐
│                        fwnode_link                               │
│  supplier ──────────────────────────► fwnode_handle (supplier)  │
│  s_hook   ── in supplier->consumers list                        │
│  consumer ──────────────────────────► fwnode_handle (consumer)  │
│  c_hook   ── in consumer->suppliers list                        │
│  flags    (CYCLE / IGNORE)                                       │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    fwnode_reference_args                         │
│  fwnode ────────────────────────────► fwnode_handle (target)    │
│  nargs                                                           │
│  args[0..15]  (u64 values)                                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      fwnode_endpoint                             │
│  local_fwnode ──────────────────────► fwnode_handle (endpoint)  │
│  port  (u32)                                                     │
│  id    (u32)                                                     │
└─────────────────────────────────────────────────────────────────┘
```

#### الـ Secondary Chain

الـ `secondary` field بيسمح بـ chaining — مثلاً device عنده ACPI node وDT node:

```
fwnode_handle (ACPI)
  └── secondary ──► fwnode_handle (DT)
                      └── secondary ──► NULL
```

لو ops الأولانية مجابتش نتيجة (مثلاً property مش موجودة في ACPI)، الـ kernel بيجرب الـ secondary تلقائياً.

---

### رسم الـ Supplier/Consumer Graph

```
        fwnode_handle [clock-controller]
        ┌─────────────────────────┐
        │  consumers list head    │◄──── s_hook (fwnode_link A)
        └─────────────────────────┘
                                             │
                                    fwnode_link A
                                    ┌─────────────────┐
                                    │ supplier → clk  │
                                    │ consumer → uart │
                                    └─────────────────┘
                                             │
        fwnode_handle [uart]                 │
        ┌─────────────────────────┐          │
        │  suppliers list head    │◄──── c_hook (fwnode_link A)
        └─────────────────────────┘
```

---

### Lifecycle Diagram

#### دورة حياة الـ `fwnode_handle`

```
1. ALLOCATION
   ──────────
   Backend allocates container struct (e.g., acpi_device, device_node)
   which embeds fwnode_handle

2. INITIALIZATION
   ───────────────
   fwnode_init(fwnode, &my_ops)
     → fwnode->ops = &my_ops
     → INIT_LIST_HEAD(&fwnode->consumers)
     → INIT_LIST_HEAD(&fwnode->suppliers)

3. REGISTRATION / LINK BUILDING
   ──────────────────────────────
   ops->add_links(fwnode)
     → fwnode_link_add(consumer, supplier, flags)
       → allocates fwnode_link
       → list_add(&link->s_hook, &supplier->consumers)
       → list_add(&link->c_hook, &consumer->suppliers)
     → fwnode->flags |= FWNODE_FLAG_LINKS_ADDED

4. DEVICE CREATION
   ─────────────────
   device_add() associates struct device with fwnode
     → fwnode->dev = dev
   fwnode_dev_initialized(fwnode, true)
     → fwnode->flags |= FWNODE_FLAG_INITIALIZED

5. USAGE (property reads, graph traversal)
   ─────────────────────────────────────────
   fwnode_call_int_op(fwnode, property_read_int_array, ...)
     → checks fwnode_has_op()
     → calls fwnode->ops->property_read_int_array(fwnode, ...)

6. TEARDOWN
   ─────────
   fwnode_links_purge(fwnode)
     → removes all fwnode_link entries from suppliers/consumers lists
     → frees fwnode_link objects
   fw_devlink_purge_absent_suppliers(fwnode)
     → يزيل links لـ suppliers مش موجودين في firmware
   fwnode_dev_initialized(fwnode, false)
     → fwnode->flags &= ~FWNODE_FLAG_INITIALIZED
   Backend frees container struct → fwnode_handle يتحرر معاه
```

---

### Call Flow Diagrams

#### قراءة property

```
driver calls: fwnode_property_read_u32(fwnode, "reg", &val)
  │
  └─► fwnode_call_int_op(fwnode, property_read_int_array, "reg", sizeof(u32), &val, 1)
        │
        ├─ fwnode_has_op(fwnode, property_read_int_array)?
        │    ├── YES → fwnode->ops->property_read_int_array(fwnode, "reg", 4, &val, 1)
        │    │           │
        │    │           ├── [OF backend]  → reads DT property bytes
        │    │           ├── [ACPI backend] → reads _DSD property
        │    │           └── [swnode backend] → reads software_node_property
        │    │
        │    └── NO → return -ENXIO (ops missing)
        │         or → return -EINVAL (fwnode is NULL/ERR)
```

#### graph endpoint traversal

```
driver wants to iterate endpoints:

fwnode_graph_get_next_endpoint(fwnode, NULL)
  │
  └─► fwnode_call_ptr_op(fwnode, graph_get_next_endpoint, NULL)
        │
        └─► ops->graph_get_next_endpoint(fwnode, prev=NULL)
              │
              └─► returns endpoint fwnode_handle (port@0/endpoint@0)

fwnode_graph_get_remote_endpoint(ep_fwnode)
  │
  └─► ops->graph_get_remote_endpoint(ep_fwnode)
        │
        └─► returns fwnode of connected remote endpoint
```

#### بناء الـ devlinks

```
fw_devlink mechanism:

for each fwnode in firmware tree:
  ops->add_links(fwnode)
    │
    ├─► parse properties that reference other fwnodes
    │     (e.g., "clocks", "power-domains", "interconnects")
    │
    └─► fwnode_link_add(this_fwnode, supplier_fwnode, 0)
          │
          ├─► alloc fwnode_link
          ├─► link->supplier = supplier_fwnode
          ├─► link->consumer = this_fwnode
          ├─► list_add(&link->s_hook, &supplier_fwnode->consumers)
          └─► list_add(&link->c_hook, &this_fwnode->suppliers)

Later, driver core uses these links to order probe:
  supplier must probe before consumer
  (unless FWLINK_FLAG_CYCLE or FWLINK_FLAG_IGNORE)
```

---

### Locking Strategy

الـ `fwnode.h` نفسه **مش بيعرّف locks**، لكن الـ locking بيحصل في الطبقات اللي فوقه:

| Resource | Lock المستخدم | مين يمسكه |
|----------|--------------|-----------|
| الـ `fwnode_link` lists (`suppliers`/`consumers`) | `device_links_lock` (في `drivers/base/core.c`) | الـ driver core لما يعدل الـ lists |
| الـ `fwnode_handle->flags` | مش محتاج lock مستقل — بيتعدل في سياق واحد (probe/init) | الـ backend نفسه |
| الـ `fwnode_handle->dev` | protected by device model lifecycle (device_lock) | الـ driver core |
| الـ OF tree (DT backend) | `of_mutex` أو RCU read lock | OF subsystem |
| الـ ACPI namespace | ACPI subsystem locks | ACPI core |

**Lock Ordering المهم:**
```
device_lock(dev)
  └─► device_links_lock  (ممنوع تعكس الترتيب ده)
```

**الـ `fwnode_init()` و `fwnode_dev_initialized()`** بيتاخدوا بدون lock لأنهم بيتشغلوا في وقت init/teardown اللي مفيش race فيه.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Inline Functions

| Function | Category | الوظيفة |
|---|---|---|
| `fwnode_init()` | Initialization | تهيئة `fwnode_handle` بربط الـ ops وتهيئة الـ lists |
| `fwnode_dev_initialized()` | State Management | set/clear الـ `FWNODE_FLAG_INITIALIZED` |

#### الـ Exported Functions

| Function | Category | الوظيفة |
|---|---|---|
| `fwnode_link_add()` | Link Management | إنشاء `fwnode_link` بين consumer و supplier |
| `fwnode_links_purge()` | Cleanup | حذف كل الـ links المرتبطة بـ fwnode معين |
| `fw_devlink_purge_absent_suppliers()` | Cleanup | حذف الـ links لـ suppliers غير موجودين |
| `fw_devlink_is_strict()` | Query | استعلام عن الـ devlink strict mode |

#### الـ vtable Ops (`struct fwnode_operations`)

| Op | Return | الوظيفة |
|---|---|---|
| `get` | `fwnode_handle *` | زيادة الـ refcount |
| `put` | `void` | تقليل الـ refcount |
| `device_is_available` | `bool` | هل الجهاز متاح في الـ firmware? |
| `device_get_match_data` | `const void *` | بيانات الـ match من الـ firmware |
| `device_dma_supported` | `bool` | هل الجهاز يدعم DMA? |
| `device_get_dma_attr` | `enum dev_dma_attr` | نوع الـ DMA coherency |
| `property_present` | `bool` | هل الـ property موجودة? |
| `property_read_bool` | `bool` | قراءة property بوليانية |
| `property_read_int_array` | `int` | قراءة array من integers |
| `property_read_string_array` | `int` | قراءة array من strings |
| `get_name` | `const char *` | اسم الـ node |
| `get_name_prefix` | `const char *` | prefix للـ node (للطباعة) |
| `get_parent` | `fwnode_handle *` | الـ parent node |
| `get_next_child_node` | `fwnode_handle *` | التكرار على الـ children |
| `get_named_child_node` | `fwnode_handle *` | child باسم محدد |
| `get_reference_args` | `int` | قراءة reference + arguments من property |
| `graph_get_next_endpoint` | `fwnode_handle *` | التكرار على الـ graph endpoints |
| `graph_get_remote_endpoint` | `fwnode_handle *` | الـ remote endpoint |
| `graph_get_port_parent` | `fwnode_handle *` | parent الـ port node |
| `graph_parse_endpoint` | `int` | parse port/endpoint IDs |
| `iomap` | `void __iomem *` | map الـ MMIO resource |
| `irq_get` | `int` | الحصول على رقم الـ IRQ |
| `add_links` | `int` | إنشاء الـ supplier links |

#### الـ Dispatch Macros

| Macro | الوظيفة |
|---|---|
| `fwnode_has_op(fwnode, op)` | تحقق من وجود الـ op قبل الاستدعاء |
| `fwnode_call_int_op(fwnode, op, ...)` | استدعاء op بـ return int مع fallback errors |
| `fwnode_call_bool_op(fwnode, op, ...)` | استدعاء op بـ return bool مع fallback false |
| `fwnode_call_ptr_op(fwnode, op, ...)` | استدعاء op بـ return pointer مع fallback NULL |
| `fwnode_call_void_op(fwnode, op, ...)` | استدعاء op بدون return value |

---

### Group 1: Initialization & State Management

هذه المجموعة مسؤولة عن تهيئة الـ `fwnode_handle` وضبط حالته. يتم استدعاؤها من الـ firmware backends (OF، ACPI، software_node) عند تسجيل node جديد.

---

#### `fwnode_init`

```c
static inline void fwnode_init(struct fwnode_handle *fwnode,
                               const struct fwnode_operations *ops)
```

بتربط الـ `ops` vtable بالـ `fwnode_handle`، وبتهيئ الـ `consumers` و `suppliers` lists كـ empty circular lists باستخدام `INIT_LIST_HEAD`. لازم تتستدعى قبل أي عملية تانية على الـ fwnode.

**Parameters:**
- `fwnode` — الـ handle الجديد اللي هيتم تهيئته
- `ops` — الـ vtable اللي بيوفر backend-specific implementations زي `of_fwnode_ops` أو `acpi_fwnode_ops`

**Return:** لا يوجد

**Key details:**
- مش بتصفي كل الـ fields؛ الـ caller مسؤول عن ضبط `secondary` و `dev` و `flags` لو محتاج
- الـ `INIT_LIST_HEAD` بيستخدم `WRITE_ONCE` داخليًا — مناسب للـ concurrent environments لو الـ caller ضامن visibility
- مفيش locking هنا — الـ caller مسؤول عن أي synchronization قبل ما يعمل expose للـ node

**Who calls it:** كل backend بيعمل register fwnode جديد: `of_node_init()`, `acpi_fwnode_init()`, `software_node_init()`

---

#### `fwnode_dev_initialized`

```c
static inline void fwnode_dev_initialized(struct fwnode_handle *fwnode,
                                          bool initialized)
```

بتضبط أو بتمسح الـ `FWNODE_FLAG_INITIALIZED` flag في `fwnode->flags`. الـ flag ده بيعبّر عن إن الـ hardware المرتبط بالـ node اتهيأ فعلًا.

**Parameters:**
- `fwnode` — الـ node المستهدف
- `initialized` — `true` لضبط الـ flag، `false` لمسحه

**Return:** لا يوجد

**Key details:**
- بتعمل guard بـ `IS_ERR_OR_NULL` — لو الـ fwnode NULL أو error pointer، بترجع بصمت
- الـ flag ده بيستخدمه الـ device link machinery لتحديد ترتيب الـ probe
- مفيش locking ضمني — الـ caller لازم يضمن thread safety

**Who calls it:** الـ device core عند `device_add()` و `device_del()`

---

### Group 2: Link Management

هذه المجموعة تدير الـ `fwnode_link` objects اللي بتمثل علاقات الـ supplier/consumer بين الـ firmware nodes. الـ device link subsystem بيعتمد عليها لترتيب الـ probe.

---

#### `fwnode_link_add`

```c
int fwnode_link_add(struct fwnode_handle *con, struct fwnode_handle *sup,
                    u8 flags);
```

بتنشئ `fwnode_link` جديد بيربط `con` (consumer) بـ `sup` (supplier). الـ link بيتضاف في `sup->consumers` list وفي `con->suppliers` list عبر الـ `s_hook` و `c_hook` fields.

**Parameters:**
- `con` — الـ consumer fwnode (الجهاز اللي محتاج الـ supplier)
- `sup` — الـ supplier fwnode (الجهاز اللي بيوفر الـ resource)
- `flags` — `FWLINK_FLAG_CYCLE` أو `FWLINK_FLAG_IGNORE` أو صفر

**Return:** `0` عند النجاح، error code سالب عند الفشل (مثلًا `-ENOMEM` لو الـ allocation فشل، أو لو الـ link موجود بالفعل)

**Key details:**
- بتتحقق من إن الـ link مش موجود بالفعل قبل الإنشاء
- الـ `FWLINK_FLAG_CYCLE` بيمنع الـ probe deferral لو الـ link جزء من cycle
- الـ `FWLINK_FLAG_IGNORE` بيخلي الـ subsystem يتجاهل الـ link حتى في cycle detection
- الـ locking يتم داخل الـ implementation (في `drivers/base/core.c`) باستخدام الـ `fwnode_links_lock`

**Who calls it:** الـ `fwnode_ops->add_links()` callback من كل backend، و `fw_devlink` subsystem أثناء الـ probe cycle

**Pseudocode flow:**
```
fwnode_link_add(con, sup, flags):
    lock fwnode_links_lock
    search con->suppliers for existing link to sup
    if found → unlock, return 0 (idempotent)
    alloc fwnode_link
    link->supplier = sup
    link->consumer = con
    link->flags = flags
    list_add(&link->s_hook, &sup->consumers)
    list_add(&link->c_hook, &con->suppliers)
    unlock
    return 0
```

---

#### `fwnode_links_purge`

```c
void fwnode_links_purge(struct fwnode_handle *fwnode);
```

بتحذف كل الـ `fwnode_link` objects اللي فيها `fwnode` كـ supplier أو consumer. بتمشي على الـ `suppliers` list والـ `consumers` list وبتحرر كل entry.

**Parameters:**
- `fwnode` — الـ node اللي هيتم تنظيف links بتاعته

**Return:** لا يوجد

**Key details:**
- بتتعامل مع الـ links من الجانبين: حذف fwnode كـ consumer من suppliers بتوعه، وكـ supplier من consumers بتوعه
- لازم تتستدعى قبل ما يتحرر الـ fwnode نفسه
- الـ locking داخلي مثل `fwnode_link_add`

**Who calls it:** `fwnode_remove()` و device unregister paths في الـ device core

---

#### `fw_devlink_purge_absent_suppliers`

```c
void fw_devlink_purge_absent_suppliers(struct fwnode_handle *fwnode);
```

بتتفحص الـ suppliers المرتبطين بالـ `fwnode` وبتحذف الـ links لـ suppliers مش موجودين في الـ system (مش registered في الـ fwnode graph). مفيدة لتجنب probe deferral لأجهزة مش هتيجي أبدًا.

**Parameters:**
- `fwnode` — الـ consumer node اللي هيتم فحص suppliers بتاعته

**Return:** لا يوجد

**Key details:**
- "absent" هنا تعني: الـ supplier fwnode موجود في الـ firmware description بس مش registered كـ device
- الـ `fw_devlink` strict mode بيأثر على السلوك ده — في strict mode، غياب الـ supplier بيسبب error
- بتتعامل مع الـ recursive case: لو الـ absent supplier عنده هو كمان suppliers

**Who calls it:** الـ `driver_probe_device()` وحالات الـ late bind في الـ device core

---

#### `fw_devlink_is_strict`

```c
bool fw_devlink_is_strict(void);
```

بترجع `true` لو الـ `fw_devlink` kernel parameter اتضبط على `strict`. في الـ strict mode، أي device عنده supplier مش bound بيتأجل probe بتاعه بدون استثناء.

**Parameters:** لا يوجد

**Return:** `true` = strict mode مفعّل، `false` = الـ default permissive behavior

**Key details:**
- القيمة بتيجي من الـ kernel command line: `fw_devlink=strict`
- الـ strict mode مهم لـ production systems اللي عايزة guarantee صح للـ probe ordering
- بتستخدمها أجزاء تانية من الـ device core عند قرار defer أو لا

**Who calls it:** `fw_devlink_purge_absent_suppliers()` وأي code بيحتاج يقرر سياسة الـ devlink

---

### Group 3: Dispatch Macros

هذه الـ macros هي الـ "glue" بين الـ generic property API والـ backend-specific ops. كلها بتتحقق من وجود الـ op قبل الاستدعاء وبتوفر fallback مناسب.

---

#### `fwnode_has_op`

```c
#define fwnode_has_op(fwnode, op) \
    (!IS_ERR_OR_NULL(fwnode) && (fwnode)->ops && (fwnode)->ops->op)
```

بيتحقق من ثلاث شروط بالترتيب: الـ fwnode مش NULL/error، الـ ops pointer مش NULL، والـ op المحدد نفسه مش NULL. الـ short-circuit evaluation بيحمي من null deref.

**Usage example:**
```c
if (fwnode_has_op(fwnode, get_name))
    name = fwnode->ops->get_name(fwnode);
```

---

#### `fwnode_call_int_op`

```c
#define fwnode_call_int_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : \
     (IS_ERR_OR_NULL(fwnode) ? -EINVAL : -ENXIO))
```

الـ fallback بيفرق بين حالتين: لو الـ fwnode نفسه invalid بترجع `-EINVAL`، لو valid بس الـ op مش موجود بترجع `-ENXIO` (operation not supported). ده أدق من رجوع `-ENOENT` عشان يعبّر إن الـ node موجود بس الـ backend مش بيدعم العملية دي.

---

#### `fwnode_call_bool_op`

```c
#define fwnode_call_bool_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : false)
```

الـ fallback هنا `false` — معناها "الخاصية دي مش موجودة" وهو السلوك الصح لمعظم حالات الـ boolean properties.

---

#### `fwnode_call_ptr_op`

```c
#define fwnode_call_ptr_op(fwnode, op, ...) \
    (fwnode_has_op(fwnode, op) ? \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) : NULL)
```

الـ fallback `NULL` — الـ caller لازم يتحقق من النتيجة قبل الاستخدام. كتير من الـ iteration APIs زي `get_next_child_node` بتستخدم هذا الـ macro.

---

#### `fwnode_call_void_op`

```c
#define fwnode_call_void_op(fwnode, op, ...) \
    do { \
        if (fwnode_has_op(fwnode, op)) \
            (fwnode)->ops->op(fwnode, ## __VA_ARGS__); \
    } while (false)
```

بيستخدم `do { } while(false)` idiom عشان يكون safe في `if/else` blocks بدون أقواس. الـ `get` و `put` ops بتستخدم هذا الـ macro عادةً.

---

### Group 4: الـ `fwnode_operations` vtable

الـ `struct fwnode_operations` هو قلب الـ fwnode abstraction. كل firmware backend بيوفر implementation خاصة بيه.

---

#### Sub-group 4.1: Reference Counting

```c
struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
void (*put)(struct fwnode_handle *fwnode);
```

الـ **`get`** بيزود الـ refcount للـ node وبيرجع نفس الـ pointer (أو NULL لو فشل). الـ **`put`** بيقلل الـ refcount وممكن يحرر الـ memory لو وصل صفر.

- في OF: بيستخدم `of_node_get()` / `of_node_put()` اللي بيعملوا wrap على `kobject_get/put`
- في ACPI: بيستخدم `acpi_get_acpi_dev()` / `acpi_dev_put()`
- في software_node: counting بسيط

---

#### Sub-group 4.2: Device Availability & Match

```c
bool (*device_is_available)(const struct fwnode_handle *fwnode);
const void *(*device_get_match_data)(const struct fwnode_handle *fwnode,
                                     const struct device *dev);
bool (*device_dma_supported)(const struct fwnode_handle *fwnode);
enum dev_dma_attr (*device_get_dma_attr)(const struct fwnode_handle *fwnode);
```

- **`device_is_available`**: في DT بيتحقق من `status = "okay"` أو غياب الـ status property. في ACPI بيتحقق من `_STA` method.
- **`device_get_match_data`**: بيرجع الـ `driver_data` من أول entry في `of_match_table` أو `acpi_match_table` يطابق الجهاز.
- **`device_dma_supported`** و **`device_get_dma_attr`**: بيحددوا هل الجهاز يدعم DMA ونوع الـ coherency (non-coherent / coherent). بيستخدمهم `of_dma_configure()` وما يعادله في ACPI.

---

#### Sub-group 4.3: Property Access

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

- **`property_present`**: وجود الـ property بغض النظر عن قيمتها.
- **`property_read_bool`**: قراءة boolean property — في DT ده property موجود بدون قيمة.
- **`property_read_int_array`**: الـ `elem_size` بيحدد حجم كل عنصر (1, 2, 4, 8 bytes). لو `val == NULL` بترجع عدد العناصر المتاحة. لو `nval == 0` بترجع عدد العناصر أيضًا.
- **`property_read_string_array`**: نفس نمط الـ `int_array` — `val == NULL` بترجع العدد.

**Pattern للقراءة الآمنة:**
```c
/* أول: عرّف العدد */
int count = fwnode_call_int_op(fwnode, property_read_int_array,
                               "reg", 4, NULL, 0);
if (count < 0)
    return count;

/* تاني: اقرأ البيانات */
u32 *buf = kcalloc(count, sizeof(u32), GFP_KERNEL);
fwnode_call_int_op(fwnode, property_read_int_array,
                   "reg", 4, buf, count);
```

---

#### Sub-group 4.4: Node Navigation

```c
const char *(*get_name)(const struct fwnode_handle *fwnode);
const char *(*get_name_prefix)(const struct fwnode_handle *fwnode);
struct fwnode_handle *(*get_parent)(const struct fwnode_handle *fwnode);
struct fwnode_handle *(*get_next_child_node)(const struct fwnode_handle *fwnode,
                                             struct fwnode_handle *child);
struct fwnode_handle *(*get_named_child_node)(const struct fwnode_handle *fwnode,
                                              const char *name);
```

- **`get_name`**: اسم الـ node — في DT هو الجزء قبل `@` في node name.
- **`get_name_prefix`**: prefix للطباعة — في DT غالبًا `"/"` أو فارغ، في ACPI غالبًا `"\\"`
- **`get_parent`**: بيرجع `fwnode_handle` للـ parent مع زيادة الـ refcount — الـ caller مسؤول عن `put`.
- **`get_next_child_node`**: iterator function — لو `child == NULL` بترجع أول child. الـ implementation بتعمل `put` للـ `child` السابق وبترجع التالي برفع refcount.
- **`get_named_child_node`**: بتدور على الـ children وبترجع أول واحد اسمه يطابق `name`، مع رفع الـ refcount.

---

#### Sub-group 4.5: Reference Arguments

```c
int (*get_reference_args)(const struct fwnode_handle *fwnode,
                          const char *prop, const char *nargs_prop,
                          unsigned int nargs, unsigned int index,
                          struct fwnode_reference_args *args);
```

بيقرأ reference property (زي `clocks`, `dmas`, `gpios` في DT) ويملى `struct fwnode_reference_args` بالـ target fwnode والـ integer arguments.

**Parameters:**
- `fwnode` — الـ consumer node
- `prop` — اسم الـ property زي `"clocks"`
- `nargs_prop` — اسم الـ property اللي فيها عدد الـ arguments زي `"#clock-cells"`، أو NULL لو fixed
- `nargs` — عدد الـ arguments لو `nargs_prop == NULL`
- `index` — index الـ entry المطلوب (الـ property ممكن يكون فيها أكتر من reference)
- `args` — output structure

**Return:** `0` عند النجاح، `-ENOENT` لو الـ property مش موجودة، `-EINVAL` لو الـ format غلط

**الـ `fwnode_reference_args` structure:**
```c
struct fwnode_reference_args {
    struct fwnode_handle *fwnode; /* الـ supplier/target node */
    unsigned int nargs;           /* عدد الـ args المقروءة */
    u64 args[NR_FWNODE_REFERENCE_ARGS]; /* max 16 argument */
};
```

---

#### Sub-group 4.6: Graph (Port/Endpoint) Operations

هذه الـ ops تدعم الـ OF graph bindings (V4L2، MIPI CSI-2، display pipelines إلخ) حيث الأجهزة مترابطة بـ endpoints.

```c
struct fwnode_handle *(*graph_get_next_endpoint)(
    const struct fwnode_handle *fwnode,
    struct fwnode_handle *prev);

struct fwnode_handle *(*graph_get_remote_endpoint)(
    const struct fwnode_handle *fwnode);

struct fwnode_handle *(*graph_get_port_parent)(
    struct fwnode_handle *fwnode);

int (*graph_parse_endpoint)(const struct fwnode_handle *fwnode,
                            struct fwnode_endpoint *endpoint);
```

```
Device Node
  └── port@0          ← port node
        └── endpoint@0  ← local endpoint
              └── remote-endpoint → endpoint@0 في device تاني
                                        └── port@0
                                              └── Device Node (remote)
```

- **`graph_get_next_endpoint`**: iterator على الـ endpoints — بتدور في كل الـ ports وترجع endpoints بالترتيب. لو `prev == NULL` بترجع أول endpoint.
- **`graph_get_remote_endpoint`**: من local endpoint بترجع الـ remote endpoint (الطرف التاني من الـ connection).
- **`graph_get_port_parent`**: من port node بترجع الـ device node (الـ parent اللي فوق الـ port).
- **`graph_parse_endpoint`**: بتملى `struct fwnode_endpoint` بالـ `port` و `id` من أسماء الـ nodes.

---

#### Sub-group 4.7: Resource Access

```c
void __iomem *(*iomap)(struct fwnode_handle *fwnode, int index);
int (*irq_get)(const struct fwnode_handle *fwnode, unsigned int index);
```

- **`iomap`**: بتعمل map لـ MMIO region بالـ `index` المحدد من الـ firmware resources. بترجع virtual address أو `IOMEM_ERR_PTR` عند الفشل.
- **`irq_get`**: بترجع الـ Linux IRQ number (virtual IRQ) بعد ترجمته من الـ firmware representation (DT interrupt specifier أو ACPI `_CRS`).

---

#### Sub-group 4.8: Link Creation

```c
int (*add_links)(struct fwnode_handle *fwnode);
```

بيحلل الـ firmware node ويعمل `fwnode_link_add()` لكل supplier مذكور في الـ properties. ده هو الـ entry point لبناء الـ dependency graph.

**Return:** `0` عند النجاح، `-ENODEV` لو supplier مش موجود ومحتاج defer، error code تاني عند فشل حقيقي.

**Who calls it:** الـ `fw_devlink` subsystem أثناء `device_add()` أو `fwnode_link_to_devlink()`.

---

### Group 5: الـ Flags والـ Constants

#### `fwnode_handle` Flags

| Flag | Bit | المعنى |
|---|---|---|
| `FWNODE_FLAG_LINKS_ADDED` | 0 | الـ fwnode اتحلل بالفعل لإضافة الـ links — تجنب الـ duplicate |
| `FWNODE_FLAG_NOT_DEVICE` | 1 | هذا الـ node مش هيترجم لـ `struct device` أبدًا (مثلًا port nodes) |
| `FWNODE_FLAG_INITIALIZED` | 2 | الـ hardware المرتبط اتهيأ |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | 3 | الـ driver محتاج كل الـ child devices تكون bound قبل ما يتعمل probe |
| `FWNODE_FLAG_BEST_EFFORT` | 4 | probe مبكر — الـ missing suppliers مش blockers إلا لو عندهم drivers |
| `FWNODE_FLAG_VISITED` | 5 | اتزار أثناء cycle detection في الـ fw_devlink |

#### `fwnode_link` Flags

| Flag | Bit | المعنى |
|---|---|---|
| `FWLINK_FLAG_CYCLE` | 0 | الـ link جزء من dependency cycle — مش هيعمل defer للـ probe |
| `FWLINK_FLAG_IGNORE` | 1 | تجاهل تام — حتى في cycle detection |

#### `enum dev_dma_attr`

| Value | المعنى |
|---|---|
| `DEV_DMA_NOT_SUPPORTED` | الجهاز مش بيدعم DMA خالص |
| `DEV_DMA_NON_COHERENT` | DMA بدون hardware coherency — محتاج explicit cache ops |
| `DEV_DMA_COHERENT` | DMA مع hardware coherency كاملة |

---

### Group 6: الـ Graph Node Naming Macros

```c
#define SWNODE_GRAPH_PORT_NAME_FMT      "port@%u"
#define SWNODE_GRAPH_ENDPOINT_NAME_FMT  "endpoint@%u"
```

الـ `software_node` backend لازم يستخدم نفس naming convention للـ ports والـ endpoints زي الـ OF graph. الـ macros دي بتضمن consistency بين الـ backends. الـ `NR_FWNODE_REFERENCE_ARGS = 16` بيحدد الحد الأقصى لعدد arguments في أي reference property.
## Phase 5: دليل الـ Debugging الشامل

الـ `fwnode` subsystem هو الـ abstraction layer اللي بيوصل بين الـ firmware descriptions (ACPI, Device Tree, software nodes) والـ kernel device model. الـ debugging بتاعه بيتركز على ٣ محاور: صحة الـ fwnode graph، الـ device links/suppliers، وصحة الـ property reading.

---

### Software Level

#### 1. debugfs Entries

الـ fwnode/devlink debugging بيظهر تحت `/sys/kernel/debug/` و `/sys/kernel/debug/devices/`:

```bash
# عرض كل الـ device links الموجودة في الـ kernel
ls /sys/kernel/debug/devices/

# الـ fwnode links بتظهر من خلال devlink interface
cat /sys/kernel/debug/devices/*/fwnode/
```

الـ entry الأهم للـ fwnode هو:

| Path | المحتوى | طريقة القراءة |
|------|---------|---------------|
| `/sys/kernel/debug/devices/*/suppliers` | list الـ supplier fwnodes | `cat` |
| `/sys/kernel/debug/devices/*/consumers` | list الـ consumer fwnodes | `cat` |
| `/sys/kernel/debug/acpi/` | ACPI namespace كامل | `ls` + `cat` |
| `/sys/kernel/debug/of/` | Device Tree nodes | `ls` recursively |

```bash
# عرض ACPI namespace كامل مع properties
cat /sys/kernel/debug/acpi/acpica

# Device Tree debug info
find /sys/kernel/debug/of/ -name "properties" -exec cat {} \;
```

#### 2. sysfs Entries

```bash
# فحص الـ fwnode المرتبط بـ device معين
ls -la /sys/bus/*/devices/*/of_node     # DT-based fwnode
ls -la /sys/bus/*/devices/*/firmware_node  # ACPI fwnode

# فحص الـ device links (supplier/consumer)
cat /sys/bus/*/devices/*/device_links/*/status
cat /sys/bus/*/devices/*/device_links/*/auto_remove_consumers
cat /sys/bus/*/devices/*/device_links/*/supplier
cat /sys/bus/*/devices/*/device_links/*/consumer
```

مثال على output طبيعي:

```
/sys/devices/platform/soc/fe300000.mmc/device_links/0--1/status
>> 32   # DL_STATE_ACTIVE = 32
```

| sysfs Path | المعنى |
|-----------|--------|
| `device_links/*/status` | حالة الـ link: 1=dormant, 2=available, 4=consumer_probe, 8=active, 32=active |
| `device_links/*/flags` | `DL_FLAG_STATELESS`, `DL_FLAG_AUTOREMOVE_CONSUMER` إلخ |
| `of_node -> /sys/firmware/devicetree/` | symlink للـ DT node |

```bash
# فحص كل الـ device links في الـ system
find /sys/devices -name "device_links" -type d | while read d; do
    echo "=== $d ==="
    ls "$d"/
done
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# تفعيل الـ fwnode/devlink events
cd /sys/kernel/debug/tracing

# الـ events المتاحة للـ device + fwnode
grep -r "fwnode\|devlink\|fw_devlink" available_events

# تفعيل كل الـ device probe events (بيشمل fwnode resolution)
echo 1 > events/device/enable

# تفعيل driver probe tracing
echo 1 > events/initcall/enable

# تفعيل الـ supplier/consumer link events
echo 1 > events/bus/enable

# مثال كامل: تتبع probe failures المرتبطة بـ fwnode
echo 0 > trace
echo 1 > events/device/enable
echo 1 > events/bus/enable
echo function > current_tracer
echo "device_link*:fwnode*:fw_devlink*" > set_ftrace_filter
cat trace_pipe
```

```bash
# تتبع فحص الـ fwnode ops calls
echo ':fwnode_call_int_op*' >> set_ftrace_filter
echo function > current_tracer
cat trace_pipe | grep -E "fwnode|devlink"
```

#### 4. printk / Dynamic Debug

```bash
# تفعيل dynamic debug لكل الـ fwnode/devlink messages
echo 'file drivers/base/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/devlink.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/swnode.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل ACPI fwnode messages
echo 'file drivers/acpi/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/acpi/scan.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل DT/OF fwnode messages
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/base.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل fwnode-related modules دفعة واحدة
echo 'module fwnode +p' > /sys/kernel/debug/dynamic_debug/control

# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk
```

الـ kernel command line لتفعيل debugging من البداية:

```bash
# في /etc/default/grub أو bootloader
GRUB_CMDLINE_LINUX="fw_devlink=on loglevel=8 dyndbg=+p"

# تفعيل fw_devlink strict mode (يكشف dependencies مفقودة)
GRUB_CMDLINE_LINUX="fw_devlink=on"

# أو عبر sysfs runtime
echo on > /sys/bus/platform/devices/.../firmware_node
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_DRIVER` | تفعيل driver core debug messages |
| `CONFIG_DEBUG_DEVRES` | تتبع الـ devres allocations المرتبطة بـ devices |
| `CONFIG_DEVICE_PRIVATE` | دعم private memory للـ devices (يحتاج debugging خاص) |
| `CONFIG_OF_DYNAMIC` | دعم runtime DT changes + debugging |
| `CONFIG_OF_UNITTEST` | unit tests للـ OF/DT fwnode |
| `CONFIG_ACPI_DEBUG` | ACPI verbose debugging |
| `CONFIG_ACPI_DEBUGGER` | interactive ACPI debugger |
| `CONFIG_DEBUG_FS` | تفعيل debugfs (مطلوب لأغلب ما سبق) |
| `CONFIG_PROVE_LOCKING` | كشف lock ordering violations في fwnode ops |
| `CONFIG_LOCKDEP` | الـ full lock dependency tracking |
| `CONFIG_KASAN` | كشف memory errors في fwnode allocations |
| `CONFIG_KCSAN` | كشف data races في concurrent fwnode access |
| `CONFIG_FW_DEVLINK_DEBUG` | debug output لـ fw_devlink (إذا موجود في kernel version) |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "DEBUG_DRIVER|OF_UNITTEST|ACPI_DEBUG|DEBUG_DEVRES"
```

#### 6. devlink والـ Tools الخاصة بالـ Subsystem

```bash
# fw_devlink: التحكم في سلوك الـ fwnode device links
cat /sys/bus/platform/devices/*/firmware_node/

# تغيير fw_devlink mode runtime
echo "on"       > /sys/bus/*/fw_devlink   # strict: defer probe حتى كل suppliers جاهزة
echo "permissive" > /sys/bus/*/fw_devlink # relaxed: probe حتى لو supplier مش موجود

# أداة: evtest لفحص device nodes المرتبطة بـ input fwnodes
evtest /dev/input/eventX

# أداة: acpidump + acpixtract لـ ACPI fwnode
acpidump > acpi.dat
acpixtract -a acpi.dat
iasl -d DSDT.dat   # decompile إلى readable ASL

# أداة: dtc لفحص Device Tree fwnodes
dtc -I fs /sys/firmware/devicetree/base/ > current.dts

# أداة: fwts (Firmware Test Suite)
fwts acpi -r results.log

# أداة: i2cdetect للـ I2C devices المرتبطة بـ fwnodes
i2cdetect -y 1
```

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `Waiting for supplier <fwnode>` | الـ device في انتظار supplier لم يُـprobe بعد | تحقق أن الـ supplier module محمّل، أو أضف `fw_devlink=permissive` |
| `fwnode ops not set` | `fwnode_handle->ops == NULL` — الـ fwnode غير مكتمل التهيئة | تحقق من `fwnode_init()` call في الـ driver |
| `ERROR: Failed to create fwnode link` | فشل `fwnode_link_add()` — غالباً memory allocation | فحص `/proc/meminfo`، ربما OOM |
| `could not find phandle` | الـ DT reference لـ phandle غير موجود في الـ DT | تحقق من الـ DTS file، الـ phandle مش معرّف |
| `of_fwnode_get: could not get ref` | فشل الحصول على reference لـ OF fwnode | الـ node مش موجود أو deleted |
| `ACPI: No handler for region` | الـ ACPI fwnode يحاول access region غير مدعومة | تحقق من الـ ACPI tables |
| `fw_devlink: Cycle detected` | الـ fwnode links فيها circular dependency (`FWLINK_FLAG_CYCLE`) | راجع الـ DTS/ACPI topology، تحقق من الـ supplier/consumer relations |
| `swnode: duplicate node name` | software node بنفس الاسم مسجّل مرتين | تحقق من الـ driver registration order |
| `fwnode_handle is NULL` | الـ device مش عنده fwnode مرتبط | تحقق من `device_set_node()` أو `of_node` assignment |
| `graph_get_next_endpoint: no endpoint` | لا يوجد endpoint في الـ fwnode graph | تحقق من الـ port/endpoint في DTS |

#### 8. أماكن إضافة `dump_stack()` و `WARN_ON()`

```c
/* في fwnode_init() — تحقق من صحة التهيئة */
static inline void fwnode_init(struct fwnode_handle *fwnode,
                               const struct fwnode_operations *ops)
{
    WARN_ON(!ops); /* ops يجب أن يكون غير NULL */
    fwnode->ops = ops;
    INIT_LIST_HEAD(&fwnode->consumers);
    INIT_LIST_HEAD(&fwnode->suppliers);
}

/* عند استدعاء fwnode_call_int_op — تحقق من الـ fwnode قبل الاستدعاء */
/* في drivers بتاعتك: */
if (WARN_ON(IS_ERR_OR_NULL(fwnode))) {
    dump_stack();
    return -EINVAL;
}

/* عند fwnode_link_add — تحقق من cycles */
WARN_ON(con == sup); /* self-link = bug */

/* في fwnode_dev_initialized — تحقق من double-init */
WARN_ON(initialized && (fwnode->flags & FWNODE_FLAG_INITIALIZED));
```

نقاط استراتيجية في الـ kernel source للـ breakpoints/tracing:

- `drivers/base/property.c` → `fwnode_property_read_*` — عند فشل قراءة property
- `drivers/base/core.c` → `device_link_add()` — عند إنشاء device link
- `drivers/base/devlink.c` → `fw_devlink_link_device()` — ربط الـ fwnode بالـ device
- `drivers/of/property.c` → `of_fwnode_get_reference_args()` — عند فشل phandle resolution

---

### Hardware Level

#### 1. التحقق أن الـ Hardware State يطابق الـ Kernel State

```bash
# مقارنة الـ devices المكتشفة في DT مع الـ probed devices
# الـ devices في DT:
find /sys/firmware/devicetree/base/ -name "compatible" -exec grep -l "." {} \; | wc -l

# الـ devices المرتبطة بـ fwnodes والمـ probed فعلاً:
find /sys/bus/*/devices/ -name "of_node" | wc -l

# مقارنة ACPI namespace مع الـ probed devices
cat /sys/kernel/debug/acpi/acpica | grep -c "Device"
ls /sys/bus/acpi/devices/ | wc -l

# فحص unbound devices (عندها fwnode بس مش عندها driver)
find /sys/bus/*/devices/ -name "driver" | wc -l
find /sys/bus/*/devices/ -maxdepth 1 -type l | wc -l
```

#### 2. Register Dump Techniques

للـ devices المرتبطة بـ fwnode عندها memory-mapped registers:

```bash
# قراءة register بـ devmem2 (يحتاج تثبيت)
devmem2 0xFE300000 w    # قراءة word من base address الـ device

# قراءة مباشرة من /dev/mem (يحتاج CONFIG_STRICT_DEVMEM=n)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE300000 / 4)) 2>/dev/null | xxd

# باستخدام io utility (من package ioport)
io -4 -r 0xFE300000

# من داخل الـ kernel باستخدام ioremap في driver مؤقت:
```

```c
/* snippet لقراءة registers من kernel module مؤقت للـ debugging */
void __iomem *base;
u32 val;

base = ioremap(0xFE300000, 0x1000); /* map 4KB */
if (!base) {
    pr_err("ioremap failed\n");
    return -ENOMEM;
}
val = readl(base + 0x00); /* قراءة register 0 */
pr_info("REG[0x00] = 0x%08x\n", val);
iounmap(base);
```

```bash
# فحص الـ iomem map لمعرفة base addresses
cat /proc/iomem | grep -i "mmc\|i2c\|spi\|gpio"
# مثال output:
# fe300000-fe3000ff : fe300000.mmc
```

#### 3. Logic Analyzer / Oscilloscope Tips

للتحقق من الـ hardware عند debugging fwnode graph endpoints:

```
Protocol       | Signal Lines         | What to Check
---------------|---------------------|------------------------------------------
I2C            | SDA, SCL            | ACK/NACK بعد address، stretch conditions
SPI            | MOSI,MISO,CLK,CS    | timing بين CS assertion والـ first clock
UART           | TX, RX              | baud rate مطابق لـ DTS clock-frequency
MIPI CSI-2     | D-PHY lanes         | lane enable sequence، LP/HS transition
I2S/SAI        | BCLK,LRCK,DATA     | sample rate = DTS-specified rate
GPIO interrupt | GPIO pin            | edge type يطابق "interrupts" property في DTS
```

نقاط القياس على الـ logic analyzer:

1. عند device probe: راقب الـ reset GPIO (مذكور في fwnode كـ `reset-gpios`)
2. الـ power sequence: `supply-gpios` / `enable-gpios` يجب تفعيلها بالترتيب الصحيح
3. الـ clock signal: تحقق أن الـ frequency مطابقة لـ `clock-frequency` في DTS

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة Hardware | Pattern في الـ Kernel Log | التشخيص |
|-----------------|--------------------------|---------|
| الـ device مش موجود فعلياً لكن في DT | `probe deferred` ثم timeout | `i2cdetect` أو scope على SDA/SCL |
| voltage supply غلط | `regulator: failed to enable` | قياس VDDIO بالـ multimeter |
| clock غلط أو مش شغال | `clk: failed to prepare` | oscilloscope على CLK pin |
| GPIO reset مش شغال | device يـfail بعد probe مباشرة | logic analyzer على RESET# pin |
| interrupt مش واصل | `irq: no irq handler` | oscilloscope على INT pin |
| DMA coherency مشكلة | `DMA-API: device not supported` | الـ `dev_dma_attr` غلط في DT |

#### 5. Device Tree Debugging

```bash
# تحويل DT الحالي المحمّل إلى readable DTS
dtc -I fs -O dts /sys/firmware/devicetree/base/ -o /tmp/current.dts 2>/dev/null

# مقارنة DTS الأصلية مع الـ compiled DTB المحمّل
diff original.dts /tmp/current.dts

# فحص node معين بالاسم
find /sys/firmware/devicetree/base/ -name "*.mmc" -o -name "fe300000*" | xargs ls

# قراءة property معينة من DT node
hexdump -C /sys/firmware/devicetree/base/soc/mmc@fe300000/clock-frequency
# output: 00000000  00 bb 8f 00                    (= 12288000 Hz)

# التحقق من الـ phandle references
cat /sys/firmware/devicetree/base/soc/mmc@fe300000/clocks | xxd

# فحص الـ fwnode graph endpoints في DT
find /sys/firmware/devicetree/base/ -name "endpoint*" -type d

# التحقق من port/endpoint naming (يجب يتطابق مع SWNODE_GRAPH_PORT_NAME_FMT)
ls /sys/firmware/devicetree/base/*/port@0/endpoint@0/
```

```bash
# تحقق أن الـ DT compatible string مطابق للـ driver
cat /sys/firmware/devicetree/base/soc/i2c@fe804000/compatible
# output: rockchip,rk3399-i2c

# مقارنة مع driver table
grep -r "rockchip,rk3399-i2c" /sys/bus/*/drivers/*/
```

---

### Practical Commands

#### مجموعة أوامر جاهزة

```bash
#!/bin/bash
# === fwnode Debug Script ===

DEVICE="${1:-}"  # مثال: fe300000.mmc

echo "=== 1. Device Links Status ==="
if [ -n "$DEVICE" ]; then
    find /sys/devices -name "*${DEVICE}*" -type d | head -3 | while read d; do
        echo "Device: $d"
        ls "$d/device_links/" 2>/dev/null | while read link; do
            echo "  Link: $link"
            cat "$d/device_links/$link/status" 2>/dev/null
            cat "$d/device_links/$link/flags" 2>/dev/null
        done
    done
fi

echo ""
echo "=== 2. Deferred Probes (waiting for fwnode suppliers) ==="
cat /sys/kernel/debug/devices/deferred 2>/dev/null || \
    dmesg | grep -i "deferred\|supplier"

echo ""
echo "=== 3. fwnode Flags Check ==="
# flags byte: bit0=LINKS_ADDED, bit1=NOT_DEVICE, bit2=INITIALIZED,
#             bit3=NEEDS_CHILD_BOUND, bit4=BEST_EFFORT, bit5=VISITED
dmesg | grep -i "fwnode\|fw_devlink" | tail -20

echo ""
echo "=== 4. DT Node Properties ==="
if [ -n "$DEVICE" ]; then
    find /sys/firmware/devicetree/base/ -name "*${DEVICE%.*}*" -type d | \
    head -1 | while read node; do
        echo "Node: $node"
        ls "$node/"
        echo "--- compatible ---"
        cat "$node/compatible" 2>/dev/null | tr '\0' '\n'
    done
fi

echo ""
echo "=== 5. ACPI Devices (if applicable) ==="
ls /sys/bus/acpi/devices/ 2>/dev/null | head -10
```

```bash
# فحص سريع لحالة الـ fw_devlink
echo "=== fw_devlink mode ==="
cat /sys/bus/platform/devices/.../firmware_node 2>/dev/null

# تفعيل verbose fw_devlink من command line
echo "fw_devlink.delay_sync_state=1" >> /proc/cmdline  # ليس runtime

# مشاهدة الـ probe order بالـ ftrace
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo "really_probe" > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... trigger probe ...
cat /sys/kernel/debug/tracing/trace | grep -A5 "really_probe"
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

```bash
# فحص شامل لـ fwnode graph endpoints
echo "=== fwnode Graph Endpoints ==="
find /sys/firmware/devicetree/base/ -path "*/port*/endpoint*" -type d | \
while read ep; do
    echo "Endpoint: $ep"
    if [ -f "$ep/remote-endpoint" ]; then
        echo "  Remote: $(cat $ep/remote-endpoint | xxd | head -1)"
    fi
done

# مثال output:
# Endpoint: /sys/firmware/devicetree/base/isi@58100000/port@0/endpoint@0
#   Remote: 00000000 00000045 00 ...  (phandle = 0x45)
```

```bash
# تفعيل dynamic debug لـ fwnode subsystem كامل
for f in \
    drivers/base/property.c \
    drivers/base/core.c \
    drivers/base/devlink.c \
    drivers/base/swnode.c \
    drivers/of/property.c \
    drivers/acpi/property.c; do
    echo "file $f +pflmt" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
done
dmesg -w | grep -E "fwnode|devlink|property"
```

```bash
# استخدام devmem2 لـ dump registers device مرتبط بـ fwnode
PHYS_BASE=$(grep -i "target-device-name" /proc/iomem | \
            awk '{print $1}' | cut -d'-' -f1)
if [ -n "$PHYS_BASE" ]; then
    for offset in 0x00 0x04 0x08 0x0c 0x10 0x14 0x18 0x1c; do
        ADDR=$((16#${PHYS_BASE} + 16#${offset#0x}))
        printf "REG[%s] = " "$offset"
        devmem2 $(printf "0x%x" $ADDR) w 2>/dev/null | grep "Value"
    done
fi
```

#### تفسير الـ Output المهم

```
# dmesg example مع fwnode deferred probe:
[    3.421] platform fe300000.mmc: Waiting for supplier of clk, id "clk_arb"

# التفسير:
# - fe300000.mmc = المـ consumer (عنده fwnode handle)
# - "clk_arb" = اسم الـ supplier fwnode (من DTS phandle)
# - الـ link: FWLINK_FLAG_CYCLE مش set → دي مش cycle، الـ supplier مش مـ probe

# الحل: تحقق أن الـ clock driver محمّل:
lsmod | grep clk
dmesg | grep "clk_arb\|clock-controller"
```

```
# ftrace function_graph output:
  0)               |  fwnode_call_int_op() {
  0)               |    of_fwnode_property_read_int_array() {
  0)   0.521 us    |      of_find_property();
  0)   1.203 us    |    } /* of_fwnode_property_read_int_array */
  0)   2.018 us    |  } /* fwnode_call_int_op */

# التفسير: الـ fwnode call أخدت ~2µs، والـ property وُجدت بنجاح
# لو return value كان -EINVAL → الـ ops مش set (fwnode_has_op فشل)
# لو return value كان -ENXIO → الـ ops موجودة بس الـ op نفسها NULL
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — Device لا يـprobe أبدًا

#### العنوان
**FWNODE_FLAG_INITIALIZED** مش بيتـset، وـdevice بيفضل في limbo

#### السياق
شغال على industrial gateway بيستخدم **Texas Instruments AM62x** SoC. الـgateway بيجمع بيانات من حساسات عبر **I2C** وبيبعتها على شبكة Ethernet. البورد اتعمل bring-up من أسبوعين، والـI2C controller driver بيـload بس الـclient devices (الحساسات) مش بتـprobe.

#### المشكلة
```bash
$ dmesg | grep i2c
[    2.341] i2c i2c-0: Added multiplexed i2c bus 1
[    2.342] i2c i2c-0: Added multiplexed i2c bus 2
# لا أي رسالة عن الـclient drivers
$ ls /sys/bus/i2c/devices/
i2c-0  i2c-1  i2c-2
# الـclient nodes موجودة في DT بس مش populated كـdevices
```

#### التحليل
الـkernel بيمشي على الـfwnode graph عشان يـcreate الـdevices. المشكلة في الـflag `FWNODE_FLAG_INITIALIZED`:

```c
// في fwnode.h
#define FWNODE_FLAG_INITIALIZED  BIT(2)

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

الـI2C multiplexer driver بيـcall `fwnode_dev_initialized()` بـ`false` بدل `true` بعد ما يـsetup الـchannels. ده بيخلي الـkernel يعتقد إن الـhardware ماتهيأش لسه، فبيـdefer تسجيل الـclient devices.

الـfwnode_handle الخاص بـmux node فيه:
```c
struct fwnode_handle {
    struct fwnode_handle *secondary; // NULL هنا
    const struct fwnode_operations *ops;
    struct device *dev;
    struct list_head suppliers;
    struct list_head consumers;
    u8 flags; // BIT(2) = 0 ← المشكلة هنا
};
```

#### الحل
```bash
# أول حاجة، تحقق من الـflags الحالية
$ cat /sys/kernel/debug/devices_deferred
# هتلاقي I2C client nodes هنا
```

في الـdriver code:
```c
// i2c-mux-driver.c — بعد ما تـsetup الـchannels
static int i2c_mux_probe(struct i2c_client *client)
{
    // ... setup code ...

    // الـsource of the bug — كان بيبعت false
    // fwnode_dev_initialized(dev_fwnode(&client->dev), false);

    // الصح:
    fwnode_dev_initialized(dev_fwnode(&client->dev), true);
    return 0;
}
```

#### الدرس المستفاد
`FWNODE_FLAG_INITIALIZED` مش مجرد flag تجميلية — الـkernel بيستخدمها عشان يعرف امتى يبدأ يـpopulate الـchild devices. أي driver بيـmanage child nodes لازم يـcall `fwnode_dev_initialized(fwnode, true)` صراحةً بعد ما يخلص الـhardware setup.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI مش بيشتغل بسبب graph endpoints

#### العنوان
**fwnode_endpoint** غلط بيكسر الـHDMI pipeline في بورد Allwinner H616

#### السياق
بتعمل Android TV box على **Allwinner H616**. الـSoC عنده display engine بيتكلم مع **HDMI transmitter** عبر TCON. بعد كل boot، الشاشة بتفضل سودة رغم إن الـkernel بيـdetect الـHDMI device.

#### المشكلة
```bash
$ dmesg | grep -i hdmi
[    3.12] sun50i-h616-hdmi: probe failed: -EPROBE_DEFER
[    3.13] sun50i-h616-hdmi: probe failed: -EPROBE_DEFER
[   10.50] sun50i-h616-hdmi: probe failed: -ENODEV
# بعد كذا retry، بيفشل نهائي
```

#### التحليل
الـHDMI driver بيستخدم `graph_get_next_endpoint` عشان يـwalk الـfwnode graph ويلاقي الـremote TCON endpoint. الـstruct اللي بيحملها:

```c
struct fwnode_endpoint {
    unsigned int port;   // رقم الـport في الـgraph
    unsigned int id;     // رقم الـendpoint
    const struct fwnode_handle *local_fwnode;
};
```

الـDevice Tree كانت فيه:
```dts
/* DT خاطئ */
&hdmi {
    ports {
        port@0 {
            endpoint@0 {  /* id = 0, correct */
                remote-endpoint = <&tcon_out>;
            };
        };
    };
};

&tcon {
    ports {
        port@1 {          /* port رقم 1 */
            endpoint@1 {  /* id = 1 — المشكلة! */
                remote-endpoint = <&hdmi_in>;
            };
        };
    };
};
```

الـdriver بيعمل `graph_parse_endpoint` على الـTCON node ويـexpect:
- `port = 1, id = 0`

بس الـDT بيديه `port = 1, id = 1`، فالـmatching بيفشل. الـkernel بيستخدم `SWNODE_GRAPH_ENDPOINT_NAME_FMT` = `"endpoint@%u"` عشان يـlookup، والـlookup بيرجع الـwrong node.

```c
// الـmacros في fwnode.h
#define SWNODE_GRAPH_PORT_NAME_FMT      "port@%u"
#define SWNODE_GRAPH_ENDPOINT_NAME_FMT  "endpoint@%u"
```

#### الحل
```dts
/* DT صح */
&tcon {
    ports {
        port@1 {
            endpoint@0 {  /* id لازم يبدأ من 0 */
                remote-endpoint = <&hdmi_in>;
            };
        };
    };
};
```

```bash
# تحقق بعد الإصلاح
$ cat /sys/kernel/debug/of_graph/hdmi/ports/port@0/endpoint@0/remote
# لازم يطلع: /soc/tcon@1/ports/port@1/endpoint@0
```

#### الدرس المستفاد
الـ`fwnode_endpoint.id` بيمثل الـindex داخل الـport، مش رقم اعتباطي. `endpoint@0` دايمًا هو أول endpoint في الـport. الـnaming convention مكتوبة صراحةً في `fwnode.h` والـmacros `SWNODE_GRAPH_*_NAME_FMT` بتأكدها.

---

### السيناريو الثالث: IoT Sensor على STM32MP1 — Probe defer loop بسبب fwnode_link cycle

#### العنوان
**FWLINK_FLAG_CYCLE** مش متـset وبيحصل infinite defer على STM32MP1

#### السياق
بتعمل IoT sensor node على **STM32MP157** SoC. الجهاز فيه **SPI** sensor متوصل بـ**GPIO expander** اللي نفسه بيتكلم على **I2C**. الـSPI driver بيـdepend على الـGPIO expander، والـGPIO expander driver بيحتاج SPI clock — circular dependency.

#### المشكلة
```bash
$ dmesg | tail -20
[  120.0] spi-stm32: probe deferred (waiting for gpio-expander)
[  120.1] gpio-expander: probe deferred (waiting for spi-stm32)
[  120.2] spi-stm32: probe deferred (waiting for gpio-expander)
# loop لا نهائي — الجهاز مبيـbootش أبدًا
```

#### التحليل
الـkernel بيبني `fwnode_link` بين الـnodes:

```c
struct fwnode_link {
    struct fwnode_handle *supplier;  // gpio-expander fwnode
    struct list_head s_hook;         // hook في suppliers list
    struct fwnode_handle *consumer;  // spi-stm32 fwnode
    struct list_head c_hook;         // hook في consumers list
    u8 flags;                        // المشكلة: FWLINK_FLAG_CYCLE مش مضبوط
};
```

الـflags الموجودة:
```c
#define FWLINK_FLAG_CYCLE   BIT(0)  // لو مضبوط، مش بيـdefer probe
#define FWLINK_FLAG_IGNORE  BIT(1)  // لو مضبوط، بيتجاهل الـlink كله
```

الـkernel cycle detection algorithm المفروض يـset `FWLINK_FLAG_CYCLE` لما يكتشف إن الـlink جزء من دائرة، بس الـDT كانت بتـexpress الـdependency بطريقة غير مباشرة عبر `clocks` property، والـcycle detection ماكنش بيمسك الـloop ده.

الـfwnode_handle للـSPI controller:
```c
// suppliers list فيها: gpio-expander
// consumers list فيها: spi-sensor
// flags: 0x00 — FWLINK_FLAG_CYCLE مش مضبوط
```

#### الحل
**Option 1** — تصحيح الـDT عشان تكسر الـcycle:
```dts
/* بدل ما الـGPIO expander يـdepend على SPI clock مباشرةً */
&gpio_expander {
    /* استخدم fixed-clock بدل SPI-derived clock */
    clocks = <&fixed_clk_26mhz>;
};
```

**Option 2** — لو الـcycle ضرورية، mark الـlink:
```bash
# عبر الـdebugfs
$ echo 1 > /sys/kernel/debug/fwnode_links/spi-stm32/gpio-expander/cycle_flag
```

**Option 3** — في الـdriver، استخدم `FWNODE_FLAG_BEST_EFFORT`:
```c
// في الـGPIO expander driver init
fwnode->flags |= FWNODE_FLAG_BEST_EFFORT;
// ده بيخلي الـdriver يـprobe حتى لو مش كل الـsuppliers جاهزة
```

#### الدرس المستفاد
الـ`fwnode_link` مع فلاجيه `FWLINK_FLAG_CYCLE` و`FWLINK_FLAG_IGNORE` هي الـmechanism اللي الـkernel بيتعامل بيها مع circular hardware dependencies. أي hardware topology فيها دائرة لازم تتعامل معاها صراحةً — إما بكسر الـcycle في الـDT أو بـset الـflag المناسبة.

---

### السيناريو الرابع: Automotive ECU على i.MX8QM — DMA مش شغال بسبب dev_dma_attr غلط

#### العنوان
**DEV_DMA_NON_COHERENT** بدل **DEV_DMA_COHERENT** بيكسر الـUSB DMA على i.MX8QM

#### السياق
بتعمل automotive ECU على **NXP i.MX8QM** SoC. الـECU محتاج **USB 3.0** عشان يـstream camera data بـhigh throughput. بعد bring-up الأولي، الـUSB بيشتغل بس بـsporatic data corruption وـthroughput منخفض جدًا.

#### المشكلة
```bash
$ dmesg | grep usb
[    4.23] usb 1-1: new SuperSpeed USB device
[    4.24] usb-storage 1-1:1.0: USB Mass Storage device detected
$ dd if=/dev/sda of=/dev/null bs=1M count=100
# throughput: 12 MB/s بدل 300+ MB/s المتوقعة
# وبعدين:
[  45.11] usb 1-1: Transfer error on endpoint 2
```

#### التحليل
الـfwnode API بيوفر `device_get_dma_attr` عشان الـdrivers تعرف الـDMA coherency model:

```c
enum dev_dma_attr {
    DEV_DMA_NOT_SUPPORTED,   // الجهاز مش بيدعم DMA أصلًا
    DEV_DMA_NON_COHERENT,    // لازم manual cache flush/invalidate
    DEV_DMA_COHERENT,        // hardware بيعمل cache coherency أوتوماتيك
};
```

الـops struct:
```c
struct fwnode_operations {
    // ...
    bool (*device_dma_supported)(const struct fwnode_handle *fwnode);
    enum dev_dma_attr (*device_get_dma_attr)(const struct fwnode_handle *fwnode);
    // ...
};
```

الـACPI table للـi.MX8QM كانت بتـreport الـUSB controller كـ`DEV_DMA_NON_COHERENT` بسبب typo في الـPMAT table. الـUSB driver بعدين بيعمل manual cache operations زيادة عن اللزوم، وده بيسبب corruption لأن الـhardware نفسه بيعمل coherency.

```c
// الـUSB driver بيعمل كده
enum dev_dma_attr attr = device_get_dma_attr(dev);
if (attr == DEV_DMA_NON_COHERENT) {
    // بيعمل cache flush قبل كل transfer — خطأ على i.MX8QM
    dma_sync_single_for_device(dev, dma_addr, size, DMA_TO_DEVICE);
}
```

#### الحل
```bash
# تحقق من الـDMA attributes الحالية
$ cat /sys/devices/platform/32f10110.usb/dma_coherent
# أو عبر أدوات ACPI
$ acpidump -n IORT | grep -A5 "USB"
```

إصلاح الـACPI IORT table لـset الـflag الصح:
```
# في IORT table — Named Component entry للـUSB
Cache coherency: 1  # كان 0 (non-coherent)
```

أو workaround في الـkernel command line:
```bash
# في bootloader
setenv bootargs "${bootargs} iommu.passthrough=1"
```

#### الدرس المستفاد
`dev_dma_attr` اللي بيتيجي من `fwnode_operations->device_get_dma_attr` بيأثر مباشرةً على كل الـDMA operations في الـsystem. على الـplatforms اللي بتستخدم ACPI (زي i.MX في بعض الـautomotive configurations)، الـIMAT/IORT table هي المصدر، وأي error فيها بيأثر على الـperformance والـcorrectness معًا.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — Software Node مش بيشتغل مع fwnode_reference_args

#### العنوان
**fwnode_reference_args** مع software nodes على RK3562 — الـargs مش بتتقرأ صح

#### السياق
بتعمل custom industrial board على **Rockchip RK3562**. البورد فيها **custom FPGA** متوصل على **SPI**، والـFPGA بتـcontrol بعض الـGPIOs. عشان ما تكتبش DT overlay معقد، قررت تستخدم **software nodes** عشان تـdescribe الـFPGA وعلاقتها بالـSPI.

#### المشكلة
```bash
$ dmesg | grep fpga
[    5.44] fpga-spi: failed to get gpio reference: -ENOENT
[    5.44] fpga-spi: probe failed: -ENOENT
```

الـdriver بيحاول يـread reference لـGPIO controller مع arguments (pin number + flags):

```c
struct fwnode_reference_args args;
ret = fwnode_property_get_reference_args(fwnode, "gpios",
                                          "#gpio-cells", 2, 0, &args);
// ret = -ENOENT دايمًا
```

#### التحليل
الـstruct المسؤولة:

```c
#define NR_FWNODE_REFERENCE_ARGS  16  // أقصى عدد args

struct fwnode_reference_args {
    struct fwnode_handle *fwnode;       // reference لـGPIO controller
    unsigned int nargs;                  // عدد الـargs (المفروض 2)
    u64 args[NR_FWNODE_REFERENCE_ARGS]; // [pin_number, flags]
};
```

الـfwnode_operations بتوفر:
```c
int (*get_reference_args)(const struct fwnode_handle *fwnode,
                          const char *prop,        // "gpios"
                          const char *nargs_prop,  // "#gpio-cells"
                          unsigned int nargs,       // 2
                          unsigned int index,       // 0
                          struct fwnode_reference_args *args);
```

المشكلة كانت في كيفية تعريف الـsoftware node:

```c
/* خاطئ — software node definition */
static const struct software_node_ref_args fpga_gpios[] = {
    SOFTWARE_NODE_REFERENCE(&gpio_controller_node, 5, 0),
};

static const struct property_entry fpga_props[] = {
    PROPERTY_ENTRY_REF_ARRAY("gpios", fpga_gpios),
    /* نسي يـdefine "#gpio-cells" على الـGPIO controller software node */
    { }
};
```

الـ`get_reference_args` implementation للـsoftware nodes بتدور على `#gpio-cells` property في الـGPIO controller node عشان تعرف كام argument تـparse. لو الـproperty دي مش موجودة، بترجع `-ENOENT`.

#### الحل
```c
/* صح — لازم تـdefine #gpio-cells على الـGPIO controller node */
static const struct property_entry gpio_ctrl_props[] = {
    PROPERTY_ENTRY_U32("#gpio-cells", 2),  // ← ده كان ناقص
    { }
};

static const struct software_node gpio_ctrl_node = {
    .name = "gpio-controller",
    .properties = gpio_ctrl_props,
};

/* والـFPGA node تـreference الـGPIO controller */
static const struct software_node_ref_args fpga_gpios[] = {
    SOFTWARE_NODE_REFERENCE(&gpio_ctrl_node, 5, 0), // pin 5, active high
};

static const struct property_entry fpga_props[] = {
    PROPERTY_ENTRY_REF_ARRAY("gpios", fpga_gpios),
    { }
};
```

```bash
# تحقق من الـsoftware nodes بعد الإصلاح
$ ls /sys/kernel/debug/software_nodes/
gpio-controller  fpga-spi

$ cat /sys/kernel/debug/software_nodes/gpio-controller/properties/#gpio-cells
2
```

#### الدرس المستفاد
الـ`fwnode_reference_args` mechanism بتعتمد على وجود الـ`#xxx-cells` property على الـsupplier node — حتى لو كنت بتستخدم software nodes بدل DT. ده mirror لنفس القاعدة في Device Tree: أي node بتعمل references ليها لازم تـdefine كام cell بتستخدم. الـ`NR_FWNODE_REFERENCE_ARGS = 16` هو الـhard limit، وأي reference محتاج أكتر من 16 argument هيفشل.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتشرح تطور الـ fwnode infrastructure في الـ kernel:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| Add ACPI _DSD and unified device properties support | [lwn.net/Articles/612062](https://lwn.net/Articles/612062/) | **الأساس** — بيشرح أول تقديم لفكرة الـ unified device property API اللي منها اتولد الـ fwnode |
| ACPI graph support | [lwn.net/Articles/718184](https://lwn.net/Articles/718184/) | بيشرح إزاي اتضافت الـ graph endpoints للـ fwnode عشان تشتغل مع ACPI و OF بنفس الـ interface |
| Device property: Introducing software nodes | [lwn.net/Articles/770825](https://lwn.net/Articles/770825/) | بيشرح الـ `software_node` — نوع fwnode مستقل عن DT و ACPI |
| Software fwnode references | [lwn.net/Articles/789099](https://lwn.net/Articles/789099/) | بيتكلم عن تحسينات الـ software node و references |
| Unified fwnode endpoint parser | [lwn.net/Articles/737780](https://lwn.net/Articles/737780/) | بيشرح توحيد الـ endpoint parser في V4L2 فوق الـ fwnode |
| Introduce fwnode in the I2C subsystem | [lwn.net/Articles/889236](https://lwn.net/Articles/889236/) | مثال عملي على إزاي الـ subsystems بتعتمد الـ fwnode |
| gpiolib: Two new helpers and way toward fwnode | [lwn.net/Articles/889987](https://lwn.net/Articles/889987/) | مثال تاني من gpiolib على migration للـ fwnode |

---

### توثيق الـ Kernel الرسمي

الملفات دي في `Documentation/` هي المرجع الرسمي:

```
Documentation/driver-api/infrastructure.rst
    → شرح device drivers infrastructure وفيه section للـ fwnode API

Documentation/firmware-guide/acpi/enumeration.rst
    → بيشرح إزاي ACPI بيستخدم الـ fwnode لتعداد الـ devices

Documentation/firmware-guide/acpi/dsd/graph.rst
    → الـ ACPI DSD graph ports/endpoints وعلاقتهم بالـ fwnode_endpoint

Documentation/firmware-guide/acpi/namespace.rst
    → بيشرح الـ ACPI namespace وتمثيله كـ fwnode tree
```

الروابط على docs.kernel.org:
- [Device drivers infrastructure](https://docs.kernel.org/driver-api/infrastructure.html)
- [ACPI Based Device Enumeration](https://docs.kernel.org/firmware-guide/acpi/enumeration.html)
- [Graphs — ACPI DSD](https://www.kernel.org/doc/html/v5.2/firmware-guide/acpi/dsd/graph.html)
- [V4L2 fwnode kAPI](https://docs.kernel.org/driver-api/media/v4l2-fwnode.html)

---

### ملفات الـ Source الأساسية في الـ Kernel

```
include/linux/fwnode.h          ← التعريفات الأساسية (الـ file ده نفسه)
include/linux/property.h        ← الـ public API فوق الـ fwnode
drivers/base/property.c         ← implementation الـ device property API
drivers/base/swnode.c           ← implementation الـ software_node fwnode
drivers/acpi/property.c         ← ACPI backend للـ fwnode operations
drivers/of/property.c           ← OF/DT backend للـ fwnode operations
```

---

### Commits المهمة في تاريخ الـ fwnode

| الوصف | المصدر |
|-------|--------|
| أول تقديم لـ `fwnode_handle` و device property API (2015 — Rafael J. Wysocki) | [github.com/torvalds/linux — fwnode.h](https://github.com/torvalds/linux/blob/master/include/linux/fwnode.h) |
| تاريخ التعديلات على الملف | [cregit — fwnode.h@4.14](https://cregit.linuxsources.org/code/4.14/include/linux/fwnode.h.html) |
| الـ patch series الخاصة بالـ secondary fwnode fallback | [linux.kernel.narkive](https://linux.kernel.narkive.com/XO6y722J/patch-v1-08-13-device-property-fallback-to-secondary-fwnode-if-primary-misses-the-property) |
| device property — implement accessors through fwnode | [linux.kernel.narkive](https://linux.kernel.narkive.com/pNOj9vdt/patch-driver-core-implement-device-property-accessors-through-fwnode-ones) |

---

### نقاشات Mailing List

- **[PATCH v9 00/12] Device property improvements, add %pfw format specifier** — نقاش مهم عن تحسينات الـ fwnode formatting وطباعة أسماء الـ nodes:
  [mail-archive.com](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2123755.html)

- **لو عايز تتابع النقاشات الجديدة**، ابحث في:
  - [lore.kernel.org/linux-acpi](https://lore.kernel.org/linux-acpi/) — نقاشات ACPI وfwnode
  - [lore.kernel.org/linux-devicetree](https://lore.kernel.org/linux-devicetree/) — نقاشات DT وfwnode

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model
  - بيشرح الـ `kobject`, `kset`, و device model اللي فوقيه الـ fwnode بيشتغل
  - متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules
  - بيغطي الـ device model الأساسي
  - **الفصل 13**: The Virtual Filesystem — مفيد لفهم abstraction layers زي الـ fwnode ops

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15**: Bootstrapping and Board Porting
  - بيشرح الـ Device Tree وإزاي الـ firmware بتصف الـ hardware — ده الأساس اللي الـ fwnode بيجرده

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 6**: Device Drivers
  - تغطية عميقة للـ device infrastructure في الـ kernel

---

### kernelnewbies.org

مفيش صفحة مخصصة لـ fwnode على kernelnewbies، لكن التغييرات المتعلقة بيه بتظهر في صفحات الـ kernel releases:

- [Linux 6.14 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.14) — فيه تغييرات متعلقة بـ V4L2 fwnode CSI-2 C-PHY
- [Linux 6.13 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.13)
- [KernelGlossary](https://kernelnewbies.org/KernelGlossary) — للمصطلحات العامة

---

### elinux.org

الـ elinux.org مفيهاش صفحة مخصصة لـ fwnode، لكن الـ Device Tree documentation المرتبطة بيه متاحة هنا:
- [Device Tree Usage — elinux.org](https://elinux.org/Device_Tree_Usage)
- [Device Tree Reference — elinux.org](https://elinux.org/Device_Tree_Reference)

---

### Search Terms للبحث عن معلومات أكثر

لو عايز تعمق أكتر، استخدم الـ keywords دي:

```
# في Google / DuckDuckGo
linux kernel fwnode_handle operations vtable
linux kernel device property unified API ACPI OF
linux kernel software_node swnode
linux kernel fwnode_link fw_devlink
linux kernel fwnode graph endpoint port

# في lore.kernel.org
fwnode_operations add_links
fw_devlink strict mode
fwnode secondary fallback
swnode software_node properties

# في git log داخل الـ kernel source
git log --oneline --follow include/linux/fwnode.h
git log --oneline --follow drivers/base/property.c
git log --oneline --follow drivers/base/swnode.c
```
## Phase 8: Writing simple module

### الفكرة

**`fwnode_link_add`** هي دالة exported موجودة في `drivers/base/core.c` بتتسمى كل ما الـ firmware node graph بيحاول يربط consumer بـ supplier (مثلاً لما الـ device tree بيحل الـ `phandle` dependencies). هنستخدم **kprobe** عشان نعترض الاستدعاء ونطبع معلومات عن الـ nodes اللي بتتربط.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * fwnode_link_add kprobe monitor
 *
 * Hooks fwnode_link_add() to trace firmware node link creation:
 * who is the consumer, who is the supplier, and what flags are set.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/fwnode.h>       /* struct fwnode_handle, fwnode_get_name */
#include <linux/property.h>     /* fwnode_get_name() wrapper */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on fwnode_link_add to trace fw node dependency links");

/* ------------------------------------------------------------------ */
/*  kprobe handler — called just BEFORE fwnode_link_add() executes     */
/* ------------------------------------------------------------------ */

/*
 * الـ pre_handler بيتشغل قبل تنفيذ الدالة الأصلية بمباشرة.
 * الـ regs بتحمل قيم الـ registers في لحظة الاستدعاء على الـ architecture.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86_64: الـ arguments بتتمرر في rdi, rsi, rdx بالترتيب.
     * fwnode_link_add(con, sup, flags)  →  rdi=con, rsi=sup, rdx=flags
     */
    struct fwnode_handle *con = (struct fwnode_handle *)regs->di;
    struct fwnode_handle *sup = (struct fwnode_handle *)regs->si;
    u8 flags = (u8)regs->dx;

    /* استخراج اسم الـ node من الـ ops->get_name لو موجودة */
    const char *con_name = "<unknown>";
    const char *sup_name = "<unknown>";

    if (con && con->ops && con->ops->get_name)
        con_name = con->ops->get_name(con);

    if (sup && sup->ops && sup->ops->get_name)
        sup_name = sup->ops->get_name(sup);

    pr_info("fwnode_link_add: consumer=[%s] supplier=[%s] flags=0x%x%s%s\n",
            con_name, sup_name, flags,
            (flags & BIT(0)) ? " CYCLE"  : "",   /* FWLINK_FLAG_CYCLE   */
            (flags & BIT(1)) ? " IGNORE" : "");  /* FWLINK_FLAG_IGNORE  */

    /* return 0 = لا تغيّر execution flow، استكمل الدالة الأصلية */
    return 0;
}

/* ------------------------------------------------------------------ */
/*  kprobe struct                                                       */
/* ------------------------------------------------------------------ */

/*
 * بنحدد اسم الـ symbol اللي هنحطّ فيه الـ probe.
 * الـ kernel بيحوّل الاسم لـ address وقت register.
 */
static struct kprobe kp = {
    .symbol_name = "fwnode_link_add",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  init / exit                                                         */
/* ------------------------------------------------------------------ */

static int __init fwnode_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بتحط breakpoint على الدالة المحددة.
     * لو فشل (مثلاً الـ symbol مش موجود أو مش kprobeable) بترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("fwnode_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("fwnode_probe: kprobe planted on fwnode_link_add @ %px\n",
            kp.addr);
    return 0;
}

static void __exit fwnode_probe_exit(void)
{
    /*
     * لازم نشيل الـ kprobe في الـ exit عشان منخليش breakpoint
     * في الـ kernel بعد ما الـ module اتفك — ده بيسبب kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("fwnode_probe: kprobe removed from fwnode_link_add\n");
}

module_init(fwnode_probe_init);
module_exit(fwnode_probe_exit);
```

---

### شرح كل قسم

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي module: `module_init`, `MODULE_LICENSE`, إلخ |
| `linux/kprobes.h` | تعريف `struct kprobe` وفنكشنز `register_kprobe` / `unregister_kprobe` |
| `linux/fwnode.h` | تعريف `struct fwnode_handle` و`struct fwnode_operations` اللي بنستخدمهم في الـ cast |
| `linux/property.h` | بيضم الـ wrappers المعيارية للـ fwnode API |

---

#### الـ `handler_pre` — الـ Callback

**الـ `pt_regs *regs`** بيحمل حالة الـ CPU registers لحظة اعتراض الدالة.
على x86_64 الـ calling convention بيحط أول 3 arguments في `rdi`, `rsi`, `rdx` — فبنعمل cast مباشر من الـ registers للـ structs المناسبة.

بنجيب اسم كل node عن طريق الـ `ops->get_name` pointer الموجود في `struct fwnode_operations` — لو الـ node مش عنده الـ op ده (مثلاً node مش مكتملة)، بنطبع `<unknown>`.

الـ flags بنحللها بنفس تعريفات `FWLINK_FLAG_CYCLE` و`FWLINK_FLAG_IGNORE` من `fwnode.h` عشان الـ output يكون readable.

---

#### الـ `struct kprobe kp`

بنحدد فقط `symbol_name` و`pre_handler`. الـ kernel بيحل الـ symbol لـ address تلقائياً وقت `register_kprobe`. لو احتجنا نشتغل بعد تنفيذ الدالة ممكن نضيف `post_handler` بنفس الأسلوب.

---

#### الـ `module_init` / `module_exit`

**الـ init** بيسجّل الـ kprobe — لو فشل بيرجع الـ error للـ kernel وما بيكملش تحميل الـ module. **الـ exit** إلزامي يشيل الـ kprobe؛ تركه مزروع بعد تفريغ الـ module بيخلي الـ kernel يقفز لكود اتشال من الذاكرة ويعمل **kernel panic**.

---

### Makefile للتجميع

```makefile
obj-m += fwnode_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### تشغيل واختبار

```bash
# تحميل الـ module
sudo insmod fwnode_probe.ko

# تشغيل أي عملية probe (مثلاً تحميل driver يستخدم device tree)
sudo modprobe some_dt_driver

# مشاهدة الـ output
sudo dmesg | grep fwnode_probe

# تفريغ الـ module
sudo rmmod fwnode_probe
```

**مثال على الـ output المتوقع:**

```
[  42.123456] fwnode_probe: kprobe planted on fwnode_link_add @ ffffffffc0a1b234
[  42.789012] fwnode_probe: fwnode_link_add: consumer=[/soc/i2c@ff160000/sensor@48] supplier=[/clocks/clk_apb] flags=0x0
[  43.001234] fwnode_probe: fwnode_link_add: consumer=[/soc/spi@ff180000] supplier=[/soc/pinctrl] flags=0x0
```

كل سطر بيكشف dependency جديدة الـ kernel بيبنيها في graph الـ firmware nodes — ده مفيد جداً في debug مشاكل الـ `deferred probe` وترتيب تحميل الـ drivers.
