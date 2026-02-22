## Phase 1: الصورة الكبيرة

# `auxiliary.c`

> **PATH**: `drivers/base/auxiliary.c`
> **Subsystem**: AUXILIARY BUS DRIVER (driver-core)
> **الوظيفة الأساسية**: تنفيذ "حافلة افتراضية" (virtual bus) تسمح للـ driver الواحد بتقسيم نفسه إلى أجهزة منطقية فرعية، كل واحدة تُشغَّل بـ driver منفصل.

---

### ما المشكلة اللي بيحلها هذا الملف؟

تخيل إنك عندك **كرت شبكة (NIC)** واحد داخل الكمبيوتر، لكن هذا الكرت يعمل حاجتين مختلفتين تماماً:

1. **شبكة عادية** (Ethernet) — يتحكم فيها driver الـ networking.
2. **RDMA** (نقل بيانات مباشر بين الذواكر بدون CPU) — يتحكم فيها driver خاص بالـ RDMA subsystem.

المشكلة: الكرت جهاز PCI واحد، بس الـ kernel محتاج درايفرين مختلفين من subsystemين مختلفين يشتغلوا عليه في نفس الوقت.

الحل القديم كان واحد من تلاتة:

| الحل | المشكلة |
|------|---------|
| **platform bus** | مخصص للأجهزة الحقيقية اللي DT/ACPI يعرفها، مش للأجهزة المنطقية |
| **MFD (Multi-Function Device)** | كمان يعتمد على أجهزة فيزيائية حقيقية |
| **monolithic driver** | driver عملاق واحد يعمل كل حاجة — صعب الصيانة والتوسعة |

**الـ Auxiliary Bus** جاء كحل نظيف: الـ parent driver (مثلاً driver الـ NIC) "يولد" أجهزة منطقية فرعية (auxiliary devices) وينشرها على حافلة خاصة، وكل subsystem يجيب driver خاص به ويربطه بالجهاز اللي يهمه.

---

### القصة كاملة — تخيل المشهد

```
┌─────────────────────────────────────────────────────┐
│                   Hardware: NIC PCI Card             │
└────────────────────┬────────────────────────────────┘
                     │  (PCI driver يشتغل هنا)
                     ▼
         ┌───────────────────────┐
         │   foo_net.ko (parent) │  ← الـ PCI driver الأصلي
         │  يعرف الـ hardware    │
         └──────┬───────┬────────┘
                │       │
    ينشئ        │       │  ينشئ
                ▼       ▼
      ┌──────────┐   ┌──────────┐
      │auxdev #0 │   │auxdev #1 │   ← أجهزة على الـ auxiliary bus
      │foo_net   │   │foo_rdma  │
      │.eth      │   │.rdma     │
      └────┬─────┘   └────┬─────┘
           │               │
           ▼               ▼
    ┌──────────┐    ┌──────────────┐
    │net driver│    │rdma driver   │  ← كل واحد في subsystem مختلف
    └──────────┘    └──────────────┘
```

الـ **match name** بيشتغل زي ما بيشتغل الـ modalias في USB:
- اسم الجهاز: `foo_net.eth.0`
- الـ driver اللي بيطابقه: أي driver عنده `{ .name = "foo_net.eth" }` في الـ `id_table`.

---

### إيه اللي بيعمله الملف ده تحديداً؟

الملف ده هو **قلب تنفيذ الـ Auxiliary Bus** — بيعمل تلاتة أشياء رئيسية:

#### 1. تسجيل الـ bus نفسه
```c
static const struct bus_type auxiliary_bus_type = {
    .name     = "auxiliary",
    .probe    = auxiliary_bus_probe,   // لما driver يلتقي بجهاز
    .remove   = auxiliary_bus_remove,  // لما الجهاز يتشال
    .shutdown = auxiliary_bus_shutdown,
    .match    = auxiliary_match,       // منطق المطابقة بين device و driver
    .uevent   = auxiliary_uevent,      // إرسال MODALIAS لـ udev
    .pm       = &auxiliary_dev_pm_ops, // إدارة الطاقة
};
```
الـ `auxiliary_bus_init()` بيسجل هذا الـ bus في بداية تشغيل الـ kernel.

#### 2. تسجيل الـ auxiliary_device (الجهاز الفرعي)
عملية من **3 خطوات**:

```c
// خطوة 1: تهيئة البنية
my_aux_dev->name   = "eth";
my_aux_dev->id     = 0;
my_aux_dev->dev.parent  = parent_dev;
my_aux_dev->dev.release = my_release_fn;

// خطوة 2: auxiliary_device_init() — يتحقق من الحقول ويعمل device_initialize()
auxiliary_device_init(my_aux_dev);

// خطوة 3: __auxiliary_device_add() — يسمي الجهاز ويضيفه للـ bus
// الاسم بيطلع: "foo_mod.eth.0"
auxiliary_device_add(my_aux_dev);
```

#### 3. تسجيل الـ auxiliary_driver (الدرايفر الفرعي)
```c
__auxiliary_driver_register(auxdrv, THIS_MODULE, KBUILD_MODNAME);
// بيسجل الدرايفر على الـ bus، وبيبدأ مطابقة مع الأجهزة الموجودة
```

#### 4. المطابقة (matching)
```c
// auxiliary_match_id: بيقارن اسم الجهاز (بدون آخر "." وبعده)
// مع أسماء الـ id_table في الدرايفر
// مثال: "foo_mod.eth.0" → prefix "foo_mod.eth" → يطابق { .name = "foo_mod.eth" }
```

#### 5. Helper functions جديدة (أبسط API)
```c
// auxiliary_device_create()  — ينشئ ويسجل الجهاز في خطوة واحدة
// auxiliary_device_destroy() — يحذف الجهاز
// __devm_auxiliary_device_create() — نسخة managed تتنظف أوتوماتيك مع الـ parent
```

---

### الفرق بين الـ API القديم والجديد

| العملية | API يدوي (3 خطوات) | API مبسط (helper) |
|---------|-------------------|------------------|
| إنشاء جهاز | `init` + `add` يدوياً | `auxiliary_device_create()` |
| حذف جهاز | `delete` + `uninit` يدوياً | `auxiliary_device_destroy()` |
| managed lifecycle | `devm_add_action_or_reset()` يدوياً | `devm_auxiliary_device_create()` |

---

### نقطة مهمة: إدارة الذاكرة

الـ Auxiliary Bus **لا يدير الذاكرة** للأجهزة — ده مسؤولية الـ parent driver تماماً. لو الـ parent driver راح وعنده auxiliary devices لسه مسجلة، هيحصل مشكلة. لذلك اتفضل دايماً استخدام `devm_*` للتنظيف الأوتوماتيكي.

---

### الملفات المكوِّنة لهذا الـ Subsystem

| الملف | الدور |
|-------|-------|
| `drivers/base/auxiliary.c` | **التنفيذ الأساسي** — هذا الملف |
| `include/linux/auxiliary_bus.h` | تعريف الـ structs والـ API العامة (`auxiliary_device`, `auxiliary_driver`, الـ macros) |
| `Documentation/driver-api/auxiliary_bus.rst` | توثيق رسمي شامل للـ API |
| `rust/helpers/auxiliary.c` | Rust bindings helpers |
| `rust/kernel/auxiliary.rs` | واجهة الـ Rust للـ auxiliary bus |
| `samples/rust/rust_driver_auxiliary.rs` | مثال كامل بالـ Rust |

### ملفات مرتبطة يجب معرفتها

| الملف | العلاقة |
|-------|---------|
| `drivers/base/bus.c` | البنية التحتية للـ bus التي يستخدمها الـ auxiliary bus |
| `drivers/base/driver.c` | تسجيل الـ drivers بشكل عام |
| `drivers/base/dd.c` | منطق الـ probe/remove (device-driver binding) |
| `include/linux/device.h` | تعريف `struct device` و `struct device_driver` |
| `include/linux/mod_devicetable.h` | تعريف `struct auxiliary_device_id` |

### أمثلة على من يستخدم الـ Auxiliary Bus في الـ kernel

- **Sound Open Firmware (SOF)**: يقسم DSP الصوتي إلى HDMI, SoundWire, مايكروفون، سماعات — كل جزء جهاز auxiliary منفصل.
- **Intel ICE (NIC)**: يصدر auxiliary devices لـ RDMA وـ SIOV.
- **RDMA subsystem**: يستقبل auxiliary devices من كروت الشبكة ويربطها بـ drivers الـ InfiniBand/RoCE.
## Phase 2: شرح الـ Auxiliary Bus Framework

---

### المشكلة — ليش احتجنا الـ Auxiliary Bus أصلاً؟

تخيل عندك جهاز PCI واحد — مثلاً كرت شبكة (NIC) من Intel — لكن هذا الجهاز الواحد يعمل **أشياء كثيرة جداً** في نفس الوقت:

- يشتغل كـ **network card** عادي (يرسل ويستقبل بيانات)
- يدعم **RDMA** (Remote Direct Memory Access) — بروتوكول خاص بالـ high-performance networking
- عنده firmware صوتي (Sound Open Firmware) يتحكم في HDMI audio + Soundwire + مايكروفون + سماعات

السؤال: كيف تكتب driver لهذا الجهاز؟

**الخيار الأول — driver وحيد "monolithic":**
اكتب driver واحد ضخم يتعامل مع كل الوظائف. المشكلة؟ هذا الـ driver سيصبح كارثة:
- أي فريق يشتغل على الجزء الصوتي يضطر يفهم كود الـ RDMA
- أي bug في الصوت يمكن يكسر الشبكة
- الكود يصبح غير قابل للصيانة أو الـ reuse

**الخيار الثاني — platform_device أو MFD (Multi-Function Device):**
- الـ **MFD** framework موجود لكنه يفترض أن كل sub-device هو **جهاز فيزيائي حقيقي** يظهر في الـ Device Tree أو ACPI
- الـ sub-functions اللي نحكي عنها **مش أجهزة فيزيائية مستقلة** — هي مجرد وظائف منطقية داخل نفس الـ PCI chip
- إذن MFD مش مناسب هنا

**الخيار الثالث — الـ Auxiliary Bus:**
الكيرنل ابتكر "باص وهمي" اسمه **Auxiliary Bus**. الفكرة: الـ parent driver (مثلاً driver الـ NIC) يـ"يصنع" أجهزة منطقية وهمية (`auxiliary_device`) ويسجلها على هذا الباص. كل جهاز وهمي يمثل وظيفة واحدة من وظائف الـ chip. بعدها كل subsystem (RDMA team, audio team, إلخ) يكتب `auxiliary_driver` خاص فيه يـ"يلتقط" الجهاز الوهمي المناسب ويشتغل معه.

---

### الحل — كيف يشتغل الـ Auxiliary Bus؟

الـ Auxiliary Bus يعتمد على نفس الـ **Linux Device Model** الاعتيادي، لكن مع "باص" افتراضي مخصص. المبدأ:

1. **Parent driver** (المسجِّل): يعرف الـ chip كاملاً، يصنع `auxiliary_device` لكل وظيفة ويسجلها
2. **الـ Auxiliary Bus**: يحتفظ بقائمة الأجهزة، يربطها بالـ drivers المناسبة
3. **Auxiliary driver** (المستهلك): يعلن عن الأجهزة اللي يدعمها، الكيرنل يستدعي `probe()` تلقائياً لما يجد تطابق

نظام المطابقة (matching) مبني على **الأسماء** — اسم الجهاز الوهمي = `modname.devname.id`، مثلاً: `foo_mod.foo_dev.0`

---

### التشبيه الحياتي

تخيل **شركة كبيرة** (الـ PCI chip) فيها أقسام مختلفة:
- قسم المبيعات
- قسم الـ IT
- قسم المحاسبة

الشركة لديها **مدير عام واحد** (الـ parent driver) يعرف الصورة الكاملة. لكن كل قسم عنده **مدير قسم مستقل** (auxiliary driver) متخصص في مجاله.

الـ **Auxiliary Bus** هو **لوحة الإعلانات** في الشركة: المدير العام يعلق على اللوحة "القسم الفلاني موجود"، ومدير القسم يمر، يشوف اسمه على اللوحة، ويقول "هذا قسمي، أنا مسؤول عنه".

---

### المعمارية الكاملة — Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Hardware (PCI Chip)                       │
│         ┌──────────┬──────────┬──────────┬──────────┐           │
│         │  Network │   RDMA   │  Audio   │  Crypto  │           │
│         │ Function │ Function │ Function │ Function │           │
│         └──────────┴──────────┴──────────┴──────────┘           │
└──────────────────────────┬──────────────────────────────────────┘
                           │  PCI bus
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Parent PCI Driver                           │
│                    (e.g., intel_nic.ko)                          │
│                                                                  │
│  يعرف الـ chip كاملاً، يصنع auxiliary_devices لكل وظيفة        │
│                                                                  │
│   auxiliary_device_init()  ──►  auxiliary_device_add()          │
└────────────┬──────────────────────────┬─────────────────────────┘
             │ registers                │ registers
             ▼                          ▼
┌────────────────────────────────────────────────────────────────┐
│                      Auxiliary Bus                              │
│              (virtual bus — "auxiliary" type)                   │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │ intel_nic.rdma.0 │  │intel_nic.audio.0 │  │intel_nic.    │  │
│  │ (auxiliary_dev)  │  │ (auxiliary_dev)  │  │crypto.0      │  │
│  └────────┬─────────┘  └────────┬─────────┘  └──────┬───────┘  │
└───────────┼─────────────────────┼────────────────────┼──────────┘
            │ match + probe       │ match + probe       │
            ▼                     ▼                     ▼
┌───────────────┐      ┌──────────────────┐   ┌────────────────┐
│  RDMA Driver  │      │   Audio Driver   │   │  Crypto Driver │
│ (rdma_mod.ko) │      │ (sof_audio.ko)   │   │(crypto_mod.ko) │
│               │      │                  │   │                │
│ probe()       │      │ probe()          │   │ probe()        │
│ remove()      │      │ remove()         │   │ remove()       │
└───────────────┘      └──────────────────┘   └────────────────┘
```

---

### الـ Structs الأساسية — كيف تترابط مع بعض

#### 1. `struct auxiliary_device` — الجهاز الوهمي

```c
struct auxiliary_device {
    struct device dev;      /* الـ device العادي في الكيرنل — يرث كل شيء */
    const char *name;       /* اسم الوظيفة، مثلاً "rdma" أو "audio" */
    u32 id;                 /* رقم تمييزي لو في أكثر من جهاز بنفس الاسم */
    struct {
        struct xarray irqs;     /* قائمة الـ IRQs الخاصة بهذا الجهاز */
        struct mutex lock;      /* حماية عمليات الـ sysfs */
        bool irq_dir_exists;    /* هل مجلد irqs موجود في sysfs؟ */
    } sysfs;
};
```

**الاسم الكامل للجهاز** يُصنع هكذا في `__auxiliary_device_add()`:
```c
dev_set_name(dev, "%s.%s.%d", modname, auxdev->name, auxdev->id);
// النتيجة مثلاً: "intel_nic.rdma.0"
```

#### 2. `struct auxiliary_driver` — الـ Driver المتخصص

```c
struct auxiliary_driver {
    int   (*probe)(struct auxiliary_device *auxdev,
                   const struct auxiliary_device_id *id);  /* اللي يشتغل لما يجد match */
    void  (*remove)(struct auxiliary_device *auxdev);       /* تنظيف عند الإزالة */
    void  (*shutdown)(struct auxiliary_device *auxdev);     /* إيقاف تشغيل النظام */
    int   (*suspend)(struct auxiliary_device *auxdev, pm_message_t state);
    int   (*resume)(struct auxiliary_device *auxdev);
    const char *name;                          /* اسم الـ driver */
    struct device_driver driver;               /* الـ driver القياسي المضمَّن */
    const struct auxiliary_device_id *id_table; /* جدول الأسماء اللي يدعمها */
};
```

#### 3. `struct auxiliary_device_id` — جدول المطابقة

```c
struct auxiliary_device_id {
    char name[40];              /* "modname.devname" — الاسم اللي يبحث عنه الـ driver */
    kernel_ulong_t driver_data; /* بيانات إضافية اختيارية */
};
```

مثال:
```c
static const struct auxiliary_device_id my_id_table[] = {
    { .name = "intel_nic.rdma" },  /* يطابق أي intel_nic.rdma.X */
    { }                             /* نهاية الجدول */
};
```

#### 4. `struct bus_type auxiliary_bus_type` — الباص نفسه

```c
static const struct bus_type auxiliary_bus_type = {
    .name     = "auxiliary",
    .probe    = auxiliary_bus_probe,    /* يستدعي probe() الخاص بالـ driver */
    .remove   = auxiliary_bus_remove,
    .shutdown = auxiliary_bus_shutdown,
    .match    = auxiliary_match,        /* يقارن اسم الجهاز باسم الـ driver */
    .uevent   = auxiliary_uevent,       /* يرسل حدث لـ udev */
    .pm       = &auxiliary_dev_pm_ops,  /* power management */
};
```

---

### كيف تترابط الـ Structs مع بعض — رسم العلاقات

```
┌─────────────────────────────────────────────────────┐
│              struct auxiliary_device                 │
│  ┌──────────────────────────────────────────────┐   │
│  │  struct device dev  ◄──── الأساس، موروث من   │   │
│  │    .bus ──────────────────► auxiliary_bus_type│   │
│  │    .parent ────────────────► parent PCI device│   │
│  │    .release ───────────────► دالة تحرير الذاكرة│  │
│  └──────────────────────────────────────────────┘   │
│  const char *name  ═══════► "rdma"                  │
│  u32 id            ═══════► 0                        │
│     ↓ combined with modname → "intel_nic.rdma.0"    │
└──────────────────────┬──────────────────────────────┘
                       │ registered on
                       ▼
┌─────────────────────────────────────────────────────┐
│            struct bus_type (auxiliary)               │
│  .match() ─── يقارن اسم الجهاز بـ id_table الـ driver│
│  .probe() ─── يستدعي auxdrv->probe() عند التطابق   │
└──────────────────────┬──────────────────────────────┘
                       │ finds match in
                       ▼
┌─────────────────────────────────────────────────────┐
│             struct auxiliary_driver                  │
│  ┌──────────────────────────────────────────────┐   │
│  │  struct device_driver driver ◄── مضمَّن      │   │
│  │    .bus ──────────────────► auxiliary_bus_type│   │
│  └──────────────────────────────────────────────┘   │
│  id_table ═══► [ {.name="intel_nic.rdma"}, {} ]     │
│  probe()  ═══► my_rdma_probe()                       │
│  remove() ═══► my_rdma_remove()                      │
└─────────────────────────────────────────────────────┘
```

---

### عملية الـ Matching — كيف يجد الكيرنل الـ Driver المناسب؟

الدالة `auxiliary_match_id()` هي قلب عملية المطابقة:

```c
static const struct auxiliary_device_id *auxiliary_match_id(
    const struct auxiliary_device_id *id,
    const struct auxiliary_device *auxdev)
{
    const char *auxdev_name = dev_name(&auxdev->dev);
    /* اسم الجهاز: "intel_nic.rdma.0" */

    const char *p = strrchr(auxdev_name, '.');
    /* p يشير إلى آخر نقطة: قبل الرقم "0" */

    int match_size = p - auxdev_name;
    /* match_size = طول "intel_nic.rdma" بدون الـ ".0" */

    for (; id->name[0]; id++) {
        /* قارن اسم الجهاز (بدون الرقم) مع كل إدخال في الجدول */
        if (strlen(id->name) == match_size &&
            !strncmp(auxdev_name, id->name, match_size))
            return id;  /* وجدنا تطابق! */
    }
    return NULL;
}
```

**مثال عملي:**
- اسم الجهاز: `"intel_nic.rdma.0"`
- اسم في الـ id_table: `"intel_nic.rdma"`
- الدالة تقطع الجزء بعد آخر نقطة (`.0`) وتقارن الباقي ← تطابق!

هذا يعني driver واحد يمكن يخدم **أجهزة متعددة** بنفس الاسم: `rdma.0`, `rdma.1`, `rdma.2` إلخ.

---

### دورة حياة الـ Auxiliary Device — من الولادة للوفاة

#### مرحلة التسجيل (3 خطوات)

```
Parent Driver
     │
     ├─► [الخطوة 1] تهيئة الـ struct
     │       auxdev->name = "rdma";
     │       auxdev->id = 0;
     │       auxdev->dev.parent = pci_dev;
     │       auxdev->dev.release = my_release_fn;
     │
     ├─► [الخطوة 2] auxiliary_device_init(auxdev)
     │       ├── يتحقق أن parent و name غير NULL
     │       ├── يربط الجهاز بالـ auxiliary_bus_type
     │       ├── يستدعي device_initialize()
     │       └── يهيئ mutex الـ sysfs
     │
     └─► [الخطوة 3] auxiliary_device_add(auxdev)
             ├── يصنع اسم كامل: "modname.name.id"
             └── يستدعي device_add() ← الجهاز يظهر على الباص
```

#### مرحلة الـ Probe (تلقائية)

```
الكيرنل (عند تسجيل device أو driver):
     │
     ├─► auxiliary_match() ← هل في driver مناسب؟
     │       └── auxiliary_match_id() ← مقارنة الأسماء
     │
     └─► auxiliary_bus_probe()
             ├── dev_pm_domain_attach() ← ربط الـ power domain
             └── auxdrv->probe(auxdev, id) ← استدعاء الـ driver
```

#### مرحلة الإزالة

```
Parent Driver يقرر إزالة الجهاز:
     │
     ├─► auxiliary_device_delete(auxdev)
     │       └── device_del() ← إزالة من الباص، يستدعي remove()
     │
     └─► auxiliary_device_uninit(auxdev)
             ├── mutex_destroy()
             └── put_device() ← تخفيض الـ reference count
                     └── عند وصول العداد لـ 0 → release() → kfree()
```

---

### نمط الـ Container — كيف يشارك الـ Parent والـ Child البيانات؟

هذا من أذكى أجزاء الـ framework. الـ parent driver يضمِّن (`embed`) الـ `auxiliary_device` داخل struct أكبر يحتوي على البيانات المشتركة:

```c
/* في الـ header المشترك بين الـ parent وكل الـ auxiliary drivers */
struct my_nic_rdma_ctx {
    struct auxiliary_device auxdev;   /* يجب أن يكون أول عضو أو يُعرف موضعه */

    /* البيانات المشتركة مع RDMA driver */
    void __iomem *rdma_bar;           /* عنوان الـ MMIO للـ RDMA */
    struct rdma_hw_caps caps;         /* قدرات الـ hardware */
    spinlock_t lock;

    /* callbacks من الـ parent للـ child */
    int (*send_cmd)(struct my_nic_rdma_ctx *ctx, u32 cmd);
};
```

الـ RDMA driver في `probe()`:

```c
static int rdma_probe(struct auxiliary_device *auxdev,
                      const struct auxiliary_device_id *id)
{
    /* container_of يرجع للـ struct الأكبر من داخل auxiliary_device */
    struct my_nic_rdma_ctx *ctx =
        container_of(auxdev, struct my_nic_rdma_ctx, auxdev);

    /* الآن ctx->rdma_bar و ctx->caps متاحة! */
    /* الـ parent أعدّها قبل استدعاء auxiliary_device_add() */
}
```

```
ذاكرة الـ Heap:
┌────────────────────────────────────────┐
│       struct my_nic_rdma_ctx           │
│  ┌─────────────────────────────────┐   │
│  │   struct auxiliary_device       │   │ ◄── auxdev pointer
│  │      struct device dev          │   │
│  │      name = "rdma"              │   │
│  │      id = 0                     │   │
│  └─────────────────────────────────┘   │
│  void __iomem *rdma_bar ════► [MMIO]  │
│  struct rdma_hw_caps caps              │
│  int (*send_cmd)(...)                  │
└────────────────────────────────────────┘
         ▲
         │  container_of(auxdev, struct my_nic_rdma_ctx, auxdev)
         │
    [rdma driver]
```

---

### الـ Power Management في الـ Auxiliary Bus

الـ framework يوفر PM تلقائي عبر:

```c
static const struct dev_pm_ops auxiliary_dev_pm_ops = {
    SET_RUNTIME_PM_OPS(
        pm_generic_runtime_suspend,  /* suspend وقت عدم الاستخدام */
        pm_generic_runtime_resume,   /* resume عند الحاجة */
        NULL
    )
    SET_SYSTEM_SLEEP_PM_OPS(
        pm_generic_suspend,          /* suspend عند نوم النظام */
        pm_generic_resume            /* resume عند استيقاظ النظام */
    )
};
```

وعند الـ probe يتم ربط الجهاز بـ **PM Domain** الخاص فيه:

```c
ret = dev_pm_domain_attach(dev,
    PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF);
```

هذا يعني الجهاز الوهمي يشارك في دورة الطاقة مع parent الفيزيائي بشكل صحيح.

---

### الـ devm Pattern — إدارة الذاكرة التلقائية

الـ framework يوفر `devm_auxiliary_device_create()`:

```c
/* بدلاً من: */
auxdev = auxiliary_device_create(dev, modname, devname, data, id);
devm_add_action_or_reset(dev, auxiliary_device_destroy, auxdev);

/* اكتب مباشرة: */
auxdev = devm_auxiliary_device_create(dev, devname, data);
```

عندما يُحذف الـ parent device، الكيرنل يستدعي `auxiliary_device_destroy()` تلقائياً — مثل `smart pointer` في C++.

---

### أين يجلس الـ Auxiliary Bus في الكيرنل؟

```
Linux Kernel
│
├── drivers/
│   ├── base/
│   │   ├── auxiliary.c          ◄── هنا تعريف الـ bus نفسه
│   │   ├── driver.c
│   │   ├── bus.c
│   │   └── ...
│   │
│   ├── net/
│   │   └── ethernet/intel/
│   │       └── ice/             ◄── مثال: parent driver يصنع auxiliary_devices
│   │
│   └── infiniband/              ◄── مثال: auxiliary_driver يستهلك الأجهزة
│
├── include/linux/
│   └── auxiliary_bus.h          ◄── تعريف الـ structs والـ macros
│
└── include/linux/
    └── mod_devicetable.h        ◄── تعريف auxiliary_device_id
```

---

### ليش مش MFD أو Platform Bus؟

| المعيار | Platform Bus | MFD | Auxiliary Bus |
|---------|-------------|-----|---------------|
| يحتاج DT/ACPI؟ | نعم | نعم | **لا** |
| الأجهزة فيزيائية؟ | نعم | نعم | **لا — منطقية فقط** |
| مشاركة بيانات بين parent وchild | صعبة | محدودة | **سهلة جداً (container_of)** |
| يدعم dynamic creation وقت التشغيل؟ | محدود | محدود | **نعم** |
| مناسب لـ NIC + RDMA pattern؟ | لا | لا | **نعم** |

---

### الـ MODALIAS وudev Integration

عند تسجيل جهاز جديد، الـ framework يرسل **uevent** لـ udev:

```c
static int auxiliary_uevent(const struct device *dev, struct kobj_uevent_env *env)
{
    name = dev_name(dev);          /* مثلاً: "intel_nic.rdma.0" */
    p = strrchr(name, '.');        /* آخر نقطة — قبل الرقم */

    add_uevent_var(env, "MODALIAS=%s%.*s",
                   AUXILIARY_MODULE_PREFIX,    /* "auxiliary:" */
                   (int)(p - name), name);     /* "intel_nic.rdma" */
    /* النتيجة: MODALIAS=auxiliary:intel_nic.rdma */
}
```

هذا يسمح لـ **udev/modprobe** بتحميل الـ kernel module المناسب تلقائياً لما يظهر الجهاز الوهمي الجديد — نفس الآلية المستخدمة مع الأجهزة الفيزيائية الحقيقية.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### الـ Config Options المهمة

| Option | الأثر |
|--------|--------|
| `CONFIG_SYSFS` | لو مفعّل: تشتغل `auxiliary_device_sysfs_irq_add` و`auxiliary_device_sysfs_irq_remove` حقيقيين. لو مش مفعّل: بيرجعوا stubs فارغة |

#### الـ Macros والـ Constants المهمة

| Macro / Constant | القيمة / الوصف |
|---|---|
| `AUXILIARY_NAME_SIZE` | `40` — الحد الأقصى لطول اسم الـ device في الـ id table |
| `AUXILIARY_MODULE_PREFIX` | `"auxiliary:"` — prefix الـ MODALIAS في udev |
| `auxiliary_device_add(auxdev)` | Wrapper لـ `__auxiliary_device_add` يحقن `KBUILD_MODNAME` تلقائيًا |
| `auxiliary_driver_register(auxdrv)` | Wrapper لـ `__auxiliary_driver_register` يحقن `THIS_MODULE` و`KBUILD_MODNAME` |
| `devm_auxiliary_device_create(dev, devname, platform_data)` | Wrapper لـ `__devm_auxiliary_device_create` بـ id=0 وmodname تلقائي |
| `module_auxiliary_driver(__auxiliary_driver)` | يستغني عن module_init/exit بالكامل |
| `PD_FLAG_ATTACH_POWER_ON` | flag للـ PM domain: اربط الـ domain لما الجهاز يشتغل |
| `PD_FLAG_DETACH_POWER_OFF` | flag للـ PM domain: افصل الـ domain لما الجهاز يقفل |

#### الـ PM Ops المستخدمة

| Op | الوظيفة |
|---|---|
| `pm_generic_runtime_suspend` | suspend وقت runtime |
| `pm_generic_runtime_resume` | resume وقت runtime |
| `pm_generic_suspend` | suspend system sleep |
| `pm_generic_resume` | resume system sleep |

---

### الـ Structs المهمة

#### 1. `struct auxiliary_device`

**الغرض:** يمثّل جهازًا فرعيًا (sub-device) على الـ auxiliary bus. فكّره زي "شريحة" من وظيفة الجهاز الأب — مش جهاز فيزيائي مستقل، بل جزء منطقي تم تقسيمه.

```c
struct auxiliary_device {
    struct device dev;      /* الـ device العام من kernel — يحتوي على parent, bus, release */
    const char *name;       /* اسم الجهاز الفرعي، مثل "foo_dev" */
    u32 id;                 /* رقم فريد لو في أكثر من device بنفس الاسم */
    struct {
        struct xarray irqs;         /* مصفوفة الـ IRQs المرتبطة بالجهاز في sysfs */
        struct mutex lock;          /* mutex يحمي عملية إنشاء irq في sysfs */
        bool irq_dir_exists;        /* هل مجلد "irqs" موجود في sysfs؟ */
    } sysfs;
};
```

| Field | النوع | الوظيفة |
|---|---|---|
| `dev` | `struct device` | الـ device العام — يربط الجهاز بالـ kernel device model |
| `name` | `const char *` | الجزء الثاني من اسم الـ match: `modname.name` |
| `id` | `u32` | رقم تمييزي لو في أكثر من instance |
| `sysfs.irqs` | `struct xarray` | تتبع IRQs عبر sysfs |
| `sysfs.lock` | `struct mutex` | يحمي عمليات sysfs |
| `sysfs.irq_dir_exists` | `bool` | حالة مجلد irqs |

**الروابط:** يحتوي على `struct device` (embedded) → هو اللي يربطه بالـ bus. الـ `dev.parent` يشير لجهاز الأب.

---

#### 2. `struct auxiliary_driver`

**الغرض:** يمثّل الـ driver اللي بيشتغل مع auxiliary devices. زي "مدير متخصص" لكل نوع من الأجهزة الفرعية.

```c
struct auxiliary_driver {
    int (*probe)(struct auxiliary_device *auxdev,
                 const struct auxiliary_device_id *id); /* إلزامي */
    void (*remove)(struct auxiliary_device *auxdev);    /* اختياري */
    void (*shutdown)(struct auxiliary_device *auxdev);  /* اختياري */
    int (*suspend)(struct auxiliary_device *auxdev, pm_message_t state); /* اختياري */
    int (*resume)(struct auxiliary_device *auxdev);     /* اختياري */
    const char *name;                    /* اسم الـ driver (اختياري) */
    struct device_driver driver;         /* الـ driver العام من kernel */
    const struct auxiliary_device_id *id_table; /* جدول الأسماء اللي يتطابق معها — إلزامي */
};
```

| Field | الإلزامية | الوظيفة |
|---|---|---|
| `probe` | **إلزامي** | يُستدعى لما يُعثر على device متطابق |
| `remove` | اختياري | يُستدعى لما يُحذف الـ device |
| `shutdown` | اختياري | يُستدعى وقت إيقاف النظام |
| `suspend` | اختياري | power management |
| `resume` | اختياري | power management |
| `name` | اختياري | لو موجود: `modname.name`، لو لأ: `modname` فقط |
| `driver` | داخلي | يحتوي على `.name`, `.owner`, `.bus`, `.mod_name` |
| `id_table` | **إلزامي** | قائمة الـ match names |

**الروابط:** يحتوي على `struct device_driver` (embedded) → هو اللي يربطه بالـ bus.

---

#### 3. `struct auxiliary_device_id`

**الغرض:** سجل في جدول التطابق. زي "بطاقة هوية" الجهاز اللي الـ driver بيقبلها.

```c
struct auxiliary_device_id {
    char name[AUXILIARY_NAME_SIZE]; /* "modname.devname" — 40 حرف كحد أقصى */
    kernel_ulong_t driver_data;     /* بيانات إضافية اختيارية للـ driver */
};
```

| Field | الوظيفة |
|---|---|
| `name[40]` | الاسم المركّب `"foo_mod.foo_dev"` — يُستخدم في المطابقة |
| `driver_data` | قيمة اختيارية يمرّرها الـ driver لـ probe() |

---

#### 4. `struct bus_type auxiliary_bus_type` (static)

**الغرض:** تعريف الـ auxiliary bus نفسه. هذا هو "قلب" الملف — بيسجّل النوع الجديد من الـ bus في الـ kernel.

```c
static const struct bus_type auxiliary_bus_type = {
    .name     = "auxiliary",
    .probe    = auxiliary_bus_probe,
    .remove   = auxiliary_bus_remove,
    .shutdown = auxiliary_bus_shutdown,
    .match    = auxiliary_match,
    .uevent   = auxiliary_uevent,
    .pm       = &auxiliary_dev_pm_ops,
};
```

---

#### 5. `struct dev_pm_ops auxiliary_dev_pm_ops` (static)

**الغرض:** تعريف سلوك الـ Power Management للـ auxiliary bus.

```c
static const struct dev_pm_ops auxiliary_dev_pm_ops = {
    SET_RUNTIME_PM_OPS(pm_generic_runtime_suspend,
                       pm_generic_runtime_resume, NULL)
    SET_SYSTEM_SLEEP_PM_OPS(pm_generic_suspend, pm_generic_resume)
};
```

---

### مخططات العلاقات بين الـ Structs

```
الـ Parent Driver (مثلاً PCI driver)
        │
        │ يخلق ويملك
        ▼
┌─────────────────────────────────┐
│       struct foo (مثال)         │
│  ┌──────────────────────────┐   │
│  │  struct auxiliary_device │   │
│  │  ┌────────────────────┐  │   │
│  │  │   struct device    │  │   │
│  │  │  .parent ──────────┼──┼───┼──► (parent device)
│  │  │  .bus ─────────────┼──┼───┼──► auxiliary_bus_type
│  │  │  .release          │  │   │
│  │  └────────────────────┘  │   │
│  │  .name = "foo_dev"       │   │
│  │  .id   = 0               │   │
│  │  .sysfs.lock (mutex)     │   │
│  │  .sysfs.irqs (xarray)    │   │
│  └──────────────────────────┘   │
│  .connect (callback)            │
│  .disconnect (callback)         │
│  .data (shared object ptr)      │
└─────────────────────────────────┘

        ▲ تطابق عبر الاسم
        │
┌───────────────────────────────────┐
│      struct my_driver (مثال)      │
│  ┌────────────────────────────┐   │
│  │   struct auxiliary_driver  │   │
│  │  ┌──────────────────────┐  │   │
│  │  │ struct device_driver │  │   │
│  │  │  .bus ───────────────┼──┼───┼──► auxiliary_bus_type
│  │  │  .name = "mod.drv"   │  │   │
│  │  └──────────────────────┘  │   │
│  │  .probe   = my_probe       │   │
│  │  .remove  = my_remove      │   │
│  │  .id_table ────────────────┼───┼──► auxiliary_device_id[]
│  └────────────────────────────┘   │
│  .ops (custom callbacks)          │
└───────────────────────────────────┘

struct auxiliary_device_id[] :
┌─────────────────────────┐
│ { .name = "foo_mod.foo_dev" } │
│ { .name = "foo_mod.bar_dev" } │
│ { }  ← نهاية الجدول          │
└─────────────────────────┘
```

---

### مخططات دورة الحياة

#### دورة حياة `auxiliary_device` — التسجيل والحذف

```
[الـ Parent Driver يريد إنشاء sub-device]
        │
        ▼
  تخصيص الذاكرة
  auxdev = kzalloc(sizeof(*auxdev), GFP_KERNEL)
        │
        ▼
  ملء الحقول الإلزامية:
    auxdev->name      = "foo_dev"
    auxdev->id        = 0
    auxdev->dev.parent = my_parent_dev
    auxdev->dev.release = my_release_fn   ◄── إلزامي!
        │
        ▼
  [الخطوة 2] auxiliary_device_init(auxdev)
    ├── تحقق: dev.parent != NULL
    ├── تحقق: name != NULL
    ├── dev.bus = &auxiliary_bus_type
    ├── device_initialize(&auxdev->dev)
    └── mutex_init(&auxdev->sysfs.lock)
        │
        ├── فشل؟ → free auxdev مباشرةً (الـ release لم يُسجَّل بعد)
        │
        ▼
  [الخطوة 3] auxiliary_device_add(auxdev)  أو  __auxiliary_device_add(auxdev, modname)
    ├── dev_set_name(dev, "%s.%s.%d", modname, name, id)
    │   مثال: "foo_mod.foo_dev.0"
    └── device_add(dev)   ← يُسجّل في الـ kernel ويُطلق المطابقة
        │
        ├── فشل؟ → auxiliary_device_uninit(auxdev) [يستدعي put_device → release]
        │
        ▼
  [الـ Device مسجّل على الـ bus]
  الـ kernel يبحث عن driver متطابق → يستدعي auxiliary_bus_probe()
        │
        ▼
  [وقت الحذف]
  auxiliary_device_delete(auxdev)   → device_del(dev)
        │
        ▼
  auxiliary_device_uninit(auxdev)
    ├── mutex_destroy(&auxdev->sysfs.lock)
    └── put_device(&auxdev->dev)
              │
              ▼ (لما كل references تنتهي)
         dev->release(dev)   ← الـ parent driver يحرر الذاكرة هنا
```

#### دورة حياة `auxiliary_driver` — التسجيل والإلغاء

```
[الـ Auxiliary Driver Module يُحمَّل]
        │
        ▼
  auxiliary_driver_register(auxdrv)
  (أو module_auxiliary_driver macro)
        │
        ▼
  __auxiliary_driver_register(auxdrv, THIS_MODULE, KBUILD_MODNAME)
    ├── تحقق: probe != NULL   (WARN_ON)
    ├── تحقق: id_table != NULL (WARN_ON)
    ├── لو auxdrv->name موجود:
    │     driver.name = kasprintf("%s.%s", modname, auxdrv->name)
    │   لو لأ:
    │     driver.name = kasprintf("%s", modname)
    ├── driver.owner    = owner (THIS_MODULE)
    ├── driver.bus      = &auxiliary_bus_type
    ├── driver.mod_name = modname
    └── driver_register(&auxdrv->driver)
              │
              ▼
        الـ kernel يمسح الـ devices الموجودة → يطابق → probe()
        │
        ▼
  [الـ Driver يعمل ويخدم الـ devices]
        │
        ▼
  [الـ Module يُفرَّغ]
  auxiliary_driver_unregister(auxdrv)
    ├── driver_unregister(&auxdrv->driver)
    └── kfree(auxdrv->driver.name)   ← المخصَّص في kasprintf
```

#### دورة حياة `auxiliary_device_create` (الطريقة المبسَّطة)

```
[بدلاً من الخطوات الثلاث اليدوية]
        │
        ▼
  auxiliary_device_create(dev, modname, devname, platform_data, id)
    ├── kzalloc(auxdev)
    ├── ملء الحقول (parent, name, id, platform_data, release)
    ├── device_set_of_node_from_dev()
    ├── auxiliary_device_init()
    └── __auxiliary_device_add()
        │
        ▼
  [لإدارة تلقائية مع devm]
  devm_auxiliary_device_create(dev, devname, platform_data)
    └── __devm_auxiliary_device_create(...)
          ├── auxiliary_device_create(...)
          └── devm_add_action_or_reset(dev, auxiliary_device_destroy, auxdev)
                  │
                  ▼ (لما dev الأب يُحرَّر)
            auxiliary_device_destroy(auxdev) تُستدعى تلقائيًا
              ├── auxiliary_device_delete()
              └── auxiliary_device_uninit()
```

---

### مخططات تدفق الاستدعاءات

#### 1. تدفق المطابقة (Matching Flow)

```
bus يستلم device أو driver جديد
        │
        ▼
  auxiliary_match(dev, drv)
    ├── to_auxiliary_dev(dev)   ← container_of
    ├── to_auxiliary_drv(drv)   ← container_of
    └── auxiliary_match_id(auxdrv->id_table, auxdev)
              │
              ▼
        auxdev_name = dev_name(&auxdev->dev)
        مثال: "foo_mod.foo_dev.0"
              │
              ▼
        p = strrchr(name, '.')   ← آخر نقطة
        match_size = p - name    ← "foo_mod.foo_dev"
              │
              ▼
        loop على id_table:
          if strlen(id->name) == match_size &&
             strncmp(name, id->name, match_size) == 0:
                return id   ✓ تطابق!
          else:
                return NULL  ✗ لا تطابق
```

#### 2. تدفق الـ Probe

```
kernel يقرر ربط device بـ driver
        │
        ▼
  auxiliary_bus_probe(dev)
    ├── to_auxiliary_drv(dev->driver)
    ├── to_auxiliary_dev(dev)
    ├── dev_pm_domain_attach(dev,
    │     PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF)
    │       │
    │       ├── فشل؟ → dev_warn() + return error
    │       │
    │       ▼
    └── auxdrv->probe(auxdev,
              auxiliary_match_id(auxdrv->id_table, auxdev))
                  │
                  ▼
            [كود الـ Driver يشتغل]
            يستطيع استخدام container_of(auxdev) للوصول لـ parent data
```

#### 3. تدفق الـ Remove

```
kernel يريد فصل device عن driver
        │
        ▼
  auxiliary_bus_remove(dev)
    ├── to_auxiliary_drv(dev->driver)
    ├── to_auxiliary_dev(dev)
    └── if auxdrv->remove:
            auxdrv->remove(auxdev)
                │
                ▼
          [الـ Driver ينظف موارده]
          pm_domain يُفصل تلقائيًا (PD_FLAG_DETACH_POWER_OFF)
```

#### 4. تدفق الـ Shutdown

```
النظام يُوقَّف
        │
        ▼
  auxiliary_bus_shutdown(dev)
    ├── if dev->driver:
    │       auxdrv = to_auxiliary_drv(dev->driver)
    │       auxdev = to_auxiliary_dev(dev)
    │
    └── if auxdrv && auxdrv->shutdown:
            auxdrv->shutdown(auxdev)

  ملاحظة: الـ shutdown آمن حتى لو لا يوجد driver مرتبط
```

#### 5. تدفق الـ udev MODALIAS

```
device يُضاف للـ bus
        │
        ▼
  auxiliary_uevent(dev, env)
    ├── name = dev_name(dev)
    │   مثال: "foo_mod.foo_dev.0"
    ├── p = strrchr(name, '.')
    │   p يشير لـ ".0"
    └── add_uevent_var(env,
            "MODALIAS=auxiliary:foo_mod.foo_dev")
                │
                ▼
          udev يقرأ MODALIAS ويحمّل الـ module المناسب
```

---

### استراتيجية الـ Locking

#### جدول الـ Locks

| الـ Lock | النوع | ما يحميه | من يمسكه |
|---|---|---|---|
| `auxdev->sysfs.lock` | `struct mutex` | إنشاء/حذف مجلد irqs في sysfs | `auxiliary_device_sysfs_irq_add` و`auxiliary_device_sysfs_irq_remove` |
| `dev->driver` reference | kernel device model (داخلي) | ربط الـ driver بالـ device | bus_type core تلقائيًا |

#### تحليل الـ Locking بالتفصيل

**الـ `sysfs.lock` (mutex):**
- يُهيَّأ في `auxiliary_device_init()` عبر `mutex_init()`
- يُدمَّر في `auxiliary_device_uninit()` عبر `mutex_destroy()`
- يحمي عمليات sysfs المتعلقة بالـ IRQs (الموجودة في ملفات أخرى غير هذا الملف)

**ملاحظة مهمة حول الـ Custom Ops:**
الملف نفسه يُحذِّر صراحةً في التوثيق:

```
/* تحذير من الكود: */
/* ops المخصصة تحتاج global locks لكل device */
/* لحمايتها من إزالة auxiliary_drv أثناء الاستدعاء */
/* البديل الأفضل: EXPORT_SYMBOL*() + module infrastructure */
```

أي أن الـ auxiliary bus **لا يوفر** lock مدمج لحماية الـ ops المخصصة — هذا مسؤولية الـ driver نفسه.

**الـ Bus-Level Locking (داخلي في kernel):**
- عمليات `device_add` / `device_del` محمية بـ locks داخلية في الـ device model
- `driver_register` / `driver_unregister` محميان بالـ `device_driver` locks
- الـ probe/remove لا يُستدعيان بشكل متزامن على نفس الـ device

#### ترتيب الـ Locks (Lock Ordering)

```
[صحيح - ترتيب آمن]

kernel device model lock  (خارجي - يديره bus core)
        │
        ▼
auxdev->sysfs.lock  (داخلي - للـ sysfs فقط)

[لا يوجد lock ordering معقّد] — الـ auxiliary bus بسيط في هذا الجانب.
```

---

### ملخص: الـ Accessor Functions المهمة

| Function | الوظيفة |
|---|---|
| `to_auxiliary_dev(dev)` | `container_of(dev, struct auxiliary_device, dev)` |
| `to_auxiliary_drv(drv)` | `container_of(drv, struct auxiliary_driver, driver)` |
| `auxiliary_get_drvdata(auxdev)` | `dev_get_drvdata(&auxdev->dev)` |
| `auxiliary_set_drvdata(auxdev, data)` | `dev_set_drvdata(&auxdev->dev, data)` |
| `auxiliary_device_uninit(auxdev)` | `mutex_destroy` + `put_device` |
| `auxiliary_device_delete(auxdev)` | `device_del(&auxdev->dev)` |
## Phase 4: شرح الـ Functions

### جدول ملخص الـ Functions

| Function | Type | Purpose |
|----------|------|---------|
| `auxiliary_match_id()` | static | تبحث عن الـ ID المناسب في جدول الـ driver لمطابقة الـ device |
| `auxiliary_match()` | static | تقرر هل الـ driver يتطابق مع الـ device |
| `auxiliary_uevent()` | static | تولّد حدث الـ uevent لإشعار الـ userspace عند إضافة device |
| `auxiliary_bus_probe()` | static | تستدعي الـ probe الخاصة بالـ driver عند ربطه بـ device |
| `auxiliary_bus_remove()` | static | تستدعي الـ remove الخاصة بالـ driver عند فصله عن الـ device |
| `auxiliary_bus_shutdown()` | static | تستدعي الـ shutdown الخاصة بالـ driver عند إغلاق النظام |
| `auxiliary_device_init()` | EXPORT_SYMBOL_GPL | تتحقق من الـ auxiliary_device وتهيئه (الخطوة الثانية من ثلاث) |
| `__auxiliary_device_add()` | EXPORT_SYMBOL_GPL | تضيف الـ auxiliary_device إلى الـ bus (الخطوة الثالثة من ثلاث) |
| `__auxiliary_driver_register()` | EXPORT_SYMBOL_GPL | تسجّل auxiliary_driver في الـ auxiliary bus |
| `auxiliary_driver_unregister()` | EXPORT_SYMBOL_GPL | تلغي تسجيل auxiliary_driver من الـ bus |
| `auxiliary_device_release()` | static | callback لتحرير الذاكرة عند حذف الـ device |
| `auxiliary_device_create()` | EXPORT_SYMBOL_GPL | helper شاملة لإنشاء auxiliary_device بخطوة واحدة |
| `auxiliary_device_destroy()` | EXPORT_SYMBOL_GPL | helper لحذف auxiliary_device تم إنشاؤه بـ `auxiliary_device_create()` |
| `__devm_auxiliary_device_create()` | EXPORT_SYMBOL_GPL | نسخة managed من `auxiliary_device_create()` تُحرّر تلقائياً |
| `auxiliary_bus_init()` | `__init` | تسجّل الـ auxiliary bus في الـ kernel عند الإقلاع |

---

### المجموعة الأولى: Matching — كيف يعرف الـ kernel مين يتكلم مع مين؟

فكرة الـ auxiliary bus كلها تعتمد على **الاسم**. كل `auxiliary_device` عنده اسم مُركّب من ثلاثة أجزاء: `modname.devname.id`، مثل `foo_mod.foo_dev.0`. الـ driver بيحدد في جدول الأسماء بتاعه (`id_table`) الأجهزة اللي يقدر يتعامل معها عبر جزء الاسم قبل آخر نقطة فقط (`foo_mod.foo_dev`). هاتين الـ functions هما اللي بتنفّذان هذا المنطق.

---

#### `auxiliary_match_id()`

```c
static const struct auxiliary_device_id *auxiliary_match_id(
    const struct auxiliary_device_id *id,
    const struct auxiliary_device *auxdev)
```

**ما الذي تفعله؟**
هذه الـ function بتأخذ جدول أسماء الـ driver (`id_table`) وكائن الـ device، وتبحث عن أول entry في الجدول اسمه يتطابق مع اسم الـ device. المقارنة بتكون على الجزء الأول من اسم الـ device فقط — أي كل شيء قبل آخر نقطة `.` في الاسم. مثلاً الاسم `foo_mod.foo_dev.3` بيُقارن بـ `foo_mod.foo_dev` فقط، الرقم `3` بيتجاهله وقت المطابقة.

**الـ Parameters:**
- **`id`**: مؤشر على أول عنصر في جدول `auxiliary_device_id` الخاص بالـ driver — الجدول بينتهي بعنصر فارغ (`name[0] == '\0'`).
- **`auxdev`**: مؤشر على الـ `auxiliary_device` المراد مطابقته.

**القيمة المُرجعة:**
- مؤشر على الـ `auxiliary_device_id` المطابق إذا وُجد.
- `NULL` إذا لم يوجد تطابق أو إذا كان اسم الـ device لا يحتوي على نقطة.

**التفاصيل المهمة:**
- بتستخدم `strrchr()` للعثور على آخر نقطة في الاسم — هذا هو الفاصل بين جزء الـ match name والـ ID العددي.
- المقارنة بـ `strncmp()` على طول محدد (`match_size`) لضمان المقارنة الدقيقة بدون زيادة أو نقصان.

**pseudocode مبسّط:**
```
auxdev_name = "foo_mod.foo_dev.3"
p = آخر نقطة → تشير إلى ".3"
match_size = طول "foo_mod.foo_dev" = 15

لكل id في الجدول:
    إذا strlen(id->name) == 15 && id->name == "foo_mod.foo_dev":
        return id  ← وجدنا تطابق!

return NULL  ← ما وجدنا
```

---

#### `auxiliary_match()`

```c
static int auxiliary_match(struct device *dev, const struct device_driver *drv)
```

**ما الذي تفعله؟**
هذه هي الـ callback الرسمية للـ bus اللي الـ kernel بيستدعيها كلما أراد أن يعرف: "هل هذا الـ driver يناسب هذا الـ device؟". هي مجرد wrapper فوق `auxiliary_match_id()` — بتحوّل الـ generic types (`device`, `device_driver`) إلى الـ auxiliary-specific types وتستدعي المطابقة الفعلية.

**الـ Parameters:**
- **`dev`**: مؤشر generic على الـ device.
- **`drv`**: مؤشر generic على الـ driver.

**القيمة المُرجعة:**
- `1` (غير صفر) إذا وُجد تطابق — أي يجب ربط هذا الـ driver بهذا الـ device.
- `0` إذا لا تطابق.

**من يستدعيها؟**
الـ kernel نفسه عبر `bus_type.match` callback عند تسجيل device أو driver جديد على الـ bus.

---

### المجموعة الثانية: Uevent — إشعار الـ userspace

#### `auxiliary_uevent()`

```c
static int auxiliary_uevent(const struct device *dev, struct kobj_uevent_env *env)
```

**ما الذي تفعله؟**
لما يُضاف `auxiliary_device` جديد إلى النظام، الـ kernel بيرسل **uevent** إلى الـ userspace (مثل `udev` أو `systemd-udevd`) ليعرف بوجوده. هذه الـ function بتضيف متغير `MODALIAS` إلى بيئة الـ uevent. الـ `MODALIAS` هو المفتاح اللي بيستخدمه `modprobe` لتحميل الـ module المناسب تلقائياً.

صيغة الـ MODALIAS المُولَّدة: `auxiliary:foo_mod.foo_dev` — أي prefix ثابت (`auxiliary:`) ثم اسم الـ device بدون جزء الـ ID العددي الأخير.

**الـ Parameters:**
- **`dev`**: الـ device الذي يُولِّد الحدث.
- **`env`**: بيئة الـ uevent — struct بتحتوي مجموعة متغيرات KEY=VALUE يتم إرسالها إلى الـ userspace.

**القيمة المُرجعة:**
- `0` عند النجاح.
- قيمة سالبة عند الفشل (مثلاً إذا امتلأت بيئة الـ uevent).

**مثال عملي:**
```
device name: "foo_mod.foo_dev.3"
MODALIAS   = "auxiliary:foo_mod.foo_dev"
```
**الـ `AUXILIARY_MODULE_PREFIX`** هو الثابت `"auxiliary:"` المُعرَّف في الـ header.

---

### المجموعة الثالثة: Bus Callbacks — دورة حياة الـ Driver

هذه الـ functions هي الـ callbacks التي يسجّلها الـ auxiliary bus في `struct bus_type`. الـ kernel يستدعيها في الأوقات المناسبة — ربط، فصل، إيقاف.

---

#### `auxiliary_bus_probe()`

```c
static int auxiliary_bus_probe(struct device *dev)
```

**ما الذي تفعله؟**
عندما يتطابق driver مع device — أي بعد نجاح `auxiliary_match()` — الـ kernel يستدعي هذه الـ function. هي أولاً تُلحق الـ device بـ **PM domain** (نطاق إدارة الطاقة) لضمان توفّر الطاقة، ثم تستدعي الـ `probe()` الفعلية الخاصة بالـ driver.

**الـ Parameters:**
- **`dev`**: الـ device المراد ربطه بالـ driver.

**القيمة المُرجعة:**
- نتيجة `auxdrv->probe()` عند النجاح.
- قيمة خطأ سالبة إذا فشل إلحاق الـ PM domain.

**التفاصيل المهمة:**
- تستخدم `dev_pm_domain_attach()` مع flags `PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF` — يعني: شغّل الطاقة عند الإلحاق، وأوقفها عند الفصل تلقائياً.
- تمرّر إلى الـ `probe()` الـ `auxiliary_device_id` المطابق ليعرف الـ driver أيّ entry في الجدول سبّب استدعاءه.

**pseudocode:**
```
1. احصل على auxdrv و auxdev من الـ dev
2. الحق الـ device بـ PM domain (شغّل الطاقة)
3. إذا فشل → return error
4. استدعِ auxdrv->probe(auxdev, matching_id)
5. return نتيجة الـ probe
```

---

#### `auxiliary_bus_remove()`

```c
static void auxiliary_bus_remove(struct device *dev)
```

**ما الذي تفعله؟**
عندما يُزال الـ device من الـ bus أو يُلغى تسجيل الـ driver، الـ kernel يستدعي هذه الـ function لإعطاء الـ driver فرصة التنظيف. هي فقط تتحقق أن الـ driver لديه `remove()` callback وتستدعيه.

**الـ Parameters:**
- **`dev`**: الـ device المراد فصله.

**القيمة المُرجعة:** لا شيء (`void`).

**ملاحظة:** الـ `remove` callback في `auxiliary_driver` اختياري — الكود يتحقق من وجوده قبل الاستدعاء (`if (auxdrv->remove)`). بعد استدعاء `remove()`، الـ PM domain يُفصل تلقائياً بفضل flag الـ `PD_FLAG_DETACH_POWER_OFF` الذي مُرِّر في `probe`.

---

#### `auxiliary_bus_shutdown()`

```c
static void auxiliary_bus_shutdown(struct device *dev)
```

**ما الذي تفعله؟**
عند إغلاق النظام (`shutdown`) أو إعادة التشغيل، الـ kernel يستدعي هذه الـ function على كل device. تتحقق أن الـ device لا يزال مرتبطاً بـ driver وأن لدى الـ driver `shutdown()` callback، ثم تستدعيه.

**الـ Parameters:**
- **`dev`**: الـ device المراد إيقافه.

**القيمة المُرجعة:** لا شيء (`void`).

**التفاصيل المهمة:**
- على عكس `auxiliary_bus_remove()`، هنا يبدأ `auxdrv` بـ `NULL` ويُعيَّن فقط إذا `dev->driver != NULL`. هذا احتياط إضافي لأن الـ shutdown يمكن أن يحدث في أي وقت — حتى قبل أن يرتبط الـ device بـ driver.

```c
/* الكود يتحقق بحذر */
if (dev->driver) {
    auxdrv = to_auxiliary_drv(dev->driver);
    auxdev = to_auxiliary_dev(dev);
}
if (auxdrv && auxdrv->shutdown)
    auxdrv->shutdown(auxdev);
```

---

### المجموعة الرابعة: تسجيل الـ Device — الخطوات الثلاث

تسجيل `auxiliary_device` يمر بثلاث خطوات متسلسلة. تخيّل الأمر كأنك تبني منزلاً: أولاً تضع الأساس، ثم تبني الجدران، ثم تُعلّق اللافتة.

```
[الخطوة 1: ملء البيانات يدوياً]
      ↓
[الخطوة 2: auxiliary_device_init()] ← يتحقق ويهيّئ device_initialize()
      ↓
[الخطوة 3: __auxiliary_device_add()] ← يضبط الاسم ويضيف إلى الـ bus
```

---

#### `auxiliary_device_init()`

```c
int auxiliary_device_init(struct auxiliary_device *auxdev)
```

**ما الذي تفعله؟**
هذه هي **الخطوة الثانية** في تسجيل الـ `auxiliary_device`. تتحقق من أن الحقول الأساسية موجودة (parent و name)، تربط الـ device بالـ auxiliary bus، وتستدعي `device_initialize()` لتهيئة الـ device object داخلياً. كذلك تهيّئ الـ mutex الخاص بـ sysfs.

**الـ Parameters:**
- **`auxdev`**: مؤشر على الـ `auxiliary_device` المراد تهيئته — يجب أن يكون `dev.parent` و `name` مملوئين مسبقاً.

**القيمة المُرجعة:**
- `0` عند النجاح.
- `-EINVAL` إذا كان `dev->parent` أو `auxdev->name` يساوي `NULL`.

**التفاصيل المهمة:**
- إذا نجحت، **أي خطأ لاحق** يستلزم استدعاء `auxiliary_device_uninit()` للتنظيف — لا تستدعِ `kfree()` مباشرة بعد هذه النقطة.
- إذا فشلت، المستدعي هو المسؤول عن تحرير الذاكرة مباشرة لأن `device_initialize()` لم تُنفَّذ.
- **الـ locking:** تهيّئ `mutex_init(&auxdev->sysfs.lock)` الذي يُستخدم لاحقاً لحماية عمليات الـ sysfs المتعلقة بالـ IRQs.

```c
int auxiliary_device_init(struct auxiliary_device *auxdev)
{
    struct device *dev = &auxdev->dev;

    if (!dev->parent) { /* تحقق من الأب */
        pr_err("...");
        return -EINVAL;
    }
    if (!auxdev->name) { /* تحقق من الاسم */
        pr_err("...");
        return -EINVAL;
    }

    dev->bus = &auxiliary_bus_type; /* اربطه بالـ bus */
    device_initialize(&auxdev->dev); /* هيّئ الـ device object */
    mutex_init(&auxdev->sysfs.lock); /* هيّئ الـ mutex */
    return 0;
}
```

---

#### `__auxiliary_device_add()`

```c
int __auxiliary_device_add(struct auxiliary_device *auxdev, const char *modname)
```

**ما الذي تفعله؟**
هذه هي **الخطوة الثالثة والأخيرة**. تُشكّل الاسم الكامل للـ device بالصيغة `modname.devname.id` (مثل `foo_mod.foo_dev.0`) وتستدعي `device_add()` لإضافته رسمياً إلى الـ bus ونظام الملفات (`sysfs`). بعد هذه الخطوة الـ device مرئي للـ kernel وجاهز للربط بـ driver.

**الـ Parameters:**
- **`auxdev`**: الـ device المراد إضافته — يجب أن يكون قد مرّ بـ `auxiliary_device_init()` أولاً.
- **`modname`**: اسم الـ module الأب — عادةً `KBUILD_MODNAME` يتم حقنه تلقائياً عبر الـ macro `auxiliary_device_add()`.

**القيمة المُرجعة:**
- `0` عند النجاح.
- `-EINVAL` إذا كان `modname` هو `NULL`.
- قيمة خطأ سالبة من `dev_set_name()` أو `device_add()`.

**التفاصيل المهمة:**
- عند الفشل بعد `auxiliary_device_init()`، المستدعي يجب أن يستدعي `auxiliary_device_uninit()` — **ليس** `kfree()` مباشرة — لأن الـ reference count يُدار الآن من قِبل نظام الـ device.
- الـ macro المستخدم عادةً هو `auxiliary_device_add(auxdev)` الذي يحقن `KBUILD_MODNAME` تلقائياً.

```
الاسم الناتج = modname + "." + auxdev->name + "." + auxdev->id
مثال        = "sound_core" + "." + "hdmi" + "." + "2"
            = "sound_core.hdmi.2"
```

---

### المجموعة الخامسة: تسجيل الـ Driver

---

#### `__auxiliary_driver_register()`

```c
int __auxiliary_driver_register(struct auxiliary_driver *auxdrv,
                                struct module *owner, const char *modname)
```

**ما الذي تفعله؟**
تُسجّل `auxiliary_driver` على الـ auxiliary bus. تبني اسم الـ driver، تربطه بالـ bus، وتستدعي `driver_register()`. بعدها الـ kernel يبدأ مباشرة في البحث عن الـ devices المطابقة ويستدعي `probe()` لكل واحد.

**الـ Parameters:**
- **`auxdrv`**: الـ driver المراد تسجيله — يجب أن يحتوي على `probe` و `id_table` (إلزاميان).
- **`owner`**: الـ module المالك — عادةً `THIS_MODULE` يُحقن تلقائياً.
- **`modname`**: اسم الـ module — عادةً `KBUILD_MODNAME` يُحقن تلقائياً.

**القيمة المُرجعة:**
- `0` عند النجاح.
- `-EINVAL` إذا كان `probe` أو `id_table` غير موجودَين.
- `-ENOMEM` إذا فشل تخصيص ذاكرة للاسم.
- قيمة خطأ من `driver_register()`.

**التفاصيل المهمة:**
- إذا كان الـ driver لديه `name` محدد: الاسم النهائي = `"modname.drivername"`.
- إذا لم يكن لديه `name`: الاسم النهائي = `"modname"` فقط.
- إذا فشل `driver_register()`، يتم تحرير الذاكرة المخصصة للاسم مباشرة (`kfree`).
- الـ macro المستخدم عادةً: `auxiliary_driver_register(auxdrv)` الذي يحقن `THIS_MODULE` و `KBUILD_MODNAME`.

```
auxdrv->name موجود   → driver.name = "mymod.mydrv"
auxdrv->name غير موجود → driver.name = "mymod"
```

---

#### `auxiliary_driver_unregister()`

```c
void auxiliary_driver_unregister(struct auxiliary_driver *auxdrv)
```

**ما الذي تفعله؟**
تلغي تسجيل الـ `auxiliary_driver` من الـ bus. الـ kernel يستدعي `remove()` على كل الـ devices المرتبطة بهذا الـ driver أولاً، ثم يُزيل الـ driver من قوائمه الداخلية. بعدها تُحرَّر الذاكرة المخصصة للاسم.

**الـ Parameters:**
- **`auxdrv`**: الـ driver المراد إلغاء تسجيله.

**القيمة المُرجعة:** لا شيء (`void`).

**التفاصيل المهمة:**
- `driver_unregister()` هي عملية blocking — تنتظر حتى تنتهي كل الـ `remove()` callbacks قبل العودة.
- الـ `kfree(auxdrv->driver.name)` يُحرّر الاسم الذي خُصِّص بـ `kasprintf()` داخل `__auxiliary_driver_register()`.

---

### المجموعة السادسة: Helpers — إنشاء وحذف الـ Device بطريقة مبسّطة

هذه الـ functions توفر طريقة أسهل لمن لا يريد إدارة الخطوات الثلاث يدوياً.

---

#### `auxiliary_device_release()` (static)

```c
static void auxiliary_device_release(struct device *dev)
```

**ما الذي تفعله؟**
هذه هي الـ `release` callback المسجّلة في `auxiliary_device_create()`. يستدعيها الـ kernel تلقائياً عندما يصل الـ reference count للـ device إلى صفر — أي عندما لا يشير إليه أحد. تُحرّر الـ device tree node (إن وجدت) وتُحرّر ذاكرة الـ `auxiliary_device` نفسها.

**الـ Parameters:**
- **`dev`**: الـ generic device المراد تحريره.

**القيمة المُرجعة:** لا شيء (`void`).

**من يستدعيها؟**
الـ kernel تلقائياً عبر `put_device()` عندما يصل الـ reference count إلى صفر.

---

#### `auxiliary_device_create()`

```c
struct auxiliary_device *auxiliary_device_create(struct device *dev,
                                                  const char *modname,
                                                  const char *devname,
                                                  void *platform_data,
                                                  int id)
```

**ما الذي تفعله؟**
هذه الـ function هي **الطريقة السهلة** لإنشاء `auxiliary_device`. بدلاً من ثلاث خطوات منفصلة، تقوم بكل شيء داخلياً: تخصص الذاكرة، تملأ الحقول، تستدعي `auxiliary_device_init()` و `__auxiliary_device_add()`. الـ release callback المُستخدمة هي `auxiliary_device_release()` المُعرَّفة في نفس الملف.

**الـ Parameters:**
- **`dev`**: الـ parent device — يجب أن يكون غير NULL.
- **`modname`**: اسم الـ module الأب — يدخل في تسمية الـ device.
- **`devname`**: اسم الـ sub-device — الجزء الثاني من الاسم المُركّب.
- **`platform_data`**: بيانات اختيارية تُخزَّن في `dev.platform_data` — يمكن أن يكون `NULL`.
- **`id`**: معرّف عددي فريد للـ device — الجزء الثالث من الاسم.

**القيمة المُرجعة:**
- مؤشر على الـ `auxiliary_device` الجديد عند النجاح.
- `NULL` عند الفشل (فشل في تخصيص الذاكرة أو في التهيئة أو في الإضافة).

**التفاصيل المهمة:**
- تستخدم `kzalloc()` — أي الذاكرة تُصفَّر تلقائياً.
- تستدعي `device_set_of_node_from_dev()` لوراثة الـ device tree node من الـ parent.
- عند فشل `__auxiliary_device_add()`، تستدعي `auxiliary_device_uninit()` وليس `kfree()` مباشرة — لأن `auxiliary_device_init()` نجحت وأخذت reference على الـ device.

**pseudocode:**
```
1. kzalloc(auxdev)              ← خصّص ذاكرة
2. املأ: id, name, parent, platform_data, release
3. خذ الـ of_node من الـ parent
4. auxiliary_device_init(auxdev) ← إذا فشل → حرّر وأرجع NULL
5. __auxiliary_device_add(auxdev, modname) ← إذا فشل → uninit وأرجع NULL
6. return auxdev                ← النجاح
```

---

#### `auxiliary_device_destroy()`

```c
void auxiliary_device_destroy(void *auxdev)
```

**ما الذي تفعله؟**
تحذف `auxiliary_device` تم إنشاؤه بـ `auxiliary_device_create()`. العملية من خطوتين: أولاً `auxiliary_device_delete()` لإزالته من الـ bus (مما يُطلق `remove()` على الـ driver المرتبط)، ثم `auxiliary_device_uninit()` للإفراج عن الـ reference.

**الـ Parameters:**
- **`auxdev`**: مؤشر من نوع `void *` على الـ `auxiliary_device` — النوع `void *` مقصود لأن هذه الـ function تُستخدم كـ callback في `devm_add_action_or_reset()`.

**القيمة المُرجعة:** لا شيء (`void`).

**ملاحظة مهمة:** التسلسل مهم جداً:
```
1. auxiliary_device_delete()  ← يُخبر الـ bus أن الـ device راح (يستدعي remove)
2. auxiliary_device_uninit()  ← يُقلّل الـ ref count (قد يُطلق release لتحرير الذاكرة)
```
عكس الترتيب يمكن أن يسبب use-after-free.

---

#### `__devm_auxiliary_device_create()`

```c
struct auxiliary_device *__devm_auxiliary_device_create(struct device *dev,
                                                         const char *modname,
                                                         const char *devname,
                                                         void *platform_data,
                                                         int id)
```

**ما الذي تفعله؟**
النسخة الـ **device-managed** من `auxiliary_device_create()`. الفكرة بسيطة: بتنشئ الـ device بنفس الطريقة، لكن بتسجّل `auxiliary_device_destroy()` كـ cleanup action مرتبط بعمر الـ parent device. يعني لما الـ parent يُزال، الـ auxiliary device بيتحذف تلقائياً — بدون أي كود تنظيف يدوي.

**الـ Parameters:** نفس `auxiliary_device_create()` تماماً.

**القيمة المُرجعة:**
- مؤشر على الـ `auxiliary_device` الجديد عند النجاح.
- `NULL` عند الفشل.

**التفاصيل المهمة:**
- تستخدم `devm_add_action_or_reset()` — إذا فشل تسجيل الـ cleanup action، تُحذف الـ device فوراً (`_or_reset` جزء من الاسم).
- الـ macro المُوصى به: `devm_auxiliary_device_create(dev, devname, platform_data)` الذي يحقن `KBUILD_MODNAME` تلقائياً ويضع `id = 0`.

**pseudocode:**
```
1. auxiliary_device_create(dev, modname, devname, platform_data, id)
   ← إذا فشل → return NULL

2. devm_add_action_or_reset(dev, auxiliary_device_destroy, auxdev)
   ← يسجّل cleanup تلقائي مرتبط بعمر dev
   ← إذا فشل → يستدعي auxiliary_device_destroy تلقائياً ويعيد NULL

3. return auxdev
```

---

### المجموعة السابعة: Initialization — إقلاع الـ Bus

#### `auxiliary_bus_init()`

```c
void __init auxiliary_bus_init(void)
```

**ما الذي تفعله؟**
هذه الـ function تُسجّل الـ auxiliary bus في الـ kernel عند الإقلاع. تُستدعى مرة واحدة فقط في حياة الـ kernel. `bus_register()` تُضيف الـ `auxiliary_bus_type` إلى قائمة الـ buses في الـ kernel وتنشئ مجلده في `/sys/bus/auxiliary/`.

**الـ Parameters:** لا يوجد.

**القيمة المُرجعة:** لا شيء (`void`) — لكن أي فشل يُطبع كـ warning عبر `WARN_ON()`.

**التفاصيل المهمة:**
- الـ `__init` annotation يعني أن الـ kernel يُحرّر ذاكرة هذه الـ function بعد الانتهاء من الإقلاع — توفيراً للذاكرة.
- لا يوجد مقابل `__exit` لأن الـ auxiliary bus لا يمكن تسجيل إلغاء تسجيله — هو جزء أساسي من الـ kernel.
- **من يستدعيها؟** يتم استدعاؤها من كود تهيئة الـ bus subsystem في الـ kernel (عادةً من `buses_init()` أو ما شابهه في `drivers/base/`).
## Phase 5: دليل الـ Debugging الشامل

الـ **Auxiliary Bus** هو bus افتراضي بالكامل — مفيش hardware حقيقي وراه، بس ده مش معناه إنك مش هتواجه مشاكل. المشاكل الشائعة فيه هي: driver مش بيعمل bind، device مش بيتسجل، أو mismatch في الاسم. الدليل ده هيوريك إزاي تشوف إيه اللي بيحصل بالضبط.

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ Auxiliary Bus

الـ auxiliary bus نفسه ما عندوش debugfs entries خاصة بيه، لكن الـ **device model** العام بيوفرها تحت `/sys/kernel/debug/`:

```bash
# شوف كل الـ devices الـ registered على الـ auxiliary bus
ls /sys/kernel/debug/devices/ 2>/dev/null | grep auxiliary

# لو عندك kernel مع CONFIG_PM_DEBUG
cat /sys/kernel/debug/pm_genpd/*/summary
```

لو الـ parent driver عنده devlink (زي drivers الـ NIC)، استخدم:

```bash
# شوف الـ devlink resources والـ params
devlink dev show
devlink dev info pci/0000:01:00.0
```

---

#### 2. مدخلات الـ sysfs المهمة

الـ auxiliary bus بيسجل devices على الـ sysfs تحت `/sys/bus/auxiliary/`. ده أهم مكان للـ debugging.

```bash
# شوف كل الـ auxiliary devices المسجلة
ls /sys/bus/auxiliary/devices/

# مثال للـ output:
# foo_mod.foo_dev.0   bar_mod.bar_dev.1   mlx5_core.eth.0

# شوف تفاصيل device معين
ls /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/

# شوف الـ driver اللي بيشتغل مع الـ device
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/driver_override
readlink /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/driver

# شوف الـ MODALIAS (ده اللي udev بيستخدمه)
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/modalias
# output: auxiliary:foo_mod.foo_dev

# شوف الـ parent device
readlink /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/../..

# شوف كل الـ drivers المسجلة على الـ auxiliary bus
ls /sys/bus/auxiliary/drivers/

# شوف الـ devices المربوطة بـ driver معين
ls /sys/bus/auxiliary/drivers/my_auxiliary_driver/

# شوف الـ IRQs المسجلة في الـ sysfs (لو auxiliary_device_sysfs_irq_add اتستخدم)
ls /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/irqs/
```

| الـ sysfs Entry | المعنى |
|---|---|
| `/sys/bus/auxiliary/devices/` | كل الـ auxiliary devices الموجودة |
| `/sys/bus/auxiliary/drivers/` | كل الـ auxiliary drivers المسجلة |
| `device/driver` | symlink للـ driver اللي بيشتغل مع الـ device |
| `device/modalias` | الـ string اللي بيستخدمه udev للـ autoload |
| `device/uevent` | معلومات الـ uevent بما فيها MODALIAS |
| `device/irqs/` | الـ IRQs المسجلة (لو موجودة) |

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# شوف الـ events المتاحة للـ device model
ls /sys/kernel/tracing/events/device/
ls /sys/kernel/tracing/events/devres/

# فعّل tracing للـ bus match (مهم جداً لمعرفة إيه اللي بيحصل وقت الـ bind)
echo 1 > /sys/kernel/tracing/events/device/device_pm_callback/enable

# trace كل الـ bus operations
echo 'auxiliary_bus_probe' > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# شوف النتيجة
cat /sys/kernel/tracing/trace

# طفّيه بعد الخلاص
echo 0 > /sys/kernel/tracing/tracing_on
echo nop > /sys/kernel/tracing/current_tracer
```

لتتبع دالة `auxiliary_match` تحديداً (ده بيشوف إيه الـ match اللي بيحصل):

```bash
echo 'auxiliary_match auxiliary_match_id auxiliary_bus_probe' \
  > /sys/kernel/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# جرب الـ bind
cat /sys/kernel/tracing/trace
```

---

#### 4. الـ printk والـ Dynamic Debug

الـ auxiliary.c بيستخدم `pr_fmt` اللي بيضيف `KBUILD_MODNAME:__func__` قبل كل رسالة. يعني الرسائل هتبان بالشكل ده:

```
auxiliary:auxiliary_device_init: auxiliary_device has a NULL dev->parent
auxiliary:__auxiliary_device_add: auxiliary device dev_set_name failed: -12
```

```bash
# فعّل dynamic debug للـ auxiliary bus كله
echo 'module auxiliary +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّله لملف معين فقط
echo 'file drivers/base/auxiliary.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع filename وline number في الرسالة (مفيد للـ debugging العميق)
echo 'file drivers/base/auxiliary.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الإعدادات الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep auxiliary

# لو حابب تفعله من الـ boot، ضيف للـ kernel command line:
# dyndbg="file drivers/base/auxiliary.c +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config Option | الوصف | متى تستخدمه |
|---|---|---|
| `CONFIG_DEBUG_DRIVER` | يطبع معلومات إضافية عن الـ driver core | الخطوة الأولى في أي مشكلة bind |
| `CONFIG_DEBUG_DEVRES` | يتتبع الـ managed resources (devm_*) | لو `devm_auxiliary_device_create` بيفشل |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug subsystem | مطلوب للـ dynamic debug فوق |
| `CONFIG_PM_DEBUG` | يضيف debugging للـ power management | لو الـ suspend/resume بيفشل |
| `CONFIG_PM_ADVANCED_DEBUG` | sysfs entries إضافية للـ PM | لفهم حالة الـ power domain |
| `CONFIG_DEBUG_KOBJECT` | يتتبع عمليات الـ kobject | لمشاكل الـ reference counting |
| `CONFIG_BUS_NOTIFIER_ERROR_INJECT` | يحقن errors في الـ bus notifiers | لاختبار error paths |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks (lockdep) | لو الـ sysfs.lock بيسبب مشكلة |
| `CONFIG_KASAN` | يكتشف memory corruption | لو فيه use-after-free في الـ release callback |

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# udevadm — شوف events الـ auxiliary devices
udevadm monitor --kernel --subsystem-match=auxiliary

# شوف الـ MODALIAS وإيه الـ module اللي المفروض يتحمل
udevadm info /sys/bus/auxiliary/devices/foo_mod.foo_dev.0

# شوف إيه الـ module المطلوب (modprobe alias)
modprobe --show-depends auxiliary:foo_mod.foo_dev

# شوف الـ module dependencies
lsmod | grep foo_mod

# شوف الـ exported symbols من الـ parent module
cat /proc/kallsyms | grep foo_mod

# شوف الـ bus كله
cat /sys/bus/auxiliary/uevent
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `auxiliary_device has a NULL dev->parent` | الـ `auxdev->dev.parent` مش متعبى | اضبط `auxdev->dev.parent = parent_dev` قبل `auxiliary_device_init()` |
| `auxiliary_device has a NULL name` | الـ `auxdev->name` = NULL | اضبط `auxdev->name = "my_dev_name"` قبل الـ init |
| `auxiliary device modname is NULL` | الـ modname في `__auxiliary_device_add` = NULL | استخدم الـ macro `auxiliary_device_add()` بدل الدالة المباشرة |
| `auxiliary device dev_set_name failed: -12` | فضل ذاكرة (ENOMEM) | مشكلة في الـ memory، شوف `dmesg` للتفاصيل |
| `adding auxiliary device failed!: -17` | الاسم موجود بالفعل (EEXIST) | غيّر الـ `id` عشان يكون unique — الاسم هو `modname.devname.id` |
| `Failed to attach to PM Domain : -N` | فشل الـ attach للـ power domain | شوف `CONFIG_PM_DOMAIN`، أو الـ PM domain مش موجود |
| `WARN_ON(!auxdrv->probe)` | الـ driver بيتسجل بدون `probe` callback | كل auxiliary driver لازم يكون عنده `probe` function |
| `WARN_ON(!auxdrv->id_table)` | الـ driver بيتسجل بدون `id_table` | أضف `id_table` بالـ device names اللي بيدعمها الـ driver |
| `WARN_ON(bus_register(...))` | فشل تسجيل الـ auxiliary bus نفسه (panic وقت البوت) | مشكلة خطيرة في الـ kernel init، شوف `initcall_debug` |

---

#### 8. أماكن استراتيجية لـ dump_stack() وWARN_ON()

لو بتطور driver وحابب تضيف debugging مؤقت:

```c
/* في auxiliary_device_init — تحقق إن الـ parent صح */
int auxiliary_device_init(struct auxiliary_device *auxdev)
{
    struct device *dev = &auxdev->dev;

    if (!dev->parent) {
        /* هنا WARN_ON موجودة بالفعل بشكل pr_err */
        pr_err("auxiliary_device has a NULL dev->parent\n");
        dump_stack(); /* أضف ده مؤقتاً لتعرف مين استدى الدالة */
        return -EINVAL;
    }
    /* ... */
}

/* في auxiliary_match — لو مش عارف ليه الـ bind مش بيحصل */
static int auxiliary_match(struct device *dev, const struct device_driver *drv)
{
    struct auxiliary_device *auxdev = to_auxiliary_dev(dev);
    const struct auxiliary_driver *auxdrv = to_auxiliary_drv(drv);
    int result;

    result = !!auxiliary_match_id(auxdrv->id_table, auxdev);
    /* أضف مؤقتاً: */
    pr_debug("matching '%s' against driver '%s': %s\n",
             dev_name(dev), drv->name, result ? "YES" : "NO");
    return result;
}

/* في auxiliary_bus_probe — لو الـ probe بيرجع error */
static int auxiliary_bus_probe(struct device *dev)
{
    /* ... */
    ret = auxdrv->probe(auxdev, ...);
    if (ret) {
        WARN(ret, "probe failed for %s: %d\n", dev_name(dev), ret);
        /* WARN_ON تطبع stack trace تلقائياً */
    }
    return ret;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتطابق حالة الـ Kernel

الـ auxiliary bus مفيش ورايه hardware حقيقي — هو بس abstraction layer. بس الـ parent device (PCI مثلاً) ممكن يكون عنده hardware state.

```bash
# تحقق إن الـ parent PCI device شغال
lspci -v -s 0000:01:00.0

# تحقق إن الـ parent device مش في error state
cat /sys/bus/pci/devices/0000:01:00.0/broken_parity_status
cat /sys/bus/pci/devices/0000:01:00.0/current_link_speed

# تحقق إن الـ auxiliary device موجود فعلاً في الـ kernel
ls /sys/bus/auxiliary/devices/ | grep "parent_modname"

# قارن الـ devices المسجلة في الـ kernel مع المتوقعة
cat /sys/bus/auxiliary/devices/*/uevent | grep MODALIAS
```

---

#### 2. تقنيات الـ Register Dump

الـ auxiliary bus نفسه ما عندوش registers، لكن الـ parent device (اللي بيسجل الـ auxiliary devices) غالباً PCI أو ACPI وعنده registers.

```bash
# اقرأ registers الـ parent PCI device باستخدام devmem2
# (محتاج تعرف الـ base address أول)
BAR0=$(cat /sys/bus/pci/devices/0000:01:00.0/resource | head -1 | awk '{print $1}')
devmem2 $BAR0 w   # اقرأ 32-bit من الـ BAR0

# استخدم /dev/mem مباشرة (محتاج CONFIG_DEVMEM=y و iomem=relaxed في الـ cmdline)
dd if=/dev/mem bs=4 count=1 skip=$((BAR0/4)) 2>/dev/null | xxd

# استخدم setpci لـ PCI config space
setpci -s 0000:01:00.0 VENDOR_ID.w
setpci -s 0000:01:00.0 STATUS.w

# شوف كل الـ config space
lspci -xxx -s 0000:01:00.0
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

الـ auxiliary bus virtual وما بيتكلمش على physical bus، بس لو الـ parent device بيكلم hardware:

```
لو الـ parent هو I2C/SPI device:
- راقب الـ SCL/SDA أو SCLK/MOSI/MISO lines
- ابحث عن الـ transactions اللي بتحصل وقت الـ probe

لو الـ parent هو PCIe:
- راقب الـ PCIe lane بـ protocol analyzer
- ابحث عن الـ TLP transactions وقت الـ device init

نقاط الـ Trigger المفيدة:
- Trigger على أول transaction بعد الـ driver bind
- Trigger على الـ power state transitions (D0 -> D3)
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | النمط في الـ dmesg | السبب المحتمل |
|---|---|---|
| الـ parent device مش شغال | `Failed to attach to PM Domain` | الـ power domain مش موجود أو مش initialized |
| الـ auxiliary device بيتعمل delete قبل ما يتعمل bind | `device_del called before driver bound` | race condition في الـ parent driver |
| الـ release callback بيتنادى قبل الأوان | use-after-free في الـ KASAN log | الـ parent بيحرر الـ auxdev قبل `auxiliary_device_uninit()` |
| اسم متكرر | `sysfs: cannot create duplicate filename` | الـ `id` مش unique لنفس `modname.devname` |
| الـ module مش بيتحمل تلقائياً | ما فيش رسائل خطأ لكن الـ driver مش موجود | الـ MODALIAS مش بيطابق أي `MODULE_DEVICE_TABLE` |

---

#### 5. الـ Device Tree Debugging

الـ auxiliary bus ما بيعتمدش على الـ DT مباشرة (زي ما الكود بيقول: "cannot live on the platform bus as they are not physical devices controlled by DT/ACPI"). لكن الـ parent device ممكن يكون DT-based، وفي `auxiliary_device_create` في الكود ده:

```c
device_set_of_node_from_dev(&auxdev->dev, dev); /* يورث الـ OF node من الـ parent */
```

```bash
# تحقق إن الـ auxiliary device ورث الـ OF node صح
cat /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/of_node/compatible 2>/dev/null

# شوف الـ full DT path للـ parent
readlink /sys/bus/auxiliary/devices/foo_mod.foo_dev.0/of_node

# قارن مع الـ DT المُفسَّر
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -A10 "your-parent-node"

# تحقق من وجود الـ parent في الـ DT
ls /sys/firmware/devicetree/base/
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "your_compat" {} \;
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. الفحص الأولي — هل الـ bus شغال؟**

```bash
# أمر شامل لحالة الـ auxiliary bus
echo "=== Auxiliary Bus Status ===" && \
ls /sys/bus/auxiliary/devices/ 2>/dev/null && \
echo "=== Registered Drivers ===" && \
ls /sys/bus/auxiliary/drivers/ 2>/dev/null && \
echo "=== Recent Kernel Messages ===" && \
dmesg | grep -i auxiliary | tail -20
```

**مثال للـ output المتوقع:**
```
=== Auxiliary Bus Status ===
mlx5_core.eth.0   mlx5_core.rdma.0   intel_sof.hdmi.0
=== Registered Drivers ===
mlx5_ib   mlx5_en   sof-audio
=== Recent Kernel Messages ===
[  2.345] auxiliary: auxiliary_bus_init: bus registered successfully
[  3.121] mlx5_core auxiliary mlx5_core.eth.0: probe called
```

---

**2. تشخيص مشكلة الـ bind**

```bash
# الخطوة 1: تحقق إن الـ device موجود
ls /sys/bus/auxiliary/devices/ | grep "device_name"

# الخطوة 2: شوف الـ MODALIAS
cat /sys/bus/auxiliary/devices/DEVICE_NAME/modalias
# المتوقع: auxiliary:modname.devname

# الخطوة 3: تحقق إن الـ driver عنده الـ id_table الصح
modinfo DRIVER_NAME | grep alias
# المتوقع: alias: auxiliary:modname.devname

# الخطوة 4: جرب الـ bind يدوياً
echo "DEVICE_NAME" > /sys/bus/auxiliary/drivers/DRIVER_NAME/bind
dmesg | tail -5

# الخطوة 5: لو الـ bind فشل، فعّل debug
echo 'file drivers/base/auxiliary.c +p' > /sys/kernel/debug/dynamic_debug/control
echo "DEVICE_NAME" > /sys/bus/auxiliary/drivers/DRIVER_NAME/bind
dmesg | tail -10
```

---

**3. مراقبة الـ events في real-time**

```bash
# في terminal 1 — راقب الـ uevents
udevadm monitor --kernel --subsystem-match=auxiliary

# في terminal 2 — راقب الـ dmesg
dmesg -w | grep -E "auxiliary|auxdev|auxdrv"

# في terminal 3 — افعل العملية اللي عايز تتبعها
modprobe parent_module
# أو
echo "DEVICE_NAME" > /sys/bus/auxiliary/drivers/DRIVER_NAME/bind
```

**مثال للـ output من udevadm:**
```
KERNEL[1234.567] add      /devices/virtual/auxiliary/foo_mod.foo_dev.0 (auxiliary)
ACTION=add
DEVPATH=/devices/virtual/auxiliary/foo_mod.foo_dev.0
SUBSYSTEM=auxiliary
MODALIAS=auxiliary:foo_mod.foo_dev
```

---

**4. تتبع الـ reference counting (مهم لمشاكل الـ use-after-free)**

```bash
# فعّل DEBUG_KOBJECT_RELEASE (من الـ kernel config أو module param)
echo 1 > /sys/module/kernel/parameters/kobject_debug 2>/dev/null || true

# شوف الـ reference count الحالي للـ device
cat /sys/bus/auxiliary/devices/DEVICE_NAME/power/runtime_usage 2>/dev/null

# تتبع كل put_device/get_device
echo 'put_device get_device device_add device_del' > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# افعل العملية
cat /sys/kernel/tracing/trace | grep -A2 -B2 "auxiliary"
echo 0 > /sys/kernel/tracing/tracing_on
```

---

**5. فحص PM Domain**

```bash
# شوف حالة الـ power domain للـ auxiliary device
cat /sys/bus/auxiliary/devices/DEVICE_NAME/power/runtime_status
# القيم المحتملة: active, suspended, error, unsupported

cat /sys/bus/auxiliary/devices/DEVICE_NAME/power/control
# القيم: on, auto

# شوف الـ power domain summary
cat /sys/kernel/debug/pm_genpd/summary 2>/dev/null | grep -A3 "DEVICE_NAME"
```

**مثال للـ output:**
```
Domain                          Status          Slave domains
foo_pd                          on
    /devices/virtual/auxiliary/foo_mod.foo_dev.0  0
```

---

**6. سكريبت شامل للـ debugging**

```bash
#!/bin/bash
# auxiliary_bus_debug.sh — سكريبت شامل لتشخيص الـ auxiliary bus

DEVICE=${1:-""}
echo "======================================"
echo "  Auxiliary Bus Debug Report"
echo "======================================"
echo ""

echo "[1] All Auxiliary Devices:"
ls /sys/bus/auxiliary/devices/ 2>/dev/null || echo "  (none)"
echo ""

echo "[2] All Auxiliary Drivers:"
ls /sys/bus/auxiliary/drivers/ 2>/dev/null || echo "  (none)"
echo ""

echo "[3] Unbound Devices:"
for dev in /sys/bus/auxiliary/devices/*/; do
    if [ ! -L "$dev/driver" ]; then
        echo "  UNBOUND: $(basename $dev)"
        echo "    MODALIAS: $(cat $dev/modalias 2>/dev/null)"
    fi
done
echo ""

echo "[4] Kernel Messages (last 30):"
dmesg | grep -iE "auxiliary" | tail -30
echo ""

if [ -n "$DEVICE" ]; then
    echo "[5] Device Details: $DEVICE"
    ls -la /sys/bus/auxiliary/devices/$DEVICE/ 2>/dev/null
    echo "  MODALIAS: $(cat /sys/bus/auxiliary/devices/$DEVICE/modalias 2>/dev/null)"
    echo "  Driver: $(readlink /sys/bus/auxiliary/devices/$DEVICE/driver 2>/dev/null | xargs basename)"
    echo "  PM Status: $(cat /sys/bus/auxiliary/devices/$DEVICE/power/runtime_status 2>/dev/null)"
fi
```

استخدامه:
```bash
chmod +x auxiliary_bus_debug.sh
./auxiliary_bus_debug.sh foo_mod.foo_dev.0
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: بورد RK3562 — الـ HDMI audio لا يظهر كـ device

#### العنوان
الـ auxiliary device للـ HDMI audio مش بيظهر في `/sys/bus/auxiliary/devices`

#### السياق
شركة بتبني Android TV box على الـ **RK3562**. الـ SoC فيه IP واحد بيتحكم في كل حاجة صوتية — HDMI audio، Soundwire، وسماعات داخلية. المهندس عمل driver للـ audio core وحاول يسجّل auxiliary device للـ HDMI audio sub-function.

#### المشكلة
بعد `insmod` للـ parent driver، مفيش أي device بيظهر في:
```bash
ls /sys/bus/auxiliary/devices/
```
والـ kernel log فيه:
```
auxiliary_bus: auxiliary_device has a NULL name
```

#### التحليل
الكود في `auxiliary_device_init()`:

```c
int auxiliary_device_init(struct auxiliary_device *auxdev)
{
    struct device *dev = &auxdev->dev;

    if (!dev->parent) {
        pr_err("auxiliary_device has a NULL dev->parent\n");
        return -EINVAL;
    }

    if (!auxdev->name) {                          /* ← هنا بيفشل */
        pr_err("auxiliary_device has a NULL name\n");
        return -EINVAL;
    }

    dev->bus = &auxiliary_bus_type;
    device_initialize(&auxdev->dev);
    ...
}
```

الـ `auxdev->name` لازم يتحدد **قبل** استدعاء `auxiliary_device_init()`. المهندس نسي يحدد الـ `name` field في الـ struct قبل الاستدعاء.

ولما `auxiliary_device_init()` بيفشل، الـ `device_initialize()` ما اتشغلتش — يعني الـ caller مسؤول يحرر الميموري مباشرة بدون ما يستدعي `auxiliary_device_uninit()`.

#### الحل
```c
/* قبل الاستدعاء */
struct rk3562_hdmi_audio {
    struct auxiliary_device auxdev;
    void __iomem *regs;
};

struct rk3562_hdmi_audio *haudio = kzalloc(sizeof(*haudio), GFP_KERNEL);

/* الخطأ: نسي يحدد الاسم */
// haudio->auxdev.name = "hdmi_audio";   /* السطر ده كان ناقص */

/* الصح */
haudio->auxdev.name   = "hdmi_audio";   /* ← لازم قبل auxiliary_device_init */
haudio->auxdev.id     = 0;
haudio->auxdev.dev.parent = &pdev->dev;
haudio->auxdev.dev.release = rk3562_hdmi_audio_release;

ret = auxiliary_device_init(&haudio->auxdev);
if (ret) {
    kfree(haudio);  /* هنا نحرر مباشرة لأن device_initialize لم تشتغل */
    return ret;
}

ret = auxiliary_device_add(&haudio->auxdev);
```

#### الدرس المستفاد
الـ `auxiliary_device_init()` بتعمل validation قبل أي حاجة. الـ `name` و`dev.parent` و`dev.release` — الثلاثة دول **شرط لازم** قبل استدعاء أي function. وفيه فرق مهم في error handling: لو `_init()` فشلت → `kfree` مباشر. لو `_add()` فشلت → لازم `auxiliary_device_uninit()` لأن `device_initialize()` اشتغلت.

---

### السيناريو الثاني: STM32MP1 — الـ driver مش بيعمل probe رغم وجود الـ device

#### العنوان
الـ auxiliary driver مش بيعمل bind مع الـ device — مشكلة في الـ match name

#### السياق
منتج industrial gateway على **STM32MP1**. الـ SoC بيستخدم Ethernet MAC مع MDIO sub-function. المهندس عمل parent driver اسمه `stm32_eth.ko` وعمل auxiliary device اسمه `mdio`. وعمل driver منفصل `stm32_mdio.ko` يشتغل على الـ device ده.

#### المشكلة
الـ device موجود:
```bash
ls /sys/bus/auxiliary/devices/
# stm32_eth.mdio.0
```
لكن الـ driver مش بيعمل probe. الـ dmesg نظيف — مفيش errors.

#### التحليل
دالة `auxiliary_match_id()` هي اللي بتقرر مين يـbind مع مين:

```c
static const struct auxiliary_device_id *auxiliary_match_id(
    const struct auxiliary_device_id *id,
    const struct auxiliary_device *auxdev)
{
    const char *auxdev_name = dev_name(&auxdev->dev);
    /* الاسم الكامل: "stm32_eth.mdio.0" */

    const char *p = strrchr(auxdev_name, '.');
    /* p يشاور على "." الأخيرة — يعني قبل "0" */

    int match_size = p - auxdev_name;
    /* match_size = طول "stm32_eth.mdio" = 14 حرف */

    for (; id->name[0]; id++) {
        if (strlen(id->name) == match_size &&
            !strncmp(auxdev_name, id->name, match_size))
            return id;
        /* بيقارن "stm32_eth.mdio" مع id->name */
    }
    return NULL;
}
```

الجزء اللي بيستخدمه الـ match هو كل حاجة **قبل النقطة الأخيرة** — يعني `stm32_eth.mdio`. المهندس كتب في الـ id_table:

```c
/* خطأ */
static const struct auxiliary_device_id stm32_mdio_id_table[] = {
    { .name = "stm32_mdio.mdio" },  /* ← اسم الـ driver module مش parent */
    {}
};
```

لكن الاسم الصح هو `stm32_eth.mdio` (اسم الـ KBUILD_MODNAME للـ parent + اسم الـ device).

#### الحل
```c
/* الصح */
static const struct auxiliary_device_id stm32_mdio_id_table[] = {
    { .name = "stm32_eth.mdio" },  /* KBUILD_MODNAME للـ parent . اسم الـ auxdev */
    {}
};
MODULE_DEVICE_TABLE(auxiliary, stm32_mdio_id_table);
```

للتحقق:
```bash
# شوف الاسم الكامل للـ device
cat /sys/bus/auxiliary/devices/stm32_eth.mdio.0/uevent
# MODALIAS=auxiliary:stm32_eth.mdio

# قارنه بالـ id_table في الـ driver
modinfo stm32_mdio.ko | grep alias
# alias: auxiliary:stm32_eth.mdio
```

#### الدرس المستفاد
الـ auxiliary bus بيستخدم نظام match اسمه **`<parent_modname>.<auxdev_name>`**. مش اسم الـ driver اللي بيعمل probe، لكن اسم الـ **parent** اللي سجّل الـ device. دايما استخدم `uevent` في sysfs للتحقق من الـ MODALIAS الفعلي.

---

### السيناريو الثاني عشر: i.MX8 — kernel panic عند rmmod للـ parent driver

#### العنوان
Kernel panic عند تفريغ الـ parent module — Use-After-Free في الـ auxiliary device

#### السياق
بورد custom bring-up على **i.MX8MP**. المنتج عبارة عن camera processing unit. الـ parent driver بيسجّل auxiliary device للـ ISP sub-function. المهندس بيحاول يعمل hot-reload للـ driver أثناء الاختبار.

#### المشكلة
```bash
rmmod imx8_camera
# [ 142.334521] BUG: unable to handle page fault for address: 00000000dead0000
# [ 142.334544] Kernel panic - not syncing: Fatal exception
```

#### التحليل
المهندس كتب `remove()` كده:

```c
static int imx8_camera_remove(struct platform_device *pdev)
{
    struct imx8_cam_priv *priv = platform_get_drvdata(pdev);

    /* خطأ: بس delete بدون uninit */
    auxiliary_device_delete(&priv->isp_auxdev.auxdev);

    kfree(priv);  /* ← حرر الميموري */
    return 0;
}
```

الـ `auxiliary_device_delete()` هي:
```c
static inline void auxiliary_device_delete(struct auxiliary_device *auxdev)
{
    device_del(&auxdev->dev);  /* بتشيل الـ device من الـ bus */
}
```

لكن `device_del()` مش بتحرر الـ reference كلها — لازم بعدها `auxiliary_device_uninit()`:
```c
static inline void auxiliary_device_uninit(struct auxiliary_device *auxdev)
{
    mutex_destroy(&auxdev->sysfs.lock);
    put_device(&auxdev->dev);  /* ← تنزيل آخر reference */
}
```

المشكلة: لو حد تاني (مثلاً sysfs أو الـ auxiliary driver نفسه) لسه شايل reference على `dev`، الـ release callback هيتشغل بعدين — بعد ما `kfree(priv)` اتعملت. فـ Use-After-Free.

والـ `auxiliary_device_release()` الموجودة في الكود:
```c
static void auxiliary_device_release(struct device *dev)
{
    struct auxiliary_device *auxdev = to_auxiliary_dev(dev);

    of_node_put(dev->of_node);
    kfree(auxdev);  /* ← ده بيحرر الميموري، مش المستخدم */
}
```

#### الحل
الطريقة الصحيحة — استخدام `devm`:

```c
/* في probe */
static int imx8_camera_probe(struct platform_device *pdev)
{
    struct imx8_cam_priv *priv;

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);

    /* استخدم devm_auxiliary_device_create بدل الطريقة اليدوية */
    priv->isp_auxdev = devm_auxiliary_device_create(
        &pdev->dev,
        "isp",           /* devname */
        priv,            /* platform_data */
    );
    /* الـ devm هيتكفل بـ delete + uninit تلقائياً عند الـ remove */
}
```

أو لو عايز يدوي:
```c
static void imx8_camera_remove(struct platform_device *pdev)
{
    struct imx8_cam_priv *priv = platform_get_drvdata(pdev);

    auxiliary_device_delete(&priv->isp_auxdev.auxdev);
    auxiliary_device_uninit(&priv->isp_auxdev.auxdev);
    /* لا تعمل kfree هنا — الـ release callback هيعملها */
}
```

#### الدرس المستفاد
الـ auxiliary device lifecycle فيه **reference counting**. الـ `delete` بتشيله من الـ bus، لكن الـ `uninit` بتنزّل الـ reference. الـ `kfree` اليدوي ممنوع بعد `device_initialize()` — اتركها للـ `release` callback. الأسهل والأأمن هو `devm_auxiliary_device_create()` اللي بتتكفل بكل ده تلقائياً.

---

### السيناريو الرابع: AM62x — الـ USB auxiliary device مش بيشتغل بعد suspend/resume

#### العنوان
الـ USB sub-function بتوقف عن الشغل بعد system suspend على AM62x IoT gateway

#### السياق
IoT sensor hub على **TI AM62x**. الـ USB controller فيه auxiliary device للـ DRD (Dual Role Device) sub-function. المنتج بيعمل suspend/resume بشكل متكرر لتوفير الطاقة. بعد أول suspend/resume cycle، الـ USB بيوقف عن الشغل.

#### المشكلة
```bash
echo mem > /sys/power/state
# بعد الاستيقاظ:
lsusb
# مفيش أي devices
dmesg | grep usb
# am62_usb_drd: Failed to resume
```

#### التحليل
الـ auxiliary bus بيستخدم `auxiliary_dev_pm_ops`:

```c
static const struct dev_pm_ops auxiliary_dev_pm_ops = {
    SET_RUNTIME_PM_OPS(pm_generic_runtime_suspend,
                       pm_generic_runtime_resume,
                       NULL)
    SET_SYSTEM_SLEEP_PM_OPS(pm_generic_suspend,
                            pm_generic_resume)
};
```

الـ `pm_generic_suspend` و`pm_generic_resume` بيشتغلوا على مستوى الـ **auxiliary device** — بينادوا الـ `suspend`/`resume` callbacks في `struct auxiliary_driver`.

المهندس عمل الـ auxiliary driver بدون `suspend`/`resume` callbacks:

```c
static struct auxiliary_driver am62_drd_auxdrv = {
    .probe   = am62_drd_probe,
    .remove  = am62_drd_remove,
    /* مفيش suspend/resume */
    .id_table = am62_drd_id_table,
};
```

لما الـ system بيعمل resume، الـ `pm_generic_resume()` بتدور على الـ `resume` op في الـ driver — مش لاقياه — فالـ device بيفضل في حالة suspended.

بالإضافة لكده، الـ `auxiliary_bus_probe()` بيعمل:

```c
ret = dev_pm_domain_attach(dev, PD_FLAG_ATTACH_POWER_ON |
                                PD_FLAG_DETACH_POWER_OFF);
```

يعني الـ PM domain بيتربط تلقائياً عند الـ probe. لو الـ PM domain فيه مشكلة في الـ AM62x، الـ device ممكن ما يصحيش.

#### الحل
```c
static int am62_drd_suspend(struct auxiliary_device *auxdev,
                            pm_message_t state)
{
    struct am62_drd_priv *priv = auxiliary_get_drvdata(auxdev);

    /* احفظ الـ state */
    am62_drd_save_context(priv);
    clk_disable_unprepare(priv->clk);
    return 0;
}

static int am62_drd_resume(struct auxiliary_device *auxdev)
{
    struct am62_drd_priv *priv = auxiliary_get_drvdata(auxdev);

    clk_prepare_enable(priv->clk);
    am62_drd_restore_context(priv);
    return 0;
}

static struct auxiliary_driver am62_drd_auxdrv = {
    .probe    = am62_drd_probe,
    .remove   = am62_drd_remove,
    .suspend  = am62_drd_suspend,   /* ← مضاف */
    .resume   = am62_drd_resume,    /* ← مضاف */
    .id_table = am62_drd_id_table,
};
```

للـ debug:
```bash
# شوف حالة الـ PM domain
cat /sys/bus/auxiliary/devices/am62_usb.drd.0/power/runtime_status

# شوف لو الـ PM domain متربط
cat /sys/bus/auxiliary/devices/am62_usb.drd.0/power/control
```

#### الدرس المستفاد
الـ auxiliary bus بيوفر **PM hooks تلقائية** عبر `auxiliary_dev_pm_ops`. لكن لو الـ `auxiliary_driver` ما عندوش `suspend`/`resume`، الـ generic PM هيـ"ينجح" بدون ما يعمل حاجة — الـ device هيفضل frozen. دايما implement الـ PM callbacks لأي peripheral بيحتاج hardware re-initialization بعد الـ sleep.

---

### السيناريو الخامس: Allwinner H616 — conflict في الـ device ID عند تشغيل متعدد الـ instances

#### العنوان
فشل تسجيل auxiliary devices متعددة على Allwinner H616 TV Box — duplicate device name

#### السياق
Android TV box على **Allwinner H616**. الـ SoC فيه **اتنين** HDMI outputs (HDMI0 و HDMI1). المهندس عمل parent driver للـ display engine، وبيسجّل auxiliary device لكل HDMI output.

#### المشكلة
```bash
dmesg | grep auxiliary
# sunxi_de: auxiliary device dev_set_name failed: -EEXIST
# sunxi_de: adding auxiliary device failed!: -EEXIST
```

بس HDMI0 شغال، HDMI1 ما بيشتغلش.

#### التحليل
دالة `__auxiliary_device_add()` بتستخدم:

```c
int __auxiliary_device_add(struct auxiliary_device *auxdev, const char *modname)
{
    struct device *dev = &auxdev->dev;
    int ret;

    ret = dev_set_name(dev, "%s.%s.%d", modname, auxdev->name, auxdev->id);
    /*
     * الاسم النهائي: "sunxi_de.hdmi.0"
     * الفورمات: <modname>.<auxdev->name>.<auxdev->id>
     */
    ...
    ret = device_add(dev);
    /* لو في device بنفس الاسم موجود → -EEXIST */
}
```

الاسم النهائي بيتكون من **ثلاث قطع**: الـ modname، الـ name، والـ **id**. المهندس عمل كده:

```c
/* مشكلة: نفس الـ id للاتنين */
for (i = 0; i < 2; i++) {
    hdmi[i].auxdev.name = "hdmi";
    hdmi[i].auxdev.id   = 0;      /* ← دايما 0 للاتنين! */
    hdmi[i].auxdev.dev.parent = &pdev->dev;
    auxiliary_device_init(&hdmi[i].auxdev);
    auxiliary_device_add(&hdmi[i].auxdev);
}
/* النتيجة: اتنين بيحاولوا يسجّلوا "sunxi_de.hdmi.0" → EEXIST */
```

#### الحل
```c
for (i = 0; i < 2; i++) {
    hdmi[i].auxdev.name = "hdmi";
    hdmi[i].auxdev.id   = i;      /* ← 0 للأول، 1 للتاني */
    hdmi[i].auxdev.dev.parent = &pdev->dev;
    hdmi[i].index = i;            /* نحفظ الـ index للاستخدام في probe */

    ret = auxiliary_device_init(&hdmi[i].auxdev);
    if (ret) {
        kfree(&hdmi[i]);
        goto err;
    }

    ret = auxiliary_device_add(&hdmi[i].auxdev);
    if (ret) {
        auxiliary_device_uninit(&hdmi[i].auxdev);
        goto err;
    }
}
/*
 * النتيجة الصح:
 * sunxi_de.hdmi.0  ← HDMI0
 * sunxi_de.hdmi.1  ← HDMI1
 */
```

للتحقق:
```bash
ls /sys/bus/auxiliary/devices/
# sunxi_de.hdmi.0
# sunxi_de.hdmi.1

# في الـ auxiliary driver، نعرف أي instance ده:
static int sunxi_hdmi_probe(struct auxiliary_device *auxdev,
                            const struct auxiliary_device_id *id)
{
    /* الـ id في الاسم: auxdev->id */
    dev_info(&auxdev->dev, "probing HDMI%d\n", auxdev->id);
    ...
}
```

#### الدرس المستفاد
الـ **`id` field** في `struct auxiliary_device` هو ما يفرّق بين instances متعددة من نفس النوع. لو عندك N devices بنفس الـ `name` و نفس الـ parent، لازم كل واحد يأخذ `id` فريد. الاسم النهائي دايما `<modname>.<name>.<id>` — وأي تكرار بيسبب `-EEXIST` في `device_add()`.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لأي حاجة تخص kernel Linux — زي صحيفة متخصصة للمطورين.

| المقال | الوصف |
|--------|--------|
| [Managing multifunction devices with the auxiliary bus](https://lwn.net/Articles/840416/) | المقال الأساسي اللي يشرح ليه اتعمل الـ auxiliary bus، وكيف يحل مشكلة الـ NIC + RDMA، وتفاصيل الـ patch set |
| [Add Intel Ethernet Protocol Driver for RDMA (irdma)](https://lwn.net/Articles/856287/) | أول استخدام فعلي للـ auxiliary bus في الكيرنل — Intel irdma driver |
| [A fresh look at the kernel's device model](https://lwn.net/Articles/645810/) | خلفية ضرورية عن device model في الكيرنل قبل ما تفهم ليه الـ auxiliary bus اتصمم كده |

---

### التوثيق الرسمي في الكيرنل

الملف الرئيسي في المصدر نفسه بيشير صراحة لـ:

```
Documentation/driver-api/auxiliary_bus.rst
```

الروابط الحية للتوثيق الرسمي:

- **[Auxiliary Bus — The Linux Kernel documentation (latest)](https://docs.kernel.org/driver-api/auxiliary_bus.html)**
  التوثيق الكامل والمحدث — يشرح الـ API والـ lifecycle والأمثلة.

- **[Auxiliary Bus — Linux 5.11 documentation](https://www.kernel.org/doc/html/v5.11/driver-api/auxiliary_bus.html)**
  النسخة الأصلية من وقت ما اتضاف الـ subsystem.

- **[GitHub — torvalds/linux — auxiliary_bus.rst](https://github.com/torvalds/linux/blob/master/Documentation/driver-api/auxiliary_bus.rst)**
  المصدر الحي على GitHub — مفيد لمتابعة التغييرات.

---

### ملفات المصدر المباشرة

| الملف | الوصف |
|-------|--------|
| `drivers/base/auxiliary.c` | الـ implementation الكاملة — القلب اللي بنشرحه |
| `include/linux/auxiliary_bus.h` | تعريفات الـ structs والـ macros والـ API العلني |

---

### كوميتات Git المهمة

- **[Add auxiliary bus support — الـ commit الأصلي (OpenDev mirror)](https://opendev.org/starlingx/kernel/commit/ef3c9a46180d8f4f70ba544693bbc70d7f9dd9a0)**
  الكوميت اللي أدخل الـ auxiliary bus للكيرنل في الـ 5.11 merge window.

- **[PATCH v4 01/10 — Add auxiliary bus support (LKML archive)](https://lkml.iu.edu/hypermail/linux/kernel/2011.1/07745.html)**
  النسخة الرابعة من الـ patch set قبل الدمج — تشوف فيها النقاشات التقنية مع Greg Kroah-Hartman.

- **[PATCH 0/4 — Auxiliary bus driver for Intel PCIe VSEC/DVSEC](https://lore.kernel.org/lkml/20211120231705.189969-1-david.e.box@linux.intel.com/)**
  مثال على توسعة الاستخدام — Intel PMT بدل MFD.

---

### نقاشات Mailing List

- **[Re: PATCH v4 01/10 — Add auxiliary bus support (Leon Romanovsky)](https://lkml.kernel.org/lkml/20201117071641.GN47002@unreal/)**
  تعليقات مهندس RDMA على الـ design — مهم تشوف ليه الـ RDMA subsystem اهتم بالموضوع ده.

- **[RE: resend/standalone PATCH v4 — Add auxiliary bus support](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2404848.html)**
  النقاش النهائي قبل الـ merge — بتشوف فيه قرارات الـ design الأخيرة.

- **[soundwire: intel: move to auxiliary bus — Patchwork](https://patchwork.kernel.org/project/alsa-devel/patch/20210511052132.28150-1-yung-chuan.liao@linux.intel.com/)**
  Intel SoundWire بتتحول للـ auxiliary bus — مثال حقيقي على migration.

- **[RFC: bus/auxiliary — introduce auxiliary bus in DPDK](https://inbox.dpdk.org/dev/20210413032329.25551-1-xuemingl@nvidia.com/T/)**
  NVIDIA اقترحت نفس الفكرة في DPDK بعد شوفت الـ kernel implementation.

---

### Phoronix Coverage

- **[Auxiliary Bus Support Coming To Linux 5.11 — Phoronix](https://www.phoronix.com/news/Linux-5.11-Auxiliary-Bus)**
  تغطية وقت الإعلان — مفيد كـ timeline ومزود بتفاصيل بسيطة.

---

### kernelnewbies.org

الـ auxiliary bus اتذكر في عدة صفحات kernel release notes:

| الصفحة | ما يخص الـ auxiliary bus |
|--------|---------------------------|
| **[Linux_5.11](https://kernelnewbies.org/Linux_5.11)** | أول ظهور — إضافة الـ infrastructure الأساسية |
| **[Linux_5.17](https://kernelnewbies.org/Linux_5.17)** | إضافة aux-bus support لـ drivers جديدة |
| **[Linux_6.1](https://kernelnewbies.org/Linux_6.1)** | microchip pci1xxxx يستخدم auxiliary bus لـ PIO |
| **[Linux_6.2](https://kernelnewbies.org/Linux_6.2)** | auxiliary device support في mana driver |
| **[Linux_6.7](https://kernelnewbies.org/Linux_6.7)** | PTP auxiliary bus support |
| **[Linux_6.11](https://kernelnewbies.org/Linux_6.11)** | auxiliary bus IRQs sysfs |

---

### elinux.org

لا توجد صفحة مخصصة للـ auxiliary bus على elinux.org — هذا الـ subsystem حديث نسبياً (5.11+) وتوثيقه مركّز في kernel.org الرسمي وLWN.net.

---

### كتب مقترحة

#### Linux Device Drivers (LDD3)

> **تحذير**: الكتاب قديم (kernel 2.6) وما بيغطي الـ auxiliary bus لأنه اتضاف في 5.11. لكن لازم تقرأه كـ foundation:

| الفصل | الصلة بالموضوع |
|-------|----------------|
| Chapter 14 — The Linux Device Model | أساس الـ `bus_type`, `device`, `device_driver` |
| Chapter 3 — Char Drivers | فهم الـ driver lifecycle |

- **[LDD3 — PDF مجاني رسمي](https://lwn.net/Kernel/LDD3/)**

#### Linux Kernel Development — Robert Love (3rd Edition)

| الفصل | الصلة |
|-------|-------|
| Chapter 17 — Devices and Modules | الـ device model وكيف يشتغل |

- **المؤلف**: Robert Love
- **الناشر**: Addison-Wesley
- **الطبعة**: الثالثة (2010)

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

| الفصل | الصلة |
|-------|-------|
| Chapter 8 — Device Driver Basics | الـ bus architecture في embedded |
| Chapter 13 — Development Tools | debugging الـ drivers |

---

### مصطلحات البحث

لو حابب تعمق أكتر، استخدم الـ search terms دي:

```
# للبحث في LKML
auxiliary_bus linux kernel
auxiliary_device_init auxiliary_driver_register
EXPORT_SYMBOL_NS auxiliary bus

# للبحث في kernel source
git log --all --oneline -- drivers/base/auxiliary.c
git log --all --oneline -- include/linux/auxiliary_bus.h

# للبحث عن users في الكيرنل نفسه
grep -r "auxiliary_driver_register\|auxiliary_device_add" drivers/

# مصطلحات ذات صلة
"ancillary bus" OR "virtual bus" linux kernel (الأسماء القديمة للـ patch)
"auxiliary bus" "Sound Open Firmware" SOF
"auxiliary bus" RDMA mlx5 irdma
```

---

### خلاصة المصادر الأهم

```
أولوية قراءة المصادر:
┌─────────────────────────────────────────────────────────────┐
│ 1. LWN Articles/840416  ← ابدأ هنا — الفهم الكامل          │
│ 2. docs.kernel.org      ← التوثيق الرسمي الكامل             │
│ 3. LKML v4 patch        ← شوف قرارات الـ design              │
│ 4. kernelnewbies Linux_5.11 ← timeline والـ context          │
│ 5. LDD3 Chapter 14      ← الأساس النظري للـ device model     │
└─────────────────────────────────────────────────────────────┘
```
## Phase 8: Writing simple module

### الفكرة: نراقب كل مرة تُضاف فيها auxiliary device للـ bus

**الـ `__auxiliary_device_add`** هي الدالة الأساسية اللي بتسجّل أي auxiliary device على الـ bus. كل مرة driver يعمل `auxiliary_device_add()`، الكود بيمشي عبرها. ممتازة للـ hook لأنها:

1. **`EXPORT_SYMBOL_GPL`** — يعني الـ kernel بيعرفها ويمكن الـ kprobe يقدر يضرب عليها.
2. بتاخد اسم الـ device واسم الـ modname، فنقدر نطبع معلومات مفيدة جداً.
3. آمنة — القراءة فقط، مش بنغيّر أي حاجة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * auxdev_spy.c
 *
 * kprobe على __auxiliary_device_add:
 * نطبع اسم الـ device واسم الـ modname كل مرة auxiliary device تتسجّل.
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/auxiliary_bus.h>/* struct auxiliary_device */
#include <linux/kernel.h>       /* pr_info */

/*
 * نحتاج auxiliary_bus.h عشان نعرف struct auxiliary_device
 * ونقدر نقرأ منها الـ name والـ id.
 *
 * نحتاج kprobes.h عشان هي اللي بتوفّر آلية الـ hook بدون ما نعدّل الـ kernel.
 */

/* --- الـ pre-handler: بيُستدعى قبل ما __auxiliary_device_add تشتغل --- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86-64:
     *   rdi = أول argument  → struct auxiliary_device *auxdev
     *   rsi = ثاني argument → const char *modname
     *
     * على ARM64:
     *   x0 = أول argument
     *   x1 = ثاني argument
     *
     * نستخدم regs_get_kernel_argument() اللي portable وتشتغل على كل architectures.
     */
    struct auxiliary_device *auxdev =
        (struct auxiliary_device *)regs_get_kernel_argument(regs, 0);
    const char *modname =
        (const char *)regs_get_kernel_argument(regs, 1);

    /*
     * نتحقق إن المؤشرات مش NULL قبل ما نقرأ منها،
     * لأن الـ kprobe بيشتغل في سياق حساس جداً.
     */
    if (auxdev && modname && auxdev->name)
        pr_info("[auxdev_spy] modname=%s  dev_name=%s  id=%u\n",
                modname, auxdev->name, auxdev->id);
    else
        pr_info("[auxdev_spy] __auxiliary_device_add called (NULL args)\n");

    /* إرجاع 0 يعني: كمّل تنفيذ الدالة الأصلية بشكل طبيعي */
    return 0;
}

/* --- تعريف الـ kprobe --- */
static struct kprobe kp = {
    /*
     * نحدد اسم الدالة اللي نبي نحقنها.
     * الـ kernel يبحث عنها في جدول الـ symbols تلقائياً.
     */
    .symbol_name = "__auxiliary_device_add",
    .pre_handler = handler_pre,
};

/* --- module_init --- */
static int __init auxdev_spy_init(void)
{
    int ret;

    /*
     * register_kprobe: يحجز نقطة توقف (breakpoint) على الدالة المحددة.
     * لو فشل (مثلاً الدالة inlined أو محمية) نرجع الخطأ مباشرة.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[auxdev_spy] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[auxdev_spy] planted kprobe on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* --- module_exit --- */
static void __exit auxdev_spy_exit(void)
{
    /*
     * unregister_kprobe ضروري جداً في الـ exit:
     * لو تركنا الـ kprobe بعد ما الـ module اتفرغ من الذاكرة،
     * الـ kernel هيحاول يستدعي handler_pre اللي مش موجودة → kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("[auxdev_spy] kprobe removed, goodbye.\n");
}

module_init(auxdev_spy_init);
module_exit(auxdev_spy_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Auxiliary Bus Explorer");
MODULE_DESCRIPTION("kprobe on __auxiliary_device_add to log auxiliary device registrations");
```

---

### شرح كل جزء بالتفصيل

#### `#include`s

| Header | ليه محتاجينه |
|---|---|
| `linux/module.h` | `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe`, `regs_get_kernel_argument` |
| `linux/auxiliary_bus.h` | تعريف `struct auxiliary_device` عشان نقرأ `name` و`id` |
| `linux/kernel.h` | `pr_info`, `pr_err` |

**الـ `kprobes.h`** هو قلب الموضوع — بيوفّر آلية الـ dynamic instrumentation اللي بتخلّينا نضع hook على أي دالة في الـ kernel بدون ما نعدّل سطر واحد في الـ source.

---

#### `handler_pre` — الـ callback

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

- **`struct pt_regs *regs`**: snapshot للـ registers اللحظة اللي لمسنا فيها الدالة. منه نستخرج الـ arguments.
- **`regs_get_kernel_argument(regs, 0)`**: macro portable يجيب الـ argument رقم N بغض النظر عن الـ architecture (x86/ARM/RISC-V).
- بنقرأ `auxdev->name` و`auxdev->id` و`modname` — يعني بنعرف exactly أي device بيتسجّل ومن أي module.
- **إرجاع 0**: أي قيمة تانية بتعمل fault injection وهو مش هدفنا هنا.

---

#### `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "__auxiliary_device_add",
    .pre_handler = handler_pre,
};
```

**الـ `.symbol_name`** بدل ما نحط عنوان hardcoded، بنحط اسم الدالة والـ kernel يحل العنوان وقت الـ register. أكثر أماناً وشغّال على أي kernel version طالما الدالة موجودة.

---

#### `module_init` / `module_exit`

**الـ `register_kprobe`** في الـ init بيضع breakpoint افتراضي على بداية `__auxiliary_device_add`. من اللحظة دي أي استدعاء ليها سيمر عبر `handler_pre` أولاً.

**الـ `unregister_kprobe`** في الـ exit **إلزامي** — تخيّل إنك شيّلت اللافتة من الشارع لكن سابت الحفرة جوه الرصيف. بدون الـ unregister، الـ kernel سيستدعي callback في ذاكرة اتحررت = crash فوري.

---

### Makefile للتجربة

```makefile
obj-m += auxdev_spy.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة عملية

```bash
# بناء الـ module
make

# تحميله
sudo insmod auxdev_spy.ko

# مشاهدة الـ output (جرّب تشغّل أي driver بيستخدم auxiliary bus مثل IntelSOF أو mlx5)
sudo dmesg | grep auxdev_spy

# تفريغه
sudo rmmod auxdev_spy

# مثال للـ output المتوقع:
# [auxdev_spy] planted kprobe on __auxiliary_device_add @ ffffffffc0a1b230
# [auxdev_spy] modname=sof-audio-pci  dev_name=codec  id=0
# [auxdev_spy] modname=mlx5_core      dev_name=rdma   id=0
# [auxdev_spy] kprobe removed, goodbye.
```

---

### لماذا `__auxiliary_device_add` وليس `auxiliary_device_init`؟

**الـ `auxiliary_device_init`** تعطيك الـ device قبل ما يُضبط اسمه الكامل. أما **`__auxiliary_device_add`** فبعدها مباشرة تجد `dev_set_name()` قد اتنفّذت، يعني الاسم جاهز بالصيغة `modname.devname.id` — المعلومة أكثر ثراءً وأوضح للتشخيص.
