## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**الـ** `include/linux/of_device.h` هو header يربط بين نظام **Device Tree** (شجرة الأجهزة) وبين **Driver Model** (نموذج التعريفات) في نواة Linux. ينتمي إلى subsystem اسمه **OPEN FIRMWARE AND FLATTENED DEVICE TREE** ويُشرف عليه Rob Herring وSaravana Kannan.

---

### القصة — تخيّل المشكلة

تخيّل أنك تبني جهاز embedded مثل Raspberry Pi أو راوتر. داخل هذا الجهاز يوجد:
- شريحة USB
- شريحة I2C
- شريحة SPI
- شريحة Ethernet

الـ kernel لا يعرف ما هو موجود على اللوحة. لا يوجد BIOS يخبره. الحل هو ملف نصي يُسمى **Device Tree Blob (DTB)** — وثيقة تصف كل جهاز على اللوحة باسمه ومواصفاته.

مثال مبسّط من Device Tree:

```dts
uart0: serial@ff000000 {
    compatible = "arm,pl011";   /* اسم الجهاز */
    reg = <0xff000000 0x1000>;  /* عنوان الذاكرة */
};
```

الكلمة `"arm,pl011"` هي **compatible string** — بطاقة هوية الجهاز.

في الجانب الآخر، كل driver يُعلن عن قائمة الأجهزة التي يدعمها:

```c
static const struct of_device_id pl011_ids[] = {
    { .compatible = "arm,pl011" },  /* أنا أدعم هذا الجهاز */
    { }
};
```

**السؤال الجوهري:** كيف يعرف الـ kernel أي driver يخدم أي جهاز في شجرة الأجهزة؟

الجواب هو هذا الملف.

---

### ماذا يفعل `of_device.h`؟

يوفر الـ **API** الذي يحل سؤالاً واحداً: **هل هذا الـ driver متوافق مع هذا الـ device؟**

| الدالة | المهمة |
|--------|--------|
| `of_match_device()` | تبحث في قائمة `of_device_id` وتُعيد أول تطابق بين device وdriver |
| `of_driver_match_device()` | wrapper سريع: هل driver معيّن يتطابق مع device؟ (true/false) |
| `of_device_modalias()` | تستخرج **modalias** — سلسلة نصية تعرّف الجهاز، يستخدمها udev لتحميل الـ module تلقائياً |
| `of_device_uevent()` | تُضيف معلومات OF إلى حدث **uevent** الذي يُرسَل لـ userspace عند اكتشاف device جديد |
| `of_device_uevent_modalias()` | تُضيف الـ modalias تحديداً إلى الـ uevent |
| `of_dma_configure()` | تقرأ من الـ Device Tree إعدادات الـ **DMA** (مثل حدود الذاكرة، IOMMU) وتُطبّقها على الـ device |
| `of_device_make_bus_id()` | تُنشئ اسماً فريداً للـ device مستخرَجاً من عنوانه في شجرة الأجهزة |

---

### لماذا يوجد `#ifdef CONFIG_OF`؟

لأن بعض المعالجات (مثل x86) لا تستخدم Device Tree أصلاً. عندما يكون `CONFIG_OF=n`، كل الدوال تُصبح stubs فارغة لا تفعل شيئاً، وهذا يمنع تعطّل الكود على أي معمارية.

```
CONFIG_OF=y  →  دوال حقيقية تقرأ شجرة الأجهزة
CONFIG_OF=n  →  stubs فارغة (للـ x86 وما شابه)
```

---

### رسم توضيحي — كيف يعمل التطابق

```
  Device Tree (DTB)               Driver
  ┌─────────────────┐           ┌──────────────────────┐
  │ node: uart0     │           │ of_match_table:       │
  │  compatible =   │   ──────► │  { "arm,pl011" }      │
  │  "arm,pl011"    │  match?   │  { "arm,pl022" }      │
  └─────────────────┘           └──────────────────────┘
           │                              │
           └──────────── of_match_device() ──► تُعيد pointer للسطر المتطابق
                                               أو NULL إذا لا يوجد تطابق
```

---

### تدفق udev — كيف يُحمَّل الـ module تلقائياً

```
1. Bootloader يُمرّر DTB للـ kernel
2. الـ kernel يقرأ DTB ويُنشئ device nodes
3. لكل node: يُطلق uevent يحمل MODALIAS="of:Nxxx..."
4. udev في userspace يقرأ MODALIAS
5. modprobe يُحمّل الـ module المناسب
6. الـ module يُسجّل driver يحمل of_match_table متطابقة
7. of_match_device() تُؤكد التطابق → probe() يُستدعى
```

---

### الملفات المرتبطة التي يجب معرفتها

| الملف | الدور |
|-------|-------|
| `include/linux/of.h` | التعريفات الأساسية: `struct device_node`، `struct property`، كل دوال قراءة الـ tree |
| `include/linux/of_platform.h` | ربط شجرة الأجهزة بالـ platform bus وإنشاء `platform_device` |
| `include/linux/of_address.h` | تحويل عناوين الـ Device Tree إلى عناوين فيزيائية |
| `include/linux/of_irq.h` | استخراج إعدادات المقاطعات (interrupts) من شجرة الأجهزة |
| `include/linux/of_dma.h` | واجهة DMA المرتبطة بشجرة الأجهزة |
| `include/linux/mod_devicetable.h` | يعرّف `struct of_device_id` نفسها |
| `include/linux/device/driver.h` | يعرّف `struct device_driver` الذي يحمل `of_match_table` |
| `drivers/of/device.c` | التنفيذ الفعلي لكل الدوال المُعلنة في هذا الـ header |
| `drivers/of/base.c` | منطق `of_match_node()` والبحث في شجرة الأجهزة |
| `drivers/of/platform.c` | يُنشئ `platform_device` من كل node في الـ DTB |
| `drivers/of/fdt.c` | قراءة وتفسير ملف DTB الثنائي |
| `drivers/of/address.c` | ترجمة عناوين الـ Device Tree |
| `drivers/of/irq.c` | معالجة إعدادات المقاطعات |
## Phase 2: شرح الـ OF Device Framework

### المشكلة: كيف يعرف الـ kernel أي driver يُشغّل لأي جهاز؟

في العالم المضمّن (embedded)، الأجهزة المتصلة بـ SoC (مثل I2C، SPI، GPIO controllers، UARTs) لا تُكتشف تلقائياً كما يحدث مع PCI أو USB. لا يوجد بروتوكول enumeration — الـ CPU ببساطة لا يعرف ما هو موصول على الـ bus.

**السؤال الجوهري:** كيف تُخبر الـ kernel بأن هناك sensor حرارة من نوع `tmp102` موصول على الـ I2C bus رقم 1، عنوانه `0x48`، ومتصل بـ interrupt line رقم 23؟

### الحل: الـ Device Tree + Open Firmware (OF) Framework

**الـ Device Tree** هو ملف بيانات يصف الـ hardware layout بشكل مستقل عن الـ kernel code. يُكتب بصيغة DTS (Device Tree Source)، ويُترجم إلى DTB (Device Tree Blob) ثنائي، ويُمرَّر من الـ bootloader إلى الـ kernel عند الإقلاع.

**الـ OF Device Framework** هو الطبقة البرمجية داخل الـ kernel المسؤولة عن:
1. قراءة وتحليل الـ DTB في الذاكرة
2. بناء شجرة من الـ `device_node` structs تمثّل كل node في الـ DT
3. ربط كل node بـ driver مناسب عبر آلية المطابقة (matching)
4. تمرير معلومات الـ DMA وغيرها من الـ resources للـ driver

---

### البنية الكبرى للنظام

```
+--------------------------+
|      Bootloader          |
|  (U-Boot / GRUB / etc.)  |
|  يُحمّل DTB إلى الذاكرة  |
+-----------+--------------+
            |
            | DTB blob (binary)
            v
+-----------+--------------+
|    OF / DT Core          |  <--- unflatten_dt_node()
|  يبني شجرة device_node   |       of_root
|                          |
|  /soc                    |
|   /i2c@40003000          |
|     /tmp102@48           |
+-----------+--------------+
            |
            | of_platform_populate()
            v
+-----------+--------------+
|   Platform Bus / I2C Bus |
|  يُنشئ struct device     |
|  لكل node في الـ DT      |
+-----------+--------------+
            |
            | driver_register() + bus matching
            v
+-----------+--------------+
|   Driver Matching        |  <--- of_match_device()
|  يُقارن compatible string |       of_driver_match_device()
|  في الـ DT مع             |
|  of_match_table في driver|
+-----------+--------------+
            |
            | match found!
            v
+-----------+--------------+
|   Driver probe()         |
|  يأخذ struct device *    |
|  ويبدأ تهيئة الـ hardware |
+--------------------------+
```

---

### التشبيه الواقعي: دليل الهاتف الحكومي

تخيّل مبنى حكومياً (الـ SoC) مليء بالموظفين (الأجهزة). لا أحد يُعلن عن نفسه.

**الـ Device Tree** = دليل المبنى المطبوع: "الغرفة 301 = محاسبة، الغرفة 302 = موارد بشرية..."

| عنصر التشبيه | المقابل الحقيقي |
|---|---|
| دليل المبنى | ملف DTB في الذاكرة |
| رقم الغرفة | عنوان الـ device (reg property) |
| اسم الغرفة / وظيفتها | الـ compatible string |
| موظف جديد يبحث عن غرفته | driver يبحث عن device |
| مدير يطابق المهارات بالوظائف | bus matching عبر `of_match_device()` |
| بطاقة الدخول | الـ `of_device_id` entry في الـ match table |
| عقد التوظيف | استدعاء `probe()` بعد المطابقة الناجحة |
| إشعار udev بتوظيف موظف جديد | `of_device_uevent()` + modalias |

الـ HR (human resources) = `of_match_device()`: تقرأ متطلبات الوظيفة (compatible في DT) وسيرة المتقدم (of_match_table في driver) وتقرر إن كانت مطابقة.

---

### الـ Core Abstractions: الهياكل المحورية

#### 1. `struct device_node` — تمثيل node واحدة في الـ DT

```c
struct device_node {
    const char *name;           /* اسم الـ node مثل "i2c" */
    phandle phandle;            /* معرّف فريد للإشارة إليها من nodes أخرى */
    const char *full_name;      /* المسار الكامل: /soc/i2c@40003000 */
    struct fwnode_handle fwnode;/* واجهة موحّدة مع firmware (OF أو ACPI) */

    struct property *properties;/* قائمة connected properties */
    struct device_node *parent; /* الـ node الأب */
    struct device_node *child;  /* أول child node */
    struct device_node *sibling;/* الـ node التالية على نفس المستوى */
};
```

**ملاحظة على الـ `fwnode_handle`:** هذا abstraction طبقة أعلى — يُوحّد بين OF (Device Tree) وACPI وغيرها. الـ kernel الحديث يفضّل التعامل عبر `fwnode` بدلاً من `device_node` مباشرةً لضمان التوافق مع منصات غير DT.

#### 2. `struct of_device_id` — بطاقة هوية المطابقة

```c
struct of_device_id {
    char name[32];        /* نادراً يُستخدم — مطابقة باسم الـ node */
    char type[32];        /* نادراً يُستخدم — device_type property */
    char compatible[128]; /* الأهم — مثل "ti,tmp102" أو "arm,pl011" */
    const void *data;     /* بيانات خاصة بالـ driver لكل entry */
};
```

الـ `compatible` string هي المفتاح. تتبع الصيغة: `"vendor,model"`. مثال:

```
/* في ملف DTS */
tmp102@48 {
    compatible = "ti,tmp102";
    reg = <0x48>;
};

/* في الـ driver */
static const struct of_device_id tmp102_of_match[] = {
    { .compatible = "ti,tmp102", .data = &tmp102_config },
    { }  /* sentinel — نهاية الجدول */
};
```

#### 3. `struct device_driver` والـ `of_match_table`

```c
struct device_driver {
    const char *name;
    const struct bus_type *bus;
    /* ... */
    const struct of_device_id *of_match_table; /* <-- جدول المطابقة */
    /* ... */
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    /* ... */
};
```

الـ driver يعلن عن نفسه عبر `of_match_table` — قائمة من الـ `compatible` strings التي يدعمها.

---

### تدفق المطابقة: `of_match_device()` و `of_driver_match_device()`

```
Driver registers with of_match_table:
  { "ti,tmp102", ... }
  { "ti,tmp101", ... }
  { }

Device has DT node with:
  compatible = "ti,tmp102"

                    of_driver_match_device(dev, drv)
                              |
                              v
                    of_match_device(drv->of_match_table, dev)
                              |
                              | يستخرج الـ compatible string من dev->of_node
                              | يمشي على جدول of_match_table entry بعد entry
                              | يُقارن strings
                              v
                    match found → يُعيد pointer للـ entry المطابقة
                    (أو NULL إذا لم يجد)
```

```c
/* الاستخدام في bus matching code */
static inline int of_driver_match_device(struct device *dev,
                                         const struct device_driver *drv)
{
    /* يُعيد 1 إذا وُجد match، 0 إذا لم يوجد */
    return of_match_device(drv->of_match_table, dev) != NULL;
}
```

الـ bus layer (I2C bus، platform bus، SPI bus...) يستدعي هذه الدالة عند تسجيل كل device أو driver جديد للبحث عن match.

---

### آلية الـ uevent والـ Modalias

عندما يُضاف device جديد لـ sysfs، يُرسل الـ kernel **uevent** لـ userspace (عبر udev/mdev). هذا الـ event يحمل الـ **modalias** — سلسلة نصية تُعرّف الـ device بشكل فريد.

```c
extern void of_device_uevent(const struct device *dev,
                              struct kobj_uevent_env *env);
extern int of_device_uevent_modalias(const struct device *dev,
                                     struct kobj_uevent_env *env);
extern ssize_t of_device_modalias(struct device *dev, char *str, ssize_t len);
```

مثال على الـ modalias لـ DT device:
```
of:NtmpC00Sti,tmp102
```
يفككها udev ليحمّل الـ module المناسب تلقائياً.

```
Device added to sysfs
        |
        v
of_device_uevent()
        |
        | يضيف MODALIAS=of:N<name>T<type>C<compatible>
        v
udev receives uevent
        |
        | modprobe of:NtmpC00Sti,tmp102
        v
Kernel loads tmp102 module automatically
```

---

### تهيئة الـ DMA: `of_dma_configure()`

```c
int of_dma_configure_id(struct device *dev,
                         struct device_node *np,
                         bool force_dma,
                         const u32 *id);

static inline int of_dma_configure(struct device *dev,
                                   struct device_node *np,
                                   bool force_dma)
{
    return of_dma_configure_id(dev, np, force_dma, NULL);
}
```

هذه الدالة تقرأ من الـ DT خصائص مثل `dma-ranges` و `#address-cells` لتُهيّئ:
- **IOMMU mapping** إذا وُجد IOMMU
- **DMA mask** للـ device (كم bit يدعم للـ DMA addresses)
- **DMA coherency** حسب الـ platform

مطلوبة من bus drivers (مثل platform_bus أو PCI) بعد إنشاء الـ device مباشرةً.

---

### البنية الكاملة: كيف تتصل الـ structs ببعض

```
struct device_driver
┌─────────────────────────────┐
│ name: "tmp102"              │
│ bus: &i2c_bus_type          │
│ of_match_table ─────────────┼──→ of_device_id[]
│ probe: tmp102_probe()       │     ┌──────────────────────────┐
│ remove: tmp102_remove()     │     │ compatible: "ti,tmp102"  │
└─────────────────────────────┘     │ data: &tmp102_cfg        │
                                    ├──────────────────────────┤
                                    │ compatible: "ti,tmp101"  │
                                    │ data: &tmp101_cfg        │
                                    ├──────────────────────────┤
                                    │ { }  ← sentinel          │
                                    └──────────────────────────┘

struct device
┌─────────────────────────────┐
│ of_node ────────────────────┼──→ struct device_node
│ driver ─────────────────────┼──→ struct device_driver (بعد المطابقة)
│ bus: &i2c_bus_type          │     ┌──────────────────────────┐
│ ...                         │     │ full_name: "/soc/i2c/    │
└─────────────────────────────┘     │  tmp102@48"              │
                                    │ properties ──────────────┼→ property list
                                    │  compatible="ti,tmp102"  │
                                    │  reg=<0x48>              │
                                    │ fwnode ──────────────────┼→ fwnode_handle
                                    └──────────────────────────┘
```

---

### ما يملكه الـ OF Framework مقابل ما يُفوّضه للـ drivers

| المسؤولية | OF Framework | الـ Driver |
|---|---|---|
| بناء شجرة الـ `device_node` | نعم — `unflatten_device_tree()` | لا |
| قراءة الـ compatible string للمطابقة | نعم — `of_match_device()` | يُعلن عنها فقط في `of_match_table` |
| إرسال uevent مع modalias | نعم — `of_device_uevent()` | لا |
| تهيئة DMA parameters من DT | نعم — `of_dma_configure()` | يستدعيها فقط |
| إعطاء bus ID للـ device | نعم — `of_device_make_bus_id()` | لا |
| قراءة properties خاصة (IRQ, clock, GPIO) | لا — يُوفّر APIs فقط | نعم عبر `of_get_property()`, `of_irq_get()`, ... |
| تهيئة الـ hardware الفعلية | لا | نعم في `probe()` |
| إدارة الـ resources (request/release) | لا | نعم |

---

### `CONFIG_OF` Guard: التصميم الدفاعي

كل الـ API مُلفوفة بـ `#ifdef CONFIG_OF`:

```c
#ifdef CONFIG_OF
extern const struct of_device_id *of_match_device(...);
/* ... real implementations ... */
#else
static inline int of_driver_match_device(...) { return 0; }
static inline int of_dma_configure(...) { return 0; }
/* ... stubs that do nothing or return -ENODEV ... */
#endif
```

هذا يسمح للـ kernel بالبناء على منصات لا تستخدم Device Tree (مثل بعض x86 embedded boards التي تعتمد على ACPI فقط) دون أي تغيير في كود الـ drivers.

---

### `of_device_make_bus_id()`: كيف يأخذ الـ device اسمه في sysfs

```c
void of_device_make_bus_id(struct device *dev);
```

تقرأ هذه الدالة الـ `reg` property من الـ DT node وتُنشئ الـ bus ID المعروض في sysfs. مثال:

```
DT node:  i2c@40003000 { reg = <0x40003000 0x1000>; }
→ bus ID: "40003000.i2c"
→ sysfs:  /sys/bus/platform/devices/40003000.i2c
```

هذا يُسهّل ربط الـ device المرئي في sysfs بالـ node المقابلة في الـ DT.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### 0. الـ Flags والـ Config Options — Cheatsheet

#### الـ CONFIG Options المؤثرة في الملف

| Config Option       | التأثير                                                                 |
|---------------------|-------------------------------------------------------------------------|
| `CONFIG_OF`         | يُفعِّل كامل منطق الـ OF device matching — بدونه كل الدوال ترجع `0` أو `NULL` |
| `CONFIG_OF_DYNAMIC` | يُفعِّل reference counting حقيقي لـ `device_node` (get/put)           |
| `CONFIG_OF_KOBJ`    | يربط `device_node` بـ kobject لظهوره في sysfs                         |
| `CONFIG_OF_PROMTREE`| يُضيف `unique_id` لكل property لدعم PROM tree                         |
| `CONFIG_SPARC`      | تفعيل كود خاص بمعمارية SPARC (irq_trans, unique_id)                   |

#### الـ Flags الخاصة بـ `device_node._flags`

| Flag Macro              | القيمة | المعنى                                              |
|-------------------------|--------|-----------------------------------------------------|
| `OF_DYNAMIC`            | 1      | الـ node والـ properties مُخصَّصة بـ `kmalloc`     |
| `OF_DETACHED`           | 2      | الـ node مفصول عن شجرة الجهاز                      |
| `OF_POPULATED`          | 3      | تم إنشاء الـ device الخاص بهذا الـ node مسبقاً      |
| `OF_POPULATED_BUS`      | 4      | تم إنشاء platform bus للأبناء                      |
| `OF_OVERLAY`            | 5      | الـ node مُخصَّص لـ overlay                         |
| `OF_OVERLAY_FREE_CSET`  | 6      | الـ node في cset جاري تحريره                       |

#### الـ `probe_type` Enum

| القيمة                    | المعنى                                                  |
|---------------------------|---------------------------------------------------------|
| `PROBE_DEFAULT_STRATEGY`  | يعمل بشكل طبيعي سواء sync أو async                     |
| `PROBE_PREFER_ASYNCHRONOUS` | مفيد للأجهزة البطيئة لتسريع الـ boot                 |
| `PROBE_FORCE_SYNCHRONOUS` | يُجبر الـ probe أن يحدث بشكل متزامن مع التسجيل        |

---

### 1. الـ Structs الأساسية

#### `struct of_device_id`
**الهدف:** جدول مطابقة يُعرِّف الأجهزة التي يدعمها الـ driver عبر Open Firmware.

| الحقل        | النوع           | الوصف                                                        |
|--------------|-----------------|--------------------------------------------------------------|
| `name`       | `char[32]`      | اسم الـ node في device tree (نادر الاستخدام اليوم)           |
| `type`       | `char[32]`      | نوع الجهاز (نادر الاستخدام اليوم)                            |
| `compatible` | `char[128]`     | **الحقل الأهم** — يطابق `compatible` property في device tree |
| `data`       | `const void *`  | بيانات خاصة بالـ driver (مثل إعدادات أو flags)              |

**مثال حقيقي:**
```c
static const struct of_device_id my_driver_of_match[] = {
    { .compatible = "vendor,my-device-v2", .data = &cfg_v2 },
    { .compatible = "vendor,my-device-v1", .data = &cfg_v1 },
    { /* sentinel — نهاية الجدول */ }
};
MODULE_DEVICE_TABLE(of, my_driver_of_match);
```

---

#### `struct device_driver`
**الهدف:** يمثل الـ driver في نموذج الجهاز الموحَّد للـ kernel.

| الحقل               | النوع                        | الوصف                                                   |
|---------------------|------------------------------|---------------------------------------------------------|
| `name`              | `const char *`               | اسم الـ driver                                          |
| `bus`               | `const struct bus_type *`    | الـ bus الذي ينتمي إليه الـ driver                     |
| `owner`             | `struct module *`            | الـ module المالك                                       |
| `of_match_table`    | `const struct of_device_id *`| **جدول مطابقة OF** — الرابط الأساسي مع `of_device.h`  |
| `acpi_match_table`  | `const struct acpi_device_id*`| جدول مطابقة ACPI                                       |
| `probe_type`        | `enum probe_type`            | استراتيجية الـ probe (sync/async)                       |
| `probe`             | `int (*)(struct device *)`   | دالة التحقق والربط بالجهاز                             |
| `remove`            | `int (*)(struct device *)`   | دالة فصل الـ driver عن الجهاز                          |
| `sync_state`        | `void (*)(struct device *)`  | مزامنة حالة الجهاز بعد اكتمال الـ consumers            |
| `pm`                | `const struct dev_pm_ops *`  | عمليات إدارة الطاقة                                   |
| `p`                 | `struct driver_private *`    | بيانات خاصة بـ driver core (لا يلمسها الـ driver)      |

---

#### `struct device_node`
**الهدف:** يمثل node واحدة في شجرة الأجهزة (Device Tree) المبنية من DTB.

| الحقل         | النوع                    | الوصف                                                  |
|---------------|--------------------------|--------------------------------------------------------|
| `name`        | `const char *`           | اسم الـ node المختصر                                   |
| `full_name`   | `const char *`           | المسار الكامل في الشجرة (مثل `/soc/uart@10000`)       |
| `phandle`     | `phandle` (u32)          | مُعرِّف فريد للإشارة إليه من nodes أخرى               |
| `fwnode`      | `struct fwnode_handle`   | واجهة firmware node الموحَّدة                          |
| `properties`  | `struct property *`      | قائمة مرتبطة بكل properties الـ node                  |
| `deadprops`   | `struct property *`      | properties محذوفة (محتفظ بها للـ overlays)            |
| `parent`      | `struct device_node *`   | الأب في الشجرة                                        |
| `child`       | `struct device_node *`   | أول ابن في الشجرة                                     |
| `sibling`     | `struct device_node *`   | الأخ التالي (نفس المستوى)                             |
| `_flags`      | `unsigned long`          | الـ flags (OF_DYNAMIC, OF_POPULATED, إلخ)             |
| `kobj`        | `struct kobject`         | تمثيل في sysfs (عند CONFIG_OF_KOBJ)                   |

---

#### `struct property`
**الهدف:** تمثيل property واحدة داخل device tree node (مثل `compatible`, `reg`, `interrupts`).

| الحقل       | النوع              | الوصف                                      |
|-------------|--------------------|--------------------------------------------|
| `name`      | `char *`           | اسم الـ property                           |
| `length`    | `int`              | طول البيانات بالـ bytes                    |
| `value`     | `void *`           | البيانات الخام (big-endian)                |
| `next`      | `struct property *`| الـ property التالية في نفس الـ node       |
| `_flags`    | `unsigned long`    | فقط عند CONFIG_OF_DYNAMIC أو CONFIG_SPARC  |
| `attr`      | `struct bin_attribute` | للظهور في sysfs عند CONFIG_OF_KOBJ     |

---

#### `struct kobj_uevent_env`
**الهدف:** يحمل متغيرات البيئة (environment variables) التي ترسل مع أحداث uevent إلى userspace (مثل `udevd`).

تُستخدم في `of_device_uevent()` و `of_device_uevent_modalias()` لإضافة `MODALIAS=of:T*Cvendor,compatible*` لتحميل الـ module تلقائياً.

---

### 2. مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────┐
│       struct device_driver      │
│─────────────────────────────────│
│  name: "my-driver"              │
│  bus: &platform_bus_type        │
│  of_match_table ──────────────────────────────────────┐
│  probe()                        │                     │
│  remove()                       │                     │
│  p: struct driver_private *     │                     │
└────────────────┬────────────────┘                     │
                 │ يرتبط بـ                             ▼
                 │                      ┌─────────────────────────────┐
                 │                      │    struct of_device_id[]    │
                 │                      │─────────────────────────────│
                 │                      │  [0] compatible="vnd,dev-v2"│
                 │                      │       data=&cfg_v2          │
                 │                      │  [1] compatible="vnd,dev-v1"│
                 │                      │       data=&cfg_v1          │
                 │                      │  [2] {0} ← sentinel         │
                 │                      └─────────────────────────────┘
                 │
                 ▼ يُطابق
┌─────────────────────────────────┐
│         struct device           │
│─────────────────────────────────│
│  of_node ─────────────────────────────────────────────┐
│  driver ──────────────► device_driver                 │
│  bus: &platform_bus_type        │                     │
│  fwnode ──────────────────────────────────────┐       │
└─────────────────────────────────┘             │       │
                                                │       ▼
                                                │  ┌────────────────────────┐
                                                │  │   struct device_node   │
                                                │  │────────────────────────│
                                                │  │ full_name="/soc/uart"  │
                                                │  │ fwnode ◄───────────────┘
                                                │  │ properties ──────────────┐
                                                │  │ parent ──► device_node   │
                                                │  │ child  ──► device_node   │
                                                │  │ sibling──► device_node   │
                                                │  │ _flags: OF_POPULATED     │
                                                │  └──────────────────────────┘
                                                │              │
                                                │              ▼
                                                │  ┌──────────────────────────┐
                                                │  │    struct property        │
                                                │  │──────────────────────────│
                                                │  │ name="compatible"        │
                                                │  │ value="vendor,my-device" │
                                                │  │ next ──► property        │
                                                │  │          (name="reg")    │
                                                │  └──────────────────────────┘
                                                │
                    fwnode_handle (embedded) ◄──┘
                    (واجهة موحَّدة تجمع OF و ACPI)
```

---

### 3. مخطط دورة الحياة — من DTB إلى Driver مرتبط

```
[Boot]
   │
   ▼
DTB مُحمَّل في الذاكرة
   │
   ▼
of_core_init()
   │ يبني شجرة device_node من DTB
   ▼
┌──────────────────────────────────┐
│  device_node لكل node في DTB     │
│  ─ properties مُعبَّأة           │
│  ─ _flags = 0                    │
└──────────┬───────────────────────┘
           │
           ▼ of_platform_populate() أو of_platform_device_create()
┌──────────────────────────────────┐
│  struct platform_device تُنشأ    │
│  ─ dev.of_node ──► device_node   │
│  ─ OF_POPULATED flag يُضبَط      │
└──────────┬───────────────────────┘
           │
           ▼ driver_register(drv)
┌──────────────────────────────────┐
│  kernel يُكرِّر على الأجهزة      │
│  of_driver_match_device() لكل    │
│  جهاز مع كل driver              │
└──────────┬───────────────────────┘
           │
           ▼ of_match_device() ──► تطابق compatible string
┌──────────────────────────────────┐
│  drv->probe(dev) يُستدعى        │
│  الـ driver يبدأ عمله           │
│  of_match_data() للحصول على data│
└──────────┬───────────────────────┘
           │
           ▼ [عند إزالة الجهاز أو unload الـ module]
┌──────────────────────────────────┐
│  drv->remove(dev) يُستدعى       │
│  OF_POPULATED flag يُمسح        │
│  of_node_put() على device_node  │
└──────────────────────────────────┘
```

---

### 4. مخططات تدفق الاستدعاءات

#### أ) تدفق `of_match_device()`

```
الـ driver core يستدعي of_driver_match_device(dev, drv)
  │
  └─► of_match_device(drv->of_match_table, dev)
        │
        ├─► يقرأ dev->of_node  (الـ device_node المرتبط بالجهاز)
        │
        ├─► يقرأ of_node->properties بحثاً عن "compatible"
        │
        └─► يُكرِّر على كل عنصر في of_device_id[]
              │
              ├─ compatible يطابق؟ ──► يُرجع pointer على of_device_id
              │
              └─ لا تطابق؟ ──────────► يُرجع NULL
```

#### ب) تدفق `of_device_uevent_modalias()`

```
udev يطلب uevent عند اكتشاف جهاز جديد
  │
  ▼
of_device_uevent(dev, env)
  │
  ├─► يضيف OF_MODALIAS متغير بيئة
  │     شكله: "of:T<type>C<compatible>A<compatible2>..."
  │
  └─► of_device_uevent_modalias(dev, env)
        │
        └─► add_uevent_var(env, "MODALIAS=of:T*Cvendor,device*")
              │
              └─► udevd يستدعي modprobe بناءً على MODALIAS
                    │
                    └─► الـ module المناسب يُحمَّل تلقائياً
```

#### ج) تدفق `of_dma_configure()`

```
driver يستدعي of_dma_configure(dev, np, force_dma)
  │
  └─► of_dma_configure_id(dev, np, force_dma, NULL)
        │
        ├─► يقرأ "dma-ranges" من device tree node
        │
        ├─► يُحدِّد iommu_ops المناسبة
        │
        ├─► يستدعي arch_setup_dma_ops()
        │     │
        │     └─► يضبط dev->dma_ops
        │
        └─► يضبط dev->dma_mask و dev->coherent_dma_mask
              بناءً على "dma-ranges" في الـ device tree
```

#### د) تدفق `of_device_make_bus_id()`

```
of_platform_device_create() يستدعي of_device_make_bus_id(dev)
  │
  ├─► يقرأ "reg" property من device_node
  │
  ├─► يستخرج العنوان الأول (big-endian u32/u64)
  │
  └─► يضبط dev_set_name(dev, "%s@%llx", node_name, addr)
        │
        └─► الجهاز يظهر في sysfs مثل: /sys/bus/platform/devices/uart@10000
```

---

### 5. استراتيجية الـ Locking

ملف `of_device.h` نفسه **لا يُعرِّف locks** مباشرة، لكن الدوال المُعلَّنة فيه تتفاعل مع آليات القفل في subsystems أخرى:

#### أ) قفل شجرة الـ Device Tree — `of_node_lock`

| Lock | النوع | ما يحميه |
|------|-------|-----------|
| `of_mutex` (داخلي في `of.c`) | `mutex` | قراءة/تعديل `device_node.properties` |
| `devtree_lock` (في بعض الأقواس) | `raw_spinlock_t` | التكرار على شجرة الـ nodes |

- **`of_match_device()`**: تقرأ `dev->of_node` و `properties` — تحتاج أن يكون الـ `device_node` ثابتاً (مضمون لأن `of_node_get()` يزيد refcount).
- **`of_node_get()` / `of_node_put()`**: يستخدمان `kobject_get/put` أو atomic refcount عند `CONFIG_OF_DYNAMIC`.

#### ب) قفل الـ Device — `device_lock`

| متى | من يمسك القفل |
|-----|----------------|
| أثناء `probe()` و `remove()` | الـ driver core يمسك `device_lock(dev)` |
| أثناء `of_dma_configure()` | يجب أن يُستدعى قبل تسجيل الجهاز أو مع القفل |

#### ج) ترتيب الـ Locks (Lock Ordering)

```
[قاعدة: دائماً من الأعلى إلى الأسفل]

device_lock(dev)              ← المستوى الأعلى
    └─► of_mutex / devtree_lock  ← للوصول لـ device_node
            └─► kobject refcount  ← atomic، لا يحتاج قفلاً إضافياً
```

> **تحذير:** لا يجوز مسك `devtree_lock` ثم محاولة مسك `device_lock` — هذا يُسبب deadlock.

#### د) الـ RCU في قراءة الـ Properties

عند `CONFIG_OF_DYNAMIC`، تعديل الـ properties (مثل overlays) يستخدم **RCU**:
- القراء يستخدمون `rcu_read_lock()` ضمنياً
- الكتّاب يستخدمون `synchronize_rcu()` قبل تحرير الـ property القديمة
- هذا يسمح بقراءة `of_match_device()` بدون قفل ثقيل في المسار السريع
## Phase 4: شرح الـ Functions

### جدول الـ Functions — Cheatsheet

| Function | النوع | الغرض الرئيسي |
|---|---|---|
| `of_match_device` | extern | تطابق device مع جدول `of_device_id` |
| `of_driver_match_device` | static inline | wrapper سريع لـ `of_match_device` |
| `of_device_modalias` | extern | استخراج modalias string من الـ device |
| `of_device_uevent` | extern | إضافة OF properties لـ uevent |
| `of_device_uevent_modalias` | extern | إضافة `MODALIAS=` لـ uevent env |
| `of_dma_configure_id` | extern | تهيئة DMA بـ id اختياري من DT |
| `of_dma_configure` | static inline | wrapper لـ `of_dma_configure_id` بدون id |
| `of_device_make_bus_id` | extern | توليد bus ID للـ device من DT path |

---

### المجموعة الأولى: Device-Driver Matching

هذه الدالتان تنتميان لآلية **OF matching** التي يعتمدها driver core لربط driver بـ device عبر مقارنة `compatible` property في الـ device tree مع `of_match_table` في الـ driver.

---

#### `of_match_device`

```c
extern const struct of_device_id *of_match_device(
    const struct of_device_id *matches,
    const struct device *dev);
```

تبحث في مصفوفة `matches` عن أول entry تتطابق مع `dev->of_node`، وتقارن حقول `compatible`, `type`, `name` حسب ما هو مُعبَّأ في كل entry.

**Parameters:**
- `matches`: مصفوفة `of_device_id` منتهية بـ entry فارغة (sentinel).
- `dev`: الـ device المراد مطابقته — يجب أن يحمل `dev->of_node` صالحاً.

**Return:** مؤشر على أول `of_device_id` entry مطابقة، أو `NULL` إن لم يوجد تطابق.

**Key details:**
- تستدعي داخلياً `of_match_node()` التي تعمل على `device_node->properties`.
- لا تحتاج locking؛ الـ device tree read-only بعد الـ boot في معظم الحالات، وأي تعديل ديناميكي محمي بـ `of_mutex` في مستوى أدنى.
- الـ `NULL` return يعني "no match" وليس خطأ.

**Caller context:** يُستدعى من bus match callback (`platform_match`, `spi_match_device`, إلخ) أثناء device/driver binding.

---

#### `of_driver_match_device`

```c
static inline int of_driver_match_device(struct device *dev,
                                          const struct device_driver *drv)
{
    return of_match_device(drv->of_match_table, dev) != NULL;
}
```

**الـ** wrapper الخفيف الذي يُحوّل نتيجة `of_match_device` إلى `bool` integer (0 أو 1).

**Parameters:**
- `dev`: الـ device المراد اختباره.
- `drv`: الـ driver الذي يحمل `of_match_table`.

**Return:** `1` إن كان الـ driver يدعم الـ device، `0` إن لم يكن.

**Key details:**
- عند `!CONFIG_OF`، تُعيد دائماً `0` — لا OF support.
- تُستخدم من `bus->match()` callbacks في سياق atomic أو process context حسب الـ bus.

---

### المجموعة الثانية: Uevent و Modalias

هذه الدوال مسؤولة عن تزويد **udevd** بمعلومات كافية لتحميل الـ module الصحيح عند اكتشاف device جديد، عبر آلية `MODALIAS`.

---

#### `of_device_modalias`

```c
extern ssize_t of_device_modalias(struct device *dev, char *str, ssize_t len);
```

تملأ buffer بـ modalias string مشتقة من أول `compatible` string في `dev->of_node`، بصيغة `of:NnameTtypeCcompat`.

**Parameters:**
- `dev`: الـ device المصدر — يجب أن يكون له `of_node`.
- `str`: buffer الإخراج.
- `len`: حجم الـ buffer بالبايت.

**Return:** عدد البايتات المكتوبة (مثل `snprintf`)، أو قيمة سالبة عند الخطأ (`-ENODEV` إن لم يكن له node).

**Key details:**
- الـ modalias المنتجة تتبع صيغة OF modalias المعيارية، ويستخدمها `modules.alias` للمطابقة.
- تُستدعى من `of_device_uevent_modalias` وأيضاً مباشرةً من sysfs `modalias` attribute handler.

---

#### `of_device_uevent`

```c
extern void of_device_uevent(const struct device *dev,
                              struct kobj_uevent_env *env);
```

تُضيف إلى بيئة الـ uevent متغيرات تعريفية من الـ device tree: `OF_NAME`, `OF_FULLNAME`, `OF_TYPE`, وسلسلة `OF_COMPATIBLE_N` لكل `compatible` entry.

**Parameters:**
- `dev`: الـ device — يجب أن يملك `of_node` صالحاً.
- `env`: بيئة الـ uevent التي يتم تعبئتها بـ `add_uevent_var()`.

**Return:** void — أخطاء `add_uevent_var` تُتجاهل ضمنياً (الـ env buffer قد يمتلئ).

**Key details:**
- تُستدعى من `device_uevent()` في `lib/kobject_uevent.c` أثناء KOBJ_ADD/REMOVE events.
- عند `!CONFIG_OF`، النسخة تكون empty stub لا تفعل شيئاً.
- الـ `OF_COMPATIBLE_N` variables هي ما يعتمدها udev لتحديد الـ module المطلوب.

**Pseudocode:**
```
of_device_uevent(dev, env):
    np = dev->of_node
    if !np: return

    add_uevent_var(env, "OF_NAME=%s", np->name)
    add_uevent_var(env, "OF_FULLNAME=%s", np->full_name)
    if np->type:
        add_uevent_var(env, "OF_TYPE=%s", np->type)

    i = 0
    for each compatible in np->compatible property:
        add_uevent_var(env, "OF_COMPATIBLE_%d=%s", i, compat)
        i++
    add_uevent_var(env, "OF_COMPATIBLE_N=%d", i)
```

---

#### `of_device_uevent_modalias`

```c
extern int of_device_uevent_modalias(const struct device *dev,
                                      struct kobj_uevent_env *env);
```

تُضيف متغير `MODALIAS=of:...` إلى بيئة الـ uevent، وهو المتغير الأساسي الذي يستخدمه udevd لاستدعاء `modprobe`.

**Parameters:**
- `dev`: الـ device المصدر.
- `env`: بيئة الـ uevent.

**Return:** `0` عند النجاح، `1` إن امتلأ الـ buffer (`-ENOMEM` داخلياً يُحوّل إلى `1`)، أو `-ENODEV` إن لم يكن للـ device `of_node`.

**Key details:**
- تستدعي `of_device_modalias()` داخلياً لبناء الـ string.
- تُستدعى من `bus->uevent()` callback (مثل `platform_uevent`).
- عند `!CONFIG_OF`، تُعيد `-ENODEV` مباشرةً.

---

### المجموعة الثالثة: DMA Configuration

هذه الدوال تُهيّئ الـ **DMA subsystem** للـ device استناداً إلى معلومات الـ device tree مثل `dma-ranges`, `#address-cells`, IOMMU nodes.

---

#### `of_dma_configure_id`

```c
int of_dma_configure_id(struct device *dev,
                         struct device_node *np,
                         bool force_dma,
                         const u32 *id);
```

الدالة الرئيسية لتهيئة DMA؛ تُحدد `dma_mask`, `coherent_dma_mask`, وتربط الـ device بالـ IOMMU المناسب إن وُجد، مستندةً إلى `dma-ranges` في الـ device tree.

**Parameters:**
- `dev`: الـ device المراد تهيئته.
- `np`: الـ device tree node المرجعي لقراءة `dma-ranges` (قد يختلف عن `dev->of_node` في بعض الحافلات).
- `force_dma`: إذا كان `true`، يُجبر على تفعيل DMA حتى لو لم تكن هناك `dma-ranges`.
- `id`: معرف اختياري يُمرَّر لـ IOMMU group lookup (مثل PCI RID)، أو `NULL`.

**Return:** `0` عند النجاح، أو error code سالب (`-ENODEV`, `-ENOMEM`, إلخ).

**Key details:**
- تُفعّل `arch_setup_dma_ops()` لربط الـ IOMMU ops بالـ device.
- تقرأ `#address-cells` و `#size-cells` من الـ parent node لحساب نطاق الـ DMA.
- إذا لم تكن `dma-ranges` موجودة و`force_dma=false`، تعود فوراً بـ `0` بدون تهيئة.
- تُستدعى في سياق init/probe — لا يجوز استدعاؤها من interrupt context.

**Pseudocode:**
```
of_dma_configure_id(dev, np, force_dma, id):
    ret = of_dma_get_range(np, &dma_addr, &paddr, &size)
    if ret < 0:
        if !force_dma: return 0   /* no DMA support */
        dma_addr = paddr = 0
        size = max_possible

    dev->bus_dma_limit = dma_addr + size - 1
    arch_setup_dma_ops(dev, dma_addr, size, iommu, coherent)
    set dma_mask and coherent_dma_mask accordingly
    return 0
```

---

#### `of_dma_configure`

```c
static inline int of_dma_configure(struct device *dev,
                                    struct device_node *np,
                                    bool force_dma)
{
    return of_dma_configure_id(dev, np, force_dma, NULL);
}
```

**الـ** inline wrapper الأبسط لـ `of_dma_configure_id` حين لا يُوجد `id` مخصص (الحالة الأكثر شيوعاً).

**Parameters:** نفس `of_dma_configure_id` باستثناء غياب `id`.

**Return:** نفس قيم `of_dma_configure_id`.

**Key details:**
- تُستخدم من معظم الـ platform/SoC drivers مباشرةً في `probe()`.
- عند `!CONFIG_OF`، تُعيد `0` (no-op).

---

### المجموعة الرابعة: Device Identity

---

#### `of_device_make_bus_id`

```c
void of_device_make_bus_id(struct device *dev);
```

تُولّد `dev->bus_id` (المُخزَّن في `dev_name(dev)`) من مسار `dev->of_node` في الـ device tree، مستخدمةً عنوان `reg` property لجعل الـ ID فريداً وقابلاً للتشخيص.

**Parameters:**
- `dev`: الـ device الذي سيُعدَّل `name` الخاص به.

**Return:** void.

**Key details:**
- تُكوّن الـ bus_id بصيغة `nodename.reg_addr`، مثل `serial@11010000` → `11010000.serial`.
- تتصاعد في شجرة الـ nodes لتجنب التكرار إن وُجد تعارض.
- تُستدعى من `of_platform_device_create_pdata()` عند إنشاء الـ platform_device من DT node.
- تُعدّل `dev_set_name(dev, ...)` مباشرةً.

**Pseudocode:**
```
of_device_make_bus_id(dev):
    np = dev->of_node
    while np:
        ret = of_get_property(np, "reg", &sz)
        if ret and sz >= sizeof(__be32):
            addr = be32_to_cpup(ret)
            dev_set_name(dev, "%llx.%s", addr, np->name)
            return
        np = np->parent
    /* fallback: use full_name */
    dev_set_name(dev, "%s", of_node_full_name(dev->of_node))
```

---

### ملاحظة: سلوك `!CONFIG_OF`

عند عدم تفعيل `CONFIG_OF`، جميع الدوال تتحول إلى stubs فارغة أو تُعيد قيماً محايدة:

| Function | القيمة عند `!CONFIG_OF` |
|---|---|
| `of_driver_match_device` | `0` (no match) |
| `of_match_device` | `NULL` |
| `of_device_modalias` | `-ENODEV` |
| `of_device_uevent` | no-op |
| `of_device_uevent_modalias` | `-ENODEV` |
| `of_dma_configure_id` | `0` (success, no-op) |
| `of_dma_configure` | `0` (success, no-op) |
| `of_device_make_bus_id` | no-op |

هذا النمط يضمن أن الكود المستخدم لهذه الدوال يُصرَّف بنجاح على platforms بدون device tree دون الحاجة لـ `#ifdef` في كل مكان.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs ذات الصلة

**الـ debugfs** يوفر نافذة مباشرة على حالة الـ OF/DT داخل الـ kernel. الجذر الافتراضي: `/sys/kernel/debug/`.

| المسار | ما يعرضه | كيفية القراءة |
|---|---|---|
| `/sys/kernel/debug/of_reserved_mem` | مناطق الذاكرة المحجوزة من الـ DT | `cat` مباشرة |
| `/sys/kernel/debug/clk/<clk_name>/clk_summary` | شجرة الـ clocks من الـ DT | `cat /sys/kernel/debug/clk/clk_summary` |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | تعيينات الـ pin المستخرجة من الـ DT | `cat` مباشرة |
| `/sys/kernel/debug/regulator/<name>/state` | حالة الـ regulator المُعرَّف في الـ DT | `cat` مباشرة |
| `/sys/kernel/debug/devices_deferred` | الأجهزة الموقوفة بسبب `-EPROBE_DEFER` | `cat /sys/kernel/debug/devices_deferred` |

```bash
# عرض الأجهزة المؤجلة التي لم يكتمل probe الخاص بها
cat /sys/kernel/debug/devices_deferred

# مثال على الناتج:
# my-device spi0.0: Waiting for supplier regulator-vcc
# → يعني أن of_dma_configure أو of_match_device نجح لكن dependency ناقصة
```

---

#### 2. مدخلات الـ sysfs ذات الصلة

**الـ sysfs** يعكس حالة الـ device/driver binding الناتجة عن `of_match_device` و`of_driver_match_device`.

| المسار | المعنى |
|---|---|
| `/sys/bus/<bus>/drivers/<driver>/` | الأجهزة المرتبطة بالـ driver |
| `/sys/devices/<dev>/of_node` | symlink إلى node الـ DT الخاص بالجهاز |
| `/sys/devices/<dev>/modalias` | الـ modalias المُولَّد بواسطة `of_device_modalias` |
| `/sys/devices/<dev>/uevent` | يحتوي على `OF_COMPATIBLE_*` المُرسَل بواسطة `of_device_uevent_modalias` |
| `/sys/devices/<dev>/driver` | symlink للـ driver المرتبط (يختفي عند فشل الـ match) |
| `/sys/firmware/devicetree/base/` | شجرة الـ DT الكاملة بصيغة sysfs |

```bash
# قراءة الـ modalias لجهاز معين
cat /sys/devices/platform/mydev/modalias
# مثال على الناتج:
# of:NmydevT<nil>Cvendor,mydev-v2
# ↑ هذا ما يُطابقه of_match_device مع حقل compatible في of_device_id

# قراءة الـ DT node الخاص بجهاز
ls -la /sys/devices/platform/mydev/of_node
# → /sys/devices/platform/mydev/of_node -> ../../../firmware/devicetree/base/mydev@40000000

# قراءة خاصية مباشرة من DT عبر sysfs
cat /sys/firmware/devicetree/base/mydev@40000000/compatible
# الناتج: vendor,mydev-v2 (بصيغة null-terminated strings)
```

---

#### 3. الـ ftrace: الـ tracepoints والـ events

**الـ ftrace** يتيح تتبع مسار `of_match_device` وعمليات الـ probe بالكامل.

```bash
# 1. تفعيل tracing لأحداث الـ device/driver
mount -t tracefs tracefs /sys/kernel/tracing  # إذا لم يكن مُركَّباً

# تفعيل أحداث الـ probe
echo 1 > /sys/kernel/tracing/events/device/enable

# تتبع أحداث الـ driver_probe_device
echo 1 > /sys/kernel/tracing/events/devres/enable

# 2. تتبع دالة of_match_device بالاسم
echo 'of_match_device' > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# تشغيل الجهاز أو modprobe الـ driver
modprobe my_driver

echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
```

```bash
# 3. تتبع دالة of_dma_configure_id مع call stack
echo 'of_dma_configure_id' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/options/func_stack_trace
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

مثال على ناتج الـ trace:
```
# tracer: function
#
my_driver-123 [001] .... 1234.567890: of_match_device <-of_driver_match_device
my_driver-123 [001] .... 1234.567891: of_dma_configure_id <-platform_dma_configure
```
التفسير: السطر الأول يُظهر أن `of_driver_match_device` استدعى `of_match_device` — إذا لم يظهر هذا السطر فالـ driver لم يُسجَّل بجدول `of_match_table` صحيح.

---

#### 4. الـ printk والـ dynamic debug

```bash
# تفعيل رسائل of_device الـ dynamic debug (pr_debug/dev_dbg)
echo 'file drivers/of/device.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/platform.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/base/dd.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل رسائل الـ DMA configuration من OF
echo 'file drivers/of/address.c +p' > /sys/kernel/debug/dynamic_debug/control

# لرؤية كل رسائل of بدون استثناء
echo 'module of +p' > /sys/kernel/debug/dynamic_debug/control

# ضبط مستوى الـ printk لرؤية KERN_DEBUG
dmesg -n 7
# أو في kernel cmdline:
# loglevel=8 ignore_loglevel
```

مثال على رسائل `dynamic_debug` المفيدة بعد التفعيل:
```
[    2.345678] of: of_match_device: compatible match: 'vendor,mydev-v2'
[    2.345679] of: of_dma_configure_id: dma_range_map not found, using CPU physical address
```

---

#### 5. خيارات الـ Kernel Config للـ debugging

| CONFIG | الوظيفة |
|---|---|
| `CONFIG_OF` | تفعيل دعم Device Tree (شرط أساسي) |
| `CONFIG_OF_DEBUG` | رسائل debug إضافية من subsystem الـ OF |
| `CONFIG_OF_DYNAMIC` | دعم تعديل الـ DT في وقت التشغيل |
| `CONFIG_OF_UNITTEST` | تشغيل unit tests للـ OF عند boot |
| `CONFIG_DEBUG_DRIVER` | رسائل verbose من driver core بما فيها الـ match |
| `CONFIG_DEBUG_DEVRES` | تتبع عمليات `devm_*` وربطها بالجهاز |
| `CONFIG_PROVE_LOCKING` | اكتشاف deadlocks في locks الـ OF |
| `CONFIG_DMA_API_DEBUG` | تتبع استدعاءات `of_dma_configure_id` والتحقق منها |
| `CONFIG_IOMMU_DEBUG` | debugging لـ DMA mapping عند استخدام IOMMU |
| `CONFIG_KALLSYMS` | لقراءة رموز الـ stack trace بوضوح |
| `CONFIG_EXPERT` + `CONFIG_OF_KOBJ` | إظهار nodes الـ DT في sysfs |

```bash
# فحص التفعيل الحالي
grep -E 'CONFIG_OF|CONFIG_DEBUG_DRIVER|CONFIG_DMA_API' /boot/config-$(uname -r)
```

---

#### 6. أدوات subsystem-specific

```bash
# أداة dtc: فحص وتفكيك الـ DTB
dtc -I dtb -O dts /sys/firmware/fdt > /tmp/current.dts
# → تحويل الـ DTB المُحمَّل حالياً إلى نص قابل للقراءة

# أداة fdtdump
fdtdump /sys/firmware/fdt | grep -A5 "mydev"

# أداة fdt في U-Boot (على الـ target):
# fdt print /mydev@40000000

# udevadm: التحقق من وصول الـ uevent (الذي يُطلقه of_device_uevent_modalias)
udevadm monitor --kernel --property &
modprobe my_driver
# ابحث عن: OF_COMPATIBLE_0=vendor,mydev-v2

# فحص الـ modalias عبر udevadm
udevadm info --attribute-walk /sys/devices/platform/mydev@40000000
# ابحث عن ATTR{modalias}

# مطابقة الـ driver يدوياً
echo "my_driver" > /sys/bus/platform/devices/mydev@40000000/driver_override
echo "mydev@40000000" > /sys/bus/platform/drivers/my_driver/bind

# إلغاء الـ bind
echo "mydev@40000000" > /sys/bus/platform/drivers/my_driver/unbind
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الإصلاح |
|---|---|---|
| `of_dma_configure: failed to get dma range map` | الـ DT لا يحتوي على `dma-ranges` | إضافة `dma-ranges` للـ node أو node الأب |
| `Driver my_driver requests probe deferral` | فشل `of_dma_configure_id` أو dependency غير جاهزة | فحص `devices_deferred` ومراجعة الـ supplier |
| `No OF device for dev` | الجهاز لا يملك `dev->of_node` مُعيَّناً | التأكد من ربط الـ platform_device بـ DT node صحيح |
| `of_match_table is NULL` | الـ driver لم يُعيَّن له `of_match_table` | إضافة جدول `of_device_id` وربطه بـ `driver.of_match_table` |
| `modalias: -ENODEV` | `CONFIG_OF` غير مُفعَّل أو لا يوجد `of_node` | فحص `CONFIG_OF` وتهيئة `dev->of_node` |
| `iommu_dma_ops not set, using direct mapping` | الجهاز لا يستخدم IOMMU رغم وجود IOMMU | مراجعة `iommus` property في الـ DT |
| `dma_direct_map_page: hwaddr ... is not addressable` | مشكلة في نطاق DMA من `of_dma_configure_id` | مراجعة `dma-ranges` في الـ DT |
| `could not get #address-cells` | بنية الـ DT خاطئة | التحقق من أن الـ parent node يحتوي `#address-cells` |

---

#### 8. نقاط الـ dump_stack() والـ WARN_ON() الاستراتيجية

```c
/* في of_match_device — لتتبع لماذا فشلت المطابقة */
const struct of_device_id *of_match_device(
    const struct of_device_id *matches,
    const struct device *dev)
{
    if (!dev->of_node) {
        /* نقطة مثالية: الجهاز بلا DT node */
        WARN_ONCE(!dev->of_node,
                  "%s: device has no of_node\n", dev_name(dev));
        return NULL;
    }
    /* ... */
}

/* في of_dma_configure_id — لتتبع فشل تهيئة الـ DMA */
int of_dma_configure_id(struct device *dev, struct device_node *np,
                        bool force_dma, const u32 *id)
{
    if (!np) {
        WARN_ON(!np);  /* يجب أن يكون np صحيحاً دائماً */
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}

/* في driver probe — لتتبع نتيجة of_driver_match_device */
static int my_driver_probe(struct device *dev)
{
    if (!of_driver_match_device(dev, dev->driver)) {
        dev_warn(dev, "of_driver_match_device failed — compatible mismatch?\n");
        WARN_ON(1);
    }
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

```bash
# الخطوة 1: قراءة الـ compatible من الـ DT الفعلي
cat /sys/firmware/devicetree/base/mydev@40000000/compatible | xxd
# الناتج: 76 65 6e 64 6f 72 2c 6d 79 64 65 76 2d 76 32 00
#         v  e  n  d  o  r  ,  m  y  d  e  v  -  v  2  \0
# تحقق: يجب أن يطابق حقل .compatible في جدول of_device_id في الـ driver

# الخطوة 2: التحقق من عنوان الـ reg
cat /sys/firmware/devicetree/base/mydev@40000000/reg | xxd
# الناتج يجب أن يطابق العنوان الفيزيائي للجهاز

# الخطوة 3: مقارنة الـ DMA mask المُعيَّنة
cat /sys/devices/platform/mydev@40000000/dma_mask
```

---

#### 2. تقنيات قراءة الـ Registers

```bash
# أداة devmem2 (تتطلب CONFIG_DEVMEM و /dev/mem)
# قراءة سجل 32-bit على عنوان 0x40000000
devmem2 0x40000000 w
# الناتج: /dev/mem opened. Memory mapped at address 0x7f8b3e4000.
# Value at address 0x40000000 (0x7f8b3e4000): 0xDEADBEEF

# قراءة عبر /dev/mem مباشرة (بدون devmem2)
python3 -c "
import mmap, os
fd = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
with mmap.mmap(fd, 4096, offset=0x40000000) as m:
    import struct
    print(hex(struct.unpack('<I', m.read(4))[0]))
"

# قراءة باستخدام io utility (من package libgpiod أو busybox)
io -4 -r 0x40000000

# إذا كان الـ driver يُسجِّل regmap، قراءة عبر debugfs
cat /sys/kernel/debug/regmap/40000000.mydev/registers
# الناتج:
# 00000000: deadbeef
# 00000004: 00000001
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**عند debugging مشاكل `of_dma_configure_id` أو `of_match_device`** المرتبطة بـ hardware:

- **الـ I2C/SPI**: تحقق من إشارات SCL/SDA أو SCK/MOSI بعد أن يُكمل `of_driver_match_device` ويبدأ الـ probe، لأن فشل الـ probe يعني عدم وجود traffic على الـ bus.
- **الـ IOMMU / DMA**: ضع probe على خط الـ READY للـ DMA controller؛ إذا لم يُفعَّل بعد `of_dma_configure_id` فهناك مشكلة في الـ DT `dma-ranges`.
- **الـ reset line**: بعض الأجهزة تحتاج GPIO reset مُعرَّفاً في الـ DT؛ راقب خط الـ reset أثناء الـ probe — إذا لم يتغير فالـ driver لم يصل لمرحلة الـ initialization.
- **نقطة القياس**: ضع trigger على خط الـ IRQ المُعرَّف في `interrupts` property؛ إذا لم يصل interrupt بعد التهيئة فالـ DT node لا يُطابق الـ hardware الفعلي.

```
[Target Board]
    |
    +-- GPIO_RST ----[CH1: Oscilloscope]
    +-- IRQ_LINE ----[CH2: Logic Analyzer]
    +-- I2C_SCL  ----[CH3: Logic Analyzer]
    +-- I2C_SDA  ----[CH4: Logic Analyzer]
    |
[Trigger: CH1 Rising Edge → probe started]
[Expect: CH3/CH4 activity within 10ms → I2C probe succeeded]
```

---

#### 4. مشاكل الـ Hardware الشائعة ↔ أنماط الـ Kernel Log

| المشكلة الـ hardware | نمط الـ kernel log | التفسير |
|---|---|---|
| عنوان الـ reg في الـ DT خاطئ | `of_dma_configure_id: ... not in range` | الـ `reg` property لا يتطابق مع الـ bus address الفعلي |
| الجهاز غير مُشغَّل (power domain) | `probe deferred: clock not ready` | الـ `power-domains` في الـ DT يشير لـ controller لم يُهيَّأ بعد |
| مشكلة في الـ pinmux | `probe deferred: pinctrl not ready` | الـ `pinctrl-0` في الـ DT يُشير لـ state غير موجودة |
| الـ compatible string لا يطابق الـ hardware | صمت تام — لا probe يحدث | `of_match_device` يُعيد NULL وينتهي الأمر |
| مشكلة في نطاق DMA | `DMA: Out of SW-IOMMU space` | `dma-ranges` في الـ DT لا يغطي الذاكرة الكاملة للنظام |
| الـ interrupts property خاطئة | `irq: no irq domain found for /mydev@40000000` | الـ interrupt-parent في الـ DT خاطئ |

---

#### 5. Debugging الـ Device Tree — التحقق من التطابق

```bash
# الخطوة 1: استخراج الـ DTB الحالي وتحويله
dtc -I dtb -O dts -o /tmp/live.dts /sys/firmware/fdt 2>/dev/null

# الخطوة 2: البحث عن node الجهاز
grep -A 20 "mydev@40000000" /tmp/live.dts

# مثال على الناتج:
# mydev@40000000 {
#     compatible = "vendor,mydev-v2";
#     reg = <0x0 0x40000000 0x0 0x1000>;
#     interrupts = <0x0 0x20 0x4>;
#     dma-ranges = <0x0 0x0 0x0 0x40000000 0x0 0x10000000>;
#     status = "okay";
# };

# الخطوة 3: مقارنة compatible مع جدول الـ driver
modinfo my_driver | grep alias
# الناتج: alias: of:NmydevTCvendor,mydev-v2
# يجب أن يطابق compatible في الـ DTS

# الخطوة 4: فحص حالة الـ node
cat /sys/firmware/devicetree/base/mydev@40000000/status
# يجب أن يكون "okay" وليس "disabled"

# الخطوة 5: التحقق من #address-cells و #size-cells للـ parent
cat /sys/firmware/devicetree/base/\#address-cells | xxd
# يجب أن يُعيد: 00000002 (أي 2 cells للعنوان على أنظمة 64-bit)
```

---

### Practical Commands

---

#### سيناريو 1: فحص لماذا لا يحدث الـ bind

```bash
#!/bin/bash
# script: debug_of_match.sh
# الغرض: تشخيص فشل of_match_device / of_driver_match_device

DEV="mydev@40000000"
DRV="my_driver"
BUS="platform"

echo "=== [1] هل الجهاز موجود؟ ==="
ls /sys/bus/$BUS/devices/ | grep $DEV

echo "=== [2] هل الـ driver محمَّل؟ ==="
lsmod | grep $DRV || echo "Driver not loaded"

echo "=== [3] الـ compatible في الـ DT ==="
cat /sys/firmware/devicetree/base/$DEV/compatible | tr '\0' '\n'

echo "=== [4] الـ compatible التي يدعمها الـ driver ==="
modinfo $DRV | grep alias

echo "=== [5] هل الجهاز مرتبط بـ driver؟ ==="
ls -la /sys/bus/$BUS/devices/$DEV/driver 2>/dev/null || echo "NOT BOUND"

echo "=== [6] الأجهزة المؤجلة ==="
cat /sys/kernel/debug/devices_deferred | grep $DEV

echo "=== [7] آخر رسائل الـ kernel لهذا الجهاز ==="
dmesg | grep -i "$DEV\|$DRV" | tail -20
```

---

#### سيناريو 2: debugging مشكلة of_dma_configure_id

```bash
# تفعيل dynamic debug لملف address.c
echo 'file drivers/of/address.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/of/device.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# إعادة تحميل الـ driver
rmmod my_driver
modprobe my_driver

# مراقبة الرسائل
dmesg | grep -E "dma_configure|dma.range|of_dma" | tail -30

# مثال على الناتج:
# [   5.123456] drivers/of/address.c:680 of_dma_configure_id: dev mydev@40000000: dma_range_map not found
# [   5.123457] drivers/of/device.c:177 of_dma_configure_id: Setting coherent dma mask from 0xffffffff to 0xffffffffffffffff
```

---

#### سيناريو 3: فحص الـ uevent وتحقق of_device_uevent_modalias

```bash
# مراقبة الـ uevents في الـ background
udevadm monitor --kernel --property 2>&1 | tee /tmp/uevents.log &
MONITOR_PID=$!

# إعادة bind الجهاز
echo $DEV > /sys/bus/platform/drivers/$DRV/unbind 2>/dev/null
sleep 1
echo $DEV > /sys/bus/platform/drivers/$DRV/bind 2>/dev/null
sleep 2

kill $MONITOR_PID

# تحليل الـ uevent
grep -A 15 "DEVPATH.*$DEV" /tmp/uevents.log
# الناتج المتوقع:
# DEVPATH=/devices/platform/mydev@40000000
# SUBSYSTEM=platform
# ACTION=bind
# MODALIAS=of:NmydevT<nil>Cvendor,mydev-v2
# OF_NAME=mydev
# OF_FULLNAME=/mydev@40000000
# OF_COMPATIBLE_0=vendor,mydev-v2
# OF_COMPATIBLE_N=1
```

---

#### سيناريو 4: ftrace كامل لمسار الـ probe

```bash
# إعداد كامل لـ ftrace function_graph
TRACING=/sys/kernel/tracing

echo nop > $TRACING/current_tracer
echo 0 > $TRACING/tracing_on

# تصفية الدوال المرتبطة بـ of_device
echo 'of_match_device
of_driver_match_device
of_dma_configure_id
of_dma_configure
of_device_modalias
of_device_uevent_modalias
of_device_make_bus_id' > $TRACING/set_ftrace_filter

echo function_graph > $TRACING/current_tracer
echo 1 > $TRACING/options/funcgraph-tail
echo 1 > $TRACING/tracing_on

# تحميل الـ driver
modprobe my_driver

echo 0 > $TRACING/tracing_on
cat $TRACING/trace | head -60
```

مثال على الناتج وتفسيره:
```
# tracer: function_graph
#
 0)               |  of_match_device() {
 0)   0.312 us    |    /* returns match for "vendor,mydev-v2" */
 0)   0.891 us    |  }
 0)               |  of_dma_configure_id() {
 0)   1.234 us    |    /* dma_range_map found, mask=0xffffffff */
 0)   2.100 us    |  }
 0)   0.100 us    |  of_device_make_bus_id();
```
التفسير:
- وجود `of_match_device` في الـ trace يعني أن الـ match جرى.
- غياب `of_dma_configure_id` يعني أن الـ probe وقف قبل تهيئة الـ DMA.
- `of_device_make_bus_id` يُستدعى مرة واحدة فقط عند أول تسجيل للجهاز.

---

#### سيناريو 5: فحص شامل للـ DMA configuration

```bash
# فحص الـ DMA mask المُعيَّنة بعد of_dma_configure_id
cat /sys/devices/platform/mydev@40000000/dma_mask
# الناتج: 0x00000000ffffffff  ← mask 32-bit (طبيعي لجهاز 32-bit DMA)

# فحص الـ coherent DMA mask
cat /sys/devices/platform/mydev@40000000/coherent_dma_mask

# فحص IOMMU المرتبط بالجهاز
ls /sys/devices/platform/mydev@40000000/iommu_group/
# إذا كان فارغاً: الجهاز يستخدم direct DMA بدون IOMMU

# فحص dma-ranges من الـ DT
fdtget /sys/firmware/fdt /mydev@40000000 dma-ranges 2>/dev/null \
  || echo "No dma-ranges — using parent's"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بوابة صناعية على RK3562 — درايفر I2C لا يتم probe بسبب خطأ في `compatible`

#### العنوان
فشل تطابق درايفر I2C sensor مع جهاز Device Tree على بوابة صناعية

#### السياق
فريق bring-up يعمل على بوابة صناعية (Industrial Gateway) مبنية على **RK3562**. الجهاز يحتوي على مستشعر درجة حرارة متصل بـ I2C bus. الفريق كتب درايفر مخصص وأضاف node في Device Tree، لكن الجهاز يبوت بدون أي أثر لمستشعر الحرارة في `/sys/bus/i2c/devices/`.

#### المشكلة
الـ `probe()` لا يُستدعى إطلاقاً. تشغيل `dmesg | grep temp` لا يعطي شيئاً. الـ node موجود في DT لكن الدرايفر لا يتعرف عليه.

#### التحليل

عند تحميل الدرايفر، الكيرنل يستدعي `of_match_device()` المُعلن في `of_device_h`:

```c
/* of_device.h - السطر 12 */
extern const struct of_device_id *of_match_device(
    const struct of_device_id *matches, const struct device *dev);
```

الدالة تمر على مصفوفة `of_match_table` داخل `struct device_driver` وتقارن كل entry مع `dev->of_node`. المقارنة تعتمد على حقل `compatible` في DT وحقل `.compatible` في `of_device_id`.

```c
/* of_device.h - السطر 20 */
static inline int of_driver_match_device(struct device *dev,
                                         const struct device_driver *drv)
{
    /* ترجع NULL إذا لم يتطابق أي entry → probe لا يُستدعى */
    return of_match_device(drv->of_match_table, dev) != NULL;
}
```

المشكلة الفعلية: مطابقة نصية دقيقة — حرف زائد أو ناقص يكسر كل شيء.

```c
/* الدرايفر: compatible صحيح */
static const struct of_device_id temp_sensor_ids[] = {
    { .compatible = "vendor,rk3562-temp-sensor" },
    {}
};

/* DT: خطأ إملائي */
temp_sensor@48 {
    compatible = "vendor,rk3562-temp_sensor"; /* underscore بدل hyphen */
    reg = <0x48>;
};
```

#### الحل

```bash
# 1. تحقق من compatible المُعلن في DT
dtc -I dtb -O dts /sys/firmware/fdt | grep -A3 "temp"

# 2. تحقق من ما يطلبه الدرايفر
grep -r "of_match_table\|compatible" /sys/bus/i2c/drivers/temp-sensor/

# 3. أو مباشرة من modalias
cat /sys/bus/i2c/devices/1-0048/modalias
# يجب أن يطابق: of:Ntemp_sensorT<C>vendor,rk3562-temp-sensor
```

```dts
/* الإصلاح في DT */
temp_sensor@48 {
    compatible = "vendor,rk3562-temp-sensor"; /* hyphen صحيح */
    reg = <0x48>;
};
```

#### الدرس المستفاد
**الـ `of_match_device()`** تطابق نصي 100% بدون أي tolerances. أي فرق في الـ `compatible` — سواء hyphen أو underscore أو حرف كبير — يجعل `of_driver_match_device()` ترجع `0` وlا يُستدعى probe. دائماً تحقق من `modalias` في sysfs للمقارنة المباشرة.

---

### السيناريو 2: Android TV Box على Allwinner H616 — uevent لا يصل لـ udev

#### العنوان
فشل تحميل firmware تلقائياً لـ WiFi chip بسبب غياب `modalias` في uevent

#### السياق
شركة تبني **Android TV Box** على **Allwinner H616**. الجهاز يحتوي على WiFi chip متصل بـ SDIO. `udev` مسؤول عن تحميل الـ firmware عند ظهور الـ uevent. بعد التحديث، WiFi لا يعمل عند boot.

#### المشكلة
الـ firmware لا يتحمل تلقائياً. الـ kernel يرى الجهاز لكن `udev` لا يحمّل أي module.

#### التحليل

الدالة `of_device_uevent_modalias()` المُعلنة في الهيدر:

```c
/* of_device.h - السطر 29 */
extern int of_device_uevent_modalias(const struct device *dev,
                                     struct kobj_uevent_env *env);
```

هذه الدالة تُضاف إلى `kobj_uevent_env` بصيغة:

```
MODALIAS=of:NwifiT<C>allwinner,h616-wifi
```

**الـ `of_device_uevent()`** المُعلن في السطر 28 يُطلقها:

```c
extern void of_device_uevent(const struct device *dev,
                              struct kobj_uevent_env *env);
```

عند تعطيل `CONFIG_OF` أو عند عدم وجود `of_node` للجهاز، نسخة الـ stub تُستدعى:

```c
/* of_device.h - السطر 51 */
static inline void of_device_uevent(const struct device *dev,
                        struct kobj_uevent_env *env) { }  /* لا تفعل شيئاً */
```

```c
/* of_device.h - السطر 60 */
static inline int of_device_uevent_modalias(const struct device *dev,
                               struct kobj_uevent_env *env)
{
    return -ENODEV;  /* udev لا يرى أي modalias */
}
```

السبب: بعد التحديث، الـ kernel compile بدون `CONFIG_OF` أو الـ `of_node` غير مربوط بالجهاز.

#### الحل

```bash
# 1. تحقق من CONFIG_OF
zcat /proc/config.gz | grep CONFIG_OF

# 2. تحقق من وجود of_node
ls -la /sys/bus/sdio/devices/mmc1:0001:1/of_node

# 3. تحقق من uevent
udevadm test /sys/bus/sdio/devices/mmc1:0001:1/ 2>&1 | grep MODALIAS
```

```dts
/* تأكد من وجود node في DT */
&mmc1 {
    status = "okay";
    wifi@1 {
        compatible = "allwinner,h616-wifi";
        reg = <1>;
    };
};
```

```bash
# إعادة trigger الـ uevent يدوياً للاختبار
echo add > /sys/bus/sdio/devices/mmc1:0001:1/uevent
dmesg | tail -20
```

#### الدرس المستفاد
**الـ `of_device_uevent_modalias()`** هي الجسر بين Device Tree و `udev`. إذا كان `CONFIG_OF` غائباً أو الـ `of_node` غير مربوط، تعود stub تُرجع `-ENODEV` وudev لا يعلم بأي جهاز. دائماً تحقق من وجود `of_node` symlink في sysfs.

---

### السيناريو 3: ECU السيارات على i.MX8 — تعارض DMA بين SPI controller ومعالج الصوت

#### العنوان
crash عشوائي في DMA transfer على ECU مبني على i.MX8 بسبب خطأ في `of_dma_configure()`

#### السياق
شركة automotive تبني **ECU** على **i.MX8**. الـ ECU يشغّل CAN bus وSPI بسرعات عالية مع DMA. بعد إضافة codec صوت على I2S، النظام يعاني من crash عشوائي في DMA مع `kernel BUG at mm/dma-mapping.c`.

#### المشكلة
الـ DMA يحاول الكتابة في عناوين فيزيائية غير صالحة. الـ crash غير قابل للتكرار المنتظم.

#### التحليل

الدالة المحورية في الهيدر:

```c
/* of_device.h - السطر 31 */
int of_dma_configure_id(struct device *dev,
                         struct device_node *np,
                         bool force_dma, const u32 *id);

/* of_device.h - السطر 34 */
static inline int of_dma_configure(struct device *dev,
                                   struct device_node *np,
                                   bool force_dma)
{
    /* id = NULL → لا يوجد stream ID مخصص */
    return of_dma_configure_id(dev, np, force_dma, NULL);
}
```

**`of_dma_configure()`** تقرأ من DT:
- `dma-ranges` — لتحديد نطاقات العناوين الصالحة
- `#dma-cells` — لتحديد عدد خلايا DMA
- IOMMU group إذا وُجد

الخطأ: الـ I2S node في DT يفتقر إلى `dma-ranges` أو `iommus` property، وعند `of_dma_configure()` يُضبط الـ `dma_mask` بشكل خاطئ، مما يسمح للـ DMA بالوصول لعناوين خارج نطاق الذاكرة الفيزيائية.

```dts
/* خاطئ - بدون dma-ranges */
sai3: sai@308c0000 {
    compatible = "fsl,imx8mp-sai";
    reg = <0x308c0000 0x10000>;
    dmas = <&sdma3 1 2 0>, <&sdma3 2 2 0>;
    /* dma-ranges غائبة → of_dma_configure تفشل صامتة */
};
```

```dts
/* صحيح */
sai3: sai@308c0000 {
    compatible = "fsl,imx8mp-sai";
    reg = <0x308c0000 0x10000>;
    dmas = <&sdma3 1 2 0>, <&sdma3 2 2 0>;
    dma-names = "rx", "tx";
    /* السماح للـ DMA بالوصول لنطاق الذاكرة الكامل */
    dma-ranges = <0x0 0x0 0x0 0x0 0xffffffff 0xffffffff>;
};
```

```bash
# تشخيص: تحقق من dma_mask المضبوط
cat /sys/bus/platform/devices/308c0000.sai/dma_mask

# فحص IOMMU groups
ls /sys/bus/platform/devices/308c0000.sai/iommu_group/
```

#### الحل

```c
/* في الدرايفر: استخدم of_dma_configure_id مع stream ID صريح للـ IOMMU */
static int imx_sai_probe(struct platform_device *pdev)
{
    u32 stream_id = 0;
    int ret;

    /* تمرير stream ID لضمان عزل IOMMU صحيح */
    ret = of_dma_configure_id(&pdev->dev,
                               pdev->dev.of_node,
                               true,   /* force DMA */
                               &stream_id);
    if (ret) {
        dev_err(&pdev->dev, "DMA configure failed: %d\n", ret);
        return ret;
    }
    /* ... باقي الـ probe */
}
```

#### الدرس المستفاد
**`of_dma_configure()`** تضع حدود الـ DMA بناءً على معلومات DT. غياب `dma-ranges` في سياق i.MX8 مع IOMMU يؤدي إلى `dma_mask` خاطئ وcorruption عشوائي. استخدم `of_dma_configure_id()` مع stream ID صريح عند وجود IOMMU لضمان العزل الصحيح بين الأجهزة.

---

### السيناريو 4: لوحة IoT على STM32MP1 — جهازان بنفس الـ compatible يسبب probe للجهاز الخطأ

#### العنوان
درايفر SPI يُضبط على جهاز خاطئ على بوارد IoT مخصصة بسبب ترتيب `of_match_table`

#### السياق
فريق يطور **IoT sensor board** على **STM32MP1**. البوارد تحتوي جهازين SPI مختلفين: `accelerometer` و`barometer`، كلاهما من نفس الشركة المصنعة. تم استخدام درايفر موحّد لكلا الجهازين مع `data` مختلفة في `of_device_id`.

#### المشكلة
قراءات الـ barometer تُظهر قيم acceleration وليس ضغطاً. الجهازان يعملان لكن البيانات معكوسة.

#### التحليل

`of_match_device()` تمر على `of_match_table` بالترتيب وتُرجع **أول تطابق**:

```c
/* of_device.h - السطر 12 */
extern const struct of_device_id *of_match_device(
    const struct of_device_id *matches, const struct device *dev);
```

```c
/* الدرايفر الخاطئ: ترتيب خاطئ في المصفوفة */
static const struct of_device_id sensor_ids[] = {
    /* هذا يطابق كلا الجهازين إذا كان compatible عاماً */
    { .compatible = "bosch,bmp3xx",      .data = &accel_config },
    { .compatible = "bosch,bmp390",      .data = &baro_config  },
    {}
};
```

```dts
/* DT */
barometer@0 {
    compatible = "bosch,bmp390", "bosch,bmp3xx"; /* عدة compatible strings */
    reg = <0>;
    spi-max-frequency = <10000000>;
};
```

`of_match_device()` تجد `"bosch,bmp3xx"` أولاً في قائمة compatible strings وتُرجع entry الـ `accel_config` رغم أن الجهاز barometer.

```c
/* كيفية استخدام نتيجة of_match_device في الدرايفر */
static int sensor_probe(struct spi_device *spi)
{
    const struct of_device_id *match;

    /* of_driver_match_device داخلياً تستدعي of_match_device */
    match = of_match_device(sensor_ids, &spi->dev);
    if (!match)
        return -ENODEV;

    /* match->data تُحدد نوع الجهاز — خاطئة هنا */
    sensor_config = match->data;
}
```

#### الحل

```c
/* الإصلاح: ترتيب من الأكثر خصوصية إلى الأعم */
static const struct of_device_id sensor_ids[] = {
    /* الأكثر تخصصاً أولاً */
    { .compatible = "bosch,bmp390",  .data = &baro_config  },
    { .compatible = "bosch,bmp388",  .data = &baro_config  },
    /* الأعم أخيراً كـ fallback */
    { .compatible = "bosch,bmp3xx",  .data = &accel_config },
    {}
};
```

```bash
# تشخيص: تحقق من of_node compatible strings بالترتيب
cat /sys/bus/spi/devices/spi0.0/of_node/compatible | xxd
# السلاسل مفصولة بـ null byte

# تحقق من الـ match الفعلي
cat /sys/bus/spi/devices/spi0.0/modalias
```

#### الدرس المستفاد
**`of_match_device()`** تُرجع أول تطابق في مصفوفة `of_match_table`. عند وجود جهاز بعدة `compatible` strings، الترتيب في المصفوفة حاسم. ضع دائماً الـ compatible الأكثر تخصصاً أولاً، والأعم (family-level) أخيراً كـ fallback.

---

### السيناريو 5: Custom Board على AM62x — `bus_id` مكرر يمنع تسجيل الجهاز

#### العنوان
فشل تسجيل جهاز UART ثانٍ على بوارد bring-up مخصصة على AM62x

#### السياق
مهندس يعمل على **custom board bring-up** مبنية على **TI AM62x**. البوارد تحتوي 4 UARTs. بعد إضافة UART3 إلى DT، التسجيل يفشل مع رسالة `device already exists`.

#### المشكلة
```
[    2.341] platform 2810000.serial: device already exists
[    2.342] platform: error adding device
```

رغم أن العناوين مختلفة في DT.

#### التحليل

الدالة `of_device_make_bus_id()` المُعلنة في الهيدر:

```c
/* of_device.h - السطر 41 */
void of_device_make_bus_id(struct device *dev);
```

هذه الدالة تبني `dev_name` للجهاز من الـ `reg` property في DT. الصيغة المعتادة:

```
<unit_address>.<bus_type>
```

مثلاً للـ UART على AM62x:
```
2800000.serial  ← UART0
2810000.serial  ← UART1
2820000.serial  ← UART2
```

المشكلة: المهندس نسخ UART node وغيّر `reg` لكن نسي تغيير `unit-address` في اسم الـ node نفسه:

```dts
/* DT الخاطئ */
serial@2810000 {           /* unit-address صحيح */
    compatible = "ti,am64-uart", "ti,am654-uart";
    reg = <0x00 0x2810000 0x00 0x100>;
    /* ... */
};

serial@2810000 {           /* مكرر! نسخ ولم يتغير */
    compatible = "ti,am64-uart", "ti,am654-uart";
    reg = <0x00 0x2820000 0x00 0x100>;  /* reg تغيّر لكن اسم Node لا */
    /* ... */
};
```

`of_device_make_bus_id()` تقرأ `reg` وتبني الـ bus_id من unit address المُستخرج، لكن إذا كان اسم الـ node نفسه مكرراً في DT، الـ DTB compiler يدمجهم أو يتعارض التسجيل.

```bash
# تشخيص: فحص الـ bus IDs المولّدة
ls /sys/bus/platform/devices/ | grep serial

# فحص DT nodes
find /sys/firmware/devicetree/base -name "serial*" -type d

# فحص reg property لكل node
for d in /sys/firmware/devicetree/base/serial*; do
    echo -n "$d: reg = "
    hexdump -C $d/reg 2>/dev/null | head -1
done
```

```dts
/* الإصلاح */
serial@2810000 {
    compatible = "ti,am64-uart", "ti,am654-uart";
    reg = <0x00 0x2810000 0x00 0x100>;
    clock-frequency = <48000000>;
    status = "okay";
};

serial@2820000 {            /* unit-address صحيح الآن */
    compatible = "ti,am64-uart", "ti,am654-uart";
    reg = <0x00 0x2820000 0x00 0x100>;
    clock-frequency = <48000000>;
    status = "okay";
};
```

```bash
# تحقق بعد الإصلاح
dtc -I dts -O dtb board.dts | dtc -I dtb -O dts | grep -A5 "serial@"
# يجب أن يُظهر node-ين مستقلين بعناوين مختلفة
```

#### الدرس المستفاد
**`of_device_make_bus_id()`** تبني هوية الجهاز من الـ `reg` property عبر unit address، لكن تعارض أسماء الـ nodes في DT المصدر يسبب دمجاً أو تعارضاً قبل أن تصل `of_device_make_bus_id()` للعمل. في bring-up جديد، دائماً شغّل `dtc -I dtb -O dts /sys/firmware/fdt` للتحقق من الـ DT الفعلي الذي يقرأه الكيرنل، وليس الملف المصدري.
## Phase 7: مصادر ومراجع

### مقالات LWN.net الأساسية

هذه المقالات تُغطي النواة الجوهرية لكيفية عمل `of_device` و`of_match_device` ضمن منظومة **Open Firmware / Device Tree** في نواة Linux:

| المقال | الموضوع |
|--------|---------|
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | كيف تُنشئ `of_platform_populate` أجهزة platform من شجرة الأجهزة، وكيف يعمل المطابقة عبر `of_match_device` |
| [The platform device API](https://lwn.net/Articles/448499/) | شرح كامل لـ `platform_driver`، جدول `of_match_table`، ودورة حياة driver/device |
| [drivercore: Add of_match_table to the common device drivers](https://lwn.net/Articles/377660/) | الـ patch الذي أضاف `of_match_table` إلى `device_driver` — الأساس الذي تقوم عليه `of_driver_match_device` |
| [Basic ARM device tree support](https://lwn.net/Articles/423607/) | البداية الحقيقية لدعم device tree على ARM في نواة Linux |
| [Device trees I: Are we having fun yet?](https://lwn.net/Articles/572692/) | مشاكل ABI وتوحيد الـ bindings — سياق مهم لفهم لماذا `of_device_uevent_modalias` ضروري |
| [Device tree overlays](https://lwn.net/Articles/616859/) | dynamic device tree وتأثيره على `of_match_device` و`of_device_uevent` |
| [ARM: DMA-mapping & IOMMU integration](https://lwn.net/Articles/444700/) | الأساس النظري لـ `of_dma_configure` وربط DMA بـ IOMMU عبر device tree |
| [Reworking the DMA mapping code](https://lwn.net/Articles/467509/) | تطور `of_dma_configure_id` وإدارة per-device DMA ops |
| [IOMMU probe deferral support](https://lwn.net/Articles/719410/) | كيف تتعامل `of_dma_configure` مع `probe_defer` عند وجود IOMMU |
| [Device tree troubles](https://lwn.net/Articles/560523/) | التحديات في توحيد device tree بين المنصات المختلفة |

---

### الوثائق الرسمية لنواة Linux

**الـ `Documentation/` paths** المباشرة في شجرة المصدر:

```
Documentation/devicetree/
├── usage-model.rst          ← كيف يُستخدم device tree في Linux
├── bindings/                ← مواصفات الـ compatible strings لكل جهاز
└── of_unittest.txt          ← اختبارات وحدة OF

Documentation/driver-api/
├── driver-model/
│   └── platform.rst         ← platform_driver + of_match_table
└── infrastructure.rst       ← device driver infrastructure

Documentation/core-api/
└── dma-api-howto.rst        ← دليل DMA mapping وعلاقته بـ of_dma_configure
```

الـ [Linux and the Devicetree](https://docs.kernel.org/devicetree/usage-model.html) هي نقطة البداية الرسمية لفهم كيف تُحوّل النواة شجرة الأجهزة إلى `struct device` مرتبطة بـ driver.

الـ [Open Firmware and Devicetree — kernel documentation](https://static.lwn.net/kerneldoc/devicetree/index.html) يجمع كل وثائق OF الرسمية.

الـ [Platform Devices and Drivers](https://www.kernel.org/doc/html/latest/driver-api/driver-model/platform.html) يشرح استخدام `of_match_table` داخل `platform_driver`.

---

### الملفات المصدرية الجوهرية في النواة

**الـ `of_device.h` هو الواجهة — التنفيذ موزع على:**

```
include/linux/of_device.h       ← الملف الحالي (API declarations)
drivers/of/device.c             ← تنفيذ of_match_device, of_device_uevent, of_dma_configure
drivers/of/base.c               ← of_match_node, of_device_is_compatible
include/linux/of.h              ← struct of_device_id, of_device_is_compatible
include/linux/device/driver.h   ← struct device_driver + of_match_table field
```

---

### Commits مهمة في تاريخ النواة

البحث عن هذه الـ commits في [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/) أو [GitHub torvalds/linux](https://github.com/torvalds/linux/commits/master/):

| الموضوع | كيفية البحث |
|---------|------------|
| إضافة `of_match_table` لـ `device_driver` | `git log --all --grep="of_match_table" -- drivers/of/` |
| تنفيذ `of_dma_configure` الأصلي | `git log --all --grep="of_dma_configure" -- drivers/of/device.c` |
| إضافة `of_dma_configure_id` | `git log --all --grep="of_dma_configure_id"` |
| `of_device_uevent_modalias` | `git log --all --grep="of_device_uevent_modalias"` |

يمكن البحث الفوري عبر [linux-commits-search.typesense.org](https://linux-commits-search.typesense.org/) باستخدام اسم الدالة مباشرة.

---

### نقاشات Mailing List

- **LKML** — الأرشيف الرئيسي: [lkml.org](https://lkml.org/) — ابحث عن `of_match_device`, `of_dma_configure`, `of_device_id`
- **[Linux drivers and devices: registration, matching, aliases and autoloading](https://blog.dowhile0.org/2022/06/10/linux-drivers-and-devices-registration-matching-aliases-and-modules-autoloading/)** — تحليل معمق لآلية المطابقة من مطور نواة
- **[What is the use of of_device_id and i2c_device_id?](https://www.linuxquestions.org/questions/linux-kernel-70/what-is-the-use-of-of_device_id-and-i2c_device_id-4175630297/)** — نقاش عملي حول `of_device_id` وكيفية استخدامه في driver حقيقي

---

### eLinux.org — مراجع عملية

| الصفحة | المحتوى |
|--------|---------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل للـ DTS syntax وربطه بـ `of_device_id` |
| [Linux Drivers Device Tree Guide](https://elinux.org/Linux_Drivers_Device_Tree_Guide) | دليل كتابة driver يستخدم `of_match_table` و`of_match_device` |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | توضيح الجوانب الغامضة مثل `of_find_node_by_phandle` و DMA config |
| [Device Tree Presentations](https://elinux.org/Device_Tree_Presentations) | عروض تقديمية من مؤتمرات kernel حول driver binding |
| [Device tree history](https://elinux.org/Device_tree_history) | التطور التاريخي من Open Firmware (PowerPC) إلى ARM وما بعده |
| [Device Trees](https://elinux.org/Device_Trees) | الصفحة الرئيسية الجامعة لكل موارد device tree |

---

### Kernelnewbies.org — تتبع التغييرات عبر الإصدارات

صفحات الإصدارات التالية تتضمن تغييرات جوهرية في `of_device` و DMA configuration:

- [Linux 6.8](https://kernelnewbies.org/Linux_6.8) — دعم تضمين device tree في kernel image
- [Linux 6.5](https://kernelnewbies.org/Linux_6.5) — تحسينات في OF device matching
- [Linux 5.16](https://kernelnewbies.org/Linux_5.16) — تحديثات DMA API وربطها بـ OF
- [Linux 3.2](https://kernelnewbies.org/Linux_3.2) — مرحلة استقرار ARM device tree support

---

### الكتب المرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model — يشرح `struct device`, `struct device_driver`, وآلية المطابقة
- **الفصل 15**: Memory Mapping and DMA — أساس `of_dma_configure`
- متاح مجانًا: [oreilly.com/library/view/linux-device-drivers](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch15.html)

> ملاحظة: LDD3 كُتب قبل انتشار device tree على ARM، لكن مبادئ device model لا تزال صالحة.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — يشرح bus/device/driver model الذي تبنى عليه `of_device`
- **الفصل 13**: The Virtual Filesystem — سياق مفيد لفهم `sysfs` وكيف يتفاعل مع `of_device_make_bus_id`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16**: Device Trees — شرح عملي لـ DTS وكيف تُولّد `of_device_id` و `compatible` strings
- الكتاب يركز على ARM embedded systems حيث `of_device.h` هو العمود الفقري

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- **الفصل 11**: Interfacing with Device Drivers — أمثلة حديثة على `of_match_table` في drivers حقيقية

---

### مصادر إضافية للتعلم العملي

#### Linux Kernel Labs
- [Device Model Lab](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html) — تمارين عملية لفهم bus/device/driver + `of_match_device`

#### Kernel Source Browsing
- [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/include/linux/of_device.h) — تصفح `of_device.h` مع cross-references لكل الدوال

---

### مصطلحات البحث الموصى بها

للعثور على معلومات إضافية، استخدم هذه المصطلحات:

```
# بحث عام
"of_match_device" linux kernel
"of_device_id" "compatible" driver matching
"of_dma_configure" IOMMU device tree
"of_device_uevent" modalias hotplug

# بحث متخصص
"of_match_table" platform_driver binding
"CONFIG_OF" device driver fallback stub
"of_device_make_bus_id" sysfs naming
"dma-ranges" device tree of_dma_configure

# بحث في الكود
site:elixir.bootlin.com of_match_device
site:github.com/torvalds/linux of_dma_configure_id
```
## Phase 8: Writing simple module

### الهدف

سنستخدم **kprobe** لاعتراض الدالة `of_match_device` المُصدَّرة من `of_device.h`، وهي الدالة التي يستدعيها kernel لمطابقة driver مع device عبر **Device Tree**. كل مرة يحاول فيها driver أن يتطابق مع device يمر الطلب عبر هذه الدالة، مما يجعلها نقطة مراقبة مثالية لفهم نشاط bus الـ platform.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_match.c
 *
 * Hook of_match_device() to log every OF driver-device match attempt.
 * Useful for understanding Device Tree probe activity at runtime.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/mod_devicetable.h> /* struct of_device_id */
#include <linux/of.h>           /* struct device_node */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe on of_match_device() to trace OF driver matching");

/* ----------------------------------------------------------------
 * pre-handler: called just before of_match_device() executes.
 *
 * Signature of the hooked function:
 *   const struct of_device_id *of_match_device(
 *       const struct of_device_id *matches,
 *       const struct device *dev);
 *
 * On x86-64 the arguments arrive in registers:
 *   regs->di = matches  (first  arg)
 *   regs->si = dev      (second arg)
 * ---------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Recover the two arguments from CPU registers */
    const struct of_device_id *matches =
            (const struct of_device_id *)regs->di;
    const struct device *dev =
            (const struct device *)regs->si;

    /* Guard against NULL pointers before dereferencing */
    if (!dev || !matches)
        return 0;

    /* of_node: the Device Tree node attached to this device */
    const char *dt_name = "none";
    if (dev->of_node && dev->of_node->full_name)
        dt_name = dev->of_node->full_name;

    /* compatible: first entry in the match table the driver offers */
    const char *compatible = "(empty)";
    if (matches->compatible[0])
        compatible = matches->compatible;

    pr_info("of_match_device: dev=%s dt_node=%s drv_compat=%s\n",
            dev_name(dev), dt_name, compatible);

    return 0; /* 0 = let the original function run normally */
}

/* ----------------------------------------------------------------
 * post-handler: called after of_match_device() returns.
 * We just log the return value (NULL = no match, ptr = matched).
 * ---------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* On x86-64 the return value is in regs->ax */
    unsigned long ret = regs->ax;

    if (ret)
        pr_info("of_match_device: MATCHED -> entry @ %px\n",
                (void *)ret);
    /* Suppress the "no match" noise to keep logs clean */
}

/* ----------------------------------------------------------------
 * kprobe descriptor — binds handlers to the symbol by name.
 * The kernel resolves the address at registration time via kallsyms.
 * ---------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "of_match_device",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ----------------------------------------------------------------
 * module_init: register the probe.
 * If the symbol does not exist (CONFIG_OF=n) registration fails
 * and the module refuses to load — safe by design.
 * ---------------------------------------------------------------- */
static int __init kprobe_of_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_of: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("kprobe_of: planted at %px (of_match_device)\n",
            kp.addr);
    return 0;
}

/* ----------------------------------------------------------------
 * module_exit: unregister the probe.
 * Must be done before the module is unloaded; leaving a dangling
 * kprobe causes a kernel crash when the hooked function is called.
 * ---------------------------------------------------------------- */
static void __exit kprobe_of_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_of: removed from of_match_device\n");
}

module_init(kprobe_of_init);
module_exit(kprobe_of_exit);
```

---

### شرح كل قسم

#### `#include` — الملفات المضمَّنة

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | يعرّف `struct kprobe` ودوال `register_kprobe / unregister_kprobe` |
| `<linux/device.h>` | يعرّف `struct device` والدالة المساعدة `dev_name()` |
| `<linux/mod_devicetable.h>` | يعرّف `struct of_device_id` الذي يحمل حقل `compatible` |
| `<linux/of.h>` | يعرّف `struct device_node` وحقله `full_name` |

نحتاج هذه الـ headers لأننا نقرأ حقولًا حقيقية من الـ structs داخل الـ handlers، ومن دونها يرفض compiler التصريف.

---

#### `handler_pre` — قبل تنفيذ الدالة

الـ `pre_handler` يُستدعى لحظة وصول CPU إلى أول بايت من `of_match_device`، قبل أن تُنفَّذ أي سطر منها. نقرأ المعاملَين من سجلات المعالج (`regs->di`, `regs->si`) لأن calling convention الـ x86-64 يضع الـ argument الأول في `rdi` والثاني في `rsi`. الإرجاع بـ `0` يعني "استمر في تنفيذ الدالة الأصلية"؛ لو أرجعنا `1` لتجاوزها kernel تمامًا.

---

#### `handler_post` — بعد تنفيذ الدالة

الـ `post_handler` يُستدعى بعد انتهاء `of_match_device` مباشرةً. نقرأ `regs->ax` لأنه يحمل قيمة الإرجاع على x86-64؛ إذا كانت غير صفر فقد نجحت المطابقة وسنطبع عنوان الـ `of_device_id` entry المُختارة.

---

#### `struct kprobe kp` — واصف الـ probe

**الـ `symbol_name`** يُعرِّف الدالة المستهدفة بالاسم بدلًا من العنوان المباشر، فيبحث kernel عنها في `kallsyms` لحظة التسجيل. هذا يجعل الكود محمولًا بين kernel builds مختلفة طالما الدالة موجودة.

---

#### `kprobe_of_init` — تسجيل الـ probe

`register_kprobe` تكتب `int3` (breakpoint instruction) فوق أول بايت من `of_match_device` في ذاكرة الـ kernel. إذا فشل التسجيل (مثلًا لأن `CONFIG_OF=n` وبالتالي الرمز غير موجود) نُرجع الخطأ مباشرةً فيرفض kernel تحميل الـ module بأمان.

---

#### `kprobe_of_exit` — إزالة الـ probe

`unregister_kprobe` تستعيد البايت الأصلي وتُلغي تسجيل الـ handlers. إهمال هذه الخطوة يترك تعليمة `int3` داخل `of_match_device`؛ عند استدعائها لاحقًا لن يجد kernel handler مسجَّلًا فيحدث **kernel panic**، لذا يجب دائمًا إلغاء التسجيل في `module_exit`.

---

### بناء الـ module

```makefile
# Kbuild (اسم الملف: Makefile)
obj-m += kprobe_of_match.o
```

```bash
# بناء وتحميل
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
sudo insmod kprobe_of_match.ko

# مراقبة المخرجات
sudo dmesg -w | grep kprobe_of

# إزالة الـ module
sudo rmmod kprobe_of_match
```

---

### مثال على المخرجات

```
[ 12.441] kprobe_of: planted at ffffffffc0a1b340 (of_match_device)
[ 12.503] of_match_device: dev=soc:serial@11002000 dt_node=/soc/serial@11002000 drv_compat=mediatek,mt6577-uart
[ 12.503] of_match_device: MATCHED -> entry @ ffff888003c14a80
[ 12.510] of_match_device: dev=soc:i2c@11007000 dt_node=/soc/i2c@11007000 drv_compat=mediatek,mt6577-i2c
```

كل سطر يكشف: اسم الـ device، مسار node الـ Device Tree، وأول `compatible` string يقدّمها الـ driver — معلومات ثمينة عند تشخيص مشاكل `probe` على منصات embedded.
