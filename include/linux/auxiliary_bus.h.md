## Phase 1: الصورة الكبيرة

# `auxiliary_bus.h`

> **PATH**: `include/linux/auxiliary_bus.h`
> **Subsystem**: AUXILIARY BUS DRIVER (driver-core)
> **الوظيفة الأساسية**: تعريف الـ API الخاص بالـ Auxiliary Bus — وهو نظام يسمح لـ driver واحد أن يُولّد أجهزة "أبناء" افتراضية داخلية، كل واحدة تُشغّلها driver مستقل.

---

### ما هي المشكلة اللي بيحلّها هذا الملف؟

تخيّل عندك **كرت شبكة PCI** واحد، لكنه في الحقيقة يعمل ثلاثة أشياء مختلفة:

1. شبكة عادية (Ethernet)
2. شبكة سريعة جداً (RDMA)
3. صوت (Audio DSP)

لو كتبت driver واحد كبير يتحكم في كل هذا، هيبقى كارثة — ضخم، صعب صيانته، وكل تيم لازم يدخل على كود التيم التاني.

الحل؟ **الـ Auxiliary Bus**: الـ driver الأساسي (الأب/Parent) يُسجّل "أجهزة وهمية" صغيرة على bus خاص اسمه الـ Auxiliary Bus. كل جهاز وهمي بيمثّل وظيفة واحدة (RDMA مثلاً). بعدين driver مستقل (تيم RDMA) يشتغل على هذا الجهاز الوهمي بشكل منفصل تماماً.

```
┌─────────────────────────────────────────────┐
│           PCI NIC Driver (الأب)             │
│   يُسجّل auxiliary_device لكل وظيفة        │
└──────┬─────────────────────┬────────────────┘
       │                     │
       ▼                     ▼
┌─────────────┐     ┌─────────────────┐
│ aux_dev:    │     │ aux_dev:        │
│ foo_mod.eth │     │ foo_mod.rdma    │
└──────┬──────┘     └────────┬────────┘
       │                     │
       ▼                     ▼
┌─────────────┐     ┌─────────────────┐
│ Ethernet    │     │  RDMA           │
│ Driver      │     │  Driver         │
└─────────────┘     └─────────────────┘
```

---

### ما الذي يعرّفه هذا الملف؟

#### 1. الـ `struct auxiliary_device` — الجهاز الوهمي

```c
struct auxiliary_device {
    struct device dev;    /* الجهاز الحقيقي في نظام Linux */
    const char *name;     /* اسم الجهاز مثل "foo_dev" */
    u32 id;               /* رقم مميز لو في أكثر من جهاز بنفس الاسم */
    struct {
        struct xarray irqs;       /* قائمة الـ IRQs الخاصة بهذا الجهاز */
        struct mutex lock;
        bool irq_dir_exists;
    } sysfs;
};
```

الاسم النهائي للجهاز على الـ bus بيكون: `modname.devname.id`
مثلاً: `foo_mod.foo_dev.0`

#### 2. الـ `struct auxiliary_driver` — الـ driver الذي يشتغل على الجهاز الوهمي

```c
struct auxiliary_driver {
    int (*probe)(...);    /* لما يُكتشف الجهاز */
    void (*remove)(...);  /* لما يُحذف الجهاز */
    void (*shutdown)(...);
    int (*suspend)(...);
    int (*resume)(...);
    const char *name;
    struct device_driver driver;
    const struct auxiliary_device_id *id_table; /* قائمة الأجهزة التي يدعمها */
};
```

#### 3. دورة حياة الجهاز — ثلاث خطوات للتسجيل

```c
// الخطوة 1: اضبط الحقول
my_aux_dev->name = "foo_dev";
my_aux_dev->id = 0;
my_aux_dev->dev.release = my_release_fn;
my_aux_dev->dev.parent  = my_parent_dev;

// الخطوة 2: ابدأ التهيئة
auxiliary_device_init(my_aux_dev);

// الخطوة 3: سجّل على الـ bus
auxiliary_device_add(my_aux_dev);
```

وللحذف، خطوتان فقط:
```c
auxiliary_device_delete(my_aux_dev);
auxiliary_device_uninit(my_aux_dev);
```

#### 4. الـ Helper Functions والـ Macros

| الاسم | الوظيفة |
|---|---|
| `auxiliary_device_add(auxdev)` | Macro يُضيف الجهاز باستخدام `KBUILD_MODNAME` تلقائياً |
| `auxiliary_driver_register(auxdrv)` | يسجّل الـ driver على الـ bus |
| `auxiliary_driver_unregister(auxdrv)` | يلغي تسجيل الـ driver |
| `module_auxiliary_driver(drv)` | Macro يستبدل `module_init/exit` لحالات الـ drivers البسيطة |
| `auxiliary_get_drvdata / auxiliary_set_drvdata` | حفظ واسترجاع بيانات خاصة بالـ driver |
| `to_auxiliary_dev(dev)` | تحويل `struct device *` إلى `struct auxiliary_device *` |
| `devm_auxiliary_device_create(...)` | إنشاء جهاز مرتبط بعمر الـ parent device (devres managed) |
| `auxiliary_device_sysfs_irq_add/remove` | إضافة/حذف IRQ من الـ sysfs |

---

### لماذا هذا الـ bus وليس Platform Bus مثلاً؟

| المقارنة | Platform Bus | Auxiliary Bus |
|---|---|---|
| من يمتلك الذاكرة؟ | الـ kernel | الـ driver الأب نفسه |
| من يُحرّر الذاكرة؟ | الـ kernel | الـ driver الأب (في `release()`) |
| Binding | بالـ name فقط | بالـ `modname.devname` |
| الهدف | أجهزة hardware فعلية | وظائف فرعية لـ driver واحد |

---

### قصة واقعية: كرت الصوت (Audio DSP)

شركة Intel عندها كرت صوت واحد بيعمل:
- HDMI audio
- SoundWire
- ميكروفونات محلية
- Speakers

الكرت نفسه PCI device واحد. بدل ما driver واحد يتحمل كل هذا، يُسجّل driver الأب ثلاثة `auxiliary_device`:
- `snd_hda.hdmi.0`
- `snd_hda.soundwire.0`
- `snd_hda.dmic.0`

كل واحد بيشتغل عليه driver متخصص مستقل.

---

### ملاحظة مهمة: إدارة الذاكرة

الـ Auxiliary Bus **مختلف** عن باقي الـ buses في نقطة مهمة جداً:

- **الـ driver الأب** هو المسؤول عن تخصيص وتحرير الذاكرة الخاصة بالـ `auxiliary_device`.
- يجب على الـ driver الأب أن يُلغي تسجيل كل الـ auxiliary devices قبل أن يُكمل دالة `remove()` الخاصة به.
- أفضل طريقة: استخدام `devm_add_action_or_reset()` لضمان التنظيف التلقائي.

---

### الملفات المكوّنة لهذا الـ Subsystem

| الملف | الدور |
|---|---|
| `include/linux/auxiliary_bus.h` | الـ API الرئيسي — structs + macros + inline functions |
| `drivers/base/auxiliary.c` | التنفيذ الفعلي للـ bus (match, probe, sysfs, ...) |
| `include/linux/mod_devicetable.h` | تعريف `struct auxiliary_device_id` للـ matching |
| `Documentation/driver-api/auxiliary_bus.rst` | التوثيق الرسمي بالتفصيل |
| `rust/kernel/auxiliary.rs` | واجهة Rust للـ Auxiliary Bus |
| `rust/helpers/auxiliary.c` | C helpers للـ Rust bindings |
| `samples/rust/rust_driver_auxiliary.rs` | مثال كامل بـ Rust |

### ملفات ذات صلة يجب معرفتها

- **`include/linux/device.h`**: يعرّف `struct device` و`struct device_driver` اللي يرثهما `auxiliary_device` و`auxiliary_driver`.
- **`include/linux/mod_devicetable.h`**: يعرّف `struct auxiliary_device_id` المستخدم في الـ id_table لمطابقة الأجهزة بالـ drivers.
- **`drivers/base/auxiliary.c`**: القلب التنفيذي — يحتوي على دوال الـ match وتسجيل الـ bus وإدارة الـ sysfs.
## Phase 2: شرح الـ Auxiliary Bus Framework

---

### المشكلة — ليش موجود أصلاً؟

تخيل عندك **بطاقة شبكة** (NIC) من نوع Intel، وهذه البطاقة مش بس تشيل وتوصّل بيانات الشبكة — هي كمان فيها:

- وحدة **RDMA** (Remote Direct Memory Access) — للوصول المباشر للذاكرة عبر الشبكة
- وحدة **crypto** — لتشفير البيانات
- وحدة **management** — لإدارة البطاقة نفسها

كل وحدة من هذه تحتاج **driver منفصل** في Linux. المشكلة: هذه الوحدات مش أجهزة مستقلة فيزيائياً على الـ PCI bus — هي **جزء من نفس الـ PCI function**، لكن يجب إدارتها بكود منفصل.

#### الحلول القديمة وإشكالياتها

**الحل الأول: driver واحد ضخم**
كتابة driver واحد يدير كل شيء. النتيجة؟ كود معقد، coupling شديد، ومستحيل اختباره أو صيانته.

**الحل الثاني: platform_device**
الـ `platform_device` موجود أصلاً لأجهزة مدمجة في الـ SoC ليس لها bus فيزيائي. استخدامه لـ PCI sub-functions كان **hack** غير مناسب:
- لا يوجد parent-child relationship واضح
- إدارة الذاكرة يدوية وخطرة
- لا يوجد اتفاق على naming convention

**الحل الثالث: مكتبة مشتركة (shared library in kernel)**
كتابة كود مشترك في module منفصل وربطه بـ symbols. المشكلة: لا يوجد lifecycle management حقيقي.

---

### الحل — ماذا فعل الـ Auxiliary Bus؟

الـ **Auxiliary Bus** (أُضيف في Linux 5.11، بواسطة Intel) هو **virtual bus** — أي لا يمثل hardware bus حقيقياً مثل PCI أو USB، بل هو بنية تنظيمية داخل الـ kernel تمكّن driver واحد ("parent/registering driver") من إنشاء "أجهزة وهمية" sub-devices وربطها بـ drivers متخصصة.

الفكرة الجوهرية:

```
Driver الأصل (مثلاً: ice.ko لبطاقة Intel)
    │
    ├── ينشئ auxiliary_device "ice.rdma.0"
    ├── ينشئ auxiliary_device "ice.crypto.0"
    └── ينشئ auxiliary_device "ice.mgmt.0"
            │
            ▼
    كل auxiliary_device يُربط بـ auxiliary_driver مستقل
    (ice_rdma.ko, ice_crypto.ko, ice_mgmt.ko)
```

هذا يعني:
- **فصل واضح** بين الكود المشترك والكود المتخصص
- **lifecycle management** حقيقي عبر الـ device model
- **مشاركة البيانات** بطريقة آمنة عبر `container_of()`

---

### الـ Real-World Analogy

تخيل **فندق كبير** (الـ PCI device):

- المبنى كله ملك شركة واحدة (الـ parent driver)
- داخل الفندق: مطعم، صالة رياضية، غرف نوم — كل قسم له **مدير مستقل** (auxiliary driver)
- الشركة الأم تبني هذه الأقسام (تسجّل الـ auxiliary devices) وتضع لافتة على كل باب بالاسم
- كل مدير قسم يدخل، يرى اللافتة، ويقول "أنا المسؤول عن هذا" (الـ probe)
- الشركة الأم تحتفظ بمفاتيح المبنى (الذاكرة المشتركة) وتحدد متى يُغلق كل قسم

---

### البنية الكبيرة — أين يقع الـ Auxiliary Bus في الـ Kernel؟

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                               │
└────────────────────────────┬────────────────────────────────────┘
                             │  syscalls
┌────────────────────────────▼────────────────────────────────────┐
│                       Linux Kernel                              │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Driver Model (drivers/base/)           │   │
│  │                                                          │   │
│  │   ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │   │
│  │   │ PCI Bus │  │ USB Bus  │  │Platform  │  │Auxiliary│  │   │
│  │   │         │  │          │  │   Bus    │  │  Bus   │  │   │
│  │   └────┬────┘  └────┬─────┘  └────┬─────┘  └───┬────┘  │   │
│  │        │            │             │             │        │   │
│  └────────┼────────────┼─────────────┼─────────────┼────────┘   │
│           │            │             │             │             │
│    ┌──────▼──────┐     │      ┌──────▼──────┐  ┌──▼──────────┐ │
│    │  ice.ko     │     │      │ gpio.ko     │  │ice_rdma.ko  │ │
│    │ (PCI driver)│     │      │(platform drv│  │(aux driver) │ │
│    │             │     │      │             │  │             │ │
│    │ ┌─────────┐ │     │      └─────────────┘  └─────────────┘ │
│    │ │creates  │ │     │                                        │
│    │ │aux devs │ │     │      ┌─────────────┐  ┌─────────────┐ │
│    │ └────┬────┘ │     │      │ice_crypto.ko│  │ice_mgmt.ko  │ │
│    └──────┼──────┘     │      │(aux driver) │  │(aux driver) │ │
│           │            │      └─────────────┘  └─────────────┘ │
│    ┌──────▼────────────────────────────────────────────────┐    │
│    │              Auxiliary Bus                            │    │
│    │   [ice.ice_rdma.0]  [ice.ice_crypto.0]  [ice.mgmt.0] │    │
│    └────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Structs الأساسية وعلاقاتها

#### `struct auxiliary_device`

هذا الـ struct يمثّل **الجهاز الوهمي** الذي ينشئه الـ parent driver:

```c
struct auxiliary_device {
    struct device dev;      /* embedded device — قلب الـ device model */
    const char *name;       /* اسم الجهاز: "rdma", "crypto", إلخ */
    u32 id;                 /* رقم تمييزي لو في أكثر من جهاز بنفس الاسم */
    struct {
        struct xarray irqs;     /* قائمة IRQs المرتبطة بالجهاز */
        struct mutex lock;      /* حماية الـ sysfs من race conditions */
        bool irq_dir_exists;    /* هل مجلد irqs موجود في sysfs؟ */
    } sysfs;
};
```

**الـ full name** الذي يُسجَّل في الـ bus يُبنى هكذا:

```
[KBUILD_MODNAME].[name].[id]
مثلاً: ice.rdma.0
        │    │   └── id
        │    └────── name
        └─────────── اسم الـ module (ice.ko)
```

#### `struct auxiliary_driver`

هذا الـ struct يمثّل **الـ driver** الذي يتعامل مع الـ auxiliary device:

```c
struct auxiliary_driver {
    int  (*probe)   (struct auxiliary_device *, const struct auxiliary_device_id *);
    void (*remove)  (struct auxiliary_device *);
    void (*shutdown)(struct auxiliary_device *);
    int  (*suspend) (struct auxiliary_device *, pm_message_t state);
    int  (*resume)  (struct auxiliary_device *);
    const char *name;
    struct device_driver driver;         /* embedded — للربط مع الـ bus core */
    const struct auxiliary_device_id *id_table; /* جدول الأسماء التي يدعمها */
};
```

#### `struct auxiliary_device_id`

```c
struct auxiliary_device_id {
    char name[AUXILIARY_NAME_SIZE];  /* 40 حرف — اسم الجهاز الكامل */
    kernel_ulong_t driver_data;      /* بيانات خاصة بالـ driver */
};
```

هذا الـ struct يُستخدم في **المطابقة** — الـ bus core يقارن `id_table` الخاص بكل driver مع اسم كل `auxiliary_device` مسجّل.

---

### العلاقة بين الـ Structs — رسم بياني

```
┌──────────────────────────────────────────────────────────────┐
│                   Parent Object (في الـ parent driver)        │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              struct auxiliary_device                   │  │
│  │                                                        │  │
│  │   ┌──────────────────────────────────────────────┐    │  │
│  │   │            struct device dev                 │    │  │
│  │   │   .parent  ──────────────────► parent device │    │  │
│  │   │   .release ──────────────────► free function │    │  │
│  │   │   .bus     ──────────────────► auxiliary_bus │    │  │
│  │   └──────────────────────────────────────────────┘    │  │
│  │                                                        │  │
│  │   .name  = "rdma"                                      │  │
│  │   .id    = 0                                           │  │
│  │   .sysfs.irqs  = {xarray of IRQ indices}               │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│   *shared_data  ─────────────────────────────────────────►  │
│   (بيانات مشتركة بين الـ parent والـ auxiliary driver)       │
└──────────────────────────────────────────────────────────────┘
                              │
                    container_of() ◄─────────────────┐
                              │                       │
                              ▼                       │
┌──────────────────────────────────────────────────┐  │
│            struct auxiliary_driver               │  │
│                                                  │  │
│   ┌────────────────────────────────────────────┐ │  │
│   │         struct device_driver driver        │ │  │
│   │   .name = "myauxiliarydrv"                 │ │  │
│   │   .bus  ──────────────────► auxiliary_bus  │ │  │
│   └────────────────────────────────────────────┘ │  │
│                                                  │  │
│   .id_table ──────► [{"ice.rdma"}, {""}, ...]    │  │
│   .probe()  ──────────────────────────────────────┘  │
│   .remove()                                      │
└──────────────────────────────────────────────────┘
```

---

### آلية الـ Name Matching — كيف يعرف الـ bus من يربط بمن؟

الـ bus core يقوم بما يلي:

1. عند تسجيل `auxiliary_device` أو `auxiliary_driver`، يُشغّل الـ bus الـ matching
2. يأخذ اسم كل device: `"ice.rdma.0"`
3. **يقتطع** رقم الـ id من النهاية → يصبح `"ice.rdma"`
4. يقارنه مع `id_table[i].name` لكل driver مسجّل
5. إذا تطابقا → يستدعي `driver->probe()`

```
Device name:    "ice.rdma.0"
                      │
                 strip ".0"
                      │
                      ▼
Match name:     "ice.rdma"
                      │
               compare with
                      │
                      ▼
id_table entry: { .name = "ice.rdma" }  ✓ MATCH!
                      │
                      ▼
               call probe()
```

---

### دورة حياة الـ auxiliary_device — خطوة بخطوة

#### التسجيل (3 خطوات إلزامية)

```c
/* خطوة 1: ملء الـ struct */
my_aux_dev->name       = "rdma";
my_aux_dev->id         = 0;
my_aux_dev->dev.release = my_release_fn;   /* إلزامي! */
my_aux_dev->dev.parent  = &pci_dev->dev;   /* الـ parent */

/* خطوة 2: التهيئة — تُشغّل device_initialize() */
if (auxiliary_device_init(my_aux_dev))
    goto fail;

/* خطوة 3: الإضافة للـ bus — تُولّد الاسم الكامل وتُضيف الجهاز */
if (auxiliary_device_add(my_aux_dev)) {
    auxiliary_device_uninit(my_aux_dev);   /* cleanup إذا فشل */
    goto fail;
}
```

ملاحظة: الـ macro `auxiliary_device_add` يُوسَّع إلى:
```c
__auxiliary_device_add(auxdev, KBUILD_MODNAME)
/* KBUILD_MODNAME = اسم الـ module المُصرَّف تلقائياً من الـ build system */
```

#### إلغاء التسجيل (خطوتان)

```c
/* خطوة 1: إزالة الجهاز من الـ bus (تُشغّل remove() للـ driver) */
auxiliary_device_delete(my_aux_dev);

/* خطوة 2: تحرير الـ reference — يُؤدي لاستدعاء release() عند وصول refcount لـ 0 */
auxiliary_device_uninit(my_aux_dev);
```

---

### إدارة الذاكرة — الجزء الأهم والأخطر

الـ Auxiliary Bus يختلف جذرياً عن الـ Platform Bus في من يملك الذاكرة:

```
Platform Bus:
  kernel core ──► ينشئ ويحرر الذاكرة

Auxiliary Bus:
  parent driver ──► هو المسؤول عن كل شيء
```

**القاعدة الذهبية:**
- الـ parent driver يُخصّص الذاكرة لـ `auxiliary_device`
- الـ parent driver يكتب دالة `release()` تُحرر هذه الذاكرة
- `release()` **لا تُستدعى مباشرة** — تُستدعى تلقائياً عندما يصل `refcount` للـ device إلى صفر
- قد يكون هذا بعد `auxiliary_device_uninit()` مباشرة أو بعد وقت طويل (لو في كود آخر يحتفظ بـ reference)

```
     Parent Driver                    Kernel
          │                              │
          │ alloc memory                 │
          │──────────────────────────►   │
          │                              │
          │ auxiliary_device_init()      │
          │──────────────────────────►   │   refcount = 1
          │                              │
          │ auxiliary_device_add()       │
          │──────────────────────────►   │   device on bus
          │                              │
          │                              │   probe() called on aux_driver
          │                              │   (aux_driver gets reference)
          │                              │   refcount = 2
          │                              │
          │ auxiliary_device_delete()    │
          │──────────────────────────►   │   remove() called
          │                              │   device removed from bus
          │                              │
          │ auxiliary_device_uninit()    │
          │──────────────────────────►   │   refcount-- → 1
          │                              │   (لم يصل لصفر بعد!)
          │                              │
          │                              │   aux_driver releases reference
          │                              │   refcount-- → 0
          │                              │
          │ ◄────────────────────────────│   release() called NOW
          │ free memory                  │
```

---

### مشاركة البيانات بين الـ Parent و الـ Auxiliary Driver

الطريقة الرسمية لمشاركة البيانات هي عبر **container_of()** pattern:

في الـ shared header (ملف .h مشترك بين الـ module الأصل والـ auxiliary driver):

```c
/* في shared_header.h */
struct my_shared_data {
    int some_value;
    void *hw_regs;
    /* ... */
};

struct my_parent_obj {
    struct auxiliary_device auxdev;   /* يجب أن يكون embedded! */
    struct my_shared_data *shared;    /* البيانات المشتركة */
    /* ... بيانات أخرى خاصة بالـ parent ... */
};
```

في الـ auxiliary driver عند الـ probe:

```c
static int my_aux_probe(struct auxiliary_device *auxdev,
                        const struct auxiliary_device_id *id)
{
    /* من الـ auxiliary_device نصل للـ parent object */
    struct my_parent_obj *parent_obj =
        container_of(auxdev, struct my_parent_obj, auxdev);

    /* الآن لدينا وصول للبيانات المشتركة */
    struct my_shared_data *data = parent_obj->shared;

    /* ... */
}
```

```
الذاكرة:
┌─────────────────────────────────────────────┐
│           struct my_parent_obj              │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │      struct auxiliary_device auxdev │◄──┼─── الـ pointer الذي يصل للـ probe
│  │                                     │   │
│  │   struct device dev                 │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  struct my_shared_data *shared  ──────────────► [shared data]
│                                             │
└─────────────────────────────────────────────┘
         ▲
         │
    container_of(auxdev, struct my_parent_obj, auxdev)
    يحسب offset الـ auxdev داخل my_parent_obj ويطرحه
    ليصل لبداية my_parent_obj
```

---

### الـ Helper Functions والـ Macros

#### الـ Inline Helpers

```c
/* الحصول على بيانات الـ driver الخاصة */
static inline void *auxiliary_get_drvdata(struct auxiliary_device *auxdev)
{
    return dev_get_drvdata(&auxdev->dev);
    /* تقرأ dev->driver_data */
}

/* تعيين بيانات الـ driver الخاصة */
static inline void auxiliary_set_drvdata(struct auxiliary_device *auxdev, void *data)
{
    dev_set_drvdata(&auxdev->dev, data);
    /* تكتب في dev->driver_data */
}

/* من struct device → struct auxiliary_device */
static inline struct auxiliary_device *to_auxiliary_dev(struct device *dev)
{
    return container_of(dev, struct auxiliary_device, dev);
}

/* من struct device_driver → struct auxiliary_driver */
static inline const struct auxiliary_driver *to_auxiliary_drv(const struct device_driver *drv)
{
    return container_of(drv, struct auxiliary_driver, driver);
}
```

#### الـ devm Variant — الأسهل والأأمن

```c
/* ينشئ auxiliary_device ويديره تلقائياً عبر devres */
#define devm_auxiliary_device_create(dev, devname, platform_data) \
    __devm_auxiliary_device_create(dev, KBUILD_MODNAME, devname, \
                                   platform_data, 0)
```

الـ `devm_` variant يعني: عندما يُزال الـ parent device، يُحذف الـ auxiliary device تلقائياً. هذا يُبسّط الكود ويُقلل من احتمالية الـ memory leaks.

#### تسجيل الـ Driver ببساطة

```c
/* بدلاً من كتابة module_init/module_exit يدوياً */
module_auxiliary_driver(my_drv);

/* يُوسَّع إلى: */
module_init(my_drv_init);   /* يستدعي auxiliary_driver_register */
module_exit(my_drv_exit);   /* يستدعي auxiliary_driver_unregister */
```

---

### الـ sysfs والـ IRQs — التفاصيل

الـ `auxiliary_device` يدعم تصدير IRQs إلى الـ sysfs:

```c
/* إضافة IRQ للـ sysfs ليراه المستخدم */
int auxiliary_device_sysfs_irq_add(struct auxiliary_device *auxdev, int irq);

/* إزالته */
void auxiliary_device_sysfs_irq_remove(struct auxiliary_device *auxdev, int irq);
```

في الـ sysfs يظهر هكذا:
```
/sys/bus/auxiliary/devices/ice.rdma.0/
    └── irqs/
        ├── 0  → 45   (IRQ number)
        └── 1  → 46
```

الـ `xarray irqs` داخل `sysfs` يحتفظ بـ mapping من index → IRQ number، وهو thread-safe بطبيعته.

---

### من يستخدم الـ Auxiliary Bus في الـ Kernel؟

أمثلة حقيقية من الـ kernel:

| الـ Parent Driver | الـ Auxiliary Devices | الغرض |
|---|---|---|
| `ice.ko` (Intel E800 NIC) | `ice.rdma`, `ice.iidc` | RDMA وإدارة الشبكة |
| `mlx5_core.ko` (Mellanox) | `mlx5_core.eth`, `mlx5_core.rdma` | Ethernet وـ RDMA |
| `irdma.ko` (Intel RDMA) | `irdma.lchnl` | قنوات الاتصال الداخلية |
| `qcom-qmp-combo.ko` | `qcom_qmp_phy.usb` | إدارة الـ PHY |

---

### مقارنة الـ Auxiliary Bus مع البدائل

| الجانب | Platform Bus | Auxiliary Bus | Custom Solution |
|---|---|---|---|
| **فيزيائي؟** | لا (virtual) | لا (virtual) | لا |
| **من يُنشئ الجهاز؟** | ACPI/DT/Code | الـ parent driver | الـ parent driver |
| **إدارة الذاكرة** | الـ kernel | الـ parent driver | يدوي |
| **Name matching** | اسم ثابت | MODNAME + name | يدوي |
| **Power management** | نعم | نعم | يدوي |
| **مشاركة البيانات** | صعبة | سهلة (container_of) | يدوي |
| **الاستخدام الصحيح** | SoC peripherals | Sub-functions في نفس الـ driver | لا يُنصح به |

---

### ملخص — لماذا الـ Auxiliary Bus ينجح؟

الـ Auxiliary Bus يحل المشكلة الكلاسيكية بأناقة:

**التحديات التي يحلها:**
1. **التنظيم**: كل وظيفة فرعية لها driver خاص بها، بدون تضخم الـ parent driver
2. **الأمان**: lifecycle management واضح — من يُخصّص يُحرر، وقواعد الـ refcounting صريحة
3. **المشاركة الآمنة**: `container_of()` يُتيح الوصول للبيانات المشتركة بدون global variables
4. **التوافق**: يستخدم نفس الـ device model الذي يعتمد عليه PCI وUSB — نفس الـ probe/remove/suspend/resume
5. **الـ sysfs تلقائياً**: الجهاز يظهر في `/sys/bus/auxiliary/devices/` فوراً
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options — Cheatsheet

#### الـ Config Options

| Option | الوظيفة |
|--------|---------|
| `CONFIG_SYSFS` | لو مفعّل، تشتغل دوال `auxiliary_device_sysfs_irq_add/remove` الحقيقية. لو موقف، يرجعوا stubs فاضية |

#### الـ Constants والـ Macros المهمة

| الاسم | القيمة | الوظيفة |
|-------|--------|---------|
| `AUXILIARY_NAME_SIZE` | 40 | الحد الأقصى لطول اسم الـ auxiliary device في جدول الـ ID |
| `AUXILIARY_MODULE_PREFIX` | `"auxiliary:"` | البريفكس اللي بيضيفه الـ kernel لأسماء الـ aliases في الـ module |
| `auxiliary_device_add(auxdev)` | macro | بيستدعي `__auxiliary_device_add` مع اسم الـ module تلقائيًا عبر `KBUILD_MODNAME` |
| `auxiliary_driver_register(auxdrv)` | macro | بيستدعي `__auxiliary_driver_register` مع `THIS_MODULE` و`KBUILD_MODNAME` |
| `module_auxiliary_driver(__drv)` | macro | يغني عن كتابة `module_init/module_exit` يدويًا للـ driver البسيط |
| `devm_auxiliary_device_create(dev, devname, data)` | macro | ينشئ device مربوط بعمر الـ parent تلقائيًا (managed resource) |

#### الـ Lifecycle Ownership Rules — ملخص سريع

| القاعدة | التفاصيل |
|--------|---------|
| من يخصص الذاكرة؟ | الـ registering driver (أي الـ parent driver) |
| من يحرر الذاكرة؟ | الـ `release()` callback اللي يحدده نفس الـ registering driver |
| متى تُحذف؟ | لما `put_device()` تنزل الـ refcount لصفر |
| ترتيب الـ unregister؟ | لازم `auxiliary_device_delete()` **قبل** `auxiliary_device_uninit()` |
| الـ shared objects؟ | عمرها لازم يساوي أو يتجاوز عمر الـ auxiliary_device |

---

### كل الـ Structs المهمة

---

#### 1. `struct auxiliary_device`

**الغرض:** يمثّل "جزء وظيفي" من device أصل. تخيّل إن عندك كارت شبكة فيه وظيفتين: شبكة عادية + RDMA. كل وظيفة بتتمثّل كـ `auxiliary_device` مستقل، بس الاثنين أبناء لنفس الـ PCI device.

```c
struct auxiliary_device {
    struct device dev;       /* الـ device الأساسي في نموذج Linux Driver Model */
    const char *name;        /* الاسم اللي الـ driver هيتعرف عليه (مثلاً "foo_dev") */
    u32 id;                  /* رقم فريد لو في أكتر من device بنفس الاسم */
    struct {
        struct xarray irqs;      /* مصفوفة الـ IRQ indices المخصصة لهذا الـ device */
        struct mutex lock;       /* mutex لحماية إنشاء sysfs entries للـ IRQs */
        bool irq_dir_exists;     /* هل مجلد "irqs" اتعمل في sysfs؟ */
    } sysfs;
};
```

**شرح الحقول:**

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `dev` | `struct device` | القلب — فيه الـ parent، الـ release callback، الـ kobject، إلخ |
| `name` | `const char *` | الجزء الثاني من match_name: مثلاً `"foo_dev"` في `"foo_mod.foo_dev"` |
| `id` | `u32` | رقم يميّز جهازين بنفس الاسم: `"foo_mod.foo_dev.0"` و`"foo_mod.foo_dev.1"` |
| `sysfs.irqs` | `struct xarray` | يخزن الـ IRQ numbers بشكل sparse وفعّال (أحسن من linked list) |
| `sysfs.lock` | `struct mutex` | يحمي عمليات إضافة/حذف الـ IRQ entries في sysfs |
| `sysfs.irq_dir_exists` | `bool` | flag لمعرفة إذا كان المجلد موجود، لتجنب إنشاءه مرتين |

**الربط بـ structs تانية:**
- **يحتوي** على `struct device` (embedded، مش pointer)
- **يُشار إليه** من الـ parent driver بـ pointer عادةً داخل struct أكبر
- الـ `struct device` اللي جوّاه **يشير** لـ `struct bus_type` الخاصة بالـ auxiliary bus

---

#### 2. `struct auxiliary_driver`

**الغرض:** زي الـ driver العادي، بس مخصص للـ auxiliary bus. هو اللي "بيعرف" كيف يتعامل مع الـ auxiliary_device لما يُكتشف.

```c
struct auxiliary_driver {
    /* callbacks إجبارية/اختيارية */
    int  (*probe)(struct auxiliary_device *auxdev,
                  const struct auxiliary_device_id *id);
    void (*remove)(struct auxiliary_device *auxdev);
    void (*shutdown)(struct auxiliary_device *auxdev);
    int  (*suspend)(struct auxiliary_device *auxdev, pm_message_t state);
    int  (*resume)(struct auxiliary_device *auxdev);

    const char *name;                          /* اسم الـ driver */
    struct device_driver driver;               /* الـ driver الأصل في kernel model */
    const struct auxiliary_device_id *id_table;/* جدول الأجهزة المدعومة */
};
```

**شرح الحقول:**

| الحقل | الوظيفة |
|-------|---------|
| `probe` | يُستدعى لما kernel يجد device مطابق — هنا الـ driver يبدأ يشتغل |
| `remove` | يُستدعى لما الـ device يُحذف من الـ bus |
| `shutdown` | عند إيقاف تشغيل النظام، لإسكات الـ hardware |
| `suspend` | لوضع الـ device في وضع النوم (power management) |
| `resume` | لإيقاظ الـ device من النوم |
| `driver` | الـ `struct device_driver` المضمّن — يحتوي على الـ bus pointer وغيره |
| `id_table` | مصفوفة من `struct auxiliary_device_id` تنتهي بـ entry فارغة |

**الربط بـ structs تانية:**
- **يحتوي** على `struct device_driver` (embedded)
- **يُشير** إلى `struct auxiliary_device_id[]` (جدول خارجي)
- الـ `probe()` يستقبل pointer لـ `struct auxiliary_device` وـ`struct auxiliary_device_id`

---

#### 3. `struct auxiliary_device_id`

**الغرض:** صف واحد في جدول مطابقة الـ driver. الـ kernel يقارن اسم الـ device بهذا الجدول ليقرر أي driver يُربط.

```c
/* في mod_devicetable.h */
#define AUXILIARY_NAME_SIZE 40

struct auxiliary_device_id {
    char name[AUXILIARY_NAME_SIZE];  /* "module_name.device_name" مثلاً "foo_mod.foo_dev" */
    kernel_ulong_t driver_data;      /* بيانات خاصة بالـ driver، مثلاً index لـ feature table */
};
```

| الحقل | الشرح |
|-------|-------|
| `name` | الاسم الكامل للمطابقة: `KBUILD_MODNAME` + `.` + `auxiliary_device.name` |
| `driver_data` | رقم اختياري يستخدمه الـ driver لتمييز أنواع مختلفة من نفس الـ device |

---

### مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────┐
│                 auxiliary_device                     │
│  ┌──────────────────────────────────────────────┐   │
│  │           struct device  (dev)               │   │
│  │  - parent  ──────────────────────────────────┼───┼──► struct device (parent PCI/platform dev)
│  │  - driver  ──────────────────────────────────┼───┼──► struct device_driver
│  │  - bus     ──────────────────────────────────┼───┼──► struct bus_type (auxiliary_bus)
│  │  - release (fn pointer)                      │   │
│  │  - kobj    ──────────────────────────────────┼───┼──► kobject (sysfs node)
│  └──────────────────────────────────────────────┘   │
│  - name  (const char *)                              │
│  - id    (u32)                                       │
│  - sysfs.irqs   (xarray)                            │
│  - sysfs.lock   (mutex)                             │
│  - sysfs.irq_dir_exists (bool)                      │
└─────────────────────────────────────────────────────┘
         ▲
         │  container_of(dev, struct auxiliary_device, dev)
         │  يعني: من الـ dev داخله، ارجع للـ auxiliary_device الكاملة
         │
┌─────────────────────────────────────────────────────┐
│               auxiliary_driver                       │
│  ┌──────────────────────────────────────────────┐   │
│  │       struct device_driver (driver)          │   │
│  │  - name                                      │   │
│  │  - bus  ─────────────────────────────────────┼───┼──► auxiliary_bus
│  │  - owner (THIS_MODULE)                       │   │
│  └──────────────────────────────────────────────┘   │
│  - probe    (fn ptr) ────────────────────────────────┼──► receives auxiliary_device*
│  - remove   (fn ptr)                                 │
│  - suspend  (fn ptr)                                 │
│  - resume   (fn ptr)                                 │
│  - shutdown (fn ptr)                                 │
│  - id_table ─────────────────────────────────────────┼──► auxiliary_device_id[]
└─────────────────────────────────────────────────────┘

┌───────────────────────────────┐
│    auxiliary_device_id[]      │
│  [0]: name="foo_mod.foo_dev"  │
│       driver_data=0x1         │
│  [1]: name="foo_mod.bar_dev"  │
│       driver_data=0x2         │
│  [2]: { 0 }  ← نهاية الجدول  │
└───────────────────────────────┘
```

---

### مخطط الـ container_of — كيف الـ Driver يصل لبياناته

تخيّل إن الـ driver عنده struct خاص `my_parent_obj` يحتوي على `auxiliary_device` + بيانات إضافية:

```
ذاكرة الـ registering driver:
┌───────────────────────────────────┐
│        my_parent_obj              │
│  ┌─────────────────────────────┐  │
│  │   auxiliary_device auxdev   │  │  ◄── الـ kernel يعطي pointer لهنا لـ probe()
│  │     struct device dev       │  │
│  └─────────────────────────────┘  │
│  struct shared_obj *shared;  ──────┼──► بيانات مشتركة مع auxiliary_driver
│  int some_private_field;           │
└───────────────────────────────────┘

في probe():
   my_parent_obj *parent = container_of(auxdev, my_parent_obj, auxdev);
   parent->shared  ← وصل للبيانات المشتركة!
```

---

### دورة حياة الـ Auxiliary Device — من الميلاد للموت

```
══════════════════════════════════════════════════════════════════
                  REGISTERING DRIVER (Parent Driver)
══════════════════════════════════════════════════════════════════

[1] تخصيص الذاكرة
    kzalloc(sizeof(my_parent_obj))
         │
         ▼
[2] تعبئة الحقول الإجبارية
    auxdev->name        = "foo_dev"
    auxdev->id          = unique_id
    auxdev->dev.release = my_release_fn    ← إجباري!
    auxdev->dev.parent  = &parent_pci_dev  ← إجباري!
         │
         ▼
[3] auxiliary_device_init(auxdev)
    ┌─────────────────────────────────────────┐
    │  • يتحقق إن release مش NULL             │
    │  • يتحقق إن parent مش NULL             │
    │  • يستدعي device_initialize()          │
    │    (يهيئ kobj، refcount=1، إلخ)        │
    └─────────────────────────────────────────┘
         │ نجح؟
         ▼
[4] auxiliary_device_add(auxdev)   [= __auxiliary_device_add(auxdev, KBUILD_MODNAME)]
    ┌─────────────────────────────────────────┐
    │  • يبني الاسم: "foo_mod.foo_dev.ID"    │
    │  • يستدعي device_add()                 │
    │  • الـ bus يبحث عن driver مطابق        │
    │  • إذا وجد → يستدعي driver->probe()   │
    └─────────────────────────────────────────┘
         │
         ▼
    ══ الـ DEVICE نشط ومتصل بـ driver ══

         │ (عند الإزالة)
         ▼
[5] auxiliary_device_delete(auxdev)
    └── device_del(&auxdev->dev)
        • ينزع الـ device من الـ bus
        • يستدعي driver->remove()
        • يُزيل sysfs entries

         │
         ▼
[6] auxiliary_device_uninit(auxdev)
    ┌─────────────────────────────────────────┐
    │  • mutex_destroy(&auxdev->sysfs.lock)   │
    │  • put_device(&auxdev->dev)             │
    │    (ينقص الـ refcount)                  │
    └─────────────────────────────────────────┘
         │ refcount == 0 ؟
         ▼
[7] يُستدعى auxdev->dev.release() تلقائيًا
    └── kfree(my_parent_obj)  ← الـ driver يحرر ذاكرته هنا
```

---

### دورة حياة الـ Auxiliary Driver — التسجيل والإلغاء

```
══════════════════════════════════════════════════════════════════
                    AUXILIARY DRIVER (Consumer Driver)
══════════════════════════════════════════════════════════════════

[1] تعريف الـ driver وجدول الـ IDs
    static const struct auxiliary_device_id my_id_table[] = {
        { .name = "foo_mod.foo_dev" },
        { }
    };
    MODULE_DEVICE_TABLE(auxiliary, my_id_table);

    static struct auxiliary_driver my_aux_drv = {
        .name     = "my_aux_driver",
        .id_table = my_id_table,
        .probe    = my_probe,
        .remove   = my_remove,
    };
         │
         ▼
[2] module_init:
    auxiliary_driver_register(&my_aux_drv)
    [= __auxiliary_driver_register(drv, THIS_MODULE, KBUILD_MODNAME)]
    ┌─────────────────────────────────────────┐
    │  • يضبط drv->driver.bus = &auxiliary_bus│
    │  • يستدعي driver_register()            │
    │  • الـ bus يبحث عن devices مطابقة      │
    │  • لكل device مطابق → probe()          │
    └─────────────────────────────────────────┘
         │
         ▼
    ══ الـ DRIVER نشط ويخدم الأجهزة ══

         │ (عند إزالة الـ module)
         ▼
[3] module_exit:
    auxiliary_driver_unregister(&my_aux_drv)
    └── driver_unregister()
        • لكل device مرتبط → remove()
        • يُزيل الـ driver من الـ bus
```

---

### مخطط تدفق الاستدعاءات — Call Flow

#### سيناريو 1: تسجيل device جديد

```
registering_driver_probe()
  │
  ├─► kzalloc(sizeof(my_obj))
  │
  ├─► [تعبئة auxdev->name, id, dev.release, dev.parent]
  │
  ├─► auxiliary_device_init(auxdev)
  │     └─► device_initialize(&auxdev->dev)
  │           └─► kobject_init()
  │               refcount = 1
  │
  └─► auxiliary_device_add(auxdev)
        └─► __auxiliary_device_add(auxdev, "foo_mod")
              └─► dev_set_name(&auxdev->dev, "%s.%s.%d",
              │                modname, auxdev->name, auxdev->id)
              └─► device_add(&auxdev->dev)
                    ├─► kobject_add()  ← يظهر في sysfs
                    ├─► bus_probe_device()
                    │     └─► auxiliary_bus_match()
                    │           └─► يقارن اسم الـ device بـ id_table
                    │               لو مطابق ↓
                    └─► auxiliary_driver->probe(auxdev, id)
                          └─► [الـ consumer driver يبدأ شغله]
```

#### سيناريو 2: إضافة IRQ لـ sysfs

```
auxiliary_device_sysfs_irq_add(auxdev, irq_num)
  │
  ├─► mutex_lock(&auxdev->sysfs.lock)
  │
  ├─► [تحقق من irq_dir_exists، أنشئه لو مش موجود]
  │
  ├─► xa_store(&auxdev->sysfs.irqs, irq_num, ...)
  │     └─► يحفظ الـ IRQ في الـ xarray
  │
  └─► mutex_unlock(&auxdev->sysfs.lock)
```

#### سيناريو 3: الـ probe يصل لبيانات الـ parent

```
my_aux_driver_probe(auxdev, id)
  │
  ├─► to_auxiliary_dev(dev)  [لو بدأت من struct device *]
  │     └─► container_of(dev, struct auxiliary_device, dev)
  │
  ├─► container_of(auxdev, struct my_parent_obj, auxdev)
  │     └─► وصل للـ parent object الكامل
  │
  ├─► parent->shared_data  ← يقرأ البيانات المشتركة
  │
  └─► auxiliary_set_drvdata(auxdev, my_private_data)
        └─► dev_set_drvdata(&auxdev->dev, my_private_data)
```

#### سيناريو 4: module_auxiliary_driver — الطريق السريع

```
module_auxiliary_driver(my_drv)
  يُولّد تلقائيًا:
  ┌────────────────────────────────────────┐
  │ module_init(my_drv_init) {            │
  │   auxiliary_driver_register(&my_drv); │
  │ }                                     │
  │ module_exit(my_drv_exit) {            │
  │   auxiliary_driver_unregister(&my_drv)│
  │ }                                     │
  └────────────────────────────────────────┘
```

---

### استراتيجية الـ Locking

الـ auxiliary_bus بسيط نسبيًا من ناحية الـ locking، والجدول التالي يوضح من يحمي ماذا:

| الـ Lock | النوع | يحمي ماذا؟ | من يمسكه؟ |
|---------|-------|------------|----------|
| `auxdev->sysfs.lock` | `struct mutex` | إنشاء وحذف مجلد `irqs` في sysfs وعناصره | `auxiliary_device_sysfs_irq_add/remove` |
| `auxdev->dev.mutex` | `struct mutex` (في device) | استدعاءات الـ driver (probe/remove) | الـ device core تلقائيًا |
| الـ xarray internal lock | spinlock داخلي | العمليات على `auxdev->sysfs.irqs` | `xa_store` / `xa_erase` داخليًا |
| الـ kobject refcount | `atomic_t` | دورة حياة الـ device | `get_device` / `put_device` |

#### ترتيب الـ Locking (Lock Ordering)

```
يجب دائمًا الإمساك بالـ locks بهذا الترتيب لتجنب deadlock:

1. dev->mutex          (الـ device core mutex — الأعلى)
      │
      ▼
2. sysfs.lock          (mutex الـ sysfs الخاص بالـ auxiliary)
      │
      ▼
3. xarray internal     (spinlock داخلي — الأسفل، يُمسك آخرًا)
```

#### قاعدة مهمة في الـ Memory Management:

```
لا يوجد lock في هذا الملف يحمي حذف ذاكرة الـ auxiliary_device نفسها!
الحماية هنا بـ reference counting:

  get_device()  → refcount++  (driver core يعمل هذا تلقائيًا)
  put_device()  → refcount--
                  لو وصل 0 → release() تُستدعى تلقائيًا
                             ولا يجب الوصول للذاكرة بعدها
```

#### ملاحظة على الـ sysfs.lock بالتفصيل:

الـ `mutex` داخل `sysfs` هدفه حماية **race condition** واحدة محددة:

```
Thread A: auxiliary_device_sysfs_irq_add(auxdev, irq=5)
Thread B: auxiliary_device_sysfs_irq_add(auxdev, irq=7)

بدون lock:
  A يتحقق: irq_dir_exists == false → يبدأ ينشئ المجلد
  B يتحقق: irq_dir_exists == false → يبدأ ينشئ المجلد أيضًا!
  النتيجة: محاولة إنشاء مجلد مرتين → خطأ

مع sysfs.lock:
  A يمسك الـ lock، يُنشئ المجلد، يضبط irq_dir_exists=true، يُطلق الـ lock
  B يمسك الـ lock، يرى irq_dir_exists=true → يتجاوز الخطوة، يضيف IRQ فقط
```
## Phase 4: شرح الـ Functions

### جدول ملخص الـ Functions

| Function | Type | Purpose |
|----------|------|---------|
| `auxiliary_get_drvdata()` | static inline | يجيب الـ private data المخزن في الـ auxiliary device |
| `auxiliary_set_drvdata()` | static inline | يحفظ الـ private data في الـ auxiliary device |
| `to_auxiliary_dev()` | static inline | يحوّل مؤشر `struct device` لـ `struct auxiliary_device` |
| `to_auxiliary_drv()` | static inline | يحوّل مؤشر `struct device_driver` لـ `struct auxiliary_driver` |
| `auxiliary_device_init()` | EXPORT | يهيئ الـ auxiliary device ويتحقق منه قبل تسجيله |
| `__auxiliary_device_add()` | EXPORT | يضيف الـ auxiliary device للـ bus (يُستخدم عبر الـ macro) |
| `auxiliary_device_add()` | macro | wrapper للـ `__auxiliary_device_add` يحقن `KBUILD_MODNAME` تلقائياً |
| `auxiliary_device_uninit()` | static inline | يدمر الـ mutex ويحرر الـ reference على الـ device |
| `auxiliary_device_delete()` | static inline | يحذف الـ device من الـ bus |
| `auxiliary_device_sysfs_irq_add()` | EXPORT (CONFIG_SYSFS) | يضيف IRQ entry في الـ sysfs للـ device |
| `auxiliary_device_sysfs_irq_remove()` | EXPORT (CONFIG_SYSFS) | يحذف IRQ entry من الـ sysfs للـ device |
| `__auxiliary_driver_register()` | EXPORT | يسجّل الـ auxiliary driver في الـ bus |
| `auxiliary_driver_register()` | macro | wrapper يحقن `THIS_MODULE` و `KBUILD_MODNAME` |
| `auxiliary_driver_unregister()` | EXPORT | يلغي تسجيل الـ driver من الـ bus |
| `auxiliary_device_create()` | EXPORT | ينشئ auxiliary device كامل بخطوة واحدة |
| `auxiliary_device_destroy()` | EXPORT | يدمر auxiliary device تم إنشاؤه بـ `auxiliary_device_create()` |
| `__devm_auxiliary_device_create()` | EXPORT | نسخة managed من `auxiliary_device_create()` |
| `devm_auxiliary_device_create()` | macro | wrapper للنسخة الـ managed يحقن `KBUILD_MODNAME` |
| `module_auxiliary_driver()` | macro | boilerplate يختصر `module_init` + `module_exit` للـ driver |

---

### المجموعة الأولى: الـ Helper / Conversion Functions

هذه دوال بسيطة جداً (inline) وظيفتها التنقل بين الـ structs المختلفة أو الوصول للـ data المخزن. فكرها زي "مفاتيح" تفتح لك الباب من جهة لجهة ثانية.

---

#### `auxiliary_get_drvdata()`

```c
static inline void *auxiliary_get_drvdata(struct auxiliary_device *auxdev)
{
    return dev_get_drvdata(&auxdev->dev);
}
```

**ما الذي تفعله؟**
تجيب الـ pointer اللي خزّنه الـ driver في الـ device باستخدام `dev_get_drvdata()`. الـ auxiliary driver يحتاج يخزن data خاصة فيه (مثلاً struct بيانات داخلية)، وهذه الدالة تجيبها له.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device` المراد استرجاع البيانات منه.

**القيمة المُعادة:**
- `void *` — مؤشر للـ private data المخزّن مسبقاً، أو `NULL` إن لم يُخزَّن شيء.

**تفاصيل مهمة:**
- لا تحتاج أي locking لأنها مجرد قراءة من field في الـ struct.
- الدالة تصل للـ `dev` المضمّن داخل `auxdev` وتمرره لـ `dev_get_drvdata()`.

**من يستدعيها؟**
الـ auxiliary driver في أي مكان يحتاج فيه للوصول لـ private data خاصة به، عادةً في `probe`, `remove`, أو أي callback آخر.

---

#### `auxiliary_set_drvdata()`

```c
static inline void auxiliary_set_drvdata(struct auxiliary_device *auxdev, void *data)
{
    dev_set_drvdata(&auxdev->dev, data);
}
```

**ما الذي تفعله؟**
تخزّن مؤشراً لـ private data داخل الـ auxiliary device. زي إنك تضع ورقة ملاحظات داخل ملف — في أي وقت بعدين تقدر تجيبها بـ `auxiliary_get_drvdata()`.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device`.
- **`data`**: الـ pointer اللي تريد تخزينه (عادةً struct خاص بالـ driver).

**القيمة المُعادة:**
- لا تُعيد شيئاً (`void`).

**من يستدعيها؟**
عادةً في `probe()` الـ auxiliary driver مباشرةً بعد تهيئة البيانات الداخلية.

---

#### `to_auxiliary_dev()`

```c
static inline struct auxiliary_device *to_auxiliary_dev(struct device *dev)
{
    return container_of(dev, struct auxiliary_device, dev);
}
```

**ما الذي تفعله؟**
تحوّل مؤشر `struct device` العام إلى مؤشر `struct auxiliary_device` الخاص. فكرة `container_of` هي إنك عندك عنوان غرفة في الشقة، وتريد عنوان الشقة الكاملة.

**المعاملات:**
- **`dev`**: مؤشر `struct device *` عادةً جاي من الـ bus core أو callback.

**القيمة المُعادة:**
- `struct auxiliary_device *` — المؤشر الكامل للـ auxiliary device الذي يحتوي على هذا الـ `dev`.

**تفاصيل مهمة:**
- تعتمد على حقيقة أن `struct device dev` هو أول field في `struct auxiliary_device`، فالحسابات الحسابية للعنوان صحيحة.
- هذه الدالة آمنة فقط إذا كان الـ `dev` فعلاً جزءاً من `struct auxiliary_device`، وإلا ستنتج سلوكاً غير محدد.

**من يستدعيها؟**
الـ bus core والـ drivers عندما يستقبلون `struct device *` في الـ callbacks ويحتاجون الـ auxiliary_device الكامل.

---

#### `to_auxiliary_drv()`

```c
static inline const struct auxiliary_driver *to_auxiliary_drv(const struct device_driver *drv)
{
    return container_of(drv, struct auxiliary_driver, driver);
}
```

**ما الذي تفعله؟**
نفس فكرة `to_auxiliary_dev()` ولكن للـ driver — تحوّل `struct device_driver *` العام إلى `struct auxiliary_driver *` الخاص.

**المعاملات:**
- **`drv`**: مؤشر `const struct device_driver *` عادةً جاي من الـ bus core.

**القيمة المُعادة:**
- `const struct auxiliary_driver *` — المؤشر الكامل للـ auxiliary driver.

**من يستدعيها؟**
الـ bus core بشكل رئيسي عند الـ matching وتنفيذ الـ callbacks.

---

### المجموعة الثانية: تسجيل وإضافة الـ Device (Registration)

هذه الدوال هي قلب الـ auxiliary bus من جانب الـ device. تسجيل الـ device يمر بثلاث خطوات: **init → add → (use) → delete → uninit**.

---

#### `auxiliary_device_init()`

```c
int auxiliary_device_init(struct auxiliary_device *auxdev);
```

**ما الذي تفعله؟**
هي الخطوة الأولى في تسجيل الـ auxiliary device. تتحقق من أن الـ struct ممتلئ صح (مثل وجود `name` و `dev.release` و `dev.parent`)، ثم تستدعي `device_initialize()` لتجهيز الـ device للإضافة للـ bus. كذلك تهيئ الـ `sysfs.lock` الـ mutex وتهيئ الـ `sysfs.irqs` الـ xarray.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device` المراد تهيئته. يجب أن يكون قد مُلئت فيه: `name`, `id`, `dev.release`, `dev.parent`.

**القيمة المُعادة:**
- `0` في حالة النجاح.
- قيمة سالبة (error code) في حالة الفشل، مثل `-EINVAL` إن كانت الـ fields ناقصة.

**تفاصيل مهمة:**
- بعد نجاح هذه الدالة، أي خطأ لاحق يستوجب استدعاء `auxiliary_device_uninit()` لتنظيف الـ resources.
- تهيئ الـ `mutex` و `xarray` الداخلية للـ sysfs.

**pseudocode:**
```
auxiliary_device_init(auxdev):
    if auxdev->name == NULL: return -EINVAL
    if auxdev->dev.release == NULL && auxdev->dev.type->release == NULL: return -EINVAL
    if auxdev->dev.parent == NULL: return -EINVAL
    mutex_init(&auxdev->sysfs.lock)
    xa_init(&auxdev->sysfs.irqs)
    device_initialize(&auxdev->dev)
    return 0
```

**من يستدعيها؟**
الـ registering driver (الـ parent driver) في خطوة التسجيل الثانية.

---

#### `__auxiliary_device_add()` و `auxiliary_device_add()`

```c
int __auxiliary_device_add(struct auxiliary_device *auxdev, const char *modname);
#define auxiliary_device_add(auxdev) __auxiliary_device_add(auxdev, KBUILD_MODNAME)
```

**ما الذي تفعله؟**
هي الخطوة الثالثة والأخيرة في تسجيل الـ device. تبني اسم الـ device الكامل على شكل `modname.devname.id`، ثم تستدعي `device_add()` لإضافة الـ device للـ bus فعلياً. بعد هذه الخطوة يبدأ الـ bus في البحث عن driver مناسب.

الـ macro `auxiliary_device_add()` هو ما يستخدمه الـ developer عادةً لأنه يحقن `KBUILD_MODNAME` تلقائياً بدل ما تكتبه يدوياً.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device` الذي تمت تهيئته بـ `auxiliary_device_init()`.
- **`modname`**: اسم الـ module المسجِّل (يُحقن تلقائياً عبر الـ macro).

**القيمة المُعادة:**
- `0` في حالة النجاح.
- قيمة سالبة في حالة الفشل (مثل `-EEXIST` إن كان الاسم مكرراً).

**تفاصيل مهمة:**
- في حالة الفشل يجب استدعاء `auxiliary_device_uninit()` للتنظيف.
- يمكن أن يتسبب هذا في استدعاء فوري لـ `probe()` الـ driver إذا كان driver مناسب مسجلاً مسبقاً.

**من يستدعيها؟**
الـ registering driver دائماً عبر الـ macro `auxiliary_device_add()`.

---

#### `auxiliary_device_uninit()`

```c
static inline void auxiliary_device_uninit(struct auxiliary_device *auxdev)
{
    mutex_destroy(&auxdev->sysfs.lock);
    put_device(&auxdev->dev);
}
```

**ما الذي تفعله؟**
تعكس ما فعلته `auxiliary_device_init()`. تدمر الـ `sysfs.lock` mutex وتستدعي `put_device()` لتحرير الـ reference على الـ device. إذا وصل عدد الـ references للصفر تُستدعى دالة `release()` التي حددها الـ registering driver لتحرير الذاكرة.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device` المراد إلغاء تهيئته.

**القيمة المُعادة:**
- لا تُعيد شيئاً (`void`).

**تفاصيل مهمة:**
- هذه الدالة لا تحرر الذاكرة مباشرةً — بل تحرر الـ reference فقط. تحرير الذاكرة الفعلي يحدث في `dev.release()`.
- يجب استدعاؤها بعد `auxiliary_device_delete()` وليس قبلها.
- إذا فشلت `auxiliary_device_init()` أو `auxiliary_device_add()`، يجب أيضاً استدعاء هذه الدالة.

**من يستدعيها؟**
الـ registering driver عند إلغاء تسجيل الـ device، أو في مسار الـ error handling.

---

#### `auxiliary_device_delete()`

```c
static inline void auxiliary_device_delete(struct auxiliary_device *auxdev)
{
    device_del(&auxdev->dev);
}
```

**ما الذي تفعله؟**
تحذف الـ device من الـ bus. هذا يتسبب في استدعاء `remove()` الـ driver المرتبط بالـ device، ويوقف أي عمليات جديدة عليه. لكنها لا تحرر الذاكرة — هذا دور `auxiliary_device_uninit()`.

فكّر فيها كـ "فصل الكهرباء" — الجهاز ما زال موجوداً لكنه لم يعد متصلاً بالـ bus.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device` المراد حذفه من الـ bus.

**القيمة المُعادة:**
- لا تُعيد شيئاً (`void`).

**تفاصيل مهمة:**
- يجب استدعاؤها قبل `auxiliary_device_uninit()`.
- الـ registering driver يجب أن يكمل هذه الخطوة قبل أن ينتهي من دالة `remove()` الخاصة به.

**من يستدعيها؟**
الـ registering driver في مرحلة التنظيف، عادةً في `driver.remove()` أو عبر `devm_add_action_or_reset()`.

---

### المجموعة الثالثة: إدارة الـ IRQs في الـ sysfs

هذه الدوال تدير ظهور الـ IRQs المرتبطة بالـ device في الـ `/sys` filesystem. تعتمد على وجود `CONFIG_SYSFS`؛ إذا لم يكن مفعّلاً تصبح الدوال stubs لا تفعل شيئاً.

---

#### `auxiliary_device_sysfs_irq_add()`

```c
int auxiliary_device_sysfs_irq_add(struct auxiliary_device *auxdev, int irq);
```

**ما الذي تفعله؟**
تضيف entry لـ IRQ رقم معين في الـ sysfs تحت مسار الـ device. هذا يجعل الـ IRQs مرئية للـ userspace تحت `/sys/bus/auxiliary/devices/<device>/irqs/`. تستخدم الـ `sysfs.irqs` xarray لتخزين الـ IRQ indices، وتنشئ مجلد `irqs/` عند أول استدعاء.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device`.
- **`irq`**: رقم الـ IRQ المراد إضافته للـ sysfs.

**القيمة المُعادة:**
- `0` في حالة النجاح.
- قيمة سالبة في حالة فشل إنشاء الـ sysfs entry.

**تفاصيل مهمة:**
- تستخدم `sysfs.lock` mutex لضمان thread-safety عند إنشاء مجلد `irqs/`.
- الـ field `sysfs.irq_dir_exists` يتتبع إذا كان المجلد قد أُنشئ.
- إذا لم يكن `CONFIG_SYSFS` مفعّلاً، تُعيد `0` مباشرةً دون فعل شيء.

**من يستدعيها؟**
الـ registering driver أو الـ auxiliary driver عند تسجيل IRQs خاصة بالـ device.

---

#### `auxiliary_device_sysfs_irq_remove()`

```c
void auxiliary_device_sysfs_irq_remove(struct auxiliary_device *auxdev, int irq);
```

**ما الذي تفعله؟**
تعكس ما فعلته `auxiliary_device_sysfs_irq_add()` — تحذف الـ IRQ entry من الـ sysfs وتزيله من الـ `sysfs.irqs` xarray.

**المعاملات:**
- **`auxdev`**: مؤشر للـ `struct auxiliary_device`.
- **`irq`**: رقم الـ IRQ المراد حذفه.

**القيمة المُعادة:**
- لا تُعيد شيئاً (`void`).

**تفاصيل مهمة:**
- إذا لم يكن `CONFIG_SYSFS` مفعّلاً، تصبح دالة فارغة (empty stub).

**من يستدعيها؟**
الـ driver عند تنظيف الـ IRQs، قبل حذف الـ device.

---

### المجموعة الرابعة: تسجيل الـ Driver (Driver Registration)

---

#### `__auxiliary_driver_register()` و `auxiliary_driver_register()`

```c
int __auxiliary_driver_register(struct auxiliary_driver *auxdrv,
                                struct module *owner,
                                const char *modname);
#define auxiliary_driver_register(auxdrv) \
    __auxiliary_driver_register(auxdrv, THIS_MODULE, KBUILD_MODNAME)
```

**ما الذي تفعله؟**
تسجّل الـ auxiliary driver في الـ bus subsystem. تضع الـ `name` في الـ driver إن لم تكن محددة (تأخذها من `modname`)، ثم تستدعي `driver_register()` لتسجيله. بعد التسجيل يبدأ الـ bus في مقارنة الـ driver بالـ devices الموجودة عبر `id_table`.

الـ macro يختصر عليك كتابة `THIS_MODULE` و `KBUILD_MODNAME` يدوياً.

**المعاملات:**
- **`auxdrv`**: مؤشر للـ `struct auxiliary_driver` المُهيّأ بالكامل مع `id_table` و `probe` و `remove`.
- **`owner`**: الـ module المالك (يُحقن تلقائياً عبر الـ macro كـ `THIS_MODULE`).
- **`modname`**: اسم الـ module (يُحقن تلقائياً عبر الـ macro كـ `KBUILD_MODNAME`).

**القيمة المُعادة:**
- `0` في حالة النجاح.
- قيمة سالبة في حالة الفشل.

**تفاصيل مهمة:**
- يجب أن يكون `id_table` ممتلئاً قبل الاستدعاء، وإلا لن يتم الـ matching مع أي device.
- التسجيل قد يتسبب فوراً في استدعاء `probe()` لـ devices موجودة مسبقاً على الـ bus.

**pseudocode:**
```
__auxiliary_driver_register(auxdrv, owner, modname):
    if auxdrv->name == NULL:
        auxdrv->name = modname
    auxdrv->driver.owner = owner
    auxdrv->driver.bus = &auxiliary_bus_type
    return driver_register(&auxdrv->driver)
```

**من يستدعيها؟**
الـ auxiliary driver في `module_init()` أو عبر الـ macro `module_auxiliary_driver()`.

---

#### `auxiliary_driver_unregister()`

```c
void auxiliary_driver_unregister(struct auxiliary_driver *auxdrv);
```

**ما الذي تفعله؟**
تلغي تسجيل الـ auxiliary driver من الـ bus. تستدعي `driver_unregister()` مما يتسبب في استدعاء `remove()` لكل device مرتبط بهذا الـ driver.

**المعاملات:**
- **`auxdrv`**: مؤشر للـ `struct auxiliary_driver` المراد إلغاء تسجيله.

**القيمة المُعادة:**
- لا تُعيد شيئاً (`void`).

**من يستدعيها؟**
الـ auxiliary driver في `module_exit()` أو عبر الـ macro `module_auxiliary_driver()`.

---

### المجموعة الخامسة: الإنشاء الآلي للـ Device (Managed Creation)

هذه الدوال تجمع خطوات الـ init + add في استدعاء واحد، وتوفر نسخة **managed** تُنظّف نفسها تلقائياً باستخدام `devres`.

---

#### `auxiliary_device_create()`

```c
struct auxiliary_device *auxiliary_device_create(struct device *dev,
                                                 const char *modname,
                                                 const char *devname,
                                                 void *platform_data,
                                                 int id);
```

**ما الذي تفعله؟**
تنشئ auxiliary device كامل بخطوة واحدة بدلاً من الثلاث خطوات اليدوية. تخصص الذاكرة، تملأ الـ fields، تستدعي `auxiliary_device_init()` ثم `__auxiliary_device_add()`. هي الطريقة الأسهل لإنشاء الـ device إذا لم تحتاج struct مخصص.

**المعاملات:**
- **`dev`**: الـ parent device (الـ device الرئيسي الذي سيُولد هذا الـ auxiliary device).
- **`modname`**: اسم الـ module المسجِّل.
- **`devname`**: اسم الـ auxiliary device الفرعي.
- **`platform_data`**: بيانات مخصصة تُمرّر للـ driver عبر `dev.platform_data`.
- **`id`**: معرّف فريد عند وجود أكثر من device بنفس الاسم.

**القيمة المُعادة:**
- مؤشر للـ `struct auxiliary_device` المُنشأ في حالة النجاح.
- `NULL` أو `ERR_PTR()` في حالة الفشل.

**تفاصيل مهمة:**
- الـ device المُنشأ يجب تدميره بـ `auxiliary_device_destroy()` وليس يدوياً.
- الـ `platform_data` يبقى مسؤولية الـ caller عن تحرير ذاكرته.

**pseudocode:**
```
auxiliary_device_create(dev, modname, devname, platform_data, id):
    auxdev = kzalloc(sizeof(*auxdev), GFP_KERNEL)
    if !auxdev: return NULL

    auxdev->name = devname
    auxdev->id = id
    auxdev->dev.parent = dev
    auxdev->dev.platform_data = platform_data
    auxdev->dev.release = auxiliary_device_release  // دالة داخلية تحرر الذاكرة

    ret = auxiliary_device_init(auxdev)
    if ret: goto err_init

    ret = __auxiliary_device_add(auxdev, modname)
    if ret: goto err_add

    return auxdev
```

**من يستدعيها؟**
الـ parent driver عندما يريد إنشاء auxiliary device بسرعة دون struct مخصص.

---

#### `auxiliary_device_destroy()`

```c
void auxiliary_device_destroy(void *auxdev);
```

**ما الذي تفعله؟**
تدمر auxiliary device تم إنشاؤه بـ `auxiliary_device_create()`. تستدعي `auxiliary_device_delete()` ثم `auxiliary_device_uninit()` بالترتيب الصحيح.

**ملاحظة:** الـ parameter نوعه `void *` وليس `struct auxiliary_device *` لأن هذه الدالة تُستخدم كـ callback مع `devm` وحالات أخرى.

**المعاملات:**
- **`auxdev`**: مؤشر `void *` للـ `struct auxiliary_device` المراد تدميره.

**القيمة المُعادة:**
- لا تُعيد شيئاً (`void`).

**من يستدعيها؟**
الـ parent driver عند التنظيف، أو يُستدعى تلقائياً بواسطة الـ `devres` في النسخة الـ managed.

---

#### `__devm_auxiliary_device_create()` و `devm_auxiliary_device_create()`

```c
struct auxiliary_device *__devm_auxiliary_device_create(struct device *dev,
                                                        const char *modname,
                                                        const char *devname,
                                                        void *platform_data,
                                                        int id);
#define devm_auxiliary_device_create(dev, devname, platform_data) \
    __devm_auxiliary_device_create(dev, KBUILD_MODNAME, devname, \
                                   platform_data, 0)
```

**ما الذي تفعله؟**
نسخة **managed** (devm) من `auxiliary_device_create()`. تربط عمر الـ auxiliary device بعمر الـ parent device — عندما يُحرر الـ parent device تُستدعى `auxiliary_device_destroy()` تلقائياً دون تدخل يدوي.

فكر فيها كـ "ضامن إيجار" — لما يخرج المالك (parent device) تُنهى جميع عقود الإيجار الفرعية (auxiliary devices) تلقائياً.

الـ macro يختصر `modname` (يستخدم `KBUILD_MODNAME`) و `id` (يضعه `0` افتراضياً).

**المعاملات:**
- **`dev`**: الـ parent device — إليه سيُربط عمر الـ auxiliary device.
- **`modname`**: اسم الـ module (يُحقن تلقائياً).
- **`devname`**: اسم الـ auxiliary device.
- **`platform_data`**: بيانات مخصصة للـ driver.
- **`id`**: معرّف الـ device (افتراضياً `0` في الـ macro).

**القيمة المُعادة:**
- مؤشر للـ `struct auxiliary_device` في حالة النجاح.
- `ERR_PTR()` في حالة الفشل.

**تفاصيل مهمة:**
- تستخدم `devm_add_action_or_reset()` داخلياً لتسجيل `auxiliary_device_destroy()` كـ cleanup action.
- لا تحتاج لاستدعاء `auxiliary_device_destroy()` يدوياً إذا استخدمت هذه الدالة.
- الـ `id` مثبت على `0` في الـ macro — استخدم `__devm_auxiliary_device_create()` مباشرةً إذا احتجت id مختلف.

**pseudocode:**
```
__devm_auxiliary_device_create(dev, modname, devname, platform_data, id):
    auxdev = auxiliary_device_create(dev, modname, devname, platform_data, id)
    if IS_ERR(auxdev): return auxdev

    ret = devm_add_action_or_reset(dev, auxiliary_device_destroy, auxdev)
    if ret:
        auxiliary_device_destroy(auxdev)
        return ERR_PTR(ret)

    return auxdev
```

**من يستدعيها؟**
الـ parent driver عادةً في `probe()` عندما يريد إنشاء auxiliary device بدون قلق من التنظيف اليدوي.

---

### المجموعة السادسة: الـ Macros المختصرة

---

#### `module_auxiliary_driver()`

```c
#define module_auxiliary_driver(__auxiliary_driver) \
    module_driver(__auxiliary_driver, auxiliary_driver_register, \
                  auxiliary_driver_unregister)
```

**ما الذي تفعله؟**
macro يختصر الـ boilerplate الروتيني لـ driver module. بدلاً من كتابة:

```c
static int __init my_drv_init(void)
{
    return auxiliary_driver_register(&my_drv);
}

static void __exit my_drv_exit(void)
{
    auxiliary_driver_unregister(&my_drv);
}

module_init(my_drv_init);
module_exit(my_drv_exit);
```

تكتب فقط:

```c
module_auxiliary_driver(my_drv);
```

**المعاملات:**
- **`__auxiliary_driver`**: اسم الـ `struct auxiliary_driver` المراد تسجيله/إلغاء تسجيله.

**تفاصيل مهمة:**
- يعمل عبر `module_driver()` الـ generic macro في الـ kernel.
- كل module يمكنه استخدام هذا الـ macro مرة واحدة فقط.
- يستبدل `module_init()` و `module_exit()` معاً.

**من يستخدمه؟**
أي auxiliary driver بسيط لا يحتاج عمليات خاصة في الـ init أو exit.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. الـ debugfs — المداخل المهمة

الـ **auxiliary bus** ما عنده مجلد خاص فيه جوا `debugfs` بشكل مباشر، بس الـ **device core** بيوفر معلومات من خلاله. فكرة الـ auxiliary bus إنه "virtual bus" — ما في hardware registers حقيقية تقرأها، كل حياته في memory الـ kernel.

```bash
# تأكد إن debugfs متوصّل
mount -t debugfs none /sys/kernel/debug

# شوف البusات المسجّلة في الـ bus_type
ls /sys/kernel/debug/devices/

# راقب الـ device references (كم واحد ماسك الـ device)
cat /sys/kernel/debug/devices/

# لو عندك driver معين، شوف الـ kobject hierarchy
ls /sys/kernel/debug/
```

الـ **xarray** اللي بيخزن الـ IRQs في `auxiliary_device.sysfs.irqs` ممكن تتبعه من خلال الـ sysfs (جزء تالي).

---

#### 2. الـ sysfs — المداخل المهمة

الـ auxiliary bus بيسجّل كل device في `/sys/bus/auxiliary/`. هنا تلاقي كل جهاز auxiliary مسجّل مع اسمه الكامل بالشكل `modname.devname.id`.

```bash
# شوف كل الـ auxiliary devices المسجّلة على الـ bus
ls /sys/bus/auxiliary/devices/

# مثال مخرج:
# foo_mod.foo_dev.0
# mlx5_core.eth.0
# mlx5_core.rdma.0

# شوف الـ driver اللي بيقود كل device
ls -la /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/driver

# شوف الـ parent device
readlink /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/../..

# شوف الـ IRQs المرتبطة بالـ device (لو auxiliary_device_sysfs_irq_add تم استدعاؤه)
ls /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/irqs/

# شوف كل الـ drivers المسجّلين على الـ auxiliary bus
ls /sys/bus/auxiliary/drivers/

# شوف الـ modalias — اللي بيستخدمه udev للـ autoload
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/modalias
# مخرج: auxiliary:foo_mod.foo_dev

# شوف الـ uevent للـ device
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/uevent

# تحقق من الـ power state
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/power/runtime_status
```

---

#### 3. الـ ftrace — الـ tracepoints والـ events

الـ **auxiliary bus** يستخدم الـ device model العادي، فكل الـ tracepoints الخاصة بالـ `device` و `bus` شغّالة عليه.

```bash
# شوف كل الـ events المتعلقة بالـ device
grep -r "device" /sys/kernel/tracing/available_events | grep -i "bus\|probe\|remove\|bind"

# فعّل تتبع الـ probe و remove
echo 1 > /sys/kernel/tracing/events/device/enable

# أو بشكل أكثر تحديداً — تتبع bus events
echo 1 > /sys/kernel/tracing/events/enable

# فعّل function tracing لدوال الـ auxiliary bus
echo auxiliary_device_init       >> /sys/kernel/tracing/set_ftrace_filter
echo __auxiliary_device_add      >> /sys/kernel/tracing/set_ftrace_filter
echo auxiliary_device_delete     >> /sys/kernel/tracing/set_ftrace_filter
echo auxiliary_driver_unregister >> /sys/kernel/tracing/set_ftrace_filter
echo __auxiliary_driver_register >> /sys/kernel/tracing/set_ftrace_filter

echo function > /sys/kernel/tracing/current_tracer
echo 1        > /sys/kernel/tracing/tracing_on

# شوف النتائج
cat /sys/kernel/tracing/trace

# تتبع الـ probe بشكل خاص — مفيد لمعرفة إذا الـ driver اتصل فعلاً
echo ':mod:your_module_name' > /sys/kernel/tracing/set_ftrace_filter
```

---

#### 4. الـ printk والـ dynamic debug

الـ **auxiliary bus** يستخدم `pr_fmt` اللي بيضيف `KBUILD_MODNAME:__func__` قبل كل رسالة، فالـ log messages واضحة المصدر.

```bash
# فعّل الـ dynamic debug لكل دوال الـ auxiliary bus
echo "file drivers/base/auxiliary.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وأسماء الدوال (أكثر تفصيلاً)
echo "file drivers/base/auxiliary.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ module معين بيستخدم الـ auxiliary bus
echo "module your_parent_module +p" > /sys/kernel/debug/dynamic_debug/control

# شوف إيش مفعّل حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep auxiliary

# رفع مستوى الـ loglevel لرؤية الـ pr_debug messages
echo 8 > /proc/sys/kernel/printk
# أو مؤقتاً:
dmesg -n 8
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_DEBUG_DRIVER` | يفعّل رسائل debug في الـ device driver core |
| `CONFIG_DEBUG_DEVRES` | يتبع الـ managed device resources (devm_*) |
| `CONFIG_PROVE_LOCKING` | lockdep — يكشف deadlocks في الـ mutex داخل auxiliary_device.sysfs.lock |
| `CONFIG_DEBUG_LOCK_ALLOC` | يتحقق من صحة استخدام الـ locks |
| `CONFIG_SYSFS` | لازم يكون مفعّل عشان auxiliary_device_sysfs_irq_add تشتغل |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug المذكور أعلاه |
| `CONFIG_KASAN` | يكشف use-after-free في حال تم تحرير auxiliary_device قبل الأوان |
| `CONFIG_KCSAN` | يكشف data races في حال عدة threads تلمس الـ device في نفس الوقت |
| `CONFIG_UBSAN` | يكشف undefined behavior |
| `CONFIG_MEMCHECK` / `CONFIG_KMEMLEAK` | يتتبع memory leaks في حال نسي أحد يعمل auxiliary_device_uninit |
| `CONFIG_REF_TRACKER` | يتتبع من ماسك reference على الـ device |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_DEBUG_DRIVER|CONFIG_DEBUG_DEVRES|CONFIG_PROVE_LOCKING|CONFIG_KASAN"

# أو
grep -E "DEBUG_DRIVER|DEBUG_DEVRES|PROVE_LOCKING" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

الـ auxiliary bus ما عنده أداة `devlink` خاصة فيه مباشرة، لكن الـ drivers اللي تستخدمه (مثل mlx5، ice) بتعرض devlink.

```bash
# لو الـ parent هو NIC مثلاً (mlx5, ice):
devlink dev show
devlink dev info pci/0000:03:00.0

# شوف الـ auxiliary devices الخاصة بـ mlx5 كمثال حقيقي
ls /sys/bus/auxiliary/devices/ | grep mlx5

# أداة bus-wide مفيدة — شوف كل bindings
for dev in /sys/bus/auxiliary/devices/*; do
    echo "Device: $(basename $dev)"
    echo "  Driver: $(readlink $dev/driver 2>/dev/null | xargs basename 2>/dev/null || echo 'unbound')"
    echo "  Modalias: $(cat $dev/modalias 2>/dev/null)"
done

# تحقق من الـ module الأب
lsmod | grep -E "mlx5|ice|your_parent_module"
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `auxiliary device already has a release function` | محاولة تعيين `dev.release` بعد `device_initialize()` | عيّن `release` قبل `auxiliary_device_init()` |
| `auxiliary device must have a name` | `auxdev->name` هو NULL أو فاضي | عيّن `name` قبل `auxiliary_device_init()` |
| `auxiliary device must have a dev` | مشكلة في الـ `dev.parent` أو `dev.release` | تحقق من أن `dev.parent` و `dev.release` غير NULL |
| `auxiliary_device_add failed` أو `device_add: error -17` | اسم الـ device مكرر — نفس `modname.devname.id` موجود | غيّر الـ `id` ليكون unique |
| `BUG: sleeping function called from invalid context` | استدعاء `auxiliary_device_init()` من interrupt context | الـ init لازم من process context فقط |
| `WARNING: CPU: X PID: Y ... mutex_destroy` | الـ `mutex` داخل `sysfs.lock` ما اتدمّر صح | تأكد من استدعاء `auxiliary_device_uninit()` بعد `auxiliary_device_delete()` |
| `kobject_add_internal failed` | مشكلة في الـ sysfs عند إضافة الـ device | تحقق من عدم وجود device بنفس الاسم |
| `Driver 'foo' requests probe deferral` | الـ probe طلب defer — الـ parent جاهز | طبيعي، الـ kernel هيحاول مرة ثانية |
| `auxiliary bus: failed to load module` | الـ module المطلوب غير موجود | `modprobe` الـ module الصح |
| `use-after-free in auxiliary_device_release` | تم الوصول للـ `auxiliary_device` بعد تحرير ذاكرته | الـ `release()` callback فيه bug، راجع الـ `container_of` والـ `kfree` |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في دالة الـ release الخاصة بك — تحقق إن ما زلت مسجّلاً */
static void my_aux_dev_release(struct device *dev)
{
    struct my_parent_obj *parent = container_of(
        to_auxiliary_dev(dev), struct my_parent_obj, aux_dev);

    /* WARN لو الـ device يتحرر وهو لسه مسجّل — bug واضح */
    WARN_ON(device_is_registered(dev));

    kfree(parent);
}

/* في الـ probe — تحقق من الـ parent */
static int my_probe(struct auxiliary_device *auxdev,
                    const struct auxiliary_device_id *id)
{
    /* تأكد إن الـ parent موجود */
    WARN_ON(!auxdev->dev.parent);

    /* لو container_of أعطاك قيمة غريبة */
    struct my_parent_obj *parent = container_of(auxdev,
        struct my_parent_obj, aux_dev);
    WARN_ON(IS_ERR_OR_NULL(parent->shared_data));

    /* dump_stack عشان تعرف من استدعى الـ probe */
    dump_stack(); /* أزله بعد الـ debugging */
    ...
}

/* في الـ remove — تأكد من cleanup صح */
static void my_remove(struct auxiliary_device *auxdev)
{
    struct my_priv *priv = auxiliary_get_drvdata(auxdev);

    /* لو priv فاضي هنا — فيه bug في الـ probe/remove pairing */
    WARN_ON(!priv);
    ...
}

/* عند تسجيل الـ IRQ في sysfs */
int ret = auxiliary_device_sysfs_irq_add(auxdev, irq);
WARN_ON(ret < 0);
```

---

### Hardware Level

---

#### 1. التحقق من حالة الـ Hardware مقابل حالة الـ Kernel

الـ **auxiliary bus** هو virtual bus — ما في hardware مادي خلفه مباشرة. الـ hardware الحقيقي هو جهاز الـ parent (PCI/platform/etc). لذا التحقق يكون على مستوى الـ parent.

```bash
# شوف إذا الـ parent PCI device موجود وسليم
lspci -vvv -s 0000:03:00.0

# تحقق من الـ PCI config space للـ parent
setpci -s 0000:03:00.0 STATUS.w

# تحقق من الـ link state للـ parent
cat /sys/bus/pci/devices/0000:03:00.0/current_link_speed
cat /sys/bus/pci/devices/0000:03:00.0/current_link_width

# قارن بين الـ devices في kernel والـ hardware
# الـ kernel
ls /sys/bus/auxiliary/devices/

# الـ hardware (من PCI perspective)
lspci | grep -i "your_device_name"
```

---

#### 2. تقنيات الـ Register Dump

بما أن الـ auxiliary bus virtual، الـ registers هي registers الـ parent device.

```bash
# اقرأ الـ BAR (Base Address Register) للـ parent PCI device
# أولاً اعرف الـ BAR address
lspci -vvv -s 0000:03:00.0 | grep "Memory at"
# مثال مخرج: Memory at f7000000 (64-bit, prefetchable) [size=4M]

# استخدم devmem2 لقراءة register معين
devmem2 0xf7000000 w        # قرأ 32-bit word
devmem2 0xf7000004 w        # السجل التالي

# بديل — استخدم /dev/mem مع dd (على kernels قديمة)
dd if=/dev/mem bs=4 count=1 skip=$((0xf7000000 / 4)) 2>/dev/null | xxd

# io utility لـ I/O ports (x86 فقط)
# sudo io -4 0x3f8    # قراءة 32-bit من port

# الأفضل في الـ kernel نفسه — من داخل driver أو module debug:
# ioread32(base + OFFSET);
# print_hex_dump(KERN_DEBUG, "regs: ", DUMP_PREFIX_OFFSET, 16, 4, base, size, false);
```

---

#### 3. نصائح Logic Analyzer وـ Oscilloscope

الـ auxiliary bus تواصله مع الـ hardware يكون عبر الـ parent device (PCI/I2C/SPI/etc). نقاط القياس تعتمد على نوع الـ parent:

**لو الـ parent هو PCIe:**
```
نقاط القياس:
- PERST# (PCIe Reset) — تأكد إنه بيصير deassert صح عند boot
- CLKREQ# — يتحكم في power gating للـ clock
- PCIe lanes (TX/RX) — بالـ logic analyzer المدعوم لـ PCIe (مثل Keysight U4301B)
- راقب الـ Link Training (LTSSM states) — يجب أن يصل لـ L0
```

**لو الـ parent هو منفذ I2C/SPI (embedded):**
```
نقاط القياس:
- SDA/SCL للـ I2C — تحقق من الـ ACK/NACK
- MOSI/MISO/SCK/CS للـ SPI — تحقق من الـ timing
- IRQ line — تحقق إنها بتشتغل بالـ polarity الصح
- VCC — تأكد من الـ voltage levels
```

**نصيحة عملية:**
```
- فعّل الـ probe في الـ driver وراقب timing الـ signals
- ابحث عن glitches في الـ power supply عند الـ probe
- تحقق من الـ pull-up resistors على خطوط I2C
```

---

#### 4. مشاكل الـ Hardware الشائعة ونمط الـ Kernel Log

| المشكلة الـ Hardware | نمط الـ Kernel Log | التفسير |
|---|---|---|
| الـ Parent PCI device ما اتعرف | `auxiliary_device_add failed: -ENODEV` | الـ hardware غير موجود أو الـ PCI scan فشل |
| الـ IRQ ما اتوصّل | لا يوجد log + الـ device يتجمّد | تحقق من `cat /proc/interrupts` |
| الـ power supply غير مستقر | `PCIe Bus Error: severity=Corrected` | noise في الـ VCC أو الـ PCIe power |
| الـ firmware غير محمّل | `probe of xxx failed with error -ENOENT` | الـ firmware.bin غير موجود |
| الـ DMA ما يشتغل | `DMA-API: device driver tries to free DMA memory it has not allocated` | mismatch في الـ DMA addressing |
| reset الـ hardware لم يكتمل | `timeout waiting for device to initialize` | الـ hardware احتاج وقت أطول للـ reset |
| overcurrent أو حرارة | `Hardware Error: Machine check events logged` | مشكلة hardware حقيقية |

---

#### 5. الـ Device Tree Debugging

الـ auxiliary bus بالأساس ما بيعتمد على الـ Device Tree مباشرة (هو virtual). لكن الـ parent device (اللي بيخلق الـ auxiliary devices) ممكن يكون DT-based.

```bash
# شوف الـ DT node للـ parent device
# أولاً عرّف الـ parent
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/../uevent | grep OF_

# اقرأ الـ DT node المقابل
ls /proc/device-tree/
# أو مع dtc
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "your_device_compatible"

# تحقق من الـ compatible string
cat /proc/device-tree/soc/your_device@address/compatible

# تحقق من الـ reg property (عناوين الـ registers)
hexdump -C /proc/device-tree/soc/your_device@address/reg

# تحقق من الـ interrupts في الـ DT
cat /proc/device-tree/soc/your_device@address/interrupts | xxd

# قارن الـ DT مع الـ hardware manual
# الـ base address في DT يجب أن يطابق ما في datasheet
```

**تحقق من صحة الـ DT:**
```bash
# استخدم dtc للتحقق من syntax
dtc -I dtb -O dts -o /tmp/current.dts /sys/firmware/fdt
grep -A 10 "your_device" /tmp/current.dts

# شوف الـ overlays المطبّقة
ls /sys/firmware/devicetree/base/
```

---

### Practical Commands

---

#### سيناريو 1: الـ auxiliary device ما بيظهر

```bash
#!/bin/bash
# تشخيص شامل لـ auxiliary bus

echo "=== Auxiliary Bus Devices ==="
ls -la /sys/bus/auxiliary/devices/ 2>/dev/null || echo "No auxiliary devices found!"

echo ""
echo "=== Auxiliary Bus Drivers ==="
ls -la /sys/bus/auxiliary/drivers/ 2>/dev/null || echo "No auxiliary drivers found!"

echo ""
echo "=== Unbound Devices ==="
for dev in /sys/bus/auxiliary/devices/*; do
    driver=$(readlink "$dev/driver" 2>/dev/null)
    if [ -z "$driver" ]; then
        echo "UNBOUND: $(basename $dev)"
        echo "  modalias: $(cat $dev/modalias 2>/dev/null)"
    fi
done

echo ""
echo "=== Recent Kernel Messages ==="
dmesg | grep -E "auxiliary|auxdev" | tail -30
```

**مثال مخرج:**
```
=== Auxiliary Bus Devices ===
lrwxrwxrwx 1 root root 0 Feb 21 10:00 mlx5_core.eth.0 -> ...
lrwxrwxrwx 1 root root 0 Feb 21 10:00 mlx5_core.rdma.0 -> ...

=== Auxiliary Bus Drivers ===
drwxr-xr-x 2 root root 0 Feb 21 10:00 mlx5e_rep

=== Unbound Devices ===
UNBOUND: mlx5_core.rdma.0
  modalias: auxiliary:mlx5_core.mlx5_core.rdma
```

التفسير: `mlx5_core.rdma.0` غير مربوط — الـ driver `mlx5_ib` مش محمّل. الحل: `modprobe mlx5_ib`.

---

#### سيناريو 2: تتبع دورة حياة الـ device بالكامل

```bash
# فعّل function tracing قبل تحميل الـ module
echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace

cat > /sys/kernel/tracing/set_ftrace_filter << 'EOF'
auxiliary_device_init
__auxiliary_device_add
auxiliary_device_delete
auxiliary_device_uninit
__auxiliary_driver_register
auxiliary_driver_unregister
EOF

echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# حمّل الـ module
modprobe your_parent_module

# وقف الـ tracing وشوف النتائج
sleep 2
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | head -50
```

**مثال مخرج مفسَّر:**
```
 # DURATION    FUNCTION CALLS
 # |           |   |   |   |
   1.234 us  | auxiliary_device_init() {
   0.891 us  |   device_initialize();
             | }
   2.100 us  | __auxiliary_device_add() {
   1.500 us  |   device_add();      /* هنا بيظهر في sysfs */
             | }
```

إذا `device_add` أعاد error، ستلاقي `ret = -EEXIST` — يعني الاسم مكرر.

---

#### سيناريو 3: التحقق من الـ IRQ sysfs

```bash
# شوف الـ IRQs المسجّلة للـ auxiliary device
AUXDEV="foo_mod.foo_dev.0"

echo "=== IRQs for $AUXDEV ==="
ls /sys/bus/auxiliary/devices/$AUXDEV/irqs/ 2>/dev/null \
    || echo "No IRQs directory (CONFIG_SYSFS not set or irqs not added)"

# شوف تفاصيل كل IRQ
for irq_dir in /sys/bus/auxiliary/devices/$AUXDEV/irqs/*/; do
    irq=$(basename $irq_dir)
    echo "IRQ $irq:"
    grep -r "" $irq_dir 2>/dev/null
done

# قارن مع /proc/interrupts
echo ""
echo "=== /proc/interrupts (relevant lines) ==="
grep -E "foo_mod|foo_dev" /proc/interrupts
```

---

#### سيناريو 4: اكتشاف الـ memory leaks

```bash
# لو CONFIG_KMEMLEAK مفعّل
echo scan > /sys/kernel/debug/kmemleak
sleep 5
cat /sys/kernel/debug/kmemleak | grep -A 10 "auxiliary\|your_module"

# لو KASAN مفعّل — شوف الـ dmesg بعد rmmod
rmmod your_parent_module
dmesg | grep -E "KASAN|use-after-free|BUG" | tail -20

# تحقق من device references قبل rmmod
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/power/runtime_usage 2>/dev/null
```

---

#### سيناريو 5: الـ bind/unbind اليدوي للـ Driver

```bash
AUXDEV="foo_mod.foo_dev.0"
DRIVER="my_auxiliary_driver"

# فكّ ربط الـ driver يدوياً (مفيد للـ testing)
echo "$AUXDEV" > /sys/bus/auxiliary/drivers/$DRIVER/unbind
dmesg | tail -5

# أعد الربط
echo "$AUXDEV" > /sys/bus/auxiliary/drivers/$DRIVER/bind
dmesg | tail -5

# أو عبر override
echo "$DRIVER" > /sys/bus/auxiliary/devices/$AUXDEV/driver_override
echo "$AUXDEV" > /sys/bus/auxiliary/drivers/probe
```

---

#### سيناريو 6: تشخيص مشكلة الـ probe deferral

```bash
# شوف الـ devices التي طلبت probe deferral
cat /sys/kernel/debug/devices_deferred 2>/dev/null

# أو من الـ dmesg
dmesg | grep "probe deferred\|deferred probe"

# أجبر الـ deferred probes تشتغل الآن
echo 1 > /sys/bus/platform/drivers_autoprobe 2>/dev/null
# أو كـ module parameter لبعض الـ kernels
```

---

#### ملخص: الأوامر السريعة للـ Emergency Debugging

```bash
# الأمر الواحد اللي يعطيك كل شيء عن الـ auxiliary bus
echo "=== AUXILIARY BUS FULL SNAPSHOT ===" && \
echo "--- Devices ---" && ls /sys/bus/auxiliary/devices/ && \
echo "--- Drivers ---" && ls /sys/bus/auxiliary/drivers/ && \
echo "--- Recent logs ---" && dmesg | grep -E "auxiliary" | tail -20 && \
echo "--- Deferred probes ---" && cat /sys/kernel/debug/devices_deferred 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على AM62x — الـ driver بيـ crash عند الـ unbind

#### العنوان
**الـ release callback مش موجودة → kernel panic عند فصل الـ auxiliary device**

#### السياق
شركة بتصنع **industrial gateway** بناءً على **Texas Instruments AM62x**. الـ gateway بيشتغل كـ Modbus-to-Ethernet bridge. الـ SoC بيحتوي على وحدة Ethernet متكاملة اسمها **CPSW** (Common Platform Switch). المطور كتب driver للـ parent (CPSW) وبيسجّل منه **auxiliary_device** اسمه `cpsw_mod.mdio_phy` عشان يعالج الـ MDIO/PHY sub-function بشكل مستقل.

#### المشكلة
لما الـ operator بيعمل `rmmod` للـ parent driver، الـ kernel بيطبع:

```
BUG: kernel NULL pointer dereference, address: 0000000000000048
RIP: device_release+0x1c/0x60
```

الـ crash بيحصل في `device_release()` داخل `drivers/base/core.c` لأن الـ `dev->release` بيساوي `NULL`.

#### التحليل
بالرجوع للـ `auxiliary_bus.h`، الـ struct الأساسي هو:

```c
struct auxiliary_device {
    struct device dev;   /* ← هنا بيكون dev.release */
    const char *name;
    u32 id;
    struct { ... } sysfs;
};
```

الـ header نفسه بيوضح في الـ doc block:

> *"The auxiliary_device.dev.type.release or auxiliary_device.dev.release must be populated with a non-NULL pointer to successfully register the auxiliary_device."*

الـ `auxiliary_device_init()` بيفحص هذا الشرط. لكن المطور كان بيمرر الـ struct ناقص:

```c
/* كود المطور الخاطئ */
static int cpsw_probe(struct platform_device *pdev)
{
    struct cpsw_priv *priv = devm_kzalloc(...);

    priv->aux_dev.name = "mdio_phy";
    priv->aux_dev.id   = 0;
    priv->aux_dev.dev.parent = &pdev->dev;
    /* ❌ نسي dev.release */

    auxiliary_device_init(&priv->aux_dev); /* يرجع -EINVAL لكن المطور تجاهل القيمة */
    auxiliary_device_add(&priv->aux_dev);
}
```

المشكلة المزدوجة: الـ `auxiliary_device_init()` بيرجع `-EINVAL` لكن المطور ما تحقق من الـ return value، فالـ device اتسجّل بـ `release = NULL`.

#### الحل

```c
/* الخطوة 1: عرّف دالة الـ release */
static void cpsw_mdio_aux_release(struct device *dev)
{
    struct auxiliary_device *auxdev = to_auxiliary_dev(dev);
    /* container_of للوصول للـ parent struct */
    struct cpsw_priv *priv = container_of(auxdev, struct cpsw_priv, aux_dev);
    /* حرر أي موارد خاصة بالـ aux device هنا */
    dev_dbg(dev, "MDIO aux device released\n");
}

/* الخطوة 2: سجّل الـ release قبل auxiliary_device_init */
static int cpsw_probe(struct platform_device *pdev)
{
    struct cpsw_priv *priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);

    priv->aux_dev.name       = "mdio_phy";
    priv->aux_dev.id         = 0;
    priv->aux_dev.dev.parent = &pdev->dev;
    priv->aux_dev.dev.release = cpsw_mdio_aux_release; /* ✅ */

    ret = auxiliary_device_init(&priv->aux_dev);
    if (ret) {  /* ✅ تحقق دائماً */
        dev_err(&pdev->dev, "aux_dev_init failed: %d\n", ret);
        return ret;
    }

    ret = auxiliary_device_add(&priv->aux_dev);
    if (ret) {
        auxiliary_device_uninit(&priv->aux_dev); /* ✅ حسب الـ doc */
        return ret;
    }
    return 0;
}
```

#### الدرس المستفاد
الـ `auxiliary_bus.h` يصرّح بوضوح إن `dev.release` إلزامي، والـ `auxiliary_device_init()` بيرجع خطأ إذا غايب. **لازم دائماً تتحقق من الـ return value** وتتبع تسلسل الـ init/uninit المذكور في الـ header بالضبط. الـ three-step registration مش اختياري.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — الـ driver ما بيـ probe

#### العنوان
**الـ id_table اسم خاطئ → الـ auxiliary driver لا يرتبط بأي device**

#### السياق
فريق يطوّر **Android TV box** بناءً على **Allwinner H616**. الـ SoC بيحتوي على وحدة HDMI متكاملة مع PHY منفصل. المطور قرر يفصل الـ HDMI controller كـ parent driver ويسجّل الـ PHY كـ `auxiliary_device`، ثم يكتب driver منفصل للـ PHY.

#### المشكلة
عند التشغيل، الـ auxiliary device بيظهر في `/sys/bus/auxiliary/devices/` بالاسم `hdmi_ctrl_mod.hdmi_phy.0` لكن ما في driver بيـ bind معه. الـ HDMI ما بيشتغل خالص.

```bash
$ ls /sys/bus/auxiliary/devices/
hdmi_ctrl_mod.hdmi_phy.0

$ cat /sys/bus/auxiliary/devices/hdmi_ctrl_mod.hdmi_phy.0/modalias
auxiliary:hdmi_ctrl_mod.hdmi_phy
```

#### التحليل
الـ `auxiliary_bus.h` بيشرح آلية الـ matching:

> *"a driver registering an auxiliary device is named 'foo_mod.ko' and the subdevice is named 'foo_dev'. The match name is therefore 'foo_mod.foo_dev'."*

يعني الـ match name = `KBUILD_MODNAME` + `.` + `auxiliary_device.name`

الماكرو:
```c
#define auxiliary_device_add(auxdev) __auxiliary_device_add(auxdev, KBUILD_MODNAME)
```

`__auxiliary_device_add` بتاخذ الـ `modname` وتبني اسم الـ device كـ `modname.auxdev->name.id`.

المطور في الـ PHY driver كتب:

```c
/* ❌ كود المطور الخاطئ في ملف phy_driver.c */
static const struct auxiliary_device_id h616_hdmi_phy_ids[] = {
    { .name = "h616_phy_mod.hdmi_phy" }, /* ← اسم المودل خاطئ! */
    {}
};
MODULE_DEVICE_TABLE(auxiliary, h616_hdmi_phy_ids);
```

لكن الـ parent module اسمه `hdmi_ctrl_mod` (الـ `KBUILD_MODNAME` في Makefile)، مش `h616_phy_mod`.

#### الحل

```bash
# أولاً: اكتشف الاسم الصحيح من sysfs
$ cat /sys/bus/auxiliary/devices/hdmi_ctrl_mod.hdmi_phy.0/modalias
auxiliary:hdmi_ctrl_mod.hdmi_phy

# الاسم الصحيح = hdmi_ctrl_mod.hdmi_phy
```

```c
/* ✅ الكود الصحيح */
static const struct auxiliary_device_id h616_hdmi_phy_ids[] = {
    { .name = "hdmi_ctrl_mod.hdmi_phy" }, /* يطابق modname.devname */
    {}
};
MODULE_DEVICE_TABLE(auxiliary, h616_hdmi_phy_ids);

struct auxiliary_driver h616_hdmi_phy_driver = {
    .name     = "h616_hdmi_phy",
    .id_table = h616_hdmi_phy_ids,
    .probe    = h616_hdmi_phy_probe,
    .remove   = h616_hdmi_phy_remove,
};

module_auxiliary_driver(h616_hdmi_phy_driver);
/* ↑ الماكرو من auxiliary_bus.h يوفر module_init/module_exit تلقائياً */
```

```makefile
# تأكد في Makefile إن اسم المودل صح
obj-$(CONFIG_H616_HDMI) += hdmi_ctrl_mod.o
obj-$(CONFIG_H616_HDMI_PHY) += h616_hdmi_phy.o
```

#### الدرس المستفاد
الـ `struct auxiliary_device_id.name` لازم يطابق `KBUILD_MODNAME.auxdev->name` بالضبط. استخدم `modalias` في sysfs كمصدر الحقيقة الوحيد. الـ `module_auxiliary_driver()` macro من `auxiliary_bus.h` بيبسّط التسجيل ويقلل الأخطاء.

---

### السيناريو الثالث: IoT Sensor Hub على STM32MP1 — memory leak عند الـ remove

#### العنوان
**عدم اتباع ترتيب الـ delete/uninit → memory leak مزمن في production**

#### السياق
منتج **IoT sensor hub** يعمل على **STM32MP157** يجمع بيانات من عشرات الـ sensors عبر **I2C** و**SPI**. الـ firmware يعمل لأشهر بدون restart. بعد أسبوع من الـ deployment، الـ system يبدأ يبطأ والذاكرة تنفد تدريجياً. الـ memory leak مش واضح في أول وهلة.

#### المشكلة
الـ kmemleak tool بيرصد:

```
unreferenced object 0xc3a40000 (size 512):
  comm "kworker", pid 47
  backtrace:
    kmalloc
    sensor_hub_probe
    auxiliary_bus_probe
```

الـ leak بيتراكم في كل مرة يتم فيها hot-reload للـ sensor driver (بسبب I2C error recovery).

#### التحليل
الـ header بيوضح في قسم `DEVICE_LIFESPAN`:

> *"Unregistering an auxiliary_device is a two-step process... First call auxiliary_device_delete(), then call auxiliary_device_uninit()."*

والأهم:
> *"The registering driver is wholly responsible for the management of the memory used for the device object."*

```c
static inline void auxiliary_device_uninit(struct auxiliary_device *auxdev)
{
    mutex_destroy(&auxdev->sysfs.lock);  /* تدمير الـ mutex */
    put_device(&auxdev->dev);            /* تقليل الـ reference count */
}

static inline void auxiliary_device_delete(struct auxiliary_device *auxdev)
{
    device_del(&auxdev->dev);            /* إزالة من الـ bus */
}
```

المطور كتب:

```c
/* ❌ كود Remove الخاطئ */
static void sensor_hub_remove(struct platform_device *pdev)
{
    struct sensor_hub_priv *priv = platform_get_drvdata(pdev);

    /* نسي auxiliary_device_delete أولاً! */
    auxiliary_device_uninit(&priv->aux_dev); /* ← uninit مباشرة بدون delete */
    kfree(priv); /* ← الذاكرة اتحررت لكن الـ device لسه على الـ bus! */
}
```

لما الـ driver بيتحمّل مجدداً، بيحاول يسجّل device بنفس الاسم، فالـ kernel بيخلق كائن جديد بدل ما يستخدم القديم، والقديم بيضل معلق.

#### الحل

```c
/* ✅ الكود الصحيح - اتبع الترتيب الإلزامي */
static void sensor_hub_remove(struct platform_device *pdev)
{
    struct sensor_hub_priv *priv = platform_get_drvdata(pdev);

    /* الخطوة 1: احذف من الـ bus أولاً */
    auxiliary_device_delete(&priv->aux_dev);

    /* الخطوة 2: uninit لتحرير الـ reference */
    auxiliary_device_uninit(&priv->aux_dev);

    /* الـ kfree بيحصل في release callback مش هنا مباشرة */
}

/* release callback هو المكان الصحيح لـ kfree */
static void sensor_aux_release(struct device *dev)
{
    struct auxiliary_device *auxdev = to_auxiliary_dev(dev);
    struct sensor_hub_priv *priv =
        container_of(auxdev, struct sensor_hub_priv, aux_dev);
    kfree(priv); /* ✅ هنا الـ kfree الصحيح */
}
```

**أو استخدم devm لتجنب المشكلة كلياً:**

```c
/* باستخدام devm_auxiliary_device_create من auxiliary_bus.h */
static int sensor_hub_probe(struct platform_device *pdev)
{
    struct auxiliary_device *aux;

    aux = devm_auxiliary_device_create(&pdev->dev, "i2c_sensor", platform_data);
    if (IS_ERR(aux))
        return PTR_ERR(aux);
    /* devm يتكفل بالـ cleanup تلقائياً عند الـ remove */
}
```

#### الدرس المستفاد
الـ `auxiliary_bus.h` يوضح إن lifecycle إدارة الذاكرة **مسؤولية الـ registering driver بالكامل**. الترتيب `delete` ثم `uninit` إلزامي وليس اختيارياً. في الـ production systems الطويلة، استخدم `devm_auxiliary_device_create()` عند الإمكان لتجنب الـ leaks.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — تعارض في الـ IDs

#### العنوان
**الـ id غير فريد → `device_add` فشل و AUTOSAR stack ما اشتغل**

#### السياق
شركة تطوير **automotive ECU** تستخدم **NXP i.MX8QM**. الـ SoC بيحتوي على أربع وحدات **CAN-FD** (FlexCAN). المطور بيسجّل كل وحدة كـ `auxiliary_device` من parent driver واحد لإدارة الـ shared DMA pool.

#### المشكلة
عند التشغيل:

```
[    2.341] auxiliary-bus flexcan_mod.can_fd: device already exists
[    2.342] auxiliary-bus flexcan_mod.can_fd: device already exists
[    2.343] auxiliary-bus flexcan_mod.can_fd: device already exists
```

ثلاثة من أربعة وحدات CAN ما اتسجّلت. الـ AUTOSAR COM stack ما اشتغل لأن بعض الـ CAN interfaces مش موجودة.

#### التحليل
الـ header يشرح:

> *"If two auxiliary_devices with the same match_name, eg 'foo_mod.foo_dev', are registered onto the bus, they must have unique id values (e.g. 'x' and 'y') so that the registered devices names are 'foo_mod.foo_dev.x' and 'foo_mod.foo_dev.y'. If match_name + id are not unique, then the device_add fails."*

الاسم النهائي للـ device في sysfs = `modname.auxdev->name.id`

```c
struct auxiliary_device {
    struct device dev;
    const char *name;
    u32 id;       /* ← هذا الحقل لازم يكون فريد لكل device بنفس الاسم */
    ...
};
```

كود المطور:

```c
/* ❌ كود المطور الخاطئ */
for (int i = 0; i < 4; i++) {
    priv->can_aux[i].name = "can_fd";
    priv->can_aux[i].id   = 0;  /* ← نفس الـ ID للكل! */
    priv->can_aux[i].dev.parent  = &pdev->dev;
    priv->can_aux[i].dev.release = flexcan_aux_release;

    auxiliary_device_init(&priv->can_aux[i]);
    auxiliary_device_add(&priv->can_aux[i]); /* فشل من الـ i=1 */
}
```

#### الحل

```c
/* ✅ الكود الصحيح */
for (int i = 0; i < 4; i++) {
    priv->can_aux[i].name = "can_fd";
    priv->can_aux[i].id   = i;  /* ✅ ID فريد: 0, 1, 2, 3 */
    priv->can_aux[i].dev.parent  = &pdev->dev;
    priv->can_aux[i].dev.release = flexcan_aux_release;

    ret = auxiliary_device_init(&priv->can_aux[i]);
    if (ret)
        goto err_init;

    ret = auxiliary_device_add(&priv->can_aux[i]);
    if (ret) {
        auxiliary_device_uninit(&priv->can_aux[i]);
        goto err_add;
    }
}

/* النتيجة في sysfs:
   flexcan_mod.can_fd.0
   flexcan_mod.can_fd.1
   flexcan_mod.can_fd.2
   flexcan_mod.can_fd.3
*/
```

**وفي الـ auxiliary driver، استخدم الـ id للتعرف على الوحدة:**

```c
static int flexcan_aux_probe(struct auxiliary_device *auxdev,
                              const struct auxiliary_device_id *id)
{
    u32 can_instance = auxdev->id; /* 0..3 */
    dev_info(&auxdev->dev, "Probing CAN-FD instance %u\n", can_instance);
    /* هيك تعرف أي وحدة CAN تشتغل معها */
}
```

#### الدرس المستفاد
الـ `u32 id` في `struct auxiliary_device` مش decoration، هو جزء أساسي من الـ unique identifier. في كل مرة تسجّل أكثر من device بنفس الـ name، لازم تضمن إن الـ id مختلف. استخدم الـ loop index أو atomic counter.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — الـ auxiliary driver ما بيـ unload نظيف

#### العنوان
**الـ shared object أطول عمراً من الـ auxiliary device → use-after-free في الـ bring-up**

#### السياق
فريق bring-up لـ **custom carrier board** يستخدم **Rockchip RK3562**. الـ SoC بيحتوي على USB OTG ومعه PHY متكامل. المطور بيفصل الـ USB PHY كـ auxiliary device لإعادة استخدام الـ PHY driver في مشاريع مختلفة. خلال الـ bring-up، كل مرة بيعمل `rmmod` بيظهر crash عشوائي.

#### المشكلة
```
[  42.117] general protection fault: 0000 [#1] PREEMPT SMP
[  42.118] RIP: 0xffffffffc0a3218b [usb_phy_drv+0x18b]
[  42.119] Call Trace:
[  42.119]  phy_power_off
[  42.120]  rk3562_usb_phy_remove
```

الـ crash في `phy_power_off` لأن الـ PHY driver بيحاول يصل لـ `shared_regs` (مؤشر للـ register map المشترك) بعد ما الـ parent driver حرّرها.

#### التحليل
الـ `auxiliary_bus.h` يحذّر بوضوح في قسم `DEVICE_LIFESPAN`:

> *"The memory for the shared object(s) must have a lifespan equal to, or greater than, the lifespan of the memory for the auxiliary_device."*

و:

> *"The auxiliary_driver should only consider that the shared object is valid as long as the auxiliary_device is still registered on the auxiliary bus."*

و:

> *"The registering driver must unregister all auxiliary devices before its own driver.remove() is completed."*

الهيكل المشترك بين الـ parent والـ auxiliary driver:

```c
/* في الـ shared header */
struct rk3562_usb_shared {
    void __iomem *regs;     /* ← الـ shared object */
    struct clk *clk;
};
```

كود الـ parent:

```c
/* ❌ كود Remove الخاطئ */
static void rk3562_usb_remove(struct platform_device *pdev)
{
    struct rk3562_usb_priv *priv = platform_get_drvdata(pdev);

    /* ❌ حرر الـ shared resources أولاً قبل حذف الـ aux device! */
    iounmap(priv->shared.regs);  /* الـ PHY driver لسه ممكن يستخدمها */
    clk_disable_unprepare(priv->shared.clk);

    /* ثم delete الـ aux device - متأخر جداً */
    auxiliary_device_delete(&priv->phy_aux_dev);
    auxiliary_device_uninit(&priv->phy_aux_dev);
}
```

#### الحل

```c
/* ✅ الترتيب الصحيح: auxiliary devices أولاً، ثم shared resources */
static void rk3562_usb_remove(struct platform_device *pdev)
{
    struct rk3562_usb_priv *priv = platform_get_drvdata(pdev);

    /* الخطوة 1: امسح الـ aux devices أولاً */
    /* هذا يضمن إن الـ PHY driver يستدعي remove() ويوقف نفسه */
    auxiliary_device_delete(&priv->phy_aux_dev);
    auxiliary_device_uninit(&priv->phy_aux_dev);

    /* الخطوة 2: الآن آمن نحرر الـ shared resources */
    iounmap(priv->shared.regs);
    clk_disable_unprepare(priv->shared.clk);
}

/* ✅ أفضل: استخدم devm_add_action_or_reset كما يقترح الـ header */
static int rk3562_usb_probe(struct platform_device *pdev)
{
    struct rk3562_usb_priv *priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);

    /* سجّل الـ cleanup function عبر devm */
    ret = devm_add_action_or_reset(&pdev->dev, rk3562_usb_cleanup_aux, priv);
    if (ret)
        return ret;

    /* ثم سجّل الـ aux device */
    ret = auxiliary_device_init(&priv->phy_aux_dev);
    ...
}

static void rk3562_usb_cleanup_aux(void *data)
{
    struct rk3562_usb_priv *priv = data;
    auxiliary_device_delete(&priv->phy_aux_dev);
    auxiliary_device_uninit(&priv->phy_aux_dev);
}
```

**للـ debug خلال الـ bring-up:**

```bash
# شوف الـ reference count قبل وبعد الـ remove
$ cat /sys/bus/auxiliary/devices/rk3562_usb_mod.usb_phy.0/dev
$ ls -la /sys/bus/auxiliary/devices/

# راقب الـ kernel log أثناء rmmod
$ dmesg -w &
$ rmmod rk3562_usb_mod

# تأكد إن الـ phy driver اتـ unbound أولاً
$ cat /sys/bus/auxiliary/devices/rk3562_usb_mod.usb_phy.0/driver
```

#### الدرس المستفاد
الـ `auxiliary_bus.h` يفرض قاعدة واضحة: **الـ shared objects لازم تعيش طول ما الـ auxiliary device مسجّل أو أكثر**. ترتيب الـ cleanup هو عكس ترتيب الـ init تماماً: aux devices أولاً، ثم shared resources. استخدام `devm_add_action_or_reset()` هو أفضل ممارسة لضمان الترتيب الصحيح تلقائياً في كل الحالات.
## Phase 7: مصادر ومراجع

---

### مقدمة

هنا بنجمع كل المراجع المهمة اللي تساعدك تفهم الـ **auxiliary bus** subsystem أكثر وأكثر. سواء كنت مبتدئ بس بدك تفهم، أو developer متمرس بدك تعمق فهمك، هاد القسم هو roadmap الكاملة.

---

### مقالات LWN.net

**الـ LWN.net** هو المصدر الأهم لأخبار وتحليلات Linux kernel. الـ articles التالية مباشرة عن الـ auxiliary bus:

| المقال | الوصف |
|--------|--------|
| [Managing multifunction devices with the auxiliary bus](https://lwn.net/Articles/840416/) | المقال الأساسي اللي شرح فيه Dave Ertman لأول مرة فكرة الـ auxiliary bus وليش احتجناه بدل MFD |
| [Auxiliary bus driver support for Intel PCIe VSEC/DVSEC](https://lwn.net/Articles/877963/) | مثال حقيقي لاستخدام الـ auxiliary bus: تحويل intel_pmt driver من MFD للـ auxiliary bus |
| [Add Intel Ethernet Protocol Driver for RDMA (irdma)](https://lwn.net/Articles/851856/) | مثال عملي ثاني: كيف RDMA stack استخدم auxiliary bus مع Intel Ethernet |
| [Add Intel LJCA device driver](https://lwn.net/Articles/927063/) | driver آخر بيستخدم الـ auxiliary bus لـ USB-based Low Latency Communication Aggregator |

**نصيحة**: ابدأ بمقال [840416](https://lwn.net/Articles/840416/) — هو اللي بيشرح الـ "ليش" بشكل واضح.

---

### التوثيق الرسمي في الـ Kernel

**الـ Documentation/** هي المرجع الأول دايمًا — مكتوبة من نفس الناس اللي كتبت الكود:

```
Documentation/driver-api/auxiliary_bus.rst
```

- **الصفحة الرسمية على docs.kernel.org**:
  [https://docs.kernel.org/driver-api/auxiliary_bus.html](https://docs.kernel.org/driver-api/auxiliary_bus.html)

- **نسخة kernel 5.11 التاريخية** (أول نسخة دخلت فيها):
  [https://www.kernel.org/doc/html/v5.11/driver-api/auxiliary_bus.html](https://www.kernel.org/doc/html/v5.11/driver-api/auxiliary_bus.html)

- **مصدر الـ RST على GitHub** (torvalds/linux):
  [https://github.com/torvalds/linux/blob/master/Documentation/driver-api/auxiliary_bus.rst](https://github.com/torvalds/linux/blob/master/Documentation/driver-api/auxiliary_bus.rst)

**ملف الـ Header** اللي بنشرحه:
```
include/linux/auxiliary_bus.h
```

**ملف الـ Implementation** الرئيسي:
```
drivers/base/auxiliary.c
```

---

### Commits المهمة في تاريخ الـ Auxiliary Bus

#### الـ Commit الأساسي — إضافة الـ auxiliary bus لأول مرة

الـ patch دخل في **Linux 5.11** (نوفمبر 2020)، بكتابة **Dave Ertman** من Intel:

- **LKML archive للـ patch الأساسي**:
  [Linux-Kernel Archive: [PATCH v4 01/10] Add auxiliary bus support](https://lkml.iu.edu/hypermail/linux/kernel/2011.1/07745.html)

- **Patchwork — تعديلات Greg KH**:
  [[3/3] driver core: auxiliary bus: minor coding style tweaks](https://patchwork.kernel.org/project/netdevbpf/patch/X8ohGE8IBKiafzka@kroah.com/)

- **LKML — نفس الـ patch من Greg KH**:
  [https://lore.kernel.org/lkml/X8ohGE8IBKiafzka@kroah.com/](https://lore.kernel.org/lkml/X8ohGE8IBKiafzka@kroah.com/)

- **StarlingX kernel mirror** (commit ef3c9a46):
  [kernel: Add auxiliary bus support](https://opendev.org/starlingx/kernel/commit/ef3c9a46180d8f4f70ba544693bbc70d7f9dd9a0)

#### إضافة IRQ sysfs support

دخلت في **Linux 6.11**، بتسمح للـ auxiliary devices تكشف الـ IRQs الخاصة بها عبر sysfs.

#### تاريخ التسمية

الـ auxiliary bus مرت بأسماء قبل ما توصل للاسم النهائي:
- بدأت باسم **"ancillary bus"**
- بعدين **"virtual bus"**
- وأخيرًا **"auxiliary bus"**

---

### نقاشات Mailing List

هاي النقاشات بتعطيك فهم لـ"ليش" القرارات التصميمية اتخذت:

| الرابط | الوصف |
|--------|--------|
| [LKML: [PATCH v4 01/10] Add auxiliary bus support](https://lkml.iu.edu/hypermail/linux/kernel/2011.1/07745.html) | أول patch رسمي مع نقاش المراجعة |
| [mail-archive: resend/standalone PATCH v4](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2419528.html) | Dan Williams يعيد إرسال الـ patch لدفع العملية للأمام |
| [lore.kernel.org: Greg KH tweaks](https://lore.kernel.org/lkml/X8ohGE8IBKiafzka@kroah.com/) | ملاحظات Greg Kroah-Hartman على الـ coding style |
| [DPDK RFC: bus/auxiliary](https://inbox.dpdk.org/dev/20210413032329.25551-1-xuemingl@nvidia.com/T/) | NVIDIA/Mellanox بيقترحوا نفس المفهوم في DPDK |

**ملاحظة**: الـ lore.kernel.org هو الأرشيف الرسمي لكل نقاشات الـ kernel — استخدمه دايمًا للبحث.

---

### صفحات kernelnewbies.org

الـ **kernelnewbies.org** بتوثق التغييرات لكل kernel version. هاي الصفحات بتحكيلك متى وكيف تطور الـ auxiliary bus:

| الصفحة | التغيير المتعلق بـ auxiliary bus |
|--------|--------------------------------|
| [Linux_5.11](https://kernelnewbies.org/Linux_5.11) | دخول الـ auxiliary bus للأول مرة |
| [Linux_5.17](https://kernelnewbies.org/Linux_5.17) | إضافة aux-bus support لمزيد من الـ drivers |
| [Linux_6.1](https://kernelnewbies.org/Linux_6.1) | auxiliary bus driver للـ pci1xxxx MFE |
| [Linux_6.2](https://kernelnewbies.org/Linux_6.2) | mana driver auxiliary device support |
| [Linux_6.7](https://kernelnewbies.org/Linux_6.7) | PTP auxiliary bus support |
| [Linux_6.11](https://kernelnewbies.org/Linux_6.11) | auxiliary bus IRQs sysfs |

---

### أخبار ومقالات تقنية أخرى

| الرابط | الوصف |
|--------|--------|
| [Phoronix: Auxiliary Bus Support Coming To Linux 5.11](https://www.phoronix.com/news/Linux-5.11-Auxiliary-Bus) | خبر إضافة الـ auxiliary bus بلغة بسيطة |
| [eLinux.org: Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) | عروض تقديمية عامة عن driver model في Linux |
| [eLinux.org: Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | روابط لكل مصادر الـ Linux kernel المهمة |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
**الكتاب الكلاسيكي** لتعلم كتابة الـ Linux drivers، حر مجانًا:
- [Free PDF من lwn.net](https://lwn.net/Kernel/LDD3/)
- **الفصول الأهم لفهم auxiliary bus**:
  - Chapter 14: The Linux Device Model — يشرح كيف `struct device`, `struct bus_type`, `struct device_driver` كلها بتشتغل مع بعض
  - Chapter 3: Char Drivers — أساس فهم الـ driver model

> **ملاحظة**: LDD3 كُتبت لـ kernel 2.6، بس أساسيات الـ driver model ما تغيرت كثيرًا. الـ auxiliary bus بُني فوق نفس الـ infrastructure.

#### Linux Kernel Development — Robert Love
- **الفصل الأهم**: Chapter 17 — Devices and Modules
- بيشرح الـ kobject, kset, sysfs بطريقة واضحة وبسيطة
- متاح على [Amazon](https://www.amazon.com/Linux-Kernel-Development-Robert-Love/dp/0672329468)

#### Embedded Linux Primer — Christopher Hallinan
- بيشرح Device Model من منظور embedded systems
- مفيد لفهم كيف الـ auxiliary bus يُستخدم في hardware حقيقي
- مهم خصوصًا لمن يعمل على أجهزة network أو RDMA

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- أعمق كتاب تقني — بيشرح كل شيء من الـ source code مباشرة
- Chapter 6: Device Drivers — مهم جدًا

---

### مصطلحات البحث

لما تبدأ تبحث بنفسك، استخدم هاي المصطلحات:

```
# للبحث في LKML وlore.kernel.org
auxiliary_bus linux kernel
auxiliary_device probe match
devm_auxiliary_device_create
module_auxiliary_driver
KBUILD_MODNAME auxiliary

# للبحث عن أمثلة في source code
grep -r "auxiliary_driver_register" drivers/
grep -r "auxiliary_device_add" drivers/
grep -r "module_auxiliary_driver" drivers/

# للبحث في Git history
git log --oneline --all -- drivers/base/auxiliary.c
git log --oneline --all -- include/linux/auxiliary_bus.h
```

#### مصطلحات إضافية للبحث

| بالإنجليزية | الوصف |
|-------------|--------|
| `auxiliary bus subsystem` | للبحث العام |
| `multifunction device linux driver sharing` | لفهم المشكلة الأصلية |
| `MFD vs auxiliary bus` | مقارنة الحلين |
| `container_of auxiliary_device` | لفهم كيف الـ drivers تصل للـ parent struct |
| `devm auxiliary device linux kernel` | للـ resource-managed version |
| `auxiliary_device_id match` | لفهم آلية الـ matching |

---

### ملخص سريع للمسار التعليمي

```
مبتدئ ──► LDD3 Chapter 14 (Driver Model الأساس)
    │
    ▼
متوسط ──► lwn.net/Articles/840416 (ليش auxiliary bus؟)
    │
    ▼
متقدم ──► drivers/base/auxiliary.c (source code)
    │       + Documentation/driver-api/auxiliary_bus.rst
    ▼
خبير ──► LKML discussions + kernel commits + أمثلة حقيقية
           مثل: drivers/net/ethernet/intel/ice/ice_aux_support.c
```

---

### روابط سريعة — Quick Reference

```
# التوثيق الرسمي
https://docs.kernel.org/driver-api/auxiliary_bus.html

# المقال الأساسي على LWN
https://lwn.net/Articles/840416/

# الـ Header File على GitHub
https://github.com/torvalds/linux/blob/master/include/linux/auxiliary_bus.h

# الـ Implementation على GitHub
https://github.com/torvalds/linux/blob/master/drivers/base/auxiliary.c

# البحث في lore.kernel.org
https://lore.kernel.org/search/?q=auxiliary_bus

# kernelnewbies 5.11 (أول إضافة)
https://kernelnewbies.org/Linux_5.11
```
## Phase 8: Writing simple module

### الفكرة العامة

هنحط **kprobe** على الدالة `__auxiliary_device_add` — وهي الدالة اللي بتتسمى كل ما driver يسجّل **auxiliary device** جديد على الـ auxiliary bus. كل مرة تتسمى، الـ kprobe بتصحّى وتطبع اسم الـ device والـ modname اللي سجّله. كده نقدر نشوف في الـ kernel log مين بيضيف devices على الـ auxiliary bus وامتى.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * auxbus_probe.c
 *
 * kprobe على __auxiliary_device_add
 * يطبع اسم الـ auxiliary device وكل modname يسجّله
 */

/* -------- الـ includes -------- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/auxiliary_bus.h>/* struct auxiliary_device */

/*
 * الـ includes دي هي اللي محتاجينها:
 * - module.h و kernel.h: أساسيات أي module
 * - kprobes.h: عشان نستخدم آلية الـ kprobe
 * - auxiliary_bus.h: عشان نقدر نفسّر البارامتر الأول (struct auxiliary_device)
 */

/* -------- الـ pre-handler (callback) -------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * الـ kprobe بتتسمى قبل تنفيذ __auxiliary_device_add
     * البارامترات بتيجي من الـ registers حسب calling convention للـ architecture:
     *   - الـ argument الأول  (RDI على x86_64) = struct auxiliary_device *auxdev
     *   - الـ argument التاني (RSI على x86_64) = const char *modname
     */

#ifdef CONFIG_X86_64
    struct auxiliary_device *auxdev = (struct auxiliary_device *)regs->di;
    const char              *modname = (const char *)regs->si;
#elif defined(CONFIG_ARM64)
    struct auxiliary_device *auxdev = (struct auxiliary_device *)regs->regs[0];
    const char              *modname = (const char *)regs->regs[1];
#else
    /* fallback — مش هيشتغل صح على architectures تانية بدون تعديل */
    struct auxiliary_device *auxdev = NULL;
    const char              *modname = "unknown";
#endif

    if (!auxdev)
        return 0;

    /*
     * auxdev->name هو الجزء اللي بعد النقطة في match_name
     * مثلاً لو modname="foo_mod" و name="foo_dev"
     * → الـ match_name بيبقى "foo_mod.foo_dev"
     */
    pr_info("[auxbus_probe] auxiliary_device_add called!\n");
    pr_info("[auxbus_probe]   modname  = %s\n", modname ? modname : "NULL");
    pr_info("[auxbus_probe]   dev name = %s\n", auxdev->name ? auxdev->name : "NULL");
    pr_info("[auxbus_probe]   dev id   = %u\n", auxdev->id);

    /* نرجّع 0 = اكمل تنفيذ الدالة الأصلية بشكل طبيعي */
    return 0;
}

/* -------- تعريف الـ kprobe -------- */
static struct kprobe kp = {
    /*
     * بنحدد اسم الدالة اللي عايزين نعمل probe عليها بالاسم الحرفي
     * الـ kernel بيقدر يلاقيها في الـ kallsyms تلقائياً
     * استخدمنا __auxiliary_device_add بدل الـ macro لأنها الدالة الفعلية المصدّرة
     */
    .symbol_name = "__auxiliary_device_add",
    .pre_handler = handler_pre,
};

/* -------- module_init -------- */
static int __init auxbus_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بتسجّل الـ hook في الـ kernel
     * لو الدالة مش موجودة أو الـ kprobes مش مفعّل، بترجّع error
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[auxbus_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[auxbus_probe] planted kprobe at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* -------- module_exit -------- */
static void __exit auxbus_probe_exit(void)
{
    /*
     * لازم نشيل الـ kprobe قبل ما الـ module يتحمّل من الـ memory
     * لو ما عملناش unregister، الـ kernel هيحاول ينفّذ handler_pre
     * بعد ما الكود اتمسح → kernel panic مضمون
     */
    unregister_kprobe(&kp);
    pr_info("[auxbus_probe] kprobe unregistered\n");
}

module_init(auxbus_probe_init);
module_exit(auxbus_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on __auxiliary_device_add to trace auxiliary bus device registration");
```

---

### شرح كل جزء

#### الـ includes

| Header | ليه محتاجينه |
|---|---|
| `linux/module.h` | الماكروهات الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe`، `register_kprobe`، `unregister_kprobe` |
| `linux/auxiliary_bus.h` | عشان نقدر نفسّر `struct auxiliary_device` من الـ register |

---

#### الـ `handler_pre` — قلب الموضوع

الـ kprobe بتتسمى **قبل** تنفيذ `__auxiliary_device_add`. في هذه اللحظة، البارامترات بتكون لسه موجودة في الـ registers. بنقرأ:

- **`auxdev`**: الـ pointer لـ `struct auxiliary_device` — منه بنجيب الـ `name` والـ `id`.
- **`modname`**: اسم الـ module اللي بيسجّل الـ device — زي `"mlx5_core"` مثلاً.

الكود بيعمل `#ifdef` عشان يتعامل مع x86_64 وARM64 بشكل صح — كل architecture عندها أسماء registers مختلفة.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "__auxiliary_device_add",
    .pre_handler = handler_pre,
};
```

بنقول للـ kernel: "اوقف التنفيذ قبل `__auxiliary_device_add` وادّي التحكم لـ `handler_pre`". الـ `symbol_name` بيحل محل الحاجة لعنوان صريح — الـ kernel بيدوّر بنفسه في الـ kallsyms.

---

#### الـ `module_init`

بنستدعي `register_kprobe` اللي تزرع الـ breakpoint داخل الـ kernel. لو فشلت (مثلاً الـ `CONFIG_KPROBES` مش مفعّل أو الدالة inline)، نطبع error ونرجّع الـ error code ومنسجّلش الـ module.

---

#### الـ `module_exit`

**الـ unregister ضروري جداً** — لو ما عملناهوش، الـ kernel بيحاول يكمّل التنفيذ لـ `handler_pre` حتى بعد ما الـ module اتحمّل من الـ memory، وده بيعمل **kernel panic** فوري. دايماً الـ cleanup في الـ exit لازم يكون مرآة للـ init.

---

### الـ Makefile

```makefile
obj-m += auxbus_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### طريقة التشغيل والاختبار

```bash
# بناء الـ module
make

# تحميله
sudo insmod auxbus_probe.ko

# شوف الـ output في الـ kernel log
sudo dmesg | grep auxbus_probe

# لو عندك driver بيستخدم auxiliary bus (مثلاً mlx5 أو ice)
# حمّله وشوف السطور بتظهر تلقائياً
sudo modprobe mlx5_core    # مثال

# إزالة الـ module
sudo rmmod auxbus_probe
```

**مثال على الـ output المتوقع:**

```
[auxbus_probe] planted kprobe at 0xffffffffc0a12340 (__auxiliary_device_add)
[auxbus_probe] auxiliary_device_add called!
[auxbus_probe]   modname  = mlx5_core
[auxbus_probe]   dev name = eth
[auxbus_probe]   dev id   = 0
```

---

### ملاحظة مهمة

لو `__auxiliary_device_add` اتعملت **inline** من الـ compiler، الـ kprobe مش هتشتغل وبترجّع `-EINVAL`. في الحالة دي، الحل هو استخدام **tracepoint** لو موجود، أو الـ `CONFIG_KPROBES_ON_FTRACE` مع `ftrace`.
