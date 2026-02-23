## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الملف `include/linux/device/driver.h` جزء من subsystem اسمه **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS** — وده من أقدم وأهم أجزاء الـ Linux kernel. المشرفين عليه: Greg Kroah-Hartman و Rafael J. Wysocki و Danilo Krummrich.

---

### الصورة الكبيرة — قصة بسيطة

تخيل إنك شغال في مطار. عندك:
- **طيارات** (devices) — كل طيارة بتيجي بشكل مختلف.
- **طيارين** (drivers) — كل طيار بيعرف يشغّل نوع معين من الطيارات.
- **مدرج/بوابة** (bus) — المكان اللي بيلاقي الطيار بطيارته.

لما طيارة جديدة تيجي للمطار، المطار بيبص على كل الطيارين اللي موجودين ويقول: "في حد منكم يعرف يشغّل الطيارة دي؟" — لو لقى طيار مناسب، بيعمل **bind** بينهم وتطير.

ده بالظبط اللي بيحصل في الـ Linux Driver Model:
- لما hardware بييجي للسيستم (USB جديد، PCI كارد، إلخ)، الـ kernel بيدور على **driver** مناسب.
- لما driver بيتسجّل، الـ kernel بيدور على **device** موجود ومحتاج الـ driver ده.
- العملية دي اسمها **probing**.

---

### الملف ده بيعمل إيه بالظبط؟

الملف `include/linux/device/driver.h` هو **العقد الرسمي** اللي بيعرّف شكل أي driver في الـ Linux kernel. زي ما بتكتب interface في OOP — أي driver لازم يوفّر الـ functions دي عشان الـ kernel يعرف يتعامل معاه.

الملف بيعرّف:

#### 1. الـ `struct device_driver` — قلب الموضوع

```c
struct device_driver {
    const char              *name;          /* اسم الـ driver */
    const struct bus_type   *bus;           /* على أي bus بيشتغل؟ PCI? USB? I2C? */
    struct module           *owner;         /* الـ kernel module اللي فيه الـ driver */

    /* جدول المطابقة — بيقول: أنا أشغّل الأجهزة دي */
    const struct of_device_id   *of_match_table;   /* Device Tree matching */
    const struct acpi_device_id *acpi_match_table;  /* ACPI matching */

    /* الـ callbacks اللي الـ kernel بيستدعيها */
    int  (*probe)   (struct device *dev);   /* لما device بيتلاقى */
    int  (*remove)  (struct device *dev);   /* لما device بيتشال */
    void (*shutdown)(struct device *dev);   /* لما الجهاز بيتقفل */
    int  (*suspend) (struct device *dev, pm_message_t state); /* نوم */
    int  (*resume)  (struct device *dev);   /* صحيان */
    void (*sync_state)(struct device *dev); /* مزامنة state */

    const struct dev_pm_ops *pm;  /* Power Management متقدم */
    struct driver_private   *p;   /* بيانات خاصة بالـ driver core */
};
```

#### 2. الـ `enum probe_type` — متى وإزاي بيتم الـ probing؟

```c
enum probe_type {
    PROBE_DEFAULT_STRATEGY,      /* الـ kernel يقرر */
    PROBE_PREFER_ASYNCHRONOUS,   /* الـ driver بطيء، شغّله في background */
    PROBE_FORCE_SYNCHRONOUS,     /* لازم يتشغّل دلوقتي، قبل ما نكمل */
};
```

الفكرة دي مهمة جداً لـ **boot time**: لو كل driver بيستنى التاني، الـ boot هيبقى بطيء. الـ async probe بيخلي الـ drivers البطيئة (زي USB أو storage) تتشغّل في الخلفية وأنت بتعمل boot.

#### 3. الـ `driver_attribute` — الـ driver بيكلم المستخدم عبر sysfs

```c
struct driver_attribute {
    struct attribute attr;
    ssize_t (*show) (struct device_driver *driver, char *buf);
    ssize_t (*store)(struct device_driver *driver, const char *buf, size_t count);
};
```

ده بيسمح للـ driver يعرض معلومات في `/sys/bus/<bus>/drivers/<driver>/` — وتقدر تقراها أو تكتب فيها من userspace.

مثال حقيقي:
```bash
# شوف version الـ driver
cat /sys/bus/pci/drivers/e1000e/version
# إجبر bind
echo "0000:00:19.0" > /sys/bus/pci/drivers/e1000e/bind
```

#### 4. الـ Helper Macros — توفير boilerplate

```c
/* كل module عنده init و exit بوجع، الـ macro دي بتعملها تلقائياً */
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ return __register(&(__driver), ##__VA_ARGS__); } \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ __unregister(&(__driver), ##__VA_ARGS__); } \
module_exit(__driver##_exit);
```

كل bus بيعمل macro فوق الـ macro دي — مثلاً:
- `module_i2c_driver(my_driver)` → بتسجّل I2C driver بسطر واحد.
- `module_platform_driver(my_driver)` → بتسجّل platform driver بسطر واحد.

---

### قصة الـ Deferred Probe — المشكلة الحقيقية

تخيل إنك عندك:
- **GPU driver** محتاج يعرف مكان الـ power regulator.
- الـ **regulator driver** لسه مأتسجلش في الـ kernel.

لو الـ GPU probe اتشغّل الأول، هيلاقي الـ regulator مش موجود، فبدل ما يفشل فشل نهائي، بيرجع `-EPROBE_DEFER` — والـ kernel بيفهم: "تمام، اعمل retry بعدين."

الفانكشن `driver_deferred_probe_add()` بتضيف الـ device لـ queue وبتشيله بعد ما كل الـ drivers يتسجلوا.

---

### تسلسل الأحداث من boot لحد bind

```
kernel boot
    │
    ├── driver_init()          ← يهيّئ الـ driver core infrastructure
    │
    ├── bus_register()         ← كل bus (PCI, USB, I2C...) بيسجّل نفسه
    │
    ├── driver_register(drv)   ← الـ driver module بيسجّل نفسه
    │       │
    │       └── bus->match()   ← بيجرّب كل device موجود على الـ bus
    │               │
    │               └── driver->probe(dev)  ← لو match ← bind!
    │
    ├── device_register(dev)   ← لما hardware جديد بييجي
    │       │
    │       └── bus->match()   ← بيجرّب كل driver مسجّل
    │               │
    │               └── driver->probe(dev)  ← لو match ← bind!
    │
    └── sync_state()           ← بعد ما كل consumers اتبندوا
```

---

### الملفات المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/device/driver.h` | **الملف ده** — تعريف `struct device_driver` والـ API |
| `include/linux/device/bus.h` | تعريف `struct bus_type` — البيئة اللي الـ driver بيشتغل فيها |
| `include/linux/device/class.h` | تعريف `struct class` — تصنيف الأجهزة (input, net, block...) |
| `include/linux/device.h` | الـ umbrella header — بيضم الـ 3 headers فوق وتعريف `struct device` |
| `drivers/base/driver.c` | الـ implementation — `driver_register()`, `driver_unregister()` |
| `drivers/base/dd.c` | الـ device-driver binding logic — قلب الـ probe mechanism |
| `drivers/base/bus.c` | إدارة الـ buses |
| `drivers/base/core.c` | الـ core device lifecycle |
| `include/linux/mod_devicetable.h` | تعريف `of_device_id`, `acpi_device_id` وغيرهم |
| `include/linux/pm.h` | تعريف `dev_pm_ops` لـ Power Management |

---

### ملفات الـ Subsystem الكاملة (DRIVER CORE)

الـ subsystem "DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS" بيتكوّن من:

**Core Implementation:**
- `drivers/base/` — كل الـ C files اللي فوق
- `fs/sysfs/` — الـ virtual filesystem اللي بيعرض الـ kernel objects
- `fs/debugfs/` — interface للـ debugging

**Headers:**
- `include/linux/device/` — driver.h, bus.h, class.h, devres.h, faux.h
- `include/linux/device.h` — الـ umbrella header
- `include/linux/kobject.h` — الـ base object model
- `include/linux/sysfs.h` — الـ sysfs API
- `include/linux/property.h` — firmware properties (DT, ACPI)
- `include/linux/fwnode.h` — firmware node abstraction

**Rust Bindings (حديثة):**
- `rust/kernel/driver.rs`
- `rust/kernel/device.rs`
- `rust/kernel/device/`
## Phase 2: شرح الـ Driver Model Framework

### المشكلة اللي الـ Driver Model بيحلها

قبل الـ driver model (قبل kernel 2.5)، كل bus subsystem (PCI, USB, platform) كان بيعمل إدارة الـ devices والـ drivers بطريقته الخاصة. النتيجة:

- **code duplication**: كل bus بيعمل matching و probing من الصفر.
- **power management فوضى**: مفيش طريقة موحدة لـ suspend/resume الـ devices.
- **sysfs غير موجود**: المستخدم مش عارف يشوف الـ devices اللي في النظام.
- **hotplug مش معمول صح**: إضافة device وقت الـ runtime كانت inconsistent.

الـ driver model اتعمل عشان يقدم **abstraction موحدة** فوق كل هذا.

---

### الحل: ثلاث abstractions أساسية

الـ kernel حل المشكلة بتقسيم العالم لثلاث كيانات:

| الكيان | الـ struct | المعنى |
|--------|-----------|--------|
| **Device** | `struct device` | أي hardware أو virtual device في النظام |
| **Driver** | `struct device_driver` | الكود اللي بيتعامل مع نوع معين من الـ devices |
| **Bus** | `struct bus_type` | القناة اللي بتربط الـ devices بالـ drivers وبتعمل الـ matching |

الـ bus هو الـ matchmaker: لما device جديد يظهر أو driver جديد يتسجل، الـ bus بيشوف إيه اللي ينفع مع بعض.

---

### الـ Big Picture Architecture

```
User Space (udev, systemd)
        |
        | netlink uevents (ADD/REMOVE/BIND/...)
        |
════════════════════════════════════════════════════════════
                        sysfs (/sys)
  /sys/bus/           /sys/devices/         /sys/module/
  ├── i2c/            ├── platform/         └── my_driver/
  │   ├── devices/    │   ├── leds/
  │   └── drivers/    │   └── soc0/
  ├── platform/       └── ...
  └── pci/
════════════════════════════════════════════════════════════
                    Driver Core (drivers/base/)
  ┌─────────────────────────────────────────────────────┐
  │                                                     │
  │   bus_register()   driver_register()   device_add() │
  │         │                │                  │       │
  │    ┌────▼────┐    ┌──────▼──────┐    ┌──────▼────┐ │
  │    │bus_type │    │device_driver│    │  device   │ │
  │    │ .match()│◄───│  .probe()   │    │ .bus      │ │
  │    │ .probe()│    │  .remove()  │    │ .driver   │ │
  │    └────┬────┘    └─────────────┘    └───────────┘ │
  │         │                                           │
  │    match loop: for each (device, driver) pair       │
  │    if bus->match(dev, drv) → call driver->probe()   │
  └─────────────────────────────────────────────────────┘
════════════════════════════════════════════════════════════
        Subsystem Implementations (Bus Drivers)
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  PCI bus │  │  I2C bus │  │  USB bus │  │ platform │
  │ pci_bus_ │  │ i2c_bus_ │  │ usb_bus_ │  │   bus    │
  │   type   │  │   type   │  │   type   │  │          │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
════════════════════════════════════════════════════════════
        Concrete Drivers (consumers of the framework)
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ e1000e   │  │ at24 eep │  │ usb-stor │  │ gpio-pl  │
  │(PCI drv) │  │(I2C drv) │  │(USB drv) │  │(plt drv) │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

---

### التمثيل الحقيقي: وكالة التوظيف

تخيل إن الـ driver model هو **وكالة توظيف متخصصة**:

| مفهوم التمثيل | المقابل الحقيقي في الكيرنل |
|---------------|---------------------------|
| **الوظيفة الشاغرة** (job posting) | `struct device` — hardware محتاج driver |
| **المتقدم للوظيفة** (candidate) | `struct device_driver` — driver يعلن قدرته |
| **نوع الوظيفة** (industry) | `struct bus_type` — يحدد نوع الـ matching |
| **معيار الاختيار** (screening criteria) | `bus_type->match()` — يقرر التوافق |
| **المقابلة الشخصية** (interview) | `driver->probe()` — الـ driver يجرب الـ device |
| **عقد التوظيف** (contract) | binding: `dev->driver = drv` |
| **فصل الموظف** (termination) | `driver->remove()` — يلغي الـ binding |
| **سجل الموظفين** (HR records) | sysfs entries تحت `/sys/bus/*/drivers/` |
| **إشعار التوظيف** (announcement) | uevent → udev → userspace |

التشبيه مش سطحي: زي ما الوكالة هي اللي بتحدد معايير الـ matching (خبرة، شهادة)، كذلك الـ `bus_type->match()` هي اللي بتحدد إزاي الـ device بيلتقي مع الـ driver — عن طريق device tree compatible string، أو PCI vendor/device ID، أو ACPI ID.

---

### الـ Core Abstraction: الـ `struct device_driver`

الـ `struct device_driver` هو العقد اللي بين الـ driver writer والـ driver core. كل driver في الكيرنل لازم يملأ هذا الـ struct قبل ما يتسجل.

```c
struct device_driver {
    const char              *name;          /* اسم الـ driver في sysfs */
    const struct bus_type   *bus;           /* الـ bus اللي بيعمل عليه */

    struct module           *owner;         /* الـ module اللي بيمتلكه (للـ refcount) */
    const char              *mod_name;      /* للـ built-in modules */

    bool suppress_bind_attrs;               /* يمنع unbind يدوي عبر sysfs */
    enum probe_type probe_type;             /* sync أو async probing */

    /* firmware matching tables */
    const struct of_device_id   *of_match_table;    /* Device Tree */
    const struct acpi_device_id *acpi_match_table;  /* ACPI */

    /* lifecycle callbacks — الـ driver بيملأها */
    int  (*probe)  (struct device *dev);    /* أهم callback: جرب الـ device */
    int  (*remove) (struct device *dev);    /* الـ device اتشال */
    void (*shutdown)(struct device *dev);   /* النظام بيقفل */
    int  (*suspend) (struct device *dev, pm_message_t state); /* نوم */
    int  (*resume)  (struct device *dev);   /* صحيان */
    void (*sync_state)(struct device *dev); /* مزامنة الـ state بعد boot */

    /* sysfs attributes */
    const struct attribute_group **groups;      /* attributes للـ driver نفسه */
    const struct attribute_group **dev_groups;  /* attributes تتضاف للـ device بعد bind */

    /* power management */
    const struct dev_pm_ops *pm;            /* PM callbacks المتقدمة */

    /* misc */
    void (*coredump)(struct device *dev);   /* لما sysfs entry تتكتب */

    struct driver_private *p;               /* driver core private data — لا تلمسه */
};
```

---

### العلاقة بين الـ structs: رسم تفصيلي

```
struct bus_type (e.g., i2c_bus_type)
┌──────────────────────────────────┐
│ name: "i2c"                      │
│ match: i2c_device_match()        │◄─── يقارن dev->of_node مع drv->of_match_table
│ probe: i2c_device_probe()        │◄─── يستدعي drv->probe() بعد ما match ينجح
│ remove: i2c_device_remove()      │
│ pm: &i2c_pm_ops                  │
└──────────────────────────────────┘
         ▲                  ▲
         │                  │
struct device_driver        struct device
(e.g., at24_driver)         (e.g., eeprom@50)
┌───────────────────┐       ┌───────────────────┐
│ name: "at24"      │       │ bus: &i2c_bus_type │
│ bus: &i2c_bus_type│       │ driver: → at24_drv │ ◄── يتملأ بعد binding
│ of_match_table    │       │ of_node: → DT node │
│ probe: at24_probe │       │ p: (private)       │
│ remove: at24_rem  │       └───────────────────┘
│ pm: &at24_pm_ops  │
│ p: (private)      │ ◄── driver_private يحتوي على kobj وlist الـ devices
└───────────────────┘
         │
         │  driver_private (internal to driver core)
         ▼
┌─────────────────────────────────┐
│ struct klist klist_devices      │ ◄── قائمة كل الـ devices المربوطة بهذا الـ driver
│ struct kobject kobj             │ ◄── sysfs representation
│ struct bus_type *bus            │
└─────────────────────────────────┘
```

---

### الـ `kobject` والـ `sysfs`: الأساس الخفي

**الـ kobject** (kernel object) هو foundation اللي كل الـ driver model بيبنى عليه. كل `device_driver` يظهر في sysfs عن طريق `kobject` مخبي في `driver_private->kobj`.

```
الـ kobject بيعمل 3 حاجات:
1. Reference counting — عبر kref، عشان نعرف امتى نحذف الـ object
2. Hierarchy — كل kobject عنده parent، يبني شجرة الـ sysfs
3. uevents — يبعت إشعارات لـ userspace لما الـ state يتغير
```

لازم تفهم الـ kobject subsystem قبل ما تتعمق في الـ driver model.

---

### الـ `probe_type`: متى يتعمل الـ probe؟

```c
enum probe_type {
    PROBE_DEFAULT_STRATEGY,    /* الكيرنل يقرر — عادةً sync */
    PROBE_PREFER_ASYNCHRONOUS, /* للـ drivers البطيئة (storage, network) */
    PROBE_FORCE_SYNCHRONOUS,   /* لو الـ driver يعتمد على ترتيب الـ init */
};
```

الـ async probing مهم جداً على embedded: بدله، الـ boot يستنى كل driver يخلص probe قبل ما يكمل. مثلاً driver الـ eMMC على ARM board ممكن ياخد 100ms، الـ async يخليه يحصل في الـ background.

---

### الـ Deferred Probe: الحل لمشكلة الـ dependencies

لو driver A يعتمد على driver B (مثلاً GPIO driver يعتمد على pinctrl driver)، ومحصلش probe للـ B لسه:

```c
int my_probe(struct device *dev)
{
    gpio = gpiod_get(dev, "reset", GPIOD_OUT_LOW);
    if (IS_ERR(gpio)) {
        if (PTR_ERR(gpio) == -EPROBE_DEFER)
            return -EPROBE_DEFER; /* اتأجل، جرب تاني */
        return PTR_ERR(gpio);
    }
    /* ... */
}
```

الـ `driver_deferred_probe_add()` بيحط الـ device في قائمة انتظار، والـ driver core بيحاول تاني لما أي driver تاني يخلص probe.

---

### الـ Matching: إزاي الـ device يلتقي بالـ driver؟

الـ matching بيحصل بثلاث طرق حسب الـ bus:

#### 1. Device Tree (ARM/embedded)
```c
/* الـ driver بيعلن الـ compatible strings اللي بيدعمها */
static const struct of_device_id my_driver_of_match[] = {
    { .compatible = "vendor,my-chip-v1" },
    { .compatible = "vendor,my-chip-v2" },
    { /* sentinel */ }
};

/* الـ bus match function بتعمل */
of_match_device(drv->of_match_table, dev);
```

#### 2. ACPI (x86/modern ARM)
```c
static const struct acpi_device_id my_acpi_match[] = {
    { "VEND0001", 0 },
    { }
};
```

#### 3. Name-based (platform bus)
```c
/* لو مفيش DT ولا ACPI، الـ matching بيبقى باسم الـ device */
platform_device_register_simple("my-chip", -1, NULL, 0);
/* والـ driver يعلن اسمه */
static struct platform_driver my_driver = {
    .driver = { .name = "my-chip" }
};
```

---

### الـ `sync_state`: مشكلة الـ boot state

الـ `sync_state` callback حلت مشكلة دقيقة: الـ bootloader أو firmware بيضبط hardware في state معين (مثلاً regulator شغال). لو الـ kernel أطفل الـ regulator دلوقتي لأن driver الـ consumer لسه مش probe، الـ system هيتكسر.

```
Timeline:
  [boot] firmware enables regulator-A (needed by ethernet PHY)
  [kernel] regulator driver probes → but should NOT disable it yet
  [kernel] ethernet PHY driver probes → binds to regulator
  [kernel] ALL consumers bound → now call sync_state on regulator
           → NOW safe to disable unused parts
```

---

### الـ `driver_attribute` والـ sysfs Interface

```c
struct driver_attribute {
    struct attribute attr;          /* name, mode (permissions) */
    ssize_t (*show)(struct device_driver *, char *buf);
    ssize_t (*store)(struct device_driver *, const char *buf, size_t count);
};

/* Helper macros */
#define DRIVER_ATTR_RW(_name)   /* read + write: 0644 */
#define DRIVER_ATTR_RO(_name)   /* read only:    0444 */
#define DRIVER_ATTR_WO(_name)   /* write only:   0200 */
```

مثال: driver الـ `at24` EEPROM ممكن يعمل attribute زي:
```
/sys/bus/i2c/drivers/at24/page_size   ← يقرأ/يكتب page size
```

---

### الـ `module_driver` macro: إزاي الـ boilerplate بيتشال

```c
/* بدون macro — boilerplate */
static int __init at24_init(void)
{
    return i2c_add_driver(&at24_driver);
}
module_init(at24_init);

static void __exit at24_exit(void)
{
    i2c_del_driver(&at24_driver);
}
module_exit(at24_exit);

/* مع macro — نفس النتيجة */
module_i2c_driver(at24_driver);

/* الـ macro الأساسي اللي كل bus macros بتبنى عليه */
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void)                        \
{                                                               \
    return __register(&(__driver), ##__VA_ARGS__);             \
}                                                               \
module_init(__driver##_init);                                   \
static void __exit __driver##_exit(void)                        \
{                                                               \
    __unregister(&(__driver), ##__VA_ARGS__);                   \
}                                                               \
module_exit(__driver##_exit);
```

---

### إيه اللي الـ Framework بيملكه vs اللي بيديه للـ Driver

#### الـ Driver Core بيملكه (لا تلمسه):
- **`driver_private *p`**: قائمة الـ devices المربوطة، الـ kobject في sysfs، الـ deferred probe list.
- **الـ matching logic**: هو اللي يقرر متى تتعمل الـ probe.
- **الـ locking**: الـ driver core يدير الـ locks اللي بتحمي الـ bind/unbind.
- **الـ uevents**: بيبعت `KOBJ_BIND`/`KOBJ_UNBIND` لـ userspace تلقائياً.
- **الـ sysfs entries**: `/sys/bus/BUS/drivers/DRIVER/` يتعمل تلقائياً.

#### الـ Driver بيملكه (لازم تملاه):
- **`probe()`**: الـ logic الخاص بالـ hardware — initialize, request resources, register with subsystem.
- **`remove()`**: free resources, unregister.
- **`of_match_table`** / **`acpi_match_table`**: تحديد الـ hardware المدعوم.
- **`pm` / `suspend` / `resume`**: power management logic خاص بالـ hardware.
- **الـ attributes**: البيانات اللي يعرضها في sysfs.

---

### مثال end-to-end: I2C EEPROM driver

```c
/* 1. الـ match table */
static const struct of_device_id at24_of_match[] = {
    { .compatible = "atmel,24c02", .data = &at24_data_24c02 },
    { /* sentinel */ }
};

/* 2. الـ driver struct */
static struct i2c_driver at24_driver = {
    .driver = {
        .name           = "at24",            /* اسم في sysfs */
        .of_match_table = at24_of_match,     /* DT matching */
        .pm             = &at24_pm_ops,      /* PM callbacks */
    },
    .probe  = at24_probe,   /* الـ i2c subsystem بيلف على device_driver->probe */
    .remove = at24_remove,
};

/* 3. التسجيل — macro يعمل module_init/module_exit */
module_i2c_driver(at24_driver);

/* 4. الـ probe — يتستدعي من driver core بعد ما match تنجح */
static int at24_probe(struct i2c_client *client)
{
    /* client->dev هو الـ struct device */
    struct at24_data *at24;

    at24 = devm_kzalloc(&client->dev, sizeof(*at24), GFP_KERNEL);
    /* devm_: resources بتتحرر تلقائياً لما الـ device يتشال */

    /* سجل مع nvmem subsystem */
    at24->nvmem = devm_nvmem_register(&client->dev, &nvmem_config);

    return 0;
}
```

**الـ flow الكامل**:
```
Device Tree node (atmel,24c02 @ i2c1)
    │
    ▼ i2c_register_board_info() or DT scan
struct i2c_client (wraps struct device)
    │
    ▼ i2c_bus_type->match() → of_driver_match_device()
    │   compares client->dev.of_node with at24_of_match
    │
    ▼ match = true → driver core calls i2c_bus_type->probe()
    │   → i2c_device_probe() → at24_driver.probe()
    │
    ▼ at24_probe() runs
    │   → dev->driver = &at24_driver.driver  (binding)
    │   → sysfs: /sys/bus/i2c/drivers/at24/1-0050 → symlink
    │   → uevent: KOBJ_BIND → udev notified
    ▼
Device operational
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, و Config Options — Cheatsheet

#### `enum probe_type` — نوع الـ probe للـ driver

| القيمة | المعنى |
|---|---|
| `PROBE_DEFAULT_STRATEGY` | الـ core يقرر: sync أو async حسب الظروف |
| `PROBE_PREFER_ASYNCHRONOUS` | الـ driver بطيء، سمح للـ core يعمله async عشان يسرّع الـ boot |
| `PROBE_FORCE_SYNCHRONOUS` | الـ driver لازم يتعمل probe بشكل synchronous مع التسجيل |

> **ملاحظة**: الهدف على المدى البعيد هو إن الـ kernel يبقى async بالكامل، لكن دلوقتي بعض الـ drivers لسه محتاجة sync.

---

#### `enum kobject_action` — أحداث الـ uevent

| القيمة | متى بتتبعت |
|---|---|
| `KOBJ_ADD` | لما kobject يتضاف للـ sysfs |
| `KOBJ_REMOVE` | لما kobject يتشال |
| `KOBJ_CHANGE` | لما حاجة في الـ kobject تتغير |
| `KOBJ_MOVE` | لما kobject ينتقل لـ parent تاني |
| `KOBJ_ONLINE` | لما device يرجع online |
| `KOBJ_OFFLINE` | لما device يتعمله offline |
| `KOBJ_BIND` | لما driver يتربط بـ device |
| `KOBJ_UNBIND` | لما driver ينفصل عن device |

---

#### `enum bus_notifier_event` — أحداث الـ bus notifier

| القيمة | المعنى |
|---|---|
| `BUS_NOTIFY_ADD_DEVICE` | device اتضاف للـ bus |
| `BUS_NOTIFY_DEL_DEVICE` | device على وشك يتشال |
| `BUS_NOTIFY_REMOVED_DEVICE` | device اتشال فعلاً |
| `BUS_NOTIFY_BIND_DRIVER` | driver على وشك يتربط |
| `BUS_NOTIFY_BOUND_DRIVER` | driver اتربط بنجاح |
| `BUS_NOTIFY_UNBIND_DRIVER` | driver على وشك ينفصل |
| `BUS_NOTIFY_UNBOUND_DRIVER` | driver انفصل بنجاح |
| `BUS_NOTIFY_DRIVER_NOT_BOUND` | driver فشل في الارتباط |

---

#### Macros مهمة

| الـ Macro | الوظيفة |
|---|---|
| `DRIVER_ATTR_RW(_name)` | ينشئ `driver_attribute` بـ read+write في sysfs |
| `DRIVER_ATTR_RO(_name)` | ينشئ `driver_attribute` بـ read-only |
| `DRIVER_ATTR_WO(_name)` | ينشئ `driver_attribute` بـ write-only |
| `BUS_ATTR_RW/RO/WO(_name)` | نفس الفكرة للـ bus |
| `module_driver(drv, reg, unreg, ...)` | يولّد `module_init` و `module_exit` تلقائياً |
| `builtin_driver(drv, reg, ...)` | زي `module_driver` بس من غير exit (للـ built-in) |
| `KLIST_INIT(_name, _get, _put)` | يهيّئ `klist` بـ spinlock و get/put callbacks |
| `to_driver(obj)` | يستخرج `driver_private` من `kobject` باستخدام `container_of` |
| `to_subsys_private(obj)` | يستخرج `subsys_private` من `kobj` |

---

### 1. الـ Structs المهمة

---

#### `struct device_driver` — العمود الفقري للـ driver

**الغرض**: ده الـ struct الأساسي اللي بيمثّل أي driver في الـ Linux kernel driver model. كل driver (PCI, USB, platform, etc.) بيضم واحد من دول.

**الـ Fields المهمة**:

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ driver زي `"e1000e"` — بيظهر في `/sys/bus/*/drivers/` |
| `bus` | `const struct bus_type *` | البـ bus اللي الـ driver بيشتغل عليه (PCI, USB, I2C, ...) |
| `owner` | `struct module *` | الـ module اللي الـ driver بيتبعه — بيتستخدم لإدارة reference counts |
| `mod_name` | `const char *` | اسم الـ module للـ built-in drivers (مش modules قابلة للتحميل) |
| `suppress_bind_attrs` | `bool` | لو `true`، بيمنع `bind`/`unbind` عبر الـ sysfs |
| `probe_type` | `enum probe_type` | sync أو async أو default |
| `of_match_table` | `const struct of_device_id *` | جدول matching للـ Device Tree |
| `acpi_match_table` | `const struct acpi_device_id *` | جدول matching للـ ACPI |
| `probe` | `int (*)(struct device *)` | **أهم callback**: الـ kernel بيستدعيه لما يلاقي device مناسب |
| `sync_state` | `void (*)(struct device *)` | بيتستدعى لما كل consumers اتربطوا بـ drivers |
| `remove` | `int (*)(struct device *)` | بيتستدعى لما device يتشال |
| `shutdown` | `void (*)(struct device *)` | بيتستدعى وقت إيقاف الجهاز |
| `suspend` | `int (*)(struct device *, pm_message_t)` | legacy PM — النوم |
| `resume` | `int (*)(struct device *)` | legacy PM — الصحيان |
| `groups` | `const struct attribute_group **` | attributes بتتضاف للـ driver directory في sysfs |
| `dev_groups` | `const struct attribute_group **` | attributes بتتضاف لكل device مربوط بالـ driver |
| `pm` | `const struct dev_pm_ops *` | مؤشر لـ struct فيه كل callbacks الـ power management |
| `coredump` | `void (*)(struct device *)` | بيتستدعى لما ييجي write على `/sys/.../coredump` |
| `p` | `struct driver_private *` | البيانات الخاصة بالـ driver core — **ما تلمسهاش** |
| `p_cb.post_unbind_rust` | `void (*)(struct device *)` | Rust-only callback بعد `remove()` |

---

#### `struct driver_private` — البيانات الداخلية للـ driver core

**الغرض**: الـ driver core بيستخدمه داخلياً عشان يربط الـ `device_driver` بالـ kobject infrastructure وبالـ devices المربوطة بيه. محدش من خارج `drivers/base/` المفروض يلمسه.

**الـ Fields**:

| الـ Field | النوع | الشرح |
|---|---|---|
| `kobj` | `struct kobject` | الـ kobject بتاع الـ driver — بيمثّله في sysfs |
| `klist_devices` | `struct klist` | قايمة بكل devices المربوطة بالـ driver ده |
| `knode_bus` | `struct klist_node` | العقدة اللي بتربط الـ driver بـ `subsys_private.klist_drivers` |
| `mkobj` | `struct module_kobject *` | مؤشر للـ module kobject (للـ `/sys/module/` entries) |
| `driver` | `struct device_driver *` | مؤشر رجعي للـ `device_driver` الأصلي |

---

#### `struct bus_type` — تعريف الـ bus

**الغرض**: بيعرّف نوع الـ bus (PCI, USB, I2C, platform, إلخ). كل نوع بيكون فيه instance واحد ثابت من `bus_type`.

**الـ Fields المهمة**:

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ bus: `"pci"`, `"usb"`, `"i2c"` |
| `dev_name` | `const char *` | pattern لتسمية الـ devices زي `"i2c-%u"` |
| `bus_groups` | `const struct attribute_group **` | attributes للـ bus directory نفسه |
| `dev_groups` | `const struct attribute_group **` | attributes بتتضاف لكل device على الـ bus |
| `drv_groups` | `const struct attribute_group **` | attributes بتتضاف لكل driver على الـ bus |
| `match` | `int (*)(device, driver)` | **الأهم**: بيقارن device بـ driver — returns 1 لو بيتناسبوا |
| `uevent` | `int (*)(device, env)` | بيضيف env vars للـ uevent |
| `probe` | `int (*)(device)` | الـ bus بيستدعي الـ probe (بيفوّض للـ driver عادةً) |
| `sync_state` | `void (*)(device)` | bus-level sync state |
| `remove` | `void (*)(device)` | بيتستدعى لما device يتشال من الـ bus |
| `shutdown` | `void (*)(device)` | shutdown على مستوى الـ bus |
| `suspend` / `resume` | function pointers | Legacy PM |
| `num_vf` | `int (*)(device)` | عدد الـ Virtual Functions (للـ SR-IOV) |
| `dma_configure` / `dma_cleanup` | function pointers | إعداد وتنظيف الـ DMA |
| `pm` | `const struct dev_pm_ops *` | PM ops على مستوى الـ bus |
| `need_parent_lock` | `bool` | لو `true`، الـ core بيلوك الـ parent device وقت probe/remove |

---

#### `struct subsys_private` — البيانات الداخلية للـ bus/class

**الغرض**: كل `bus_type` أو `class` عنده `subsys_private` واحد بيشيل الحالة الداخلية بتاعته (الـ kobject hierarchy، قوايم الـ devices والـ drivers، الـ mutex).

| الـ Field | النوع | الشرح |
|---|---|---|
| `subsys` | `struct kset` | الـ kset بتاع الـ subsystem كله |
| `devices_kset` | `struct kset *` | الـ kset بتاع الـ devices directory |
| `interfaces` | `struct list_head` | قايمة الـ subsystem interfaces |
| `mutex` | `struct mutex` | بيحمي قوايم الـ devices والـ interfaces |
| `drivers_kset` | `struct kset *` | الـ kset بتاع الـ drivers directory |
| `klist_devices` | `struct klist` | klist لـ iteration على devices |
| `klist_drivers` | `struct klist` | klist لـ iteration على drivers |
| `bus_notifier` | `struct blocking_notifier_head` | قايمة الـ notifiers المسجّلين |
| `drivers_autoprobe` | `unsigned int :1` | لو `1`، الـ bus يعمل auto-probe لما driver يتسجّل |
| `bus` | `const struct bus_type *` | مؤشر رجعي للـ `bus_type` |
| `dev_root` | `struct device *` | الـ parent device الافتراضي |
| `glue_dirs` | `struct kset` | dirs وسيطة لتجنب تعارض الأسماء |
| `class` | `const struct class *` | مؤشر رجعي للـ class (لو كان class مش bus) |

---

#### `struct kobject` — اللبنة الأساسية

**الغرض**: كل entity في الـ driver model (driver, device, bus, class) بيضم `kobject` عشان يكون موجود في الـ sysfs ويكون له reference counting.

| الـ Field | الشرح |
|---|---|
| `name` | اسم الـ entry في sysfs |
| `entry` | ربطه في قايمة الـ parent kset |
| `parent` | الـ kobject الأب (للـ hierarchy) |
| `kset` | الـ kset اللي بينتمي ليه |
| `ktype` | نوعه (بيحدد release callback والـ sysfs ops) |
| `sd` | الـ kernfs node الفعلي في sysfs |
| `kref` | الـ reference count |
| `state_*` | bits بتوصف الحالة الحالية |

---

#### `struct klist` و `struct klist_node` — قوايم thread-safe

**الغرض**: `klist` هي linked list آمنة للاستخدام من threads متعددة بدون deadlocks — بتستخدم reference counting على كل node عشان تمنع use-after-free أثناء الـ iteration.

```c
struct klist {
    spinlock_t    k_lock;   /* protects the list */
    struct list_head k_list;
    void (*get)(struct klist_node *); /* inc ref */
    void (*put)(struct klist_node *); /* dec ref */
};

struct klist_node {
    void            *n_klist;  /* back pointer (opaque) */
    struct list_head n_node;   /* the actual list link */
    struct kref      n_ref;    /* reference count */
};
```

---

#### `struct driver_attribute` — attributes الـ driver في sysfs

**الغرض**: بيعرّف ملف في `/sys/bus/*/drivers/<driver>/` بتقدر تقراه أو تكتب فيه.

```c
struct driver_attribute {
    struct attribute attr;   /* name + permissions */
    ssize_t (*show)(struct device_driver *, char *buf);
    ssize_t (*store)(struct device_driver *, const char *buf, size_t count);
};
```

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
                    ┌─────────────────────────────────────┐
                    │         struct bus_type              │
                    │  name, match(), probe(), pm, ...     │
                    └──────────────────┬──────────────────┘
                                       │ bus_to_subsys()
                                       ▼
                    ┌─────────────────────────────────────┐
                    │        struct subsys_private         │
                    │  subsys (kset)                       │
                    │  devices_kset ──────────────────────►│ /sys/bus/X/devices/
                    │  drivers_kset ──────────────────────►│ /sys/bus/X/drivers/
                    │  klist_devices ─── klist_node ◄──────┤ device_private.knode_bus
                    │  klist_drivers ─── klist_node ◄──────┤ driver_private.knode_bus
                    │  mutex                               │
                    │  bus_notifier                        │
                    └─────────────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │       struct device_driver           │
                    │  name, bus ──────────────────────────┼──► bus_type
                    │  owner (module *)                    │
                    │  probe(), remove(), pm, ...          │
                    │  of_match_table, acpi_match_table    │
                    │  p ──────────────────────────────────┼──► driver_private
                    └─────────────────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────┐
                    │       struct driver_private          │
                    │  kobj (kobject) ─────────────────────┼──► /sys/bus/X/drivers/Y/
                    │  klist_devices ─── klist_node ◄──────┤ device_private.knode_driver
                    │  knode_bus ──────────────────────────┼──► subsys_private.klist_drivers
                    │  mkobj ──────────────────────────────┼──► /sys/module/Y/
                    │  driver ─────────────────────────────┼──► device_driver (back ptr)
                    └─────────────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │         struct kobject               │
                    │  name, parent, kset, ktype           │
                    │  kref (reference count)              │
                    │  sd ─────────────────────────────────┼──► kernfs_node (sysfs)
                    └─────────────────────────────────────┘
                    (مضمون جوا driver_private.kobj و subsys_private.subsys.kobj)
```

---

### 3. مخطط الـ Lifecycle

#### تسجيل الـ driver (driver_register)

```
driver alloc & init (static or kmalloc)
         │
         ▼
   driver_register(drv)
         │
         ├─► kobject_init_and_add(&drv->p->kobj, ...)
         │        └─► ينشئ /sys/bus/<bus>/drivers/<name>/
         │
         ├─► klist_add_tail(&drv->p->knode_bus,
         │                  &bus->p->klist_drivers)
         │        └─► يضيف الـ driver لقايمة الـ bus
         │
         ├─► module_add_driver(drv->owner, drv)
         │        └─► يربطه بـ /sys/module/<name>/drivers/
         │
         ├─► driver_create_file(drv, &driver_attr_uevent)
         │
         ├─► driver_add_groups(drv, drv->groups)
         │
         └─► bus_add_driver(drv)
                  └─► لو drivers_autoprobe=1:
                           driver_attach(drv)
                                └─► يجرب يعمل probe لكل device على الـ bus
```

#### probe لـ device

```
device يتضاف للـ bus أو driver يتسجّل
         │
         ▼
   bus->match(dev, drv) == 1 ?
         │ نعم
         ▼
   device_lock(dev)
         │
         ▼
   really_probe(dev, drv)
         │
         ├─► pinctrl_bind_pins(dev)
         ├─► dev->bus->dma_configure(dev)
         ├─► drv->probe(dev)   ◄── الـ driver code الفعلي
         │       │
         │       ├─► success: dev->driver = drv
         │       │            klist_add_tail(dev, &drv->p->klist_devices)
         │       │            kobject_uevent(KOBJ_BIND)
         │       │
         │       └─► -EPROBE_DEFER: يتضاف لـ deferred_probe_list
         │
         └─► device_unlock(dev)
```

#### إزالة الـ driver (driver_unregister)

```
driver_unregister(drv)
         │
         ├─► bus_remove_driver(drv)
         │        └─► driver_detach(drv)
         │                 └─► لكل device في klist_devices:
         │                          drv->remove(dev)
         │                          klist_remove(knode_driver)
         │                          kobject_uevent(KOBJ_UNBIND)
         │
         ├─► driver_remove_groups(drv, drv->groups)
         │
         ├─► module_remove_driver(drv)
         │
         ├─► klist_remove(&drv->p->knode_bus)
         │
         └─► kobject_put(&drv->p->kobj)
                  └─► لما kref يوصل صفر: driver_private يتتحرر
```

---

### 4. مخطط الـ Call Flow

#### bus->match → driver->probe

```
new device detected (e.g., PCI scan)
  → device_add(dev)
    → bus_probe_device(dev)
      → device_attach(dev)
        → bus_for_each_drv(bus, __device_attach_driver)
          → __device_attach_driver(drv, dev)
            → bus->match(dev, drv)           [e.g., pci_bus_match]
              → compares IDs (vendor/device/class)
              → returns 1 if match
            → device_driver_attach(drv, dev)
              → device_lock(dev)
              → really_probe(dev, drv)
                → drv->bus->dma_configure(dev)
                → drv->probe(dev)            [e.g., e1000_probe]
                  → hardware init
                  → request_irq()
                  → register_netdev()
                → device_unlock(dev)
```

#### driver_find_device

```
driver_find_device(drv, start, data, match_fn)
  → klist_iter_init(&drv->p->klist_devices, &i)
    → loop:
        klist_next(&i)
          → acquires k_lock (spinlock)
          → increments node kref
          → returns next klist_node
        → container_of(node, struct device_private, knode_driver)
        → container_of(dev_priv, struct device, p)
        → match_fn(dev, data)
          → e.g., device_match_name: strcmp(dev_name(dev), name)
        → if match: kobject_get(dev), break
    → klist_iter_exit(&i)
```

#### sync_state flow

```
late_initcall_sync fires
  → device_links_driver_cleanup(dev) [for each consumer]
    → إذا كل consumers اتربطوا بـ drivers:
        → dev_sync_state(dev)
          → if dev->bus->sync_state:
              dev->bus->sync_state(dev)
          → elif dev->driver->sync_state:
              dev->driver->sync_state(dev)
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks الموجودة وما بتحميه

| الـ Lock | النوع | ما بيحميه |
|---|---|---|
| `device_lock(dev)` | `mutex` (في `struct device`) | كل عمليات probe/remove/bind/unbind لـ device بعينها |
| `subsys_private.mutex` | `struct mutex` | قوايم الـ devices والـ interfaces في الـ bus |
| `klist.k_lock` | `spinlock_t` | الـ klist نفسها أثناء add/remove/iteration |
| `kset.list_lock` | `spinlock_t` | قايمة الـ kobjects في الـ kset |
| `kobject.kref` | atomic | reference counting للـ kobject |
| `klist_node.n_ref` | `struct kref` | reference counting لكل node في الـ klist |

#### ترتيب الـ Locks (Lock Ordering) — لتجنب الـ Deadlock

```
الترتيب الصحيح من الخارج للداخل:

1. device_lock(parent)          ← لو need_parent_lock=true
2. device_lock(dev)             ← دايماً قبل أي عملية على الـ device
3. subsys_private.mutex         ← لو محتاج تعدّل قوايم الـ bus
4. klist.k_lock (spinlock)      ← لفترة قصيرة جداً فقط

⚠️ قاعدة مهمة: الـ bus notifiers بيتبعتوا وهو held على device_lock،
   فـ callback الـ notifier ما ينفعش يحاول يعمل device_lock تاني
   على نفس الـ device (deadlock).
```

#### الـ Reference Counting وعلاقته باللوك

الـ `klist` مش بس list، هي list مع reference counting على كل node. ده معناه:

```c
/* أثناء iteration على klist_devices: */
klist_iter_init(&drv->p->klist_devices, &iter);
while ((node = klist_next(&iter))) {
    /* هنا الـ node ref count اتزاد */
    dev = container_of(...);
    /* نقدر نشتغل على dev بأمان حتى لو اتشال من القايمة */
}
klist_iter_exit(&iter); /* بتنزّل الـ ref count للـ node الأخير */
```

الفكرة: حتى لو thread تاني عمل `klist_del(node)` أثناء الـ iteration، الـ node مش هيتحرر من الـ memory لحد ما الـ ref count يوصل صفر — بيمنع use-after-free من غير ما يحتاج تلوك طول وقت الـ iteration.

#### `WRITE_ONCE` / `READ_ONCE` في `device_set_driver`

```c
/* في device_set_driver: */
WRITE_ONCE(dev->driver, (struct device_driver *)drv);
```

**ليه؟** لأن قراءة `dev->driver` في كود الـ uevent بتحصل من غير `device_lock` (عشان ما تبلوكش وقت طويل). الـ `WRITE_ONCE` بيضمن إن الكتابة atomic على مستوى الـ CPU فمحدش يقرأ قيمة نص-نص.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `driver_register` | `int driver_register(struct device_driver *drv)` | تسجيل الـ driver في الـ driver model |
| `driver_unregister` | `void driver_unregister(struct device_driver *drv)` | إزالة الـ driver من الـ driver model |
| `driver_init` | `void driver_init(void)` | تهيئة الـ driver core subsystem أثناء boot |

#### Lookup & Search

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `driver_find` | `struct device_driver *driver_find(name, bus)` | جيب الـ driver بالاسم من bus معين |
| `driver_find_device` | `struct device *driver_find_device(drv, start, data, match)` | iterate على devices اللي bound للـ driver |
| `driver_find_device_by_name` | `static inline ...` | shortcut: ابحث بالاسم |
| `driver_find_device_by_of_node` | `static inline ...` | shortcut: ابحث بالـ DT node |
| `driver_find_device_by_fwnode` | `static inline ...` | shortcut: ابحث بالـ fwnode |
| `driver_find_device_by_devt` | `static inline ...` | shortcut: ابحث بالـ dev_t |
| `driver_find_next_device` | `static inline ...` | الـ device اللي بعد start في قائمة الـ driver |
| `driver_find_device_by_acpi_dev` | `static inline ...` | shortcut: ابحث بالـ ACPI companion |
| `driver_for_each_device` | `int driver_for_each_device(drv, start, data, fn)` | iterate على كل devices الـ driver |

#### sysfs Attributes

| Function | الغرض |
|---|---|
| `driver_create_file` | إضافة sysfs attribute للـ driver |
| `driver_remove_file` | حذف sysfs attribute |
| `driver_set_override` | كتابة قيمة override string مع validation |

#### Probe Synchronization

| Function | الغرض |
|---|---|
| `driver_probe_done` | هل خلص الـ initial probing؟ |
| `wait_for_device_probe` | block لحد ما يخلص كل الـ deferred probes |
| `wait_for_init_devices_probe` | `__init` version — تُستخدم وقت boot فقط |
| `driver_deferred_probe_add` | أضف الـ device لـ deferred probe queue |
| `driver_deferred_probe_check_state` | هل الشروط اتوفرت للـ deferred probe؟ |

#### Macros

| Macro | الغرض |
|---|---|
| `module_driver` | بيولّد `module_init` + `module_exit` بشكل تلقائي |
| `builtin_driver` | بيولّد `device_initcall` للـ built-in drivers |
| `DRIVER_ATTR_RW/RO/WO` | إعلان `driver_attribute` بشكل مختصر |

---

### Group 1: Registration & Lifecycle

الـ registration functions هي الجسر بين الـ driver code والـ driver model. لما `driver_register` تتنفذ، الـ core بيعمل kobject للـ driver في sysfs، وبيحاول يعمل match مع كل الـ devices الموجودة على الـ bus. الـ `driver_unregister` بتعمل العكس بالظبط وبتضمن إن الـ devices تتفصل قبل ما يُحذف الـ driver.

---

#### `driver_register`

```c
int __must_check driver_register(struct device_driver *drv);
```

بتسجّل الـ `device_driver` في الـ bus اللي محدد في `drv->bus`. الـ core بيعمل `driver_private` struct، بيضيف الـ kobject في sysfs تحت `/sys/bus/<bus>/drivers/<name>`، وبيشغّل الـ bus `match()` على كل الـ unbound devices. لو فيه devices كانت deferred، بيعيد تجربتهم.

- **`drv`**: pointer للـ `device_driver` struct — لازم يكون `->bus` و`->name` متعبيين.
- **Return**: `0` نجاح، أو error code سالب (مثلاً `-EBUSY` لو الاسم موجود).
- **Locking**: بتحجز `bus->p->drivers_autoprobe` rwsem داخلياً.
- **Side effects**: بيخلق sysfs entries + بيشغل probe للـ matching devices.
- **Caller context**: process context فقط — ممنوع من interrupt context.
- **`__must_check`**: الـ return value لازم يتفحص — كتير من الـ bugs بتيجي من تجاهله.

```
driver_register(drv)
  ├── driver_find(drv->name, drv->bus)  → if found, error (duplicate)
  ├── bus_add_driver(drv)
  │     ├── alloc driver_private
  │     ├── kobject_init_and_add → sysfs entry
  │     ├── driver_attach(drv)   → try to bind unbound devices
  │     └── add to bus->p->klist_drivers
  └── driver_add_groups(drv, drv->groups)
```

---

#### `driver_unregister`

```c
void driver_unregister(struct device_driver *drv);
```

بتشيل الـ driver من الـ bus، وبتفصل (unbind) كل الـ devices المرتبطة بيه. بتحذف الـ sysfs entries وبتعمل `kobject_put` على الـ driver's kobject. بتنتظر لو فيه operations جارية على الـ driver.

- **`drv`**: نفس الـ pointer اللي اتبعت لـ `driver_register`.
- **Return**: `void` — ولكن بتعمل `WARN_ON` لو الـ driver مش مسجل.
- **Locking**: بتحجز `device_lock` على كل device وقت الـ unbind.
- **Side effects**: بتشغّل `drv->remove()` لكل device مربوطة.
- **Caller context**: process context — بتنام لحد ما يخلص كل الـ unbind operations.

---

#### `driver_init`

```c
void __init driver_init(void);
```

بتهيئ الـ driver core subsystem كله في بداية الـ boot. بتعمل ksets للـ devices والـ buses والـ drivers في sysfs. بتتنادى من `do_basic_setup()` قبل أي `initcall`.

- **Return**: `void`
- **`__init`**: بتتحرر من الذاكرة بعد الـ boot.
- **Caller context**: early boot، single-threaded.

---

### Group 2: Lookup & Search

الـ search functions بتسمح للـ driver code يلاقي devices بطريقة آمنة مع reference counting. الـ core pattern هو `driver_find_device` اللي بتعمل walk على الـ `klist_devices` مع lock مناسب، وبترجع device بعد ما تعمل `get_device()` عليها — يعني المتصل مسؤول يعمل `put_device()` لما يخلص.

---

#### `driver_find`

```c
struct device_driver *driver_find(const char *name, const struct bus_type *bus);
```

بتدور على driver بالاسم في bus معين بتعمل lookup في الـ `bus->p->drivers_kset`. مفيدة لو عايز تتأكد إن driver معين مسجل قبل ما تعمل حاجة.

- **`name`**: اسم الـ driver (string).
- **`bus`**: الـ bus_type اللي هتدور فيه.
- **Return**: pointer للـ `device_driver` لو لقاه (بدون extra ref)، أو `NULL`.
- **Key detail**: مش بتعمل `get` على الـ driver — النتيجة ممكن تتحرر لو حد عمل `driver_unregister` في نفس الوقت. استخدمها بس لو أنت شايل lock تاني.

---

#### `driver_find_device`

```c
struct device *driver_find_device(const struct device_driver *drv,
                                  struct device *start, const void *data,
                                  device_match_t match);
```

الـ generic iterator اللي بتبني عليه كل الـ `driver_find_device_by_*` wrappers. بتعمل walk على `drv->p->klist_devices` من `start` وتشغّل `match()` على كل device لحد ما تلاقي واحدة تعمل match.

- **`drv`**: الـ driver اللي هنتدور في devices بتاعته.
- **`start`**: ابدأ من بعد الـ device دي (أو `NULL` تبدأ من الأول).
- **`data`**: بيتبعت كـ argument تاني لـ `match()`.
- **`match`**: callback — بترجع non-zero لما تلاقي الـ device المطلوبة.
- **Return**: `struct device *` بعد `get_device()` — **لازم** تعمل `put_device()` — أو `NULL`.
- **Locking**: بتحجز `klist_iter` على `drv->p->klist_devices`.

---

#### `driver_find_device_by_name` (inline)

```c
static inline struct device *
driver_find_device_by_name(const struct device_driver *drv, const char *name);
```

Shortcut بتستدعي `driver_find_device(..., name, device_match_name)`. بتقارن `dev_name(dev)` بالـ `name` المطلوب.

---

#### `driver_find_device_by_of_node` (inline)

```c
static inline struct device *
driver_find_device_by_of_node(const struct device_driver *drv,
                               const struct device_node *np);
```

بتدور على device اللي عندها `dev->of_node == np`. لازم في DT-based platforms لما تحتاج تربط بين kernel subsystems.

---

#### `driver_find_device_by_fwnode` (inline)

```c
static inline struct device *
driver_find_device_by_fwnode(struct device_driver *drv,
                              const struct fwnode_handle *fwnode);
```

نفس الفكرة بس بتستخدم `fwnode_handle` — اللي ممكن يكون DT أو ACPI أو software node.

---

#### `driver_find_device_by_devt` (inline)

```c
static inline struct device *
driver_find_device_by_devt(const struct device_driver *drv, dev_t devt);
```

بتدور بـ `dev_t` (major:minor). مفيدة في character/block device drivers.

---

#### `driver_find_next_device` (inline)

```c
static inline struct device *
driver_find_next_device(const struct device_driver *drv, struct device *start);
```

بترجع الـ device اللي بعد `start` في قائمة الـ driver. بتستخدم `device_match_any` — يعني أي device تيجي بعد `start` تنجح. مفيدة للـ manual iteration.

---

#### `driver_find_device_by_acpi_dev` (inline, CONFIG_ACPI)

```c
static inline struct device *
driver_find_device_by_acpi_dev(const struct device_driver *drv,
                                const struct acpi_device *adev);
```

بتدور على device اللي `ACPI_COMPANION(dev) == adev`. لو `CONFIG_ACPI` مش معمول، بترجع `NULL` دايماً.

---

#### `driver_for_each_device`

```c
int __must_check driver_for_each_device(struct device_driver *drv,
                                         struct device *start,
                                         void *data, device_iter_t fn);
```

بتعمل walk على كل الـ devices المربوطة بالـ driver وتشغّل `fn()` على كل واحدة. لو `fn()` رجعت non-zero، الـ iteration بتوقف وبترجع نفس القيمة.

- **`drv`**: الـ driver.
- **`start`**: ابدأ من بعدها أو `NULL`.
- **`data`**: بيتبعت لـ `fn`.
- **`fn`**: `int (*fn)(struct device *, void *)` — بترجع `0` لو تكمل، non-zero لو توقف.
- **Return**: آخر return value من `fn` أو `0`.
- **Locking**: بتحجز ref على كل device أثناء الـ callback.
- **__must_check**: لأن `fn` ممكن ترجع error.

---

### Group 3: sysfs Attributes

الـ sysfs attribute system بيسمح للـ drivers يعرضوا معلومات أو يقبلوا أوامر من user space عبر ملفات في `/sys/bus/<bus>/drivers/<driver>/`.

---

#### `driver_create_file`

```c
int __must_check driver_create_file(const struct device_driver *driver,
                                    const struct driver_attribute *attr);
```

بتضيف sysfs attribute file للـ driver. الـ `attr` بتحدد الاسم والـ permissions وتابعي الـ `show`/`store`.

- **`driver`**: الـ driver اللي هيضاف ليه الـ attribute.
- **`attr`**: الـ `driver_attribute` — بيتعرّف عادةً بـ `DRIVER_ATTR_RW/RO/WO`.
- **Return**: `0` نجاح، أو `-ENODEV` / `-ENOMEM` في حالة فشل.
- **Side effects**: بتعمل kernfs node تحت driver's kobject directory.

---

#### `driver_remove_file`

```c
void driver_remove_file(const struct device_driver *driver,
                        const struct driver_attribute *attr);
```

بتحذف الـ sysfs attribute اللي اتضافت بـ `driver_create_file`. لازم تتنادى في الـ cleanup path أو لو فشلت خطوة بعد `driver_create_file`.

- **`driver`**, **`attr`**: نفس اللي اتبعوا لـ `driver_create_file`.
- **Return**: `void`.

---

#### `driver_set_override`

```c
int driver_set_override(struct device *dev, const char **override,
                        const char *s, size_t len);
```

بتعمل update لـ string override field في device (زي `driver_override` في platform/PCI devices). بتعمل `kstrdup` للـ string الجديدة، وبتحرر القديمة، وبتتعامل مع الـ edge case لما `s` تكون `"\n"` أو فاضية (بتعمل clear للـ override).

- **`dev`**: الـ device اللي هيتغير فيها الـ override.
- **`override`**: pointer لـ pointer الـ string داخل الـ device struct.
- **`s`**, **`len`**: الـ string الجديدة وطولها (من sysfs `store`).
- **Return**: `0` نجاح أو `-ENOMEM`.
- **Locking**: بتحجز `device_lock(dev)` أثناء الـ update.

**مثال عملي:**
```c
/* في sysfs store callback لـ driver_override */
static ssize_t driver_override_store(struct device *dev,
    struct device_attribute *attr, const char *buf, size_t count)
{
    struct platform_device *pdev = to_platform_device(dev);
    int ret;

    ret = driver_set_override(dev, &pdev->driver_override, buf, count);
    if (ret)
        return ret;
    return count;
}
```

---

### Group 4: Probe Synchronization & Deferred Probe

الـ deferred probe mechanism موجود عشان الـ devices ممكن تحتاج resources من devices تانية لسه ما اتعملهاش probe. لو `probe()` رجعت `-EPROBE_DEFER`، الـ core بيضيف الـ device للـ deferred list ويحاول تاني بعد ما أي driver جديد يتسجّل.

---

#### `driver_probe_done`

```c
bool __init driver_probe_done(void);
```

بترجع `true` لو الـ initial synchronous probing خلصت (يعني الـ deferred probe queue فاضية). بتُستخدم وقت الـ boot في `do_initcalls` لتحديد timing.

- **Return**: `true` لو خلص، `false` لو لسه فيه deferred devices.
- **`__init`**: للـ boot phase فقط.

---

#### `wait_for_device_probe`

```c
void wait_for_device_probe(void);
```

بتعمل block وتنام لحد ما يخلص **كل** الـ pending probes (synchronous + asynchronous). بتُستخدم من الـ code اللي محتاج يتأكد إن كل الـ devices اتعملها probe قبل ما يكمل (مثلاً: mount root filesystem).

- **Return**: `void`.
- **Caller context**: process context — بتنام باستخدام `wait_event`.
- **مثال**: kernel يحتاجها قبل `prepare_namespace()`.

---

#### `wait_for_init_devices_probe`

```c
void __init wait_for_init_devices_probe(void);
```

نسخة `__init` من `wait_for_device_probe`، بتُستخدم فقط في `do_initcalls` context أثناء boot. بتضمن إن deferred probes خلصت قبل الـ `late_initcall`.

---

#### `driver_deferred_probe_add`

```c
void driver_deferred_probe_add(struct device *dev);
```

بتضيف الـ device في `deferred_probe_pending_list`. بتتنادى لما `probe()` ترجع `-EPROBE_DEFER` من داخل `really_probe()` في `drivers/base/dd.c`.

- **`dev`**: الـ device اللي فشل probe بتاعها.
- **Return**: `void`.
- **Locking**: بتحجز `deferred_probe_mutex`.

---

#### `driver_deferred_probe_check_state`

```c
int driver_deferred_probe_check_state(struct device *dev);
```

بتفحص هل ينفع نعيد تجربة الـ deferred probe دلوقتي. بترجع `-EPROBE_DEFER` لو الـ system لسه في early boot ومش كل الـ built-in modules اتهيأت. بعد الـ late_initcall، بترجع `-ENXIO` عشان الـ device تعرف إن الـ dependency مش موجودة خالص.

- **`dev`**: الـ device اللي هنفحص حالتها.
- **Return**:
  - `-EPROBE_DEFER`: لسه ممكن تتحل مع الوقت.
  - `-ENXIO`: الـ system بعد الـ init، والـ dependency مش موجودة.
- **Caller context**: process context من داخل `probe()`.

---

### Group 5: Helper Macros

الـ macros دي بتحل مشكلة تكرار boilerplate code في كل driver.

---

#### `module_driver`

```c
#define module_driver(__driver, __register, __unregister, ...)
```

بيولّد `module_init` و `module_exit` functions تلقائياً. كل bus بيعمل wrapper فوقيه زي `module_platform_driver`, `module_i2c_driver`, `module_spi_driver`.

**مثال: تعريف `module_platform_driver`**
```c
/* من include/linux/platform_device.h */
#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
                  platform_driver_unregister)

/* الاستخدام في driver */
static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = { .name = "my-device", .of_match_table = my_of_match },
};
module_platform_driver(my_driver);
/* ↑ بيولد: module_init(my_driver_init) + module_exit(my_driver_exit) */
```

- **`__driver`**: الـ driver struct variable name.
- **`__register`**: function تتنادى في `module_init` (مثلاً `platform_driver_register`).
- **`__unregister`**: function تتنادى في `module_exit`.
- **`...`**: أي arguments إضافية تتبعت للـ register/unregister.

**الكود المولّد:**
```c
/* ما بيولّده module_driver(my_driver, my_register, my_unregister) */
static int __init my_driver_init(void) {
    return my_register(&my_driver);
}
module_init(my_driver_init);

static void __exit my_driver_exit(void) {
    my_unregister(&my_driver);
}
module_exit(my_driver_exit);
```

---

#### `builtin_driver`

```c
#define builtin_driver(__driver, __register, ...)
```

نفس `module_driver` بس بدون `module_exit` — لأن الـ built-in drivers مش بتتحمّل من module، وبالتالي مفيش exit. بيستخدم `device_initcall` بدل `module_init`.

```c
/* مثال: driver مدمج في الكرنل مش module */
builtin_platform_driver(my_builtin_driver);
/* يولّد: device_initcall(my_builtin_driver_init) فقط */
```

---

#### `DRIVER_ATTR_RW / RO / WO`

```c
#define DRIVER_ATTR_RW(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_RW(_name)
#define DRIVER_ATTR_RO(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_RO(_name)
#define DRIVER_ATTR_WO(_name) \
    struct driver_attribute driver_attr_##_name = __ATTR_WO(_name)
```

بيعرّف `driver_attribute` struct جاهز للاستخدام مع `driver_create_file`. الـ `_RW` بيفترض وجود `<name>_show` و `<name>_store` functions. الـ `_RO` بس `show`. الـ `_WO` بس `store`.

```c
/* مثال كامل */
static ssize_t version_show(struct device_driver *drv, char *buf)
{
    return sprintf(buf, "1.0\n");
}
static DRIVER_ATTR_RO(version);  /* يولّد: driver_attr_version */

/* في driver_register أو probe */
driver_create_file(drv, &driver_attr_version);
```

---

### Group 6: الـ `struct device_driver` — الـ Struct الأساسية

```c
struct device_driver {
    const char              *name;              /* اسم الـ driver في sysfs */
    const struct bus_type   *bus;               /* الـ bus اللي شغّال عليه */
    struct module           *owner;             /* THIS_MODULE عادةً */
    const char              *mod_name;          /* للـ built-in modules */
    bool                     suppress_bind_attrs;/* تعطيل bind/unbind من sysfs */
    enum probe_type          probe_type;        /* sync أو async probe */

    /* Firmware matching tables */
    const struct of_device_id   *of_match_table;
    const struct acpi_device_id *acpi_match_table;

    /* Callbacks */
    int   (*probe)    (struct device *dev);
    void  (*sync_state)(struct device *dev);
    int   (*remove)   (struct device *dev);
    void  (*shutdown) (struct device *dev);
    int   (*suspend)  (struct device *dev, pm_message_t state);
    int   (*resume)   (struct device *dev);

    /* sysfs attribute groups */
    const struct attribute_group **groups;      /* على الـ driver نفسه */
    const struct attribute_group **dev_groups;  /* على كل device bound */

    const struct dev_pm_ops *pm;               /* advanced PM ops */
    void  (*coredump) (struct device *dev);

    struct driver_private *p;                  /* private للـ driver core */
    struct {
        void (*post_unbind_rust)(struct device *dev); /* Rust only */
    } p_cb;
};
```

**أهم الـ callbacks:**

| Callback | متى يتنادى | Return |
|---|---|---|
| `probe` | لما device تعمل match | `0` أو error |
| `remove` | لما device تتنزع | `void` |
| `shutdown` | وقت `halt`/`reboot` | `void` |
| `sync_state` | بعد كل consumers يعملوا probe | `void` |
| `suspend`/`resume` | legacy PM | `0` أو error |
| `coredump` | لما user يكتب في sysfs | `void` |

**الـ `probe_type` enum:**

| القيمة | المعنى |
|---|---|
| `PROBE_DEFAULT_STRATEGY` | الـ core يختار sync أو async |
| `PROBE_PREFER_ASYNCHRONOUS` | شغّل الـ probe في workqueue لتسريع الـ boot |
| `PROBE_FORCE_SYNCHRONOUS` | لازم يتعمل sync مع الـ registration |

**ملاحظة على `sync_state`:** بتتنادى مرة واحدة فقط لما **كل** الـ consumers اللي كانوا موجودين وقت `late_initcall` يعملوا bind. لو consumer واحد مش بيعمل bind أبداً، `sync_state` مش هتتنادى. مفيدة جداً للـ power management وتقليل الـ runtime constraints.

---

### مثال عملي متكامل

```c
#include <linux/module.h>
#include <linux/platform_device.h>

/* match table للـ Device Tree */
static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device" },
    {}
};
MODULE_DEVICE_TABLE(of, my_of_match);

static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    /* لو resource مش جاهزة */
    // return -EPROBE_DEFER;  ← driver core هيحطها في deferred list

    dev_info(dev, "probed successfully\n");
    return 0;
}

static void my_remove(struct platform_device *pdev)
{
    dev_info(&pdev->dev, "removed\n");
}

static struct platform_driver my_driver = {
    .probe  = my_probe,
    .remove = my_remove,
    .driver = {
        .name          = "my-device",
        .of_match_table = my_of_match,
        .probe_type    = PROBE_PREFER_ASYNCHRONOUS, /* لتسريع boot */
    },
};

/* يولّد module_init + module_exit تلقائياً */
module_platform_driver(my_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example");
```

**الـ flow الداخلي لما `insmod` يشتغل:**
```
module_init(my_driver_init)
  └── platform_driver_register(&my_driver)
        └── driver_register(&my_driver.driver)
              ├── bus_add_driver(drv)        → sysfs entry
              └── driver_attach(drv)
                    └── for each unbound platform_device:
                          bus->match(dev, drv) → of_driver_match_device()
                          if match: really_probe(dev, drv)
                                     └── my_probe(pdev)
```
## Phase 5: دليل الـ Debugging الشامل

> الـ subsystem ده هو **Linux Driver Model** — بالتحديد `struct device_driver` و `struct bus_type` اللي بيتعاملوا مع binding/unbinding/probing بين الـ drivers والـ devices.

---

### Software Level

#### 1. debugfs entries

الـ driver core بيعمل entries في `/sys/kernel/debug/` مش `/debugfs/` مباشرة، لكن الـ deferred probe system عنده:

```
/sys/kernel/debug/devices_deferred
```

ده بيورّي كل الـ devices اللي فضلت في قائمة الـ deferred probe (يعني `driver_deferred_probe_add` اتنادت عليها).

```bash
# اقرأ الـ devices اللي لسه مش بيلاقوا driver
cat /sys/kernel/debug/devices_deferred

# مثال output:
# platform:serial0 [waiting for: clk-provider]
# i2c-0-0050       [waiting for: regulator-core]
```

كل سطر فيه اسم الـ device وسبب التأجيل. لو الـ device لقاله driver والـ deferred probe اتحلت، السطر ده هيتمسح.

---

#### 2. sysfs entries

الـ driver model بيعمل hierarchy كاملة في `/sys/bus/<bus>/drivers/<driver>/`:

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/<bus>/drivers/` | كل الـ drivers المسجلين على الـ bus |
| `/sys/bus/<bus>/drivers/<drv>/bind` | اكتب اسم device لـ bind يدوي |
| `/sys/bus/<bus>/drivers/<drv>/unbind` | اكتب اسم device لـ unbind يدوي |
| `/sys/bus/<bus>/drivers/<drv>/uevent` | trigger uevent يدوي |
| `/sys/bus/<bus>/devices/` | كل الـ devices على الـ bus |
| `/sys/devices/<path>/driver` | symlink للـ driver المرتبط بالـ device |
| `/sys/devices/<path>/driver_override` | forcibly override driver binding |

```bash
# شوف كل الـ drivers المسجلين على platform bus
ls /sys/bus/platform/drivers/

# شوف إيه الـ driver اللي بيخدم device معين
readlink /sys/devices/platform/serial8250/driver
# output: ../../../../bus/platform/drivers/serial8250

# bind يدوي لـ device على driver معين
echo "0000:00:1f.2" > /sys/bus/pci/drivers/ahci/bind

# unbind device من driver
echo "0000:00:1f.2" > /sys/bus/pci/drivers/ahci/unbind

# تحقق من suppress_bind_attrs
cat /sys/bus/platform/drivers/my_driver/uevent
```

---

#### 3. ftrace — tracepoints وـ events

الـ driver model عنده tracepoints مهمة جداً في `drivers/base/`:

```bash
# فعّل كل events الـ device core
echo 1 > /sys/kernel/debug/tracing/events/devfreq/enable
echo 1 > /sys/kernel/debug/tracing/events/rpm/enable

# الأهم: trace الـ probe calls
echo 1 > /sys/kernel/debug/tracing/events/initcall/enable

# trace الـ driver binding
trace-cmd record -e 'initcall:*' sleep 5
trace-cmd report

# استخدم function tracer على driver_probe_device
echo function > /sys/kernel/debug/tracing/current_tracer
echo driver_probe_device > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل العملية اللي بتسبب المشكلة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

```bash
# trace الـ deferred probe retries
echo 1 > /sys/kernel/debug/tracing/events/probe/enable

# trace كامل لـ probe/remove cycle على driver معين
echo "driver_probe_device" > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk و dynamic debug

```bash
# فعّل dynamic debug لكل كود الـ driver core
echo "file drivers/base/dd.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/bus.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/driver.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ module معين بالاسم
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# أو عن طريق kernel cmdline
# dyndbg="file drivers/base/dd.c +p"

# شوف إيه الـ dynamic debug flags الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep "drivers/base/dd"

# رفع loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# أو
dmesg -n 8
```

الـ `drivers/base/dd.c` هو قلب الـ probe logic — فعّل dynamic debug عليه هتشوف كل خطوة في `driver_probe_device()`.

---

#### 5. Kernel config options للـ debugging

| Config | الفايدة |
|--------|---------|
| `CONFIG_DEBUG_DRIVER` | يفعّل `dev_dbg()` في كل الـ driver core |
| `CONFIG_DEBUG_DEVRES` | يتتبع كل الـ devres allocations/frees |
| `CONFIG_PROVE_LOCKING` | lockdep — يكتشف deadlocks في probe/remove |
| `CONFIG_DEBUG_KOBJECT` | يطبع كل kobject add/remove/put |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يكشف use-after-free في kobject release |
| `CONFIG_DEBUG_SLAB` | يكشف memory corruption في allocations |
| `CONFIG_KASAN` | Address Sanitizer — يكشف memory bugs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dynamic debug infrastructure |
| `CONFIG_DEFERRED_STRUCT_PAGE_INIT` | مش للـ driver لكن بيأثر على probe timing |
| `CONFIG_PM_DEBUG` | debug الـ power management callbacks |
| `CONFIG_PM_SLEEP_DEBUG` | debug suspend/resume في الـ driver |
| `CONFIG_DRIVER_PE_DEBUG` | debug الـ PCIe error handling (لو PCI driver) |

```bash
# تحقق إن الـ options دي موجودة في kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_DEBUG_(DRIVER|DEVRES|KOBJECT)"
```

---

#### 6. devlink وـ subsystem-specific tools

```bash
# devlink لـ drivers اللي بتدعمه (network drivers أساساً)
devlink dev show
devlink dev info pci/0000:00:1f.6

# udevadm لتتبع الـ uevent pipeline
udevadm monitor --kernel --udev --property
# في نافذة تانية: افصل وأعد توصيل الـ device

# udevadm لمعرفة إيه الـ rules اللي بتتطبق
udevadm test /sys/devices/platform/my_device 2>&1

# systemd-analyze لـ probe timing
systemd-analyze blame | head -20

# lsmod مع الـ driver اللي بتتعامل معاه
lsmod | grep my_driver
modinfo my_driver

# تحقق من الـ driver binding الحالي لكل bus
for bus in /sys/bus/*/drivers/*/; do echo "$bus"; ls "$bus" 2>/dev/null; done
```

---

#### 7. جدول رسائل الـ error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---------------------|--------|------|
| `probe with driver X failed with error -EPROBE_DEFER` | الـ driver يحتاج resource مش جاهزة | انتظر أو تحقق من الـ dependency chain في `devices_deferred` |
| `Driver 'X' is already registered, aborting...` | نفس الـ driver اتسجّل مرتين | تحقق من `module_driver()` / `device_initcall()` مش اتنادوا مرتين |
| `kobject_add_internal failed for X with -EEXIST` | اسم الـ kobject موجود فعلاً في sysfs | تأكد إن الـ device name فريد، شوف `dev_name()` |
| `Unable to find driver X` | `driver_find()` مرجعتش حاجة | الـ driver مش متسجّل، أو الـ bus غلط |
| `device_driver_attach: driver probe failed` | الـ `probe()` function رجعت error | راجع الـ probe function نفسها + resources |
| `bind_store: X is not a bus device` | حاولت bind device مش موجود على الـ bus | تأكد من اسم الـ device في `/sys/bus/<bus>/devices/` |
| `Timeout waiting for device X` | `wait_for_device_probe()` انتهت المهلة | الـ async probe لسه شغالة أو علقت |
| `Driver X did not claim device Y` | الـ `match()` function مش بترجع positive | راجع `of_match_table` أو `id_table` في الـ driver |
| `Cannot allocate memory for devres` | نفدت الـ devres allocations | راجع `CONFIG_DEBUG_DEVRES` + memory pressure |
| `synchronous probe of driver X failed` | `PROBE_FORCE_SYNCHRONOUS` driver فشل | ابحث عن سبب الفشل في probe, راجع -errno |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
    int ret;

    /* نقطة 1: تحقق إن الـ driver والـ device متربوطوش قبل ما نبدأ */
    WARN_ON(dev->driver != NULL);

    ret = really_probe(dev, drv);

    /* نقطة 2: لو فشل بـ errno غير متوقع */
    if (ret && ret != -EPROBE_DEFER && ret != -ENOMEM) {
        dev_warn(dev, "unexpected probe failure: %d\n", ret);
        dump_stack();  /* نرى من وين اتنادينا */
    }

    return ret;
}
```

```c
void driver_detach(struct device_driver *drv)
{
    struct device_private *dev_prv;

    /* نقطة 3: تحقق إن اللي شايل الـ lock هو اللي المفروض يشيله */
    WARN_ON(!mutex_is_locked(&drv->p->lock));

    while (!list_empty(&drv->p->klist_devices.k_list)) {
        dev_prv = /* ... */;
        /* نقطة 4: تحقق إن الـ device فعلاً مربوط بالـ driver ده */
        WARN_ON(dev_prv->device->driver != drv);
    }
}
```

```c
/* في الـ driver نفسه أثناء التطوير */
static int my_driver_probe(struct device *dev)
{
    struct resource *res;

    res = platform_get_resource(to_platform_device(dev), IORESOURCE_MEM, 0);

    /* نقطة 5: لو resource مش موجودة — crash early بدل late */
    if (WARN_ON(!res))
        return -EINVAL;

    return 0;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

```bash
# تحقق إن الـ device موجود فعلاً على الـ bus
lspci -vvv -s 0000:00:1f.2         # PCI device
ls /sys/bus/platform/devices/       # platform devices
ls /sys/bus/i2c/devices/            # I2C devices

# تحقق من الـ power state
cat /sys/devices/pci0000:00/0000:00:1f.2/power/runtime_status
# يرجع: active / suspended / error

# تحقق إن الـ device enabled في hardware
setpci -s 0000:00:1f.2 COMMAND      # PCI command register
# bit 0 = I/O space, bit 1 = Memory space, bit 2 = Bus master

# تحقق من interrupt assignment
cat /proc/interrupts | grep my_driver
cat /sys/devices/platform/my_device/irq
```

---

#### 2. Register dump techniques

```bash
# devmem2 — اقرأ register من physical address
# مثال: اقرأ 32-bit من عنوان 0xFED00000
devmem2 0xFED00000 w

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0xFED00000/4)) 2>/dev/null | xxd

# io utility (من package ioport)
io -4 -r 0x3F8    # اقرأ 32-bit من I/O port

# لـ PCI devices: اقرأ config space
setpci -s 0000:00:1f.2 0x10.l    # BAR0
lspci -xxx -s 0000:00:1f.2       # dump كل config space

# لو الـ driver بيعمل ioremap، تقدر تشوف الـ mapped regions
cat /proc/iomem | grep -i "my_device"
cat /proc/ioports
```

```bash
# مثال عملي: تحقق من UART registers
# اعرف الـ base address أولاً
cat /proc/tty/driver/serial
# output: 0: uart:16550A port:3F8 irq:4 tx:0 rx:0

# اقرأ الـ LCR register (offset 3)
devmem2 $((0x3F8 + 3)) b
```

---

#### 3. Logic Analyzer / Oscilloscope tips

للـ driver model نفسه مش محتاج logic analyzer، لكن لو الـ probe بيفشل بسبب hardware:

- **I2C drivers**: شيك على SCL/SDA — الـ `probe()` بتبعت `START + address`، لو مفيش `ACK` هيرجع `-ENXIO`
- **SPI drivers**: شيك على CS (Chip Select) يطلع low وقت الـ probe
- **Platform drivers**: شيك على interrupt line — بعد `devm_request_irq()` الـ IRQ المفروض يكون enabled
- **USB drivers**: الـ D+ / D- بيتغيروا وقت enumeration — لو الـ probe فشل بعد enumeration، ابحث في software

```
Oscilloscope checklist أثناء probe:
  ┌─────────────────────────────────────┐
  │ 1. Power rails stable? (VDD, VDDIO) │
  │ 2. Reset signal deasserted?          │
  │ 3. Clock signal present?             │
  │ 4. IRQ line idle (high)?             │
  │ 5. Bus signals clean (no glitches)?  │
  └─────────────────────────────────────┘
```

---

#### 4. Common hardware issues → kernel log patterns

| مشكلة الـ hardware | الـ pattern في dmesg | التفسير |
|--------------------|---------------------|---------|
| Device مش موجود على الـ bus | `my_driver: probe of X failed with error -19` (ENODEV) | الـ hardware مش موجود فعلاً أو power off |
| Power supply مش كافي | `my_driver: probe of X failed with error -5` (EIO) | الـ device بيرد بـ garbage بسبب power noise |
| IRQ conflict | `genirq: Flags mismatch irq X` | اتنين devices بيطلبوا نفس الـ IRQ بـ flags مختلفة |
| Clock مش جاهز | `probe with driver X failed with error -517` (EPROBE_DEFER) | الـ clock provider لسه مش probe |
| DMA address مش متاح | `DMA: Out of SW-IOMMU space` | الـ device بيحتاج DMA addressing أكتر من المتاح |
| Firmware missing | `firmware: failed to load X.bin` | الـ firmware file مش موجود في `/lib/firmware/` |
| Voltage regulator مش شغال | `regulator_get: X supply not found` ثم EPROBE_DEFER | الـ regulator driver لسه مش probe |

---

#### 5. Device Tree debugging

```bash
# تحقق إن الـ DT nodes موجودة صح
ls /sys/firmware/devicetree/base/

# اقرأ property من DT مباشرة
cat /sys/firmware/devicetree/base/soc/serial@3f8/compatible | xxd
# المفروض تطابق ما في the driver's of_match_table

# fdtdump — عرض الـ DT المترجم
fdtdump /boot/dtb-$(uname -r) 2>/dev/null | grep -A10 "my_device"

# dtc — decompile الـ DTB للـ DTS
dtc -I dtb -O dts /boot/dtb-$(uname -r) 2>/dev/null | grep -B5 -A20 "my_device"

# تحقق من الـ of_match_table في الـ driver
# الـ compatible string في DT يجب تطابق تماماً
grep compatible /sys/firmware/devicetree/base/soc/my_device/

# شوف إيه الـ driver اللي اتعرف على الـ DT node
cat /sys/devices/platform/my_device/of_node/compatible
readlink /sys/devices/platform/my_device/driver
```

```bash
# مثال: driver عنده of_match_table:
# { .compatible = "vendor,my-chip-v2" }
# لكن في DT:
# compatible = "vendor,my-chip-v1";
# النتيجة: probe مش هتتنادى أصلاً
# الحل: fdtdump + مقارنة مع of_match_table في الـ source
```

---

### Practical Commands

#### جاهز للـ copy-paste

```bash
# === 1. شوف إيه الـ devices اللي مش لاقية driver ===
cat /sys/kernel/debug/devices_deferred

# === 2. تتبع الـ probe لـ driver معين ===
echo "file drivers/base/dd.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
dmesg -w &
# شغّل الـ driver أو أعد توصيل الـ device
modprobe my_driver
dmesg | tail -50

# === 3. شوف كل الـ drivers على bus معين ===
for d in /sys/bus/platform/drivers/*/; do
    echo "Driver: $(basename $d)"
    ls "$d" | grep -v uevent | grep -v bind | grep -v unbind | grep -v module
done

# === 4. تتبع كامل لـ probe باستخدام ftrace ===
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo function_graph > current_tracer
echo "driver_probe_device" > set_graph_function
echo 1 > tracing_on
modprobe my_driver
echo 0 > tracing_on
cat trace | head -100

# === 5. Force rebind لـ device على driver ===
DEV="0000:00:14.0"
DRV="xhci_hcd"
echo "$DEV" > /sys/bus/pci/drivers/$DRV/unbind
sleep 1
echo "$DEV" > /sys/bus/pci/drivers/$DRV/bind

# === 6. شوف reference count لـ driver ===
cat /sys/module/my_driver/refcnt

# === 7. تحقق من deferred probe state ===
echo "" > /sys/bus/platform/drivers_probe   # trigger re-probe
cat /sys/kernel/debug/devices_deferred

# === 8. devres debug ===
# لازم kernel مكمبايل بـ CONFIG_DEBUG_DEVRES
# هيطبع تلقائياً في dmesg كل alloc/free

# === 9. شوف الـ uevent pipeline ===
udevadm monitor --kernel &
echo "add" > /sys/devices/platform/my_device/uevent

# === 10. اعرف سبب فشل الـ probe ===
dmesg | grep -i "probe\|driver\|deferred" | tail -30
```

#### تفسير الـ output المتوقع

```
# output من devices_deferred:
platform:serial@3f8 [waiting for: clk]
# المعنى: الـ device ده محتاج clock provider يتسجّل أول

# output من ftrace (function_graph):
 1)               |  driver_probe_device() {
 2)               |    really_probe() {
 3)   0.543 us    |      bus_type::match();         /* match returned 1 */
 4)               |      driver::probe() {
 5) # 1234.5 us   |        my_driver_probe();       /* هنا الوقت الطويل */
 6)               |      }
 7)   0.123 us    |      driver_bound();             /* binding ناجح */
 8)               |    }
 9)               |  }
# لو رقم 5 كبير جداً: الـ probe بتاعتك بطيئة — فكر في async probe

# output من dynamic_debug (dd.c):
drivers/base/dd.c:457: calling my_driver_probe+0x0 @ 3
drivers/base/dd.c:461: my_driver_probe+0x0 returned 0 after 1234 usecs
# الـ returned 0 = probe نجحت
# لو رجعت -EPROBE_DEFER: هتتأجل وترجع تاني

# output من dmesg لـ successful bind:
[   12.345678] my_driver: my_driver_probe: device my_device@3f8 initialized
[   12.345679] my_device 3f8.my_device: driver [my_driver] registered
```

---

### خلاصة الـ debugging workflow

```
مشكلة probe?
     │
     ├─► cat /sys/kernel/debug/devices_deferred
     │       └─ موجود؟ → تحقق من الـ dependency (clock/regulator/gpio)
     │
     ├─► dmesg | grep -i "probe.*failed"
     │       └─ شوف الـ -errno
     │           ├─ EPROBE_DEFER → dependency problem
     │           ├─ ENODEV       → hardware absent
     │           ├─ ENOMEM       → memory issue
     │           └─ غيرهم        → فعّل dynamic_debug على dd.c
     │
     ├─► تحقق من sysfs:
     │   readlink /sys/devices/<path>/driver
     │       └─ مفيش symlink؟ → مفيش bind حصل
     │
     └─► ftrace function_graph على driver_probe_device
             └─ شوف وين بتقف أو بترجع error
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Driver لا بيتعمل probe على RK3562 — مشكلة `of_match_table`

#### العنوان
**Industrial Gateway على RK3562 — UART driver ما بيشتغلش بعد reboot**

#### السياق
شركة بتعمل industrial gateway بتستخدم **Rockchip RK3562** — الـ gateway بيتواصل مع PLC عن طريق RS485 على `/dev/ttyS3`. الـ kernel هو custom build من kernel 6.6. بعد ما الفريق عمل `defconfig` جديدة وبنى الـ kernel من الأول، الـ UART driver بطل يشتغل خالص.

#### المشكلة
```bash
$ dmesg | grep serial
# لا في output خالص — الـ driver ما اتعملش probe
$ ls /dev/ttyS*
# /dev/ttyS0 بس — /dev/ttyS3 مش موجود
```

الـ sysfs بيأكد إن الـ device موجود بس مفيش driver ربطه:

```bash
$ cat /sys/bus/platform/devices/fe670000.serial/driver
# ملقيش output — مفيش driver مربوط
```

#### التحليل

الـ `struct device_driver` فيها الحقل:

```c
struct device_driver {
    const char *name;
    const struct bus_type *bus;
    /* ... */
    const struct of_device_id *of_match_table; /* <-- المشكلة هنا */
    /* ... */
    int (*probe)(struct device *dev);
};
```

الـ `of_match_table` هو اللي بيخلي الـ bus يعمل match بين الـ device node في الـ Device Tree والـ driver. لما الـ kernel bus يعمل `bus->match()` على كل device جديدة، بيقارن compatible strings من الـ DT بقائمة `of_match_table` الموجودة في الـ driver.

في الـ driver code القديم كان فيه:

```c
static const struct of_device_id rk3562_serial_of_match[] = {
    { .compatible = "rockchip,rk3562-uart" },
    { .compatible = "snps,dw-apb-uart" },
    {}
};
MODULE_DEVICE_TABLE(of, rk3562_serial_of_match);

static struct uart_driver rk_uart_driver = {
    .driver = {
        .name           = "rk-serial",
        .of_match_table = rk3562_serial_of_match,
        /* ... */
    },
};
```

لما الفريق عمل refactor للـ driver، نسوا يحطوا `of_match_table` في الـ `device_driver.driver` الصح. الـ field اتحطت في مكان تاني بالغلط.

بدون `of_match_table`، الـ `bus->match()` مش هيلاقي أي compatible match، فالـ `probe()` ما بيتبعتلهاش خالص.

#### الحل

```c
/* تأكد إن of_match_table موجودة في المكان الصح */
static struct platform_driver rk_uart_platform_driver = {
    .probe  = rk_uart_probe,
    .remove = rk_uart_remove,
    .driver = {
        .name           = "rk-serial",
        .owner          = THIS_MODULE,
        .of_match_table = rk3562_serial_of_match, /* لازم تكون هنا */
        .pm             = &rk_uart_pm_ops,
    },
};
```

التحقق من الـ fix:

```bash
$ modinfo rk_serial | grep alias
alias: of:N*T*Crockchip,rk3562-uartC*
alias: of:N*T*Csnps,dw-apb-uartC*

$ ls /sys/bus/platform/devices/fe670000.serial/driver
/sys/bus/platform/devices/fe670000.serial/driver -> ../../../../bus/platform/drivers/rk-serial
```

#### الدرس المستفاد

الـ `of_match_table` في `struct device_driver` هي البوابة الوحيدة للـ DT-based probing. لو مش موجودة أو غلط، الـ kernel ما هيبصش على الـ driver خالص حتى لو الـ DT node سليم 100%. دايماً verify بـ `modinfo` إن الـ aliases اتضافت صح.

---

### السيناريو الثاني: `EPROBE_DEFER` loop لا نهائي على STM32MP1 — I2C sensor

#### العنوان
**IoT Sensor Board على STM32MP1 — I2C temperature sensor عالق في deferred probe**

#### السياق
فريق بيعمل IoT sensor node بيستخدم **STM32MP157** مع INA226 power monitor على I2C bus. الـ board بتستخدم **Yocto** مع kernel 6.1. الـ driver بيشتغل تمام على dev machine بس على الـ production board الـ sensor مش بيظهر في `/sys/class/hwmon/`.

#### المشكلة

```bash
$ dmesg | grep ina226
ina226 1-0040: probe deferred
ina226 1-0040: probe deferred
ina226 1-0040: probe deferred
# بيتكرر كل ثواني بلا نهاية
```

الـ system بيبوت بشكل طبيعي بس الـ sensor مش بيظهر أبدًا.

#### التحليل

الـ `struct device_driver` بيدعم deferred probe من خلال:

```c
void driver_deferred_probe_add(struct device *dev);
int driver_deferred_probe_check_state(struct device *dev);
```

لما الـ `probe()` بيرجع `-EPROBE_DEFER`، الـ core بيضيف الـ device لـ deferred list وبيحاول يعمل probe تاني لما أي driver تاني يتسجل. الـ flow:

```
bus->probe(dev)
  └─> driver->probe(dev)
        └─> returns -EPROBE_DEFER
              └─> driver_deferred_probe_add(dev)
                    └─> ينتظر driver تاني يتسجل
                          └─> يحاول probe تاني
```

المشكلة الحقيقية: الـ INA226 driver بيعمل `devm_regulator_get()` لـ regulator اسمه `"vs"`. الـ regulator driver اتسجل بس الـ DT node مفيهوش `regulator-name = "vs"` الصح، فالـ `regulator_get` بيرجع `-EPROBE_DEFER` كل مرة ظانًا إن الـ regulator لسه ما ظهرش.

```dts
/* الغلط */
vcc_sensor: regulator-vcc {
    compatible = "regulator-fixed";
    regulator-name = "vcc_3v3";  /* الاسم غلط */
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
};

/* الصح */
vcc_sensor: regulator-vcc {
    compatible = "regulator-fixed";
    regulator-name = "vs";  /* لازم يتطابق مع ما الـ driver بيطلبه */
    regulator-min-microvolt = <3300000>;
    regulator-max-microvolt = <3300000>;
};
```

#### الحل

```bash
# أول خطوة: شوف الـ deferred devices
$ cat /sys/kernel/debug/devices_deferred
ina226 1-0040: Driver ina226 requests probe deferral

# شوف الـ regulators المتاحة
$ ls /sys/class/regulator/
regulator.0  regulator.1  ...

$ cat /sys/class/regulator/regulator.1/name
vcc_3v3   # <-- هو ده المشكلة، اسمه غلط
```

بعد تصحيح الـ DT وإعادة البناء:

```bash
$ dmesg | grep ina226
ina226 1-0040: probe succeeded
$ ls /sys/class/hwmon/hwmon2/
curr1_input  in0_input  in1_input  name  power1_input
```

#### الدرس المستفاد

الـ `driver_deferred_probe_add()` هو آلية صحيحة بس ممكن تتحول لـ infinite loop لو الـ dependency اللي الـ driver بينتظرها ما بتيجيش أبدًا. دايماً اتحقق من `/sys/kernel/debug/devices_deferred` واتأكد إن أسماء الـ resources في الـ DT بتتطابق بالظبط مع ما الـ driver بيطلبه.

---

### السيناريو الثالث: `suppress_bind_attrs` و security على i.MX8 — Android TV Box

#### العنوان
**Android TV Box على i.MX8MQ — HDMI driver binding مش محمي**

#### السياق
شركة بتبني **Android TV box** بـ **NXP i.MX8MQ**. الـ security team لاقت إن أي process بـ root access ممكن تعمل unbind للـ HDMI driver من الـ sysfs وتسبب kernel panic أو تقدر تستغل الـ race condition:

```bash
# أي حد بـ root يقدر يعمل كده
$ echo "fb_hdmi" > /sys/bus/platform/unbind
# HDMI اتوقف فجأة!
```

#### المشكلة

الـ HDMI driver مش بيستخدم `suppress_bind_attrs`، فالـ sysfs بيعرض `/sys/bus/platform/drivers/fb_hdmi/bind` و `unbind`، وده بيسمح بـ runtime unbinding.

#### التحليل

في `struct device_driver`:

```c
struct device_driver {
    const char *name;
    const struct bus_type *bus;
    /* ... */
    bool suppress_bind_attrs;   /* disables bind/unbind via sysfs */
    /* ... */
};
```

الـ `suppress_bind_attrs = true` بيخلي الـ driver core لا يعمل sysfs entries لـ `bind` و `unbind`. لما الـ field بتكون `false` (الـ default)، الـ core بينشئ:

```
/sys/bus/<bus>/drivers/<name>/bind
/sys/bus/<bus>/drivers/<name>/unbind
```

وده بيسمح لأي process بـ CAP_SYS_ADMIN يعمل manual bind/unbind وهو حاجة خطيرة على production devices.

#### الحل

```c
static struct platform_driver imx8_hdmi_driver = {
    .probe  = imx8_hdmi_probe,
    .remove = imx8_hdmi_remove,
    .driver = {
        .name                = "fb_hdmi",
        .owner               = THIS_MODULE,
        .of_match_table      = imx8_hdmi_of_match,
        .suppress_bind_attrs = true,  /* <-- امنع bind/unbind من sysfs */
        .pm                  = &imx8_hdmi_pm_ops,
    },
};
```

التحقق:

```bash
# بعد الـ fix
$ ls /sys/bus/platform/drivers/fb_hdmi/
# bind و unbind مش موجودين دلوقتي
$ ls /sys/bus/platform/drivers/fb_hdmi/
uevent  fea80000.hdmi
```

بالإضافة لكده، ممكن تضيف SELinux policy تمنع الوصول:

```
# في الـ sepolicy
neverallow { domain -init } sysfs_driver:file write;
```

#### الدرس المستفاد

**الـ `suppress_bind_attrs`** لازم تكون `true` لأي driver بيتحكم في hardware حساس (display, audio, security hardware) على production systems. الـ default هو `false` وده بيفتح ثغرة لـ runtime manipulation. على Android و embedded systems دايماً راجع كل driver وتأكد من الـ field دي.

---

### السيناريو الرابع: `probe_type` وبطء الـ boot على AM62x — Automotive ECU

#### العنوان
**Automotive ECU على TI AM62x — Boot time بطيء بسبب synchronous probe**

#### السياق
فريق embedded بيعمل **automotive ECU** بيستخدم **Texas Instruments AM62x** (Sitara). الـ requirement إن الـ system يبوت في أقل من 3 ثواني. الـ boot time الحالي 6.5 ثانية بسبب إن كل الـ drivers بتتعمل probe بشكل synchronous واحدة بعد التانية.

#### المشكلة

```bash
$ systemd-analyze
Startup finished in 1.2s (kernel) + 5.3s (userspace) = 6.5s
```

أغلب وقت الـ kernel جاي من probe sequence للـ drivers.

```bash
$ dmesg -T | grep "probe"
[  0.234] platform ffa00000.mcan: probe with driver tcan4x5x
[  0.891] platform ffa00000.mcan: probe complete   # 657ms على driver واحد!
[  0.892] platform 2b00000.usb: probe with driver dwc3
[  1.450] platform 2b00000.usb: probe complete     # 558ms كمان!
```

#### التحليل

الـ `struct device_driver` بيوفر `probe_type` للتحكم في ترتيب الـ probing:

```c
enum probe_type {
    PROBE_DEFAULT_STRATEGY,      /* الـ core يقرر */
    PROBE_PREFER_ASYNCHRONOUS,   /* async إن أمكن — بيسرع الـ boot */
    PROBE_FORCE_SYNCHRONOUS,     /* sync دايماً — آمن بس بطيء */
};

struct device_driver {
    /* ... */
    enum probe_type probe_type;
    /* ... */
};
```

الـ drivers اللي بياخد وقت في probe (USB, CAN, Ethernet) لازم يتعملوا async عشان مش محتاجين يخلصوا قبل ما الـ boot يكمل.

#### الحل

للـ drivers اللي مش critical للـ boot:

```c
/* CAN driver — مش محتاج يخلص probe قبل ما الـ system يبوت */
static struct platform_driver am62x_mcan_driver = {
    .probe  = am62x_mcan_probe,
    .remove = am62x_mcan_remove,
    .driver = {
        .name       = "am62x-mcan",
        .of_match_table = am62x_mcan_of_match,
        .probe_type = PROBE_PREFER_ASYNCHRONOUS, /* <-- async probe */
    },
};

/* USB driver — نفس الشيء */
static struct platform_driver am62x_dwc3_driver = {
    .probe  = am62x_dwc3_probe,
    .remove = am62x_dwc3_remove,
    .driver = {
        .name       = "am62x-dwc3",
        .of_match_table = am62x_dwc3_of_match,
        .probe_type = PROBE_PREFER_ASYNCHRONOUS,
    },
};

/* Clock driver — لازم يخلص الأول قبل أي حاجة تانية تشتغل */
static struct platform_driver am62x_clk_driver = {
    .probe  = am62x_clk_probe,
    .remove = am62x_clk_remove,
    .driver = {
        .name       = "am62x-clk",
        .of_match_table = am62x_clk_of_match,
        .probe_type = PROBE_FORCE_SYNCHRONOUS, /* <-- sync لأن التانيين بيعتمدوا عليه */
    },
};
```

النتيجة:

```bash
$ systemd-analyze
Startup finished in 0.9s (kernel) + 2.1s (userspace) = 3.0s
# من 6.5s لـ 3.0s
```

#### الدرس المستفاد

الـ `probe_type` في `struct device_driver` أداة قوية لتحسين boot time. على automotive systems مع hard real-time boot requirements، راجع كل driver وقرر: هل محتاج يخلص probe قبل ما الـ system يكمل boot؟ اللي مش critical حوله لـ `PROBE_PREFER_ASYNCHRONOUS`.

---

### السيناريو الخامس: `driver_for_each_device` لـ runtime inventory على Allwinner H616

#### العنوان
**Android TV Box على Allwinner H616 — Dynamic USB device inventory system**

#### السياق
شركة بتعمل **Android TV box** بـ **Allwinner H616** (الـ SoC اللي بيستخدمه Orange Pi Zero 2). الـ product بيدعم USB hub مع كذا device (keyboard، mouse، USB-to-Ethernet adapter). محتاجين يعملوا sysfs attribute بيعدد كل الـ USB devices المربوطة بـ driver معين في وقت الـ runtime لأغراض diagnostics.

#### المشكلة

الـ diagnostics tool بتحتاج تعرف كل الـ devices اللي مربوطة بـ `usb_serial` driver في أي وقت بدون ما تعمل parse لـ `/sys/bus/usb/devices/` يدويًا.

#### التحليل

الـ `include/linux/device/driver.h` بيوفر:

```c
int __must_check driver_for_each_device(
    struct device_driver *drv,  /* الـ driver اللي هنمشي على devices بتاعته */
    struct device *start,       /* نبدأ من بعد الـ device دي (NULL = من الأول) */
    void *data,                 /* data بنمررها للـ callback */
    device_iter_t fn            /* الـ callback function */
);
```

الـ function دي بتمشي على كل الـ devices المربوطة بالـ driver وبتنادي الـ callback على كل واحدة.

```c
struct usb_inventory {
    char *buf;
    size_t len;
    size_t max;
};

/* الـ callback اللي بيتنادى على كل device */
static int collect_usb_device_info(struct device *dev, void *data)
{
    struct usb_inventory *inv = data;
    struct usb_device *udev = to_usb_device(dev);

    /* أضف معلومات الـ device للـ buffer */
    inv->len += scnprintf(inv->buf + inv->len,
                          inv->max - inv->len,
                          "%s: VID=%04x PID=%04x\n",
                          dev_name(dev),
                          le16_to_cpu(udev->descriptor.idVendor),
                          le16_to_cpu(udev->descriptor.idProduct));
    return 0; /* كمّل على باقي الـ devices */
}

/* الـ sysfs show function */
static ssize_t usb_inventory_show(struct device_driver *drv, char *buf)
{
    struct usb_inventory inv = {
        .buf = buf,
        .len = 0,
        .max = PAGE_SIZE,
    };

    /* امشي على كل الـ devices المربوطة بالـ driver */
    driver_for_each_device(drv, NULL, &inv, collect_usb_device_info);

    return inv.len;
}

/* سجّل الـ attribute */
static DRIVER_ATTR_RO(usb_inventory);
```

بعدين في الـ driver init:

```c
static int __init h616_usb_serial_init(void)
{
    int ret;

    ret = usb_register(&h616_usb_serial_driver);
    if (ret)
        return ret;

    /* أضف الـ sysfs attribute */
    return driver_create_file(
        &h616_usb_serial_driver.drvwrap.driver,
        &driver_attr_usb_inventory
    );
}
```

الاستخدام:

```bash
# اقرأ الـ inventory من الـ sysfs
$ cat /sys/bus/usb/drivers/usb_serial/usb_inventory
1-1.2:1.0: VID=067b PID=2303   # PL2303 USB-Serial
1-1.3:1.0: VID=1a86 PID=7523   # CH340 USB-Serial
```

#### الدرس المستفاد

الدالة `driver_for_each_device()` مع `DRIVER_ATTR_RO` و `driver_create_file()` بيديك آلية نظيفة لعمل runtime introspection على كل الـ devices المربوطة بـ driver معين. ده أحسن بكتير من الـ userspace parsing لـ sysfs tree، وبيديك interface موحد للـ diagnostics tools.
## Phase 7: مصادر ومراجع

### مقالات LWN.net الأساسية

دي أهم المقالات اللي بتغطي الـ Linux Driver Model وكل المفاهيم المرتبطة بـ `struct device_driver`:

| المقال | الموضوع |
|--------|---------|
| [The zen of kobjects](https://lwn.net/Articles/51437/) | الـ `kobject` اللي بيشكّل العمود الفقري للـ driver model |
| [kobjects and sysfs](https://lwn.net/Articles/54651/) | ربط الـ kobjects بالـ sysfs hierarchy |
| [kobjects and hotplug events](https://lwn.net/Articles/52621/) | الـ uevent وآليات الـ hotplug في الـ driver model |
| [New kobject/kset/ktype documentation](https://lwn.net/Articles/260139/) | توثيق محدث للـ kobject API مع أمثلة كود |
| [Driver porting: Device model overview](https://lwn.net/Articles/31185/) | نظرة عامة على الـ device model من منظور port الـ drivers |
| [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/) | مراجعة شاملة للـ driver model مع التطور التاريخي |
| [Manual driver binding and unbinding](https://lwn.net/Articles/143397/) | الـ bind/unbind عبر sysfs — المقابل لـ `suppress_bind_attrs` |
| [Deferred driver probing](https://lwn.net/Articles/450460/) | آلية الـ deferred probe و`-EPROBE_DEFER` |
| [drivercore: Add driver probe deferral mechanism](https://lwn.net/Articles/485194/) | الـ commit الأول لـ deferred probe في driver core |
| [Device dependencies and deferred probing](https://lwn.net/Articles/662820/) | device links والعلاقة بـ `sync_state` |
| [Asynchronous device/driver probing support](https://lwn.net/Articles/629895/) | الـ `PROBE_PREFER_ASYNCHRONOUS` وتسريع الـ boot |
| [Multi-threaded device probing](https://lwn.net/Articles/192851/) | الخلفية الأقدم لفكرة الـ async probing |
| [add driver matching priorities](https://lwn.net/Articles/121227/) | ترتيب الأولوية في مطابقة الـ driver مع الـ device |

---

### التوثيق الرسمي في kernel.org

**الـ `Documentation/driver-api/driver-model/`** هو المصدر الأساسي:

```
Documentation/driver-api/driver-model/
├── overview.rst       ← نظرة عامة على الـ model كله
├── driver.rst         ← الـ struct device_driver بالتفصيل
├── binding.rst        ← آلية الـ probe والـ binding
├── bus.rst            ← الـ bus_type وعلاقتها بالـ driver
├── device.rst         ← الـ struct device
├── devres.rst         ← managed resources (devm_*)
└── design-patterns.rst← أنماط التصميم الشائعة
```

**الـ `Documentation/kobject.rst`** للأساسيات:

- [Device Drivers — kernel docs](https://docs.kernel.org/driver-api/driver-model/driver.html)
- [The Linux Kernel Device Model — Overview](https://docs.kernel.org/driver-api/driver-model/overview.html)
- [Driver Model Index](https://docs.kernel.org/driver-api/driver-model/index.html)
- [Driver Binding](https://static.lwn.net/kerneldoc/driver-api/driver-model/binding.html)
- [Device drivers infrastructure](https://static.lwn.net/kerneldoc/driver-api/infrastructure.html)

---

### كوميتات مهمة في تاريخ الـ Driver Model

دي أبرز التحولات في تطور الـ `struct device_driver`:

| الميزة | الوصف |
|--------|-------|
| **الـ driver model الأصلي** | Pat Mochel أضاف الـ unified driver model في سلسلة kernel 2.5 |
| **deferred probe** | إضافة `driver_deferred_probe_add()` — يرجع `-EPROBE_DEFER` |
| **async probing** | إضافة `enum probe_type` مع `PROBE_PREFER_ASYNCHRONOUS` |
| **sync_state** | callback للـ sync بين الـ hardware state والـ software state بعد اكتمال الـ consumers |
| **module_driver macro** | تبسيط boilerplate الـ module_init/module_exit |

للبحث في تاريخ الكوميتات:
```bash
git log --oneline drivers/base/driver.c
git log --oneline include/linux/device/driver.h
git log --grep="sync_state" --oneline drivers/base/
```

---

### مناقشات الـ Mailing List

- [LKML Archive](https://lkml.org/) — ابحث عن `device_driver`, `deferred probe`, `sync_state`
- [lore.kernel.org/lkml](https://lore.kernel.org/lkml/) — الأرشيف الرسمي الأحدث
- [spinics.net/lists/kernel](https://www.spinics.net/lists/kernel/) — أرشيف بديل للـ LKML
- [PATCH driver core: Emit reason for pending deferred probe](https://www.spinics.net/lists/kernel/msg5008419.html) — مثال على نقاش driver core في الـ mailing list
- [PATCH: Disable driver deferred probe timeout](https://lore.kernel.org/lkml/2b8c40ba-0068-99ca-6dc8-64d075e9112c@kali.org/T/) — نقاش عن الـ deferred probe timeout

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)

الكتاب المرجعي الأساسي لكتابة الـ drivers — متاح مجانًا:

- **الفصل 14: The Linux Device Model** — بيشرح الـ `kobject`, `kset`, `device_driver` بالتفصيل
- [PDF مباشر من LWN](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)
- الكتاب كاملًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 17: Devices and Modules** — بيغطي الـ device model وعلاقته بالـ sysfs
- بيشرح الـ `kobject` hierarchy وكيف بيتبنى الـ `/sys` منه

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل 8: Device Driver Basics** — مدخل عملي لكتابة الـ drivers
- مناسب لمن بيشتغل على embedded targets مع platform drivers

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- بيغطي الـ driver model بعمق من منظور الـ kernel internals
- مفيد لفهم الـ `driver_private` وآليات الـ reference counting

---

### kernelnewbies.org

- [Drivers — Linux Kernel Newbies](https://kernelnewbies.org/Drivers) — بوابة رئيسية لمصادر كتابة الـ drivers
- [WritingPortableDrivers](https://kernelnewbies.org/WritingPortableDrivers) — كتابة drivers تشتغل على معماريات متعددة
- [Linux_3.10-DriversArch](https://kernelnewbies.org/Linux_3.10-DriversArch) — تغييرات الـ driver architecture في 3.10
- [Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) — تغييرات الـ driver architecture في 3.15
- [Linux_3.1_DriverArch](https://kernelnewbies.org/Linux_3.1_DriverArch) — تغييرات الـ driver architecture في 3.1

---

### elinux.org

- [Linux Device Drivers](https://elinux.org/Linux_Device_Drivers) — صفحة مرجعية شاملة مع روابط للكتب والـ tutorials
- [Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) — مجموعة عروض تقنية عن الـ driver development
- [So you want to write a Linux driver subsystem?](https://elinux.org/images/6/6d/SoYouWantToWriteALinuxDriverSubsystem.pdf) — PDF ممتاز لمن يريد كتابة subsystem كامل
- [Linux Drivers Device Tree Guide](https://elinux.org/index.php?redirect=no&title=Linux_Drivers_Device_Tree_Guide) — دليل الـ Device Tree وعلاقته بالـ driver binding

---

### مصادر إضافية

- [Linux Device Model — linux-kernel-labs](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html) — تمارين عملية مع شرح نظري
- [The Driver Model Core, Part I — Linux Journal](https://www.linuxjournal.com/article/6717) — مقال تاريخي من Linux Journal
- [Troubleshooting deferred probe issues](https://blog.dowhile0.org/2022/06/21/how-to-troubleshoot-deferred-probe-issues-in-linux/) — دليل عملي لتشخيص مشاكل الـ deferred probe

---

### مصطلحات البحث

لإيجاد معلومات أكتر عن الموضوع ده، استخدم الكلمات دي:

```
linux kernel device_driver struct probe
linux driver model kobject sysfs
linux deferred probe EPROBE_DEFER
linux sync_state driver device links
linux probe_type asynchronous synchronous
linux driver_register bus_type matching
linux module_driver builtin_driver macro
linux driver sysfs bind unbind
linux driver_private internal driver core
linux device driver OF match table ACPI
linux driver coredump devcoredump
linux wait_for_device_probe boot sequence
```
## Phase 8: Writing simple module

### الفكرة

**`driver_register()`** هي الفانكشن الأساسية اللي بتتسجّل فيها كل device driver في الـ kernel. كل مرة أي driver بيتضاف للسيستم — سواء عند الـ boot أو عند تحميل module جديد — بتتنادى. ده بيخليها نقطة ممتازة للـ hook باستخدام **kprobe**، لأنها:
- مش بتأثر على الـ system stability لو بس اتفرجنا عليها
- بتكشف اسم الـ driver، الـ bus اللي شغال عليه، ونوع الـ probe

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_driver_register.c
 *
 * Hooks driver_register() to log every driver being registered
 * into the kernel driver model.
 */

/* kprobes API and struct kprobe definition */
#include <linux/kprobes.h>

/* pr_info(), pr_err() macros */
#include <linux/kernel.h>

/* module_init, module_exit, MODULE_LICENSE, etc. */
#include <linux/module.h>

/* struct device_driver, struct bus_type */
#include <linux/device/driver.h>

/*
 * pre_handler - يتنادى مباشرةً قبل ما driver_register() تتنفذ.
 * الـ regs بتحتوي على state المعالج في لحظة الـ intercept.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64, the first argument (struct device_driver *drv)
     * is passed in register RDI.
     */
    struct device_driver *drv = (struct device_driver *)regs->di;

    /* Guard against a NULL pointer just in case */
    if (!drv)
        return 0;

    pr_info("kprobe_drvreg: driver_register() called\n"
            "  driver name : %s\n"
            "  bus         : %s\n"
            "  probe_type  : %d\n"
            "  owner module: %s\n",
            drv->name        ? drv->name        : "(null)",
            (drv->bus && drv->bus->name) ? drv->bus->name : "(no bus)",
            drv->probe_type,
            (drv->owner && drv->owner->name) ? drv->owner->name : "(built-in)");

    return 0; /* 0 = don't alter execution, let the real function run */
}

/*
 * struct kprobe - الـ descriptor بتاع الـ hook.
 * بنحدد فيه اسم الـ symbol اللي عايزين نعمله probe.
 */
static struct kprobe kp = {
    .symbol_name = "driver_register",
    .pre_handler = handler_pre,
};

/*
 * module_init - بيتنادى لما الـ module يتحمّل.
 * بنسجّل الـ kprobe هنا عشان الـ kernel يعرف يعمل patch للـ instruction.
 */
static int __init kprobe_drvreg_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_drvreg: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_drvreg: planted kprobe on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/*
 * module_exit - بيتنادى لما الـ module يتشال بـ rmmod.
 * لازم نشيل الـ kprobe عشان الـ kernel يرجع يعمل restore للـ instruction
 * الأصلية ومفيش dangling pointer في جدول الـ probes.
 */
static void __exit kprobe_drvreg_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_drvreg: kprobe removed from driver_register\n");
}

module_init(kprobe_drvreg_init);
module_exit(kprobe_drvreg_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on driver_register() — logs every driver as it joins the driver model");
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه موجود |
|--------|-----------|
| `<linux/kprobes.h>` | بيعرّف `struct kprobe`، `register_kprobe()`، `unregister_kprobe()` |
| `<linux/kernel.h>` | بيجيب `pr_info()` و `pr_err()` |
| `<linux/module.h>` | ضروري لأي kernel module — `MODULE_LICENSE`، `module_init`، إلخ |
| `<linux/device/driver.h>` | بيعرّف `struct device_driver` و `struct bus_type` اللي بنقرأ منهم |

---

#### الـ `pre_handler`

الـ `pre_handler` بيتنادى قبل ما الـ CPU ينفّذ أول instruction في `driver_register()`. في x86-64 الـ calling convention بيحط الأرغيومنت الأول في الـ register `RDI`، فـ `regs->di` بتديك مباشرةً الـ pointer لـ `struct device_driver`.

بنطبع:
- `drv->name`: اسم الـ driver زي `"ahci"` أو `"e1000e"`
- `drv->bus->name`: اسم الـ bus زي `"pci"` أو `"usb"` أو `"platform"`
- `drv->probe_type`: enum بيقول هل الـ probe sync ولا async
- `drv->owner->name`: اسم الـ kernel module اللي بيسجّل الـ driver ده

---

#### الـ `struct kprobe`

الـ field الوحيد الإلزامي هو `symbol_name`. الـ kernel بيعمل lookup للسيمبول في الـ kallsyms table ويحسب الـ address تلقائي. لو حبيت تعمل probe على offset معين جوه الفانكشن تقدر تستخدم `offset` field.

---

#### الـ `module_init` و `module_exit`

**`register_kprobe()`** بتعمل runtime patching — بتحط `int3` (breakpoint instruction) في أول byte من `driver_register()`. لما الـ CPU يوصلها بيحصل trap، الـ kernel يشغّل الـ `pre_handler` بتاعنا، وبعدين يرجع ينفّذ الفانكشن الأصلية.

**`unregister_kprobe()`** في الـ exit ضرورية جداً: بتشيل الـ `int3` وترجع الـ original bytes. لو نسيتها والـ module اتشال، الـ `pre_handler` pointer بيبقى dangling وأي invocation بعد كده بتكرش الـ kernel.

---

### طريقة التشغيل

```bash
# بناء الـ module (مع Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod kprobe_driver_register.ko

# مشاهدة الـ log (جرّب تحمّل أي driver تاني)
sudo dmesg -w | grep kprobe_drvreg

# إزالة الـ module
sudo rmmod kprobe_driver_register
```

**مثال على output متوقع:**

```
kprobe_drvreg: planted kprobe on driver_register at ffffffff815a2340
kprobe_drvreg: driver_register() called
  driver name : xhci_hcd
  bus         : pci
  probe_type  : 0
  owner module: xhci_pci
kprobe_drvreg: driver_register() called
  driver name : e1000e
  bus         : pci
  probe_type  : 0
  owner module: e1000e
```

---

### ملاحظة على الـ `probe_type` values

| قيمة | المعنى |
|------|--------|
| `0` | `PROBE_DEFAULT_STRATEGY` — الـ kernel يختار sync أو async |
| `1` | `PROBE_PREFER_ASYNCHRONOUS` — مناسب للأجهزة البطيئة أو اللي مش لازم تبقى جاهزة في الـ boot |
| `2` | `PROBE_FORCE_SYNCHRONOUS` — الـ driver لازم يتـ probe قبل ما الـ registration ترجع |
