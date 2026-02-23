## Phase 1: الصورة الكبيرة ببساطة

### المشكلة اللي بتحلها

تخيل إنك بتشتري جهاز راوتر. جوا الراوتر في chips كتير: chip للـ WiFi، chip للـ Ethernet، chip للـ USB. كل chip دي محتاجة driver في الـ Linux kernel يشغّلها. الـ driver محتاج يعرف معلومات عن الـ hardware: "انت بتشتغل على IRQ رقم كام؟ العنوان في الـ memory إيه؟ هل بتدعم big-endian؟"

**السؤال هو: فين الـ driver هيلاقي المعلومات دي؟**

الإجابة مش واحدة — بتختلف حسب نوع الـ platform:

| Platform | طريقة وصف الـ Hardware |
|----------|------------------------|
| ARM / Embedded | **Device Tree (DT)** — ملف `.dts` بيوصف الـ hardware |
| x86 / Laptop / Server | **ACPI** — جداول بيكتبها الـ BIOS/UEFI |
| Virtual / Test | **Software Nodes** — properties بتتكتب في الـ C code مباشرة |

المشكلة الكبيرة: كل طريقة ليها API مختلفة تماماً. الـ driver اللي بيشتغل على ARM بيستخدم `of_property_read_u32()`، ونفس الـ driver لو شغّال على x86 هيستخدم `acpi_dev_get_property()`. ده معناه إن نفس الـ driver محتاج يتكتب مرتين — كارثة.

---

### الحل: Unified Device Property Interface

**الـ `property.h`** هو رأس ملف الـ **Unified Device Property Interface** — طبقة abstraction وحيدة فوق كل طرق وصف الـ hardware.

الفكرة بسيطة جداً: بدل ما كل driver يعرف إيه الـ firmware اللي شغال عليه (DT ولا ACPI ولا غيره)، يكلم API واحدة بس، وهي هي اللي تروح تسأل الـ firmware المناسب.

```
Driver Code
    │
    ▼
┌─────────────────────────────────────────┐
│  property.h  (Unified API)              │
│  device_property_read_u32(dev, "speed") │
└──────────────┬──────────────────────────┘
               │  fwnode_handle (abstract pointer)
       ┌───────┼───────────────┐
       ▼       ▼               ▼
   OF/DT     ACPI        Software Node
  (ARM)     (x86)        (pure code)
```

---

### الـ fwnode_handle: البطل الأساسي

**الـ `struct fwnode_handle`** هو الـ abstraction الأساسية في الموضوع ده كله. هو مجرد pointer صغير فيه:
- `ops` — جدول function pointers (زي vtable في C++)
- `secondary` — fallback لـ fwnode تاني
- flags وlinks للـ device dependencies

كل firmware type (DT، ACPI، swnode) بيعمل implementation خاصة بيه للـ `fwnode_operations`، وبكده الـ `property.h` API بتشتغل على الكل بنفس الكود.

---

### قصة حقيقية: driver لـ I2C sensor

تخيل إنك بتكتب driver لـ temperature sensor بيتوصل بـ I2C. الـ sensor ده بيتستخدم في:

- Raspberry Pi (ARM + Device Tree)
- Intel NUC (x86 + ACPI)
- QEMU virtual machine (Software Node في الـ test code)

**بدون `property.h`**، كنت هتكتب كده:

```c
#ifdef CONFIG_OF
    of_property_read_u32(np, "polling-interval-ms", &interval);
#elif defined(CONFIG_ACPI)
    acpi_dev_get_property(adev, "polling-interval-ms", ACPI_TYPE_INTEGER, &val);
#endif
```

**بـ `property.h`**، بتكتب مرة واحدة بس:

```c
/* works on DT, ACPI, and software nodes — same code */
device_property_read_u32(dev, "polling-interval-ms", &interval);
```

---

### مكونات الـ API في property.h

#### 1. قراءة الـ Properties
دوال لقراءة أنواع مختلفة من الـ properties — integers، strings، booleans، arrays:

```c
/* قراءة integer واحد */
device_property_read_u32(dev, "clock-frequency", &freq);

/* قراءة array */
device_property_read_u32_array(dev, "reg", regs, count);

/* تحقق من وجود property */
device_property_present(dev, "enable-gpios");

/* قراءة boolean property */
device_property_read_bool(dev, "wakeup-source");
```

#### 2. التنقل في شجرة الـ nodes

الـ hardware ممكن يكون له structure هرمي — chip رئيسية وجوّاها sub-nodes (ports، endpoints، channels). الـ `property.h` بيوفر macros للتنقل:

```c
/* iterate على كل child nodes */
device_for_each_child_node(dev, child) {
    fwnode_property_read_u32(child, "reg", &reg);
}

/* scoped version — بيعمل put تلقائي */
device_for_each_child_node_scoped(dev, child) {
    /* child بيتحرر تلقائياً عند الخروج من الـ scope */
}
```

#### 3. الـ Graph API (للـ Media و Networking)

لما يكون في connections بين devices — زي camera متوصلة بـ ISP متوصل بـ display — الـ fwnode graph API بيعبّر عن العلاقات دي:

```c
/* امشي على كل endpoints */
fwnode_graph_for_each_endpoint(fwnode, ep) {
    remote = fwnode_graph_get_remote_endpoint(ep);
}
```

#### 4. الـ Software Nodes

لما الـ hardware مش موصوف في DT أو ACPI (زي في الـ testing أو الـ x86 devices اللي مش ليها ACPI entries)، الـ `software_node` بيخليك تعرّف الـ properties مباشرة في الـ C code:

```c
static const struct property_entry my_dev_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("status", "okay"),
    PROPERTY_ENTRY_BOOL("wakeup-source"),
    { }  /* sentinel */
};

static const struct software_node my_swnode = {
    .name       = "my-sensor",
    .properties = my_dev_props,
};

/* attach to device */
device_add_software_node(dev, &my_swnode);
```

---

### الـ Subsystem في MAINTAINERS

الملف ده جزء من subsystem اسمه **SOFTWARE NODES AND DEVICE PROPERTIES** وموجود في قسم `drivers/base/`.

---

### الملفات اللي بتكوّن الـ Subsystem

| الملف | الدور |
|-------|-------|
| `include/linux/property.h` | الـ public API header — ده ملفنا |
| `include/linux/fwnode.h` | تعريف `fwnode_handle` و`fwnode_operations` |
| `drivers/base/property.c` | التنفيذ الفعلي لكل دوال الـ `device_property_*` و`fwnode_property_*` |
| `drivers/base/swnode.c` | تنفيذ الـ software nodes — الـ fwnode_operations للـ swnode backend |

### ملفات backend بتنفّذ نفس الـ interface

| الملف | الدور |
|-------|-------|
| `drivers/of/property.c` | Device Tree backend |
| `drivers/acpi/property.c` | ACPI backend |
| `drivers/acpi/acpi_lpss.c` | مثال على إضافة software nodes لـ x86 devices |

### ملفات مرتبطة بالـ use cases

| الملف | الدور |
|-------|-------|
| `include/media/v4l2-fwnode.h` | استخدام الـ graph API في الـ camera subsystem |
| `drivers/media/v4l2-core/v4l2-fwnode.c` | تنفيذه |
| `drivers/net/mdio/fwnode_mdio.c` | استخدام الـ property API في الـ network MDIO |

---

### الخلاصة

**الـ `property.h`** هو "محوّل كهربائي" بين الـ drivers والـ firmware. الـ driver مش محتاج يعرف إيه الـ firmware اللي شغّال عليه — بيسأل `device_property_read_*()` وهي هي تعرف تروح فين. ده بيخلي الـ drivers أبسط، أقل `#ifdef`، وتشتغل على كل الـ platforms من نفس الكود.
## Phase 2: شرح الـ Unified Device Property Framework

### المشكلة: ليه الـ Framework ده موجود أصلاً؟

في الـ embedded Linux، الـ driver بيحتاج يعرف إيه الـ configuration بتاعت الـ hardware اللي شغال عليه — مثلاً:
- الـ I2C sensor ده بيشتغل على أنهي address؟
- الـ GPIO reset line بتاعت الـ controller ده رقم كام؟
- الـ clock frequency المطلوبة إيه؟
- فيه `big-endian` registers ولا لأ؟

المشكلة إن في الـ Linux kernel في **3 طرق مختلفة** للـ hardware description:

| الطريقة | المصدر | المنصة |
|---------|--------|--------|
| **Device Tree (DT)** | `.dts` file compiled to DTB | ARM, RISC-V, embedded |
| **ACPI** | BIOS/firmware tables | x86, ARM servers |
| **Software Nodes** | Static C structs in driver code | Platforms with no firmware |

لو كتبت driver بيقرأ من الـ DT مباشرةً باستخدام `of_property_read_u32()` — الـ driver ده مش هيشتغل على ACPI system والعكس بالعكس.

**الـ fragmentation كان فادح:** نفس الـ driver كان لازم يتكتب مرتين، أو يبقى مليان `#ifdef CONFIG_OF` و`#ifdef CONFIG_ACPI`.

---

### الحل: Unified Device Property Interface

الـ kernel حل المشكلة بـ **abstraction layer** اسمه **Unified Device Property API** — موجود في `include/linux/property.h` و`drivers/base/property.c`.

الفكرة الجوهرية: كل مصدر للـ firmware (DT، ACPI، Software) بيـ"wrap" نفسه في object واحد اسمه **`fwnode_handle`** — وكل `fwnode_handle` بيحمل معاه pointer لـ vtable من النوع **`fwnode_operations`** فيه implementation خاصة بيه.

الـ driver مش بيكلم الـ DT أو الـ ACPI directly — بيكلم الـ `fwnode_handle` فقط، والـ kernel هو اللي يروح يجيب القيمة من أي backend موجود.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Driver Code                             │
│                                                                 │
│  device_property_read_u32(dev, "clock-frequency", &val)        │
│  fwnode_property_read_string(fwnode, "compatible", &str)       │
└──────────────────────────┬──────────────────────────────────────┘
                           │  Unified API  (property.h)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    fwnode_handle  (fwnode.h)                    │
│                                                                 │
│   struct fwnode_handle {                                        │
│       const struct fwnode_operations  *ops;  ◄── vtable        │
│       struct fwnode_handle            *secondary;              │
│       struct device                   *dev;                    │
│       struct list_head                 suppliers;              │
│       struct list_head                 consumers;              │
│       u8                               flags;                  │
│   }                                                             │
└────────────┬───────────────────────┬───────────────────────────┘
             │                       │
     ┌───────▼──────┐     ┌──────────▼──────────┐     ┌──────────────────┐
     │  OF (DT)     │     │    ACPI              │     │  Software Node   │
     │  Backend     │     │    Backend           │     │  Backend         │
     │              │     │                      │     │                  │
     │ of_fwnode_ops│     │ acpi_fwnode_ops      │     │ swnode_ops       │
     │              │     │                      │     │                  │
     │ reads from   │     │ reads from           │     │ reads from       │
     │ DTB in RAM   │     │ ACPI tables          │     │ property_entry[] │
     └──────────────┘     └──────────────────────┘     └──────────────────┘
```

---

### التشبيه الواقعي: نظام الخرائط في الهاتف

تخيل إن أنت بتكتب app على الموبايل محتاج يعرض خريطة. عندك خيارات:
- Google Maps API
- Apple Maps API
- OpenStreetMap API

لو كتبت الـ app بتاعك يكلم Google Maps directly — مش هتشتغل على iOS.

الحل: بتستخدم **Map Abstraction Layer** — interface واحد اسمه `MapProvider` — وكل provider بيعمل `implement` لنفس الـ interface.

دلوقتي:

| الـ Map Abstraction | الـ Kernel Equivalent |
|--------------------|----------------------|
| `MapProvider` interface | `struct fwnode_handle` + `fwnode_operations` |
| `GoogleMapsProvider` | `of_fwnode_ops` (Device Tree backend) |
| `AppleMapsProvider` | `acpi_fwnode_ops` (ACPI backend) |
| `OSMProvider` | `swnode_ops` (Software Node backend) |
| `getLocation(name)` | `fwnode_property_read_u32(fwnode, name, &val)` |
| الـ App code | الـ Driver code |
| الخريطة الـ underlying | الـ DTB / ACPI Tables / property_entry[] |

**التطابق الكامل:**
- زي ما الـ app مش بيعرف إيه الـ map provider الـ underlying — الـ driver مش بيعرف إيه الـ firmware source.
- زي ما الـ `MapProvider` بيعمل reference counting (لو الـ map object اتحذف) — الـ `fwnode_handle` عنده `get/put` في الـ `fwnode_operations`.
- زي ما الـ Map app ممكن يـ"chain" بين providers (fallback) — الـ `fwnode_handle.secondary` بيخلي الـ kernel يـ"chain" بين DT و Software Node لنفس الـ device.

---

### الـ Core Abstraction: `fwnode_handle` والـ vtable

ده الـ heart of the subsystem. كل firmware backend بيعمل `embed` لـ `fwnode_handle` جوا struct أكبر:

```c
/* مثال: DT backend (of/base.c) */
struct device_node {
    const char *name;
    /* ... */
    struct fwnode_handle fwnode;  /* embedded — هنا بيتم الـ embedding */
};

/* مثال: Software Node (swnode.c) */
struct swnode {
    struct kobject kobj;
    struct fwnode_handle fwnode;  /* embedded */
    const struct software_node *node;
    /* ... */
};
```

ده نفس pattern الـ **container_of** — لما عندك `fwnode_handle *` تقدر توصل للـ outer struct باستخدام `to_of_node()` أو `to_software_node()`.

```
┌────────────────────────────────────────┐
│         struct device_node             │
│  ┌──────────────────────────────────┐  │
│  │     struct fwnode_handle         │  │
│  │   ops  ──► of_fwnode_ops         │  │
│  │   dev  ──► struct device         │  │
│  │   secondary ──► NULL / swnode    │  │
│  └──────────────────────────────────┘  │
│  name: "i2c@40005400"                  │
│  properties: { clock-frequency: ... } │
└────────────────────────────────────────┘
           ▲
           │  container_of
           │
     to_of_node(fwnode)
```

---

### الـ vtable بالتفصيل: `struct fwnode_operations`

```c
struct fwnode_operations {
    /* Reference counting */
    struct fwnode_handle *(*get)(struct fwnode_handle *fwnode);
    void (*put)(struct fwnode_handle *fwnode);

    /* Device availability (مثلاً: status = "okay" في DT) */
    bool (*device_is_available)(const struct fwnode_handle *fwnode);

    /* Driver match data (لـ platform_device_id / of_device_id) */
    const void *(*device_get_match_data)(const struct fwnode_handle *fwnode,
                                          const struct device *dev);

    /* DMA support */
    bool (*device_dma_supported)(const struct fwnode_handle *fwnode);

    /* Property reading */
    bool (*property_present)(const struct fwnode_handle *, const char *propname);
    bool (*property_read_bool)(const struct fwnode_handle *, const char *propname);
    int  (*property_read_int_array)(const struct fwnode_handle *,
                                    const char *propname,
                                    unsigned int elem_size,   /* 1, 2, 4, or 8 bytes */
                                    void *val, size_t nval);
    int  (*property_read_string_array)(const struct fwnode_handle *,
                                       const char *propname,
                                       const char **val, size_t nval);

    /* Tree navigation */
    const char *(*get_name)(const struct fwnode_handle *fwnode);
    struct fwnode_handle *(*get_parent)(const struct fwnode_handle *fwnode);
    struct fwnode_handle *(*get_next_child_node)(const struct fwnode_handle *,
                                                  struct fwnode_handle *child);
    struct fwnode_handle *(*get_named_child_node)(const struct fwnode_handle *,
                                                   const char *name);

    /* Reference properties (phandles في DT) */
    int (*get_reference_args)(const struct fwnode_handle *fwnode,
                               const char *prop, const char *nargs_prop,
                               unsigned int nargs, unsigned int index,
                               struct fwnode_reference_args *args);

    /* Graph API (camera pipelines, display links) */
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);

    /* Hardware resources */
    void __iomem *(*iomap)(struct fwnode_handle *fwnode, int index);
    int (*irq_get)(const struct fwnode_handle *fwnode, unsigned int index);

    /* Device links / probe ordering */
    int (*add_links)(struct fwnode_handle *fwnode);
};
```

**ملاحظة مهمة:** `property_read_int_array` بتاخد `elem_size` parameter — ده بيخليها تشتغل كـ u8، u16، u32، u64 بدون تكرار الكود. الـ `device_property_read_u32_array()` في property.h بتستدعيها بـ `elem_size = sizeof(u32)`.

---

### الـ Macro Dispatch: `fwnode_call_int_op`

```c
#define fwnode_call_int_op(fwnode, op, ...)                         \
    (fwnode_has_op(fwnode, op) ?                                    \
     (fwnode)->ops->op(fwnode, ## __VA_ARGS__) :                    \
     (IS_ERR_OR_NULL(fwnode) ? -EINVAL : -ENXIO))
```

ده الـ dispatcher اللي بيحول كل الـ API calls للـ vtable:
- لو الـ `fwnode` = NULL → `-EINVAL`
- لو الـ backend مش بيدعم الـ operation → `-ENXIO`
- لو كل حاجة تمام → بيستدعي الـ function pointer مباشرةً

---

### طبقات الـ API: `device_*` vs `fwnode_*`

الـ API عنده طبقتين:

```
device_property_read_u32(dev, "reg", &val)
         │
         ▼
    dev_fwnode(dev)          ← يجيب الـ fwnode من الـ device
         │
         ▼
fwnode_property_read_u32(fwnode, "reg", &val)
         │
         ▼
fwnode_call_int_op(fwnode, property_read_int_array, "reg", 4, &val, 1)
         │
         ▼
fwnode->ops->property_read_int_array(fwnode, "reg", 4, &val, 1)
         │
         ▼
    [DT implementation OR ACPI implementation OR swnode implementation]
```

الـ `dev_fwnode(dev)` macro بتستخدم `_Generic` لـ type safety:

```c
#define dev_fwnode(dev)                                 \
    _Generic((dev),                                     \
             const struct device *: __dev_fwnode_const, \
             struct device *:       __dev_fwnode)(dev)
```

لو الـ `dev` هو `const struct device *` → بترجع `const struct fwnode_handle *`.
لو الـ `dev` هو `struct device *` → بترجع `struct fwnode_handle *`.
ده بيمنع الـ driver من accidental modification للـ fwnode لما الـ device const.

---

### الـ Property Types: `enum dev_prop_type`

```c
enum dev_prop_type {
    DEV_PROP_U8,
    DEV_PROP_U16,
    DEV_PROP_U32,
    DEV_PROP_U64,
    DEV_PROP_STRING,
    DEV_PROP_REF,     /* reference لـ node تاني */
};
```

الـ `DEV_PROP_REF` خاصة جداً — بتمثل الـ **phandle** في DT أو الـ equivalent في ACPI. مثلاً في DT:

```dts
clocks = <&rcc 0>;  /* reference لـ clock controller + argument */
```

ده بيتقرأ بـ `fwnode_property_get_reference_args()` وبيرجع `struct fwnode_reference_args`:

```c
struct fwnode_reference_args {
    struct fwnode_handle *fwnode;        /* الـ node المشار إليه */
    unsigned int          nargs;         /* عدد الـ arguments */
    u64                   args[16];      /* الـ arguments نفسها */
};
```

---

### `struct property_entry`: الـ Software Node Properties

ده الـ building block بتاع الـ Software Nodes — بيخلي الـ driver أو platform code يعرّف properties في C code مباشرةً بدون أي firmware:

```c
struct property_entry {
    const char         *name;       /* اسم الـ property */
    size_t              length;     /* حجم الـ data بالـ bytes */
    bool                is_inline;  /* هل القيمة stored inline في الـ struct؟ */
    enum dev_prop_type  type;
    union {
        const void *pointer;        /* لو الـ data كبيرة → pointer للـ array */
        union {
            u8         u8_data[8];
            u16        u16_data[4];
            u32        u32_data[2];
            u64        u64_data[1];
            const char *str[1];     /* sizeof(u64)/sizeof(char*) = 1 on 64-bit */
        } value;                    /* لو الـ data صغيرة → stored inline */
    };
};
```

**الـ inline optimization:** القيم الصغيرة (u8 ، u16، u32، u64 واحدة، string واحدة) بتتخزن مباشرةً جوا الـ `struct property_entry` نفسه بدون heap allocation. الـ `is_inline = true` بيعلّم ده.

**مثال عملي:**

```c
static const struct property_entry my_sensor_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),   /* inline */
    PROPERTY_ENTRY_BOOL("wakeup-source"),             /* bool: is_inline=true, length=0 */
    PROPERTY_ENTRY_STRING("label", "front-sensor"),  /* inline string */
    PROPERTY_ENTRY_U32_ARRAY("reg-names-ids",
                              (u32[]){0x10, 0x20, 0x30}, 3), /* pointer */
    { }  /* sentinel — NULL name = end of array */
};

static const struct software_node my_sensor_node = {
    .name       = "my-sensor",
    .properties = my_sensor_props,
};
```

ثم في الـ driver أو board file:

```c
device_add_software_node(dev, &my_sensor_node);
```

---

### الـ Software Node Tree: `struct software_node`

```c
struct software_node {
    const char                  *name;
    const struct software_node  *parent;    /* لبناء tree hierarchy */
    const struct property_entry *properties;
};
```

```
software_node: "i2c-bus"
   ├── software_node: "sensor@48"
   │       properties: [address=0x48, ...]
   └── software_node: "eeprom@50"
           properties: [address=0x50, ...]
```

ده بيحاكي نفس الـ tree structure بتاع الـ DT أو ACPI — الـ driver بيـ traverse الـ tree بنفس الـ API.

---

### الـ Graph API: ليه موجود؟

الـ `fwnode_graph_*` functions بتحل مشكلة specific: **media pipelines** و **display pipelines** في embedded systems.

مثلاً board عندها:

```
Camera Sensor ──► MIPI CSI-2 Receiver ──► ISP ──► DMA ──► Memory
```

كل "سلك" بين component وتاني بيتمثل بـ **endpoint** في DT:

```dts
sensor {
    port {
        sensor_out: endpoint {
            remote-endpoint = <&csi_in>;
        };
    };
};

csi2_rx {
    ports {
        port@0 {                    /* input port */
            csi_in: endpoint@0 {
                remote-endpoint = <&sensor_out>;
            };
        };
        port@1 {                    /* output port */
            csi_out: endpoint@0 {
                remote-endpoint = <&isp_in>;
            };
        };
    };
};
```

الـ Graph API بيخليك تـ traverse الـ pipeline ده:

```c
/* جيب كل endpoints بتاعت الـ device */
fwnode_graph_for_each_endpoint(fwnode, ep) {
    /* جيب الـ endpoint على الطرف التاني */
    remote = fwnode_graph_get_remote_endpoint(ep);
    /* جيب الـ device اللي الـ remote endpoint تبعه */
    remote_parent = fwnode_graph_get_remote_port_parent(ep);
}
```

---

### الـ `secondary` Field: الـ DT + Software Node Combo

الـ `fwnode_handle.secondary` بيخلي الـ kernel يـ"chain" بين firmware sources لنفس الـ device:

```
struct device
    └── fwnode ──► DT fwnode_handle
                        secondary ──► swnode fwnode_handle
                                          secondary ──► NULL
```

لو property مش موجودة في الـ DT — الـ kernel بيـ fallback للـ `secondary` تلقائياً. ده بيخلي platform code يـ"override" أو يـ"augment" الـ DT properties بـ software nodes بدون تعديل الـ DTB.

**Use case حقيقي:** ACPI-based tablet عنده camera sensor الـ ACPI table بتاعه ناقصة بعض الـ properties المطلوبة للـ driver. الـ kernel بيضيف software node بالـ missing properties وبيـ chain له على الـ ACPI fwnode.

---

### الـ Scoped Iteration: `__free(fwnode_handle)`

```c
#define fwnode_for_each_child_node_scoped(fwnode, child)        \
    for (struct fwnode_handle *child __free(fwnode_handle) =    \
            fwnode_get_next_child_node(fwnode, NULL);           \
         child; child = fwnode_get_next_child_node(fwnode, child))
```

ده بيستخدم الـ **cleanup attribute** (مضاف في kernel 6.x) اللي بيعمل `fwnode_handle_put()` تلقائياً لما الـ scope بينتهي — حتى لو فيه `break` أو `return` جوا الـ loop. من غيره، الـ developer ممكن ينسى الـ put ويحصل reference leak.

---

### الـ Connection API: `fwnode_connection_find_match`

```c
typedef void *(*devcon_match_fn_t)(const struct fwnode_handle *fwnode,
                                    const char *id, void *data);

void *fwnode_connection_find_match(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match);
```

ده بيخلي subsystem (مثلاً GPIO أو regulator) يـ"match" connections بين devices بناءً على property names. مثلاً الـ regulator framework بيستخدمه لإيجاد "vdd-supply" connection.

---

### إيه اللي الـ Subsystem ده بيملكه vs إيه اللي بيفوضه للـ Backends

| ما يملكه الـ Subsystem | ما يفوضه للـ Backends |
|------------------------|----------------------|
| الـ unified API surface (`device_property_*`, `fwnode_property_*`) | تفسير الـ DTB أو ACPI tables |
| الـ `fwnode_handle` struct وإدارة الـ reference counting | بناء الـ tree hierarchy من الـ firmware |
| الـ `fwnode_call_*_op` dispatch macros | تحديد إيه معنى "device is available" |
| الـ `secondary` chaining logic | قراءة الـ IRQ numbers من الـ firmware |
| الـ Software Node registration/lookup | iomap من firmware resources |
| الـ Graph traversal API (port/endpoint model) | add_links للـ probe ordering |
| الـ inline optimization في `property_entry` | |
| الـ scoped cleanup للـ fwnode references | |

---

### الـ `dev_dma_attr` و DMA Support

الـ `fwnode_operations` بتعرض:

```c
bool (*device_dma_supported)(const struct fwnode_handle *fwnode);
enum dev_dma_attr (*device_get_dma_attr)(const struct fwnode_handle *fwnode);
```

الـ `enum dev_dma_attr`:

```c
enum dev_dma_attr {
    DEV_DMA_NOT_SUPPORTED,   /* الـ device مش بيدعم DMA */
    DEV_DMA_NON_COHERENT,    /* بيدعم DMA بس محتاج explicit cache ops */
    DEV_DMA_COHERENT,        /* بيدعم coherent DMA (hardware does cache sync) */
};
```

في DT، ده بيتحدد من `dma-coherent` property. الـ DMA subsystem بيسأل عنه عشان يعرف يستخدم `dma_alloc_coherent()` ولا `dma_alloc_noncoherent()`.

---

### مثال كامل: Driver بيستخدم الـ API

```c
static int my_sensor_probe(struct i2c_client *client)
{
    struct device *dev = &client->dev;
    u32 clock_freq;
    const char *label;
    struct fwnode_handle *ep;
    int ret;

    /* قراءة scalar property */
    ret = device_property_read_u32(dev, "clock-frequency", &clock_freq);
    if (ret)
        return dev_err_probe(dev, ret, "missing clock-frequency\n");

    /* قراءة string property */
    ret = device_property_read_string(dev, "label", &label);
    if (ret)
        label = "unknown";

    /* check boolean property */
    if (device_property_read_bool(dev, "wakeup-source"))
        device_init_wakeup(dev, true);

    /* check compatible */
    if (device_is_compatible(dev, "myvendor,sensor-v2")) {
        /* v2-specific init */
    }

    /* traverse graph endpoints */
    fwnode_graph_for_each_endpoint(dev_fwnode(dev), ep) {
        struct fwnode_handle *remote;
        remote = fwnode_graph_get_remote_port_parent(ep);
        /* process remote device */
        fwnode_handle_put(remote);
    }

    return 0;
}
```

نفس الـ driver ده هيشتغل على DT board، ACPI laptop، وحتى platform بدون firmware لو عملنا `device_add_software_node()` للـ device.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### `enum dev_prop_type` — نوع الـ property

| القيمة | المعنى |
|---|---|
| `DEV_PROP_U8` | بايت واحد unsigned |
| `DEV_PROP_U16` | 16-bit unsigned |
| `DEV_PROP_U32` | 32-bit unsigned (الأكثر استخداماً) |
| `DEV_PROP_U64` | 64-bit unsigned |
| `DEV_PROP_STRING` | نص `const char *` |
| `DEV_PROP_REF` | مرجع لـ `software_node_ref_args` |

#### `enum dev_dma_attr` — قدرات الـ DMA

| القيمة | المعنى |
|---|---|
| `DEV_DMA_NOT_SUPPORTED` | الجهاز مش بيعمل DMA خالص |
| `DEV_DMA_NON_COHERENT` | DMA بس محتاج cache sync يدوي |
| `DEV_DMA_COHERENT` | DMA coherent — hardware بيعمل sync لوحده |

#### `fwnode_handle` flags

| الـ Flag | الـ Bit | المعنى |
|---|---|---|
| `FWNODE_FLAG_LINKS_ADDED` | `BIT(0)` | الـ fwnode اتحلل وأُضيفت links بتاعته |
| `FWNODE_FLAG_NOT_DEVICE` | `BIT(1)` | مش هيتحول لـ `struct device` أبداً |
| `FWNODE_FLAG_INITIALIZED` | `BIT(2)` | الـ hardware المقابل له اتهيّأ |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | `BIT(3)` | محتاج الـ children يعملوا bind الأول |
| `FWNODE_FLAG_BEST_EFFORT` | `BIT(4)` | يـprobe بدري حتى لو في suppliers ناقصة |
| `FWNODE_FLAG_VISITED` | `BIT(5)` | اتزار أثناء traversal (cycle detection) |

#### `fwnode_link` flags

| الـ Flag | الـ Bit | المعنى |
|---|---|---|
| `FWLINK_FLAG_CYCLE` | `BIT(0)` | الـ link جزء من cycle — متـdefer الـ probe |
| `FWLINK_FLAG_IGNORE` | `BIT(1)` | تجاهل الـ link كلياً حتى في cycle detection |

#### `fwnode_graph` lookup flags (في `property.h`)

| الـ Flag | الـ Bit | المعنى |
|---|---|---|
| `FWNODE_GRAPH_ENDPOINT_NEXT` | `BIT(0)` | لو مفيش exact match، خد أقرب endpoint ID أكبر |
| `FWNODE_GRAPH_DEVICE_DISABLED` | `BIT(1)` | اسمح بـ endpoints تبع devices معطّلة أو غير متصلة |

#### `NR_FWNODE_REFERENCE_ARGS` = 16
الحد الأقصى لعدد الـ integer arguments في أي reference property.

---

### الـ Structs المهمة

---

#### 1. `struct fwnode_handle` (في `fwnode.h`)

**الغرض:** الـ handle الموحّد اللي بيمثّل أي firmware node — سواء كان OF (Device Tree)، أو ACPI، أو software node. ده الـ abstraction layer اللي بيخلي الـ drivers ما تشوفش الفرق بين الأنواع دي.

| الحقل | النوع | الدور |
|---|---|---|
| `secondary` | `struct fwnode_handle *` | pointer لـ fwnode تاني (fallback/chain) — بيُستخدم لربط ACPI وـ OF لنفس الجهاز |
| `ops` | `const struct fwnode_operations *` | جدول العمليات (vtable) — ده اللي بيخلي الـ polymorphism يشتغل |
| `dev` | `struct device *` | الـ device المرتبط (بيُستخدم حصرياً من device links) |
| `suppliers` | `struct list_head` | قايمة الـ fwnode links اللي ده consumer فيها |
| `consumers` | `struct list_head` | قايمة الـ fwnode links اللي ده supplier فيها |
| `flags` | `u8` | مجموعة `FWNODE_FLAG_*` |

---

#### 2. `struct fwnode_operations` (في `fwnode.h`)

**الغرض:** الـ vtable — كل نوع firmware node (OF/ACPI/swnode) بيسجّل نسخته الخاصة من الـ ops دي. ده اللي بيخلي `fwnode_property_read_u32()` تشتغل على DT وعلى ACPI بدون ما الـ driver يعرف الفرق.

| الـ Op | التوقيع المختصر | الدور |
|---|---|---|
| `get` / `put` | `fwnode → fwnode` / `void` | reference counting |
| `device_is_available` | `fwnode → bool` | هل الجهاز enabled في الـ firmware? |
| `device_get_match_data` | `fwnode, dev → void *` | الـ driver data المرتبط بالـ compatible/HID |
| `device_dma_supported` | `fwnode → bool` | هل بيدعم DMA? |
| `device_get_dma_attr` | `fwnode → enum dev_dma_attr` | coherent ولا non-coherent? |
| `property_present` | `fwnode, name → bool` | هل الـ property موجودة؟ |
| `property_read_bool` | `fwnode, name → bool` | قيمة property بوليانية |
| `property_read_int_array` | `fwnode, name, elem_size, val, nval → int` | قراءة array من integers بأي حجم |
| `property_read_string_array` | `fwnode, name, val, nval → int` | قراءة array من strings |
| `get_name` / `get_name_prefix` | `fwnode → const char *` | اسم الـ node |
| `get_parent` | `fwnode → fwnode` | الـ node الأب |
| `get_next_child_node` | `fwnode, prev → fwnode` | iteration على الأبناء |
| `get_named_child_node` | `fwnode, name → fwnode` | ابن باسم معين |
| `get_reference_args` | `fwnode, prop, nargs_prop, nargs, idx → args` | reference property مع arguments |
| `graph_get_next_endpoint` | `fwnode, prev → fwnode` | iteration على endpoints |
| `graph_get_remote_endpoint` | `fwnode → fwnode` | الـ endpoint التاني في الـ connection |
| `graph_get_port_parent` | `fwnode → fwnode` | الـ device اللي بيملك الـ port |
| `graph_parse_endpoint` | `fwnode, endpoint → int` | استخراج port/id |
| `iomap` | `fwnode, index → void __iomem *` | map الـ MMIO region |
| `irq_get` | `fwnode, index → int` | رقم الـ IRQ |
| `add_links` | `fwnode → int` | إنشاء fwnode links للـ suppliers |

---

#### 3. `struct fwnode_link` (في `fwnode.h`)

**الغرض:** بيمثّل dependency link بين firmware node supplier وـ consumer — ده اللي بيستخدمه الـ `fw_devlink` system لضمان ترتيب الـ probe.

| الحقل | النوع | الدور |
|---|---|---|
| `supplier` | `struct fwnode_handle *` | الـ node اللي بيقدّم الـ resource |
| `s_hook` | `struct list_head` | hook في قايمة `suppliers` بتاعة الـ supplier |
| `consumer` | `struct fwnode_handle *` | الـ node اللي بيحتاج الـ resource |
| `c_hook` | `struct list_head` | hook في قايمة `consumers` بتاعة الـ consumer |
| `flags` | `u8` | `FWLINK_FLAG_*` |

---

#### 4. `struct fwnode_endpoint` (في `fwnode.h`)

**الغرض:** بيمثّل endpoint في الـ fwnode graph — نقطة الاتصال بين جهازين (مثلاً: كاميرا → ISP).

| الحقل | النوع | الدور |
|---|---|---|
| `port` | `unsigned int` | رقم الـ port على الـ device |
| `id` | `unsigned int` | رقم الـ endpoint داخل الـ port |
| `local_fwnode` | `const struct fwnode_handle *` | الـ fwnode node بتاع الـ endpoint نفسه |

---

#### 5. `struct fwnode_reference_args` (في `fwnode.h`)

**الغرض:** نتيجة تحليل reference property — بيجيب الـ fwnode المُشار إليه مع الـ integer arguments المصاحبة له. مثلاً في DT: `gpios = <&gpioc 5 GPIO_ACTIVE_LOW>` → fwnode=gpioc، args=[5, 1].

| الحقل | النوع | الدور |
|---|---|---|
| `fwnode` | `struct fwnode_handle *` | الـ node المُشار إليه |
| `nargs` | `unsigned int` | عدد الـ arguments الفعلية |
| `args[16]` | `u64[]` | الـ arguments (max 16) |

---

#### 6. `struct property_entry` (في `property.h`)

**الغرض:** تمثيل "مدمج" لأي device property — بيُستخدم في الـ software nodes والـ built-in properties. بيخلّيك تعرّف properties في الـ C code مباشرةً بدون DT أو ACPI.

| الحقل | النوع | الدور |
|---|---|---|
| `name` | `const char *` | اسم الـ property |
| `length` | `size_t` | حجم البيانات بالبايت |
| `is_inline` | `bool` | `true` لو القيمة مخزّنة جوّا الـ struct نفسه |
| `type` | `enum dev_prop_type` | نوع البيانات |
| `pointer` | `const void *` | pointer للبيانات لو مش inline |
| `value` | union | البيانات inline (u8[8] / u16[4] / u32[2] / u64[1] / str[1]) |

الـ union الداخلي بيخلي القيمة الواحدة تتخزن مباشرة في الـ struct بدون allocation لو حجمها ≤ 8 bytes. لو أكبر، بيتخزن في `pointer`.

---

#### 7. `struct software_node_ref_args` (في `property.h`)

**الغرض:** reference property في الـ software nodes — بيخلّيك تشير من software node لـ software node تاني (أو لـ fwnode_handle) مع إرفاق integer arguments زي ما DT بيعمل.

| الحقل | النوع | الدور |
|---|---|---|
| `swnode` | `const struct software_node *` | الـ target لو كان software node |
| `fwnode` | `struct fwnode_handle *` | الـ target لو كان fwnode مباشر |
| `nargs` | `unsigned int` | عدد الـ args |
| `args[16]` | `u64[]` | الـ integer arguments |

---

#### 8. `struct software_node` (في `property.h`)

**الغرض:** بيخلّيك تكتب firmware description للجهاز في C code خالص — من غير DT ومن غير ACPI. بيُستخدم لـ devices اللي الـ BIOS/firmware بتاعها ناقصة أو غلط.

| الحقل | النوع | الدور |
|---|---|---|
| `name` | `const char *` | اسم الـ node (بيبقى unique جوّا parent) |
| `parent` | `const struct software_node *` | الـ node الأب في الشجرة |
| `properties` | `const struct property_entry *` | array من الـ properties (NULL-terminated) |

---

### مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────────┐
│                         struct device                               │
│                    (kernel device object)                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  dev->fwnode
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                    struct fwnode_handle                          │
│  ┌──────────┐  ┌─────────────────────────┐  ┌────────────────┐  │
│  │secondary ├─►│ fwnode_handle (ACPI/OF) │  │ops (vtable ptr)│  │
│  └──────────┘  └─────────────────────────┘  └───────┬────────┘  │
│  ┌──────────┐                                        │           │
│  │suppliers │◄──── fwnode_link.s_hook                │           │
│  └──────────┘                                        ▼           │
│  ┌──────────┐                           ┌─────────────────────┐  │
│  │consumers │◄──── fwnode_link.c_hook   │ fwnode_operations   │  │
│  └──────────┘                           │ (OF/ACPI/swnode ops)│  │
└──────────────────────────────────────────┴─────────────────────┘

                    ▲ (embedded in)
┌───────────────────┴──────────────────────────────────────────────┐
│              struct software_node_fwnode (internal)              │
│  ┌──────────────────────┐   points to                           │
│  │  struct software_node├──────────────────────────────────────►│
│  │  .name               │   ┌────────────────────────────────┐  │
│  │  .parent─────────────┼──►│  struct software_node (parent) │  │
│  │  .properties─────────┼──►│  ...                           │  │
│  └──────────────────────┘   └────────────────────────────────┘  │
│         │ .properties                                            │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  struct property_entry[]  (NULL-terminated array)        │   │
│  │  [0]: { name="clock-frequency", type=U32, val=100000000} │   │
│  │  [1]: { name="compatible",  type=STRING, ptr="ti,bq..."}│   │
│  │  [2]: { name="vdd-supply",  type=REF,    ptr=ref_args } │   │
│  │  [3]: { .name = NULL }  ← sentinel                       │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

```
struct fwnode_link — ربط supplier-consumer

┌──────────────────┐         ┌──────────────────┐
│  fwnode_handle   │         │  fwnode_handle   │
│  (supplier)      │         │  (consumer)      │
│  .consumers list │         │  .suppliers list │
└──────┬───────────┘         └──────────┬───────┘
       │                                │
       │       ┌──────────────────┐     │
       └──────►│  fwnode_link     │◄────┘
               │  .supplier ──────┼──► supplier fwnode
               │  .s_hook (list)  │
               │  .consumer ──────┼──► consumer fwnode
               │  .c_hook (list)  │
               │  .flags          │
               └──────────────────┘
```

```
fwnode graph — ربط endpoints (كاميرا ← → ISP مثلاً)

  [camera device]              [ISP device]
  fwnode_handle                fwnode_handle
       │                            │
   port@0                       port@0
       │                            │
   endpoint@0 ◄────────────► endpoint@0
   (local)     remote-endpoint  (remote)
       │                            │
  fwnode_endpoint              fwnode_endpoint
  .port=0                      .port=0
  .id=0                        .id=0
  .local_fwnode=*              .local_fwnode=*
```

---

### دورة الحياة — Lifecycle Diagrams

#### دورة حياة `software_node`

```
[تعريف static في C code]
        │
        │  static const struct property_entry props[] = {
        │      PROPERTY_ENTRY_U32("clock-frequency", 400000),
        │      PROPERTY_ENTRY_STRING("compatible", "vendor,chip"),
        │      { }
        │  };
        │  static const struct software_node node = {
        │      .name = "my-i2c-device",
        │      .properties = props,
        │  };
        ▼
[التسجيل — Registration]
        │
        ├─ software_node_register(&node)
        │      → يخلق swnode_fwnode داخلي
        │      → يربطه بالـ fwnode_operations بتاعة swnode
        │      → يضيفه لـ global software node list
        ▼
[الربط بالـ device]
        │
        ├─ device_add_software_node(dev, &node)
        │      → يعمل dev->fwnode = software_node_fwnode(&node)
        │      → الـ driver دلوقتي يقدر يقرأ properties عبر
        │        device_property_read_u32(dev, ...)
        ▼
[الاستخدام — Usage]
        │
        ├─ device_property_read_u32(dev, "clock-frequency", &val)
        ├─ device_is_compatible(dev, "vendor,chip")
        ├─ device_for_each_child_node(dev, child) { ... }
        ▼
[الإزالة — Teardown]
        │
        ├─ device_remove_software_node(dev)
        │      → يفكّ ربط الـ dev عن الـ fwnode
        │
        └─ software_node_unregister(&node)
               → يزيله من الـ global list
               → يحرر الـ swnode_fwnode
```

#### دورة حياة `fwnode_handle` مع Reference Counting

```
[الإنشاء]
  fwnode_init(fwnode, &my_ops)
       → fwnode->ops = &my_ops
       → INIT_LIST_HEAD(suppliers/consumers)

[زيادة الـ reference]
  fwnode_handle_get(fwnode)
       → ops->get(fwnode)   [implementation-specific refcount]

[الاستخدام في loops — مع scoped cleanup]
  fwnode_for_each_child_node_scoped(parent, child) {
      /* child automatically put() on loop exit */
      device_property_read_u32(child_dev, ...);
  }
  /* __free(fwnode_handle) → fwnode_handle_put(child) */

[الإنهاء]
  fwnode_handle_put(fwnode)
       → ops->put(fwnode)   [decrement refcount, maybe free]
```

#### دورة حياة `fwnode_reference_args`

```
[طلب reference property]
  fwnode_property_get_reference_args(fwnode,
      "clocks",      /* property name */
      "#clock-cells",/* nargs property */
      0,             /* nargs fallback */
      0,             /* index */
      &args)         /* output */
       ↓
  [ops->get_reference_args() يملا:]
       args.fwnode  = pointer لـ clock provider
       args.nargs   = 1
       args.args[0] = clock ID

[الاستخدام]
  clk = clk_get_from_fwnode(&args);

[التحرير]
  fwnode_handle_put(args.fwnode);
```

---

### مخططات تدفق الاستدعاء — Call Flow Diagrams

#### قراءة property من driver

```
driver calls:
  device_property_read_u32(dev, "clock-frequency", &val)
    │
    ├─► dev_fwnode(dev)          [_Generic macro → __dev_fwnode(dev)]
    │        → returns dev->fwnode
    │
    └─► fwnode_property_read_u32(fwnode, "clock-frequency", &val)
              │
              └─► fwnode_property_read_u32_array(fwnode, name, &val, 1)
                        │
                        └─► fwnode_call_int_op(fwnode,
                                property_read_int_array,
                                name, sizeof(u32), &val, 1)
                                  │
                            ┌─────┴──────────────────────────┐
                            │ if fwnode is OF node:          │
                            │   of_property_read_u32_array() │
                            │   → reads from DTB             │
                            ├────────────────────────────────┤
                            │ if fwnode is ACPI node:        │
                            │   acpi_dev_prop_read()         │
                            │   → reads from ACPI tables     │
                            ├────────────────────────────────┤
                            │ if fwnode is software_node:    │
                            │   software_node_read_array()   │
                            │   → reads from property_entry[]│
                            └────────────────────────────────┘
```

#### البحث عن device عبر fwnode graph

```
driver (ISP) يدور على camera endpoint:
  fwnode_graph_get_next_endpoint(isp_fwnode, NULL)
    │
    └─► ops->graph_get_next_endpoint(fwnode, NULL)
              │
              └─► [OF: of_graph_get_next_endpoint()]
                  [ACPI: acpi_graph_get_next_endpoint()]
                        │
                        returns endpoint fwnode (local)
                              │
  fwnode_graph_get_remote_endpoint(ep_fwnode)
    │
    └─► ops->graph_get_remote_endpoint(ep_fwnode)
              │
              └─► reads "remote-endpoint" property
                  returns remote endpoint fwnode
                        │
  fwnode_graph_get_port_parent(remote_ep)
    │
    └─► ops->graph_get_port_parent(remote_ep)
              │
              returns camera device fwnode
                    │
  device = fwnode_to_device(camera_fwnode)
    → driver gets reference to camera device
```

#### تسجيل software_node وإضافتها لـ device

```
platform code:
  software_node_register_node_group(node_group[])
    │
    ├─ for each node in group:
    │     software_node_register(node)
    │           │
    │           └─► kobject_init_and_add()  [يضيفه لـ sysfs]
    │               swnode->fwnode.ops = &software_node_ops
    │               fwnode_init(&swnode->fwnode, &software_node_ops)
    │
    └─ [later, when i2c/spi device is created]
         device_add_software_node(dev, node)
               │
               └─► software_node_fwnode(node)
                         → returns &swnode->fwnode
                   set_secondary_fwnode(dev, fwnode)
                         → dev->fwnode->secondary = swnode_fwnode
                           (يكون fallback لو primary فشل)
```

#### connection lookup (devcon)

```
driver calls:
  device_connection_find_match(dev, "i2c-bus", data, my_match_fn)
    │
    └─► fwnode_connection_find_match(fwnode, "i2c-bus", data, match)
              │
              ├─ يبحث في الـ fwnode references
              │
              └─ for each connection found:
                    result = match(candidate_fwnode, "i2c-bus", data)
                    if result != NULL → return result
```

---

### استراتيجية الـ Locking

الـ `property.h` نفسه **ما بيعرّفش locks صريحة** — الـ locking موزّع على الـ subsystems اللي بتنفّذ الـ ops:

| الـ Layer | الـ Lock المستخدم | اللي بيحميه |
|---|---|---|
| **OF (Device Tree)** | `of_mutex` / `devtree_lock` | قراءة/كتابة الـ DT nodes وـ properties |
| **ACPI** | `acpi_device_lock` | الـ ACPI namespace |
| **software_node** | `swnode_lock` (mutex) | الـ global software nodes list وـ kset |
| **fwnode_link** | `device_links_lock` (mutex) | قائمتي `suppliers`/`consumers` |
| **fwnode flags** | implicit (boot-time mostly) | الـ `flags` field في `fwnode_handle` |
| **reference counting** | atomic ops / `kref` | `get`/`put` ops |

#### ترتيب الـ Locks (Lock Ordering)

```
[أعلى في الترتيب - يُمسك أولاً]
        device_links_lock  (mutex)
               ↓
        swnode_lock  (mutex)
               ↓
        of_mutex / devtree_lock
               ↓
        كل lock داخل الـ driver نفسه
[أسفل في الترتيب - يُمسك أخيراً]
```

**ملاحظات مهمة:**
- الـ `device_property_read_*` functions كلها **read-only** — آمنة تُستدعى من أي context بعد الـ probe.
- الـ `software_node_register` / `software_node_unregister` يحتاجوا `swnode_lock` — لازم يتعملوا من process context.
- **الـ `fwnode_for_each_child_node_scoped`** بيستخدم `__free(fwnode_handle)` لضمان `put()` تلقائي عند الخروج من الـ scope — بيمنع leaks لو حصل `return` أو `break` في النص.
- الـ `fwnode_handle.suppliers` و `.consumers` lists محمية بـ `device_links_lock` — **ما تقراهاش من interrupt context**.
## Phase 4: شرح الـ Functions

---

### Cheatsheet — كل الـ Functions والـ APIs دفعة واحدة

#### Group 1: Device Property — قراءة Properties من الـ `struct device`

| Function | الغرض | Return |
|---|---|---|
| `device_property_present` | هل الـ property موجودة؟ | `bool` |
| `device_property_read_bool` | قراءة boolean property | `bool` |
| `device_property_read_u8` | قراءة u8 واحد | `int` (0 or -errno) |
| `device_property_read_u16` | قراءة u16 واحد | `int` |
| `device_property_read_u32` | قراءة u32 واحد | `int` |
| `device_property_read_u64` | قراءة u64 واحد | `int` |
| `device_property_read_u8_array` | قراءة array من u8 | `int` |
| `device_property_read_u16_array` | قراءة array من u16 | `int` |
| `device_property_read_u32_array` | قراءة array من u32 | `int` |
| `device_property_read_u64_array` | قراءة array من u64 | `int` |
| `device_property_read_string` | قراءة string واحدة | `int` |
| `device_property_read_string_array` | قراءة array من strings | `int` |
| `device_property_match_string` | إيجاد index لـ string في property | `int` (index or -errno) |
| `device_property_match_property_string` | مطابقة property string مع array ثابت | `int` |
| `device_property_count_u8/16/32/64` | عدد elements في property | `int` |
| `device_property_string_array_count` | عدد strings في property | `int` |
| `device_is_big_endian` | هل الجهاز big-endian؟ | `bool` |
| `device_is_compatible` | مطابقة compatible string | `bool` |
| `device_dma_supported` | هل DMA مدعوم؟ | `bool` |
| `device_get_dma_attr` | نوع الـ DMA coherency | `enum dev_dma_attr` |
| `device_get_match_data` | الـ driver match data | `const void *` |
| `device_get_phy_mode` | قراءة PHY mode | `int` |

#### Group 2: fwnode Property — قراءة Properties من الـ `fwnode_handle`

| Function | الغرض | Return |
|---|---|---|
| `fwnode_property_present` | هل الـ property موجودة في الـ fwnode؟ | `bool` |
| `fwnode_property_read_bool` | قراءة boolean property | `bool` |
| `fwnode_property_read_u8/16/32/64` | قراءة scalar integer | `int` |
| `fwnode_property_read_u8/16/32/64_array` | قراءة integer array | `int` |
| `fwnode_property_read_string` | قراءة string | `int` |
| `fwnode_property_read_string_array` | قراءة string array | `int` |
| `fwnode_property_match_string` | إيجاد index لـ string | `int` |
| `fwnode_property_match_property_string` | مطابقة مع array ثابت | `int` |
| `fwnode_property_count_u8/16/32/64` | عدد elements | `int` |
| `fwnode_property_string_array_count` | عدد strings | `int` |
| `fwnode_property_get_reference_args` | جلب reference مع arguments | `int` |
| `fwnode_get_phy_mode` | PHY mode من fwnode | `int` |
| `fwnode_device_is_available` | هل الـ device متاح؟ | `bool` |
| `fwnode_device_is_big_endian` | big-endian check | `bool` |
| `fwnode_device_is_compatible` | compatible string match | `bool` |

#### Group 3: fwnode Tree Navigation

| Function | الغرض | Return |
|---|---|---|
| `fwnode_get_name` | اسم الـ node | `const char *` |
| `fwnode_get_name_prefix` | prefix للطباعة | `const char *` |
| `fwnode_name_eq` | مقارنة اسم الـ node | `bool` |
| `fwnode_get_parent` | الـ parent node | `fwnode_handle *` |
| `fwnode_get_next_parent` | الـ parent التالي (مع consume للحالي) | `fwnode_handle *` |
| `fwnode_count_parents` | عدد الـ parents | `unsigned int` |
| `fwnode_get_nth_parent` | الـ parent عند depth معين | `fwnode_handle *` |
| `fwnode_get_next_child_node` | الـ child node التالي | `fwnode_handle *` |
| `fwnode_get_next_available_child_node` | الـ child التالي المتاح | `fwnode_handle *` |
| `fwnode_get_named_child_node` | child بالاسم | `fwnode_handle *` |
| `fwnode_get_child_node_count` | عدد children | `unsigned int` |
| `fwnode_get_named_child_node_count` | عدد children باسم معين | `unsigned int` |
| `device_get_next_child_node` | child التالي من device | `fwnode_handle *` |
| `device_get_named_child_node` | child بالاسم من device | `fwnode_handle *` |
| `device_get_child_node_count` | عدد children للـ device | `unsigned int` |
| `device_get_named_child_node_count` | عدد children باسم | `unsigned int` |
| `fwnode_handle_get` | increment refcount | `fwnode_handle *` |
| `fwnode_handle_put` | decrement refcount | `void` |
| `fwnode_find_reference` | جلب reference node باسم وindex | `fwnode_handle *` |

#### Group 4: fwnode Graph (V4L2/media graph model)

| Function | الغرض | Return |
|---|---|---|
| `fwnode_graph_get_next_endpoint` | الـ endpoint التالي في الـ graph | `fwnode_handle *` |
| `fwnode_graph_get_port_parent` | الـ parent للـ port | `fwnode_handle *` |
| `fwnode_graph_get_remote_port_parent` | الـ remote device | `fwnode_handle *` |
| `fwnode_graph_get_remote_port` | الـ remote port | `fwnode_handle *` |
| `fwnode_graph_get_remote_endpoint` | الـ remote endpoint | `fwnode_handle *` |
| `fwnode_graph_get_endpoint_by_id` | endpoint بـ port/endpoint id | `fwnode_handle *` |
| `fwnode_graph_get_endpoint_count` | عدد endpoints | `unsigned int` |
| `fwnode_graph_parse_endpoint` | parse endpoint info | `int` |
| `fwnode_graph_is_endpoint` | هل الـ node endpoint؟ | `bool` |
| `fwnode_iomap` | map MMIO region من fwnode | `void __iomem *` |
| `fwnode_irq_get` | جلب IRQ number بـ index | `int` |
| `fwnode_irq_get_byname` | جلب IRQ number بالاسم | `int` |

#### Group 5: Connection / Consumer-Supplier

| Function | الغرض | Return |
|---|---|---|
| `fwnode_connection_find_match` | إيجاد أول connection match | `void *` |
| `device_connection_find_match` | نفس السابق من device | `void *` |
| `fwnode_connection_find_matches` | إيجاد كل connections | `int` |

#### Group 6: Software Nodes

| Function | الغرض | Return |
|---|---|---|
| `is_software_node` | هل الـ fwnode software node؟ | `bool` |
| `to_software_node` | cast من fwnode لـ software_node | `const struct software_node *` |
| `software_node_fwnode` | جلب fwnode_handle من software_node | `fwnode_handle *` |
| `software_node_find_by_name` | إيجاد software_node بالاسم | `const struct software_node *` |
| `software_node_register` | تسجيل node واحد | `int` |
| `software_node_unregister` | إلغاء تسجيل node واحد | `void` |
| `software_node_register_node_group` | تسجيل مجموعة nodes | `int` |
| `software_node_unregister_node_group` | إلغاء تسجيل مجموعة | `void` |
| `fwnode_create_software_node` | إنشاء software node من properties | `fwnode_handle *` |
| `fwnode_remove_software_node` | حذف software node | `void` |
| `device_add_software_node` | ربط software node بـ device | `int` |
| `device_remove_software_node` | فك الربط | `void` |
| `device_create_managed_software_node` | إنشاء وربط managed node | `int` |

#### Group 7: Helpers للـ `dev_fwnode`

| Function/Macro | الغرض | Return |
|---|---|---|
| `dev_fwnode(dev)` | جلب `fwnode_handle` من device (const-safe) | `fwnode_handle *` |
| `__dev_fwnode` | non-const version | `fwnode_handle *` |
| `__dev_fwnode_const` | const version | `const fwnode_handle *` |

#### Group 8: property_entry Helpers والـ Macros

| Macro | الغرض |
|---|---|
| `PROPERTY_ENTRY_U8/16/32/64` | scalar inline entry |
| `PROPERTY_ENTRY_U8/16/32/64_ARRAY` | array entry (pointer to data) |
| `PROPERTY_ENTRY_STRING` | single string entry |
| `PROPERTY_ENTRY_STRING_ARRAY` | string array entry |
| `PROPERTY_ENTRY_BOOL` | boolean (exists = true) |
| `PROPERTY_ENTRY_REF` | reference entry |
| `PROPERTY_ENTRY_REF_ARRAY` | array of references |
| `SOFTWARE_NODE_REFERENCE` | بناء `software_node_ref_args` |
| `SOFTWARE_NODE` | compound literal لـ `software_node` |
| `property_entries_dup` | نسخ property array في الـ heap |
| `property_entries_free` | تحرير المنسوخ |

---

### Group 1: Device Property Read Functions

الـ group ده بيوفر الـ API الأساسية اللي بيستخدمها الـ driver مباشرة مع `struct device`. كل function فيه بيعمل `dev_fwnode(dev)` جوا وبيفوّض للـ `fwnode_*` counterpart. الغرض هو abstraction كاملة فوق DT / ACPI / software nodes.

---

#### `device_property_present`

```c
bool device_property_present(const struct device *dev, const char *propname);
```

بتتحقق إن الـ property موجودة أصلاً في الـ firmware description للـ device. مش بتقرأ القيمة، بس بتأكد وجودها كـ boolean check.

**Parameters:**
- `dev` — الـ device المستهدف
- `propname` — اسم الـ property كـ string literal

**Return:** `true` لو الـ property موجودة، `false` لو مش موجودة أو فيه error.

**Key details:** بتمشي على الـ fwnode chain (primary → secondary) لو فيه chaining. لا locking مطلوب من الـ caller.

**Who calls it:** أي driver محتاج يعمل conditional check قبل قراءة property اختيارية، مثلاً `if (device_property_present(dev, "wakeup-source"))`.

---

#### `device_property_read_bool`

```c
bool device_property_read_bool(const struct device *dev, const char *propname);
```

بتقرأ boolean property صريحة. الفرق عن `_present`: ممكن الـ property تكون موجودة وقيمتها false.

**Parameters:**
- `dev` — الـ device
- `propname` — اسم الـ property

**Return:** قيمة الـ boolean property.

**Key details:** في DT، boolean properties بتكون موجودة = `true`، مش موجودة = `false`. في ACPI فيه فرق حقيقي بين وجود الـ property وقيمتها.

---

#### `device_property_read_u8_array` (و u16/u32/u64 variants)

```c
int device_property_read_u8_array(const struct device *dev, const char *propname,
                                  u8 *val, size_t nval);
```

بتقرأ array من قيم integer من الـ firmware property. الـ variants الأربعة مختلفة بس في حجم العنصر.

**Parameters:**
- `dev` — الـ device
- `propname` — اسم الـ property
- `val` — الـ buffer اللي هيتكتب فيه، لو `NULL` والـ `nval == 0` بترجع عدد العناصر
- `nval` — عدد العناصر المطلوب قراءتهم، لو `0` بترجع عدد العناصر

**Return:** `0` عند النجاح، أو عدد العناصر لو `val == NULL && nval == 0`. عند الفشل: `-EINVAL` (اسم غلط)، `-ENODATA` (property موجودة بس فاضية)، `-EOVERFLOW` (الـ array أكبر من الـ buffer)، `-ENXIO` (مش موجودة).

**Key details:** الـ trick الأساسي — تمرير `NULL, 0` بيحوّل الدالة لـ "count" function بدون allocation. الـ endianness conversion بيحصل داخلياً حسب الـ firmware backend.

**Who calls it:** الـ drivers اللي بتقرأ numeric arrays من DT/ACPI زي interrupt configurations, timing tables.

**Pseudocode:**
```
device_property_read_u8_array(dev, prop, val, nval):
    fwnode = dev_fwnode(dev)
    return fwnode_property_read_u8_array(fwnode, prop, val, nval)
        → calls fwnode->ops->property_read_int_array(fwnode, prop, sizeof(u8), val, nval)
```

---

#### `device_property_read_string`

```c
int device_property_read_string(const struct device *dev, const char *propname,
                                const char **val);
```

بتقرأ أول string من property. الـ pointer اللي بترجعه بيشاور على الـ firmware data نفسه، مش نسخة.

**Parameters:**
- `dev` — الـ device
- `propname` — اسم الـ property
- `val` — pointer لـ pointer هيشاور على الـ string

**Return:** `0` عند النجاح، `-errno` عند الفشل.

**Key details:** الـ string مش mutable، ومش محتاج تعمل `kfree` عليها.

---

#### `device_property_match_string`

```c
int device_property_match_string(const struct device *dev,
                                 const char *propname, const char *string);
```

بتدور على `string` جوا property اسمها `propname` اللي بتحتوي على string list. بتعمل exact string match.

**Parameters:**
- `dev` — الـ device
- `propname` — الـ property اللي فيها list من strings
- `string` — الـ string اللي بندور عليها

**Return:** الـ index (≥ 0) لو لقتها، `-ENODATA` لو الـ property فاضية، `-EILSEQ` لو مش لاقي match.

**Who calls it:** common pattern في الـ PHY subsystem والـ clock drivers لتحديد operating mode من string list.

---

#### `device_property_count_u8/16/32/64` و `device_property_string_array_count`

```c
static inline int device_property_count_u32(const struct device *dev, const char *propname)
{
    return device_property_read_u32_array(dev, propname, NULL, 0);
}
```

بتستغل الـ trick الخاصة بالـ `_array` functions: لو `val == NULL` و `nval == 0`، بترجع عدد العناصر. الـ inline wrapper بيوضّح النية.

**Return:** عدد العناصر (≥ 0) أو `-errno`.

---

### Group 2: fwnode Property Functions

الـ device_property_* functions كلها مجرد thin wrappers فوق الـ `fwnode_property_*`. الـ fwnode API هو الـ real implementation layer اللي بيتكلم مع الـ backend (OF, ACPI, software nodes).

#### `fwnode_property_present`

```c
bool fwnode_property_present(const struct fwnode_handle *fwnode,
                             const char *propname);
```

بتفحص وجود property في fwnode معين بشكل مباشر. بتستدعي `fwnode->ops->property_present` عبر الـ `fwnode_call_bool_op` macro.

**Key details:** لو الـ fwnode `NULL` أو `IS_ERR`، بترجع `false` automatically بسبب `fwnode_call_bool_op`.

---

#### `fwnode_property_get_reference_args`

```c
int fwnode_property_get_reference_args(const struct fwnode_handle *fwnode,
                                       const char *prop, const char *nargs_prop,
                                       unsigned int nargs, unsigned int index,
                                       struct fwnode_reference_args *args);
```

دي من أهم الـ APIs لأنها بتقرأ phandle references مع arguments زي ما بتيجي في DT. مثال: `clocks = <&clk_ctrl 2 0>` — هنا الـ phandle هو `&clk_ctrl` والـ `2` و `0` هم arguments.

**Parameters:**
- `fwnode` — الـ consumer node
- `prop` — اسم الـ property اللي فيها الـ reference (مثلاً `"clocks"`)
- `nargs_prop` — اسم الـ property في الـ provider اللي بيحدد عدد الـ args (مثلاً `"#clock-cells"`)، ممكن `NULL`
- `nargs` — عدد ثابت للـ args لو `nargs_prop == NULL`
- `index` — رقم الـ entry في الـ list (لو فيه أكثر من reference)
- `args` — الـ output struct، بتتملى بـ `fwnode` الـ provider والـ `args[]`

**Return:** `0` عند النجاح، `-errno` عند الفشل.

**Key details:** بتبقى essential لكل subsystem بيستخدم reference cells: clock, GPIO, interrupts, DMA, reset.

**Pseudocode:**
```
fwnode_property_get_reference_args(fwnode, prop, nargs_prop, nargs, index, args):
    call fwnode->ops->get_reference_args(...)
    if success:
        args->fwnode = resolved_provider_fwnode  // refcount incremented
        args->nargs = actual_nargs
        args->args[] = extracted values
    return 0 or -errno
```

---

#### `fwnode_find_reference`

```c
struct fwnode_handle *fwnode_find_reference(const struct fwnode_handle *fwnode,
                                            const char *name,
                                            unsigned int index);
```

بتجيب reference node من property بالاسم والـ index بدون arguments. أبسط من `get_reference_args` للحالات اللي فيها reference بس من غير cells.

**Return:** `fwnode_handle *` مع refcount مرفوع، أو `ERR_PTR(-errno)`.

---

### Group 3: Tree Navigation

الـ group ده بيوفر الـ API لاستعراض شجرة الـ firmware nodes.

#### `fwnode_get_parent`

```c
struct fwnode_handle *fwnode_get_parent(const struct fwnode_handle *fwnode);
```

بتعيد الـ parent node مع زيادة الـ refcount. الـ caller مسؤول عن استدعاء `fwnode_handle_put()`.

**Return:** `fwnode_handle *` الـ parent أو `NULL` لو مفيش parent.

---

#### `fwnode_get_next_parent`

```c
struct fwnode_handle *fwnode_get_next_parent(struct fwnode_handle *fwnode);
```

بتعيد الـ parent وفي نفس الوقت بتعمل `put` على الـ fwnode اللي اتبعتلها. ده بيخليها مناسبة للاستخدام في loops.

**Key details:** الفرق عن `fwnode_get_parent`: الـ input fwnode بيتعمله `put` جوا الدالة. ده بيسمح بـ iteration without leaks.

```c
// مثال: المشي لفوق في الشجرة
struct fwnode_handle *fwnode = fwnode_handle_get(start);
fwnode_for_each_parent_node(start, fwnode) {
    // fwnode هو parent حالي
    // الـ put بيحصل automatically في الـ loop
}
```

---

#### `fwnode_get_next_child_node`

```c
struct fwnode_handle *fwnode_get_next_child_node(
    const struct fwnode_handle *fwnode, struct fwnode_handle *child);
```

بتعيد الـ child التالي في الـ iteration. لو `child == NULL`، بتبدأ من أول child.

**Key details:** `child` السابق بيتعمله `put` جوا الدالة. لو خرجت من الـ loop بـ `break`، لازم تعمل `fwnode_handle_put(child)` يدوياً أو تستخدم الـ `_scoped` variant.

**Who calls it:** مباشرة عن طريق الـ `fwnode_for_each_child_node` macro.

---

#### `fwnode_for_each_child_node_scoped` (macro)

```c
#define fwnode_for_each_child_node_scoped(fwnode, child)            \
    for (struct fwnode_handle *child __free(fwnode_handle) =        \
            fwnode_get_next_child_node(fwnode, NULL);               \
         child; child = fwnode_get_next_child_node(fwnode, child))
```

بيستخدم الـ cleanup attribute (`__free(fwnode_handle)`) عشان الـ `child` يتعمله `put` automatically عند الخروج من الـ scope — سواء بـ `break`، `return`، أو error. هو أكثر أماناً من `fwnode_for_each_child_node` العادي.

---

#### `fwnode_handle_get` / `fwnode_handle_put`

```c
struct fwnode_handle *fwnode_handle_get(struct fwnode_handle *fwnode);

static inline void fwnode_handle_put(struct fwnode_handle *fwnode)
{
    fwnode_call_void_op(fwnode, put);
}
```

الـ reference counting الأساسي. `get` بيزود الـ refcount، `put` بيقلله وممكن يعمل free.

**Key details:**
- `DEFINE_FREE(fwnode_handle, struct fwnode_handle *, fwnode_handle_put(_T))` بيعرّف الـ cleanup destructor للـ `__free` attribute.
- لازم كل `get` يقابله `put`.

---

#### `fwnode_get_nth_parent`

```c
struct fwnode_handle *fwnode_get_nth_parent(struct fwnode_handle *fwn,
                                            unsigned int depth);
```

بتجيب الـ ancestor على بُعد `depth` من الـ node. `depth == 0` بيعيد الـ node نفسه مع `get`، `depth == 1` بيعيد الـ parent.

**Key details:** بتعمل `fwnode_handle_put` على الـ intermediate nodes جوا الـ loop.

---

### Group 4: fwnode Graph API

الـ fwnode graph API بتعكس نموذج الـ **OF graph** اللي اتعمل أصلاً لـ V4L2 و media pipelines. كل device ممكن يكون عنده **ports**، وكل port عنده **endpoints**. الـ endpoint بيربط على endpoint تاني في device تاني.

```
Device A                    Device B
  └─ port@0                   └─ port@0
       └─ endpoint@0 ←────────── endpoint@0
```

---

#### `fwnode_graph_get_next_endpoint`

```c
struct fwnode_handle *fwnode_graph_get_next_endpoint(
    const struct fwnode_handle *fwnode, struct fwnode_handle *prev);
```

بتعمل iteration على كل الـ endpoints في device. لو `prev == NULL`، بتبدأ من أول endpoint.

**Return:** الـ endpoint التالي مع refcount مرفوع، أو `NULL` لو خلصوا.

**Key details:** `prev` بيتعمله `put` جوا الدالة زي الـ child iteration. استخدم الـ `fwnode_graph_for_each_endpoint` macro.

---

#### `fwnode_graph_get_remote_endpoint`

```c
struct fwnode_handle *fwnode_graph_get_remote_endpoint(
    const struct fwnode_handle *fwnode);
```

بتعيد الـ endpoint الـ remote اللي الـ local endpoint ده متصل بيه.

**Return:** `fwnode_handle *` للـ remote endpoint أو `NULL`.

---

#### `fwnode_graph_get_remote_port_parent`

```c
struct fwnode_handle *fwnode_graph_get_remote_port_parent(
    const struct fwnode_handle *fwnode);
```

بتعيد الـ **device** اللي الـ remote endpoint ينتمي له (مش الـ port، مش الـ endpoint — الـ device كلها). ده الأكثر استخداماً في الـ media drivers عشان يعرفوا مين الـ device المتصلة.

**Who calls it:** V4L2 subdev drivers, MIPI CSI drivers, display controllers.

---

#### `fwnode_graph_get_endpoint_by_id`

```c
struct fwnode_handle *
fwnode_graph_get_endpoint_by_id(const struct fwnode_handle *fwnode,
                                u32 port, u32 endpoint, unsigned long flags);
```

بتجيب endpoint محدد بـ port ID وendpoint ID.

**Parameters:**
- `fwnode` — الـ device node
- `port` — رقم الـ port
- `endpoint` — رقم الـ endpoint في الـ port
- `flags` — `FWNODE_GRAPH_ENDPOINT_NEXT` أو `FWNODE_GRAPH_DEVICE_DISABLED`

**Key details:**
- `FWNODE_GRAPH_ENDPOINT_NEXT`: لو مش لاقي exact match، بياخد الـ endpoint اللي عنده ID أكبر.
- `FWNODE_GRAPH_DEVICE_DISABLED`: بيرجع endpoints حتى لو الـ remote device disabled.

---

#### `fwnode_graph_parse_endpoint`

```c
int fwnode_graph_parse_endpoint(const struct fwnode_handle *fwnode,
                                struct fwnode_endpoint *endpoint);
```

بتملأ `struct fwnode_endpoint` بالـ `port` و `id` و `local_fwnode` من الـ endpoint node. ده الـ parsing step الأساسي في أي media driver.

**Return:** `0` عند النجاح.

---

#### `fwnode_iomap`

```c
void __iomem *fwnode_iomap(struct fwnode_handle *fwnode, int index);
```

بتعمل map لـ MMIO region محددة بـ index من الـ fwnode. الـ ACPI backend بيستخدم الـ `_CRS` resources.

**Return:** `void __iomem *` pointer أو `NULL` عند الفشل.

---

#### `fwnode_irq_get` / `fwnode_irq_get_byname`

```c
int fwnode_irq_get(const struct fwnode_handle *fwnode, unsigned int index);
int fwnode_irq_get_byname(const struct fwnode_handle *fwnode, const char *name);
```

بيجيبوا الـ Linux IRQ number. `fwnode_irq_get` بيستخدم رقم index، `_byname` بيستخدم اسم الـ interrupt من `interrupt-names` property.

**Return:** IRQ number (> 0) عند النجاح، `-errno` عند الفشل.

---

### Group 5: Connection Find Match

#### `fwnode_connection_find_match`

```c
void *fwnode_connection_find_match(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match);
```

بتدور على connections ربطها الـ firmware ودى الـ match function callback لكل واحدة. أول ما الـ callback يرجع non-NULL قيمة، الـ function بترجعها.

**Parameters:**
- `fwnode` — الـ consumer
- `con_id` — connection ID string (مثلاً `"default"` للـ pinctrl)
- `data` — بيتمرر as-is للـ match callback
- `match` — الـ callback: `typedef void *(*devcon_match_fn_t)(const struct fwnode_handle *fwnode, const char *id, void *data)`

**Return:** أول non-NULL result من الـ match callback.

**Who calls it:** regulator framework، pinctrl، phy consumers.

---

#### `fwnode_connection_find_matches`

```c
int fwnode_connection_find_matches(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match,
                                   void **matches, unsigned int matches_len);
```

زي السابق بس بيجمع **كل** الـ matches في array بدل ما توقف عند أول واحدة.

**Parameters:**
- `matches` — array هيتملى بالـ results
- `matches_len` — حجم الـ array

**Return:** عدد الـ matches اللي اتلقوا أو `-errno`.

---

### Group 6: Software Nodes

الـ **software nodes** بيحلوا مشكلة الـ devices اللي مش موصوفة في DT أو ACPI — زي الـ I2C devices اللي بتتعرفها بـ PCI ID. بيوفروا نفس الـ fwnode interface بس من memory structures ثابتة في الـ kernel.

---

#### `software_node_register` / `software_node_unregister`

```c
int software_node_register(const struct software_node *node);
void software_node_unregister(const struct software_node *node);
```

بيسجّل node واحد في الـ software node global registry. بعد التسجيل، الـ node بقى accessible عبر الـ fwnode API.

**Key details:**
- لازم الـ parent يتسجل قبل الـ children.
- الـ unregister: لازم تشيل الـ children الأول.
- الـ registration بيعمل `fwnode_init` للـ embedded `fwnode_handle`.

---

#### `software_node_register_node_group`

```c
int software_node_register_node_group(const struct software_node * const *node_group);
```

بتسجل array من pointers لـ `software_node` (منتهية بـ `NULL`). بيسهل تسجيل hierarchy كاملة دفعة واحدة.

**Return:** `0` عند النجاح، `-errno` لو في node فيها error — مع **rollback** تلقائي لكل اللي اتسجلوا قبلها.

---

#### `fwnode_create_software_node`

```c
struct fwnode_handle *
fwnode_create_software_node(const struct property_entry *properties,
                            const struct fwnode_handle *parent);
```

بتنشئ software node ديناميكياً من `property_entry` array. الـ node بيتنشئ في الـ heap، مش static كالـ `software_node_register`.

**Parameters:**
- `properties` — array من `property_entry` منتهية بـ empty entry
- `parent` — الـ parent fwnode (اختياري، ممكن `NULL`)

**Return:** `fwnode_handle *` الجديد أو `ERR_PTR(-errno)`.

**Who calls it:** الـ drivers اللي محتاجة تضيف properties runtime مثل platform glue code.

---

#### `device_add_software_node`

```c
int device_add_software_node(struct device *dev, const struct software_node *node);
```

بتربط software node بـ device موجودة. الـ device الـ `fwnode` بتاعها بتتبدل أو بتتسلسل مع الـ node الجديدة.

**Return:** `0` أو `-errno`.

**Key details:** لو الـ device عندها فعلاً fwnode، الـ software node بيتحط كـ secondary fwnode. ده بيخلي الـ properties من الاتنين accessible.

---

#### `device_create_managed_software_node`

```c
int device_create_managed_software_node(struct device *dev,
                                        const struct property_entry *properties,
                                        const struct software_node *parent);
```

بتجمع الإنشاء والربط في خطوة واحدة، والأهم إنها **managed** — يعني الـ node بيتحذف تلقائياً لما الـ device تتـ destroy عبر الـ devres framework.

**Who calls it:** الـ bus drivers اللي بتضيف properties لـ devices اتعرفت من PCI/USB IDs.

---

#### `is_software_node` / `to_software_node`

```c
bool is_software_node(const struct fwnode_handle *fwnode);
const struct software_node *to_software_node(const struct fwnode_handle *fwnode);
```

**الـ** `is_software_node` بيتحقق إن الـ ops pointer بتاع الـ fwnode هو الـ software node ops. الـ `to_software_node` بيعمل `container_of` للوصول للـ parent struct.

**Key details:** `to_software_node` بترجع `NULL` لو الـ fwnode مش software node.

---

### Group 7: dev_fwnode و Endianness Helpers

#### `dev_fwnode` (macro)

```c
#define dev_fwnode(dev)                             \
    _Generic((dev),                                 \
             const struct device *: __dev_fwnode_const, \
             struct device *: __dev_fwnode)(dev)
```

الـ macro ده بيستخدم C11 `_Generic` للـ type-safe dispatch. لو الـ input `const struct device *`، بيستدعي `__dev_fwnode_const` اللي بترجع `const fwnode_handle *`. ده بيمنع الـ const-correctness violations في الـ type system.

**Who calls it:** كل الـ `device_property_*` و `device_get_*` functions جوا تنفيذها.

---

#### `device_is_big_endian`

```c
static inline bool device_is_big_endian(const struct device *dev)
{
    return fwnode_device_is_big_endian(dev_fwnode(dev));
}
```

#### `fwnode_device_is_big_endian`

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

بتحدد إذا كانت registers الجهاز big-endian. الـ logic:
1. لو فيه `big-endian` property → دايماً BE
2. لو فيه `native-endian` → BE بس لو الـ kernel اتبنى لـ BE

**Who calls it:** الـ network و USB controllers اللي عندها byte-swapped register access (`ioread32be`/`iowrite32be`).

---

### Group 8: property_entry Macros والـ Structs

#### `struct property_entry`

```c
struct property_entry {
    const char *name;
    size_t length;
    bool is_inline;
    enum dev_prop_type type;
    union {
        const void *pointer;   // for arrays (non-inline)
        union {
            u8  u8_data[8];
            u16 u16_data[4];
            u32 u32_data[2];
            u64 u64_data[1];
            const char *str[1]; // on 64-bit: 1 pointer fits
        } value;               // for inline scalar values
    };
};
```

ده الـ representation الأساسي للـ property في الـ memory. الـ `is_inline` flag بيحدد إذا القيمة مخزنة جوا الـ struct (في `value`) أو بتشاور على external buffer (في `pointer`).

**Key details:**
- الـ `is_inline = false` + `length = 0` = boolean property (مجرد وجودها = `true`)
- الـ inline storage محدودة بـ 8 bytes (حجم `u64`)
- الـ arrays بتستخدم الـ `pointer` field دايماً

---

#### `PROPERTY_ENTRY_U32` (و variants)

```c
#define PROPERTY_ENTRY_U32(_name_, _val_)   \
    __PROPERTY_ENTRY_ELEMENT(_name_, u32_data, U32, _val_)
```

**الـ** internal macro `__PROPERTY_ENTRY_ELEMENT`:

```c
#define __PROPERTY_ENTRY_ELEMENT(_name_, _elem_, _Type_, _val_)  \
(struct property_entry) {                                        \
    .name = _name_,                                              \
    .length = sizeof_field(struct property_entry, value._elem_[0]), \
    .is_inline = true,                                           \
    .type = DEV_PROP_##_Type_,                                   \
    { .value = { ._elem_[0] = _val_ } },                        \
}
```

بيبني compound literal مباشرة. مناسب للـ static initialization في الـ driver code.

**مثال كامل:**
```c
static const struct property_entry my_dev_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("label", "my-sensor"),
    PROPERTY_ENTRY_BOOL("wakeup-source"),
    PROPERTY_ENTRY_U32_ARRAY("reg-io-width", ((u32[]){1, 4})),
    { }  // sentinel
};
```

---

#### `PROPERTY_ENTRY_BOOL`

```c
#define PROPERTY_ENTRY_BOOL(_name_)     \
(struct property_entry) {               \
    .name = _name_,                     \
    .is_inline = true,                  \
}
```

بيبني boolean property. لاحظ إن `length == 0` و `type == 0` (DEV_PROP_U8). الـ interpretation هي: لو الـ property موجودة في الـ array → `true`.

---

#### `PROPERTY_ENTRY_REF`

```c
#define PROPERTY_ENTRY_REF(_name_, _ref_, ...)                    \
(struct property_entry) {                                         \
    .name = _name_,                                               \
    .length = sizeof(struct software_node_ref_args),              \
    .type = DEV_PROP_REF,                                         \
    { .pointer = &SOFTWARE_NODE_REFERENCE(_ref_, ##__VA_ARGS__) },\
}
```

بيبني reference property تشاور على software node أو fwnode تاني مع optional integer arguments. الـ `SOFTWARE_NODE_REFERENCE` macro بيستخدم `_Generic` للتمييز بين `software_node *` و `fwnode_handle *`.

**مثال:**
```c
// reference لـ software node بدون args
PROPERTY_ENTRY_REF("power-domains", &my_power_domain_node),

// reference مع argument
PROPERTY_ENTRY_REF("clocks", &my_clk_node, 2),
```

---

#### `property_entries_dup` / `property_entries_free`

```c
struct property_entry *
property_entries_dup(const struct property_entry *properties);
void property_entries_free(const struct property_entry *properties);
```

`property_entries_dup` بتعمل deep copy لـ array من `property_entry` في الـ heap، بما في ذلك الـ strings والـ data pointers. ضروري لأن الـ original array ممكن تبقى على الـ stack.

`property_entries_free` بتحرر الـ heap memory اللي `dup` خصصها.

**Key details:** الـ copy بيشمل الـ name strings والـ pointed-to data — مش مجرد shallow copy.

**Who calls it:** الـ `device_create_managed_software_node` والـ `fwnode_create_software_node` بيستخدموا `property_entries_dup` جوا تنفيذهم.

---

### Group 9: Device-Level Utility Functions

#### `device_dma_supported` / `device_get_dma_attr`

```c
bool device_dma_supported(const struct device *dev);
enum dev_dma_attr device_get_dma_attr(const struct device *dev);
```

`device_dma_supported` بتعيد `true` لو الـ firmware وصف الـ device بإنها تدعم DMA. `device_get_dma_attr` بترجع نوع الـ coherency: `DEV_DMA_NOT_SUPPORTED`، `DEV_DMA_NON_COHERENT`، أو `DEV_DMA_COHERENT`.

**Who calls it:** الـ DMA subsystem initialization، IOMMU setup code.

---

#### `device_get_match_data`

```c
const void *device_get_match_data(const struct device *dev);
```

بترجع الـ driver-specific data المرتبطة بالـ device ID table entry اللي اتعرف بيها الجهاز. مثلاً في DT، الـ `of_device_id.data` pointer.

**Return:** `const void *` pointer للـ data أو `NULL`.

**Who calls it:** شائع جداً في الـ drivers عشان يجيب per-variant configuration بدون switch/case على compatible strings.

```c
// typical driver usage
const struct my_config *cfg = device_get_match_data(dev);
```

---

#### `device_get_phy_mode` / `fwnode_get_phy_mode`

```c
int device_get_phy_mode(struct device *dev);
int fwnode_get_phy_mode(const struct fwnode_handle *fwnode);
```

بيقرأوا `phy-mode` أو `phy-connection-type` property من الـ firmware ويحوّلوها لـ enum value من `PHY_INTERFACE_MODE_*` (معرّفة في `linux/phy.h`).

**Return:** قيمة enum موجبة عند النجاح، `-errno` عند الفشل.

**Who calls it:** Ethernet MAC drivers لتحديد ما إذا كانت الـ interface MII/RMII/RGMII/SGMII...إلخ.

---

### Iteration Macros — ملخص

| Macro | الوصف | Safe break? |
|---|---|---|
| `fwnode_for_each_child_node` | iterate كل children | لا، لازم manual put |
| `fwnode_for_each_available_child_node` | iterate المتاحين فقط | لا، لازم manual put |
| `fwnode_for_each_named_child_node` | iterate children باسم معين | لا |
| `fwnode_for_each_child_node_scoped` | iterate مع auto-cleanup | نعم (`__free`) |
| `fwnode_for_each_available_child_node_scoped` | available + auto-cleanup | نعم |
| `fwnode_for_each_parent_node` | iterate parents للأعلى | لا |
| `device_for_each_child_node` | من device مباشرة | لا |
| `device_for_each_child_node_scoped` | من device مع cleanup | نعم |
| `device_for_each_named_child_node` | من device باسم | لا |
| `device_for_each_named_child_node_scoped` | من device باسم + cleanup | نعم |
| `fwnode_graph_for_each_endpoint` | iterate endpoints في graph | لا |

**القاعدة:** أي macro بينتهي بـ `_scoped` آمن للـ `break`/`return` لأنه بيستخدم الـ `__free(fwnode_handle)` cleanup attribute. الباقي لازم `fwnode_handle_put(child)` قبل الخروج.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

**الـ `fwnode`** والـ `software_node` مش بيعملوا entries في debugfs بشكل مباشر، لكن الـ device model نفسه بيعمل حاجات مفيدة جداً:

```bash
# شوف كل الـ fwnode links (supplier/consumer relationships)
cat /sys/kernel/debug/device_component/devices

# الـ fw_devlink — أهم مصدر لمشاكل الـ probe defer
cat /sys/kernel/debug/devices_deferred

# شوف شجرة الـ software nodes (kernel >= 5.8)
ls /sys/kernel/debug/software_nodes/

# لو عندك OF (Device Tree):
ls /sys/kernel/debug/devicetree/
cat /sys/kernel/debug/devicetree/dynamic
```

##### إزاي تقرأهم:

```bash
# كل سطر في devices_deferred بيقولك اسم الـ device والسبب
# مثال:
# spi-0:  Driver spi-gpio is not ready
# i2c-1:  Supplier /soc/i2c@... is not ready

# software_nodes — بيشوفك الـ nodes المسجلة حالياً:
ls /sys/kernel/debug/software_nodes/
# my-sensor
# my-regulator

cat /sys/kernel/debug/software_nodes/my-sensor/properties
# speed-hz: 1000000
# compatible: vendor,my-sensor
```

---

#### 2. مدخلات الـ sysfs

```bash
# الـ fwnode الخاص بـ device معين
ls /sys/bus/platform/devices/my-device/
# of_node -> رابط لـ DT node (لو OF)
# firmware_node -> رابط لـ ACPI node (لو ACPI)
# driver_override

# اقرأ property مباشرة من sysfs لـ DT:
cat /sys/firmware/devicetree/base/soc/i2c@40005400/clock-frequency

# شوف الـ compatible string:
cat /sys/bus/platform/devices/my-device/of_node/compatible

# الـ software nodes (kernel >= 5.17):
ls /sys/firmware/software_nodes/

# الـ devlink state لكل device:
cat /sys/bus/platform/devices/my-device/power/runtime_status
```

---

#### 3. استخدام الـ ftrace

##### تفعيل الـ tracepoints المهمة:

```bash
# --- تفعيل kprobes على دوال الـ property API ---

# تتبع كل قراءة property
echo 'p:probe_prop_read device_property_read_u32_array dev=%di propname=%si' \
    > /sys/kernel/debug/tracing/kprobe_events

echo 1 > /sys/kernel/debug/tracing/events/kprobes/probe_prop_read/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# --- تفعيل فعاليات الـ fwnode graph ---
echo 1 > /sys/kernel/debug/tracing/events/device_driver/enable

# تتبع مشاكل الـ probe deferral
echo 1 > /sys/kernel/debug/tracing/events/devres/enable
```

##### تتبع `fwnode_property_present` و `fwnode_device_is_available`:

```bash
# بـ function_graph tracer
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'fwnode_property_present
fwnode_device_is_available
device_property_read_u32_array
software_node_register' > /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ driver ثم:
cat /sys/kernel/debug/tracing/trace | grep -E 'fwnode|property|swnode'
```

---

#### 4. الـ printk والـ Dynamic Debug

##### تفعيل dynamic debug للـ property subsystem:

```bash
# تفعيل كل رسائل الـ debug في ملفات الـ property
echo 'file drivers/base/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/swnode.c +p'   > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/fwnode.c +p'   > /sys/kernel/debug/dynamic_debug/control

# تفعيل بالـ module name (مثلاً driver بيستخدم property API)
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug الفعّالة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep property
```

##### من الـ bootloader (kernel cmdline):

```
dyndbg="file drivers/base/property.c +p; file drivers/base/swnode.c +p"
```

##### في الـ driver نفسه — استخدام `dev_dbg`:

```c
/* قبل تسجيل الـ software node */
dev_dbg(dev, "registering software node '%s'\n", node->name);

/* بعد قراءة property */
ret = device_property_read_u32(dev, "clock-frequency", &freq);
dev_dbg(dev, "clock-frequency=%u ret=%d\n", freq, ret);
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config | الوظيفة |
|---|---|
| `CONFIG_OF` | يفعّل دعم Device Tree |
| `CONFIG_OF_DYNAMIC` | يسمح بـ DT overlays ديناميكية |
| `CONFIG_OF_OVERLAY` | دعم DT overlays كاملاً |
| `CONFIG_ACPI` | يفعّل ACPI firmware node |
| `CONFIG_DEBUG_FS` | مطلوب للـ debugfs entries |
| `CONFIG_TRACING` | مطلوب للـ ftrace |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg` / dynamic debug |
| `CONFIG_DEBUG_DEVRES` | يتتبع leaks في الـ managed resources |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ fwnode locking |
| `CONFIG_DEBUG_OBJECTS` | يكشف use-after-free في الـ fwnode objects |
| `CONFIG_KASAN` | يكشف memory corruption في الـ property arrays |
| `CONFIG_KCSAN` | يكشف race conditions على الـ fwnode flags |
| `CONFIG_BOOT_PRINTK_DELAY` | مفيد لو المشكلة بتحصل early في الـ boot |
| `CONFIG_FW_DEVLINK` | يتحكم في الـ device link behavior |

---

#### 6. أدوات خاصة بالـ Subsystem

##### فحص الـ Device Tree من userspace:

```bash
# dtc — اعمل dump للـ DT الحي
dtc -I fs /sys/firmware/devicetree/base > /tmp/live.dts

# fdtdump — لو عندك binary DT file
fdtdump /boot/dtb/my-board.dtb | grep -A5 "my-sensor"

# شوف الـ DT properties بشكل human-readable
ls /sys/firmware/devicetree/base/soc/
hexdump -C /sys/firmware/devicetree/base/soc/my-sensor/reg
```

##### فحص الـ ACPI:

```bash
# dump ACPI tables
acpidump > /tmp/acpi.dat
acpixtract -a /tmp/acpi.dat
iasl -d DSDT.dat  # decompile إلى DSL

# شوف الـ ACPI device properties
cat /sys/bus/acpi/devices/*/status
```

##### فحص الـ software nodes:

```bash
# لو kernel بيدعم software_nodes في sysfs
ls /sys/firmware/software_nodes/
cat /sys/firmware/software_nodes/my-sensor/properties/speed-hz
```

##### فحص الـ fwnode connections:

```bash
# شوف graph connections (camera pipelines مثلاً)
media-ctl -p  # لو عندك V4L2 media controller
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `device_property_read_u32: -EINVAL` | الـ fwnode نفسه NULL أو مفيهوش ops | تأكد إن `dev_fwnode(dev)` مش NULL قبل الاستدعاء |
| `device_property_read_u32: -ENODATA` | الـ property موجودة بس فاضية | صحح الـ DTS أو الـ software_node |
| `device_property_read_u32: -ENXIO` | الـ fwnode مش بيدعم العملية دي | الـ fwnode_ops مش بيوفر `property_read_int_array` |
| `device_property_read_u32: -EOVERFLOW` | الـ array أكبر من الـ nval المطلوب | زوّد الـ buffer أو استخدم `_count` أولاً |
| `fwnode_handle_put: already freed` | double free على الـ fwnode | استخدم `__free(fwnode_handle)` أو اتأكد من lifecycle |
| `software_node_register: -EEXIST` | node باسم مسجل قبل كده | تحقق من uniqueness أو call `unregister` أولاً |
| `fwnode_graph_get_next_endpoint: NULL` | مفيش graph endpoints | تحقق إن الـ DTS فيه `port` و `endpoint` nodes صح |
| `device ... not ready, waiting for supplier` | الـ fw_devlink مش لاقي supplier | تحقق من الـ DT phandle أو سجّل الـ software node الـ supplier أولاً |
| `fwnode_property_present: fwnode is NULL` | بيتسأل على device مش متهيأ | تحقق من ترتيب الـ init وإن الـ of_node/acpi_node اتعيّن |
| `property_entries_dup: -ENOMEM` | نفد الـ memory | فحص memory leaks، قلل عدد الـ properties |
| `fwnode_connection_find_match: no match` | الـ connection ID مش موجود | تحقق من الـ con_id string في الـ DTS والـ driver |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* في drivers/base/property.c — عند قراءة property بـ fwnode مشبوه */
int fwnode_property_read_u32_array(const struct fwnode_handle *fwnode,
                                   const char *propname, u32 *val, size_t nval)
{
    /* WARN لو الـ fwnode وصلنا بـ NULL — دايماً علامة bug في الـ caller */
    if (WARN_ON(!fwnode))
        return -EINVAL;
    /* ... */
}

/* في device_add_software_node — لو الـ device خلاص عنده node */
int device_add_software_node(struct device *dev, const struct software_node *node)
{
    WARN_ON(dev_fwnode(dev) != NULL); /* مش المفروض نضيف على فوق node موجودة */
    /* ... */
}

/* في كود الـ driver — عند استخدام fwnode_for_each_child_node */
fwnode_for_each_child_node(fwnode, child) {
    if (unlikely(!child->ops)) {
        WARN_ON(1);  /* child بـ NULL ops — مشكلة خطيرة */
        break;
    }
}

/* عند break مبكر من loop — لازم تعمل put يدوي */
device_for_each_child_node(dev, child) {
    if (some_condition) {
        fwnode_handle_put(child);  /* mandatory — وإلا leak */
        break;
    }
}

/* نقطة مناسبة لـ dump_stack في driver init */
ret = device_property_read_u32(dev, "reg", &addr);
if (ret) {
    dev_err(dev, "failed to read 'reg': %d\n", ret);
    dump_stack();  /* مفيد لو الـ probe بيتسمى من مكان غير متوقع */
    return ret;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ hardware بتطابق حالة الـ kernel

```bash
# خطوة 1: اقرأ الـ property من الـ DTS/ACPI
cat /sys/firmware/devicetree/base/soc/spi@40013000/cs-gpios
# أو
hexdump -C /sys/firmware/devicetree/base/soc/spi@40013000/clock-frequency

# خطوة 2: قارن مع ما قرأه الـ driver فعلاً
dmesg | grep -i "spi\|clock-frequency\|cs-gpio"

# خطوة 3: تحقق من الـ gpio state الفعلي
cat /sys/kernel/debug/gpio  # شوف كل GPIO direction/value

# خطوة 4: تحقق من الـ clock فعلاً شغّال
cat /sys/kernel/debug/clk/clk_summary | grep spi
```

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 — اقرأ/اكتب registers بشكل مباشر
# مثال: اقرأ SPI base address من DTS ثم تحقق منه
SPI_BASE=$(cat /sys/firmware/devicetree/base/soc/spi@40013000/reg | \
           od -An -tx4 | awk '{print "0x"$1}')

devmem2 $SPI_BASE    # قرأ CR1
devmem2 $((SPI_BASE + 4))  # CR2
devmem2 $((SPI_BASE + 8))  # SR

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x40013000/4)) 2>/dev/null | od -tx4

# io utility (من package ioport)
io -4 -r 0x40013000

# في kernel module — ioread/iowrite
void __iomem *base = fwnode_iomap(fwnode, 0);
pr_info("CR1=0x%08x SR=0x%08x\n",
        ioread32(base),
        ioread32(base + 0x08));
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

```
مشكلة شائعة: الـ driver قرأ "spi-max-frequency = <10000000>" من الـ DTS
              لكن الـ bus شغّال بـ 1MHz فعلاً.

خطوة 1: وصّل الـ logic analyzer على SCK
خطوة 2: ابعت transaction بسيط
خطوة 3: قيس الـ frequency الفعلية

لو مختلفة → ابحث عن:
  a) الـ clock divider في الـ parent clock مش صح في DTS
  b) الـ SoC datasheet بيقول max = 5MHz بس الـ DTS قال 10MHz
  c) الـ driver مش بيحترم fwnode_get_phy_mode() وبيستخدم قيمة default

لـ I2C:
  - تأكد من الـ pull-up resistors قيمتهم تتوافق مع "clock-frequency"
  - 100kHz → 4.7kΩ, 400kHz → 2.2kΩ, 1MHz → 1kΩ

لـ GPIO:
  - استخدم oscilloscope على الـ CS pin
  - تحقق إن polarity في DTS (active-low) بتوافق الـ hardware
```

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| مشكلة الـ HW | النمط في الـ dmesg | السبب |
|---|---|---|
| غلط في الـ IRQ number في DTS | `irq: no irq domain found for /soc/foo@0` | الـ interrupt-parent مش صح |
| الـ GPIO controller مش موجود | `gpio request failed: -EPROBE_DEFER` | الـ gpiochip لسه مش registered |
| clock parent غلط | `clk: failed to reparent` | الـ clocks property في DTS غلط |
| regulator voltage مش صح | `regulator: invalid voltage range` | `voltage-ranges` في DTS مش بيطابق الـ HW spec |
| مشكلة في الـ pinmux | `pinctrl: request pin failed` | الـ pinctrl-0 في DTS بيتعارض مع device تاني |
| الـ MMIO address غلط | `ioremap failed for 0x...` | الـ reg property في DTS مش بيطابق الـ SoC memory map |
| DMA channel مش متاح | `dma: no DMA channel available` | الـ dmas property فيها index غلط |

---

#### 5. تصحيح مشاكل الـ Device Tree

```bash
# الخطوة الأولى: تحقق إن الـ DT اتحمل صح
dmesg | grep -E "OF|DTB|devicetree"
# المفروض تشوف: "OF: fdt: Machine model: My Board Rev 1.0"

# قارن الـ DT المُفسَّر بالـ source
dtc -I fs /sys/firmware/devicetree/base -O dts > /tmp/live.dts
diff /tmp/live.dts my-board.dts

# تحقق من وجود property معينة
grep -r "my-sensor" /sys/firmware/devicetree/base/ 2>/dev/null

# فحص الـ phandles — لو فيه reference لـ node مش موجودة
# ظهور -ENODEV أو NULL fwnode من fwnode_find_reference()
# الحل: تأكد إن الـ node المرجوع إليها موجودة فعلاً في الـ DTS

# تحقق من الـ status property
cat /sys/firmware/devicetree/base/soc/my-device/status
# يجب أن يكون: "okay" مش "disabled"

# فحص الـ compatible string بدقة (case-sensitive, dash-sensitive)
cat /sys/firmware/devicetree/base/soc/my-device/compatible | strings
# مقارنة مع ما في الـ driver's of_match_table

# فحص الـ DT overlay بعد تطبيقه
cat /sys/kernel/config/device-tree/overlays/my-overlay/status
```

---

### Practical Commands

#### Section A: فحص سريع للـ property subsystem

```bash
#!/bin/bash
# === property_debug.sh === فحص شامل وسريع ===

DEVICE=${1:-"my-device"}

echo "=== [1] Device fwnode type ==="
ls -la /sys/bus/platform/devices/$DEVICE/of_node 2>/dev/null && echo "OF (DT)" \
  || ls -la /sys/bus/platform/devices/$DEVICE/firmware_node 2>/dev/null && echo "ACPI" \
  || echo "Software Node or Unknown"

echo ""
echo "=== [2] DT properties (if OF) ==="
ls /sys/firmware/devicetree/base/**/$DEVICE/ 2>/dev/null | head -20

echo ""
echo "=== [3] Deferred probe reason ==="
grep $DEVICE /sys/kernel/debug/devices_deferred 2>/dev/null || echo "Not deferred"

echo ""
echo "=== [4] Recent kernel messages ==="
dmesg --since "5 minutes ago" | grep -i "$DEVICE\|fwnode\|swnode\|property" | tail -30

echo ""
echo "=== [5] Software nodes ==="
ls /sys/kernel/debug/software_nodes/ 2>/dev/null || echo "No software_nodes debugfs"

echo ""
echo "=== [6] GPIO state ==="
grep $DEVICE /sys/kernel/debug/gpio 2>/dev/null || echo "No GPIO for $DEVICE"
```

##### تشغيل:
```bash
bash property_debug.sh spi-0
```

---

#### Section B: تفعيل الـ tracing لـ property reads

```bash
# === تفعيل شامل ===
cd /sys/kernel/debug/tracing

# صفّر الـ trace
echo 0 > tracing_on
echo > trace

# اختر الـ tracer
echo function > current_tracer

# filter على دوال الـ property API
cat > set_ftrace_filter << 'EOF'
device_property_read_u32_array
device_property_read_string
fwnode_property_present
fwnode_property_read_u32_array
fwnode_device_is_available
software_node_register
software_node_unregister
device_add_software_node
device_remove_software_node
fwnode_graph_get_next_endpoint
fwnode_graph_parse_endpoint
EOF

echo 1 > tracing_on
# trigger the driver probe
echo "my_driver" > /sys/bus/platform/drivers/my_driver/bind 2>/dev/null
sleep 2
echo 0 > tracing_on

# اقرأ النتيجة
cat trace | head -100
```

---

#### Section C: فحص graph endpoints

```bash
# شوف كل الـ endpoints في الـ system
find /sys/firmware/devicetree/base -name "endpoint*" -type d 2>/dev/null

# تحقق من remote-endpoint references
find /sys/firmware/devicetree/base -name "remote-endpoint" | while read f; do
    echo "=== $f ==="
    cat "$f" | od -tx4  # الـ phandle value
done

# لو عندك V4L2 / media devices
media-ctl --print-topology 2>/dev/null
v4l2-ctl --list-devices 2>/dev/null
```

---

#### Section D: فحص مشاكل الـ fw_devlink

```bash
# شوف الـ devlink strict mode
cat /proc/sys/kernel/fw_devlink

# مؤقتاً عطّل الـ strict devlink لـ debugging
echo "permissive" > /proc/sys/kernel/fw_devlink

# تحقق من الـ suppliers
cat /sys/kernel/debug/devices_deferred

# شوف links لـ device معين
ls /sys/bus/platform/devices/my-device/
# consumer_links  supplier_links  ...

ls /sys/bus/platform/devices/my-device/consumer_links/
ls /sys/bus/platform/devices/my-device/supplier_links/
```

---

#### Section E: مثال output وتفسيره

```
# مثال: dmesg بعد تفعيل dynamic_debug على swnode.c

[    2.341234] swnode: software_node_register: registering node 'mpu6050'
[    2.341235] swnode: swnode_property_read_int_array: 'i2c-addr' 1 elements
[    2.341310] swnode: software_node_register: registering node 'bmp280'

# تفسير:
# - 'mpu6050' اتسجّل كـ software node
# - الـ driver قرأ property 'i2c-addr' وطلب element واحد (u32)
# - 'bmp280' اتسجّل بعديه

# مثال: مشكلة probe defer
[    3.452100] i2c i2c-1: Driver i2c-mpu6050 is waiting for supplier /soc/i2c@40005400

# تفسير:
# الـ fw_devlink شاف إن 'i2c-1' محتاجة supplier هو /soc/i2c@40005400
# الـ supplier ده لسه مش bound → probe اتأجّل
# الحل: تأكد إن الـ parent I2C controller driver اتحمل أولاً
# أو تحقق إن الـ phandle في DTS صح

# مثال: property مش موجودة
[    3.891234] my_driver my-device: device_property_read_u32 'clock-frequency' failed: -EINVAL
# تفسير: الـ fwnode نفسه NULL → مش بس الـ property مش موجودة
# الحل: تحقق من of_node أو acpi_node على الـ device قبل الـ probe

# مثال صح:
[    3.891300] my_driver my-device: clock-frequency = 1000000 Hz
[    3.891301] my_driver my-device: probe success
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — SPI لا بيشتغل بعد bring-up

#### العنوان
**الـ SPI controller مش بيحصل probe على industrial gateway بـ RK3562**

#### السياق
فريق بيعمل bring-up لـ industrial gateway مبني على RK3562. البورد بيحتوي على SPI flash خارجي متوصل بـ SPI2. الـ DTS موجود، الـ kernel بيبوت، لكن الـ SPI device مش ظاهر في `/sys/bus/spi/devices/`.

#### المشكلة
الدرايفر بيعمل `device_property_read_u32(dev, "num-cs", &num_cs)` ورجع `-EINVAL`. الـ probe اتوقف ورجع error.

#### التحليل
**الـ `device_property_read_u32`** في `property.h` هو inline wrapper:

```c
static inline int device_property_read_u32(const struct device *dev,
                                           const char *propname, u32 *val)
{
    /* بتقرأ عنصر واحد من array size=1 */
    return device_property_read_u32_array(dev, propname, val, 1);
}
```

**الـ `device_property_read_u32_array`** بتروح لـ `fwnode_property_read_u32_array` عبر `dev_fwnode(dev)`:

```c
#define dev_fwnode(dev)                         \
    _Generic((dev),                             \
             const struct device *: __dev_fwnode_const,  \
             struct device *: __dev_fwnode)(dev)
```

الـ `_Generic` بيختار الـ const-correct version تلقائياً. المشكلة إن الـ `fwnode_handle` للـ device كان `NULL` — يعني الـ DTS node مش متوصل صح بالـ device.

التحقيق كشف إن اسم الـ node في DTS كان `spi@fe610000` لكن الـ `compatible` كان `"rockchip,rk3562-spi"` وفي نفس الوقت الـ `status` كان `"disabled"` — فـ `fwnode_device_is_available()` رجع `false` والـ framework ما ربطش الـ fwnode بالـ device.

```c
/* property.h */
bool fwnode_device_is_available(const struct fwnode_handle *fwnode);
```

#### الحل

```bash
# تحقق من الـ fwnode
cat /sys/bus/platform/devices/fe610000.spi/of_node/status
# Output: disabled  <-- هنا المشكلة

# أو عبر debugfs
grep -r "fe610000" /sys/firmware/devicetree/base/ 2>/dev/null
```

```dts
/* الإصلاح في DTS */
&spi2 {
    status = "okay";   /* كانت "disabled" */
    num-cs = <1>;
    pinctrl-names = "default";
    pinctrl-0 = <&spi2m0_pins>;
};
```

#### الدرس المستفاد
**الـ `fwnode_device_is_available()`** بيتحقق من property `"status"` قبل ما أي `device_property_*` يشتغل. لو الـ node `disabled`، الـ fwnode مش بيتوصل بالـ device أصلاً، وكل قراءة property بترجع `-EINVAL`. دايماً افحص `status = "okay"` أول حاجة في الـ bring-up.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش بيشوف الشاشة

#### العنوان
**الـ HDMI output مش بيشتغل على Android TV box بسبب خطأ في graph endpoint**

#### السياق
منتج Android TV box مبني على Allwinner H616. الـ display pipeline: `DE2 → TCON → HDMI`. الـ kernel بيبوت والـ HDMI كودك موجود، لكن الشاشة فاضية.

#### المشكلة
الدرايفر بتاع HDMI بيعمل `fwnode_graph_get_remote_endpoint()` عشان يلاقي الـ TCON port، لكن الـ function بترجع `NULL`. الـ display pipeline مش متوصل.

#### التحليل
الـ API المستخدم من `property.h`:

```c
struct fwnode_handle *fwnode_graph_get_remote_endpoint(
    const struct fwnode_handle *fwnode);

struct fwnode_handle *fwnode_graph_get_next_endpoint(
    const struct fwnode_handle *fwnode, struct fwnode_handle *prev);
```

والـ macro للـ iteration:

```c
#define fwnode_graph_for_each_endpoint(fwnode, child)              \
    for (child = fwnode_graph_get_next_endpoint(fwnode, NULL); child; \
         child = fwnode_graph_get_next_endpoint(fwnode, child))
```

الـ `fwnode_graph_is_endpoint()` بتتحقق عبر:

```c
static inline bool fwnode_graph_is_endpoint(const struct fwnode_handle *fwnode)
{
    /* بتدور على property اسمها "remote-endpoint" */
    return fwnode_property_present(fwnode, "remote-endpoint");
}
```

المشكلة: في الـ DTS، الـ `remote-endpoint` في node الـ HDMI بيشاور على `phandle` غلط — بيشاور على الـ port node مش على الـ endpoint node جوّاه.

```dts
/* غلط */
hdmi_in: port {
    hdmi_in_tcon: endpoint {
        remote-endpoint = <&tcon_out>;  /* بيشاور على port مش endpoint */
    };
};

/* صح */
hdmi_in: port {
    hdmi_in_tcon: endpoint {
        remote-endpoint = <&tcon_out_hdmi>;  /* endpoint-to-endpoint */
    };
};
```

#### الحل

```bash
# افحص الـ graph links
find /sys/firmware/devicetree/base -name "remote-endpoint" | while read f; do
    echo "=== $f ==="; cat "$f" | xxd | head -2
done

# استخدم fwnode_graph_parse_endpoint للـ debug
# في kernel driver مؤقت:
struct fwnode_endpoint ep;
fwnode_graph_parse_endpoint(endpoint_fwnode, &ep);
pr_info("port=%u id=%u\n", ep.port, ep.id);
```

```dts
/* الإصلاح الكامل */
&tcon0 {
    ports {
        tcon_out: port@1 {
            reg = <1>;
            tcon_out_hdmi: endpoint@1 {
                reg = <1>;
                remote-endpoint = <&hdmi_in_tcon>;
            };
        };
    };
};

&hdmi {
    ports {
        hdmi_in: port@0 {
            reg = <0>;
            hdmi_in_tcon: endpoint@0 {
                reg = <0>;
                remote-endpoint = <&tcon_out_hdmi>;
            };
        };
    };
};
```

#### الدرس المستفاد
**الـ `fwnode_graph_*` API** بيشتغل على مستوى endpoint-to-endpoint مش port-to-port. الـ `remote-endpoint` لازم يشاور على الـ `endpoint` node نفسه. استخدم `fwnode_graph_parse_endpoint()` لتأكيد إن الـ port/id صح.

---

### السيناريو 3: IoT Sensor على STM32MP1 — قراءة I2C sensor بترجع قيم غلط

#### العنوان
**درايفر I2C sensor على STM32MP1 بيقرأ endianness غلط**

#### السياق
board بتشغّل IoT environmental sensor (temperature + pressure) متوصل بـ I2C على STM32MP1. الـ sensor بيرجع big-endian data. الدرايفر بيقرأ القيم لكن الأرقام معكوسة تماماً.

#### المشكلة
الدرايفر بيستخدم `readl/writel` العادية (little-endian) بدل `ioread32be`. المطور ما كانش عارف إن الـ device DTS عنده `big-endian` property.

#### التحليل
**الـ `device_is_big_endian()`** في `property.h`:

```c
static inline bool device_is_big_endian(const struct device *dev)
{
    /* بتفوّض لـ fwnode version */
    return fwnode_device_is_big_endian(dev_fwnode(dev));
}

static inline bool fwnode_device_is_big_endian(const struct fwnode_handle *fwnode)
{
    /* الحالة 1: property صريحة "big-endian" */
    if (fwnode_property_present(fwnode, "big-endian"))
        return true;
    /* الحالة 2: kernel compiled as BE + device له "native-endian" */
    if (IS_ENABLED(CONFIG_CPU_BIG_ENDIAN) &&
        fwnode_property_present(fwnode, "native-endian"))
        return true;
    return false;
}
```

الـ DTS كان صح:
```dts
&i2c2 {
    sensor@76 {
        compatible = "bosch,bmp280";
        reg = <0x76>;
        big-endian;   /* property موجودة */
    };
};
```

لكن الدرايفر ما كانش بيسأل:

```c
/* الكود الغلط */
static int bmp280_read_raw(struct iio_dev *indio_dev, ...)
{
    /* ما بيتحققش من endianness */
    *val = readl(dev->regs + BMP280_REG_DATA);
}
```

#### الحل

```c
/* الكود الصح */
static int bmp280_probe(struct i2c_client *client)
{
    struct bmp280_data *data = ...;

    /* بنسأل property.h API */
    data->big_endian = device_is_big_endian(&client->dev);

    dev_info(&client->dev, "endian mode: %s\n",
             data->big_endian ? "big" : "little");
}

static int bmp280_read_raw(struct iio_dev *indio_dev, ...)
{
    u32 raw;
    if (data->big_endian)
        raw = ioread32be(data->regs + BMP280_REG_DATA);
    else
        raw = ioread32(data->regs + BMP280_REG_DATA);
    *val = raw;
}
```

```bash
# تحقق من الـ property
cat /sys/firmware/devicetree/base/soc/i2c@5c002000/sensor@76/big-endian
# لو الملف موجود (حتى لو فاضي) = property موجودة = big-endian
```

#### الدرس المستفاد
**`device_is_big_endian()`** بيشيل عنك كل التعقيد: big-endian explicit أو native-endian على BE kernel. دايماً استخدمها في بداية الـ probe واحفظ النتيجة في driver state.

---

### السيناريو 4: Automotive ECU على i.MX8 — software_node لـ UART بدون DTS

#### العنوان
**إضافة UART configuration لـ automotive ECU بـ i.MX8 بدون تعديل DTS**

#### السياق
automotive ECU مبني على i.MX8QM. الـ UART3 متوصل بـ LIN transceiver. الـ board manufacturer ما وفّرش DTS source، بس الـ binary DTB موجود. الفريق محتاج يضيف properties إضافية للـ UART3 driver بدون إعادة compile الـ DTB.

#### المشكلة
الـ LIN transceiver driver محتاج property اسمها `"lin-mode"` بـ value `<1>` و`"transceiver-speed"` بـ value `<19200>`. مش موجودين في الـ DTB.

#### التحليل
**الـ Software Node API** في `property.h` بيحل المشكلة دي بالظبط:

```c
/* struct software_node */
struct software_node {
    const char *name;
    const struct software_node *parent;
    const struct property_entry *properties;  /* array of properties */
};

/* device_add_software_node */
int device_add_software_node(struct device *dev,
                             const struct software_node *node);

/* device_create_managed_software_node */
int device_create_managed_software_node(struct device *dev,
                                        const struct property_entry *properties,
                                        const struct software_node *parent);
```

الـ `property_entry` macros بتبني الـ properties بشكل static:

```c
/* PROPERTY_ENTRY_U32 بيبني inline property */
#define PROPERTY_ENTRY_U32(_name_, _val_)   \
    __PROPERTY_ENTRY_ELEMENT(_name_, u32_data, U32, _val_)

/* __PROPERTY_ENTRY_ELEMENT بيضع القيمة inline في value union */
#define __PROPERTY_ENTRY_ELEMENT(_name_, _elem_, _Type_, _val_)  \
(struct property_entry) {                                         \
    .name = _name_,                                               \
    .length = sizeof_field(struct property_entry, value._elem_[0]), \
    .is_inline = true,                                            \
    .type = DEV_PROP_##_Type_,                                    \
    { .value = { ._elem_[0] = _val_ } },                         \
}
```

#### الحل

```c
/* في driver أو platform init code */
#include <linux/property.h>

static const struct property_entry lin_uart_props[] = {
    PROPERTY_ENTRY_U32("lin-mode", 1),
    PROPERTY_ENTRY_U32("transceiver-speed", 19200),
    PROPERTY_ENTRY_STRING("transceiver-type", "tja1021"),
    PROPERTY_ENTRY_BOOL("lin-checksum-enhanced"),
    { }  /* sentinel */
};

static int imx8_lin_attach(struct platform_device *pdev)
{
    struct device *uart_dev;
    int ret;

    /* نلاقي الـ UART device */
    uart_dev = bus_find_device_by_name(&platform_bus_type,
                                       NULL, "30890000.serial");
    if (!uart_dev)
        return -ENODEV;

    /* بنضيف software node فوق الـ DT node الموجود */
    ret = device_create_managed_software_node(uart_dev,
                                              lin_uart_props, NULL);
    if (ret) {
        dev_err(uart_dev, "failed to add sw node: %d\n", ret);
        put_device(uart_dev);
        return ret;
    }

    /* دلوقتي الـ LIN driver يقدر يقرأ */
    u32 speed;
    if (!device_property_read_u32(uart_dev, "transceiver-speed", &speed))
        dev_info(uart_dev, "LIN speed: %u bps\n", speed);

    put_device(uart_dev);
    return 0;
}
```

```bash
# التحقق من الـ software node
ls /sys/bus/platform/devices/30890000.serial/software_node/
# properties  name  ...

cat /sys/bus/platform/devices/30890000.serial/software_node/properties/lin-mode
```

#### الدرس المستفاد
**الـ software node API** (`device_add_software_node`, `device_create_managed_software_node`) بيخليك تضيف أو تعدّل device properties في runtime بدون مس الـ DTB. مثالي لـ closed-source BSP أو automotive platforms اللي الـ DTS مش متاح. الـ **managed** version بتعمل cleanup أوتوماتيك مع device lifecycle.

---

### السيناريو 5: Custom Board Bring-up على AM62x — memory leak في driver بسبب fwnode reference

#### العنوان
**Memory leak في custom sensor hub driver على AM62x بسبب نسيان `fwnode_handle_put()`**

#### السياق
فريق بيكتب driver لـ custom sensor hub board مبني على AM62x (Texas Instruments). الـ hub بيضم 4 sensors متوصلين بـ I2C، كل sensor عنده child node في DTS. الـ driver بيدور على الـ child nodes ويـ configure كل sensor.

#### المشكلة
بعد شهور من production، الـ system بدأ يحتاج restart دوري. الـ kmemleak بيرجع objects من نوع `fwnode_handle` مش بتتحرر. الـ driver بيعمل `device_for_each_child_node` ويعمل `break` جوّا الـ loop من غير ما يعمل `fwnode_handle_put`.

#### التحليل
الـ API في `property.h` واضح في موضوع ownership:

```c
/* fwnode_handle_put - Drop reference */
static inline void fwnode_handle_put(struct fwnode_handle *fwnode)
{
    fwnode_call_void_op(fwnode, put);
}

/* DEFINE_FREE بيسمح بـ __free(fwnode_handle) scoped cleanup */
DEFINE_FREE(fwnode_handle, struct fwnode_handle *, fwnode_handle_put(_T))
```

الكود المسرّب كان هكذا:

```c
/* كود غلط - leak عند break */
static int hub_configure_sensors(struct device *dev)
{
    struct fwnode_handle *child;
    int ret = 0;

    device_for_each_child_node(dev, child) {
        u32 sensor_id;
        if (device_property_read_u32(dev, "sensor-id",
                                     &sensor_id))
            continue;

        if (sensor_id == SPECIAL_SENSOR) {
            /* break بدون put = leak! */
            ret = configure_special(dev, child);
            break;
        }
    }
    return ret;
}
```

#### الحل: الطريقة الأولى — manual put

```c
/* الإصلاح اليدوي */
static int hub_configure_sensors(struct device *dev)
{
    struct fwnode_handle *child;
    int ret = 0;

    device_for_each_child_node(dev, child) {
        u32 sensor_id;
        if (fwnode_property_read_u32(child, "sensor-id",
                                     &sensor_id))
            continue;

        if (sensor_id == SPECIAL_SENSOR) {
            ret = configure_special(dev, child);
            /* لازم نعمل put قبل break */
            fwnode_handle_put(child);
            break;
        }
    }
    return ret;
}
```

#### الحل: الطريقة الثانية — scoped macro (الأفضل)

```c
/* الإصلاح بـ scoped version - cleanup أوتوماتيكي */
static int hub_configure_sensors(struct device *dev)
{
    int ret = 0;

    /* __free(fwnode_handle) بيعمل put تلقائياً عند خروج الـ scope */
    device_for_each_child_node_scoped(dev, child) {
        u32 sensor_id;
        if (fwnode_property_read_u32(child, "sensor-id",
                                     &sensor_id))
            continue;

        if (sensor_id == SPECIAL_SENSOR) {
            ret = configure_special(dev, child);
            break;  /* الـ __free(fwnode_handle) بيتكلم بدالنا */
        }
    }
    return ret;
}
```

```bash
# اكتشاف الـ leak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep -A5 "fwnode"

# أو بـ kasan
CONFIG_KASAN=y
# في dmesg بتلاقي stack trace للـ allocation بدون free
```

#### الدرس المستفاد
كل `fwnode_handle` بترجعه `device_for_each_child_node` أو `fwnode_get_next_child_node` عنده refcount. لو بتعمل `break` أو `return` من جوّا الـ loop، **لازم** تعمل `fwnode_handle_put()` قبلها. الحل الأسهل والأأمن هو استخدام **`_scoped`** versions من الـ macros (`device_for_each_child_node_scoped`, `fwnode_for_each_available_child_node_scoped`) اللي بتستخدم `DEFINE_FREE` وبتعمل cleanup أوتوماتيكي.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ unified device property framework بشكل مباشر:

| المقال | الأهمية |
|--------|---------|
| [Add ACPI _DSD and unified device properties support](https://lwn.net/Articles/612062/) | الـ patch series الأصلية اللي قدمت الـ `property.h` API — Rafael Wysocki + Mika Westerberg |
| [device property: Introducing software nodes](https://lwn.net/Articles/770825/) | إزاي اتغيرت الـ `struct property_set` لـ `software_node` المستقلة |
| [Software fwnode references](https://lwn.net/Articles/789099/) | إضافة دعم الـ `software_node_ref_args` وعمل `fwnode_property_get_reference_args()` مع الـ software nodes |
| [ACPI graph support](https://lwn.net/Articles/718184/) | إزاي اتضافت الـ port/endpoint graph بالـ `_DSD` في ACPI — نفس الـ `fwnode_graph_*` API |
| [Unified fwnode endpoint parser](https://lwn.net/Articles/734838/) | توحيد الـ endpoint parser بين DT وACPI باستخدام `fwnode_graph_parse_endpoint()` |
| [Dynamic DT device nodes](https://lwn.net/Articles/872164/) | الـ software nodes كبديل للـ DT الديناميكي |
| [introduce fwnode in the I2C subsystem](https://lwn.net/Articles/889236/) | مثال عملي على استخدام الـ `fwnode_handle` في subsystem حقيقي |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في شجرة الـ kernel تحت `Documentation/`:

```
Documentation/firmware-guide/acpi/
├── dsd/
│   ├── graph.rst          # الـ fwnode graph API مع ACPI _DSD
│   └── leds.rst           # مثال على properties مع LEDs
├── enumeration.rst        # ACPI-based device enumeration
└── index.rst

Documentation/devicetree/
├── usage-model.rst        # إزاي الـ DT properties بتتقرأ
└── kernel-api.rst         # OF API مقارنةً بـ fwnode API
```

**الروابط المباشرة:**
- [Linux and the Devicetree — Usage Model](https://www.kernel.org/doc/html/latest/devicetree/usage-model.html)
- [DeviceTree Kernel API](https://docs.kernel.org/devicetree/kernel-api.html)
- [Graphs — ACPI DSD](https://www.kernel.org/doc/html/v5.2/firmware-guide/acpi/dsd/graph.html)
- [ACPI Based Device Enumeration](https://docs.kernel.org/firmware-guide/acpi/enumeration.html)
- [_DSD Device Properties Related to GPIO](https://static.lwn.net/kerneldoc/firmware-guide/acpi/gpio-properties.html)

---

### الملفات المصدرية الأساسية في الـ Kernel

الـ API اللي في `property.h` بتتنفذ في:

```
include/linux/property.h        ← الـ header الرئيسي (الملف ده)
include/linux/fwnode.h          ← تعريف struct fwnode_handle و ops
drivers/base/property.c         ← تنفيذ device_property_*
drivers/base/swnode.c           ← تنفيذ software_node_*
drivers/base/core.c             ← ربط الـ fwnode بالـ struct device
```

---

### Commits المهمة

الـ commits دي شكّلت الـ API كما هو دلوقتي:

| الوصف | المرجع |
|-------|--------|
| الإضافة الأولى للـ unified property API (kernel 3.19) | [patch series على narkive](https://linux.kernel.narkive.com/pNOj9vdt/patch-driver-core-implement-device-property-accessors-through-fwnode-ones) |
| إضافة `fwnode_device_is_available()` | [Patchwork](https://patchwork.kernel.org/project/linux-acpi/patch/1496741861-8240-5-git-send-email-sakari.ailus@linux.intel.com/) |
| تحويل `dev_fwnode()` لـ public | [stable-commits](https://www.spinics.net/lists/stable-commits/msg70095.html) |
| fallback لـ secondary fwnode | [narkive patch](https://linux.kernel.narkive.com/XO6y722J/patch-v1-08-13-device-property-fallback-to-secondary-fwnode-if-primary-misses-the-property) |

**الـ kernel.org git log الأساسي:**
```bash
# مشاهدة تاريخ الملف
git log --follow include/linux/property.h

# أول commit للملف
git log --follow --diff-filter=A -- include/linux/property.h
```

---

### نقاشات Mailing List

- [LKML: fwnode_device_is_available() — Sakari Ailus](https://patchwork.kernel.org/project/linux-acpi/patch/1496741861-8240-5-git-send-email-sakari.ailus@linux.intel.com/)
- [LKML: device property fallback to secondary fwnode](https://linux.kernel.narkive.com/XO6y722J/patch-v1-08-13-device-property-fallback-to-secondary-fwnode-if-primary-misses-the-property)
- [Unified Device Properties Interface slides — LinuxCon](https://events.static.linuxfound.org/sites/events/files/slides/unified_properties_API_0.pdf) — عرض Rafael Wysocki الأصلي للـ API

---

### كتب مُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model
  - بيشرح الـ `struct device`، الـ `kobject`، والـ sysfs
  - الـ `property.h` API بتبني فوق الـ device model ده
- متاح مجانًا: [lwn.net/Kernel/LDD2](https://lwn.net/Kernel/LDD2/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 17**: Devices and Modules
  - الـ device model، الـ bus، الـ driver registration
  - السياق اللي بيتشتغل فيه الـ `device_property_*`

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 16**: Kernel Initialization
- **الفصل 17**: Device Drivers
  - بيشرح الـ Device Tree properties في سياق الـ embedded systems
  - قريب جدًا من الاستخدام الفعلي للـ `fwnode` على ARM

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 6**: Device Drivers
  - coverage كامل للـ device model وكيفية ربط الـ firmware descriptions

---

### مراجع eLinux.org

- [Device Tree Reference](https://elinux.org/Device_Tree_Reference) — مرجع شامل للـ DT properties اللي بيوصل ليها `fwnode_property_read_*`
- [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) — شرح للـ `#xxx-cells` properties اللي بتتعامل معاها `fwnode_property_get_reference_args()`

---

### مراجع KernelNewbies.org

| إصدار | التغيير المهم |
|-------|--------------|
| [Linux 3.14](https://kernelnewbies.org/Linux_3.14) | أول إضافة للـ unified device property API |
| [Linux 5.6](https://kernelnewbies.org/Linux_5.6) | تحديثات الـ device property framework |
| [Linux 6.1](https://kernelnewbies.org/Linux_6.1) | دعم firmware properties في input drivers |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | تحديثات wireless drivers لاستخدام `fwnode` |
| [Linux 6.17](https://kernelnewbies.org/Linux_6.17) | Rust bindings لـ `device_property_read_*` |

---

### مصطلحات البحث

لو عايز تعرف أكتر، استخدم الكلمات دي في البحث:

```
# للـ API العامة
"unified device property" linux kernel
"fwnode_handle" "property.h" linux
"device_property_read" ACPI "device tree"

# للـ software nodes
"software_node" linux kernel driver
"property_entry" "PROPERTY_ENTRY_U32" driver example

# للـ graph API
"fwnode_graph" port endpoint linux
"fwnode_graph_get_next_endpoint" v4l2

# للـ ACPI side
"ACPI _DSD" device properties linux
"acpi_device_data" fwnode

# لمناقشات الـ mailing list
site:lkml.org "property.h" fwnode "software_node"
site:patchwork.kernel.org "device property"
```

---

### ملخص خريطة الـ Subsystems

```
property.h
    │
    ├── Device Tree (OF)
    │       └── drivers/of/property.c
    │           Documentation/devicetree/
    │
    ├── ACPI
    │       └── drivers/acpi/property.c
    │           Documentation/firmware-guide/acpi/
    │
    └── Software Nodes
            └── drivers/base/swnode.c
                (لما الـ firmware description مش موجودة)
```
## Phase 8: Writing simple module

### الهدف

هنعمل module بيستخدم **kprobe** على الـ `device_property_read_u32_array` — دي من أكتر functions الـ property API استخداماً، بيتنادى عليها من drivers زي I2C و SPI و platform devices عشان يقرأ قيم زي clock-frequency أو reg أو interrupt numbers. الـ hook هيطبع اسم الـ device والـ property اللي اتطلبت وعدد العناصر المطلوبة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * prop_probe.c — kprobe on device_property_read_u32_array()
 *
 * Logs every call: which device asked for which u32 property and how many
 * elements were requested.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit          */
#include <linux/kernel.h>      /* pr_info, pr_err                            */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, ...        */
#include <linux/device.h>      /* struct device, dev_name()                  */
#include <linux/property.h>    /* prototype of the probed function           */

/* ------------------------------------------------------------------ */
/* kprobe callback — called just before device_property_read_u32_array */
/* ------------------------------------------------------------------ */

/*
 * Function signature we are probing:
 *   int device_property_read_u32_array(const struct device *dev,
 *                                      const char *propname,
 *                                      u32 *val, size_t nval);
 *
 * On x86-64 the arguments land in registers:
 *   regs->di = dev   (arg0)
 *   regs->si = propname (arg1)
 *   regs->dx = val   (arg2)
 *   regs->cx = nval  (arg3)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Cast register values back to their original C types */
    const struct device *dev      = (const struct device *)regs->di;
    const char          *propname = (const char *)regs->si;
    size_t               nval     = (size_t)regs->cx;

    /* nval == 0 means the caller just wants to know the array length */
    if (nval == 0) {
        pr_info("prop_probe: [count-query] dev=%s prop=\"%s\"\n",
                dev ? dev_name(dev) : "(null)",
                propname ? propname : "(null)");
    } else {
        pr_info("prop_probe: dev=%s prop=\"%s\" nval=%zu\n",
                dev ? dev_name(dev) : "(null)",
                propname ? propname : "(null)",
                nval);
    }

    return 0; /* 0 = let the real function run normally */
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    .symbol_name = "device_property_read_u32_array",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                           */
/* ------------------------------------------------------------------ */

static int __init prop_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("prop_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("prop_probe: planted kprobe on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit prop_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("prop_probe: kprobe removed\n");
}

module_init(prop_probe_init);
module_exit(prop_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe demo: trace device_property_read_u32_array calls");
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/device.h>
#include <linux/property.h>
```

الـ `kprobes.h` بيجيب `struct kprobe` وكل الـ API الخاص بالـ hooking. الـ `device.h` محتاجه عشان `dev_name()` تشتغل صح. الـ `property.h` مش إلزامي للتشغيل لكن بيوضح أن الـ prototype اللي بنـ probe عليه موجود في الـ build.

#### الـ `handler_pre` والـ arguments

**الـ `pt_regs`** بيحمل state الـ CPU لحظة الـ probe. على x86-64، الـ calling convention بيحط أول 4 arguments في `rdi, rsi, rdx, rcx` — فبنقرأهم من `regs->di` وما بعده مباشرةً من غير ما نغير أي حاجة في الـ stack.

الـ check على `nval == 0` مهم لأن الـ callers بيبعتوا صفر عمداً عشان يستعلموا عن حجم الـ array الأول — ده pattern موجود في كل الـ device property API.

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "device_property_read_u32_array",
    .pre_handler = handler_pre,
};
```

**الـ `symbol_name`** بيخلي الـ kernel يحل عنوان الـ function تلقائياً من الـ symbol table في وقت الـ register. الـ `pre_handler` بيتشغل قبل الـ function الأصلية بالظبط — يعني قبل ما تبدأ تتنفذ.

#### الـ `module_init`

بنسجل الـ kprobe وبنطبع العنوان الفعلي اللي انحُل (`kp.addr`) كتأكيد إن الـ hook اشتغل. لو `register_kprobe` رجع error (مثلاً الـ function مش exported أو CONFIG_KPROBES=n) بنفشل بشكل نظيف.

#### الـ `module_exit`

**`unregister_kprobe`** إلزامي هنا — لو المودول اتأزل من غير ما ينزع الـ hook، الـ kernel هيستمر في الـ jump لكود اتمحى من الذاكرة وده يعمل panic فوري.

---

### Makefile للبناء

```makefile
obj-m += prop_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

### تشغيل واختبار

```bash
# بناء المودول
make

# تحميله
sudo insmod prop_probe.ko

# شوف اللوجات — هتظهر في الثواني الأولى من أي device probe
sudo dmesg | grep prop_probe

# مثال output عند تحميل driver جديد:
# prop_probe: dev=0000:00:1f.3 prop="clock-frequency" nval=1
# prop_probe: dev=i2c-0        prop="reg"              nval=1
# prop_probe: [count-query]    dev=spi0.0 prop="cs-gpios"

# إزالة المودول
sudo rmmod prop_probe
```

### متطلبات الـ kernel config

```
CONFIG_KPROBES=y
CONFIG_KPROBE_EVENTS=y   # اختياري لكن مفيد مع perf/ftrace
CONFIG_KALLSYMS=y        # عشان symbol_name resolution يشتغل
```
