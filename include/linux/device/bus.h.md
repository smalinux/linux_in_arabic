## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الملف `include/linux/device/bus.h` جزء من **Driver Core** — القلب النابض لـ Linux Device Model. المسؤول عنه في MAINTAINERS هو Greg Kroah-Hartman وRafael J. Wysocki تحت entry اسمه **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS**.

---

### القصة من الأول: ليه أصلاً محتاجين Bus؟

تخيل معايا مطار كبير. عندك:
- **الطائرات** = الـ devices (USB keyboard، PCI graphics card، I2C sensor).
- **الـ pilots** = الـ drivers (الكود اللي يعرف يشغّل الجهاز).
- **الـ gates (بوابات السفر)** = الـ buses (USB bus، PCI bus، I2C bus، platform bus).

الـ kernel لما بيشتغل بيلاقي آلاف الأجهزة وآلاف الـ drivers. المشكلة: إزاي كل driver يلاقي الجهاز اللي هو مسؤول عنه؟ ومين المسؤول عن ربطهم ببعض؟

الإجابة هي الـ **bus**. كل جهاز في Linux لازم يكون متعلق بـ bus معين — حتى لو هو bus افتراضي زي `platform`. والـ bus هو اللي بيقول: "أنا بعرف أطابق devices بـ drivers، وأنا اللي بيجري الـ probe، وأنا اللي بيدير الـ power management."

---

### الملف `bus.h`: إيه اللي بيعمله بالظبط؟

الملف ده هو الـ **header** الرسمي اللي بيعرّف:

1. **`struct bus_type`** — الـ struct الأساسي اللي كل bus في Linux بيعرّف نفسه بيه. لما تيجي تعمل USB bus أو PCI bus أو I2C bus، بتملا نسخة من الـ struct ده.

2. **الـ callbacks (function pointers)** — الـ bus بيقول للـ kernel: "لما يجي device جديد، استدعي الـ `match` function عندي عشان أشوف مين من الـ drivers يقدر يشتغل معاه."

3. **الـ APIs** — functions زي `bus_register`, `bus_find_device`, `bus_for_each_dev` اللي كل كود kernel يقدر يستخدمها للتعامل مع أي bus.

4. **الـ notifiers** — نظام إشعارات بيخلي أي جزء في الـ kernel يتابع أحداث زي: "جهاز اتضاف للـ bus ده" أو "driver اتربط بجهاز".

---

### الـ `struct bus_type` بالتفصيل

```c
struct bus_type {
    const char *name;           /* اسم الـ bus زي "usb", "pci", "i2c" */
    const char *dev_name;       /* template لتسمية الأجهزة */

    /* attribute groups للـ sysfs */
    const struct attribute_group **bus_groups;
    const struct attribute_group **dev_groups;
    const struct attribute_group **drv_groups;

    /* الـ callbacks — ده قلب الموضوع */
    int (*match)(struct device *dev, const struct device_driver *drv);
    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    void (*sync_state)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    /* power management */
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    /* DMA */
    int (*dma_configure)(struct device *dev);
    void (*dma_cleanup)(struct device *dev);

    const struct dev_pm_ops *pm;
    bool need_parent_lock;
};
```

---

### القصة الكاملة: إيه اللي بيحصل لما تحشر USB stick؟

```
┌─────────────────────────────────────────────────────┐
│  1. Hardware: USB stick اتحشر                       │
│                    │                                 │
│  2. USB Host Controller يكتشف device جديد           │
│                    │                                 │
│  3. drivers/base/bus.c يضيف الـ device على USB bus  │
│                    │                                 │
│  4. bus_type.match() بتتكرر على كل USB driver       │
│     وبتسأل: "أنت قادر تشتغل مع الـ device ده؟"     │
│                    │                                 │
│  5. Driver اللي يقول "أيوه" → bus_type.probe()      │
│     تستدعي driver->probe()                          │
│                    │                                 │
│  6. Device جاهز للاستخدام في /dev/sda1             │
└─────────────────────────────────────────────────────┘
```

**الـ `bus.h`** هو العقد اللي بيحدد شكل كل خطوة من الخطوات دي.

---

### الـ Bus Notifier System

```c
enum bus_notifier_event {
    BUS_NOTIFY_ADD_DEVICE,       /* جهاز اتضاف */
    BUS_NOTIFY_DEL_DEVICE,       /* جهاز هيتشال */
    BUS_NOTIFY_REMOVED_DEVICE,   /* جهاز اتشال فعلاً */
    BUS_NOTIFY_BIND_DRIVER,      /* driver هيتربط */
    BUS_NOTIFY_BOUND_DRIVER,     /* driver اتربط */
    BUS_NOTIFY_UNBIND_DRIVER,    /* driver هيتفك */
    BUS_NOTIFY_UNBOUND_DRIVER,   /* driver اتفك */
    BUS_NOTIFY_DRIVER_NOT_BOUND, /* driver فشل في الـ probe */
};
```

أي subsystem في الـ kernel يقدر يسجّل نفسه عن طريق `bus_register_notifier()` وياخد إشعار بأي حدث على أي bus. مثلاً: subsystem الـ IOMMU محتاج يعرف لما device بيتضاف على PCI bus عشان يعمل له mapping.

---

### Device Matching: الـ Helper Functions

الملف بيوفر مجموعة جاهزة من functions للـ matching تقدر تستخدمها على أي bus:

| Function | بتطابق إيه |
|---|---|
| `device_match_name` | اسم الـ device |
| `device_match_of_node` | الـ Device Tree node |
| `device_match_fwnode` | الـ firmware node (DT أو ACPI) |
| `device_match_devt` | الـ device type number |
| `device_match_acpi_dev` | الـ ACPI companion device |
| `device_match_any` | أي device (للـ iteration) |

---

### ليه الملف ده مهم؟

كل bus في Linux — USB، PCI، I2C، SPI، platform، MDIO، CAN، وغيرهم — بيعتمد على الـ `struct bus_type` المعرّف هنا. بدون الـ bus abstraction دي، كان كل driver محتاج يكتب كوده الخاص لاكتشاف الأجهزة وربط الـ drivers بيها — فوضى كاملة.

---

### الملفات المرتبطة اللي المفروض تعرفها

**Core Implementation:**
- `drivers/base/bus.c` — التنفيذ الفعلي لكل functions الـ bus
- `drivers/base/dd.c` — **device-driver binding** — ده اللي بينفذ الـ probe/remove
- `drivers/base/core.c` — الـ device lifecycle الكاملة
- `drivers/base/driver.c` — الـ `struct device_driver` APIs

**Headers المرتبطة:**
- `include/linux/device.h` — الـ header الرئيسي، بيضم `bus.h` وغيره
- `include/linux/device/driver.h` — الـ `struct device_driver`
- `include/linux/device/class.h` — الـ device class (مختلف عن الـ bus)
- `include/linux/kobject.h` — الـ infrastructure تحت الـ device model
- `include/linux/pm.h` — الـ power management types

**أمثلة على Buses حقيقية:**
- `drivers/usb/core/driver.c` — USB bus_type
- `drivers/pci/pci-driver.c` — PCI bus_type
- `drivers/i2c/i2c-core-base.c` — I2C bus_type
- `drivers/base/platform.c` — Platform bus_type

**Documentation:**
- `Documentation/driver-api/driver-model/bus.rst`
- `Documentation/driver-api/driver-model/binding.rst`
## Phase 2: شرح الـ Bus Framework

### المشكلة — ليه الـ Bus Framework موجود أصلاً؟

في الـ embedded Linux، عندك عشرات الـ devices متوصلة بالـ SoC — بعضها على I2C، بعضها على SPI، بعضها على PCI، وبعضها على platform bus وهمي. كل نوع bus له طريقة مختلفة للـ:

- **اكتشاف الـ devices** (enumeration): PCI بيعمل scan تلقائي، I2C بيحتاج device tree.
- **مطابقة الـ driver بالـ device** (matching): SPI بيتطابق بالـ modalias، OF بيتطابق بالـ compatible string.
- **إدارة الـ power**: كل bus له سياق power management مختلف.

لو مفيش abstraction موحد، كل driver هيتعامل مع كل البنية التحتية بنفسه — كل device registration ستبقى manual، الـ sysfs entries ستكون مبعثرة، والـ power management ستكون chaos.

**الـ Bus Framework** حل المشكلة دي بإنه وفّر **طبقة وسطى** تعرف "قواعد اللعبة" لكل نوع bus: إزاي بيتعرف على الـ devices، إزاي بيربط الـ drivers، وإزاي بيدير الـ lifecycle بتاعهم.

---

### الحل — الـ Kernel Driver Model

الـ kernel اختار يعمل **Driver Model** موحد. الفكرة المركزية إن في ثلاث عناصر أساسية:

| العنصر | التعريف |
|--------|---------|
| **`bus_type`** | قواعد اللعبة — المنطق المشترك لنوع الـ bus |
| **`device`** | الـ hardware موجود على الـ bus |
| **`device_driver`** | الكود اللي بيعرف يتعامل مع نوع معين من الـ devices |

الـ bus هو الـ orchestrator — هو اللي يقرر إيه الـ driver اللي يشتغل مع إيه الـ device.

---

### التشبيه الواقعي — وكالة التوظيف

تخيل الـ bus هو **وكالة توظيف متخصصة**:

- **الـ `bus_type`** هي إدارة الوكالة — بتحدد شروط التوظيف وبتدير العملية.
- **الـ `device`** هو صاحب العمل — عنده شُغل محدد محتاجه.
- **الـ `device_driver`** هو المتقدم للوظيفة — عنده مهارات محددة.
- **`match()`** هي مقابلة الـ HR — بتقرر هل المتقدم مناسب للوظيفة دي.
- **`probe()`** هي أول يوم شغل — المتقدم يبدأ فعلاً.
- **`remove()`** هي نهاية العقد — تنظيف وإنهاء العلاقة.
- **`bus_register_notifier()`** هو الـ LinkedIn notifications — كل الناس المهتمة بتتعلم لما حاجة تتغير.
- **الـ `klist`** اللي جوا الـ bus هي قائمة الموظفين الحاليين والوظائف الشاغرة.
- **`sysfs` entries** هي الـ website بتاع الوكالة — الكل يشوف مين اشتغل فين.

لما يجي device جديد (صاحب عمل جديد) → الوكالة بتعمل `match()` مع كل الـ drivers المسجلين → لو في مناسب → بتبعت له `probe()`.

---

### الـ Big Picture Architecture

```
User Space
    │
    │  (uevents → udev → /dev nodes)
    ▼
┌──────────────────────────────────────────────────────┐
│                     sysfs (/sys)                     │
│  /sys/bus/i2c/    /sys/bus/spi/    /sys/bus/pci/     │
│     devices/          drivers/        devices/       │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│              Linux Driver Core (drivers/base/)        │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │              bus_type (e.g. i2c_bus_type)   │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │    │
│  │  │  match() │  │ probe()  │  │ uevent() │  │    │
│  │  └──────────┘  └──────────┘  └──────────┘  │    │
│  │  ┌────────────────────────────────────────┐ │    │
│  │  │  klist of devices  │  klist of drivers │ │    │
│  │  └────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────┘    │
│          │                        │                  │
│  ┌───────▼──────┐        ┌────────▼──────┐          │
│  │   device     │        │ device_driver │          │
│  │  (i2c_client)│        │ (i2c_driver)  │          │
│  └───────┬──────┘        └───────┬───────┘          │
│          │  match succeeds       │                  │
│          └───────────────────────┘                  │
│                driver->probe(dev) called             │
└──────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│              Hardware (Physical / Virtual)            │
│  I2C Controller    SPI Controller    Platform regs    │
└──────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الـ `struct bus_type`

الـ `struct bus_type` هو **قلب الـ framework**. هو مش بيمثل bus instance واحدة — هو بيمثل **نوع** الـ bus كله. يعني `i2c_bus_type` واحد بس بيخدم كل الـ I2C controllers والـ devices في النظام.

```c
struct bus_type {
    const char  *name;          /* اسم الـ bus في sysfs مثل "i2c", "spi", "pci" */
    const char  *dev_name;      /* للـ enumeration مثل "i2c%u" */

    /* attribute groups تظهر في sysfs */
    const struct attribute_group **bus_groups;  /* /sys/bus/<name>/  */
    const struct attribute_group **dev_groups;  /* /sys/bus/<name>/devices/<dev>/ */
    const struct attribute_group **drv_groups;  /* /sys/bus/<name>/drivers/<drv>/ */

    /* callbacks — اللي البص يعرفه وبس */
    int (*match)(struct device *dev, const struct device_driver *drv);
    int (*uevent)(const struct device *dev, struct kobj_uevent_env *env);
    int (*probe)(struct device *dev);
    void (*sync_state)(struct device *dev);
    void (*remove)(struct device *dev);
    void (*shutdown)(struct device *dev);

    /* IRQ affinity — لازم تعرف topology الـ CPU */
    const struct cpumask *(*irq_get_affinity)(struct device *dev, unsigned int irq_vec);

    /* hot-plug support */
    int (*online)(struct device *dev);
    int (*offline)(struct device *dev);

    /* legacy PM — قبل dev_pm_ops */
    int (*suspend)(struct device *dev, pm_message_t state);
    int (*resume)(struct device *dev);

    /* SR-IOV support */
    int (*num_vf)(struct device *dev);

    /* DMA setup */
    int (*dma_configure)(struct device *dev);
    void (*dma_cleanup)(struct device *dev);

    /* modern PM ops */
    const struct dev_pm_ops *pm;

    bool need_parent_lock; /* لو محتاج lock الـ parent أثناء probe/remove */
};
```

#### العلاقة بين الـ Structs

```
struct bus_type
├── name ──────────────────────► "/sys/bus/i2c"
├── match() ────────────────────► قارن device_id مع driver_id
├── probe() ────────────────────► استدعي driver->probe(dev)
├── pm ─────────────────────────► struct dev_pm_ops
│                                  ├── suspend()
│                                  ├── resume()
│                                  └── ...
├── bus_groups ─────────────────► struct attribute_group[]
│                                  └── struct attribute[]
│                                      ├── show()
│                                      └── store()
└── [private data inside drivers/base]
    ├── struct klist  ── klist of devices ──► struct device (linked)
    │                                          └── struct klist_node
    └── struct klist  ── klist of drivers ──► struct device_driver (linked)
                                               └── struct klist_node
```

**ملاحظة مهمة**: الـ `struct bus_type` اللي بنشوفه في الـ header ما فيهوش الـ klist مباشرة — دي موجودة في `struct subsys_private` اللي في `drivers/base/base.h` وبتتربط جوا `bus_register()`. الـ API الخارجي بيخفي التفاصيل دي عمداً.

---

### الـ Kobject و Kset — الأساس اللي الـ Bus بيتبنى عليه

> **Kobject Subsystem**: طبقة generic لإدارة الـ objects في الـ kernel — بتوفر reference counting، sysfs representation، وـ uevent notifications. كل حاجة في الـ driver model مبنية فوقيها.

```
struct kobject
├── name        ── اسم الـ directory في sysfs
├── parent      ── الـ kobject الأب (يبني شجرة /sys)
├── kset        ── المجموعة اللي ينتمي ليها
├── kref        ── reference count (لو وصل 0 → release())
└── sd          ── kernfs_node (الـ sysfs entry الفعلي)

struct kset
├── list        ── linked list من كل الـ kobjects
├── list_lock   ── spinlock لحماية الـ list
├── kobj        ── embedded kobject (الـ kset نفسه له kobj!)
└── uevent_ops  ── filter/name/uevent callbacks
```

الـ bus بيستخدم `kset` لتجميع الـ devices والـ drivers تحت `/sys/bus/<name>/devices/` و `/sys/bus/<name>/drivers/`.

---

### الـ Klist — Thread-Safe List مع Reference Counting

> الـ `klist` مش مجرد `list_head` عادي — هو بيضيف `kref` لكل node، يعني تقدر تعمل iterate على الـ list وفي نفس الوقت node بتتحذف من thread تاني من غير race condition.

```c
struct klist {
    spinlock_t      k_lock;   /* يحمي العمليات على الـ list */
    struct list_head k_list;  /* الـ list الفعلية */
    void (*get)(struct klist_node *);  /* زيادة الـ ref count */
    void (*put)(struct klist_node *);  /* نقص الـ ref count */
};

struct klist_node {
    void            *n_klist;  /* pointer للـ klist الأم */
    struct list_head n_node;   /* الـ list linkage */
    struct kref      n_ref;    /* reference count */
};
```

الـ bus بيخزن الـ devices والـ drivers في `klist` عشان:
1. **Thread safety** أثناء الـ hotplug — device بتتحذف وdriver بيعمل probe في نفس الوقت.
2. **Safe iteration** — `bus_for_each_dev()` و `bus_for_each_drv()` بيشتغلوا بدون حاجة لـ global lock طول فترة الـ iteration.

---

### دورة حياة الـ Device على الـ Bus

#### 1. تسجيل الـ Bus

```c
/* مثال: تسجيل i2c bus */
const struct bus_type i2c_bus_type = {
    .name       = "i2c",
    .match      = i2c_device_match,
    .probe      = i2c_device_probe,
    .remove     = i2c_device_remove,
    .shutdown   = i2c_device_shutdown,
};

/* في i2c_init() */
bus_register(&i2c_bus_type);
/* النتيجة: /sys/bus/i2c/ ظهر */
```

#### 2. إضافة Device

```c
/* لما kernel يكتشف i2c device من device tree */
device_register(&client->dev);
/*
 * drivers/base/core.c: device_add()
 *   ├── kobject_add()          → يعمل /sys/bus/i2c/devices/0-0050/
 *   ├── bus_add_device()       → يضيف للـ klist
 *   └── bus_probe_device()     → يبدأ عملية الـ matching
 */
```

#### 3. عملية الـ Matching

```c
/* bus_type->match() بتستدعيها الـ driver core */
static int i2c_device_match(struct device *dev, const struct device_driver *drv)
{
    /* أول حاجة: جرب of_driver_match_device() */
    /* تاني حاجة: جرب acpi_driver_match_device() */
    /* آخر حاجة: قارن i2c_client->name مع i2c_driver->id_table */
    return i2c_match_id(i2c_drv->id_table, client) != NULL;
}
```

#### 4. الـ Probe

```c
/* bus_type->probe() بتستدعيها driver core لما match ينجح */
static int i2c_device_probe(struct device *dev)
{
    struct i2c_client *client = to_i2c_client(dev);
    struct i2c_driver *driver = to_i2c_driver(dev->driver);

    /* بتستدعي driver->probe() الفعلي */
    return driver->probe(client);
}
```

#### 5. تسجيل الـ Driver

```c
/* مثال: driver لـ EEPROM على I2C */
static const struct i2c_device_id eeprom_id[] = {
    { "at24c256", 0 },
    { }
};

static struct i2c_driver eeprom_driver = {
    .driver = {
        .name = "at24",
    },
    .probe  = at24_probe,
    .remove = at24_remove,
    .id_table = eeprom_id,
};

i2c_add_driver(&eeprom_driver);
/* النتيجة: /sys/bus/i2c/drivers/at24/ ظهر */
/* + kernel بيجرب يعمل match مع كل الـ devices الموجودة */
```

---

### الـ Bus Notifier System

الـ `bus_notifier_event` بيسمح لأي كود في الـ kernel يتابع أحداث الـ bus:

```c
enum bus_notifier_event {
    BUS_NOTIFY_ADD_DEVICE,        /* device اتضاف للـ bus */
    BUS_NOTIFY_DEL_DEVICE,        /* device على وشك يتشال */
    BUS_NOTIFY_REMOVED_DEVICE,    /* device اتشال فعلاً */
    BUS_NOTIFY_BIND_DRIVER,       /* driver على وشك يعمل probe */
    BUS_NOTIFY_BOUND_DRIVER,      /* probe نجح */
    BUS_NOTIFY_UNBIND_DRIVER,     /* driver على وشك يتفك */
    BUS_NOTIFY_UNBOUND_DRIVER,    /* driver اتفك */
    BUS_NOTIFY_DRIVER_NOT_BOUND,  /* probe فشل */
};
```

**مثال واقعي**: الـ IOMMU subsystem بيسجل notifier على PCI bus عشان يعرف لما device جديدة بتتضاف فيعمل لها IOMMU mapping قبل ما الـ driver يشوفها.

```c
/* IOMMU code */
static struct notifier_block iommu_dev_nb = {
    .notifier_call = iommu_bus_notifier,
};

bus_register_notifier(&pci_bus_type, &iommu_dev_nb);

static int iommu_bus_notifier(struct notifier_block *nb,
                               unsigned long action, void *data)
{
    struct device *dev = data;
    if (action == BUS_NOTIFY_ADD_DEVICE)
        iommu_probe_device(dev); /* setup IOMMU mapping */
    return NOTIFY_DONE;
}
```

---

### الـ Device Matching Functions

الـ framework بيوفر matching functions جاهزة تقدر تستخدمها مباشرة مع `bus_find_device()`:

```c
/* البحث بالاسم */
struct device *dev = bus_find_device_by_name(&i2c_bus_type, NULL, "0-0050");

/* البحث بـ device tree node */
struct device *dev = bus_find_device_by_of_node(&platform_bus_type, np);

/* البحث بـ fwnode (يشمل DT و ACPI) */
struct device *dev = bus_find_device_by_fwnode(&pci_bus_type, fwnode);

/* البحث بـ dev_t */
struct device *dev = bus_find_device_by_devt(&block_bus_type, devt);

/* iteration كامل */
bus_for_each_dev(&usb_bus_type, NULL, &data, my_callback);
```

الـ `device_match_t` هو typedef بسيط:
```c
typedef int (*device_match_t)(struct device *dev, const void *data);
/* بترجع != 0 لو الـ device هو اللي بندور عليه */
```

---

### الـ Bus Attributes في sysfs

```c
struct bus_attribute {
    struct attribute attr;            /* name + permissions */
    ssize_t (*show)(const struct bus_type *bus, char *buf);
    ssize_t (*store)(const struct bus_type *bus, const char *buf, size_t count);
};

/* الـ macros لتعريف attributes */
BUS_ATTR_RO(version);   /* يعمل: struct bus_attribute bus_attr_version */
BUS_ATTR_RW(power);     /* قابل للقراءة والكتابة */
BUS_ATTR_WO(rescan);    /* write-only */
```

**مثال**: PCI bus بيعمل `/sys/bus/pci/rescan` — تكتب فيه `1` يعمل rescan للـ PCI bus.

---

### إيه اللي الـ Bus يمتلكه vs إيه اللي بيفوّضه للـ Drivers

| المسؤولية | الـ Bus (`bus_type`) | الـ Driver (`device_driver`) |
|-----------|---------------------|------------------------------|
| تحديد شروط الـ matching | `match()` — البص يقرر الـ protocol | الـ driver بيوفر `id_table` أو `of_match_table` |
| استدعاء الـ probe | `probe()` — البص يختار الـ calling convention | الـ driver يعمل التهيئة الفعلية للـ hardware |
| sysfs hierarchy | البص يعمل `/sys/bus/<name>/` | الـ driver يضيف attributes خاصة بيه |
| Power management | الـ `pm` ops توفر الـ framework | كل driver بيعمل `suspend/resume` الخاص بيه |
| Device enumeration | البص يعرف إزاي يكتشف الـ devices | الـ driver مش مهتم بعدد الـ devices |
| Hotplug uevents | `uevent()` يضيف bus-specific env vars | الـ driver مش بيتدخل في الـ uevents |
| DMA setup | `dma_configure()` — bus-level IOMMU/DMA config | الـ driver بيستخدم DMA APIs بعد الـ setup |
| IRQ affinity | `irq_get_affinity()` — bus topology aware | الـ driver بس بيطلب الـ IRQ |

---

### الـ `sync_state` — Concept المهم

الـ `sync_state()` callback هو حل لمشكلة دقيقة في الـ boot:

- الـ bootloader بيفعّل hardware في حالة معينة (مثلاً: clock enabled، regulator on).
- الـ kernel بيحتاج يبقى في نفس الحالة لحد ما الـ driver يبدأ.
- بس لو في dependencies — driver A محتاج driver B يشتغل الأول.

**الـ `sync_state()` بيتستدعى فقط لما كل الـ consumers اللي عارفينهم وقت الـ boot عملوا bind لـ driver.** يعني: "دلوقتي تأكدنا إن كل اللي محتاجك شغال — تقدر تبقى consistent مع الـ software state."

```
Boot time:
  Bootloader: CLK_A = enabled
  Kernel start: CLK_A still enabled (for consumers)

  Consumer1 → binds driver ✓
  Consumer2 → binds driver ✓

  → sync_state(CLK_A_dev) called
  → CLK driver can now disable CLK_A if software says it's off
```

---

### خلاصة — الـ Bus Framework كـ Contract

الـ `bus_type` هو **عقد** بين ثلاث أطراف:

```
    Device Hardware
          │
          ▼
    struct bus_type  ◄──── defines the rules
          │
    ┌─────┴──────┐
    ▼            ▼
struct device   struct device_driver
(what exists)   (what can handle it)
          │
          └─► match() → probe() → running system
```

الـ framework بيضمن إن:
1. كل device لها مكان في `/sys`.
2. كل driver بيلاقي الـ device المناسبة تلقائياً.
3. الـ power management بيشتغل بشكل موحد.
4. الـ hotplug بيتعمل بدون race conditions.
5. الـ userspace بيتعلم بكل حاجة عن طريق uevents.

ده هو سر قوة الـ Linux driver model — مش بس abstraction، هو **نظام تنظيمي كامل** بيخلي الـ kernel قادر يشغل ملايين الـ devices المختلفة بنفس المنطق الأساسي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### `enum bus_notifier_event`

| القيمة | المعنى |
|--------|--------|
| `BUS_NOTIFY_ADD_DEVICE` | device اتضاف للـ bus |
| `BUS_NOTIFY_DEL_DEVICE` | device على وشك يتشال |
| `BUS_NOTIFY_REMOVED_DEVICE` | device اتشال فعلاً |
| `BUS_NOTIFY_BIND_DRIVER` | driver على وشك يتربط بـ device |
| `BUS_NOTIFY_BOUND_DRIVER` | driver اترابط فعلاً |
| `BUS_NOTIFY_UNBIND_DRIVER` | driver على وشك يتفك |
| `BUS_NOTIFY_UNBOUND_DRIVER` | driver اتفك فعلاً |
| `BUS_NOTIFY_DRIVER_NOT_BOUND` | driver فشل في الـ binding |

> الـ notifier بيتبعت لكل entity مشتركة في الـ bus notifier chain — المستقبل بياخد `struct device *` كـ argument.

#### `enum kobject_action` (من `kobject.h`)

| القيمة | المعنى |
|--------|--------|
| `KOBJ_ADD` | kobject اتضاف |
| `KOBJ_REMOVE` | kobject اتشال |
| `KOBJ_CHANGE` | attribute اتغير |
| `KOBJ_MOVE` | kobject اتنقل |
| `KOBJ_ONLINE` | device اتحول online |
| `KOBJ_OFFLINE` | device اتحول offline |
| `KOBJ_BIND` | driver اترابط |
| `KOBJ_UNBIND` | driver اتفك |

#### Macros للـ Bus Attributes

| الـ Macro | النتيجة |
|-----------|---------|
| `BUS_ATTR_RW(_name)` | attribute قابلة للقراية والكتابة |
| `BUS_ATTR_RO(_name)` | attribute للقراية بس |
| `BUS_ATTR_WO(_name)` | attribute للكتابة بس |

---

### الـ Structs المهمة

#### 1. `struct bus_type`

**الغرض:** التعريف الكامل لنوع الـ bus في الـ driver model — بيحتوي على الاسم والـ callbacks وعمليات الـ PM.

| الـ Field | النوع | الوظيفة |
|-----------|-------|---------|
| `name` | `const char *` | اسم الـ bus (مثلاً `"pci"`, `"usb"`) |
| `dev_name` | `const char *` | format لتسمية الـ devices تلقائياً (`"foo%u"`) |
| `bus_groups` | `const struct attribute_group **` | attributes ظاهرة في `/sys/bus/<name>/` |
| `dev_groups` | `const struct attribute_group **` | attributes تتضاف لكل device على الـ bus |
| `drv_groups` | `const struct attribute_group **` | attributes تتضاف لكل driver |
| `match` | function pointer | بيقارن device بـ driver — رجعته positive = match |
| `uevent` | function pointer | بيضيف env variables للـ uevent |
| `probe` | function pointer | بيستدعي driver's probe لما يحصل match |
| `sync_state` | function pointer | بيعمل sync بعد ما كل الـ consumers يتربطوا |
| `remove` | function pointer | بيتنادى لما device يتشال |
| `shutdown` | function pointer | بيعمل quiesce للـ device وقت الـ shutdown |
| `irq_get_affinity` | function pointer | بيرجع `cpumask` للـ IRQ affinity |
| `online` / `offline` | function pointers | للـ hot-plug — online بيرجّع device، offline بيجهّزه للشيل |
| `suspend` / `resume` | function pointers | legacy PM callbacks |
| `num_vf` | function pointer | عدد الـ Virtual Functions (للـ SR-IOV) |
| `dma_configure` | function pointer | setup الـ DMA للـ device |
| `dma_cleanup` | function pointer | cleanup بعد فك ارتباط الـ device |
| `pm` | `const struct dev_pm_ops *` | الـ PM callbacks المنظمة (runtime + sleep states) |
| `need_parent_lock` | `bool` | لو `true`، الـ core بيقفل parent قبل probe/remove |

**الارتباط بالـ Structs التانية:**
- **الـ** `struct bus_type` بيشاور على `struct device` و`struct device_driver` في كل callbacks.
- **الـ** kernel بيحتفظ بـ private data مرتبطة بيه في `struct bus_type_private` (في `drivers/base/base.h`) — اللي فيها الـ `klist` للـ devices والـ drivers.

---

#### 2. `struct bus_attribute`

**الغرض:** تعريف attribute واحدة ظاهرة في `/sys/bus/<name>/`.

| الـ Field | النوع | الوظيفة |
|-----------|-------|---------|
| `attr` | `struct attribute` | الاسم والـ permissions في sysfs |
| `show` | function pointer | بيتقرا لما تعمل `cat` على الـ file |
| `store` | function pointer | بيتنادى لما تكتب في الـ file |

---

#### 3. `struct kobject` (من `kobject.h`)

**الغرض:** الـ building block الأساسي لكل object في الـ driver model — بيوفر reference counting وتمثيل في sysfs.

| الـ Field | الوظيفة |
|-----------|---------|
| `name` | اسم الـ object في sysfs |
| `entry` | حلقة في قائمة الـ kset |
| `parent` | الـ kobject الأب (hierarchy) |
| `kset` | المجموعة اللي ينتمي ليها |
| `ktype` | نوع الـ object (callbacks للـ release والـ sysfs) |
| `sd` | الـ kernfs_node — الملف في sysfs |
| `kref` | الـ reference count |
| `state_initialized` | اتعمل له `kobject_init`؟ |
| `state_in_sysfs` | موجود في sysfs؟ |
| `state_add_uevent_sent` | بُعت ليه ADD uevent؟ |
| `state_remove_uevent_sent` | بُعت ليه REMOVE uevent؟ |
| `uevent_suppress` | لو `true`، مش بيبعت uevents |

---

#### 4. `struct kset` (من `kobject.h`)

**الغرض:** مجموعة من الـ kobjects — بيمثل `/sys/bus/` كلها مثلاً.

| الـ Field | الوظيفة |
|-----------|---------|
| `list` | قائمة كل الـ kobjects في المجموعة |
| `list_lock` | spinlock لحماية القائمة |
| `kobj` | الـ kobject الخاص بالـ kset نفسه |
| `uevent_ops` | callbacks للـ filter والـ uevent على المستوى الجماعي |

---

#### 5. `struct klist` و`struct klist_node` (من `klist.h`)

**الغرض:** قائمة مترابطة thread-safe بـ reference counting — الـ bus_type_private بيستخدمها لقائمة الـ devices والـ drivers.

| الـ Field | الوظيفة |
|-----------|---------|
| `k_lock` | spinlock لحماية القائمة |
| `k_list` | الـ list_head الفعلية |
| `get` / `put` | callbacks لزيادة/تقليل الـ reference count عند الـ iteration |

**الـ** `struct klist_node`:

| الـ Field | الوظيفة |
|-----------|---------|
| `n_klist` | pointer للـ klist الأم (private) |
| `n_node` | الـ list_head |
| `n_ref` | الـ kref الخاص بالـ node |

---

#### 6. `struct kobj_uevent_env` (من `kobject.h`)

**الغرض:** حاوية الـ environment variables اللي بتتبعت مع كل uevent.

| الـ Field | الوظيفة |
|-----------|---------|
| `argv[3]` | arguments للـ userspace helper |
| `envp[64]` | pointers للـ variables |
| `buf[2048]` | buffer للكتابة فيه |
| `buflen` | الطول المستخدم حالياً |

---

### رسم العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────┐
                    │         struct bus_type          │
                    │  name, dev_name                  │
                    │  bus_groups / dev_groups /        │
                    │  drv_groups                       │
                    │  match(), probe(), remove() ...   │
                    │  pm → struct dev_pm_ops           │
                    └──────────────┬──────────────────┘
                                   │ kernel stores private data
                                   ▼
                    ┌─────────────────────────────────┐
                    │     struct bus_type_private      │
                    │  (drivers/base/base.h)           │
                    │  subsys  → struct kset ──────────┼──► /sys/bus/<name>/
                    │  drivers_kset → struct kset ─────┼──► /sys/bus/<name>/drivers/
                    │  devices_kset → struct kset ─────┼──► /sys/bus/<name>/devices/
                    │  klist_devices → struct klist ───┼──► [device₁] ↔ [device₂] ↔ ...
                    │  klist_drivers → struct klist ───┼──► [driver₁] ↔ [driver₂] ↔ ...
                    │  bus_notifier → blocking chain   │
                    └─────────────────────────────────┘

  struct kset
  ┌──────────────────────┐
  │  list (list_head)    │◄──── يحتوي على kobjects
  │  list_lock (spinlock)│
  │  kobj (kobject) ─────┼──► parent kobject في sysfs hierarchy
  │  uevent_ops          │
  └──────────────────────┘

  struct kobject
  ┌──────────────────────┐
  │  name                │
  │  parent ─────────────┼──► kobject أعلى في الشجرة
  │  kset  ──────────────┼──► المجموعة اللي ينتمي ليها
  │  ktype ──────────────┼──► kobj_type (release, sysfs_ops)
  │  sd ─────────────────┼──► kernfs_node (sysfs entry)
  │  kref                │    (reference count)
  └──────────────────────┘

  struct bus_attribute
  ┌──────────────────────┐
  │  attr (attribute)    │◄─── name + mode
  │  show()              │
  │  store()             │
  └──────────────────────┘
       embedded في /sys/bus/<name>/<attr_name>
```

---

### دورة حياة الـ Bus — Lifecycle Diagram

```
[Module/Subsystem Init]
        │
        ▼
  bus_register(bus)
        │
        ├── يخلق bus_type_private
        ├── يعمل kset_register للـ subsys_kset  ──► /sys/bus/<name>/
        ├── يخلق devices_kset                   ──► /sys/bus/<name>/devices/
        ├── يخلق drivers_kset                   ──► /sys/bus/<name>/drivers/
        ├── يعمل bus_create_file للـ uevent attr
        ├── يضيف bus_groups attributes
        └── يسجل في bus_kset العالمي
        │
        ▼
  [Bus Ready — يقبل devices وdrivers]
        │
        ├─── device_register(dev)
        │         ├── dev->bus = bus
        │         ├── بيضاف لـ klist_devices
        │         └── بيحاول match مع كل driver
        │
        ├─── driver_register(drv)
        │         ├── drv->bus = bus
        │         ├── بيضاف لـ klist_drivers
        │         └── بيحاول match مع كل device
        │
        ├─── bus_for_each_dev() / bus_find_device()
        │         └── iteration آمن على klist_devices
        │
        ├─── bus_register_notifier()
        │         └── بيضيف notifier_block للـ bus_notifier chain
        │
        ▼
  bus_unregister(bus)
        │
        ├── بيشيل كل attributes
        ├── بيدمر devices_kset و drivers_kset
        ├── بيشيل من bus_kset
        └── بيحرر bus_type_private
        │
        ▼
  [Bus Gone]
```

---

### Call Flow Diagrams

#### 1. تسجيل Bus جديد

```
bus_register(bus)
  → bus_type_private = kzalloc()
  → kset_init(priv->subsys)
  → kobject_set_name(&priv->subsys.kobj, "%s", bus->name)
  → kset_register(&priv->subsys)
      → kobject_add(&kset->kobj, parent, ...)
          → kernfs_create_dir()   ← /sys/bus/<name>/
  → kset_create_and_add("devices", ...)
  → kset_create_and_add("drivers", ...)
  → bus_create_file(bus, &bus_attr_uevent)
  → bus_add_groups(bus, bus->bus_groups)
  → bus_add_attrs(bus)
```

#### 2. مطابقة Device بـ Driver (Probe Flow)

```
device_add(dev)
  → bus_probe_device(dev)
      → device_initial_probe(dev)
          → __device_attach(dev)
              → bus_for_each_drv(bus, ...)
                  → __device_attach_driver(drv, dev)
                      → driver_match_device(drv, dev)
                          → bus->match(dev, drv)        ← bus callback
                              → returns 1 if match
                      → driver_probe_device(drv, dev)
                          → bus->probe(dev)             ← bus callback
                              → drv->probe(dev)         ← driver callback
                                  → hardware init
```

#### 3. Bus Find Device

```
bus_find_device(bus, start, data, match_fn)
  → klist_iter_init_node(&bus_priv->klist_devices, &i, start->p->knode_bus)
  → loop:
      klist_next(&i)
          → next klist_node
          → container_of → device
      match_fn(dev, data)
          → device_match_name() / device_match_of_node() / ...
              → returns 1 if found
  → klist_iter_exit(&i)   ← يحرر reference
  → returns device أو NULL
```

#### 4. Bus Notifier Event Flow

```
[event happens, e.g., device added]
  → blocking_notifier_call_chain(&bus->p->bus_notifier, event, dev)
      → for each notifier_block nb in chain:
          nb->notifier_call(nb, BUS_NOTIFY_ADD_DEVICE, dev)
              → custom code في الـ subsystem أو الـ module
```

#### 5. Bus Attribute Read من sysfs

```
user: cat /sys/bus/pci/uevent
  → kernfs_fop_read()
      → sysfs_kf_seq_show()
          → bus_attr_show()
              → attr->show(bus, buf)    ← bus_attribute.show callback
                  → fills buf
  → returns data to user
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة وما تحميه

| الـ Lock | النوع | بيحمي إيه |
|----------|-------|-----------|
| `bus_type_private->subsys.kobj` kobject lock | mutex (داخل kobject) | الـ sysfs operations على الـ bus |
| `klist_devices.k_lock` | spinlock | إضافة/شيل من قائمة الـ devices |
| `klist_drivers.k_lock` | spinlock | إضافة/شيل من قائمة الـ drivers |
| `device->mutex` | mutex | الـ probe/remove/bind operations على device معين |
| `device->parent->mutex` | mutex (لو `need_parent_lock=true`) | الـ probe/remove لما الـ bus يحتاج lock على الأب |
| `kset.list_lock` | spinlock | قائمة kobjects داخل الـ kset |

#### ترتيب الـ Locking (Lock Ordering)

```
[الأعلى مستوى ← يتقفل أول]

  bus_type_private mutex (bus-level operations)
      │
      ▼
  device->parent->mutex  (لو need_parent_lock = true)
      │
      ▼
  device->mutex  (probe/remove/bind/unbind)
      │
      ▼
  klist spinlock  (إضافة/شيل من القائمة — قصير جداً)
```

> **تحذير مهم:** الـ bus notifier callbacks بيتنادوا وهو ماسك `device->mutex` في الغالب — لازم تتجنب أي lock acquisition ممكن يعمل deadlock.

#### الـ klist والـ Reference Counting

**الـ** `klist` بيضمن إن iteration آمن حتى لو حصل remove في نفس الوقت:

```
klist_iter_init()
    → بياخد reference على الـ node الحالي (kref_get)

klist_next() / klist_prev()
    → بيحرر reference على الـ node السابق
    → بياخد reference على الـ node الجديد

klist_iter_exit()
    → بيحرر reference على آخر node
```

يعني لو device اتشال وهو في الـ iteration — الـ node هيفضل في الذاكرة لحد ما الـ iterator يخلص، وبعدين يتحرر. ده بيمنع use-after-free.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function | الغرض |
|---|---|
| `bus_register()` | تسجيل bus جديد في driver model |
| `bus_unregister()` | إزالة bus من driver model |
| `bus_rescan_devices()` | إعادة مسح الـ devices على الـ bus |

#### Attribute Management

| Function | الغرض |
|---|---|
| `bus_create_file()` | إنشاء sysfs attribute file للـ bus |
| `bus_remove_file()` | حذف sysfs attribute file من الـ bus |

#### Device Iteration & Search

| Function | الغرض |
|---|---|
| `bus_for_each_dev()` | تكرار على كل الـ devices في الـ bus |
| `bus_find_device()` | البحث عن device بـ match callback مخصص |
| `bus_find_device_reverse()` | بحث عكسي في قائمة الـ devices |
| `bus_find_device_by_name()` | البحث بالاسم (inline wrapper) |
| `bus_find_device_by_of_node()` | البحث بـ DT node (inline wrapper) |
| `bus_find_device_by_fwnode()` | البحث بـ firmware node (inline wrapper) |
| `bus_find_device_by_devt()` | البحث بـ dev_t (inline wrapper) |
| `bus_find_next_device()` | الـ device التالي بعد device معين |
| `bus_find_device_by_acpi_dev()` | البحث بـ ACPI companion device |

#### Driver Iteration

| Function | الغرض |
|---|---|
| `bus_for_each_drv()` | تكرار على كل الـ drivers المسجلة في الـ bus |
| `bus_sort_breadthfirst()` | ترتيب الـ devices بـ BFS order |

#### Notifier

| Function | الغرض |
|---|---|
| `bus_register_notifier()` | تسجيل notifier block لأحداث الـ bus |
| `bus_unregister_notifier()` | إلغاء تسجيل notifier block |

#### Device Matching Helpers

| Function | الغرض |
|---|---|
| `device_match_name()` | مقارنة بالاسم |
| `device_match_type()` | مقارنة بالـ device_type |
| `device_match_of_node()` | مقارنة بـ DT node |
| `device_match_fwnode()` | مقارنة بـ fwnode_handle |
| `device_match_devt()` | مقارنة بـ dev_t |
| `device_match_acpi_dev()` | مقارنة بـ ACPI device |
| `device_match_acpi_handle()` | مقارنة بـ ACPI handle |
| `device_match_any()` | تطابق دائمًا (first match) |

#### Utility

| Function | الغرض |
|---|---|
| `bus_get_kset()` | الحصول على الـ kset الخاص بالـ bus |
| `bus_get_dev_root()` | الحصول على الـ root device للـ bus |

---

### Group 1: Registration & Lifecycle

هذه المجموعة مسؤولة عن إنشاء وإزالة الـ bus من الـ driver model. عند `bus_register()`، الـ kernel يبني الـ sysfs hierarchy (مثل `/sys/bus/platform/`) ويهيئ الـ internal structures. عند `bus_unregister()` يتم تنظيف كل شيء.

---

#### `bus_register`

```c
int __must_check bus_register(const struct bus_type *bus);
```

بتسجّل الـ `bus_type` في الـ driver model. بتعمل `kset` للـ bus نفسه، وـ `kset` للـ devices، وـ `kset` للـ drivers، وبتضيف الـ `bus_groups` attributes في sysfs. لو فشل أي خطوة داخلية، بترجع error code سلبي.

**Parameters:**
- `bus` — pointer للـ `bus_type` struct المكتوب ببيانات الـ bus (الاسم، الـ callbacks، الـ groups)

**Return:**
- `0` عند النجاح
- error code سلبي (مثل `-ENOMEM`, `-EEXIST`) عند الفشل
- مميز بـ `__must_check` — لازم تتحقق من النتيجة

**Key Details:**
- يُستدعى من `__init` context عادةً (module init أو kernel init)
- داخليًا بيستخدم `bus_kset` الـ global كـ parent
- بيهيّئ `struct subsys_private` المخصص للـ bus (مش exposed في الـ header ده)
- الـ `bus->name` لازم يكون unique وإلا فشل بـ `-EEXIST`

**Caller Context:** Module init أو core kernel init (مثلاً `platform_bus_init()`, `pci_driver_init()`)

**Pseudocode:**
```
bus_register(bus):
    alloc subsys_private priv
    priv->bus = bus
    kset_register(&priv->subsys)          // /sys/bus/<name>/
    kset_create_and_add("devices", ...)   // /sys/bus/<name>/devices/
    kset_create_and_add("drivers", ...)   // /sys/bus/<name>/drivers/
    add_probe_files(bus)                  // uevent, drivers_autoprobe
    bus_add_groups(bus, bus->bus_groups)
    return 0
```

---

#### `bus_unregister`

```c
void bus_unregister(const struct bus_type *bus);
```

بتعكس كل اللي عمله `bus_register()`. بتشيل الـ sysfs entries وبتحرر الـ `subsys_private`. لازم تتأكد إن مفيش devices أو drivers لسه مسجلين على الـ bus قبل الاستدعاء.

**Parameters:**
- `bus` — نفس الـ pointer اللي تم تسجيله بـ `bus_register()`

**Return:** void

**Key Details:**
- لا تستدعيها لو في devices/drivers لسه على الـ bus — ده undefined behavior
- بترسل uevent `KOBJ_REMOVE` للـ bus kobject
- الـ `subsys_private` بيتحرر عبر kobject reference counting

---

#### `bus_rescan_devices`

```c
int __must_check bus_rescan_devices(const struct bus_type *bus);
```

بتحاول تربط كل device غير مربوط (`unbound`) على الـ bus بـ driver مناسب. مفيدة لما يتم تحميل driver جديد أو تغيير حالة binding.

**Parameters:**
- `bus` — الـ bus المطلوب إعادة مسحه

**Return:**
- `0` عند النجاح
- error code لو فشل أي device في الـ probe

**Key Details:**
- داخليًا بتعمل `bus_for_each_dev()` وبتستدعي `device_attach()` على كل device
- مش بتفصل الـ devices المربوطة بالفعل

---

### Group 2: Attribute Management (sysfs)

الـ `bus_attribute` بيسمح بإنشاء files مخصصة في `/sys/bus/<name>/`. مفيد لـ debugging أو لـ userspace control.

---

#### `BUS_ATTR_RW` / `BUS_ATTR_RO` / `BUS_ATTR_WO` — Macros

```c
#define BUS_ATTR_RW(_name) \
    struct bus_attribute bus_attr_##_name = __ATTR_RW(_name)
#define BUS_ATTR_RO(_name) \
    struct bus_attribute bus_attr_##_name = __ATTR_RO(_name)
#define BUS_ATTR_WO(_name) \
    struct bus_attribute bus_attr_##_name = __ATTR_WO(_name)
```

Convenience macros لإنشاء `bus_attribute` objects. بتربط الـ `show`/`store` callbacks تلقائيًا باستخدام naming convention (`<name>_show`, `<name>_store`).

**مثال واقعي:**
```c
// driver يعرّف:
static ssize_t my_attr_show(const struct bus_type *bus, char *buf) {
    return sprintf(buf, "value\n");
}
static BUS_ATTR_RO(my_attr);  // ينتج bus_attr_my_attr
```

---

#### `bus_create_file`

```c
int __must_check bus_create_file(const struct bus_type *bus,
                                  struct bus_attribute *attr);
```

بتضيف الـ `bus_attribute` كـ sysfs file تحت `/sys/bus/<name>/`. بيستدعي `sysfs_create_file()` على الـ bus kobject.

**Parameters:**
- `bus` — الـ bus المستهدف
- `attr` — الـ attribute المراد إضافتها (لازم تكون initialized)

**Return:**
- `0` عند النجاح
- `-ENOMEM` أو `-EEXIST` أو غيرها عند الفشل

**Key Details:**
- بتستدعيها بعد `bus_register()` مباشرةً
- الـ `attr->attr.mode` بيحدد permissions الـ file

---

#### `bus_remove_file`

```c
void bus_remove_file(const struct bus_type *bus, struct bus_attribute *attr);
```

بتشيل الـ sysfs file المضافة سابقًا بـ `bus_create_file()`. بتستدعي `sysfs_remove_file()` داخليًا.

**Parameters:**
- `bus` — الـ bus المستهدف
- `attr` — الـ attribute المراد حذفها

**Return:** void

---

### Group 3: Device Matching Helpers

الـ `device_match_t` هو typedef للـ callback المستخدم في البحث:

```c
typedef int (*device_match_t)(struct device *dev, const void *data);
```

كل function في المجموعة دي بترجع nonzero لو الـ device يطابق الـ criteria، وصفر لو لأ.

---

#### `device_match_name`

```c
int device_match_name(struct device *dev, const void *name);
```

بتقارن `dev_name(dev)` بالـ string المعطى. بتستخدم `strcmp()` داخليًا.

**Parameters:**
- `dev` — الـ device المراد فحصه
- `name` — `const char *` cast إلى `const void *`

**Return:** nonzero لو الأسماء متطابقة، صفر لو لأ

---

#### `device_match_type`

```c
int device_match_type(struct device *dev, const void *type);
```

بتقارن `dev->type` بالـ pointer المعطى. مقارنة pointer-to-pointer (مش strcmp).

**Parameters:**
- `dev` — الـ device
- `type` — `const struct device_type *` cast إلى `const void *`

---

#### `device_match_of_node`

```c
int device_match_of_node(struct device *dev, const void *np);
```

بتتحقق إن `dev->of_node == np`. مهمة جدًا في embedded systems عند البحث بـ Device Tree node.

**Parameters:**
- `dev` — الـ device
- `np` — `const struct device_node *` cast إلى `const void *`

---

#### `device_match_fwnode`

```c
int device_match_fwnode(struct device *dev, const void *fwnode);
```

بتقارن `dev_fwnode(dev)` بالـ fwnode المعطى. تدعم كل من DT و ACPI عبر الـ firmware node abstraction.

**Parameters:**
- `dev` — الـ device
- `fwnode` — `const struct fwnode_handle *` cast إلى `const void *`

---

#### `device_match_devt`

```c
int device_match_devt(struct device *dev, const void *pdevt);
```

بتتحقق إن `dev->devt == *(dev_t *)pdevt`. مفيدة للبحث بـ major:minor number.

**Parameters:**
- `dev` — الـ device
- `pdevt` — `const dev_t *` cast إلى `const void *`

---

#### `device_match_acpi_dev`

```c
int device_match_acpi_dev(struct device *dev, const void *adev);
```

بتتحقق إن `ACPI_COMPANION(dev) == adev`. متاحة فقط لو `CONFIG_ACPI` معمول بيها.

**Parameters:**
- `dev` — الـ device
- `adev` — `const struct acpi_device *` cast إلى `const void *`

---

#### `device_match_acpi_handle`

```c
int device_match_acpi_handle(struct device *dev, const void *handle);
```

بتتحقق إن الـ ACPI handle للـ device يساوي الـ handle المعطى. مفيدة للبحث بـ ACPI namespace path.

---

#### `device_match_any`

```c
int device_match_any(struct device *dev, const void *unused);
```

بترجع دايمًا `1` (تطابق دائم). مستخدمة كـ match callback في `bus_find_next_device()` للحصول على أول device أو الـ device التالي في القائمة.

---

### Group 4: Device Iteration & Search

---

#### `bus_for_each_dev`

```c
int bus_for_each_dev(const struct bus_type *bus, struct device *start,
                     void *data, device_iter_t fn);
```

بتمشي على كل الـ devices المسجلة على الـ bus وبتستدعي الـ callback `fn` على كل واحد. لو الـ callback رجع nonzero، الـ iteration بتتوقف وبترجع نفس الـ value.

**Parameters:**
- `bus` — الـ bus المستهدف
- `start` — لو مش NULL، الـ iteration بتبدأ بعد الـ device ده (exclusive)
- `data` — pointer بيتمرر للـ callback كـ context
- `fn` — `int (*fn)(struct device *dev, void *data)` — الـ callback على كل device

**Return:**
- `0` لو اتعمل على كل الـ devices من غير توقف
- الـ value اللي رجعها الـ callback لو رجع nonzero (early exit)

**Key Details:**
- بياخد reference (`get_device()`) على كل device قبل استدعاء الـ callback وبيرجعه بعدين — آمن حتى لو الـ device اتشال أثناء الـ iteration
- الـ iteration بتمشي على `klist_devices` داخل `subsys_private`
- الـ `device_iter_t` هو `typedef int (*device_iter_t)(struct device *dev, void *data)`

**مثال واقعي:**
```c
static int find_powered_device(struct device *dev, void *data)
{
    if (device_is_on(dev)) {
        *(struct device **)data = dev;
        return 1; /* stop iteration */
    }
    return 0;
}

struct device *pdev = NULL;
bus_for_each_dev(&my_bus_type, NULL, &pdev, find_powered_device);
```

---

#### `bus_find_device`

```c
struct device *bus_find_device(const struct bus_type *bus,
                               struct device *start,
                               const void *data,
                               device_match_t match);
```

الـ core search function. بتمشي على الـ devices من البداية (أو بعد `start`) وبترجع أول device يرجع فيه الـ `match` callback nonzero.

**Parameters:**
- `bus` — الـ bus المستهدف
- `start` — الـ device اللي الـ search بتبدأ بعده (أو NULL للبداية)
- `data` — بيتمرر للـ `match` callback كـ search criteria
- `match` — callback بيرجع nonzero لو الـ device مطابق

**Return:**
- pointer للـ device المطابق (مع `get_device()` — المستدعي مسؤول عن `put_device()`)
- `NULL` لو مفيش device مطابق

**Key Details:**
- الـ device المرجوع معاه reference count مرفوع — **لازم** تستدعي `put_device()` لما تخلص
- داخليًا بتستدعي `bus_for_each_dev()` مع wrapper يوقف الـ iteration عند أول match

**Pseudocode:**
```
bus_find_device(bus, start, data, match):
    for each dev in bus->devices (after start):
        get_device(dev)
        if match(dev, data) != 0:
            return dev   // caller must put_device()
        put_device(dev)
    return NULL
```

---

#### `bus_find_device_reverse`

```c
struct device *bus_find_device_reverse(const struct bus_type *bus,
                                       struct device *start,
                                       const void *data,
                                       device_match_t match);
```

نفس `bus_find_device()` بس بتمشي من آخر القائمة للأول. مفيدة لما تتوقع إن الـ device المطلوب أضيف مؤخرًا (في نهاية القائمة).

**Parameters:** نفس `bus_find_device()`

**Return:** نفس `bus_find_device()` — reference مرفوع، لازم `put_device()`

---

#### `bus_find_device_by_name` (inline)

```c
static inline struct device *
bus_find_device_by_name(const struct bus_type *bus,
                        struct device *start,
                        const char *name)
{
    return bus_find_device(bus, start, name, device_match_name);
}
```

Wrapper مباشر فوق `bus_find_device()` باستخدام `device_match_name`. بيبحث بالاسم المعطى.

**مثال واقعي:**
```c
struct device *dev = bus_find_device_by_name(&platform_bus_type, NULL, "my-device.0");
if (dev) {
    // استخدم dev
    put_device(dev);
}
```

---

#### `bus_find_device_by_of_node` (inline)

```c
static inline struct device *
bus_find_device_by_of_node(const struct bus_type *bus,
                           const struct device_node *np)
{
    return bus_find_device(bus, NULL, np, device_match_of_node);
}
```

البحث بـ Device Tree node. شائع الاستخدام في الـ platform/SoC drivers لما عندك `np` من الـ DT وعايز الـ device المقابل.

---

#### `bus_find_device_by_fwnode` (inline)

```c
static inline struct device *
bus_find_device_by_fwnode(const struct bus_type *bus,
                          const struct fwnode_handle *fwnode)
{
    return bus_find_device(bus, NULL, fwnode, device_match_fwnode);
}
```

يدعم كل من ACPI و DT عبر الـ `fwnode_handle` abstraction. مفيد لـ cross-platform drivers.

---

#### `bus_find_device_by_devt` (inline)

```c
static inline struct device *
bus_find_device_by_devt(const struct bus_type *bus, dev_t devt)
{
    return bus_find_device(bus, NULL, &devt, device_match_devt);
}
```

البحث بـ major:minor number. مستخدم في character device subsystems.

---

#### `bus_find_next_device` (inline)

```c
static inline struct device *
bus_find_next_device(const struct bus_type *bus, struct device *cur)
{
    return bus_find_device(bus, cur, NULL, device_match_any);
}
```

بترجع الـ device التالي بعد `cur` في قائمة الـ bus. مستخدمة لبناء manual iteration loops بديلًا عن `bus_for_each_dev()`.

---

#### `bus_find_device_by_acpi_dev` (inline, CONFIG_ACPI)

```c
static inline struct device *
bus_find_device_by_acpi_dev(const struct bus_type *bus,
                            const struct acpi_device *adev)
{
    return bus_find_device(bus, NULL, adev, device_match_acpi_dev);
}
```

البحث بـ ACPI companion device. لو `CONFIG_ACPI` مش موجود، بترجع `NULL` دايمًا. مستخدمة في ACPI-aware drivers لإيجاد الـ `struct device` المقابل لـ ACPI namespace object.

---

### Group 5: Driver Iteration

---

#### `bus_for_each_drv`

```c
int bus_for_each_drv(const struct bus_type *bus,
                     struct device_driver *start,
                     void *data,
                     int (*fn)(struct device_driver *, void *));
```

بتمشي على كل الـ drivers المسجلة على الـ bus وبتستدعي `fn` على كل واحد. مشابهة لـ `bus_for_each_dev()` لكن للـ drivers.

**Parameters:**
- `bus` — الـ bus المستهدف
- `start` — لو مش NULL، الـ iteration بتبدأ بعد الـ driver ده
- `data` — context pointer بيتمرر للـ callback
- `fn` — الـ callback على كل driver، يوقف الـ iteration لو رجع nonzero

**Return:**
- `0` لو اتعمل على كل الـ drivers
- nonzero لو الـ callback أوقف الـ iteration مبكرًا

**Key Details:**
- بياخد `driver_lock` على الـ driver قبل استدعاء الـ callback
- الـ iteration على `klist_drivers` في `subsys_private`

---

#### `bus_sort_breadthfirst`

```c
void bus_sort_breadthfirst(const struct bus_type *bus,
                           int (*compare)(const struct device *a,
                                          const struct device *b));
```

بترتب قائمة الـ devices على الـ bus بـ breadth-first order بناءً على الـ `compare` callback. مستخدمة لضمان ترتيب الـ probe (مثلاً PCI bus يرتب devices حسب topology).

**Parameters:**
- `bus` — الـ bus المستهدف
- `compare` — callback بيقارن device `a` بـ device `b`، نفس semantics `qsort`

**Return:** void

**Key Details:**
- بتعمل `klist_sort()` على `klist_devices` في `subsys_private`
- التأثير يظهر على ترتيب `bus_for_each_dev()` الاستدعاءات اللاحقة

---

### Group 6: Bus Notifier

الـ notifier chain على الـ bus بيسمح لـ code خارج الـ driver model إنه يتابع أحداث الـ binding/unbinding وإضافة/إزالة الـ devices.

---

#### `bus_register_notifier`

```c
int bus_register_notifier(const struct bus_type *bus,
                          struct notifier_block *nb);
```

بيضيف الـ `notifier_block` لـ notifier chain الخاص بالـ bus. بعد التسجيل، الـ `nb->notifier_call` بيتستدعى عند كل حدث من `enum bus_notifier_event`.

**Parameters:**
- `bus` — الـ bus المراد متابعته
- `nb` — الـ notifier block المجهز بـ `notifier_call` callback وـ `priority`

**Return:**
- `0` عند النجاح
- error code سلبي عند الفشل

**Key Details:**
- الـ notifier بيتستدعى **مع device lock محتملة** — احذر من deadlock
- الـ `notifier_call` بتستقبل `(bus_notifier_event, struct device *)` كـ arguments
- بتستدعي `blocking_notifier_chain_register()` داخليًا

**مثال واقعي:**
```c
static int my_bus_notifier_call(struct notifier_block *nb,
                                unsigned long event, void *dev)
{
    if (event == BUS_NOTIFY_BIND_DRIVER)
        pr_info("driver binding to %s\n", dev_name(dev));
    return NOTIFY_OK;
}

static struct notifier_block my_nb = {
    .notifier_call = my_bus_notifier_call,
};

// في init:
bus_register_notifier(&platform_bus_type, &my_nb);
```

---

#### `bus_unregister_notifier`

```c
int bus_unregister_notifier(const struct bus_type *bus,
                            struct notifier_block *nb);
```

بتشيل الـ `notifier_block` من الـ chain. لازم تستدعيها قبل unloading الـ module.

**Parameters:**
- `bus` — الـ bus
- `nb` — نفس الـ `notifier_block` المسجل سابقًا

**Return:** `0` عند النجاح، error code عند الفشل

---

#### `enum bus_notifier_event` — أحداث الـ Notifier

| Event | المعنى |
|---|---|
| `BUS_NOTIFY_ADD_DEVICE` | device أُضيف للـ bus |
| `BUS_NOTIFY_DEL_DEVICE` | device على وشك الإزالة |
| `BUS_NOTIFY_REMOVED_DEVICE` | device اتشال بنجاح |
| `BUS_NOTIFY_BIND_DRIVER` | driver على وشك الـ bind |
| `BUS_NOTIFY_BOUND_DRIVER` | driver اتبند بنجاح |
| `BUS_NOTIFY_UNBIND_DRIVER` | driver على وشك الـ unbind |
| `BUS_NOTIFY_UNBOUND_DRIVER` | driver اتفصل بنجاح |
| `BUS_NOTIFY_DRIVER_NOT_BOUND` | driver فشل في الـ bind |

---

### Group 7: Utility Functions

---

#### `bus_get_kset`

```c
struct kset *bus_get_kset(const struct bus_type *bus);
```

بترجع الـ `kset` الخاص بالـ bus subsystem (مش الـ devices أو الـ drivers ksets). مفيدة لو محتاج تعمل operations مباشرة على الـ kobject hierarchy.

**Parameters:**
- `bus` — الـ bus المستهدف

**Return:** pointer للـ `kset` الخاص بالـ bus

---

#### `bus_get_dev_root`

```c
struct device *bus_get_dev_root(const struct bus_type *bus);
```

بترجع الـ root device للـ bus (الـ `dev_root` في `subsys_private`). ده الـ device المستخدم كـ parent لكل الـ devices على الـ bus في الـ sysfs hierarchy.

**Parameters:**
- `bus` — الـ bus المستهدف

**Return:**
- pointer للـ root device (مع reference مرفوع — لازم `put_device()`)
- `NULL` لو مفيش root device

---

### الـ `struct bus_type` — Callbacks المهمة

الـ callbacks التالية في `struct bus_type` بتتستدعى من الـ driver core في سياقات محددة:

| Callback | متى بيتستدعى |
|---|---|
| `match(dev, drv)` | عند إضافة device أو driver جديد |
| `probe(dev)` | بعد `match()` ناجح لبدء الـ initialization |
| `remove(dev)` | عند إزالة device |
| `shutdown(dev)` | عند system shutdown |
| `sync_state(dev)` | بعد ما كل الـ consumers يتبندوا |
| `uevent(dev, env)` | عند إرسال uevent لـ userspace |
| `suspend(dev, state)` | عند دخول الـ device في sleep |
| `resume(dev)` | عند استيقاظ الـ device |
| `online(dev)` | بعد hot-plug |
| `offline(dev)` | قبل hot-remove |
| `num_vf(dev)` | للاستعلام عن SR-IOV virtual functions |
| `dma_configure(dev)` | لضبط DMA parameters |
| `dma_cleanup(dev)` | لتنظيف DMA configuration |
| `irq_get_affinity(dev, irq_vec)` | للاستعلام عن IRQ affinity mask |

**الـ `need_parent_lock` flag:**
لو `true`، الـ driver core بياخد lock على الـ parent device قبل الـ probe أو الـ remove. مهم للـ buses اللي فيها parent-child dependency (مثلاً USB).

---

### ASCII Diagram — Bus Registration Flow

```
bus_register(bus)
       │
       ▼
alloc subsys_private
       │
       ├──► kset_register()        ──► /sys/bus/<name>/
       │
       ├──► kset_create_and_add()  ──► /sys/bus/<name>/devices/
       │
       ├──► kset_create_and_add()  ──► /sys/bus/<name>/drivers/
       │
       ├──► add_probe_files()      ──► /sys/bus/<name>/uevent
       │                               /sys/bus/<name>/drivers_autoprobe
       │
       └──► bus_add_groups()       ──► custom bus_groups attributes
```

### ASCII Diagram — Device Search Flow

```
bus_find_device(bus, start, data, match)
        │
        ▼
  klist_iter_init(klist_devices)
        │
        ▼
  ┌─────────────────────────────┐
  │  next = klist_next()        │
  │  if !next → return NULL     │
  │                             │
  │  get_device(dev)            │
  │  if match(dev, data) != 0:  │
  │      return dev  ◄──────────┼── caller must put_device()
  │  put_device(dev)            │
  └─────────────────────────────┘
```
## Phase 5: دليل الـ Debugging الشامل

الـ `bus_type` هو قلب الـ driver model في Linux — أي مشكلة في الـ probe أو الـ bind أو الـ match هتظهر هنا. الدليل ده بيغطي كل أدوات الـ debugging من الـ sysfs لحد الـ logic analyzer.

---

### Software Level

#### 1. debugfs entries

الـ driver core مش بيعمل entries خاصة بيه في الـ debugfs، لكن الـ kobject/kset infrastructure بتفضح بيانات مفيدة جداً:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ buses المسجلة
ls /sys/kernel/debug/devices/

# الـ driver_probe_defer list — الأجهزة اللي اتأجل الـ probe بتاعها
cat /sys/kernel/debug/devices/devices_deferred

# عدد الـ deferred probes الحالية
cat /sys/kernel/debug/devices/devices_deferred | wc -l
```

مثال عملي — لو جهاز عمل `-EPROBE_DEFER`:
```
$ cat /sys/kernel/debug/devices/devices_deferred
soc:uart@ff000000                        Driver my_uart needs clk not ready yet
```

---

#### 2. sysfs entries

الـ `bus_register()` بتعمل تلقائياً hierarchy في `/sys/bus/`:

```bash
# قائمة كل الـ buses المسجلة في الـ kernel
ls /sys/bus/

# الأجهزة على bus معين (مثلاً platform)
ls /sys/bus/platform/devices/

# الـ drivers المسجلة على bus معين
ls /sys/bus/platform/drivers/

# هل device معين معاه driver ولا لأ؟ (symlink موجود = bound)
ls -la /sys/bus/platform/devices/my_device/driver

# rescan الـ bus يدوياً
echo 1 > /sys/bus/platform/rescan

# unbind device من driver
echo "my_device" > /sys/bus/platform/drivers/my_driver/unbind

# bind device لـ driver يدوياً
echo "my_device" > /sys/bus/platform/drivers/my_driver/bind

# الـ uevent للـ device
cat /sys/bus/platform/devices/my_device/uevent

# الـ attributes اللي الـ bus_groups بتوفرها
ls /sys/bus/platform/
```

جدول الـ sysfs paths المهمة:

| Path | المحتوى |
|------|---------|
| `/sys/bus/<name>/devices/` | كل الأجهزة على الـ bus |
| `/sys/bus/<name>/drivers/` | كل الـ drivers على الـ bus |
| `/sys/bus/<name>/drivers_probe` | trigger manual probe |
| `/sys/bus/<name>/drivers_autoprobe` | enable/disable autoprobe |
| `/sys/bus/<name>/uevent` | trigger uevent يدوياً |
| `/sys/devices/.../driver` | symlink للـ driver اللي bind ده الجهاز |
| `/sys/devices/.../subsystem` | symlink للـ bus |

---

#### 3. ftrace — tracepoints وـ events

الـ driver core عنده tracepoints مباشرة في الـ probe/bind path:

```bash
# شوف كل الـ events المتعلقة بالـ device/driver
ls /sys/kernel/tracing/events/devres/
ls /sys/kernel/tracing/events/rpm/        # runtime PM

# enable الـ probe events
echo 1 > /sys/kernel/tracing/events/enable  # كل حاجة (تقيل)

# أو بشكل انتقائي — track الـ driver probe
echo 1 > /sys/kernel/tracing/events/rpm/rpm_idle/enable
echo 1 > /sys/kernel/tracing/events/rpm/rpm_suspend/enable
echo 1 > /sys/kernel/tracing/events/rpm/rpm_resume/enable

# filter على device معين
echo 'name == "my_device"' > /sys/kernel/tracing/events/rpm/rpm_idle/filter

# function tracer — trace مسار الـ probe
echo function > /sys/kernel/tracing/current_tracer
echo "really_probe" > /sys/kernel/tracing/set_ftrace_filter
echo "bus_probe_device" >> /sys/kernel/tracing/set_ftrace_filter
echo "driver_probe_device" >> /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on

# قرأ الـ trace
cat /sys/kernel/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on
echo nop > /sys/kernel/tracing/current_tracer
```

مثال على output مفيد:
```
# TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
# modprobe-1234  [000] .....  12.345678: really_probe: device my_uart driver my_uart_drv
# modprobe-1234  [000] .....  12.345700: really_probe <-- driver_probe_device
```

---

#### 4. printk وـ dynamic debug

الـ driver core بيستخدم `pr_debug` و `dev_dbg` بكثرة — بتفعّلها بدون recompile:

```bash
# enable dynamic debug لكل drivers/base/
echo "file drivers/base/bus.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/dd.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/core.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو enable كل حاجة في الـ driver core دفعة واحدة
echo "module=drivers/base +p" > /sys/kernel/debug/dynamic_debug/control

# enable الـ dev_dbg بتاع driver معين
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# اتأكد إن الـ dynamic debug شغال
cat /sys/kernel/debug/dynamic_debug/control | grep "bus.c"

# kernel log level — اعمل verbose
dmesg -n 7    # KERN_DEBUG level
echo 7 > /proc/sys/kernel/printk
```

لـ boot-time debugging لو المشكلة قبل الـ sysfs:
```bash
# في kernel command line
dyndbg="file drivers/base/bus.c +p; file drivers/base/dd.c +p"
```

---

#### 5. Kernel config options للـ debugging

| CONFIG | الوظيفة |
|--------|--------|
| `CONFIG_DEBUG_DRIVER` | يفعّل `pr_debug` في driver core كله |
| `CONFIG_DEBUG_DEVRES` | يتبع الـ devres allocations ويكشف leaks |
| `CONFIG_PROVE_LOCKING` | lockdep — يكتشف deadlocks في الـ bus lock |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يكتشف use-after-free في الـ kobject release |
| `CONFIG_DEBUG_KOBJECT` | verbose logging لكل kobject operations |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg`/`pr_debug` بدون recompile |
| `CONFIG_BUS_NOTIFY_TRACE` | trace الـ bus notifier callbacks |
| `CONFIG_SAMPLES` | أمثلة كاملة لـ bus/driver |
| `CONFIG_PM_DEBUG` | verbose power management على الـ bus |
| `CONFIG_PM_ADVANCED_DEBUG` | sysfs entries إضافية لكل device PM state |
| `CONFIG_ACPI_DEBUG` | لو الـ bus بيستخدم ACPI matching |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep CONFIG_DEBUG_DRIVER
zcat /proc/config.gz | grep CONFIG_DEBUG_DEVRES
zcat /proc/config.gz | grep CONFIG_PROVE_LOCKING
```

---

#### 6. أدوات خاصة بالـ subsystem

**udevadm** — monitor الـ bus events في الـ userspace:
```bash
# monitor كل الـ uevent events على الـ bus
udevadm monitor --kernel --udev

# simulate uevent
udevadm test /sys/bus/platform/devices/my_device

# decode uevent environment
udevadm info --query=all --path=/sys/bus/platform/devices/my_device

# trigger rescan بالـ uevent
udevadm trigger --subsystem-match=platform
```

**busybox devmem / iomem** للـ bus registers:
```bash
# اقرأ physical register
devmem2 0xFF000000 w    # word read
devmem2 0xFF000000 b    # byte read

# قائمة كل الـ IO regions
cat /proc/iomem

# قائمة كل الـ IO ports
cat /proc/ioports
```

**كشف الـ driver binding:**
```bash
# اعرف كل device مش عنده driver (unbound)
for d in /sys/bus/*/devices/*; do
  [ ! -L "$d/driver" ] && echo "UNBOUND: $d"
done
```

---

#### 7. Common error messages

| رسالة الـ kernel log | المعنى | الحل |
|---------------------|--------|------|
| `bus: 'platform': really_probe: probe of my_dev failed with error -19` | الـ driver مش شايل الجهاز ده (`-ENODEV`) | تحقق من `match()` وـ `of_match_table` |
| `bus: 'platform': really_probe: probe of my_dev failed with error -517` | `-EPROBE_DEFER` — dependency مش جاهزة | انتظر أو تحقق من الـ dependency chain في `devices_deferred` |
| `Driver 'my_driver' needs to be in list of trusted adapters` | bus security policy رفض الـ driver | راجع `bus_type->match()` return value |
| `kobject: 'my_dev' failed to add 'uevent' sysfs file` | مشكلة في الـ sysfs registration | تحقق من `CONFIG_DEBUG_KOBJECT` والـ kernel log |
| `WARNING: CPU: 0 PID: 1 at drivers/base/dd.c:xxx` | deadlock أو سوء استخدام الـ device lock | فعّل `CONFIG_PROVE_LOCKING` |
| `bus_register called twice for platform` | الـ bus اتسجل أكتر من مرة | تأكد من `__initcall` ordering |
| `Unable to handle kernel NULL pointer dereference` في `bus_probe_device` | `bus_type->probe` أو `drv->probe` NULL | تحقق من الـ driver initialization |
| `device_add: bus 'my_bus' not found` | الـ bus اتسجل بعد الجهاز | صحح الـ initcall order |

---

#### 8. Strategic WARN_ON / dump_stack placement

النقاط الاستراتيجية اللي تضع فيها `WARN_ON()` أو `dump_stack()`:

```c
/* في bus_type->match() — لو match بيرجع قيمة غريبة */
int my_bus_match(struct device *dev, const struct device_driver *drv)
{
    int ret = do_real_match(dev, drv);
    /* detect unexpected negative values outside -EPROBE_DEFER */
    WARN_ON(ret < 0 && ret != -EPROBE_DEFER);
    return ret;
}

/* في bus_type->probe() — تحقق من state consistency */
int my_bus_probe(struct device *dev)
{
    /* bus data يجب يكون موجود دايماً وقت الـ probe */
    WARN_ON(!dev->bus);
    WARN_ON(!dev->driver);
    return do_probe(dev);
}

/* في bus_type->remove() — تحقق من cleanup صح */
void my_bus_remove(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    /* لو الـ probe اتنفذ لازم يكون في priv */
    if (WARN_ON(!priv))
        return;
    do_cleanup(priv);
}

/* في الـ notifier — track unexpected transitions */
static int my_bus_notifier(struct notifier_block *nb,
                            unsigned long event, void *data)
{
    struct device *dev = data;
    if (event == BUS_NOTIFY_DRIVER_NOT_BOUND) {
        /* هنا يفيد dump_stack لفهم من طلب الـ bind */
        dev_warn(dev, "driver failed to bind\n");
        dump_stack();
    }
    return NOTIFY_DONE;
}
```

---

### Hardware Level

#### 1. التحقق من أن hardware state يطابق kernel state

الهدف: الـ kernel يقول الجهاز bound — هل الـ hardware فعلاً شغال؟

```bash
# تحقق من الـ bind state في sysfs
ls -la /sys/bus/platform/devices/my_device/driver
# لو symlink موجود → kernel يعتبره bound

# تحقق من الـ power state
cat /sys/bus/platform/devices/my_device/power/runtime_status
# expected: "active" بعد probe ناجح

# تحقق من الـ PM runtime counters
cat /sys/bus/platform/devices/my_device/power/runtime_usage
cat /sys/bus/platform/devices/my_device/power/runtime_active_time

# مقارنة: هل الـ clocks فعلاً enabled؟
cat /sys/kernel/debug/clk/clk_summary | grep my_device_clk

# هل الـ regulators شغالة؟
cat /sys/kernel/debug/regulator/regulator_summary | grep my_dev
```

---

#### 2. Register dump techniques

```bash
# devmem2 — اقرأ/اكتب registers فيزيائية
# اقرأ control register
devmem2 0xFE200000 w

# اكتب قيمة لـ register
devmem2 0xFE200000 w 0x00000001

# /dev/mem مع Python لـ block read
python3 - <<'EOF'
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 0x1000, offset=0xFE200000)
    for i in range(0, 0x40, 4):
        val = struct.unpack('<I', mm[i:i+4])[0]
        print(f"[0x{0xFE200000+i:08x}] = 0x{val:08x}")
    mm.close()
EOF

# io tool (من package 'io' أو 'iotools')
io -4 -r 0xFE200000    # read 32-bit

# لو الـ driver استخدم ioremap — اعمل custom debugfs
# في الـ driver code:
debugfs_create_regset32("regs", 0444, dbg_dir, &my_regset);
# ثم:
cat /sys/kernel/debug/my_device/regs
```

---

#### 3. Logic Analyzer وـ Oscilloscope tips

للـ bus protocols الشائعة:

**I2C Bus (عبر bus_type):**
```
Logic Analyzer setup:
- Channel 0: SDA
- Channel 1: SCL
- Trigger: SDA falling أثناء SCL high = START condition
- Sample rate: 4x أعلى من bus frequency (مثلاً 1.6 MHz لـ 400 KHz I2C)
- ابحث عن: ACK/NACK bits — NACK = device مش موجود أو address غلط
```

**SPI Bus:**
```
Logic Analyzer setup:
- Channels: MOSI, MISO, CLK, CS
- Trigger: CS falling edge
- ابحث عن: MISO all-zeros أو all-ones = device مش respond
```

**Platform Bus (GPIO/MMIO):**
```
Oscilloscope:
- Probe على الـ interrupt line
- Probe على الـ chip enable
- تحقق من الـ voltage levels: 3.3V vs 1.8V mismatch يسبب no-response
```

ربط الـ hardware observations بالـ kernel log:
```bash
# فعّل kernel timestamps بـ microseconds precision
echo 1 > /sys/kernel/debug/tracing/options/irqsoff-tracer
# أو
dmesg -T    # human-readable timestamps
dmesg --follow    # real-time مع logic analyzer في نفس الوقت
```

---

#### 4. Common hardware issues وـ kernel log patterns

| المشكلة | Pattern في kernel log | الحل |
|--------|----------------------|------|
| الجهاز مش موجود فيزيائياً | `probe of device failed with error -19 (ENODEV)` | تحقق من power supply والـ bus wiring |
| Clock مش enabled | `failed to get clock: -ENOENT` أو stuck في probe | تحقق من الـ clk tree بـ `clk_summary` |
| Interrupt line مش متوصلة | timeout في الـ probe أو no IRQ fire | `cat /proc/interrupts` وتحقق من الـ IRQ counter |
| Voltage level mismatch | data corruption أو NACK على I2C | قِس الـ VDD بالـ multimeter |
| Bus address conflict | `address 0x50 already in use` | تحقق من DT أو ACPI table |
| Spurious interrupts | kernel log فيه interrupt storm | `cat /proc/interrupts` — counter بيطلع بسرعة |
| DMA buffer not aligned | `WARN: DMA map failed` | تحقق من `dma_configure` في الـ bus_type |

---

#### 5. Device Tree debugging

```bash
# اتحقق من الـ DT اللي الـ kernel قرأه فعلاً (compiled DTB)
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null > /tmp/live_dt.dts
# أو
cat /sys/firmware/devicetree/base/compatible   # root compatible

# تحقق من node بعينه
cat /sys/firmware/devicetree/base/soc/my-device@FF000000/compatible
cat /sys/firmware/devicetree/base/soc/my-device@FF000000/reg  | xxd

# هل الـ device اتتعرف من الـ DT؟
ls /sys/bus/platform/devices/ | grep my-device

# مقارنة الـ DT node مع الـ driver's of_match_table
# الـ driver بيعرّف:
# static const struct of_device_id my_of_match[] = {
#     { .compatible = "vendor,my-device" },
# };
# لازم يطابق بالظبط الـ compatible string في الـ DT

# تحقق من الـ status property
cat /sys/firmware/devicetree/base/soc/my-device@FF000000/status
# "okay" = مفعّل، "disabled" = الـ kernel هيتجاهله

# استخدم of_node_to_nid لـ debugging
ls /sys/bus/platform/devices/my-device.0/of_node    # symlink للـ DT node

# overlay debugging (لو بتستخدم DT overlays)
ls /sys/kernel/config/device-tree/overlays/
cat /sys/kernel/config/device-tree/overlays/my-overlay/status
```

---

### Practical Commands

#### مجموعة أوامر جاهزة — سيناريوهات شائعة

**سيناريو 1: Device موجود في DT بس مش بيعمل probe**

```bash
#!/bin/bash
DEV="my_device.0"
BUS="platform"

echo "=== DT Status ==="
cat /sys/firmware/devicetree/base/soc/${DEV}/status 2>/dev/null || echo "no DT node"

echo "=== Sysfs Registration ==="
ls /sys/bus/${BUS}/devices/ | grep ${DEV} || echo "NOT registered"

echo "=== Driver Binding ==="
ls -la /sys/bus/${BUS}/devices/${DEV}/driver 2>/dev/null || echo "NO driver bound"

echo "=== Power State ==="
cat /sys/bus/${BUS}/devices/${DEV}/power/runtime_status 2>/dev/null

echo "=== Deferred Probes ==="
cat /sys/kernel/debug/devices/devices_deferred | grep ${DEV}

echo "=== Recent dmesg ==="
dmesg | grep -i "${DEV}\|probe\|defer" | tail -20
```

**سيناريو 2: تفعيل كامل للـ debugging قبل load الـ driver**

```bash
#!/bin/bash
# فعّل dynamic debug
echo "file drivers/base/bus.c +pmfl" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/dd.c +pmfl" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/core.c +pmfl" > /sys/kernel/debug/dynamic_debug/control

# فعّل ftrace على probe path
cd /sys/kernel/tracing
echo function_graph > current_tracer
echo "really_probe
bus_probe_device
driver_probe_device
__driver_attach
bus_for_each_drv" > set_graph_function
echo 1 > tracing_on

# حمّل الـ driver
modprobe my_driver

# وقف الـ tracing وقرأ النتيجة
echo 0 > tracing_on
cat trace | head -100
```

مثال على output مع تفسيره:
```
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
  0)               |  bus_probe_device() {
  0)               |    device_attach() {
  0)               |      __device_attach() {
  0)   0.250 us    |        driver_match_device();   /* match() returned 1 */
  0)               |        really_probe() {
  0)   1.500 us    |          my_driver_probe();     /* هنا probe الـ driver */
  0)   2.100 us    |        }                        /* returned 0 = success */
```

**سيناريو 3: مراقبة الـ bus notifier events**

```bash
# كتابة kernel module بسيط للـ test (مفيد في التطوير)
# أو استخدام udevadm
udevadm monitor --kernel 2>&1 | while read line; do
    echo "[$(date +%T.%N)] $line"
done &

# trigger event
echo 1 > /sys/bus/platform/rescan

# النتيجة المتوقعة:
# [12:34:56.123456789] KERNEL[12.345678] add      /devices/platform/my_device.0 (platform)
# [12:34:56.234567890] KERNEL[12.456789] bind     /devices/platform/my_device.0 (platform)
```

**سيناريو 4: فحص الـ bus_type attributes**

```bash
BUS="platform"

echo "=== Bus Groups (bus-level attributes) ==="
ls /sys/bus/${BUS}/

echo "=== Dev Groups (per-device attributes) ==="
# اختار device عشوائي وشوف attributes
DEV=$(ls /sys/bus/${BUS}/devices/ | head -1)
ls /sys/bus/${BUS}/devices/${DEV}/

echo "=== Drv Groups (per-driver attributes) ==="
DRV=$(ls /sys/bus/${BUS}/drivers/ | head -1)
ls /sys/bus/${BUS}/drivers/${DRV}/
```

**سيناريو 5: كشف unbound devices بشكل شامل**

```bash
#!/bin/bash
echo "=== Unbound Devices Report ==="
for bus in /sys/bus/*/devices/*/; do
    dev=$(basename $bus)
    bus_name=$(echo $bus | awk -F/ '{print $4}')
    if [ ! -L "${bus}driver" ]; then
        # تحقق من الـ DT status
        dt_status=$(cat "${bus}of_node/status" 2>/dev/null || echo "no-dt")
        echo "UNBOUND | bus: ${bus_name} | dev: ${dev} | dt_status: ${dt_status}"
    fi
done
```

مثال على output وتفسيره:
```
UNBOUND | bus: platform | dev: serial0.0 | dt_status: okay
# → device موجود وmfعّل في DT بس مفيش driver probe عليه
# خطوة التالية: تحقق من compatible string في DT مقابل الـ driver

UNBOUND | bus: platform | dev: leds.0 | dt_status: no-dt
# → device اتسجل manually مش من DT
# خطوة التالية: دور على من عمل platform_device_register لهذا الجهاز
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — driver مش بيتربط بالـ I2C device

#### العنوان
**bus `match` callback بترجع نتيجة غلط وبيمنع probe الـ I2C temperature sensor**

#### السياق
إنت بتعمل bring-up لـ industrial gateway مبني على **RK3562** فيه sensor حرارة **TMP102** على **I2C bus**. الـ device tree موجود صح، الـ driver كمان موجود في الـ kernel، بس الـ device مش بتشتغل خالص.

#### المشكلة
```bash
$ dmesg | grep tmp102
# لا في أي output — الـ driver مش بيعمل probe أصلاً
$ ls /sys/bus/i2c/drivers/tmp102/
# مجلد فاضي — مفيش binding
```

الـ device موجود في sysfs:
```bash
$ ls /sys/bus/i2c/devices/
1-0048  # ده الـ TMP102 على address 0x48
```

بس الـ driver مش بيتربط بيه.

#### التحليل
الكود في `bus.h` بيوضح إن الـ `struct bus_type` عندها callback اسمه `match`:

```c
struct bus_type {
    const char *name;
    // ...
    int (*match)(struct device *dev, const struct device_driver *drv);
    // ...
};
```

الـ `match` callback بتاعة الـ `i2c_bus_type` بتستخدم `of_driver_match_device()` اللي بتقارن `compatible` string في الـ DT بالـ `of_match_table` في الـ driver. المشكلة إن في الـ DT كتبنا:

```dts
// غلط
compatible = "ti,tmp-102";
```

بدل:
```dts
// صح
compatible = "ti,tmp102";
```

الـ `match` callback بترجع `0` (no match)، فالـ driver core مش بيكمل لـ `probe` أصلاً — وده بالضبط اللي `bus_type.match` مسؤول عنه حسب الـ comment في الهيدر:

```c
* @match: Called, perhaps multiple times, whenever a new device or driver
*         is added for this bus. It should return a positive value if the
*         given device can be handled by the given driver and zero
*         otherwise.
```

#### الحل
```dts
/* في ملف الـ DT */
&i2c1 {
    tmp102@48 {
        compatible = "ti,tmp102";  /* صح */
        reg = <0x48>;
        status = "okay";
    };
};
```

للتحقق بعد الحل:
```bash
# تأكد إن الـ match شغال
$ cat /sys/bus/i2c/devices/1-0048/driver
/sys/bus/i2c/drivers/tmp102

# أو استخدم bus iterator للـ debug
$ ls /sys/bus/i2c/drivers/tmp102/
1-0048  # يظهر هنا لو الـ binding تم
```

#### الدرس المستفاد
الـ `match` callback في `bus_type` هي البوابة الأولى — لو رجعت `0`، الـ `probe` مش بيتكلمش أصلاً. أي مشكلة في الـ compatible string أو الـ `of_match_table` بتوقف كل حاجة من الأول. دايماً ابدأ debug من الـ `match` مش من الـ `probe`.

---

### السيناريو 2: Android TV Box على Allwinner H616 — USB device بيتأخر في الـ enumeration بسبب `need_parent_lock`

#### العنوان
**`need_parent_lock = true` بيعمل deadlock في USB hub bring-up على H616**

#### السياق
بتشتغل على Android TV box بـ **Allwinner H616**. الـ USB WiFi dongle (RTL8811CU) مع USB hub بيعمل مشكلة — أحياناً الـ system بتـ hang عند boot لما الـ hub بيعمل enumerate للـ devices اللي تحته.

#### المشكلة
```bash
$ dmesg | grep -i "usb\|hub"
usb 1-1: new high-speed USB device number 2 using xhci-hcd
usb 1-1: New USB device found, ...  (hub)
usb 1-1.1: new high-speed USB device number 3 using xhci-hcd
# hang هنا — مفيش continuation
```

الـ system بيـ freeze ولازم hard reset.

#### التحليل
الـ `struct bus_type` عندها field:

```c
struct bus_type {
    // ...
    bool need_parent_lock;
};
```

الـ comment بيوضح:
```c
* @need_parent_lock: When probing or removing a device on this bus, the
*                    device core should lock the device's parent.
```

الـ USB bus عندها `need_parent_lock = true` لأن الـ hub بيدير children. لما الـ hub نفسه بيكون في الـ middle of probe وبيحاول يـ enumerate الـ child devices، الـ driver core بيحاول يـ lock الـ parent (الـ hub نفسه) عشان يـ probe الـ child — بس الـ hub لسه مش fully initialized، فالـ lock بيكون held بالفعل من thread تاني، وحصل **deadlock**.

المشكلة كانت في custom USB hub driver في الـ BSP بتاع H616 بيعمل `probe` للـ children قبل ما يـ release الـ device lock:

```c
/* في الـ hub driver — BSP code غلط */
static int h616_hub_probe(struct usb_interface *intf, ...)
{
    // ... setup ...
    usb_enumerate_children(intf);  /* ده بيطلب parent lock — deadlock! */
    // ...
}
```

#### الحل
الـ hub driver المصحح لازم يـ defer الـ enumeration:

```c
static int h616_hub_probe(struct usb_interface *intf,
                          const struct usb_device_id *id)
{
    struct h616_hub *hub;

    hub = devm_kzalloc(&intf->dev, sizeof(*hub), GFP_KERNEL);
    if (!hub)
        return -ENOMEM;

    usb_set_intfdata(intf, hub);

    /* defer children enumeration to workqueue — بعيداً عن probe context */
    INIT_WORK(&hub->enum_work, h616_hub_enumerate_work);
    schedule_work(&hub->enum_work);

    return 0;
}
```

للتشخيص:
```bash
# شوف lockdep output
$ dmesg | grep -i "lockdep\|deadlock\|circular"

# أو فعّل lockdep في kernel config
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_LOCKDEP=y
```

#### الدرس المستفاد
الـ `need_parent_lock` في `bus_type` موجود لسبب وجيه — بيحمي الـ hierarchy. بس لو الـ driver بيعمل operations تستدعي lock الـ parent وهو شايل الـ parent lock بالفعل، الـ deadlock حتمي. الـ children enumeration لازم تتعمل في context منفصل عن الـ probe.

---

### السيناريو 3: IoT Sensor على STM32MP1 — `bus_find_device_by_of_node` بترجع NULL

#### العنوان
**كود يستخدم `bus_find_device_by_of_node` للـ SPI bus ومش لاقي الـ device رغم إنه موجود**

#### السياق
بتكتب custom kernel module لـ IoT sensor platform على **STM32MP1**. الـ module محتاج يلاقي الـ SPI flash device (W25Q128) المتصل بـ **SPI1** عشان يعمل له direct access من driver تاني (inter-driver communication pattern).

#### المشكلة
```c
/* في الـ module code */
struct device_node *np = of_find_node_by_name(NULL, "w25q128");
struct device *dev = bus_find_device_by_of_node(&spi_bus_type, np);
if (!dev) {
    pr_err("W25Q128 not found on SPI bus!\n");
    return -ENODEV;
}
```

الـ log بيقول:
```
W25Q128 not found on SPI bus!
```

رغم إن الـ device واضح في sysfs:
```bash
$ ls /sys/bus/spi/devices/
spi0.0  spi1.0  spi1.1
```

#### التحليل
الـ `bus_find_device_by_of_node` في `bus.h` معرّفة كالآتي:

```c
static inline struct device *
bus_find_device_by_of_node(const struct bus_type *bus,
                           const struct device_node *np)
{
    return bus_find_device(bus, NULL, np, device_match_of_node);
}
```

الـ `device_match_of_node` بتقارن `dev->of_node` بالـ `np` اللي إنت بتديه. المشكلة في إزاي إنت جبت الـ `np`:

```c
/* المشكلة هنا */
struct device_node *np = of_find_node_by_name(NULL, "w25q128");
```

الـ `of_find_node_by_name` بتدور على الـ node باسم الـ node في الـ DT — وده بيعتمد على اسم الـ node مش الـ compatible. في الـ DT اسم الـ node كان:

```dts
/* في الـ DT */
&spi1 {
    flash@0 {                        /* اسم الـ node = "flash" */
        compatible = "winbond,w25q128";
        reg = <0>;
    };
};
```

فالـ `of_find_node_by_name(NULL, "w25q128")` بترجع `NULL` لأن الاسم هو `"flash"` مش `"w25q128"`. لما الـ `np` يكون `NULL`، الـ `device_match_of_node` مش بتعمل match لأي device.

#### الحل
```c
/* الطريقة الصح — استخدم compatible string */
struct device_node *np = of_find_compatible_node(NULL, NULL,
                                                  "winbond,w25q128");
if (!np) {
    pr_err("DT node for W25Q128 not found\n");
    return -ENODEV;
}

struct device *dev = bus_find_device_by_of_node(&spi_bus_type, np);
of_node_put(np);  /* مهم — release الـ reference */

if (!dev) {
    pr_err("SPI device for W25Q128 not found\n");
    return -ENODEV;
}
```

أو لو عارف الـ path بالظبط:
```c
struct device_node *np = of_find_node_by_path("/soc/spi@5c000/flash@0");
```

للتحقق:
```bash
# شوف الـ of_node بتاع الـ SPI device
$ cat /sys/bus/spi/devices/spi1.0/of_node
# بيظهر symlink للـ DT node
$ ls -la /sys/bus/spi/devices/spi1.0/of_node
```

#### الدرس المستفاد
الـ `bus_find_device_by_of_node` بتعمل pointer comparison على `dev->of_node` — لازم الـ `device_node` pointer يكون بالظبط نفس الـ pointer اللي الـ device اتعمل بيه. أي مسار غلط لجلب الـ `np` بيخلي الـ search تفشل. استخدم `of_find_compatible_node` أو `of_find_node_by_path` بدل `of_find_node_by_name` إلا لو إنت متأكد من اسم الـ node.

---

### السيناريو 4: Automotive ECU على i.MX8 — Bus Notifier لمراقبة UART devices بيعمل kernel panic

#### العنوان
**`bus_register_notifier` callback بيعمل lock violation لما بيقرأ device data في `BUS_NOTIFY_BIND_DRIVER` event**

#### السياق
بتشتغل على **automotive ECU** بـ **i.MX8MP** فيه 4 UART channels متصلة بـ CAN transceivers. إنت كتبت kernel module بيستخدم `bus_register_notifier` عشان يعرف فوراً لما UART driver بيتربط بأي device — ده عشان تعمل custom initialization للـ CAN transceiver بعد ما الـ UART يكون جاهز.

#### المشكلة
```bash
$ insmod uart_monitor.ko
$ dmesg | tail -30
BUG: sleeping function called from invalid context
at kernel/locking/mutex.c:580
in_atomic(): 1, irqs_disabled(): 0
...
Call trace:
 uart_monitor_notifier_call+0x5c/0x120 [uart_monitor]
 blocking_notifier_call_chain+0x68/0x90
 bus_notify+0x2c/0x38
```

#### التحليل
الـ `bus.h` بيوضح إن الـ notifier events هي:

```c
enum bus_notifier_event {
    BUS_NOTIFY_ADD_DEVICE,
    BUS_NOTIFY_DEL_DEVICE,
    BUS_NOTIFY_REMOVED_DEVICE,
    BUS_NOTIFY_BIND_DRIVER,    /* <-- ده اللي بنستخدمه */
    BUS_NOTIFY_BOUND_DRIVER,
    BUS_NOTIFY_UNBIND_DRIVER,
    BUS_NOTIFY_UNBOUND_DRIVER,
    BUS_NOTIFY_DRIVER_NOT_BOUND,
};
```

والـ comment المهم في الهيدر:
```c
* Note that bus notifiers are likely to be called with the device lock already
* held by the driver core, so be careful in any notifier callback as to what
* you do with the device structure.
```

الكود الغلط في الـ notifier callback:

```c
/* غلط — بيحاول يعمل mutex_lock وهو في locked context */
static int uart_monitor_notifier_call(struct notifier_block *nb,
                                       unsigned long event, void *data)
{
    struct device *dev = data;

    if (event == BUS_NOTIFY_BIND_DRIVER) {
        /* ده بيعمل kmalloc(GFP_KERNEL) اللي ممكن يـ sleep */
        struct uart_config *cfg = kzalloc(sizeof(*cfg), GFP_KERNEL);

        /* وده بيـ lock mutex */
        mutex_lock(&imx8_uart_global_lock);
        setup_can_transceiver(dev, cfg);
        mutex_unlock(&imx8_uart_global_lock);

        kfree(cfg);
    }
    return NOTIFY_OK;
}
```

الـ notifier callback بيتنادى وهو الـ device lock محجوز من الـ driver core. أي `kmalloc(GFP_KERNEL)` أو `mutex_lock` جوا الـ callback ممكن يـ sleep — وده مش مسموح في هذا الـ context.

#### الحل
```c
struct uart_monitor_work {
    struct work_struct work;
    struct device *dev;
};

static void uart_monitor_work_fn(struct work_struct *work)
{
    struct uart_monitor_work *mw =
        container_of(work, struct uart_monitor_work, work);

    /* هنا آمن نعمل كل حاجة تحتاج sleep */
    struct uart_config *cfg = kzalloc(sizeof(*cfg), GFP_KERNEL);
    if (cfg) {
        mutex_lock(&imx8_uart_global_lock);
        setup_can_transceiver(mw->dev, cfg);
        mutex_unlock(&imx8_uart_global_lock);
        kfree(cfg);
    }
    put_device(mw->dev);
    kfree(mw);
}

static int uart_monitor_notifier_call(struct notifier_block *nb,
                                       unsigned long event, void *data)
{
    struct device *dev = data;
    struct uart_monitor_work *mw;

    if (event != BUS_NOTIFY_BOUND_DRIVER)  /* بعد ما الـ binding يكتمل */
        return NOTIFY_OK;

    /* لا sleep ولا mutex هنا — بس schedule work */
    mw = kmalloc(sizeof(*mw), GFP_ATOMIC);  /* GFP_ATOMIC مش GFP_KERNEL */
    if (!mw)
        return NOTIFY_OK;

    get_device(dev);  /* زوّد الـ refcount عشان الـ work يستخدمه بأمان */
    mw->dev = dev;
    INIT_WORK(&mw->work, uart_monitor_work_fn);
    schedule_work(&mw->work);

    return NOTIFY_OK;
}

static struct notifier_block uart_monitor_nb = {
    .notifier_call = uart_monitor_notifier_call,
};

/* في الـ module init */
bus_register_notifier(&platform_bus_type, &uart_monitor_nb);
```

#### الدرس المستفاد
الـ `bus_register_notifier` قوي جداً بس الـ comment في `bus.h` صريح: الـ device lock ممكن يكون محجوز. أي عملية تحتاج sleep (kmalloc GFP_KERNEL، mutex، msleep) ممنوعة مباشرة في الـ callback. الحل دايماً: defer الشغل لـ workqueue وابعت بـ `GFP_ATOMIC` فقط في الـ callback نفسه.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `dma_configure` callback في custom bus بيتجاهل IOMMU

#### العنوان
**custom bus type على AM62x مش بتعمل implement لـ `dma_configure` وبيحصل DMA corruption في HDMI output**

#### السياق
بتعمل bring-up لبورد custom مبني على **TI AM62x** فيه custom **FPGA-based display bus** بيطلع HDMI. إنت عملت `bus_type` خاصة للـ FPGA bus وفيها devices بتعمل DMA direct للـ HDMI framebuffer. الـ display بتظهر بس مع artifacts وـ corruption.

#### المشكلة
```bash
$ dmesg | grep -i "dma\|iommu"
iommu: Adding device fpga-display.0 to group 3
# لا في أي "dma configured" message — الـ IOMMU مش شغال على الـ device
```

الـ artifacts بتظهر بشكل عشوائي — أحياناً كل ربع ساعة، أحياناً كل 5 دقايق.

#### التحليل
الـ `struct bus_type` في `bus.h` عندها callbacks للـ DMA:

```c
struct bus_type {
    // ...
    int (*dma_configure)(struct device *dev);
    void (*dma_cleanup)(struct device *dev);
    // ...
};
```

وعندها برضو:
```c
* @dma_configure:   Called to setup DMA configuration on a device on
*                   this bus.
* @dma_cleanup:     Called to cleanup DMA configuration on a device on
*                   this bus.
```

الـ bus registration اللي عملها المطور:

```c
/* ناقص dma_configure */
static struct bus_type fpga_bus_type = {
    .name        = "fpga",
    .match       = fpga_bus_match,
    .probe       = fpga_bus_probe,
    .remove      = fpga_bus_remove,
    /* .dma_configure = NULL  <-- لم يُعرَّف! */
};
```

لما الـ `dma_configure` بيكون `NULL`، الـ driver core بيتخطى إعداد الـ IOMMU للـ device. على AM62x، الـ IOMMU لازم يتعمل configure صح عشان الـ DMA transfers تكون آمنة ومحكومة — بدونه، الـ FPGA device بيعمل DMA على physical addresses مباشرة بدون translation، وممكن يكتب فوق memory غلط خصوصاً لو الـ memory layout اتغير.

الـ bus types الـ built-in زي `platform_bus_type` بتعمل implement لـ `dma_configure` عن طريق `of_dma_configure`:

```c
/* من drivers/base/platform.c — للمقارنة */
static int platform_dma_configure(struct device *dev)
{
    struct fwnode_handle *fwnode = dev_fwnode(dev);
    enum dev_dma_attr attr;
    int ret = 0;

    if (is_of_node(fwnode)) {
        ret = of_dma_configure(dev, to_of_node(fwnode), true);
    } else if (is_acpi_device_node(fwnode)) {
        attr = acpi_get_dma_attr(to_acpi_device_node(fwnode));
        ret = acpi_dma_configure(dev, attr);
    }
    return ret;
}
```

#### الحل
نضيف الـ `dma_configure` و `dma_cleanup` للـ custom bus:

```c
static int fpga_bus_dma_configure(struct device *dev)
{
    /* استخدم نفس الـ mechanism بتاع platform bus */
    struct fwnode_handle *fwnode = dev_fwnode(dev);

    if (is_of_node(fwnode))
        return of_dma_configure(dev, to_of_node(fwnode), true);

    /* fallback لو مفيش DT node */
    return 0;
}

static void fpga_bus_dma_cleanup(struct device *dev)
{
    arch_teardown_dma_ops(dev);
}

static struct bus_type fpga_bus_type = {
    .name           = "fpga",
    .match          = fpga_bus_match,
    .probe          = fpga_bus_probe,
    .remove         = fpga_bus_remove,
    .dma_configure  = fpga_bus_dma_configure,   /* مضاف */
    .dma_cleanup    = fpga_bus_dma_cleanup,     /* مضاف */
};
```

التحقق بعد الحل:
```bash
# تأكد إن الـ IOMMU اتعمل configure
$ dmesg | grep -i "fpga-display\|iommu"
iommu: Adding device fpga-display.0 to group 3
OF: DMA configuration: dma-ranges found

# شوف الـ DMA mask بتاع الـ device
$ cat /sys/bus/fpga/devices/fpga-display.0/dma_mask_bits
32

# لو محتاج تـ debug أعمق
$ cat /sys/kernel/debug/iommu/devices
```

في الـ DT لازم يكون:
```dts
fpga_display: display@40000000 {
    compatible = "myco,fpga-hdmi";
    reg = <0x40000000 0x1000>;
    dma-ranges = <0x0 0x0 0x80000000>;  /* مهم لـ IOMMU */
    iommus = <&iommu 0x10>;
};
```

#### الدرس المستفاد
لما بتعمل custom `bus_type`، الـ `dma_configure` مش optional لو الـ devices على الـ bus بتستخدم DMA وفي IOMMU في الـ system. تجاهل الـ callback بيعمل الـ devices تشتغل بـ "raw" DMA بدون IOMMU protection، والـ corruption بيكون intermittent وصعب diagnosis. دايماً اعمل implement لـ `dma_configure` في أي custom bus وابقى على نفس الـ semantics بتاعة `platform_bus_type`.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

الـ kernel بيوفر توثيق مباشر للـ driver model في مسارات متعددة:

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/driver-model/bus.rst` | شرح `bus_type` ودورة حياة الـ bus |
| `Documentation/driver-api/driver-model/overview.rst` | نظرة عامة على الـ device model |
| `Documentation/driver-api/driver-model/binding.rst` | آلية الـ match والـ probe بين driver وdevice |
| `Documentation/driver-api/driver-model/driver.rst` | هيكل `device_driver` وعلاقته بالـ bus |
| `Documentation/driver-api/driver-model/devres.rst` | إدارة الـ resources المرتبطة بالـ device |
| `Documentation/ABI/testing/sysfs-bus-*` | ملفات الـ sysfs الخاصة بكل bus |

الـ online version على:
- [Bus Types — The Linux Kernel documentation](https://docs.kernel.org/driver-api/driver-model/bus.html)
- [The Linux Kernel Device Model](https://docs.kernel.org/driver-api/driver-model/overview.html)
- [Driver Binding](https://static.lwn.net/kerneldoc/driver-api/driver-model/binding.html)

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأهم لفهم تطور الـ kernel — كل مقال جوه بيشرح السياق التاريخي والتقني:

#### مقالات أساسية

1. **[A fresh look at the kernel's device model](https://lwn.net/Articles/645810/)**
   مقال 2015 بيراجع الـ device model من زاوية جديدة. بيوضح إزاي الـ `bus_type` بيربط الـ i2c bus بالـ drivers الخاصة بيه. ده من أفضل الشروحات العملية الحديثة.

2. **[Managing multifunction devices with the auxiliary bus](https://lwn.net/Articles/840416/)**
   بيشرح إزاي الـ `auxiliary bus` اتضاف كـ bus جديد لحل مشكلة الـ multi-function devices. مثال ممتاز على إزاي `struct bus_type` بيتوسع.

3. **[PCI Express Port Bus Driver](https://lwn.net/Articles/116311/)**
   بيوضح كيف يُستخدم الـ bus model لتقديم خدمات متعددة (AER, PME, hotplug) فوق bus واحد.

4. **[Driver porting: Device model overview](https://lwn.net/Articles/31185/)**
   مقال تاريخي من فترة الـ 2.5 kernel — بيشرح إزاي الـ driver model اتبنى من الصفر.

5. **[Driver porting: Devices and attributes](https://lwn.net/Articles/31220/)**
   بيوضح إزاي الـ `bus_attribute` و`BUS_ATTR_RW` اتصمموا وعلاقتهم بالـ sysfs.

#### مقالات الـ kobject والـ sysfs (الأساس التحتاني)

6. **[The zen of kobjects](https://lwn.net/Articles/51437/)**
   الـ `kobject` هو الأساس اللي `bus_type` اتبنى عليه — لازم تفهمه أول.

7. **[kobjects and sysfs](https://lwn.net/Articles/54651/)**
   العلاقة بين الـ kobject وظهور الـ bus في `/sys/bus/`.

8. **[Examining a kobject hierarchy](https://lwn.net/Articles/55847/)**
   بيتتبع الـ hierarchy اللي الـ bus بيكونها في sysfs.

9. **[Documentation/kobject.txt](https://lwn.net/Articles/266722/)**
   نسخة من الـ kernel documentation مع شرح LWN.

---

### PDF الـ OLS وورق بحثي

- **[The Linux Kernel Device Model — Patrick Mochel (OLS 2002)](https://www.landley.net/kdocs/ols/2002/ols2002-pages-368-375.pdf)**
  الورقة البحثية الأصلية اللي Patrick Mochel قدمها في Ottawa Linux Symposium 2002 — ده اللي صمم `bus_type` أساساً.

- **[The Linux Kernel Driver Model — Patrick Mochel (LCA 2003)](http://mirror.linux.org.au/pub/linux.conf.au/2003/papers/Patrick_Mochel/Patrick_Mochel.pdf)**
  شرح تفصيلي لكيفية تسجيل الـ bus وربط الـ devices بالـ drivers.

---

### KernelNewbies

- **[Drivers — Linux Kernel Newbies](https://kernelnewbies.org/Drivers)**
  صفحة تجميعية للـ resources عن كتابة الـ drivers بما فيها bus-specific guides.

- **[Linux_3.5_DriverArch](https://kernelnewbies.org/Linux_3.5_DriverArch)**
  تغييرات الـ driver architecture في kernel 3.5 — بتشمل تغييرات على الـ bus model.

- **[How does the probe function gets called on a PCI device driver?](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-January/015658.html)**
  نقاش عملي عن `pci_bus_type->match` و`pci_bus_type->probe` بأمثلة حقيقية.

- **[When "probe" is called?](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-February/000815.html)**
  شرح للـ flow من `bus_register` → `bus_add_device` → `match` → `probe`.

---

### eLinux.org

- **[Threaded Device Probing](https://elinux.org/Threaded_Device_Probing)**
  بيشرح ميزة الـ threaded probing في الـ PCI bus وتأثيرها على وقت الـ boot.

- **[Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations)**
  مجموعة presentations عن الـ I2C bus، Serial Device Bus، وغيرها من الـ bus implementations.

---

### مناقشات الـ Mailing List

- **[LKML.ORG Archive](https://lkml.org/)** — الأرشيف الرسمي لكل نقاشات الـ kernel
- **[lore.kernel.org — driver-core](https://lore.kernel.org/driver-core/)** — القائمة المتخصصة لتطوير الـ driver core والـ bus model
- **[lore.kernel.org — linux-kernel](https://lore.kernel.org/linux-kernel/)** — النقاشات العامة للـ kernel

للبحث عن نقاشات محددة عن `bus_type`:
```
site:lore.kernel.org "bus_type" "bus_register"
site:lkml.org "struct bus_type"
```

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
الكتاب الأكلاسيكي — متاح مجاناً:
- **الفصل 14: The Linux Device Model** — [PDF مباشر](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)
  بيشرح `bus_type`, `device`, `device_driver` بالتفصيل مع أمثلة.
- الكتاب كامل: https://lwn.net/Kernel/LDD3/

> ملاحظة: LDD3 كُتب لـ kernel 2.6.10، بعض التفاصيل اتغيرت، لكن المفاهيم الأساسية لسه صح.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17: Devices and Modules** — بيغطي الـ device model والـ kobject والـ sysfs
- ISBN: 978-0672329463

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 8: Device Driver Basics** — بيشرح الـ bus model في سياق embedded systems
- ISBN: 978-0137017836

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 6: Device Drivers** — شرح عميق للـ bus model مع source code walkthrough
- ISBN: 978-0470343432

---

### مصادر الـ Source Code

الملفات الأساسية في الـ kernel source:

```
include/linux/device/bus.h        ← الـ header موضوع الدراسة
drivers/base/bus.c                ← تنفيذ bus_register, bus_add_device, match/probe
drivers/base/core.c               ← device_add, device_bind_driver
drivers/base/dd.c                 ← driver_attach, bus_probe_device
include/linux/device.h            ← struct device, struct device_driver
include/linux/kobject.h           ← الأساس التحتاني
```

للبحث في git log عن تغييرات `bus_type`:
```bash
git log --oneline --all -- include/linux/device/bus.h
git log --oneline --all -- drivers/base/bus.c
git log --grep="bus_type" --oneline
```

---

### Kernel Labs والـ Lab Exercises

- **[Linux Device Model — linux-kernel-labs](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html)**
  تمارين عملية لكتابة bus جديد من الصفر، تسجيل devices وdrivers، وربطهم.

- **[Everything you never wanted to know about kobjects, ksets, and ktypes](https://www.kernel.org/doc/html/v5.15/core-api/kobject.html)**
  الدليل الرسمي لفهم الـ infrastructure اللي `bus_type` بتبني عليها.

---

### كلمات البحث المفيدة

للقاء معلومات أكتر عن هذا الـ subsystem:

```
linux kernel bus_type implementation
linux driver model bus probe match
struct bus_type registration sysfs
linux bus_register bus_add_device
linux device driver binding unbinding
linux bus notifier BUS_NOTIFY_BIND_DRIVER
linux auxiliary bus multifunction device
linux platform bus virtual bus
linux deferred probe EPROBE_DEFER
linux bus_for_each_dev bus_find_device
```
## Phase 8: Writing simple module

### الفكرة: Bus Notifier على `platform` bus

**الـ** `bus_register_notifier` / `bus_unregister_notifier` هما من أكثر الـ hooks فائدةً في الـ driver model — بيخلوك تشوف كل device بتتضاف أو بتتشال أو بيتربط بيها driver على أي bus من غير ما تعدّل في الـ kernel نفسه.

هنعمل module بيسجّل notifier على الـ `platform` bus، وبيطبع اسم الـ device وحالتها كل ما حصل event (add, bind, unbind, remove...).

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * bus_notifier_demo.c
 * Hooks into the platform bus notifier chain to observe
 * device/driver lifecycle events in real time.
 */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/notifier.h>     /* notifier_block, notifier_fn_t */
#include <linux/device/bus.h>   /* bus_register_notifier, bus_notifier_event */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/platform_device.h> /* platform_bus_type */

/* ------------------------------------------------------------------ */
/* Callback: called by the bus core on every event                     */
/* ------------------------------------------------------------------ */
static int bus_event_handler(struct notifier_block *nb,
                             unsigned long action, void *data)
{
    /*
     * data هنا دايماً struct device * — كل bus notifiers بتبعت
     * الـ target device كـ data pointer حسب التوثيق في bus.h
     */
    struct device *dev = data;

    /* map the numeric action to a human-readable string */
    const char *event_str;
    switch ((enum bus_notifier_event)action) {
    case BUS_NOTIFY_ADD_DEVICE:
        event_str = "ADD_DEVICE";
        break;
    case BUS_NOTIFY_DEL_DEVICE:
        event_str = "DEL_DEVICE";
        break;
    case BUS_NOTIFY_REMOVED_DEVICE:
        event_str = "REMOVED_DEVICE";
        break;
    case BUS_NOTIFY_BIND_DRIVER:
        event_str = "BIND_DRIVER";
        break;
    case BUS_NOTIFY_BOUND_DRIVER:
        event_str = "BOUND_DRIVER";
        break;
    case BUS_NOTIFY_UNBIND_DRIVER:
        event_str = "UNBIND_DRIVER";
        break;
    case BUS_NOTIFY_UNBOUND_DRIVER:
        event_str = "UNBOUND_DRIVER";
        break;
    case BUS_NOTIFY_DRIVER_NOT_BOUND:
        event_str = "DRIVER_NOT_BOUND";
        break;
    default:
        event_str = "UNKNOWN";
        break;
    }

    /*
     * dev_name() بترجع الـ kobject name للـ device —
     * ده نفس الاسم اللي بتشوفه في /sys/bus/platform/devices/
     * driver اللي متربط (لو فيه) بنجيبه من dev->driver->name
     */
    pr_info("[bus_notifier_demo] event=%-20s device=%-30s driver=%s\n",
            event_str,
            dev_name(dev),
            (dev->driver ? dev->driver->name : "<none>"));

    /* NOTIFY_OK = عملنا اللي علينا، كمّل على باقي الـ notifiers */
    return NOTIFY_OK;
}

/* ------------------------------------------------------------------ */
/* notifier_block: الـ struct اللي بيربط الـ callback بالـ chain      */
/* ------------------------------------------------------------------ */
static struct notifier_block bus_nb = {
    .notifier_call = bus_event_handler,
    /*
     * priority = 0 (default) — الـ notifiers بتتنفذ من الأعلى
     * priority للأقل، صفر هو الوسط المناسب للـ observation فقط
     */
    .priority = 0,
};

/* ------------------------------------------------------------------ */
/* module_init: سجّل الـ notifier على platform bus                    */
/* ------------------------------------------------------------------ */
static int __init bus_notifier_demo_init(void)
{
    int ret;

    /*
     * platform_bus_type هو الـ bus_type الـ global المعرّف في
     * drivers/base/platform.c — بنمرره مباشرةً لـ bus_register_notifier
     */
    ret = bus_register_notifier(&platform_bus_type, &bus_nb);
    if (ret) {
        pr_err("[bus_notifier_demo] failed to register notifier: %d\n", ret);
        return ret;
    }

    pr_info("[bus_notifier_demo] registered on platform bus — watching events\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: ازل الـ notifier لتجنب use-after-free                 */
/* ------------------------------------------------------------------ */
static void __exit bus_notifier_demo_exit(void)
{
    /*
     * لازم نشيل الـ notifier_block قبل ما الـ module يتـ unload —
     * لو سبناه، الـ bus core هيحاول يكال callback في memory اتحررت
     * وده kernel panic مضمون
     */
    bus_unregister_notifier(&platform_bus_type, &bus_nb);
    pr_info("[bus_notifier_demo] unregistered — goodbye\n");
}

module_init(bus_notifier_demo_init);
module_exit(bus_notifier_demo_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Demo Author <demo@example.com>");
MODULE_DESCRIPTION("Platform bus notifier demo — observe device/driver lifecycle");
```

---

### Makefile

```makefile
obj-m += bus_notifier_demo.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات الـ module الأساسية: `module_init`, `MODULE_LICENSE` |
| `linux/notifier.h` | تعريف `struct notifier_block` و `NOTIFY_OK` |
| `linux/device/bus.h` | `bus_register_notifier`, `bus_unregister_notifier`, `enum bus_notifier_event` |
| `linux/device.h` | `struct device`, `dev_name()` |
| `linux/platform_device.h` | الـ `platform_bus_type` external symbol |

---

#### الـ Callback: `bus_event_handler`

**الـ** `action` parameter بييجي كـ `unsigned long` بس قيمته دايماً واحدة من `enum bus_notifier_event` — عشان كده بنعمل `switch` عليه بعد ما نعمل cast صريح. الـ `data` pointer حسب التوثيق في `bus.h` هو دايماً `struct device *` لكل bus notifiers.

**الـ** `dev->driver` ممكن يكون `NULL` لو الـ event هو `ADD_DEVICE` قبل ما يتربط driver — عشان كده بنعمل check قبل ما نوصل `driver->name`.

---

#### الـ `notifier_block`

**الـ** `struct notifier_block` هو الـ "registration ticket" — بيحتوي على pointer للـ callback وأولوية التنفيذ. الـ bus core بيحتفظ بـ linked list من كل الـ blocks المسجلة وبينفذهم بالترتيب عند كل event.

---

#### الـ `module_init` و `module_exit`

**الـ** `bus_register_notifier` بتضيف الـ `bus_nb` في الـ notifier chain الخاصة بالـ `platform_bus_type`. في الـ `exit`، لازم نشيله أولاً قبل ما الـ kernel يـ unmap الـ module memory — ده مش optional، ده safety requirement.

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميله
sudo insmod bus_notifier_demo.ko

# جرّب تحمّل device — مثلاً عن طريق إضافة platform device
# أو شاهد الـ events الموجودة لو فيه modules تتحمّل

# مشاهدة الـ output
sudo dmesg | grep bus_notifier_demo

# إزالته
sudo rmmod bus_notifier_demo
```

**مثال على الـ output المتوقع:**

```
[bus_notifier_demo] registered on platform bus — watching events
[bus_notifier_demo] event=BIND_DRIVER          device=serial8250              driver=serial8250
[bus_notifier_demo] event=BOUND_DRIVER         device=serial8250              driver=serial8250
[bus_notifier_demo] event=ADD_DEVICE           device=reg-dummy               driver=<none>
[bus_notifier_demo] unregistered — goodbye
```
