## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **Linux Driver Model** — اللي بيتسمى كمان **Driver Core** — وده الـ subsystem اللي بيتحكم في العلاقة بين الـ hardware devices والـ software drivers جوه الـ kernel. الـ implementation الأساسية موجودة في `drivers/base/`.

---

### القصة من الأول

تخيل عندك مصنع كبير (الـ Linux kernel) وبييجيه شغل (devices) من كل حتة — USB، PCI، I2C، SPI، إلخ. المصنع محتاج نظام يعرف: "إيه الموظف (driver) المناسب لكل شغلة (device)؟"

قبل الـ Driver Model كان الـ kernel فوضى — كل bus type بيعمل نظامه الخاص لمطابقة الـ drivers بالـ devices، ومفيش طريقة موحدة لإدارة الـ lifecycle (probe, remove, suspend, resume). Patrick Mochel في 2001 جاب فكرة بسيطة: خلينا نعمل **نموذج موحد** لكل driver في النظام.

---

### الـ Big Picture بالتفصيل

**الهدف الأساسي من الملف ده:**

`include/linux/device/driver.h` هو الـ header اللي بيعرّف `struct device_driver` — ده **بطاقة هوية** أي driver في الـ kernel. أي driver من USB لـ GPU لـ sensor صغير — لازم يملي الـ struct ده.

**ليه ده مهم؟**

الـ kernel محتاج يعرف عن كل driver:
- اسمه إيه؟ (`name`)
- بيشتغل على إيه نوع bus؟ (`bus`)
- إزاي يتحقق من إنه يقدر يشتغل مع device معين؟ (`probe`)
- إزاي يقفل نفسه لما الـ device يتشال؟ (`remove`)
- إزاي يتعامل مع الـ power management؟ (`suspend`, `resume`, `pm`)

---

### تشبيه عملي: وكالة توظيف

```
┌─────────────────────────────────────────────────────────┐
│                    Linux Kernel                         │
│                                                         │
│  ┌──────────────┐     match()     ┌──────────────────┐  │
│  │   Device     │ ◄────────────── │   bus_type       │  │
│  │  (Hardware)  │                 │  (USB, PCI, I2C) │  │
│  └──────┬───────┘                 └────────┬─────────┘  │
│         │                                  │            │
│         │         probe()                  │            │
│         └─────────────────────────────────►│            │
│                                            │            │
│                                  ┌─────────▼─────────┐  │
│                                  │  device_driver    │  │
│                                  │  (struct defined  │  │
│                                  │   in driver.h)    │  │
│                                  └───────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

الـ bus هو "وكالة التوظيف" — بتعمل `match()` تطابق device بـ driver. لو الـ match نجح بتنادي `probe()` على الـ driver عشان يبدأ يشتغل.

---

### أهم المفاهيم في الملف

#### 1. `struct device_driver` — قلب الموضوع

```c
struct device_driver {
    const char          *name;           /* اسم الـ driver */
    const struct bus_type *bus;          /* الـ bus اللي بيشتغل عليه */
    struct module       *owner;          /* الـ kernel module صاحبه */

    /* جداول المطابقة */
    const struct of_device_id   *of_match_table;   /* Device Tree */
    const struct acpi_device_id *acpi_match_table;  /* ACPI */

    /* دورة حياة الـ device */
    int  (*probe)  (struct device *dev);  /* هنا الـ driver بيتعرف على الـ device */
    int  (*remove) (struct device *dev);  /* لما الـ device يتشال */
    void (*shutdown)(struct device *dev); /* وقت إغلاق النظام */
    int  (*suspend) (struct device *dev, pm_message_t state); /* نوم */
    int  (*resume)  (struct device *dev); /* صحيان */

    /* power management متقدم */
    const struct dev_pm_ops *pm;

    struct driver_private *p; /* بيانات داخلية للـ driver core بس */
};
```

#### 2. `enum probe_type` — متى يحصل الـ Probe؟

```c
enum probe_type {
    PROBE_DEFAULT_STRATEGY,    /* الـ core يقرر */
    PROBE_PREFER_ASYNCHRONOUS, /* موازي = أسرع boot */
    PROBE_FORCE_SYNCHRONOUS,   /* لازم ينتهي قبل ما يكمل */
};
```

الـ asynchronous probing بيسمح لـ slow devices (زي HDD أو USB) إنها تعمل probe في background وما توقفش الـ boot.

#### 3. `struct driver_attribute` و Macros — الـ sysfs Interface

```c
/* بيخلي الـ driver يعرض معلومات في /sys/bus/<bus>/drivers/<driver>/ */
#define DRIVER_ATTR_RO(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_RO(_name)
```

#### 4. `module_driver()` — Boilerplate Killer

```c
/* بدل ما تكتب module_init و module_exit يدويًا */
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ return __register(&(__driver), ##__VA_ARGS__); } \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ __unregister(&(__driver), ##__VA_ARGS__); } \
module_exit(__driver##_exit);
```

كل الـ bus-specific macros زي `module_platform_driver()` و `module_i2c_driver()` بتستخدم ده داخليًا.

---

### قصة الـ Deferred Probe

مشكلة حقيقية: driver A محتاج GPIO controller يكون جاهز الأول، لكن الـ kernel حمّل driver A قبل GPIO driver. الحل:

```
driver A probe() يرجع -EPROBE_DEFER
        ↓
الـ driver core بيحط الـ device في قايمة انتظار
        ↓
لما GPIO driver يبقى جاهز
        ↓
driver_deferred_probe_add() تلغي الانتظار وتعيد المحاولة
```

الـ `driver_deferred_probe_add()` و `driver_deferred_probe_check_state()` الموجودين في الملف ده بيديروا الميكانيكية دي.

---

### ليه الملف ده مهم؟

بدون `struct device_driver`:
- مفيش طريقة موحدة للـ kernel يعمل match بين device و driver
- مفيش lifecycle management (الـ suspend/resume مش هيشتغل)
- مفيش sysfs entries للـ drivers (`/sys/bus/*/drivers/`)
- الـ hot-plug (USB اللي بتوصله وانت شغال) مش هيشتغل

---

### ملفات الـ Subsystem الأساسية

| الملف | الدور |
|-------|-------|
| `include/linux/device/driver.h` | **الملف ده** — تعريف `struct device_driver` والـ API |
| `include/linux/device/bus.h` | تعريف `struct bus_type` — الـ bus abstraction |
| `include/linux/device/class.h` | تعريف `struct class` — تجميع devices منطقيًا |
| `include/linux/device/devres.h` | الـ managed resources (devres) API |
| `include/linux/device.h` | الـ umbrella header — يجمع كل الـ device headers |
| `drivers/base/driver.c` | تنفيذ `driver_register()`, `driver_unregister()`, `driver_find()` |
| `drivers/base/dd.c` | تنفيذ الـ device-driver binding — قلب الـ probe logic |
| `drivers/base/bus.c` | تنفيذ الـ bus operations والـ match loop |
| `drivers/base/core.c` | الـ device lifecycle — `device_add()`, `device_del()` |
| `drivers/base/init.c` | تهيئة الـ driver core عند بداية الـ kernel |
| `drivers/base/devres.c` | الـ managed resources — cleanup تلقائي |

---

### ملفات مفيد تعرفها

- **`Documentation/driver-api/driver-model/`** — الـ official documentation للـ Driver Model
- **`include/linux/mod_devicetable.h`** — تعريفات الـ `of_device_id` و `acpi_device_id` (جداول المطابقة)
- **`include/linux/pm.h`** — تعريف `pm_message_t` و `dev_pm_ops`
- **`include/linux/kobject.h`** — الـ kernel object model اللي بيبني عليه كل ده
- **`include/linux/platform_device.h`** — مثال عملي لـ bus يستخدم `struct device_driver`
## Phase 2: شرح الـ Driver Model Framework

### المشكلة — ليه الـ Driver Model موجود أصلاً؟

قبل الـ driver model (قبل kernel 2.5)، كل bus subsystem (PCI, USB, platform, ...) كان بيعمل الـ device/driver management بطريقته الخاصة. النتيجة:

- كل bus كان عنده كود تاني لعمل probe، removal، power management.
- مفيش طريقة موحدة تعرف منها "إيه الـ devices الموجودة في النظام؟"
- الـ `/proc` كانت مبعثرة وصعبة الـ introspection.
- الـ power management كان nightmare — كل driver بيعمله بطريقة مختلفة.
- مفيش visibility للـ userspace على الـ device topology.

**الـ driver model** جاء يحل ده بفكرة واحدة: **كل device وكل driver هو object موحد** — بيتم تسجيله، تتبعه، وإدارته من مكان مركزي واحد هو الـ **driver core**.

---

### الحل — الـ Unified Driver Model

الـ kernel بيعمل **abstraction layer** فوق كل الـ buses. الـ abstraction بيشتغل على ٣ محاور:

| المحور | الـ abstraction | الـ struct |
|--------|----------------|------------|
| الجهاز نفسه | `struct device` | يمثل أي device في النظام |
| السوفتوير بتاع الجهاز | `struct device_driver` | يمثل الـ driver |
| القناة اللي بتربطهم | `struct bus_type` | يمثل الـ bus |

الـ driver core بيعمل **match** بين الـ device والـ driver عبر الـ bus، وبعدين بيعمل **probe** لربطهم ببعض.

---

### الـ Real-World Analogy — وكالة توظيف

تخيل إن في **وكالة توظيف** (الـ driver core):

- **الـ job seekers** = الـ drivers (عندهم skills محددة، بيسجلوا نفسهم في الوكالة).
- **الـ companies** = الـ devices (بيظهروا لما يتوصل hardware جديد، بيسجلوا احتياجهم).
- **الـ industry sector** = الـ bus_type (e.g., قطاع IT = PCI bus، قطاع طبي = USB bus).
- **المقابلة الوظيفية** = الـ `match()` function (الوكالة بتحدد إيه الـ driver اللي يناسب الـ device).
- **التعاقد الفعلي** = الـ `probe()` call (الـ driver بيشتغل مع الـ device وبيعمل initialize).
- **فسخ العقد** = الـ `remove()` call.
- **ملف الموظف** = الـ sysfs entry (كل معلومات الـ driver والـ device متاحة للـ userspace).
- **الـ HR system** = الـ kobject/klist (بيتابع كل حاجة، reference counting، cleanup).
- **إشعار الفصل** = الـ uevent (الـ userspace (udev) بيتبلغ بأي تغيير).

الـ driver مش بيدور على الـ device بنفسه — الـ driver core هو اللي بيعمل المطابقة.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USERSPACE                                │
│   udev daemon ◄──── uevents (add/remove/bind) ────────────────  │
│   /sys/bus/*/drivers/   /sys/bus/*/devices/   /sys/devices/    │
└──────────────────────────┬──────────────────────────────────────┘
                           │ sysfs (kernfs)
┌──────────────────────────▼──────────────────────────────────────┐
│                      DRIVER CORE                                │
│                                                                 │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐ │
│   │  bus_type    │    │device_driver │    │     device       │ │
│   │  (bus.h)     │    │  (driver.h)  │    │   (device.h)     │ │
│   │              │    │              │    │                  │ │
│   │ .match()     │◄───│ .probe()     │───►│ .driver (ptr)    │ │
│   │ .probe()     │    │ .remove()    │    │ .bus    (ptr)    │ │
│   │ .remove()    │    │ .shutdown()  │    │ .of_node         │ │
│   │ .uevent()    │    │ .pm          │    │ .devres_head     │ │
│   └──────┬───────┘    └──────┬───────┘    └────────┬─────────┘ │
│          │                   │                     │           │
│          └───────────────────┴──────────┬──────────┘           │
│                                         │                      │
│                              ┌──────────▼──────────┐           │
│                              │   kobject / klist   │           │
│                              │  (reference count,  │           │
│                              │   sysfs nodes,      │           │
│                              │   device lists)     │           │
│                              └─────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                           │
         ┌─────────────────┼──────────────────┐
         ▼                 ▼                  ▼
   ┌───────────┐    ┌───────────┐    ┌──────────────┐
   │ PCI bus   │    │ USB bus   │    │platform bus  │
   │ drivers/  │    │ drivers/  │    │ drivers/     │
   │ pci/      │    │ usb/      │    │ base/        │
   └───────────┘    └───────────┘    └──────────────┘
```

---

### الـ Core Abstractions — التفاصيل الكاملة

#### 1. `struct device_driver` — قلب الموضوع

```c
struct device_driver {
    const char              *name;          /* اسم الـ driver في sysfs */
    const struct bus_type   *bus;           /* الـ bus اللي بيشتغل عليه */

    struct module           *owner;         /* للـ reference counting على الـ module */
    const char              *mod_name;      /* للـ built-in modules */

    bool suppress_bind_attrs;               /* يمنع bind/unbind من sysfs */
    enum probe_type probe_type;             /* sync أو async probe؟ */

    /* firmware matching tables */
    const struct of_device_id    *of_match_table;   /* Device Tree matching */
    const struct acpi_device_id  *acpi_match_table; /* ACPI matching */

    /* lifecycle callbacks */
    int  (*probe)    (struct device *dev);  /* ربط الـ driver بالـ device */
    void (*sync_state)(struct device *dev); /* مزامنة الـ state بعد boot */
    int  (*remove)   (struct device *dev);  /* فك الربط */
    void (*shutdown) (struct device *dev);  /* إيقاف الجهاز وقت shutdown */
    int  (*suspend)  (struct device *dev, pm_message_t state);
    int  (*resume)   (struct device *dev);

    /* sysfs attributes */
    const struct attribute_group **groups;      /* attributes على الـ driver نفسه */
    const struct attribute_group **dev_groups;  /* attributes على كل device مربوط */

    const struct dev_pm_ops *pm;    /* modern power management ops */
    void (*coredump)(struct device *dev);

    struct driver_private *p;       /* private data للـ driver core فقط */
    struct {
        void (*post_unbind_rust)(struct device *dev); /* Rust-only callback */
    } p_cb;
};
```

#### الـ `struct driver_private` — البيانات الداخلية

الـ `p` field هو pointer لـ `struct driver_private` اللي فيه:

```
struct driver_private {
    struct kobject kobj;        ← sysfs node للـ driver
    struct klist klist_devices; ← قائمة الـ devices المربوطة بيه
    struct klist_node knode_bus;← node في قائمة الـ bus
    struct module_kobject *mkobj;
    struct device_driver *driver;
};
```

**الـ driver core بس** اللي يلمس الـ `p` — الـ driver نفسه لا.

---

#### 2. `struct bus_type` — الـ Bus كـ Policy Layer

الـ bus مش بس "قناة hardware" — هو **policy layer** بيقرر:

- **إزاي** يتم الـ matching (الـ `match()` callback).
- **مين** يعمل الـ probe الفعلي (الـ bus أو الـ driver).
- **إيه** الـ DMA configuration.
- **إزاي** يتم الـ power management على مستوى الـ bus.

```c
struct bus_type {
    const char  *name;          /* "pci", "usb", "platform", ... */
    const char  *dev_name;      /* format string لأسماء الـ devices */

    /* sysfs attribute groups */
    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;

    /* THE most important callback */
    int (*match)(struct device *dev, const struct device_driver *drv);

    int (*probe) (struct device *dev);      /* bus-level probe wrapper */
    void (*remove)(struct device *dev);
    void (*sync_state)(struct device *dev);
    void (*shutdown)(struct device *dev);

    int (*online) (struct device *dev);     /* hot-plug: bring back online */
    int (*offline)(struct device *dev);     /* hot-plug: prepare for removal */

    int  (*suspend)(struct device *dev, pm_message_t state);
    int  (*resume) (struct device *dev);

    int  (*dma_configure)(struct device *dev);
    void (*dma_cleanup)  (struct device *dev);

    const struct dev_pm_ops *pm;
    bool need_parent_lock;  /* lock parent during probe/remove? */
};
```

**مثال real-world على الـ match():**

```c
/* من drivers/usb/core/driver.c */
static int usb_device_match(struct device *dev, const struct device_driver *drv)
{
    /* USB بيعمل match على أساس idVendor:idProduct أو class/subclass */
    if (is_usb_device(dev)) {
        if (is_usb_device_driver(drv))
            return 1; /* usb_generic_driver يمسك أي USB device */
        return 0;
    }
    /* USB interface matching: بيشوف usb_device_id table */
    return usb_match_id(to_usb_interface(dev),
                        to_usb_driver(drv)->id_table) != NULL;
}
```

---

#### 3. `struct kobject` — الأساس اللي كل حاجة مبنية عليه

> **مهم:** الـ kobject subsystem هو الـ foundation اللي الـ driver model بُني عليه. هو بيوفر: reference counting، sysfs integration، uevent notification.

```c
struct kobject {
    const char          *name;          /* اسم الـ directory في sysfs */
    struct list_head     entry;         /* linked list entry */
    struct kobject      *parent;        /* الـ parent في الـ hierarchy */
    struct kset         *kset;          /* الـ set اللي ينتمي ليه */
    const struct kobj_type *ktype;      /* الـ type: release, sysfs_ops */
    struct kernfs_node  *sd;            /* sysfs directory entry */
    struct kref          kref;          /* reference counter */
    /* state flags ... */
};
```

**الـ kobject hierarchy** هو اللي بيعمل الـ `/sys` tree:

```
/sys/
├── bus/
│   ├── pci/
│   │   ├── devices/   ← symlinks للـ devices
│   │   └── drivers/   ← directories للـ drivers
│   └── usb/
│       ├── devices/
│       └── drivers/
└── devices/
    └── pci0000:00/    ← الـ physical hierarchy
        └── 0000:00:1f.2/
            └── driver → ../../../../bus/pci/drivers/ahci
```

كل `device_driver` عنده `kobject` جوه الـ `driver_private->kobj`. الـ path بيكون:
`/sys/bus/<bus_name>/drivers/<driver_name>/`

---

### الـ Probe Flow — من Registration للـ Binding

ده الـ flow الكامل لما driver وdevice بيلتقوا:

```
driver_register(drv)
        │
        ▼
bus_add_driver(drv)
        │
        ├── يضيف الـ driver في bus->p->klist_drivers
        ├── يعمل kobject_init_and_add() → /sys/bus/X/drivers/Y/
        │
        └── driver_attach(drv)
                │
                ▼
        bus_for_each_dev() ← بيلف على كل device في الـ bus
                │
                ▼
        __driver_attach(dev, drv)
                │
                ▼
        driver_match_device(drv, dev)
                │   يستدعي bus->match(dev, drv)
                │   لو الـ bus مالوش match، بيشوف of_match_table
                │
         ┌──── match ✓ ────┐
         │                 │
         ▼                 ▼
   device_driver_attach()
         │
         ▼
   really_probe(dev, drv)
         │
         ├── dev->driver = drv   ← ربط مبدئي
         ├── bus->probe(dev)     ← لو الـ bus عنده probe
         │       أو
         └── drv->probe(dev)     ← لو الـ bus مالوش
                 │
         ┌───── نجح ─────┐
         │               │
         ▼               ▼
   device bound!     -EPROBE_DEFER
   uevent BIND       ← يتضاف لـ deferred list
                     ← يتجرب تاني بعدين
```

---

### الـ `enum probe_type` — متى يكون الـ Probe Async؟

```c
enum probe_type {
    PROBE_DEFAULT_STRATEGY,       /* الـ core يقرر */
    PROBE_PREFER_ASYNCHRONOUS,    /* للـ slow devices، متوقفش الـ boot */
    PROBE_FORCE_SYNCHRONOUS,      /* لازم يكمل قبل ما الـ init يكمل */
};
```

**مثال:** الـ storage drivers غالباً `PROBE_FORCE_SYNCHRONOUS` لأن الـ rootfs محتاجة تكون mounted قبل الـ boot يكمل. أما display drivers ممكن تكون `PROBE_PREFER_ASYNCHRONOUS`.

---

### الـ `sync_state` — مشكلة الـ Boot Dependencies

ده concept مهم خصوصاً في الـ embedded:

**المشكلة:** الـ regulator driver بيشتغل قبل ما الـ consumer driver (مثلاً GPU) يعمل probe. الـ regulator ممكن يكون عامل voltage عالي مش محتاجه في الـ runtime. إمتى يقدر يخفضه؟

**الحل:** `sync_state` بيتبعت للـ device **بس بعد ما** كل الـ consumers اللي موجودين وقت `late_initcall` اتربطوا بـ drivers. يعني:

```
late_initcall:
    - كل الـ consumers اللي expected موجودين وعملوا probe بنجاح؟
    - نعم → اعمل sync_state على الـ provider (الـ regulator)
    - لأ → استنى لحد ما يعملوا probe
```

---

### الـ sysfs Interface — الـ Driver Attributes

```c
struct driver_attribute {
    struct attribute attr;              /* name, mode (permissions) */
    ssize_t (*show) (struct device_driver *driver, char *buf);
    ssize_t (*store)(struct device_driver *driver,
                     const char *buf, size_t count);
};
```

**الـ macros للتسهيل:**

```c
/* read/write attribute */
DRIVER_ATTR_RW(my_setting);
/* ينشئ: struct driver_attribute driver_attr_my_setting */
/* الـ file: /sys/bus/<bus>/drivers/<drv>/my_setting */

/* مثال: i2c driver عنده attribute لـ scan timeout */
static ssize_t timeout_show(struct device_driver *drv, char *buf)
{
    struct my_i2c_drv *d = container_of(drv, struct my_i2c_drv, driver);
    return sprintf(buf, "%d\n", d->timeout_ms);
}
static ssize_t timeout_store(struct device_driver *drv,
                              const char *buf, size_t count)
{
    /* ... parse and set ... */
    return count;
}
static DRIVER_ATTR_RW(timeout);
```

---

### الـ `module_driver` Macro — إزاي الـ Drivers بتسجل نفسها

الـ boilerplate اللي كل driver بيعمله:

```c
/* بدون الـ macro: */
static int __init my_drv_init(void) {
    return platform_driver_register(&my_drv);
}
static void __exit my_drv_exit(void) {
    platform_driver_unregister(&my_drv);
}
module_init(my_drv_init);
module_exit(my_drv_exit);

/* مع الـ macro: */
module_driver(my_drv, platform_driver_register, platform_driver_unregister);
/* أو أبسط: */
module_platform_driver(my_drv);
/* ده macro فوق macro:
 * module_platform_driver(drv) →
 *   module_driver(drv, platform_driver_register, platform_driver_unregister)
 */
```

الـ `builtin_driver` نفس الفكرة لكن بدون `__exit` — للـ drivers اللي compiled-in ومش modules.

---

### الـ Deferred Probe — حل مشكلة الترتيب

**المشكلة:** الـ driver B محتاج resource من الـ driver A (مثلاً GPIO controller). لو B اتـ probe قبل A، هيفشل.

**الحل:** الـ `-EPROBE_DEFER` return value:

```c
static int my_driver_probe(struct device *dev)
{
    struct gpio_desc *gpio;

    /* لو الـ GPIO controller لسه مش جاهز */
    gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
    if (IS_ERR(gpio)) {
        if (PTR_ERR(gpio) == -EPROBE_DEFER)
            return -EPROBE_DEFER;  /* جرب تاني بعدين */
        return PTR_ERR(gpio);
    }
    /* ...  */
}
```

الـ driver core بيضيف الـ device في **deferred probe list**، وبعد ما أي driver تاني يعمل probe بنجاح، بيجرب الـ deferred devices تاني.

```
driver_deferred_probe_add(dev)  ← يضيف في القائمة
driver_deferred_probe_check_state(dev) ← يفحص إيه الـ state
```

---

### الـ Driver Find APIs

```c
/* دور على device باسمه عند driver معين */
struct device *driver_find_device_by_name(const struct device_driver *drv,
                                           const char *name);

/* دور على device بـ of_node */
struct device *driver_find_device_by_of_node(const struct device_driver *drv,
                                              const struct device_node *np);

/* دور على device بـ fwnode (firmware node — يشمل DT وACPI) */
struct device *driver_find_device_by_fwnode(struct device_driver *drv,
                                             const struct fwnode_handle *fwnode);

/* iterate على كل devices عند driver */
int driver_for_each_device(struct device_driver *drv,
                            struct device *start,
                            void *data, device_iter_t fn);
```

كلهم بيستخدموا `driver_find_device()` الـ generic اللي بتاخد `device_match_t` function pointer.

---

### الـ Ownership — ايه اللي الـ Driver Core بيمتلكه؟

```
┌─────────────────────────────────────────────────────┐
│             DRIVER CORE يمتلك:                      │
│                                                     │
│  ✓ الـ matching logic (بتاع الـ bus)               │
│  ✓ الـ probe orchestration (deferred, async, sync)  │
│  ✓ الـ sysfs hierarchy                              │
│  ✓ الـ uevent emission                              │
│  ✓ الـ reference counting (kobject/kref)            │
│  ✓ الـ driver/device lists (klist)                  │
│  ✓ الـ bind/unbind logic                           │
│  ✓ الـ devres (devm_*) cleanup على remove          │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│             الـ DRIVER يمتلك:                       │
│                                                     │
│  ✓ الـ probe() logic (device-specific init)        │
│  ✓ الـ remove() logic (device-specific cleanup)    │
│  ✓ الـ interrupt handling                           │
│  ✓ الـ hardware register access                    │
│  ✓ الـ power management ops (dev_pm_ops)           │
│  ✓ الـ match tables (of_match_table, id_table)     │
│  ✗ لا يعرف إيه الـ device الجاي بعده              │
│  ✗ لا يعرف إمتى يتبعت الـ probe                   │
└─────────────────────────────────────────────────────┘
```

---

### الـ Struct Relationship Diagram

```
struct bus_type
    │
    │  "أنا الـ bus اللي عليه كل حاجة"
    │
    ├──► klist of devices ──────────► struct device
    │         │                           │
    │         │                           ├── struct kobject (sysfs node)
    │         │                           ├── struct device_driver *driver
    │         │                           ├── const struct bus_type *bus
    │         │                           └── struct dev_pm_ops *pm
    │         │
    └──► klist of drivers ──────────► struct device_driver
              │                           │
              │                           ├── struct driver_private *p
              │                           │       └── struct kobject kobj
              │                           │       └── klist klist_devices
              │                           ├── const struct bus_type *bus
              │                           ├── of_match_table
              │                           ├── probe()
              │                           ├── remove()
              │                           └── pm ops
              │
              └── bus->match(dev, drv) ← الـ core بيستدعيه لكل زوج
```

---

### خلاصة

الـ driver model هو **unified contract** بين ٣ أطراف:
- **الـ hardware** (ممثل بـ `struct device`)
- **الـ software** (ممثل بـ `struct device_driver`)
- **الـ protocol** اللي بيربطهم (ممثل بـ `struct bus_type`)

الـ driver core هو الـ **matchmaker** المركزي — بيعرف الـ devices، بيعرف الـ drivers، وبيقرر إمتى وإزاي يلتقوا، مع ضمان الـ visibility الكاملة للـ userspace عبر sysfs والـ uevents.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### `enum probe_type`

| القيمة | المعنى | متى تستخدمه |
|--------|--------|-------------|
| `PROBE_DEFAULT_STRATEGY` | الـ core يقرر sync أو async | الغالبية العظمى من الـ drivers |
| `PROBE_PREFER_ASYNCHRONOUS` | الـ driver يفضّل probe بشكل async | أجهزة بطيئة التهيئة (USB storage مثلاً) |
| `PROBE_FORCE_SYNCHRONOUS` | الـ probe لازم يخلص قبل ما يكمل الـ boot | أجهزة critical زي الـ root filesystem controller |

#### `enum bus_notifier_event`

| الـ Event | وقت إطلاقه |
|-----------|------------|
| `BUS_NOTIFY_ADD_DEVICE` | لما device تتضاف للـ bus |
| `BUS_NOTIFY_DEL_DEVICE` | قبل ما device تتشال |
| `BUS_NOTIFY_REMOVED_DEVICE` | بعد ما تتشال فعلاً |
| `BUS_NOTIFY_BIND_DRIVER` | قبل ما driver يتربط بـ device |
| `BUS_NOTIFY_BOUND_DRIVER` | بعد ما driver اتربط بنجاح |
| `BUS_NOTIFY_UNBIND_DRIVER` | قبل ما driver يتفك |
| `BUS_NOTIFY_UNBOUND_DRIVER` | بعد ما driver اتفك |
| `BUS_NOTIFY_DRIVER_NOT_BOUND` | لو الـ probe فشل |

#### Config Options المهمة

| الـ Option | التأثير |
|-----------|---------|
| `CONFIG_ACPI` | يفعّل `driver_find_device_by_acpi_dev()` وربط الـ ACPI companion |
| `suppress_bind_attrs` (field) | يمنع الـ bind/unbind عبر `/sys/bus/.../drivers/` |
| `need_parent_lock` (bus field) | الـ core يلاقي parent lock وقت الـ probe/remove |

---

### الـ Structs المهمة

#### 1. `struct device_driver`

**الغرض:** ده العمود الفقري لأي driver في الـ kernel — بيمثّل الـ driver نفسه (مش instance جهاز معين).

```c
struct device_driver {
    const char              *name;           // اسم الـ driver — بيظهر في sysfs
    const struct bus_type   *bus;            // الـ bus اللي الـ driver شغّال عليه

    struct module           *owner;          // الـ module اللي بيحمل الـ driver (THIS_MODULE)
    const char              *mod_name;       // للـ built-in modules (مش loadable)

    bool suppress_bind_attrs;               // true = منع bind/unbind من sysfs
    enum probe_type probe_type;             // sync / async / default

    const struct of_device_id    *of_match_table;   // Device Tree matching table
    const struct acpi_device_id  *acpi_match_table;  // ACPI matching table

    int  (*probe)    (struct device *dev);  // ربط الـ driver بـ device
    void (*sync_state)(struct device *dev); // مزامنة الـ state بعد كل الـ consumers يتربطوا
    int  (*remove)   (struct device *dev);  // فك الربط لما device تتشال
    void (*shutdown) (struct device *dev);  // وقت الـ system shutdown
    int  (*suspend)  (struct device *dev, pm_message_t state); // دخول وضع النوم
    int  (*resume)   (struct device *dev);  // الصحيان من النوم

    const struct attribute_group **groups;      // attributes للـ driver نفسه في sysfs
    const struct attribute_group **dev_groups;  // attributes بتتضاف للـ device بعد الـ bind

    const struct dev_pm_ops *pm;            // power management ops (أحدث من suspend/resume)
    void (*coredump)(struct device *dev);   // لما يُطلب coredump من sysfs

    struct driver_private *p;              // بيانات private للـ driver core (لا تلمس)
    struct {
        void (*post_unbind_rust)(struct device *dev); // Rust-only: بعد remove وبعد devres
    } p_cb;
};
```

**العلاقات:**
- بيشاور على `bus_type` — كل driver عارف عليه bus شغّال.
- الـ `driver_private *p` بيحمل الـ klist nodes اللي بتربط الـ driver بالـ devices المتصلة بيه.
- الـ `dev_pm_ops *pm` بيوفّر granular PM hooks أحدث وأكمل من `suspend`/`resume` البسيطين.

---

#### 2. `struct driver_attribute`

**الغرض:** بيمثّل attribute واحد في sysfs تحت مسار الـ driver (`/sys/bus/<bus>/drivers/<drv>/<attr>`).

```c
struct driver_attribute {
    struct attribute attr;                           // اسم + permissions
    ssize_t (*show) (struct device_driver *, char *buf);          // قراءة
    ssize_t (*store)(struct device_driver *, const char *buf, size_t count); // كتابة
};
```

**المacros المتاحة:**

```c
DRIVER_ATTR_RW(name)  // read + write
DRIVER_ATTR_RO(name)  // read only
DRIVER_ATTR_WO(name)  // write only
```

**مثال عملي:**
```c
// driver يعرض version في sysfs
static ssize_t version_show(struct device_driver *drv, char *buf)
{
    return sprintf(buf, "2.1.0\n");
}
static DRIVER_ATTR_RO(version);

// في driver_register أو بعدها:
driver_create_file(&my_driver, &driver_attr_version);
```

---

#### 3. `struct bus_type`

**الغرض:** بيعرّف نوع الـ bus (PCI, USB, I2C, platform, إلخ) — بيحدد ازاي الـ devices والـ drivers بيتقابلوا.

```c
struct bus_type {
    const char  *name;         // "pci", "usb", "i2c", "platform", ...
    const char  *dev_name;     // template للـ device names ("i2c%d")

    const struct attribute_group **bus_groups; // attrs للـ bus نفسه
    const struct attribute_group **dev_groups; // attrs لكل device على الـ bus
    const struct attribute_group **drv_groups; // attrs لكل driver على الـ bus

    int  (*match)(struct device *dev, const struct device_driver *drv); // matching logic
    int  (*uevent)(const struct device *dev, struct kobj_uevent_env *);  // uevent env
    int  (*probe)(struct device *dev);           // bus-level probe (قبل driver probe)
    void (*sync_state)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    int  (*online)(struct device *dev);          // hot-plug online
    int  (*offline)(struct device *dev);         // hot-removal offline

    int  (*suspend)(struct device *dev, pm_message_t state);
    int  (*resume)(struct device *dev);
    int  (*num_vf)(struct device *dev);          // SR-IOV virtual functions count

    int  (*dma_configure)(struct device *dev);   // DMA setup
    void (*dma_cleanup)(struct device *dev);     // DMA teardown

    const struct dev_pm_ops *pm;
    bool need_parent_lock;                       // lock parent أثناء probe/remove
};
```

---

#### 4. `struct bus_attribute`

**الغرض:** attribute في sysfs على مستوى الـ bus نفسه (`/sys/bus/<bus>/<attr>`).

```c
struct bus_attribute {
    struct attribute attr;
    ssize_t (*show) (const struct bus_type *bus, char *buf);
    ssize_t (*store)(const struct bus_type *bus, const char *buf, size_t count);
};
```

---

#### 5. `struct klist` و `struct klist_node`

**الغرض:** linked list آمن من ناحية الـ concurrency — بيستخدم reference counting عشان الـ iteration تكون safe حتى لو حصل removal في نفس الوقت.

```c
struct klist {
    spinlock_t       k_lock;  // يحمي الـ list نفسها
    struct list_head k_list;  // الـ list الفعلية
    void (*get)(struct klist_node *); // زيادة الـ refcount
    void (*put)(struct klist_node *); // نقص الـ refcount (ممكن يحذف)
};

struct klist_node {
    void            *n_klist; // pointer للـ parent klist (internal)
    struct list_head n_node;  // entry في الـ list
    struct kref      n_ref;   // reference counter
};
```

**الـ driver core بيستخدمهم في `driver_private` لتخزين:**
- قائمة الـ devices المربوطة بالـ driver
- قائمة الـ drivers على الـ bus

---

### رسم العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────────────┐
                    │            struct bus_type               │
                    │  name, match(), probe(), pm, ...         │
                    │  bus_groups, dev_groups, drv_groups      │
                    └──────────────┬──────────────────────────┘
                                   │  (1 bus : N drivers)
                                   │  via bus->p->drivers_klist
                                   │
              ┌────────────────────▼──────────────────────────┐
              │           struct device_driver                  │
              │  name, probe(), remove(), shutdown()           │
              │  of_match_table, acpi_match_table              │
              │  pm → struct dev_pm_ops                        │
              │  bus ──────────────────────────────────────────┼──► struct bus_type
              │  owner ────────────────────────────────────────┼──► struct module
              │  p ─────────────────────────────────────────── ┼──► struct driver_private
              └───────────────────────────────────────────────┘
                              │                    │
                              │ groups             │ dev_groups
                              ▼                    ▼
                  struct attribute_group    struct attribute_group
                  (driver sysfs attrs)      (device sysfs attrs after bind)


              ┌───────────────────────────────────────────────┐
              │           struct driver_private (opaque)       │
              │  struct klist   klist_devices  ─────────────── ┼──► [device1, device2, ...]
              │  struct klist_node knode_bus   ─────────────── ┼──► (node في bus drivers list)
              │  struct kobject  kobj          ─────────────── ┼──► sysfs entry
              └───────────────────────────────────────────────┘


              ┌───────────────────────────────────────────────┐
              │           struct driver_attribute              │
              │  struct attribute  attr  (name + mode)        │
              │  show() / store()                             │
              └───────────────────────────────────────────────┘
                              │
                              │ driver_create_file()
                              ▼
                   /sys/bus/<bus>/drivers/<drv>/<attr>


              ┌───────────────────────────────────────────────┐
              │            struct bus_attribute                │
              │  struct attribute  attr                       │
              │  show() / store()                             │
              └───────────────────────────────────────────────┘
                              │
                              │ bus_create_file()
                              ▼
                   /sys/bus/<bus>/<attr>
```

---

### دورة حياة الـ Driver — Lifecycle Diagram

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     DRIVER LIFECYCLE                                │
  └─────────────────────────────────────────────────────────────────────┘

  [1] DEFINITION
      ┌────────────────────────────────────────┐
      │  static struct my_driver = {           │
      │      .name  = "my_driver",             │
      │      .bus   = &platform_bus_type,      │
      │      .probe = my_probe,                │
      │      .remove= my_remove,               │
      │  };                                    │
      └────────────────┬───────────────────────┘
                       │
  [2] REGISTRATION     ▼
      ┌────────────────────────────────────────┐
      │  driver_register(&my_driver)           │
      │    → alloc driver_private              │
      │    → kobject_init + add (sysfs entry)  │
      │    → klist_add to bus->p->drivers_klist│
      │    → create default sysfs groups       │
      │    → driver_attach() → try probe       │
      └────────────────┬───────────────────────┘
                       │
  [3] PROBE            ▼
      ┌────────────────────────────────────────┐
      │  for each device on bus:               │
      │    bus->match(dev, drv) == true ?      │
      │      → driver_probe_device(dev, drv)  │
      │        → drv->probe(dev) called        │
      │          → device gets driver bound    │
      │          → dev_groups added to device  │
      └────────────────┬───────────────────────┘
                       │
  [4] USAGE            ▼
      ┌────────────────────────────────────────┐
      │  Driver handles:                       │
      │    - IRQs, I/O ops via device          │
      │    - PM: suspend() / resume()          │
      │    - sync_state() (deferred consumers) │
      │    - sysfs attrs (show/store)          │
      └────────────────┬───────────────────────┘
                       │
  [5] REMOVE           ▼
      ┌────────────────────────────────────────┐
      │  device_release_driver(dev)            │
      │    → drv->remove(dev)                  │
      │    → devres cleanup                    │
      │    → p_cb.post_unbind_rust() [Rust]    │
      │    → dev_groups removed from device    │
      │    → device unbound from driver        │
      └────────────────┬───────────────────────┘
                       │
  [6] UNREGISTRATION   ▼
      ┌────────────────────────────────────────┐
      │  driver_unregister(&my_driver)         │
      │    → detach all remaining devices      │
      │    → klist_remove from bus list        │
      │    → kobject_put (sysfs removal)       │
      │    → free driver_private               │
      └────────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### Flow 1: `driver_register()` — تسجيل driver جديد

```
driver_register(drv)
  │
  ├─► other_driver_has_same_name? → return -EBUSY
  │
  ├─► bus_add_driver(drv)
  │     ├─► alloc + init driver_private (drv->p)
  │     ├─► kobject_init_and_add()
  │     │     └─► creates /sys/bus/<bus>/drivers/<drv>/
  │     ├─► klist_add_tail(&drv->p->knode_bus, &bus->p->drivers_klist)
  │     ├─► driver_create_file() → bind/unbind attrs (if !suppress_bind_attrs)
  │     ├─► driver_add_groups(drv->groups)
  │     └─► driver_attach(drv)
  │           └─► bus_for_each_dev()
  │                 └─► __driver_attach(dev, drv)
  │                       ├─► driver_match_device(drv, dev)
  │                       │     └─► bus->match(dev, drv)
  │                       └─► driver_probe_device(drv, dev)
  │                             ├─► really_probe(dev, drv)
  │                             │     ├─► dev->driver = drv
  │                             │     ├─► drv->probe(dev)   ◄── driver code runs here
  │                             │     ├─► device_add_groups(dev, drv->dev_groups)
  │                             │     └─► driver_bound(dev)
  │                             └─► [on failure] driver_deferred_probe_add(dev)
  │
  └─► add_driver_module_link(drv)  → module reference
```

#### Flow 2: `driver_unregister()` — إلغاء تسجيل

```
driver_unregister(drv)
  │
  ├─► wait_for_device_probe()       → انتظر أي probe جارية تخلص
  │
  └─► bus_remove_driver(drv)
        ├─► driver_detach(drv)
        │     └─► for each device in drv->p->klist_devices:
        │           device_release_driver(dev)
        │             ├─► drv->remove(dev)
        │             ├─► devres_release_all(dev)
        │             ├─► p_cb.post_unbind_rust(dev)  [if set]
        │             └─► dev->driver = NULL
        │
        ├─► driver_remove_groups(drv->groups)
        ├─► driver_remove_file() → bind/unbind attrs
        ├─► klist_remove(&drv->p->knode_bus)
        └─► kobject_put() → sysfs removal
```

#### Flow 3: `driver_find_device()` — البحث عن device

```
driver_find_device(drv, start, data, match_fn)
  │
  └─► bus_find_device(drv->bus, start, data, match_fn)
        │
        └─► klist_iter_init(bus->p->klist_devices)
              │
              └─► loop: klist_next()
                    ├─► get device reference (kref_get)
                    ├─► match_fn(dev, data) == true ?
                    │     └─► return dev  (with reference held)
                    └─► put device reference (kref_put)
```

**الـ match functions المتاحة:**

| الدالة | تطابق على أساس |
|--------|---------------|
| `device_match_name` | اسم الـ device |
| `device_match_of_node` | `device_node` في Device Tree |
| `device_match_fwnode` | `fwnode_handle` |
| `device_match_devt` | `dev_t` (major:minor) |
| `device_match_acpi_dev` | ACPI companion device |
| `device_match_any` | أي device (iteration) |

#### Flow 4: `module_driver()` macro expansion

```c
// الـ macro:
module_driver(my_drv, platform_driver_register, platform_driver_unregister);

// يتمدد لـ:
static int __init my_drv_init(void) {
    return platform_driver_register(&my_drv);
}
module_init(my_drv_init);

static void __exit my_drv_exit(void) {
    platform_driver_unregister(&my_drv);
}
module_exit(my_drv_exit);
```

```
insmod my_driver.ko
  │
  └─► my_drv_init()
        └─► platform_driver_register(&my_drv)
              └─► driver_register(&my_drv.driver)
                    └─► [Flow 1 above]

rmmod my_driver.ko
  │
  └─► my_drv_exit()
        └─► platform_driver_unregister(&my_drv)
              └─► driver_unregister(&my_drv.driver)
                    └─► [Flow 2 above]
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة وحمايتها

| الـ Lock | النوع | يحمي |
|---------|------|------|
| `device->mutex` | `struct mutex` | bind/unbind state، الـ driver pointer، الـ PM state |
| `bus->p->drivers_klist.k_lock` | `spinlock_t` | قائمة الـ drivers على الـ bus |
| `bus->p->klist_devices.k_lock` | `spinlock_t` | قائمة الـ devices على الـ bus |
| `drv->p->klist_devices.k_lock` | `spinlock_t` | قائمة الـ devices المربوطة بالـ driver |
| `klist_node::n_ref` (kref) | atomic | حماية الـ nodes من الحذف أثناء الـ iteration |
| `device_lock(dev->parent)` | `mutex` | لو `bus->need_parent_lock == true` |

#### ترتيب الـ Locks (Lock Ordering)

```
الترتيب الصحيح (من الخارج للداخل):

  parent->mutex  (إذا need_parent_lock)
       │
       ▼
  dev->mutex
       │
       ▼
  klist spinlock  (للـ iteration فقط — لا تمسك معاها mutex)
```

**قاعدة مهمة:** الـ bus notifiers بيتبعتوا وهو مع `device->mutex` مشغول — ممنوع تحاول تعمل lock على نفس الـ device جوّا الـ notifier callback أو هتاخد deadlock.

#### الـ Deferred Probe والـ Async Probe

```
probe_type = PROBE_PREFER_ASYNCHRONOUS:

  driver_attach()
    └─► async_schedule(__driver_attach, dev)
          │ (runs in worker thread)
          └─► driver_probe_device()
                └─► device_lock(dev)   ← هنا بس تاخد lock
                      └─► really_probe()


probe returns -EPROBE_DEFER:

  really_probe()
    └─► return -EPROBE_DEFER
          └─► driver_deferred_probe_add(dev)
                └─► يضيف dev لـ deferred_probe_pending_list
                      └─► workqueue بيعيد المحاولة لاحقاً
                            (always asynchronously, even if FORCE_SYNCHRONOUS)
```

#### الـ `driver_private` — البيانات الـ Internal

```
struct driver_private {
    struct kobject        kobj;          // sysfs representation
    struct klist          klist_devices; // كل devices المربوطة بالـ driver
    struct klist_node     knode_bus;     // entry في bus->drivers list
    struct module_kobject *mkobj;        // ربط الـ driver بالـ module
    struct device_driver  *driver;       // back pointer للـ driver
};
```

الـ `driver_private` مش معرّف في `driver.h` — موجود في `drivers/base/base.h` وده قصد عمدي عشان يكون opaque تماماً لأي كود خارج الـ driver core.

---

### ملخص العلاقات الكاملة

```
  struct module
       ▲
       │ owner
       │
  ┌────┴─────────────────────────────────────────────────────────┐
  │                   struct device_driver                        │
  │   name ─────────────────────────────────────► sysfs entry   │
  │   bus ──────────────────────────────────────► struct bus_type│
  │   probe_type ────────────────────────────────► enum          │
  │   of_match_table ───────────────────────────► OF matching   │
  │   acpi_match_table ─────────────────────────► ACPI matching │
  │   probe/remove/shutdown/suspend/resume ──────► callbacks     │
  │   sync_state ───────────────────────────────► deferred sync │
  │   groups ───────────────────────────────────► driver attrs  │
  │   dev_groups ───────────────────────────────► device attrs  │
  │   pm ───────────────────────────────────────► dev_pm_ops    │
  │   coredump ─────────────────────────────────► debug         │
  │   p ────────────────────────────────────────► driver_private│
  │   p_cb.post_unbind_rust ────────────────────► Rust cleanup  │
  └──────────────────────────────────────────────────────────────┘
                              │
                              │ p
                              ▼
  ┌───────────────────────────────────────────────────────────────┐
  │                  struct driver_private                         │
  │  kobj ──────────────────────────────────────► kobject (sysfs)│
  │  klist_devices ─────────────────────────────► [dev, dev, ...]│
  │  knode_bus ─────────────────────────────────► bus drv list   │
  └───────────────────────────────────────────────────────────────┘
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ APIs — Cheatsheet

#### Registration & Lifecycle

| Function | Signature (مختصر) | الغرض |
|---|---|---|
| `driver_register` | `int driver_register(struct device_driver *drv)` | تسجيل الـ driver في الـ driver core |
| `driver_unregister` | `void driver_unregister(struct device_driver *drv)` | إزالة الـ driver من النظام |
| `driver_init` | `void driver_init(void)` | تهيئة الـ driver core نفسه عند boot |

#### Discovery & Search

| Function | الغرض |
|---|---|
| `driver_find` | إيجاد driver بالاسم والـ bus |
| `driver_find_device` | إيجاد device مرتبط بـ driver باستخدام match callback |
| `driver_find_device_by_name` | إيجاد device بالاسم |
| `driver_find_device_by_of_node` | إيجاد device بالـ DT node |
| `driver_find_device_by_fwnode` | إيجاد device بالـ fwnode |
| `driver_find_device_by_devt` | إيجاد device بالـ dev_t |
| `driver_find_next_device` | الـ device التالي في قائمة الـ driver |
| `driver_find_device_by_acpi_dev` | إيجاد device بالـ ACPI companion |

#### Iteration

| Function | الغرض |
|---|---|
| `driver_for_each_device` | تطبيق callback على كل device مرتبط بالـ driver |

#### sysfs Attributes

| Function | الغرض |
|---|---|
| `driver_create_file` | إنشاء sysfs attribute تحت `/sys/bus/<bus>/drivers/<drv>/` |
| `driver_remove_file` | حذف sysfs attribute |
| `driver_set_override` | كتابة قيمة override string في sysfs |

#### Deferred Probe

| Function | الغرض |
|---|---|
| `driver_deferred_probe_add` | إضافة device لقائمة الـ deferred probe |
| `driver_deferred_probe_check_state` | التحقق من حالة الـ deferred probe |

#### Boot Synchronization

| Function | الغرض |
|---|---|
| `driver_probe_done` | هل انتهت كل الـ probes الأولية؟ |
| `wait_for_device_probe` | انتظار انتهاء كل الـ probe المعلق |
| `wait_for_init_devices_probe` | انتظار كل الـ init devices |

#### Macros

| Macro | الغرض |
|---|---|
| `module_driver` | boilerplate لـ module_init/exit |
| `builtin_driver` | boilerplate لـ built-in drivers |
| `DRIVER_ATTR_RW/RO/WO` | إنشاء `driver_attribute` struct |

---

### Group 1: Registration & Lifecycle

الـ registration group هو أساس الـ driver model. لما driver يتسجل، الـ core يربطه بالـ bus المناسبة ويعمل match مع كل الـ devices الموجودة، وكمان يعمل probe لو الـ match نجح.

---

#### `driver_register`

```c
int __must_check driver_register(struct device_driver *drv);
```

بتسجل الـ `device_driver` في الـ kernel driver core. داخليًا بتعمل `bus_add_driver()` اللي بتضيف الـ driver لـ `bus->p->klist_drivers`، وبتعمل `kobject_init_and_add` عشان تظهر في sysfs. بعد كده بتعمل `driver_attach()` اللي بتلف على كل الـ devices المسجلة على الـ bus وتحاول probe.

**Parameters:**
- `drv`: pointer للـ `device_driver` struct المهيأ — لازم يكون `drv->bus` محدد.

**Return:**
- `0` عند النجاح.
- كود خطأ سالب (مثلاً `-EBUSY` لو الـ driver مسجل فعلاً).

**Key details:**
- مُعلَّم بـ `__must_check` — لازم تتحقق من الـ return value.
- لو `drv->probe_type == PROBE_PREFER_ASYNCHRONOUS`، الـ probe بيحصل في async context.
- بيستدعي `driver_sysfs_add()` اللي بتعمل `sysfs_create_group` للـ `drv->groups`.
- الـ locking: بيمسك `bus->p->mutex` أثناء إضافة الـ driver للـ klist.

**Who calls it:**
- دايمًا من `module_init()` context، سواء مباشرة أو عبر ماكرو زي `module_i2c_driver`.

---

#### `driver_unregister`

```c
void driver_unregister(struct device_driver *drv);
```

بتشيل الـ driver من النظام. بتعمل `bus_remove_driver()` اللي بتعمل `driver_detach()` أولاً — تفصل كل device مرتبط بالـ driver عن طريق استدعاء `drv->remove()` — ثم بتشيل الـ kobject من sysfs.

**Parameters:**
- `drv`: الـ driver المراد إزالته — لازم يكون مسجل مسبقاً.

**Return:** لا يرجع قيمة.

**Key details:**
- بتنتظر انتهاء كل الـ async probes قبل ما تكمل (`wait_for_device_probe()`).
- بعد الإزالة، الـ `drv->p` بيتحرر — لا تستخدم الـ struct تاني.
- Context: process context فقط، بتنام لو في operations معلقة.

**Who calls it:**
- من `module_exit()` context.

---

#### `driver_init`

```c
void driver_init(void);
```

بتهيئ الـ driver subsystem كله عند early boot. بتعمل `devtmpfs_init()` و `devices_init()` و `buses_init()` وتحضر الـ infrastructure الأساسية.

**Parameters:** لا يوجد.

**Return:** لا يوجد.

**Key details:**
- مُعلَّمة بـ `__init` — بتتنفذ مرة واحدة عند boot وبيتحرر الكود.
- بيتم استدعاؤها من `driver_init()` في `init/main.c` قبل أي driver يتسجل.

---

### Group 2: Discovery & Search

الـ search functions بتسمح لأي code يلاقي driver أو device بكريتيريا مختلفة. كلها تقريبًا wrappers حوالين `driver_find_device()` الأساسية.

---

#### `driver_find`

```c
struct device_driver *driver_find(const char *name, const struct bus_type *bus);
```

بتدور على driver معين بالاسم على bus معينة. بتبحث في `bus->p->drivers_kset` عن kobject بنفس الاسم.

**Parameters:**
- `name`: اسم الـ driver (نفس `drv->name`).
- `bus`: الـ bus type المراد البحث فيها.

**Return:**
- Pointer لـ `device_driver` لو لاقاه، مع increment للـ reference count.
- `NULL` لو مش موجود.

**Key details:**
- بترجع pointer مع ref count مرفوع — لازم تعمل `put_driver()` بعدين.
- لا تستخدم النتيجة بعد `driver_unregister()` بدون reference.

---

#### `driver_find_device`

```c
struct device *driver_find_device(const struct device_driver *drv,
                                  struct device *start,
                                  const void *data,
                                  device_match_t match);
```

الـ core iterator اللي كل الـ `driver_find_device_by_*` بترجع إليها. بتلف على كل device مرتبط بالـ driver (via `drv->p->klist_devices`) وبتستدعي `match()` على كل واحدة.

**Parameters:**
- `drv`: الـ driver اللي هنبحث في devices المرتبطة بيه.
- `start`: لو مش NULL، بتبدأ البحث من بعد هذا الـ device (للـ pagination).
- `data`: بيتبعت لـ `match()` كـ `const void *data`.
- `match`: callback من نوع `int (*)(struct device *dev, const void *data)` — بترجع non-zero لو الـ device هو المطلوب.

**Return:**
- Pointer لأول `device` نجح فيه الـ match، مع `get_device()` (ref count مرفوع).
- `NULL` لو ما لقاش.

**Key details:**
- الـ caller مسؤول عن `put_device()` على النتيجة.
- الـ iteration بتستخدم `klist_iter` اللي thread-safe.

**Pseudocode:**
```
klist_iter_init_node(&drv->p->klist_devices, &i, start)
while (dev = next_device(&i)):
    if match(dev, data):
        get_device(dev)
        klist_iter_exit(&i)
        return dev
klist_iter_exit(&i)
return NULL
```

---

#### `driver_find_device_by_name`

```c
static inline struct device *
driver_find_device_by_name(const struct device_driver *drv, const char *name);
```

Thin wrapper فوق `driver_find_device()` باستخدام `device_match_name`.

**Parameters:**
- `drv`: الـ driver.
- `name`: الاسم المراد مطابقته مع `dev_name(dev)`.

**Return:** نفس `driver_find_device()`.

---

#### `driver_find_device_by_of_node`

```c
static inline struct device *
driver_find_device_by_of_node(const struct device_driver *drv,
                               const struct device_node *np);
```

بتدور على device مرتبط بالـ driver ومرتبط بـ Device Tree node معين. بتستخدم `device_match_of_node` اللي بتقارن `dev->of_node == np`.

**Parameters:**
- `drv`: الـ driver.
- `np`: الـ DT node المراد إيجاد مقابله.

**Return:** الـ `device` المناسب أو `NULL`.

**Key details:**
- مفيد جداً في drivers اللي بتشتغل مع DT وبتحتاج ترجع لـ device من node معروف.

---

#### `driver_find_device_by_fwnode`

```c
static inline struct device *
driver_find_device_by_fwnode(struct device_driver *drv,
                              const struct fwnode_handle *fwnode);
```

مثل السابق لكن بالـ `fwnode_handle` — بيشتغل مع DT وكمان ACPI لأن الـ fwnode هو abstraction فوقيهم.

---

#### `driver_find_device_by_devt`

```c
static inline struct device *
driver_find_device_by_devt(const struct device_driver *drv, dev_t devt);
```

بتدور بالـ `dev_t` (major:minor) — مفيد في character/block device drivers.

---

#### `driver_find_next_device`

```c
static inline struct device *
driver_find_next_device(const struct device_driver *drv, struct device *start);
```

بترجع الـ device التالي في قائمة الـ driver بعد `start`. بتستخدم `device_match_any` اللي دايمًا بترجع 1.

**Key details:**
- مفيد للـ pagination — تعمل loop يدوي على devices الـ driver.

---

#### `driver_find_device_by_acpi_dev`

```c
#ifdef CONFIG_ACPI
static inline struct device *
driver_find_device_by_acpi_dev(const struct device_driver *drv,
                                const struct acpi_device *adev);
#endif
```

بتدور على device بالـ ACPI companion struct. لو `CONFIG_ACPI` مش موجود، بترجع `NULL` دايمًا.

---

### Group 3: Device Iteration

---

#### `driver_for_each_device`

```c
int __must_check driver_for_each_device(struct device_driver *drv,
                                         struct device *start,
                                         void *data,
                                         device_iter_t fn);
```

بتلف على **كل** الـ devices المرتبطة بالـ driver وبتنفذ `fn` على كل واحدة. لو `fn` رجعت قيمة غير صفر، الـ iteration بتوقف.

**Parameters:**
- `drv`: الـ driver اللي هنلف على devices بتاعته.
- `start`: device ابدأ من بعده (لو `NULL` ابدأ من الأول).
- `data`: pointer تعسفي بيتبعت لـ `fn`.
- `fn`: callback من نوع `int (*)(struct device *dev, void *data)`.

**Return:**
- `0` لو اتمت على الكل بنجاح.
- قيمة الـ return الأولى غير الصفرية من `fn`.

**Key details:**
- `__must_check` — لازم تتحقق من الـ return.
- Thread-safe — بتستخدم `klist_iter`.
- الـ fn بتتنفذ مع `device_lock` ممسوك على كل device.

---

### Group 4: sysfs Attribute Management

الـ sysfs attributes بتخلي الـ driver يعرض معلومات أو يقبل أوامر عبر `/sys/bus/<bus>/drivers/<driver>/`.

---

#### `driver_create_file`

```c
int __must_check driver_create_file(const struct device_driver *driver,
                                    const struct driver_attribute *attr);
```

بتضيف sysfs attribute file للـ driver's kobject directory. بتستدعي `sysfs_create_file()` على `driver->p->kobj`.

**Parameters:**
- `driver`: الـ driver المراد إضافة الـ attribute له.
- `attr`: الـ `driver_attribute` struct اللي فيها الاسم والـ show/store callbacks.

**Return:**
- `0` عند النجاح.
- كود خطأ سالب (مثلاً `-EEXIST` لو الـ file موجود).

**Key details:**
- بتنفذ في process context.
- الـ `attr->show` و `attr->store` بيشتغلوا في سياق الـ sysfs I/O.

---

#### `driver_remove_file`

```c
void driver_remove_file(const struct device_driver *driver,
                        const struct driver_attribute *attr);
```

بتشيل sysfs attribute file من الـ driver kobject. Wrapper فوق `sysfs_remove_file()`.

**Parameters:**
- `driver`: الـ driver.
- `attr`: الـ attribute المراد حذفها.

**Return:** لا يوجد.

---

#### `driver_set_override`

```c
int driver_set_override(struct device *dev, const char **override,
                        const char *s, size_t len);
```

بتسمح لـ userspace يكتب override string (مثلاً driver name) لـ device معين عبر sysfs. الـ function بتعمل `kfree` للقيمة القديمة و`kstrdup_and_replace` للجديدة. مفيد في `driver_override` mechanism.

**Parameters:**
- `dev`: الـ device المرتبط بالـ override.
- `override`: pointer لـ pointer الـ string — بيتم تحديثه.
- `s`: الـ string الجديدة من userspace.
- `len`: طول الـ string.

**Return:**
- عدد البايتات المكتوبة عند النجاح.
- كود خطأ سالب.

**Key details:**
- بتمسك `device_lock(dev)` أثناء التعديل.
- لو `s` فاضي أو `\n`، بيعمل clear للـ override.

---

### Group 5: Deferred Probe

الـ deferred probe mechanism بيحل مشكلة الـ driver اللي محتاج resource مش متاح وقت الـ probe. بدل ما يفشل نهائيًا، بيحط نفسه في قائمة انتظار ويجرب تاني.

---

#### `driver_deferred_probe_add`

```c
void driver_deferred_probe_add(struct device *dev);
```

بتضيف الـ device لـ `deferred_probe_pending_list`. بيتم استدعاؤها تلقائيًا من الـ driver core لما `probe()` بترجع `-EPROBE_DEFER`.

**Parameters:**
- `dev`: الـ device اللي عايز ينتظر.

**Return:** لا يوجد.

**Key details:**
- الـ driver core بيعمل reschedule للـ deferred probe list لما أي driver تاني يتسجل.
- لا تستدعيها مباشرة من الـ driver — الـ core بيتعامل معاها.

---

#### `driver_deferred_probe_check_state`

```c
int driver_deferred_probe_check_state(struct device *dev);
```

بتتحقق هل الـ device يستحق ينتظر أكتر أو لا. لو الـ system وصل للـ late_initcall وما اتحلتش المشكلة، بترجع `-ENODEV` بدل `-EPROBE_DEFER`.

**Parameters:**
- `dev`: الـ device المراد التحقق منه.

**Return:**
- `-EPROBE_DEFER`: استمر في الانتظار.
- `-ENODEV`: ما فيش أمل، الـ probe هيفشل نهائياً.

**Key details:**
- مهمة جداً في قرار الـ driver: هل يكمل ينتظر ولا يفشل بشكل نهائي.

---

### Group 6: Boot Synchronization

---

#### `driver_probe_done`

```c
bool __init driver_probe_done(void);
```

بترجع `true` لو كل الـ asynchronous probes الأولية انتهت. بتفحص الـ `atomic_read(&probe_count)` الداخلي.

**Parameters:** لا يوجد.

**Return:** `true` لو مفيش probe شغال، `false` لو لسه فيه.

**Key details:**
- `__init` — بتستخدم في boot phase فقط.
- مفيدة في `do_basic_setup()` عشان ينتظر قبل mount الـ rootfs.

---

#### `wait_for_device_probe`

```c
void wait_for_device_probe(void);
```

بتنام لحد ما كل الـ pending device probes (sync وasync) تخلص. بتستخدم `wait_event()` على الـ `probe_count` counter.

**Parameters:** لا يوجد.

**Return:** لا يوجد.

**Key details:**
- Process context فقط — ممكن تنام لفترة طويلة.
- بتتحاول في `wait_for_init_devices_probe()` وفي scenarios تانية.

---

#### `wait_for_init_devices_probe`

```c
void __init wait_for_init_devices_probe(void);
```

خصوصاً للـ init sequence — بتنتظر انتهاء كل probes الـ init devices قبل ما الـ system يكمل boot.

**Parameters:** لا يوجد.

**Return:** لا يوجد.

**Key details:**
- `__init` — boot phase فقط.

---

### Group 7: Macros

---

#### `module_driver`

```c
#define module_driver(__driver, __register, __unregister, ...)
```

بيولد `module_init()` و`module_exit()` تلقائياً. بدل ما تكتب:

```c
static int __init my_driver_init(void) {
    return platform_driver_register(&my_drv);
}
module_init(my_driver_init);

static void __exit my_driver_exit(void) {
    platform_driver_unregister(&my_drv);
}
module_exit(my_driver_exit);
```

بتكتب ببساطة:
```c
module_driver(my_drv, platform_driver_register, platform_driver_unregister);
```

**Parameters:**
- `__driver`: اسم الـ `device_driver` struct (مش pointer).
- `__register`: دالة التسجيل (مثلاً `i2c_add_driver`).
- `__unregister`: دالة الإلغاء.
- `...`: args إضافية تتبعت للدالتين.

**Key details:**
- الـ macro بيمرر `&(__driver)` — يعني بياخد address الـ struct تلقائياً.
- بتستخدم `##__VA_ARGS__` للـ variadic support.
- الـ bus-specific macros زي `module_i2c_driver` و`module_platform_driver` مبنية فوقيها.

---

#### `builtin_driver`

```c
#define builtin_driver(__driver, __register, ...)
```

زي `module_driver` لكن بدون `module_exit` لأن الـ built-in drivers ما بتتحملش. بيستخدم `device_initcall` بدل `module_init`.

**Parameters:**
- `__driver`: الـ driver struct.
- `__register`: دالة التسجيل.
- `...`: args إضافية.

**Key details:**
- للـ drivers اللي compiled مباشرة في الـ kernel بدون `CONFIG_MODULES`.

---

### Group 8: Attribute Macros (sysfs helpers)

---

#### `DRIVER_ATTR_RW` / `DRIVER_ATTR_RO` / `DRIVER_ATTR_WO`

```c
#define DRIVER_ATTR_RW(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_RW(_name)

#define DRIVER_ATTR_RO(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_RO(_name)

#define DRIVER_ATTR_WO(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_WO(_name)
```

بيعملوا `driver_attribute` struct جاهز للاستخدام. الـ naming convention: `driver_attr_<name>`.

**مثال:**
```c
/* تعريف attribute اسمه "version" read-only */
static ssize_t version_show(struct device_driver *drv, char *buf)
{
    return sprintf(buf, "1.0\n");
}
static DRIVER_ATTR_RO(version);

/* في driver_register أو init */
driver_create_file(&my_driver, &driver_attr_version);
```

**Key details:**
- `__ATTR_RW` بيضبط `mode = 0644`، `__ATTR_RO` يضبط `0444`، `__ATTR_WO` يضبط `0200`.
- الـ `show`/`store` callbacks اللي بتعرفها لازم اسمها `<name>_show` و`<name>_store`.

---

### الـ `struct device_driver` — توضيح الـ Callbacks

الـ struct نفسه مش function لكن callbacks بتاعته هي جوهر كل driver:

| Callback | متى يتم استدعاؤه | Return |
|---|---|---|
| `probe(dev)` | لما الـ bus تعمل match ناجح | `0` أو error code |
| `remove(dev)` | لما الـ device يتشال | `void` |
| `shutdown(dev)` | system shutdown/reboot | `void` |
| `suspend(dev, state)` | دخول sleep mode | `0` أو error |
| `resume(dev)` | خروج من sleep | `0` أو error |
| `sync_state(dev)` | بعد ما كل consumers يتربطوا | `void` |
| `coredump(dev)` | كتابة على sysfs coredump entry | `void` |

**الـ `p_cb.post_unbind_rust`:** callback خاص بـ Rust drivers فقط — بيتنفذ بعد `remove()` وبعد تحرير كل الـ devres entries. ما يتستدعيش من C drivers.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ Driver Model

الـ **debugfs** بيكشف معلومات داخلية عن الـ driver core. افتراضياً بيتعمل mount على `/sys/kernel/debug`.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف مجلدات الـ driver core
ls /sys/kernel/debug/driver_model/ 2>/dev/null || echo "not exposed directly"

# الـ deferred probe queue — أهم entry للـ debugging
cat /sys/kernel/debug/devices_deferred

# كل الـ devices اللي لسه مش اتـ probe
cat /sys/kernel/debug/devices_deferred
```

| Debugfs Entry | المعنى | إزاي تقراه |
|---|---|---|
| `/sys/kernel/debug/devices_deferred` | قائمة الـ devices اللي في حالة `EPROBE_DEFER` | `cat` عادي |
| `/sys/kernel/debug/sleep_time` | وقت الـ suspend/resume لكل device | `cat` + تحليل |
| `/sys/kernel/debug/wakeup_sources` | مصادر الـ wakeup النشطة | `cat` |
| `/sys/kernel/debug/gpio` | حالة الـ GPIO (مفيد مع platform drivers) | `cat` |

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ device_driver

الـ **sysfs** هو المرآة المباشرة لـ `struct device_driver` وما بيحتويه.

```bash
# الـ driver يظهر في:
ls /sys/bus/<bus_name>/drivers/<driver_name>/

# مثال حقيقي لـ i2c driver
ls /sys/bus/i2c/drivers/at24/

# الـ bind/unbind (suppressed لو suppress_bind_attrs=true)
echo "0-0050" > /sys/bus/i2c/drivers/at24/unbind
echo "0-0050" > /sys/bus/i2c/drivers/at24/bind

# كل الـ devices المربوطة بالـ driver
ls /sys/bus/i2c/drivers/at24/

# module المرتبط بالـ driver
cat /sys/bus/i2c/drivers/at24/module/refcnt

# probe_type — عادي مش مكشوف مباشرة لكن تعرف عنه من dmesg
cat /sys/bus/platform/drivers/my_driver/uevent

# dev_groups — attributes إضافية بتظهر في /sys/devices/.../<device>/
ls /sys/devices/platform/my_device/

# coredump trigger
echo 1 > /sys/class/<class>/<device>/coredump
```

**مواقع sysfs المهمة:**

| المسار | ما يحتويه |
|---|---|
| `/sys/bus/<bus>/drivers/<drv>/` | الـ driver entry + مسارات الـ devices |
| `/sys/bus/<bus>/drivers/<drv>/bind` | ربط device يدوياً |
| `/sys/bus/<bus>/drivers/<drv>/unbind` | فك ربط device يدوياً |
| `/sys/bus/<bus>/drivers/<drv>/module` | symlink للـ module |
| `/sys/devices/.../driver` | symlink من الـ device للـ driver |
| `/sys/devices/.../driver_override` | override اسم الـ driver |

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ **ftrace** هو الأداة الأقوى لتتبع مسار الـ probe والـ match.

```bash
# تفعيل tracing للـ driver core events
cd /sys/kernel/debug/tracing

# شوف الـ events المتاحة
grep -r "." available_events | grep -E "devres|driver|probe|bus"

# تفعيل أهم events
echo 1 > events/devres/devres_log/enable
echo 1 > events/initcall/initcall_start/enable
echo 1 > events/initcall/initcall_finish/enable

# تتبع كل calls للـ driver probe
echo 'p:probe_entry drivers/base/dd.c:really_probe' >> kprobe_events
echo 1 > events/kprobes/probe_entry/enable

# function tracing للـ driver_register
echo driver_register > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
cat trace

# تتبع مسار really_probe كامل
echo really_probe > set_graph_function
echo function_graph > current_tracer
echo 1 > tracing_on
cat trace | head -100

# تنظيف
echo 0 > tracing_on
echo nop > current_tracer
```

**Events مهمة في driver core:**

| Event | متى يحدث |
|---|---|
| `kprobes/really_probe` | عند بدء تنفيذ الـ probe |
| `kprobes/driver_register` | عند تسجيل driver جديد |
| `kprobes/bus_probe_device` | عند محاولة مطابقة device بـ driver |
| `kprobes/driver_deferred_probe_add` | عند إضافة device لقائمة الـ deferred |

---

#### 4. الـ printk والـ Dynamic Debug

**Dynamic debug** بيخلي تفعّل `dev_dbg()` و `pr_debug()` runtime بدون recompile.

```bash
# تفعيل debug messages لـ driver core كله
echo "file drivers/base/dd.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/bus.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/driver.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ module معين
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وfunction names
echo "module my_driver +pflm" > /sys/kernel/debug/dynamic_debug/control
# p=print, f=function, l=line, m=module

# شوف الـ rules النشطة
cat /sys/kernel/debug/dynamic_debug/control | grep my_driver

# تعيين loglevel عشان تشوف كل الـ messages
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk

# شوف driver probe messages في real-time
dmesg -w | grep -E "probe|driver|bind"
```

```bash
# لو بتكتب driver وعايز تضيف debug مؤقت
# في الكود:
dev_dbg(dev, "probe called, state=%d\n", state);
pr_debug("driver_register: %s on bus %s\n", drv->name, drv->bus->name);
```

---

#### 5. الـ Kernel Config Options للـ Debugging

```bash
# اتحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "DEBUG_DRIVER|DEBUG_KOBJECT|PROBE_EVENTS"
# أو
grep -E "DEBUG_DRIVER|DEBUG_KOBJECT|DEBUG_DEVRES|KPROBES" /boot/config-$(uname -r)
```

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_DRIVER` | يفعّل `dev_dbg()` messages في driver core |
| `CONFIG_DEBUG_KOBJECT` | يفعّل kobject lifecycle debugging |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يكشف use-after-free في kobject release |
| `CONFIG_DEBUG_DEVRES` | يطبع كل managed resource alloc/free |
| `CONFIG_KPROBES` | يتيح kprobe tracing على كل functions |
| `CONFIG_FTRACE` | يفعّل function tracing |
| `CONFIG_DYNAMIC_DEBUG` | يتيح runtime تفعيل `dev_dbg()` |
| `CONFIG_PROVE_LOCKING` | يكشف deadlock محتمل في driver locks |
| `CONFIG_LOCKDEP` | lock dependency graph |
| `CONFIG_DEBUG_SPINLOCK` | يكشف spinlock misuse |
| `CONFIG_ASYNC_CORE` | مطلوب لـ `PROBE_PREFER_ASYNCHRONOUS` |
| `CONFIG_PM_DEBUG` | debug لـ suspend/resume في الـ driver |
| `CONFIG_PM_SLEEP_DEBUG` | تفاصيل أكثر عن sleep transitions |

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# udevadm — مراقبة أحداث الـ uevent (KOBJ_ADD, KOBJ_BIND, إلخ)
udevadm monitor --kernel --udev
# مثال output:
# KERNEL[123.456] add /devices/platform/my_dev (platform)
# UDEV  [123.789] add /devices/platform/my_dev (platform)

# udevadm info — شوف كل attributes للـ device
udevadm info -a -p /sys/devices/platform/my_dev

# لمعرفة الـ driver المرتبط بأي device
udevadm info /sys/bus/i2c/devices/0-0050 | grep DRIVER

# lsmod — شوف الـ modules المحملة وعدد مرات استخدامها
lsmod | grep my_driver

# modinfo — شوف بيانات الـ driver module
modinfo my_driver

# systool — شوف معلومات sysfs بشكل منظم
systool -b i2c -v     # كل الـ devices على i2c bus
systool -d my_device -v  # device معين

# driver_probe_done check (من userspace بشكل غير مباشر)
dmesg | grep "Freeing init memory"  # يظهر بعد كل probing انتهى
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `probe of <dev> failed with error -ENODEV` | الـ driver ما قدرش يتعرف على الـ hardware | تحقق من `of_match_table` أو `id_table` |
| `probe of <dev> failed with error -EPROBE_DEFER` | dependency غير جاهزة (clock, regulator, إلخ) | انتظر — الـ kernel هيعيد المحاولة تلقائي |
| `Driver 'foo' is already registered` | محاولة تسجيل نفس الـ driver مرتين | تحقق من `module_init` ومن الـ `driver_register` |
| `<dev>: deferred probe pending` | الـ device في قائمة الانتظار ولم يُـprobe بعد | `cat /sys/kernel/debug/devices_deferred` |
| `bind_store: driver_bind(<dev>) failed` | فشل ربط يدوي من sysfs | شوف الـ probe return value في dmesg |
| `kobject: can't set name: nomem` | نفاذ الـ memory عند إنشاء kobject name | تحقق من memory pressure |
| `sysfs: cannot create duplicate filename` | محاولة إنشاء نفس الـ sysfs entry مرتين | تحقق من الـ driver cleanup في remove() |
| `driver_override set to <drv>` | تم تغيير الـ driver يدوياً | informational فقط |
| `Refusing to bind to driver <drv>` | `suppress_bind_attrs = true` | لا تستخدم bind/unbind من sysfs |
| `really_probe: probe of <dev> rejects match -<N>` | الـ probe رجع error غير EPROBE_DEFER | شوف الـ error code وتصحح الـ probe() |
| `Calling deferred probe for <dev>` | إعادة محاولة probe بعد ما dependency جهزت | informational |

---

#### 8. أماكن استراتيجية لـ dump_stack() وWARN_ON()

```c
int my_driver_probe(struct device *dev)
{
    struct my_priv *priv;

    /* تحقق من الـ state قبل كل حاجة */
    WARN_ON(!dev->driver);  /* لازم يتعمل set قبل probe */

    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* تحقق من الـ parent bus */
    WARN_ON_ONCE(!dev->bus);

    /* لو حصل state غير متوقع، طبع الـ stack */
    if (some_unexpected_condition) {
        dev_err(dev, "unexpected state at probe\n");
        dump_stack();  /* يطبع call trace كامل */
        return -EINVAL;
    }

    return 0;
}

void my_driver_remove(struct device *dev)
{
    /* تحقق إن الـ resources اتحررت صح */
    WARN_ON(atomic_read(&my_ref_count) != 0);
}

/* في driver_register wrapper */
int __must_check driver_register(struct device_driver *drv)
{
    /* WARN لو بتسجل بدون bus */
    if (WARN_ON(!drv->bus))
        return -EINVAL;
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# تحقق إن الـ device موجود فعلاً على الـ bus
# لـ PCI:
lspci -vvv | grep -A 20 "My Device"

# تحقق إن الـ kernel شاف نفس الـ device
ls /sys/bus/pci/devices/

# لـ I2C — فحص الـ device موجود على الـ bus
i2cdetect -y 1   # bus number 1

# لـ platform devices — تحقق من الـ Device Tree node
ls /sys/firmware/devicetree/base/

# مقارنة الـ driver state بالـ hardware registers مباشرة
devmem2 0xFE200000 w   # قراءة register على عنوان معين

# تحقق من الـ device bound للـ driver الصح
readlink /sys/devices/platform/my_device/driver
# Output المتوقع: ../../../../bus/platform/drivers/my_driver
```

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 — قراءة/كتابة physical memory address
# تثبيت: apt install devmem2
devmem2 0xFE200000 w        # قراءة word (32-bit)
devmem2 0xFE200004 b        # قراءة byte
devmem2 0xFE200008 w 0x1    # كتابة قيمة

# /dev/mem — قراءة مباشرة من userspace (يحتاج CONFIG_STRICT_DEVMEM=n)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE200000/4)) 2>/dev/null | xxd

# io utility (من package ioport)
io -4 0xFE200000    # قراءة 32-bit I/O port

# ioremap من داخل kernel module مؤقت للـ debugging
```

```c
/* داخل kernel — dump registers بشكل منظم */
static void my_driver_dump_regs(struct my_priv *priv)
{
    dev_info(priv->dev, "=== Register Dump ===\n");
    dev_info(priv->dev, "CTRL:   0x%08x\n", readl(priv->base + CTRL_REG));
    dev_info(priv->dev, "STATUS: 0x%08x\n", readl(priv->base + STATUS_REG));
    dev_info(priv->dev, "IRQ_EN: 0x%08x\n", readl(priv->base + IRQ_EN_REG));
}

/* استدعيها في probe لو فشل */
if (ret) {
    my_driver_dump_regs(priv);
    return ret;
}
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

```
I2C Bus Analysis:
─────────────────
SCL ──┐  ┌──┐  ┌──┐  ┌──
      └──┘  └──┘  └──┘
SDA ─START─[ADDR 7bit]─[R/W]─[ACK]─[DATA]─STOP─

نقاط القياس:
- بين master وأول slave: تحقق من الـ pull-up resistors (عادة 4.7kΩ)
- سرعة SCL: 100kHz (Standard), 400kHz (Fast), 1MHz (Fast+)
- الـ probe يُشغّل بعد ما الـ kernel يبعث START condition

SPI Bus Analysis:
─────────────────
SCLK ─┐ ┌─┐ ┌─┐ ┌─
      └─┘ └─┘ └─
MOSI ══[D7][D6][D5]...[D0]══
CS   ─┐                    ┌─
      └────────────────────┘

نقاط القياس:
- CS يخش low قبل أول clock edge
- CPOL/CPHA يتحدد في driver setup
```

```bash
# للتحقق إن الـ driver enable الـ clock قبل الـ probe:
# تقيس الـ clock output على الـ oscilloscope
# لو مش شايل signal = clock لم يتفعّل = probe هيفشل بـ EPROBE_DEFER

# لـ GPIO debugging:
gpioget gpiochip0 5    # قراءة GPIO pin 5
gpioset gpiochip0 5=1  # set GPIO
gpiomon gpiochip0 5    # مراقبة تغييرات
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ dmesg | التشخيص |
|---|---|---|
| الـ device مش موجود فعلياً | `probe of xyz failed with error -ENODEV` | `i2cdetect`/`lspci` لتأكيد وجوده |
| خطأ في عنوان الـ device | `No ACK from device` أو probe -ENXIO | تحقق من address في DT/ACPI |
| clock غير مفعّل | `clk_enable failed` أو probe -EPROBE_DEFER | تحقق من CCF config |
| regulator غير جاهز | `regulator_enable failed` | تحقق من supply في DT |
| IRQ conflict | `request_irq failed -EBUSY` | `cat /proc/interrupts` |
| power supply خاطئ | حالات undefined behavior في الـ probe | قيس الـ voltage على pin الـ VCC |
| reset line مش controlled | register values غير منطقية | تحقق من GPIO reset في DT |
| DMA alignment issue | `DMA mapping failed` | تحقق من `dma_set_mask` في probe |

---

#### 5. الـ Device Tree Debugging

```bash
# تحقق من الـ DT node موجود وصح
ls /sys/firmware/devicetree/base/

# شوف كل properties لـ node معين
ls /sys/firmware/devicetree/base/soc/i2c@fe804000/
cat /sys/firmware/devicetree/base/soc/i2c@fe804000/compatible | xxd

# fdtdump — dump الـ DTB كامل
fdtdump /boot/dtbs/my_board.dtb | grep -A 10 "my_device"

# dtc — decompile الـ DTB
dtc -I dtb -O dts /boot/dtbs/my_board.dtb > /tmp/my_board.dts
grep -A 20 "my_device" /tmp/my_board.dts

# تحقق إن الـ compatible string في DT يطابق of_match_table في الـ driver
# في الـ driver:
# static const struct of_device_id my_of_match[] = {
#     { .compatible = "vendor,my-device" },
#     {}
# };

# في الـ DT:
# my_device@0 {
#     compatible = "vendor,my-device";  <-- لازم يطابق بالظبط
# };

# تحقق من الـ status property
cat /sys/firmware/devicetree/base/soc/my_device/status
# لازم تكون "okay" مش "disabled"

# overlay debugging — لو بتستخدم DT overlays
dtoverlay -l    # list loaded overlays
dtoverlay my_overlay  # apply overlay
dmesg | tail -20      # شوف نتيجة الـ probe
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ لكل تقنية

```bash
#!/bin/bash
# ===== Driver Debug Script =====

DRIVER_NAME="my_driver"
BUS="platform"

echo "=== 1. Driver Registration Status ==="
ls /sys/bus/${BUS}/drivers/${DRIVER_NAME}/ 2>/dev/null || \
    echo "Driver NOT registered on ${BUS} bus"

echo ""
echo "=== 2. Bound Devices ==="
for dev in /sys/bus/${BUS}/drivers/${DRIVER_NAME}/*/; do
    [ -L "$dev/driver" ] && echo "  Device: $(basename $dev)"
done

echo ""
echo "=== 3. Deferred Probe Queue ==="
cat /sys/kernel/debug/devices_deferred 2>/dev/null | \
    grep -i "${DRIVER_NAME}" || echo "No deferred devices for this driver"

echo ""
echo "=== 4. Recent Driver Events ==="
dmesg --since "5 minutes ago" | grep -iE "${DRIVER_NAME}|probe|bind" | tail -30

echo ""
echo "=== 5. Module Info ==="
modinfo ${DRIVER_NAME} 2>/dev/null | grep -E "filename|version|depends"

echo ""
echo "=== 6. Active Dynamic Debug Rules ==="
grep "${DRIVER_NAME}" /sys/kernel/debug/dynamic_debug/control 2>/dev/null || \
    echo "No dynamic debug rules active"
```

---

```bash
# تفعيل full driver debug session في خطوات:

# خطوة 1: رفع الـ log level
dmesg -n 7

# خطوة 2: تفعيل dynamic debug لكل drivers/base
echo "file drivers/base/* +p" > /sys/kernel/debug/dynamic_debug/control

# خطوة 3: تفعيل ftrace على really_probe
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function_graph > current_tracer
echo really_probe > set_graph_function
echo 1 > tracing_on

# خطوة 4: trigger الـ probe (مثلاً bind يدوي)
echo "my_device" > /sys/bus/platform/drivers/my_driver/bind

# خطوة 5: اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace | head -50
echo 0 > tracing_on
```

---

#### مثال Output وتفسيره

```
# output من: cat /sys/kernel/debug/devices_deferred
my_i2c_device@0050: wait for supplier /soc/i2c@fe804000

# التفسير:
# my_i2c_device@0050 = الـ device name
# wait for supplier = بينتظر dependency
# /soc/i2c@fe804000 = الـ i2c controller اللي لسه ماتـ probe-ش
```

```
# output من ftrace على really_probe:
 0)               |  really_probe() {
 0)               |    dev_driver_string() {
 0)   0.250 us    |      dev_bus_name();
 0)   0.583 us    |    }
 0)               |    my_driver_probe() {   /* <-- الـ probe الفعلي */
 0) + 15.200 us   |      devm_kzalloc();
 0) ! 123.450 us  |      my_hw_init();       /* هنا وقت طويل = مشكلة */
 0)   0.100 us    |    }
 0) ! 145.000 us  |  }

# التفسير:
# ! = وقت تجاوز حد معين (طبيعي أو بطيء)
# my_hw_init أخد 123us = محتاج تحقيق
```

```
# output من udevadm monitor أثناء probe:
KERNEL[1234.567890] add      /devices/platform/my_device (platform)
KERNEL[1234.568100] bind     /devices/platform/my_device (platform)
UDEV  [1234.600000] add      /devices/platform/my_device (platform)
UDEV  [1234.610000] bind     /devices/platform/my_device (platform)

# التفسير:
# add → الـ device اتضاف للـ sysfs (kobject_uevent KOBJ_ADD)
# bind → اتربط بـ driver بنجاح (KOBJ_BIND)
# ترتيب KERNEL ثم UDEV طبيعي (kernel يرسل أولاً، udev يعالج بعدين)
```

```bash
# فحص سريع: هل الـ driver_register اشتغل صح؟
dmesg | grep -E "driver_register|registered driver"

# فحص: هل الـ suppress_bind_attrs فعّال؟
ls /sys/bus/platform/drivers/my_driver/ | grep -E "bind|unbind"
# لو مفيش bind/unbind = suppress_bind_attrs = true

# فحص: الـ probe type
dmesg | grep -i "async\|asynchronous\|deferred" | grep my_driver
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART Driver مش بيـprobe على RK3562 Industrial Gateway

#### العنوان
**الـ `of_match_table` فيها compatible string غلط — الـ driver مش بيتـbind**

#### السياق
شركة بتبني industrial gateway على RK3562 لـmonitoring معدات مصنع. الـboard عندها RS485 port بتشتغل عبر UART3. المهندس كتب custom platform driver للـUART وعمل device tree node، بس الـdevice مش بتشتغل خالص.

#### المشكلة
```bash
$ dmesg | grep uart
# لا في output خالص — الـdriver مش بيظهر
$ ls /sys/bus/platform/drivers/my_uart_drv/
# فارغ — مفيش device bound
```

الـprobe function مش اتنادت. الـkonsole مش بتطلع error.

#### التحليل
الـ`struct device_driver` فيها:
```c
const struct of_device_id *of_match_table;
```

الـkernel بيعمل match بين الـnode في DT والـ`of_match_table` الموجودة في الـ`device_driver`. لو الـ`compatible` string مش متطابقة بالظبط — مفيش match، ومفيش call لـ`probe`.

```c
/* الـdriver registration */
static const struct of_device_id my_uart_of_ids[] = {
    { .compatible = "rockchip,rk3562-uart-custom" },  /* غلط */
    {}
};

static struct platform_driver my_uart_driver = {
    .driver = {
        .name           = "my_uart_drv",
        .of_match_table = my_uart_of_ids,
        /* probe_type = PROBE_DEFAULT_STRATEGY (default) */
    },
    .probe  = my_uart_probe,
    .remove = my_uart_remove,
};
```

الـDT node:
```dts
/* غلط — compatible string مش متطابقة */
&uart3 {
    compatible = "rockchip,rk3562-uart";  /* مختلفة! */
    status = "okay";
};
```

الـkernel بيمر على كل device في الـplatform bus، بينادي `bus_type->match` اللي بيقارن الـ`of_device_id` بالـ`compatible` property في DT. لو مفيش match → مفيش call لـ`device_driver->probe`.

#### الحل
**Step 1:** تطابق الـcompatible strings:
```dts
/* DT fix */
&uart3 {
    compatible = "rockchip,rk3562-uart-custom";
    status = "okay";
};
```

**Step 2:** تأكد بـdebug:
```bash
# شوف كل الـof_match_table entries للـdriver
$ cat /sys/bus/platform/drivers/my_uart_drv/uevent

# فحص الـdevice
$ cat /sys/bus/platform/devices/fe670000.uart/uevent | grep OF_COMPATIBLE
OF_COMPATIBLE_0=rockchip,rk3562-uart-custom
```

**Step 3:** لو محتاج تتأكد إن الـdriver اتسجل:
```bash
$ ls /sys/bus/platform/drivers/my_uart_drv/
fe670000.uart  bind  uevent  unbind
```

#### الدرس المستفاد
**الـ`of_match_table`** في `struct device_driver` هي الوحيدة اللي الـkernel بيستخدمها للـOF matching — أي حرف زيادة أو ناقص في الـ`compatible` string يعني إن الـ`probe` مش هتتنادى خالص.

---

### السيناريو 2: SPI Sensor بيـcrash عند الـboot على STM32MP1 IoT Device

#### العنوان
**`PROBE_FORCE_SYNCHRONOUS` مع dependency على clock driver لسه مش جاهز — kernel panic**

#### السياق
IoT sensor node على STM32MP1 بيستخدم SPI accelerometer (BMI088). المهندس عمل custom SPI driver وحدد `probe_type = PROBE_FORCE_SYNCHRONOUS` عشان يضمن إن الـdriver يتـinit قبل الـapplication layer. بس عند الـboot الـkernel بيـpanic.

#### المشكلة
```
[    2.341] spi-bmi088: probe of spi0.0 failed with error -ENODEV
[    2.342] Kernel panic - not syncing: Fatal exception
```

أو في حالات تانية:
```
[    2.341] bmi088_spi: probe of spi0.0 failed with error -EPROBE_DEFER
[    2.342] bmi088_spi: probe called again (deferred)
```

الـdevice بيـdefer للأبد لأن الـclock supplier مش موجود.

#### التحليل
```c
enum probe_type {
    PROBE_DEFAULT_STRATEGY,
    PROBE_PREFER_ASYNCHRONOUS,
    PROBE_FORCE_SYNCHRONOUS,  /* <-- ده المستخدم */
};
```

من الـheader:
> `PROBE_FORCE_SYNCHRONOUS`: probe routines to run synchronously with driver and device registration (with the exception of -EPROBE_DEFER handling - re-probing always ends up being done asynchronously).

المشكلة إن المهندس فهم غلط: `PROBE_FORCE_SYNCHRONOUS` بيخلي الـprobe يشتغل synchronously مع الـregistration، بس الـ`-EPROBE_DEFER` لو رجعت من الـclock subsystem → الـre-probe هيبقى async وممكن يتأخر أو يفشل لو الـsystem مش setup صح.

```c
static struct spi_driver bmi088_driver = {
    .driver = {
        .name           = "bmi088_spi",
        .of_match_table = bmi088_of_ids,
        .probe_type     = PROBE_FORCE_SYNCHRONOUS, /* مش مناسب هنا */
    },
    .probe  = bmi088_probe,
    .remove = bmi088_remove,
};
```

#### الحل
**Option A:** غير الـ`probe_type` لـ`PROBE_DEFAULT_STRATEGY` وخلي الـ`-EPROBE_DEFER` يشتغل طبيعي:

```c
static struct spi_driver bmi088_driver = {
    .driver = {
        .name           = "bmi088_spi",
        .of_match_table = bmi088_of_ids,
        .probe_type     = PROBE_DEFAULT_STRATEGY, /* الأنسب */
    },
    .probe  = bmi088_probe,
    .remove = bmi088_remove,
};
```

**Option B:** لو محتاج synchronous فعلاً، اتأكد إن كل الـsuppliers (clocks, regulators) بيتـregister قبل الـSPI devices:
```bash
# فحص deferred probe queue
$ cat /sys/kernel/debug/devices_deferred
bmi088_spi: (reason: clk not ready)
```

**DT fix** لضمان الـclock dependency:
```dts
&spi1 {
    bmi088@0 {
        compatible = "bosch,bmi088-accel";
        reg = <0>;
        clocks = <&rcc SPI1_K>;  /* explicit dependency */
    };
};
```

#### الدرس المستفاد
**الـ`probe_type`** في `struct device_driver` مش بتحل مشاكل الـdependencies — دي وظيفة الـ`-EPROBE_DEFER`. استخدم `PROBE_FORCE_SYNCHRONOUS` بس لو عارف إن كل dependencies جاهزة في وقت الـregistration.

---

### السيناريو 3: I2C HDMI Bridge مش بيـunbind صح على i.MX8 Android TV Box

#### العنوان
**`suppress_bind_attrs = true` بيمنع الـhotplug script من الـrebind — HDMI مش بتشتغل بعد الـreconnect**

#### السياق
Android TV box على i.MX8MP بيستخدم IT6663 HDMI bridge via I2C. المهندس حدد `suppress_bind_attrs = true` في الـdriver "للأمان". لما الـHDMI cable بيتفصل وترجع، الـbridge مش بترجع.

#### المشكلة
```bash
# الـuser بيشتكي: HDMI مش بترجع بعد فصل الكابل
$ ls /sys/bus/i2c/drivers/it6663/
# لا bind ولا unbind files!
$ echo "2-0048" > /sys/bus/i2c/drivers/it6663/bind
bash: /sys/bus/i2c/drivers/it6663/bind: Permission denied
```

#### التحليل
من `struct device_driver`:
```c
bool suppress_bind_attrs; /* disables bind/unbind via sysfs */
```

لما `suppress_bind_attrs = true`، الـdriver core مش بيخلق الـ`bind` و`unbind` sysfs entries. الـhotplug script بتاع Android بيحاول يعمل manual rebind عبر:
```bash
echo "2-0048" > /sys/bus/i2c/drivers/it6663/unbind
echo "2-0048" > /sys/bus/i2c/drivers/it6663/bind
```
بس العملية دي مش شغالة خالص.

```c
/* الكود المسبب للمشكلة */
static struct i2c_driver it6663_driver = {
    .driver = {
        .name                = "it6663",
        .of_match_table      = it6663_of_ids,
        .suppress_bind_attrs = true,  /* <-- ده بيكسر الـhotplug */
    },
    .probe  = it6663_probe,
    .remove = it6663_remove,
};
```

#### الحل
**Step 1:** شيل `suppress_bind_attrs` أو حطه `false`:

```c
static struct i2c_driver it6663_driver = {
    .driver = {
        .name                = "it6663",
        .of_match_table      = it6663_of_ids,
        .suppress_bind_attrs = false,  /* default — اسمح بالـsysfs bind/unbind */
    },
    .probe  = it6663_probe,
    .remove = it6663_remove,
};
```

**Step 2:** تأكد إن الـremove function بتنظف الـstate صح:
```c
static int it6663_remove(struct i2c_client *client)
{
    struct it6663_priv *priv = i2c_get_clientdata(client);
    /* reset bridge state */
    it6663_bridge_disable(priv);
    return 0;
}
```

**Step 3:** الـhotplug script بعد الـfix:
```bash
DEVICE="2-0048"
DRIVER="it6663"
echo "$DEVICE" > /sys/bus/i2c/drivers/$DRIVER/unbind
sleep 0.5
echo "$DEVICE" > /sys/bus/i2c/drivers/$DRIVER/bind
```

#### الدرس المستفاد
**الـ`suppress_bind_attrs`** في `struct device_driver` بيمنع أي تدخل يدوي من الـuserspace. استخدمه بس لو الـdriver عنده state machine معقدة مش تقدر تعمل لها rebind آمن — وإلا خليه `false`.

---

### السيناريو 4: USB Driver بيـleak Memory عند الـremove على AM62x Automotive ECU

#### العنوان
**الـ`remove` callback مش بتتنادى بالترتيب الصح — memory leak في الـdevres**

#### السياق
Automotive ECU على TI AM62x بيستخدم USB-to-CAN adapter (PEAK PCAN-USB). الـECU بيعمل safe shutdown عند emergency stop. بعد كذا cycle من plug/unplug، الـkernel بيـlog:
```
[  234.1] usb 1-1: USB disconnect, device number 3
[  234.2] pcan_usb: remove called
[  234.3] BUG: memory leak detected in pcan_usb
```

#### المشكلة
الـdriver بيـallocate resources في الـprobe بس مش بينظفها صح في الـremove. المهندس بيحاول يفهم الـsequence بتاعة الـdriver model.

#### التحليل
من `struct device_driver`:
```c
int (*probe)  (struct device *dev);  /* allocation هنا */
int (*remove) (struct device *dev);  /* cleanup لازم يبقى هنا */
```

وفيه كمان الـ`p_cb` الجديد:
```c
struct {
    void (*post_unbind_rust)(struct device *dev);  /* Rust only */
} p_cb;
```

الـsequence بتاعة الـremove في الـdriver model:
```
USB disconnect
    → usb_disconnect()
        → device_driver->remove()    ← بتتنادى هنا
        → devres_release_all()       ← بعدها الـdevres بتتحرر
        → p_cb.post_unbind_rust()    ← Rust cleanup (لو موجود)
```

المشكلة إن المهندس بيعمل `kfree` لـpointer اتخزن في `devres` — فالـmemory بتتحرر مرتين أو ما بتتحررش خالص:

```c
static int pcan_probe(struct device *dev)
{
    struct pcan_priv *priv;

    /* خطأ: استخدم devm_kzalloc بدل kzalloc */
    priv = kzalloc(sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    dev_set_drvdata(dev, priv);
    /* ... */
    return 0;
}

static int pcan_remove(struct device *dev)
{
    /* المهندس نسي الـkfree! */
    return 0;  /* memory leak */
}
```

#### الحل
**Option A:** استخدم `devm_kzalloc` — الـdevres هتتحرر تلقائي بعد الـremove:

```c
static int pcan_probe(struct device *dev)
{
    struct pcan_priv *priv;

    /* devm_ prefix = auto-cleanup after remove() */
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    dev_set_drvdata(dev, priv);
    return 0;
}

/* remove يبقى فاضي أو بس للـnon-devres resources */
static int pcan_remove(struct device *dev)
{
    return 0; /* devm handles cleanup */
}
```

**Option B:** لو لازم manual management:
```c
static int pcan_remove(struct device *dev)
{
    struct pcan_priv *priv = dev_get_drvdata(dev);
    kfree(priv);  /* explicit free */
    return 0;
}
```

**Debug:**
```bash
# فحص الـmemory leak
$ cat /sys/kernel/debug/kmemleak
unreferenced object 0xffff888012340000 (size 512):
  comm "kworker/u4:2", pid 89
  backtrace:
    pcan_probe+0x4c
    platform_probe+0x40
```

#### الدرس المستفاد
الـ`remove` callback في `struct device_driver` بيتنادى قبل تحرير الـdevres — لو استخدمت `devm_` APIs في الـ`probe`، الـcleanup بيحصل تلقائي. الـ`post_unbind_rust` في `p_cb` بيتنادى بعد الـdevres release وده للـRust drivers بس.

---

### السيناريو 5: Custom Clock Driver بيـhang الـboot على Allwinner H616 Media Box

#### العنوان
**`driver_probe_done()` وـ`wait_for_device_probe()` — الـinit script بيستهلك الـclock قبل ما الـdriver يـprobe**

#### السياق
Media streaming box على Allwinner H616 بيستخدم custom HDMI audio clock driver. الـinit system (systemd) بيحاول يفتح `/dev/snd/controlC0` في أوائل الـboot قبل ما الـclock driver يـprobe. النتيجة: الـaudio مش شغال والـboot بيتأخر.

#### المشكلة
```
[    3.21] systemd[1]: Starting audio service...
[    3.22] alsa: cannot open /dev/snd/controlC0: No such device
[    3.23] systemd[1]: audio.service: Start request repeated too quickly
[    3.24] systemd[1]: audio.service: Failed
```

الـclock driver نفسه بيـprobe بعد كده:
```
[    4.10] sun50i-h616-clk: probed successfully
[    4.11] sun8i-codec: probed (depends on clk)
```

#### التحليل
من الـheader:
```c
bool __init driver_probe_done(void);
void wait_for_device_probe(void);
void __init wait_for_init_devices_probe(void);
```

الـ`driver_probe_done()` بيرجع `true` لما الـdeferred probe list فاضية — يعني كل الـdevices اللي اتأجلت بـ`-EPROBE_DEFER` اتـprobe أو فشلت. الـ`wait_for_device_probe()` بتـblock لحد ما الـprobe threads تخلص.

الـclock driver معين `PROBE_PREFER_ASYNCHRONOUS`:
```c
static struct platform_driver h616_ccu_driver = {
    .driver = {
        .name           = "sun50i-h616-clk",
        .of_match_table = h616_ccu_dt_ids,
        .probe_type     = PROBE_PREFER_ASYNCHRONOUS, /* بيـprobe متأخر */
    },
    .probe = h616_ccu_probe,
};
```

والـaudio codec بيعمل `clk_get` في الـprobe — لو الـclock driver مش جاهز → `-EPROBE_DEFER` → الـaudio codec بيتأجل. بس لو الـuserspace بدأ قبل ما الـdeferred probes تخلص → مشكلة.

#### الحل
**Option A:** في الـinit system، استخدم `wait_for_device_probe` equivalent — اللي هو الانتظار على الـsysfs:

```bash
# في الـsystemd service file
[Unit]
Description=Audio Service
After=sound.target
ConditionPathExists=/dev/snd/controlC0

[Service]
ExecStartPre=/bin/sh -c 'while [ ! -e /dev/snd/controlC0 ]; do sleep 0.1; done'
ExecStart=/usr/bin/audio-daemon
```

**Option B:** غير الـ`probe_type` للـclock driver:
```c
static struct platform_driver h616_ccu_driver = {
    .driver = {
        .name           = "sun50i-h616-clk",
        .of_match_table = h616_ccu_dt_ids,
        .probe_type     = PROBE_FORCE_SYNCHRONOUS, /* اتأكد إنه يـprobe مبكر */
    },
    .probe = h616_ccu_probe,
};
```

**Option C:** في الـkernel init code، بعد تسجيل الـdrivers الـcritical، استخدم:
```c
/* في الـplatform init code */
static int __init h616_audio_init(void)
{
    /* سجل الـclock driver الأول */
    platform_driver_register(&h616_ccu_driver);

    /* انتظر لحد ما الـprobe يخلص */
    wait_for_device_probe();

    /* دلوقتي سجل الـaudio driver */
    platform_driver_register(&h616_audio_driver);
    return 0;
}
```

**Debug — فحص الـdeferred probe queue:**
```bash
$ cat /sys/kernel/debug/devices_deferred
sun8i-codec: waiting for sun50i-h616-clk (clock dependency)

# فحص متى الـdriver_probe_done يبقى true
$ dmesg | grep "Freeing init memory"
# ده بيحصل بعد wait_for_init_devices_probe()
```

**Driver lookup للـdiagnostics:**
```c
/* في الـdebug code — تحقق إن الـclock driver موجود */
struct device_driver *drv;
drv = driver_find("sun50i-h616-clk", &platform_bus_type);
if (!drv)
    pr_warn("clock driver not registered yet!\n");
```

#### الدرس المستفاد
الـ`PROBE_PREFER_ASYNCHRONOUS` في `struct device_driver` بيسرع الـboot بس بيكسر الـdependencies الضمنية. الـ`wait_for_device_probe()` و`driver_probe_done()` موجودين عشان تـsynchronize الـinit stages — استخدمهم لما عندك dependency chain بين drivers والـuserspace.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/driver-api/driver-model/driver.rst`](https://docs.kernel.org/driver-api/driver-model/driver.html) | الصفحة الرسمية لـ `struct device_driver` — تشرح الـ probe/remove/suspend/resume callbacks |
| [`Documentation/driver-api/driver-model/index.rst`](https://docs.kernel.org/driver-api/driver-model/index.html) | الفهرس الكامل لـ driver model documentation |
| [`Documentation/driver-api/infrastructure.html`](https://docs.kernel.org/driver-api/infrastructure.html) | شرح البنية التحتية للـ driver core بالكامل |
| [`Documentation/driver-api/driver-model/devres.rst`](https://static.lwn.net/kerneldoc/driver-api/driver-model/devres.html) | الـ managed device resources (devres) — مرتبط بعمل الـ driver مع الـ lifetime |

---

### مقالات LWN.net الأساسية

#### الـ Driver Model والـ kobject

- [**Driver porting: Device model overview**](https://lwn.net/Articles/31185/) — مقدمة عملية للـ driver model في 2.6، تشرح كيف يرتبط `struct device_driver` بالـ bus وكيف يحدث الـ matching.
- [**A fresh look at the kernel's device model**](https://lwn.net/Articles/645810/) — مراجعة حديثة للـ device model، تناقش مشاكل التصميم الحالية والمقترحات لتحسينها.
- [**The zen of kobjects**](https://lwn.net/Articles/51437/) — الأساس النظري للـ `kobject` اللي يستند عليه كل الـ driver model.
- [**kobjects and sysfs**](https://lwn.net/Articles/54651/) — كيف يظهر الـ `device_driver` في الـ `/sys/bus/*/drivers/` عبر الـ kobject/sysfs integration.
- [**Examining a kobject hierarchy**](https://lwn.net/Articles/55847/) — مثال عملي على تتبع الـ kobject hierarchy في `/sys`.
- [**kobjects and hotplug events**](https://lwn.net/Articles/52621/) — العلاقة بين الـ kobject وأحداث الـ hotplug/uevent.
- [**Porting device drivers to the 2.6 kernel**](https://lwn.net/Articles/driver-porting/) — سلسلة مقالات تغطي الانتقال من 2.4 إلى 2.6 وتشمل الـ driver model بالكامل.
- [**Driver porting: Devices and attributes**](https://lwn.net/Articles/31220/) — كيف تستخدم `DRIVER_ATTR_RO/RW/WO` وتعرض attributes في sysfs.
- [**Avoiding sysfs surprises**](https://lwn.net/Articles/36850/) — مشاكل الـ reference counting مع الـ kobject في sysfs.

#### الـ Probe Mechanism والـ Deferred Probe

- [**Deferred driver probing**](https://lwn.net/Articles/450460/) — شرح آلية `EPROBE_DEFER` وكيف يؤجل الـ driver core إعادة الـ probe لما تتوفر الموارد.
- [**drivercore: Add driver probe deferral mechanism**](https://lwn.net/Articles/485194/) — الـ patch الأصلي اللي أضاف الـ deferred probe للـ kernel.
- [**Device dependencies and deferred probing**](https://lwn.net/Articles/662820/) — مناقشة أعمق لمشكلة الـ device dependencies وكيف يحلها الـ deferred probe.
- [**Asynchronous device/driver probing support**](https://lwn.net/Articles/629895/) — شرح `PROBE_PREFER_ASYNCHRONOUS` وتأثيره على سرعة الـ boot.
- [**driver-core: async probe support**](https://lwn.net/Articles/614939/) — الـ patch الأولي لإضافة الـ async probe للـ driver core.
- [**Multi-threaded device probing**](https://lwn.net/Articles/192851/) — المرحلة الأولى من محاولات الـ multi-threaded probing في الـ kernel.

---

### مصادر Kernel Newbies

- [**Drivers — Linux Kernel Newbies**](https://kernelnewbies.org/Drivers) — نقطة بداية لفهم كتابة الـ drivers، تشمل links لأمثلة وشروح.
- [**Linux_3.4_DriverArch**](https://kernelnewbies.org/Linux_3.4_DriverArch) — تغييرات الـ driver architecture في kernel 3.4.
- [**Linux_4.0-DriversArch**](https://kernelnewbies.org/Linux_4.0-DriversArch) — تغييرات الـ driver architecture في kernel 4.0.
- [**Linux_3.19-DriversArch**](https://kernelnewbies.org/Linux_3.19-DriversArch) — تغييرات الـ driver architecture في kernel 3.19.
- [**EPROBE_DEFER mailing list discussion**](https://kernelnewbies.kernelnewbies.narkive.com/wq803lme/eprobe-defer-and-how-it-is-supposed-to-work) — نقاش على الـ mailing list يشرح كيف يشتغل `EPROBE_DEFER` بشكل صحيح.

---

### مصادر eLinux.org

- [**Linux Device Drivers — eLinux.org**](https://elinux.org/Linux_Device_Drivers) — مرجع شامل لـ LDD3 وموارد تعلم الـ driver model.
- [**Device Drivers Presentations — eLinux.org**](https://elinux.org/Device_Drivers_Presentations) — عروض تقديمية من مؤتمرات Linux عن الـ drivers.
- [**Device Tree Linux — eLinux.org**](https://elinux.org/Linux_Drivers_Device_Tree_Guide) — الربط بين `of_match_table` في `struct device_driver` والـ Device Tree.
- [**EBC Exercise 26 Device Drivers — eLinux.org**](https://elinux.org/EBC_Exercise_26_Device_Drivers) — أمثلة عملية لكتابة kernel modules بسيطة.

---

### نقاشات Mailing List

- [**PATCH: drivercore — Add driver probe deferral mechanism**](https://linux-kernel.vger.kernel.narkive.com/NprB4Quy/patch-drivercore-add-driver-probe-deferral-mechanism) — الـ patch الأصلي على الـ LKML لإضافة الـ deferred probe.
- [**Device probing — Linus Torvalds**](https://yarchive.net/comp/linux/device_probing.html) — رأي Linus التاريخي في كيفية عمل الـ device probing.
- [**How to troubleshoot deferred probe issues**](https://blog.dowhile0.org/2022/06/21/how-to-troubleshoot-deferred-probe-issues-in-linux/) — دليل عملي لتتبع مشاكل الـ `EPROBE_DEFER` في الـ production.

---

### Kernel Git — Commits مهمة

| الـ Commit | الوصف |
|-----------|-------|
| [`drivers/base/driver.c` history](https://github.com/torvalds/linux/commits/master/drivers/base/driver.c) | تاريخ التعديلات على الـ driver core |
| [`include/linux/device/driver.h` history](https://github.com/torvalds/linux/commits/master/include/linux/device/driver.h) | تاريخ التعديلات على الـ header الرئيسي |
| [Greg KH driver-core tree](https://kernel.googlesource.com/pub/scm/linux/kernel/git/gregkh/driver-core/) | شجرة Greg Kroah-Hartman الخاصة بالـ driver core development |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الكتاب**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المباشر**: Chapter 14 — *The Linux Device Model* ([PDF مباشر](https://static.lwn.net/images/pdf/LDD3/ch14.pdf))
- **الفصل الداعم**: Chapter 2 — *Building and Running Modules* (يغطي `module_init`/`module_exit` والـ `module_driver` macro)
- **متاح مجاناً**: [LWN.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل**: Chapter 17 — *Devices and Modules*
- يشرح الـ `device_driver` registration flow وعلاقته بالـ kobject layer
- يغطي الـ sysfs وكيف يعكس الـ driver model على الـ filesystem

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل**: Chapter 8 — *Device Driver Basics*
- تركيز على الـ `probe`/`remove` في سياق الـ embedded systems والـ platform drivers
- يشرح الـ `of_match_table` والـ Device Tree matching من منظور embedded

#### The Linux Kernel Module Programming Guide
- [متاح مجاناً](https://sysprog21.github.io/lkmpg/) — يغطي كتابة الـ modules من الصفر مع أمثلة على الـ `module_driver` macro

---

### توثيق الـ Kernel Labs

- [**Linux Device Model — kernel labs**](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html) — تمارين عملية على الـ driver model مع شرح خطوة بخطوة للـ `device_driver` registration

---

### مصطلحات البحث

لإيجاد معلومات أكتر عن الموضوع ده، استخدم الـ search terms دي:

```
linux kernel "struct device_driver" probe remove
linux driver model "driver_register" "driver_unregister"
linux "EPROBE_DEFER" deferred probe mechanism
linux kernel "probe_type" PROBE_PREFER_ASYNCHRONOUS boot speed
linux "sync_state" callback device supply chain
linux "driver_find_device" iterator pattern
linux "module_driver" macro boilerplate
linux "DRIVER_ATTR_RW" sysfs driver attribute
linux kobject sysfs driver binding unbind
linux "suppress_bind_attrs" driver security
```
## Phase 8: Writing simple module

### الفكرة

الـ `driver_register()` هي من أهم الـ exported functions في الـ driver model — كل driver بيتسجل في الـ kernel عن طريقها. هنعمل **kprobe** عليها عشان نشوف اسم كل driver بيتسجل في الـ system بشكل real-time.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_driver_register.c
 * Hooks driver_register() to log every driver being registered in the system.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/device/driver.h> /* struct device_driver */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc-Arabic");
MODULE_DESCRIPTION("kprobe on driver_register() to log driver names");

/*
 * pre_handler — called right BEFORE driver_register() executes.
 * pt_regs carries the CPU registers at the probe point.
 * On x86-64, first argument (struct device_driver *drv) is in rdi.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve the first argument: pointer to device_driver */
    struct device_driver *drv = (struct device_driver *)regs->di;

    /* Safety: never deref a NULL pointer inside a probe */
    if (!drv || !drv->name)
        return 0;

    pr_info("kprobe_drv: driver_register() called — driver='%s' bus='%s'\n",
            drv->name,
            (drv->bus && drv->bus->name) ? drv->bus->name : "unknown");

    return 0; /* returning 0 means: let driver_register() proceed normally */
}

/* Define the kprobe — symbol_name points to the function we want to hook */
static struct kprobe kp = {
    .symbol_name = "driver_register",
    .pre_handler = handler_pre,
};

static int __init kprobe_drv_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_drv: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("kprobe_drv: planted kprobe at %p (driver_register)\n", kp.addr);
    return 0;
}

static void __exit kprobe_drv_exit(void)
{
    /* MUST unregister before module is removed — otherwise the probe
     * callback points to freed memory and the kernel will crash */
    unregister_kprobe(&kp);
    pr_info("kprobe_drv: kprobe removed\n");
}

module_init(kprobe_drv_init);
module_exit(kprobe_drv_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية زي `module_init` و `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | الـ `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/device/driver.h` | الـ `struct device_driver` عشان نقدر نقرأ الـ `name` و `bus` |

---

#### الـ `handler_pre` — قلب الـ module

الـ kprobe بيوقف التنفيذ قبل دخول `driver_register()` ويديك الـ `pt_regs` — وهي snapshot للـ CPU registers في اللحظة دي. على الـ x86-64، الـ ABI بيحط أول argument في الـ register `rdi`، فبناخد منه الـ `device_driver *` مباشرةً.

التحقق من `NULL` ضروري لأن الـ probe بتشتغل في سياق interrupt-like وأي crash هيوقع الـ kernel كله.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "driver_register",
    .pre_handler = handler_pre,
};
```

الـ `symbol_name` بيخلي الـ kernel يحوّل الاسم لـ address تلقائياً عن طريق الـ kallsyms، فمش محتاجين نحدد الـ address يدوياً.

---

#### الـ `module_init` و `module_exit`

الـ `register_kprobe` بتزرع الـ breakpoint في الـ kernel memory. لو رجعت قيمة سالبة يبقى فيه مشكلة (زي إن الـ CONFIG_KPROBES مش مفعّل أو الـ function محمية بـ `__kprobes`).

الـ `unregister_kprobe` في الـ exit ضرورية جداً — من غيرها لو الـ module اتشال من الـ memory والـ probe لسه موجودة، أول حد يعمل `driver_register()` هيـ jump لـ address مش موجودة وهيحصل kernel panic.

---

### طريقة التجربة

```bash
# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod kprobe_driver_register.ko

# Watch output (شوف أي driver بيتسجل)
sudo dmesg -w | grep kprobe_drv

# Trigger some driver registration (مثلاً)
sudo modprobe usbcore

# Unload
sudo rmmod kprobe_driver_register
```

**مثال على الـ output:**

```
kprobe_drv: planted kprobe at ffffffffc0123456 (driver_register)
kprobe_drv: driver_register() called — driver='usbcore' bus='unknown'
kprobe_drv: driver_register() called — driver='usb' bus='usb'
kprobe_drv: kprobe removed
```

---

### Makefile

```makefile
obj-m += kprobe_driver_register.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
