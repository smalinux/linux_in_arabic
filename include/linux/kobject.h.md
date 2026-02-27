## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الملف `include/linux/kobject.h` جزء من **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS** — وده من أهم subsystems في الـ Linux kernel. المسؤول عنه هو Greg Kroah-Hartman.

---

### الفكرة الكبيرة — قبل أي كود

#### تخيل المشكلة

افتضل معايا لحظة وفكر: الـ Linux kernel بتشغّل آلاف الـ devices مختلفة — USB، PCI، network cards، sensors، إلخ. كل device ليه اسم، له حالة، محتاج يتحذف لما مفيش حد بيستخدمه، ومحتاج يعرّف نفسه للـ userspace (يعني للبرامج اللي بتشتغل فوق الـ kernel).

السؤال: إيه الـ **foundation** اللي بيبني عليه الكيرنل كل ده؟

الجواب هو: **kobject**.

---

#### الـ kobject زي الـ "شجرة نسب" للـ devices

تخيل إن كل device أو subsystem في الـ kernel هو "شخص" في عيلة كبيرة. الـ **kobject** هو بطاقة هوية كل شخص في العيلة دي. البطاقة دي بتقول:
- اسمه إيه؟ (`name`)
- أبوه مين؟ (`parent`)
- ينتمي لأي مجموعة؟ (`kset`)
- إيه نوعيته؟ (`ktype`)
- مستخدم من كام حد دلوقتي؟ (`kref` — reference count)

لما بتضم الـ kobject جوه struct تاني زي `struct device`، انت بتقول: "الـ device ده له هوية في نظام الكيرنل".

---

#### الـ sysfs — الـ kobjects بتتجسم كـ files

كل kobject بياخد ليه **directory في `/sys`**. يعني لو عندك USB device اسمه `usb1`، هتلاقي `/sys/bus/usb/devices/usb1/`. ده مش صدفة — ده بالظبط الـ kobject بتاعه بيعمل directory في الـ sysfs.

الـ sysfs ده filesystem بس مش على disk — هو view مباشر على الـ kobject tree جوه الكيرنل. البرامج زي `udev` و `systemd` بتقرأ منه عشان تعرف إيه الـ devices الموجودة.

```
/sys/
├── bus/
│   └── usb/
│       └── devices/
│           └── usb1/           <-- kobject directory
│               ├── idVendor    <-- attribute (file)
│               └── idProduct   <-- attribute (file)
├── kernel/                     <-- kernel_kobj
├── firmware/                   <-- firmware_kobj
└── power/                      <-- power_kobj
```

---

#### الـ Reference Counting — مين بيمسك الـ object؟

المشكلة الكلاسيكية: لو driver A بيستخدم device X، وفي نفس الوقت driver B كمان بيستخدم نفس الـ device — مين المسؤول عن الـ cleanup؟ لو A حذف الـ object وB لسه بيستخدمه → crash.

الحل: **kref** — counter بسيط جوه الـ kobject. كل حد بياخد reference بـ `kobject_get()` العداد بيزيد. كل حد يخلص بـ `kobject_put()` العداد بينقص. لما العداد يوصل صفر → الـ `release()` function في الـ `kobj_type` بتتنادي تلقائي وبتعمل cleanup.

---

#### الـ uevent — الـ kernel بيبعت رسايل للـ userspace

لما device يتضاف أو يتشال، الـ kernel محتاج يقول للـ userspace "في device جديد اتضاف!". ده بيحصل عبر **uevent** — رسالة بتتبعت على netlink socket لبرامج زي `udev`.

الـ `enum kobject_action` بيحدد نوع الحدث:
- `KOBJ_ADD` — device اتضاف
- `KOBJ_REMOVE` — device اتشال
- `KOBJ_CHANGE` — حاجة اتغيرت
- `KOBJ_MOVE` — الـ object اتنقل في الشجرة

---

### الـ Structs الأساسية في الملف

#### `struct kobject` — اللبنة الأساسية

```c
struct kobject {
    const char        *name;          // اسم الـ directory في sysfs
    struct list_head   entry;         // ربطه في قايمة الـ kset
    struct kobject    *parent;        // أبو الـ object في الشجرة
    struct kset       *kset;          // المجموعة اللي ينتمي ليها
    const struct kobj_type *ktype;    // نوعيته وعمليات الـ sysfs
    struct kernfs_node *sd;           // entry الـ sysfs الفعلي
    struct kref        kref;          // reference counter

    unsigned int state_initialized:1;         // اتعمله init؟
    unsigned int state_in_sysfs:1;            // موجود في sysfs؟
    unsigned int state_add_uevent_sent:1;     // بعت ADD uevent؟
    unsigned int state_remove_uevent_sent:1;  // بعت REMOVE uevent؟
    unsigned int uevent_suppress:1;           // متبعتش uevents؟
};
```

#### `struct kobj_type` — "شخصية" الـ kobject

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);              // cleanup لما العداد يوصل 0
    const struct sysfs_ops *sysfs_ops;                  // read/write attributes
    const struct attribute_group **default_groups;      // attributes افتراضية في sysfs
    // namespace operations...
};
```

#### `struct kset` — مجموعة الـ kobjects

الـ **kset** هو container بيجمع kobjects من نفس النوع. مثلاً `/sys/bus/` هو kset بيجمع كل الـ buses. الـ kset نفسه له kobject جوّاه (تودد في التصميم).

```c
struct kset {
    struct list_head list;                    // قايمة الـ kobjects
    spinlock_t list_lock;                     // lock للـ thread safety
    struct kobject kobj;                      // هويته هو كـ kobject
    const struct kset_uevent_ops *uevent_ops; // تحكم في uevents
};
```

#### `struct kobj_uevent_env` — بيانات الـ uevent

```c
struct kobj_uevent_env {
    char *argv[3];                        // arguments للـ userspace helper
    char *envp[UEVENT_NUM_ENVP];          // environment variables (64 متغير)
    char buf[UEVENT_BUFFER_SIZE];         // buffer للمتغيرات (2KB)
    int buflen;
};
```

---

### قصة حياة الـ kobject — من الولادة للوفاة

```
1. تعريف الـ struct
   struct my_driver { struct kobject kobj; int data; };

2. تهيئة
   kobject_init(&my->kobj, &my_ktype);

3. إضافة للشجرة
   kobject_add(&my->kobj, parent, "my_device");
   → بييجي directory /sys/.../my_device/
   → بيتبعت KOBJ_ADD uevent → udev بيعرف

4. الاستخدام
   kobject_get() / kobject_put() بكل حد بيستخدمه

5. الحذف
   kobject_del() → بيتشال من sysfs
   kobject_put() → العداد ينقص
   → لو وصل 0: my_ktype.release() بتتنادي → بيتحذف الـ memory
```

---

### الـ Global kobjects المعرّفة في الملف

| المتغير | الـ Path في sysfs | الاستخدام |
|---|---|---|
| `kernel_kobj` | `/sys/kernel/` | معلومات عامة عن الـ kernel |
| `mm_kobj` | `/sys/kernel/mm/` | إعدادات الـ memory management |
| `hypervisor_kobj` | `/sys/hypervisor/` | معلومات الـ hypervisor |
| `power_kobj` | `/sys/power/` | إعدادات الـ power management |
| `firmware_kobj` | `/sys/firmware/` | معلومات الـ firmware |

---

### الملفات المهمة في الـ Subsystem ده

#### الـ Core Files

| الملف | الوظيفة |
|---|---|
| `include/linux/kobject.h` | **الملف ده** — التعريفات والـ API |
| `include/linux/kobject_ns.h` | namespace support للـ kobjects |
| `include/linux/kobject_api.h` | API إضافي |
| `include/linux/kref.h` | الـ reference counting المستخدم في الـ kobject |
| `lib/kobject.c` | التنفيذ الفعلي لكل functions الـ kobject |
| `lib/kobject_uevent.c` | إرسال الـ uevents للـ userspace |

#### الـ sysfs Layer

| الملف | الوظيفة |
|---|---|
| `include/linux/sysfs.h` | الـ sysfs API |
| `fs/sysfs/` | تنفيذ الـ sysfs filesystem |

#### الـ Driver Core (البني عليه الـ kobject)

| الملف | الوظيفة |
|---|---|
| `include/linux/device.h` | الـ `struct device` اللي بتضم kobject |
| `drivers/base/core.c` | الـ device model الكامل |
| `drivers/base/bus.c` | إدارة الـ buses |
| `drivers/base/class.c` | إدارة الـ device classes |
| `drivers/base/driver.c` | إدارة الـ drivers |

#### الـ Documentation

| الملف | المحتوى |
|---|---|
| `Documentation/core-api/kobject.rst` | الدوكيومنتيشن الرسمية — **إقراها الأول** |
| `Documentation/driver-api/driver-model/` | الـ driver model بالكامل |

---

### الملخص في جملة واحدة

**الـ `kobject.h`** بيعرّف الـ building block الأساسي اللي الـ Linux kernel بيبني عليه كل الـ device model — كل device أو subsystem في الـ kernel هو في الآخر kobject بيظهر كـ directory في `/sys/` ويتواصل مع الـ userspace عبر uevents، والـ reference counting بيضمن إن الـ memory متتحذفش غلط.
## Phase 2: شرح الـ kobject Framework

### المشكلة اللي الـ kobject بيحلها

الكيرنل بيتعامل مع آلاف الـ objects في وقت واحد — devices, drivers, buses, modules. كل object ده محتاج:

1. **Lifetime management**: مين بيحذفه ومتى؟ لو حذفت object وفيه حاجة لسه بتستخدمه → kernel panic.
2. **Hierarchy**: الـ USB device جوه USB bus جوه PCI controller — محتاج تعبر عن العلاقة دي.
3. **Userspace visibility**: الـ `/sys` filesystem لازم يعكس الـ objects دي بشكل تلقائي.
4. **Event notification**: لما device اتضاف/اتشال → userspace (udev) لازم يعرف.

قبل الـ kobject، كل subsystem كان بيعمل الـ reference counting والـ sysfs integration بطريقته الخاصة — كان فيه code duplication هائل وbugs كتير في الـ lifetime management.

---

### الحل — الـ kobject Framework

الكيرنل عمل **abstraction layer واحد** بيوفر:

- **Reference counting** آمن عن طريق `kref`
- **Hierarchical naming** ينعكس على `/sys`
- **Automatic sysfs integration** — الـ directory بيتعمل لوحده
- **uevent broadcasting** لـ userspace

الفكرة الأساسية: أي kernel object عايز الخدمات دي → يضم `struct kobject` جواه، وبالتالي يرث كل الـ infrastructure دي مجاناً.

---

### تشبيه من الواقع — نظام ملفات الشركة

تخيل شركة كبيرة فيها موظفين، أقسام، ومعدات:

| مفهوم الشركة | مفهوم الكيرنل |
|---|---|
| **بطاقة هوية الموظف** (ID card مشترك لكل موظف) | `struct kobject` (مضمن في كل kernel object) |
| **عدد الناس اللي بيستخدموا غرفة** (lock قبل ما تقفلها) | `kref` reference count |
| **الهيكل الإداري** (مدير → قسم → موظف) | `kobject->parent` hierarchy |
| **قسم بيضم موظفين من نفس النوع** | `struct kset` |
| **دليل الهاتف الداخلي** (directory يعكس الهيكل) | `/sys` filesystem |
| **نظام الإشعارات** (لما موظف يدخل/يخرج → HR تعرف) | `uevent` → udev |
| **لوائح القسم** (إيه اللي مسموح وإيه اللي ممنوع) | `struct kobj_type` |
| **مدير القسم** اللي بيقرر إيه اللي ينتشر للخارج | `kset_uevent_ops` |

التشبيه ده مش سطحي — لما موظف (device) بيجوين الشركة:
1. بيأخذ بطاقة هوية (`kobject_init`)
2. بيتسجل في دليل الشركة (`kobject_add` → directory في `/sys`)
3. HR بتبعت إشعار للكل (`KOBJ_ADD` uevent → udev)
4. لما بيسيب، لازم كل الناس يوقفوا استخدامه الأول (`kref_put` → لما العداد يوصل صفر → `release()`)

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Userspace                                 │
│   udev daemon          /sys filesystem        applications       │
└────────┬───────────────────────┬────────────────────────────────┘
         │ netlink socket        │ read/write files
         │ (uevent)              │
═════════╪═══════════════════════╪════════════════════════════════
         │                       │
┌────────▼───────────────────────▼────────────────────────────────┐
│                    kobject Core (lib/kobject.c)                  │
│                                                                  │
│   ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│   │  Reference   │    │   sysfs      │    │    uevent       │   │
│   │  Counting    │    │ Integration  │    │  Broadcasting   │   │
│   │  (kref)      │    │ (kernfs)     │    │ (kobject_uevent)│   │
│   └─────────────┘    └──────────────┘    └─────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              struct kobject (the atom)                   │   │
│   │  name | parent* | kset* | ktype* | sd* | kref | state  │   │
│   └─────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────┘
                                     │ embedded in
          ┌──────────────────────────┼──────────────────────────┐
          │                          │                           │
┌─────────▼──────┐        ┌──────────▼──────┐        ┌─────────▼──────┐
│  struct device │        │  struct bus_type│        │  struct module  │
│  ┌──────────┐  │        │  ┌──────────┐   │        │  ┌──────────┐  │
│  │ kobject  │  │        │  │ kobject  │   │        │  │ kobject  │  │
│  └──────────┘  │        │  └──────────┘   │        │  └──────────┘  │
│  + dev-specific│        │  + bus-specific │        │  + mod-specific│
└────────────────┘        └─────────────────┘        └────────────────┘
     Consumers                  Consumers                 Consumers
```

---

### الـ Core Structs بالتفصيل

#### `struct kobject` — الذرة الأساسية

```c
struct kobject {
    const char          *name;        /* اسمه في /sys */
    struct list_head     entry;       /* ربطه في list الـ kset */
    struct kobject      *parent;      /* أبوه في الـ hierarchy */
    struct kset         *kset;        /* المجموعة اللي بيتبعها */
    const struct kobj_type *ktype;    /* نوعه (vtable) */
    struct kernfs_node  *sd;          /* /sys directory entry */
    struct kref          kref;        /* reference counter */

    /* state flags — bitfields */
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;
};
```

**الـ state flags** مهمين:
- `state_initialized`: اتعمله `kobject_init`؟ لو لأ وعملت `kobject_add` → warning
- `state_in_sysfs`: موجود في الـ `/sys`؟ بيمنع double-add
- `state_add_uevent_sent`: اتبعت `KOBJ_ADD` uevent؟ بيمنع uevent مكرر
- `uevent_suppress`: بعض الـ objects مش عايزين يبعتوا events (مثلاً virtual devices)

#### `struct kref` — الـ Reference Counter

الـ `kref` مش مجرد integer — ده **atomic reference counter** مع protection من الـ overflow والـ use-after-free:

```c
struct kref {
    refcount_t refcount;  /* not atomic_t — refcount_t has overflow protection */
};
```

الفرق المهم بين `atomic_t` و`refcount_t`: الـ `refcount_t` بيعمل saturation عند الـ overflow ومش بيوصل صفر عن طريق الغلط. ده مهم جداً لأن الـ use-after-free vulnerabilities كتير بتيجي من reference counting bugs.

```c
/* lifetime cycle */
kobject_init(kobj, ktype);   /* kref = 1 */
kobject_get(kobj);           /* kref++ */
kobject_put(kobj);           /* kref-- → لو وصل 0 → ktype->release() */
```

لما الـ count يوصل صفر → بيتنادى `ktype->release()` → **الـ driver** هو المسؤول عن تحرير الميموري، مش الكيرنل.

#### `struct kobj_type` — الـ vtable

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);
    /* ↑ لازم تكون موجودة — دي اللي بتتنادى لما ref count = 0 */

    const struct sysfs_ops *sysfs_ops;
    /* ↑ show/store callbacks لقراءة/كتابة الـ sysfs attributes */

    const struct attribute_group **default_groups;
    /* ↑ الـ attributes اللي بتتعمل تلقائياً لما الـ kobject يتضاف */

    const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
    /* ↑ للـ network namespacing — أنهي namespace type يستخدم الأطفال */

    const void *(*namespace)(const struct kobject *kobj);
    /* ↑ بترجع الـ namespace pointer للـ kobject نفسه */

    void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
    /* ↑ مين صاحب الـ sysfs entry — مفيد للـ user namespaces */
};
```

**الـ `release` function** هي أهم حاجة في الـ `kobj_type` — لازم تعمل `container_of` عشان توصل للـ struct الحقيقية:

```c
/* مثال حقيقي — driver يعمل release لـ struct device */
static void my_device_release(struct kobject *kobj)
{
    struct my_device *dev = container_of(kobj, struct my_device, kobj);
    /* دلوقتي آمن تحرر الـ struct */
    kfree(dev);
}
```

#### `struct kset` — المجموعة

الـ `kset` هو **container of kobjects** — بيجمع kobjects من نفس النوع ويديلهم parent مشترك:

```c
struct kset {
    struct list_head          list;        /* قايمة كل الـ kobjects */
    spinlock_t                list_lock;   /* protection للـ list */
    struct kobject            kobj;        /* الـ kset نفسه هو kobject! */
    const struct kset_uevent_ops *uevent_ops; /* filter/modify uevents */
} __randomize_layout;
```

لاحظ: الـ `kset` **نفسه هو kobject** — يعني الـ hierarchy ممكن تكون:

```
kset (kobject)
├── kobject
├── kobject
└── kset (kobject) ← nested kset
    ├── kobject
    └── kobject
```

#### `struct kset_uevent_ops` — تحكم في الـ Events

```c
struct kset_uevent_ops {
    int (* const filter)(const struct kobject *kobj);
    /* ↑ return 0 → امنع الـ uevent (مش كل object يستاهل إشعار) */

    const char *(* const name)(const struct kobject *kobj);
    /* ↑ اسم الـ subsystem في الـ uevent (SUBSYSTEM=usb مثلاً) */

    int (* const uevent)(const struct kobject *kobj, struct kobj_uevent_env *env);
    /* ↑ ضيف environment variables للـ uevent (DEVNAME=sda مثلاً) */
};
```

---

### العلاقة بين الـ Structs — رسم تفصيلي

```
                    ┌──────────────────────────────────┐
                    │         struct kset               │
                    │  ┌────────────────────────────┐  │
                    │  │      struct kobject (kobj) │  │
                    │  │  name="devices"            │  │
                    │  │  parent → kernel_kobj      │  │
                    │  │  kref = 1                  │  │
                    │  └────────────────────────────┘  │
                    │  list ──────────────────────────┐ │
                    │  list_lock                      │ │
                    │  uevent_ops → device_uevent_ops │ │
                    └─────────────────────────────────│─┘
                                                      │
              ┌───────────────────────────────────────┤
              │                   │                   │
    ┌─────────▼──────┐  ┌─────────▼──────┐  ┌────────▼───────┐
    │  struct device │  │  struct device │  │  struct device │
    │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │
    │  │ kobject  │  │  │  │ kobject  │  │  │  │ kobject  │  │
    │  │ kset=↑   │  │  │  │ kset=↑   │  │  │  │ kset=↑   │  │
    │  │ ktype=↓  │  │  │  │ ktype=↓  │  │  │  │ ktype=↓  │  │
    │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │
    └────────────────┘  └────────────────┘  └────────────────┘
              │
    ┌─────────▼─────────────┐
    │    struct kobj_type   │
    │  release()            │
    │  sysfs_ops            │
    │  default_groups       │
    └───────────────────────┘
```

---

### الـ sysfs Integration

الـ kobject مرتبط مباشرة بالـ **kernfs** subsystem (اللي هو implementation الـ sysfs).

> **kernfs**: virtual filesystem layer بيحول kernel data structures لـ files وdirectories في `/sys`.

لما بتعمل `kobject_add()`:
1. الكيرنل بيتنادى `sysfs_create_dir_ns()`
2. اللي بيعمل `kernfs_node` جديد ← ده الـ `sd` field في الـ kobject
3. الـ directory بيتعمل في `/sys` على طول

لما بتعمل `kobject_del()`:
1. الـ sysfs directory بيتشال
2. **لكن الـ object مش بيتحذف** — لو فيه حاجة لسه عاملة `kobject_get()`

---

### الـ uevent System

الـ uevent هو الوسيلة اللي الكيرنل بيبعت بيها events لـ userspace عن طريق **netlink socket**:

```c
struct kobj_uevent_env {
    char *argv[3];               /* path to uevent helper */
    char *envp[UEVENT_NUM_ENVP]; /* environment variables (64 max) */
    int   envp_idx;
    char  buf[UEVENT_BUFFER_SIZE]; /* 2048 bytes buffer */
    int   buflen;
};
```

الـ event بيتبعت عن طريق `kobject_uevent()`:

```c
/* مثال — USB device اتضاف */
kobject_uevent(&dev->kobj, KOBJ_ADD);

/* الـ uevent اللي udev بيشوفه:
   ACTION=add
   DEVPATH=/devices/pci0000:00/0000:00:1d.0/usb1/1-1
   SUBSYSTEM=usb
   SEQNUM=1234
*/
```

الـ `uevent_seqnum` هو **monotonic counter** — udev بيستخدمه عشان يعرف ترتيب الـ events ومش يفوته حاجة.

---

### إيه اللي الـ kobject بيمتلكه مقابل إيه اللي بيفوضه للـ drivers

| المسؤولية | الـ kobject Core | الـ Driver |
|---|---|---|
| **Reference counting** (increment/decrement) | نعم — `kobject_get/put` | لأ |
| **سysfs directory creation/deletion** | نعم — تلقائي | لأ |
| **uevent broadcasting** | نعم — `kobject_uevent` | لأ (بس ممكن يضيف env vars) |
| **Object destruction** (تحرير الميموري) | لأ — بس بيعرف إمتى | نعم — `ktype->release()` |
| **sysfs attribute content** (show/store) | لأ | نعم — عن طريق `sysfs_ops` |
| **uevent filtering** | لأ | نعم — عن طريق `kset_uevent_ops->filter` |
| **Namespace isolation** | البنية التحتية | نعم — `ktype->namespace()` |
| **الـ hierarchy تحديد الأب** | لأ | نعم — بيمرر `parent` في `kobject_add` |

---

### Global kobjects — نقاط الربط الثابتة

الكيرنل بيوفر kobjects جاهزة كـ "anchors" لكل الـ subsystems تتعلق فيها:

```c
extern struct kobject *kernel_kobj;    /* /sys/kernel/    */
extern struct kobject *mm_kobj;        /* /sys/kernel/mm/ */
extern struct kobject *hypervisor_kobj;/* /sys/hypervisor/*/
extern struct kobject *power_kobj;     /* /sys/power/     */
extern struct kobject *firmware_kobj;  /* /sys/firmware/  */
```

مثلاً، لما بتعمل kobject جديد وبتديه `kernel_kobj` كـ parent → هيظهر في `/sys/kernel/your_name/`.

---

### Namespace Support في الـ kobject

الـ kobject بيدعم الـ **network namespaces** عشان نفس الـ device يظهر بشكل مختلف في namespaces مختلفة:

```c
enum kobj_ns_type {
    KOBJ_NS_TYPE_NONE = 0,
    KOBJ_NS_TYPE_NET,   /* network namespace */
    KOBJ_NS_TYPES
};
```

الـ `kobj_ns_type_operations` بيعرف إزاي الكيرنل:
- يمسك الـ namespace الحالي (`grab_current_ns`)
- يعرف الـ namespace اللي الـ socket بيتبعه (`netlink_ns`)
- يحرر الـ reference (`drop_ns`)

ده مهم جداً للـ containers — كل container بيشوف `/sys/class/net` بشكل مختلف بناءً على الـ network namespace بتاعته.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums، الـ Flags، والـ Config Options

#### `enum kobject_action` — أنواع الـ uevent

| القيمة | المعنى | مثال واقعي |
|---|---|---|
| `KOBJ_ADD` | كائن اتضاف لأول مرة | device جديد اتعرف على الـ bus |
| `KOBJ_REMOVE` | كائن اتشال | USB device اتفصل |
| `KOBJ_CHANGE` | حالة الكائن اتغيرت | battery level تغير |
| `KOBJ_MOVE` | الكائن اتنقل لـ parent تاني | device اتنقل بين subsystems |
| `KOBJ_ONLINE` | الكائن بقى online | CPU اتفعّل بعد hotplug |
| `KOBJ_OFFLINE` | الكائن بقى offline | CPU اتعطّل قبل ما يتشال |
| `KOBJ_BIND` | driver اترتبط بالكائن | driver اتبند على device |
| `KOBJ_UNBIND` | driver اتفصل | driver اتفصل قبل remove |

#### `enum kobj_ns_type` — أنواع الـ Namespace

| القيمة | المعنى |
|---|---|
| `KOBJ_NS_TYPE_NONE` | مفيش namespace (الحالة الافتراضية) |
| `KOBJ_NS_TYPE_NET` | network namespace |
| `KOBJ_NS_TYPES` | عدد الأنواع المتاحة (sentinel) |

#### Bitfield States في `struct kobject`

| الـ Field | المعنى |
|---|---|
| `state_initialized` | `kobject_init()` اتنادى عليه |
| `state_in_sysfs` | الكائن موجود في sysfs دلوقتي |
| `state_add_uevent_sent` | الـ `KOBJ_ADD` uevent اتبعت |
| `state_remove_uevent_sent` | الـ `KOBJ_REMOVE` uevent اتبعت |
| `uevent_suppress` | اتمنع إرسال أي uevent لهذا الكائن |

#### Compile-time Config Options

| الـ Option | الأثر |
|---|---|
| `CONFIG_UEVENT_HELPER` | يفعّل `uevent_helper[]` — path لـ userspace helper binary |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يضيف `delayed_work release` في الـ kobject لتأخير الـ release لكشف use-after-free |

#### Constants مهمة

| الثابت | القيمة | الاستخدام |
|---|---|---|
| `UEVENT_HELPER_PATH_LEN` | 256 | طول path الـ helper binary |
| `UEVENT_NUM_ENVP` | 64 | عدد environment variable pointers |
| `UEVENT_BUFFER_SIZE` | 2048 | حجم buffer متغيرات الـ uevent |

---

### الـ Structs المهمة

#### 1. `struct kobject`

**الغرض:** الوحدة الأساسية في الـ driver model. كل device، bus، driver، أو subsystem بيحتوي على kobject مضمّن فيه. بيوفر: اسم، مكان في sysfs، reference counting، وإرسال uevent.

```c
struct kobject {
    const char          *name;         /* اسم الدايركتوري في sysfs */
    struct list_head    entry;         /* لربطه في list الـ kset بتاعه */
    struct kobject      *parent;       /* الـ kobject الأب — بيحدد مكانه في sysfs */
    struct kset         *kset;         /* المجموعة اللي بيتبعها */
    const struct kobj_type *ktype;     /* ops: release, sysfs_ops, default attrs */
    struct kernfs_node  *sd;           /* الـ inode بتاعه في sysfs */
    struct kref         kref;          /* reference counter */

    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    unsigned int state_add_uevent_sent:1;
    unsigned int state_remove_uevent_sent:1;
    unsigned int uevent_suppress:1;

#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
    struct delayed_work release;       /* تأخير الـ release للـ debug */
#endif
};
```

**الارتباطات:**
- `parent` → kobject تاني (بيشكّل شجرة sysfs)
- `kset` → `struct kset` (المجموعة)
- `ktype` → `struct kobj_type` (الـ vtable)
- `sd` → `struct kernfs_node` (sysfs backend)
- `kref` → `struct kref` (reference count)

---

#### 2. `struct kref`

**الغرض:** reference counter آمن. بيستخدم `refcount_t` اللي بيحمي من integer overflow وuse-after-free.

```c
struct kref {
    refcount_t refcount;   /* atomic reference count */
};
```

**الارتباطات:** مضمّن مباشرة في `struct kobject`. الـ `kobject_put()` بينادي `kref_put()` اللي لو وصل لـ صفر بينادي `ktype->release()`.

---

#### 3. `struct kobj_type`

**الغرض:** الـ "vtable" للـ kobject. بيحدد السلوك الخاص لكل نوع من الكائنات — إزاي يتحرر، إزاي sysfs يقرأ/يكتب فيه، وإيه الـ attributes الافتراضية بتاعته.

```c
struct kobj_type {
    void (*release)(struct kobject *kobj);
    /* destructor — اللي بينادي kfree على الـ container struct */

    const struct sysfs_ops *sysfs_ops;
    /* show/store callbacks لـ sysfs */

    const struct attribute_group **default_groups;
    /* مصفوفة من groups من attributes تتعمل تلقائيًا */

    const struct kobj_ns_type_operations *(*child_ns_type)(const struct kobject *kobj);
    /* نوع الـ namespace للـ children */

    const void *(*namespace)(const struct kobject *kobj);
    /* رجع الـ namespace الخاص بالكائن ده */

    void (*get_ownership)(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
    /* بيحدد UID/GID لـ sysfs entries بتاعته */
};
```

**الارتباطات:** الـ `kobject->ktype` بيشاور عليه. الـ `release` callback هو الطريقة الوحيدة الصح لتحرير memory الكائن.

---

#### 4. `struct kset`

**الغرض:** مجموعة من kobjects من نفس النوع. بيحتوي على embedded kobject يمثّله هو نفسه في sysfs. مسؤول عن فلترة وتوليد uevents لكل الكائنات فيه.

```c
struct kset {
    struct list_head list;                       /* list بكل kobjects المنتمية له */
    spinlock_t       list_lock;                  /* حماية الـ list */
    struct kobject   kobj;                       /* كيانه هو في sysfs */
    const struct kset_uevent_ops *uevent_ops;    /* فلترة وإضافة env للـ uevent */
} __randomize_layout;
```

**ملاحظة:** `__randomize_layout` بيعني الـ kernel ممكن يعيد ترتيب الـ fields عشان KASLR.

**الارتباطات:**
- `kobj` مضمّن — الـ kset نفسه ليه أب في sysfs
- `list` بيحتوي على كل `kobject->entry` المنتمية له
- `uevent_ops` → `struct kset_uevent_ops`

---

#### 5. `struct kset_uevent_ops`

**الغرض:** بيتحكم في الـ uevent اللي بيتبعت لـ userspace عند أي حدث على الكائنات في الـ kset.

```c
struct kset_uevent_ops {
    int (* const filter)(const struct kobject *kobj);
    /* لو رجع 0 → متبعتش الـ uevent لهذا الكائن */

    const char *(* const name)(const struct kobject *kobj);
    /* اسم الـ subsystem في الـ uevent (e.g., "block", "net") */

    int (* const uevent)(const struct kobject *kobj, struct kobj_uevent_env *env);
    /* ضيف environment variables إضافية للـ uevent */
};
```

---

#### 6. `struct kobj_uevent_env`

**الغرض:** يمثّل بيئة الـ uevent المبعوت لـ userspace. بيحتوي على الـ argv والـ envp والـ buffer الخاص بالمتغيرات.

```c
struct kobj_uevent_env {
    char *argv[3];                    /* arguments للـ helper binary */
    char *envp[UEVENT_NUM_ENVP];      /* pointers لـ environment variables (64) */
    int  envp_idx;                    /* index فين وصلنا في الـ envp */
    char buf[UEVENT_BUFFER_SIZE];     /* buffer فعلي بيحتوي KEY=VALUE strings (2048 bytes) */
    int  buflen;                      /* المستخدم من الـ buffer */
};
```

---

#### 7. `struct kobj_attribute`

**الغرض:** attribute خاص بـ kobject مباشر (مش عبر device_attribute). بيُستخدم لـ /sys/kernel/ entries.

```c
struct kobj_attribute {
    struct attribute attr;    /* الـ name والـ mode (permissions) */
    ssize_t (*show)(struct kobject *kobj, struct kobj_attribute *attr, char *buf);
    ssize_t (*store)(struct kobject *kobj, struct kobj_attribute *attr,
                     const char *buf, size_t count);
};
```

---

#### 8. `struct kobj_ns_type_operations`

**الغرض:** callbacks بتعرّف للـ kernel إزاي يتعامل مع namespace معين في sysfs (حاليًا بس network namespace).

```c
struct kobj_ns_type_operations {
    enum kobj_ns_type type;
    bool  (*current_may_mount)(void);     /* الـ process الحالي مسموحله يعمل mount؟ */
    void *(*grab_current_ns)(void);       /* جيب reference للـ namespace الحالي */
    const void *(*netlink_ns)(struct sock *sk); /* الـ namespace بتاع socket معين */
    const void *(*initial_ns)(void);      /* الـ init namespace */
    void (*drop_ns)(void *);              /* حرر reference للـ namespace */
};
```

---

### Struct Relationship Diagram

```
                    ┌─────────────────────────────────────┐
                    │           struct kset                │
                    │  ┌─────────────────────┐            │
                    │  │   struct kobject     │ (embedded) │
                    │  │     kobj             │            │
                    │  └──────────┬──────────┘            │
                    │  list ──────┼───────────────────┐   │
                    │  list_lock  │                   │   │
                    │  uevent_ops─┼──► kset_uevent_ops│   │
                    └────────────┼───────────────────┼───┘
                                 │                   │
                    ▲            │                   │ (entry list_head)
                    │ parent     │                   │
                    │            ▼                   ▼
               ┌────┴──────────────────────────────────────┐
               │              struct kobject                │
               │  name ──────────────► "my_device"         │
               │  parent ────────────► kobject (parent)    │
               │  kset ──────────────► struct kset         │
               │  ktype ─────────────► struct kobj_type    │
               │  sd ────────────────► kernfs_node (sysfs) │
               │  kref ──────────────► struct kref         │
               │  state bits                               │
               └───────────────────────────────────────────┘
                         │             │
                         │             │
                         ▼             ▼
               ┌──────────────┐  ┌──────────────────────┐
               │ struct kref  │  │   struct kobj_type   │
               │  refcount_t  │  │  release()           │
               │  (atomic)    │  │  sysfs_ops ──────────┼──► sysfs_ops
               └──────────────┘  │  default_groups      │
                                 │  child_ns_type()     │
                                 │  namespace()         │
                                 │  get_ownership()     │
                                 └──────────────────────┘
```

---

### Kobject في sysfs — شجرة المسارات

```
/sys/
 └── kernel/          ◄── kernel_kobj  (struct kobject*)
 │    └── mm/         ◄── mm_kobj
 ├── hypervisor/      ◄── hypervisor_kobj
 ├── power/           ◄── power_kobj
 └── firmware/        ◄── firmware_kobj

مثال: device في /sys/bus/pci/devices/0000:00:1f.0/
  └── kobject->parent = &pci_bus_kobj
  └── kobject->kset   = devices_kset
  └── kobject->sd     = kernfs_node لمجلد 0000:00:1f.0
```

---

### Lifecycle Diagram — دورة حياة الـ kobject

```
1. ALLOCATION
   ─────────────────────────────────────────────────────
   المستخدم بيخصص struct يحتوي على kobject مضمّن:

   struct my_dev {
       int data;
       struct kobject kobj;   ← مضمّن
   };
   dev = kzalloc(sizeof(*dev), GFP_KERNEL);

2. INITIALIZATION
   ─────────────────────────────────────────────────────
   kobject_init(&dev->kobj, &my_ktype);
     → kref_init(&kobj->kref)        // refcount = 1
     → kobj->ktype = &my_ktype
     → state_initialized = 1

3. REGISTRATION (ADD TO SYSFS)
   ─────────────────────────────────────────────────────
   kobject_add(&dev->kobj, parent_kobj, "my_device");
     → يتحقق state_initialized
     → بيشوف kset لو موجود
     → create_dir() في sysfs
     → state_in_sysfs = 1
     → kobject_uevent(KOBJ_ADD)
       → state_add_uevent_sent = 1

   [أو في خطوة واحدة]
   kobject_init_and_add(...)    ← init + add معًا
   kobject_create_and_add(...)  ← alloc + init + add معًا

4. USAGE
   ─────────────────────────────────────────────────────
   kobject_get(&dev->kobj)   → kref++ (refcount = 2)
   kobject_put(&dev->kobj)   → kref-- (refcount = 1)

   كل مرة كود تاني بياخد reference بينادي kobject_get
   وبيرجعها بـ kobject_put

5. REMOVAL
   ─────────────────────────────────────────────────────
   kobject_del(&dev->kobj);
     → kobject_uevent(KOBJ_REMOVE)
       → state_remove_uevent_sent = 1
     → sysfs_remove_dir()
     → state_in_sysfs = 0
     → list_del من kset

6. TEARDOWN (آخر kobject_put)
   ─────────────────────────────────────────────────────
   kobject_put(&dev->kobj)
     → kref-- → وصل صفر
       → kobj_type->release(kobj)
         → container_of(kobj, struct my_dev, kobj)
         → kfree(dev)
```

**تحذير مهم:** `kobject_del()` و `kobject_put()` مستقلّين. لازم تنادي `del` قبل آخر `put`. لو نادى `put` الأول بدون `del` → سيحصل memory leak في sysfs.

---

### Call Flow Diagrams

#### إضافة kobject جديد

```
driver_probe() أو __init
  └── kobject_init_and_add(&kobj, &ktype, parent, "name")
        ├── kobject_init(&kobj, &ktype)
        │     ├── kref_init(&kobj->kref)         // refcount=1
        │     ├── INIT_LIST_HEAD(&kobj->entry)
        │     └── kobj->state_initialized = 1
        └── kobject_add_varg(&kobj, parent, fmt, ...)
              ├── kobject_set_name_vargs()        // kobj->name = "name"
              └── kobject_add_internal()
                    ├── kobject_get(parent)       // parent refcount++
                    ├── kobj->parent = parent
                    ├── [kset] → list_add_tail(&kobj->entry, &kset->list)
                    ├── create_dir() → sysfs_create_dir_ns()
                    │     └── kernfs_create_dir() → kobj->sd
                    └── kobj->state_in_sysfs = 1
```

#### إرسال uevent

```
kobject_uevent(&kobj, KOBJ_ADD)
  └── kobject_uevent_env(&kobj, KOBJ_ADD, NULL)
        ├── top_kobj = بيطلع لفوق لحد ما يلاقي kset
        ├── kset->uevent_ops->filter(kobj)       // هل نبعته؟
        │     └── لو رجع 0 → skip
        ├── kset->uevent_ops->name(kobj)         // subsystem name
        ├── env = kzalloc(kobj_uevent_env)
        ├── add_uevent_var(env, "ACTION=add")
        ├── add_uevent_var(env, "DEVPATH=...")
        ├── add_uevent_var(env, "SUBSYSTEM=...")
        ├── kset->uevent_ops->uevent(kobj, env)  // custom vars
        ├── [netlink] kobject_uevent_net_broadcast()
        └── [helper] call_usermodehelper(uevent_helper, argv, envp)
```

#### تحرير الـ kobject (آخر reference)

```
kobject_put(&kobj)
  └── kref_put(&kobj->kref, kobject_release)
        └── [refcount==0] kobject_release(&kref)
              ├── [DEBUG] schedule delayed_work  ← CONFIG_DEBUG_KOBJECT_RELEASE
              └── kobject_cleanup(kobj)
                    ├── [state_in_sysfs] kobject_del(kobj)  // لو نُسي
                    ├── ktype->release(kobj)
                    │     └── container_of → kfree(my_struct)
                    └── kobject_put(parent)       // parent refcount--
```

#### قراءة sysfs attribute

```
userspace: cat /sys/kernel/my_obj/my_attr
  └── vfs_read()
        └── kernfs_fop_read()
              └── kobj_attr_show()
                    └── kobj_attribute->show(kobj, attr, buf)
                          └── driver code fills buf
                                └── return count
```

---

### Locking Strategy

#### الـ Locks المستخدمة وما يحميه

| الـ Lock | النوع | ما يحميه |
|---|---|---|
| `kset->list_lock` | `spinlock_t` | الـ `list` بتاع الـ kset — إضافة وحذف kobjects |
| `kref->refcount` | `refcount_t` (atomic) | الـ reference count نفسه |
| `kobject->sd` | محمي بـ kernfs locks | sysfs operations |

#### ترتيب الـ Locks (Lock Ordering)

```
من الخارج للداخل:
  kset->list_lock
    └── (لا يحتاج locks تانية أثناء إمساكه)

في kobject_add_internal:
  spin_lock(&kset->list_lock)
    list_add_tail(&kobj->entry, &kset->list)
  spin_unlock(&kset->list_lock)
  [ثم] sysfs operations (kernfs يمسك locks داخلية منفصلة)
```

#### Reference Counting Rules (القواعد الذهبية)

```
1. kobject_init()  → refcount = 1
2. kobject_get()   → refcount++
3. kobject_put()   → refcount--  [لو 0 → release()]
4. kobject_del()   لا يغير الـ refcount → بس يشيله من sysfs

القاعدة الأساسية:
  - كل كود بياخد pointer لـ kobject لازم يعمل kobject_get()
  - وقبل ما يخلص يعمل kobject_put()
  - kobject_get_unless_zero() → آمن لو الكائن ممكن يكون بيتحذف
```

#### Context Safety

| الـ Operation | Safe في IRQ context؟ |
|---|---|
| `kobject_get()` / `kobject_put()` | نعم (atomic) |
| `kobject_add()` / `kobject_del()` | لا (قد يخصص memory أو ينتظر) |
| `kobject_uevent()` | لا (بيعمل alloc وnetlink) |
| `kset->list_lock` spin_lock | نعم (spinlock) |
## Phase 4: شرح الـ Functions

---

### جدول الـ API — Cheatsheet

#### kobject Lifecycle

| Function | Signature Summary | Purpose |
|---|---|---|
| `kobject_init` | `(kobj, ktype)` | يبدأ الـ kobject بدون ما يضيفه لـ sysfs |
| `kobject_add` | `(kobj, parent, fmt, ...)` | يضيف الـ kobject لـ sysfs تحت parent |
| `kobject_init_and_add` | `(kobj, ktype, parent, fmt, ...)` | init + add في خطوة واحدة |
| `kobject_del` | `(kobj)` | يشيل الـ kobject من sysfs |
| `kobject_create_and_add` | `(name, parent)` | يعمل kobject dynamic ويضيفه لـ sysfs |

#### kobject Reference Counting

| Function | Signature Summary | Purpose |
|---|---|---|
| `kobject_get` | `(kobj)` | يزود الـ refcount |
| `kobject_get_unless_zero` | `(kobj)` | يزود الـ refcount لو مش صفر |
| `kobject_put` | `(kobj)` | ينقص الـ refcount، لو وصل صفر بيستدعي release |

#### kobject Naming & Path

| Function | Signature Summary | Purpose |
|---|---|---|
| `kobject_set_name` | `(kobj, fmt, ...)` | يعيّن اسم الـ kobject |
| `kobject_set_name_vargs` | `(kobj, fmt, vargs)` | نفس السابق بـ va_list |
| `kobject_name` | `(kobj)` → `const char*` | يرجع اسم الـ kobject |
| `kobject_get_path` | `(kobj, gfp)` → `char*` | يرجع الـ sysfs path كاملاً |
| `kobject_rename` | `(kobj, new_name)` | يغيّر اسم الـ kobject في sysfs |
| `kobject_move` | `(kobj, new_parent)` | ينقل الـ kobject لـ parent جديد |

#### kobject Namespace & Ownership

| Function | Signature Summary | Purpose |
|---|---|---|
| `kobject_namespace` | `(kobj)` → `const void*` | يرجع الـ namespace الخاص بالـ kobject |
| `kobject_get_ownership` | `(kobj, uid, gid)` | يرجع الـ UID/GID لـ sysfs entries |

#### kset Lifecycle

| Function | Signature Summary | Purpose |
|---|---|---|
| `kset_init` | `(kset)` | يبدأ الـ kset بدون تسجيل |
| `kset_register` | `(kset)` | يسجّل الـ kset ويضيفه لـ sysfs |
| `kset_unregister` | `(kset)` | يلغي تسجيل الـ kset ويشيله |
| `kset_create_and_add` | `(name, uevent_ops, parent)` | يعمل kset dynamic ويسجّله |
| `kset_find_obj` | `(kset, name)` → `kobject*` | يدوّر على kobject بالاسم جوّا الـ kset |

#### kset Reference Counting

| Function | Signature Summary | Purpose |
|---|---|---|
| `kset_get` | `(k)` → `kset*` | يزود الـ refcount للـ kset |
| `kset_put` | `(k)` | ينقص الـ refcount للـ kset |
| `to_kset` | `(kobj)` → `kset*` | يحوّل من kobject لـ kset |

#### uevent API

| Function | Signature Summary | Purpose |
|---|---|---|
| `kobject_uevent` | `(kobj, action)` | يبعت uevent بدون env إضافي |
| `kobject_uevent_env` | `(kobj, action, envp[])` | يبعت uevent مع env إضافي |
| `kobject_synth_uevent` | `(kobj, buf, count)` | يبعت synthetic uevent من userspace |
| `add_uevent_var` | `(env, fmt, ...)` | يضيف متغير لـ uevent environment |

#### Namespace (kobject_ns)

| Function | Signature Summary | Purpose |
|---|---|---|
| `kobj_ns_type_register` | `(ops)` | يسجّل namespace type جديد |
| `kobj_ns_type_registered` | `(type)` | يتأكد لو الـ type مسجّل |
| `kobj_child_ns_ops` | `(parent)` | يرجع ops الـ namespace للـ children |
| `kobj_ns_ops` | `(kobj)` | يرجع ops الـ namespace للـ kobject نفسه |
| `kobj_ns_grab_current` | `(type)` | يمسك reference على الـ namespace الحالي |
| `kobj_ns_drop` | `(type, ns)` | يسيّب reference على الـ namespace |

---

### المجموعة 1: Initialization & Registration

الـ group ده مسؤول عن إنشاء وتسجيل الـ kobjects والـ ksets. الفكرة الأساسية إن الـ kobject لازم يتعمله `init` الأول لإنه بيصفر الـ state ويربطه بـ `ktype`، وبعدين `add` عشان يتظهر في sysfs.

---

#### `kobject_init`

```c
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype);
```

بتعمل initialize للـ kobject عن طريق إنها تضبط الـ `kref` على 1 وتربط الـ `ktype` بالـ kobject وتعمل init للـ `list_head`. بتشيل الـ `state_initialized` flag عشان تمنع double-init. لازم تتستدعي قبل أي `kobject_add`.

**Parameters:**
- `kobj` — البوينتر للـ kobject اللي عايز تبدأه
- `ktype` — الـ `kobj_type` اللي بتحدد `release`، `sysfs_ops`، وغيرها — **لازم مش NULL**

**Return:** void

**Key details:**
- بتعمل `WARN_ON` لو `kobj` أو `ktype` كانوا NULL
- بتعمل `WARN_ON` لو الـ kobject اتعمله init قبل كده (double-init)
- لا بتضيف لـ sysfs ولا بتبعت uevent

**Who calls it:** أي كود بدأ kobject statically أو embedded في struct أكبر.

---

#### `kobject_add`

```c
int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);
```

بتضيف الـ kobject لـ sysfs تحت الـ parent المحدد، وبتعمل mkdir للـ directory وبتضيف الـ default attributes من `ktype->default_groups`. بتبعت `KOBJ_ADD` uevent بعد النجاح.

**Parameters:**
- `kobj` — الـ kobject اللي اتعمله `kobject_init` قبل كده
- `parent` — الـ kobject الأب في الـ sysfs hierarchy — ممكن يكون NULL فيتعلق بـ root
- `fmt, ...` — format string للاسم — يتحول لـ `kobj->name`

**Return:** 0 عند النجاح، errno سالب عند الفشل (مثلاً `-ENOMEM`, `-EEXIST`)

**Key details:**
- `__must_check` — لازم تتحقق من الـ return value
- بتعمل `WARN_ON` لو الـ kobject مش متعمله init
- لو الـ kobject عنده `kset` وما فيش parent صريح، بيستخدم `kset->kobj` كـ parent
- بتضيف الـ kobject لـ `kset->list` لو في kset

**Pseudocode:**
```
kobject_add(kobj, parent, fmt):
    kobject_set_name(kobj, fmt)
    kobj->parent = parent ?: kset->kobj
    create_dir(kobj)              // kernfs_create_dir
    add default_groups to sysfs
    add to kset->list
    set state_in_sysfs = 1
    send KOBJ_ADD uevent          // via kobject_uevent
```

---

#### `kobject_init_and_add`

```c
int kobject_init_and_add(struct kobject *kobj,
                          const struct kobj_type *ktype,
                          struct kobject *parent,
                          const char *fmt, ...);
```

بتعمل `kobject_init` + `kobject_add` في خطوة واحدة atomic من الـ caller's perspective. لو `kobject_add` فشل، بتعمل `kobject_put` تلقائياً عشان تنظّف.

**Parameters:** نفس `kobject_init` و`kobject_add` مجموعين.

**Return:** 0 عند النجاح، errno سالب عند الفشل.

**Key details:**
- أسهل API للاستخدام في drivers لأنها بتتعامل مع الـ cleanup تلقائياً
- لو فشلت، الـ kobject بيبقى في حالة initialized بس مش في sysfs

---

#### `kobject_del`

```c
void kobject_del(struct kobject *kobj);
```

بتشيل الـ kobject من sysfs بشكل كامل: بتحذف الـ directory والـ attributes وبتبعت `KOBJ_REMOVE` uevent وبتشيله من `kset->list`. **مش بتحرر الـ memory** — ده دور `kobject_put`.

**Parameters:**
- `kobj` — الـ kobject المراد إزالته من sysfs

**Return:** void

**Key details:**
- بتعمل `state_in_sysfs = 0` بعد الإزالة
- لازم يجيها call قبل الـ `kobject_put` النهائي لو الـ kobject اتضاف لـ sysfs
- لو الـ `uevent_suppress` مضبوط، الـ KOBJ_REMOVE uevent مش بيتبعت

---

#### `kobject_create_and_add`

```c
struct kobject *kobject_create_and_add(const char *name, struct kobject *parent);
```

بتعمل allocate لـ kobject جديد dynamically، بتعمله init بـ `dynamic_kobj_ktype` (اللي `release` فيها `kfree`)، وبتضيفه لـ sysfs. ده الـ API المفضّل لعمل simple sysfs directories.

**Parameters:**
- `name` — اسم الـ directory في sysfs
- `parent` — الـ parent kobject — ممكن NULL (root `/sys/`)

**Return:** pointer للـ kobject الجديد، أو NULL عند الفشل.

**Key details:**
- `__must_check`
- الـ kobject المنشأ بيتحرر automatically لما `kobject_put` تخلّي الـ refcount يوصل صفر
- المستخدم مش محتاج يعرف حاجة عن `kobj_type`

---

#### `kset_init`

```c
void kset_init(struct kset *kset);
```

بتعمل initialize للـ kset بدون ما تضيفه لـ sysfs. بتعمل init لـ `kset->list` و`kset->list_lock` وبتستدعي `kobject_init` على `kset->kobj`.

**Parameters:**
- `kset` — الـ kset المراد تهيئته

**Return:** void

---

#### `kset_register`

```c
int kset_register(struct kset *kset);
```

بتسجّل الـ kset اللي اتعمله `kset_init` مسبقاً وبتضيف `kset->kobj` لـ sysfs. بتبعت `KOBJ_ADD` uevent.

**Parameters:**
- `kset` — الـ kset المراد تسجيله

**Return:** 0 عند النجاح، errno عند الفشل.

**Key details:**
- `__must_check`
- لو فشل `kobject_add` الداخلي، بيرجع الـ error مباشرةً

---

#### `kset_unregister`

```c
void kset_unregister(struct kset *kset);
```

بتعكس `kset_register`: بتشيل `kset->kobj` من sysfs وبتعمل `kobject_put` عليه. لما الـ refcount يوصل صفر، بيتحرر الـ kset.

**Parameters:**
- `kset` — الـ kset المراد إلغاء تسجيله

**Return:** void

---

#### `kset_create_and_add`

```c
struct kset *kset_create_and_add(const char *name,
                                  const struct kset_uevent_ops *u,
                                  struct kobject *parent_kobj);
```

بتعمل allocate وتعمل init وتسجّل الـ kset في خطوة واحدة. ده الـ API الأشيع في subsystems زي `bus_type` و`device_type`.

**Parameters:**
- `name` — اسم الـ kset في sysfs
- `u` — الـ uevent ops الخاصة بالـ kset — ممكن NULL
- `parent_kobj` — الـ parent في sysfs hierarchy

**Return:** pointer للـ kset الجديد، أو NULL عند الفشل.

---

### المجموعة 2: Reference Counting

الـ kobject subsystem مبني على **reference counting** من خلال `struct kref`. الـ invariant الأساسي: لما الـ refcount يوصل صفر، `ktype->release` بتتستدعي. الـ caller مسؤول عن balance بين كل `get` و`put`.

---

#### `kobject_get`

```c
struct kobject *kobject_get(struct kobject *kobj);
```

بتزود الـ refcount للـ kobject عن طريق `kref_get`. Safe مع NULL (بترجع NULL).

**Parameters:**
- `kobj` — الـ kobject — ممكن NULL

**Return:** نفس `kobj` المُمرَّر، أو NULL لو كان NULL.

**Key details:**
- بتستدعي `kref_get(&kobj->kref)` داخلياً
- **مش** thread-safe لو الـ kobject ممكن يتحرر concurrently — في الحالة دي استخدم `kobject_get_unless_zero`

---

#### `kobject_get_unless_zero`

```c
struct kobject *kobject_get_unless_zero(struct kobject *kobj);
```

بتزود الـ refcount بس لو مش صفر — مفيدة في RCU lookups عشان تتجنب race مع الـ release. بتستخدم `kref_get_unless_zero` اللي بتستخدم `refcount_inc_not_zero` اللي هي atomic.

**Parameters:**
- `kobj` — الـ kobject المراد الـ get عليه

**Return:** `kobj` عند النجاح، NULL لو الـ refcount كان وصل صفر.

**Key details:**
- `__must_check`
- الاستخدام الصح: RCU read lock + lookup + `kobject_get_unless_zero` + تحقق من return

---

#### `kobject_put`

```c
void kobject_put(struct kobject *kobj);
```

بتنقص الـ refcount. لو وصل صفر، بتستدعي `ktype->release(kobj)` اللي المفروض تحرر الـ memory. Safe مع NULL.

**Parameters:**
- `kobj` — الـ kobject المراد تحرير reference منه

**Return:** void

**Key details:**
- **مش** mutex-protected — الـ release callback بتتستدعي بدون أي lock
- لو الـ `CONFIG_DEBUG_KOBJECT_RELEASE` مفعّل، الـ release بتتأجل بـ `delayed_work` لكشف use-after-free
- **الأكثر أهمية:** `kobject_del` لازم تتستدعي قبل الـ put النهائي لو الـ kobject كان في sysfs

**Pseudocode:**
```
kobject_put(kobj):
    if kobj == NULL: return
    kref_put(&kobj->kref, kobject_release)
        → if refcount hits 0:
            kobject_cleanup(kobj)
                → kobject_del(kobj)    // if still in sysfs
                → ktype->release(kobj) // free memory
```

---

#### `kset_get` / `kset_put`

```c
static inline struct kset *kset_get(struct kset *k);
static inline void kset_put(struct kset *k);
```

**الـ `kset_get`** بتزود الـ refcount للـ kset عن طريق `kobject_get(&k->kobj)` ثم `to_kset`.
**الـ `kset_put`** بتنقص الـ refcount للـ kset عن طريق `kobject_put(&k->kobj)`.

**Key details:** الـ kset مش عنده refcount مستقل — بيعتمد على الـ kobject المضمّن فيه.

---

### المجموعة 3: Naming & Path

---

#### `kobject_set_name`

```c
int kobject_set_name(struct kobject *kobj, const char *name, ...);
```

بتعيّن اسم الـ kobject بعد format expansion. بتعمل allocate لـ string جديدة وتحطها في `kobj->name`. لو في اسم قديم، بتحرره.

**Parameters:**
- `kobj` — الـ kobject المراد تعيين اسمه
- `name, ...` — printf-style format string

**Return:** 0 عند النجاح، `-ENOMEM` عند فشل الـ allocation.

**Key details:**
- بتستخدم `kasprintf` داخلياً
- `__printf(2, 3)` لـ compile-time format checking

---

#### `kobject_set_name_vargs`

```c
int kobject_set_name_vargs(struct kobject *kobj, const char *fmt, va_list vargs);
```

نفس `kobject_set_name` بس بتستقبل `va_list` بدل `...`. بتتستدعي من `kobject_set_name` نفسها.

---

#### `kobject_name`

```c
static inline const char *kobject_name(const struct kobject *kobj);
```

بترجع `kobj->name` مباشرةً بدون أي synchronization. Fast و inline.

---

#### `kobject_get_path`

```c
char *kobject_get_path(const struct kobject *kobj, gfp_t flag);
```

بتبني الـ full sysfs path للـ kobject (مثلاً `/devices/pci0000:00/...`) عن طريق المشي على chain الـ parents. بتعمل allocate لـ string.

**Parameters:**
- `kobj` — الـ kobject
- `flag` — الـ GFP flags للـ memory allocation (مثلاً `GFP_KERNEL`)

**Return:** الـ path string المخصصة، أو NULL عند الفشل. المستدعي مسؤول عن `kfree`.

---

#### `kobject_rename`

```c
int kobject_rename(struct kobject *kobj, const char *new_name);
```

بتغيّر اسم الـ kobject في sysfs عن طريق rename الـ kernfs node. بتبعت uevent بعد النجاح.

**Parameters:**
- `kobj` — الـ kobject المراد تغيير اسمه
- `new_name` — الاسم الجديد

**Return:** 0 عند النجاح، errno عند الفشل.

**Key details:**
- `__must_check`
- بتطلب reference زيادة على الـ kobject وعلى الـ parent أثناء العملية
- مش بتعمل atomic rename في الـ sysfs — ممكن يكون في race window

---

#### `kobject_move`

```c
int kobject_move(struct kobject *kobj, struct kobject *new_parent);
```

بتنقل الـ kobject من تحت parent لآخر في الـ sysfs hierarchy. بتعمل rename للـ kernfs node وبتبعت `KOBJ_MOVE` uevent مع `DEVPATH_OLD` كـ env variable.

**Parameters:**
- `kobj` — الـ kobject المراد نقله
- `new_parent` — الـ parent الجديد

**Return:** 0 عند النجاح، errno عند الفشل.

**Key details:**
- `__must_check`
- بتزود الـ refcount على `new_parent` وبتنقصه من القديم بعد النجاح

---

### المجموعة 4: Namespace & Ownership

---

#### `kobject_namespace`

```c
const void *kobject_namespace(const struct kobject *kobj);
```

بترجع الـ namespace pointer المرتبط بالـ kobject عن طريق استدعاء `ktype->namespace(kobj)`. الـ sysfs بيستخدم ده لعزل entries بين namespaces (مثلاً network interfaces مختلفة في net namespaces مختلفة).

**Parameters:**
- `kobj` — الـ kobject

**Return:** الـ namespace pointer، أو NULL لو مفيش namespace.

---

#### `kobject_get_ownership`

```c
void kobject_get_ownership(const struct kobject *kobj, kuid_t *uid, kgid_t *gid);
```

بتملأ `uid` و`gid` بالقيم المناسبة لـ sysfs entries الخاصة بالـ kobject ده. بتستدعي `ktype->get_ownership` لو موجودة، أو بترجع root:root كـ default.

**Parameters:**
- `kobj` — الـ kobject
- `uid` — pointer لـ kuid_t لتخزين الـ UID فيه
- `gid` — pointer لـ kgid_t لتخزين الـ GID فيه

---

### المجموعة 5: uevent API

الـ uevent mechanism هو الطريقة اللي بيتواصل بيها الـ kernel مع الـ userspace (خصوصاً `udev`/`systemd-udevd`) لما حاجة بتتضاف أو بتتشال أو بتتغير.

---

#### `kobject_uevent`

```c
int kobject_uevent(struct kobject *kobj, enum kobject_action action);
```

بتبعت uevent بسيط بدون environment variables إضافية. بتستدعي `kobject_uevent_env` داخلياً بـ `NULL` envp.

**Parameters:**
- `kobj` — الـ kobject المُرسِل
- `action` — الـ action (`KOBJ_ADD`, `KOBJ_REMOVE`, `KOBJ_CHANGE`, إلخ)

**Return:** 0 عند النجاح، errno عند الفشل.

**Key details:**
- لو `kobj->uevent_suppress` = 1، الـ uevent بيتعمل suppress
- `kset->uevent_ops->filter` ممكن تمنع الإرسال
- بتستخدم netlink socket لإرسال الـ uevent للـ userspace

---

#### `kobject_uevent_env`

```c
int kobject_uevent_env(struct kobject *kobj,
                        enum kobject_action action,
                        char *envp[]);
```

بتبعت uevent مع environment variables إضافية. الـ envp بيبقى array من strings بصيغة `KEY=VALUE` منتهي بـ NULL.

**Parameters:**
- `kobj` — الـ kobject المُرسِل
- `action` — الـ action
- `envp[]` — array من الـ env variables الإضافية، منتهي بـ NULL

**Return:** 0 عند النجاح، errno عند الفشل.

**Key details:**
- بتبني `struct kobj_uevent_env` على الـ stack وبتملأها بـ:
  - `ACTION=add/remove/...`
  - `DEVPATH=/sys/...`
  - `SUBSYSTEM=...`
  - الـ envp المُمرَّر
  - أي variables من `kset->uevent_ops->uevent`
- بترفع `uevent_seqnum` atomic counter لكل uevent
- بتستدعي الـ uevent helper (`/sbin/hotplug`) لو `CONFIG_UEVENT_HELPER` مفعّل

**Pseudocode:**
```
kobject_uevent_env(kobj, action, envp):
    if kobj->uevent_suppress: return 0
    if kset && kset->uevent_ops->filter(kobj) == 0: return 0

    env = alloc kobj_uevent_env
    add_uevent_var(env, "ACTION=%s", action_string)
    add_uevent_var(env, "DEVPATH=%s", path)
    add_uevent_var(env, "SUBSYSTEM=%s", subsystem)
    // append caller's envp
    // call kset->uevent_ops->uevent(kobj, env)
    atomic64_inc(&uevent_seqnum)
    add_uevent_var(env, "SEQNUM=%llu", seqnum)
    netlink_broadcast(uevent_sock, env)
    if CONFIG_UEVENT_HELPER:
        call_usermodehelper(uevent_helper, env)
```

---

#### `kobject_synth_uevent`

```c
int kobject_synth_uevent(struct kobject *kobj, const char *buf, size_t count);
```

بتسمح لـ userspace بإرسال synthetic uevent عن طريق sysfs (بالكتابة على `/sys/.../uevent`). بتعمل parse للـ buf لتستخرج الـ action وأي env variables إضافية.

**Parameters:**
- `kobj` — الـ kobject المستهدف
- `buf` — الـ string المكتوبة من userspace بصيغة `ACTION [KEY=VAL ...]`
- `count` — طول الـ buf

**Return:** عدد البايتات المُعالجة عند النجاح (== count)، errno عند الفشل.

---

#### `add_uevent_var`

```c
int add_uevent_var(struct kobj_uevent_env *env, const char *format, ...);
```

بتضيف environment variable جديدة لـ `kobj_uevent_env`. بتعمل printf للـ string في `env->buf` وبتضيف pointer فيها في `env->envp`.

**Parameters:**
- `env` — الـ uevent environment المراد الإضافة إليه
- `format, ...` — printf-style format string للـ variable

**Return:** 0 عند النجاح، `-ENOMEM` لو الـ buffer أو الـ envp pointers امتلأوا.

**Key details:**
- `__printf(2, 3)` لـ compile-time checking
- الـ buffer size هو `UEVENT_BUFFER_SIZE` (2048 bytes) والـ pointers هم `UEVENT_NUM_ENVP` (64 pointer)
- بتُستدعى من subsystems زي `driver_core` و`net` قبل ما يستدعوا `kobject_uevent_env`

---

### المجموعة 6: kset Helpers & Lookup

---

#### `to_kset`

```c
static inline struct kset *to_kset(struct kobject *kobj);
```

بتستخدم `container_of` للوصول للـ kset المُحيط بالـ kobject. Safe لو `kobj == NULL` (بترجع NULL).

---

#### `get_ktype`

```c
static inline const struct kobj_type *get_ktype(const struct kobject *kobj);
```

بترجع `kobj->ktype` مباشرةً. Accessor بسيط.

---

#### `kset_find_obj`

```c
struct kobject *kset_find_obj(struct kset *kset, const char *name);
```

بتدوّر على kobject بالاسم جوّا `kset->list` عن طريق linear scan. بتزود الـ refcount على الـ kobject اللي لاقته قبل ما ترجعه.

**Parameters:**
- `kset` — الـ kset المراد البحث فيه
- `name` — الاسم المراد إيجاده

**Return:** pointer للـ kobject مع refcount مزوّد، أو NULL لو مش موجود.

**Key details:**
- بتمسك `kset->list_lock` (spinlock) أثناء البحث
- المستدعي مسؤول عن `kobject_put` على النتيجة
- O(n) في عدد الـ kobjects — مش للاستخدام في hot paths

---

### المجموعة 7: Namespace Registration (kobject_ns)

---

#### `kobj_ns_type_register`

```c
int kobj_ns_type_register(const struct kobj_ns_type_operations *ops);
```

بتسجّل namespace type جديد في الـ kernel. حالياً بس `KOBJ_NS_TYPE_NET` مدعوم. بتتستدعي مرة واحدة أثناء الـ boot.

**Parameters:**
- `ops` — الـ operations struct الخاصة بالـ namespace type

**Return:** 0 عند النجاح، `-EINVAL` عند الفشل.

---

#### `kobj_ns_grab_current` / `kobj_ns_drop`

```c
void *kobj_ns_grab_current(enum kobj_ns_type type);
void  kobj_ns_drop(enum kobj_ns_type type, void *ns);
```

**`kobj_ns_grab_current`** بتمسك reference على الـ namespace الحالي (بالنسبة للـ task الحالي) عن طريق استدعاء `ops->grab_current_ns`.

**`kobj_ns_drop`** بتسيّب الـ reference اللي أخدتها `grab_current`.

**Key details:** بيُستخدموا من sysfs أثناء mount operations لعزل الـ visibility.

---

### Global kobjects — Well-Known Anchors

الـ kernel بيحدد عدد من الـ kobjects العالمية اللي بتخدم كـ anchor points في sysfs:

| Global kobject | sysfs Path | الاستخدام |
|---|---|---|
| `kernel_kobj` | `/sys/kernel/` | kernel parameters, debug, mm |
| `mm_kobj` | `/sys/kernel/mm/` | memory management settings |
| `hypervisor_kobj` | `/sys/hypervisor/` | hypervisor info |
| `power_kobj` | `/sys/power/` | power management |
| `firmware_kobj` | `/sys/firmware/` | firmware & ACPI |

أي subsystem أو driver عايز يضيف sysfs entries، يقدر يعمل kobject تحت الـ anchor المناسب باستخدام `kobject_create_and_add(name, kernel_kobj)` مثلاً.

---

### دورة حياة متكاملة — Putting It All Together

```
[Static Allocation]          [Dynamic Allocation]
kobject embedded in struct   kobject_create_and_add()
        ↓                           ↓
kobject_init(kobj, ktype)    (done internally)
        ↓
kobject_add(kobj, parent, name)
        ↓
    [KOBJ_ADD uevent sent → udev notified]
        ↓
    [kobject visible in /sys/...]
        ↓
    [runtime: kobject_get/put to manage refs]
        ↓
kobject_del(kobj)            ← removes from sysfs, sends KOBJ_REMOVE
        ↓
kobject_put(kobj)            ← decrements ref, if 0: ktype->release() called
        ↓
    [memory freed by release callback]
```

**القاعدة الذهبية:** كل `kobject_get` لازم يقابله `kobject_put`. وكل `kobject_add` لازم يقابله `kobject_del` قبل الـ put النهائي.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs entries

**الـ kobject** مش بيعمل entries في debugfs مباشرةً، لكن الـ kernel بيوفر infrastructure مهمة:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# عرض كل الـ kobject-related tracing
ls /sys/kernel/debug/tracing/events/kobject/

# عرض الـ kref refcount tracking (لو CONFIG_DEBUG_KOBJECT_RELEASE مفعّل)
ls /sys/kernel/debug/kobject/
```

| debugfs Path | المحتوى |
|---|---|
| `/sys/kernel/debug/tracing/events/kobject/` | tracepoints للـ kobject lifecycle |
| `/sys/kernel/debug/tracing/events/kobject/kobject_add/` | trace كل `kobject_add()` |
| `/sys/kernel/debug/tracing/events/kobject/kobject_del/` | trace كل `kobject_del()` |
| `/sys/kernel/debug/tracing/events/kobject/kobject_get/` | trace زيادة الـ refcount |
| `/sys/kernel/debug/tracing/events/kobject/kobject_put/` | trace نقصان الـ refcount |

---

#### 2. sysfs entries

**الـ sysfs** هو الواجهة الأساسية للـ kobject — كل kobject مسجّل بيظهر كـ directory:

```bash
# شوف الـ kobject hierarchy كاملة
ls -la /sys/
tree /sys/kernel/ -L 2

# الـ global kobjects المعرّفة في kobject.h
ls /sys/kernel/      # kernel_kobj
ls /sys/kernel/mm/   # mm_kobj
ls /sys/hypervisor/  # hypervisor_kobj
ls /sys/power/       # power_kobj
ls /sys/firmware/    # firmware_kobj

# تتبع kobject معيّن عن طريق اسمه
find /sys/ -name "uevent" | head -20

# قراءة uevent لـ device معين
cat /sys/bus/pci/devices/0000:00:01.0/uevent
```

**مثال على محتوى uevent:**
```
MAJOR=8
MINOR=0
DEVNAME=sda
DEVTYPE=disk
```

---

#### 3. ftrace — tracepoints وأحداث مهمة

```bash
# تفعيل كل أحداث الـ kobject
echo 1 > /sys/kernel/debug/tracing/events/kobject/enable

# أو أحداث محددة
echo 1 > /sys/kernel/debug/tracing/events/kobject/kobject_add/enable
echo 1 > /sys/kernel/debug/tracing/events/kobject/kobject_del/enable
echo 1 > /sys/kernel/debug/tracing/events/kobject/kobject_get/enable
echo 1 > /sys/kernel/debug/tracing/events/kobject/kobject_put/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي بتحقق فيها
modprobe <module>   # مثلاً

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# تصفية على kobject معيّن بالاسم
echo 'name == "my_device"' > /sys/kernel/debug/tracing/events/kobject/kobject_add/filter

# تنظيف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
```

**مثال على trace output:**
```
     kworker/0:1-42    [000] ....  1234.567890: kobject_add: my_device (parent: devices)
     kworker/0:1-42    [000] ....  1234.567900: kobject_get: my_device refcount=2
          rmmod-99    [001] ....  1235.123456: kobject_del: my_device
          rmmod-99    [001] ....  1235.123460: kobject_put: my_device refcount=1
```

---

#### 4. printk و dynamic debug

```bash
# تفعيل dynamic debug لكل kobject core
echo 'file lib/kobject.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file lib/kobject_uevent.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع stack trace
echo 'file lib/kobject.c +ps' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل debugging في الـ sysfs layer
echo 'file fs/sysfs/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ messages فوراً
dmesg -w | grep -i kobject

# أو عبر /proc/kmsg
tail -f /proc/kmsg | grep -i "kobject\|kref\|uevent"
```

**الـ log levels المهمة للـ kobject:**
```c
/* في الكود — مستويات مختلفة */
pr_debug("kobject: '%s' (%p): %s\n", kobject_name(kobj), kobj, __func__);
pr_err("kobject: '%s': uevent() failed with error %d\n", kobject_name(kobj), retval);
```

---

#### 5. Kernel Config options للـ debugging

| Config Option | الوظيفة | متى تستخدمها |
|---|---|---|
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يضيف `delayed_work` في `kobject` لاكتشاف use-after-free | دايماً في dev kernels |
| `CONFIG_DEBUG_OBJECTS_KOBJECT` | يتبع lifecycle الـ kobject عبر object debugging framework | لتتبع double-free وmissed init |
| `CONFIG_DEBUG_OBJECTS` | الـ base object debugging framework | شرط لـ CONFIG_DEBUG_OBJECTS_KOBJECT |
| `CONFIG_KASAN` | اكتشاف use-after-free وout-of-bounds في الـ kobject | أقوى أداة لـ memory bugs |
| `CONFIG_LOCKDEP` | تحليل deadlocks في `kset->list_lock` | لمشاكل الـ locking |
| `CONFIG_PROVE_LOCKING` | إثبات صحة ترتيب الـ locks | مع LOCKDEP |
| `CONFIG_REFCOUNT_FULL` | detection كامل لـ refcount overflow/underflow | لمشاكل الـ kref |
| `CONFIG_UBSAN` | اكتشاف undefined behavior | عام |
| `CONFIG_KCSAN` | اكتشاف data races في الـ kobject operations | multi-threaded bugs |

```bash
# تحقق من الـ config المفعّل في الكيرنل الحالي
grep -E 'CONFIG_DEBUG_KOBJECT|CONFIG_DEBUG_OBJECTS|CONFIG_KASAN|CONFIG_REFCOUNT' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ subsystem

```bash
# udevadm — أهم أداة لتتبع الـ uevents المرسلة من kobject_uevent()
udevadm monitor --kernel --udev

# مشاهدة تفاصيل الـ uevent
udevadm monitor --kernel --property

# إجبار إرسال uevent يدوياً لـ device موجود
echo add > /sys/bus/usb/devices/1-1/uevent

# kobject_synth_uevent — synthetic uevents
echo "add" > /sys/class/net/eth0/uevent

# تتبع الـ netlink socket للـ uevents (NETLINK_KOBJECT_UEVENT)
# الـ uevents بتتبعث على group 1
python3 -c "
import socket, struct
sock = socket.socket(socket.AF_NETLINK, socket.SOCK_RAW, 15)
sock.bind((0, 1))
while True:
    data = sock.recv(65536)
    print(data.decode('utf-8', errors='replace'))
"

# قراءة الـ uevent sequence number (uevent_seqnum في kobject.h)
cat /sys/kernel/uevent_seqnum

# عرض شجرة الـ sysfs بشكل مفيد
systemd-cgls  # للـ cgroup kobjects
lsusb -t      # USB kobject tree
lspci -t      # PCI kobject tree
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `kobject: '%s': kobject_add_internal() failed with error -EEXIST` | kobject بنفس الاسم موجود بالفعل في نفس الـ parent | تأكد من uniqueness الاسم، استخدم `kobject_rename()` |
| `kobject: '%s': fill_kobj_path() failed with error %d` | فشل تخصيص memory لمسار الـ sysfs | مشكلة memory، افحص `dmesg` لـ OOM |
| `kobject_add_internal: parent is not initialized` | الـ parent kobject لم يمر بـ `kobject_init()` | اتأكد من init قبل add |
| `WARNING: kobject bug: ops missing` | `kobj_type` بدون `release` function | أضف `release` callback دايماً |
| `kobject: '%s': does not have a release() function` | نفس الفوق — سبب use-after-free | أضف `release` function |
| `uevent: failed to send uevent` | فشل إرسال netlink message | افحص netlink socket، ممكن buffer ممتلي |
| `kobject_uevent_env: uevent() failed with error %d` | الـ `kset_uevent_ops->uevent` callback رجّع error | افحص الـ uevent callback في الـ kset |
| `KOBJECT_ADD event for '%s' already sent` | محاولة إرسال ADD uevent مرتين | مشكلة في lifecycle management |
| `kobject: '%s' (%p): kobject_init called on uninitialized object` | double init | تأكد من init مرة واحدة فقط |
| `refcount_t: underflow; use-after-free` | `kobject_put()` أكثر من `kobject_get()` | تتبع ownership بدقة |
| `refcount_t: addition on zero; use-after-free` | `kobject_get()` على object محذوف | استخدم `kobject_get_unless_zero()` |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في kobject_add_internal() — تحقق من state */
WARN_ON(!kobj->state_initialized);
WARN_ON(kobj->state_in_sysfs);

/* في release function — تحقق من refcount */
WARN_ON(kref_read(&kobj->kref) != 0);

/* في kobject_put() — قبل الـ release */
WARN_ON_ONCE(kref_read(&kobj->kref) == 0);

/* تتبع من أين اتعمل kobject_get() */
static struct kobject *my_kobject_get(struct kobject *kobj)
{
    if (!kobj)
        return NULL;
    WARN_ON(kref_read(&kobj->kref) == 0); /* already freed! */
    return kobject_get(kobj);
}

/* في kset operations — تحقق من consistency */
WARN_ON(!kobj->kset);
WARN_ON(kobj->kset->kobj.state_initialized == 0);

/* اكتشاف الـ use-after-free عن طريق poison */
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
/* الـ delayed_work في struct kobject بتأخر الـ release
   عشان تكتشف لو في access بعد الـ free */
#endif
```

---

### Hardware Level

---

#### 1. التحقق من تطابق hardware state مع kernel state

**الـ kobject** مش بيتعامل مع hardware مباشرةً — هو abstraction layer. لكن لو الـ kobject يمثّل device حقيقي:

```bash
# تحقق إن الـ device موجود فعلاً في hardware
lspci -vvv | grep -A 20 "My Device"
lsusb -v | grep -A 20 "My Device"

# تطابق الـ kobject path مع hardware path
udevadm info --path=/sys/bus/pci/devices/0000:00:1f.2
udevadm info --attribute-walk --path=/sys/bus/pci/devices/0000:00:1f.2

# تحقق من الـ state bits في struct kobject
# state_initialized=1 → kobject_init() اتعمل
# state_in_sysfs=1    → kobject_add() نجح
# state_add_uevent_sent=1 → uevent ADD اتبعت
# state_remove_uevent_sent=1 → uevent REMOVE اتبعت

# قراءة الـ uevent_suppress flag
cat /sys/devices/platform/my_device/uevent
```

---

#### 2. Register dump techniques

**الـ kobject** نفسه software-only، بس لو بتحقق في device خلف الـ kobject:

```bash
# قراءة registers باستخدام devmem2
# مثال: قراءة register على عنوان 0xFEDC0000
devmem2 0xFEDC0000 w

# أو باستخدام /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0xFEDC0000/4)) 2>/dev/null | xxd

# io tool لـ x86 I/O ports
# sudo apt install iotools
io inl 0x3F8

# قراءة PCI config space لـ device ورا kobject
setpci -s 0000:00:1f.2 COMMAND
setpci -s 0000:00:1f.2 0:4

# dump كامل لـ PCI config space
lspci -xxx -s 0000:00:1f.2

# لـ platform devices — قراءة memory-mapped registers
# افترض base = 0xFF000000, size = 0x1000
sudo hexdump -C /dev/mem 2>/dev/null | grep -A 4 "ff000"
```

---

#### 3. Logic analyzer / Oscilloscope tips

بما إن الـ kobject بيتحكم في الـ uevents والـ sysfs، مش مباشرةً في hardware signals، الـ tracing المهم هو:

- **I2C/SPI probe timing**: لو الـ kobject بيتعمل أثناء probe، قيس الـ timing بين أول request وآخر register write
- **GPIO state**: لو الـ device بيعمل GPIO toggle عند KOBJ_ADD، استخدم logic analyzer على الـ GPIO pin
- **Power sequences**: بعض الـ kobjects بتتعمل أثناء power-on sequences — قيس الـ VCC rails مع الـ timestamps من dmesg

```
Timeline مثالي على logic analyzer:
                    kobject_add()
                         |
GPIO_RESET ─────────┐    |    ┌──────────────
                    └────┘

I2C SCL  ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─────────────────
           └─┘ └─┘ └─┘ └─┘

uevent   ──────────────────────┐ ┌───────────
 (GPIO)                        └─┘
```

---

#### 4. Hardware issues → kernel log patterns

| المشكلة | Pattern في kernel log | التشخيص |
|---|---|---|
| Device مش موجود فعلياً | `kobject_add failed: -ENODEV` | افحص connections، power |
| Device timeout أثناء probe | `kobject: '%s': add failed` + `timeout` قبلها | افحص I2C/SPI signals |
| Device address conflict | `kobject_add: -EEXIST` | عندك devices بنفس الاسم |
| Firmware load failure | `kobject_uevent: failed` + `firmware: failed` | افحص `/lib/firmware/` |
| Memory pressure | `kobject_add: -ENOMEM` | افحص `free -m`، OOM في dmesg |
| Probe race condition | `kobject_add` ثم `kobject_del` فوراً | driver probe فاشل، افحص dmesg كامل |
| uevent not received | `state_add_uevent_sent=0` بعد add | netlink socket مش شغال، افحص udevd |

---

#### 5. Device Tree debugging

```bash
# تحقق إن الـ DT node موجود وصح
dtc -I fs -O dts /sys/firmware/devicetree/base/ > /tmp/current.dts 2>/dev/null
grep -A 20 "my_device" /tmp/current.dts

# تحقق من compatible string
cat /sys/firmware/devicetree/base/soc/my_device/compatible

# تحقق من الـ status
cat /sys/firmware/devicetree/base/soc/my_device/status
# يجب يكون "okay" عشان يتعمل probe

# مقارنة DT مع الـ sysfs kobject
# لو DT موجود لكن kobject مش موجود في sysfs → probe فاشل
ls /sys/bus/platform/devices/ | grep my_device

# معرفة الـ driver المرتبط بالـ kobject
cat /sys/bus/platform/devices/my_device/driver/module/name

# تحقق من الـ reg property يطابق الـ actual hardware address
python3 -c "
import struct
with open('/sys/firmware/devicetree/base/soc/my_device/reg', 'rb') as f:
    data = f.read()
    # big-endian u32 pairs: (base, size)
    for i in range(0, len(data), 8):
        base, size = struct.unpack('>II', data[i:i+8])
        print(f'Base: 0x{base:08X}, Size: 0x{size:08X}')
"

# فحص الـ interrupts في DT
cat /sys/firmware/devicetree/base/soc/my_device/interrupts | xxd

# ovftrace — تتبع DT matching مع drivers
echo 1 > /sys/kernel/debug/tracing/events/of/enable
dmesg | grep -E "of: probe|compatible"
```

---

### Practical Commands

---

#### جدول الأوامر الجاهزة للنسخ

```bash
# ============================================================
# 1. عرض الـ kobject hierarchy كاملة
# ============================================================
find /sys/ -name "uevent" -exec dirname {} \; 2>/dev/null | head -50

# ============================================================
# 2. مراقبة كل الـ uevents في real-time
# ============================================================
udevadm monitor --kernel --property 2>&1 | tee /tmp/uevent_log.txt

# ============================================================
# 3. تفعيل كل kobject tracing
# ============================================================
echo 1 > /sys/kernel/debug/tracing/events/kobject/enable && \
echo 1 > /sys/kernel/debug/tracing/tracing_on && \
echo "Tracing started. Press Ctrl+C to stop..." && \
trap 'echo 0 > /sys/kernel/debug/tracing/tracing_on; cat /sys/kernel/debug/tracing/trace' INT && \
read

# ============================================================
# 4. فحص refcount لـ kobject معيّن (عبر crash أو gdb)
# ============================================================
# في crash utility:
# crash> struct kobject <address>
# crash> p ((struct kobject *)<address>)->kref.refcount.refs.counter

# ============================================================
# 5. dynamic debug لـ kobject core كاملاً
# ============================================================
for f in lib/kobject.c lib/kobject_uevent.c fs/sysfs/dir.c; do
    echo "file $f +pflmt" > /sys/kernel/debug/dynamic_debug/control
done
dmesg -w

# ============================================================
# 6. إيجاد كل الـ kobjects اللي uevent_suppress=1
# ============================================================
find /sys/ -name "uevent" 2>/dev/null | while read f; do
    dir=$(dirname "$f")
    # لو الـ uevent file فاضي = suppressed
    [ ! -s "$f" ] && echo "Suppressed: $dir"
done

# ============================================================
# 7. اختبار kobject_synth_uevent
# ============================================================
# إرسال synthetic uevent لـ device
echo "change INTERFACE=eth0 IFINDEX=2" > /sys/class/net/eth0/uevent
udevadm monitor --kernel &
echo "change" > /sys/class/net/eth0/uevent
wait

# ============================================================
# 8. فحص CONFIG_DEBUG_KOBJECT_RELEASE
# ============================================================
grep CONFIG_DEBUG_KOBJECT_RELEASE /boot/config-$(uname -r)
# لو =y: الـ release بتتأخر عشان تكشف use-after-free
# dmesg بيظهر: "kobject_release, parent X, releasing."

# ============================================================
# 9. تتبع kref leaks باستخدام kmemleak
# ============================================================
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep -i kobject

# ============================================================
# 10. فحص الـ sysfs ops لـ kobject معيّن
# ============================================================
# في /proc/kallsyms إيجاد عنوان الـ kobj_type
grep "my_driver.*ktype\|kobj_type" /proc/kallsyms
```

---

#### تفسير الـ output

**مثال على udevadm monitor output:**
```
KERNEL[1234.5678] add      /devices/platform/my_device (platform)
ACTION=add
DEVPATH=/devices/platform/my_device
SUBSYSTEM=platform
MODALIAS=platform:my_device
SEQNUM=4321
```

التفسير:
- **KERNEL[timestamp]**: الوقت بالثواني من boot
- **add**: الـ action = `KOBJ_ADD`
- **/devices/platform/my_device**: الـ `kobject_get_path()` للـ kobject
- **SEQNUM**: قيمة `uevent_seqnum` الـ atomic counter

**مثال على ftrace kobject output:**
```
     kworker/u8:2-156   [003] ...1  5678.901234: kobject_add: my_device (parent: platform)
     kworker/u8:2-156   [003] ...1  5678.901240: kobject_get: my_device refcount=2
          systemd-1-1   [000] ...1  5678.950000: kobject_put: my_device refcount=1
```

التفسير:
- الـ refcount بدأ بـ 1 بعد `kobject_init()`
- زاد لـ 2 بعد عملية `kobject_get()`
- رجع لـ 1 بعد `kobject_put()`
- لما يوصل لـ 0 → بيتعمل call للـ `kobj_type->release()`
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Kernel Panic أثناء Hotplug لـ USB على بورد RK3562

#### العنوان
**Use-After-Free بسبب `kobject_put` ناقصة في USB driver مخصص**

#### السياق
شركة بتعمل industrial gateway على RK3562. البورد بتشغل Linux 6.1، وفيه USB hub بيتوصل بيه USB-to-Serial adapters بشكل متكرر (hotplug). الـ driver مخصص وبيعمل `kobject_create_and_add` لكل device جديد عشان يعمل sysfs entry.

#### المشكلة
بعد تقريبًا 50 plug/unplug cycle، الكيرنل بيعمل panic بـ:

```
BUG: KASAN: use-after-free in kobject_put+0x2c/0x80
```

#### التحليل
الـ `struct kobject` معرفة في `kobject.h`:

```c
struct kobject {
    const char      *name;
    struct list_head entry;
    struct kobject  *parent;
    struct kset     *kset;
    const struct kobj_type *ktype;
    struct kernfs_node *sd; /* sysfs directory entry */
    struct kref     kref;   // <-- reference count هنا
    unsigned int state_initialized:1;
    unsigned int state_in_sysfs:1;
    // ...
};
```

الـ `kref` جوه `kobject` هو `struct kref` من `kref.h` اللي بيحتوي `refcount_t`. كل `kobject_get()` بتزود الـ refcount، وكل `kobject_put()` بتنقصه — لو وصل صفر بيتنادى `kobj_type->release`.

المشكلة: الـ driver بيعمل `kobject_get()` في thread واحد لما بيقرأ sysfs attribute، لكن لما بيجي disconnect، بيعمل `kobject_del()` ثم `kobject_put()` مرة واحدة بس — بينما في thread تاني لسه بيحتفظ بـ reference ماعملش `kobject_put()` عليها.

```c
// الكود الغلط في الـ driver
void my_device_disconnect(struct my_dev *dev)
{
    kobject_del(&dev->kobj);   // بيشيل من sysfs
    kobject_put(&dev->kobj);   // بس لو في thread تاني عامل get، refcount != 0
    kfree(dev);                // BUG: free قبل ما refcount يوصل 0
}
```

#### الحل
لازم `kfree` يتحط جوه `release` callback اللي بتتنادى أوتوماتيك لما refcount يوصل صفر:

```c
static struct kobj_type my_ktype = {
    .release   = my_kobject_release,  // هنا بس نعمل kfree
    .sysfs_ops = &kobj_sysfs_ops,
};

static void my_kobject_release(struct kobject *kobj)
{
    struct my_dev *dev = container_of(kobj, struct my_dev, kobj);
    kfree(dev); // آمن لأن refcount وصل صفر
}

void my_device_disconnect(struct my_dev *dev)
{
    kobject_del(&dev->kobj);
    kobject_put(&dev->kobj); // لو كل thread عامل put، هيتنادى release
}
```

```bash
# تأكد إن الـ refcount بيتصرف صح
echo 1 > /sys/kernel/debug/kmemleak/scan
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
**`kfree` مكانها الصح هو جوه `kobj_type->release`** — مش بعد `kobject_put` مباشرة. الـ `kref` في `kobject.h` بيضمن إن الـ release مش بتتنادى إلا لما آخر reference تتحرر.

---

### السيناريو 2: sysfs Attribute مش بتظهر في STM32MP1 Industrial Board

#### العنوان
**`state_in_sysfs` بيفضل صفر بسبب `kobject_add` فاشلة صامتة**

#### السياق
فريق بيعمل bring-up لـ custom board على STM32MP1 (Cortex-A7). بيعملوا driver لـ temperature sensor متصل بـ I2C، وعايزين يعرضوا readings عبر sysfs تحت `/sys/devices/platform/`.

#### المشكلة
الـ driver بيلود بدون errors، بس `/sys/devices/platform/my_sensor/` مش موجودة. الـ `ls /sys/devices/platform/` مش بيظهرها خالص.

#### التحليل
الـ `struct kobject` بيه flag مهم:

```c
struct kobject {
    // ...
    unsigned int state_initialized:1;   // اتعمله init؟
    unsigned int state_in_sysfs:1;      // اتضاف لـ sysfs؟
    unsigned int state_add_uevent_sent:1;
    // ...
};
```

الـ `state_in_sysfs` بيتبقى صفر لو `kobject_add()` فشلت أو اتنسيت. الـ engineer نسي يتحقق من return value:

```c
// كود غلط — ignore return value
kobject_init(&dev->kobj, &sensor_ktype);
kobject_add(&dev->kobj, &platform_bus, "my_sensor"); // return ignored!
// بيكمل وكأن كل حاجة تمام
```

الـ `kobject_add` معمارها في `kobject.h`:

```c
__printf(3, 4) __must_check int kobject_add(struct kobject *kobj,
                                             struct kobject *parent,
                                             const char *fmt, ...);
```

الـ `__must_check` بيخلي compiler يحذر، بس لو `-Werror` ماشيش أو الحذر اتجاهل.

السبب الفعلي: اسم الـ kobject فيه `/` — الـ sensor اسمه `"i2c/temp_sensor"` — وده بيخلي `kobject_add` ترجع `-EINVAL` لأن `/` ممنوعة في أسماء sysfs.

#### الحل
```c
// صح: تحقق من return value
ret = kobject_init_and_add(&dev->kobj, &sensor_ktype,
                           &platform_bus, "i2c_temp_sensor"); // بدون /
if (ret) {
    pr_err("kobject_add failed: %d\n", ret);
    kobject_put(&dev->kobj);
    return ret;
}

// تحقق يدوي إن الـ flag اتضبط
pr_debug("state_in_sysfs = %u\n", dev->kobj.state_in_sysfs);
```

```bash
# debug على البورد
dmesg | grep "kobject"
ls /sys/devices/platform/ | grep sensor
```

#### الدرس المستفاد
الـ `__must_check` على `kobject_add()` موجود لسبب — **دايمًا تحقق من return value**. وأسماء الـ kobjects ممنوع يكون فيها `/` أو أي character ممنوعة في filesystem paths.

---

### السيناريو 3: udev مش بيشغل Rules على Android TV Box بـ Allwinner H616

#### العنوان
**`uevent_suppress` مضبوط بشكل خاطئ بيمنع udev من شغل rules**

#### السياق
منتج Android TV box بيشتغل على Allwinner H616. الـ team بيضيف HDMI CEC driver. المفروض لما الـ CEC device يظهر، udev rule تشغل script بيحدد الـ TV remote mapping. بس الـ script مش بتتشغل.

#### المشكلة
الـ udev rules موجودة وصح، بس `udevadm monitor` مش بيشوف أي uevent لما الـ CEC device يتسجل.

#### التحليل
في `kobject.h`، الـ `struct kobject` بيه:

```c
struct kobject {
    // ...
    unsigned int uevent_suppress:1;  // <-- لو 1، مفيش uevent بيتبعت
};
```

الـ `kset_uevent_ops` بيه `filter` callback:

```c
struct kset_uevent_ops {
    int (* const filter)(const struct kobject *kobj);  // لو رجع 0، uevent اتمنع
    const char *(* const name)(const struct kobject *kobj);
    int (* const uevent)(const struct kobject *kobj, struct kobj_uevent_env *env);
};
```

الـ engineer في الـ CEC kset عمل:

```c
static int cec_uevent_filter(const struct kobject *kobj)
{
    // كان قصده يفلتر events معينة، بس غلط في الـ logic
    return 0; // بيمنع كل events!
}
```

بالإضافة إلى إن في مكان تاني في الـ init code:

```c
dev->kobj.uevent_suppress = 1; // نسي يرجعه 0 بعد الـ initialization
```

#### الحل
```c
// إصلاح الـ filter — لازم يرجع 1 عشان يسمح بالـ uevent
static int cec_uevent_filter(const struct kobject *kobj)
{
    struct cec_device *cec = container_of(kobj, struct cec_device, kobj);
    // امنع فقط devices اللي مش initialized
    return cec->initialized ? 1 : 0;
}

// وبعد الـ init، تأكد إن suppress = 0
dev->kobj.uevent_suppress = 0;

// ثم أبعت uevent يدوي لو محتاج
kobject_uevent(&dev->kobj, KOBJ_ADD);
```

```bash
# تشخيص
udevadm monitor --kernel --udev
udevadm test /sys/class/cec/cec0
cat /sys/class/cec/cec0/uevent
```

#### الدرس المستفاد
**`uevent_suppress` و `filter` callback** اللي بترجع 0 بيعملوا نفس الحاجة — بيمنعوا uevents. الفرق: `uevent_suppress` بيأثر على kobject واحد، و`filter` بيأثر على كل الـ kset. لازم تتحقق من الاتنين لو udev مش بيستجاوب.

---

### السيناريو 4: Memory Leak في IoT Sensor Node على AM62x

#### العنوان
**`kobject_create_and_add` بدون `kobject_put` بتسبب leak في long-running system**

#### السياق
IoT sensor node بيشتغل على TI AM62x، بيعمل sensor readings كل دقيقة. الـ firmware بتعمل dynamic kobjects عشان تعرض sensor metadata في sysfs، وبتعمل `kobject_del` لما تخلص. الجهاز المفروض يشتغل لسنوات.

#### المشكلة
بعد أسابيع، `kmemleak` بيبدأ يبلغ عن leaked objects. الـ `/proc/meminfo` بيبين تناقص تدريجي في `MemFree`.

#### التحليل
`kobject_create_and_add` معرفة في `kobject.h`:

```c
struct kobject * __must_check kobject_create_and_add(const char *name,
                                                      struct kobject *parent);
```

الـ function دي بتعمل:
1. `kzalloc` لـ kobject جديد
2. `kobject_init` بـ `dynamic_kobj_ktype`
3. `kobject_add`

وترجع pointer للـ kobject الجديد. الـ kobject ده عنده internal `kref` مبدئه 1.

الكود الغلط:

```c
void publish_sensor_data(const char *sensor_name, struct kobject *parent)
{
    struct kobject *kobj;

    kobj = kobject_create_and_add(sensor_name, parent);
    if (!kobj)
        return;

    // ... نشر البيانات ...

    kobject_del(kobj); // بيشيله من sysfs، بس refcount لسه 1
    // نسي kobject_put(kobj)! — memory leak هنا
}
```

`kobject_del` بتشيل الـ kobject من sysfs (بتصفر `state_in_sysfs`)، بس مش بتنقص الـ refcount. لازم `kobject_put` بعدها عشان ينزل الـ refcount لصفر ويتنادى `release` ويتحرر الـ memory.

#### الحل
```c
void publish_sensor_data(const char *sensor_name, struct kobject *parent)
{
    struct kobject *kobj;

    kobj = kobject_create_and_add(sensor_name, parent);
    if (!kobj)
        return;

    // ... نشر البيانات ...

    kobject_del(kobj);
    kobject_put(kobj); // لازم دايمًا بعد kobject_del للـ dynamic kobjects
}
```

```bash
# كشف الـ leak على البورد
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | head -50

# أو بـ valgrind-style kernel tool
cat /proc/slabinfo | grep kobject
```

#### الدرس المستفاد
**`kobject_del` ≠ free**. الـ pair الصح دايمًا:
- `kobject_create_and_add` → يقابله `kobject_del` + `kobject_put`
- `kobject_init_and_add` → يقابله `kobject_del` + `kobject_put`

الـ `kobject_put` هي اللي بتحرر الـ memory في النهاية عبر `release` callback.

---

### السيناريو 5: Race Condition في Automotive ECU على i.MX8

#### العنوان
**`kobject_get_unless_zero` ضرورية لتجنب race بين lookup و teardown**

#### السياق
Automotive ECU على NXP i.MX8QM بيشتغل بـ AGL (Automotive Grade Linux). فيه driver بيدير CAN interfaces، وكل interface عنده `struct kobject` بيتبعت events عليه. الـ system بيعمل dynamic add/remove للـ CAN interfaces بناءً على network configuration.

#### المشكلة
بشكل عشوائي، الـ system بيهنج مع:

```
general protection fault in kobject_get+0x18/0x30
```

بيحصل لما في thread بيعمل lookup لـ kobject بالاسم، وفي نفس الوقت thread تاني بيعمل teardown.

#### التحليل
الكود المشكلة:

```c
// Thread A: يدور على kobject بالاسم ويشتغل عليه
struct kobject *kobj = kset_find_obj(can_kset, "can0");
if (kobj) {
    // Race window هنا! Thread B ممكن يعمل kobject_put ويحذف الـ kobj
    do_something_with(kobj);
    kobject_put(kobj); // ده ممكن يبقى use-after-free
}
```

```c
// Thread B: بيشيل الـ interface
kobject_del(can_kobj);
kobject_put(can_kobj); // refcount → 0 → free
```

الـ `kset_find_obj` بتعمل `kobject_get` داخليًا، بس لو الـ refcount وصل صفر قبل الـ `get`، بيبقى مشكلة.

الحل الصح هو استخدام `kobject_get_unless_zero` اللي معرفة في `kobject.h`:

```c
struct kobject * __must_check kobject_get_unless_zero(struct kobject *kobj);
```

الـ function دي atomic — بتزود الـ refcount بس لو مش صفر خالص:

```c
// kref.h: المنطق التحتي
static inline int __must_check kref_get_unless_zero(struct kref *kref)
{
    return refcount_inc_not_zero(&kref->refcount); // atomic operation
}
```

#### الحل
```c
// Thread A: آمن من الـ race
struct kobject *kobj;
struct kobject *tmp;

// أولًا دور على الـ kobj بدون زيادة refcount
spin_lock(&can_kset->list_lock);
list_for_each_entry(tmp, &can_kset->list, entry) {
    if (kobject_name(tmp) && strcmp(kobject_name(tmp), "can0") == 0) {
        // atomic: زود refcount بس لو مش صفر
        kobj = kobject_get_unless_zero(tmp);
        break;
    }
}
spin_unlock(&can_kset->list_lock);

if (kobj) {
    do_something_with(kobj); // آمن — refcount >= 1
    kobject_put(kobj);       // آمن — ننزله بعد ما خلصنا
}
```

```bash
# اكتشاف الـ race conditions
CONFIG_DEBUG_KOBJECT_RELEASE=y  # في kernel config
# بيأخر الـ release عشان يكشف use-after-free

# أو بـ KASAN
CONFIG_KASAN=y
dmesg | grep "KASAN"
```

#### الدرس المستفاد
في multithreaded environments، **`kobject_get_unless_zero` هي الطريقة الآمنة للـ lookup** — مش `kobject_get` العادية. الفرق إنها atomic مع check على الـ refcount، فمش ممكن تزود refcount لـ object اتحذف فعلًا. ده مهم جدًا في automotive systems اللي فيها real-time constraints وmultiple threads بتشتغل على نفس devices.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لأي حاجة بتخص kernel internals — الـ kobject مش استثنا.

| المقال | الوصف |
|--------|-------|
| [The zen of kobjects](https://lwn.net/Articles/51437/) | أول مقال بيشرح فلسفة الـ kobject وفكرة الـ reference counting — نقطة البداية الحقيقية |
| [kobjects and sysfs](https://lwn.net/Articles/54651/) | يربط الـ kobject بالـ sysfs ويشرح إزاي كل kobject بيتحول لـ directory في `/sys` |
| [kobjects and hotplug events](https://lwn.net/Articles/52621/) | يشرح الـ kset hotplug operations وإزاي `/sbin/hotplug` بيتنادى لما kobject يتحذف أو يتضاف |
| [New kobject/kset/ktype documentation and example code](https://lwn.net/Articles/260139/) | توثيق محدّث مع sample code عملي — Greg Kroah-Hartman كتبه بنفسه |
| [kobject/kset/ktype documentation and example code updated](https://lwn.net/Articles/263200/) | نسخة أحدث من التوثيق بعد تعديلات على الـ API |
| [kref, a tiny, sane, reference count object](https://lwn.net/Articles/75659/) | بيشرح الـ `struct kref` اللي الـ kobject بيعتمد عليه داخليًا |
| [The debut of kref](https://lwn.net/Articles/75920/) | إزاي الـ kref ظهر في kernel 2.6.5 وعلاقته بالـ kobject |
| [Linux kernel design patterns - part 1](https://lwn.net/Articles/336224/) | بيشرح design patterns زي container_of اللي الـ kobject بيعتمد عليها |
| [kobject: support namespace aware udev](https://lwn.net/Articles/656941/) | patch أضاف دعم الـ namespaces للـ kobject uevents |

---

### التوثيق الرسمي في kernel

#### ملفات `Documentation/`

```
Documentation/core-api/kobject.rst       ← الوثيقة الأساسية (اقرأها قبل أي حاجة تانية)
Documentation/core-api/kref.rst          ← توثيق struct kref
Documentation/filesystems/sysfs.rst      ← العلاقة بين kobject و sysfs
Documentation/driver-api/driver-model/   ← driver model المبني فوق kobject
```

- **الـ `kobject.rst`** بيقول صراحةً:

  > "Please read this before using the kobject interface, ESPECIALLY the parts about reference counts and object destructors."

- الـ online version على kernel.org:
  - [Everything you never wanted to know about kobjects](https://www.kernel.org/doc/html/v5.15/core-api/kobject.html)
  - [Adding reference counters (krefs)](https://docs.kernel.org/core-api/kref.html)
  - [kobject.txt (older version)](https://www.kernel.org/doc/Documentation/kobject.txt)

---

### Commits مهمة في تاريخ الـ kobject

الـ kobject اتكتب أصلًا من Patrick Mochel في 2002–2003، وبعدين Greg Kroah-Hartman استلم المسؤولية من 2006.

```bash
# عشان تشوف تاريخ الملف في git
git log --follow include/linux/kobject.h

# commit اللي أضاف sample kobject code
# [PATCH 179/196] kobject: add sample code for how to use kobjects
# https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg255670.html

# cleanup كبير من Greg KH
# [PATCH 098/196] kobject: clean up rpadlpar horrid sysfs abuse
# https://lkml.kernel.org/lkml/1201246425-5058-19-git-send-email-gregkh@suse.de/
```

- [kobject auto-cleanup on final unref](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg255671.html) — patch مهم سمح بـ auto-cleanup لما refcount توصل 0
- [sysfs: fix kobject refcount races](https://lore.kernel.org/all/YPgFVRAMQ9hN3dnB@kroah.com/T/) — fix لـ race condition بين removal وـ attribute access

---

### نقاشات Mailing List

| النقاش | الرابط |
|--------|--------|
| `[PATCH] kobject: send KOBJ_REMOVE uevent when object removed from sysfs` | [mail-archive.com](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2171076.html) |
| `[PATCH v4] sysfs: fix kobject refcount to address races` | [lore.kernel.org](https://lore.kernel.org/all/YPgFVRAMQ9hN3dnB@kroah.com/T/) |
| `[PATCH] sysfs: Add sysfs_emit to replace sprintf` | [mail-archive.com](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2287361.html) |
| Kobjects and sysfs — USENIX 2004 presentation | [usenix.org](https://www.usenix.org/conference/2004-linux-kernel-developers-summit/presentation/kobjects-and-sysfs) |

---

### كتب مهمة

#### Linux Device Drivers (LDD3)

**الـ LDD3** هو أكتر كتاب عملي بيشرح kobject بأمثلة حقيقية:

- **Chapter 14: The Linux Device Model** — كامل مخصص للـ kobject/kset/ktype/sysfs
- PDF مجاني رسمي من O'Reilly على lwn.net: [LDD3 Chapter 14 PDF](https://static.lwn.net/images/pdf/LDD3/ch14.pdf)
- الكتاب كاملًا: [free-electrons.com/doc/books/ldd3.pdf](https://lwn.net/Kernel/LDD3/)

```
LDD3, Chapter 14:
  - 14.1  The Kobject
  - 14.2  Kobject Hierarchies, Ksets, and Subsystems
  - 14.3  Low-Level Sysfs Operations
  - 14.4  Default Attributes
  - 14.5  Nondefault Attributes
  - 14.6  Binary Attributes
  - 14.7  Symbolic Links
  - 14.8  Hotplug Event Generation
  - 14.9  Buses, Devices, and Drivers
```

#### Linux Kernel Development — Robert Love (3rd Edition)

- **Chapter 17: Devices and Modules** — بيشرح الـ device model وعلاقته بالـ kobject
- بيغطي الـ sysfs وكيفية ربط struct بـ kobject باستخدام `container_of`
- مستوى أقل تفصيلًا من LDD3 لكنه أوضح من ناحية الـ big picture

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **Chapter 8: Device Driver Basics** — بيتكلم عن الـ sysfs interface من perspective الـ embedded developer
- مفيد لفهم إزاي udev و kobject uevents بيشتغلوا على embedded systems

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- **Chapter 6: Device Drivers** — تغطية عميقة للـ kobject internals وكيفية بناء الـ device model

---

### روابط kernelnewbies.org و elinux.org

| الموقع | الصفحة | المحتوى |
|--------|--------|---------|
| kernelnewbies.org | [KernelProjects/usermode-helper](https://kernelnewbies.org/KernelProjects/usermode-helper) | يشرح kobject uevents وكيفية استخدام `/sbin/hotplug` |
| kernelnewbies.org | [InternalKernelDataTypes](https://kernelnewbies.org/InternalKernelDataTypes) | types مهمة زي kref وlist_head المستخدمة في kobject |
| kernelnewbies.org | [Documents](https://kernelnewbies.org/Documents) | فهرس كامل للوثائق الـ kernel المتاحة |
| elinux.org | [EBC Flashing an LED](https://elinux.org/EBC_Flashing_an_LED) | مثال عملي لاستخدام kobject/sysfs على BeagleBone |
| win.tue.nl | [Sysfs and kobjects](https://www.win.tue.nl/~aeb/linux/lk/lk-13.html) | شرح تقني مفصل لـ sysfs وعلاقتها بالـ kobject |

---

### Source Code للرجوع إليه

```bash
# الملفات الأساسية في kernel tree
include/linux/kobject.h          # التعريفات الرئيسية
include/linux/kref.h             # struct kref
include/linux/kobject_ns.h       # namespace support
lib/kobject.c                    # implementation الأساسي
lib/kobject_uevent.c             # uevent generation
lib/kref.c                       # kref implementation
fs/sysfs/                        # sysfs filesystem
drivers/base/core.c              # device model built on kobject

# مثال رسمي موجود في kernel tree
samples/kobject/kobject-example.c
samples/kobject/kset-example.c
```

---

### Search Terms للبحث عن مزيد من المعلومات

```
# للبحث في المصادر التقنية
"linux kobject reference counting"
"kobject kset ktype explained"
"linux sysfs kobject tutorial"
"kobject uevent udev kernel"
"kobject_put release function kernel"
"kref vs kobject linux kernel"
"linux driver model kobject hierarchy"
"kobject_init_and_add example"
"struct kobj_type release callback"
"linux kernel /sys directory kobject"

# للبحث في git log
git log --all --oneline -- lib/kobject.c
git log --all --oneline -- include/linux/kobject.h
git log --grep="kobject" --oneline v5.15..v6.6
```
## Phase 8: Writing simple module

### الفكرة

**`kobject_uevent`** هي الدالة المسؤولة عن إرسال أي **uevent** (زي `KOBJ_ADD` أو `KOBJ_REMOVE`) من الـ kernel للـ userspace عبر netlink. كل مرة بتتضاف device أو بتتشال، بتعدي من هنا — ده بيخليها نقطة مثيرة للـ hook بالـ **kprobe**.

هنستخدم `kprobe` عشان:
- الدالة مش `static` — ممكن نلاقيها بالاسم.
- بتتنادى كتير بأحداث حقيقية (USB، network، block devices).
- بتوفر معلومات مفيدة: اسم الـ kobject + نوع الـ action.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_uevent.c
 *
 * Hooks kobject_uevent() via kprobe and logs the kobject name
 * and action type every time a uevent is fired.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/kobject.h>     /* struct kobject, enum kobject_action */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc-AR");
MODULE_DESCRIPTION("kprobe on kobject_uevent — logs every uevent with kobject name and action");

/* ---------------------------------------------------------------
 * action_to_str: convert enum kobject_action to readable string.
 * The enum values match the index in lib/kobject_uevent.c exactly.
 * --------------------------------------------------------------- */
static const char *action_to_str(enum kobject_action action)
{
    switch (action) {
    case KOBJ_ADD:     return "ADD";
    case KOBJ_REMOVE:  return "REMOVE";
    case KOBJ_CHANGE:  return "CHANGE";
    case KOBJ_MOVE:    return "MOVE";
    case KOBJ_ONLINE:  return "ONLINE";
    case KOBJ_OFFLINE: return "OFFLINE";
    case KOBJ_BIND:    return "BIND";
    case KOBJ_UNBIND:  return "UNBIND";
    default:           return "UNKNOWN";
    }
}

/* ---------------------------------------------------------------
 * pre_handler: called BEFORE kobject_uevent executes.
 *
 * Prototype of kobject_uevent:
 *   int kobject_uevent(struct kobject *kobj, enum kobject_action action);
 *
 * On x86-64:
 *   regs->di = 1st arg (kobj pointer)
 *   regs->si = 2nd arg (action enum value)
 * --------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract arguments from registers */
    struct kobject *kobj   = (struct kobject *)regs->di;
    enum kobject_action act = (enum kobject_action)regs->si;

    /* Guard: kobj must be valid and have a name */
    if (!kobj || !kobject_name(kobj))
        return 0;

    /* Log: kobject path and action type */
    pr_info("kprobe_uevent: action=%-8s kobj='%s' refcnt=%u\n",
            action_to_str(act),
            kobject_name(kobj),
            kref_read(&kobj->kref));   /* how many refs on this kobject */

    return 0; /* 0 = continue normal execution */
}

/* ---------------------------------------------------------------
 * kprobe struct: bind handler to kobject_uevent by symbol name.
 * --------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "kobject_uevent",
    .pre_handler = handler_pre,
};

/* ---------------------------------------------------------------
 * module_init: register the kprobe.
 * --------------------------------------------------------------- */
static int __init kprobe_uevent_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_uevent: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_uevent: planted kprobe on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit: unregister the kprobe.
 * يجب الـ unregister قبل ما الـ module يتشال من الذاكرة،
 * عشان لو جه uevent بعد كده هيحاول ينفذ handler مش موجود.
 * --------------------------------------------------------------- */
static void __exit kprobe_uevent_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_uevent: kprobe removed\n");
}

module_init(kprobe_uevent_init);
module_exit(kprobe_uevent_exit);
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/kobject.h>
```

الـ `kprobes.h` بيجيب الـ API كله: `struct kprobe`، `register_kprobe`، `unregister_kprobe`.
الـ `kobject.h` بيعرّف `struct kobject`، `enum kobject_action`، ودالة `kobject_name()` اللي بنستخدمها عشان ناخد الاسم بأمان بدل ما ندخل الـ pointer مباشرة.

---

#### الدالة `action_to_str`

```c
static const char *action_to_str(enum kobject_action action)
```

**الـ `enum kobject_action`** قيمه integers — الـ log بيبقى مش readable لو طبعناهم كأرقام. الدالة دي بتحولهم لنصوص زي `"ADD"` أو `"REMOVE"` عشان الـ dmesg يبقى مفهوم على طول.

---

#### الـ `pre_handler` وقراءة الـ arguments

```c
struct kobject *kobj   = (struct kobject *)regs->di;
enum kobject_action act = (enum kobject_action)regs->si;
```

على **x86-64** الـ calling convention (System V ABI) بيحط أول argument في `rdi` وتاني argument في `rsi`. الـ `pt_regs` بيعكس الـ registers عند لحظة الـ probe — فبنقرأ منه مباشرة من غير ما نغير توقيع الدالة الأصلية.

```c
pr_info("kprobe_uevent: action=%-8s kobj='%s' refcnt=%u\n",
        action_to_str(act),
        kobject_name(kobj),
        kref_read(&kobj->kref));
```

بنطبع تلاتة معلومات مفيدة: نوع الحدث، اسم الـ kobject (زي `"sda"` أو `"eth0"`), ورقم الـ reference count اللي بيوضح مين شايل الـ object ده.

---

#### الـ `kprobe` struct

```c
static struct kprobe kp = {
    .symbol_name = "kobject_uevent",
    .pre_handler = handler_pre,
};
```

الـ kernel بيحل اسم الـ symbol وقت `register_kprobe` ويحط `int3` (breakpoint) على أول instruction. مش محتاج نحدد عنوان يدوي — الـ symbol name كافي طالما الدالة مش `static` ومش inlined.

---

#### `module_init` و `module_exit`

```c
ret = register_kprobe(&kp);
```

لو الـ register فشل (مثلاً الدالة `__kprobes` أو `nokprobe_inline`)، بنرجع الـ error فوراً ومش بنكمل.

```c
unregister_kprobe(&kp);
```

الـ unregister في الـ exit ضروري جداً — لو شلنا الـ module من الذاكرة وسابنا الـ kprobe مسجل، أي uevent بعدها هيجري على كود اتشال وهيحصل **kernel panic**. الـ unregister بيشيل الـ `int3` ويرجع الـ instruction الأصلية.

---

### تجربة الـ module

```bash
# بناء الـ module (مع kernel headers متاحين)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod kprobe_uevent.ko

# شوف الـ log — افصل وصل USB أو شيل/حط network interface
dmesg | grep kprobe_uevent

# مثال على output
# [  142.301] kprobe_uevent: planted kprobe on kobject_uevent @ ffffffff81a3bc20
# [  145.882] kprobe_uevent: action=ADD      kobj='1-1.2' refcnt=1
# [  145.883] kprobe_uevent: action=ADD      kobj='1-1.2:1.0' refcnt=1
# [  148.110] kprobe_uevent: action=REMOVE   kobj='1-1.2:1.0' refcnt=1

# إزالة
sudo rmmod kprobe_uevent
```

---

### ملف `Makefile`

```makefile
obj-m += kprobe_uevent.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
