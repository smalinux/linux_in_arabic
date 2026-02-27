## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ sysfs؟

تخيل إنك اشتريت جهاز كمبيوتر جديد، وعايز تعرف مثلاً: الـ CPU بتاعك بيشتغل بكام ميجاهرتز دلوقتي؟ الـ battery فيها كام بالمية؟ الـ USB device اللي وصلته إيه اسمه وإيه firmware version بتاعه؟

في الـ Linux، الإجابة على كل الأسئلة دي موجودة في مكان واحد اسمه **`/sys`** — ده الـ **sysfs filesystem**.

الـ **sysfs** هو نظام ملفات افتراضي (مش موجود فعلاً على القرص الصلب) بيعرض معلومات الـ kernel والـ devices والـ drivers كملفات نصية عادية في `/sys`. أي برنامج user-space يقدر يقرأ الملفات دي أو يكتب فيها عشان يتحكم في الجهاز أو يستعلم عنه.

---

### القصة: المشكلة اللي الـ sysfs بيحلها

**قبل الـ sysfs** (زمان في Linux القديم):

الـ drivers كانوا بيحطوا معلوماتهم في `/proc` بشكل عشوائي تماماً — كل driver بيعمل ملفاته في `/proc` زي ما هو عايز، مفيش standard، مفيش تنظيم. لو عندك device درايفر ومحتاج تعرض معلومة زي "إيه الـ voltage دلوقتي"، تحطها فين؟ بأي format؟ مفيش حد عارف.

النتيجة: `/proc` بقى **فوضى تامة** — مليان معلومات kernel مع معلومات devices مع أي حاجة تانية كلها متخلطة.

**الحل: الـ sysfs**

Greg Kroah-Hartman وفريقه قرروا يعملوا نظام منظم:
- كل **device** ليه directory في `/sys/devices/`
- كل **attribute** (خاصية) للـ device بتبقى ملف نصي جوه الـ directory ده
- لو عايز تعرف الـ voltage، تعمل `cat /sys/bus/i2c/devices/0-0048/temp1_input`
- لو عايز تغير إعداد، تعمل `echo value > /sys/...`

**الـ `sysfs.h`** هو الـ header اللي بيعرّف كل الـ types والـ macros والـ functions اللي الـ drivers بيستخدموها عشان يضيفوا الملفات دي لـ `/sys`.

---

### نظرة نظرية: كيف الـ sysfs شغال؟

```
   User Space                 Kernel Space
   ----------                 ------------

   cat /sys/devices/          sysfs layer
       .../power/voltage   →  يستدعي show() callback
                              في الـ driver
                          ←  driver بيرجع قيمة الـ voltage

   echo "1" > /sys/.../      sysfs layer
       enable              →  يستدعي store() callback
                              في الـ driver
                              driver بيغير الإعداد
```

الـ architecture هو طبقات:

```
  /sys (user visible)
       ↑
   sysfs layer   ← sysfs.h بيعرّف الـ API ده
       ↑
   kernfs layer  ← kernfs.h الـ low-level filesystem engine
       ↑
   VFS layer     ← Linux Virtual Filesystem
```

الـ **kernfs** هو الـ engine الفعلي اللي بيتعامل مع الـ VFS، أما الـ **sysfs** فهو طبقة فوقيه بيربط الـ kobjects والـ attributes بالـ filesystem.

---

### الـ Building Blocks الأساسية في الـ sysfs.h

#### 1. الـ `struct attribute` — أبسط وحدة

```c
struct attribute {
    const char  *name;   /* اسم الملف في /sys */
    umode_t      mode;   /* permissions: 0444 = read-only, 0644 = read-write */
};
```

ده بببساطة: "ملف اسمه `name` بـ permissions معينة".

#### 2. الـ `struct sysfs_ops` — مين بيقرأ ومين بيكتب؟

```c
struct sysfs_ops {
    ssize_t (*show)(struct kobject *, struct attribute *, char *);
    ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

- **`show`**: اتنادي لما حد يعمل `cat` على الملف — بترجع القيمة
- **`store`**: اتنادي لما حد يكتب في الملف — بتستقبل القيمة الجديدة

#### 3. الـ `struct bin_attribute` — للبيانات البينري (مش نص)

```c
struct bin_attribute {
    struct attribute attr;
    size_t           size;
    ssize_t (*read)(...);
    ssize_t (*write)(...);
    int     (*mmap)(...);  /* ممكن تعمل mmap عليه كمان! */
};
```

للحالات اللي البيانات مش نص — زي firmware images أو EEPROM data.

#### 4. الـ `struct attribute_group` — مجموعة attributes مع بعض

```c
struct attribute_group {
    const char          *name;       /* اسم subdirectory (optional) */
    umode_t (*is_visible)(...);      /* هل الـ attribute يظهر؟ */
    struct attribute    **attrs;     /* قايمة الـ attributes */
    struct bin_attribute **bin_attrs; /* قايمة الـ binary attributes */
};
```

بدل ما تضيف كل attribute لوحده، بتجمعهم في group وتضيفهم مرة واحدة.

---

### مثال حقيقي: driver بسيط

لو عندك temperature sensor driver:

```c
/* تعريف الـ attribute */
static ssize_t temp_show(struct kobject *kobj, struct attribute *attr, char *buf)
{
    return sysfs_emit(buf, "%d\n", read_sensor_temp());
    /* sysfs_emit أأمن من sprintf — بتحمي من overflow */
}

static struct attribute temp_attr = {
    .name = "temperature",
    .mode = 0444,  /* read-only للكل */
};

/* لما يعمل: cat /sys/devices/.../temperature */
/* هيطلع: 37  */
```

---

### المفاهيم المهمة في الـ API

| الدالة | وظيفتها |
|--------|---------|
| `sysfs_create_file()` | بيضيف attribute واحد لـ kobject |
| `sysfs_create_group()` | بيضيف مجموعة attributes مع بعض |
| `sysfs_create_link()` | بيعمل symlink من kobject لـ kobject تاني |
| `sysfs_notify()` | بيبلّغ user-space إن قيمة attribute اتغيرت (poll/inotify) |
| `sysfs_emit()` | آمن بديل لـ sprintf لكتابة في الـ sysfs buffer |
| `sysfs_remove_file()` | بيشيل attribute من /sys |
| `sysfs_create_bin_file()` | بيضيف binary attribute |

---

### الـ Namespace Support

الـ sysfs بيدعم **namespaces** — يعني نفس الـ device ممكن يظهر بشكل مختلف في network namespaces مختلفة. ده مهم جداً للـ containers (Docker مثلاً). الـ `kobject_ns.h` بيعرّف `enum kobj_ns_type` و `struct kobj_ns_type_operations` للغرض ده.

---

### الـ Visibility Control

الـ macros `DEFINE_SYSFS_GROUP_VISIBLE` و `DEFINE_SIMPLE_SYSFS_GROUP_VISIBLE` بتسمح للـ driver إنه يخفي أو يظهر attributes أو حتى directories كاملة في `/sys` ديناميكياً بناءً على حالة الـ hardware. مثلاً: لو الـ feature مش مدعومة في الـ hardware ده، إخفي الـ attribute بدل ما تعرضه بقيمة فاضية أو error.

---

### الفرق بين الـ `CONFIG_SYSFS` الـ enabled والـ disabled

الـ header بيعمل كل الـ functions **stub (فاضية)** لو `CONFIG_SYSFS=n`. ده يعني إن الـ drivers اللي بتستخدم sysfs مش محتاجة تعمل `#ifdef` في كل حتة — الـ code بيتكمبايل عادي بس مفيش حاجة بتحصل.

---

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/sysfs.h` | الـ API الرئيسي — ده هو ملفنا |
| `include/linux/kernfs.h` | الـ low-level engine اللي sysfs بيبني عليه |
| `include/linux/kobject.h` | تعريف `struct kobject` — المحور الأساسي |
| `include/linux/kobject_ns.h` | دعم الـ namespaces للـ kobjects |
| `fs/sysfs/` | الـ implementation الفعلي للـ sysfs |
| `fs/kernfs/` | الـ kernfs implementation |
| `drivers/base/` | الـ device model اللي بيستخدم sysfs |
| `include/linux/device.h` | الـ device abstraction layer فوق الـ kobject |
| `Documentation/filesystems/sysfs.rst` | الـ documentation الرسمي |
| `Documentation/core-api/kobject.rst` | شرح الـ kobject model |

---

### ملفات الـ Subsystem الأساسية

**Core:**
- `fs/sysfs/dir.c` — إدارة الـ directories
- `fs/sysfs/file.c` — إدارة الـ regular attributes
- `fs/sysfs/bin.c` — إدارة الـ binary attributes
- `fs/sysfs/symlink.c` — إدارة الـ symlinks
- `fs/sysfs/mount.c` — الـ mount logic

**Headers:**
- `include/linux/sysfs.h` — الـ public API
- `include/linux/kernfs.h` — الـ underlying engine
- `include/linux/kobject.h` — الـ object model

**الـ Driver Model (بيستخدم sysfs):**
- `drivers/base/core.c` — تسجيل الـ devices في sysfs
- `drivers/base/bus.c` — الـ bus subsystem
- `drivers/base/class.c` — الـ device classes
- `lib/kobject.c` — الـ kobject implementation
## Phase 2: شرح الـ sysfs Framework

### المشكلة: ليه sysfs موجود أصلاً؟

قبل الـ sysfs، الـ kernel كانت بتتكلم مع الـ userspace بطرق مبعثرة:

- **`/proc`** — اتعمل الأول للـ process info، بس الناس بدأت تحشر فيه كل حاجة (network stats، hardware info، driver config)، فبقى فوضى.
- **`ioctl`** — binary interface، صعب debug، مفيش standardization.
- **Custom char devices** — كل driver بيعمل interface خاص بيه من الصفر.

النتيجة؟ عندك driver بيحتاج يعرض للـ userspace معلومة زي "الـ CPU frequency دلوقتي كام؟" أو "الـ device دي enabled ولا لأ؟" — مفيش طريقة موحدة. كل driver بيعمل الحاجة بطريقته.

**المشكلة الأعمق**: الـ kernel داخلياً بتنظم الأجهزة في hierarchy — bus → device → driver. الـ userspace مكانتش شايفة الـ hierarchy دي خالص. لو عايز تعرف إيه الـ devices اللي على الـ USB bus، كنت بتعمل إيه بالظبط؟

---

### الحل: Virtual Filesystem كـ Contract

الـ **sysfs** هو virtual filesystem بيتـ mount على `/sys` وبيعكس الـ kernel's internal object model كـ directory tree. كل **kobject** في الـ kernel بييجي ليه directory في `/sys`. كل **attribute** بييجي ليه file داخل الـ directory ده.

القاعدة الذهبية في sysfs:

> **كل file في `/sys` بيمثل attribute واحد بس — مش أكتر.**

يعني مش هتلاقي file واحد بيعرض 5 قيم في نفس الوقت. كل قيمة = file منفصل. ده design decision مقصود عشان الـ userspace tools تقدر تتعامل معاه ببساطة بـ `cat` و `echo`.

---

### الـ Real-World Analogy — وبنربطه بالـ Kernel

تخيل شركة كبيرة فيها **organizational chart (org chart)**. الـ org chart ده:

```
شركة (Company)
├── قسم الـ Engineering
│   ├── فريق الـ Hardware
│   │   ├── الموظف A  →  [ملف: الراتب، الإجازات، الـ projects]
│   │   └── الموظف B  →  [ملف: الراتب، الإجازات، الـ projects]
│   └── فريق الـ Software
└── قسم الـ Sales
```

دلوقتي ربطه بالـ kernel:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| الشركة نفسها | الـ kernel كله |
| الـ Org Chart | الـ `/sys` directory tree |
| كل قسم / فريق | `kobject` (بيعمل directory) |
| كل موظف | device/driver/bus مرتبط بـ kobject |
| ملف بيانات الموظف | `attribute` (file في `/sys`) |
| حقل واحد في الملف (الراتب مثلاً) | قيمة attribute واحدة |
| موظف HR بيقرأ الملف | userspace process بيعمل `cat /sys/...` |
| موظف HR بيعدل الملف | userspace process بيعمل `echo val > /sys/...` |
| الـ HR system نفسه | الـ sysfs framework |
| طريقة قراءة بيانات الموظف | `show()` callback |
| طريقة تعديل بيانات الموظف | `store()` callback |
| مدير القسم اللي بيقرر مين يشوف إيه | `is_visible()` callback في `attribute_group` |

الـ org chart مش بيخزن البيانات الحقيقية — البيانات جوه الموظفين أنفسهم. نفس الكلام: الـ sysfs مش بيخزن قيم الـ attributes — هو بس بيوفر الـ interface، والـ driver هو اللي عنده القيم الحقيقية.

---

### الـ Big Picture Architecture

```
User Space
──────────────────────────────────────────────────────────────────
  $ cat /sys/bus/i2c/devices/0-0068/power/runtime_status
  $ echo 200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
──────────────────────────────────────────────────────────────────
                              │  VFS syscalls (read/write/open)
                              ▼
──────────────────────────────────────────────────────────────────
             VFS Layer (Virtual File System)
             (inode, dentry, file operations)
──────────────────────────────────────────────────────────────────
                              │
                              ▼
──────────────────────────────────────────────────────────────────
                       kernfs Layer
          ┌─────────────────────────────────────┐
          │  kernfs_node (per file/dir/symlink)  │
          │  kernfs_ops  (file ops callbacks)    │
          │  kernfs_root (mount point)           │
          │  Handles: locking, reference counts  │
          │           seq_file integration       │
          │           poll/notify                │
          └─────────────────────────────────────┘
                              │
                              ▼
──────────────────────────────────────────────────────────────────
                       sysfs Layer
          ┌─────────────────────────────────────┐
          │  struct attribute      (text files)  │
          │  struct bin_attribute  (binary files)│
          │  struct attribute_group              │
          │  struct sysfs_ops {show, store}      │
          │  sysfs_create_file / sysfs_notify    │
          └─────────────────────────────────────┘
                              │
                              ▼
──────────────────────────────────────────────────────────────────
                      kobject Layer
          ┌─────────────────────────────────────┐
          │  struct kobject  → sysfs directory   │
          │  struct ktype    → sysfs_ops         │
          │  struct kset     → kobject container │
          └─────────────────────────────────────┘
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
──────────────────────    ──────────────────────────────────────
   Bus Subsystem               Device / Driver Subsystem
   (pci, usb, i2c, ...)        (platform_device, i2c_client, ...)
──────────────────────    ──────────────────────────────────────
          │                              │
          ▼                              ▼
  Driver registers             Driver registers attributes:
  bus attributes:              DEVICE_ATTR_RO(temp1_input)
  BUS_ATTR_RO(...)             DEVICE_ATTR_RW(enable)
──────────────────────────────────────────────────────────────────
Physical Hardware (I2C sensors, PCI cards, platform registers ...)
```

الـ **kernfs** ده subsystem مستقل بيعمل الـ heavy lifting للـ VFS integration. الـ sysfs بيبني فوقيه. قبل الـ kernfs (قديماً)، الـ sysfs كان بيتكلم مع الـ VFS مباشرة — ده كان مشكلة في الـ locking. الـ kernfs اتعمل عشان يفصل الـ concerns ده.

---

### الـ Core Abstractions — الـ Structs المهمة

#### 1. `struct attribute` — الوحدة الأساسية

```c
struct attribute {
    const char  *name;   /* اسم الـ file في /sys */
    umode_t      mode;   /* permissions: 0444, 0644, 0200 ... */
};
```

ده الـ metadata بس. مش فيه قيمة ولا callback. الـ attribute هو وصف الـ file — مش الـ file نفسه.

---

#### 2. `struct sysfs_ops` — الـ Dispatch Table

```c
struct sysfs_ops {
    ssize_t (*show) (struct kobject *, struct attribute *, char *);
    ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

الـ `show` اتبعت لما userspace يعمل `read()` على الـ file.
الـ `store` اتبعت لما userspace يعمل `write()` على الـ file.

**نقطة مهمة**: الـ `sysfs_ops` بتتحدد على مستوى الـ `ktype` (نوع الـ kobject)، مش على مستوى كل attribute منفردة. يعني كل الـ attributes اللي في نفس الـ kobject بتعدي من نفس الـ show/store — والـ driver هو اللي بيعمل dispatch داخلياً على أساس الـ `struct attribute *`.

---

#### 3. `struct bin_attribute` — للـ Binary Data

```c
struct bin_attribute {
    struct attribute  attr;       /* base attribute */
    size_t            size;       /* max size */
    void             *private;    /* driver private data */
    ssize_t (*read) (struct file *, struct kobject *,
                     const struct bin_attribute *, char *, loff_t, size_t);
    ssize_t (*write)(struct file *, struct kobject *,
                     const struct bin_attribute *, char *, loff_t, size_t);
    int     (*mmap) (struct file *, struct kobject *,
                     const struct bin_attribute *, struct vm_area_struct *);
};
```

الفرق الجوهري عن الـ `attribute` العادية:
- الـ text attribute: بتبعت buffer كامل، بيكتب فيه الـ driver text، بتعرضه كـ string.
- الـ bin_attribute: فيها `loff_t offset` — يعني بتدعم الـ seeking، ممكن تبقى أكبر من PAGE_SIZE، وممكن تعمل `mmap`. مثال حقيقي: EEPROM driver بيعرض محتوى الـ EEPROM كـ binary file في `/sys`.

---

#### 4. `struct attribute_group` — التجميع المنطقي

```c
struct attribute_group {
    const char          *name;           /* NULL = same dir as kobject */
    umode_t (*is_visible)(struct kobject *, struct attribute *, int);
    umode_t (*is_bin_visible)(struct kobject *, const struct bin_attribute *, int);
    size_t  (*bin_size)(struct kobject *, const struct bin_attribute *, int);
    struct attribute    **attrs;         /* NULL-terminated array */
    const struct bin_attribute *const *bin_attrs;
};
```

الـ `attribute_group` بيحل مشكلتين:
1. **تجميع attributes** تحت subdirectory (لو `name` مش NULL).
2. **Dynamic visibility** — تقدر تخفي attribute بالكامل بناءً على state الـ hardware. مثلاً: thermal sensor فيه attributes للـ alarm فقط لو الـ hardware يدعمها.

الـ `is_visible()` بترجع `0` تعني "الـ attribute مش موجودة"، أو بترجع الـ mode المطلوب.

---

### علاقة الـ Structs ببعض — رسم تفصيلي

```
struct kobject
┌──────────────────────────┐
│  name: "hwmon0"          │──────────────────► /sys/class/hwmon/hwmon0/
│  ktype ──────────────────┼──► struct ktype
│  parent ─────────────────┼──► parent kobject    ┌────────────────────┐
│  sd (kernfs_node) ───────┼──► kernfs_node        │  sysfs_ops         │
└──────────────────────────┘    (actual dir)       │  ┌──────────────┐  │
                                                   │  │ .show = ...  │  │
                                                   │  │ .store = ... │  │
              sysfs_create_group()                 │  └──────────────┘  │
              ┌──────────────────────────────────► │  default_attrs     │
              │                                    └────────────────────┘
              │
    struct attribute_group
    ┌─────────────────────────────────────┐
    │  name: "power"                      │──► creates subdir /sys/.../power/
    │  is_visible: power_attr_visible()   │
    │  attrs: ──────────────────────────► │ [attr1, attr2, attr3, NULL]
    └─────────────────────────────────────┘        │
                                                   │
                              ┌────────────────────┘
                              ▼
                    struct attribute
                    ┌──────────────────┐
                    │  name: "status"  │──► file: /sys/.../power/status
                    │  mode: 0444      │
                    └──────────────────┘
                              │
                              │  when user reads the file:
                              ▼
                    ktype->sysfs_ops->show(kobj, attr, buf)
                              │
                              │  driver identifies attr by pointer comparison
                              ▼
                    driver fills buf with string, returns length
```

---

### الـ Namespace Support — `kobj_ns_type`

ده مفهوم متقدم مهم. المشكلة: لو عندك containers (Docker/LXC)، كل container المفروض يشوف بس الـ network devices بتاعته في `/sys/class/net/`، مش devices الـ container التاني.

الـ sysfs بيحل ده بـ **namespace tagging**:

```c
enum kobj_ns_type {
    KOBJ_NS_TYPE_NONE = 0,
    KOBJ_NS_TYPE_NET,   /* network namespace */
    KOBJ_NS_TYPES
};
```

كل `kernfs_node` ممكن يتـ tag بـ `ns` pointer. لما userspace يقرأ directory، الـ sysfs بيفلتر الـ entries على أساس الـ namespace اللي الـ process ده شايله. الـ `sysfs_enable_ns(kn)` بتفعل الـ filtering ده على node معين.

---

### الـ Notification Mechanism — `sysfs_notify()`

```c
void sysfs_notify(struct kobject *kobj, const char *dir, const char *attr);
```

ده بيسمح للـ driver إنه يقول للـ userspace "القيمة اتغيرت". الـ userspace بيعمل `poll()` على الـ sysfs file، والـ driver بيكول `sysfs_notify()` لما الـ hardware state يتغير. مثال حقيقي: thermal driver بيعمل `sysfs_notify()` لما الـ temperature تعدي threshold.

داخلياً ده بيتم عبر `kernfs_notify()` اللي بتـ wake up الـ waiters على الـ `kernfs_node`.

---

### الـ `VERIFY_OCTAL_PERMISSIONS` Macro — Safety Net

```c
#define VERIFY_OCTAL_PERMISSIONS(perms)
    /* USER_READABLE >= GROUP_READABLE >= OTHER_READABLE */
    /* USER_WRITABLE >= GROUP_WRITABLE */
    /* OTHER_WRITABLE? Generally considered a bad idea. */
```

الـ macro ده بيعمل **compile-time checks** على الـ permissions. لو حد غلطاً عمل attribute بـ `0666` (world-writable)، الـ code مش هيكمبايل. ده security enforcement على مستوى الـ compiler.

---

### الـ Helper Macros — من أكتر لأقل abstraction

الـ kernel بيوفر hierarchy من الـ macros عشان تسهل كتابة الـ attributes:

```c
/* Level 1: الأساسي */
struct attribute my_attr = { .name = "foo", .mode = 0444 };

/* Level 2: __ATTR — بيربط الـ show/store بشكل أوضح */
static struct device_attribute dev_attr_foo = __ATTR(foo, 0444, foo_show, NULL);

/* Level 3: Convenience macros */
static DEVICE_ATTR_RO(foo);   /* يفترض وجود foo_show() */
static DEVICE_ATTR_RW(bar);   /* يفترض وجود bar_show() و bar_store() */

/* Level 4: ATTRIBUTE_GROUPS — تجميع كامل */
static struct attribute *my_attrs[] = { &dev_attr_foo.attr, NULL };
ATTRIBUTE_GROUPS(my);  /* بيعمل my_group و my_groups[] */
```

الـ `ATTRIBUTE_GROUPS` macro بيستخدم `_Generic` (C11) عشان يدعم الـ `const` و non-const variants بدون code duplication.

---

### الـ `sysfs_emit()` vs `sprintf()`

```c
int sysfs_emit(char *buf, const char *fmt, ...);
int sysfs_emit_at(char *buf, int at, const char *fmt, ...);
```

لازم تستخدم `sysfs_emit` مش `sprintf` في الـ `show()` callbacks. السبب:
- الـ `buf` في الـ show callback حجمه `PAGE_SIZE` (4096 bytes).
- الـ `sysfs_emit` بيعمل bounds checking تلقائياً — مش هيكتب أكتر من `PAGE_SIZE`.
- الـ `sysfs_emit_at` بيسمحلك تكتب في offset محدد داخل الـ buffer، مفيد لو بتبني الـ output على عدة خطوات.

---

### ما الـ sysfs بيملكه vs ما بيفوضه للـ Drivers

| المسؤولية | الـ sysfs Framework | الـ Driver |
|---|---|---|
| إنشاء الـ VFS inode | ✓ (عبر kernfs) | |
| الـ Locking على الـ file | ✓ (kernfs active ref) | |
| الـ Buffer allocation للـ show | ✓ (PAGE_SIZE buffer) | |
| ملء الـ buffer بالقيمة | | ✓ (`show()` callback) |
| تفسير الـ user input | | ✓ (`store()` callback) |
| تحديد الـ permissions | مشترك (umode_t في attr) | ✓ (is_visible override) |
| الـ Namespace filtering | ✓ | |
| الـ Poll/notify infrastructure | ✓ | يستدعي `sysfs_notify()` |
| معرفة قيمة الـ register الحقيقية | | ✓ |
| الـ Hardware access (readl/writel) | | ✓ |
| الـ Symlinks بين kobjects | ✓ (`sysfs_create_link`) | |
| الـ Attribute group visibility | مشترك | ✓ (is_visible logic) |

**الخلاصة**: الـ sysfs بيملك الـ plumbing (VFS, locking, lifecycle)، والـ driver بيملك الـ semantics (القيمة الحقيقية وتفسيرها).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Permissions & Config Options — Cheatsheet

#### Permission Flags (sysfs-specific)

| Macro | القيمة | الاستخدام |
|---|---|---|
| `SYSFS_PREALLOC` | `010000` (octal) | buffer مخصص مسبقاً عند الـ open — يُستخدم مع `__ATTR_PREALLOC` |
| `SYSFS_GROUP_INVISIBLE` | `020000` (octal) | إخفاء الـ directory بتاع الـ group كاملاً من sysfs |

#### Standard Permission Shortcuts

| Macro | Mode | المعنى |
|---|---|---|
| `__ATTR_RO` | `0444` | read-only لكل الـ users |
| `__ATTR_WO` | `0200` | write-only للـ owner |
| `__ATTR_RW` | `0644` | read للكل، write للـ owner |
| `__BIN_ATTR_ADMIN_RO` | `0400` | read-only للـ root فقط |
| `__BIN_ATTR_ADMIN_RW` | `0600` | read/write للـ root فقط |

#### Config Guards المهمة

| Config | الأثر |
|---|---|
| `CONFIG_SYSFS` | لو مش enabled، كل الـ functions بتبقى stubs ترجع 0 |
| `CONFIG_DEBUG_LOCK_ALLOC` | يفعّل `lockdep` fields في `struct attribute` + يحتاج `sysfs_attr_init()` |
| `CONFIG_CFI` | يغير `__SYSFS_FUNCTION_ALTERNATIVE` من `union` لـ `struct` عشان CFI safety |

#### `kernfs_node_flag` Cheatsheet

| Flag | المعنى |
|---|---|
| `KERNFS_ACTIVATED` | الـ node ظهر فعلاً في الـ hierarchy |
| `KERNFS_NS` | namespace filtering مفعّل تحته |
| `KERNFS_HAS_SEQ_SHOW` | بيستخدم `seq_file` API للقراءة |
| `KERNFS_HAS_MMAP` | بيدعم `mmap` |
| `KERNFS_LOCKDEP` | lockdep tracking مفعّل |
| `KERNFS_HIDDEN` | مخفي من الـ namespace الحالي |
| `KERNFS_SUICIDAL` | في طريقه للحذف |
| `KERNFS_REMOVING` | عملية الـ remove جارية |

---

### كل الـ Structs المهمة

#### 1. `struct attribute`

**الغرض:** أساس كل attribute في sysfs — يمثل ملف نصي واحد (`/sys/.../name`).

```c
struct attribute {
    const char   *name;   /* اسم الملف في sysfs */
    umode_t       mode;   /* permissions مثل 0644 */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    bool          ignore_lockdep:1;    /* تجاهل lockdep لهذا الـ attr */
    struct lock_class_key *key;        /* pointer للـ lockdep key */
    struct lock_class_key  skey;       /* embedded key للـ static attrs */
#endif
};
```

**الارتباطات:**
- مضمّن داخل `struct bin_attribute` كأول field
- مضمّن داخل device_attribute, class_attribute, etc.
- **الـ** `attribute_group` بتحتوي array منهم

---

#### 2. `struct bin_attribute`

**الغرض:** attribute ثنائي (binary) — بيدعم قراءة/كتابة arbitrary bytes مع offset، وبيدعم `mmap`. مثال: قراءة EEPROM أو firmware blob.

```c
struct bin_attribute {
    struct attribute  attr;       /* يرث الـ name/mode من attribute عادي */
    size_t            size;       /* حجم الـ binary data (0 = غير محدود) */
    void             *private;   /* بيانات خاصة بالـ driver */
    struct address_space *(*f_mapping)(void);  /* لو بيدعم page cache */

    /* File operations للـ binary data */
    ssize_t (*read) (struct file *, struct kobject *,
                     const struct bin_attribute *, char *, loff_t, size_t);
    ssize_t (*write)(struct file *, struct kobject *,
                     const struct bin_attribute *, char *, loff_t, size_t);
    loff_t  (*llseek)(struct file *, struct kobject *,
                      const struct bin_attribute *, loff_t, int);
    int     (*mmap) (struct file *, struct kobject *,
                     const struct bin_attribute *, struct vm_area_struct *);
};
```

**الارتباطات:**
- يحتوي على `struct attribute` كأول member (أي يمكن cast بينهم)
- **الـ** `attribute_group.bin_attrs` بيشاور عليه
- بيتسجل عبر `sysfs_create_bin_file()`

---

#### 3. `struct attribute_group`

**الغرض:** تجميع مجموعة attributes (عادية أو binary) تحت directory واحدة أو مباشرة تحت الـ kobject. بيسمح بالتحكم في visibility الـ attrs كلها دفعة واحدة.

```c
struct attribute_group {
    const char *name;   /* اسم subdirectory (NULL = في nevel الـ kobject نفسه) */

    /* اختيار: تحديد الـ permissions ديناميكياً (بيبطل static mode) */
    union/struct {  /* union في non-CFI، struct في CFI */
        umode_t (*is_visible)(struct kobject *, struct attribute *, int);
        umode_t (*is_visible_const)(struct kobject *, const struct attribute *, int);
    };
    umode_t (*is_bin_visible)(struct kobject *, const struct bin_attribute *, int);
    size_t  (*bin_size)      (struct kobject *, const struct bin_attribute *, int);

    /* قائمة الـ attributes — لازم واحدة منهم على الأقل */
    union {
        struct attribute       **attrs;        /* mutable */
        const struct attribute *const *attrs_const; /* immutable */
    };
    const struct bin_attribute *const *bin_attrs; /* binary attrs */
};
```

**الارتباطات:**
- بتحتوي array من `struct attribute*`
- بتحتوي array من `struct bin_attribute*`
- بتُعطى لـ `kobject` عبر `sysfs_create_group()`

---

#### 4. `struct sysfs_ops`

**الغرض:** الـ vtable اللي بيحدد إزاي يتم قراءة/كتابة الـ text attributes لـ `kobj_type` معين.

```c
struct sysfs_ops {
    ssize_t (*show) (struct kobject *, struct attribute *, char *);
    ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};
```

**الارتباطات:**
- بتتسجل في `struct kobj_type` (مش موجودة في الـ header ده بس مرتبطة به)
- **الـ** sysfs core بيلاقيها عبر `kobject->ktype->sysfs_ops`
- الـ `show` بتاخد buffer PAGE_SIZE وبترجع عدد البايتات المكتوبة

---

#### 5. `struct kernfs_node` (من kernfs.h)

**الغرض:** الـ building block الفعلي لكل entry في sysfs filesystem — كل ملف أو directory أو symlink في `/sys` عنده `kernfs_node`.

```c
struct kernfs_node {
    atomic_t         count;       /* reference count */
    atomic_t         active;      /* active users count */
    struct kernfs_node __rcu *__parent; /* parent directory */
    const char       __rcu *name; /* اسم الـ entry */
    struct rb_node   rb;          /* في الـ rbtree بتاع الـ parent */
    const void      *ns;          /* namespace tag للـ filtering */
    unsigned int     hash;        /* ns + name hash للبحث السريع */
    unsigned short   flags;       /* kernfs_node_flag */
    umode_t          mode;
    union {
        struct kernfs_elem_dir     dir;     /* لو KERNFS_DIR */
        struct kernfs_elem_symlink symlink; /* لو KERNFS_LINK */
        struct kernfs_elem_attr    attr;    /* لو KERNFS_FILE */
    };
    u64              id;           /* inode number */
    void            *priv;         /* private data (مثلاً pointer للـ attribute) */
    struct kernfs_iattrs *iattr;   /* extended attributes */
    struct rcu_head  rcu;
};
```

**الارتباطات:**
- **الـ** sysfs بتبني كل attribute فوق `kernfs_node`
- `sysfs_get_dirent()` بترجع `kernfs_node*`
- `sysfs_notify_dirent()` بتستخدمه مباشرة

---

#### 6. `struct kobj_ns_type_operations` (من kobject_ns.h)

**الغرض:** operations table لنوع معين من الـ namespaces (مثلاً network namespace). بيسمح لـ sysfs يعرض entries مختلفة لكل namespace.

```c
struct kobj_ns_type_operations {
    enum kobj_ns_type type;                          /* KOBJ_NS_TYPE_NET مثلاً */
    bool  (*current_may_mount)(void);                /* هل مسموح بالـ mount؟ */
    void *(*grab_current_ns)(void);                  /* reference للـ namespace الحالي */
    const void *(*netlink_ns)(struct sock *sk);      /* namespace الخاص بـ socket */
    const void *(*initial_ns)(void);                 /* الـ initial/root namespace */
    void  (*drop_ns)(void *);                        /* release الـ namespace reference */
};
```

---

### Struct Relationship Diagram

```
                        ┌─────────────────────────────────┐
                        │         struct kobject           │
                        │  (defined in kobject.h)          │
                        │  .ktype → struct kobj_type       │
                        └──────────────┬──────────────────┘
                                       │ ktype->sysfs_ops
                                       ▼
                        ┌─────────────────────────────────┐
                        │       struct sysfs_ops           │
                        │  .show()                         │
                        │  .store()                        │
                        └─────────────────────────────────┘

kobject registers attrs:
sysfs_create_group(kobj, grp)
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│              struct attribute_group                        │
│  .name       → subdirectory name (optional)               │
│  .is_visible → dynamic permission callback                │
│  .attrs[]  ──┼──► struct attribute  { name, mode }        │
│              │         ▲                                   │
│              │         └── embedded in device_attribute    │
│  .bin_attrs[]┼──► struct bin_attribute                    │
│                         { .attr, .size, .read, .write,    │
│                           .mmap, .llseek, .private }       │
└───────────────────────────────────────────────────────────┘
        │
        │ sysfs layer creates for each attr:
        ▼
┌───────────────────────────────────────────────────────────┐
│              struct kernfs_node                            │
│  .flags = KERNFS_FILE                                     │
│  .priv  → pointer to struct attribute                     │
│  .attr.ops → struct kernfs_ops (show/store bridge)        │
│  .__parent → kernfs_node (parent dir)                     │
│  .rb       → in parent's rbtree                           │
└───────────────────────────────────────────────────────────┘
        │
        │ parent dir node:
        ▼
┌───────────────────────────────────────────────────────────┐
│         struct kernfs_node (KERNFS_DIR)                    │
│  .dir.children → rb_root (rbtree of children)             │
│  .dir.root     → struct kernfs_root                       │
└───────────────────────────────────────────────────────────┘

Namespace filtering:
┌──────────────────────────────────────────┐
│     struct kobj_ns_type_operations       │
│  .grab_current_ns()                      │
│  .netlink_ns()                           │
│  .drop_ns()                              │
└──────────┬───────────────────────────────┘
           │ registered per ns_type
           ▼
    kernfs_node.ns = namespace tag
    (only matching ns sees the node)
```

---

### Lifecycle Diagrams

#### دورة حياة الـ `attribute` العادي

```
[1] DEFINE / DECLARE
    static struct attribute my_attr = {
        .name = "my_file",
        .mode = 0644,
    };
    أو بالـ macro:  __ATTR_RW(my_file)
         │
         ▼
[2] INIT (لو dynamic allocation)
    sysfs_attr_init(&my_attr);   ← مطلوب فقط لو CONFIG_DEBUG_LOCK_ALLOC
         │
         ▼
[3] REGISTER
    sysfs_create_file(kobj, &my_attr);
      ↳ sysfs_create_file_ns(kobj, attr, NULL)
      ↳ kernfs_create_file_ns()
      ↳ ينشئ kernfs_node جديد (KERNFS_FILE)
         │
         ▼
[4] IN USE
    User reads:  cat /sys/.../my_file
      ↳ kernfs → sysfs_kf_seq_show()
      ↳ kobj->ktype->sysfs_ops->show(kobj, attr, buf)
      ↳ driver's show() function

    User writes: echo "val" > /sys/.../my_file
      ↳ kernfs → sysfs_kf_write()
      ↳ kobj->ktype->sysfs_ops->store(kobj, attr, buf, size)
      ↳ driver's store() function
         │
         ▼
[5] NOTIFY (poll/inotify)
    sysfs_notify(kobj, dir, attr_name);
      ↳ kernfs_notify(kn)
      ↳ يوقظ الـ poll waiters
         │
         ▼
[6] TEARDOWN
    sysfs_remove_file(kobj, &my_attr);
      ↳ sysfs_remove_file_ns(kobj, attr, NULL)
      ↳ kernfs_remove_by_name_ns()
      ↳ يحذف الـ kernfs_node ويريح الـ reference
```

---

#### دورة حياة الـ `attribute_group`

```
[1] DECLARE GROUP
    static struct attribute *my_attrs[] = { &attr1, &attr2, NULL };
    ATTRIBUTE_GROUPS(my);   ← يولّد my_group و my_groups[]
         │
         ▼
[2] REGISTER GROUP
    sysfs_create_group(kobj, &my_group);
      ↳ لو .name موجود: ينشئ subdirectory kernfs_node
      ↳ لكل attr: بيستدعي .is_visible() (لو موجود)
      ↳ لو returned mode != 0: ينشئ kernfs_node للـ attr
         │
         ▼
[3] UPDATE (runtime visibility change)
    sysfs_update_group(kobj, &my_group);
      ↳ بيعيد فحص is_visible() لكل attr
      ↳ يضيف/يشيل nodes حسب النتيجة
         │
         ▼
[4] TEARDOWN
    sysfs_remove_group(kobj, &my_group);
      ↳ يشيل كل attrs ثم الـ directory لو موجود
```

---

#### دورة حياة الـ `bin_attribute`

```
[1] DECLARE
    BIN_ATTR_RO(firmware, 4096);
    /* ينشئ: struct bin_attribute bin_attr_firmware */
         │
         ▼
[2] REGISTER
    sysfs_create_bin_file(kobj, &bin_attr_firmware);
      ↳ kernfs_create_file_ns() بـ kernfs_ops خاصة بـ binary
         │
         ▼
[3] IN USE
    User reads:  read(fd, buf, 4096)
      ↳ kernfs → bin_attr->read(file, kobj, attr, buf, off, count)
      ↳ driver returns binary data

    User mmaps:  mmap(NULL, 4096, PROT_READ, MAP_SHARED, fd, 0)
      ↳ bin_attr->mmap(file, kobj, attr, vma)
         │
         ▼
[4] TEARDOWN
    sysfs_remove_bin_file(kobj, &bin_attr_firmware);
```

---

### Call Flow Diagrams

#### قراءة text attribute (cat /sys/bus/pci/devices/.../vendor)

```
userspace: read(fd, buf, PAGE_SIZE)
  │
  ▼ VFS
kernfs_fop_read_iter()
  │
  ▼
seq_read()   [لأن KERNFS_HAS_SEQ_SHOW]
  │
  ▼
kernfs_ops->seq_show()   [= sysfs_kf_seq_show()]
  │
  ▼  يبحث عن sysfs_ops من kobj->ktype
kobj->ktype->sysfs_ops->show(kobj, attr, buf)
  │
  ▼  driver callback
vendor_show(dev, attr, buf)
  └─► sprintf(buf, "0x%04x\n", pdev->vendor)
      returns count

  ◄── sysfs_emit() recommended wrapper (checks PAGE_SIZE)
```

#### كتابة text attribute (echo "1" > /sys/.../enable)

```
userspace: write(fd, "1\n", 2)
  │
  ▼ VFS
kernfs_fop_write_iter()
  │
  ▼
sysfs_kf_write()
  │  يجيب الـ attribute من kn->priv
  ▼
kobj->ktype->sysfs_ops->store(kobj, attr, buf, count)
  │
  ▼ driver callback
enable_store(dev, attr, buf, count)
  └─► kstrtoint(buf, 0, &val)
      do_something(val)
      return count
```

#### إنشاء group كامل

```
driver probe:
sysfs_create_groups(kobj, my_groups[])
  │
  ▼  for each group in groups[]:
sysfs_create_group(kobj, grp)
  │
  ├─► لو grp->name:
  │     kernfs_create_dir_ns()  → kernfs_node(KERNFS_DIR)
  │
  └─► for each attr in grp->attrs[]:
        grp->is_visible(kobj, attr, n)  → umode_t
        لو mode != 0:
          sysfs_add_file_mode_ns()
            → kernfs_create_file_ns()
              → kernfs_node(KERNFS_FILE)
              → kn->priv = attr
```

#### sysfs_notify flow

```
driver detects change:
sysfs_notify(kobj, NULL, "state")
  │
  ▼
kernfs_find_and_get(kobj->sd, "state")  → kn
  │
  ▼
kernfs_notify(kn)
  │
  ├─► يضيف kn لـ notify_list
  │
  └─► schedule_work(&kernfs_notify_work)
        │
        ▼
      kernfs_notify_workfn()
        │
        ├─► kn->attr.open->poll_waitq  ← يوقظ poll()
        └─► inotify/fsnotify events
```

---

### Locking Strategy

#### الـ Locks المستخدمة

| Lock | يحمي | مكانه |
|---|---|---|
| `kernfs_rwsem` (per-root) | hierarchy structure — إضافة/حذف nodes | `kernfs_root` |
| `kernfs_global_locks.open_file_mutex[hash]` | `kernfs_open_node->files` list | global hashed array |
| `kernfs_open_file->mutex` | single open file state | per open file |
| `kernfs_open_file->prealloc_mutex` | الـ prealloc buffer | per open file |
| `lockdep` على الـ `attribute` | يكشف انتهاكات lock ordering في sysfs callbacks | debug only |

#### Lock Ordering

```
kernfs_rwsem (write)       ← أعلى مستوى، يحمي الـ tree
  └─► open_file_mutex[i]   ← لما نحتاج نعدّل open files list
        └─► of->mutex      ← أثناء تنفيذ read/write على ملف واحد
```

#### Active Reference vs Count Reference

**الـ** `kernfs_node` عنده مستويين من الـ reference counting:

```
kernfs_node.count   (atomic_t)
  ├── كل من عنده pointer للـ node يحجز reference
  └── لما توصل 0 → الـ memory بتتحرر

kernfs_node.active  (atomic_t)
  ├── يمثل "active users" — من بيعمل I/O دلوقتي
  ├── طالما > 0: الـ node مش هيتحذف
  ├── sysfs_break_active_protection() → بتعمل put للـ active ref
  └── sysfs_unbreak_active_protection() → بتعيد الـ active ref
```

#### lockdep و `sysfs_attr_init()`

لما `CONFIG_DEBUG_LOCK_ALLOC` مفعّل، كل attribute لازم يتعمله init قبل التسجيل:

```c
/* لو الـ attr static → key متضمن في .skey تلقائياً */
static struct attribute my_static_attr = { ... };
/* OK مباشرة */

/* لو الـ attr dynamic → لازم sysfs_attr_init */
struct attribute *my_dyn_attr = kmalloc(...);
sysfs_attr_init(my_dyn_attr);  /* بيحدد lock_class_key فريد */
sysfs_create_file(kobj, my_dyn_attr);
```

بدون `sysfs_attr_init()` على الـ dynamic attrs، lockdep بيطلع warning عشان مش قادر يميز بين lock classes مختلفة.

#### CFI و `__SYSFS_FUNCTION_ALTERNATIVE`

في الـ kernels المبنية بـ `CONFIG_CFI` (Control Flow Integrity):

```c
/* بدون CFI: union — الاتنين بيشتركوا في نفس الـ memory */
union {
    umode_t (*is_visible)(struct kobject *, struct attribute *, int);
    umode_t (*is_visible_const)(struct kobject *, const struct attribute *, int);
};

/* مع CFI: struct — كل pointer له مكانه الخاص */
/* لأن CFI بتتحقق من type signature الـ function pointer */
/* والـ union بيخلي الاتنين يشتركوا في نفس الـ address وده بيكسر CFI */
struct {
    umode_t (*is_visible)(...);
    umode_t (*is_visible_const)(...);
};
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ APIs

#### Directory Management

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `sysfs_create_dir_ns` | `(kobj, ns) → int` | إنشاء directory لـ kobject مع namespace |
| `sysfs_remove_dir` | `(kobj)` | حذف الـ directory |
| `sysfs_rename_dir_ns` | `(kobj, new_name, new_ns) → int` | إعادة تسمية directory مع تغيير namespace |
| `sysfs_move_dir_ns` | `(kobj, new_parent, new_ns) → int` | نقل directory لـ parent جديد |
| `sysfs_create_mount_point` | `(parent_kobj, name) → int` | إنشاء نقطة mount داخل sysfs |
| `sysfs_remove_mount_point` | `(parent_kobj, name)` | حذف نقطة الـ mount |

#### File (Attribute) Management

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `sysfs_create_file` | `(kobj, attr) → int` | إنشاء attribute file (wrapper بدون namespace) |
| `sysfs_create_file_ns` | `(kobj, attr, ns) → int` | إنشاء attribute file مع namespace |
| `sysfs_create_files` | `(kobj, attrs[]) → int` | إنشاء مجموعة attributes دفعة واحدة |
| `sysfs_remove_file` | `(kobj, attr)` | حذف attribute file |
| `sysfs_remove_file_ns` | `(kobj, attr, ns)` | حذف attribute file مع namespace |
| `sysfs_remove_file_self` | `(kobj, attr) → bool` | حذف الـ file من داخل الـ show/store callback نفسه |
| `sysfs_remove_files` | `(kobj, attrs[])` | حذف مجموعة attributes |
| `sysfs_chmod_file` | `(kobj, attr, mode) → int` | تغيير permissions الـ attribute |

#### Binary Attribute Management

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `sysfs_create_bin_file` | `(kobj, bin_attr) → int` | إنشاء binary attribute file |
| `sysfs_remove_bin_file` | `(kobj, bin_attr)` | حذف binary attribute file |
| `sysfs_bin_attr_simple_read` | `(file, kobj, attr, buf, off, count) → ssize_t` | قراءة بسيطة لـ binary attr من private data |

#### Symlink Management

| Function | الغرض |
|---|---|
| `sysfs_create_link` | إنشاء symlink لـ kobject آخر |
| `sysfs_create_link_nowarn` | نفس السابق بدون warning لو الـ link موجود |
| `sysfs_remove_link` | حذف symlink |
| `sysfs_rename_link_ns` | إعادة تسمية symlink مع تغيير namespace |
| `sysfs_delete_link` | حذف symlink لو target مختلف عن المتوقع (safe) |
| `compat_only_sysfs_link_entry_to_kobj` | إنشاء symlink من entry داخل kobj لـ kobj آخر (legacy) |

#### Group Management

| Function | الغرض |
|---|---|
| `sysfs_create_group` | إنشاء attribute group كاملة (مع subdirectory اختياري) |
| `sysfs_create_groups` | إنشاء مصفوفة من الـ groups |
| `sysfs_update_group` | تحديث visibility الـ attributes داخل group موجودة |
| `sysfs_update_groups` | تحديث مصفوفة groups |
| `sysfs_remove_group` | حذف group كاملة |
| `sysfs_remove_groups` | حذف مصفوفة groups |
| `sysfs_add_file_to_group` | إضافة attribute لـ group موجودة |
| `sysfs_remove_file_from_group` | حذف attribute من group موجودة |
| `sysfs_merge_group` | دمج attrs من group في kobj الحالي بدون إنشاء subdirectory |
| `sysfs_unmerge_group` | عكس `sysfs_merge_group` |
| `sysfs_add_link_to_group` | إضافة symlink داخل group |
| `sysfs_remove_link_from_group` | حذف symlink من group |

#### Notification

| Function | الغرض |
|---|---|
| `sysfs_notify` | إرسال poll notification لـ attribute معين |
| `sysfs_notify_dirent` | إرسال notification مباشرة عبر `kernfs_node` |

#### Ownership

| Function | الغرض |
|---|---|
| `sysfs_file_change_owner` | تغيير uid/gid لـ file محدد |
| `sysfs_change_owner` | تغيير uid/gid لكل الـ kobject (dir + files) |
| `sysfs_link_change_owner` | تغيير uid/gid لـ symlink |
| `sysfs_groups_change_owner` | تغيير uid/gid لمصفوفة groups |
| `sysfs_group_change_owner` | تغيير uid/gid لـ group واحدة |

#### Output Helpers

| Function | الغرض |
|---|---|
| `sysfs_emit` | كتابة formatted string في buffer الـ show callback بشكل آمن |
| `sysfs_emit_at` | نفس `sysfs_emit` لكن من offset معين |

#### Kernfs Node Helpers (inline)

| Function | الغرض |
|---|---|
| `sysfs_enable_ns` | تفعيل namespace filtering على kernfs_node |
| `sysfs_get_dirent` | البحث عن kernfs_node بالاسم داخل parent |
| `sysfs_get` | زيادة refcount لـ kernfs_node |
| `sysfs_put` | تخفيض refcount لـ kernfs_node |
| `sysfs_break_active_protection` | كسر الحماية على node نشط (لاستخدامه في self-removal) |
| `sysfs_unbreak_active_protection` | استعادة الحماية بعد `sysfs_break_active_protection` |

#### Initialization

| Function | الغرض |
|---|---|
| `sysfs_init` | تهيئة الـ sysfs subsystem أثناء boot |
| `sysfs_attr_init` | تهيئة dynamic attribute (lockdep) |
| `sysfs_bin_attr_init` | تهيئة dynamic binary attribute (lockdep) |

---

### Group 1: Directory Management

هذه الـ functions مسؤولة عن إدارة الـ directories داخل sysfs. كل `kobject` يمثل directory واحدة، والـ functions دي بتنشئ وتحذف وتنقل وتسمي هذه الـ directories. الـ namespace (`ns`) هنا بيُستخدم لعزل directories بين network namespaces مختلفة.

---

#### `sysfs_create_dir_ns`

```c
int __must_check sysfs_create_dir_ns(struct kobject *kobj, const void *ns);
```

بتنشئ directory جديدة في sysfs تمثل الـ `kobject` المعطى. الـ directory بتتنشأ كـ child للـ parent kobject أو في root لو مافيش parent. الـ `ns` بيُستخدم للـ namespace tagging على الـ kernfs node.

**Parameters:**
- `kobj` — الـ kobject المراد إنشاء directory له. `kobj->parent` بيحدد الـ parent directory.
- `ns` — مؤشر namespace (مثلاً `struct net *`)؛ `NULL` لو مافيش namespace isolation.

**Return:** `0` عند النجاح، error code سالب (مثل `-EEXIST`, `-ENOMEM`) عند الفشل.

**Key Details:**
- `__must_check` — لازم تفحص الـ return value دايمًا.
- بيستدعي `kernfs_create_dir_ns` داخليًا.
- الـ caller بيكون دايمًا `kobject_add()` في `lib/kobject.c`.

---

#### `sysfs_remove_dir`

```c
void sysfs_remove_dir(struct kobject *kobj);
```

بتحذف الـ directory المرتبطة بالـ `kobject` من sysfs. بتمسح كل الـ attributes والـ links الموجودة جوا الـ directory قبل الحذف.

**Parameters:**
- `kobj` — الـ kobject المراد حذف directory-ه.

**Key Details:**
- بتفصل `kobj->sd` (الـ `kernfs_node`) عن الـ kobject وبتشيل reference.
- آمنة للاستدعاء لو `kobj->sd` كان `NULL` (no-op).
- الـ caller الأساسي هو `kobject_del()`.

---

#### `sysfs_rename_dir_ns`

```c
int __must_check sysfs_rename_dir_ns(struct kobject *kobj,
                                     const char *new_name,
                                     const void *new_ns);
```

بتغير اسم الـ directory المرتبطة بالـ kobject ومؤشر الـ namespace بتاعها في نفس الوقت.

**Parameters:**
- `kobj` — الـ kobject المراد إعادة تسميته.
- `new_name` — الاسم الجديد للـ directory.
- `new_ns` — الـ namespace pointer الجديد.

**Return:** `0` عند النجاح، `-EEXIST` لو الاسم موجود، `-ENOMEM`.

**Key Details:**
- بتستدعي `kernfs_rename_ns` داخليًا.
- الـ caller الأساسي هو `kobject_rename()`.

---

#### `sysfs_move_dir_ns`

```c
int __must_check sysfs_move_dir_ns(struct kobject *kobj,
                                   struct kobject *new_parent_kobj,
                                   const void *new_ns);
```

بتنقل directory الـ kobject لتصبح child لـ parent جديد مع تحديث الـ namespace.

**Parameters:**
- `kobj` — الـ kobject المراد نقله.
- `new_parent_kobj` — الـ parent الجديد؛ `NULL` يعني الـ sysfs root.
- `new_ns` — الـ namespace الجديد.

**Return:** `0` أو error code.

**Key Details:**
- بتستدعي `kernfs_rename_ns` مع تغيير الـ parent.
- الـ caller الأساسي هو `kobject_move()`.

---

#### `sysfs_create_mount_point` / `sysfs_remove_mount_point`

```c
int __must_check sysfs_create_mount_point(struct kobject *parent_kobj,
                                          const char *name);
void sysfs_remove_mount_point(struct kobject *parent_kobj, const char *name);
```

بتنشئ directory فاضية داخل sysfs مخصصة كـ mount point لـ filesystem تاني (مثل `cgroup`، `pstore`). مش مرتبطة بـ kobject خاص.

**Parameters:**
- `parent_kobj` — الـ parent kobject اللي هتتنشأ فيه الـ directory.
- `name` — اسم الـ directory الجديدة.

**Key Details:**
- بتستخدم `kernfs_create_empty_dir` داخليًا.
- الـ directory دي معنهاش attributes — هي بس "hook point" لـ filesystem تاني.

---

### Group 2: File (Attribute) Management

الـ files في sysfs هي الـ attributes — كل file بيمثل attribute واحدة للـ kobject. الـ read/write على الـ file بيتوجه لـ `show`/`store` callbacks المسجلة في الـ `sysfs_ops` أو في الـ `attribute` مباشرة (في حالة device attributes).

---

#### `sysfs_create_file_ns`

```c
int __must_check sysfs_create_file_ns(struct kobject *kobj,
                                      const struct attribute *attr,
                                      const void *ns);
```

بتنشئ file داخل directory الـ kobject تمثل attribute واحدة مع دعم namespace.

**Parameters:**
- `kobj` — الـ kobject اللي هيتنشأ فيه الـ file.
- `attr` — الـ attribute (بتحتوي على name وmode والـ lockdep key).
- `ns` — الـ namespace pointer؛ `NULL` في الغالب.

**Return:** `0` أو error code سالب.

**Key Details:**
- بتتحقق إن `attr->mode` مش writable بـ world (0-bit).
- بتستدعي `kernfs_create_file_ns` داخليًا بعد ما تربط `kobj->sd` كـ parent.
- **Locking:** بتحتاج `kobj->sd` يكون valid (الـ kobject لازم يكون مضاف لـ sysfs قبل الاستدعاء).

---

#### `sysfs_create_file` (inline wrapper)

```c
static inline int __must_check sysfs_create_file(struct kobject *kobj,
                                                  const struct attribute *attr)
{
    return sysfs_create_file_ns(kobj, attr, NULL);
}
```

الـ wrapper الأكثر استخداماً — بتستدعي `sysfs_create_file_ns` بـ `ns = NULL`.

---

#### `sysfs_create_files`

```c
int __must_check sysfs_create_files(struct kobject *kobj,
                                    const struct attribute * const *attr);
```

بتنشئ مجموعة attributes دفعة واحدة. لو فشل إنشاء أي attribute، بترجع وبتحذف كل اللي نشأ قبلها (rollback كامل).

**Parameters:**
- `kobj` — الـ kobject المستهدف.
- `attr` — مصفوفة من pointers للـ attributes، منتهية بـ `NULL`.

**Return:** `0` أو error code مع rollback تلقائي.

**Pseudocode:**
```
for each attr in attrs[]:
    ret = sysfs_create_file(kobj, *attr)
    if ret:
        while attr > attrs[0]:
            attr--
            sysfs_remove_file(kobj, *attr)
        return ret
return 0
```

---

#### `sysfs_remove_file` / `sysfs_remove_file_ns` / `sysfs_remove_files`

```c
static inline void sysfs_remove_file(struct kobject *kobj,
                                     const struct attribute *attr);
void sysfs_remove_file_ns(struct kobject *kobj,
                          const struct attribute *attr, const void *ns);
void sysfs_remove_files(struct kobject *kobj,
                        const struct attribute * const *attr);
```

بتحذف attribute file من sysfs. `sysfs_remove_file` هي wrapper بدون namespace. `sysfs_remove_files` بتحذف مصفوفة كاملة.

**Key Details:**
- بتستدعي `kernfs_remove_by_name_ns` داخليًا.
- آمنة للاستدعاء لو الـ file مش موجودة (بتعمل no-op).
- **لا ترجع error** — الـ caller مش بيحتاج يتحقق.

---

#### `sysfs_remove_file_self`

```c
bool sysfs_remove_file_self(struct kobject *kobj,
                            const struct attribute *attr);
```

بتحذف الـ attribute file من داخل الـ `show` أو `store` callback بتاعها نفسها. ده الحل الوحيد الآمن لـ self-removal بدون deadlock.

**Return:** `true` لو النجاح، `false` لو الـ kernfs_node مش موجود.

**Key Details:**
- بتستدعي `kernfs_remove_self` داخليًا اللي بتتعامل مع الـ active reference بشكل خاص.
- بدونها، الـ `sysfs_remove_file` العادية هتـ deadlock لأن الـ active reference موجودة.
- الـ caller: أي driver بيريد يشيل attribute بعد عملية واحدة (one-shot attributes).

---

#### `sysfs_chmod_file`

```c
int __must_check sysfs_chmod_file(struct kobject *kobj,
                                  const struct attribute *attr,
                                  umode_t mode);
```

بتغير permissions الـ attribute file في runtime.

**Parameters:**
- `kobj` — الـ kobject المالك.
- `attr` — الـ attribute المراد تغيير permissions-ها.
- `mode` — الـ mode الجديد (مثلاً `0644`).

**Return:** `0` أو error code.

**Key Details:**
- بتستدعي `kernfs_setattr` داخليًا.
- الـ `VERIFY_OCTAL_PERMISSIONS` macro بتتحقق من الـ mode وقت compile في الـ `__ATTR` macros، لكن `sysfs_chmod_file` runtime بتتحقق يدويًا.

---

#### `sysfs_break_active_protection` / `sysfs_unbreak_active_protection`

```c
struct kernfs_node *sysfs_break_active_protection(struct kobject *kobj,
                                                  const struct attribute *attr);
void sysfs_unbreak_active_protection(struct kernfs_node *kn);
```

بيسمحوا لـ driver يحذف attribute من داخل الـ callback بتاعها (self-removal pattern أكثر تحكماً). `sysfs_break_active_protection` بتشيل الـ active reference اللي الـ kernfs حاطتها وبترجع الـ kernfs_node. `sysfs_unbreak_active_protection` بتستعيد الحماية.

**Key Details:**
- بتستدعيا `kernfs_break_active_protection` و`kernfs_unbreak_active_protection`.
- لو مش بتستخدم `sysfs_remove_file_self`، استخدم دول مع `sysfs_remove_file`.
- **Caller context:** must be called from within the attribute's show/store callback.

---

### Group 3: Binary Attribute Management

الـ `bin_attribute` هي attribute خاصة بتدعم قراءة/كتابة binary data (مش text). مثلاً firmware، EEPROM data، أو ACPI tables. بتدعم `read`، `write`، `mmap`، و`llseek` callbacks.

---

#### `sysfs_create_bin_file`

```c
int __must_check sysfs_create_bin_file(struct kobject *kobj,
                                       const struct bin_attribute *attr);
```

بتنشئ binary attribute file في directory الـ kobject.

**Parameters:**
- `kobj` — الـ kobject المستهدف.
- `attr` — الـ `bin_attribute` اللي بتحتوي على name وsize وread/write/mmap callbacks.

**Return:** `0` أو error code.

**Key Details:**
- مش بتدعم namespace مباشرة (بتستخدم `sysfs_create_file_ns` مع special ops).
- الـ `attr->size` بيحدد الـ size المرئي في `ls -la`، بس مش بيقيد القراءة/الكتابة الفعلية.
- الـ `attr->f_mapping` callback لو موجودة بتُستخدم لـ mmap support.

---

#### `sysfs_remove_bin_file`

```c
void sysfs_remove_bin_file(struct kobject *kobj,
                           const struct bin_attribute *attr);
```

بتحذف binary attribute file. آمنة لو الـ file مش موجودة.

---

#### `sysfs_bin_attr_simple_read`

```c
ssize_t sysfs_bin_attr_simple_read(struct file *file, struct kobject *kobj,
                                   const struct bin_attribute *attr,
                                   char *buf, loff_t off, size_t count);
```

Implementation جاهزة لـ `read` callback لـ binary attributes اللي بياناتها موجودة في `attr->private`.

**Key Details:**
- بتعمل `memory_read_from_buffer` داخليًا — بتتعامل مع الـ offset والـ count بشكل صحيح.
- الـ `BIN_ATTR_SIMPLE_RO` و`BIN_ATTR_SIMPLE_ADMIN_RO` macros بتستخدمها مباشرة.
- الـ driver بس يحتاج يضع data pointer في `attr->private` وبيحدد `attr->size`.

---

### Group 4: Symlink Management

الـ symlinks في sysfs بتمثل relationships بين kobjects — مثلاً `/sys/class/net/eth0` هو symlink يشير لـ `/sys/devices/pci.../net/eth0`. الـ namespace بيتحكم في visibility الـ symlinks.

---

#### `sysfs_create_link`

```c
int __must_check sysfs_create_link(struct kobject *kobj,
                                   struct kobject *target,
                                   const char *name);
```

بتنشئ symlink باسم `name` داخل directory الـ `kobj`، بيشير لـ directory الـ `target`.

**Parameters:**
- `kobj` — الـ kobject اللي هيتنشأ فيه الـ symlink.
- `target` — الـ kobject المراد الإشارة إليه.
- `name` — اسم الـ symlink.

**Return:** `0` أو `-EEXIST` أو `-ENOMEM`.

**Key Details:**
- لو الـ symlink موجود بالفعل، بتعمل WARN وبترجع `-EEXIST`.
- بتستدعي `sysfs_do_create_link_sd` داخليًا.
- الـ caller الأساسي: `kobject_add_internal` لإنشاء back-links.

---

#### `sysfs_create_link_nowarn`

```c
int __must_check sysfs_create_link_nowarn(struct kobject *kobj,
                                          struct kobject *target,
                                          const char *name);
```

مثل `sysfs_create_link` بالظبط لكن من غير warning لو الـ link موجود. مفيدة في حالات الـ race conditions المقصودة أو للـ driver initialization اللي ممكن يتكرر.

---

#### `sysfs_remove_link`

```c
void sysfs_remove_link(struct kobject *kobj, const char *name);
```

بتحذف symlink باسم `name` من directory الـ `kobj`. بتستدعي `kernfs_remove_by_name` داخليًا.

---

#### `sysfs_rename_link_ns`

```c
int sysfs_rename_link_ns(struct kobject *kobj, struct kobject *target,
                         const char *old_name, const char *new_name,
                         const void *new_ns);
```

بتغير اسم symlink ومؤشر الـ namespace بتاعه. بتتحقق إن الـ symlink الموجود يشير للـ `target` المحدد قبل ما تعمل rename.

**Parameters:**
- `kobj` — الـ directory المحتوية على الـ symlink.
- `target` — الـ target المتوقع للـ symlink (للتحقق).
- `old_name` — الاسم الحالي.
- `new_name` — الاسم الجديد.
- `new_ns` — الـ namespace الجديد.

---

#### `sysfs_delete_link`

```c
void sysfs_delete_link(struct kobject *dir, struct kobject *targ,
                       const char *name);
```

بتحذف symlink من `dir` باسم `name` بس لو بيشير للـ `targ` المحدد. آمنة لو التارجت اتغير (بتتحقق من الـ target قبل الحذف عبر kernel NS).

**Key Details:**
- مفيدة في cleanup paths اللي ممكن تتنفذ بعد removal الـ target نفسه.

---

#### `compat_only_sysfs_link_entry_to_kobj`

```c
int compat_only_sysfs_link_entry_to_kobj(struct kobject *kobj,
                                         struct kobject *target_kobj,
                                         const char *target_name,
                                         const char *symlink_name);
```

بتنشئ symlink من `symlink_name` في `kobj` يشير لـ entry اسمه `target_name` داخل `target_kobj`. مخصصة للـ backwards compatibility paths فقط (`compat_only` في الاسم دليل على ده).

---

### Group 5: Attribute Group Management

الـ `attribute_group` بتسمح بتجميع مجموعة attributes (text وbinary) وإنشاؤها كلها دفعة واحدة. لو الـ group عندها `name`، بتتنشأ في subdirectory. الـ `is_visible` callback بتحكم في visibility كل attribute ديناميكيًا.

---

#### `sysfs_create_group`

```c
int __must_check sysfs_create_group(struct kobject *kobj,
                                    const struct attribute_group *grp);
```

بتنشئ attribute group كاملة. لو `grp->name` موجود، بتنشئ subdirectory بالاسم ده. بتستدعي `is_visible` / `is_bin_visible` على كل attribute قبل إنشاؤها — الـ attribute اللي الـ callback ترجعلها `0` مش بتتنشأ.

**Parameters:**
- `kobj` — الـ kobject المستهدف.
- `grp` — الـ group descriptor بالـ attrs وbin_attrs وcallbacks.

**Return:** `0` أو error code مع rollback كامل.

**Pseudocode:**
```
if grp->name:
    parent_kn = kernfs_create_dir(kobj->sd, grp->name)
else:
    parent_kn = kobj->sd

for each attr in grp->attrs:
    mode = grp->is_visible ? grp->is_visible(kobj, attr, i) : attr->mode
    if mode == 0: skip
    if mode == SYSFS_GROUP_INVISIBLE: hide directory
    kernfs_create_file(parent_kn, attr->name, mode, ...)

for each bin_attr in grp->bin_attrs:
    ... similar ...
```

---

#### `sysfs_create_groups`

```c
int __must_check sysfs_create_groups(struct kobject *kobj,
                                     const struct attribute_group **groups);
```

بتنشئ مصفوفة من الـ groups دفعة واحدة مع rollback كامل لو فشل أي group.

**Parameters:**
- `groups` — مصفوفة من pointers لـ `attribute_group`، منتهية بـ `NULL`.

---

#### `sysfs_update_group` / `sysfs_update_groups`

```c
int sysfs_update_group(struct kobject *kobj,
                       const struct attribute_group *grp);
int __must_check sysfs_update_groups(struct kobject *kobj,
                                     const struct attribute_group **groups);
```

بتعيد تقييم `is_visible` لكل attribute في الـ group وبتضيف أو تحذف attributes بناءً على النتيجة. مفيدة لما تتغير حالة الـ hardware ويتغير معها visibility الـ attributes.

**Key Details:**
- مش بتحذف وتعيد إنشاء الـ group — بس بتضيف/تحذف الـ attributes اللي اتغير حالها.
- الـ `sysfs_update_groups` بتعمل loop على كل الـ groups.

---

#### `sysfs_remove_group` / `sysfs_remove_groups`

```c
void sysfs_remove_group(struct kobject *kobj,
                        const struct attribute_group *grp);
void sysfs_remove_groups(struct kobject *kobj,
                         const struct attribute_group **groups);
```

بتحذف group كاملة مع الـ subdirectory لو موجود. آمنة للاستدعاء حتى لو الـ group مش موجودة (no-op).

---

#### `sysfs_merge_group` / `sysfs_unmerge_group`

```c
int sysfs_merge_group(struct kobject *kobj,
                      const struct attribute_group *grp);
void sysfs_unmerge_group(struct kobject *kobj,
                         const struct attribute_group *grp);
```

بتضيف الـ attrs من `grp` مباشرة في directory الـ `kobj` (مش في subdirectory حتى لو `grp->name` موجود). مفيدة لـ mixin pattern — لما عايز تضيف مجموعة attributes لـ kobject موجود من subsystem تاني.

**Key Details:**
- `sysfs_merge_group` بتتجاهل `grp->name` — بتنشئ الـ files في الـ kobj نفسه.
- `sysfs_unmerge_group` بتشيلها.

---

#### `sysfs_add_file_to_group` / `sysfs_remove_file_from_group`

```c
int sysfs_add_file_to_group(struct kobject *kobj,
                            const struct attribute *attr,
                            const char *group);
void sysfs_remove_file_from_group(struct kobject *kobj,
                                  const struct attribute *attr,
                                  const char *group);
```

بتضيف/تحذف attribute واحدة من/إلى group موجودة بالاسم. الـ group لازم تكون موجودة بالفعل (مع subdirectory).

**Parameters:**
- `group` — اسم الـ group (اسم الـ subdirectory)؛ `NULL` لو الـ attrs في الـ kobject مباشرة.

---

#### `sysfs_add_link_to_group` / `sysfs_remove_link_from_group`

```c
int sysfs_add_link_to_group(struct kobject *kobj,
                            const char *group_name,
                            struct kobject *target,
                            const char *link_name);
void sysfs_remove_link_from_group(struct kobject *kobj,
                                  const char *group_name,
                                  const char *link_name);
```

بتضيف/تحذف symlink من داخل group معينة. مفيدة لـ subsystems اللي بتدير relationships بين kobjects داخل named attribute groups.

---

### Group 6: Notification

---

#### `sysfs_notify`

```c
void sysfs_notify(struct kobject *kobj, const char *dir, const char *attr);
```

بتبعث poll notification لـ attribute معينة — أي process بيعمل `poll()` أو `select()` على الـ sysfs file ده هيستيقظ.

**Parameters:**
- `kobj` — الـ kobject المالك.
- `dir` — اسم الـ subdirectory لو الـ attribute في subdirectory؛ `NULL` لو في الـ kobj مباشرة.
- `attr` — اسم الـ attribute file.

**Key Details:**
- بتستدعي `kernfs_notify` داخليًا اللي بتستخدم `kernfs_open_node` وبتعمل `wake_up_interruptible`.
- مش بتضمن الترتيب — لو اتنين threads بيعملوا notify في نفس الوقت، الـ events ممكن تتدمج.
- **Caller context:** يمكن استدعاؤها من interrupt context.

---

#### `sysfs_notify_dirent` (inline)

```c
static inline void sysfs_notify_dirent(struct kernfs_node *kn)
{
    kernfs_notify(kn);
}
```

نفس `sysfs_notify` لكن بتاخد `kernfs_node` مباشرة بدل ما تبحث عنه. أسرع لما يكون عندك الـ `kn` بالفعل (بتفضل الـ `kn` pointer بدل البحث في كل مرة).

---

### Group 7: Ownership Management

هذه الـ functions مهمة خصوصاً في user namespaces — لما container بيستخدم sysfs، الـ files بتحتاج تكون مملوكة لـ UID/GID صح داخل الـ namespace.

---

#### `sysfs_change_owner`

```c
int sysfs_change_owner(struct kobject *kobj, kuid_t kuid, kgid_t kgid);
```

بتغير UID/GID للـ kobject directory وكل الـ files والـ symlinks الموجودة فيها.

**Parameters:**
- `kobj` — الـ kobject المستهدف.
- `kuid` — الـ UID الجديد (mapped kernel UID).
- `kgid` — الـ GID الجديد.

**Return:** `0` أو error code.

**Key Details:**
- بتعمل loop على كل children الـ kobject.
- الـ caller الأساسي: `device_change_owner()` في `drivers/base/core.c`.

---

#### `sysfs_file_change_owner`

```c
int sysfs_file_change_owner(struct kobject *kobj,
                            const char *name,
                            kuid_t kuid, kgid_t kgid);
```

بتغير ownership لـ file محدد بالاسم داخل directory الـ kobject.

---

#### `sysfs_link_change_owner`

```c
int sysfs_link_change_owner(struct kobject *kobj, struct kobject *targ,
                            const char *name, kuid_t kuid, kgid_t kgid);
```

بتغير ownership لـ symlink محدد داخل directory الـ `kobj`.

---

#### `sysfs_groups_change_owner` / `sysfs_group_change_owner`

```c
int sysfs_groups_change_owner(struct kobject *kobj,
                              const struct attribute_group **groups,
                              kuid_t kuid, kgid_t kgid);
int sysfs_group_change_owner(struct kobject *kobj,
                             const struct attribute_group *groups,
                             kuid_t kuid, kgid_t kgid);
```

بتغير ownership لكل الـ files داخل مصفوفة groups أو group واحدة.

**Key Details:**
- الكل بيستدعي `kernfs_setattr` داخليًا على كل node.
- مهمة جداً في `user namespace` و`cgroup` scenarios.

---

### Group 8: Output Helpers

---

#### `sysfs_emit`

```c
__printf(2, 3)
int sysfs_emit(char *buf, const char *fmt, ...);
```

بتكتب formatted string في الـ `buf` المعطى (الـ page buffer لـ show callback). آمنة لأنها بتتحقق إن الـ `buf` هو فعلاً page-aligned kernel page (الـ sysfs buffer الصح).

**Parameters:**
- `buf` — الـ buffer المعطى للـ `show` callback (حجمه `PAGE_SIZE`).
- `fmt` — format string مثل `printf`.
- `...` — arguments.

**Return:** عدد الـ bytes المكتوبة أو error code.

**Key Details:**
- بتتحقق داخليًا إن `buf` هو `PAGE_SIZE`-aligned page باستخدام `WARN_ON_ONCE`.
- بتحدد الـ output بـ `PAGE_SIZE - 1` تلقائياً لتجنب overflow.
- **الاستخدام الصحيح الوحيد** داخل `show` callbacks — لا تستخدم `sprintf` أو `snprintf` مباشرة.

**مثال عملي:**
```c
static ssize_t my_attr_show(struct device *dev,
                            struct device_attribute *attr, char *buf)
{
    struct my_dev *mdev = dev_get_drvdata(dev);
    return sysfs_emit(buf, "%d\n", mdev->value);
}
```

---

#### `sysfs_emit_at`

```c
__printf(3, 4)
int sysfs_emit_at(char *buf, int at, const char *fmt, ...);
```

بتكتب formatted string في `buf` ابتداءً من offset `at`. مفيدة لبناء output تدريجي في الـ `show` callback.

**Parameters:**
- `buf` — نفس buffer الـ show callback.
- `at` — الـ offset داخل الـ buffer للبداية من عنده.
- `fmt` — format string.

**Return:** عدد الـ bytes المكتوبة في هذه الاستدعاء.

**Key Details:**
- بتتحقق إن `at` في نطاق `[0, PAGE_SIZE-1]`.
- بتحدد الكتابة بـ `PAGE_SIZE - at - 1`.

**مثال عملي:**
```c
static ssize_t multi_attr_show(struct device *dev,
                               struct device_attribute *attr, char *buf)
{
    int count = 0;
    count += sysfs_emit_at(buf, count, "value1: %d\n", val1);
    count += sysfs_emit_at(buf, count, "value2: %d\n", val2);
    return count;
}
```

---

### Group 9: Kernfs Node Helpers (inline)

---

#### `sysfs_enable_ns`

```c
static inline void sysfs_enable_ns(struct kernfs_node *kn)
{
    return kernfs_enable_ns(kn);
}
```

بتفعّل namespace filtering على kernfs_node محدد. بعد الاستدعاء، الـ node ده بيظهر بس للـ processes اللي namespace بتاعهم مطابق.

**Key Details:**
- لازم تتستدعى قبل ما الـ node يتفعّل (activated).
- الـ caller الأساسي: `sysfs_create_dir_ns` عبر `kobject_add`.

---

#### `sysfs_get_dirent`

```c
static inline struct kernfs_node *sysfs_get_dirent(struct kernfs_node *parent,
                                                    const char *name)
{
    return kernfs_find_and_get(parent, name);
}
```

بتبحث عن kernfs_node باسمه داخل parent وبتزود الـ refcount بتاعه.

**Return:** `kernfs_node *` أو `NULL` لو مش موجود.

**Key Details:**
- لازم تستدعي `sysfs_put` على النتيجة لما تخلص.
- الـ `kernfs_find_and_get` بتعمل RCU lookup.

---

#### `sysfs_get` / `sysfs_put`

```c
static inline struct kernfs_node *sysfs_get(struct kernfs_node *kn)
{
    kernfs_get(kn);
    return kn;
}

static inline void sysfs_put(struct kernfs_node *kn)
{
    kernfs_put(kn);
}
```

reference counting للـ kernfs_node. `sysfs_get` بتزود الـ count وبترجع الـ node. `sysfs_put` بتخفض الـ count وبتحذف الـ node لو وصل للصفر.

---

### Group 10: Macros للـ Attribute Declaration

---

#### `__ATTR` Family

```c
#define __ATTR(_name, _mode, _show, _store) { ... }
#define __ATTR_RO(_name)          /* 0444, _name##_show */
#define __ATTR_WO(_name)          /* 0200, _name##_store */
#define __ATTR_RW(_name)          /* 0644, _name##_show, _name##_store */
#define __ATTR_RO_MODE(_name, _mode)
#define __ATTR_RW_MODE(_name, _mode)
#define __ATTR_PREALLOC(_name, _mode, _show, _store)
#define __ATTR_IGNORE_LOCKDEP(_name, _mode, _show, _store)
```

macros لإنشاء `kobj_attribute` / `device_attribute` بشكل مختصر. `VERIFY_OCTAL_PERMISSIONS` بتعمل compile-time check على الـ permissions.

**مثال:**
```c
/* Expands to a compound literal initializing struct kobj_attribute */
static struct kobj_attribute my_attr = __ATTR_RW(my_value);
/* Requires: my_value_show() and my_value_store() to exist */
```

---

#### `ATTRIBUTE_GROUPS` / `BIN_ATTRIBUTE_GROUPS`

```c
#define ATTRIBUTE_GROUPS(_name)     /* Creates _name_group + _name_groups[] */
#define BIN_ATTRIBUTE_GROUPS(_name) /* Creates binary version */
```

بتنشئ static `attribute_group` و`attribute_group *[]` من `_name##_attrs` array. الـ `_Generic` في `ATTRIBUTE_GROUPS` بيدعم كلاً من `struct attribute **` و`const struct attribute *const *`.

**مثال:**
```c
static struct attribute *my_attrs[] = {
    &dev_attr_value.attr,
    NULL,
};
ATTRIBUTE_GROUPS(my);
/* Creates: my_group + my_groups[] ready for device registration */
```

---

#### `sysfs_attr_init` / `sysfs_bin_attr_init`

```c
#define sysfs_attr_init(attr) do { (attr)->key = &__key; } while (0)
#define sysfs_bin_attr_init(bin_attr) sysfs_attr_init(&(bin_attr)->attr)
```

لازم تتستدعى على أي attribute متخصصة dynamically (مش static). بتعطي الـ lockdep key الفريد للـ attribute. بدونها، lockdep هيطلع warning أو error عند إضافة الـ attribute لـ sysfs.

**Key Details:**
- No-op لو `CONFIG_DEBUG_LOCK_ALLOC` مش مفعّل.
- `__key` بتكون `static` داخل الـ macro — يعني كل call site ليه key فريد.

---

#### `DEFINE_SYSFS_GROUP_VISIBLE` / `SYSFS_GROUP_VISIBLE`

```c
#define DEFINE_SYSFS_GROUP_VISIBLE(name)   /* Defines sysfs_group_visible_##name() */
#define SYSFS_GROUP_VISIBLE(fn)            /* = sysfs_group_visible_##fn */
```

pattern لـ dynamic group visibility بيجمع بين visibility الـ group نفسها وvisibility كل attribute فيها. بتعمل wiring بين `name##_group_visible(kobj)` و`name##_attr_visible(kobj, attr, n)`.

**Pseudocode للـ generated function:**
```c
static inline umode_t sysfs_group_visible_example(
    struct kobject *kobj, struct attribute *attr, int n)
{
    if (n == 0 && !example_group_visible(kobj))
        return SYSFS_GROUP_INVISIBLE; /* Hide the whole directory */
    return example_attr_visible(kobj, attr, n);
}
```

**Key Details:**
- `n == 0` check خاص — بيتستدعى على أول attribute علشان يقرر visibility الـ directory نفسها.
- `SYSFS_GROUP_INVISIBLE` (020000) بيخبر sysfs إن الـ group directory كلها مش مرئية.
- `DEFINE_SIMPLE_SYSFS_GROUP_VISIBLE` للحالة البسيطة اللي كل attributes بنفس visibility الـ group.

---

#### `__BIN_ATTR` Family

```c
#define __BIN_ATTR(_name, _mode, _read, _write, _size) { ... }
#define __BIN_ATTR_RO(_name, _size)
#define __BIN_ATTR_WO(_name, _size)
#define __BIN_ATTR_RW(_name, _size)
#define __BIN_ATTR_ADMIN_RO(_name, _size)   /* mode 0400 */
#define __BIN_ATTR_ADMIN_RW(_name, _size)   /* mode 0600 */
#define BIN_ATTR(_name, _mode, _read, _write, _size)
#define BIN_ATTR_RO(_name, _size)
#define BIN_ATTR_SIMPLE_RO(_name)           /* Uses sysfs_bin_attr_simple_read */
#define BIN_ATTR_SIMPLE_ADMIN_RO(_name)
```

macros لإنشاء `bin_attribute` بشكل مختصر. `ADMIN_*` variants بتستخدم `0400`/`0600` بدل `0444`/`0644`. `SIMPLE_RO` variants بتستخدم `sysfs_bin_attr_simple_read` مباشرة.

---

#### `VERIFY_OCTAL_PERMISSIONS`

```c
#define VERIFY_OCTAL_PERMISSIONS(perms) (BUILD_BUG_ON_ZERO(...) + (perms))
```

compile-time macro بتتحقق من صحة permissions:
- قيمة بين `0` و`0777`.
- `user_readable >= group_readable >= other_readable`.
- `user_writable >= group_writable`.
- **مش writable by others** (bit 1 لازم يكون صفر).

لو أي شرط اتكسر → compile error مع `BUILD_BUG_ON_ZERO`.

---

### Group 11: Initialization

---

#### `sysfs_init`

```c
int __must_check sysfs_init(void);
```

بتهيئ الـ sysfs subsystem أثناء kernel boot. بتنشئ الـ `kernfs_root` الخاص بـ sysfs وبتعمل mount.

**Key Details:**
- بتتستدعى مرة واحدة بس من `mnt_init()` في `fs/namespace.c`.
- لو فشلت → kernel panic (critical path).
- الـ `!CONFIG_SYSFS` stub بترجع `0` مباشرة.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ **sysfs** نفسه متعلق بالـ **kernfs** تحته، ومفيش debugfs entries خاصة بـ sysfs مباشرة، لكن الـ kernfs بيوفر معلومات من خلال `/sys/kernel/debug/` لو `CONFIG_DEBUG_FS=y`.

| Entry | الوصف | إزاي تقراه |
|---|---|---|
| `/sys/kernel/debug/kernfs/` | مش موجود بشكل standard لكن بعض subsystems بتضيف entries | `ls /sys/kernel/debug/` |
| `/proc/mounts` | بيوريك إن sysfs متماونت فين | `grep sysfs /proc/mounts` |
| `/proc/slabinfo` | بيوريك استخدام الـ slab لـ kernfs_node objects | `grep kernfs /proc/slabinfo` |

```bash
# اعرف كل sysfs mounts
grep sysfs /proc/mounts

# اعرف عدد kernfs_node objects في الـ slab
grep -E 'kernfs|sysfs' /proc/slabinfo

# اعرف حجم الـ sysfs tree الكلي
find /sys -maxdepth 5 | wc -l
```

---

#### 2. مدخلات الـ sysfs المهمة للـ debugging

الـ sysfs نفسه هو الـ filesystem، فكل حاجة موجودة فيه هي entry. الـ entries المهمة للـ debugging:

| المسار | الوصف |
|---|---|
| `/sys/kernel/` | kernel-level attributes وـ kset entries |
| `/sys/bus/<bus>/drivers/<drv>/` | driver attributes |
| `/sys/devices/<dev>/` | device attributes |
| `/sys/class/<class>/<dev>/` | class device attributes |
| `/sys/module/<mod>/parameters/` | module parameters قابلة للتعديل |
| `/sys/kernel/mm/` | memory management attributes |
| `/sys/kernel/profiling` | kernel profiling state |

```bash
# اعرض كل attributes لـ device معين
ls -la /sys/devices/platform/<device>/

# اقرأ attribute معين
cat /sys/bus/platform/drivers/<driver>/bind

# اعرض permissions الـ attribute
stat /sys/class/net/eth0/operstate

# تحقق من وجود binary attributes
ls -la /sys/bus/pci/devices/0000:00:00.0/

# اعرض كل attribute groups لـ device
find /sys/devices/ -name "uevent" -ls 2>/dev/null | head -20
```

---

#### 3. استخدام الـ ftrace

الـ **ftrace** بيوفر tracepoints جاهزة للـ sysfs/kernfs:

```bash
# اعرض كل events متعلقة بـ sysfs
ls /sys/kernel/debug/tracing/events/ | grep -i sysfs

# اعرض كل events متعلقة بـ kobject
ls /sys/kernel/debug/tracing/events/kobject/

# فعّل كل kobject events
echo 1 > /sys/kernel/debug/tracing/events/kobject/enable

# فعّل tracing لـ sysfs_create_file و sysfs_remove_file
echo 'sysfs_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي عايز تتابعها
modprobe <module>

# اقرأ النتايج
cat /sys/kernel/debug/tracing/trace | head -50

# أوقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**تتبع `sysfs_notify` بالتحديد:**

```bash
# trace سطر بسطر لأي kobject notification
echo 'sysfs_notify' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

**تتبع الـ kernfs read/write paths:**

```bash
# trace كل قراءات/كتابات sysfs attributes
echo 'kernfs_fop_read_iter kernfs_fop_write_iter' > \
    /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. تفعيل الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لكل sysfs messages
echo 'file fs/sysfs/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file fs/kernfs/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug لـ kobject
echo 'file lib/kobject.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file lib/kobject_uevent.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug لكل attributes operations
echo 'module <module_name> +p' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ dynamic debug control table الحالي
cat /sys/kernel/debug/dynamic_debug/control | grep sysfs

# فعّل كل الـ debug messages في subsystem معين
echo 'file drivers/<subsys>/*.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p=print, f=function name, l=line number, m=module name, t=thread id
```

**إضافة printk في driver للـ debugging:**

```c
/* في show() callback — تتبع كل قراءة للـ attribute */
static ssize_t my_attr_show(struct kobject *kobj,
                             struct attribute *attr, char *buf)
{
    pr_debug("%s: reading attr %s from kobj %s\n",
             __func__, attr->name, kobject_name(kobj));
    /* ... */
}

/* في store() callback — تتبع كل كتابة */
static ssize_t my_attr_store(struct kobject *kobj,
                              struct attribute *attr,
                              const char *buf, size_t count)
{
    pr_debug("%s: writing %zu bytes to attr %s\n",
             __func__, count, attr->name);
    /* ... */
}
```

---

#### 5. Kernel config options للـ debugging

| Config | الوظيفة |
|---|---|
| `CONFIG_DEBUG_LOCK_ALLOC` | بيفعّل lockdep على كل `struct attribute` — بيكتشف lock order violations |
| `CONFIG_LOCKDEP` | الـ lock dependency engine — متطلب لـ `CONFIG_DEBUG_LOCK_ALLOC` |
| `CONFIG_SYSFS` | الـ sysfs subsystem نفسه — لازم يكون مفعّل |
| `CONFIG_DEBUG_FS` | بيفعّل debugfs — مهم جداً للـ debugging |
| `CONFIG_KOBJECT_UEVENT` | بيفعّل uevent notifications |
| `CONFIG_DEBUG_KOBJECT` | بيطبع debug messages لكل kobject operations |
| `CONFIG_PROVE_LOCKING` | بيكتشف deadlocks محتملة في الـ lock hierarchy |
| `CONFIG_DEBUG_OBJECTS_WORK` | بيكتشف use-after-free في workqueue objects |
| `CONFIG_SLUB_DEBUG` | بيساعد في كتشاف corruption في kernfs_node allocations |
| `CONFIG_KASAN` | kernel address sanitizer — بيكتشف memory bugs في attribute callbacks |
| `CONFIG_KCSAN` | data race detector — مهم لو عندك shared attribute state |

```bash
# اتحقق من الـ configs الحالية
grep -E 'CONFIG_DEBUG_LOCK_ALLOC|CONFIG_SYSFS|CONFIG_DEBUG_FS|CONFIG_DEBUG_KOBJECT' \
    /boot/config-$(uname -r)

# compile-time: أضفهم في .config
scripts/config --enable CONFIG_DEBUG_LOCK_ALLOC
scripts/config --enable CONFIG_DEBUG_KOBJECT
scripts/config --enable CONFIG_SLUB_DEBUG
```

---

#### 6. أدوات خاصة بالـ subsystem

**الـ udevadm — تتبع uevent notifications من sysfs:**

```bash
# تابع كل uevents في real-time
udevadm monitor --environment --udev

# تابع uevents لـ subsystem معين
udevadm monitor --subsystem-match=platform

# اعرض كل attributes لـ device
udevadm info --query=all --name=/dev/sda
udevadm info --attribute-walk --name=/dev/sda

# test trigger لـ uevent
udevadm trigger --subsystem-match=net --action=change

# اعرض hierarchy كامل
udevadm info --export-db | head -100
```

**inotifywait — تابع تغييرات في sysfs:**

```bash
# تابع تغييرات في attribute معين
inotifywait -m /sys/class/net/eth0/operstate

# تابع كل تغييرات في directory
inotifywait -r -m /sys/bus/platform/devices/
```

**sysfs-specific inspection tools:**

```bash
# اعرض sysfs tree
find /sys/devices -name "*.txt" -o -name "modalias" | xargs ls -la 2>/dev/null

# اعرض كل kobject hierarchy
ls -la /sys/kernel/

# تحقق من attribute permissions
find /sys/bus -name "bind" -o -name "unbind" | xargs ls -la 2>/dev/null

# اعرض كل symlinks في sysfs (بتكشف الـ kobject relationships)
find /sys -maxdepth 4 -type l -ls 2>/dev/null | head -30
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `sysfs: cannot create duplicate filename '/...'` | حاولت تعمل attribute بنفس الاسم مرتين | اتحقق إن `sysfs_create_file()` مش اتنادى مرتين على نفس الـ attribute |
| `kernfs: can not remove 'X', no directory` | بتحاول تشيل attribute من kobj مش موجود أو اتمسح | تأكد إن الـ kobj لسه valid قبل `sysfs_remove_file()` |
| `sysfs group 'X' not found for kobject 'Y'` | `sysfs_update_group()` على group مش متسجل | اتأكد إن `sysfs_create_group()` اتنادت أول |
| `WARNING: CPU: X PID: Y at fs/kernfs/dir.c:...` | kernfs active reference leak | تأكد من `sysfs_unbreak_active_protection()` بعد كل `sysfs_break_active_protection()` |
| `BUG: lock held when returning to user space` | lock مش اتحرر في show/store callback | افحص كل exit paths في callbacks |
| `kobject_add_internal failed for X` | فشل إضافة kobject لـ sysfs — غالباً اسم مكرر أو parent NULL | افحص return value وتأكد من uniqueness الاسم |
| `attribute_container: device ... not registered` | الـ device مش متسجل في الـ container | تأكد من تسلسل التسجيل الصح |
| `sysfs_attr_init() not called for attr` | dynamic attribute بدون init — lockdep complaining | نادي `sysfs_attr_init()` على كل dynamic attribute |
| `kernfs_node 'X' has active references` | remove وفيه references لسه active | انتظر كل references تتحرر أو استخدم `sysfs_break_active_protection()` |
| `call_usermodehelper failed: -ENOMEM` | الـ uevent helper فشل من قلة الـ memory | افحص memory pressure وـ `/proc/meminfo` |

---

#### 8. أماكن استراتيجية لـ dump_stack() وـ WARN_ON()

```c
/* 1. في sysfs_create_file — اتحقق من state قبل الإضافة */
int sysfs_create_file_ns(struct kobject *kobj,
                          const struct attribute *attr,
                          const void *ns)
{
    WARN_ON(!kobj || !kobj->sd);  /* kobj لازم يكون registered */
    WARN_ON(!attr->name);         /* اسم الـ attribute لازم موجود */
    /* ... */
}

/* 2. في show()/store() callbacks — guard لـ NULL state */
static ssize_t my_attr_show(struct kobject *kobj,
                              struct attribute *attr, char *buf)
{
    struct my_device *dev = container_of(kobj, struct my_device, kobj);

    if (WARN_ON(!dev->initialized)) {
        dump_stack();  /* اعرف مين نادى show() قبل الـ init */
        return -EINVAL;
    }
    /* ... */
}

/* 3. في is_visible() callback — تحقق من consistency */
static umode_t my_attr_visible(struct kobject *kobj,
                                 struct attribute *attr, int n)
{
    struct my_dev *d = container_of(kobj, struct my_dev, kobj);

    /* لو state inconsistent — اصرخ بوضوح */
    WARN_ON(d->state == STATE_A && n > MAX_STATE_A_ATTRS);
    return attr->mode;
}

/* 4. بعد sysfs_remove_group — تأكد إن الـ group اتشال */
sysfs_remove_group(kobj, &my_group);
WARN_ON(sysfs_get_dirent(kobj->sd, my_group.name) != NULL);
```

---

### Hardware Level

---

#### 1. التحقق إن hardware state بيطابق kernel state

الـ sysfs هو الواجهة بين الـ hardware state والـ userspace. عشان تتحقق:

```bash
# قارن ما بين قراءة الـ register مباشرة وقيمة الـ sysfs attribute
# مثال: network interface
cat /sys/class/net/eth0/operstate          # kernel's view
ethtool eth0 | grep "Link detected"       # hardware query

# مثال: CPU frequency
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
rdmsr 0x198 2>/dev/null  # MSR_IA32_PERF_STATUS (x86)

# مثال: GPIO state
cat /sys/class/gpio/gpio<N>/value
# قارن مع register مباشر عبر devmem2

# مثال: thermal zone
cat /sys/class/thermal/thermal_zone0/temp
```

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2: اقرأ physical memory address مباشرة
# اقرأ register بحجم 32-bit
devmem2 0xFE000000 w

# اقرأ 64-bit register
devmem2 0xFE000008 q

# اكتب قيمة في register
devmem2 0xFE000004 w 0x00000001

# /dev/mem: raw access (يحتاج CONFIG_STRICT_DEVMEM=n أو للـ offsets الأولى)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE000000/4)) 2>/dev/null | xxd

# io utility (من package ioport): للـ I/O ports على x86
inb 0x3F8   # اقرأ byte من port
outb 0x01 0x3F8  # اكتب byte في port

# /sys/bus/pci/devices/*/resource*: اقرأ PCI device BARs مباشرة
hexdump -C /sys/bus/pci/devices/0000:00:1f.0/resource0 | head

# ioport: اعرض كل I/O port allocations
cat /proc/ioports

# iomem: اعرض كل memory-mapped regions
cat /proc/iomem | grep -i <device>
```

**مثال كامل: تحقق من state لـ PCI device عبر sysfs وـ devmem2:**

```bash
# 1. اعرف الـ BAR address من sysfs
BAR=$(cat /sys/bus/pci/devices/0000:01:00.0/resource | awk 'NR==1{print $1}')
echo "BAR0 starts at: $BAR"

# 2. اقرأ أول register مباشرة
devmem2 $BAR w

# 3. قارن مع ما يقوله الـ driver في sysfs attribute
cat /sys/bus/pci/devices/0000:01:00.0/vendor
cat /sys/bus/pci/devices/0000:01:00.0/device
```

---

#### 3. Logic Analyzer / Oscilloscope tips

لما بتعمل debug لـ sysfs attribute بيتحكم في hardware:

```
الـ workflow:
1. ضع probe على الـ hardware signal (GPIO, SPI, I2C, clock line).
2. في نفس الوقت، اعمل trigger في الـ kernel:
   - استخدم GPIO مخصص للـ debug → toggle في show()/store() callback.
   - أو استخدم ftrace timestamp من trace output.

3. قارن timestamp الـ kernel event مع edge الـ signal على الـ scope.
```

**GPIO debug trigger من kernel code:**

```c
#include <linux/gpio/consumer.h>

/* في store() callback: toggle debug GPIO */
static ssize_t my_store(struct kobject *kobj, struct attribute *attr,
                         const char *buf, size_t count)
{
    /* Toggle a spare GPIO as scope trigger */
    gpiod_set_value(debug_gpio, 1);
    /* ... actual work ... */
    gpiod_set_value(debug_gpio, 0);
    return count;
}
```

**نقاط المراقبة الأساسية:**

| ما تراقبه | أداة | ما تبحث عنه |
|---|---|---|
| I2C bus عند كتابة attribute | Logic Analyzer | latency بين sysfs write وـ I2C transaction |
| GPIO toggle عند تغيير attribute | Oscilloscope | glitches أو unexpected transitions |
| Power rail عند تغيير power attribute | Oscilloscope | voltage droop أو overshoot |
| SPI clock عند firmware update bin_attr | Logic Analyzer | clock stretching أو frame errors |

---

#### 4. Hardware issues الشائعة وـ kernel log patterns

| المشكلة | ما يظهر في الـ kernel log | كيف تتحقق |
|---|---|---|
| Hardware register لا يستجيب | `timeout waiting for device`, ثم `EIO` من store() | اقرأ الـ register مباشرة بـ devmem2 |
| Device غير موجود فعلياً لكن DT بيقول موجود | `sysfs: cannot create duplicate` أو attribute موجود لكن فارغ | افحص physical connections |
| Race بين hardware interrupt وـ sysfs read | `WARNING: suspicious RCU usage` في show() | ضع sysfs attribute access تحت proper locking |
| Device removed وـ sysfs attribute لسه موجود | `kernfs: can not remove` عند rmmod | تأكد من cleanup order في driver remove |
| Clock أو power لـ device مش enabled | `driver: probe failed: -ENODEV` ثم sysfs attributes مش بتعمل | افحص clk/regulator state عبر debugfs |

---

#### 5. تحقق من الـ Device Tree مع الـ Hardware

```bash
# اعرض الـ DT compiled blob
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# اعرض property معينة
cat /sys/firmware/devicetree/base/soc/<device>/compatible | xxd
cat /sys/firmware/devicetree/base/soc/<device>/reg | xxd

# قارن الـ DT reg مع الـ actual resource
cat /sys/firmware/devicetree/base/soc/<device>/reg | \
    od -A x -t x4 -v

# تحقق من الـ device بعد probe
ls /sys/bus/platform/devices/<device>/
cat /sys/bus/platform/devices/<device>/driver  # هل اتبند لـ driver؟

# اتحقق من resources الـ device
cat /proc/iomem | grep <device>

# overlay DT debugging
ls /sys/kernel/config/device-tree/overlays/

# اعرض كل DT nodes اللي اتعملها probe ناجح
ls /sys/bus/platform/drivers/<driver>/
```

**سيناريو شائع: DT بيقول reg=0x1000 لكن hardware على 0x2000:**

```bash
# 1. اقرأ الـ reg من DT
cat /sys/firmware/devicetree/base/soc/mydev@1000/reg | od -An -tx4
# Output: 00001000 00000100  ← base=0x1000, size=0x100

# 2. اتحقق من resource الـ kernel allocate
cat /proc/iomem | grep mydev
# لو مش موجود = resource conflict أو مش اتعملوا map

# 3. جرب اقرأ مباشرة بـ devmem2
devmem2 0x1000 w  # لو error = wrong address
devmem2 0x2000 w  # جرب العنوان الحقيقي

# 4. لو طلع البيانات صح عند 0x2000، عدل الـ DT
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**فحص كامل للـ sysfs attribute:**

```bash
#!/bin/bash
# sysfs-attr-debug.sh: debug a specific sysfs attribute completely
ATTR_PATH="${1:-/sys/class/net/eth0/operstate}"
ATTR_DIR=$(dirname "$ATTR_PATH")
ATTR_NAME=$(basename "$ATTR_PATH")

echo "=== Attribute Info ==="
stat "$ATTR_PATH"

echo "=== Current Value ==="
cat "$ATTR_PATH"

echo "=== kobject directory ==="
ls -la "$ATTR_DIR/"

echo "=== uevent info ==="
cat "$ATTR_DIR/uevent" 2>/dev/null || echo "no uevent"

echo "=== Symlinks ==="
find "$ATTR_DIR" -maxdepth 1 -type l -ls

echo "=== udevadm info ==="
udevadm info --path="${ATTR_DIR#/sys}" 2>/dev/null | head -30
```

**تفعيل كامل لـ debugging الـ sysfs:**

```bash
#!/bin/bash
# enable-sysfs-debug.sh

DEBUGFS=/sys/kernel/debug
DYNDBG=$DEBUGFS/dynamic_debug/control
TRACE=$DEBUGFS/tracing

# 1. Dynamic debug للـ sysfs/kernfs
echo 'file fs/sysfs/*.c +p' > $DYNDBG
echo 'file fs/kernfs/*.c +p' > $DYNDBG
echo 'file lib/kobject.c +p' > $DYNDBG

# 2. ftrace للـ sysfs functions
echo 0 > $TRACE/tracing_on
echo function > $TRACE/current_tracer
echo 'sysfs_* kernfs_* kobject_*' > $TRACE/set_ftrace_filter
echo 1 > $TRACE/tracing_on

echo "Debug enabled. Run: cat $TRACE/trace_pipe"
```

**تعطيل الـ debugging بعد الانتهاء:**

```bash
#!/bin/bash
# disable-sysfs-debug.sh

DEBUGFS=/sys/kernel/debug
DYNDBG=$DEBUGFS/dynamic_debug/control
TRACE=$DEBUGFS/tracing

echo 0 > $TRACE/tracing_on
echo nop > $TRACE/current_tracer
echo > $TRACE/set_ftrace_filter

echo 'file fs/sysfs/*.c -p' > $DYNDBG
echo 'file fs/kernfs/*.c -p' > $DYNDBG
echo 'file lib/kobject.c -p' > $DYNDBG

echo "Debug disabled."
```

---

#### مثال output وإزاي تفسره

**output من `udevadm info`:**

```
P: /devices/platform/my_driver/my_device
N: my_device
L: 0
E: DEVPATH=/devices/platform/my_driver/my_device
E: SUBSYSTEM=platform
E: DRIVER=my_driver
E: ATTR{my_attribute}=42
```

التفسير:
- **P** = sysfs path للـ device (بيبدأ من `/sys`)
- **E: ATTR{name}=value** = قيمة الـ sysfs attribute وقت الاستعلام
- **SUBSYSTEM=platform** = الـ kobject مسجل تحت الـ platform bus

**output من `cat /sys/kernel/debug/tracing/trace`:**

```
     bash-1234  [001] ....  1234.567890: sysfs_create_file <-kobject_add_internal
     bash-1234  [001] ....  1234.567891: kernfs_new_node <-sysfs_add_file_mode_ns
     bash-1234  [001] ....  1234.567892: kernfs_activate <-sysfs_create_file_ns
```

التفسير:
- **process-pid** = العملية اللي عملت الـ syscall
- **[CPU]** = الـ CPU core
- **timestamp** = وقت الحدث بالـ nanoseconds
- **function <- caller** = الـ call chain — تقرأها من تحت لفوق للـ call stack

**output من lockdep عند نسيان `sysfs_attr_init()`:**

```
WARNING: CPU: 0 PID: 123 at fs/sysfs/file.c:...
kobject: 'my_obj' sysfs_create_file: missing sysfs_attr_init()
Call Trace:
  sysfs_create_file_ns+0x...
  kobject_add_internal+0x...
  kobject_add+0x...
  my_driver_probe+0x...
```

الإصلاح:

```c
/* أضف sysfs_attr_init() لكل attribute بتعملها dynamically */
struct attribute *my_attr = kzalloc(sizeof(*my_attr), GFP_KERNEL);
my_attr->name = "my_value";
my_attr->mode = 0644;
sysfs_attr_init(my_attr);  /* لازم قبل sysfs_create_file() */
sysfs_create_file(kobj, my_attr);
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — `sysfs_notify` مش بيشغّل `poll()`

#### العنوان
**الـ userspace daemon مش بيستحوذ على تغييرات درجة الحرارة في gateway صناعي**

#### السياق
شركة بتبني industrial gateway بيستخدم RK3562 لجمع بيانات من حساسات Modbus. الـ daemon بتاعهم بيعمل `poll()` على `/sys/class/thermal/thermal_zone0/temp` عشان يتحرك لما الحرارة تتغير. المنتج اشتغل على development board، بس في production board مخصوص، الـ daemon بيستهلك 100% CPU في loop مش بيتوقف.

#### المشكلة
الـ daemon بيعمل:
```c
struct pollfd fds = { .fd = fd, .events = POLLPRI | POLLERR };
poll(&fds, 1, -1);  /* should block until value changes */
```
بس `poll()` بترجع فورًا في كل مرة حتى لو الحرارة ما اتغيرتش.

#### التحليل
الـ `sysfs_notify()` في `sysfs.h` بيقول:
```c
void sysfs_notify(struct kobject *kobj, const char *dir, const char *attr);
```
الـ kernel thermal driver لازم يستدعي `sysfs_notify()` بعد كل تغيير في قيمة `temp` عشان الـ `kernfs_notify(kn)` يتنادى، اللي بدوره يوقظ كل thread بيعمل `poll()` على الـ file.

في الـ production board، الـ thermal driver اتعدّل بشكل custom وحذفوا استدعاء `sysfs_notify()` عن طريق الخطأ:

```c
/* BROKEN: missing sysfs_notify after updating temp */
static void rk3562_thermal_update(struct rk3562_thermal *data)
{
    data->temperature = read_adc();
    /* sysfs_notify was removed here by mistake */
}
```

الصح كان لازم يكون:
```c
static void rk3562_thermal_update(struct rk3562_thermal *data)
{
    data->temperature = read_adc();
    /* wake up any poll() waiters on this attribute */
    sysfs_notify(&data->kobj, NULL, "temp");
}
```

بدون `sysfs_notify()`، الـ `kernfs_node` مش بيتعلم إن في تغيير حصل، فـ`poll()` بترجع فورًا (أو بتتجاهل الـ event).

#### الحل
```bash
# تحقق إن sysfs_notify بيتستدعى فعلاً
grep -r "sysfs_notify" drivers/thermal/
# لو مش موجود في الـ custom driver، يبقى ده سبب المشكلة
```

```c
/* Fix in drivers/thermal/rk3562_thermal.c */
static void rk3562_thermal_update(struct rk3562_thermal *data)
{
    data->temperature = read_adc();
    sysfs_notify(&data->kobj, NULL, "temp"); /* unblock poll() waiters */
}
```

ومن ناحية الـ userspace، لازم الـ daemon يعمل `lseek` إلى البداية قبل `read()` بعد كل `poll()`:
```c
poll(&fds, 1, -1);
lseek(fd, 0, SEEK_SET);   /* rewind before re-reading */
read(fd, buf, sizeof(buf));
```

#### الدرس المستفاد
`sysfs_notify()` مش optional — هي اللي بتربط kernel-side changes بـ userspace `poll()`/`select()`. أي driver بيعرض قيمة متغيرة عبر sysfs لازم يستدعيها بعد كل تحديث، وإلا الـ userspace هيضطر يعمل busy-poll.

---

### السيناريو 2: Android TV Box على Allwinner H616 — `VERIFY_OCTAL_PERMISSIONS` بيمنع compile

#### العنوان
**Build error في driver الـ HDMI بسبب permissions خاطئة في `__ATTR`**

#### السياق
فريق بيشتغل على Android TV box بيستخدم Allwinner H616. بيضيفوا sysfs attribute جديدة لـ HDMI driver عشان يقرأ الـ EDID من userspace. الـ developer كتب الكود بسرعة وحط permissions `0666`.

#### المشكلة
```c
/* Causes build error */
static struct device_attribute hdmi_edid_attr =
    __ATTR(edid, 0666, edid_show, edid_store);
```

الـ build بيفشل:
```
error: call to '__compiletime_assert_...' declared with attribute error:
BUILD_BUG_ON failed: (((0666) >> 3) & 2) > (((0666) >> 6) & 2)
```

#### التحليل
**الـ `VERIFY_OCTAL_PERMISSIONS`** macro في `sysfs.h` بيعمل compile-time validation:

```c
#define VERIFY_OCTAL_PERMISSIONS(perms)
    (BUILD_BUG_ON_ZERO((perms) < 0) +
     BUILD_BUG_ON_ZERO((perms) > 0777) +
     /* USER_READABLE >= GROUP_READABLE >= OTHER_READABLE */
     BUILD_BUG_ON_ZERO((((perms) >> 6) & 4) < (((perms) >> 3) & 4)) +
     BUILD_BUG_ON_ZERO((((perms) >> 3) & 4) < ((perms) & 4)) +
     /* USER_WRITABLE >= GROUP_WRITABLE */
     BUILD_BUG_ON_ZERO((((perms) >> 6) & 2) < (((perms) >> 3) & 2)) +
     /* OTHER_WRITABLE?  Generally considered a bad idea. */
     BUILD_BUG_ON_ZERO((perms) & 2) +
     (perms))
```

الـ `0666` بتعمل fail على الشرط الأخير: `BUILD_BUG_ON_ZERO((perms) & 2)` — الـ "other writable" bit ممنوع في sysfs لأسباب أمنية. كمان الشرط `USER_WRITABLE >= GROUP_WRITABLE` بيفشل لو الـ group write bit أعلى من user.

في `0666`، الـ bits هي `rw-rw-rw-` — `other` لها write permission، وده محظور.

#### الحل
```c
/* Correct: root can write, others only read */
static struct device_attribute hdmi_edid_attr =
    __ATTR(edid, 0644, edid_show, edid_store);

/* Or if only root should read/write EDID */
static struct device_attribute hdmi_edid_attr =
    __ATTR(edid, 0600, edid_show, edid_store);

/* Read-only for all: use __ATTR_RO macro */
static DEVICE_ATTR_RO(edid);
```

قواعد `VERIFY_OCTAL_PERMISSIONS`:
| القاعدة | السبب |
|---|---|
| no `other_write` (no `& 2`) | منع تعديل kernel state من أي user |
| `user_write >= group_write` | consistency مع Unix permissions model |
| `group_read <= user_read` | لا تعطي group أكثر من user |

#### الدرس المستفاد
الـ `VERIFY_OCTAL_PERMISSIONS` موجود عشان يمنع security holes قبل ما الكود يوصل للـ runtime. الـ `0644` هو الـ default الآمن لـ sysfs attributes القابلة للقراءة والكتابة، والـ `0444` للقراءة فقط. لا تكتب `0666` أبدًا في sysfs.

---

### السيناريو 3: IoT Sensor على STM32MP1 — `attribute_group` مع `is_visible` بيخبي attributes

#### العنوان
**بعض الـ sysfs attributes بتختفي بشكل dynamic على STM32MP1 IoT sensor**

#### السياق
شركة بتبني IoT sensor node بيستخدم STM32MP1 بيقيس الضغط والحرارة والرطوبة. بيستخدموا driver واحد بثلاث attribute groups — واحدة لكل حساس. على بعض variants من الـ board، الحساس الخاص بالرطوبة مش موجود hardware. الـ engineer حاول يستخدم `is_visible` عشان يخبي الـ attributes دي بشكل dynamic بس المشكلة إن الـ directory نفسه بيفضل ظاهر فاضي.

#### المشكلة
```c
static umode_t humidity_attr_visible(struct kobject *kobj,
                                     struct attribute *attr, int n)
{
    if (!board_has_humidity_sensor())
        return 0; /* hide attribute */
    return attr->mode;
}

static struct attribute_group humidity_group = {
    .name       = "humidity",
    .is_visible = humidity_attr_visible,
    .attrs      = humidity_attrs,
};
```

النتيجة: directory `/sys/bus/iio/devices/iio:device0/humidity/` بيظهر فاضي بدل ما يختفي كليًا.

#### التحليل
الـ `SYSFS_GROUP_INVISIBLE` flag في `sysfs.h`:
```c
#define SYSFS_GROUP_INVISIBLE   020000
```

وشرحه في comment الـ `attribute_group`:
> Additionally when a group is named, `@is_visible` and `@is_bin_visible` may return `SYSFS_GROUP_INVISIBLE` to control visibility of **the directory itself**.

يعني عشان تخبي الـ directory نفسه (مش بس الـ attributes جوّاه)، الـ `is_visible` لازم ترجع `SYSFS_GROUP_INVISIBLE` لما `n == 0`.

الطريقة الصح هي استخدام الـ macro الجاهز `DEFINE_SYSFS_GROUP_VISIBLE`:

```c
/* Step 1: implement the two callbacks */
static bool humidity_group_visible(struct kobject *kobj)
{
    return board_has_humidity_sensor();
}

static umode_t humidity_attr_visible(struct kobject *kobj,
                                     struct attribute *attr, int n)
{
    return attr->mode; /* all attrs visible if group is visible */
}

/* Step 2: generate the combined is_visible function */
DEFINE_SYSFS_GROUP_VISIBLE(humidity);

/* Step 3: use SYSFS_GROUP_VISIBLE() macro as the callback */
static struct attribute_group humidity_group = {
    .name       = "humidity",
    .is_visible = SYSFS_GROUP_VISIBLE(humidity), /* expands to sysfs_group_visible_humidity */
    .attrs      = humidity_attrs,
};
```

الـ macro `DEFINE_SYSFS_GROUP_VISIBLE` بيولّد function بتعمل:
```c
static inline umode_t sysfs_group_visible_humidity(
    struct kobject *kobj, struct attribute *attr, int n)
{
    if (n == 0 && !humidity_group_visible(kobj))
        return SYSFS_GROUP_INVISIBLE; /* hide directory on first call */
    return humidity_attr_visible(kobj, attr, n);
}
```

#### الحل
لو كل الـ attributes بنفس الـ visibility (مش محتاج per-attribute logic):
```c
static bool humidity_group_visible(struct kobject *kobj)
{
    return board_has_humidity_sensor();
}

/* Simpler macro: no need for separate attr_visible callback */
DEFINE_SIMPLE_SYSFS_GROUP_VISIBLE(humidity);

static struct attribute_group humidity_group = {
    .name       = "humidity",
    .is_visible = SYSFS_GROUP_VISIBLE(humidity),
    .attrs      = humidity_attrs,
};
```

تحقق:
```bash
# Board with sensor:
ls /sys/bus/iio/devices/iio:device0/humidity/
# temperature  pressure  (shows attributes)

# Board without sensor:
ls /sys/bus/iio/devices/iio:device0/humidity/
# ls: cannot access '...': No such file or directory
```

#### الدرس المستفاد
الـ `is_visible` callback بمفردها بتخبي الـ attributes بس مش الـ directory. عشان تتحكم في ظهور الـ directory نفسه، لازم ترجع `SYSFS_GROUP_INVISIBLE` لما `n == 0`، وأسهل طريقة للعمل ده هي استخدام `DEFINE_SIMPLE_SYSFS_GROUP_VISIBLE` أو `DEFINE_SYSFS_GROUP_VISIBLE`.

---

### السيناريو 4: Automotive ECU على i.MX8 — `bin_attribute` و `mmap` لنقل firmware

#### العنوان
**نقل firmware update بطيء جداً عبر sysfs write على i.MX8 ECU**

#### السياق
automotive ECU بيستخدم i.MX8 بيحتاج يستقبل firmware updates لمعالج DSP خارجي عبر SPI. الـ engineer الأول استخدم text attribute عادية، البيانات بتتنقل بـ 4KB في كل write (حجم الـ sysfs page buffer)، وتحديث الـ 512KB بياخد وقت طويل.

#### المشكلة
```c
/* Slow approach: text attribute limited to PAGE_SIZE per write */
static struct device_attribute dsp_fw_attr =
    __ATTR(firmware, 0200, NULL, dsp_fw_store);

/* userspace must write in 4KB chunks */
for (offset = 0; offset < fw_size; offset += 4096) {
    write(fd, fw_buf + offset, 4096); /* slow, many syscalls */
}
```

#### التحليل
**الـ `struct bin_attribute`** في `sysfs.h` هو الحل:

```c
struct bin_attribute {
    struct attribute    attr;
    size_t              size;           /* total size of binary data */
    void               *private;
    struct address_space *(*f_mapping)(void);
    ssize_t (*read)(struct file *, struct kobject *, const struct bin_attribute *,
                    char *, loff_t, size_t);
    ssize_t (*write)(struct file *, struct kobject *, const struct bin_attribute *,
                     char *, loff_t, size_t);
    loff_t  (*llseek)(...);
    int     (*mmap)(struct file *, struct kobject *, const struct bin_attribute *,
                    struct vm_area_struct *vma);  /* zero-copy option */
};
```

الـ `bin_attribute` بيدعم:
1. **writes بأي حجم** — مش محدود بـ PAGE_SIZE
2. **`llseek`** — للـ random access
3. **`mmap`** — zero-copy للـ large transfers

الحل الأمثل للـ ECU هو استخدام `mmap` لنقل الـ firmware مرة واحدة:

```c
/* DSP firmware binary attribute with mmap support */
static int dsp_fw_mmap(struct file *filp, struct kobject *kobj,
                       const struct bin_attribute *attr,
                       struct vm_area_struct *vma)
{
    struct imx8_dsp *dsp = dev_get_drvdata(kobj_to_dev(kobj));
    /* map DSP SRAM directly to userspace */
    return remap_pfn_range(vma, vma->vm_start,
                           dsp->sram_phys >> PAGE_SHIFT,
                           dsp->sram_size, vma->vm_page_prot);
}

static ssize_t dsp_fw_write(struct file *filp, struct kobject *kobj,
                            const struct bin_attribute *attr,
                            char *buf, loff_t off, size_t count)
{
    struct imx8_dsp *dsp = dev_get_drvdata(kobj_to_dev(kobj));
    memcpy_toio(dsp->sram + off, buf, count);  /* copy to DSP SRAM */
    return count;
}

/* Declare the binary attribute */
static BIN_ATTR_WO(dsp_firmware, DSP_SRAM_SIZE);
/* expands to: struct bin_attribute bin_attr_dsp_firmware =
               __BIN_ATTR(dsp_firmware, 0200, NULL, dsp_firmware_write, DSP_SRAM_SIZE) */
```

ومن الـ userspace:
```c
/* Zero-copy firmware load via mmap */
int fd = open("/sys/bus/platform/devices/imx8-dsp.0/dsp_firmware", O_RDWR);
void *sram = mmap(NULL, DSP_SRAM_SIZE, PROT_WRITE, MAP_SHARED, fd, 0);
memcpy(sram, fw_data, fw_size);  /* direct copy, no syscall overhead */
munmap(sram, DSP_SRAM_SIZE);
```

#### الحل
استبدال text attribute بـ `bin_attribute` مع `mmap`:

```c
static struct bin_attribute dsp_fw_binattr = {
    .attr  = { .name = "dsp_firmware", .mode = 0600 },
    .size  = DSP_SRAM_SIZE,
    .write = dsp_fw_write,
    .mmap  = dsp_fw_mmap,
};

/* In probe */
ret = sysfs_create_bin_file(&dev->kobj, &dsp_fw_binattr);
```

مقارنة الأداء:
| الطريقة | عدد syscalls لـ 512KB | الزمن التقريبي |
|---|---|---|
| text attribute (4KB/write) | 128 writes | ~50ms |
| bin_attribute (write) | 1 write | ~5ms |
| bin_attribute (mmap) | 1 mmap + 1 memcpy | ~1ms |

#### الدرس المستفاد
**الـ `bin_attribute`** هو الاختيار الصح لأي بيانات binary أو large data transfers. الـ `mmap` callback بيدي zero-copy performance لازم يكون الاختيار الأول في الأنظمة embedded بـ real-time requirements.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `sysfs_attr_init` و lockdep warning

#### العنوان
**Lockdep warning عند إضافة dynamically-allocated attribute في AM62x bring-up**

#### السياق
مهندس بيعمل bring-up لـ custom industrial board بيستخدم TI AM62x. الـ driver بتاعه بينشئ sysfs attributes بشكل dynamic (مش static) بناءً على عدد الـ channels اللي بيتقراه من Device Tree. البيستخدم kernel مع `CONFIG_DEBUG_LOCK_ALLOC=y`، وفي الـ dmesg بيظهر lockdep warning عند كل `sysfs_create_file()`.

#### المشكلة
```
[ 12.345678] WARNING: suspicious RCU usage
[ 12.345679] ... lockdep: sysfs attribute added without sysfs_attr_init()
```

الكود:
```c
static int am62x_adc_probe(struct platform_device *pdev)
{
    int num_channels = of_property_count_u32_elems(np, "channels");

    attrs = kcalloc(num_channels, sizeof(struct attribute *), GFP_KERNEL);
    for (i = 0; i < num_channels; i++) {
        attrs[i] = kzalloc(sizeof(struct attribute), GFP_KERNEL);
        attrs[i]->name = kasprintf(GFP_KERNEL, "channel%d", i);
        attrs[i]->mode = 0444;
        /* BUG: missing sysfs_attr_init! */
    }

    return sysfs_create_files(&pdev->dev.kobj,
                              (const struct attribute **)attrs);
}
```

#### التحليل
الـ `sysfs_attr_init` macro في `sysfs.h`:

```c
#ifdef CONFIG_DEBUG_LOCK_ALLOC
#define sysfs_attr_init(attr)           \
do {                                    \
    static struct lock_class_key __key; \
                                        \
    (attr)->key = &__key;               \
} while (0)
#else
#define sysfs_attr_init(attr) do {} while (0)
#endif
```

لما `CONFIG_DEBUG_LOCK_ALLOC` يكون enabled، كل `struct attribute` لازم يكون ليها `lock_class_key` مرتبط بيها. الـ static attributes الـ kernel بيتعامل معاها تلقائيًا، بس الـ **dynamically allocated** ones محتاجة `sysfs_attr_init()` صريح.

المشكلة إن الـ static `lock_class_key` جوا الـ macro بتاخد address ثابت لكل **call site**، يعني لو استدعيت الـ macro في loop، كل الـ attributes هتشارك نفس الـ key — اللي هو سلوك صح ومقصود (هم كلهم من نفس النوع).

```c
/* Correct: call sysfs_attr_init for each dynamic attribute */
for (i = 0; i < num_channels; i++) {
    attrs[i] = kzalloc(sizeof(struct attribute), GFP_KERNEL);
    attrs[i]->name = kasprintf(GFP_KERNEL, "channel%d", i);
    attrs[i]->mode = 0444;
    sysfs_attr_init(attrs[i]);  /* initialize lock_class_key */
}
```

بالمثل للـ `bin_attribute`:
```c
static BIN_ATTR_RO(calibration_data, 256);
/* For dynamic bin_attribute: */
sysfs_bin_attr_init(&my_bin_attr);  /* calls sysfs_attr_init(&attr->attr) */
```

#### الحل

```c
static int am62x_adc_probe(struct platform_device *pdev)
{
    int num_channels = of_property_count_u32_elems(np, "channels");

    attrs = kcalloc(num_channels + 1, sizeof(struct attribute *), GFP_KERNEL);
    for (i = 0; i < num_channels; i++) {
        struct attribute *attr = kzalloc(sizeof(*attr), GFP_KERNEL);
        if (!attr)
            goto err_free;

        attr->name = kasprintf(GFP_KERNEL, "channel%d", i);
        attr->mode = 0444;
        sysfs_attr_init(attr);   /* register with lockdep */
        attrs[i] = attr;
    }
    attrs[num_channels] = NULL;  /* NULL-terminate the list */

    return sysfs_create_files(&pdev->dev.kobj,
                              (const struct attribute *const *)attrs);
}
```

تأكد من cleanup عند `remove`:
```c
static int am62x_adc_remove(struct platform_device *pdev)
{
    sysfs_remove_files(&pdev->dev.kobj,
                       (const struct attribute *const *)attrs);
    for (i = 0; attrs[i]; i++) {
        kfree(attrs[i]->name);
        kfree(attrs[i]);
    }
    kfree(attrs);
    return 0;
}
```

ومن الـ dmesg بعد الإصلاح:
```bash
dmesg | grep -i "lockdep\|sysfs_attr"
# no output = no warnings = correct initialization
```

#### الدرس المستفاد
`sysfs_attr_init()` مطلوبة لكل dynamically allocated `struct attribute`. الـ static attributes (اللي بتتعرف بـ `__ATTR`, `DEVICE_ATTR`, إلخ) بتتعامل معاها تلقائيًا. نسيانها في kernel مع `CONFIG_DEBUG_LOCK_ALLOC=y` بيدي warning مزعج في production logs وممكن يدل على مشكلة حقيقية في thread-safety.
## Phase 7: مصادر ومراجع

### مصادر البحث المستخدمة

تم البحث في المصادر التالية قبل كتابة هذه المرحلة:
- `site:lwn.net sysfs`
- `site:kernelnewbies.org sysfs`
- `site:elinux.org sysfs`
- `linux kernel sysfs mailing list discussion kernfs attribute_group`

---

### توثيق الـ Kernel الرسمي

أهم مرجع مباشر هو التوثيق الرسمي المذكور في أول سطور `sysfs.h` نفسها:

```
/* Please see Documentation/filesystems/sysfs.rst for more information. */
```

| الملف | المحتوى |
|---|---|
| `Documentation/filesystems/sysfs.rst` | الوثيقة الرئيسية لـ sysfs — التصميم، القواعد، الأمثلة |
| `Documentation/core-api/kobject.rst` | كل شيء عن kobject و kset و ktype |
| `Documentation/ABI/` | توثيق كل attribute موجود في `/sys` مع وصف وصلاحيات |
| `Documentation/filesystems/kernfs.rst` | توثيق kernfs اللي بيشتغل تحت sysfs مباشرةً |

الرابط الرسمي للتوثيق على الويب:
- [sysfs — The Linux Kernel documentation](https://docs.kernel.org/filesystems/sysfs.html)
- [kobjects and ksets — The Linux Kernel documentation](https://docs.kernel.org/core-api/kobject.html)

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع التاريخي الأول لفهم تطور sysfs في الـ kernel.

| المقال | الأهمية |
|---|---|
| [kobjects and sysfs (2003)](https://lwn.net/Articles/54651/) | مقدمة قديمة بتشرح إزاي kobject بيربط sysfs بالـ kernel |
| [Avoiding sysfs surprises (2004)](https://lwn.net/Articles/36850/) | قواعد التصميم الصحيح لـ sysfs attributes وإزاي تتجنب المشاكل |
| [A critical look at sysfs attribute values (2010)](https://lwn.net/Articles/378884/) | نقاش عميق حول قاعدة "one file, one value" وليه هي مهمة |
| [sysfs is dumb (2010)](https://lwn.net/Articles/357409/) | نقد لـ sysfs ومقارنته بـ sysctl و ioctl — مهم لفهم القيود |
| [Sysfs and namespaces (2010)](https://lwn.net/Articles/295587/) | إزاي sysfs بيتعامل مع network namespaces |
| [sysfs: separate out kernfs, part #1 (2013)](https://lwn.net/Articles/571590/) | التحول التاريخي من sysfs مباشر إلى kernfs كـ layer تحت sysfs |
| [Enforcing mount options for sysfs and proc (2015)](https://lwn.net/Articles/647757/) | أمان الـ sysfs في namespaces والـ mount options |

> الـ [مقال Patrick Mochel في OLS 2005](https://www.kernel.org/pub/linux/kernel/people/mochel/doc/papers/ols-2005/mochel.pdf) — ورقة بحثية من مؤلف sysfs نفسه تشرح التصميم الأصلي.

---

### Kernel Commits المهمة

#### نشأة sysfs

**الـ sysfs** اتكتب من الصفر بواسطة **Patrick Mochel** خلال دورة تطوير 2.5 (2002-2003) كبديل لتلويث `procfs` بمعلومات الـ devices.

الـ commits الأساسية يمكن تتبعها عبر:

```bash
# تاريخ sysfs من بداياته
git log --oneline -- fs/sysfs/

# التحول إلى kernfs في kernel 3.14
git log --oneline -- fs/kernfs/
```

#### فصل kernfs (2014 — Linux 3.14)

أهم تغيير بنيوي في تاريخ sysfs هو فصل الـ core منه إلى **kernfs** مستقل:

```
commit 5d0e7705: sysfs: separate out kernfs
```

**الـ kernfs** بيوفر الـ infrastructure اللي بتبني عليه sysfs و debugfs وغيرهم.

#### إضافة `sysfs_emit` (Linux 5.10)

```bash
git log --oneline --grep="sysfs_emit"
```

**الـ `sysfs_emit()`** اتضافت كبديل آمن لـ `sprintf()` في الـ `show()` callbacks — بتمنع overflow تلقائيًا.

---

### نقاشات Mailing List

| النقاش | الرابط |
|---|---|
| Driver core and sysfs changes for attribute groups | [narkive archive](https://fa.linux.kernel.narkive.com/tE5OK0qG/driver-core-and-sysfs-changes-for-attribute-groups) |
| PATCH v3: sysfs attribute groups binary support | [narkive archive](https://linux.kernel.narkive.com/OQhGUYxO/patch-v3-01-10-driver-core-and-sysfs-changes-for-attribute-groups) |

للبحث في الـ LKML مباشرة: [lkml.org](https://lkml.org/) — ابحث عن `sysfs_emit`, `attribute_group`, `kernfs`.

---

### كتب موصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلف**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 14 — *The Linux Device Model*
- **التنزيل المجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **ملاحظة**: الكتاب قديم (kernel 2.6.10) لكن الـ concepts أساسية لفهم kobject و sysfs.

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل المهم**: Chapter 17 — *Devices and Modules*
- يشرح device model بشكل واضح ومختصر مع علاقة kobject بـ sysfs.

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل المهم**: Chapter 8 — *Device Driver Basics*
- مناسب جداً لفهم sysfs في سياق embedded systems والـ GPIO والـ sensors.

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل المهم**: Chapter 6 — *Device Drivers*
- أعمق تحليل بنيوي لـ kobject/kset/ktype وعلاقتهم بـ sysfs.

---

### مصادر kernelnewbies.org

- [Linux_2_6_11 — kernelnewbies.org](https://kernelnewbies.org/Linux_2_6_11): أول ظهور منظم لـ `/sys/kernel`
- [Dynamic Sysfs Attribute Files — kernelnewbies mailing list](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2013-April/008097.html): نقاش عملي حول إضافة sysfs files ديناميكياً من module
- [Using binary attributes for sysfs — kernelnewbies mailing list](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2019-March/019891.html): متى تستخدم `bin_attribute` وقيوده

---

### مصادر elinux.org

**الـ elinux.org** مفيد للجانب العملي في embedded systems:

- [EBC Flashing an LED — elinux.org](https://elinux.org/EBC_Flashing_an_LED): مثال عملي على استخدام sysfs للـ GPIO
- [CI20 GPIO LED Blink Tutorial — elinux.org](https://elinux.org/CI20_GPIO_LED_Blink_Tutorial): `/sys/class/gpio/` في العمل الفعلي
- [PWM Subsystem with sysfs — elinux.org](https://elinux.org/images/e/ea/OSS_ELC_2021_Exploring_PWM_Subsystem.pptx): presentation من ELC 2021 عن PWM عبر sysfs

---

### مقالات وموارد إضافية

| المصدر | الرابط |
|---|---|
| التوثيق القديم (sysfs.txt) | [kernel.org/doc/Documentation/filesystems/sysfs.txt](https://www.kernel.org/doc/Documentation/filesystems/sysfs.txt) |
| ورقة Patrick Mochel (OLS 2005) | [mochel.pdf](https://www.kernel.org/pub/linux/kernel/people/mochel/doc/papers/ols-2005/mochel.pdf) |
| sysfs — Wikipedia | [en.wikipedia.org/wiki/Sysfs](https://en.wikipedia.org/wiki/Sysfs) |
| Sysfs tutorial — embetronicx | [embetronicx.com](https://embetronicx.com/tutorials/linux/device-drivers/sysfs-in-linux-kernel/) |
| Complete guide to sysfs — Medium | [medium.com](https://medium.com/@emanuele.santini.88/sysfs-in-linux-kernel-a-complete-guide-part-1-c3629470fc84) |
| sysfs & kobjects — win.tue.nl | [lk-13.html](https://www.win.tue.nl/~aeb/linux/lk/lk-13.html) |

---

### مصطلحات البحث

لو عايز تدور على معلومات أكثر، استخدم الـ search terms دي:

```
# بحث عام
linux kernel sysfs attribute kobject
linux sysfs bin_attribute binary attribute
linux sysfs_emit PAGE_SIZE
linux attribute_group visibility is_visible

# بحث متخصص
linux kernfs kobject sysfs separation
linux sysfs lockdep attribute init
linux sysfs namespace network containers
linux sysfs notify poll inotify userspace

# بحث في الـ kernel source
git log --oneline -- include/linux/sysfs.h
git log --oneline -- fs/sysfs/
git log --oneline -- fs/kernfs/
git log --grep="sysfs_emit"
git log --grep="attribute_group"
```

---

### خريطة الملفات في الـ Kernel Source

```
include/linux/sysfs.h          ← الـ public API (الملف الحالي)
include/linux/kobject.h        ← struct kobject وعلاقتها بـ sysfs
include/linux/kernfs.h         ← الـ kernfs_node وكل الـ internal types

fs/sysfs/                      ← implementation كامل
├── file.c                     ← show/store و bin_attr operations
├── dir.c                      ← إنشاء/حذف directories
├── symlink.c                  ← symlinks في /sys
├── group.c                    ← attribute_group logic
├── mount.c                    ← mount sysfs
└── sysfs.h                    ← internal header

fs/kernfs/                     ← الـ layer التحتاني
├── file.c
├── dir.c
├── mount.c
└── kernfs-internal.h

lib/kobject.c                  ← kobject_add → sysfs_create_dir_ns
```
## Phase 8: Writing simple module

### الفكرة

هنعمل module بيستخدم **kprobe** عشان يعمل hook على `sysfs_create_file_ns` — الـ function الأساسية اللي بتتنادى كل ما أي driver أو subsystem يضيف attribute جديد في `/sys`. ده بيخلينا نشوف في الـ dmesg اسم الـ kobject وكمان اسم الـ attribute اللي اتضاف، وده مفيد جداً لفهم إيه اللي بيحصل في الـ sysfs أثناء runtime.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * sysfs_watcher.c
 * Hooks sysfs_create_file_ns() via kprobe to log every new sysfs attribute.
 */

/* --- Includes --- */
#include <linux/module.h>       /* module_init / module_exit / MODULE_* macros */
#include <linux/kernel.h>       /* pr_info / pr_err */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ... */
#include <linux/kobject.h>      /* struct kobject, kobject_name() */
#include <linux/sysfs.h>        /* struct attribute */

/*
 * الـ includes دي لازمة عشان:
 * - linux/kprobes.h بيجيب كل الـ API الخاص بالـ kprobe mechanism.
 * - linux/kobject.h بيعرّف struct kobject اللي هو أول argument في الـ function.
 * - linux/sysfs.h بيعرّف struct attribute اللي بيحوي اسم الـ attribute وصلاحياته.
 */

/* --- kprobe pre-handler --- */

/*
 * بيتنادى قبل ما sysfs_create_file_ns تنفّذ.
 * الـ regs بتحتوي على قيم الـ registers وقت الـ probe،
 * ومنها بنقدر نسحب الـ arguments (calling convention: rdi, rsi, rdx على x86_64).
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86_64:
     *   rdi = arg0 = struct kobject *kobj
     *   rsi = arg1 = const struct attribute *attr
     *   rdx = arg2 = const void *ns  (مش محتاجينه هنا)
     */
    struct kobject    *kobj = (struct kobject *)regs->di;
    struct attribute  *attr = (struct attribute *)regs->si;

    /* Validate pointers before dereferencing to avoid NULL-deref in probe */
    if (!kobj || !attr)
        return 0;

    pr_info("sysfs_watcher: new attr [%s] added to kobject [%s] (mode=%04o)\n",
            attr->name  ? attr->name  : "(null)",
            kobject_name(kobj) ? kobject_name(kobj) : "(null)",
            attr->mode);

    /*
     * pr_info بيطبع:
     *  - اسم الـ attribute (زي "uevent", "power", "vendor_id" ...)
     *  - اسم الـ kobject اللي اتضاف تحته (زي "cpu0", "eth0", ...)
     *  - الـ mode بالـ octal (مثلاً 0644 أو 0444)
     * الـ return 0 معناه "استكمل تنفيذ الـ function الأصلية بشكل عادي".
     */
    return 0;
}

/* --- kprobe struct --- */

/*
 * بنحدد الـ function اللي هنعمل عليها probe باسمها كـ string،
 * والـ kernel بيحولها لعنوانها في الذاكرة وقت الـ registration.
 */
static struct kprobe kp = {
    .symbol_name = "sysfs_create_file_ns",
    .pre_handler = handler_pre,
};

/*
 * الـ struct kprobe بيربط بين:
 * - اسم الـ symbol (الـ function المستهدفة)
 * - الـ callback (handler_pre) اللي هيتنادى قبل تنفيذها
 * اخترنا pre_handler لأننا عايزين نشوف الـ arguments قبل ما تتعدّل.
 */

/* --- module_init --- */

static int __init sysfs_watcher_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("sysfs_watcher: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("sysfs_watcher: planted kprobe on %s at %p\n",
            kp.symbol_name, kp.addr);

    /*
     * register_kprobe بتفتش عن عنوان sysfs_create_file_ns في الـ kallsyms
     * وتكتب breakpoint (int3 على x86) في أول الـ function.
     * لو رجعت قيمة سالبة يبقى في مشكلة (الـ function مش exported أو CONFIG_KPROBES=n).
     */
    return 0;
}

/* --- module_exit --- */

static void __exit sysfs_watcher_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("sysfs_watcher: kprobe removed from %s\n", kp.symbol_name);

    /*
     * unregister_kprobe ضروري في الـ exit عشان:
     * 1. يشيل الـ breakpoint من الـ function الأصلية ويرجّعها زي ما كانت.
     * 2. يضمن إن الـ handler مش هيتنادى بعد ما الـ module يتفكّ من الـ kernel،
     *    وده بيمنع use-after-free لأن الـ handler code هيتمسح من الذاكرة.
     */
}

module_init(sysfs_watcher_init);
module_exit(sysfs_watcher_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on sysfs_create_file_ns to log new sysfs attributes");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/kobject.h` | تعريف `struct kobject` و `kobject_name()` |
| `linux/sysfs.h` | تعريف `struct attribute` (name + mode) |

#### الـ `handler_pre`

الـ `pt_regs` بيحوي snapshot للـ CPU registers وقت الـ probe. على x86_64 الـ calling convention بتحط أول argument في `rdi` وتاني argument في `rsi`، فبنعمل cast مباشر منهم لـ pointers على `struct kobject` و `struct attribute`. الـ check على `NULL` ضروري لأن الـ probe بتشتغل في context حساس وأي deref غلط بيعمل kernel panic.

#### الـ `struct kprobe`

الـ `.symbol_name` بيخلي الـ kernel يحدد عنوان الـ function تلقائياً من الـ kallsyms بدل ما نحدد عنوان ثابت. الـ `.pre_handler` بيشتغل **قبل** تنفيذ الـ function المستهدفة، وده بيضمن إن الـ arguments لسه موجودة في الـ registers ولم تتغير.

#### الـ `module_init` / `module_exit`

`register_kprobe` بتكتب `int3` instruction في بداية `sysfs_create_file_ns`، لما أي كود ينادي الـ function دي الـ CPU بيدخل في الـ kprobe mechanism وينادي الـ handler. `unregister_kprobe` في الـ exit بتشيل الـ `int3` وترجّع الـ original bytes، وده **إجباري** عشان لو الـ module اتفكّ والـ breakpoint فضل موجود هيتنادى كود اتمسح من الذاكرة.

---

### تجربة الـ Module

```bash
# بناء الـ module (Makefile بجانب sysfs_watcher.c)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod sysfs_watcher.ko

# نشوف اللوج في الـ dmesg لما أي device يتضاف
sudo dmesg -w | grep sysfs_watcher

# مثلاً بعد تحميل USB device هنشوف:
# sysfs_watcher: new attr [uevent] added to kobject [1-1] (mode=0200)
# sysfs_watcher: new attr [idVendor] added to kobject [1-1:1.0] (mode=0444)

# إزالة الـ module
sudo rmmod sysfs_watcher
```

---

### Makefile

```makefile
obj-m += sysfs_watcher.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
