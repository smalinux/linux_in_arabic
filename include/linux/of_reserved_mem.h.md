## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الـ `include/linux/of_reserved_mem.h` هو واجهة (header) لنظام **Reserved Memory** داخل إطار عمل **Open Firmware / Flattened Device Tree** في Linux kernel. يُعرّف هذا الملف الـ structs والـ macros والـ API التي تتيح للـ kernel قراءة مناطق الذاكرة المحجوزة من الـ **Device Tree (DT)**، وربطها بالأجهزة التي تحتاج إليها.

---

### القصة البسيطة — تخيّل مبنى سكني

تخيّل أن ذاكرة الـ RAM هي شقق في مبنى كبير. عند الإقلاع، يأتي مدير المبنى (الـ kernel) ويوزّع الشقق على السكان (التطبيقات والأجهزة). لكن بعض الشقق مُعلَّق عليها لافتة **"محجوز — لا تُلمس"** منذ البداية — قبل أن يصل أي مستأجر. هذه هي الـ **reserved memory regions**.

**من يضع هذه اللافتات؟** ملف الـ Device Tree — وهو وثيقة نصية تصف عتاد (hardware) الجهاز. فيها يكتب المهندس الذي صمّم اللوحة الإلكترونية: "هذه المنطقة من العنوان `0x80000000` بحجم `16MB` محجوزة لكاميرا الـ ISP".

**مشكلة:** لكن كيف يعرف الـ device driver الخاص بالكاميرا أين بالضبط تقع هذه المنطقة؟ وكيف يطلبها؟

**الحل:** هذا الملف يوفّر الـ API الذي يربط بين الـ device driver والمنطقة المحجوزة المُعرَّفة في الـ Device Tree.

---

### المشكلة التي يحلّها هذا الملف

في أنظمة المُدمَجة (embedded systems) مثل الهواتف والكاميرات والسيارات، بعض الـ hardware components تحتاج إلى ذاكرة:
- **مستمرة فيزيائياً (physically contiguous)**: مثل كاميرا تحتاج DMA buffer لا يُقطَّع.
- **بعيدة عن وصول الـ kernel**: مثل ذاكرة مشتركة مع معالج آخر (co-processor).
- **غير مُهيَّأة (no-map)**: المنطقة محجوزة لكن الـ kernel لا يُدرجها في نظام الصفحات.
- **مُهيَّأة بطريقة خاصة**: مثل منطقة **CMA (Contiguous Memory Allocator)** أو **ramoops** (تسجيل الـ crash في RAM).

قبل هذا النظام، كان المطوّرون يُشفّرون العناوين مباشرة في الـ kernel source — وهو كابوس عند تغيير اللوحة. الـ Device Tree + هذا الـ header حلّا المشكلة بمرونة كاملة.

---

### الهيكل الجوهري: `struct reserved_mem`

```c
struct reserved_mem {
    const char              *name;      /* اسم المنطقة من الـ DT */
    unsigned long           fdt_node;   /* موقع العقدة في الـ FDT blob */
    const struct reserved_mem_ops *ops; /* عمليات init/release */
    phys_addr_t             base;       /* العنوان الفيزيائي للبداية */
    phys_addr_t             size;       /* الحجم بالبايت */
    void                    *priv;      /* بيانات خاصة بالـ driver */
};
```

هذا الـ struct يمثّل منطقة واحدة محجوزة — مثل سجل في دفتر العقارات يقول: "الشقة 5B، الطابق 3، مساحة 80م²، مؤجَّرة لشركة ISP".

---

### الـ ops: كيف تتصل الأجهزة بالذاكرة

```c
struct reserved_mem_ops {
    /* يُستدعى عندما يُربط device بالمنطقة */
    int  (*device_init)(struct reserved_mem *rmem, struct device *dev);
    /* يُستدعى عند تحرير الارتباط */
    void (*device_release)(struct reserved_mem *rmem, struct device *dev);
};
```

كل نوع من أنواع المناطق المحجوزة (CMA، no-map، shared-dma-pool) يُسجّل نفسه عبر الـ `RESERVEDMEM_OF_DECLARE` macro ويوفّر هذه الـ ops.

---

### الـ Macro السحري: `RESERVEDMEM_OF_DECLARE`

```c
#define RESERVEDMEM_OF_DECLARE(name, compat, init) \
    _OF_DECLARE(reservedmem, name, compat, init, reservedmem_of_init_fn)
```

هذا الـ macro يسجّل **handler** لنوع معيّن من المناطق المحجوزة — يُعرَّف بالـ `compatible` string في الـ Device Tree. مثلاً:

```dts
/* في ملف الـ Device Tree */
reserved-memory {
    fb_reserved: framebuffer@90000000 {
        compatible = "shared-dma-pool";
        reg = <0x90000000 0x4000000>;
        reusable;
    };
};
```

عند إقلاع الـ kernel، يجد هذا الـ node ويستدعي الـ handler المسجَّل لـ `"shared-dma-pool"`.

---

### الـ API الرئيسية

| الدالة | الغرض |
|--------|--------|
| `of_reserved_mem_device_init(dev)` | ربط الجهاز بأول منطقة محجوزة في الـ DT node (الأكثر استخداماً) |
| `of_reserved_mem_device_init_by_idx(dev, np, idx)` | ربط بمنطقة بعينها عبر الـ index |
| `of_reserved_mem_device_init_by_name(dev, np, name)` | ربط بمنطقة بعينها عبر الاسم |
| `of_reserved_mem_device_release(dev)` | تحرير الارتباط عند إزالة الجهاز |
| `of_reserved_mem_lookup(np)` | البحث عن `reserved_mem` بـ device node |
| `of_reserved_mem_region_to_resource(np, idx, res)` | تحويل المنطقة إلى `struct resource` |
| `of_reserved_mem_region_count(np)` | عدد المناطق المرتبطة بجهاز |

---

### رحلة كاملة من الإقلاع حتى استخدام الجهاز

```
Boot (early_init_dt_scan)
        │
        ▼
  يقرأ الـ kernel الـ FDT ويجد عقد reserved-memory
        │
        ▼
  يحجز المناطق في memblock (قبل أن يبدأ نظام الصفحات)
        │
        ▼
  يستدعي الـ init function لكل compatible مُسجَّل
  (مثلاً: cma_init_reserved_mem لـ "shared-dma-pool")
        │
        ▼
  الجهاز يُسجَّل في الـ kernel (device probe)
        │
        ▼
  الـ driver يستدعي of_reserved_mem_device_init(dev)
        │
        ▼
  يُعيَّن الـ DMA configuration للجهاز من المنطقة المحجوزة
        │
        ▼
  الـ driver يعمل ويستخدم الذاكرة المحجوزة
```

---

### الملفات ذات الصلة

#### Core

| الملف | الدور |
|-------|-------|
| `drivers/of/of_reserved_mem.c` | التنفيذ الكامل للـ API |
| `drivers/of/fdt.c` | قراءة الـ FDT وإطلاق early scanning |
| `drivers/of/base.c` | البنية التحتية لنظام الـ OF |

#### Headers

| الملف | الدور |
|-------|-------|
| `include/linux/of_reserved_mem.h` | **هذا الملف** — واجهة الـ API |
| `include/linux/of.h` | تعريفات `device_node`، `of_phandle_args` |
| `include/linux/of_fdt.h` | واجهة قراءة الـ Flattened Device Tree |
| `include/linux/cma.h` | الـ CMA subsystem المرتبط |
| `include/linux/memblock.h` | حجز الذاكرة المبكر |

#### أمثلة على Drivers تستخدم هذا الـ API

| الملف | الاستخدام |
|-------|-----------|
| `drivers/remoteproc/remoteproc_core.c` | ذاكرة مشتركة مع co-processors |
| `drivers/gpu/drm/` | frame buffers محجوزة |
| `drivers/media/platform/` | DMA buffers للكاميرات |
| `mm/cma.c` | تنفيذ الـ CMA natively |

#### Device Tree Bindings

| الملف | الدور |
|-------|-------|
| `Documentation/devicetree/bindings/reserved-memory/reserved-memory.yaml` | مواصفة عقد الـ reserved-memory |

---

### خلاصة

الـ `of_reserved_mem.h` هو **جسر** بين وصف العتاد في الـ Device Tree وبين الـ drivers التي تحتاج ذاكرة خاصة. بدونه، كل driver يجب أن يعرف عنوان الذاكرة المحجوزة بشكل مُشفَّر (hardcoded)، وهذا يجعل الـ kernel غير قابل للنقل بين الأجهزة المختلفة.
## Phase 2: شرح الـ OF Reserved Memory Framework

---

### المشكلة: لماذا يوجد هذا الـ Subsystem؟

في الأنظمة المدمجة (embedded systems)، كثير من الـ hardware peripherals تحتاج إلى مناطق ذاكرة **محجوزة مسبقاً** لا يمكن للـ kernel أن يستخدمها أو يُدير تخصيصها بحرية. أمثلة حقيقية:

| الجهاز | سبب الحجز |
|--------|-----------|
| GPU (مثل Mali) | frame buffer ثابت في عنوان فيزيائي معروف |
| DSP / Co-processor | shared memory للتواصل مع الـ main CPU |
| Camera ISP | contiguous DMA buffers لا يمكن تشتيتها |
| Secure World (TrustZone) | منطقة معزولة لا يلمسها الـ Linux kernel |
| Firmware blobs | مساحة لتحميل firmware خاص بالـ hardware |

**المشكلة الجوهرية:** الـ Linux memory allocator (`buddy system`) لا يعلم بوجود هذه القيود الـ hardware-specific. بدون آلية منضبطة:
- قد يُوزِّع الـ kernel هذه المناطق لاستخدامات عادية
- لا يوجد مكان موحَّد لتعريف هذه المناطق والـ drivers التي تملكها
- كل driver سيخترع حله الخاص (hard-coded addresses، custom boot args)

---

### الحل: نهج الـ Kernel

الـ kernel يحل المشكلة على **ثلاث مستويات**:

**1. التعريف في الـ Device Tree:**
المنطقة المحجوزة تُعرَّف في الـ DT تحت عقدة `reserved-memory`، مما يجعلها جزءاً من وصف الـ hardware وليس hard-coded في الكود.

**2. الحجز المبكر (Early Boot):**
الـ kernel يقرأ هذه المناطق في أبكر مراحل الـ boot (قبل تهيئة الـ buddy allocator) ويُخبر الـ memory subsystem بعدم لمسها.

**3. الربط مع الـ Drivers (Runtime):**
عند تشغيل driver يحتاج منطقة محجوزة، يستخدم الـ framework لتوصيله بالـ `reserved_mem` الصحيحة وتهيئة الـ DMA mapping للـ device تلقائياً.

---

### التشبيه الواقعي — المطار والبوابات

تخيل **مطاراً** (الـ RAM الكاملة):

- **صالة الانتظار العامة** = ذاكرة يديرها الـ kernel للمستخدمين العاديين
- **بوابات VIP محجوزة** = مناطق `reserved-memory` مخصصة لجهات محددة
- **خريطة المطار في لوحة المدخل** = الـ Device Tree الذي يصف كل بوابة ومن يملكها
- **ضابط الأمن عند كل بوابة** = الـ `reserved_mem_ops` الذي يتحكم من يدخل ويخرج
- **بطاقة الصعود (boarding pass)** = `memory-region` phandle في الـ DT — الـ device لا يصل إلا بها
- **مكتب المطار المركزي** = `of_reserved_mem_lookup()` الذي يُرشد كل جهة إلى بوابتها

الآن الربط الكامل:

| المطار | الـ Kernel |
|--------|-----------|
| خريطة البوابات في لوحة المدخل | `reserved-memory` node في الـ FDT |
| بطاقة الصعود بيد المسافر | `memory-region = <&rmem_node>` في الـ device node |
| ضابط أمن كل بوابة | `struct reserved_mem_ops { .device_init, .device_release }` |
| رقم البوابة وموقعها | `phys_addr_t base` + `phys_addr_t size` في `struct reserved_mem` |
| السجل الداخلي للمطار | `struct reserved_mem.priv` — بيانات خاصة بالـ allocator |
| ضابط التسجيل المبكر | `RESERVEDMEM_OF_DECLARE` — يسجل الـ handler قبل الـ boot |

---

### المعمارية الكاملة — ASCII Diagram

```
Device Tree (FDT)
────────────────────────────────────────────────────────────
/ {
    reserved-memory {
        #address-cells = <1>;
        #size-cells    = <1>;

        fb_region: framebuffer@0x8F000000 {   ← device_node
            compatible = "shared-dma-pool";
            reg = <0x8F000000 0x1000000>;     ← base + size
            no-map;
        };

        secure_mem: secure@0x8E000000 {
            compatible = "acme,secure-pool";
            reg = <0x8E000000 0x800000>;
            no-map;
        };
    };

    gpu@0xFF300000 {
        memory-region = <&fb_region>;         ← phandle reference
    };
};
────────────────────────────────────────────────────────────

Early Boot (before buddy allocator)
┌──────────────────────────────────────────────────────────┐
│  fdt_scan() → finds reserved-memory nodes                │
│      ↓                                                   │
│  __reserved_mem_init_node()                              │
│      ↓                                                   │
│  يبحث في جدول RESERVEDMEM_OF_DECLARE                    │
│      ↓                                                   │
│  يستدعي .init_fn(rmem) للـ handler المطابق              │
│      ↓                                                   │
│  memblock_remove(base, size)  ← يُخبر buddy بعدم اللمس  │
└──────────────────────────────────────────────────────────┘

Kernel Runtime
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│   Driver Probe                                                      │
│      │                                                              │
│      ▼                                                              │
│   of_reserved_mem_device_init(dev)                                  │
│      │                                                              │
│      ├─ يقرأ "memory-region" من device_node في الـ DT              │
│      │                                                              │
│      ├─ of_reserved_mem_lookup(np)  →  struct reserved_mem *rmem   │
│      │                                                              │
│      └─ rmem->ops->device_init(rmem, dev)                          │
│              │                                                      │
│              └─ يُهيئ DMA mapping للـ dev (CMA / IOMMU / etc.)     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────┐
                    │   struct reserved_mem │
                    ├──────────────────────┤
                    │ name: "framebuffer"  │
                    │ fdt_node: 0x...      │
                    │ base: 0x8F000000     │
                    │ size: 0x1000000      │
                    │ ops: ──────────────► │ struct reserved_mem_ops
                    │ priv: ─────────────► │   .device_init()
                    └──────────────────────┘   .device_release()
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية هي **فصل التعريف عن الاستخدام**:

- **التعريف** يحدث في الـ Device Tree (hardware description) + `RESERVEDMEM_OF_DECLARE` (software handler)
- **الاستخدام** يحدث عند probe الـ driver عبر `of_reserved_mem_device_init()`

هذا يعني أن الـ driver لا يحتاج أن يعرف العنوان الفيزيائي للمنطقة — يكتفي بطلبها بالاسم أو بالـ index، والـ framework يتولى الباقي.

---

### البنى الأساسية وعلاقتها

```
struct device_node          struct reserved_mem
(من الـ Device Tree)    ──► (تُنشأ في early boot)
       │                          │
       │ (memory-region phandle)  │
       │                          ├── base (phys_addr_t)
       ▼                          ├── size (phys_addr_t)
   GPU driver                     ├── ops ──► reserved_mem_ops
       │                          │             ├── device_init()
       │ of_reserved_mem_         │             └── device_release()
       │ device_init(dev)         │
       └──────────────────────────┘
                                  └── priv ──► allocator-specific data
                                               (مثل: cma struct *)
```

#### `struct reserved_mem` — البنية المحورية

```c
struct reserved_mem {
    const char              *name;      /* اسم المنطقة من الـ DT */
    unsigned long           fdt_node;   /* offset في الـ FDT blob */
    const struct reserved_mem_ops *ops; /* vtable: init/release */
    phys_addr_t             base;       /* العنوان الفيزيائي الأول */
    phys_addr_t             size;       /* الحجم بالـ bytes */
    void                    *priv;      /* private data للـ allocator */
};
```

الـ `priv` هنا مثير للاهتمام: كل نوع من الـ reserved memory يخزن فيه ما يحتاجه. مثلاً:
- الـ **CMA (Contiguous Memory Allocator)** يخزن فيه `struct cma *`
- الـ **shared-dma-pool** يخزن فيه بنية تتبع الـ allocations

#### `struct reserved_mem_ops` — الـ vtable

```c
struct reserved_mem_ops {
    /* يُستدعى عند ربط device بالمنطقة */
    int  (*device_init)   (struct reserved_mem *rmem, struct device *dev);
    /* يُستدعى عند فك الربط (driver unload) */
    void (*device_release)(struct reserved_mem *rmem, struct device *dev);
};
```

هذا نمط الـ **polymorphism** الكلاسيكي في الـ kernel: نفس الواجهة، سلوك مختلف حسب نوع المنطقة.

#### `reservedmem_of_init_fn` — دالة التهيئة المبكرة

```c
typedef int (*reservedmem_of_init_fn)(struct reserved_mem *rmem);
```

هذه الدالة تُستدعى **مرة واحدة** في الـ early boot لكل منطقة مطابقة. مهمتها:
1. ملء `rmem->ops` بالـ vtable المناسب
2. ملء `rmem->priv` ببيانات خاصة بالـ allocator
3. تهيئة أي structures داخلية (مثل: إنشاء CMA heap)

---

### `RESERVEDMEM_OF_DECLARE` — آلية التسجيل

```c
#define RESERVEDMEM_OF_DECLARE(name, compat, init)          \
    _OF_DECLARE(reservedmem, name, compat, init, reservedmem_of_init_fn)
```

هذا الماكرو يُنشئ entry في section خاص في الـ kernel binary (`.section "__reservedmem_of_table"`). عند الـ boot، يمشي الـ kernel على هذا الجدول ويُطابق كل منطقة في الـ DT بالـ handler المناسب حسب `compatible` string.

مثال حقيقي — تسجيل الـ CMA handler:

```c
/* في drivers/of/of_reserved_mem.c أو mm/cma.c */
RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup);
//                     ^     ^                  ^
//                     اسم   compatible في DT   دالة التهيئة
```

عندما يجد الـ kernel عقدة بـ `compatible = "shared-dma-pool"` في الـ DT، يستدعي `rmem_cma_setup(rmem)` تلقائياً.

---

### الـ API المُصدَّر للـ Drivers

#### تهيئة الـ device بالمنطقة

```c
/* الأبسط: تأخذ أول memory-region في الـ DT node */
int of_reserved_mem_device_init(struct device *dev);

/* تأخذ منطقة بالـ index (للـ devices التي تحتاج أكثر من منطقة) */
int of_reserved_mem_device_init_by_idx(struct device *dev,
                                       struct device_node *np,
                                       int idx);

/* تأخذ منطقة بالاسم */
int of_reserved_mem_device_init_by_name(struct device *dev,
                                        struct device_node *np,
                                        const char *name);
```

#### تحرير الربط

```c
void of_reserved_mem_device_release(struct device *dev);
```

#### البحث المباشر

```c
/* إذا أراد الـ driver الوصول المباشر لبيانات المنطقة */
struct reserved_mem *of_reserved_mem_lookup(struct device_node *np);
```

#### استخراج الـ resource

```c
/* تحويل المنطقة إلى struct resource (start/end/flags) */
int of_reserved_mem_region_to_resource(const struct device_node *np,
                                       unsigned int idx,
                                       struct resource *res);

/* بالاسم */
int of_reserved_mem_region_to_resource_byname(const struct device_node *np,
                                              const char *name,
                                              struct resource *res);

/* عدد المناطق المرتبطة بـ device */
int of_reserved_mem_region_count(const struct device_node *np);
```

---

### ماذا يملك هذا الـ Framework؟ وماذا يُفوِّض؟

| المسؤولية | يملكها Framework | يُفوِّضها للـ Driver/Allocator |
|-----------|:---:|:---:|
| قراءة `reserved-memory` من الـ FDT | ✓ | |
| الحجز المبكر عبر `memblock` | ✓ | |
| مطابقة `compatible` بالـ handler | ✓ | |
| بنية `struct reserved_mem` | ✓ | |
| تهيئة الـ DMA ops للـ device | | ✓ (في `device_init`) |
| إدارة الـ allocations داخل المنطقة | | ✓ (CMA, genpool, etc.) |
| تحديد ما إذا كانت المنطقة `no-map` | ✓ | |
| السياسة الداخلية للـ allocator | | ✓ |

---

### Subsystems ذات صلة يجب معرفتها

- **OF (Open Firmware / Device Tree):** النظام الذي يُوفِّر `struct device_node` وقراءة الـ FDT — هذا الـ framework يبني فوقه مباشرة.
- **CMA (Contiguous Memory Allocator):** أحد أبرز مستخدمي هذا الـ framework — يُسجِّل نفسه عبر `RESERVEDMEM_OF_DECLARE` ويُدير الـ contiguous allocations.
- **DMA Mapping:** الـ `device_init` في الغالب يُهيئ الـ DMA ops للـ device — يجب فهم `dma_map_ops` لفهم ما يحدث فيه.
- **memblock:** الـ early boot memory allocator الذي يُستخدم قبل تشغيل الـ buddy system لحجز المناطق.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Config Options والـ Macros

| الرمز | النوع | الأثر عند التفعيل | الأثر عند عدم التفعيل |
|---|---|---|---|
| `CONFIG_OF_RESERVED_MEM` | Kconfig | يُفعّل كل الـ API الحقيقي | كل الدوال تُرجع `‑ENOSYS` أو `NULL` |
| `RESERVEDMEM_OF_DECLARE` | Macro | يُسجّل handler في section خاص | يُسجّل stub فارغ عبر `_OF_DECLARE_STUB` |
| `CONFIG_OF_DYNAMIC` | Kconfig (من of.h) | يُضيف `_flags` لـ `struct property` | يُحذف الحقل |
| `CONFIG_OF_KOBJ` | Kconfig (من of.h) | يُضيف `kobject` و `bin_attribute` | يُحذف الحقلان |
| `CONFIG_OF_PROMTREE` | Kconfig (من of.h) | يُضيف `unique_id` لـ `struct property` | يُحذف الحقل |

---

### جدول الـ typedefs الأساسية

| الـ typedef | التعريف | الاستخدام |
|---|---|---|
| `reservedmem_of_init_fn` | `int (*)(struct reserved_mem *rmem)` | callback تُستدعى عند init منطقة محجوزة |
| `phys_addr_t` | `u32` أو `u64` حسب الـ arch | تُمثّل عنواناً فيزيائياً |
| `phandle` | `u32` | معرّف node في FDT |

---

### الـ Structs الأساسية

#### 1. `struct reserved_mem`

**الغرض**: يُمثّل منطقة ذاكرة محجوزة واحدة معرّفة في الـ Device Tree تحت `reserved-memory`.

```c
struct reserved_mem {
    const char              *name;      /* اسم المنطقة من FDT */
    unsigned long            fdt_node;  /* offset الـ node في FDT blob */
    const struct reserved_mem_ops *ops; /* vtable: init/release لكل device */
    phys_addr_t              base;      /* العنوان الفيزيائي البداية */
    phys_addr_t              size;      /* الحجم بالبايت */
    void                    *priv;      /* بيانات خاصة بالـ driver/subsystem */
};
```

| الحقل | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | مُستخرج من خاصية `label` أو اسم الـ node في FDT |
| `fdt_node` | `unsigned long` | offset داخل الـ FDT blob الخام، يُستخدم للبحث |
| `ops` | `const struct reserved_mem_ops *` | مؤشر على vtable يُوفّره الـ driver عند التسجيل |
| `base` | `phys_addr_t` | عنوان البداية الفيزيائي للمنطقة المحجوزة |
| `size` | `phys_addr_t` | الحجم بالبايت |
| `priv` | `void *` | حقل حر — الـ driver يضع فيه ما يشاء (CMA descriptor، heap pointer…) |

**الارتباطات**: يُشير إلى `reserved_mem_ops` (vtable) وتُشير إليه `struct device` بشكل غير مباشر عبر الـ DMA ops.

---

#### 2. `struct reserved_mem_ops`

**الغرض**: vtable تُعرّف كيف تربط منطقةً محجوزة بـ device معين وكيف تُحرّرها.

```c
struct reserved_mem_ops {
    /* يُربط المنطقة بـ device: يُعدّل DMA ops الخاصة به */
    int  (*device_init)   (struct reserved_mem *rmem, struct device *dev);
    /* يُلغي الربط عند انتهاء الاستخدام */
    void (*device_release)(struct reserved_mem *rmem, struct device *dev);
};
```

| الحقل | التوقيع | المسؤولية |
|---|---|---|
| `device_init` | `int(rmem, dev)` | تهيئة الـ DMA mapping للـ device من المنطقة المحجوزة |
| `device_release` | `void(rmem, dev)` | تحرير الموارد عند انتهاء استخدام الـ device |

**المثال الحقيقي**: subsystem الـ CMA يُسجّل ops تجعل `device_init` تُعيّن `dev->dma_mem` أو تُعدّل `dev->dma_range_map`.

---

#### 3. `struct device_node` (من `include/linux/of.h`)

**الغرض**: يُمثّل node واحداً في شجرة الـ Device Tree داخل الـ kernel.

```c
struct device_node {
    const char          *name;        /* اسم الـ node */
    phandle              phandle;     /* phandle الفريد */
    const char          *full_name;   /* المسار الكامل */
    struct fwnode_handle fwnode;      /* واجهة firmware موحّدة */
    struct property     *properties;  /* قائمة الخصائص */
    struct device_node  *parent;      /* الـ node الأب */
    struct device_node  *child;       /* أول ابن */
    struct device_node  *sibling;     /* الـ node المجاور */
    unsigned long        _flags;      /* flags داخلية */
    void                *data;        /* بيانات خاصة */
};
```

**الارتباط بالـ reserved_mem**: كل `reserved_mem` يُقابل node في `reserved-memory` ضمن FDT؛ الـ `fdt_node` يُشير إلى offset هذا الـ node.

---

#### 4. `struct of_phandle_args` (من `include/linux/of.h`)

**الغرض**: يحمل نتيجة parse لـ property تحتوي phandle + arguments (مثل `memory-region = <&cma_pool>`).

```c
struct of_phandle_args {
    struct device_node *np;                   /* الـ node المُشار إليه */
    int                 args_count;           /* عدد الـ arguments */
    uint32_t            args[MAX_PHANDLE_ARGS]; /* قيم الـ arguments */
};
```

**الاستخدام في السياق**: الـ API الداخلي لـ `of_reserved_mem_device_init_by_idx` يُنشئ `of_phandle_args` لقراءة `memory-region` property ثم يبحث عن `reserved_mem` المقابل.

---

### مخطط علاقات الـ Structs

```
  Device Tree (FDT blob)
  ┌─────────────────────────────┐
  │  /reserved-memory           │
  │    ├── linux,cma            │ ←── fdt_node offset
  │    └── framebuffer@3c000000 │ ←── fdt_node offset
  └─────────────┬───────────────┘
                │  parsed at boot
                ▼
  ┌─────────────────────────────────────────┐
  │         struct reserved_mem             │
  │  name ──► "linux,cma"                   │
  │  fdt_node = 0x1A0                       │
  │  base = 0x3c000000                      │
  │  size = 0x4000000                       │
  │  priv ──────────────────────────────┐  │
  │  ops ─────────────────────────┐     │  │
  └───────────────────────────────│─────│──┘
                                  │     │
                                  ▼     ▼
  ┌──────────────────────┐    ┌─────────────────────┐
  │  struct reserved_mem │    │  CMA descriptor /   │
  │  _ops                │    │  heap private data  │
  │  .device_init()  ────┼──► │  (subsystem-specific│
  │  .device_release()   │    │   priv data)        │
  └──────────────────────┘    └─────────────────────┘
                                  ▲
                                  │ DMA ops configured
  ┌───────────────────────┐       │
  │     struct device     │───────┘
  │  of_node ─────────────┼──► struct device_node
  │  dma_mem              │      │
  └───────────────────────┘      │ properties list
                                  ▼
                           ┌─────────────────┐
                           │  "memory-region"│
                           │  property       │
                           │  value = phandle│──► reserved_mem node
                           └─────────────────┘
```

---

### مخطط دورة حياة `reserved_mem`

```
BOOT TIME
─────────────────────────────────────────────────────────────
  FDT في الذاكرة
       │
       │  early_init_fdt_scan_reserved_mem()
       ▼
  يُمسح كل node تحت /reserved-memory
       │
       │  __reserved_mem_check_root()
       │  __fdt_scan_reserved_mem()
       ▼
  يُنشأ reserved_mem[] array (static, حجم ثابت)
  ┌──────────────────────────────────┐
  │  reserved_mem[0] = { name, base, │
  │    size, fdt_node, ops=NULL,     │
  │    priv=NULL }                   │
  └──────────────────────────────────┘
       │
       │  RESERVEDMEM_OF_DECLARE  ──► يُسجّل init callbacks
       │  في section __reservedmem_of_table
       │
       │  fdt_init_reserved_mem()
       │  يُطابق compat string مع الـ callbacks
       │  يستدعي reservedmem_of_init_fn(rmem)
       ▼
  الـ driver يُعيّن rmem->ops و rmem->priv
  (مثلاً: cma_init_reserved_mem يُسجّل cma_ops)

DEVICE PROBE TIME
─────────────────────────────────────────────────────────────
  driver probe
       │
       │  of_reserved_mem_device_init(dev)
       │    │
       │    └─► of_reserved_mem_device_init_by_idx(dev, np, 0)
       │              │
       │              │ يقرأ "memory-region" property
       │              │ يُحوّل phandle → reserved_mem
       │              │
       │              └─► rmem->ops->device_init(rmem, dev)
       ▼
  الـ device جاهز: DMA ops مُعيّنة، priv مُهيّأ

DEVICE REMOVE TIME
─────────────────────────────────────────────────────────────
  driver remove
       │
       │  of_reserved_mem_device_release(dev)
       │    └─► rmem->ops->device_release(rmem, dev)
       ▼
  الـ DMA ops تُحرَّر، priv قد يُحرَّر
  لكن struct reserved_mem نفسها تبقى (static حتى shutdown)
```

---

### مخطط تدفق الاستدعاء الكامل

#### مسار `of_reserved_mem_device_init_by_idx`

```
driver: of_reserved_mem_device_init(dev)
  │
  └─► of_reserved_mem_device_init_by_idx(dev, dev->of_node, idx=0)
        │
        ├─ of_parse_phandle_with_args(np, "memory-region",
        │      "#memory-region-cells", idx, &args)
        │      │
        │      └─► يُعيد args.np = device_node للمنطقة المحجوزة
        │
        ├─ of_reserved_mem_lookup(args.np)
        │      │
        │      └─► يبحث في reserved_mems[] عن تطابق fdt_node
        │          يُعيد struct reserved_mem *rmem
        │
        ├─ [تحقق: rmem != NULL و rmem->ops != NULL]
        │
        └─► rmem->ops->device_init(rmem, dev)
               │
               ├─► (CMA path) set_dma_ops(dev, &cma_dma_ops)
               │   dev->cma_area = rmem->priv
               │
               └─► (SRAM/custom path) ioremap, set dev->dma_mem ...
```

#### مسار `of_reserved_mem_device_init_by_name`

```
driver: of_reserved_mem_device_init_by_name(dev, np, "codec-mem")
  │
  ├─ of_property_match_string(np, "memory-region-names", "codec-mem")
  │    └─► يُعيد idx
  │
  └─► of_reserved_mem_device_init_by_idx(dev, np, idx)
        └─► [نفس المسار السابق]
```

#### مسار `RESERVEDMEM_OF_DECLARE` عند البوت

```
RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup)
  │
  │  يضع struct of_device_id في:
  │  section: __reservedmem_of_table
  │
  │  [عند boot]
  ▼
fdt_init_reserved_mem()
  │
  ├── for each reserved_mem[i]:
  │     ├── of_get_flat_dt_prop(node, "compatible", ...)
  │     ├── يُطابق مع كل entry في __reservedmem_of_table
  │     └── يستدعي entry->data(rmem)
  │              │
  │              └─► rmem_cma_setup(rmem)
  │                     ├── cma_init_reserved_mem(...)
  │                     └── rmem->ops = &cma_ops
  │                         rmem->priv = cma
  └── [انتهى]
```

---

### استراتيجية الـ Locking

هذا الـ header لا يُعرّف locks صريحة، لكن الـ locking يحدث في طبقات متعددة:

#### جدول الـ Locking

| المورد المحمي | الـ Lock المستخدم | الملاحظات |
|---|---|---|
| `reserved_mems[]` array (قراءة) | لا lock — read-only بعد البوت | يُكتب مرة واحدة خلال `early_init` قبل SMP |
| `rmem->ops` و `rmem->priv` | لا lock — يُكتبان مرة واحدة في `device_init` | atomic بطبيعة probe serialization |
| `dev->dma_mem` | `dev->mutex` (device core lock) | الـ device core يحمي probe/remove |
| `struct device` أثناء init/release | `device_lock(dev)` | الـ driver core يضمن serialization |

#### قواعد الـ Lock Ordering (من الأعلى للأسفل)

```
1. device_lock(dev)            ← الأعلى
2. [internal subsystem lock]   ← مثلاً CMA lock
3. [hardware register access]  ← الأدنى
```

> **تحذير**: لا يجوز استدعاء `of_reserved_mem_device_init` من سياق atomic (interrupt/softirq) لأن الـ implementation الداخلية قد تستدعي `kmalloc(GFP_KERNEL)`.

#### تسلسل Probe-safe

```
kernel thread (probe)
  │
  ├─ device_lock(dev)          ← الـ device core يحمل هذا
  │
  ├─ of_reserved_mem_device_init(dev)   ← آمن هنا
  │     └─► rmem->ops->device_init(rmem, dev)
  │               └─► قد يُخصّص ذاكرة (GFP_KERNEL)
  │
  └─ device_unlock(dev)
```

---

### ملخص العلاقات الكاملة

```
                    ┌─────────────────────────────┐
                    │     FDT / Device Tree        │
                    │  /reserved-memory { ... }    │
                    └──────────────┬──────────────┘
                                   │ boot scan
                    ┌──────────────▼──────────────┐
                    │    reserved_mems[] (static)  │
                    │  ┌────────────────────────┐  │
                    │  │   struct reserved_mem  │  │
                    │  │   .name                │  │
                    │  │   .fdt_node            │  │
                    │  │   .base / .size        │  │
                    │  │   .ops ──────────────┐ │  │
                    │  │   .priv ───────────┐ │ │  │
                    │  └────────────────────│─│─┘  │
                    └───────────────────────│─│────┘
                                            │ │
              ┌─────────────────────────────┘ │
              ▼                               ▼
  ┌──────────────────────┐      ┌─────────────────────────┐
  │  struct reserved_    │      │  subsystem private data  │
  │  mem_ops             │      │  (CMA area, SRAM heap,   │
  │  .device_init()      │      │   ION heap, ...)         │
  │  .device_release()   │      └─────────────────────────┘
  └──────────┬───────────┘
             │ called with
             ▼
  ┌──────────────────────┐
  │    struct device     │
  │  .of_node ──────────►│struct device_node
  │  .dma_mem            │  .properties ──► "memory-region"
  │  .dma_ops            │                    │
  └──────────────────────┘                    │ phandle
                                              ▼
                                    struct reserved_mem (lookup)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Structs والـ Types الأساسية

| الاسم | النوع | الوصف |
|---|---|---|
| `struct reserved_mem` | struct | يمثّل منطقة ذاكرة محجوزة مُعرَّفة في FDT |
| `struct reserved_mem_ops` | struct | جدول الـ operations الخاص بكل منطقة محجوزة |
| `reservedmem_of_init_fn` | typedef | نوع الدالة المُسجَّلة لتهيئة منطقة محجوزة |

#### الـ Functions والـ Macros — نظرة شاملة

| الدالة / الـ Macro | الفئة | الوصف المختصر |
|---|---|---|
| `RESERVEDMEM_OF_DECLARE` | Registration | تسجيل handler لـ compatible string في reserved-memory |
| `of_reserved_mem_device_init` | Runtime | ربط أول منطقة محجوزة بالـ device (index 0) |
| `of_reserved_mem_device_init_by_idx` | Runtime | ربط منطقة محجوزة بالـ device عبر index |
| `of_reserved_mem_device_init_by_name` | Runtime | ربط منطقة محجوزة بالـ device عبر الاسم |
| `of_reserved_mem_device_release` | Cleanup | تحرير الـ reserved memory المرتبطة بـ device |
| `of_reserved_mem_lookup` | Helper | إيجاد `struct reserved_mem` من `device_node` |
| `of_reserved_mem_region_to_resource` | Helper | تحويل منطقة محجوزة (by index) إلى `struct resource` |
| `of_reserved_mem_region_to_resource_byname` | Helper | تحويل منطقة محجوزة (by name) إلى `struct resource` |
| `of_reserved_mem_region_count` | Helper | إرجاع عدد مناطق `memory-region` في node معيّن |

---

### تصنيف الـ Functions

---

### Registration — التسجيل المبكر في Boot

هذه الفئة تتعامل مع تسجيل handlers لمناطق الذاكرة المحجوزة قبل أن يبدأ الـ kernel في تهيئة الـ devices. يحدث ذلك خلال مرحلة `early_init_dt_scan_reserved_mem` حيث يُمسح الـ FDT ويُربط كل node يحمل `compatible` مُسجَّلًا بـ init function خاصة به.

---

#### `RESERVEDMEM_OF_DECLARE`

```c
#define RESERVEDMEM_OF_DECLARE(name, compat, init) \
    _OF_DECLARE(reservedmem, name, compat, init, reservedmem_of_init_fn)
```

**ما تفعله:** يُسجّل `init` function مرتبطة بـ `compatible` string معيّنة في الـ section الخاص بـ reserved memory (`__reservedmem_of_table`). عند boot، يمرّ الـ kernel على هذا الـ table ويستدعي الـ init function لكل node في FDT يطابق الـ compatible المُسجَّل.

**المعاملات:**
- `name` — رمز فريد (identifier) يُستخدم كاسم الـ entry في الـ linker table، لا يجب أن يتكرر.
- `compat` — الـ compatible string المطابقة لما في FDT (مثل `"shared-dma-pool"`).
- `init` — مؤشر دالة من نوع `reservedmem_of_init_fn`، تُستدعى عند اكتشاف node مطابق.

**تفاصيل جوهرية:**
- يعتمد على `_OF_DECLARE` الذي يضع الـ entry في section خاص داخل الـ kernel image يُقرأ في وقت مبكر جدًا.
- عند تعطيل `CONFIG_OF_RESERVED_MEM`، يُستبدل بـ `_OF_DECLARE_STUB` الذي يُبقي الـ entry لمنع تحذيرات الـ compiler لكن لا يُسجّل شيئًا فعليًا.
- **الـ init function** تستقبل `struct reserved_mem *` وتقوم عادةً بتهيئة CMA أو heap أو أي allocator خاص.

**مثال واقعي:**
```c
/* تسجيل handler لـ shared-dma-pool (CMA) */
static int rmem_cma_setup(struct reserved_mem *rmem)
{
    /* تهيئة CMA area من base/size */
    return cma_init_reserved_mem(rmem->base, rmem->size, 0, rmem->name, &cma);
}
RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup);
```

---

### Runtime — ربط الـ Device بالذاكرة المحجوزة

هذه الفئة تُنفَّذ خلال الـ device probe، وتربط الـ device بمنطقة ذاكرة محجوزة وتُعيّن الـ DMA ops المناسبة.

---

#### `of_reserved_mem_device_init`

```c
static inline int of_reserved_mem_device_init(struct device *dev);
```

**ما تفعله:** wrapper مُبسَّط يُحوِّل إلى `of_reserved_mem_device_init_by_idx` بـ `idx = 0`. يَقرأ أول عنصر في خاصية `memory-region` من `dev->of_node` ويربط الـ device بها.

**المعاملات:**
- `dev` — الـ device المراد تهيئته، يُستخدم `dev->of_node` لقراءة الـ DT property.

**القيمة المُرجَعة:** `0` عند النجاح، أو كود خطأ سلبي (`-ENODEV`, `-EINVAL`, ...).

**من يستدعيها:** driver في `probe()` عندما يكون الـ device يملك property واحدة `memory-region`.

**ملاحظة:** إذا كان الـ device يملك أكثر من منطقة، استخدم `_by_idx` أو `_by_name`.

---

#### `of_reserved_mem_device_init_by_idx`

```c
int of_reserved_mem_device_init_by_idx(struct device *dev,
                                        struct device_node *np,
                                        int idx);
```

**ما تفعله:** تُحلّل خاصية `memory-region` في الـ `np` node وتستخرج الـ phandle عند الـ `idx` المحدد، ثم تبحث عن `struct reserved_mem` المقابل وتستدعي `ops->device_init(rmem, dev)` لتُعيّن الـ DMA ops على الـ device.

**المعاملات:**
- `dev` — الـ device الهدف الذي ستُعيَّن عليه الـ DMA ops.
- `np` — الـ device_node الذي يحتوي على `memory-region` property (غالبًا `dev->of_node`).
- `idx` — الفهرس (0-based) داخل مصفوفة الـ phandles في `memory-region`.

**القيمة المُرجَعة:** `0` عند النجاح. أخطاء محتملة:
- `-ENODEV` — لم يُعثر على node أو لا يوجد `reserved_mem` مقابل.
- `-EINVAL` — الـ idx خارج النطاق أو الـ node غير صحيح.
- أي كود خطأ تُرجعه `ops->device_init`.

**تفاصيل جوهرية:**
- تستدعي `of_parse_phandle` لاسترداد الـ phandle من `memory-region[idx]`.
- تستدعي `of_reserved_mem_lookup` للبحث عن الـ `reserved_mem` المقابل.
- إذا لم تكن `ops` أو `ops->device_init` مُعيَّنة → ترجع `-ENODEV`.
- لا تحتاج lock صريح لأن الـ reserved_mem table تُبنى في boot ولا تتغير لاحقًا.

**Pseudocode:**
```
of_reserved_mem_device_init_by_idx(dev, np, idx):
    target_node = of_parse_phandle(np, "memory-region", idx)
    if !target_node → return -ENODEV

    rmem = of_reserved_mem_lookup(target_node)
    if !rmem → return -ENODEV

    if !rmem->ops || !rmem->ops->device_init → return -ENODEV

    return rmem->ops->device_init(rmem, dev)
```

---

#### `of_reserved_mem_device_init_by_name`

```c
int of_reserved_mem_device_init_by_name(struct device *dev,
                                         struct device_node *np,
                                         const char *name);
```

**ما تفعله:** تُحوِّل الـ `name` إلى index عبر `of_property_match_string` على خاصية `memory-region-names`، ثم تُحيل إلى `of_reserved_mem_device_init_by_idx`.

**المعاملات:**
- `dev` — الـ device الهدف.
- `np` — الـ node المحتوي على `memory-region` و`memory-region-names`.
- `name` — الاسم المطلوب كما يظهر في `memory-region-names` في DTS.

**القيمة المُرجَعة:** نفس `of_reserved_mem_device_init_by_idx`. تُرجع `-ENODEV` إضافةً إذا لم يُعثر على الاسم.

**تفاصيل جوهرية:**
- تعتمد على أن `memory-region-names` و`memory-region` مُتزامنتان في الـ index (العنصر الأول من `names` يقابل العنصر الأول من `memory-region`).

**مثال واقعي:**
```c
/* في DTS */
memory-region = <&fb_region>, <&codec_region>;
memory-region-names = "framebuffer", "codec";

/* في driver */
of_reserved_mem_device_init_by_name(dev, dev->of_node, "codec");
```

---

### Cleanup — تحرير الموارد

---

#### `of_reserved_mem_device_release`

```c
void of_reserved_mem_device_release(struct device *dev);
```

**ما تفعله:** تُلغي ربط الـ device بالذاكرة المحجوزة. تبحث عن `reserved_mem` المرتبط بالـ device وتستدعي `ops->device_release(rmem, dev)` لتنظيف الـ DMA ops وأي موارد خصّصتها `device_init`.

**المعاملات:**
- `dev` — الـ device المراد تحريره، يجب أن يكون قد مرّ بـ `device_init` مسبقًا.

**القيمة المُرجَعة:** `void`.

**تفاصيل جوهرية:**
- إذا لم تكن `ops->device_release` مُعيَّنة، لا تفعل شيئًا.
- يجب استدعاؤها في `remove()` أو `shutdown()` callback للـ driver مقابل كل `device_init` تمّت.
- عدم الاستدعاء قد يُبقي الـ DMA ops مُعيَّنة على device يُزال، مما يُسبب use-after-free.

**من يستدعيها:** driver في `remove()` أو `platform_driver.remove`.

---

### Helpers — البحث والاستعلام

هذه الفئة توفر واجهات للبحث والاستعلام عن معلومات مناطق الذاكرة المحجوزة دون الحاجة لتعديل الـ device state.

---

#### `of_reserved_mem_lookup`

```c
struct reserved_mem *of_reserved_mem_lookup(struct device_node *np);
```

**ما تفعله:** تبحث في الـ global table الخاص بـ reserved memory regions عن المنطقة التي تقابل الـ `fdt_node` الخاص بالـ `np`، وتُرجع مؤشرًا إليها.

**المعاملات:**
- `np` — الـ device_node الخاص بالـ reserved-memory node (وليس الـ consumer device node).

**القيمة المُرجَعة:**
- مؤشر صالح إلى `struct reserved_mem` عند النجاح.
- `NULL` إذا لم تُعثر على منطقة مقابلة.

**تفاصيل جوهرية:**
- الـ table (`reserved_mem[]`) تُبنى خلال `early_init_dt_scan_reserved_mem` في الـ boot المبكر وتبقى ثابتة بعد ذلك — لا يحتاج الاستدعاء لحماية بـ lock.
- المقارنة تتم بالـ `fdt_node` offset وليس بالاسم، مما يجعلها فريدة وسريعة.

**من يستدعيها:** داخليًا من `of_reserved_mem_device_init_by_idx`، وكذلك drivers المتقدمة التي تحتاج الوصول المباشر لـ `base`/`size`/`priv`.

---

#### `of_reserved_mem_region_to_resource`

```c
int of_reserved_mem_region_to_resource(const struct device_node *np,
                                        unsigned int idx,
                                        struct resource *res);
```

**ما تفعله:** تستخرج المنطقة المحجوزة عند الـ `idx` المحدد في خاصية `memory-region` للـ `np` node، وتُحوِّل معلومات `base` و`size` إلى `struct resource` من نوع `IORESOURCE_MEM`.

**المعاملات:**
- `np` — الـ device node المحتوي على `memory-region` property.
- `idx` — فهرس المنطقة المطلوبة (0-based).
- `res` — مؤشر إلى `struct resource` يُملأ بالنتيجة (`start`, `end`, `flags`).

**القيمة المُرجَعة:** `0` عند النجاح، كود خطأ سلبي عند الفشل.

**تفاصيل جوهرية:**
- مفيدة لـ drivers التي تحتاج تسجيل الذاكرة المحجوزة كـ `IORESOURCE_MEM` (مثلًا لإدارتها عبر `request_mem_region`).
- تُرجع `-ENOSYS` عند تعطيل `CONFIG_OF_RESERVED_MEM`.

---

#### `of_reserved_mem_region_to_resource_byname`

```c
int of_reserved_mem_region_to_resource_byname(const struct device_node *np,
                                               const char *name,
                                               struct resource *res);
```

**ما تفعله:** نفس `of_reserved_mem_region_to_resource` لكن تستخدم الاسم بدلًا من الـ index. تبحث في `memory-region-names` عن `name` لتحديد الـ index ثم تستدعي منطق التحويل.

**المعاملات:**
- `np` — الـ device node.
- `name` — الاسم كما في `memory-region-names`.
- `res` — مؤشر إلى `struct resource` يُملأ.

**القيمة المُرجَعة:** `0` عند النجاح، كود خطأ سلبي عند الفشل أو عدم وجود الاسم.

---

#### `of_reserved_mem_region_count`

```c
int of_reserved_mem_region_count(const struct device_node *np);
```

**ما تفعله:** تُرجع عدد المناطق المحجوزة المُعرَّفة في خاصية `memory-region` للـ node المُعطى.

**المعاملات:**
- `np` — الـ device node المحتوي على `memory-region`.

**القيمة المُرجَعة:**
- عدد صحيح موجب يمثّل عدد الـ phandles في `memory-region`.
- `0` عند تعطيل `CONFIG_OF_RESERVED_MEM` أو عدم وجود الـ property.

**تفاصيل جوهرية:**
- مفيدة لـ drivers التي تحتاج تهيئة حلقة `for` على جميع مناطق الـ device دون hardcoding عددها.

**مثال واقعي:**
```c
/* تهيئة كل مناطق الذاكرة المحجوزة للـ device */
int count = of_reserved_mem_region_count(dev->of_node);
for (int i = 0; i < count; i++)
    of_reserved_mem_device_init_by_idx(dev, dev->of_node, i);
```

---

### الـ Structs — شرح تفصيلي

---

#### `struct reserved_mem`

```c
struct reserved_mem {
    const char              *name;      /* اسم الـ node من FDT */
    unsigned long            fdt_node;  /* offset الـ node في FDT الخام */
    const struct reserved_mem_ops *ops; /* جدول الـ operations */
    phys_addr_t              base;      /* عنوان البداية الفيزيائي */
    phys_addr_t              size;      /* الحجم بالبايت */
    void                    *priv;      /* بيانات خاصة بالـ allocator (CMA, heap, ...) */
};
```

**الـ `priv`:** يُستخدم من قِبل الـ handler لتخزين حالة خاصة به. مثلًا في CMA يُخزَّن فيه مؤشر `struct cma *`، وفي الـ heap المخصص يُخزَّن فيه مؤشر الـ heap object.

---

#### `struct reserved_mem_ops`

```c
struct reserved_mem_ops {
    int  (*device_init)   (struct reserved_mem *rmem, struct device *dev);
    void (*device_release)(struct reserved_mem *rmem, struct device *dev);
};
```

**`device_init`:** تُستدعى عند ربط device بالمنطقة. تُعيّن الـ DMA ops وتُخصص موارد per-device إذا لزم.

**`device_release`:** تُستدعى عند فصل الـ device. يجب أن تُعكس فيها كل ما فعلته `device_init`.

**ملاحظة:** كلا الـ callbacks اختياريان — إذا كانت `NULL`، يُعامَل الأمر كـ no-op أو يُرجع `-ENODEV`.

---

#### `reservedmem_of_init_fn`

```c
typedef int (*reservedmem_of_init_fn)(struct reserved_mem *rmem);
```

نوع الدالة التي تُسجَّل عبر `RESERVEDMEM_OF_DECLARE`. تُستدعى مرة واحدة في الـ boot المبكر لكل node في `reserved-memory` يطابق الـ compatible المُسجَّل. تُهيّئ الـ allocator وتُعيّن `rmem->ops` و`rmem->priv`.

---

### تدفق العمل الكامل — Flow Diagram

```
Boot مبكر:
  FDT scan
    └──> RESERVEDMEM_OF_DECLARE table
           └──> reservedmem_of_init_fn(rmem)
                  └──> يُهيّئ rmem->ops, rmem->priv

Driver probe():
  of_reserved_mem_device_init(dev)
    └──> of_reserved_mem_device_init_by_idx(dev, np, 0)
           ├──> of_parse_phandle(np, "memory-region", idx)
           ├──> of_reserved_mem_lookup(target_node)
           └──> rmem->ops->device_init(rmem, dev)
                  └──> يُعيّن DMA ops على dev

Driver remove():
  of_reserved_mem_device_release(dev)
    └──> rmem->ops->device_release(rmem, dev)
           └──> يُلغي DMA ops ويُحرر الموارد
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs ذات الصلة

الـ `reserved_mem` subsystem لا يُنشئ مدخلات debugfs بشكل مباشر، لكن يمكن الوصول إلى معلومات الـ memory regions عبر:

```bash
# قراءة شجرة الـ Device Tree المُحمَّلة في الذاكرة
ls /sys/kernel/debug/of_reserved_mem/ 2>/dev/null || echo "Not available — check CONFIG_DEBUG_FS"

# الـ DMA contiguous allocator (CMA) — غالباً مرتبط بـ reserved_mem
ls /sys/kernel/debug/cma/
cat /sys/kernel/debug/cma/cma-0/used
cat /sys/kernel/debug/cma/cma-0/bitmap
```

**تفسير المخرجات:**
- `used` → عدد الـ pages المُستخدمة حالياً من الـ CMA region
- `bitmap` → خريطة bit لكل page: `1` = مشغولة، `0` = حرة

```bash
# تفاصيل الـ memory map الكاملة بما فيها reserved regions
cat /sys/kernel/debug/memblock/reserved
# المخرج: عنوان البداية، الحجم، الـ flags لكل region
```

---

#### 2. مدخلات الـ sysfs ذات الصلة

```bash
# التحقق من ربط reserved_mem بجهاز معين
ls /sys/bus/platform/devices/<device_name>/
cat /sys/bus/platform/devices/<device_name>/of_node/memory-region

# الـ DMA mask والـ coherent memory للجهاز
cat /sys/bus/platform/devices/<device_name>/dma_mask
cat /sys/bus/platform/devices/<device_name>/coherent_dma_mask

# قراءة كل عقد الـ DT من sysfs
ls /sys/firmware/devicetree/base/reserved-memory/
# كل مجلد = region واحد من الـ Device Tree
cat /sys/firmware/devicetree/base/reserved-memory/<region>/reg
# المخرج: bytes خام → استخدم xxd لفكّه
xxd /sys/firmware/devicetree/base/reserved-memory/<region>/reg
```

**مثال حقيقي:**
```bash
xxd /sys/firmware/devicetree/base/reserved-memory/framebuffer@3e000000/reg
# 00000000: 0000 0000 3e00 0000 0000 0000 0200 0000
#           ^^^^^^^^^^^^^^^^^^^ base=0x3e000000
#                               ^^^^^^^^^^^^^^^^^^^ size=0x200000 (2MB)
```

---

#### 3. استخدام الـ ftrace — Tracepoints/Events

```bash
# تفعيل أحداث الـ memory allocation في reserved regions
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/trace

# تتبع دوال of_reserved_mem عبر function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo "of_reserved_mem*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الجهاز أو أعد تحميل الـ driver
cat /sys/kernel/debug/tracing/trace

# تتبع أحداث الـ DMA (مرتبطة بـ reserved_mem عبر DMA ops)
echo 1 > /sys/kernel/debug/tracing/events/dma_fence/enable
echo 1 > /sys/kernel/debug/tracing/events/kmem/enable

# تتبع الـ CMA allocations
echo 1 > /sys/kernel/debug/tracing/events/cma/cma_alloc_start/enable
echo 1 > /sys/kernel/debug/tracing/events/cma/cma_alloc_finish/enable
echo 1 > /sys/kernel/debug/tracing/events/cma/cma_release/enable
cat /sys/kernel/debug/tracing/trace
```

**مثال مخرج الـ ftrace:**
```
          <...>-1234  [000] .... 12345.678901: of_reserved_mem_device_init_by_idx <-platform_drv_probe
          <...>-1234  [000] .... 12345.678950: cma_alloc_start: name=linux,cma pfn=0x3e000 align=0x200 count=512
```

---

#### 4. تفعيل الـ printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لـ of_reserved_mem subsystem
# الملفات المصدرية: drivers/of/of_reserved_mem.c

echo "file drivers/of/of_reserved_mem.c +p" > /sys/kernel/debug/dynamic_debug/control
# أو تفعيل كل الـ OF subsystem
echo "module of +p" > /sys/kernel/debug/dynamic_debug/control

# التحقق مما تم تفعيله
grep "of_reserved_mem" /sys/kernel/debug/dynamic_debug/control

# رفع مستوى الـ loglevel لرؤية الرسائل
dmesg -n 8        # KERN_DEBUG
# أو في /proc
echo 8 > /proc/sys/kernel/printk

# مراقبة الرسائل في الوقت الحقيقي
dmesg -w | grep -i "reserved\|rmem\|cma\|of_reserved"
```

**في kernel command line (مفيد عند boot):**
```bash
# أضف للـ bootargs في U-Boot أو grub
earlycon loglevel=8 ignore_loglevel
# أو تفعيل dynamic debug مبكراً
dyndbg="file drivers/of/of_reserved_mem.c +p"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الوظيفة | متى تُفعّله |
|---|---|---|
| `CONFIG_OF_RESERVED_MEM` | يُفعّل الـ subsystem أصلاً | دائماً في embedded |
| `CONFIG_DEBUG_FS` | يتيح `/sys/kernel/debug` | أثناء التطوير |
| `CONFIG_CMA_DEBUGFS` | مدخلات debugfs للـ CMA | تتبع CMA allocations |
| `CONFIG_CMA_DEBUG` | printk تفصيلي لكل CMA op | تشخيص فشل alloc |
| `CONFIG_DMA_API_DEBUG` | يتحقق من صحة DMA ops | كشف misuse الـ DMA |
| `CONFIG_OF_UNITTEST` | اختبارات وحدة الـ OF | التحقق من صحة parse |
| `CONFIG_MEMBLOCK_DEBUG` | debug الـ memblock allocator | مشاكل boot-time |
| `CONFIG_DEBUG_MEMORY_INIT` | تلوين الذاكرة عند init | كشف use-before-init |
| `CONFIG_KASAN` | كشف memory corruption | إذا اشتُبه بـ overflow |
| `CONFIG_SLUB_DEBUG` | debug الـ slab allocator | مشاكل alloc/free |

```bash
# التحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(OF_RESERVED|CMA|DMA_API|MEMBLOCK)_DEBUG"
# أو
grep -E "CONFIG_(OF_RESERVED|CMA|DMA_API|MEMBLOCK)_DEBUG" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# أداة dtc — فكّ تشفير الـ DTB الحالي للتحقق من reserved-memory
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 20 "reserved-memory"

# fdtdump — بديل
apt install device-tree-compiler
fdtdump /boot/dtb-$(uname -r) | grep -A 30 "reserved-memory"

# قراءة /proc/iomem لرؤية reserved regions المسجّلة
cat /proc/iomem | grep -i "reserved\|cma\|framebuffer"

# مثال مخرج:
# 3e000000-3fffffff : reserved
# 40000000-bfffffff : System RAM

# قراءة /proc/meminfo للـ CMA stats
cat /proc/meminfo | grep -i cma
# CmaTotal:        65536 kB
# CmaFree:         61440 kB

# تتبع allocations الـ DMA في الـ device
cat /sys/kernel/debug/dma-api/dump 2>/dev/null

# أداة iomem لمقارنة الـ physical map
cat /proc/iomem
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `Reserved memory: failed to init node <name>` | فشل `init_fn` الخاص بالـ region | تحقق من تنفيذ `RESERVEDMEM_OF_DECLARE` وتطابق الـ compatible string |
| `Reserved memory: missing "size" property` | خاصية `size` غائبة في DT node | أضف `size = <0x...>;` للـ reserved-memory node |
| `Reserved memory: unsupported region` | الـ compatible غير مدعوم أو الـ init_fn غير مسجّلة | تحقق من تحميل الـ driver المناسب |
| `of_reserved_mem_device_init: failed, ret=-22` | EINVAL — الـ device node لا يحتوي `memory-region` | أضف `memory-region = <&rmem_node>;` في DT للجهاز |
| `of_reserved_mem_device_init: failed, ret=-2` | ENOENT — الـ phandle لا يشير لـ node موجود | تحقق من صحة الـ phandle في DT |
| `cma: Failed to reserve X MiB` | الذاكرة المطلوبة غير متاحة عند boot | قلّل الحجم أو عدّل `mem=` في bootargs |
| `DMA-API: device tries to free DMA memory it has not allocated` | الـ driver حرّر ذاكرة لم يخصصها | راجع الـ driver، فعّل `CONFIG_DMA_API_DEBUG` |
| `Reserved mem: failed to allocate memory for node` | مساحة الـ memblock نفدت | تحقق من ترتيب الـ regions وعدم التداخل |
| `ERROR: reservedmem: reg property not found` | `reg` غائبة والـ region ليست dynamic | أضف `reg = <base size>;` أو فعّل dynamic allocation |
| `of_reserved_mem_lookup: node has no reg` | الـ node غير مكتمل | أضف خاصية `reg` أو استخدم `size` + `alignment` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

**في `drivers/of/of_reserved_mem.c` — نقاط مقترحة:**

```c
/* في دالة __reserved_mem_init_node — تحقق من نجاح init */
int __reserved_mem_init_node(struct reserved_mem *rmem)
{
    /* ... */
    ret = initfn(rmem);
    /* نقطة مهمة: تحقق من فشل التهيئة */
    if (ret && ret != -ENODEV) {
        WARN_ON(1); /* سيطبع stack trace مع الـ context */
        pr_err("Reserved memory: init failed for %s, ret=%d\n",
               rmem->name, ret);
    }
    return ret;
}

/* في of_reserved_mem_device_init_by_idx — تحقق من القيود */
int of_reserved_mem_device_init_by_idx(struct device *dev,
                                        struct device_node *np, int idx)
{
    struct reserved_mem *rmem;
    /* ... */
    /* تحقق: يجب ألا تكون rmem->ops فارغة إن وجدت الـ region */
    if (rmem && !rmem->ops) {
        WARN(1, "reserved_mem %s has no ops, device %s may malfunction\n",
             rmem->name, dev_name(dev));
    }

    /* تحقق من التوافق بين الـ DMA mask والـ region */
    if (rmem && (rmem->base + rmem->size) > dma_get_mask(dev)) {
        dev_warn(dev, "reserved_mem region [%pa + %pa] exceeds DMA mask\n",
                 &rmem->base, &rmem->size);
        dump_stack(); /* لمعرفة من طلب الـ init */
    }
}
```

---

### Hardware Level

---

#### 1. التحقق من توافق حالة الـ Hardware مع حالة الـ Kernel

```bash
# 1. قراءة reserved regions من kernel
cat /proc/iomem | grep -i reserved

# 2. مقارنة مع ما يقوله الـ bootloader (U-Boot مثلاً)
# في U-Boot console:
# bdinfo          → يُظهر memory layout
# fdt print /reserved-memory   → يُظهر DT reserved regions

# 3. التحقق من أن الـ kernel لم يكتب على reserved regions
# قراءة محتوى الـ region (إذا كانت non-cacheable)
devmem2 0x3e000000 w   # اقرأ أول word
# إذا كانت القيمة 0xdeadbeef → الـ region لم تُلمس (initialized by bootloader)
```

**تشخيص التعارض بين الـ kernel والـ hardware:**
```
+-------------------+         +-------------------+
|  Kernel iomem     |   vs.   |  Hardware Reality |
|  /proc/iomem      |         |  (DTB / SoC spec) |
+-------------------+         +-------------------+
| 0x3e000000 reserved|  <-->  | Frame buffer      |
| 0x40000000 RAM     |  <-->  | SDRAM bank 0      |
+-------------------+         +-------------------+
```

---

#### 2. تقنيات قراءة الـ Registers (Register Dump)

```bash
# devmem2 — قراءة/كتابة physical memory
apt install devmem2
# اقرأ 4 bytes من بداية الـ reserved region
devmem2 0x3e000000 w
# اقرأ 4 bytes في العنوان 0x3e000004
devmem2 0x3e000004 w

# /dev/mem — قراءة عبر dd
dd if=/dev/mem of=/tmp/reserved_dump.bin bs=4096 count=1 skip=$((0x3e000000/4096))
xxd /tmp/reserved_dump.bin | head -20

# busybox devmem (embedded systems)
busybox devmem 0x3e000000

# io utility (في بعض distros)
io -4 0x3e000000   # read 32-bit
io -4 -r 0x3e000000 0x100  # read 256 bytes
```

**قراءة الـ MMIO registers لـ memory controller (مثال على RK3399):**
```bash
# قراءة DRAM controller base
devmem2 0xffa80000 w   # DDRC0 base
# المخرج: Value at address 0xFFA80000 (0x...) : 0x00000001
# bit[0]=1 → controller active
```

---

#### 3. نصائح استخدام Logic Analyzer / Oscilloscope

**حالات الاستخدام مع reserved_mem:**

| حالة | القناة | ما تبحث عنه |
|---|---|---|
| تحقق من استخدام الـ DMA region | قناة DRAM data bus | traffic أثناء DMA transfers |
| مراقبة الـ CMA allocation | GPIO debug pin (set in driver) | pulse عند `cma_alloc` / `cma_release` |
| تعارض الـ cache | قناة AXI bus | read-after-write conflicts |
| timing الـ device init | GPIO pin في `device_init` callback | تأكيد ترتيب التهيئة |

**تقنية GPIO bitbang للتوقيت:**
```c
/* في driver's device_init callback */
#include <linux/gpio.h>

static int my_rmem_device_init(struct reserved_mem *rmem, struct device *dev)
{
    /* Toggle GPIO للإشارة ببدء init — مرئي على oscilloscope */
    gpio_set_value(DEBUG_GPIO, 1);

    /* ... الكود الفعلي ... */

    gpio_set_value(DEBUG_GPIO, 0);
    return 0;
}
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة الـ Hardware | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| عنوان الـ reserved region خاطئ في DT | `Reserved memory: failed to reserve memory for node` | قارن `reg` في DT مع SoC memory map |
| تداخل الـ regions (overlap) | `Reserved memory: region overlaps with previous one` | راجع DT، تأكد من عدم تداخل `[base, base+size]` |
| الـ RAM أصغر من المُعلَن | `Failed to reserve X MiB at offset Y` | تحقق من حجم الـ DRAM الفعلي عبر `bdinfo` |
| الـ cache غير مُعطَّل لـ region | DMA data corruption بدون log واضح | تحقق من `no-map` property في DT |
| عنوان غير محاذٍ (alignment) | `cma: Failed to reserve memory` | الـ base يجب أن يكون محاذياً لـ `PAGE_SIZE` أو أكثر |
| الـ IOMMU يحجب الـ DMA | `iommu fault` أو `dma_map_sg failed` | تحقق من `iommu-map` في DT أو أضف `dma-ranges` |

---

#### 5. تشخيص الـ Device Tree (التحقق من مطابقة DT للـ Hardware)

```bash
# 1. استخراج DTB الفعلي المُستخدَم من الـ kernel
cp /sys/firmware/fdt /tmp/current.dtb
dtc -I dtb -O dts /tmp/current.dtb > /tmp/current.dts
grep -A 30 "reserved-memory" /tmp/current.dts

# 2. مقارنة مع DTB المصدر
dtc -I dtb -O dts /boot/dtb-$(uname -r) > /tmp/boot.dts
diff /tmp/current.dts /tmp/boot.dts

# 3. التحقق من صحة phandles
grep -n "memory-region" /tmp/current.dts     # في device nodes
grep -n "linux,cma\|framebuffer\|rmem" /tmp/current.dts   # في reserved-memory

# مثال DT صحيح:
# reserved-memory {
#     #address-cells = <1>;
#     #size-cells = <1>;
#     ranges;
#
#     cma_pool: linux,cma@3e000000 {
#         compatible = "shared-dma-pool";
#         reusable;
#         reg = <0x3e000000 0x2000000>;  /* 32MB */
#         linux,cma-default;
#     };
# };
#
# my_device@ff000000 {
#     memory-region = <&cma_pool>;  /* phandle يشير للـ region */
# };
```

**فحص `no-map` property — حاسم للـ DMA coherency:**
```bash
grep -A 5 "no-map" /tmp/current.dts
# وجود no-map = الـ region محمية من الـ kernel VA mapping
# غياب no-map على CMA = الـ region قابلة للاستخدام من الـ kernel أيضاً
```

---

### Practical Commands

---

#### 1. أوامر جاهزة للنسخ

**فحص شامل لحالة الـ reserved_mem:**
```bash
#!/bin/bash
# reserved_mem_debug.sh — فحص شامل

echo "=== /proc/iomem (reserved regions) ==="
cat /proc/iomem | grep -i "reserved\|cma\|frame"

echo ""
echo "=== /proc/meminfo (CMA stats) ==="
cat /proc/meminfo | grep -i cma

echo ""
echo "=== Device Tree reserved-memory nodes ==="
if [ -d /sys/firmware/devicetree/base/reserved-memory ]; then
    for node in /sys/firmware/devicetree/base/reserved-memory/*/; do
        name=$(basename "$node")
        echo "--- $name ---"
        # اقرأ compatible
        [ -f "$node/compatible" ] && echo "  compatible: $(strings "$node/compatible")"
        # اقرأ reg (base + size)
        [ -f "$node/reg" ] && echo "  reg (hex): $(xxd "$node/reg" | head -2)"
        # تحقق من no-map
        [ -f "$node/no-map" ] && echo "  no-map: YES (not mapped in kernel VA)"
    done
else
    echo "DT sysfs not available"
fi

echo ""
echo "=== CMA debugfs ==="
for cma_dir in /sys/kernel/debug/cma/*/; do
    [ -d "$cma_dir" ] || continue
    name=$(basename "$cma_dir")
    used=$(cat "$cma_dir/used" 2>/dev/null)
    count=$(cat "$cma_dir/count" 2>/dev/null)
    echo "CMA $name: used=$used / total=$count pages"
done

echo ""
echo "=== Dynamic debug — enable of_reserved_mem ==="
echo "Run: echo 'file drivers/of/of_reserved_mem.c +p' > /sys/kernel/debug/dynamic_debug/control"
```

```bash
chmod +x reserved_mem_debug.sh && ./reserved_mem_debug.sh
```

---

**تفعيل ftrace لتتبع دورة حياة كاملة:**
```bash
#!/bin/bash
# ftrace_rmem.sh

TRACEFS=/sys/kernel/debug/tracing

# إيقاف وتنظيف
echo 0 > $TRACEFS/tracing_on
echo > $TRACEFS/trace

# اختيار الـ tracer
echo function > $TRACEFS/current_tracer

# فلترة دوال of_reserved_mem فقط
echo "of_reserved_mem*" > $TRACEFS/set_ftrace_filter

# تفعيل أحداث الـ CMA أيضاً
echo 1 > $TRACEFS/events/cma/enable 2>/dev/null

# بدء التتبع
echo 1 > $TRACEFS/tracing_on
echo "Tracing started — trigger device probe now..."

# انتظر 10 ثواني أو حتى الـ probe
sleep 10

echo 0 > $TRACEFS/tracing_on
echo ""
echo "=== Trace Output ==="
cat $TRACEFS/trace | grep -v "^#" | head -50

# تنظيف
echo nop > $TRACEFS/current_tracer
echo > $TRACEFS/set_ftrace_filter
```

---

**تشخيص فشل الـ init بالتفصيل:**
```bash
# تفعيل كل debug messages للـ OF subsystem
echo 8 > /proc/sys/kernel/printk
echo "file drivers/of/of_reserved_mem.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/of/platform.c +p" > /sys/kernel/debug/dynamic_debug/control

# إعادة تحميل الـ driver لرؤية الـ init sequence
modprobe -r my_driver
dmesg -c > /dev/null   # مسح الـ dmesg
modprobe my_driver
dmesg | grep -E "reserved|rmem|cma|of_node|memory.region"
```

---

#### 2. مثال مخرج حقيقي وتفسيره

**مخرج `/proc/iomem` على board مع CMA:**
```
00000000-0fffffff : System RAM       ← 256MB RAM
  00000000-00ffffff : Kernel code
  01000000-017fffff : Kernel data
3c000000-3dffffff : reserved          ← 32MB: reserved للـ GPU firmware
3e000000-3fffffff : reserved          ← 32MB: CMA pool
  3e000000-3fffffff : linux,cma       ← مُسمَّى في DT
40000000-bfffffff : System RAM       ← 2GB إضافية
```

**التفسير:**
- الـ CMA pool يبدأ عند `0x3e000000` بحجم `0x2000000` (32MB)
- الـ GPU firmware reserved يسبقه مباشرة — أي driver يطلب DMA من هذه المنطقة سيحصل من الـ CMA

**مخرج `dmesg` أثناء boot ناجح:**
```
[    0.000000] Reserved memory: creating CMA memory pool at 0x000000003e000000, size 32 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    2.345678] of_reserved_mem_device_init_by_idx: assigned reserved memory node linux,cma
[    2.345690] dma-direct: reserve mem DMA ops set for device my-device.0
```

**مخرج `dmesg` عند فشل:**
```
[    0.000000] Reserved memory: failed to reserve memory for node framebuffer@3e000000: base 0x3e000000, size 32 MiB
[    0.000001] Reserved memory: region 3e000000..3fffffff overlaps with previous one
```

**التشخيص:** الـ region `0x3e000000-0x3fffffff` محجوزة مسبقاً بواسطة node آخر — راجع الـ DT للتعارضات.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بوابة صناعية على RK3562 — فشل تهيئة CAN بسبب `memory-region` مفقود

#### العنوان
**الـ CAN controller لا يعمل في بوابة IoT صناعية على RK3562**

#### السياق
شركة تبني بوابة صناعية (Industrial Gateway) تعمل على **Rockchip RK3562**. الجهاز يستخدم **CAN bus** للتواصل مع أجهزة الاستشعار في خط الإنتاج. المنتج في مرحلة الاختبار النهائي قبل الشحن.

#### المشكلة
عند تشغيل الجهاز، يفشل الـ CAN driver أثناء الـ probe بالخطأ:

```
rockchip-canfd fe570000.can: Failed to get reserved memory(-19)
rockchip-canfd fe570000.can: probe with driver rockchip-canfd failed with error -19
```

الكود `-19` يعني `ENODEV`، لكن المهندس يعرف أن الـ hardware موجود فعلاً.

#### التحليل
الـ driver يستدعي في دالة الـ probe:

```c
/* driver calls this to associate DMA buffer from reserved memory */
ret = of_reserved_mem_device_init(dev);
if (ret) {
    dev_err(dev, "Failed to get reserved memory(%d)\n", ret);
    return ret;
}
```

**الـ `of_reserved_mem_device_init`** هي wrapper تستدعي:

```c
static inline int of_reserved_mem_device_init(struct device *dev)
{
    /* uses index 0 — first memory-region entry */
    return of_reserved_mem_device_init_by_idx(dev, dev->of_node, 0);
}
```

الدالة `of_reserved_mem_device_init_by_idx` تبحث في **`memory-region`** property داخل device tree node الخاص بالـ CAN. إذا لم تجد هذه الـ property أو لم يشر الـ phandle إلى `reserved-memory` node صحيح — ترجع `-ENODEV`.

فحص الـ DTS الحالي:

```dts
/* WRONG: missing memory-region property */
&can0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&can0_pins>;
};
```

الـ `reserved-memory` node موجود في الـ DTS لكن لم يُربط بالـ CAN node.

#### الحل
إضافة `memory-region` property في DTS، مع إنشاء node مناسب في `reserved-memory`:

```dts
/ {
    reserved-memory {
        #address-cells = <2>;
        #size-cells = <2>;
        ranges;

        /* Reserve 256KB for CAN DMA buffers */
        can_reserved: can@10000000 {
            reg = <0x0 0x10000000 0x0 0x40000>;
            no-map;
        };
    };
};

&can0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&can0_pins>;
    /* Link device to reserved memory region */
    memory-region = <&can_reserved>;
};
```

التحقق بعد الإصلاح:

```bash
# verify the phandle resolution
cat /proc/device-tree/can@fe570000/memory-region
# check reserved memory map
cat /proc/iomem | grep "reserved"
# confirm DMA pool initialized
dmesg | grep -i "can\|reserved"
```

#### الدرس المستفاد
**الـ `of_reserved_mem_device_init`** تفشل صامتةً بـ `-ENODEV` إذا لم يكن `memory-region` موجوداً في الـ DTS. كثير من الـ drivers تعتمد على هذا الـ call بصمت، لذا يجب دائماً التحقق من الـ DTS node قبل إلقاء اللوم على الـ hardware.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تعارض في الـ reserved memory بين الـ GPU والـ VPU

#### العنوان
**تعطل الفيديو 4K مع رسائل `dma_alloc` فشل على Allwinner H616**

#### السياق
منتج **Android TV Box** يعتمد على **Allwinner H616**. يشتكي العملاء من أن تشغيل فيديو **4K H.265** يتسبب في تجمد الشاشة أو خروج الـ VPU driver عن الخدمة. الـ GPU و الـ VPU يعملان في نفس الوقت.

#### المشكلة
رسائل الـ kernel:

```
sunxi-cedrus 1c0e000.video-codec: dma_alloc_coherent failed for size 33554432
sunxi-cedrus 1c0e000.video-codec: Failed to allocate codec buffers
mali ff300000.gpu: [MALI] Memory region overlap detected
```

#### التحليل
كلا الـ drivers يستدعيان `of_reserved_mem_device_init`:

```c
/* VPU driver probe */
ret = of_reserved_mem_device_init(dev);  /* idx=0 */

/* GPU driver probe */
ret = of_reserved_mem_device_init(dev);  /* idx=0 */
```

فحص الـ DTS الخاطئ:

```dts
reserved-memory {
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    /* PROBLEM: both GPU and VPU point to same region! */
    media_reserved: media@42000000 {
        reg = <0x42000000 0x8000000>; /* 128MB */
        no-map;
    };
};

&vpu {
    memory-region = <&media_reserved>;
};

&gpu {
    memory-region = <&media_reserved>;  /* WRONG: shares same region */
};
```

عند تحليل `of_reserved_mem_lookup`:

```c
/* This finds the reserved_mem struct by node pointer */
struct reserved_mem *of_reserved_mem_lookup(struct device_node *np)
```

كلا الـ devices تحصل على نفس `struct reserved_mem`، ما يتسبب في تضارب `priv` pointer:

```c
struct reserved_mem {
    const char          *name;
    unsigned long        fdt_node;
    const struct reserved_mem_ops *ops;
    phys_addr_t          base;   /* same base for both! */
    phys_addr_t          size;
    void                *priv;   /* overwritten by second device_init */
};
```

عندما يستدعي الـ GPU الـ `device_init` على نفس الـ `rmem`، يُعيد كتابة `rmem->priv`، مما يُفسد الـ VPU DMA pool.

#### الحل
فصل الـ regions في الـ DTS:

```dts
reserved-memory {
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    /* Dedicated 64MB for VPU codec buffers */
    vpu_reserved: vpu@42000000 {
        reg = <0x42000000 0x4000000>;
        no-map;
    };

    /* Dedicated 64MB for GPU memory pool */
    gpu_reserved: gpu@46000000 {
        reg = <0x46000000 0x4000000>;
        no-map;
    };
};

&vpu {
    memory-region = <&vpu_reserved>;
};

&gpu {
    memory-region = <&gpu_reserved>;
};
```

التحقق:

```bash
# List all reserved memory regions with their owners
cat /sys/kernel/debug/memblock/reserved
# Verify no overlap
cat /proc/iomem | grep -A2 "reserved"
```

#### الدرس المستفاد
**الـ `struct reserved_mem`** ليست مُصممة للمشاركة بين أكثر من device. الـ `priv` pointer يُعاد كتابته في كل `device_init` call. يجب دائماً تخصيص region مستقل لكل device تحتاج DMA reserved memory.

---

### السيناريو 3: STM32MP1 — عدم تحرير الـ reserved memory عند إزالة الـ module

#### العنوان
**تسرب ذاكرة عند إعادة تحميل الـ SPI DMA driver على STM32MP1**

#### السياق
جهاز **IoT sensor hub** يعمل على **STMicroelectronics STM32MP1**. الـ SPI controller يستخدم DMA مع reserved memory لتبادل البيانات مع حساسات ضغط عالية السرعة. المهندس يختبر تحديث الـ firmware بإزالة الـ module وإعادة تحميله (`rmmod` / `insmod`).

#### المشكلة
بعد عدة دورات من `rmmod`/`insmod`، يبدأ الـ kernel في رفض الـ DMA allocations:

```
spi-stm32 44009000.spi: Unable to allocate DMA buffer
spi-stm32 44009000.spi: probe with driver spi-stm32 failed with error -12
```

الخطأ `-12` هو `ENOMEM`.

#### التحليل
الـ driver يحتوي على خطأ في الـ `remove` function:

```c
/* driver remove — BUGGY version */
static int spi_stm32_remove(struct platform_device *pdev)
{
    struct spi_stm32 *spi = platform_get_drvdata(pdev);

    spi_unregister_master(spi->master);
    clk_disable_unprepare(spi->clk);
    /* BUG: missing of_reserved_mem_device_release! */
    return 0;
}
```

بينما الـ header يُعرّف:

```c
/* Must be called to release device association with reserved memory */
void of_reserved_mem_device_release(struct device *dev);
```

هذه الدالة تُلغي ربط الـ device بالـ `reserved_mem` region وتحرر الـ CMA/DMA pool. بدون استدعائها، يظل الـ `rmem->priv` يشير إلى الـ pool القديم، وعند الـ `insmod` التالي يفشل `device_init_by_idx` في إعادة تهيئة الـ pool لأن الـ node ما زال "مرتبطاً".

فحص الـ `struct reserved_mem_ops`:

```c
struct reserved_mem_ops {
    int  (*device_init)(struct reserved_mem *rmem, struct device *dev);
    /* This must be called on driver remove */
    void (*device_release)(struct reserved_mem *rmem, struct device *dev);
};
```

#### الحل
إضافة استدعاء `of_reserved_mem_device_release` في دالة الـ `remove`:

```c
/* Fixed driver remove */
static int spi_stm32_remove(struct platform_device *pdev)
{
    struct spi_stm32 *spi = platform_get_drvdata(pdev);

    spi_unregister_master(spi->master);
    clk_disable_unprepare(spi->clk);

    /* Release reserved memory association — symmetric with probe */
    of_reserved_mem_device_release(&pdev->dev);

    return 0;
}
```

التحقق:

```bash
# Before fix — watch refcount leak
cat /sys/kernel/debug/memblock/reserved | grep spi

# Test rmmod/insmod cycle
for i in $(seq 1 10); do
    rmmod spi_stm32
    insmod spi_stm32.ko
    dmesg | tail -3
done
```

#### الدرس المستفاد
**الـ `of_reserved_mem_device_init` و `of_reserved_mem_device_release`** يجب أن تكونا متناظرتين تماماً مثل `probe`/`remove`. غيابها يُسبب تسرباً تراكمياً لا يظهر إلا بعد دورات إعادة تحميل متعددة — سيناريو شائع في أنظمة الـ OTA update الصناعية.

---

### السيناريو 4: NXP i.MX8M Plus — فشل تشغيل الـ ISP camera بسبب ترتيب `memory-region` خاطئ

#### العنوان
**كاميرا ISP لا تعمل على لوحة مخصصة مبنية على i.MX8M Plus**

#### السياق
مشروع **bring-up** للوحة مخصصة تعمل على **NXP i.MX8M Plus** لتطبيق رؤية حاسوبية في خط التجميع الآلي. الـ ISP (Image Signal Processor) يحتاج منطقتين من الـ reserved memory: واحدة لـ firmware loading وأخرى لـ DMA frame buffers.

#### المشكلة
الـ ISP driver يُسجّل:

```
imx8-isi 32e00000.isi: reserved memory region 'isp-fw' not found
imx8-isi 32e00000.isi: Failed to load ISP firmware
```

لكن المهندس يتحقق من الـ DTS ويجد كلا الـ regions موجودتين.

#### التحليل
الـ driver يستخدم الـ API الجديد بالاسم:

```c
/* Request firmware region by name */
ret = of_reserved_mem_device_init_by_name(dev, dev->of_node, "isp-fw");
if (ret) {
    dev_err(dev, "reserved memory region 'isp-fw' not found\n");
    return ret;
}

/* Request frame buffer region by index */
ret = of_reserved_mem_device_init_by_idx(dev, dev->of_node, 1);
```

فحص الـ DTS:

```dts
/* WRONG ORDER: names don't match property order */
&isi {
    memory-region = <&isp_dma_buf>, <&isp_fw_mem>;
    memory-region-names = "frame-buf", "isp-fw";
    /* index 0 = isp_dma_buf labeled "frame-buf"  */
    /* index 1 = isp_fw_mem  labeled "isp-fw"     */
};
```

الـ `of_reserved_mem_device_init_by_name` تبحث في **`memory-region-names`** property لتجد الفهرس المقابل لـ `"isp-fw"`, تجده عند الفهرس `1`, ثم تستدعي `of_reserved_mem_device_init_by_idx(dev, np, 1)` التي تُحلّل الـ phandle رقم `1` في `memory-region`.

الـ `reserved_mem` المُسترجع هو `isp_fw_mem`، الذي حُجز بـ `no-map` ولا يمكن استخدامه لـ DMA مباشرة من الـ CPU — مما يتسبب في فشل الـ firmware copy.

#### الحل
مطابقة ترتيب الـ phandles مع أسماء الـ regions:

```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* ISP firmware load area — must be CPU-accessible */
    isp_fw_mem: isp-fw@c0000000 {
        reg = <0x0 0xc0000000 0x0 0x1000000>; /* 16MB */
        /* no 'no-map' — CPU needs to copy firmware here */
    };

    /* ISP DMA frame buffers */
    isp_dma_buf: isp-dma@c1000000 {
        reg = <0x0 0xc1000000 0x0 0x4000000>; /* 64MB */
        no-map;
    };
};

&isi {
    /* Order matches memory-region-names */
    memory-region = <&isp_fw_mem>, <&isp_dma_buf>;
    memory-region-names = "isp-fw", "frame-buf";
};
```

التحقق باستخدام الـ API المتاح:

```bash
# Count expected regions
cat /sys/firmware/devicetree/base/isi@32e00000/memory-region-names
# Verify lookup works
cat /sys/kernel/debug/of_reserved_mem
```

يمكن أيضاً استخدام `of_reserved_mem_region_count` للتحقق البرمجي:

```c
/* Sanity check: ensure at least 2 regions are declared */
int count = of_reserved_mem_region_count(dev->of_node);
if (count < 2) {
    dev_err(dev, "Need 2 memory regions, got %d\n", count);
    return -EINVAL;
}
```

#### الدرس المستفاد
**الـ `of_reserved_mem_device_init_by_name`** تعتمد على ترتيب الـ phandles في `memory-region` property لتطابق الأسماء في `memory-region-names`. الترتيب حرج — أي اختلاف يُعطي الـ driver منطقة ذاكرة خاطئة دون رسالة خطأ واضحة. دائماً تحقق من الترتيب في الـ DTS.

---

### السيناريو 5: TI AM62x — تشخيص مشكلة `RESERVEDMEM_OF_DECLARE` في driver مخصص لـ DSP

#### العنوان
**الـ DSP لا يحصل على reserved memory رغم وجود `compatible` صحيح على AM62x**

#### السياق
تطوير **ECU (Engine Control Unit)** سيارة يعمل على **Texas Instruments AM62x**. المهندس كتب driver مخصصاً لتحميل firmware على الـ DSP core (R5F) مع تخصيص reserved memory خاصة به. الـ bring-up الأولي فاشل.

#### المشكلة
رسائل الـ boot:

```
Reserved memory: failed to init node ti-dsp-fw
ti-dsp-fw: compatible 'ti,am62-dsp-pool' not found in reserved_mem table
remoteproc remoteproc0: Failed to get memory pool
```

#### التحليل
المهندس كتب الكود التالي:

```c
/* Custom reserved memory initializer for DSP firmware pool */
static int ti_dsp_pool_init(struct reserved_mem *rmem)
{
    pr_info("DSP reserved memory: base=%pa size=%pa\n",
            &rmem->base, &rmem->size);
    return 0;
}

/* WRONG: missing the RESERVEDMEM_OF_DECLARE macro */
/* The driver never registered this initializer! */
```

**الـ `RESERVEDMEM_OF_DECLARE`** macro المُعرّف في الـ header هو:

```c
#define RESERVEDMEM_OF_DECLARE(name, compat, init)      \
    _OF_DECLARE(reservedmem, name, compat, init, reservedmem_of_init_fn)
```

هذا الـ macro يضع الـ initializer في section خاص (`__reservedmem_of_table`) يقرأه الـ kernel أثناء الـ boot لمطابقة الـ `compatible` strings في `reserved-memory` nodes.

بدون هذا الـ macro، لا تُسجَّل الـ `compatible` string `"ti,am62-dsp-pool"` في الجدول، وعندما يحاول الـ kernel تهيئة الـ node يُسجّل الخطأ المذكور.

الـ DTS كان صحيحاً:

```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    dsp_fw_pool: dsp-fw@9c000000 {
        /* compatible must match RESERVEDMEM_OF_DECLARE */
        compatible = "ti,am62-dsp-pool";
        reg = <0x0 0x9c000000 0x0 0x2000000>; /* 32MB */
        no-map;
    };
};

&dsp0 {
    memory-region = <&dsp_fw_pool>;
};
```

#### الحل
إضافة الـ macro في نهاية ملف الـ driver:

```c
static int ti_dsp_pool_init(struct reserved_mem *rmem)
{
    /* Store base/size for later use in device_init callback */
    rmem->priv = (void *)(uintptr_t)rmem->base;

    pr_info("DSP reserved memory registered: base=%pa size=%pa\n",
            &rmem->base, &rmem->size);
    return 0;
}

/* Register initializer — links "ti,am62-dsp-pool" to ti_dsp_pool_init */
RESERVEDMEM_OF_DECLARE(ti_dsp_pool, "ti,am62-dsp-pool", ti_dsp_pool_init);
```

وتطبيق الـ `reserved_mem_ops` لتكتمل الدورة:

```c
static int ti_dsp_device_init(struct reserved_mem *rmem, struct device *dev)
{
    /* Called when driver does of_reserved_mem_device_init() */
    return dma_declare_coherent_memory(dev, rmem->base,
                                       rmem->base, rmem->size);
}

static void ti_dsp_device_release(struct reserved_mem *rmem, struct device *dev)
{
    dma_release_coherent_memory(dev);
}

static const struct reserved_mem_ops ti_dsp_ops = {
    .device_init    = ti_dsp_device_init,
    .device_release = ti_dsp_device_release,
};

static int ti_dsp_pool_init(struct reserved_mem *rmem)
{
    rmem->ops = &ti_dsp_ops;
    pr_info("DSP pool at %pa (%pa bytes)\n", &rmem->base, &rmem->size);
    return 0;
}

RESERVEDMEM_OF_DECLARE(ti_dsp_pool, "ti,am62-dsp-pool", ti_dsp_pool_init);
```

التحقق:

```bash
# Verify the compatible was registered and matched
dmesg | grep -i "dsp\|reserved"

# Check the reserved memory table entries
cat /sys/kernel/debug/of_reserved_mem

# Confirm resource mapping
of_reserved_mem_region_to_resource — use in driver to get struct resource
```

يمكن أيضاً استخدام `of_reserved_mem_region_to_resource` لتحويل المنطقة إلى `struct resource` معيارية:

```c
struct resource res;
/* Convert reserved memory region 0 to kernel resource */
ret = of_reserved_mem_region_to_resource(dev->of_node, 0, &res);
if (ret == 0)
    pr_info("DSP memory: %pR\n", &res);
```

#### الدرس المستفاد
**الـ `RESERVEDMEM_OF_DECLARE`** هو نقطة الدخول الإلزامية لأي driver يريد التعامل مع `reserved-memory` node بـ `compatible` مخصص. بدونها، الـ kernel لا يُهيئ الـ node ولا يستدعي الـ callbacks. الـ macro يربط الـ DTS compatible بالـ init function في compile time عبر section خاص في binary الـ kernel.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الأهمية |
|--------|---------|
| [A deep dive into CMA](https://lwn.net/Articles/486301/) | شرح معمّق لـ CMA وعلاقته بـ reserved memory |
| [CMA & device tree, once again](https://lwn.net/Articles/605348/) | كيف دُمج CMA مع device tree وتهيئة `reserved-memory` |
| [Contiguous memory allocation for drivers](https://lwn.net/Articles/396702/) | المقال الأصلي الذي اقترح CMA كبديل للحجز الثابت |
| [Contiguous Memory Allocator](https://lwn.net/Articles/461849/) | شرح الـ API الخاص بـ CMA وتفاعله مع الـ driver |
| [mm/memblock: Add "reserve_mem" to reserved named memory at boot up](https://lwn.net/Articles/977976/) | إضافة حجز الذاكرة المسماة وقت الإقلاع عبر `memblock` |
| [Improving support for large, contiguous allocations](https://lwn.net/Articles/753167/) | تحسينات على CMA لدعم تخصيصات الذاكرة الكبيرة |
| [Boot time memory management](https://static.lwn.net/kerneldoc/core-api/boot-time-mm.html) | توثيق `memblock` وإدارة الذاكرة وقت الإقلاع |

---

### التوثيق الرسمي في kernel source

**الـ** device tree binding الرئيسي:

```
Documentation/devicetree/bindings/reserved-memory/reserved-memory.yaml
Documentation/devicetree/bindings/reserved-memory/reserved-memory.txt
```

روابط kernel.org:

- [reserved-memory.yaml](https://www.kernel.org/doc/Documentation/devicetree/bindings/reserved-memory/reserved-memory.yaml)
- [reserved-memory.txt](https://www.kernel.org/doc/Documentation/devicetree/bindings/reserved-memory/reserved-memory.txt)
- [فهرس مجلد reserved-memory](https://www.kernel.org/doc/Documentation/devicetree/bindings/reserved-memory/)

**ملفات الـ source المرتبطة مباشرةً:**

```
include/linux/of_reserved_mem.h       ← الـ header الذي نوثّقه
drivers/of/of_reserved_mem.c          ← التنفيذ الكامل للـ API
mm/cma.c                              ← تنفيذ CMA
include/linux/cma.h                   ← واجهة CMA
kernel/dma/contiguous.c               ← ربط DMA بـ CMA
```

---

### Patch series أصلية (مشاريع وتعديلات جوهرية)

| الرابط | الوصف |
|--------|-------|
| [PATCH v5 00/11: reserved-memory regions/CMA](https://lore.kernel.org/lkml/20140301202048.39FB9C40B6D@trevor.secretlab.ca/T/) | السلسلة الأصلية التي أدخلت `of_reserved_mem` إلى الـ kernel |
| [PATCH v5 06/11: drivers: of: initialize and assign reserved memory](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1392985527-6260-7-git-send-email-m.szyprowski@samsung.com/) | الـ patch الذي أضاف `of_reserved_mem_device_init()` |
| [PATCH v5: Device Tree support for CMA](https://lwn.net/Articles/563167/) | دعم CMA عبر device tree |

---

### توثيق خارجي موثوق

| الموقع | الرابط | المحتوى |
|--------|--------|---------|
| elinux.org | [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | مرجع شامل لـ device tree بما في ذلك `reserved-memory` |
| elinux.org | [Memory Management](https://elinux.org/Memory_Management) | إدارة الذاكرة في Linux المدمج |
| elinux.org | [Device Tree Information](https://www.elinux.org/Device_Tree_Information) | معلومات عامة عن device tree في Linux |
| AMD/Xilinx Wiki | [Linux Reserved Memory](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841683/Linux+Reserved+Memory) | مثال عملي على استخدام reserved memory في SoC حقيقي |
| kernelnewbies.org | [KernelMemoryAllocation](https://kernelnewbies.org/KernelMemoryAllocation) | شرح مبسّط لأساليب تخصيص الذاكرة في الـ kernel |
| kernelnewbies.org | [Linux_6.1](https://kernelnewbies.org/Linux_6.1) | تغييرات الذاكرة في إصدار 6.1 |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 15**: Memory Mapping and DMA
  - يشرح `dma_alloc_coherent()` وكيف يرتبط بالـ reserved memory
  - متاح مجاناً: https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 12**: Memory Management
  - يغطي `bootmem`، zones، وتخصيص الذاكرة المادية
  - مهم لفهم لماذا يُحجز بعض الـ RAM قبل بدء الـ buddy allocator

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 7**: Bootloaders
- **الفصل 14**: Kernel Debugging Techniques
  - يشرح device tree في السياق المدمج وكيفية تعريف `reserved-memory`

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 3**: Memory Management
  - تفاصيل `memblock`، zones، وإدارة الذاكرة المادية

---

### مصادر الـ mailing list

- **LKML Archive**: https://lore.kernel.org/lkml/
  - ابحث بـ: `of_reserved_mem` أو `reserved-memory binding`
- **linux-mm mailing list**: https://lore.kernel.org/linux-mm/
  - نقاشات إدارة الذاكرة المتعلقة بـ CMA و reserved regions
- **devicetree@vger**: https://lore.kernel.org/devicetree/
  - نقاشات binding الـ `reserved-memory` وتغييراته

---

### مصطلحات البحث الموصى بها

للبحث في الـ kernel source أو الإنترنت:

```
of_reserved_mem_device_init
RESERVEDMEM_OF_DECLARE
reserved-memory device tree binding
CMA shared-dma-pool
memory-region property device tree
fdt_reserved_mem
of_reserved_mem_lookup
CONFIG_OF_RESERVED_MEM
memblock_reserve linux kernel
dma_declare_contiguous
```

للبحث في lore.kernel.org:

```bash
# البحث عن نقاشات of_reserved_mem
https://lore.kernel.org/search/?q=of_reserved_mem&l=all

# البحث عن patch series الأصلية
https://lore.kernel.org/search/?q=reserved-memory+binding+marek+szyprowski
```

---

### مخطط تسلسل المراجع

```
Reserved Memory في Linux
├── المفهوم الأساسي
│   ├── LDD3 — Chapter 15 (DMA)
│   └── Linux Kernel Development — Chapter 12
│
├── Device Tree Binding
│   ├── Documentation/devicetree/bindings/reserved-memory/
│   ├── elinux.org/Device_Tree_Reference
│   └── devicetree-spec (chapter 3)
│
├── التنفيذ في الـ Kernel
│   ├── drivers/of/of_reserved_mem.c
│   ├── LWN: CMA & device tree (lwn.net/Articles/605348)
│   └── Patch series (lore.kernel.org — 2014)
│
└── أمثلة عملية
    ├── Xilinx/AMD Wiki
    └── elinux.org/Memory_Management
```
## Phase 8: Writing simple module

### الهدف

سنقوم بـ hook الدالة `of_reserved_mem_device_init_by_idx` باستخدام **kprobe**، وهي الدالة المسؤولة عن ربط جهاز ما بمنطقة ذاكرة محجوزة (reserved memory region) معرّفة في Device Tree. هذه الدالة تُستدعى في وقت التشغيل عند تهيئة الأجهزة التي تحتاج إلى ذاكرة DMA مخصصة، مما يجعلها نقطة مثيرة للمراقبة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe hook on of_reserved_mem_device_init_by_idx
 * Logs: device name, device_node name, and region index
 * whenever a device claims a reserved memory region.
 */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/device.h>       /* struct device */
#include <linux/of.h>           /* struct device_node */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on of_reserved_mem_device_init_by_idx");

/*
 * pre_handler is called just before the probed function executes.
 * Signature of the hooked function:
 *   int of_reserved_mem_device_init_by_idx(struct device *dev,
 *                                           struct device_node *np,
 *                                           int idx);
 * On x86_64: rdi=dev, rsi=np, rdx=idx
 * On arm64:  x0=dev,  x1=np,  x2=idx
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#if defined(CONFIG_X86_64)
    struct device      *dev = (struct device *)regs->di;
    struct device_node *np  = (struct device_node *)regs->si;
    int                 idx = (int)regs->dx;
#elif defined(CONFIG_ARM64)
    struct device      *dev = (struct device *)regs->regs[0];
    struct device_node *np  = (struct device_node *)regs->regs[1];
    int                 idx = (int)regs->regs[2];
#else
    /* Architecture not explicitly handled — skip safe dereference */
    pr_info("rmem_probe: called (arch args not decoded)\n");
    return 0;
#endif

    /* Guard against NULL pointers before dereferencing */
    pr_info("rmem_probe: dev=%s  node=%s  idx=%d\n",
            dev  && dev_name(dev)  ? dev_name(dev)  : "<null>",
            np   && np->name       ? np->name        : "<null>",
            idx);

    return 0; /* 0 = continue execution normally */
}

/* post_handler fires after the probed instruction completes */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* Nothing needed here; kept to show the hook point exists */
}

/* Define the kprobe struct and set the target symbol */
static struct kprobe kp = {
    .symbol_name = "of_reserved_mem_device_init_by_idx",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init rmem_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("rmem_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("rmem_probe: planted at %p\n", kp.addr);
    return 0;
}

static void __exit rmem_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("rmem_probe: removed\n");
}

module_init(rmem_probe_init);
module_exit(rmem_probe_exit);
```

---

### Makefile

```makefile
obj-m += rmem_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod rmem_probe.ko
sudo dmesg | tail -20          # مراقبة المخرجات
sudo rmmod rmem_probe
```

---

### شرح كل جزء

#### `#include <linux/kprobes.h>`
يوفر واجهة الـ **kprobe** التي تتيح زرع نقطة مراقبة (breakpoint) على أي رمز kernel قابل للـ probe دون تعديل الكود الأصلي.

#### `struct kprobe kp`
**الـ kprobe struct** يحمل اسم الرمز المستهدف والمعالجات (handlers)؛ الـ kernel يحوّل الاسم إلى عنوان فعلي تلقائياً عند التسجيل.

#### `handler_pre` — الـ pre-handler
يُستدعى قبل تنفيذ أول تعليمة في الدالة المستهدفة، لذا تكون المعاملات (arguments) لا تزال موجودة في السجلات — نقرأها مباشرةً من `pt_regs` مع مراعاة الفرق بين x86_64 وARM64 في ترتيب السجلات.

#### `dev_name(dev)` و `np->name`
نطبع اسم الجهاز (مثل `soc@0:dma-controller`) واسم الـ device_node (مثل `reserved-memory`) والـ index لمعرفة أي منطقة محجوزة يطلبها الجهاز بالضبط.

#### `return 0` من `handler_pre`
القيمة `0` تعني "تابع التنفيذ الطبيعي"؛ أي قيمة أخرى تُلغي تنفيذ الدالة الأصلية وهذا خطير في حالتنا لأنه قد يُعطّل تهيئة الذاكرة.

#### `register_kprobe` / `unregister_kprobe`
التسجيل يزرع الـ breakpoint في الذاكرة، وإلغاء التسجيل في `module_exit` ضروري لاستعادة التعليمة الأصلية قبل إزالة الـ module — تركه مسجلاً بعد إزالة الكود يُسبب **kernel panic**.

---

### مثال على مخرجات `dmesg`

```
[  42.318450] rmem_probe: planted at ffffffffc08a1230
[  45.771203] rmem_probe: dev=fe300000.gpu  node=linux,cma  idx=0
[  45.772014] rmem_probe: dev=fc000000.video-codec  node=reserved-memory  idx=0
[  48.003120] rmem_probe: removed
```

**الـ** `fe300000.gpu` يطلب منطقة CMA الأولى (idx=0) المُعرّفة في Device Tree node `linux,cma` — وهو سلوك شائع على لوحات ARM مثل Raspberry Pi وRockchip.
