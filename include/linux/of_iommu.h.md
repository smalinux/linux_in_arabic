## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الـ `of_iommu.h` جزء من **IOMMU Subsystem** في الـ Linux kernel، وبالتحديد هو الجسر بين الـ **Device Tree (OF = Open Firmware)** وبين الـ IOMMU framework. الـ maintainers الأساسيين هم Joerg Roedel و Will Deacon، والـ mailing list هو `iommu@lists.linux.dev`.

---

### الصورة الكبيرة — قبل أي كود

#### الـ IOMMU إيه أصلاً؟

تخيل إن عندك بيت (الـ RAM)، وعندك ضيوف (الـ devices زي GPU، NIC، USB controller). كل ضيف محتاج يدخل أوض معينة (memory addresses). من غير حارس، أي device ممكن يقرأ أو يكتب في أي مكان في الـ RAM — ده خطر أمني ضخم.

الـ **IOMMU** (Input-Output Memory Management Unit) هو الحارس ده. هو hardware موجود على الـ CPU/SoC بيتحكم في الـ DMA (Direct Memory Access) للـ devices. بيعمل virtual address space للـ device تماماً زي ما الـ MMU بيعمله للـ processes.

```
[ Device (GPU/NIC) ] --DMA request--> [ IOMMU ] --translated--> [ Physical RAM ]
                                          |
                                    checks page tables
                                    (per-device isolation)
```

**فايدته:**
- **Security**: isolates devices عن بعض (مهم في virtualization وـ VFIO)
- **Protection**: device معطوب أو malicious مش هيقدر يكتب في kernel memory
- **Virtualization**: بيخلي VM تتحكم في device حقيقي بأمان (VFIO/VFIO-PCI)

---

#### الـ Device Tree إيه وليه محتاجينه هنا؟

في عالم الـ embedded وـ ARM/RISC-V السيسطمات، الـ hardware مش بيعرف يـ self-describe نفسه زي PCI. فبيتوصف في ملف نصي اسمه **Device Tree (DT)**، وده بيتحول لـ binary blob اسمه **DTB** وبيتحمل مع الـ kernel.

مثلاً في الـ Device Tree:
```
gpu@ff000000 {
    iommus = <&smmu 0x42>;  /* ← connected to SMMU, stream ID 0x42 */
};

smmu: iommu@fd800000 {
    compatible = "arm,smmu-v3";
    #iommu-cells = <1>;
};
```

الـ `iommus` property دي بتقول: الـ GPU ده متوصل بالـ IOMMU controller اللي اسمه `smmu`، وعنده stream ID = `0x42`.

---

#### دور `of_iommu.h` تحديداً

الملف ده هو **واجهة (interface/API header)** بسيطة جداً بتعلن عن دالتين بس، مسؤوليتهم هي:

**أولاً: `of_iommu_configure()`**
لما الـ kernel بيعمل probe لـ device معين، بيحتاج يعرف: هل الـ device ده موصل بـ IOMMU؟ وإيه الـ stream ID بتاعه؟ الدالة دي بتقرأ معلومات الـ Device Tree وبتربط الـ device بالـ IOMMU driver الصح.

```
device probe
    └─> of_iommu_configure()
            └─> reads "iommus" / "iommu-map" from DT
            └─> calls iommu_ops->of_xlate() on the right driver
            └─> device now has iommu_fwspec attached
```

**ثانياً: `of_iommu_get_resv_regions()`**
بعض الـ devices محتاجة مناطق معينة في الـ address space تكون محجوزة (reserved) — زي MSI doorbell addresses. الدالة دي بتجيب الـ reserved regions دي من الـ Device Tree.

---

### القصة الكاملة — من البداية للنهاية

تخيل عندك SoC فيها GPU ومتوصلة بـ ARM SMMU (وهو IOMMU من ARM):

1. **Boot time**: الـ kernel بيقرأ الـ DTB، بيلاقي node الـ GPU.
2. **Driver probe**: driver الـ GPU بيتعمل probe.
3. **IOMMU setup**: الـ bus code بيستدعي `of_iommu_configure(dev, gpu_node, NULL)`.
4. **DT parsing**: الدالة بتقرأ `iommus = <&smmu 0x42>` من الـ Device Tree.
5. **Driver lookup**: بتدور على الـ SMMU driver اللي registered نفسه.
6. **Translation**: بتستدعي `smmu_ops->of_xlate(dev, &spec)` عشان تسجل الـ stream ID.
7. **fwspec attached**: دلوقتي الـ device عنده `iommu_fwspec` فيها كل معلومات الـ IOMMU.
8. **Domain allocation**: الـ IOMMU core بيعمل `iommu_domain` ويربط الـ device بيه.
9. **DMA isolated**: كل DMA من الـ GPU دلوقتي بيعدي من الـ SMMU وبيتترجم.

الـ `of_iommu.h` هو الـ header اللي بيعلن عن خطوة 3 دي.

---

### ليه الملف صغير جداً؟

الملف عمداً بسيط — هو مجرد **public API declaration**. كل الـ implementation موجودة في `drivers/iommu/of_iommu.c`. الفصل ده مهم: أي driver أو subsystem محتاج يـ configure IOMMU من DT، بيـ include الـ header ده بس من غير ما يعرف أي تفاصيل implementation.

كمان لو `CONFIG_OF_IOMMU` مش enabled (زي على x86 من غير DT)، الـ inline stubs بترجع `-ENODEV` بشكل صامت — الكود مش هيـ break.

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/of_iommu.h` | الملف ده — public API header |
| `drivers/iommu/of_iommu.c` | الـ implementation الكاملة للـ API |
| `include/linux/iommu.h` | الـ IOMMU core API (domains, ops, fwspec) |
| `drivers/iommu/iommu.c` | الـ IOMMU core framework |
| `include/linux/of.h` | الـ Device Tree core API |
| `drivers/iommu/arm/arm-smmu/arm-smmu.c` | مثال على IOMMU driver بيستخدم `of_xlate` |
| `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` | ARM SMMU v3 driver |
| `drivers/iommu/dma-iommu.c` | الـ DMA layer فوق IOMMU |
| `Documentation/devicetree/bindings/iommu/` | الـ DT bindings specs |

---

### ملفات الـ Subsystem الأساسية (Core + Headers + HW Drivers)

**Core:**
- `drivers/iommu/iommu.c` — القلب الأساسي للـ framework
- `drivers/iommu/of_iommu.c` — الـ DT glue layer
- `drivers/iommu/dma-iommu.c` — DMA API integration
- `drivers/iommu/iova.c` — IOVA allocator

**Headers:**
- `include/linux/iommu.h` — الـ main IOMMU API
- `include/linux/of_iommu.h` — الـ DT-IOMMU bridge API (الملف ده)
- `include/linux/iova.h` — IOVA types

**HW Drivers (أمثلة):**
- `drivers/iommu/arm/arm-smmu/arm-smmu.c` — ARM SMMU v1/v2
- `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` — ARM SMMU v3
- `drivers/iommu/intel/iommu.c` — Intel VT-d
- `drivers/iommu/amd/iommu.c` — AMD IOMMU (AMD-Vi)
- `drivers/iommu/apple-dart.c` — Apple DART (M1/M2)
- `drivers/iommu/tegra-smmu.c` — NVIDIA Tegra SMMU
- `drivers/iommu/mtk_iommu.c` — MediaTek IOMMU
## Phase 2: شرح الـ OF-IOMMU Framework

### المشكلة اللي بيحلها الـ Subsystem ده

في السيستم الـ embedded، عندك devices زي GPU، DMA controller، أو Network card كلها بتعمل **DMA** — بتكتب وبتقرأ من الـ RAM مباشرة.

من غير حماية، أي device ممكن يوصل لأي عنوان في الـ RAM — حتى لو ده RAM بتاع kernel أو process تانية. ده security hole واسع جداً، وفي بيئة الـ virtualization بيبقى كارثة.

الحل هو الـ **IOMMU** (Input-Output Memory Management Unit): هو unit هاردوير موجود بين الـ bus والـ RAM، بيترجم الـ **IOVA** (I/O Virtual Address) اللي الـ device شايفها لـ Physical Address حقيقي، وبيطبّق صلاحيات Read/Write على كل mapping.

بس السؤال: **إزاي الـ kernel يعرف إن الـ device دي متوصلة بأنهي IOMMU؟ وإيه الـ stream ID بتاعتها؟**

في البيئات اللي بتستخدم **Device Tree** (معظم ARM embedded)، الإجابة موجودة في الـ DT nodes. وده بالضبط اللي بيعمله الـ `of_iommu` subsystem — يربط بين الـ Device Tree descriptions وبين الـ IOMMU framework.

---

### الحل: OF-IOMMU كـ Glue Layer

الـ **`of_iommu`** مش IOMMU driver بنفسه — هو **glue layer** بين:

- الـ **Device Tree parser** (OF = Open Firmware) اللي بيفهم الـ DT nodes
- الـ **IOMMU core framework** اللي بيدير الـ domains والـ mappings
- الـ **IOMMU hardware drivers** (مثلاً `arm-smmu`, `intel-iommu`, `ipmmu-vmsa`)

الـ subsystem بيعمل حاجتين فقط:
1. `of_iommu_configure()` — يقرأ الـ DT ويوصّل الـ device بالـ IOMMU driver الصح
2. `of_iommu_get_resv_regions()` — يجيب الـ memory regions اللي الـ IOMMU محتاج يحجزها لنفسه (مثلاً MSI regions)

---

### الـ Big Picture Architecture

```
  ┌────────────────────────────────────────────────────────────────────┐
  │                        Device Tree (DTB)                           │
  │                                                                    │
  │  gpu@fc000000 {                                                    │
  │      iommus = <&smmu 0x120>;   ← phandle + stream-ID              │
  │  };                                                                │
  │                                                                    │
  │  smmu: iommu@d0000000 {                                            │
  │      compatible = "arm,smmu-v2";                                   │
  │      #iommu-cells = <1>;       ← عدد args بعد الـ phandle         │
  │  };                                                                │
  └──────────────────────────┬─────────────────────────────────────────┘
                             │  of_parse_phandle_with_args()
                             ▼
  ┌──────────────────────────────────────────────────────┐
  │              of_iommu.c  (OF-IOMMU Glue)             │
  │                                                      │
  │  of_iommu_configure(dev, master_np, id)              │
  │    ├─ يقرأ "iommus" property من الـ DT              │
  │    ├─ يجيب الـ iommu_ops المسجّلة للـ IOMMU node    │
  │    └─ يستدعي iommu_ops->of_xlate()                  │
  │                                                      │
  │  of_iommu_get_resv_regions(dev, list)                │
  │    └─ يستدعي iommu_ops->get_resv_regions()           │
  └──────────────────────┬───────────────────────────────┘
                         │
        ┌────────────────┼────────────────────┐
        ▼                ▼                    ▼
  ┌──────────┐    ┌──────────────┐    ┌──────────────┐
  │arm-smmu.c│    │arm-smmu-v3.c │    │ipmmu-vmsa.c  │
  │(IOMMU    │    │(IOMMU        │    │(IOMMU        │
  │ Driver)  │    │ Driver)      │    │ Driver)      │
  │          │    │              │    │              │
  │.of_xlate │    │.of_xlate     │    │.of_xlate     │
  └──────────┘    └──────────────┘    └──────────────┘
        │                │                    │
        └────────────────┴────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │              IOMMU Core Framework                    │
  │                                                      │
  │  iommu_group  ←─── devices sharing same page table  │
  │  iommu_domain ←─── the actual IO page table         │
  │  iommu_ops    ←─── vtable للـ hardware driver       │
  └──────────────────────────────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │              IOMMU Hardware (ARM SMMU)               │
  │  translates IOVA → PA for all bus masters            │
  └──────────────────────────────────────────────────────┘
```

---

### الـ Device Tree Integration: بالتفصيل

الـ DT بيحدد العلاقة بين device وبين IOMMU عن طريق الـ `iommus` property:

```dts
/* مثال حقيقي من Qualcomm SoC */
adreno_gpu: gpu@5000000 {
    compatible = "qcom,adreno-630.2";
    reg = <0x05000000 0x40000>;
    iommus = <&adreno_smmu 0x0>,   /* stream ID للـ vertex/pixel */
             <&adreno_smmu 0x1>;   /* stream ID للـ compute     */
};

adreno_smmu: iommu@5040000 {
    compatible = "qcom,adreno-smmu";
    reg = <0x05040000 0x10000>;
    #iommu-cells = <1>;            /* كل device بتديله stream ID واحد */
};
```

الـ `#iommu-cells = <1>` معناه إن كل reference للـ IOMMU دي بتاخد argument واحد إضافي بعد الـ phandle — وهو الـ **stream ID** (أو **SID**).

الـ **Stream ID** هو الرقم اللي الـ SMMU بيعرف بيه مين بعت الـ transaction دي على الـ bus. كل device ليها SID فريد، والـ SMMU بيستخدمه عشان يبحث في الـ stream table ويجيب الـ page table المناسبة.

---

### الـ Core Abstraction: الـ `of_phandle_args`

الـ struct الأساسي في اللعبة دي هو `struct of_phandle_args` (معرّف في `include/linux/of.h`):

```c
struct of_phandle_args {
    struct device_node *np;      /* pointer للـ IOMMU node نفسه */
    int args_count;              /* عدد الـ arguments (الـ cells) */
    uint32_t args[MAX_PHANDLE_ARGS]; /* الـ stream IDs أو أي args تانية */
};
```

لما الـ kernel يقرأ `iommus = <&smmu 0x120>`:
- `np` → بيشاور على الـ `smmu` node
- `args_count` = 1
- `args[0]` = `0x120` (الـ stream ID)

---

### إيه اللي بيحصل تحديداً في `of_iommu_configure()`

```
of_iommu_configure(dev, master_np, id)
       │
       ├─ 1. يجيب device_node للـ device
       │
       ├─ 2. يعمل loop على كل entry في "iommus" property
       │      └─ of_parse_phandle_with_args(np, "iommus", "#iommu-cells", ...)
       │
       ├─ 3. لكل entry: يجيب الـ iommu_ops المسجّلة للـ IOMMU node ده
       │      └─ of_fwnode_handle(iommu_np) → iommu_ops
       │
       ├─ 4. يستدعي iommu_ops->of_xlate(dev, &iommu_spec)
       │      └─ الـ driver بيضيف الـ device للـ iommu_group الصح
       │         وبيحفظ الـ stream ID عشان يبني الـ stream table entry
       │
       └─ 5. الـ device دلوقتي connected للـ IOMMU وجاهزة تتاخد domain
```

---

### تشبيه الدنيا الحقيقية — الـ Hotel والـ Room Key System

تخيل فندق كبير (الـ SoC):

| المفهوم الحقيقي | التشبيه |
|---|---|
| الـ IOMMU hardware | الـ Security system في الفندق |
| الـ IOVA | رقم الغرفة اللي الـ device شايفاه |
| الـ Physical Address | الموقع الفعلي للغرفة في المبنى |
| الـ stream ID | رقم البطاقة الخاصة بكل نزيل |
| الـ iommu_domain | الـ floor اللي المفتاح بيفتح غرفه بس |
| الـ iommu_group | مجموعة نزلاء بيشتركوا نفس الـ floor |
| الـ Device Tree | كتاب الحجوزات اللي بيقول مين في أنهي غرفة |
| **الـ of_iommu** | **موظف الاستقبال** اللي بيقرأ الحجز ويسجّل النزيل في الـ system |

موظف الاستقبال (of_iommu) مش بينفّذ الـ security بنفسه — بس هو اللي بيربط المعلومات الموجودة في كتاب الحجوزات (DT) بالـ security system (IOMMU driver). بدونه، كل device تبقى stranger والـ security system مش عارف يعاملها صح.

---

### الـ `of_iommu_get_resv_regions()`: ليه موجودة؟

بعض منطق الـ IOMMU محتاج **reserved memory regions** — مناطق في الـ IO address space لازم تتعامل بطريقة خاصة:

- **MSI doorbell addresses**: لما device بتبعت MSI interrupt، بتكتب على address معينة. الـ IOMMU لازم يعرف العنوان ده ويتعامل معاه كـ `IOMMU_RESV_MSI` مش mapping عادي.
- **Direct-mapped regions**: مناطق لازم تفضل physical == iova (identity mapped) طول الوقت.

الـ `of_iommu_get_resv_regions()` بتمشي على الـ DT وبتجمع كل الـ regions دي في `list_head` عشان الـ IOMMU core يعرف يتعامل معاها صح.

---

### الـ CONFIG_OF_IOMMU Guard

الملف الكامل محاط بـ `#ifdef CONFIG_OF_IOMMU`:

```c
#ifdef CONFIG_OF_IOMMU
extern int of_iommu_configure(...);
extern void of_iommu_get_resv_regions(...);
#else
/* stub functions تبعت -ENODEV أو void */
static inline int of_iommu_configure(...) { return -ENODEV; }
static inline void of_iommu_get_resv_regions(...) { }
#endif
```

ده pattern كلاسيكي في الـ kernel:
- لو الـ platform مش بتستخدم DT (مثلاً x86 ACPI-only) → الـ `CONFIG_OF_IOMMU` بيبقى `n` والـ stubs بيتكمبايلوا بدلها
- الـ callers مش محتاجين `#ifdef` في كودهم — كل حاجة transparent

---

### إيه اللي الـ of_iommu بيمتلكه vs إيه اللي بيفوّضه

| المسؤولية | of_iommu يملكها | يفوّضها لـ |
|---|---|---|
| قراءة `iommus` property من DT | ✅ | — |
| parse الـ phandle args | ✅ | OF core (`of_parse_phandle_with_args`) |
| ربط device بـ iommu_group | ❌ | IOMMU driver عبر `of_xlate()` |
| بناء stream table entries | ❌ | IOMMU driver (arm-smmu, etc.) |
| تخصيص الـ iommu_domain | ❌ | IOMMU core |
| تنفيذ الـ page table walks | ❌ | IOMMU hardware + driver |
| جمع reserved regions | ✅ (orchestration) | IOMMU driver عبر `get_resv_regions()` |
| التعامل مع ACPI/non-DT | ❌ | `iort.c` (IORT subsystem) |

---

### العلاقة مع Subsystems تانية

- **OF (Open Firmware / Device Tree subsystem)**: الـ `of_iommu` يعتمد عليه عشان يقرأ الـ DT nodes. الـ `struct device_node` و`struct of_phandle_args` كلهم من OF subsystem.
- **IOMMU core** (`drivers/iommu/iommu.c`): هو اللي بيدير الـ groups والـ domains. الـ `of_iommu` بيكلمه عبر `iommu_ops`.
- **DMA API** (`kernel/dma/`): بيستخدم الـ IOMMU domain عشان يعمل DMA mappings آمنة. الـ `of_iommu` هو الخطوة الأولى اللي بتخلي الـ DMA API يعرف يستخدم الـ IOMMU.
- **Platform/bus drivers**: لما device بتتسجّل، الـ bus code (مثلاً `platform_bus_type`) بيستدعي `of_iommu_configure()` تلقائياً.

---

### الـ Struct Relationship Map

```
struct device
    └── dev->iommu_group ──────────────────────┐
    └── dev->fwnode (= device_node->fwnode)     │
              │                                 │
              │  of_iommu_configure()           ▼
              │  reads "iommus" property   struct iommu_group
              │                                 │
              ▼                                 │
    struct of_phandle_args                      │
        ├── np → struct device_node (IOMMU)     │
        └── args[] = [stream_id, ...]           │
              │                                 │
              │  iommu_ops->of_xlate()          │
              ▼                                 │
    struct iommu_ops  (per IOMMU hardware)      │
        ├── .of_xlate()   ─── adds dev to ──────┘
        ├── .probe_device()
        ├── .domain_alloc_paging()
        ├── .get_resv_regions()  ◄── of_iommu_get_resv_regions()
        └── ...
              │
              ▼
    struct iommu_domain
        ├── type (DMA / IDENTITY / BLOCKED)
        ├── ops → struct iommu_domain_ops
        │           ├── .map_pages()
        │           ├── .unmap_pages()
        │           └── .iova_to_phys()
        └── iova_cookie (DMA iova allocator state)
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### نظرة عامة على الملف

الـ `include/linux/of_iommu.h` هو **header بسيط جداً** — بيعرّف واجهة الربط بين الـ **Open Firmware (Device Tree)** والـ **IOMMU subsystem**. مش فيه structs خاصة بيه، لكن بيستخدم structs من subsystems تانية وبيعرّف API بيشتغل عليها. الفهم الحقيقي جاي من السياق الأكبر.

---

### 0. الـ Config Options والـ Flags — Cheatsheet

#### Config Options

| Option | المعنى |
|--------|--------|
| `CONFIG_OF_IOMMU` | لو مفعّل، الـ kernel يدعم ربط الـ Device Tree بالـ IOMMU. لو مش مفعّل، الدوال بتبقى stubs بترجع `-ENODEV` أو لا تعمل حاجة. |

#### الـ IOMMU Permission Flags (من `iommu.h`)

| Flag | القيمة | المعنى |
|------|--------|--------|
| `IOMMU_READ` | `1 << 0` | السماح بالقراءة |
| `IOMMU_WRITE` | `1 << 1` | السماح بالكتابة |
| `IOMMU_CACHE` | `1 << 2` | DMA cache coherency |
| `IOMMU_NOEXEC` | `1 << 3` | منع التنفيذ |
| `IOMMU_MMIO` | `1 << 4` | للـ MMIO زي MSI doorbells |
| `IOMMU_PRIV` | `1 << 5` | صلاحيات supervisor فقط |

#### الـ IOMMU Domain Types

| Macro | المعنى |
|-------|--------|
| `IOMMU_DOMAIN_BLOCKED` | كل الـ DMA محجوب — عزل كامل |
| `IOMMU_DOMAIN_IDENTITY` | الـ DMA addresses = الـ physical addresses (1:1) |
| `IOMMU_DOMAIN_UNMANAGED` | الـ mappings بتتدار من المستخدم (VMs) |
| `IOMMU_DOMAIN_DMA` | للاستخدام الداخلي مع DMA-API |
| `IOMMU_DOMAIN_DMA_FQ` | نفس الفوق + batched TLB invalidation |
| `IOMMU_DOMAIN_SVA` | Shared Virtual Addressing مع process |
| `IOMMU_DOMAIN_NESTED` | nested stage-2 translation |

#### الـ Reserved Region Types (`enum iommu_resv_type`)

| القيمة | المعنى |
|--------|--------|
| `IOMMU_RESV_DIRECT` | لازم تتعمل map 1:1 دايماً |
| `IOMMU_RESV_DIRECT_RELAXABLE` | 1:1 بس ممكن تتخفف في حالات (USB/Graphics) |
| `IOMMU_RESV_RESERVED` | محجوزة — ممنوع تتعمل map خالص |
| `IOMMU_RESV_MSI` | MSI hardware region (untranslated) |
| `IOMMU_RESV_SW_MSI` | MSI translation window بتتدار بـ software |

#### الـ device_node Flags

| Flag | القيمة | المعنى |
|------|--------|--------|
| `OF_DYNAMIC` | 1 | الـ node اتخصصتله ذاكرة بـ `kmalloc` |
| `OF_DETACHED` | 2 | انفصل عن الـ device tree |
| `OF_POPULATED` | 3 | اتعمله device بالفعل |
| `OF_POPULATED_BUS` | 4 | اتعمله platform bus للـ children |
| `OF_OVERLAY` | 5 | مخصص لـ overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | في overlay cset بيتحرر |

---

### 1. الـ Structs المهمة

#### `struct device` (forward declaration في الملف)

**الغرض:** يمثّل أي device في الـ kernel — سواء PCI أو platform أو غيره.

الملف بيستخدمه كـ parameter في الدالتين الرئيسيتين:
- `of_iommu_configure(struct device *dev, ...)` — بيعمل configure للـ IOMMU للـ device ده.
- `of_iommu_get_resv_regions(struct device *dev, ...)` — بيجيب الـ reserved regions الخاصة بالـ device.

**الحقول المهمة في السياق ده (من `device.h`):**
- `dev->iommu_group` — الـ IOMMU group اللي الـ device ده جزء منه.
- `dev->fwnode` — الـ firmware node (هيكون `device_node` في حالة DT).
- `dev->iommu` — الـ per-device IOMMU data.

---

#### `struct device_node` (forward declaration + تعريف في `of.h`)

**الغرض:** يمثّل **node في الـ Device Tree** — يعني كل جهاز/controller موصوف في الـ DTS.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `name` | `const char *` | اسم الـ node |
| `phandle` | `phandle` (u32) | معرّف فريد للـ reference من nodes تانية |
| `full_name` | `const char *` | الاسم الكامل مع المسار |
| `fwnode` | `struct fwnode_handle` | الواجهة الموحدة لكل firmware nodes |
| `properties` | `struct property *` | قائمة linked list من الـ properties |
| `parent` | `struct device_node *` | الـ parent node |
| `child` | `struct device_node *` | أول child |
| `sibling` | `struct device_node *` | الـ node الجنبه على نفس المستوى |
| `_flags` | `unsigned long` | الـ flags (OF_DYNAMIC, OF_POPULATED, ...) |

**الـ `master_np` في `of_iommu_configure`:** ده pointer لـ `device_node` بيمثّل الـ **IOMMU master** — يعني الـ device اللي بيستخدم الـ IOMMU (مش الـ IOMMU controller نفسه). الـ core بيقرأ الـ property `iommus` من الـ DTS عشان يلاقي الـ IOMMU controller المناسب.

---

#### `struct iommu_ops` (forward declaration في الملف)

**الغرض:** جدول الـ operations الخاص بـ IOMMU driver معين — الـ vtable.

الملف بيعلنه كـ forward declaration بس، لكن الـ core بيستخدمه داخلياً في `of_iommu_configure` عشان يعرف إيه الـ IOMMU driver المسؤول عن الـ device ده.

**أهم الـ ops في السياق ده:**
- `ops->xlate` — بيترجم `of_phandle_args` لـ IOMMU group.
- `ops->get_resv_regions` — بيجيب الـ reserved regions.
- `ops->of_xlate` — نسخة تانية للترجمة من DT.

---

#### `struct iommu_resv_region` (من `iommu.h`)

**الغرض:** وصف **منطقة ذاكرة محجوزة** ما يجيش الـ IOMMU يعمل لها map عادية.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `list` | `struct list_head` | ربطها في الـ list |
| `start` | `phys_addr_t` | بداية المنطقة في الذاكرة الفعلية |
| `length` | `size_t` | الطول بالـ bytes |
| `prot` | `int` | صلاحيات IOMMU (READ/WRITE/...) |
| `type` | `enum iommu_resv_type` | نوع الحجز |

**الـ `list` في `of_iommu_get_resv_regions`:** الـ function بتضيف عناصر لـ `list_head` الجاهزة من الـ caller — كل عنصر هو `iommu_resv_region` بيوصف منطقة محجوزة.

---

#### `struct of_phandle_args` (من `of.h`)

**الغرض:** بيحمل نتيجة parse الـ property `iommus` من الـ DTS — يعني الـ phandle للـ IOMMU controller + الـ arguments (زي الـ stream ID).

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `np` | `struct device_node *` | الـ node الـ IOMMU controller |
| `args_count` | `int` | عدد الـ arguments |
| `args[]` | `uint32_t[MAX_PHANDLE_ARGS]` | الـ arguments (زي stream ID أو SID) |

---

### 2. مخططات علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                    of_iommu_configure()                     │
│            (struct device *dev, device_node *master_np)     │
└──────────────┬──────────────────────────┬───────────────────┘
               │                          │
               ▼                          ▼
    ┌──────────────────┐      ┌────────────────────────┐
    │   struct device  │      │   struct device_node   │
    │                  │      │   (master_np)          │
    │  ┌────────────┐  │      │                        │
    │  │ fwnode ────┼──┼─────►│ fwnode_handle          │
    │  └────────────┘  │      │                        │
    │  ┌────────────┐  │      │ properties ────────────┼──►  "iommus" property
    │  │iommu_group │  │      │   (iommus = <&iommu0   │     (phandle + args)
    │  └────────────┘  │      │             0x400>)    │
    └──────────────────┘      └───────────┬────────────┘
                                          │ of_parse_phandle_with_args()
                                          ▼
                              ┌────────────────────────┐
                              │  struct of_phandle_args│
                              │  np ──────────────────►│ IOMMU controller node
                              │  args[0] = 0x400       │ (stream ID / SID)
                              └───────────┬────────────┘
                                          │ iommu_ops->of_xlate()
                                          ▼
                              ┌────────────────────────┐
                              │   struct iommu_ops     │
                              │   (IOMMU driver vtable)│
                              │   .of_xlate()          │
                              │   .get_resv_regions()  │
                              └───────────┬────────────┘
                                          │
                                          ▼
                              ┌────────────────────────┐
                              │   struct iommu_group   │
                              │   (device assigned)    │
                              └────────────────────────┘
```

```
of_iommu_get_resv_regions():

    struct device *dev
         │
         │ dev->iommu_group → iommu_ops
         ▼
    iommu_ops->get_resv_regions()
         │
         │ يضيف عناصر في الـ list
         ▼
    struct list_head *list
         │
         ├──► struct iommu_resv_region (MSI region)
         ├──► struct iommu_resv_region (DIRECT mapping)
         └──► struct iommu_resv_region (RESERVED range)
```

---

### 3. دورة حياة الـ IOMMU configuration عبر DT

```
Boot / Device Probe
        │
        ▼
┌───────────────────────────────┐
│   DTS Parser يقرأ device tree │
│   ويبني struct device_node    │
│   لكل node فيه               │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  Platform/PCI bus driver      │
│  يعمل probe للـ device        │
│  ويجهّز struct device         │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  of_iommu_configure() يتحاكى │
│  لما الـ bus بيسجّل الـ device │
│  (من iommu_probe_device)      │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  يقرأ property "iommus" من   │
│  master_np → of_phandle_args  │
│  يجيب stream ID + controller │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  يلاقي الـ iommu_ops الخاصة  │
│  بالـ IOMMU controller ده    │
│  عن طريق الـ driver المسجّل  │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  iommu_ops->of_xlate()        │
│  بيحوّل الـ phandle args      │
│  لـ iommu_group               │
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  الـ device يتضاف للـ group  │
│  ويتعمله domain attach        │
│  جاهز للـ DMA                │
└───────────────────────────────┘


Device Removal:
        │
        ▼
┌───────────────────────────────┐
│  iommu_release_device()       │
│  بتشيل الـ device من الـ group│
│  وتحرر الـ domain لو محتاج   │
└───────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### `of_iommu_configure()` — التدفق الكامل

```
bus driver / iommu_probe_device()
    │
    └─► of_iommu_configure(dev, master_np, id)       [of_iommu.c]
            │
            ├─► of_property_read_bool(master_np, "iommu-map")
            │       └── لو موجودة: مسار iommu-map (SMMU v3 style)
            │
            ├─► of_parse_phandle_with_args(master_np, "iommus", ...)
            │       └── بيجيب: of_phandle_args { np=&iommu_node, args[0]=stream_id }
            │
            ├─► iommu_fwspec_init(dev, &iommu_np->fwnode, iommu_ops)
            │       └── بيجهّز struct iommu_fwspec للـ device
            │
            ├─► iommu_ops->of_xlate(dev, &iommu_spec)
            │       └── الـ IOMMU driver بيسجّل الـ stream ID للـ device
            │
            └─► iommu_probe_device(dev)
                    └── بيعمل:
                        ├─► iommu_ops->probe_device(dev)  → iommu_group
                        ├─► iommu_group_add_device(group, dev)
                        └─► iommu_setup_dma_ops(dev, ...)
```

#### `of_iommu_get_resv_regions()` — التدفق

```
iommu_get_resv_regions(dev, list)           [iommu.c]
    │
    └─► of_iommu_get_resv_regions(dev, list)  [of_iommu.c]
            │
            ├─► of_get_dma_range_info(dev, ...)
            │       └── بيقرأ "dma-ranges" من DTS
            │
            ├─► لكل MSI controller في الـ DTS:
            │       iommu_alloc_resv_region(msi_addr, size,
            │                               IOMMU_RESV_MSI, ...)
            │       list_add_tail(&region->list, list)
            │
            └─► الـ caller (iommu core) يستخدم الـ list
                لتجنب map الـ regions دي في الـ IOMMU domain
```

---

### 5. استراتيجية الـ Locking

الملف نفسه ما عندوش locking — ده header بيعرّف API بس. لكن الـ implementation في `drivers/iommu/of_iommu.c` بتعتمد على:

#### الـ Locks المستخدمة في السياق

| الـ Lock | النوع | بيحمي إيه |
|----------|-------|-----------|
| `iommu_group->mutex` | `struct mutex` | الـ devices list داخل الـ group وأي تعديل عليه |
| `iommu_group_list_lock` | `struct mutex` | الـ global list بتاع كل الـ groups |
| `iommu_sva_lock` | `struct mutex` | الـ SVA domains list في الـ mm_struct |
| `device_lock(dev)` | implicitly | بيتعمل hold من الـ bus driver أثناء الـ probe |
| `of_node` refcounting | atomic | الـ `of_node_get/put` للـ reference counting |

#### ترتيب الـ Locks (Lock Ordering)

```
device_lock(dev)
    └─► iommu_group->mutex
            └─► iommu_group_list_lock
                    └─► (hardware register access — no sleep)
```

**قاعدة مهمة:** مش ينفع تمسك `iommu_group->mutex` وتحاول تعمل `device_lock` — ده هيعمل deadlock. الـ bus driver دايماً بيمسك الـ device lock الأول.

#### الـ Reference Counting للـ device_node

الـ `struct device_node` بيستخدم **refcounting** عبر `of_node_get()` و `of_node_put()`. لما `of_iommu_configure` تاخد الـ `master_np`، الـ caller مسؤول إن الـ node يفضل valid طول فترة الاستدعاء. الـ `of_parse_phandle_with_args` بيعمل `of_node_get` داخلياً ولازم يتبعه `of_node_put` بعد الاستخدام.

```c
/* مثال على الـ safe usage */
struct of_phandle_args iommu_spec;
int ret;

ret = of_parse_phandle_with_args(master_np, "iommus",
                                  "#iommu-cells", idx, &iommu_spec);
if (ret)
    return ret;

/* ... استخدام iommu_spec.np ... */

of_node_put(iommu_spec.np);  /* لازم دايماً بعد الاستخدام */
```
## Phase 4: شرح الـ Functions

### جدول ملخص الـ API

| Function | Config Guard | Return | الغرض |
|---|---|---|---|
| `of_iommu_configure` | `CONFIG_OF_IOMMU` | `int` | ربط device بالـ IOMMU المناسب عبر DT |
| `of_iommu_get_resv_regions` | `CONFIG_OF_IOMMU` | `void` | جلب reserved memory regions من DT |

---

### تصنيف الـ Functions

الـ header ده بسيط جداً — بيوفّر **فقط** واجهتين (2 exported functions) اللي بتشكّل الـ glue layer بين الـ **Open Firmware / Device Tree (OF)** والـ **IOMMU subsystem**. الفكرة الكاملة هي: بدل ما كل driver يفسّر الـ DT nodes يدوياً، الـ `of_iommu` layer بتعمل ده مرة واحدة بشكل موحّد.

الـ functions الاتنين بيتم تعريفهم مرتين:
- **Real implementation** لما `CONFIG_OF_IOMMU=y`
- **Stub (no-op / error)** لما `CONFIG_OF_IOMMU` مش موجود — ده pattern شائع في الـ kernel علشان تتجنب `#ifdef` في كل مكان بره الـ header.

---

### المجموعة الأولى: Configuration & Binding

#### `of_iommu_configure`

```c
extern int of_iommu_configure(struct device *dev,
                              struct device_node *master_np,
                              const u32 *id);
```

**بتعمل إيه:**
الـ function دي هي **نقطة الدخول الرئيسية** لربط أي device بالـ IOMMU بتاعه عبر الـ Device Tree. بتمشي على الـ `iommus` property في الـ DT node الخاص بالـ device، بتحدد الـ IOMMU driver المناسب، وبتستدعي `iommu_probe_device()` علشان تخلي الـ IOMMU driver يعمل setup للـ device. الـ function دي بتضمن إن الـ device هيشتغل صح مع الـ IOMMU قبل ما يبدأ أي DMA.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي محتاج يتربط بالـ IOMMU — بيجي من الـ bus probe path |
| `master_np` | `struct device_node *` | الـ DT node الخاص بالـ device — بيحتوي على الـ `iommus` property |
| `id` | `const u32 *` | مؤشر لـ stream ID اختياري (ممكن يكون `NULL`) — بُعض الـ IOMMUs زي ARM SMMU بتستخدمه كـ stream identifier |

**Return Value:**
- `0` — النجاح، الـ device اتربط بالـ IOMMU بنجاح
- `-ENODEV` — في الـ stub version (يعني `CONFIG_OF_IOMMU` مش موجود)
- Error codes سالبة تانية — فشل في الـ probe أو مفيش IOMMU مناسب في الـ DT

**Key Details:**

- **Locking:** مفيش lock صريح في الـ header، لكن الـ implementation الداخلية بتعمل `device_lock(dev)` قبل ما تمس الـ `dev->iommu`.
- **DT Parsing:** بتستخدم `of_parse_phandle_with_args()` داخلياً علشان تقرأ الـ `iommus = <&smmu 0x100>` property وتجيب الـ phandle والـ args.
- **Side Effect رئيسي:** بتضبط `dev->iommu_group` — لو نجحت، الـ device بقى part من IOMMU group وهيتعامل معاه الـ DMA API بشكل مختلف.
- **Error Path:** لو الـ IOMMU driver مش loaded أو مش registered بعد، بترجع error — الـ caller (عادةً bus notifier) ممكن يعيد المحاولة.

**من بيستدعيها:**
بيتم الـ call من `of_iommu_configure()` في مسار الـ device registration، تحديداً من:
- `platform_bus` notifier
- الـ `iommu_probe_device()` path في بعض الـ architectures
- الـ bus-specific `add_device()` hooks زي `arm_smmu_add_device()`

**Pseudocode Flow:**

```c
of_iommu_configure(dev, master_np, id):
    // 1. ابحث عن iommu_ops اللي بيخدم الـ DT node ده
    ops = of_iommu_xlate(master_np)
    if (!ops)
        return -ENODEV

    // 2. اربط الـ ops بالـ device
    dev->bus->iommu_ops = ops

    // 3. استدع الـ IOMMU driver علشان يعمل probe للـ device
    return iommu_probe_device(dev)
```

---

### المجموعة التانية: Reserved Regions

#### `of_iommu_get_resv_regions`

```c
extern void of_iommu_get_resv_regions(struct device *dev,
                                      struct list_head *list);
```

**بتعمل إيه:**
بتقرأ الـ **reserved memory regions** من الـ Device Tree وبتضيفهم لـ list. الـ reserved regions دي بتقول للـ IOMMU subsystem: "العناوين دي محجوزة — متعملش فيها map لأي device تاني." ده مهم جداً للـ firmware regions، الـ MSI doorbells، والـ memory اللي الـ bootloader حجزها.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي بنجيب regions الخاصة به |
| `list` | `struct list_head *` | الـ list اللي هنضيف عليها `iommu_resv_region` structs — الـ caller بيوفّرها |

**Return Value:**
`void` — بتضيف عناصر للـ list مباشرةً بدون return value. الـ caller مسؤول عن الـ cleanup (عبر `iommu_put_resv_regions()`).

**Key Details:**

- **Memory Allocation:** كل `iommu_resv_region` بيتعمله `kzalloc` داخلياً — الـ caller لازم يعمل `free` عليهم بعد كده.
- **DT Source:** بتقرأ من `reserved-memory` nodes في الـ DT اللي بيكون فيها `iommu-addresses` property.
- **Stub Behavior:** في الـ `!CONFIG_OF_IOMMU` version، الـ function جسم فاضي تماماً — الـ list بتفضل زي ما هي.
- **Side Effect:** الـ list بتكبر — كل region بتتضاف كـ `list_add_tail()` على الـ list اللي الـ caller بعتها.
- **Region Types:** الـ regions اللي بتتضاف ممكن تكون من نوع `IOMMU_RESV_DIRECT` (1:1 mapping إجباري) أو `IOMMU_RESV_RESERVED` (محجوز خالص).

**من بيستدعيها:**
بيتم الـ call منها عادةً من:
- `iommu_get_resv_regions(dev, list)` — الـ generic IOMMU API
- الـ IOMMU domain setup path لما بيتحدد إيه العناوين اللي محظورة
- الـ `dma_configure()` path في بعض الـ platform drivers

**Pseudocode Flow:**

```c
of_iommu_get_resv_regions(dev, list):
    // 1. اجيب الـ DT node بتاع الـ device
    np = dev->of_node

    // 2. امشي على كل iommu-addresses property
    for each reserved_region in np->iommu_addresses:
        // 3. اعمل alloc لـ region descriptor
        region = iommu_alloc_resv_region(start, length, prot, type)

        // 4. ضيفه للـ list
        list_add_tail(&region->list, list)
```

---

### ملاحظة على الـ Stub Pattern

الـ header بيستخدم الـ **dual-definition pattern** الشائع في الـ kernel:

```c
#ifdef CONFIG_OF_IOMMU
// Real functions declared as extern
extern int of_iommu_configure(...);
extern void of_iommu_get_resv_regions(...);
#else
// Stubs as static inline — compiler optimizes them away completely
static inline int of_iommu_configure(...) { return -ENODEV; }
static inline void of_iommu_get_resv_regions(...) { }
#endif
```

الـ `static inline` في الـ stubs معناها الـ compiler هيشيلهم تماماً في الـ `!CONFIG_OF_IOMMU` builds — **zero overhead**. الـ `-ENODEV` في `of_iommu_configure` بيقول للـ caller صراحةً: "مفيش IOMMU، تجاهل وكمّل."

---

### العلاقة بين الـ Functions والـ Subsystem

```
Device Tree (DT)
     │
     │  iommus = <&smmu 0x100>;
     │  iommu-addresses = <0x...>;
     ▼
of_iommu_configure()          ←── ربط device بالـ IOMMU driver
     │
     └──► iommu_probe_device()
               │
               └──► dev->iommu_group (set)
                         │
                         ▼
                    DMA API يشتغل صح

of_iommu_get_resv_regions()   ←── جلب العناوين المحجوزة
     │
     └──► list of iommu_resv_region
               │
               ▼
          IOMMU domain setup (no-map zones)
```

الاتنين مع بعض بيضمنوا إن أي device embedded في DT-based system:
1. يلاقي الـ IOMMU driver المناسب ويتربط بيه.
2. الـ IOMMU domain بتاعه بيعرف العناوين المحجوزة ومش هيعمل فيها map بالغلط.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو **OF-IOMMU** — الكود المسئول عن ربط الـ Device Tree بالـ IOMMU framework في الـ Linux kernel. الـ entry points الأساسية هي `of_iommu_configure()` و`of_iommu_get_resv_regions()`.

---

### Software Level

#### 1. debugfs — المداخل المهمة

الـ IOMMU framework بيعمل entries في `/sys/kernel/debug/iommu/`:

```bash
# اعرض كل الـ domains المسجلة
ls /sys/kernel/debug/iommu/

# لو الـ driver بيدعم debugfs (مثلاً ARM SMMU)
ls /sys/kernel/debug/iommu/arm-smmu/
cat /sys/kernel/debug/iommu/arm-smmu/masters

# اعرض الـ fwspec الخاص بجهاز معين
ls /sys/kernel/debug/iommu/<device-name>/
```

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/iommu/` | قائمة الـ IOMMU devices المسجلة |
| `/sys/kernel/debug/iommu/<dev>/mappings` | الـ IOVA mappings النشطة |
| `/sys/kernel/debug/iommu/<dev>/domain` | معلومات الـ domain |

#### 2. sysfs — المداخل المهمة

```bash
# تحقق إن الجهاز اتربط بـ IOMMU
cat /sys/bus/platform/devices/<dev>/iommu_group/type

# اعرض كل الـ IOMMU groups
ls /sys/kernel/iommu_groups/

# اعرض الأجهزة داخل group معين
ls /sys/kernel/iommu_groups/0/devices/

# تحقق من الـ reserved regions
cat /sys/kernel/iommu_groups/<N>/reserved_regions
```

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# فعّل الـ IOMMU events كلها
echo 1 > /sys/kernel/debug/tracing/events/iommu/enable

# او فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/iommu/attach_device_to_domain/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/detach_device_from_domain/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/map/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/unmap/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/io_page_fault/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# تتبع دالة of_iommu_configure تحديداً
echo 'of_iommu_configure' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

مثال على output مفيد:

```
          <idle>-0   [001] ....  123.456: iommu_attach_device_to_domain: device=10000000.ethernet
          <idle>-0   [001] ....  123.457: iommu_map: iova=0x00010000 size=0x1000 prot=3
```

#### 4. printk / Dynamic Debug

```bash
# فعّل كل الـ debug messages في of_iommu
echo 'file drivers/iommu/of_iommu.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug messages في كامل الـ iommu subsystem
echo 'file drivers/iommu/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق من اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep iommu

# على مستوى الـ kernel boot، أضف في command line:
# dyndbg="file drivers/iommu/of_iommu.c +p"
```

الـ `dev_dbg()` في `of_iommu_configure()` بتطبع:
```
of_iommu_configure: Adding to IOMMU failed: -19
```
**الـ -19 = -ENODEV** — يعني مفيش IOMMU driver مسجل.

#### 5. Kernel Config Options للـ Debugging

| الـ Config | الغرض |
|-----------|-------|
| `CONFIG_OF_IOMMU` | يفعّل الـ OF-IOMMU subsystem أصلاً |
| `CONFIG_IOMMU_DEBUG` | رسائل debug إضافية في الـ IOMMU core |
| `CONFIG_IOMMU_DEBUGFS` | يفعّل الـ debugfs entries للـ IOMMU |
| `CONFIG_ARM_SMMU` / `CONFIG_ARM_SMMU_V3` | الـ driver للـ ARM SMMU |
| `CONFIG_IOMMU_DMA` | الـ DMA-IOMMU integration |
| `CONFIG_OF_ADDRESS` | مطلوب لـ `of_iommu_get_resv_regions()` |
| `CONFIG_DEBUG_SHIRQ` | debug للـ shared interrupts |
| `CONFIG_PROVE_LOCKING` | يكتشف locking bugs في الـ mutex |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg()` و`pr_debug()` |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'IOMMU|OF_IOMMU'
```

#### 6. devlink وأدوات خاصة بالـ Subsystem

```bash
# اعرض الـ IOMMU info عبر dmesg أثناء boot
dmesg | grep -i iommu
dmesg | grep -i 'of_iommu\|iommu-map\|iommus'

# استخدم iommufd (للـ kernels الأحدث)
ls /dev/iommu

# تحقق من الـ fwnode handles
dmesg | grep 'fwnode\|of_xlate\|iommu_fwspec'

# لرؤية الـ PCI aliases
lspci -vv | grep -i 'iommu\|ats'
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة في الـ log | المعنى | الحل |
|-----------------|--------|------|
| `Adding to IOMMU failed: -19` | مفيش IOMMU driver مسجل (`-ENODEV`) | تأكد من `CONFIG_ARM_SMMU` أو الـ driver المناسب |
| `Adding to IOMMU failed: -517` | الـ driver لسه ما اتحملش (`-EPROBE_DEFER`) | طبيعي في الـ boot — انتظر أو تحقق من ترتيب الـ initcalls |
| `failed to parse memory region` | الـ `reg` property في الـ DT بايظ | راجع الـ DT binding للـ reserved-memory |
| `treating non-direct mapping as reservation` | الـ IOVA مش مطابق للـ physical | راجع `iommu-addresses` في الـ DT |
| `Cannot reserve IOVA region of 0 size` | الـ `iommu-addresses` بيحدد size = 0 | تحقق من الـ DT node للـ memory-region |
| `iommu_fwspec_init failed` | فشل الـ allocation أو mismatch في الـ fwnode | تحقق من الـ memory وإن الـ IOMMU node موجود |
| `of_xlate returned error` | الـ IOMMU driver رفض الـ stream ID | تحقق من صحة `iommus = <&smmu SID>` في الـ DT |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

الأماكن المناسبة في `drivers/iommu/of_iommu.c`:

```c
/* في of_iommu_xlate() — لو مفيش of_xlate callback */
if (!ops->of_xlate) {
    WARN_ON(!ops->of_xlate); /* الـ driver ناقص implementation */
    return -ENODEV;
}

/* في of_iommu_configure() — لو الـ fwspec اتبنى بدون bus */
if (!err && dev->bus && !dev_iommu_present) {
    /* هنا تقدر تحط dump_stack() لو iommu_probe_device فشل */
}

/* في of_iommu_get_resv_regions() — لو الـ region فشل */
region = iommu_alloc_resv_region(...);
if (!region) {
    WARN_ON_ONCE(!region); /* OOM situation */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State مطابق للـ Kernel State

```bash
# تحقق من وجود الـ IOMMU في الـ Device Tree المحمّل
dtc -I fs /sys/firmware/devicetree/base | grep -A5 'iommu'

# قارن الـ stream IDs في الـ DT مع اللي في الـ driver
dmesg | grep 'stream\|sid\|SID'

# تحقق من الـ SMMU registers — هل الـ device attached؟
dmesg | grep 'smmu.*attach\|context bank'
```

#### 2. Register Dump

```bash
# افترض إن الـ SMMU base address = 0x09000000 (مثال ARM SMMU)
# باستخدام devmem2
devmem2 0x09000000 w    # CR0 register — هل الـ SMMU enabled؟
devmem2 0x09000008 w    # STATUSR

# قراءة نطاق من الـ registers
for offset in 0 4 8 c 10 14; do
    echo -n "offset 0x$offset: "
    devmem2 $((0x09000000 + 0x$offset)) w 2>/dev/null
done

# عبر /dev/mem
dd if=/dev/mem bs=4 count=16 skip=$((0x09000000/4)) 2>/dev/null | xxd
```

#### 3. Logic Analyzer / Oscilloscope

لو بتشك في مشكلة hardware-level في الـ IOMMU interconnect:

- **راقب الـ AXI/ACE bus**: تأكد إن الـ transaction IDs (Stream IDs) بتوصل صح للـ SMMU
- **ATS (Address Translation Service) للـ PCIe**: راقب الـ ATS Request/Completion على الـ PCIe bus
- **TLB Invalidation**: راقب الـ SMMU CMD_SYNC و TLBI commands على الـ interconnect
- **نقاط القياس**: قبل وبعد الـ SMMU في الـ memory interconnect

#### 4. مشاكل Hardware شائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في الـ Log |
|---------|-------------------|
| الـ SMMU مش مفعّل في الـ hardware | `iommu: Failed to probe smmu` |
| Stream ID خاطئ (mismatch بين DT والـ HW) | `arm-smmu: Unhandled context fault` + `FSR=0x...` |
| الـ ATS مش مدعوم في الـ hardware | `ats-supported` في الـ DT بس الـ PCIe EP رافض |
| الـ reserved memory متداخل | `iommu: mapping already present` أو page fault |
| Power management issue | الـ SMMU بيتقفل بعد suspend وما بيرجعش |

```bash
# اقرأ الـ context fault info للـ ARM SMMU
dmesg | grep -E 'arm-smmu|context fault|FSR|FAR|FSYNR'
```

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT المحمّل كاملاً وابحث عن iommu bindings
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B5 -A10 'iommus'

# تحقق من وجود iommu-map (للـ PCI)
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B2 -A5 'iommu-map'

# تحقق من الـ #iommu-cells في الـ SMMU node
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B10 '#iommu-cells'

# تحقق من الـ reserved-memory nodes
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -A20 'reserved-memory'

# تحقق من الـ iommu-addresses في الـ memory-region nodes
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B2 -A8 'iommu-addresses'
```

**مثال DT صح لجهاز مربوط بـ SMMU:**

```dts
/* الـ SMMU node */
smmu: iommu@9000000 {
    compatible = "arm,smmu-v3";
    reg = <0x9000000 0x800000>;
    #iommu-cells = <1>;   /* مطلوب لـ of_iommu_configure_dev */
};

/* الجهاز اللي بيستخدم الـ SMMU */
ethernet@10000000 {
    compatible = "vendor,eth";
    reg = <0x10000000 0x1000>;
    iommus = <&smmu 0x100>;  /* Stream ID = 0x100 */
    memory-region = <&eth_reserved>;
};

/* الـ reserved memory */
reserved-memory {
    eth_reserved: eth-buffer@80000000 {
        reg = <0x80000000 0x1000000>;
        iommu-addresses = <&ethernet 0x0 0x80000000 0x0 0x1000000>;
    };
};
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === OF-IOMMU Debug Script ===

DEVICE=${1:-"10000000.ethernet"}  # غيّر للـ device name

echo "=== [1] dmesg IOMMU Messages ==="
dmesg | grep -iE 'iommu|smmu|of_iommu' | tail -50

echo ""
echo "=== [2] IOMMU Groups ==="
ls /sys/kernel/iommu_groups/
for g in /sys/kernel/iommu_groups/*/; do
    echo "Group $(basename $g):"
    ls $g/devices/
done

echo ""
echo "=== [3] Device IOMMU Group ==="
readlink /sys/bus/platform/devices/$DEVICE/iommu_group 2>/dev/null || \
    echo "Device not in any IOMMU group!"

echo ""
echo "=== [4] Reserved Regions ==="
for g in /sys/kernel/iommu_groups/*/; do
    cat $g/reserved_regions 2>/dev/null
done

echo ""
echo "=== [5] Device Tree IOMMU Bindings ==="
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B5 -A5 'iommus\|iommu-map\|iommu-addresses' 2>/dev/null | head -100

echo ""
echo "=== [6] Enable IOMMU ftrace Events ==="
echo 1 > /sys/kernel/debug/tracing/events/iommu/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "Tracing enabled. Run: cat /sys/kernel/debug/tracing/trace_pipe"
```

#### تفسير output مهم

```bash
# لو شفت في dmesg:
# [    2.345678] platform 10000000.ethernet: Adding to IOMMU failed: -19
# المعنى: CONFIG_ARM_SMMU مش مفعّل أو الـ SMMU driver ما اتحملش

# لو شفت:
# [    2.345678] platform 10000000.ethernet: Adding to IOMMU failed: -517
# المعنى: طبيعي — الـ SMMU driver لسه ما اتحملش، الـ kernel هيحاول تاني

# لو شفت:
# [    2.345678] arm-smmu-v3 9000000.iommu: Unhandled context fault
# المعنى: جهاز بيحاول يوصل لعنوان مش mapped — راجع الـ DMA mappings
```

```bash
# تفعيل dynamic debug وتتبع الـ of_iommu_configure
echo 'file drivers/iommu/of_iommu.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# الـ +pflmt معناها:
#   p = print
#   f = function name
#   l = line number
#   m = module name
#   t = thread ID

# راقب النتيجة
dmesg -w | grep of_iommu
```

```bash
# تحقق من الـ IOMMU fwspec لجهاز معين (عبر kernel oops أو debugfs)
# لو عندك kernel بـ CONFIG_IOMMU_DEBUGFS:
cat /sys/kernel/debug/iommu/devices 2>/dev/null

# أو استخدم crash/gdb مع vmlinux:
# p ((struct iommu_fwspec *)dev->iommu->fwspec)->num_ids
# p ((struct iommu_fwspec *)dev->iommu->fwspec)->ids[0]
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ IOMMU مش بيتفعّل

#### العنوان
الـ DMA isolation مش شغّال على gateway صناعي بسبب غياب `iommus` property في الـ DT

#### السياق
شركة بتبني **industrial gateway** على أساس **RK3562** بيشغّل Linux 6.x. المنتج بيستخدم Ethernet controller و USB host controller عشان ينقل بيانات من sensors. الـ security team طلبت إن كل peripheral يشتغل في **IOMMU domain** منعزل عشان لو في exploit في driver، ما يقدرش يوصل لباقي الـ memory.

#### المشكلة
الـ engineer لاحظ إن الـ USB controller شغّال بس بدون IOMMU protection. الـ dmesg مفيهوش أي رسالة تقول إن IOMMU اتكوّن للـ USB device.

```bash
dmesg | grep -i iommu
# النتيجة: مفيش أي إشارة لـ USB في الـ IOMMU log
```

#### التحليل
الـ kernel بيمشي على المسار ده:

```
probe(usb_hcd)
  → iommu_probe_device()
    → of_iommu_configure(dev, dev->of_node, NULL)   ← نقطة الدخول في of_iommu.h
```

الـ `of_iommu_configure()` بتدوّر على `iommus` property في الـ device node:

```c
/* of_iommu.c — simplified */
int of_iommu_configure(struct device *dev, struct device_node *master_np,
                       const u32 *id)
{
    /* بتدور على iommus = <&iommu_node id> في الـ DT */
    err = of_iommu_configure_device(master_np, dev, &info);
    /* لو مفيش property → ترجع -ENODEV وما بتعملش حاجة */
}
```

لما فتح الـ DTS لقى:

```dts
/* قبل التعديل — USB node بدون iommus */
usbhost: usb@fe800000 {
    compatible = "rockchip,rk3562-xhci";
    reg = <0x0 0xfe800000 0x0 0x100000>;
    /* iommus property غايبة! */
};
```

لأن `CONFIG_OF_IOMMU` موجود في الـ kernel config، الـ real function اتكلمت، بس رجعت `-ENODEV` لأن مفيش `iommus` في الـ DT node. لو `CONFIG_OF_IOMMU` كانت غايبة، كانت الـ stub inline في الـ header هي اللي اتكلمت وكمان رجعت `-ENODEV` — نفس النتيجة.

#### الحل
إضافة `iommus` property في الـ DTS:

```dts
/* بعد التعديل */
iommu: iommu@fe010000 {
    compatible = "rockchip,rk3568-iommu"; /* RK3562 compatible */
    reg = <0x0 0xfe010000 0x0 0x1000>;
    #iommu-cells = <0>;
};

usbhost: usb@fe800000 {
    compatible = "rockchip,rk3562-xhci";
    reg = <0x0 0xfe800000 0x0 0x100000>;
    iommus = <&iommu>;   /* ← ده اللي بيخلي of_iommu_configure تلاقي حاجة */
};
```

```bash
# تحقق بعد التعديل
dmesg | grep -i iommu
# [    2.345] rockchip-iommu fe010000.iommu: bound usb@fe800000 (ops rockchip_iommu_ops)
```

#### الدرس المستفاد
الـ `of_iommu_configure()` في `of_iommu.h` هي **entry point** بس — هي مش بتفعّل IOMMU لوحدها، بتعتمد **100%** على وجود `iommus` property في الـ DT. غيابها = سكوت تام بدون error واضح للـ user.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ Mali GPU بييجي panic عند الـ boot

#### العنوان
`of_iommu_get_resv_regions()` بتسبب kernel panic لأن الـ IOMMU driver مش loaded قبل الـ GPU

#### السياق
منتج **Android TV box** بيستخدم **Allwinner H616** مع Mali-G31 GPU. الـ engineer بيعمل bring-up لـ Android 13. الـ GPU driver محتاج IOMMU عشان يعمل memory isolation للـ graphics buffers.

#### المشكلة
الـ kernel بيعمل panic أثناء الـ boot:

```
[    3.112] Unable to handle kernel NULL pointer dereference at virtual address 0000000000000018
[    3.113] Call trace:
[    3.113]  of_iommu_get_resv_regions+0x44/0xb0
[    3.114]  iommu_get_resv_regions+0x2c/0x60
[    3.115]  arm_smmu_probe+0x3a8/0x890
```

#### التحليل
الـ `of_iommu_get_resv_regions()` المُعرّفة في `of_iommu.h` (والـ implementation في `of_iommu.c`) بتمشي على الـ DT عشان تجيب **reserved memory regions** اللي الـ IOMMU محتاجها:

```c
/* of_iommu.c */
void of_iommu_get_resv_regions(struct device *dev, struct list_head *list)
{
    struct device_node *np = dev->of_node;
    /* بتدوّر على reserved-memory في الـ DT وبتضيفها للـ list */
    /* لو الـ ops pointer null → NULL deref */
}
```

الـ panic بييجي لأن الـ IOMMU ops structure اتفعّل قبل ما الـ IOMMU driver نفسه يخلّص الـ init. الـ `of_iommu.h` بتوفّر الـ declaration، لكن الـ implementation بتـ assume إن الـ iommu_ops مش NULL.

الترتيب الخاطئ في الـ DT:

```dts
/* الـ GPU جاي قبل الـ IOMMU في الـ DT — ده بيأثر على probe order */
gpu@1800000 {
    compatible = "allwinner,sun50i-h616-mali";
    iommus = <&mmu_aw 2>;   /* بيرفر على IOMMU اللي ما اتـ probe دليه */
};

mmu_aw: iommu@30f0000 {
    compatible = "allwinner,sun50i-h616-iommu";
};
```

#### الحل
إما ترتيب الـ DT عشان الـ IOMMU يتـ probe أول (الـ kernel بيأخد الـ nodes بالترتيب كأساس):

```dts
/* الـ IOMMU الأول */
mmu_aw: iommu@30f0000 {
    compatible = "allwinner,sun50i-h616-iommu";
    reg = <0x030f0000 0x10000>;
    #iommu-cells = <1>;
};

gpu@1800000 {
    compatible = "allwinner,sun50i-h616-mali";
    iommus = <&mmu_aw 2>;
};
```

أو استخدام `deferred probe` عن طريق التأكد إن الـ driver بيرجع `EPROBE_DEFER` لو الـ IOMMU ما اتـ probe:

```bash
# تحقق من probe order
cat /sys/kernel/debug/devices_deferred
```

#### الدرس المستفاد
الـ `of_iommu_get_resv_regions()` في الـ header بتتكلم من سياق الـ device probe. لو الـ IOMMU provider ما اتـ probe لسه، الـ ops تبقى NULL. ترتيب الـ nodes في الـ DTS بيأثر على الـ probe order، والـ driver لازم يتعامل مع `EPROBE_DEFER`.

---

### السيناريو 3: IoT Sensor Hub على STM32MP1 — الـ `CONFIG_OF_IOMMU` مش معموله enable

#### العنوان
الـ stub function في `of_iommu.h` بترجع `-ENODEV` وبتـ block الـ SPI driver من الشغل

#### السياق
**IoT sensor hub** بيستخدم **STM32MP157** بيشغّل buildroot Linux. الـ product بيقرأ من 4 SPI sensors. الـ engineer عمل custom kernel config خفيف جداً عشان يقلل الـ boot time.

#### المشكلة
الـ SPI controller مش بيـ probe:

```bash
dmesg | grep spi
# [    1.234] spi-stm32 44009000.spi: error -19 (ENODEV) configuring IOMMU
# [    1.235] spi-stm32 44009000.spi: probe failed
```

Error code `-19` هو `-ENODEV`.

#### التحليل
لما `CONFIG_OF_IOMMU` مش موجود في الـ `.config`، الـ header بيستخدم الـ **stub inline function**:

```c
/* of_iommu.h — الـ stub اللي بتتكلم لما CONFIG_OF_IOMMU=n */
static inline int of_iommu_configure(struct device *dev,
                                     struct device_node *master_np,
                                     const u32 *id)
{
    return -ENODEV;   /* دايماً بترجع error */
}
```

الـ `iommu_probe_device()` في الـ IOMMU core بتكلم `of_iommu_configure()` وبتتعامل مع الـ return value. بعض الـ kernel versions بتعتبر `-ENODEV` يعني "مفيش IOMMU مطلوب" وبتكمّل، بس في الـ STM32MP1 الـ SPI driver نفسه كان بيعمل explicit check:

```c
/* في stm32_spi_probe() */
ret = iommu_probe_device(dev);
if (ret && ret != -ENODEV) {
    dev_err(dev, "error %d configuring IOMMU\n", ret);
    return ret;
}
/* المشكلة: الـ condition الغلطانة — كان المفروض يكون (ret < 0 && ret != -ENODEV) */
```

#### الحل
الـ fix بييجي على مستويين:

**1. الـ kernel config** — إما تفعّل `CONFIG_OF_IOMMU=y` لو الـ SoC بيدعمه:

```bash
# في menuconfig
CONFIG_OF_IOMMU=y
CONFIG_ARM_SMMU=y   # أو الـ driver المناسب لـ STM32MP1
```

**2. أو صلّح الـ driver check** عشان يتعامل صح مع `-ENODEV`:

```c
ret = iommu_probe_device(dev);
if (ret == -ENODEV)
    ret = 0;  /* مفيش IOMMU = مش error */
if (ret) {
    dev_err(dev, "IOMMU error %d\n", ret);
    return ret;
}
```

#### الدرس المستفاد
الـ stub في `of_iommu.h` موجودة عشان الـ code يـ compile بدون IOMMU support، بس الـ `-ENODEV` اللي بترجعه **مش دايماً معناها مش محتاج** — بعض الـ drivers بتـ misinterpret الـ return code. الـ engineer لازم يفرّق بين "مفيش IOMMU" و"حصل error".

---

### السيناريو 4: Automotive ECU على i.MX8QM — الـ reserved regions مش بتتـ map صح

#### العنوان
الـ `of_iommu_get_resv_regions()` مش بتجيب الـ firmware reserved memory على i.MX8QM

#### السياق
**automotive ECU** بيستخدم **NXP i.MX8QM** بيشغّل Linux مع AUTOSAR بجانبه على cores تانية. الـ SCU (System Control Unit) بتحجز regions في الـ memory عشان الـ firmware. الـ Linux IOMMU لازم يعرف بيها عشان ما يـ mapش فوقيها.

#### المشكلة
الـ camera ISP driver بيـ crash لأنه بيحاول يـ map address فيه firmware data:

```
[    5.678] arm-smmu-v3 51400000.iommu: fault on device: 1 on address 0x88000000
[    5.679] iommu fault: page fault
```

#### التحليل
الـ `of_iommu_get_resv_regions()` بتـ read الـ reserved regions من الـ DT:

```c
/* of_iommu.c */
void of_iommu_get_resv_regions(struct device *dev, struct list_head *list)
{
    /* بتدوّر على nodes من نوع:
       - memory-region
       - no-map
       - iommu-addresses
    */
}
```

الـ DT على i.MX8QM كان ناقصه `iommu-addresses` للـ SCU reserved region:

```dts
/* قبل التعديل */
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;

    scu_reserved: scu@88000000 {
        reg = <0x0 0x88000000 0x0 0x200000>;
        no-map;
        /* مفيش iommu-addresses — of_iommu_get_resv_regions مش بتشوفها */
    };
};

isp: isp@58000000 {
    compatible = "nxp,imx8qm-isi";
    /* مفيش ربط بين الـ ISP وأي reserved region */
};
```

الـ `of_iommu_get_resv_regions()` اتكلمت من سياق الـ ISP device probe، بس ما لقتش `iommu-addresses` في الـ ISP node ولا في الـ reserved memory nodes، فما أضافتش أي region للـ list.

#### الحل

```dts
reserved-memory {
    #address-cells = <2>;
    #size-cells = <2>;

    scu_reserved: scu@88000000 {
        reg = <0x0 0x88000000 0x0 0x200000>;
        no-map;
    };
};

isp: isp@58000000 {
    compatible = "nxp,imx8qm-isi";
    iommus = <&smmu 0x4>;
    /* إضافة iommu-addresses عشان of_iommu_get_resv_regions تلاقيها */
    iommu-addresses = <&scu_reserved 0x0 0x88000000 0x0 0x200000>;
};
```

```bash
# تحقق إن الـ regions اتـ register
cat /sys/kernel/debug/iommu/devices/isp@58000000/reserved_regions
```

#### الدرس المستفاد
الـ `of_iommu_get_resv_regions()` بتشتغل على المعلومات اللي في الـ DT **بس**. لو الـ firmware أو الـ bootloader حجز memory بس ما ذكرهاش في الـ DT كـ `iommu-addresses`، الـ kernel مش هيعرف بيها والـ IOMMU هيسمح بالـ map فيها — وده بيأدي لـ corruption أو fault.

---

### السيناريو 5: Custom Board Bring-Up على AM62x — الـ IOMMU configure بيفشل صامت

#### العنوان
الـ `of_iommu_configure()` بترجع success بس الـ device مش داخل أي IOMMU domain فعلي

#### السياق
فريق بيعمل bring-up لـ custom board مبنية على **TI AM62x** (AM6254). الـ board بتستخدم الـ onboard **CPSW Ethernet** controller. الـ TI SDK بيستخدم IOMMU (TISCI-based) عشان يعمل isolation للـ networking subsystem.

#### المشكلة
الـ Ethernet يشتغل عادي، بس الـ security audit لقى إن الـ CPSW مش واقع تحت أي IOMMU domain رغم إن الـ kernel config فيه `CONFIG_OF_IOMMU=y` و `CONFIG_TI_IOMMU=y`:

```bash
cat /sys/kernel/debug/iommu/devices/*/domain
# مفيش الـ CPSW في القائمة
```

#### التحليل
الـ `of_iommu_configure()` اتكلمت بنجاح (رجعت 0)، بس الـ IOMMU ما اتـ attachش فعلاً. السبب: في الـ AM62x DTS اللي ورثوه من الـ TI reference design، الـ `iommus` property كانت موجودة بس بـ `status = "disabled"` على الـ IOMMU node نفسه:

```dts
/* am62x.dtsi — الـ base file من TI */
main_navss_iommu: iommu@30f00000 {
    compatible = "ti,am654-iommu";
    reg = <0x00 0x30f00000 0x00 0x100>;
    status = "disabled";   /* ← معمول disable في الـ base! */
    #iommu-cells = <4>;
};

cpsw: ethernet@8000000 {
    compatible = "ti,am642-cpsw-nuss";
    iommus = <&main_navss_iommu 0 1 1 1>;
};
```

لما الـ `of_iommu_configure()` بتشتغل، بتكلم:

```c
int of_iommu_configure(struct device *dev, struct device_node *master_np,
                       const u32 *id)
{
    /* بتـ lookup الـ IOMMU provider */
    iommu_np = of_parse_phandle(master_np, "iommus", 0);
    /* لو الـ node موجود بس status=disabled → الـ provider مش registered */
    /* الـ function بترجع -EPROBE_DEFER أو -ENODEV بدون error واضح */
}
```

بما إن الـ IOMMU node نفسه `disabled`، الـ driver ما اتـ probe أصلاً، والـ ops ما اتـ register، والـ `of_iommu_configure()` رجعت بهدوء.

#### الحل
في الـ board-specific DTS overlay:

```dts
/* am62x-custom-board.dts */
#include "am62x.dtsi"

/* تفعيل الـ IOMMU */
&main_navss_iommu {
    status = "okay";   /* ← ده بس اللي كان ناقص */
};
```

```bash
# بعد التعديل
dmesg | grep iommu
# [    1.890] ti,am654-iommu 30f00000.iommu: registered

cat /sys/kernel/debug/iommu/devices/ethernet@8000000/domain
# 30f00000.iommu:domain-0
```

#### الدرس المستفاد
الـ `of_iommu_configure()` بتـ depend على إن الـ IOMMU provider node يكون `status = "okay"` وإن الـ driver بتاعه اتـ probe بنجاح. الـ header بيكشف الـ API فقط — الـ silent failure بييجي من الـ DT configuration مش من الـ code. في الـ board bring-up، دايماً افحص `status` في الـ IOMMU node الـ base قبل ما تفترض إن الـ IOMMU شغال.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**LWN.net** هو المرجع الأول لمتابعة تطور kernel subsystems. الروابط دي اتأكدت من نتائج البحث الفعلية:

| المقال | الأهمية |
|--------|---------|
| [mm: iommu: An API to unify IOMMU, CPU and device memory management](https://lwn.net/Articles/395142/) | الأساس النظري لـ IOMMU API في الـ kernel |
| [Add NVIDIA Tegra124 IOMMU support](https://lwn.net/Articles/603810/) | مثال عملي لـ `of_iommu_configure` مع device tree bindings |
| [Exynos SYSMMU (IOMMU) integration with DT and DMA-mapping subsystem](https://lwn.net/Articles/607626/) | تكامل IOMMU مع device tree و DMA-mapping — مثال مباشر لـ `of_iommu` |
| [Apple M1 DART IOMMU driver](https://lwn.net/Articles/849968/) | driver حديث بيستخدم device tree لتكوين IOMMU |
| [Linux RISC-V IOMMU Support](https://lwn.net/Articles/972035/) | تطبيق IOMMU على معمارية جديدة بـ device tree bindings |
| [IOMMU user API enhancement](https://lwn.net/Articles/824306/) | تطور الـ API للـ userspace |
| [Introduce /dev/iommu for userspace I/O address space management](https://lwn.net/Articles/869818/) | مستقبل إدارة IOMMU من userspace |
| [vfio/mdev: IOMMU aware mediated device](https://lwn.net/Articles/780522/) | استخدام IOMMU في سياق الـ virtualization |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في source tree مباشرةً:

```
Documentation/devicetree/bindings/iommu/iommu.txt   ← DT bindings للـ iommus property
Documentation/driver-api/iommu.rst                  ← IOMMU subsystem API
Documentation/admin-guide/kernel-parameters.txt     ← iommu= parameters
Documentation/userspace-api/iommufd.rst             ← IOMMUFD user API
```

**الـ** kernel documentation على الويب:
- [IOMMU DT Bindings](https://www.kernel.org/doc/Documentation/devicetree/bindings/iommu/iommu.txt) — بيشرح الـ `iommus` property اللي `of_iommu_configure` بتقراها
- [x86 IOMMU Support](https://docs.kernel.org/arch/x86/iommu.html) — مرجع للمقارنة مع platform-agnostic approach
- [IOMMUFD Documentation](https://static.lwn.net/kerneldoc/userspace-api/iommufd.html) — الاتجاه الحديث

---

### Source Files المهمة في الـ Kernel

الملفات دي مترابطة مباشرةً بـ `of_iommu.h`:

```
include/linux/of_iommu.h          ← الملف الأساسي (موضوع الدراسة)
drivers/iommu/of_iommu.c          ← التنفيذ الكامل لـ of_iommu_configure و of_iommu_get_resv_regions
drivers/iommu/iommu.c             ← IOMMU core subsystem
include/linux/iommu.h             ← iommu_ops struct و core API
drivers/iommu/Kconfig             ← CONFIG_OF_IOMMU و dependencies
```

**على GitHub:**
- [drivers/iommu/of_iommu.c](https://github.com/torvalds/linux/blob/master/drivers/iommu/of_iommu.c)
- [drivers/iommu/iommu.c](https://github.com/torvalds/linux/blob/master/drivers/iommu/iommu.c)
- [drivers/iommu/Kconfig](https://github.com/torvalds/linux/blob/master/drivers/iommu/Kconfig)

---

### Kernel Commits المهمة

**الـ** commits دي شكّلت الـ `of_iommu` subsystem:

| الـ Commit | الوصف |
|-----------|-------|
| [`8b0b0a0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/iommu/of_iommu.c) | تاريخ ملف `of_iommu.c` كامل عبر `git log` |
| [IOMMU of_iommu initial integration](https://lkml.org/) | البحث في LKML عن `of_iommu_configure` يطلع الـ patch series الأصلية |

للحصول على الـ commits الفعلية:
```bash
git log --oneline drivers/iommu/of_iommu.c
git log --oneline include/linux/of_iommu.h
```

---

### Mailing List — نقاشات LKML

- **[LKML Archive](https://lkml.org/)** — ابحث عن `of_iommu_configure` أو `CONFIG_OF_IOMMU`
- **[Patchwork — IOMMU](https://patchwork.kernel.org/project/linux-iommu/list/)** — patches الـ iommu subsystem الرسمية
- **Lore Kernel** — `https://lore.kernel.org/linux-iommu/` — أرشيف كامل لـ mailing list الـ iommu

نقاشات مهمة ابحث عنها:
- `[PATCH] iommu/of: Add of_iommu_get_resv_regions`
- `[PATCH] of/iommu: Add OF_IOMMU Kconfig option`
- `[PATCH] iommu: Add of_iommu_configure() for platform devices`

---

### Kernelnewbies.org — تطور IOMMU عبر الإصدارات

**الـ** kernelnewbies بيوثق التغييرات في كل إصدار kernel:

| الإصدار | التغيير المتعلق بـ IOMMU |
|---------|--------------------------|
| [Linux 5.3](https://kernelnewbies.org/Linux_5.3) | إضافة `virtio-iommu` driver — para-virtualized IOMMU |
| [Linux 5.16](https://kernelnewbies.org/Linux_5.16) | تحسينات IOMMU subsystem |
| [Linux 6.2](https://kernelnewbies.org/Linux_6.2) | إضافة `iommufd` — user API جديد لإدارة IO page tables |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | تطوير IOMMU nested translation |
| [Linux 6.7](https://kernelnewbies.org/Linux_6.7) | دعم dirty tracking عبر IOPTEs للـ VM migration |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | IO page fault delivery لـ userspace عبر IOMMUFD |

---

### eLinux.org — IOMMU في الـ Embedded Linux

**الـ** [Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) على eLinux.org بتشمل:
- عرض Laurent Pinchart (Renesas) عن DMA API و IOMMU integration
- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — مصادر عامة للـ kernel subsystems

للـ embedded platforms المهتمة بـ `of_iommu`:
- [BeagleBoard DSP Clarification](https://elinux.org/BeagleBoard/DSP_Clarification) — IOMMU في سياق OMAP/DSP

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 15: Memory Mapping and DMA** — بيشرح DMA addressing و IOMMU role
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- أهم sections: `dma_map_single`, `dma_set_mask`, DMA coherency

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 19: Portability** — device abstraction و platform independence
- **الفصل 12: Memory Management** — virtual/physical address mapping
- الكتاب بيرسم الصورة الكاملة للـ kernel architecture اللي `of_iommu` جزء منها

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 7: Bootloaders** — Device Tree و `iommus` property
- **الفصل 14: Device Drivers** — platform drivers و DT integration
- مهم لفهم كيف `of_iommu_configure` بتُستدعى أثناء الـ boot

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 18: Page Tables** — address translation concepts مرتبطة بـ IOMMU
- بيشرح الفرق بين CPU MMU و IOMMU

---

### مصادر إضافية

#### Lenovo Press — Technical Paper
- [An Introduction to IOMMU Infrastructure in the Linux Kernel](https://lenovopress.lenovo.com/lp1467-an-introduction-to-iommu-infrastructure-in-the-linux-kernel) — شرح شامل للـ IOMMU infrastructure مع أمثلة عملية
- [PDF version](https://lenovopress.lenovo.com/lp1467.pdf)

#### Kernel Documentation Online
- [x86 Intel IOMMU](https://www.kernel.org/doc/html/v5.8/x86/intel-iommu.html) — مرجع implementation reference

---

### Search Terms للبحث عن معلومات أكثر

استخدم الـ search terms دي للحصول على نتائج دقيقة:

```
# للبحث في LKML
site:lore.kernel.org of_iommu_configure
site:lkml.org "of_iommu_get_resv_regions"

# للبحث عن DT bindings
"iommus" "iommu-map" device tree binding linux

# للبحث عن implementations
of_iommu_configure platform_device ARM SMMU
CONFIG_OF_IOMMU linux kernel

# للبحث عن commits
git.kernel.org of_iommu.c history
github torvalds linux of_iommu

# للبحث عن articles
lwn.net "iommu_ops" device tree ARM SMMU
lwn.net of_iommu platform bus IOMMU configure
```

---

### خريطة الـ Subsystems المرتبطة

```
of_iommu.h
    │
    ├── OF (Open Firmware / Device Tree)
    │       Documentation/devicetree/bindings/iommu/
    │
    ├── IOMMU Core
    │       Documentation/driver-api/iommu.rst
    │       include/linux/iommu.h
    │
    ├── DMA Mapping
    │       Documentation/core-api/dma-api.rst
    │       include/linux/dma-mapping.h
    │
    └── Platform Bus
            Documentation/driver-api/driver-model/platform.rst
```

كل subsystem من دول عنده مصادره الخاصة — الـ `of_iommu` هو نقطة التقاطع بينهم.
## Phase 8: Writing simple module

### الفكرة

الـ header الـ `of_iommu.h` بيعرف فانكشنين exported هما `of_iommu_configure` و `of_iommu_get_resv_regions`. الأنسب للـ hook هو **`of_iommu_configure`** لأنه بيتكلم كل ما device بتحاول تتربط بـ IOMMU عبر Device Tree — ده نقطة حيوية لمراقبة أي device بيدخل تحت حماية IOMMU.

هنستخدم **kprobe** لأن `of_iommu_configure` فانكشن عادية exported وملهاش notifier خاص بيها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, pre_handler */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/of.h>           /* struct device_node, of_node_full_name() */
#include <linux/printk.h>       /* pr_info */

/*
 * pre_handler: called just before of_iommu_configure() executes.
 * Arguments mirror of_iommu_configure(dev, master_np, id):
 *   regs->di  (x86-64 rdi) = dev
 *   regs->si  (x86-64 rsi) = master_np
 *   regs->dx  (x86-64 rdx) = id
 *
 * We cast them out of pt_regs using the standard kprobe ABI helpers.
 */
static int of_iommu_configure_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * نجيب البوينترات من الـ registers حسب calling convention x86-64:
     * rdi → أول argument (dev), rsi → تاني (master_np), rdx → تالت (id)
     */
    struct device      *dev       = (struct device *)regs->di;
    struct device_node *master_np = (struct device_node *)regs->si;
    const u32          *id        = (const u32 *)regs->dx;

    /* نتأكد إن البوينترات مش NULL قبل ما نقرأ منهم */
    const char *dev_name_str = dev       ? dev_name(dev)                 : "(null)";
    const char *np_name      = master_np ? of_node_full_name(master_np)  : "(null)";

    if (id)
        pr_info("of_iommu_configure: dev=%s node=%s id=0x%x\n",
                dev_name_str, np_name, *id);
    else
        pr_info("of_iommu_configure: dev=%s node=%s id=(null)\n",
                dev_name_str, np_name);

    return 0; /* 0 = استمر في تنفيذ الفانكشن الأصلية */
}

/* تعريف الـ kprobe وربطه بالفانكشن المستهدفة بالاسم */
static struct kprobe kp = {
    .symbol_name = "of_iommu_configure",
    .pre_handler = of_iommu_configure_pre,
};

static int __init of_iommu_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_iommu_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("of_iommu_probe: hooked of_iommu_configure at %p\n", kp.addr);
    return 0;
}

static void __exit of_iommu_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_iommu_probe: kprobe removed\n");
}

module_init(of_iommu_probe_init);
module_exit(of_iommu_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on of_iommu_configure to trace IOMMU DT binding");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يوفر `struct kprobe`، `register_kprobe`، `unregister_kprobe`، وتعريف `pre_handler` |
| `linux/device.h` | عشان نستخدم `struct device` و `dev_name()` لطباعة اسم الـ device |
| `linux/of.h` | عشان `struct device_node` و `of_node_full_name()` اللي بترجع المسار الكامل في الـ Device Tree |
| `linux/printk.h` | الـ `pr_info` / `pr_err` للـ logging |

---

#### الـ `pre_handler`

الـ `pre_handler` بيتشغل **قبل** ما الـ CPU ينفذ أول instruction في `of_iommu_configure`. الـ kernel بيمرر `struct pt_regs *regs` اللي فيه حالة الـ registers لحظة الـ call — فبنقدر نجيب الـ arguments منه مباشرة حسب ABI.

بنطبع اسم الـ device، مسار الـ node في الـ Device Tree، والـ stream ID لو موجود — ده بيديك صورة واضحة لكل device بيتربط بـ IOMMU أثناء الـ boot أو الـ hotplug.

---

#### الـ `kp.symbol_name`

بدل ما نحسب عنوان الفانكشن يدويًا، بنديه اسمها كـ string وبيحوله لعنوان تلقائيًا من الـ kallsyms — أسهل وأكثر portability بين kernel versions.

---

#### الـ `module_init` / `module_exit`

- **`register_kprobe`** في الـ init بيزرع breakpoint في كود الـ kernel عند بداية الفانكشن المستهدفة.
- **`unregister_kprobe`** في الـ exit ضروري جدًا: لو ما ازلتش الـ breakpoint قبل ما الـ module يتـ unload، الـ kernel هيحاول ينفذ الـ handler في ذاكرة مش موجودة تاني وهيحصل kernel panic.

---

### Makefile لبناء الـ module

```makefile
obj-m += of_iommu_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وملاحظة الـ output

```bash
# بناء الـ module
make

# تحميله
sudo insmod of_iommu_probe.ko

# متابعة الـ log (أفضل وقت هو أثناء hotplug لـ device)
sudo dmesg -w | grep of_iommu

# إزالته
sudo rmmod of_iommu_probe
```

مثال على output متوقع أثناء إضافة device:

```
of_iommu_probe: hooked of_iommu_configure at ffffffffc0a12340
of_iommu_configure: dev=0000:01:00.0 node=/smmu@fd800000 id=0x1a
of_iommu_probe: kprobe removed
```

ده بيوضح إن الـ device `0000:01:00.0` (PCIe device) اتربط بالـ SMMU اللي موجود في عنوان `0xfd800000` في الـ Device Tree بـ stream ID قيمته `0x1a`.
