## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **IRQ Subsystem** — بالتحديد من الـ infrastructure اللي بتخلي الـ kernel يوزع الـ interrupts على الــ CPUs بشكل ذكي.

---

### الحكاية من الأول

تخيل عندك سيرفر فيه 128 CPU core، موزعين على 4 NUMA nodes، وكل node فيه clusters. وعندك NVMe SSD بيدعم 32 hardware queue متوازية.

لما الـ SSD بيبعت interrupt، مين من الـ 128 core يرد عليه؟

لو قلت "أي CPU" — هتبقى كارثة:
- الـ CPU اللي بيرد ممكن يكون على NUMA node تانية بعيدة عن الـ memory اللي فيها البيانات → latency عالية.
- ممكن كل الـ interrupts تروح على core واحد → bottleneck.
- ممكن تتجاهل الـ CPU topology (SMT siblings, clusters) → cache misses.

**المشكلة الحقيقية**: عايز توزع N queue على الـ CPUs المتاحين بشكل:
1. **عادل** — كل group ياخد CPUs قريبة من بعض.
2. **NUMA-aware** — CPUs من نفس الـ NUMA node يبقوا في نفس الـ group قدر الإمكان.
3. **Cluster-aware** — CPUs من نفس الـ cluster (يشاركوا L2/L3 cache) في نفس الـ group.
4. **بدون تداخل** — مفيش CPU يتعد في أكتر من group واحد.

---

### الـ File ده بيعمل إيه بالظبط

الـ `group_cpus.h` هو مجرد **header صغير جداً** — سطر واحد فعلي:

```c
struct cpumask *group_cpus_evenly(unsigned int numgrps, unsigned int *nummasks);
```

الـ API كلها في function واحدة:
- بتاخد عدد الـ groups المطلوبة (`numgrps`).
- بترجع array من `cpumask` — كل element فيه CPUs اللي اتخصصوا للـ group ده.
- الـ `nummasks` بيقولك كام group اتعمل فعلاً (ممكن أقل من اللي طلبته).

---

### الـ Algorithm بالتفصيل (في `lib/group_cpus.c`)

#### مرحلتين أساسيتين

**المرحلة الأولى — Present CPUs:**
بيوزع الـ CPUs اللي موجودة فعلاً (`cpu_present_mask`) على الـ groups.

**المرحلة الثانية — Possible CPUs:**
بيوزع الـ CPUs اللي ممكن تتضاف بعدين (hotplug) على الـ groups المتبقية.

#### التسلسل الهرمي للتوزيع

```
numgrps
   │
   ├─► هل numgrps <= عدد NUMA nodes؟
   │      نعم: كل node تاخد group واحدة
   │      لا:  وزّع الـ groups على الـ nodes بالتناسب مع عدد الـ CPUs
   │
   └─► لكل node:
          هل فيه clusters؟
          نعم: وزّع groups على الـ clusters أولاً (cluster-aware)
          لا:  وزّع الـ CPUs مباشرة على الـ groups (NUMA-aware)
```

#### ضمانة رياضية مهمة

الـ code فيه proof رياضي في الـ comments نفسها: لكل node، عدد الـ groups المخصصة ليه ≤ عدد الـ CPUs فيه — يعني مفيش group تبقى فاضية.

---

### مين بيستخدم الـ API دي؟

| الـ User | الغرض |
|---|---|
| `kernel/irq/affinity.c` | توزيع MSI/MSI-X interrupt vectors على الـ CPUs |
| `block/blk-mq-cpumap.c` | ربط كل block queue بمجموعة CPUs |
| `drivers/virtio/virtio_vdpa.c` | توزيع virtio queues |
| `fs/fuse/virtio_fs.c` | توزيع request queues في virtio-fs |

---

### القصة الكاملة بمثال

عندك NVMe driver فيه 8 hardware queues، والسيرفر فيه 16 CPU على NUMA node واحدة، منهم 8 في cluster A و 8 في cluster B:

1. الـ driver بيستدعي `group_cpus_evenly(8, &nr_masks)`.
2. الـ function بتقسم الـ CPUs: 4 groups من cluster A و 4 groups من cluster B.
3. كل group فيها 2 CPU من نفس الـ cluster (بيشاركوا cache).
4. الـ driver بيربط كل hardware queue بـ group → الـ interrupt بتروح دايماً على CPU قريب من الـ data.

**النتيجة**: أقل latency، أحسن cache locality، وتوزيع عادل للـ load.

---

### الفرق بين SMP و non-SMP

في الـ non-SMP build (CPU واحد):
```c
// كل الـ groups بتاخد نفس الـ CPU الوحيد
cpumask_copy(&masks[0], cpu_possible_mask);
*nummasks = 1;
```

في الـ SMP: الـ algorithm الكاملة اللي اتشرحت فوق.

---

### الفايلات المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/group_cpus.h` | الـ public API — الـ file الحالي |
| `lib/group_cpus.c` | الـ implementation الكاملة |
| `kernel/irq/affinity.c` | أهم مستخدم — توزيع IRQ affinity |
| `include/linux/cpumask.h` | الـ `cpumask` type والـ operations عليه |
| `include/linux/topology.h` | الـ `topology_sibling_cpumask`, `topology_cluster_cpumask` |
| `include/linux/nodemask.h` | الـ `nodemask_t` للـ NUMA nodes |
| `kernel/irq/` | الـ IRQ subsystem الكامل |
| `block/blk-mq-cpumap.c` | استخدام في الـ block multi-queue |

---

### ملخص

**الـ `group_cpus.h`** هو interface بسيط لـ algorithm ذكية — بتحوّل مشكلة "وزّع الـ resources على الـ CPUs" لحل مُحسَّن يراعي الـ NUMA topology، الـ CPU clusters، والـ cache locality في نفس الوقت. من غيره، كل driver كان هيعيد اختراع العجلة بطريقة أسوأ.
## Phase 2: شرح الـ group_cpus Framework

### المشكلة اللي بيحلها الـ subsystem ده

في أي نظام بيشتغل فيه **SMP (Symmetric Multi-Processing)**، لازم الـ kernel يوزّع الـ interrupt vectors (IRQs) على الـ CPUs بطريقة ذكية. لما عندك NVMe controller بـ 32 queue، أو network card بـ 16 TX/RX queue pair، كل queue محتاجة CPU affinity منفصل عشان تتجنب:

- **False sharing**: اتنين CPUs بيعالجوا نفس الـ interrupt queue → cache thrashing.
- **NUMA penalty**: CPU بيعالج interrupt جاي من device على NUMA node تاني → latency كبير جداً لأن الـ memory access هيكون remote.
- **SMT (Hyper-Threading) imbalance**: الـ sibling threads بيشتركوا في نفس الـ execution units، فلو وزّعت الـ interrupts عليهم بشكل عشوائي هتعمل bottleneck.

**السؤال:** إزاي توزّع N مجموعة من الـ CPUs على M queue بشكل عادل، مع مراعاة NUMA topology و CPU cluster locality؟

ده بالضبط اللي بيحله `group_cpus_evenly`.

---

### الحل اللي اتخده الـ kernel

الـ kernel بيستخدم **two-stage، topology-aware grouping**:

1. **Stage 1**: توزيع الـ `cpu_present_mask` (الـ CPUs اللي موجودة فعلاً) على الـ groups.
2. **Stage 2**: توزيع الـ `cpu_possible_mask - cpu_present_mask` (الـ CPUs اللي ممكن تتضاف بعدين بـ hotplug) على نفس الـ groups.

كل stage بيمشي بـ hierarchy من 3 مستويات:

```
NUMA Node → CPU Cluster → CPU Siblings (SMT)
```

الفكرة: CPUs اللي بتشارك نفس الـ last-level cache (LLC) أو نفس الـ NUMA node يتحطوا في نفس الـ group قبل ما نبدأ نفرّق عليهم.

---

### المثال الواقعي: توزيع طوابير في سوبر ماركت

تخيّل سوبر ماركت ضخم (الـ system) فيه:
- **3 فروع في 3 مناطق مختلفة** = NUMA nodes.
- **كل فرع فيه أقسام** = CPU clusters (مثلاً قسم الخضار، قسم اللحوم).
- **كل قسم فيه موظفين** = CPU siblings (SMT/HyperThreading).
- **الكاشيرات** = CPUs.
- **الطوابير** = interrupt groups.

الـ `group_cpus_evenly` بتعمل إيه؟ بتقول: "عايز 8 طوابير (groups)، وزّع الكاشيرات عليهم."

| المفهوم في السوبر ماركت | المقابل في الـ kernel |
|---|---|
| فرع في منطقة جغرافية | NUMA node |
| قسم داخل الفرع | CPU cluster |
| موظفين في نفس القسم | SMT siblings |
| كاشير | CPU core |
| طابور خدمة | interrupt group (cpumask) |
| قاعدة "الزبون يروح أقرب فرع" | NUMA locality |
| "موظفين نفس القسم يخدموا نفس الطابور" | cluster affinity |

الـ **invariant** المهم: كل كاشير (CPU) في طابور واحد بس — مفيش CPU في مجموعتين في نفس الوقت.

---

### Big Picture: فين بيقع الـ subsystem ده في الـ kernel؟

```
┌─────────────────────────────────────────────────────┐
│                   Device Drivers                     │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐  │
│  │  NVMe    │  │  NIC      │  │  SCSI/block dev  │  │
│  │ driver   │  │ driver    │  │  driver          │  │
│  └────┬─────┘  └─────┬─────┘  └────────┬─────────┘  │
│       │              │                  │             │
│       └──────────────┼──────────────────┘             │
│              call group_cpus_evenly(N)                │
└──────────────────────┼──────────────────────────────┘
                       │
               ┌───────▼────────┐
               │ group_cpus.c   │  ← lib/group_cpus.c
               │                │
               │  two-stage     │
               │  NUMA-aware    │
               │  grouping      │
               └───────┬────────┘
                       │  reads
          ┌────────────┼─────────────┐
          │            │             │
   ┌──────▼──────┐ ┌───▼────┐ ┌─────▼──────────┐
   │cpu_present  │ │cpu_    │ │topology_sibling │
   │_mask        │ │possible│ │_cpumask()       │
   │             │ │_mask   │ │topology_cluster │
   └─────────────┘ └────────┘ │_cpumask()       │
                              └─────────────────┘
                       │
               ┌───────▼────────┐
               │  Returns:      │
               │  struct        │
               │  cpumask[]     │
               │  (array of N   │
               │   cpumasks)    │
               └────────────────┘
```

---

### الـ Core Abstraction: مصفوفة الـ cpumasks

الـ output الوحيد من `group_cpus_evenly` هو:

```c
struct cpumask *group_cpus_evenly(unsigned int numgrps, unsigned int *nummasks);
```

**الـ return** هو array من `struct cpumask`، حيث:
- `masks[i]` = الـ bitmask اللي بيقول "الـ group i يشمل الـ CPUs دي".
- `*nummasks` = عدد الـ masks اللي اتملت فعلاً (ممكن يكون أقل من `numgrps` لو الـ CPUs أقل).

الـ `struct cpumask` نفسه ببساطة هو:

```c
// من include/linux/cpumask_types.h
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```

يعني في الآخر هو **bitmap** — bit واحدة لكل CPU في الـ system. لو الـ bit رقم 3 متضبطة، يعني CPU3 في المجموعة دي.

---

### التسلسل الداخلي للخوارزمية

```
group_cpus_evenly(numgrps)
│
├── alloc_node_to_cpumask()         ← بناء map: node → CPUs فيه
│   └── build_node_to_cpumask()
│
├── Stage 1: cpu_present_mask
│   └── __group_cpus_evenly(startgrp=0)
│       │
│       ├── get_nodes_in_cpumask()  ← كم NUMA node في الـ mask؟
│       │
│       ├── [Case A] numgrps <= nodes:
│       │   └── كل group يأخد node كامل (simple spread)
│       │
│       └── [Case B] numgrps > nodes:
│           ├── alloc_nodes_groups() ← كم group لكل node؟ (proportional)
│           └── for each node:
│               ├── __try_group_cluster_cpus()  ← try cluster-aware split
│               │   └── alloc_cluster_groups()  ← نفس المنطق على cluster level
│               │       └── topology_cluster_cpumask(cpu)
│               └── assign_cpus_to_groups()
│                   └── grp_spread_init_one()
│                       └── topology_sibling_cpumask(cpu) ← SMT siblings أولاً
│
└── Stage 2: cpu_possible - cpu_present
    └── __group_cpus_evenly(startgrp=nr_present)
        └── (نفس الـ logic فوق)
```

---

### الـ struct المحورية: `node_groups`

```c
struct node_groups {
    unsigned id;       // رقم الـ NUMA node (أو الـ cluster)

    union {
        unsigned ngroups;  // عدد الـ groups المخصصة لـ node ده
        unsigned ncpus;    // عدد الـ CPUs في الـ node ده (مؤقت أثناء الحساب)
    };
};
```

اللـ union ده ذكي: في بداية الحساب `ncpus` بتتحسب، وبعدين `ngroups` بتتحسب وبتحل محلها في نفس الـ memory.

---

### الـ NUMA-Proportional Allocation: الرياضيات

في `alloc_groups_to_nodes`، المنطق بيوزّع الـ groups على الـ NUMA nodes بنسبة عدد الـ CPUs:

```
grps(node_X) = max(1, floor(numgrps × ncpus(X) / remaining_ncpus))
```

**مثال عملي:**
- عندنا 2 NUMA nodes: A (4 CPUs) و B (12 CPUs).
- عايزين 8 groups.

```
total CPUs = 16
grps(A) = max(1, floor(8 × 4 / 16)) = max(1, 2) = 2
grps(B) = 8 - 2 = 6
```

الـ invariant المضمون بالـ proof في الـ comment: `grps(X) <= ncpus(X)` دايماً، يعني مش هينفع تاخد node فيها 2 CPU وتديها 3 groups.

---

### الـ SMT-Aware Grouping: `grp_spread_init_one`

لما بيملا group واحدة بـ CPUs، بيعمل إيه؟

```c
static void grp_spread_init_one(struct cpumask *irqmsk, struct cpumask *nmsk,
                                unsigned int cpus_per_grp)
{
    // خد أول CPU متاح
    cpu = cpumask_first(nmsk);
    cpumask_clear_cpu(cpu, nmsk);
    cpumask_set_cpu(cpu, irqmsk);
    cpus_per_grp--;

    // استخدم الـ SMT siblings بتاعه الأول قبل ما تروح لـ CPU تاني
    siblmsk = topology_sibling_cpumask(cpu);
    for (sibl = -1; cpus_per_grp > 0; ) {
        sibl = cpumask_next(sibl, siblmsk);
        // ...
        cpumask_set_cpu(sibl, irqmsk);
    }
}
```

**ليه؟** الـ SMT siblings بيشتركوا في نفس الـ physical core (L1/L2 cache)، فلو عندهم نفس الـ interrupt → cache warm → أسرع بكتير من إن thread على core تاني يعالج الـ interrupt.

---

### الـ Cluster Awareness: مستوى تاني من الـ hierarchy

الـ **CPU cluster** هو مجموعة من cores بيشتركوا في نفس الـ LLC (Last Level Cache) داخل نفس الـ NUMA node. شايعة في ARM big.LITTLE و Intel hybrid architectures.

```
NUMA Node 0
├── Cluster 0 (shared LLC)
│   ├── CPU 0 (core 0)
│   │   ├── SMT sibling: CPU 1
│   └── CPU 2 (core 1)
│       └── SMT sibling: CPU 3
└── Cluster 1 (shared LLC)
    ├── CPU 4
    └── CPU 5
```

`topology_cluster_cpumask(cpu)` → بيرجع الـ CPUs اللي في نفس الـ cluster مع الـ cpu ده.

لو `ngroups < ncluster`، الكود بيقول "مش هنقدر نخصص group لكل cluster منفصل، ارجع للـ node-level grouping الأبسط".

---

### الـ Two-Stage Design: ليه؟

**المشكلة:** الـ `cpu_present_mask` ممكن يتغير أثناء الـ CPU hotplug. لو أخدنا lock على الـ hotplug طول وقت الحساب → deadlock محتمل مع كود الـ hotplug نفسه.

**الحل:**
```c
// snapshot سريع بدون lock
cpumask_copy(npresmsk, data_race(cpu_present_mask));
```

الـ `data_race()` هنا بيقول للـ KCSAN (Kernel Concurrency Sanitizer): "عارفين إن في race محتمل هنا، بس مقبول intentionally."

لو CPU اتضاف أثناء Stage 1، هيتعالج في Stage 2 (possible - present). لو CPU اتشال، ممكن يتعالج في Stage 1 أو Stage 2، والنتيجة صح في الحالتين.

---

### إيه اللي الـ subsystem بيمتلكه vs إيه اللي بيفوّضه للـ drivers

| الـ subsystem بيمتلك | الـ driver بيمتلك |
|---|---|
| خوارزمية التوزيع NUMA/cluster/SMT-aware | قرار عدد الـ groups المطلوب |
| قراءة الـ topology من الـ kernel | ربط كل group بـ IRQ أو queue |
| ضمان no overlap بين الـ groups | تطبيق الـ affinity على الـ hardware |
| memory allocation للـ cpumask array | free الـ array بعد الاستخدام |
| التعامل مع hotplug CPUs (two-stage) | التعامل مع device queues |

---

### مثال واقعي: NVMe Driver

```c
// في drivers/nvme/host/pci.c
struct cpumask *qmap = group_cpus_evenly(dev->nr_queues, &nr_maps);
if (qmap) {
    for (i = 0; i < nr_maps; i++) {
        // ربط الـ queue i بالـ CPUs في masks[i]
        nvme_assign_io_queue(dev, i, &qmap[i]);
    }
    kfree(qmap);  // الـ driver مسؤول عن الـ free
}
```

الـ NVMe driver بيطلب التوزيع، وبعدين بيربط كل queue بالـ IRQ الخاص بيه عن طريق `irq_set_affinity_hint()` أو الـ managed IRQ affinity system.

---

### علاقة الـ subsystem ده بـ subsystems تانية

- **الـ IRQ subsystem** (`include/linux/interrupt.h`): بيستخدم نتيجة `group_cpus_evenly` عشان يضبط الـ `irq_affinity` لكل IRQ vector.
- **الـ NUMA subsystem** (`include/linux/numa.h`): الـ `cpu_to_node()` و `nr_node_ids` اللي بيستخدمهم الكود جاييين منه.
- **الـ CPU topology subsystem** (`include/linux/topology.h`): بيوفّر `topology_sibling_cpumask()` و `topology_cluster_cpumask()` اللي هي اللبنة الأساسية للـ locality-aware grouping.
- **الـ cpumask subsystem** (`include/linux/cpumask.h`): الـ data structure الأساسية — bitmap واحد لكل ممكن 8192 CPU.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الملف المصدر

**`include/linux/group_cpus.h`** — header بسيط جداً، يعلن عن دالة واحدة فقط:
```c
struct cpumask *group_cpus_evenly(unsigned int numgrps, unsigned int *nummasks);
```

الـ implementation الحقيقي في **`lib/group_cpus.c`**، وده اللي هنحلله بالتفصيل.

---

### 0. الـ Config Options والـ Flags المهمة

| Option / Macro | المعنى |
|---|---|
| `CONFIG_SMP` | لو مفعّل: يستخدم full NUMA-aware algorithm، لو لأ: يحط كل الـ CPUs في group واحدة |
| `nr_cpu_ids` | عدد الـ CPUs الفعلي في النظام |
| `nr_node_ids` | عدد الـ NUMA nodes |
| `cpu_present_mask` | الـ CPUs الموجودة فعلاً (plugged in) |
| `cpu_possible_mask` | الـ CPUs الممكن وجودها (بما فيها offline) |
| `UINT_MAX` | marker بيتحط في `node_groups[n].ncpus` لو الـ node مش active |
| `GFP_KERNEL` | flag الـ memory allocation — sleeping allowed |
| `data_race()` | بيقول للـ sanitizer "عارف إن في race هنا بس مقصودة" |

---

### 1. الـ Structs المهمة

#### `struct cpumask`

**الغرض:** bitmap بيمثل مجموعة من الـ CPUs، كل bit = CPU واحدة.

| الـ field | النوع | المعنى |
|---|---|---|
| `bits[]` | `unsigned long[]` | الـ bitmap نفسه |

**الارتباطات:**
- النتيجة النهائية لـ `group_cpus_evenly` هي **array** من `struct cpumask`.
- كل عنصر في الـ array = group واحدة من الـ CPUs.
- `cpumask_var_t` هي إما pointer أو embedded struct حسب الـ config.

---

#### `struct node_groups`  *(defined in `lib/group_cpus.c`)*

**الغرض:** بيخزن معلومات التوزيع لـ NUMA node واحد (أو cluster واحد) — كام CPU فيه وكام group هياخد.

```c
struct node_groups {
    unsigned id;       /* node ID أو cluster ID */

    union {
        unsigned ngroups;  /* عدد الـ groups المخصصة لهذا الـ node */
        unsigned ncpus;    /* عدد الـ CPUs فيه (قبل التخصيص) */
    };
};
```

**ملاحظة مهمة على الـ union:** الـ field الواحد بيُستخدم بمعنيين مختلفين في مراحل مختلفة:
- **مرحلة الحساب (قبل `alloc_groups_to_nodes`):** `ncpus` = عدد الـ CPUs في الـ node.
- **مرحلة التخصيص (بعدها):** نفس الـ memory بتصبح `ngroups` = عدد الـ groups المخصصة.

**الارتباطات:**
- Array منها بيتعمل بـ `kcalloc(nr_node_ids, ...)` في `__group_cpus_evenly`.
- نفس الـ struct بيُستخدم لتمثيل الـ clusters داخل node واحد في `alloc_cluster_groups`.

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
group_cpus_evenly()
│
│  يخصص:
│
├─► struct cpumask  masks[numgrps]     ← النتيجة النهائية (array)
│       كل عنصر فيه: set of CPUs لـ group واحدة
│
├─► cpumask_var_t  nmsk               ← temp mask للحسابات
├─► cpumask_var_t  npresmsk           ← snapshot من cpu_present_mask
│
└─► cpumask_var_t  node_to_cpumask[nr_node_ids]
        كل عنصر: CPUs التابعة لـ NUMA node معين
        │
        └─► يُستخدم في __group_cpus_evenly()
                │
                └─► struct node_groups  node_groups[nr_node_ids]
                        .id      = node index
                        .ncpus   = CPU count (قبل التخصيص)
                        .ngroups = group count (بعد التخصيص)
                        │
                        └─► في alloc_cluster_groups():
                            struct node_groups  cluster_groups[ncluster]
                            const struct cpumask *clusters[ncluster]
                                └─► topology_cluster_cpumask(cpu)
                                    (pointer مباشر للـ topology data)
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
[ALLOCATION PHASE]
    kcalloc(numgrps, sizeof(cpumask))  →  masks[]
    zalloc_cpumask_var(&nmsk)          →  temp mask
    zalloc_cpumask_var(&npresmsk)      →  present snapshot
    alloc_node_to_cpumask()            →  node_to_cpumask[]
        └── kcalloc(nr_node_ids, ...)
        └── zalloc_cpumask_var() × nr_node_ids

[POPULATION PHASE]
    build_node_to_cpumask()
        └── for_each_possible_cpu → cpumask_set_cpu(cpu, masks[cpu_to_node(cpu)])

    cpumask_copy(npresmsk, cpu_present_mask)   ← data_race() snapshot

[GROUPING PHASE — 2 stages]
    Stage 1: __group_cpus_evenly(0, numgrps, ..., npresmsk, ...)
        → present CPUs first
    Stage 2: __group_cpus_evenly(curgrp, numgrps, ..., possible \ present, ...)
        → non-present CPUs second

[TEARDOWN — internal temps]
    free_node_to_cpumask(node_to_cpumask)
    free_cpumask_var(npresmsk)
    free_cpumask_var(nmsk)

[RETURN]
    → caller gets: struct cpumask *masks  (owned by caller, freed with kfree)
    → *nummasks = min(nr_present + nr_others, numgrps)
```

---

### 4. مخطط الـ Call Flow التفصيلي

```
group_cpus_evenly(numgrps, *nummasks)
│
├─ alloc_node_to_cpumask()
│   └─ kcalloc + zalloc_cpumask_var × N
│
├─ build_node_to_cpumask()
│   └─ for_each_possible_cpu → cpu_to_node() → cpumask_set_cpu()
│
├─ [Stage 1] __group_cpus_evenly(0, numgrps, ..., cpu_present_mask, ...)
│   │
│   ├─ get_nodes_in_cpumask()          ← كام NUMA node في الـ mask؟
│   │
│   ├─ [if numgrps <= nodes]           ← groups أقل من nodes: spread مباشر
│   │   └─ for_each_node_mask → cpumask_or(&masks[curgrp], ...)
│   │
│   └─ [if numgrps > nodes]           ← groups أكتر من nodes: نوزع بالنسبة
│       ├─ alloc_nodes_groups()
│       │   ├─ حساب ncpus لكل node
│       │   └─ alloc_groups_to_nodes()
│       │       └─ sort(node_groups, ncpus_cmp_func)  ← ترتيب تصاعدي
│       │           └─ for n: ngroups[n] = max(1, G * ncpus[n] / remaining)
│       │
│       └─ for each node_groups[i]:
│           ├─ cpumask_and(nmsk, cpu_mask, node_to_cpumask[i])
│           │
│           ├─ __try_group_cluster_cpus()   ← حاول cluster-aware grouping
│           │   ├─ alloc_cluster_groups()
│           │   │   ├─ topology_cluster_cpumask(cpu)  ← كل cluster
│           │   │   ├─ alloc_groups_to_nodes() على الـ clusters
│           │   │   └─ return ncluster
│           │   │
│           │   └─ for each cluster:
│           │       └─ assign_cpus_to_groups()
│           │           └─ grp_spread_init_one()
│           │               ├─ cpumask_first(nmsk)
│           │               ├─ topology_sibling_cpumask(cpu)  ← SMT siblings أول
│           │               └─ cpumask_set_cpu(cpu, irqmsk)
│           │
│           └─ [fallback] assign_cpus_to_groups()  ← لو مفيش clusters
│               └─ grp_spread_init_one()
│
└─ [Stage 2] __group_cpus_evenly(curgrp, numgrps, ..., possible\present, ...)
    └─ نفس الـ flow بس على الـ CPUs غير الموجودة
```

---

### 5. خوارزمية توزيع الـ Groups على الـ Nodes

**المبدأ:** كل node ياخد groups بنسبة عدد الـ CPUs فيه، مع ضمان ≥ 1 group لكل node active.

```
alloc_groups_to_nodes():

  sort nodes ascending by ncpus
  remaining_ncpus = total_cpus
  numgrps = requested_groups

  for each node (smallest first):
      ngroups = max(1,  numgrps * ncpus / remaining_ncpus)
      node.ngroups = ngroups
      remaining_ncpus -= ncpus
      numgrps -= ngroups

الضمان الرياضي:
  ∀ node X:  ngroups(X) ≤ ncpus(X)
  (البرهان موجود في الكود نفسه — انظر التعليقات)
```

---

### 6. الـ Priority في اختيار الـ CPUs (Locality)

الكود بيراعي التوضع المادي للـ CPUs بالترتيب ده:

```
Priority 1 (أعلى): نفس الـ NUMA Node
    ↓
Priority 2: نفس الـ CPU Cluster (topology_cluster_cpumask)
    ↓
Priority 3: نفس الـ SMT Core (topology_sibling_cpumask)
    ↓
Priority 4 (أدنى): أي CPU في الـ mask
```

---

### 7. استراتيجية الـ Locking

| السيناريو | الـ mechanism |
|---|---|
| قراءة `cpu_present_mask` | `data_race()` macro — بيأخد snapshot بدون lock |
| CPU hotplug أثناء التنفيذ | مقبول — الـ CPU الـ hotplugged هيظهر في Stage 1 أو 2، كلاهما صح |
| الـ node_to_cpumask | local allocation — مفيش sharing → مفيش lock |
| الـ masks[] النتيجة | الـ caller مسؤول عن الـ locking بعد الإرجاع |

**السبب في تجنب cpu hotplug lock:**
> الكود صراحةً بيقول إنه قرار مقصود لتفادي deadlock مع كود الـ hotplug. الـ race على `cpu_present_mask` لا تأثير لها على الصحة — أسوأ حالة: الـ CPU الـ hotplugged يُعامَل كـ "non-present" ويُخصَّص في Stage 2.

---

### 8. الفرق بين SMP وغيره

```c
#ifdef CONFIG_SMP
/* Full NUMA-aware, cluster-aware, SMT-aware algorithm */
struct cpumask *group_cpus_evenly(...) { ... }

#else
/* UP (Uniprocessor): trivial — كل الـ CPUs في group واحدة */
struct cpumask *group_cpus_evenly(unsigned int numgrps, unsigned int *nummasks)
{
    masks = kcalloc(numgrps, sizeof(*masks), GFP_KERNEL);
    cpumask_copy(&masks[0], cpu_possible_mask);
    *nummasks = 1;
    return masks;
}
#endif
```

---

### 9. ملخص الـ API

| العنصر | التفاصيل |
|---|---|
| **الدالة** | `group_cpus_evenly(numgrps, *nummasks)` |
| **الـ input** | `numgrps`: عدد الـ groups المطلوبة |
| **الـ output** | array من `struct cpumask` مخصص بـ `kcalloc` |
| **`*nummasks`** | عدد الـ masks الفعلية المُعبَّأة (≤ numgrps) |
| **الـ ownership** | الـ caller هو المسؤول عن `kfree(masks)` |
| **الـ export** | `EXPORT_SYMBOL_GPL` — متاح للـ modules |
| **الـ use case** | توزيع الـ IRQ vectors على الـ CPU cores بتوعية بالـ topology |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| Function | Scope | الغرض |
|---|---|---|
| `group_cpus_evenly` | `EXPORT_SYMBOL_GPL` | الـ public API — توزيع الـ CPUs على groups بشكل متساوي مع مراعاة NUMA/cluster locality |
| `__group_cpus_evenly` | `static` | الـ core engine — يوزع CPUs من mask معين على الـ groups مع مراعاة NUMA topology |
| `alloc_nodes_groups` | `static` | يحسب كام group تاخد كل NUMA node بناءً على نسبة الـ CPUs |
| `alloc_groups_to_nodes` | `static` | يوزع الـ group count على الـ nodes بـ proportional allocation مع sorting |
| `assign_cpus_to_groups` | `static` | يعمل الـ actual assignment — CPUs لـ groups داخل node واحد |
| `grp_spread_init_one` | `static` | يملا group واحد بـ CPUs مع تفضيل الـ SMT siblings |
| `__try_group_cluster_cpus` | `static` | يحاول توزيع الـ CPUs بمراعاة CPU cluster topology قبل fallback لـ NUMA-only |
| `alloc_cluster_groups` | `static` | يكتشف الـ clusters داخل node ويوزع الـ groups عليهم |
| `build_node_to_cpumask` | `static` | يبني mapping من كل NUMA node لـ cpumask الخاص بيه |
| `alloc_node_to_cpumask` | `static` | يخصص مصفوفة من الـ `cpumask_var_t` بعدد الـ NUMA nodes |
| `free_node_to_cpumask` | `static` | يحرر المصفوفة اللي اتعملت في `alloc_node_to_cpumask` |
| `get_nodes_in_cpumask` | `static` | بيرجع عدد الـ NUMA nodes اللي فيها CPUs من mask معين |
| `ncpus_cmp_func` | `static` | comparator للـ `sort()` — يرتب `node_groups` تصاعدياً بعدد الـ CPUs |

---

### الـ Data Structure المحورية

```c
struct node_groups {
    unsigned id;       /* NUMA node ID أو cluster index */
    union {
        unsigned ngroups;  /* عدد الـ groups المخصصة لهذا الـ node */
        unsigned ncpus;    /* عدد الـ CPUs الـ active في هذا الـ node */
    };
};
```

الـ union ده استخدامه بيتغير حسب مرحلة الحساب — أول ما يتملا `ncpus`، بعدين بعد `alloc_groups_to_nodes` يتملا `ngroups` في نفس المكان.

---

### Group 1: Public API

#### `group_cpus_evenly`

```c
struct cpumask *group_cpus_evenly(unsigned int numgrps, unsigned int *nummasks);
```

ده الـ exported function الوحيد في الملف — هو الـ entry point لكل اللي بيستخدم subsystem ده (IRQ affinity spreading، NVMe queues، إلخ).

**بيعمل إيه:**
الـ function بتوزع كل الـ CPUs في النظام على `numgrps` groups بطريقة locality-aware — الـ CPUs القريبة من بعض (NUMA node أو CPU cluster) بتتحط في نفس الـ group. بيشتغل على مرحلتين: الأول بيوزع الـ **present CPUs** (المتاحة فعلاً)، وبعدين بيوزع الـ **possible-but-not-present CPUs** (مفيدة لـ hotplug scenarios) ابتداءً من الـ group التالي.

**Parameters:**
- `numgrps` — عدد الـ groups المطلوبة، لو صفر بيرجع NULL فوراً.
- `nummasks` — pointer لـ `unsigned int` — بيتكتب فيه عدد الـ masks اللي اتملت فعلاً (ممكن يكون أقل من `numgrps` لو عدد الـ CPUs أقل).

**Return value:**
- Pointer لـ array من `struct cpumask` بحجم `numgrps` — المُعالج بيكون مسؤول عن `kfree()`ه.
- `NULL` لو `numgrps == 0` أو في allocation failure.

**Key details:**
- بيستخدم `data_race(cpu_present_mask)` — قصداً بدون lock لأن الـ two-stage design يتعامل مع الـ race بشكل مقبول.
- لو `nr_present >= numgrps` في المرحلة الأولى، المرحلة التانية تبدأ من `curgrp = 0` (wrap around) بدل ما توسع.
- لو SMP مش enabled، النسخة البسيطة بتحط كل الـ CPUs في الـ group الأولى وترجع `*nummasks = 1`.
- `EXPORT_SYMBOL_GPL` — متاح لـ kernel modules بس بـ GPL license.

**Caller context:** Process context فقط، بيعمل `GFP_KERNEL` allocations. المستخدمين الرئيسيين: `pci_alloc_irq_vectors`، NVMe، SCSI multi-queue subsystems.

**Pseudocode flow:**
```
group_cpus_evenly(numgrps, nummasks):
    alloc: nmsk, npresmsk, node_to_cpumask[], masks[numgrps]
    build_node_to_cpumask()

    // Stage 1: present CPUs
    npresmsk = cpu_present_mask (data_race snapshot)
    nr_present = __group_cpus_evenly(0, numgrps, ..., npresmsk, masks)

    // Stage 2: possible but not present
    curgrp = (nr_present >= numgrps) ? 0 : nr_present
    npresmsk = cpu_possible_mask AND NOT npresmsk
    nr_others = __group_cpus_evenly(curgrp, numgrps, ..., npresmsk, masks)

    *nummasks = min(nr_present + nr_others, numgrps)
    return masks
```

---

### Group 2: Core Distribution Engine

#### `__group_cpus_evenly`

```c
static int __group_cpus_evenly(unsigned int startgrp, unsigned int numgrps,
                               cpumask_var_t *node_to_cpumask,
                               const struct cpumask *cpu_mask,
                               struct cpumask *nmsk, struct cpumask *masks);
```

الـ engine الأساسي للتوزيع — بيشتغل على mask واحد (present أو possible).

**بيعمل إيه:**
أول ما بيحدد عدد الـ NUMA nodes اللي فيها CPUs من الـ `cpu_mask`. لو عدد الـ groups <= عدد الـ nodes، بيعمل 1-to-1 mapping بسيط (كل group = CPUs من node). لو الـ groups أكتر من الـ nodes، بيستدعي `alloc_nodes_groups` عشان يقسم الـ groups على الـ nodes proportionally، وبعدين لكل node بيحاول أولاً `__try_group_cluster_cpus` وبعدين fallback لـ `assign_cpus_to_groups`.

**Parameters:**
- `startgrp` — الـ index اللي يبدأ منه التوزيع في مصفوفة الـ masks (مهم للـ two-stage design).
- `numgrps` — إجمالي عدد الـ groups في المصفوفة.
- `node_to_cpumask` — مصفوفة mapping من node ID لـ cpumask.
- `cpu_mask` — الـ CPUs المراد توزيعها (present أو possible).
- `nmsk` — scratch cpumask مؤقت.
- `masks` — الـ output array من الـ cpumasks اللي بيتكتب فيها النتيجة.

**Return value:**
- عدد الـ groups اللي اتملت (양수).
- `-ENOMEM` لو فشل الـ `kcalloc`.

**Key details:**
- بيستخدم `curgrp` بـ wrap-around عشان يتعامل مع الـ `startgrp != 0`.
- لما `numgrps <= nodes`، بيعمل `cpumask_or` مش `cpumask_copy` عشان يدعم الـ two-stage overlap.

---

### Group 3: NUMA Node Allocation

#### `alloc_nodes_groups`

```c
static void alloc_nodes_groups(unsigned int numgrps,
                               cpumask_var_t *node_to_cpumask,
                               const struct cpumask *cpu_mask,
                               const nodemask_t nodemsk,
                               struct cpumask *nmsk,
                               struct node_groups *node_groups);
```

**بيعمل إيه:**
بيحسب عدد الـ active CPUs لكل NUMA node (intersection بين `cpu_mask` و `node_to_cpumask[n]`)، وبعدين بيستدعي `alloc_groups_to_nodes` عشان يوزع الـ `numgrps` على الـ nodes بنسب CPUs كل node.

**Parameters:**
- `numgrps` — عدد الـ groups المراد توزيعها.
- `node_to_cpumask` — mapping من node لـ cpumask.
- `cpu_mask` — الـ CPUs المراد النظر فيهم.
- `nodemsk` — nodemask بيه الـ nodes الفعلية.
- `nmsk` — scratch mask.
- `node_groups` — المصفوفة اللي بيتكتب فيها النتيجة (بـ `nr_node_ids` عناصر).

**Key details:**
- الـ nodes اللي مش في `nodemsk` بتاخد `ncpus = UINT_MAX` كـ sentinel تتجاهل.
- `numgrps = min(numcpus, numgrps)` — بيضمن إن مش في group فاضية لو الـ CPUs أقل.

---

#### `alloc_groups_to_nodes`

```c
static void alloc_groups_to_nodes(unsigned int numgrps,
                                  unsigned int numcpus,
                                  struct node_groups *node_groups,
                                  unsigned int num_nodes);
```

**بيعمل إيه:**
بيرتب الـ nodes تصاعدياً بعدد الـ CPUs (smallest-first عشان يضمن إن كل node تاخد على الأقل group واحدة)، وبعدين بيوزع الـ groups بنسبة `ncpus / remaining_ncpus`. الـ formula هي `ngroups = max(1, numgrps * ncpus / remaining_ncpus)` — بيضمن mathematically إن `ngroups <= ncpus` لكل node.

**Parameters:**
- `numgrps` — إجمالي الـ groups المراد توزيعها.
- `numcpus` — إجمالي الـ CPUs.
- `node_groups` — مصفوفة الـ nodes (بتتعدل in-place).
- `num_nodes` — حجم المصفوفة.

**Key details:**
- الـ `sort()` بيغير ترتيب المصفوفة — الـ `id` field بيحتفظ بالـ original node ID بعد الـ sort.
- الـ nodes ذات `ncpus == UINT_MAX` (sentinel) بيتخطاها.
- `WARN_ON_ONCE(ngroups > ncpus)` — تأكد من الـ invariant.

---

### Group 4: CPU Cluster Awareness

#### `__try_group_cluster_cpus`

```c
static bool __try_group_cluster_cpus(unsigned int ncpus,
                                     unsigned int ngroups,
                                     struct cpumask *node_cpumask,
                                     struct cpumask *masks,
                                     unsigned int *curgrp,
                                     unsigned int last_grp);
```

**بيعمل إيه:**
بيحاول يوزع الـ CPUs مع مراعاة الـ **CPU cluster topology** (اللي هي أدق من NUMA node — زي الـ big.LITTLE clusters في ARM). لو نجح، بيرجع `true` وبيعمل الـ assignment. لو فشل (مفيش clusters، أو allocations فشلت، أو `ngroups < ncluster`)، بيرجع `false` والـ caller يعمل fallback لـ `assign_cpus_to_groups`.

**Parameters:**
- `ncpus` — عدد الـ CPUs في الـ node.
- `ngroups` — عدد الـ groups المخصصة لهذا الـ node.
- `node_cpumask` — mask الـ CPUs في الـ node.
- `masks` — الـ output masks array.
- `curgrp` — pointer للـ current group index (بيتعدل in-place).
- `last_grp` — upper bound للـ wrap-around.

**Return value:** `true` لو اتعمل cluster-aware grouping، `false` لو لازم fallback.

**Key details:**
- بيستخدم `topology_cluster_cpumask(cpu)` — kernel topology API للـ CPU clusters.
- الـ condition `ncluster > ngroups` يعني حتحتاج cross-cluster groups — في الحالة دي أفضل ما تحاول وترجع `false`.

---

#### `alloc_cluster_groups`

```c
static int alloc_cluster_groups(unsigned int ncpus,
                                unsigned int ngroups,
                                struct cpumask *node_cpumask,
                                cpumask_var_t msk,
                                const struct cpumask ***clusters_ptr,
                                struct node_groups **cluster_groups_ptr);
```

**بيعمل إيه:**
بيكتشف عدد الـ CPU clusters في الـ node عن طريق `topology_cluster_cpumask()` وبيتسلل من خلال الـ CPUs. بعدين بيخصص مصفوفتين — `clusters[]` للـ cpumasks، و `cluster_groups[]` لـ proportional distribution. بيستدعي `alloc_groups_to_nodes` لتوزيع الـ `ngroups` على الـ clusters.

**Parameters:**
- `ncpus` / `ngroups` — عدد الـ CPUs والـ groups للـ node الحالية.
- `node_cpumask` — mask الـ CPUs في الـ node.
- `msk` — scratch cpumask.
- `clusters_ptr` / `cluster_groups_ptr` — output pointers (caller مسؤول عن `kfree()`هم).

**Return value:** عدد الـ clusters المكتشفة، أو `0` لو مفيش clusters أو في failure.

---

### Group 5: CPU Assignment Primitives

#### `assign_cpus_to_groups`

```c
static void assign_cpus_to_groups(unsigned int ncpus,
                                  struct cpumask *nmsk,
                                  struct node_groups *nv,
                                  struct cpumask *masks,
                                  unsigned int *curgrp,
                                  unsigned int last_grp);
```

**بيعمل إيه:**
بيوزع الـ `ncpus` على `nv->ngroups` groups بالتساوي مع حساب الـ rounding error — الـ groups الأولى بتاخد `cpus_per_grp + 1` لو فيه فرق في القسمة. بيستدعي `grp_spread_init_one` لكل group.

**Parameters:**
- `ncpus` — عدد الـ CPUs في الـ nmsk.
- `nmsk` — الـ CPUs المراد توزيعها (بيتعدل in-place — بيتم clear الـ CPUs منه أثناء التوزيع).
- `nv` — الـ `node_groups` struct اللي فيه `ngroups`.
- `masks` — الـ output array.
- `curgrp` — pointer للـ current index (بيتعدل بـ `+1` لكل group).
- `last_grp` — `numgrps` للـ wrap-around check.

**Key details:**
- `extra_grps = ncpus - nv->ngroups * (ncpus / nv->ngroups)` — ده عدد الـ groups اللي هتاخد CPU إضافي عشان تعوض الـ integer division truncation.
- الـ wrap-around: `if (*curgrp >= last_grp) *curgrp = 0`.

---

#### `grp_spread_init_one`

```c
static void grp_spread_init_one(struct cpumask *irqmsk, struct cpumask *nmsk,
                                unsigned int cpus_per_grp);
```

**بيعمل إيه:**
بيملا `irqmsk` بـ `cpus_per_grp` من الـ CPUs مأخوذة من `nmsk`، مع **تفضيل الـ SMT siblings** — لما بياخد CPU، بيحاول ياخد الـ sibling CPUs بتاعته الأول (اللي هما الـ hardware threads على نفس الـ physical core). ده بيضمن إن الـ group تشمل الـ full physical core بدل ما تتقسم على groups مختلفة.

**Parameters:**
- `irqmsk` — الـ output mask للـ group — بيتعمله set بالـ CPUs المختارة.
- `nmsk` — pool الـ CPUs المتاحة — بيتعمله clear للـ CPUs اللي اتاخدت.
- `cpus_per_grp` — عدد الـ CPUs المطلوب في الـ group.

**Key details:**
- بيستخدم `topology_sibling_cpumask(cpu)` — اللي بترجع الـ SMT thread siblings (الـ hyperthreads).
- `cpumask_test_and_clear_cpu` — atomic-free هنا لأن مفيش concurrency في هذا الـ context.
- لو `cpu >= nr_cpu_ids`، بيخرج بدون WARN — الـ comment في الكود بيقول "too lazy to think about it" لكن عملياً ده shouldn't happen.

**Pseudocode flow:**
```
grp_spread_init_one(irqmsk, nmsk, cpus_per_grp):
    while cpus_per_grp > 0:
        cpu = first CPU in nmsk
        if cpu invalid: return
        clear cpu from nmsk, set in irqmsk, cpus_per_grp--

        // prefer SMT siblings
        siblmsk = topology_sibling_cpumask(cpu)
        for each sibl in siblmsk:
            if cpus_per_grp == 0: break
            if sibl in nmsk:
                clear sibl from nmsk, set in irqmsk, cpus_per_grp--
```

---

### Group 6: Memory Management Helpers

#### `alloc_node_to_cpumask`

```c
static cpumask_var_t *alloc_node_to_cpumask(void);
```

**بيعمل إيه:**
بيخصص مصفوفة من الـ `cpumask_var_t` بحجم `nr_node_ids` — يعني cpumask واحد لكل NUMA node ممكن في النظام.

**Return value:** Pointer للمصفوفة، أو `NULL` لو أي allocation فشلت.

**Key details:**
- بيستخدم `kcalloc` للمصفوفة الخارجية و `zalloc_cpumask_var` لكل عنصر.
- الـ error path `out_unwind` بيحرر كل الـ cpumasks اللي اتخصصت قبل الـ failure — تنظيف كامل.

---

#### `free_node_to_cpumask`

```c
static void free_node_to_cpumask(cpumask_var_t *masks);
```

**بيعمل إيه:** عكس `alloc_node_to_cpumask` — بيحرر كل الـ `nr_node_ids` cpumasks وبعدين الـ array نفسه.

---

#### `build_node_to_cpumask`

```c
static void build_node_to_cpumask(cpumask_var_t *masks);
```

**بيعمل إيه:** بيتمشى على كل الـ possible CPUs عن طريق `for_each_possible_cpu` ويحط كل CPU في الـ `masks[cpu_to_node(cpu)]` — يعني بيبني الـ mapping من NUMA node لـ CPUs.

---

#### `get_nodes_in_cpumask`

```c
static int get_nodes_in_cpumask(cpumask_var_t *node_to_cpumask,
                                const struct cpumask *mask, nodemask_t *nodemsk);
```

**بيعمل إيه:** بيتمشى على كل الـ nodes، لو فيه intersection بين `mask` والـ CPUs في الـ node — بيضيف الـ node في `nodemsk` وبيعد.

**Return value:** عدد الـ NUMA nodes اللي فيها CPUs من الـ `mask`.

---

#### `ncpus_cmp_func`

```c
static int ncpus_cmp_func(const void *l, const void *r);
```

**بيعمل إيه:** comparator بسيط لـ `sort()` — بيرتب الـ `node_groups` تصاعدياً بعدد الـ CPUs.

**Key details:** الـ subtraction `ln->ncpus - rn->ncpus` آمنة هنا لأن الـ values محدودة (`ncpus` unsigned int صغير، مش بيحصل overflow).

---

### التدفق الكامل — Big Picture

```
group_cpus_evenly(N groups)
│
├─ alloc_node_to_cpumask()        // allocate per-node cpumask array
├─ build_node_to_cpumask()        // populate: node → CPUs
│
├─ Stage 1: present CPUs
│   └─ __group_cpus_evenly(start=0, cpu_present_mask)
│       ├─ get_nodes_in_cpumask() // how many NUMA nodes?
│       ├─ [groups <= nodes] → simple 1-node-per-group OR
│       └─ [groups > nodes]  → alloc_nodes_groups()
│           └─ alloc_groups_to_nodes() // sort + proportional alloc
│               └─ for each node:
│                   ├─ __try_group_cluster_cpus()  // cluster-aware
│                   │   └─ alloc_cluster_groups()
│                   │       └─ assign_cpus_to_groups() per cluster
│                   └─ [fallback] assign_cpus_to_groups()
│                       └─ grp_spread_init_one() per group
│
└─ Stage 2: possible-but-not-present CPUs
    └─ __group_cpus_evenly(start=nr_present, npresmsk complement)
        └─ (same flow as Stage 1)
```

الـ locality priority من الأعلى للأدنى: **SMT siblings → CPU cluster → NUMA node → everything else**.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بندرسه هو `group_cpus_evenly()` — الـ function اللي بتوزع الـ CPUs على groups بشكل متساوي مع مراعاة الـ NUMA locality والـ CPU cluster topology. الـ debugging هنا بيدور حوالين ثلاث محاور: صحة التوزيع، الـ memory allocation، وتطابق الـ topology.

---

### Software Level

#### 1. debugfs Entries

الـ subsystem ده مفيش له entries مخصصة في debugfs، لكن الـ IRQ affinity اللي هو أكبر مستخدم لـ `group_cpus_evenly()` موجود:

| المسار | الوصف | أمر القراءة |
|---|---|---|
| `/sys/kernel/debug/irq/irqs/<N>/` | معلومات الـ IRQ الكاملة | `ls /sys/kernel/debug/irq/irqs/0/` |
| `/proc/irq/<N>/smp_affinity` | الـ cpumask للـ IRQ بصيغة hex | `cat /proc/irq/24/smp_affinity` |
| `/proc/irq/<N>/smp_affinity_list` | نفس المعلومة بصيغة CPU list | `cat /proc/irq/24/smp_affinity_list` |
| `/proc/interrupts` | جدول كامل بكل الـ IRQs وتوزيعها | `cat /proc/interrupts` |

```bash
# اقرأ الـ affinity لكل IRQ في وقت واحد
for irq in /proc/irq/*/smp_affinity_list; do
    echo "$irq: $(cat $irq)"
done
```

#### 2. sysfs Entries المتعلقة بالـ Topology

الـ function بتستخدم `topology_sibling_cpumask()` و`topology_cluster_cpumask()` — لازم تتحقق منهم:

```bash
# topology_sibling_cpumask (HT siblings)
cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list

# topology_cluster_cpumask
cat /sys/devices/system/cpu/cpu0/topology/cluster_cpus_list

# NUMA node لكل CPU
cat /sys/devices/system/cpu/cpu0/topology/physical_package_id
for cpu in /sys/devices/system/cpu/cpu[0-9]*/; do
    node=$(cat ${cpu}topology/physical_package_id 2>/dev/null)
    echo "$(basename $cpu): NUMA node $node"
done

# عدد الـ online CPUs
cat /sys/devices/system/cpu/online

# cpu_present_mask - اللي بتستخدمه group_cpus_evenly مباشرة
cat /sys/devices/system/cpu/present
cat /sys/devices/system/cpu/possible
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# فعّل tracing على مستوى الـ memory allocation
cd /sys/kernel/debug/tracing

# تابع kmalloc/kcalloc اللي بتستخدمها group_cpus_evenly
echo 1 > events/kmem/kmalloc/enable
echo 1 > events/kmem/kfree/enable

# فلتر على الـ caller (lib/group_cpus.c)
echo 'call_site == group_cpus_evenly' > events/kmem/kmalloc/filter

# تابع cpumask operations
echo 1 > events/sched/sched_cpu_mask/enable 2>/dev/null || true

# function tracer على group_cpus_evenly نفسها
echo function > current_tracer
echo group_cpus_evenly > set_ftrace_filter
echo __group_cpus_evenly >> set_ftrace_filter
echo alloc_nodes_groups >> set_ftrace_filter
echo 1 > tracing_on
# ... شغّل الكود اللي بيستدعي group_cpus_evenly ...
cat trace
echo 0 > tracing_on
echo nop > current_tracer
```

#### 4. printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لـ lib/group_cpus.c
echo 'file lib/group_cpus.c +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إن الـ debug messages اتفعلت
cat /sys/kernel/debug/dynamic_debug/control | grep group_cpus

# لو عايز تفعّل كل الـ memory subsystem debug
echo 'module slub +p' > /sys/kernel/debug/dynamic_debug/control

# في kernel cmdline لـ early debug
# dyndbg="file lib/group_cpus.c +p"
```

**ملاحظة:** الكود الحالي مش فيه `pr_debug()` calls صريحة، لكن الـ `WARN_ON_ONCE()` calls بتطلع تلقائياً في dmesg.

#### 5. Kernel Config Options للـ Debugging

| الـ Config | الغرض |
|---|---|
| `CONFIG_SMP` | لازم يكون enabled — بدونه الـ function بترجع mask واحد بس |
| `CONFIG_NUMA` | لتفعيل الـ NUMA-aware grouping |
| `CONFIG_DEBUG_SPINLOCK` | كشف race conditions في cpu hotplug |
| `CONFIG_LOCKDEP` | تتبع dependency بين الـ locks (مهم مع cpu hotplug) |
| `CONFIG_KASAN` | كشف out-of-bounds في cpumask arrays |
| `CONFIG_KMSAN` | كشف use of uninitialized cpumask bits |
| `CONFIG_DEBUG_PAGEALLOC` | كشف use-after-free في الـ masks المعادة |
| `CONFIG_SLUB_DEBUG` | تفاصيل أكتر عن كل kcalloc/kfree |
| `CONFIG_CPUMASK_OFFSTACK` | يخلي الـ cpumask_var_t على الـ heap بدل الـ stack |
| `CONFIG_PROVE_LOCKING` | يثبت صحة الـ locking order |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(SMP|NUMA|DEBUG_SPINLOCK|KASAN|LOCKDEP|CPUMASK)'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# irqbalance - الـ daemon اللي بيستخدم مبدأ مشابه
systemctl status irqbalance
cat /proc/irq/default_smp_affinity

# تحقق من توزيع الـ IRQs على الـ NVMe أو الـ network بعد group_cpus_evenly
ls /sys/block/nvme0n1/device/irq*
cat /sys/class/net/eth0/queues/rx-0/rps_cpus

# اعرض كل IRQ vectors لجهاز معين (مثلاً NVMe)
cat /proc/interrupts | grep nvme

# hwloc - أداة رائعة لفهم الـ topology
lstopo --of ascii
numactl --hardware
lscpu --extended
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `WARNING: CPU: X PID: Y at lib/group_cpus.c:201` | `WARN_ON_ONCE(numgrps == 0)` — وصلنا لـ node بدون groups متبقية | تحقق من الـ numgrps المُمرر — احتمال أقل من عدد الـ NUMA nodes |
| `WARNING: CPU: X PID: Y at lib/group_cpus.c:206` | `WARN_ON_ONCE(ngroups > ncpus)` — allocated أكتر groups من CPUs | bug في حساب الـ ratio — check الـ NUMA topology |
| `WARNING: CPU: X PID: Y at lib/group_cpus.c:390` | `WARN_ON_ONCE(nv->ngroups > nc)` — cluster groups > cluster CPUs | الـ cluster topology inconsistent |
| `WARNING: CPU: X PID: Y at lib/group_cpus.c:457` | `WARN_ON_ONCE(nv->ngroups > ncpus)` per-node check | نفس مشكلة الـ ratio على مستوى الـ node |
| `out of memory` في dmesg أثناء device probe | `kcalloc()` فشلت في `group_cpus_evenly` | نقص ذاكرة أو memory fragmentation — check `/proc/meminfo` |
| `group_cpus_evenly: numgrps=0` | المُستدعي بعت 0 كـ numgrps | bug في الـ caller — الـ function بترجع NULL مباشرة |

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

في حالة debugging مكثف، ضيف في `lib/group_cpus.c`:

```c
/* بعد كل alloc فاشل — لتتبع من استدعى الـ function */
static struct cpumask *group_cpus_evenly(unsigned int numgrps,
                                          unsigned int *nummasks)
{
    WARN_ON_ONCE(numgrps == 0);  /* موجودة بالفعل بشكل غير مباشر */

    /* أضف هنا لو شك في الـ caller */
    if (numgrps > num_possible_cpus()) {
        pr_warn("group_cpus_evenly: numgrps(%u) > possible_cpus(%u)\n",
                numgrps, num_possible_cpus());
        dump_stack();
    }
    ...

    /* بعد نهاية التوزيع — verify الـ coverage */
    /* كل CPU لازم يكون في exactly group واحدة */
}
```

```c
/* في alloc_groups_to_nodes - لو الـ math غلط */
WARN_ON_ONCE(ngroups == 0);
WARN_ON_ONCE(ngroups > ncpus);
```

---

### Hardware Level

#### 1. التحقق من أن الـ Hardware State يطابق الـ Kernel State

الـ `group_cpus_evenly` بتبني grouping بناءً على ما الـ kernel شايفه من topology — لازم يتطابق مع الـ hardware الحقيقي:

```bash
# مقارنة ما الـ kernel شايفه مقابل ما الـ BIOS بيعلن
# الـ kernel view
lscpu | grep -E 'Socket|Core|Thread|NUMA'

# BIOS/ACPI view
dmidecode --type processor | grep -E 'Core|Thread|Count'

# تحقق إن الـ HT siblings صح
# كل CPU لازم يشوف نفس الـ siblings
diff <(cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list) \
     <(cat /sys/devices/system/cpu/cpu1/topology/thread_siblings_list)

# تحقق إن الـ NUMA distances معقولة
numactl --hardware
cat /sys/devices/system/node/node*/distance
```

#### 2. Register Dump Techniques

الـ `group_cpus_evenly` نفسها مش بتتعامل مع hardware registers مباشرة، لكن الـ IRQ affinity اللي بتستخدمها بتكتب في APIC/MSI registers:

```bash
# قراءة MSI-X table لجهاز PCI (مثلاً NVMe)
# أول لازم تعرف الـ BAR للـ MSI-X table
lspci -vvv -s 0000:01:00.0 | grep -A5 MSI-X

# استخدام devmem2 لقراءة الـ MSI-X table (خطر — بس للـ debugging فقط)
# devmem2 <physical_address> w

# أو عبر /sys
cat /sys/bus/pci/devices/0000:01:00.0/msi_irqs/*

# APIC state
cat /proc/iomem | grep APIC
# استخدام x2apic registers (على x86)
rdmsr -a 0x802  # APIC ID register لكل CPU
```

#### 3. Logic Analyzer / Oscilloscope Tips

الـ `group_cpus_evenly` نفسها software-only، لكن التحقق من صحة الـ CPU grouping على مستوى الـ hardware:

- **لقياس الـ IRQ latency per group:** استخدم oscilloscope على الـ IRQ lines مع تتبع الـ CPU core اللي بيخدمها (عبر GPIO toggling في الـ ISR).
- **لتأكيد الـ NUMA locality:** قس الـ memory access latency باستخدام `perf mem` — مجموعة CPUs في نفس الـ group المفروض تبقى access times متقاربة.
- **لاختبار الـ cluster grouping:** راقب cache coherency traffic بين clusters — الـ groups الصح المفروض تقلل inter-cluster traffic.

```bash
# قياس NUMA memory latency
numactl --cpunodebind=0 --membind=0 -- perf bench mem memcpy
numactl --cpunodebind=0 --membind=1 -- perf bench mem memcpy
# الفرق بين النتيجتين يوضح الـ NUMA penalty

# قياس cache locality للـ groups
perf stat -e cache-misses,cache-references -C 0,1,2,3 -- sleep 1
```

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة | الأعراض في dmesg | طريقة التشخيص |
|---|---|---|
| NUMA topology غلط من BIOS | `WARN_ON_ONCE` في `alloc_groups_to_nodes` — ngroups > ncpus | `dmidecode --type 17` مقابل `numactl --hardware` |
| Hyperthreading معطل في BIOS لكن الـ kernel مش عارف | `thread_siblings_list` بيرجع كل CPU وحده | `cat /sys/devices/system/cpu/cpu0/topology/thread_siblings_list` يرجع `0` بس |
| CPU hotplug race مع group_cpus_evenly | IRQ affinity mask فيها CPUs offline | `cat /sys/devices/system/cpu/online` بعد كل hotplug |
| Cluster topology غير مدعوم على الـ CPU | `topology_cluster_cpumask()` بترجع empty mask | `cat /sys/devices/system/cpu/cpu0/topology/cluster_cpus_list` يرجع فاضي |
| ACPI مش بيعلن عن كل الـ NUMA nodes | groups مش موزعة بالتساوي على الـ sockets | `acpidump -n SRAT | acpixtract -a && iasl -d SRAT.dat` |

#### 5. Device Tree Debugging (ARM/ARM64 Systems)

على الـ embedded systems (ARM)، الـ CPU topology بتيجي من الـ DT:

```bash
# تحقق من الـ DT nodes للـ CPU topology
cat /sys/firmware/devicetree/base/cpus/cpu@0/cpu-map 2>/dev/null

# اعرض الـ cluster structure من DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A10 'cluster'

# مقارنة DT clusters مع ما الـ kernel شايفه
for cpu in /sys/devices/system/cpu/cpu*/topology/cluster_cpus_list; do
    echo "$cpu: $(cat $cpu)"
done
```

**مشكلة شائعة:** الـ DT بيعلن عن clusters لكن الـ kernel لم يكتشفها — لأن `topology_cluster_cpumask()` بترجع empty mask لو الـ arch لم تـ implement الـ `arch_get_cluster_cpumask()`.

```bash
# تحقق إن الـ arch بتدعم cluster topology
grep -r 'topology_cluster_cpumask\|cluster_cpus' /sys/devices/system/cpu/ 2>/dev/null
# لو مفيش output يعني الـ cluster-aware grouping مش شغال
```

---

### Practical Commands

#### أوامر جاهزة للنسخ والتشغيل

**1. عرض كامل للـ CPU grouping الحالي (لجهاز NVMe مثلاً):**

```bash
#!/bin/bash
DEV="nvme0"
echo "=== IRQ Groups for $DEV ==="
for irq in $(grep $DEV /proc/interrupts | awk '{print $1}' | tr -d ':'); do
    aff=$(cat /proc/irq/$irq/smp_affinity_list 2>/dev/null)
    name=$(cat /proc/irq/$irq/actions 2>/dev/null | head -1)
    echo "IRQ $irq ($name): CPUs [$aff]"
done
```

**مثال على الـ output وكيف تفسره:**
```
=== IRQ Groups for nvme0 ===
IRQ 24 (nvme0q1): CPUs [0-3]
IRQ 25 (nvme0q2): CPUs [4-7]
IRQ 26 (nvme0q3): CPUs [8-11]
IRQ 27 (nvme0q4): CPUs [12-15]
```
ده معناه إن `group_cpus_evenly(4, ...)` وزّعت الـ 16 CPU على 4 groups كل group 4 CPUs — توزيع سليم.

---

**2. تحقق من الـ NUMA locality للـ groups:**

```bash
#!/bin/bash
echo "=== NUMA Node per IRQ Group ==="
for irq_dir in /proc/irq/*/smp_affinity_list; do
    irq=$(echo $irq_dir | grep -o '[0-9]*')
    cpus=$(cat $irq_dir 2>/dev/null)
    [ -z "$cpus" ] && continue
    # خذ أول CPU في الـ mask
    first_cpu=$(echo $cpus | cut -d'-' -f1 | cut -d',' -f1)
    node=$(cat /sys/devices/system/cpu/cpu${first_cpu}/topology/physical_package_id 2>/dev/null)
    echo "IRQ $irq: CPUs[$cpus] -> Socket $node"
done
```

---

**3. اكتشاف مشاكل الـ grouping:**

```bash
#!/bin/bash
# تحقق إن مفيش CPU مكرر في أكتر من group
echo "=== Checking for CPU overlaps in IRQ affinities ==="
declare -A cpu_irq_map
for irq_dir in /proc/irq/*/; do
    irq=$(basename $irq_dir)
    mask=$(cat ${irq_dir}smp_affinity_list 2>/dev/null)
    [ -z "$mask" ] && continue
    # حوّل الـ list لـ individual CPUs
    for cpu in $(seq 0 $(nproc --all)); do
        if taskset -c $cpu true 2>/dev/null; then
            if [[ "${cpu_irq_map[$cpu]}" ]]; then
                echo "WARNING: CPU $cpu appears in both IRQ ${cpu_irq_map[$cpu]} and IRQ $irq"
            fi
        fi
    done
done
echo "Done."
```

---

**4. ftrace كامل لتتبع استدعاء group_cpus_evenly:**

```bash
#!/bin/bash
TRACE=/sys/kernel/debug/tracing

# reset
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# إعداد الـ function graph tracer
echo function_graph > $TRACE/current_tracer
echo group_cpus_evenly > $TRACE/set_graph_function

# فعّل
echo 1 > $TRACE/tracing_on

# شغّل الـ trigger — مثلاً probe جهاز NVMe
modprobe -r nvme; modprobe nvme

# وقف وعرض
echo 0 > $TRACE/tracing_on
head -100 $TRACE/trace

# cleanup
echo nop > $TRACE/current_tracer
echo > $TRACE/set_graph_function
```

**مثال على الـ output:**
```
# CPU  DURATION               FUNCTION CALLS
# |     |   |                  |   |   |   |
 0)               |  group_cpus_evenly() {
 0)   0.521 us    |    zalloc_cpumask_var();
 0)   0.318 us    |    zalloc_cpumask_var();
 0)   1.243 us    |    alloc_node_to_cpumask();
 0)   0.892 us    |    build_node_to_cpumask();
 0)   2.105 us    |    __group_cpus_evenly();  /* stage 1: present CPUs */
 0)   0.441 us    |    __group_cpus_evenly();  /* stage 2: possible CPUs */
 0) + 6.234 us    |  }
```

---

**5. مراقبة الـ WARN_ON في realtime:**

```bash
# راقب الـ dmesg للـ WARN_ON من group_cpus.c
dmesg -W | grep -E 'group_cpus|WARNING.*group'

# أو بشكل أكثر تفصيلاً
dmesg --follow --decode | grep -A5 'lib/group_cpus'
```

---

**6. التحقق من الـ cluster topology على ARM:**

```bash
# اعرض الـ cluster لكل CPU
for cpu in $(seq 0 $(($(nproc --all) - 1))); do
    cluster=$(cat /sys/devices/system/cpu/cpu${cpu}/topology/cluster_cpus_list 2>/dev/null || echo "N/A")
    node=$(cat /sys/devices/system/cpu/cpu${cpu}/topology/physical_package_id 2>/dev/null || echo "?")
    echo "CPU$cpu: NUMA=$node, Cluster=[$cluster]"
done
```

**مثال على الـ output على ARM big.LITTLE:**
```
CPU0: NUMA=0, Cluster=[0-3]     # little cores cluster
CPU1: NUMA=0, Cluster=[0-3]
CPU2: NUMA=0, Cluster=[0-3]
CPU3: NUMA=0, Cluster=[0-3]
CPU4: NUMA=0, Cluster=[4-7]     # big cores cluster
CPU5: NUMA=0, Cluster=[4-7]
CPU6: NUMA=0, Cluster=[4-7]
CPU7: NUMA=0, Cluster=[4-7]
```
الـ `group_cpus_evenly(8, ...)` المفروض ترجع 8 groups — كل CPU في group لوحده، مع cluster locality preserved.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: NVMe لـ Latency عالية على سيرفر NUMA بـ 4 سوكيت

#### العنوان
توزيع IRQ vectors غلط على سيرفر NUMA بـ AMD EPYC بيخلي NVMe بطيء

#### السياق
سيرفر data center بـ AMD EPYC 7763 — 4 NUMA nodes، كل node فيه 16 core (64 core إجمالي). الـ NVMe SSD بيطلب 32 MSI-X interrupt vector. الـ driver بيستخدم `group_cpus_evenly(32, &nummasks)` عشان يوزع الـ vectors على الـ CPUs.

#### المشكلة
الـ I/O latency على نص الـ queues أعلى بشكل غير طبيعي — P99 latency بيوصل لـ 800µs بدل 120µs. الـ profiling بيكشف إن interrupts من node 0 بتتعالج على CPUs في node 3.

#### التحليل
في `group_cpus_evenly`:
```c
// المشكلة: الـ node_to_cpumask اتبنى صح، لكن
// alloc_nodes_groups حسبت numgrps = min(numcpus, numgrps)
// = min(64, 32) = 32 — صح لحد هنا

// لكن في alloc_groups_to_nodes، الـ sort بيرتب الـ nodes
// حسب ncpus ascending: كل node فيه 16 CPU
// node 0: ngroups = max(1, 32 * 16 / 64) = 8
// node 1: ngroups = max(1, 24 * 16 / 48) = 8
// ...صح رياضياً
```
المشكلة مش في الكود نفسه — المشكلة إن الـ `startgrp` في الـ driver كان بيبدأ من آخر group محجوز بدل ما يبدأ من 0، فـ `curgrp` كان بيعدي `last_grp` ويعمل wraparound غلط، فالـ groups اللي المفروض تخص node 0 اتعملها assign لـ CPUs من node 3.

في `assign_cpus_to_groups`:
```c
if (*curgrp >= last_grp)
    *curgrp = 0;  // الـ wraparound صح هنا
grp_spread_init_one(&masks[*curgrp], nmsk, cpus_per_grp);
```
الـ driver كان بيمرر `startgrp = numgrps - 1` بدل `startgrp = 0`.

#### الحل
```bash
# أولاً: تشخيص توزيع الـ IRQs الحالي
for i in $(ls /proc/irq/); do
    cat /proc/irq/$i/smp_affinity_list 2>/dev/null | \
    awk -v irq=$i '{print "IRQ " irq ": CPUs " $0}'
done | head -40

# مقارنة الـ NUMA node لكل CPU group
numactl --hardware
cat /sys/bus/pci/devices/0000:01:00.0/msi_irqs/*/affinity_hint
```
الحل في الـ driver: تمرير `startgrp = 0` لـ `__group_cpus_evenly`، أو إعادة تعيين الـ IRQ affinity يدوياً:
```bash
# ربط queue 0-7 بـ node 0 (CPUs 0-15)
for q in $(seq 0 7); do
    echo 0-15 > /proc/irq/$((BASE_IRQ + q))/smp_affinity_list
done
```

#### الدرس المستفاد
**الـ `group_cpus_evenly` بيرجع masks صح، لكن الـ `startgrp` اللي بيمرره الـ driver بيحدد من أين يبدأ الـ assignment.** أي driver بيستخدم الـ API لازم يبدأ من 0 ما لم يكن عنده سبب وجيه للـ offset، وإلا الـ NUMA locality راح كلها.

---

### السيناريو الثاني: RK3562 — Wi-Fi Driver بيعمل IRQ على Core واحد بس

#### العنوان
الـ Wi-Fi على industrial gateway بـ RK3562 بيستخدم core واحد بس من أصل 4

#### السياق
gateway صناعي بـ RK3562 (Cortex-A53 quad-core، single NUMA node). الـ Wi-Fi chip هو RTL8852BE PCIe — بيطلب 4 MSI-X vectors. الـ kernel driver بيطلب `group_cpus_evenly(4, &nummasks)`.

#### المشكلة
الـ Wi-Fi throughput لا يتخطى 80 Mbps مع إن الـ chip يقدر يعمل 300+ Mbps. `htop` بيوري إن IRQ handling كله على CPU0 بس.

#### التحليل
RK3562 بيعمل compile بـ `CONFIG_SMP=y` لكن الـ NUMA disabled — يعني `nr_node_ids = 1`.

في `__group_cpus_evenly`:
```c
nodes = get_nodes_in_cpumask(node_to_cpumask, cpu_mask, &nodemsk);
// nodes = 1 (single NUMA node)

if (numgrps <= nodes) {  // 4 <= 1 ؟ لا — نعدي
    ...
}
// نكمل لـ alloc_nodes_groups
```

المشكلة اكتُشفت في `grp_spread_init_one`:
```c
// topology_sibling_cpumask(cpu) على RK3562 بترجع
// كل الـ 4 CPUs في نفس الـ mask (SMT غير موجود،
// لكن الـ DT كان مضبوط غلط)
siblmsk = topology_sibling_cpumask(cpu);
// siblmsk = {0,1,2,3} للـ CPU0
// فـ cpus_per_grp = 1، بس الـ sibling loop
// بتاخد CPU1 و CPU2 و CPU3 في نفس الـ group!
```
الـ DT كان فيه `cpu-map` غلط بيجعل الـ kernel يشوف الـ 4 cores كـ siblings في نفس الـ cluster.

#### الحل
تصحيح الـ Device Tree:
```dts
/* غلط — بيخلي كل الـ cores siblings */
cpu-map {
    cluster0 {
        core0 { cpu = <&cpu0>; };
        core1 { cpu = <&cpu1>; };
        core2 { cpu = <&cpu2>; };
        core3 { cpu = <&cpu3>; };
    };
};

/* صح — كل core في cluster منفصل */
cpu-map {
    cluster0 { core0 { cpu = <&cpu0>; }; };
    cluster1 { core0 { cpu = <&cpu1>; }; };
    cluster2 { core0 { cpu = <&cpu2>; }; };
    cluster3 { core0 { cpu = <&cpu3>; }; };
};
```
```bash
# تحقق بعد التصحيح
cat /sys/devices/system/cpu/cpu0/topology/core_siblings_list
# يجب أن يرجع: 0 (مش 0-3)
```

#### الدرس المستفاد
**`grp_spread_init_one` بتستخدم `topology_sibling_cpumask` للـ optimization، لكن لو الـ DT غلط والـ siblings mask شاملة كل الـ cores، الـ function هتحشر كل الـ CPUs في group واحد.** دايماً تحقق من `sysfs topology` قبل تشخيص الـ IRQ spreading.

---

### السيناريو الثالث: STM32MP1 — IRQ لـ SPI Controller على Core غلط بعد CPU Hotplug

#### العنوان
SPI master على STM32MP1 بيتوقف عن الشغل بعد ما CPU1 يتعمله offline

#### السياق
IoT sensor node بـ STM32MP157 (dual Cortex-A7 + M4 coprocessor). الـ SPI controller بيستخدم DMA مع interrupt. الـ power management كود بيعمل CPU1 offline في الـ idle mode لتوفير الطاقة.

#### المشكلة
بعد ما CPU1 يتعمله `echo 0 > /sys/devices/system/cpu/cpu1/online`، الـ SPI transactions بتتوقف وبتطلع `spi_transfer timeout` في الـ kernel log.

#### التحليل
الـ SPI driver بيطلب `group_cpus_evenly(2, &nummasks)` عند الـ probe:
```c
masks = group_cpus_evenly(2, &nummasks);
// masks[0] = {cpu0}, masks[1] = {cpu1}
// الـ IRQ الخاص بـ SPI اتعمله affinity لـ masks[1] = CPU1
```

في `group_cpus_evenly`:
```c
// المرحلة الأولى: present CPUs
cpumask_copy(npresmsk, data_race(cpu_present_mask));
// npresmsk = {0, 1} — كلاهم present وقت الـ probe

ret = __group_cpus_evenly(curgrp, numgrps, ...);
// masks[0] = {0}, masks[1] = {1}
```

الـ driver بعدين بيعمل:
```c
irq_set_affinity(spi_irq, &masks[1]);  // ربط بـ CPU1
```

لما CPU1 يتعمله offline، الـ kernel بيحاول ينقل الـ IRQ أوتوماتيك، لكن لو الـ `IRQF_NO_BALANCING` كان set، الـ migration ما بتحصلش.

#### الحل
الـ driver المفروض يستخدم `managed IRQ affinity` أو يستمع لـ CPU hotplug notifier:
```c
/* بدل ما يعمل affinity ثابت، يستخدم managed affinity */
irq_set_affinity_hint(spi_irq, &masks[1]);
/* ده بيسمح للـ kernel يعمل migration أوتوماتيك */

/* أو: إضافة hotplug callback */
static int spi_cpu_dead(unsigned int cpu)
{
    if (cpumask_test_cpu(cpu, &current_affinity))
        rebalance_spi_irq();
    return 0;
}
cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "spi:online",
                  NULL, spi_cpu_dead);
```
```bash
# تشخيص: هل الـ IRQ اتنقل؟
watch -n1 'cat /proc/interrupts | grep spi'
# لو الـ count واقف، الـ IRQ stuck على CPU offline
```

#### الدرس المستفاد
**`group_cpus_evenly` بتحسب الـ masks وقت الاستدعاء بس — هي مش aware بالـ hotplug اللي بيحصل بعدين.** الـ drivers اللي بتستخدم الـ API لازم تتعامل مع CPU hotplug events أو تستخدم `irq_set_affinity_hint` بدل الـ hard affinity.

---

### السيناريو الرابع: i.MX8QM — USB 3.0 Performance سيئة بسبب Cluster Topology غير مكتشف

#### العنوان
USB 3.0 throughput على i.MX8QM ما بيوصلش للـ 5 Gbps المتوقعة

#### السياق
automotive ECU بـ NXP i.MX8QM — بروسيسور heterogeneous: cluster A بـ Cortex-A72 (2 cores، عالية الأداء)، cluster B بـ Cortex-A53 (4 cores، موفرة للطاقة). الـ USB xHCI controller بيطلب 6 MSI-X vectors.

#### المشكلة
نقل ملفات USB 3.0 بيوصل لـ 2.1 Gbps بس. الـ perf stat بيوري إن نص الـ USB IRQs بتتعالج على A53 cores اللي latency فيها أعلى.

#### التحليل
الـ driver بيطلب `group_cpus_evenly(6, &nummasks)`.

في `alloc_cluster_groups`:
```c
/* الكود بيحاول يكتشف الـ clusters */
while (1) {
    cpu = cpumask_first(msk);
    cluster_mask = topology_cluster_cpumask(cpu);
    if (!cpumask_weight(cluster_mask))
        goto no_cluster;  // ← هنا المشكلة!
    ...
}
```

على i.MX8QM مع kernel قديم (< 5.15)، الـ `topology_cluster_cpumask` مكنتش مدعومة صح — بترجع empty mask. فالكود بيعمل `goto no_cluster` ويسقط للـ node-level grouping العادي.

في `__group_cpus_evenly`، بما إن في NUMA node واحد:
```c
// 6 groups على 6 CPUs
// grp_spread_init_one بتوزع بالترتيب
// CPU0(A72), CPU1(A72), CPU2(A53), CPU3(A53), CPU4(A53), CPU5(A53)
// groups 0,1 = A72 cores
// groups 2,3,4,5 = A53 cores
// الـ xHCI driver بياخد أول group متاح — ممكن يقع على A53
```

#### الحل
الـ kernel >= 6.1 بيدعم cluster topology صح على i.MX8QM. كـ workaround للـ kernel القديم:
```bash
# تعيين أولوية الـ USB IRQs للـ A72 cores يدوياً
USB_IRQS=$(cat /proc/interrupts | grep xhci | awk '{print $1}' | tr -d ':')
for irq in $USB_IRQS; do
    echo 0-1 > /proc/irq/$irq/smp_affinity_list
done

# تحقق من الـ cluster topology المكتشفة
for cpu in /sys/devices/system/cpu/cpu*/topology/; do
    echo -n "CPU $(basename $(dirname $cpu)): cluster_id="
    cat ${cpu}cluster_id 2>/dev/null || echo "N/A"
done
```

للـ driver level، إضافة `irq_create_affinity_masks` مع hints:
```c
struct irq_affinity affd = {
    .pre_vectors = 1,   /* keep 1 vector for general use */
};
/* xhci_pci_setup عندها option لـ affinity hints */
pci_alloc_irq_vectors_affinity(pdev, 1, 6,
    PCI_IRQ_MSI | PCI_IRQ_AFFINITY, &affd);
```

#### الدرس المستفاد
**الـ `alloc_cluster_groups` بتعتمد على `topology_cluster_cpumask` — لو الـ SoC vendor ما حطش الـ cluster topology صح في الـ DT أو الـ kernel ما بيدعمهاش، الكود بيسقط gracefully للـ node grouping العادي.** لازم تتحقق من `/sys/devices/system/cpu/cpuX/topology/cluster_id` عشان تعرف هل الـ cluster awareness شغالة.

---

### السيناريو الخامس: Allwinner H616 — Android TV Box بـ IRQ Spreading غلط على Big.LITTLE

#### العنوان
جهاز Android TV بـ H616 بيعمل audio glitch تحت load بسبب توزيع IRQ غلط

#### السياق
Android TV box بـ Allwinner H616 (Cortex-A53 quad-core، single cluster من وجهة نظر الـ NUMA). الـ HDMI audio controller بيستخدم interrupt. الـ multimedia pipeline فيها 4 IRQ vectors موزعة بـ `group_cpus_evenly(4, &nummasks)`.

#### المشكلة
تحت load ثقيل (4K video + network)، في audio dropout كل دقيقتين تقريباً. الـ `top` بيوري CPU0 على 100% أحياناً بينما CPU2 و CPU3 فاضيين.

#### التحليل
H616 بيستخدم kernel مع `CONFIG_NUMA` disabled و`CONFIG_SMP=y`. الـ 4 cores كلهم في نفس الـ cluster.

في `group_cpus_evenly(4, &nummasks)`:
```c
// المرحلة الأولى: present CPUs = {0,1,2,3}
// __group_cpus_evenly(0, 4, ..., npresmsk={0,1,2,3}, ...)

// nodes = 1 (no NUMA)
// numgrps (4) > nodes (1) → نكمل

// alloc_nodes_groups:
// node 0 gets all 4 groups

// في alloc_cluster_groups:
// topology_cluster_cpumask(cpu0) على H616 بترجع {0,1,2,3}
// ncluster = 1 (كل الـ cores في cluster واحد)
// ncluster (1) <= ngroups (4) → نكمل

// alloc_groups_to_nodes(4, 4, cluster_groups, 1):
// cluster 0: ngroups = max(1, 4*4/4) = 4

// assign_cpus_to_groups:
// ncpus = 4, ngroups = 4
// cpus_per_grp = 1 لكل group
// grp_spread_init_one للـ group 0:
//   cpu = cpumask_first(nmsk) = 0
//   cpumask_set_cpu(0, irqmsk[0])  ← أضاف CPU0
//   siblmsk = topology_sibling_cpumask(0) = {0,1,2,3}
//   cpus_per_grp = 0 بعد CPU0 — الـ loop خلصت
// → masks[0]={0}, masks[1]={1}, masks[2]={2}, masks[3]={3} ✓
```

الـ masks صح! المشكلة مش في `group_cpus_evenly`. المشكلة إن الـ Android vendor kernel كان بيستخدم `irq_balance` disabled، والـ HDMI driver كان بيعمل `irq_set_affinity` لكل الـ vectors على masks[0] فقط بسبب bug في الـ index calculation.

```c
/* الكود الغلط في الـ vendor driver */
for (i = 0; i < num_vectors; i++) {
    /* BUG: always uses masks[0] */
    irq_set_affinity(irqs[i], &masks[0]);
}

/* الصح */
for (i = 0; i < num_vectors; i++) {
    irq_set_affinity(irqs[i], &masks[i % nummasks]);
}
```

#### الحل
```bash
# تشخيص: عرض الـ affinity الحالية لكل IRQ
for irq in $(cat /proc/interrupts | grep hdmi | awk '{print $1}' | tr -d ':'); do
    echo -n "IRQ $irq affinity: "
    cat /proc/irq/$irq/smp_affinity_list
done
# لو كل الـ IRQs على 0 → هذه هي المشكلة

# حل مؤقت: توزيع يدوي
HDMI_IRQS=($(cat /proc/interrupts | grep hdmi | awk '{print $1}' | tr -d ':'))
for i in "${!HDMI_IRQS[@]}"; do
    echo $i > /proc/irq/${HDMI_IRQS[$i]}/smp_affinity_list
done

# تشغيل irqbalance كـ workaround
apt install irqbalance
systemctl start irqbalance
```

تصحيح الـ driver:
```c
/* في دالة الـ probe، بعد group_cpus_evenly */
masks = group_cpus_evenly(num_vectors, &nummasks);
if (masks) {
    for (i = 0; i < num_vectors; i++) {
        /* استخدم modulo عشان تتعامل مع nummasks < num_vectors */
        irq_set_affinity(irqs[i], &masks[i % nummasks]);
    }
}
```

#### الدرس المستفاد
**`group_cpus_evenly` ممكن تشتغل صح تماماً، لكن الـ bug يكون في كيفية استخدام الـ caller للـ masks الراجعة.** دايماً استخدم `i % nummasks` مش `i` مباشرة، وتحقق من الـ `smp_affinity_list` لكل IRQ بعد الـ setup عشان تتأكد إن التوزيع صح.
## Phase 7: مصادر ومراجع

### مقدمة

الـ `group_cpus_evenly()` function موجودة في `include/linux/group_cpus.h` وتتبع الـ implementation في `lib/group_cpus.c`، بتاريخ من 2016 بأيدي **Thomas Gleixner** و**Christoph Hellwig**. المصادر دي هتساعدك تتعمق أكتر في الـ IRQ affinity، الـ cpumask، والـ CPU topology spreading اللي الـ function دي جزء منهم.

---

### مقالات LWN.net

| المقال | الأهمية |
|--------|---------|
| [automatic interrupt affinity for MSI/MSI-X capable devices V3](https://lwn.net/Articles/693653/) | أصل الفكرة — automatic spreading لـ MSI vectors على CPUs مختلفة |
| [automatic IRQ affinity for virtio V2](https://lwn.net/Articles/712773/) | تطبيق الـ spreading على virtio devices |
| [kernel/irq: allow more precise irq affinity policies](https://lwn.net/Articles/406700/) | نقاش مبكر لسياسات الـ affinity |
| [pci: automatic interrupt affinity for MSI/MSI-X V2](https://lwn.net/Articles/694378/) | iteration ثانية للـ MSI affinity automation |
| [x86: Rework the vector management](https://lwn.net/Articles/733618/) | إعادة هيكلة إدارة الـ interrupt vectors على x86 |
| [irq: sysfs interface improvements for SMP affinity control](https://lwn.net/Articles/933315/) | تحسينات الـ sysfs للتحكم في الـ affinity |
| [cpumask conversion patches for sched](https://lwn.net/Articles/308300/) | تحويل كود الـ scheduler لاستخدام الـ cpumask API الجديدة |

---

### التوثيق الرسمي داخل الـ kernel

```
Documentation/core-api/irq/irq-affinity.rst   ← SMP IRQ affinity الرئيسي
Documentation/core-api/genericirq.rst          ← Generic IRQ handling framework
Documentation/admin-guide/kernel-parameters.txt ← irqaffinity= boot param
Documentation/admin-guide/cgroup-v1/cpusets.rst ← cpusets وعلاقتها بالـ affinity
```

- **الرابط المباشر للتوثيق**: [SMP IRQ affinity — kernel docs](https://docs.kernel.org/core-api/irq/irq-affinity.html)
- **Generic IRQ handling**: [static.lwn.net/kerneldoc/core-api/genericirq.html](https://static.lwn.net/kerneldoc/core-api/genericirq.html)
- **IRQ-affinity.txt (قديم)**: [kernel.org/doc/Documentation/IRQ-affinity.txt](https://www.kernel.org/doc/Documentation/IRQ-affinity.txt)

---

### الـ source files المباشرة في الـ kernel

| الملف | الدور |
|-------|-------|
| `lib/group_cpus.c` | الـ implementation الكاملة لـ `group_cpus_evenly()` |
| `include/linux/group_cpus.h` | الـ header — الملف اللي بندرسه |
| `kernel/irq/affinity.c` | المستخدم الرئيسي للـ function — بيبني affinity masks |
| `include/linux/cpumask.h` | الـ cpumask API اللي بتعتمد عليها الـ function |
| `include/linux/cpu.h` | CPU hotplug و topology APIs |

**الرابط على GitHub:**
[kernel/irq/affinity.c على torvalds/linux](https://github.com/torvalds/linux/blob/master/kernel/irq/affinity.c)

---

### كومِتات مهمة في الـ kernel

| الكومِت | الوصف |
|---------|-------|
| `f7b3ea8cf72f3` | `genirq/affinity: Move group_cpus_evenly() into lib/` — نقل الـ function من genirq إلى lib/ |
| `841c35169323` | تعديل في `lib/group_cpus.c` — موجود على [Software Heritage](https://archive.softwareheritage.org/browse/revision/841c35169323cd833294798e58b9bf63fa4fa1de/?path=lib/group_cpus.c) |

---

### نقاشات الـ mailing list

الـ `group_cpus_evenly()` فيها سلسلة patches طويلة على **linux-block@vger.kernel.org**:

| الرابط | الموضوع |
|--------|---------|
| [PATCH v7 01/10 — group_masks_cpus_evenly()](https://www.mail-archive.com/linux-block@vger.kernel.org/msg43376.html) | إضافة variant جديدة تعيد عدد الـ masks المُهيَّأة |
| [PATCH — let group_cpu_evenly return number of groups](https://www.mail-archive.com/linux-block@vger.kernel.org/msg42688.html) | تغيير الـ API لإرجاع عدد المجموعات |
| [PATCH — lib/group_cpus: make group CPU cluster aware](http://www.mail-archive.com/linux-block@vger.kernel.org/msg44047.html) | جعل الـ grouping يراعي الـ CPU clusters |
| [LKML: Revert "avoid acquiring cpu hotplug lock"](https://lkml.org/lkml/2026/2/26/1057) | revert لتعديل الـ hotplug lock بعد اكتشاف مشاكل |
| [Patchew: avoid to acquire cpu hotplug lock in group_cpus_evenly](https://patchew.org/linux/20231120083559.285174-1-ming.lei@redhat.com/) | الـ patch الأصلي لتحسين الـ locking |
| [Kernel-managed IRQ affinity (mailing list)](https://lists.gt.net/linux/kernel/3554349) | نقاش عام حول الـ managed IRQ affinity |

---

### kernelnewbies.org — تغييرات ذات صلة عبر الإصدارات

| الإصدار | الرابط | ما يخص موضوعنا |
|---------|--------|-----------------|
| Linux 6.6 | [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6) | تحسينات الـ IRQ affinity و cpumask |
| Linux 6.2 | [kernelnewbies.org/Linux_6.2](https://kernelnewbies.org/Linux_6.2) | تغييرات في إدارة الـ interrupt vectors |
| Linux 5.6 | [kernelnewbies.org/Linux_5.6](https://kernelnewbies.org/Linux_5.6) | `managed_irq` لحماية isolated CPUs |
| Linux 3.10 | [kernelnewbies.org/Linux_3.10](https://kernelnewbies.org/Linux_3.10) | NUMA affinity لـ unbound workqueues |
| KernelGlossary | [kernelnewbies.org/KernelGlossary](https://kernelnewbies.org/KernelGlossary) | مصطلحات الـ kernel العامة |

---

### elinux.org — مصادر embedded Linux

| الرابط | المحتوى |
|--------|---------|
| [Soft IRQ Threads — elinux.org](https://elinux.org/Soft_IRQ_Threads) | شرح الـ softirq threads وعلاقتها بالـ CPU affinity |
| [Ftrace — elinux.org](https://elinux.org/Ftrace) | أداة لتتبع الـ IRQ distribution على CPUs |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 10**: Interrupt Handling — شرح `request_irq()` و SMP affinity
- **الفصل 9**: Communicating with Hardware — أساسيات الـ IRQ
- **متاح مجاناً**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 7**: Interrupts and Interrupt Handlers
- **الفصل 9**: An Introduction to Kernel Synchronization — مهم لفهم الـ CPU hotplug lock في `group_cpus_evenly()`
- **الفصل 3**: Process Management — يفيد في فهم الـ CPU topology

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 14**: Kernel Debugging Techniques — تشمل tracing الـ IRQ affinity
- يُغطي الـ IRQ affinity في السياق الـ embedded وإدارة الـ real-time interrupts

---

### مصادر إضافية

#### توثيق رسمي خارجي
- [Real-time Ubuntu: How to tune IRQ affinity](https://documentation.ubuntu.com/real-time/latest/how-to/tune-irq-affinity/) — دليل عملي
- [Linux Foundation: Default IRQ affinity](https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cpu-partitioning/irqaffinity) — في سياق الـ PREEMPT_RT
- [SMP-affinity.txt — University of Waterloo](https://cs.uwaterloo.ca/~brecht/servers/apic/SMP-affinity.txt) — نسخة تاريخية من التوثيق

#### أدوات للتحقق العملي
```bash
# عرض الـ affinity الحالية لكل IRQ
cat /proc/irq/*/smp_affinity

# تغيير الـ affinity يدوياً
echo "03" > /proc/irq/42/smp_affinity

# عرض الـ affinity كـ CPU list
cat /proc/irq/*/smp_affinity_list

# تتبع توزيع الـ IRQs على CPUs
watch -n1 cat /proc/interrupts
```

---

### مصطلحات بحث للمزيد

للبحث في LKML و patchwork و git log:

```
group_cpus_evenly
irq_build_affinity_masks
irq_create_affinity_masks
cpumask_of_node NUMA IRQ spread
genirq affinity managed interrupts
lib/group_cpus.c
kernel/irq/affinity.c
MSI-X vector spreading CPU topology
cpu_present_mask hotplug affinity
```

للبحث على **git.kernel.org**:
```bash
git log --oneline -- lib/group_cpus.c
git log --oneline -- kernel/irq/affinity.c
git log --oneline -- include/linux/group_cpus.h
```
## Phase 8: Writing simple module

### الفكرة

الـ `group_cpus.h` بيعرّف function واحدة بس وهي `group_cpus_evenly()` — بتوزع الـ CPUs على عدد من الـ groups بالتساوي، وبترجع array من الـ `cpumask*`. الـ drivers اللي بتدعم multiple interrupt vectors (زي NVMe و SCSI) بتستخدمها عشان تعرف أي CPUs تـ affine لكل IRQ.

هنستخدم **kprobe** على `group_cpus_evenly` عشان نـ intercept أي call ليها ونطبع عدد الـ groups المطلوبة وعدد الـ CPUs الكلي — مفيد جداً للـ debugging لما driver بيطلب CPU grouping غريب.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_group_cpus.c
 * Intercepts group_cpus_evenly() calls and logs their arguments.
 */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit  */
#include <linux/kernel.h>       /* pr_info                           */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe    */
#include <linux/cpumask.h>      /* nr_cpu_ids                        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc");
MODULE_DESCRIPTION("kprobe on group_cpus_evenly — logs numgrps requests");

/* ------------------------------------------------------------------ */
/* pre-handler: بيتنفذ قبل ما group_cpus_evenly تبدأ                  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention:
     *   rdi = first arg  → numgrps   (unsigned int)
     *   rsi = second arg → nummasks* (pointer, we ignore its value here)
     */
    unsigned int numgrps = (unsigned int)regs->di;

    pr_info("group_cpus_evenly called: numgrps=%u, online_cpus=%u\n",
            numgrps, num_online_cpus());

    /*
     * لو الـ driver طلب groups أكتر من الـ CPUs الـ online
     * ده موقف مثير للاهتمام — نلفت انتباه الـ developer
     */
    if (numgrps > num_online_cpus())
        pr_warn("group_cpus_evenly: numgrps(%u) > online_cpus(%u)!\n",
                numgrps, num_online_cpus());

    return 0; /* 0 = kernel continues normal execution */
}

/* ------------------------------------------------------------------ */
/* post-handler: بيتنفذ بعد ما group_cpus_evenly ترجع                 */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * رجعنا بعد الـ function — نسجل إن الـ call انتهت.
     * الـ return value في rax، بس هنا بس نأكد إن الـ hook شغال.
     */
    pr_info("group_cpus_evenly returned (rax=0x%lx)\n", regs->ax);
}

/* ------------------------------------------------------------------ */
/* تعريف الـ kprobe struct                                             */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "group_cpus_evenly", /* الـ symbol اللي هنـ probe عليه */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init: بنسجل الـ kprobe                                       */
/* ------------------------------------------------------------------ */
static int __init kp_group_cpus_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: لازم نـ unregister عشان منسبّش dangling probe          */
/* ------------------------------------------------------------------ */
static void __exit kp_group_cpus_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe on group_cpus_evenly removed\n");
}

module_init(kp_group_cpus_init);
module_exit(kp_group_cpus_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات الـ module زي `MODULE_LICENSE` و `module_init/exit` |
| `linux/kernel.h` | `pr_info`, `pr_warn`, `pr_err` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/cpumask.h` | `num_online_cpus()` عشان نقارن بيه الـ numgrps |

#### الـ `handler_pre`

الـ `pt_regs` بيحمل state الـ CPU لحظة الـ probe — على x86-64 بنقرأ `regs->di` عشان ده أول argument (numgrps). بنطبع القيمة وبنحذر لو الـ driver طلب groups أكتر من الـ CPUs الـ online، وده يكشف bugs في الـ IRQ affinity setup.

#### الـ `handler_post`

بيتنفذ بعد رجوع الـ function مباشرةً — بنسجل الـ return value من `regs->ax` اللي هو الـ pointer للـ cpumask array. ده بيأكد إن الـ hook بيشتغل كـ pair مع الـ pre-handler.

#### الـ `struct kprobe`

الـ `.symbol_name` بيخلي الـ kernel يحول الاسم لـ address وقت الـ `register_kprobe` — مش محتاجين نعرف الـ address يدوياً. لو الـ symbol مش موجود (مثلاً `CONFIG_GROUP_CPUS` مش مفعّل) الـ register بيرجع error سلبي.

#### الـ `module_exit`

الـ `unregister_kprobe` ضروري جداً — لو نسيناه والـ module اتـ unload، الـ handler بيبقى مسجل على address إحنا محلناه من الـ memory وده kernel panic مضمون.

---

### تجربة الـ Module

```bash
# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod kprobe_group_cpus.ko

# اشغّل أي device بيستخدم group_cpus_evenly — مثلاً NVMe rescan
echo 1 | sudo tee /sys/bus/pci/rescan

# شوف الـ logs
sudo dmesg | grep group_cpus

# Unload
sudo rmmod kprobe_group_cpus
```

**مثال output متوقع:**

```
[  42.100] kprobe planted on group_cpus_evenly at ffffffffc0123456
[  42.200] group_cpus_evenly called: numgrps=4, online_cpus=8
[  42.201] group_cpus_evenly returned (rax=0xffff888003a40000)
```

---

### الـ Makefile

```makefile
obj-m += kprobe_group_cpus.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
