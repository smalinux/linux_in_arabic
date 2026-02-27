## Phase 1: الصورة الكبيرة ببساطة

### ينتمي لأي subsystem؟

الـ `faux.h` جزء من **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS** subsystem — المسؤول عنه Greg Kroah-Hartman وRafael J. Wysocki، وهو قلب driver model في Linux kernel.

---

### الفكرة من البداية — قبل أي كود

#### المشكلة الأصلية

في Linux، أي device لازم يكون جزء من **bus**. الـ bus هو "الطريق" اللي بيوصّل الـ hardware بالـ driver. عندك `PCI bus`، `USB bus`، `platform bus`، وهكذا.

الـ **`platform bus`** كان الحل التقليدي لأي device مش له bus حقيقي — يعني hardware مدمج في الـ SoC أو boards مش بتستخدم PCI/USB. لكن المشكلة إن `platform_device` بيجبر معاه كمية كبيرة من complexity:
- لازم تعرّف resources (IRQ، memory regions).
- لازم تتعامل مع driver binding logic.
- لازم تفهم Device Tree أو ACPI أو platform_data.
- لازم كل هذا حتى لو الـ device مش بيحتاج أي resources أصلاً.

#### القصة الحقيقية — ليه الـ faux bus؟

تخيّل إنك بتكتب driver لـ firmware loader، أو crypto module، أو أي شيء software-only محتاج بس "كيان" في الـ kernel يتسجّل تحته — محتاج يظهر في `/sys`، يخزّن private data، يعمل probe/remove lifecycle — بس **مفيش أي hardware resources**.

لو استخدمت `platform_device`، هتضطر تعبي في كود مش محتاجه. الـ `faux bus` جاي يقول: **"عايز device بدون أي تعقيد؟ استخدم faux."**

الاسم نفسه فاضح — "faux" بالفرنسي معناها **"مزيف"** أو **"وهمي"**. device حقيقي في نظر الـ kernel، لكن مش بيمثّل hardware فعلي.

---

### الهدف من الملف

الـ `include/linux/device/faux.h` هو الـ **public API header** للـ faux bus subsystem. بيعرّف:

| الـ structure / function | الدور |
|---|---|
| `struct faux_device` | الـ wrapper فوق `struct device` — هو الـ "device" الوهمي |
| `struct faux_device_ops` | callbacks اختيارية: `probe` و`remove` |
| `faux_device_create()` | أنشئ device وسجّله في الـ driver core |
| `faux_device_create_with_groups()` | نفس الفكرة + sysfs attribute groups |
| `faux_device_destroy()` | احذف الـ device وحرّر الذاكرة |
| `faux_device_get_drvdata()` | اقرأ الـ private driver data |
| `faux_device_set_drvdata()` | اكتب الـ private driver data |

---

### الصورة الكبيرة — ELI5

#### تشبيه من الواقع

تخيّل مبنى فيه إدارة تسجيل (= الـ kernel). أي موظف جديد (= driver) لازم يكون له مكتب (= device) عشان يستلم مهامه رسمياً. المكتب الحقيقي محتاج كهرباء وتليفون وإنترنت (= hardware resources). لكن لو الموظف شغلته بالكامل على الكمبيوتر ومش محتاج أي توصيلات خاصة — تعطيه **مكتب وهمي بدون أي بنية تحتية** (= faux device).

#### الـ lifecycle ببساطة

```
faux_device_create("my_dev", NULL, &my_ops)
         │
         ▼
   كـ kernel ينشئ struct faux_object داخلياً
         │
         ▼
   device_add() → يسجّل الـ device في sysfs
         │
         ▼
   faux_match() → دايماً بيقول "matched"
         │
         ▼
   faux_probe() → بيستدعي ops->probe() لو موجود
         │
         ▼
   الـ device جاهز → يظهر في /sys/bus/faux/devices/
         │
         ▼
faux_device_destroy() → device_del() + put_device() + kfree()
```

#### الفرق بين faux_device وplatform_device

| | `platform_device` | `faux_device` |
|---|---|---|
| **resources** | IRQ، MMIO، DMA | لا شيء |
| **التعقيد** | كبير | بسيط جداً |
| **Device Tree** | مدعوم | غير مدعوم |
| **الاستخدام** | hardware حقيقي | software-only |
| **driver binding** | معقد | واحد driver بيـ match كل حاجة |

#### ليه مفيد؟

- بتحتاج تحمّل firmware وتسجّل نفسك في الـ kernel؟ → استخدم `faux_device`.
- بتعمل virtual device للـ testing؟ → `faux_device`.
- بتبني subsystem محتاج يظهر في sysfs بدون hardware؟ → `faux_device`.
- بتكتب Rust driver وعايز أبسط طريقة؟ → `faux_device` (مدعوم في `rust/kernel/faux.rs`).

---

### الملفات المكوّنة للـ subsystem

#### Core Files

| الملف | الدور |
|---|---|
| `drivers/base/faux.c` | التنفيذ الكامل: bus init، probe، remove، create، destroy |
| `include/linux/device/faux.h` | الـ public API — هذا الملف |

#### Dependencies المباشرة

| الملف | الدور |
|---|---|
| `include/linux/device.h` | يعرّف `struct device`، `bus_type`، `device_driver` |
| `include/linux/container_of.h` | يعرّف `container_of_const` المستخدم في `to_faux_device()` |
| `drivers/base/base.h` | internal driver core headers |

#### Rust Bindings

| الملف | الدور |
|---|---|
| `rust/kernel/faux.rs` | Rust-safe wrapper للـ faux API |
| `samples/rust/rust_driver_faux.rs` | مثال عملي لكتابة faux driver بـ Rust |

#### ملفات مجاورة مهمة للفهم

| الملف | الدور |
|---|---|
| `include/linux/platform_device.h` | البديل الأثقل — اقرأه للمقارنة |
| `drivers/base/platform.c` | تنفيذ الـ platform bus |
| `drivers/base/core.c` | قلب الـ driver core: `device_add`، `device_del` |
| `include/linux/device.h` | أساس كل device في الـ kernel |
## Phase 2: شرح الـ Faux Device Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في kernel driver model، كل device لازم يكون على **bus**. الـ bus هو الـ abstraction اللي بيربط بين driver وdevice من خلال آلية **probe/match**. الـ kernel عنده buses حقيقية زي PCI وUSB وI2C، وعنده buses افتراضية زي **platform bus** اللي بتستخدم للـ SoC peripherals اللي مش بيتكلموا عبر بروتوكول قياسي.

لكن في حالات كتير، المطور محتاج **device مش بيمثل أي hardware حقيقي**، ومحتاج:
- يعمل entry في sysfs
- يعمل firmware load
- يعمل kobject للـ reference counting
- يعمل devres (device-managed resources)
- يربط حاجة بالـ device hierarchy

لو استخدم `platform_device`، هيضطر:
1. يعرّف `struct platform_device` مع resources وIDs وكل حاجة
2. يعرّف `struct platform_driver` مع `.probe` و`.remove`
3. يعمل `platform_device_register()` ثم `platform_driver_register()`
4. يتعامل مع الـ match logic بين driver وdevice

ده overkill لو الـ device مش ليه resources وعايز بس "يعلّق" حاجة على الـ device tree.

**الحل؟** الـ `faux_device` — أبسط device ممكن في الـ kernel.

---

### الحل — نهج الـ kernel

الـ **faux bus** هو internal bus خاص يسجّله الـ kernel تلقائياً. أي `faux_device` بيتضاف على الـ faux bus أوتوماتيك.

المطور مش محتاج يعرّف bus، مش محتاج يعرّف driver منفصل. الـ framework بيوفرله function واحدة تعمل الـ device وتعمل الـ probe في نفس الوقت:

```c
struct faux_device *faux_device_create(const char *name,
                                       struct device *parent,
                                       const struct faux_device_ops *faux_ops);
```

وفي النهاية:

```c
void faux_device_destroy(struct faux_device *faux_dev);
```

بس. مفيش أي حاجة تانية.

---

### تشبيه من الواقع — الـ Post-it Note على الـ Corkboard

تخيل مكتب فيه **لوحة إعلانات (corkboard)** — دي الـ **faux bus**.

- لو عايز تعلق إعلان بسيط، بتاخد **post-it note** وبتلصقه. دي الـ `faux_device`.
- الإعلان مش بحاجة لـ power outlet، مش محتاج wire، مش محتاج special bracket — بس اسم وموضع على اللوحة.
- لو عايز تتصرف لما الإعلان ينشر أو يُشال، بتكتب تعليمات على ورقة تانية وبتربطها بيه. دي الـ `faux_device_ops`.

قابل النموذج على الـ kernel:

| التشبيه | الـ Kernel Concept |
|---|---|
| لوحة الإعلانات (corkboard) | الـ `faux_bus` — internal `bus_type` |
| الـ post-it note | `struct faux_device` |
| اسم الإعلان | `name` parameter في `faux_device_create()` |
| ربط الإعلان بجزء من اللوحة | `parent` parameter — موقع الـ device في الـ hierarchy |
| التعليمات المربوطة بالإعلان | `struct faux_device_ops` |
| لما حد يقرأ الإعلان (يشغّله) | `.probe` callback |
| لما حد ينزع الإعلان | `.remove` callback |
| رمي الإعلان في الأوراق المهمة (sysfs) | `faux_device_create_with_groups()` |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Driver Core (drivers/base/)            │  │
│  │                                                          │  │
│  │   ┌──────────────┐   ┌──────────────┐  ┌─────────────┐  │  │
│  │   │  PCI bus     │   │ Platform bus │  │  faux bus   │  │  │
│  │   │  bus_type    │   │  bus_type    │  │  bus_type   │  │  │
│  │   └──────┬───────┘   └──────┬───────┘  └──────┬──────┘  │  │
│  │          │                  │                  │         │  │
│  │   [real HW devices]  [SoC peripherals]  [virtual devs]  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               Faux Device Layer                          │  │
│  │                                                          │  │
│  │  faux_device_create("my-dev", parent, &my_ops)          │  │
│  │         │                                                │  │
│  │         ▼                                                │  │
│  │  ┌─────────────────┐      ┌──────────────────────┐      │  │
│  │  │  faux_device    │      │  faux_device_ops      │      │  │
│  │  │  ┌───────────┐  │      │  ┌────────────────┐  │      │  │
│  │  │  │ struct    │  │◄────►│  │ .probe()       │  │      │  │
│  │  │  │ device    │  │      │  │ .remove()      │  │      │  │
│  │  │  │  .kobj    │  │      │  └────────────────┘  │      │  │
│  │  │  │  .bus ────┼──┼──────┼──► faux_bus          │      │  │
│  │  │  │  .parent  │  │      └──────────────────────┘      │  │
│  │  │  │  .driver_data     │  │                             │  │
│  │  │  └───────────┘  │      │                              │  │
│  │  └─────────────────┘      │                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              sysfs (/sys/devices/)                       │  │
│  │  /sys/devices/faux/my-dev/                               │  │
│  │      ├── uevent                                          │  │
│  │      ├── power/                                          │  │
│  │      └── [custom attribute groups]                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**المستخدمون (Consumers) — مين بيستخدم faux_device؟**

- Subsystems محتاجة device لـ firmware loading (مثلاً thermal subsystem)
- Virtual devices للـ testing وdebugging
- Drivers بتحتاج kobject/sysfs entry بدون hardware
- الـ IIO subsystem لبعض virtual sensors
- أي subsystem عايز يعمل devres-managed objects

**الـ Provider — مين بيوفر الـ faux bus؟**

الـ `drivers/base/faux.c` — كود واحد في الـ kernel بيسجل الـ faux bus وبيتحكم في كل الـ lifecycle.

---

### الـ Core Abstraction — الفكرة المركزية

الـ abstraction الأساسية هي: **"device بدون resources"**.

في الـ kernel driver model، كل device بيمر بـ lifecycle محدد:

```
device_register()  →  match()  →  probe()  →  [active]  →  remove()
```

الـ faux framework بيبسّط ده عن طريق دمج الـ device registration والـ driver binding في خطوة واحدة. الـ "driver" هنا مش entity منفصلة — هو مجرد `faux_device_ops` بيتحدد وقت الـ creation.

```
faux_device_create()
       │
       ├── alloc struct faux_device
       ├── set faux_dev->dev.bus = &faux_bus
       ├── device_register(&faux_dev->dev)
       └── call ops->probe(faux_dev)    ← immediate, no match needed
```

مفيش match logic، مفيش driver table، مفيش deferred probe. الـ probe بيحصل مباشرة.

---

### تفاصيل الـ Structs وعلاقتها

#### `struct faux_device`

```c
struct faux_device {
    struct device dev;   /* embedded struct device — must be first */
};
```

**ليه الـ embedding وmش pointer؟**

الـ kernel بيستخدم pattern الـ **embedded struct** بدل الـ inheritance. الـ `struct device` هو الـ base class اللي بيمثل أي device في الـ kernel. بـ embedding بدل pointer:

1. memory allocation واحدة بدل اتنين
2. الـ `container_of` macro بيشتغل بكفاءة — بيحسب offset وقت الـ compile
3. الـ device lifetime مرتبط بالـ faux_device نفسه

```
faux_device في الـ memory:
┌────────────────────────────────────────┐
│  struct device dev                     │
│  ┌────────────────────────────────┐    │
│  │ struct kobject kobj            │    │
│  │ struct device *parent          │    │
│  │ const struct bus_type *bus ────┼────┼──► faux_bus
│  │ struct device_driver *driver   │    │
│  │ void *driver_data              │    │
│  │ struct dev_pm_info power       │    │
│  │ ...                            │    │
│  └────────────────────────────────┘    │
│                                        │
│  [offset = 0, dev is first member]     │
└────────────────────────────────────────┘
         ▲
         │
         │  to_faux_device(dev_ptr):
         │  container_of_const(dev_ptr, struct faux_device, dev)
         │  = (faux_device*)((char*)dev_ptr - offsetof(faux_device, dev))
         │  = (faux_device*)dev_ptr  [لأن offset = 0]
```

#### الـ `to_faux_device` macro

```c
#define to_faux_device(x) container_of_const((x), struct faux_device, dev)
```

الـ `container_of_const` (من `linux/container_of.h`) بيحافظ على الـ `const` qualifier:

```c
// لو x هو const struct device * → ترجع const struct faux_device *
// لو x هو struct device *       → ترجع struct faux_device *
```

ده أهم من `container_of` العادي لأنه type-safe.

#### `struct faux_device_ops`

```c
struct faux_device_ops {
    int  (*probe)(struct faux_device *faux_dev);
    void (*remove)(struct faux_device *faux_dev);
};
```

كلاهما optional — لو مش محتاج تعمل حاجة وقت الـ probe، تحط `NULL`.

مقارنة بالـ `platform_driver`:

| | `faux_device_ops` | `platform_driver` |
|---|---|---|
| هيكل منفصل | لا — جزء من نفس call | نعم — يتسجل لوحده |
| match logic | لا يوجد | نعم (by name/id table) |
| resources | لا | نعم (IRQ, MMIO, etc) |
| device tree binding | لا | نعم (of_match_table) |
| module owner | محمول من الـ create call | `.owner = THIS_MODULE` |

---

### الـ drvdata Pattern

```c
static inline void *faux_device_get_drvdata(const struct faux_device *faux_dev)
{
    return dev_get_drvdata(&faux_dev->dev);
}

static inline void faux_device_set_drvdata(struct faux_device *faux_dev, void *data)
{
    dev_set_drvdata(&faux_dev->dev, data);
}
```

الـ `driver_data` field في `struct device` هو `void *` — pointer عام يخزن فيه الـ driver أي private data. الـ faux framework بيوفر wrapper functions بتتعامل مع الـ inner `dev` struct بدل ما تعرضه مباشرة.

**Pattern نموذجي للاستخدام:**

```c
struct my_private_data {
    int counter;
    struct work_struct work;
};

static int my_faux_probe(struct faux_device *faux_dev)
{
    struct my_private_data *priv;

    priv = devm_kzalloc(&faux_dev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    faux_device_set_drvdata(faux_dev, priv);  /* حفظ الـ private data */
    return 0;
}

static void my_faux_remove(struct faux_device *faux_dev)
{
    struct my_private_data *priv = faux_device_get_drvdata(faux_dev);
    /* priv هيتحرر تلقائياً لأنه devm-allocated */
}

static const struct faux_device_ops my_ops = {
    .probe  = my_faux_probe,
    .remove = my_faux_remove,
};
```

---

### الـ `faux_device_create_with_groups`

```c
struct faux_device *faux_device_create_with_groups(
    const char *name,
    struct device *parent,
    const struct faux_device_ops *faux_ops,
    const struct attribute_group **groups);
```

الـ `attribute_group` هو مجموعة من `sysfs attributes` بتظهر كـ files تحت `/sys/devices/faux/<name>/`. ده مفيد لو الـ device محتاج يكشف معلومات لـ userspace.

**مثال:**

```c
/* attribute يظهر كـ /sys/devices/faux/my-dev/version */
static ssize_t version_show(struct device *dev,
                             struct device_attribute *attr, char *buf)
{
    return sysfs_emit(buf, "1.0\n");
}
static DEVICE_ATTR_RO(version);

static struct attribute *my_attrs[] = {
    &dev_attr_version.attr,
    NULL,
};

static const struct attribute_group my_group = {
    .attrs = my_attrs,
};

static const struct attribute_group *my_groups[] = {
    &my_group,
    NULL,
};

/* إنشاء الـ device مع الـ sysfs files */
fdev = faux_device_create_with_groups("my-dev", NULL, &my_ops, my_groups);
```

---

### الـ kobject / sysfs Subsystem — شرح مختصر

الـ **kobject subsystem** هو الـ subsystem اللي بيبني الـ `/sys` hierarchy. كل `struct device` بيحتوي على `struct kobject kobj` اللي بيمثل الـ node في sysfs tree. الـ faux_device بيورث ده أوتوماتيك لأنه بيتضمن `struct device`.

الـ **devres (device resource management)** هو subsystem تاني مهم — بيسمح بتخصيص resources بتتحرر أوتوماتيك لما الـ device يتشال (`devm_kmalloc`, `devm_request_irq`, إلخ). الـ faux_device بيدعم ده كاملاً لأن `&faux_dev->dev` هو valid device pointer.

---

### الـ Ownership — إيه اللي الـ Framework بيتملكه وإيه اللي بيفوّضه للـ Driver

| المسؤولية | الـ faux framework | الـ Driver |
|---|---|---|
| إنشاء وتسجيل الـ device | ✓ | — |
| الـ faux bus registration | ✓ | — |
| الـ sysfs node creation | ✓ | — |
| الـ kobject lifecycle | ✓ | — |
| reference counting | ✓ (عبر `get_device`/`put_device`) | — |
| الـ probe callback | يستدعيه | ينفّذه |
| الـ remove callback | يستدعيه | ينفّذه |
| الـ private data | — | ✓ (drvdata) |
| الـ sysfs attributes المخصصة | — | ✓ (attribute groups) |
| resource management | — | ✓ (devm_* أو يدوي) |
| الـ parent device | — | ✓ (يحدده وقت الـ create) |

---

### مقارنة سريعة: faux vs platform vs miscdevice

```
المشكلة: محتاج device بسيط في الـ kernel

         ┌─────────────────────────────────────────────────┐
         │  هل في hardware resources (MMIO, IRQ, clocks)?  │
         └─────────────────┬───────────────────────────────┘
                           │
               ┌───────────┴───────────┐
               │ نعم                   │ لا
               ▼                       ▼
      ┌─────────────────┐   ┌──────────────────────────────┐
      │ platform_device │   │  هل محتاج character device?  │
      │ + platform_driver│   └──────────┬───────────────────┘
      └─────────────────┘              │
                              ┌────────┴────────┐
                              │ نعم              │ لا
                              ▼                  ▼
                    ┌─────────────────┐  ┌─────────────────┐
                    │   misc_register │  │  faux_device    │
                    │   /dev/mydev    │  │  sysfs only     │
                    └─────────────────┘  └─────────────────┘
```

الـ `faux_device` هو الأبسط — لما مش محتاج لا `/dev` entry ولا hardware resources، هو الخيار الصح.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — Flags, Enums, Config Options

#### **الـ probe_type** لـ `faux_driver`

| القيمة | المعنى |
|---|---|
| `PROBE_FORCE_SYNCHRONOUS` | الـ probe بيتعمل synchronously — مش في background thread. ده مهم عشان `faux_device_create` لازم تعرف نتيجة الـ probe قبل ما ترجع. |

#### **الـ driver flags** في `faux_driver`

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `suppress_bind_attrs` | `true` | بيخفي ملفات `bind`/`unbind` من sysfs — المستخدم مش المفروض يتحكم في ده manually. |

#### **الـ `device_link_state` enum** (من `device.h` — مرتبط بـ `struct device`)

| القيمة | المعنى |
|---|---|
| `DL_STATE_NONE` | مفيش tracking للـ driver |
| `DL_STATE_DORMANT` | لا supplier ولا consumer موجودين |
| `DL_STATE_AVAILABLE` | الـ supplier موجود، الـ consumer لأ |
| `DL_STATE_CONSUMER_PROBE` | الـ consumer بيعمل probe |
| `DL_STATE_ACTIVE` | الاتنين موجودين |
| `DL_STATE_SUPPLIER_UNBIND` | الـ supplier بيعمل unbind |

#### **الـ `dl_dev_state` enum**

| القيمة | المعنى |
|---|---|
| `DL_DEV_NO_DRIVER` | مفيش driver |
| `DL_DEV_PROBING` | driver بيعمل probe |
| `DL_DEV_DRIVER_BOUND` | driver مربوط |
| `DL_DEV_UNBINDING` | driver بيعمل unbind |

#### **الـ Device Link Flags** (`DL_FLAG_*`)

| الـ Flag | الـ Bit | المعنى |
|---|---|---|
| `DL_FLAG_STATELESS` | 0 | الـ core مش بيحذف الـ link تلقائياً |
| `DL_FLAG_AUTOREMOVE_CONSUMER` | 1 | احذف الـ link لما الـ consumer يعمل unbind |
| `DL_FLAG_PM_RUNTIME` | 2 | استخدمه في runtime PM |
| `DL_FLAG_RPM_ACTIVE` | 3 | شغّل `pm_runtime_get_sync` على الـ supplier |
| `DL_FLAG_AUTOREMOVE_SUPPLIER` | 4 | احذف الـ link لما الـ supplier يعمل unbind |
| `DL_FLAG_AUTOPROBE_CONSUMER` | 5 | probe الـ consumer تلقائياً بعد الـ supplier |
| `DL_FLAG_MANAGED` | 6 | internal — الـ core بيتابع الـ drivers |
| `DL_FLAG_SYNC_STATE_ONLY` | 7 | بس بيأثر على `sync_state()` |
| `DL_FLAG_INFERRED` | 8 | اتستنتج من firmware مش من الـ driver |
| `DL_FLAG_CYCLE` | 9 | الـ link جزء من dependency cycle |

> **ملاحظة مهمة:** الـ faux subsystem ما بيستخدمش device links — بس `struct device` بيحمل `struct dev_links_info` دايماً.

---

### 1. الـ Structs المهمة

#### `struct faux_device` — الـ Public Handle

**الغرض:** الواجهة العامة اللي الـ driver بيشتغل بيها. بتعرّف device وهمية مش ليها resources حقيقية.

```c
/* include/linux/device/faux.h */
struct faux_device {
    struct device dev;   /* embedded device — ده الـ core representation */
};
```

| الـ Field | النوع | المعنى |
|---|---|---|
| `dev` | `struct device` | الـ device الحقيقي اللي بيتسجل في الـ driver core |

**الاتصال بالـ structs التانية:**
- **مضمّن جوا** `struct faux_object` (داخلي في `faux.c`)
- بيوصل لـ `struct device` اللي فيها `bus`, `driver`, `parent`, `mutex`, إلخ

**الـ Macro:**
```c
#define to_faux_device(x) container_of_const((x), struct faux_device, dev)
```
بيستخدمه الـ kernel لما عنده `struct device *` ويعوز يوصل لـ `struct faux_device`.

---

#### `struct faux_device_ops` — الـ Callbacks

**الغرض:** بتحدد الـ hooks اللي الـ driver عايز يتنفذ وقت الـ probe والـ remove.

```c
/* include/linux/device/faux.h */
struct faux_device_ops {
    int  (*probe)(struct faux_device *faux_dev);   /* optional — رد بـ 0 يعني نجاح */
    void (*remove)(struct faux_device *faux_dev);  /* optional */
};
```

| الـ Field | المعنى |
|---|---|
| `probe` | بيتنادى قبل ما الـ device يكون fully bound. لو رجع قيمة سالبة، الـ creation بتفشل. ممكن يكون NULL. |
| `remove` | بيتنادى لما الـ device بيتشال. ممكن يكون NULL. |

**الاتصال بالـ structs التانية:**
- بيتخزن في `struct faux_object` داخلياً
- الـ `faux_probe()` و`faux_remove()` في `faux.c` بينادوا عليه

---

#### `struct faux_object` — الـ Internal Wrapper (في `faux.c` فقط)

**الغرض:** الـ struct الداخلية اللي بتجمع الـ `faux_device` مع الـ ops والـ groups. المستخدم خارج `faux.c` مش بيشوفها.

```c
/* drivers/base/faux.c — internal only */
struct faux_object {
    struct faux_device          faux_dev;   /* embedded — الـ public face */
    const struct faux_device_ops *faux_ops; /* الـ callbacks المخزنة */
    const struct attribute_group **groups;  /* sysfs attribute groups */
};

#define to_faux_object(dev) \
    container_of_const(dev, struct faux_object, faux_dev.dev)
```

| الـ Field | النوع | المعنى |
|---|---|---|
| `faux_dev` | `struct faux_device` | مضمّن — ده اللي الـ allocator بيرجعه للـ driver |
| `faux_ops` | `const struct faux_device_ops *` | الـ callbacks المحفوظة لوقت الـ probe/remove |
| `groups` | `const struct attribute_group **` | الـ sysfs attributes اللي بتتضاف بعد الـ probe |

**الـ Macro** بياخد `struct device *` ويوصل للـ `faux_object` الأكبر منه.

---

#### `struct bus_type faux_bus_type` — الـ Bus (static في `faux.c`)

```c
static const struct bus_type faux_bus_type = {
    .name   = "faux",
    .match  = faux_match,   /* دايماً بيرجع 1 */
    .probe  = faux_probe,   /* بيستدعي faux_ops->probe */
    .remove = faux_remove,  /* بيستدعي faux_ops->remove */
};
```

| الـ Field | المعنى |
|---|---|
| `name` | اسم الـ bus في sysfs: `/sys/bus/faux/` |
| `match` | دايماً بيرجع 1 — في driver واحد بس لكل الأجهزة |
| `probe` | الـ core بيناديها لما الـ device يتسجل |
| `remove` | الـ core بيناديها لما الـ device بيتشال |

---

#### `struct device_driver faux_driver` — الـ Driver (static في `faux.c`)

```c
static struct device_driver faux_driver = {
    .name                = "faux_driver",
    .bus                 = &faux_bus_type,
    .probe_type          = PROBE_FORCE_SYNCHRONOUS,
    .suppress_bind_attrs = true,
};
```

**ليه `PROBE_FORCE_SYNCHRONOUS`?** لأن `faux_device_create` لازم تتأكد إن الـ probe خلص قبل ما ترجع — لو الـ probe اتعمل في thread تاني هتبقى race condition.

---

#### `struct device` — الـ Core Device (من `include/linux/device.h`)

**الـ fields المهمة اللي الـ faux بيستخدمها:**

| الـ Field | النوع | الاستخدام في faux |
|---|---|---|
| `parent` | `struct device *` | الـ parent اللي اتحدد في `faux_device_create` أو `faux_bus_root` |
| `bus` | `const struct bus_type *` | بيتضبط على `&faux_bus_type` |
| `driver` | `struct device_driver *` | بيتعبى بعد الـ probe — لو NULL يعني الـ probe فشل |
| `driver_data` | `void *` | الـ private driver data — `faux_device_get/set_drvdata` بتوصله |
| `release` | function pointer | بيتضبط على `faux_device_release` — بيعمل `kfree` على الـ `faux_object` |
| `mutex` | `struct mutex` | بيحمي الـ calls للـ driver |

---

### 2. مخطط علاقات الـ Structs

```
  ┌─────────────────────────────────────────────────────────┐
  │                    struct faux_object                   │
  │  (internal — allocated by faux_device_create_with_groups│
  │                    via kzalloc)                         │
  │                                                         │
  │  ┌──────────────────────────────────┐                   │
  │  │        struct faux_device        │  ◄── returned     │
  │  │                                  │      to caller    │
  │  │  ┌────────────────────────────┐  │                   │
  │  │  │       struct device dev    │  │                   │
  │  │  │  .parent ──────────────────┼──┼──► struct device  │
  │  │  │  .bus ─────────────────────┼──┼──► faux_bus_type  │
  │  │  │  .driver ──────────────────┼──┼──► faux_driver    │
  │  │  │  .driver_data (void*)      │  │    (after probe)  │
  │  │  │  .release = faux_device_   │  │                   │
  │  │  │             release()      │  │                   │
  │  │  │  .mutex                    │  │                   │
  │  │  └────────────────────────────┘  │                   │
  │  └──────────────────────────────────┘                   │
  │                                                         │
  │  .faux_ops ────────────────────────────────────────────►│
  │  │         struct faux_device_ops                       │
  │  │         .probe(faux_dev)                             │
  │  │         .remove(faux_dev)                            │
  │                                                         │
  │  .groups ──────────────────────────────────────────────►│
  │           const struct attribute_group **               │
  │           (added to sysfs after probe succeeds)         │
  └─────────────────────────────────────────────────────────┘


  faux_bus_type ──────────────────────────────────────────────┐
  .match = faux_match()     (always returns 1)                │
  .probe = faux_probe()  ───► calls faux_ops->probe()         │
  .remove = faux_remove()───► calls faux_ops->remove()        │
                                                              │
  faux_driver ────────────────────────────────────────────────┤
  .bus = &faux_bus_type                                       │
  .probe_type = PROBE_FORCE_SYNCHRONOUS                       │
  .suppress_bind_attrs = true                                 │
                                                              │
  faux_bus_root (struct device*) ─────────────────────────────┘
  default parent لكل faux devices لو parent = NULL
```

---

### 3. مخطط دورة حياة الـ Device

```
  BOOT / MODULE INIT
  ──────────────────
  faux_bus_init()
    │
    ├─► kzalloc(faux_bus_root)
    │     device_register(faux_bus_root)    ← /sys/devices/faux/
    │
    ├─► bus_register(&faux_bus_type)        ← /sys/bus/faux/
    │
    └─► driver_register(&faux_driver)       ← /sys/bus/faux/drivers/faux_driver/


  CREATE
  ──────
  caller: faux_device_create(name, parent, ops)
    │
    └─► faux_device_create_with_groups(name, parent, ops, NULL)
          │
          ├─► kzalloc(sizeof(faux_object))
          │
          ├─► faux_obj->faux_ops = ops
          ├─► faux_obj->groups   = groups
          │
          ├─► device_initialize(dev)
          ├─► dev->release = faux_device_release
          ├─► dev->parent  = parent ?: faux_bus_root
          ├─► dev->bus     = &faux_bus_type
          ├─► dev_set_name(dev, name)
          ├─► device_set_pm_not_required(dev)
          │
          └─► device_add(dev)
                │
                └─► driver core: faux_match() → returns 1
                      │
                      └─► faux_probe(dev)
                            │
                            ├─► faux_ops->probe(faux_dev)  [if not NULL]
                            │     └── returns 0: OK / negative: FAIL → return NULL
                            │
                            └─► device_add_groups(dev, groups)
                                  └── if FAIL → faux_ops->remove(faux_dev)

          ── check: dev->driver != NULL ?
          │    NO  → faux_device_destroy() → return NULL
          └── YES → return &faux_obj->faux_dev


  USAGE
  ─────
  caller يحتفظ بـ struct faux_device*
    │
    ├─► faux_device_set_drvdata(faux_dev, my_data)
    │     └─► dev_set_drvdata(&faux_dev->dev, my_data)
    │
    └─► faux_device_get_drvdata(faux_dev)
          └─► dev_get_drvdata(&faux_dev->dev)


  DESTROY
  ───────
  caller: faux_device_destroy(faux_dev)
    │
    ├─► device_del(dev)
    │     │
    │     └─► driver core: faux_remove(dev)
    │               │
    │               ├─► device_remove_groups(dev, groups)
    │               └─► faux_ops->remove(faux_dev)  [if not NULL]
    │
    └─► put_device(dev)
          │
          └── refcount → 0 → faux_device_release(dev)
                                └─► kfree(faux_object)  ← الـ memory اتحررت
```

---

### 4. مخطط الـ Call Flow

#### إنشاء الـ device

```
driver code
  └─► faux_device_create("my-fw-loader", NULL, &my_ops)
        └─► faux_device_create_with_groups("my-fw-loader", NULL, &my_ops, NULL)
              │
              ├── kzalloc(faux_object)              [GFP_KERNEL]
              ├── device_initialize(&dev)
              ├── dev->bus = &faux_bus_type
              ├── dev_set_name(&dev, "my-fw-loader")
              │
              └── device_add(&dev)
                    │
                    └── driver_core: bus_probe_device()
                          │
                          └── faux_bus_type.match(dev, drv) → 1
                                │
                                └── faux_bus_type.probe(dev)
                                      │  [= faux_probe()]
                                      │
                                      ├── to_faux_object(dev) → faux_obj
                                      ├── faux_ops->probe(faux_dev)  ← driver code
                                      │     └── [driver initializes hardware state]
                                      │
                                      └── device_add_groups(dev, groups)
                                            └── creates sysfs files

              ── dev->driver != NULL?  YES
              └── return &faux_obj->faux_dev   ← caller gets handle
```

#### تدمير الـ device

```
driver code
  └─► faux_device_destroy(faux_dev)
        │
        ├── device_del(&faux_dev->dev)
        │     │
        │     └── driver_core: __device_release_driver()
        │               │
        │               └── faux_bus_type.remove(dev)
        │                     │  [= faux_remove()]
        │                     │
        │                     ├── device_remove_groups(dev, groups)
        │                     │     └── removes sysfs files
        │                     │
        │                     └── faux_ops->remove(faux_dev)  ← driver code
        │                           └── [driver cleans up]
        │
        └── put_device(&faux_dev->dev)
              └── kobject refcount → 0
                    └── faux_device_release(dev)
                          └── kfree(faux_obj)
```

#### الـ probe فشل

```
faux_device_create(name, parent, ops)
  └─► device_add(dev)
        └─► faux_probe(dev)
              └─► faux_ops->probe(faux_dev) → returns -ENODEV
                    └── faux_probe returns -ENODEV
                          └── driver core: probe failed, dev->driver = NULL

  ── check: dev->driver == NULL  → TRUE
  └─► faux_device_destroy(faux_dev)   ← cleanup
  └─► return NULL                     ← caller يعرف إن فيه error
```

---

### 5. استراتيجية الـ Locking

#### مفيش locks خاصة بالـ faux subsystem

الـ faux subsystem نفسه ما عندوش locks مخصصة — بيعتمد بالكامل على الـ driver core locking.

#### الـ Locks اللي الـ driver core بيستخدمها

| الـ Lock | النوع | بيحمي إيه | مين يمسكه |
|---|---|---|---|
| `dev->mutex` | `struct mutex` | الـ calls للـ driver (probe/remove) | الـ driver core تلقائياً |
| `device_lock(dev)` | wrapper على `dev->mutex` | ضمان إن probe/remove مش بيحصلوا concurrent | الـ core قبل ما يستدعي probe أو remove |
| `kobject` internal lock | spinlock | الـ refcount (kref) | الـ core لما `put_device` / `get_device` |

#### ترتيب الـ Locks (Lock Ordering)

```
device_lock(parent)          ← الـ parent الأعلى
  └── device_lock(child)     ← الـ child device
        └── [driver probe/remove runs here]
```

> مهم: الـ faux_ops->probe() و faux_ops->remove() بيتنادوا وهو مسكك `dev->mutex` — يعني الـ driver **ممنوع** يحاول يعمل `device_lock()` على نفس الـ device من جوا الـ probe/remove أو هيحصل deadlock.

#### الـ `PROBE_FORCE_SYNCHRONOUS` وعلاقته بالـ Locking

لو الـ probe بيحصل في thread تاني، الـ `dev->driver` مش هيتضبط قبل ما `faux_device_create` تيجي تتحقق منه — وده race condition. بسبب إن الـ faux بيستخدم `PROBE_FORCE_SYNCHRONOUS`، الـ probe بيخلص كامل **قبل** ما `device_add()` يرجع، فمفيش race.

#### الـ `kref` / refcount

```
device_initialize()   → kref = 1
device_add()          → get_device() → kref = 2  [لما بيسجّله]
                       (put_device after add internal) → kref = 1
faux_device_destroy() → put_device() → kref = 0 → faux_device_release() → kfree()
```

**ملاحظة:** لو `device_add()` فشل، الكود بيستدعي `put_device()` اللي بيوصل الـ kref لـ 0 وبيحرر الـ memory تلقائياً — مفيش leak.
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet للـ Functions والـ APIs

| Function / Macro | النوع | الغرض |
|---|---|---|
| `faux_device_create()` | exported API | ينشئ ويسجل faux device بدون sysfs groups |
| `faux_device_create_with_groups()` | exported API | ينشئ faux device مع sysfs attribute groups |
| `faux_device_destroy()` | exported API | يحذف ويفكك faux device |
| `faux_device_get_drvdata()` | inline helper | يجيب driver private data |
| `faux_device_set_drvdata()` | inline helper | يخزن driver private data |
| `to_faux_device(x)` | macro | يحول `struct device *` لـ `struct faux_device *` |
| `faux_bus_init()` | internal `__init` | يبدّل الـ faux bus infrastructure عند boot |
| `faux_match()` | internal bus callback | matching دايماً ينجح — bus واحدة وdriver واحد |
| `faux_probe()` | internal bus callback | يستدعي user probe ثم يضيف sysfs groups |
| `faux_remove()` | internal bus callback | يشيل sysfs groups ثم يستدعي user remove |
| `faux_device_release()` | internal release | يحرر الـ `faux_object` من الـ heap |

---

### المجموعة الأولى: Infrastructure Initialization

#### الغرض
الـ faux bus مش bus حقيقية — هي abstraction خفيفة فوق الـ driver core عشان تديّ devices وهمية بدون أي hardware resources. الـ `faux_bus_init()` تبني الـ infrastructure الكاملة مرة واحدة عند الـ boot.

---

#### `faux_bus_init`

```c
int __init faux_bus_init(void)
```

**بتعمل إيه:**
بتنشئ الـ root device اللي هتتعلق تحيته كل faux devices في sysfs، بعدين بتسجل الـ `faux_bus_type`، وفي الآخر بتسجل الـ `faux_driver` الوحيد اللي بيـmatch كل devices على الـ bus دي.

**Parameters:** لا يوجد.

**Return value:** `0` لو نجح، error code سالب لو فشل.

**Key details:**
- مُعلَّم بـ `__init` — بيتنفذ مرة واحدة بس أثناء kernel initialization ثم بتُرمى الـ code section
- الـ `faux_driver` عنده `probe_type = PROBE_FORCE_SYNCHRONOUS` — يعني الـ probe بيحصل sync في نفس السياق مش كـ async work
- `suppress_bind_attrs = true` — بيمنع ظهور `/sys/.../bind` و`/sys/.../unbind` في sysfs عشان المستخدم ميقدرش يـbind/unbind يدوياً
- Error path بيعمل unwind بالترتيب العكسي: لو `driver_register` فشل → `bus_unregister` → `device_unregister`

**Pseudocode flow:**
```
faux_bus_root = kzalloc()
device_register(faux_bus_root)        → يظهر في sysfs كـ /sys/devices/faux
bus_register(&faux_bus_type)          → يظهر في sysfs كـ /sys/bus/faux
driver_register(&faux_driver)         → يظهر في sysfs كـ /sys/bus/faux/drivers/faux_driver
```

---

### المجموعة التانية: Device Creation (Public API)

#### الغرض
دي الـ API الوحيدة اللي المستخدم بيشوفها. بتبسّط كل الـ device model boilerplate في call واحدة أو اتنين.

---

#### `faux_device_create`

```c
struct faux_device *faux_device_create(const char *name,
                                       struct device *parent,
                                       const struct faux_device_ops *faux_ops);
```

**بتعمل إيه:**
wrapper بسيط على `faux_device_create_with_groups()` بيبعت `NULL` كـ groups. الـ use case الأساسي لما مش محتاج تضيف custom sysfs attributes.

**Parameters:**
- `name` — اسم الـ device، لازم يكون unique بين كل faux devices. هيظهر كـ directory في sysfs
- `parent` — الـ parent device في الـ device tree. لو `NULL` بيتحط تحت الـ `faux_bus_root` مباشرة
- `faux_ops` — pointer لـ callbacks table (probe/remove)، ممكن يكون `NULL` لو مش محتاج

**Return value:**
- Pointer لـ `struct faux_device` مسجل ومتأكد إن الـ probe نجح
- `NULL` لو فيه أي error (allocation فشل، probe فشل، `device_add` فشل)

**Key details:**
- الـ probe callback ممكن يتنفذ قبل ما الـ function ترجع (بسبب `PROBE_FORCE_SYNCHRONOUS`)
- لو الـ probe فشل، الـ function بترجع `NULL` مش error code — simplification مقصودة
- موضوعة كـ `EXPORT_SYMBOL_GPL`

**Who calls it:** أي kernel module أو driver محتاج device وهمية بدون hardware binding.

---

#### `faux_device_create_with_groups`

```c
struct faux_device *faux_device_create_with_groups(const char *name,
                                                   struct device *parent,
                                                   const struct faux_device_ops *faux_ops,
                                                   const struct attribute_group **groups);
```

**بتعمل إيه:**
دي الـ implementation الحقيقية. بتـallocate الـ `faux_object` الداخلية، بتـinitialize الـ `struct device`، بتسجّلها في الـ driver core، وبتتحقق إن الـ probe نجح فعلاً قبل ما ترجع للـ caller.

**Parameters:**
- `name` — اسم الـ device في sysfs، لازم unique
- `parent` — parent device، أو `NULL` عشان يتحط تحت faux root
- `faux_ops` — callbacks table، ممكن `NULL`
- `groups` — NULL-terminated array من `attribute_group` pointers للـ sysfs attributes، ممكن `NULL`

**Return value:** نفس `faux_device_create` — pointer أو `NULL`.

**Key details:**
- الـ `faux_object` هو الـ container الداخلي اللي بيحتوي على `faux_device` + `faux_ops` + `groups`
- الـ `device_initialize()` بتعمل kref init — بعد كده بتقدر تعمل `put_device()` آمن
- `device_set_pm_not_required(dev)` — بيقول للـ PM framework إن الـ device دي مش محتاجة power management
- الـ sysfs groups بتتضاف في الـ `faux_probe()` داخلياً بعد ما الـ user probe ينجح — مش هنا مباشرة
- بعد `device_add()` الـ probe ممكن يحصل فوراً قبل ما نرجع من `device_add()`
- التحقق من `dev->driver` بعد `device_add()` هو الطريقة الوحيدة للتأكد إن الـ probe نجح

**Pseudocode flow:**
```
faux_obj = kzalloc(sizeof(faux_object))
faux_obj->faux_ops = faux_ops
faux_obj->groups   = groups

device_initialize(dev)
dev->release = faux_device_release   // ضمان تحرير الـ memory عند آخر put_device
dev->parent  = parent ?: faux_bus_root
dev->bus     = &faux_bus_type
dev_set_name(dev, name)
device_set_pm_not_required(dev)

ret = device_add(dev)
    → يبدأ match/probe دلوقتي بسبب PROBE_FORCE_SYNCHRONOUS
    if ret != 0:
        put_device(dev)   // يستدعي faux_device_release → kfree
        return NULL

if dev->driver == NULL:             // probe فشل
    faux_device_destroy(faux_dev)
    return NULL

return faux_dev                     // نجاح
```

---

### المجموعة التالتة: Device Destruction (Public API)

---

#### `faux_device_destroy`

```c
void faux_device_destroy(struct faux_device *faux_dev);
```

**بتعمل إيه:**
بتشيل الـ device من الـ driver core وبتحرر الـ memory. الـ `device_del()` بيفصل الـ driver (فبيستدعي `faux_remove()` → user remove callback) وبيشيلها من sysfs. الـ `put_device()` بتنقص الـ refcount وبتستدعي `faux_device_release()` لو وصل لصفر.

**Parameters:**
- `faux_dev` — pointer للـ device المراد تدميره، لو `NULL` بيرجع فوراً (null check موجود)

**Return value:** لا يوجد (void).

**Key details:**
- الـ `device_del()` بيستدعي الـ `faux_remove()` اللي هو:
  1. بيشيل الـ sysfs groups بـ `device_remove_groups()`
  2. بيستدعي `faux_ops->remove()` لو موجود
- الـ memory بتتحرر عند `put_device()` مش عند `device_del()` — لأن ممكن يكون في `get_device()` من حتة تانية
- آمن للاستدعاء من أي context عادي (process context)
- موضوعة كـ `EXPORT_SYMBOL_GPL`

**Who calls it:** الـ caller اللي استدعى `faux_device_create()` أو لما الـ probe يفشل من داخل `faux_device_create_with_groups()`.

---

### المجموعة الرابعة: Driver Data Helpers (Inline)

#### الغرض
wrapper functions بسيطة على `dev_get/set_drvdata()` عشان الـ caller يتعامل مع `struct faux_device *` مباشرة من غير ما يحتاج يـaccess الـ `dev` field يدوياً.

---

#### `faux_device_get_drvdata`

```c
static inline void *faux_device_get_drvdata(const struct faux_device *faux_dev)
{
    return dev_get_drvdata(&faux_dev->dev);
}
```

**بتعمل إيه:**
بترجع الـ private data pointer اللي اتخزن في الـ device بـ `dev->driver_data`.

**Parameters:**
- `faux_dev` — الـ faux device المراد جلب البيانات منه

**Return value:** الـ `void *` pointer اللي اتحط بـ `faux_device_set_drvdata()`، أو `NULL` لو لم يُعيَّن.

**Key details:** inline — zero overhead في الـ runtime. الـ `dev_get_drvdata()` نفسها بترجع `dev->driver_data` مباشرة.

---

#### `faux_device_set_drvdata`

```c
static inline void faux_device_set_drvdata(struct faux_device *faux_dev, void *data)
{
    dev_set_drvdata(&faux_dev->dev, data);
}
```

**بتعمل إيه:**
بتخزن private data pointer في `dev->driver_data`، الـ pattern المعتاد هو إن الـ probe callback بيعمل `kzalloc()` لـ private struct ثم بيحطه هنا.

**Parameters:**
- `faux_dev` — الـ faux device
- `data` — أي pointer، عادةً private struct مخصص للـ driver

**Return value:** لا يوجد (void).

**Key details:** لا يوجد locking داخلي — الـ caller مسؤول إن الـ set يحصل قبل ما أي thread تاني يستخدم الـ get.

---

### المجموعة الخامسة: Internal Bus Callbacks

#### الغرض
دي callbacks بتتسجل في `faux_bus_type` — الـ driver core بيستدعيها تلقائياً، المستخدم الخارجي ما بيستدعيهاش.

---

#### `faux_match`

```c
static int faux_match(struct device *dev, const struct device_driver *drv)
{
    return 1; /* Match always succeeds */
}
```

**بتعمل إيه:**
دايماً بترجع 1 (match ناجح). لأن الـ faux bus عندها driver واحد بس (`faux_driver`) وكل device على الـ bus المفروض تتـbind بيه.

**Key details:** بساطة مقصودة — مفيش ID table، مفيش OF matching، مفيش ACPI.

---

#### `faux_probe`

```c
static int faux_probe(struct device *dev)
```

**بتعمل إيه:**
الـ driver core بيستدعيها لما device جديدة تتسجل وتـmatch الـ driver. بتستدعي الـ user probe أولاً، لو نجح بتضيف الـ sysfs groups.

**Key details:**
- لو `faux_ops->probe()` فشل، بتوقف وبترجع error — groups ما بتتضافش
- لو `device_add_groups()` فشل بعد ما الـ probe نجح، بيستدعي `faux_ops->remove()` عشان يعمل cleanup للـ user resources
- الترتيب مهم جداً: probe → groups (مش العكس)

---

#### `faux_remove`

```c
static void faux_remove(struct device *dev)
```

**بتعمل إيه:**
الـ driver core بيستدعيها لما device تتفصل من الـ driver (عند الـ destroy). بتشيل الـ sysfs groups الأول ثم بتستدعي الـ user remove.

**Key details:** عكس الـ probe — groups بتتشال أول قبل ما نستدعي الـ user remove عشان نضمن إن الـ sysfs ما بيستخدمش resources اتحررت في الـ remove.

---

#### `faux_device_release`

```c
static void faux_device_release(struct device *dev)
{
    struct faux_object *faux_obj = to_faux_object(dev);
    kfree(faux_obj);
}
```

**بتعمل إيه:**
بتتسجل في `dev->release` — الـ driver core بيستدعيها تلقائياً لما الـ refcount بيوصل لصفر. بتحرر الـ `faux_object` الكاملة من الـ heap.

**Key details:**
- الـ `faux_object` بيحتوي على `faux_device` اللي بيحتوي على `struct device` — الـ kfree بيحرر كل الـ container بعملية واحدة
- لو الـ `dev->release` مش متعين والـ refcount وصل لصفر، الـ kernel بيطبع warning — الـ release دي ضرورية

---

### المجموعة السادسة: Macros / Container Access

---

#### `to_faux_device`

```c
#define to_faux_device(x) container_of_const((x), struct faux_device, dev)
```

**بتعمل إيه:**
بتحول `struct device *` لـ `struct faux_device *` بالـ container_of idiom المعتاد في الـ kernel. بتستخدم `container_of_const` عشان تحترم الـ `const` qualifier لو الـ input كان `const`.

**Parameters:**
- `x` — pointer لـ `struct device`، ممكن يكون `const`

**Usage pattern:**
```c
/* في sysfs show function مثلاً */
static ssize_t my_attr_show(struct device *d, ...)
{
    struct faux_device *fdev = to_faux_device(d);
    struct my_private *priv = faux_device_get_drvdata(fdev);
    ...
}
```

---

### ملاحظة على الـ struct faux_object

الـ `struct faux_object` مش جزء من الـ public API — هي implementation detail داخلية:

```c
struct faux_object {
    struct faux_device faux_dev;        /* الـ public-facing device */
    const struct faux_device_ops *faux_ops;  /* user callbacks */
    const struct attribute_group **groups;   /* sysfs groups */
};
```

الـ design هنا classical kernel pattern: الـ `faux_device` هو "الوجه" اللي المستخدم بيشوفه، و`faux_object` هو الـ "body" الداخلي اللي بيحتوي على كل الـ state. الـ `to_faux_object()` macro (internal فقط) بيعمل الـ downcast من `struct device` للـ container الكامل.

---

### تدفق دورة حياة الـ Device — شامل

```
 فراغ
   │
   ▼
faux_device_create(name, parent, ops)
   │
   ├─► kzalloc(faux_object)
   ├─► device_initialize()
   ├─► dev->release = faux_device_release
   ├─► dev->bus = &faux_bus_type
   ├─► device_add(dev)
   │       │
   │       └─► faux_match() → 1  (always match)
   │       └─► faux_probe()
   │               ├─► faux_ops->probe()   [user code]
   │               └─► device_add_groups()
   │
   ├─► dev->driver != NULL ?
   │       ├─ NO  → faux_device_destroy() → return NULL
   │       └─ YES → return faux_dev       [نجاح]
   │
  ...استخدام الـ device...
   │
   ▼
faux_device_destroy(faux_dev)
   │
   ├─► device_del(dev)
   │       └─► faux_remove()
   │               ├─► device_remove_groups()
   │               └─► faux_ops->remove()  [user code]
   │
   └─► put_device(dev)
           └─► [refcount == 0] → faux_device_release() → kfree(faux_obj)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ faux bus نفسه ما بيستخدمش debugfs بشكل مباشر، لكن الـ driver core بيوفر معلومات مفيدة:

```bash
# mount debugfs لو مش موجودة
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ devices المسجلة في الـ driver core
ls /sys/kernel/debug/devices/

# تفاصيل الـ device links
cat /sys/kernel/debug/devices/*/consumers 2>/dev/null
cat /sys/kernel/debug/devices/*/suppliers 2>/dev/null
```

لو الـ kernel اتبنى بـ `CONFIG_DEBUG_DRIVER`:

```bash
# تفعيل debug output للـ driver core كله
echo 1 > /sys/kernel/debug/driver_core
```

---

#### 2. مدخلات الـ sysfs

الـ faux bus بيظهر في sysfs تحت مسارين أساسيين:

```
/sys/bus/faux/                     ← الـ bus نفسه
/sys/bus/faux/devices/             ← كل الـ faux devices المسجلة
/sys/bus/faux/drivers/             ← الـ faux_driver الوحيد
/sys/bus/faux/drivers/faux_driver/ ← الـ driver المرتبط بكل الـ devices
/sys/devices/faux/                 ← الـ root device للـ faux bus
/sys/devices/faux/<name>/          ← كل device باسمها
```

**الـ attributes المهمة لكل device:**

```bash
# اسم الـ device
cat /sys/bus/faux/devices/<name>/uevent

# هل الـ driver اتـ bind؟
ls /sys/bus/faux/devices/<name>/driver

# الـ parent
ls -la /sys/bus/faux/devices/<name>/../

# الـ power state
cat /sys/bus/faux/devices/<name>/power/runtime_status

# الـ custom sysfs groups اللي أضافها الـ driver
ls /sys/bus/faux/devices/<name>/
```

**أوامر جاهزة:**

```bash
# اعرض كل الـ faux devices الموجودة
ls /sys/bus/faux/devices/

# تحقق من الـ bind لكل device
for d in /sys/bus/faux/devices/*; do
    echo -n "$d: driver="
    ls "$d/driver" 2>/dev/null || echo "UNBOUND"
done

# اقرأ uevent لـ device معينة
cat /sys/bus/faux/devices/my_device/uevent
```

**مثال output:**

```
MAJOR=0
MINOR=0
DEVNAME=
DEVTYPE=
MODALIAS=faux:my_device
```

---

#### 3. الـ ftrace — tracepoints وأحداث مهمة

الـ faux subsystem ما بيضيفش tracepoints خاصة بيه، لكن الـ driver core بيوفر أحداث كافية:

```bash
# اعرض الـ events المتاحة للـ device core
ls /sys/kernel/tracing/events/devlink/
ls /sys/kernel/tracing/events/module/

# تفعيل تتبع device_add / device_del
echo 1 > /sys/kernel/tracing/events/enable

# تتبع الـ probe فقط — استخدم function tracer
echo function > /sys/kernel/tracing/current_tracer
echo faux_probe > /sys/kernel/tracing/set_ftrace_filter
echo faux_remove >> /sys/kernel/tracing/set_ftrace_filter
echo faux_device_create_with_groups >> /sys/kernel/tracing/set_ftrace_filter
echo faux_device_destroy >> /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل الكود اللي عايز تتبعه، وبعدين
cat /sys/kernel/tracing/trace
echo 0 > /sys/kernel/tracing/tracing_on
```

**تتبع الـ device registration بالكامل:**

```bash
echo function_graph > /sys/kernel/tracing/current_tracer
echo faux_device_create_with_groups > /sys/kernel/tracing/set_graph_function
echo 1 > /sys/kernel/tracing/tracing_on
# ... تشغيل الكود ...
cat /sys/kernel/tracing/trace | head -100
```

**مثال output للـ function_graph:**

```
 1)               |  faux_device_create_with_groups() {
 1)               |    kzalloc() { ... }
 1)               |    device_initialize() { ... }
 1)               |    device_add() {
 1)               |      faux_probe() {
 1)   0.312 us    |        my_driver_probe();
 1)               |      }
 1)               |    }
 1)   5.100 us    |  }
```

---

#### 4. الـ printk والـ Dynamic Debug

**تفعيل debug messages للـ driver core:**

```bash
# Dynamic debug — تفعيل كل الـ dev_dbg في drivers/base/faux.c
echo 'file drivers/base/faux.c +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق من التفعيل
grep 'faux' /sys/kernel/debug/dynamic_debug/control
```

**مثال output بعد التفعيل:**

```
drivers/base/faux.c:172 [faux]faux_device_create_with_groups =p "probe did not succeed, tearing down the device\n"
```

**تفعيل debug للـ driver core كله:**

```bash
echo 'module drivers_base +p' > /sys/kernel/debug/dynamic_debug/control
# أو
echo 'file drivers/base/*.c +p' > /sys/kernel/debug/dynamic_debug/control
```

**لمستوى printk الـ kernel:**

```bash
# اضبط loglevel عشان تشوف KERN_DEBUG
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. خيارات الـ Kconfig للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_DRIVER` | يفعّل dev_dbg() في الـ driver core |
| `CONFIG_DEBUG_DEVRES` | يتتبع الـ devres allocations ويكشف leaks |
| `CONFIG_DEBUG_KOBJECT` | يطبع كل kobject create/destroy |
| `CONFIG_DEBUG_KOBJECT_RELEASE` | يكشف use-after-free في الـ kobject release |
| `CONFIG_PROVE_LOCKING` | lockdep — يكشف deadlocks في الـ device lock |
| `CONFIG_KASAN` | يكشف memory corruption في الـ faux_object |
| `CONFIG_KMSAN` | يكشف use of uninitialized memory |
| `CONFIG_KCSAN` | يكشف data races في الـ concurrent probe/remove |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بتفعيل dev_dbg بدون إعادة compile |
| `CONFIG_FTRACE` | أساسي للـ function tracing |
| `CONFIG_FAULT_INJECTION` | يسمح بـ inject errors في kzalloc مثلاً |

**تفعيل fault injection عشان تختبر error paths:**

```bash
# inject kzalloc failure لاختبار الـ NULL return
echo 1 > /sys/kernel/debug/failslab/task-filter
echo 1 > /proc/self/make-it-fail
```

---

#### 6. أدوات خاصة بالـ Subsystem

الـ faux bus بسيط جداً وما بيستخدمش devlink، لكن في أدوات الـ driver core العامة:

```bash
# udevadm — مراقبة أحداث الـ uevent الخاصة بـ faux devices
udevadm monitor --subsystem-match=faux

# مثال output:
# KERNEL[1234.567] add      /devices/faux/my_device (faux)
# KERNEL[1234.890] remove   /devices/faux/my_device (faux)

# udevadm info — تفاصيل device معينة
udevadm info -a -p /sys/bus/faux/devices/my_device

# فحص الـ device hierarchy
udevadm info --tree /sys/bus/faux/

# lsdev / lshw بدائل
find /sys/bus/faux/devices/ -maxdepth 1 -mindepth 1 -type l | xargs -I{} readlink -f {}
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `faux_device_create_with_groups: device_add for faux device 'X' failed with -EEXIST` | اسم الـ device مكرر | استخدم اسم فريد، افحص بـ `ls /sys/bus/faux/devices/` |
| `faux_device_create_with_groups: device_add for faux device 'X' failed with -ENOMEM` | الـ kzalloc فشل | افحص memory pressure، راجع `/proc/meminfo` |
| `probe did not succeed, tearing down the device` | الـ probe callback رجع error أو الـ driver ما اتـ bind | افحص الـ probe return value، فعّل dynamic debug لـ faux.c |
| `faux_probe: device_add_groups failed` | الـ sysfs group ما اتضافتش | افحص الـ attribute_group structure، تأكد إن الـ attrs مش NULL |
| `WARNING: ... in faux_device_destroy` | الـ device بتتدمر وهي still in use | تأكد إن ما فيش reference باقية قبل `faux_device_destroy()` |
| `kobject: 'X' ... already registered` | نفس مشكلة الاسم المكرر على مستوى الـ kobject | نفس الحل — فريد الاسم |
| `Call Trace: faux_remove+0x...` | crash في الـ remove callback | افحص الـ driver data بـ `faux_device_get_drvdata()` قبل استخدامه |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في faux_probe — تحقق إن faux_dev مش NULL */
static int faux_probe(struct device *dev)
{
    struct faux_object *faux_obj = to_faux_object(dev);

    WARN_ON(!faux_obj);  /* لو الـ container_of أرجع حاجة غلط */

    if (faux_obj->faux_ops && faux_obj->faux_ops->probe) {
        int ret = faux_obj->faux_ops->probe(&faux_obj->faux_dev);
        WARN_ON(ret > 0);  /* probe لازم يرجع 0 أو negative */
        if (ret)
            return ret;
    }
    /* ... */
}

/* في driver يستخدم faux — تحقق من الـ drvdata */
static int my_probe(struct faux_device *faux_dev)
{
    struct my_priv *priv = kzalloc(sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    faux_device_set_drvdata(faux_dev, priv);
    return 0;
}

static void my_remove(struct faux_device *faux_dev)
{
    struct my_priv *priv = faux_device_get_drvdata(faux_dev);

    /* نقطة استراتيجية — لو priv NULL معناه في bug في الـ probe */
    if (WARN_ON(!priv))
        return;

    /* cleanup ... */
    kfree(priv);
}
```

**نقاط يستحق فيها `dump_stack()`:**

```c
/* لو شكّ إن الـ destroy بيتكلم من context غلط */
void faux_device_destroy(struct faux_device *faux_dev)
{
    if (!faux_dev) {
        dump_stack();  /* مين بعت NULL؟ */
        return;
    }
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقابل الـ Kernel State

الـ faux device **بالتعريف** ما بيمثلش hardware حقيقي — هو virtual device. فالتحقق بيكون على مستوى الـ software state فقط:

```bash
# تحقق إن الـ device موجودة في sysfs
test -d /sys/bus/faux/devices/my_device && echo "EXISTS" || echo "NOT FOUND"

# تحقق إن الـ driver اتـ bind (probe نجح)
test -L /sys/bus/faux/devices/my_device/driver && echo "BOUND" || echo "UNBOUND"

# اقرأ الـ drvdata بشكل غير مباشر عبر custom sysfs attrs
cat /sys/bus/faux/devices/my_device/my_custom_attr 2>/dev/null
```

لو الـ faux device بتتحكم في hardware فعلي (مثلاً firmware loader يتكلم مع GPU):

```bash
# افحص الـ parent device اللي الـ faux device ماشي تحته
readlink /sys/bus/faux/devices/my_device/../..

# افحص الـ firmware files المرتبطة
ls /lib/firmware/
dmesg | grep "firmware"
```

---

#### 2. Register Dump Techniques

الـ faux device نفسه ما بيملكش registers، لكن لو الـ driver اللي بيستخدمه بيتعامل مع MMIO:

```bash
# devmem2 — قراءة register مباشرة من userspace
# (محتاج root + CONFIG_STRICT_DEVMEM=n أو /dev/mem access)
devmem2 0xFEDC0000 w    # قرا word من عنوان معين
devmem2 0xFEDC0004 b    # قرا byte

# بديل بـ /dev/mem مباشرة
dd if=/dev/mem bs=4 count=1 skip=$((0xFEDC0000/4)) 2>/dev/null | xxd

# لو الـ driver expose registers عبر debugfs
cat /sys/kernel/debug/my_faux_driver/registers
```

لو الـ subsystem بيحتاج ioport:

```bash
# io utility (من package ioport)
io -4 -r 0x3F8     # قرا 4 bytes من IO port
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ faux bus نفسه ما بيحتاجش logic analyzer لأنه software-only. لكن لو الـ faux device يتحكم في interface حقيقية (I2C, SPI, UART):

- **I2C**: ابحث عن الـ ACK/NACK على SDA بعد كل byte — NACK في الـ address يعني device مش موجود فعلياً
- **SPI**: تأكد إن الـ CS يتـ assert قبل الـ SCLK بـ setup time كافي
- **GPIO**: لو الـ faux device بيـ toggle GPIO كـ debug signal — استخدم:

```bash
# toggle GPIO يدوياً للـ sync مع oscilloscope
echo 1 > /sys/class/gpio/gpio50/value
# ... الـ code المشبوه ...
echo 0 > /sys/class/gpio/gpio50/value
```

**نقطة مهمة**: لو الـ probe بيأخذ وقت طويل ظاهر على الـ scope — ابحث عن `PROBE_FORCE_SYNCHRONOUS` في الـ `faux_driver` definition؛ ده طبيعي ومتعمد.

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ dmesg | ملاحظة |
|---|---|---|
| الـ parent device مش موجود | `device_add: parent '/devices/...' not registered` | الـ parent لازم يتسجل قبل الـ faux device |
| memory pressure في وقت الـ init | `faux_device_create_with_groups: device_add for ... failed with -ENOMEM` | OOM killer active |
| firmware مش موجود | `firmware: failed to load my_fw.bin` | الـ firmware لازم موجود في `/lib/firmware/` |
| race condition في الـ probe/remove | `WARNING: ... lock held when returning to user space` | استخدم KCSAN لتكشفها |
| double destroy | `kernel BUG at lib/list_debug.c` | الـ faux_device_destroy اتكلم أكثر من مرة |

---

#### 5. Device Tree Debugging

الـ faux bus **لا يستخدم Device Tree** — هو مصمم تحديداً للـ devices اللي ما عندهاش physical resources أو DT nodes. لو لاقيت نفسك محتاج DT مع faux → استخدم `platform_device` بدلاً منه.

**للتحقق إن الـ faux device مش مرتبط بـ DT عن طريق الخطأ:**

```bash
# الـ of_node لازم يكون NULL
cat /sys/bus/faux/devices/my_device/of_node 2>/dev/null
# لو الأمر ده أرجع حاجة — في مشكلة

# تحقق من firmware node
ls /sys/bus/faux/devices/my_device/firmware_node 2>/dev/null
```

---

### Practical Commands — كل الأوامر جاهزة للـ Copy

#### فحص سريع شامل للـ faux subsystem

```bash
#!/bin/bash
# faux_debug.sh — فحص شامل للـ faux bus

echo "=== Faux Bus Status ==="
echo ""

echo "[1] Faux Bus Present:"
test -d /sys/bus/faux && echo "YES" || echo "NO — kernel may not have faux support"
echo ""

echo "[2] Faux Root Device:"
test -d /sys/devices/faux && ls /sys/devices/faux/ || echo "Not found"
echo ""

echo "[3] Registered Faux Devices:"
ls /sys/bus/faux/devices/ 2>/dev/null || echo "None"
echo ""

echo "[4] Driver Binding Status:"
for d in /sys/bus/faux/devices/*/; do
    name=$(basename "$d")
    if [ -L "$d/driver" ]; then
        drv=$(basename $(readlink "$d/driver"))
        echo "  $name → BOUND to $drv"
    else
        echo "  $name → UNBOUND (probe failed?)"
    fi
done
echo ""

echo "[5] Faux Driver Info:"
ls /sys/bus/faux/drivers/ 2>/dev/null
echo ""

echo "[6] Recent Kernel Messages:"
dmesg | grep -i 'faux' | tail -20
```

#### تفعيل الـ Dynamic Debug

```bash
# تفعيل كل messages في faux.c
echo 'file drivers/base/faux.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# التحقق
grep faux /sys/kernel/debug/dynamic_debug/control

# إيقاف التفعيل
echo 'file drivers/base/faux.c -p' > /sys/kernel/debug/dynamic_debug/control
```

#### مراقبة الـ uevents في الوقت الفعلي

```bash
# في terminal منفصل
udevadm monitor --kernel --subsystem-match=faux

# ثم في terminal تاني — شغّل الـ module اللي بيعمل faux device
modprobe my_faux_driver

# output متوقع:
# KERNEL[...] add    /devices/faux/my_device (faux)
```

#### ftrace جاهز للـ Copy

```bash
#!/bin/bash
# trace_faux.sh

TRACEFS=/sys/kernel/tracing

# reset
echo nop > $TRACEFS/current_tracer
echo > $TRACEFS/trace
echo > $TRACEFS/set_ftrace_filter

# ضبط الـ tracer
echo function_graph > $TRACEFS/current_tracer
cat >> $TRACEFS/set_graph_function << 'EOF'
faux_device_create_with_groups
faux_device_create
faux_device_destroy
faux_probe
faux_remove
EOF

# ابدأ التسجيل
echo 1 > $TRACEFS/tracing_on
echo "Tracing started. Run your test now..."
read -p "Press Enter when done..."
echo 0 > $TRACEFS/tracing_on

# احفظ النتيجة
cp $TRACEFS/trace /tmp/faux_trace_$(date +%s).txt
echo "Trace saved to /tmp/faux_trace_*.txt"
cat $TRACEFS/trace | head -50
```

#### تشخيص فشل الـ probe

```bash
# الخطوات بالترتيب لو faux_device_create رجعت NULL

# 1. افحص الـ kernel log
dmesg | grep -E 'faux|probe|device_add' | tail -30

# 2. فعّل dynamic debug وجرب تاني
echo 'file drivers/base/faux.c +p' > /sys/kernel/debug/dynamic_debug/control
# أعد تشغيل الـ module
# دمج | grep faux

# 3. لو الاسم مكرر
ls /sys/bus/faux/devices/ | grep my_device

# 4. لو الـ parent مش مسجل
udevadm info -p /sys/devices/platform/my_parent 2>/dev/null || echo "parent not in sysfs"

# 5. فحص memory
grep -i 'oom\|out of memory\|alloc fail' /var/log/kern.log | tail -10
```

#### قراءة sysfs attributes بالكامل لـ device معينة

```bash
DEVICE="my_device"
DEVPATH="/sys/bus/faux/devices/$DEVICE"

echo "=== $DEVICE sysfs dump ==="
find "$DEVPATH" -maxdepth 2 -type f 2>/dev/null | while read f; do
    echo "--- $f ---"
    cat "$f" 2>/dev/null
done
```

#### اختبار الـ error path بـ fault injection

```bash
# تأكد إن CONFIG_FAULT_INJECTION=y في الـ kernel
ls /sys/kernel/debug/failslab/ 2>/dev/null || echo "Fault injection not available"

# inject kzalloc failure بنسبة 10%
echo 10 > /sys/kernel/debug/failslab/probability
echo 1 > /sys/kernel/debug/failslab/verbose
echo Y > /sys/kernel/debug/failslab/task-filter

# الآن أي kzalloc هيفشل بنسبة 10% — اختبر إن faux_device_create بترجع NULL بشكل صحيح
insmod my_faux_driver.ko
dmesg | grep -i 'faux\|alloc\|nomem' | tail -10

# أوقف fault injection
echo 0 > /sys/kernel/debug/failslab/probability
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Firmware Loader على Industrial Gateway بـ AM62x

#### العنوان
درايفر تحميل firmware لـ DSP على gateway صناعي — probe بيفشل بصمت

#### السياق
شركة بتطور industrial gateway على TI AM62x. الـ gateway بيستخدم DSP core داخلي لمعالجة بروتوكول Modbus RTU. المطور عمل درايفر خاص بيحمّل firmware للـ DSP عبر remoteproc API. قرر يستخدم `faux_device` لأن الـ DSP مش مربوط بأي bus حقيقي من ناحية الـ driver model.

#### المشكلة
بعد تشغيل الكرنل، الـ `/sys/bus/faux/devices/` فارغ والـ DSP مش شغال. مفيش error واضح في dmesg.

```bash
$ ls /sys/bus/faux/devices/
# فاضي تماماً

$ dmesg | grep dsp
# لا شيء
```

#### التحليل
المطور بص في الكود وشاف إنه بيعمل:

```c
static int dsp_probe(struct faux_device *faux_dev)
{
    struct dsp_priv *priv;

    priv = devm_kzalloc(&faux_dev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* firmware load fails silently here */
    if (load_dsp_firmware(priv) != 0)
        return -ENODEV;   /* ← بيرجع error هنا */

    faux_device_set_drvdata(faux_dev, priv);
    return 0;
}

static const struct faux_device_ops dsp_faux_ops = {
    .probe  = dsp_probe,
    .remove = dsp_remove,
};
```

الـ `faux_device_create()` بيستدعي داخلياً `device_add()` → `faux_probe()` → `faux_ops->probe()`. لو الـ probe رجع error، الكود في `faux.c` بيعمل التالي:

```c
/* من faux.c */
if (!dev->driver) {
    dev_dbg(dev, "probe did not succeed, tearing down the device\n");
    faux_device_destroy(faux_dev);
    faux_dev = NULL;
}
return faux_dev;  /* بيرجع NULL */
```

المشكلة: `dev_dbg` مش بتظهر إلا لو `CONFIG_DYNAMIC_DEBUG` فعّال أو الـ loglevel عالٍ. والكود اللي بيستدعي `faux_device_create()` مش بيتحقق من الـ NULL return.

```c
/* الكود الغلط في module init */
static int __init dsp_module_init(void)
{
    g_faux_dev = faux_device_create("am62x-dsp", NULL, &dsp_faux_ops);
    /* ← مفيش check على NULL! */
    return 0;  /* بيرجع success حتى لو فشل */
}
```

#### الحل

**أولاً:** تشغيل dynamic debug لمشاهدة `dev_dbg`:

```bash
$ echo "file drivers/base/faux.c +p" > /sys/kernel/debug/dynamic_debug/control
$ dmesg | grep faux
# [  2.341] faux am62x-dsp: probe did not succeed, tearing down the device
```

**ثانياً:** إصلاح الكود — التحقق من الـ return value:

```c
static int __init dsp_module_init(void)
{
    g_faux_dev = faux_device_create("am62x-dsp", NULL, &dsp_faux_ops);
    if (!g_faux_dev) {
        pr_err("dsp: faux_device_create failed\n");
        return -ENODEV;
    }
    return 0;
}
```

**ثالثاً:** إصلاح سبب الفشل الحقيقي — الـ firmware مش موجود:

```bash
$ ls /lib/firmware/am62x-dsp.bin
# ls: cannot access: No such file or directory

$ cp dsp_fw.bin /lib/firmware/am62x-dsp.bin
$ modprobe am62x_dsp_drv
$ ls /sys/bus/faux/devices/
am62x-dsp
```

#### الدرس المستفاد
**الـ** `faux_device_create()` **بترجع** `NULL` **لو الـ probe فشل — مش بترجع error code.** لازم دايماً تتحقق من الـ return value. والـ `dev_dbg` في `faux.c` بتتعامى لو الـ dynamic debug مش مفعّل، خليها في حسبانك وقت الـ bring-up.

---

### السيناريو 2: Android TV Box على Allwinner H616 — name collision بيخلي الجهاز ميبدأش

#### العنوان
تعارض في اسم الـ faux device بين درايفر HDMI audio و HDMI video على Allwinner H616

#### السياق
منتج Android TV Box بيستخدم Allwinner H616. التيم عنده درايفران مستقلان: واحد للـ HDMI display engine وواحد للـ HDMI audio codec. كل درايفر قرر يعمل `faux_device` خاص بيه لتسجيل state في sysfs.

#### المشكلة
الجهاز بيبوت طبيعي أحياناً وأحياناً تانية الـ HDMI audio مش شغال.

```bash
$ dmesg | grep -i faux
[  3.112] faux: device_add for faux device 'h616-hdmi' failed with -17
[  3.113] h616_hdmi_audio: faux_device_create failed
```

الـ error code `-17` هو `EEXIST` — اسم متكرر.

#### التحليل
الـ `faux_bus_type` فيه `faux_match` بترجع دايماً `1`، يعني كل device على الـ faux bus بيتطابق مع الـ faux_driver الوحيد. لكن تمييز الـ devices بيكون بالاسم بس. الكود في `faux_device_create_with_groups()`:

```c
dev_set_name(dev, "%s", name);  /* الاسم هو الـ unique identifier */

ret = device_add(dev);
if (ret) {
    pr_err("%s: device_add for faux device '%s' failed with %d\n",
           __func__, name, ret);
    put_device(dev);
    return NULL;
}
```

`device_add()` بترفض أي device باسم موجود على نفس الـ bus. الدرايفران الاتنين بيستخدموا نفس الاسم:

```c
/* في hdmi_display_drv.c */
faux_dev = faux_device_create("h616-hdmi", NULL, &display_ops);

/* في hdmi_audio_drv.c */
faux_dev = faux_device_create("h616-hdmi", NULL, &audio_ops);  /* ← نفس الاسم! */
```

والمشكلة بتحصل بشكل متقطع لأن الـ module loading order مش ثابت — أيهم يسجّل أول بياخد الاسم والتاني بيفشل.

#### الحل

**إصلاح فوري** — تغيير الأسماء لتكون unique:

```c
/* في hdmi_display_drv.c */
faux_dev = faux_device_create("h616-hdmi-display", NULL, &display_ops);

/* في hdmi_audio_drv.c */
faux_dev = faux_device_create("h616-hdmi-audio", NULL, &audio_ops);
```

**التحقق بعد الإصلاح:**

```bash
$ ls /sys/bus/faux/devices/
h616-hdmi-audio  h616-hdmi-display

$ cat /sys/bus/faux/devices/h616-hdmi-audio/uevent
DEVTYPE=faux_device
```

**لو محتاج parent-child relationship** (مثلاً audio تحت display):

```c
/* audio بتكون child من display */
struct faux_device *display_faux = faux_device_create("h616-hdmi-display", NULL, &display_ops);
struct faux_device *audio_faux  = faux_device_create("h616-hdmi-audio",
                                                      &display_faux->dev,  /* parent */
                                                      &audio_ops);
```

```bash
$ ls /sys/bus/faux/devices/h616-hdmi-display/
h616-hdmi-audio/   power/   uevent
```

#### الدرس المستفاد
**الـ** `name` **في** `faux_device_create()` **لازم يكون globally unique على الـ faux bus.** مش كفاية يكون unique داخل الموديول — أي درايفر تاني على نفس الكرنل ممكن يتعارض معاك. استخدم اسم descriptive وmeaningful بيتضمن اسم الـ SoC والـ subsystem.

---

### السيناريو 3: IoT Sensor Hub على STM32MP1 — memory leak بسبب فهم غلط لـ lifecycle

#### العنوان
memory leak في درايفر sensor hub بسبب استخدام `kmalloc` بدل `devm_kzalloc` مع الـ faux device

#### السياق
جهاز IoT sensor hub مبني على STM32MP1. الكرنل بيشتغل على Cortex-A7. الجهاز بيقرأ من 8 sensors عبر I2C وبيجمعهم في abstraction layer. المطور استخدم `faux_device` كـ virtual device للـ sensor aggregator لأنه مش مرتبط بـ bus فيزيائي.

#### المشكلة
بعد تشغيل لفترة طويلة، الـ `/proc/meminfo` بيُظهر انخفاض مستمر في `MemFree`. كمان عند `rmmod` وإعادة `insmod` الموديول، الـ memory بتنخفض أكتر.

```bash
$ cat /proc/meminfo | grep MemFree
MemFree:          45232 kB   # أول تشغيل

$ rmmod sensor_hub && insmod sensor_hub.ko
$ cat /proc/meminfo | grep MemFree
MemFree:          44891 kB   # نقصت بعد إعادة التحميل
```

#### التحليل
المطور كتب الكود ده:

```c
struct sensor_hub_priv {
    struct i2c_client *sensors[8];
    u8 *data_buffer;      /* 4KB buffer */
};

static int sensor_hub_probe(struct faux_device *faux_dev)
{
    struct sensor_hub_priv *priv;

    /* ← استخدم kmalloc مش devm_kzalloc */
    priv = kmalloc(sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->data_buffer = kmalloc(4096, GFP_KERNEL);
    if (!priv->data_buffer) {
        kfree(priv);
        return -ENOMEM;
    }

    faux_device_set_drvdata(faux_dev, priv);
    return 0;
}

static void sensor_hub_remove(struct faux_device *faux_dev)
{
    struct sensor_hub_priv *priv = faux_device_get_drvdata(faux_dev);

    /* ← نسي يعمل kfree للـ data_buffer! */
    kfree(priv);
}
```

لما `faux_device_destroy()` اتنادى، `faux_remove()` في `faux.c` اشتغلت:

```c
static void faux_remove(struct device *dev)
{
    struct faux_object *faux_obj = to_faux_object(dev);
    struct faux_device *faux_dev = &faux_obj->faux_dev;
    const struct faux_device_ops *faux_ops = faux_obj->faux_ops;

    device_remove_groups(dev, faux_obj->groups);

    if (faux_ops && faux_ops->remove)
        faux_ops->remove(faux_dev);  /* بتنادي sensor_hub_remove */
}
```

`sensor_hub_remove` اتنادت لكن `priv->data_buffer` ما اتعملش `kfree`.

#### الحل

**الحل الأمثل:** استخدام `devm_kzalloc` بحيث الـ kernel يتولى الـ cleanup تلقائياً:

```c
static int sensor_hub_probe(struct faux_device *faux_dev)
{
    struct sensor_hub_priv *priv;

    /* devm_kzalloc: automatically freed when device is destroyed */
    priv = devm_kzalloc(&faux_dev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    priv->data_buffer = devm_kzalloc(&faux_dev->dev, 4096, GFP_KERNEL);
    if (!priv->data_buffer)
        return -ENOMEM;   /* priv بيتحرر تلقائياً */

    faux_device_set_drvdata(faux_dev, priv);
    return 0;
}

/* remove ممكن تبقى فاضية أو NULL لو مفيش cleanup manual */
static const struct faux_device_ops sensor_hub_ops = {
    .probe  = sensor_hub_probe,
    .remove = NULL,   /* devm handles cleanup */
};
```

**التحقق بـ kmemleak:**

```bash
$ echo scan > /sys/kernel/debug/kmemleak
$ cat /sys/kernel/debug/kmemleak
unreferenced object 0xc3a12000 (size 4096):
  comm "insmod", pid 234, jiffies 4294902310
  backtrace:
    kmalloc+0x...
    sensor_hub_probe+0x...
```

#### الدرس المستفاد
الـ `faux_device` بيدعم `devm_*` allocations لأن `faux_dev->dev` هو `struct device` حقيقي. استخدم `devm_kzalloc(&faux_dev->dev, ...)` دايماً بدل `kmalloc` العادي وخلي الـ kernel يتولى الـ cleanup. الـ `faux_device_release()` في `faux.c` بيعمل `kfree` للـ `faux_object` بس مش للـ private data.

---

### السيناريو 4: Automotive ECU على i.MX8 — sysfs attributes مش بتظهر بعد probe

#### العنوان
الـ sysfs attributes الخاصة بـ ECU diagnostics مش بتظهر رغم إن الـ device اتسجّل

#### السياق
automotive ECU على NXP i.MX8QM. الـ ECU بيراقب CAN bus traffic ويعرض diagnostics. المطور استخدم `faux_device_create_with_groups()` لتسجيل device مع sysfs attributes لعرض الـ diagnostics data.

#### المشكلة
الـ device بيظهر في sysfs لكن الـ attributes مش موجودة:

```bash
$ ls /sys/bus/faux/devices/imx8-ecu-diag/
power/  uevent  # فقط — مفيش attributes المتوقعة!

$ cat /sys/bus/faux/devices/imx8-ecu-diag/can_error_count
cat: /sys/bus/faux/devices/imx8-ecu-diag/can_error_count: No such file or directory
```

#### التحليل
المطور كتب الكود ده:

```c
static DEVICE_ATTR_RO(can_error_count);
static DEVICE_ATTR_RO(can_bus_load);

static struct attribute *ecu_attrs[] = {
    &dev_attr_can_error_count.attr,
    &dev_attr_can_bus_load.attr,
    NULL,
};

static const struct attribute_group ecu_group = {
    .attrs = ecu_attrs,
};

static const struct attribute_group *ecu_groups[] = {
    &ecu_group,
    NULL,
};

static int ecu_probe(struct faux_device *faux_dev)
{
    /* ... init code ... */

    /* ← حاول يضيف الـ groups يدوياً هنا */
    return device_add_groups(&faux_dev->dev, ecu_groups);
}

static const struct faux_device_ops ecu_ops = {
    .probe  = ecu_probe,
    .remove = ecu_remove,
};

/* وبعدين استدعى */
faux_dev = faux_device_create_with_groups("imx8-ecu-diag", NULL,
                                           &ecu_ops, ecu_groups);
```

الإشكالية إن `faux_probe()` في `faux.c` بتضيف الـ groups **بعد** الـ `faux_ops->probe()`:

```c
static int faux_probe(struct device *dev)
{
    /* ... */
    if (faux_ops && faux_ops->probe) {
        ret = faux_ops->probe(faux_dev);   /* 1. بتنادي ecu_probe */
        if (ret)
            return ret;
    }

    /* 2. بعدين بتضيف groups من faux_obj->groups */
    ret = device_add_groups(dev, faux_obj->groups);
    if (ret && faux_ops && faux_ops->remove)
        faux_ops->remove(faux_dev);

    return ret;
}
```

المطور عمل `device_add_groups()` داخل `ecu_probe()` **و** بعت نفس الـ groups لـ `faux_device_create_with_groups()`. النتيجة:

1. `ecu_probe()` عملت `device_add_groups(ecu_groups)` — نجحت
2. `faux_probe()` عملت `device_add_groups(ecu_groups)` تاني — فشلت بـ `-EEXIST`
3. لأن الفشل حصل، `faux_ops->remove()` اتنادت وهي بتعمل `device_remove_groups(ecu_groups)`
4. النتيجة النهائية: الـ attributes اتحذفت!

#### الحل

**لا تضيف الـ groups يدوياً داخل probe لو بتستخدم `faux_device_create_with_groups()`:**

```c
static int ecu_probe(struct faux_device *faux_dev)
{
    struct ecu_priv *priv;

    priv = devm_kzalloc(&faux_dev->dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    /* init CAN interface */
    priv->can_dev = can_interface_init();
    if (!priv->can_dev)
        return -ENODEV;

    faux_device_set_drvdata(faux_dev, priv);
    return 0;   /* الـ groups هتتضاف تلقائياً بعد return */
}

/* الـ groups بتتبعت لـ faux_device_create_with_groups فقط */
faux_dev = faux_device_create_with_groups("imx8-ecu-diag", NULL,
                                           &ecu_ops, ecu_groups);
```

**التحقق:**

```bash
$ ls /sys/bus/faux/devices/imx8-ecu-diag/
can_bus_load  can_error_count  power/  uevent

$ cat /sys/bus/faux/devices/imx8-ecu-diag/can_error_count
42
```

#### الدرس المستفاد
الـ `faux_probe()` بتضيف الـ `groups` تلقائياً **بعد** نجاح `faux_ops->probe()`. لو أضفتهم داخل الـ probe يدوياً وبعتهم كمان في `faux_device_create_with_groups()`، هيتضافوا مرتين والثانية هتفشل وتشيل الأولى. اختار واحد بس: إما `faux_device_create_with_groups()` أو `device_add_groups()` داخل probe.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — crash عند module unload

#### العنوان
kernel panic عند `rmmod` لموديول يستخدم faux device بسبب ترتيب خاطئ في cleanup

#### السياق
custom board بيشتغل على RK3562 لـ industrial HMI. المطور بيعمل bring-up لdrايفر virtual touchpad controller — الـ hardware موجود على I2C لكن الـ driver logic محتاج virtual device كـ abstraction layer بين الـ I2C driver والـ input subsystem.

#### المشكلة
الموديول بيشتغل كويس، لكن عند `rmmod`:

```bash
$ rmmod rk3562_vtpad
[  45.231] BUG: kernel NULL pointer dereference, address: 0000000000000010
[  45.231] CPU: 0 PID: 891 Comm: rmmod Not tainted
[  45.231] Call Trace:
[  45.231]  faux_remove+0x2c/0x58
[  45.231]  device_release_driver_internal+0x...
[  45.231]  faux_device_destroy+0x...
[  45.231]  vtpad_module_exit+0x18/0x30
```

#### التحليل
المطور عمل الكود التالي:

```c
static struct faux_device *g_faux_dev;
static struct input_dev   *g_input_dev;

static int vtpad_probe(struct faux_device *faux_dev)
{
    g_input_dev = input_allocate_device();
    if (!g_input_dev)
        return -ENOMEM;

    /* setup input_dev */
    input_set_drvdata(g_input_dev, faux_dev);
    return input_register_device(g_input_dev);
}

static void vtpad_remove(struct faux_device *faux_dev)
{
    /* ← غلط: input_unregister_device بيعمل free للـ input_dev */
    input_unregister_device(g_input_dev);
    g_input_dev = NULL;
}

static void __exit vtpad_module_exit(void)
{
    faux_device_destroy(g_faux_dev);   /* ← 1: بتنادي vtpad_remove */
    /* الـ g_input_dev بقى NULL هنا */
}
```

المشكلة إن في مكان تاني في الكود، الـ driver بيعمل:

```c
/* في interrupt handler أو work queue */
static void process_touch_event(struct work_struct *work)
{
    struct vtpad_priv *priv = container_of(work, ...);

    /* لو الـ work بيشتغل بالتوازي مع rmmod */
    input_report_key(g_input_dev, BTN_TOUCH, 1);  /* ← NULL dereference! */
}
```

الـ `faux_device_destroy()` بتنادي `device_del()` ثم `put_device()`. الـ `device_del()` بتنادي الـ driver core اللي بينادي `faux_remove()` → `vtpad_remove()` اللي بتعمل `g_input_dev = NULL`. لو في نفس الوقت الـ work queue لسه شغال، بيحصل race condition.

الترتيب الصحيح للـ cleanup واضح من `faux.c`:

```c
void faux_device_destroy(struct faux_device *faux_dev)
{
    struct device *dev = &faux_dev->dev;

    if (!faux_dev)
        return;

    device_del(dev);       /* ← بينادي faux_remove → vtpad_remove هنا */
    put_device(dev);       /* ← بعدين بيعمل release للـ memory */
}
```

#### الحل

**أولاً:** إلغاء الـ work queue قبل destroy:

```c
static void __exit vtpad_module_exit(void)
{
    /* 1. أوقف الـ work queue الأول */
    cancel_work_sync(&vtpad_work);

    /* 2. بعدين destroy الـ faux device */
    faux_device_destroy(g_faux_dev);
    g_faux_dev = NULL;
}
```

**ثانياً:** استخدام `devm_input_allocate_device()` بدل `input_allocate_device()`:

```c
static int vtpad_probe(struct faux_device *faux_dev)
{
    struct input_dev *input_dev;

    /* devm: automatically unregistered when faux_dev is destroyed */
    input_dev = devm_input_allocate_device(&faux_dev->dev);
    if (!input_dev)
        return -ENOMEM;

    /* setup */
    faux_device_set_drvdata(faux_dev, input_dev);
    return input_register_device(input_dev);
}

/* remove مش محتاجة manual cleanup بعد كده */
static const struct faux_device_ops vtpad_ops = {
    .probe  = vtpad_probe,
    .remove = NULL,
};
```

**التحقق بعد الإصلاح:**

```bash
$ insmod rk3562_vtpad.ko
$ ls /sys/bus/faux/devices/
rk3562-vtpad

$ ls /sys/bus/input/devices/ | grep vtpad
input5  # الـ input device ظهر

$ rmmod rk3562_vtpad
$ dmesg | tail -5
[  78.441] rk3562-vtpad: device removed cleanly
# مفيش crash
```

#### الدرس المستفاد
**الـ** `faux_device_destroy()` **بتنادي** `faux_remove()` **→** `faux_ops->remove()` **بشكل synchronous داخل** `device_del()`. لازم تتأكد إن أي work queues أو threads بتستخدم الـ private data اتوقفت قبل ما `faux_device_destroy()` تتنادى. استخدم `devm_*` allocations وربطها بـ `faux_dev->dev` لضمان الـ cleanup order الصحيح تلقائياً. الـ `faux_device_release()` في `faux.c` بتتنادى من `put_device()` وبتعمل `kfree(faux_obj)` — أي memory مش managed بـ devm هتتسرب.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/) | مراجعة شاملة لـ device model في الكيرنل — بيوضح إزاي الـ devices مش لازم تكون real hardware |
| [The platform device API](https://lwn.net/Articles/448499/) | شرح تفصيلي لـ `platform_device` — المقال ده أساسي لفهم ليه `faux_device` جه كبديل |
| [Platform devices and device trees](https://lwn.net/Articles/448502/) | الجزء التاني — بيشرح العلاقة بين platform devices وـ device tree |
| [Linux Device Drivers, Third Edition](https://lwn.net/Kernel/LDD3/) | الكتاب الكامل متاح على LWN — المرجع الأساسي لكتابة drivers |
| [Device drivers infrastructure](https://static.lwn.net/kerneldoc/driver-api/infrastructure.html) | توثيق رسمي لـ driver API في الكيرنل |

---

### توثيق الكيرنل الرسمي (`Documentation/`)

الملفات دي موجودة في source tree الكيرنل:

```
Documentation/driver-api/driver-model/device.rst
Documentation/driver-api/driver-model/driver.rst
Documentation/driver-api/driver-model/bus.rst
Documentation/driver-api/driver-model/overview.rst
Documentation/core-api/kobject.rst
Documentation/filesystems/sysfs.rst
```

**الـ online docs:**

| الرابط | الموضوع |
|--------|---------|
| [The Basic Device Structure](https://docs.kernel.org/driver-api/driver-model/device.html) | توثيق `struct device` — الأساس اللي بيتبنى عليه `struct faux_device` |
| [Device Drivers](https://docs.kernel.org/driver-api/driver-model/driver.html) | توثيق `struct device_driver` وآلية الـ probe/remove |
| [Everything you never wanted to know about kobjects](https://docs.kernel.org/next/core-api/kobject.html) | شرح كامل لـ `kobject` — اللبنة الأساسية للـ device model |
| [sysfs filesystem](https://docs.kernel.org/filesystems/sysfs.html) | توثيق الـ sysfs — اللي بيظهر فيه الـ `faux_device` تحت `/sys/bus/faux/devices/` |

---

### الـ Commits والـ Patches

**الـ Faux Bus** اتضاف في Linux 6.14، وده أهم الروابط:

| الرابط | الوصف |
|--------|-------|
| [Faux Bus Merged For Linux 6.14 — Phoronix](https://www.phoronix.com/news/Linux-6.14-Faux-Bus-Merged) | خبر الـ merge الرسمي مع C وRust bindings في نفس الـ commit |
| [Faux Bus Proposal — Phoronix](https://www.phoronix.com/news/Linux-Faux-Bus-Proposal) | إعلان الـ proposal الأول من Greg Kroah-Hartman في فبراير 2025 |
| [v4 Patch Series on Patchew](https://patchew.org/linux/2025021023-sandstorm-precise-9f5d@gregkh/) | الـ patch series الكامل — الـ v4 اللي اتقبل |
| [GIT PULL Driver core updates for 6.14-rc1](https://lore.kernel.org/lkml/2025012824-concept-scoring-6e2e@gregkh/T/) | الـ pull request الرسمي من gregkh لـ 6.14 |
| [drm/vkms: convert to use faux_device](https://lkml.rescloud.iu.edu/2506.1/09199.html) | مثال حقيقي — تحويل vkms driver من platform_device لـ faux_device |

**للبحث في الـ git log:**
```bash
git log --oneline --all -- drivers/base/faux.c
git log --oneline --all -- include/linux/device/faux.h
git log --oneline --all -- rust/kernel/faux.rs
```

---

### نقاشات Mailing List

| الرابط | الوصف |
|--------|-------|
| [lore.kernel.org — linux-kernel](https://lore.kernel.org/lkml/) | الأرشيف الكامل لـ LKML — ابحث عن `faux_device` |
| [Phoronix Forums Discussion](https://www.phoronix.com/forums/forum/phoronix/latest-phoronix-articles/1523627-faux-bus-proposed-for-the-linux-kernel-to-better-deal-with-simple-devices) | نقاش المجتمع على الـ proposal |

**search query مباشر على lore:**
```
https://lore.kernel.org/lkml/?q=faux_device
```

---

### الكتب الموصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **متاح مجانًا:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **الفصول ذات الصلة:**
  - Chapter 14: *The Linux Device Model* — الـ `kobject`, `kset`, `ktype`, bus/driver/device
  - Chapter 3: *Char Drivers* — مثال على `dev_set_drvdata` / `dev_get_drvdata`

#### Linux Kernel Development — Robert Love
- **الطبعة:** 3rd Edition (2010)
- **الفصول ذات الصلة:**
  - Chapter 17: *Devices and Modules* — شرح device model وـ sysfs
  - بيشرح العلاقة بين `bus_type` وـ `device_driver` وـ `device`

#### Embedded Linux Primer — Christopher Hallinan
- **الطبعة:** 2nd Edition
- **الفصل ذو الصلة:**
  - Chapter 8: *Device Driver Basics* — بيشرح platform devices وـ virtual devices في السياق الـ embedded

---

### kernelnewbies.org

| الرابط | الوصف |
|--------|-------|
| [Linux_6.14 — Kernel Newbies](https://kernelnewbies.org/Linux_6.14) | تغييرات Linux 6.14 — بتشمل إضافة الـ faux bus |
| [Documents — Kernel Newbies](https://kernelnewbies.org/Documents) | فهرس الكتب والمراجع المنصوح بيها للمبتدئين |
| [FirstKernelPatch — Kernel Newbies](https://kernelnewbies.org/FirstKernelPatch) | دليل إرسال أول patch — مهم لو عايز تساهم في الـ faux subsystem |

---

### elinux.org

| الرابط | الوصف |
|--------|-------|
| [Linux Device Drivers — eLinux](https://elinux.org/Linux_Device_Drivers) | صفحة مرجعية شاملة لـ device drivers |
| [Device Drivers Presentations — eLinux](https://elinux.org/Device_Drivers_Presentations) | presentations من مؤتمرات مختلفة عن الـ driver model |
| [Device Tree Reference — eLinux](https://elinux.org/Device_Tree_Reference) | مرجع الـ Device Tree — للمقارنة مع الـ faux approach في الأنظمة المدمجة |

---

### مصطلحات البحث

لو عايز تلاقي معلومات أكتر:

```
"faux_device" site:lore.kernel.org
"faux bus" linux kernel driver
linux kernel "platform_device" replacement virtual device
linux driver core bus_type probe remove
gregkh faux.c drivers/base
linux kernel device model sysfs kobject tutorial
```

**على بحث GitHub:**
```
repo:torvalds/linux faux_device_create
repo:torvalds/linux filename:faux.c
```

---

### ملخص الموارد الأساسية

```
المصدر الأول  → include/linux/device/faux.h  (الـ header نفسه)
المصدر التاني → drivers/base/faux.c           (الـ implementation)
المصدر التالت → rust/kernel/faux.rs           (الـ Rust bindings)
المصدر الرابع → LDD3 Chapter 14              (الخلفية النظرية)
المصدر الخامس → lwn.net/Articles/645810      (device model overview)
المصدر السادس → lwn.net/Articles/448499      (platform device — الـ predecessor)
```
## Phase 8: Writing simple module

### الفكرة

**`faux_device_create`** هي الدالة الأنسب للـ kprobe — بيتم استدعاؤها كل ما حد في الـ kernel يعمل faux device جديد (زي أي driver بيستخدم الـ faux bus بدل platform_device). هنعمل kprobe عليها عشان نشوف اسم الـ device اللي بيتعمل، ومين parent بتاعه.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * faux_kprobe.c — kprobe on faux_device_create to log every faux device
 * creation event: name, parent device name, and ops pointer.
 */

/* ---- includes ---- */
#include <linux/module.h>       /* MODULE_*, module_init/exit            */
#include <linux/kernel.h>       /* pr_info, pr_err                       */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...   */
#include <linux/device/faux.h>  /* struct faux_device, faux_device_ops   */
#include <linux/device.h>       /* struct device, dev_name()             */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on faux_device_create — log every faux device creation");

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler                                                  */
/*  Called just BEFORE faux_device_create() executes.                  */
/*                                                                      */
/*  Prototype of the hooked function:                                   */
/*    struct faux_device *faux_device_create(                           */
/*        const char               *name,                               */
/*        struct device            *parent,                             */
/*        const struct faux_device_ops *faux_ops);                      */
/*                                                                      */
/*  On x86-64 the arguments live in:                                    */
/*    regs->di  → name      (1st arg)                                   */
/*    regs->si  → parent    (2nd arg)                                   */
/*    regs->dx  → faux_ops  (3rd arg)                                   */
/* ------------------------------------------------------------------ */
static int faux_create_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Pull the three arguments straight from CPU registers */
    const char               *name     = (const char *)regs->di;
    struct device            *parent   = (struct device *)regs->si;
    const struct faux_device_ops *ops  = (const struct faux_device_ops *)regs->dx;

    /*
     * dev_name() returns the kobject name of the parent device.
     * If parent is NULL the device will hang under the faux bus root.
     */
    pr_info("faux_kprobe: creating faux device '%s'  parent='%s'  ops=%ps\n",
            name,
            parent ? dev_name(parent) : "(faux-root)",
            ops);

    /* Print which callbacks are registered, if any */
    if (ops) {
        pr_info("faux_kprobe:   ops->probe=%ps  ops->remove=%ps\n",
                ops->probe, ops->remove);
    }

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe definition                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe faux_kp = {
    .symbol_name = "faux_device_create",   /* symbol to hook          */
    .pre_handler = faux_create_pre,        /* called before the func  */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init faux_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&faux_kp);
    if (ret < 0) {
        pr_err("faux_kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("faux_kprobe: hooked faux_device_create at %p\n",
            faux_kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit faux_kprobe_exit(void)
{
    unregister_kprobe(&faux_kp);
    pr_info("faux_kprobe: unhooked faux_device_create\n");
}

module_init(faux_kprobe_init);
module_exit(faux_kprobe_exit);
```

---

### Makefile

```makefile
obj-m += faux_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه موجود |
|--------|-----------|
| `<linux/module.h>` | الـ macros الأساسية لأي kernel module — `MODULE_LICENSE`، `module_init/exit` |
| `<linux/kprobes.h>` | بيعرّف `struct kprobe` وكل الـ API الخاصة بالـ kprobes |
| `<linux/device/faux.h>` | عشان نستخدم `struct faux_device_ops` في الـ cast بتاع الـ arguments |
| `<linux/device.h>` | بيوفر `dev_name()` اللي بنستخدمه لطباعة اسم الـ parent device |

#### الـ `pre_handler` — `faux_create_pre`

**الـ `struct pt_regs *regs`** هو snapshot للـ CPU registers في لحظة ما الـ kprobe اتفعّل. على x86-64، الـ calling convention بتحط أول 3 arguments في `rdi`/`rsi`/`rdx`، فبنعمل cast مباشر من `regs->di`، `regs->si`، `regs->dx` للـ types الصح.

الـ `%ps` في `pr_info` بيطبع اسم الـ symbol المقابل للـ pointer (زي اسم الـ function) بدل الـ address الخام — ده مفيد جداً لو الـ ops struct جوّا module معروف.

#### الـ `struct kprobe`

**`.symbol_name`** هو كل اللي محتاجه — الـ kernel بيعمل lookup للـ symbol في الـ kallsyms تلقائياً ويحط الـ breakpoint. مش محتاج تحسب الـ address يدوياً.

الـ **`.pre_handler`** بيتشغّل قبل تنفيذ الـ instruction الأولى في `faux_device_create`، فبتشوف الـ arguments كما أرسلها الـ caller قبل أي تعديل.

#### الـ `module_init`

بيسجّل الـ kprobe — من اللحظة دي أي حد في الـ kernel يعمل `faux_device_create()` هيتوقف لحظة وهيشغّل الـ pre_handler بتاعنا. لو الـ `register_kprobe` رجع error (مثلاً لو الـ symbol مش موجود أو CONFIG_KPROBES مش enabled) بنرجع الـ error فوراً.

#### الـ `module_exit`

**`unregister_kprobe`** ضروري جداً — لو عملنا `rmmod` من غير ما نشيل الـ kprobe، الـ kernel هيفضل بيستدعي كود اتحذف من الـ memory وده kernel panic مضمون. دي مش optional.

---

### اختبار تشغيلي

```bash
# بناء الـ module
make

# تحميله
sudo insmod faux_kprobe.ko

# شوف الـ log (أي driver بيستخدم faux_device_create هيظهر هنا)
sudo dmesg | tail -20

# لو عايز تجرب بنفسك بدون انتظار driver:
# اعمل module تاني بسيط يعمل faux_device_create() وبعدين faux_device_destroy()

# إزالة الـ module
sudo rmmod faux_kprobe

# تأكيد
sudo dmesg | grep faux_kprobe
```

**مثال على الـ output المتوقع:**

```
[  245.123456] faux_kprobe: hooked faux_device_create at ffffffffc0ab1234
[  246.789012] faux_kprobe: creating faux device 'my-fw-loader'  parent='(faux-root)'  ops=0xffffffffc0ab5678
[  246.789015] faux_kprobe:   ops->probe=fw_loader_probe  ops->remove=fw_loader_remove
```
