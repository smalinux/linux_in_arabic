## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ IOMMU؟

تخيل إنك بتشغّل برنامج على الكمبيوتر، والـ CPU بيستخدم الـ **virtual memory** عشان يحمي كل process من التانية — كل process شايفة عالم وهمي منفصل، ومش ممكن تقرأ memory تانية. الـ **MMU** (Memory Management Unit) هي اللي بتترجم العناوين الوهمية لعناوين physical حقيقية في الـ RAM.

طيب، ماذا عن الأجهزة؟

الـ **GPU**، الـ **NIC** (كارت الشبكة)، الـ **NVMe controller** — كل دول بيعملوا **DMA** (Direct Memory Access)، يعني بيكتبوا/بيقرأوا الـ RAM مباشرةً من غير ما يمروا على الـ CPU. المشكلة إن الجهاز بيشتغل بـ physical addresses، ومفيش حاجة تمنعه من إنه يكتب في أي حتة في الـ RAM!

لو عندك GPU في VM (virtual machine)، وسمحت للـ VM تتحكم فيه مباشرةً — ممكن الـ GPU يكتب في memory الـ host kernel ويعمل privilege escalation. ده خطر أمني فادح.

هنا بييجي دور الـ **IOMMU** (Input-Output Memory Management Unit).

---

### القصة: مشكلة الـ DMA

**السيناريو:** عندك سيرفر بيشغّل KVM hypervisor، وفيه VM بتاعة عميل. العميل عايز يستخدم الـ GPU بشكل مباشر (GPU passthrough). لو ديت الـ GPU للـ VM من غير حماية:

1. الـ VM تقدر تبرمج الـ GPU يكتب في أي physical address.
2. الـ GPU ممكن يكتب في kernel memory أو يقرأ passwords من memory الـ host.
3. الـ host اتاخد بالكامل.

**الحل:** نحط IOMMU بين الـ bus (PCIe) والـ RAM. الـ IOMMU بيعمل نفس فكرة الـ MMU — بيترجم **IOVA** (IO Virtual Address) اللي الجهاز بيستخدمها لعنوان physical حقيقي، بس بس العناوين المسموح بيها في الـ mapping.

```
┌──────────┐    IOVA     ┌─────────┐   Physical   ┌──────────┐
│  Device  │────────────▶│  IOMMU  │─────────────▶│   RAM    │
│ (GPU/NIC)│             │  (H/W)  │              │          │
└──────────┘             └─────────┘              └──────────┘
                              │
                         page tables
                         (في الـ RAM،
                         controlled by
                          kernel)
```

الـ kernel بيبني page tables للـ IOMMU، ويحدد: "الجهاز ده مسموحله يوصل لـ physical pages دي بس." أي access تاني — الـ IOMMU بيبلوكها ويطلق fault.

---

### ما هو الـ `include/linux/iommu.h`؟

الفايل ده هو **الـ contract الرئيسي** لكل الـ IOMMU subsystem في الـ Linux kernel. هو بيعرّف:

- كل الـ **data structures** اللي الـ subsystem بيشتغل بيها
- كل الـ **ops tables** (vtables) اللي الـ drivers بيـimplementوها
- كل الـ **public API** اللي باقي الـ kernel بيستخدمها

ببساطة: أي كود في الـ kernel عايز يتعامل مع الـ IOMMU (سواء driver، hypervisor code، أو DMA layer) — بيـinclude الفايل ده.

---

### الـ Subsystem

الفايل ده ينتمي لـ **IOMMU SUBSYSTEM** المذكور في MAINTAINERS:

| Maintainer | Role |
|---|---|
| Joerg Roedel `<joro@8bytes.org>` | Primary |
| Will Deacon `<will@kernel.org>` | Primary |
| Robin Murphy `<robin.murphy@arm.com>` | Reviewer |

الـ mailing list: `iommu@lists.linux.dev`

---

### أهم المفاهيم في الفايل

#### 1. الـ `iommu_domain` — قلب الـ subsystem

الـ **domain** هو "فضاء العناوين" اللي جهاز (أو مجموعة أجهزة) بيشتغل جوه. الـ domain بيحتوي على page tables ترجمة الـ IOVA → Physical.

أنواع الـ domains:

| النوع | المعنى |
|---|---|
| `IOMMU_DOMAIN_IDENTITY` | الجهاز بيشوف physical addresses مباشرةً (passthrough) |
| `IOMMU_DOMAIN_BLOCKED` | كل الـ DMA محظور، عزل كامل |
| `IOMMU_DOMAIN_DMA` | الـ kernel بيدير الـ mapping للـ DMA API |
| `IOMMU_DOMAIN_UNMANAGED` | الـ user (مثلاً VFIO/KVM) بيدير الـ mapping |
| `IOMMU_DOMAIN_SVA` | Shared Virtual Addressing — الجهاز بيشارك الـ process address space |
| `IOMMU_DOMAIN_NESTED` | Two-stage translation للـ virtualization |

#### 2. الـ `iommu_ops` — vtable الـ hardware driver

كل hardware IOMMU (Intel VT-d، AMD-Vi، ARM SMMU) بيملي struct `iommu_ops` بـ function pointers. الـ core subsystem بيكلم كل drivers من خلالها.

أهم operations:
- `domain_alloc_paging()` — خلق domain جديد
- `probe_device()` — لما جهاز ينضم للـ IOMMU
- `device_group()` — تحديد الـ isolation group

#### 3. الـ `iommu_domain_ops` — operations الـ domain نفسه

- `attach_dev()` — ربط جهاز بـ domain
- `map_pages()` / `unmap_pages()` — بناء/تدمير الـ page table entries
- `iova_to_phys()` — ترجمة عكسية للـ debug
- `iotlb_sync()` — flush الـ IOMMU TLB بعد التغييرات

#### 4. الـ `iommu_group` — وحدة الـ Isolation

الـ **IOMMU group** هي أصغر وحدة isolation ممكنة. لو جهازين في نفس الـ group — مش ممكن تعزلهم عن بعض (الـ hardware مش بيدعم ده).

مثال: لو عندك PCIe switch، الـ devices وراءه ممكن تكون في نفس الـ group لأنهم بيشاركوا نفس الـ PCIe traffic.

```
iommu_group_0
    ├── GPU (0000:01:00.0)
    └── GPU Audio (0000:01:00.1)   ← لازم يتعاملوا كوحدة واحدة

iommu_group_1
    └── NVMe (0000:02:00.0)        ← ممكن يتعزل لوحده
```

#### 5. الـ PASID والـ SVA

الـ **PASID** (Process Address Space ID) هو رقم بيتبعت مع كل PCIe transaction يميز الـ process. ده بيخلي جهاز واحد يشتغل لـ processes مختلفة في نفس الوقت، وكل process ليها domain منفصل.

الـ **SVA** (Shared Virtual Addressing) يستخدم الـ PASID عشان الجهاز يشوف نفس virtual address space بتاع الـ CPU process — الـ GPU مثلاً يقدر يستخدم نفس pointers الـ CPU من غير copies.

#### 6. الـ IO Page Fault (IOPF)

لو جهاز طلب عنوان مش في الـ mapping، الـ IOMMU بيعمل **page fault**. الـ kernel ممكن يـhandle الـ fault ده (يبني الـ mapping on-demand) بدل ما يـkill الـ transaction. الـ `struct iopf_queue` و `struct iopf_group` بيديروا الـ fault queue ده.

#### 7. الـ Dirty Tracking

الـ `iommu_dirty_ops` بتدعم **dirty page tracking** للـ live migration — الـ IOMMU بيراقب أنهي pages اتكتب فيها الجهاز أثناء الـ migration عشان الـ hypervisor يعرف يـcopy التغييرات للـ destination.

---

### الـ IOMMU بـ ELI5: التشبيه الكامل

تخيل مطار كبير (الـ RAM). الركاب (الـ devices) عايزين يوصلوا لـ terminals (memory regions). من غير حماية، أي راكب يقدر يدخل أي حتة.

الـ **IOMMU** هو الـ security checkpoint في المطار:
- كل جهاز عنده "تذكرة" (IOVA) بس بتخليه يدخل على بوابات معينة بس (physical pages).
- الـ **domain** هو قائمة البوابات المسموح بيها لكل مجموعة ركاب.
- الـ **group** هو رحلة — لو أشخاص في نفس الرحلة، لازم تتعامل معاهم مع بعض.
- الـ **PASID** هو badge شخصي يميز كل راكب جوه نفس الرحلة.
- الـ kernel هو اللي بيطبع ويـmanage التذاكر.

---

### الملفات المرتبطة

#### Core — الـ Subsystem نفسه

| الملف | الدور |
|---|---|
| `include/linux/iommu.h` | الـ header الرئيسي — types + API |
| `drivers/iommu/iommu.c` | الـ core implementation |
| `drivers/iommu/iommu-sva.c` | Shared Virtual Addressing |
| `drivers/iommu/io-pgfault.c` | IO Page Fault handling |
| `drivers/iommu/dma-iommu.c` | الـ DMA API layer فوق الـ IOMMU |
| `drivers/iommu/iova.c` | IOVA allocator |
| `include/linux/iova.h` | IOVA types |
| `include/linux/of_iommu.h` | Device Tree integration |

#### IOMMUFD — الـ userspace interface الحديث

| الملف | الدور |
|---|---|
| `drivers/iommu/iommufd/main.c` | Entry point |
| `drivers/iommu/iommufd/device.c` | Device management |
| `drivers/iommu/iommufd/io_pagetable.c` | Page table management |
| `include/uapi/linux/iommufd.h` | الـ userspace ABI |

#### Hardware Drivers

| الملف | الـ Hardware |
|---|---|
| `drivers/iommu/intel/iommu.c` | Intel VT-d |
| `drivers/iommu/amd/iommu.c` | AMD-Vi |
| `drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c` | ARM SMMU v3 |
| `drivers/iommu/apple-dart.c` | Apple M1 DART |
| `drivers/iommu/virtio-iommu.c` | Virtual IOMMU |
| `drivers/iommu/riscv/iommu.c` | RISC-V IOMMU |
| `drivers/iommu/tegra-smmu.c` | NVIDIA Tegra |

#### IO Page Tables

| الملف | الدور |
|---|---|
| `drivers/iommu/io-pgtable.c` | Generic page table framework |
| `drivers/iommu/io-pgtable-arm.c` | ARM LPAE page tables |
| `drivers/iommu/io-pgtable-arm-v7s.c` | ARM v7s short descriptor |

---

### ليه الفايل ده مهم جداً؟

الـ `iommu.h` هو **العقد** (contract) بين ثلاث طرفين:

1. **Hardware drivers** (Intel VT-d، ARM SMMU…) — بيـimplementوا الـ `iommu_ops`
2. **Kernel consumers** (DMA API، VFIO، iommufd) — بيستخدموا الـ API
3. **Userspace** (QEMU، containers) — عن طريق VFIO أو iommufd

بدون الـ IOMMU:
- الـ GPU passthrough للـ VMs مستحيل بأمان
- الـ DMA من devices ممكن يكسر الـ kernel security
- الـ containers مش ممكن تعزل device access
- الـ live migration مع GPU مستحيل
## Phase 2: شرح الـ IOMMU Framework

### المشكلة — ليه الـ IOMMU موجود أصلاً؟

لو عندك device زي GPU أو NIC بيعمل DMA (Direct Memory Access)، ده معناه إن الـ hardware بيقدر يكتب أو يقرأ في أي مكان في الـ physical memory مباشرةً — من غير ما يمر على الـ CPU.

ده بيخلق مشاكل تلاتة جوهرية:

1. **الأمان (Security):** device واحد ممكن يقرأ memory الـ device التاني أو حتى الـ kernel. في بيئات الـ virtualization، guest VM ممكن تعمل DMA على memory تبعت الـ host أو VM تانية.
2. **الـ Address Space Mismatch:** الـ device ممكن يكون 32-bit وما يقدرش يوصل للـ physical memory اللي فوق الـ 4GB. الـ kernel لازم يعمل bounce buffers — ده overhead حقيقي.
3. **الـ Scatter-Gather Complexity:** الـ kernel بيشتغل بـ virtual memory وصفحات متفرقة. لو محتاج تديها لـ device يتوقع buffer متصل، محتاج تعمل copy — overhead تاني.

الـ **IOMMU (Input/Output Memory Management Unit)** هو hardware chip (أو جزء من الـ SoC) بيجلس بين الـ device وبين الـ memory bus. بيعمل address translation لأي DMA transaction — بالظبط زي الـ MMU للـ CPU لكن للـ I/O.

---

### الحل — النهج اللي الـ kernel بياخده

الـ kernel بيعمل لكل device (أو مجموعة devices) **IOMMU domain** — ده عبارة عن address space معزول ليه page tables خاصة بيه. الـ device مش بيشوف الـ physical addresses — بيشوف **IOVA (I/O Virtual Addresses)** بس.

```
Device يعمل DMA على IOVA 0x1000
    ↓
IOMMU يعمل lookup في page table الـ domain
    ↓
يلاقي إن IOVA 0x1000 mapped على Physical 0xDEAD0000
    ↓
الـ memory transaction بيروح على 0xDEAD0000 بالفعل
```

لو الـ IOVA مش mapped في domain الـ device — الـ IOMMU بيبلوك التransaction ويرفع fault. Device مش قادر يوصل لأي حاجة خارج domain بتاعه.

---

### الـ Real-World Analogy — مبنى فيه موظفين

تخيل مبنى شركة كبير فيه موظفين من شركات مختلفة (tenants). عندهم:

- **الـ receptionist (IOMMU):** كل واحد عايز يدخل مكتب تاني لازم يمر عليه.
- **الـ badge (IOVA):** كل موظف عنده badge برقم داخلي مش بيعرفوش العنوان الحقيقي للمكتب.
- **الـ directory (page tables):** الـ receptionist عنده directory بيحول badge numbers لأرقام مكاتب حقيقية.
- **الـ tenant isolation (iommu domain):** موظف من شركة X ممكن يدخل بس المكاتب المسموحة ليه — حتى لو عرف رقم مكتب شركة Y مش هيوصله.
- **الـ security alert (IOMMU fault):** لو حاول يدخل مكتب مش في directory بتاعه — إنذار فوري.

**Mapping كامل:**

| العالم الحقيقي | IOMMU Framework |
|---|---|
| الموظف | الـ DMA device |
| الـ Badge Number | IOVA |
| العنوان الحقيقي للمكتب | Physical Address |
| الـ receptionist | IOMMU hardware |
| الـ directory | IO Page Table |
| قسم الموظف (tenant) | iommu_domain |
| مجموعة موظفين بنفس صلاحيات الأمان | iommu_group |
| شركة التأمين اللي بتحدد السياسة | IOMMU driver (مثلاً ARM SMMU) |

---

### الـ Big Picture Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Linux Kernel                                 │
│                                                                      │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │  DMA API    │    │  VFIO/KVM    │    │   iommufd (userspace) │  │
│  │(dma_map_*)  │    │(VM passthru) │    │   /dev/iommu          │  │
│  └──────┬──────┘    └──────┬───────┘    └──────────┬───────────┘   │
│         │                  │                        │               │
│         └──────────────────┴────────────────────────┘               │
│                            │                                         │
│                  ┌─────────▼──────────┐                             │
│                  │   IOMMU Core       │  ← iommu.h هنا             │
│                  │  (iommu_map,       │                             │
│                  │   iommu_domain,    │                             │
│                  │   iommu_group)     │                             │
│                  └─────────┬──────────┘                             │
│                            │                                         │
│          ┌─────────────────┼──────────────────┐                    │
│          │                 │                  │                     │
│  ┌───────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐            │
│  │  ARM SMMU    │  │  Intel VT-d  │  │  AMD IOMMU   │            │
│  │  driver      │  │  driver      │  │  driver      │            │
│  └───────┬──────┘  └───────┬──────┘  └───────┬──────┘            │
└──────────┼─────────────────┼─────────────────┼────────────────────┘
           │                 │                  │
    ┌──────▼──────────────────▼──────────────────▼──────┐
    │              IOMMU Hardware                        │
    │  (SMMU / VT-d / AMD-Vi sits on memory bus)        │
    └───────────────────────────────┬───────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │         System Memory          │
                    │      (Physical RAM)            │
                    └───────────────────────────────┘

  Devices:  [NIC] [GPU] [USB] [PCIe EP] ──→ IOMMU Hardware
```

---

### الـ Core Abstractions — المفاهيم الأساسية

#### 1. `struct iommu_domain` — قلب الـ Framework

الـ **iommu_domain** هو الـ address space العازل. كل domain عنده page table خاصة بيه. الـ device لما بيتعمله attach لـ domain، كل DMA بتاعه بيترجم عبر الـ page table دي.

```c
struct iommu_domain {
    unsigned type;                    /* نوع الـ domain */
    const struct iommu_domain_ops *ops; /* operations خاصة بالـ domain */
    const struct iommu_ops *owner;    /* الـ driver اللي عمله */
    unsigned long pgsize_bitmap;      /* أحجام الصفحات المدعومة */
    struct iommu_domain_geometry geometry; /* نطاق الـ IOVA المسموح بيه */
    int (*iopf_handler)(struct iopf_group *group); /* fault handler */

    union { /* cookie: بيتغير حسب نوع الـ domain */
        struct iommu_dma_cookie *iova_cookie;   /* لـ DMA API */
        struct iommu_dma_msi_cookie *msi_cookie; /* لـ MSI */
        struct iommufd_hw_pagetable *iommufd_hwpt; /* لـ userspace */
        struct { iommu_fault_handler_t handler; void *handler_token; };
        struct { struct mm_struct *mm; int users; struct list_head next; }; /* SVA */
    };
};
```

**أنواع الـ domain:**

| النوع | القيمة | المعنى |
|---|---|---|
| `IOMMU_DOMAIN_BLOCKED` | `0` | كل DMA محجوب — عزل تام |
| `IOMMU_DOMAIN_IDENTITY` | `PT` flag | IOVA = Physical Address (passthrough) |
| `IOMMU_DOMAIN_UNMANAGED` | `PAGING` flag | الـ driver بيدير الـ mapping يدوياً |
| `IOMMU_DOMAIN_DMA` | `PAGING\|DMA_API` | الـ DMA API بيدير الـ mapping تلقائي |
| `IOMMU_DOMAIN_SVA` | `SVA` flag | Shared Virtual Addressing مع process |
| `IOMMU_DOMAIN_NESTED` | `NESTED` flag | two-stage translation لـ VMs |

---

#### 2. `struct iommu_group` — وحدة الـ Isolation

الـ **iommu_group** هو مجموعة devices بتشارك نفس IOMMU context ولازم تتعزل مع بعض. مش كل device عنده domain منفصل — كل الـ devices في نفس الـ group بيشاركوا نفس الـ domain.

ليه? بسبب الـ hardware. مثلاً: PCIe ACS (Access Control Services) ممكن ميكونش مدعوم على سويتش معين، فالـ devices خلفيه بتقدر تعمل peer-to-peer DMA من غير ما تعدي على الـ IOMMU. في الحالة دي، لازم يكونوا في نفس الـ group.

```
┌─────── iommu_group ───────┐
│  ┌────────┐  ┌────────┐   │
│  │ GPU fn0│  │ GPU fn1│   │
│  └────────┘  └────────┘   │
│       ↕ peer DMA ↕        │
│  (can bypass IOMMU)       │
│    shared iommu_domain    │
└───────────────────────────┘
```

---

#### 3. `struct iommu_ops` — الـ Driver Interface

الـ **iommu_ops** هو الـ vtable اللي الـ IOMMU hardware driver بيملاه. الـ core framework بيتكلم مع الـ hardware من خلاله بس.

```c
struct iommu_ops {
    /* الـ driver بيعلن capabilities */
    bool (*capable)(struct device *dev, enum iommu_cap);

    /* إزاي تعمل domain جديد */
    struct iommu_domain *(*domain_alloc_paging)(struct device *dev);
    struct iommu_domain *(*domain_alloc_sva)(struct device *dev, struct mm_struct *mm);
    struct iommu_domain *(*domain_alloc_nested)(struct device *dev,
                          struct iommu_domain *parent, u32 flags, ...);

    /* إدارة الـ devices */
    struct iommu_device *(*probe_device)(struct device *dev);
    void (*release_device)(struct device *dev);
    struct iommu_group *(*device_group)(struct device *dev);

    /* default domains */
    struct iommu_domain *identity_domain; /* passthrough جاهز */
    struct iommu_domain *blocked_domain;  /* block-all جاهز */

    /* virtual IOMMU support (for VMs) */
    int (*viommu_init)(struct iommufd_viommu *viommu, ...);
};
```

---

#### 4. `struct iommu_domain_ops` — عمليات الـ Page Table

الـ **iommu_domain_ops** بيحتوي على العمليات الخاصة بكل domain — mapping وunmapping والـ TLB.

```c
struct iommu_domain_ops {
    /* attach/detach device من domain */
    int (*attach_dev)(struct iommu_domain *domain, struct device *dev,
                      struct iommu_domain *old);

    /* map/unmap pages */
    int (*map_pages)(struct iommu_domain *domain, unsigned long iova,
                     phys_addr_t paddr, size_t pgsize, size_t pgcount,
                     int prot, gfp_t gfp, size_t *mapped);
    size_t (*unmap_pages)(struct iommu_domain *domain, unsigned long iova,
                          size_t pgsize, size_t pgcount,
                          struct iommu_iotlb_gather *iotlb_gather);

    /* TLB management */
    void (*flush_iotlb_all)(struct iommu_domain *domain);
    void (*iotlb_sync)(struct iommu_domain *domain,
                       struct iommu_iotlb_gather *iotlb_gather);

    /* translation */
    phys_addr_t (*iova_to_phys)(struct iommu_domain *domain, dma_addr_t iova);

    void (*free)(struct iommu_domain *domain);
};
```

---

#### 5. `struct iommu_device` — تسجيل الـ Hardware

الـ **iommu_device** بيمثل IOMMU hardware instance واحد (مثلاً: SMMU controller واحد). الـ driver بيسجله بـ `iommu_device_register()`.

```c
struct iommu_device {
    struct list_head list;        /* في list الـ registered IOMMUs */
    const struct iommu_ops *ops;  /* driver ops */
    struct fwnode_handle *fwnode; /* firmware node (DT/ACPI) */
    struct device *dev;           /* sysfs device */
    struct iommu_group *singleton_group; /* optimization لـ single-device drivers */
    u32 max_pasids;               /* عدد الـ PASIDs المدعومة */
};
```

---

#### 6. `struct dev_iommu` — الـ Per-Device Data

كل `struct device` بيحتوي على pointer لـ `struct dev_iommu` — ده اللي بيربط الـ device بالـ IOMMU subsystem.

```c
struct dev_iommu {
    struct mutex lock;
    struct iommu_fault_param __rcu *fault_param; /* fault handling data */
    struct iommu_fwspec        *fwspec;    /* firmware IDs (stream IDs) */
    struct iommu_device        *iommu_dev; /* الـ IOMMU controller */
    void                       *priv;      /* driver private data */
    u32 max_pasids;
    u32 attach_deferred:1;        /* attach لسه ما تمش */
    u32 pci_32bit_workaround:1;   /* 32-bit DMA limitation */
    u32 require_direct:1;         /* يحتاج 1:1 mapping لمناطق معينة */
    u32 shadow_on_flush:1;        /* shadow page tables */
};
```

---

### العلاقة بين الـ Structs

```
struct device
    └── dev->iommu  ──→  struct dev_iommu
                              ├── iommu_dev  ──→  struct iommu_device
                              │                        └── ops  ──→  struct iommu_ops
                              └── fwspec     ──→  struct iommu_fwspec (stream IDs)

struct iommu_group  ──────────────────────────────────────────────────
    ├── [device 1] ──→  struct device
    ├── [device 2] ──→  struct device
    └── domain      ──→  struct iommu_domain
                              ├── ops    ──→  struct iommu_domain_ops
                              │               (map_pages, unmap_pages, iotlb_sync...)
                              ├── dirty_ops ──→  struct iommu_dirty_ops
                              │                   (dirty page tracking for live migration)
                              └── cookie  ──→  iova_cookie / msi_cookie / iommufd_hwpt
```

---

### الـ TLB Management — الـ iotlb_gather

الـ IOMMU عنده TLB (Translation Lookaside Buffer) بيخزن الترجمات — زي الـ CPU TLB بالظبط. لما بتعمل `iommu_unmap`، الـ TLB القديم لازم يتشال قبل ما الـ memory تتحرر — غير كده device ممكن يوصل لـ memory اتأعيد استخدامها.

الـ **iommu_iotlb_gather** بيجمع (gather) عمليات الـ unmap مع بعض وبيعمل flush واحد كبير بدل flush لكل page لوحدها — ده optimization مهم.

```c
struct iommu_iotlb_gather {
    unsigned long start;  /* بداية الـ IOVA range المحتاج flush */
    unsigned long end;    /* نهاية الـ IOVA range */
    size_t pgsize;        /* granularity الصفحات */
    struct iommu_pages_list freelist; /* صفحات تتحرر بعد الـ flush */
    bool queued;          /* flush هيتعمل batch لاحقاً */
};
```

الـ workflow:

```
iommu_unmap(page1)  → iommu_iotlb_gather_add_page(gather, ...)
iommu_unmap(page2)  → iommu_iotlb_gather_add_page(gather, ...)
iommu_unmap(page3)  → iommu_iotlb_gather_add_page(gather, ...)
                    ↓
              iommu_iotlb_sync(domain, gather)
                    ↓
              flush TLB مرة واحدة للـ range كله
                    ↓
              حرر الصفحات من الـ freelist
```

---

### الـ Fault Handling — الـ IOPF

لما device يحاول يوصل لـ IOVA مش mapped، الـ IOMMU بيرفع **IO Page Fault**. في الـ SVA وبيئات الـ virtualization، ده مش بالضرورة error — ممكن يكون demand paging.

```
IOMMU Hardware
    ↓ fault interrupt
struct iopf_fault
    ↓ يتحط في
struct iopf_group (مجموعة فولتات من نفس الـ PASID)
    ↓ يتبعت لـ
struct iopf_queue (workqueue)
    ↓
domain->iopf_handler(group)
    ↓
iommu_page_response (success/failure/invalid)
    ↓
iommu_ops->page_response()
    ↓ ترجع للـ IOMMU Hardware
```

---

### الـ PASID — Process Address Space ID

الـ **PASID** (اللي بيتسمى كمان SSID في ARM SMMU) بيخلي device واحد يستخدم page tables متعددة في نفس الوقت. كل transaction من الـ device بيحمل PASID، والـ IOMMU بيستخدمه عشان يختار الـ domain الصح.

ده بيفتح باب لـ **SVA (Shared Virtual Addressing)**: device يستخدم نفس الـ page table بتاعت process معينة — IOVA = virtual address بتاع الـ process. الـ device بيشوف نفس الـ address space اللي الـ CPU بيشوفه.

```
Process A (pid=100):        Process B (pid=200):
  PASID = 5                   PASID = 7
  page table = mm_A           page table = mm_B

Device DMA transaction:
  PASID=5, IOVA=0x1000  →  IOMMU يستخدم mm_A page table
  PASID=7, IOVA=0x1000  →  IOMMU يستخدم mm_B page table
```

---

### الـ Reserved Regions

بعض المناطق في الـ physical memory لازم تتعامل معاها بطريقة خاصة — ما تنقلش.

```c
enum iommu_resv_type {
    IOMMU_RESV_DIRECT,          /* لازم mapped 1:1 دايماً */
    IOMMU_RESV_DIRECT_RELAXABLE, /* 1:1 لكن ممكن يتغير في device assignment */
    IOMMU_RESV_RESERVED,        /* ممنوع تماماً */
    IOMMU_RESV_MSI,             /* MSI doorbell addresses — hardware untranslated */
    IOMMU_RESV_SW_MSI,          /* MSI window managed by software */
};
```

مثال حقيقي: MSI (Message Signaled Interrupts) في PCIe. الـ device بيبعت interrupt عن طريق DMA write لعنوان معين. الـ IOMMU محتاج يعرف إن العنوان ده مش يترجم — ولا الـ interrupt مش هيوصل.

---

### الـ Nested Translation — لـ Virtualization

الـ **IOMMU_DOMAIN_NESTED** بيدعم two-stage translation — مهم جداً لـ VFIO وـ iommufd:

```
Guest IOVA (Stage 1 - managed by guest)
    ↓
Guest Physical Address (= Host IOVA)
    ↓  (Stage 2 - managed by host)
Host Physical Address (Real RAM)
```

الـ host بيتحكم في Stage 2 وده بيضمن إن الـ guest مهما عمل في Stage 1 مش هيقدر يتخطى عزل الـ host.

---

### الـ Framework بيتملك إيه؟ وبيفوض إيه للـ Drivers؟

**الـ IOMMU Core بيتملك:**
- منطق الـ grouping (إيه devices يدخلوا نفس الـ group)
- الـ DMA API integration (الـ `dma_map_*` calls بتروح هنا)
- الـ fault queue management
- الـ PASID allocation وlifecycle
- الـ sysfs representation
- الـ domain lifecycle (alloc/free) من حيث الـ policy

**بيفوض للـ Drivers (عبر `iommu_ops` و`iommu_domain_ops`):**
- كيفية بناء الـ page table (كل hardware ليه format مختلف)
- TLB flush commands (ARM SMMU بيستخدم commands مختلفة عن Intel VT-d)
- الـ hardware-specific capabilities وقدراتها
- تفسير الـ firmware data (stream IDs في DT، ACPI _IRT)
- MSI remapping في hardware level
- virtual IOMMU implementation للـ VMs
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### أولاً — Permission Flags للـ Mapping (بيتحدد لما تعمل `iommu_map`)

| Flag | Value | المعنى |
|---|---|---|
| `IOMMU_READ` | `1 << 0` | الـ device يقدر يقرأ من الـ IOVA |
| `IOMMU_WRITE` | `1 << 1` | الـ device يقدر يكتب على الـ IOVA |
| `IOMMU_CACHE` | `1 << 2` | DMA cache coherency مطلوبة |
| `IOMMU_NOEXEC` | `1 << 3` | المنطقة مش executable |
| `IOMMU_MMIO` | `1 << 4` | منطقة MMIO (زي MSI doorbells) |
| `IOMMU_PRIV` | `1 << 5` | الصلاحيات دي بس للـ supervisor mode |

#### ثانياً — Fault Permission Flags (بيجي في الـ page fault)

| Flag | Value | المعنى |
|---|---|---|
| `IOMMU_FAULT_PERM_READ` | `1 << 0` | الـ fault ناتج عن read |
| `IOMMU_FAULT_PERM_WRITE` | `1 << 1` | الـ fault ناتج عن write |
| `IOMMU_FAULT_PERM_EXEC` | `1 << 2` | الـ fault ناتج عن execute |
| `IOMMU_FAULT_PERM_PRIV` | `1 << 3` | الـ fault من privileged mode |

#### ثالثاً — Domain Internal Feature Flags (`__IOMMU_DOMAIN_*`)

| Flag | Bit | الاستخدام |
|---|---|---|
| `__IOMMU_DOMAIN_PAGING` | 0 | يدعم `iommu_map`/`iommu_unmap` |
| `__IOMMU_DOMAIN_DMA_API` | 1 | مخصص لتنفيذ DMA-API |
| `__IOMMU_DOMAIN_PT` | 2 | identity mapping (passthrough) |
| `__IOMMU_DOMAIN_DMA_FQ` | 3 | يستخدم flush queue |
| `__IOMMU_DOMAIN_SVA` | 4 | Shared Virtual Addressing |
| `__IOMMU_DOMAIN_PLATFORM` | 5 | legacy platform domain |
| `__IOMMU_DOMAIN_NESTED` | 6 | nested translation (stage-2) |

#### رابعاً — Domain Types العامة (cheatsheet)

| Type | تركيبة الـ Flags | الاستخدام |
|---|---|---|
| `IOMMU_DOMAIN_BLOCKED` | `0` | يبلوك كل DMA — عزل تام |
| `IOMMU_DOMAIN_IDENTITY` | `PT` | عناوين DMA = عناوين فيزيائية |
| `IOMMU_DOMAIN_UNMANAGED` | `PAGING` | للـ VMs — المستخدم يدير الـ mapping |
| `IOMMU_DOMAIN_DMA` | `PAGING\|DMA_API` | الاستخدام العادي (kernel DMA API) |
| `IOMMU_DOMAIN_DMA_FQ` | `PAGING\|DMA_API\|DMA_FQ` | زي DMA بس مع flush queue |
| `IOMMU_DOMAIN_SVA` | `SVA` | process address space مشترك |
| `IOMMU_DOMAIN_NESTED` | `NESTED` | nested (iommufd/vIOMMU) |

#### خامساً — enum iommu_cap (قدرات الـ IOMMU)

| القيمة | المعنى |
|---|---|
| `IOMMU_CAP_CACHE_COHERENCY` | الـ hardware يدعم `IOMMU_CACHE` |
| `IOMMU_CAP_NOEXEC` | يدعم `IOMMU_NOEXEC` |
| `IOMMU_CAP_PRE_BOOT_PROTECTION` | الـ firmware فعّل IOMMU قبل الـ boot |
| `IOMMU_CAP_ENFORCE_CACHE_COHERENCY` | `enforce_cache_coherency()` شغالة على الجهاز |
| `IOMMU_CAP_DEFERRED_FLUSH` | الـ driver مش بيعمل TLB flush في `unmap` مباشرةً |
| `IOMMU_CAP_DIRTY_TRACKING` | يدعم تتبع الـ dirty pages |

#### سادساً — enum iommu_resv_type (أنواع المناطق المحجوزة)

| القيمة | المعنى |
|---|---|
| `IOMMU_RESV_DIRECT` | لازم تبقى mapped 1:1 دايماً |
| `IOMMU_RESV_DIRECT_RELAXABLE` | 1:1 بس ممكن يتغير في device assignment |
| `IOMMU_RESV_RESERVED` | ممنوع تديها لأي device |
| `IOMMU_RESV_MSI` | منطقة MSI untranslated |
| `IOMMU_RESV_SW_MSI` | MSI translation window مدار بالـ software |

#### سابعاً — PASID Constants

| الثابت | القيمة | المعنى |
|---|---|---|
| `IOMMU_NO_PASID` | `0` | DMA بدون PASID (الـ RID فقط) |
| `IOMMU_FIRST_GLOBAL_PASID` | `1` | أول PASID قابل للتخصيص |
| `IOMMU_PASID_INVALID` | `-1U` | قيمة خطأ / غير صالحة |

#### ثامناً — iommu_page_response_code

| القيمة | المعنى |
|---|---|
| `IOMMU_PAGE_RESP_SUCCESS` | تمت معالجة الـ fault، أعد المحاولة |
| `IOMMU_PAGE_RESP_INVALID` | طلب غير صالح، لا تعيد |
| `IOMMU_PAGE_RESP_FAILURE` | خطأ عام، أوقف الـ faults من الجهاز |

#### تاسعاً — Config Options المهمة

| Option | الأثر |
|---|---|
| `CONFIG_IOMMU_API` | يفعّل كل الـ IOMMU subsystem |
| `CONFIG_IOMMU_DMA` | يفعّل DMA-API فوق الـ IOMMU |
| `CONFIG_IOMMU_IOPF` | يفعّل IO Page Fault queue |
| `CONFIG_IOMMU_MM_DATA` | يفعّل SVA + PASID للـ processes |
| `CONFIG_IOMMU_DEBUGFS` | يضيف `/sys/kernel/debug/iommu/` |
| `CONFIG_IRQ_MSI_IOMMU` | دعم MSI remapping عبر IOMMU |
| `CONFIG_FSL_PAMU` | legacy path للـ Freescale PAMU |

---

### الـ Structs المهمة

#### 1. `struct iommu_domain`

**الغرض:** الكيان الأساسي في الـ IOMMU subsystem — يمثل جدول ترجمة IOVA→PA واحد. كل device مربوط بـ domain، وكل عملية map/unmap تتم عليه.

| الحقل | النوع | المعنى |
|---|---|---|
| `type` | `unsigned` | نوع الـ domain (BLOCKED/IDENTITY/DMA/…) |
| `cookie_type` | `enum iommu_domain_cookie_type` | يحدد أي فرع من الـ union نشيل |
| `ops` | `*iommu_domain_ops` | pointer للعمليات الخاصة بهذا الـ domain |
| `dirty_ops` | `*iommu_dirty_ops` | عمليات dirty tracking (live migration) |
| `owner` | `*iommu_ops` | الـ IOMMU driver اللي أنشأ الـ domain |
| `pgsize_bitmap` | `unsigned long` | الـ page sizes المدعومة |
| `geometry` | `iommu_domain_geometry` | نطاق الـ IOVA المسموح به |
| `iopf_handler` | function pointer | معالج الـ IO page faults |
| `iova_cookie` | union | للـ DMA-API: IOVA allocator state |
| `msi_cookie` | union | للـ MSI: بيانات MSI mapping |
| `iommufd_hwpt` | union | للـ iommufd: hardware page table |
| `handler`/`handler_token` | union | legacy fault handler |
| `mm`/`users`/`next` | union | للـ SVA: الـ process mm_struct |

**الارتباطات:** مرتبط بـ `iommu_ops` (الـ owner)، وبـ `iommu_domain_ops` (العمليات)، وبـ `iommu_dirty_ops` (الـ dirty tracking).

---

#### 2. `struct iommu_ops`

**الغرض:** الـ vtable الرئيسي للـ IOMMU driver — يُسجَّل مرة واحدة لكل IOMMU hardware وبيعرّف كل العمليات اللي الـ core يقدر يطلبها.

| الحقل | المعنى |
|---|---|
| `capable` | هل الـ device يدعم capability معينة؟ |
| `hw_info` | بيرجع معلومات الـ hardware (نوع، طول) |
| `domain_alloc_paging_flags` | ينشئ domain يدعم paging مع flags |
| `domain_alloc_paging` | ينشئ domain عادي (الأبسط) |
| `domain_alloc_sva` | ينشئ domain للـ SVA |
| `domain_alloc_nested` | ينشئ nested domain |
| `probe_device` | يضيف device للـ IOMMU driver، يرجع `iommu_device*` |
| `release_device` | يزيل device من الـ IOMMU driver |
| `probe_finalize` | إعداد نهائي بعد attach الـ device لـ group وdomain |
| `device_group` | يحدد الـ iommu_group للـ device |
| `get_resv_regions` | يجيب المناطق المحجوزة للـ device |
| `of_xlate` | يضيف OF stream IDs لـ grouping |
| `page_response` | يرسل الرد على page fault request |
| `def_domain_type` | يحدد الـ domain type الافتراضي للـ device |
| `get_viommu_size` | حجم الـ vIOMMU struct للـ driver |
| `viommu_init` | ينشئ vIOMMU للـ virtualization |
| `identity_domain` | domain جاهز للـ passthrough دايماً |
| `blocked_domain` | domain جاهز للـ block دايماً |
| `default_domain` | الـ domain الافتراضي (legacy) |
| `user_pasid_table` | الـ driver يدعم PASID table يديره الـ user |

---

#### 3. `struct iommu_domain_ops`

**الغرض:** عمليات خاصة بكل domain instance — الـ `iommu_ops` بتعمل allocate وتبدأ الـ domain، وهنا الـ ops اللي بتشتغل عليه.

| الحقل | المعنى |
|---|---|
| `attach_dev` | يربط device بالـ domain |
| `set_dev_pasid` | يربط PASID معين من device بالـ domain |
| `map_pages` | يضيف mapping في الـ page table |
| `unmap_pages` | يزيل mapping |
| `flush_iotlb_all` | يمسح كل الـ IOMMU TLB |
| `iotlb_sync_map` | sync بعد map |
| `iotlb_sync` | sync الـ flush queue |
| `cache_invalidate_user` | flush من user space (nested domains) |
| `iova_to_phys` | ترجمة IOVA لعنوان فيزيائي |
| `enforce_cache_coherency` | يمنع no-snoop TLPs |
| `set_pgtable_quirks` | يضبط quirks في الـ page table |
| `free` | يحرر الـ domain |

---

#### 4. `struct iommu_device`

**الغرض:** يمثل IOMMU hardware instance واحد (مثلاً: AMD-Vi على PCIe bus أو ARM SMMU على AMBA bus). بيتسجل في الـ iommu core list.

| الحقل | المعنى |
|---|---|
| `list` | ربطه في القائمة العامة للـ IOMMUs المسجلة |
| `ops` | الـ vtable بتاعه |
| `fwnode` | رابط الـ firmware node (DT أو ACPI) |
| `dev` | الـ `struct device` بتاعه (لـ sysfs) |
| `singleton_group` | الـ group الوحيد (لو الـ driver بسيط) |
| `max_pasids` | أقصى عدد PASIDs |
| `ready` | بيتضبط `true` بعد `iommu_device_register()` |

---

#### 5. `struct dev_iommu`

**الغرض:** بيانات IOMMU خاصة بكل device منفردة — بيتخزن في `device->iommu`. ده الـ glue بين الـ device الـ generic والـ IOMMU subsystem.

| الحقل | المعنى |
|---|---|
| `lock` | mutex يحمي الـ struct كلها |
| `fault_param` | pointer (RCU-protected) لبيانات الـ fault |
| `fwspec` | بيانات الـ firmware (stream IDs) |
| `iommu_dev` | الـ `iommu_device` المرتبط |
| `priv` | driver private data |
| `max_pasids` | أقصى PASIDs لهذا الـ device |
| `attach_deferred:1` | تأجيل الـ DMA domain attach |
| `pci_32bit_workaround:1` | حصر الـ IOVA في 32-bit |
| `require_direct:1` | يلزم `IOMMU_RESV_DIRECT` |
| `shadow_on_flush:1` | الـ TLB flush يتزامن مع shadow tables |

---

#### 6. `struct iommu_fault_param`

**الغرض:** بيانات الـ IO Page Fault لكل device — يربط الـ device بالـ fault queue ويتتبع الـ pending faults.

| الحقل | المعنى |
|---|---|
| `lock` | mutex يحمي قائمة الـ faults |
| `users` | refcount للـ lifetime management |
| `rcu` | للـ kfree_rcu safe freeing |
| `dev` | الـ device المالك |
| `queue` | الـ iopf_queue المرتبط |
| `queue_list` | node في `queue->devices` |
| `partial` | faults جزئية لم يكتمل الـ group بعد |
| `faults` | الـ pending faults اللي تستنى رد |

---

#### 7. `struct iopf_queue`

**الغرض:** الـ workqueue المخصص لمعالجة الـ IO Page Faults بشكل async.

| الحقل | المعنى |
|---|---|
| `wq` | الـ workqueue نفسه |
| `devices` | قائمة الـ devices المرتبطة |
| `lock` | mutex يحمي القائمة |

---

#### 8. `struct iopf_group`

**الغرض:** مجموعة page faults منتمية لنفس الـ Page Request Group (PRI group) — بيتم معالجتهم كوحدة واحدة.

| الحقل | المعنى |
|---|---|
| `last_fault` | آخر fault في الـ group (نسخة كاملة) |
| `faults` | قائمة كل الـ faults في الـ group |
| `fault_count` | عدد الـ faults |
| `pending_node` | node في `iommu_fault_param::faults` |
| `work` | الـ work_struct للمعالجة async |
| `attach_handle` | الـ domain المرتبط بهذا الـ fault |
| `fault_param` | بيانات fault الـ device |
| `node` | يخلي الـ handler يضمه في قوائمه |
| `cookie` | معرف فريد للـ group |

---

#### 9. `struct iopf_fault`

**الغرض:** wrapper بسيط حول `iommu_fault` بيضيفه في قائمة الـ pending faults.

| الحقل | المعنى |
|---|---|
| `fault` | بيانات الـ fault الفعلية |
| `list` | node في قائمة الـ faults |

---

#### 10. `struct iommu_fault` و `struct iommu_fault_page_request`

**الغرض:** تمثيل عام لأي IO Page Fault.

`iommu_fault`:
- `type` — نوع الـ fault (حالياً `IOMMU_FAULT_PAGE_REQ` فقط)
- `prm` — بيانات الـ page request

`iommu_fault_page_request`:
| الحقل | المعنى |
|---|---|
| `flags` | صلاحية الحقول + هل آخر page في الـ group |
| `pasid` | Process Address Space ID |
| `grpid` | Page Request Group Index |
| `perm` | الصلاحية المطلوبة (READ/WRITE/EXEC/PRIV) |
| `addr` | عنوان الصفحة الـ faulting |
| `private_data[2]` | بيانات خاصة بالـ device |

---

#### 11. `struct iommu_iotlb_gather`

**الغرض:** يجمع نطاقات الـ IOVA المحتاجة TLB flush بعد `unmap` متعددة، بدل ما يعمل flush لكل عملية منفردة.

| الحقل | المعنى |
|---|---|
| `start` | بداية النطاق المجمّع |
| `end` | نهاية النطاق (inclusive) |
| `pgsize` | حجم الصفحة (للـ page-based flush) |
| `freelist` | صفحات تُحرَّر بعد الـ sync |
| `queued` | الـ flush هيتأجل للـ `flush_all` |

---

#### 12. `struct iommu_dirty_bitmap`

**الغرض:** يربط الـ IOVA bitmap بالـ TLB gather أثناء dirty tracking (لـ live migration).

| الحقل | المعنى |
|---|---|
| `bitmap` | الـ IOVA bitmap للـ dirty pages |
| `gather` | TLB flush range مرتبط |

---

#### 13. `struct iommu_dirty_ops`

**الغرض:** عمليات dirty tracking الخاصة بـ domain.

| الحقل | المعنى |
|---|---|
| `set_dirty_tracking` | يفعّل/يوقف تتبع الـ dirty pages |
| `read_and_clear_dirty` | يقرأ الـ dirty PTEs ويمسحها |

---

#### 14. `struct iommu_resv_region`

**الغرض:** وصف منطقة IOVA محجوزة — الـ driver يُبلّغ عنها عبر `get_resv_regions` والـ core يتجنبها في الـ allocation.

| الحقل | المعنى |
|---|---|
| `list` | ربط في قائمة المناطق |
| `start` | العنوان الفيزيائي للبداية |
| `length` | الطول بالـ bytes |
| `prot` | الصلاحيات |
| `type` | نوع المنطقة (RESV_DIRECT/MSI/…) |
| `free` | callback لتحرير الذاكرة |

---

#### 15. `struct iommu_fwspec`

**الغرض:** بيانات الـ firmware لكل device — يحتوي على الـ stream IDs (أو requester IDs في PCIe) اللي بيستخدمها الـ device عند التعامل مع الـ IOMMU.

| الحقل | المعنى |
|---|---|
| `iommu_fwnode` | الـ firmware node للـ IOMMU المرتبط |
| `flags` | `IOMMU_FWSPEC_PCI_RC_ATS` أو `CANWBS` |
| `num_ids` | عدد الـ IDs |
| `ids[]` | flexible array من الـ stream/requester IDs |

---

#### 16. `struct iommu_attach_handle`

**الغرض:** يمثل العلاقة بين domain وPASID (أو RID) معين لجهاز معين — بيُخزَّن في الـ iommu_group أثناء الـ attach.

| الحقل | المعنى |
|---|---|
| `domain` | الـ domain المرتبط |

---

#### 17. `struct iommu_sva`

**الغرض:** handle لعلاقة SVA بين device وprocess — يرث من `iommu_attach_handle`.

| الحقل | المعنى |
|---|---|
| `handle` | الـ attach_handle الأساسي |
| `dev` | الـ device المرتبط |
| `users` | refcount |

---

#### 18. `struct iommu_mm_data`

**الغرض:** IOMMU-specific data للـ `mm_struct` — يحتفظ بالـ PASID المخصص للـ process وقائمة الـ SVA domains.

| الحقل | المعنى |
|---|---|
| `pasid` | الـ PASID المخصص للـ process |
| `mm` | pointer للـ mm_struct |
| `sva_domains` | قائمة الـ SVA domains المرتبطة بالـ mm |
| `mm_list_elm` | node في القائمة العامة للـ mm |

---

#### 19. `struct iommu_domain_geometry`

| الحقل | المعنى |
|---|---|
| `aperture_start` | أول IOVA يمكن mapping فيه |
| `aperture_end` | آخر IOVA يمكن mapping فيه |
| `force_aperture` | DMA مسموح فقط في النطاق ده |

---

#### 20. `struct iommu_user_data` و `struct iommu_user_data_array`

**الغرض:** بيانات user space تنتقل من `iommufd` للـ driver — بتحتوي pointer وnوع وطول.

---

### رسوم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────┐
│                    struct device                     │
│  .iommu ──────────────────────────────────────────► │
└─────────────────────────────────────────────────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │   struct dev_iommu  │
               │  .iommu_dev ──────► iommu_device ──► iommu_ops
               │  .fwspec ─────────► iommu_fwspec
               │  .fault_param ────► iommu_fault_param ──► iopf_queue
               │  .priv            (driver private)
               └─────────────────────┘

┌──────────────────┐      ┌──────────────────────────────────────┐
│  iommu_device    │      │          iommu_ops                   │
│  .ops ──────────►│──────│  .probe_device()                     │
│  .list           │      │  .domain_alloc_paging()              │
│  .fwnode         │      │  .device_group()                     │
│  .singleton_group│      │  .identity_domain ──────────────────►│
└──────────────────┘      │  .blocked_domain  ──────────────────►│
                          │  .default_domain_ops ───────────────►│
                          └──────────────────────────────────────┘
                                        │ .domain_alloc*()
                                        ▼
                          ┌──────────────────────────────────────┐
                          │         iommu_domain                 │
                          │  .ops ────────────────────────────►  │
                          │          iommu_domain_ops            │
                          │            .attach_dev()             │
                          │            .map_pages()              │
                          │            .unmap_pages()            │
                          │            .iotlb_sync()             │
                          │            .iova_to_phys()           │
                          │  .dirty_ops ──────────────────────►  │
                          │          iommu_dirty_ops             │
                          │            .set_dirty_tracking()     │
                          │            .read_and_clear_dirty()   │
                          │  .owner ──────────────────────────►  │
                          │          iommu_ops (circular ref)    │
                          └──────────────────────────────────────┘

                          ┌──────────────────────────────────────┐
                          │         iommu_group                  │
                          │  (تعريفه في iommu_group.h)           │
                          │  ◄── devices (list of dev_iommu)     │
                          │  .default_domain ─────────────────►  │
                          │         iommu_domain                 │
                          └──────────────────────────────────────┘
                                 ▲
                    device_group()|
                    iommu_ops    │
                                 │
              struct device ─────┘

┌─────────────────────────────────────────────────────────┐
│                    IO Page Fault Chain                  │
│                                                         │
│  iommu_fault_param                                      │
│    .queue ──────────────────────► iopf_queue            │
│    .faults (list) ──────────────► iopf_group (pending)  │
│    .partial (list) ─────────────► iopf_fault (partial)  │
│                                                         │
│  iopf_group                                             │
│    .faults (list) ──────────────► iopf_fault            │
│    .last_fault ─────────────────► iopf_fault            │
│    .attach_handle ──────────────► iommu_attach_handle   │
│    .fault_param ────────────────► iommu_fault_param     │
│    .work ───────────────────────► workqueue processing  │
│                                                         │
│  iommu_attach_handle                                    │
│    .domain ─────────────────────► iommu_domain          │
│                                                         │
│  iommu_sva (embeds iommu_attach_handle)                 │
│    .handle.domain ──────────────► iommu_domain (SVA)   │
│    .dev ────────────────────────► struct device         │
└─────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   SVA Chain                              │
│                                                          │
│  mm_struct                                               │
│    .iommu_mm ────────────────────► iommu_mm_data         │
│                                      .pasid              │
│                                      .mm ──────► mm_struct (circular)
│                                      .sva_domains ──────►│
│                                         iommu_domain (SVA, linked list)
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│              Dirty Tracking Chain                        │
│                                                          │
│  iommu_dirty_bitmap                                      │
│    .bitmap ──────────────────────► iova_bitmap           │
│    .gather ──────────────────────► iommu_iotlb_gather    │
│                                      .freelist ─────────►│
│                                         iommu_pages_list │
└──────────────────────────────────────────────────────────┘
```

---

### دورة الحياة — Lifecycle Diagrams

#### دورة حياة الـ IOMMU Device (الـ Driver نفسه)

```
Driver Module Load
       │
       ▼
  iommu_device_sysfs_add()   ← ينشئ الـ sysfs entries
       │
       ▼
  iommu_device_register()    ← يضيفه للقائمة العامة + يعلّم .ready = true
       │
       ▼  (بعد اكتشاف devices)
  ops->probe_device()        ← يُنشئ dev->iommu (struct dev_iommu)
       │
       ▼
  ops->device_group()        ← يحدد/ينشئ iommu_group للـ device
       │
       ▼
  ops->probe_finalize()      ← إعداد نهائي بعد attach للـ group
       │
       ▼  (عند إيقاف الـ Driver)
  ops->release_device()      ← ينظف dev->iommu
       │
       ▼
  iommu_device_unregister()  ← يزيله من القائمة
       │
       ▼
  iommu_device_sysfs_remove() ← ينظف sysfs
```

---

#### دورة حياة الـ Domain

```
طلب domain جديد (مثلاً من DMA-API أو iommufd)
       │
       ▼
  ops->domain_alloc_paging()  أو  domain_alloc_paging_flags()
       │
       ▼
  iommu_domain مع:
    - .type = IOMMU_DOMAIN_DMA
    - .ops = driver's domain_ops
    - .pgsize_bitmap = supported sizes
       │
       ▼ (attach الـ device)
  domain_ops->attach_dev()    ← يربط الـ device بالـ domain
       │
       ▼  (mapping)
  domain_ops->map_pages()     ← يُضيف IOVA→PA mapping في الـ HW
       │
       ▼  (DMA يحصل بين device والـ PA)
       │
       ▼  (unmap)
  domain_ops->unmap_pages()   ← يزيل الـ mapping
  + يجمع الـ range في iommu_iotlb_gather
       │
       ▼
  domain_ops->iotlb_sync()    ← يعمل TLB flush للنطاق المجمّع
  + يحرر iommu_pages_list
       │
       ▼ (teardown)
  domain_ops->free()          ← يحرر الـ domain وكل بيانات الـ driver
```

---

#### دورة حياة الـ IO Page Fault (IOPF)

```
Hardware detects Page Fault
       │
       ▼
  iommu_report_device_fault()
       │
       ▼
  iopf_queue (workqueue)
       │
       ▼  (لو fault جزئية، تُضاف لـ fault_param->partial)
  iopf_fault جديد يُضاف لـ iopf_group
       │
       ▼  (لما يجي الـ last page في الـ group)
  iopf_group تكتمل
       │
       ▼
  queue_work() ─────────────────────────────────────────────────────────┐
                                                                         │
               [Workqueue Thread]                                        │
                    │ ◄──────────────────────────────────────────────────┘
                    ▼
  domain->iopf_handler(group)   ← handler محدد (SVA handler, iommufd handler, …)
                    │
                    ▼ (يعالج الـ fault — مثلاً يعمل page table walk أو يرفض)
                    │
                    ▼
  iopf_group_response(group, IOMMU_PAGE_RESP_SUCCESS / FAILURE / INVALID)
                    │
                    ▼
  ops->page_response()          ← يرسل الرد للـ hardware
                    │
                    ▼
  iopf_free_group(group)        ← ينظف الـ iopf_group وكل faults فيه
```

---

#### دورة حياة الـ SVA Bind

```
process userspace                kernel SVA core
       │
       ▼
  iommu_sva_bind_device(dev, mm)
       │
       ▼
  يخصص PASID للـ mm (لو مش موجود):
    iommu_alloc_global_pasid()
    ─────────────────────────────────► mm->iommu_mm->pasid = PASID
       │
       ▼
  ops->domain_alloc_sva(dev, mm)     ← ينشئ iommu_domain من نوع SVA
       │                                   (domain->mm = mm)
       ▼
  iommu_attach_device_pasid(domain, dev, pasid, &sva->handle)
       │                              ← يربط الـ PASID بالـ domain في الـ HW
       ▼
  يرجع struct iommu_sva*   ─────────► driver/userspace يستخدمه
       │
       │  (عند detach)
       ▼
  iommu_sva_unbind_device(handle)
       │
       ▼
  iommu_detach_device_pasid()        ← يفك الربط من الـ HW
       │
       ▼
  iommu_domain_free() / تحرير SVA domain
       │
       ▼
  (لو آخر bind للـ mm) mm_pasid_drop() ← يحرر الـ PASID
```

---

### Call Flow Diagrams

#### iommu_map — رحلة map كاملة

```
Driver/DMA-API calls iommu_map(domain, iova, paddr, size, prot, gfp)
  │
  ▼
iommu_map() [iommu.c]
  ├─ يتحقق من alignment مع pgsize_bitmap
  ├─ يحسب الـ page count
  └─► domain->ops->map_pages(domain, iova, paddr, pgsize, pgcount, prot, gfp, &mapped)
          │
          ▼  [مثلاً: arm_smmu_map_pages() في arm-smmu-v3]
        يكتب في الـ hardware page table
          │
          ▼
        domain->ops->iotlb_sync_map(domain, iova, size)  ← (اختياري) sync فوري
```

---

#### iommu_unmap — مع TLB gather

```
iommu_unmap(domain, iova, size)
  │
  ▼
iommu_unmap_fast(domain, iova, size, &gather)
  │
  ├─► domain->ops->unmap_pages(domain, iova, pgsize, pgcount, &gather)
  │       │
  │       ▼  [driver يمسح entries من الـ HW page table]
  │       └─ iommu_iotlb_gather_add_page(domain, &gather, iova, pgsize)
  │               ├─ لو disjoint: iommu_iotlb_sync(domain, &gather) أولاً
  │               └─ يحدّث gather.start, gather.end, gather.pgsize
  │
  ▼
iommu_iotlb_sync(domain, &gather)
  │
  ├─► domain->ops->iotlb_sync(domain, &gather)   ← flush الـ TLB للنطاق
  │
  └─ iommu_iotlb_gather_init(&gather)             ← reset للـ gather
     + تحرير gather.freelist
```

---

#### probe_device — ربط device بالـ IOMMU subsystem

```
device added to bus (e.g. PCI hotplug)
  │
  ▼
iommu_probe_device(dev)   [iommu.c, محمي بـ iommu_probe_device_lock]
  │
  ├─► ops->probe_device(dev)
  │       ├─ ينشئ dev->iommu (alloc dev_iommu)
  │       ├─ يملأ dev->iommu->fwspec
  │       └─ يرجع iommu_device*  ─────────────► dev->iommu->iommu_dev = iommu_device
  │
  ├─► ops->device_group(dev)
  │       └─ يرجع iommu_group* (موجود أو جديد)
  │
  ├─ iommu_group_add_device(group, dev)     ← يضيف الـ device للـ group
  │
  ├─ (لو group جديد) domain_alloc + attach_dev  ← ينشئ default domain
  │
  └─► ops->probe_finalize(dev)              ← إعداد نهائي
```

---

#### dirty tracking — live migration flow

```
VFIO/iommufd يطلب dirty tracking قبل migration
  │
  ▼
iommu_dirty_ops->set_dirty_tracking(domain, true)
  │                                  [driver يفعّل dirty bit في الـ page table]
  ▼
DMA يحصل ─────────────────────────────────► HW يسجّل dirty bits في الـ page table
  │
  ▼ (عند checkpoint)
iommu_dirty_bitmap_init(&dirty, bitmap, &gather)
  │
  ▼
iommu_dirty_ops->read_and_clear_dirty(domain, iova, size, flags, &dirty)
  │
  ├─ [driver يمشي في الـ page table]
  ├─ لكل PTE dirty: iommu_dirty_bitmap_record(&dirty, iova, len)
  │       ├─► iova_bitmap_set(bitmap, iova, len)      ← يسجل في الـ bitmap
  │       └─► iommu_iotlb_gather_add_range(&gather, iova, len)
  │
  └─ (لو !IOMMU_DIRTY_NO_CLEAR): يمسح الـ dirty bits
  │
  ▼
iommu_iotlb_sync(domain, &gather)   ← TLB flush للـ dirty ranges
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة وما تحميه

| الـ Lock | نوعه | ما يحميه |
|---|---|---|
| `dev_iommu.lock` | `mutex` | كل حقول الـ `dev_iommu` struct |
| `iommu_fault_param.lock` | `mutex` | قوائم `faults` و `partial` في الـ fault_param |
| `iopf_queue.lock` | `mutex` | قائمة `devices` في الـ iopf_queue |
| `iommu_probe_device_lock` | global `mutex` | كل عملية `probe_device` (serializes الـ probing) |
| RCU | `rcu_head` في `iommu_fault_param` | يتيح قراءة `fault_param` بدون lock، والتحرير آمن |
| `iommu_sva_lock` | (في iommu_sva.c) | قائمة `mm->iommu_mm->sva_domains` |

#### ترتيب الـ Locks (Lock Ordering)

```
iommu_probe_device_lock       (الخارجي — الأعلى رتبةً)
       │
       ▼
dev_iommu.lock                (لكل device على حدة)
       │
       ▼
iommu_fault_param.lock        (nested تحت dev_iommu.lock)
       │
       ▼
iopf_queue.lock               (مستقل، لا يُمسك مع fault_param.lock)
```

**ملاحظات مهمة:**
- **الـ `fault_param` يُقرأ تحت RCU** (`rcu_read_lock`) — فالتحرير يتم بـ `kfree_rcu()` وليس مباشرةً.
- **الـ `iommu_group_mutex_assert(dev)`** موجود للـ lockdep validation — يتحقق إن الـ group lock ممسوك عند استدعاء بعض الـ APIs.
- **الـ ops callbacks** (مثل `map_pages`, `attach_dev`) مش عندهم lock من الـ core — كل driver مسؤول عن حماية الـ page table الداخلية بتاعه.
- **الـ SVA domains list** في `mm->iommu_mm->sva_domains` محمية بـ `iommu_sva_lock` المستقل.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ Functions والـ APIs المهمة

#### Registration & Device Management

| Function | Category | Brief |
|---|---|---|
| `iommu_device_register()` | Registration | سجّل IOMMU hardware instance في الـ core |
| `iommu_device_unregister()` | Registration | إلغاء تسجيل الـ IOMMU device |
| `iommu_device_sysfs_add()` | Sysfs | أضف الـ IOMMU device لـ sysfs |
| `iommu_device_sysfs_remove()` | Sysfs | أزل من sysfs |
| `iommu_device_link()` | Sysfs | أنشئ sysfs link بين device والـ IOMMU |
| `iommu_device_unlink()` | Sysfs | احذف الـ sysfs link |
| `iommu_probe_device()` | Probe | اربط device بالـ IOMMU driver |
| `dev_to_iommu_device()` | Helper | استخرج `iommu_device` من `device` |
| `iommu_get_iommu_dev()` | Helper | macro للوصول لـ driver-level struct |

#### Domain Allocation & Lifecycle

| Function | Category | Brief |
|---|---|---|
| `iommu_paging_domain_alloc()` | Alloc | خصّص paging domain |
| `iommu_paging_domain_alloc_flags()` | Alloc | خصّص paging domain مع flags |
| `iommu_domain_free()` | Cleanup | حرّر الـ domain |
| `iommu_is_dma_domain()` | Helper | تحقق إن الـ domain هو DMA-API domain |

#### Attach / Detach

| Function | Category | Brief |
|---|---|---|
| `iommu_attach_device()` | Attach | اربط domain بـ device |
| `iommu_detach_device()` | Detach | افصل domain عن device |
| `iommu_attach_group()` | Attach | اربط domain بـ iommu group كامل |
| `iommu_detach_group()` | Detach | افصل domain عن group |
| `iommu_attach_device_pasid()` | PASID | اربط domain بـ PASID محدد |
| `iommu_detach_device_pasid()` | PASID | افصل domain عن PASID |
| `iommu_deferred_attach()` | Attach | attach مؤجّل حتى يُهيّأ الـ device driver |

#### Mapping & Translation

| Function | Category | Brief |
|---|---|---|
| `iommu_map()` | Map | map physical range على IOVA |
| `iommu_map_nosync()` | Map | map بدون TLB sync فوري |
| `iommu_sync_map()` | Sync | sync بعد `iommu_map_nosync` |
| `iommu_unmap()` | Unmap | unmap وعمل TLB sync كامل |
| `iommu_unmap_fast()` | Unmap | unmap بدون TLB sync (مع gather) |
| `iommu_map_sg()` | Map | map scatterlist على IOVA |
| `iommu_map_sgtable()` | Map | wrapper لـ `iommu_map_sg` من `sg_table` |
| `iommu_iova_to_phys()` | Translate | ترجم IOVA → physical address |

#### TLB Invalidation

| Function | Category | Brief |
|---|---|---|
| `iommu_flush_iotlb_all()` | TLB | flush كل الـ TLB بشكل synchronous |
| `iommu_iotlb_sync()` | TLB | sync الـ queued TLB ranges وأعد init الـ gather |
| `iommu_iotlb_gather_init()` | TLB Helper | initialise gather struct |
| `iommu_iotlb_gather_add_range()` | TLB Helper | أضف range لـ gather (address-based) |
| `iommu_iotlb_gather_add_page()` | TLB Helper | أضف page لـ gather (page-size-aware) |
| `iommu_iotlb_gather_is_disjoint()` | TLB Helper | تحقق إن range الجديد منفصل عن الـ gathered range |
| `iommu_iotlb_gather_queued()` | TLB Helper | تحقق إن الـ flush مؤجّل لـ flush_all |

#### Group Management

| Function | Category | Brief |
|---|---|---|
| `iommu_group_alloc()` | Group | خصّص iommu_group جديد |
| `iommu_group_get()` | Group | احصل على ref للـ group الخاص بـ device |
| `iommu_group_ref_get()` | Group | زد refcount |
| `iommu_group_put()` | Group | قلّل refcount وحرّر إذا وصل صفر |
| `iommu_group_add_device()` | Group | أضف device للـ group |
| `iommu_group_remove_device()` | Group | أزل device من group |
| `iommu_group_for_each_dev()` | Group | iterate على كل devices في group |
| `iommu_group_id()` | Group | استرجع ID رقمي للـ group |
| `iommu_group_default_domain()` | Group | استرجع الـ default domain للـ group |
| `iommu_group_get_iommudata()` | Group | استرجع driver private data للـ group |
| `iommu_group_set_iommudata()` | Group | اضبط driver private data |
| `iommu_group_set_name()` | Group | اضبط اسم الـ group |
| `pci_device_group()` | Group | grouping function لـ PCI devices |
| `generic_device_group()` | Group | generic grouping function |
| `generic_single_device_group()` | Group | كل device في group خاص بيه |
| `fsl_mc_device_group()` | Group | grouping لـ FSL-MC devices |

#### DMA Ownership

| Function | Category | Brief |
|---|---|---|
| `iommu_group_claim_dma_owner()` | DMA Owner | احجز ملكية DMA لـ group |
| `iommu_group_release_dma_owner()` | DMA Owner | حرّر ملكية DMA للـ group |
| `iommu_group_dma_owner_claimed()` | DMA Owner | تحقق إن الـ group له owner |
| `iommu_device_claim_dma_owner()` | DMA Owner | احجز ملكية DMA لـ device |
| `iommu_device_release_dma_owner()` | DMA Owner | حرّر ملكية DMA للـ device |

#### Reserved Regions

| Function | Category | Brief |
|---|---|---|
| `iommu_get_resv_regions()` | Resv | استرجع قائمة الـ reserved regions للـ device |
| `iommu_put_resv_regions()` | Resv | حرّر القائمة |
| `iommu_get_group_resv_regions()` | Resv | استرجع reserved regions لكل الـ group |
| `iommu_alloc_resv_region()` | Resv | خصّص `iommu_resv_region` descriptor |

#### Fault Handling

| Function | Category | Brief |
|---|---|---|
| `iommu_set_fault_handler()` | Fault | اضبط legacy fault handler على domain |
| `report_iommu_fault()` | Fault | بلّغ عن fault حصل |
| `iommu_report_device_fault()` | IOPF | أرسل IOPF fault لـ processing queue |
| `iopf_queue_alloc()` | IOPF | خصّص workqueue-backed fault queue |
| `iopf_queue_free()` | IOPF | حرّر الـ fault queue |
| `iopf_queue_add_device()` | IOPF | ربط device بـ fault queue |
| `iopf_queue_remove_device()` | IOPF | افصل device عن fault queue |
| `iopf_queue_flush_dev()` | IOPF | انتظر حتى تنتهي كل pending faults للـ device |
| `iopf_queue_discard_partial()` | IOPF | احذف incomplete page request groups |
| `iopf_group_response()` | IOPF | ابعت response للـ hardware |
| `iopf_free_group()` | IOPF | حرّر `iopf_group` بعد معالجته |

#### SVA (Shared Virtual Addressing)

| Function | Category | Brief |
|---|---|---|
| `iommu_sva_bind_device()` | SVA | اربط mm_struct بـ device عبر SVA |
| `iommu_sva_unbind_device()` | SVA | افصل الـ SVA bond |
| `iommu_sva_get_pasid()` | SVA | استرجع PASID المرتبط بالـ SVA handle |
| `iommu_sva_invalidate_kva_range()` | SVA | invalidate KVA range على كل SVA domains |
| `mm_pasid_init()` | SVA | initialize الـ PASID pointer في mm |
| `mm_valid_pasid()` | SVA | تحقق إن لـ mm PASID مخصص |
| `mm_get_enqcmd_pasid()` | SVA | استرجع PASID للـ ENQCMD instruction |
| `mm_pasid_drop()` | SVA | حرّر PASID resources لـ mm |
| `iommu_alloc_global_pasid()` | PASID | خصّص global PASID للـ device |
| `iommu_free_global_pasid()` | PASID | حرّر global PASID |

#### Firmware / Probe

| Function | Category | Brief |
|---|---|---|
| `iommu_fwspec_init()` | FW | أنشئ `iommu_fwspec` للـ device |
| `iommu_fwspec_add_ids()` | FW | أضف stream IDs للـ fwspec |
| `dev_iommu_fwspec_get()` | FW | استرجع fwspec pointer |
| `dev_iommu_fwspec_set()` | FW | اضبط fwspec pointer |
| `dev_iommu_priv_get()` | FW | استرجع driver private data |
| `dev_iommu_priv_set()` | FW | اضبط driver private data |

#### Dirty Tracking

| Function | Category | Brief |
|---|---|---|
| `iommu_dirty_bitmap_init()` | Dirty | initialise dirty bitmap + gather |
| `iommu_dirty_bitmap_record()` | Dirty | سجّل dirty IOVA range |

#### User-Space Data Copy

| Function/Macro | Category | Brief |
|---|---|---|
| `iommu_copy_struct_from_user` | uAPI | انسخ struct من user space مع type/size check |
| `iommu_copy_struct_from_user_array` | uAPI | انسخ entry واحد من user array |
| `iommu_copy_struct_from_full_user_array()` | uAPI | انسخ كل الـ array من user space |
| `iommu_copy_struct_to_user` | uAPI | انسخ struct لـ user space |

#### Misc

| Function | Category | Brief |
|---|---|---|
| `device_iommu_capable()` | Cap | تحقق من capability معيّنة للـ device |
| `iommu_group_has_isolated_msi()` | MSI | تحقق إن الـ group له isolated MSI |
| `iommu_get_msi_cookie()` | MSI | خصّص MSI cookie لـ DMA domain |
| `iommu_dma_prepare_msi()` | MSI | جهّز MSI address في IOMMU domain |
| `iommu_set_pgtable_quirks()` | Quirks | فعّل page table quirks |
| `iommu_set_dma_strict()` | Config | فرض strict DMA mode |
| `iommu_set_default_passthrough()` | Config | اضبط default mode = passthrough |
| `iommu_set_default_translated()` | Config | اضبط default mode = translated |
| `iommu_default_passthrough()` | Config | استرجع الـ default mode |
| `iommu_get_domain_for_dev()` | Query | استرجع الـ domain المرتبط بـ device |
| `iommu_driver_get_domain_for_dev()` | Query | استرجع domain من نظر الـ driver |
| `iommu_get_dma_domain()` | Query | استرجع الـ DMA-API domain للـ device |
| `iommu_device_use_default_domain()` | Default | استخدم الـ default domain للـ DMA API |
| `iommu_device_unuse_default_domain()` | Default | أوقف استخدام الـ default domain |
| `iommu_group_mutex_assert()` | Lockdep | تأكيد lockdep إن الـ group mutex مقفول |
| `tegra_dev_iommu_get_stream_id()` | Platform | استرجع stream ID لـ Tegra devices |
| `pci_dev_reset_iommu_prepare()` | PCI Reset | جهّز IOMMU قبل PCI device reset |
| `pci_dev_reset_iommu_done()` | PCI Reset | أنهِ IOMMU ops بعد PCI device reset |
| `iommu_debugfs_setup()` | Debug | إنشاء debugfs directory للـ IOMMU |

---

### Group 1: IOMMU Device Registration

هذه الفئة مسؤولة عن **تسجيل الـ IOMMU hardware instance** في الـ iommu core، وعن تمثيله في sysfs، وبناء الـ sysfs links بينه وبين الـ devices التي يديرها. الـ driver يستدعيها في مرحلة الـ probe الخاصة بيه.

---

#### `iommu_device_register()`

```c
int iommu_device_register(struct iommu_device *iommu,
                           const struct iommu_ops *ops,
                           struct device *hwdev);
```

بتسجّل الـ IOMMU hardware instance في الـ iommu core list وتربطه بالـ `iommu_ops`. بعد ما تكتمل بنجاح، بتضبط `iommu->ready = true`، وده بيخلي الـ core يقدر يستخدم الـ IOMMU لـ probe الـ devices المرتبطة بيه.

- **`iommu`**: الـ `iommu_device` struct اللي الـ driver خصّصه.
- **`ops`**: مؤشر على الـ `iommu_ops` الخاصة بالـ driver.
- **`hwdev`**: الـ physical device اللي يمثّل الـ hardware (للـ sysfs).
- **Return**: `0` عند النجاح، error code سالب عند الفشل.
- **Key details**: بعد الـ register، الـ bus notifier ممكن يُطلق على طول لـ probe الـ devices المعلّقة. بيحتاج الـ driver يكون قد أعدّ كل الـ `iommu_ops` قبل الاستدعاء.

---

#### `iommu_device_unregister()`

```c
void iommu_device_unregister(struct iommu_device *iommu);
```

بتعكس الـ `iommu_device_register()`. بتزيل الـ IOMMU من الـ global list وبتضبط `iommu->ready = false`. لازم تتستدعى قبل ما الـ driver يحرّر resources الـ IOMMU hardware.

- **`iommu`**: الـ instance المراد إلغاء تسجيله.
- **Return**: void.

---

#### `iommu_device_sysfs_add()`

```c
int iommu_device_sysfs_add(struct iommu_device *iommu,
                            struct device *parent,
                            const struct attribute_group **groups,
                            const char *fmt, ...);
```

بتنشئ `struct device` في sysfs تحت `parent` تمثّل الـ IOMMU instance. بتدعم printf-style format للاسم وبتضيف attribute groups مخصصة.

- **`iommu`**: الـ iommu device المراد إضافته.
- **`parent`**: الـ parent device في sysfs.
- **`groups`**: attribute groups إضافية (NULL-terminated array).
- **`fmt, ...`**: اسم الـ device بصيغة printf.
- **Return**: `0` أو error code سالب.
- **Key**: الـ `__printf(4, 5)` annotation بتفرض compile-time format check.

---

#### `iommu_device_link()` / `iommu_device_unlink()`

```c
int  iommu_device_link(struct iommu_device *iommu, struct device *link);
void iommu_device_unlink(struct iommu_device *iommu, struct device *link);
```

بينشئوا/بيحذفوا sysfs symlink بين الـ device المُدار والـ IOMMU instance اللي بيخدمه. مهم لأدوات user-space زي `iommu-setup` لتتبع أي device متصل بأي IOMMU.

---

#### `dev_to_iommu_device()`

```c
static inline struct iommu_device *dev_to_iommu_device(struct device *dev)
{
    return (struct iommu_device *)dev_get_drvdata(dev);
}
```

Helper بسيطة — بتفترض إن `dev` هو الـ `iommu_device->dev` وبترجع الـ parent `iommu_device`. الـ driver نفسه بيستخدمها داخلياً.

---

#### `iommu_get_iommu_dev()` macro

```c
#define iommu_get_iommu_dev(dev, type, member) \
    container_of(__iommu_get_iommu_dev(dev), type, member)
```

بيوصّل للـ driver-level IOMMU struct اللي embeds الـ `iommu_device` عن طريق `container_of`. الـ `__iommu_get_iommu_dev()` بتسترجع `dev->iommu->iommu_dev`.

---

### Group 2: Domain Allocation & Lifecycle

الـ **IOMMU domain** هو الوحدة الأساسية للـ address space في الـ IOMMU subsystem. كل domain بيمثّل I/O page table مستقلة. هذه الفئة بتغطي إنشاء وتحرير الـ domains.

---

#### `iommu_paging_domain_alloc()` / `iommu_paging_domain_alloc_flags()`

```c
struct iommu_domain *iommu_paging_domain_alloc_flags(struct device *dev,
                                                      unsigned int flags);
static inline struct iommu_domain *iommu_paging_domain_alloc(struct device *dev);
```

بتخصّص `iommu_domain` من النوع الـ paging (يدعم `iommu_map`/`iommu_unmap`). الـ `_flags` version بتمرّر flags إضافية — مثلاً `IOMMU_DOMAIN_CACHE` لضمان coherency. الـ core بيستدعي `iommu_ops->domain_alloc_paging_flags()` أو `domain_alloc_paging()` حسب ما ينفذه الـ driver.

- **`dev`**: device يُستخدم لتحديد الـ IOMMU المسؤول عن التخصيص.
- **`flags`**: flags إضافية للـ domain (0 في النسخة البسيطة).
- **Return**: pointer على `iommu_domain` أو `ERR_PTR` عند الفشل.
- **Key**: الـ domain المُخصص بيكون في حالة `IOMMU_DOMAIN_UNMANAGED` مبدئياً.

---

#### `iommu_domain_free()`

```c
void iommu_domain_free(struct iommu_domain *domain);
```

بتحرّر الـ domain وكل الـ page table resources المرتبطة بيه. لازم يكون الـ domain detached من كل الـ devices قبل الاستدعاء، وإلا WARN_ON.

- **`domain`**: الـ domain المراد تحريره.
- **Key**: بتستدعي `domain->ops->free()` — لا تستدعيها على الـ `identity_domain` أو `blocked_domain` لأنهم static.

---

#### `iommu_is_dma_domain()`

```c
static inline bool iommu_is_dma_domain(struct iommu_domain *domain)
{
    return domain->type & __IOMMU_DOMAIN_DMA_API;
}
```

تحقق سريع إن الـ domain مُدار بواسطة الـ DMA-API layer (مش managed يدوياً).

---

### Group 3: Device Attach / Detach

الربط بين الـ domain والـ device هو اللي بيفعّل فعلياً الـ I/O address translation.

---

#### `iommu_attach_device()`

```c
int iommu_attach_device(struct iommu_domain *domain, struct device *dev);
```

بتربط الـ domain بالـ device المحدد (بالـ RID بتاعه). بتستدعي `domain->ops->attach_dev()` داخلياً. الـ device بيبقى يُرجم كل الـ DMA transactions عبر الـ page table بتاع الـ domain ده.

- **`domain`**: الـ domain المراد ربطه.
- **`dev`**: الـ end-point device.
- **Return**: `0` أو error code. `EINVAL` لو الـ domain/device غير متوافقَين. `EBUSY` لو الـ device already attached.
- **Locking**: بتحتاج الـ group lock يكون مقفول من المستدعي في سياقات معينة. الـ core بيضمن ده في معظم الحالات.

---

#### `iommu_detach_device()`

```c
void iommu_detach_device(struct iommu_domain *domain, struct device *dev);
```

بتفصل الـ device عن الـ domain. بعد الـ detach، الـ device بيُربط تلقائياً بالـ `blocked_domain` أو `identity_domain` حسب إعداد الـ driver.

---

#### `iommu_attach_group()` / `iommu_detach_group()`

```c
int iommu_attach_group(struct iommu_domain *domain, struct iommu_group *group);
void iommu_detach_group(struct iommu_domain *domain, struct iommu_group *group);
```

نفس مبدأ الـ `attach_device` لكن على مستوى الـ group كاملاً. بتعمل iterate على كل devices في الـ group وبتعمل attach/detach لكل واحد. مهم جداً لسيناريوهات الـ VFIO حيث الـ group بيتربط بـ container domain.

---

#### `iommu_attach_device_pasid()`

```c
int iommu_attach_device_pasid(struct iommu_domain *domain,
                               struct device *dev, ioasid_t pasid,
                               struct iommu_attach_handle *handle);
```

بتربط domain بـ PASID محدد للـ device — ده اللي بيمكّن الـ **PASID-based translation** حيث كل process أو context بيستخدم address space مختلف. الـ `handle` بيُخزَّن في الـ group لربط الـ domain بالـ PASID.

- **`pasid`**: الـ Process Address Space ID (IOMMU_FIRST_GLOBAL_PASID وما بعده).
- **`handle`**: يُخزَّن من المستدعي — لازم يبقى valid طول فترة الـ attach.
- **Return**: `0` أو error.
- **Use case**: مستخدم في SVA وـ VFIO mediated devices.

---

#### `iommu_detach_device_pasid()`

```c
void iommu_detach_device_pasid(struct iommu_domain *domain,
                                struct device *dev, ioasid_t pasid);
```

بتعكس `iommu_attach_device_pasid()`. لازم يتستدعى قبل `iommu_free_global_pasid()`.

---

#### `iommu_deferred_attach()`

```c
int iommu_deferred_attach(struct device *dev, struct iommu_domain *domain);
```

لما الـ `iommu_ops->is_attach_deferred()` ترجع `true`، الـ attach بيتأجّل حتى الـ device driver نفسه يكون جاهز. بتُستدعى من الـ device driver (مش الـ iommu driver) لإكمال الـ attach.

---

### Group 4: IOVA Mapping & Translation

هذه الفئة هي **قلب عمل الـ IOMMU** — إنشاء وإزالة الـ virtual-to-physical mappings في الـ I/O page table.

---

#### `iommu_map()`

```c
int iommu_map(struct iommu_domain *domain, unsigned long iova,
              phys_addr_t paddr, size_t size, int prot, gfp_t gfp);
```

بتنشئ mapping من `iova` → `paddr` بحجم `size` في الـ domain. بتستدعي `domain->ops->map_pages()` بعد ما تتحقق من alignment وـ page size. بعد الـ map بتستدعي `iotlb_sync_map()` تلقائياً.

- **`iova`**: الـ I/O Virtual Address الابتدائي — لازم يكون aligned على `pgsize`.
- **`paddr`**: الـ physical address الهدف.
- **`size`**: الحجم بالـ bytes — لازم يكون من مضاعفات حجم الـ page المدعوم.
- **`prot`**: مزيج من `IOMMU_READ | IOMMU_WRITE | IOMMU_CACHE | IOMMU_NOEXEC | IOMMU_MMIO | IOMMU_PRIV`.
- **`gfp`**: flags للتخصيص الداخلي للـ page table entries.
- **Return**: `0` أو error.

**Pseudocode:**
```
iommu_map(domain, iova, paddr, size, prot, gfp):
    validate alignment
    while size > 0:
        pgsize = largest supported page size ≤ size
        domain->ops->map_pages(iova, paddr, pgsize, 1, prot)
        iova += pgsize; paddr += pgsize; size -= pgsize
    domain->ops->iotlb_sync_map(domain, original_iova, total_size)
```

---

#### `iommu_map_nosync()` / `iommu_sync_map()`

```c
int iommu_map_nosync(struct iommu_domain *domain, unsigned long iova,
                     phys_addr_t paddr, size_t size, int prot, gfp_t gfp);
int iommu_sync_map(struct iommu_domain *domain, unsigned long iova, size_t size);
```

الـ `nosync` version بتعمل الـ map بدون استدعاء `iotlb_sync_map` — بتسمح بعمل batch من الـ mappings ثم sync مرة واحدة. مفيدة جداً لتحسين الأداء في الـ DMA-API.

---

#### `iommu_unmap()`

```c
size_t iommu_unmap(struct iommu_domain *domain, unsigned long iova, size_t size);
```

بتمسح الـ mappings ابتداءً من `iova` بحجم `size` وبتعمل **full TLB invalidation** متزامن. بترجع عدد الـ bytes اللي اتمسحت فعلاً.

- **Key**: بعد الـ unmap، الـ pages مش آمنة للـ reuse حتى تتأكد إن الـ TLB اتفلش — الـ `iommu_unmap` بيضمن ده.

---

#### `iommu_unmap_fast()`

```c
size_t iommu_unmap_fast(struct iommu_domain *domain, unsigned long iova,
                         size_t size, struct iommu_iotlb_gather *iotlb_gather);
```

زي `iommu_unmap` لكن **بدون** TLB flush فوري. بتضيف الـ range للـ `iotlb_gather` للـ flush لاحقاً في batch. الـ pages المحرّرة تُضاف لـ `gather->freelist` ولا تُحرَّر حتى `iommu_iotlb_sync()`.

- **Use case**: الـ DMA-API flush queue (IOMMU_DOMAIN_DMA_FQ).

---

#### `iommu_map_sg()` / `iommu_map_sgtable()`

```c
ssize_t iommu_map_sg(struct iommu_domain *domain, unsigned long iova,
                     struct scatterlist *sg, unsigned int nents, int prot, gfp_t gfp);
static inline ssize_t iommu_map_sgtable(struct iommu_domain *domain,
                                         unsigned long iova, struct sg_table *sgt, int prot);
```

بتعمل map على scatterlist كامل على نطاق IOVA متصل. كل entry في الـ SG بتتحول لـ mapping مستقلة. الـ `iommu_map_sgtable` مجرد wrapper بتمرّر `sgt->sgl` و `sgt->orig_nents`.

- **`nents`**: عدد entries الـ scatterlist.
- **Return**: عدد الـ bytes المُضافة كـ `ssize_t`، أو error سالب.

---

#### `iommu_iova_to_phys()`

```c
phys_addr_t iommu_iova_to_phys(struct iommu_domain *domain, dma_addr_t iova);
```

بترجم IOVA لـ physical address عن طريق walk الـ I/O page table. بتستدعي `domain->ops->iova_to_phys()`.

- **Return**: الـ physical address، أو `0` لو الـ IOVA مش مُعيَّن (not mapped).
- **Use case**: debugging، وـ VFIO guest physical address translation.

---

### Group 5: TLB Invalidation & Gather

الـ **IOTLB** (I/O TLB) هو الـ cache للـ page table walks في الـ IOMMU hardware. بعد أي unmap، لازم نعمل invalidation للتأكد من عدم استخدام entries قديمة.

---

#### `iommu_flush_iotlb_all()`

```c
static inline void iommu_flush_iotlb_all(struct iommu_domain *domain)
```

بتعمل **synchronous flush** لكل الـ IOTLB entries الخاصة بالـ domain. أبطأ لكن أضمن — بتُستخدم لما الـ gather queue تكون ممتلئة أو في مسارات الـ suspend/resume.

---

#### `iommu_iotlb_sync()`

```c
static inline void iommu_iotlb_sync(struct iommu_domain *domain,
                                     struct iommu_iotlb_gather *iotlb_gather)
```

بتعمل flush للـ gathered ranges وبترسلها لـ `domain->ops->iotlb_sync()`. بعد الـ sync بتعمل `iommu_iotlb_gather_init()` لإعادة تهيئة الـ gather struct.

- **Locking**: لا يوجد locking داخلي — المستدعي مسؤول عن الـ synchronization.
- **Side effect**: بتحرّر الـ `gather->freelist` pages بعد الـ flush.

---

#### `iommu_iotlb_gather_init()`

```c
static inline void iommu_iotlb_gather_init(struct iommu_iotlb_gather *gather)
```

بتـ initialize الـ gather struct بقيم افتراضية:
- `start = ULONG_MAX` (أي range جديدة هتكون أصغر منه)
- `end = 0`
- `pgsize = 0`
- `freelist` فاضية

---

#### `iommu_iotlb_gather_add_range()`

```c
static inline void iommu_iotlb_gather_add_range(struct iommu_iotlb_gather *gather,
                                                  unsigned long iova, size_t size)
```

بتوسّع الـ gathered range لتشمل `[iova, iova+size-1]`. بس بتتبع الـ range بدون اعتبار لـ page size — مناسبة للـ IOMMUs اللي بتقبل arbitrary ranges.

---

#### `iommu_iotlb_gather_add_page()`

```c
static inline void iommu_iotlb_gather_add_page(struct iommu_domain *domain,
                                                struct iommu_iotlb_gather *gather,
                                                unsigned long iova, size_t size)
```

أذكى من الـ `add_range` — بتتحقق إن الـ page size متطابقة مع الـ gather الحالي، وإلا بتعمل sync فوري وبتبدأ gather جديد. كمان لو الـ range الجديدة **disjoint** عن الحالية، بتعمل sync.

**Pseudocode:**
```
add_page(domain, gather, iova, size):
    if gather->pgsize != size OR is_disjoint(gather, iova, size):
        iommu_iotlb_sync(domain, gather)   // flush current, reset gather
    gather->pgsize = size
    add_range(gather, iova, size)
```

---

#### `iommu_iotlb_gather_is_disjoint()`

```c
static inline bool iommu_iotlb_gather_is_disjoint(
    struct iommu_iotlb_gather *gather, unsigned long iova, size_t size)
```

بتتحقق إن `[iova, iova+size-1]` لا يتقاطع ولا يتجاور مع `[gather->start, gather->end]`. مهمة لـ IOMMUs اللي فيها flush على range منفصلة أسرع من merge.

---

#### `iommu_iotlb_gather_queued()`

```c
static inline bool iommu_iotlb_gather_queued(struct iommu_iotlb_gather *gather)
```

بترجع `true` لو الـ flush هيتم كـ `flush_all` مؤجّل (DMA_FQ mode) مش كـ sync فوري.

---

### Group 6: Group Management

الـ **iommu_group** بيمثّل مجموعة devices يجب أن تشترك في نفس الـ domain (بسبب isolation requirements أو PCIe topology).

---

#### `iommu_group_alloc()`

```c
struct iommu_group *iommu_group_alloc(void);
```

بتخصص وتـ initialise `iommu_group` جديد. بتنشئ كمان `/sys/kernel/iommu_groups/<id>/` directory. الـ ID بيُخصّص بشكل atomic من counter global.

- **Return**: pointer على الـ group أو `ERR_PTR(-ENOMEM)`.

---

#### `iommu_group_get()` / `iommu_group_ref_get()` / `iommu_group_put()`

```c
struct iommu_group *iommu_group_get(struct device *dev);
struct iommu_group *iommu_group_ref_get(struct iommu_group *group);
void iommu_group_put(struct iommu_group *group);
```

الـ `iommu_group` بيستخدم **kobject reference counting**. الـ `get` بتزيد الـ refcount وبترجع pointer، والـ `put` بتقلّله وبتحرّر الـ group لو وصل صفر. لازم كل `get` يقابله `put`.

---

#### `iommu_group_add_device()`

```c
int iommu_group_add_device(struct iommu_group *group, struct device *dev);
```

بتضيف device للـ group. بتنشئ sysfs links في `/sys/kernel/iommu_groups/<id>/devices/` وبتضيف الـ device لـ `group->devices` list. بتستدعي الـ group notifiers.

- **Locking**: بتاخد `group->mutex` داخلياً.
- **Return**: `0` أو error (مثلاً `EBUSY` لو الـ device already in a group).

---

#### `iommu_group_remove_device()`

```c
void iommu_group_remove_device(struct device *dev);
```

بتزيل الـ device من group وبتحرّر الـ sysfs links. لو الـ group فضي، بيتحرّر تلقائياً.

---

#### `iommu_group_for_each_dev()`

```c
int iommu_group_for_each_dev(struct iommu_group *group, void *data,
                               int (*fn)(struct device *, void *));
```

بتعمل iterate على كل devices في الـ group وبتستدعي `fn` لكل واحد. لو `fn` رجعت قيمة غير صفر، الـ iteration بتوقف وبتُرجع نفس القيمة.

- **Locking**: بتاخد `group->mutex`.

---

#### `iommu_group_set_iommudata()` / `iommu_group_get_iommudata()`

```c
void iommu_group_set_iommudata(struct iommu_group *group, void *iommu_data,
                                void (*release)(void *iommu_data));
void *iommu_group_get_iommudata(struct iommu_group *group);
```

بيخزّنوا driver-private data داخل الـ group. الـ `release` callback بتتستدعى تلقائياً لما الـ group يتحرّر — مفيد لـ cleanup الـ platform-specific resources.

---

#### `pci_device_group()` / `generic_device_group()` / `generic_single_device_group()`

```c
struct iommu_group *pci_device_group(struct device *dev);
struct iommu_group *generic_device_group(struct device *dev);
struct iommu_group *generic_single_device_group(struct device *dev);
```

الـ helper functions اللي الـ `iommu_ops->device_group` op بتبعت إليها عادةً:
- **`pci_device_group`**: بتجمّع devices حسب PCIe topology (devices في نفس الـ PCIe segment مع no ACS).
- **`generic_device_group`**: بتجمّع devices بناءً على bus/topology generic logic.
- **`generic_single_device_group`**: كل device في group وحده — أكثر أماناً لكن أقل كفاءة.

---

### Group 7: Fault Handling (Legacy & IOPF)

في الـ IOMMU subsystem في فئتين من الـ fault handling:
1. **Legacy** (`iommu_fault_handler_t`): domain-level، بتنادي callback مباشرة.
2. **IOPF** (I/O Page Faults): per-device، قائم على workqueue، بيدعم الـ PCI PRI (Page Request Interface).

---

#### `iommu_set_fault_handler()`

```c
void iommu_set_fault_handler(struct iommu_domain *domain,
                               iommu_fault_handler_t handler, void *token);
```

بتضبط legacy fault handler على الـ domain. بتُستخدم بشكل رئيسي من الـ VFIO لاستقبال unhandled page faults.

- **`handler`**: `int (*)(struct iommu_domain *, struct device *, unsigned long iova, int flags, void *token)`.
- **`token`**: opaque pointer بيتمرّر للـ handler.
- **Cookie type**: بتضبط `domain->cookie_type = IOMMU_COOKIE_FAULT_HANDLER`.

---

#### `report_iommu_fault()`

```c
int report_iommu_fault(struct iommu_domain *domain, struct device *dev,
                        unsigned long iova, int flags);
```

بتُستدعى من الـ IOMMU driver لما يكتشف fault. بتستدعي الـ `domain->handler` لو موجود.

- **`flags`**: `IOMMU_FAULT_READ` أو `IOMMU_FAULT_WRITE`.
- **Return**: `0` لو الـ handler عالج الـ fault، `-ENOSYS` لو مفيش handler.

---

#### `iopf_queue_alloc()` / `iopf_queue_free()`

```c
struct iopf_queue *iopf_queue_alloc(const char *name);
void iopf_queue_free(struct iopf_queue *queue);
```

بتخصّصوا/بيحرّروا الـ `iopf_queue` اللي بيتضمّن `workqueue_struct`. الـ workqueue ده مسؤول عن معالجة الـ IOPF faults بشكل async.

- **`name`**: اسم الـ workqueue (ظاهر في `ps` و `/proc`).

---

#### `iopf_queue_add_device()` / `iopf_queue_remove_device()`

```c
int iopf_queue_add_device(struct iopf_queue *queue, struct device *dev);
void iopf_queue_remove_device(struct iopf_queue *queue, struct device *dev);
```

بتربطوا/بيفصلوا الـ device بالـ fault queue. بعد الـ add، أي fault على الـ device بيتوجه للـ queue ده.

- **Locking**: بتاخدوا `queue->lock`.

---

#### `iommu_report_device_fault()`

```c
int iommu_report_device_fault(struct device *dev, struct iopf_fault *evt);
```

الـ entry point الرئيسية من الـ IOMMU driver لرفع IOPF fault. بتضيف الـ fault لـ `iommu_fault_param->partial` list (لو مش last) أو بتبني `iopf_group` كاملة وبتـ queue الـ work item لو الـ fault هو آخر في الـ group.

**Pseudocode:**
```
report_device_fault(dev, evt):
    fault_param = dev->iommu->fault_param (rcu protected)
    if not last_page_in_group:
        add to fault_param->partial list
        return
    build iopf_group from partial list + this fault
    group->attach_handle = find handle for device RID/PASID
    INIT_WORK(&group->work, iopf_handler_work)
    queue_work(fault_param->queue->wq, &group->work)
```

---

#### `iopf_group_response()`

```c
void iopf_group_response(struct iopf_group *group,
                          enum iommu_page_response_code status);
```

بعد ما الـ handler معالج الـ group، بتبعت response للـ IOMMU hardware. بتستدعي `iommu_ops->page_response()` لكل fault في الـ group.

- **`status`**: `IOMMU_PAGE_RESP_SUCCESS`, `IOMMU_PAGE_RESP_INVALID`, أو `IOMMU_PAGE_RESP_FAILURE`.

---

#### `iopf_free_group()`

```c
void iopf_free_group(struct iopf_group *group);
```

بتحرّر الـ `iopf_group` وكل الـ `iopf_fault` entries فيه. لازم تتستدعى بعد `iopf_group_response()`.

---

#### `iopf_queue_flush_dev()`

```c
int iopf_queue_flush_dev(struct device *dev);
```

بتنتظر synchronously حتى تنتهي كل الـ pending faults للـ device. مهمة قبل `iopf_queue_remove_device()` لضمان عدم وجود work items في flight.

---

### Group 8: SVA (Shared Virtual Addressing)

الـ **SVA** بتسمح للـ device إنها تستخدم نفس الـ virtual address space بتاع الـ process — بدون copy. كل process له PASID مستقل بـ page table منفصلة تمثّل الـ `mm_struct`.

---

#### `iommu_sva_bind_device()`

```c
struct iommu_sva *iommu_sva_bind_device(struct device *dev, struct mm_struct *mm);
```

بتربط الـ `mm_struct` بالـ device — بتنشئ SVA domain إذا مش موجود، وبترجع handle. الـ `iommu_sva->handle.domain` بيبقى domain بـ page table مشتركة مع الـ CPU.

- **`dev`**: الـ device اللي هيستخدم الـ SVA.
- **`mm`**: الـ mm_struct بتاع الـ process.
- **Return**: `iommu_sva *` أو `ERR_PTR`.
- **Locking**: بتاخد `iommu_sva_lock`.

---

#### `iommu_sva_unbind_device()`

```c
void iommu_sva_unbind_device(struct iommu_sva *handle);
```

بتقلّل الـ refcount على الـ SVA handle. لو وصل صفر، بتحرّر الـ SVA domain وبتـ detach من الـ device.

---

#### `iommu_sva_get_pasid()`

```c
u32 iommu_sva_get_pasid(struct iommu_sva *handle);
```

بترجع الـ PASID المخصص للـ mm المرتبطة بالـ handle ده. الـ device driver بيستخدمه لبرمجة الـ PASID في الـ submission queues.

---

#### `mm_pasid_init()` / `mm_valid_pasid()` / `mm_get_enqcmd_pasid()`

```c
static inline void mm_pasid_init(struct mm_struct *mm);
static inline bool mm_valid_pasid(struct mm_struct *mm);
static inline u32 mm_get_enqcmd_pasid(struct mm_struct *mm);
```

- **`mm_pasid_init`**: بتـ zero الـ `mm->iommu_mm` pointer في `dup_mm()` لتجنب الـ use-after-free.
- **`mm_valid_pasid`**: بتتحقق (عبر `READ_ONCE`) إن الـ mm عنده PASID مخصص.
- **`mm_get_enqcmd_pasid`**: بترجع الـ PASID للاستخدام في ENQCMD instruction (Intel DSA وما شابه).

---

#### `iommu_alloc_global_pasid()` / `iommu_free_global_pasid()`

```c
ioasid_t iommu_alloc_global_pasid(struct device *dev);
void iommu_free_global_pasid(ioasid_t pasid);
```

بيخصّصوا/بيحرّروا PASID من الـ global PASID space (من `IOMMU_FIRST_GLOBAL_PASID` للأمام). الـ global PASIDs بتنفع لـ kernel-managed contexts زي SVA.

---

### Group 9: Firmware / FwSpec Management

الـ **iommu_fwspec** بتحتوي على الـ firmware-provided metadata لتعريف الـ device للـ IOMMU — أهمها الـ stream IDs.

---

#### `iommu_fwspec_init()`

```c
int iommu_fwspec_init(struct device *dev, struct fwnode_handle *iommu_fwnode);
```

بتخصّص `iommu_fwspec` للـ device وبتضبط الـ firmware node الخاص بالـ IOMMU. بتُستدعى من الـ bus/platform driver في الـ `of_xlate` أو DT parsing.

- **Return**: `0` أو `ENOMEM`.

---

#### `iommu_fwspec_add_ids()`

```c
int iommu_fwspec_add_ids(struct device *dev, const u32 *ids, int num_ids);
```

بتضيف stream IDs (أو requestor IDs) للـ fwspec. الـ IOMMU driver بيستخدمهم لتحديد أي transactions تنتمي لأي device.

---

#### `dev_iommu_priv_get()` / `dev_iommu_priv_set()`

```c
static inline void *dev_iommu_priv_get(struct device *dev);
void dev_iommu_priv_set(struct device *dev, void *priv);
```

بيمكّنوا الـ IOMMU driver من تخزين واسترجاع private data خاصة بالـ device. البيانات بتُخزَّن في `dev->iommu->priv`.

---

### Group 10: Dirty Tracking

مفيدة في سيناريوهات الـ **VM live migration** — لتتبع صفحات الـ I/O اللي اتغيّرت خلال الـ migration.

---

#### `iommu_dirty_bitmap_init()`

```c
static inline void iommu_dirty_bitmap_init(struct iommu_dirty_bitmap *dirty,
                                             struct iova_bitmap *bitmap,
                                             struct iommu_iotlb_gather *gather)
```

بتـ initialise الـ dirty tracking state — بتربط الـ `iova_bitmap` بالـ `gather` structure. الـ bitmap بتُستخدم لتتبع الـ dirty pages والـ gather لـ TLB invalidation.

---

#### `iommu_dirty_bitmap_record()`

```c
static inline void iommu_dirty_bitmap_record(struct iommu_dirty_bitmap *dirty,
                                               unsigned long iova, unsigned long length)
```

بتُعلّم الـ IOVA range كـ dirty في الـ bitmap وبتضيفها للـ gather. بتُستدعى من داخل `iommu_dirty_ops->read_and_clear_dirty()` لكل dirty page.

---

### Group 11: User-Space Data Copy (iommufd uAPI)

الـ iommufd subsystem بيسمح لـ user space بإدارة الـ IOMMU domains. هذه الـ helpers بتضمن **type-safe، backward-compatible** copy بين kernel وـ user space.

---

#### `iommu_copy_struct_from_user` macro

```c
#define iommu_copy_struct_from_user(kdst, user_data, data_type, min_last)
```

بتنسخ struct من user space مع التحقق من:
1. `user_data->type == data_type` (type safety).
2. `user_data->len >= offsetofend(typeof(*kdst), min_last)` (backward compat — القديم لازم يشمل الحقول الأساسية).
3. `sizeof(*kdst) >= user_data->len` (kernel struct كبير بما يكفي).

- **`kdst`**: kernel destination pointer.
- **`min_last`**: اسم آخر field في الـ initial version من الـ struct.

---

#### `iommu_copy_struct_from_user_array` macro

```c
#define iommu_copy_struct_from_user_array(kdst, user_array, data_type, index, min_last)
```

بتنسخ entry واحد من `index` في user array. بتحسب الـ offset تلقائياً: `uptr + entry_len * index`.

---

#### `iommu_copy_struct_from_full_user_array()`

```c
static inline int iommu_copy_struct_from_full_user_array(
    void *kdst, size_t kdst_entry_size,
    struct iommu_user_data_array *user_array, unsigned int data_type);
```

بتنسخ **كل** entries الـ array. لو `entry_len == kdst_entry_size`، بتستخدم `copy_from_user` مرة واحدة (fast path). غير كده بتعمل `copy_struct_from_user` لكل entry (بيعالج الـ size mismatch بين kernel وـ user structs).

---

#### `iommu_copy_struct_to_user` macro

```c
#define iommu_copy_struct_to_user(user_data, ksrc, data_type, min_last)
```

عكس الـ `from_user` — بينسخ struct من kernel لـ user space. بيستخدم `copy_struct_to_user()` اللي بتتعامل مع الـ zero-padding للحقول الجديدة غير المعروفة للـ old user space.

---

### Group 12: Reserved Regions

الـ **reserved regions** هي ranges لا يجب على الـ DMA-API ولا الـ IOMMU driver الـ map فيها — إما لأن الـ hardware يحتاجها للـ 1:1 mapping أو لأنها restricted تماماً.

---

#### `iommu_get_resv_regions()` / `iommu_put_resv_regions()`

```c
void iommu_get_resv_regions(struct device *dev, struct list_head *list);
void iommu_put_resv_regions(struct device *dev, struct list_head *list);
```

بتستدعيان `iommu_ops->get_resv_regions()` للحصول على قائمة `iommu_resv_region`. الـ `put` بتستدعي الـ `free` callback لكل region.

---

#### `iommu_alloc_resv_region()`

```c
struct iommu_resv_region *iommu_alloc_resv_region(phys_addr_t start, size_t length,
                                                    int prot, enum iommu_resv_type type,
                                                    gfp_t gfp);
```

بتخصّص وتـ initialise `iommu_resv_region` descriptor. بتُستخدم من الـ IOMMU driver في `get_resv_regions` op.

- **`type`**: `IOMMU_RESV_DIRECT`, `IOMMU_RESV_RESERVED`, `IOMMU_RESV_MSI`, etc.

---

### Group 13: DMA Ownership Control

نظام الـ **DMA ownership** بيمنع تعارض بين الـ kernel DMA-API والـ user-space VFIO/iommufd على نفس الـ device.

---

#### `iommu_group_claim_dma_owner()`

```c
int iommu_group_claim_dma_owner(struct iommu_group *group, void *owner);
```

بتسجّل `owner` كـ DMA owner للـ group. لو في owner تاني → ترجع `EBUSY`. الـ VFIO وـ iommufd بيستخدمواها عند فتح device.

---

#### `iommu_group_release_dma_owner()`

```c
void iommu_group_release_dma_owner(struct iommu_group *group);
```

بتحرّر الـ ownership. الـ VFIO بيستدعيها عند إغلاق الـ device.

---

#### `iommu_group_dma_owner_claimed()`

```c
bool iommu_group_dma_owner_claimed(struct iommu_group *group);
```

بتتحقق إن الـ group عنده DMA owner مسجّل (مثلاً قبل ما تحاول تستخدمه مع kernel DMA-API).

---

### Group 14: Miscellaneous & Config

---

#### `device_iommu_capable()`

```c
bool device_iommu_capable(struct device *dev, enum iommu_cap cap);
```

بتسأل الـ IOMMU عن capability معيّنة من `enum iommu_cap`. بتستدعي `iommu_ops->capable()`. مثلاً `IOMMU_CAP_CACHE_COHERENCY` بيقول إن `IOMMU_CACHE` مدعوم.

---

#### `iommu_set_default_passthrough()` / `iommu_set_default_translated()`

```c
void iommu_set_default_passthrough(bool cmd_line);
void iommu_set_default_translated(bool cmd_line);
```

بيضبطوا الـ default IOMMU mode للـ DMA. الـ `cmd_line` flag بيحدد إن الإعداد جاي من command line أو من kernel config.

---

#### `iommu_set_pgtable_quirks()`

```c
int iommu_set_pgtable_quirks(struct iommu_domain *domain, unsigned long quirks);
```

بتفعّل quirks خاصة بالـ hardware في الـ I/O page table (مثلاً `IO_PGTABLE_QUIRK_ARM_MTK_EXT`). بتستدعي `domain->ops->set_pgtable_quirks()`.

---

#### `iommu_get_msi_cookie()`

```c
int iommu_get_msi_cookie(struct iommu_domain *domain, dma_addr_t base);
```

بتضبط الـ `domain->msi_cookie` للـ MSI doorbell window ابتداءً من `base`. مطلوبة لـ domains بتحتاج IOMMU-translated MSI addresses.

---

#### `iommu_map_sgtable()`

```c
static inline ssize_t iommu_map_sgtable(struct iommu_domain *domain,
                                         unsigned long iova, struct sg_table *sgt, int prot)
```

Wrapper مريحة بتمرّر `sgt->sgl` و `sgt->orig_nents` لـ `iommu_map_sg`. الـ `GFP_KERNEL` hardcoded.

---

#### `iommu_group_mutex_assert()`

```c
void iommu_group_mutex_assert(struct device *dev);
```

لما `CONFIG_LOCKDEP` مفعّل، بتتحقق إن الـ group mutex مقفول في السياق الحالي. بتُستخدم في الـ hot paths الـ internal للتأكد من الـ locking correctness.

---

#### `tegra_dev_iommu_get_stream_id()`

```c
static inline bool tegra_dev_iommu_get_stream_id(struct device *dev, u32 *stream_id)
```

Helper خاصة بـ Tegra SoCs — بتسترجع الـ stream ID من الـ fwspec. بشترط إن الـ device عنده بالظبط `num_ids == 1`.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

الـ IOMMU subsystem بيكشف معلومات مهمة جداً عبر `debugfs`.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# قائمة كل directories خاصة بـ IOMMU
ls /sys/kernel/debug/iommu/

# على ARM SMMU مثلاً
ls /sys/kernel/debug/arm-smmu/
ls /sys/kernel/debug/arm-smmu-v3/

# على Intel VT-d
ls /sys/kernel/debug/iommu/intel/
```

| Entry | المحتوى | كيف تقراه |
|---|---|---|
| `/sys/kernel/debug/iommu/intel/iommu_perf` | performance counters لكل IOMMU | `cat` مباشرة |
| `/sys/kernel/debug/iommu/intel/devices` | ربط كل device بـ domain | `cat` |
| `/sys/kernel/debug/iommu/intel/pt_show_domain` | عرض page tables لـ domain معين | `echo <domain_id> > ...` ثم `cat` |
| `/sys/kernel/debug/arm-smmu-v3/*/cmdq` | حالة command queue | `cat` |
| `/sys/kernel/debug/arm-smmu-v3/*/evtq` | event queue (الـ faults) | `cat` |
| `/sys/kernel/debug/iommu/groups` | كل iommu_group والـ devices فيه | `cat` |

```bash
# قراءة domain info لكل device
cat /sys/kernel/debug/iommu/intel/dmar_translation_struct

# عرض حالة TLB invalidation
cat /sys/kernel/debug/iommu/intel/invalidation_queue
```

---

#### 2. sysfs — المسارات المهمة

```bash
# كل iommu_group موجود في النظام
ls /sys/kernel/iommu_groups/

# الـ devices في كل group
ls /sys/kernel/iommu_groups/<N>/devices/

# الـ reserved regions لـ group معين
cat /sys/kernel/iommu_groups/<N>/reserved_regions

# type الـ domain الافتراضي للـ group
cat /sys/kernel/iommu_groups/<N>/type

# الـ IOMMU device المرتبط بـ device معين
ls /sys/devices/.../iommu/

# قدرات الـ IOMMU
cat /sys/bus/platform/devices/<iommu-dev>/iommu_group/type
```

```bash
# مثال عملي: فحص device PCI
DEVICE="0000:01:00.0"
GROUP=$(readlink -f /sys/bus/pci/devices/$DEVICE/iommu_group | xargs basename)
echo "IOMMU Group: $GROUP"
cat /sys/kernel/iommu_groups/$GROUP/reserved_regions
cat /sys/kernel/iommu_groups/$GROUP/type
ls /sys/kernel/iommu_groups/$GROUP/devices/
```

**الـ sysfs** بيعرض `iommu_device` لكل hardware instance:
```bash
# كل iommu_device مسجل في النظام
ls /sys/class/iommu/
cat /sys/class/iommu/<iommu>/intel-iommu/cap
cat /sys/class/iommu/<iommu>/intel-iommu/ecap
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# عرض كل events خاصة بـ IOMMU
ls /sys/kernel/debug/tracing/events/iommu/

# الـ events المتاحة عادةً:
# iommu_group_add_device
# iommu_group_remove_device
# iommu_attach_device
# iommu_detach_device
# iommu_map
# iommu_unmap
# io_page_fault
# map               (تحت iommu/)
# unmap
```

```bash
# تفعيل كل iommu events
echo 1 > /sys/kernel/debug/tracing/events/iommu/enable

# تفعيل event معين فقط
echo 1 > /sys/kernel/debug/tracing/events/iommu/io_page_fault/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/iommu_map/enable

# بدء الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تشغيل الـ workload المشكلة
<your-command>

# قراءة النتائج
cat /sys/kernel/debug/tracing/trace | grep -i iommu

# إيقاف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/iommu/enable
```

```bash
# استخدام trace-cmd للتسهيل
trace-cmd record -e 'iommu:*' <your-command>
trace-cmd report | head -100
```

**مثال على output من `io_page_fault`:**
```
kworker/0:1-123  [000] ....  123.456: io_page_fault:
    device=0000:01:00.0 iova=0x00007f1234560000 flags=0x0
```
يعني الـ device حاول يوصل لـ IOVA مش mapped — غالباً DMA buffer مش mapped صح.

---

#### 4. printk / dynamic debug

```bash
# تفعيل debug messages لـ iommu core
echo "module iommu +p" > /sys/kernel/debug/dynamic_debug/control

# لـ driver معين مثل arm-smmu
echo "file drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c +p" \
    > /sys/kernel/debug/dynamic_debug/control

# لـ intel-iommu
echo "file drivers/iommu/intel/iommu.c +pflmt" \
    > /sys/kernel/debug/dynamic_debug/control

# عرض الـ control entries الحالية
grep iommu /sys/kernel/debug/dynamic_debug/control

# تفعيل كل iommu modules بالكامل
echo "module intel_iommu +p" > /sys/kernel/debug/dynamic_debug/control
echo "module arm_smmu +p"    > /sys/kernel/debug/dynamic_debug/control
```

```bash
# متابعة الـ kernel log في real-time
dmesg -w | grep -i iommu
journalctl -k -f | grep -i iommu

# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_IOMMU_API` | تفعيل IOMMU core API |
| `CONFIG_IOMMU_DEBUG` | رسائل debug إضافية لـ core |
| `CONFIG_IOMMU_DEBUGFS` | كشف entries في debugfs |
| `CONFIG_IOMMU_FAULT_INJECTION` | fault injection للاختبار |
| `CONFIG_ARM_SMMU_V3_SVA` | debugging لـ SVA على SMMU v3 |
| `CONFIG_INTEL_IOMMU_DEBUG` | debug messages لـ VT-d |
| `CONFIG_INTEL_IOMMU_SVM` | Shared Virtual Memory على Intel |
| `CONFIG_DMA_API_DEBUG` | فحص صحة استخدام DMA API |
| `CONFIG_DMA_API_DEBUG_SG` | فحص scatter-gather mapping |
| `CONFIG_IOMMU_SVA` | Shared Virtual Addressing support |
| `CONFIG_IOMMU_IOPF` | IO Page Fault handling |
| `CONFIG_KCSAN` | race condition detection |
| `CONFIG_KASAN` | memory access error detection |
| `CONFIG_LOCKDEP` | lock ordering violations |

```bash
# فحص الـ options الحالية
grep -E "CONFIG_IOMMU|CONFIG_DMA_API_DEBUG|CONFIG_INTEL_IOMMU" /boot/config-$(uname -r)

# أو من /proc
zcat /proc/config.gz | grep -i iommu
```

---

#### 6. devlink وأدوات IOMMU-specific

```bash
# iommufd — userspace interface للـ IOMMU
# الـ device node
ls -la /dev/iommu

# أدوات من iommufd (VFIO/KVM use case)
# فحص VFIO groups
ls /dev/vfio/
cat /sys/bus/pci/devices/0000:01:00.0/iommu_group/type

# تحويل device لـ vfio-pci
echo "0000:01:00.0" > /sys/bus/pci/drivers/vfio-pci/bind

# أوامر iommu عبر /sys
# فرض IDENTITY domain على group
echo identity > /sys/kernel/iommu_groups/<N>/type

# lspci مع IOMMU info
lspci -vvv | grep -A 5 "IOMMU\|ATS\|PRI\|PASID"

# rdmsr/wrmsr لقراءة registers على Intel
rdmsr 0xCF8   # Intel VT-d capability
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `DMAR: DRHD: handling fault status reg` | Intel VT-d اكتشف fault | فحص device driver — غالباً DMA buffer مش mapped |
| `AMD-Vi: Event logged [IO_PAGE_FAULT]` | AMD IOMMU page fault | فحص `iova_to_phys` — mapping غلط أو مش موجود |
| `arm-smmu-v3: event 0x10` — Translation Fault | ARM SMMU translation error | الـ IOVA مش mapped في page table |
| `iommu: Failed to alloc domain` | فشل تخصيص domain | نقص في memory أو الـ driver مش عارف يعمل domain |
| `iommu group has no device` | device مش مضاف لـ group | مشكلة في `probe_device` أو bus scanning |
| `iommu_map: unknown page size` | pgsize مش supported | فحص `pgsize_bitmap` في الـ domain |
| `DMA-API: device driver failed to check returned DMA address` | driver مش بيتحقق من DMA mapping | إصلاح driver ليفحص `dma_mapping_error()` |
| `iommu fault handler not defined` | domain بدون fault handler | register handler بـ `iommu_set_fault_handler()` |
| `AMD-Vi: Completion-Wait loop timed out` | command queue timeout | firmware bug أو hardware failure |
| `DMAR: Intel(R) Virtualization Technology...disabled` | VT-d معطل في BIOS | تفعيله من BIOS/UEFI |
| `iommu: Failed to configure irq remapping` | فشل IRQ remapping | فحص `CONFIG_IRQ_REMAP` والـ ACPI tables |
| `iommu: Unsupported iommu page size bitmap` | pgsize غير متوافق | مشكلة في driver — فحص `pgsize_bitmap` |

---

#### 8. نقاط `dump_stack()` و`WARN_ON()` الاستراتيجية

أفضل أماكن تحط فيها debug assertions:

```c
/* في iommu_attach_device — للتحقق من صحة الحالة */
int iommu_attach_device(struct iommu_domain *domain, struct device *dev)
{
    WARN_ON(!domain);
    WARN_ON(!dev->iommu);        /* device لازم يكون probed */
    WARN_ON(!dev->iommu->iommu_dev); /* لازم يكون linked لـ IOMMU */
    ...
}

/* في iommu_map — عند mapping غير محاذي */
WARN_ON(!IS_ALIGNED(iova, pgsize));
WARN_ON(!IS_ALIGNED(paddr, pgsize));

/* في fault handler */
static int my_fault_handler(struct iommu_domain *domain,
                             struct device *dev,
                             unsigned long iova, int flags, void *token)
{
    WARN_ON(1); /* أي fault هنا unexpected */
    dump_stack();
    return -ENOSYS;
}

/* في iotlb_sync — لو الـ gather range غلط */
WARN_ON(gather->start > gather->end);

/* في domain_alloc — لو نوع الـ domain مش معروف */
WARN_ON(type != IOMMU_DOMAIN_DMA && type != IOMMU_DOMAIN_IDENTITY);
```

```bash
# تفعيل panic on WARN لإيقاف النظام فور حدوث WARN_ON
echo 1 > /proc/sys/kernel/panic_on_warn

# أو من kernel cmdline
# kernel cmdline: oops=panic panic_on_warn=1 iommu=pt iommu.passthrough=1
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

```bash
# فحص الـ IOMMU hardware capabilities المُبلَّغة للـ kernel
# Intel VT-d
cat /sys/class/iommu/dmar*/intel-iommu/cap
cat /sys/class/iommu/dmar*/intel-iommu/ecap

# ARM SMMU
cat /sys/class/iommu/*/type

# فحص ACPI DMAR table (Intel)
acpidump -n DMAR | acpixtract -a - | iasl -d -
cat /proc/iomem | grep DMAR

# ACPI IVRS table (AMD)
acpidump -n IVRS

# مقارنة عدد الـ IOMMU devices في الـ kernel vs hardware
dmesg | grep -E "DMAR|AMD-Vi|SMMU" | head -20
```

**الـ kernel يقرأ `iommu_device.max_pasids`** — لازم يتوافق مع hardware register:
```bash
# فحص PASID support
lspci -vvv -s 0000:01:00.0 | grep -i pasid
```

---

#### 2. Register Dump Techniques

```bash
# قراءة register بـ devmem2
# مثال: Intel VT-d DMAR base address من ACPI
BASE=$(acpidump -n DMAR | grep "Register Base Address" | awk '{print $NF}')
devmem2 $((BASE + 0x00)) w   # Version register
devmem2 $((BASE + 0x08)) q   # Capability register (CAP)
devmem2 $((BASE + 0x10)) q   # Extended Capability (ECAP)
devmem2 $((BASE + 0x18)) w   # Global Command (GCMD)
devmem2 $((BASE + 0x1C)) w   # Global Status (GSTS)
devmem2 $((BASE + 0x20)) q   # Root Table Address (RTADDR)

# ARM SMMU v3 registers
BASE=0x09050000   # مثال لـ base address من DT
devmem2 $((BASE + 0x000)) w  # IDR0
devmem2 $((BASE + 0x004)) w  # IDR1
devmem2 $((BASE + 0x100)) w  # CR0
devmem2 $((BASE + 0x104)) w  # CR0ACK
devmem2 $((BASE + 0x108)) w  # CR1
devmem2 $((BASE + 0x10C)) w  # CR2
devmem2 $((BASE + 0x110)) q  # STRTAB_BASE
devmem2 $((BASE + 0x118)) w  # STRTAB_BASE_CFG

# AMD IOMMU — القراءة عبر MMIO
# Base address من IVRS table
devmem2 $((AMD_BASE + 0x0000)) w  # Device Table Base Address Lo
devmem2 $((AMD_BASE + 0x0018)) w  # Control Register
devmem2 $((AMD_BASE + 0x2020)) w  # Status Register
```

```bash
# استخدام /dev/mem مع mmap (في driver)
# أو io utility
io -4 -r 0xFED90000   # Intel VT-d default MMIO base
```

**Intel VT-d CAP register bit fields:**
```
CAP[3:0]   = Number of domains supported
CAP[8]     = Advanced Fault Logging support
CAP[39]    = Dirty bit tracking support (IOMMU_CAP_DIRTY_TRACKING)
ECAP[6]    = Pass Through support
ECAP[8]    = Snoop Control
ECAP[10]   = Device IOTLB support (ATS)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ PCIe IOMMU debugging:**
- راقب **ATS (Address Translation Service)** transactions على PCIe bus
- **PRI (Page Request Interface)** messages — تظهر لما device يعمل page fault ويطلب mapping
- راقب **TLP headers**: الـ `Requester ID` هو الـ BDF (Bus:Device.Function)

```
PCIe Sniffer setup:
  Ch1: REFCLK (100MHz)
  Ch2: TX differential pair (device → root complex)
  Ch3: RX differential pair (root complex → device)

Trigger on: Non-Posted TLP with address > 4GB (لاكتشاف 64-bit DMA issues)
```

**لـ ARM platform:**
- راقب AXI bus signals: `AWPROT`, `ARPROT` — بيحدد privilege level
- الـ `ARCACHE`/`AWCACHE` bits بتتحكم في cache coherency (IOMMU_CACHE flag)
- راقب الـ `SMMU_GERROR` output GPIO لو موجود

---

#### 4. Hardware Issues الشائعة وأنماط الـ Kernel Log

| مشكلة Hardware | نمط الـ kernel log | التشخيص |
|---|---|---|
| IOMMU disabled in BIOS | `DMAR: IOMMU disabled` أو غياب أي رسالة IOMMU | فعّل VT-d/AMD-Vi من BIOS |
| Page table walk failure | `io_page_fault: iova=X device=Y` | الـ mapping مش موجود — driver bug |
| ATS not supported | `iommu: ATS not supported` | تأكد من PCIe ATS في device capability |
| TLB invalidation timeout | `AMD-Vi: completion wait loop timed out` | hardware bug أو BIOS firmware issue |
| Wrong PASID | `iommu: PASID X not attached to device` | race condition في SVA setup |
| Reserved region conflict | `iommu: Device attempts to map reserved region` | فحص `iommu_resv_region` list |
| SMMU stage-2 fault | `arm-smmu: Unhandled context fault` | nested domain misconfiguration |
| MSI interrupt not working after IOMMU enable | silence بعد `request_irq` | MSI doorbell address لازم يتمر عبر IOMMU أو يكون في `IOMMU_RESV_MSI` |

---

#### 5. Device Tree Debugging

```bash
# عرض الـ DT node الخاص بـ IOMMU
dtc -I fs /sys/firmware/devicetree/base -O dts | grep -A 20 "iommu"

# فحص iommus property في device node
dtc -I fs /sys/firmware/devicetree/base -O dts | grep -B 5 -A 5 "iommus ="

# فحص الـ IOMMU controller node
cat /sys/firmware/devicetree/base/smmu/compatible
cat /sys/firmware/devicetree/base/smmu/reg        # base address
xxd /sys/firmware/devicetree/base/smmu/reg        # hex dump

# فحص stream IDs
xxd /sys/firmware/devicetree/base/pcie/iommus

# مقارنة الـ DT base address مع devmem2
SMMU_BASE=$(dtc -I fs /sys/firmware/devicetree/base -O dts | \
    awk '/smmu/{found=1} found && /reg =/{print; exit}' | \
    grep -oP '0x[0-9a-f]+' | head -1)
echo "SMMU Base: $SMMU_BASE"
devmem2 $SMMU_BASE w   # IDR0
```

**أشياء لازم تتحقق منها في الـ DT:**

```dts
/* صح ✓ */
smmu: iommu@9050000 {
    compatible = "arm,smmu-v3";
    reg = <0x0 0x09050000 0x0 0x20000>;
    interrupts = <GIC_SPI 74 IRQ_TYPE_EDGE_RISING>,
                 <GIC_SPI 75 IRQ_TYPE_EDGE_RISING>;
    #iommu-cells = <1>;
};

pcie: pcie@10000000 {
    iommus = <&smmu 0x100>;   /* stream ID */
    ...
};

/* غلط ✗ — base address خطأ أو stream ID مش مطابق للـ hardware */
```

```bash
# التحقق من أن kernel قرأ الـ DT صح
dmesg | grep -i "smmu\|iommu" | head -30

# لو الـ SMMU مش probed:
# 1. تحقق من compatible string
# 2. تحقق من clocks وpower domains
# 3. فحص interrupts صح
dmesg | grep -E "probe|defer" | grep smmu
```

---

### Practical Commands

#### Ready-to-copy Shell Commands

**1. فحص شامل سريع للـ IOMMU setup:**
```bash
#!/bin/bash
echo "=== IOMMU Status ==="
dmesg | grep -E "IOMMU|iommu|DMAR|SMMU|AMD-Vi" | head -30

echo ""
echo "=== IOMMU Groups ==="
for g in /sys/kernel/iommu_groups/*/; do
    echo -n "Group $(basename $g): "
    ls "$g/devices/" 2>/dev/null | tr '\n' ' '
    echo " | type: $(cat $g/type 2>/dev/null)"
done

echo ""
echo "=== Domain Types ==="
for g in /sys/kernel/iommu_groups/*/; do
    echo "Group $(basename $g): $(cat $g/type 2>/dev/null)"
done
```

**2. تتبع DMA mapping لـ device معين:**
```bash
# تفعيل DMA API debug
echo 1 > /sys/kernel/debug/dma-api/disabled  # disable لإعادة enable

# تفعيل iommu map/unmap tracing
echo 1 > /sys/kernel/debug/tracing/events/iommu/iommu_map/enable
echo 1 > /sys/kernel/debug/tracing/events/iommu/iommu_unmap/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe &
TRACE_PID=$!
# شغّل workload
sleep 5
kill $TRACE_PID
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**3. تفعيل io_page_fault monitoring:**
```bash
echo 1 > /sys/kernel/debug/tracing/events/iommu/io_page_fault/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
dmesg -w &
cat /sys/kernel/debug/tracing/trace_pipe | tee /tmp/iommu_faults.log
```

**4. فحص reserved regions:**
```bash
for g in /sys/kernel/iommu_groups/*/; do
    RESV="$g/reserved_regions"
    if [ -s "$RESV" ]; then
        echo "=== Group $(basename $g) ==="
        cat "$RESV"
    fi
done
```

**5. iova_to_phys للـ debugging:**
```bash
# من kernel space — تضيفه في driver مؤقتاً
phys_addr_t phys = iommu_iova_to_phys(domain, suspect_iova);
pr_debug("IOVA 0x%lx -> phys 0x%pa\n", suspect_iova, &phys);
# لو phys == 0 → الـ mapping مش موجود
```

**6. فحص TLB flush behavior:**
```bash
# تفعيل IOMMU_CAP_DEFERRED_FLUSH debugging
echo "file drivers/iommu/dma-iommu.c +p" \
    > /sys/kernel/debug/dynamic_debug/control

# مراقبة iotlb_sync calls
echo 1 > /sys/kernel/debug/tracing/events/iommu/enable
grep -i "iotlb\|tlb" /sys/kernel/debug/tracing/trace
```

**7. SVA (Shared Virtual Addressing) debugging:**
```bash
# فحص PASID allocation
cat /sys/kernel/debug/iommu/intel/pasid_info 2>/dev/null

# مراقبة SVA bind/unbind
echo 1 > /sys/kernel/debug/tracing/events/iommu/iommu_group_add_device/enable

# فحص mm_struct binding
cat /proc/<pid>/smaps | grep -i iommu
```

**8. فحص dirty tracking:**
```bash
# لو الـ IOMMU_CAP_DIRTY_TRACKING مدعوم
dmesg | grep -i "dirty tracking"

# من Intel VT-d — فحص CAP bit 39
CAP=$(cat /sys/class/iommu/dmar0/intel-iommu/cap)
echo "CAP: $CAP"
# bit 39 = dirty page tracking
```

---

#### تفسير الـ Output

**مثال output من `/sys/kernel/iommu_groups/`:**
```
/sys/kernel/iommu_groups/1/
├── devices/
│   └── 0000:01:00.0 -> ../../../../devices/pci0000:00/0000:00:1c.0/0000:01:00.0
├── reserved_regions
│   0x00000000fee00000 0x00000000feefffff msi
│   0x00000000fed90000 0x00000000fed90fff direct
└── type
    DMA
```
- `type = DMA` يعني الـ device تحت IOMMU translation
- `type = identity` يعني passthrough (bypass)
- `type = blocked` يعني كل الـ DMA blocked

**مثال output من `io_page_fault` event:**
```
 kworker/u4:2-89   [001] d... 1234.567890: io_page_fault:
     device=0000:02:00.0 iova=0x0000001000000000 flags=IOMMU_FAULT_READ
```
التفسير:
- `device=0000:02:00.0` → الـ BDF للـ device المشكوك فيه
- `iova=0x1000000000` → الـ virtual address اللي الـ device حاول يوصله
- `flags=READ` → كانت read operation

الخطوات بعد كده:
1. `iommu_iova_to_phys(domain, 0x1000000000)` → لو رجع 0، المشكلة في الـ mapping
2. فحص الـ driver إنه بيعمل `dma_map_*` قبل ما يبعت الـ descriptor للـ device
3. فحص الـ IOMMU_READ/WRITE flags في `iommu_map()`
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: DMA fault على بورد RK3562 في gateway صناعي

#### العنوان
**IOMMU page fault** يعطّل الـ Ethernet DMA في industrial gateway

#### السياق
بورد مبني على **RK3562** مستخدم كـ industrial gateway فيه Ethernet controller بيشتغل على DMA. الـ IOMMU مفعّل في الكيرنل. البورد بيشتغل كويس لساعات وبعدين يموت الـ network interface فجأة مع رسالة kernel panic.

#### المشكلة
الـ Ethernet driver بيعمل `iommu_map()` لبعض الـ scatter-gather buffers بس بيفضل يستخدم الـ IOVA القديمة بعد `iommu_unmap()` — classic **use-after-unmap**. الـ IOMMU hardware بيرفع page fault لأن الـ translation اتشالت من الـ page table.

#### التحليل

الكيرنل بيستدعي `report_iommu_fault()` المعرّفة في الهيدر:

```c
extern int report_iommu_fault(struct iommu_domain *domain, struct device *dev,
                              unsigned long iova, int flags);
```

الـ `flags` بيكون `IOMMU_FAULT_READ` أو `IOMMU_FAULT_WRITE` (القيم `0x0` / `0x1`). الـ handler المسجّل عبر `iommu_set_fault_handler()` بيتنفّذ:

```c
extern void iommu_set_fault_handler(struct iommu_domain *domain,
                        iommu_fault_handler_t handler, void *token);
```

النوع:
```c
typedef int (*iommu_fault_handler_t)(struct iommu_domain *,
                struct device *, unsigned long, int, void *);
```

الـ `iommu_domain` اللي بيشيله الـ Ethernet domain نوعه `IOMMU_DOMAIN_DMA` — يعني `type = __IOMMU_DOMAIN_PAGING | __IOMMU_DOMAIN_DMA_API`. الـ `iommu_is_dma_domain()` بترجع `true` لأن `type & __IOMMU_DOMAIN_DMA_API != 0`.

مسار الـ unmap:
1. الـ driver يستدعي `iommu_unmap_fast()` اللي بتملأ `struct iommu_iotlb_gather`.
2. الـ TLB invalidation بياخد وقت — الـ gather object:

```c
struct iommu_iotlb_gather {
    unsigned long   start;   /* IOVA range start */
    unsigned long   end;     /* IOVA range end   */
    size_t          pgsize;
    struct iommu_pages_list freelist;
    bool            queued;  /* flush is deferred */
};
```

لو `queued = true` يعني الـ flush مش اتعمل لسه، والـ driver يستخدم الـ IOVA تاني قبل `iommu_iotlb_sync()`.

#### الحل

```bash
# تأكيد المشكلة من dmesg
dmesg | grep -i "iommu\|fault\|dma"

# فحص الـ domain في sysfs
cat /sys/kernel/debug/iommu/devices
```

في الكود، التأكد إن الـ sync بيحصل قبل إعادة استخدام الـ buffer:

```c
/* After iommu_unmap_fast(), must sync before reuse */
iommu_iotlb_sync(domain, &gather);
/* Now safe to remap */
iommu_map(domain, iova, paddr, size, IOMMU_READ | IOMMU_WRITE, GFP_KERNEL);
```

أو الاعتماد على `iommu_unmap()` العادي اللي بيعمل الـ sync داخلياً بدل `iommu_unmap_fast()`.

#### الدرس المستفاد
**الـ `iommu_unmap_fast()` + `iommu_iotlb_gather` مش بتضمن إن الـ TLB اتفلش فوراً.** لازم استدعاء `iommu_iotlb_sync()` صريح قبل أي إعادة استخدام للـ IOVA range.

---

### السيناريو 2: STM32MP1 IoT sensor — IOMMU مش موجود والـ driver بيـ crash

#### العنوان
الـ driver بيفترض وجود IOMMU على **STM32MP1** وبيـ crash عند الـ probe

#### السياق
مهندس بيعمل port لـ sensor driver مكتوب أصلاً لـ i.MX8 على بورد **STM32MP1** مستخدم كـ IoT sensor node. الـ STM32MP1 مش فيه IOMMU. الـ driver بيتعمله build مع `CONFIG_IOMMU_API=n`.

#### المشكلة
الكود في الـ driver بيستدعي `iommu_attach_device()` مباشرةً بدون التحقق من وجود IOMMU. عند الـ build بدون `CONFIG_IOMMU_API`، الـ inline stub بترجع `-ENODEV` بس الـ driver مش بيتعامل مع الـ error.

#### التحليل

في الهيدر، عند `CONFIG_IOMMU_API` غير مفعّل:

```c
static inline int iommu_attach_device(struct iommu_domain *domain,
                                      struct device *dev)
{
    return -ENODEV;  /* stub — no IOMMU hardware */
}
```

وبالمثل:
```c
static inline bool device_iommu_capable(struct device *dev, enum iommu_cap cap)
{
    return false;
}
```

الـ driver الأصلي كان بيفترض إن `device_iommu_capable(dev, IOMMU_CAP_CACHE_COHERENCY)` بترجع `true` وبيعمل setup خاص. على STM32MP1 بترجع `false` وبيكمل الكود بـ path مش متوقع.

أخطر من كده، لو الـ driver بيمسك الـ `struct iommu_fwspec` عبر:

```c
static inline struct iommu_fwspec *dev_iommu_fwspec_get(struct device *dev)
{
    if (dev->iommu)
        return dev->iommu->fwspec;
    else
        return NULL;  /* on STM32MP1: dev->iommu == NULL */
}
```

والكود بيعمل deref على الـ pointer من غير فحص → NULL pointer dereference.

#### الحل

```c
/* Safe pattern for drivers that optionally use IOMMU */
static int my_driver_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    /* Check IOMMU presence before any IOMMU call */
    if (device_iommu_capable(dev, IOMMU_CAP_CACHE_COHERENCY)) {
        /* IOMMU path — i.MX8, RK3562, etc. */
        priv->domain = iommu_paging_domain_alloc(dev);
        if (IS_ERR(priv->domain))
            return PTR_ERR(priv->domain);
    } else {
        /* No-IOMMU path — STM32MP1, bare-metal boards */
        priv->domain = NULL;
    }
    return 0;
}
```

#### الدرس المستفاد
**دايماً استخدم `device_iommu_capable()` كـ guard قبل أي IOMMU operation.** الـ stubs في الهيدر بترجع قيم safe بس مش بتمنع الـ NULL dereference لو الـ driver مش بيفحص.

---

### السيناريو 3: i.MX8 Android TV Box — SVA domain والـ GPU driver

#### العنوان
**Shared Virtual Addressing (SVA)** بيسبّب memory corruption في الـ GPU على **i.MX8**

#### السياق
Android TV box مبني على **i.MX8MP** فيه GPU بيستخدم SVA علشان processes المستخدم تعمل DMA مباشرة من الـ user virtual address space. الـ `IOMMU_DOMAIN_SVA` مفعّل.

#### المشكلة
عند انتهاء الـ process، الـ GPU driver أحياناً بيفضل يستخدم الـ SVA domain بعد إن الـ `mm_struct` اتحرر — لأن الـ reference counting على `iommu_mm_data` غلط.

#### التحليل

الـ SVA domain معرّف في `struct iommu_domain` بالـ union:

```c
struct iommu_domain {
    unsigned type;  /* IOMMU_DOMAIN_SVA = __IOMMU_DOMAIN_SVA */
    /* ... */
    union {
        /* IOMMU_DOMAIN_SVA */
        struct {
            struct mm_struct *mm;  /* the process mm */
            int users;             /* reference count */
            struct list_head next; /* in mm->iommu_mm->sva_domains */
        };
    };
};
```

والـ `struct iommu_mm_data`:

```c
struct iommu_mm_data {
    u32          pasid;
    struct mm_struct *mm;
    struct list_head sva_domains;  /* list of SVA domains for this mm */
    struct list_head mm_list_elm;
};
```

الـ attach handle:
```c
struct iommu_sva {
    struct iommu_attach_handle handle; /* contains domain pointer */
    struct device   *dev;
    refcount_t       users;  /* must reach 0 before domain free */
};
```

المشكلة: الـ GPU driver بيعمل `iommu_sva_unbind_device()` بس قبل ما `refcount_t users` يوصل صفر — يعني الـ domain لسه مربوط والـ mm اتحرر → use-after-free على الـ `mm_struct`.

#### الحل

```bash
# تفعيل KASAN للكشف عن use-after-free
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y

# متابعة الـ SVA domains في runtime
cat /sys/kernel/debug/iommu/groups/*/type
```

في الكود، التأكد من إن الـ unbind بيستنى الـ users:

```c
/* Must call iommu_sva_unbind_device() for EACH bind call */
while (refcount_read(&sva->users) > 0)
    iommu_sva_unbind_device(sva);
/* Only then the domain is safe to free */
```

و في الـ driver exit:
```c
/* Verify domain type before access */
if (domain->type == IOMMU_DOMAIN_SVA && domain->mm) {
    mmput(domain->mm); /* release mm reference */
}
```

#### الدرس المستفاد
**الـ `iommu_domain` من نوع `IOMMU_DOMAIN_SVA` بيحمل pointer على الـ `mm_struct` — لازم الـ `users` counter يوصل صفر قبل أي free.** الـ refcount في `struct iommu_sva` مش بديل عن الـ symmetric bind/unbind calls.

---

### السيناريو 4: AM62x — Reserved Regions وخبطة في الـ MSI doorbell

#### العنوان
الـ PCIe MSI على **AM62x** بيعطي interrupts باردة — الـ IOMMU بيبلوك الـ MSI doorbell

#### السياق
بورد industrial مبني على **TI AM62x** فيه PCIe endpoint (NVMe SSD). الـ IOMMU مفعّل. الـ NVMe بيعمل enumerate كويس بس الـ MSI interrupts مش بتوصل — الـ NVMe commands بتـ timeout.

#### المشكلة
الـ IOMMU بيمنع الـ DMA write على عنوان الـ MSI doorbell لأن المنطقة دي مش mapped في الـ domain. الـ MSI doorbell address محتاج يكون في منطقة `IOMMU_RESV_MSI` أو `IOMMU_RESV_SW_MSI`.

#### التحليل

الـ reserved regions في الهيدر:

```c
enum iommu_resv_type {
    IOMMU_RESV_DIRECT,           /* 1:1 mapping — must always be mapped */
    IOMMU_RESV_DIRECT_RELAXABLE, /* 1:1 but relaxable (USB, Graphics) */
    IOMMU_RESV_RESERVED,         /* never map — off-limits */
    IOMMU_RESV_MSI,              /* HW MSI region — untranslated */
    IOMMU_RESV_SW_MSI,           /* SW-managed MSI translation window */
};
```

الـ `struct iommu_resv_region`:

```c
struct iommu_resv_region {
    struct list_head list;
    phys_addr_t      start;   /* physical address of the region */
    size_t           length;
    int              prot;    /* IOMMU_READ | IOMMU_WRITE | ... */
    enum iommu_resv_type type;
    void (*free)(struct device *dev, struct iommu_resv_region *region);
};
```

الـ IOMMU driver على AM62x لازم يستدعي `get_resv_regions` في `struct iommu_ops` ويرجع الـ MSI doorbell range:

```c
void (*get_resv_regions)(struct device *dev, struct list_head *list);
```

لو الـ driver ما رجعّش الـ MSI region كـ `IOMMU_RESV_MSI`، الـ DMA API مش هتعمله mapping وهيتبلوك.

#### الحل

```bash
# فحص الـ reserved regions للـ PCIe device
cat /sys/kernel/debug/iommu/devices/*/resv_regions

# أو عبر الـ group
iommu_group=$(cat /sys/bus/pci/devices/0000:01:00.0/iommu_group)
cat /sys/kernel/iommu_groups/${iommu_group}/reserved_regions
```

في الـ DT لـ AM62x:

```dts
&pcie0 {
    iommu-map = <0x0 &main_iommu 0x0 0x1>;
    /* MSI doorbell must be in the IOMMU bypass window */
    msi-parent = <&main_gic_its>;
};
```

وفي الـ IOMMU driver:

```c
static void am62_iommu_get_resv_regions(struct device *dev,
                                         struct list_head *list)
{
    struct iommu_resv_region *region;

    /* Add MSI doorbell as untranslated region */
    region = iommu_alloc_resv_region(MSI_DOORBELL_BASE,
                                      MSI_DOORBELL_SIZE,
                                      IOMMU_WRITE,
                                      IOMMU_RESV_MSI,
                                      GFP_KERNEL);
    if (region)
        list_add_tail(&region->list, list);
}
```

#### الدرس المستفاد
**الـ MSI doorbell address لازم تتعمل register كـ `IOMMU_RESV_MSI` في `get_resv_regions`.** لو ما اتعملتش، الـ DMA API هتبلوك الـ write وهيبان كـ interrupt timeout مش كـ IOMMU fault واضح.

---

### السيناريو 5: Allwinner H616 — Custom Board Bring-up وغياب `of_xlate`

#### العنوان
الـ USB controller على **Allwinner H616** مش بياخد IOMMU group صح عند الـ bring-up

#### السياق
مهندس بيعمل bring-up لبورد custom مبني على **Allwinner H616** (شائع في TV boxes). الـ USB 3.0 controller محتاج IOMMU isolation بس الـ dmesg بيظهر إنه بياخد default identity domain مش الـ DMA domain المتوقع.

#### المشكلة
الـ `iommu-map` في الـ DT موجود بس الـ IOMMU driver ما implementش الـ `of_xlate` callback في `struct iommu_ops` — فالـ kernel مش قادر يربط الـ USB device بالـ IOMMU instance الصح.

#### التحليل

الـ `of_xlate` في `struct iommu_ops`:

```c
struct iommu_ops {
    /* ... */
    int (*of_xlate)(struct device *dev, const struct of_phandle_args *args);
    /* ... */
};
```

الـ callback ده بياخد الـ master IDs من الـ DT (`iommu-map` property) ويضيفها للـ IOMMU group. بدونه، الـ `iommu_fwspec` بتفضل فاضية:

```c
struct iommu_fwspec {
    struct fwnode_handle *iommu_fwnode; /* points to IOMMU node in DT */
    u32          flags;
    unsigned int num_ids; /* = 0 if of_xlate not called */
    u32          ids[];   /* stream IDs / master IDs */
};
```

الـ `dev_iommu_fwspec_get()` هترجع struct بـ `num_ids = 0`، فالـ `device_group` callback مش هيعرف يحط الـ USB device في الـ group الصح وهيرجعله `singleton_group` أو `NULL`.

الـ probe sequence بالترتيب:

```
iommu_probe_device(dev)
  → iommu_ops->probe_device(dev)     /* allocates dev->iommu */
  → of_iommu_configure(dev, ...)     /* calls of_xlate for each master ID */
  → iommu_ops->probe_finalize(dev)   /* final setup after group attach */
```

لو `of_xlate` مش implemented، `of_iommu_configure` بيفشل صامتاً.

#### الحل

```bash
# فحص الـ fwspec للـ USB device
cat /sys/kernel/debug/iommu/devices/

# فحص الـ iommu-map في DT
dtc -I fs /proc/device-tree | grep -A5 "usb"

# التأكد من الـ group assignment
ls /sys/bus/platform/devices/usb3/iommu_group
```

في الـ IOMMU driver لـ H616:

```c
static int sun50i_iommu_of_xlate(struct device *dev,
                                  const struct of_phandle_args *args)
{
    /* args->args[0] = master ID from iommu-map in DT */
    return iommu_fwspec_add_ids(dev, &args->args[0], 1);
}

static const struct iommu_ops sun50i_iommu_ops = {
    /* ... */
    .of_xlate     = sun50i_iommu_of_xlate,
    .device_group = generic_device_group,
    /* ... */
};
```

والـ DT الصح:

```dts
&usb3 {
    /* sid = stream ID for USB controller */
    iommu-map = <0 &iommu 0x500 1>;
    iommu-map-mask = <0xfffff>;
};
```

#### الدرس المستفاد
**الـ `of_xlate` callback هو الجسر بين الـ Device Tree والـ IOMMU group.** غيابه بيخلي `iommu_fwspec->num_ids = 0` والـ device بياخد identity mapping بدل الـ isolated DMA domain — وده خطر أمني كامل في بيئات multi-tenant أو virtualization.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**LWN.net** هو المرجع الأول لمتابعة تطور الـ IOMMU subsystem في Linux kernel. الـ articles دي غطّت التطور من الأساسيات للـ features المتقدمة:

| المقال | الوصف |
|--------|-------|
| [IOMMU, what is it?](https://lwn.net/Articles/288430/) | مقدمة أساسية للـ IOMMU — شرح المفهوم وليه محتاجينه في الـ DMA |
| [Intel IOMMU Pass Through Support](https://lwn.net/Articles/329174/) | شرح الـ `intel_iommu=pt` وكيف بيشتغل الـ pass-through mode |
| [mm: iommu: An API to unify IOMMU, CPU and device memory management](https://lwn.net/Articles/394034/) | محاولة توحيد الـ API بين IOMMU وـ CPU memory management |
| [iommu: Per-group default domain type](https://lwn.net/Articles/808400/) | تفاصيل الـ IOMMU groups وكيف بيتم تخصيص الـ default domain |
| [Shared Virtual Addressing for the IOMMU](https://lwn.net/Articles/747230/) | الـ SVA (Shared Virtual Addressing) — مشاركة الـ process address space مع الـ devices |
| [IOMMU and VT-d driver support for Shared Virtual Address (SVA)](https://lwn.net/Articles/754331/) | تفاصيل تطبيق SVA على Intel VT-d |
| [iommu: Shared Virtual Addressing for SMMUv3 (PT sharing part)](https://lwn.net/Articles/831868/) | تطبيق SVA على ARM SMMUv3 |
| [vfio: expose virtual Shared Virtual Addressing to VMs](https://lwn.net/Articles/827208/) | تمرير SVA للـ VMs عبر VFIO |
| [iommu/amd: Introduce hardware info reporting and nested translation support](https://lwn.net/Articles/954730/) | الـ nested translation في AMD IOMMU وـ HW-vIOMMU |
| [Linux RISC-V IOMMU Support](https://lwn.net/Articles/972035/) | إضافة IOMMU support للـ RISC-V architecture |
| [Hyper-V: Add para-virtualized IOMMU support for Linux guests](https://lwn.net/Articles/1049711/) | دعم الـ para-virtualized IOMMU في Hyper-V |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في `Documentation/` داخل الـ kernel source tree:

```
Documentation/core-api/dma-api.rst          ← الـ DMA API الكامل
Documentation/core-api/dma-api-howto.rst    ← دليل عملي للـ DMA mapping
Documentation/driver-api/iommu.rst          ← IOMMU subsystem API
Documentation/driver-api/iommufd.rst        ← IOMMUFD — الـ userspace API الجديد
Documentation/driver-api/vfio.rst           ← VFIO وعلاقته بالـ IOMMU
Documentation/arch/x86/iommu.rst            ← x86 IOMMU (Intel VT-d / AMD-Vi)
Documentation/arch/x86/sva.rst              ← Shared Virtual Addressing مع ENQCMD
```

الروابط على kernel.org:
- [x86 IOMMU Support](https://docs.kernel.org/arch/x86/iommu.html)
- [Shared Virtual Addressing (SVA) with ENQCMD](https://docs.kernel.org/arch/x86/sva.html)
- [IOMMU subsystem — kernel docs](https://docs.kernel.org/driver-api/iommu.html)

---

### الـ Header File المرجعي

الملف اللي بندرسه:

```
include/linux/iommu.h       ← التعريفات الأساسية: iommu_ops, iommu_domain, iommu_group
drivers/iommu/iommu.c       ← التطبيق الأساسي للـ subsystem
drivers/iommu/intel/iommu.c ← Intel VT-d driver
drivers/iommu/amd/iommu.c   ← AMD-Vi driver
drivers/iommu/arm/arm-smmu-v3/ ← ARM SMMUv3 driver
drivers/iommu/iommufd/       ← IOMMUFD — الـ userspace interface الجديد
```

---

### Kernel Commits المهمة

| الـ Commit | الوصف |
|------------|-------|
| [`2007` — AMD IOMMU initial](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/iommu) | Joerg Roedel كتب أول IOMMU framework لـ AMD |
| الـ IOMMUFD framework | Jason Gunthorpe أضاف `/dev/iommu` userspace interface في kernel 6.2 |
| الـ SVA API | Jean-Philippe Brucker أضاف `iommu_sva_bind_device()` API |
| الـ dirty tracking | إضافة `iommu_dirty_ops` لدعم VM live migration في kernel 6.7 |

للبحث في الـ git log:
```bash
git log --oneline drivers/iommu/
git log --oneline include/linux/iommu.h
```

---

### Mailing List

الـ IOMMU subsystem بيتناقش على:

- **القائمة الرسمية**: `iommu@lists.linux.dev`
- **أرشيف lore.kernel.org**: [https://lore.kernel.org/iommu/](https://lore.kernel.org/iommu/)
- **LKML archive**: [https://lkml.org/](https://lkml.org/) — ابحث عن `[PATCH] iommu:`

---

### KernelNewbies — تغييرات IOMMU بالـ Kernel Versions

| الـ Kernel Version | أهم تغيير في الـ IOMMU |
|--------------------|----------------------|
| [Linux 5.3](https://kernelnewbies.org/Linux_5.3) | إضافة `virtio-iommu` driver — para-virtualized IOMMU |
| [Linux 5.16](https://kernelnewbies.org/Linux_5.16) | تحسينات الـ IOMMU groups وـ default domains |
| [Linux 6.2](https://kernelnewbies.org/Linux_6.2) | إضافة **IOMMUFD** — الـ userspace API الجديد (`/dev/iommu`) |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | تحسينات nested translation |
| [Linux 6.7](https://kernelnewbies.org/Linux_6.7) | إضافة dirty tracking لـ IOMMU (مهم لـ VM live migration) |
| [Linux 6.8](https://kernelnewbies.org/Linux_6.8) | تحسينات AMD/ARM IOMMU nested translation |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | IOMMUFD IO page fault delivery لـ userspace |

---

### eLinux.org

الـ eLinux.org مش بيغطي الـ IOMMU بشكل مستقل، لكن في صفحات ذات صلة:

- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — روابط عامة للـ kernel development
- [Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) — presentations بتغطي DMA وـ IOMMU integration
- [BeagleBoard/DSP Clarification](https://elinux.org/BeagleBoard/DSP_Clarification) — مثال عملي على IOMMU في DSP/IPC على الـ embedded platforms

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 15**: Memory Mapping and DMA
  - الـ `dma_map_single()`, `dma_map_sg()` وكيف بيتعاملوا مع الـ IOMMU internally
  - متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 12**: Memory Management — فهم الـ virtual memory اللي الـ IOMMU بيعتمد عليه
- **الفصل 13**: The Virtual Filesystem — context مفيد لفهم الـ device model

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: Kernel Debugging Techniques
- **الفصل 8**: Device Driver Basics — الـ DMA وعلاقته بالـ IOMMU في الـ embedded systems

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 3**: Memory Management — تفاصيل عميقة في الـ page tables اللي الـ IOMMU بيستخدم نفس مبدأها

---

### مصطلحات البحث

للبحث عن معلومات إضافية استخدم الـ search terms دي:

```
# بحث عام
linux kernel iommu subsystem
iommu_ops iommu_domain linux
linux dma-iommu api

# features محددة
linux IOMMUFD /dev/iommu userspace
linux SVA shared virtual addressing PASID
linux iommu nested translation stage1 stage2
linux iommu dirty tracking live migration
linux iommu groups vfio passthrough

# architectures محددة
intel VT-d linux driver
AMD-Vi IOMMU linux kernel
ARM SMMU SMMUv3 linux
RISC-V IOMMU linux kernel

# debugging
linux iommu=pt passthrough
linux iommu=on force enable
linux iommu fault handling
```

---

### ملخص سريع للمراجع الأهم

```
┌─────────────────────────────────────────────────────┐
│  أفضل نقطة بداية للفهم                              │
│  → lwn.net/Articles/288430  (IOMMU, what is it?)    │
│                                                     │
│  أفضل reference للـ API                             │
│  → docs.kernel.org/driver-api/iommu.html            │
│                                                     │
│  أفضل reference للـ userspace interface              │
│  → docs.kernel.org/driver-api/iommufd.html          │
│                                                     │
│  لمتابعة التطورات الجديدة                            │
│  → lore.kernel.org/iommu                            │
│  → kernelnewbies.org/LinuxChanges                   │
└─────────────────────────────────────────────────────┘
```
## Phase 8: Writing simple module

### الفكرة: kprobe على `iommu_map`

الـ `iommu_map` هي واحدة من أكثر الـ exported functions إثارةً في الـ IOMMU subsystem — بتُنفَّذ في كل مرة بيحتاج فيها أي device driver يعمل DMA mapping عبر الـ IOMMU. سنعمل **kprobe** عليها عشان نشوف مين بيعمل mapping، وعلى أي IOVA، وبأي size وأي permission flags.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * iommu_map_probe.c
 * Attaches a kprobe to iommu_map() and logs every IOMMU mapping request:
 * IOVA, physical address, size, and protection flags.
 */

#include <linux/module.h>      /* MODULE_*, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/iommu.h>       /* iommu_domain, IOMMU_READ/WRITE/... */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("IOMMU Explorer");
MODULE_DESCRIPTION("kprobe on iommu_map() to trace DMA IOVA mappings");

/*
 * iommu_map signature (from include/linux/iommu.h):
 *   int iommu_map(struct iommu_domain *domain, unsigned long iova,
 *                 phys_addr_t paddr, size_t size, int prot, gfp_t gfp);
 *
 * On x86_64 the args come in: rdi=domain, rsi=iova, rdx=paddr,
 *                              rcx=size,   r8=prot,  r9=gfp
 * We use regs_get_kernel_argument() which is arch-independent.
 */

/* ---------------------------------------------------------- */
/* Helper: decode prot flags into a human-readable string     */
/* ---------------------------------------------------------- */
static void prot_to_str(int prot, char *buf, size_t bufsz)
{
    snprintf(buf, bufsz, "%s%s%s%s%s",
        (prot & IOMMU_READ)    ? "R" : "-",
        (prot & IOMMU_WRITE)   ? "W" : "-",
        (prot & IOMMU_CACHE)   ? "C" : "-",
        (prot & IOMMU_NOEXEC)  ? "X" : "-",
        (prot & IOMMU_MMIO)    ? "M" : "-");
}

/* ---------------------------------------------------------- */
/* kprobe pre-handler: called just before iommu_map executes  */
/* ---------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Grab arguments from CPU registers in an arch-neutral way */
    struct iommu_domain *domain =
        (struct iommu_domain *)regs_get_kernel_argument(regs, 0);
    unsigned long iova  = (unsigned long)regs_get_kernel_argument(regs, 1);
    phys_addr_t   paddr = (phys_addr_t)  regs_get_kernel_argument(regs, 2);
    size_t        size  = (size_t)       regs_get_kernel_argument(regs, 3);
    int           prot  = (int)          regs_get_kernel_argument(regs, 4);

    char prot_str[8];
    prot_to_str(prot, prot_str, sizeof(prot_str));

    /*
     * domain->type carries the domain kind (DMA / IDENTITY / SVA …).
     * Printing it helps us filter out irrelevant identity-mapped calls.
     */
    pr_info("iommu_map: domain=%p type=0x%x iova=%#lx paddr=%#llx "
            "size=%zu prot=[%s] comm=%s pid=%d\n",
            domain,
            domain ? domain->type : 0xdeadU,
            iova,
            (unsigned long long)paddr,
            size,
            prot_str,
            current->comm,
            current->pid);

    return 0; /* 0 = continue normal execution */
}

/* ---------------------------------------------------------- */
/* kprobe descriptor                                          */
/* ---------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "iommu_map",   /* kernel symbol to probe  */
    .pre_handler = handler_pre,   /* our callback            */
};

/* ---------------------------------------------------------- */
/* Module init: register the kprobe                           */
/* ---------------------------------------------------------- */
static int __init iommu_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("iommu_map_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("iommu_map_probe: planted at %p\n", kp.addr);
    return 0;
}

/* ---------------------------------------------------------- */
/* Module exit: MUST unregister before the module is removed  */
/* ---------------------------------------------------------- */
static void __exit iommu_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("iommu_map_probe: removed\n");
}

module_init(iommu_probe_init);
module_exit(iommu_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | الماكرو الأساسية لأي kernel module |
| `<linux/kernel.h>` | `pr_info` / `pr_err` |
| `<linux/kprobes.h>` | الـ `struct kprobe` وكل الـ API بتاعتها |
| `<linux/iommu.h>` | تعريف `iommu_domain` والـ flags `IOMMU_READ/WRITE/…` |

#### الدالة `prot_to_str`

بتحوّل الـ bitmask للـ `prot` لـ string قصير زي `RW---` عشان الـ log يبقى مقروء بسرعة بدل ما تطبع أرقام hex محتاج تفككها يدوياً.

#### الـ `handler_pre`

ده قلب الـ kprobe. بيتشغّل قبل ما `iommu_map` تنفّذ سطرها الأول.
- **`regs_get_kernel_argument(regs, N)`**: طريقة portable لجيب الـ argument رقم N من الـ registers بدون ما تحتاج تعرف الـ calling convention للـ architecture.
- بنطبع: مين الـ domain، الـ IOVA اللي هيتعمل عليها الـ mapping، الـ physical address، الـ size، والـ prot flags، وكمان اسم الـ process اللي طلب الـ mapping ومعاه الـ PID — ده مهم عشان الـ DMA mapping بتتطلب من أي context (kernel thread, softirq, …).

#### الـ `struct kprobe kp`

- **`symbol_name`**: بيقول للـ kernel "حط الـ breakpoint على الـ symbol ده بالاسم" — مش محتاج تعرف الـ address يدوياً.
- **`pre_handler`**: الـ callback اللي هيتشغّل قبل الدالة.

#### الـ `module_init` / `module_exit`

- الـ `register_kprobe` بتحط one-byte breakpoint (INT3 على x86) في بداية `iommu_map` وبتسجّل الـ callback.
- الـ `unregister_kprobe` في الـ exit **إجباري**: لو شلنا الـ module من غير ما نشيل الـ kprobe، أي call لـ `iommu_map` بعدها هيجمّد الـ kernel لأن الـ handler pointer بقى invalid.

---

### كيفية البناء والتشغيل

```bash
# Makefile بسيط
obj-m += iommu_map_probe.o

# بناء
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod iommu_map_probe.ko

# مشاهدة الـ log
sudo dmesg -w | grep iommu_map

# إزالة
sudo rmmod iommu_map_probe
```

### مثال على الـ output المتوقع

```
[  142.381204] iommu_map_probe: planted at ffffffffc0a1b2e0
[  142.501337] iommu_map: domain=ffff8881234ab000 type=0x3 iova=0x100000000 paddr=0x80000000 size=4096 prot=[RW---] comm=kworker/u8:2 pid=312
[  142.501412] iommu_map: domain=ffff8881234ab000 type=0x3 iova=0x100001000 paddr=0x80001000 size=4096 prot=[RW---] comm=kworker/u8:2 pid=312
```

**الـ type=0x3** يعني `IOMMU_DOMAIN_DMA` (الـ bit 0 + bit 1)، وده الـ domain النوع الأكثر شيوعاً في أنظمة الـ DMA عبر الـ IOMMU.
