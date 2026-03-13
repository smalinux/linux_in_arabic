## Phase 1: الصورة الكبيرة ببساطة

### المشكلة اللي الـ kernel بيحلها هنا

تخيل عندك جهاز — مثلاً chip لـ sensor حرارة — ومحتاج تعرف:
- هو شغّال على I2C ولا SPI؟
- عنوانه (address) إيه؟
- السرعة المدعومة كام؟
- محتاج GPIO رقم كام عشان يتوقف (shutdown pin)?

المعلومات دي بتيجي من مكانين مختلفين تماماً حسب نوع الـ platform:

| الـ platform | مصدر المعلومات |
|---|---|
| **ARM / Embedded** | Device Tree (ملفات `.dts`) |
| **x86 / UEFI** | ACPI tables |
| **Software / Testing** | Software Nodes (بيانات C مكتوبة في الكود مباشرة) |

**المشكلة الحقيقية:** كل مصدر من دول له API مختلف تماماً. الـ driver لو عايز يشتغل على ARM *و* x86 لازم يكتب كود مختلف لكل واحد — ده كان كابوس.

---

### الحل: Unified Device Property Interface

**الـ `property.h`** هو الـ header الرئيسي لـ **Unified Device Property Interface** — طبقة تجريد واحدة فوق Device Tree و ACPI و Software Nodes.

بدل ما الـ driver يسأل:
> "لو device tree → اعمل `of_property_read_u32()`، لو ACPI → اعمل `acpi_dev_get_property()`..."

بيسأل مرة واحدة:
```c
device_property_read_u32(dev, "clock-frequency", &freq);
```

الـ kernel من تحت بيحول السؤال ده للمصدر الصح تلقائياً.

---

### القصة الكاملة بشكل بسيط

```
┌─────────────────────────────────────────────────┐
│              Device Driver Code                 │
│   device_property_read_u32(dev, "reg", &val)    │
└─────────────────────┬───────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│         Unified Property API (property.h)       │
│   fwnode_property_read_u32(fwnode, "reg", &val) │
└───────┬──────────────┬──────────────┬───────────┘
        │              │              │
        ▼              ▼              ▼
   ┌─────────┐   ┌──────────┐   ┌──────────────┐
   │ OF/DTS  │   │   ACPI   │   │ Software Node│
   │ backend │   │ backend  │   │   backend    │
   └─────────┘   └──────────┘   └──────────────┘
```

**الـ `fwnode_handle`** هو القلب — pointer مجرّد بيمثل "نقطة وصف" لأي device بغض النظر عن مصدر المعلومات. كل backend بيملي الـ `fwnode_operations` بـ function pointers خاصة بيه.

---

### إيه اللي موجود في الملف ده تحديداً؟

الـ `property.h` بيعرّف **ثلاث طبقات**:

#### 1. Property Reading API
قراءة القيم من أي مصدر:
```c
// اقرأ قيمة واحدة
device_property_read_u32(dev, "clock-frequency", &val);

// اقرأ array من القيم
device_property_read_u32_array(dev, "reg", buf, count);

// اتحقق من وجود property
device_property_present(dev, "wakeup-source");

// اقرأ string
device_property_read_string(dev, "label", &str);
```

نفس الـ API موجود على مستوى **`fwnode`** مباشرة (بدون `device`) لما تحتاج تتعامل مع sub-nodes.

#### 2. Node Graph API
للـ devices اللي فيها connections زي cameras و display pipelines:
```c
// جيب الـ endpoint الجاي في الـ graph
fwnode_graph_get_next_endpoint(fwnode, prev);

// جيب الـ remote endpoint المتصل بيه
fwnode_graph_get_remote_endpoint(fwnode);
```

ده بيُستخدم كتير في V4L2 (كاميرات) و display subsystem.

#### 3. Software Nodes
لما مش فيه Device Tree ولا ACPI — بتكتب الـ properties مباشرة في الكود C:
```c
static const struct property_entry sensor_props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("label", "front-camera"),
    PROPERTY_ENTRY_BOOL("wakeup-source"),
    { }  /* sentinel */
};

static const struct software_node sensor_node = {
    .name       = "ov5640",
    .properties = sensor_props,
};
```

ده بيُستخدم لـ x86 platforms اللي firmware بتاعها ناقص أو للـ testing.

---

### الـ `struct property_entry` بالتفصيل

الـ struct ده هو اللبنة الأساسية لتخزين property واحدة inline أو بـ pointer:

```c
struct property_entry {
    const char *name;       /* اسم الـ property */
    size_t length;          /* حجم البيانات */
    bool is_inline;         /* هل القيمة مخزنة مباشرة في الـ struct؟ */
    enum dev_prop_type type;/* نوع البيانات: U8/U16/U32/U64/STRING/REF */
    union {
        const void *pointer; /* pointer للبيانات لو خارجية */
        union {
            u8  u8_data[8];
            u16 u16_data[4];
            u32 u32_data[2];
            u64 u64_data[1];
            const char *str[1];
        } value;             /* بيانات inline لو صغيرة */
    };
};
```

الـ macros الكتير في الملف (`PROPERTY_ENTRY_U32`, `PROPERTY_ENTRY_STRING`...) هي shortcut لملء الـ struct ده بشكل صح.

---

### الـ Connection API

بيسمح للـ drivers يلاقوا connections بين devices حتى لو الـ firmware ما وصفهاش صريح:

```c
// دور على connection بـ id معين وجيب match
void *fwnode_connection_find_match(fwnode, con_id, data, match_fn);
```

---

### ليه ده مهم جداً؟

- **Driver واحد يشتغل على كل platform** — Intel NUC و Raspberry Pi و Qualcomm phone يستخدموا نفس الـ sensor driver بدون أي `#ifdef`.
- **Testing بسهولة** — تقدر تحقن properties وهمية بـ `software_node` وتختبر الـ driver من غير hardware حقيقي.
- **ACPI quirks** — كتير من laptops الـ x86 firmware بتاعها بيوصف الـ hardware غلط، فبيستخدموا software nodes يصلحوا الـ properties.

---

### الملفات المكوّنة للـ Subsystem

| الملف | الدور |
|---|---|
| `include/linux/property.h` | الـ public API header — ده ملفنا |
| `include/linux/fwnode.h` | الـ low-level types: `fwnode_handle`, `fwnode_operations` |
| `drivers/base/property.c` | الـ implementation الأساسي للـ API |
| `drivers/base/swnode.c` | الـ software nodes backend |

### ملفات ذات صلة مهمة

| الملف | الصلة |
|---|---|
| `drivers/of/property.c` | الـ Device Tree backend |
| `drivers/acpi/property.c` | الـ ACPI backend |
| `include/media/v4l2-fwnode.h` | استخدام الـ graph API في كاميرات V4L2 |
| `drivers/net/mdio/fwnode_mdio.c` | استخدام الـ API في MDIO/networking |
## Phase 2: شرح الـ Unified Device Property Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

في embedded Linux، الـ hardware موصوف في مكانين مختلفين تماماً حسب الـ platform:

- **ARM/ARM64**: الـ Device Tree (DT) — ملف `.dts` بيوصف كل device وخصائصه كـ properties.
- **x86/ACPI**: الـ ACPI tables — binary tables بتوصف نفس الحاجة بصيغة مختلفة.
- **Embedded بدون firmware**: محتاج تحدد الـ properties من C code مباشرة (software nodes).

المشكلة: كل driver لو كتب كود خاص بيه عشان يتعامل مع DT أو ACPI، الكود هيتكرر 3 مرات. الـ I2C driver مثلاً محتاج يقرأ `clock-frequency` سواء جاي من DT أو من ACPI أو من software node — وده مش منطقي.

**الـ Unified Device Property API** (المعروف بـ `property.h`) اتعمل عشان يحل ده: طبقة abstraction واحدة فوق كل مصادر الـ firmware، الـ driver يسأل بالاسم وما يهمهوش الـ source.

---

### الحل — الـ Approach اللي الـ Kernel اتخده

الـ kernel عمل **interface موحد** قائم على مفهومين:

1. **`fwnode_handle`**: pointer مجرد (abstract handle) بيشير لأي نوع من nodes الـ firmware — DT node، ACPI node، أو software node. النوع محدد عبر الـ `ops` vtable الجوانية.
2. **Property API**: دوال زي `fwnode_property_read_u32()` بتقرأ خاصية باسمها من أي `fwnode_handle` بدون ما تعرف نوعه.

الـ driver بيتكلم مع `fwnode_handle` فقط. الـ backend (DT أو ACPI أو swnode) بيعمل implement للـ `fwnode_operations` ويوفر البيانات.

---

### التشبيه الواقعي — المطعم والـ Menu

تخيل إنك بتبني تطبيق delivery. المطاعم مختلفة: واحد بيستخدم نظام Foodics، واحد تاني بيستخدم ورقة وقلم، وتالت بيستخدم Excel.

| التشبيه | المفهوم الحقيقي |
|---------|-----------------|
| التطبيق (الـ consumer) | الـ driver |
| الـ menu الموحد (interface) | `fwnode_handle` + Property API |
| نظام Foodics backend | OF (Device Tree) backend |
| ورقة وقلم | ACPI backend |
| Excel | Software Node backend |
| "كم سعر البيتزا؟" | `fwnode_property_read_u32(fwnode, "clock-frequency", &val)` |
| الـ menu item name | الـ property name (string) |
| السعر الفعلي | القيمة اللي بترجع في `val` |
| المطعم مش موجود | `fwnode == NULL` → error |

التطبيق مش محتاج يعرف المطعم بيستخدم إيه — هو بيسأل بالاسم وبيجيب الإجابة. نفس الـ driver بيسأل بـ property name وما يهمهوش هل البيانات جاية من DT أو ACPI.

---

### البيكتشر الكبير — معمارية الـ Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Code                              │
│   device_property_read_u32(dev, "clock-frequency", &clk)        │
└───────────────────────────────┬─────────────────────────────────┘
                                │  calls
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              Unified Property API  (property.h)                 │
│                                                                 │
│  device_property_*()  ──► fwnode_property_*()                   │
│                                │                                │
│                         dev_fwnode(dev)                         │
│                                │                                │
│                    gets  fwnode_handle *                         │
└───────────────────────────────┬─────────────────────────────────┘
                                │  dispatches via ops vtable
               ┌────────────────┼──────────────────┐
               ▼                ▼                  ▼
   ┌──────────────────┐  ┌──────────────┐  ┌───────────────────┐
   │  OF (Device Tree) │  │    ACPI      │  │  Software Node    │
   │    Backend        │  │   Backend    │  │    Backend        │
   │                   │  │              │  │                   │
   │ of_fwnode_ops     │  │ acpi_fwnode  │  │ swnode_fwnode_ops │
   │                   │  │   _ops       │  │                   │
   │ Reads from:       │  │ Reads from:  │  │ Reads from:       │
   │  device_node      │  │  acpi_node_  │  │  software_node    │
   │  (parsed DTB)     │  │  info        │  │  (C struct array) │
   └──────────────────┘  └──────────────┘  └───────────────────┘
          ▲                     ▲                    ▲
          │                     │                    │
    ARM/ARM64              x86 / ACPI         No firmware or
    platforms              platforms          quirks/fixups
```

---

### الـ Core Abstraction — الفكرة المحورية

**الـ `fwnode_handle`** هو الـ central abstraction. هو مجرد struct صغير:

```c
struct fwnode_handle {
    struct fwnode_handle    *secondary;   /* fallback fwnode لو الأول ما عندوش property */
    const struct fwnode_operations *ops;  /* vtable — مين اللي بيعمل implement */

    struct device           *dev;         /* الـ device المرتبط (لو موجود) */
    struct list_head         suppliers;   /* fwnode links للـ supply chain */
    struct list_head         consumers;
    u8                       flags;       /* FWNODE_FLAG_* */
};
```

الـ `ops` هو الـ vtable اللي بيخلي كل backend يتصرف بشكل مختلف. لما `fwnode_property_read_u32()` بتتنادى، هي بتعمل:

```c
// simplified
fwnode->ops->property_read_int_array(fwnode, "clock-frequency",
                                     sizeof(u32), &val, 1);
```

الـ `fwnode_operations` vtable بيحتوي على كل العمليات الممكنة:

```c
struct fwnode_operations {
    // lifecycle
    struct fwnode_handle *(*get)(struct fwnode_handle *);
    void (*put)(struct fwnode_handle *);

    // device status
    bool (*device_is_available)(const struct fwnode_handle *);
    const void *(*device_get_match_data)(...);
    bool (*device_dma_supported)(...);
    enum dev_dma_attr (*device_get_dma_attr)(...);

    // property reading
    bool (*property_present)(...);
    bool (*property_read_bool)(...);
    int  (*property_read_int_array)(...);
    int  (*property_read_string_array)(...);

    // tree traversal
    const char *(*get_name)(...);
    struct fwnode_handle *(*get_parent)(...);
    struct fwnode_handle *(*get_next_child_node)(...);
    struct fwnode_handle *(*get_named_child_node)(...);
    int (*get_reference_args)(...);

    // graph (V4L2, MIPI CSI, etc.)
    struct fwnode_handle *(*graph_get_next_endpoint)(...);
    struct fwnode_handle *(*graph_get_remote_endpoint)(...);
    struct fwnode_handle *(*graph_get_port_parent)(...);
    int (*graph_parse_endpoint)(...);

    // hardware access helpers
    void __iomem *(*iomap)(...);
    int (*irq_get)(...);

    // devlink
    int (*add_links)(...);
};
```

---

### العلاقة بين الـ Structs

```
struct device
    │
    └──► fwnode_handle *fwnode  ──────────────────────────────────┐
                                                                  │
                                                    ┌─────────────▼──────────────┐
                                                    │      fwnode_handle          │
                                                    │  ┌─────────────────────┐   │
                                                    │  │ ops ────────────────┼──►│ fwnode_operations
                                                    │  │ secondary ──────────┼──►│ fwnode_handle (fallback)
                                                    │  │ flags               │   │
                                                    │  │ suppliers/consumers │   │
                                                    │  └─────────────────────┘   │
                                                    └────────────────────────────┘
                                                            │
                    ┌───────────────────────────────────────┤
                    │                                       │
          ┌─────────▼──────────┐               ┌───────────▼──────────┐
          │   device_node      │               │   software_node       │
          │  (OF / DT)         │               │                       │
          │  .properties[]     │               │  .name                │
          │  (parsed from DTB) │               │  .parent              │
          └────────────────────┘               │  .properties[]        │
                                               │   ──────────────────  │
                                               │  property_entry[]     │
                                               │  ┌──────────────────┐ │
                                               │  │ .name = "clk-freq│ │
                                               │  │ .type = U32      │ │
                                               │  │ .value = {32000} │ │
                                               │  └──────────────────┘ │
                                               └───────────────────────┘
```

---

### الـ Software Node — لما الـ Firmware مش كفاية

الـ **`software_node`** هو solution للـ hardware اللي ما عندوش DT node كامل أو للـ x86 devices اللي محتاجة extra properties:

```c
/* مثال حقيقي: إضافة properties لـ I2C touchscreen على x86 */
static const struct property_entry ts_props[] = {
    PROPERTY_ENTRY_U32("touchscreen-size-x", 1920),
    PROPERTY_ENTRY_U32("touchscreen-size-y", 1080),
    PROPERTY_ENTRY_BOOL("touchscreen-inverted-x"),
    PROPERTY_ENTRY_STRING("firmware-name", "gsl1680.fw"),
    { }  /* sentinel */
};

static const struct software_node ts_node = {
    .properties = ts_props,
};

/* في probe أو init: */
device_add_software_node(dev, &ts_node);
```

بعد ده الـ driver يقدر يعمل:
```c
device_property_read_u32(dev, "touchscreen-size-x", &width);
// بيشتغل بدون DT بدون ACPI — كده كده
```

---

### الـ `property_entry` — تخزين القيمة Inline أو بـ Pointer

الـ `property_entry` struct مصمم بذكاء عشان يوفر memory:

```c
struct property_entry {
    const char      *name;       /* اسم الـ property */
    size_t           length;     /* حجم البيانات بالبايت */
    bool             is_inline;  /* هل القيمة متخزنة مباشرة في الـ struct؟ */
    enum dev_prop_type type;     /* U8, U16, U32, U64, STRING, REF */
    union {
        const void *pointer;     /* لو البيانات كبيرة (array) */
        union {
            u8  u8_data[8];      /* inline storage — 8 bytes كحد أقصى */
            u16 u16_data[4];
            u32 u32_data[2];
            u64 u64_data[1];
            const char *str[1];  /* على 64-bit: pointer واحد = 8 bytes */
        } value;
    };
};
```

- لو `is_inline = true`: القيمة موجودة في `value` مباشرة داخل الـ struct — zero allocation.
- لو `is_inline = false`: في pointer في `pointer` بيشاور على array خارجي.

**المثال:**
```c
/* single u32 — stored inline, no heap */
PROPERTY_ENTRY_U32("clock-frequency", 400000)
// ينتج:  is_inline=true, value.u32_data[0] = 400000

/* array of u32 — stored via pointer */
static const u32 speeds[] = { 100000, 400000, 1000000 };
PROPERTY_ENTRY_U32_ARRAY("supported-speeds", speeds)
// ينتج:  is_inline=false, pointer = speeds
```

---

### الـ Fwnode Graph — ربط الـ Hardware بـ Topology

الـ **fwnode graph** بيعبر عن الـ topology الفيزيائية لربط الـ hardware — مثلاً camera sensor متوصل بـ MIPI CSI-2 بـ host controller.

```
camera-sensor (fwnode)
    └── port@0
            └── endpoint@0
                    remote-endpoint ──────► endpoint@0
                                              └── port@0
                                                    └── csi-host (fwnode)
```

الـ API:
```c
/* من الـ camera driver: ابعت الصورة لمين؟ */
struct fwnode_handle *ep = fwnode_graph_get_next_endpoint(dev_fwnode(dev), NULL);
struct fwnode_handle *remote_parent = fwnode_graph_get_remote_port_parent(ep);
// remote_parent = fwnode للـ CSI host
```

ده بيُستخدم في: V4L2 media subsystem، MIPI DSI، MIPI CSI-2، USB Type-C alt modes.

---

### الـ `secondary` Field — الـ Fallback Chain

الـ `fwnode_handle` عنده `secondary` pointer. ده بيُستخدم عشان تربط software node بـ DT node:

```
device_node (DT fwnode)
    │
    └── secondary ──► software_node (swnode fwnode)
                           │
                           └── secondary ──► NULL
```

لما `fwnode_property_present()` بتُستدعى، هي بتبحث في الأول في الـ primary، ولو ما لقتش بتروح للـ secondary. ده بيسمح بـ **override DT properties** أو **إضافة properties مش موجودة في DT**.

---

### الـ Reference Properties — روابط بين الـ Nodes

الـ **`DEV_PROP_REF`** نوع خاص — مش value عادية لكن **reference لـ node تاني**.

في DT مثلاً:
```dts
i2c0: i2c@40005000 {
    clocks = <&rcc 42>;   /* reference لـ clock node مع argument */
};
```

في الـ software node المكافئ:
```c
static const struct software_node_ref_args clk_ref =
    SOFTWARE_NODE_REFERENCE(&rcc_node, 42);

static const struct property_entry i2c_props[] = {
    PROPERTY_ENTRY_REF("clocks", &clk_ref),
    { }
};
```

الـ driver بيقرأها عبر:
```c
struct fwnode_reference_args args;
fwnode_property_get_reference_args(fwnode, "clocks", "#clock-cells",
                                   0, 0, &args);
// args.fwnode = fwnode للـ rcc clock controller
// args.args[0] = 42  (clock ID)
```

ده بالظبط اللي بيعتمد عليه الـ **clock subsystem** والـ **pinctrl subsystem** والـ **regulator subsystem** عشان يربطوا resources ببعض.

> **ملاحظة**: الـ clock subsystem عبارة عن framework منفصل (`include/linux/clk.h`) بيدير الـ clock tree للـ SoC — لكن الربط بين device والـ clock بيتم عبر `fwnode_reference_args` هنا.

---

### الـ devcon — Device Connections

الـ **`fwnode_connection_find_match()`** بيبحث عن connection بين اتنين devices عبر الـ firmware graph. ده بيُستخدم في الـ **generic consumer/supplier** model:

```c
typedef void *(*devcon_match_fn_t)(const struct fwnode_handle *fwnode,
                                   const char *id, void *data);

/* مثال: GPIO driver يبحث عن connection بـ "enable-gpios" */
void *result = fwnode_connection_find_match(
    dev_fwnode(dev), "enable-gpios", NULL, my_match_fn);
```

---

### ايه اللي الـ Framework ده بيمتلكه vs. ايه اللي بيفوضه للـ Drivers

| الجانب | الـ Property Framework يتولاه | الـ Driver / Backend يتولاه |
|--------|-------------------------------|------------------------------|
| قراءة properties بالاسم | ✅ unified API | ❌ |
| تفسير القيمة (معناها إيه) | ❌ | ✅ الـ driver يعرف "clock-frequency" معناها إيه |
| تخزين الـ DT node tree | ❌ | ✅ OF layer |
| تخزين الـ ACPI tables | ❌ | ✅ ACPI layer |
| software node storage | ✅ `software_node` structs | ❌ |
| tree traversal (children/parents) | ✅ unified | ❌ |
| graph traversal (ports/endpoints) | ✅ unified | ❌ |
| IRQ number resolution | ✅ `fwnode_irq_get()` delegates | ✅ IRQ domain layer |
| Clock / Regulator binding | ✅ reference args API | ✅ clock/regulator subsystems |
| iomap | ✅ `fwnode_iomap()` delegates | ✅ backend provides implementation |
| match data (driver_data) | ✅ `device_get_match_data()` | ✅ backend reads from OF/ACPI tables |

---

### الـ Two-Level API Design

الـ API فيها مستويين متوازيين عمداً:

```
device_property_read_u32(dev, "prop", &val)
        │
        ▼
fwnode_property_read_u32(dev_fwnode(dev), "prop", &val)
        │
        ▼
fwnode->ops->property_read_int_array(fwnode, "prop", sizeof(u32), &val, 1)
```

- **`device_property_*()`**: للـ drivers اللي عندهم `struct device *` — الحالة الطبيعية.
- **`fwnode_property_*()`**: للـ code اللي بيتعامل مع sub-nodes أو nodes مش مرتبطة بـ device مباشرة (زي child nodes في DT).

لما `nval = 0` في أي `_array` function، بيرجع عدد العناصر بدل ما يقرأ — ده بيخلي `device_property_count_u32()` مجرد wrapper:

```c
static inline int device_property_count_u32(const struct device *dev,
                                             const char *propname)
{
    return device_property_read_u32_array(dev, propname, NULL, 0);
}
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Flags — Cheatsheet

#### `enum dev_prop_type` — نوع قيمة الـ property

| القيمة | المعنى |
|--------|--------|
| `DEV_PROP_U8` | بيانات 8-bit unsigned |
| `DEV_PROP_U16` | بيانات 16-bit unsigned |
| `DEV_PROP_U32` | بيانات 32-bit unsigned |
| `DEV_PROP_U64` | بيانات 64-bit unsigned |
| `DEV_PROP_STRING` | بيانات نصية (`const char *`) |
| `DEV_PROP_REF` | مرجع لـ `software_node_ref_args` |

#### `enum dev_dma_attr` — قدرة الـ DMA

| القيمة | المعنى |
|--------|--------|
| `DEV_DMA_NOT_SUPPORTED` | الجهاز مش بيدعم DMA خالص |
| `DEV_DMA_NON_COHERENT` | DMA بدون cache coherency |
| `DEV_DMA_COHERENT` | DMA مع cache coherency كامل |

#### `FWNODE_FLAG_*` — فلاجز الـ `fwnode_handle`

| الفلاج | الـ Bit | المعنى |
|--------|---------|--------|
| `FWNODE_FLAG_LINKS_ADDED` | 0 | الـ fwnode اتعمله parse للـ links |
| `FWNODE_FLAG_NOT_DEVICE` | 1 | مش هيتحول لـ `struct device` |
| `FWNODE_FLAG_INITIALIZED` | 2 | الـ hardware اتعمل له initialize |
| `FWNODE_FLAG_NEEDS_CHILD_BOUND_ON_ADD` | 3 | محتاج الـ children يتعملوا bind الأول |
| `FWNODE_FLAG_BEST_EFFORT` | 4 | يـ probe بدون انتظار كل الـ suppliers |
| `FWNODE_FLAG_VISITED` | 5 | اتزار أثناء الـ traversal |

#### `FWLINK_FLAG_*` — فلاجز الـ `fwnode_link`

| الفلاج | الـ Bit | المعنى |
|--------|---------|--------|
| `FWLINK_FLAG_CYCLE` | 0 | اللينك جزء من cycle — لا توقف الـ probe |
| `FWLINK_FLAG_IGNORE` | 1 | تجاهل اللينك ده خالص |

#### `FWNODE_GRAPH_*` — فلاجز البحث في الـ graph

| الفلاج | الـ Bit | المعنى |
|--------|---------|--------|
| `FWNODE_GRAPH_ENDPOINT_NEXT` | 0 | لو مفيش match بالظبط، جيب الـ ID الأعلى |
| `FWNODE_GRAPH_DEVICE_DISABLED` | 1 | اقبل الـ endpoints حتى لو الجهاز disabled |

---

### 1. الـ Structs المهمة

---

#### `struct fwnode_handle` _(في `fwnode.h`)_

**الغرض:** ده الـ handle الأساسي لأي node في شجرة الـ firmware — سواء كانت DT node أو ACPI node أو software node. كل حاجة في الـ property API بتتعامل مع `fwnode_handle`.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `secondary` | `struct fwnode_handle *` | فلو مفيش الـ op في الـ primary، بيجرب الـ secondary (مثلاً DT + swnode معاً) |
| `ops` | `const struct fwnode_operations *` | جدول العمليات (vtable) الخاص بنوع الـ fwnode |
| `dev` | `struct device *` | الـ device المرتبط بيه (بيستخدمه device links فقط) |
| `suppliers` | `struct list_head` | قايمة الـ suppliers في الـ device link graph |
| `consumers` | `struct list_head` | قايمة الـ consumers في الـ device link graph |
| `flags` | `u8` | `FWNODE_FLAG_*` الفلاجز |

**الارتباط بالـ structs التانية:**
- بيشاور على `fwnode_operations` → الـ vtable
- بيتملكه `struct device` → `dev->fwnode`
- بيتبنى فوقيه `software_node` عبر `software_node_fwnode()`

---

#### `struct fwnode_operations` _(في `fwnode.h`)_

**الغرض:** الـ vtable اللي كل backend (DT, ACPI, software_node) بيعمل له implement. بيسمح للـ property API تشتغل بدون ما تعرف نوع الـ firmware.

| الـ Op | الشرح |
|--------|-------|
| `get` / `put` | إدارة الـ reference counting |
| `device_is_available` | هل الجهاز متاح؟ (مش disabled في DT) |
| `device_get_match_data` | بيجيب الـ driver data المرتبطة |
| `device_dma_supported` | هل الجهاز بيدعم DMA؟ |
| `device_get_dma_attr` | نوع الـ DMA coherency |
| `property_present` | هل الـ property موجودة؟ |
| `property_read_bool` | اقرأ property boolean |
| `property_read_int_array` | اقرأ array من integers |
| `property_read_string_array` | اقرأ array من strings |
| `get_name` / `get_name_prefix` | اسم الـ node |
| `get_parent` / `get_next_child_node` / `get_named_child_node` | تصفح الشجرة |
| `get_reference_args` | اجلب reference مع arguments |
| `graph_get_next_endpoint` | التالي في graph endpoints |
| `graph_get_remote_endpoint` | الـ endpoint الطرف التاني |
| `graph_get_port_parent` | parent الـ port |
| `graph_parse_endpoint` | احلل port وـ endpoint ID |
| `iomap` | map الـ register region |
| `irq_get` | اجلب IRQ رقم |
| `add_links` | أنشئ device links من الـ firmware |

---

#### `struct property_entry`

**الغرض:** تمثيل **مدمج في الكود** لأي property جهاز — بتستخدمها لما تعمل `software_node` أو تضيف properties ببرنامج بدون DT أو ACPI.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `name` | `const char *` | اسم الـ property (مثلاً `"clock-frequency"`) |
| `length` | `size_t` | حجم البيانات بالـ bytes |
| `is_inline` | `bool` | القيمة مخزنة inline داخل الـ struct نفسه |
| `type` | `enum dev_prop_type` | نوع البيانات |
| `pointer` | `const void *` | بوينتر للبيانات لو مش inline |
| `value` | union | البيانات inline: `u8[8]`, `u16[4]`, `u32[2]`, `u64[1]`, `str[1]` |

**ملاحظة مهمة:** الـ `value` union بتملأ 8 bytes بالظبط — نفس حجم `u64` — بحيث تخزن inline أي نوع صغير.

**الارتباط بالـ structs:**
- بيتحمل في `software_node.properties`
- الـ `DEV_PROP_REF` بيشاور على `software_node_ref_args`

---

#### `struct software_node`

**الغرض:** بديل للـ DT/ACPI لما الـ hardware description ناقصة أو بتيجي من الـ driver نفسه. بيشكل شجرة nodes ليها properties وـ parent/child.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `name` | `const char *` | اسم الـ node |
| `parent` | `const struct software_node *` | الـ parent في الشجرة |
| `properties` | `const struct property_entry *` | array من الـ properties (بتخلص بـ `{}` فاضي) |

**الارتباط:**
- بيتربط بـ `fwnode_handle` عبر `software_node_fwnode()`
- بيتربط بـ `device` عبر `device_add_software_node()`
- بيحتوي على `property_entry[]`

---

#### `struct software_node_ref_args`

**الغرض:** property من نوع `DEV_PROP_REF` — بتشاور على node تاني مع arguments إضافية (مثلاً GPIO number أو clock ID).

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `swnode` | `const struct software_node *` | المرجع لو النوع `software_node` |
| `fwnode` | `struct fwnode_handle *` | المرجع لو النوع `fwnode_handle` مباشرة |
| `nargs` | `unsigned int` | عدد الـ args |
| `args[16]` | `u64[]` | الـ arguments (أقصى 16 — `NR_FWNODE_REFERENCE_ARGS`) |

---

#### `struct fwnode_reference_args`

**الغرض:** نفس الفكرة بس على مستوى الـ fwnode العام (مش software_node بس) — النتيجة اللي بترجع من `fwnode_property_get_reference_args()`.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `fwnode` | `struct fwnode_handle *` | الـ node المُشار إليه |
| `nargs` | `unsigned int` | عدد الـ args |
| `args[16]` | `u64[]` | الـ arguments |

---

#### `struct fwnode_link`

**الغرض:** يمثل علاقة dependency بين جهازين في الـ firmware graph — supplier وـ consumer.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `supplier` | `struct fwnode_handle *` | الـ node المورِّد |
| `s_hook` | `struct list_head` | ربطه في قايمة `supplier->consumers` |
| `consumer` | `struct fwnode_handle *` | الـ node المستهلِك |
| `c_hook` | `struct list_head` | ربطه في قايمة `consumer->suppliers` |
| `flags` | `u8` | `FWLINK_FLAG_*` |

---

#### `struct fwnode_endpoint`

**الغرض:** بيمثل endpoint في الـ OF graph — كل port/endpoint بيجيب منه port ID وـ endpoint ID.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `port` | `unsigned int` | رقم الـ port |
| `id` | `unsigned int` | رقم الـ endpoint داخل الـ port |
| `local_fwnode` | `const struct fwnode_handle *` | مؤشر للـ fwnode نفسه |

---

### 2. مخططات العلاقات بين الـ Structs

```
┌──────────────────────────────────────────────────────────────┐
│                        struct device                         │
│                       dev->fwnode ──────────────────────┐   │
└──────────────────────────────────────────────────────────┼───┘
                                                           │
                                         ┌─────────────────▼──────────────────┐
                                         │        struct fwnode_handle         │
                                         │  ┌──────────────────────────────┐   │
                                         │  │ ops ──► fwnode_operations    │   │
                                         │  │ secondary ──► fwnode_handle  │   │
                                         │  │ dev ──► struct device        │   │
                                         │  │ suppliers: list_head         │   │
                                         │  │ consumers: list_head         │   │
                                         │  │ flags: u8                    │   │
                                         │  └──────────────────────────────┘   │
                                         └───────────────┬────────────────────┘
                                                         │
                         ┌───────────────────────────────┼──────────────────────┐
                         │                               │                      │
              ┌──────────▼──────────┐        ┌───────────▼──────────┐           │
              │  struct software_   │        │  struct fwnode_link   │           │
              │  node               │        │  supplier ──► fwnode  │           │
              │  name               │        │  consumer ──► fwnode  │           │
              │  parent ──► swnode  │        │  flags               │           │
              │  properties ──────┐ │        └──────────────────────┘           │
              └───────────────────┼─┘                                           │
                                  │                               ┌─────────────▼──┐
                    ┌─────────────▼──────────────────┐            │ fwnode_endpoint │
                    │    struct property_entry[]      │            │ port            │
                    │  name, length, is_inline, type  │            │ id              │
                    │  pointer ──► raw data           │            │ local_fwnode   │
                    │  value (inline union 8B)        │            └────────────────┘
                    │   └─► u8/u16/u32/u64/str        │
                    │  DEV_PROP_REF ──────────────┐   │
                    └─────────────────────────────┼───┘
                                                  │
                              ┌───────────────────▼──────────────────┐
                              │   struct software_node_ref_args       │
                              │   swnode ──► software_node            │
                              │   fwnode ──► fwnode_handle            │
                              │   nargs, args[16]                     │
                              └───────────────────────────────────────┘
```

---

#### علاقة الـ `fwnode_operations` كـ vtable

```
┌───────────────────────┐      ┌─────────────────────────────────────┐
│  fwnode_handle        │      │  fwnode_operations (vtable)         │
│  ops ─────────────────┼─────►│  .get()                             │
└───────────────────────┘      │  .put()                             │
                               │  .property_present()                │
                               │  .property_read_int_array()         │
                               │  .property_read_string_array()      │
                               │  .get_parent()                      │
                               │  .get_next_child_node()             │
                               │  .graph_get_next_endpoint()         │
                               │  .add_links()                       │
                               │  ...                                │
                               └──────────┬──────────────────────────┘
                                          │ implemented by:
                     ┌────────────────────┼─────────────────┐
                     │                    │                  │
              ┌──────▼──────┐   ┌─────────▼──────┐  ┌───────▼──────┐
              │  of_fwnode_ │   │ acpi_fwnode_   │  │ swnode_ops   │
              │  ops (DT)   │   │ ops (ACPI)     │  │ (software)   │
              └─────────────┘   └────────────────┘  └──────────────┘
```

---

### 3. مخططات الـ Lifecycle

#### Lifecycle الـ `software_node` الـ static

```
[Compile Time]
    │
    ▼
static const struct property_entry props[] = {
    PROPERTY_ENTRY_U32("clock-frequency", 400000),
    PROPERTY_ENTRY_STRING("label", "my-sensor"),
    {}   ← terminator
};

static const struct software_node my_node = {
    .name = "my-sensor",
    .properties = props,
};
    │
    ▼
[Boot / Driver Init]
    │
    ├─► software_node_register(&my_node)
    │       └─► يشجر الـ node في الـ kernel ويخلق fwnode_handle مربوط بيه
    │
    ├─► device_add_software_node(dev, &my_node)
    │       └─► يربط الـ fwnode بالـ device (dev->fwnode = ...)
    │
    ├─► [Driver probes]
    │       └─► device_property_read_u32(dev, "clock-frequency", &val)
    │
    ▼
[Driver Remove / Device Unbind]
    │
    ├─► device_remove_software_node(dev)
    │
    └─► software_node_unregister(&my_node)
```

#### Lifecycle الـ `software_node` الـ dynamic (managed)

```
[Driver Probe]
    │
    ▼
device_create_managed_software_node(dev, props, parent)
    │  ← kernel بيعمل alloc + register + bind في خطوة واحدة
    │  ← بيتحرر أوتوماتيك لما الـ device يتحرر (managed)
    │
    ▼
[Use]
device_property_read_*() / fwnode_property_read_*()
    │
    ▼
[Device Released]
    └─► kernel بيحرر الـ software_node أوتوماتيك
```

#### Lifecycle الـ `fwnode_handle` Reference Counting

```
fwnode_handle_get(fwnode)   ← زيادة الـ refcount
        │
        ▼
   [use fwnode ...]
        │
        ▼
fwnode_handle_put(fwnode)   ← تخفيض الـ refcount
        │
        ├─► refcount > 0 → الـ node لسه شغال
        └─► refcount == 0 → بيتحرر

/* الـ scoped macros بتعمل put أوتوماتيك */
fwnode_for_each_child_node_scoped(parent, child) {
    /* child بيتحرر أوتوماتيك في نهاية الـ scope */
}
```

---

### 4. مخططات تدفق الاستدعاء

#### قراءة property من driver

```
driver calls:
device_property_read_u32(dev, "clock-frequency", &val)
    │
    ▼
dev_fwnode(dev)           ← _Generic macro → يجيب fwnode من الـ device
    │
    ▼
fwnode_property_read_u32(fwnode, "clock-frequency", &val)
    │
    ▼
fwnode_property_read_u32_array(fwnode, propname, val, 1)
    │
    ▼
fwnode_call_int_op(fwnode, property_read_int_array, ...)
    │  ← macro بيتحقق من fwnode->ops->property_read_int_array
    │
    ├─► [DT backend]  of_fwnode_ops.property_read_int_array()
    │       └─► of_property_read_u32() → reads from DTB
    │
    ├─► [ACPI backend] acpi_fwnode_ops.property_read_int_array()
    │       └─► acpi_dev_get_property() → reads from ACPI table
    │
    └─► [swnode backend] swnode_ops.property_read_int_array()
            └─► يبحث في property_entry[] بالاسم
                └─► بيرجع القيمة من value.u32_data[0] أو pointer
```

#### تصفح child nodes

```
driver calls:
device_for_each_child_node(dev, child) { ... }
    │
    ▼
device_get_next_child_node(dev, child)
    │
    ▼
fwnode_get_next_child_node(dev_fwnode(dev), child)
    │
    ▼
fwnode_call_ptr_op(fwnode, get_next_child_node, child)
    │
    ├─► DT: بيرجع الـ of_node التالي
    └─► swnode: بيتصفح الـ software_node children

[داخل الـ loop]
    ▼
fwnode_property_read_*()   ← يقرأ properties من الـ child
    │
[نهاية الـ loop أو break]
    ▼
fwnode_handle_put(child)   ← لازم يتعمل manually لو break/return
                             أو استخدم _scoped variant
```

#### الـ graph endpoint traversal

```
fwnode_graph_for_each_endpoint(fwnode, child)
    │
    ▼
fwnode_graph_get_next_endpoint(fwnode, prev)
    │  ← بيتصفح ports ثم endpoints جوا كل port
    │
    ▼
fwnode_graph_parse_endpoint(child, &ep)
    │  ← بيملا fwnode_endpoint { .port, .id, .local_fwnode }
    │
    ▼
fwnode_graph_get_remote_endpoint(child)
    │  ← يجيب الـ endpoint في الطرف التاني
    │
    ▼
fwnode_graph_get_port_parent(remote_ep)
    │  ← يجيب الـ device الـ remote
    │
    ▼
fwnode_graph_get_remote_port_parent(child)
    │  ← shortcut يعمل نفس الخطوتين السابقتين
```

#### `fwnode_property_get_reference_args` flow

```
driver calls:
fwnode_property_get_reference_args(fwnode, "clocks", "#clock-cells",
                                    0, 0, &args)
    │
    ▼
fwnode_call_int_op(fwnode, get_reference_args, ...)
    │
    ├─► DT: of_parse_phandle_with_args()
    │       └─► يملا fwnode_reference_args { .fwnode, .nargs, .args[] }
    │
    └─► swnode: يبحث في property_entry[] عن DEV_PROP_REF
            └─► software_node_ref_args → { .swnode, .nargs, .args[] }
                └─► يحوله لـ fwnode_reference_args
```

---

### 5. استراتيجية الـ Locking

الـ `property.h` نفسه **مش بيعرّف locks**، بس العمليات المختلفة بتعتمد على locking في طبقات أعمق:

#### جدول الـ Locking بالطبقة

| الطبقة | الـ Lock المستخدم | المحمي |
|--------|------------------|--------|
| **DT backend** | `of_mutex` (في `of/base.c`) | قراءة/كتابة الـ DT tree |
| **ACPI backend** | `acpi_lock` | الـ ACPI namespace |
| **software_node** | `swnode_root` spinlock / `device_lock` | تسجيل/إلغاء الـ software nodes |
| **fwnode_link** | `device_links_lock` (rwsem) | إضافة/حذف الـ fwnode links |
| **device.fwnode** | `device_lock(dev)` | تعيين `dev->fwnode` |

#### قواعد عامة

```
/* الـ read-only properties API → آمنة من أي context */
device_property_read_u32(dev, ...)   /* no lock needed by caller */

/* تسجيل software_node → بس من process context */
software_node_register()             /* takes internal mutex */
device_add_software_node()           /* takes device_lock */

/* fwnode_link_add → من process context */
fwnode_link_add(con, sup, flags)     /* takes device_links_lock */
```

#### ترتيب الـ Locks (Lock Ordering)

```
device_lock(dev)           ← الأعلى
    └─► swnode internal lock
            └─► of_mutex / acpi_lock (بيتاخدوا من الـ ops callbacks)
```

> القاعدة: **مش بتاخد of_mutex جوا device_lock**، وإلا deadlock. الـ fwnode ops بيتعملوا call وهما بيمسكوا الـ device_lock فقط لو الـ backend نفسه thread-safe.

#### الـ Reference Counting

```
fwnode_handle_get()  ← atomic refcount increment
fwnode_handle_put()  ← atomic refcount decrement + optional free

/* مش محتاج lock خارجي — atomic ops كفاية */
/* لكن لازم تضمن مش بتـ put بعد ما حد تاني حرر */
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Group 1: Device Property Read (device_property_*)

| Function | الغرض |
|---|---|
| `device_property_present` | هل الـ property موجودة على الـ device؟ |
| `device_property_read_bool` | اقرأ property من نوع boolean |
| `device_property_read_u8/u16/u32/u64` | اقرأ scalar integer property |
| `device_property_read_u8/u16/u32/u64_array` | اقرأ array of integers |
| `device_property_read_string` | اقرأ string property واحدة |
| `device_property_read_string_array` | اقرأ array of strings |
| `device_property_match_string` | ابحث عن string في property array وارجع الـ index |
| `device_property_match_property_string` | طابق قيمة property مع array of known strings |
| `device_property_count_u8/u16/u32/u64` | عدّ عناصر integer array property |
| `device_property_string_array_count` | عدّ عناصر string array property |

#### Group 2: Fwnode Property Read (fwnode_property_*)

| Function | الغرض |
|---|---|
| `fwnode_property_present` | هل الـ property موجودة على fwnode معين؟ |
| `fwnode_property_read_bool` | boolean property على fwnode |
| `fwnode_property_read_u8/u16/u32/u64` | scalar integer من fwnode |
| `fwnode_property_read_u8/u16/u32/u64_array` | integer array من fwnode |
| `fwnode_property_read_string` | string property من fwnode |
| `fwnode_property_read_string_array` | string array من fwnode |
| `fwnode_property_match_string` | ابحث عن string في array على fwnode |
| `fwnode_property_match_property_string` | طابق قيمة property مع array of strings |
| `fwnode_property_count_u8/u16/u32/u64` | عدّ عناصر integer array على fwnode |
| `fwnode_property_string_array_count` | عدّ عناصر string array على fwnode |

#### Group 3: Fwnode Navigation

| Function | الغرض |
|---|---|
| `dev_fwnode(dev)` | جيب الـ fwnode_handle من الـ device |
| `fwnode_get_name` | اجيب اسم الـ node |
| `fwnode_get_name_prefix` | prefix للطباعة |
| `fwnode_name_eq` | قارن اسم الـ node بـ string |
| `fwnode_get_parent` | الـ parent node (ref++) |
| `fwnode_get_next_parent` | parent ويـ drop الـ ref القديم |
| `fwnode_count_parents` | عدد الـ parents |
| `fwnode_get_nth_parent` | الـ parent عند depth معينة |
| `fwnode_get_next_child_node` | iteration على الـ children |
| `fwnode_get_next_available_child_node` | iteration على الـ available children بس |
| `fwnode_get_named_child_node` | child باسم محدد |
| `fwnode_get_child_node_count` | عدد الـ child nodes |
| `fwnode_get_named_child_node_count` | عدد الـ children باسم معين |
| `device_get_next_child_node` | iteration على children من الـ device |
| `device_get_named_child_node` | child باسم من الـ device |
| `device_get_child_node_count` | عدد children للـ device |
| `device_get_named_child_node_count` | عدد children باسم للـ device |

#### Group 4: Reference Handling

| Function | الغرض |
|---|---|
| `fwnode_handle_get` | زوّد الـ refcount |
| `fwnode_handle_put` | نزّل الـ refcount |
| `fwnode_property_get_reference_args` | اجيب referenced fwnode مع arguments |
| `fwnode_find_reference` | اجيب referenced fwnode بـ index |

#### Group 5: Graph API

| Function | الغرض |
|---|---|
| `fwnode_graph_get_next_endpoint` | التالي في الـ graph endpoints |
| `fwnode_graph_get_port_parent` | الـ device node من endpoint/port |
| `fwnode_graph_get_remote_port_parent` | الـ device الـ remote |
| `fwnode_graph_get_remote_port` | الـ remote port |
| `fwnode_graph_get_remote_endpoint` | الـ remote endpoint |
| `fwnode_graph_get_endpoint_by_id` | endpoint بـ port+id |
| `fwnode_graph_get_endpoint_count` | عدد الـ endpoints |
| `fwnode_graph_parse_endpoint` | parse port/endpoint IDs |
| `fwnode_graph_is_endpoint` | هل الـ node endpoint؟ |

#### Group 6: Software Node API

| Function | الغرض |
|---|---|
| `software_node_register` | سجّل software node واحدة |
| `software_node_unregister` | شيل software node |
| `software_node_register_node_group` | سجّل array من software nodes |
| `software_node_unregister_node_group` | شيل array من software nodes |
| `fwnode_create_software_node` | أنشئ software node من properties |
| `fwnode_remove_software_node` | احذف software node |
| `device_add_software_node` | الصق software node بـ device |
| `device_remove_software_node` | افصل software node من device |
| `device_create_managed_software_node` | أنشئ + الصق software node مدارة بـ devres |
| `is_software_node` | هل الـ fwnode software node؟ |
| `to_software_node` | cast من fwnode لـ software_node |
| `software_node_fwnode` | جيب fwnode_handle من software_node |
| `software_node_find_by_name` | ابحث عن software node باسمها |
| `property_entries_dup` | نسخ array of property_entry |
| `property_entries_free` | حرر array of property_entry |

#### Group 7: Device Helpers

| Function | الغرض |
|---|---|
| `device_is_big_endian` | هل الـ device big-endian? |
| `device_is_compatible` | هل تـ match الـ compatible string? |
| `device_dma_supported` | هل يدعم DMA؟ |
| `device_get_dma_attr` | جيب DMA coherency attribute |
| `device_get_match_data` | جيب driver match data |
| `device_get_phy_mode` | جيب PHY mode |
| `fwnode_get_phy_mode` | جيب PHY mode من fwnode |
| `fwnode_iomap` | map MMIO region من fwnode |
| `fwnode_irq_get` | جيب IRQ number بـ index |
| `fwnode_irq_get_byname` | جيب IRQ number بالاسم |
| `fwnode_device_is_available` | هل الـ device available? |
| `fwnode_device_is_big_endian` | هل big-endian؟ |
| `fwnode_device_is_compatible` | هل compatible string تتطابق؟ |

#### Group 8: Connection API

| Function | الغرض |
|---|---|
| `fwnode_connection_find_match` | دوّر على أول connection تـ match |
| `device_connection_find_match` | نفس الفكرة من device |
| `fwnode_connection_find_matches` | جيب كل الـ matches في buffer |

---

### Group 1: Property Access — Device Level

الـ `device_property_*` functions هي الـ high-level API اللي يستخدمها الـ driver مباشرة. كل function بتعمل `dev_fwnode(dev)` جوّاها وتفوّت الطلب للـ fwnode layer. الفكرة إنك ما تحتاج تعرف إن كان الـ firmware جاي من DT أو ACPI أو software node.

---

#### `device_property_present`

```c
bool device_property_present(const struct device *dev, const char *propname);
```

بتتحقق إن الـ property بالاسم ده موجودة في firmware description بتاع الـ device. بتفوّت الطلب لـ `fwnode_property_present` عبر `dev_fwnode(dev)`. مش بتقرأ القيمة، بس بتتحقق من وجودها.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device المطلوب |
| `propname` | اسم الـ property (مثال: `"clock-frequency"`) |

**Return:** `true` لو موجودة، `false` لو مش موجودة أو الـ fwnode null.

**Key details:** safe لو `dev->fwnode == NULL`، الـ `fwnode_call_bool_op` بيرجع `false` في الحالة دي.

---

#### `device_property_read_u8_array` / `u16` / `u32` / `u64`

```c
int device_property_read_u8_array(const struct device *dev, const char *propname,
                                  u8 *val, size_t nval);
```

بتقرأ array من integers من الـ property. لو `val == NULL` و `nval == 0`، بترجع عدد العناصر المتاحة (trick لـ count). بتـ dispatch لـ `fwnode_property_read_int_array` في الـ fwnode ops.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |
| `propname` | اسم الـ property |
| `val` | buffer الاستقبال، أو `NULL` لـ count mode |
| `nval` | عدد العناصر المطلوبة، أو `0` لـ count mode |

**Return:** `0` on success، عدد العناصر لو `val==NULL`، أو negative error code (`-EINVAL`, `-ENODATA`, `-EOVERFLOW`).

**الـ scalar wrappers:** `device_property_read_u8/u16/u32/u64` مجرد wrappers بتستدعي الـ array version بـ `nval=1`.

**الـ count wrappers:** `device_property_count_u8/u16/u32/u64` بتستدعيها بـ `val=NULL, nval=0`.

---

#### `device_property_read_string`

```c
int device_property_read_string(const struct device *dev, const char *propname,
                                const char **val);
```

بتقرأ أول string في الـ property. الـ pointer بيـ point لـ read-only string في الـ firmware data، متعملش copy منها. تحت الغطا بتستدعي `device_property_read_string_array` بـ `nval=1`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ device |
| `propname` | اسم الـ property |
| `val` | pointer يستقبل الـ string pointer |

**Return:** `0` on success، `-EINVAL` أو `-ENODATA` on error.

---

#### `device_property_match_string`

```c
int device_property_match_string(const struct device *dev,
                                 const char *propname, const char *string);
```

بتدوّر على `string` جوّا array property وبترجع الـ index بتاعها. شائع الاستخدام مع `"clock-names"` و `"interrupt-names"` لتحديد مكان resource معينة.

**Return:** الـ index (>= 0) لو لُقيت، أو `-ENODATA` / `-ENOENT`.

---

#### `device_property_match_property_string`

```c
static inline int device_property_match_property_string(const struct device *dev,
                                                         const char *propname,
                                                         const char * const *array, size_t n);
```

بتقرأ قيمة string property واحدة وتطابقها مع `array` من strings معروفة. مفيدة لـ parse mode/type properties.

**Pseudocode:**
```
value = read_string(dev, propname)
for i in 0..n:
    if value == array[i]: return i
return -ENOENT
```

---

### Group 2: Property Access — Fwnode Level

الـ `fwnode_property_*` functions هي الـ mid-level API. بتشتغل مباشرة مع `fwnode_handle` بدل الـ device. مفيدة لما تبقى شغّال على child node أو reference node مش عنده `struct device`.

---

#### `fwnode_property_present`

```c
bool fwnode_property_present(const struct fwnode_handle *fwnode, const char *propname);
```

بتستدعي `fwnode->ops->property_present(fwnode, propname)` عبر `fwnode_call_bool_op`. لو الـ fwnode عنده secondary (مثال: DT + swnode)، بيتفحص الاثنين بالترتيب. السلوك ده في `fwnode_call_bool_op` بيطلع من الـ core implementation مش من الـ header.

**Key details:** الـ secondary fwnode هو الـ fallback عند عدم إيجاد property في الـ primary.

---

#### `fwnode_property_read_u8/u16/u32/u64_array`

```c
int fwnode_property_read_u8_array(const struct fwnode_handle *fwnode,
                                  const char *propname, u8 *val, size_t nval);
```

نفس semantics `device_property_read_u8_array` بس على fwnode مباشرة. بتـ dispatch لـ `ops->property_read_int_array` مع `elem_size = sizeof(u8)`.

---

### Group 3: Fwnode Navigation

الغرض من الـ navigation API هو traversal للـ firmware tree سواء كان DT أو ACPI أو software node tree. كل الـ functions اللي بترجع `fwnode_handle *` بتزود الـ refcount، لازم تعمل `fwnode_handle_put` بعدها.

---

#### `dev_fwnode(dev)` — Macro

```c
#define dev_fwnode(dev) \
    _Generic((dev), \
             const struct device *: __dev_fwnode_const, \
             struct device *: __dev_fwnode)(dev)
```

الـ macro ده type-safe بيستخدم `_Generic` عشان يرجع `const struct fwnode_handle *` للـ const device و `struct fwnode_handle *` للـ non-const. بيضمن correctness على مستوى الـ type system بدون casting يدوي.

---

#### `fwnode_get_parent`

```c
struct fwnode_handle *fwnode_get_parent(const struct fwnode_handle *fwnode);
```

بترجع الـ parent node مع increment للـ refcount. الـ caller مسؤول عن `fwnode_handle_put`.

---

#### `fwnode_get_next_parent`

```c
struct fwnode_handle *fwnode_get_next_parent(struct fwnode_handle *fwnode);
```

بترجع parent الـ node وتـ drop الـ ref بتاع الـ node الحالية في نفس الوقت. مصممة للاستخدام في iteration loop بدون leak.

```c
/* تعلّو الشجرة بدون leak */
for (node = fwnode_get_parent(start); node;
     node = fwnode_get_next_parent(node))
    /* ... */
```

أو الأسهل: الـ macro `fwnode_for_each_parent_node`.

---

#### `fwnode_get_next_child_node`

```c
struct fwnode_handle *fwnode_get_next_child_node(
    const struct fwnode_handle *fwnode, struct fwnode_handle *child);
```

بترجع الـ child التالي في الـ iteration. لو `child == NULL` بتبدأ من الأول. كل مرة بتزود refcount للـ returned node وتـ drop الـ ref بتاع `child` المدخل.

**Caller context:** عادةً بتتاستخدم جوا `fwnode_for_each_child_node` أو `device_for_each_child_node`.

---

#### `fwnode_get_next_available_child_node`

```c
struct fwnode_handle *fwnode_get_next_available_child_node(
    const struct fwnode_handle *fwnode, struct fwnode_handle *child);
```

نفس `fwnode_get_next_child_node` بس بتتخطى الـ nodes اللي `device_is_available` بترجع false عليها (disabled في DT مثلاً).

---

#### Iteration Macros

```c
/* بيتر على كل children */
#define fwnode_for_each_child_node(fwnode, child)

/* بيتر على available children بس */
#define fwnode_for_each_available_child_node(fwnode, child)

/* بيتر على children باسم معين */
#define fwnode_for_each_named_child_node(fwnode, child, name)
```

**الـ `_scoped` variants** بيستخدموا `__free(fwnode_handle)` cleanup attribute من `<linux/cleanup.h>` عشان تلقائياً يعملوا `fwnode_handle_put` على الـ `child` لو الـ loop اتكسرت بـ `break` أو `return`. ده بيمنع resource leak.

```c
/* الـ child هيتحرر تلقائياً */
fwnode_for_each_child_node_scoped(parent, child) {
    if (condition)
        return child; /* لا leak هنا */
}
```

---

#### `fwnode_get_nth_parent`

```c
struct fwnode_handle *fwnode_get_nth_parent(struct fwnode_handle *fwn,
                                            unsigned int depth);
```

بترجع الـ ancestor على عمق `depth` من الـ node الحالية. `depth=0` يرجع الـ node نفسها (ref++). بتتصاعد الشجرة `depth` مرات باستخدام `fwnode_get_next_parent`.

---

#### `fwnode_name_eq`

```c
bool fwnode_name_eq(const struct fwnode_handle *fwnode, const char *name);
```

بتقارن اسم الـ node (بدون address suffix زي `@0`) بالـ string المعطاة. مهمة لـ `fwnode_for_each_named_child_node`.

---

### Group 4: Reference Count Management

كل `fwnode_handle *` اللي بيرجع من navigation functions بيتعمله refcount++. لازم دايماً يتعمله `fwnode_handle_put` بعد الاستخدام.

---

#### `fwnode_handle_get`

```c
struct fwnode_handle *fwnode_handle_get(struct fwnode_handle *fwnode);
```

بتستدعي `ops->get(fwnode)` لزيادة الـ refcount. بترجع نفس الـ pointer عشان تستخدمها في assignment.

**Return:** نفس `fwnode` لو نجح، `NULL` لو `fwnode == NULL`.

---

#### `fwnode_handle_put`

```c
static inline void fwnode_handle_put(struct fwnode_handle *fwnode);
```

بتستدعي `ops->put(fwnode)` لإنقاص الـ refcount. لو وصل لـ 0، ممكن يتحرر الـ node حسب الـ implementation (OF، ACPI، أو swnode). لازم تستدعيها بعد كل `fwnode_get_*` call لما تخلص من الـ handle.

**Key details:**
- الـ `DEFINE_FREE(fwnode_handle, ...)` بيعمل cleanup class باستخدامها، ده اللي بتعتمد عليه الـ `_scoped` macros.
- آمنة للاستدعاء على `NULL`.

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

بتجيب referenced fwnode بالإضافة لـ integer arguments من property. ده الـ DT `phandle-with-args` pattern (مثال: `clocks`, `interrupts`, `dmas`).

| Parameter | الوصف |
|---|---|
| `fwnode` | الـ node الحالية |
| `prop` | اسم الـ reference property (مثال: `"clocks"`) |
| `nargs_prop` | اسم property عدد الـ args (مثال: `"#clock-cells"`), أو `NULL` |
| `nargs` | عدد الـ args لو `nargs_prop == NULL` |
| `index` | رقم الـ reference في array (0-based) |
| `args` | struct يستقبل النتيجة |

**Return:** `0` on success، `-ENOENT` لو مش موجودة، `-EINVAL` لو format خاطئ.

**Key details:** الـ `args->fwnode` بيتزود refcount، لازم تعمله `fwnode_handle_put`.

---

#### `fwnode_find_reference`

```c
struct fwnode_handle *fwnode_find_reference(const struct fwnode_handle *fwnode,
                                            const char *name,
                                            unsigned int index);
```

نسخة مبسطة من `fwnode_property_get_reference_args` من غير arguments، بس تـ return الـ referenced fwnode مباشرةً.

**Return:** `fwnode_handle *` مع ref++ on success، `ERR_PTR` on error.

---

### Group 5: Graph API

الـ fwnode graph API بتعبّر عن الـ media/video/sensor pipelines. كل device بيبقى ليها ports، كل port بيتضمن endpoints، والـ endpoint بيتوصل بـ endpoint على device تاني.

```
device-A                device-B
  port@0                  port@0
    endpoint@0 ---------> endpoint@0
```

---

#### `fwnode_graph_get_next_endpoint`

```c
struct fwnode_handle *fwnode_graph_get_next_endpoint(
    const struct fwnode_handle *fwnode, struct fwnode_handle *prev);
```

iterator بيمشي على كل الـ endpoints بتاعة device. لو `prev == NULL` بيبدأ من الأول. بترجع ref++ pointer، لازم `fwnode_handle_put`.

**الاستخدام المعتاد:**
```c
fwnode_graph_for_each_endpoint(dev_fwnode(dev), ep) {
    /* process ep */
}
```

---

#### `fwnode_graph_get_port_parent`

```c
struct fwnode_handle *fwnode_graph_get_port_parent(const struct fwnode_handle *fwnode);
```

من endpoint أو port node، بترجع الـ device node الـ parent. مفيدة للتنقل من endpoint لـ device حتى لو عندك endpoint fwnode فقط.

---

#### `fwnode_graph_get_remote_port_parent`

```c
struct fwnode_handle *fwnode_graph_get_remote_port_parent(
    const struct fwnode_handle *fwnode);
```

من local endpoint، بتجيب الـ device node بتاع الـ remote side. ده اللي بتستخدمه عادةً في media drivers لإيجاد الـ connected device.

---

#### `fwnode_graph_get_remote_endpoint`

```c
struct fwnode_handle *fwnode_graph_get_remote_endpoint(
    const struct fwnode_handle *fwnode);
```

بترجع الـ endpoint node على الجانب الـ remote. مختلف عن `get_remote_port_parent` لأنك بتوصل للـ endpoint نفسه مش الـ device.

---

#### `fwnode_graph_get_endpoint_by_id`

```c
struct fwnode_handle *fwnode_graph_get_endpoint_by_id(
    const struct fwnode_handle *fwnode,
    u32 port, u32 endpoint, unsigned long flags);
```

بتجيب endpoint بـ port number وـ endpoint ID. الـ `flags`:
- `FWNODE_GRAPH_ENDPOINT_NEXT`: لو مفيش exact match، جيب الأقرب endpoint بـ ID أكبر.
- `FWNODE_GRAPH_DEVICE_DISABLED`: ما تتجاهلش الـ endpoints على disabled devices.

---

#### `fwnode_graph_parse_endpoint`

```c
int fwnode_graph_parse_endpoint(const struct fwnode_handle *fwnode,
                                struct fwnode_endpoint *endpoint);
```

بتملأ `struct fwnode_endpoint` (port + id + local_fwnode) من الـ endpoint node. لازم تستدعيها على endpoint node مش device node.

---

#### `fwnode_graph_is_endpoint`

```c
static inline bool fwnode_graph_is_endpoint(const struct fwnode_handle *fwnode);
```

بتتحقق من وجود `"remote-endpoint"` property على الـ node. التحقق ده كافي لأن الـ endpoints الصح لازم يكون عندها remote-endpoint.

---

### Group 6: Software Node API

الـ software nodes (swnodes) بتوفر firmware description للـ devices اللي مفيهاش DT node أو ACPI entry. شائع الاستخدام في x86 platforms للـ embedded controllers أو في تعديل properties موجودة.

```
struct software_node {
    const char *name;
    const struct software_node *parent;
    const struct property_entry *properties;
};
```

---

#### `software_node_register`

```c
int software_node_register(const struct software_node *node);
```

بتسجّل software node واحدة في الـ kernel's software node tree. بتنشئ `fwnode_handle` داخلي وتربطه بالـ node. لازم الـ `node->parent` يكون متسجل قبل الـ child.

**Return:** `0` on success، `-EINVAL` لو الـ parent مش متسجل، `-EEXIST` لو متسجلة فعلاً.

**Locking:** بتستخدم internal mutex في الـ software node subsystem.

---

#### `software_node_unregister`

```c
void software_node_unregister(const struct software_node *node);
```

بتشيل الـ node من الـ tree وتـ drop الـ ref بتاعها. لو في device لسه مربوطها، بيبقى فيه problem. لازم تعمل `device_remove_software_node` الأول.

---

#### `software_node_register_node_group`

```c
int software_node_register_node_group(const struct software_node * const *node_group);
```

بتسجّل NULL-terminated array من software nodes بالترتيب. لو أي node فشلت، بتعمل rollback وتـ unregister اللي اتسجلوا قبلها.

**Pseudocode:**
```
for each node in group (until NULL):
    ret = software_node_register(node)
    if ret < 0:
        unregister all previously registered
        return ret
return 0
```

---

#### `fwnode_create_software_node`

```c
struct fwnode_handle *fwnode_create_software_node(
    const struct property_entry *properties,
    const struct fwnode_handle *parent);
```

بتنشئ software node من array of `property_entry` وتربطها بـ parent fwnode. الـ node المنشأة بتبقى **anonymous** (no name) وبتتملكها الـ fwnode infrastructure. الـ `properties` بتتنسخ جوّاها.

**Return:** `fwnode_handle *` on success، `ERR_PTR(-ENOMEM)` on failure.

**Caller context:** بيتاستخدم لما تعوز تضيف properties لـ fwnode موجودة بدون اسم محدد.

---

#### `fwnode_remove_software_node`

```c
void fwnode_remove_software_node(struct fwnode_handle *fwnode);
```

عكس `fwnode_create_software_node`. بتحرر الـ node والـ properties المنسوخة.

---

#### `device_add_software_node`

```c
int device_add_software_node(struct device *dev, const struct software_node *node);
```

بتلصق software node بـ device عن طريق تعيينها كـ `dev->fwnode` (secondary لو في fwnode موجودة). الـ device بيكتسب كل properties الـ node دي.

**Return:** `0` on success، `-ENODEV` لو الـ device مش موجودة، `-EINVAL` لو الـ node مش متسجلة.

**Key details:** ما بتسجلش الـ node أوتوماتيك، لازم تكون متسجلة بـ `software_node_register` الأول. أو استخدم `device_create_managed_software_node` للـ all-in-one.

---

#### `device_remove_software_node`

```c
void device_remove_software_node(struct device *dev);
```

بتفصل الـ software node من الـ device وتـ drop الـ reference. لو كانت secondary، بترجع الـ device لـ primary fwnode.

---

#### `device_create_managed_software_node`

```c
int device_create_managed_software_node(struct device *dev,
                                        const struct property_entry *properties,
                                        const struct software_node *parent);
```

**الـ all-in-one API** للـ software nodes. بتنشئ anonymous software node من `properties`، وتـ register على `parent`، وتربطها بـ `dev`. الأهم: الـ node دي بتتحرر تلقائياً لما الـ device يتحرر (devres-managed).

**Return:** `0` on success، negative error code otherwise.

**متى تستخدمها؟** لما تكون في driver probe وعايز تضيف properties لـ device من غير ما تدير الـ lifecycle يدوياً.

---

#### `is_software_node` / `to_software_node` / `software_node_fwnode`

```c
bool is_software_node(const struct fwnode_handle *fwnode);
const struct software_node *to_software_node(const struct fwnode_handle *fwnode);
struct fwnode_handle *software_node_fwnode(const struct software_node *node);
```

الثلاثة ده type-checking وـ casting utilities:
- **`is_software_node`**: بتتحقق إن الـ ops pointer بتاع الـ fwnode هو `software_node_ops`.
- **`to_software_node`**: `container_of`-based cast، بترجع `NULL` لو مش software node.
- **`software_node_fwnode`**: عكسها، من `software_node *` لـ `fwnode_handle *`.

---

#### `software_node_find_by_name`

```c
const struct software_node *software_node_find_by_name(
    const struct software_node *parent,
    const char *name);
```

بتدوّر على software node باسمها تحت `parent` محدد. لو `parent == NULL` بتدوّر في كل الـ registered nodes.

**Return:** الـ node on success، `NULL` لو مش موجودة.

---

### Group 7: Property Entries — Static Description API

الـ `struct property_entry` هو الـ building block لوصف properties statically في الـ kernel code (compile-time أو early boot).

```
struct property_entry {
    const char *name;
    size_t length;
    bool is_inline;         /* value stored in-place */
    enum dev_prop_type type;
    union {
        const void *pointer;   /* for arrays */
        union { u8/u16/u32/u64/str value[...]; };  /* for scalars */
    };
};
```

---

#### Initialization Macros

| Macro | النوع | المثال |
|---|---|---|
| `PROPERTY_ENTRY_U8(name, val)` | scalar u8 | `PROPERTY_ENTRY_U8("reg", 0x3c)` |
| `PROPERTY_ENTRY_U32(name, val)` | scalar u32 | `PROPERTY_ENTRY_U32("clock-frequency", 400000)` |
| `PROPERTY_ENTRY_STRING(name, val)` | scalar string | `PROPERTY_ENTRY_STRING("status", "okay")` |
| `PROPERTY_ENTRY_BOOL(name)` | boolean (no value) | `PROPERTY_ENTRY_BOOL("wakeup-source")` |
| `PROPERTY_ENTRY_U32_ARRAY(name, arr)` | u32 array | `PROPERTY_ENTRY_U32_ARRAY("reg", regs)` |
| `PROPERTY_ENTRY_REF(name, ref, ...)` | reference | `PROPERTY_ENTRY_REF("clocks", &clk_node)` |
| `PROPERTY_ENTRY_REF_ARRAY(name, arr)` | ref array | `PROPERTY_ENTRY_REF_ARRAY("clocks", clk_refs)` |

**BOOL property** بتعرفها مجرد وجودها (length=0، value فارغة). ما فيش value حقيقية.

**الـ inline optimization**: الـ scalars بتتخزن مباشرة جوا الـ struct في الـ `value` union (is_inline=true)، فمفيش external allocation.

---

#### `property_entries_dup`

```c
struct property_entry *property_entries_dup(const struct property_entry *properties);
```

بتعمل deep copy لـ NULL-terminated array من `property_entry`. الـ deep copy مهمة لأن بعض الـ entries بتـ point لـ external data. بتستخدم `kmemdup` وـ `kstrdup` جوّاها.

**Return:** pointer لـ copied array on success، `ERR_PTR(-ENOMEM)` on failure.

**Caller context:** بيتاستدعى من `fwnode_create_software_node` وغيرها عشان تعمل ownership.

---

#### `property_entries_free`

```c
void property_entries_free(const struct property_entry *properties);
```

عكس `property_entries_dup`. بتحرر الـ deep-copied data. لو الـ array static (مش من `dup`)، لا تستدعيها.

---

#### `SOFTWARE_NODE_REFERENCE` Macro

```c
#define SOFTWARE_NODE_REFERENCE(_ref_, ...)
```

بينشئ `struct software_node_ref_args` compound literal. بيستخدم `_Generic` لتحديد نوع الـ `_ref_` (software_node * أو fwnode_handle *) وتعيينه للـ field الصح. الـ `__VA_ARGS__` هي الـ integer arguments اللي بتتعبأ في `args[]`.

---

### Group 8: Device Attribute Helpers

---

#### `device_dma_supported`

```c
bool device_dma_supported(const struct device *dev);
```

بتسأل الـ firmware node لو الـ device تدعم DMA. بتـ dispatch لـ `ops->device_dma_supported`. على ACPI/DT بتتحقق من الـ bus topology وـ IOMMU presence.

---

#### `device_get_dma_attr`

```c
enum dev_dma_attr device_get_dma_attr(const struct device *dev);
```

بترجع `DEV_DMA_NOT_SUPPORTED`، `DEV_DMA_NON_COHERENT`، أو `DEV_DMA_COHERENT` حسب الـ firmware description. على DT بتبص على `dma-coherent` property.

---

#### `device_get_match_data`

```c
const void *device_get_match_data(const struct device *dev);
```

بترجع الـ `data` pointer من الـ match table entry اللي اتطابقت مع الـ device. على DT بتبص في `of_match_table`، على ACPI في `acpi_match_table`. مفيدة لتمرير variant-specific config من driver tables.

---

#### `device_get_phy_mode` / `fwnode_get_phy_mode`

```c
int device_get_phy_mode(struct device *dev);
int fwnode_get_phy_mode(const struct fwnode_handle *fwnode);
```

بتقرأ `"phy-mode"` property وتحوّلها لـ enum value (`PHY_INTERFACE_MODE_*`). بتستخدم `device_property_match_string` مع array من الـ mode names.

**Return:** PHY mode enum value (>= 0) on success، `-ENODEV` أو `-ENOENT` on error.

---

#### `fwnode_iomap`

```c
void __iomem *fwnode_iomap(struct fwnode_handle *fwnode, int index);
```

بتـ map MMIO region بـ `index` من الـ fwnode. على DT بتستخدم `of_iomap`، على ACPI بتستخدم memory resources. بترجع `__iomem` pointer جاهز للاستخدام.

---

#### `fwnode_irq_get`

```c
int fwnode_irq_get(const struct fwnode_handle *fwnode, unsigned int index);
```

بتجيب Linux IRQ number من الـ firmware node بـ index. على DT بتستدعي `of_irq_get`، على ACPI بتستدعي `acpi_irq_get`. بتـ map الـ hardware interrupt لـ Linux virtual IRQ.

**Return:** IRQ number (> 0) on success، negative error code otherwise.

---

#### `fwnode_irq_get_byname`

```c
int fwnode_irq_get_byname(const struct fwnode_handle *fwnode, const char *name);
```

بتجيب IRQ بالاسم بدل الـ index. بتبحث في `"interrupt-names"` property عن `name`، بعدين بتستدعي `fwnode_irq_get` بالـ index المقابل.

---

#### `fwnode_device_is_available`

```c
bool fwnode_device_is_available(const struct fwnode_handle *fwnode);
```

بتتحقق إن الـ device مش disabled في الـ firmware. على DT بتبص على `status` property (`"okay"` أو `"ok"`). على ACPI بتبص على `_STA` method.

---

#### `fwnode_device_is_big_endian`

```c
static inline bool fwnode_device_is_big_endian(const struct fwnode_handle *fwnode);
```

بتتحقق من:
1. وجود `"big-endian"` property → `true` مباشرةً
2. لو `CONFIG_CPU_BIG_ENDIAN` وفيه `"native-endian"` property → `true`
3. غير كده → `false`

---

#### `device_is_big_endian` / `device_is_compatible`

```c
static inline bool device_is_big_endian(const struct device *dev);
static inline bool device_is_compatible(const struct device *dev, const char *compat);
```

الاثنين مجرد wrappers بيستدعوا `fwnode_device_is_big_endian` و `fwnode_device_is_compatible` بعد جيب الـ fwnode من الـ device. الـ `device_is_compatible` بيستدعي `fwnode_property_match_string(fwnode, "compatible", compat) >= 0`.

---

### Group 9: Connection API

الـ connection API بتعبر عن device interconnections (مثال: regulator-to-consumer، GPIO provider-to-consumer) بطريقة firmware-agnostic.

---

#### `fwnode_connection_find_match`

```c
void *fwnode_connection_find_match(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match);
```

بتدوّر على connections للـ `fwnode` بالـ `con_id`. الـ `match` callback بياخد فيه كل connected fwnode وبيحاول يحوّله لـ useful object (مثال: regulator، GPIO descriptor). بتوقف عند أول match ناجح.

```c
typedef void *(*devcon_match_fn_t)(const struct fwnode_handle *fwnode,
                                   const char *id, void *data);
```

**Return:** الـ value اللي رجعه الـ `match` callback، أو `NULL` لو مفيش match.

---

#### `fwnode_connection_find_matches`

```c
int fwnode_connection_find_matches(const struct fwnode_handle *fwnode,
                                   const char *con_id, void *data,
                                   devcon_match_fn_t match,
                                   void **matches, unsigned int matches_len);
```

نسخة موسّعة بتجمع كل الـ matches في `matches` array بحجم `matches_len`. مفيدة لما في أكتر من connection لنفس الـ id.

**Return:** عدد الـ matches الموجودة on success، negative error on failure.

---

### ملاحظات عامة مهمة

**الـ refcount discipline** هي القاعدة الأساسية في الـ fwnode API:
- كل function بترجع `fwnode_handle *` بتزود الـ refcount.
- الـ caller مسؤول دايماً عن `fwnode_handle_put`.
- الـ `_scoped` macros بتتكفل بده تلقائياً.
- الـ `DEFINE_FREE(fwnode_handle, ...)` هي الـ cleanup hook المستخدمة مع `__free(fwnode_handle)`.

**الـ secondary fwnode** بيسمح بـ layering: مثلاً device عندها DT node كـ primary وـ software node كـ secondary. الـ property lookup بيبدأ بالـ primary وبيفضل بالـ secondary كـ fallback.

**الـ NULL safety**: معظم الـ fwnode operations آمنة مع `NULL` أو `ERR_PTR` input. الـ `fwnode_has_op` بيتحقق من ده قبل الـ dispatch.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

**الـ unified device property subsystem** مش عنده entries مباشرة في debugfs بس الـ fwnode graph وال software nodes بتظهر من خلال:

```bash
# اقرأ كل الـ fwnode links المتاحة
mount -t debugfs none /sys/kernel/debug 2>/dev/null
cat /sys/kernel/debug/devices_deferred          # devices أترجأ probe بسببها
cat /sys/kernel/debug/device_component/all      # component matching

# لو kernel مبني بـ CONFIG_FWNODE_MDESC
ls /sys/kernel/debug/acpi/                      # ACPI fwnode entries
```

الـ DT fwnode بيظهر من خلال:
```bash
# شوف كل الـ device nodes الـ DT وعلاقاتها
ls /sys/kernel/debug/of/                        # OF (OpenFirmware) debug entries
cat /sys/kernel/debug/of/0x<node_addr>/name
```

---

#### 2. sysfs — المسارات المفيدة

```bash
# اقرأ properties مباشرة من sysfs لأي device
ls /sys/bus/platform/devices/<dev_name>/
cat /sys/bus/platform/devices/<dev_name>/of_node/compatible
cat /sys/bus/platform/devices/<dev_name>/of_node/status

# software nodes المسجلة
ls /sys/kernel/software_nodes/
ls /sys/kernel/software_nodes/<node_name>/properties/

# مثال عملي: فحص I2C device
ls /sys/bus/i2c/devices/0-0050/of_node/
cat /sys/bus/i2c/devices/0-0050/of_node/reg

# فحص fwnode flags لـ device معين
cat /sys/bus/platform/devices/<dev>/firmware_node/
```

جدول أهم sysfs entries:

| المسار | المحتوى | الاستخدام |
|--------|---------|-----------|
| `/sys/bus/*/devices/<dev>/of_node/` | DT properties للـ device | تحقق من الـ DT properties |
| `/sys/bus/*/devices/<dev>/firmware_node/` | fwnode handle info | تتبع الـ fwnode المرتبط |
| `/sys/kernel/software_nodes/` | software nodes المسجلة | debug الـ swnode |
| `/sys/bus/*/devices/<dev>/of_node/compatible` | compatible string | تحقق من matching |
| `/sys/firmware/acpi/` | ACPI device properties | على x86 |

---

#### 3. ftrace — Tracepoints وكيفية التفعيل

```bash
# فعّل tracing للـ device probe وال fwnode operations
cd /sys/kernel/debug/tracing

# تتبع device_property_read functions
echo 'device_property_read_u32_array' >> set_ftrace_filter
echo 'fwnode_property_present' >> set_ftrace_filter
echo 'software_node_register' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
# شغّل المودل/الـ device
cat trace | grep -E 'property|fwnode|swnode'
echo 0 > tracing_on

# تتبع كل الـ driver probe مع fwnode lookups
echo 1 > events/device/enable
cat /sys/kernel/debug/tracing/events/devlink/available

# تتبع الـ graph traversal
echo ':mod:of' >> set_ftrace_filter
echo ':mod:acpi' >> set_ftrace_filter
```

استخدام **trace-cmd** (أسهل):
```bash
trace-cmd record -p function_graph \
  -l 'device_property_*' \
  -l 'fwnode_property_*' \
  -l 'software_node_*' \
  modprobe <your_driver>
trace-cmd report | less
```

---

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug لملفات الـ property subsystem
echo 'file drivers/base/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/swnode.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/acpi/property.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/property.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لموديول معين بالكامل
echo 'module <your_module> +p' > /sys/kernel/debug/dynamic_debug/control

# زوّد verbosity الـ DT parsing
echo 'file drivers/of/*.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف النتيجة
dmesg -w | grep -E 'property|fwnode|swnode'
```

من kernel cmdline:
```
dyndbg="file drivers/base/property.c +p"
of_debug=1
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة | متى تفعّله |
|--------|---------|-----------|
| `CONFIG_OF` | تفعيل Device Tree support | على ARM/ARM64 |
| `CONFIG_OF_DYNAMIC` | DT dynamic modification | لتست الـ DT overlays |
| `CONFIG_OF_UNITTEST` | OF unit tests | لتأكيد صحة DT |
| `CONFIG_DEBUG_DRIVER` | تفعيل driver debug messages | أي وقت تعمل debug |
| `CONFIG_ACPI_DEBUG` | ACPI debug messages | على x86 مع ACPI |
| `CONFIG_ACPI_DEBUGGER` | ACPI debugger interface | debug عميق |
| `CONFIG_SWNODE_LOGLEVEL_DEBUG` | software node verbose logging | لـ swnode debug |
| `CONFIG_GENERIC_ARCH_TOPOLOGY` | topology via fwnode | عند debug topology |
| `CONFIG_FWNODE_MDESC` | machine description via fwnode | ARM platforms |
| `CONFIG_DYNAMIC_DEBUG` | dynamic pr_debug activation | ضروري للـ step 4 |
| `CONFIG_KALLSYMS_ALL` | full symbol table | لتحسين stack traces |
| `CONFIG_PROVE_LOCKING` | lockdep للـ fwnode locks | لكتشاف deadlocks |

---

#### 6. devlink وأدوات خاصة بالـ Subsystem

**الـ `dtc` tool** لتحليل الـ DT:
```bash
# فك تشفير الـ DTB الحالي
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/current.dts
grep -A5 'compatible = "myvendor,mydev"' /tmp/current.dts

# تحقق من syntax الـ DTS
dtc -I dts -O dts -o /dev/null my_device.dts 2>&1
```

**`fdtdump` و `fdtget`**:
```bash
fdtget /sys/firmware/fdt /soc/i2c@ff160000 compatible
fdtget /sys/firmware/fdt /soc/i2c@ff160000 status
fdtdump /sys/firmware/fdt 2>/dev/null | grep -A10 "your_node"
```

**ACPI على x86**:
```bash
# اقرأ ACPI properties
acpidump -n DSDT > /tmp/dsdt.dat
acpixtract -a /tmp/dsdt.dat
iasl -d DSDT.dat
grep -A5 '_DSD' DSDT.dsl        # _DSD = device-specific data = properties
```

**`lsdevtree` script** بسيط:
```bash
# طبع كل devices مع compatible strings
for dev in /sys/bus/*/devices/*/of_node; do
    echo "=== $dev ==="; cat "$dev/compatible" 2>/dev/null; echo
done
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel | السبب | الحل |
|-----------------|-------|------|
| `device_property_read_u32: property 'foo' not found` | الـ property مش موجودة في DT/ACPI/swnode | أضف الـ property في DT أو تحقق من الاسم |
| `fwnode_property_present: fwnode is NULL` | device مش مربوط بـ fwnode | تأكد من `dev_fwnode(dev)` مش NULL قبل الاستخدام |
| `software_node_register: node already exists` | تسجيل swnode مكرر | استخدم `software_node_unregister` أولاً أو تأكد من عدم التكرار |
| `fwnode_get_reference_args: reference not found` | الـ phandle مش صحيح في DT | تحقق من الـ phandle والـ property الاسم |
| `-ENODATA` من `device_property_read_*` | الـ property موجودة بس فاضية | تحقق من قيمة الـ property في DT |
| `-EPROTO` من `device_property_read_*` | نوع البيانات مش متطابق | تأكد إن الـ type في DT يطابق الـ DEV_PROP_* |
| `fwnode: Could not find device driver` | الـ compatible مش مطابق لأي driver | تحقق من compatible string في DT مقابل driver |
| `deferred probe pending` | supplier fwnode مش ready | انتظر أو تحقق من boot order |
| `fwnode_graph_get_endpoint_by_id: port N not found` | الـ graph topology في DT غلط | راجع ports/endpoints في DT |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* نقطة 1: بعد قراءة property حساسة */
ret = device_property_read_u32(dev, "clock-frequency", &freq);
if (ret) {
    dev_warn(dev, "missing clock-frequency, using default\n");
    /* WARN_ON هنا لو الـ property mandatory */
    WARN_ON(ret == -ENODATA);
}

/* نقطة 2: تحقق إن fwnode مش NULL قبل traversal */
fwnode = dev_fwnode(dev);
if (WARN_ON(!fwnode)) {
    dev_err(dev, "device has no fwnode!\n");
    dump_stack();
    return -EINVAL;
}

/* نقطة 3: عند loop على child nodes */
device_for_each_child_node(dev, child) {
    ret = fwnode_property_read_string(child, "label", &label);
    WARN_ON_ONCE(ret && ret != -EINVAL); /* -EINVAL = not present, ok */
}

/* نقطة 4: عند register software node */
ret = software_node_register(&my_node);
if (WARN_ON(ret)) {
    pr_err("swnode register failed: %d\n", ret);
    return ret;
}

/* نقطة 5: تحقق من reference args */
ret = fwnode_property_get_reference_args(fwnode, "gpios", "#gpio-cells",
                                          0, idx, &args);
if (ret) {
    dev_err(dev, "gpio ref %d: %d\n", idx, ret);
    /* dump_stack لو في doubt من caller chain */
    if (ret == -EINVAL)
        dump_stack();
}
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel State

```bash
# تحقق من إن الـ device "available" في DT يتطابق مع حالة الـ HW
cat /sys/bus/platform/devices/<dev>/of_node/status
# يجب أن يكون "okay" لو الـ device موجود

# تحقق من الـ clock frequency في DT مقابل الـ hardware
cat /sys/bus/platform/devices/<dev>/of_node/clock-frequency
# قارنها مع الـ oscillator الفعلي على الـ board

# تحقق من base address
cat /sys/bus/platform/devices/<dev>/of_node/reg
# يجب يطابق الـ memory map في الـ datasheet

# تحقق interrupt numbers
cat /sys/bus/platform/devices/<dev>/of_node/interrupts
cat /proc/interrupts | grep <dev_name>
```

---

#### 2. Register Dump Techniques

```bash
# باستخدام devmem2 (أسهل)
# اقرأ register على عنوان 0xFF160000
devmem2 0xFF160000 w         # قرأ word (32-bit)
devmem2 0xFF160000 b         # قرأ byte
devmem2 0xFF160004 w 0x1234  # اكتب قيمة

# باستخدام /dev/mem مباشرة (يحتاج CONFIG_STRICT_DEVMEM=n)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xFF160000)
    val = struct.unpack('<I', m.read(4))[0]
    print(f'REG[0]: 0x{val:08X}')
"

# باستخدام io utility (من package io-utils)
io -4 -r 0xFF160000           # قرأ 32-bit register
io -4 -w 0xFF160000 0x00000001 # اكتب register

# على x86 لـ MMIO باستخدام /proc/iomem
cat /proc/iomem | grep <device_name>
```

تفعيل regmap debugfs (لو الـ driver يستخدم regmap):
```bash
ls /sys/kernel/debug/regmap/
cat /sys/kernel/debug/regmap/<dev_addr>/registers
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو بتـ debug **I2C device** مع property مش شغالة صح:

```
Logic Analyzer Setup:
- Channel 0: SDA (I2C data)
- Channel 1: SCL (I2C clock)
- Trigger: falling edge على SCL
- Sample rate: ≥ 10× bus frequency (400kHz bus → 4MHz sample)

شوف:
1. الـ ACK/NACK بعد كل byte
2. الـ address match (7-bit address + R/W bit)
3. الـ register address المرسلة
4. الـ data المقروءة/المكتوبة
```

لو **SPI**:
```
- MOSI, MISO, SCLK, CS# على 4 channels
- Trigger: CS# falling edge
- تحقق من CPOL/CPHA في DT يطابق الـ device datasheet
  (clock-phase, clock-polarity properties)
```

لو **IRQ**:
```
- شيل probe على الـ interrupt line من الـ device
- قارن مع /proc/interrupts counter
- تحقق من edge/level polarity في DT (interrupts property)
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Log | السبب |
|---------------------|-------------------|-------|
| الـ device مش موجود على البورد | `probe failed: -ENODEV` أو `-ENXIO` | base address غلط في DT |
| الـ IRQ غلط | `IRQ N: nobody cared` | interrupt number أو polarity غلط في DT |
| الـ clock مش enabled | `clk_prepare_enable failed: -ENOENT` | missing `clocks` property أو clock provider |
| power supply غلط | `regulator_get: supply 'vdd' not found` | missing `vdd-supply` phandle |
| DMA config خطأ | `dma_request_chan failed: -ENODEV` | `dmas` property غلط في DT |
| الـ I2C address conflict | `i2c: Address 0x50 already in use` | تكرار address في DT |
| endianness خطأ | قراءة قيم مقلوبة من الـ HW | missing `big-endian` property في DT |
| GPIO مش متاح | `gpio: export: invalid GPIO N` | `gpios` property أو gpio controller غلط |

---

#### 5. Device Tree Debugging — تحقق من تطابق DT مع الـ Hardware

```bash
# قارن الـ DTB الـ booted مع الـ DTS source
dtc -I dtb -O dts /sys/firmware/fdt -o /tmp/booted.dts
diff /tmp/booted.dts my_board.dts

# تحقق من وجود الـ node
fdtget /sys/firmware/fdt /soc/i2c@ff160000 status
# المتوقع: okay

# تحقق من الـ compatible string بالضبط
fdtget /sys/firmware/fdt /soc/i2c@ff160000 compatible
# يجب يطابق driver's of_device_id table

# تحقق من الـ reg property (base address + size)
fdtget -t x /sys/firmware/fdt /soc/uart@ff180000 reg
# مثلاً: 0xff180000 0x100

# تحقق من إن الـ DT overlay اتطبق صح
ls /sys/firmware/devicetree/base/
cat /sys/firmware/devicetree/base/__symbols__/<node_alias>

# تحقق من الـ interrupts
fdtget /sys/firmware/fdt /soc/gpio@ff750000 interrupts
# قارن مع الـ GIC/interrupt controller documentation

# فحص phandles للـ references
fdtget -t x /sys/firmware/fdt /soc/i2c@ff160000 clocks
# الـ phandle يجب يشاور على clock provider صح
```

لو بتستخدم **DT overlays**:
```bash
# تحقق من الـ overlays المطبقة
ls /sys/firmware/devicetree/overlays/
# أو
ls /proc/device-tree/

# فعّل OF debug لرؤية الـ overlay merge
echo 1 > /sys/kernel/debug/of/debug
dmesg | grep 'overlay\|fwnode'
```

---

### Practical Commands

#### 1. فحص Device Properties — جاهزة للنسخ

```bash
#!/bin/bash
# === فحص شامل لـ device properties ===

DEV="my_device_name"          # غيّر للاسم الصح
DT_NODE="/soc/i2c@ff160000"   # غيّر للـ DT path

echo "=== [1] Device fwnode type ==="
ls -la /sys/bus/platform/devices/$DEV/firmware_node 2>/dev/null || \
  ls -la /sys/bus/i2c/devices/$DEV/of_node 2>/dev/null

echo "=== [2] DT Properties ==="
for f in /sys/bus/platform/devices/$DEV/of_node/*; do
    echo -n "$(basename $f): "
    cat "$f" 2>/dev/null | xxd | head -2
done

echo "=== [3] compatible string ==="
cat /sys/bus/platform/devices/$DEV/of_node/compatible 2>/dev/null

echo "=== [4] status ==="
cat /sys/bus/platform/devices/$DEV/of_node/status 2>/dev/null

echo "=== [5] software nodes ==="
ls /sys/kernel/software_nodes/ 2>/dev/null

echo "=== [6] deferred devices ==="
cat /sys/kernel/debug/devices_deferred 2>/dev/null

echo "=== [7] IRQs ==="
cat /proc/interrupts | grep $DEV

echo "=== [8] dmesg for device ==="
dmesg | grep -i "$DEV" | tail -20
```

---

#### 2. فعّل Tracing لـ property subsystem

```bash
#!/bin/bash
# === ftrace للـ property functions ===

TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace                    # clear
echo function > $TRACE/current_tracer

# اختار الـ functions
for fn in \
    device_property_read_u32_array \
    device_property_read_string \
    fwnode_property_present \
    software_node_register \
    software_node_unregister \
    fwnode_graph_get_next_endpoint \
    fwnode_get_reference_args; do
    echo "$fn" >> $TRACE/set_ftrace_filter
done

echo 1 > $TRACE/tracing_on

# شغّل اللي تريد تتتبعه
modprobe my_driver

sleep 2
echo 0 > $TRACE/tracing_on

echo "=== Trace Output ==="
cat $TRACE/trace | grep -v '^#' | head -100
```

---

#### 3. مثال Output وكيف تقرأه

```
# مثال output من dmesg بعد تفعيل dynamic debug
[    2.345678] my_driver: device_property_read_u32: 'clock-frequency' = 400000
[    2.345699] my_driver: fwnode_property_present: 'big-endian' NOT found
[    2.345712] swnode: software_node_register: registered 'my-device'
[    2.346001] my_driver: deferred probe: waiting for 'my-clock' supply

# التفسير:
# السطر 1: property اتقرأت بنجاح، قيمتها 400000 (400kHz)
# السطر 2: الـ device مش big-endian → سيستخدم little-endian (الافتراضي)
# السطر 3: software node اتسجل بنجاح
# السطر 4: ⚠ الـ probe اترجأ — clock provider لسه مش ready

# الحل للسطر 4:
# تحقق من الـ clock node في DT:
fdtget /sys/firmware/fdt / '#size-cells'
grep -r 'my-clock' /tmp/current.dts
```

```bash
# مثال output من ftrace
# تقرأه من يسار لـ يمين:
# [timestamp] [CPU] [process-PID] [function_name] <- [caller]

  2.345000: my_driver-123  [001] device_property_read_u32_array <- my_driver_probe
  2.345001: my_driver-123  [001]   fwnode_property_read_int_array <- device_property_read_u32_array
  2.345002: my_driver-123  [001]     of_property_read_variable_u32_array <- fwnode_property_read_int_array
  # → الـ property اتقرأت من DT (of_property_read_*) مش ACPI مش swnode

# لو شفت acpi_dev_get_property في الـ chain → ACPI device
# لو شفت software_node_get_property → software node
```

---

#### 4. فحص software nodes بشكل مفصل

```bash
# اطبع كل software nodes مع properties
for node in /sys/kernel/software_nodes/*/; do
    echo "=== Node: $(basename $node) ==="
    for prop in $node/properties/*/; do
        echo -n "  $(basename $prop): "
        cat "$prop/value" 2>/dev/null | xxd | head -1
    done
done

# تحقق من الـ hierarchy
find /sys/kernel/software_nodes/ -name "parent" -exec cat {} \; 2>/dev/null
```

---

#### 5. فحص fwnode graph (Camera/Display pipelines)

```bash
# اطبع كل الـ endpoints
for ep in /sys/bus/platform/devices/*/of_node/port*/endpoint*; do
    echo "=== Endpoint: $ep ==="
    echo -n "  remote-endpoint: "; cat "$ep/remote-endpoint" 2>/dev/null | xxd | head -1
    echo -n "  bus-type: "; cat "$ep/bus-type" 2>/dev/null
done

# تحقق من الـ V4L2 graph (كاميرا)
media-ctl -d /dev/media0 --print-topology 2>/dev/null | head -50

# تحقق من الـ DRM graph (display)
cat /sys/kernel/debug/dri/0/state 2>/dev/null | head -30
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — UART لا يشتغل بعد bring-up

#### العنوان
**الـ UART driver بيرفض يـ probe على gateway صناعي بسبب property مش موجودة**

#### السياق
فريق bring-up بيشغّل industrial gateway على RK3562. الـ board بتحتوي على RS-485 transceiver متوصل بـ UART2. الـ driver بيحتاج يقرأ property اسمها `rs485-rts-gpio` عشان يتحكم في الـ direction pin.

#### المشكلة
الـ driver بيطلع `probe` بـ `-EINVAL` وملقتش الـ device في `/sys/bus/platform/devices`. الـ kernel log بيقول:

```
serial 20820000.serial: failed to get rs485-rts-gpio
```

#### التحليل
الـ driver بيستخدم:

```c
/* driver code trying to read the GPIO property */
ret = device_property_read_u32(dev, "rs485-rts-gpio", &gpio_num);
if (ret)
    return ret;  /* returns -EINVAL if property missing */
```

الـ flow اللي بيحصل:
1. `device_property_read_u32` بتكلّم `device_property_read_u32_array(dev, propname, val, 1)`.
2. الأخيرة بتوصل لـ `fwnode_property_read_u32_array` عبر `dev_fwnode(dev)`.
3. الـ `fwnode_call_int_op` بيتحقق من `fwnode_has_op(fwnode, property_read_int_array)` — الـ fwnode موجود وصالح.
4. الـ OF backend بيدور على property `rs485-rts-gpio` في الـ DT node ومش لاقيها → بيرجّع `-EINVAL`.

المشكلة مش في الكود، في الـ DT.

#### الحل
```dts
/* arch/arm64/boot/dts/rockchip/rk3562-gateway.dts */
&uart2 {
    status = "okay";
    pinctrl-0 = <&uart2m1_xfer>;

    /* كان ناقص — لازم يتضاف */
    rs485-rts-gpio = <&gpio3 RK_PA4 GPIO_ACTIVE_HIGH>;
    linux,rs485-enabled-at-boot-time;
};
```

بعد إضافة الـ property، `device_property_read_u32` بترجع 0 والـ driver يـ probe بنجاح.

#### الدرس المستفاد
**الـ `device_property_read_*` بترجع `-EINVAL` لما الـ property مش موجودة خالص، مش بس لما القيمة غلط.** لازم دايمًا تفرّق بين الحالتين في الـ driver: استخدم `device_property_present()` الأول لو الـ property اختيارية.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش بيظهر

#### العنوان
**الـ HDMI encoder مش بيلاقي الـ remote endpoint في الـ fwnode graph**

#### السياق
منتج Android TV box بيستخدم Allwinner H616. الـ display pipeline: `DE2 → TCON → HDMI`. الـ driver بيستخدم `fwnode_graph_get_remote_endpoint()` عشان يمشي من الـ TCON port لـ HDMI.

#### المشكلة
شاشة سودا بالكامل. الـ dmesg بيظهر:

```
sun50i-de2 5000000.display-engine: could not get remote endpoint
```

#### التحليل
الـ driver بيستخدم:

```c
/* walking the graph: TCON → HDMI */
ep = fwnode_graph_get_next_endpoint(dev_fwnode(dev), NULL);
remote = fwnode_graph_get_remote_endpoint(ep);
if (!remote) {
    dev_err(dev, "could not get remote endpoint\n");
    return -ENODEV;
}
```

الـ `fwnode_graph_get_next_endpoint` بيدور على node اسمها `endpoint` داخل `ports/port@X` — بيرجع endpoint موجود.

الـ `fwnode_graph_get_remote_endpoint` بتعمل `fwnode_call_ptr_op(fwnode, graph_get_remote_endpoint)` — بيروح يشوف الـ `remote-endpoint` phandle في الـ DT.

المشكلة: الـ `fwnode_graph_is_endpoint()` بيتحقق عن طريق:

```c
static inline bool fwnode_graph_is_endpoint(const struct fwnode_handle *fwnode)
{
    return fwnode_property_present(fwnode, "remote-endpoint");
}
```

الـ DT عنده الـ ports معرّفة غلط — الـ `remote-endpoint` phandle موجود في الـ TCON بس مش موجود في الـ HDMI encoder node.

#### الحل
```dts
/* الـ DT الغلط */
&tcon_top {
    ports {
        port@1 {
            tcon_out_hdmi: endpoint {
                remote-endpoint = <&hdmi_in>;  /* موجود */
            };
        };
    };
};

&hdmi {
    ports {
        port@0 {
            hdmi_in: endpoint {
                /* remote-endpoint ناقصة هنا! */
            };
        };
    };
};

/* الـ DT الصح */
&hdmi {
    ports {
        port@0 {
            hdmi_in: endpoint {
                remote-endpoint = <&tcon_out_hdmi>;  /* لازم تكون هنا */
            };
        };
    };
};
```

#### الدرس المستفاد
**الـ `fwnode_graph_is_endpoint()` بتشوف `remote-endpoint` property فقط.** لو الـ endpoint في أي طرف من الـ graph مش فيه الـ phandle، الـ graph traversal بيفشل بصمت. الـ graph لازم يكون bidirectional في الـ DT.

---

### السيناريو 3: IoT Sensor على STM32MP1 — Driver بيـ crash عند استخدام software_node

#### العنوان
**NULL pointer dereference عند تسجيل `software_node` لحساس I2C بدون DT**

#### السياق
board مخصصة بدون DT كامل — الـ engineer بيحاول يوصف خصائص حساس temperature (TMP117) عبر `software_node` في الـ driver بدل DT، عشان الـ board lacked proper firmware support.

#### المشكلة
الـ kernel بيـ crash بـ NULL pointer dereference فور ما الـ module يتحمّل:

```
BUG: kernel NULL pointer dereference at 0000000000000008
```

#### التحليل
الكود الغلط:

```c
static const struct property_entry tmp117_props[] = {
    PROPERTY_ENTRY_U32("ti,conversion-avg", 8),
    PROPERTY_ENTRY_STRING("label", "board-temp"),
    /* نسي الـ terminator! */
};

static const struct software_node tmp117_node = {
    .name = "tmp117",
    .properties = tmp117_props,
};
```

الـ `property_entries_dup()` بتكلّم `property_entries_free()` — كلاهما بيعتمدوا على array ينتهي بـ **zero terminator** `{}`.

بدون الـ terminator، الـ loop بتكمل في memory عشوائي. الـ struct `property_entry` تحتاج أول field `name` يكون NULL عشان تعرف إن الـ array خلصت:

```c
/* property_entries_dup iterates until name == NULL */
for (p = properties; p->name; p++)
    count++;
/* بدون terminator → p->name بيقرأ garbage → crash */
```

#### الحل
```c
static const struct property_entry tmp117_props[] = {
    PROPERTY_ENTRY_U32("ti,conversion-avg", 8),
    PROPERTY_ENTRY_STRING("label", "board-temp"),
    { }  /* terminator إجباري */
};

/* التسجيل الصحيح */
static int __init tmp117_init(void)
{
    int ret;

    ret = software_node_register(&tmp117_node);
    if (ret)
        return ret;

    return i2c_add_driver(&tmp117_driver);
}

static void __exit tmp117_exit(void)
{
    i2c_del_driver(&tmp117_driver);
    software_node_unregister(&tmp117_node);  /* الترتيب مهم */
}
```

#### الدرس المستفاد
**كل array من `property_entry` لازم تنتهي بـ `{ }` فارغة.** الـ kernel ما عندوش طريقة يعرف طول الـ array غير كده. استخدم دايمًا `PROPERTY_ENTRY_BOOL`, `PROPERTY_ENTRY_U32`, إلخ. بدل ما تملي الـ struct يدويًا.

---

### السيناريو 4: Automotive ECU على i.MX8 — SPI device بيقرأ endianness غلط

#### العنوان
**register values معكوسة في SPI ADC driver بسبب endianness property مش متحوّطة**

#### السياق
ECU في سيارة بيستخدم i.MX8QM. الـ SoC ARM64 little-endian بيتكلم مع ADC خارجي (ADS131M08) عبر SPI. الـ ADC بيرسل data بـ big-endian. الـ driver القديم كان hardcoded `be32_to_cpu()` — لكن في نسخة جديدة الـ developer قرر يدعم الاتجاهين عبر DT property.

#### المشكلة
الـ ADC بيرجع قراءات غلط تمامًا في بعض الـ boards. الفرق بالظبط هو byte-swap.

#### التحليل
الـ developer كتب:

```c
static int ads131_probe(struct spi_device *spi)
{
    struct device *dev = &spi->dev;

    /* check endianness from DT */
    priv->big_endian = device_is_big_endian(dev);
    ...
}
```

الـ `device_is_big_endian` بتعمل:

```c
static inline bool device_is_big_endian(const struct device *dev)
{
    return fwnode_device_is_big_endian(dev_fwnode(dev));
}
```

والـ `fwnode_device_is_big_endian`:

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

المشكلة: الـ CPU هنا little-endian (ARM64 LE build)، والـ DT عنده `native-endian` مش `big-endian`:

```dts
/* DT غلط */
&spi0 {
    ads131m08@0 {
        compatible = "ti,ads131m08";
        native-endian;  /* غلط! CPU is LE, device is BE */
    };
};
```

لأن `CONFIG_CPU_BIG_ENDIAN` مش مفعّل، `native-endian` بيرجع `false` → الـ driver يعامل الـ data كـ LE → القراءات غلط.

#### الحل
```dts
/* DT صح */
&spi0 {
    ads131m08@0 {
        compatible = "ti,ads131m08";
        reg = <0>;
        spi-max-frequency = <8000000>;
        big-endian;  /* الصح: الـ device نفسه BE بغض النظر عن الـ CPU */
    };
};
```

```bash
# تحقق في runtime
cat /sys/kernel/debug/spi0.0/properties  # لو مفعّل debugfs
```

#### الدرس المستفاد
**`native-endian` تعني "نفس endianness الـ CPU"، مش "الـ device big-endian".** لو الـ peripheral دايمًا BE، استخدم `big-endian` صريحة. `native-endian` مفيدة بس للـ devices اللي بتعكس endianness حسب الـ SoC (زي بعض الـ PCIe controllers).

---

### السيناريو 5: Custom Board Bring-up على AM62x — I2C child nodes مش بتتقرأ

#### العنوان
**`device_for_each_child_node` بيرجع فاضي رغم وجود child nodes في DT**

#### السياق
فريق bring-up لـ AM62x-based HMI panel. الـ I2C bus فيه touch controller (GT911) و EEPROM (AT24). الـ driver الخاص بالـ HMI بيحاول يمشي على child nodes عشان يعرف الـ peripherals الموجودة dynamically.

#### المشكلة
الـ driver بيـ probe بنجاح بس الـ child nodes count بيرجع 0:

```c
/* في الـ driver */
unsigned int count = device_get_child_node_count(dev);
dev_info(dev, "found %u child nodes\n", count);
/* Output: found 0 child nodes */
```

#### التحليل
الـ DT عنده:

```dts
&i2c1 {
    status = "okay";
    clock-frequency = <400000>;

    hmi_ctrl: hmi@50 {
        compatible = "mycompany,hmi-ctrl";
        reg = <0x50>;

        gt911@14 {
            compatible = "goodix,gt911";
            reg = <0x14>;
            /* status مش موجودة = disabled by default؟ */
        };

        at24@57 {
            compatible = "atmel,24c32";
            reg = <0x57>;
            status = "disabled";  /* عمدًا disabled */
        };
    };
};
```

الكود:

```c
device_get_child_node_count(dev)
    → fwnode_get_child_node_count(dev_fwnode(dev))
```

الـ `fwnode_get_child_node_count` بيستخدم `fwnode_get_next_child_node` مش `fwnode_get_next_available_child_node` — يعني المفروض يعدّ كل الـ nodes بغض النظر عن `status`.

المشكلة الحقيقية: الـ `dev` اللي بيتعمله `dev_fwnode()` هو الـ `hmi_ctrl` device — بس لسه `i2c_new_client_device` ما اشتغلش على child nodes، فالـ fwnode الفعلي للـ `hmi_ctrl` هو **I2C client node** مش الـ I2C bus node.

الـ `fwnode_get_next_child_node` بيرجع child nodes للـ fwnode نفسه صح — المشكلة إن الـ `gt911` node ما عندوش `status = "okay"` صريح.

في بعض versions الـ OF layer بيعامل absent `status` كـ "okay"، بس الـ driver كان بيستخدم `fwnode_for_each_available_child_node` مش `fwnode_for_each_child_node`:

```c
/* الكود الفعلي في الـ driver — غلط */
fwnode_for_each_available_child_node(dev_fwnode(dev), child) {
    /* gt911 بدون status مش بيظهر هنا لو الـ OF layer conservative */
    count++;
}

/* الصح: استخدم fwnode_for_each_child_node لو تريد كل الـ nodes */
fwnode_for_each_child_node(dev_fwnode(dev), child) {
    if (!fwnode_device_is_available(child))
        continue;  /* skip ما هو صراحة disabled */
    count++;
    fwnode_handle_put(child);  /* مهم: تحرير الـ reference */
}
```

أو الأصح:

```dts
/* إضافة status صريح للـ gt911 */
gt911@14 {
    compatible = "goodix,gt911";
    reg = <0x14>;
    status = "okay";  /* صريح */
};
```

#### الدرس المستفاد
**فرّق بين `fwnode_for_each_child_node` (كل الـ nodes) و`fwnode_for_each_available_child_node` (اللي `status = "okay"` بس).** كمان، لو استخدمت `fwnode_for_each_child_node` من غير الـ `_scoped` variant، لازم تعمل `fwnode_handle_put(child)` لو خرجت من الـ loop بـ `break` أو `return` عشان ما تعملش resource leak.

```c
/* الطريقة الآمنة مع scoped variant — لا تحتاج put يدوي */
fwnode_for_each_available_child_node_scoped(dev_fwnode(dev), child) {
    /* child بيتحرر تلقائيًا بعد كل iteration أو عند exit */
    process_child(child);
}
```
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

دي أهم المقالات اللي بتغطي موضوع `property.h` والـ fwnode framework بشكل مباشر:

| المقال | الأهمية |
|--------|---------|
| [Add ACPI _DSD and unified device properties support](https://lwn.net/Articles/612062/) | النقطة الأساسية — بيوصف المبادرة الأولى اللي Rafael Wysocki أطلقها عشان توحّد قراءة الـ properties من ACPI ومن Device Tree |
| [device property: Introducing software nodes](https://lwn.net/Articles/770825/) | بيشرح إزاي `struct software_node` اتولدت كـ fwnode مستقل يحلّ محل `property_set` القديم |
| [ACPI graph support](https://lwn.net/Articles/718184/) | بيغطي إضافة دعم الـ fwnode graph (ports/endpoints) لـ ACPI _DSD |
| [Unified fwnode endpoint parser](https://lwn.net/Articles/737780/) | بيوصف توحيد محلّل الـ endpoint بين DT وACPI في الـ media subsystem |
| [introduce fwnode in the I2C subsystem](https://lwn.net/Articles/889236/) | مثال حي على تبنّي الـ fwnode API في subsystem كامل |
| [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/) | خلفية مهمة عن device model والعلاقة بين `struct device` والـ fwnode |
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | يشرح تاريخ الـ platform data وإزاي انتقلنا لفكرة الـ properties |

---

### توثيق الـ kernel الرسمي

المسارات دي موجودة في شجرة kernel مباشرةً أو على `docs.kernel.org`:

```
Documentation/driver-api/device_connection.rst
Documentation/driver-api/infrastructure.rst
Documentation/firmware-guide/acpi/dsd/properties-rules.rst
Documentation/firmware-guide/acpi/dsd/graph.rst
Documentation/devicetree/bindings/
```

**روابط مباشرة:**

- [Device drivers infrastructure — kernel docs](https://docs.kernel.org/driver-api/infrastructure.html)
  فيها توثيق `fwnode_handle`، `set_primary_fwnode()`، و`device_property_*()`.

- [Graphs — ACPI DSD kernel docs](https://www.kernel.org/doc/html/latest/firmware-guide/acpi/dsd/graph.html)
  تفاصيل بناء الـ fwnode graph (ports، endpoints، remote-endpoint).

- [V4L2 fwnode kAPI](https://docs.kernel.org/driver-api/media/v4l2-fwnode.html)
  مثال كامل على استخدام الـ fwnode graph API في subsystem حقيقي.

---

### Commits مهمة في kernel git

```bash
# الـ commit الأصلي اللي عمل property.h
git log --oneline --all -- include/linux/property.h | tail -5

# أو تبحث في cgit:
# https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/property.h
```

| الوصف | المرجع |
|-------|--------|
| الـ commit الأول للـ unified device properties (2014) | Rafael Wysocki + Mika Westerberg — Intel |
| إضافة `software_node` | Heikki Krogerus — [patchwork](https://patchwork.kernel.org/project/linux-acpi/patch/1496741861-8240-5-git-send-email-sakari.ailus@linux.intel.com/) |
| إضافة `fwnode_device_is_available()` | Sakari Ailus — [patchwork](https://patchwork.kernel.org/project/linux-acpi/patch/1496741861-8240-5-git-send-email-sakari.ailus@linux.intel.com/) |
| إضافة `fwnode_get_name` op | [mail-archive](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2091140.html) |
| `software_node` + `fwnode_graph*()` | [mail-archive](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2312231.html) |

**للبحث في git مباشرةً:**
```bash
git log --oneline --follow include/linux/property.h
git log --oneline --follow drivers/base/property.c
```

---

### نقاشات الـ Mailing List

- **[PATCH v5 09/12] Driver core: Unified interface for firmware node properties**
  https://lists.gt.net/linux/kernel/2028866
  — النقاش الأصلي اللي وضع الأساس للـ API

- **[PATCH] driver core: Implement device property accessors through fwnode ones**
  https://linux.kernel.narkive.com/pNOj9vdt/patch-driver-core-implement-device-property-accessors-through-fwnode-ones
  — بيوضح إزاي `device_property_*()` اتحوّلت لـ wrappers حول `fwnode_property_*()`

- **[PATCH v1 08/13] device property: Fallback to secondary fwnode**
  https://linux.kernel.narkive.com/XO6y722J/patch-v1-08-13-device-property-fallback-to-secondary-fwnode-if-primary-misses-the-property
  — يشرح فكرة الـ primary/secondary fwnode

---

### Slides ومواد مؤتمرات

- **Unified Device Properties Interface for ACPI and Device Trees** — Rafael J. Wysocki (Linux Foundation Events)
  https://events.static.linuxfound.org/sites/events/files/slides/unified_properties_API_0.pdf
  — أفضل عرض شامل للمشروع ده من صاحبه

---

### صفحات eLinux.org

| الصفحة | الفائدة |
|--------|---------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع سريع لكل properties الـ DT الشائعة |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | يوضح الـ `#xxx-cells`، الـ phandles، والـ references |
| [Device Tree Linux](https://elinux.org/Device_Tree_Linux) | الـ `status` property وكيفية enable/disable الـ nodes |
| [Device Tree Presentations](https://elinux.org/Device_Tree_Presentations) | مجموعة presentations من مؤتمرات مختلفة |

---

### صفحات KernelNewbies.org

| الصفحة | ما يخص الموضوع |
|--------|----------------|
| [Linux 4.9](https://kernelnewbies.org/Linux_4.9) | أول kernel نضجت فيه الـ fwnode graph API |
| [Linux 5.2](https://kernelnewbies.org/Linux_5.2) | تحسينات الـ software_node |
| [Linux 5.8](https://kernelnewbies.org/Linux_5.8) | تحديثات في device properties والـ firmware nodes |
| [Linux 5.10](https://kernelnewbies.org/Linux_5.10) | `fwnode_for_each_*_scoped` macros وتحسينات cleanup |

---

### كتب مُوصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet، Alessandro Rubini، Greg Kroah-Hartman
- **الفصول المهمة:**
  - Chapter 3: *Char Drivers* — يفهّمك `struct device` الأساسي
  - Chapter 14: *The Linux Device Model* — الـ kobject، sysfs، والعلاقة بين الأجهزة
- **ملاحظة:** الكتاب قديم (2005)، لكن Chapter 14 لا يزال ذا قيمة للفهم العميق للـ device model. الـ fwnode API نفسها جاءت بعد صدوره.
- **رابط مجاني:** https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل المهم:** Chapter 17: *Devices and Modules*
- بيشرح الـ platform device، الـ driver model، وكيفية ربط الـ driver بالـ hardware description

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصول المهمة:**
  - Chapter 6: *System Initialization*
  - Chapter 11: *BusyBox* و*Device Tree* integration
- مثالي لفهم السياق العملي لاستخدام `device_property_read_u32()` وأصدقاؤها في drivers الـ embedded

---

### مصادر الكود المباشرة في الـ kernel

```
include/linux/property.h       ← الـ header الرئيسي (الملف المدروس)
include/linux/fwnode.h         ← تعريف struct fwnode_handle وfwnode_operations
drivers/base/property.c        ← تنفيذ device_property_*() وfwnode_property_*()
drivers/base/swnode.c          ← تنفيذ software_node كاملاً
drivers/acpi/property.c        ← تنفيذ ACPI-specific fwnode operations
drivers/of/property.c          ← تنفيذ OF/DT-specific fwnode operations
```

---

### مصطلحات البحث (Search Terms)

لو عايز تتعمق أكتر، الكلمات دي بتطلع نتائج مفيدة:

```
linux kernel fwnode_handle unified device properties
linux kernel software_node property_entry
linux device property ACPI DT abstraction layer
linux fwnode graph endpoint port
device_property_read_u32 driver example
linux property_entries_dup software node kobject
Rafael Wysocki Mika Westerberg unified properties API
Heikki Krogerus software_node kernel
linux kernel fwnode_operations struct
```
## Phase 8: Writing simple module

### الفكرة

**`device_property_read_string`** هي دالة exported آمنة ومثيرة للاهتمام — كل driver تقريباً يستخدمها وهو بيقرأ properties زي `"compatible"` أو `"label"` من firmware node (DT أو ACPI). هنعمل **kprobe** عليها عشان نشوف أي device بيحاول يقرأ أي property وإيه النتيجة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe_dev_prop.c
 * Hooks device_property_read_string() to log which device reads which property.
 */

/* kernel module basics */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

/* kprobe API */
#include <linux/kprobes.h>

/* struct device definition */
#include <linux/device.h>

/* device property API — gives us the prototype we're probing */
#include <linux/property.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on device_property_read_string to trace firmware property reads");

/* ------------------------------------------------------------------ */
/* Pre-handler: fires just BEFORE device_property_read_string executes */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Argument layout on x86-64 (System V ABI):
     *   rdi = const struct device *dev
     *   rsi = const char *propname
     *   rdx = const char **val  (output pointer — not useful pre-call)
     *
     * On ARM64:
     *   x0 = dev, x1 = propname, x2 = val
     *
     * regs_get_kernel_argument() is arch-agnostic helper.
     */
    const struct device *dev =
        (const struct device *)regs_get_kernel_argument(regs, 0);

    const char *propname =
        (const char *)regs_get_kernel_argument(regs, 1);

    /* dev_name() returns the kobject name — e.g. "0000:00:1f.3" or "gpio-keys" */
    pr_info("kprobe_dev_prop: [%s] reading property \"%s\"\n",
            dev ? dev_name(dev) : "<null>",
            propname ? propname : "<null>");

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* Post-handler: fires just AFTER the function returns                 */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86-64 the return value sits in rax.
     * regs_return_value() abstracts this across arches.
     * 0 = success, negative = error (e.g. -EINVAL, -ENODATA).
     */
    long ret = regs_return_value(regs);

    if (ret == 0)
        pr_info("kprobe_dev_prop: -> property found OK\n");
    else
        pr_info("kprobe_dev_prop: -> not found (err=%ld)\n", ret);
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    /* symbol_name lets the kernel resolve the address at load time */
    .symbol_name = "device_property_read_string",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* Module init / exit                                                  */
/* ------------------------------------------------------------------ */
static int __init devprop_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_dev_prop: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_dev_prop: planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

static void __exit devprop_probe_exit(void)
{
    /*
     * MUST unregister before the module is unloaded — otherwise the
     * handler pointer dangling in kernel memory will cause a panic.
     */
    unregister_kprobe(&kp);
    pr_info("kprobe_dev_prop: removed\n");
}

module_init(devprop_probe_init);
module_exit(devprop_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية للـ module: `MODULE_LICENSE`، `module_init`/`module_exit` |
| `linux/kprobes.h` | `struct kprobe`، `register_kprobe`، `unregister_kprobe`، `regs_get_kernel_argument` |
| `linux/device.h` | `struct device` و`dev_name()` عشان نطبع اسم الـ device |
| `linux/property.h` | بروتوتايب `device_property_read_string` — مش لازم تضمينه للـ kprobe بس بيوضح القصد |

---

#### الـ `handler_pre`

الـ **pre-handler** بيتشغّل قبل ما الدالة الأصلية تنفذ، فـ arguments لسه موجودة في الـ registers. بنستخدم `regs_get_kernel_argument()` عشان الكود يشتغل على x86-64 وARM64 من غير ما نفرّق بينهم manually. بنطبع اسم الـ device والـ property اللي بتتقرأ — ده المهم للـ tracing.

---

#### الـ `handler_post`

الـ **post-handler** بيتشغّل بعد ما الدالة ترجع. `regs_return_value()` بترجع الـ return value من الـ register المناسب للـ arch. بنطبع هل الـ property اتلقت ولا لأ — فيديو كامل للـ call.

---

#### الـ `struct kprobe`

الـ **`symbol_name`** بيخلي الـ kernel يعمل `kallsyms_lookup_name` وقت التحميل ويحط العنوان في `kp.addr` تلقائياً — مش محتاجين hardcode أي address.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتزرع breakpoint في الذاكرة على عنوان الدالة. في الـ exit، `unregister_kprobe` **لازم** تتنادى قبل ما الـ module يتـunload عشان لو الـ kernel اتصل بالـ handler بعد الـ unload هيـcrash فوراً.

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

### تشغيل وتتبع النتائج

```bash
# بناء الـ module
make

# تحميله
sudo insmod kprobe_dev_prop.ko

# متابعة الـ log في real-time
sudo dmesg -w | grep kprobe_dev_prop

# فك التحميل
sudo rmmod kprobe_dev_prop
```

**مثال على output متوقع:**

```
kprobe_dev_prop: planted at ffffffffc0a12340 (device_property_read_string)
kprobe_dev_prop: [0000:00:14.0] reading property "label"
kprobe_dev_prop: -> not found (err=-22)
kprobe_dev_prop: [gpio-keys] reading property "compatible"
kprobe_dev_prop: -> property found OK
```

---

### ملاحظة أمان

الـ **kprobe** على `device_property_read_string` آمن لأن الدالة بتتنادى من process context وفيها كل الـ locking اللازم بالفعل. الـ handler نفسه بس بيقرأ بيانات ومش بيغيّر أي حاجة، فمفيش خطر race condition أو deadlock.
