## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **LOCKING PRIMITIVES** subsystem في الـ Linux kernel، والـ maintainers بتوعه هما Peter Zijlstra وIngo Molnar (اللي كتب الـ mutex الأصلي سنة 2004) وWill Deacon وBoqun Feng.

---

### القصة من الأول

تخيل عندك مطبخ فيه موقد واحد بس. لو اتنين طباخين حاولوا يستخدموه في نفس الوقت — هيحصل فوضى، أكل هييجي غلط، وممكن حاجة تتحرق. الحل؟ **واحد بس يستخدم الموقد في أي وقت**، والتاني ينتظر.

ده بالظبط اللي بيحله الـ **mutex** (اختصار لـ **MUTual EXclusion**).

في الـ kernel، عندك موارد مشتركة — buffer في الميموري، جهاز hardware، data structure معينة. لو اتنين **threads** أو **processes** حاولوا يكتبوا فيها في نفس الوقت، هتحصل **race condition** وهيبقى فيه corruption في البيانات.

الـ mutex بيقول: "اللي شال الـ lock، هو بس اللي يشتغل. غيره ينام وينتظر."

---

### ليه mutex وماشيش بـ spinlock؟

| الـ Primitive | بيعمل إيه لما الـ lock متاخد |
|---|---|
| **spinlock** | بيدور في loop (busy-wait) ويستنى — مفيد لو الانتظار هيكون قصير جداً |
| **mutex** | بينيّم الـ thread وييجي يصحيه لما الـ lock يتحرر — مفيد لو الانتظار ممكن يطول |

الـ mutex مش بيحرق CPU وهو بيستنى — بيعمل **sleep** حقيقي. ده بيخليه أنسب لأغلب الحالات في الـ kernel.

---

### الـ Optimistic Spinning — الحيلة الذكية

بس انتظر، النوم والصحيان ليه تكلفة (overhead). فالـ mutex في Linux بيعمل حيلة ذكية:

**قبل ما ينيّم الـ thread**، بيستنى شوية وهو "واقف" (spin) — بس بذكاء. لو الـ task اللي شايل الـ lock لسه شغال على CPU، الـ waiter بيستنى على أمل إن الـ lock هيتحرر بسرعة. لو الـ owner اتوقف (مش شغال)، وقتها بيقرر ينام.

الـ queue بتاع الـ optimistic spinning دي اسمها **OSQ** (Optimistic Spin Queue) — وده المقصود بـ `struct optimistic_spin_queue osq` جوا الـ struct.

---

### الـ struct mutex من جوا

```c
/* من mutex_types.h — النسخة العادية (non-PREEMPT_RT) */
context_lock_struct(mutex) {
    atomic_long_t       owner;      /* مين شايل الـ lock (pointer للـ task) */
    raw_spinlock_t      wait_lock;  /* يحمي الـ wait_list نفسها */
    struct optimistic_spin_queue osq; /* queue للـ optimistic spinners */
    struct list_head    wait_list;  /* قايمة الـ tasks النايمة وبتستنى */
    void               *magic;      /* debug فقط */
    struct lockdep_map  dep_map;    /* debug فقط */
};
```

- الـ **`owner`**: `atomic_long_t` بيخزن pointer للـ `task_struct` بتاع اللي شايل الـ lock. لو صفر = الـ mutex مش متاخد.
- الـ **`wait_lock`**: spinlock صغير بيحمي الـ `wait_list` نفسها من الـ race.
- الـ **`osq`**: الـ MCS-based queue للـ optimistic spinning.
- الـ **`wait_list`**: الـ tasks اللي فعلاً نامت وبتستنى، متنظمة في doubly-linked list.

---

### الـ PREEMPT_RT فرق إيه؟

في نظام الـ **Real-Time** (`CONFIG_PREEMPT_RT`)، الـ mutex بيتغير لـ **rtmutex** — نوع أقوى بيدعم **priority inheritance** (لو task عالي الـ priority بيستنى، الـ owner بياخد الـ priority بتاعه مؤقتاً عشان يخلص بسرعة). الملف بيتعامل مع الحالتين بـ `#ifdef`.

---

### الـ API الأساسية

```c
/* تهيئة */
mutex_init(&mtx);                        // تهيئة ديناميكية
DEFINE_MUTEX(mtx);                       // تعريف ستاتيك

/* الـ lock — بتنام وتستنى */
mutex_lock(&mtx);                        // lock عادي — مش ممكن يتقطع
mutex_lock_interruptible(&mtx);          // قابل للإيقاف بـ signal
mutex_lock_killable(&mtx);              // يتوقف بس بـ fatal signal
mutex_lock_io(&mtx);                    // نسخة لـ I/O waiting

/* الـ trylock — ما بينامش */
mutex_trylock(&mtx);                     // يجرب، لو مش متاح يرجع 0

/* الفك */
mutex_unlock(&mtx);

/* تنظيف */
mutex_destroy(&mtx);                    // في debug mode بس
```

---

### الـ Lock Guards — الحماية التلقائية

الملف بيعرّف **RAII-style guards** باستخدام ماكروهات الـ `cleanup.h`:

```c
DEFINE_LOCK_GUARD_1(mutex, struct mutex,
    mutex_lock(_T->lock),
    mutex_unlock(_T->lock))
```

ده بيخليك تكتب:

```c
/* الـ mutex هيتحرر تلقائياً لما تخرج من الـ scope */
CLASS(mutex, guard)(&my_mutex);
/* شغل هنا بأمان */
```

---

### الـ Lockdep — نظام كشف الـ Deadlock

الملف بيدعم **lockdep** — نظام في الـ kernel بيراقب ترتيب الـ locks ويكشف الـ deadlocks المحتملة. الـ `mutex_lock_nested()` بتحدد "subclass" الـ lock عشان الـ lockdep يفهم الـ hierarchy.

```c
/* مثال: lock للقراءة من device */
mutex_lock_nested(&dev->lock, SINGLE_DEPTH_NESTING);
```

---

### ليه الملف ده مهم؟

كل مكان في الـ kernel بيحمي data مشتركة — من الـ filesystem لـ network stack لـ device drivers — بيستخدم mutex. ده من أكتر الـ primitives استخداماً في كل الـ codebase.

---

### الملفات المرتبطة اللي لازم تعرفها

| الملف | الدور |
|---|---|
| `include/linux/mutex.h` | الـ API الرئيسية والـ macros — ده ملفنا |
| `include/linux/mutex_types.h` | تعريف `struct mutex` نفسه |
| `kernel/locking/mutex.c` | تنفيذ الـ lock/unlock الفعلي |
| `kernel/locking/mutex-debug.c` | كود الـ debug والـ validation |
| `kernel/locking/mutex.h` | header داخلي للـ implementation |
| `kernel/locking/osq_lock.c` | تنفيذ الـ Optimistic Spin Queue |
| `include/linux/osq_lock.h` | تعريف `struct optimistic_spin_queue` |
| `include/linux/lockdep.h` | نظام كشف الـ deadlocks |
| `include/linux/spinlock_types.h` | تعريف `raw_spinlock_t` المستخدمة في الـ mutex |
| `include/linux/cleanup.h` | ماكروهات الـ RAII lock guards |
| `include/linux/rtmutex.h` | الـ RT-mutex المستخدم في `PREEMPT_RT` |
| `Documentation/locking/mutex-design.rst` | التوثيق الكامل للـ design |
## Phase 2: شرح الـ Mutex Framework

### المشكلة اللي الـ Mutex بيحلها

في أي نظام multi-threaded، عندنا **shared resources** — سواء كانت data structures، hardware registers، أو buffers في الـ kernel. لو أكتر من task وصلت لنفس الـ resource في نفس الوقت من غير تنسيق، هتحصل **race condition** وبيانات هتتخرب.

الـ Linux kernel عنده أكتر من نوع lock، وكل واحد ليه use-case:

| Lock Type | يحجب الـ CPU؟ | يجوز يـsleep؟ | مكان الاستخدام |
|---|---|---|---|
| `spinlock` | يحجب (busy-wait) | لا | interrupt handlers, short critical sections |
| `mutex` | لا، يـsleep | نعم | long critical sections, kernel threads |
| `rwsem` | لا، يـsleep | نعم | read-heavy workloads |
| `semaphore` | لا، يـsleep | نعم | counting semaphores |

**الـ mutex** موجود عشان يحل مشكلة تحديدة: عايزين نحمي resource بس الـ critical section ممكن تاخد وقت طويل، ومش منطقي نخلي الـ CPU يـspin (يضيع cycles). الحل: الـ task اللي مش لاقية الـ lock تـsleep وتتوقظ لما يتحرر.

---

### الـ Solution — الأسلوب اللي الـ Kernel اتخده

الـ mutex في الـ Linux kernel بيعمل **ثلاث مراحل للحصول على الـ lock**، بالترتيب من الأسرع للأبطأ:

```
مرحلة 1: Fast Path  →  atomic CAS على الـ owner field
مرحلة 2: Mid Path   →  Optimistic Spinning (OSQ)
مرحلة 3: Slow Path  →  sleep في الـ wait_list
```

**ليه التدرج ده؟** لأن الـ context switch غالي جداً (مئات الـ nanoseconds)، فالـ kernel بيحاول يتجنبه لو الـ lock هيتحرر قريب.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Kernel Space                           │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐   │
│  │  Driver A   │  │   Driver B   │  │   Subsystem X   │   │
│  │             │  │              │  │                 │   │
│  │ mutex_lock()│  │ mutex_lock() │  │  mutex_trylock()│   │
│  └──────┬──────┘  └──────┬───────┘  └────────┬────────┘   │
│         │                │                    │            │
│         └────────────────┴────────────────────┘            │
│                          │                                  │
│              ┌───────────▼────────────┐                    │
│              │     mutex subsystem    │                    │
│              │  kernel/locking/       │                    │
│              │  mutex.c               │                    │
│              │                        │                    │
│              │  ┌─────────────────┐   │                    │
│              │  │   Fast Path     │   │                    │
│              │  │ atomic_long_cmpxchg│ │                   │
│              │  └────────┬────────┘   │                    │
│              │           │ fail       │                    │
│              │  ┌────────▼────────┐   │                    │
│              │  │   Mid Path      │   │                    │
│              │  │  OSQ Spinning   │   │                    │
│              │  └────────┬────────┘   │                    │
│              │           │ fail       │                    │
│              │  ┌────────▼────────┐   │                    │
│              │  │   Slow Path     │   │                    │
│              │  │  wait_list +    │   │                    │
│              │  │  schedule()     │   │                    │
│              │  └─────────────────┘   │                    │
│              └────────────────────────┘                    │
│                          │                                  │
│              ┌───────────▼────────────┐                    │
│              │    Scheduler (sched)   │                    │
│              │  task_struct, wait_q   │                    │
│              └────────────────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

**الـ mutex subsystem مش بيتعامل مع hardware مباشرة** — بيتعامل مع الـ scheduler وبيعتمد على الـ atomic operations اللي بيوفرها الـ architecture (ARM, x86, etc.).

---

### التشبيه الحقيقي — المطعم بالـ وقفة الواحدة

تخيل **مطعم فيه طاولة VIP واحدة بس** (= الـ shared resource):

| عنصر المطعم | الـ Kernel Concept |
|---|---|
| الطاولة الـ VIP | الـ shared resource المحمية بالـ mutex |
| الزبون الجالس | الـ task اللي holds the mutex (الـ owner) |
| الاستقبال (receptionist) | الـ mutex struct نفسه |
| لوحة "مشغول/فاضي" | الـ `owner` field (atomic_long_t) |
| ناس واقفة قدام الباب بيتفرجوا | OSQ spinners — بيستنوا قصير الأمد |
| ناس قاعدين في صالة الانتظار | wait_list — sleepers |
| الاستقبال بيندي على واحد | `wake_up()` عند unlock |
| الاستقبال نفسه محمي بـ lock صغير | الـ `wait_lock` (raw_spinlock_t) |
| رقم الطلب عند كل واحد واقف | OSQ node — كل CPU ليه node في الـ MCS queue |

**عمق التشبيه:**

- الزبون اللي **واقف قدام الباب** (OSQ spinner) شايف الطاولة بعينه — يعني هو active يراقب الـ `owner` field بنفسه، مش محتاج حد يصحيه.
- الزبون اللي **قاعد في الانتظار** (wait_list) مش شايف الطاولة — محتاج الاستقبال يجي يصحيه لما تفضى (= الـ scheduler يعمل `wake_up()`).
- **الـ wait_lock** هو المفتاح الداخلي للاستقبال — لو عايز تحجز مقعد في الانتظار أو تقوم منه، لازم الاستقبال يمسك المفتاح ده الأول (spinlock قصير).
- الـ **OSQ** ده طابور منظم للناس الواقفين — مش بوضع عشوائي، كل واحد بيستنى اللي قبله يخلص. ده يمنع الـ **thundering herd** (كل الواقفين يهجموا على الطاولة في نفس الوقت).

---

### الـ Core Data Structure

```c
// من include/linux/mutex_types.h
// context_lock_struct(mutex) يتوسع لـ struct mutex
struct mutex {
    atomic_long_t       owner;      // Fast path: LSBs = flags, rest = task_struct ptr
    raw_spinlock_t      wait_lock;  // يحمي الـ wait_list نفسها
    struct optimistic_spin_queue osq; // Mid path: MCS queue للـ spinners
    struct list_head    wait_list;  // Slow path: قائمة الـ sleepers

    // Debug only:
    void               *magic;      // CONFIG_DEBUG_MUTEXES: pointer للـ mutex نفسه
    struct lockdep_map  dep_map;    // CONFIG_DEBUG_LOCK_ALLOC: lockdep tracking
};
```

**تفاصيل الـ `owner` field — الأهم:**

الـ `owner` مش بس بيخزن الـ owner — بيخزن **flags في الـ low bits** لأن الـ `task_struct` دايماً مـaligned على الأقل على 8 bytes:

```
 63                              3    2    1    0
 ┌────────────────────────────────┬────┬────┬────┐
 │     task_struct pointer        │ ?? │HNDSTRW │
 └────────────────────────────────┴────┴────┴────┘

 bit 0: MUTEX_FLAG_WAITERS  — في الـ wait_list حد مستني
 bit 1: MUTEX_FLAG_HANDOFF  — الـ unlock لازم يحول الـ lock لأول واحد في القائمة
 bit 2: MUTEX_FLAG_PICKUP   — الـ lock اتحول، المستلم لازم يكمله
```

ده design ذكي — بدل ما يخزن الـ flags في field منفصل (محتاج lock)، بيستخدم الـ low bits في الـ pointer عشان العمليات تبقى **atomic**.

---

### تفصيل العلاقات بين الـ Structs

```
struct mutex
    │
    ├── atomic_long_t owner
    │       └── القيمة = (task_struct ptr) | flags
    │                          │
    │                          └──► struct task_struct
    │                                   (current owner)
    │
    ├── raw_spinlock_t wait_lock
    │       └── يحمي الـ wait_list من concurrent access
    │
    ├── struct optimistic_spin_queue osq
    │       └── atomic_t tail
    │               └── tail = CPU number of last spinner
    │                   كل CPU عنده optimistic_spin_node على الـ stack
    │
    └── struct list_head wait_list
            └── list of struct mutex_waiter
                    ├── struct list_head    list   (chain في الـ wait_list)
                    ├── struct task_struct *task   (من المنتظر؟)
                    └── bool               handoff (هل محتاج handoff؟)
```

---

### الـ OSQ — Optimistic Spin Queue بالتفصيل

**الـ OSQ** هو MCS lock مخصص للـ mutex spinning. الفكرة:

بدل ما كل الـ threads يـspin على نفس الـ cache line (ده بيخلي الـ cache يـbounce بين الـ CPUs)، كل CPU بيـspin على **node خاص بيه** في stack محلي.

```
CPU0 (owner)     CPU1 (spinner)    CPU2 (spinner)    CPU3 (spinner)
┌──────────┐     ┌──────────┐      ┌──────────┐      ┌──────────┐
│ holds    │     │ spinning │      │ spinning │      │ spinning │
│ mutex    │     │ on own   │      │ on own   │      │ own node │
│          │     │ node     │      │ node     │      │ (tail)   │
└──────────┘     └──────────┘      └──────────┘      └────┬─────┘
                      ▲                  ▲                 │
                      │ next             │ next            │
                      └──────────────────┘                 │
                                                           │
                 osq.tail ─────────────────────────────────┘
                           (= CPU3's encoded ID)
```

لما CPU0 يـunlock:
1. يـunlock الـ OSQ → CPU1 يحصل على الحق يجرب الـ mutex
2. CPU1 يجرب atomic CAS على الـ owner
3. لو نجح → خلص، لو فشل → يـsleep

**ليه OSQ موجود؟** لأن الـ task اللي بتـspin ممكن تكون لسه شغالة على نفس الـ CPU اللي كان بيشغل الـ owner — فالـ lock ممكن يتحرر في خلال microseconds. أرخص من context switch.

---

### الـ Preempt-RT Variant

في `CONFIG_PREEMPT_RT`، الـ mutex بيتحول لـ **rtmutex**:

```c
// CONFIG_PREEMPT_RT
struct mutex {
    struct rt_mutex_base rtmutex;  // priority-inheritance mutex
    struct lockdep_map   dep_map;
};
```

**الـ rtmutex** هو subsystem منفصل — بيحل مشكلة **priority inversion**: لو task ذات priority عالية استنت على mutex ممسوك بـ task ذات priority منخفضة، الـ rtmutex **بيرفع الـ priority** بتاعة الـ low-priority task مؤقتاً عشان تخلص بسرعة وتحرر الـ lock. ده مهم جداً في الـ real-time systems.

---

### الـ Lock Guard (Scoped Locking)

```c
// من mutex.h
DEFINE_LOCK_GUARD_1(mutex, struct mutex,
    mutex_lock(_T->lock),
    mutex_unlock(_T->lock))
```

ده بيسمح بـ **scope-based locking** اعتماداً على الـ GCC `__attribute__((cleanup))`:

```c
// الطريقة التقليدية — عرضة لـ leaks
mutex_lock(&dev->lock);
if (error) {
    mutex_unlock(&dev->lock);  // لو نسيت السطر ده، هتحصل مشكلة
    return error;
}
do_work();
mutex_unlock(&dev->lock);

// الطريقة الحديثة — automatic unlock
{
    guard(mutex)(&dev->lock);  // lock هنا
    if (error)
        return error;          // unlock تلقائي عند الخروج من الـ scope
    do_work();
}  // unlock تلقائي هنا كمان
```

الـ `cleanup.h` بيوفر الـ macro infrastructure اللي تبني عليه الـ lock guards دي.

---

### الـ devm_mutex_init — Device-Managed Mutex

```c
// mutex.h
#define devm_mutex_init(dev, mutex) \
    __devm_mutex_init(dev, __mutex_init_ret(mutex))
```

الـ **devm** prefix معناه "device managed" — الـ mutex هيتـdestroy تلقائياً لما الـ device يتـremove. ده جزء من الـ **devres subsystem** اللي بيتتبع الـ resources المرتبطة بالـ device ويـcleanup لما الـ driver يتفصل.

---

### ماذا يمتلك الـ Mutex Framework مقابل ما يفوّض

| المسؤولية | الـ Mutex Framework | يُفوَّض لـ |
|---|---|---|
| الـ Fast path (CAS) | يملكها | الـ arch atomic ops (`cmpxchg`) |
| الـ OSQ spinning | يملكها | `osq_lock.c` |
| الـ Sleep/Wake | يملكها (سياسة) | الـ Scheduler (`schedule()`, `wake_up()`) |
| الـ wait_list protection | يملكها (wait_lock) | — |
| الـ Lock ordering validation | لا يملكها | الـ **lockdep subsystem** |
| الـ Priority inheritance | لا يملكها (في RT) | الـ **rtmutex subsystem** |
| الـ Device lifetime management | لا يملكها | الـ **devres subsystem** |
| الـ Arch-specific atomics | لا يملكها | `asm/atomic.h` (ARM/x86/...) |

**الـ lockdep** هو subsystem للـ runtime lock dependency validation — بيبني graph للـ lock acquisition order وبيكشف الـ deadlocks المحتملة قبل ما تحصل. كل `struct lockdep_map` هو node في الـ graph ده.

---

### الـ API الكاملة — متى تستخدم إيه؟

```c
// أساسي — blocking حتى تحصل على الـ lock
void mutex_lock(struct mutex *lock);

// قابل للإلغاء بـ Ctrl+C (SIGINT)
int mutex_lock_interruptible(struct mutex *lock);  // returns -EINTR

// قابل للإلغاء بـ SIGKILL فقط
int mutex_lock_killable(struct mutex *lock);  // returns -EINTR

// non-blocking — يرجع 1 لو نجح، 0 لو مش متاح
int mutex_trylock(struct mutex *lock);

// unlock — لازم نفس الـ task اللي عملت lock
void mutex_unlock(struct mutex *lock);

// IO context — يشوف الـ lock كـ IO wait في الـ scheduler statistics
void mutex_lock_io(struct mutex *lock);
```

**متى تستخدم `mutex_lock_interruptible` بدل `mutex_lock`؟**

في أي ioctl أو syscall ممكن تاخد وقت طويل — دايماً استخدم الـ interruptible variant عشان الـ user-space process يقدر يـcancel العملية. لو استخدمت `mutex_lock` وحدث deadlock، الـ process مش هتستجيب للـ signals.

```c
// مثال واقعي: i2c driver
static int i2c_write_reg(struct i2c_client *client, u8 reg, u8 val)
{
    struct my_dev *dev = i2c_get_clientdata(client);
    int ret;

    // نستخدم interruptible لأن I2C ممكن تكون بطيئة
    ret = mutex_lock_interruptible(&dev->lock);
    if (ret)
        return ret;  // -EINTR

    ret = i2c_smbus_write_byte_data(client, reg, val);

    mutex_unlock(&dev->lock);
    return ret;
}
```

---

### القواعد الصارمة للـ Mutex

الـ mutex في الـ Linux kernel له **قواعد غير قابلة للكسر**:

1. **مش يتاخد في interrupt context** — لأن الـ interrupt handler مش task، ومش ممكن تـsleep
2. **بس الـ owner يـunlock** — مش زي الـ semaphore اللي أي task تقدر تـrelease
3. **مش recursive** — نفس الـ task تاخد نفس الـ mutex مرتين = deadlock فوري
4. **الـ task مش تـexit وهي ماسكة mutex** — الـ kernel بيـcheck ده عند exit
5. **مش تتـinitialize بـ memcpy/memset** — الـ internal state معقد ومش يتنقل كده
6. **المنطقة اللي فيها الـ mutex مش تتـfree وهو مقفول**
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, Config Options — Cheatsheet

#### Config Options المؤثرة في الـ mutex

| Config Option | الأثر على الـ mutex |
|---|---|
| `CONFIG_DEBUG_MUTEXES` | يفعّل `magic` pointer للكشف عن corruption، ويفرض `mutex_destroy()` الحقيقية |
| `CONFIG_DEBUG_LOCK_ALLOC` | يفعّل `dep_map` (lockdep integration)، يضيف nested/subclass variants |
| `CONFIG_MUTEX_SPIN_ON_OWNER` | يفعّل الـ `osq` field — optimistic spinning قبل النوم |
| `CONFIG_PREEMPT_RT` | يستبدل الـ mutex كلياً بـ `rt_mutex_base` (priority inheritance) |
| `CONFIG_LOCKDEP` | يتحكم في تفعيل lockdep tracking بشكل عام |

#### Lock Acquisition Variants

| Function / Macro | السلوك عند الـ contention | يُرجع |
|---|---|---|
| `mutex_lock()` | ينام — uninterruptible | `void` |
| `mutex_lock_interruptible()` | ينام — قابل للمقاطعة بـ signal | `int` (0 أو `-EINTR`) |
| `mutex_lock_killable()` | ينام — قابل للمقاطعة بـ fatal signal فقط | `int` (0 أو `-EINTR`) |
| `mutex_lock_io()` | مثل `mutex_lock` لكن يحسب الوقت كـ I/O wait | `void` |
| `mutex_trylock()` | لا ينام — يرجع فوراً | `1` (acquired) أو `0` (busy) |
| `mutex_unlock()` | يحرر الـ lock ويوقظ أول waiter | `void` |

#### OSQ State Values

| Value | المعنى |
|---|---|
| `OSQ_UNLOCKED_VAL` (0) | الـ queue فاضية — مفيش spinners |
| أي قيمة غير صفر | رقم الـ CPU الـ tail spinner مُرمَّز |

---

### 1. الـ Structs المهمة

#### `struct mutex` — البنية الأساسية

**الغرض:** يمثل الـ mutex نفسه — sleeping lock لا يُستخدم إلا في process context.

##### نسخة `!CONFIG_PREEMPT_RT` (الـ standard)

```c
/* defined via context_lock_struct(mutex) macro in mutex_types.h */
struct mutex {
    atomic_long_t       owner;      /* encoded: owner task_struct* + flags */
    raw_spinlock_t      wait_lock;  /* protects wait_list */
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    struct optimistic_spin_queue osq; /* MCS spinner queue */
#endif
    struct list_head    wait_list;  /* list of blocked tasks */
#ifdef CONFIG_DEBUG_MUTEXES
    void               *magic;      /* points to itself — detects corruption */
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;    /* lockdep class tracking */
#endif
};
```

##### شرح كل field

| Field | النوع | الشرح |
|---|---|---|
| `owner` | `atomic_long_t` | الـ pointer لـ `task_struct` الـ owner مُرمَّز فيه flags في الـ low bits |
| `wait_lock` | `raw_spinlock_t` | spinlock يحمي `wait_list` — يُمسك لفترة قصيرة جداً |
| `osq` | `optimistic_spin_queue` | قائمة انتظار الـ optimistic spinners قبل ما ينام الـ task |
| `wait_list` | `struct list_head` | قائمة الـ tasks النايمة تنتظر الـ mutex |
| `magic` | `void *` | debug فقط — يُشير لنفس الـ struct، لو اتغير = corruption |
| `dep_map` | `struct lockdep_map` | lockdep فقط — يخزن اسم الـ lock وكلاسه |

##### الـ flags المُرمَّزة في `owner`

الـ `owner` field مش مجرد pointer — الـ low 3 bits بيتستخدموا كـ flags:

| Bit | الاسم (internal) | المعنى |
|---|---|---|
| Bit 0 | `MUTEX_FLAG_WAITERS` | في waiters في الـ `wait_list` |
| Bit 1 | `MUTEX_FLAG_HANDOFF` | الـ owner الحالي لازم يسلّم الـ lock لأول waiter |
| Bit 2 | `MUTEX_FLAG_PICKUP` | الـ lock اتسلّم للـ waiter، بس مستنيه يـ pickup |

لما يبقى `owner == 0` كلياً → الـ mutex مُحرَّر.

##### نسخة `CONFIG_PREEMPT_RT`

```c
struct mutex {
    struct rt_mutex_base    rtmutex;  /* full RT mutex with PI */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map      dep_map;
#endif
};
```

الـ RT variant بيستخدم `rt_mutex_base` اللي بيدعم **priority inheritance** — يعني لو task عالي الـ priority استنى، الـ kernel يرفع priority الـ owner مؤقتاً.

---

#### `struct optimistic_spin_queue` — الـ MCS Spinner Queue

**الغرض:** قائمة الـ CPUs اللي بتعمل optimistic spinning (busy-wait قصير) بدل ما تنام فوراً.

```c
struct optimistic_spin_queue {
    atomic_t tail;  /* encoded CPU# of last spinner, 0 = empty */
};
```

| Field | الشرح |
|---|---|
| `tail` | رقم CPU الـ spinner الأخير في الـ queue مُرمَّز atomically. القيمة 0 تعني مفيش spinners |

**الفكرة:** لو الـ mutex محتجز لفترة قصيرة جداً، أوفر من context switch إن الـ CPU يبقى يـ spin بأسلوب MCS (كل CPU بـ spin على node خاصه، مش shared variable) لحد ما الـ owner يحرر الـ lock.

---

#### `struct lockdep_map` — الـ Lockdep Integration

**الغرض:** يُمكّن lockdep من تتبع الـ lock class وكشف deadlocks.

الـ fields الأساسية (من `lockdep_types.h`):

| Field | الشرح |
|---|---|
| `name` | اسم الـ lock كـ string (للـ debug output) |
| `wait_type_inner` | نوع الانتظار — للـ mutex = `LD_WAIT_SLEEP` |
| `key` | الـ `lock_class_key` الخاص بالـ lock |

---

### 2. Struct Relationship Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           struct mutex                  │
                    │                                         │
                    │  atomic_long_t  owner ──────────────────┼──► task_struct* (+ flags in low bits)
                    │                                         │
                    │  raw_spinlock_t wait_lock               │
                    │                                         │
                    │  struct list_head wait_list ────────────┼──► mutex_waiter ──► mutex_waiter ──► ...
                    │                                         │        │
                    │  [MUTEX_SPIN_ON_OWNER]                  │        └──► task_struct* (sleeping task)
                    │  struct optimistic_spin_queue osq ──────┼──► (atomic tail → optimistic_spin_node)
                    │          │                              │
                    │          └── atomic_t tail              │
                    │                                         │
                    │  [DEBUG_MUTEXES]                        │
                    │  void *magic ───────────────────────────┼──► (points back to itself)
                    │                                         │
                    │  [DEBUG_LOCK_ALLOC]                     │
                    │  struct lockdep_map dep_map ────────────┼──► lock_class_key
                    └─────────────────────────────────────────┘

          [CONFIG_PREEMPT_RT variant]
                    ┌─────────────────────────────────────────┐
                    │           struct mutex (RT)             │
                    │                                         │
                    │  struct rt_mutex_base rtmutex ──────────┼──► (PI wait tree, owner task_struct*)
                    │  struct lockdep_map dep_map             │
                    └─────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagram

```
  CREATION
  ────────
  DEFINE_MUTEX(name)              ← static compile-time init
      │
      ├── owner = ATOMIC_LONG_INIT(0)       (unlocked)
      ├── wait_lock = __RAW_SPIN_LOCK_UNLOCKED
      └── wait_list = LIST_HEAD_INIT

  OR at runtime:
  mutex_init(&m)
      │
      └── __mutex_init(&m, "m", &__key)
              │
              ├── [!DEBUG_LOCK_ALLOC] mutex_init_generic()
              │       └── initializes fields directly
              │
              └── [DEBUG_LOCK_ALLOC] mutex_init_lockep()
                      └── lockdep_init_map() + field init

  MANAGED (devm):
  devm_mutex_init(dev, &m)
      │
      ├── mutex_init(&m)
      └── [DEBUG_MUTEXES] __devm_mutex_init() registers mutex_destroy() as devm action

  ─────────────────────────────────────────────────────────────────

  ACQUISITION (fast path — uncontended)
  ──────────────────────────────────────
  mutex_lock(&m)
      │
      └── try atomic CAS: owner 0 → current task_struct*
              │
              └── SUCCESS → return (lock held)

  ACQUISITION (slow path — contended)
  ─────────────────────────────────────
  mutex_lock(&m)
      │
      └── fast path fails
              │
              ├── [MUTEX_SPIN_ON_OWNER] osq_lock(&m.osq)
              │       │
              │       ├── join MCS spinner queue
              │       ├── spin while owner is running on CPU
              │       │       └── retry fast-path CAS each iteration
              │       └── owner scheduled out → osq_unlock() → go to sleep
              │
              └── spin_lock(&m.wait_lock)
                      │
                      ├── enqueue mutex_waiter on m.wait_list
                      ├── spin_unlock(&m.wait_lock)
                      └── schedule() → task sleeps
                              │
                              └── woken by mutex_unlock()
                                      └── retry acquisition

  ─────────────────────────────────────────────────────────────────

  RELEASE
  ───────
  mutex_unlock(&m)
      │
      ├── fast path: atomic store owner = 0 (if no waiters/flags)
      │       └── done
      │
      └── slow path: (WAITERS or HANDOFF flag set)
              │
              ├── spin_lock(&m.wait_lock)
              ├── pick first waiter from wait_list
              ├── [HANDOFF] set PICKUP flag, set waiter as next owner
              ├── spin_unlock(&m.wait_lock)
              └── wake_up_process(waiter->task)

  ─────────────────────────────────────────────────────────────────

  TEARDOWN
  ────────
  [DEBUG_MUTEXES]  mutex_destroy(&m)
      │
      └── asserts mutex is unlocked, clears magic field
              └── marks struct as destroyed to catch use-after-free

  [devm] → called automatically when device is removed
```

---

### 4. Call Flow Diagrams

#### 4.1 mutex_lock() — Full Flow

```
  caller: mutex_lock(&m)
      │
      ▼
  [macro, DEBUG_LOCK_ALLOC]
  mutex_lock_nested(lock, subclass=0)
      │
      ▼
  __mutex_lock(lock, state=TASK_UNINTERRUPTIBLE, ...)
      │
      ├─► lockdep_acquire()           ← record lock acquisition for deadlock detection
      │
      ├─► fast path:
      │       atomic_long_try_cmpxchg(&lock->owner, 0, current)
      │           └── SUCCESS → return 0
      │
      ├─► optimistic spin (if MUTEX_SPIN_ON_OWNER):
      │       osq_lock(&lock->osq)
      │           └── for each iteration:
      │               owner = READ_ONCE(lock->owner) & ~flags
      │               if owner && owner_is_on_cpu(owner) → spin
      │               else → break → go sleep
      │       osq_unlock(&lock->osq)
      │
      └─► sleep path:
              raw_spin_lock(&lock->wait_lock)
                  enqueue waiter on lock->wait_list
                  set MUTEX_FLAG_WAITERS in lock->owner
              raw_spin_unlock(&lock->wait_lock)
              set_current_state(TASK_UNINTERRUPTIBLE)
              schedule()
                  └── [woken] → retry fast-path CAS
                                  └── SUCCESS → return 0
```

#### 4.2 mutex_lock_interruptible() — الفرق

```
  mutex_lock_interruptible(&m)
      │
      └── __mutex_lock(state=TASK_INTERRUPTIBLE, ...)
              │
              └── schedule()
                      │
                      ├── [woken by unlock] → retry → return 0
                      └── [signal received] → dequeue waiter → return -EINTR
```

#### 4.3 mutex_trylock()

```
  mutex_trylock(&m)
      │
      └── atomic_long_try_cmpxchg(&lock->owner, 0, current)
              ├── SUCCESS → lockdep_acquire() → return 1
              └── FAIL    → return 0   (no sleep, no spin)
```

#### 4.4 mutex_unlock() — Full Flow

```
  caller: mutex_unlock(&m)
      │
      ▼
  __mutex_unlock(lock)
      │
      ├─► lockdep_release()
      │
      ├─► fast path:
      │       owner = current (no flags)
      │       atomic_long_cmpxchg(&lock->owner, current, 0)
      │           └── SUCCESS → return
      │
      └─► slow path (flags set):
              raw_spin_lock(&lock->wait_lock)
                  │
                  ├── [HANDOFF flag] → pick waiter → set owner = waiter->task | PICKUP
                  │                                   wake_up_process(waiter->task)
                  │
                  └── [WAITERS, no HANDOFF] → clear owner
                                              wake_up_process(first waiter)
              raw_spin_unlock(&lock->wait_lock)
```

#### 4.5 devm_mutex_init() Flow

```
  devm_mutex_init(dev, &m)
      │
      ├── __mutex_init_ret(&m)
      │       └── mutex_init(&m)   ← initializes the mutex
      │               └── returns &m
      │
      └── __devm_mutex_init(dev, &m)
              │
              └── [DEBUG_MUTEXES]
                      devm_add_action(dev, mutex_destroy, &m)
                          └── mutex_destroy() called when device released
```

#### 4.6 Lock Guard (Scoped Locking) Flow

```c
/* Usage: */
{
    CLASS(mutex, guard)(&m);   /* mutex_lock(&m) called here */
    /* ... critical section ... */
}                              /* scope ends → mutex_unlock(&m) auto-called */

/* Trylock variant: */
CLASS(mutex_try, guard)(&m);
if (!guard)                    /* trylock failed */
    return -EBUSY;

/* Interruptible variant: */
CLASS(mutex_intr, guard)(&m);
if (!guard)
    return -EINTR;
```

```
  CLASS(mutex, guard)(&m)
      │
      ├── DEFINE_LOCK_GUARD_1 expansion:
      │       _T->lock = &m
      │       mutex_lock(_T->lock)         ← acquire
      │
      └── [scope exits / __free() triggered]
              mutex_unlock(_T->lock)       ← release
```

---

### 5. Locking Strategy

#### 5.1 الـ Locks الموجودة في الـ mutex

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `mutex->owner` | `atomic_long_t` (lockless) | حالة الـ mutex (locked/unlocked) والـ owner — يتعدّل بـ atomic CAS |
| `mutex->wait_lock` | `raw_spinlock_t` | الـ `wait_list` وأي تعديل على الـ `owner` flags في الـ slow path |
| `mutex->osq` | MCS queue (lockless) | قائمة الـ optimistic spinners — كل spinner بـ spin على node خاصه |

#### 5.2 Lock Ordering — الترتيب

```
  [الـ caller's mutex]
      │
      └── يُمسك أولاً
              │
              └── [mutex->wait_lock] (raw spinlock)
                      │
                      └── يُمسك ثانياً — فقط في الـ slow path
                              └── يُحرَّر قبل ما نخرج من الـ slow path
```

**القاعدة الذهبية:** الـ `wait_lock` بيتمسك لأقل وقت ممكن — فقط لتعديل `wait_list` والـ flags. لا يجوز يتمسك `wait_lock` من داخل الـ mutex نفسه (deadlock).

#### 5.3 Atomic Owner Encoding — الـ Lockless Fast Path

```
  owner field (atomic_long_t):
  ┌──────────────────────────────────────────┬───┬───┬───┐
  │   task_struct pointer (aligned, >>3)     │ P │ H │ W │
  └──────────────────────────────────────────┴───┴───┴───┘
                                              │   │   │
                                              │   │   └── Bit 0: WAITERS
                                              │   └────── Bit 1: HANDOFF
                                              └────────── Bit 2: PICKUP
```

الـ `task_struct` مضمون aligned على 8 bytes على الأقل، فالـ low 3 bits دايماً 0 في الـ pointer → تتستخدم كـ flags بدون overhead.

الـ fast path بيعمل CAS واحد فقط:
```c
/* lock fast path */
atomic_long_try_cmpxchg(&lock->owner, 0, (long)current);

/* unlock fast path */
atomic_long_cmpxchg(&lock->owner, (long)current, 0);
```

لو الـ CAS فشل → slow path → يمسك `wait_lock`.

#### 5.4 Optimistic Spin — متى يكون مفيد

```
  Scenario: mutex held for 1µs, context switch = 10µs

  Without OSQ:
  ├── task A: lock fails → sleep → wake = 10µs overhead
  └── total: ~11µs

  With OSQ:
  ├── task A: spins ~1µs on osq → owner releases → CAS succeeds
  └── total: ~1µs  (10x faster)

  Scenario: mutex held for 100µs

  With OSQ:
  ├── task A: spins → owner scheduled out → osq_unlock → sleep
  └── OSQ avoids wasting CPU on long waits automatically
```

الـ OSQ بيكتشف لو الـ owner اتشال من CPU (مش running) ويوقف الـ spinning فوراً.

#### 5.5 RT Mutex — اختلاف الـ Locking

في `CONFIG_PREEMPT_RT`، الـ mutex بيستخدم `rt_mutex_base` اللي بيعمل **Priority Inheritance**:

```
  task_high (priority 90) waiting for mutex
      │
      └── mutex held by task_low (priority 10)
              │
              └── kernel raises task_low priority → 90  (temporarily)
                      └── task_low finishes faster → releases → task_high runs
```

ده بيمنع **Priority Inversion** في الـ RT systems.

---
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Initialization & Lifecycle

| Function / Macro | Signature (مختصرة) | الغرض |
|---|---|---|
| `mutex_init(mutex)` | macro → `__mutex_init(lock, name, key)` | تهيئة الـ mutex في runtime |
| `mutex_init_with_key(mutex, key)` | macro → `__mutex_init(lock, name, key)` | تهيئة مع lockdep key صريح |
| `DEFINE_MUTEX(name)` | macro | تعريف وتهيئة static mutex |
| `__MUTEX_INITIALIZER(name)` | macro | static initializer للـ struct |
| `mutex_destroy(lock)` | `void mutex_destroy(struct mutex *)` | تنظيف بعد الاستخدام (debug only) |
| `devm_mutex_init(dev, mutex)` | macro → `__devm_mutex_init(dev, mutex)` | تهيئة مربوطة بـ device lifetime |
| `mutex_init_generic(lock)` | `void mutex_init_generic(struct mutex *)` | تهيئة فعلية بدون lockdep |
| `mutex_init_lockep(lock, name, key)` | `void mutex_init_lockep(...)` | تهيئة فعلية مع lockdep |

#### Locking — Acquire

| Function | يبلوك؟ | قابل للمقاطعة؟ | ملاحظة |
|---|---|---|---|
| `mutex_lock(lock)` | نعم | لا | TASK_UNINTERRUPTIBLE |
| `mutex_lock_interruptible(lock)` | نعم | أي signal | يرجع `-EINTR` |
| `mutex_lock_killable(lock)` | نعم | fatal signals فقط | يرجع `-EINTR` |
| `mutex_lock_io(lock)` | نعم | لا | يحسب الـ task كـ IO-waiting |
| `mutex_trylock(lock)` | لا | — | يرجع 1=نجح، 0=فشل |
| `mutex_lock_nested(lock, subclass)` | نعم | لا | lockdep nesting |
| `mutex_lock_interruptible_nested(lock, sub)` | نعم | نعم | lockdep nesting + interruptible |
| `mutex_lock_io_nested(lock, sub)` | نعم | لا | IO accounting + nesting |
| `mutex_lock_nest_lock(lock, nest)` | نعم | لا | explicit nest_lock annotation |
| `mutex_lock_killable_nested(lock, sub)` | نعم | fatal | nesting + killable |

#### Unlocking & Helpers

| Function | الغرض |
|---|---|
| `mutex_unlock(lock)` | تحرير الـ mutex |
| `mutex_is_locked(lock)` | استعلام: هل الـ mutex مقفول؟ |
| `mutex_get_owner(lock)` | إرجاع عنوان الـ task المالك |
| `atomic_dec_and_mutex_lock(cnt, lock)` | atomic decrement → lock لو وصل لصفر |

#### Lock Guards (C cleanup API)

| Guard Class | يحصل lock | يطلق lock | شرط نجاح |
|---|---|---|---|
| `CLASS(mutex, g)(lock)` | `mutex_lock` | `mutex_unlock` | دائماً |
| `CLASS(mutex_try, g)(lock)` | `mutex_trylock` | `mutex_unlock` | لو اكتسب |
| `CLASS(mutex_intr, g)(lock)` | `mutex_lock_interruptible` | `mutex_unlock` | `ret == 0` |
| `CLASS(mutex_init, g)(lock)` | `mutex_init` | — | دائماً |

---

### Category 1: Initialization & Lifecycle

الهدف من هذه المجموعة هو إعداد الـ `struct mutex` وتجهيزه للاستخدام. كل mutex لازم يتهيأ قبل أي استخدام — استخدام `memset(0)` غير مسموح.

---

#### `__mutex_init_generic` (internal)

```c
static void __mutex_init_generic(struct mutex *lock)
{
    atomic_long_set(&lock->owner, 0);          // owner = NULL, flags = 0
    raw_spin_lock_init(&lock->wait_lock);       // init wait_lock spinlock
    INIT_LIST_HEAD(&lock->wait_list);           // empty wait queue
#ifdef CONFIG_MUTEX_SPIN_ON_OWNER
    osq_lock_init(&lock->osq);                 // init MCS optimistic spin queue
#endif
    debug_mutex_init(lock);                    // set magic pointer for debug
}
```

الدالة الداخلية الأساسية اللي كل طرق التهيئة بتمر عليها. بتصفّر الـ `owner` الـ atomic بالكامل، وتهيئ الـ `wait_lock` الـ raw spinlock، وتفرّغ قائمة الانتظار. لو كان `CONFIG_MUTEX_SPIN_ON_OWNER` شغال، بتهيئ كمان الـ OSQ (MCS lock) للـ optimistic spinning.

**لا يُصدَّر للموديولات** — internal فقط.

---

#### `mutex_init_generic`

```c
void mutex_init_generic(struct mutex *lock);
```

**الغرض:** wrapper بسيط لـ `__mutex_init_generic`، بيُصدَّر لو `CONFIG_DEBUG_LOCK_ALLOC` مش شغال.

- **lock**: الـ mutex المراد تهيئته.
- **Return**: void.
- **Caller**: `__mutex_init()` macro path بدون lockdep.

---

#### `mutex_init_lockep`

```c
void mutex_init_lockep(struct mutex *lock, const char *name,
                       struct lock_class_key *key);
```

**الغرض:** نفس `mutex_init_generic` لكن مع تسجيل lockdep. بتستدعي `debug_check_no_locks_freed()` للتحقق إن الـ lock مش محفوظ حالياً قبل إعادة التهيئة، وبعدين `lockdep_init_map_wait()` بـ `LD_WAIT_SLEEP` لأن الـ mutex sleeping lock.

- **lock**: الـ mutex.
- **name**: اسم نصي للـ lockdep graph (عادةً `#mutex` من الـ macro).
- **key**: الـ `lock_class_key` الـ static — بيحدد الـ lock class في lockdep.
- **Return**: void.
- **Key Detail**: الـ `LD_WAIT_SLEEP` annotation مهمة — بتخلي lockdep يعرف إن الـ mutex ممنوع يتمسك في atomic context.

---

#### `mutex_init(mutex)` — Macro

```c
#define mutex_init(mutex)                               \
do {                                                    \
    static struct lock_class_key __key;                 \
    __mutex_init((mutex), #mutex, &__key);              \
} while (0)
```

**الغرض:** الـ API العام للتهيئة في runtime. بيعمل `static lock_class_key` محلي تلقائياً لكل موقع استدعاء في الكود — ده الـ trick اللي بيخلي lockdep يميّز بين locks مختلفة حتى لو نفس النوع.

- **لازم تستخدم الـ macro مش `__mutex_init` مباشرة** عشان الـ `static __key` يتعمل per-callsite.
- ممنوع تهيئ lock مقفول.

---

#### `DEFINE_MUTEX(mutexname)` — Macro

```c
#define DEFINE_MUTEX(mutexname) \
    struct mutex mutexname = __MUTEX_INITIALIZER(mutexname)
```

**الغرض:** تعريف وتهيئة mutex static في compile time. بيعمل `struct mutex` وبيملي فيه `__MUTEX_INITIALIZER` اللي بيصفّر الـ owner ويهيئ الـ wait_lock والـ wait_list.

---

#### `devm_mutex_init(dev, mutex)` — Macro

```c
#define devm_mutex_init(dev, mutex) \
    __devm_mutex_init(dev, __mutex_init_ret(mutex))
```

**الغرض:** بيهيئ الـ mutex وبيربطه بـ device lifecycle. لو `CONFIG_DEBUG_MUTEXES` شغال، بيسجّل cleanup action في الـ devres فريمورك عشان `mutex_destroy()` تتنادى تلقائياً لما الـ device يتحرر.

- **dev**: الـ `struct device *` المرتبط.
- **mutex**: الـ mutex المراد تهيئته.
- **Return**: `int` — صفر أو error code.
- **Key Detail**: لو `CONFIG_DEBUG_MUTEXES` مش شغال، `mutex_destroy` no-op فبالتالي الـ `__devm_mutex_init` بترجع 0 مباشرة.

---

#### `mutex_destroy`

```c
/* CONFIG_DEBUG_MUTEXES=y */
extern void mutex_destroy(struct mutex *lock);

/* CONFIG_DEBUG_MUTEXES=n */
static inline void mutex_destroy(struct mutex *lock) {}
```

**الغرض:** في debug builds بتعمل sanity check وبتصفّر الـ magic pointer عشان تكتشف أي استخدام للـ mutex بعد تحريره. في production (بدون `CONFIG_DEBUG_MUTEXES`) هي no-op كاملة.

- **Caller**: يُنادى بعد انتهاء استخدام الـ mutex نهائياً.
- **Key Detail**: ما بتحرر memory — بس بتعمل poison للـ struct في debug mode.

---

### Category 2: Fast Path Internals

هذه الدوال الداخلية هي قلب أداء الـ mutex. الـ fast path بيحاول يكتسب ويطلق الـ lock باستخدام `atomic_long_try_cmpxchg` بدون أي spinlock.

---

#### `__mutex_trylock_fast`

```c
static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;
    unsigned long zero = 0UL;

    if (atomic_long_try_cmpxchg_acquire(&lock->owner, &zero, curr))
        return true;
    return false;
}
```

**الغرض:** يحاول يعمل lock بـ single atomic CAS. لو الـ `owner == 0` (unlocked, no flags) يكتب `current` كـ owner مع acquire barrier. **Optimistic فقط** — بيفشل لو في أي flag مضبوط حتى لو الـ lock فاضي.

- **Return**: `true` لو نجح، `false` لو في contention أو flags.
- **لا يُنادى إلا من الـ fast path في `mutex_lock` و`mutex_lock_interruptible` و`mutex_lock_killable`.**

---

#### `__mutex_unlock_fast`

```c
static __always_inline bool __mutex_unlock_fast(struct mutex *lock)
{
    unsigned long curr = (unsigned long)current;
    return atomic_long_try_cmpxchg_release(&lock->owner, &curr, 0UL);
}
```

**الغرض:** يحاول يطلق الـ lock بـ single atomic CAS. لو `owner == current` (مفيش flags) يضبط `owner = 0` مع release barrier. لو في flags (WAITERS أو HANDOFF) بيفشل ويروح الـ slow path.

- **Return**: `true` لو نجح، `false` لو في waiters أو HANDOFF.

---

#### `__mutex_trylock_common`

```c
static inline struct task_struct *
__mutex_trylock_common(struct mutex *lock, bool handoff)
```

**الغرض:** الـ core trylock logic اللي بتتعامل مع الـ flags. بتشوف `owner` field وبتحاول CAS في loop:

- لو `owner != 0` وفي `MUTEX_FLAG_PICKUP` وهو الـ `current` → يمسح الـ PICKUP ويكتسب.
- لو `handoff=true` وفي `MUTEX_FLAG_HANDOFF` → يضيف HANDOFF (بيبعت إشارة للـ unlock إنه مستعد).
- لو `owner == 0` → يضع `current` كـ owner.

```
owner = atomic_long_read(&lock->owner)
loop:
  flags = owner & MUTEX_FLAGS
  task  = owner & ~MUTEX_FLAGS
  if task != 0:
    if PICKUP && task == current:  clear PICKUP, set task=current → CAS
    elif handoff && !HANDOFF:      set HANDOFF flag → CAS (signal unlock)
    else: break (locked by someone else)
  else:
    task = current → CAS
  if CAS succeeds && task == current: return NULL (success)
return __owner_task(owner)  // failure: return current owner
```

- **Return**: `NULL` لو اكتسب، أو الـ `task_struct *` للمالك لو فشل.
- **بتُنادى من:** `__mutex_trylock()` و`__mutex_trylock_or_handoff()` و`__mutex_trylock_or_owner()`.

---

#### `__mutex_trylock`

```c
static inline bool __mutex_trylock(struct mutex *lock)
{
    return !__mutex_trylock_common(lock, false);
}
```

Wrapper بسيط — **handoff=false**. يُستخدم عند محاولة الاكتساب بدون إشارة handoff.

---

#### `__mutex_trylock_or_handoff`

```c
static inline bool __mutex_trylock_or_handoff(struct mutex *lock, bool handoff)
{
    return !__mutex_trylock_common(lock, handoff);
}
```

يُستخدم في الـ sleep loop بعد الاستيقاظ: لو الـ waiter هو الـ first في القائمة يمرر `handoff=true` عشان يطلب handoff من الـ unlocker.

---

### Category 3: Optimistic Spinning (MCS/OSQ)

هذه المجموعة تمثل آلية الـ **adaptive spinning** — بدل ما الـ task ينام فوراً، يعمل busy-wait لفترة قصيرة لو الـ owner شغال على CPU تانية.

---

#### `mutex_can_spin_on_owner`

```c
static inline int mutex_can_spin_on_owner(struct mutex *lock)
```

**الغرض:** الـ initial check قبل ما ندخل في الـ spin loop. بيشيك إن:
1. `need_resched()` مش مضبوط (الـ scheduler مش محتاجنا نتنازل).
2. الـ lock owner موجود وشغال على CPU (`owner_on_cpu(owner)`).

لو الـ owner مش موجود (المـ mutex حُرر)، يرجع `1` عشان نحاول trylock في الـ spin path مباشرة.

- **Preemption**: لازم تكون disabled قبل النداء (بيتحقق بـ `lockdep_assert_preemption_disabled()`).
- **Return**: `1` لو ممكن نسبين، `0` لو لازم ننام.

---

#### `mutex_spin_on_owner`

```c
static noinline
bool mutex_spin_on_owner(struct mutex *lock, struct task_struct *owner,
                         struct ww_acquire_ctx *ww_ctx,
                         struct mutex_waiter *waiter)
```

**الغرض:** الـ inner spin loop الفعلي. بيتفرج على `lock->owner` في loop وبيتحقق كل iteration إن:
1. الـ owner لسه هو نفسه (ما تغيرش).
2. الـ owner لسه شغال على CPU (`owner_on_cpu`).
3. `need_resched()` ما اتضبطش.
4. لو ww_ctx: `ww_mutex_spin_on_owner()` ما قالش نوقف.

بيستخدم `barrier()` للـ compiler barrier و`cpu_relax()` للـ hardware hint.

- **Return**: `true` لو المـ owner حرر الـ lock (تقدر تحاول trylock)، `false` لو وقفنا.
- **`noinline`**: عشان يظهر واضح في perf profiles.

---

#### `mutex_optimistic_spin`

```c
static __always_inline bool
mutex_optimistic_spin(struct mutex *lock, struct ww_acquire_ctx *ww_ctx,
                      struct mutex_waiter *waiter)
```

**الغرض:** الـ outer optimistic spin loop. الفلسفة: لو الـ owner شغال على CPU تانية، من المرجح إنه هيحرر الـ lock قريباً — فأهون نـ spin من ما نعمل context switch كامل.

**Flow:**
```
if !waiter:
    if !mutex_can_spin_on_owner() → goto fail
    if !osq_lock(&lock->osq)     → goto fail  // join MCS queue
loop:
    owner = __mutex_trylock_or_owner(lock)
    if !owner: break  // got it!
    if !mutex_spin_on_owner(lock, owner, ww_ctx, waiter):
        goto fail_unlock
    cpu_relax()

if !waiter: osq_unlock(&lock->osq)
return true

fail_unlock: osq_unlock if !waiter
fail:
    if need_resched(): schedule_preempt_disabled()
return false
```

- **waiter != NULL**: لما الـ task بقى في wait_list وصحي ليه spin مباشرة بدون OSQ.
- **OSQ (MCS)**: بيضمن إن spinner واحد بس بيـ spin على الـ lock field، والباقيين بيـ spin على node خاصة بيهم — بيقلل cache contention.
- **`CONFIG_MUTEX_SPIN_ON_OWNER=n`**: الـ function بترجع `false` دائماً (no-op implementation).

---

### Category 4: Slow Path — Lock Acquisition

الـ slow path بيُنادى لما الـ fast path والـ optimistic spin فشلوا. الـ task بيضيف نفسه للـ wait queue وبينام.

---

#### `__mutex_lock_common` — الـ Core Slowpath

```c
static __always_inline int __sched
__mutex_lock_common(struct mutex *lock, unsigned int state,
                    unsigned int subclass, struct lockdep_map *nest_lock,
                    unsigned long ip, struct ww_acquire_ctx *ww_ctx,
                    const bool use_ww_ctx)
```

**الغرض:** الدالة الأم التي تستدعيها كل طرق الـ lock. بتمر بمراحل:

**Parameters:**
- **lock**: الـ mutex المراد اكتسابه.
- **state**: حالة الـ task أثناء الانتظار (`TASK_UNINTERRUPTIBLE` / `TASK_INTERRUPTIBLE` / `TASK_KILLABLE`).
- **subclass**: الـ lockdep subclass للـ nesting annotation.
- **nest_lock**: lockdep map للـ explicit nesting.
- **ip**: return address للـ lockdep tracing (`_RET_IP_`).
- **ww_ctx**: الـ wound-wait context، أو NULL للـ plain mutex.
- **use_ww_ctx**: flag يفصل بين WW و plain path.

**Return**: `0` نجاح، `-EINTR` عند signal، `-EDEADLK` في WW context.

**Pseudocode Flow:**
```
preempt_disable()
mutex_acquire_nest(lockdep)
trace_contention_begin()

// Fast attempts before sleeping
if __mutex_trylock() || mutex_optimistic_spin():
    lock_acquired(); preempt_enable(); return 0

raw_spin_lock_irqsave(&lock->wait_lock)

// Last trylock under wait_lock
if __mutex_trylock():
    goto skip_wait

// Setup waiter
waiter.task = current
if use_ww_ctx: add in stamp order (WW)
else:           list_add_tail (FIFO)

set_current_state(state)

// Main sleep loop
for (;;):
    if __mutex_trylock(): goto acquired
    if signal_pending_state(): ret = -EINTR; goto err
    if ww_ctx: check kill conditions

    raw_spin_unlock_irqrestore_wake()  // release wait_lock + wake pending
    schedule_preempt_disabled()        // SLEEP

    // woke up
    set_current_state(state)
    if __mutex_trylock_or_handoff(lock, first): break

    if first && mutex_optimistic_spin(): break  // spin as first waiter

    raw_spin_lock_irqsave()

acquired:
    __set_current_state(TASK_RUNNING)
    __mutex_remove_waiter()

skip_wait:
    lock_acquired(lockdep)
    raw_spin_unlock_irqrestore_wake()
    preempt_enable()
    return 0

err:
    __set_current_state(TASK_RUNNING)
    __mutex_remove_waiter()
    mutex_release(lockdep)
    preempt_enable()
    return ret
```

**Key Details:**
- بيستخدم `raw_spin_lock_irqsave` على `wait_lock` داخلياً — الـ wait_lock بيحمي الـ wait_list وعمليات handoff.
- `schedule_preempt_disabled()` بدل `schedule()` عشان الـ preemption لازم تتعمل manually.
- الـ FIFO ordering للـ waiters بيمنع starvation في الحالة العادية.
- في WW mode، الـ ordering بيبقى حسب الـ stamp (timestamp) مش FIFO.

---

#### `__mutex_add_waiter` / `__mutex_remove_waiter`

```c
static void __mutex_add_waiter(struct mutex *lock,
                               struct mutex_waiter *waiter,
                               struct list_head *list)

static void __mutex_remove_waiter(struct mutex *lock,
                                  struct mutex_waiter *waiter)
```

**`__mutex_add_waiter`:**
بتضيف الـ waiter للقائمة وتضبط `MUTEX_FLAG_WAITERS` لو كان أول waiter. ده بيخلي الـ unlocker يعرف إن في ناس منتظرين ويروح الـ slow path.

**`__mutex_remove_waiter`:**
بتشيل الـ waiter من القائمة. لو القائمة اتفرغت، تمسح كل الـ flags (`MUTEX_FLAGS`). كلاهما بينادوا `hung_task_set_blocker` / `hung_task_clear_blocker` للـ hung task detection.

- **Caller context**: لازم تتنادوا وهو ماسك `wait_lock`.

---

#### `__mutex_handoff`

```c
static void __mutex_handoff(struct mutex *lock, struct task_struct *task)
```

**الغرض:** تسليم ملكية الـ lock مباشرة لـ task معين بدون تحريره للعموم. بتضبط `owner = task | MUTEX_FLAG_PICKUP | (WAITERS if any)` مع release semantics. الـ task المستقبل بيكتسب عن طريق `__mutex_trylock_common` اللي بيشوف الـ PICKUP flag.

- **task = NULL**: يعمل unlock عادي (بيمسح الـ owner).
- **بتضمن**: لا يوجد race بين handoff والـ waiter اللي بيـ spin.
- **Caller**: `__mutex_unlock_slowpath` لو الـ HANDOFF flag مضبوط.

---

### Category 5: Public Lock API

---

#### `mutex_lock`

```c
void __sched mutex_lock(struct mutex *lock);
```

**الغرض:** الـ API الأساسي للـ locking. يقفل الـ mutex ويبلوك الـ task لحد ما يقدر يحصل عليه. **ما بيرجعش حتى تكتسب الـ lock** — TASK_UNINTERRUPTIBLE.

**Flow:**
```c
might_sleep();  // debug: warn if in atomic context
if (!__mutex_trylock_fast(lock))
    __mutex_lock_slowpath(lock);  // → __mutex_lock_common(TASK_UNINTERRUPTIBLE)
```

- **lock**: الـ mutex المراد قفله.
- **Return**: void.
- **Caller context**: Process context فقط، لا interrupt context.
- **Key Rules**:
  - نفس الـ task اللي عمل lock لازم يعمل unlock.
  - لا recursive locking.
  - الـ task ما يخرجش بدون ما يعمل unlock.
  - الـ memory اللي فيها الـ mutex ما تتحررش وهو مقفول.

---

#### `mutex_lock_interruptible`

```c
int __sched mutex_lock_interruptible(struct mutex *lock);
```

**الغرض:** زي `mutex_lock` بس بيرجع `-EINTR` لو وصل signal أثناء الانتظار.

```c
might_sleep();
if (__mutex_trylock_fast(lock)) return 0;
return __mutex_lock_interruptible_slowpath(lock);
// → __mutex_lock_common(TASK_INTERRUPTIBLE)
```

- **Return**: `0` نجح، `-EINTR` signal جه.
- **مهم**: `__must_check` — لازم تتحقق من القيمة المرجعة دائماً.
- **الفرق عن `mutex_lock`**: الـ task حالته `TASK_INTERRUPTIBLE` → signal delivery بتصحّيه.

---

#### `mutex_lock_killable`

```c
int __sched mutex_lock_killable(struct mutex *lock);
```

**الغرض:** زي `mutex_lock_interruptible` بس بيستجيب **فقط للـ fatal signals** (مش كل signal). ده أفضل لو عايز تسمح بـ SIGKILL/SIGTERM بس مش SIGCONT أو signals تانية.

```c
might_sleep();
if (__mutex_trylock_fast(lock)) return 0;
return __mutex_lock_killable_slowpath(lock);
// → __mutex_lock_common(TASK_KILLABLE)
```

- **Return**: `0` نجح، `-EINTR` عند fatal signal.

---

#### `mutex_lock_io`

```c
void __sched mutex_lock_io(struct mutex *lock);
```

**الغرض:** زي `mutex_lock` بالظبط، بس بيستخدم `io_schedule_prepare/finish()` عشان يخلي الـ scheduler يحسب الـ task ضمن IO-bound tasks أثناء الانتظار. مهم للـ accounting الصحيح في `iotop` و `/proc/diskstats`.

```c
token = io_schedule_prepare();
mutex_lock(lock);
io_schedule_finish(token);
```

- **متى تستخدمه**: لو الـ mutex بيحمي I/O resource (disk، block device).

---

#### `mutex_trylock`

```c
int __sched mutex_trylock(struct mutex *lock);
```

**الغرض:** محاولة اكتساب الـ lock **بدون blocking**. لو الـ mutex مقفول يرجع فوراً بـ 0.

**مهم جداً — Convention الإرجاع:**
- يرجع `1` = نجح (**عكس `down_trylock`**).
- يرجع `0` = فشل (contention).
- **نفس convention الـ `spin_trylock`**.

- **Caller context**: Process context فقط (مش interrupt).
- **Key Detail**: في DEBUG builds، بيستخدم `_mutex_trylock_nest_lock` مع lockdep annotation.

---

#### `mutex_lock_nested`

```c
void __sched mutex_lock_nested(struct mutex *lock, unsigned int subclass);
```

**الغرض:** `mutex_lock` مع **lockdep subclass** annotation. بتستخدمه لما عندك نفس النوع من الـ mutex في levels مختلفة (مثلاً: parent lock قبل child lock) وعايز lockdep يفهم إن ده مقصود مش deadlock.

- **subclass**: رقم من `0` لـ `MAX_LOCKDEP_SUBCLASSES-1`.
- **بدون CONFIG_DEBUG_LOCK_ALLOC**: الـ macro بيـ map لـ `mutex_lock` مباشرة.

---

#### `mutex_lock_nest_lock`

```c
#define mutex_lock_nest_lock(lock, nest_lock)
```

**الغرض:** بيعمل lock ويقول لـ lockdep إن `nest_lock` هو الـ "outer lock". بيستخدمه الـ subsystems اللي عارفة إن لازم يمسكوا lock A قبل lock B — بيمنع lockdep false alarms.

- **nest_lock**: أي object عنده `struct lockdep_map dep_map`.

---

#### `mutex_lock_io_nested`

```c
void __sched mutex_lock_io_nested(struct mutex *lock, unsigned int subclass);
```

**الغرض:** دمج `mutex_lock_io` + `mutex_lock_nested` — IO accounting مع lockdep subclass.

```c
token = io_schedule_prepare();
__mutex_lock_common(lock, TASK_UNINTERRUPTIBLE, subclass, ...);
io_schedule_finish(token);
```

---

### Category 6: Slow Path — Lock Release

---

#### `mutex_unlock`

```c
void __sched mutex_unlock(struct mutex *lock);
```

**الغرض:** تحرير الـ mutex. لازم يتنادى من نفس الـ task اللي عمل lock.

**Flow:**
```c
#ifndef CONFIG_DEBUG_LOCK_ALLOC
if (__mutex_unlock_fast(lock)) return;  // fast path: no waiters, no flags
#endif
__mutex_unlock_slowpath(lock, _RET_IP_);
```

- **ممنوع في interrupt context**.
- **ممنوع تحرير mutex مش مقفول**.
- **مهم**: الـ caller لازم يضمن إن الـ mutex موجود في memory طول مدة الدالة — `mutex_unlock` ما تُستخدمش لتحرير object ممكن تانية thread تحذفه في نفس الوقت.

---

#### `__mutex_unlock_slowpath`

```c
static noinline void __sched
__mutex_unlock_slowpath(struct mutex *lock, unsigned long ip)
```

**الغرض:** الـ slow path للتحرير — بيتعامل مع الـ waiters والـ handoff.

**Pseudocode Flow:**
```
mutex_release(lockdep)

// Try to clear owner atomically (fast CAS loop)
owner = atomic_long_read(&lock->owner)
loop:
    if owner & HANDOFF: break  // must do handoff
    CAS(owner → flags_only)    // clear task pointer, keep flags
    if CAS succeeds:
        if WAITERS: break      // need to wake someone
        return                 // no waiters, done fast

// Slow: take wait_lock, find next waiter
raw_spin_lock_irqsave(&wait_lock)
if !list_empty(wait_list):
    next = first waiter's task
    wake_q_add(next)           // will be woken after spinlock release

if HANDOFF: __mutex_handoff(lock, next)  // direct ownership transfer

raw_spin_unlock_irqrestore_wake()        // release lock + do wake
```

**Key Details:**
- الـ CAS loop الأولى بتحاول تمسح الـ owner بدون ما تمسك الـ wait_lock — optimistic.
- لو الـ `MUTEX_FLAG_HANDOFF` مضبوط (أول waiter طلب handoff)، بيعمل direct ownership transfer بدل ما يحرر للعموم.
- `raw_spin_unlock_irqrestore_wake()` بتعمل wake_q flush بعد ما تطلق الـ spinlock عشان تقلل lock contention.

---

### Category 7: State Query & Inspection

---

#### `mutex_is_locked`

```c
bool mutex_is_locked(struct mutex *lock);
```

**الغرض:** يشوف هل الـ mutex مقفول حالياً.

```c
return __mutex_owner(lock) != NULL;
// __mutex_owner: atomic_long_read(&lock->owner) & ~MUTEX_FLAGS
```

- **Return**: `true` لو مقفول، `false` لو فاضي.
- **مهم**: النتيجة **غير موثوقة** في وجود concurrency إلا لو واضح من الـ design إن الـ mutex محتجز حالياً.
- **Used by**: `lockdep_assert_held()` داخلياً في كتير من الـ subsystems.

---

#### `mutex_get_owner`

```c
unsigned long mutex_get_owner(struct mutex *lock);
```

**الغرض:** بيرجع عنوان الـ `task_struct` للمالك الحالي. مستخدم في الـ debug infrastructure.

```c
unsigned long owner = atomic_long_read(&lock->owner);
return (unsigned long)__owner_task(owner);
// __owner_task: owner & ~MUTEX_FLAGS
```

- **Return**: عنوان الـ task كـ `unsigned long` — لا تستخدمه كـ pointer مباشرة بدون تحقق (speculative).
- **الهدف**: diagnostics وtracing فقط.

---

### Category 8: Atomic Helper

---

#### `atomic_dec_and_mutex_lock`

```c
int atomic_dec_and_mutex_lock(atomic_t *cnt, struct mutex *lock);
```

**الغرض:** يعمل atomic decrement للـ counter، ولو وصل للصفر يكتسب الـ mutex ويرجع `1`. ده الـ pattern الكلاسيكي لتنفيذ "آخر مستخدم يعمل cleanup".

**Implementation:**
```c
// Fast path: if cnt > 1, just dec and return 0
if (atomic_add_unless(cnt, -1, 1))
    return 0;

// Slow path: cnt might hit 0
mutex_lock(lock);
if (!atomic_dec_and_test(cnt)) {
    // someone else decremented, we didn't hit 0
    mutex_unlock(lock);
    return 0;
}
// we hit 0, holding lock
return 1;
```

**المنطق:** `atomic_add_unless(cnt, -1, 1)` بيـ decrement لو `cnt != 1`. لو `cnt == 1`، بنمسك الـ mutex الأول ثم بنعمل الـ dec تحت الحماية عشان نضمن إن مفيش race بين وصول الصفر وأخد الـ lock.

- **Return**: `1` + lock held لو وصلنا صفر، `0` + lock not held غير كده.
- **Caller**: `kobject_put`، `kref_put` patterns، وأي كود بيحتاج last-reference cleanup.

---

### Category 9: Lock Guard API (C Cleanup)

الكرنل بيوفر `guard()` / `scoped_guard()` macros بناءً على `DEFINE_LOCK_GUARD_1` للتهيئة والتنظيف التلقائي.

```c
DEFINE_LOCK_GUARD_1(mutex, struct mutex,
    mutex_lock(_T->lock),   // acquire
    mutex_unlock(_T->lock)) // release

DEFINE_LOCK_GUARD_1_COND(mutex, _try,
    mutex_trylock(_T->lock))

DEFINE_LOCK_GUARD_1_COND(mutex, _intr,
    mutex_lock_interruptible(_T->lock), _RET == 0)

DEFINE_LOCK_GUARD_1(mutex_init, struct mutex,
    mutex_init(_T->lock), /* no cleanup */)
```

**الاستخدام العملي:**

```c
/* scoped_guard: lock automatically released at end of block */
scoped_guard(mutex, &dev->lock) {
    /* critical section */
    dev->state = NEW_STATE;
}
/* mutex_unlock called automatically here */

/* guard with trylock */
guard(mutex_try)(&dev->lock);
if (!guard_is_acquired())
    return -EBUSY;

/* interruptible */
scoped_guard(mutex_intr, &dev->lock) {
    if (!guard_is_acquired())
        return -EINTR;
    /* ... */
}
```

| Guard Class | `_T->lock` behavior | ملاحظة |
|---|---|---|
| `mutex` | lock/unlock دائماً | standard blocking |
| `mutex_try` | trylock، unlock لو نجح | non-blocking |
| `mutex_intr` | interruptible lock | يتحقق من `_RET == 0` |
| `mutex_init` | init فقط، no unlock | للـ deferred init pattern |

---

### Internal Flag System

الـ `owner` field مش بس pointer للـ task — الـ 3 bits الأقل منه بتحمل flags:

| Flag | Value | المعنى |
|---|---|---|
| `MUTEX_FLAG_WAITERS` | bit 0 | في waiters في القائمة |
| `MUTEX_FLAG_HANDOFF` | bit 1 | أول waiter طلب direct handoff |
| `MUTEX_FLAG_PICKUP` | bit 2 | الـ lock جاهز يتؤخذ من task محدد |

**تفاعل الـ flags مع الـ unlock:**

```
UNLOCK path:
  no HANDOFF, no WAITERS → fast path, done
  WAITERS but no HANDOFF → wake first waiter, they'll trylock
  HANDOFF set            → __mutex_handoff(next) → set PICKUP for next task
                           next task sees PICKUP → takes lock directly
```

ده الـ **direct handoff mechanism** اللي بيقلل latency للـ first waiter بشكل ملحوظ في contended scenarios.

---

### الفرق بين `CONFIG_PREEMPT_RT` و non-RT

| Aspect | `!PREEMPT_RT` | `PREEMPT_RT` |
|---|---|---|
| Implementation | native mutex (هذا الملف) | `rtmutex` (priority inheritance) |
| `mutex_is_locked` | `__mutex_owner != NULL` | `rt_mutex_base_is_locked` |
| `__mutex_init` | `mutex_init_generic/lockep` | `mutex_rt_init_generic/lockdep` |
| Priority inheritance | لا | نعم |
| Suitable for RT tasks | لا | نعم |

في `CONFIG_PREEMPT_RT`، الـ `struct mutex` بيحتوي على `struct rt_mutex_base rtmutex` بدل الـ `atomic_long_t owner` الـ normal، وكل الـ locking يمر عبر الـ rtmutex infrastructure.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

الـ **lockdep** بيكتب معلوماته في debugfs — دي أهم نقطة لـ debugging الـ mutex.

| Entry | الوصف | أمر القراءة |
|---|---|---|
| `/sys/kernel/debug/locking/lockdep` | كل الـ lock classes المسجلة | `cat /sys/kernel/debug/locking/lockdep` |
| `/sys/kernel/debug/locking/lockdep_chains` | سلاسل الـ lock acquisition | `cat /sys/kernel/debug/locking/lockdep_chains` |
| `/sys/kernel/debug/locking/lockdep_stats` | إحصائيات الـ lockdep | `cat /sys/kernel/debug/locking/lockdep_stats` |
| `/sys/kernel/debug/locking/lock_stat` | إحصائيات contention لكل lock | `cat /sys/kernel/debug/locking/lock_stat` |

```bash
# قراءة إحصائيات الـ contention لكل mutex class
cat /sys/kernel/debug/locking/lock_stat | grep -A5 "mutex"

# عدد الـ lock classes المسجلة حالياً
cat /sys/kernel/debug/locking/lockdep_stats | grep "lock-classes"
```

**تفسير الـ lock_stat output:**

```
                              class name    con-bounces    contentions   waittime-min   waittime-max waittime-total    acq-bounces   acquisitions   holdtime-min   holdtime-max holdtime-total
----------------------------------------------------------------------------------------------------
         &inode->i_mutex-W:            0              4           0.27        4211.09       4936.11           1412         374832           0.18         5882.34     1234567.89
```

- **con-bounces**: مرات الـ bounce بين CPUs أثناء الانتظار
- **contentions**: إجمالي مرات الـ contention
- **waittime-max**: أطول وقت انتظار (بالـ µs) — أي قيمة كبيرة هنا تستحق التحقيق
- **holdtime-max**: أطول وقت الـ lock بيفضل محجوز

---

#### 2. sysfs entries

| Entry | الوصف |
|---|---|
| `/proc/lockdep` | نسخة مبسطة من lockdep info |
| `/proc/lockdep_chains` | سلاسل الـ dependency |
| `/proc/locks` | الـ locks المفعّلة حالياً (بالـ inode) |

```bash
# عرض كل الـ locks الحالية
cat /proc/locks

# فلترة mutex-related entries
cat /proc/lockdep | grep -i mutex
```

---

#### 3. ftrace — Tracepoints والـ Events

الـ **mutex** عنده tracepoints مباشرة في الـ kernel تحت `CONFIG_LOCK_STAT`.

```bash
# رؤية كل events المتاحة للـ locking
ls /sys/kernel/debug/tracing/events/lock/

# تفعيل كل lock events
echo 1 > /sys/kernel/debug/tracing/events/lock/enable

# تفعيل events محددة للـ mutex contention
echo 1 > /sys/kernel/debug/tracing/events/lock/contention_begin/enable
echo 1 > /sys/kernel/debug/tracing/events/lock/contention_end/enable

# تشغيل الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... تشغيل السيناريو المشكوك فيه ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**تفعيل function tracer على mutex_lock نفسها:**

```bash
# تتبع كل استدعاءات mutex_lock و mutex_unlock
echo function > /sys/kernel/debug/tracing/current_tracer
echo mutex_lock mutex_unlock mutex_trylock > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

**تفعيل function_graph للـ call stack الكامل:**

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo mutex_lock > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

مثال output:

```
# tracer: function_graph
  0)               |  mutex_lock() {
  0)   0.250 us    |    _mutex_lock();
  0)   0.820 us    |  }
```

---

#### 4. printk / Dynamic Debug

**تفعيل dynamic debug للـ locking subsystem:**

```bash
# تفعيل كل debug messages في ملفات الـ mutex
echo 'file kernel/locking/mutex.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/locking/mutex-debug.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع call stack
echo 'file kernel/locking/mutex.c +ps' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ output
dmesg -w | grep -i mutex
```

**تفعيل lockdep verbose output:**

```bash
# في kernel cmdline:
# lockdep.verbose=1

# أو runtime:
echo 1 > /proc/sys/kernel/panic_on_warn   # يوقف النظام عند أي lockdep warning
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف | التأثير |
|---|---|---|
| `CONFIG_DEBUG_MUTEXES` | يضيف `magic` field + `mutex_destroy()` فعّال | يكتشف double-free وuse-after-free |
| `CONFIG_DEBUG_LOCK_ALLOC` | يفعّل lockdep integration مع الـ mutex | يضيف `dep_map` للـ struct |
| `CONFIG_LOCKDEP` | نظام تتبع الـ lock dependencies الكامل | يكتشف deadlocks وAABB patterns |
| `CONFIG_LOCK_STAT` | إحصائيات contention لكل lock class | يملأ `/sys/kernel/debug/locking/lock_stat` |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ locking في runtime | أقوى أداة — overhead عالي |
| `CONFIG_LOCKDEP_SMALL` | نسخة أصغر من lockdep | مناسبة لبيئات محدودة الذاكرة |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكتشف mutex_lock من atomic context | مهم جداً |
| `CONFIG_PREEMPT_RT` | يحوّل mutex لـ rtmutex | تغيير جوهري في السلوك |
| `CONFIG_DEBUG_RT_MUTEXES` | debug خاص بالـ rtmutex | مع `CONFIG_PREEMPT_RT` |

```bash
# التحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(DEBUG_MUTEX|LOCKDEP|PROVE_LOCKING|LOCK_STAT|DEBUG_ATOMIC)'
```

---

#### 6. devlink وأدوات الـ Subsystem

**الـ mutex مش hardware device، فمفيش devlink مباشر — لكن:**

```bash
# perf لقياس lock contention على مستوى النظام
perf lock record -a -- sleep 5
perf lock report

# perf lock report output example:
#                                    Name   acquired  contended  avg wait (ns)  total wait (ns)
# ---------------------------------------------------------------------------------------------------
#              &mm->mmap_lock-W               45231        234          12340         2887560
```

```bash
# أداة bpftrace لتتبع mutex_lock contentions
bpftrace -e '
kprobe:mutex_lock {
    @start[tid] = nsecs;
}
kretprobe:mutex_lock /@start[tid]/ {
    $delta = nsecs - @start[tid];
    if ($delta > 1000000) {  // أكثر من 1ms
        printf("PID %d held mutex for %d us\n", pid, $delta/1000);
    }
    delete(@start[tid]);
}
'
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `BUG: sleeping function called from invalid context` | `mutex_lock()` من داخل interrupt أو spinlock context | استبدل بـ spinlock أو trylock |
| `WARNING: possible circular locking dependency detected` | lockdep اكتشف ABBA deadlock محتمل | راجع ترتيب الـ locking — حدد lock ordering ثابت |
| `INFO: task XXX blocked for more than 120 seconds` | task عالقة تنتظر mutex | فحص من يملك الـ lock بـ `mutex_get_owner()` |
| `BUG: workqueue lockup` | workqueue thread عالق بسبب mutex | تحقق أن الـ mutex مش محجوز بـ flush_work context |
| `DEADLOCK (false positive possible)` | lockdep أعلن deadlock (قد يكون false positive) | استخدم `mutex_lock_nested()` مع subclass صحيح |
| `DEBUG_LOCKS_WARN_ON(mutex->magic != mutex)` | استخدام mutex بعد `mutex_destroy()` | تحقق من lifetime الـ object |
| `mutex_unlock of unlocked mutex` | unlock لـ mutex مش محجوز | bug في control flow — راجع كل exit paths |
| `lock held when returning to user space` | task رجعت لـ userspace وهي لسه شايلة mutex | leak في الـ locking |

---

#### 8. Strategic Points لـ dump_stack() وWARN_ON()

```c
/* في mutex_lock — تحقق من context صحيح */
void debug_mutex_lock_common(struct mutex *lock, struct mutex_waiter *waiter)
{
    /* تحقق أن مش في atomic context */
    WARN_ON_ONCE(in_atomic());

    /* تحقق من أن الـ magic field سليم */
    WARN_ON(lock->magic != lock);
}
```

**أفضل نقاط لوضع WARN_ON() يدوياً:**

```c
/* 1. قبل mutex_lock في path غير متوقع */
WARN_ON(!mutex_is_locked(&dev->lock));  /* يجب أن يكون محجوز هنا */

/* 2. بعد mutex_trylock للتحقق من النتيجة */
if (!mutex_trylock(&lock)) {
    WARN(1, "contention detected on %s lock\n", "my_lock");
    dump_stack();
    mutex_lock(&lock);
}

/* 3. للكشف عن double-lock */
WARN_ON(mutex_is_locked(&lock) &&
        (mutex_get_owner(&lock) == (unsigned long)current));

/* 4. في cleanup paths للتحقق من unlock */
WARN_ON(mutex_is_locked(&priv->mutex));  /* يجب أن يكون unlocked عند الـ cleanup */
```

---

### Hardware Level

#### 1. التحقق أن الـ Hardware State يطابق الـ Kernel State

الـ mutex هو software primitive بحت — ما فيش hardware register مباشر — لكن الـ atomic operations اللي بيستخدمها بتعتمد على hardware:

```bash
# التحقق أن الـ CPU يدعم الـ atomic compare-and-swap
grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E 'cx8|cx16|lock'

# على ARM — التحقق من دعم LDREX/STREX أو LSE atomics
grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E 'atomics|lse'
```

**الـ `owner` field في struct mutex هو `atomic_long_t`** — بيستخدم `cmpxchg` أو `ldxr/stxr` على ARM. أي مشكلة في الـ cache coherency بتظهر كـ contention عالي بشكل غير مبرر.

---

#### 2. Register Dump Techniques

الـ mutex نفسه memory structure — مفيش hardware registers خاصة بيه. لكن ممكن نفحص الـ struct مباشرة من الذاكرة:

```bash
# قراءة قيمة owner الـ mutex من الذاكرة (يحتاج عنوان الـ struct)
# أولاً: اعرف العنوان من /proc/kallsyms أو crash dump
cat /proc/kallsyms | grep "my_mutex_name"

# استخدام crash tool لفحص struct mutex
crash vmlinux /proc/kcore
crash> struct mutex 0xffffffff82345678
crash> p my_global_mutex
```

```bash
# devmem2 لقراءة الـ owner field مباشرة (إذا عرفت العنوان الفيزيائي)
# owner هو أول field في struct mutex بعد الـ wait_lock
devmem2 0x<physical_address_of_mutex> w
```

**فحص struct mutex في الـ crash dump:**

```
crash> struct mutex
struct mutex {
  atomic_long_t owner;        /* 0  bytes - الـ task pointer + flags */
  raw_spinlock_t wait_lock;   /* 8  bytes */
  struct list_head wait_list; /* 16 bytes */
  ...
}

# قيمة owner:
# 0x0     = unlocked
# task*   = locked (lower 3 bits = flags: HANDOFF, WAITERS, PICKUP)
# bit 0   = MUTEX_FLAG_WAITERS (في الـ wait_list)
# bit 1   = MUTEX_FLAG_HANDOFF (handoff to waiter in progress)
# bit 2   = MUTEX_FLAG_PICKUP  (waiter has been given lock)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ mutex debugging بـ logic analyzer بيكون غير مباشر — عبر GPIO أو hardware events:

```c
/* أضف GPIO toggling حول الـ critical section للقياس */
#include <linux/gpio.h>

gpio_set_value(DEBUG_GPIO, 1);  /* set high قبل mutex_lock */
mutex_lock(&lock);
gpio_set_value(DEBUG_GPIO, 0);  /* set low بعد الـ lock */
/* ... critical section ... */
gpio_set_value(DEBUG_GPIO, 1);  /* set high قبل unlock */
mutex_unlock(&lock);
gpio_set_value(DEBUG_GPIO, 0);
```

- **Logic Analyzer**: قس الفترة اللي الـ GPIO فيها high = وقت الانتظار على الـ lock
- **Oscilloscope**: لو الـ pulse طويل جداً (>1ms في real-time system) → contention problem

**PMU (Performance Monitoring Unit) counters:**

```bash
# قياس cache misses أثناء mutex operations (دليل على contention بين CPUs)
perf stat -e cache-misses,cache-references,bus-cycles \
    -p <pid_of_process> sleep 5
```

---

#### 4. Common Hardware Issues وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في الـ Kernel Log | السبب |
|---|---|---|
| Cache coherency failure (SMP) | Extreme mutex contention بدون سبب منطقي | الـ LLC مش بيـ invalidate بسرعة — راجع BIOS cache settings |
| CPU frequency scaling أثناء critical section | Variable wait times في `lock_stat` | استخدم `cpupower frequency-set -g performance` |
| NUMA topology mismatch | High `con-bounces` في lock_stat | الـ mutex محجوز من CPU على NUMA node مختلف |
| Memory ordering issue (ARM) | Intermittent deadlocks | تأكد أن الـ memory barriers في الـ atomic ops صح |
| Virtualization overhead | High wait times فجأة | Hypervisor preempted الـ lock holder |

```bash
# تحقق من NUMA topology وأثره
numactl --hardware
cat /proc/sys/kernel/numa_balancing

# فحص CPU migration أثناء lock hold
perf stat -e migrations -p <pid> sleep 5
```

---

#### 5. Device Tree Debugging

الـ mutex مش مرتبط بـ DT مباشرة — لكن لو الـ mutex بيحمي hardware resource معرّفة في DT:

```bash
# تحقق أن الـ device اللي الـ mutex بيحميه موجود في DT
ls /sys/bus/platform/devices/
cat /sys/bus/platform/devices/<device>/of_node/compatible

# تحقق من الـ regulator أو clock اللي محتاج mutex
cat /sys/kernel/debug/clk/clk_summary | grep <clock_name>
cat /sys/kernel/debug/regulator/regulator_summary
```

```bash
# لو الـ mutex بيحمي I2C/SPI access — تحقق من الـ DT bindings
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "your-device"
```

---

### Practical Commands

#### جاهز للنسخ — كل التقنيات

**1. تفعيل lockdep والـ contention stats من البداية:**

```bash
# في /etc/default/grub أضف للـ GRUB_CMDLINE_LINUX:
# lockdep.verbose=1 prove_locking=1

# أو تحقق إنهم enabled في الـ kernel المحلي
zcat /proc/config.gz | grep -E 'PROVE_LOCKING|LOCK_STAT|DEBUG_MUTEX'
```

**2. فحص الـ deadlock فوراً:**

```bash
# أسرع طريقة — lockdep بيطبع في dmesg
dmesg | grep -E "possible circular|deadlock|WARNING.*locking|held lock"
```

**3. مشاهدة lock contention في الوقت الفعلي:**

```bash
# إعادة تعيين الإحصائيات
echo 0 > /sys/kernel/debug/locking/lock_stat

# تشغيل الـ workload المشكوك فيه لـ 10 ثواني
sleep 10

# قراءة النتائج مرتبة حسب الأعلى contention
sort -k4 -rn /sys/kernel/debug/locking/lock_stat | head -20
```

**مثال output وتفسيره:**

```
             &dev->mutex:          5        1823         0.15      45230.20      5234567     12345        987654      0.01      12345.67    234567890
```
- الـ class اسمها `&dev->mutex`
- حدث contention **1823 مرة**
- أطول انتظار **45ms** → مشكلة واضحة، الـ critical section طويلة جداً

**4. تتبع من الذي يملك الـ mutex:**

```bash
# باستخدام crash على live kernel
echo "p my_global_mutex" | crash /proc/kcore

# باستخدام gdb على vmlinux
gdb vmlinux /proc/kcore -ex "p my_global_mutex.owner" -ex quit
```

**5. perf lock لتحليل شامل:**

```bash
# تسجيل lock events لـ 5 ثواني
perf lock record -a -g -- sleep 5

# تقرير مفصل
perf lock report --threads --lock-name

# مثال output:
# Name                  acquired  contended  avg wait (ns)
# &inode->i_rwsem              0       1234          56789
# &mm->mmap_lock-W          5678        123           4567
```

**6. bpftrace لـ real-time mutex contention:**

```bash
# طباعة أي mutex_lock أخد أكثر من 100µs
bpftrace -e '
kprobe:mutex_lock { @ts[tid] = nsecs; @nm[tid] = arg0; }
kretprobe:mutex_lock /@ts[tid]/ {
  $d = (nsecs - @ts[tid]) / 1000;
  if ($d > 100) {
    printf("%-16s PID:%-6d WAIT:%d us  MUTEX:0x%lx\n",
           comm, pid, $d, @nm[tid]);
  }
  delete(@ts[tid]); delete(@nm[tid]);
}'
```

**7. فحص task عالقة على mutex:**

```bash
# إيجاد كل tasks في حالة D (uninterruptible sleep = تنتظر mutex)
ps aux | awk '$8 == "D" {print $2, $11}'

# فحص stack trace للـ task العالقة
cat /proc/<PID>/wchan          # اسم الـ kernel function اللي بتنتظر فيها
cat /proc/<PID>/stack          # الـ call stack الكامل

# مثال output:
# [<0>] __mutex_lock+0x4a2/0xa60
# [<0>] i2c_transfer+0x67/0x130
# [<0>] my_driver_read+0x23/0x80
```

**8. تفعيل كل mutex debug دفعة واحدة (debug kernel):**

```bash
#!/bin/bash
# mutex-debug-enable.sh

# تفعيل lock events في ftrace
echo 1 > /sys/kernel/debug/tracing/events/lock/enable

# تفعيل dynamic debug
echo 'file kernel/locking/mutex.c +pflmt' > \
    /sys/kernel/debug/dynamic_debug/control

# إعادة تعيين lock stats
echo 0 > /sys/kernel/debug/locking/lock_stat

# تفعيل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

echo "Mutex debugging enabled. Run your workload, then:"
echo "  cat /sys/kernel/debug/tracing/trace"
echo "  cat /sys/kernel/debug/locking/lock_stat"
```

**9. تحليل lockdep graph:**

```bash
# طباعة كل الـ lock dependencies المعروفة
cat /proc/lockdep_chains | head -100

# عدد الـ lock classes — لو اقترب من الـ MAX_LOCKDEP_KEYS (8192) → مشكلة
cat /sys/kernel/debug/locking/lockdep_stats | grep "lock-classes"
# lock-classes:                        512  [max: 8191]
```

**10. سكريبت شامل لتشخيص deadlock:**

```bash
#!/bin/bash
# deadlock-diagnose.sh

echo "=== Lockdep Warnings ==="
dmesg | grep -A20 "possible circular locking"

echo "=== Tasks in D state ==="
for pid in $(ps -eo pid,stat | awk '$2 ~ /^D/ {print $1}'); do
    echo "--- PID $pid: $(cat /proc/$pid/comm 2>/dev/null) ---"
    cat /proc/$pid/stack 2>/dev/null
done

echo "=== Top Contended Locks ==="
sort -t: -k2 -rn /sys/kernel/debug/locking/lock_stat 2>/dev/null | head -10

echo "=== Held Locks Summary ==="
cat /proc/lockdep 2>/dev/null | grep "held by" | sort | uniq -c | sort -rn | head -20
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Deadlock في Industrial Gateway على RK3562

#### العنوان
**Deadlock** بين driver الـ SPI و driver الـ I2C في gateway صناعي

#### السياق
بورد RK3562 بيشتغل كـ industrial gateway بيجمع بيانات من sensors عبر I2C و يبعتها عبر SPI إلى controller تاني. الـ firmware اشتغل تمام في الـ testing لكن بعد 6 ساعات في production بيتجمد الجهاز تماماً.

#### المشكلة
الـ system بيتعلق كامل، الـ watchdog مش بيتفعل، والـ `/proc/sysrq-trigger` مش بيرد. الـ dmesg آخر رسالة فيه:

```
[ 21643.112] spi-rockchip: waiting for mutex...
[ 21643.112] i2c-rk3x: waiting for mutex...
```

#### التحليل
التتبع في الكود:

```c
/* driver A - SPI driver */
mutex_lock(&spi_bus_lock);          // SPI driver lock SPI mutex أولاً
    mutex_lock(&shared_config_lock); // بعدين بيحاول يلاقي shared mutex

/* driver B - I2C driver - في thread تاني */
mutex_lock(&shared_config_lock);    // I2C driver lock shared mutex أولاً
    mutex_lock(&spi_bus_lock);       // بعدين بيحاول يلاقي SPI mutex ← DEADLOCK!
```

الـ `mutex_lock()` في `include/linux/mutex.h` هو **blocking call** — بيعمل sleep للـ task لحد ما يحصل على الـ lock. ما فيش timeout. لما thread A يمسك `spi_bus_lock` ويستنى `shared_config_lock`، وفي نفس الوقت thread B ماسك `shared_config_lock` ومستنى `spi_bus_lock` — circular dependency كامل.

الـ `mutex_init()` macro من الملف:

```c
#define mutex_init(mutex)       \
do {                            \
    static struct lock_class_key __key; \
    __mutex_init((mutex), #mutex, &__key); \
} while (0)
```

كل mutex عنده `lock_class_key` مختلف. الـ lockdep كان ممكن يكتشف الموضوع لو كان مفعّل.

#### الحل

**أولاً:** تفعيل lockdep في kernel config للـ development build:

```bash
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_LOCKDEP=y
```

**ثانياً:** فرض ترتيب ثابت لأخذ الـ mutexes — دايماً `shared_config_lock` قبل أي bus lock:

```c
/* الترتيب الصح — في الـ SPI driver */
mutex_lock(&shared_config_lock);   // shared أولاً دايماً
mutex_lock(&spi_bus_lock);
    /* critical section */
mutex_unlock(&spi_bus_lock);
mutex_unlock(&shared_config_lock);

/* الترتيب الصح — في الـ I2C driver */
mutex_lock(&shared_config_lock);   // نفس الترتيب
mutex_lock(&i2c_bus_lock);
    /* critical section */
mutex_unlock(&i2c_bus_lock);
mutex_unlock(&shared_config_lock);
```

**ثالثاً:** لو مش ممكن تغيير الترتيب، استخدم `mutex_trylock()` مع fallback:

```c
/* mutex_trylock returns 1 on success, 0 on contention */
if (!mutex_trylock(&spi_bus_lock)) {
    /* release what we have and retry later */
    mutex_unlock(&shared_config_lock);
    schedule_timeout(msecs_to_jiffies(1));
    goto retry;
}
```

#### الدرس المستفاد
الـ `mutex_lock()` ما عندوش timeout بطبيعته — بيستنى للأبد. في systems بتشتغل لفترات طويلة، **lock ordering** لازم يتوثق وينفّذ بشكل صارم. الـ lockdep tool في kernel لو اشتغل في development بيكتشف الـ deadlock من أول مرة يحصل فيها هذا الـ ordering.

---

### السيناريو 2: Mutex في Interrupt Context على STM32MP1

#### العنوان
**BUG: scheduling while atomic** في driver الـ UART على STM32MP1 IoT sensor

#### السياق
بورد STM32MP1 بيشتغل كـ IoT sensor node — بيقرأ قراءات من ADC وبيبعتها عبر UART كل 100ms. الـ driver اشتغل تمام في bench testing لكن تحت load عالي الـ kernel بيطلع panic.

#### المشكلة
```
BUG: scheduling while atomic: sensor_daemon/1234/0x00000002
Call Trace:
  __schedule+0x...
  schedule+0x...
  mutex_lock+0x...         ← هنا المشكلة
  uart_send_irq_handler+0x...
```

#### التحليل
المبرمج كتب الكود ده في الـ IRQ handler:

```c
/* WRONG: called from interrupt context */
static irqreturn_t uart_tx_irq(int irq, void *dev_id)
{
    struct uart_data *data = dev_id;

    mutex_lock(&data->tx_lock);   /* BUG: mutex_lock can sleep! */
    /* fill TX FIFO */
    mutex_unlock(&data->tx_lock);

    return IRQ_HANDLED;
}
```

الـ mutex في `include/linux/mutex.h` معرّف بـ `LD_WAIT_SLEEP`:

```c
#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define __DEP_MAP_MUTEX_INITIALIZER(lockname)      \
        , .dep_map = {                              \
            .name = #lockname,                      \
            .wait_type_inner = LD_WAIT_SLEEP,   /* ← هنا */ \
        }
```

الـ `LD_WAIT_SLEEP` معناه إن الـ mutex يقدر يعمل sleep للـ caller لو الـ lock مشغول. الـ interrupt context مش بيقدر يعمل sleep — مفيش task context صالح للـ scheduler.

لما `mutex_lock()` لاقى الـ lock مشغول، حاول يحط الـ current thread في wait queue ويستدعي `schedule()`. الـ kernel اكتشف إن في `atomic context` (interrupt) ورفع الـ BUG.

#### الحل

استبدال الـ mutex بـ `spinlock` في interrupt context:

```c
/* In driver init */
spin_lock_init(&data->tx_lock);  /* spinlock بدل mutex */

/* IRQ handler — الكود الصح */
static irqreturn_t uart_tx_irq(int irq, void *dev_id)
{
    struct uart_data *data = dev_id;
    unsigned long flags;

    spin_lock_irqsave(&data->tx_lock, flags);  /* safe in IRQ */
    /* fill TX FIFO */
    spin_unlock_irqrestore(&data->tx_lock, flags);

    return IRQ_HANDLED;
}

/* Process context code — لو محتاج mutex في مكان تاني */
static ssize_t uart_write(struct file *f, const char __user *buf, ...)
{
    mutex_lock(&data->config_lock);  /* OK: process context */
    /* change config */
    mutex_unlock(&data->config_lock);
}
```

لو لازم shared lock بين process context وinterrupt context، استخدم `spin_lock_irqsave` في الاتنين:

```c
/* process context */
spin_lock_irqsave(&data->tx_lock, flags);
/* ... */
spin_unlock_irqrestore(&data->tx_lock, flags);
```

#### الدرس المستفاد
الـ `mutex` في Linux **ممنوع تماماً في interrupt context** — مش مجرد bad practice، ده BUG حرفي. القاعدة البسيطة: لو في `_irq` أو `_bh` في اسم الـ function، استخدم spinlock مش mutex. الـ `LD_WAIT_SLEEP` في الـ `__DEP_MAP_MUTEX_INITIALIZER` هو العلامة الرسمية لده.

---

### السيناريو 3: Priority Inversion في Android TV Box على Allwinner H616

#### العنوان
**Priority Inversion** يسبب stuttering في الـ HDMI output على Android TV box

#### السياق
TV box بيشتغل بـ Allwinner H616، Android 12. المستخدمين بيشتكوا من stuttering كل دقيقتين تقريباً في الـ 4K playback عبر HDMI. الـ CPU usage عادي والـ GPU مش bottleneck.

#### المشكلة
الـ video decoder thread (priority عالية) بيتعطل لفترات مش منطقية حتى لو الـ system مش مشحون. الـ `ftrace` بيظهر إن الـ decoder بيستنى mutex ماسكه thread أقل أهمية.

```
# trace output
decoder_thread-89  [003] 15234.1: mutex_lock: &vpu_lock
background_task-45 [001] 15234.0: mutex_lock: &vpu_lock (held)
background_task-45 [001] 15234.0: sched_wakeup: comm=kworker ...
```

#### التحليل
المشكلة classic **priority inversion**:

```
High Priority:   [decoder_thread]  → waiting for &vpu_lock
                         ↑ blocked
Medium Priority: [network_thread]  → preempts background_task (no lock relation)
                         ↑ preempts
Low Priority:    [background_task] → holds &vpu_lock, can't run!
```

الـ standard `mutex` في الـ non-RT kernel:

```c
/* include/linux/mutex.h - non-RT path */
#ifndef CONFIG_PREEMPT_RT
#define __MUTEX_INITIALIZER(lockname) \
        { .owner = ATOMIC_LONG_INIT(0) \
        , .wait_lock = __RAW_SPIN_LOCK_UNLOCKED(lockname.wait_lock) \
        , .wait_list = LIST_HEAD_INIT(lockname.wait_list) \
        ...
```

مش بيعمل **priority inheritance** تلقائياً. الـ background task بيفضل يشتغل بـ priority منخفضة، والـ decoder العالي بيستنى.

أما لو كان الـ kernel compiled بـ `CONFIG_PREEMPT_RT`:

```c
/* include/linux/mutex.h - RT path */
#else /* !CONFIG_PREEMPT_RT */
#define __MUTEX_INITIALIZER(mutexname)          \
{                                               \
    .rtmutex = __RT_MUTEX_BASE_INITIALIZER(mutexname.rtmutex) \
    ...
```

الـ RT mutex **بيعمل priority inheritance** — الـ low priority task اللي ماسكة الـ lock بيتحولها مؤقتاً لـ priority بتاع الـ high priority waiter.

#### الحل

**الحل الأول** — تفعيل `CONFIG_PREEMPT_RT` في الـ kernel:

```bash
# في kernel config
CONFIG_PREEMPT_RT=y
# هيحول كل mutex تلقائياً لـ rtmutex مع priority inheritance
```

**الحل الثاني** — لو مش ممكن RT kernel، استخدم `rt_mutex` صراحةً:

```c
#include <linux/rtmutex.h>

struct vpu_device {
    struct rt_mutex vpu_lock;  /* priority-inheriting mutex */
    /* ... */
};

/* init */
rt_mutex_init(&vpu->vpu_lock);

/* lock */
rt_mutex_lock(&vpu->vpu_lock);
/* ... */
rt_mutex_unlock(&vpu->vpu_lock);
```

**الحل الثالث** — تصميمي: فصل الـ critical sections:

```c
/* بدل ما الـ background task يمسك VPU lock وقت طويل */
/* اعمل copy of data أولاً بدون lock */
local_copy = expensive_compute();  /* no lock needed here */

/* بعدين امسك الـ lock بس وقت التحديث السريع */
mutex_lock(&vpu_lock);
vpu->state = local_copy;           /* fast update */
mutex_unlock(&vpu_lock);
```

#### الدرس المستفاد
الـ standard `mutex` في Linux مش بيعمل priority inheritance في الـ non-RT kernel. في multimedia systems فيها real-time requirements، إما `CONFIG_PREEMPT_RT` أو `rt_mutex` صراحةً، أو تصميم يخلي الـ critical sections قصيرة جداً عشان الـ inversion فترتها قصيرة ومقبولة.

---

### السيناريو 4: Lockdep Warning في Automotive ECU على i.MX8

#### العنوان
**Lockdep false positive** بيعطّل بورد bring-up على i.MX8 automotive ECU

#### السياق
فريق bring-up بيشتغل على ECU جديد بيشتغل على i.MX8QM لـ تطبيق automotive. عملوا driver للـ CAN bus الداخلي. في أول boot بـ debug kernel، الـ kernel بيطلع warning طويل وبيوقف.

#### المشكلة
```
WARNING: possible circular locking dependency detected
can_driver/89 is trying to acquire lock:
  (&can_dev->tx_mutex){+.+.}-{3:3}

but task is already holding lock:
  (&can_dev->rx_mutex){+.+.}-{3:3}

which lock already depends on the new lock.
```

لكن المبرمج واثق إنه مش في deadlock حقيقي — الـ TX path والـ RX path مش بيتقاطعوا فعلاً في runtime.

#### التحليل
الـ lockdep بيشتغل على **lock classes** مش على instances. الـ `mutex_init()` macro:

```c
#define mutex_init(mutex)                   \
do {                                        \
    static struct lock_class_key __key;     /* ← static key per call-site */ \
    __mutex_init((mutex), #mutex, &__key);  \
} while (0)
```

لو المبرمج عمل `mutex_init` للـ `tx_mutex` وللـ `rx_mutex` في نفس الـ function (نفس call site pattern)، ممكن الـ lockdep يعتبرهم **same class** في بعض الحالات، أو العكس — بيبني dependency graph من كل lock acquisitions تاريخياً.

المشكلة: في test scenario معين، تم أخذ `rx_mutex` ثم `tx_mutex`. في scenario تاني، تم أخذ `tx_mutex` أولاً. الـ lockdep اعتبر ده circular dependency محتمل حتى لو في runtime مش ممكن يحصلوا في نفس الوقت.

الحل باستخدام `mutex_init_with_key()` من الملف:

```c
/* include/linux/mutex.h */
#define mutex_init_with_key(mutex, key) __mutex_init((mutex), #mutex, (key))
```

ده بيخليك تحدد الـ lock class صراحةً وتفصل الـ RX class عن الـ TX class.

#### الحل

**أولاً:** تعريف lock classes صريحة:

```c
/* can_driver.c */
static struct lock_class_key can_tx_lock_class;
static struct lock_class_key can_rx_lock_class;

static int can_probe(struct platform_device *pdev)
{
    struct can_device *dev = /* ... */;

    /* assign separate classes explicitly */
    mutex_init_with_key(&dev->tx_mutex, &can_tx_lock_class);
    mutex_init_with_key(&dev->rx_mutex, &can_rx_lock_class);

    return 0;
}
```

**ثانياً:** لو الـ nesting مقصود وآمن، استخدم `mutex_lock_nested()`:

```c
#ifdef CONFIG_DEBUG_LOCK_ALLOC
/* subclass 0 = outer lock, subclass 1 = inner lock */
mutex_lock_nested(&dev->rx_mutex, 0);  /* outer */
mutex_lock_nested(&dev->tx_mutex, 1);  /* inner — lockdep knows the order */
#else
mutex_lock(&dev->rx_mutex);
mutex_lock(&dev->tx_mutex);
#endif
```

**ثالثاً:** تحقق من الـ warning بأدوات kernel:

```bash
# اقرأ الـ lockdep report كامل
dmesg | grep -A 50 "circular locking"

# شوف الـ lock dependency graph
cat /proc/lockdep_chains

# عدد الـ lock classes المسجلة
cat /proc/lockdep_stats
```

#### الدرس المستفاد
الـ lockdep بيشتغل على **lock classes** مش على instances — لو عندك mutexes بتتصرف بشكل مختلف conceptually (TX vs RX)، استخدم `mutex_init_with_key()` لتعيين classes مختلفة. الـ false positive من lockdep علامة إن الـ locking design محتاج مراجعة حتى لو مش deadlock حقيقي.

---

### السيناريو 5: devm_mutex_init في Custom Board Bring-up على AM62x

#### العنوان
**Use-after-free** بسبب manual `mutex_destroy()` مع `devm_mutex_init` على AM62x

#### السياق
فريق bring-up بيشتغل على custom industrial board بيشتغل على AM62x (Texas Instruments). الـ board عنده display controller custom عبر MIPI DSI. الـ driver بيشتغل تمام في الـ normal operation لكن لما بيتعمل `modprobe -r` للـ driver، الـ kernel بيعمل crash.

#### المشكلة
```
BUG: kernel NULL pointer dereference
RIP: mutex_lock+0x23
Call Trace:
  display_cleanup+0x...
  __devm_mutex_destruct+0x...   ← devm trying to destroy
  display_remove+0x...           ← driver remove
```

#### التحليل
المبرمج استخدم `devm_mutex_init()` بس كمان عمل `mutex_destroy()` يدوياً في الـ `remove()`:

```c
/* include/linux/mutex.h */
#define devm_mutex_init(dev, mutex) \
    __devm_mutex_init(dev, __mutex_init_ret(mutex))
```

الـ `devm_mutex_init` بيسجّل `mutex_destroy()` callback في الـ device resource management. لما الـ driver يتعمل remove، الـ `devm` infrastructure بيستدعي `mutex_destroy()` تلقائياً.

لكن المبرمج كمان عمل:

```c
/* WRONG: manual destroy + devm destroy = double destroy */
static int display_remove(struct platform_device *pdev)
{
    struct display_dev *dev = platform_get_drvdata(pdev);

    mutex_destroy(&dev->display_mutex);  /* first destroy */
    /* devm will call mutex_destroy AGAIN later! */
    return 0;
}
```

مع `CONFIG_DEBUG_MUTEXES=y`، الـ `mutex_destroy()` بيزق الـ `magic` pointer في الـ mutex struct:

```c
/* kernel/locking/mutex-debug.c */
void mutex_destroy(struct mutex *lock)
{
    DEBUG_LOCKS_WARN_ON(mutex_is_locked(lock));
    lock->magic = NULL;  /* ← بيعمل nullify */
}
```

لما الـ `devm` يجي يعمل الـ destroy التاني، الـ pointer بقى NULL وحصل crash.

#### الحل

**القاعدة:** استخدم إما `devm_mutex_init` أو `mutex_init` + `mutex_destroy` يدوي — مش الاتنين مع بعض.

**الطريقة الصح مع devm:**

```c
static int display_probe(struct platform_device *pdev)
{
    struct display_dev *dev;
    int ret;

    dev = devm_kzalloc(&pdev->dev, sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;

    /* devm will handle mutex_destroy automatically on remove */
    ret = devm_mutex_init(&pdev->dev, &dev->display_mutex);
    if (ret)
        return ret;

    platform_set_drvdata(pdev, dev);
    return 0;
}

static int display_remove(struct platform_device *pdev)
{
    struct display_dev *dev = platform_get_drvdata(pdev);

    /* DO NOT call mutex_destroy here! devm handles it */
    display_disable_output(dev);
    return 0;
}
```

**الطريقة الصح بدون devm:**

```c
static int display_probe(struct platform_device *pdev)
{
    mutex_init(&dev->display_mutex);  /* manual init */
    return 0;
}

static int display_remove(struct platform_device *pdev)
{
    mutex_destroy(&dev->display_mutex);  /* manual destroy — ONE TIME ONLY */
    return 0;
}
```

**للتحقق:** فعّل `CONFIG_DEBUG_MUTEXES` في kernel:

```bash
CONFIG_DEBUG_MUTEXES=y
```

هيفعّل الـ `magic` field validation اللي بيكتشف الـ double-destroy فوراً بدل ما تحصل null deref غامض.

لاحظ من الملف إن `mutex_destroy` بتتصرف بشكل مختلف حسب الـ config:

```c
/* include/linux/mutex.h */
#ifdef CONFIG_DEBUG_MUTEXES
extern void mutex_destroy(struct mutex *lock);  /* real implementation */
#else
static inline void mutex_destroy(struct mutex *lock) {}  /* nop! */
#endif
```

في production kernels بدون `CONFIG_DEBUG_MUTEXES`، الـ `mutex_destroy` مجرد nop — الـ crash مش هيحصل لكن الـ `devm` هيفضل يحاول يدمر mutex اتدمر بالفعل، وده undefined behavior صامت خطير.

#### الدرس المستفاد
الـ `devm_mutex_init()` هو convenience wrapper بيضيف `mutex_destroy()` لـ device cleanup list. **ممنوع** تجمع بينه وبين `mutex_destroy()` يدوي. اختار نهج واحد وامسك بيه. فعّل `CONFIG_DEBUG_MUTEXES` في development دايماً — هو اللي بيكتشف ده بشكل صريح بدل crash غامض في production.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/locking/mutex-design.rst`](https://docs.kernel.org/locking/mutex-design.html) | التوثيق الرسمي لتصميم الـ mutex — الـ fastpath والـ midpath والـ slowpath |
| [`kernel/locking/mutex.c`](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) | الـ implementation الكاملة في الـ source tree |
| [`include/linux/mutex.h`](https://github.com/torvalds/linux/blob/master/include/linux/mutex.h) | الـ header الرئيسي — نفس الملف اللي بندرسه |
| [`include/linux/mutex_types.h`](https://github.com/torvalds/linux/blob/master/include/linux/mutex_types.h) | تعريف الـ `struct mutex` نفسها |
| [`Documentation/locking/locktypes.rst`](https://docs.kernel.org/locking/locktypes.html) | مقارنة بين كل أنواع الـ locking في الـ kernel |
| [`Documentation/locking/lockdep-design.txt`](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt) | تصميم الـ lockdep — نظام كشف الـ deadlocks |

---

### مقالات LWN.net

دي أهم المقالات على [lwn.net](https://lwn.net) اللي بتغطي الـ mutex بالتفصيل:

#### المقالات الأساسية

- **[The mutex API](https://lwn.net/Articles/167034/)** — Corbet, 2006
  شرح مفصّل لأول إضافة لـ mutex API في الـ kernel، بيوضح الـ struct وكل دالة وإزاي بتختلف عن الـ semaphore.

- **[MUTEX: Introduce simple mutex implementation](https://lwn.net/Articles/163807/)** — 2006
  الـ patch الأصلي اللي Ingo Molnar قدّمه لإضافة الـ generic mutex subsystem. بيشرح الـ design decisions من الأول.

- **[Documentation/mutex-design.txt](https://lwn.net/Articles/602393/)** — تغطية LWN لتوثيق الـ mutex design.

- **[A surprise with mutexes and reference counts](https://lwn.net/Articles/575460/)** — Corbet, 2013
  مقال بيشرح kernel crash حصل بسبب interaction غلط بين الـ mutex والـ reference counting — مثال real-world مهم جداً.

- **[Wait/wound mutexes](https://lwn.net/Articles/548909/)** — 2013
  بيشرح الـ `ww_mutex` — نوع خاص من الـ mutex لحل مشكلة الـ deadlock لما بتحتاج تاخد أكتر من lock في نفس الوقت (زي الـ GPU drivers).

- **[Adaptive mutexes in user space](https://lwn.net/Articles/704843/)** — 2016
  بيناقش إزاي الـ adaptive spinning اللي اتعمل في الـ kernel mutex اتطبق على الـ user-space futexes.

- **[Linux 2.6 Real Time Kernel - Mutex Patch](https://lwn.net/Articles/105867/)** — 2004
  من البداية — الـ real-time mutex patch اللي أثّر على تصميم الـ mutex العادي.

- **[Local locks in the kernel](https://lwn.net/Articles/828477/)** — 2021
  بيشرح الـ `local_lock` والفرق بينه وبين الـ mutex في سياق الـ PREEMPT_RT.

---

### نقاشات الـ Mailing List (LKML)

- **[mutex: Fix optimistic spinning vs. BKL](https://lore.kernel.org/lkml/20100701173208.005781188@clark.site/)** — LKML 2010
  نقاش عن حالة edge case لما الـ optimistic spinning يتعارض مع الـ Big Kernel Lock.

- **[mutex: Disable optimistic spinning on some architectures](https://lists.openwall.net/linux-kernel/2014/07/16/912)** — LKML 2014
  قرار تعطيل الـ spinning على معمارياّت معينة بسبب مشاكل الـ performance.

- **[PATCH: misc patches for mutex and rwsem](https://lkml.kernel.org/lkml/20211013134154.1085649-1-yanfei.xu@windriver.com/T/)** — LKML 2021
  patches حديثة للـ mutex والـ rwsem.

---

### الـ Kernel Commits المهمة

| الوصف | الرابط |
|-------|--------|
| الـ commit الأصلي لـ mutex subsystem (Ingo Molnar, 2006) | [`torvalds/linux commits — mutex`](https://github.com/torvalds/linux/commits/master/kernel/locking/mutex.c) |
| الـ source الكامل لـ `mutex.c` في `master` | [`kernel/locking/mutex.c`](https://github.com/torvalds/linux/blob/master/kernel/locking/mutex.c) |
| تاريخ الـ `include/linux/mutex.h` | [`include/linux/mutex.h commits`](https://github.com/torvalds/linux/commits/master/include/linux/mutex.h) |

لمشاهدة أول commit لـ mutex subsystem ابحث في `git log --follow` عن:
```bash
git log --oneline --follow kernel/locking/mutex.c | tail -20
```

---

### KernelNewbies.org

- **[SMPSynchronisation](https://kernelnewbies.org/SMPSynchronisation)** — شرح مبسّط لكل آليات الـ synchronization في الـ SMP kernel، بما فيها الـ mutex.
- **[KernelGlossary — mutex](https://kernelnewbies.org/KernelGlossary)** — تعريف الـ mutex مقارنةً بالـ spinlock والـ semaphore.
- **[Linux_2_6_16](https://kernelnewbies.org/Linux_2_6_16)** — أول kernel version رسمياً دعم الـ generic mutex subsystem.
- **[BigKernelLock](https://kernelnewbies.org/BigKernelLock)** — بيشرح ليه اتحول الـ BKL لـ mutex وبعدين اتشال خالص.

---

### eLinux.org

- **[Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources)** — صفحة موارد عامة فيها links لتوثيق الـ locking.
- **[R-Car/Debugging kernel options causes performance down](https://elinux.org/R-Car/Debugging-kernel-options-causes-performance-down)** — بيشرح تأثير تفعيل `CONFIG_DEBUG_MUTEXES` و`CONFIG_LOCKDEP` على الـ performance في الأنظمة المدمجة.

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 5: Concurrency and Race Conditions**
  بيشرح الـ mutex في سياق الـ driver development، ومتى تستخدمه بدل الـ spinlock.
  - متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 10: Kernel Synchronization Methods**
  بيغطي `mutex_lock()`, `mutex_unlock()`, `mutex_trylock()` مع أمثلة.
- **الفصل 9: An Introduction to Kernel Synchronization**
  بيشرح الـ critical sections والـ race conditions اللي الـ mutex بيحلها.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14: Kernel Debugging Techniques**
  بيتكلم عن `CONFIG_DEBUG_MUTEXES` وازاي تستخدم الـ lockdep في الـ embedded systems.

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)
- **الفصل 5: Kernel Synchronization**
  تحليل عميق لـ locking mechanisms بما فيها الـ mutex على مستوى الـ architecture.

---

### مصادر تكميلية

- **[linux-insides: Mutex](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-4.html)** — شرح تفصيلي بالأمثلة لتطبيق الـ mutex في الـ kernel من كتاب "Linux Insides" مفتوح المصدر.
- **[kernel.org: mutex-design.rst](https://www.kernel.org/doc/Documentation/locking/mutex-design.rst)** — النسخة الـ plain text من التوثيق الرسمي.
- **[mjmwired: mutex-design.txt](https://mjmwired.net/kernel/Documentation/locking/mutex-design.txt)** — نسخة archived من الـ documentation.

---

### مصطلحات للبحث

لو عايز تلاقي معلومات أكتر، استخدم الـ search terms دي:

```
linux kernel mutex fastpath slowpath optimistic spinning
linux kernel mutex vs semaphore performance
linux kernel ww_mutex wait-wound deadlock
linux kernel PREEMPT_RT rtmutex
linux kernel lockdep mutex annotation
linux kernel mutex MCS lock osq_lock
linux kernel devm_mutex_init managed resources
linux kernel mutex_lock_interruptible killable
CONFIG_DEBUG_MUTEXES kernel hacking
linux kernel locking/mutex.c git history
```
## Phase 8: Writing simple module

الهدف: نكتب kernel module بيستخدم **kprobe** على `mutex_lock` عشان نراقب كل عملية lock بتحصل في النظام — مين عملها، على أنهي mutex، وكام مرة.

---

### الفكرة

**`mutex_lock`** هي أكثر دالة في الـ mutex API استخداماً — كل driver وكل subsystem بيستخدمها. الـ kprobe بيسمحلنا نحقن handler قبل تنفيذ الدالة مباشرة من غير ما نعدّل الـ kernel source، وده بيخليها مثالية للـ tracing والـ debugging.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * mutex_lock_tracer.c
 * Traces every call to mutex_lock() using a kprobe.
 * Prints: who called, which mutex, and a running counter.
 */

#include <linux/kernel.h>       /* pr_info, printk helpers */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/mutex.h>        /* struct mutex definition */
#include <linux/sched.h>        /* current, task_comm_len */
#include <linux/atomic.h>       /* atomic64_t for safe counter */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("Trace mutex_lock() calls via kprobe");

/* Running counter — incremented on every mutex_lock() entry */
static atomic64_t lock_count = ATOMIC64_INIT(0);

/*
 * pre_handler — called right BEFORE mutex_lock() executes.
 *
 * @p   : pointer to the kprobe struct we registered
 * @regs: CPU register snapshot at the probe site
 *
 * On x86-64, the first argument (struct mutex *lock) lives in %rdi.
 * regs_get_kernel_argument() gives us arg0 portably.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve the mutex pointer passed as first argument */
    struct mutex *lock = (struct mutex *)regs_get_kernel_argument(regs, 0);

    u64 count = atomic64_inc_return(&lock_count);

    pr_info("mutex_lock_tracer: [#%llu] task='%s' pid=%d mutex@%px owner_raw=%lx\n",
            count,
            current->comm,          /* name of the calling task */
            current->pid,
            lock,                   /* address of the mutex itself */
            atomic_long_read(&lock->owner)); /* raw owner field — 0 means unlocked */

    return 0; /* 0 = let execution continue normally */
}

/*
 * post_handler — called right AFTER mutex_lock() returns (optional).
 * We use it only to confirm the lock was actually acquired.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * At this point mutex_lock() has returned, so the calling task
     * owns the mutex. Just log a short confirmation at debug level.
     */
    pr_debug("mutex_lock_tracer: lock acquired by pid=%d\n", current->pid);
}

/* kprobe descriptor — we probe mutex_lock by symbol name */
static struct kprobe kp = {
    .symbol_name  = "mutex_lock",   /* kernel exports this symbol */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init mutex_tracer_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("mutex_lock_tracer: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("mutex_lock_tracer: planted kprobe on '%s' at %px\n",
            kp.symbol_name, kp.addr); /* kp.addr filled by kernel after register */
    return 0;
}

static void __exit mutex_tracer_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("mutex_lock_tracer: removed kprobe. total lock calls traced: %llu\n",
            atomic64_read(&lock_count));
}

module_init(mutex_tracer_init);
module_exit(mutex_tracer_exit);
```

---

### Makefile

```makefile
# Place next to mutex_lock_tracer.c
obj-m += mutex_lock_tracer.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# Build and load
make
sudo insmod mutex_lock_tracer.ko
# Watch output
sudo dmesg -wH | grep mutex_lock_tracer
# Unload
sudo rmmod mutex_lock_tracer
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/mutex.h` | محتاجينه عشان نعرف شكل `struct mutex` ونقرأ `lock->owner` |
| `linux/sched.h` | بيديّنا الـ macro `current` اللي بيرجع `task_struct` للـ process الحالي |
| `linux/atomic.h` | **`atomic64_t`** بيضمن إن الـ counter آمن عند الـ concurrent access من multiple CPUs |

#### الـ `handler_pre`

الـ pre_handler بيتنفذ **قبل** ما `mutex_lock` تشتغل، يعني لسه المتغير جاي باسمه. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان نجيب أول argument (الـ `struct mutex *`) بطريقة portable تشتغل على x86-64 وARM64 من غير ما نـhard-code اسم register.

#### الـ `handler_post`

الـ post_handler اختياري بس مفيد للتأكد إن الـ lock اتأخد فعلاً — ده مهم لأن `mutex_lock` ممكن تـblock لو المتكس مش available، فالـ post بيتنفذ بس لما ترجع، يعني اللحظة اللي الـ task بقى يمتلك فيها الـ lock.

#### الـ `lock_count` — `atomic64_t`

**`mutex_lock`** بتتنادى من CPUs مختلفة في نفس الوقت — لو استخدمنا `int` عادي هيحصل race condition. الـ `atomic64_inc_return` بيعمل increment وreturn في عملية واحدة لا تتقسم (atomic).

#### الـ `module_init` — `mutex_tracer_init`

`register_kprobe` بتزرع **breakpoint** (INT3 على x86) عند بداية `mutex_lock` في الذاكرة. لو فشلت (مثلاً الـ symbol مش موجود أو الـ kprobes معطّل) بنرجع الـ error فوراً بدل ما الـ module يشتغل بدون hook.

#### الـ `module_exit` — `mutex_tracer_exit`

**لازم** نعمل `unregister_kprobe` في الـ exit وإلا لما الـ module يتشال من الذاكرة، الـ breakpoint هيفضل موجود وهيـcall handler مش موجود — kernel panic مضمون. بنطبع الـ total count كـ summary مفيد.

---

### مثال على الـ output

```
[  +0.000042] mutex_lock_tracer: planted kprobe on 'mutex_lock' at ffffffff81234560
[  +0.012301] mutex_lock_tracer: [#1] task='kworker/0:1' pid=23 mutex@ffff888003a1c200 owner_raw=0
[  +0.012415] mutex_lock_tracer: [#2] task='systemd-journald' pid=412 mutex@ffff888012340080 owner_raw=0
[  +0.013100] mutex_lock_tracer: [#3] task='bash' pid=1204 mutex@ffff888004560100 owner_raw=0
```

- **`owner_raw=0`** معناها المتكس كان unlocked وأتأخد على طول.
- لو `owner_raw != 0` معناها فيه task تاني بيمسك المتكس وده بداية تحقيق في **lock contention**.
