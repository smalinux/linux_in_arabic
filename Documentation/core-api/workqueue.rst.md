## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Workqueue؟

تخيل إنك بتشتغل في مطعم. الـ **cashier** بيستقبل الطلبات بسرعة من الزباين، بس مش هو اللي بيطبخ — هو بيكتب الطلب على ورقة ويحطها في **queue** (طابور)، والطباخين في المطبخ هما اللي بيشتغلوا على الطلبات دي واحدة واحدة.

الـ **Workqueue** في الـ Linux kernel هو نفس الفكرة بالظبط:
- الـ **cashier** = أي driver أو subsystem بيحتاج يشغّل كود في الـ background.
- الـ **ورقة** = الـ `work_struct` — struct صغيرة فيها pointer للـ function المطلوب تنفيذها.
- الـ **Queue** = الـ `workqueue_struct` — الطابور اللي بيستقبل الـ work items.
- الـ **طباخ** = الـ `kworker` thread — thread كيرنل بيشتغل على الـ work items.

---

### ليه محتاجين workqueue أصلاً؟

في الكيرنل في contexts بتشتغل فيها وانت **مش مسموحلك تنام** (زي الـ interrupt handlers و softirqs). لو جات ليك مهمة تاخد وقت (مثلاً: تكتب على disk أو تعمل network call)، مش هينفع تعملها في اللحظة دي.

الـ workqueue بيحل المشكلة دي: بدل ما تعمل الشغل الثقيل في الـ interrupt context، بتحط الـ work item في الـ queue، وبعدين يجي الـ kworker thread (اللي مسموحله ينام ويتحرك) ويشتغل عليه.

**مثال من الواقع:**
- الـ **network driver** بيستقبل packet (ده interrupt context — لازم يخلص بسرعة).
- بدل ما يعمل processing ثقيل في الـ interrupt، بيحط work item في الـ workqueue.
- الـ kworker بعدين بياخد الـ packet ويعمله processing كامل.

---

### المشكلة القديمة والحل الجديد (cmwq)

#### قبل الـ cmwq:
كل workqueue كان عنده workers خاصين بيه. لو عندك 10 workqueues و32 CPU، هيبقى عندك 320 kworker thread مش بتشتغل معظم الوقت — ده **هدر فاضح في الـ PIDs والـ memory**.

#### الحل: **Concurrency Managed Workqueue (cmwq)**
الـ cmwq بيعمل **shared worker pools** بين كل الـ workqueues. كل CPU عنده pool واحد للـ normal work وواحد للـ high priority. الـ workers بيتشاركوا بين كل الـ workqueues اللي على نفس الـ CPU.

```
قبل cmwq:
  wq_A  → [worker1, worker2, worker3]  ← per-CPU workers
  wq_B  → [worker4, worker5, worker6]  ← per-CPU workers
  wq_C  → [worker7, worker8, worker9]  ← per-CPU workers
  (ده تدمير في الـ resources)

بعد cmwq:
  CPU0_normal_pool → [kworker/0:0, kworker/0:1, kworker/0:2]
       ↑ مشترك بين wq_A و wq_B و wq_C و كل الـ workqueues
```

---

### إزاي الـ cmwq بيدير الـ Concurrency؟

الـ cmwq **بيتكلم مع الـ scheduler** عشان يعرف إمتى الـ worker بينام وإمتى بيصحى. القاعدة بسيطة:

> طالما في واحد على الأقل شغّال على الـ CPU، متبدأش worker جديد. لو آخر worker نام، ابدأ worker جديد فوراً عشان الـ CPU ما يبقاش باطل.

ده بيخلي عدد الـ workers **minimal لكن كافي** — مش أكتر من اللازم ومش أقل.

---

### أنواع الـ Workqueues

| النوع | الوصف | الـ Flag |
|-------|--------|---------|
| **Bound (per-CPU)** | الـ work بيتنفذ على نفس الـ CPU اللي queue عليه | افتراضي |
| **Unbound** | الـ work بيتنفذ على أي CPU | `WQ_UNBOUND` |
| **BH (Bottom Half)** | زي softirq — مينفعش ينام | `WQ_BH` |
| **High Priority** | بيتنفذ بـ elevated nice level | `WQ_HIGHPRI` |
| **Ordered** | work item واحد بس في كل وقت | `alloc_ordered_workqueue()` |
| **Freezable** | بيوقف أثناء system suspend | `WQ_FREEZABLE` |
| **MEM_RECLAIM** | ضروري لمسارات الـ memory reclaim | `WQ_MEM_RECLAIM` |

---

### الـ Structs الأساسية

**الـ `work_struct`** — اللبنة الأساسية:
```c
struct work_struct {
    atomic_long_t data;     /* flags + pointer to pool or pwq */
    struct list_head entry; /* قائمة الـ work items في الـ pool */
    work_func_t func;       /* الـ function اللي هتتنفذ */
};
```

**الـ `delayed_work`** — للـ work اللي عايزه يتأخر:
```c
struct delayed_work {
    struct work_struct work;
    struct timer_list timer;  /* timer بيتحول لـ work عند انتهاء الوقت */
    struct workqueue_struct *wq;
    int cpu;
};
```

**الـ `rcu_work`** — للـ work بعد RCU grace period:
```c
struct rcu_work {
    struct work_struct work;
    struct rcu_head rcu;
    struct workqueue_struct *wq;
};
```

---

### قصة: إزاي ممكن تتعمل deadlock بدون cmwq؟

تخيل الـ workqueue القديم:
1. الـ memory reclaim محتاج يشغّل work item **A**.
2. الـ work item **A** محتاج ينتظر work item **B** يخلص.
3. الـ work item **B** محتاج memory عشان يتنفذ.
4. الـ memory reclaim فاضل واقف مستنى **A**، و**A** مستنى **B**، و**B** مش هيجيله memory.

**Deadlock تام.**

الـ cmwq حل ده بـ **rescue workers**: كل workqueue عنده `WQ_MEM_RECLAIM` ليه worker محجوز خصيصاً يشتغل حتى لو الـ memory ضيقت — ده بيكسر الـ deadlock.

---

### الـ Affinity Scopes للـ Unbound Workqueues

الـ unbound workqueues بتجمع الـ CPUs في **pods** عشان تحسن الـ cache locality:

```
CACHE scope (الافتراضي):
  [CPU0, CPU1] ← يشاركوا L3 cache → pod واحد → worker pool واحد
  [CPU2, CPU3] ← يشاركوا L3 cache → pod واحد → worker pool واحد
```

لو الـ issuer على CPU0 وعمل queue work، الـ worker هيشتغل على CPU0 أو CPU1 (في نفس الـ L3) — الداتا هتبقى في الـ cache وما تحتاجش تتجيب تاني.

---

### الـ System Workqueues الجاهزة

الكيرنل بيجيب workqueues جاهزة تقدر تستخدمها من غير ما تعمل حاجة:

| الاسم | الاستخدام |
|-------|----------|
| `system_wq` | الـ `events` — الأكثر استخداماً |
| `system_highpri_wq` | الـ `events_highpri` |
| `system_long_wq` | للـ work اللي بياخد وقت طويل |
| `system_unbound_wq` | الـ `events_unbound` |
| `system_freezable_wq` | للـ suspend-aware work |
| `system_power_efficient_wq` | بيفضّل unbound على NUMA systems |

---

### الملفات الأساسية في الـ Subsystem

| الملف | الدور |
|-------|-------|
| `kernel/workqueue.c` | الـ implementation الكاملة — قلب الـ subsystem |
| `kernel/workqueue_internal.h` | الـ internal structs (`worker_pool`, `pool_workqueue`) |
| `include/linux/workqueue.h` | الـ public API — flags, macros, inline functions |
| `include/linux/workqueue_types.h` | الـ core types (`work_struct`, `work_func_t`) |
| `Documentation/core-api/workqueue.rst` | الملف ده — الـ documentation الرسمية |
| `tools/workqueue/wq_dump.py` | أداة لعرض الـ pools والـ affinity configuration |
| `tools/workqueue/wq_monitor.py` | أداة لـ monitoring الـ workqueue operations |

---

### ملفات تانية مهمة تعرفها

- **`kernel/softirq.c`** — الـ BH workqueues بتنفذ في الـ softirq context اللي اتعمل هنا.
- **`include/linux/timer.h`** — الـ `delayed_work` بيستخدم الـ timer subsystem.
- **`kernel/sched/core.c`** — الـ cmwq بيتكامل مع الـ scheduler عشان يدير الـ concurrency.
- **`mm/vmscan.c`** — مثال على استخدام `WQ_MEM_RECLAIM` في مسارات الـ memory reclaim.
## Phase 2: شرح الـ Workqueue Framework

### المشكلة — ليه الـ Workqueue موجود أصلاً؟

في الـ kernel، كتير من الأحداث بتحصل في سياق غير مناسب للتنفيذ الكامل — مثلاً interrupt handler بيشتغل وهو **preemption مقفول**، مش قادر ينام، ومش قادر يعمل عمليات بتاخد وقت زي الـ I/O أو allocation. المشكلة إن في شغل **مش لازم يتعمل دلوقتي** وفي نفس الوقت مش ينفع يتعمل هنا.

الحل البدائي كان كل subsystem يعمل **kthread** خاص بيه يستنى الشغل. النتيجة؟ الـ kernel بدأ يفقس threads بالجملة — كل driver، كل subsystem، كل wq كانت بتخلق thread جديد. في أنظمة بـ 32+ CPU كان الـ 32k PID space بتتملا من غير ما الجهاز يشتغل كويس.

المشكلة التانية: حتى لما الـ thread موجود، الـ concurrency كانت بائسة. كل wq عندها pool منفصل — يعني MT workqueue بتديك **thread واحد بس per-CPU**. لو الـ work item نام (ينتظر I/O مثلاً) اتقفل الـ CPU كله.

---

### الحل — الـ Concurrency Managed Workqueue (cmwq)

الـ cmwq أعاد رسم الصورة بالكامل بفكرة بسيطة وعميقة:

> افصل بين الـ **workqueue** (اللي الـ drivers بتتكلم معاه) وبين الـ **worker-pool** (اللي بينفذ الشغل فعلاً).

بدل ما كل wq تملك workers خاصين بها، الـ workers بقوا **shared** — pool واحد per-CPU بيخدم كل الـ workqueues. الـ kernel نفسه بيدير عدد الـ workers ديناميكياً حسب احتياج الـ concurrency.

---

### التشبيه الواقعي — المطبخ والـ Tickets

تخيل مطعم فيه:

| عنصر في المطبخ | المقابل في cmwq |
|---|---|
| **الطلب (Order Ticket)** | `struct work_struct` — الوحدة الأساسية للشغل |
| **لوحة الأوردرات** (order board) | الـ **workqueue** — queue الـ driver بيحط فيها الطلبات |
| **الطباخين** (cooks) | الـ **kworker threads** |
| **المطبخ كله** | الـ **worker-pool** |
| **مدير المطبخ** | الـ **cmwq concurrency manager** |

في المطعم القديم: كل أوردر بورد عندها طباخ خاص بيها. لو الطباخ ينتظر اللحمة تطلع من الفرن — بيقعد باطل. المدير الجديد (cmwq) عنده **مطبخ مشترك**: لو طباخ استنى (يعني الـ work item نام)، المدير فوراً يبعت طباخ تاني للأوردر اللي جاهزة، والـ CPU مشغلاش.

**الـ mapping الكامل:**
- **الطلب الأكله** = الـ function اللي محتاجة تتنفذ async
- **الأوردر بورد (workqueue)** = الـ domain للـ `flush`, `cancel`, و progress guarantee
- **الطباخين الاحتياط** = الـ rescue workers (مضمونين موجودين حتى في memory pressure)
- **أنواع الأوردر**: طلب عادي (normal pool) أو VIP (highpri pool)
- **وردية الليل والصبح** = bound wq (ملزوم بـ CPU معين) vs unbound wq

---

### الـ Big Picture Architecture

```
Drivers / Subsystems
     │
     │  queue_work(my_wq, &work_item)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    workqueue_struct (my_wq)                      │
│  flags: WQ_UNBOUND | WQ_MEM_RECLAIM   max_active: 0 (default)  │
│  → domain for flush/cancel/forward-progress guarantees          │
└────────────────────────────┬────────────────────────────────────┘
                             │  routes work to pool based on attrs
           ┌─────────────────┼─────────────────────┐
           ▼                 ▼                     ▼
    ┌─────────────┐  ┌─────────────┐       ┌──────────────┐
    │ worker_pool │  │ worker_pool │  ...  │ worker_pool  │
    │  CPU-0      │  │  CPU-1      │       │  UNBOUND     │
    │  nice=0     │  │  nice=0     │       │  (dynamic)   │
    └──────┬──────┘  └──────┬──────┘       └──────┬───────┘
           │                │                      │
     ┌─────▼──┐       ┌─────▼──┐            ┌─────▼──┐
     │kworker │       │kworker │            │kworker │
     │  0:1   │       │  1:3   │            │  u:5   │
     └────────┘       └────────┘            └────────┘

Per-CPU pools (2 per CPU):
  pool[2n]   → nice=0   (normal priority)
  pool[2n+1] → nice=-20 (highpri, WQ_HIGHPRI workqueues)

Unbound pools:
  Dynamic — created on demand based on affinity scope (LLC, NUMA, etc.)
  Shared among all unbound workqueues with matching attrs

BH workqueues:
  Each per-CPU has one pseudo-worker = softirq execution context
  No concurrency management needed (single context per CPU)
```

---

### الـ Core Abstraction — الـ work_struct

الـ abstraction الأساسية هي `struct work_struct` — وحدة شغل قابلة للـ queue:

```c
struct work_struct {
    atomic_long_t data;       /* multipurpose field — flags + pointer */
    struct list_head entry;   /* links work into pool's worklist */
    work_func_t func;         /* the function to execute */
};
```

الحيلة الذكية في `data`: الـ field ده مش بس pointer — هو **متعدد الأدوار** حسب حالة الـ work item:

```
When work is ON-QUEUE (WORK_STRUCT_PWQ set):
MSB [ pwq pointer          ] [ flush color 4b ] [ flags 4b ] LSB
      ^
      pointer to pool_workqueue — الرابط للـ pool

When work is OFF-QUEUE (not queued):
MSB [ pool ID 31b ] [ disable depth 16b ] [ OFFQ flags ] [ flags ] LSB
      ^
      ID of last pool that executed this work
```

يعني بـ word واحدة (64-bit) الـ kernel عارف: هل الشغل عليه queue؟ على أنهي pool؟ هل متوقف (disabled)؟ وأنهي flush color؟ — كل ده بـ zero overhead.

---

### الـ Struct Relationships

```
workqueue_struct (logical view - user facing)
    │
    │  has per-CPU or per-pool
    ▼
pool_workqueue (pwq) ──────────── worker_pool
    │   links wq ↔ pool              │
    │   tracks active/inactive        │   manages workers
    │   enforces max_active           ▼
    │                           ┌──────────────┐
    │                           │  worker       │
    │                           │  (kthread)    │
    │                           │  current_work │
    └──► work_struct ◄──────────│  ← executing  │
              │                 └──────────────┘
              │ entry (list_head)
              ▼
         worklist (in worker_pool)
         ← pending works queued here

struct delayed_work:
    ┌─────────────────┐
    │  work_struct    │ ← the actual work
    │  timer_list     │ ← fires after delay, then queues work
    │  *wq            │ ← target workqueue
    └─────────────────┘

struct rcu_work:
    ┌─────────────────┐
    │  work_struct    │ ← queued after RCU grace period
    │  rcu_head       │ ← RCU callback triggers the queue
    │  *wq            │
    └─────────────────┘
```

---

### الـ Concurrency Management — الفكرة الجوهرية

الـ cmwq بيتكلم مع الـ **scheduler** مباشرة. لما worker thread بينام (ينتظر lock أو I/O)، الـ scheduler بيبلغ الـ worker-pool. الـ pool بيشوف:

```
if (nr_running workers on this CPU == 0 && worklist not empty):
    wake up idle worker  OR  create new worker
```

لما الـ worker يصحى تاني:

```
if (nr_running workers > 0):
    don't start new worker — enough concurrency
```

النتيجة: دايماً **worker واحد على الأقل شغال** لو في pending work، ومفيش CPU بياخد break وفي شغل مستني. ده بيحقق **minimal but sufficient concurrency** من غير overhead.

**استثناء مهم — `WQ_CPU_INTENSIVE`**: الـ work items اللي محتاجة CPU كتير **مش بتحتسبش** في عدد الـ running workers. يعني لو عندي work item بتحرق الـ CPU — الـ pool مش هيمنعه من بدء workers تانية. التنظيم بيعمله الـ scheduler العادي.

---

### الـ Flags — التحكم في السلوك

| Flag | المعنى |
|---|---|
| `WQ_BH` | تنفذ في softirq context — مش بتنام، equivalent لـ softirq مباشرة |
| `WQ_UNBOUND` | workers مش مقيدين بـ CPU معين — للشغل اللي بياخد وقت طويل |
| `WQ_FREEZABLE` | الـ work items بتتوقف عند suspend (freeze phase) |
| `WQ_MEM_RECLAIM` | **مهم جداً** — يضمن rescue worker محجوز حتى في memory pressure |
| `WQ_HIGHPRI` | بتروح على الـ pool الـ nice=-20 |
| `WQ_CPU_INTENSIVE` | لا تعد كـ runnable في concurrency calculation |
| `WQ_SYSFS` | expose الـ wq في `/sys/devices/virtual/workqueue/` |

---

### الـ Rescue Worker — ضمان التقدم

**الـ forward progress guarantee** ده concept مهم:

تخيل إن الـ memory pressure شديد والـ kernel مش قادر يـ allocate thread جديد. لو كل الـ workers موجودين على sleep ومحتاجين memory reclaim يكمل — deadlock.

الـ wq اللي عليها `WQ_MEM_RECLAIM` بيكون معاها **rescue worker** محجوز مسبقاً — thread موجود من بدري، مش محتاج allocation جديدة. لو الـ worker-pool اتعطل بسبب memory pressure، الـ rescue worker بيكمل الشغل.

**القاعدة الذهبية**: أي كود في مسار الـ memory reclaim لازم يستخدم wq بـ `WQ_MEM_RECLAIM`.

---

### الـ Affinity Scopes — تحسين الـ Locality للـ Unbound WQs

الـ unbound workqueues بتجمع CPUs في **pods** حسب الـ scope:

```
System with 4 CPUs, 2 NUMA nodes, 2 L3 caches:

CPU0─CPU1  (NUMA 0, L3-A)    CPU2─CPU3  (NUMA 1, L3-B)

WQ_AFFN_CPU:    [0] [1] [2] [3]   → 4 pods, كل CPU منفرد
WQ_AFFN_CACHE:  [0,1] [2,3]       → 2 pods, كل L3 pod
WQ_AFFN_NUMA:   [0,1] [2,3]       → 2 pods (نفس النتيجة هنا)
WQ_AFFN_SYSTEM: [0,1,2,3]         → pod واحد
```

الـ default هو `WQ_AFFN_CACHE` — trade-off مناسب بين locality وutilization.

`affinity_strict=0` (الـ default): الـ worker بيبدأ داخل الـ pod لكن الـ scheduler حر يحركه. أحسن للـ utilization.
`affinity_strict=1`: الـ worker مقيد دايماً بالـ pod. أحسن للـ power أو workload isolation.

---

### الـ System Workqueues — الـ Built-in Pools

الـ kernel بيوفر workqueues جاهزة للاستخدام المباشر:

```
system_wq              → events_*        bound, normal
system_highpri_wq      → events_highpri  bound, highpri
system_long_wq         → events_long     bound, long-running
system_unbound_wq      → events_unbound  unbound
system_freezable_wq    → events_freezable bound, freezable
system_power_efficient_wq → events_power_efficient
```

لو شغلك بسيط ومش بيعمل flood — استخدم `system_wq`. لو محتاج فصل (flush group، أو ممكن تـ saturate الـ system wq) — اعمل dedicated wq.

---

### الـ cmwq بيملك إيه vs بيفوّض إيه للـ Drivers

| الـ cmwq بيملك | الـ Driver بيملك |
|---|---|
| worker threads (kworkers) | تعريف الـ `work_func_t` |
| worker-pools (shared, per-CPU + unbound) | قرار إمتى تعمل `queue_work()` |
| concurrency management (scheduler hooks) | اختيار الـ workqueue المناسبة والـ flags |
| rescuer threads | الـ `work_struct` نفسه (embedded في driver struct) |
| pool creation/destruction | `cancel_work_sync()` و `flush_work()` عند الحاجة |
| worker lifecycle (idle timeout, creation) | — |

الـ driver **مش** محتاج يتعامل مع threads خالص — بس يحدد الشغل (function) ومتى يتبعت (queue_work). كل حاجة تانية الـ cmwq بيديرها.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### `enum wq_flags` — فلاجز الـ Workqueue

| Flag | القيمة | المعنى |
|------|--------|--------|
| `WQ_BH` | `1 << 0` | تنفيذ في softirq context (BH) |
| `WQ_UNBOUND` | `1 << 1` | مش مربوط بـ CPU محدد |
| `WQ_FREEZABLE` | `1 << 2` | بيتجمد وقت الـ suspend |
| `WQ_MEM_RECLAIM` | `1 << 3` | ممكن يتاستخدم في memory reclaim — **لازم** يتعين |
| `WQ_HIGHPRI` | `1 << 4` | أولوية عالية (nice منخفض) |
| `WQ_CPU_INTENSIVE` | `1 << 5` | مش بيأثر على concurrency level |
| `WQ_SYSFS` | `1 << 6` | ظاهر في `/sys/devices/virtual/workqueue/` |
| `WQ_POWER_EFFICIENT` | `1 << 7` | بيتحول لـ unbound لو `power_efficient` param اتفعّل |
| `WQ_PERCPU` | `1 << 8` | مربوط بـ CPU معين |
| `__WQ_DESTROYING` | `1 << 15` | internal: الـ wq بيتحذف |
| `__WQ_DRAINING` | `1 << 16` | internal: الـ wq بيتصرف |
| `__WQ_ORDERED` | `1 << 17` | internal: ordered execution |
| `__WQ_LEGACY` | `1 << 18` | internal: من API القديم |

#### `enum work_bits` — بتات الـ `work_struct->data`

الـ `data` field في الـ `work_struct` بيحمل معلومات كتير في نفس الـ `unsigned long`:

| المجال | البتات | المعنى |
|--------|--------|--------|
| `WORK_STRUCT_PENDING_BIT` | 0 | الـ work item في الانتظار |
| `WORK_STRUCT_INACTIVE_BIT` | 1 | الـ work item غير نشط (تجاوز `max_active`) |
| `WORK_STRUCT_PWQ_BIT` | 2 | الـ data بيشاور على pwq |
| `WORK_STRUCT_LINKED_BIT` | 3 | الـ next work مربوط بيه |
| `WORK_STRUCT_COLOR_SHIFT` | bits 4–7 | لون الـ flush (4 bits) |
| `WORK_STRUCT_PWQ_SHIFT` | bits 8+ | مؤشر الـ pwq (لما PWQ_BIT=1) |
| `WORK_OFFQ_BH_BIT` | بعد الـ flags | BH work item |
| `WORK_OFFQ_DISABLE_SHIFT` | +1 | disable depth (16 bit) |
| `WORK_OFFQ_POOL_SHIFT` | +17 | ID الـ pool الأخير (31 bit) |

#### `enum wq_affn_scope` — نطاقات الـ Affinity

| Scope | المعنى |
|-------|--------|
| `WQ_AFFN_DFL` | استخدم الـ default (module param) |
| `WQ_AFFN_CPU` | pod واحد لكل CPU |
| `WQ_AFFN_SMT` | pod واحد لكل physical core |
| `WQ_AFFN_CACHE` | pod واحد لكل LLC — **الـ default** |
| `WQ_AFFN_NUMA` | pod واحد لكل NUMA node |
| `WQ_AFFN_SYSTEM` | كل الـ CPUs في pod واحد |

#### `enum pool_workqueue_stats` — إحصائيات الـ PWQ

| Stat | المعنى |
|------|--------|
| `PWQ_STAT_STARTED` | عدد الـ work items اللي اتبدأت |
| `PWQ_STAT_COMPLETED` | عدد الـ work items اللي اتكملت |
| `PWQ_STAT_CPU_TIME` | إجمالي وقت CPU المستهلك |
| `PWQ_STAT_CPU_INTENSIVE` | انتهاكات `wq_cpu_intensive_thresh_us` |
| `PWQ_STAT_CM_WAKEUP` | تنبيهات concurrency-management |
| `PWQ_STAT_REPATRIATED` | workers اتجابوا جوا scope |
| `PWQ_STAT_MAYDAY` | طلبات نجدة للـ rescuer |
| `PWQ_STAT_RESCUED` | work items اتنفذت بواسطة rescuer |

#### `enum wq_consts` — ثوابت مهمة

| Constant | القيمة | المعنى |
|----------|--------|--------|
| `WQ_MAX_ACTIVE` | 2048 | أقصى حد لـ `max_active` |
| `WQ_DFL_ACTIVE` | 1024 | الـ default لما `max_active = 0` |
| `WQ_DFL_MIN_ACTIVE` | 8 | الـ default لـ `min_active` لكل node |

---

### أهم الـ Structs

#### 1. `struct work_struct`
**التعريف:** `/include/linux/workqueue_types.h`

الوحدة الأساسية — بتمثل مهمة واحدة عايز تتنفذ بشكل async.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `data` | `atomic_long_t` | بيحمل الـ flags + مؤشر الـ pwq أو ID الـ pool (كل ده في word واحد) |
| `entry` | `struct list_head` | ربطه في قائمة الـ worker pool |
| `func` | `work_func_t` | الـ callback اللي بيتنفذ |
| `lockdep_map` | `struct lockdep_map` | للـ lockdep tracking (CONFIG_LOCKDEP فقط) |

**الـ `data` field:** مش مجرد pointer — بيحمل state machine صغيرة:
```
MSB                                                    LSB
[ pool ID (31b) ][ disable depth (16b) ][ OFFQ flags ][ STRUCT flags (4-5b) ]
                           OR (لما PWQ_BIT=1)
[ pwq pointer            ][ flush color (4b) ][ STRUCT flags (4-5b) ]
```

---

#### 2. `struct delayed_work`
**التعريف:** `/include/linux/workqueue.h`

امتداد للـ `work_struct` بيضيف timer لتأخير التنفيذ.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `work` | `struct work_struct` | الـ work الأساسي |
| `timer` | `struct timer_list` | الـ timer اللي بيطلق الـ work بعد الـ delay |
| `wq` | `struct workqueue_struct *` | الـ wq اللي هيتحط عليه الـ work لما الـ timer يطلق |
| `cpu` | `int` | الـ CPU المستهدف |

---

#### 3. `struct rcu_work`
**التعريف:** `/include/linux/workqueue.h`

بيأجل تنفيذ الـ work لحد ما تخلص كل الـ RCU grace periods الحالية.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `work` | `struct work_struct` | الـ work الأساسي |
| `rcu` | `struct rcu_head` | الـ RCU callback |
| `wq` | `struct workqueue_struct *` | الـ wq المستهدف |

---

#### 4. `struct workqueue_attrs`
**التعريف:** `/include/linux/workqueue.h`

بتحدد خصائص الـ unbound workqueue وبالتالي الـ worker pool المناسب.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `nice` | `int` | مستوى الأولوية للـ workers |
| `cpumask` | `cpumask_var_t` | الـ CPUs المسموح بيها |
| `__pod_cpumask` | `cpumask_var_t` | internal: الـ CPUs الخاصة بـ pod واحد |
| `affn_strict` | `bool` | لو true: workers ملزمون بالـ pod — لو false: best-effort |
| `affn_scope` | `enum wq_affn_scope` | نطاق الـ affinity |
| `ordered` | `bool` | تنفيذ ترتيبي (work item واحد في كل مرة) |

---

#### 5. `struct worker_pool`
**التعريف:** `/kernel/workqueue.c`

قلب النظام — بيدير مجموعة من الـ workers وقائمة الـ work items.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `lock` | `raw_spinlock_t` | يحمي معظم الـ fields |
| `cpu` | `int` | الـ CPU المرتبط به (أو -1 لو unbound) |
| `node` | `int` | الـ NUMA node |
| `id` | `int` | ID فريد في `worker_pool_idr` |
| `flags` | `unsigned int` | pool flags |
| `nr_running` | `int` | عدد الـ workers النشطة حاليًا (لـ concurrency management) |
| `worklist` | `struct list_head` | قائمة الـ work items المنتظرة |
| `nr_workers` | `int` | إجمالي عدد الـ workers |
| `nr_idle` | `int` | عدد الـ workers العاطلة |
| `idle_list` | `struct list_head` | قائمة الـ workers العاطلة |
| `idle_timer` | `struct timer_list` | بيطلق idle timeout لحذف workers زيادة |
| `mayday_timer` | `struct timer_list` | SOS timer للـ rescuer |
| `busy_hash` | `DECLARE_HASHTABLE` | hash table للـ workers المشغولة (key: work address) |
| `manager` | `struct worker *` | الـ worker اللي بيدير الـ pool حاليًا |
| `workers` | `struct list_head` | كل الـ workers المرتبطين بالـ pool |
| `attrs` | `struct workqueue_attrs *` | خصائص الـ pool |
| `refcnt` | `int` | عدد الـ pwqs اللي بيستخدموا الـ pool (للـ unbound) |

---

#### 6. `struct worker`
**التعريف:** `/kernel/workqueue_internal.h`

الـ kthread اللي بينفذ الـ work items فعليًا.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `entry` / `hentry` | union | في `idle_list` لو عاطل، في `busy_hash` لو مشغول |
| `current_work` | `struct work_struct *` | الـ work اللي بيشتغل عليه دلوقتي |
| `current_func` | `work_func_t` | الـ function الحالية |
| `current_pwq` | `struct pool_workqueue *` | الـ pwq المرتبط بالـ work الحالي |
| `current_at` | `u64` | وقت بدء التنفيذ (لـ CPU intensive detection) |
| `sleeping` | `int` | الـ worker نايم؟ (للـ concurrency management) |
| `last_func` | `work_func_t` | آخر function اتنفذت (لـ scheduler identity) |
| `scheduled` | `struct list_head` | قائمة الـ work items المجدولة لنفس الـ worker |
| `task` | `struct task_struct *` | الـ kthread نفسه |
| `pool` | `struct worker_pool *` | الـ pool المرتبط بيه |
| `last_active` | `unsigned long` | آخر timestamp نشاط |
| `flags` | `unsigned int` | worker flags |
| `desc` | `char[32]` | وصف نصي للـ debugging |
| `rescue_wq` | `struct workqueue_struct *` | الـ wq اللي بينقذه (للـ rescuers فقط) |

---

#### 7. `struct pool_workqueue` (pwq)
**التعريف:** `/kernel/workqueue.c`

الـ bridge بين الـ `workqueue_struct` والـ `worker_pool`. كل wq ليه pwq واحد لكل pool بيستخدمه.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `pool` | `struct worker_pool *` | الـ pool المرتبط |
| `wq` | `struct workqueue_struct *` | الـ wq المالك |
| `work_color` | `int` | اللون الحالي للـ work items (للـ flush) |
| `flush_color` | `int` | اللون اللي الـ flush بينتظره |
| `refcnt` | `int` | reference count |
| `nr_in_flight[]` | `int[WORK_NR_COLORS]` | عدد الـ work items النشطة لكل لون |
| `plugged` | `bool` | لو true: التنفيذ متوقف مؤقتًا |
| `nr_active` | `int` | عدد الـ work items النشطة حاليًا |
| `inactive_works` | `struct list_head` | work items في الانتظار لأن `nr_active >= max_active` |
| `mayday_node` | `struct list_head` | في قائمة `wq->maydays` لما بيطلب نجدة |
| `stats[]` | `u64[PWQ_NR_STATS]` | إحصائيات التنفيذ |

**ملاحظة مهمة:** الـ pwq لازم يكون aligned على `1 << WORK_STRUCT_PWQ_SHIFT` عشان البتات المنخفضة في `work->data` تُستخدم كـ flags وما تتعارضش مع الـ pointer.

---

#### 8. `struct workqueue_struct`
**التعريف:** `/kernel/workqueue.c`

الواجهة الخارجية اللي الـ drivers بيتعاملوا معاها.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `pwqs` | `struct list_head` | كل الـ pwqs المرتبطة |
| `list` | `struct list_head` | في القائمة العالمية لكل الـ workqueues |
| `mutex` | `struct mutex` | يحمي الـ wq نفسه |
| `work_color` | `int` | اللون الحالي للـ work items |
| `flush_color` | `int` | اللون المنتظر إنهاؤه للـ flush |
| `nr_pwqs_to_flush` | `atomic_t` | عدد الـ pwqs اللي لسه في الـ flush |
| `first_flusher` | `struct wq_flusher *` | أول waiter على الـ flush |
| `flusher_queue` | `struct list_head` | قائمة انتظار الـ flush |
| `maydays` | `struct list_head` | pwqs طالبة نجدة (تحت memory pressure) |
| `rescuer` | `struct worker *` | الـ rescue worker المخصص (لـ WQ_MEM_RECLAIM) |
| `max_active` | `int` | أقصى work items نشطة |
| `min_active` | `int` | أدنى work items نشطة (للـ unbound) |
| `unbound_attrs` | `struct workqueue_attrs *` | خصائص الـ unbound wq |
| `dfl_pwq` | `struct pool_workqueue *` | الـ default pwq للـ unbound |
| `name[]` | `char[WQ_NAME_LEN]` | اسم الـ wq |
| `node_nr_active[]` | `struct wq_node_nr_active **` | per-NUMA-node active counters |

---

#### 9. `struct wq_node_nr_active`
**التعريف:** `/kernel/workqueue.c`

للـ unbound workqueues — بيعدّل الـ `max_active` per NUMA node عشان يتجنب الـ shared counter overhead.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `max` | `int` | الـ max_active الخاص بالـ node |
| `nr` | `atomic_t` | العدد الحالي |
| `lock` | `raw_spinlock_t` | يحمي القائمة (nested داخل pool locks) |
| `pending_pwqs` | `struct list_head` | pwqs منتظرة slot فاضي |

---

#### 10. `struct wq_flusher`
**التعريف:** `/kernel/workqueue.c`

بيمثل طلب `flush_workqueue()` واحد.

| Field | النوع | الوظيفة |
|-------|-------|---------|
| `list` | `struct list_head` | في `wq->flusher_queue` |
| `flush_color` | `int` | اللون اللي بينتظر انتهاؤه |
| `done` | `struct completion` | بيتسيجنل لما الـ flush يخلص |

---

### رسم علاقات الـ Structs

```
                    ┌─────────────────────────────────────────────────────┐
                    │              workqueue_struct (wq)                  │
                    │  mutex, flags, max_active, name                     │
                    │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
                    │  │  pwq[0]  │  │  pwq[1]  │  │  pwq[N]  │ ◄──pwqs  │
                    │  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
                    │       │              │              │                │
                    │  rescuer ──────────────────────────────────────►    │
                    └───────┼──────────────┼──────────────┼───────────────┘
                            │              │              │
                    ┌───────▼──────────────▼──────────────▼───────────────┐
                    │           pool_workqueue (pwq)                      │
                    │  nr_active, inactive_works, stats[]                 │
                    │       pool ──────────────────────────────────────►  │
                    │       wq ◄──────────── back pointer                 │
                    └───────────────────────────────────────┬─────────────┘
                                                            │
                    ┌───────────────────────────────────────▼─────────────┐
                    │              worker_pool                             │
                    │  lock, cpu, node, nr_running                        │
                    │  worklist ──────► [work_struct] → [work_struct] →   │
                    │  idle_list ─────► [worker] → [worker]               │
                    │  busy_hash ─────► {work_addr: worker}               │
                    │  attrs ─────────► workqueue_attrs                   │
                    └──────────────────────┬──────────────────────────────┘
                                           │ workers list
                    ┌──────────────────────▼──────────────────────────────┐
                    │                  worker                             │
                    │  task (kthread), pool, current_work                 │
                    │  sleeping, flags, desc                              │
                    └──────────────────────┬──────────────────────────────┘
                                           │ current_work
                    ┌──────────────────────▼──────────────────────────────┐
                    │               work_struct                           │
                    │  data (flags + pwq/pool ID), func, entry            │
                    └─────────────────────────────────────────────────────┘
```

**علاقات الـ Unbound WQ مع NUMA:**
```
workqueue_struct
  └── node_nr_active[0] ──► wq_node_nr_active (node 0)
  └── node_nr_active[1] ──► wq_node_nr_active (node 1)
                                └── pending_pwqs ──► [pwq] → [pwq]
```

---

### دورة حياة الـ Work Item

```
INIT_WORK(&work, my_func)
        │
        ▼
  work_struct::data = WORK_DATA_INIT()
  work_struct::func = my_func
  work_struct::entry = empty list
        │
        ▼
queue_work(wq, &work)
        │
        ├─ set WORK_STRUCT_PENDING_BIT
        │
        ├─ اختار الـ pwq المناسب (per-cpu أو unbound)
        │
        ├─ لو nr_active < max_active:
        │     أضف لـ pool->worklist
        │
        └─ لو nr_active >= max_active:
              set WORK_STRUCT_INACTIVE_BIT
              أضف لـ pwq->inactive_works
                    │
                    ▼
           worker thread (worker_thread)
                    │
                    ├─ clear PENDING_BIT
                    ├─ set worker في busy_hash
                    ├─ worker->current_work = &work
                    ├─ work->func(&work)  ◄── التنفيذ الفعلي
                    │
                    └─ بعد الانتهاء:
                          clear current_work
                          move worker لـ idle_list
                          check inactive_works → activate next
                                │
                                ▼
                         work_struct حر من أي pool
                         data يرجع لـ WORK_STRUCT_NO_POOL
```

---

### دورة حياة الـ Worker Pool وإدارة الـ Concurrency

```
CPU يبدأ (CPU hotplug online)
        │
        ▼
init_worker_pool(pool)
        │
        ├─ pool->cpu = CPU
        ├─ pool->attrs = normal أو highpri
        └─ create_worker(pool) ──► kthread_create(worker_thread)
                │
                ▼
        worker_thread() [loop]
                │
                ├─ لو في work في worklist:
                │     process_one_work()
                │
                ├─ لو nr_running > 0 وفي work:
                │     sleep (concurrency management — مش هنبدأ worker جديد)
                │
                └─ لو nr_running == 0 وفي work:
                      wakeup أو create worker جديد
                │
                ▼
        manage_workers()
                │
                ├─ لو pool بيحتاج workers أكتر: create_worker()
                └─ لو idle workers كتير: idle_timer → destroy_worker()

CPU يوقف (CPU hotplug offline)
        │
        ├─ unbound كل الـ workers
        └─ workers يكملوا شغل على أي CPU
```

---

### دورة حياة الـ Workqueue (من الإنشاء للحذف)

```
alloc_workqueue("name", flags, max_active)
        │
        ├─ kzalloc(workqueue_struct)
        ├─ alloc_workqueue_attrs()
        ├─ alloc_and_link_pwqs()  ──► create/find worker_pool لكل CPU/node
        ├─ init_rescuer()  ──► (لو WQ_MEM_RECLAIM) create rescue kthread
        ├─ list_add(&wq->list, &workqueues)
        └─ workqueue_sysfs_register()  ──► (لو WQ_SYSFS)
                │
                ▼
         [wq جاهز للاستخدام]
                │
                ▼
destroy_workqueue(wq)
        │
        ├─ set __WQ_DESTROYING
        ├─ drain_workqueue()  ──► ينتظر كل الـ work items تخلص
        ├─ set __WQ_DRAINING
        ├─ destroy_rescuer()
        ├─ unlink_pwqs()
        └─ kfree(wq)
```

---

### Call Flow Diagrams

#### تدفق الـ `queue_work()`

```
driver calls queue_work(wq, work)
  │
  └─► __queue_work(cpu, wq, work)
        │
        ├─ test_and_set_bit(PENDING) ──► return إذا كان pending
        │
        ├─ get_work_pool_id(work)  ──► آخر pool استخدمه الـ work
        │
        ├─ select_pwq:
        │     per-cpu wq ──► pwq الخاص بالـ CPU الحالي
        │     unbound wq ──► pwq الخاص بالـ pod اللي CPU ينتمي ليه
        │
        ├─ pwq_tryinc_nr_active(pwq)
        │     success ──► pool->worklist  ◄── work اتضاف
        │     fail    ──► pwq->inactive_works (INACTIVE flag set)
        │
        └─ wake_up_worker(pool)
              │
              └─► إيقاظ idle worker أو create_worker() لو ضروري
```

#### تدفق الـ `flush_workqueue()`

```
flush_workqueue(wq)
  │
  └─► __flush_workqueue(wq)
        │
        ├─ حساب الـ flush color الحالي
        │
        ├─ تحديث wq->work_color لكل الـ pwqs  (color++)
        │     كل work item جديد هياخد اللون الجديد
        │
        ├─ إنشاء wq_flusher{flush_color = old_color}
        │
        ├─ انتظار completion  [فيه block هنا]
        │
        └─ لما آخر work item باللون القديم يخلص:
              pwq_dec_nr_in_flight()
                └─► wq_flusher_wake_up_all()
                      └─► complete(&flusher->done)
```

#### تدفق الـ Rescuer (memory pressure)

```
worker_pool::mayday_timer fires
  │
  └─► send_mayday(work)
        │
        ├─ أضف pwq لـ wq->maydays
        └─ wake_up(wq->rescuer->task)
              │
              ▼
        rescuer_thread()
              │
              ├─ lock pool
              ├─ خذ work من pool->worklist
              ├─ نفذه بدون أي قيود memory allocation
              └─ كرر لحد ما wq->maydays تفضى
```

#### تدفق الـ Delayed Work

```
queue_delayed_work(wq, dwork, delay)
  │
  └─► mod_timer(&dwork->timer, jiffies + delay)
        │
        [بعد الـ delay]
        │
        ▼
  delayed_work_timer_fn(timer)
        │
        └─► __queue_work(dwork->cpu, dwork->wq, &dwork->work)
                │
                ▼
          نفس تدفق queue_work العادي
```

---

### استراتيجية الـ Locking

الـ workqueue subsystem بيستخدم طبقات متداخلة من الـ locks بترتيب صارم لتجنب الـ deadlocks.

#### الـ Locks وما يحمونه

| Lock | النوع | يحمي |
|------|-------|------|
| `pool->lock` | `raw_spinlock_t` | كل الـ pool state: worklist, workers, nr_running |
| `wq->mutex` | `mutex` | كل الـ wq state: pwqs, flush state, attrs |
| `wq_pool_mutex` | `mutex` | القوائم العالمية: worker_pool_idr, unbound_pool_hash |
| `wq_pool_attach_mutex` | `mutex` | إرفاق/فصل workers من pools |
| `wq_mayday_lock` | `raw_spinlock_t` | `wq->maydays` |
| `wq_node_nr_active->lock` | `raw_spinlock_t` | per-node active counts + pending_pwqs |

#### ترتيب الـ Locks (من الأعلى للأدنى)

```
wq_pool_mutex
    └─► wq->mutex
            └─► pool->lock
                    └─► wq_node_nr_active->lock
                                └─► wq_mayday_lock
```

#### تقنيات الـ Locking المستخدمة

**الـ RCU:**
- قراءة `workqueues` list (القائمة العالمية) بدون `wq_pool_mutex`
- قراءة `wq->dfl_pwq` بدون `wq->mutex`
- الوصول لـ `worker_pool` من `get_work_pool()` بدون lock

**الـ Annotations في الكود:**

| Annotation | المعنى |
|------------|--------|
| `L:` | محمي بـ `pool->lock` |
| `WQ:` | محمي بـ `wq->mutex` |
| `WR:` | writes بـ `wq->mutex`، reads بـ RCU |
| `PL:` | محمي بـ `wq_pool_mutex` |
| `PR:` | writes بـ `wq_pool_mutex`، reads بـ RCU |
| `A:` | محمي بـ `wq_pool_attach_mutex` |
| `MD:` | محمي بـ `wq_mayday_lock` |
| `K:` | الـ worker نفسه بس (أو بـ pool->lock) |
| `S:` | الـ worker self فقط |
| `I:` | immutable بعد الإنشاء |

**مثال عملي — إضافة work item:**

```c
// caller context (أي context)
spin_lock(&pool->lock);              // L: يحمي worklist
list_add_tail(&work->entry, &pool->worklist);
spin_unlock(&pool->lock);

// concurrency management — بدون lock (atomic ops)
atomic_inc(&pool->nr_running);       // P: preemption disabled فقط
```

**الـ `nr_running` counter:**
خاص بيه حماية بسيطة — بيتزاد في process context مع preemption disabled، وبيتقلل مع `pool->lock` held. ده بيخلي الـ concurrency management يشتغل بأقل overhead ممكن.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Work Item Initialization

| الـ Macro / Function | الغرض |
|---|---|
| `INIT_WORK(work, func)` | تهيئة `work_struct` على الـ heap |
| `INIT_WORK_ONSTACK(work, func)` | تهيئة `work_struct` على الـ stack مع lockdep tracking |
| `INIT_DELAYED_WORK(work, func)` | تهيئة `delayed_work` |
| `INIT_DEFERRABLE_WORK(work, func)` | delayed work مع `TIMER_DEFERRABLE` |
| `INIT_RCU_WORK(work, func)` | تهيئة `rcu_work` |
| `DECLARE_WORK(n, f)` | إعلان وتهيئة static `work_struct` |
| `DECLARE_DELAYED_WORK(n, f)` | إعلان وتهيئة static `delayed_work` |
| `destroy_work_on_stack(work)` | cleanup الـ lockdep map لـ on-stack work |

#### Workqueue Allocation & Configuration

| الـ Function | الغرض |
|---|---|
| `alloc_workqueue(fmt, flags, max_active, ...)` | إنشاء workqueue جديد |
| `alloc_ordered_workqueue(fmt, flags, ...)` | ordered workqueue — work item واحد في كل وقت |
| `destroy_workqueue(wq)` | تدمير الـ workqueue وانتظار كل الـ work items |
| `alloc_workqueue_attrs()` | allocate `workqueue_attrs` struct |
| `free_workqueue_attrs(attrs)` | تحرير `workqueue_attrs` |
| `apply_workqueue_attrs(wq, attrs)` | تطبيق attributes على unbound workqueue |
| `workqueue_set_max_active(wq, max)` | تعديل `max_active` runtime |
| `workqueue_set_min_active(wq, min)` | تعديل `min_active` runtime |
| `workqueue_sysfs_register(wq)` | expose الـ wq في `/sys` |

#### Queueing

| الـ Function | الغرض |
|---|---|
| `queue_work(wq, work)` | queue على الـ local CPU |
| `queue_work_on(cpu, wq, work)` | queue على CPU محدد |
| `queue_work_node(node, wq, work)` | queue على NUMA node محدد |
| `queue_delayed_work(wq, dwork, delay)` | queue بعد delay بـ jiffies |
| `queue_delayed_work_on(cpu, wq, dwork, delay)` | delayed queue على CPU محدد |
| `mod_delayed_work(wq, dwork, delay)` | تعديل أو إضافة delayed work |
| `mod_delayed_work_on(cpu, wq, dwork, delay)` | mod على CPU محدد |
| `queue_rcu_work(wq, rwork)` | queue بعد RCU grace period |
| `schedule_work(work)` | queue على `system_percpu_wq` |
| `schedule_work_on(cpu, work)` | queue على CPU محدد في `system_percpu_wq` |
| `schedule_delayed_work(dwork, delay)` | delayed على `system_percpu_wq` |
| `schedule_delayed_work_on(cpu, dwork, delay)` | delayed على CPU محدد system wq |

#### Flush & Cancel

| الـ Function | الغرض |
|---|---|
| `flush_workqueue(wq)` | انتظر اكتمال كل الـ work items الحالية |
| `drain_workqueue(wq)` | drain حتى يكون الـ wq فاضي تماماً |
| `flush_work(work)` | انتظر اكتمال work item بعينه |
| `flush_delayed_work(dwork)` | flush delayed work (يبدأ التنفيذ فوراً لو pending) |
| `flush_rcu_work(rwork)` | flush rcu_work |
| `cancel_work(work)` | إلغاء بدون انتظار |
| `cancel_work_sync(work)` | إلغاء وانتظار اكتمال التنفيذ لو بدأ |
| `cancel_delayed_work(dwork)` | إلغاء delayed work بدون انتظار |
| `cancel_delayed_work_sync(dwork)` | إلغاء وانتظار |

#### Disable / Enable

| الـ Function | الغرض |
|---|---|
| `disable_work(work)` | يرفع disable depth ويلغي pending قيد التنفيذ |
| `disable_work_sync(work)` | نفس السابق لكن ينتظر لو الـ work شغال |
| `enable_work(work)` | يخفض disable depth، يرجع true لو وصل صفر |
| `enable_and_queue_work(wq, work)` | enable ثم queue دفعة واحدة |
| `disable_delayed_work(dwork)` | نفس disable_work للـ delayed variant |
| `disable_delayed_work_sync(dwork)` | sync variant للـ delayed |
| `enable_delayed_work(dwork)` | enable للـ delayed variant |

#### Runtime Helpers & Introspection

| الـ Function | الغرض |
|---|---|
| `work_pending(work)` | هل الـ work pending? |
| `delayed_work_pending(w)` | هل الـ delayed work pending? |
| `work_busy(work)` | bitmask — هل pending أو running؟ |
| `current_work()` | يرجع الـ `work_struct` اللي بيشغله الـ worker الحالي |
| `current_is_workqueue_rescuer()` | هل الـ thread الحالي هو rescuer worker؟ |
| `workqueue_congested(cpu, wq)` | هل الـ wq مازحوم على CPU معين؟ |
| `set_worker_desc(fmt, ...)` | يضبط description لـ kworker يظهر في `/proc` |
| `print_worker_info(log_lvl, task)` | يطبع معلومات الـ worker thread |
| `from_work(var, callback_work, field)` | `container_of` مختصر للـ work callback |
| `work_on_cpu(cpu, fn, arg)` | تشغيل function على CPU محدد بشكل synchronous |
| `schedule_on_each_cpu(func)` | تشغيل work على كل CPU |
| `execute_in_process_context(fn, ew)` | تشغيل في process context أو queue لو في interrupt |

#### Suspend / Freeze

| الـ Function | الغرض |
|---|---|
| `freeze_workqueues_begin()` | يبدأ freeze لكل `WQ_FREEZABLE` workqueues |
| `freeze_workqueues_busy()` | يتحقق إن كل الـ freezable wqs اتفرزت |
| `thaw_workqueues()` | يرفع الـ freeze ويستأنف التنفيذ |

---

### التصنيف حسب المجموعات الوظيفية

---

### 1. تهيئة الـ Work Items (Initialization)

هذه المجموعة مسؤولة عن تحضير الـ `work_struct` أو `delayed_work` أو `rcu_work` قبل أي عملية قueueing. التهيئة الصحيحة ضرورية لأن الـ kernel يستخدم الـ `data` field لتمرير flags ومعرف الـ worker pool.

---

#### `INIT_WORK`

```c
#define INIT_WORK(_work, _func)
```

يهيئ `work_struct` على الـ heap — يضبط `data` بـ `WORK_DATA_INIT()` وهو `WORK_STRUCT_NO_POOL` (يعني مش مرتبط بأي pool)، ويضبط `func` وينشئ `INIT_LIST_HEAD` للـ entry. داخلياً يمر على `__init_work` للـ debugobjects وعلى `lockdep_init_map` لو `CONFIG_LOCKDEP`.

- **`_work`**: pointer لـ `work_struct` يتم تهيئته
- **`_func`**: دالة من النوع `void (*)(struct work_struct *)`
- **Return**: لا يرجع قيمة
- **تفاصيل**: لا يجوز الاستخدام على on-stack work إلا مع `INIT_WORK_ONSTACK` — الـ lockdep map بيتعمل static key وتتسرب لو اتعمل destroy بدون `destroy_work_on_stack`.

---

#### `INIT_WORK_ONSTACK`

```c
#define INIT_WORK_ONSTACK(_work, _func)
```

نفس `INIT_WORK` لكن يمرر `_onstack=1` لـ `__init_work` حتى يسجل الـ debugobjects إنه on-stack. **لازم يتبعه** `destroy_work_on_stack()` قبل ما الـ stack frame يتحرر.

---

#### `INIT_DELAYED_WORK`

```c
#define INIT_DELAYED_WORK(_work, _func)
```

يهيئ `delayed_work` — يعمل `INIT_WORK` على الـ embedded `work` وينشئ `timer_list` بـ `delayed_work_timer_fn` كـ callback مع `TIMER_IRQSAFE`. لما الـ timer يشتغل، بيضيف الـ `work` للـ target workqueue.

---

#### `INIT_DEFERRABLE_WORK`

```c
#define INIT_DEFERRABLE_WORK(_work, _func)
```

نفس `INIT_DELAYED_WORK` لكن يضيف `TIMER_DEFERRABLE` للـ timer — يعني الـ timer ممكن يتأجل لو الـ CPU في low-power state. مفيد جداً للـ background tasks اللي مش time-critical.

---

#### `INIT_RCU_WORK`

```c
#define INIT_RCU_WORK(_work, _func)
```

يهيئ `rcu_work` — الـ implementation هي مجرد `INIT_WORK` على الـ embedded `work`. الـ RCU grace period بتتعامل معاها `queue_rcu_work` وليس التهيئة نفسها.

---

#### `destroy_work_on_stack` / `destroy_delayed_work_on_stack`

```c
void destroy_work_on_stack(struct work_struct *work);
void destroy_delayed_work_on_stack(struct delayed_work *work);
```

يحرر الـ lockdep map الخاص بـ on-stack work item. في الـ non-debug builds هي no-ops. **لازم تتستدعى** قبل ما الـ stack يتحرر لتجنب lockdep memory leak.

---

### 2. إنشاء وتهيئة الـ Workqueue (Allocation & Configuration)

الـ workqueue هو الـ domain اللي بتتضبط فيه الـ forward progress guarantee والـ flushing behavior والـ work item attributes. الـ allocation هنا بتحدد سلوك كل الـ work items اللي هتتضاف للـ wq ده.

---

#### `alloc_workqueue`

```c
struct workqueue_struct *alloc_workqueue(const char *fmt,
                                          unsigned int flags,
                                          int max_active, ...);
```

ينشئ workqueue جديد. الـ macro بيوجه الاستدعاء لـ `alloc_workqueue_noprof` عبر `alloc_hooks` لدعم memory allocation profiling.

- **`fmt`**: printf-style format string لاسم الـ wq — بيُستخدم أيضاً لتسمية الـ rescuer thread
- **`flags`**: بيتكون من `OR` لـ `enum wq_flags`:
  - `WQ_BH` — تنفيذ في BH/softirq context
  - `WQ_UNBOUND` — workers مش مربوطين بـ CPU معين
  - `WQ_PERCPU` — كل CPU ليه worker pool مستقل (الـ default)
  - `WQ_FREEZABLE` — بيتجمد أثناء system suspend
  - `WQ_MEM_RECLAIM` — **إلزامي** لو ممكن يُستخدم في memory reclaim path — بيضمن rescuer worker
  - `WQ_HIGHPRI` — workers بـ elevated priority (nice level منخفض)
  - `WQ_CPU_INTENSIVE` — الـ work items مش بتأثر على concurrency management
  - `WQ_SYSFS` — بيظهر في `/sys/devices/virtual/workqueue/`
  - `WQ_POWER_EFFICIENT` — per-cpu افتراضياً، unbound لو `wq_power_efficient` parameter مضبوط
- **`max_active`**: أقصى عدد work items شغالين في نفس الوقت per-CPU. القيمة 0 تستخدم الـ default (1024). الحد الأقصى 2048
- **Return**: pointer لـ `workqueue_struct` أو `NULL` عند الفشل
- **تفاصيل**: يأخذ mutex داخلياً (`wq_pool_mutex`). لو `WQ_BH` يُضبط مع `max_active != 0` — هيفشل.

**Pseudocode:**
```
alloc_workqueue(fmt, flags, max_active):
    allocate workqueue_struct
    if WQ_BH:
        assign to BH worker pool
    else:
        if WQ_UNBOUND: create/find unbound worker pools per affinity scope
        else: bind to per-CPU normal/highpri pools
    if WQ_MEM_RECLAIM: allocate rescuer kthread
    if WQ_SYSFS: workqueue_sysfs_register()
    return wq
```

---

#### `alloc_ordered_workqueue`

```c
#define alloc_ordered_workqueue(fmt, flags, args...)
    // expands to:
    alloc_workqueue(fmt, WQ_UNBOUND | __WQ_ORDERED | (flags), 1, ##args)
```

ينشئ workqueue بيشغل work item واحد في كل وقت بترتيب الـ queueing. هو في الأساس `WQ_UNBOUND` مع `max_active=1` و`__WQ_ORDERED` flag. استُبدل به النمط القديم `max_active=1 + WQ_UNBOUND`.

---

#### `destroy_workqueue`

```c
void destroy_workqueue(struct workqueue_struct *wq);
```

يدمر الـ workqueue بشكل آمن. ينتظر اكتمال كل الـ work items الشغالة وكل اللي في الـ queue، يوقف الـ rescuer thread لو موجود، ويحرر كل الـ resources.

- **`wq`**: الـ workqueue المراد تدميره
- **Return**: لا يرجع قيمة
- **تفاصيل مهمة**: **blocking call** — ممكن تستغرق وقت طويل. لا يجوز استدعاؤها في interrupt context. بتضبط `__WQ_DESTROYING` flag أولاً لمنع queueing جديد.

---

#### `apply_workqueue_attrs`

```c
int apply_workqueue_attrs(struct workqueue_struct *wq,
                           const struct workqueue_attrs *attrs);
```

يطبق attributes جديدة على **unbound workqueue** فقط. بيتحكم في `nice` level، `cpumask`، `affn_scope`، و`affn_strict`. بيعمل drain للـ workers الحاليين وينشئ worker pools جديدة مطابقة للـ attrs الجديدة.

- **`wq`**: unbound workqueue (بيفشل مع per-cpu wq)
- **`attrs`**: struct بيحتوي على الـ attributes الجديدة
- **Return**: `0` عند النجاح، negative error code عند الفشل
- **Locking**: يأخذ `wq_pool_mutex`
- **تفاصيل**: بيستدعيها أيضاً `wq_update_unbound_pod_attrs` لما تتغير CPU topology.

---

#### `workqueue_set_max_active` / `workqueue_set_min_active`

```c
void workqueue_set_max_active(struct workqueue_struct *wq, int max_active);
void workqueue_set_min_active(struct workqueue_struct *wq, int min_active);
```

يعدّل concurrency limits أثناء الـ runtime. `min_active` بيضمن minimum level من الـ concurrency حتى على NUMA nodes قليلة الـ CPUs — مهم لتجنب deadlocks في scenarios الـ interdependent work items.

- **تفاصيل**: يأخذ `wq->mutex` داخلياً. لو الـ `max_active` الجديد أكبر، بيحاول تشغيل الـ inactive items.

---

### 3. إضافة الـ Work Items للـ Queue (Queueing)

---

#### `queue_work_on`

```c
bool queue_work_on(int cpu, struct workqueue_struct *wq,
                   struct work_struct *work);
```

الـ core function للـ queueing. تضيف الـ work item لـ worker pool المرتبط بالـ CPU المحدد على الـ wq المعطى.

- **`cpu`**: رقم الـ CPU أو `WORK_CPU_UNBOUND` لاستخدام الـ local CPU
- **`wq`**: الـ workqueue المستهدف
- **`work`**: الـ work item
- **Return**: `true` لو تم الـ queue (كان off-queue)، `false` لو كان already pending
- **Memory ordering**: لو رجعت `true`، كل الـ stores السابقة للـ `queue_work_on` بتكون visible للـ worker thread قبل ما يشغل الـ work. ده guarantee مهم جداً لتمرير data للـ worker.
- **Locking**: يستخدم atomic operations على `work->data` — لا يحتاج lock خارجي
- **تفاصيل**: لو `WORK_STRUCT_PENDING` set → ترجع `false` بدون أي تغيير. لو clear → تضبط الـ bit بشكل atomic وتضيف للـ pool worklist.

---

#### `queue_work`

```c
static inline bool queue_work(struct workqueue_struct *wq,
                               struct work_struct *work);
```

wrapper على `queue_work_on(WORK_CPU_UNBOUND, wq, work)`. الـ `WORK_CPU_UNBOUND` هنا يعني "prefer local CPU" وليس unbound pool — يعني الـ work بيتضاف لـ pool الـ CPU الحالي.

---

#### `queue_work_node`

```c
bool queue_work_node(int node, struct workqueue_struct *wq,
                     struct work_struct *work);
```

يضيف الـ work item على أي CPU داخل NUMA node معين. مفيد لـ NUMA-aware code اللي محتاج locality بدون pin على CPU محدد.

---

#### `queue_delayed_work_on`

```c
bool queue_delayed_work_on(int cpu, struct workqueue_struct *wq,
                            struct delayed_work *work,
                            unsigned long delay);
```

يضيف `delayed_work` بعد `delay` jiffies. يبدأ الـ `timer_list` المدمج في الـ `delayed_work`، ولما يـ expire بيستدعي `delayed_work_timer_fn` اللي بتضيف الـ embedded `work_struct` للـ wq.

- **`delay`**: بالـ jiffies. القيمة 0 تعمل queue فوري
- **Return**: `true` لو تم تشغيل الـ timer، `false` لو كان الـ work already pending
- **تفاصيل**: بيحفظ الـ `wq` والـ `cpu` في الـ `delayed_work` struct نفسه لاستخدامهم لاحقاً لما الـ timer يـ expire.

---

#### `mod_delayed_work_on`

```c
bool mod_delayed_work_on(int cpu, struct workqueue_struct *wq,
                          struct delayed_work *dwork,
                          unsigned long delay);
```

يعدّل الـ delay لـ delayed work موجودة أو يضيف جديد. لو كانت pending → يعمل cancel للـ timer القديم ويبدأ جديد بالـ delay الجديد.

- **Return**: `true` لو كانت فعلاً pending وتم تعديلها، `false` لو كانت off-queue وتم إضافتها كجديد
- **تفاصيل**: `mod_delayed_work_on` مع `delay=0` + `cpu` معين = way لإعادة-queue الـ work على CPU محدد فوراً.

---

#### `queue_rcu_work`

```c
bool queue_rcu_work(struct workqueue_struct *wq, struct rcu_work *rwork);
```

يضيف `rcu_work` للـ queue بعد انتهاء الـ RCU grace period الحالية. يستخدم `call_rcu()` داخلياً — الـ `rcu_head` callback بيضيف الـ embedded `work_struct` للـ wq.

- **Return**: `true` لو تم الـ register مع RCU، `false` لو كان already pending
- **Caller context**: يمكن استدعاؤها من أي context بما فيه atomic

---

#### `schedule_work` / `schedule_work_on`

```c
static inline bool schedule_work(struct work_struct *work);
static inline bool schedule_work_on(int cpu, struct work_struct *work);
```

**الـ shortcut الأكثر استخداماً** — بيضيف على `system_percpu_wq` بدل ما تمرر wq صريح. مناسب للـ work items اللي مش محتاجة isolation أو فلوش مجمع.

---

### 4. الـ Flush والـ Cancel

هذه المجموعة من أهم المجموعات لأن الاستخدام الخاطئ يؤدي لـ deadlocks أو races.

---

#### `flush_workqueue` / `__flush_workqueue`

```c
#define flush_workqueue(wq)
void __flush_workqueue(struct workqueue_struct *wq);
```

ينتظر اكتمال كل الـ work items اللي كانت queued **وقت الاستدعاء**. يستخدم الـ "flush colors" mechanism — بيلون الـ work items الحالية بلون معين، وينتظر كل الألوان تنتهي.

- **تحذير**: `flush_workqueue` مع system-wide workqueues (`system_percpu_wq`, etc.) بيطبع compiler warning — استخدم `flush_work` بدلها على item محدد.
- **Caller**: يجب استدعاؤها من process context فقط (blocking)
- **Deadlock risk**: لو work item حاول يعمل `flush_workqueue` على نفس الـ wq من داخل worker → deadlock مضمون

---

#### `drain_workqueue`

```c
void drain_workqueue(struct workqueue_struct *wq);
```

يعمل drain متكرر حتى يصبح الـ wq فارغاً تماماً — بيكرر الـ flush لأن بعض work items ممكن تضيف work items جديدة. أبطأ من `flush_workqueue` لكن بيضمن القفو الكامل.

- **تفاصيل**: بيضبط `__WQ_DRAINING` flag لمنع إضافة work items جديدة من خارج الـ workers أنفسهم

---

#### `flush_work`

```c
bool flush_work(struct work_struct *work);
```

ينتظر اكتمال work item **بعينه**. لو الـ work شغال → ينتظر. لو pending → ينتظر ريح يشتغل وينتهي.

- **Return**: `true` لو كان فعلاً pending أو running وانتظر حتى انتهى، `false` لو كان already idle
- **Mechanism**: يستخدم `completion` داخلية — بيضيف "barrier work item" بعد الـ target ولما ينتهي الـ barrier يتحرر الـ flush
- **Caller**: process context فقط

---

#### `cancel_work` / `cancel_work_sync`

```c
bool cancel_work(struct work_struct *work);
bool cancel_work_sync(struct work_struct *work);
```

- **`cancel_work`**: يعمل cancel لو كان pending بس (لو شغال ما يوقفوش). يفضل في الـ list لو كان queued.
- **`cancel_work_sync`**: أقوى — لو الـ work شغال بيستنى ينتهي قبل ما يرجع. **بيضمن** إن الـ work مش شغال ومش pending بعد ما يرجع.

- **Return**: `true` لو كانت pending وتم إلغاؤها
- **Locking**: يستخدم `WORK_STRUCT_PENDING` bit بشكل atomic
- **تحذير**: `cancel_work_sync` ما تتستدعاش من داخل الـ work function نفسها → deadlock

---

#### `cancel_delayed_work` / `cancel_delayed_work_sync`

```c
bool cancel_delayed_work(struct delayed_work *dwork);
bool cancel_delayed_work_sync(struct delayed_work *dwork);
```

نفس سلوك `cancel_work` و`cancel_work_sync` لكن للـ delayed work. `cancel_delayed_work` بيعمل `del_timer_sync` داخلياً لوقف الـ timer لو لسه ما انطلقش.

---

### 5. الـ Disable / Enable

آلية جديدة نسبياً تسمح بـ "تجميد" work item مؤقتاً دون حذفه. استُخدمت لحل مشاكل الـ races في الـ drivers اللي محتاجين إيقاف الـ work مؤقتاً أثناء الـ suspend أو إعادة الـ configuration.

---

#### `disable_work` / `disable_work_sync`

```c
bool disable_work(struct work_struct *work);
bool disable_work_sync(struct work_struct *work);
```

يرفع الـ **disable depth** بمقدار 1. لو الـ work كان pending → يعمل cancel له. أي محاولة لـ `queue_work` بعدها بتفشل صامتة ما يُقيد الـ depth.

- **`disable_work_sync`**: زيادة عن ده، لو الـ work كان شغال وقت الاستدعاء → بيستنى ينتهي
- **Return**: `true` لو كان pending وتم إلغاؤه
- **الـ depth**: مخزون في الـ bits `WORK_OFFQ_DISABLE_BITS` داخل `work->data` (16-bit counter)

---

#### `enable_work`

```c
bool enable_work(struct work_struct *work);
```

يخفض الـ disable depth بمقدار 1. لو وصل صفر → الـ work بيصبح قابلاً للـ queue مرة تانية.

- **Return**: `true` لو الـ depth وصل 0 (تم الـ enable)
- **تحذير**: لا يعيد queue الـ work تلقائياً — استخدم `enable_and_queue_work` لو محتاج ده

---

#### `enable_and_queue_work`

```c
static inline bool enable_and_queue_work(struct workqueue_struct *wq,
                                          struct work_struct *work);
```

يجمع `enable_work` + `queue_work` في call واحد. لو `enable_work` رجع `true` (يعني الـ depth وصل 0) → يعمل `queue_work` فوراً.

- **Return**: `true` لو تم الـ enable والـ queue، `false` لو الـ depth لسه > 0
- **تنبيه**: الـ work بيتضاف دايماً لما الـ depth يوصل صفر. لو محتاج conditional queueing حسب events حصلت وهو disabled → اعمل explicit check بعد `enable_work`.

---

### 6. الـ Helper Functions والـ Introspection

---

#### `work_busy`

```c
unsigned int work_busy(struct work_struct *work);
```

يرجع bitmask:
- `WORK_BUSY_PENDING (1)` — الـ work في queue
- `WORK_BUSY_RUNNING (2)` — الـ work شغال على worker thread
- 0 — idle

- **تحذير**: النتيجة **snapshot** — ممكن تتغير فوراً بعد الاستدعاء. ما يتستخدمش لـ synchronization — استخدمه للـ debugging فقط.

---

#### `set_worker_desc`

```c
void set_worker_desc(const char *fmt, ...);
```

يضبط description للـ worker الحالي (max 32 chars) — بيظهر في `ps` output كـ `kworker/u4:1+my_description`. مفيد جداً للـ debugging لمعرفة أي work item بيشتغل.

- **Caller**: يستدعيها الـ work function من داخل نفسها
- **تفاصيل**: بيكتب في `worker->desc` buffer — thread-local فعلياً

---

#### `current_work`

```c
struct work_struct *current_work(void);
```

يرجع الـ `work_struct` اللي الـ worker الحالي بيشغله. لو الـ thread الحالي مش worker → يرجع `NULL`.

- **Caller**: من داخل work function لمعرفة الـ work item الحالي
- **Use case**: لو نفس الـ function بتتستخدم في أكتر من work item وعايز تعرف أنت مين

---

#### `current_is_workqueue_rescuer`

```c
bool current_is_workqueue_rescuer(void);
```

يتحقق لو الـ thread الحالي هو **rescuer worker** — ده الـ thread الاحتياطي المخصص لـ `WQ_MEM_RECLAIM` workqueues. مهم للـ code اللي محتاج يعرف لو هو شغال في memory pressure context.

---

#### `workqueue_congested`

```c
bool workqueue_congested(int cpu, struct workqueue_struct *wq);
```

يرجع `true` لو في work items inactive (مش قادرة تشتغل) في الـ wq على الـ CPU المحدد. ده مؤشر على ضغط. يستخدمه بعض الـ subsystems (زي network) لعمل backpressure.

---

#### `from_work`

```c
#define from_work(var, callback_work, work_fieldname)
    // expands to:
    container_of(callback_work, typeof(*var), work_fieldname)
```

macro مريح بديل لـ `container_of` الصريح. بدل:
```c
struct my_struct *s = container_of(work, struct my_struct, my_work);
```
تكتب:
```c
struct my_struct *s = from_work(s, work, my_work);
```

---

#### `work_on_cpu`

```c
long work_on_cpu(int cpu, long (*fn)(void *), void *arg);
```

يشغل function **بشكل synchronous** على CPU محدد — blocking call بينتظر النتيجة. يستخدمه بشكل كبير الـ ACPI code والـ firmware interfaces اللي محتاجة تشتغل على CPU 0 مثلاً.

- **`fn`**: الدالة المراد تشغيلها
- **`arg`**: argument واحد من نوع `void *`
- **Return**: قيمة الـ `long` اللي رجعتها الـ function
- **تفاصيل**: في CONFIG_SMP بيستخدم لو key مستقل per-caller لـ lockdep. في non-SMP بيستدعي الـ function مباشرة

---

#### `execute_in_process_context`

```c
int execute_in_process_context(work_func_t fn, struct execute_work *ew);
```

لو الـ current context هو process context → يستدعي `fn` مباشرة. لو في interrupt/atomic context → يعمل queue على `system_percpu_wq`.

- **Return**: 0 لو اتنفذت مباشرة، 1 لو تم الـ queue
- **Use case**: code اللي ممكن يتستدعى من أي context ومحتاج process context لبعض العمليات

---

#### `schedule_on_each_cpu`

```c
int schedule_on_each_cpu(work_func_t func);
```

يشغل `func` على **كل CPU online** بالترتيب. blocking — ينتظر الانتهاء على كل CPU. يستخدمه مثلاً `percpu_counter` flush.

- **Return**: 0 عند النجاح، negative error لو فشل الـ allocation
- **Caller**: process context فقط

---

### 7. System Workqueues المدمجة

الـ kernel بيوفر workqueues جاهزة مش محتاج تنشئها:

| الـ Workqueue | الوصف | متى تستخدمه |
|---|---|---|
| `system_percpu_wq` | per-cpu, concurrency managed | الاستخدام العام — `schedule_work()` بيضيف هنا |
| `system_highpri_wq` | per-cpu, high priority | work items تحتاج latency منخفض |
| `system_long_wq` | per-cpu, للـ long-running work | work items بتاخد وقت طويل |
| `system_unbound_wq` | unbound, no concurrency mgmt | work items مش CPU-bound |
| `system_dfl_wq` | unbound, default affinity | الـ default للـ unbound use cases |
| `system_freezable_wq` | per-cpu + freezable | يحتاج توقف أثناء suspend |
| `system_power_efficient_wq` | per-cpu أو unbound حسب kernel param | power-sensitive work |
| `system_bh_wq` | BH/softirq context | work items بتحل محل softirq handlers |
| `system_bh_highpri_wq` | BH + high priority | high priority softirq-like work |

> **تحذير**: لا تعمل `flush_workqueue` على الـ system workqueues — الـ kernel بيطبع warning عليها. كل subsystem ممكن يضيف على نفس الـ queue فالـ flush مش آمن.

---

### 8. الـ Suspend / Freeze APIs

---

#### `freeze_workqueues_begin`

```c
void freeze_workqueues_begin(void);
```

يبدأ مرحلة الـ freeze لكل الـ workqueues اللي عندها `WQ_FREEZABLE`. بيوقف تشغيل work items جديدة وينتظر الحالية تنتهي. يتستدعى من `suspend_freeze_processes()` أثناء الـ system suspend.

---

#### `freeze_workqueues_busy`

```c
bool freeze_workqueues_busy(void);
```

يتحقق لو كل الـ freezable workqueues فرغت من الـ work items. يتستخدم لـ polling بعد `freeze_workqueues_begin` للتأكد من الانتهاء.

- **Return**: `true` لو لسه في work items شغالة (لسه busy)، `false` لو تم الـ freeze بالكامل

---

#### `thaw_workqueues`

```c
void thaw_workqueues(void);
```

يرفع الـ freeze ويستأنف تشغيل الـ work items على الـ frozen workqueues. يتستدعى أثناء الـ resume path.

---

### 9. الـ CPU Hotplug APIs

تتستدعى تلقائياً من الـ CPU hotplug infrastructure — **لا تستدعيها مباشرة**.

```c
int workqueue_prepare_cpu(unsigned int cpu);  // قبل bring-up الـ CPU
int workqueue_online_cpu(unsigned int cpu);   // بعد ما الـ CPU يصبح online
int workqueue_offline_cpu(unsigned int cpu);  // قبل ما الـ CPU يصبح offline
```

الـ `workqueue_online_cpu` بينشئ الـ worker pools للـ CPU الجديد. الـ `workqueue_offline_cpu` بيعمل migrate للـ workers الموجودين لـ CPUs تانية.

---

### 10. الـ Init Functions

```c
void __init workqueue_init_early(void);
void __init workqueue_init(void);
void __init workqueue_init_topology(void);
```

- **`workqueue_init_early`**: يتستدعى في بداية `start_kernel()` — ينشئ الـ system workqueues الأساسية قبل ما الـ scheduler يكون جاهز تماماً
- **`workqueue_init`**: يتستدعى بعد ما الـ scheduler يكون جاهز — ينشئ الـ kworker threads الفعليين
- **`workqueue_init_topology`**: يتستدعى بعد اكتشاف الـ CPU topology — يضبط الـ affinity scopes حسب الـ cache وNUMA boundaries

---

### ملاحظات ختامية للـ Expert

**الـ Non-reentrance guarantee**: الـ workqueue بيضمن إن الـ work item مش هيشتغل على أكتر من worker في نفس الوقت بشرط:
1. الـ work function ما اتغيرتش
2. محدش عمل queue للـ work item على wq تاني
3. الـ work item ما اتعمللوش reinit

**الـ Memory ordering**: `queue_work` بتعمل implicit memory barrier — أي stores قبلها بتبقى visible للـ worker thread قبل ما ينفذ الـ work. ده بيلغي الحاجة لـ explicit barriers في كتير من الـ use cases.

**الـ Flush colors**: الـ kernel بيستخدم 4-bit color field في الـ `work->data` لتتبع أي "generation" من الـ work items المراد flush — ده بيسمح بـ concurrent flushes بدون false serialization.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المهمة

الـ workqueue subsystem بيكشف معلوماته الداخلية عبر `/sys/kernel/debug/workqueue/`:

```bash
# اعرض كل الـ worker pools وحالتها
ls /sys/kernel/debug/workqueue/

# اقرأ الـ pool stats لكل pool
cat /sys/kernel/debug/workqueue/pool_stats
```

| المسار | المحتوى | كيف تقرأه |
|--------|---------|-----------|
| `/sys/kernel/debug/workqueue/wq_stats` | إحصائيات لكل workqueue (total, infl, CPUtime, CPUhog) | ارقام متراكمة، قارن بين قراءتين |
| `/sys/kernel/debug/workqueue/pool_stats` | حالة كل worker-pool: idle, running, nr_workers | idle/workers بيفرق لو في مشكلة concurrency |
| `/proc/THE_KWORKER_PID/stack` | الـ call stack للـ kworker المشبوه | اعرف أي work function بتاكل الـ CPU |

```bash
# مثال عملي: اعرف stack الـ kworker المشبوه
ps aux | grep kworker
cat /proc/5671/stack
```

**مثال output:**
```
[<0>] my_work_handler+0x3c/0x80 [my_driver]
[<0>] process_one_work+0x1a5/0x3b0
[<0>] worker_thread+0x4b/0x3e0
[<0>] kthread+0x11a/0x140
```

---

#### 2. مدخلات الـ sysfs المهمة

لو الـ workqueue اتعمل بـ `WQ_SYSFS` flag، هيبقى ليه directory تحت:

```
/sys/devices/virtual/workqueue/<WQ_NAME>/
```

| الملف | القيمة | الوصف |
|-------|--------|-------|
| `affinity_scope` | `cpu / smt / cache / numa / system / default` | الـ CPU affinity scope الحالي |
| `affinity_strict` | `0` أو `1` | هل الـ affinity strict أم لا |
| `cpumask` | hex bitmask | الـ CPUs المسموح بيها |
| `max_active` | رقم | أقصى عدد work items متزامنة per-CPU |

```bash
# اقرأ الـ affinity scope لـ kcryptd مثلاً
cat /sys/devices/virtual/workqueue/kcryptd/affinity_scope

# غير الـ affinity scope live
echo cache > /sys/devices/virtual/workqueue/kcryptd/affinity_scope

# اعرف الـ cpumask المسموح بيه
cat /sys/devices/virtual/workqueue/kcryptd/cpumask
```

---

#### 3. الـ ftrace: Tracepoints وإزاي تشغلهم

الـ workqueue subsystem عنده tracepoints جاهزة في `kernel/workqueue.c`:

| الـ Tracepoint | متى بيتفعل | الفايدة |
|---------------|-----------|--------|
| `workqueue:workqueue_queue_work` | لما work item يتحط في الـ queue | اكتشف busy-looping |
| `workqueue:workqueue_execute_start` | أول ما worker يبدأ تنفيذ work | قيس latency من queue لـ execute |
| `workqueue:workqueue_execute_end` | لما work ينتهي | احسب execution time |
| `workqueue:workqueue_activate_work` | لما work ينتقل من inactive لـ active | تابع الـ throttling |

```bash
# شغّل tracing لمشكلة busy-looping
echo workqueue:workqueue_queue_work > /sys/kernel/tracing/set_event
cat /sys/kernel/tracing/trace_pipe > /tmp/wq_trace.txt &
sleep 5
kill %1

# شغّل كل الـ workqueue events دفعة واحدة
echo 'workqueue:*' > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe

# ابحث في الـ output عن الـ function اللي بتتكرر كتير
grep "workqueue_queue_work" /tmp/wq_trace.txt | awk '{print $NF}' | sort | uniq -c | sort -rn | head -20
```

**مثال output يدل على مشكلة:**
```
  4523 func=my_buggy_work_fn workqueue=events work=0xffff...
  4521 func=my_buggy_work_fn workqueue=events work=0xffff...
```
لو نفس الـ `func` بيتكرر بجنون → الـ work function دي بتعمل re-queue على طول.

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لكل kernel/workqueue.c
echo 'file kernel/workqueue.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل لـ module معين بيستخدم workqueue
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

# اعرض كل الـ debug messages الفعّالة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep workqueue
```

لو بتعمل driver وعايز تضيف debug prints داخل الـ work function:

```c
static void my_work_handler(struct work_struct *work)
{
    /* pr_debug بيتحكم فيه dynamic_debug */
    pr_debug("my_work_handler: starting on CPU %d\n", smp_processor_id());

    /* ... شغل الـ work ... */

    pr_debug("my_work_handler: done\n");
}
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config | الوصف | متى تفعّله |
|-----------|-------|-----------|
| `CONFIG_DEBUG_OBJECTS_WORK` | يكشف use-after-free وdouble-free على الـ work structs | أي مشكلة corruption في work items |
| `CONFIG_WQ_WATCHDOG` | watchdog بيصرخ لو worker pool اتوقف عن الشغل | اكتشف stalls وdeadlocks |
| `CONFIG_LOCKDEP` | يتبع لو work item اتنفذ من context خطأ | مشاكل locking معقدة |
| `CONFIG_PROVE_LOCKING` | إثبات صحة الـ lock ordering | deadlocks في workqueue |
| `CONFIG_DEBUG_WORKQUEUE` | debug إضافي للـ cmwq internals | فهم السلوك الداخلي |
| `CONFIG_TRACING` | يفعّل كل الـ tracepoints | أي profiling أو tracing |
| `CONFIG_FUNCTION_TRACER` | يفعّل ftrace function tracing | تتبع execution flow |

```bash
# اعرض الـ config الحالي
grep -E "CONFIG_(DEBUG_OBJECTS_WORK|WQ_WATCHDOG|LOCKDEP|DEBUG_WORKQUEUE)" /boot/config-$(uname -r)
```

---

#### 6. أدوات الـ Workqueue المتخصصة

الـ kernel بييجي مع Python scripts جاهزة:

```bash
# 1. wq_dump.py: اعرض تكوين الـ workqueues والـ worker pools
python3 tools/workqueue/wq_dump.py

# 2. wq_monitor.py: راقب workqueue operations في real-time
python3 tools/workqueue/wq_monitor.py
# راقب workqueue معينة بالاسم
python3 tools/workqueue/wq_monitor.py events

# 3. اعرض كل الـ kworker threads وأرقام الـ PID
ps -eo pid,comm | grep kworker

# 4. اعرف أي workqueue بتستخدم pool معين
python3 tools/workqueue/wq_dump.py | grep -A5 "Workqueue CPU"
```

**مثال output من wq_monitor.py وكيف تفسره:**

```
                            total  infl  CPUtime  CPUhog CMW/RPR  mayday rescued
events                      18545     0      6.1       0       5       -       -
events_unbound              38306     0      0.1       -       7       -       -
```

| العمود | المعنى |
|--------|--------|
| `total` | إجمالي الـ work items اللي اتنفذت |
| `infl` | الـ work items اللي شغالة دلوقتي (in-flight) |
| `CPUtime` | وقت الـ CPU المستهلك بالثواني |
| `CPUhog` | عدد المرات اللي work item اكلت CPU أكتر من اللازم |
| `CMW/RPR` | concurrency management wakeups / repatriations |
| `mayday` | طلبات الـ rescue worker (علامة خطر لو كبيرة) |
| `rescued` | الـ work items اللي الـ rescue worker أنقذها |

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ kernel | المعنى | الحل |
|-----------------|--------|------|
| `BUG: workqueue lockup - pool cpus=X node=Y flags=0x0 nice=0 stuck for Ns!` | worker pool اتوقف عن المعالجة (deadlock أو كل الـ workers blocked) | فعّل `CONFIG_WQ_WATCHDOG`، افحص stack الـ workers بـ `cat /proc/PID/stack` |
| `WARNING: CPU: X PID: Y at kernel/workqueue.c:NNN check_flush_dependency` | flush من context غلط — مثلاً flush من داخل work function نفسها | استخدم `flush_work()` بس من خارج الـ workqueue أو استخدم `cancel_work_sync()` |
| `WARN_ON(in_atomic())` داخل work handler | work function بتحاول تنام وهي في atomic context (أو BH workqueue) | BH workqueues ممنوع فيها sleep، حوّل لـ threaded workqueue |
| `kernel BUG at kernel/workqueue.c:NNN` مع `work already on a list` | double-queue لنفس الـ work item | استخدم `schedule_work()` مش `queue_work()` مباشرة، أو تأكد من `work_pending()` قبل queue |
| `use-after-free on work_struct` | الـ work struct اتحرر قبل ما يتنفذ | استخدم `cancel_work_sync()` قبل الـ free |
| `pool NNN: workers = N, nr_running = N, nr_idle = N, nr_unbound_pwq = N` لو كل الـ workers blocked | memory pressure deadlock — كل الـ workers في reclaim path | تأكد إن الـ wq اللي في reclaim path عندها `WQ_MEM_RECLAIM` |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
static void my_work_handler(struct work_struct *work)
{
    struct my_data *data = container_of(work, struct my_data, work);

    /* تأكد إننا مش في atomic context */
    WARN_ON(in_atomic());
    WARN_ON(in_interrupt());

    /* تأكد إن الـ data لسه valid */
    WARN_ON(!data || IS_ERR(data));

    /* لو work بتتنفذ على CPU معين وده مش مطلوب */
    if (unlikely(smp_processor_id() != data->expected_cpu)) {
        pr_warn("work running on wrong CPU %d expected %d\n",
                smp_processor_id(), data->expected_cpu);
        dump_stack(); /* اطبع الـ stack عشان نفهم من أين جاء الـ queue */
    }
}

/* عند تدمير الـ workqueue أو الـ driver */
static void my_driver_remove(struct platform_device *pdev)
{
    struct my_data *data = platform_get_drvdata(pdev);

    /* WARN لو في work لسه pending */
    WARN_ON(work_pending(&data->work));

    cancel_work_sync(&data->work); /* انتظر أي work شغّال ينتهي */
    destroy_workqueue(data->wq);
}
```

---

### Hardware Level

#### 1. تحقق إن الـ Hardware State متطابق مع الـ Kernel State

الـ workqueue subsystem software-only، بس لو الـ work items بتتعامل مع hardware:

```bash
# تحقق إن الـ interrupt lines شغّالة صح (الـ work بيتقال من IRQ handler عادةً)
cat /proc/interrupts | grep -i "my_device"

# لو الـ work queue بتستخدم في driver معين، تحقق إن الـ device موجودة
lspci -v | grep -i "my_device"
# أو
ls /sys/bus/platform/devices/

# تحقق من الـ IRQ affinity لو الـ work يتقال من ISR
cat /proc/irq/NNN/smp_affinity
```

---

#### 2. تقنيات الـ Register Dump

الـ workqueue نفسه ما بيتعامل مع hardware registers، بس الـ drivers اللي بتستخدمه ممكن تحتاج:

```bash
# devmem2: اقرأ register بـ physical address
# تثبيت: apt install devmem2
devmem2 0xFE200000 w   # اقرأ 32-bit register

# /dev/mem مباشرة (محتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0xFE200000/4)) | xxd

# io utility (من package ioport)
io -4 0x3F8   # اقرأ IO port

# لو الـ driver بيعمل register dump في debugfs
cat /sys/kernel/debug/my_driver/registers
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو الـ work items بتتحكم في hardware signals:

- **راقب الـ interrupt line**: المفروض الـ IRQ يحصل → work يتقال → يتنفذ في وقت معقول (أقل من 1ms عادةً لـ normal priority).
- **قيس الـ latency** من rising edge على الـ interrupt pin لحد أول memory write من الـ work function باستخدام GPIO toggling:

```c
static void my_work_handler(struct work_struct *work)
{
    /* Toggle GPIO للقياس بـ oscilloscope */
    gpiod_set_value(debug_gpio, 1);

    /* ... الشغل الحقيقي ... */

    gpiod_set_value(debug_gpio, 0);
}
```

- **لو الـ work بتأخر** أكتر من المتوقع على الـ scope: احتمال الـ worker pool مشغول أو الـ CPU تحت load عالي.

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في الـ Kernel Log | التشخيص |
|--------------------|----------------------|---------|
| **IRQ storm** → work items بتتكدس | `workqueue_queue_work` events بتتكرر بسرعة جنونية في ftrace | راجع الـ IRQ handler، تأكد من clear الـ interrupt في الـ hardware |
| **DMA stall** → work ينتظر DMA اللي ما بيكملش | `pool NNN: stuck for 30s` مع stack يظهر `dma_wait` | افحص الـ DMA controller registers، تحقق من الـ DMA completion interrupt |
| **Power management** → CPU أوف وwork يستنى | كثير `workqueue_activate_work` بدون `execute_start` | افحص الـ cpuidle stats، قد تحتاج `pm_qos_add_request` |
| **Clock disabled** → work blocked في hardware access | `BUG: scheduling while atomic` لو في timeout | تحقق من الـ clock enable قبل ما تقال الـ work |

---

#### 5. تحقق من الـ Device Tree

الـ workqueue نفسه مش له DT entries، بس الـ devices اللي بتستخدمه بتحتاج DT صح:

```bash
# اعرض الـ DT المحمّل فعلاً في الـ kernel
ls /sys/firmware/devicetree/base/

# تحقق من الـ interrupts property للـ device
cat /sys/firmware/devicetree/base/soc/my_device/interrupts | xxd

# قارن الـ DT المترجم مع الـ DTS source
dtc -I fs /sys/firmware/devicetree/base > /tmp/current.dts
diff /tmp/current.dts my_board.dts

# تحقق من الـ compatible string
grep -r "my,device-v1" /sys/firmware/devicetree/
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === WQ Debug Toolkit ===

# 1. عرض كل الـ kworkers وCPU usage
echo "=== kworker CPU usage ==="
ps -eo pid,pcpu,comm --sort=-pcpu | grep kworker | head -20

# 2. stack الـ kworker الأكتر CPU usage
TOP_KWORKER=$(ps -eo pid,pcpu,comm --sort=-pcpu | grep kworker | head -1 | awk '{print $1}')
echo "=== Stack of PID $TOP_KWORKER ==="
cat /proc/$TOP_KWORKER/stack

# 3. تشغيل workqueue queue tracing لمدة 5 ثواني
echo "=== Enabling workqueue tracing for 5s ==="
echo workqueue:workqueue_queue_work > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/tracing/tracing_on
echo "=== Top queued work functions ==="
grep workqueue_queue_work /sys/kernel/tracing/trace \
    | grep -oP 'func=\S+' | sort | uniq -c | sort -rn | head -10
echo "" > /sys/kernel/tracing/trace  # clear

# 4. عرض workqueue stats من wq_monitor style
echo "=== Workqueue pool info ==="
python3 tools/workqueue/wq_dump.py 2>/dev/null || \
    cat /sys/kernel/debug/workqueue/wq_stats 2>/dev/null

# 5. فحص الـ WQ_WATCHDOG timeout
dmesg | grep -i "workqueue.*lockup\|stuck for\|mayday"
```

---

#### تشخيص busy-looping work item خطوة بخطوة

```bash
# خطوة 1: افتح الـ tracing
echo workqueue:workqueue_queue_work > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/tracing_on

# خطوة 2: استنى فترة قصيرة
sleep 3

# خطوة 3: وقف الـ tracing واحفظ
echo 0 > /sys/kernel/tracing/tracing_on
cp /sys/kernel/tracing/trace /tmp/wq_capture.txt

# خطوة 4: حلل النتيجة
echo "Top work functions by queue frequency:"
grep -oP 'func=\K\S+' /tmp/wq_capture.txt \
    | sort | uniq -c | sort -rn | head -5

echo ""
echo "Top workqueues by frequency:"
grep -oP 'workqueue=\K\S+' /tmp/wq_capture.txt \
    | sort | uniq -c | sort -rn | head -5
```

**مثال output وتفسيره:**
```
Top work functions by queue frequency:
   8921 ieee80211_iface_work          ← كتير أوي، محتمل مشكلة في WiFi driver
     45 vmstat_shepherd
     12 flush_to_ldisc
```

الـ `ieee80211_iface_work` بيتقال ~9000 مرة في 3 ثواني = ~3000/ثانية → دي مشكلة واضحة.

---

#### تشخيص stall / deadlock في workqueue

```bash
# 1. فعّل WQ_WATCHDOG في kernel config أو عبر sysctl
echo 30 > /sys/module/workqueue/parameters/watchdog_thresh  # 30 ثانية timeout

# 2. لو لقيت رسالة lockup في dmesg، اعرف الـ stuck workers
dmesg | grep -A 20 "workqueue.*lockup"

# 3. اعرض stack كل الـ kworkers المتوقفة
for pid in $(ps aux | grep kworker | awk '{print $2}'); do
    state=$(cat /proc/$pid/status 2>/dev/null | grep State | awk '{print $2}')
    if [ "$state" = "D" ]; then  # D = uninterruptible sleep = blocked
        echo "=== Blocked kworker PID $pid ==="
        cat /proc/$pid/stack
        echo ""
    fi
done
```

---

#### قياس latency من queue لـ execute

```bash
# شغّل execute_start و execute_end معاً
echo 'workqueue:workqueue_queue_work workqueue:workqueue_execute_start workqueue:workqueue_execute_end' \
    > /sys/kernel/tracing/set_event
echo 1 > /sys/kernel/tracing/tracing_on
sleep 2
echo 0 > /sys/kernel/tracing/tracing_on

# اعرض الـ timestamps
grep -E "workqueue_queue_work|workqueue_execute" /sys/kernel/tracing/trace | head -40
```

الـ timestamp بالـ microseconds. الفرق بين `queue_work` و `execute_start` هو الـ **scheduling latency**. لو كبير → الـ worker pool مشغول أو الـ CPU تحت ضغط.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — تجميد النظام أثناء memory reclaim

#### العنوان
**الـ workqueue بدون `WQ_MEM_RECLAIM` يسبب deadlock في gateway صناعي**

#### السياق
شركة تصنع industrial gateway بناءً على **Texas Instruments AM62x** يشغّل Linux 6.6. الـ gateway بيجمع بيانات من 8 حساسات عبر I2C ويرسلها عبر Ethernet. كل حساس بيبعث burst من البيانات كل 100ms، والـ driver بيستخدم workqueue لمعالجة البيانات وكتابتها على flash storage.

#### المشكلة
بعد تشغيل 6 ساعات، النظام بيتجمد تماماً. لا kernel panic، لا أي response. الـ watchdog بيعمل reboot بعد 30 ثانية. في الـ kmsg قبل الـ freeze:

```
INFO: task kworker/0:2:123 blocked for more than 120 seconds.
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
kworker/0:2        D    0   123      2 0x00000000
Call Trace:
 schedule+0x74/0xf0
 schedule_timeout+0x118/0x148
 wait_for_completion+0x80/0x110
 flush_work+0x9c/0x110
 ...
 kmalloc -> page_alloc -> direct_reclaim -> ...
```

#### التحليل
الـ documentation بتقول صراحةً:

> *"All wq which might be used in the memory reclaim paths **MUST** have this flag set."*

الـ driver عمل:
```c
/* WRONG: missing WQ_MEM_RECLAIM */
wq = alloc_workqueue("sensor_proc", 0, 0);
```

لما الـ system وصل لـ memory pressure، الـ direct reclaim path احتاج يخلص work item عشان يحرر memory. لكن الـ worker pool نفسه كان محتاج memory عشان يخلق worker جديد. **Deadlock كلاسيكي.**

الـ cmwq بتحل ده بـ **rescue workers** — كل wq عليها `WQ_MEM_RECLAIM` بيكون عندها rescue worker محجوز مسبقاً ومش محتاج يعمل allocation وقت الأزمة. بدون الـ flag، مفيش rescue worker، والـ system بيتعلق.

#### الحل
```c
/* CORRECT: declare memory reclaim capability */
wq = alloc_workqueue("sensor_proc",
                     WQ_MEM_RECLAIM,  /* reserve rescue worker */
                     0);
```

لو في dependency بين work items متعددة في reclaim path:
```c
/* Each independent chain needs its own WQ_MEM_RECLAIM wq */
wq_read  = alloc_workqueue("sensor_read",  WQ_MEM_RECLAIM, 0);
wq_write = alloc_workqueue("sensor_write", WQ_MEM_RECLAIM, 0);
```

للتحقق:
```bash
# تأكد إن الـ rescue worker موجود
cat /sys/kernel/debug/workqueue/sensor_proc/flags
```

#### الدرس المستفاد
أي workqueue بتلمس memory allocation (kmalloc, vmalloc, page cache write) لازم تاخد `WQ_MEM_RECLAIM`. مش بس الـ path المباشرة — أي function في call chain ممكن تعمل allocation.

---

### السيناريو 2: Android TV Box على Allwinner H616 — kworker بياكل 100% CPU

#### العنوان
**busy-loop في work queueing بيخلي الـ UI يتجمد على TV box**

#### السياق
منتج **Android TV box** بناءً على **Allwinner H616** (quad-core Cortex-A53). الـ HDMI driver بيستخدم workqueue لمعالجة HPD (Hot Plug Detect) events. المستخدمين بيشتكوا إن الـ UI بيبقى laggy ومش responsive بعد ما يوصلوا شاشة HDMI.

#### المشكلة
```bash
$ top
PID   USER  PR  NI    VIRT    RES  S  %CPU  COMMAND
 891  root  20   0       0      0  R  98.2  kworker/2:1
```

الـ kworker/2:1 بياكل CPU core كامل. الـ UI thread ومش لاقي وقت يشتغل.

#### التحليل
الـ debugging section في الـ documentation بتقول:

> *"If something is busy looping on work queueing, it would be dominating the output"*

الأداة اللازمة:
```bash
$ echo workqueue:workqueue_queue_work > /sys/kernel/tracing/set_event
$ cat /sys/kernel/tracing/trace_pipe > /tmp/wq_trace.txt &
# انتظر 3 ثواني
$ grep -c "hdmi_hpd_work" /tmp/wq_trace.txt
47823
```

في 3 ثواني: ~48k queue operation — بالتأكيد busy loop.

الـ bug في الـ HDMI driver:
```c
static void hdmi_hpd_work(struct work_struct *work)
{
    struct hdmi_dev *hdmi = container_of(work, struct hdmi_dev, hpd_work);

    if (hdmi_read_reg(HDMI_PHY_STATUS) & HPD_PIN) {
        /* process connection */
        hdmi_handle_connect(hdmi);
        /* BUG: re-queues immediately without checking state */
        schedule_work(&hdmi->hpd_work);  /* <-- infinite loop! */
    }
}
```

الـ workqueue documentation بتقول إن requeue من داخل الـ work function آمن (non-reentrance guarantees محتفظ بيها)، لكن في الحالة دي الـ requeue مش مشروط بشرط صح.

#### الحل
```c
static void hdmi_hpd_work(struct work_struct *work)
{
    struct hdmi_dev *hdmi = container_of(work, struct hdmi_dev, hpd_work);
    u32 status = hdmi_read_reg(HDMI_PHY_STATUS);

    if (status & HPD_PIN) {
        hdmi_handle_connect(hdmi);
        /* Only re-queue if state actually changed */
        if (hdmi->last_status != status) {
            hdmi->last_status = status;
            schedule_delayed_work(&hdmi->hpd_dwork,
                                  msecs_to_jiffies(200)); /* debounce */
        }
    }
}
```

للتحقق السريع بدون tracing:
```bash
$ cat /proc/891/stack
[<0>] worker_thread+0x1e4/0x3c0
[<0>] hdmi_hpd_work+0x48/0xa0    # <-- الـ offender واضح
[<0>] kthread+0x10c/0x130
```

#### الدرس المستفاد
الـ `schedule_work()` من داخل الـ work function مش خطأ في حد ذاته، لكن لازم يكون مشروط بتغيير حالة حقيقي. الـ workqueue tracing tool أسرع diagnostic ممكن — 3 ثواني trace بتكشف أي busy loop.

---

### السيناريو 3: IoT Sensor على STM32MP1 — تأخر في معالجة SPI بسبب `max_active` خاطئ

#### العنوان
**`max_active=1` بيسبب تأخر تراكمي في معالجة بيانات SPI sensors**

#### السياق
بورد IoT بناءً على **STM32MP1** (dual Cortex-A7 + Cortex-M4) بيقرأ بيانات من 4 حساسات IMU عبر SPI. كل حساس بيبعث interrupt كل 10ms. الـ driver بيستخدم workqueue للمعالجة عشان ميشغلش وقت طويل في الـ interrupt context.

#### المشكلة
الـ data timestamps بتُظهر إن البيانات بتتأخر بشكل تراكمي — بعد دقيقة التأخر 500ms، وبعد 5 دقائق 3 ثواني. الـ sensors بيبعثوا في الوقت الصح، لكن المعالجة بتتأخر.

#### التحليل
الـ driver الأصلي:
```c
/* Developer thought ST wq ensures ordering */
wq = alloc_workqueue("imu_proc", WQ_UNBOUND, 1); /* max_active=1 */
```

الـ developer استخدم `max_active=1` ظناً منه إن ده بيضمن الـ ordering. لكن الـ documentation بتوضح:

> *"the combination of @max_active of 1 and WQ_UNBOUND used to achieve this behavior, this is no longer the case. Use alloc_ordered_workqueue() instead."*

ومع `max_active=1` على بورد dual-core، كل 4 sensors بيتنافسوا على execution context واحد. الـ backlog بيتراكم.

الـ monitoring:
```bash
$ tools/workqueue/wq_monitor.py imu_proc
                    total  infl  CPUtime  CPUhog CMW/RPR  mayday rescued
imu_proc              423    1      0.8       0       0       -       -
# infl=1 دايماً = bottleneck واضح
```

#### الحل
```c
/* Option A: اللو ordering مهم */
wq = alloc_ordered_workqueue("imu_proc", 0);

/* Option B: لو الـ ordering مش ضروري، اسمح لـ 4 concurrent items */
wq = alloc_workqueue("imu_proc",
                     WQ_UNBOUND,
                     4);  /* one per sensor */

/* Option C: الأبسط والأصح لمعظم cases */
wq = alloc_workqueue("imu_proc",
                     WQ_UNBOUND,
                     0);  /* let cmwq manage, default=1024 */
```

في الحالة دي الـ processing لكل sensor مستقل تماماً، فـ Option B أو C الأنسب.

للتحقق بعد الإصلاح:
```bash
$ tools/workqueue/wq_monitor.py imu_proc
                    total  infl  CPUtime
imu_proc             1823     0      1.2
# infl=0 = لا backlog
```

#### الدرس المستفاد
`max_active=1` ≠ ordered execution في الـ kernels الحديثة. لازم `alloc_ordered_workqueue()` صراحةً عشان تضمن الترتيب. وكقاعدة عامة، `max_active=0` (default) هو الأنسب إلا لو عندك سبب محدد للتقييد.

---

### السيناريو 4: Automotive ECU على i.MX8 — مشكلة suspend/resume مع `WQ_FREEZABLE`

#### العنوان
**work items بتشتغل أثناء suspend على ECU في سيارة — data corruption**

#### السياق
**Automotive ECU** بناءً على **NXP i.MX8** بيتحكم في نظام الإضاءة. الـ ECU بيدخل في sleep mode لما السيارة تتقفل. الـ driver بيستخدم workqueue لمعالجة CAN bus messages وكتابة الـ state على EEPROM عبر I2C.

#### المشكلة
بعد wake-up، الـ EEPROM أحياناً بيكون فيه corrupted data. الـ analysis بيُظهر إن الـ I2C transactions بتحصل أثناء الـ suspend sequence — بعد ما الـ I2C controller اتـ disable.

#### التحليل
الـ driver:
```c
/* Missing WQ_FREEZABLE */
wq = alloc_workqueue("ecu_can", WQ_MEM_RECLAIM, 0);
```

الـ documentation بتوضح:

> *"A freezable wq participates in the freeze phase of the system suspend operations. Work items on the wq are drained and no new work item starts execution until thawed."*

بدون `WQ_FREEZABLE`، لما الـ system يبدأ suspend:
1. الـ `pm_freeze_processes()` بتـ freeze الـ user-space
2. لكن الـ kworker threads ماشغلاش `WQ_FREEZABLE` بتفضل تشتغل
3. Work items جديدة بتتضاف للـ queue (من CAN interrupts)
4. الـ I2C controller اتـ powered down
5. Worker بيحاول يكتب على EEPROM → corruption

#### الحل
```c
/* Correct: freeze workqueue during suspend */
wq = alloc_workqueue("ecu_can",
                     WQ_MEM_RECLAIM | WQ_FREEZABLE,
                     0);
```

للتأكد من الـ behavior:
```bash
# قبل suspend، تأكد إن الـ wq بيتـ drain
$ cat /sys/kernel/debug/workqueue/ecu_can/flags
WQ_MEM_RECLAIM|WQ_FREEZABLE

# أثناء الـ suspend sequence في الـ kernel log
[  45.234] Freezing workqueues...
[  45.235] ecu_can: draining 3 work items
[  45.240] Workqueues frozen.
```

اضافةً لده، الـ driver محتاج يـ handle الـ wake-up correctly:
```c
static int ecu_resume(struct device *dev)
{
    struct ecu_priv *priv = dev_get_drvdata(dev);
    /* wq automatically thaws after resume, but re-init state */
    queue_work(priv->wq, &priv->state_sync_work);
    return 0;
}
```

#### الدرس المستفاد
في الـ embedded systems اللي بتدخل sleep/suspend، أي workqueue بتتفاعل مع hardware لازم تاخد `WQ_FREEZABLE`. مش بس لحماية الـ hardware، كمان لضمان consistent state بعد الـ resume.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — أداء USB بطيء بسبب affinity scope خاطئ

#### العنوان
**USB throughput ضعيف على RK3562 بسبب إعداد affinity scope غلط في الـ UAS driver**

#### السياق
فريق bring-up بيختبر بورد مخصص بناءً على **Rockchip RK3562** (quad Cortex-A53) متصل بـ NVMe over USB 3.2 عبر UAS (USB Attached SCSI). المشروع: industrial storage appliance. الـ target: 400 MBps read throughput. القراءة الفعلية: 180 MBps فقط.

#### المشكلة
الـ CPU utilization منخفض (40%)، والـ NVMe نفسه بيعمل 450 MBps على بورد x86. المشكلة في الـ Linux workqueue configuration للـ UAS processing.

#### التحليل
الـ RK3562 عنده quad-core بـ L2 cache مشترك لكل الـ cores (single L3 shared). الـ cmwq default affinity scope هو "cache".

```bash
$ tools/workqueue/wq_dump.py
CACHE (default)
  nr_pods  1
  pod_cpus [0]=0000000f   # كل الـ 4 cores في pod واحد
  cpu_pod  [0]=0 [1]=0 [2]=0 [3]=0
```

في الحالة دي، الـ "cache" scope عملياً = "system" scope لأن كل الـ cores بيشاركوا نفس الـ L2/L3. المشكلة مش في الـ scope نفسه، بل في إن الـ UAS driver بيستخدم `WQ_UNBOUND` مع strict cache affinity وعنده `affinity_strict=1` بشكل غلط من config قديم.

```bash
$ cat /sys/devices/virtual/workqueue/uas/affinity_scope
cache (strict)

$ cat /sys/devices/virtual/workqueue/uas/affinity_strict
1
```

الـ strict mode بيمنع الـ workers من الانتقال لـ cores تانية حتى لو في work items منتظرة. مع quad-core mono-L2 topology، ده بيخلق false bottleneck.

الـ documentation بتقول:

> *"cache (strict)" shows whopping 20% bandwidth loss* (في الـ scenario اللي فيه insufficient issuers)

وعلى الـ RK3562 مع single storage issuer الـ loss أكبر بكثير.

#### الحل

**Option A: تغيير runtime عبر sysfs**
```bash
# تعطيل strict mode فوراً
$ echo 0 > /sys/devices/virtual/workqueue/uas/affinity_strict

# أو تغيير الـ scope لـ system
$ echo system > /sys/devices/virtual/workqueue/uas/affinity_scope
```

**Option B: في الـ driver code**
```c
static int uas_probe(struct usb_interface *intf,
                     const struct usb_device_id *id)
{
    struct workqueue_attrs *attrs;

    /* ... */

    attrs = alloc_workqueue_attrs();
    if (!attrs)
        return -ENOMEM;

    /* On single-L2 SoCs, 'system' scope is better than strict 'cache' */
    attrs->affinity_scope = WQ_AFFINITY_SYSTEM;
    apply_workqueue_attrs(uas_wq, attrs);
    free_workqueue_attrs(attrs);
}
```

**Option C: module parameter للـ flexibility**
```bash
$ modprobe uas workqueue_affinity_scope=system
```

للتحقق بعد الإصلاح:
```bash
$ tools/workqueue/wq_monitor.py uas
                    total  infl  CPUtime  CPUhog
uas                 15234     2     12.4       0
# CPUtime ارتفع = workers بيشتغلوا على كل الـ cores

$ fio --filename=/dev/sdb --direct=1 --rw=read --bs=128k \
      --ioengine=libaio --iodepth=32 --runtime=30 --name=test
READ: bw=398MiB/s  # قريب من الـ target
```

التحسن: من 180 MBps لـ 398 MBps بتغيير سطر واحد.

#### الدرس المستفاد
الـ affinity scope "cache" مش دايماً هو الأمثل. على SoCs الصغيرة بـ single shared cache، الـ "system" أو non-strict "cache" أحسن. لازم تفهم الـ topology الفعلي للـ CPU باستخدام `wq_dump.py` قبل ما تقرر الإعداد. الـ `WQ_SYSFS` flag ضروري في production boards عشان تقدر تعمل tuning بدون recompile.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لمتابعة تطور الـ workqueue في الـ kernel. الروابط دي حقيقية ومتحققة:

| المقال | الأهمية |
|--------|---------|
| [Details of the workqueue interface](https://lwn.net/Articles/11360/) | أول توثيق للـ API الأصلي — تاريخي |
| [Driver porting: the workqueue interface](https://lwn.net/Articles/23634/) | دليل عملي لاستخدام الـ workqueue في الـ drivers |
| [Workqueues get a rework](https://lwn.net/Articles/211279/) | أول محاولة إعادة هيكلة قبل الـ cmwq |
| [Overview of concurrency managed workqueue](https://lwn.net/Articles/393172/) | المقال الأساسي عن الـ cmwq — لازم تقرأه |
| [Concurrency-managed workqueues and thread priorities](https://lwn.net/Articles/393171/) | تفاصيل الـ priority في الـ cmwq |
| [workqueue: concurrency managed workqueue, take#6](https://lwn.net/Articles/394084/) | patchset نهائي قبل الـ merge في 2.6.36 |
| [Working on workqueues](https://lwn.net/Articles/403891/) | تغطية دخول الـ cmwq في kernel 2.6.36 |
| [Power-efficient workqueues](https://lwn.net/Articles/731052/) | الـ `WQ_POWER_EFFICIENT` flag وتأثيره على الـ power management |
| [workqueue: Improve unbound workqueue execution locality](https://lwn.net/Articles/932431/) | الـ affinity scopes — جديد في v6.5 |
| [A pair of workqueue improvements](https://lwn.net/Articles/937416/) | تحسينات إضافية على الـ unbound pools |

---

### توثيق الـ Kernel الرسمي

**الـ Documentation/** في الـ kernel source هو المرجع الأول دايمًا:

```
Documentation/core-api/workqueue.rst
```
→ الملف الأساسي اللي بنعمل توثيقه، كتبه **Tejun Heo** و **Florian Mickler** سنة 2010.

**الـ online version على kernel.org:**
- [docs.kernel.org/core-api/workqueue.html](https://docs.kernel.org/core-api/workqueue.html) — أحدث نسخة
- [kernel.org/doc/html/v4.10/core-api/workqueue.html](https://www.kernel.org/doc/html/v4.10/core-api/workqueue.html) — نسخة v4.10 للمقارنة التاريخية

**الـ header الرئيسي:**
```
include/linux/workqueue.h
```
→ فيه كل الـ macros والـ API اللي بيستخدمها الـ driver developers.

**الـ implementation:**
```
kernel/workqueue.c
```
→ الـ source code الكامل للـ cmwq — متاح على [GitHub](https://github.com/torvalds/linux/blob/master/kernel/workqueue.c).

**أدوات الـ monitoring والـ debugging:**
```
tools/workqueue/wq_dump.py
tools/workqueue/wq_monitor.py
```

---

### Kernel Commits المهمة

الـ commits دي هي نقاط التحول في تاريخ الـ workqueue:

| الحدث | الـ kernel version | الـ commit / الـ patch |
|-------|-------------------|----------------------|
| إدخال الـ cmwq الأصلي | v2.6.36 | [patchset take#4 — lkml](https://lkml.kernel.org/lkml/1267187000-18791-31-git-send-email-tj@kernel.org/t/) |
| إضافة الـ sysfs interface للـ unbound pools | v3.10 | kernelnewbies: [Linux_3.10](https://kernelnewbies.org/Linux_3.10) |
| إضافة الـ lockup detector | v4.5 | kernelnewbies: [Linux_4.5](https://kernelnewbies.org/Linux_4.5) |
| إدخال الـ affinity scopes | v6.5 | LWN: [932431](https://lwn.net/Articles/932431/) |
| إدخال الـ BH workqueues | v6.x | docs.kernel.org |

---

### Mailing List

الـ **LKML** هو المكان الأساسي لمتابعة النقاشات:

- **الأرشيف الرسمي:** [lore.kernel.org/lkml](https://lore.kernel.org/lkml/) — ابحث بـ `workqueue` أو `cmwq`
- **الأرشيف البديل:** [lkml.org](https://lkml.org/) — أسهل في البحث
- **مثال على نقاش حقيقي:** [WARN at kernel/workqueue.c](https://lists.archive.carbon60.com/linux/kernel/1925238) — debugging discussion
- **الـ patchset الأصلي للـ cmwq:** [lkml.kernel.org/lkml/...tj@kernel.org](https://lkml.kernel.org/lkml/1267187000-18791-31-git-send-email-tj@kernel.org/t/)

---

### KernelNewbies

**الـ [kernelnewbies.org](https://kernelnewbies.org)** بيغطي كل release notes وبيشرح التغييرات بأسلوب أبسط:

- [Linux_2_6_36](https://kernelnewbies.org/Linux_2_6_36) — الـ release اللي دخل فيه الـ cmwq
- [Linux_3.10](https://kernelnewbies.org/Linux_3.10) — إضافة الـ sysfs workqueue attributes
- [Linux_4.0](https://kernelnewbies.org/Linux_4.0) — تحسينات على الـ worker pools
- [Linux_4.5](https://kernelnewbies.org/Linux_4.5) — الـ lockup detector
- [Linux_6.0](https://kernelnewbies.org/Linux_6.0) — تحسينات الـ unbound workqueues
- [Linux_6.12](https://kernelnewbies.org/Linux_6.12) — أحدث تغييرات
- [Tejun_Heo](https://kernelnewbies.org/Tejun_Heo) — صفحة المطور الرئيسي للـ cmwq
- [Tasklet vs workqueues — mailing list](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-August/002793.html) — مقارنة مفيدة للـ beginners

---

### eLinux.org

البحث على elinux.org ما أرجعش صفحة مخصصة للـ workqueue. الـ embedded Linux resources عن الموضوع ده موجودة في المصادر التانية المذكورة. للـ embedded context، الأفضل تبص على:

- [elinux.org — High Resolution Timers](https://elinux.org/High_Resolution_Timers) — مرتبط بالـ deferred execution بشكل عام
- [elinux.org — Main Page](https://elinux.org/Main_Page) — واسرح في الـ kernel internals section

---

### الكتب الموصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل ذو الصلة:** Chapter 7 — *Time, Delays, and Deferred Work*
- **ملاحظة:** الكتاب ده عمل، بس الـ workqueue API اتغيرت كتير من وقته. استخدمه للمفاهيم مش للـ API الحرفي.
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل ذو الصلة:** Chapter 8 — *Bottom Halves and Deferring Work*
- يشرح الـ workqueues جنب الـ tasklets والـ softirqs ويوضح الفرق بينهم
- أحسن كتاب لفهم الـ big picture للـ deferred execution في الـ kernel

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل ذو الصلة:** Chapter 14 — *Kernel Debugging Techniques* + Chapter 16
- مناسب لمن يشتغل على الـ embedded systems ويحتاج يفهم الـ workqueue في context الـ embedded

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)
- **الفصل ذو الصلة:** Chapter 4 — *Interrupts and Exceptions*
- بيغطي الـ work queues كجزء من منظومة الـ interrupt handling

---

### مصادر إضافية

- **Async execution with workqueues (PDF — Linux Foundation):**
  [events.static.linuxfound.org/.../Async execution with wqs.pdf](https://events.static.linuxfound.org/sites/events/files/slides/Async%20execution%20with%20wqs.pdf)
  → سلايدز من **Bhaktipriya Shridhar** — عملية ومفيدة جداً للـ developers

---

### Search Terms للبحث عن معلومات أكثر

لو عايز تعمق أكثر، الـ keywords دي هتجيب نتايج ممتازة:

```
# للبحث في الـ LKML
workqueue cmwq site:lore.kernel.org
"alloc_workqueue" site:lore.kernel.org
"WQ_MEM_RECLAIM" deadlock site:lkml.org

# للبحث في LWN
workqueue site:lwn.net
"worker pool" concurrency site:lwn.net

# للبحث في الـ kernel source
git log --oneline -- kernel/workqueue.c
git log --oneline -- include/linux/workqueue.h
grep -r "alloc_workqueue" drivers/

# للبحث العام
linux kernel "concurrency managed workqueue" internals
cmwq "worker-pool" "rescue worker" linux
linux workqueue "BH workqueue" softirq
linux workqueue affinity scope v6.5
```
## Phase 8: Writing simple module

### الفكرة: Hook على tracepoint الـ `workqueue_queue_work`

الـ documentation نفسه بيقول إن أسهل طريقة لتتبع مين بيـqueue شغل كتير هي الـ tracepoint ده:

```bash
$ echo workqueue:workqueue_queue_work > /sys/kernel/tracing/set_event
```

ده tracepoint موجود جوه kernel الـ workqueue subsystem — بيتفعّل كل ما حد يعمل `queue_work()` أو `schedule_work()`. هنعمل module بيـhook عليه ويطبع اسم الـ workqueue، الـ CPU، واسم الـ work function.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * wq_trace_mon.c
 *
 * Hooks into the workqueue_queue_work tracepoint to observe
 * every work item being queued system-wide.
 *
 * Build with a Makefile that sets:
 *   obj-m := wq_trace_mon.o
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>       /* pr_info                                 */
#include <linux/tracepoint.h>   /* for_each_kernel_tracepoint, tracepoint  */
#include <linux/workqueue.h>    /* work_struct, pool_workqueue              */

/* Tracepoint events live in their own generated header */
#define CREATE_TRACE_POINTS
#include <trace/events/workqueue.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("Monitor workqueue_queue_work tracepoint");

/* ------------------------------------------------------------------ */
/*  Callback — called every time queue_work() fires the tracepoint     */
/* ------------------------------------------------------------------ */
/*
 * الـ prototype ده مطابق للي kernel بيتوقعه من handler الـ tracepoint:
 *   - pwq  : الـ pool_workqueue اللي الـ work هيتحط فيه
 *   - work : الـ work_struct نفسه
 *   - req_cpu : الـ CPU اللي طلب الـ queue
 *   - cpu     : الـ CPU اللي هيتنفذ عليه الـ work فعلاً
 */
static void on_queue_work(void *ignore,
                          struct work_struct *work,
                          const char *workqueue_name,
                          int req_cpu,
                          int cpu)
{
    /*
     * بنطبع:
     *   - اسم الـ workqueue (زي "events" أو "kblockd")
     *   - عنوان الـ work function — ممكن تحوّله لاسم بـ kallsyms لو محتاج
     *   - الـ CPU اللي طلب الـ queue والـ CPU اللي هيتنفذ عليه
     */
    pr_info("wq_mon: wq=%-24s func=%pF req_cpu=%d exec_cpu=%d\n",
            workqueue_name,
            work->func,
            req_cpu,
            cpu);
}

/* ------------------------------------------------------------------ */
/*  module_init — register the tracepoint probe                        */
/* ------------------------------------------------------------------ */
static int __init wq_trace_mon_init(void)
{
    int ret;

    /*
     * register_trace_workqueue_queue_work() ده macro بيولّده kernel
     * من تعريف الـ tracepoint في trace/events/workqueue.h.
     * بيربط الـ callback بتاعنا بالـ tracepoint بشكل atomic-safe.
     */
    ret = register_trace_workqueue_queue_work(on_queue_work, NULL);
    if (ret) {
        pr_err("wq_mon: failed to register tracepoint probe: %d\n", ret);
        return ret;
    }

    pr_info("wq_mon: loaded — watching all workqueue_queue_work events\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit — MUST unregister before the module text disappears    */
/* ------------------------------------------------------------------ */
static void __exit wq_trace_mon_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما kernel يشيل كود الـ module من الذاكرة،
     * لأن لو الـ tracepoint اتفعّل بعد unload هيعمل page fault (use-after-free).
     * tracepoint_synchronize_unregister() بتستنى كل RCU read-side critical
     * sections تخلص قبل ما الـ function ترجع.
     */
    unregister_trace_workqueue_queue_work(on_queue_work, NULL);
    tracepoint_synchronize_unregister();
    pr_info("wq_mon: unloaded\n");
}

module_init(wq_trace_mon_init);
module_exit(wq_trace_mon_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية: `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info`, `pr_err` |
| `linux/tracepoint.h` | `register_trace_*` / `unregister_trace_*` / `tracepoint_synchronize_unregister` |
| `linux/workqueue.h` | تعريف `work_struct` اللي بنستخدمه في الـ callback |
| `trace/events/workqueue.h` | الـ tracepoint declarations الخاصة بالـ workqueue subsystem |

**الـ `CREATE_TRACE_POINTS`** macro لازم يتعرّف قبل الـ include عشان kernel يولّد الـ `register_trace_workqueue_queue_work()` function في الـ translation unit ده.

---

#### الـ Callback: `on_queue_work`

```
void *ignore       ← بيان الـ "data" اللي بعتناه في register (NULL هنا)
work_struct *work  ← الـ work item اللي اتأضاف للـ queue
workqueue_name     ← اسم الـ workqueue زي "events_highpri"
req_cpu            ← الـ CPU اللي نادى queue_work() — ممكن يكون WORK_CPU_UNBOUND
cpu                ← الـ CPU اللي هيتنفذ عليه فعلاً
```

الـ `%pF` format specifier بيطبع اسم الـ function من رمزه (kernel symbol) — يعني بدل ما تشوف `0xffffffffc0ab1234` هتشوف `my_driver_work_handler`.

---

#### الـ `module_init`

**الـ `register_trace_workqueue_queue_work()`** بيعمل حاجتين:
1. بيضيف الـ callback لـ linked list الـ probes الخاصة بالـ tracepoint.
2. بيعمل memory barrier عشان يضمن إن الـ probe تبقى visible لكل الـ CPUs.

لو فضل error (مثلاً: `ENOENT` لو الـ tracepoint مش موجود في الـ kernel الحالي)، بنطبع الـ error ونرجع فوراً.

---

#### الـ `module_exit`

الـ `unregister_trace_workqueue_queue_work()` بيشيل الـ callback من الـ probe list.
الـ `tracepoint_synchronize_unregister()` بتستنى RCU grace period كاملة عشان تضمن إن محدش لسه شغّال في الـ callback وقت ما الـ module بيتـunload — من غير ده هيحصل kernel crash.

---

### Makefile

```makefile
obj-m := wq_trace_mon.o

KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء الـ module
$ make

# تحميله
$ sudo insmod wq_trace_mon.ko

# شوف الـ output في kernel log
$ sudo dmesg -w | grep wq_mon

# مثال على output
# [  123.456] wq_mon: wq=events                    func=vmstat_update       req_cpu=2 exec_cpu=2
# [  123.457] wq_mon: wq=events_highpri             func=blk_mq_run_hw_queue req_cpu=0 exec_cpu=0
# [  123.460] wq_mon: wq=kblockd                    func=blk_mq_requeue_work req_cpu=1 exec_cpu=-1

# تفريغه
$ sudo rmmod wq_trace_mon
```

> **ملاحظة:** الـ `exec_cpu=-1` معناه `WORK_CPU_UNBOUND` — الـ work هيتنفذ على أي CPU متاح.

---

### ليه الـ tracepoint وأهميته

الـ documentation صراحةً بيقول إن أسرع طريقة لاكتشاف مين بيعمل busy-loop على الـ workqueue هي:

```bash
$ echo workqueue:workqueue_queue_work > /sys/kernel/tracing/set_event
```

الـ module ده بيعمل نفس الشيء من kernel-space بدل user-space tracing، وبيديك flexibility تفلتر أو تحسب statistics جوه الـ kernel نفسه بدون overhead الـ trace buffer.
