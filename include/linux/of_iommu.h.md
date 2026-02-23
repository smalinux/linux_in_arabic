## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ IOMMU؟

تخيّل أن لديك مدينة كبيرة (الذاكرة RAM)، وفيها أحياء سكنية (عناوين الذاكرة). كل جهاز في الحاسوب (GPU، بطاقة شبكة، وحدة صوت) يريد الوصول إلى هذه المدينة ليقرأ ويكتب البيانات. لو تركت كل جهاز يتجوّل بحرية في المدينة كلها، فأي جهاز مخترق أو معطوب قد يُفسد بيانات نظام التشغيل أو يسرق كلمات مرور المستخدمين.

**الـ IOMMU** (Input-Output Memory Management Unit) هو "الشرطي" الذي يجلس بين الأجهزة والذاكرة. كل جهاز لا يرى العناوين الحقيقية للذاكرة — بل يرى عناوين وهمية خاصة به تسمى **IOVA** (I/O Virtual Address). الـ IOMMU يترجم هذه العناوين الوهمية إلى الحقيقية، ويرفض أي وصول غير مُصرَّح به.

---

### أين تأتي مشكلة الـ Device Tree؟

في عالم الـ x86 (Intel/AMD)، كل جهاز يُعرّف نفسه عبر الـ PCI بـ ID واضح، فيعرف الـ IOMMU من هو. لكن في عالم الـ ARM (هواتف ذكية، راوترات، أجهزة مدمجة)، لا يوجد PCI في الغالب — الأجهزة مدمجة مباشرة في الـ SoC ولا تُعرّف نفسها تلقائياً. لذلك يُستخدم **Device Tree (DT)** — ملف وصف ثابت يخبر الكيرنل: "هذا الجهاز موجود، وهذا عنوانه، وهو متصل بهذا الـ IOMMU بهذا الـ ID."

---

### ما هو دور `of_iommu.h`؟

**الـ `of_iommu.h`** هو الـ header الذي يُعلن عن جسر الربط بين عالم الـ **OF (Open Firmware / Device Tree)** وعالم الـ **IOMMU subsystem** في الكيرنل.

القصة بالتفصيل:

1. عند إقلاع النظام، يقرأ الكيرنل الـ Device Tree ويكتشف الأجهزة.
2. لكل جهاز، يحتاج الكيرنل أن يعرف: هل هذا الجهاز متصل بـ IOMMU؟ وما هو الـ ID الذي يستخدمه للتعريف عن نفسه لهذا الـ IOMMU؟
3. `of_iommu_configure()` هي الدالة المسؤولة عن قراءة خاصية `iommus` أو `iommu-map` من الـ Device Tree لهذا الجهاز، ثم إخبار الـ IOMMU driver بأن يُسجّل هذا الجهاز.
4. `of_iommu_get_resv_regions()` تقرأ خاصية `memory-region` و`iommu-addresses` من الـ DT لتعرف: ما هي مناطق الـ IOVA المحجوزة لهذا الجهاز (مثل مناطق الـ firmware أو الـ DMA الثابت) التي يجب ألا يستخدمها أي شخص آخر.

---

### القصة الكاملة بمثال

لنقل أن لديك SoC من ARM يحتوي على:
- GPU متصل بـ SMMU (ARM's IOMMU)
- بطاقة شبكة متصلة بنفس الـ SMMU

في الـ Device Tree:
```
gpu@ff000000 {
    iommus = <&smmu 0x01>;  /* هذا الـ GPU له ID = 0x01 في الـ SMMU */
};

eth@ff100000 {
    iommus = <&smmu 0x02>;  /* الشبكة لها ID = 0x02 */
};
```

عندما يبدأ الكيرنل:
1. يجد الـ GPU ويريد تهيئته مع الـ IOMMU.
2. يُنادي `of_iommu_configure(gpu_dev, gpu_node, NULL)`.
3. هذه الدالة تقرأ `iommus = <&smmu 0x01>` من الـ DT.
4. تجد الـ SMMU driver وتُسجّل الـ GPU معه بـ stream ID = 0x01.
5. الآن الـ GPU محاصر داخل مساحة العناوين الوهمية الخاصة به — لا يستطيع لمس ذاكرة الكيرنل أو ذاكرة الشبكة.

---

### لماذا هذا الـ header مهم؟

- **الفصل بين طبقات الكود**: الـ bus code (مثل `platform_bus`، `amba_bus`) يُنادي `of_iommu_configure()` دون أن يعرف شيئاً عن تفاصيل أي IOMMU driver محدد.
- **دعم `CONFIG_OF_IOMMU`**: إذا لم يكن الـ Device Tree مُفعّلاً في الـ build، الدوال تتحول إلى stubs فارغة (أو ترجع `-ENODEV`) فلا يتعطل الكود على x86.
- **تجريد النظام**: نفس كود الـ bus يعمل على Qualcomm SMMU، Apple DART، ARM SMMU v1/v2/v3، MediaTek IOMMU — لأن الجميع يتحدث عبر هذا الـ interface الموحّد.

---

### الملفات المرتبطة التي يجب معرفتها

| الملف | الدور |
|---|---|
| `include/linux/of_iommu.h` | **هذا الملف** — الـ API العلني لربط DT بالـ IOMMU |
| `drivers/iommu/of_iommu.c` | التنفيذ الفعلي للدالتين المُعلنتين في الـ header |
| `include/linux/iommu.h` | الـ core IOMMU API — تعريف `iommu_ops`، `iommu_domain`، `iommu_fwspec` |
| `include/linux/iova.h` | إدارة مساحة عناوين الـ I/O الوهمية |
| `drivers/iommu/iommu.c` | الـ core IOMMU subsystem — `iommu_probe_device`، `iommu_fwspec_init` |
| `drivers/iommu/arm/arm-smmu/` | درايفر ARM SMMU v1/v2 (يستخدم `of_xlate`) |
| `drivers/iommu/arm/arm-smmu-v3/` | درايفر ARM SMMU v3 الأحدث |
| `drivers/iommu/intel/` | درايفر Intel VT-d IOMMU |
| `drivers/iommu/amd/` | درايفر AMD-Vi IOMMU |
| `drivers/iommu/apple-dart.c` | درايفر Apple DART IOMMU |
| `drivers/iommu/mtk_iommu.c` | درايفر MediaTek IOMMU |
| `Documentation/devicetree/bindings/iommu/` | توثيق خصائص الـ DT للـ IOMMU |

---

### ملخص الهدف

**الـ `of_iommu.h`** يُعلن عن دالتين فقط، لكنهما تُمثّلان الجسر الحيوي الذي يجعل كل جهاز مدمج في ARM محمياً بالـ IOMMU تلقائياً عند الإقلاع — بمجرد أن يصف الـ Device Tree العلاقة بين الجهاز والـ IOMMU. بدونهما، لا IOMMU يعمل على أي منصة تعتمد على الـ Device Tree.
## Phase 2: شرح الـ OF_IOMMU Framework

### المشكلة — لماذا يوجد هذا الـ subsystem؟

في الأنظمة المدمجة الحديثة (مثل SoCs تعتمد ARM)، يوجد عادةً **IOMMU** (Input-Output Memory Management Unit) — وهو وحدة hardware تترجم الـ I/O virtual addresses التي تصدرها الأجهزة الطرفية (DMA masters) إلى physical addresses حقيقية. هذا يعطي عزلاً (isolation) وحماية للذاكرة.

المشكلة: لكي يعمل الـ IOMMU، يجب أن يعرف الـ kernel:
1. أي IOMMU يتحكم في أي device؟
2. ما هو الـ stream ID (أو master ID) الخاص بكل device؟

في بيئة قائمة على **Device Tree** (وهي البيئة الشائعة في ARM embedded)، كل هذه المعلومات موجودة في الـ `.dts` file — لكن لا يوجد بدون `of_iommu` أي آلية لتحويل هذه البيانات النصية إلى ارتباط حقيقي داخل الـ kernel بين الـ device والـ IOMMU subsystem.

**الـ `of_iommu` subsystem يحل هذه المشكلة بالضبط**: يقرأ الـ Device Tree، ويربط كل device بالـ IOMMU driver المناسب، ويمرر الـ stream IDs الصحيحة.

---

### الحل — ماذا يفعل الـ kernel؟

الـ kernel يقسم المسألة إلى مرحلتين:

**1. وقت البوت (boot time):** الـ Device Tree يُحمَّل وتُبنى منه شجرة `device_node` في الذاكرة.

**2. وقت probe الجهاز:** عندما يُراد ربط device بـ driver، تُستدعى `of_iommu_configure()` لتقرأ الـ DT properties مثل `iommus = <&smmu 0x100>` وتُنشئ الارتباط الصحيح مع الـ IOMMU driver.

> **ملاحظة:** الـ **IOMMU subsystem** في الـ kernel (`linux/iommu.h`) هو subsystem مستقل يُعرِّف `struct iommu_ops`، `struct iommu_domain`، إلخ. الـ `of_iommu` ليس سوى **جسر DT** فوقه.

---

### المكونات المترابطة — بنية الـ Device Tree ذات الصلة

قبل أن نفهم الكود، لازم نفهم كيف يبدو الـ DT:

```
/* مثال حقيقي: Arm Cortex-A SoC */
smmu0: iommu@d4000000 {
    compatible = "arm,mmu-500";
    reg = <0xd4000000 0x20000>;
    #iommu-cells = <1>;           /* كل device يمرر stream ID واحد */
};

usb: usb@d1000000 {
    compatible = "generic-ehci";
    reg = <0xd1000000 0x1000>;
    iommus = <&smmu0 0x100>;      /* مرتبط بـ smmu0 بـ stream ID = 0x100 */
};
```

**الـ `iommus` property** هي الرابط. الـ `of_iommu` يقرأها ويترجمها.

---

### المشهد الكبير — أين يقع OF_IOMMU في الـ kernel؟

```
+--------------------------------------------------+
|            User Space (DMA operations)           |
+--------------------------------------------------+
          |
+--------------------------------------------------+
|           Linux Kernel                           |
|                                                  |
|  +-----------+    +----------+    +-----------+  |
|  |  Device   |    |  DMA API |    | IOMMUFD   |  |
|  |  Driver   |--->| (iommu_  |    | (vIOMMU   |  |
|  | (usb,gpu) |    |  map/    |    | for VMs)  |  |
|  +-----------+    | unmap)   |    +-----------+  |
|       |           +----+-----+         |         |
|       |                |               |         |
|  +----|----------------|-----------+   |         |
|  |    v                v           |   |         |
|  |        IOMMU Core Subsystem     |   |         |
|  |   (iommu.h: domain, group,      |<--+         |
|  |    iommu_ops, iommu_attach...)  |             |
|  +-----------+---------------------+             |
|              |                                   |
|  +-----------v---------------------+             |
|  |      OF_IOMMU Bridge            |             |
|  |  (of_iommu.h)                   |             |
|  |  - of_iommu_configure()         |             |
|  |  - of_iommu_get_resv_regions()  |             |
|  +-----------+---------------------+             |
|              |                                   |
|  +-----------v---------------------+             |
|  |     Device Tree Subsystem       |             |
|  |  (of.h: device_node, property,  |             |
|  |   of_phandle_args, phandle...)  |             |
|  +-----------+---------------------+             |
|              |                                   |
|  +-----------v---------------------+             |
|  |  Platform IOMMU Driver          |             |
|  |  (arm-smmu.c, arm-smmu-v3.c,   |             |
|  |   intel-iommu.c, etc.)          |             |
|  |  implements: iommu_ops           |             |
|  |  - .of_xlate()                  |             |
|  |  - .probe_device()              |             |
|  |  - .get_resv_regions()          |             |
|  +-----------+---------------------+             |
|              |                                   |
+--------------|-----------------------------------+
               |
     +---------v---------+
     |  IOMMU Hardware   |
     | (ARM SMMUv3, etc) |
     +-------------------+
```

---

### تشريح الـ API — الدوال الجوهرية

الملف `of_iommu.h` صغير لكنه محوري. يُعرِّف دالتين فقط:

#### `of_iommu_configure()`

```c
extern int of_iommu_configure(struct device *dev,
                              struct device_node *master_np,
                              const u32 *id);
```

**هذه هي الدالة المحورية.** تُستدعى من `of_dma_configure()` في وقت الـ probe لأي device مدعوم بـ DT.

| المعامل | النوع | الوظيفة |
|---------|-------|---------|
| `dev` | `struct device *` | الجهاز المراد تهيئته (مثلاً USB controller) |
| `master_np` | `struct device_node *` | الـ DT node الخاص بهذا الجهاز |
| `id` | `const u32 *` | stream ID اختياري (NULL = يقرأ من DT) |

**ماذا تفعل داخلياً؟**

```
of_iommu_configure(dev, master_np, id)
     |
     v
[1] تقرأ "iommus" property من الـ DT node
     |
     v
[2] تحلل struct of_phandle_args:
     .np    = &smmu0_node   (phandle → device_node)
     .args  = [0x100]       (stream ID)
     |
     v
[3] تجد الـ IOMMU driver المسجل لهذا الـ node
     (عبر of_parse_phandle_with_args + iommu_fwspec_init)
     |
     v
[4] تستدعي iommu_ops->of_xlate(dev, &args)
     (يضيف الـ stream ID إلى iommu_fwspec الخاص بالجهاز)
     |
     v
[5] تستدعي iommu_probe_device(dev)
     (تجعل الـ IOMMU driver يبدأ تتبع هذا الجهاز)
     |
     v
[6] ترجع 0 عند النجاح، أو error code
```

#### `of_iommu_get_resv_regions()`

```c
extern void of_iommu_get_resv_regions(struct device *dev,
                                      struct list_head *list);
```

تقرأ من الـ DT مناطق الذاكرة المحجوزة (reserved regions) المرتبطة بالجهاز — مثل مناطق MSI أو مناطق يجب mapping-ها 1:1. تُضاف إلى `list` كـ `struct iommu_resv_region`.

**لماذا هذا مهم؟** بعض الـ hardware (مثل MSI controllers) يتطلب أن تبقى عناوين معينة بدون ترجمة IOMMU (passthrough)، أو يجب حجزها لكيلا تُعطى للـ DMA API.

---

### الـ Stub عند غياب `CONFIG_OF_IOMMU`

```c
#ifndef CONFIG_OF_IOMMU

static inline int of_iommu_configure(struct device *dev,
                                     struct device_node *master_np,
                                     const u32 *id)
{
    return -ENODEV;  /* لا يوجد IOMMU في الـ DT */
}

static inline void of_iommu_get_resv_regions(struct device *dev,
                                             struct list_head *list)
{
    /* لا شيء */
}

#endif
```

هذا النمط شائع جداً في الـ kernel: **compile-time dead code elimination**. إذا لم يكن `CONFIG_OF_IOMMU` مفعَّلاً، الـ compiler يُبسِّط كل الاستدعاءات إلى `return -ENODEV` أو no-op، بدون أي overhead.

---

### الـ Structs المترابطة — كيف تتصل ببعضها

```
struct device
+----------------------------+
| struct device_node *of_node|----> struct device_node (DT node)
|                            |      +----------------------+
| struct iommu_group *iommu_ |      | const char *name     |
|   group                    |      | struct property      |
|                            |      |   *properties        |
| struct dev_iommu *iommu    |      |  (iommus, ...)       |
|   +----------------------+ |      +----------------------+
|   | struct iommu_fwspec  | |
|   |   *fwspec            | |
|   |   +----------------+ | |
|   |   | struct fwnode  | | |     struct iommu_ops
|   |   |  _handle *iommu| |----> +--------------------+
|   |   |  _fwnode       | | |   | .of_xlate()        |
|   |   | u32 ids[]      | | |   | .probe_device()    |
|   |   | (stream IDs)   | | |   | .get_resv_regions()|
|   |   +----------------+ | |   +--------------------+
|   +----------------------+ |
+----------------------------+

struct of_phandle_args       (ناتج تحليل "iommus = <&smmu 0x100>")
+---------------------------+
| struct device_node *np    |---> node الخاص بالـ IOMMU hardware
| int args_count            |     (smmu0 في مثالنا)
| uint32_t args[MAX]        |     args[0] = 0x100 (stream ID)
+---------------------------+

struct iommu_resv_region     (لـ of_iommu_get_resv_regions)
+----------------------------+
| struct list_head list      |
| phys_addr_t start          |
| size_t length              |
| int prot                   |
| enum iommu_resv_type type  |
|   IOMMU_RESV_DIRECT        |
|   IOMMU_RESV_MSI           |
|   IOMMU_RESV_RESERVED ...  |
| void (*free)(...)          |
+----------------------------+
```

---

### التشابه الواقعي — المطار وبطاقات الصعود

تخيل **مطاراً حديثاً** به نظام أمني متعدد المناطق:

| عنصر الـ kernel | التشابه في المطار |
|----------------|-----------------|
| **IOMMU hardware** | بوابات الأمن في المطار (تتحقق من كل شخص يدخل) |
| **IOMMU driver** (`arm-smmu.c`) | طاقم الأمن المتخصص في نوع البوابة |
| **IOMMU Core** (`iommu.h`) | إدارة المطار المركزية (سياسات موحدة) |
| **Device Tree** | قاعدة بيانات الرحلات والمسافرين (من يسافر على أي رحلة) |
| **`of_iommu_configure()`** | عملية **check-in**: تطابق المسافر (device) بالرحلة (IOMMU) وتصدر بطاقة صعود (fwspec مع stream ID) |
| **`of_iommu_get_resv_regions()`** | قائمة المناطق المحظورة في المطار (لا يُسمح لأي أحد بالدخول إليها) |
| **`struct of_phandle_args`** | بطاقة الهوية + رقم المقعد (معرف الـ device + stream ID) |
| **`iommu_ops->of_xlate()`** | موظف الأمن الذي يسجل المسافر في النظام |
| **`CONFIG_OF_IOMMU=n`** | مطار بدون نظام أمني — الكل يدخل بدون فحص (أو الدخول ممنوع بالكامل) |

الدقة في التشابه: عملية الـ check-in (`of_iommu_configure`) لا تكون عند كل رحلة — بل **مرة واحدة فقط** عند boot وقت probe الجهاز، تماماً كما يسجّل المسافر نفسه في النظام مرة واحدة عند وصوله للمطار.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ OF_IOMMU subsystem يحقق فصلاً واضحاً:**

```
[ما يصفه الـ Hardware Engineer في DTS]
    iommus = <&smmu0 0x100>;
          |
          | (of_iommu_configure يحوِّل هذا الوصف إلى...)
          v
[ما يستخدمه الـ Kernel Driver]
    dev->iommu->fwspec->ids[] = {0x100}
    dev->iommu_group = smmu0_group
```

**الـ `iommu_fwspec`** هو القلب المجرد: هيكل يجمع كل الـ stream IDs الخاصة بجهاز واحد، ويربطها بـ firmware node (fwnode) للـ IOMMU المسؤول. الـ IOMMU driver لا يحتاج أن يعرف شيئاً عن DT — هذا عمل `of_iommu` وحده.

---

### ما يملكه OF_IOMMU مقابل ما يفوِّضه للـ Drivers

| المسؤولية | مَن يتولاها |
|-----------|------------|
| قراءة `iommus` property من DT | `of_iommu` (عبر `of_parse_phandle_with_args`) |
| تحديد IOMMU driver المناسب | `of_iommu` (عبر `fwnode` lookup) |
| تحليل stream ID وتسجيله في `fwspec` | الـ IOMMU driver (عبر `iommu_ops->of_xlate`) |
| بناء الـ `iommu_group` | الـ IOMMU driver (عبر `iommu_ops->device_group`) |
| إعداد الـ `iommu_domain` وربطه | الـ IOMMU Core + driver |
| قراءة reserved regions من DT | `of_iommu` (عبر `of_iommu_get_resv_regions`) |
| تطبيق reserved regions على الـ hardware | الـ IOMMU driver (عبر `iommu_ops->get_resv_regions`) |
| mapping/unmapping فعلي في الـ page tables | الـ IOMMU driver بالكامل |

الخلاصة: **الـ `of_iommu` يملك فقط الـ DT parsing وربط الـ device بالـ IOMMU driver.** كل شيء يتعلق بالـ hardware (page tables، TLB flush، fault handling) مفوَّض بالكامل لـ IOMMU driver.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags — Cheatsheet

#### الـ Config Options

| Option | الوصف |
|--------|-------|
| `CONFIG_OF_IOMMU` | يُفعِّل دعم الـ IOMMU عبر Device Tree. بدونه تُرجع الدوال stubs تُعيد `‑ENODEV` أو لا تفعل شيئاً |
| `CONFIG_OF_ADDRESS` | مطلوب داخل `of_iommu_get_resv_regions()` لتفسير عناوين الـ DT عبر `of_address_to_resource()` |
| `CONFIG_IOMMU_API` | يُفعِّل الـ IOMMU core بالكامل بما فيه `iommu_probe_device()` |
| `CONFIG_FSL_PAMU` | يُفعِّل المسار القديم `domain_alloc` في `iommu_ops` (Freescale فقط) |

#### الـ IOMMU Protection Flags (prot)

| Flag | القيمة | المعنى |
|------|--------|--------|
| `IOMMU_READ` | bit 0 | الـ device يقرأ من الذاكرة |
| `IOMMU_WRITE` | bit 1 | الـ device يكتب إلى الذاكرة |
| `IOMMU_CACHE` | bit 2 | الـ mapping يحترم الـ cache coherency |
| `IOMMU_NOEXEC` | bit 3 | منع تنفيذ الكود |
| `IOMMU_MMIO` | bit 4 | منطقة MMIO |

#### الـ iommu_resv_type (Enum)

| القيمة | المعنى |
|--------|--------|
| `IOMMU_RESV_DIRECT` | خريطة 1:1 صارمة بين IOVA والعنوان الفيزيائي |
| `IOMMU_RESV_DIRECT_RELAXABLE` | 1:1 لكن يمكن تخفيفها في حالات كـ device assignment |
| `IOMMU_RESV_RESERVED` | محجوز — لا يُعطى للـ device ولا يُربط |
| `IOMMU_RESV_MSI` | منطقة MSI الـ hardware (غير مترجَمة) |
| `IOMMU_RESV_SW_MSI` | نافذة ترجمة MSI مُدارة بالـ software |

#### الـ iommu_fwspec flags

| Flag | المعنى |
|------|--------|
| `IOMMU_FWSPEC_PCI_RC_ATS` | الـ PCI Root Complex يدعم ATS (Address Translation Services) |

---

### الـ Structs المهمة

#### 1. `struct device_node` — تعريف عقدة الـ Device Tree

**الغرض**: تمثيل عقدة واحدة في شجرة الـ Device Tree داخل الكرنل. كل جهاز لديه `device_node` يصف خصائصه الـ hardware.

```c
struct device_node {
    const char *name;          /* اسم العقدة مثل "ethernet@1000" */
    phandle phandle;           /* معرّف فريد للإشارة إليها من عقد أخرى */
    const char *full_name;     /* المسار الكامل مثل "/soc/ethernet@1000" */
    struct fwnode_handle fwnode; /* واجهة موحّدة للـ firmware nodes */

    struct property *properties; /* قائمة الـ properties (iommus=, memory-region=...) */
    struct property *deadprops;  /* properties محذوفة */
    struct device_node *parent;  /* العقدة الأب في الشجرة */
    struct device_node *child;   /* العقدة الابن الأول */
    struct device_node *sibling; /* العقدة المجاورة */
    unsigned long _flags;        /* OF_POPULATED, OF_DYNAMIC, ... */
    void *data;                  /* بيانات خاصة بالـ arch */
};
```

**الارتباطات**:
- `of_iommu_configure()` تستقبلها كـ `master_np` لتحديد عقدة الجهاز في الـ DT
- `of_iommu_xlate()` تصل إلى `fwnode` منها لجلب الـ `iommu_ops`
- `of_iommu_get_resv_regions()` تسير عبر `memory-region` phandles في `dev->of_node`

---

#### 2. `struct of_phandle_args` — وسيطات الـ Phandle

**الغرض**: حاوية مؤقتة تُعبّر عن نتيجة تفسير `phandle` من الـ DT، مع وسيطاته العددية (مثل stream ID).

```c
struct of_phandle_args {
    struct device_node *np;             /* العقدة المُشار إليها (IOMMU controller node) */
    int args_count;                     /* عدد الوسيطات المُمرَّرة */
    uint32_t args[MAX_PHANDLE_ARGS];    /* الوسيطات: مثلاً stream ID أو SID */
};
```

**مثال حقيقي من الـ DT**:
```
/* في ملف dts */
ethernet0: ethernet@1000 {
    iommus = <&smmu 0x42>;   /* np → &smmu, args[0] = 0x42 (stream ID) */
};
```

**الارتباطات**:
- `of_iommu_configure_dev()` تُنشئها عبر `of_parse_phandle_with_args()`
- `of_iommu_configure_dev_id()` تُنشئها عبر `of_map_id()`
- تُمرَّر إلى `ops->of_xlate()` لتسجيل الـ stream ID في الـ IOMMU driver

---

#### 3. `struct of_phandle_iterator` — مُكرِّر الـ Phandles

**الغرض**: حالة مؤقتة لتكرار قائمة phandles في property واحدة (مثل `memory-region`).

```c
struct of_phandle_iterator {
    const char *cells_name;     /* اسم الـ property التي تحدد حجم الوسيطات */
    int cell_count;             /* عدد الـ cells لكل entry */
    const struct device_node *parent;
    const __be32 *list_end;     /* نهاية قائمة الـ phandles في الـ binary blob */
    const __be32 *phandle_end;
    const __be32 *cur;          /* الموضع الحالي في القراءة */
    uint32_t cur_count;
    phandle phandle;
    struct device_node *node;   /* العقدة الحالية المُفسَّرة */
};
```

**الارتباطات**: تُستخدَم في `of_iommu_get_resv_regions()` عبر الماكرو `of_for_each_phandle()` لتكرار كل `memory-region` مرتبط بالجهاز.

---

#### 4. `struct iommu_ops` — جدول عمليات الـ IOMMU Driver

**الغرض**: الواجهة الرئيسية بين الـ IOMMU core وأي driver حقيقي. يُسجِّل الـ driver نفسه بملء هذا الـ struct.

```c
struct iommu_ops {
    /* --- قدرات عامة --- */
    bool (*capable)(struct device *dev, enum iommu_cap);

    /* --- إنشاء الـ domains --- */
    struct iommu_domain *(*domain_alloc_identity)(struct device *dev);
    struct iommu_domain *(*domain_alloc_paging)(struct device *dev);
    struct iommu_domain *(*domain_alloc_sva)(struct device *dev, struct mm_struct *mm);
    struct iommu_domain *(*domain_alloc_nested)(...);

    /* --- إدارة الأجهزة --- */
    struct iommu_device *(*probe_device)(struct device *dev);
    void (*release_device)(struct device *dev);
    void (*probe_finalize)(struct device *dev);
    struct iommu_group *(*device_group)(struct device *dev);

    /* --- المناطق المحجوزة (نقطة الربط مع of_iommu.h) --- */
    void (*get_resv_regions)(struct device *dev, struct list_head *list);

    /* --- Device Tree integration (نقطة الربط مع of_iommu.h) --- */
    int (*of_xlate)(struct device *dev, const struct of_phandle_args *args);

    bool (*is_attach_deferred)(struct device *dev);
    int (*def_domain_type)(struct device *dev);

    /* --- معلومات ثابتة --- */
    const struct iommu_domain_ops *default_domain_ops;
    struct module *owner;
    struct iommu_domain *identity_domain;
    struct iommu_domain *blocked_domain;
    struct iommu_domain *default_domain;
    u8 user_pasid_table:1;
};
```

**نقاط الربط مع `of_iommu.h`**:
- الـ `of_xlate` callback: يُستدعى من `of_iommu_xlate()` لتسجيل كل stream ID
- الـ `get_resv_regions` callback: يمكن للـ driver تفويضه إلى `of_iommu_get_resv_regions()`

---

#### 5. `struct iommu_resv_region` — منطقة ذاكرة محجوزة

**الغرض**: وصف نطاق IOVA يجب حجزه أو ربطه بطريقة خاصة — لا يعطيه الـ IOMMU core لأي driver DMA.

```c
struct iommu_resv_region {
    struct list_head list;   /* لربطها في قائمة المناطق المحجوزة للجهاز */
    phys_addr_t start;       /* بداية النطاق الفيزيائي (أو IOVA في حالة RESV_RESERVED) */
    size_t length;           /* الطول بالبايت */
    int prot;                /* IOMMU_READ | IOMMU_WRITE | IOMMU_CACHE */
    enum iommu_resv_type type; /* نوع الحجز: DIRECT, RESERVED, MSI... */
    void (*free)(...);       /* callback لتحرير الذاكرة المرتبطة (اختياري) */
};
```

**الارتباطات**:
- `of_iommu_get_resv_regions()` تُنشئ هذه الكائنات وتضيفها إلى `list`
- تُستدعى عبر `iommu_ops->get_resv_regions()` من الـ IOMMU core
- الـ DMA core يستعلم عنها عبر `iommu_get_resv_regions()` ويتجنب التعيين عليها

---

#### 6. `struct iommu_device` — تمثيل الـ IOMMU Core لـ Hardware Instance

**الغرض**: الـ IOMMU core يُتتبَّع به كل IOMMU hardware مسجَّل في النظام.

```c
struct iommu_device {
    struct list_head list;          /* مرتبط بقائمة iommu_device_list العالمية */
    const struct iommu_ops *ops;    /* driver ops لهذا الـ hardware */
    struct fwnode_handle *fwnode;   /* يربطه بعقدة الـ DT أو ACPI */
    struct device *dev;             /* الـ sysfs device */
    struct iommu_group *singleton_group;
    u32 max_pasids;
    u32 ready:1;
};
```

**الارتباطات**:
- `iommu_ops_from_fwnode()` تبحث عن `iommu_device` عبر `fwnode` لإرجاع الـ `iommu_ops`
- هذا الاستعلام يحدث في `of_iommu_xlate()` بعد تحديد عقدة الـ IOMMU من الـ DT

---

#### 7. `struct of_pci_iommu_alias_info` — بيانات مساعدة للـ PCI

**الغرض**: struct محلي في `of_iommu.c` يُمرَّر كـ `void *data` إلى `pci_for_each_dma_alias()`.

```c
struct of_pci_iommu_alias_info {
    struct device *dev;      /* الجهاز الـ PCI المُراد تهيئته */
    struct device_node *np;  /* عقدة الـ DT الخاصة بالجهاز (master_np) */
};
```

**الارتباطات**: يُنشأ مؤقتاً في `of_iommu_configure()` عند الكشف عن جهاز PCI، ويُمرَّر إلى `of_pci_iommu_init()` callback.

---

### مخططات علاقات الـ Structs (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                     Device Tree (DTS)                   │
│  ethernet0: { iommus = <&smmu 0x42>;                   │
│               memory-region = <&resv0>;  }              │
└────────────┬───────────────────┬────────────────────────┘
             │                   │
             ▼                   ▼
  ┌──────────────────┐   ┌──────────────────────┐
  │  device_node     │   │  device_node         │
  │  (master_np)     │   │  (memory-region node)│
  │  .name = "eth0"  │   │  .name = "resv0"     │
  │  .fwnode ─────────────► fwnode_handle       │
  │  .properties     │   │  .properties         │
  │  ─────────────── │   │  (reg, iommu-addrs)  │
  └──────┬───────────┘   └──────────────────────┘
         │                        │
         │  of_parse_phandle      │  of_for_each_phandle
         ▼                        ▼
  ┌──────────────────┐   ┌──────────────────────┐
  │  of_phandle_args │   │ of_phandle_iterator  │
  │  .np ──────────────► smmu device_node       │
  │  .args[0] = 0x42 │   │ .node (current DT)   │
  └──────┬───────────┘   └──────┬───────────────┘
         │                      │
         │ of_iommu_xlate()     │ iommu_alloc_resv_region()
         ▼                      ▼
  ┌──────────────────┐   ┌──────────────────────┐
  │  iommu_ops       │   │ iommu_resv_region    │
  │  .of_xlate ──────┘   │ .start = iova        │
  │  .get_resv_regions   │ .length              │
  │  .probe_device   │   │ .type = DIRECT/RESV  │
  │  .owner ─────────────► struct module        │
  └──────┬───────────┘   └──────┬───────────────┘
         │                      │ list_add_tail
         ▼                      ▼
  ┌──────────────────┐   ┌──────────────────────┐
  │  iommu_device    │   │  list_head (resv list│
  │  .ops ───────────┘   │  in device->iommu)   │
  │  .fwnode             └──────────────────────┘
  │  .dev
  └──────────────────┘
```

---

### مخطط دورة حياة `of_iommu_configure()`

```
  الجهاز يظهر في الـ bus (probe)
         │
         ▼
  of_dma_configure() أو bus code
         │
         ▼
  of_iommu_configure(dev, master_np, id)
         │
         ├─► [master_np == NULL?] ──► return -ENODEV
         │
         ├─► mutex_lock(&iommu_probe_device_lock)
         │
         ├─► [fwspec موجود بالفعل?] ──► unlock + return 0 (مُهيَّأ مسبقاً)
         │
         ├─► [PCI device?]
         │       │
         │       └─► pci_request_acs()
         │           pci_for_each_dma_alias()
         │             └─► of_pci_iommu_init()
         │                   └─► of_iommu_configure_dev_id()
         │                         └─► of_map_id() → of_phandle_args
         │                               └─► of_iommu_xlate()
         │                                     └─► iommu_fwspec_init()
         │                                         ops->of_xlate()  ◄── Driver callback
         │
         ├─► [Platform device?]
         │       │
         │       └─► of_iommu_configure_device()
         │               │
         │               ├─► [id != NULL] → of_iommu_configure_dev_id()
         │               │                   └─► of_map_id() → of_iommu_xlate()
         │               │
         │               └─► [id == NULL] → of_iommu_configure_dev()
         │                                   └─► of_parse_phandle_with_args() loop
         │                                         └─► of_iommu_xlate() per entry
         │
         ├─► mutex_unlock(&iommu_probe_device_lock)
         │
         └─► [!err && !dev_iommu_present?]
               └─► iommu_probe_device(dev)
                     └─► ops->probe_device()  ◄── Driver callback
                         ops->probe_finalize()
```

---

### مخطط دورة حياة `of_iommu_get_resv_regions()`

```
  iommu_probe_device() أو iommu_get_resv_regions()
         │
         ▼
  ops->get_resv_regions(dev, list)
    [إذا قرر الـ driver استخدام helper الـ DT]
         │
         ▼
  of_iommu_get_resv_regions(dev, list)
         │
         ├─► [CONFIG_OF_ADDRESS غير مُفعَّل?] ──► return فوراً
         │
         ├─► of_for_each_phandle(&it, err, dev->of_node, "memory-region", ...)
         │       │
         │       ▼  لكل memory-region node
         │   ┌───────────────────────────────────────────┐
         │   │ of_address_to_resource(it.node) → phys   │
         │   │ of_get_property("iommu-addresses") → maps│
         │   │                                           │
         │   │  لكل entry في iommu-addresses:           │
         │   │    be32_to_cpup() → phandle              │
         │   │    of_find_node_by_phandle() → np        │
         │   │                                           │
         │   │    [np == dev->of_node?]                 │
         │   │       │                                   │
         │   │       ▼                                   │
         │   │    of_translate_dma_region() → iova,len  │
         │   │    iommu_resv_region_get_type() → type   │
         │   │    iommu_alloc_resv_region(iova,len,prot)│
         │   │    list_add_tail(&region->list, list)    │
         │   └───────────────────────────────────────────┘
         │
         └─► return  (القائمة list مُمتلئة بالـ regions)

  لاحقاً:
  iommu_put_resv_regions(dev, list)
    └─► kfree() أو region->free() لكل region
```

---

### مخطط تدفق الـ Calls الكامل

```
  bus/platform driver init
         │
         ▼
  of_dma_configure(dev, dev->of_node)              [drivers/of/device.c]
         │
         └─► of_iommu_configure(dev, np, NULL)     [of_iommu.c - PUBLIC API]
                 │
                 └─► of_iommu_configure_dev(np, dev)
                         │
                         └─► of_parse_phandle_with_args(np, "iommus",
                             │                           "#iommu-cells", idx)
                             ▼
                         of_iommu_xlate(dev, &iommu_spec)
                             │
                             ├─► iommu_fwspec_init(dev, fwnode)
                             │       └─► alloc dev->iommu->fwspec
                             │
                             ├─► iommu_ops_from_fwnode(fwnode)
                             │       └─► بحث في قائمة iommu_device_list
                             │
                             └─► ops->of_xlate(dev, &iommu_spec)
                                     └─► [SMMU driver مثلاً]
                                         arm_smmu_of_xlate()
                                           └─► iommu_fwspec_add_ids(dev, sid)

  ──────────────────────────────────────────

  iommu_get_resv_regions(dev, list)            [iommu.c - core]
         │
         └─► ops->get_resv_regions(dev, list)
                 │
                 └─► of_iommu_get_resv_regions(dev, list)  [of_iommu.c - PUBLIC API]
                         │
                         └─► iommu_alloc_resv_region(iova, len, prot, type)
                                 └─► kzalloc(sizeof(*region))
                                     region->start = iova
                                     region->type  = DIRECT / RESERVED
                                     list_add_tail(&region->list, list)

  ──────────────────────────────────────────

  DMA subsystem (dma-iommu.c):
  iommu_dma_get_resv_regions(dev, list)
         └─► of_iommu_get_resv_regions(dev, list)
```

---

### استراتيجية الـ Locking

#### الـ Mutex الرئيسي: `iommu_probe_device_lock`

| الـ Lock | النوع | ما يحميه |
|---------|------|---------|
| `iommu_probe_device_lock` | `mutex` | سباق التهيئة بين خيوط متعددة تحاول تهيئة نفس الجهاز |

**التفاصيل**:
```
of_iommu_configure()
  mutex_lock(&iommu_probe_device_lock)   ← احجز قبل أي فحص لـ dev->iommu
  if (dev_iommu_fwspec_get(dev)) {       ← تحقق من التهيئة المسبقة
      mutex_unlock(...)
      return 0;                          ← آمن: لا تكرار للتهيئة
  }
  /* تهيئة dev->iommu->fwspec هنا */
  mutex_unlock(&iommu_probe_device_lock) ← حرر بعد إتمام fwspec
```

**لماذا بالتحديد هنا؟**
الـ `dev->iommu` و `dev->iommu->fwspec` يمكن أن يُكتَب فيهما من مسارات متعددة:
- من bus probe (مثلاً `platform_bus`)
- من `iommu_probe_device()` المُستدعى من IOMMU notifier

الـ mutex يضمن أن فقط خيط واحد يُنشئ الـ fwspec ويُسجِّل الـ stream IDs.

#### الـ `module_get` / `module_put` في `of_iommu_xlate()`

```c
if (!ops->of_xlate || !try_module_get(ops->owner))
    return -ENODEV;

ret = ops->of_xlate(dev, iommu_spec);  /* استدعاء آمن — module لن يُفرَّغ */
module_put(ops->owner);
```

هذا حماية من unloading الـ IOMMU driver module أثناء تنفيذ callback الـ `of_xlate`.

#### الـ `of_node_put()` — إدارة المراجع

```
of_parse_phandle_with_args(..., &iommu_spec)
    └─► iommu_spec.np مُحتجَز بـ refcount زائد

of_iommu_xlate(dev, &iommu_spec)

of_node_put(iommu_spec.np)   ← تحرير المرجع فور الانتهاء
```

لا يوجد lock صريح هنا — الـ Device Tree nodes محمية بـ reference counting (kref داخلي).

#### ملخص ترتيب الـ Locking

```
1. iommu_probe_device_lock  (mutex — الأعلى)
      └── يحمي: dev->iommu، dev->iommu->fwspec
2. module refcount (try_module_get)
      └── يحمي: iommu_ops vtable
3. of_node refcount (of_node_get/put)
      └── يحمي: device_node أثناء الاستخدام
```

لا يوجد خطر deadlock لأن هذه الـ locks لا تتشابك: الـ mutex يُفك قبل استدعاء `iommu_probe_device()` التي قد تحتاج locks خاصة بها.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| الـ Function | الـ Config Guard | الـ Return | الغرض |
|---|---|---|---|
| `of_iommu_configure()` | `CONFIG_OF_IOMMU` | `int` (0 / errno) | ربط device بـ IOMMU عبر Device Tree |
| `of_iommu_get_resv_regions()` | `CONFIG_OF_IOMMU` + `CONFIG_OF_ADDRESS` | `void` | قراءة reserved IOVA regions من DT وإضافتها للـ list |

**الـ stubs** (عند غياب `CONFIG_OF_IOMMU`):

| الـ Function | الـ Return |
|---|---|
| `of_iommu_configure()` stub | `-ENODEV` دائماً |
| `of_iommu_get_resv_regions()` stub | لا شيء، no-op |

---

### تصنيف الـ Functions

```
of_iommu.h
│
├── [Configuration]
│   └── of_iommu_configure()          ← ربط device بـ IOMMU من DT
│
└── [Reserved Regions Query]
    └── of_iommu_get_resv_regions()   ← استخراج IOVA reservations من DT
```

---

### الفئة الأولى: Configuration

هذه الفئة مسؤولة عن **إعداد وربط الـ device بالـ IOMMU** المناسب بالاعتماد على معلومات الـ Device Tree. تُستدعى في مرحلة الـ probe للـ bus (مثل platform bus وPCI bus) وتقوم بملء `iommu_fwspec` الخاص بالـ device وإطلاق `iommu_probe_device()` إذا لزم.

---

#### `of_iommu_configure()`

```c
int of_iommu_configure(struct device *dev,
                       struct device_node *master_np,
                       const u32 *id);
```

**ما تفعله:**
الـ function الرئيسية المُصدَّرة في الـ header. تفحص الـ device وتقرر مسار الـ configuration: إذا كان الـ device PCI تستخدم `pci_for_each_dma_alias()` لتغطية جميع الـ aliases، وإذا كان non-PCI تستخدم `of_iommu_configure_device()` مباشرةً. تحمي العملية بـ `iommu_probe_device_lock` لضمان atomicity عند بناء الـ `fwspec`.

في حالة نجاح الـ configuration وكان الـ device لم يكن مرتبطاً بـ IOMMU سابقاً (أي `dev->iommu == NULL` في بداية الاستدعاء)، تستدعي `iommu_probe_device()` لإكمال عملية الربط الكاملة.

**الـ Parameters:**

| الـ Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device المراد ربطه بالـ IOMMU |
| `master_np` | `struct device_node *` | الـ DT node الخاص بالـ device (أو الـ master في حالة PCI) |
| `id` | `const u32 *` | مؤشر لـ stream ID اختياري — إذا كان `NULL` يُقرأ من property `iommus` مباشرةً، وإلا يُستخدم مع `iommu-map` |

**الـ Return Value:**

| القيمة | المعنى |
|---|---|
| `0` | نجاح — تم ربط الـ device بـ IOMMU |
| `-ENODEV` | لا يوجد IOMMU في الـ DT لهذا الـ device، أو `master_np == NULL` |
| `-EPROBE_DEFER` | الـ IOMMU driver لم يُحمَّل بعد — يجب إعادة المحاولة |
| أي errno آخر | خطأ فادح في الـ configuration |

**التفاصيل المهمة:**

- **Locking:** تأخذ `iommu_probe_device_lock` (mutex) في البداية وتحرره قبل استدعاء `iommu_probe_device()`. هذا يمنع race condition عند محاولة تكوين الـ `fwspec` من threads متعددة.
- **Idempotency:** إذا كان `dev_iommu_fwspec_get(dev)` يُعيد قيمة غير NULL (أي الـ device مُعدّ بالفعل)، تعود فوراً بـ `0` دون عمل أي شيء.
- **Cleanup on error:** إذا فشلت الـ configuration وكان `dev->iommu` موجوداً قبل الاستدعاء، تستدعي `iommu_fwspec_free()`. أما إذا أُنشئ `dev->iommu` أثناء الاستدعاء ثم فشل، تستدعي `dev_iommu_free()` لتنظيف الموارد.
- **PCI path:** تستدعي `pci_request_acs()` أولاً لضمان تفعيل ACS، ثم `of_pci_check_device_ats()` للتحقق من دعم ATS.

**من يستدعيها:**
الـ bus code أثناء الـ DMA configuration — تحديداً `of_dma_configure_id()` في `drivers/of/device.c`، والتي تُستدعى من `of_dma_configure()` التي يستدعيها كل bus driver أثناء الـ probe.

**Pseudocode Flow:**

```c
of_iommu_configure(dev, master_np, id):
    if master_np == NULL:
        return -ENODEV

    lock(iommu_probe_device_lock)
    if dev_iommu_fwspec_get(dev) != NULL:
        unlock(iommu_probe_device_lock)
        return 0  // already configured

    dev_iommu_present = (dev->iommu != NULL)

    if dev_is_pci(dev):
        pci_request_acs()
        err = pci_for_each_dma_alias(dev, of_pci_iommu_init, {dev, master_np})
        of_pci_check_device_ats(dev, master_np)
    else:
        err = of_iommu_configure_device(master_np, dev, id)

    if err:
        if dev_iommu_present:
            iommu_fwspec_free(dev)
        elif dev->iommu:
            dev_iommu_free(dev)

    unlock(iommu_probe_device_lock)

    if !err && dev->bus && !dev_iommu_present:
        err = iommu_probe_device(dev)  // trigger full IOMMU attach

    return err
```

---

### الفئة الثانية: Reserved Regions Query

هذه الفئة تُعنى بـ **استخراج مناطق الـ IOVA المحجوزة** (reserved regions) من الـ Device Tree وإضافتها إلى القائمة التي يديرها الـ IOMMU subsystem. تُستخدم من قِبَل الـ IOMMU drivers لتنفيذ الـ callback `get_resv_regions()` الخاص بكل driver.

---

#### `of_iommu_get_resv_regions()`

```c
void of_iommu_get_resv_regions(struct device *dev,
                                struct list_head *list);
```

**ما تفعله:**
تمر على جميع الـ nodes المشار إليها بـ property `memory-region` في الـ DT node الخاص بالـ device. لكل node تحاول قراءة عنوان الـ physical memory من property `reg` (اختيارية)، ثم تقرأ property `iommu-addresses` لمعرفة الـ IOVA mapping المطلوبة لهذا الـ device تحديداً. تُحدد نوع الـ reservation (direct / reserved) وتُخصص `iommu_resv_region` وتضيفها لنهاية الـ list.

الدالة تعمل فقط عند تفعيل `CONFIG_OF_ADDRESS` (محاطة بـ `#if IS_ENABLED(CONFIG_OF_ADDRESS)`).

**الـ Parameters:**

| الـ Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device الذي نستخرج الـ reservations الخاصة به |
| `list` | `struct list_head *` | القائمة التي ستُضاف إليها الـ `iommu_resv_region` المُخصصة |

**الـ Return Value:** `void` — لا تُعيد شيئاً. الأخطاء يتم تسجيلها بـ `dev_err()` / `dev_warn()` وتُتخطى (best-effort).

**التفاصيل المهمة:**

- **DMA Coherency:** إذا كان الـ device coherent (من خلال `of_dma_is_coherent()`), تُضاف `IOMMU_CACHE` لـ `prot` flags الخاصة بالـ region.
- **نوع الـ Reservation:** تستدعي `iommu_resv_region_get_type()` الداخلية:
  - إذا كان الـ physical address مطابقاً للـ IOVA تماماً → `IOMMU_RESV_DIRECT`
  - إذا لم يكن هناك physical region (property `reg` غير موجودة أو فارغة) → `IOMMU_RESV_RESERVED`
  - Mapping جزئي أو غير متطابق → `IOMMU_RESV_RESERVED` مع تحذير
- **Memory allocation:** تستخدم `iommu_alloc_resv_region()` مع `GFP_KERNEL`، مما يعني أنها **لا تُستدعى من atomic context**.
- **`iommu-addresses` property format:** تحتوي على phandle يشير لـ device node ثم الـ IOVA range. الدالة تتحقق أن الـ phandle يطابق `dev->of_node` قبل معالجة الـ entry — أي أن نفس الـ memory region يمكن أن تكون لأكثر من device، ولكل device إدخاله الخاص.
- **Zero-length guard:** إذا كان الـ length صفراً بعد `of_translate_dma_region()`، يُصدر تحذير ويُتخطى الـ entry.

**من يستدعيها:**
يستدعيها الـ IOMMU driver في تنفيذ الـ callback `iommu_ops::get_resv_regions()`. مثلاً: `arm_smmu_get_resv_regions()` في `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` تستدعي هذه الدالة كـ helper.

**Pseudocode Flow:**

```c
of_iommu_get_resv_regions(dev, list):
    // iterate over each memory-region phandle in dev->of_node
    for each node in of_for_each_phandle(dev->of_node, "memory-region"):
        phys = {0}

        if of_property_present(node, "reg"):
            err = of_address_to_resource(node, 0, &phys)
            if err: log error, continue

        maps = of_get_property(node, "iommu-addresses", &size)
        if !maps: continue

        // walk iommu-addresses entries
        while maps < end:
            phandle = read_be32(maps++)
            np = of_find_node_by_phandle(phandle)

            if np == dev->of_node:
                prot = IOMMU_READ | IOMMU_WRITE
                if of_dma_is_coherent(dev->of_node):
                    prot |= IOMMU_CACHE

                maps = of_translate_dma_region(np, maps, &iova, &length)
                if length == 0: warn, continue

                type = iommu_resv_region_get_type(dev, &phys, iova, length)
                region = iommu_alloc_resv_region(iova, length, prot, type, GFP_KERNEL)
                if region:
                    list_add_tail(&region->list, list)
```

---

### الـ Stubs عند غياب `CONFIG_OF_IOMMU`

عند عدم تفعيل `CONFIG_OF_IOMMU`، يُعرَّف كلا الـ functions كـ `static inline` stubs في نهاية الـ header:

```c
/* stub when CONFIG_OF_IOMMU is not set */

static inline int of_iommu_configure(struct device *dev,
                                     struct device_node *master_np,
                                     const u32 *id)
{
    return -ENODEV;  /* no IOMMU support at all */
}

static inline void of_iommu_get_resv_regions(struct device *dev,
                                              struct list_head *list)
{
    /* no-op */
}
```

هذا النمط يسمح لـ callers بكتابة كود خالٍ من الـ `#ifdef` — الـ `of_dma_configure()` مثلاً لا تحتاج لـ guard حول استدعائها لـ `of_iommu_configure()`.

---

### العلاقة بين الـ Functions والـ Subsystem

```
Bus probe (e.g., platform_bus)
        │
        ▼
of_dma_configure()
        │
        ▼
of_iommu_configure(dev, np, id)          ← [of_iommu.h]
        │
        ├── [non-PCI] of_iommu_configure_device()
        │       └── of_iommu_xlate()
        │               ├── iommu_fwspec_init()
        │               └── ops->of_xlate()      ← driver callback
        │
        └── [PCI] pci_for_each_dma_alias()
                └── of_pci_iommu_init()
                        └── of_iommu_configure_dev_id()
                                └── of_iommu_xlate()


IOMMU driver get_resv_regions() callback
        │
        ▼
of_iommu_get_resv_regions(dev, list)     ← [of_iommu.h]
        │
        ├── of_for_each_phandle() → memory-region nodes
        ├── of_address_to_resource() → phys address
        ├── iommu_resv_region_get_type() → DIRECT / RESERVED
        └── iommu_alloc_resv_region() → iommu_resv_region
                └── list_add_tail() → list
```

---

### جدول الـ Types والـ Structs المستخدمة

| النوع | الملف | الدور |
|---|---|---|
| `struct iommu_fwspec` | `include/linux/iommu.h` | يحمل الـ stream IDs والـ fwnode الخاص بالـ IOMMU لكل device |
| `struct iommu_resv_region` | `include/linux/iommu.h` | تمثيل منطقة IOVA محجوزة (start, length, prot, type) |
| `enum iommu_resv_type` | `include/linux/iommu.h` | `DIRECT`, `DIRECT_RELAXABLE`, `RESERVED`, `MSI`, `SW_MSI` |
| `struct of_phandle_args` | `include/linux/of.h` | حجج الـ phandle المُحلَّلة من DT (np + args[]) |
| `struct of_phandle_iterator` | `include/linux/of.h` | iterator لـ `of_for_each_phandle()` |
## Phase 5: دليل الـ Debugging الشامل

> **النطاق:** subsystem الـ `of_iommu` — الواجهة بين الـ Device Tree والـ IOMMU في Linux kernel.
> الملفات الأساسية: `include/linux/of_iommu.h` · `drivers/iommu/of_iommu.c`

---

### Software Level

---

#### 1. debugfs — المداخل والقراءة

الـ IOMMU subsystem يُصدِّر معلومات تشخيصية عبر `debugfs`. نقطة البداية:

```bash
# تأكد من mount الـ debugfs
mount -t debugfs none /sys/kernel/debug

# استعراض مجلد الـ iommu
ls /sys/kernel/debug/iommu/
```

| المدخل | المعنى | كيفية القراءة |
|---|---|---|
| `/sys/kernel/debug/iommu/` | مجلد رئيسي للـ IOMMU driver | `ls` ثم `cat` لكل ملف |
| `/sys/kernel/debug/iommu/<driver>/` | مخصص لكل driver (smmu, intel-iommu...) | يختلف بحسب الـ driver |
| `/sys/kernel/debug/iommu/domain_<N>` | معلومات الـ domain: mappings, page tables | `cat domain_<N>` |
| `/sys/kernel/debug/iommu/groups` | الـ iommu groups وأعضاؤها | `cat groups` |

```bash
# مثال: قراءة groups
cat /sys/kernel/debug/iommu/groups

# مثال: قراءة معلومات domain معين
cat /sys/kernel/debug/iommu/domain_0
```

**نقطة تشخيص خاصة بـ of_iommu:** لا يوجد debugfs entry مخصص لـ `of_iommu` تحديداً — التشخيص يتم من خلال نتائج عملية `of_iommu_configure()` على الـ device نفسه.

```bash
# التحقق من أن الـ device ارتبط بـ IOMMU group بعد of_iommu_configure
ls /sys/bus/platform/devices/<device-name>/iommu_group
# مثال حقيقي
ls /sys/bus/platform/devices/fd800000.gpu/iommu_group
```

---

#### 2. sysfs — المداخل والقراءة

```bash
# هل الـ device يملك iommu_group؟ (نجاح of_iommu_configure يعني وجوده)
readlink /sys/bus/platform/devices/<dev>/iommu_group

# اسم الـ IOMMU المرتبط
cat /sys/bus/platform/devices/<dev>/iommu_group/name

# قائمة الـ devices في نفس الـ group
ls /sys/bus/platform/devices/<dev>/iommu_group/devices/

# هل الـ device في وضع passthrough أم DMA-remapping؟
cat /sys/bus/platform/devices/<dev>/iommu_group/type

# المناطق المحجوزة (resv_regions) — ناتجة عن of_iommu_get_resv_regions
cat /sys/kernel/debug/iommu/domain_*/resv_regions 2>/dev/null

# فحص iommu-addresses من DT هل ظهرت في reserved regions
cat /sys/kernel/debug/iommu/groups
```

| المدخل | المعنى |
|---|---|
| `iommu_group/` symlink موجود | `of_iommu_configure()` نجح |
| `iommu_group/` symlink غائب | فشل التهيئة أو `CONFIG_OF_IOMMU` غير مفعّل |
| `iommu_group/type` = `DMA` | وضع الـ DMA remapping الكامل |
| `iommu_group/type` = `identity` | وضع الـ passthrough |

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# تفعيل كل events الـ iommu
echo 1 > /sys/kernel/debug/tracing/events/iommu/enable

# أو تفعيل events بعينها
echo 1 > /sys/kernel/debug/tracing/events/iommu/map/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/unmap/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/io_page_fault/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/attach_device_to_domain/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/detach_device_from_domain/enable

# تشغيل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# قراءة النتائج
cat /sys/kernel/debug/tracing/trace

# تصفية بـ device معين (مثلاً)
echo 'name == "fd800000.gpu"' > /sys/kernel/debug/tracing/events/iommu/attach_device_to_domain/filter
```

**الـ events الأهم لتشخيص `of_iommu`:**

| Event | متى يُطلق | ما يكشفه |
|---|---|---|
| `attach_device_to_domain` | بعد نجاح `of_iommu_configure` | تأكيد ارتباط الـ device |
| `io_page_fault` | عند خطأ في الـ IOVA | عنوان الخطأ + الـ device |
| `map` / `unmap` | عند إنشاء/إزالة mapping | IOVA + PA + size |

---

#### 4. printk / dynamic debug

```bash
# تفعيل dynamic debug لـ of_iommu.c بالكامل
echo 'file drivers/iommu/of_iommu.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لكل ملفات الـ iommu
echo 'file drivers/iommu/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع timestamps و module name
echo 'file drivers/iommu/of_iommu.c +pmfl' > /sys/kernel/debug/dynamic_debug/control
# p = printk, m = module name, f = function name, l = line number

# مشاهدة النتائج
dmesg -w | grep -i iommu

# تفعيل loglevel لكامل الـ iommu subsystem
echo 8 > /proc/sys/kernel/printk
```

الرسالة الأساسية في `of_iommu.c` التي تظهر عند الفشل:

```
<device>: Adding to IOMMU failed: <errno>
```

هذه تظهر فقط في `dev_dbg()` — تحتاج dynamic debug لرؤيتها.

---

#### 5. Kernel Config Options للـ Debugging

| الخيار | الوظيفة |
|---|---|
| `CONFIG_OF_IOMMU` | تفعيل الـ subsystem نفسه — يجب أن يكون `y` |
| `CONFIG_IOMMU_DEBUG` | تفعيل مسارات debug إضافية في الـ IOMMU core |
| `CONFIG_IOMMU_DEBUGFS` | تفعيل مداخل الـ debugfs للـ IOMMU |
| `CONFIG_IOMMU_DEFAULT_DMA_STRICT` | فرض strict DMA — يكشف أخطاء الـ mapping |
| `CONFIG_IOMMU_DMA` | دعم الـ DMA API فوق الـ IOMMU |
| `CONFIG_OF_ADDRESS` | مطلوب لـ `of_iommu_get_resv_regions()` |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `dev_dbg()` في runtime |
| `CONFIG_DEBUG_SHIRQ` | لكشف مشاكل الـ interrupt sharing مع الـ IOMMU |

```bash
# التحقق من الـ config الحالية
zcat /proc/config.gz | grep -E 'IOMMU|OF_IOMMU|OF_ADDRESS'
# أو
grep -E 'IOMMU|OF_IOMMU' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# iommufd - الواجهة الجديدة
ls /dev/iommu  # يظهر إذا كان CONFIG_IOMMUFD مفعّلاً

# فحص iommu-map في DT باستخدام dtc
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "iommu-map"

# فحص iommus property
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "iommus"

# أداة lsiommu (إذا متوفرة في distro)
lsiommu -v 2>/dev/null || echo "not available"

# استخدام iommu_group عبر sysfs لفحص الـ mappings
for g in /sys/kernel/iommu_groups/*/; do
    echo "=== Group $g ==="
    ls $g/devices/
done
```

---

#### 7. جدول أخطاء شائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `Adding to IOMMU failed: -19` (`ENODEV`) | لم يُعثر على IOMMU في DT أو الـ driver غير محمّل | تحقق من `iommus =` في DT وتحميل الـ IOMMU driver |
| `Adding to IOMMU failed: -517` (`EPROBE_DEFER`) | الـ IOMMU driver لم يُحمَّل بعد | انتظر أو أعد ترتيب الـ initcalls |
| `failed to parse memory region <node>: <err>` | `iommu-addresses` في DT تشير لـ node بـ `reg` خاطئ | صحح الـ `reg` في DT لمنطقة الـ reserved-memory |
| `Cannot reserve IOVA region of 0 size` | الـ `iommu-addresses` بـ length = 0 | تحقق من صحة الـ address-cells وترجمة DMA |
| `treating non-direct mapping ... as reservation` | IOVA لا تطابق PA — يُعامَل كـ reservation | تحقق من قصد الـ DT: هل يُراد direct mapping؟ |
| `iommu_fwspec_init failed` | فشل تهيئة الـ fwspec للـ device | تحقق من الـ memory allocation وحالة الـ device |
| `of_xlate failed: -22` (`EINVAL`) | الـ IOMMU driver رفض الـ phandle args | تحقق من `#iommu-cells` وعدد الـ args في DT |
| `ats-supported property found but...` | ATS مفعّل لكن الـ hardware لا يدعمه | أزل `ats-supported` من DT |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

في `drivers/iommu/of_iommu.c` — النقاط المثلى للتشخيص:

```c
/* نقطة 1: التحقق من نجاح xlate */
static int of_iommu_xlate(struct device *dev,
                          struct of_phandle_args *iommu_spec)
{
    const struct iommu_ops *ops;
    int ret;

    if (!of_device_is_available(iommu_spec->np)) {
        dev_warn(dev, "IOMMU node %pOF not available\n", iommu_spec->np);
        dump_stack(); /* هنا لمعرفة من استدعى xlate */
        return -ENODEV;
    }
    /* ... */
    WARN_ON(!ops->of_xlate); /* تحذير إذا الـ driver لا يدعم xlate */
}

/* نقطة 2: بعد of_iommu_configure لفحص الـ fwspec */
int of_iommu_configure(struct device *dev, ...)
{
    /* ... */
    WARN_ON(err && err != -ENODEV && err != -EPROBE_DEFER);
    /* أي خطأ غير متوقع يستحق stack trace */
}

/* نقطة 3: في of_iommu_get_resv_regions لفحص الـ regions */
if (length == 0) {
    WARN_ONCE(1, "%s: zero-length IOVA region in DT\n",
              dev_name(dev));
}
```

---

### Hardware Level

---

#### 1. التحقق من توافق حالة الـ Hardware مع الـ Kernel

الـ `of_iommu` يعتمد على DT لوصف الارتباط بين الـ device والـ IOMMU. التحقق:

```bash
# 1. تأكد أن الـ IOMMU موصوف في DT
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B2 -A10 "iommu-cells"

# 2. تأكد أن الـ device يملك iommus property
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B5 -A5 "iommus = "

# 3. تأكد أن الـ SMMU/IOMMU مفعّل في الـ hardware
# مثال لـ ARM SMMU — تحقق من registers
devmem2 0xfd800000 w  # عنوان SMMU base من DT

# 4. مقارنة الـ iommu_group في kernel مع توصيف الـ hardware
cat /sys/kernel/iommu_groups/0/devices  # ماذا يرى الـ kernel
# يجب أن يطابق ما في DT
```

---

#### 2. Register Dump — devmem2 / /dev/mem

```bash
# تثبيت devmem2
apt-get install devmem2  # Debian/Ubuntu
# أو
yum install devmem2      # RHEL/Fedora

# مثال: ARM SMMU-v3 — قراءة SMMU_IDR0 (offset 0x0)
SMMU_BASE=0xfd800000  # من DT: reg = <0x0 0xfd800000 0x0 0x20000>
devmem2 $SMMU_BASE w   # قراءة IDR0

# قراءة SMMU_CR0 (offset 0x20) — هل الـ SMMU مفعّل؟
devmem2 $((SMMU_BASE + 0x20)) w

# استخدام /dev/mem مباشرة (يحتاج CONFIG_STRICT_DEVMEM=n)
dd if=/dev/mem bs=4 count=1 skip=$((SMMU_BASE / 4)) 2>/dev/null | xxd

# استخدام io utility
io -4 -r $SMMU_BASE  # إذا متوفر
```

**Registers مهمة في ARM SMMU-v3:**

| Register | Offset | ما يكشفه |
|---|---|---|
| `SMMU_IDR0` | 0x000 | قدرات الـ hardware |
| `SMMU_CR0` | 0x020 | هل الـ SMMU enabled |
| `SMMU_CR0ACK` | 0x024 | تأكيد الـ enable |
| `SMMU_GBPA` | 0x044 | Global Bypass Attribute |
| `SMMU_GERROR` | 0x060 | Global errors |

---

#### 3. Logic Analyzer / Oscilloscope

عند تشخيص مشاكل الـ IOMMU على مستوى الـ bus:

```
الحافلة: AXI / ACE-Lite (ARM) أو PCIe TLP
الإشارات المهمة للرصد:
  - AWADDR / ARADDR  → عنوان الـ DMA request
  - AWPROT / ARPROT  → صلاحيات الـ access
  - BRESP / RRESP    → استجابة الـ SMMU (SLVERR = fault)

نصائح:
  1. ضع trigger على BRESP = 2'b10 (SLVERR) لالتقاط الـ fault
  2. راقب عنوان الـ IOVA في AWADDR — قارنه مع الـ kernel mappings
  3. تحقق من أن الـ AxUSER bits تحمل StreamID الصحيح
     (يطابق ما يُعطى لـ of_iommu_configure عبر الـ id parameter)
```

```
ASCII: مسار الـ DMA Request عبر SMMU
┌──────────┐   IOVA    ┌──────────┐   PA     ┌──────────┐
│  Device  │ ────────► │  SMMU   │ ────────► │  Memory  │
│ (master) │           │ (xlate) │           │          │
└──────────┘           └────┬─────┘           └──────────┘
                            │ fault?
                            ▼
                      ┌──────────┐
                      │  CPU     │
                      │ (irq)    │
                      └──────────┘
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Kernel Log | التشخيص |
|---|---|---|
| StreamID خاطئ | `iommu fault: unhandled context fault` | تحقق من `id` parameter في `of_iommu_configure` |
| الـ SMMU غير مفعّل في الـ hardware | لا يظهر أي SMMU device في `dmesg` | تحقق من `SMMU_CR0` register |
| الـ SMMU لم يُبرمَج بالـ STE | `arm-smmu-v3: event=0x10` (Translation fault) | تحقق من `iommu-map` في DT |
| reserved-memory غير مضبوطة | `treating non-direct mapping as reservation` | صحح `iommu-addresses` في DT |
| مشكلة TLB shootdown | `arm-smmu: TLB sync timeout` | مشكلة hardware في الـ SMMU TLB |
| ATS غير مدعوم | PCIe error في الـ log | أزل `ats-supported` من DT |

---

#### 5. Device Tree Debugging — التحقق من التوافق

```bash
# 1. تفريغ الـ DT الفعلي المُحمَّل في الـ kernel
dtc -I fs /sys/firmware/devicetree/base -O dts -o /tmp/live.dts 2>/dev/null

# 2. البحث عن كل الـ devices التي تملك iommus property
grep -B10 "iommus = " /tmp/live.dts | grep -E "node-name|iommus"

# 3. التحقق من iommu-map للـ PCI
grep -A5 "iommu-map" /tmp/live.dts

# 4. التحقق من iommu-map-mask
grep "iommu-map-mask" /tmp/live.dts

# 5. التحقق من #iommu-cells في الـ IOMMU node
grep -A3 "iommu-controller" /tmp/live.dts | grep "iommu-cells"

# 6. تحقق من reserved-memory وiommu-addresses
grep -B5 -A10 "iommu-addresses" /tmp/live.dts
```

**مثال DT صحيح:**

```dts
/* الـ IOMMU node */
smmu: iommu@fd800000 {
    compatible = "arm,smmu-v3";
    reg = <0x0 0xfd800000 0x0 0x20000>;
    #iommu-cells = <1>;   /* مطلوب لـ of_iommu_xlate */
};

/* الـ device المرتبط */
gpu@fc000000 {
    compatible = "vendor,gpu";
    reg = <0x0 0xfc000000 0x0 0x10000>;
    iommus = <&smmu 0x200>;  /* StreamID = 0x200 */
    memory-region = <&gpu_reserved>;
};

/* منطقة محجوزة */
reserved-memory {
    gpu_reserved: gpu-reserved@80000000 {
        reg = <0x0 0x80000000 0x0 0x1000000>;
        iommu-addresses = <&gpu 0x0 0x80000000 0x0 0x1000000>;
    };
};
```

```bash
# تحقق من تطابق #iommu-cells مع عدد args
# إذا #iommu-cells = <1> فيجب أن يكون: iommus = <&smmu ONE_ARG>
python3 -c "
import struct, sys
# قراءة iommu-cells من sysfs
with open('/sys/firmware/devicetree/base/iommu@fd800000/#iommu-cells','rb') as f:
    val = struct.unpack('>I', f.read(4))[0]
    print(f'#iommu-cells = {val}')
"
```

---

### Practical Commands

---

#### الأوامر الجاهزة للنسخ

**فحص شامل سريع:**

```bash
#!/bin/bash
# of_iommu_debug.sh — فحص شامل لـ of_iommu subsystem

echo "=== IOMMU Groups ==="
for g in /sys/kernel/iommu_groups/*/; do
    echo "Group $(basename $g):"
    ls $g/devices/ 2>/dev/null | sed 's/^/  /'
done

echo ""
echo "=== Devices with IOMMU ==="
for dev in /sys/bus/platform/devices/*/; do
    if [ -L "$dev/iommu_group" ]; then
        group=$(basename $(readlink "$dev/iommu_group"))
        echo "  $(basename $dev) → $group"
    fi
done

echo ""
echo "=== IOMMU in dmesg ==="
dmesg | grep -iE 'iommu|smmu|of_iommu' | tail -30

echo ""
echo "=== DT iommus properties ==="
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -E "iommus|iommu-map|iommu-cells|iommu-addresses" | \
    sort -u
```

**تفعيل الـ Tracing الكامل:**

```bash
#!/bin/bash
# iommu_trace.sh — تفعيل كامل الـ iommu tracing

cd /sys/kernel/debug/tracing

# تنظيف
echo > trace

# تفعيل events
echo 1 > events/iommu/enable

# تفعيل dynamic debug
echo 'file drivers/iommu/of_iommu.c +pmfl' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/iommu/*.c +pmfl' > /sys/kernel/debug/dynamic_debug/control

# تشغيل
echo 1 > tracing_on
echo "Tracing started. Press Enter to stop and view..."
read
echo 0 > tracing_on

cat trace | grep -E "iommu|smmu" | head -100
```

**قراءة reserved regions:**

```bash
# فحص الـ resv_regions المُسجَّلة لكل device
for dev in /sys/bus/platform/devices/*/; do
    name=$(basename $dev)
    if [ -L "$dev/iommu_group" ]; then
        echo "=== $name ==="
        cat $dev/iommu_group/reserved_regions 2>/dev/null || \
            echo "  (no reserved_regions sysfs entry)"
    fi
done
```

---

#### مثال على Output وكيفية تفسيره

**مثال: دmesg عند نجاح of_iommu_configure:**

```
[    2.345678] arm-smmu-v3 fd800000.iommu: probing hardware configuration...
[    2.345900] arm-smmu-v3 fd800000.iommu: SMMU initialised
[    3.102345] gpu fc000000.gpu: Adding to iommu group 0
```

التفسير: الـ `of_iommu_configure()` نجحت — الـ GPU ارتبط بـ IOMMU group 0.

**مثال: دmesg عند فشل EPROBE_DEFER:**

```
[    3.100000] gpu fc000000.gpu: Adding to IOMMU failed: -517
[    5.200000] arm-smmu-v3 fd800000.iommu: probing hardware configuration...
[    5.201000] gpu fc000000.gpu: Adding to iommu group 0
```

التفسير: الـ GPU جرّب قبل الـ SMMU — الـ kernel أعاد المحاولة تلقائياً بعد تحميل الـ SMMU.

**مثال: دmesg عند خطأ في DT (IOVA size = 0):**

```
[    3.100000] gpu fc000000.gpu: Cannot reserve IOVA region of 0 size
```

التفسير: `iommu-addresses` في DT تحتوي length = 0 — راجع الـ address-cells وصحة الـ DT encoding.

**مثال: ftrace output لـ io_page_fault:**

```
    kworker/0:1-45    [000] ....  3.100000: io_page_fault: \
        iommu=fd800000.iommu type=1 device=fc000000.gpu \
        iova=0xdeadbeef flags=0x0
```

التفسير: الـ GPU حاول الوصول لعنوان IOVA `0xdeadbeef` غير المُعيَّن — احتمال bug في الـ driver أو mapping غير مكتمل.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: GPU لا يرسم — بوابة صناعية على RK3562

#### العنوان
**الـ Mali GPU يفشل في تهيئة الـ DMA** على بوابة صناعية مبنية على RK3562

#### السياق
شركة تصنع بوابات صناعية (Industrial Gateway) تعمل بنظام Linux 6.6 على RK3562. الجهاز يضم Mali-G52 GPU مدمجاً. أثناء bring-up جديد، يتعطل driver الـ Mali عند التحميل ولا تظهر أي شاشة.

#### المشكلة
عند تحميل `mali_kbase.ko` يطبع kernel:

```
mali: probe of ff400000.gpu failed with error -19
```

الـ `-19` هو `ENODEV` — وهو بالضبط ما تُعيده الدالة `of_iommu_configure` في حالة `CONFIG_OF_IOMMU` غير معرّف.

#### التحليل
الدالة في `of_iommu.h` عند غياب `CONFIG_OF_IOMMU`:

```c
static inline int of_iommu_configure(struct device *dev,
                                     struct device_node *master_np,
                                     const u32 *id)
{
    return -ENODEV;  /* لا يوجد IOMMU مهيّأ من DT */
}
```

سلسلة الاستدعاء:

```
mali_kbase_probe()
  └─> of_dma_configure()          [drivers/of/device.c]
        └─> of_iommu_configure()  [include/linux/of_iommu.h]
              └─> returns -ENODEV (stub لأن CONFIG_OF_IOMMU=n)
```

الـ `of_dma_configure()` تعتبر `ENODEV` خطأ فادحاً وتُفشل الـ probe.

#### الحل
**أولاً:** تفعيل `CONFIG_OF_IOMMU=y` و `CONFIG_IOMMU_SUPPORT=y` في kernel config:

```bash
# في .config
CONFIG_IOMMU_SUPPORT=y
CONFIG_OF_IOMMU=y
CONFIG_ROCKCHIP_IOMMU=y
```

**ثانياً:** التحقق من DT — يجب أن يحتوي node الـ GPU على `iommus` phandle:

```dts
/* arch/arm64/boot/dts/rockchip/rk3562.dtsi */
gpu: gpu@ff400000 {
    compatible = "arm,mali-bifrost";
    reg = <0x0 0xff400000 0x0 0x4000>;
    iommus = <&gpu_mmu>;          /* هذا السطر ضروري */
    ...
};

gpu_mmu: iommu@ff408000 {
    compatible = "rockchip,iommu";
    reg = <0x0 0xff408000 0x0 0x100>;
    interrupts = <GIC_SPI 159 IRQ_TYPE_LEVEL_HIGH>;
    #iommu-cells = <0>;
};
```

**ثالثاً:** إعادة البناء والتحقق:

```bash
make ARCH=arm64 menuconfig   # تفعيل CONFIG_OF_IOMMU
make ARCH=arm64 dtbs          # إعادة بناء DTB
cat /sys/kernel/debug/iommu/devices  # التحقق بعد البوت
```

#### الدرس المستفاد
الـ `of_iommu.h` يوفر stub يُعيد `ENODEV` عند غياب `CONFIG_OF_IOMMU`. هذا السلوك الصامت يُضلل المهندس ليعتقد أن المشكلة في driver الـ GPU، بينما الجذر هو غياب CONFIG في kernel.

---

### السيناريو الثاني: USB Host يُسبب kernel panic — لوحة مبنية على i.MX8MP

#### العنوان
**الـ `of_iommu_get_resv_regions` تُعيد منطقة غير صحيحة** وتُسبب تعارضاً في DMA على i.MX8MP

#### السياق
منتج Android TV Box يعمل على NXP i.MX8M Plus. الـ USB3 Host يعمل مع محرك أقراص USB. بعد تحديث kernel من 5.15 إلى 6.1، يظهر kernel panic عشوائي عند نقل ملفات كبيرة.

#### المشكلة
```
BUG: IOMMU: DMA address collision in reserved region
[  42.318] iommu fault: addr=0x80000000 not in allowed window
```

#### التحليل
الدالة في `of_iommu.h`:

```c
extern void of_iommu_get_resv_regions(struct device *dev,
                                      struct list_head *list);
```

هذه الدالة (المُعرَّفة في `drivers/of/of_iommu.c`) تقرأ الـ DT وتبني قائمة من `iommu_resv_region` تصف نطاقات العناوين المحجوزة التي لا يجب للـ IOMMU أن يماپها للـ DMA.

سلسلة الاستدعاء عند تسجيل الـ device:

```
usb_add_hcd()
  └─> device_add()
        └─> iommu_probe_device()
              └─> iommu_get_resv_regions(dev, &list)
                    └─> of_iommu_get_resv_regions(dev, list)
                          └─> يقرأ "reserved-memory" من DT
                                └─> يُضيف iommu_resv_region للقائمة
```

المشكلة: في kernel 6.1 تغيّر تفسير `no-map` في DT — منطقة كانت مُستثناة أصبحت مُدرجة كـ `IOMMU_RESV_DIRECT` فتعارضت مع buffer الـ USB DMA.

#### الحل
**فحص المناطق المحجوزة الفعلية:**

```bash
# على الجهاز المُشغَّل
cat /sys/kernel/debug/iommu/reserved_regions
# أو
dmesg | grep -i "resv\|reserved\|iommu"
```

**تصحيح DT لتحديد المنطقة بدقة:**

```dts
/* arch/arm64/boot/dts/freescale/imx8mp.dtsi */
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* منطقة firmware — محجوزة ولا يُمسها IOMMU */
    fw_reserved: firmware@80000000 {
        reg = <0 0x80000000 0 0x01000000>;
        no-map;
    };
};

&usb3_0 {
    iommus = <&iommu1 0x900>;
    /* لا تضع USB DMA buffer داخل نطاق fw_reserved */
    dma-ranges = <0x40000000 0x40000000 0xbf000000>;
};
```

**كود التحقق في driver لفهم القائمة المُبنية:**

```c
/* debug مؤقت في driver */
struct iommu_resv_region *region;
LIST_HEAD(resv_list);

of_iommu_get_resv_regions(dev, &resv_list);

list_for_each_entry(region, &resv_list, list) {
    dev_info(dev, "resv: start=0x%llx len=0x%zx type=%d\n",
             (u64)region->start, region->length, region->type);
}

iommu_put_resv_regions(dev, &resv_list);
```

#### الدرس المستفاد
**الـ `of_iommu_get_resv_regions`** ليست بسيطة — تُترجم `reserved-memory` nodes من DT إلى قيود IOMMU. أي تغيير في تفسير الـ DT بين إصدارات kernel يُغيّر ما تُضيفه هذه الدالة وقد يُفسد DMA بالكامل.

---

### السيناريو الثالث: حساس IoT لا يُرسل بيانات — STM32MP1 بدون IOMMU

#### العنوان
**فهم سلوك الـ stub** على منصة STM32MP1 التي لا تحتوي IOMMU

#### السياق
جهاز IoT sensor صغير يعمل على STM32MP157 (Cortex-A7 + Cortex-M4). Driver الـ SPI يستخدم DMA لنقل بيانات الحساسات. المطوّر يرى في log:

```
spi-stm32 4000b000.spi: failed to configure iommu: -19
```

ويعتقد أن هناك خطأ حقيقي.

#### المشكلة
المطوّر يجهل الفرق بين "لا يوجد IOMMU" و"فشل IOMMU". الرسالة مُضللة.

#### التحليل
على STM32MP1 لا يوجد IOMMU hardware. لذلك `CONFIG_OF_IOMMU=n` والـ stub يُستخدم:

```c
/* المسار المُفعَّل على STM32MP1 — لأن CONFIG_OF_IOMMU غير معرّف */
static inline int of_iommu_configure(struct device *dev,
                                     struct device_node *master_np,
                                     const u32 *id)
{
    return -ENODEV;  /* طبيعي تماماً — لا يوجد IOMMU */
}
```

الـ `of_dma_configure()` في `drivers/of/device.c` تتعامل مع `ENODEV` بشكل خاص:

```c
/* من drivers/of/device.c */
ret = of_iommu_configure(dev, np, id);
if (ret == -ENODEV) {
    /*
     * -ENODEV means no IOMMU configured — not an error.
     * Continue with direct DMA (physical == virtual addresses).
     */
    ret = 0;
}
```

إذن الـ `-ENODEV` لا يُوقف الـ probe — هو مسار طبيعي على منصات بدون IOMMU.

الرسالة الخاطئة صادرة من كود في driver الـ SPI نفسه الذي يُسجّلها بمستوى خطأ بدلاً من debug.

#### الحل
**التحقق السريع أن الـ SPI يعمل رغم الرسالة:**

```bash
# على الجهاز
ls /sys/bus/spi/devices/          # هل يظهر spi device؟
cat /sys/bus/spi/devices/spi0.0/modalias
# تشغيل اختبار بسيط
spidev_test -D /dev/spidev0.0 -s 1000000 -p "Hello"
```

**إسكات الرسالة غير المفيدة في driver SPI بتغيير مستوى الـ log:**

```c
/* في drivers/spi/spi-stm32.c */
ret = of_iommu_configure(&spi->dev, np, NULL);
if (ret && ret != -ENODEV) {
    /* خطأ حقيقي */
    dev_err(&spi->dev, "failed to configure iommu: %d\n", ret);
    return ret;
} else if (ret == -ENODEV) {
    /* طبيعي — المنصة بدون IOMMU */
    dev_dbg(&spi->dev, "no iommu, using direct DMA\n");
}
```

**للتمييز بين الحالتين في DT:**

```dts
/* STM32MP1 — لا توجد iommus property = لا IOMMU = -ENODEV طبيعي */
spi: spi@4000b000 {
    compatible = "st,stm32h7-spi";
    reg = <0x4000b000 0x400>;
    dmas = <&dmamux1 35 0x400 0x01>,
           <&dmamux1 36 0x400 0x01>;
    /* بدون سطر iommus — of_iommu_configure ستُعيد ENODEV */
};
```

#### الدرس المستفاد
الـ stub في `of_iommu.h` يُعيد `ENODEV` كـ "لا يوجد IOMMU" لا كـ "فشل". المهندس يجب أن يميّز هذه الحالة في أي driver يستخدم `of_iommu_configure` بشكل مباشر وألا يُسجّلها كخطأ.

---

### السيناريو الرابع: Display Engine يُفسد الذاكرة — Allwinner H616 في صندوق TV

#### العنوان
**الـ `of_iommu_configure` تُخفق في ربط DE2 بالـ IOMMU** وتُسبب تلوث ذاكرة على Allwinner H616

#### السياق
صندوق Android TV يعمل على Allwinner H616 (Orange Pi Zero 2). الـ Display Engine 2 (DE2) يرسم الواجهة عبر HDMI. بعد تشغيل فيديو 4K لعدة دقائق، تتجمد الشاشة وتظهر artifacts.

#### المشكلة
```
iommu: DMA map for sun50i-de2 overlaps reserved region
iommu: Failed to map PA=0x5c000000 in domain
```

#### التحليل
سلسلة التهيئة عند boot:

```
sun50i_de2_probe()
  └─> of_dma_configure(dev, np, true)   [drivers/of/device.c]
        └─> of_iommu_configure(dev, np, NULL)
              └─> يقرأ "iommus" من DT لـ DE2 node
              └─> يربط الجهاز بـ sun50i_iommu driver
              └─> يُعيد 0 عند النجاح
```

```c
/* العملية الكاملة عند CONFIG_OF_IOMMU=y */
extern int of_iommu_configure(struct device *dev,
                               struct device_node *master_np,
                               const u32 *id);
/*
 * تقرأ "iommus" phandle من master_np
 * تستدعي iommu_ops->of_xlate لتحويل phandle إلى IOMMU SID
 * تربط dev بـ iommu_group المناسب
 */
```

المشكلة الفعلية: DT يُعرّف `iommus` لـ DE2 لكن `reserved-memory` للـ firmware تتداخل مع نطاق DMA الذي يستخدمه DE2. `of_iommu_get_resv_regions` لم تُعرّف هذه المنطقة كـ `IOMMU_RESV_DIRECT` فحاولت DE2 الكتابة فيها.

**فحص MIS-map:**

```bash
# قراءة regions المحجوزة
cat /proc/iomem | grep -i reserved
# فحص iommu groups
ls /sys/kernel/iommu_groups/
cat /sys/kernel/iommu_groups/*/devices
# تتبع allocations
cat /sys/kernel/debug/dma-api/dump
```

#### الحل
**تصحيح DT لـ H616 — تعريف المنطقة المحجوزة صراحةً:**

```dts
/* arch/arm64/boot/dts/allwinner/sun50i-h616.dtsi */
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    /* منطقة محجوزة للـ secure firmware — يجب أن يعرفها IOMMU */
    secmon_reserved: secmon@40000000 {
        reg = <0x0 0x40000000 0x0 0x01000000>;
        no-map;
    };
};

display_engine: display-engine@1000000 {
    compatible = "allwinner,sun50i-h616-display-engine";
    iommus = <&iommu 0>;
    /* تحديد نطاق DMA يتجنب منطقة firmware */
    dma-ranges = <0x0 0x42000000 0x0 0x3e000000>;
};
```

**إضافة `IOMMU_RESV_DIRECT` يدوياً عبر DT:**

```dts
iommu: iommu@30f0000 {
    compatible = "allwinner,sun50i-h616-iommu";
    reg = <0x030f0000 0x10000>;
    #iommu-cells = <1>;
    /* هذا يجعل of_iommu_get_resv_regions تُضيف المنطقة للقائمة */
};
```

#### الدرس المستفاد
**الـ `of_iommu_configure` و `of_iommu_get_resv_regions` يعملان معاً**: الأولى تربط الجهاز بالـ IOMMU، والثانية تُحدد ما هو محمي. إغفال أي منهما يُفضي إلى تلوث الذاكرة الذي يصعب تشخيصه.

---

### السيناريو الخامس: ECU سيارة — AM62x يرفض تحميل driver الـ CAN

#### العنوان
**`CONFIG_OF_IOMMU` غير مُفعَّل في BSP مخصص** يمنع driver الـ CAN من الارتباط بـ IOMMU على AM62x

#### السياق
وحدة تحكم إلكترونية (ECU) في سيارة تعمل على Texas Instruments AM62x (Cortex-A53). النظام يشغّل Linux PREEMPT_RT للتحكم في ناقل CAN. المورّد قدّم BSP مخصصاً بـ kernel مُقيَّد. عند تحميل `can-j1939.ko`، الـ DMA transfers تفشل بشكل متقطع.

#### المشكلة
```
mcan: DMA transfer timeout on can0
mcan: iommu not configured, falling back to coherent allocation
```

النظام يعمل لكن latency الـ CAN تتجاوز المتطلبات الزمنية الصارمة بسبب الـ fallback إلى `dma_alloc_coherent` بدلاً من الـ streaming DMA عبر IOMMU.

#### التحليل
BSP المورّد يضبط:

```bash
# في .config المُقدَّم من المورّد
CONFIG_OF_IOMMU=n      # مُعطَّل لتبسيط الـ build
CONFIG_IOMMU_SUPPORT=n
```

لذلك يُستخدم الـ stub:

```c
/* المسار الفعلي بسبب CONFIG_OF_IOMMU=n */
static inline int of_iommu_configure(struct device *dev,
                                     struct device_node *master_np,
                                     const u32 *id)
{
    return -ENODEV;  /* لا IOMMU — DMA مباشر بدون protection */
}

static inline void of_iommu_get_resv_regions(struct device *dev,
                                              struct list_head *list)
{
    /* لا شيء — القائمة تبقى فارغة */
}
```

الـ MCAN driver يُلاحظ غياب IOMMU وينتقل لاستخدام `consistent memory` الأبطأ، ما يُسبب تجاوز نافذة الـ 1ms المطلوبة في J1939.

**قياس الـ latency:**

```bash
# على الجهاز المُشغَّل
cyclictest -p 90 -t1 -n -i 1000 -l 10000
# فحص iommu status
dmesg | grep iommu
cat /sys/class/iommu/*/enabled 2>/dev/null || echo "no iommu sysfs"
```

#### الحل
**الخطوة 1:** إعادة تفعيل IOMMU في kernel config:

```bash
# إضافة إلى kernel .config
CONFIG_IOMMU_SUPPORT=y
CONFIG_OF_IOMMU=y
CONFIG_TI_K3_IOMMU=y        # IOMMU driver لـ AM62x
CONFIG_IOMMU_DMA=y
```

**الخطوة 2:** التحقق من DT لـ MCAN node:

```dts
/* arch/arm64/boot/dts/ti/k3-am625-sk.dts */
&main_mcan0 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&main_mcan0_pins_default>;
    phys = <&transceiver0>;
    /* ربط MCAN بـ IOMMU للسماح بـ streaming DMA */
    iommus = <&main_iommu0 0x0c40>;
};
```

**الخطوة 3:** التحقق بعد التطبيق:

```bash
# التحقق أن of_iommu_configure نجحت
dmesg | grep -E "mcan|iommu" | head -20
# يجب أن يظهر:
# iommu: Adding device 2701000.can to group 5

# قياس latency بعد التصحيح
./can_timing_test --iface can0 --period 1ms --count 1000
```

**الخطوة 4:** إذا كان تفعيل IOMMU كاملاً غير ممكن (قيود BSP)، استخدام `iommu=pt` للـ passthrough كحل وسط:

```bash
# في bootargs — IOMMU passthrough يُبقي الـ API يعمل بدون translation overhead
BOOT_ARGS="console=ttyS2,115200 iommu=pt"
```

#### الدرس المستفاد
على أنظمة real-time كـ ECU، الانتقال الصامت من IOMMU streaming DMA إلى coherent memory (بسبب `of_iommu_configure` تُعيد `ENODEV`) يُكسر المتطلبات الزمنية. **الـ `CONFIG_OF_IOMMU`** يجب أن يكون جزءاً من checklist الـ real-time BSP، وليس مجرد feature اختياري.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net هو المرجع الأول لمتابعة تطور kernel Linux، وفيما يلي أهم المقالات المتعلقة بـ `of_iommu` والـ IOMMU بشكل عام:

| المقال | الوصف |
|--------|-------|
| [ARM: DMA-mapping & IOMMU integration](https://lwn.net/Articles/444700/) | التكامل الأولي بين DMA mapping وطبقة IOMMU على ARM — مهم لفهم أصول `of_iommu` |
| [Exynos SYSMMU: DT and DMA-mapping subsystem](https://lwn.net/Articles/607626/) | سلسلة patches تُظهر كيف يُستخدم device tree لربط IOMMU بالأجهزة على Exynos SoC |
| [Add NVIDIA Tegra124 IOMMU support](https://lwn.net/Articles/603810/) | مثال عملي على generic DT bindings للـ IOMMU في Tegra — يُوضح كيف تعمل `iommus` property |
| [Apple M1 DART IOMMU driver](https://lwn.net/Articles/849969/) | driver حديث يعتمد على `of_iommu` و DT bindings لـ IOMMU مخصص |
| [Introduce /dev/iommu for userspace I/O address space management](https://lwn.net/Articles/869818/) | مقترح IOMMUFD — يُعيد تصميم واجهة IOMMU للمستخدم، خلفية مهمة |
| [Linux RISC-V IOMMU Support](https://lwn.net/Articles/972035/) | أحدث تطبيق لـ IOMMU مع DT bindings على معمارية RISC-V |
| [KS2012: ARM: DMA mapping](https://lwn.net/Articles/513939/) | ملخص نقاش kernel summit حول دمج IOMMU مع DMA mapping على ARM |
| [A new DMA-mapping API](https://lwn.net/Articles/1020437/) | اقتراح API جديد يُحسّن مسار IOMMU في DMA mapping |
| [mm: iommu: An API to unify IOMMU, CPU and device memory management](https://lwn.net/Articles/395142/) | مقال مبكر يناقش توحيد إدارة الذاكرة عبر IOMMU |

---

### التوثيق الرسمي في kernel (`Documentation/`)

```
Documentation/core-api/dma-api.rst          # واجهة DMA mapping الكاملة
Documentation/core-api/dma-api-howto.rst    # دليل عملي لاستخدام DMA API
Documentation/driver-api/iommu.rst          # توثيق IOMMU subsystem
Documentation/driver-api/iommufd.rst        # IOMMUFD — الواجهة الحديثة
Documentation/devicetree/bindings/iommu/    # DT bindings لمختلف IOMMU controllers
Documentation/arch/x86/iommu.rst            # دعم IOMMU على x86 (Intel VT-d / AMD-Vi)
```

الـ bindings المباشرة ذات الصلة:

```
Documentation/devicetree/bindings/iommu/iommu.txt
Documentation/devicetree/bindings/iommu/arm,smmu.yaml
Documentation/devicetree/bindings/iommu/arm,smmu-v3.yaml
```

---

### ملفات المصدر الأساسية في kernel

الملفات التي يجب قراءتها لفهم `of_iommu` بشكل كامل:

```
include/linux/of_iommu.h       # الملف الرئيسي — الواجهة العامة
drivers/iommu/of_iommu.c       # التنفيذ الكامل لـ of_iommu_configure()
drivers/iommu/iommu.c          # IOMMU core — iommu_probe_device() وما يليها
include/linux/iommu.h          # تعريفات iommu_ops و iommu_fwspec و iommu_group
include/linux/of.h             # Device Tree core API
```

---

### Commits بارزة في تاريخ `of_iommu`

يمكن البحث عن هذه الـ commits في [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git) أو عبر `git log --all --grep`:

| التغيير | الدلالة |
|---------|---------|
| **تقديم `of_iommu_configure()`** | الدالة الأساسية التي تربط DT node بـ IOMMU driver |
| **إضافة `of_iommu_get_resv_regions()`** | دعم المناطق المحجوزة (reserved regions) من DT |
| **v6.8: "Do not return struct iommu_ops from of_iommu_configure()"** | إعادة هيكلة — الـ probe يعتمد على `iommu_fwspec` لا على ops مباشرة ([مصدر](https://lore.kernel.org/lkml/ZZ6JNzDHy8-i0-VU@8bytes.org/)) |
| **v6.8: "Use -ENODEV consistently in of_iommu_configure()"** | توحيد قيم الخطأ في مسارات الـ probe |

للبحث عن commits مباشرة:

```bash
git log --oneline -- drivers/iommu/of_iommu.c
git log --oneline -- include/linux/of_iommu.h
```

---

### نقاشات mailing list

- **[git pull] IOMMU Updates for Linux v6.8** — [lore.kernel.org](https://lore.kernel.org/lkml/ZZ6JNzDHy8-i0-VU@8bytes.org/) — يشمل إعادة هيكلة `of_iommu_configure()`
- **[git pull] IOMMU Updates for Linux v5.14** — [lore.kernel.org](https://lore.kernel.org/lkml/YN7IDbKZFQnYFCNq@8bytes.org/) — تغييرات core في طريقة probe الأجهزة
- **[PATCH 00/29] Exynos SYSMMU DT integration** — [lore.kernel.org](https://lore.kernel.org/all/?t=20240110130946) — نقاش مُفصَّل عن DT bindings
- للبحث عن نقاشات أحدث: [lore.kernel.org/iommu](https://lore.kernel.org/iommu/)

---

### kernelnewbies.org — تغييرات IOMMU عبر إصدارات kernel

| الإصدار | التغيير المتعلق بـ IOMMU |
|---------|--------------------------|
| [Linux 5.3](https://kernelnewbies.org/Linux_5.3) | إضافة virtio-iommu driver |
| [Linux 5.13](https://kernelnewbies.org/Linux_5.13) | stall support في SMMUv3 + I/O Page Fault handler |
| [Linux 5.16](https://kernelnewbies.org/Linux_5.16) | تحسينات IOMMUFD وdirty tracking |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | تحديثات IOMMU core |
| [Linux 6.7](https://kernelnewbies.org/Linux_6.7) | IOMMUFD Dirty Tracking للـ VM migration |
| [Linux 6.8](https://kernelnewbies.org/Linux_6.8) | إعادة هيكلة `of_iommu_configure()` وإزالة bus_ops |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | IO page fault delivery إلى userspace عبر IOMMUFD |
| [Linux 6.13](https://kernelnewbies.org/Linux_6.13) | vIOMMU infrastructure + SMMUv3 nested translation |

---

### elinux.org — موارد embedded

- [R-Car IO-Virtualization](https://elinux.org/R-Car/IO-Virtualization) — مثال عملي على استخدام IOMMU مع VFIO في أنظمة embedded (Renesas R-Car SoCs)
- [Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) — عروض تشمل DMA و IOMMU integration في embedded Linux
- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — فهرس شامل لمصادر kernel development

---

### كتب موصى بها

#### Linux Device Drivers (LDD3)
- **الفصل 15: Memory Mapping and DMA** — [PDF مجاني](https://static.lwn.net/images/pdf/LDD3/ch15.pdf)
- يغطي: DMA mapping، scatter-gather، bus addresses مقابل virtual addresses
- الأساس النظري لفهم لماذا يُحتاج إلى IOMMU من منظور driver developer

#### Linux Kernel Development — Robert Love (الطبعة الثالثة)
- **الفصل 15: The Process Address Space** — فهم memory management
- **الفصل 19: Portability** — يذكر اختلافات DMA بين المعماريات
- مرجع ممتاز للـ kernel internals بشكل عام

#### Embedded Linux Primer — Christopher Hallinan
- الفصول المتعلقة بـ device tree و platform buses
- يشرح كيف تُربط الأجهزة ببعضها في SoCs المضمنة — سياق مباشر لفهم `of_iommu`

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي memory subsystem بعمق أكبر من LDD3

---

### وثائق kernel الرسمية على الإنترنت

- [IOMMU Subsystem — docs.kernel.org](https://docs.kernel.org/driver-api/iommu.html)
- [IOMMUFD — docs.kernel.org](https://docs.kernel.org/userspace-api/iommufd.html)
- [DMA API — docs.kernel.org](https://docs.kernel.org/core-api/dma-api.html)
- [x86 IOMMU Support — docs.kernel.org](https://docs.kernel.org/arch/x86/iommu.html)
- [An Introduction to IOMMU Infrastructure — Lenovo Press](https://lenovopress.lenovo.com/lp1467-an-introduction-to-iommu-infrastructure-in-the-linux-kernel)

---

### مصطلحات البحث

للعثور على معلومات إضافية استخدم هذه المصطلحات:

```
of_iommu_configure
iommu_fwspec linux kernel
iommu_ops probe_device
linux device tree iommu-map property
iommus phandle device tree binding
CONFIG_OF_IOMMU linux kernel
ARM SMMU device tree binding
iommu_group linux kernel internals
of_iommu_get_resv_regions
linux iommu reserved regions
iommu bus_ops removal v6.8
```

---

### ملخص سريع للمسار التعليمي

```
LDD3 ch15 (DMA basics)
        ↓
lwn.net/Articles/444700 (ARM DMA+IOMMU integration)
        ↓
Documentation/driver-api/iommu.rst
        ↓
drivers/iommu/of_iommu.c  ←  include/linux/of_iommu.h
        ↓
Documentation/devicetree/bindings/iommu/
        ↓
lwn.net/Articles/869818 (IOMMUFD — المستقبل)
```
## Phase 8: Writing simple module

### الفكرة

**`of_iommu_configure()`** هي الدالة الأنسب للـ hook — تُستدعى في كل مرة يحاول فيها الكرنل ربط جهاز (device) بـ IOMMU عبر Device Tree، وهذا يعني أنها تُغطي كل أجهزة النظام التي تحتاج عزل DMA. سنستخدم **kprobe** لاعتراضها وطباعة اسم الجهاز واسم device_node.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on of_iommu_configure()
 * Prints device name and DT node name each time a device is
 * being bound to an IOMMU via the Open Firmware / Device Tree path.
 */

/* --- includes --- */
#include <linux/kernel.h>   /* pr_info, pr_err */
#include <linux/module.h>   /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>  /* struct kprobe, register_kprobe */
#include <linux/device.h>   /* struct device, dev_name() */
#include <linux/of.h>       /* struct device_node */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Learner");
MODULE_DESCRIPTION("kprobe demo: trace of_iommu_configure() calls");

/* -----------------------------------------------------------
 * kprobe pre-handler
 * Called just BEFORE of_iommu_configure() executes.
 *
 * Signature of the hooked function:
 *   int of_iommu_configure(struct device *dev,
 *                          struct device_node *master_np,
 *                          const u32 *id);
 *
 * On x86-64 the arguments arrive in:
 *   regs->di  -> dev        (1st arg)
 *   regs->si  -> master_np  (2nd arg)
 *   regs->dx  -> id         (3rd arg)
 *
 * On ARM64:
 *   regs->regs[0] -> dev
 *   regs->regs[1] -> master_np
 *   regs->regs[2] -> id
 * We use regs_get_kernel_argument() which is arch-independent.
 * ----------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve raw pointer values from CPU registers */
    struct device      *dev       = (struct device *)
                                    regs_get_kernel_argument(regs, 0);
    struct device_node *master_np = (struct device_node *)
                                    regs_get_kernel_argument(regs, 1);
    const u32          *id        = (const u32 *)
                                    regs_get_kernel_argument(regs, 2);

    /* Guard against NULL — can happen during early probe paths */
    if (!dev || !master_np)
        return 0;

    pr_info("of_iommu_configure: dev=\"%s\"  dt_node=\"%s\"  stream_id=%s(%u)\n",
            dev_name(dev),          /* human-readable device name    */
            master_np->full_name,   /* full DT path, e.g. /soc/eth0  */
            id ? "" : "(none) ",    /* stream-id present?            */
            id ? *id : 0);

    return 0; /* returning 0 lets the real function execute normally */
}

/* -----------------------------------------------------------
 * kprobe descriptor
 * symbol_name tells the kernel which function to hook;
 * pre_handler is called before the first instruction runs.
 * ----------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "of_iommu_configure",
    .pre_handler = handler_pre,
};

/* --- module init --- */
static int __init of_iommu_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_iommu_kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("of_iommu_kprobe: hooked of_iommu_configure at %p\n",
            kp.addr);
    return 0;
}

/* --- module exit --- */
static void __exit of_iommu_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_iommu_kprobe: unhooked of_iommu_configure\n");
}

module_init(of_iommu_probe_init);
module_exit(of_iommu_probe_exit);
```

---

### Makefile

```makefile
obj-m += of_iommu_kprobe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
make
sudo insmod of_iommu_kprobe.ko
# لمشاهدة المخرجات:
sudo dmesg | grep of_iommu_kprobe
# أو أثناء probe جهاز جديد:
sudo bash -c 'echo "0000:01:00.0" > /sys/bus/pci/drivers/xhci_hcd/unbind'
sudo bash -c 'echo "0000:01:00.0" > /sys/bus/pci/drivers/xhci_hcd/bind'
sudo rmmod of_iommu_kprobe
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | يعرّف `struct kprobe` وكل API الـ kprobe |
| `linux/device.h` | يعرّف `struct device` و`dev_name()` |
| `linux/of.h` | يعرّف `struct device_node` وحقل `full_name` |
| `linux/module.h` | ضروري لكل وحدة kernel (`MODULE_*`, `module_init`) |

#### الـ `handler_pre` — لماذا `pre` وليس `post`؟

نختار **pre_handler** لأننا نريد قراءة الـ arguments قبل أن تُعدّل الدالة أي منها؛ بعد الانتهاء قد يُعاد استخدام السجلات لأغراض أخرى. **`regs_get_kernel_argument()`** دالة مجردة تتعامل مع فروقات الـ ABI بين x86-64 وARM64 تلقائياً.

#### **`dev_name(dev)`** و **`master_np->full_name`**

**`dev_name`** يُعيد النص المسجّل في sysfs للجهاز (مثل `"0000:01:00.0"` لجهاز PCI)، بينما **`full_name`** يُعيد المسار الكامل داخل Device Tree مثل `"/soc@0/iommu@15000000"` — وهذان معاً يُعطياننا صورة واضحة لأي جهاز يجري ربطه بأي IOMMU.

#### حارس الـ `NULL`

**`of_iommu_configure()`** نفسها تتحقق من `master_np == NULL` وتُعيد `-ENODEV`، لذا يجب أن يتحقق handler الخاص بنا كذلك لتجنب kernel panic عند قراءة `full_name` من مؤشر null.

#### **`module_exit`** — لماذا `unregister_kprobe`؟

إن لم نُلغ تسجيل الـ kprobe عند تفريغ الوحدة، يبقى مؤشر `handler_pre` مُسجَّلاً في جدول الكرنل بينما الكود الذي يُشير إليه قد أُزيل من الذاكرة — النتيجة **kernel panic** عند الاستدعاء التالي. **`unregister_kprobe()`** تُزيل الـ breakpoint وتضمن عدم وجود استدعاءات جارية قبل العودة.

#### لماذا `of_iommu_configure` تحديداً؟

هي الـ **entry point** الوحيدة الـ exported في هذا الـ header، وتُستدعى من `drivers/base/dd.c` أثناء كل عملية probe لجهاز يملك `iommus` property في Device Tree — أي أن hooking هذه الدالة يُعطي رؤية شاملة لكل أجهزة النظام التي تحتاج IOMMU بدون أي تعديل في كود الكرنل.
