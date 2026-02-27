## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ `include/linux/device.h` جزء من **Driver Core, Kobjects, Debugfs and Sysfs** subsystem — المُعرَّف في `MAINTAINERS` تحت القسم بالاسم ده. المسؤول عنه Greg Kroah-Hartman وRafael J. Wysocki. كل الكود بتاعه في `drivers/base/` و`include/linux/device/`.

---

### القصة — المشكلة اللي بيحلها

تخيل إن عندك مصنع فيه آلاف الماكينات — USB، PCI، I2C، GPIO، sensors، disks، network cards. كل ماكينة بتحتاج حد يشغّلها (driver)، وكل driver بيحتاج يعرف مع إيه بيتكلم.

من غير نظام مركزي، كل driver هيعمل كود خاص بيه لـ:
- اكتشاف الجهاز
- ربطه بالـ driver المناسب
- إدارة الـ power management
- تسجيله في `/sys/`
- إدارة الـ DMA
- التعامل مع الـ hotplug

ده هيخلي الـ kernel فوضى — كل driver بيعيد اختراع العجلة. **الـ Driver Model** جه يحل المشكلة دي بتوحيد التمثيل.

---

### الـ Big Picture — ELI5

**الـ `struct device`** هي الـ "بطاقة هوية" لكل جهاز في النظام. زي ما كل إنسان عنده بطاقة قومية فيها اسمه، عنوانه، رقمه، وعلاقاته — كل device في Linux عنده `struct device` فيها كل حاجة الـ kernel محتاجها عشان يتعامل معاه.

```
                         ┌─────────────────────────────┐
                         │         struct device        │
                         │  ┌────────────────────────┐  │
                         │  │ kobj  (اسم + sysfs)    │  │
                         │  │ parent (الجهاز الأب)   │  │
                         │  │ bus   (نوع البص)        │  │
                         │  │ driver (الـ driver)     │  │
                         │  │ power (إدارة الطاقة)   │  │
                         │  │ links (supplier/consumer│  │
                         │  │ dma_*  (إعدادات DMA)   │  │
                         │  │ of_node (device tree)  │  │
                         │  └────────────────────────┘  │
                         └─────────────────────────────┘
```

الـ file ده مش بس header — هو **العقد** اللي بيحكم العلاقة بين كل الأجزاء في الـ kernel.

---

### ليه مهم؟ — الهدف

الـ `device.h` بيحقق **Linux Unified Device Model** — نموذج موحد يخلي:

| المشكلة | الحل |
|--------|------|
| كل driver بيسجّل نفسه بطريقة مختلفة | `device_register()` واحدة لكل الأجهزة |
| Hotplug فوضى | `uevent` موحد عبر sysfs |
| Power management مش منسق | `dev_pm_info` مركزي في كل device |
| DMA setup متكرر | `dma_mask`, `dma_ops` موحدين |
| Dependency بين أجهزة مش مضبوط | `device_link` لإدارة supplier/consumer |

---

### المكونات الرئيسية في الـ File

#### 1. `struct device` — قلب الكل

```c
struct device {
    struct kobject kobj;         // الاسم + التمثيل في sysfs
    struct device *parent;       // الجهاز الأب (مثلاً: USB hub)
    const struct bus_type *bus;  // نوع البص (USB, PCI, I2C...)
    struct device_driver *driver; // الـ driver المرتبط
    void *platform_data;         // بيانات خاصة بالـ board
    void *driver_data;           // بيانات خاصة بالـ driver
    struct dev_links_info links; // علاقات supplier/consumer
    struct dev_pm_info power;    // إدارة الطاقة
    u64 *dma_mask;               // حدود الـ DMA
    struct device_node *of_node; // Device Tree node
    struct fwnode_handle *fwnode;// Firmware node (ACPI/DT)
    dev_t devt;                  // رقم الجهاز في /dev/
    const struct class *class;   // الـ class (مثلاً: block, net)
    void (*release)(struct device *dev); // دالة التحرير
};
```

#### 2. `struct device_link` — إدارة التبعيات

لو عندك GPU محتاج الـ PCIe power domain يشتغل قبله — ده بالظبط `device_link`. الـ supplier مش هيخلّي consumer يبدأ قبل ما هو يكون جاهز.

```c
struct device_link {
    struct device *supplier;    // الجهاز المُوَرِّد
    struct device *consumer;    // الجهاز المُستهلِك
    enum device_link_state status; // حالة الرابط
    u32 flags;                  // DL_FLAG_PM_RUNTIME etc.
};
```

#### 3. `struct device_attribute` + Macros — الـ sysfs interface

```c
// بيعمل ملف /sys/devices/.../temperature
DEVICE_ATTR_RO(temperature);

// تنفيذه:
static ssize_t temperature_show(struct device *dev,
                                 struct device_attribute *attr,
                                 char *buf) {
    return sprintf(buf, "%d\n", read_temp(dev));
}
```

#### 4. `struct device_type` — تنويع داخل الـ bus

الـ USB bus ممكن يحمل `hub` و`keyboard` و`storage` — كلهم devices على نفس البص بس بأنواع مختلفة. `device_type` بتحدد السلوك الخاص لكل نوع.

#### 5. `struct subsys_interface` — واجهات إضافية

بتخلي كود إضافي يتعلق بـ subsystem كامل من غير ما يكون driver حصري على device. مثلاً: thermal monitoring لكل devices في subsystem.

---

### دورة حياة الجهاز — القصة كاملة

```
اكتشاف الجهاز (hardware/firmware)
          │
          ▼
    device_initialize()    ← تهيئة struct device
          │
          ▼
    device_add()           ← تسجيل في sysfs + إرسال uevent
          │
          ▼
    bus->match()           ← البحث عن driver مناسب
          │
          ▼
    driver->probe()        ← الـ driver يأخذ السيطرة
          │
          ▼
    [الجهاز شغّال]
          │
          ▼
    device_del()           ← إزالة من sysfs
          │
          ▼
    put_device()           ← تحرير المرجع → release()
```

---

### الـ Device Links — مثال عملي

تخيل laptop فيه:
- **PCIe controller** (supplier)
- **NVMe SSD** (consumer)
- **GPU** (consumer)

الـ kernel لازم يضمن إن الـ PCIe controller جاهز قبل ما أي consumer يبدأ. `device_link_add()` بتضمن ده:

```c
// في driver الـ NVMe
device_link_add(nvme_dev, pcie_dev, DL_FLAG_PM_RUNTIME | DL_FLAG_AUTOREMOVE_CONSUMER);
```

لو الـ PCIe نام (runtime suspend) — الـ NVMe هينام تلقائياً قبله.

---

### الملفات المرتبطة

#### الـ Core Implementation
| الملف | الدور |
|------|------|
| `drivers/base/core.c` | تنفيذ `device_register/add/del` و device links |
| `drivers/base/bus.c` | تسجيل الـ buses وربط devices بـ drivers |
| `drivers/base/driver.c` | تسجيل الـ drivers |
| `drivers/base/dd.c` | منطق probe/attach/detach |
| `drivers/base/class.c` | إدارة device classes |
| `drivers/base/devres.c` | الـ managed resources (devm_*) |
| `drivers/base/power/main.c` | Power management للـ devices |

#### الـ Headers الفرعية
| الملف | الدور |
|------|------|
| `include/linux/device/bus.h` | `struct bus_type` |
| `include/linux/device/driver.h` | `struct device_driver` |
| `include/linux/device/class.h` | `struct class` |
| `include/linux/device/devres.h` | devm_* resource management |
| `include/linux/kobject.h` | الـ base object (sysfs integration) |
| `include/linux/pm.h` | Power management types |
| `include/linux/fwnode.h` | Firmware node (ACPI/DT) |

#### الـ Sysfs/Kobject Layer
| الملف | الدور |
|------|------|
| `fs/sysfs/` | تنفيذ الـ sysfs filesystem |
| `lib/kobject.c` | تنفيذ الـ kobject |

---

### الملفات اللي القارئ المبتدئ يبدأ بيها

1. `Documentation/driver-api/driver-model/overview.rst` — نظرة عامة
2. `drivers/base/core.c` — القلب الحقيقي للتنفيذ
3. `include/linux/device/bus.h` — فهم كيف buses بتشتغل
4. `include/linux/device/driver.h` — فهم كيف drivers بتتسجّل
5. أي driver بسيط زي `drivers/leds/led-core.c` — يشوف كيف بيستخدم الـ API
## Phase 2: شرح الـ Linux Device Model Framework

### المشكلة — ليه الـ Framework ده اتعمل أصلاً؟

قبل الـ Linux 2.6، كل bus subsystem (PCI, USB, I2C, ...) كان بيعمل الـ device management بتاعه هو بشكل منفصل تماماً — مفيش abstraction مشتركة. النتيجة كانت:

- **كود متكرر** في كل subsystem: كل واحد بيعمل matching بين device وdriver على طريقته.
- **power management** مش موحد — كل bus بيعمل suspend/resume بطريقة مختلفة.
- **sysfs** مش موجود — مفيش طريقة uniform للـ userspace يشوف الـ devices.
- **hotplug** (زي USB plug/unplug) محتاج معرفة كل bus على حدة.
- **resource cleanup** غير موثوق — memory leaks عند الـ driver unbind.

الـ kernel محتاج نموذج موحد يقول: "كل device في النظام له تمثيل واحد، وكل driver له lifecycle موحد."

---

### الحل — الـ Linux Device Model

الـ kernel اتبنى فيه **Driver Model** مركزي جوه `drivers/base/` بيوفر:

| ما بيقدمه | كيف |
|-----------|-----|
| Unified object model | `struct kobject` كـ base لكل object |
| Hierarchical topology | `struct device` مع `parent` pointer |
| Driver-Device matching | `struct bus_type` بيدير الـ match loop |
| sysfs representation | كل device وdriver ليه directory في `/sys/` |
| Reference counting | `get_device()` / `put_device()` |
| Resource management | devres (managed resources) |
| Power management hooks | `struct dev_pm_ops` |
| Device links/dependencies | `struct device_link` |

---

### الـ Analogy الحقيقية — مطار دولي

تخيل **مطار كبير** فيه:

```
المطار = الـ Kernel
═══════════════════════════════════════════════════════

 Terminal A (PCI Bus)      Terminal B (USB Bus)
 ┌──────────────────┐      ┌──────────────────┐
 │  Gate 1 (GPU)    │      │  Gate 5 (Webcam) │
 │  Gate 2 (NIC)    │      │  Gate 6 (Mouse)  │
 └──────────────────┘      └──────────────────┘

 موظف الاستقبال = struct bus_type (.match callback)
 الراكب = struct device
 الطيار = struct device_driver
 بطاقة الصعود = of_match_table / PCI device ID
 بوابة الصعود = probe()
 مغادرة الطائرة = remove()
 إدارة المطار = driver core (drivers/base/)
```

**التفاصيل الكاملة للـ analogy:**

| عنصر في المطار | عنصر في الـ Kernel | التفسير |
|----------------|---------------------|---------|
| بطاقة هوية الراكب | `dev->of_node` أو `pci_device_id` | معرف الـ device في الـ firmware |
| Terminal/Concourse | `struct bus_type` | يجمع devices من نفس النوع |
| نظام مطابقة الراكب بالرحلة | `bus_type->match()` | يقارن device بكل driver مسجل |
| الطيار يقبل الرحلة | `driver->probe()` returns 0 | الـ driver يقول "أنا قادر أشتغل مع الـ device ده" |
| مكتب الـ check-in المركزي | `device_register()` | بيضيف الـ device لكل الـ lists والـ sysfs |
| سجلات المطار | `/sys/bus/`, `/sys/devices/` | كل راكب وطيار ليه entry |
| مدير المطار | `struct device_private *p` | الـ private data بتاعة الـ driver core |
| رحلة الإياب | `driver->remove()` | cleanup عند فصل الـ device |
| طوارئ | `device_shutdown()` | system shutdown path |

---

### الـ Big Picture Architecture

```
User Space
    │  read/write /sys/bus/i2c/devices/0-0050/
    ▼
╔══════════════════════════════════════════════════════════╗
║                    sysfs (Virtual FS)                    ║
║  /sys/devices/  /sys/bus/  /sys/class/  /sys/drivers/   ║
╚══════════════════════════════════════════════════════════╝
    │ kernfs_node (per kobject)
    ▼
╔══════════════════════════════════════════════════════════╗
║              Kobject / Kset Layer                        ║
║  struct kobject  ◄── reference count, name, sysfs node  ║
║  struct kset     ◄── group of related kobjects           ║
╚══════════════════════════════════════════════════════════╝
    │  (embedded in every device/driver/bus)
    ▼
╔══════════════════════════════════════════════════════════╗
║              Driver Core  (drivers/base/)                ║
║                                                          ║
║  ┌─────────────┐   match()   ┌──────────────────┐       ║
║  │ struct      │ ──────────► │ struct           │       ║
║  │ device      │             │ device_driver    │       ║
║  │             │ ◄────────── │                  │       ║
║  │ .bus ──────►│   probe()   │ .bus ────────────►bus    ║
║  └─────────────┘             └──────────────────┘       ║
║         │                           │                   ║
║         └──── struct bus_type ──────┘                   ║
║                  .match()                                ║
║                  .probe()                                ║
║                  .remove()                               ║
╚══════════════════════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════════════════════╗
║         Bus-Specific Subsystems                          ║
║  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  ║
║  │ PCI bus  │ │ USB bus  │ │ I2C bus  │ │ Platform  │  ║
║  │ pci_bus  │ │ usb_bus  │ │ i2c_bus  │ │ bus       │  ║
║  └──────────┘ └──────────┘ └──────────┘ └───────────┘  ║
╚══════════════════════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════════════════════╗
║         Hardware / Device Tree / ACPI                    ║
╚══════════════════════════════════════════════════════════╝
```

---

### الـ Core Abstraction — `struct device`

الـ **`struct device`** هي الـ central idea في الـ framework. كل حاجة في الـ kernel اتعملت device بتمثل instance واحدة منها.

```c
struct device {
    struct kobject kobj;          /* sysfs + refcount foundation */
    struct device        *parent; /* tree hierarchy */
    struct device_private *p;     /* driver core private — don't touch */

    const char           *init_name;   /* name before kobject init */
    const struct device_type *type;    /* e.g., "disk", "partition" */

    const struct bus_type   *bus;      /* which bus: pci_bus, i2c_bus... */
    struct device_driver    *driver;   /* bound driver, NULL if unbound */

    void *platform_data;  /* board-specific config (BSP data) */
    void *driver_data;    /* driver-private, via dev_set_drvdata() */

    struct mutex mutex;   /* serialize calls to driver callbacks */

    struct dev_links_info links;    /* supplier/consumer dependencies */
    struct dev_pm_info    power;    /* runtime & system PM state */
    struct dev_pm_domain *pm_domain;/* PM domain grouping */

    /* DMA capabilities */
    u64          *dma_mask;
    u64           coherent_dma_mask;
    u64           bus_dma_limit;
    struct device_dma_parameters *dma_parms;

    struct device_node   *of_node;  /* Device Tree node */
    struct fwnode_handle *fwnode;   /* generic firmware node (DT or ACPI) */

    dev_t    devt;           /* major:minor → creates /sys/dev entry */
    u32      id;             /* instance number */

    spinlock_t        devres_lock;
    struct list_head  devres_head; /* managed resources list */

    const struct class         *class;   /* e.g., "block", "net", "input" */
    const struct attribute_group **groups;

    void (*release)(struct device *dev); /* MUST be set — called on last put */
    struct iommu_group *iommu_group;
};
```

#### كيف تتشابك الـ structs مع بعض

```
struct device
├── struct kobject kobj
│   ├── struct kref kref          ← reference count
│   ├── struct kernfs_node *sd    ← sysfs directory
│   └── struct kset *kset         ← group (e.g., all devices on bus)
│
├── struct device *parent         ← tree: /sys/devices/platform/i2c0/0-0050
│
├── const struct bus_type *bus ──► struct bus_type
│                                  ├── .name = "i2c"
│                                  ├── .match()    ← compare device to driver
│                                  ├── .probe()    ← call driver->probe()
│                                  └── .remove()
│
├── struct device_driver *driver ► struct device_driver
│                                  ├── .name
│                                  ├── .probe()
│                                  ├── .remove()
│                                  ├── .of_match_table[]
│                                  └── const struct dev_pm_ops *pm
│
├── struct dev_pm_info power      ← runtime PM state machine
├── struct dev_links_info links   ← supplier/consumer graph
│   ├── list_head suppliers
│   └── list_head consumers
│
├── struct device_node *of_node   ← DT node (of subsystem — Open Firmware)
└── void (*release)(dev)          ← destructor
```

---

### الـ Kobject — الأساس اللي كل حاجة بتبني عليه

**الـ kobject subsystem** هو layer قاعدي منفصل (مش جزء من الـ device model نفسه) — بيقدم:

1. **Reference counting** عبر `struct kref` — الـ object بيتحذف لما الـ count يوصل صفر
2. **sysfs representation** — كل kobject ليه directory في `/sys/`
3. **uevent notification** — يبعت events لـ udev عند add/remove/change
4. **Hierarchy** — كل kobject عنده `parent` pointer

```c
struct kobject {
    const char       *name;       /* directory name in sysfs */
    struct kobject   *parent;     /* parent dir in sysfs */
    struct kset      *kset;       /* containing group */
    struct kref       kref;       /* reference count */
    struct kernfs_node *sd;       /* actual sysfs entry */
    unsigned int state_in_sysfs:1;
    unsigned int uevent_suppress:1;
};
```

الـ `struct device` بـ**يضم** الـ kobject (مش بيورثه — composition مش inheritance):

```c
struct device {
    struct kobject kobj;   /* FIRST member → container_of works */
    ...
};

/* للوصول من kobject لـ device */
#define kobj_to_dev(__kobj) \
    container_of_const(__kobj, struct device, kobj)
```

---

### الـ bus_type — قلب الـ Matching

```c
struct bus_type {
    const char *name;   /* "pci", "usb", "i2c", "platform" */

    /* Core callbacks */
    int  (*match)(struct device *dev, const struct device_driver *drv);
    int  (*probe)(struct device *dev);   /* wraps driver->probe() */
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);
    void (*sync_state)(struct device *dev);

    /* Power management */
    int  (*suspend)(struct device *dev, pm_message_t state);
    int  (*resume)(struct device *dev);
    const struct dev_pm_ops *pm;

    /* DMA */
    int  (*dma_configure)(struct device *dev);
    void (*dma_cleanup)(struct device *dev);
};
```

**Flow الـ matching:**

```
device_register(dev)
        │
        ▼
bus->match(dev, drv)   ← لكل driver مسجل على نفس الـ bus
        │
   match returns 1
        │
        ▼
bus->probe(dev)
   └─► driver->probe(dev)
              │
         returns 0
              │
              ▼
         dev->driver = drv   ← الـ binding اتم
```

---

### الـ device_driver — العقد اللي الـ Driver بيلتزم بيه

```c
struct device_driver {
    const char            *name;
    const struct bus_type *bus;   /* يجب يتطابق مع bus الـ device */

    /* Matching tables */
    const struct of_device_id   *of_match_table;   /* Device Tree */
    const struct acpi_device_id *acpi_match_table;  /* ACPI */

    /* Lifecycle */
    int  (*probe)(struct device *dev);     /* required */
    int  (*remove)(struct device *dev);    /* cleanup */
    void (*shutdown)(struct device *dev);
    void (*sync_state)(struct device *dev); /* after all consumers bound */

    /* Power */
    int  (*suspend)(struct device *dev, pm_message_t state);
    int  (*resume)(struct device *dev);
    const struct dev_pm_ops *pm;

    enum probe_type probe_type; /* sync vs async probing */
    struct driver_private *p;   /* internal driver core data */
};
```

---

### الـ device_link — إدارة الـ Dependencies

ده feature مهم جداً في embedded. لو عندك **clock driver** (supplier) و**SPI controller** (consumer) — الـ SPI controller مش المفروض يتعمله probe قبل الـ clock driver يكون جاهز.

```c
struct device_link {
    struct device *supplier;      /* e.g., clock device */
    struct device *consumer;      /* e.g., SPI controller */
    enum device_link_state status;
    u32 flags;                    /* DL_FLAG_PM_RUNTIME, etc. */
    struct list_head s_node;      /* في supplier->links.consumers */
    struct list_head c_node;      /* في consumer->links.suppliers */
};
```

**States:**

```
DL_STATE_NONE
    │
    ▼
DL_STATE_DORMANT ──────── لا driver على الطرفين
    │
    ▼ (supplier driver probed)
DL_STATE_AVAILABLE ─────── supplier جاهز، consumer لسه
    │
    ▼ (consumer starts probing)
DL_STATE_CONSUMER_PROBE
    │
    ▼ (consumer probe() success)
DL_STATE_ACTIVE ─────────── كلاهم شغالين
    │
    ▼ (supplier starts unbinding)
DL_STATE_SUPPLIER_UNBIND
```

**مثال واقعي:**

```c
/* في probe() بتاع SPI controller */
struct device_link *link = device_link_add(
    &spi_ctrl->dev,     /* consumer */
    &clk_dev->dev,      /* supplier */
    DL_FLAG_PM_RUNTIME | DL_FLAG_AUTOREMOVE_CONSUMER
);
```

---

### الـ devres — Managed Resources

**الـ devres subsystem** (اللي بيتعمله include من `device/devres.h`) بيحل مشكلة الـ resource leaks عند الـ driver unbind. كل resource اتعمله allocate بـ `devm_*` بيتحرر automatically لما الـ device يتفصل.

```
dev->devres_head  ← linked list of struct devres
    ├── [devm_kmalloc allocated buffer]
    ├── [devm_ioremap mapped region]
    ├── [devm_clk_get acquired clock]
    └── [devm_gpio_request_one GPIO]

عند device_del() أو probe() failure:
    الـ core بيمشي على الـ list بالعكس وبيحرر كل حاجة
```

---

### الـ sysfs Attributes

الـ **`struct device_attribute`** بيمثل ملف في `/sys/devices/.../`:

```c
struct device_attribute {
    struct attribute attr;          /* name + mode (permissions) */
    ssize_t (*show)(struct device *dev,
                    struct device_attribute *attr,
                    char *buf);    /* cat /sys/.../foo */
    ssize_t (*store)(struct device *dev,
                     struct device_attribute *attr,
                     const char *buf, size_t count); /* echo > /sys/.../foo */
};

/* مثال حقيقي */
static ssize_t voltage_show(struct device *dev,
                             struct device_attribute *attr, char *buf)
{
    struct my_pmic *pmic = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", pmic->current_voltage_mv);
}
DEVICE_ATTR_RO(voltage);  /* ينشئ dev_attr_voltage */
```

```
/sys/devices/platform/my_pmic/
    ├── voltage        ← dev_attr_voltage.show()
    ├── driver/        ← symlink to driver
    ├── power/
    │   ├── runtime_status
    │   └── control
    └── uevent
```

---

### ما الـ Driver Core بيمتلكه vs ما بيفوضه

| المسؤولية | Driver Core يمتلكه | يفوضه للـ Driver/Bus |
|-----------|-------------------|----------------------|
| Object lifecycle (kref) | نعم — `get_device`/`put_device` | لا |
| sysfs directory creation | نعم — عند `device_add()` | لا |
| Driver-Device matching loop | نعم — عند register أي منهما | الـ bus يقدم `match()` |
| `probe()` invocation | نعم — الـ core بيستدعيه | الـ driver ينفذه |
| devres cleanup | نعم — automatic | الـ driver يستخدم `devm_*` |
| DMA mask setup | الـ bus يعملها في `dma_configure()` | الـ driver يضبط الـ mask |
| Interrupt handling | لا — مش جزء من الـ device model | الـ driver بيعمل `request_irq()` |
| Protocol communication (SPI/I2C/...) | لا | الـ bus subsystem |
| Hardware-specific register access | لا | الـ driver حصراً |
| PM runtime decisions | Core يوفر infrastructure | الـ driver يقرر متى `pm_runtime_get/put` |
| uevent generation | نعم — عند add/remove/bind | الـ bus يضيف env vars عبر `uevent()` |

---

### مثال كامل — من Registration للـ Probe

```c
/* 1. تعريف الـ driver */
static const struct of_device_id my_sensor_of_match[] = {
    { .compatible = "vendor,my-temp-sensor" },
    { }
};

static struct i2c_driver my_sensor_driver = {
    .driver = {
        .name = "my-temp-sensor",
        .of_match_table = my_sensor_of_match,
        .pm = &my_sensor_pm_ops,
    },
    .probe  = my_sensor_probe,
    .remove = my_sensor_remove,
};

/* 2. Registration */
module_i2c_driver(my_sensor_driver);
/*
 * expands to:
 *   module_init → i2c_add_driver(&my_sensor_driver)
 *   module_exit → i2c_del_driver(&my_sensor_driver)
 *
 * i2c_add_driver() → driver_register()
 *   → bus_for_each_dev(i2c_bus, match_fn)
 *     → i2c_bus.match(dev, &my_sensor_driver.driver)
 *       → of_driver_match_device() checks of_match_table
 *         → match found → device_driver_attach()
 *           → my_sensor_probe(dev) called
 */

/* 3. probe() */
static int my_sensor_probe(struct i2c_client *client)
{
    struct my_sensor *sensor;

    /* devm_kzalloc: auto-freed on remove() */
    sensor = devm_kzalloc(&client->dev, sizeof(*sensor), GFP_KERNEL);
    if (!sensor)
        return -ENOMEM;

    /* store private data in device */
    dev_set_drvdata(&client->dev, sensor);
    sensor->client = client;

    /* devm_regmap_init: auto-cleaned on remove() */
    sensor->regmap = devm_regmap_init_i2c(client, &my_regmap_cfg);
    if (IS_ERR(sensor->regmap))
        return PTR_ERR(sensor->regmap);

    return 0; /* 0 = probe success = binding complete */
}
```

عند `my_sensor_probe()` يرجع 0:
```
dev->driver = &my_sensor_driver.driver   ← binding
kobject_uevent(KOBJ_BIND)                ← udev يعمل /dev node لو محتاج
/sys/bus/i2c/drivers/my-temp-sensor/    ← symlink يتضاف
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### `enum device_link_state` — حالة الـ link بين supplier وconsumer

| القيمة | المعنى |
|---|---|
| `DL_STATE_NONE` (-1) | مش بيتتبع presence الـ drivers |
| `DL_STATE_DORMANT` (0) | لا supplier ولا consumer موجودين |
| `DL_STATE_AVAILABLE` | الـ supplier موجود، الـ consumer لأ |
| `DL_STATE_CONSUMER_PROBE` | الـ consumer بيعمل probe دلوقتي |
| `DL_STATE_ACTIVE` | الاتنين موجودين ومشتغلين |
| `DL_STATE_SUPPLIER_UNBIND` | الـ supplier بيعمل unbind |

#### `DL_FLAG_*` — Device Link Flags

| الـ Flag | الـ Bit | المعنى |
|---|---|---|
| `DL_FLAG_STATELESS` | BIT(0) | الـ core مش هيشيل الـ link أوتوماتيك |
| `DL_FLAG_AUTOREMOVE_CONSUMER` | BIT(1) | شيل الـ link لما الـ consumer يعمل unbind |
| `DL_FLAG_PM_RUNTIME` | BIT(2) | الـ runtime PM هيستخدم الـ link ده |
| `DL_FLAG_RPM_ACTIVE` | BIT(3) | run `pm_runtime_get_sync()` على الـ supplier وقت الإنشاء |
| `DL_FLAG_AUTOREMOVE_SUPPLIER` | BIT(4) | شيل الـ link لما الـ supplier يعمل unbind |
| `DL_FLAG_AUTOPROBE_CONSUMER` | BIT(5) | probe الـ consumer أوتوماتيك بعد ما الـ supplier يتبند |
| `DL_FLAG_MANAGED` | BIT(6) | الـ core بيتتبع الـ drivers (داخلي) |
| `DL_FLAG_SYNC_STATE_ONLY` | BIT(7) | الـ link بيأثر على `sync_state()` بس |
| `DL_FLAG_INFERRED` | BIT(8) | مستنتج من firmware مش من action الـ driver |
| `DL_FLAG_CYCLE` | BIT(9) | الـ link جزء من دورة dependency |

#### `enum dl_dev_state` — حالة الـ driver على الـ device

| القيمة | المعنى |
|---|---|
| `DL_DEV_NO_DRIVER` | مفيش driver |
| `DL_DEV_PROBING` | driver بيعمل probe |
| `DL_DEV_DRIVER_BOUND` | الـ driver اتبند |
| `DL_DEV_UNBINDING` | الـ driver بيعمل unbind |

#### `enum device_removable` — قابلية إزالة الـ device

| القيمة | المعنى |
|---|---|
| `DEVICE_REMOVABLE_NOT_SUPPORTED` | الـ attribute مش مدعوم (default) |
| `DEVICE_REMOVABLE_UNKNOWN` | مش عارف |
| `DEVICE_FIXED` | مش ممكن يتشال |
| `DEVICE_REMOVABLE` | ممكن يتشال من المستخدم |

#### `enum probe_type` — نوع الـ probe للـ driver

| القيمة | المعنى |
|---|---|
| `PROBE_DEFAULT_STRATEGY` | sync أو async — سواء |
| `PROBE_PREFER_ASYNCHRONOUS` | للـ devices البطيئة، افضل async لتسريع الـ boot |
| `PROBE_FORCE_SYNCHRONOUS` | لازم sync مع التسجيل |

#### `enum bus_notifier_event` — أحداث الـ bus notifier

| الحدث | المعنى |
|---|---|
| `BUS_NOTIFY_ADD_DEVICE` | device اتضاف |
| `BUS_NOTIFY_DEL_DEVICE` | device هيتشال |
| `BUS_NOTIFY_REMOVED_DEVICE` | اتشال بنجاح |
| `BUS_NOTIFY_BIND_DRIVER` | driver على وشك يتبند |
| `BUS_NOTIFY_BOUND_DRIVER` | اتبند بنجاح |
| `BUS_NOTIFY_UNBIND_DRIVER` | على وشك يعمل unbind |
| `BUS_NOTIFY_UNBOUND_DRIVER` | عمل unbind بنجاح |
| `BUS_NOTIFY_DRIVER_NOT_BOUND` | فشل الـ bind |

#### `enum device_physical_location_panel` — موقع الـ connection point

| القيمة | المعنى |
|---|---|
| `DEVICE_PANEL_TOP/BOTTOM/LEFT/RIGHT/FRONT/BACK` | الوجه اللي فيه الـ port |
| `DEVICE_PANEL_UNKNOWN` | مجهول |

---

### الـ Structs المهمة

#### 1. `struct device` — الـ building block الأساسي

ده القلب اللي كل حاجة تانية بتتبنى فوقيه. كل device في النظام ممثلة بـ instance منه.

| الـ Field | النوع | الغرض |
|---|---|---|
| `kobj` | `struct kobject` | الـ reference counting والـ sysfs entry |
| `parent` | `struct device *` | الـ parent device (الـ bus controller عادةً) |
| `p` | `struct device_private *` | البيانات الخاصة بالـ driver core (محجوبة) |
| `init_name` | `const char *` | الاسم المبدئي قبل ما الـ kobject يكون جاهز |
| `type` | `const struct device_type *` | نوع الـ device (disk، partition، إلخ) |
| `bus` | `const struct bus_type *` | الـ bus اللي عليه |
| `driver` | `struct device_driver *` | الـ driver المتبند |
| `platform_data` | `void *` | بيانات خاصة بالـ board (embedded/SoC) |
| `driver_data` | `void *` | بيانات الـ driver، set/get بـ `dev_set/get_drvdata()` |
| `mutex` | `struct mutex` | لحماية الـ calls للـ driver |
| `links` | `struct dev_links_info` | قوائم الـ suppliers والـ consumers |
| `power` | `struct dev_pm_info` | كل بيانات الـ power management |
| `pm_domain` | `struct dev_pm_domain *` | PM callbacks للـ subsystem |
| `em_pd` | `struct em_perf_domain *` | الـ energy model (لو `CONFIG_ENERGY_MODEL`) |
| `pins` | `struct dev_pin_info *` | الـ pinctrl info (لو `CONFIG_PINCTRL`) |
| `msi` | `struct dev_msi_info` | بيانات الـ MSI interrupts |
| `dma_ops` | `const struct dma_map_ops *` | عمليات الـ DMA mapping |
| `dma_mask` | `u64 *` | الـ DMA mask (pointer لأنه shared مع الـ driver) |
| `coherent_dma_mask` | `u64` | الـ mask لـ coherent allocations |
| `bus_dma_limit` | `u64` | حد الـ DMA من الـ upstream bridge |
| `dma_parms` | `struct device_dma_parameters *` | قيود الـ segments للـ IOMMU |
| `dma_pools` | `struct list_head` | قائمة الـ DMA pools |
| `of_node` | `struct device_node *` | نود الـ device tree |
| `fwnode` | `struct fwnode_handle *` | الـ firmware node (ACPI أو DT) |
| `devt` | `dev_t` | الـ major/minor لـ sysfs `dev` |
| `id` | `u32` | instance number |
| `devres_lock` | `spinlock_t` | يحمي قائمة الـ devres resources |
| `devres_head` | `struct list_head` | قائمة الـ managed resources |
| `class` | `const struct class *` | الـ class اللي ينتمي ليها الـ device |
| `groups` | `const struct attribute_group **` | attribute groups إضافية |
| `release` | `void (*)(struct device *)` | callback لتحرير الـ device لما refcount = 0 |
| `iommu_group` | `struct iommu_group *` | الـ IOMMU group |
| `iommu` | `struct dev_iommu *` | الـ IOMMU runtime data |
| `physical_location` | `struct device_physical_location *` | الموقع الفيزيائي |
| `removable` | `enum device_removable` | قابلية الإزالة |
| `offline_disabled:1` | bool bitfield | لو set، الـ device دايماً online |
| `offline:1` | bool bitfield | اتعمله offline بنجاح |
| `state_synced:1` | bool bitfield | الـ hardware state اتزامن مع الـ software |
| `can_match:1` | bool bitfield | اتطابق مع driver على الأقل مرة |
| `dma_coherent:1` | bool bitfield | الـ device coherent حتى لو المعمارية مش كده |
| `dma_iommu:1` | bool bitfield | بيستخدم الـ default IOMMU implementation للـ DMA |

---

#### 2. `struct device_driver` — تعريف الـ driver

| الـ Field | الغرض |
|---|---|
| `name` | اسم الـ driver |
| `bus` | الـ bus اللي بيشتغل عليه |
| `owner` | الـ module اللي فيه الـ driver |
| `suppress_bind_attrs` | إخفاء bind/unbind من sysfs |
| `probe_type` | استراتيجية الـ probe |
| `of_match_table` | جدول مطابقة الـ device tree |
| `acpi_match_table` | جدول مطابقة الـ ACPI |
| `probe` | callback لما device تتطابق |
| `sync_state` | callback لما كل consumers يتبندوا |
| `remove` | callback لما device تتشال |
| `shutdown` | callback وقت إغلاق النظام |
| `suspend/resume` | callbacks الـ power |
| `groups` | attributes على الـ driver نفسه في sysfs |
| `dev_groups` | attributes بتتضاف للـ device لما تتبند |
| `pm` | الـ power management ops |
| `coredump` | callback لعمل coredump |
| `p` | private data للـ driver core |

---

#### 3. `struct bus_type` — تعريف الـ bus

| الـ Field | الغرض |
|---|---|
| `name` | اسم الـ bus |
| `dev_name` | template للـ enumeration ("foo%u") |
| `bus_groups/dev_groups/drv_groups` | default attributes لـ sysfs |
| `match` | بيطابق device بـ driver، بيرجع > 0 لو match |
| `uevent` | بيضيف env variables للـ uevents |
| `probe` | بيعمل probe للـ device على الـ bus |
| `sync_state` | sync الـ hardware state |
| `remove` | بيشيل device من الـ bus |
| `shutdown` | تهدئة الـ device وقت shutdown |
| `online/offline` | hotplug support |
| `suspend/resume` | legacy PM callbacks |
| `dma_configure/cleanup` | إعداد وتنظيف الـ DMA |
| `pm` | الـ dev_pm_ops الموحدة |
| `need_parent_lock` | لازم نلوك الـ parent وقت probe/remove |

---

#### 4. `struct class` — تجريد أعلى مستوى للـ devices

| الـ Field | الغرض |
|---|---|
| `name` | اسم الـ class (مثلاً "block"، "net") |
| `class_groups` | attributes على الـ class نفسه |
| `dev_groups` | attributes على كل device في الـ class |
| `dev_uevent` | uevent callback للـ device |
| `devnode` | بيدي الـ devtmpfs path والـ mode |
| `class_release` | تحرير الـ class نفسه |
| `dev_release` | تحرير الـ device |
| `shutdown_pre` | قبل driver shutdown |
| `ns_type` | لـ namespace support في sysfs |
| `namespace` | بيرجع الـ namespace للـ device |
| `get_ownership` | uid/gid لـ sysfs entries |
| `pm` | default power ops للـ class |

---

#### 5. `struct device_link` — رابط dependency بين devices

| الـ Field | الغرض |
|---|---|
| `supplier` | الـ device اللي بيوفر الـ resource |
| `s_node` | hook في قائمة `supplier->links.consumers` |
| `consumer` | الـ device اللي بياخد الـ resource |
| `c_node` | hook في قائمة `consumer->links.suppliers` |
| `link_dev` | device حقيقي لعرض الـ link في sysfs |
| `status` | حالة الـ link (`enum device_link_state`) |
| `flags` | الـ `DL_FLAG_*` |
| `rpm_active` | `refcount_t` — هل الـ consumer active من ناحية RPM |
| `kref` | لمنع duplicate links |
| `rm_work` | work_struct لإزالة الـ link أسينكرونياً |
| `supplier_preactivated` | الـ supplier اتنشط قبل probe الـ consumer |

---

#### 6. `struct dev_links_info` — بيانات الـ links في الـ device

| الـ Field | الغرض |
|---|---|
| `suppliers` | list_head لقائمة الـ suppliers |
| `consumers` | list_head لقائمة الـ consumers |
| `defer_sync` | hook في الـ global deferred sync_state list |
| `status` | `enum dl_dev_state` — حالة الـ driver |

---

#### 7. `struct device_type` — نوع الـ device داخل bus أو class

| الـ Field | الغرض |
|---|---|
| `name` | بيظهر في DEVTYPE في الـ uevent |
| `groups` | attributes خاصة بالنوع |
| `uevent` | uevent callback |
| `devnode` | path وصلاحيات الـ device node |
| `release` | تحرير الـ device |
| `pm` | power ops خاصة بالنوع |

---

#### 8. `struct device_attribute` و`struct dev_ext_attribute`

```c
struct device_attribute {
    struct attribute attr;           /* name + mode */
    ssize_t (*show)(..., char *buf); /* قراءة من sysfs */
    ssize_t (*store)(..., const char *buf, size_t count); /* كتابة */
};

struct dev_ext_attribute {
    struct device_attribute attr;    /* الـ attribute الأساسي */
    void *var;                       /* pointer لمتغير تلقائي */
};
```

**الـ Macros للتعريف السريع:**

| الـ Macro | النتيجة |
|---|---|
| `DEVICE_ATTR_RW(foo)` | mode=0644، show=foo_show، store=foo_store |
| `DEVICE_ATTR_RO(foo)` | mode=0444، show=foo_show فقط |
| `DEVICE_ATTR_WO(foo)` | mode=0200، store=foo_store فقط |
| `DEVICE_ATTR_ADMIN_RW(foo)` | mode=0600، RW للـ admin فقط |
| `DEVICE_ULONG_ATTR(foo, mode, var)` | مربوط أوتوماتيك بـ unsigned long var |
| `DEVICE_INT_ATTR(foo, mode, var)` | مربوط بـ int var |
| `DEVICE_BOOL_ATTR(foo, mode, var)` | مربوط بـ bool var |
| `DEVICE_STRING_ATTR_RO(foo, mode, var)` | قراءة فقط من string |

---

#### 9. `struct device_dma_parameters`

```c
struct device_dma_parameters {
    unsigned int max_segment_size;      /* أكبر segment في bytes */
    unsigned int min_align_mask;        /* الـ alignment الأدنى */
    unsigned long segment_boundary_mask;/* حد الـ segment boundary */
};
```

بيعلّم الـ IOMMU بقيود الـ scatter-gather.

---

#### 10. `struct device_physical_location`

بيصف الموقع الفيزيائي للـ port (الوجه، الموضع، هل هو في docking station، هل على غطاء اللابتوب).

---

#### 11. `struct subsys_interface` و`struct class_interface`

بيسمحوا لأكتر من entity إنها تتعامل مع devices في subsystem/class من غير ما تـ "تملكهم" زي الـ driver.

---

### مخططات العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────┐
                    │         struct bus_type          │
                    │  name, match(), probe(), pm      │
                    └────────────┬────────────────────┘
                                 │ bus (pointer)
                    ┌────────────▼────────────────────┐
                    │      struct device_driver        │
                    │  name, probe(), remove()         │
                    │  of_match_table, pm              │
                    └────────────┬────────────────────┘
                                 │ driver (pointer)
                    ┌────────────▼────────────────────┐
                    │         struct device            │◄──── kobj_to_dev()
                    │  ┌──────────────────────────┐   │
                    │  │ struct kobject kobj       │   │
                    │  └──────────────────────────┘   │
                    │  parent ──────────────────────► struct device (parent)
                    │  bus   ───────────────────────► struct bus_type
                    │  driver ──────────────────────► struct device_driver
                    │  class ───────────────────────► struct class
                    │  type  ───────────────────────► struct device_type
                    │  p     ───────────────────────► struct device_private
                    │  links ───────────────────────► struct dev_links_info
                    │  power ───────────────────────► struct dev_pm_info
                    │  pm_domain ───────────────────► struct dev_pm_domain
                    │  of_node ─────────────────────► struct device_node
                    │  fwnode ──────────────────────► struct fwnode_handle
                    │  iommu_group ─────────────────► struct iommu_group
                    │  msi.domain ──────────────────► struct irq_domain
                    │  dma_parms ───────────────────► struct device_dma_parameters
                    └─────────────────────────────────┘
```

---

### مخطط الـ Device Links

```
   ┌──────────────────┐         struct device_link         ┌──────────────────┐
   │  supplier device │                                     │ consumer device  │
   │                  │◄── supplier ──┌──────────────┐      │                  │
   │  links.consumers │               │device_link   │      │  links.suppliers │
   │  (list_head) ────┼──── s_node ──►│  flags       │      │  (list_head)     │
   │                  │               │  status      │◄─────┼── c_node         │
   └──────────────────┘               │  rpm_active  │      └──────────────────┘
                                      │  kref        │
                                      │  link_dev ───┼──► struct device (sysfs)
                                      └──────────────┘
```

---

### مخطط الـ Class/Bus/Device Hierarchy

```
struct class ("block")
    │
    ├── struct class_interface (يراقب كل devices)
    │
    └── struct device ("sda")         struct device ("sdb")
            │                                │
            ├── bus ──► struct bus_type ("scsi")
            ├── driver ─► struct device_driver ("sd")
            └── type ──► struct device_type ("disk")
```

---

### Lifecycle الـ Device

```
[Allocation]
    │
    │  كل الـ fields بتكون zeroed أو initialized
    ▼
device_initialize(dev)
    │  kobj init، mutex init، devres list init
    │  links init، pm init
    ▼
device_add(dev)
    │  kobject_add() ← ينشئ الـ sysfs entry
    │  bus_add_device() ← يضيف الـ device للـ bus
    │  bus_probe_device() ← يحاول يلاقي driver
    │  device_create_file() ← ينشئ attributes
    │  blocking_notifier_call_chain(BUS_NOTIFY_ADD_DEVICE)
    ▼
[Device is live — driver bound or waiting]
    │
    ▼
device_del(dev)
    │  bus_remove_device()
    │  device_remove_file() ← بيشيل attributes
    │  blocking_notifier_call_chain(BUS_NOTIFY_DEL_DEVICE)
    │  kobject_del() ← بيشيل الـ sysfs entry
    ▼
put_device(dev)  [refcount ← 0]
    │
    ▼
dev->release(dev)   ← driver/bus allocated memory free هنا
```

---

### Lifecycle الـ Driver

```
driver_register(drv)
    │  bus_add_driver()
    │  driver_attach() ← يحاول يبند كل device مطابقة
    │
    ▼  لكل device تطابق:
bus->match(dev, drv) > 0 ?
    │  YES
    ▼
really_probe(dev, drv)
    │  dev->driver = drv
    │  bus->probe(dev)  أو  drv->probe(dev)
    │    ← لو نجح: BUS_NOTIFY_BOUND_DRIVER
    │    ← لو فشل بـ EPROBE_DEFER: أضيف لـ deferred list
    ▼
[Driver bound]
    │
    ▼
device_release_driver(dev)
    │  drv->remove(dev)
    │  dev->driver = NULL
    │  BUS_NOTIFY_UNBOUND_DRIVER
    ▼
driver_unregister(drv)
```

---

### Lifecycle الـ Device Link

```
device_link_add(consumer, supplier, flags)
    │  alloc struct device_link
    │  list_add supplier→links.consumers
    │  list_add consumer→links.suppliers
    │  kobject_add(link_dev) ← ظهور في sysfs
    │  تحديد initial status بناءً على حالة الـ drivers
    ▼
[Link active — يتحكم في RPM وprobe ordering]
    │
    │  لما consumer يعمل probe:
    │    status → DL_STATE_CONSUMER_PROBE
    │    لما ينتهي: → DL_STATE_ACTIVE
    │
    │  لما supplier يعمل unbind:
    │    status → DL_STATE_SUPPLIER_UNBIND
    │    الـ consumer بيعمل unbind كمان
    ▼
device_link_del(link)
    │  بيحذف من القوائم
    │  schedule rm_work (async removal)
    ▼
[Link removed]
```

---

### Call Flow — Driver Probe

```
bus_probe_device(dev)
  └─► __device_attach(dev)
        └─► bus_for_each_drv()  [iterate all drivers]
              └─► __device_attach_driver(drv, dev)
                    └─► driver_match_device(drv, dev)
                          └─► bus->match(dev, drv)   [hardware ID check]
                                └─► really_probe(dev, drv)
                                      │
                                      ├─ pinctrl_bind_pins(dev)
                                      ├─ dma_configure(dev)
                                      ├─ dev->driver = drv
                                      │
                                      ├─► bus->probe(dev)
                                      │     └─► drv->probe(dev)
                                      │           └─► [driver code runs]
                                      │                 ← dev_set_drvdata()
                                      │                 ← devm_*() allocations
                                      │
                                      └─ sysfs driver link created
```

---

### Call Flow — Device Creation من الـ Subsystem

```
device_create(cls, parent, devt, drvdata, fmt, ...)
  └─► kobject_set_name()           [set name from fmt]
  └─► device_initialize(dev)
  └─► dev->class = cls
  └─► dev->devt  = devt
  └─► dev_set_drvdata(dev, drvdata)
  └─► device_add(dev)
        ├─► kobject_add()          [/sys/class/xxx/name]
        ├─► device_create_file()   [dev attribute → major:minor]
        ├─► device_add_groups()    [class dev_groups]
        └─► kobject_uevent(KOBJ_ADD) → udevd → /dev node created
```

---

### Call Flow — Device Attribute Read من sysfs

```
user: cat /sys/bus/platform/devices/foo/my_attr
  └─► VFS read()
        └─► sysfs_kf_seq_show()
              └─► dev_attr_show()
                    └─► attr->show(dev, attr, buf)
                          └─► [driver's show function]
                                └─► sprintf(buf, "%d\n", value)
                                      └─► return len to user
```

---

### Call Flow — sync_state

```
late_initcall_sync:
  device_links_supplier_sync_state_resume()
    └─► لكل device في defer_sync list:
          dev_has_sync_state(dev) ?
            ├─► drv->sync_state(dev)   [لو الـ driver عنده]
            └─► bus->sync_state(dev)   [لو الـ bus عنده]
```

---

### استراتيجية الـ Locking

#### الـ Locks الأساسية وما بيحموه

| الـ Lock | النوع | ما بيحميه | كيفية الاستخدام |
|---|---|---|---|
| `dev->mutex` | `struct mutex` | الـ calls للـ driver (probe/remove)، وكل عمليات الـ bind/unbind | `device_lock(dev)` / `device_unlock(dev)` |
| `dev->devres_lock` | `spinlock_t` | قائمة الـ devres resources | `spin_lock_irqsave()` |
| `dev->kobj.uevent_suppress` | atomic (في kobj) | الـ uevent suppression flag | |
| hotplug_lock | global mutex (خارج device.h) | عمليات الـ hotplug من sysfs | `lock_device_hotplug()` |

#### ترتيب الـ Locks (Lock Ordering)

```
hotplug_lock           (الأعلى — global)
    └─► parent->mutex  (لو bus->need_parent_lock)
          └─► dev->mutex
                └─► dev->devres_lock  (spinlock — الأسفل)
```

**قاعدة مهمة:** لو عندك `dev->mutex` وعايز تعمل حاجة على الـ parent، لازم تاخد `parent->mutex` الأول قبل ما تدخل probe.

#### الـ Lockdep Integration

```c
/* بيغير الـ lock class مؤقتاً وقت bind */
device_lock_set_class(dev, &my_lock_class_key);
    /* داخل probe() */

/* بيرجع للـ default بعد unbind */
device_lock_reset_class(dev);
    /* داخل remove() */
```

ده بيساعد الـ lockdep يفرق بين mutex الـ parent والـ child من غير ما يـ false-positive.

#### RPM Locking في الـ Device Links

```
DL_FLAG_PM_RUNTIME + DL_FLAG_RPM_ACTIVE:
    وقت إنشاء الـ link:
        pm_runtime_get_sync(supplier)    ← link→rpm_active++
    وقت consumer probe ينجح:
        الـ link يعمل pm_runtime_put(supplier)  ← rpm_active--
    وقت إزالة الـ link:
        لو rpm_active > 0: pm_runtime_put_sync(supplier)
```

#### الـ Bus Notifier Locking

> تحذير من الكود نفسه: الـ bus notifiers بيتعملوا call وهما شايلين `dev->mutex` بالفعل، فمحدش يعمل `device_lock()` جوه الـ notifier callback.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function | Category | الغرض |
|---|---|---|
| `device_initialize()` | Lifecycle | تهيئة الـ `struct device` بدون إضافته للـ sysfs |
| `device_add()` | Lifecycle | إضافة device مهيأ مسبقاً للـ driver core |
| `device_register()` | Lifecycle | `initialize` + `add` في خطوة واحدة |
| `device_del()` | Lifecycle | حذف device من الـ driver core وإزالة sysfs entries |
| `device_unregister()` | Lifecycle | `del` + `put_device` في خطوة واحدة |
| `device_create()` | Dynamic | إنشاء device ديناميكياً مع class وdev_t |
| `device_create_with_groups()` | Dynamic | نفس `device_create` مع attribute groups إضافية |
| `device_destroy()` | Dynamic | تدمير device أُنشئ بـ `device_create` |
| `root_device_register()` | Root | تسجيل root device تحت `/sys/devices` |
| `root_device_unregister()` | Root | حذف root device |

#### Reference Counting

| Function | الغرض |
|---|---|
| `get_device()` | زيادة الـ refcount بشكل atomic |
| `put_device()` | تقليل الـ refcount — يستدعي `release` عند الوصول لـ 0 |
| `kill_device()` | إخبار الـ core أن الـ device لن يُسجَّل |

#### Driver Binding

| Function | الغرض |
|---|---|
| `device_bind_driver()` | ربط driver بـ device يدوياً |
| `device_release_driver()` | فك ربط driver عن device |
| `device_attach()` | البحث عن driver مناسب وربطه |
| `driver_attach()` | فرض ربط driver بكل devices المناسبة على الـ bus |
| `device_driver_attach()` | ربط driver محدد بـ device محدد |
| `device_reprobe()` | إعادة probe device (يفك الربط ثم يُعيده) |
| `device_is_bound()` | هل الـ device مرتبط بـ driver؟ |
| `device_initial_probe()` | أول probe عند إضافة device |

#### Child/Parent Traversal

| Function | الغرض |
|---|---|
| `device_for_each_child()` | تكرار على كل الـ children بالترتيب |
| `device_for_each_child_reverse()` | تكرار عكسي |
| `device_for_each_child_reverse_from()` | تكرار عكسي بدءاً من device معين |
| `device_find_child()` | إيجاد child بدالة match مخصصة |
| `device_find_child_by_name()` | إيجاد child باسمه |
| `device_find_any_child()` | إيجاد أي child |

#### Bus Traversal

| Function | الغرض |
|---|---|
| `bus_for_each_dev()` | تكرار على كل devices في bus |
| `bus_find_device()` | إيجاد device في bus بدالة match |
| `bus_find_device_by_name()` | إيجاد device بالاسم |
| `bus_find_device_by_of_node()` | إيجاد device بـ of_node |
| `bus_find_device_by_fwnode()` | إيجاد device بـ fwnode |
| `bus_find_device_by_devt()` | إيجاد device بـ dev_t |
| `bus_for_each_drv()` | تكرار على كل الـ drivers في bus |

#### Device Links

| Function | الغرض |
|---|---|
| `device_link_add()` | إنشاء رابط supplier→consumer |
| `device_link_del()` | حذف رابط stateless |
| `device_link_remove()` | حذف رابط بالـ consumer pointer |
| `device_links_supplier_sync_state_pause()` | تعليق sync_state |
| `device_links_supplier_sync_state_resume()` | استئناف sync_state |
| `device_link_wait_removal()` | انتظار اكتمال حذف الروابط |
| `device_link_test()` | اختبار flags في رابط |

#### sysfs Attributes

| Function | الغرض |
|---|---|
| `device_create_file()` | إضافة attribute ملف لـ sysfs |
| `device_remove_file()` | حذف attribute ملف من sysfs |
| `device_remove_file_self()` | حذف attribute من داخل show/store callback نفسه |
| `device_create_bin_file()` | إضافة binary attribute |
| `device_remove_bin_file()` | حذف binary attribute |
| `device_add_groups()` | إضافة مجموعة attributes |
| `device_remove_groups()` | حذف مجموعة attributes |
| `device_add_group()` | إضافة group واحد (wrapper) |
| `device_remove_group()` | حذف group واحد (wrapper) |
| `devm_device_add_group()` | managed version لـ `device_add_group` |

#### Naming & Topology

| Function | الغرض |
|---|---|
| `dev_set_name()` | تعيين اسم الـ device (printf-style) |
| `dev_name()` | الحصول على اسم الـ device |
| `dev_bus_name()` | الحصول على اسم الـ bus أو class |
| `device_rename()` | إعادة تسمية device (مع sysfs) |
| `device_move()` | نقل device لـ parent جديد |
| `device_change_owner()` | تغيير uid/gid لـ sysfs entries |

#### Firmware Node

| Function | الغرض |
|---|---|
| `set_primary_fwnode()` | تعيين primary fwnode |
| `set_secondary_fwnode()` | تعيين secondary fwnode |
| `device_set_node()` | تعيين fwnode مع of_node |
| `device_add_of_node()` | إضافة of_node لـ device |
| `device_remove_of_node()` | إزالة of_node من device |
| `device_set_of_node_from_dev()` | نسخ of_node من device آخر |
| `get_dev_from_fwnode()` | إيجاد device من fwnode handle |
| `dev_of_node()` | الحصول على of_node من device |

#### Power Management Helpers (inline)

| Function | الغرض |
|---|---|
| `device_enable_async_suspend()` | تفعيل async suspend |
| `device_disable_async_suspend()` | تعطيل async suspend |
| `device_async_suspend_enabled()` | هل async suspend مفعّل؟ |
| `device_set_pm_not_required()` | علامة أن الـ PM غير مطلوب |
| `dev_pm_syscore_device()` | تعيين syscore flag |
| `dev_pm_set_driver_flags()` | تعيين PM driver flags |
| `dev_pm_test_driver_flags()` | اختبار PM driver flags |
| `dev_pm_smart_suspend()` | هل smart_suspend مفعّل؟ |
| `dev_pm_set_strict_midlayer()` | تعيين strict_midlayer flag |

#### Locking

| Function | الغرض |
|---|---|
| `device_lock()` | قفل device mutex |
| `device_unlock()` | فتح device mutex |
| `device_trylock()` | محاولة قفل بدون انتظار |
| `device_lock_interruptible()` | قفل قابل للمقاطعة |
| `device_lock_assert()` | lockdep assertion |
| `lock_device_hotplug()` | قفل hotplug global lock |
| `unlock_device_hotplug()` | فتح hotplug global lock |
| `lock_device_hotplug_sysfs()` | قفل hotplug من sysfs context |

#### devres (managed resources)

| Function | الغرض |
|---|---|
| `devres_alloc()` | تخصيص devres entry |
| `devres_free()` | تحرير devres entry غير مُضاف |
| `devres_add()` | إضافة entry لـ device's devres list |
| `devres_find()` | إيجاد entry في devres list |
| `devres_get()` | إيجاد أو إضافة entry |
| `devres_remove()` | إزالة entry من القائمة بدون release |
| `devres_destroy()` | إزالة + تحرير entry |
| `devres_release()` | إزالة + استدعاء release callback |
| `devres_open_group()` | فتح group للـ devres entries |
| `devres_close_group()` | إغلاق group |
| `devres_remove_group()` | حذف group بدون release |
| `devres_release_group()` | حذف group مع release |
| `devm_kmalloc()` | managed kmalloc |
| `devm_kzalloc()` | managed kzalloc |
| `devm_krealloc()` | managed krealloc |
| `devm_kfree()` | تحرير managed allocation يدوياً |
| `devm_kmemdup()` | managed kmemdup |
| `devm_kstrdup()` | managed kstrdup |
| `devm_kasprintf()` | managed kasprintf |
| `devm_ioremap_resource()` | managed ioremap من resource |
| `devm_add_action()` | إضافة cleanup action |
| `devm_add_action_or_reset()` | إضافة action أو تنفيذه فوراً عند الفشل |
| `devm_remove_action()` | حذف action من الـ devres stack |
| `devm_release_action()` | تنفيذ action وحذفه |

#### Bus Registration & Subsystem

| Function | الغرض |
|---|---|
| `bus_register()` | تسجيل bus_type جديد |
| `bus_unregister()` | إلغاء تسجيل bus_type |
| `bus_rescan_devices()` | إعادة scan كل devices على bus |
| `subsys_interface_register()` | تسجيل subsystem interface |
| `subsys_interface_unregister()` | إلغاء تسجيل subsystem interface |
| `subsys_system_register()` | تسجيل bus كـ system subsystem |
| `subsys_virtual_register()` | تسجيل virtual bus subsystem |
| `bus_register_notifier()` | تسجيل notifier للـ bus events |
| `bus_unregister_notifier()` | إلغاء تسجيل notifier |

#### Driver Registration

| Function | الغرض |
|---|---|
| `driver_register()` | تسجيل driver في الـ bus |
| `driver_unregister()` | إلغاء تسجيل driver |
| `driver_find()` | إيجاد driver بالاسم في bus |
| `driver_for_each_device()` | تكرار على devices مرتبطة بـ driver |
| `driver_find_device()` | إيجاد device مرتبط بـ driver |
| `driver_deferred_probe_add()` | إضافة device لـ deferred probe queue |
| `driver_deferred_probe_check_state()` | التحقق من حالة deferred probe |

---

### 1. تسجيل وإدارة دورة حياة الـ Device

هذه المجموعة هي جوهر الـ driver model — تتحكم في إضافة وحذف devices من الـ kernel's device tree وربطها بـ sysfs.

---

#### `device_initialize()`

```c
void device_initialize(struct device *dev);
```

تهيئة الـ `struct device` ذاتياً: تُعِدّ الـ `kobject`، تُهيئ الـ mutex، وتُعدّ قوائم الـ devres والـ links. لا تُضيف الـ device للـ sysfs ولا تُسجّلها في الـ bus — هي مجرد تهيئة داخلية.

**Parameters:**
- `dev` — مؤشر للـ `struct device` المُخصَّصة مسبقاً (عادةً مضمّنة في larger structure)

**Return:** void

**Key details:**
- تُعيّن refcount الـ kobject على 1
- تُهيئ `dev->mutex` و `dev->devres_lock` و `dev->devres_head`
- تُعيّن `dev->links.status = DL_DEV_NO_DRIVER`
- يجب استدعاؤها قبل أي تعديل على fields الـ device
- لا تُسجّل في sysfs — آمن للاستدعاء في أي context

**Who calls it:** bus drivers، platform code، أي كود يريد الفصل بين initialize وadd

---

#### `device_add()`

```c
int __must_check device_add(struct device *dev);
```

يُكمل تسجيل device مُهيَّأة مسبقاً بـ `device_initialize()`. يُضيف الـ kobject للـ sysfs، يُنشئ الـ symlinks، يُطلق bus notification، ويُحاول probe.

**Parameters:**
- `dev` — device مُهيَّأة بـ `device_initialize()`

**Return:** 0 نجاح، errno سالب عند الفشل (`__must_check` — لازم تتحقق)

**Key details:**
- يستدعي `kobject_add()` لإنشاء sysfs entry
- يُطلق `BUS_NOTIFY_ADD_DEVICE` notification
- يستدعي `bus_probe_device()` لمحاولة probe
- يُنشئ `dev` entry في sysfs إذا كان `dev->devt` معيّناً
- عند الفشل، يُنظّف كل ما أُضيف (cleanup path كامل)

**Pseudocode:**
```
device_add(dev):
    set_dev_node(dev)
    kobject_add(&dev->kobj, dev->parent->kobj, dev->init_name)
    bus_add_device(dev)              // sysfs bus symlinks
    dpm_sysfs_add(dev)               // power management sysfs
    device_add_attrs(dev)            // type + class + group attrs
    bus_notify(BUS_NOTIFY_ADD_DEVICE)
    kobject_uevent(KOBJ_ADD)         // netlink uevent to udevd
    bus_probe_device(dev)            // try to find & bind driver
    device_pm_add(dev)               // add to PM list
```

**Who calls it:** `device_register()` و platform bus init code

---

#### `device_register()`

```c
int __must_check device_register(struct device *dev);
```

**الـ one-stop shop** لتسجيل device — تستدعي `device_initialize()` ثم `device_add()` في خطوة واحدة.

**Parameters:**
- `dev` — struct device مُعبَّأة (يجب أن تكون `init_name`، `parent`، `bus`، `release` معيّنة)

**Return:** 0 نجاح، errno سالب عند الفشل

**Key details:**
- الفرق الوحيد عن `initialize()+add()`: لا يمكن إضافة devres entries بين الخطوتين
- يجب تعيين `dev->release` قبل الاستدعاء وإلا WARN
- `dev->parent` إذا كان NULL فالـ device يظهر تحت `/sys/devices` مباشرة

**Who calls it:** معظم bus drivers في init code

---

#### `device_del()`

```c
void device_del(struct device *dev);
```

يعكس `device_add()` — يُزيل الـ device من الـ sysfs والـ bus وقوائم PM، لكنه لا يُحرّر الـ struct نفسها (refcount لا يزال قائماً).

**Parameters:**
- `dev` — device مُسجَّلة مسبقاً

**Return:** void

**Key details:**
- يستدعي `bus_remove_device()` → driver unbind أولاً
- يُطلق `BUS_NOTIFY_DEL_DEVICE` ثم `BUS_NOTIFY_REMOVED_DEVICE`
- يحذف كل sysfs files والـ symlinks
- يُزيل device من PM list
- **لا يُحرّر memory** — يجب استدعاء `put_device()` بعده
- `DEFINE_FREE(device_del, ...)` متاح للـ cleanup.h scope-based cleanup

**Who calls it:** `device_unregister()` وbus drivers في remove path

---

#### `device_unregister()`

```c
void device_unregister(struct device *dev);
```

يستدعي `device_del()` ثم `put_device()`. هذا هو الـ pair الصحيح لـ `device_register()`.

**Parameters:**
- `dev` — device مُسجَّلة بـ `device_register()`

**Return:** void

**Key details:**
- بعد الاستدعاء، إذا كان refcount = 0 فـ `dev->release()` تُستدعى
- إذا كانت هناك references أخرى (من `get_device()`)، الـ `release` تُؤجَّل

**Who calls it:** أي كود يُسجّل device بـ `device_register()`

---

#### `device_create()`

```c
__printf(5, 6) struct device *
device_create(const struct class *cls, struct device *parent, dev_t devt,
              void *drvdata, const char *fmt, ...);
```

إنشاء device ديناميكياً وتسجيلها — مفيد جداً في character drivers والـ misc devices.

**Parameters:**
- `cls` — الـ class الذي تنتمي له الـ device (مثل `misc_class`)
- `parent` — الـ parent device أو NULL
- `devt` — الـ major:minor لإنشاء `/dev` entry عبر udev
- `drvdata` — driver private data (يُعيّن بـ `dev_set_drvdata()`)
- `fmt, ...` — printf-style format لاسم الـ device

**Return:** مؤشر للـ device الجديدة، أو `ERR_PTR(-errno)` عند الفشل

**Key details:**
- تُخصّص `struct device` داخلياً وتُحرَّر تلقائياً عند `device_destroy()`
- تُنشئ sysfs entry تحت `/sys/class/<cls>/`
- udevd تُنشئ `/dev/<name>` تلقائياً بناءً على الـ uevent
- الـ devres الخاص بالـ device تتحرر عند `device_destroy()`

**Who calls it:** character device drivers (مثل drivers في `drivers/char/`)

---

#### `device_destroy()`

```c
void device_destroy(const struct class *cls, dev_t devt);
```

يبحث عن device بالـ class وdevt ثم يستدعي `device_unregister()` عليها.

**Parameters:**
- `cls` — class الـ device
- `devt` — major:minor للبحث

**Return:** void

**Key details:**
- يستدعي `get_device()` أثناء البحث، ثم `put_device()` بعد الحذف
- إذا لم يجد الـ device لا يفعل شيئاً (لا يُطلق error)

---

### 2. إدارة الـ Reference Counting

---

#### `get_device()`

```c
struct device *get_device(struct device *dev);
```

يزيد الـ kobject refcount بشكل atomic. يضمن أن الـ device لن تُحذف memory-wise طالما أنت تمسكها.

**Parameters:**
- `dev` — device المراد الإمساك بها (يمكن أن تكون NULL)

**Return:** نفس الـ `dev` المُمرَّر، أو NULL إذا كان `dev` نفسه NULL

**Key details:**
- wrapper بسيط على `kobject_get(&dev->kobj)`
- آمن للاستدعاء من أي context (atomic-safe)
- يجب مقابلتها بـ `put_device()` حتماً

---

#### `put_device()`

```c
void put_device(struct device *dev);
```

يُقلّل الـ refcount. عند وصوله لـ 0، يستدعي `dev->release()` (أو `device_type->release`) الذي يُحرّر الـ memory.

**Parameters:**
- `dev` — device يُراد تحرير الـ reference منها

**Return:** void

**Key details:**
- عند refcount == 0: `device_release()` → `dev->type->release()` أو `dev->class->dev_release()` أو `dev->release()`
- **لا تقترب من الـ device بعد `put_device()` إذا كنت تملك آخر reference**
- `DEFINE_FREE(put_device, ...)` متاح لاستخدامها مع `__free(put_device)`

---

#### `kill_device()`

```c
bool kill_device(struct device *dev);
```

يُعلّم الـ device بأنها "ميتة" — يمنع أي تسجيل devres جديد عليها ويُسرّع cleanup.

**Parameters:**
- `dev` — الـ device المُستهدفة

**Return:** `true` إذا كانت الـ device حيّة قبل الاستدعاء، `false` إذا كانت ميتة أصلاً

**Key details:**
- مفيد في error paths عندما تريد منع driver من إضافة resources بعد فشل probe
- لا يُزيل الـ device من sysfs — هذا دور `device_del()`

---

### 3. ربط Driver بـ Device (Binding)

---

#### `device_attach()`

```c
int __must_check device_attach(struct device *dev);
```

يبحث في كل drivers المسجّلة على الـ bus ليجد driver مناسب ويُجري الـ probe.

**Parameters:**
- `dev` — device يُراد ربطها بـ driver

**Return:**
- `1` إذا تمّ الربط بنجاح
- `0` إذا لم يُجرِ probe (مثلاً الـ device مرتبطة بالفعل أو suppressed)
- errno سالب عند خطأ حقيقي

**Key details:**
- يحتاج device lock (`dev->mutex`) يجب أن يكون محجوزاً من المُستدعي
- يتكرر على `bus->p->drivers_kset` بـ `bus_for_each_drv()`
- إذا أعاد driver الـ `-EPROBE_DEFER` يُضاف لـ deferred probe list

---

#### `device_driver_attach()`

```c
int __must_check device_driver_attach(const struct device_driver *drv,
                                      struct device *dev);
```

يربط driver محدد بـ device محدد مباشرةً — يتحقق من الـ match أولاً ثم يُجري probe.

**Parameters:**
- `drv` — الـ driver المُراد ربطه
- `dev` — الـ device المُستهدفة

**Return:** 0 نجاح، errno سالب فشل

**Key details:**
- يستدعي `bus->match(dev, drv)` أولاً
- يحجز `device_lock()` داخلياً
- يستدعي `really_probe()` الذي يستدعي `drv->probe()`

---

#### `device_release_driver()`

```c
void device_release_driver(struct device *dev);
```

يفك ربط الـ driver عن الـ device ويستدعي `drv->remove()`.

**Parameters:**
- `dev` — device المُراد فك ربطها

**Return:** void

**Key details:**
- يحجز `device_lock()` داخلياً
- يستدعي `__device_release_driver()` → `drv->remove()` → devres cleanup
- يُطلق `BUS_NOTIFY_UNBIND_DRIVER` و `BUS_NOTIFY_UNBOUND_DRIVER`

---

#### `driver_attach()`

```c
int __must_check driver_attach(const struct device_driver *drv);
```

يُحاول ربط الـ driver بجميع الـ devices المناسبة الموجودة على الـ bus.

**Parameters:**
- `drv` — الـ driver المُراد ربطه

**Return:** دائماً 0 (الأخطاء الفردية تُتجاهل لإكمال الـ scan)

**Key details:**
- يستدعي `bus_for_each_dev(drv->bus, NULL, drv, __driver_attach)`
- يُستخدم عند تسجيل driver جديد لتوصيله بالـ devices الموجودة مسبقاً

---

#### `device_reprobe()`

```c
int __must_check device_reprobe(struct device *dev);
```

يفك الربط الحالي ثم يُجري probe من جديد — مفيد عند تغيّر firmware أو power state.

**Parameters:**
- `dev` — device يُراد إعادة probe-ها

**Return:** نتيجة `device_attach()` الجديدة

**Key details:**
- يستدعي `device_release_driver()` أولاً إذا كان مرتبطاً
- **لا تستدعه من probe() أو remove() — deadlock محتمل**
- يُشغَّل من workqueue في بعض الحالات

---

### 4. تكرار على الـ Children والـ Bus

---

#### `device_for_each_child()`

```c
int device_for_each_child(struct device *parent, void *data,
                          device_iter_t fn);
```

يُكرّر على كل الـ children المباشرة للـ parent ويستدعي `fn` على كلٍّ منهم بالترتيب.

**Parameters:**
- `parent` — الـ parent device
- `data` — pointer مُمرَّر لـ `fn` بدون تعديل
- `fn` — callback يُستدعى على كل child: `int fn(struct device *dev, void *data)`

**Return:** أول قيمة non-zero تُعيدها `fn`، أو 0 إذا اكتمل الـ iteration

**Key details:**
- يحجز klist lock داخلياً — آمن من concurrent add/remove
- إذا أعادت `fn` قيمة non-zero يتوقف الـ iteration فوراً
- يستدعي `get_device()`/`put_device()` على كل child أثناء الـ iteration

---

#### `device_find_child()`

```c
struct device *device_find_child(struct device *parent, const void *data,
                                 device_match_t match);
```

يُكرّر على الـ children ويُعيد أول child تُعيد فيه `match` قيمة non-zero.

**Parameters:**
- `parent` — الـ parent device
- `data` — بيانات المقارنة تُمرَّر لـ `match`
- `match` — `int match(struct device *dev, const void *data)`

**Return:** مؤشر للـ device مع refcount مُزاد، أو NULL — **يجب استدعاء `put_device()` بعد الاستخدام**

**Key details:**
- يُزيد الـ refcount تلقائياً على الـ device المُعادة
- استخدم `device_find_child_by_name()` أو `device_find_any_child()` كـ wrappers جاهزة

---

#### `bus_for_each_dev()`

```c
int bus_for_each_dev(const struct bus_type *bus, struct device *start,
                     void *data, device_iter_t fn);
```

يُكرّر على كل الـ devices المسجّلة على bus بدءاً من `start` (أو من الأول إذا كان NULL).

**Parameters:**
- `bus` — الـ bus المُستهدف
- `start` — device ابدأ بعدها (NULL = من البداية)
- `data` — يُمرَّر لـ `fn`
- `fn` — callback على كل device

**Return:** أول قيمة non-zero من `fn`، أو 0

**Key details:**
- يستخدم `klist_iter` على `bus->p->klist_devices`
- آمن من concurrent modifications بفضل klist locking
- `start` إذا أُعطي يجب أن تكون مسجّلة على نفس الـ bus

---

#### `bus_find_device()`

```c
struct device *bus_find_device(const struct bus_type *bus, struct device *start,
                               const void *data, device_match_t match);
```

يبحث عن device في bus باستخدام دالة match مخصصة.

**Parameters:**
- `bus` — الـ bus المُستهدف
- `start` — ابدأ البحث بعد هذه الـ device (NULL = من البداية)
- `data` — data تُمرَّر لـ `match`
- `match` — دالة المقارنة

**Return:** device مع refcount مُزاد أو NULL — **يجب `put_device()` بعدها**

**Key details:**
- باقي الـ `bus_find_device_by_*` functions هي wrappers على هذه
- المقارنات الجاهزة: `device_match_name`, `device_match_of_node`, `device_match_fwnode`, `device_match_devt`, `device_match_acpi_dev`, `device_match_any`

---

### 5. Device Links (Supplier-Consumer)

الـ **device links** تُمثّل تبعيات runtime بين devices: supplier يجب أن يكون active قبل consumer.

---

#### `device_link_add()`

```c
struct device_link *device_link_add(struct device *consumer,
                                    struct device *supplier, u32 flags);
```

يُنشئ رابطاً بين consumer وsupplier مع الـ flags المطلوبة.

**Parameters:**
- `consumer` — الـ device التي تعتمد على الـ supplier
- `supplier` — الـ device المُزوِّدة
- `flags` — مزيج من:
  - `DL_FLAG_STATELESS` — الـ core لن يُدير هذا الرابط تلقائياً
  - `DL_FLAG_AUTOREMOVE_CONSUMER` — احذف الرابط عند unbind الـ consumer driver
  - `DL_FLAG_AUTOREMOVE_SUPPLIER` — احذف الرابط عند unbind الـ supplier driver
  - `DL_FLAG_PM_RUNTIME` — استخدم الرابط لإدارة runtime PM
  - `DL_FLAG_RPM_ACTIVE` — استدعِ `pm_runtime_get_sync()` على الـ supplier فوراً
  - `DL_FLAG_AUTOPROBE_CONSUMER` — probe الـ consumer تلقائياً بعد supplier probe

**Return:** مؤشر للـ `struct device_link`، أو NULL عند الفشل

**Key details:**
- يحجز `device_links_write_lock()` (rwsem) داخلياً
- إذا كان الرابط موجوداً مسبقاً يُعيد نفس الـ pointer (مع زيادة kref)
- يُؤثّر على ترتيب suspend/resume: supplier يُعاد تشغيله قبل consumer
- الـ links تظهر في sysfs تحت `devices/<consumer>/supplier:<supplier>/`

**Pseudocode:**
```
device_link_add(consumer, supplier, flags):
    lock device_links_write_lock
    link = find_existing_link(consumer, supplier)
    if link:
        kref_get(&link->kref)
        return link
    link = kzalloc(struct device_link)
    link->supplier = get_device(supplier)
    link->consumer = get_device(consumer)
    link->flags = flags | DL_FLAG_MANAGED
    init_link_state(link, supplier->links.status)
    list_add(&link->s_node, &supplier->links.consumers)
    list_add(&link->c_node, &consumer->links.suppliers)
    if flags & DL_FLAG_PM_RUNTIME:
        pm_runtime_new_link(consumer)
    kobject_add(&link->link_dev.kobj)  // sysfs
    unlock
    return link
```

---

#### `device_link_del()`

```c
void device_link_del(struct device_link *link);
```

يحذف رابط **stateless** (`DL_FLAG_STATELESS`). يُعدّ counterpart لـ `device_link_add()` مع `DL_FLAG_STATELESS`.

**Parameters:**
- `link` — الرابط المُراد حذفه

**Return:** void

**Key details:**
- **يُستخدم فقط مع `DL_FLAG_STATELESS` links** — إذا استخدمته مع managed link → WARN
- يُقلّل kref — الحذف الفعلي يحدث عند وصول kref لـ 0
- يحجز `device_links_write_lock()`

---

#### `device_link_remove()`

```c
void device_link_remove(void *consumer, struct device *supplier);
```

يحذف رابط stateless بالبحث عن consumer وsupplier بدلاً من pointer مباشر للرابط.

**Parameters:**
- `consumer` — الـ consumer device (void* لتجنب include issues)
- `supplier` — الـ supplier device

**Return:** void

---

### 6. الـ sysfs Attributes

---

#### `device_create_file()`

```c
int device_create_file(struct device *device,
                       const struct device_attribute *entry);
```

يُضيف ملف attribute لـ sysfs entry الخاص بالـ device.

**Parameters:**
- `device` — الـ device المُستهدفة
- `entry` — الـ attribute المُعرَّفة بـ `DEVICE_ATTR_*` macros

**Return:** 0 نجاح، `-ENODEV` إذا لم تكن الـ device مُسجَّلة، `-EEXIST` إذا كان الملف موجوداً

**Key details:**
- wrapper على `sysfs_create_file(&dev->kobj, &entry->attr)`
- الـ `show`/`store` callbacks تُستدعى من sysfs context مع holding device reference
- استخدم `DEVICE_ATTR_RO/RW/WO` للتعريف السريع

---

#### `device_remove_file()`

```c
void device_remove_file(struct device *dev,
                        const struct device_attribute *attr);
```

يحذف ملف attribute من sysfs. آمن للاستدعاء حتى لو الملف غير موجود.

---

#### `device_remove_file_self()`

```c
bool device_remove_file_self(struct device *dev,
                             const struct device_attribute *attr);
```

يُستخدم عند الرغبة في حذف attribute من داخل الـ `show` أو `store` callback نفسها — يتجنب الـ deadlock.

**Return:** `true` إذا كانت الـ attribute موجودة وتمّ حذفها

**Key details:**
- يستخدم `sysfs_remove_file_self()` التي تتعامل مع الـ in-flight read/write
- الاستخدام الكلاسيكي: `oneshot` attributes تحذف نفسها بعد أول قراءة

---

#### `device_add_groups()`

```c
int __must_check device_add_groups(struct device *dev,
                                   const struct attribute_group **groups);
```

يُضيف مصفوفة من `attribute_group` للـ device دفعة واحدة.

**Parameters:**
- `dev` — الـ device المُستهدفة
- `groups` — مصفوفة NULL-terminated من pointers لـ attribute_groups

**Return:** 0 نجاح، errno عند الفشل مع rollback تلقائي لكل ما أُضيف

**Key details:**
- يُكرّر على المصفوفة ويستدعي `sysfs_create_group()` على كل عنصر
- عند فشل أي group، يُزيل كل الـ groups السابقة التي أُضيفت

---

### 7. Naming وتغيير الـ Topology

---

#### `dev_set_name()`

```c
__printf(2, 3) int dev_set_name(struct device *dev, const char *name, ...);
```

يُعيّن اسم الـ device بصيغة printf. يجب استدعاؤه قبل `device_add()`.

**Parameters:**
- `dev` — الـ device المُستهدفة
- `name, ...` — format string وarguments (مثلاً `"i2c-%d", 0`)

**Return:** 0 نجاح، errno عند فشل memory allocation

**Key details:**
- wrapper على `kobject_set_name_vargs()`
- يُخصّص memory للاسم — محتاج `put_device()` لتحريرها
- بعد `device_add()`، استخدم `device_rename()` للتغيير

---

#### `device_rename()`

```c
int device_rename(struct device *dev, const char *new_name);
```

يُعيد تسمية device **مُسجَّلة** مع تحديث sysfs وكل الـ symlinks.

**Parameters:**
- `dev` — device مُسجَّلة
- `new_name` — الاسم الجديد

**Return:** 0 نجاح، errno عند الفشل

**Key details:**
- يحجز device lock داخلياً
- يُحدّث `kobject` name وينقل الـ sysfs directory
- يُحدّث كل الـ symlinks التي تُشير للـ device
- عملية ثقيلة نسبياً — تجنّب استخدامها في hot paths

---

#### `device_move()`

```c
int device_move(struct device *dev, struct device *new_parent,
                enum dpm_order dpm_order);
```

ينقل device لـ parent جديد في الـ device tree مع تحديث sysfs والـ PM order.

**Parameters:**
- `dev` — الـ device المُراد نقلها
- `new_parent` — الـ parent الجديد (NULL = top level)
- `dpm_order` — ترتيب PM عند النقل:
  - `DPM_ORDER_NONE` — لا تغيير في الترتيب
  - `DPM_ORDER_DEV_AFTER_PARENT` — الـ device بعد الـ parent في قائمة PM
  - `DPM_ORDER_PARENT_BEFORE_DEV` — الـ parent قبل الـ device
  - `DPM_ORDER_DEV_LAST` — الـ device في نهاية قائمة PM

**Return:** 0 نجاح، errno عند الفشل

**Key details:**
- يحجز `lock_device_hotplug()` ومن ثم device locks
- يُطلق `KOBJ_MOVE` uevent لإعلام udevd

---

### 8. Firmware Node Management

---

#### `set_primary_fwnode()`

```c
void set_primary_fwnode(struct device *dev, struct fwnode_handle *fwnode);
```

يُعيّن الـ primary firmware node للـ device — هو الـ `fwnode` الرئيسي الذي يُمثّل الـ device في firmware (DT أو ACPI).

**Parameters:**
- `dev` — الـ device المُستهدفة
- `fwnode` — الـ fwnode الجديد (يمكن أن يكون NULL لإزالة الحالي)

**Return:** void

**Key details:**
- يُحدّث `dev->fwnode` وإذا كان هناك secondary fwnode يُدمجهم في linked list
- يستخدمه عادةً platform code وDT parsing code

---

#### `get_dev_from_fwnode()`

```c
struct device *get_dev_from_fwnode(struct fwnode_handle *fwnode);
```

يجد الـ device المُرتبطة بـ fwnode handle معين.

**Parameters:**
- `fwnode` — الـ fwnode handle المُراد البحث عنه

**Return:** `struct device *` مع refcount مُزاد، أو NULL — **يجب `put_device()`**

**Key details:**
- wrapper على `fwnode->ops->get_dev(fwnode)`
- مفيد في cross-device lookups عبر firmware descriptions

---

### 9. Device Locking

الـ **device mutex** (`dev->mutex`) يحمي:
- driver binding/unbinding
- sysfs attribute access من الـ driver
- state transitions

---

#### `device_lock()` / `device_unlock()`

```c
static inline void device_lock(struct device *dev);
static inline void device_unlock(struct device *dev);
```

يحجز/يحرر الـ device mutex.

**Key details:**
- يجب حجزه قبل أي وصول لـ `dev->driver`
- `DEFINE_GUARD(device, ...)` متاح للاستخدام مع `guard(device)` scope-based locking
- الـ probe() وremove() callbacks تُستدعى مع هذا الـ lock محجوزاً

---

#### `device_lock_interruptible()`

```c
static inline int device_lock_interruptible(struct device *dev);
```

يحجز الـ mutex بشكل قابل للمقاطعة بـ signal.

**Return:** 0 نجاح، `-EINTR` إذا جاء signal قبل الحجز

**Key details:**
- مفيد في code المتفاعل مع userspace حيث يجب الاستجابة للـ signals

---

#### `lock_device_hotplug()` / `unlock_device_hotplug()`

```c
void lock_device_hotplug(void);
void unlock_device_hotplug(void);
```

يحجز/يحرر الـ **global hotplug lock** الذي يحمي device add/remove operations على مستوى النظام.

**Key details:**
- هو `device_hotplug_lock` mutex عالمي
- يُحجز تلقائياً أثناء `device_add()`، `device_del()`، `device_move()`
- `lock_device_hotplug_sysfs()` تُعيد `-EBUSY` إذا فشل التحجيز (للـ sysfs context)

---

### 10. Power Management Helpers

---

#### `device_enable_async_suspend()` / `device_disable_async_suspend()`

```c
static inline void device_enable_async_suspend(struct device *dev);
static inline void device_disable_async_suspend(struct device *dev);
```

تُفعّل/تُعطّل الـ **asynchronous suspend** للـ device — تُحسّن وقت الـ suspend بتوازي العمليات.

**Key details:**
- يجب استدعاؤهما قبل `device_add()` أو في probe() مباشرةً
- **لا تعمل بعد `dev->power.is_prepared = true`** (أثناء suspend)

---

#### `dev_pm_set_driver_flags()`

```c
static inline void dev_pm_set_driver_flags(struct device *dev, u32 flags);
```

يُعيّن PM driver flags التي تُؤثّر على سلوك الـ PM framework مع هذا الـ driver.

**الـ flags الشائعة:**
- `DPM_FLAG_NO_DIRECT_COMPLETE` — لا تُكمل suspend مباشرةً (force full suspend path)
- `DPM_FLAG_SMART_PREPARE` — الـ driver يتعامل مع optimize prepare
- `DPM_FLAG_MAY_SKIP_RESUME` — يمكن تخطي resume في بعض الحالات

---

#### `device_set_pm_not_required()`

```c
static inline void device_set_pm_not_required(struct device *dev);
```

يُعلّم الـ device بأنها لا تحتاج PM callbacks — يُحسّن performance في بعض الـ virtual devices.

**Key details:**
- يُعيّن `dev->power.no_pm = true` و `no_callbacks = true` (إذا `CONFIG_PM`)
- الـ PM framework يتخطّى هذه الـ device في suspend/resume cycles

---

### 11. الـ devres — Managed Resources

الـ **devres** (device resources) نظام يضمن تحرير resources تلقائياً عند unbind الـ driver أو remove الـ device. يُلغي احتياج `remove()` لكود cleanup يدوي.

---

#### `devres_alloc()`

```c
#define devres_alloc(release, size, gfp) \
    __devres_alloc_node(release, size, gfp, NUMA_NO_NODE, #release)
```

يُخصّص block من memory مرتبط بـ release callback. لا يُضيفه لأي device بعد.

**Parameters:**
- `release` — callback تُستدعى عند cleanup: `void release(struct device *dev, void *res)`
- `size` — حجم الـ data المطلوبة (بدون header الـ devres نفسه)
- `gfp` — GFP flags للـ allocation

**Return:** pointer للـ data area (مش لـ devres header)، NULL عند فشل الـ allocation

**Key details:**
- الـ devres header مخفي قبل الـ data pointer المُعاد
- يجب إضافته لـ device بـ `devres_add()` أو تحريره بـ `devres_free()`

---

#### `devres_add()`

```c
void devres_add(struct device *dev, void *res);
```

يُضيف resource لقائمة الـ devres الخاصة بالـ device.

**Parameters:**
- `dev` — الـ device المالكة
- `res` — الـ pointer المُعاد من `devres_alloc()`

**Return:** void

**Key details:**
- يحجز `dev->devres_lock` (spinlock) أثناء الإضافة
- بعد الإضافة، سيُحرَّر تلقائياً عند `device_del()` أو يدوياً بـ `devres_destroy()`

---

#### `devm_kmalloc()` / `devm_kzalloc()`

```c
void *devm_kmalloc(struct device *dev, size_t size, gfp_t gfp);
static inline void *devm_kzalloc(struct device *dev, size_t size, gfp_t gfp);
```

يُخصّص memory مرتبطة بـ device — تُحرَّر تلقائياً عند driver detach.

**Parameters:**
- `dev` — الـ device المالكة
- `size` — حجم الـ allocation
- `gfp` — GFP flags

**Return:** pointer للـ memory، أو NULL عند الفشل

**Key details:**
- `devm_kzalloc` = `devm_kmalloc(..., gfp | __GFP_ZERO)`
- داخلياً: `devres_alloc()` + copy + `devres_add()`
- **يمكن تحريرها يدوياً بـ `devm_kfree()`** إذا احتجت

---

#### `devm_ioremap_resource()`

```c
void __iomem *devm_ioremap_resource(struct device *dev, const struct resource *res);
```

يُنفّذ `ioremap()` على resource مع تسجيل cleanup تلقائي.

**Parameters:**
- `dev` — الـ device المالكة
- `res` — الـ I/O memory resource (عادةً من `platform_get_resource()`)

**Return:** virtual address مُعيَّن (void __iomem *)، أو `ERR_PTR(-errno)` عند الفشل

**Key details:**
- يطلب الـ region بـ `request_mem_region()` أولاً
- تُحرَّر الـ mapping وتُطلق الـ region تلقائياً عند driver remove
- الـ `_wc` variant تستخدم write-combining mapping

---

#### `devm_add_action()` / `devm_add_action_or_reset()`

```c
#define devm_add_action(dev, action, data) ...
#define devm_add_action_or_reset(dev, action, data) ...
```

يُضيف custom cleanup function للـ devres stack.

**Parameters:**
- `dev` — الـ device المالكة
- `action` — `void action(void *data)` — تُستدعى عند cleanup
- `data` — pointer يُمرَّر لـ `action`

**Return:** 0 نجاح، errno عند فشل alloc

**الفرق بين النسختين:**
- `devm_add_action()`: عند فشل الإضافة تُعيد error بدون تنفيذ action
- `devm_add_action_or_reset()`: عند فشل الإضافة تُنفّذ `action(data)` فوراً ثم تُعيد error — **الأكثر استخداماً** لأنه يضمن cleanup حتى في error paths

**مثال واقعي:**
```c
// في probe():
clk = clk_get(dev, "mclk");
devm_add_action_or_reset(dev, (void(*)(void*))clk_put, clk);
// عند remove() أو probe failure: clk_put(clk) تُستدعى تلقائياً
```

---

#### `devres_open_group()` / `devres_release_group()`

```c
void *devres_open_group(struct device *dev, void *id, gfp_t gfp);
void devres_close_group(struct device *dev, void *id);
int devres_release_group(struct device *dev, void *id);
```

تُجمّع devres entries في **group** يمكن release-ه أو remove-ه دفعة واحدة.

**الاستخدام الكلاسيكي في probe() مع multiple sub-systems:**
```c
/* open group to track sub-system A resources */
id = devres_open_group(dev, NULL, GFP_KERNEL);
ret = setup_subsystem_a(dev);  // uses devm_ calls inside
if (ret) {
    devres_release_group(dev, id);  // undo everything in group
    return ret;
}
devres_close_group(dev, id);
```

---

### 12. Bus وDriver Registration

---

#### `bus_register()`

```c
int __must_check bus_register(const struct bus_type *bus);
```

يُسجّل bus type جديد — ينشئ `/sys/bus/<name>/` وهياكل الـ drivers وdevices.

**Parameters:**
- `bus` — الـ bus_type struct مع name وcallbacks معيّنة

**Return:** 0 نجاح، errno عند الفشل

**Key details:**
- يُنشئ `bus->p` (subsys_private) داخلياً
- يُنشئ klist للـ devices وklist للـ drivers
- يُنشئ sysfs entries: `/sys/bus/<name>/devices/` و `/sys/bus/<name>/drivers/`
- يُسجَّل مع `/sys/bus/<name>/uevent` لأحداث الـ hotplug

---

#### `driver_register()`

```c
int __must_check driver_register(struct device_driver *drv);
```

يُسجّل driver في bus الخاص به ويُحاول match مع الـ devices الموجودة.

**Parameters:**
- `drv` — الـ driver struct مع name وbus وprobe معيّنة

**Return:** 0 نجاح، `-EBUSY` إذا كان مُسجَّلاً بالفعل، errno أخرى

**Key details:**
- يستدعي `bus_add_driver()` → يضيف لـ bus's klist_drivers
- يستدعي `driver_attach()` تلقائياً لتوصيله بالـ devices الموجودة
- ينشئ `/sys/bus/<name>/drivers/<drv_name>/` في sysfs
- `module_driver()` macro يُبسّط `module_init/module_exit` pattern

---

#### `subsys_interface_register()`

```c
int subsys_interface_register(struct subsys_interface *sif);
```

يُسجّل interface على subsystem — يستدعي `sif->add_dev` على كل device موجودة حالياً.

**Parameters:**
- `sif` — الـ interface مع `subsys`، `add_dev`، `remove_dev` معيّنة

**Return:** 0 نجاح، errno عند الفشل

**Key details:**
- يضمن استدعاء `add_dev` على كل device حالية دفعة واحدة عند التسجيل
- عند تسجيل device جديدة لاحقاً، الـ bus core يستدعي `add_dev` عليها تلقائياً
- مثال: `cpu_cooling` interface يُسجّل على `cpu` subsystem

---

### 13. Online/Offline والـ Hotplug

---

#### `device_offline()` / `device_online()`

```c
int device_offline(struct device *dev);
int device_online(struct device *dev);
```

يُحوّل device لـ offline state (تهيئة للـ hot-remove) أو يُعيدها online.

**Parameters:**
- `dev` — الـ device المُستهدفة

**Return:** 0 نجاح، errno عند الفشل (مثلاً إذا لم يدعم الـ bus `offline`)

**Key details:**
- يستدعي `dev->bus->offline(dev)` / `dev->bus->online(dev)`
- `device_supports_offline()` تتحقق إذا كان الـ bus يدعم هذا
- يُطلق uevent `KOBJ_OFFLINE` / `KOBJ_ONLINE`
- يحجز `lock_device_hotplug()` أثناء العملية

---

### 14. الـ Macros والـ Helpers الصغيرة

---

#### `kobj_to_dev()`

```c
#define kobj_to_dev(__kobj) container_of_const(__kobj, struct device, kobj)
```

يُحوّل pointer لـ `kobject` لـ `struct device *`. الاستخدام الكلاسيكي في `kobject` show/store callbacks.

---

#### `dev_get_drvdata()` / `dev_set_drvdata()`

```c
static inline void *dev_get_drvdata(const struct device *dev);
static inline void dev_set_drvdata(struct device *dev, void *data);
```

يُخزّن/يسترجع الـ driver-private data pointer من `dev->driver_data`.

**Key details:**
- هذا هو **الطريقة الصحيحة الوحيدة** لتخزين driver state — لا تضع state في global variables
- يُعيَّن في probe(): `dev_set_drvdata(dev, priv)`
- يُسترجع في كل callback: `priv = dev_get_drvdata(dev)`

---

#### `dev_get_platdata()`

```c
static inline void *dev_get_platdata(const struct device *dev);
```

يُعيد `dev->platform_data` — البيانات الخاصة بالـ board التي تُمرَّر لـ driver من board file أو DT parsing.

---

#### `device_is_registered()`

```c
static inline int device_is_registered(struct device *dev);
```

يُعيد true إذا كانت الـ device مُضافة لـ sysfs (`kobj.state_in_sysfs`).

---

#### `dev_has_sync_state()` / `dev_set_drv_sync_state()`

```c
static inline bool dev_has_sync_state(struct device *dev);
static inline int dev_set_drv_sync_state(struct device *dev,
                                          void (*fn)(struct device *dev));
```

يتعاملان مع الـ **sync_state** callback — يُستدعى عندما تُصبح جميع consumers مرتبطة بـ drivers، يُستخدم مثلاً في clk وregulator drivers لتنظيف hardware state تركه الـ bootloader.

---

#### `dev_to_node()` / `set_dev_node()`

```c
static inline int dev_to_node(struct device *dev);
static inline void set_dev_node(struct device *dev, int node);
```

يقرأ/يُعيّن الـ NUMA node للـ device. مهم لـ memory allocation performance في NUMA systems — استخدم `dev_to_node(dev)` في `kmalloc_node()` calls.

---

#### `device_iommu_mapped()`

```c
static inline bool device_iommu_mapped(struct device *dev);
```

يُعيد true إذا كانت الـ device تحت IOMMU group — تعني أن الـ DMA transactions تمر عبر IOMMU translation.

---

#### `DEVICE_ATTR_*` Macros

```c
DEVICE_ATTR(_name, _mode, _show, _store)     // general
DEVICE_ATTR_RO(_name)                        // 0444, <name>_show
DEVICE_ATTR_RW(_name)                        // 0644, <name>_show + <name>_store
DEVICE_ATTR_WO(_name)                        // 0200, <name>_store
DEVICE_ATTR_ADMIN_RO(_name)                  // 0400
DEVICE_ATTR_ADMIN_RW(_name)                  // 0600
DEVICE_ATTR_PREALLOC(_name, _mode, _show, _store)  // SYSFS_PREALLOC
```

تُعرّف `struct device_attribute dev_attr_<name>`. الاستخدام:
```c
/* تعريف الـ attribute */
static ssize_t power_state_show(struct device *dev,
                                 struct device_attribute *attr, char *buf)
{
    return sprintf(buf, "%d\n", get_power_state(dev));
}
DEVICE_ATTR_RO(power_state);   // → dev_attr_power_state

/* الإضافة في probe() */
device_create_file(dev, &dev_attr_power_state);
```

---

#### `DEVICE_ULONG_ATTR` / `DEVICE_INT_ATTR` / `DEVICE_BOOL_ATTR`

```c
DEVICE_ULONG_ATTR(_name, _mode, _var)
DEVICE_INT_ATTR(_name, _mode, _var)
DEVICE_BOOL_ATTR(_name, _mode, _var)
DEVICE_STRING_ATTR_RO(_name, _mode, _var)
```

تُعرّف `struct dev_ext_attribute` مع automatic show/store handlers تقرأ/تكتب على `_var` مباشرةً.

```c
static unsigned long tx_count;
DEVICE_ULONG_ATTR(tx_count, 0444, tx_count);
// قراءة من sysfs تُعيد قيمة tx_count مباشرةً
```

---

#### `module_driver()` / `builtin_driver()`

```c
#define module_driver(__driver, __register, __unregister, ...)
#define builtin_driver(__driver, __register, ...)
```

يُلغيان الحاجة لكتابة `module_init/module_exit` يدوياً.

```c
/* بدون macro */
static int __init my_driver_init(void) { return platform_driver_register(&my_drv); }
static void __exit my_driver_exit(void) { platform_driver_unregister(&my_drv); }
module_init(my_driver_init);
module_exit(my_driver_exit);

/* مع macro */
module_platform_driver(my_drv);
// الذي يتوسّع لـ: module_driver(my_drv, platform_driver_register, platform_driver_unregister)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ driver model مش بيعتمد على debugfs بشكل مباشر، لكن subsystems كتير بتعمل entries فيه.

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# أهم entries عامة للـ device model
ls /sys/kernel/debug/devices/
ls /sys/kernel/debug/driver_probe_defer   # الـ devices اللي latch على defer probe

# قراءة الـ deferred probe list — بيديك أسماء الـ devices اللي فشلت تـ probe وبتستنى
cat /sys/kernel/debug/devices_deferred    # kernel >= 5.10
```

```bash
# لو الـ device في bus معين (مثلاً platform)
ls /sys/kernel/debug/regulator/           # energy model / pm domain dependencies
ls /sys/kernel/debug/pinctrl/            # pin control state لكل device
ls /sys/kernel/debug/pm_genpd/           # power domain tree
cat /sys/kernel/debug/pm_genpd/summary   # ملخص كامل لحالة كل power domain
```

---

#### 2. مدخلات الـ sysfs

كل `struct device` بيتحول لـ directory في sysfs. دي أهم المسارات:

```
/sys/devices/                        ← شجرة كل الـ devices
/sys/bus/<bus>/devices/              ← devices على bus معين
/sys/bus/<bus>/drivers/              ← drivers مسجلة على bus
/sys/bus/<bus>/drivers_autoprobe     ← 1 = autoprobe مفعّل
/sys/bus/<bus>/drivers_probe         ← اكتب اسم device لـ probe يدوي
/sys/class/<class>/                  ← devices مصنفة بالـ class
```

**الـ attributes الأساسية لكل device:**

| Sysfs Attribute | المعنى | قراءة / كتابة |
|---|---|---|
| `uevent` | إرسال uevent يدوي | write |
| `power/control` | `auto` أو `on` — runtime PM | R/W |
| `power/runtime_status` | `active` / `suspended` / `error` | R |
| `power/runtime_suspended_time` | وقت الـ suspend بالـ ms | R |
| `power/runtime_active_time` | وقت الـ active | R |
| `driver` | symlink للـ driver الحالي | R |
| `driver_override` | إجبار driver بعينه | R/W |
| `modalias` | string تعريف الجهاز | R |
| `remove` | اكتب `1` لإزالة الجهاز | W |
| `online` / `offline` | toggle online state | R/W |
| `removable` | `fixed` / `removable` / `unknown` | R |
| `of_node` | symlink لـ DT node | R |
| `physical_location/*` | موقع الـ device في الـ chassis | R |

```bash
# قراءة حالة runtime PM
cat /sys/devices/platform/my-device/power/runtime_status

# قراءة اسم الـ driver الحالي
readlink /sys/devices/platform/my-device/driver

# عرض كل الـ device links (supplier/consumer)
ls /sys/bus/platform/devices/my-device/supplier:*/
ls /sys/bus/platform/devices/my-device/consumer:*/

# عرض حالة الـ device link
cat /sys/bus/platform/devices/my-device/supplier:platform:other-device/status
```

**الـ device link states المحتملة في sysfs:**

| Status Value | المعنى |
|---|---|
| `not tracked` | DL_STATE_NONE |
| `dormant` | DL_STATE_DORMANT |
| `available` | DL_STATE_AVAILABLE |
| `consumer probing` | DL_STATE_CONSUMER_PROBE |
| `active` | DL_STATE_ACTIVE |
| `supplier unbinding` | DL_STATE_SUPPLIER_UNBIND |

---

#### 3. استخدام الـ ftrace

```bash
# تفعيل tracing لكل events الـ device model
cd /sys/kernel/debug/tracing

# عرض الـ events المتاحة المرتبطة بـ device و driver
grep -r "devres\|driver\|device" available_events | grep -v "#"

# أهم events:
# driver:driver_probe_start
# driver:driver_probe_end
# driver:driver_probe_err

# تفعيل probe tracing
echo 1 > events/driver/enable

# أو بشكل أكثر دقة
echo 1 > events/driver/driver_probe_start/enable
echo 1 > events/driver/driver_probe_end/enable

# تشغيل trace
echo 1 > tracing_on
# ... شغّل الجهاز أو افعل hotplug ...
cat trace
echo 0 > tracing_on
```

```bash
# تتبع function معينة في drivers/base
echo 'really_probe' > set_ftrace_filter
echo 'device_bind_driver' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
cat trace_pipe
```

```bash
# تتبع device_link_add لتشخيص مشاكل supplier/consumer
echo 'device_link_add' > set_ftrace_filter
echo function_graph > current_tracer
echo 1 > tracing_on
```

---

#### 4. تفعيل الـ printk / dynamic debug

```bash
# تفعيل dynamic debug لكل ملفات الـ device core
echo 'file drivers/base/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ probe فقط
echo 'file drivers/base/dd.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug الـ device links
echo 'file drivers/base/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug الـ devres
echo 'file drivers/base/devres.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل شيء مع line numbers و function names
echo 'file drivers/base/*.c +plf' > /sys/kernel/debug/dynamic_debug/control

# عرض الـ rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep drivers/base

# رفع مستوى الـ kernel log level (0-7)
echo 8 > /proc/sys/kernel/printk
```

```bash
# ضبط loglevel عبر kernel cmdline عند boot
# في /etc/default/grub أو في GRUB commandline:
# GRUB_CMDLINE_LINUX="loglevel=8 initcall_debug"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| CONFIG Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_DRIVER` | يفعّل dev_dbg() output من الـ driver core |
| `CONFIG_DEBUG_KOBJECT` | يطبع كل kobject init/add/del في الـ kernel log |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يكتشف use-after-free في kobject release |
| `CONFIG_DEBUG_DEVRES` | يطبع كل devres alloc/free operations |
| `CONFIG_LOCKDEP` | يكتشف deadlock في device_lock/mutex |
| `CONFIG_DEBUG_MUTEXES` | تشخيص شامل لمشاكل الـ mutex |
| `CONFIG_PROVE_LOCKING` | يُثبت correctness الـ locking |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock misuse |
| `CONFIG_KASAN` | يكتشف memory errors في driver allocations |
| `CONFIG_KMSAN` | يكتشف uninitialized memory |
| `CONFIG_UBSAN` | يكتشف undefined behavior |
| `CONFIG_DEBUG_SLAB` | تشخيص slab allocator (kmalloc) |
| `CONFIG_DEBUG_OBJECTS` | يتتبع lifetime الـ objects |
| `CONFIG_FTRACE` | تفعيل function tracing |
| `CONFIG_TRACEPOINTS` | تفعيل tracepoints |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بتفعيل dev_dbg() runtime |
| `CONFIG_PM_DEBUG` | تفعيل power management debugging |
| `CONFIG_PM_SLEEP_DEBUG` | تشخيص system sleep transitions |
| `CONFIG_PM_RUNTIME` | يُفعّل runtime PM infrastructure |

```bash
# التحقق من الـ config الحالي
grep -E 'CONFIG_DEBUG_DRIVER|CONFIG_DEBUG_KOBJECT|CONFIG_DEBUG_DEVRES' /boot/config-$(uname -r)
# أو
zcat /proc/config.gz | grep -E 'CONFIG_DEBUG_DRIVER|CONFIG_DEBUG_KOBJECT'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**أداة `udevadm` لتشخيص uevent و sysfs:**

```bash
# عرض كل attributes لجهاز معين
udevadm info --attribute-walk --path=/sys/devices/platform/my-device

# مراقبة كل uevents في real-time
udevadm monitor --kernel --udev --property

# trigger uevent يدوي لجهاز
udevadm trigger --action=add --sysname=my-device

# test الـ udev rules بدون تطبيق
udevadm test /sys/devices/platform/my-device
```

**أداة `lsmod` و `modinfo`:**

```bash
# التحقق من أن الـ module مُحمّل
lsmod | grep my_driver

# عرض معلومات الـ module والـ aliases
modinfo my_driver
modinfo my_driver | grep alias   # للتطابق مع modalias
```

**أداة `systool` من sysfsutils:**

```bash
# عرض كل attributes الـ bus
systool -v -b platform

# عرض كل attributes جهاز معين
systool -v -d my-device
```

**مراقبة الـ device links:**

```bash
# عرض كل supplier links لجهاز
find /sys/devices -name "supplier:*" -path "*/my-device/*" -ls

# عرض consumer devices لـ supplier معين
find /sys/devices -name "consumer:*" -path "*/my-supplier/*" -ls
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|---|---|---|
| `probe of <dev> failed with error -EPROBE_DEFER` | جهاز مش جاهز (supplier لسه ما اتـ probe) | انتظر أو افحص supplier dependencies |
| `probe of <dev> failed with error -ENOMEM` | نفاد الـ memory أثناء probe | افحص `CONFIG_KASAN`, زود الـ RAM أو قلل allocations |
| `probe of <dev> failed with error -ENODEV` | الـ hardware مش موجود أو الـ DT مش صح | افحص الـ DT و hardware wiring |
| `probe of <dev> failed with error -EINVAL` | parameter خاطئ في probe | افحص الـ DT properties و platform_data |
| `Driver <name> requests probe deferral` | الـ driver صريح في طلب defer | انتظر وافحص لو الـ dependency بتتـ probe |
| `deferred probe pending` في dmesg عند late boot | dependency ما اتـ probe أبداً | افحص missing drivers أو DT errors |
| `kobject_add_internal failed for <name>` | اسم mكرر أو parent غلط | افحص dev_set_name() و parent hierarchy |
| `sysfs: cannot create duplicate filename` | device بنفس الاسم registered مرتين | الـ driver بـ probe مرتين أو اسم غلط |
| `device_add called multiple times` | race condition في init | افحص الـ locking في الـ driver |
| `Release called without put_device` | memory leak في refcount | افحص كل مسارات get_device/put_device |
| `WARNING: CPU:X PID:X at drivers/base/core.c` | WARN_ON في الـ device core | اقرأ الـ stack trace وافحص السطر |
| `BUG: sleeping function called from invalid context` | device_lock() في interrupt context | الـ driver بيحاول يـ lock الـ device من ISR |
| `unregistered device while device locked` | device_del() وهو locked | افحص الترتيب، الـ lock لازم يتـ release قبل del |
| `Trying to vfree() nonexistent vm area` | devres free خاطئ | افحص devm_kzalloc وأقاربها |
| `WARNING: refcount_t: underflow` | put_device() زيادة عن get_device() | اتبع الـ refcount lifecycle |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

ضع `WARN_ON()` في النقاط دي عشان تكتشف bugs مبكراً:

```c
/* في بداية الـ probe — تأكد إن الـ device مش bound */
static int my_driver_probe(struct device *dev)
{
    WARN_ON(device_is_bound(dev)); /* shouldn't be bound yet */
    WARN_ON(!dev->of_node);       /* DT node must exist */
    ...
}
```

```c
/* عند استخدام dev_get_drvdata — تأكد إن مش NULL */
static irqreturn_t my_irq_handler(int irq, void *data)
{
    struct my_priv *priv = data;
    WARN_ON(!priv);               /* should never be NULL */
    ...
}
```

```c
/* عند الـ remove — تأكد إن الـ resources اتـ cleanup */
static void my_driver_remove(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    WARN_ON(priv->irq_enabled);   /* IRQ should be disabled before remove */
    ...
}
```

```c
/* تشخيص مشاكل الـ device_link state */
static int supplier_probe(struct device *dev)
{
    struct device_link *link;
    link = device_link_add(consumer_dev, dev, DL_FLAG_MANAGED | DL_FLAG_PM_RUNTIME);
    WARN_ON(!link);
    WARN_ON(link->status != DL_STATE_AVAILABLE);
    ...
}
```

```c
/* dump_stack() في situations غير متوقعة */
static int my_bus_probe(struct device *dev)
{
    if (!dev->driver) {
        dev_err(dev, "probe called without driver binding\n");
        dump_stack(); /* تسجيل الـ call trace كاملاً */
        return -EINVAL;
    }
    ...
}
```

---

### Hardware Level

---

#### 1. التحقق إن hardware state يطابق kernel state

```bash
# تحقق من إن الجهاز مرئي للـ kernel
ls /sys/devices/platform/                    # platform devices
ls /sys/bus/i2c/devices/                     # I2C devices
ls /sys/bus/spi/devices/                     # SPI devices

# تحقق من bind الـ driver
cat /sys/bus/platform/devices/my-device/driver  # فاضي = مش bound

# تحقق من الـ power state
cat /sys/devices/platform/my-device/power/runtime_status
# expected: "active" أو "suspended"

# تحقق من الـ clock state لو الجهاز clock-dependent
cat /sys/kernel/debug/clk/my-device-clk/clk_enable_count
cat /sys/kernel/debug/clk/my-device-clk/clk_rate

# تحقق من الـ regulator state
cat /sys/kernel/debug/regulator/my-supply/state      # "enabled" / "disabled"
cat /sys/kernel/debug/regulator/my-supply/microvolts # الجهد الفعلي
```

---

#### 2. تقنيات الـ Register Dump

```bash
# قراءة register بعينه عبر /dev/mem (يحتاج CONFIG_STRICT_DEVMEM=n)
devmem2 0xFE200000 w       # قراءة 32-bit من العنوان

# قراءة range من الـ registers
for addr in $(seq 0xFE200000 4 0xFE2000FF); do
    printf "0x%08x: 0x%08x\n" $addr $(devmem2 $addr w 2>/dev/null | awk '/^0x/{print $NF}')
done

# عبر /dev/mem مباشرة بـ dd
dd if=/dev/mem bs=4 count=64 skip=$((0xFE200000/4)) 2>/dev/null | xxd

# io utility (من package iotools)
io -4 -r 0xFE200000     # read 32-bit
io -4 -w 0xFE200000 0x1 # write 32-bit

# لـ PCI devices — قراءة config space
lspci -xxx -s 00:02.0    # config space dump
setpci -s 00:02.0 0x10.l # قراءة BAR0
```

```bash
# لو الجهاز في memory mapped IO — استخدام /proc/iomem
cat /proc/iomem | grep -i "my-device"
# يديك العنوان الفيزيائي المعين للجهاز

# التحقق من الـ resources المخصصة للجهاز
cat /sys/devices/platform/my-device/resources 2>/dev/null
# أو
cat /proc/ioports  # لـ IO-mapped devices
```

---

#### 3. نصائح Logic Analyzer / Oscilloscope

```
+------------------+     +------------------+     +------------------+
|   Logic Analyzer  |     |   Oscilloscope   |     |   Combined Tool  |
|  (Protocol Decode)|     |  (Signal Quality)|     |  (Both Worlds)   |
+------------------+     +------------------+     +------------------+
        |                         |                         |
   I2C / SPI / UART          VDD / CLK / RST          Sigrok / DSLogic
   timing violations          voltage levels             + PulseView
   ACK/NAK patterns           signal integrity
```

**نقاط probe مهمة:**

| Signal | ما تبحث عنه |
|---|---|
| RST / RESETN | يجب أن يكون active (low/high حسب الجهاز) قبل probe |
| VDD / AVDD | stable قبل kernel probe بـ Tstart المطلوب في datasheet |
| INT / IRQ | لا يُطلَق قبل الـ driver يعمل request_irq |
| SCL / SDA (I2C) | ACK بعد كل byte، ما فيش NACK في initialization |
| CS / SCK (SPI) | polarity صح حسب CPOL/CPHA في الـ DT |
| EN (enable) | الـ enable line يُفعَّل في الترتيب الصح |

```bash
# Sigrok command-line capture لـ I2C bus
sigrok-cli -d fx2lafw --config samplerate=1MHz \
  --channels D0=SCL,D1=SDA \
  --protocol-decoders i2c \
  -o capture.sr

# تحليل الـ capture
pulseview capture.sr
```

---

#### 4. مشاكل Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في dmesg | التشخيص |
|---|---|---|
| الـ device مش موجود فيزيائياً | `probe of xxx failed with error -ENODEV` | افحص الـ power و wiring |
| الـ voltage مش كافي | `probe failed -ETIMEDOUT` + أخطاء I2C/SPI | قِس VDD بـ multimeter |
| الـ reset مش طُلق | device يرد بـ garbage أو timeout | تحقق من RST line polarity في DT |
| interrupt storm | `irq X: nobody cared` أو `do_IRQ: x.Y No irqhandler` | افحص IRQ trigger type (edge/level) |
| DMA misalignment | `DMA-API: device driver has pending DMA allocations` | افحص الـ dma_mask و alignment |
| clock مش مفعَّل | `clk: failed to enable clock` في probe | افحص الـ clock tree و DT |
| regulator مش موجود | `regulator_get: my-supply supply not found` | افحص الـ DT regulator references |
| I2C NAK | `i2c i2c-X: sendbytes: NAK bailout.` | عنوان الجهاز غلط أو الجهاز مش موجود |

---

#### 5. تشخيص الـ Device Tree

```bash
# عرض الـ DT compiled (runtime)
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | less

# مقارنة DT المُستخدم مع الـ hardware datasheet
# عرض node معين
cat /sys/firmware/devicetree/base/soc/my-device/compatible
cat /sys/firmware/devicetree/base/soc/my-device/reg
cat /sys/firmware/devicetree/base/soc/my-device/status

# التحقق من الـ OF matching
# الـ compatible string في DT لازم تطابق of_match_table في الـ driver
grep -r "my,device-v1" /sys/bus/platform/drivers/

# التحقق من phandle resolution
fdtget /boot/dtbs/$(uname -r)/my-board.dtb /soc/my-device clocks
fdtget /boot/dtbs/$(uname -r)/my-board.dtb /soc/my-device reg

# استخدام of_find_node_by_name في runtime عبر debugfs
ls /sys/firmware/devicetree/base/
find /sys/firmware/devicetree/base/ -name "compatible" -exec grep -l "my,device" {} \;
```

```bash
# التحقق من إن fwnode_handle و of_node متصلين صح
# (من kernel source, device->of_node و device->fwnode)
# في runtime:
cat /sys/devices/platform/my-device/of_node  # symlink لـ DT node
# لو الـ symlink فاضي أو غلط = مشكلة في DT matching
```

**Overlay debugging:**

```bash
# تطبيق DT overlay في runtime
mkdir /sys/kernel/config/device-tree/overlays/test
cp my-overlay.dtbo /sys/kernel/config/device-tree/overlays/test/dtbo
echo 1 > /sys/kernel/config/device-tree/overlays/test/status

# التحقق من نجاح الـ overlay
cat /sys/kernel/config/device-tree/overlays/test/status  # "applied"
dmesg | tail -20  # افحص errors
```

---

### Practical Commands

---

#### الأوامر الجاهزة

**1. تشخيص probe failure شامل:**

```bash
#!/bin/bash
# comprehensive device probe diagnosis
DEVICE="my-device"  # ← عدّل الاسم

echo "=== Device Status ==="
ls /sys/bus/platform/devices/ | grep $DEVICE

echo "=== Driver Binding ==="
readlink /sys/bus/platform/devices/$DEVICE/driver 2>/dev/null || echo "NOT BOUND"

echo "=== Power State ==="
cat /sys/bus/platform/devices/$DEVICE/power/runtime_status 2>/dev/null

echo "=== Recent dmesg for device ==="
dmesg | grep -i "$DEVICE" | tail -30

echo "=== Deferred probes ==="
cat /sys/kernel/debug/devices_deferred 2>/dev/null
```

**2. تفعيل full debug trace لـ probe:**

```bash
# تشغيل trace لـ driver probe
echo 'really_probe:traceon' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# trigger re-probe
echo -n > /sys/bus/platform/drivers/my-driver/unbind 2>/dev/null
echo my-device > /sys/bus/platform/drivers/my-driver/bind 2>/dev/null

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace | head -100
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**3. فحص device links سريع:**

```bash
# عرض كل device links في النظام
find /sys/devices -name "supplier:*" -o -name "consumer:*" 2>/dev/null | \
  while read link; do
    echo "$link -> $(readlink $link)"
    echo "  status: $(cat ${link}/status 2>/dev/null)"
  done
```

**4. dump كل sysfs attributes لـ device:**

```bash
DEV_PATH="/sys/devices/platform/my-device"
find $DEV_PATH -maxdepth 2 -type f -readable | while read f; do
    echo "--- $f ---"
    cat "$f" 2>/dev/null
    echo ""
done
```

**5. مراقبة uevents real-time:**

```bash
udevadm monitor --kernel --property 2>&1 | grep -A 10 "my-device"
```

---

#### مثال عملي: تشخيص `probe defer` loop

**الموقف:** جهاز `my-sensor` بيظهر دايماً `-EPROBE_DEFER` ومش بيـ probe.

```bash
# الخطوة 1: شوف مين بيتـ defer
dmesg | grep "EPROBE_DEFER\|deferred"
# مثال output:
# [    4.123] my-sensor: probe of my-sensor failed with error -517
# [    4.124] my-sensor: Added to deferred list

# الخطوة 2: افحص الـ deferred list
cat /sys/kernel/debug/devices_deferred
# output:
# my-sensor        Driver my-sensor requests probe deferral

# الخطوة 3: افحص الـ supplier links
ls /sys/bus/platform/devices/my-sensor/supplier:*/
# output: supplier:platform:my-regulator

# الخطوة 4: افحص حالة الـ supplier
cat /sys/bus/platform/devices/my-regulator/power/runtime_status
# output: suspended   ← المشكلة! الـ regulator مش available

# الخطوة 5: افحص لو الـ regulator driver موجود
readlink /sys/bus/platform/devices/my-regulator/driver
# output: (empty) ← driver مش bound!

# الحل: حمّل regulator driver أو افحص DT compatible string
modprobe my-regulator-driver
# أو
dmesg | grep "my-regulator" | grep "compatible\|no match"
```

**مثال output من ftrace أثناء probe:**

```
# tracer: function
#
# TASK-PID   CPU#    TIMESTAMP  FUNCTION
  kworker-123 [001]   4.100:  really_probe <-- my_bus_probe
  kworker-123 [001]   4.101:    devm_regulator_get <-- my_driver_probe
  kworker-123 [001]   4.102:      regulator_get <-- devm_regulator_get
  kworker-123 [001]   4.103:        _regulator_get.isra.0 <-- regulator_get
  kworker-123 [001]   4.104:  driver_deferred_probe_add <-- really_probe
  # ← الـ regulator_get طلّع NULL → defer
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Driver لـ SPI على RK3562 بييجي -EPROBE_DEFER مش بيخلص

#### العنوان
**Industrial Gateway على RK3562 — SPI flash driver عالق في deferred probe loop**

#### السياق
شركة بتعمل industrial gateway بيستخدم RK3562 كـ main SoC. الـ gateway بيتعامل مع Modbus/RS-485 عبر SPI-connected microcontroller. أثناء bring-up، الـ SPI flash driver بيرجع `-EPROBE_DEFER` في كل boot وما بيكمل probe أبدًا، مع إن الـ hardware موجود وشتغل تمام في downstream kernel قديم.

#### المشكلة
الـ driver بيعمل `device_link_add()` على supplier device (clk controller) قبل ما الـ supplier يكون bound لـ driver. الـ link بـ `DL_FLAG_PM_RUNTIME | DL_FLAG_AUTOREMOVE_CONSUMER`. بعد late_initcall، الـ `sync_state` على الـ clk supplier ما اتنادى عليه، فالـ consumer ظل في `DL_STATE_CONSUMER_PROBE` وما اكتمل.

```
[    3.821453] spi-rockchip ff1d0000.spi: probe deferred
[    3.821490] spi-rockchip ff1d0000.spi: probe deferred
...
(repeated indefinitely)
```

#### التحليل
في `device.h`، الـ `struct device` بتحمل:

```c
struct dev_links_info links;  /* suppliers + consumers lists */
```

والـ `struct dev_links_info` بتحتوي:

```c
struct dev_links_info {
    struct list_head suppliers;   /* links to supplier devices */
    struct list_head consumers;
    struct list_head defer_sync;  /* hook to global defer_sync list */
    enum dl_dev_state status;
};
```

الـ `enum dl_dev_state` بيحدد الحالة:

```c
enum dl_dev_state {
    DL_DEV_NO_DRIVER = 0,
    DL_DEV_PROBING,          /* <-- consumer stuck here */
    DL_DEV_DRIVER_BOUND,
    DL_DEV_UNBINDING,
};
```

الـ `device_link_state` للـ link نفسه:

```c
enum device_link_state {
    DL_STATE_NONE = -1,
    DL_STATE_DORMANT = 0,
    DL_STATE_AVAILABLE,          /* supplier bound, consumer not yet */
    DL_STATE_CONSUMER_PROBE,     /* <-- link stuck here */
    DL_STATE_ACTIVE,
    DL_STATE_SUPPLIER_UNBIND,
};
```

السبب: الـ clk supplier لديه `sync_state` callback معرَّف في الـ `struct device_driver`، والـ kernel ينتظر كل consumers يكملوا probe قبل ما يستدعي `sync_state`. لأن في consumer تاني (pinctrl) ما اتعرفش على أي driver، الـ `defer_sync` list ما اتصفيش أبدًا.

#### الحل

**أولًا، تشخيص الوضع:**

```bash
# شوف links الـ supplier
cat /sys/kernel/debug/device_component/ff1d0000.spi

# شوف كل deferred devices
cat /sys/kernel/debug/devices_deferred

# شوف sync_state status
cat /sys/bus/platform/devices/ff1d0000.spi/supplier\:ff760000.clock/auto_remove_on
```

**ثانيًا، الـ fix في الـ DT — إضافة الـ pinctrl node الناقص:**

```dts
/* rk3562-gateway.dts */
&spi0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_pins>;   /* كان ناقص */
    cs-gpios = <&gpio2 RK_PB0 GPIO_ACTIVE_LOW>;
};

&pinctrl {
    spi0 {
        spi0_pins: spi0-pins {
            rockchip,pins =
                <2 RK_PA5 2 &pcfg_pull_none>,
                <2 RK_PA6 2 &pcfg_pull_none>,
                <2 RK_PA7 2 &pcfg_pull_none>;
        };
    };
};
```

**ثالثًا، إن مش ممكن تضيف pinctrl، استخدم `DL_FLAG_STATELESS` بدل الـ managed link لتجنب الـ sync_state block:**

```c
/* في الـ driver probe */
link = device_link_add(&consumer->dev, &supplier->dev,
                        DL_FLAG_STATELESS | DL_FLAG_PM_RUNTIME);
/* مع ضرورة التنظيف اليدوي في remove() */
```

#### الدرس المستفاد
**الـ `dev_links_info.defer_sync`** وحالة `DL_STATE_CONSUMER_PROBE` بيفضلوا locked لو حتى consumer واحد من consumers الـ supplier ما اكتملش. لازم تتأكد إن كل consumers الـ supplier موجودين ومتربطين بـ driver، وإلا الـ `sync_state` ما هيتنادى عليه أبدًا ومش بس driver واحد هيتأثر.

---

### السيناريو 2: Memory Leak في Android TV Box على Allwinner H616 — `release` Callback ناقص

#### العنوان
**Android TV Box على H616 — kernel panic بعد USB hotplug متكرر بسبب `struct device` بدون `release`**

#### السياق
منتج TV box بيستخدم Allwinner H616. المنتج بيدعم USB keyboard وmouse hotplug. بعد ~50 cycle من plug/unplug سريع، الـ kernel بيعمل panic بـ:

```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000000
kobject: '(null)' (000000001a2b3c4d): is not initialized, yet kobject_put() is being called.
```

#### المشكلة
الـ custom USB HID driver بيعمل allocate لـ `struct device` embedded في struct خاصة بيه، لكن ما حددش الـ `.release` callback. لما الـ reference count بيوصل لصفر، الـ device core بيحاول يستدعي `dev->release` أو `dev->type->release`، وكلاهم NULL.

#### التحليل
في `device.h`، الـ `struct device` عندها:

```c
void (*release)(struct device *dev);
```

وفي الـ `struct device_type`:

```c
struct device_type {
    const char *name;
    const struct attribute_group **groups;
    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
    char *(*devnode)(const struct device *dev, umode_t *mode,
                     kuid_t *uid, kgid_t *gid);
    void (*release)(struct device *dev);  /* fallback release */
    const struct dev_pm_ops *pm;
};
```

الـ `get_device()` و`put_device()` المُعرَّفتين في نفس الملف:

```c
struct device *get_device(struct device *dev);  /* increments kobj refcount */
void put_device(struct device *dev);            /* decrements; calls release at 0 */
```

الـ `put_device()` بتنادي على `kobject_put(&dev->kobj)` اللي بيتنادي في النهاية على `device_release()` في `drivers/base/core.c`. الـ `device_release()` بيبحث عن release بالترتيب: `dev->release` → `dev->type->release` → `dev->class->devnode`. لو الكل NULL، بيطبع:

```
Device 'xxx' does not have a release() function, it is broken and must be fixed.
```

وبعدين WARN_ON وreturn بدون free. الـ memory بتتسرب، والـ kobject بيبقى في حالة inconsistent.

#### الحل

```c
/* الـ struct الخاصة بالـ driver */
struct my_hid_device {
    struct device dev;       /* embedded — لازم يكون أول field */
    struct usb_device *udev;
    /* ... باقي البيانات */
};

/* الـ release callback الصح */
static void my_hid_device_release(struct device *dev)
{
    struct my_hid_device *hdev =
        container_of(dev, struct my_hid_device, dev);
    kfree(hdev);  /* حرر الـ struct كاملة */
}

/* في الـ probe أو alloc function */
static struct my_hid_device *my_hid_alloc(struct usb_device *udev)
{
    struct my_hid_device *hdev;

    hdev = kzalloc(sizeof(*hdev), GFP_KERNEL);
    if (!hdev)
        return NULL;

    hdev->dev.release = my_hid_device_release; /* لازم يتحدد هنا */
    hdev->dev.parent  = &udev->dev;
    hdev->dev.bus     = &usb_bus_type;
    /* ... */
    device_initialize(&hdev->dev);
    return hdev;
}
```

**للتحقق من leak:**

```bash
# راقب /sys/kernel/debug/kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep -A5 "my_hid"

# شوف reference counts
cat /sys/bus/usb/devices/*/uevent | grep DEVTYPE
```

#### الدرس المستفاد
**كل `struct device` embedded في struct خاصة بالـ driver لازم تحدد `release` callback**. الـ device model في `device.h` صمّم على أساس إن الـ `put_device()` هو المسؤول عن الـ free، مش الـ driver مباشرةً. بدون `release`، الـ memory هتتسرب بصمت في أحسن الأحوال، أو kernel panic في أسوأها.

---

### السيناريو 3: sysfs Attribute Race Condition على STM32MP1 — IoT Sensor Node

#### العنوان
**IoT Sensor Node على STM32MP1 — userspace بيقرأ attribute قبل ما الـ driver يكمل init**

#### السياق
شركة بتبني IoT sensor node بـ STM32MP1 لقياس درجة الحرارة والرطوبة عبر I2C. الـ userspace daemon بيبدأ يقرأ `/sys/bus/i2c/devices/1-0048/temperature` فورًا بعد boot. في بعض الأحيان بييجي قراءة `0` أو garbage values بدل القيمة الصح.

#### المشكلة
الـ driver بيعمل `device_create_file()` لـ `temperature` attribute قبل ما يكمل hardware initialization (I2C transaction + calibration). الـ userspace بيقرأ الـ attribute في خلال الـ window الصغيرة دي.

#### التحليل
الـ `struct device_attribute` في `device.h`:

```c
struct device_attribute {
    struct attribute attr;        /* اسم الـ attribute وـ permissions */
    ssize_t (*show)(struct device *dev, struct device_attribute *attr,
                    char *buf);   /* callback لما حد يقرأ من sysfs */
    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                     const char *buf, size_t count);
};
```

الـ `device_create_file()` بتضيف الـ attribute لـ sysfs على طول، وبعدها أي `read()` على الـ path بيستدعي `show()` بـ `device_lock()` مش مضمون:

```c
/* من device.h */
static inline void device_lock(struct device *dev)
{
    mutex_lock(&dev->mutex);  /* الـ dev->mutex */
}
```

المشكلة: الـ driver بيضيف الـ attribute قبل ما يعمل lock أو يحدد flag يقول "data ready". الـ `show` callback بيرجع بيانات من struct الـ driver من غير ما يتحقق إن الـ init اكتمل.

#### الحل

**أولًا، استخدم `dev->driver_data` عشان تحط flag:**

```c
struct temp_sensor_data {
    struct i2c_client *client;
    s32 last_temp;
    bool data_ready;    /* flag للتحقق في show() */
    struct mutex lock;
};

static ssize_t temperature_show(struct device *dev,
                                  struct device_attribute *attr,
                                  char *buf)
{
    /* dev_get_drvdata() بترجع dev->driver_data */
    struct temp_sensor_data *data = dev_get_drvdata(dev);

    if (!data->data_ready)
        return -EAGAIN;   /* userspace بيحصل على error واضح */

    return sysfs_emit(buf, "%d\n", data->last_temp);
}

static DEVICE_ATTR_RO(temperature);  /* macro من device.h */

static int temp_sensor_probe(struct i2c_client *client)
{
    struct temp_sensor_data *data;
    int ret;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    /* اربط الـ data بالـ device قبل ما تضيف الـ attributes */
    dev_set_drvdata(&client->dev, data);   /* يحدد dev->driver_data */

    /* اعمل hardware init الأول */
    ret = temp_sensor_hw_init(data, client);
    if (ret)
        return ret;

    data->data_ready = true;  /* بعد الـ init بالكامل */

    /* بعدين بس ضيف الـ sysfs attribute */
    return device_create_file(&client->dev, &dev_attr_temperature);
}
```

**ثانيًا، للـ suppression المؤقت أثناء init:**

```c
/* منع uevents لحين اكتمال الـ init */
dev_set_uevent_suppress(&client->dev, 1);
/* ... init ... */
dev_set_uevent_suppress(&client->dev, 0);
kobject_uevent(&client->dev.kobj, KOBJ_CHANGE);
```

#### الدرس المستفاد
**`device_create_file()` بيجعل الـ attribute قابلة للقراءة فورًا**. لازم تضيف attributes بعد ما تكمل hardware init وتحدد `driver_data` بالكامل. استخدم `dev_set_drvdata()` / `dev_get_drvdata()` اللي بيتعاملوا مع `dev->driver_data` مباشرةً، وضيف guard في الـ `show` callback.

---

### السيناريو 4: Device Link وـ Runtime PM على i.MX8 — HDMI Display Pipeline

#### العنوان
**Custom Display Board على i.MX8MQ — HDMI بيتعطل بعد system resume من suspend**

#### السياق
بورد custom مبنية على i.MX8MQ لتطبيق digital signage. الـ HDMI output بيشتغل كويس، لكن بعد `echo mem > /sys/power/state` وـ resume، الـ HDMI ما بيرجع يشتغل. الـ HDMI controller driver (consumer) بيحاول يـ resume قبل الـ PHY supplier.

#### المشكلة
الـ device link بين HDMI controller (consumer) والـ HDMI PHY (supplier) تم إنشاؤه بـ `DL_FLAG_PM_RUNTIME` فقط بدون `DL_FLAG_RPM_ACTIVE`. لما الـ system بيعمل resume، الـ PM core بيصحّي الـ consumer قبل الـ supplier لأن الـ link order مش ضامن system sleep.

#### التحليل
في `device.h`، الـ flags المتاحة للـ device links:

```c
#define DL_FLAG_STATELESS          BIT(0)
#define DL_FLAG_AUTOREMOVE_CONSUMER BIT(1)
#define DL_FLAG_PM_RUNTIME         BIT(2)  /* runtime PM فقط */
#define DL_FLAG_RPM_ACTIVE         BIT(3)  /* اعمل get_sync على supplier */
#define DL_FLAG_AUTOREMOVE_SUPPLIER BIT(4)
#define DL_FLAG_AUTOPROBE_CONSUMER BIT(5)
#define DL_FLAG_MANAGED            BIT(6)
#define DL_FLAG_SYNC_STATE_ONLY    BIT(7)
```

الـ `struct device_link`:

```c
struct device_link {
    struct device *supplier;
    struct list_head s_node;
    struct device *consumer;
    struct list_head c_node;
    struct device link_dev;
    enum device_link_state status;
    u32 flags;
    refcount_t rpm_active;  /* عدد المرات اللي الـ consumer طلب فيها PM get */
    struct kref kref;
    struct work_struct rm_work;
    bool supplier_preactivated;
};
```

المشكلة: `DL_FLAG_PM_RUNTIME` بيضمن بس إن runtime PM يراعي الـ link (supplier لازم يكون active قبل consumer يـ resume في runtime). لكن system sleep/resume بيتحكم فيه `struct dev_pm_info` في `struct device` وبيحتاج الـ supplier يكون في `dev_links_info.status == DL_STATE_ACTIVE` قبل resume الـ consumer.

```c
/* للتأكد من حالة الـ link */
static inline bool device_link_test(const struct device_link *link, u32 flags)
{
    return !!(link->flags & flags);
}
```

#### الحل

**أولًا، أنشئ الـ link بالـ flags الصح:**

```c
/* في HDMI controller probe */
static int imx8_hdmi_probe(struct platform_device *pdev)
{
    struct device *phy_dev;
    struct device_link *link;

    /* اجيب الـ PHY device من الـ of_node */
    phy_dev = get_dev_from_fwnode(
        fwnode_find_reference(dev_fwnode(&pdev->dev), "fsl,hdmi-phy", 0));
    if (!phy_dev)
        return -EPROBE_DEFER;

    /*
     * DL_FLAG_PM_RUNTIME: runtime PM يراعي الـ link
     * DL_FLAG_RPM_ACTIVE: تأكد إن الـ supplier active عند إنشاء الـ link
     * لا تستخدم DL_FLAG_STATELESS عشان الـ core يدير الـ lifecycle
     */
    link = device_link_add(&pdev->dev, phy_dev,
                           DL_FLAG_PM_RUNTIME |
                           DL_FLAG_RPM_ACTIVE  |
                           DL_FLAG_AUTOREMOVE_CONSUMER);
    if (!link) {
        put_device(phy_dev);
        return -EINVAL;
    }

    put_device(phy_dev);
    /* ... باقي الـ init */
}
```

**ثانيًا، تحقق من الـ link state بعد resume:**

```bash
# شوف الـ link state
cat /sys/bus/platform/devices/32c00000.hdmi/supplier\:32c80000.phy/status
# المتوقع: active

# شوف الـ PM state
cat /sys/bus/platform/devices/32c80000.phy/power/runtime_status
# المتوقع: active قبل ما الـ consumer يـ resume
```

**ثالثًا، إضافة system-wide PM order عبر DT:**

```dts
/* imx8mq-signage.dts */
&hdmi {
    power-domains = <&pgc_display>;
    /* الـ display power domain بيضمن supplier يصحى أول */
};
```

#### الدرس المستفاد
**`DL_FLAG_PM_RUNTIME` وحده ما بيضمنش correct resume order في system sleep**. لازم تضيف `DL_FLAG_RPM_ACTIVE` عشان الـ core يعمل `pm_runtime_get_sync()` على الـ supplier لحظة إنشاء الـ link، وبكده الـ status بيبقى `DL_STATE_ACTIVE` اللي بيضمن الترتيب الصح في resume.

---

### السيناريو 5: Custom sysfs Attribute مع `dev_ext_attribute` على AM62x — Automotive ECU

#### العنوان
**Automotive ECU على AM62x — diagnostic tool بيقرأ قيم calibration من sysfs بشكل غلط**

#### السياق
ECU للسيارة مبني على TI AM62x. الـ driver بيـ expose قيم calibration داخلية عبر sysfs عشان diagnostic tool يقدر يقراها ويكتب عليها. المشكلة إن الـ diagnostic tool بيقرأ قيم غلط أحيانًا، خصوصًا لما في calibration update في نفس الوقت.

#### المشكلة
الـ driver استخدم `DEVICE_ULONG_ATTR` لـ expose متغير `unsigned long` مباشرة من غير locking. الـ `device_show_ulong` و`device_store_ulong` بيتعاملوا مع الـ `dev_ext_attribute.var` مباشرةً، ومفيش synchronization.

#### التحليل
في `device.h`:

```c
/* الـ DEVICE_ULONG_ATTR macro */
#define DEVICE_ULONG_ATTR(_name, _mode, _var) \
    struct dev_ext_attribute dev_attr_##_name = \
        { __ATTR(_name, _mode, device_show_ulong, device_store_ulong), &(_var) }

/* الـ struct اللي بتحمل pointer للمتغير مباشرة */
struct dev_ext_attribute {
    struct device_attribute attr;
    void *var;       /* pointer مباشر للمتغير في الـ driver struct */
};
```

الـ `device_show_ulong()` بتعمل:
1. تاخد `dev_ext_attribute` عبر `container_of`
2. تقرأ `*(unsigned long *)eattr->var` مباشرة

مفيش `device_lock()` في الـ path ده بشكل تلقائي. الـ sysfs بيستدعي `show/store` بـ kernel lock على الـ inode، لكن مش بالضرورة بيحمي الـ variable من تعديل concurrent من الـ driver نفسه.

```c
/* device.h — الـ lock المتاح */
static inline void device_lock(struct device *dev)
{
    mutex_lock(&dev->mutex);  /* dev->mutex */
}
```

#### الحل

**أولًا، استخدم `DEVICE_ATTR_RW` مع lock صريح بدل `DEVICE_ULONG_ATTR`:**

```c
struct ecu_calibration {
    struct device dev;
    unsigned long cal_gain;
    unsigned long cal_offset;
    spinlock_t cal_lock;     /* lock خاص بالـ calibration data */
};

static ssize_t cal_gain_show(struct device *dev,
                               struct device_attribute *attr,
                               char *buf)
{
    struct ecu_calibration *ecu = dev_get_drvdata(dev);
    unsigned long val;
    unsigned long flags;

    spin_lock_irqsave(&ecu->cal_lock, flags);
    val = ecu->cal_gain;
    spin_unlock_irqrestore(&ecu->cal_lock, flags);

    return sysfs_emit(buf, "%lu\n", val);
}

static ssize_t cal_gain_store(struct device *dev,
                                struct device_attribute *attr,
                                const char *buf, size_t count)
{
    struct ecu_calibration *ecu = dev_get_drvdata(dev);
    unsigned long val;
    unsigned long flags;
    int ret;

    ret = kstrtoul(buf, 10, &val);
    if (ret)
        return ret;

    spin_lock_irqsave(&ecu->cal_lock, flags);
    ecu->cal_gain = val;
    spin_unlock_irqrestore(&ecu->cal_lock, flags);

    return count;
}

/* استخدم DEVICE_ATTR_RW بدل DEVICE_ULONG_ATTR */
static DEVICE_ATTR_RW(cal_gain);

/* أو لو المتغير read-only للـ diagnostic tool */
static DEVICE_ATTR_ADMIN_RO(cal_gain);  /* mode=0400, root فقط */
```

**ثانيًا، لو الأمان مهم (automotive)، استخدم `DEVICE_ATTR_ADMIN_RW`:**

```c
/* mode=0600 — root فقط يقدر يقرأ ويكتب */
#define DEVICE_ATTR_ADMIN_RW(_name) \
    struct device_attribute dev_attr_##_name = __ATTR_RW_MODE(_name, 0600)

static DEVICE_ATTR_ADMIN_RW(cal_gain);
```

**ثالثًا، تحقق من الـ permissions في sysfs:**

```bash
# تأكد إن الـ permissions صح
ls -la /sys/bus/platform/devices/am62x-ecu.0/cal_gain
# المتوقع: -rw------- root root

# اختبار concurrent access
for i in $(seq 1 1000); do
    cat /sys/bus/platform/devices/am62x-ecu.0/cal_gain &
    echo 12345 > /sys/bus/platform/devices/am62x-ecu.0/cal_gain &
done
wait
```

**رابعًا، استخدم `device_lock` في الـ store لو الـ driver بيستخدم `dev->mutex` بالفعل:**

```c
static ssize_t cal_gain_store(struct device *dev,
                                struct device_attribute *attr,
                                const char *buf, size_t count)
{
    struct ecu_calibration *ecu = dev_get_drvdata(dev);
    unsigned long val;
    int ret;

    ret = kstrtoul(buf, 10, &val);
    if (ret)
        return ret;

    device_lock(dev);          /* mutex_lock(&dev->mutex) */
    ecu->cal_gain = val;
    device_unlock(dev);        /* mutex_unlock(&dev->mutex) */

    return count;
}
```

#### الدرس المستفاد
**`DEVICE_ULONG_ATTR` و`DEVICE_INT_ATTR` و`DEVICE_BOOL_ATTR` بتـ expose المتغير مباشرةً بدون أي locking**. في أي سياق بيكون في concurrent access — خصوصًا في automotive أو industrial اللي فيه hardware interrupts بتعدل نفس المتغير — لازم تبني `show/store` callbacks يدوية مع lock مناسب، أو تضمن إن الـ variable فقط بيتعدل من context واحد.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو أهم مصدر للمقالات التقنية العميقة عن kernel Linux.

| المقال | الوصف |
|--------|-------|
| [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/) | مراجعة شاملة للـ device model، بتغطي `struct device`، buses، classes، و kobjects بأمثلة عملية |
| [Tracking functional dependencies between devices](https://lwn.net/Articles/710494/) | المقال الأساسي اللي شرح فكرة الـ device links (supplier/consumer) قبل ما تتضاف للـ kernel |
| [Functional dependencies between devices](https://lwn.net/Articles/703058/) | نقاش مبكر في الـ mailing list عن ربط الـ devices ببعض عبر dependencies |
| [The managed resource API](https://lwn.net/Articles/222860/) | شرح الـ `devres` API ووصف فكرة الـ `devm_*` functions اللي بتحرر resources تلقائياً |
| [Managed device resources](https://lwn.net/Articles/215861/) | تغطية تانية للـ devres من زاوية driver author |
| [Device model changes in store](https://lwn.net/Articles/128644/) | مقال تاريخي بيوثق التغييرات اللي اتعملت في الـ device model في kernel 2.6 |
| [Driver porting: Device classes](https://lwn.net/Articles/31370/) | شرح الـ device classes وإزاي بيشتغلوا مع `/sys/class` |
| [device property: Introducing software nodes](https://lwn.net/Articles/770825/) | تقديم الـ software fwnode كنوع جديد من firmware nodes بديل للـ `of_node` |
| [A debugfs file system for managed resources (devres)](https://lwn.net/Articles/606554/) | إضافة debugfs interface للـ devres لتسهيل الـ debugging |

---

### التوثيق الرسمي في الـ kernel

كل الملفات دي موجودة في مصدر الـ kernel تحت `Documentation/`:

```
Documentation/driver-api/driver-model/overview.rst
Documentation/driver-api/driver-model/devres.rst
Documentation/driver-api/driver-model/device.rst
Documentation/driver-api/driver-model/driver.rst
Documentation/driver-api/driver-model/bus.rst
Documentation/driver-api/driver-model/class.rst
Documentation/driver-api/device_link.rst
Documentation/driver-api/pm/devices.rst
Documentation/driver-api/pin-control.rst
Documentation/core-api/kobject.rst
Documentation/devicetree/usage-model.rst
```

روابط HTML المُولَّدة:

| الصفحة | الرابط |
|--------|--------|
| Linux Kernel Device Model (overview) | [docs.kernel.org](https://docs.kernel.org/driver-api/driver-model/overview.html) |
| kobject, kset, ktype | [docs.kernel.org](https://docs.kernel.org/core-api/kobject.html) |
| Device links | [static.lwn.net/kerneldoc](https://static.lwn.net/kerneldoc/driver-api/device_link.html) |
| Devres — Managed Device Resource | [static.lwn.net/kerneldoc](https://static.lwn.net/kerneldoc/driver-api/driver-model/devres.html) |
| Device drivers infrastructure | [docs.kernel.org](https://www.kernel.org/doc/html/v4.14/driver-api/infrastructure.html) |
| kobject.txt (classic) | [kernel.org](https://www.kernel.org/doc/Documentation/kobject.txt) |

---

### ملفات الـ source الأساسية في الـ kernel

الـ header الرئيسي اللي بنشتغل عليه:

```
include/linux/device.h          ← الملف الرئيسي (struct device + كل الـ API)
include/linux/device/bus.h      ← struct bus_type
include/linux/device/class.h    ← struct class
include/linux/device/driver.h   ← struct device_driver
include/linux/device/devres.h   ← devres API declarations
include/linux/kobject.h         ← struct kobject, struct kset
drivers/base/core.c             ← device_register, device_add, device_del
drivers/base/bus.c              ← bus_register, driver_attach
drivers/base/class.c            ← class_register
drivers/base/driver.c           ← driver_register
drivers/base/devres.c           ← devm_* implementation
```

---

### Commits مهمة في تاريخ الـ device model

| الحدث | الوصف |
|-------|-------|
| Linux 2.5 (2002) | Patrick Mochel كتب الـ device model الأصلي — أُعلن عنه في [OLS 2002](https://www.landley.net/kdocs/ols/2002/ols2002-pages-368-375.pdf) |
| Linux 2.6.27 (2008) | إضافة الـ `devres` API بواسطة Tejun Heo |
| Linux 4.10 (2017) | بدأت مناقشات الـ device links |
| Linux 4.13 (2017) | أول إضافة للـ `device_link_add()` |
| Linux 5.1 (2019) | تحسينات الـ `sync_state` callback |
| Linux 5.9+ | تثبيت الـ `DL_FLAG_CYCLE` وتحسينات الـ managed links |

للبحث عن commits بنفسك:

```bash
git log --oneline --follow -- drivers/base/core.c | head -30
git log --oneline --follow -- include/linux/device.h | head -30
git log --oneline --grep="device_link" -- drivers/base/ | head -20
```

---

### نقاشات الـ Mailing List

- **LKML (linux-kernel mailing list)**: البحث في [lore.kernel.org](https://lore.kernel.org/linux-kernel/) عن `device_link` أو `devres` أو `struct device`
- **linux-driver-devel**: نقاشات خاصة بتطوير الـ drivers
- مقال LWN القديم [Kernel development](https://lwn.net/Articles/93651/) بيوثق نقاشات مبكرة حول الـ device model في 2.6

```bash
# ابحث في أرشيف lore.kernel.org
# https://lore.kernel.org/linux-kernel/?q=device_link_add
```

---

### الكتب المرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: "The Linux Device Model" — شرح مفصل للـ kobjects، ksets، ktypes، sysfs، hotplug
- الكتاب كامل متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- الفصل 14 مباشرة: [PDF من LDD3](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)
- **ملاحظة**: الكتاب قديم (kernel 2.6.10) بس الأساسيات لسه صح

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: "Devices and Modules" — بيشرح الـ device model بشكل مختصر وعملي
- بيغطي: kobjects، sysfs، hotplug، device classes
- الكتاب مناسب كـ overview قبل ما تدخل في الكود

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 8 و 9**: بيشرحوا الـ device drivers في السياق الـ embedded
- بيغطي الـ platform devices وإزاي بيتعاملوا مع الـ device model
- مفيد للـ embedded developers اللي بيشتغلوا على ARM/MIPS SoCs

#### Linux Kernel in a Nutshell — Greg Kroah-Hartman
- متاح مجاناً: [kroah.com/lkn](http://www.kroah.com/lkn/)
- مرجع سريع لـ kernel configuration والـ driver APIs

---

### موارد kernelnewbies.org و elinux.org

| الموقع | الرابط | المحتوى |
|--------|--------|----------|
| kernelnewbies.org | [Drivers](https://kernelnewbies.org/Drivers) | قائمة مراجع وكتب drivers |
| kernelnewbies.org | [WritingPortableDrivers](https://kernelnewbies.org/WritingPortableDrivers) | دليل كتابة drivers محمولة عبر architectures |
| kernelnewbies.org | [Linux_3.5_DriverArch](https://kernelnewbies.org/Linux_3.5_DriverArch) | تغييرات الـ driver architecture في 3.5 |
| kernelnewbies.org | [Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | تغييرات الـ device model في 2.6.24 |
| elinux.org | [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل للـ Device Tree وعلاقته بالـ device model |
| elinux.org | [Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) | عروض تقديمية عن device drivers من مؤتمرات مختلفة |
| elinux.org | [Linux Device Drivers](https://elinux.org/Linux_Device_Drivers) | صفحة مجمعة لمراجع الـ drivers على elinux |
| linux-kernel-labs.github.io | [Device Model Lab](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html) | تمارين عملية على الـ device model مع كود |

---

### موارد إضافية على الإنترنت

| المصدر | الرابط | الملاحظة |
|--------|--------|----------|
| O'Reilly LDD3 Online | [Chapter 14](https://www.oreilly.com/library/view/linux-device-drivers/0596005903/ch14.html) | نسخة HTML من الفصل 14 |
| OLS 2002 Paper | [landley.net PDF](https://www.landley.net/kdocs/ols/2002/ols2002-pages-368-375.pdf) | الورقة الأصلية لـ Patrick Mochel عن الـ device model |
| Linux Journal | [The Driver Model Core, Part I](https://www.linuxjournal.com/article/6717) | مقال قديم بيشرح الـ driver model core |
| embetronicx.com | [Sysfs Tutorial](https://embetronicx.com/tutorials/linux/device-drivers/sysfs-in-linux-kernel/) | شرح عملي للـ sysfs مع كود |

---

### مصطلحات البحث

لو عايز تلاقي معلومات أكتر، استخدم الكلمات دي في Google أو في `git log`:

```
# للـ device model الأساسي
"linux kernel device model" kobject sysfs
"struct device" linux kernel driver binding
linux "device_register" "device_add" internals

# للـ device links
linux "device_link_add" supplier consumer probe order
linux DL_FLAG_PM_RUNTIME devlinks

# للـ devres
linux devm_ managed resources cleanup
"devm_kzalloc" "devres_add" implementation

# للـ firmware nodes
linux fwnode_handle of_node device tree platform
linux "set_primary_fwnode" ACPI OF unified

# للـ power management
linux "dev_pm_ops" runtime PM device suspend resume
linux "device_link" PM ordering

# للـ sysfs attributes
linux DEVICE_ATTR device_attribute show store sysfs
linux "device_create_file" kobject sysfs entry
```

---

### خريطة القراءة المقترحة

للمبتدئ اللي عايز يفهم الملف `include/linux/device.h` من الصفر:

```
1. ابدأ بـ LDD3 الفصل 14 (نظرة عامة)
   ↓
2. اقرأ docs.kernel.org/driver-api/driver-model/overview.html
   ↓
3. اقرأ docs.kernel.org/core-api/kobject.html
   ↓
4. اقرأ مقال LWN: "A fresh look at the kernel's device model"
   ↓
5. اتفرج على linux-kernel-labs device model lab (كود عملي)
   ↓
6. ادرس drivers/base/core.c جنب include/linux/device.h
   ↓
7. للـ device links: اقرأ LWN Articles/710494 + kernel doc device_link.rst
   ↓
8. للـ devres: اقرأ LWN Articles/222860 + drivers/base/devres.c
```
## Phase 8: Writing simple module

### الفكرة

**`device_add()`** هي الدالة المُصدَّرة اللي بتتحكم في تسجيل أي device في kernel. كل مرة بيتضاف hardware جديد أو virtual device — زي لما تعمل `modprobe` أو تشيل USB — بيتم استدعاؤها. هنعمل **kprobe** عليها علشان نشوف كل device بيتسجل في الـ kernel في real-time.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * device_add_probe.c
 * Hooks device_add() via kprobe to log every newly registered device.
 */

/* kprobe API and struct kprobe definition */
#include <linux/kprobes.h>

/* struct device, dev_name(), dev_bus_name() */
#include <linux/device.h>

/* pr_info(), THIS_MODULE */
#include <linux/kernel.h>

/* module_init / module_exit macros */
#include <linux/module.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDocs");
MODULE_DESCRIPTION("kprobe on device_add() — log every registered device");

/*
 * pre_handler: called just BEFORE device_add() executes.
 * regs: CPU registers at the moment of the probe hit.
 * The first argument (RDI on x86-64) is "struct device *dev".
 */
static int device_add_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve first argument: struct device *dev */
    struct device *dev = (struct device *)regs->di;

    /* Guard against a NULL pointer — safety first */
    if (!dev)
        return 0;

    pr_info("device_add: name='%s' bus/class='%s' driver='%s'\n",
            dev_name(dev),                          /* kobject name or init_name  */
            dev_bus_name(dev),                      /* bus or class name          */
            dev->driver ? dev->driver->name : "-"); /* driver if already bound    */

    return 0; /* 0 = keep running normally; non-zero would skip the probed fn */
}

/* kprobe struct: only .symbol_name and .pre_handler are mandatory */
static struct kprobe kp = {
    .symbol_name = "device_add",  /* target function by name */
    .pre_handler = device_add_pre,
};

static int __init device_probe_init(void)
{
    int ret;

    /* register_kprobe() patches one instruction with a breakpoint */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("device_add_probe: register_kprobe failed (%d)\n", ret);
        return ret;
    }

    pr_info("device_add_probe: kprobe planted at %p\n", kp.addr);
    return 0;
}

static void __exit device_probe_exit(void)
{
    /*
     * unregister_kprobe() removes the breakpoint and waits for any
     * in-flight handler to finish — prevents use-after-free on unload.
     */
    unregister_kprobe(&kp);
    pr_info("device_add_probe: kprobe removed\n");
}

module_init(device_probe_init);
module_exit(device_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe`، `register_kprobe()`، `unregister_kprobe()` |
| `linux/device.h` | بيعرّف `struct device`، `dev_name()`، `dev_bus_name()` |
| `linux/kernel.h` | بيجيب `pr_info()` و `THIS_MODULE` |
| `linux/module.h` | `module_init` / `module_exit` و macros المعتادة |

---

#### الـ `pre_handler` — قلب الـ kprobe

**الـ `regs->di`** على x86-64 بيمثل الـ register `RDI` اللي بيحمل أول argument للدالة — وده هو `struct device *dev`. بنعمل cast ليه علشان نقدر نوصل لحقول الـ struct زي الاسم والـ bus والـ driver.

**الـ `dev_name()`** بترجع اسم الـ device من الـ kobject أو `init_name`، و **`dev_bus_name()`** بترجع اسم الـ bus أو الـ class اللي مسجّل عليه الـ device. ده بيدينا صورة واضحة عن كل device بيتضاف لحظة بلحظة.

---

#### الـ `struct kprobe`

**الـ `.symbol_name`** بيخلي الـ kernel يحول الاسم لعنوان تلقائياً عبر الـ kallsyms. **الـ `.pre_handler`** هو الـ callback اللي بيتنفذ قبل أول instruction في `device_add()` — ده اللي بيدينا إمكانية قراءة الـ arguments قبل ما تتغير.

---

#### الـ `module_init`

`register_kprobe()` بتكتب breakpoint instruction (int3 على x86) في مكان `device_add()` في الذاكرة، وبتحفظ العنوان الحقيقي في `kp.addr` للـ logging. لو فشلت — مثلاً لأن الرمز مش موجود أو محمي بـ `CONFIG_KPROBES_INSN_SLOTS` — بنرجع الخطأ فوراً.

---

#### الـ `module_exit`

`unregister_kprobe()` بتشيل الـ breakpoint وبتنتظر أي thread شغّال جوه الـ handler خلاص ينهي تنفيذه. من غير السطر ده، ممكن يحصل kernel panic لو حاول thread يرجع لـ handler بعد ما الـ module اتشال من الذاكرة.

---

### Makefile للبناء

```makefile
obj-m += device_add_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod device_add_probe.ko

# شاهد الـ output (كل device بيتسجل بيظهر هنا)
sudo dmesg -w | grep device_add

# مثال عملي: شيل وأعد تشغيل USB device لتشوف الـ output
# أو حمّل driver جديد: sudo modprobe dummy

# إزالة الـ module
sudo rmmod device_add_probe
```

**مثال على الـ output المتوقع:**
```
device_add: name='input0' bus/class='input' driver='-'
device_add: name='eth0' bus/class='platform' driver='virtio_net'
device_add: name='tty0' bus/class='tty' driver='-'
```
