## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الملف `include/linux/iova.h` بيتبع لـ subsystem اسمه **IOMMU DMA-API Layer** — وده جزء من الـ **IOMMU Subsystem** اللي بيتحكم في ازاي الـ hardware devices بتترجم العناوين عشان توصل للـ memory.

---

### القصة من الأول: ليه IOVA أصلاً موجود؟

تخيل إن عندك **GPU أو NIC** (كارت شبكة) وعايزه يكتب على الـ RAM مباشرة بدون ما يمر على الـ CPU — ده اسمه **DMA (Direct Memory Access)**.

المشكلة: الـ device مش شايف نفس الـ memory addresses اللي شايفها الـ kernel. الـ kernel بيتعامل مع **Virtual Addresses** و **Physical Addresses**، لكن الـ device بيحتاج حاجة تالتة: **IOVA — I/O Virtual Address**.

الـ **IOVA** هو عنوان وهمي مخصوص للـ device — بتحدده الـ IOMMU (وحدة الترجمة الخاصة بالـ I/O) — وبتعمل mapping بينه وبين الـ physical address اللي في الـ RAM.

```
┌─────────────────────────────────────────────────────────────┐
│   Device (GPU/NIC)                                          │
│   بتشوف: IOVA (I/O Virtual Address)                        │
│           ↓                                                  │
│       [ IOMMU ]  ← بتترجم IOVA → Physical Address          │
│           ↓                                                  │
│       [ RAM ]    ← الـ actual memory                        │
└─────────────────────────────────────────────────────────────┘
```

---

### تشبيه من الواقع

فكر في **مطار** فيه مواقف سيارات مرقمة. كل ضيف (device) بياخد رقم موقف (IOVA). إدارة المطار (IOMMU) عارفة إن رقم الموقف 47 ده في الأرض الجنوبية (Physical Address). الضيف مش بيحتاج يعرف الخريطة الكاملة — بس بيحتاج رقم الموقف.

الـ `iova.h` ده هو الـ **نظام اللي بيحجز ويدير أرقام المواقف دي**.

---

### هدف الملف

الـ `iova.h` بيعرّف:

1. **`struct iova`** — بيمثل نطاق عناوين IOVA محجوزة (من `pfn_lo` لـ `pfn_hi`).
2. **`struct iova_domain`** — بيمثل "مخزن" كل الـ IOVAs الخاصة بـ device معين أو domain معين، مع الـ red-black tree اللي بتخزنهم فيه.
3. **Inline helper functions** — لعمليات زي حساب الـ size والـ offset والـ alignment.
4. **API declarations** — لحجز وتحرير الـ IOVAs (سواء slow path أو fast path بالـ rcache).

---

### الـ Key Concepts جوه الملف

#### الـ `struct iova`
```c
struct iova {
    struct rb_node  node;    /* node في الـ red-black tree */
    unsigned long   pfn_hi;  /* أعلى page frame number محجوز */
    unsigned long   pfn_lo;  /* أدنى page frame number محجوز */
};
```
كل `iova` بتمثل **range متواصل** من الـ I/O pages المحجوزة. الـ `pfn` (Page Frame Number) بدل الـ byte address عشان نوفر memory ونسهّل الحسابات.

#### الـ `struct iova_domain`
ده الـ "مدير الحجوزات" الكامل:
- **`rbroot`**: شجرة حمراء-سوداء لكل الـ IOVAs المحجوزة — بتضمن البحث والحجز في O(log n).
- **`cached_node` / `cached32_node`**: تسريع الحجز بالـ caching آخر node اتحجز (عشان غالباً الحجز بيكون sequential).
- **`granule`**: أصغر وحدة حجز — غالباً بيساوي الـ page size (4KB مثلاً).
- **`dma_32bit_pfn`**: حد الـ 32-bit DMA addresses (بعض الأجهزة القديمة مش بتعرف تتجاوز 4GB).
- **`rcaches`**: **Per-CPU caches** للـ IOVAs المحررة — عشان نتجنب الـ lock contention في الـ fast path.

#### الـ Fast Path vs Slow Path
| | Slow Path | Fast Path |
|---|---|---|
| الدالة | `alloc_iova()` | `alloc_iova_fast()` |
| الأسلوب | بيدور في الـ rbtree | بياخد من الـ per-CPU rcache |
| الـ Lock | spinlock كامل | lock-free في الغالب |
| الاستخدام | أول مرة أو لما الـ cache فاضي | الحالة العادية |

#### الـ Inline Helpers
```c
/* حجم الـ IOVA بالـ pages */
iova_size(iova)

/* تحويل من pfn لـ DMA address */
iova_dma_addr(iovad, iova)

/* تحويل من DMA address لـ pfn */
iova_pfn(iovad, iova)

/* alignment للـ granule */
iova_align(iovad, size)
```

---

### القصة الكاملة: ازاي بتتعمل DMA Operation

```
1. Driver بيطلب حجز IOVA لـ buffer حجمه 8KB
        ↓
2. alloc_iova_fast() → بتدور في الـ rcache الأول
        ↓
3. لو مفيش في الـ cache → alloc_iova() → بتدور في الـ rbtree
        ↓
4. بترجع iova struct فيها pfn_lo و pfn_hi
        ↓
5. الـ IOMMU بيعمل mapping: IOVA Range → Physical Pages
        ↓
6. الـ device بيعمل DMA باستخدام الـ IOVA address
        ↓
7. لما خلص → free_iova_fast() → بترجع الـ range للـ rcache
```

---

### ليه الـ Red-Black Tree؟

الـ IOVAs لازم تكون **unique ومتداخلتش**. الـ rbtree بتخلي:
- الـ **allocation** سريع: بنلاقي أول فراغ مناسب في O(log n)
- الـ **lookup** سريع: نلاقي أي IOVA بالـ pfn في O(log n)
- الـ **free** سريع: نشيل الـ node ونعيد التوازن

---

### ليه فيه `CONFIG_IOMMU_IOVA` Guard؟

مش كل الـ kernels محتاجة IOMMU. لو الـ config مش مفعّل، كل الـ functions بترجع `NULL` أو `0` بدون ما تعمل حاجة — بدل ما نكسر الـ code اللي بيعتمد عليها.

---

### الملفات اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `drivers/iommu/iova.c` | الـ implementation الكاملة لكل دوال الـ iova.h |
| `drivers/iommu/dma-iommu.c` | الـ DMA-API layer اللي بتستخدم الـ IOVA لعمل mappings |
| `drivers/iommu/dma-iommu.h` | header خاص بالـ DMA-IOMMU layer |
| `include/linux/iommu.h` | الـ IOMMU subsystem API الأساسية |
| `include/linux/iommu-dma.h` | interface بين الـ DMA layer والـ IOMMU |
| `include/linux/dma-mapping.h` | الـ DMA API العامة اللي الـ drivers بتستخدمها |
| `include/linux/rbtree.h` | تعريف الـ red-black tree اللي بيخزن الـ IOVAs |

---

### ملفات الـ HW Drivers اللي بتستخدم الـ IOVA

```
drivers/iommu/
├── intel/          ← Intel VT-d IOMMU
├── amd/            ← AMD-Vi IOMMU
├── arm/            ← ARM SMMU
├── apple-dart.c    ← Apple DART IOMMU
├── tegra-smmu.c    ← NVIDIA Tegra SMMU
├── msm_iommu.c     ← Qualcomm MSM IOMMU
└── mtk_iommu.c     ← MediaTek IOMMU
```

كل الـ drivers دي ممكن تستخدم الـ IOVA allocator بشكل غير مباشر عن طريق الـ `dma-iommu.c`.
## Phase 2: شرح الـ IOVA (I/O Virtual Address) Framework

---

### المشكلة اللي بيحلها الـ IOVA Framework

لما device زي GPU أو NIC أو DMA controller عايز يقرأ أو يكتب في الـ RAM، بيحتاج يعرف العنوان في الـ physical memory. المشكلة في كذا نقطة:

1. **الـ physical memory مش contiguous دايمًا** — الـ kernel ممكن يخصص pages متفرقة في الـ RAM لكن الـ device محتاج range واحدة متصلة.
2. **الـ device بيشوف address space مختلف** — مش نفس الـ virtual address بتاع الـ kernel ولا نفس الـ physical address بالضرورة.
3. **الحماية والعزل** — في الـ virtualization، guest VMs محتاجة تفكر إن devices بتاعتها بتشوف physical addresses، لكن من غير ما تقدر تعدّل على الـ host memory برا نطاقها.

الحل الموجود في الـ hardware هو الـ **IOMMU** (Input-Output Memory Management Unit)، وهو unit في الـ CPU/chipset بيعمل address translation للـ DMA transactions بدل ما الـ CPU يعملها للـ CPU accesses.

لكن الـ IOMMU محتاج حد يدير الـ address space بتاعه — يعني يقرر: "الـ DMA address رقم X هيشير على الـ physical page رقم Y". ده بالظبط اللي بيعمله الـ IOVA framework: **بيدير تخصيص وإدارة الـ I/O Virtual Addresses**.

---

### الحل اللي بيقدمه الـ IOVA Framework

الـ IOVA framework بيوفر allocator متخصص لـ **I/O address space** داخل domain معين. كل domain ليه نطاق (range) من العناوين المتاحة، وكل ما جاء driver محتاج يعمل DMA mapping، بيطلب من الـ IOVA allocator يديله chunk من العناوين دي.

المميزات الأساسية:

- **Red-Black Tree** لتتبع المناطق المخصصة — بيسمح بـ O(log n) search وinsert وdelete.
- **Per-CPU rcaches** (rcache = remote cache) — بتقلل الـ lock contention في الـ multicore systems عن طريق caching الـ freed IOVAs على مستوى كل CPU قبل ما ترجع للـ tree.
- **دعم 32-bit limit** — بعض الـ legacy devices مش قادرة تتعامل مع addresses فوق الـ 4GB، فالـ framework بيتعامل مع ده بـ `dma_32bit_pfn` و`cached32_node`.
- **Granule alignment** — كل الـ allocations لازم تكون aligned على الـ granule size (عادةً page size أو أكبر).

---

### التشبيه الواقعي — مكتب تخصيص الشقق

تخيل مدينة (= الـ I/O address space) فيها شقق (= pages). في المدينة دي مكتب تخصيص (= `iova_domain`) شغله إنه يخصص عناوين للسكان (= devices).

| المدينة / المكتب | الـ Kernel Concept |
|---|---|
| المدينة كلها (من الشارع رقم 1 لـ N) | الـ I/O address space بتاع الـ domain |
| مكتب التخصيص | `struct iova_domain` |
| سجل المناطق المحجوزة | الـ Red-Black Tree (`rbroot`) |
| وحدة قياس (شقة / طابق) | الـ `granule` |
| أدنى شارع متاح | `start_pfn` |
| حد أقصى للـ legacy devices (الـ 4GB) | `dma_32bit_pfn` |
| عقد إيجار لشقة معينة | `struct iova` (بيحدد `pfn_lo` و`pfn_hi`) |
| موظف سريع بيحتفظ بسجل محلي | الـ `iova_rcache` per-CPU |

الـ device لما بيطلب DMA mapping، بيقول "أنا محتاج 16 page". المكتب بيبص في السجل (الـ RB-tree)، يلاقي مكان فاضي، يكتب عقد إيجار (`struct iova` بـ pfn_lo و pfn_hi)، ويديه العنوان. لما الـ DMA خلص، العقد بيتشال والمنطقة بترجع متاحة.

الـ rcache زي موظف في كل فرع (CPU) عنده شوية عقود إيجار فاضية جاهزة — بدل ما يروح للمكتب المركزي (الـ global RB-tree + spinlock) كل مرة.

---

### البيك بيكتشر — موقع الـ IOVA في الـ Kernel

```
  +------------------+     +------------------+     +------------------+
  |   Network Driver  |     |   GPU Driver      |     |   NVMe Driver     |
  |  (e3100, mlx5)   |     |  (amdgpu, i915)   |     |   (nvme-pci)      |
  +--------+---------+     +--------+----------+     +--------+---------+
           |                        |                          |
           |      dma_map_*()       |                          |
           +------------------------+--------------------------+
                                    |
                         +----------v----------+
                         |    DMA Mapping API   |  <-- linux/dma-mapping.h
                         |  (dma_map_single,   |
                         |   dma_map_sg, ...)  |
                         +----------+----------+
                                    |
                         +----------v----------+
                         |    IOMMU Core Layer  |  <-- drivers/iommu/iommu.c
                         |   (iommu_map,       |
                         |    iommu_unmap)     |
                         +----------+----------+
                                    |
               +--------------------+--------------------+
               |                                         |
    +----------v----------+                   +----------v----------+
    |   IOVA Allocator    |                   |  IOMMU HW Driver    |
    | (iova.h / iova.c)   |                   | (intel_iommu.c,     |
    |                     |                   |  arm_smmu.c, ...)   |
    | - iova_domain       |                   |                     |
    | - RB-tree           |                   | - page table mgmt   |
    | - rcaches           |                   | - TLB flush         |
    +---------------------+                   +---------------------+
                                                        |
                                              +---------v---------+
                                              |   IOMMU Hardware  |
                                              |  (Intel VT-d,     |
                                              |   ARM SMMUv3)     |
                                              +-------------------+
                                                        |
                                              +---------v---------+
                                              |   Physical RAM     |
                                              +-------------------+
```

**ملاحظة مهمة:** الـ IOVA framework مش جزء من الـ IOMMU hardware driver ولا من الـ DMA API — هو طبقة وسط منفصلة مسؤولة فقط عن **address space management**.

---

### الـ Core Abstraction — ما هو المفهوم المحوري؟

المفهوم المحوري هو **الـ IOVA Domain**: نطاق معزول من العناوين الافتراضية خاص بـ device معين (أو مجموعة devices). كل domain مستقل تمامًا — الـ addresses فيه مش مرتبطة بالـ virtual memory بتاع الـ kernel.

الـ **IOVA** نفسه (الـ struct) هو مجرد range: من `pfn_lo` لـ `pfn_hi`. المفهوم بالكامل مبني على الـ **PFN (Page Frame Number)** بدل الـ byte addresses — عشان يكون أكثر كفاءة في الحسابات والمقارنات.

```
  IOVA Address Space (domain-specific):

  0                    dma_32bit_pfn           max_pfn
  |____________________|___________________________|
       32-bit zone              64-bit zone

  |----[iova_1]----| |--[iova_2]--| |-[iova_3]-|
      pfn_lo pfn_hi  pfn_lo pfn_hi  pfn_lo pfn_hi

  granule = page_size (e.g., 4096 bytes = 1 pfn)
```

---

### الـ Structs الأساسية وعلاقتها ببعض

#### `struct iova` — وحدة التخصيص

```c
struct iova {
    struct rb_node  node;    /* node في الـ RB-tree */
    unsigned long   pfn_hi;  /* أعلى PFN في الـ range (inclusive) */
    unsigned long   pfn_lo;  /* أدنى PFN في الـ range (inclusive) */
};
```

الـ `rb_node` هو الـ intrusive link — بدل ما تعمل pointer من الـ tree node للـ iova، الـ iova نفسه بيحمل الـ node جوه. للوصول من الـ node للـ iova بتستخدم `rb_entry(node_ptr, struct iova, node)`.

#### `struct iova_domain` — المدير الكلي

```c
struct iova_domain {
    spinlock_t       iova_rbtree_lock;  /* يحمي الـ RB-tree من concurrent access */
    struct rb_root   rbroot;            /* root الـ RB-tree */
    struct rb_node  *cached_node;       /* آخر node تم alloc منه (64-bit) */
    struct rb_node  *cached32_node;     /* آخر node تم alloc منه (32-bit) */
    unsigned long    granule;           /* أصغر وحدة تخصيص (بالـ PFNs) */
    unsigned long    start_pfn;         /* بداية الـ address space */
    unsigned long    dma_32bit_pfn;     /* حد الـ 32-bit (4GB >> page_shift) */
    unsigned long    max32_alloc_size;  /* أكبر alloc فشلت في الـ 32-bit zone */
    struct iova      anchor;            /* sentinel node للـ tree traversal */
    struct iova_rcache *rcaches;        /* per-CPU free lists */
    struct hlist_node  cpuhp_dead;      /* للـ CPU hotplug cleanup */
};
```

#### الـ RB-Tree وعلاقته بالـ `iova`

الـ tree مرتب بالـ `pfn_lo` تصاعديًا. كل node هو `struct iova` مُضمّن داخله `rb_node`.

```
                        iova_domain
                        +----------+
                        | rbroot   |---+
                        | ...      |   |
                        +----------+   |
                                       |
                          RB-Tree (ordered by pfn_lo)

                             [anchor: pfn=0]
                            /               \
                    [iova A]               [iova C]
                   pfn: 10-19             pfn: 50-79
                   /         \
              [iova X]    [iova B]
             pfn: 5-9    pfn: 30-39


  iova A:
  +----------+
  | rb_node  |  <-- متصل بالـ tree
  | pfn_hi=19|
  | pfn_lo=10|
  +----------+
```

---

### الـ Inline Helper Functions — فهم عميق

#### تحويل PFN ↔ DMA Address

الـ granule هو دايمًا power of 2، فـ `iova_shift` بيحسب log2(granule) باستخدام `__ffs` (find first set bit):

```c
/* لو granule = 4096 (0x1000), الـ shift = 12 */
static inline unsigned long iova_shift(struct iova_domain *iovad) {
    return __ffs(iovad->granule);  /* = 12 for 4KB pages */
}

/* تحويل iova struct لـ DMA address */
static inline dma_addr_t iova_dma_addr(struct iova_domain *iovad, struct iova *iova) {
    return (dma_addr_t)iova->pfn_lo << iova_shift(iovad);
    /* pfn_lo=10, shift=12 → dma_addr = 0xA000 */
}

/* تحويل DMA address لـ PFN */
static inline unsigned long iova_pfn(struct iova_domain *iovad, dma_addr_t iova) {
    return iova >> iova_shift(iovad);
    /* 0xA000 >> 12 = 0xA = 10 */
}
```

#### الـ Alignment و Offset

```c
/* الـ mask للـ bits أدنى من الـ granule */
static inline unsigned long iova_mask(struct iova_domain *iovad) {
    return iovad->granule - 1;  /* granule=4096 → mask=0xFFF */
}

/* الـ offset داخل الـ granule */
static inline size_t iova_offset(struct iova_domain *iovad, dma_addr_t iova) {
    return iova & iova_mask(iovad);  /* مثلاً 0xA123 & 0xFFF = 0x123 */
}

/* تقريب للأعلى لأقرب granule */
static inline size_t iova_align(struct iova_domain *iovad, size_t size) {
    return ALIGN(size, iovad->granule);  /* 5000 bytes → 8192 (next 4KB) */
}
```

---

### الـ Per-CPU rcache — تحسين الـ Performance

الـ `iova_rcache` هو optimization حيوي في الـ high-throughput scenarios. المشكلة: كل `alloc_iova` و`free_iova` بتحتاج تاخد الـ `spinlock_t iova_rbtree_lock`، وده bottleneck في الـ systems اللي فيها كتير من network/storage I/O على multicore.

الحل: كل CPU عنده **free list** (rcache) من الـ IOVAs اللي اتحررت قريبًا. لما تيجي تـ allocate:
1. أول حاجة بتبص في الـ rcache بتاع الـ CPU الحالي.
2. لو لقيت IOVA بالـ size المطلوب، بتاخده على طول بدون lock.
3. لو ما لقيتش، تروح للـ global RB-tree بالـ lock.

لما تحرر IOVA:
1. بتحطها في الـ rcache أول.
2. لو الـ rcache امتلأ، بتـ flush مجموعة منهم للـ global tree.

```
CPU 0:                 CPU 1:                 CPU 2:
+----------+          +----------+           +----------+
| rcache[0]|          | rcache[0]|           | rcache[0]|
| [iova_x] |          | [iova_y] |           | [iova_z] |
| [iova_a] |          | [iova_b] |           |          |
+----+-----+          +----+-----+           +----------+
     |                     |
     +---------------------+----> Global RB-Tree (spinlock protected)
```

الـ `iova_rcache_range()` بترجع أكبر size قادر الـ rcache يدير cache له.

---

### الـ Fast vs Slow Path

الـ API بيوفر مسارين:

| Function | المسار | متى تستخدمه |
|---|---|---|
| `alloc_iova()` | Slow (RB-tree مباشرة) | الـ initial setup أو الـ cases النادرة |
| `alloc_iova_fast()` | Fast (rcache أولاً) | الـ hot path — كل DMA mapping عادي |
| `free_iova()` | Slow (RB-tree مباشرة) | نادر |
| `free_iova_fast()` | Fast (يحط في rcache) | الـ hot path — كل DMA unmap |

---

### ما يملكه الـ IOVA Framework vs ما يفوّضه للـ Drivers

| الـ IOVA Framework يمتلك | يفوّضه للـ IOMMU Driver |
|---|---|
| تخصيص وتحرير الـ I/O virtual addresses | بناء الـ page tables الفعلية في الـ hardware |
| تتبع المناطق المحجوزة (RB-tree) | إجراء الـ TLB flushes |
| الـ per-CPU caching لتحسين الأداء | تهيئة الـ IOMMU hardware نفسه |
| الـ 32-bit zone management | تحديد حجم الـ page tables المدعومة |
| تحويلات الـ PFN ↔ DMA address | التنسيق مع الـ CPU architecture |

---

### الـ Lifecycle الكامل لـ IOVA Domain

```
1. init_iova_domain(iovad, PAGE_SIZE, start_pfn)
        |
        v
2. iova_domain_init_rcaches(iovad)   [اختياري للـ performance]
        |
        v
3. reserve_iova(iovad, lo, hi)       [حجز مناطق خاصة مثلاً MSI regions]
        |
        v
4. alloc_iova_fast(iovad, npages, limit_pfn, flush_rcache)
   → يرجع pfn_lo للـ allocated range
        |
        v
5. IOMMU driver يعمل iommu_map() للـ physical pages على الـ IOVAs
        |
        v
6. Device يعمل DMA باستخدام (pfn_lo << page_shift) كـ bus address
        |
        v
7. free_iova_fast(iovad, pfn, size)
        |
        v
8. put_iova_domain(iovad)            [cleanup كل حاجة]
```

---

### مفاهيم من Subsystems تانية محتاج تعرفها

- **IOMMU Subsystem** (`drivers/iommu/`): بيستخدم الـ IOVA framework لإدارة الـ address space، وهو المسؤول عن الـ hardware page tables. الـ IOVA framework مجرد allocator — الـ IOMMU هو اللي بيعمل الـ mapping الفعلي.
- **DMA Mapping API** (`include/linux/dma-mapping.h`): الـ high-level API اللي الـ drivers بتستخدمه. بتتعامل مع الـ IOMMU (وبالتالي الـ IOVA) أو مع الـ SWIOTLB لو مفيش IOMMU.
- **Red-Black Tree** (`include/linux/rbtree.h`): data structure الأساسي. بيضمن O(log n) في كل العمليات. الـ kernel بيستخدمه كتير في الـ memory management (مثلاً الـ VMAs في `mm_struct`).
- **CPU Hotplug** (`cpuhp_dead` field): لما CPU بيتوقف، الـ rcache بتاعه لازم يتـ flush للـ global tree عشان الـ IOVAs مش تضيع.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Compile-time Guards

| Option | الأثر |
|--------|-------|
| `CONFIG_IOMMU_IOVA` | لو مش موجود، كل الـ API بتتحول لـ stubs فاضية — مفيش allocations، مفيش domain |

الـ guard بيكون بـ `IS_REACHABLE(CONFIG_IOMMU_IOVA)` مش `IS_ENABLED` — يعني بيشتغل حتى لو الـ config في module خارجي.

---

### الـ Structs

#### 1. `struct iova`

**الهدف:** تمثيل نطاق عناوين IOVA واحد (I/O Virtual Address range) مخصص لجهاز معين.

```c
struct iova {
    struct rb_node  node;    /* embeds the rbtree linkage */
    unsigned long   pfn_hi;  /* highest PFN in range (inclusive) */
    unsigned long   pfn_lo;  /* lowest PFN in range (inclusive) */
};
```

| Field | النوع | الشرح |
|-------|-------|-------|
| `node` | `struct rb_node` | الـ hook اللي بيربط الـ iova بالـ rbtree في الـ domain |
| `pfn_hi` | `unsigned long` | أعلى page frame number في النطاق (inclusive) |
| `pfn_lo` | `unsigned long` | أدنى page frame number في النطاق (inclusive) |

- الـ size بتحسبها: `pfn_hi - pfn_lo + 1`
- الـ DMA address الفعلي: `pfn_lo << granule_shift`
- ممكن يكون `anchor` ثابت في الـ domain زي sentinel node.

---

#### 2. `struct iova_domain`

**الهدف:** الـ container الرئيسي اللي بيحتوي على كل الـ IOVA allocations لـ IOMMU domain واحد (يعني لجهاز أو مجموعة أجهزة).

```c
struct iova_domain {
    spinlock_t       iova_rbtree_lock; /* protects the rbtree */
    struct rb_root   rbroot;           /* root of the allocation rbtree */
    struct rb_node  *cached_node;      /* last allocated node (fast path) */
    struct rb_node  *cached32_node;    /* last 32-bit allocated node */
    unsigned long    granule;          /* allocation granularity in pages */
    unsigned long    start_pfn;        /* minimum allocatable PFN */
    unsigned long    dma_32bit_pfn;    /* upper bound for 32-bit DMA */
    unsigned long    max32_alloc_size; /* largest failed 32-bit allocation */
    struct iova      anchor;           /* sentinel node in the rbtree */
    struct iova_rcache *rcaches;       /* per-CPU rcache array pointer */
    struct hlist_node  cpuhp_dead;     /* CPU hotplug death notifier */
};
```

| Field | الشرح |
|-------|-------|
| `iova_rbtree_lock` | **spinlock** بيحمي الـ rbtree من concurrent modifications |
| `rbroot` | جذر الـ Red-Black Tree اللي فيه كل الـ `iova` nodes مرتبة بـ PFN |
| `cached_node` | مؤشر للآخر node اتخصص — تحسين للـ allocation الجديدة (بتبدأ من الآخر) |
| `cached32_node` | نفس الفكرة بس للـ allocations اللي lazim تبقى تحت `dma_32bit_pfn` |
| `granule` | وحدة الـ allocation بالـ pages — lazim تكون power-of-2 |
| `start_pfn` | الحد الأدنى للـ IOVA space (مش ممكن تخصص قبله) |
| `dma_32bit_pfn` | الحد الأقصى لـ 32-bit DMA (عادةً `0x100000000 >> PAGE_SHIFT`) |
| `max32_alloc_size` | بتتحدث لما allocation تفشل — بتتجنب محاولات أكبر منها في المستقبل |
| `anchor` | sentinel node ثابت في الـ rbtree بيسهّل الـ boundary checks |
| `rcaches` | مصفوفة من `iova_rcache` بتوفر per-CPU caching لتسريع الـ alloc/free |
| `cpuhp_dead` | بتسجل الـ domain في notifier chain لما CPU يموت (لتنظيف الـ rcaches) |

---

#### 3. `struct iova_rcache` (forward declaration فقط في الـ header)

**الهدف:** per-CPU cache للـ iova pages — بتتجنب الـ spinlock في الـ fast path.

مش معرّفة في الـ header (forward decl فقط)، لكن المعروف من implementation:
- كل CPU عنده magazine من الـ freed iovas جاهزة للـ reuse.
- `alloc_iova_fast` / `free_iova_fast` بيستخدموها مباشرةً.
- `iova_rcache_range()` بترجع الـ size المخصص لها.

---

### الـ Inline Helper Functions

| Function | الوظيفة |
|----------|---------|
| `iova_size(iova)` | عدد الـ pages في الـ range = `pfn_hi - pfn_lo + 1` |
| `iova_shift(iovad)` | الـ bit shift للـ granule = `__ffs(granule)` |
| `iova_mask(iovad)` | الـ mask للـ offset داخل granule = `granule - 1` |
| `iova_offset(iovad, addr)` | الـ byte offset داخل الـ granule |
| `iova_align(iovad, size)` | round up للـ granule |
| `iova_align_down(iovad, size)` | round down للـ granule |
| `iova_dma_addr(iovad, iova)` | تحويل `pfn_lo` لـ `dma_addr_t` |
| `iova_pfn(iovad, addr)` | تحويل `dma_addr_t` لـ PFN |

---

### مخطط العلاقات بين الـ Structs

```
  struct iova_domain
  ┌─────────────────────────────────────────────┐
  │  spinlock_t  iova_rbtree_lock               │
  │  struct rb_root  rbroot ──────────────────┐ │
  │  struct rb_node *cached_node ──────────┐  │ │
  │  struct rb_node *cached32_node ──────┐ │  │ │
  │  unsigned long  granule             │ │  │ │
  │  unsigned long  start_pfn           │ │  │ │
  │  unsigned long  dma_32bit_pfn       │ │  │ │
  │  struct iova    anchor (sentinel) ──┼─┼──┘ │
  │  struct iova_rcache *rcaches ──┐   │ │     │
  │  struct hlist_node cpuhp_dead  │   │ │     │
  └────────────────────────────────┼───┼─┼─────┘
                                   │   │ │
                    ┌──────────────┘   │ └──────────────────────┐
                    ▼                  │                         │
         struct iova_rcache[]          │                         │
         (per-CPU magazines)           ▼                         ▼
                                   Red-Black Tree (rbroot)
                              ┌─────────┴──────────┐
                              │                    │
                         struct iova          struct iova
                         ┌──────────┐         ┌──────────┐
                         │ rb_node  │         │ rb_node  │
                         │ pfn_lo   │         │ pfn_lo   │
                         │ pfn_hi   │         │ pfn_hi   │
                         └──────────┘         └──────────┘
                              │
                    ┌─────────┴──────────┐
                    │                   │
               struct iova         struct iova
               ...                 ...
```

**rb_node embedding:** الـ `struct iova` بتعمل embed لـ `struct rb_node` — يعني `container_of(node, struct iova, node)` بيرجع الـ iova الكاملة.

```
  struct iova في الـ memory:
  ┌────────────┬──────────┬──────────┐
  │  rb_node   │  pfn_hi  │  pfn_lo  │
  │ (24 bytes) │ (8 bytes)│ (8 bytes)│
  └────────────┴──────────┴──────────┘
       ↑
       └── rb_entry(ptr, struct iova, node) يوصل هنا
```

---

### الـ Lifecycle Diagram

```
  CREATION / INIT
  ───────────────
  caller
    └─► init_iova_domain(iovad, granule, start_pfn)
            │  ► init spinlock
            │  ► init rbroot
            │  ► set granule, start_pfn, dma_32bit_pfn
            │  ► insert anchor node in rbtree
            └─► iova_domain_init_rcaches(iovad)   [optional]
                    │  ► iova_cache_get()  [ref-count global slab cache]
                    └─► alloc per-CPU rcache magazines

  ALLOCATION (slow path)
  ──────────────────────
  driver
    └─► alloc_iova(iovad, size, limit_pfn, size_aligned)
            │  ► acquire iova_rbtree_lock (spin)
            │  ► search rbtree from cached_node downward
            │  ► find free gap >= size below limit_pfn
            │  ► alloc struct iova from slab
            │  ► insert into rbtree + rebalance
            │  ► update cached_node
            └─► release lock
            └── return struct iova*

  ALLOCATION (fast path)
  ──────────────────────
  driver
    └─► alloc_iova_fast(iovad, size, limit_pfn, flush_rcache)
            │  ► try rcache[cpu] (lockless)
            │  ► if hit → return pfn directly (no lock!)
            └─► if miss → alloc_iova() slow path
                        → on flush_rcache: drain all rcaches first

  LOOKUP
  ──────
  caller
    └─► find_iova(iovad, pfn)
            │  ► acquire lock
            │  ► binary search rbtree by pfn
            └─► release lock → return struct iova* or NULL

  RESERVE (static regions)
  ────────────────────────
  caller
    └─► reserve_iova(iovad, pfn_lo, pfn_hi)
            │  ► alloc struct iova
            └─► force-insert into rbtree (marks range as used)

  FREE (fast path)
  ────────────────
  driver
    └─► free_iova_fast(iovad, pfn, size)
            │  ► try to put in rcache[cpu] (lockless)
            └─► if rcache full → __free_iova() slow path

  FREE (slow path)
  ────────────────
  driver
    └─► free_iova(iovad, pfn)   [lookup by pfn then free]
          OR
        __free_iova(iovad, iova) [direct free of known iova*]
            │  ► acquire lock
            │  ► rb_erase(node, &iovad->rbroot)
            │  ► release lock
            └─► kmem_cache_free(iova_cache, iova)

  TEARDOWN
  ────────
  caller
    └─► put_iova_domain(iovad)
            │  ► drain all rcaches
            │  ► iova_cache_put()  [dec ref-count]
            └─► rbtree_postorder_for_each_entry_safe → free each iova
```

---

### الـ Call Flow Diagrams

#### alloc_iova_fast — الـ Fast Path

```
driver calls alloc_iova_fast(iovad, size, limit_pfn, flush_rcache)
  │
  ├─► check rcaches[smp_processor_id()]
  │     ├─► [HIT]  pop pfn from magazine  →  return pfn  (NO LOCK)
  │     └─► [MISS]
  │           ├─► if flush_rcache:
  │           │     drain_all rcaches → move freed iovas back to rbtree
  │           └─► alloc_iova(iovad, size, limit_pfn, true)
  │                   │
  │                   ├─► spin_lock(&iovad->iova_rbtree_lock)
  │                   ├─► __alloc_and_insert_iova_range()
  │                   │     ├─► start from cached_node (or cached32_node)
  │                   │     ├─► walk rbtree looking for gap
  │                   │     ├─► if gap found: alloc from slab, insert node
  │                   │     └─► update cached_node pointer
  │                   └─► spin_unlock(&iovad->iova_rbtree_lock)
  │                   └─► return iova->pfn_lo
  └── return pfn (0 on failure)
```

#### free_iova_fast — الـ Fast Path

```
driver calls free_iova_fast(iovad, pfn, size)
  │
  ├─► try iova_rcache_insert(iovad, pfn, size)
  │     ├─► [SUCCESS]  pfn stored in per-CPU magazine  (NO LOCK)
  │     └─► [FAIL - magazine full]
  │           └─► __free_iova(iovad, iova_ptr)
  │                   ├─► spin_lock(&iovad->iova_rbtree_lock)
  │                   ├─► rb_erase(&iova->node, &iovad->rbroot)
  │                   ├─► spin_unlock()
  │                   └─► kmem_cache_free(iova_cache, iova)
  └── done
```

#### find_iova

```
caller → find_iova(iovad, pfn)
  │
  ├─► spin_lock(&iovad->iova_rbtree_lock)
  ├─► node = rbroot.rb_node
  ├─► while node:
  │     iova = rb_entry(node, struct iova, node)
  │     if pfn < iova->pfn_lo  → node = node->rb_left
  │     if pfn > iova->pfn_hi  → node = node->rb_right
  │     else                   → FOUND → break
  ├─► spin_unlock()
  └─► return iova or NULL
```

---

### استراتيجية الـ Locking

| Lock | النوع | يحمي إيه | متى يتأخذ |
|------|-------|-----------|-----------|
| `iova_rbtree_lock` | `spinlock_t` | الـ `rbroot`، الـ `cached_node`، الـ `cached32_node`، الـ `max32_alloc_size` | قبل أي read/write للـ rbtree |
| rcache (lockless) | per-CPU magazine | الـ `iova_rcache` per-CPU slots | مفيش lock — atomic per-CPU access |

**ترتيب الـ Locking:**
- الـ `iova_rbtree_lock` دايمًا بيتأخذ قبل أي تعديل في الـ rbtree.
- الـ rcache بتشتغل lockless طالما الـ CPU ماشتغلش على cpu آخر — لو فيه overflow بتنزل للـ slow path وتأخذ الـ spinlock.
- مفيش nested locks داخل الـ iova subsystem نفسه.

**تحذير الـ CPU Hotplug:**
- لما CPU بيموت، الـ `cpuhp_dead` بيـ notify الـ domain عشان يـ drain الـ rcache بتاعته — ده بيحصل تحت الـ spinlock عشان ينقل الـ freed iovas للـ rbtree بدل ما تتضيع.

---

### ملخص العلاقات الكاملة

```
  iova_domain (1)
       │
       ├──[spinlock]──► protects rbroot + cached_nodes
       │
       ├──[rbroot]────► Red-Black Tree
       │                    │
       │              (N) struct iova nodes
       │                 sorted by pfn_lo
       │
       ├──[anchor]────► sentinel struct iova (pfn_lo = start_pfn - 1)
       │                inserted permanently to simplify boundary checks
       │
       └──[rcaches]───► struct iova_rcache[]  (one per CPU)
                             │
                        per-CPU magazines of free pfns
                        (lockless fast alloc/free)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Inline Helpers

| Function | Category | Returns | Purpose |
|---|---|---|---|
| `iova_size(iova)` | Helper | `unsigned long` | عدد الـ pages في الـ IOVA |
| `iova_shift(iovad)` | Helper | `unsigned long` | الـ bit-shift الخاص بالـ granule |
| `iova_mask(iovad)` | Helper | `unsigned long` | mask للـ offset داخل الـ granule |
| `iova_offset(iovad, iova)` | Helper | `size_t` | الـ byte offset داخل الـ granule |
| `iova_align(iovad, size)` | Helper | `size_t` | round-up للـ granule |
| `iova_align_down(iovad, size)` | Helper | `size_t` | round-down للـ granule |
| `iova_dma_addr(iovad, iova)` | Helper | `dma_addr_t` | تحويل من `pfn_lo` لـ DMA address |
| `iova_pfn(iovad, iova)` | Helper | `unsigned long` | تحويل DMA address لـ PFN |

#### Core API (CONFIG_IOMMU_IOVA)

| Function | Category | Returns | Purpose |
|---|---|---|---|
| `iova_cache_get()` | Init | `int` | تهيئة الـ slab cache للـ `struct iova` |
| `iova_cache_put()` | Cleanup | `void` | تحرير الـ slab cache |
| `iova_rcache_range()` | Query | `unsigned long` | أقصى size يُخزَّن في الـ rcache |
| `init_iova_domain()` | Init | `void` | تهيئة الـ `iova_domain` |
| `iova_domain_init_rcaches()` | Init | `int` | تفعيل الـ per-CPU rcaches |
| `alloc_iova()` | Alloc | `struct iova *` | تخصيص IOVA من الـ rbtree |
| `alloc_iova_fast()` | Alloc | `unsigned long` | تخصيص سريع عبر الـ rcache |
| `free_iova()` | Free | `void` | تحرير IOVA بالـ PFN |
| `__free_iova()` | Free | `void` | تحرير IOVA مباشرة بالـ pointer |
| `free_iova_fast()` | Free | `void` | إعادة IOVA للـ rcache |
| `reserve_iova()` | Reserve | `struct iova *` | حجز range ثابت لا يُخصَّص |
| `find_iova()` | Lookup | `struct iova *` | البحث عن IOVA بالـ PFN |
| `put_iova_domain()` | Cleanup | `void` | تدمير الـ domain وتحرير كل الـ IOVAs |

---

### Group 1: Initialization & Teardown

هاي المجموعة مسؤولة عن إعداد البنية التحتية للـ IOVA subsystem وتدميرها. لازم تتعمل قبل أي allocation.

---

#### `iova_cache_get`

```c
int iova_cache_get(void);
```

بتسجّل الـ `struct iova` slab cache (`iova_cache`) بواسطة `kmem_cache_create`. الـ IOMMU drivers بتناديها مرة واحدة في وقت الـ module/driver init. بتستخدم reference counting داخلي، يعني لو أكتر من driver نادوها، الـ cache ما بتتعمل أكتر من مرة.

- **Parameters**: لا يوجد.
- **Return**: `0` عند النجاح، `-ENOMEM` لو فشل الـ `kmem_cache_create`.
- **Key details**: غير thread-safe عند أول استدعاء — لازم تتنادى من context آمن (مثل `module_init`). الـ cache مشترك بين كل الـ IOMMU domains.
- **Who calls it**: IOMMU drivers مثل Intel VT-d, AMD-Vi عند التهيئة.

---

#### `iova_cache_put`

```c
void iova_cache_put(void);
```

عكس `iova_cache_get`، بتنقّص الـ reference count وبتدمّر الـ slab cache لما يوصل للصفر. لازم تتنادى عند الـ driver/module cleanup.

- **Parameters**: لا يوجد.
- **Return**: `void`.
- **Key details**: لو تنوديها أكتر من `iova_cache_get`، بتطبع WARN وبترجع بدون تدمير.
- **Who calls it**: نفس الـ drivers في مسار الـ `module_exit` أو الـ error path.

---

#### `init_iova_domain`

```c
void init_iova_domain(struct iova_domain *iovad,
                      unsigned long granule,
                      unsigned long start_pfn);
```

بتهيّئ الـ `iova_domain` من الصفر: بتعمل `spin_lock_init` للـ `iova_rbtree_lock`، وبتضبط الـ `granule` و`start_pfn`، وبتعمل insert للـ `anchor` node في الـ rbtree كـ sentinel يمنع الـ allocation من تحت الـ `start_pfn`.

- **Parameters**:
  - `iovad`: الـ domain المراد تهيئته.
  - `granule`: حجم الوحدة الأساسية للتخصيص بالـ bytes (لازم يكون power of 2، غالباً `PAGE_SIZE`).
  - `start_pfn`: أول PFN قابل للتخصيص في الـ IOVA space.
- **Return**: `void`.
- **Key details**: ما بتخصّص الـ rcaches — لازم تنادي `iova_domain_init_rcaches` بشكل منفصل لو محتاج performance. الـ `dma_32bit_pfn` بيتضبط على `DMA_BIT_MASK(32) >> iova_shift`.
- **Who calls it**: الـ IOMMU domain allocation code (مثل `intel_iommu_domain_alloc`).

---

#### `iova_domain_init_rcaches`

```c
int iova_domain_init_rcaches(struct iova_domain *iovad);
```

بتخصّص وبتهيّئ الـ per-CPU **rcaches** (remote caches) — بنية بتخزّن الـ recently-freed IOVAs لإعادة استخدامها بدون الحاجة لـ lock على الـ rbtree. كل CPU عنده مجموعة من الـ buckets، كل bucket لـ size معين.

- **Parameters**:
  - `iovad`: الـ domain اللي رح يستفيد من الـ rcaches.
- **Return**: `0` عند النجاح، `-ENOMEM` لو فشل الـ `kzalloc`.
- **Key details**: بتسجّل الـ domain في الـ `cpuhp_dead` notifier لتنظيف الـ per-CPU state لما CPU يموت. الـ rcaches بتشتغل على مبدأ LIFO stack per CPU per size-bucket.
- **Who calls it**: IOMMU drivers اللي بدها high-throughput DMA (مثل network drivers).

---

#### `put_iova_domain`

```c
void put_iova_domain(struct iova_domain *iovad);
```

بتدمّر الـ domain بالكامل: بتفرغ كل الـ rcaches، بتشيل الـ domain من الـ cpuhp notifier، وبتحرر كل الـ `struct iova` المخصّصة في الـ rbtree باستخدام `rbtree_postorder_for_each_entry_safe`.

- **Parameters**:
  - `iovad`: الـ domain المراد تدميره.
- **Return**: `void`.
- **Key details**: بعد الاستدعاء، الـ `iovad` بيصير unusable. ما بتحرر الـ `iovad` struct نفسه — المستدعي مسؤول. بتاخد الـ `iova_rbtree_lock` أثناء التنظيف.
- **Who calls it**: `iommu_domain_free` أو الـ IOMMU driver cleanup paths.

---

### Group 2: Allocation

الـ IOVA allocation بيصير بطريقتين: slow path عبر الـ rbtree مباشرة، وfast path عبر الـ rcache.

---

#### `alloc_iova`

```c
struct iova *alloc_iova(struct iova_domain *iovad,
                        unsigned long size,
                        unsigned long limit_pfn,
                        bool size_aligned);
```

الـ **slow-path allocator**. بيخصّص `struct iova` من الـ slab cache وبيدوّر على مكان فاضي في الـ rbtree. البحث بيصير من أعلى الـ IOVA space للأسفل (top-down) لتفادي الـ fragmentation.

- **Parameters**:
  - `iovad`: الـ domain المراد التخصيص منه.
  - `size`: عدد الـ pages (بالـ granule units) المطلوبة.
  - `limit_pfn`: الحد الأعلى للـ PFN (شامل). مفيد لتقييد الـ allocation ضمن 32-bit space مثلاً.
  - `size_aligned`: لو `true`، بيضمن إن الـ `pfn_lo` يكون aligned على الـ `size` نفسه (power-of-2 alignment).
- **Return**: `struct iova *` عند النجاح، `NULL` لو ما في space كافي.
- **Key details**: بيمسك الـ `iova_rbtree_lock` طول فترة التخصيص. بيحدّث الـ `cached_node` أو `cached32_node` لتسريع الـ allocations اللي بعدها. الـ `anchor` node بيمنع التخصيص تحت الـ `start_pfn`.

**Pseudocode Flow:**
```
alloc_iova(iovad, size, limit_pfn, size_aligned):
    iova = kmem_cache_zalloc(iova_cache)
    if !iova: return NULL

    spin_lock(iova_rbtree_lock)

    /* Start search from cached node or limit_pfn */
    curr = find_start_node(iovad, limit_pfn)

    loop:
        candidate_pfn_hi = curr->pfn_lo - 1
        candidate_pfn_lo = candidate_pfn_hi - size + 1

        if size_aligned:
            align candidate_pfn_lo to size boundary

        if candidate_pfn_lo >= iovad->start_pfn:
            /* Found a gap */
            iova->pfn_lo = candidate_pfn_lo
            iova->pfn_hi = candidate_pfn_hi
            insert into rbtree
            update cached_node
            break

        curr = rb_prev(curr)  /* Go lower */
        if no more nodes: FAIL

    spin_unlock(iova_rbtree_lock)
    return iova (or NULL on failure)
```

- **Who calls it**: `alloc_iova_fast` عند cache miss، وأي driver يحتاج IOVA مباشرة.

---

#### `alloc_iova_fast`

```c
unsigned long alloc_iova_fast(struct iova_domain *iovad,
                               unsigned long size,
                               unsigned long limit_pfn,
                               bool flush_rcache);
```

الـ **fast-path allocator**. بيحاول يجيب IOVA من الـ per-CPU rcache أولاً. لو ما لاقى، بيرجع لـ `alloc_iova` (slow path). بيرجع PFN مباشرة بدل pointer لتسهيل الاستخدام.

- **Parameters**:
  - `iovad`: الـ domain.
  - `size`: الحجم بالـ granule units.
  - `limit_pfn`: الحد الأعلى.
  - `flush_rcache`: لو `true`، بيفرغ الـ rcache ويعيد المحاولة لو فشل الـ slow path — آلية لاسترداد الـ IOVAs المخبّأة.
- **Return**: الـ PFN الأول من الـ IOVA المخصّص، أو `0` عند الفشل.
- **Key details**: الـ rcache lookup بيصير بدون lock (per-CPU). فقط لما بيصير رجوع للـ rbtree بيحتاج الـ lock. الـ `flush_rcache` بيسرّب كل الـ rcaches للـ rbtree ويعيد المحاولة.
- **Who calls it**: الـ IOMMU DMA map ops مثل `iommu_dma_alloc_iova` — الـ hot path لكل DMA mapping.

---

### Group 3: Deallocation

---

#### `free_iova`

```c
void free_iova(struct iova_domain *iovad, unsigned long pfn);
```

بتحرر الـ IOVA المرتبطة بالـ PFN المعطى. بتدوّر على الـ `struct iova` في الـ rbtree بواسطة `find_iova`، بتشيله من الـ tree، وبترجعه للـ slab cache.

- **Parameters**:
  - `iovad`: الـ domain.
  - `pfn`: أي PFN داخل الـ IOVA range (غالباً الـ `pfn_lo`).
- **Return**: `void`.
- **Key details**: بتمسك الـ `iova_rbtree_lock`. الـ `cached_node` وَ `cached32_node` بيتم invalidate لهم لو الـ node المحرر كان cached. Slow path — ما بتستخدم الـ rcache.
- **Who calls it**: Error paths أو الـ drivers اللي ما بتستخدم fast path.

---

#### `__free_iova`

```c
void __free_iova(struct iova_domain *iovad, struct iova *iova);
```

نسخة أسرع من `free_iova` — بتاخد الـ `struct iova *` مباشرة بدل البحث بالـ PFN. بتشيل الـ node من الـ rbtree وبترجعه للـ slab.

- **Parameters**:
  - `iovad`: الـ domain.
  - `iova`: الـ pointer المباشر للـ struct.
- **Return**: `void`.
- **Key details**: بتمسك الـ `iova_rbtree_lock`. المستدعي مسؤول عن صحة الـ pointer.
- **Who calls it**: Code اللي عنده الـ `struct iova *` موجود (مثل `reserve_iova` cleanup، أو بعد `alloc_iova`).

---

#### `free_iova_fast`

```c
void free_iova_fast(struct iova_domain *iovad,
                    unsigned long pfn,
                    unsigned long size);
```

الـ **fast-path deallocator**. بدل ما ترجع الـ IOVA فوراً للـ rbtree، بتخزّنها في الـ per-CPU rcache. لو الـ rcache امتلأ، بتسرّب batch منه للـ rbtree.

- **Parameters**:
  - `iovad`: الـ domain.
  - `pfn`: الـ `pfn_lo` للـ IOVA المراد تحريرها.
  - `size`: عدد الـ pages.
- **Return**: `void`.
- **Key details**: Lock-free للـ rcache نفسه (per-CPU access). الـ drain للـ rbtree بيحتاج الـ lock. الـ `size` بيُستخدم لتحديد الـ bucket المناسب في الـ rcache.
- **Who calls it**: `iommu_dma_free_iova` — الـ hot path لكل DMA unmap.

---

### Group 4: Reservation & Lookup

---

#### `reserve_iova`

```c
struct iova *reserve_iova(struct iova_domain *iovad,
                           unsigned long pfn_lo,
                           unsigned long pfn_hi);
```

بتحجز range محدد في الـ IOVA space وبتمنع تخصيصه لأي شخص آخر. الـ use case الرئيسي هو حجز الـ identity-mapped regions أو الـ MSI address spaces.

- **Parameters**:
  - `iovad`: الـ domain.
  - `pfn_lo`: بداية الـ range (شامل).
  - `pfn_hi`: نهاية الـ range (شامل).
- **Return**: `struct iova *` للـ reserved node، أو `NULL` عند الفشل.
- **Key details**: بتمسك الـ `iova_rbtree_lock`. لو الـ range متعارض مع IOVA موجودة، بترجع `NULL`. الـ reserved IOVAs بتظهر في الـ rbtree زي أي IOVA عادية.
- **Who calls it**: IOMMU setup code لحجز الـ MMIO regions، مثل `iommu_dma_get_resv_regions`.

---

#### `find_iova`

```c
struct iova *find_iova(struct iova_domain *iovad, unsigned long pfn);
```

بتدوّر في الـ rbtree على الـ `struct iova` اللي بتحتوي على الـ PFN المعطى (يعني `pfn_lo <= pfn <= pfn_hi`).

- **Parameters**:
  - `iovad`: الـ domain.
  - `pfn`: الـ PFN المراد إيجاده.
- **Return**: `struct iova *` إذا وُجد، وإلا `NULL`.
- **Key details**: بتمسك الـ `iova_rbtree_lock`. O(log n) بحكم الـ rbtree. بتُستخدم داخلياً من `free_iova`.
- **Who calls it**: `free_iova`، وأي code بحاجة للـ reverse lookup من PFN لـ IOVA object.

---

### Group 5: Cache Management

---

#### `iova_rcache_range`

```c
unsigned long iova_rcache_range(void);
```

بترجع أقصى IOVA size (بالـ pages) ممكن تُخزَّن في الـ rcache. الـ sizes الأكبر من القيمة دي بتروح مباشرة للـ rbtree.

- **Parameters**: لا يوجد.
- **Return**: الـ maximum cacheable size كـ page count.
- **Key details**: القيمة ثابتة وبتتحدد من عدد الـ rcache buckets المعرّفة compile-time.
- **Who calls it**: IOMMU code اللي بيقرر هل يستخدم fast path أو لا.

---

### Group 6: Inline Helper Functions

الـ helpers دي كلها `static inline` — بتتحوّل لـ zero-overhead code في الغالب.

---

#### `iova_size`

```c
static inline unsigned long iova_size(struct iova *iova)
```

بترجع حجم الـ IOVA بالـ granule units: `pfn_hi - pfn_lo + 1`. النتيجة بتمثّل عدد الـ pages اللي الـ IOVA بتغطيها.

---

#### `iova_shift`

```c
static inline unsigned long iova_shift(struct iova_domain *iovad)
```

بتحسب `__ffs(iovad->granule)` — يعني بترجع عدد الـ bits اللي لازم تعمل shift عشان تحوّل من PFN لعنوان. لو الـ `granule = PAGE_SIZE = 4096`، الـ shift = 12.

---

#### `iova_mask`

```c
static inline unsigned long iova_mask(struct iova_domain *iovad)
```

بترجع `granule - 1`، وهي الـ bitmask للـ offset داخل الـ granule. مثال: `granule=4096` → `mask=0xFFF`.

---

#### `iova_offset`

```c
static inline size_t iova_offset(struct iova_domain *iovad, dma_addr_t iova)
```

بتحسب الـ byte offset لعنوان IOVA داخل الـ granule الخاص فيه: `iova & iova_mask(iovad)`. مفيدة للـ scatter-gather operations.

---

#### `iova_align`

```c
static inline size_t iova_align(struct iova_domain *iovad, size_t size)
```

بتعمل round-up للـ `size` لأقرب مضاعف للـ `granule` باستخدام `ALIGN(size, iovad->granule)`. لازم تتعمل على الـ size قبل ما تنادي `alloc_iova`.

---

#### `iova_align_down`

```c
static inline size_t iova_align_down(struct iova_domain *iovad, size_t size)
```

عكس `iova_align` — round-down لأقرب مضاعف للـ `granule`. مفيدة لحساب الـ base address.

---

#### `iova_dma_addr`

```c
static inline dma_addr_t iova_dma_addr(struct iova_domain *iovad,
                                        struct iova *iova)
```

بتحوّل الـ `pfn_lo` لـ `dma_addr_t`: `(dma_addr_t)iova->pfn_lo << iova_shift(iovad)`. هاي القيمة هي اللي بتنعطي للـ device في الـ DMA descriptor.

---

#### `iova_pfn`

```c
static inline unsigned long iova_pfn(struct iova_domain *iovad, dma_addr_t iova)
```

التحويل العكسي: من DMA address لـ PFN بواسطة right-shift: `iova >> iova_shift(iovad)`. بتُستخدم في الـ unmap paths.

---

### العلاقة بين الـ Functions — Data Flow

```
Driver calls iommu_dma_map_page()
        │
        ▼
alloc_iova_fast(iovad, size, limit_pfn, flush)
        │
        ├──[rcache hit]──► returns pfn directly (fast, lock-free)
        │
        └──[rcache miss]──► alloc_iova(iovad, size, limit_pfn, aligned)
                                    │
                                    ├── kmem_cache_zalloc(iova_cache)
                                    ├── spin_lock(iova_rbtree_lock)
                                    ├── rbtree search (top-down)
                                    ├── rb_insert()
                                    └── spin_unlock → returns struct iova*

        │
        ▼
iova_dma_addr(iovad, iova) → dma_addr_t → given to device

... DMA operation completes ...

Driver calls iommu_dma_unmap_page()
        │
        ▼
free_iova_fast(iovad, pfn, size)
        │
        ├──[rcache not full]──► store in per-CPU bucket (lock-free)
        │
        └──[rcache full]──► drain batch to rbtree via __free_iova()
                                    │
                                    ├── spin_lock(iova_rbtree_lock)
                                    ├── rb_erase()
                                    └── kmem_cache_free(iova_cache)
```

---

### ملاحظات على الـ Locking Model

| Operation | Lock Required | Type |
|---|---|---|
| `alloc_iova` | نعم | `spin_lock_irqsave(iova_rbtree_lock)` |
| `__free_iova` | نعم | `spin_lock_irqsave(iova_rbtree_lock)` |
| `find_iova` | نعم | `spin_lock_irqsave(iova_rbtree_lock)` |
| `reserve_iova` | نعم | `spin_lock_irqsave(iova_rbtree_lock)` |
| `alloc_iova_fast` (rcache hit) | لا | per-CPU access |
| `free_iova_fast` (rcache not full) | لا | per-CPU access |

الـ `spinlock` المستخدم هو `iova_rbtree_lock` داخل `struct iova_domain`. الـ rcache paths بتتجنب هاد الـ lock تماماً لما تكون في الـ happy path، وهذا السبب الرئيسي وراء الـ performance gain.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

الـ IOVA subsystem مش بيعمل entries خاصة بيه في debugfs بشكل مباشر، لكن الـ IOMMU اللي بيستخدمه بيعمل كده.

```bash
# mount debugfs لو مش متعملش
mount -t debugfs none /sys/kernel/debug

# IOMMU groups وdomain info (Intel VT-d مثلاً)
ls /sys/kernel/debug/iommu/
cat /sys/kernel/debug/iommu/domain_pools   # لو موجود في الـ driver

# Intel IOMMU specific
ls /sys/kernel/debug/iommu/intel/
# بيظهر دومينات وعدد الـ IOVAs المخصصة لكل domain
```

**المهم:** الـ `iova_domain` نفسه محفوظ في memory الـ driver — مش مكشوف مباشرة في debugfs. لازم تضيف debug entries يدوياً في الـ driver المستخدم.

---

#### 2. sysfs entries

```bash
# IOMMU groups
ls /sys/kernel/iommu_groups/
ls /sys/kernel/iommu_groups/0/devices/

# لكل device: هل بيستخدم IOMMU؟
cat /sys/bus/pci/devices/0000:01:00.0/iommu_group/type
# القيمة: DMA / DMA-FQ / identity / blocked

# granule الـ IOMMU (page size)
cat /sys/bus/platform/drivers/arm-smmu/*/iommu-pages   # بعض الـ drivers

# حالة rcache (مش موجود في sysfs افتراضياً)
# لكن عدد الـ IOVAs المستخدمة ممكن تتبعه عبر tracing
```

---

#### 3. ftrace — tracepoints وevents

```bash
# اعرف الـ events المتاحة للـ IOMMU
grep -r iommu /sys/kernel/tracing/available_events

# فعّل events الـ IOMMU map/unmap
echo 1 > /sys/kernel/tracing/events/iommu/map/enable
echo 1 > /sys/kernel/tracing/events/iommu/unmap/enable
echo 1 > /sys/kernel/tracing/events/iommu/io_page_fault/enable

# فعّل tracing
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe

# تتبع دوال الـ IOVA تحديداً بـ function filter
echo alloc_iova_fast > /sys/kernel/tracing/set_ftrace_filter
echo free_iova_fast  >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
cat /sys/kernel/tracing/trace_pipe
```

**مثال output:**

```
kworker/0:1-42    [000] .... alloc_iova_fast+0x0/0x80 [iova]
kworker/0:1-42    [000] .... free_iova_fast+0x0/0x60 [iova]
```

---

#### 4. printk / dynamic debug

```bash
# تفعيل الـ dynamic debug للـ iommu module
echo "module iommu +p" > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل debug لملف iova.c تحديداً
echo "file drivers/iommu/iova.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ output
dmesg -w | grep -i iova

# تفعيل كل IOMMU debug messages
echo "module intel_iommu +p" > /sys/kernel/debug/dynamic_debug/control
```

---

#### 5. Kernel Config options للـ debugging

| Option | الوظيفة |
|--------|---------|
| `CONFIG_IOMMU_IOVA` | يفعّل الـ IOVA subsystem نفسه |
| `CONFIG_IOMMU_DEBUG` | يفعّل رسائل debug في IOMMU |
| `CONFIG_IOMMU_DEBUGFS` | يضيف entries في debugfs للـ IOMMU |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف lock misuse في `iova_rbtree_lock` |
| `CONFIG_LOCKDEP` | يحلل lock ordering ويكتشف deadlocks |
| `CONFIG_SLUB_DEBUG` | يكتشف use-after-free في `iova_cache` |
| `CONFIG_KASAN` | يكتشف heap corruption في الـ `struct iova` |
| `CONFIG_KMEMCHECK` | يتحقق من memory access صح |
| `CONFIG_DEBUG_KMEMLEAK` | يكتشف memory leaks في الـ IOVA allocations |
| `CONFIG_DMA_API_DEBUG` | يتحقق من صحة استخدام DMA API فوق الـ IOVA |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "IOMMU|IOVA|DMA_API_DEBUG"
```

---

#### 6. devlink وأدوات خاصة بالـ subsystem

```bash
# iommufd - الـ interface الجديد (kernel 6.0+)
ls /dev/iommu   # الـ character device

# استخدام iommufd عبر الـ userspace tools
# مثلاً vfio-pci + iommufd
ls /sys/bus/pci/drivers/vfio-pci/

# devlink للـ NICs التي تستخدم IOVA عبر RDMA
devlink dev show
devlink dev info pci/0000:01:00.0

# للتحقق من DMA mapping errors في NIC
ethtool -S eth0 | grep -i dma

# intel_iommu خاص
cat /sys/class/iommu/dmar0/intel-iommu/cap
cat /sys/class/iommu/dmar0/intel-iommu/ecap
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|------------|--------|------|
| `alloc_iova: Out of IOVA space` | الـ IOVA domain امتلأ | زوّد `dma_32bit_pfn` أو خفض الـ DMA allocations |
| `iommu: Failed to allocate iova` | فشل `alloc_iova_fast` بعد flush الـ rcache | تحقق من تسريب IOVAs — leak في `free_iova_fast` |
| `BUG: spinlock lockup` على `iova_rbtree_lock` | thread محجوز على الـ lock | ابحث عن interrupt handler بيطلب الـ lock بدون `irqsave` |
| `WARN_ON(1)` في `iova_insert_rbtree` | محاولة insert IOVA بـ `pfn_lo` مكرر | double-allocation — تتبع الـ caller |
| `DMA-API: mapping error` | الـ IOMMU رفض الـ mapping | تحقق من الـ IOVA range مش محجوز بـ `reserve_iova` |
| `iommu: io_page_fault` | device وصل لعنوان مش mapped | الـ IOVA اتحررت قبل إن الـ device يخلص |
| `BUG_ON` في `init_iova_domain` | granule أكبر من PAGE_SIZE أو مش power of 2 | صلّح الـ granule المبرر في `init_iova_domain()` |
| `kernel BUG at lib/rbtree.c` | الـ rbtree اتكسر بسبب corruption | غالباً use-after-free في `struct iova` — فعّل KASAN |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في __alloc_and_insert_iova_range — لما يفشل الـ alloc */
if (ret == -ENOMEM) {
    WARN_ON(1); /* طباعة backtrace عشان نعرف مين طلب الـ alloc */
    dump_stack();
}

/* في iova_insert_rbtree — الـ WARN_ON موجود أصلاً */
WARN_ON(1); /* duplicate pfn_lo — تحقق من الـ caller */

/* في free_iova_mem — تأكد من إن مش بنحرر الـ anchor */
if (WARN_ON(iova->pfn_lo == IOVA_ANCHOR))
    return;

/* في __cached_rbnode_delete_update — تتبع الـ cache corruption */
WARN_ON(!iovad->cached_node); /* الـ node المحفوز NULL */
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

```bash
# تحقق من إن الـ IOMMU Hardware مفعّل فعلاً
dmesg | grep -i "DMAR\|IOMMU\|smmu\|iova"

# Intel VT-d
dmesg | grep DMAR
# لازم تشوف: "DMAR: Intel(R) Virtualization Technology for Directed I/O"

# ARM SMMU
dmesg | grep smmu
# لازم تشوف: "arm-smmu: probing hardware"

# تأكد من إن الـ BIOS فعّل VT-d
cat /proc/cpuinfo | grep -i vmx  # Intel
# أو تفتح BIOS وتتحقق من Intel VT-d / AMD-Vi

# قارن الـ IOVA granule مع الـ IOMMU page size
# granule لازم يكون <= PAGE_SIZE وpower of 2
getconf PAGE_SIZE  # عادة 4096
```

---

#### 2. Register Dump techniques

```bash
# قراءة DMAR registers (Intel) عبر /dev/mem
# عنوان الـ DMAR MMIO من ACPI tables
cat /sys/firmware/acpi/tables/DMAR | xxd | head -40

# استخدام devmem2 لقراءة IOMMU registers
# أولاً: اعرف العنوان الفيزيائي من dmesg
dmesg | grep "DMAR.*BIOS.*0x"
# مثال: DMAR: DRHD base: 0xfed90000 flags: 0x0

# اقرأ registers
devmem2 0xfed90000 w   # Version register
devmem2 0xfed90008 q   # Capability register
devmem2 0xfed90010 q   # Extended capability register

# ARM SMMU registers (من DT أو platform info)
devmem2 0x09000000 w   # SMMU_IDR0
devmem2 0x09000004 w   # SMMU_IDR1

# لو devmem2 مش متاح، استخدم /dev/mem مع Python
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xfed90000)
    cap = struct.unpack('<Q', m[8:16])[0]
    print(f'DMAR CAP: {cap:#018x}')
    m.close()
"
```

---

#### 3. Logic Analyzer / Oscilloscope tips

لما بتشتغل على IOMMU issues على مستوى الـ bus:

```
PCIe Bus:
  ┌─────────┐    TLP (Transaction Layer Packet)    ┌──────────┐
  │  Device │ ──────────────────────────────────▶  │  IOMMU   │
  │ (NIC/GPU│    Address في الـ TLP = IOVA          │ HW Block │
  └─────────┘                                       └──────────┘
                                                        │
                                              تحويل IOVA → PA
                                                        │
                                                  ┌─────▼────┐
                                                  │   DRAM   │
                                                  └──────────┘
```

**نقاط المراقبة:**
- راقب الـ **AXI/PCIe transactions** على الـ bus قبل الـ IOMMU وبعده
- قارن الـ address قبل التحويل (IOVA) وبعده (Physical Address)
- ابحث عن `Unsupported Request (UR)` completion — دليل على IOVA fault
- راقب سيگنال **IOMMU fault interrupt** (عادة MSI)

```bash
# تتبع IOMMU fault interrupts
cat /proc/interrupts | grep -i dmar
# أو
watch -n 1 "cat /proc/interrupts | grep DMAR"
# لو العداد بيزيد = في faults مستمرة
```

---

#### 4. Hardware issues شائعة وpatterns في kernel log

| المشكلة الـ HW | Pattern في dmesg | التفسير |
|---------------|-----------------|---------|
| IOMMU مش مفعّل في BIOS | لا يوجد `DMAR:` أو `smmu: probing` | فعّل VT-d/AMD-Vi في BIOS |
| Device بيعمل DMA على عنوان غلط | `DMAR: DRHD: handling fault` | bug في الـ driver — بيحرر IOVA قبل DMA ينتهي |
| IOMMU page size mismatch | `BUG_ON` في `init_iova_domain` | granule مش متوافق مع HW page size |
| TLB مش بيتعمل flush | device بيقرأ data قديمة | مشكلة في `iommu_flush_tlb` — تحقق من IOMMU driver |
| Interrupt remapping fault | `DMAR: fault addr` مع interrupt vector | مشكلة في interrupt remapping table |
| ATS (Address Translation Services) bug | `iommu: device ... is broken` | PCIe ATS reporting wrong translation |

```bash
# استخراج كل IOMMU faults من الـ log
dmesg | grep -E "fault|DMAR|iommu.*error|DMA.*error" | head -50

# تفاصيل Intel DMAR fault
dmesg | grep "DMAR:"
# مثال: DMAR: DRHD: handling fault, Fault address: 0x1234000,
#        Fault Reason: 6 (non-zero reserved field set in page-table entry)
```

**Fault Reason codes (Intel VT-d):**

| Code | المعنى |
|------|--------|
| 1 | الـ root-entry مش present |
| 2 | الـ context-entry مش present |
| 6 | reserved bits مضبوطة في page table |
| 12 | الـ IOVA مش موجود في page table — أشهر سبب |

---

#### 5. Device Tree debugging

```bash
# تحقق من الـ DT للـ SMMU
cat /proc/device-tree/iommu@9000000/compatible
# المفروض: "arm,smmu-v3" أو "arm,mmu-500" إلخ

# تحقق من الـ iommu-map في الـ PCIe node
fdtdump /boot/dtb-$(uname -r) 2>/dev/null | grep -A5 "iommu-map"

# أو من sysfs
find /proc/device-tree -name "iommus" | while read f; do
    echo "=== $f ==="
    xxd "$f"
done

# تحقق من الـ phandle يشاور على الـ SMMU الصح
# iommu-map = <PCI_RID IOMMU_PHANDLE SID SIZE>
cat /proc/device-tree/pcie@*/iommu-map | xxd

# مقارنة الـ granule في DT مع ما يقوله الـ kernel
dmesg | grep "page size"
# ARM SMMU: "arm-smmu-v3: Supported page sizes: 0x61310000"
# التحويل: bit 12 = 4KB, bit 16 = 64KB, bit 29 = 512MB
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# iova-debug.sh — سكريبت debugging شامل للـ IOVA subsystem

echo "=== IOMMU Status ==="
dmesg | grep -iE "iommu|dmar|smmu" | tail -30

echo ""
echo "=== IOMMU Groups ==="
for g in /sys/kernel/iommu_groups/*/; do
    echo -n "Group $(basename $g): "
    ls "$g/devices/" 2>/dev/null | tr '\n' ' '
    echo ""
done

echo ""
echo "=== IOMMU Fault Interrupts ==="
cat /proc/interrupts | grep -iE "dmar|smmu|iommu"

echo ""
echo "=== DMA API Debug (if enabled) ==="
dmesg | grep "DMA-API" | tail -20

echo ""
echo "=== Kernel Config ==="
zcat /proc/config.gz 2>/dev/null | grep -E "CONFIG_IOMMU|CONFIG_IOVA|CONFIG_DMA_API"

echo ""
echo "=== Current IOVA rcache range ==="
# لو الـ module مكشوف via debugfs
cat /sys/kernel/debug/iommu/iova_rcache 2>/dev/null || echo "Not exposed in debugfs"
```

```bash
# تفعيل ftrace كامل للـ IOVA
cd /sys/kernel/tracing
echo 0 > tracing_on
echo function > current_tracer
echo "alloc_iova alloc_iova_fast free_iova free_iova_fast \
      reserve_iova find_iova put_iova_domain init_iova_domain" \
     > set_ftrace_filter
echo 1 > tracing_on
sleep 10
echo 0 > tracing_on
cat trace | head -100
```

```bash
# كشف memory leak في الـ IOVA allocations
# فعّل kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep -A5 "iova"
```

```bash
# مراقبة IOMMU faults في real-time
watch -n 0.5 'dmesg | grep -c "fault\|DMAR.*fault\|io_page_fault"'

# أو عبر perf
perf stat -e iommu:map,iommu:unmap,iommu:io_page_fault \
    -a sleep 5
```

#### تفسير الـ Output

```
# مثال output من ftrace:
#   swapper/0-0     [000] d... alloc_iova_fast+0x5/0x80
#   swapper/0-0     [000] d...   alloc_iova+0x20/0x90
#   swapper/0-0     [000] d...     __alloc_and_insert_iova_range+0x30/0x120

# التفسير:
# - "d..." = interrupts disabled — طبيعي لأننا جوه spinlock
# - alloc_iova_fast فشل في rcache ووصل لـ alloc_iova الأبطأ
# - لو الـ chain بتظهر كتير جداً = rcache مش شغال كويس

# مثال DMAR fault:
# DMAR: DRHD: handling fault, iommu: 0, device: 01:00.0,
#       fault addr: ffff8800deadbeef, PASID: 00000000,
#       fault reason: 12, fault type: 2 (Device)
# التفسير:
# - device 01:00.0 وصل لعنوان ffff8800deadbeef
# - reason 12 = الـ IOVA مش موجود في الـ page table
# - الحل: تحقق من إن الـ driver مش بيعمل DMA بعد ما حرر الـ IOVA
```

```bash
# اختبار بسيط: هل الـ IOVA domain شغال؟
# عبر test module أو بـ vfio
modprobe vfio-pci
echo "0000:01:00.0" > /sys/bus/pci/drivers/vfio-pci/bind
dmesg | tail -20
# لو شغال هتشوف: "vfio-pci: add [1234:5678]" بدون errors
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — DMA Hang بسبب `limit_pfn` غلط

#### العنوان
**الـ IOMMU domain بيعمل hang لأن `alloc_iova_fast` بترجع 0 طول الوقت**

#### السياق
بتشتغل على industrial gateway بيستخدم **RK3562** مع Ethernet controller بيعمل DMA لنقل بيانات Modbus من sensors. الـ kernel config فيه `CONFIG_IOMMU_IOVA=y` والـ IOMMU مفعّل. الـ product اتشحن وبعد أسبوع بدأ الـ Ethernet driver يعمل crash بشكل عشوائي.

#### المشكلة
```
kernel: iommu: Failed to allocate iova
kernel: dwmac-rk: DMA mapping failed, stopping device
```
الـ network interface بيوقف ونفسه ما بيرجع غير بعد reboot.

#### التحليل
الكود في `iova.h` بيعرّف:

```c
unsigned long alloc_iova_fast(struct iova_domain *iovad, unsigned long size,
                              unsigned long limit_pfn, bool flush_rcache);
```

الـ driver بيبعت `limit_pfn = iovad->dma_32bit_pfn` بغض النظر عن الـ physical memory layout. على RK3562 الـ RAM ابتدت من `0x80000000` (2GB offset)، فالـ `dma_32bit_pfn` بيحسب غلط — بييجي أصغر من `start_pfn`.

لما `limit_pfn < start_pfn`:
- الـ `alloc_iova_fast` بترجع `0` فوراً
- الـ `max32_alloc_size` field في `iova_domain` بيتحدّث لقيمة كبيرة
- الطلبات الجديدة كلها بتفشل لأن الكود بيتجنب retry بعد ما `max32_alloc_size` اتحدّث

#### الحل
**تحقق من الـ `start_pfn` الصح عند init:**

```c
/* في كود الـ driver قبل ما يستدعي init_iova_domain */
unsigned long start = iommu_dma_get_merge_base(domain);
unsigned long start_pfn = max_t(unsigned long,
                                1,  /* never 0 */
                                start >> PAGE_SHIFT);

init_iova_domain(&iovad, PAGE_SIZE, start_pfn);
```

وتحقق من الـ DT node:
```bash
# تحقق إن dma-ranges موجود وصح
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "dma-ranges"
```

#### الدرس المستفاد
**الـ `start_pfn` في `iova_domain` لازم يتطابق مع الـ physical DMA window الحقيقية للـ SoC.** غلطة واحدة في الـ init بتخلي كل الـ `alloc_iova_fast` calls تفشل صامتة.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تسريب IOVA بسبب `free_iova_fast` ما اتستدعاش

#### العنوان
**الـ GPU driver بيأكل كل الـ IOVA space خلال ساعات وبعدين بيموت**

#### السياق
Android TV Box بيشغّل Mali-G31 GPU على **Allwinner H616**. بعد كام ساعة streaming، الـ system بيبطّأ جداً وبتلاقي في logcat:

```
IOMMU: no free iova in domain
mali: dma_map_page failed
SurfaceFlinger: failed to allocate buffer
```

#### المشكلة
الـ IOVA space (32-bit, ~4GB) بيخلص تدريجياً. كل buffer بيتعمله map ما بيتعمله unmap صح.

#### التحليل
الـ header بيوفر اتنين طريقة لتحرير الـ IOVA:

```c
void free_iova(struct iova_domain *iovad, unsigned long pfn);       /* slow path */
void free_iova_fast(struct iova_domain *iovad, unsigned long pfn,
                    unsigned long size);                             /* fast path via rcache */
```

الـ rcache (represented بـ `struct iova_rcache *rcaches` في `iova_domain`) بيخزن الـ free IOVAs في per-CPU cache لأداء أحسن. لو الـ driver بيستدعي `free_iova` بس من غير `free_iova_fast`، الـ rcache ما بيتملاش، ومع الوقت الـ allocations بتفضل تاخد من الـ rbtree وما بترجعش.

المشكلة الأدق: الـ Mali driver كان بيستدعي `alloc_iova_fast` للـ allocation لكن بيستدعي `__free_iova` مباشرة للـ deallocation — الـ size الـ rcache مش بيعرفه.

```c
/* غلط: bypass الـ rcache */
__free_iova(iovad, iova);

/* صح: */
free_iova_fast(iovad, iova->pfn_lo, iova_size(iova));
```

#### الحل
```bash
# اتحقق من الـ IOVA leak في runtime
cat /sys/kernel/debug/iommu/mali-iommu/iova_map

# او count الـ nodes في rbtree (لو ارتفع بمرور الوقت = leak)
grep -r "iova" /sys/kernel/debug/iommu/
```

Fix في الـ driver:
```c
/* لما بتعمل free، لازم تمرر الـ size عشان الـ rcache يشتغل */
size_t size = iova_size(iova);           /* pfn_hi - pfn_lo + 1 */
free_iova_fast(iovad, iova->pfn_lo, size);
```

#### الدرس المستفاد
**`alloc_iova_fast` و`free_iova_fast` لازم يتزاوجوا.** لو استخدمت الـ fast allocator ورجعت بـ slow free، الـ rcache بيبقى فاضي وبيعمل IOVA exhaustion مع الوقت.

---

### السيناريو 3: IoT Sensor Hub على STM32MP1 — `granule` غلط بيكسر الـ alignment

#### العنوان
**الـ SPI DMA بيعمل data corruption على STM32MP1 بسبب IOVA misalignment**

#### السياق
IoT device بيجمع بيانات من 8 SPI sensors بيعمل DMA transfer لـ flash storage. على **STM32MP157** مع IOMMU مفعّل، البيانات المتخزنة corrupted — كل packet فيه garbage في آخره.

#### المشكلة
الـ SPI driver بيطلب buffers بحجم 63 bytes. الـ DMA بيشتغل، لكن الـ IOMMU بيمرر access لـ extra bytes جوا الـ page التالية.

#### التحليل
الـ `iova_align` inline function:

```c
static inline size_t iova_align(struct iova_domain *iovad, size_t size)
{
    return ALIGN(size, iovad->granule);
}
```

لو `granule = 64` (صح للـ SPI controller ده)، فـ 63 bytes بيتحولوا لـ 64 bytes — تمام.

لكن المشكلة كانت إن `init_iova_domain` اتستدعى بـ `granule = 4096` (PAGE_SIZE) بدل ما يتطابق مع الـ SPI controller's DMA alignment requirement (64 bytes).

```c
/* غلط: granule كبير جداً */
init_iova_domain(&spi_iovad, PAGE_SIZE, start_pfn);

/* صح: granule يتطابق مع الـ controller */
init_iova_domain(&spi_iovad, SPI_DMA_ALIGN, start_pfn);
```

الـ `iova_offset` function:
```c
static inline size_t iova_offset(struct iova_domain *iovad, dma_addr_t iova)
{
    return iova & iova_mask(iovad);  /* granule - 1 */
}
```

بـ granule كبير غلط، الـ offset حسابه غلط، والـ DMA engine بيبدأ من عنوان غلط جوا الـ page.

#### الحل
```c
/* في الـ SPI driver init */
#define STM32_SPI_DMA_GRANULE   64UL  /* من datasheet */

init_iova_domain(&priv->iovad,
                 STM32_SPI_DMA_GRANULE,
                 1UL /* start_pfn */);
```

تحقق:
```bash
# اتحقق من الـ granule المستخدم
cat /sys/kernel/debug/iommu/domains/*/iova_granule 2>/dev/null
```

#### الدرس المستفاد
**الـ `granule` في `iova_domain` لازم يتطابق مع متطلبات الـ hardware DMA alignment الحقيقية.** استخدام `PAGE_SIZE` دايماً مش صح — بعض الـ controllers بتحتاج granularity أصغر.

---

### السيناريو 4: Automotive ECU على i.MX8QM — `reserve_iova` ما اتستدعاش وبيحصل overlap مع firmware

#### العنوان
**الـ IOMMU بيعمل fault لأن الـ GPU driver بيحجز IOVA فوق منطقة الـ firmware**

#### السياق
Automotive ECU بيشغّل ADAS workloads على **i.MX8QM** مع إن الـ firmware (SCU) محجوز في physical memory من `0x00000000` لـ `0x1FFFFFFF`. الـ GPU IOMMU domain بيعمل fault بشكل عشوائي أثناء inference.

#### المشكلة
```
arm-smmu 5000000.iommu: Unhandled context fault for GPU
iova: 0x10000000 maps to reserved firmware region
```

#### التحليل
الـ `iova_domain` struct فيه:

```c
struct iova  anchor; /* rbtree lookup anchor */
```

وفي الـ API:

```c
struct iova *reserve_iova(struct iova_domain *iovad,
                           unsigned long pfn_lo,
                           unsigned long pfn_hi);
```

الـ `reserve_iova` بتحجز range في الـ rbtree عشان `alloc_iova_fast` ما تيجيش توديها. المشكلة إن الـ GPU driver ما استدعاش `reserve_iova` للـ range بتاع الـ firmware قبل ما يبدأ allocations.

الـ allocator — لأنه بيشتغل من أعلى لأسفل (high to low addresses) باستخدام `cached_node` و`cached32_node` — وصل في النهاية للـ addresses المحجوزة للـ firmware بعد ما خلّص الـ high addresses.

```c
/* لازم يتعمل فور بعد init_iova_domain */
init_iova_domain(&gpu_iovad, PAGE_SIZE, 1UL);

/* احجز منطقة الـ SCU firmware */
reserve_iova(&gpu_iovad,
             0x00000000 >> PAGE_SHIFT,
             0x1FFFFFFF >> PAGE_SHIFT);
```

#### الحل
```c
/* في الـ driver probe */
static int imx8_gpu_iommu_init(struct device *dev,
                                struct iova_domain *iovad)
{
    init_iova_domain(iovad, PAGE_SIZE, 1);

    /* احجز الـ firmware regions من الـ DT */
    struct resource *fw_res;
    fw_res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "scu-fw");
    if (fw_res)
        reserve_iova(iovad,
                     fw_res->start >> PAGE_SHIFT,
                     fw_res->end >> PAGE_SHIFT);
    return iova_domain_init_rcaches(iovad);
}
```

في الـ DT:
```dts
gpu@1000000 {
    memory-region = <&scu_fw_reserved>;
    /* ... */
};
```

#### الدرس المستفاد
**أي منطقة محجوزة في الـ physical memory (firmware, carveout, secure world) لازم تتحجز في الـ `iova_domain` بـ `reserve_iova` قبل أي allocation.** الـ allocator ما بيعرفش بالـ physical layout لوحده.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `iova_cache_get` ما اتستدعاش وبيطلع NULL pointer

#### العنوان
**الـ USB driver بيـ crash في أول DMA transfer على board جديد بـ AM62x**

#### السياق
فريق bring-up شغّال على custom industrial board بـ **TI AM625** (AM62x family) مع USB 3.0 host controller وكاميرا USB. الـ kernel بيبوت لكن أول لما الكاميرا بتتوصل:

```
BUG: kernel NULL pointer dereference at 0000000000000008
PC: alloc_iova+0x4c/0x180
```

#### المشكلة
الـ crash بييجي في أول استدعاء لـ `alloc_iova` قبل ما الـ iova slab cache يتعمل.

#### التحليل
الـ header بيوضح:

```c
int iova_cache_get(void);   /* يجب استدعاؤها قبل أي alloc */
void iova_cache_put(void);  /* يجب استدعاؤها عند cleanup */
```

الـ `iova_cache_get` بتعمل `kmem_cache_create` للـ `struct iova` objects. لو ما اتستدعتش، الـ `alloc_iova` بتحاول تـ allocate من NULL cache pointer وبيحصل crash.

الـ USB driver الجديد — اللي اتكتب سريع للـ bring-up — فيه:

```c
/* غلط: skip الـ cache init */
static int am62x_usb_probe(struct platform_device *pdev)
{
    init_iova_domain(&priv->iovad, PAGE_SIZE, 1);
    /* ... بدأ يستخدم alloc_iova مباشرة */
}
```

بدل:

```c
/* صح */
static int am62x_usb_probe(struct platform_device *pdev)
{
    int ret = iova_cache_get();  /* لازم أول حاجة */
    if (ret)
        return ret;

    init_iova_domain(&priv->iovad, PAGE_SIZE, 1);
    ret = iova_domain_init_rcaches(&priv->iovad);
    if (ret)
        goto err_cache;
    /* ... */

err_cache:
    iova_cache_put();
    return ret;
}

static int am62x_usb_remove(struct platform_device *pdev)
{
    put_iova_domain(&priv->iovad);
    iova_cache_put();   /* لازم في الـ cleanup */
    return 0;
}
```

#### الحل
الترتيب الصح لأي driver بيستخدم IOVA:

```
1. iova_cache_get()           ← سlab cache creation
2. init_iova_domain()         ← rbtree + locks init
3. reserve_iova() [اختياري]  ← حجز المناطق المحجوزة
4. iova_domain_init_rcaches() ← per-CPU rcache init
--- استخدام الـ domain ---
5. put_iova_domain()          ← تحرير كل الـ IOVAs
6. iova_cache_put()           ← تحرير الـ slab cache
```

تحقق سريع:
```bash
# اتحقق إن الـ iova slab موجود
cat /proc/slabinfo | grep iova

# المتوقع:
# iova-domain        128   128    128  32    1 ...
```

#### الدرس المستفاد
**`iova_cache_get` و`iova_cache_put` هما الـ lifecycle المسؤولين عن الـ slab allocator للـ `struct iova`.** أي driver بيستخدم IOVA مباشرة (مش عن طريق الـ IOMMU DMA API) لازم يستدعيهم صح في الـ probe/remove — ونسيانهم في الـ bring-up phase سبب رقم واحد لـ NULL pointer crashes.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأول لمتابعة تطور الـ Linux kernel. الـ articles دي غطّت الـ IOVA والـ IOMMU من زوايا مختلفة:

| المقال | الرابط | الأهمية |
|--------|--------|----------|
| A new DMA-mapping API | [lwn.net/Articles/1020437](https://lwn.net/Articles/1020437/) | بيشرح الـ API الجديد اللي بيخلّي الـ users يديروا الـ IOVA space بنفسهم |
| Dancing the DMA two-step | [lwn.net/Articles/997563](https://lwn.net/Articles/997563/) | شرح عميق لعملية الـ DMA mapping وعلاقتها بالـ IOVA |
| Use 1st-level for IOVA translation | [lwn.net/Articles/807079](https://lwn.net/Articles/807079/) | patchset لنقل الـ IOVA translation للـ 1st-level page table في Intel VT-d scalable mode |
| Use 1st-level for DMA remapping | [lwn.net/Articles/805870](https://lwn.net/Articles/805870/) | مكمّل للمقال اللي فوقه — الـ DMA remapping عبر Intel IOMMU |
| iommufd: Add nesting infrastructure | [lwn.net/Articles/948937](https://lwn.net/Articles/948937/) | بنية الـ IOVA range management في سياق الـ nested translation |
| iommufd: Enable noiommu mode for cdev | [lwn.net/Articles/1048880](https://lwn.net/Articles/1048880/) | آخر تطورات الـ iommufd وعلاقتها بالـ IOVA spaces |
| IOMMUFD kernel documentation (via LWN mirror) | [static.lwn.net/kerneldoc/userspace-api/iommufd.html](https://static.lwn.net/kerneldoc/userspace-api/iommufd.html) | الـ IOAS (IO Address Space) هو الاسم الجديد لمفهوم الـ IOVA domain |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في شجرة المصدر وعلى [docs.kernel.org](https://docs.kernel.org):

| الملف / الصفحة | الرابط | المحتوى |
|----------------|--------|----------|
| `Documentation/arch/x86/iommu.rst` | [docs.kernel.org/arch/x86/iommu.html](https://docs.kernel.org/arch/x86/iommu.html) | شرح الـ Intel IOMMU على x86 وحجز الـ IOVA ranges |
| `Documentation/core-api/dma-api-howto.rst` | [docs.kernel.org/core-api/dma-api-howto.html](https://docs.kernel.org/core-api/dma-api-howto.html) | الـ Dynamic DMA mapping guide — الأساس اللي بيتبنى عليه الـ IOVA |
| `Documentation/core-api/dma-api.rst` | [docs.kernel.org/core-api/dma-api.html](https://docs.kernel.org/core-api/dma-api.html) | الـ DMA API المرجعي الكامل |
| `Documentation/userspace-api/iommufd.rst` | [docs.kernel.org/userspace-api/iommufd.html](https://docs.kernel.org/userspace-api/iommufd.html) | الـ iommufd subsystem — المستقبل لإدارة الـ IOVA spaces |

---

### الكود المصدري المرجعي

الملفات دي هي اللي بتستخدم الـ `iova.h` مباشرةً أو بتشكّل السياق المحيط بيه:

| الملف | الوصف |
|-------|-------|
| `include/linux/iova.h` | الـ header موضوع الدراسة |
| `drivers/iommu/iova.c` | التنفيذ الفعلي لكل الـ functions المعلنة في الـ header |
| `drivers/iommu/dma-iommu.c` | المستخدم الرئيسي للـ IOVA domain في الـ DMA-IOMMU path — [GitHub](https://github.com/torvalds/linux/blob/master/drivers/iommu/dma-iommu.c) |
| `include/linux/iommu.h` | الـ IOMMU API العام المرتبط بالـ IOVA domains |

---

### نقاط الدخول في Git

للتتبع التاريخي للكود:

- **الـ iova.h على GitHub**: [github.com/torvalds/linux/blob/master/include/linux/iova.h](https://github.com/torvalds/linux/blob/master/include/linux/iova.h)
- **تاريخ الـ commits**: ابدأ بـ `git log --follow include/linux/iova.h` في شجرة الـ kernel
- **الـ blame للـ struct الأساسية**: `git blame include/linux/iova.h` — هيوريك إن الأصل من Intel (Anil S Keshavamurthy، 2006)
- **نقاط الدخول المهمة تاريخياً**:
  - إدخال الـ `iova_rcache` (per-CPU caching) — بحث عنها في `git log --grep="iova rcache"`
  - إدخال `iova_domain_init_rcaches` — ابحث عن `git log --grep="rcache init"`

---

### نقاشات Mailing List

| المصدر | الرابط | الموضوع |
|--------|--------|---------|
| linux-arm-kernel (narkive) | [IOVA allocation improvements for iommu-dma](https://linux-arm-kernel.infradead.narkive.com/3nKumNOt/patch-resend-0-3-iova-allocation-improvements-for-iommu-dma) | تحسينات الـ IOVA allocation مع الـ per-CPU caching |
| linux-kernel (narkive) | [iommu/dma: limit IOVA to dma-ranges](https://linux.kernel.narkive.com/c2qhVayP/patch-iommu-dma-limit-the-iova-allocated-to-dma-ranges-region) | تحديد الـ IOVA range بناءً على الـ dma-ranges |
| linux-arm-kernel (narkive) | [iommu/dma: PCI allocation optimisation](https://linux-arm-kernel.infradead.narkive.com/Bt9BVC2M/patch-v2-2-2-iommu-dma-implement-pci-allocation-optimisation) | تحسين الـ 32-bit IOVA allocation للـ PCI devices |

للمتابعة الحالية، اشترك في:
- `linux-iommu@lists.linux-foundation.org` — القائمة الرسمية للـ IOMMU subsystem
- `iommu@lists.linux-foundation.org`

---

### KernelNewbies.org

**الـ** kernelnewbies.org مش عنده صفحة مخصصة للـ IOVA، لكن الـ changelogs المفيدة:

- **Linux 5.14**: [kernelnewbies.org/Linux_5.14](https://kernelnewbies.org/Linux_5.14) — بيذكر تحسينات الـ IOVA fault handling في GPU subsystems
- **Linux 3.15 Drivers**: [kernelnewbies.org/Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) — تغييرات الـ IOMMU/DMA في تلك المرحلة
- **Linux 6.13**: [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) — آخر تطورات الـ iommufd

---

### eLinux.org

بحث elinux.org ما رجعش نتايج مباشرة عن الـ IOVA — الـ topic ده kernel-internal وأعمق من محتوى elinux المعتاد (embedded boards). بدل كده:

- الـ elinux مفيد للـ embedded context اللي بيستخدم IOMMU، زي ARM SMMU على boards زي BeagleBone أو i.MX
- ابحث على [elinux.org](https://elinux.org) بكلمة "SMMU" أو "DMA" للـ embedded context

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 15: Memory Mapping and DMA** — بيغطي الـ DMA coherent/streaming APIs اللي تحتها الـ IOVA
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- ملحوظة: الكتاب قديم (2005)، الـ IOVA rcache والـ iommufd مش موجودين فيه

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 12: Memory Management** — الأساس النظري للـ virtual address spaces
- **الفصل 13: The Virtual Filesystem** — context مفيد لفهم الـ domain abstraction
- الـ IOVA كـ subsystem مش مغطّي بشكل مباشر، لكن الكتاب بيبني الأساس اللازم

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14: Kernel Debugging Techniques** — مفيد لـ debug مشاكل الـ DMA/IOVA
- الـ IOMMU/IOVA مش مغطّي مباشرةً، الكتاب embedded-focused

#### مراجع إضافية مهمة
- **"An Introduction to IOMMU Infrastructure in the Linux Kernel"** — Lenovo Press whitepaper: [lenovopress.lenovo.com/lp1467](https://lenovopress.lenovo.com/lp1467-an-introduction-to-iommu-infrastructure-in-the-linux-kernel) — شرح شامل للـ IOMMU بالـ DMA translation mode والـ pass-through mode
- **IOMMU Introduction** (Terence Li): [terenceli.github.io](https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2019/08/04/iommu-introduction) — مقال تقني عميق بيشرح علاقة الـ IOVA بالـ page tables في Intel VT-d

---

### مصطلحات البحث

استخدم الكلمات دي للبحث عن معلومات أكتر:

```
# للبحث العام
"iova domain" linux kernel
"iova_domain" site:lore.kernel.org
"alloc_iova_fast" linux kernel

# للـ IOMMU context
"iommu iova" linux dma mapping
"iommu_dma" iova rcache linux
"CONFIG_IOMMU_IOVA" kernel

# للـ Intel VT-d context
"intel iommu" IOVA "pfn_lo" "pfn_hi"
"VT-d" DMA remapping IOVA linux

# للـ ARM SMMU context
"arm smmu" iova allocation linux
"iommu-dma" iova_rcache per-cpu

# على lore.kernel.org
site:lore.kernel.org iova rcache
site:lore.kernel.org iova_domain
```

---

### مواقع مفيدة للقراءة التفاعلية في الكود

| الموقع | الرابط | الاستخدام |
|--------|--------|-----------|
| Elixir Cross Referencer | [elixir.bootlin.com/linux/latest/source/include/linux/iova.h](https://elixir.bootlin.com/linux/latest/source/include/linux/iova.h) | تتبع كل مكان بيستخدم `struct iova` أو أي function |
| kernel.org git | [git.kernel.org/.../iova.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/iova.h) | تاريخ الـ commits الكامل للملف |
| GitHub mirror | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/iova.h) | سهل للـ browsing والـ blame |
| lore.kernel.org | [lore.kernel.org/linux-iommu](https://lore.kernel.org/linux-iommu/) | أرشيف الـ mailing list الرسمي |
## Phase 8: Writing simple module

### الفكرة

**`alloc_iova_fast`** هي الـ function الأكتر استخدامًا في الـ IOVA subsystem — بتتحرك في كل مرة device بيعمل DMA mapping. هنعمل **kprobe** عليها عشان نطبع معلومات عن كل allocation: الـ domain، الـ size المطلوبة، والـ limit_pfn.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe_iova_fast.c
 * Hooks alloc_iova_fast() to log every fast IOVA allocation.
 */

/* kprobes API */
#include <linux/kprobes.h>
/* pr_info / pr_err */
#include <linux/kernel.h>
/* module_init / module_exit / MODULE_* */
#include <linux/module.h>
/* struct iova_domain definition */
#include <linux/iova.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Learner");
MODULE_DESCRIPTION("kprobe on alloc_iova_fast to log DMA IOVA allocations");

/*
 * pre_handler — called just before alloc_iova_fast() executes.
 *
 * Signature of the hooked function:
 *   unsigned long alloc_iova_fast(struct iova_domain *iovad,
 *                                 unsigned long size,
 *                                 unsigned long limit_pfn,
 *                                 bool flush_rcache);
 *
 * On x86-64 the arguments live in registers:
 *   regs->di = iovad      (arg1)
 *   regs->si = size       (arg2)
 *   regs->dx = limit_pfn  (arg3)
 *   regs->cx = flush_rcache (arg4)
 */
static int iova_fast_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* pull the iova_domain pointer from the first argument register */
    struct iova_domain *iovad = (struct iova_domain *)regs->di;

    /* size in pages requested by the caller */
    unsigned long size      = regs->si;

    /* upper PFN limit the allocation must stay below */
    unsigned long limit_pfn = regs->dx;

    /* granule tells us the page size used by this domain */
    unsigned long granule = iovad ? iovad->granule : 0;

    pr_info("iova_probe: alloc_iova_fast | iovad=%px granule=%lu "
            "size_pages=%lu limit_pfn=0x%lx\n",
            iovad, granule, size, limit_pfn);

    /* returning 0 means: let the real function run normally */
    return 0;
}

/* kprobe descriptor — we only need pre_handler for read-only tracing */
static struct kprobe kp = {
    .symbol_name = "alloc_iova_fast",
    .pre_handler = iova_fast_pre,
};

static int __init iova_probe_init(void)
{
    int ret;

    /* register_kprobe يحط breakpoint على أول instruction في الـ function */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("iova_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("iova_probe: hooked alloc_iova_fast at %px\n", kp.addr);
    return 0;
}

static void __exit iova_probe_exit(void)
{
    /*
     * unregister_kprobe لازم يتعمل في exit عشان لو الـ module اتفك
     * من غير ما يشيل الـ breakpoint هيحصل kernel panic أول ما
     * الـ function تتنفذ وتلاقيش handler.
     */
    unregister_kprobe(&kp);
    pr_info("iova_probe: unhooked alloc_iova_fast\n");
}

module_init(iova_probe_init);
module_exit(iova_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/kernel.h` | بيجيب `pr_info` / `pr_err` ومعظم الـ macros الأساسية |
| `linux/module.h` | لازم لأي kernel module — بيوفر `module_init`, `module_exit`, `MODULE_*` |
| `linux/iova.h` | بيعرّف `struct iova_domain` عشان نقدر نوصل لحقل `granule` |

---

#### الـ `pre_handler`

الـ callback بياخد `struct pt_regs *regs` اللي فيه حالة الـ CPU registers لحظة الـ hook. على x86-64، الـ calling convention بيحط الـ arguments في `rdi → rsi → rdx → rcx` بالترتيب، فاحنا بنسحب كل argument بشكل مباشر من الـ register المناسب.

طبعنا `iovad` كـ pointer، الـ `granule` اللي بيحدد حجم الـ page في الـ domain ده، الـ `size` بعدد الـ pages المطلوبة، والـ `limit_pfn` عشان نعرف أقصى عنوان مسموح بيه للـ allocation.

---

#### الـ `kp` struct

الـ `symbol_name` بس كافي — الـ kernel بيحل العنوان الفعلي للـ function أثناء `register_kprobe`. احنا مش محتاجين نحدد العنوان يدويًا.

---

#### الـ `module_init`

`register_kprobe` بيعمل patching للـ function في الـ memory — بيحط `int3` (breakpoint) على أول byte. لو فشل (مثلًا الـ function مش exported أو الـ kprobes disabled في الـ kernel config)، بنرجع الـ error فورًا.

---

#### الـ `module_exit`

`unregister_kprobe` ضروري عشان يشيل الـ breakpoint من الـ memory ويحررالـ resources. لو فضل الـ breakpoint من غير handler، أي استدعاء للـ function بعد كده هيسبب **kernel panic**.

---

### طريقة التجربة

```bash
# Makefile بسيط
obj-m += kprobe_iova_fast.o

# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod kprobe_iova_fast.ko

# شغّل أي عملية DMA مثلًا:
# sudo dd if=/dev/zero of=/dev/null bs=1M count=100
# أو شغّل أي driver بيستخدم IOMMU

# شوف الـ log
sudo dmesg | grep iova_probe

# Unload
sudo rmmod kprobe_iova_fast
```

**مثال output متوقع:**
```
[  42.123456] iova_probe: hooked alloc_iova_fast at ffffffffc0ab1234
[  42.234567] iova_probe: alloc_iova_fast | iovad=ffff888012345678 granule=512 size_pages=8 limit_pfn=0xfffff
[  42.234890] iova_probe: alloc_iova_fast | iovad=ffff888012345678 granule=512 size_pages=1 limit_pfn=0xfffff
```

> **ملاحظة:** الـ kprobes بتشتغل بس لو `CONFIG_KPROBES=y` في الـ kernel config، وممكن تتأكد بـ `grep CONFIG_KPROBES /boot/config-$(uname -r)`.
