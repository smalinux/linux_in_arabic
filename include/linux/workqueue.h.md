## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

**الـ Workqueue** subsystem جزء من نواة Linux وبيتولاه Tejun Heo. الملفات الأساسية بتاعته:

| الملف | الدور |
|---|---|
| `include/linux/workqueue_types.h` | تعريف الـ structs الأساسية (`work_struct`) |
| `include/linux/workqueue.h` | الـ public API الكاملة — macros وflags وfunctions |
| `kernel/workqueue.c` | الـ implementation الكاملة |
| `kernel/workqueue_internal.h` | internals للـ kernel نفسه |
| `Documentation/core-api/workqueue.rst` | التوثيق الرسمي |

---

### القصة: ليه الـ Workqueue موجود أصلاً؟

تخيل إنك شغال في مطعم، وجاء طلب كبير من زبون وانت بتخدم زبون تاني. مش ممكن تقاطع الزبون الحالي، بس كمان مش ممكن تتجاهل الطلب الجديد. الحل؟ **تكتب الطلب في ورقة** وترميه في صندوق، وفي وقت مناسب واحد تاني يشيل الصندوق ويخدم الطلب.

ده بالظبط اللي بيحصل في الـ kernel.

**المشكلة الأصلية:** كتير من الأحداث في الـ kernel بتحصل في سياق ما ينفعش فيه تشتغل لفترة طويلة — زي **interrupt handlers** اللي لازم تخلص بسرعة جداً عشان الـ CPU يرجع لشغله. بس أحياناً الـ interrupt handler محتاج يعمل حاجة تاخد وقت زي: يكتب على disk، يبعت network packet، أو يخصص memory.

الحل هو إنك **تأجل** الشغل ده لـ worker thread عادي بيشتغل في process context، وتمشي بسرعة.

---

### الـ Workqueue كأداة تأجيل — التصور البسيط

```
Interrupt Handler / أي كود kernel
        |
        |  "عندي شغل محتاج يتعمل بعدين"
        |
        v
  [ queue_work(wq, &my_work) ]
        |
        v
  +-----------------------+
  |   Workqueue Queue     |  ← صندوق المهام
  |  [work1][work2][work3]|
  +-----------------------+
        |
        v
  [ Worker Thread (kworker) ]  ← موظف بياخد من الصندوق وبينفذ
        |
        v
  my_work_handler() يشتغل في process context طبيعي
```

الـ **worker thread** هو kernel thread اسمه `kworker/N:M` بتشوفه في `ps aux`.

---

### المفاهيم الجوهرية في الملف

#### 1. الـ `work_struct` — وحدة الشغل

```c
struct work_struct {
    atomic_long_t data;   /* flags + pointer للـ pool */
    struct list_head entry; /* ربطه في قائمة الانتظار */
    work_func_t func;       /* الدالة اللي هتتنفذ */
};
```

ده الـ "ورقة الطلب". بتملاها بالدالة اللي عايزها تتنفذ.

#### 2. الـ `delayed_work` — شغل بعد تأخير

```c
struct delayed_work {
    struct work_struct work;
    struct timer_list timer; /* timer بيطلق الـ work بعد delay */
    struct workqueue_struct *wq;
    int cpu;
};
```

بدل ما الشغل يبدأ فوراً، بتقوله "اشتغل بعد X jiffies". ده مفيد جداً مثلاً لـ retry logic في drivers.

#### 3. الـ `rcu_work` — شغل بعد RCU grace period

```c
struct rcu_work {
    struct work_struct work;
    struct rcu_head rcu; /* بينتظر الـ RCU grace period خلاص */
    struct workqueue_struct *wq;
};
```

للحالات اللي محتاج فيها تضمن إن مفيش reader قديم لسه بيشوف بيانات قبل ما تشتغل.

#### 4. الـ `workqueue_struct` — الـ Queue نفسها

ده الـ "صندوق المهام" — مش معلن في الـ header كـ struct كامل (forward declaration بس)، بس بتتعامل معاه بـ pointer. بتعمله `alloc_workqueue()` وبتدمره بـ `destroy_workqueue()`.

---

### الـ Flags — بتتحكم في طبيعة الـ Queue

| الـ Flag | المعنى |
|---|---|
| `WQ_UNBOUND` | الـ workers مش مربوطين بـ CPU معين — أحسن للشغل الطويل |
| `WQ_HIGHPRI` | أولوية عالية |
| `WQ_FREEZABLE` | الـ queue بتتوقف أثناء suspend |
| `WQ_MEM_RECLAIM` | مضمون إن فيه worker حتى لو الـ memory ضيقة |
| `WQ_CPU_INTENSIVE` | بيعلم الـ scheduler إن الشغل ده بياكل CPU كتير |
| `WQ_BH` | بيشتغل في **bottom half (softirq) context** بدل process context |
| `WQ_POWER_EFFICIENT` | per-CPU بشكل افتراضي لكن بيتحول لـ unbound لو `wq_power_efficient=1` |

---

### الـ System-wide Workqueues الجاهزة

الـ kernel بيجيب معاه queues جاهزة:

| الـ Queue | الاستخدام |
|---|---|
| `system_percpu_wq` | الأكثر استخداماً — `schedule_work()` بتحط فيها |
| `system_highpri_wq` | للشغل ذو الأولوية العالية |
| `system_long_wq` | للشغل اللي بياخد وقت طويل |
| `system_unbound_wq` | مش مربوط بـ CPU |
| `system_freezable_wq` | بتتجمد أثناء suspend |
| `system_bh_wq` | softirq context |

---

### دورة حياة الـ Work Item — من التسجيل للتنفيذ

```
1. تعريف وتهيئة:
   INIT_WORK(&my_work, my_handler_fn);

2. إضافة للقائمة:
   queue_work(my_wq, &my_work);
   أو: schedule_work(&my_work);  /* على system_percpu_wq */

3. انتظار أو إلغاء:
   flush_work(&my_work);        /* انتظر لحد ما يخلص */
   cancel_work_sync(&my_work);  /* ألغه وانتظر */

4. الـ Worker Thread بينفذ:
   my_handler_fn(&my_work);
   /* جوا: container_of أو from_work() عشان توصل للـ parent struct */
```

**الـ `from_work` macro** هو الطريقة الصح للوصول للـ parent struct:
```c
static void my_handler(struct work_struct *work)
{
    struct my_device *dev = from_work(dev, work, work_field);
    /* ... */
}
```

---

### الـ work_bits — سحر الـ data field

الـ `data` field في `work_struct` مش بس pointer — بيحمل معلومات كتير متضغطة في bits واحدة:

```
MSB                                           LSB
[ pool ID (31 bits) ][ disable depth (16b) ][ OFFQ flags ][ STRUCT flags (4-5b) ]
                         عندما الـ work مش في queue

[ pwq pointer ][ flush color (4b) ][ STRUCT flags (4-5b) ]
                         عندما الـ work في queue
```

- **`WORK_STRUCT_PENDING`**: الـ work معلق وينتظر التنفيذ
- **`WORK_STRUCT_PWQ`**: الـ data بيشاور على per-pool workqueue
- **`WORK_STRUCT_INACTIVE`**: الـ work متوقف مؤقتاً

---

### الـ Affinity وإدارة الـ CPU Pods

الـ `workqueue_attrs` struct بيتحكم في **تقارب** الـ workers من الـ CPUs:

```
WQ_AFFN_CPU    → worker pool واحد لكل CPU
WQ_AFFN_SMT    → واحد لكل SMT (Hyper-Threading pair)
WQ_AFFN_CACHE  → واحد لكل LLC cache
WQ_AFFN_NUMA   → واحد لكل NUMA node
WQ_AFFN_SYSTEM → واحد للسيستم كله
```

ده مهم جداً للـ performance — عايزك تعمل الشغل قريب من الـ cache اللي فيه البيانات.

---

### قصة واقعية: Driver بيستخدم Workqueue

تخيل driver لكرت network:

1. **Interrupt** بيجي من الـ hardware: "وصلت بيانات!"
2. الـ interrupt handler **بيعلم** بس مش بيعمل processing ثقيل:
   ```c
   irqreturn_t nic_interrupt(int irq, void *dev_id)
   {
       schedule_work(&nic->rx_work); /* أحط في القائمة وأمشي */
       return IRQ_HANDLED;
   }
   ```
3. الـ `kworker` thread بياخد الشغل ويعمل الـ processing الحقيقي في process context.

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | ليه مهم |
|---|---|
| `include/linux/workqueue_types.h` | تعريف `work_struct` و`work_func_t` |
| `kernel/workqueue.c` | كل الـ implementation |
| `kernel/workqueue_internal.h` | تعريفات الـ `worker_pool` و`pool_workqueue` |
| `Documentation/core-api/workqueue.rst` | الشرح التفصيلي الرسمي |
| `include/linux/timer.h` | مستخدم في `delayed_work` |
| `include/linux/rcupdate.h` | مستخدم في `rcu_work` |
## Phase 2: شرح الـ Workqueue Framework

---

### المشكلة — ليه الـ Workqueue موجود أصلاً؟

في الكيرنل عندك كتير من الأكواد محتاجة تشتغل **خارج الـ interrupt context** — يعني في سياق عادي فيه process context، تقدر تعمل sleep، تعمل lock، تتعامل مع memory بحرية.

المشكلة إن:
- الـ **interrupt handler** (hardirq) بيشتغل بسرعة ومش مسموح ييجي sleep.
- الـ **softirq/tasklet** أحسن شوية، بس برضو مش مسموح يعمل sleep وهو shared globally مع CPUs تانية.
- المطور محتاج يقول للكيرنل: "أنا عندي شغل تقيل، عمله في الـ background في وقتك."

**مثال حقيقي:** الـ USB driver لما يستقبل packet، الـ IRQ handler بياخد البيانات بسرعة، وبعدين بيحول الـ parsing والـ protocol processing لـ workqueue — عشان مش يعطل الـ interrupt line.

المشكلة القديمة كانت إن كل driver كان بيعمل **thread خاص بيه**:
```c
// الطريقة القديمة — كل driver بيعمل kthread
kthread_run(my_work_fn, data, "my-driver-thread");
```
ده كان بيخلق عشرات الـ threads فاضية تأكل stack memory وتثقل الـ scheduler.

---

### الحل — الـ Concurrency Managed Workqueue (cmwq)

بدلاً من كل driver يعمل thread خاص، الكيرنل بيديك **worker pool** مشترك ومُدار بذكاء.

الـ cmwq بيعمل:
1. **Worker threads** (kworkers) مُشتركة بين workqueues مختلفة.
2. بيزود أو ينقص عدد الـ workers **automatically** حسب الحمل — لو worker اتعمل فيها blocking wait، الـ pool بتولّد worker جديدة عشان ما يحصلش starvation.
3. بيضمن الـ **concurrency** من غير ما يغرق الـ CPU بـ threads زيادة عن اللازم.

---

### التشبيه الحقيقي — مطبخ المطعم

تخيل مطعم فيه:

| مفهوم في المطبخ | المقابل في الكيرنل |
|---|---|
| الزبون بيطلب أكل | الـ driver بيعمل `queue_work()` |
| ورقة الطلب (order slip) | `struct work_struct` — بيحمل الـ func وبياناتها |
| الـ queue على الشباك | الـ workqueue (list of pending works) |
| النادل/المطبخ (worker) | الـ kworker thread |
| مجموعة مطابخ متخصصة | worker pool (per-CPU أو unbound) |
| مدير المطعم | الـ cmwq core — بيقرر مين يشتغل امتى |
| طلب جاهز للتسليم | work item قاعد في الـ queue |

**التعمق في التشبيه:**
- **ورقة الطلب (work_struct):** زي أوردر ورقة فيها رقم الطلب + تعليمات الطبخ. الـ `data` field فيها الـ flags + pointer للـ pool أو الـ pwq، يعني زي أوردر بيشير لأنهي فرن يتحط فيه.
- **الزبون ما بيستناش:** الـ `queue_work()` بترجع فوراً (non-blocking)، الطلب اتسجل وخلاص.
- **مطابخ per-CPU:** زي إن كل فرع مطعم (CPU) عنده طباخين خاصين بيه، الطلبات اللي بتيجي من فرع بتتطبخ فيه (cache locality).
- **المدير الذكي:** لو الطباخ وقف ينتظر (blocked on I/O)، المدير بيجيب طباخ مؤقت بدله — ده هو الـ dynamic worker scaling.
- **Ordered workqueue:** زي إن المطعم عنده قانون "كل طلب يتسلّم بالترتيب" — max_active=1.

---

### الـ Big Picture Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              Kernel Subsystems / Drivers     │
                    │   USB   NIC   Block   ACPI   crypto   ...   │
                    └──────────────┬──────────────────────────────┘
                                   │  queue_work() / schedule_work()
                                   ▼
                    ┌─────────────────────────────────────────────┐
                    │           workqueue_struct (wq)             │
                    │  (logical queue owned by driver/subsystem)   │
                    │                                             │
                    │  system_percpu_wq  system_unbound_wq  ...   │
                    └──────┬─────────────────────┬───────────────┘
                           │                     │
               ┌───────────▼──────┐   ┌──────────▼────────────┐
               │  per-CPU pool    │   │   unbound pool(s)      │
               │  (bound wq)      │   │  (NUMA-aware pods)     │
               │                  │   │                        │
               │  CPU0   CPU1 ... │   │  node0   node1  ...   │
               │  pool   pool     │   │  pool    pool          │
               └────────┬─────────┘   └──────────┬────────────┘
                        │                         │
               ┌────────▼─────────────────────────▼────────────┐
               │            worker_pool                         │
               │   kworker/0:0  kworker/0:1  kworker/u:0 ...   │
               │   (kernel threads — the actual executors)      │
               └────────────────────────────────────────────────┘
                        │
               ┌────────▼──────────────────────────────────────┐
               │   work_struct (func pointer + data + flags)   │
               │   delayed_work (work + timer)                  │
               │   rcu_work    (work + rcu_head)               │
               └───────────────────────────────────────────────┘
```

---

### الـ Core Abstractions — الهياكل الأساسية

#### `struct work_struct` — وحدة الشغل

```c
struct work_struct {
    atomic_long_t data;       /* flags + pwq pointer أو pool ID — مضغوطين في long واحد */
    struct list_head entry;   /* ربط الـ work في queue الـ pool */
    work_func_t func;         /* الدالة اللي هتتنفذ */
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};
```

**الـ `data` field — مهم جداً وفيه تفاصيل دقيقة:**

الـ `data` بيحمل معلومتين مختلفتين حسب حالة الـ work:

```
لما الـ work في queue (WORK_STRUCT_PWQ set):
MSB ────────────────────────────────────────── LSB
[ pwq pointer (high bits) ][ flush color 4b ][ flags 4-5b ]

لما الـ work مش في queue (off-queue):
MSB ────────────────────────────────────────── LSB
[ pool ID (31b) ][ disable depth (16b) ][ OFFQ flags ][ STRUCT flags ]
```

ده مثال على تقنية **pointer tagging** — الـ kernel بيستخدم الـ low bits اللي الـ alignment بيضمن إنها صفر عشان يخزن flags فيها، وبيـmask الـ high bits عشان يجيب الـ pointer.

الـ flags المهمة:
| Flag | وظيفتها |
|---|---|
| `WORK_STRUCT_PENDING` | الـ work اتحطت في queue، مش تتحط تاني |
| `WORK_STRUCT_INACTIVE` | الـ work مش active (لو max_active اتجاوز) |
| `WORK_STRUCT_PWQ` | الـ data بيشاور على pwq مش pool ID |
| `WORK_OFFQ_BH` | الـ work تشتغل في BH (softirq) context |

---

#### `struct delayed_work` — شغل بعد تأخير

```c
struct delayed_work {
    struct work_struct work;    /* الـ work الأصلي */
    struct timer_list timer;    /* timer بيستنى الـ delay ينتهي */
    struct workqueue_struct *wq;/* الـ wq اللي هيتحول له */
    int cpu;                    /* الـ CPU اللي الـ timer هيشتغل عليه */
};
```

الـ flow:
```
queue_delayed_work(wq, dwork, delay)
        │
        ▼
   arm timer (delayed_work_timer_fn)
        │
        │ (بعد delay jiffies)
        ▼
   delayed_work_timer_fn() fires
        │
        ▼
   queue_work_on(cpu, wq, &dwork->work)  ← هنا بس يدخل الـ workqueue
```

**الـ timer يشتغل في softirq context** (`TIMER_IRQSAFE`)، لكن الـ work نفسه بيشتغل في process context.

---

#### `struct rcu_work` — شغل بعد RCU grace period

```c
struct rcu_work {
    struct work_struct work;
    struct rcu_head rcu;         /* RCU callback */
    struct workqueue_struct *wq;
};
```

> **RCU (Read-Copy-Update):** هو synchronization mechanism بيضمن إن readers مايشوفوش data اتمسحت. لما تعمل `queue_rcu_work()`، الـ work مش بتشتغل غير لما كل الـ RCU readers الحاليين يخلصوا — يعني guaranteed safe to free shared data.

الـ flow:
```
queue_rcu_work()
      │
      ▼
  call_rcu(&rwork->rcu, rcu_work_rcufn)
      │
      │ (after RCU grace period)
      ▼
  rcu_work_rcufn() → queue_work(rwork->wq, &rwork->work)
```

---

#### `struct workqueue_attrs` — تحكم في سلوك الـ unbound workqueue

```c
struct workqueue_attrs {
    int nice;                      /* priority (-20 to 19) */
    cpumask_var_t cpumask;         /* CPUs المسموح بيها */
    cpumask_var_t __pod_cpumask;   /* داخلي: لتحديد الـ pod */
    bool affn_strict;              /* لو true: enforce cpumask بصرامة */
    enum wq_affn_scope affn_scope; /* granularity بتاع الـ pod */
    bool ordered;                  /* max_active = 1 */
};
```

**الـ Pod concept:** الـ kernel بيقسم الـ CPUs لـ "pods" حسب topology. لو اخترت `WQ_AFFN_NUMA`، كل NUMA node بيبقى pod واحد — الـ workers اللي في نفس الـ pod بيشاركوا worker pool واحد، وده بيحسن الـ cache locality.

```
WQ_AFFN_SYSTEM  → pod واحد للنظام كله
WQ_AFFN_NUMA    → pod لكل NUMA node
WQ_AFFN_CACHE   → pod لكل LLC (Last Level Cache)
WQ_AFFN_SMT     → pod لكل pair من الـ SMT siblings
WQ_AFFN_CPU     → pod لكل CPU منفرد (= per-CPU behavior)
```

---

### الـ WQ Flags — الـ Workqueue Flags بالتفصيل

```
enum wq_flags {
    WQ_BH            // تشتغل في softirq context — سريع جداً بس no-sleep
    WQ_UNBOUND       // مش مربوطة بـ CPU معين
    WQ_FREEZABLE     // بتتجمد وقت الـ system suspend
    WQ_MEM_RECLAIM   // بتضمن worker متاح حتى لو memory ضغطانة
    WQ_HIGHPRI       // nice = -20
    WQ_CPU_INTENSIVE // الـ scheduler يبعد ده عن الـ concurrency management
    WQ_POWER_EFFICIENT // per-CPU بس ممكن يبقى unbound بأمر من kernel param
    WQ_PERCPU        // مربوطة بـ CPU محدد
}
```

**`WQ_MEM_RECLAIM`** مهم خصوصاً في embedded: لو system في low memory، الـ kernel ممكن يمنع إنشاء threads جديدة — إلا لو الـ workqueue معمارها `WQ_MEM_RECLAIM` وعندها **rescuer thread** مخصص بيضمن الشغل يكمل حتى في أصعب الظروف.

---

### الـ System-Wide Workqueues الجاهزة

الكيرنل بيجيب workqueues جاهزة:

| الـ Workqueue | الاستخدام |
|---|---|
| `system_percpu_wq` | الافتراضي — `schedule_work()` بتستخدمه |
| `system_highpri_wq` | أعلى أولوية — `WQ_HIGHPRI` |
| `system_long_wq` | شغل بياخد وقت طويل |
| `system_unbound_wq` | مش مربوط بـ CPU |
| `system_dfl_wq` | unbound بدون concurrency management |
| `system_freezable_wq` | بيتجمد مع suspend |
| `system_bh_wq` | softirq context، بيشتغل ordered على كل CPU |

---

### علاقة الـ Structs ببعض

```
workqueue_struct (wq)
    │
    ├── per-CPU pwq (pool_workqueue) ←→ worker_pool (per-CPU)
    │       │                                │
    │       └── work_struct list             └── worker threads (kworker/N:M)
    │
    └── unbound pwq list ←→ unbound worker_pool (per-pod)
                                    │
                                    └── worker threads (kworker/u:N)

work_struct
    ├── data  → (flags | pwq ptr)  OR  (flags | pool_id | disable_depth)
    ├── entry → linked in pool's worklist
    └── func  → the actual callback

delayed_work
    ├── work_struct
    └── timer_list → fires → queues work_struct into wq

rcu_work
    ├── work_struct
    └── rcu_head → grace period → queues work_struct into wq
```

---

### الـ Workqueue بيملك إيه ومش بيملك إيه؟

| الـ Workqueue يملك | الـ Driver (المستخدم) يوفر |
|---|---|
| Worker threads (kworkers) | تعريف الـ `work_struct` وتهيئتها |
| Concurrency management | الـ callback function (func) |
| CPU affinity وإدارة الـ pools | قرار امتى تـ`queue_work()` |
| Flushing وضمان اكتمال الشغل | الـ data اللي الـ work بيشتغل عليها |
| الـ rescuer thread لـ MEM_RECLAIM | ربط الـ work بـ struct أكبر بـ `container_of` |
| الـ freezing أثناء suspend | اختيار الـ wq المناسب (per-CPU / unbound / BH) |

---

### مثال كامل من الواقع — شبيه بـ USB أو Network Driver

```c
/* 1. تعريف الـ work داخل struct الـ device */
struct my_device {
    struct net_device *ndev;
    struct work_struct rx_work;     /* process received data */
    struct delayed_work stats_work; /* update stats every second */
};

/* 2. الـ callback function */
static void rx_work_fn(struct work_struct *work)
{
    /* من الـ work pointer نرجع لـ struct الـ device الكبير */
    struct my_device *dev = from_work(dev, work, rx_work);
    /* هنا مسموح نعمل sleep, mutex, alloc */
    process_rx_buffer(dev);
}

/* 3. من الـ IRQ handler — سريع جداً */
static irqreturn_t my_irq(int irq, void *data)
{
    struct my_device *dev = data;
    copy_rx_data_to_buffer(dev);       /* copy فقط */
    schedule_work(&dev->rx_work);       /* delegate الباقي */
    return IRQ_HANDLED;
}

/* 4. الـ init */
static int my_probe(...)
{
    INIT_WORK(&dev->rx_work, rx_work_fn);
    INIT_DELAYED_WORK(&dev->stats_work, stats_work_fn);
    /* queue delayed work كل ثانية */
    schedule_delayed_work(&dev->stats_work, HZ);
}
```

**الـ `from_work` macro** (اللي عرفناه في الهيدر) = `container_of` مخصوص للـ workqueue — بيرجع لـ parent struct من الـ work pointer.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags — Cheatsheet

#### `enum work_bits` — layout بتات الـ `data` field

| Bit/Field | الاسم | الوصف |
|---|---|---|
| 0 | `WORK_STRUCT_PENDING_BIT` | الـ work item في الانتظار (لم يُنفَّذ بعد) |
| 1 | `WORK_STRUCT_INACTIVE_BIT` | الـ work item غير نشط (disabled أو waiting) |
| 2 | `WORK_STRUCT_PWQ_BIT` | الـ `data` يشير لـ `pwq` (pool workqueue) |
| 3 | `WORK_STRUCT_LINKED_BIT` | الـ work item التالي مرتبط بهذا |
| 4 (debug) | `WORK_STRUCT_STATIC_BIT` | تم إنشاؤه بـ static initializer (debugobjects فقط) |
| `[COLOR_SHIFT .. COLOR_SHIFT+4)` | flush color | 4 bits للـ flush color (16 لون ممكن) |
| `[PWQ_SHIFT ..)` | pwq pointer | المؤشر للـ `pool_workqueue` عندما `PWQ_BIT=1` |
| `[OFFQ_FLAG_SHIFT]` | `WORK_OFFQ_BH_BIT` | الـ work ينتمي لـ BH workqueue |
| `[OFFQ_DISABLE_SHIFT .. +16)` | disable depth | عمق الـ disable (16 bit، حتى 65535 مستوى) |
| `[OFFQ_POOL_SHIFT .. +31)` | pool ID | آخر pool نُفِّذ فيه الـ work |

**تخطيط الـ `data` field (64-bit):**
```
MSB                                                              LSB
[ pool ID (31b) ][ disable depth (16b) ][ OFFQ flags (1b) ][ STRUCT flags (4-5b) ]
                          --- عندما PWQ_BIT = 0 (off-queue) ---

[ pwq pointer (high bits) ][ flush color (4b) ][ STRUCT flags (4-5b) ]
                          --- عندما PWQ_BIT = 1 (on-queue) ---
```

---

#### `enum work_flags` — الـ flags الجاهزة للاستخدام

| Flag | القيمة | الاستخدام |
|---|---|---|
| `WORK_STRUCT_PENDING` | bit 0 | يُقرأ بـ `work_pending()` macro |
| `WORK_STRUCT_INACTIVE` | bit 1 | يُعيَّن عند `disable_work()` |
| `WORK_STRUCT_PWQ` | bit 2 | يُعيَّن عند وضع الـ work في قائمة الانتظار |
| `WORK_STRUCT_LINKED` | bit 3 | barrier work يربط work items |
| `WORK_STRUCT_STATIC` | bit 4 أو 0 | debug فقط |

---

#### `enum wq_flags` — خصائص الـ workqueue

| Flag | الوصف | متى تستخدمه |
|---|---|---|
| `WQ_BH` | ينفَّذ في BH (softirq) context | بديل سريع لـ tasklet |
| `WQ_UNBOUND` | غير مربوط بـ CPU معين | عمل طويل أو CPU-intensive |
| `WQ_FREEZABLE` | يتجمد عند suspend | عمل يجب أن يتوقف مع النظام |
| `WQ_MEM_RECLAIM` | مخصص rescuer thread | تجنب deadlock أثناء memory pressure |
| `WQ_HIGHPRI` | أولوية عالية | latency-sensitive work |
| `WQ_CPU_INTENSIVE` | الـ work يستهلك CPU كثيراً | يمنع تأثيره على concurrency management |
| `WQ_SYSFS` | ظاهر في `/sys/bus/workqueue/` | workqueues يحتاج المستخدم يتحكم فيها |
| `WQ_POWER_EFFICIENT` | unbound إذا فُعِّل `wq_power_efficient` | توفير طاقة على حساب أداء بسيط |
| `WQ_PERCPU` | مربوط بـ CPU معين | عمل يفيد من cache locality |
| `__WQ_DESTROYING` | internal: جاري الحذف | لا تستخدم مباشرة |
| `__WQ_DRAINING` | internal: جاري التفريغ | لا تستخدم مباشرة |
| `__WQ_ORDERED` | internal: ordered (max_active=1) | يُضبَط بـ `alloc_ordered_workqueue` |
| `__WQ_LEGACY` | internal: API قديم | من `create_workqueue()` القديم |

---

#### `enum wq_affn_scope` — نطاق تقارب الـ unbound workers

| القيمة | المعنى |
|---|---|
| `WQ_AFFN_DFL` | افتراضي النظام |
| `WQ_AFFN_CPU` | pod واحد لكل CPU |
| `WQ_AFFN_SMT` | pod واحد لكل SMT (hyper-thread group) |
| `WQ_AFFN_CACHE` | pod واحد لكل LLC (Last Level Cache) |
| `WQ_AFFN_NUMA` | pod واحد لكل NUMA node |
| `WQ_AFFN_SYSTEM` | pod واحد للنظام كله |

---

#### `enum wq_consts` — حدود عددية

| الثابت | القيمة | الوصف |
|---|---|---|
| `WQ_MAX_ACTIVE` | 2048 | أقصى عدد work items نشطة في آنٍ واحد |
| `WQ_UNBOUND_MAX_ACTIVE` | 2048 | نفس الحد للـ unbound |
| `WQ_DFL_ACTIVE` | 1024 | الافتراضي (نصف الحد) |
| `WQ_DFL_MIN_ACTIVE` | 8 | حد أدنى per-node لضمان التقدم |
| `WORKER_DESC_LEN` | 32 | أقصى طول لوصف الـ worker thread |

---

#### `enum wq_misc_consts`

| الثابت | الوصف |
|---|---|
| `WORK_NR_COLORS` | 16 لون (4 bits للـ flush color) |
| `WORK_CPU_UNBOUND` | اختر CPU المحلي (= NR_CPUS) |
| `WORK_BUSY_PENDING` | bit 0 في نتيجة `work_busy()` |
| `WORK_BUSY_RUNNING` | bit 1 في نتيجة `work_busy()` |

---

### الـ Structs المهمة

---

#### 1. `struct work_struct` (في `workqueue_types.h`)

**الغرض:** الوحدة الأساسية للعمل — يمثل مهمة واحدة تنفَّذ بشكل asynchronous.

```c
struct work_struct {
    atomic_long_t data;        /* packed: flags + pwq pointer أو pool ID */
    struct list_head entry;    /* ربط في قائمة انتظار الـ pool */
    work_func_t func;          /* الدالة التي ستُنفَّذ */
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;  /* للـ deadlock detection */
#endif
};
```

| Field | النوع | الوصف |
|---|---|---|
| `data` | `atomic_long_t` | يحمل الـ flags + مؤشر الـ pwq أو pool ID — القراءة/الكتابة atomic |
| `entry` | `list_head` | نقطة الربط في قائمة الـ worker pool أو الـ pwq |
| `func` | `work_func_t` | الـ callback: `void f(struct work_struct *)` |
| `lockdep_map` | `lockdep_map` | debug فقط — يتتبع من يُشغِّل الـ work |

**علاقته بالآخرين:**
- مُضمَّن داخل `delayed_work.work` و `rcu_work.work` و `execute_work.work`
- مؤشر `data` يشير لـ `pool_workqueue` (pwq) عندما يكون in-queue
- `entry` يربطه في `worker_pool->worklist` أو `pool_workqueue->inactive_works`

---

#### 2. `struct delayed_work`

**الغرض:** work item يُنفَّذ بعد تأخير زمني محدد بـ jiffies.

```c
struct delayed_work {
    struct work_struct work;       /* الـ work الأصلي (يجب أن يكون أول field) */
    struct timer_list timer;       /* timer يُطلق الـ work بعد الـ delay */
    struct workqueue_struct *wq;   /* الـ workqueue المستهدف */
    int cpu;                       /* CPU الذي سيُنفَّذ عليه */
};
```

| Field | الوصف |
|---|---|
| `work` | الـ work الفعلي — يُوضع في الـ workqueue عند انتهاء الـ timer |
| `timer` | `TIMER_IRQSAFE` timer — يستدعي `delayed_work_timer_fn()` |
| `wq` | يُحفظ هنا لأن الـ timer callback يحتاجه لاحقاً |
| `cpu` | الـ CPU المستهدف — قد يكون `WORK_CPU_UNBOUND` |

**تحويل:** `to_delayed_work(work)` ← `container_of(work, struct delayed_work, work)`

---

#### 3. `struct rcu_work`

**الغرض:** work item يُنفَّذ بعد انتهاء الـ RCU grace period الحالية.

```c
struct rcu_work {
    struct work_struct work;       /* الـ work الفعلي */
    struct rcu_head rcu;           /* يُسجَّل في RCU callback chain */
    struct workqueue_struct *wq;   /* الـ workqueue المستهدف */
};
```

| Field | الوصف |
|---|---|
| `rcu` | عند `queue_rcu_work()` يُسجَّل `rcu_head` في RCU — عند انتهاء grace period يُوضع `work` في الـ wq |
| `wq` | محفوظ لأن الـ RCU callback يحتاجه |

---

#### 4. `struct workqueue_attrs`

**الغرض:** يصف خصائص الـ unbound workqueue — يُستخدم في الإنشاء أو التعديل بـ `apply_workqueue_attrs()`.

```c
struct workqueue_attrs {
    int nice;                       /* nice level للـ worker threads */
    cpumask_var_t cpumask;          /* CPUs المسموح بتنفيذ الـ work عليها */
    cpumask_var_t __pod_cpumask;    /* (internal) CPUs الـ pod الحالي */
    bool affn_strict;               /* صارم = workers لا يخرجون من cpumask */
    enum wq_affn_scope affn_scope;  /* نطاق تجميع CPUs في pods */
    bool ordered;                   /* work item واحد في الوقت نفسه */
};
```

| Field | الوصف |
|---|---|
| `nice` | أولوية الـ kernel threads (من -20 إلى +19) |
| `cpumask` | mask الـ CPUs المسموحة — صارمة إذا `affn_strict=true` |
| `__pod_cpumask` | subset من `cpumask` — للـ pod الحالي (internal) |
| `affn_strict` | إذا `false`: الـ scheduler يقدر ينقل الـ worker خارج الـ pod |
| `affn_scope` | يحدد كيف تُجمَّع الـ CPUs في pods (NUMA / LLC / SMT ...) |
| `ordered` | يجعل الـ workqueue sequential |

---

#### 5. `struct execute_work`

```c
struct execute_work {
    struct work_struct work;
};
```

**الغرض:** wrapper بسيط يُستخدم مع `execute_in_process_context()` — ينفِّذ دالة في process context إذا كان المستدعي أصلاً في process context، وإلا يضعها في workqueue.

---

### مخطط العلاقات بين الـ Structs

```
                    ┌──────────────────────┐
                    │   workqueue_struct   │
                    │  (wq - the queue)    │
                    └──────────┬───────────┘
                               │ 1:N (per CPU أو per pod)
                               ▼
                    ┌──────────────────────┐
                    │   pool_workqueue     │  (pwq - bridge)
                    │   (internal struct)  │
                    └──────────┬───────────┘
                               │ N:1
                               ▼
                    ┌──────────────────────┐
                    │    worker_pool       │  (internal struct)
                    │  (threads + queue)   │
                    └──────────┬───────────┘
                               │ 1:N
                               ▼
                    ┌──────────────────────┐
                    │     work_struct      │◄──── func pointer ────► callback()
                    │  data | entry | func │
                    └──────────────────────┘
                           ▲        ▲
              ─────────────┘        └─────────────────
             │                                        │
  ┌──────────────────┐                    ┌───────────────────┐
  │  delayed_work    │                    │    rcu_work       │
  │  work (embedded) │                    │  work (embedded)  │
  │  timer_list      │                    │  rcu_head         │
  │  *wq             │                    │  *wq              │
  │  cpu             │                    └───────────────────┘
  └──────────────────┘

  ┌──────────────────────────────────────────┐
  │           workqueue_attrs                │
  │  nice | cpumask | affn_scope | ordered   │
  └────────────────────────┬─────────────────┘
                           │ يُطبَّق على
                           ▼
                  workqueue_struct (unbound فقط)
```

---

### مخطط دورة الحياة

#### `work_struct` — Creation → Execution → Teardown

```
[1] DECLARE_WORK(w, fn)          ← static init في compile time
         أو
    INIT_WORK(&w, fn)             ← dynamic init في runtime
         │
         │  data = WORK_DATA_INIT (no pool, not pending)
         │  entry = empty list
         │  func = fn
         ▼
[2] queue_work(wq, &w)
         │
         ├─ اختبر PENDING bit (atomic) → إذا كان set: return false (already queued)
         ├─ set PENDING bit
         ├─ اختر worker_pool المناسب
         ├─ set PWQ bit + ربط data بالـ pwq
         ├─ أضف w->entry لـ pool->worklist
         └─ wake_up worker thread
         │
         ▼
[3] worker thread ينفِّذ
         │
         ├─ يزيل w من worklist
         ├─ clear PWQ bit (data يصبح off-queue)
         ├─ يستدعي w->func(&w)
         └─ الـ callback ينتهي
         │
         ▼
[4] الـ work جاهز للإعادة (PENDING cleared)
         │
         ├─ flush_work(&w)   ← ينتظر حتى ينتهي التنفيذ
         ├─ cancel_work(&w)  ← يلغي إذا كان pending
         └─ لا teardown صريح للـ work_struct (عكس timer مثلاً)
            إلا في حالة ONSTACK → destroy_work_on_stack()
```

#### `delayed_work` — مع Timer

```
[1] INIT_DELAYED_WORK(&dw, fn)
         │
         ├─ INIT_WORK(&dw.work, fn)
         └─ timer_init(&dw.timer, delayed_work_timer_fn, TIMER_IRQSAFE)
         │
         ▼
[2] queue_delayed_work(wq, &dw, delay_jiffies)
         │
         ├─ احفظ wq و cpu في dw.wq / dw.cpu
         └─ mod_timer(&dw.timer, jiffies + delay)
         │
         ▼
[3] Timer يطلق (في softirq context)
         │
         └─ delayed_work_timer_fn() → queue_work_on(dw.cpu, dw.wq, &dw.work)
         │
         ▼
[4] نفس مسار work_struct العادي من هنا
```

#### `rcu_work` — مع RCU Grace Period

```
[1] INIT_RCU_WORK(&rw, fn)
         │
         └─ INIT_WORK(&rw.work, fn)
         │
         ▼
[2] queue_rcu_work(wq, &rw)
         │
         ├─ احفظ wq في rw.wq
         └─ call_rcu(&rw.rcu, rcu_work_rcufn)
         │
         ▼
[3] RCU grace period تنتهي
         │
         └─ rcu_work_rcufn() → queue_work(rw.wq, &rw.work)
         │
         ▼
[4] نفس مسار work_struct العادي
```

#### `workqueue_struct` — Creation → Destruction

```
[1] alloc_workqueue("name", WQ_UNBOUND, 0)
         │
         ├─ alloc workqueue_struct
         ├─ alloc pool_workqueues (per CPU أو per NUMA node)
         ├─ ربط كل pwq بـ worker_pool مناسب
         ├─ إنشاء rescuer thread (إذا WQ_MEM_RECLAIM)
         └─ workqueue_sysfs_register() (إذا WQ_SYSFS)
         │
         ▼
[2] الاستخدام: queue_work / flush_workqueue / apply_workqueue_attrs
         │
         ▼
[3] destroy_workqueue(wq)
         │
         ├─ set __WQ_DRAINING
         ├─ drain_workqueue() ← انتظر انتهاء كل الـ work items
         ├─ set __WQ_DESTROYING
         ├─ إلغاء تسجيل من sysfs
         ├─ تحرير كل pool_workqueues
         └─ تحرير workqueue_struct
```

---

### مخطط Call Flow

#### `queue_work(wq, work)` ← الحالة الأساسية

```
queue_work(wq, work)
  └─► queue_work_on(WORK_CPU_UNBOUND, wq, work)
        │
        ├─ local_irq_save()               ← منع interrupts
        ├─ اختبار WORK_STRUCT_PENDING      ← atomic test_and_set_bit
        │    └─ إذا كان set → return false (already queued)
        │
        ├─ اختر pwq (pool_workqueue)
        │    ├─ per-CPU wq: pwq[this_cpu]
        │    └─ unbound wq: pwq[this_node]
        │
        ├─ spin_lock(pool->lock)
        ├─ إذا max_active مكتمل:
        │    └─ أضف لـ pwq->inactive_works (set INACTIVE bit)
        │  وإلا:
        │    └─ أضف لـ pool->worklist
        │         └─ wake_up_worker(pool)
        ├─ spin_unlock(pool->lock)
        └─ local_irq_restore()
        └─ return true
```

#### `flush_work(work)` ← انتظار انتهاء التنفيذ

```
flush_work(work)
  │
  ├─ أنشئ barrier work item (WORK_STRUCT_LINKED)
  ├─ أضف الـ barrier بعد work المستهدف في نفس الـ pool
  │
  └─ wait_event()  ← ينام حتى يُنفَّذ الـ barrier
       │
       (worker thread ينفِّذ work ثم ينفِّذ barrier)
       │
       └─ barrier callback → wake_up() المنتظرين
```

#### `cancel_work_sync(work)` ← إلغاء + انتظار

```
cancel_work_sync(work)
  │
  ├─ محاولة cancel قبل التنفيذ:
  │    └─ atomic: clear PENDING bit
  │         ├─ نجح → work لم يُنفَّذ، أزله من القائمة
  │         └─ فشل → work يُنفَّذ الآن
  │
  └─ إذا فشل الإلغاء:
       └─ flush_work(work)  ← انتظر انتهاء التنفيذ الحالي
```

#### `apply_workqueue_attrs(wq, attrs)` ← تغيير خصائص unbound wq

```
apply_workqueue_attrs(wq, attrs)
  │
  ├─ alloc_workqueue_attrs() + copy attrs
  ├─ احسب pods المناسبة بناءً على affn_scope
  ├─ لكل pod:
  │    ├─ ابحث عن worker_pool بنفس الـ attrs
  │    │    └─ إذا ما وُجد: أنشئ pool جديد + worker threads
  │    └─ أنشئ pwq جديد → ربطه بالـ pool
  │
  ├─ mutex_lock(wq->mutex)
  ├─ استبدل الـ pwqs القديمة بالجديدة
  ├─ mutex_unlock(wq->mutex)
  └─ حرر الـ pwqs القديمة (بعد RCU grace period)
```

---

### استراتيجية الـ Locking

#### الأقفال المستخدمة في workqueue subsystem

| القفل | النوع | يحمي |
|---|---|---|
| `worker_pool->lock` | `spinlock_t` | `worklist`، `inactive_works`، إحصائيات الـ pool، حالة الـ workers |
| `wq->mutex` | `mutex` | تغيير الـ pwqs، تطبيق attrs، flushing |
| `wq_pool_mutex` (global) | `mutex` | قائمة كل الـ pools وكل الـ workqueues |
| `wq_pool_attach_mutex` (global) | `mutex` | إضافة/إزالة workers من pools |
| `work->data` (atomic) | `atomic_long_t` | flags الـ work item (PENDING، INACTIVE، ...) |

#### ترتيب الأقفال (Lock Ordering)

```
wq_pool_mutex
    └─► wq->mutex
            └─► worker_pool->lock
                    └─► (لا قفل إضافي)
```

> **قاعدة:** لا يجوز الإمساك بـ `worker_pool->lock` ثم طلب `wq->mutex` — هذا يسبب deadlock.

#### الـ PENDING bit كـ lock خفيف

الـ `WORK_STRUCT_PENDING_BIT` في `work->data` يُستخدم كـ mutex ضمني:
- `test_and_set_bit(PENDING)` ← أول من يضبطه يملك الـ work
- `clear_bit(PENDING)` ← يُفرَّج عنه بعد الإدراج في القائمة أو بعد التنفيذ
- هذا يمنع إدراج نفس الـ work item مرتين في نفس الوقت

#### الـ BH workqueues والـ locking

الـ `WQ_BH` workqueues تعمل في softirq context:
- لا تستخدم `spinlock_t` عادي — تستخدم `local_bh_disable()` للحماية
- لا يمكن الانتظار (sleep) داخل الـ BH work callback
- الـ `__WQ_BH_ALLOWS` يقيِّد الـ flags المسموحة: `WQ_BH | WQ_HIGHPRI | WQ_PERCPU` فقط

#### الـ RCU وعلاقته بالـ workqueue

- قائمة الـ pwqs في `workqueue_struct` محمية بـ RCU للقراءة
- تحديث الـ pwqs يحتاج `wq->mutex` + RCU grace period قبل تحرير القديمة
- `rcu_work` يستغل هذه البنية: يسجِّل في RCU chain، وعند انتهاء الـ grace period يُوضع الـ work في الـ wq
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Initialization & Allocation

| Function / Macro | الغرض |
|---|---|
| `INIT_WORK(work, func)` | تهيئة `work_struct` |
| `INIT_WORK_ONSTACK(work, func)` | تهيئة على الـ stack مع debugobjects |
| `INIT_DELAYED_WORK(work, func)` | تهيئة `delayed_work` |
| `INIT_DEFERRABLE_WORK(work, func)` | `delayed_work` مع `TIMER_DEFERRABLE` |
| `INIT_RCU_WORK(work, func)` | تهيئة `rcu_work` |
| `DECLARE_WORK(n, f)` | تعريف static `work_struct` |
| `DECLARE_DELAYED_WORK(n, f)` | تعريف static `delayed_work` |
| `alloc_workqueue(fmt, flags, max_active, ...)` | إنشاء workqueue جديد |
| `alloc_ordered_workqueue(fmt, flags, ...)` | إنشاء ordered workqueue |
| `alloc_workqueue_attrs()` | تخصيص `workqueue_attrs` |
| `destroy_workqueue(wq)` | تدمير workqueue |
| `free_workqueue_attrs(attrs)` | تحرير الـ attrs |

#### Queuing

| Function / Macro | الغرض |
|---|---|
| `queue_work(wq, work)` | إضافة work على local CPU |
| `queue_work_on(cpu, wq, work)` | إضافة work على CPU محدد |
| `queue_work_node(node, wq, work)` | إضافة work على NUMA node محدد |
| `queue_delayed_work(wq, dwork, delay)` | إضافة delayed work |
| `queue_delayed_work_on(cpu, wq, dwork, delay)` | delayed work على CPU محدد |
| `mod_delayed_work(wq, dwork, delay)` | تعديل delay أو إضافة delayed work |
| `mod_delayed_work_on(cpu, wq, dwork, delay)` | نفس السابق مع تحديد CPU |
| `queue_rcu_work(wq, rwork)` | إضافة work بعد RCU grace period |
| `schedule_work(work)` | queue على `system_percpu_wq` |
| `schedule_work_on(cpu, work)` | queue على CPU محدد من `system_percpu_wq` |
| `schedule_delayed_work(dwork, delay)` | delayed queue على global wq |
| `schedule_delayed_work_on(cpu, dwork, delay)` | delayed queue على CPU محدد |

#### Flush & Cancel

| Function / Macro | الغرض |
|---|---|
| `flush_work(work)` | انتظر اكتمال تنفيذ work |
| `flush_delayed_work(dwork)` | queue فوراً ثم انتظر |
| `flush_rcu_work(rwork)` | انتظر RCU grace period والتنفيذ |
| `flush_workqueue(wq)` | انتظر اكتمال كل الـ works الحالية |
| `drain_workqueue(wq)` | drain كامل حتى يصبح فارغاً |
| `cancel_work(work)` | إلغاء بدون انتظار |
| `cancel_work_sync(work)` | إلغاء مع انتظار التنفيذ الحالي |
| `cancel_delayed_work(dwork)` | إلغاء delayed work |
| `cancel_delayed_work_sync(dwork)` | إلغاء مع انتظار |

#### Disable / Enable

| Function | الغرض |
|---|---|
| `disable_work(work)` | منع الـ queuing مع إلغاء pending |
| `disable_work_sync(work)` | منع + انتظار التنفيذ الحالي |
| `enable_work(work)` | إعادة تفعيل الـ work |
| `disable_delayed_work(dwork)` | نفس disable_work لكن للـ delayed |
| `disable_delayed_work_sync(dwork)` | نفس disable_work_sync |
| `enable_delayed_work(dwork)` | إعادة تفعيل delayed work |
| `enable_and_queue_work(wq, work)` | enable + queue بعملية واحدة |

#### Introspection & Tuning

| Function | الغرض |
|---|---|
| `work_pending(work)` | هل الـ work في الانتظار؟ |
| `delayed_work_pending(w)` | هل الـ delayed_work في الانتظار؟ |
| `work_busy(work)` | حالة الـ work (pending / running) |
| `current_work()` | الـ work_struct الحالي للـ worker |
| `current_is_workqueue_rescuer()` | هل الـ thread الحالي rescuer؟ |
| `workqueue_congested(cpu, wq)` | هل الـ wq مزدحم على هذا CPU؟ |
| `workqueue_set_max_active(wq, n)` | ضبط max_active |
| `workqueue_set_min_active(wq, n)` | ضبط min_active |
| `set_worker_desc(fmt, ...)` | وصف الـ worker الحالي في ps/top |
| `apply_workqueue_attrs(wq, attrs)` | تطبيق attrs جديدة على unbound wq |

#### CPU Hotplug & Init

| Function | الغرض |
|---|---|
| `workqueue_init_early()` | تهيئة مبكرة (قبل الـ SMP) |
| `workqueue_init()` | تهيئة كاملة بعد الـ SMP |
| `workqueue_init_topology()` | تحديث topology بعد boot |
| `workqueue_prepare_cpu(cpu)` | تحضير CPU قبل online |
| `workqueue_online_cpu(cpu)` | إضافة CPU إلى الـ pools |
| `workqueue_offline_cpu(cpu)` | إزالة CPU من الـ pools |

---

### المجموعة 1: Initialization — تهيئة الـ Work Items

هذه المجموعة تغطي كل الـ macros والـ helpers اللي بتحضّر الـ `work_struct` و`delayed_work` و`rcu_work` قبل الـ queuing. التهيئة الصحيحة ضرورية لأن الـ `data` field بيحمل flags وpointer للـ pwq في نفس الوقت، ولو اتبعت بشكل غلط ممكن يحصل corruption.

---

#### `INIT_WORK(_work, _func)`

```c
#define INIT_WORK(_work, _func) \
    __INIT_WORK((_work), (_func), 0)
```

**ما بيعمله:** بيهيئ `work_struct` للاستخدام الديناميكي (heap أو global). بيصفّر الـ `data` مع ضبط `WORK_STRUCT_NO_POOL`، بيعمل `INIT_LIST_HEAD` للـ `entry`، وبيسجّل lockdep_map باستخدام static key محلية.

| Parameter | الوصف |
|---|---|
| `_work` | مؤشر للـ `work_struct` المراد تهيئته |
| `_func` | الـ callback بالشكل `void func(struct work_struct *)` |

**Return:** لا يرجع قيمة.

**Key Details:**
- الـ `_onstack = 0` يعني debugobjects مش هيعامله كـ stack object.
- كل مرة بتستخدم `INIT_WORK` بتتولد static `lock_class_key` جديدة — مهم لـ lockdep علشان يفرق بين أنواع مختلفة من الـ works.
- آمن استدعاؤه في أي context (process, IRQ) طالما الـ work مش على قائمة.

---

#### `INIT_WORK_ONSTACK(_work, _func)`

```c
#define INIT_WORK_ONSTACK(_work, _func) \
    __INIT_WORK((_work), (_func), 1)
```

**ما بيعمله:** مثل `INIT_WORK` لكن بيخبر debugobjects إن الـ work على الـ stack. لازم يتبعه باستدعاء `destroy_work_on_stack()` قبل ما الـ stack frame ينتهي.

**Key Details:**
- بيستخدمه كود زي `flush_work` اللي محتاج ينتظر work مؤقت.
- لو `CONFIG_DEBUG_OBJECTS_WORK` مفعّل، `__init_work` بتسجّل الـ object مع debugobjects tracker.

---

#### `INIT_DELAYED_WORK(_work, _func)`

```c
#define INIT_DELAYED_WORK(_work, _func) \
    __INIT_DELAYED_WORK(_work, _func, 0)
```

**ما بيعمله:** بيهيئ `delayed_work` — بيعمل `INIT_WORK` على الـ `work` الداخلي، وبيهيئ الـ `timer_list` باستخدام `delayed_work_timer_fn` كـ callback مع `TIMER_IRQSAFE`. الـ timer هو اللي بيعمل queue للـ work بعد انتهاء الـ delay.

**Key Details:**
- الـ `TIMER_IRQSAFE` ضروري لأن الـ timer callback ممكن يتنفذ من IRQ context وبيحتاج يضيف work للـ queue.
- `INIT_DEFERRABLE_WORK` هو نفسه لكن بيضيف `TIMER_DEFERRABLE` — يعني الـ timer مش هيوقظ CPU نايم.

---

#### `INIT_RCU_WORK(_work, _func)`

```c
#define INIT_RCU_WORK(_work, _func) \
    INIT_WORK(&(_work)->work, (_func))
```

**ما بيعمله:** بيهيئ `rcu_work` — بس بيهيئ الـ `work_struct` الداخلي. الـ `rcu_head` مش محتاج تهيئة صريحة. لما تعمل `queue_rcu_work()`، الـ subsystem هو اللي بيسجّل الـ RCU callback.

---

#### `destroy_work_on_stack(work)` / `destroy_delayed_work_on_stack(dwork)`

```c
extern void destroy_work_on_stack(struct work_struct *work);
extern void destroy_delayed_work_on_stack(struct delayed_work *work);
```

**ما بتعمله:** بتخبر debugobjects إن الـ stack object اتنهى منه قبل ما الـ frame يتحرر. بدونها، debugobjects هيشتكي إن object على stack اتحرر من الـ tracking list.

**Key Details:** No-ops إذا `CONFIG_DEBUG_OBJECTS_WORK` مش مفعّل.

---

### المجموعة 2: Workqueue Creation & Destruction

هذه المجموعة مسؤولة عن إنشاء الـ `workqueue_struct` وضبط خصائصه وتدميره. فهم الـ flags الصحيح هنا مهم جداً لأداء النظام.

---

#### `alloc_workqueue(fmt, flags, max_active, ...)`

```c
__printf(1, 4) struct workqueue_struct *
alloc_workqueue_noprof(const char *fmt, unsigned int flags, int max_active, ...);
#define alloc_workqueue(...) alloc_hooks(alloc_workqueue_noprof(__VA_ARGS__))
```

**ما بيعمله:** بيخصص ويسجّل workqueue جديد. بيحدد نوع الـ workers (per-cpu أو unbound)، الـ priority، والـ concurrency limits. الـ `alloc_hooks` wrapper موجود لـ memory allocation tagging (CONFIG_MEM_ALLOC_PROFILING).

| Parameter | الوصف |
|---|---|
| `fmt` | printf-style format string لاسم الـ wq (يظهر في `/proc/`) |
| `flags` | مجموعة من `WQ_*` flags (موضّحة أدناه) |
| `max_active` | أقصى عدد works تشتغل بالتوازي (0 = default = `WQ_DFL_ACTIVE`) |
| `...` | args للـ format string |

**الـ WQ_* Flags المهمة:**

| Flag | التأثير |
|---|---|
| `WQ_BH` | التنفيذ في softirq context (BH workqueue) |
| `WQ_UNBOUND` | Workers مش مربوطين بـ CPU — scheduler حر |
| `WQ_FREEZABLE` | يتوقف أثناء system suspend/freeze |
| `WQ_MEM_RECLAIM` | بيضمن rescuer thread لتجنب deadlock أثناء memory pressure |
| `WQ_HIGHPRI` | Workers بـ nice = -20 |
| `WQ_CPU_INTENSIVE` | بيخبر الـ concurrency manager إن الـ work بياخد وقت طويل |
| `WQ_PERCPU` | مربوط بـ CPU محدد (bound) |
| `WQ_POWER_EFFICIENT` | يتحول unbound لو `wq_power_efficient=1` |
| `WQ_SYSFS` | يظهر في sysfs تحت `/sys/bus/workqueue/` |

**Return:** pointer للـ `workqueue_struct` اللي اتخصص، أو `NULL` عند الفشل.

**Key Details:**
- لـ per-cpu wq: الـ `max_active` بيُطبّق per-CPU.
- لـ unbound wq: الـ `max_active` system-wide لكن بيتوزع على NUMA nodes، ومضمون إن كل node عنده على الأقل `min_active = min(max_active, WQ_DFL_MIN_ACTIVE)`.
- `WQ_MEM_RECLAIM` ضروري لأي wq ممكن يتاستخدم في memory reclaim path علشان يتجنب deadlock لما كل الـ workers busy.

**Pseudocode:**
```
alloc_workqueue_noprof(fmt, flags, max_active):
    wq = kzalloc(workqueue_struct)
    snprintf(wq->name, fmt, ...)
    wq->flags = flags
    wq->max_active = max_active ?: WQ_DFL_ACTIVE
    if WQ_UNBOUND:
        alloc_unbound_pwq_table(wq)
    else:
        alloc_percpu_pwq_table(wq)
    if WQ_MEM_RECLAIM:
        create_rescuer_thread(wq)
    list_add(&wq_list)
    return wq
```

---

#### `alloc_ordered_workqueue(fmt, flags, args...)`

```c
#define alloc_ordered_workqueue(fmt, flags, args...) \
    alloc_workqueue(fmt, WQ_UNBOUND | __WQ_ORDERED | (flags), 1, ##args)
```

**ما بيعمله:** macro فوق `alloc_workqueue` بيضيف `WQ_UNBOUND | __WQ_ORDERED` ويضبط `max_active = 1`. الـ `__WQ_ORDERED` flag يمنع reordering حتى لو حاول أحد يغير الـ max_active لاحقاً.

**Key Details:**
- مفيد للـ drivers اللي تحتاج serialization بدون mutex صريح.
- الفرق عن single-threaded wq: ordered wq هو unbound — الـ work ممكن يتنفذ على أي CPU لكن واحد في الوقت.

---

#### `destroy_workqueue(wq)`

```c
extern void destroy_workqueue(struct workqueue_struct *wq);
```

**ما بيعمله:** بيوقف الـ workqueue ويحرر كل موارده. بيعمل drain كامل لكل الـ pending works، بيدمر الـ rescuer thread إن كان موجود، بيحرر الـ per-cpu أو unbound worker pools، وبيشيل الـ wq من الـ global list.

**Key Details:**
- **Blocking** — بينتظر حتى آخر work ينتهي.
- لازم ما تقدر تعمل `queue_work()` على الـ wq بعد الاستدعاء.
- داخلياً بيضبط `__WQ_DESTROYING` flag لمنع أي queuing جديد.
- لو حصل deadlock أثناء الـ drain (work ينتظر work تاني)، الـ rescuer thread بيتدخل لو `WQ_MEM_RECLAIM` مضبوط.

---

#### `alloc_workqueue_attrs()` / `free_workqueue_attrs(attrs)`

```c
struct workqueue_attrs *alloc_workqueue_attrs_noprof(void);
void free_workqueue_attrs(struct workqueue_attrs *attrs);
```

**ما بيعملوا:** بيخصصوا/بيحرروا `workqueue_attrs` struct اللي بيستخدم مع `apply_workqueue_attrs()`. الـ struct بيحمل `nice`، `cpumask`، `affn_scope`، و`affn_strict`.

---

#### `apply_workqueue_attrs(wq, attrs)`

```c
int apply_workqueue_attrs(struct workqueue_struct *wq,
                          const struct workqueue_attrs *attrs);
```

**ما بيعمله:** بيطبّق attrs جديدة على unbound workqueue وهو شغّال. بيعمل rehash للـ worker pools بناءً على الـ cpumask وalffinity scope الجديدة.

| Parameter | الوصف |
|---|---|
| `wq` | الـ workqueue المراد تغييره (لازم يكون `WQ_UNBOUND`) |
| `attrs` | الـ attributes الجديدة |

**Return:** `0` عند النجاح، أو كود خطأ سالب.

**Key Details:**
- فقط للـ unbound workqueues — per-cpu wq مش بيقبل.
- الاستدعاء قد يخصص worker pools جديدة ويتخلى عن القديمة.

---

### المجموعة 3: Work Queuing — إضافة العمل للتنفيذ

هذه هي الـ APIs الأساسية اللي بيستخدمها الـ driver code. الفهم الدقيق للـ memory ordering ضروري.

---

#### `queue_work_on(cpu, wq, work)`

```c
extern bool queue_work_on(int cpu, struct workqueue_struct *wq,
                          struct work_struct *work);
```

**ما بيعمله:** بيضيف `work` على الـ `wq` المرتبط بـ `cpu`. لو الـ work كان فعلاً pending (WORK_STRUCT_PENDING مضبوط)، ما بيعمل حاجة ويرجع `false`. لو نجح، بيضبط الـ pending bit atomically ويضيف الـ work على الـ list.

| Parameter | الوصف |
|---|---|
| `cpu` | رقم الـ CPU المطلوب، أو `WORK_CPU_UNBOUND` للـ local CPU |
| `wq` | الـ target workqueue |
| `work` | الـ work item |

**Return:** `true` لو أضاف الـ work (لم يكن pending)، `false` لو كان pending.

**Key Details:**
- **Memory ordering**: لو رجع `true`، كل الـ stores قبل الاستدعاء مرئية للـ CPU اللي هينفذ الـ work — بيستخدم `smp_mb()` ضمنياً خلال الـ atomic test-and-set.
- آمن من أي context بما فيه IRQ/NMI.
- للـ per-cpu wq: الـ work بيتنفذ على الـ CPU المحدد (أو أي CPU لو مات).
- للـ unbound wq: الـ `cpu` hint بس، الـ work ممكن يروح على pool تاني.

---

#### `queue_work(wq, work)`

```c
static inline bool queue_work(struct workqueue_struct *wq,
                               struct work_struct *work)
{
    return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```

**ما بيعمله:** wrapper بسيط فوق `queue_work_on` بـ `WORK_CPU_UNBOUND` — يعني بيفضّل الـ local CPU.

---

#### `queue_work_node(node, wq, work)`

```c
extern bool queue_work_node(int node, struct workqueue_struct *wq,
                             struct work_struct *work);
```

**ما بيعمله:** بيضيف الـ work على worker pool مرتبط بـ NUMA node محدد. مفيد لتحسين locality لما تعرف إن الـ work هيشتغل على data موجودة في NUMA node معين.

**Key Details:** بيستخدم مع unbound wqs فقط.

---

#### `queue_delayed_work_on(cpu, wq, dwork, delay)`

```c
extern bool queue_delayed_work_on(int cpu, struct workqueue_struct *wq,
                                   struct delayed_work *work,
                                   unsigned long delay);
```

**ما بيعمله:** لو `delay == 0`، بيعمل `queue_work_on` مباشرة. لو `delay > 0`، بيضبط الـ `dwork->wq` و`dwork->cpu` ثم بيبدأ الـ timer. لما الـ timer يفيق، `delayed_work_timer_fn` بتعمل `queue_work_on`.

| Parameter | الوصف |
|---|---|
| `cpu` | CPU للـ timer affinity وللـ queue target |
| `wq` | الـ workqueue |
| `dwork` | الـ delayed work |
| `delay` | عدد الـ jiffies للانتظار |

**Return:** `true` لو أضاف الـ work، `false` لو كان pending.

---

#### `mod_delayed_work_on(cpu, wq, dwork, delay)`

```c
extern bool mod_delayed_work_on(int cpu, struct workqueue_struct *wq,
                                 struct delayed_work *dwork,
                                 unsigned long delay);
```

**ما بيعمله:** بيعدّل الـ delay للـ delayed work اللي كان pending. لو كان الـ work pending، بيلغي الـ timer القديم ويبدأ timer جديد بالـ delay الجديد. لو الـ work مش pending، بيعمل queue جديد.

**Return:** `true` لو كان pending وعُدّل، `false` لو تمّ قueueing جديد.

**Key Details:**
- مهم: لو رجع `false`، ممكن الـ work يكون شغّال حالياً وما اتعدّل.
- بيستخدم `del_timer_sync` داخلياً لضمان إلغاء الـ timer القديم أمان.

---

#### `queue_rcu_work(wq, rwork)`

```c
extern bool queue_rcu_work(struct workqueue_struct *wq, struct rcu_work *rwork);
```

**ما بيعمله:** بيسجّل `rwork->rcu` كـ RCU callback. لما تنتهي الـ RCU grace period، الـ callback بيعمل `queue_work` على الـ `rwork->work` الداخلي.

**Key Details:**
- يُستخدم لتأخير عملية حتى بعد ما كل الـ RCU readers ينتهوا.
- الـ `rwork->wq` بيتحدد تلقائياً عند الاستدعاء.
- مثال استخدام: حذف object محمي بـ RCU بعد ما الـ readers ينتهوا.

---

#### `schedule_work(work)` / `schedule_work_on(cpu, work)`

```c
static inline bool schedule_work(struct work_struct *work) {
    return queue_work(system_percpu_wq, work);
}
static inline bool schedule_work_on(int cpu, struct work_struct *work) {
    return queue_work_on(cpu, system_percpu_wq, work);
}
```

**ما بيعملوا:** shortcuts لـ `queue_work` على الـ global `system_percpu_wq`. الأكثر استخداماً في الـ kernel code.

---

### المجموعة 4: Flush & Drain — انتظار اكتمال التنفيذ

---

#### `flush_work(work)`

```c
extern bool flush_work(struct work_struct *work);
```

**ما بيعمله:** ينتظر حتى يكتمل تنفيذ `work` إذا كان running أو pending. بيستخدم mechanism الـ "flush barriers" — بيضيف barrier work بعد الـ target work ويستنى.

**Return:** `true` لو انتظر فعلاً (كان pending أو running)، `false` لو كان idle.

**Key Details:**
- **Blocking** — لا يُستخدم من IRQ context.
- ممكن يعمل deadlock لو `work` حاول يعمل `flush_work` على نفسه.
- آمن لو الـ work على per-cpu أو unbound wq.
- بيستخدم `wq_barrier` داخلياً كـ fence.

---

#### `flush_delayed_work(dwork)`

```c
extern bool flush_delayed_work(struct delayed_work *dwork);
```

**ما بيعمله:** لو الـ timer لسه شغّال، بيلغيه ويعمل `queue_work` فوراً ثم بيعمل `flush_work` عليه. يعني بيضمن إن الـ work يتنفذ وينتهي.

---

#### `flush_rcu_work(rwork)`

```c
extern bool flush_rcu_work(struct rcu_work *rwork);
```

**ما بيعمله:** لو الـ rcu_work لسه في الـ RCU queue، بينتظر الـ grace period تنتهي ثم ينتظر الـ work ينتهي. يجمع `synchronize_rcu()` و`flush_work()` في خطوة.

---

#### `__flush_workqueue(wq)` / `flush_workqueue(wq)`

```c
extern void __flush_workqueue(struct workqueue_struct *wq);
```

**ما بيعمله:** ينتظر حتى تكتمل كل الـ work items اللي كانت في الـ queue **وقت الاستدعاء**. بيعمل هذا بضبط flush color وانتظار كل الـ pwqs ترفع الـ color الجديد.

**Key Details:**
- `flush_workqueue(wq)` هو macro فوق `__flush_workqueue` بيتحقق compile-time إذا الـ wq هو system-wide wq ويحذّر.
- `flush_scheduled_work()` deprecated — بيحذّر دائماً.
- الـ flush بيتم per-color (16 color available) — الـ works الجديدة اللي بتتضاف بعد الاستدعاء مش بتنتظر.

---

#### `drain_workqueue(wq)`

```c
extern void drain_workqueue(struct workqueue_struct *wq);
```

**ما بيعمله:** أقوى من flush — بيعمل flush loops حتى ما يبقاش أي work في الـ queue. Works اللي بتعيد queue نفسها هتنتظر هي كمان.

**Key Details:**
- **Blocking** وممكن ياخد وقت طويل.
- بيستخدمه `destroy_workqueue()` داخلياً.

---

### المجموعة 5: Cancel — إلغاء التنفيذ

---

#### `cancel_work(work)`

```c
extern bool cancel_work(struct work_struct *work);
```

**ما بيعمله:** بيحاول يشيل الـ work من الـ queue قبل ما ينفذ. لو نجح، يمسح الـ PENDING bit. لو الـ work شغّال حالياً أو مش pending، يرجع `false`.

**Return:** `true` لو أُلغي بنجاح (كان pending)، `false` غير كده.

**Key Details:**
- **Non-blocking** — مش بينتظر لو الـ work شغّال.
- لو محتاج تضمن إن الـ work مش شغّال، استخدم `cancel_work_sync`.

---

#### `cancel_work_sync(work)`

```c
extern bool cancel_work_sync(struct work_struct *work);
```

**ما بيعمله:** بيلغي الـ work لو pending، ولو كان شغّال بينتظر حتى ينتهي. بعد ما يرجع، مضمون إن الـ work مش pending ومش running.

**Return:** `true` لو كان pending عند الاستدعاء.

**Key Details:**
- **Blocking** — لا يُستخدم من IRQ context.
- آمن للاستدعاء من الـ `work->func` نفسها؟ لا — deadlock.
- لازم تستخدمه قبل `kfree` للـ struct اللي بتحمل الـ work.

```
cancel_work_sync(work):
    loop:
        if try_cancel(work):     // clear PENDING atomically
            break
        if work_is_running(work):
            wait_for_completion()  // wait current execution
    return was_pending
```

---

#### `cancel_delayed_work(dwork)` / `cancel_delayed_work_sync(dwork)`

```c
extern bool cancel_delayed_work(struct delayed_work *dwork);
extern bool cancel_delayed_work_sync(struct delayed_work *dwork);
```

**ما بيعملوا:** `cancel_delayed_work` بيلغي الـ timer ويمسح الـ PENDING — non-blocking. `cancel_delayed_work_sync` بيضمن إن الـ work مش شغّال كمان — blocking.

**Key Details:**
- `cancel_delayed_work` آمن من IRQ context.
- `cancel_delayed_work_sync` لازم من process context.

---

### المجموعة 6: Disable / Enable — تعطيل/تفعيل الـ Work

هذه API جديدة نسبياً (أُضيفت في kernel 6.x) بتوفر mechanism لمنع الـ work من الـ queuing مع tracking depth.

---

#### `disable_work(work)`

```c
extern bool disable_work(struct work_struct *work);
```

**ما بيعمله:** بيزيد الـ disable depth في الـ `data` field (bits الـ WORK_OFFQ_DISABLE). كمان بيلغي الـ pending work. لو الـ work كان pending، بيرجع `true`.

**Return:** `true` لو كان pending قبل الـ disable.

**Key Details:**
- كل `disable_work` لازم يقابله `enable_work`.
- الـ disable depth مخزونة في 16-bit field في الـ `work->data`.
- مفيد لـ suspend handlers أو خلال hardware reset.

---

#### `disable_work_sync(work)`

```c
extern bool disable_work_sync(struct work_struct *work);
```

**ما بيعمله:** مثل `disable_work` لكن بينتظر لو الـ work كان شغّال حالياً. بعد ما يرجع، مضمون إن الـ work مش pending ومش running.

---

#### `enable_work(work)`

```c
extern bool enable_work(struct work_struct *work);
```

**ما بيعمله:** بينزّل الـ disable depth بواحد. لو وصلت الـ depth لصفر (آخر `enable`)، يرجع `true`. الـ caller بيقدر يقرر إذا يعمل `queue_work` أو لا.

**Return:** `true` لو وصل الـ disable depth لصفر.

---

#### `enable_and_queue_work(wq, work)`

```c
static inline bool enable_and_queue_work(struct workqueue_struct *wq,
                                          struct work_struct *work)
{
    if (enable_work(work)) {
        queue_work(wq, work);
        return true;
    }
    return false;
}
```

**ما بيعمله:** بيجمع `enable_work` و`queue_work` في خطوة واحدة. لو الـ disable depth وصل صفر، بيعمل queue فوراً.

**Key Details:**
- الـ work دايماً بيتعمل له queue لما يصل الـ depth لصفر — حتى لو ما حصلش أي event أثناء الـ disable. لو محتاج conditional queuing، استخدم `enable_work` وقرر أنت.

---

### المجموعة 7: Introspection & Diagnostics

---

#### `work_pending(work)` / `delayed_work_pending(w)`

```c
#define work_pending(work) \
    test_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))
```

**ما بيعمله:** بيتحقق من الـ PENDING bit في الـ `work->data` باستخدام atomic `test_bit`. يرجع non-zero لو الـ work في الانتظار.

**Key Details:**
- النتيجة snapshot فقط — ممكن تتغير في أي لحظة.
- لا تستخدمه كـ synchronization primitive.

---

#### `work_busy(work)`

```c
extern unsigned int work_busy(struct work_struct *work);
```

**ما بيعمله:** بيفحص إذا الـ work pending أو running. بيرجع bitmask.

**Return:** `0` (idle)، `WORK_BUSY_PENDING` (في الـ queue)، `WORK_BUSY_RUNNING` (بيتنفذ حالياً)، أو كلاهما.

---

#### `current_work()`

```c
extern struct work_struct *current_work(void);
```

**ما بيعمله:** بيرجع الـ `work_struct` اللي الـ current worker thread بينفذه حالياً. لو الـ thread الحالي مش worker، يرجع `NULL`.

**Key Details:**
- بيستخدمه الـ work callback علشان يعرف معلومات عن نفسه.
- يُستخدم مع `from_work()` macro لاسترجاع الـ containing struct.

---

#### `from_work(var, callback_work, work_fieldname)`

```c
#define from_work(var, callback_work, work_fieldname) \
    container_of(callback_work, typeof(*var), work_fieldname)
```

**ما بيعمله:** macro مبني على `container_of` — بيرجع الـ parent struct من الـ `work_struct` pointer.

**مثال:**
```c
struct my_dev {
    int val;
    struct work_struct work;
};

void my_work_fn(struct work_struct *work)
{
    /* استخراج الـ parent struct من الـ work pointer */
    struct my_dev *dev = from_work(dev, work, work);
    dev->val = 42;
}
```

---

#### `set_worker_desc(fmt, ...)`

```c
extern __printf(1, 2) void set_worker_desc(const char *fmt, ...);
```

**ما بيعمله:** بيضبط description string للـ worker thread الحالي. الـ string بيظهر في `ps`/`top` وفي kernel traces.

**Key Details:**
- لازم يُستدعى من داخل الـ work callback نفسه.
- الـ `WORKER_DESC_LEN = 32` حرف.
- مفيد جداً لـ debugging — بدلاً من `kworker/u8:2`، ممكن يظهر اسم وظيفة.

---

#### `workqueue_congested(cpu, wq)`

```c
extern bool workqueue_congested(int cpu, struct workqueue_struct *wq);
```

**ما بيعمله:** بيتحقق إذا كان الـ wq's per-cpu list للـ CPU المحدد فيه works كتير (أكثر من الـ max_active).

**Return:** `true` لو مزدحم.

**Key Details:**
- بيستخدمه الـ networking stack وغيره للـ backpressure.
- النتيجة approximate — race-prone.

---

#### `workqueue_set_max_active(wq, max_active)` / `workqueue_set_min_active(wq, min_active)`

```c
extern void workqueue_set_max_active(struct workqueue_struct *wq, int max_active);
extern void workqueue_set_min_active(struct workqueue_struct *wq, int min_active);
```

**ما بيعملوا:** بيغيروا الـ concurrency limits في runtime. `set_max_active` آمن لأي wq. `set_min_active` فقط للـ unbound wqs.

**Key Details:**
- الزيادة في `max_active` قد توقظ works كانت inactive.
- الـ `min_active` بيضمن forward progress على كل NUMA node.

---

### المجموعة 8: CPU Hotplug & System Init

---

#### `workqueue_init_early()` / `workqueue_init()` / `workqueue_init_topology()`

```c
void __init workqueue_init_early(void);
void __init workqueue_init(void);
void __init workqueue_init_topology(void);
```

**ما بيعملوا:**

- **`workqueue_init_early()`**: بيتستدعى في بداية `start_kernel()` قبل SMP وقبل memory allocators يكتملوا. بينشئ الـ system workqueues (system_wq, system_highpri_wq, إلخ) والـ initial worker pools للـ boot CPU.

- **`workqueue_init()`**: بيتستدعى بعد SMP وbody allocators. بينشئ الـ worker threads الفعلية لكل CPU، وبيكمّل الـ unbound pools initialization.

- **`workqueue_init_topology()`**: بيتستدعى بعد ما تكتمل الـ CPU topology information. بيعيد حساب الـ CPU pods وبيحدث الـ worker pools بناءً على الـ LLC/NUMA/SMT topology الفعلية.

---

#### `workqueue_prepare_cpu(cpu)` / `workqueue_online_cpu(cpu)` / `workqueue_offline_cpu(cpu)`

```c
int workqueue_prepare_cpu(unsigned int cpu);
int workqueue_online_cpu(unsigned int cpu);
int workqueue_offline_cpu(unsigned int cpu);
```

**ما بيعملوا:** بيتستدعوا من الـ CPU hotplug callbacks:

- **`prepare_cpu`**: قبل ما الـ CPU يبدأ — بيخصص worker pool structures.
- **`online_cpu`**: بعد ما الـ CPU يبدأ — بيخلق worker threads ويوصلهم بالـ pools.
- **`offline_cpu`**: بعد ما الـ CPU يتوقف — بينقل الـ pending works للـ rescue CPU وبيدمر الـ threads.

**Key Details:**
- مسجّلين كـ `cpuhp_setup_state` handlers في `workqueue_init()`.
- الـ `WQ_MEM_RECLAIM` workqueues عندها rescuer thread بيضمن إن الـ works تتنفذ حتى لو كل الـ worker threads مشغولين أو الـ CPU مات.

---

### المجموعة 9: Freezer Integration

---

#### `freeze_workqueues_begin()` / `freeze_workqueues_busy()` / `thaw_workqueues()`

```c
#ifdef CONFIG_FREEZER
extern void freeze_workqueues_begin(void);
extern bool freeze_workqueues_busy(void);
extern void thaw_workqueues(void);
#endif
```

**ما بيعملوا:**

- **`freeze_workqueues_begin()`**: بيضبط الـ `WQ_FREEZABLE` flag effect على كل الـ freezable workqueues — بيوقف قبول works جديدة.
- **`freeze_workqueues_busy()`**: بيتحقق إذا كان لسه في works شغّالة في الـ freezable workqueues.
- **`thaw_workqueues()`**: بيرجع الـ freezable workqueues لحالتها الطبيعية بعد resume.

**Key Details:**
- بيُستخدموا في `suspend_freeze_processes()` وsystem hibernate path.
- فقط الـ workqueues اللي عندها `WQ_FREEZABLE` flag بتتأثر.

---

### المجموعة 10: Utility Helpers

---

#### `execute_in_process_context(fn, ew)`

```c
int execute_in_process_context(work_func_t fn, struct execute_work *ew);
```

**ما بيعمله:** لو الـ caller في process context (ما فيش atomic)، بينفذ `fn` مباشرة. لو في atomic context (interrupt, spinlock)، بيعمل queue للـ `fn` على `system_percpu_wq` باستخدام `execute_work` struct الجاهز.

**Return:** `0` لو نُفّذت مباشرة، `1` لو أُضيفت للـ queue.

**Key Details:**
- بيستخدمه كود زي الـ kobject cleanup اللي ممكن يتاستخدم من contexts مختلفة.

---

#### `schedule_on_each_cpu(func)`

```c
extern int schedule_on_each_cpu(work_func_t func);
```

**ما بيعمله:** بيشغّل `func` على كل CPU في النظام بالتوالي وينتظر اكتمال التنفيذ. بيعمل queue مرة على كل CPU ثم بيعمل flush.

**Return:** `0` عند النجاح، أو كود خطأ.

**Key Details:**
- **Blocking** وبياخد وقت بناءً على عدد الـ CPUs.
- بيُستخدم لعمليات زي cache flush أو per-CPU state update.

---

#### `work_on_cpu(cpu, fn, arg)` / `work_on_cpu_key(cpu, fn, arg, key)`

```c
#ifdef CONFIG_SMP
long work_on_cpu_key(int cpu, long (*fn)(void *),
                     void *arg, struct lock_class_key *key);
#define work_on_cpu(_cpu, _fn, _arg) \
    ({ static struct lock_class_key __key; \
       work_on_cpu_key(_cpu, _fn, _arg, &__key); })
#else
static inline long work_on_cpu(int cpu, long (*fn)(void *), void *arg)
{ return fn(arg); }
#endif
```

**ما بيعمله:** بينفذ `fn(arg)` على الـ CPU المحدد وينتظر النتيجة. على UP kernels، بينفذ مباشرة.

**Return:** قيمة `long` اللي رجعتها `fn`.

**Key Details:**
- بيستخدمه كود زي الـ ACPI وMCE handlers اللي تحتاج تنفذ على CPU معين.
- **Blocking** — لازم من process context.

---

#### `to_delayed_work(work)` / `to_rcu_work(work)`

```c
static inline struct delayed_work *to_delayed_work(struct work_struct *work)
{
    return container_of(work, struct delayed_work, work);
}
static inline struct rcu_work *to_rcu_work(struct work_struct *work)
{
    return container_of(work, struct rcu_work, work);
}
```

**ما بيعملوا:** type-safe cast من `work_struct*` للـ containing type. بيُستخدموا في callbacks اللي بتستقبل `work_struct*` وتحتاج تعرف الـ timer أو الـ rcu_head.

---

#### `print_worker_info(log_lvl, task)` / `show_all_workqueues()` / `show_one_workqueue(wq)`

```c
extern void print_worker_info(const char *log_lvl, struct task_struct *task);
extern void show_all_workqueues(void);
extern void show_freezable_workqueues(void);
extern void show_one_workqueue(struct workqueue_struct *wq);
extern void wq_worker_comm(char *buf, size_t size, struct task_struct *task);
```

**ما بيعملوا:** debugging/diagnostic functions:
- `print_worker_info`: بيطبع معلومات الـ worker thread في kernel log — بيُستخدم في crash handlers.
- `show_all_workqueues` / `show_one_workqueue`: بيطبعوا state الـ workqueues — بيُستدعوا من `sysrq-t`.
- `wq_worker_comm`: بيملى الـ buffer باسم الـ worker المناسب للـ `/proc/PID/comm`.

---

#### `wq_watchdog_touch(cpu)`

```c
#ifdef CONFIG_WQ_WATCHDOG
void wq_watchdog_touch(int cpu);
#endif
```

**ما بيعمله:** بيـ"يربت" على الـ workqueue watchdog للـ CPU المحدد علشان يمنع false positive stall warnings. بيُستدعى من worker threads اللي بتنفذ tasks طويلة.

---

#### `workqueue_sysfs_register(wq)`

```c
int workqueue_sysfs_register(struct workqueue_struct *wq);
```

**ما بيعمله:** بيسجّل الـ workqueue في sysfs تحت `/sys/bus/workqueue/devices/`. مطلوب للـ workqueues اللي عندها `WQ_SYSFS` flag.

**Return:** `0` عند النجاح، كود خطأ سالب عند الفشل.

---

### ملاحظات ختامية — Key Patterns

**الـ data field encoding:**
```
MSB                                                          LSB
┌──────────────────┬──────────┬──────────┬───────────────────┐
│   pool/pwq ptr   │  color   │  OFFQ    │   STRUCT flags    │
│   (when PWQ set) │  4 bits  │  flags   │   4-5 bits        │
└──────────────────┴──────────┴──────────┴───────────────────┘
```

**قاعدة اختيار الـ WQ:**

| الاستخدام | الـ WQ المناسب |
|---|---|
| أي عمل قصير من interrupt | `system_percpu_wq` / `schedule_work()` |
| عمل طويل | `system_long_wq` |
| high priority | `system_highpri_wq` |
| memory reclaim path | workqueue مع `WQ_MEM_RECLAIM` |
| serialization مطلوبة | `alloc_ordered_workqueue()` |
| softirq replacement | `system_bh_wq` مع `WQ_BH` |
| power-sensitive | `system_power_efficient_wq` |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ workqueue بيعرض معلومات تشخيصية تفصيلية تحت `/sys/kernel/debug/workqueue/`:

```bash
# اقرأ كل الـ workqueues الموجودة
ls /sys/kernel/debug/workqueue/

# اقرأ state معين لـ workqueue
cat /sys/kernel/debug/workqueue/events/pwq/2   # pool_workqueue للـ CPU 2
```

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/workqueue/` | دليل يحتوي على subdir لكل workqueue |
| كل subdir | يحتوي على `pwq/N` لكل CPU pool |

الأداة الأهم هي `show_all_workqueues()` — بتطبعها عبر:

```bash
# اطبع حالة كل الـ workqueues (مفيد جداً في deadlock investigation)
echo t > /proc/sysrq-trigger
# ثم ابحث في dmesg عن "workqueue"
dmesg | grep -A 5 "Showing busy workqueues"
```

---

#### 2. مدخلات الـ sysfs

الـ workqueues اللي اتعملت بـ `WQ_SYSFS` تظهر تحت `/sys/bus/workqueue/devices/`:

```bash
# اعرض كل الـ workqueues المرئية في sysfs
ls /sys/bus/workqueue/devices/

# مثال: workqueue اسمه "events"
ls /sys/bus/workqueue/devices/events/
# المخرجات المحتملة:
# affinity_scope  affinity_strict  cpumask  max_active  min_active  name  nice

# اقرأ الـ max_active الحالي
cat /sys/bus/workqueue/devices/events/max_active

# غيّر الـ max_active (مفيد لـ performance testing)
echo 4 > /sys/bus/workqueue/devices/events/max_active

# اقرأ الـ cpumask المسموح بيه
cat /sys/bus/workqueue/devices/events/cpumask

# اقرأ الـ affinity scope
cat /sys/bus/workqueue/devices/events/affinity_scope
# possible values: default, cpu, smt, cache, numa, system
```

| الملف | المعنى | القيم |
|-------|---------|-------|
| `max_active` | أقصى عدد work items تشتغل بالتوازي | 1 → 2048 |
| `min_active` | حد أدنى مضمون per-node | افتراضي 8 |
| `cpumask` | الـ CPUs المسموح بتشغيل الـ workers عليها | hex bitmask |
| `affinity_scope` | نطاق الـ pod affinity | cpu/smt/cache/numa/system |
| `affinity_strict` | هل الـ scheduler ملزم بالـ cpumask | 0 أو 1 |
| `nice` | priority الـ worker threads | -20 إلى 19 |

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# اعرض كل الـ tracepoints المتعلقة بالـ workqueue
grep -r workqueue /sys/kernel/tracing/available_events

# المجموعة الكاملة:
# workqueue:workqueue_queue_work
# workqueue:workqueue_activate_work
# workqueue:workqueue_execute_start
# workqueue:workqueue_execute_end

# فعّل كل أحداث الـ workqueue
echo 1 > /sys/kernel/tracing/events/workqueue/enable

# فعّل حدث واحد بس (متابعة إيمتى بيتحط الـ work)
echo 1 > /sys/kernel/tracing/events/workqueue/workqueue_queue_work/enable

# فعّل الـ tracing وشوف النتيجة
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe
```

**مثال على مخرجات `workqueue_queue_work`:**
```
kworker/u8:2-123   [002] ....   42.123456: workqueue_queue_work:
    work struct=0xffff888012345678 function=my_work_handler
    workqueue=0xffff888087654321 req_cpu=2 cpu=2
```

**تفسير الحقول:**
- `work struct` → عنوان الـ `work_struct` في الذاكرة
- `function` → الـ callback المسجّل في `work->func`
- `req_cpu` → الـ CPU اللي طلب الـ queue
- `cpu` → الـ CPU اللي هيشتغل عليه فعلياً

```bash
# filter على function معينة بس
echo 'function == my_work_handler' > \
    /sys/kernel/tracing/events/workqueue/workqueue_queue_work/filter

# تتبع الـ execute start/end لقياس latency
echo 1 > /sys/kernel/tracing/events/workqueue/workqueue_execute_start/enable
echo 1 > /sys/kernel/tracing/events/workqueue/workqueue_execute_end/enable
cat /sys/kernel/tracing/trace | grep "execute_"
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ workqueue subsystem
echo 'file kernel/workqueue.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لملف معين بيستخدم workqueue
echo 'file drivers/my_driver.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ pr_debug في الـ workqueue
echo 'module workqueue +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ dynamic debug المفعّلة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep workqueue
```

لتتبع مشاكل الـ work callbacks في الكود الخاص:

```c
/* في الـ work handler نفسه */
static void my_work_handler(struct work_struct *work)
{
    struct my_data *data = from_work(data, work, work_field);
    pr_debug("%s: executing on CPU %d, data=%p\n",
             __func__, smp_processor_id(), data);
}
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الخيار | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_OBJECTS_WORK` | يفحص double-init وuse-after-free للـ work_struct |
| `CONFIG_WQ_WATCHDOG` | يرصد الـ worker pools المتوقفة (stall detection) |
| `CONFIG_LOCKDEP` | يتتبع الـ locking order داخل الـ work callbacks |
| `CONFIG_DEBUG_WQ_FORCE_RR_CPU` | يجبر الـ round-robin CPU selection لكشف race conditions |
| `CONFIG_PROVE_LOCKING` | يثبت صحة الـ locking patterns بشكل كامل |
| `CONFIG_SCHED_DEBUG` | معلومات إضافية عن الـ kworker threads في الـ scheduler |
| `CONFIG_TRACEPOINTS` | يفعّل الـ workqueue tracepoints |
| `CONFIG_FTRACE` | البنية التحتية للـ tracing |

```bash
# تحقق إذا CONFIG_DEBUG_OBJECTS_WORK مفعّل
grep CONFIG_DEBUG_OBJECTS_WORK /boot/config-$(uname -r)

# تحقق من WQ_WATCHDOG
grep CONFIG_WQ_WATCHDOG /boot/config-$(uname -r)
```

---

#### 6. أدوات الـ Workqueue الخاصة

**الـ wq_watchdog:**
```bash
# اضبط timeout الـ watchdog (بالثواني، 0 = disable)
echo 30 > /sys/module/workqueue/parameters/watchdog_thresh

# راقب الـ stall messages في dmesg
dmesg -w | grep "BUG: workqueue lockup"
```

**الـ show_all_workqueues:**
```bash
# اطبع حالة كل الـ workqueues مع pending counts
echo t > /proc/sysrq-trigger
dmesg | tail -100 | grep -E "(workqueue|kworker)"
```

**أوامر مفيدة لمراقبة الـ kworker threads:**
```bash
# اعرض كل الـ kworker threads الشغّالة
ps aux | grep kworker

# اعرض الـ kworker مع الـ CPU المرتبط بيه
ps -eo pid,psr,comm | grep kworker

# راقب CPU usage للـ kworker threads
top -b -n 1 | grep kworker | sort -k9 -rn | head -20

# اعرض الـ worker description (لو اتسطّب بـ set_worker_desc)
cat /proc/$(pgrep -f "kworker/u")/status
```

---

#### 7. رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|---------|-------|
| `BUG: workqueue lockup - pool cpus=X node=Y flags=0xZ nice=N stuck for Ns!` | worker pool متوقف — workers مش بتخلص شغلها | فتش عن blocking ops أو deadlock في الـ work handler |
| `WARNING: CPU: X PID: Y at kernel/workqueue.c:NNN` | violation في invariant داخلي | راجع الـ stack trace ودور على double-queue أو use-after-free |
| `workqueue: work is queued to the wrong workqueue` | `work_struct` اتحط في wq مختلف عن اللي اشتغل منه | تأكد من استخدام نفس الـ wq في كل مكان |
| `ODEBUG: destroy active object type: work_struct` | `destroy_work_on_stack` اتشال وفي work لسه شغّال | ضيف `cancel_work_sync()` قبل الـ destroy |
| `BUG: sleeping function called from invalid context` | work handler بيعمل sleep في BH workqueue | استخدم `WQ_UNBOUND` أو `system_wq` بدل `system_bh_wq` |
| `kernel: INFO: task kworker:blocked for more than Xs` | work handler بياخد وقت طويل جداً | قسّم الـ handler أو انقل الشغل الثقيل لـ thread آخر |
| `workqueue: flush_workqueue of wq, consider using local wq` | محاولة flush system-wide workqueue | أنشئ workqueue خاص بدل استخدام system_wq |
| `DEBUG_OBJECTS: work_struct is being used after being freed` | use-after-free في work_struct | ضمّن الـ work_struct في الـ container struct وتأكد من lifetime management |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* تحقق إن الـ work مش pending قبل إنك تعمل destroy */
static void my_cleanup(struct my_data *data)
{
    /* الصح: */
    cancel_work_sync(&data->work);

    /* للـ debugging: تأكد إن الـ work اتلغى فعلاً */
    WARN_ON(work_pending(&data->work));
}

/* تحقق من الـ context داخل الـ work handler */
static void my_work_handler(struct work_struct *work)
{
    /* تأكد إننا فعلاً في workqueue context */
    WARN_ON(!current_work());

    /* تأكد مش في interrupt context */
    WARN_ON(in_interrupt());
}

/* راقب الـ congestion */
static void my_queue_work(struct my_data *data)
{
    if (workqueue_congested(WORK_CPU_UNBOUND, my_wq)) {
        pr_warn_ratelimited("workqueue congested!\n");
        dump_stack();   /* اعرف مين المتسبب */
    }
    queue_work(my_wq, &data->work);
}

/* تحقق من حالة الـ work قبل re-queue */
static void my_requeue(struct work_struct *work)
{
    unsigned int busy = work_busy(work);
    WARN_ON_ONCE(busy & WORK_BUSY_RUNNING); /* هنا race condition محتمل */
}
```

**نقاط مهمة للـ WARN_ON:**
1. بعد `cancel_work_sync()` — تأكد إن `work_pending()` يرجع false
2. في بداية الـ handler — تأكد إن `current_work() == work`
3. عند الـ destroy — تأكد إن الـ work مش في أي queue
4. عند تغيير الـ wq attrs — تأكد إن الـ wq مش بيشتغل

---

### Hardware Level

#### 1. التحقق من توافق حالة الـ Hardware مع الـ Kernel

الـ workqueue subsystem هو abstraction خالص في الـ software ومش بيتعامل مباشرة مع hardware. لكن المشاكل الأكثر شيوعاً بتيجي من الـ work handlers نفسها اللي بتتكلم مع hardware:

```bash
# تحقق إن الـ kworker threads موزّعة على الـ CPUs الصح
for cpu in /sys/devices/system/cpu/cpu*/; do
    echo -n "CPU $(basename $cpu): "
    cat ${cpu}online 2>/dev/null || echo "N/A"
done

# تأكد إن الـ CPUs المحددة في cpumask فعلاً online
cpu_mask=$(cat /sys/bus/workqueue/devices/events/cpumask)
echo "Workqueue cpumask: $cpu_mask"

# قارن مع الـ online CPUs
cat /sys/devices/system/cpu/online
```

---

#### 2. تقنيات الـ Register Dump

الـ workqueue نفسه مش عنده registers hardware، بس ممكن تحتاج تقرأ state الـ worker threads عبر الـ proc:

```bash
# اقرأ state الـ kworker thread
PID=$(pgrep -f "kworker/2:1")
cat /proc/$PID/status
cat /proc/$PID/wchan    # الـ kernel function اللي الـ worker نايم فيها

# اقرأ الـ stack trace للـ kworker المتوقف
cat /proc/$PID/stack

# لو عندك devmem2 وعارف عنوان الـ work_struct
# (مفيد في embedded debugging)
devmem2 0xffff888012345678 w   # اقرأ الـ data field
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

الـ workqueue debugging بالـ logic analyzer بيكون مفيد لما الـ work handler بيتحكم في GPIO أو hardware signals:

```c
/* ضيف GPIO toggling في الـ work handler للـ timing measurement */
static void my_work_handler(struct work_struct *work)
{
    /* بداية الـ work — اطبع pulse على GPIO */
    gpiod_set_value(debug_gpio, 1);

    /* ... الشغل الفعلي ... */

    /* نهاية الـ work */
    gpiod_set_value(debug_gpio, 0);
}
```

**نقاط القياس المفيدة:**
- القياس بين `queue_work()` ولحظة بداية التنفيذ = **scheduling latency**
- مدة تنفيذ الـ handler = **execution time**
- التكرار بين كل execute = **frequency analysis**

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ Log | التشخيص |
|---------|----------------|---------|
| IRQ storm يملّي الـ workqueue | كتير من `workqueue_queue_work` events في وقت قصير | راجع الـ interrupt handler وشوف لو بيعمل queue بشكل مفرط |
| قطع في CPU Hotplug أثناء تشغيل work | `workqueue: per-cpu work items are flushed` + `CPU X is now offline` | تحقق من `WQ_PERCPU` behavior عند CPU hotplug |
| NUMA imbalance | workers كلها على node واحد | غيّر `affinity_scope` لـ `numa` في sysfs |
| DMA completion في wrong context | `sleeping function called in softirq context` | استخدم `system_wq` بدل `system_bh_wq` للـ DMA completions |

---

#### 5. تشخيص الـ Device Tree

الـ workqueue مش بيعتمد على الـ DT مباشرة، لكن الـ drivers اللي بتستخدمه ممكن:

```bash
# تحقق من الـ DT nodes الخاصة بالـ driver اللي بيستخدم الـ workqueue
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 5 "my-device"

# تحقق من أن الـ interrupts محددة صح في الـ DT
# (لأن كتير من الـ drivers بتعمل queue_work من الـ ISR)
cat /proc/interrupts | grep my_driver

# تأكد من الـ NUMA topology للـ device
cat /sys/bus/platform/devices/my_device/numa_node
```

---

### Practical Commands

#### مجموعة الأوامر الجاهزة للنسخ

**1. فحص شامل سريع لحالة الـ workqueues:**
```bash
#!/bin/bash
echo "=== Workqueue Status ==="
echo "--- System WQs ---"
ls /sys/bus/workqueue/devices/ 2>/dev/null || echo "WQ_SYSFS not enabled"

echo "--- Active kworker threads ---"
ps -eo pid,psr,stat,comm | grep kworker | wc -l

echo "--- Congestion check via sysrq ---"
echo t > /proc/sysrq-trigger
dmesg | tail -50 | grep -E "(workqueue|kworker|pool)" | head -20
```

**2. تفعيل الـ ftrace الكامل للـ workqueue:**
```bash
#!/bin/bash
# وقّف أي tracing سابق
echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace

# فعّل workqueue events
for event in queue_work activate_work execute_start execute_end; do
    echo 1 > /sys/kernel/tracing/events/workqueue/workqueue_${event}/enable
done

# سجّل لمدة 5 ثواني
echo 1 > /sys/kernel/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/tracing/tracing_on

# احفظ النتيجة
cp /sys/kernel/tracing/trace /tmp/wq_trace_$(date +%s).txt
echo "Saved to /tmp/wq_trace_*.txt"
```

**مثال على المخرجات وتفسيرها:**
```
# tracer: nop
#
# TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
     kworker/2:1-47 [002] ....  1234.567890: workqueue_queue_work:
        work struct=0xffff888012340000 function=process_one_work
        workqueue=events req_cpu=4294967295 cpu=2
     kworker/2:1-47 [002] ....  1234.567891: workqueue_activate_work:
        work struct=0xffff888012340000
     kworker/2:1-47 [002] ....  1234.567892: workqueue_execute_start:
        work struct=0xffff888012340000 function=my_driver_work
     kworker/2:1-47 [002] ....  1234.568100: workqueue_execute_end:
        work struct=0xffff888012340000 function=my_driver_work
```

**التفسير:**
- الـ latency من queue → execute_start = `1234.567892 - 1234.567890 = 0.002ms` — ممتاز
- مدة التنفيذ = `1234.568100 - 1234.567892 = 0.208ms`

**3. تشخيص الـ workqueue lockup:**
```bash
# فعّل الـ watchdog
echo 30 > /sys/module/workqueue/parameters/watchdog_thresh

# راقب الـ lockup messages
dmesg -w | grep -E "(lockup|stuck|BUG.*workqueue)"

# لو لقيت lockup، اعرف أي pool المشكلة
echo t > /proc/sysrq-trigger
dmesg | grep -A 20 "Showing busy workqueues"
```

**مثال مخرجات workqueue lockup:**
```
[ 1234.567890] BUG: workqueue lockup - pool cpus=0-3 node=0 flags=0x0 nice=0 stuck for 31s!
[ 1234.567891] Showing busy workqueues and worker pools:
[ 1234.567892] workqueue events: flags=0x0
[ 1234.567893]   pwq 0: cpus=0-3 node=0 flags=0x0 nice=0 active=1/256 refcnt=2
[ 1234.567894]   in-flight: 47:my_blocking_work_handler
[ 1234.567895]   pending: another_work, yet_another_work
```

**التفسير:** `my_blocking_work_handler` بيبلّك — الـ PID 47 متوقف

```bash
# افحص الـ PID المتوقف
cat /proc/47/stack
cat /proc/47/wchan
```

**4. فحص الـ work_struct state في runtime:**
```bash
# لو عندك عنوان الـ work_struct من الـ crash dump أو ftrace
# افحص الـ flags (أول 4/5 bits من الـ data field)
python3 -c "
data = 0xffff888012340001  # مثال: WORK_STRUCT_PENDING set
flags = data & 0x1f  # 5 bits لو DEBUG_OBJECTS_WORK
pending = bool(flags & (1 << 0))
inactive = bool(flags & (1 << 1))
has_pwq = bool(flags & (1 << 2))
linked = bool(flags & (1 << 3))
print(f'PENDING={pending} INACTIVE={inactive} HAS_PWQ={has_pwq} LINKED={linked}')
"
```

**5. تتبع delayed_work timing:**
```bash
# فعّل timer tracing + workqueue tracing معاً
echo 1 > /sys/kernel/tracing/events/timer/enable
echo 1 > /sys/kernel/tracing/events/workqueue/enable
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل الكود اللي فيه delayed_work ثم:
grep -E "(timer_start|delayed_work_timer_fn|workqueue_queue_work)" \
    /sys/kernel/tracing/trace | head -30
```

**6. فحص أداء unbound workqueue:**
```bash
# قارن affinity scopes مختلفة
wq_name="my_unbound_wq"
for scope in cpu smt cache numa system; do
    echo $scope > /sys/bus/workqueue/devices/${wq_name}/affinity_scope
    echo "=== Scope: $scope ==="
    cat /sys/bus/workqueue/devices/${wq_name}/cpumask
    # شغّل الـ benchmark هنا
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — deadlock بسبب flush_workqueue على system_wq

#### العنوان
**Deadlock** في industrial gateway بسبب استدعاء `flush_workqueue` على system-wide workqueue من داخل work handler

#### السياق
gateway صناعي يشتغل على **RK3562** بيستخدم Modbus over RS485 (UART). الـ driver بيقرأ البيانات من الـ UART في interrupt handler، وبيـqueue work على `system_wq` لمعالجة البروتوكول. مشروع embedded Linux، kernel 6.6.

#### المشكلة
الـ gateway بيتفرز (hang كامل) بعد تشغيل تقريباً 4 ساعات تحت load. الـ watchdog بيعمل reset. في الـ log قبل الـ hang:

```
INFO: task kworker/u8:2:142 blocked for more than 120 seconds.
```

#### التحليل
الكود المشكلة:

```c
/* modbus_worker — runs in system_wq context */
static void modbus_process_work(struct work_struct *work)
{
    struct modbus_dev *mdev = from_work(mdev, work, rx_work);

    /* ... process frame ... */

    /* BUG: flush_workqueue on system_wq from INSIDE system_wq worker! */
    flush_workqueue(system_wq);  /* deadlock: worker waits for itself */
}
```

الـ `flush_workqueue` macro في `workqueue.h` بيستدعي `__flush_workqueue(system_wq)`. الـ kernel بيحاول يستنى لحد ما كل الـ work items الـ pending تخلص — لكن الـ worker نفسه هو اللي بيعمل الـ flush، فبيحصل self-deadlock.

علاوة على كده، `flush_workqueue` على أي system-wide workqueue (`system_wq`, `system_percpu_wq`, etc.) بيطلع warning:

```c
#define flush_workqueue(wq) \
({ \
    ... \
    if (... _wq == system_percpu_wq ...) \
        __warn_flushing_systemwide_wq(); \
    __flush_workqueue(_wq); \
})
```

والـ `__warn_flushing_systemwide_wq` معمولها:
```c
extern void __warn_flushing_systemwide_wq(void)
    __compiletime_warning("Please avoid flushing system-wide workqueues.");
```

يعني حتى لو فاتت compile بدون error، الـ runtime warning كانت موجودة في dmesg لكن حد تجاهلها.

#### الحل

**1. أنشئ dedicated workqueue بدل system_wq:**

```c
/* في probe */
mdev->wq = alloc_ordered_workqueue("modbus_%s", WQ_MEM_RECLAIM,
                                    dev_name(&pdev->dev));
INIT_WORK(&mdev->rx_work, modbus_process_work);
```

**2. احذف flush_workqueue من داخل الـ worker تماماً — استخدم cancel_work_sync عند cleanup بس:**

```c
static void modbus_process_work(struct work_struct *work)
{
    struct modbus_dev *mdev = from_work(mdev, work, rx_work);
    /* process only, no flush */
}

/* في remove/cleanup فقط */
static int modbus_remove(struct platform_device *pdev)
{
    cancel_work_sync(&mdev->rx_work);
    destroy_workqueue(mdev->wq);
}
```

**3. debug commands للتشخيص:**

```bash
# شوف الـ blocked workers
cat /proc/sysrq-trigger  # ثم 't' لـ task dump
# أو
echo t > /proc/sysrq-trigger

# شوف state الـ workqueues
cat /sys/kernel/debug/workqueue/* 2>/dev/null

# أو الأسهل
dmesg | grep -i "workqueue\|blocked\|kworker"
```

#### الدرس المستفاد
**لا تستخدم `flush_workqueue` على system-wide workqueues أبداً** — الـ header نفسه بيحذرك بـ compile-time warning. وأبداً لا تعمل flush من داخل worker على نفس الـ queue. دايماً أنشئ dedicated workqueue لكل driver.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI hotplug يخسر events بسبب race في queue_work

#### العنوان
**Race condition** في HDMI hotplug detection على Allwinner H616 — الشاشة بتتعرفش أحياناً عند التوصيل

#### السياق
Android TV box بيستخدم **Allwinner H616** مع HDMI output. الـ HDMI driver بيـdetect الـ HPD (Hot Plug Detect) interrupt وبيـqueue work لقراءة الـ EDID وتهيئة الـ display pipeline.

#### المشكلة
بنسبة ~15% من المرات، لما المستخدم يوصل الـ HDMI، الشاشة مبتعرفش لحد ما يشيل ويرجع التوصيل. في الـ log:

```
hdmi-sun50i-h6 6000000.hdmi: HPD interrupt fired
hdmi-sun50i-h6 6000000.hdmi: HPD interrupt fired
```

بيظهر الـ interrupt مرتين لكن الـ display مبيتهيئش.

#### التحليل
الكود الأصلي:

```c
static irqreturn_t hdmi_hpd_irq(int irq, void *dev_id)
{
    struct hdmi_dev *hdev = dev_id;

    /* queue_work returns false if work ALREADY PENDING */
    queue_work(hdev->wq, &hdev->hpd_work);

    return IRQ_HANDLED;
}
```

الـ `queue_work` بترجع `false` لو الـ work item كان already pending:

```c
/* من workqueue.h */
static inline bool queue_work(struct workqueue_struct *wq,
                               struct work_struct *work)
{
    return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```

`queue_work_on` داخلياً بيعمل `test_and_set_bit(WORK_STRUCT_PENDING_BIT, ...)`. لو الـ bit كان set بالفعل (الـ work لسه شاغل من interrupt أول) — الـ second queue بتتجاهل.

الـ scenario:
1. HPD interrupt الأولانية — queue_work تنجح (returns true)، الـ work بيتـqueue
2. قبل ما الـ worker يخلص، HPD interrupt تانية جت (bounce) — queue_work returns **false**، الـ work اتحطش تاني
3. الـ worker الأولاني شاف الـ cable disconnected (bounced state) وما عملش حاجة
4. مفيش work تاني لـdetect الـ final connected state

#### الحل

استخدم **flag + re-check** أو `mod_delayed_work` عشان تـdebounce:

```c
#define HPD_DEBOUNCE_MS  100

static void hdmi_hpd_work_fn(struct work_struct *work)
{
    struct hdmi_dev *hdev = from_work(hdev, work, hpd_dwork.work);
    bool connected = hdmi_read_hpd_pin(hdev);

    if (connected)
        hdmi_enable_display(hdev);
    else
        hdmi_disable_display(hdev);
}

static irqreturn_t hdmi_hpd_irq(int irq, void *dev_id)
{
    struct hdmi_dev *hdev = dev_id;

    /*
     * mod_delayed_work resets the timer even if work is pending —
     * this naturally debounces rapid HPD transitions
     */
    mod_delayed_work(hdev->wq, &hdev->hpd_dwork,
                     msecs_to_jiffies(HPD_DEBOUNCE_MS));

    return IRQ_HANDLED;
}
```

```bash
# تأكيد الـ fix — شوف إن الـ delayed work بيتـqueue صح
cat /sys/kernel/debug/workqueue/*/stat
```

#### الدرس المستفاد
`queue_work` بتـdrop الـ work لو كانت already pending — ده intentional behavior موثق في الـ header. لأي event-driven hardware (HPD, GPIO, USB VBUS)، استخدم `INIT_DELAYED_WORK` مع `mod_delayed_work` عشان تعمل debounce وتضمن إن آخر state هو اللي بيتـprocess.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — suspend/resume hang بسبب WQ_FREEZABLE مش موجود

#### العنوان
**System hang** عند الدخول في suspend على STM32MP1 IoT sensor board بسبب workqueue مش freezable

#### السياق
IoT sensor board بيستخدم **STM32MP1** بـbattery، بيـcollect بيانات من I2C sensors (BME280, BH1750) وبيـupload لـ cloud. الـ system بيستخدم runtime PM وـsuspend-to-RAM لتوفير battery.

#### المشكلة
الـ system بيـhang أحياناً عند دخول suspend. الـ log:

```
PM: Syncing filesystems ... done.
Freezing user space processes ... done.
Freezing remaining freezable tasks ...
```

بيتوقف هنا indefinitely.

#### التحليل
الـ sensor driver بيعمل workqueue بدون `WQ_FREEZABLE`:

```c
/* في probe — خطأ: مفيش WQ_FREEZABLE */
priv->wq = alloc_workqueue("sensor_collect", WQ_UNBOUND, 0);
INIT_DELAYED_WORK(&priv->collect_dwork, sensor_collect_fn);

/* schedule بيتكرر كل ثانية */
static void sensor_collect_fn(struct work_struct *work)
{
    struct sensor_priv *priv = from_work(priv, work, collect_dwork.work);

    read_i2c_sensors(priv);
    upload_data(priv);  /* هذا بيـblock لحد ما الـ network يرد */

    /* reschedule نفسه */
    queue_delayed_work(priv->wq, &priv->collect_dwork, HZ);
}
```

عند suspend، الـ kernel بيحاول يـfreeze كل الـ tasks والـ workqueues. الـ workqueues اللي عليها `WQ_FREEZABLE` بتـstop بعد إن الـ current work يخلص. لكن الـ workqueue ده مفيش عليه `WQ_FREEZABLE`، فالـ kernel بيفضل يستنى إنه يـfreeze وده مبيحصلش.

من `workqueue.h`:
```c
enum wq_flags {
    WQ_FREEZABLE = 1 << 2, /* freeze during suspend */
    ...
};
```

وفي كود الـ freezer:
```c
/* kernel/freezer.c — بيـcheck WQ_FREEZABLE flag */
```

#### الحل

**1. أضف `WQ_FREEZABLE`:**

```c
priv->wq = alloc_workqueue("sensor_collect",
                            WQ_UNBOUND | WQ_FREEZABLE | WQ_MEM_RECLAIM,
                            0);
```

**2. استخدم `system_freezable_wq` لو مش محتاج dedicated queue:**

```c
/* بدل alloc_workqueue */
INIT_DELAYED_WORK(&priv->collect_dwork, sensor_collect_fn);

/* queue على system_freezable_wq */
queue_delayed_work(system_freezable_wq, &priv->collect_dwork, HZ);
```

**3. اعمل cleanup صح في suspend callback:**

```c
static int sensor_suspend(struct device *dev)
{
    struct sensor_priv *priv = dev_get_drvdata(dev);
    cancel_delayed_work_sync(&priv->collect_dwork);
    return 0;
}

static int sensor_resume(struct device *dev)
{
    struct sensor_priv *priv = dev_get_drvdata(dev);
    queue_delayed_work(priv->wq, &priv->collect_dwork, HZ);
    return 0;
}
```

```bash
# debug الـ freeze issue
dmesg | grep -i "freezing\|frozen\|workqueue"

# شوف مين اللي blocking الـ freeze
cat /proc/$(pgrep kworker | head -1)/status
```

#### الدرس المستفاد
أي workqueue بتـdo I/O أو network وبتـreschedule نفسها في نظام بيستخدم suspend **لازم** يكون عليها `WQ_FREEZABLE` أو يتـmanage يدوياً في suspend/resume callbacks. `system_freezable_wq` هو shortcut ممتاز للـ sensor drivers.

---

### السيناريو 4: Automotive ECU على i.MX8 — priority inversion في CAN bus processing بسبب غياب WQ_HIGHPRI

#### العنوان
**Real-time latency violation** في automotive ECU على i.MX8 — الـ CAN safety messages بتتأخر

#### السياق
automotive ECU بيستخدم **i.MX8QM** بيـprocess CAN bus messages. فيه نوعين messages: safety-critical (brake, steering) بتحتاج latency < 1ms، وnon-critical (infotainment data) ممكن تتأخر. المشروع يستخدم Linux مع PREEMPT_RT.

#### المشكلة
في الـ testing، safety-critical CAN messages بتتأخر أحياناً 3-5ms فوق الـ threshold المسموح. الـ automotive certification team ترفض الـ build.

#### التحليل
الـ driver الأصلي:

```c
struct can_ecu {
    struct workqueue_struct *can_wq; /* workqueue واحد للكل */
    struct work_struct safety_work;
    struct work_struct infotainment_work;
};

static int can_ecu_probe(struct platform_device *pdev)
{
    /* workqueue واحد بدون priority — خطأ */
    ecu->can_wq = alloc_workqueue("can_ecu", WQ_UNBOUND, 8);

    INIT_WORK(&ecu->safety_work, can_safety_handler);
    INIT_WORK(&ecu->infotainment_work, can_info_handler);
}

static irqreturn_t can_rx_irq(int irq, void *dev_id)
{
    /* كل المsssages على نفس الـ queue */
    if (is_safety_frame(frame))
        queue_work(ecu->can_wq, &ecu->safety_work);
    else
        queue_work(ecu->can_wq, &ecu->infotainment_work);
}
```

المشكلة: الـ `infotainment_work` بياخد CPU time وبيأخر الـ `safety_work` لأنهم على نفس الـ queue بنفس الـ priority.

من `workqueue.h`:
```c
enum wq_flags {
    WQ_HIGHPRI = 1 << 4, /* high priority */
    ...
};
```

الـ workers على `WQ_HIGHPRI` workqueue بيتـschedule على `kworker/u:Hx` threads اللي لها nice value أقل (أعلى priority).

#### الحل

**افصل الـ workqueues حسب الـ priority:**

```c
static int can_ecu_probe(struct platform_device *pdev)
{
    /* high priority queue للـ safety messages */
    ecu->safety_wq = alloc_workqueue("can_safety",
                                      WQ_UNBOUND | WQ_HIGHPRI | WQ_MEM_RECLAIM,
                                      4);

    /* normal queue للـ infotainment */
    ecu->info_wq = alloc_workqueue("can_info",
                                    WQ_UNBOUND,
                                    16);

    INIT_WORK(&ecu->safety_work, can_safety_handler);
    INIT_WORK(&ecu->infotainment_work, can_info_handler);
}

static irqreturn_t can_rx_irq(int irq, void *dev_id)
{
    if (is_safety_frame(frame))
        /* WQ_HIGHPRI worker يـpreempt normal workers */
        queue_work(ecu->safety_wq, &ecu->safety_work);
    else
        queue_work(ecu->info_wq, &ecu->infotainment_work);
}
```

**تعديل إضافي — تحديد الـ CPU affinity لـ determinism:**

```c
struct workqueue_attrs *attrs = alloc_workqueue_attrs();
attrs->nice = -20;  /* max priority */
cpumask_set_cpu(3, attrs->cpumask);  /* pin to isolated CPU3 */
apply_workqueue_attrs(ecu->safety_wq, attrs);
free_workqueue_attrs(attrs);
```

```bash
# تحقق من الـ priority
ps aux | grep kworker | grep -i "can\|H"

# قياس الـ latency
sudo cyclictest -p 99 -t 4 -n -q -D 60s
```

#### الدرس المستفاد
في الـ real-time و safety-critical systems، **لازم تفصل الـ workqueues حسب الـ criticality**. `WQ_HIGHPRI` بيخلي الـ workers تشتغل على threads بـpriority أعلى. وممكن تستخدم `apply_workqueue_attrs` لتحديد الـ CPU affinity وضمان الـ determinism المطلوب للـ automotive certification.

---

### السيناريو 5: Custom Board Bring-up على AM62x — memory leak من عدم destroy الـ workqueue في error path

#### العنوان
**Memory leak** عند board bring-up على AM62x — الـ driver probe بيفشل وبيسيب workqueue معلقة

#### السياق
bring-up لـ custom board بيستخدم **TI AM62x** مع SPI-connected sensor array. الـ driver جديد، وعند testing لقينا إن الـ `/proc/meminfo` بيقل كل مرة بيفشل فيها الـ driver probe (مثلاً لما الـ SPI device مش موجود).

#### المشكلة
```bash
# بعد 100 modprobe/rmmod cycle على board بدون الـ SPI device
$ cat /proc/meminfo | grep MemFree
MemFree:          18432 kB   # كانت 64MB!
```

الـ memory بتتاكل مع كل failed probe.

#### التحليل
الكود المشكلة:

```c
static int spi_sensor_probe(struct spi_device *spi)
{
    struct sensor_dev *sdev;
    int ret;

    sdev = devm_kzalloc(&spi->dev, sizeof(*sdev), GFP_KERNEL);
    if (!sdev)
        return -ENOMEM;

    /* alloc workqueue — بيـalloc memory */
    sdev->wq = alloc_workqueue("spi_sensor_%s",
                                WQ_UNBOUND | WQ_MEM_RECLAIM, 4,
                                dev_name(&spi->dev));
    if (!sdev->wq)
        return -ENOMEM;

    INIT_WORK(&sdev->read_work, spi_sensor_read_fn);

    /* SPI communication test */
    ret = spi_sensor_check_id(sdev);
    if (ret) {
        /* BUG: مرجعناش من غير ما نعمل destroy للـ workqueue! */
        return ret;
    }

    /* ... rest of probe ... */
}
```

`alloc_workqueue` بيـalloc structures من الـ kernel memory. لو الـ probe فشل بعد الـ alloc بدون `destroy_workqueue`، الـ memory بتضيع.

ملاحظة: الـ `devm_kzalloc` بيتحرر تلقائياً، لكن `alloc_workqueue` **مش** `devm`-managed — لازم تتحرر يدوياً.

#### الحل

**الحل الأساسي — error path صح:**

```c
static int spi_sensor_probe(struct spi_device *spi)
{
    struct sensor_dev *sdev;
    int ret;

    sdev = devm_kzalloc(&spi->dev, sizeof(*sdev), GFP_KERNEL);
    if (!sdev)
        return -ENOMEM;

    sdev->wq = alloc_workqueue("spi_sensor_%s",
                                WQ_UNBOUND | WQ_MEM_RECLAIM, 4,
                                dev_name(&spi->dev));
    if (!sdev->wq)
        return -ENOMEM;

    INIT_WORK(&sdev->read_work, spi_sensor_read_fn);

    ret = spi_sensor_check_id(sdev);
    if (ret)
        goto err_destroy_wq;  /* cleanup قبل الـ return */

    spi_set_drvdata(spi, sdev);
    return 0;

err_destroy_wq:
    destroy_workqueue(sdev->wq);  /* حرر الـ memory */
    return ret;
}

static void spi_sensor_remove(struct spi_device *spi)
{
    struct sensor_dev *sdev = spi_get_drvdata(spi);

    cancel_work_sync(&sdev->read_work);
    destroy_workqueue(sdev->wq);
}
```

**الحل الأفضل — استخدم devm_add_action_or_reset:**

```c
static void spi_sensor_wq_release(void *data)
{
    destroy_workqueue((struct workqueue_struct *)data);
}

static int spi_sensor_probe(struct spi_device *spi)
{
    /* ... */
    sdev->wq = alloc_workqueue("spi_sensor_%s", WQ_UNBOUND, 4,
                                dev_name(&spi->dev));
    if (!sdev->wq)
        return -ENOMEM;

    /* register cleanup automatically — safe for any error path after this */
    ret = devm_add_action_or_reset(&spi->dev, spi_sensor_wq_release, sdev->wq);
    if (ret)
        return ret;

    /* أي return بعد كده بيـtrigger الـ cleanup تلقائياً */
    ret = spi_sensor_check_id(sdev);
    if (ret)
        return ret;  /* spi_sensor_wq_release بيتكال automatically */
    /* ... */
}
```

```bash
# كشف الـ leak أثناء bring-up
# قبل وبعد modprobe
cat /proc/meminfo | grep -E "MemFree|Slab"
modprobe spi_sensor
cat /proc/meminfo | grep -E "MemFree|Slab"

# أو استخدم kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
`alloc_workqueue` **مش** resource-managed — لازم يقابلها `destroy_workqueue` في كل error path وفي الـ `remove` callback. أفضل pattern في bring-up هو `devm_add_action_or_reset` عشان يضمن cleanup تلقائي بغض النظر عن أي error path بييجي بعده. كمان `cancel_work_sync` أو `cancel_delayed_work_sync` لازم تيجي **قبل** `destroy_workqueue` عشان تضمن مفيش work شاغل وقت الـ destroy.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لتاريخ وتطور الـ workqueue subsystem في الـ Linux kernel.

| المقال | الأهمية |
|--------|---------|
| [Details of the workqueue interface](https://lwn.net/Articles/11360/) | أول توثيق للـ API — الـ work queue كـ list of tasks مع per-CPU kernel thread |
| [Driver porting: the workqueue interface](https://lwn.net/Articles/23634/) | دليل عملي للـ driver developers على الـ workqueue API القديم |
| [Workqueues get a rework](https://lwn.net/Articles/211279/) | تغييرات الـ API في مراحل مبكرة من إعادة التصميم |
| [Overview of concurrency managed workqueue](https://lwn.net/Articles/393172/) | نظرة عامة على تصميم الـ **cmwq** (Concurrency Managed Workqueue) بقلم Tejun Heo |
| [Concurrency-managed workqueues and thread priorities](https://lwn.net/Articles/393171/) | تفاصيل الـ thread priorities وآلية الـ concurrency management |
| [workqueue: concurrency managed workqueue, take#6](https://lwn.net/Articles/394084/) | الـ patch set النهائي للـ cmwq قبيل الدمج في kernel 2.6.36 |
| [Working on workqueues](https://lwn.net/Articles/403891/) | تأثير الـ cmwq على الـ kworker threads بعد دمجه في 2.6.36 |
| [Power-efficient workqueues](https://lwn.net/Articles/731052/) | شرح `WQ_POWER_EFFICIENT` وكيف يقلل استهلاك الطاقة |
| [workqueue: Improve unbound workqueue execution locality](https://lwn.net/Articles/932431/) | تحسينات الـ locality للـ unbound workqueues مع الـ LLC pods |
| [A pair of workqueue improvements](https://lwn.net/Articles/937416/) | تحسينات إضافية على الـ concurrency model وآلية الـ dispatch |

---

### التوثيق الرسمي للـ Kernel

#### الـ Documentation/ paths

```
Documentation/core-api/workqueue.rst
```

الـ URL المباشر للنسخة الأحدث:

- [Workqueue — The Linux Kernel documentation (latest)](https://www.kernel.org/doc/html/latest/core-api/workqueue.html)
- [Concurrency Managed Workqueue (cmwq) — v5.15](https://www.kernel.org/doc/html/v5.15/core-api/workqueue.html)
- [Workqueue — static.lwn.net (kerneldoc mirror)](https://static.lwn.net/kerneldoc/core-api/workqueue.html)

#### الـ source files الرئيسية في الـ kernel tree

```
include/linux/workqueue.h          ← الـ public API والـ macros
include/linux/workqueue_types.h    ← الـ struct work_struct و struct workqueue_struct
kernel/workqueue.c                 ← التنفيذ الكامل للـ subsystem
```

---

### الـ Kernel Commits المهمة

| الحدث | الوصف |
|-------|-------|
| **cmwq introduction — kernel 2.6.36** | الـ commit الأساسي بقلم Tejun Heo لإعادة كتابة الـ workqueue subsystem بالكامل — استبدال per-workqueue threads بـ shared worker pools |
| **WQ_POWER_EFFICIENT — kernel 4.x** | إضافة الـ flag لتحويل per-cpu workqueues إلى unbound عند تفعيل `workqueue.power_efficient` |
| **WQ_UNBOUND + locality pods** | تحسينات الـ NUMA locality وإضافة `wq_affn_scope` والـ pod-based pools |
| **WQ_BH + system_bh_wq** | إضافة الـ BH (bottom-half/softirq) context workqueues كـ convenience interface |
| **workqueue watchdog** | إضافة `CONFIG_WQ_WATCHDOG` للكشف عن الـ stall في الـ worker threads |
| **disable/enable work API** | إضافة `disable_work()` / `enable_work()` وآلية الـ disable depth |

**الـ git pull request للـ v6.13** (أحدث تغييرات موثقة):
- [GIT PULL workqueue: Changes for v6.13 — Tejun Heo](https://lore.kernel.org/lkml/Zztf9nRuKNjH34Bd@slm.duckdns.org/)

**الـ source على GitHub** للاستعراض السريع للتاريخ:
- [kernel/workqueue.c — torvalds/linux](https://github.com/torvalds/linux/blob/master/kernel/workqueue.c)

---

### نقاشات الـ Mailing List

| المصدر | الرابط |
|--------|--------|
| LKML Archive (بحث: workqueue) | [lkml.org](https://lkml.org/) |
| lore.kernel.org (الأحدث والأكثر اكتمالاً) | `https://lore.kernel.org/linux-kernel/?q=workqueue` |
| نقاش تحويل system-wide flush | [lkml.kernel.org/r/49925af7-...](https://lkml.kernel.org/r/49925af7-78a8-a3dd-bce6-cfc02e1a9236@I-love.SAKURA.ne.jp) |

> **ملاحظة:** الـ URL الأخير مذكور مباشرةً في `include/linux/workqueue.h` السطر 763 كمرجع لسبب deprecation الـ `flush_workqueue()` على الـ system-wide workqueues.

---

### الـ Kernelnewbies.org

- [Linux 2.6.36 — cmwq introduction](https://kernelnewbies.org/Linux_2_6_36): تفاصيل الـ workqueue redesign ووصف الـ kworker threads
- [Tejun Heo page](https://kernelnewbies.org/Tejun_Heo): الصفحة الخاصة بالمطور الرئيسي للـ cmwq مع روابط لأعماله
- [Linux 3.10 — sysfs interface](https://kernelnewbies.org/Linux_3.10): إضافة `/sys/bus/workqueue/devices/WQ_NAME`
- [Linux 4.5 — watchdog](https://kernelnewbies.org/Linux_4.5): تفاصيل workqueue lockup detector
- [Linux 6.12](https://kernelnewbies.org/Linux_6.12) و [Linux 6.13](https://kernelnewbies.org/Linux_6.13): آخر التحسينات

---

### الـ eLinux.org

لا يوجد صفحة مخصصة للـ workqueue على elinux.org. للمعلومات المتعلقة بالـ embedded context:
- [High Resolution Timers — eLinux.org](https://elinux.org/High_Resolution_Timers): مكمل للـ `delayed_work` والـ `INIT_DEFERRABLE_WORK`

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)

| الفصل | المحتوى |
|-------|---------|
| **Chapter 7: Time, Delays, and Deferred Work** | الـ `work_struct`، الـ `INIT_WORK`، الـ `schedule_work`، الـ `delayed_work` |

> تنبيه: LDD3 يصف الـ API القديم ما قبل الـ cmwq (pre-2.6.36). الـ concepts صحيحة لكن بعض الـ functions مُهملة (`create_workqueue` وغيرها).

#### Linux Kernel Development, 3rd Edition — Robert Love

| الفصل | المحتوى |
|-------|---------|
| **Chapter 8: Bottom Halves and Deferring Work** | مقارنة الـ bottom halves: softirqs، tasklets، workqueues. الـ `work_struct` lifecycle كاملاً |

> الكتاب يغطي kernel 2.6.34 — قبل الـ cmwq مباشرةً، لكنه الأفضل للفهم الأساسي.

#### Understanding the Linux Kernel, 3rd Edition — Bovet & Cesati

- **Chapter 4** و **Chapter 11**: الـ process scheduling وكيف يتعامل الـ kernel مع الـ deferred work في سياق الـ kernel threads.

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan

- **Chapter 14: Kernel Debugging Techniques**: يتناول الـ workqueue stall وأدوات الـ debugging بما فيها الـ watchdog والـ `show_all_workqueues()`.

---

### الـ Presentation Materials

- [Async execution with workqueues — Bhaktipriya Shridhar (Linux Foundation)](https://events.static.linuxfound.org/sites/events/files/slides/Async%20execution%20with%20wqs.pdf): شرائح تقدم عملي ممتاز للـ workqueue API وأنماط الاستخدام

---

### Search Terms للبحث عن مزيد من المعلومات

```
linux kernel workqueue cmwq
linux workqueue_struct worker_pool
linux alloc_workqueue WQ_UNBOUND
linux delayed_work rcu_work
linux workqueue concurrency management
linux WQ_MEM_RECLAIM memory reclaim deadlock
linux workqueue power_efficient
linux kworker cpu 100%
linux workqueue flush cancel sync
linux workqueue affinity NUMA pod
linux WQ_BH softirq bottom half
linux workqueue watchdog stall
```

---

### ملخص المصادر حسب الأولوية

```
1. Documentation/core-api/workqueue.rst    ← الأهم — التوثيق الرسمي
2. lwn.net/Articles/393172/               ← فهم تصميم الـ cmwq
3. kernel/workqueue.c                     ← التنفيذ الفعلي
4. LKD Chapter 8 (Robert Love)            ← الأساس النظري
5. lwn.net/Articles/932431/               ← أحدث تطورات الـ locality
```
## Phase 8: Writing simple module

### الفكرة

سنعمل **kprobe** على الدالة `queue_work_on` — وهي نقطة دخول كل work item بيتحط على أي workqueue في الـ kernel. ده بيخلينا نشوف اسم الـ workqueue والـ CPU اللي اتقال عليه والـ function pointer للـ work نفسه في كل مرة يتقال فيها `queue_work_on`.

الاختيار ده مناسب لأن:
- الدالة `exported` وبتتنادى من كل حتة في الـ kernel.
- signature بتاعتها واضحة: `(int cpu, struct workqueue_struct *wq, struct work_struct *work)`.
- **kprobe** بتسمح لنا نقرأ الـ arguments قبل ما الدالة تتنفذ بدون ما نعدل أي حاجة.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * wq_probe.c — kprobe on queue_work_on() to trace work submissions
 */

/* Standard module headers */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

/* kprobe infrastructure */
#include <linux/kprobes.h>

/* workqueue types: work_struct, workqueue_struct, work_func_t */
#include <linux/workqueue.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Tracing Example");
MODULE_DESCRIPTION("Trace every queue_work_on() call via kprobe");

/* ---------------------------------------------------------------
 * pre_handler — بتتنادى مباشرة قبل ما queue_work_on تتنفذ.
 * بنقرأ الـ arguments من الـ registers عبر regs_get_kernel_argument().
 * --------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * queue_work_on(int cpu, struct workqueue_struct *wq,
     *               struct work_struct *work)
     *
     * على x86-64:
     *   arg0 (cpu)  → rdi
     *   arg1 (wq)   → rsi
     *   arg2 (work) → rdx
     *
     * regs_get_kernel_argument() بتجيب الـ argument رقم N
     * بطريقة portable مش محتاج نعرف register بالاسم.
     */
    int cpu = (int)regs_get_kernel_argument(regs, 0);

    /* pointer للـ workqueue — مش هنعمل deref كامل، بس هنطبع العنوان */
    struct workqueue_struct *wq =
        (struct workqueue_struct *)regs_get_kernel_argument(regs, 1);

    /* pointer للـ work item اللي فيه الـ func callback */
    struct work_struct *work =
        (struct work_struct *)regs_get_kernel_argument(regs, 2);

    /*
     * نطبع:
     *  - اسم الـ process اللي بيعمل queue (current->comm)
     *  - رقم الـ CPU المطلوب (WORK_CPU_UNBOUND = NR_CPUS يعني "أي CPU")
     *  - عنوان الـ workqueue
     *  - عنوان الـ work_func_t اللي هيتنفذ
     *
     * استخدمنا %ps عشان الـ kernel يطبع اسم الـ symbol تلقائياً
     * لو الـ function موجودة في الـ kallsyms.
     */
    pr_info("wq_probe: comm=%-16s cpu=%d wq=%px func=%ps\n",
            current->comm,
            cpu,
            wq,
            work ? (void *)work->func : NULL);

    /* لازم نرجع 0 — أي قيمة تانية بتعمل single-step fault */
    return 0;
}

/* ---------------------------------------------------------------
 * kprobe struct — بتربط الـ handler بالدالة المستهدفة بالاسم.
 * --------------------------------------------------------------- */
static struct kprobe kp = {
    /* اسم الدالة كـ string — الـ kernel بيحلها عبر kallsyms */
    .symbol_name = "queue_work_on",
    .pre_handler = handler_pre,
};

/* ---------------------------------------------------------------
 * module_init — بيسجل الـ kprobe عند تحميل الـ module.
 * لو فشل التسجيل نرجع error code عشان insmod يفشل بشكل واضح.
 * --------------------------------------------------------------- */
static int __init wq_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("wq_probe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("wq_probe: kprobe planted on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit — لازم نشيل الـ kprobe قبل ما الـ module يتشال من الذاكرة.
 * لو ما عملناش ده، الـ kernel هيكمل ينادي handler_pre اللي بقى عنوانه
 * invalid → kernel panic.
 * --------------------------------------------------------------- */
static void __exit wq_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("wq_probe: kprobe removed\n");
}

module_init(wq_probe_init);
module_exit(wq_probe_exit);
```

---

### Makefile

```makefile
obj-m += wq_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية: `MODULE_LICENSE`, `module_init`, `module_exit` |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل دوال `register/unregister_kprobe` |
| `linux/workqueue.h` | تعريف `struct work_struct` و`struct workqueue_struct` عشان نعمل cast صح للـ arguments |

---

#### الـ `pre_handler`

الدالة بتتنادى في سياق الـ CPU اللي قال `queue_work_on` — يعني ممكن تكون في interrupt context أو process context. عشان كده:
- **مش** بنعمل أي memory allocation.
- بنستخدم `pr_info` فقط (آمنة في كل context لأنها بتكتب في ring buffer).
- **الـ `regs_get_kernel_argument(regs, N)`** portable على كل architecture — الـ kernel بيعرف هو يجيب الـ argument من الـ register الصح (rdi/rsi/rdx على x86-64، r0/r1/r2 على ARM64).

---

#### ليه `work->func` مهم؟

الـ `work_func_t func` جوه `struct work_struct` هو الـ callback الحقيقي اللي هيتنفذ في الـ worker thread. لما نطبعه كـ `%ps` بنعرف بالظبط مين اللي queue الشغل — مثلاً هنشوف أسماء زي `vmstat_update` أو `flush_to_ldisc` أو أي subsystem تاني.

---

#### الـ `module_exit` والـ unregister

لو خرجنا من الـ module من غير ما نعمل `unregister_kprobe`، الـ breakpoint هيفضل موجود في الـ kernel code وأي نداء لـ `queue_work_on` هيعمل int3 exception وهيـروح لـ handler على عنوان اتمسح → **kernel panic**. الـ unregister بيشيل الـ breakpoint ويستنى أي handler شغال يخلص قبل ما يرجع.

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod wq_probe.ko

# متابعة الـ output
sudo dmesg -w | grep wq_probe

# مثال على output متوقع:
# wq_probe: kprobe planted on queue_work_on @ ffffffffa1b2c3d0
# wq_probe: comm=kworker/u8:2   cpu=512 wq=ffff888... func=vmstat_update
# wq_probe: comm=jbd2/sda1-8    cpu=512 wq=ffff888... func=kjournald2_commit_transaction
# wq_probe: comm=NetworkManager cpu=0   wq=ffff888... func=process_one_work

# إزالته
sudo rmmod wq_probe
```

> **ملحوظة:** `cpu=512` يعني `WORK_CPU_UNBOUND` (= `NR_CPUS` على نظام 512-CPU config) — أي "اختار CPU بنفسك".
