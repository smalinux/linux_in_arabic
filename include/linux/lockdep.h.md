## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem؟

**الـ `lockdep.h`** جزء من subsystem اسمه **"LOCKING PRIMITIVES"** في الـ Linux kernel — وهو المسؤول عن كل أدوات الـ locking: spinlocks, mutexes, rwsems, rwlocks، وفوق كل ده **lockdep** نفسه، اللي هو الـ runtime validator بتاع صحة الـ locking.

---

### القصة من الأول

تخيل إنك بتبني مطعم كبير جداً فيه 100 طباخ شغالين في نفس الوقت. كل طباخ محتاج يمسك بعض الأدوات (locks) عشان يعمل شغله — مثلاً طباخ A محتاج يمسك "السكينة" الأول، وبعدين "الطاجن". طباخ B محتاج يمسك "الطاجن" الأول، وبعدين "السكينة".

النتيجة؟ **Deadlock** — A واقف يستنى الطاجن من B، وB واقف يستنى السكينة من A، وكلهم واقفين للأبد.

في الـ Linux kernel، ده بيحصل فعلاً. عندك آلاف الـ locks في كل مكان، وأي كود غلط في ترتيب أخذ الـ locks ممكن يخلي الـ system كله يقف. المشكلة إن الـ deadlock مش بيحصل دايماً — بيحصل بس في ظروف معينة، يعني ممكن تشحن الكود ويشتغل لأسابيع وبعدين الـ server يقف في الساعة 3 الصبح.

**اللي جه Ingo Molnar عام 2006 يعمله** هو: أنا مش هستنى الـ deadlock يحصل فعلاً — هشتغل كـ"مفتش" جنب كل lock، وكل مرة حد يمسك lock أسجّل "مين ماسك إيه وبأي ترتيب"، وبناء على ده أبني **graph** كامل للـ dependencies، وأكتشف أي **دايرة** (cycle) في الـ graph دي قبل ما الـ deadlock يحصل فعلاً.

ده هو **lockdep** — الـ **Runtime Locking Correctness Validator**.

---

### إيه اللي بيعمله lockdep تحديداً؟

```
                    ┌─────────────────────────────┐
                    │        Your Code             │
                    │  mutex_lock(A) → mutex_lock(B)│
                    └────────────┬────────────────┘
                                 │  كل lock acquire/release
                                 ▼
                    ┌─────────────────────────────┐
                    │         lockdep              │
                    │  1. سجّل الـ lock في الـ graph│
                    │  2. ابحث عن cycles (BFS)     │
                    │  3. تحقق من IRQ safety        │
                    │  4. تحقق من ترتيب الـ locking │
                    └────────────┬────────────────┘
                                 │
                    ┌────────────▼────────────────┐
                    │   لو لقى مشكلة → WARN/BUG   │
                    │   مع stack trace كامل        │
                    └─────────────────────────────┘
```

**lockdep** بيراقب 4 أنواع من المشاكل:

| المشكلة | التفسير |
|---------|---------|
| **Deadlock (cycle detection)** | A→B وB→A في نفس الوقت = مستحيل |
| **IRQ-safety violations** | lock أُخذ في hardirq context وفي normal context بدون disable irq |
| **Lock ordering violations** | نفس اللقلوب بس بترتيب عكسي في threads مختلفة |
| **Wrong context** | lock sleeping اتأخذ من atomic context |

---

### إيه اللي بيعمله `lockdep.h` تحديداً؟

الـ header ده هو **الواجهة العلنية** للـ lockdep — يعني هو اللي بيقول للكود التاني "عايز تتكلم مع lockdep؟ تعال من هنا".

فيه ثلاث طبقات:

#### 1. الـ Data Structures الأساسية

```c
/* كل lock instance عنده map بتربطه بـ class */
struct lockdep_map {
    struct lock_class_key   *key;           /* مفتاح فريد لكل نوع lock */
    struct lock_class       *class_cache[]; /* cache للأداء */
    const char              *name;          /* اسم الـ lock للـ debugging */
    u8                      wait_type_inner; /* نوع الانتظار */
    u8                      wait_type_outer;
    u8                      lock_type;
};

/* قائمة الـ dependencies بين الـ locks */
struct lock_list {
    struct lock_class   *class;    /* الـ lock الحالي */
    struct lock_class   *links_to; /* اللي بيرتبط بيه */
    u16                 distance;  /* بُعده في الـ graph */
    u8                  dep;       /* نوع الـ dependency */
    struct lock_list    *parent;   /* للـ BFS traversal */
};

/* سلسلة الـ locks المتداخلة */
struct lock_chain {
    unsigned int    irq_context : 2; /* hardirq أم softirq أم لا */
    unsigned int    depth       : 6; /* كام lock متداخل */
    u64             chain_key;       /* hash فريد للسلسلة */
};
```

#### 2. الـ Core API

```c
/* أهم وظيفة — بتتسمى تلقائياً من كل spin_lock/mutex_lock/etc */
extern void lock_acquire(struct lockdep_map *lock,
                         unsigned int subclass,
                         int trylock,
                         int read,    /* 0=write, 1=read, 2=recursive read */
                         int check,   /* 0=simple, 1=full */
                         struct lockdep_map *nest_lock,
                         unsigned long ip);  /* من فين اتعمل الـ call */

extern void lock_release(struct lockdep_map *lock, unsigned long ip);
```

#### 3. الـ Macros اللي بتربط كل lock types

```c
/* كل lock في الـ kernel بيمشي على lockdep من خلال macros زي دي */
#define spin_acquire(l, s, t, i)   lock_acquire_exclusive(l, s, t, NULL, i)
#define mutex_acquire(l, s, t, i)  lock_acquire_exclusive(l, s, t, NULL, i)
#define rwsem_acquire_read(l,s,t,i) lock_acquire_shared(l, s, t, NULL, i)
```

---

### مفهوم الـ "Lock Class"

ده أهم concept في lockdep. مش كل **instance** من mutex عنده record منفصل — لأن الـ kernel ممكن يخلق مليون mutex — بدل كده، كل الـ mutexes اللي اتعملت بنفس الطريقة (من نفس الكود) بتنتمي لنفس **class**، واللي بنراقبه هو العلاقات بين الـ classes مش الـ instances.

```
mutex A (instance 1) ─────┐
mutex A (instance 2) ─────┤──► lock_class "A-class" ──► dependency graph
mutex A (instance 3) ─────┘
```

---

### الـ CONFIG options

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_LOCKDEP` | يشغّل كل الـ lockdep infrastructure |
| `CONFIG_PROVE_LOCKING` | يشغّل الـ deadlock/ordering checking |
| `CONFIG_LOCK_STAT` | يجمع statistics عن الـ lock contention |
| `CONFIG_PROVE_RAW_LOCK_NESTING` | يتحقق من raw lock nesting في PREEMPT_RT |
| `CONFIG_DEBUG_LOCKING_API_SELFTESTS` | يشغّل self-tests للتحقق من صحة lockdep نفسه |

لما `CONFIG_LOCKDEP` مش enabled، كل الـ macros تتحول لـ `do {} while (0)` — يعني **zero overhead** في الـ production builds.

---

### مثال واقعي: كيف بيشتغل lockdep مع mutex

```c
/* لما تعمل: */
mutex_lock(&my_mutex);

/* الـ kernel بينفذ داخلياً: */
mutex_acquire(&my_mutex.dep_map, 0, 0, _RET_IP_);
    └─► lock_acquire(dep_map, subclass=0, trylock=0,
                     read=0, check=1, nest_lock=NULL, ip)
        └─► lockdep validates:
            • هل في cycle في الـ dependency graph؟
            • هل الـ IRQ state صح؟
            • هل الـ context مناسب للـ lock type ده؟
```

---

### الـ Assert Macros — للـ Documentation والـ Safety

```c
/* بتقول: "عند النقطة دي، المفروض الـ lock ده ماسكه" */
lockdep_assert_held(&my_lock);

/* بتقول: "المفروض الـ IRQs enabled دلوقتي" */
lockdep_assert_irqs_enabled();

/* بتقول: "المفروض preemption enabled" */
lockdep_assert_preemption_enabled();
```

دول مش بس للـ debugging — دول **documentation living in code** بيقولوا للقاري إيه المتطلبات.

---

### الـ Lock Pinning

```c
/* لما عايز تضمن إن lock مش هيتحرر في الوقت الغلط */
struct pin_cookie cookie = lockdep_pin_lock(&my_lock);
/* ... */
lockdep_unpin_lock(&my_lock, cookie);
```

---

### ليه ده مهم جداً؟

بدون lockdep، الـ deadlocks في الـ kernel كانت هتتكتشف بس لما تحصل فعلاً — يعني في production على server حقيقي. مع lockdep، أي developer بيشغّل kernel بـ `CONFIG_PROVE_LOCKING=y` في testing وبيعمل أي عملية، lockdep بيفحص كل الـ lock paths في real-time ويصرخ قبل ما المشكلة تحصل.

الـ overhead كبير (بطىء الـ system 10-30x في testing)، لكن ده acceptable في development/testing وـ zero في production.

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/lockdep.h` | الـ public API — الملف ده |
| `include/linux/lockdep_types.h` | الـ core types: `lock_class`, `lockdep_map`, `held_lock` |
| `kernel/locking/lockdep.c` | التطبيق الكامل — آلاف الأسطر، قلب النظام |
| `kernel/locking/lockdep_internals.h` | internal helpers للـ lockdep.c نفسه |
| `kernel/locking/lockdep_states.h` | تعريفات حالات الـ IRQ/softirq |
| `kernel/locking/lockdep_proc.c` | الـ `/proc/lockdep*` interface لعرض الـ stats |
| `include/linux/spinlock.h` | بيستخدم `spin_acquire/release` macros |
| `include/linux/mutex.h` | بيستخدم `mutex_acquire/release` macros |
| `include/linux/rwsem.h` | بيستخدم `rwsem_acquire*` macros |
| `Documentation/locking/lockdep-design.rst` | الشرح النظري الكامل للـ algorithm |

---

### ملخص بسيط

**الـ `lockdep.h`** هو بوابة نظام "المفتش" بتاع الـ locks في الـ Linux kernel. بدل ما تستنى deadlock يحصل، lockdep بيبني **خريطة** لكل العلاقات بين الـ locks وبيدور على أي دايرة أو تناقض **في real-time أثناء التطوير والاختبار**. الـ header ده بيعرّف الـ API اللي بيربط كل أنواع الـ locks (spinlock, mutex, rwsem, ...) بنظام المراقبة ده.
## Phase 2: شرح الـ Lockdep Framework

### المشكلة — ليه Lockdep موجود أصلاً؟

في kernel، عندك آلاف الـ locks منتشرة في كل subsystem. الـ **deadlock** مش لازم يحصل دايماً — ممكن يحصل بس في race condition نادرة جداً، زي لما CPU1 يشيل lock A وبعدين lock B، بينما CPU2 بيشيل lock B وبعدين lock A. الاتنين بيستنوا بعض — deadlock. المشكلة إن ده ممكن ميحصلش إلا بعد شهور من production.

**الأسوأ من كده**: مش بس deadlock، في أيضاً:
- **lock order violations**: spinlock بتتاخد جوا hardirq context بس في مكان تاني بتتاخد في process context — ده يعمل deadlock لو الـ IRQ جه وسط الـ lock.
- **recursive locks**: thread بياخد lock هو شايله بالفعل (في locks مش recursive-safe).
- **lock held across sleep**: بتشيل spinlock وبتعمل `schedule()` — كارثة.

**المشكلة الجوهرية**: الـ testing العادي مش كفاية. الـ deadlock بيظهر تحت ظروف معينة من الـ timing. Lockdep بيحل المشكلة بطريقة مختلفة: **بدل ما تستنى الـ deadlock يحصل، يثبت رياضياً إن الـ lock ordering ممكن يؤدي لـ deadlock**.

---

### الحل — الـ Approach الجوهري

**الـ Lockdep** بيشتغل كـ **runtime graph-based dependency validator**. الفكرة:

1. كل ما lock A اتاخدت وبعدين lock B اتاخدت (يعني B اتاخدت وهي A شايلها)، Lockdep بيسجل edge في dependency graph: `A → B`.
2. كل ما lock جديدة بتتاخد، بيعمل **BFS/DFS** في الـ graph يشوف: هل في cycle؟ يعني هل ممكن يوصل من الـ lock الجديدة دي لـ lock شايلاها بالفعل؟ لو آه — **POTENTIAL DEADLOCK** ويطبع warning فوراً.
3. بيتتبع برضو الـ **IRQ context**: شيل lock في hardirq context مختلف عن شيلها في process context.

الـ key insight: Lockdep مش بيشتغل على instances — بيشتغل على **classes**. كل الـ spinlocks اللي instantiated من نفس الـ static definition هي نفس الـ class. ده بيخليه يكتشف violations من أول تشغيل، مش بس لما نفس الـ instances بالذات بيتقابلوا.

---

### تشبيه واقعي — المطعم والطاولات

تخيل مطعم كبير فيه **قواعد ترتيب**: عشان تاخد طاولة VIP (B) لازم تحجز أولاً مكتب الاستقبال (A). دايماً الترتيب: A ثم B.

لو في يوم واحد جاء حد وقلب الترتيب — حجز B الأول وبعدين طلب A — ده مش بالضرورة مشكلة دلوقتي لو مفيش حد تاني شايل A. بس لو تكرر الموضوع ده مع customers تانيين اللي بيشيلوا A وبيستنوا B، هتلاقي deadlock.

**Lockdep بيعمل إيه؟** زي إنك تعين "مدير نظام" يكتب كل طلب: "حجز A ثم B". لو جاء حد يعمل "حجز B ثم A"، المدير فوراً يصرخ: "مستحيل! في الكتب عندنا إن A دايما قبل B — ده cycle محتمل!"

**Map الـ analogy للـ kernel**:

| الـ Analogy | الـ Kernel Concept |
|---|---|
| الطاولة/المكتب | الـ lock instance (مثلاً `spinlock_t`) |
| نوع الطاولة (VIP/عادي) | الـ `lock_class` — كل locks من نفس النوع |
| قاعدة "A قبل B" | الـ edge في dependency graph |
| المدير بتاع الكتب | الـ lockdep validator نفسه |
| صرخة المدير | الـ `WARN_ON` + stack trace |
| الكتب نفسها | الـ `lock_list` + `lock_chain` structures |
| customer بياخد طاولتين | الـ task بتشيل أكتر من lock (`held_lock` array) |

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                                  │
├─────────────────────────────────────────────────────────────────────┤
│                    LOCK CONSUMERS (Drivers/Subsystems)               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │spinlock_t│  │ mutex_t  │  │rwlock_t  │  │ rwsem_t / seqlock│   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘   │
│       │              │              │                  │              │
│       │  spin_acquire│  mutex_acquire  rwlock_acquire  rwsem_acquire │
│       └──────────────┴──────────────┴──────────────────┘            │
│                              │                                        │
│                    ┌─────────▼──────────┐                            │
│                    │  lock_acquire()    │  ← single entry point      │
│                    │  lock_release()    │                             │
│                    └─────────┬──────────┘                            │
│                              │                                        │
├──────────────────────────────┼──────────────────────────────────────┤
│              LOCKDEP CORE ENGINE                                     │
│                              │                                        │
│  ┌───────────────────────────▼──────────────────────────────────┐   │
│  │                    lockdep_map                                │   │
│  │   key ──► lock_class_key  (identity of lock class)           │   │
│  │   class_cache[2]          (fast lookup cache)                 │   │
│  │   wait_type_inner/outer   (IRQ context constraints)           │   │
│  │   lock_type               (normal/percpu/override)            │   │
│  └───────────────────────────┬──────────────────────────────────┘   │
│                              │                                        │
│  ┌───────────────────────────▼──────────────────────────────────┐   │
│  │                     lock_class                                │   │
│  │   hash_entry          ← in global hash table                  │   │
│  │   locks_after  ──────────────────────────────┐               │   │
│  │   locks_before ◄─────────────────────────────┼──┐            │   │
│  │   usage_mask          (IRQ state bitmap)      │  │            │   │
│  │   wait_type_inner/outer                       │  │            │   │
│  └───────────────────────────────────────────────┼──┘            │   │
│                                                   │                   │
│  ┌────────────────────────────────────────────────▼─────────────┐   │
│  │                      lock_list                                │   │
│  │   class ──► (this node's class)                              │   │
│  │   links_to ──► (the class this edge points to)               │   │
│  │   dep        (dependency type bitmask)                        │   │
│  │   distance   (BFS distance from head)                         │   │
│  │   parent ──► (BFS backtrack pointer)                          │   │
│  └───────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌────────────────────────────┐  ┌───────────────────────────────┐  │
│  │         lock_chain         │  │          held_lock             │  │
│  │  chain_key (64-bit hash)   │  │  prev_chain_key               │  │
│  │  depth (locks in chain)    │  │  instance ──► lockdep_map     │  │
│  │  base  (index in array)    │  │  irq_context (hard/soft)      │  │
│  │  irq_context               │  │  read/trylock/check           │  │
│  └────────────────────────────┘  │  pin_count                    │  │
│                                   └───────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────────┤
│               PER-TASK STATE (in task_struct)                        │
│   held_locks[48]    ← stack of currently held locks                 │
│   lockdep_depth     ← current depth                                 │
│   lockdep_recursion ← guard against re-entry                        │
│   curr_chain_key    ← running hash of held lock chain               │
└─────────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

**الـ lock_class**: ده قلب الموضوع. Lockdep مش بيهتم بـ instances — بيهتم بـ **classes**. كل الـ spinlocks اللي declared في نفس الكود (أو بنفس الـ `DEFINE_SPINLOCK` في نفس الـ scope) هي نفس الـ class. الـ class هي "نوع" الـ lock مش "الـ lock نفسها".

**ليه ده مهم؟** لأن الـ deadlock بيتم اكتشافه من أول مرة في الـ system تتشيل lock من class A وبعدين lock من class B. مش محتاج تستنى نفس الـ instance بالذات تعمل المشكلة.

```c
// هنا LOCK_A وLOCK_B هما instances، بس lockdep بيعاملهم كـ class واحدة
// لأن متعرفوش بنفس الـ static definition
static DEFINE_SPINLOCK(lock_a_instance1);
static DEFINE_SPINLOCK(lock_a_instance2);
// الاتنين دول نفس الـ class في نظر lockdep!
```

---

### الـ Key Data Structures بالتفصيل

#### 1. `lock_class_key` و `lockdep_subclass_key`

```c
struct lockdep_subclass_key {
    char __one_byte;
} __attribute__ ((__packed__));

struct lock_class_key {
    union {
        struct hlist_node        hash_entry;     // for dynamic keys
        struct lockdep_subclass_key subkeys[8];  // 8 subclasses max
    };
};
```

**الـ key** هي الـ identity. كل lock type عنده key واحدة. الـ `subkeys` بتسمح بـ **subclasses**: نفس الـ lock type بس في contexts مختلفة. مثلاً `rq->lock` بتتشيل على مستويين — single lock وdouble lock — فالـ subclass بيفصلهم.

الـ `subkeys[MAX_LOCKDEP_SUBCLASSES]` ذكي جداً: الـ array نفسها جوا الـ struct بتديك 8 addresses مختلفة (`&subkeys[0]`, `&subkeys[1]`, ...) وكل واحدة منهم هي الـ key للـ subclass المقابلة. مش في allocation إضافية.

#### 2. `lock_class` — نود في الـ Dependency Graph

```c
struct lock_class {
    struct hlist_node   hash_entry;      // في global hash table
    struct list_head    lock_entry;      // في all_lock_classes list
    struct list_head    locks_after;     // locks اتشالت بعد الـ class دي
    struct list_head    locks_before;    // locks اتشالت قبل الـ class دي
    const struct lockdep_subclass_key *key;
    unsigned int        subclass;
    unsigned long       usage_mask;      // بيتم شيلها في IRQ/softirq؟
    const struct lock_trace *usage_traces[LOCK_TRACE_STATES];
    u8                  wait_type_inner; // LD_WAIT_SPIN/SLEEP/etc
    u8                  wait_type_outer;
    u8                  lock_type;       // normal/percpu/override
};
```

الـ `usage_mask` مهم جداً. بيسجل في أنهي contexts اتشالت الـ lock دي:
- اتشالت وهي hardirqs مفعّلة؟
- اتشالت وهي hardirqs معطّلة؟
- اتشالت في softirq context؟

ده بيسمح لـ lockdep يكتشف: "الـ lock دي بتتشيل في process context وفي IRQ context — ده dangerous لأن لو process شايلها وجاء IRQ يحاول ياخدها → deadlock".

#### 3. `lock_list` — الـ Edges في الـ Graph

```c
struct lock_list {
    struct list_head    entry;      // جزء من locks_after أو locks_before
    struct lock_class   *class;     // الـ class اللي الـ edge خارجة منها
    struct lock_class   *links_to;  // الـ class اللي الـ edge رايحة ليها
    const struct lock_trace *trace; // stacktrace لما الـ dependency اتسجلت
    u16                 distance;   // BFS distance
    u8                  dep;        // نوع الـ dependency (bitmask)
    u8                  only_xr;   // BFS flag
    struct lock_list    *parent;    // BFS backtrack pointer
};
```

**الـ dep bitmask** بيفرق بين أنواع الـ dependencies:
- Exclusive acquire بعد exclusive acquire
- Read acquire بعد exclusive acquire
- وغيرها

ده مهم لأن read-read dependency مش نفسها exclusive-exclusive.

#### 4. `lockdep_map` — الـ Handle الملصق في كل Lock

```c
struct lockdep_map {
    struct lock_class_key   *key;                           // مين الـ class؟
    struct lock_class       *class_cache[2];                // fast lookup
    const char              *name;
    u8                      wait_type_outer;
    u8                      wait_type_inner;
    u8                      lock_type;
};
```

ده الـ struct اللي بيتـ embed جوا كل `spinlock_t`, `mutex`, `rwsem` تحت اسم `dep_map`. هو الـ bridge بين الـ lock instance والـ lockdep engine.

الـ `class_cache[2]` تحسين للأداء: بدل ما كل مرة يعمل hash table lookup يلاقي الـ `lock_class`، بيcache الـ pointer. بس لما بتعمل `lockdep_copy_map()` لازم تمسح الـ cache لأن الـ copy ممكن تتشال في thread تاني ويحصل race على الـ cache.

#### 5. `held_lock` — Stack Entry في الـ Task

```c
struct held_lock {
    u64              prev_chain_key;  // hash الـ chain قبل الـ lock دي
    unsigned long    acquire_ip;      // عنوان الكود اللي شال الـ lock
    struct lockdep_map *instance;     // الـ lock الفعلية
    struct lockdep_map *nest_lock;    // lock برانية لو في nesting
    unsigned int     class_idx:13;   // index في lock_classes array
    unsigned int     irq_context:2;  // 0=process, 1=softirq, 2=hardirq
    unsigned int     trylock:1;
    unsigned int     read:2;
    unsigned int     check:1;
    unsigned int     hardirqs_off:1;
    unsigned int     references:11;
    unsigned int     pin_count;
};
```

كل task عندها `held_locks[48]` — stack الـ locks اللي شايلها دلوقتي. الـ `irq_context` بيفرق بين الـ lock اتشالت في process context ولا في interrupt context — ده مهم لـ chain hashing لأن الـ chains بتبدأ من الصفر لو دخلنا interrupt context.

#### 6. `lock_chain` — الـ Chain Cache

```c
struct lock_chain {
    unsigned int    irq_context : 2;
    unsigned int    depth       : 6;   // max 64 locks in chain
    unsigned int    base        : 24;  // index in chain_hlocks array
    struct hlist_node entry;           // في hash table
    u64             chain_key;         // unique hash of the chain
};
```

الـ chain هي "sequence الـ locks اللي شايلها الـ task في الـ context ده". الـ `chain_key` هو rolling hash يتبني خطوة خطوة. لو شيلت A ثم B ثم C، الـ chain_key = hash(hash(hash(A), B), C). لو نفس الـ chain اتشافت من قبل (cache hit) — مش محتاج تعمل BFS تاني.

---

### Struct Relationships Diagram

```
task_struct
├── held_locks[48]          ← per-task lock stack
│   └── held_lock
│       ├── instance ──────────────────► lockdep_map
│       │                                 ├── key ──────────► lock_class_key
│       │                                 │                    └── subkeys[8]
│       │                                 └── class_cache[2] ─► lock_class
│       └── prev_chain_key                                      ├── locks_after
│                                                               │   └── lock_list ──► lock_class (B)
├── curr_chain_key                                              │       └── lock_list ──► lock_class (C)
├── lockdep_depth                                               └── locks_before
└── lockdep_recursion                                               └── lock_list ──► lock_class (A)

Global State:
lock_classes[]      ← flat array of all known lock_class objects
lock_class_hash[]   ← hash table: key → lock_class
chain_hlocks[]      ← flat array storing chain contents
lock_chains[]       ← hash table: chain_key → lock_chain
```

---

### الـ Wait Types — IRQ Context Tracking

**الـ `lockdep_wait_type`** enum بيحدد إيه الـ waiting semantics الـ lock بتقدمها:

```
LD_WAIT_FREE    ← مفيش wait (atomic, RCU-like)
LD_WAIT_SPIN    ← spin-wait (raw_spinlock_t)
LD_WAIT_CONFIG  ← preemptible في PREEMPT_RT (spinlock_t)
LD_WAIT_SLEEP   ← sleeping lock (mutex, semaphore)
```

**القاعدة الذهبية**: مش مسموح تشيل lock بـ wait type أعلى جوا context شايل lock بـ wait type أقل. يعني:
- ممكن تشيل `LD_WAIT_SPIN` جوا `LD_WAIT_SPIN` context ✓
- ممكن تشيل `LD_WAIT_SPIN` جوا `LD_WAIT_SLEEP` context ✓
- **مش** مسموح تشيل `LD_WAIT_SLEEP` (mutex) وانت شايل `LD_WAIT_SPIN` (spinlock) ✗

الـ `wait_type_inner` بتقول "الـ lock دي بتعمل إيه جواها (تنام؟ تسبين؟)". الـ `wait_type_outer` بتقول "الـ lock دي ممكن تتشال في أنهي context من الخارج".

---

### الـ Recursion Guard — `lockdep_recursion`

```c
#define LOCKDEP_RECURSION_BITS  16
#define LOCKDEP_OFF             (1U << LOCKDEP_RECURSION_BITS)
#define LOCKDEP_RECURSION_MASK  (LOCKDEP_OFF - 1)

#define lockdep_off()   current->lockdep_recursion += LOCKDEP_OFF
#define lockdep_on()    current->lockdep_recursion -= LOCKDEP_OFF
```

الـ counter بيتقسم لجزئين:
- **الـ upper 16 bits**: عدد مرات `lockdep_off()` — لو > 0 يعني lockdep disabled
- **الـ lower 16 bits**: recursion depth الفعلية

الفصل ده بيخلي lock_acquire نفسها لما بتشتغل جوا lockdep engine مش تسبب recursion loop. لأن lockdep engine نفسه بياخد locks داخلية — ولو كل acquire بيشغل lockdep → infinite loop.

---

### IRQ State Assertions

```c
// per-CPU variables للـ IRQ state
DECLARE_PER_CPU(int, hardirqs_enabled);
DECLARE_PER_CPU(int, hardirq_context);
DECLARE_PER_CPU(unsigned int, lockdep_recursion);

#define lockdep_assert_irqs_enabled()   \
    WARN_ON_ONCE(__lockdep_enabled && !this_cpu_read(hardirqs_enabled))

#define lockdep_assert_irqs_disabled()  \
    WARN_ON_ONCE(__lockdep_enabled && this_cpu_read(hardirqs_enabled))
```

دول macros بيسمحوا لـ code يassert الـ IRQ state المتوقعة. لو function متوقعة تشتغل مع IRQs disabled وفي واحدة enabled → WARN فوراً. ده مختلف عن الـ deadlock detection — ده context validation.

---

### LOCK_STAT Subsystem — قياس الأداء

```c
#define LOCK_CONTENDED(_lock, try, lock)        \
do {                                            \
    if (!try(_lock)) {                          \
        lock_contended(&(_lock)->dep_map, _RET_IP_);  // سجل contention
        lock(_lock);                            \
    }                                           \
    lock_acquired(&(_lock)->dep_map, _RET_IP_); // سجل acquire
} while (0)
```

**الـ `CONFIG_LOCK_STAT`** هو subsystem منفصل داخل lockdep بيقيس:
- كم مرة الـ lock اتعمل عليها contention
- متوسط وقت الانتظار
- أطول وأقصر hold time

الـ stats دي متاحة عبر `/proc/lock_stat`.

---

### إيه الـ Lockdep بيمتلك؟ وإيه اللي بيفوضه للـ Drivers؟

| Lockdep يمتلك | يفوضه للـ Lock Implementations |
|---|---|
| الـ dependency graph (lock_class, lock_list) | الـ actual locking mechanism (cmpxchg, etc.) |
| الـ chain cache (lock_chain) | تحديد `wait_type_inner` و `wait_type_outer` |
| الـ per-task lock stack (held_lock) | تعريف الـ `lock_class_key` (DEFINE_SPINLOCK) |
| الـ BFS/DFS cycle detection | اختيار الـ subclass المناسب |
| الـ IRQ state tracking | تسمية الـ lock عشان تظهر في الـ error |
| إطفاء نفسه لو شاف error | إزاي يعمل trylock |
| الـ LOCK_STAT statistics | الـ lock/unlock الفعلي |

**الـ lock implementations** زي `spinlock.h`, `mutex.c` بتعمل `spin_acquire()` و `spin_release()` اللي هي مجرد macros بتنادي `lock_acquire()` و `lock_release()` — كل الـ intelligence في lockdep.

---

### الـ Pin Mechanism

```c
struct pin_cookie { unsigned int val; };

extern struct pin_cookie lock_pin_lock(struct lockdep_map *lock);
extern void lock_repin_lock(struct lockdep_map *lock, struct pin_cookie);
extern void lock_unpin_lock(struct lockdep_map *lock, struct pin_cookie);
```

الـ **pin** بيمنع lockdep يعدّ الـ lock released حتى لو الكود عمل `lock_release()`. ده useful في scenarios زي الـ `pagefault_disable()` اللي بيشيل lock conceptually بس الـ release ممكن يحصل في context تاني. الـ `pin_count` في `held_lock` بيعد كتير كده pins موجودة.

---

### الـ Novalidate والـ Notrack Classes

```c
#define lockdep_set_novalidate_class(lock) \
    lockdep_set_class_and_name(lock, &__lockdep_no_validate__, #lock)

#define lockdep_set_notrack_class(lock) \
    lockdep_set_class_and_name(lock, &__lockdep_no_track__, #lock)
```

- **`novalidate`**: lockdep لسه بيسجل الـ lock ويطبعها في dump، بس مش بيعمل ordering validation عليها.
- **`notrack`**: تجاهل تام — مش حتى بيسجلها. استُخدم في `bcachefs` اللي بياخد أكتر من 48 lock في نفس الوقت (الـ limit الافتراضي).

---

### Flow كامل — لما بتعمل `spin_lock(&my_lock)`

```
spin_lock(&my_lock)
    └── spin_acquire(&my_lock.dep_map, 0, 0, _RET_IP_)
        └── lock_acquire_exclusive(...)
            └── lock_acquire(&dep_map, subclass=0, trylock=0, read=0, check=1, NULL, ip)
                │
                ├── 1. check lockdep_recursion → لو set، exit فوراً
                ├── 2. lookup lock_class via dep_map.key (check cache first)
                ├── 3. check wait_type compatibility مع الـ context الحالي
                ├── 4. compute new chain_key = hash(curr_chain_key, class)
                ├── 5. lookup chain في chain cache
                │   ├── cache hit → skip BFS, just push to held_locks
                │   └── cache miss →
                │       ├── add new lock_list edges إلى graph
                │       └── run BFS: هل في cycle؟
                │           ├── cycle found → WARN + print deadlock report
                │           └── no cycle → add chain to cache
                ├── 6. push held_lock إلى task->held_locks[depth++]
                └── 7. update usage_mask لـ الـ lock_class
```

---

### Subsystem Dependencies

- **الـ RCU**: بيستخدم `LD_WAIT_FREE` — lockdep بيعرف إن الـ RCU read-side critical sections "wait-free" وبيتعامل معاها accordingly.
- **الـ per-CPU variables**: الـ `hardirqs_enabled`, `hardirq_context`, `lockdep_recursion` كلها per-CPU لأن الـ IRQ state بتختلف من CPU للتاني.
- **الـ stacktrace subsystem**: بيستخدمه lockdep يسجل `lock_trace` — الـ call stack لما الـ dependency اتسجلت أول مرة.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Flags والـ Enums والـ Config Options

#### `enum lockdep_wait_type` — نوع الانتظار المسموح بيه جوه السياق

| القيمة | القيمة العددية | المعنى |
|---|---|---|
| `LD_WAIT_INV` | 0 | غير محدد — catch-all، مش بيتفحص |
| `LD_WAIT_FREE` | 1 | wait-free زي RCU، مينفعش تنتظر هنا |
| `LD_WAIT_SPIN` | 2 | spin loop، `raw_spinlock_t` |
| `LD_WAIT_CONFIG` | 2 أو 3 | زي spin في non-RT، preemptible في PREEMPT_RT |
| `LD_WAIT_SLEEP` | 3 أو 4 | sleeping locks زي `mutex_t` |
| `LD_WAIT_MAX` | — | sentinel — لازم يفضل آخر واحد |

**الـ** `wait_type_inner` بيقول "أنا lock من النوع ده"، و`wait_type_outer` بيقول "مينفعش تأخذني إلا في سياق من النوع ده أو أعلى".

#### `enum lockdep_lock_type` — نوع الـ Lock

| القيمة | المعنى |
|---|---|
| `LD_LOCK_NORMAL` | عادي — الغالبية العظمى من الـ locks |
| `LD_LOCK_PERCPU` | per-CPU lock |
| `LD_LOCK_WAIT_OVERRIDE` | annotation فقط — بيغير السياق مش lock حقيقي |

#### `enum bounce_type` — (CONFIG_LOCK_STAT فقط) إحصائيات الـ Contention

| القيمة | المعنى |
|---|---|
| `bounce_acquired_write` | اتأخد write وكان في contention |
| `bounce_acquired_read` | اتأخد read وكان في contention |
| `bounce_contended_write` | كان في انتظار write |
| `bounce_contended_read` | كان في انتظار read |

#### `enum xhlock_context_t` — سياق الـ Cross-Hardware Lock

| القيمة | المعنى |
|---|---|
| `XHLOCK_HARD` | hardirq context |
| `XHLOCK_SOFT` | softirq context |

#### Constants المهمة

| الـ Macro | القيمة | المعنى |
|---|---|---|
| `MAX_LOCKDEP_SUBCLASSES` | 8 | أقصى عدد subclasses لـ lock class واحدة |
| `NR_LOCKDEP_CACHING_CLASSES` | 2 | عدد الـ classes المخزنة في `lockdep_map` كـ cache |
| `MAX_LOCKDEP_KEYS_BITS` | 13 | عدد bits لـ index الـ lock class |
| `MAX_LOCKDEP_KEYS` | 8192 | أقصى عدد lock classes في النظام كله |
| `LOCKSTAT_POINTS` | 4 | عدد نقاط الإحصاء لكل lock |
| `LOCK_TRACE_STATES` | 10 | عدد حالات الـ usage tracking (IRQ/softirq) |
| `LOCKDEP_RECURSION_BITS` | 16 | bits مخصصة للـ recursion counter |
| `LOCKDEP_OFF` | `1 << 16` | mask لتعطيل lockdep مؤقتاً |
| `SINGLE_DEPTH_NESTING` | 1 | subclass للـ single-level nesting |
| `INITIAL_CHAIN_KEY` | -1 | القيمة الابتدائية لـ hash chain |

#### قيم `read` في `lock_acquire()`

| القيمة | المعنى |
|---|---|
| 0 | exclusive (write) acquire |
| 1 | read-acquire — no recursion |
| 2 | read-acquire — recursion مسموح |
| -1 | any (في `lock_is_held_type()`) |

#### قيم `check` في `lock_acquire()`

| القيمة | المعنى |
|---|---|
| 0 | simple checks فقط (freeing, held-at-exit) |
| 1 | full validation |

#### `LOCK_STATE_*` — حالة الـ Lock

| الـ Macro | القيمة | المعنى |
|---|---|---|
| `LOCK_STATE_UNKNOWN` | -1 | غير معروف |
| `LOCK_STATE_NOT_HELD` | 0 | مش مأخود |
| `LOCK_STATE_HELD` | 1 | مأخود |

#### Config Options الرئيسية

| الـ Config | الأثر |
|---|---|
| `CONFIG_LOCKDEP` | تفعيل runtime locking validator كامل |
| `CONFIG_PROVE_LOCKING` | تفعيل full dependency checking (`might_lock`, `lockdep_assert_*`) |
| `CONFIG_LOCK_STAT` | تفعيل lock statistics (contention, wait times) |
| `CONFIG_PROVE_RAW_LOCK_NESTING` | التحقق من raw/non-raw lock nesting على PREEMPT_RT |
| `CONFIG_DEBUG_LOCKING_API_SELFTESTS` | تفعيل selftests + `force_read_lock_recursive` |

---

### 1. الـ Structs المهمة

#### `struct lockdep_subclass_key`

```c
struct lockdep_subclass_key {
    char __one_byte;
} __attribute__ ((__packed__));
```

**الغرض**: بتاخد عنوان unique في الـ binary لكل subclass — الـ address نفسه هو الـ key، مش المحتوى.

| الحقل | النوع | المعنى |
|---|---|---|
| `__one_byte` | `char` | byte وهمي — العنوان هو الـ key الحقيقي |

**الاتصال**: مضمونة جوه `lock_class_key` كـ array من 8 عناصر.

---

#### `struct lock_class_key`

```c
struct lock_class_key {
    union {
        struct hlist_node          hash_entry;
        struct lockdep_subclass_key subkeys[MAX_LOCKDEP_SUBCLASSES];
    };
};
```

**الغرض**: الـ "بصمة" الثابتة لكل نوع lock — بيتحدد statically في وقت compile أو dynamically.

| الحقل | النوع | المعنى |
|---|---|---|
| `hash_entry` | `hlist_node` | للـ dynamic keys — بيتسجل في hash table |
| `subkeys[8]` | `lockdep_subclass_key[]` | عناوين الـ subclasses الـ 8 |

**الاتصال**: الـ `lockdep_map` بيشاور عليها بـ pointer، وال `lock_class` بيخزن `*key`.

---

#### `struct lock_class`

```c
struct lock_class {
    struct hlist_node  hash_entry;      // في class hash table
    struct list_head   lock_entry;      // في all_lock_classes أو free list
    struct list_head   locks_after;     // dependency graph — forward edges
    struct list_head   locks_before;    // dependency graph — backward edges
    const struct lockdep_subclass_key *key;
    lock_cmp_fn        cmp_fn;
    lock_print_fn      print_fn;
    unsigned int       subclass;
    unsigned int       dep_gen_id;
    unsigned long      usage_mask;
    const struct lock_trace *usage_traces[LOCK_TRACE_STATES];
    const char         *name;
    int                name_version;
    u8                 wait_type_inner;
    u8                 wait_type_outer;
    u8                 lock_type;
    // CONFIG_LOCK_STAT:
    unsigned long      contention_point[4];
    unsigned long      contending_point[4];
} __no_randomize_layout;
```

**الغرض**: التمثيل الـ abstract لنوع الـ lock — مش الـ instance، بس النوع. كل الـ spinlocks اللي بتشاور على نفس الـ key بتتشارك class واحدة.

| الحقل | المعنى |
|---|---|
| `hash_entry` | entry في class hash table للبحث السريع |
| `lock_entry` | entry في القائمة العامة `all_lock_classes` |
| `locks_after` | list من `lock_list` — locks اتأخدت بعد الـ class دي |
| `locks_before` | list من `lock_list` — locks اتأخدت قبل الـ class دي |
| `key` | pointer لـ subclass key = المعرف الفريد |
| `usage_mask` | bitmask — إيه الـ contexts اللي اتأخد فيها (IRQ/softirq/read/write) |
| `usage_traces[]` | stack traces لكل usage state |
| `dep_gen_id` | generation ID للـ BFS traversal عشان مشيش على نفس الـ node تاني |
| `wait_type_inner` | نوع الانتظار اللي بيمثله الـ lock ده |
| `wait_type_outer` | الحد الأدنى المسموح للسياق اللي يأخده |

**الاتصال**: `lockdep_map` → `lock_class` (عن طريق `class_cache`)، و`lock_list` بيربط بين الـ classes.

---

#### `struct lockdep_map`

```c
struct lockdep_map {
    struct lock_class_key *key;
    struct lock_class     *class_cache[NR_LOCKDEP_CACHING_CLASSES]; // [2]
    const char            *name;
    u8                     wait_type_outer;
    u8                     wait_type_inner;
    u8                     lock_type;
    // CONFIG_LOCK_STAT:
    int                    cpu;
    unsigned long          ip;
};
```

**الغرض**: الـ "بطاقة هوية" اللي بتتضمن جوه كل lock instance (`spinlock_t`, `mutex`, إلخ). هي الواجهة الوحيدة اللي lockdep شايفها.

| الحقل | المعنى |
|---|---|
| `key` | مين أنا — pointer للـ lock class key |
| `class_cache[2]` | cache لـ 2 classes (subclass 0 و 1) لتجنب البحث في الـ hash |
| `name` | اسم نصي للـ lock (للـ debug messages) |
| `wait_type_inner` | نوع الانتظار اللي الـ lock ده بيمثله |
| `wait_type_outer` | السياق المطلوب لأخد الـ lock ده |
| `lock_type` | normal / percpu / wait_override |
| `cpu` + `ip` | (LOCK_STAT) آخر CPU وعنوان أخد فيه الـ lock |

**الاتصال**: مضمونة جوه كل أنواع الـ locks (`spinlock_t.rlock.dep_map`، إلخ). بتشاور على `lock_class_key` وبتعمل cache لـ `lock_class`.

---

#### `struct held_lock`

```c
struct held_lock {
    u64              prev_chain_key;    // hash السلسلة قبل الـ lock ده
    unsigned long    acquire_ip;        // عنوان الكود اللي أخد الـ lock
    struct lockdep_map *instance;       // الـ lock نفسه
    struct lockdep_map *nest_lock;      // الـ outer lock (للـ nesting)
    // CONFIG_LOCK_STAT:
    u64              waittime_stamp;
    u64              holdtime_stamp;
    // bitfields:
    unsigned int class_idx:13;          // index في lock_classes[]
    unsigned int irq_context:2;         // bit0=soft, bit1=hard
    unsigned int trylock:1;
    unsigned int read:2;
    unsigned int check:1;
    unsigned int hardirqs_off:1;
    unsigned int sync:1;
    unsigned int references:11;
    unsigned int pin_count;
};
```

**الغرض**: entry في الـ lock stack بتاع الـ task (بتخزن جوه `task_struct`). كل lock مأخود دلوقتي بيبقى فيه `held_lock`.

| الحقل | المعنى |
|---|---|
| `prev_chain_key` | الـ hash التراكمي للسلسلة من قبل الـ lock ده |
| `acquire_ip` | instruction pointer وقت الـ lock_acquire |
| `instance` | الـ `lockdep_map` للـ lock المأخود |
| `nest_lock` | لو في outer nest lock — للـ nesting validation |
| `class_idx` | index في global `lock_classes[]` array |
| `irq_context` | إيه السياق وقت الأخد (process=0، soft=1، hard=2، hardnmi=3) |
| `trylock` | اتأخد بـ trylock؟ |
| `read` | 0=exclusive، 1=shared، 2=recursive-shared |
| `references` | عدد مرات أخد نفس الـ lock recursively |
| `pin_count` | عدد مرات الـ pin — منع الـ release |

---

#### `struct lock_list`

```c
struct lock_list {
    struct list_head    entry;
    struct lock_class  *class;
    struct lock_class  *links_to;
    const struct lock_trace *trace;
    u16                 distance;
    u8                  dep;
    u8                  only_xr;
    struct lock_list   *parent;   // للـ BFS — bit0 = visited flag
};
```

**الغرض**: edge في dependency graph بين lock classes. كل edge = "الـ class دي اتأخدت وبعدين اتأخدت تانية".

| الحقل | المعنى |
|---|---|
| `class` | الـ class اللي بيجي منها الـ edge |
| `links_to` | الـ class اللي بيروح ليها الـ edge |
| `trace` | stack trace وقت تسجيل الـ dependency |
| `distance` | طول المسافة في الـ graph (للـ BFS) |
| `dep` | bitmask لأنواع الـ dependencies المختلفة |
| `only_xr` | هل المسار ده بس `-(*R)->` (read-only path)؟ |
| `parent` | الـ node السابق في BFS، bit0 = visited |

---

#### `struct lock_chain`

```c
struct lock_chain {
    unsigned int irq_context :  2;
    unsigned int depth       :  6;   // عدد الـ locks في السلسلة (max 64)
    unsigned int base        : 24;   // بداية السلسلة في chain_hlocks[]
    struct hlist_node entry;
    u64           chain_key;
};
```

**الغرض**: تمثيل مضغوط للـ dependency chain كاملة. بيتخزن في hash table عشان نتجنب re-validation لنفس السلسلة.

| الحقل | المعنى |
|---|---|
| `irq_context` | السياق (process/softirq/hardirq) |
| `depth` | عدد الـ held locks في الـ chain (max 63) |
| `base` | index في `chain_hlocks[]` global array |
| `chain_key` | hash الـ 64-bit للـ chain كامل |

---

#### `struct lock_time` و `struct lock_class_stats` (CONFIG_LOCK_STAT)

```c
struct lock_time {
    s64  min;    // أقل وقت
    s64  max;    // أكبر وقت
    s64  total;  // المجموع
    unsigned long nr; // العدد
};

struct lock_class_stats {
    unsigned long contention_point[4];   // أكتر 4 نقاط contention
    unsigned long contending_point[4];   // أكتر 4 نقاط contending
    struct lock_time read_waittime;
    struct lock_time write_waittime;
    struct lock_time read_holdtime;
    struct lock_time write_holdtime;
    unsigned long bounces[4];            // إحصائيات الـ cache bouncing
};
```

---

#### `struct pin_cookie`

```c
struct pin_cookie { unsigned int val; };
```

**الغرض**: token بيتعمل لما تعمل `lock_pin_lock()` — بيمنع الـ lock من الـ release غير المتوقع. بيتعمل `NIL_COOKIE` لما الـ lockdep معطل.

---

### 2. علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                    lock_class_key                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ union {                                             │   │
│  │   hash_entry (hlist_node)  ← dynamic keys          │   │
│  │   subkeys[8] (subclass_key) ← عناوين فريدة          │   │
│  │ }                                                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
         ▲                              ▲
         │ *key                         │ *key
         │                              │
┌─────────────────┐           ┌──────────────────────────┐
│   lockdep_map   │           │      lock_class           │
│  (embedded in   │           │  (abstract type node)     │
│  spinlock, etc) │           │                           │
│                 │           │ hash_entry ──→ class_hash │
│ *key ───────────┼──────────►│ lock_entry ──→ all_lock.. │
│ class_cache[2] ─┼──────────►│ locks_after ──┐           │
│ name            │           │ locks_before ─┼──┐        │
│ wait_type_inner │           │ usage_mask    │  │        │
│ wait_type_outer │           │ usage_traces[]│  │        │
└─────────────────┘           └───────────────┘  │        │
                                        │         │        │
                              ┌─────────▼─────────▼──────┐│
                              │       lock_list           ││
                              │  (directed graph edge)    ││
                              │                           ││
                              │ *class   ─────────────────┼┘
                              │ *links_to ────────────────┘
                              │ *trace
                              │ distance, dep, only_xr
                              │ *parent  (BFS traversal)
                              └───────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                   task_struct                              │
│                                                            │
│  held_locks[48] ─────────────────────────────────────────┐│
│  lockdep_depth                                            ││
│  lockdep_recursion                                        ││
│  curr_chain_key                                           ││
└────────────────────────────────────────────────────────────┘
                                                            │
                                         ┌──────────────────▼──┐
                                         │     held_lock        │
                                         │  (per acquired lock) │
                                         │                      │
                                         │ prev_chain_key       │
                                         │ *instance ──────────►│ lockdep_map
                                         │ *nest_lock ─────────►│ lockdep_map
                                         │ class_idx ──────────►│ lock_classes[]
                                         │ irq_context          │
                                         │ read, trylock        │
                                         │ pin_count            │
                                         └──────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                    lock_chain                              │
│  (cached dependency chain)                                 │
│                                                            │
│  chain_key ─────────────── unique 64-bit hash             │
│  base ──────────────────── index → chain_hlocks[]         │
│  depth ─────────────────── عدد الـ locks في السلسلة        │
│  entry ─────────────────── في chain_hash[]                │
└────────────────────────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### دورة حياة الـ Lock Class

```
COMPILE TIME / EARLY BOOT
         │
         ▼
  ┌─────────────────┐
  │  lock_class_key │  ← static variable in code (e.g., DEFINE_SPINLOCK)
  │  defined static │     أو dynamically allocated
  └────────┬────────┘
           │ lockdep_register_key() [dynamic only]
           ▼
  ┌─────────────────────────────────────┐
  │  lockdep_init_map_type()            │
  │  • ابحث عن class في hash table      │
  │  • لو مش موجودة → alloc جديدة من   │
  │    free_lock_classes pool           │
  │  • خزنها في class_hash[]            │
  │  • حطها في all_lock_classes list    │
  │  • خزن cache في lockdep_map         │
  └────────────────────────────────────┘
           │
           ▼
  ┌────────────────────────────────────┐
  │         IN USE                     │
  │  lock_acquire() → add to           │
  │    - held_lock stack في task        │
  │    - chain_hlocks[]                 │
  │    - update lock_chain hash         │
  │    - check for cycles (BFS)         │
  │    - update usage_mask              │
  └────────────────────────────────────┘
           │
           ▼ lock_release()
  ┌────────────────────────────────────┐
  │  pop من held_lock stack            │
  │  update chain_key في task           │
  └────────────────────────────────────┘
           │ [module unload / dynamic key]
           ▼
  ┌────────────────────────────────────┐
  │  lockdep_free_key_range() /        │
  │  lockdep_unregister_key()          │
  │  • إزالة من hash table             │
  │  • إضافة للـ zapped_classes list   │
  │  • تحرير بعد grace period          │
  └────────────────────────────────────┘
```

#### دورة حياة الـ held_lock (per task)

```
lock_acquire()
    │
    ▼
task->held_locks[task->lockdep_depth++]
    │  تملأ الـ held_lock:
    │    - instance = &lock->dep_map
    │    - class_idx = class index
    │    - prev_chain_key = task->curr_chain_key
    │    - acquire_ip = caller address
    │    - irq_context = current context
    │    - read/trylock/check flags
    │
    ▼ [hash chain key]
task->curr_chain_key = hash(prev_key, class_idx, irq_context)
    │
    ▼ [lookup in chain_hash]
 hit? ──YES──► skip validation (cache hit)
  │
 NO
  │
  ▼
full BFS dependency check
  │
  ▼
add new lock_chain to cache
    │
lock_release()
    │
    ▼
task->lockdep_depth--
task->curr_chain_key = held_locks[depth-1].prev_chain_key
```

---

### 4. Call Flow Diagrams

#### تدفق `lock_acquire()` الكامل

```
spinlock code calls spin_lock(lock)
  │
  ▼
spin_acquire(&lock->dep_map, subclass, trylock, ip)
  │
  ▼ expands to:
lock_acquire_exclusive(l, s, t, NULL, ip)
  │
  ▼ expands to:
lock_acquire(l, s, t, read=0, check=1, nest_lock=NULL, ip)
  │
  ├─► [if lockdep_recursion || !debug_locks] → return early (fast path)
  │
  ▼
lockdep_recursion++ (prevent recursion)
  │
  ▼
__lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip, ...)
  │
  ├─► lookup_or_create class()
  │     │
  │     ▼
  │   class_cache hit? → use it
  │   miss? → search classhash_table[]
  │          → not found? → register_lock_class()
  │                           → alloc from free_lock_classes
  │                           → add to hash + all_lock_classes
  │
  ├─► validate_chain()   ← الجزء الأهم
  │     │
  │     ▼
  │   compute new chain_key = hash(curr_key, class_idx, irq_context)
  │     │
  │     ▼
  │   lookup chain_hash[chain_key]
  │     │
  │     ├── HIT → chain already validated → return OK (fast)
  │     │
  │     └── MISS → check_deadlock()
  │                  │
  │                  ▼
  │                BFS on dependency graph
  │                  │
  │                  ├── cycle found? → print_circular_bug() → lockdep off
  │                  │
  │                  └── OK → add_chain_cache()
  │                            add new edges to lock_class.locks_after/before
  │
  ├─► check_irq_usage()
  │     → هل الـ lock اتأخد في IRQ context وكمان في process context؟
  │       → potential deadlock مع IRQ
  │
  ├─► mark_lock_irq() / mark_usage()
  │     → update lock_class.usage_mask
  │
  ▼
push held_lock onto task->held_locks[task->lockdep_depth++]
  │
lockdep_recursion--
```

#### تدفق `lock_release()`

```
spin_unlock(lock)
  │
  ▼
spin_release(&lock->dep_map, ip)
  │
  ▼
lock_release(&lock->dep_map, ip)
  │
  ▼
__lock_release(lock, ip)
  │
  ├─► find held_lock in task->held_locks[]
  │     (بيدور من الآخر — LIFO order expected)
  │
  ├─► check pin_count == 0 (مش مـ pinned)
  │
  ├─► decrement references (لو recursive lock)
  │     if references > 0 → return (still held)
  │
  ├─► remove from held_locks[] (shift down)
  │
  ▼
task->lockdep_depth--
restore task->curr_chain_key = prev held_lock's key
```

#### تدفق `lock_pin_lock()` / `lock_unpin_lock()`

```
lock_pin_lock(lockdep_map)
  │
  ▼
find held_lock for this map
  │
  ▼
held_lock->pin_count++
return pin_cookie { .val = current->pin_count_total }

lock_unpin_lock(lockdep_map, cookie)
  │
  ▼
validate cookie matches
  │
  ▼
held_lock->pin_count--
```

#### تدفق BFS لكشف الـ Deadlock

```
new dependency: A → B (تأخذ B وانت شايل A)
  │
  ▼
BFS من B للأمام (locks_after)
  │
  ├── زرت C, D, E ...
  │
  └── وصلت لـ A؟
        │
        YES → CYCLE DETECTED!
        │       A → B → ... → A = deadlock
        │       print_circular_bug()
        │       debug_locks = 0
        │
        NO  → safe, add edge A→B
```

---

### 5. استراتيجية الـ Locking في lockdep نفسه

lockdep بيحتاج يحمي datastructures الداخلية بتاعته، لكن مقدرش يستخدم locks عادية (هيتسبب في recursion). الحل:

#### الـ Locks المستخدمة داخلياً

```
lockdep internal data
        │
        ▼
┌──────────────────────────────────────────────────┐
│  graph_lock  (raw_spinlock_t)                    │
│  ← يحمي: lock_classes[], class hash table,       │
│           all_lock_classes list,                  │
│           lock_list edges (locks_after/before)    │
│           chain_hlocks[], chain_hash[]            │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│  lockdep_recursion (per-CPU counter)             │
│  ← يمنع lockdep من تتبع نفسه                    │
│    LOCKDEP_OFF bit = global disable              │
│    lower 16 bits = recursion depth               │
└──────────────────────────────────────────────────┘
```

#### ترتيب الـ Locking (Lock Ordering)

```
Rule: lockdep_recursion يتزيد قبل أي دخول لـ lockdep code
      ويتنقص بعد الخروج مباشرة

graph_lock
  └── يتأخد بـ raw_spin_lock_irqsave()
      (يعطل الـ IRQs عشان يحمي من IRQ context)

Task held_locks[] stack:
  └── protected by the fact that:
      - كل task بتعدل stack بتاعتها بس
      - lock/unlock operations are serialized per-task
      - لا يوجد sharing بين tasks
```

#### آليات منع الـ Recursion

```c
/* lockdep_off() — يعطل مؤقتاً */
current->lockdep_recursion += LOCKDEP_OFF;  // sets bit 16+

/* lockdep_on() — يعيد التشغيل */
current->lockdep_recursion -= LOCKDEP_OFF;

/* check في بداية كل acquire/release */
#define __lockdep_enabled (debug_locks && !this_cpu_read(lockdep_recursion))
```

#### الـ debug_locks Global Flag

```
debug_locks = 1   →   lockdep يعمل
debug_locks = 0   →   lockdep معطل تماماً (بعد أول bug)

debug_locks_off() → atomic xchg(&debug_locks, 0)
                    يطبع WARN مرة واحدة بس
                    بعدها كل شيء صامت
```

#### جدول ملخص — مين بيحمي إيه

| البيانات | الحماية |
|---|---|
| `lock_classes[]`, class hash | `graph_lock` (raw_spinlock + irqsave) |
| `all_lock_classes` list | `graph_lock` |
| `lock_list` edges | `graph_lock` |
| `chain_hlocks[]`, `chain_hash[]` | `graph_lock` |
| `task->held_locks[]` | per-task (no sharing) + `lockdep_recursion` |
| `task->lockdep_depth` | per-task |
| `task->curr_chain_key` | per-task |
| `lock_class.usage_mask` | `graph_lock` |
| `debug_locks` | atomic `xchg` |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Initialization

| Function / Macro | الغرض |
|---|---|
| `lockdep_init()` | تهيئة الـ lockdep subsystem عند boot |
| `lockdep_init_task(task)` | تهيئة الـ lockdep state في الـ task_struct |
| `lockdep_register_key(key)` | تسجيل dynamically-allocated lock key |
| `lockdep_unregister_key(key)` | إلغاء تسجيل key قبل تحرير الذاكرة |
| `lockdep_init_map_type(...)` | الـ core initializer لأي lock map |
| `lockdep_init_map_waits(...)` | wrapper مع outer wait type |
| `lockdep_init_map_wait(...)` | wrapper مع inner wait type فقط |
| `lockdep_init_map(...)` | أبسط wrapper للاستخدام العادي |
| `lockdep_copy_map(to, from)` | نسخ lockdep_map مع clear لـ class_cache |
| `STATIC_LOCKDEP_MAP_INIT(name, key)` | static initializer لـ lockdep_map |

#### Class Management

| Function / Macro | الغرض |
|---|---|
| `lockdep_set_class(lock, key)` | إعادة تعيين class للـ lock |
| `lockdep_set_class_and_name(lock, key, name)` | تعيين class مع اسم مخصص |
| `lockdep_set_class_and_subclass(lock, key, sub)` | تعيين class مع subclass |
| `lockdep_set_subclass(lock, sub)` | تغيير الـ subclass فقط |
| `lockdep_set_novalidate_class(lock)` | تعطيل ordering checks مع الإبقاء على التتبع |
| `lockdep_set_notrack_class(lock)` | تعطيل التتبع بالكامل (bcachefs) |
| `lock_set_class(lock, name, key, sub, ip)` | runtime class change |
| `lock_set_subclass(lock, sub, ip)` | runtime subclass change |
| `lockdep_match_class(lock, key)` | مقارنة class عبر pointer |
| `lockdep_match_key(lock, key)` | مقارنة key مباشرة |

#### Acquire / Release / Sync

| Function / Macro | الغرض |
|---|---|
| `lock_acquire(lock, sub, trylock, read, check, nest, ip)` | الـ core acquire event |
| `lock_release(lock, ip)` | الـ core release event |
| `lock_sync(lock, sub, read, check, nest, ip)` | sync point بدون acquire/release |
| `lock_downgrade(lock, ip)` | تحويل write lock إلى read lock |
| `lock_acquire_exclusive(l, s, t, n, i)` | acquire write/exclusive |
| `lock_acquire_shared(l, s, t, n, i)` | acquire read/shared |
| `lock_acquire_shared_recursive(l, s, t, n, i)` | acquire read مع recursion |

#### Lock-Type-Specific Wrappers

| Macro | يُستخدم مع |
|---|---|
| `spin_acquire / spin_release` | `spinlock_t` |
| `rwlock_acquire / rwlock_acquire_read / rwlock_release` | `rwlock_t` |
| `mutex_acquire / mutex_acquire_nest / mutex_release` | `mutex_t` |
| `rwsem_acquire / rwsem_acquire_read / rwsem_release` | `rw_semaphore` |
| `seqcount_acquire / seqcount_acquire_read / seqcount_release` | `seqcount_t` |
| `lock_map_acquire / lock_map_release / lock_map_sync` | generic `lockdep_map` |

#### Query & Assertions

| Function / Macro | الغرض |
|---|---|
| `lock_is_held_type(lock, read)` | هل الـ lock محجوز؟ (مع نوع) |
| `lock_is_held(lock)` | هل محجوز بأي نوع؟ |
| `lockdep_is_held(lock)` | wrapper على dep_map |
| `lockdep_is_held_type(lock, r)` | wrapper مع type |
| `lockdep_depth(tsk)` | عدد الـ locks المحجوزة حالياً |
| `lockdep_recursing(tsk)` | هل lockdep recursing? |
| `lockdep_assert_held(l)` | panic لو الـ lock مش محجوز |
| `lockdep_assert_not_held(l)` | panic لو الـ lock محجوز |
| `lockdep_assert_held_write(l)` | يأكد exclusive hold |
| `lockdep_assert_held_read(l)` | يأكد shared hold |
| `lockdep_assert_held_once(l)` | once-variant |
| `lockdep_assert_none_held_once()` | يأكد إن ما فيش locks محجوزة |
| `lockdep_assert(cond)` | generic assertion |
| `lockdep_assert_once(cond)` | WARN_ON_ONCE variant |

#### IRQ / Preemption Context Assertions

| Macro | يفحص |
|---|---|
| `lockdep_assert_irqs_enabled()` | hardirqs مفتوحة |
| `lockdep_assert_irqs_disabled()` | hardirqs مغلقة |
| `lockdep_assert_in_irq()` | داخل hardirq context |
| `lockdep_assert_no_hardirq()` | مش في hardirq وإن hardirqs enabled |
| `lockdep_assert_preemption_enabled()` | preemption مفتوحة |
| `lockdep_assert_preemption_disabled()` | preemption مغلقة |
| `lockdep_assert_in_softirq()` | داخل softirq فقط |
| `lockdep_assert_RT_in_threaded_ctx()` | PREEMPT_RT: hardirq threaded |

#### Pin API

| Function / Macro | الغرض |
|---|---|
| `lock_pin_lock(lock)` | يمنع task migration مع tracking |
| `lock_repin_lock(lock, cookie)` | إعادة pin باستخدام cookie سابق |
| `lock_unpin_lock(lock, cookie)` | رفع الـ pin |
| `lockdep_pin_lock(l)` | wrapper على dep_map |
| `lockdep_repin_lock(l, c)` | wrapper |
| `lockdep_unpin_lock(l, c)` | wrapper |

#### Statistics & Debug

| Function / Macro | الغرض |
|---|---|
| `lock_contended(lock, ip)` | يسجل contention event |
| `lock_acquired(lock, ip)` | يسجل successful acquire بعد wait |
| `LOCK_CONTENDED(_lock, try, lock)` | macro يجمع try + contention record |
| `LOCK_CONTENDED_RETURN(_lock, try, lock)` | نفسه مع return value |
| `print_irqtrace_events(curr)` | طباعة IRQ trace للـ task |
| `lockdep_rcu_suspicious(file, line, s)` | إبلاغ عن RCU usage مريب |
| `lockdep_reset()` | reset كامل للـ state |
| `lockdep_reset_lock(lock)` | reset lock واحد فقط |
| `lockdep_free_key_range(start, size)` | تحرير range من الـ keys |
| `lockdep_sys_exit()` | يفحص held locks عند sys_exit |
| `lockdep_set_selftest_task(task)` | يحدد task للـ selftest |
| `lockdep_set_lock_cmp_fn(map, cmp, print)` | تعيين custom comparator |

---

### Group 1: Initialization & Registration

هذه المجموعة مسؤولة عن تهيئة الـ lockdep subsystem نفسه، وتسجيل الـ lock keys والـ maps. الهدف هو إعداد الـ metadata اللي يحتاجها الـ validator قبل أي lock operation.

---

#### `lockdep_init`

```c
extern void lockdep_init(void);
```

بتشغّل الـ lockdep subsystem أثناء الـ boot. بتهيئ الـ hash tables الخاصة بالـ lock classes والـ dependency chains. بتُستدعى مرة واحدة فقط من `start_kernel()`.

- **Parameters:** لا يوجد.
- **Return:** `void`.
- **Key details:** لازم تُستدعى قبل أي lock registration. بتشغل internal locks جوّاها.
- **Caller context:** Early boot، single-thread.

---

#### `lockdep_init_task`

```c
extern void lockdep_init_task(struct task_struct *task);
```

بتهيئ الـ lockdep-related fields في الـ `task_struct` الجديد (زي `lockdep_depth`، `curr_chain_key`، `held_locks[]`). كل task بيتولد بيمرّ منها.

- **Parameters:** `task` — الـ task الجديد.
- **Return:** `void`.
- **Key details:** بتُستدعى من `copy_process()`. لو مش متهيئ صح، الـ validator هيبلّش بـ garbage state.
- **Caller context:** Process creation path، مش بحاجة لـ locks خاصة.

---

#### `lockdep_register_key`

```c
extern void lockdep_register_key(struct lock_class_key *key);
```

بتسجّل **dynamically-allocated** `lock_class_key` في الـ lockdep key hash table. الـ static keys (اللي بتتعرّف بـ `DEFINE_SPINLOCK` مثلاً) مش بتحتاجها لأن عنوانها في `.data` section يكون unique by definition.

- **Parameters:** `key` — pointer للـ key المخصصة ديناميكياً.
- **Return:** `void`.
- **Key details:** لو مش registered وحصل conflict، الـ lockdep هيفكّرهم نفس الـ class. بتأخذ internal `lockdep_lock` (raw_spinlock) جوّاها.
- **Caller context:** Module init أو device probe، يحتاج process context.

---

#### `lockdep_unregister_key`

```c
extern void lockdep_unregister_key(struct lock_class_key *key);
```

بتشيل الـ key من الـ hash table وبتعمل cleanup للـ lock classes المرتبطة بيها. **لازم تُستدعى قبل `kfree()` للـ key** وإلا هيبقى dangling pointer في الـ validator.

- **Parameters:** `key` — الـ key اللي اتسجّلت قبل كده.
- **Return:** `void`.
- **Key details:** بتعمل `lockdep_reset_lock()` لكل class بتستخدم الـ key ده. بتأخذ `lockdep_lock`. مش safe تستدعيها وفي threads لسه بتستخدم locks من الـ key ده.
- **Caller context:** Module exit أو device removal.

---

#### `lockdep_init_map_type`

```c
extern void lockdep_init_map_type(struct lockdep_map *lock, const char *name,
    struct lock_class_key *key, int subclass, u8 inner, u8 outer, u8 lock_type);
```

ده الـ **core initializer** لأي `lockdep_map`. بيربط الـ map بالـ class key، بيحفظ الاسم للـ debug output، وبيضبط الـ wait-type semantics. كل wrappers التانية بتوصل هنا في النهاية.

- **Parameters:**
  - `lock` — الـ `lockdep_map` المراد تهيئتها.
  - `name` — اسم نصي للـ lock (للـ debug فقط).
  - `key` — الـ `lock_class_key` اللي بتحدد الـ class.
  - `subclass` — الـ subclass index (0 = main class).
  - `inner` — `lockdep_wait_type` اللي بيقدّمه الـ lock (context داخله).
  - `outer` — الـ context اللي ممكن يتأخذ فيه الـ lock.
  - `lock_type` — `lockdep_lock_type` (NORMAL, PERCPU, WAIT_OVERRIDE).
- **Return:** `void`.
- **Key details:** بتعمل class lookup أو allocation في `lock_classes[]`. بتأخذ `graph_lock` لو محتاجة تضيف class جديدة. بتعمل clear للـ `class_cache[]` في الـ map.
- **Caller context:** Lock initialization time، يجوز في أي context قبل أول استخدام.

**Pseudocode flow:**
```
lockdep_init_map_type(lock, name, key, subclass, inner, outer, type):
    if key == NULL → use &__lockdep_no_validate__
    lock->key  = key
    lock->name = name
    lock->wait_type_inner = inner
    lock->wait_type_outer = outer
    lock->lock_type = type
    clear lock->class_cache[]
    if subclass != 0:
        register_lock_class(lock, subclass, 0)  // warm up cache
```

---

#### `lockdep_init_map_waits` / `lockdep_init_map_wait` / `lockdep_init_map`

```c
static inline void lockdep_init_map_waits(struct lockdep_map *lock, const char *name,
    struct lock_class_key *key, int subclass, u8 inner, u8 outer);

static inline void lockdep_init_map_wait(struct lockdep_map *lock, const char *name,
    struct lock_class_key *key, int subclass, u8 inner);

static inline void lockdep_init_map(struct lockdep_map *lock, const char *name,
    struct lock_class_key *key, int subclass);
```

ثلاث wrappers بتبسّط استخدام `lockdep_init_map_type`:
- **الـ `waits`**: بتحدد `inner` و`outer` معاً، بتستخدم `LD_LOCK_NORMAL`.
- **الـ `wait`**: بتحدد `inner` فقط، بتبعت `outer = LD_WAIT_INV`.
- **الـ `map`**: أبسطهم، بتبعت `inner = LD_WAIT_INV` كمان.

الـ `LD_WAIT_INV` معناه "مش بيُفحص"، وده الـ default للـ locks العادية اللي مش بتحتاج wait-type checking.

---

#### `lockdep_copy_map`

```c
static inline void lockdep_copy_map(struct lockdep_map *to,
                                    struct lockdep_map *from);
```

بتنسخ `lockdep_map` بالكامل، لكن بتعمل **clear للـ `class_cache[]`** عمداً. السبب إن الـ cache بيتعدّل concurrently، وعلى معمارية 64-bit مع 32-bit copy instructions ممكن يحصل half-pointer corruption.

- **Parameters:** `to` — الـ destination. `from` — الـ source.
- **Return:** `void`.
- **Key details:** بياخد performance hit بسبب cache miss، لكن correctness أهم. **ملحوظة**: مش بتشتغل صح مع `lockdep_set_class_and_subclass()` اللي بتعتمد على abuse الـ cache.
- **Caller context:** يُستخدم لما بيتعمل copy لـ embedded locks (زي copy لـ waitqueue).

---

### Group 2: Runtime Acquire / Release / Sync

ده قلب الـ lockdep. الـ functions دي هي اللي بتتفاعل مع الـ dependency graph وبتشوف لو فيه potential deadlock.

---

#### `lock_acquire`

```c
extern void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                         int trylock, int read, int check,
                         struct lockdep_map *nest_lock, unsigned long ip);
```

الـ **core annotation function** اللي بتقول للـ lockdep "الـ current task اتأخذ lock". بتعمل:
1. تسجيل الـ lock في `current->held_locks[]`.
2. فحص الـ dependency graph لو وُجد cycle (= potential deadlock).
3. تحديث الـ IRQ usage tracking.
4. إضافة edges جديدة للـ dependency graph لو محتاج.

- **Parameters:**
  - `lock` — الـ `lockdep_map` الخاص بالـ lock.
  - `subclass` — الـ subclass (0 في الغالب).
  - `trylock` — 1 لو trylock (فشل ما فيهوش deadlock).
  - `read` — 0=exclusive, 1=shared, 2=shared-recursive.
  - `check` — 0=basic checks, 1=full graph validation.
  - `nest_lock` — الـ lock اللي بيحمي الـ lock ده (لـ nested locks).
  - `ip` — الـ instruction pointer (للـ stack trace).
- **Return:** `void`.
- **Key details:** بتعمل `lockdep_off()` جوّاها عشان تمنع recursion. لو `debug_locks == 0`، بترجع فوراً. الـ graph walk بتستخدم BFS. **مش بتأخذ lock ليها**، الـ lock الحقيقي بيتأخذ بالـ architecture code.
- **Caller context:** يُستدعى من الـ `spin_lock()` وأخواتها بعد ما الـ lock الحقيقي يتأخذ.

**Pseudocode flow:**
```
lock_acquire(lock, subclass, trylock, read, check, nest_lock, ip):
    if !debug_locks || lockdep_recursion → return
    lockdep_recursion++
    hlock = &current->held_locks[current->lockdep_depth]
    fill hlock fields (class_idx, read, trylock, irq_context, ...)
    if check:
        validate_chain(current, lock, hlock, nest_lock, ip)
        // BFS on dependency graph looking for cycles
    add_lock_to_held_locks(current, hlock)
    trace_lock_acquired(lock, ip)
    lockdep_recursion--
```

---

#### `lock_release`

```c
extern void lock_release(struct lockdep_map *lock, unsigned long ip);
```

بتقول للـ lockdep إن الـ lock اتحرر. بتشيل الـ entry من `current->held_locks[]` وبتعمل update للـ chain key.

- **Parameters:**
  - `lock` — الـ `lockdep_map`.
  - `ip` — instruction pointer.
- **Return:** `void`.
- **Key details:** بتفحص إن الـ lock كان فعلاً محجوزاً من نفس الـ task (otherwise = BUG). بتعمل LIFO check على الـ held_locks stack. لو الـ lock اتحرر وهو مش في القمة (out-of-order release)، بيطلع warning.
- **Caller context:** يُستدعى بعد ما الـ real lock يتحرر.

---

#### `lock_sync`

```c
extern void lock_sync(struct lockdep_map *lock, unsigned int subclass,
                      int read, int check, struct lockdep_map *nest_lock,
                      unsigned long ip);
```

بتسجّل **sync point** من غير acquire/release. بتُستخدم لـ synchronization primitives زي `synchronize_rcu()` اللي مش بتمسك lock بس بتضمن ordering. بتضيف dependency edge في الـ graph بدون إضافة entry في `held_locks[]`.

- **Parameters:** نفس `lock_acquire` تقريباً بدون `trylock`.
- **Return:** `void`.
- **Key details:** مهمة لـ correctness الـ dependency graph. بتمنع false negatives في cases زي RCU synchronization.

---

#### `lock_downgrade`

```c
extern void lock_downgrade(struct lockdep_map *lock, unsigned long ip);
```

بتحوّل exclusive (write) lock إلى shared (read) lock في الـ lockdep tracking. مقابلتها الـ hardware op في `rwlock` أو `rwsem`.

- **Parameters:** `lock` — الـ map. `ip` — caller address.
- **Return:** `void`.
- **Key details:** بتعمل update لـ `held_lock->read` من 0 إلى 1 في `current->held_locks[]`. لو الـ lock مش exclusive أصلاً، بيطلع BUG.

---

### Group 3: Class Management

---

#### `lock_set_class`

```c
extern void lock_set_class(struct lockdep_map *lock, const char *name,
                           struct lock_class_key *key, unsigned int subclass,
                           unsigned long ip);
```

بتغير الـ class الخاصة بـ lock **وهو شغّال** (runtime reclassification). بتُستخدم لما بيكون عندك lock بيخدم أدوار مختلفة في contextsمختلفة وعايز تقسّمهم في الـ dependency graph.

- **Parameters:**
  - `lock` — الـ map المراد تغيير classها.
  - `name` — اسم جديد للـ debug.
  - `key` — الـ key الجديدة.
  - `subclass` — الـ subclass الجديد.
  - `ip` — caller IP للـ trace.
- **Return:** `void`.
- **Key details:** بتعمل release للـ old class ثم acquire للـ new class في الـ held_locks. بتأخذ `graph_lock` لو محتاجة تضيف class جديدة.

---

#### `lock_set_subclass`

```c
static inline void lock_set_subclass(struct lockdep_map *lock,
        unsigned int subclass, unsigned long ip);
```

Wrapper على `lock_set_class()` بس بيغير الـ subclass فقط مع الإبقاء على نفس الـ key و name. شائع في code زي `double_lock_bh()` اللي بياخذ نفس النوع من الـ lock مرتين.

---

#### `lockdep_set_class` و أخواتها (Macros)

```c
#define lockdep_set_class(lock, key)
#define lockdep_set_class_and_name(lock, key, name)
#define lockdep_set_class_and_subclass(lock, key, sub)
#define lockdep_set_subclass(lock, sub)
```

الـ macros دي بتستدعي `lockdep_init_map_type()` على الـ `dep_map` الموجود في الـ lock، مع الحفاظ على باقي الـ fields (wait types، lock type). بتُستخدم عند init الـ lock لو عايز class مختلفة عن الـ static address.

---

#### `lockdep_set_novalidate_class`

```c
#define lockdep_set_novalidate_class(lock) \
    lockdep_set_class_and_name(lock, &__lockdep_no_validate__, #lock)
```

بتعيّن class خاصة `__lockdep_no_validate__` اللي بتقول للـ lockdep "اسجّل إن الـ lock اتأخذ، لكن متفحصش الـ ordering". مفيد للـ locks اللي بترتيبها صح بـ design بس الـ validator مش قادر يثبت كده.

---

#### `lockdep_set_notrack_class`

```c
#define lockdep_set_notrack_class(lock) \
    lockdep_set_class_and_name(lock, &__lockdep_no_track__, #lock)
```

أقوى من `novalidate` — بتعطّل التتبع **بالكامل**. تُستخدم حالياً في `bcachefs` اللي بياخذ أكتر من 48 lock (الـ limit الافتراضي للـ lockdep held_locks array).

---

#### `lockdep_match_key`

```c
static inline int lockdep_match_key(struct lockdep_map *lock,
                                    struct lock_class_key *key);
```

بتقارن بين الـ `key` الموجودة في الـ lock والـ key المعطاة بـ pointer comparison مباشرة.

- **Return:** 1 لو match، 0 لو لأ.
- **Key details:** مش بتشتغل لو `!CONFIG_LOCKDEP` — الـ caller لازم يعمل `#ifdef` بنفسه.

---

### Group 4: Query API — هل الـ Lock محجوز؟

---

#### `lock_is_held_type`

```c
extern int lock_is_held_type(const struct lockdep_map *lock, int read);
```

بتفتّش في `current->held_locks[]` وبتشوف لو الـ lock المحدد موجود.

- **Parameters:**
  - `lock` — الـ map المراد التحقق منه.
  - `read` — -1=أي نوع، 0=exclusive فقط، 1=shared فقط.
- **Return:** `LOCK_STATE_HELD (1)`، `LOCK_STATE_NOT_HELD (0)`، أو `LOCK_STATE_UNKNOWN (-1)` لو `debug_locks` معطّل.
- **Key details:** بتمشي على الـ held_locks array بالكامل. O(n) حيث n = lockdep_depth. بتعمل `lockdep_off()` جوّاها.

---

#### `lock_is_held`

```c
static inline int lock_is_held(const struct lockdep_map *lock);
```

Wrapper على `lock_is_held_type(lock, -1)` — بتشوف أي نوع من الـ holding.

---

#### `lockdep_assert_held` و أخواتها

```c
#define lockdep_assert_held(l)
#define lockdep_assert_not_held(l)
#define lockdep_assert_held_write(l)
#define lockdep_assert_held_read(l)
#define lockdep_assert_held_once(l)
#define lockdep_assert_none_held_once()
```

بتستخدم `WARN_ON` أو `WARN_ON_ONCE` للـ assertion. لو `debug_locks == 0`، بتتجاهل الفحص. الـ `lockdep_assert_held` بتستدعي كمان `__assume_ctx_lock(l)` اللي بتتيح للـ static analyzer (Clang) إنه يعرف إن الـ lock محجوز في الـ code path ده.

**مثال:**
```c
void update_shared_data(struct my_struct *s)
{
    lockdep_assert_held(&s->lock);  /* يضمن إن المتصل حاجز الـ lock */
    s->data++;
}
```

---

### Group 5: IRQ & Preemption Context Assertions

هذه المجموعة من الـ macros بتفحص الـ execution context الحالي باستخدام per-CPU variables اللي بيحافظ عليها الـ lockdep نفسه.

الـ `__lockdep_enabled` macro هو guard مشترك:
```c
#define __lockdep_enabled  (debug_locks && !this_cpu_read(lockdep_recursion))
```
بيضمن إن الـ checks دي مش بتشتغل وإحنا جوّا الـ lockdep نفسه.

---

#### `lockdep_assert_irqs_enabled` / `lockdep_assert_irqs_disabled`

```c
#define lockdep_assert_irqs_enabled()
#define lockdep_assert_irqs_disabled()
```

بتقرأ per-CPU variable `hardirqs_enabled` اللي بيتحدّث من `trace_hardirqs_on/off()`. أدق من `irqs_disabled()` العادية لأنها بتتعامل مع الـ lockdep model للـ IRQ state مش الـ hardware state مباشرة.

---

#### `lockdep_assert_in_irq`

```c
#define lockdep_assert_in_irq()
```

بتتحقق إن `hardirq_context > 0`، يعني إحنا داخل hardirq handler. بتُستخدم في functions لازم تتعمل فقط من interrupt context.

---

#### `lockdep_assert_no_hardirq`

```c
#define lockdep_assert_no_hardirq()
```

بتتحقق من شرطين: مش في hardirq **و** hardirqs enabled. مفيد لـ code اللي بيحتاج process context مع IRQs مفتوحة.

---

#### `lockdep_assert_preemption_enabled` / `lockdep_assert_preemption_disabled`

```c
#define lockdep_assert_preemption_enabled()
#define lockdep_assert_preemption_disabled()
```

بتفحص `preempt_count() == 0` و`hardirqs_enabled` معاً. فقط active لو `CONFIG_PREEMPT_COUNT` معرّف.

---

#### `lockdep_assert_in_softirq`

```c
#define lockdep_assert_in_softirq()
```

بتتحقق من `in_softirq() && !in_hardirq() && !in_nmi()`. الـ semantics تبعتها ambiguous (زي ما النظام بيقول) — بتعمل assert إن إحنا في softirq context خالص.

---

#### `lockdep_assert_RT_in_threaded_ctx`

```c
#define lockdep_assert_RT_in_threaded_ctx()
```

خاص بـ `CONFIG_PROVE_RAW_LOCK_NESTING`. بيتأكد إن على `PREEMPT_RT` الكود اللي بيأخذ `raw_spinlock` من hardirq context بيشتغل في threaded IRQ context (عشان على RT يكون الـ hardirq threaded).

---

### Group 6: Pin API

**الـ pin** هو mechanism بيمنع الـ task من ما يتعمله migrate لـ CPU تاني وهو شايل lock. ده مهم للـ per-CPU locks.

---

#### `lock_pin_lock`

```c
extern struct pin_cookie lock_pin_lock(struct lockdep_map *lock);
```

بتثبّت الـ `pin_count` في الـ `held_lock` المقابل للـ lock، وبتُرجع `pin_cookie` اللي فيه الـ pin counter value.

- **Return:** `struct pin_cookie { unsigned int val; }` — بيُستخدم للـ verification عند الـ unpin.
- **Key details:** لو الـ lock مش في `held_locks[]`، بيرجع `NIL_COOKIE`. الـ cookie بيضمن إن الـ unpin بيتطابق مع الـ pin الصح.

---

#### `lock_repin_lock`

```c
extern void lock_repin_lock(struct lockdep_map *lock, struct pin_cookie);
```

بتعمل re-pin بعد ما الـ context اتغيّر. بتُستخدم في scenarios زي `schedule()` حيث ممكن الـ pin يحتاج تجديد.

---

#### `lock_unpin_lock`

```c
extern void lock_unpin_lock(struct lockdep_map *lock, struct pin_cookie);
```

بتفكّ الـ pin وبتتحقق إن الـ cookie المعطاة تطابق الـ pin الأصلي. لو مش متطابق، بيطلع BUG — ده protection ضد unpin خطأ.

---

### Group 7: Statistics — `CONFIG_LOCK_STAT`

---

#### `lock_contended`

```c
extern void lock_contended(struct lockdep_map *lock, unsigned long ip);
```

بتسجّل إن task حاول يأخذ الـ lock ولقاه مشغول. بتُضيف الـ `ip` في `lock_class->contention_point[]` array (ثابتة 4 entries).

- **Parameters:** `lock` — الـ map. `ip` — عنوان المتصل.
- **Return:** `void`.
- **Caller context:** بتُستدعى من `LOCK_CONTENDED` macro بعد ما يفشل الـ try.

---

#### `lock_acquired`

```c
extern void lock_acquired(struct lockdep_map *lock, unsigned long ip);
```

بتسجّل إن الـ lock اتأخذ بنجاح **بعد wait**. بتحسب الـ wait time وبتضيفها لـ `lock_class_stats`. بتُفرّق بين write وread acquire.

---

#### `LOCK_CONTENDED` / `LOCK_CONTENDED_RETURN`

```c
#define LOCK_CONTENDED(_lock, try, lock)        \
do {                                            \
    if (!try(_lock)) {                          \
        lock_contended(&(_lock)->dep_map, _RET_IP_); \
        lock(_lock);                            \
    }                                           \
    lock_acquired(&(_lock)->dep_map, _RET_IP_); \
} while (0)
```

بيجمع الـ try → contention tracking → actual lock في macro واحد. الـ `LOCK_CONTENDED_RETURN` نفسه بس بيرجع error code من الـ lock function.

**مثال استخدام:**
```c
/* من spin_lock_bh implementation */
LOCK_CONTENDED(&sem, down_read_trylock, down_read);
```

---

#### `print_irqtrace_events`

```c
extern void print_irqtrace_events(struct task_struct *curr);
```

بتطبع تاريخ الـ IRQ enable/disable events للـ task المحدد. بتُستدعى تلقائياً لما الـ lockdep يكتشف potential deadlock تضمّن IRQ context.

- **Parameters:** `curr` — الـ task المراد طباعة trace تبعته.
- **Return:** `void`.
- **Caller context:** Debug/warning path فقط.

---

### Group 8: System & Debug Utilities

---

#### `lockdep_sys_exit`

```c
extern asmlinkage void lockdep_sys_exit(void);
```

بتُستدعى من `syscall_exit_to_user_mode()`. بتفحص إن الـ current task مش لسه شايل أي locks لما بيرجع لـ userspace. لو في locks محجوزة، بيطلع splat وبيعمل dump للـ held_locks.

- **Key details:** `asmlinkage` لأنها بتتعمل call من assembly entry/exit path. مهمة لاكتشاف lock leaks في syscalls.

---

#### `lockdep_reset` / `lockdep_reset_lock`

```c
extern void lockdep_reset(void);
extern void lockdep_reset_lock(struct lockdep_map *lock);
```

- **`lockdep_reset()`**: بتعمل full reset للـ lockdep state (classes، chains، إلخ). بتُستخدم في testing فقط.
- **`lockdep_reset_lock()`**: بتشيل كل الـ classes المرتبطة بـ map واحدة. بتُستخدم من `lockdep_unregister_key()`.

---

#### `lockdep_free_key_range`

```c
extern void lockdep_free_key_range(void *start, unsigned long size);
```

بتحرّر كل الـ lock classes اللي بتستخدم keys في الـ address range المحدد. بتُستخدم عند module unload لتنظيف الـ classes اللي كانت مرتبطة بـ `.data` section الخاصة بالـ module.

---

#### `lockdep_rcu_suspicious`

```c
void lockdep_rcu_suspicious(const char *file, const int line, const char *s);
```

بتطبع warning لما RCU يكتشف إن code بيوصل لـ RCU-protected data من context مش صح (مش داخل RCU read-side critical section). بتُستدعى من `rcu_dereference_check()` macros.

- **Parameters:** `file`، `line` — موقع الـ code. `s` — رسالة تصف المشكلة.
- **Key details:** بتطبع الـ held_locks وإن كانت IRQs enabled/disabled عشان تساعد في debugging.

---

#### `lockdep_set_lock_cmp_fn`

```c
void lockdep_set_lock_cmp_fn(struct lockdep_map *, lock_cmp_fn, lock_print_fn);
```

بتسجّل custom comparator وprinter للـ lock class. بتُستخدم لـ locks اللي عندها ordering semantics مختلفة زي `ww_mutex` (wound/wait mutexes) اللي بتحتاج تقارن lock addresses لتحديد acquisition order.

- **Only active when:** `CONFIG_PROVE_LOCKING`.

---

### Group 9: Control Flow Macros — `lockdep_off` / `lockdep_on`

```c
#define lockdep_off()   \
do {                    \
    current->lockdep_recursion += LOCKDEP_OFF; \
} while (0)

#define lockdep_on()    \
do {                    \
    current->lockdep_recursion -= LOCKDEP_OFF; \
} while (0)
```

**الـ `lockdep_recursion`** field بيستخدم bitfield:
- الـ low 16 bits (mask = `LOCKDEP_RECURSION_MASK`): recursion depth (لما الـ lockdep نفسه بياخذ internal locks).
- الـ bit 16 فصاعداً (`LOCKDEP_OFF`): عدد مرات الـ explicit disable.

الـ macros دي معرّفة عمداً كـ macros وليس inline functions عشان تتجنب tracing وكذلك kprobes من intercept الـ disable path نفسها.

**مثال:**
```c
/* لما بتعمل dump_stack() من داخل lockdep */
lockdep_off();
dump_stack();
lockdep_on();
```

---

### Group 10: Lock-Type Facade Macros

الـ kernel بتستخدم unified `lock_acquire` / `lock_release` API لكن بتوفّر per-type macros للـ clarity والـ documentation:

```
spin_acquire(l, s, t, i)         → lock_acquire_exclusive(l, s, t, NULL, i)
rwlock_acquire(l, s, t, i)       → lock_acquire_exclusive(l, s, t, NULL, i)
rwlock_acquire_read(l, s, t, i)  → lock_acquire_shared or shared_recursive
mutex_acquire(l, s, t, i)        → lock_acquire_exclusive(l, s, t, NULL, i)
rwsem_acquire_read(l, s, t, i)   → lock_acquire_shared(l, s, t, NULL, i)
seqcount_acquire(l, s, t, i)     → lock_acquire_exclusive(l, s, t, NULL, i)
```

الـ `rwlock_acquire_read` هو الحالة الاستثنائية اللي بتشوف `read_lock_is_recursive()` — لأن `read_lock()` على uniprocessor قديم كانت recursive، والـ lockdep لازم يحترم ده.

الـ `lock_map_*` macros بتستخدمها direct callers من الـ lockdep annotation API زي `ww_mutex` و`work_on_cpu()`:

```c
#define lock_map_acquire(l)       lock_acquire_exclusive(l, 0, 0, NULL, _THIS_IP_)
#define lock_map_acquire_try(l)   lock_acquire_exclusive(l, 0, 1, NULL, _THIS_IP_)
#define lock_map_release(l)       lock_release(l, _THIS_IP_)
#define lock_map_sync(l)          lock_sync(l, 0, 0, 1, NULL, _THIS_IP_)
```

---

### Group 11: `might_lock` — Dependency Hints

```c
#define might_lock(lock)
#define might_lock_read(lock)
#define might_lock_nested(lock, subclass)
```

بتضيف **phantom** acquire+release في نفس النقطة. مش بتمسك الـ lock فعلاً، بس بتقول للـ lockdep "الـ function دي **ممكن** تحتاج الـ lock ده". ده بيسمح للـ validator يكتشف لو function عندها might_lock بتتعمل call من context بيحمل lock ممكن يتعارض.

**الفرق عن `lockdep_assert_held()`:** الـ `might_lock` بتقول "ممكن أحتاج"، والـ `assert_held` بتقول "لازم يكون محجوز".

---

### Group 12: `DEFINE_WAIT_OVERRIDE_MAP`

```c
#define DEFINE_WAIT_OVERRIDE_MAP(_name, _wait_type)  \
    struct lockdep_map _name = {                     \
        .name = #_name "-wait-type-override",        \
        .wait_type_inner = _wait_type,               \
        .lock_type = LD_LOCK_WAIT_OVERRIDE, }
```

بيعرّف map خاصة بـ `LD_LOCK_WAIT_OVERRIDE` type. بتُستخدم مع `lock_map_acquire_try()` (مش `lock_map_acquire`) عشان تمنع الـ lockdep من اعتبارها جزء من الـ dependency chain. الاستخدام الرئيسي هو override الـ wait-type checking في context معين بدون إنشاء dependency edges حقيقية.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بـ lockdep

الـ **lockdep** بيكتب بياناته في `/sys/kernel/debug/locking/` (بعد mount الـ debugfs).

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# عرض كل lock classes المسجّلة
cat /sys/kernel/debug/locking/lock_stat

# عرض الـ classes اللي فيها contention عالي
cat /sys/kernel/debug/locking/lock_stat | sort -k3 -rn | head -30
```

**شرح مخرجات `lock_stat`:**

```
class name          con-bounces    contentions   waittime-min   waittime-max   waittime-total   acq-bounces   acquisitions   holdtime-min   holdtime-max   holdtime-total
----------------------------------------------------------------------------------------
&mm->mmap_lock-W:   102            544           0.25ns         1.2ms          23.4ms           1200          98231          0.10ns         4.5ms          12.3s
```

| العمود | المعنى |
|--------|--------|
| `con-bounces` | مرات الـ contention مع bounce بين CPUs |
| `contentions` | إجمالي مرات الانتظار على اللوك |
| `waittime-max` | أطول وقت انتظار — المشبوه الأول في حالة الـ latency |
| `holdtime-total` | إجمالي وقت الحمل — كبير جداً = bottleneck |

```bash
# reset الإحصائيات وابدأ من الأول
echo 0 > /sys/kernel/debug/locking/lock_stat
```

الـ **dependency graph** نفسه:

```bash
# عدد الـ lock classes المستخدمة حالياً
cat /sys/kernel/debug/lockdep

# عرض الـ dependency chain كاملة (ضخمة — أستخدم grep)
cat /sys/kernel/debug/lockdep_chains

# عرض locks المحتجزة الآن
cat /sys/kernel/debug/lockdep | grep -A5 "held locks"
```

---

#### 2. مدخلات الـ sysfs المتعلقة بـ lockdep

```bash
# حالة debug_locks — لو 0 معناها lockdep اكتشف مشكلة وأوقف نفسه
cat /proc/sys/kernel/debug_locks           # ليست في sysfs الرسمي، بس /proc

# عدد الـ lock classes المسجّلة
cat /sys/kernel/debug/lockdep_stats
```

**مخرجات `lockdep_stats` المهمة:**

```
lock-classes:                          1243 [max: 8192]
direct dependencies:                  38291 [max: 65536]
indirect dependencies:               182741
all direct dependencies:             147291
dependency chains:                    14823 [max: 65536]
in-hardirq chains:                     1023
in-softirq chains:                     2134
...
```

| السطر | لو اقترب من الـ max |
|-------|---------------------|
| `lock-classes` | زوّد `MAX_LOCKDEP_KEYS_BITS` في الكود |
| `dependency chains` | علامة على تعقيد شديد في التصميم |
| `debug_locks: 0` | حصل deadlock detection — اقرأ dmesg فوراً |

---

#### 3. ftrace — الـ tracepoints والـ events

```bash
# تفعيل lock tracing events
cd /sys/kernel/debug/tracing

# عرض الـ events المتاحة للـ locking
ls events/lock/

# lock:lock_acquire  lock:lock_release  lock:lock_contended  lock:lock_acquired

# تتبع acquire و release فقط
echo 1 > events/lock/lock_acquire/enable
echo 1 > events/lock/lock_release/enable
echo 1 > tracing_on

# تشغيل الـ workload المشكوك فيه
./my_suspicious_workload

echo 0 > tracing_on
cat trace | head -100
```

**مثال مخرجات:**

```
     kworker/0:1-45    [000]  1234.567890: lock_acquire: &(&zone->lock)->rlock read=0 try=0
     kworker/0:1-45    [000]  1234.567920: lock_acquire: &(&pgdat->lru_lock)->rlock read=0 try=0
```

هنا واضح إن `zone->lock` اتأخذ قبل `lru_lock` — لو في thread تاني العكس هيبان deadlock.

```bash
# تتبع فقط lock معين بالاسم
echo 'name == "mmap_lock"' > events/lock/lock_acquire/filter
echo 1 > events/lock/lock_acquire/enable
```

```bash
# function graph tracer لمتابعة stack وقت الـ deadlock
echo function_graph > current_tracer
echo mutex_lock > set_graph_function
echo 1 > tracing_on
```

---

#### 4. printk والـ dynamic debug

```bash
# تفعيل رسائل lockdep verbose عبر dynamic debug
echo 'module lockdep +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/locking/lockdep.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل طباعة الـ stack trace مع كل رسالة
echo 'file kernel/locking/lockdep.c +ps' > /sys/kernel/debug/dynamic_debug/control

# عرض ما تم تفعيله
cat /sys/kernel/debug/dynamic_debug/control | grep lockdep
```

**في الكود مباشرة** لو بتكتب driver:

```c
/* طباعة stack عند الشك في ترتيب اللوكات */
pr_debug("acquiring lock A while holding lock B at %s:%d\n",
         __FILE__, __LINE__);
dump_stack();
```

لمستوى أعلى من التفاصيل، فعّل `CONFIG_LOCK_STAT` وستظهر إحصائيات كاملة في `/sys/kernel/debug/locking/lock_stat`.

---

#### 5. خيارات الـ kernel config للـ debugging

| Config | الوظيفة | تأثير الأداء |
|--------|---------|--------------|
| `CONFIG_LOCKDEP` | تفعيل الـ lockdep بالكامل | كبير (~10-20%) |
| `CONFIG_PROVE_LOCKING` | يُفعّل `might_lock()`, `lockdep_assert_held()` إلخ | كبير |
| `CONFIG_LOCK_STAT` | إحصائيات `lock_stat` (contention, hold time) | متوسط |
| `CONFIG_DEBUG_LOCKING_API_SELFTESTS` | self-tests عند boot | صغير |
| `CONFIG_PROVE_RAW_LOCK_NESTING` | يتحقق من nesting صحيح raw vs normal spinlock في RT | متوسط |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف double-lock, use-after-free على spinlocks | صغير |
| `CONFIG_DEBUG_MUTEXES` | نفس الفكرة للـ mutex | صغير |
| `CONFIG_DEBUG_RWSEMS` | للـ rwsem | صغير |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يحذر لو نمت داخل atomic context | صغير |
| `CONFIG_PREEMPT_COUNT` | يلزم لـ `lockdep_assert_preemption_*` | متوسط |
| `CONFIG_PROVE_RAW_LOCK_NESTING` | يفعّل `lockdep_assert_RT_in_threaded_ctx()` | صغير |

**الـ .config المثالي لجلسة debugging:**

```
CONFIG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
CONFIG_LOCK_STAT=y
CONFIG_DEBUG_LOCKING_API_SELFTESTS=y
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_MUTEXES=y
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_FRAME_POINTER=y
CONFIG_STACKTRACE=y
```

---

#### 6. أدوات subsystem-specific

**الـ `lockdep_info` في `/proc/lockdep_stats`** (نفس المحتوى):

```bash
cat /proc/lockdep_stats
```

**أداة `perf`** لرصد contention:

```bash
# رصد lock contention events
perf lock record -a -- sleep 10
perf lock report

# تفاصيل حول أكثر اللوكات contention
perf lock report --key acquired --sort acquired -i perf.data
```

**أداة `bpftrace`** لتتبع lockdep events في real-time:

```bash
# من يمسك mutex أكثر من 1ms؟
bpftrace -e '
kprobe:mutex_lock { @start[tid] = nsecs; @name[tid] = arg0; }
kretprobe:mutex_lock /@start[tid]/ {
  $d = nsecs - @start[tid];
  if ($d > 1000000) {
    printf("PID %d held mutex for %lldns\n", pid, $d);
    printf("stack: %s\n", kstack());
  }
  delete(@start[tid]);
}'
```

**الـ `crash` tool** لتحليل vmcore بعد kernel panic ناتج عن deadlock:

```bash
# داخل crash
crash> bt -a          # stack trace لكل threads
crash> foreach bt     # كل الـ stacks
crash> lock           # عرض held locks في الـ crash dump
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|-----------------------|--------|-------|
| `WARNING: possible circular locking dependency detected` | lockdep اكتشف دورة محتملة في graph اللوكات | راجع ترتيب الـ acquire، استخدم subclass أو class-split |
| `WARNING: possible recursive locking detected` | نفس الـ thread حاول يأخذ نفس اللوك مرتين بدون recursive declaration | استخدم `lock_acquire_shared_recursive()` أو `mutex_lock_nested()` |
| `WARNING: possible irq lock inversion dependency detected` | لوك اتأخذ في IRQ context وبرّاه بترتيب معكوس | استخدم `spin_lock_irqsave()` بدلاً من `spin_lock()` |
| `BUG: spinlock already locked on CPU#N` | `CONFIG_DEBUG_SPINLOCK` اكتشف double-lock | في الغالب منطق خاطئ في الكود — تحقق من كل code paths |
| `BUG: sleeping function called from invalid context` | `mutex_lock()` أو ما شابهها اتنادت من atomic context | استبدل بـ `mutex_trylock()` أو أعد تصميم الـ locking |
| `lockdep: MAX_LOCKDEP_ENTRIES too low!` | نفد مساحة الـ dependency graph | زوّد `MAX_LOCKDEP_ENTRIES` أو قلل عدد الـ classes |
| `WARNING: lock held when returning to user space` | thread رجع لـ userspace وفي لوك محتجز | bug في cleanup path — تحقق من كل exits |
| `INFO: possible deadlock scenario` | مسار محتمل لـ deadlock (مش confirmed بعد) | حلل الـ chain المطبوعة وعكس الترتيب |
| `debug_locks is 0` | lockdep أوقف نفسه بعد أول مشكلة | اقرأ dmesg كاملاً للرسالة الأولى |
| `WARNING: bad unlock balance detected` | `lock_release()` أكثر من `lock_acquire()` | تتبع كل code paths بـ ftrace |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

**في `lock_acquire()` / `lock_release()` إذا شككت في caller:**

```c
void my_subsystem_lock(struct my_struct *s)
{
    /* تحقق إن اللوك مش محتجز بالفعل من نفس الـ thread */
    WARN_ON(lockdep_is_held(&s->lock) == LOCK_STATE_HELD);
    mutex_lock(&s->lock);
}
```

**عند دخول critical section:**

```c
void process_data(struct my_struct *s)
{
    /* يضمن إن lockdep يعرف الـ caller مسك اللوك */
    lockdep_assert_held(&s->lock);

    /* أو بشكل صريح بـ WARN */
    if (WARN_ON(!mutex_is_locked(&s->mutex))) {
        dump_stack();
        return;
    }
    /* ... */
}
```

**في IRQ handlers:**

```c
irqreturn_t my_irq_handler(int irq, void *dev)
{
    /* يتحقق إن IRQs معطّلة فعلاً */
    lockdep_assert_irqs_disabled();

    /* يتحقق إن احنا فعلاً في hardirq context */
    lockdep_assert_in_irq();

    spin_lock(&dev->lock);
    /* ... */
    spin_unlock(&dev->lock);
    return IRQ_HANDLED;
}
```

**عند تحرير objects:**

```c
void my_object_free(struct my_struct *s)
{
    /* تأكد مفيش locks محتجزة على هذا الـ object */
    DEBUG_LOCKS_WARN_ON(lockdep_is_held(&s->lock));
    kfree(s);
}
```

**نقاط WARN_ON الذهبية:**

```c
/* عند كل function تتوقع preemption enabled */
lockdep_assert_preemption_enabled();

/* عند كل function تتوقع softirq context */
lockdep_assert_in_softirq();

/* مباشرة قبل sleep في context مشكوك فيه */
might_sleep();  /* تحتوي داخلياً على lockdep checks */
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ hardware مع الـ kernel state

الـ lockdep بطبيعته software-only، لكن لو اللوك ينحمي حالة hardware register:

```bash
# تحقق إن IRQ lines معطّلة فعلاً على الـ CPU
# عبر MSR على x86
rdmsr 0x9b   # IA32_KERNEL_GS_BASE أو استخدم /proc/interrupts

# /proc/interrupts يعطيك interrupt counts — لو توقف = hardware hung
watch -n 1 'cat /proc/interrupts | grep my_device'

# لو اللوك يحمي DMA engine state:
# تحقق إن الـ DMA فعلاً idle
cat /sys/bus/dma/devices/*/in_use  # لو موجود
```

```bash
# في حالة spinlock يحمي hardware queue:
# تحقق من hardware queue depth من الـ sysfs
cat /sys/block/sda/queue/nr_requests
cat /sys/block/sda/queue/in_flight
```

---

#### 2. تقنيات الـ Register Dump

```bash
# قراءة register بـ devmem2 (يحتاج package)
devmem2 0xFEA00000 w    # اقرأ 32-bit من عنوان فيزيائي

# بدون devmem2، عبر /dev/mem مباشرة (يحتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0xFEA00000/4)) 2>/dev/null | xxd

# على ARM/ARM64 — لو الـ registers mapped في iomap:
# استخدم /proc/iomem لتحديد العنوان أولاً
cat /proc/iomem | grep -i "my_device"

# ثم اقرأ
devmem2 <physical_addr> w
```

**في kernel code مباشرة** للـ dump أثناء الـ debugging:

```c
/* داخل driver — dump registers عند deadlock شكّ */
static void dump_hw_state(struct my_device *dev)
{
    u32 status = readl(dev->base + MY_STATUS_REG);
    u32 ctrl   = readl(dev->base + MY_CTRL_REG);
    u32 irq    = readl(dev->base + MY_IRQ_STATUS_REG);

    dev_err(dev->dev,
        "HW state: status=0x%08x ctrl=0x%08x irq=0x%08x\n",
        status, ctrl, irq);

    /* ثم dump_stack لنعرف من استدعى هذا */
    dump_stack();
}

/* استدعيها من داخل lock_acquire path لو WARN_ON تحقق */
```

```bash
# على x86 — عرض APIC registers (مهمة لـ IRQ debugging)
cat /proc/interrupts
# لو في مشكلة IRQ affinity مع لوك:
cat /proc/irq/42/smp_affinity
```

---

#### 3. نصائح Logic Analyzer / Oscilloscope

لو اللوك يحمي **SPI/I2C/GPIO** access:

```
Logic Analyzer Setup:
┌─────────────────────────────────────────────────────┐
│  Channel 0: GPIO_IRQ_LINE                           │
│  Channel 1: SPI_CLK                                 │
│  Channel 2: SPI_CS                                  │
│  Channel 3: GPIO_LOCK_INDICATOR (debug pin)         │
│                                                     │
│  Trigger: Rising edge on GPIO_IRQ_LINE              │
│  Sample Rate: 10x فوق أعلى frequency للـ bus       │
│  Buffer: 10M samples على الأقل                     │
└─────────────────────────────────────────────────────┘
```

**Debug GPIO trick** — اضف GPIO toggle حول الـ critical section:

```c
/* مؤقتاً فقط للـ debugging */
#ifdef CONFIG_MY_DRIVER_DEBUG
    gpiod_set_value(dev->debug_gpio, 1);  /* lock acquired */
#endif
    spin_lock_irqsave(&dev->hw_lock, flags);
    /* critical section */
    spin_unlock_irqrestore(&dev->hw_lock, flags);
#ifdef CONFIG_MY_DRIVER_DEBUG
    gpiod_set_value(dev->debug_gpio, 0);  /* lock released */
#endif
```

على الـ oscilloscope هتشوف:
- **النبضة طويلة جداً** = مشكلة في الـ hold time
- **نبضتان متداخلتان على CPUs مختلفة** = race condition

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في الـ log | الحل |
|---------|----------------|-------|
| IRQ storm يمنع lock release | `irq X: nobody cared` + lock warning | تحقق من `irq_handler` — لازم يكلير الـ interrupt flag في الـ hardware |
| DMA finish flag مش بيتحدّث | `WARNING: CPU#0 stuck for Ns` + dmesg يتوقف | تحقق من coherency — `dma_sync_single_for_cpu()` |
| Hardware queue overflow وlock يحمي counter خاطئ | OOM أو silent corruption | تحقق من `writel` ordering — استخدم `writel_relaxed` بحذر |
| Clock gating يوقف device وسط critical section | `BUG: soft lockup` | disable clock gating أو امسك PM lock قبل device lock |
| PCIe MMIO قبل link up | `Oops: general protection fault` عند register read | تحقق من link state قبل أي access |

---

#### 5. Device Tree Debugging

الـ lockdep نفسه ما له علاقة مباشرة بالـ DT، لكن الـ driver اللي بيستخدم اللوكات ممكن يتأثر بـ DT configuration خاطئة:

```bash
# تحقق إن الـ DT nodes متوافقة مع ما يتوقعه الـ driver
dtc -I fs -O dts /sys/firmware/devicetree/base > /tmp/current.dts

# ابحث عن الـ node بتاعك
grep -A 20 'my_device' /tmp/current.dts

# تحقق من الـ interrupts property
cat /sys/firmware/devicetree/base/soc/my_device/interrupts | xxd

# تحقق إن الـ clocks صح
cat /sys/firmware/devicetree/base/soc/my_device/clocks | xxd
```

**مشكلة شائعة:** driver يعمل `spin_lock_irqsave()` وينتظر interrupt من device، لكن الـ interrupt number في الـ DT غلط:

```bash
# تحقق إن الـ IRQ اتسجّل صح
cat /proc/interrupts | grep my_device

# لو مش موجود — الـ DT node فيه مشكلة
# فعّل DT debugging
echo 8 > /proc/sys/kernel/printk
modprobe my_driver  # هيطبع DT parsing errors
```

```bash
# أداة dtb_check لو متاحة
dtb_check -s my_binding.yaml my.dtb
```

---

### Practical Commands

---

#### 1. أوامر جاهزة للنسخ

**--- فحص أساسي فوري ---**

```bash
# هل lockdep شغّال وسليم؟
cat /proc/sys/kernel/debug_locks    # 1 = OK, 0 = problem detected

# كم lock class مسجّل؟
grep "lock-classes" /sys/kernel/debug/lockdep_stats

# عرض أحدث lockdep warnings
dmesg | grep -A 30 "possible circular locking"
dmesg | grep -A 20 "possible deadlock"
dmesg | grep -A 15 "possible recursive locking"
```

**--- lock statistics ---**

```bash
# reset + collect fresh stats
echo 0 > /sys/kernel/debug/locking/lock_stat
sleep 30   # أو شغّل الـ workload
cat /sys/kernel/debug/locking/lock_stat | \
    awk 'NR>2 {print $3, $1}' | sort -rn | head -20
```

**--- ftrace لـ lock events ---**

```bash
#!/bin/bash
# تفعيل lock tracing كامل
TRACING=/sys/kernel/debug/tracing

echo 0 > $TRACING/tracing_on
echo > $TRACING/trace

echo 1 > $TRACING/events/lock/lock_acquire/enable
echo 1 > $TRACING/events/lock/lock_release/enable
echo 1 > $TRACING/events/lock/lock_contended/enable
echo 1 > $TRACING/events/lock/lock_acquired/enable

echo 1 > $TRACING/tracing_on
echo "Tracing for 10 seconds..."
sleep 10
echo 0 > $TRACING/tracing_on

# احفظ النتائج
cat $TRACING/trace > /tmp/lock_trace_$(date +%s).txt
echo "Saved to /tmp/lock_trace_*.txt"
```

**--- بحث عن deadlock scenario ---**

```bash
# لو في hang، اجمع info كاملة
echo t > /proc/sysrq-trigger   # طبّع stack trace لكل threads
echo w > /proc/sysrq-trigger   # طبّع tasks في D state
echo l > /proc/sysrq-trigger   # طبّع backtrace لكل CPUs

# ثم
dmesg | grep -A 50 "INFO: task.*blocked"
dmesg | grep "held by"
```

**--- perf lock profiling ---**

```bash
# record
perf lock record -a -g -- sleep 30

# report
perf lock report --key wait_total --sort wait_total

# مخرجات مثال:
# Name                    acquired  contended  avg wait  max wait  total wait
# &(&zone->lock)->rlock    1234567      12345    1.23us  234.56us  15.23ms
```

**--- bpftrace لـ real-time lock latency ---**

```bash
# أي mutex بياخد أكتر من 500 microsecond؟
bpftrace -e '
kprobe:mutex_lock_interruptible {
    @start[tid] = nsecs;
    @lock[tid] = arg0;
}
kretprobe:mutex_lock_interruptible /@start[tid]/ {
    $lat = (nsecs - @start[tid]) / 1000;
    if ($lat > 500) {
        printf("%-16s %-6d %ldus\n", comm, pid, $lat);
        printf("%s\n\n", kstack(5));
    }
    delete(@start[tid]);
    delete(@lock[tid]);
}
'
```

---

#### 2. مثال كامل — قراءة وتفسير lockdep warning

**الـ warning:**

```
======================================================
WARNING: possible circular locking dependency detected
6.8.0 #1 SMP PREEMPT_DYNAMIC
------------------------------------------------------
kworker/u4:2/89 is trying to acquire lock:
ffffffff82345678 (&dev->spinlock){..-.}-{2:2}, at: my_driver_irq+0x4c/0x120

but task is already holding lock:
ffffffff82345690 (&mm->mmap_lock#2){++++}-{3:3}, at: do_page_fault+0x123/0x400

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:

-> #2 (&mm->mmap_lock#2){++++}-{3:3}:
       lock_acquire+0xdc/0x2b0
       down_read+0x42/0x190
       do_page_fault+0x123/0x400

-> #1 (&dev->spinlock){..-.}-{2:2}:
       lock_acquire+0xdc/0x2b0
       _raw_spin_lock_irq+0x42/0x70
       my_driver_write+0x89/0x200

-> #0 (&mm->mmap_lock#2){++++}-{3:3}:
       lock_acquire+0xdc/0x2b0
       down_read+0x42/0x190
       my_driver_irq+0x4c/0x120
```

**التفسير:**

```
سيناريو الـ deadlock:

Thread A (page fault):                Thread B (IRQ handler):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━        ━━━━━━━━━━━━━━━━━━━━━━━━
1. يمسك mmap_lock (read)              1. يمسك dev->spinlock
2. يحاول يمسك dev->spinlock    ←wait→  2. يحاول يمسك mmap_lock
                                           (DEADLOCK!)
```

**الحل:**

```c
/* بدل ما الـ IRQ handler يمسك mmap_lock مباشرة،
 * استخدم work queue لعمل الـ page access خارج IRQ context */
static irqreturn_t my_driver_irq(int irq, void *data)
{
    struct my_device *dev = data;

    spin_lock(&dev->spinlock);
    dev->pending_work = true;
    spin_unlock(&dev->spinlock);

    /* بدل down_read(&mm->mmap_lock) هنا */
    schedule_work(&dev->page_work);  /* يشتغل في process context */

    return IRQ_HANDLED;
}
```

---

#### 3. script تشخيص سريع شامل

```bash
#!/bin/bash
# lockdep_diag.sh - تشخيص سريع للوضع

echo "=== lockdep Health ==="
DLOCKS=$(cat /proc/sys/kernel/debug_locks 2>/dev/null || echo "N/A")
echo "debug_locks: $DLOCKS"
[ "$DLOCKS" = "0" ] && echo "*** WARNING: lockdep disabled itself - check dmesg ***"

echo ""
echo "=== Lock Classes Usage ==="
grep -E "lock-classes|dependency chains|chain.hlocks" \
    /sys/kernel/debug/lockdep_stats 2>/dev/null

echo ""
echo "=== Recent Lockdep Warnings ==="
dmesg | grep -c "possible circular"
dmesg | grep -c "possible deadlock"
dmesg | grep -c "possible recursive"

echo ""
echo "=== Tasks in D state (possible deadlock) ==="
ps aux | awk '$8 == "D" {print $0}'

echo ""
echo "=== Top 5 contended locks ==="
if [ -f /sys/kernel/debug/locking/lock_stat ]; then
    awk 'NR>2 && $3>0 {print $3, $1}' \
        /sys/kernel/debug/locking/lock_stat | \
        sort -rn | head -5
fi
```

```bash
chmod +x lockdep_diag.sh
./lockdep_diag.sh
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Deadlock في درايفر I2C على RK3562 — Industrial Gateway

#### العنوان
**Circular Dependency** بين `i2c_lock` و `regulator_mutex` بتخلّي الـ gateway يتعلق عند الـ boot

#### السياق
شركة بتبني industrial gateway على RK3562. الجهاز بيقرأ بيانات من sensors عن I2C وبيتحكم في power rails عن طريق regulator framework. المنتج اشتغل كويس في testing، لكن في production بدأ يتعلق عشوائياً عند الـ boot بعد تفعيل power management.

#### المشكلة
الـ system بيتعلق تماماً — لا watchdog response، لا kernel log جديد. الـ serial console بتفضل فاضية.

#### التحليل

لما فعّلوا `CONFIG_LOCKDEP=y` في debug build، الكرنل طلع الـ warning دي قبل ما يتعلق:

```
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
i2c_thread/1 is trying to acquire lock:
  (&adap->bus_lock){+.+.}, at: i2c_lock_bus+0x28/0x3c

but task is already holding lock:
  (&rdev->mutex){+.+.}, at: regulator_enable+0x44/0x1c0

which lock already belongs to the lock dependency chain:
  (&rdev->mutex) --> (&adap->bus_lock)
```

**الـ lockdep شغّال إزاي هنا:**

الـ `lock_acquire()` في `lockdep.h` اتنادى لما I2C thread حاول يـ acquire الـ `bus_lock`:

```c
/* lockdep.h line 227 */
extern void lock_acquire(struct lockdep_map *lock, unsigned int subclass,
                         int trylock, int read, int check,
                         struct lockdep_map *nest_lock, unsigned long ip);
```

الـ lockdep يبني **dependency graph** — كل lock فيه `struct lock_list` بيعمل linked list من الـ locks اللي اتأخدت بعده:

```c
/* lockdep.h lines 48-64 */
struct lock_list {
    struct list_head    entry;
    struct lock_class   *class;
    struct lock_class   *links_to;   /* الـ lock اللي الـ edge بيوصله */
    const struct lock_trace *trace;
    u16                 distance;
    u8                  dep;
    u8                  only_xr;
    struct lock_list    *parent;     /* للـ BFS traversal */
};
```

الـ BFS algorithm اكتشف الـ cycle:
- Thread A: أخد `regulator_mutex` → حاول ياخد `i2c bus_lock`
- Thread B: أخد `i2c bus_lock` → حاول ياخد `regulator_mutex` (عشان regulator بيستخدم I2C)

الـ `lock_chain` بيخزن الـ chain_key:
```c
/* lockdep.h lines 75-83 */
struct lock_chain {
    unsigned int irq_context :  2,
                 depth       :  6,
                 base        : 24;
    struct hlist_node   entry;
    u64                 chain_key;  /* hash فريد للـ dependency chain */
};
```

الـ `chain_key` مختلف في كل context (IRQ vs process)، وده ساعد lockdep يحدد إن الـ cycle بتحصل في process context بس.

#### الحل

```c
/* في driver/i2c/busses/i2c-rk3x.c — فصل الـ regulator enable عن الـ I2C transaction */

/* قبل الحل: كود خاطئ */
static int rk3x_i2c_xfer(struct i2c_adapter *adap, ...)
{
    /* رداً على طلب I2C، بيعمل regulator enable وهو شايل i2c lock */
    regulator_enable(i2c->vdd);  /* DEADLOCK هنا */
    /* ... */
}

/* بعد الحل: فصل الـ power management */
static int rk3x_i2c_xfer(struct i2c_adapter *adap, ...)
{
    /* الـ regulator enable بره الـ i2c lock */
    /* ... التراسنزاكشن العادية */
}

static int rk3x_i2c_probe(struct platform_device *pdev)
{
    /* enable الـ regulator مرة واحدة في الـ probe، مش في كل transaction */
    ret = regulator_enable(i2c->vdd);
}
```

**أو استخدام `lockdep_set_class` لتعريف ordering صريح:**

```c
static DEFINE_MUTEX(regulator_i2c_mutex);
/* في الـ probe */
lockdep_set_class_and_name(&i2c->bus_lock, &i2c_after_reg_key, "i2c_after_reg");
```

```bash
# للكشف في production بدون recompile:
echo 1 > /proc/sys/kernel/hung_task_timeout_secs
cat /proc/lockdep_stats
cat /proc/lockdep
```

#### الدرس المستفاد
الـ `struct lock_list` والـ BFS graph في lockdep بيكتشف الـ circular dependencies اللي ممكن تاخد أسابيع لو اعتمدنا على الـ testing العادي. في embedded SoCs زي RK3562، الـ power management و bus drivers بيتشابكوا كتير — لازم تفعّل `CONFIG_LOCKDEP` في مرحلة الـ bring-up.

---

### السيناريو 2: IRQ-Safe Lock Violation في STM32MP1 — IoT Sensor Node

#### العنوان
**الـ mutex بيتأخد جوا IRQ handler** في درايفر SPI على STM32MP1، وده بيودي لـ `lockdep_assert_irqs_disabled()` failure

#### السياق
فريق بيشغّل IoT sensor node على STM32MP1 بيقرأ accelerometer بيانات عن SPI. الكود القديم كان بيشتغل على STM32F4 bare-metal، والمطور port الكود للـ Linux driver بسرعة مع تغييرات بسيطة.

#### المشكلة
الـ system بيطير `BUG: sleeping function called from invalid context` بشكل عشوائي، مع stack trace بيشير لـ `mutex_lock()` جوا interrupt handler.

#### التحليل

الـ lockdep كشف المشكلة عن طريق `lockdep_assert_no_hardirq()`:

```c
/* lockdep.h lines 590-594 */
#define lockdep_assert_no_hardirq()                 \
do {                                                \
    WARN_ON_ONCE(__lockdep_enabled &&               \
                 (this_cpu_read(hardirq_context) || \
                  !this_cpu_read(hardirqs_enabled))); \
} while (0)
```

الـ `__lockdep_enabled` macro:
```c
/* lockdep.h line 573 */
#define __lockdep_enabled   (debug_locks && !this_cpu_read(lockdep_recursion))
```

الـ `lockdep_recursion` per-CPU counter بيمنع lockdep من تتبع نفسه recursively. لما الـ IRQ handler حاول ياخد الـ mutex، الـ `lock_acquire()` اتنادى مع:
- `read = 0` (exclusive)
- `check = 1` (full validation)

الـ lockdep بص على `held_lock.irq_context`:
```c
/* lockdep_types.h line 248 */
unsigned int irq_context:2; /* bit 0 - soft, bit 1 - hard */
```

لقى إن الـ lock بيتأخد في `irq_context = 2` (hardirq)، بس الـ `wait_type_inner` للـ mutex هو `LD_WAIT_SLEEP` — contradiction واضحة.

الـ `lock_chain` اللي اتسجّل:
```c
/* lock_chain.irq_context = 2 (hardirq) */
/* لكن lock_class.wait_type_inner = LD_WAIT_SLEEP */
/* CONFLICT! */
```

#### الحل

```c
/* الكود الخاطئ */
static irqreturn_t stm32_spi_irq(int irq, void *dev_id)
{
    struct stm32_spi *spi = dev_id;
    mutex_lock(&spi->lock);   /* WRONG: mutex in IRQ context */
    /* process data */
    mutex_unlock(&spi->lock);
    return IRQ_HANDLED;
}

/* الكود الصح: استخدام spinlock مع IRQ save */
static irqreturn_t stm32_spi_irq(int irq, void *dev_id)
{
    struct stm32_spi *spi = dev_id;
    unsigned long flags;

    spin_lock_irqsave(&spi->lock, flags);  /* IRQ-safe */
    /* process data */
    spin_unlock_irqrestore(&spi->lock, flags);
    return IRQ_HANDLED;
}

/* تغيير الـ lock type في الـ probe */
static int stm32_spi_probe(struct platform_device *pdev)
{
    spin_lock_init(&spi->lock);
    /* lockdep بيعرف إن spin_lock wait_type هو LD_WAIT_SPIN */
}
```

```bash
# رؤية الـ lock usage states:
cat /proc/lockdep | grep "stm32_spi"

# رؤية الـ IRQ contexts المحتملة:
cat /proc/lockdep_stats | grep -i "irq"
```

#### الدرس المستفاد
الـ `lockdep_wait_type` enum في `lockdep_types.h` بيحدد بدقة أي context مسموح فيه بأخد الـ lock. الـ `LD_WAIT_SLEEP` مش compatible مع hardirq context. فهم الـ `irq_context` field في `held_lock` بيخليك تفهم إيه اللي lockdep شايفه بالظبط.

---

### السيناريو 3: False Positive في i.MX8 HDMI Driver — Android TV Box

#### العنوان
**lockdep false positive** بسبب shared `lock_class_key` بين instances مختلفة في درايفر HDMI على i.MX8

#### السياق
Android TV box على i.MX8MQ بيستخدم HDMI output. الـ driver بيدير multiple display pipelines، وكل pipeline فيها mutex خاص بيها. الـ lockdep بيبعّت warning بيقول في circular dependency، بس الـ code review مفيش مشكلة حقيقية.

#### المشكلة
```
WARNING: possible recursive locking detected
imx8_hdmi_thread is trying to acquire lock:
  (&pipe->mutex){+.+.}, at: imx8_hdmi_update+0x5c

but task is already holding lock:
  (&pipe->mutex){+.+.}, at: imx8_hdmi_enable+0x44
```

الـ lockdep شايف إن الـ thread بياخد نفس الـ lock مرتين — بس في الحقيقة هما instance مختلفتين من الـ mutex!

#### التحليل

المشكلة إن كل الـ `pipe->mutex` instances بتشارك نفس الـ `lock_class_key` لأنهم static structs في نفس الـ array:

```c
/* إيه اللي بيحصل في الـ driver */
struct imx8_pipe {
    struct mutex mutex;
    /* ... */
};

static struct imx8_pipe pipes[2];  /* pipe0 و pipe1 */
```

الـ lockdep بيستخدم `lockdep_match_key()`:
```c
/* lockdep.h lines 207-211 */
static inline int lockdep_match_key(struct lockdep_map *lock,
                                    struct lock_class_key *key)
{
    return lock->key == key;  /* pointer comparison للـ key */
}
```

لأن كل الـ mutex instances في نفس الـ struct definition، الـ lockdep بيديلهم نفس `lock_class_key`، فبيشوفهم كـ "نفس الـ lock". لما الكود بياخد `pipes[0].mutex` وبعدين `pipes[1].mutex`، الـ lockdep بيعتبره recursion على نفس الـ class.

الـ `lockdep_copy_map()` اللي بيتنادى عند copy of lock:
```c
/* lockdep.h lines 26-42 */
static inline void lockdep_copy_map(struct lockdep_map *to,
                                    struct lockdep_map *from)
{
    int i;
    *to = *from;
    /* Clear the cache to avoid half-pointer observations on 64-bit */
    for (i = 0; i < NR_LOCKDEP_CACHING_CLASSES; i++)
        to->class_cache[i] = NULL;
}
```

الـ cache بيتمسح بس الـ `key` بيفضل نفسه — وده أصل المشكلة.

#### الحل

استخدام `lockdep_set_class` لإعطاء كل instance class مختلف:

```c
/* في الـ probe أو الـ initialization */
static struct lock_class_key imx8_pipe0_key;
static struct lock_class_key imx8_pipe1_key;
static struct lock_class_key *imx8_pipe_keys[] = {
    &imx8_pipe0_key,
    &imx8_pipe1_key,
};

static int imx8_hdmi_init_pipe(struct imx8_pipe *pipe, int idx)
{
    mutex_init(&pipe->mutex);
    /* كل instance باخد key مختلف */
    lockdep_set_class(&pipe->mutex, imx8_pipe_keys[idx]);
    return 0;
}
```

أو لو الـ pipes بتتداخل بشكل شرعي، استخدام subclass:

```c
/* pipe0 subclass 0، pipe1 subclass 1 — lockdep بيعرف الـ ordering */
lockdep_set_class_and_subclass(&pipe->mutex, &imx8_pipe_class, idx);
```

```bash
# التحقق من الـ class assignment:
cat /proc/lockdep | grep "imx8_pipe"

# رؤية كل الـ lock classes:
cat /proc/lockdep_chains | head -50
```

#### الدرس المستفاد
الـ `lockdep_match_key()` بيعتمد على **pointer identity** للـ `lock_class_key`، مش الـ value. لما عندك array من locks، لازم تدي كل instance key منفصل باستخدام `lockdep_set_class()` أو `lockdep_set_class_and_subclass()`. ده بيمنع الـ false positives ويخلّي lockdep يفرق بين الـ instances الحقيقية.

---

### السيناريو 4: Lock Held Across Sleep في AM62x — Automotive ECU

#### العنوان
**Lock held عند exit من syscall** — lockdep بيكتشف إن ECU على AM62x بيخرج من syscall وهو لسه شايل mutex

#### السياق
Automotive ECU على TI AM62x بيشغّل AUTOSAR-inspired Linux stack. الكود بيدير CAN bus communication عن طريق custom character device. أثناء stress testing، الـ system بدأ يظهر hangs عشوائية مع log messages غريبة.

#### المشكلة
```
BUG: lock held when returning to user space!
ffff800008a12340 (&can_dev->lock){+.+.}, at: can_ioctl+0x78/0x200
1 lock held by can_ecud/1234:
 #0: ffff800008a12340 (&can_dev->lock){+.+.}, at: can_ioctl+0x78
```

#### التحليل

الـ lockdep عنده `lockdep_sys_exit()` — بيتنادى عند كل syscall exit:

```c
/* lockdep.h line 92 */
extern asmlinkage void lockdep_sys_exit(void);
```

الـ function دي بتتحقق من `current->lockdep_depth` — لو مش zero، يبقى في locks لسه محتفظ بيها.

الـ `lockdep_depth` macro:
```c
/* lockdep.h line 276 */
#define lockdep_depth(tsk) (debug_locks ? (tsk)->lockdep_depth : 0)
```

والـ `lockdep_assert_none_held_once()`:
```c
/* lockdep.h lines 299-300 */
#define lockdep_assert_none_held_once()     \
    lockdep_assert_once(!current->lockdep_depth)
```

**الكود الخاطئ في الـ driver:**

```c
static long can_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct can_dev *dev = file->private_data;

    mutex_lock(&dev->lock);

    switch (cmd) {
    case CAN_IOCTL_SEND:
        /* ... */
        if (copy_from_user(&frame, (void __user *)arg, sizeof(frame)))
            return -EFAULT;  /* BUG: return بدون unlock! */
        /* ... */
        break;
    case CAN_IOCTL_RESET:
        /* ... */
        return 0;  /* BUG: return تاني بدون unlock! */
    }

    mutex_unlock(&dev->lock);
    return 0;
}
```

الـ `lock_acquire()` اتنادى (عبر `mutex_acquire` macro):
```c
/* lockdep.h line 532 */
#define mutex_acquire(l, s, t, i) lock_acquire_exclusive(l, s, t, NULL, i)
```

لكن `lock_release()` ما اتناداش في الـ error paths، فالـ `held_lock` فضل في stack الـ task.

```c
/* lockdep.h line 231 */
extern void lock_release(struct lockdep_map *lock, unsigned long ip);
```

الـ `held_lock.acquire_ip` خزّن الـ instruction pointer اللي أخد فيه الـ lock — ده اللي ظهر في الـ warning وساعد في تحديد المشكلة.

#### الحل

```c
static long can_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct can_dev *dev = file->private_data;
    long ret = 0;

    mutex_lock(&dev->lock);

    switch (cmd) {
    case CAN_IOCTL_SEND:
        if (copy_from_user(&frame, (void __user *)arg, sizeof(frame))) {
            ret = -EFAULT;
            goto out;  /* cleanup path واحد */
        }
        /* ... */
        break;
    case CAN_IOCTL_RESET:
        /* ... */
        break;
    default:
        ret = -EINVAL;
    }

out:
    mutex_unlock(&dev->lock);  /* دايماً بيتنفذ */
    return ret;
}
```

```bash
# تفعيل lock held at exit detection:
# CONFIG_DEBUG_LOCKING_API_SELFTESTS=y
# CONFIG_PROVE_LOCKING=y

# رؤية الـ held locks للـ task الحالي:
cat /proc/1234/status | grep -i lock

# kernel log:
dmesg | grep "lock held"
```

#### الدرس المستفاد
الـ `lockdep_sys_exit()` بيعمل consistency check على كل task عند خروجه من kernel space. الـ `held_lock.acquire_ip` بيخزن مكان الـ lock acquisition بالظبط — ده بيوفر وقت هائل في debugging. في driver code، استخدم `goto out` pattern دايماً عشان تضمن إن الـ unlock بيحصل في كل الـ paths.

---

### السيناريو 5: Softirq Lock Nesting Violation في Allwinner H616 — Custom Board Bring-up

#### العنوان
**lockdep_assert_in_softirq() failure** في درايفر USB على Allwinner H616 أثناء الـ bring-up الأولي

#### السياق
فريق bring-up بيشغّل custom board على Allwinner H616 (المستخدم في Orange Pi boards). الـ board فيها USB hub متصل عليه عدة devices. أثناء أول boot مع `CONFIG_PROVE_LOCKING=y`، الكرنل بيطلع warnings غريبة من subsystem خاص بالـ USB network adapter (RTL8153).

#### المشكلة
```
WARNING: WARN_ON_ONCE(__lockdep_enabled && (!in_softirq() || in_hardirq() || in_nmi()))
at drivers/net/usb/r8152.c:xxx in rtl8152_rx_bottom
```

الـ driver بيدّعي إنه في softirq context، لكن lockdep شايف غير كده.

#### التحليل

الـ `lockdep_assert_in_softirq()` macro:
```c
/* lockdep.h lines 616-620 */
#define lockdep_assert_in_softirq()                         \
do {                                                        \
    WARN_ON_ONCE(__lockdep_enabled              &&          \
                 (!in_softirq() || in_hardirq() || in_nmi())); \
} while (0)
```

والـ `__lockdep_enabled`:
```c
/* lockdep.h line 573 */
#define __lockdep_enabled   (debug_locks && !this_cpu_read(lockdep_recursion))
```

الـ `lockdep_recursion` per-CPU variable:
```c
/* lockdep.h line 571 */
DECLARE_PER_CPU(unsigned int, lockdep_recursion);
```

**الـ LOCKDEP_OFF/ON macros:**
```c
/* lockdep.h lines 100-117 */
#define LOCKDEP_RECURSION_BITS  16
#define LOCKDEP_OFF             (1U << LOCKDEP_RECURSION_BITS)
#define LOCKDEP_RECURSION_MASK  (LOCKDEP_OFF - 1)

#define lockdep_off()       current->lockdep_recursion += LOCKDEP_OFF
#define lockdep_on()        current->lockdep_recursion -= LOCKDEP_OFF
```

الـ split design (16 bits for recursion depth، upper bits for "off" state) بيخلي lockdep يعرف الفرق بين "معطّل" و"recursion عادي".

**جذر المشكلة على الـ H616:**

الـ Allwinner H616 USB DMA interrupt handler بيستدعي الـ completion callback في hardirq context بدل softirq — bug في الـ SoC driver الجديد:

```c
/* الكود الخاطئ في sun8i_usb_host.c */
static irqreturn_t sun8i_usb_irq(int irq, void *data)
{
    /* بيشغّل completion وهو في hardirq */
    usb_hcd_poll_rh_status(hcd);  /* ده بيـ trigger softirq callbacks */
    return IRQ_HANDLED;
}
```

لما الـ r8152 driver اتنادى من hardirq بدل softirq، الـ `in_hardirq()` رجعت true، فالـ assertion فشلت.

**تتبع الـ xhlock_context_t:**
```c
/* lockdep.h lines 420-424 */
enum xhlock_context_t {
    XHLOCK_HARD,   /* hardirq context */
    XHLOCK_SOFT,   /* softirq context */
    XHLOCK_CTX_NR,
};
```

الـ lockdep بيستخدم الـ context type ده لتتبع الـ cross-context lock dependencies.

#### الحل

**الحل الأول: إصلاح الـ interrupt handling في الـ SoC driver:**

```c
/* إصلاح sun8i_usb_host.c */
static irqreturn_t sun8i_usb_irq(int irq, void *data)
{
    /* schedule tasklet بدل الاستدعاء المباشر */
    tasklet_schedule(&hcd->poll_tasklet);
    return IRQ_HANDLED;
}

static void sun8i_usb_poll_tasklet(unsigned long data)
{
    /* ده بيشتغل في softirq context — صح */
    usb_hcd_poll_rh_status(hcd);
}
```

**الحل الثاني: إضافة `lockdep_off()` كـ workaround مؤقت للـ bring-up:**

```c
/* مؤقت للـ debugging فقط */
static irqreturn_t sun8i_usb_irq(int irq, void *data)
{
    lockdep_off();           /* disable lockdep مؤقتاً */
    usb_hcd_poll_rh_status(hcd);
    lockdep_on();
    return IRQ_HANDLED;
}
```

```bash
# تشخيص الـ context:
# في kernel/trace، enable function tracer:
echo function > /sys/kernel/debug/tracing/current_tracer
echo sun8i_usb_irq > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# رؤية الـ softirq stats:
cat /proc/softirqs

# رؤية الـ hardirq context violations:
dmesg | grep -i "softirq\|hardirq\|lockdep"

# قراءة lock stats:
cat /proc/lock_stat | head -30
```

**الـ LOCK_CONTENDED macro للـ statistics:**
```c
/* lockdep.h lines 441-448 */
#define LOCK_CONTENDED(_lock, try, lock)            \
do {                                                \
    if (!try(_lock)) {                              \
        lock_contended(&(_lock)->dep_map, _RET_IP_); \
        lock(_lock);                                \
    }                                               \
    lock_acquired(&(_lock)->dep_map, _RET_IP_);     \
} while (0)
```

الـ `_RET_IP_` بيخزن الـ caller address — ده بيظهر في `/proc/lock_stat` ويساعد في تحديد hotspots.

#### الدرس المستفاد
أثناء SoC bring-up على platforms زي Allwinner H616 اللي documentation عندها gaps، الـ `lockdep_assert_in_softirq()` و `lockdep_assert_no_hardirq()` بيكونوا أسرع من أي oscilloscope في كشف الـ incorrect context execution. الـ `lockdep_off()` / `lockdep_on()` pair مفيد كـ workaround مؤقت أثناء الـ bring-up — بس **لازم يتحذف قبل الـ upstream submission**. الـ split counter design (LOCKDEP_OFF vs LOCKDEP_RECURSION_MASK) بيضمن إن lockdep نفسه ما يدخلش في recursion loop.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأول لمتابعة تطور lockdep في kernel — كل المقالات دي حقيقية ومش مفبركة:

| المقال | الوصف |
|--------|-------|
| [ANNOUNCE: lock validator -V1](https://lwn.net/Articles/185605/) | أول إعلان رسمي عن lockdep من Ingo Molnar سنة 2006 — نقطة الانطلاق |
| [Interrupts, threads, and lockdep](https://lwn.net/Articles/321663/) | كيف بيتعامل lockdep مع IRQ context و threaded interrupts |
| [lockdep: Introduce wait-type checks](https://lwn.net/Articles/579849/) | إضافة فحص نوع الانتظار (wait-type) — أساس `LD_WAIT_*` اللي في الكود |
| [Lockdep-RCU](https://lwn.net/Articles/371986/) | تكامل lockdep مع RCU — ظهر `lockdep_rcu_suspicious()` |
| [Crossrelease Lockdep](https://lwn.net/Articles/705379/) | فكرة تتبع locks اللي بتتحرر في thread مختلف |
| [Enhancing lockdep with crossrelease](https://lwn.net/Articles/709849/) | التفاصيل التقنية لـ crossrelease extension |
| [User-space lockdep](https://lwn.net/Articles/536363/) | تشغيل lockdep في userspace لاختبار locking logic |
| [liblockdep: userspace lockdep](https://lwn.net/Articles/548906/) | `liblockdep` — مكتبة userspace بتعيد استخدام كود kernel |
| [DEPT(DEPendency Tracker)](https://lwn.net/Articles/959345/) | البديل المقترح لـ lockdep — بيتتبع wait/event مش بس locks |
| [The dependency tracker for complex deadlock detection](https://lwn.net/Articles/1036222/) | تحليل DEPT وقدرته على اكتشاف deadlocks مستحيلة على lockdep |
| [lockdep: Implement crossrelease feature](https://lwn.net/Articles/700481/) | patch series أضافت crossrelease للـ kernel |

---

### التوثيق الرسمي في kernel

الملفات دي موجودة في `Documentation/locking/` وهي المرجع الأساسي:

```
Documentation/locking/lockdep-design.rst     ← التصميم الكامل والمنطق الداخلي
Documentation/locking/lockdep-design.txt     ← النسخة القديمة (لا تزال متاحة)
Documentation/locking/lockstat.rst           ← إحصائيات lock_contended / lock_acquired
Documentation/locking/mutex-design.rst       ← كيف mutex بتتكامل مع lockdep
Documentation/locking/rt-mutex-design.rst    ← RT-mutex و priority inheritance
Documentation/locking/ww-mutex-design.rst    ← Wound/Wait mutexes ومنع deadlock
```

**الـ** `lockdep-design.rst` هو أهم ملف — بيشرح:
- مفهوم **lock class** وإزاي بيتم grouping
- خوارزمية **BFS** اللي بتشوف في `struct lock_list`
- **IRQ-safety** و 4 حالات الـ usage state
- حدود النظام: `MAX_LOCKDEP_CLASSES`، `MAX_LOCK_DEPTH`

---

### الكود المصدري الأساسي في kernel

الملفات دي هي قلب التنفيذ:

```
kernel/locking/lockdep.c          ← التنفيذ الرئيسي — أكبر ملف في kernel
kernel/locking/lockdep_proc.c     ← /proc/lockdep و /proc/lockdep_stats
include/linux/lockdep.h           ← الملف اللي بنوثقه
include/linux/lockdep_types.h     ← struct lockdep_map, lock_class_key, إلخ
include/linux/lockdep_internals.h ← constants داخلية (MAX_LOCKDEP_CLASSES)
```

**الـ** GitHub للنسخة الحالية من الـ header:
- [include/linux/lockdep.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/lockdep.h)
- [kernel/locking/lockdep.c على GitHub](https://github.com/torvalds/linux/blob/master/kernel/locking/lockdep.c)

---

### Commits مهمة في تاريخ lockdep

| الـ Commit / الإصدار | الوصف |
|----------------------|-------|
| **Linux 2.6.18** (2006) | أول إصدار رسمي لـ lockdep في mainline — بواسطة Ingo Molnar |
| **Linux 2.6.23** | إضافة `CONFIG_LOCK_STAT` وإحصائيات `lock_contended` / `lock_acquired` |
| **Linux 2.6.34** | دعم `lockdep_rcu_suspicious()` — تكامل مع RCU |
| **Linux 4.15+** | إضافة wait-type checks (`LD_WAIT_FREE`, `LD_WAIT_SPIN`, إلخ) |
| **Linux 5.x+** | `lockdep_set_notrack_class()` — لـ subsystems زي bcachefs اللي بتتجاوز حد الـ 48 lock |

لمشاهدة تاريخ التغييرات على الـ header مباشرة:
```bash
git log --follow include/linux/lockdep.h
```

---

### نقاشات Mailing List

**الـ** LKML (Linux Kernel Mailing List) فيه نقاشات مهمة:

- [RFC: lockdep support for recursive read locks (2018)](https://lists.openwall.net/linux-kernel/2018/02/22/173) — Boqun Feng بيضيف دعم deadlock detection للـ recursive read locks
- [RFC: DEPT (DEPendency Tracker)](https://lkml.kernel.org/lkml/1643245873-15542-14-git-send-email-byungchul.park@lge.com/T/) — Byungchul Park بيقترح بديل أشمل من lockdep
- [DEPT v14 implementation](https://www.mail-archive.com/dri-devel@lists.freedesktop.org/msg492598.html) — أحدث patch series لـ DEPT

---

### توثيق kernel.org الرسمي على الإنترنت

- [Runtime locking correctness validator — kernel.org docs](https://docs.kernel.org/locking/lockdep-design.html) ← النسخة الـ HTML من `lockdep-design.rst`
- [RCU and lockdep checking](https://www.infradead.org/~mchehab/kernel_docs/RCU/lockdep.html) ← تكامل lockdep مع RCU API
- [Wound/Wait Mutex Design](https://www.kernel.org/doc/html/v5.4/locking/ww-mutex-design.html) ← deadlock avoidance بدون lockdep

---

### Kernelnewbies.org

**الـ** kernelnewbies.org بيوثق lockdep كجزء من تغييرات إصدارات kernel:

- [Linux 2.6.18 — إطلاق lockdep](https://kernelnewbies.org/Linux_2_6_18) ← صفحة الإصدار اللي ظهر فيه lockdep لأول مرة
- [Linux 2.6.23 — إضافة lock statistics](https://kernelnewbies.org/Linux_2_6_23) ← `CONFIG_LOCK_STAT` و `lock_contended()`
- [Linux 2.6.34 — lockdep-style RCU checking](https://kernelnewbies.org/Linux_2_6_34) ← `rcu_dereference()` بدأ يستخدم lockdep
- [Linux 6.4 — تحديثات locking](https://kernelnewbies.org/Linux_6.4) ← تغييرات locking في الإصدارات الحديثة

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 5**: Concurrency and Race Conditions — بيشرح semaphores و spinlocks
- **الفصل 7**: Time, Delays, and Deferred Work — context و locking في interrupts
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- lockdep نفسه ما اتغطاش بشكل كافي في LDD3 لأنه ظهر بعد الكتاب — ارجع للـ kernel docs

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 9**: An Introduction to Kernel Synchronization — يغطي spinlock، mutex، semaphore
- **الفصل 10**: Kernel Synchronization Methods — `spin_lock_irqsave()`، `mutex_lock()`
- **الفصل 11**: Timers and Time Management — IRQ context وأثره على locking
- بيشرح **سبب** وجود lockdep من خلال أمثلة deadlock حقيقية

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 14**: Kernel Debugging Techniques — بيذكر lockdep كأداة debugging
- مفيد لفهم كيفية تفعيل lockdep على embedded targets

#### Linux Kernel Programming — Kaiwan N. Billimoria
- بيغطي lockdep بشكل مباشر مع أمثلة عملية
- أحدث كتاب عملياً في موضوع kernel locking

---

### search terms للبحث عن معلومات أكثر

```
# في kernel source
git log --all --grep="lockdep" --oneline -- include/linux/lockdep.h
git log --all --grep="LOCK_STAT"

# في LKML
site:lkml.kernel.org "lockdep" "deadlock"
site:lkml.kernel.org "lock_acquire" "lock_release"

# في LWN
site:lwn.net "lockdep" "lock class"
site:lwn.net "liblockdep"

# بحث عام
linux kernel "lock_class_key" usage
linux lockdep "false positive" workaround
linux lockdep "lockdep_set_subclass" example
linux lockdep splat analysis
CONFIG_PROVE_LOCKING CONFIG_LOCKDEP difference
linux lockdep_assert_held usage driver
```

---

### أدوات مكملة لـ lockdep

| الأداة | الـ Config | الوصف |
|--------|-----------|-------|
| **lockdep** | `CONFIG_PROVE_LOCKING` | الأداة الرئيسية — deadlock detection |
| **lock stats** | `CONFIG_LOCK_STAT` | إحصائيات contention وhold-time |
| **liblockdep** | userspace | نفس الـ engine لاختبار userspace libs |
| **DEPT** | مقترح — لم يُدمج بعد | تتبع wait/event بدلاً من lock order فقط |
| **KCSAN** | `CONFIG_KCSAN` | كشف data races بدون lockdep |
| **KASAN** | `CONFIG_KASAN` | memory safety — مكمل لـ lockdep |

**ملاحظة مهمة:** lockdep و DEPT ليسا بديلاً لبعض — الـ kernel docs بتوضح إن DEPT أقل false-positives لكن lockdep لا يزال المعتمد في mainline.
## Phase 8: Writing simple module

### الفكرة

**`lock_acquire`** هي الدالة المركزية اللي بتستدعيها كل primitive لـlocking في الـkernel (spinlock، mutex، rwlock، إلخ) عشان تسجّل الـacquisition في الـlockdep graph. سنعمل **kprobe** عليها عشان نطبع معلومات عن كل lock بيتأخذ في الـsystem — اسم الـlock، نوعه (read/write/trylock)، والـprocess اللي أخده.

---

### الـModule الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on lock_acquire() — prints lock metadata whenever any lock is acquired.
 * Requires CONFIG_LOCKDEP=y and CONFIG_KPROBES=y in kernel config.
 */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit         */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe           */
#include <linux/lockdep.h>      /* struct lockdep_map — the probed arg type */
#include <linux/sched.h>        /* current, task_struct — for process name  */
#include <linux/printk.h>       /* pr_info                                  */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("kprobe on lock_acquire() to trace lockdep map acquisitions");

/*
 * lock_acquire() signature (from include/linux/lockdep.h):
 *
 *   void lock_acquire(struct lockdep_map *lock,
 *                     unsigned int subclass,
 *                     int trylock,
 *                     int read,
 *                     int check,
 *                     struct lockdep_map *nest_lock,
 *                     unsigned long ip);
 *
 * On x86-64, arguments map to registers:
 *   rdi = lock, rsi = subclass, rdx = trylock,
 *   rcx = read, r8  = check,   r9  = nest_lock
 *   (ip is on the stack — we skip it, not needed here)
 */

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve function arguments from CPU registers (x86-64 ABI) */
    struct lockdep_map *lock    = (struct lockdep_map *)regs->di;
    unsigned int        subclass = (unsigned int)regs->si;
    int                 trylock  = (int)regs->dx;
    int                 read     = (int)regs->cx;

    /* Guard: lockdep_map might be NULL in some rare paths */
    if (!lock || !lock->name)
        return 0;

    /*
     * Avoid flooding dmesg — only log locks taken in user-context tasks,
     * skip kernel threads whose name starts with '['.
     */
    if (current->comm[0] == '[')
        return 0;

    pr_info("lock_acquire: lock=%-24s subclass=%u trylock=%d read=%d"
            " task=%s pid=%d\n",
            lock->name,      /* the human-readable name of the lock class   */
            subclass,        /* nesting depth / subclass (0 = top-level)    */
            trylock,         /* 1 = trylock (non-blocking attempt)           */
            read,            /* 0=exclusive, 1=shared, 2=shared+recursive    */
            current->comm,   /* name of the process taking the lock          */
            current->pid);

    return 0; /* 0 = let the original function continue normally */
}

/* Called after lock_acquire returns — we don't need post-processing here */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* intentionally empty */
}

static struct kprobe kp = {
    .symbol_name = "lock_acquire", /* function to probe by name              */
    .pre_handler  = handler_pre,   /* called just before the function runs   */
    .post_handler = handler_post,  /* called just after the function returns */
};

static int __init lockacq_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("lockacq_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("lockacq_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit lockacq_probe_exit(void)
{
    /*
     * Must unregister before module memory is freed.
     * If we skip this, the kernel will call a handler that no longer exists
     * and panic immediately on the next lock acquisition.
     */
    unregister_kprobe(&kp);
    pr_info("lockacq_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(lockacq_probe_init);
module_exit(lockacq_probe_exit);
```

---

### شرح كل جزء

#### الـIncludes

| Header | السبب |
|---|---|
| `linux/module.h` | الـmacros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` ودوال الـregistration |
| `linux/lockdep.h` | تعريف `struct lockdep_map` اللي بنستخدمه في الـcallback |
| `linux/sched.h` | الـmacro `current` وحقل `comm` و`pid` في `task_struct` |
| `linux/printk.h` | الـmacro `pr_info` / `pr_err` |

الـincludes دي الحد الأدنى — ما فيش حاجة زيادة.

#### الـCallback `handler_pre`

بيتشغّل **قبل** دخول `lock_acquire` مباشرةً. الـarguments بنجيبها من الـregisters بدل ما نأخدها من الـstack عشان الـkprobe interceptor بيشتغل قبل ما الـstack frame يتبنى بالكامل على بعض الـarchitectures.

**الـfiltering بـ`current->comm[0] == '['`**: الـkernel threads زي `[kworker]` بيأخدوا locks بعدد هائل، فبنتجاهلهم عشان الـdmesg ميتملاش في ثوانٍ.

**قيمة الـreturn `0`**: معناها "استمر في تنفيذ الـoriginal function بشكل طبيعي". لو رجّعنا قيمة غير صفرية، الـkprobe هيـhijack الـexecution — مش ده اللي عايزينه هنا.

#### الـCallback `handler_post`

فاضية عن قصد — الـkprobe API بيطلب تعريفها بس مش لازم تعمل حاجة فيها.

#### `module_init` — `lockacq_probe_init`

بيسجّل الـkprobe بالاسم `"lock_acquire"` والـkernel بيحوّل الاسم لـaddress عن طريق `kallsyms`. لو الـfunction مش exported أو مش موجودة في الـsymbol table بيرجّع error سالب.

#### `module_exit` — `lockacq_probe_exit`

الـunregister **إجباري** هنا: لو الـmodule اتـunload من غير ما يشيل الـkprobe، الـkernel في أي `lock_acquire` جاي هيقفز لـaddress داخل الـmodule اللي اتشالت من الـmemory → **kernel panic** فوري. ده مش خيار، ده safety requirement.

---

### Makefile لبناء الـModule

```makefile
# Minimal Makefile — replace KDIR with your kernel build directory
obj-m += lockacq_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وإخراج متوقع

```bash
# Build
make

# Load
sudo insmod lockacq_probe.ko

# Watch output
sudo dmesg -w | grep lock_acquire

# Expected lines (example):
# [  42.123] lockacq_probe: planted kprobe at lock_acquire (ffffffff81a3c210)
# [  42.201] lock_acquire: lock=&mm->mmap_lock       subclass=0 trylock=0 read=1 task=bash pid=1234
# [  42.202] lock_acquire: lock=sk_lock-AF_INET       subclass=0 trylock=0 read=0 task=sshd pid=899
# [  42.203] lock_acquire: lock=rtnl_mutex            subclass=0 trylock=1 read=0 task=ip   pid=1300

# Unload
sudo rmmod lockacq_probe
```

---

### ملاحظات مهمة

- **`CONFIG_LOCKDEP=y`** مطلوب وإلا `lock_acquire` هتكون no-op macro وما فيش حاجة تـprobe.
- **`CONFIG_KPROBES=y`** مطلوب وإلا `register_kprobe` مش موجودة.
- الـmodule ده لـdebugging فقط — الـpr_info في كل lock acquisition overhead كبير في production.
- على **arm64** الـregisters بتتغير: `x0=lock, x1=subclass, x2=trylock, x3=read` — الـcode محتاج تعديل بسيط لو شغّالته على arm.
