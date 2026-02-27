## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `include/linux/device/class.h` جزء من **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS** subsystem — المسؤول عنه Greg Kroah-Hartman. ده القلب اللي بيشغّل كل إدارة الأجهزة في Linux kernel.

---

### المشكلة اللي بيحلها — القصة من الأول

تخيّل معايا إنك بتبني مكتبة ضخمة فيها آلاف الكتب. كل كتاب له مؤلف مختلف، ناشر مختلف، وطريقة طباعة مختلفة. لو سبت الناس يدوّروا على كتاب بحسب *"مين طبعه"* أو *"على أنهي آلة طُبع"*، هيتهوّروا. الأسهل إنك تقسّم المكتبة **بالموضوع**: قسم روايات، قسم علوم، قسم تاريخ — بغض النظر عن طريقة الطباعة.

الـ Linux kernel واجه نفس المشكلة بالظبط مع الأجهزة:

- عندك SCSI disk على bus نوعه SCSI
- عندك ATA disk على bus نوعه ATA (IDE)
- عندك NVMe disk على bus نوعه PCIe

الثلاثة **قرص تخزين**. المستخدم (أو الـ userspace application) مش مهتم بالفرق — هو عايز يكتب ويقرأ بيانات بس.

**الـ class** هو الحل: بدل ما userspace يتعامل مع SCSI device أو ATA device، هو بيتعامل مع `disk` class. الـ kernel هو اللي يتعامل مع التفاصيل دي من تحت.

---

### الـ Device Class — المفهوم النظري

**الـ `class`** في Linux هو **نظرة عالية المستوى** للجهاز بتجرّد منك التفاصيل التقنية. الفكرة:

```
Hardware Level:          Class Level (Abstract):
─────────────────        ────────────────────────
SCSI Disk           ──►
ATA Disk            ──►  "disk" class   ──►  /sys/class/block/
NVMe Disk           ──►

USB Mouse           ──►
PS/2 Mouse          ──►  "input" class  ──►  /sys/class/input/
Touchpad            ──►

TCP Socket          ──►
UDP Socket          ──►  "net" class    ──►  /sys/class/net/
WiFi Interface      ──►
```

الـ userspace (أو udev أو systemd) بيشوف `/sys/class/block/` ويلاقي كل الـ block devices في مكان واحد، مش مهم هما على أنهي bus.

---

### هدف الملف `class.h`

الملف ده **header** بيعرّف:

1. **`struct class`** — الـ struct الأساسي اللي بيوصف أي class في الـ kernel (زي `block_class`، `input_class`، `net_class`...)
2. **`struct class_dev_iter`** — أداة للمشي على كل الأجهزة اللي مسجّلة في class معين
3. **`struct class_attribute`** — attributes بتظهر في `/sys/class/<name>/` نفسها (مش في الأجهزة)
4. **`struct class_interface`** — آلية للكود اللي عايز يتنبّه لما device يتضاف أو يتشال من class
5. **Functions API** زي `class_register`, `class_destroy`, `class_find_device`, `class_for_each_device`

---

### الـ `struct class` — تفصيلة سريعة

```c
struct class {
    const char *name;  // اسم الـ class زي "block" أو "input"

    // attributes تظهر في /sys/class/<name>/
    const struct attribute_group **class_groups;
    // attributes تظهر في كل device تحت الـ class
    const struct attribute_group **dev_groups;

    // يتبعت لـ udev لما device يتضاف/يتشال
    int (*dev_uevent)(...);
    // بيحدد اسم الـ device node في /dev/
    char *(*devnode)(...);

    // cleanup callbacks
    void (*class_release)(...);
    void (*dev_release)(...);

    // power management
    const struct dev_pm_ops *pm;
};
```

---

### مثال واقعي: class الـ `input`

لما بتوصّل mouse USB:

1. الـ USB subsystem بيكتشف الجهاز
2. بيعمل `struct device` جديدة
3. بيسجّلها في **`input_class`**
4. الـ udev بيستقبل **uevent** (عن طريق `dev_uevent` callback)
5. بيعمل `/dev/input/mouse0` بالـ permissions الصح
6. الـ application بيفتح `/dev/input/mouse0` بدون ما يعرف إنه USB أصلاً

---

### الـ `class_interface` — فكرة ذكية

لو عندك code عايز يتعمل فيه حاجة **لكل device في class معين** تلقائيًا:

```c
struct class_interface {
    struct list_head node;
    const struct class *class;

    int  (*add_dev)(struct device *dev);    // اتضاف device جديد
    void (*remove_dev)(struct device *dev); // اتشال device
};
```

مثلاً: الـ `serio` subsystem عنده interface على `input_class` عشان كل ما يتضاف input device يعمل mapping تلقائي.

---

### الـ `class_compat` — backward compatibility

القديم كان في `/sys/class/<name>/<device>/` symlinks لـ bus devices. الـ `class_compat` بيحافظ على اللينكات دي عشان الـ userspace tools القديمة ما تتكسرش.

---

### الفرق بين `class` و `bus_type`

| الجانب | `bus_type` | `class` |
|--------|-----------|---------|
| السؤال | الجهاز متوصّل ازاي؟ | الجهاز بيعمل إيه؟ |
| مثال | USB, PCI, I2C | block, input, net |
| المسار في sysfs | `/sys/bus/` | `/sys/class/` |
| الـ driver binding | نعم (match/probe) | لأ |
| الهدف | تنظيم hardware topology | تجريد وظيفي للـ userspace |

---

### الملفات المهمة اللي المفروض تعرفها

**الـ Core Implementation:**
- `drivers/base/class.c` — التطبيق الفعلي لكل functions المعرّفة في `class.h`
- `drivers/base/core.c` — قلب الـ device model كله
- `drivers/base/bus.c` — تطبيق الـ bus_type المكمّل للـ class

**الـ Headers:**
- `include/linux/device/class.h` — الملف ده نفسه
- `include/linux/device/bus.h` — الـ bus_type API (مُدرَج في class.h)
- `include/linux/device.h` — الـ `struct device` الأساسية
- `include/linux/kobject.h` — الـ kobject الأساسي (العمود الفقري لـ sysfs)

**أمثلة على Drivers بتستخدمه:**
- `drivers/input/input.c` — بيعرّف `input_class`
- `drivers/block/` — بيستخدم `block_class`
- `net/core/net-sysfs.c` — بيستخدم `net_class`
- `drivers/tty/tty_io.c` — بيستخدم `tty_class`

**الـ Documentation:**
- `Documentation/driver-api/driver-model/` — الوثائق الرسمية للـ driver model
## Phase 2: شرح الـ Device Class Framework

### المشكلة — ليه الـ class موجود أصلاً؟

الـ kernel بيتعامل مع hardware من خلال buses (PCI, USB, I2C, platform, ...) وكل bus عنده طريقة مختلفة في probe وmatch والـ lifecycle. لكن من وجهة نظر userspace، مش مهم disk ده connected على SCSI ولا SATA ولا USB — كلهم "disks" وكلهم عايزين `/dev/sdX` وعايزين يتعاملوا بنفس الطريقة.

**المشكلة المحددة:**
- الـ bus model بيوصف **كيف** الجهاز متوصل (physical topology).
- فيه حاجة لازم تعبر عن **ماذا** يعمل الجهاز (functional role).
- بدون abstraction موحدة، كل driver لازم يعمل `/dev` entry بنفسه، يديق sysfs بنفسه، ويتعامل مع udevd بنفسه — كل ده boilerplate ومكرر.

---

### الحل — الـ class abstraction

الـ **`struct class`** بتقدم "تصنيف وظيفي" للأجهزة. بدل ما كل driver يعمل كل حاجة من الصفر، الـ class بيوفر:

1. **مكان مشترك في sysfs**: كل devices من نفس الـ class بتظهر تحت `/sys/class/<classname>/`.
2. **devnode callback موحد**: بيحدد اسم الـ device node في `/dev`.
3. **uevent hooks مشتركة**: بيضيف environment variables لـ udevd لما جهاز اتضاف أو اتشال.
4. **PM callbacks افتراضية**: power management مشترك للـ class كله.
5. **class attributes**: attributes موحدة تظهر في sysfs على مستوى الـ class.

---

### الـ Analogy — شركة توصيل

تخيل شركة logistics كبيرة فيها مركبات مختلفة: عربيات، موتوسيكلات، شاحنات. كل واحدة متوصلة بطريقة مختلفة (fuel type، engine size، load capacity). لكن من وجهة نظر العميل، كلهم "مركبات توصيل" وليهم interface موحد.

| مفهوم الشركة | مقابله في الـ kernel |
|---|---|
| نوع المركبة (عربية / شاحنة) | `struct bus_type` — طريقة الاتصال |
| وظيفة التوصيل (توصيل طرود) | `struct class` — ماذا يفعل الجهاز |
| رقم المركبة في الأسطول | `dev_t` — major/minor |
| لوحة المعلومات الموحدة | sysfs directory تحت `/sys/class/` |
| عقد الخدمة الموحد | `class_interface` |
| إشعار للعميل لما مركبة جديدة | `dev_uevent` → udevd → udev rules |
| مركز إدارة الأسطول | `subsys_private` داخلياً في الـ kernel |

لما مركبة جديدة (device) بتنضم للأسطول، المركز (class framework) بيُشعر udevd (إشعار العميل) اللي بدوره بيعمل `/dev` entry بناءً على udev rules.

---

### Big Picture — مكان الـ class في الـ kernel

```
User Space
 ┌───────────────────────────────────────────────────────┐
 │  /dev/sda  /dev/ttyS0  /dev/video0  /dev/input/event0 │
 │       ↑          ↑          ↑               ↑          │
 │                     udevd                              │
 └──────────────────────┬────────────────────────────────┘
                        │ uevent (netlink)
─────────────────────────────────────────────────────────
Kernel Space
 ┌─────────────────────────────────────────────────────────────┐
 │                    sysfs  (/sys)                            │
 │  /sys/class/block/    /sys/class/tty/    /sys/class/input/ │
 │         ↑                   ↑                  ↑           │
 │  ┌──────┴──────┐   ┌────────┴───────┐  ┌───────┴───────┐  │
 │  │ class:block │   │  class: tty    │  │ class: input  │  │
 │  └──────┬──────┘   └────────┬───────┘  └───────┬───────┘  │
 │         │                   │                  │           │
 │  ┌──────▼──────────────────▼──────────────────▼─────────┐ │
 │  │              Device Class Framework                   │ │
 │  │   class_register / class_dev_iter / class_interface   │ │
 │  └────────────────────────┬──────────────────────────────┘ │
 │                           │                                │
 │  ┌────────────────────────▼──────────────────────────────┐ │
 │  │                   struct device                       │ │
 │  │   (embedded في كل driver: block_device, tty_struct)  │ │
 │  └───────┬───────────────────────────────────┬───────────┘ │
 │          │                                   │             │
 │  ┌───────▼──────┐                   ┌────────▼─────────┐  │
 │  │  bus_type    │                   │  device_driver   │  │
 │  │ (PCI/USB/..) │                   │ (scsi/ahci/xhci) │  │
 │  └──────────────┘                   └──────────────────┘  │
 └─────────────────────────────────────────────────────────────┘
```

**ملاحظة مهمة**: الـ bus بيصف *كيف* الجهاز متوصل، والـ class بيصف *ماذا* يفعل الجهاز. جهاز واحد ممكن يكون على `bus_type = usb` وفي نفس الوقت `class = block`.

---

### الـ Core Abstraction — `struct class`

```c
struct class {
    const char  *name;  /* اسم الـ class → /sys/class/<name>/ */

    /* sysfs attributes على مستوى الـ class نفسه */
    const struct attribute_group **class_groups;

    /* sysfs attributes تتضاف لكل device في الـ class */
    const struct attribute_group **dev_groups;

    /* بيتبعت لـ udevd لما device يتضاف/يتشال */
    int (*dev_uevent)(const struct device *dev, struct kobj_uevent_env *env);

    /* بيحدد اسم وـ permissions الـ /dev node */
    char *(*devnode)(const struct device *dev, umode_t *mode);

    /* cleanup لما الـ class نفسه يتحذف */
    void (*class_release)(const struct class *class);

    /* cleanup لما device في الـ class يتحذف */
    void (*dev_release)(struct device *dev);

    /* بيتكلم قبل driver shutdown */
    int (*shutdown_pre)(struct device *dev);

    /* namespace support لـ containers */
    const struct kobj_ns_type_operations *ns_type;
    const void *(*namespace)(const struct device *dev);

    /* ownership لـ sysfs entries (uid/gid) */
    void (*get_ownership)(const struct device *dev, kuid_t *uid, kgid_t *gid);

    /* default PM ops للـ class كله */
    const struct dev_pm_ops *pm;
};
```

الـ `struct class` مفيهاش state — هي **descriptor ثابت** (const في معظم الكود الحديث). الـ state الفعلي موجود في `struct subsys_private` اللي الـ kernel core بيخصصه داخلياً لما تعمل `class_register()`.

---

### الـ subsys_private — اللي الـ class بيستخدمه داخلياً

الـ `struct subsys_private` (معرفة في `drivers/base/base.h`) هي الـ runtime state للـ class:

```
struct class
    └── (via class_register) → struct subsys_private
                                    ├── struct klist klist_devices   ← قائمة كل devices في الـ class
                                    ├── struct kobject glue_dir      ← /sys/class/<name>/
                                    ├── struct list_head interfaces  ← كل class_interface مسجلة
                                    └── struct mutex mutex           ← thread safety
```

لما تعمل `class_register()`، الـ kernel بيخصص `subsys_private` ويربطها بالـ class. لما تعمل `class_dev_iter_init()`، الـ iterator بيشتغل على الـ `klist_devices` الموجودة في الـ `subsys_private`.

---

### الـ class_dev_iter — كيف بنـloop على devices

```c
struct class_dev_iter {
    struct klist_iter   ki;   /* iterator على klist — thread-safe مع reference counting */
    const struct device_type *type;  /* filter اختياري */
    struct subsys_private   *sp;     /* pointer للـ private state */
};
```

الـ **`struct klist`** مش `list_head` عادي — كل `klist_node` عنده `kref` خاص بيه. ده معناه إنك لو عملت `class_dev_iter_next()` وحد حذف الـ device في نفس الوقت، الـ reference بيمنع الـ memory من التحرير لحد ما تخلص من الـ iterator. ده بيحل race condition كلاسيكي.

**الاستخدام الصحيح:**
```c
struct class_dev_iter iter;
struct device *dev;

class_dev_iter_init(&iter, &my_class, NULL, NULL);
while ((dev = class_dev_iter_next(&iter))) {
    /* آمن تماماً حتى لو devices بتتضاف/تتحذف */
    do_something(dev);
}
class_dev_iter_exit(&iter);  /* مهم: بيحرر الـ reference على آخر device */
```

---

### الـ class_interface — Observer Pattern في الـ kernel

```c
struct class_interface {
    struct list_head    node;           /* في قائمة interfaces الـ class */
    const struct class *class;          /* الـ class اللي بتراقبه */

    int  (*add_dev)   (struct device *dev);    /* بيتكلم لما device يتضاف */
    void (*remove_dev)(struct device *dev);    /* بيتكلم لما device يتشال */
};
```

ده **Observer Pattern** خالص. أي subsystem تاني عايز يعمل حاجة لما device جديد يظهر في class معين، بيعمل `class_interface_register()`.

**مثال حقيقي**: الـ `input` subsystem بيستخدم class_interface لإنشاء `/dev/input/eventX` تلقائياً لما input device جديد يتضاف، بدون ما الـ input driver نفسه يعرف حاجة عن `/dev`.

---

### الـ class_attribute و class_attribute_string

```c
struct class_attribute {
    struct attribute attr;   /* اسم + permissions في sysfs */
    ssize_t (*show) (const struct class *class,
                     const struct class_attribute *attr, char *buf);
    ssize_t (*store)(const struct class *class,
                     const struct class_attribute *attr,
                     const char *buf, size_t count);
};
```

ده attributes على مستوى الـ class نفسه — مش على مستوى device. مثلاً `/sys/class/net/` ممكن يكون فيه attribute بيبين عدد الـ network interfaces الكلي.

الـ `CLASS_ATTR_RW`, `CLASS_ATTR_RO`, `CLASS_ATTR_WO` macros بتختصر التعريف:

```c
/* بدل ما تكتب الـ struct يدوياً */
CLASS_ATTR_RO(version);
/* بيتحول لـ */
struct class_attribute class_attr_version = __ATTR_RO(version);
/* وده بيفترض إن في function اسمها version_show */
```

الـ `class_attribute_string` ده special case — بتعرف static string مرة وتظهرها في sysfs بدون ما تكتب show function:

```c
CLASS_ATTR_STRING(my_attr, 0444, "hello from class");
/* → /sys/class/myclass/my_attr يرجع "hello from class" */
```

---

### الـ class_compat — للـ Backward Compatibility فقط

```c
struct class_compat *class_compat_register(const char *name);
void class_compat_create_link(struct class_compat *cls, struct device *dev);
```

ده legacy mechanism من أيام الـ kernel القديمة لما `/sys/class/` كانت structured بشكل مختلف. بيعمل symlinks في `/sys/class/<name>/` تشاور على devices. **الكود الجديد مش المفروض يستخدمه.**

---

### الـ Find Helpers — كيف تلاقي device في class

الـ header بيوفر مجموعة inline wrappers فوق `class_find_device()`:

```c
/* البحث بالاسم */
class_find_device_by_name(&block_class, "sda");

/* البحث بالـ device tree node */
class_find_device_by_of_node(&i2c_bus_type, np);

/* البحث بالـ firmware node (ACPI أو DT) */
class_find_device_by_fwnode(&gpio_class, fwnode);

/* البحث بالـ major:minor */
class_find_device_by_devt(&block_class, MKDEV(8, 0));
```

كلهم بيستخدموا `class_find_device()` اللي بتعمل `class_dev_iter` داخلياً وبترجع device مع **reference count مرفوع** — لازم تعمل `put_device()` بعد ما تخلص.

---

### ما الذي الـ class يملكه مقابل ما يفوضه للـ driver

```
┌─────────────────────────────────────────────────────────────┐
│                    CLASS OWNS                               │
├─────────────────────────────────────────────────────────────┤
│ • /sys/class/<name>/ directory                              │
│ • قائمة (klist) بكل devices في الـ class                   │
│ • إشعار udevd بـ KOBJ_ADD / KOBJ_REMOVE events             │
│ • تنفيذ class_find_device / class_for_each_device          │
│ • إدارة class_interface observers                           │
│ • thread-safe iteration عبر klist                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                 CLASS DELEGATES TO DRIVER                   │
├─────────────────────────────────────────────────────────────┤
│ • dev_uevent: إيه الـ env vars اللي udevd يحتاجها          │
│ • devnode: اسم الـ /dev node وـ permissions                 │
│ • dev_release: إزاي تحرر الـ device struct                 │
│ • class_release: إزاي تحرر الـ class نفسه                  │
│ • pm: تفاصيل الـ power management للـ class                │
│ • namespace: في حالة container isolation                    │
└─────────────────────────────────────────────────────────────┘
```

---

### مثال كامل — تسجيل class بسيط

```c
#include <linux/device/class.h>

/* الـ class descriptor — static لأن مفيهوش state */
static struct class my_sensor_class = {
    .name = "sensor",
    /* بيحدد /dev/sensor0, /dev/sensor1, ... */
    .devnode = sensor_devnode,
    /* بيضيف SENSOR_TYPE للـ uevent environment */
    .dev_uevent = sensor_uevent,
    /* بيتحرر لما الـ device يتشال */
    .dev_release = sensor_dev_release,
};

static int __init sensor_init(void)
{
    /* بيعمل /sys/class/sensor/ ويخصص subsys_private */
    return class_register(&my_sensor_class);
}

/* لما تضيف device للـ class */
device_initialize(&sdev->dev);
sdev->dev.class = &my_sensor_class;
sdev->dev.devt  = MKDEV(SENSOR_MAJOR, minor);
dev_set_name(&sdev->dev, "sensor%d", minor);
device_add(&sdev->dev);
/* ده بيعمل:
 * - /sys/class/sensor/sensor0/
 * - يبعت uevent → udevd → يعمل /dev/sensor0
 * - يضيف الـ device لـ klist في subsys_private
 */
```

---

### علاقة الـ class بالـ subsystems التانية

| Subsystem | علاقته بالـ class |
|---|---|
| **kobject / sysfs** | الأساس — الـ class عبارة عن kobject hierarchy في sysfs |
| **klist** | الـ mechanism اللي الـ class بيحتفظ بيه بقائمة devices، مع thread-safe iteration |
| **PM (dev_pm_ops)** | الـ class بيقدر يحدد default PM behavior للـ devices بتاعته |
| **uevents / udevd** | الـ class هو اللي بيتحكم في format الـ uevents اللي udevd بيقراها عشان يعمل /dev nodes |
| **bus_type** | الـ bus والـ class مستقلين — device واحد ممكن يكون على USB bus وفي block class في نفس الوقت |

الـ **kobject subsystem** مهم تفهمه قبل الـ class: كل kobject هو object في sysfs بـ reference counting. الـ class نفسه مبني على kobjects — كل `/sys/class/<name>/` هو kobject، وكل device entry جواه هو kobject كمان.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Config Options والـ Macros — Cheatsheet

#### الـ `enum kobject_action` (مستخدم مع uevents)

| القيمة | المعنى |
|--------|--------|
| `KOBJ_ADD` | device اتضاف للـ class |
| `KOBJ_REMOVE` | device اتشال من الـ class |
| `KOBJ_CHANGE` | حصل تغيير في الـ device |
| `KOBJ_MOVE` | الـ device اتنقل لـ parent تاني |
| `KOBJ_ONLINE` | الـ device بقى online |
| `KOBJ_OFFLINE` | الـ device بقى offline |
| `KOBJ_BIND` | driver اتربط بالـ device |
| `KOBJ_UNBIND` | driver اتفصل من الـ device |

#### الـ Config Options المؤثرة

| الـ Config | التأثير |
|-----------|---------|
| `CONFIG_ACPI` | يفعّل `class_find_device_by_acpi_dev()` |
| `CONFIG_UEVENT_HELPER` | يفعّل `uevent_helper[]` path للـ userspace |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يضيف `delayed_work` في `kobject` لـ debug التحرير |

#### الـ Macros للـ class attributes

| الـ Macro | الناتج |
|-----------|--------|
| `CLASS_ATTR_RW(_name)` | attribute قابلة للقراءة والكتابة |
| `CLASS_ATTR_RO(_name)` | attribute للقراءة فقط |
| `CLASS_ATTR_WO(_name)` | attribute للكتابة فقط |
| `CLASS_ATTR_STRING(_name, _mode, _str)` | attribute ثابتة قيمتها string |

---

### 1. الـ Structs المهمة

---

#### `struct class`

**الغرض:** ده الـ struct الأساسي اللي بيمثل "صنف" من الـ devices — زي `net`، أو `block`، أو `input`. بيجمع كل الـ devices اللي بتعمل نفس الوظيفة تحت تصنيف واحد في sysfs تحت `/sys/class/<name>/`.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `name` | `const char *` | اسم الـ class — بيظهر في `/sys/class/` |
| `class_groups` | `const struct attribute_group **` | attributes تظهر في دليل الـ class نفسه |
| `dev_groups` | `const struct attribute_group **` | attributes تظهر لكل device تنتمي للـ class |
| `dev_uevent` | function pointer | بيتنادى لما device اتضاف أو اتشال — بيضيف env vars للـ uevent |
| `devnode` | function pointer | بيرجع اسم الـ devtmpfs node والـ mode — زي `/dev/input/event0` |
| `class_release` | function pointer | بيتنادى لما الـ class نفسه بيتحرر |
| `dev_release` | function pointer | بيتنادى لما device تنتمي للـ class بتتحرر |
| `shutdown_pre` | function pointer | بيتنادى قبل الـ driver shutdown وقت إقفال النظام |
| `ns_type` | `const struct kobj_ns_type_operations *` | callbacks للـ namespace في sysfs — مهم لـ network namespaces |
| `namespace` | function pointer | بيرجع الـ namespace اللي ينتمي له الـ device |
| `get_ownership` | function pointer | بيحدد uid/gid لمجلدات sysfs الخاصة بالـ devices |
| `pm` | `const struct dev_pm_ops *` | عمليات الـ power management الافتراضية للـ class |

**الارتباطات:** `struct class` بيتخزن جواه `subsys_private` (internal, مش مُعلَن في الـ header ده) اللي بيحمل الـ `klist` الخاص بالـ devices.

---

#### `struct class_dev_iter`

**الغرض:** بيمثل iterator لإجراء loop على كل الـ devices المسجلة في class معين — بيستخدم الـ `klist` آلية للـ thread-safe iteration.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `ki` | `struct klist_iter` | الـ iterator الحقيقي على الـ klist |
| `type` | `const struct device_type *` | فلتر اختياري — لو مش NULL بيرجع devices من النوع ده بس |
| `sp` | `struct subsys_private *` | pointer للـ subsystem private data اللي فيها الـ klist |

**الاستخدام الطبيعي:**
```c
struct class_dev_iter iter;
struct device *dev;

class_dev_iter_init(&iter, &my_class, NULL, NULL);
while ((dev = class_dev_iter_next(&iter))) {
    /* process dev */
}
class_dev_iter_exit(&iter);
```

---

#### `struct class_attribute`

**الغرض:** بيمثل attribute واحدة تظهر في مجلد الـ class في sysfs — زي `/sys/class/net/uevent`.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `attr` | `struct attribute` | الـ base attribute — فيها name وmode |
| `show` | function pointer | بيتنادى لما userspace يقرأ الـ attribute |
| `store` | function pointer | بيتنادى لما userspace يكتب في الـ attribute |

---

#### `struct class_attribute_string`

**الغرض:** نسخة مبسطة من `class_attribute` للـ attributes اللي قيمتها string ثابتة فقط (read-only).

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `attr` | `struct class_attribute` | الـ attribute الأساسية |
| `str` | `char *` | الـ string الثابتة اللي هتظهر |

---

#### `struct class_interface`

**الغرض:** بيسمح لـ subsystem يشترك في أحداث class معين — لما device تتضاف أو تتشال من الـ class. الـ `input` subsystem بيستخدمه مثلاً لتطبيق evdev.

| الفيلد | النوع | الشرح |
|--------|-------|-------|
| `node` | `struct list_head` | بيربطه في قايمة الـ interfaces الخاصة بالـ class |
| `class` | `const struct class *` | الـ class اللي بيتابعه |
| `add_dev` | function pointer | بيتنادى لما device جديدة تتضاف للـ class |
| `remove_dev` | function pointer | بيتنادى لما device تتشال من الـ class |

---

#### `struct klist` و `struct klist_iter` (من `klist.h`)

**الغرض:** `klist` هو linked list آمن للـ concurrent access — بيستخدم reference counting على كل node عشان iteration مش بتحتاج تـ hold lock طول الوقت.

| الفيلد | الشرح |
|--------|-------|
| `k_lock` | spinlock يحمي الـ list |
| `k_list` | الـ list_head الحقيقي |
| `get` / `put` | callbacks لـ reference counting |

**الـ `klist_iter`:**
| الفيلد | الشرح |
|--------|-------|
| `i_klist` | pointer للـ klist |
| `i_cur` | الـ node الحالية |

---

#### `struct kobject` (من `kobject.h`)

**الغرض:** الـ building block لأي object في الـ kernel driver model — بيوفر اسم، parent، sysfs entry، وreference counting.

| الفيلد | الشرح |
|--------|-------|
| `name` | اسم الـ object في sysfs |
| `parent` | الـ kobject الأب — بيحدد مكانه في شجرة sysfs |
| `kset` | الـ kset اللي ينتمي له |
| `ktype` | نوع الـ object — بيحدد release وsysfs ops |
| `sd` | `kernfs_node` — الـ entry الحقيقي في sysfs |
| `kref` | reference counter |
| `state_*` | bit flags لحالة الـ object |

---

### 2. مخططات علاقات الـ Structs

```
                         ┌─────────────────────────────────┐
                         │          struct class            │
                         │─────────────────────────────────│
                         │ name                            │
                         │ class_groups ──────────────────►│ attribute_group[]
                         │ dev_groups ────────────────────►│ attribute_group[]
                         │ dev_uevent()                    │
                         │ devnode()                       │
                         │ class_release()                 │
                         │ dev_release()                   │
                         │ shutdown_pre()                  │
                         │ ns_type ───────────────────────►│ kobj_ns_type_operations
                         │ namespace()                     │
                         │ get_ownership()                 │
                         │ pm ────────────────────────────►│ dev_pm_ops
                         └─────────────────────────────────┘
                                        │
                                        │ class_register()
                                        ▼
                         ┌─────────────────────────────────┐
                         │       subsys_private (内部)      │
                         │─────────────────────────────────│
                         │ class_subsys (kset)             │
                         │ klist_devices ─────────────────►│ klist of devices
                         │ interfaces ────────────────────►│ list of class_interface
                         │ glue_dirs (kobject)             │
                         └─────────────────────────────────┘
                                        │
                          ┌─────────────┼─────────────┐
                          ▼             ▼             ▼
                    struct device  struct device  struct device
                    (class_dev_iter iterates these via klist)


┌──────────────────────────────────────────────────────────────┐
│                    class_dev_iter                            │
│──────────────────────────────────────────────────────────────│
│ ki (klist_iter) ──────────────────────────────────────────► │
│   i_klist ─────────────────────────────────────────────────►│ subsys_private.klist_devices
│   i_cur ───────────────────────────────────────────────────►│ current klist_node
│ type ──────────────────────────────────────────────────────►│ device_type (filter, optional)
│ sp ────────────────────────────────────────────────────────►│ subsys_private
└──────────────────────────────────────────────────────────────┘


┌──────────────────────────────────┐
│       struct class_interface     │
│──────────────────────────────────│
│ node (list_head) ◄──────────────────── linked in subsys_private.interfaces
│ class ──────────────────────────►│ struct class (back pointer)
│ add_dev()                        │
│ remove_dev()                     │
└──────────────────────────────────┘


┌──────────────────────────────────┐
│      struct class_attribute      │
│──────────────────────────────────│
│ attr (struct attribute)          │
│   name                           │
│   mode                           │
│ show()                           │
│ store()                          │
└──────────────────────────────────┘
           ▲
           │ embeds
┌──────────────────────────────────┐
│   struct class_attribute_string  │
│──────────────────────────────────│
│ attr (class_attribute)           │
│ str ────────────────────────────►│ static string value
└──────────────────────────────────┘
```

---

### 3. دورة حياة الـ Class — من الإنشاء للتدمير

#### المسار الأول: static class (الـ driver يعرفه بنفسه)

```
┌──────────────────────────────────────────────────────────────────┐
│                        LIFECYCLE                                 │
│                                                                  │
│  1. DEFINE                                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ static const struct class my_class = {                  │    │
│  │     .name = "mydev",                                    │    │
│  │     .dev_release = my_dev_release,                      │    │
│  │ };                                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                       │
│                          ▼                                       │
│  2. REGISTER                                                     │
│  class_register(&my_class)                                       │
│    → allocates subsys_private                                    │
│    → creates /sys/class/mydev/                                   │
│    → registers class_groups attributes                           │
│    → sends KOBJ_ADD uevent                                       │
│                          │                                       │
│                          ▼                                       │
│  3. IN USE                                                       │
│  device_register(dev)  ──► dev->class = &my_class               │
│    → creates /sys/class/mydev/<devname>/                         │
│    → calls class->dev_uevent() → adds env vars                  │
│    → notifies all registered class_interfaces (add_dev)         │
│    → registers dev_groups attributes on the device              │
│                          │                                       │
│  class_dev_iter_init()   │                                       │
│  class_dev_iter_next()  ─┤ iterating devices                    │
│  class_dev_iter_exit()   │                                       │
│                          │                                       │
│  device_unregister(dev) ─► calls class_interfaces (remove_dev)  │
│    → sends KOBJ_REMOVE uevent                                    │
│    → removes /sys/class/mydev/<devname>/                         │
│    → calls class->dev_release(dev)                               │
│                          │                                       │
│                          ▼                                       │
│  4. UNREGISTER                                                   │
│  class_unregister(&my_class)                                     │
│    → removes /sys/class/mydev/                                   │
│    → frees subsys_private                                        │
│    → calls class->class_release() if set                        │
└──────────────────────────────────────────────────────────────────┘
```

#### المسار الثاني: dynamic class (بتستخدم `class_create`)

```
  class_create("mydev")
    → kmalloc(struct class)
    → class_register()
    → يرجع pointer للـ class الجديد
           │
           ▼
    استخدام عادي...
           │
           ▼
  class_destroy(cls)
    → class_unregister()
    → kfree(cls)
```

---

### 4. مخططات تدفق الـ Calls

#### تسجيل الـ Class

```
driver module init
  └─► class_register(class)
        └─► kobject_set_name(&cp->subsys.kobj, class->name)
              └─► kset_register(&cp->subsys)
                    └─► kobject_add(&kset->kobj)
                          └─► sysfs_create_dir_ns()
                                └─► creates /sys/class/<name>/
        └─► add_class_attrs(class)
              └─► class_create_file() per attribute in class_groups
```

#### إضافة Device للـ Class

```
device_register(dev)   [dev->class = &my_class]
  └─► device_add(dev)
        └─► device_add_class_symlinks(dev)
              └─► sysfs_create_link()  → /sys/class/<name>/<dev>/
        └─► device_add_attrs(dev)
              └─► class->dev_groups attributes
        └─► kobject_uevent(KOBJ_ADD)
              └─► class->dev_uevent(dev, env)  → adds env vars
        └─► class_interface_notify_add_device(dev)
              └─► list_for_each(interface, &cp->interfaces)
                    └─► interface->add_dev(dev)
```

#### الـ Iteration على devices الـ Class

```
caller
  └─► class_dev_iter_init(&iter, class, start, type)
        └─► klist_iter_init_node(&cp->klist_devices, &iter->ki, ...)
  └─► class_dev_iter_next(&iter)
        └─► klist_next(&iter->ki)          ← acquires kref on node
              └─► filters by device_type if iter->type != NULL
              └─► returns struct device *
  └─► [process device]
  └─► class_dev_iter_next(&iter)  [repeat...]
  └─► class_dev_iter_exit(&iter)
        └─► klist_iter_exit(&iter->ki)     ← releases kref on current node
```

#### `class_for_each_device` (wrapper للـ iteration)

```
class_for_each_device(class, start, data, fn)
  └─► class_dev_iter_init()
  └─► loop:
        dev = class_dev_iter_next()
        if dev == NULL → break
        get_device(dev)              ← extra ref لحماية الـ device
        error = fn(dev, data)        ← caller callback
        put_device(dev)
        if error → break
  └─► class_dev_iter_exit()
  └─► return error
```

#### `class_find_device` (بيدور على device بشرط معين)

```
class_find_device(class, start, data, match)
  └─► class_for_each_device(class, start, &info, __class_find_device_helper)
        └─► for each dev:
              if match(dev, data) == true:
                get_device(dev)      ← يزود الـ ref count
                return dev
  └─► return NULL if not found
```

#### تسجيل `class_interface`

```
class_interface_register(iface)
  └─► get_class(iface->class)         ← ref on class
  └─► mutex_lock(&cp->mutex)
  └─► list_add_tail(&iface->node, &cp->interfaces)
  └─► for each existing device in class:
        iface->add_dev(dev)           ← notify about existing devices
  └─► mutex_unlock(&cp->mutex)
```

#### إنشاء class attribute في sysfs

```
class_create_file(class, attr)
  └─► class_create_file_ns(class, attr, NULL)
        └─► sysfs_create_file_ns(
              &cp->subsys.kobj,       ← parent = /sys/class/<name>/
              &attr->attr,
              ns
            )
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks المستخدمة

| الـ Lock | النوع | يحمي ماذا؟ |
|----------|-------|------------|
| `subsys_private->mutex` | `mutex` | قايمة الـ interfaces (`cp->interfaces`) وإضافة/حذف devices للـ class |
| `klist.k_lock` | `spinlock_t` | قايمة الـ devices داخل الـ klist أثناء add/remove |
| `klist_node.n_ref` | `kref` | حماية كل device node أثناء iteration |
| `kobject.kref` | `kref` | reference count على الـ class kobject نفسه |

#### ترتيب الـ Locks (Lock Ordering)

```
subsys_private->mutex        [outer lock - يُمسك أولاً]
    └─► klist.k_lock         [inner lock - spinlock قصيرة المدة]
```

**قاعدة مهمة:** مش المفروض تمسك `klist.k_lock` وتحاول تمسك `mutex` — ده هيعمل deadlock.

#### الـ klist وآلية الـ Reference Counting

الـ `klist_iter` بيعمل `kref_get` على الـ node الحالية قبل ما يرجعها للـ caller، وبيعمل `kref_put` على الـ node السابقة. ده بيخلي:

- الـ lock بتتمسك بس وقت الـ traversal الفعلي (مش طول الوقت).
- الـ device مش هتتحرر وهي بتتعمل عليها iteration حتى لو حصل concurrent `device_unregister`.

```
klist_iter_next():
  spin_lock(&klist->k_lock)
    find next node
    klist_node_get(next_node)    ← زود ref
    klist_node_put(current_node) ← نقص ref على السابق (قد يحرر)
  spin_unlock(&klist->k_lock)
  return next_node
```

#### الـ class_interface والـ mutex

لما `class_interface_register` بيتنادى، بيمسك الـ mutex قبل ما يضيف الـ interface للقايمة ويبلغ الـ devices الموجودة — ده بيضمن إن مفيش device بتتضاف أو بتتشال في نفس الوقت.

```
class_interface_register():
  mutex_lock(&cp->mutex)     ← يمنع أي device add/remove
    list_add(iface)
    for_each_device → iface->add_dev()
  mutex_unlock(&cp->mutex)

device_add_to_class():
  mutex_lock(&cp->mutex)     ← نفس الـ mutex
    klist_add(dev)
    for_each_interface → iface->add_dev(dev)
  mutex_unlock(&cp->mutex)
```

ده بيضمن إن الـ `class_interface` مش هيفوّت أي device — إما بيشوفها في الـ loop الأولى أو بيشوفها لما تتضاف بعد التسجيل.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function | Category | Description |
|---|---|---|
| `class_register()` | Registration | بتسجل الـ class في الـ driver core |
| `class_unregister()` | Registration | بتشيل الـ class من الـ driver core |
| `class_is_registered()` | Query | بتتحقق إن الـ class متسجل |
| `class_create()` | Dynamic | بتعمل class ديناميكياً من الـ heap |
| `class_destroy()` | Dynamic | بتحذف class اتعمل بـ `class_create` |

#### Compatibility Layer

| Function | Category | Description |
|---|---|---|
| `class_compat_register()` | Compat | بتسجل class قديم للـ backward compat |
| `class_compat_unregister()` | Compat | بتشيل الـ compat class |
| `class_compat_create_link()` | Compat | بتعمل symlink لجهاز في الـ compat class |
| `class_compat_remove_link()` | Compat | بتحذف الـ symlink بتاع جهاز من الـ compat class |

#### Device Iteration

| Function | Category | Description |
|---|---|---|
| `class_dev_iter_init()` | Iterator | بتبدأ iteration على الأجهزة التابعة لـ class |
| `class_dev_iter_next()` | Iterator | بتجيب الجهاز الجاي في الـ iteration |
| `class_dev_iter_exit()` | Iterator | بتخلص الـ iteration وتحرر الـ references |
| `class_for_each_device()` | Iterator | بتلف على كل الأجهزة التابعة لـ class وتستدعي callback |
| `class_find_device()` | Search | بتدور على جهاز بـ matching function |

#### Device Search Helpers (inline wrappers)

| Function | Match Criterion |
|---|---|
| `class_find_device_by_name()` | Device name string |
| `class_find_device_by_of_node()` | Device Tree node pointer |
| `class_find_device_by_fwnode()` | Firmware node handle |
| `class_find_device_by_devt()` | `dev_t` (major/minor) |
| `class_find_device_by_acpi_dev()` | ACPI companion device |

#### Sysfs Attributes

| Function | Category | Description |
|---|---|---|
| `class_create_file_ns()` | Sysfs | بتضيف attribute file للـ class مع namespace support |
| `class_remove_file_ns()` | Sysfs | بتشيل attribute file من الـ class مع namespace |
| `class_create_file()` | Sysfs | wrapper بدون namespace |
| `class_remove_file()` | Sysfs | wrapper بدون namespace |
| `show_class_attr_string()` | Sysfs | show handler جاهز للـ static strings |

#### Class Interface

| Function | Category | Description |
|---|---|---|
| `class_interface_register()` | Interface | بتسجل interface يستقبل add/remove events للأجهزة |
| `class_interface_unregister()` | Interface | بتشيل الـ interface |

---

### Group 1: Registration & Lifecycle

الـ registration group هي الـ entry/exit points بتاعة أي class في الـ kernel driver model. الـ `class` هو abstraction layer فوق الـ bus — بيسمح لـ userspace يشوف الأجهزة بناءً على وظيفتها مش طريقة تعريفها.

---

#### `class_register`

```c
int __must_check class_register(const struct class *class);
```

بتأخذ الـ `struct class` اللي المبرمج معرّفها وبتسجّلها في الـ `class_subsys` داخل `/sys/class/`. بتعمل `subsys_private` جوّاها وبتربطها بالـ kobject hierarchy. لازم تتحقق من الـ return value بسبب `__must_check`.

**Parameters:**
- `class`: pointer للـ `struct class` الـ statically أو dynamically معرّفة. الـ struct لازم تعيش طول عمر الـ class.

**Return value:** `0` عند النجاح، أو negative error code (مثلاً `-ENOMEM`, `-EEXIST`).

**Key details:**
- بتعمل allocate لـ `subsys_private` داخلياً وبتحفظها في الـ private kernel structures.
- بتضيف الـ `class_groups` كـ sysfs attributes تلقائياً.
- الـ caller context: init time أو module load، مش في interrupt context.

---

#### `class_unregister`

```c
void class_unregister(const struct class *class);
```

عكس `class_register`، بتشيل الـ class من الـ sysfs وبتحرر الـ `subsys_private`. لازم كل الأجهزة التابعة للـ class تتشال قبل استدعائها، غير كده ممكن يبقى undefined behavior.

**Parameters:**
- `class`: نفس الـ pointer اللي اتبعت لـ `class_register`.

**Return value:** void.

**Key details:**
- بتستدعي `class_release` callback لو موجودة.
- مش thread-safe مع حد بيعمل iteration على نفس الـ class في نفس الوقت.
- الـ caller context: module unload أو cleanup path.

---

#### `class_is_registered`

```c
bool class_is_registered(const struct class *class);
```

بتتحقق إن الـ class معندهاش `NULL` private pointer، اللي معناها إنها متسجلة. مفيدة في الـ defensive checks.

**Parameters:**
- `class`: الـ class المراد التحقق منها.

**Return value:** `true` لو متسجلة، `false` لو لأ.

**Key details:** لا locking — snapshot check بس. ممكن تتغير الحالة فوراً بعد الاستدعاء.

---

#### `class_create`

```c
struct class * __must_check class_create(const char *name);
```

بتعمل `struct class` ديناميكياً من الـ heap وبتسجّلها تلقائياً. مفيدة لـ drivers محتاجة تعمل class وقت الـ runtime بدون تعريف static. الـ `__must_check` لأن بترجع pointer ممكن يكون error مشفر (IS_ERR).

**Parameters:**
- `name`: اسم الـ class — بيظهر كـ `/sys/class/<name>/`.

**Return value:** pointer للـ class الجديدة، أو `ERR_PTR()` عند الخطأ. لازم تتحقق بـ `IS_ERR()`.

**Pseudocode flow:**
```
class_create(name):
    alloc struct class
    set class->name = name
    call class_register()
    if error:
        free class
        return ERR_PTR(err)
    return class
```

**Key details:**
- الـ class المعمولة بيها `owner` pointer داخلي للـ module.
- لازم تتحرر بـ `class_destroy()` بس، مش `class_unregister()` مباشرةً.

---

#### `class_destroy`

```c
void class_destroy(const struct class *cls);
```

بتعمل unregister وتحرر الـ class اللي اتعملت بـ `class_create`. آمنة لاستدعائها حتى لو الـ pointer `NULL` أو ERR_PTR.

**Parameters:**
- `cls`: الـ class المراد تحريرها.

**Return value:** void.

**Key details:** بتتحقق من `IS_ERR_OR_NULL` داخلياً، فآمن في الـ cleanup paths.

---

### Group 2: Compatibility Layer

الـ compat layer موجودة بسبب تغيير الـ sysfs layout في إصدارات قديمة من الـ kernel. بعض الـ userspace tools كانت بتتوقع symlinks في أماكن معينة.

---

#### `class_compat_register`

```c
struct class_compat *class_compat_register(const char *name);
```

بتعمل كيان خاص (مش `struct class` عادية) بيسمح بإنشاء symlinks لأجهزة تحت اسم class قديم في sysfs. بتُستخدم لدعم الـ backward compatibility مع `/sys/class/` layouts قديمة.

**Parameters:**
- `name`: اسم الـ compat class.

**Return value:** pointer لـ `struct class_compat` أو `NULL` عند الفشل.

---

#### `class_compat_unregister`

```c
void class_compat_unregister(struct class_compat *cls);
```

بتحرر الـ `struct class_compat` وبتشيل الـ kset بتاعها من sysfs.

**Parameters:**
- `cls`: الـ compat class المراد تحريرها.

---

#### `class_compat_create_link`

```c
int class_compat_create_link(struct class_compat *cls, struct device *dev);
```

بتعمل symlink من الـ compat class directory لـ sysfs entry بتاع الجهاز. الـ symlink اسمه اسم الجهاز.

**Parameters:**
- `cls`: الـ compat class.
- `dev`: الجهاز المراد عمل link له.

**Return value:** `0` أو negative error.

---

#### `class_compat_remove_link`

```c
void class_compat_remove_link(struct class_compat *cls, struct device *dev);
```

بتشيل الـ symlink اللي عملته `class_compat_create_link`.

**Parameters:**
- `cls`: الـ compat class.
- `dev`: الجهاز المراد حذف الـ link بتاعه.

---

### Group 3: Device Iteration

الـ iteration group بتوفر طريقة thread-safe للمرور على أجهزة الـ class. بتستخدم الـ `klist` لضمان إن الجهاز مش بيتحذف وإحنا شايلين reference عليه.

---

#### `class_dev_iter_init`

```c
void class_dev_iter_init(struct class_dev_iter *iter,
                         const struct class *class,
                         const struct device *start,
                         const struct device_type *type);
```

بتهيّئ الـ iterator بتاع الأجهزة التابعة لـ class معينة. بتستخدم الـ `klist_iter_init_node` داخلياً، اللي بيعمل reference على العنصر الحالي لمنع حذفه.

**Parameters:**
- `iter`: buffer الـ iterator (stack allocated عادةً).
- `class`: الـ class المراد الـ iteration عليها.
- `start`: جهاز للبدء منه (أو `NULL` للبدء من الأول).
- `type`: filter بـ `device_type` (أو `NULL` لإرجاع كل الأجهزة).

**Return value:** void.

**Key details:**
- لازم تتبعها بـ `class_dev_iter_exit()` دايماً حتى لو ما فيش أجهزة.
- الـ `subsys_private` بيتجلب من الـ class ويُحفظ في الـ iter.

---

#### `class_dev_iter_next`

```c
struct device *class_dev_iter_next(struct class_dev_iter *iter);
```

بترجع الجهاز الجاي في الـ iteration. بتتخطى الأجهزة اللي مش من الـ type المطلوب (لو اتحدد type في `init`). بتعمل `get_device()` على الجهاز الجاي وتعمل `put_device()` على السابق — safe traversal.

**Parameters:**
- `iter`: الـ iterator المهيأ مسبقاً.

**Return value:** pointer للجهاز الجاي، أو `NULL` لو خلص الـ list.

**Key details:**
- الـ caller بيمسك reference على الجهاز اللي رجعت — الـ reference بتتحرر تلقائياً في الـ next call أو في `iter_exit`.
- مش safe تعمل `device_unregister()` على الجهاز الحالي جوه الـ loop إلا بحذر.

---

#### `class_dev_iter_exit`

```c
void class_dev_iter_exit(struct class_dev_iter *iter);
```

بتنهي الـ iteration وبتحرر الـ reference على الجهاز الأخير. لازم تتستدعى دايماً بعد `class_dev_iter_init`.

**Parameters:**
- `iter`: الـ iterator المراد تنظيفه.

**Return value:** void.

---

#### `class_for_each_device`

```c
int class_for_each_device(const struct class *class,
                           const struct device *start,
                           void *data,
                           device_iter_t fn);
```

**الـ typedef:** `typedef int (*device_iter_t)(struct device *dev, void *data);`

بتلف على كل الأجهزة التابعة للـ class من `start` وبتستدعي `fn` على كل جهاز. لو الـ callback رجعت non-zero، الـ iteration بتوقف وبترجع نفس القيمة.

**Parameters:**
- `class`: الـ class المراد الـ iteration عليها.
- `start`: جهاز للبدء منه، أو `NULL` للبدء من الأول.
- `data`: بيانات تتمرر لكل استدعاء للـ callback.
- `fn`: الـ callback — لازم ترجع `0` للاستمرار، أو non-zero للإيقاف.

**Return value:** `0` لو كملت على كل الأجهزة، أو أول non-zero return من الـ callback.

**Key details:**
- بتعمل `get_device()` قبل الـ callback وتـ`put_device()` بعدها.
- الـ caller لازم ميمسكش locks تمنع الـ klist traversal.

**Pseudocode flow:**
```
class_for_each_device(class, start, data, fn):
    class_dev_iter_init(&iter, class, start, NULL)
    while (dev = class_dev_iter_next(&iter)):
        error = fn(dev, data)
        if error:
            break
    class_dev_iter_exit(&iter)
    return error
```

---

#### `class_find_device`

```c
struct device *class_find_device(const struct class *class,
                                  const struct device *start,
                                  const void *data,
                                  device_match_t match);
```

**الـ typedef:** `typedef int (*device_match_t)(struct device *dev, const void *data);`

بتدور على أول جهاز في الـ class تحقق فيه الـ `match` function شرطها. بترجع الجهاز مع reference مرفوعة عليه (بيلزم `put_device()` من الـ caller).

**Parameters:**
- `class`: الـ class المراد البحث فيها.
- `start`: جهاز للبدء منه أو `NULL`.
- `data`: البيانات اللي بتتمرر للـ match function.
- `match`: callback بترجع non-zero لو الجهاز ده هو المطلوب.

**Return value:** pointer للجهاز الأول اللي match، أو `NULL` لو ما فيش. الـ caller مسؤول عن `put_device()`.

**Key details:**
- الـ returned device بيبقى معاه reference — مهم جداً تعمل `put_device()` بعد الاستخدام.
- الـ match callback بتتنادى من process context مع device lock ممكن يبقى محجوز.

---

### Group 4: Device Search Helpers

كلها inline wrappers فوق `class_find_device` بـ matching functions جاهزة. كلها بترجع `struct device *` (أو `NULL`) وبيلزم `put_device()` بعد الاستخدام.

---

#### `class_find_device_by_name`

```c
static inline struct device *class_find_device_by_name(
    const struct class *class, const char *name);
```

بتبحث عن جهاز بالاسم (مقارنة string). بتستدعي `device_match_name` كـ match function.

**Parameters:**
- `class`: الـ class المراد البحث فيها.
- `name`: الاسم المراد البحث عنه (string مقارنة مع `dev_name(dev)`).

**Return value:** pointer للجهاز أو `NULL`.

---

#### `class_find_device_by_of_node`

```c
static inline struct device *class_find_device_by_of_node(
    const struct class *class, const struct device_node *np);
```

بتبحث عن جهاز عنده `dev->of_node == np`. مفيدة جداً في الـ DT-based systems لما بتعرف الـ node بس مش الـ device.

**Parameters:**
- `class`: الـ class.
- `np`: الـ device tree node المراد matching بيه.

---

#### `class_find_device_by_fwnode`

```c
static inline struct device *class_find_device_by_fwnode(
    const struct class *class, const struct fwnode_handle *fwnode);
```

بتبحث بـ `fwnode_handle` — تغطي كل من DT وACPI firmware nodes. الـ `device_match_fwnode` بتستخدم `dev_fwnode(dev)` للمقارنة.

---

#### `class_find_device_by_devt`

```c
static inline struct device *class_find_device_by_devt(
    const struct class *class, dev_t devt);
```

بتبحث عن جهاز بـ `dev_t` (major + minor). `device_match_devt` بتقارن `dev->devt`.

**Parameters:**
- `devt`: الـ device number (`MKDEV(major, minor)`).

---

#### `class_find_device_by_acpi_dev`

```c
static inline struct device *class_find_device_by_acpi_dev(
    const struct class *class, const struct acpi_device *adev);
```

بتبحث عن جهاز مرتبط بـ ACPI companion. متاحة بس لو `CONFIG_ACPI` مفعّل — غير كده بترجع `NULL` دايماً.

---

### Group 5: Sysfs Attribute Management

الـ class بتدعم attributes بتظهر في `/sys/class/<classname>/`. الـ namespace support موجود عشان الـ classes جوه network namespaces (مثلاً `net` class).

---

#### `class_create_file_ns`

```c
int __must_check class_create_file_ns(const struct class *class,
                                       const struct class_attribute *attr,
                                       const void *ns);
```

بتضيف attribute file للـ class directory في sysfs مع namespace tag. الـ `__must_check` لأن فشل الـ sysfs creation لازم يتعالج.

**Parameters:**
- `class`: الـ class المراد إضافة الـ attribute ليها.
- `attr`: الـ `struct class_attribute` اللي بتحدد الاسم والـ show/store callbacks.
- `ns`: namespace pointer (أو `NULL` لو مش في namespace).

**Return value:** `0` أو negative error.

**Key details:**
- بتستدعي `sysfs_create_file_ns()` على الـ class kobject.
- لو الـ attribute موجودة بالفعل بترجع `-EEXIST`.

---

#### `class_remove_file_ns`

```c
void class_remove_file_ns(const struct class *class,
                           const struct class_attribute *attr,
                           const void *ns);
```

بتشيل الـ attribute file من sysfs. لازم تتستدعى بنفس الـ ns اللي اتعمل بيه الـ create.

**Parameters:**
- `class`: الـ class.
- `attr`: الـ attribute المراد حذفه.
- `ns`: نفس الـ namespace pointer اللي اتبعت في الـ create.

---

#### `class_create_file` / `class_remove_file`

```c
static inline int __must_check class_create_file(
    const struct class *class, const struct class_attribute *attr);

static inline void class_remove_file(
    const struct class *class, const struct class_attribute *attr);
```

Wrappers بسيطة بتمرر `NULL` كـ namespace. دي النسخة المستخدمة في معظم drivers.

---

#### `show_class_attr_string`

```c
ssize_t show_class_attr_string(const struct class *class,
                                const struct class_attribute *attr,
                                char *buf);
```

الـ generic `show` handler للـ `struct class_attribute_string`. بتقرأ الـ `str` المخزنة في الـ `class_attribute_string` وبتنسخها للـ `buf`.

**Parameters:**
- `class`: الـ class المالكة للـ attribute.
- `attr`: بيتعمله `container_of` للـ `class_attribute_string` الأصلية.
- `buf`: الـ sysfs page buffer (PAGE_SIZE).

**Return value:** عدد البايتات المكتوبة (نتيجة `sysfs_emit`).

**Key details:** بتُستخدم مع ماكرو `CLASS_ATTR_STRING` لتعريف static string attributes بدون كتابة show function يدوياً.

```c
/* Usage example */
static CLASS_ATTR_STRING(version, 0444, "1.0");

/* Equivalent to: */
static struct class_attribute_string class_attr_version = {
    .attr = __ATTR(version, 0444, show_class_attr_string, NULL),
    .str  = "1.0",
};
```

---

### Group 6: Class Interface

الـ `class_interface` هي آلية notification داخل الـ kernel — بتسمح لـ subsystem ثاني يعرف لما جهاز بيتضاف أو يتشال من class معينة. مختلف عن الـ uevent لأنه kernel-internal.

---

#### `class_interface_register`

```c
int __must_check class_interface_register(struct class_interface *class_intf);
```

بتسجل الـ interface في الـ class وبتستدعي `add_dev` على كل الأجهزة الموجودة حالياً في الـ class. ده يضمن إن الـ interface ما يفوتش أجهزة اتضافت قبل التسجيل.

**Parameters:**
- `class_intf`: الـ interface اللي فيها pointers للـ `add_dev` و`remove_dev` callbacks والـ `class` المستهدفة.

**Return value:** `0` أو negative error.

**Key details:**
- بتحجز الـ class mutex أثناء التسجيل.
- الـ `add_dev` callback بتتنادى من نفس context التسجيل على الأجهزة الموجودة.
- لو الـ callback على جهاز موجود فشلت، التسجيل بيكمل (الـ error بيتجاهل).

**Pseudocode flow:**
```
class_interface_register(intf):
    mutex_lock(&class->mutex)
    list_add_tail(&intf->node, &class->interfaces)
    for each dev in class->devices:
        if intf->add_dev:
            intf->add_dev(dev)
    mutex_unlock(&class->mutex)
    return 0
```

---

#### `class_interface_unregister`

```c
void class_interface_unregister(struct class_interface *class_intf);
```

عكس `class_interface_register`، بتشيل الـ interface من الـ class وبتستدعي `remove_dev` على كل الأجهزة الحالية.

**Parameters:**
- `class_intf`: الـ interface المراد إزالتها.

**Return value:** void.

**Key details:**
- بتحجز نفس الـ mutex.
- الـ `remove_dev` بتتنادى بترتيب عكسي (reverse order) للأجهزة.

---

### الـ Macros الخاصة بالـ Attributes

```c
/* Class-level attributes */
#define CLASS_ATTR_RW(_name)   /* read-write: show + store */
#define CLASS_ATTR_RO(_name)   /* read-only: show only */
#define CLASS_ATTR_WO(_name)   /* write-only: store only */

/* Static string attribute */
#define CLASS_ATTR_STRING(_name, _mode, _str)
```

كلهم بيولدوا `struct class_attribute class_attr_<name>`. الـ `CLASS_ATTR_RW` بيتوقع إن في functions اسمها `<name>_show` و`<name>_store` معرّفة.

```c
/* Typical usage */
static ssize_t my_prop_show(const struct class *class,
                             const struct class_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "value\n");
}
static ssize_t my_prop_store(const struct class *class,
                              const struct class_attribute *attr,
                              const char *buf, size_t count)
{
    /* parse buf */
    return count;
}
static CLASS_ATTR_RW(my_prop); /* generates class_attr_my_prop */
```

---

### الـ Data Structures المهمة

#### `struct class`

```c
struct class {
    const char                          *name;          /* /sys/class/<name> */
    const struct attribute_group        **class_groups; /* class-level sysfs attrs */
    const struct attribute_group        **dev_groups;   /* per-device sysfs attrs */
    int (*dev_uevent)(...);   /* uevent env vars callback */
    char *(*devnode)(...);    /* devtmpfs path/mode */
    void (*class_release)(...);  /* class cleanup */
    void (*dev_release)(...);    /* device cleanup */
    int (*shutdown_pre)(...);    /* pre-shutdown hook */
    const struct kobj_ns_type_operations *ns_type;
    const void *(*namespace)(...);
    void (*get_ownership)(...);  /* uid/gid for sysfs entries */
    const struct dev_pm_ops *pm; /* default PM ops */
};
```

#### `struct class_dev_iter`

```c
struct class_dev_iter {
    struct klist_iter       ki;   /* underlying klist iterator */
    const struct device_type *type; /* optional filter */
    struct subsys_private   *sp;  /* class's private subsystem data */
};
```

الـ iterator بيحتفظ بـ `subsys_private` pointer لأن الأجهزة متخزنة في الـ `klist` الموجودة في `sp->klist_devices`. وجود الـ `klist_iter` بيضمن إن الجهاز الحالي معاه reference مرفوعة طول الـ iteration.

#### `struct class_interface`

```c
struct class_interface {
    struct list_head    node;       /* linked into class->interfaces */
    const struct class  *class;     /* target class */
    int (*add_dev)(struct device *dev);      /* called on device add */
    void (*remove_dev)(struct device *dev);  /* called on device remove */
};
```

#### `struct class_attribute`

```c
struct class_attribute {
    struct attribute attr;    /* name + mode */
    ssize_t (*show)(const struct class *, const struct class_attribute *, char *);
    ssize_t (*store)(const struct class *, const struct class_attribute *,
                     const char *, size_t);
};
```

#### `struct class_attribute_string`

```c
struct class_attribute_string {
    struct class_attribute attr; /* embeds the attribute */
    char *str;                   /* the static string to show */
};
```

---

### Locking Notes

| Operation | Lock Used |
|---|---|
| `class_register/unregister` | الـ `subsys_mutex` في الـ driver core |
| `class_dev_iter_*` | الـ `klist` spinlock (داخلي) |
| `class_interface_register/unregister` | الـ class mutex (`sp->mutex`) |
| `class_for_each_device` | الـ klist reference counting |
| `class_find_device` | الـ klist reference counting |

الـ klist mechanism بتضمن إن الـ device لو اتحذف جوه الـ iteration، الحذف الفعلي بيتأجل لحد ما الـ reference تنزل لصفر — ده اللي بيخلي الـ iteration آمنة حتى مع concurrent unregistrations.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ **device class subsystem** مش بيستخدم debugfs بشكل مباشر، لكن الـ **kobject** و الـ **klist** اللي بيبنوا عليها موجودين بشكل غير مباشر. أهم نقاط الدخول:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# عرض كل الـ uevents اللي اتبعتت
cat /sys/kernel/debug/tracing/events/kobject/kobject_action/enable

# عرض الـ klist state للـ devices داخل class
# مفيش مدخل مباشر — بنوصله عن طريق sysfs أو crash/gdb
```

> ملحوظة: الـ `class_dev_iter` اللي بيستخدم `klist_iter` بيعمل iteration على الـ `subsys_private->klist_devices`. لو عايز تشوف حالة الـ klist في runtime، استخدم **crash utility** أو **/proc/kcore**.

```bash
# عرض subsystem private pointer لـ class معينة (مثلاً block)
# عن طريق crash tool:
# crash> p block_class->p
```

---

#### 2. مدخلات الـ sysfs

الـ **class** بتظهر في sysfs تحت `/sys/class/`. كل class ليها directory بيحتوي على:

| المسار | المحتوى |
|--------|---------|
| `/sys/class/<classname>/` | الـ devices المسجلة في الـ class |
| `/sys/class/<classname>/<dev>/uevent` | الـ uevent environment variables |
| `/sys/class/<classname>/<dev>/subsystem` | symlink للـ class |
| `/sys/class/<classname>/<dev>/power/` | الـ PM state |
| `/sys/class/<classname>/<dev>/dev` | الـ major:minor number |

```bash
# عرض كل الـ classes المسجلة
ls /sys/class/

# عرض كل الـ devices داخل class معينة (مثلاً net)
ls /sys/class/net/

# قراءة uevent لـ device معينة
cat /sys/class/net/eth0/uevent
# Output example:
# DEVTYPE=net
# INTERFACE=eth0
# IFINDEX=2

# عرض الـ class attributes لـ device
ls /sys/class/block/sda/

# التحقق من الـ kobject path كاملاً
readlink -f /sys/class/net/eth0
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# تفعيل كل events الـ kobject (اللي بتشمل class add/remove)
echo 1 > /sys/kernel/debug/tracing/events/kobject/enable

# تفعيل event معين: kobject_add
echo 1 > /sys/kernel/debug/tracing/events/kobject/kobject_add/enable

# تتبع الـ uevent actions
echo 1 > /sys/kernel/debug/tracing/events/kobject/kobject_action/enable

# قراءة الـ trace buffer
cat /sys/kernel/debug/tracing/trace

# مثال على output:
# kworker/0:1-123  [000] .... kobject_add: name=eth0, parent=net
# udevd-456       [001] .... kobject_action: action=add, name=eth0

# تتبع function calls مباشرة لـ class_register
echo 'class_register' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on

# تتبع class_dev_iter_next و class_for_each_device
echo 'class_dev_iter_next class_for_each_device class_find_device' \
  > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. الـ printk و Dynamic Debug

```bash
# تفعيل dynamic debug لـ drivers/base/class.c
echo 'file drivers/base/class.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل debug messages في drivers/base/
echo 'file drivers/base/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل dynamic debug لـ kobject
echo 'file lib/kobject.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file lib/kobject_uevent.c +p' > /sys/kernel/debug/dynamic_debug/control

# عرض الـ dynamic debug flags الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep 'class\|kobject'

# رفع مستوى printk مؤقتاً (بيأثر على dmesg)
echo 8 > /proc/sys/kernel/printk

# قراءة الـ kernel log مع timestamps
dmesg -T | grep -i 'class\|kobject\|uevent'

# متابعة live
dmesg -wT | grep -i 'class\|device'
```

---

#### 5. كونفيجريشن الـ Kernel الخاصة بالـ Debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_DEBUG_KOBJECT` | طباعة رسائل debug لكل kobject_add/del/get/put |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | تأخير الـ release لاكتشاف use-after-free |
| `CONFIG_SYSFS` | لازم يكون enabled عشان تشتغل class |
| `CONFIG_UEVENT_HELPER` | بيسمح بـ userspace uevent helper script |
| `CONFIG_DEBUG_DEVRES` | debug الـ managed device resources |
| `CONFIG_KASAN` | اكتشاف heap use-after-free في kobject release |
| `CONFIG_LOCKDEP` | اكتشاف deadlocks في klist spinlock |
| `CONFIG_PROVE_LOCKING` | تحقق من locking correctness |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug messages |
| `CONFIG_TRACING` | لازم يكون enabled لـ ftrace |

```bash
# التحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_DEBUG_KOBJECT|CONFIG_SYSFS|CONFIG_UEVENT'
# أو
grep -E 'CONFIG_DEBUG_KOBJECT|CONFIG_SYSFS|CONFIG_UEVENT' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ udevadm** — الأداة الرئيسية لمراقبة class events:

```bash
# مراقبة كل uevents في real-time
udevadm monitor --udev --kernel

# مثال على output:
# KERNEL[1234.567] add    /class/net/eth0 (net)
# UDEV  [1234.890] add    /class/net/eth0 (net)

# عرض معلومات device كاملة بما فيها class
udevadm info --query=all --name=/dev/sda

# إعادة trigger uevent لـ device معينة
udevadm trigger --action=change /sys/class/net/eth0

# test rules بدون تطبيق
udevadm test /sys/class/block/sda

# عرض الـ class_compat links (للـ devices القديمة)
ls -la /sys/class/scsi_host/

# التحقق من الـ class interface المسجلة
# مافيش tool مباشر — لازم تقرأ /proc/kcore أو تستخدم crash
```

**الـ sysfs verification:**

```bash
# التحقق من إن class_register شتغل صح
ls /sys/class/<your_class_name>

# التحقق من الـ class attributes
ls /sys/class/<your_class_name>/
# لو في class_groups هيظهر attributes

# عرض namespace info
cat /sys/class/net/<dev>/ifindex  # net namespace specific
```

---

#### 7. رسائل الخطأ الشائعة — الأسباب والحلول

| رسالة الخطأ | السبب | الحل |
|-------------|-------|------|
| `class_register failed for <name>: -EEXIST` | class مسجلة مسبقاً بنفس الاسم | تأكد إن `class_unregister` اتنادى قبل كل re-register؛ ابحث عن duplicate registration |
| `kobject: '<name>': kobject_add_internal failed` | parent kobject مش موجود أو اسم مكرر | تحقق من الـ parent lifetime وتأكد من uniqueness الاسم |
| `device_create failed` | الـ class مش مسجلة أو الـ devt مكررة | تأكد إن `class_register` اتنادى أولاً بنجاح |
| `WARNING: CPU: X PID: Y at drivers/base/core.c:... kobject_put` | استخدام kobject بعد ما اتحرر (use-after-free) | فعّل `CONFIG_DEBUG_KOBJECT_RELEASE` و KASAN |
| `sysfs: cannot create duplicate filename '/class/<name>'` | اسم class أو device مكرر في sysfs | تحقق من الـ dev_name() وتأكد من uniqueness |
| `BUG: spinlock bad magic` | الـ klist spinlock اتخرب | heap corruption؛ فعّل KASAN وKMEMLEAK |
| `device: '<name>': device_add: parent '<parent>' is not online` | الـ parent device مش online | انتظر parent device يكون ready قبل add |
| `class_interface_register failed` | الـ class مش initialized | تأكد من ترتيب initialization |
| `kernel BUG at lib/klist.c` | iterator استخدم بعد `klist_iter_exit` | تحقق من iter lifetime في code |
| `Unable to handle kernel NULL pointer dereference` في `class_for_each_device` | الـ `fn` callback بيرجع negative value أو الـ `data` pointer غلط | تحقق من الـ callback return values والـ data pointers |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في class_register — تحقق من صحة الـ class pointer */
int class_register(const struct class *class)
{
    /* WARN لو class name فاضية */
    if (WARN_ON(!class->name))
        return -EINVAL;

    /* WARN لو class اتسجلت مرتين */
    if (WARN_ON(class_is_registered(class))) {
        dump_stack();
        return -EEXIST;
    }
    /* ... */
}

/* في dev_release callback — تحقق من refcount */
void my_dev_release(struct device *dev)
{
    /* WARN لو بنحرر device والـ kobject kref مش zero */
    WARN_ON(kref_read(&dev->kobj.kref) != 0);
    /* ... */
}

/* في class_dev_iter — guard ضد NULL */
struct device *class_dev_iter_next(struct class_dev_iter *iter)
{
    /* WARN لو iter اتعمله exit بالفعل */
    WARN_ON(!iter->sp);
    /* ... */
}

/* في class_for_each_device callback */
static int my_callback(struct device *dev, void *data)
{
    /* WARN لو device في state غلط */
    WARN_ON(!device_is_registered(dev));
    return 0;
}
```

**أماكن استراتيجية تانية:**
- قبل وبعد `class_register` / `class_unregister` في الـ module init/exit
- في الـ `class_release` callback لو الـ class بتتحرر قبل ما كل devices تتسجل خروجها
- في `add_dev` / `remove_dev` callbacks في `class_interface_register`

---

### Hardware Level

---

#### 1. التحقق من إن الـ hardware state بيطابق الـ kernel state

الـ `struct class` هي abstraction بحتة — مفيش hardware registers مرتبطة بيها مباشرة. لكن الـ devices اللي بتنتمي للـ class بتتعامل مع hardware. طريقة التحقق:

```bash
# التحقق من إن الـ device اتشافها الـ kernel صح
lspci -v | grep -A 10 "your_device"
lsusb -v | grep -A 10 "your_device"

# مقارنة الـ class devices بما هو موجود فعلاً
ls /sys/class/net/           # network interfaces
ls /sys/class/block/         # block devices
ls /sys/class/input/         # input devices

# التحقق من إن كل device ليها driver صح
for dev in /sys/class/net/*/; do
    echo "=== $(basename $dev) ==="
    cat "$dev/uevent" 2>/dev/null
    readlink "$dev/device/driver" 2>/dev/null
done

# مقارنة عدد الـ devices المكتشفة مع المتوقعة
cat /proc/net/dev | wc -l   # عدد network interfaces
```

---

#### 2. تقنيات الـ Register Dump

الـ class subsystem بيتكلم مع hardware بشكل غير مباشر عن طريق الـ driver. لـ register dump مباشر:

```bash
# استخدام devmem2 لقراءة physical memory (للـ MMIO registers)
# مثال: قراءة 4 bytes من عنوان 0xFEDC0000
devmem2 0xFEDC0000 w

# لو devmem2 مش متاح، استخدم /dev/mem
# (بيحتاج CONFIG_DEVMEM=y و kernel.devmem_restricted=0)
dd if=/dev/mem bs=4 count=1 skip=$((0xFEDC0000 / 4)) 2>/dev/null | xxd

# أو عن طريق Python:
python3 -c "
import mmap, os
fd = os.open('/dev/mem', os.O_RDONLY)
m = mmap.mmap(fd, 4096, offset=0xFEDC0000)
print(hex(int.from_bytes(m.read(4), 'little')))
m.close()
os.close(fd)
"

# استخدام io utility (من package iotools)
io -4 -r 0xFEDC0000

# للـ PCI devices: قراءة الـ config space
lspci -xxx -s 00:1f.0   # dump hex للـ PCI config

# للـ platform devices: البحث عن الـ MMIO range
cat /proc/iomem | grep -i "your_device"
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

للـ class-level debugging، الـ hardware signals المهمة:

**I2C / SMBus (مثلاً class = i2c-adapter):**
```
- Probe SDA (data) + SCL (clock) على نفس الوقت
- ابحث عن: START condition → Address frame → ACK/NACK
- NACK = device مش موجودة أو عنوانها غلط
- Clock stretching = device slow response
- Trigger على: SDA falling edge أثناء SCL high (START)
```

**USB (class = usb_device):**
```
- استخدم USB protocol analyzer (Beagle, Total Phase)
- ابحث عن: SETUP packets للـ descriptor requests
- Class descriptor (bDescriptorType = 0x21 للـ HID مثلاً)
- Trigger على: SOF packet أو SETUP token
```

**PCIe:**
```
- Protocol analyzer ضروري (Lecroy, Teledyne)
- Config Read/Write transactions لـ BAR setup
- Completion timeouts = device مش بترد
```

**GPIO/IRQ للـ interrupt-driven class devices:**
```
- Probe الـ IRQ line
- قيّس latency من interrupt إلى kernel handler
- ابحث عن spurious interrupts (edge بدون cause)
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | نمط kernel log | التفسير |
|---------|---------------|---------|
| Device مش بترد على الـ bus | `device_attach: ... failed` + `driver_probe_device: ... -ENODEV` | الـ driver مش لاقي device على الـ bus |
| Power issue عند الـ probe | `pm_runtime_get: ... -EAGAIN` | الـ device مش صحيت من power-down |
| DMA failure | `dma_alloc_coherent failed` | الـ device مش بتدعم الـ DMA range |
| Interrupt مش وصل | `irq X: nobody cared` + stacktrace | IRQ routing غلط في الـ hardware |
| Class device اتضافت وانمسحت سريع | `kobject_add ... kobject_del` في نفس الـ dmesg sequence | hardware instability أو power issue |
| Namespace conflict | `sysfs: cannot create duplicate filename` | نفس الـ device اتعرفت مرتين (driver conflict) |

```bash
# فلترة kernel log للـ class/device errors
dmesg -T | grep -E 'WARN|BUG|class|kobject|uevent|probe|device_add'

# عرض الـ errors فقط
dmesg -T --level=err,crit,alert,emerg
```

---

#### 5. Device Tree Debugging

الـ class subsystem بيستخدم `class_find_device_by_of_node()` للربط مع الـ DT nodes:

```bash
# التحقق من إن الـ DT compiled صح
dtc -I fs /sys/firmware/devicetree/base 2>&1 | head -20

# عرض كل الـ DT nodes
find /sys/firmware/devicetree/base -name "compatible" | \
  while read f; do echo "$(dirname $f): $(cat $f)"; done

# التحقق من إن node معين موجود في الـ DT
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "your,device" {} \;

# مقارنة الـ of_node بالـ class device
ls /sys/class/net/eth0/device/of_node
readlink /sys/class/net/eth0/device/of_node

# التحقق من إن الـ driver اتعرف على الـ compatible string
cat /sys/bus/platform/drivers/your_driver/uevent

# عرض الـ DT node كاملاً
dtc -I fs /sys/firmware/devicetree/base/your_node 2>/dev/null

# تحقق من الـ clocks/resets/interrupts في الـ DT node
cat /sys/firmware/devicetree/base/soc/your_device/clocks 2>/dev/null | xxd
cat /sys/firmware/devicetree/base/soc/your_device/interrupts 2>/dev/null | xxd

# مقارنة ما في الـ DT مع ما اكتشفه الـ kernel فعلاً
cat /proc/device-tree/soc/your_device/status 2>/dev/null
# يجب أن يكون "okay"
```

**تحقق من الـ fwnode matching (المستخدمة في `class_find_device_by_fwnode()`):**
```bash
# عرض الـ fwnode path لـ device
udevadm info /sys/class/net/eth0 | grep DEVPATH
# ثم ابحث عن corresponding DT node

# التحقق من ACPI device matching (class_find_device_by_acpi_dev)
cat /sys/class/net/eth0/device/firmware_node/path 2>/dev/null
# أو
cat /sys/class/net/eth0/device/firmware_node/description 2>/dev/null
```

---

### Practical Commands

---

#### ملخص الأوامر الجاهزة للنسخ

**أولاً: فحص سريع لحالة الـ class system:**

```bash
#!/bin/bash
# class-health-check.sh — فحص شامل سريع لحالة device classes

echo "=== Registered Classes ==="
ls /sys/class/ | sort

echo ""
echo "=== Class Device Counts ==="
for cls in /sys/class/*/; do
    count=$(ls "$cls" 2>/dev/null | wc -l)
    printf "%-30s : %d devices\n" "$(basename $cls)" "$count"
done

echo ""
echo "=== Recent Class Events (dmesg) ==="
dmesg -T | grep -iE 'class_register|class_unregister|device_add|device_del' | tail -20

echo ""
echo "=== Uevent Sequence Number ==="
cat /sys/kernel/uevent_seqnum
```

**ثانياً: تفعيل الـ tracing لـ class debugging:**

```bash
#!/bin/bash
# enable-class-trace.sh

TRACE=/sys/kernel/debug/tracing

# cleanup أولاً
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# تفعيل kobject events
echo 1 > $TRACE/events/kobject/enable

# ضبط الـ tracer
echo function_graph > $TRACE/current_tracer
echo 'class_register class_unregister class_dev_iter_next class_for_each_device class_find_device' \
  > $TRACE/set_ftrace_filter

# تفعيل الـ tracing
echo 1 > $TRACE/tracing_on

echo "Tracing enabled. Run your test, then:"
echo "  cat $TRACE/trace"
echo "  echo 0 > $TRACE/tracing_on"
```

**ثالثاً: مراقبة الـ uevents:**

```bash
# مراقبة uevents مع filtering
udevadm monitor --kernel --property 2>&1 | grep -v 'SEQNUM'

# تسجيل كل uevents لمدة 30 ثانية
timeout 30 udevadm monitor --kernel --property > /tmp/uevents.log 2>&1
echo "Captured $(wc -l < /tmp/uevents.log) lines"

# تحليل الـ uevents المسجلة
grep 'ACTION\|SUBSYSTEM\|DEVNAME' /tmp/uevents.log | head -60
```

**رابعاً: debug الـ class iterator:**

```bash
# عرض كل devices في class معينة مع معلوماتها
CLASS="net"
echo "=== Devices in class: $CLASS ==="
for dev in /sys/class/$CLASS/*/; do
    devname=$(basename "$dev")
    driver=$(readlink "$dev/device/driver" 2>/dev/null | xargs basename 2>/dev/null || echo "no driver")
    uevent=$(cat "$dev/uevent" 2>/dev/null | tr '\n' ' ')
    printf "Device: %-15s Driver: %-20s Uevent: %s\n" "$devname" "$driver" "$uevent"
done
```

**خامساً: التحقق من use-after-free في الـ kobject:**

```bash
# تفعيل KASAN + DEBUG_KOBJECT_RELEASE في kernel config:
# CONFIG_KASAN=y
# CONFIG_DEBUG_KOBJECT_RELEASE=y
# CONFIG_DEBUG_KOBJECT=y

# ثم rebuild الـ kernel وشاهد dmesg:
dmesg -T | grep -E 'KASAN|use-after-free|kobject'

# تفعيل slub debug لاكتشاف heap corruption
echo 1 > /sys/kernel/slab/kmalloc-512/sanity_checks 2>/dev/null
```

**سادساً: تحقق من الـ class_compat links:**

```bash
# الـ class_compat بتُستخدم للـ backward compatibility مع الـ old sysfs layout
# تحقق من وجودها
ls /sys/class/scsi_host/ | head -5
ls -la /sys/class/scsi_host/host0/

# لو عايز تتحقق من class_compat_create_link نجح
# دور على symlinks في الـ class directory
find /sys/class/ -type l -name "device" | head -10
```

---

#### مثال على تفسير الـ Output

**مثال: dmesg بعد تفعيل `CONFIG_DEBUG_KOBJECT=y`:**

```
[  2.345678] kobject: 'eth0' (ffff888012345678): kobject_add_internal: parent: 'net', set: 'net'
[  2.345890] kobject: 'eth0' (ffff888012345678): kobject_uevent_env: action=add
[  2.346100] kobject: 'eth0' (ffff888012345678): fill_kobj_path: entry.name = eth0
```

**تفسير:**
- `ffff888012345678` = عنوان الـ kobject في الـ kernel memory — مفيد للـ crash debugging
- `parent: 'net'` = الـ class هي `net`
- `set: 'net'` = الـ kset اللي بيحتوي الـ kobject
- `action=add` = الـ uevent اتبعت بـ action=add

**مثال: ftrace output لـ class_for_each_device:**

```
 0)               |  class_for_each_device() {
 0)   0.234 us    |    class_dev_iter_init();
 0)               |    class_dev_iter_next() {
 0)   0.123 us    |      klist_next();
 0)   0.089 us    |    }  /* class_dev_iter_next */
 0)   1.456 us    |    fn();                    /* your callback */
 0)               |    class_dev_iter_next() {
 0)   0.098 us    |      klist_next();           /* returns NULL = no more devices */
 0)   0.067 us    |    }
 0)   0.145 us    |    class_dev_iter_exit();
 0) + 3.456 us    |  }  /* class_for_each_device */
```

**تفسير:**
- الـ iteration على 1 device استغرقت ~3.5 µs — طبيعي
- لو الـ `fn()` استغرقت وقت طويل = مشكلة في الـ callback مش في الـ class iterator
- لو `klist_next` اتنادت كتير = في devices كتير في الـ class
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — `class_find_device_by_of_node` بترجع NULL

#### العنوان
**الـ custom industrial gateway بيعمل probe للـ UART driver بس مش لاقي الـ device في الـ class**

#### السياق
بتشتغل على industrial gateway بيستخدم **RK3562** SoC، الـ product بيتحكم في 4 UART ports لتوصيل sensors. الـ driver بتاعك بيحاول يجيب الـ tty device المرتبط بـ DT node معين عشان يعمل له configuration خاصة.

#### المشكلة
الـ driver بيعمل call لـ `class_find_device_by_of_node` وبترجعله `NULL` رغم إن الـ UART port شغال ظاهريًا في `/dev/ttyS1`.

```c
struct device *uart_dev;
uart_dev = class_find_device_by_of_node(&tty_class, uart_np);
if (!uart_dev) {
    dev_err(dev, "can't find tty device for uart node\n");
    return -ENODEV;  /* driver بيفشل هنا دايمًا */
}
```

#### التحليل
الـ `class_find_device_by_of_node` معرفة في `class.h` كـ static inline:

```c
static inline struct device *class_find_device_by_of_node(const struct class *class,
                                                          const struct device_node *np)
{
    return class_find_device(class, NULL, np, device_match_of_node);
}
```

الـ `class_find_device` بتعمل iterate على كل الـ devices المسجلة في الـ class عن طريق `class_dev_iter`:

```c
struct class_dev_iter {
    struct klist_iter        ki;      /* iterator على الـ klist الداخلية */
    const struct device_type *type;   /* فلتر اختياري بالنوع */
    struct subsys_private    *sp;     /* private data للـ subsystem */
};
```

المشكلة الحقيقية: الـ tty device اتسجل قبل ما الـ of_node يتربط بيه. الـ `device_match_of_node` بتتحقق من `dev->of_node`، لو الـ serial driver استخدم `platform_device_alloc` وما عملش `device_set_of_node_from_dev` الصح، الـ `of_node` بيبقى NULL.

```bash
# تأكد من الـ of_node
ls -la /sys/class/tty/ttyS1/
cat /sys/class/tty/ttyS1/uevent
# لو مفيش of_node يبقى المشكلة في الـ device creation
```

#### الحل
```bash
# خطوة 1: افحص الـ of_node
ls /sys/class/tty/ttyS1/of_node 2>/dev/null && echo "of_node موجود" || echo "of_node مش موجود"

# خطوة 2: تتبع الـ probe
echo 'p serial8250_register_8250_port' >> /sys/kernel/debug/tracing/uprobe_events
```

في الـ driver code، استخدم `class_find_device_by_name` كـ fallback:

```c
/* fallback: ابحث بالاسم لو of_node مش متربط */
uart_dev = class_find_device_by_of_node(&tty_class, uart_np);
if (!uart_dev) {
    /* try by name as fallback */
    uart_dev = class_find_device_by_name(&tty_class, "ttyS1");
}
```

أو الحل الصح: تأكد إن الـ platform device بيورث الـ of_node من parent:

```c
/* في الـ serial driver */
platform_set_drvdata(pdev, port);
/* الـ of_node بيتسجل تلقائيًا من pdev->dev.of_node */
```

#### الدرس المستفاد
الـ `class_find_device_by_of_node` بتشتغل بس لو الـ device اتسجل وعنده `dev->of_node` مضبوط. لازم تفحص الـ device creation path وتتأكد إن الـ of_node بيتربط بالـ device قبل ما تعمل find.

---

### السيناريو 2: Android TV Box على Allwinner H616 — `dev_uevent` callback بيعمل crash

#### العنوان
**الـ HDMI class بيعمل kernel panic وقت plug/unplug الـ monitor**

#### السياق
Android TV box بيستخدم **Allwinner H616**، عندك custom HDMI class بتضيفه للـ display subsystem. كل ما user بيوصل/يفصل monitor، الـ kernel بيعمل panic.

#### المشكلة
```
BUG: kernel NULL pointer dereference at 0000000000000018
Call Trace:
  drm_hdmi_dev_uevent+0x2c/0x60
  dev_uevent+0x1a4/0x2e0
  kobject_uevent_env+0x3b8/0x5a0
```

#### التحليل
الـ `struct class` عنده callback:

```c
struct class {
    const char  *name;
    /* ... */
    int (*dev_uevent)(const struct device *dev, struct kobj_uevent_env *env);
    /* ... */
};
```

الـ `dev_uevent` callback بيتكال من `dev_uevent()` في `drivers/base/core.c` كل ما في add/remove event. المشكلة في الـ custom implementation:

```c
/* الكود الغلط */
static int drm_hdmi_dev_uevent(const struct device *dev,
                                struct kobj_uevent_env *env)
{
    struct drm_connector *conn = dev_get_drvdata(dev);
    /* conn ممكن يبقى NULL لو الـ device لسه بتتسجل */
    add_uevent_var(env, "HDMI_STATUS=%s",
                   conn->status == connector_status_connected ?  /* CRASH هنا */
                   "connected" : "disconnected");
    return 0;
}
```

المشكلة: الـ `dev_uevent` بيتكال أثناء `class_register` نفسها (uevent للـ add event) قبل ما `drvdata` يتضبط.

#### الحل
```c
/* الكود الصح: دايمًا تحقق من NULL */
static int drm_hdmi_dev_uevent(const struct device *dev,
                                struct kobj_uevent_env *env)
{
    struct drm_connector *conn = dev_get_drvdata(dev);

    if (!conn)  /* guard ضروري */
        return 0;

    add_uevent_var(env, "HDMI_STATUS=%s",
                   conn->status == connector_status_connected ?
                   "connected" : "disconnected");

    add_uevent_var(env, "HDMI_CONNECTOR_ID=%d", conn->base.id);
    return 0;
}

static const struct class hdmi_class = {
    .name       = "hdmi",
    .dev_uevent = drm_hdmi_dev_uevent,
    .dev_release = hdmi_dev_release,  /* لازم تعرفه */
};
```

```bash
# debug الـ uevents
udevadm monitor --kernel --property
# شوف الـ events وقت plug/unplug
```

#### الدرس المستفاد
الـ `dev_uevent` callback في `struct class` بيتكال في وقت مبكر جدًا. لازم دايمًا تحمي الـ drvdata access بـ NULL check، ولازم تعرف `dev_release` callback عشان الـ kernel محتاجه.

---

### السيناريو 3: IoT Sensor Hub على STM32MP1 — `class_for_each_device` بيعمل deadlock

#### العنوان
**الـ I2C sensor manager بيتجمد وقت الـ system suspend على STM32MP1**

#### السياق
IoT device بيجمع بيانات من 8 I2C sensors مختلفة على **STM32MP157**. عندك custom `sensor_class` وبتعمل `class_for_each_device` في الـ suspend path عشان تحفظ state كل sensor.

#### المشكلة
الـ system بيتجمد تمامًا وقت محاولة الـ suspend. `sysrq+t` بيظهر:

```
[  245.123456] INFO: task kworker/u4:2:89 blocked for more than 120 seconds.
[  245.123457]       Waiting for mutex: 0xc1a2b3c4 (class_mutex)
```

#### التحليل
الـ `class_for_each_device` بتأخد lock على الـ class أثناء الـ iteration:

```c
/* من drivers/base/class.c — المنطق الداخلي */
int class_for_each_device(const struct class *class,
                           const struct device *start,
                           void *data, device_iter_t fn)
{
    /* بتأخد klist lock أثناء الـ iteration */
    class_dev_iter_init(&iter, class, start, NULL);
    while ((dev = class_dev_iter_next(&iter))) {
        error = fn(dev, data);  /* بيكال الـ callback وهو شايل lock */
        if (error)
            break;
    }
    class_dev_iter_exit(&iter);
}
```

الـ `class_dev_iter` بتستخدم `klist_iter` اللي بتشيل reference على الـ node. المشكلة: الـ suspend callback بتاعتك بتحاول تعمل `device_lock` على كل device، والـ PM core نفسه عامل `device_lock` وبيحاول يأخد الـ class lock:

```
Thread A (suspend):        Thread B (class_for_each_device):
device_lock(dev_A)  ✓     class_lock ✓
  try class_lock    ✗←——→   try device_lock(dev_A) ✗
```

**Deadlock كلاسيكي.**

#### الحل
```c
/* الكود الغلط — لا تعمل device_lock جوه class_for_each_device */
static int save_sensor_state(struct device *dev, void *data)
{
    device_lock(dev);  /* DEADLOCK! */
    /* ... */
    device_unlock(dev);
    return 0;
}

/* الكود الصح — استخدم الـ class pm ops بدل manual iteration */
static const struct class sensor_class = {
    .name = "sensor",
    .pm   = &sensor_class_pm_ops,  /* استخدم pm_ops بدل class_for_each_device */
};

/* أو لو لازم iteration، استخدم trylock */
static int save_sensor_state(struct device *dev, void *data)
{
    if (!device_trylock(dev))
        return -EBUSY;  /* skip واكمل */
    /* ... */
    device_unlock(dev);
    return 0;
}
```

```bash
# اكتشف الـ deadlock
cat /proc/lockdep_stats
echo l > /proc/sysrq-trigger  # يطبع lockdep chains
```

#### الدرس المستفاد
الـ `class_for_each_device` بتشيل lock على الـ klist أثناء الـ iteration. أي محاولة لأخد lock تاني (device lock, mutex) جوه الـ callback ممكن تعمل deadlock. الحل الأسلم هو استخدام `pm` field في `struct class` بدل manual iteration في الـ suspend path.

---

### السيناريو 4: Automotive ECU على i.MX8 — `class_interface` لـ SPI بيتسجل مرتين

#### العنوان
**الـ SPI flash driver على i.MX8QM بيعمل kernel warning وبيسجل نفسه مرتين**

#### السياق
Automotive ECU بيستخدم **i.MX8QM** لتحديثات OTA، عندك custom SPI NOR flash class interface بتعمله register عشان تـ hook على كل SPI device بيتضاف.

#### المشكلة
```
WARNING: CPU: 2 PID: 156 at drivers/base/class.c:493
spi_nor_ota_interface_register+0x48/0xb0 [spi_nor_ota]
```

الـ `add_dev` callback بيتكال مرتين لنفس الـ device.

#### التحليل
الـ `struct class_interface` معرفة كالآتي:

```c
struct class_interface {
    struct list_head    node;    /* node في قايمة interfaces الـ class */
    const struct class  *class;

    int  (*add_dev)  (struct device *dev);    /* بيتكال لكل device موجودة + كل جديدة */
    void (*remove_dev)(struct device *dev);
};
```

المهم: لما بتعمل `class_interface_register`، الـ kernel بيكال `add_dev` على كل الـ devices المسجلة حاليًا في الـ class تلقائيًا. لو module بتاعك اتعمله unload وreload، أو لو في init sequence خاطئ:

```c
/* الكود الغلط */
static int __init spi_nor_ota_init(void)
{
    /* المشكلة: بيتسجل مرتين في بعض الحالات */
    class_interface_register(&spi_nor_ota_interface);
    /* لو في error path بيعمل register تاني */
    if (setup_ota_service() < 0)
        class_interface_register(&spi_nor_ota_interface);  /* مرتين! */
    return 0;
}
```

الـ `class_interface_register` ما فيهاش check إن الـ interface اتسجل قبل كده، الـ `node` بيتضاف للـ list مرتين.

#### الحل
```c
/* استخدم flag لتجنب double registration */
static bool ota_interface_registered = false;

static int __init spi_nor_ota_init(void)
{
    int ret;

    if (ota_interface_registered)
        return 0;

    ret = class_interface_register(&spi_nor_ota_interface);
    if (ret)
        return ret;

    ota_interface_registered = true;

    ret = setup_ota_service();
    if (ret) {
        class_interface_unregister(&spi_nor_ota_interface);
        ota_interface_registered = false;
        return ret;
    }

    return 0;
}

/* الـ add_dev لازم تكون idempotent */
static int spi_nor_ota_add_dev(struct device *dev)
{
    /* تحقق إن الـ device مش مسجلة خلاص */
    if (dev_get_drvdata(dev) != NULL)
        return 0;  /* skip لو عندها drvdata بالفعل */

    /* ... باقي الـ setup */
    return 0;
}
```

```bash
# افحص الـ interfaces المسجلة
cat /sys/kernel/debug/devices_deferred 2>/dev/null
ls /sys/class/spi_master/
```

#### الدرس المستفاد
`class_interface_register` بتكال `add_dev` تلقائيًا على كل الـ devices الموجودة حاليًا وقت التسجيل. لازم الـ `add_dev` callback تكون idempotent وتتحقق من الـ state قبل أي عملية.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `devnode` callback بيدي permissions غلط لـ USB device

#### العنوان
**الـ USB gadget على AM62x مش بيتوصل من الـ userspace بدون root**

#### السياق
بتعمل bring-up لـ custom board بيستخدم **TI AM625** SoC لـ industrial HMI. الـ board عندها USB gadget interface، والـ userspace application المكتوبة بـ libusb محتاجة توصل للـ device بدون root permissions.

#### المشكلة
الـ device بتظهر في `/dev/bus/usb/001/002` بس الـ permissions كانت `crw-r--r--` فقط root يقدر يكتب عليها. الـ udev rules مش شغالة.

#### التحليل
الـ `struct class` عندها callback اسمه `devnode`:

```c
struct class {
    const char  *name;
    /* ... */
    char *(*devnode)(const struct device *dev, umode_t *mode);
    /* ... */
};
```

الـ `devnode` callback بيتحكم في:
1. **المسار** — فين الـ device بتظهر في `/dev`
2. **الـ mode** — الـ permissions بتاعتها عن طريق `umode_t *mode`

بالإضافة لده، الـ `struct class` عندها:

```c
void (*get_ownership)(const struct device *dev, kuid_t *uid, kgid_t *gid);
```

اللي بتتحكم في الـ uid/gid. المشكلة: الـ USB class الافتراضية بتدي mode = `0644` بس الـ write access بتحتاج `0664` أو udev rule. الـ udev rules مش شغالة لأن `dev_uevent` callback ما بتبعتش الـ properties الصح:

```c
/* افحص الـ uevent properties */
udevadm info -a -n /dev/bus/usb/001/002
udevadm test /sys/bus/usb/devices/1-1
```

الـ `dev_uevent` في الـ USB class ما بتبعتش `ID_VENDOR` و`ID_MODEL` بالشكل الصح فالـ udev rules مش بتـ match.

#### الحل

**الحل 1: تعديل الـ devnode callback (kernel space)**

```c
/* custom class للـ USB gadget */
static char *usb_gadget_devnode(const struct device *dev, umode_t *mode)
{
    if (mode) {
        /* اسمح للـ group بالـ read/write */
        *mode = 0664;
    }
    return NULL;  /* استخدم الاسم الافتراضي */
}

static void usb_gadget_get_ownership(const struct device *dev,
                                      kuid_t *uid, kgid_t *gid)
{
    /* plugdev group = gid 46 على Ubuntu/Debian */
    *gid = KGIDT_INIT(46);
}

static const struct class usb_gadget_class = {
    .name          = "usb_gadget",
    .devnode       = usb_gadget_devnode,
    .get_ownership = usb_gadget_get_ownership,
};
```

**الحل 2: udev rule (userspace) — الأبسط والأفضل**

```bash
# /etc/udev/rules.d/99-usb-gadget.rules
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d6b", ATTRS{idProduct}=="0104", \
    MODE="0664", GROUP="plugdev"
```

```bash
# تطبيق الـ rule بدون reboot
udevadm control --reload-rules
udevadm trigger --subsystem-match=usb
ls -la /dev/bus/usb/001/002
```

**الحل 3: تأكد من الـ uevent properties**

```bash
# افحص إيه الـ properties المتبعتة
udevadm monitor --kernel --property &
# وصّل الـ USB device
# هتشوف SUBSYSTEM, DEVTYPE, etc.
```

```c
/* في الـ dev_uevent callback تأكد من بعت كل الـ properties */
static int usb_gadget_uevent(const struct device *dev,
                              struct kobj_uevent_env *env)
{
    add_uevent_var(env, "DEVTYPE=usb_device");
    add_uevent_var(env, "ID_VENDOR=MyCompany");
    add_uevent_var(env, "ID_MODEL=HMI_Gadget");
    return 0;
}
```

#### الدرس المستفاد
الـ `devnode` و`get_ownership` callbacks في `struct class` هما المسؤولان عن الـ permissions الأولية للـ device file. لكن الـ best practice هو الاعتماد على udev rules في الـ userspace بدل تغيير الـ kernel class، لأن الـ udev أكثر مرونة ومش بتحتاج recompile الـ kernel. الـ `dev_uevent` callback لازم تبعت كل الـ properties اللي الـ udev rules محتاجاها عشان تـ match صح.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المسار | الوصف |
|--------|-------|
| `Documentation/driver-api/driver-model/class.rst` | الدليل الرسمي لنظام الـ device class |
| `Documentation/driver-api/driver-model/overview.rst` | نظرة عامة على الـ Linux Device Model |
| `Documentation/driver-api/driver-model/porting.rst` | دليل نقل الـ drivers للـ driver model الجديد |
| `Documentation/driver-api/driver-model/driver.rst` | توثيق الـ driver model وربطه بالـ class |
| `Documentation/driver-api/infrastructure.html` | البنية التحتية الكاملة لـ device drivers |

**الـ online versions:**
- [Device Classes — The Linux Kernel documentation](https://www.kernel.org/doc/html/v5.4/driver-api/driver-model/class.html)
- [Driver Model — The Linux Kernel documentation](https://docs.kernel.org/driver-api/driver-model/index.html)
- [The Linux Kernel Device Model Overview](https://docs.kernel.org/driver-api/driver-model/overview.html)
- [Device drivers infrastructure — kernel.org](https://www.kernel.org/doc/html/latest/driver-api/infrastructure.html)

---

### مقالات LWN.net

دي أهم المقالات اللي بتشرح تطور الـ device class في الـ kernel:

#### الأساسيات والمقدمة

- [Driver porting: Device model overview](https://lwn.net/Articles/31185/)
  نظرة تأسيسية على الـ device model في kernel 2.6، من سلسلة "porting drivers to 2.6".

- [Driver porting: Device classes](https://lwn.net/Articles/31370/)
  شرح مباشر لآلية الـ `struct class` وكيفية ربط الـ devices بالـ classes — أهم مقال للموضوع ده.

- [Driver porting: Devices and attributes](https://lwn.net/Articles/31220/)
  بيكمّل الصورة بشرح الـ `sysfs attributes` على مستوى الـ class والـ device.

#### التطور والتغييرات

- [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/)
  مراجعة شاملة للـ device model بعد سنوات من النضج — بيغطي إزاي الـ `struct class` اتبسطت.

- [Device model changes in store](https://lwn.net/Articles/128644/)
  بيناقش التغييرات المقترحة على الـ driver core في مرحلة انتقالية مهمة.

- [Driver core finally changing](https://lwn.net/Articles/188707/)
  بيوثّق التغيير الجوهري اللي شاف فيه الـ `class_device` بيتوحّد مع الـ `struct device`.

- [nesting class_device in sysfs to solve world peace](https://lwn.net/Articles/153697/)
  نقاش تقني عن تداخل الـ `class_device` في الـ sysfs tree.

- [unify sysfs device tree](https://lwn.net/Articles/169754/)
  المقترح اللي أدى في النهاية لإلغاء `class_device` ودمجه في `struct device`.

- [driver core: add uid and gid to devtmpfs](https://lwn.net/Articles/546464/)
  بيتعلق بـ `get_ownership` callback الموجودة في `struct class`.

#### الـ kobject والـ sysfs (البنية التحتية)

- [The zen of kobjects](https://lwn.net/Articles/51437/)
  لازم تفهم الـ `kobject` عشان تفهم الـ `class` — ده المرجع الأساسي.

- [kobjects and sysfs](https://lwn.net/Articles/54651/)
  بيشرح العلاقة بين الـ `kobject` وتمثيله في الـ sysfs.

- [Examining a kobject hierarchy](https://lwn.net/Articles/55847/)
  بيرسم الشجرة الكاملة للـ kobject hierarchy بالأمثلة.

- [Changing driver core/sysfs/kobject symbol exports to GPL only](https://lwn.net/Articles/104392/)
  قرار مهم أثّر على كل من بيستخدم الـ class API.

- [API changes in the 2.6 kernel series](https://lwn.net/Articles/183225/)
  مرجع تاريخي للتغييرات في الـ API بما فيها الـ class.

---

### مصادر مكتبة LDD3 على LWN.net

- [Linux Device Drivers, Third Edition — LWN.net](https://lwn.net/Kernel/LDD3/)
  النسخة الكاملة مجانية.

- [The Linux Device Model — LDD3 Chapter 14 (PDF)](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)
  الفصل 14 بيغطي الـ kobject، الـ sysfs، الـ class، والـ bus بالتفصيل — مرجع لازم.

---

### Mailing List Discussions

- [LKML.ORG — Linux Kernel Mailing List Archive](https://lkml.org/)
  ابحث فيه عن: `struct class`, `class_create`, `class_register`, `gregkh driver-core`.

- [class_device_create() and class_interfaces — LKML Archive](https://linux.kernel.narkive.com/F6B67Oag/class-device-create-and-class-interfaces)
  نقاش تقني عن العلاقة بين الـ `class_device_create` والـ `class_interface`.

- [Driving Me Nuts — Device Classes | Linux Journal](https://www.linuxjournal.com/article/6872)
  مقال Greg KH نفسه بيشرح فيه الـ device classes بأمثلة عملية.

---

### Kernel Source — Files ذات الصلة

```
include/linux/device/class.h       ← الملف الرئيسي موضوع الدراسة
include/linux/device.h             ← struct device + macros عامة
drivers/base/class.c               ← implementation كامل للـ class API
drivers/base/core.c                ← device lifecycle + uevent
drivers/base/devtmpfs.c            ← ربط الـ devnode callback بالـ filesystem
lib/kobject.c                      ← kobject البنية التحتية
```

---

### Kernel Commits المهمة

ابحث في [GitHub torvalds/linux](https://github.com/torvalds/linux) بالـ keywords دي:

| الموضوع | Search Term |
|---------|-------------|
| إنشاء `struct class` الأولية | `"driver model: add class"` |
| إزالة `class_device` | `"remove class_device"` |
| إضافة `get_ownership` | `"class: add get_ownership"` |
| تحويل `class_release` لـ `const` | `"class: make class_release const"` |
| فصل `class.h` من `device.h` | `"move class to separate header"` |

**الـ Git log المباشر:**
```bash
git log --oneline -- include/linux/device/class.h
git log --oneline -- drivers/base/class.c
```

---

### KernelNewbies.org

- [Drivers — Linux Kernel Newbies](https://kernelnewbies.org/Drivers)
  نقطة بداية لفهم كيفية كتابة الـ drivers باستخدام الـ class API.

- [Documents — Linux Kernel Newbies](https://kernelnewbies.org/Documents)
  فهرس شامل للمراجع والكتب الموصى بها.

- [Linux 2.6.15 — Kernel Newbies](https://kernelnewbies.org/Linux_2_6_15)
  بيوثّق تحسينات الـ driver core وتداخل الـ `class_device`.

- [Linux 2.6.8 — Kernel Newbies](https://kernelnewbies.org/Linux_2_6_8)
  أول ظهور للـ class support في `cpuid.c` و `msr.c`.

---

### eLinux.org

- [Linux Drivers Device Tree Guide — eLinux.org](https://elinux.org/Linux_Drivers_Device_Tree_Guide)
  بيربط الـ device tree بالـ driver model وكيف الـ `class` بيتفاعل مع الـ OF nodes.

- [Device Tree Reference — eLinux.org](https://elinux.org/Device_Tree_Reference)
  مرجع للـ `of_node` اللي بيُستخدم في `class_find_device_by_of_node`.

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 14 — *The Linux Device Model*
  - بيشرح الـ `kobject`، `kset`، الـ `bus`، الـ `class`، والـ `sysfs` بالتسلسل
- **التحميل المجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition
- **المؤلف:** Robert Love
- **الفصل المهم:** Chapter 17 — *Devices and Modules*
  - بيغطي الـ device model وكيف الـ `class` بيلعب دور التجريد
- **ISBN:** 978-0-672-32946-3

#### Embedded Linux Primer, 2nd Edition
- **المؤلف:** Christopher Hallinan
- **الفصل المهم:** Chapter 8 — *Device Driver Basics*
  - بيشرح الـ `class_create` و `device_create` في سياق الـ embedded systems
- **ISBN:** 978-0-13-701783-7

#### Professional Linux Kernel Architecture
- **المؤلف:** Wolfgang Mauerer
- **الفصل المهم:** Chapter 6 — *Device Drivers*
  - تغطية عميقة للـ sysfs، الـ kobject hierarchy، وبنية الـ class
- **ISBN:** 978-0-470-34343-2

---

### Labs تطبيقية

- [Linux Device Model Lab — linux-kernel-labs.github.io](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html)
  تمارين عملية بتعلمك تسجّل class، تنشئ device، وتضيف sysfs attributes خطوة بخطوة.

---

### Search Terms للبحث عن المزيد

```
linux kernel "struct class" sysfs driver model
linux class_create device_create udev devtmpfs
linux kernel class_interface add_dev remove_dev
linux "class_for_each_device" iterator pattern
linux sysfs /sys/class namespace kobject
gregkh driver-core class device model lkml
linux class_compat backward compatibility sysfs
```

---

### ملخص سريع للمراجع بالأولوية

| الأولوية | المرجع | السبب |
|----------|--------|-------|
| 1 | LDD3 Ch.14 | الأساس النظري الكامل |
| 2 | [lwn.net/Articles/31370](https://lwn.net/Articles/31370/) | شرح الـ class بشكل مباشر |
| 3 | [docs.kernel.org driver-model/class](https://www.kernel.org/doc/html/v5.4/driver-api/driver-model/class.html) | التوثيق الرسمي |
| 4 | [lwn.net/Articles/645810](https://lwn.net/Articles/645810/) | الفهم الحديث للـ model |
| 5 | `drivers/base/class.c` في الـ source | الـ implementation الفعلي |
## Phase 8: Writing simple module

### الفكرة

هنستخدم **`kprobe`** عشان نعمل hook على الفنكشن `class_register` — دي الفنكشن اللي بتتكلم عليها كل driver لما بيسجّل class جديدة في الـ kernel (زي `net`, `input`, `block`, إلخ). كل مرة class جديدة بتتسجل، الـ probe بتطبع اسمها.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_class_register.c
 * Hooks class_register() and logs the name of every class being registered.
 */

/* Core kernel headers */
#include <linux/kernel.h>       /* pr_info, pr_err */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ... */
#include <linux/device/class.h> /* struct class — to read the name field */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDocs");
MODULE_DESCRIPTION("kprobe on class_register — logs every registered class name");

/* ------------------------------------------------------------------ */
/*  pre_handler: runs just BEFORE class_register() executes           */
/* ------------------------------------------------------------------ */

/*
 * pt_regs* regs — بيحتوي على قيم الـ registers لحظة الـ intercept.
 * على x86-64: أول argument بيجي في rdi، وده بالظبط بيـpoint لـ struct class*.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* استخرج الـ argument الأول (struct class *) من الـ register المناسب */
#ifdef CONFIG_X86_64
    const struct class *cls = (const struct class *)regs->di;
#elif defined(CONFIG_ARM64)
    const struct class *cls = (const struct class *)regs->regs[0];
#else
    /* fallback — على architectures تانية نعدّيها بدون طباعة */
    return 0;
#endif

    /* تحقق إن الـ pointer مش NULL وإن name موجودة قبل ما نطبع */
    if (cls && cls->name)
        pr_info("kprobe_class: class_register called → class name = \"%s\"\n",
                cls->name);

    return 0; /* إرجاع 0 = كمّل تنفيذ الفنكشن الأصلية بشكل طبيعي */
}

/* ------------------------------------------------------------------ */
/*  struct kprobe — بتحدد الفنكشن اللي هنعمل عليها hook             */
/* ------------------------------------------------------------------ */

/*
 * symbol_name: اسم الـ symbol زي ما هو في /proc/kallsyms.
 * الـ kernel بيحوّله لعنوان وقت الـ registration.
 */
static struct kprobe kp = {
    .symbol_name = "class_register",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module_init                                                        */
/* ------------------------------------------------------------------ */

static int __init kprobe_class_init(void)
{
    int ret;

    /*
     * register_kprobe() بتحجز الـ probe وبتحط breakpoint افتراضي
     * على أول instruction في class_register().
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_class: register_kprobe failed → %d\n", ret);
        return ret;
    }

    pr_info("kprobe_class: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                        */
/* ------------------------------------------------------------------ */

static void __exit kprobe_class_exit(void)
{
    /*
     * unregister_kprobe() لازم يتنادى في الـ exit عشان:
     *  1. يشيل الـ breakpoint من الـ code.
     *  2. يضمن إن مفيش callback بيتنفذ بعد ما الـ module اتـunload.
     *     لو نسينا ده، الـ kernel هيتكسر (use-after-free على handler_pre).
     */
    unregister_kprobe(&kp);
    pr_info("kprobe_class: kprobe unregistered\n");
}

module_init(kprobe_class_init);
module_exit(kprobe_class_exit);
```

---

### Makefile للـ Out-of-Tree Build

```makefile
obj-m += kprobe_class_register.o

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
| `linux/kernel.h` | **`pr_info`** و `pr_err` لطباعة الـ log |
| `linux/module.h` | ماكروهات الـ module الأساسية (`module_init`, `module_exit`, `MODULE_*`) |
| `linux/kprobes.h` | **`struct kprobe`** وكل API الـ kprobes |
| `linux/device/class.h` | **`struct class`** عشان نقدر نوصل لـ `cls->name` |

---

#### الـ `handler_pre` — لماذا `pt_regs`؟

الـ kprobe بتشتغل على مستوى الـ machine instruction، يعني ما عندهاش معرفة بـ C function signature. عشان كده بنجيب الـ argument الأول من الـ calling convention:
- **x86-64**: أول argument → `rdi` → `regs->di`
- **ARM64**: أول argument → `x0` → `regs->regs[0]`

الـ check على `cls && cls->name` ضروري عشان نتجنب null dereference لو فيه caller غريب.

---

#### ليه `class_register`؟

دي الفنكشن الوحيدة المُصدَّرة اللي كل **class** في الـ kernel بتمر بيها. بمجرد ما تـload الـ module، هتشوف في `dmesg` كل class جديدة بتتسجل — سواء من driver بيتـload أو من `modprobe`. ده بيديك visibility حقيقية على الـ driver model.

---

#### الـ `module_exit` ولماذا `unregister_kprobe`؟

لو ما استدعيناش `unregister_kprobe()` في الـ exit:
1. الـ kernel بيفضل عارف إن في breakpoint على `class_register`.
2. لما الفنكشن دي بتتنادى، بيجري `handler_pre` — لكن الـ handler ده في memory اتـfree بعد الـ unload.
3. النتيجة: **kernel panic** (use-after-free أو page fault في kernel space).

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod kprobe_class_register.ko

# شوف الـ output (جرّب تحمّل أي driver تاني بعده)
sudo dmesg | grep kprobe_class

# مثال على output متوقع:
# kprobe_class: planted kprobe at class_register (ffffffffc0ab1234)
# kprobe_class: class_register called → class name = "input"
# kprobe_class: class_register called → class name = "tty"

# تفريغه
sudo rmmod kprobe_class_register
sudo dmesg | tail -3
# kprobe_class: kprobe unregistered
```

---

### تحقق إن الـ Symbol موجود

```bash
# قبل البناء، تأكد إن class_register مش inline ومش مخفي
sudo grep class_register /proc/kallsyms | head -5
# لو ظهر السطر → الـ kprobe هتشتغل
# لو ما ظهرش → الـ function مربما inlined وهتحتاج tracepoint بديل
```
