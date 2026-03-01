## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `include/linux/spinlock.h` جزء من **LOCKING PRIMITIVES** subsystem في Linux kernel. الـ maintainers هم Peter Zijlstra وIngo Molnar وWill Deacon — وده من أهم الـ subsystems في الكيرنل. الـ tree بتاعه: `locking/core`.

---

### القصة من الأول: ليه أصلاً محتاجين locks؟

تخيل عندك محل بقالة فيه كاشير واحد بس. لو اتنين زبون حاولوا يدفعوا في نفس الوقت — مفيش مشكلة، الكاشير هيتعامل مع واحد بعد التاني.

دلوقتي تخيل عندك **4 كاشيرات** شغالين في نفس الوقت، وكلهم بيحاولوا يسجلوا مبيعات في **دفتر حسابات واحد مشترك**. لو اتنين كتبوا في نفس الوقت؟ الأرقام هتتكسر وهتلاقي بيانات فاسدة.

ده بالضبط المشكلة في الـ **SMP** (Symmetric Multi-Processing) — يعني أجهزة فيها أكتر من CPU. كل CPU ممكن يحاول يوصل لنفس الـ data structure في نفس اللحظة. محتاجين طريقة نضمن بيها إن **واحد بس في الوقت ده** بيعمل التعديل.

الـ **spinlock** هو أبسط وأسرع solution: بدل ما الـ thread ينام لحد ما الـ lock يتحرر (زي الـ mutex)، هو **يفضل يلف في loop** وينتظر (spinning). سريع جداً لو وقت الانتظار قصير.

---

### الصورة الكبيرة: هدف الملف

**الـ `spinlock.h`** هو الـ **"واجهة النهائية"** اللي الـ kernel code بيتعامل معاها. مش بيعرّف implementation، بيعمل الآتي:

1. **يجمع** كل طبقات الـ spinlock في مكان واحد — من الـ arch-specific لحد الـ high-level API.
2. **يبني** الـ `spin_lock()` / `spin_unlock()` API اللي كل الكيرنل بيستخدمه.
3. **يفرّق** بين بيئتين:
   - **SMP** (أكتر من CPU): يستخدم الـ real locking.
   - **UP** (CPU واحد): يحوّل الـ locks لـ NOPs في الغالب عشان مفيش داعي.
4. **يدعم** سيناريوهات مختلفة: مع disable للـ IRQs، مع disable للـ bottom halves، مع trylock.

---

### الطبقات (Layer Cake)

```
┌─────────────────────────────────────────────────────┐
│         spin_lock() / spin_unlock()                  │  ← الـ API النهائي
│              spinlock.h                              │    اللي بيستخدمه kernel code
├─────────────────────────────────────────────────────┤
│      raw_spin_lock() / raw_spin_unlock()             │  ← raw layer
│         spinlock_api_smp.h / spinlock_api_up.h       │
├─────────────────────────────────────────────────────┤
│      _raw_spin_lock() / do_raw_spin_lock()           │  ← kernel/locking/spinlock.c
│                                                      │
├─────────────────────────────────────────────────────┤
│      arch_spin_lock() / arch_spin_unlock()           │  ← arch-specific
│           asm/spinlock.h                             │    (x86, ARM, etc.)
└─────────────────────────────────────────────────────┘
```

السبب في وجود كل الطبقات دي: **PREEMPT_RT**. في الـ Real-Time kernels، الـ `spinlock_t` مش هي `raw_spinlock_t` — هي `rt_mutex` كاملة! الـ `spinlock.h` هو اللي بيخبّي الفرق ده عن باقي الكيرنل.

---

### الـ `raw_spinlock_t` vs `spinlock_t`

| | `raw_spinlock_t` | `spinlock_t` |
|---|---|---|
| **التعريف** | دايماً spinlock حقيقي | spinlock عادي أو rt_mutex حسب config |
| **PREEMPT_RT** | مش بيتغير — spinning فعلي | بيتحول لـ rt_mutex |
| **متى تستخدمه؟** | في الـ low-level code (interrupt handlers, core scheduler) | في الـ normal kernel code |
| **مثال** | `raw_spin_lock(&rq->lock)` في الـ scheduler | `spin_lock(&dev->lock)` في driver |

---

### سيناريوهات مهمة: ليه في variants كتير؟

**المشكلة:** لو أنت لاقط الـ lock وجه interrupt على نفس الـ CPU وهو برضو عايز نفس الـ lock — **deadlock!**

```
CPU0: spin_lock(&mylock)     ← اخد الـ lock
CPU0: [interrupt fires!]
CPU0: spin_lock(&mylock)     ← يلف للأبد، الـ lock مش هيتحرر!
                               ← DEADLOCK
```

الحل: الـ variants المختلفة:

```c
spin_lock_irq(lock)         /* disable IRQs + take lock */
spin_lock_irqsave(lock, flags)  /* save IRQ state + disable + take lock */
spin_lock_bh(lock)          /* disable softirqs + take lock */
```

---

### الـ `smp_mb__after_spinlock()` — الجانب الخفي

الـ spinlock مش بس لحماية الـ data — ده أيضاً **memory barrier**. الـ `smp_mb__after_spinlock()` بيضمن إن التعديلات اللي حصلت قبل الـ lock تبقى مرئية لكل الـ CPUs التانية بعد ما تاخد الـ lock. ده مهم جداً في الـ scheduler (`__schedule()`) وفي `try_to_wake_up()`.

---

### الـ Lock Guards (الـ RAII style)

في آخر الملف، في macro patterns زي:

```c
DEFINE_LOCK_GUARD_1(spinlock, spinlock_t,
                    spin_lock(_T->lock),
                    spin_unlock(_T->lock))
```

دي بتسمح بكتابة كود زي:

```c
scoped_guard(spinlock, &mylock) {
    /* critical section */
}  /* unlock automatic */
```

بدل إنك تنسى الـ `spin_unlock()`، الـ cleanup بيحصل أوتوماتيك عند نهاية الـ scope — زي RAII في C++.

---

### الـ `alloc_bucket_spinlocks` — Scalable Locking

للـ data structures الكبيرة اللي فيها تنافس عالي (مثلاً hash tables)، مش منطقي تستخدم lock واحد. الفكرة: تعمل **array من الـ locks** وكل element يروح على lock معين بالـ hash. ده بيقلل الـ contention بشكل كبير.

---

### الملفات المرتبطة اللي المفروض تعرفها

**Core Headers:**
- `include/linux/spinlock.h` — الملف ده، الـ API النهائي
- `include/linux/spinlock_types.h` — تعريف `spinlock_t` و`raw_spinlock_t`
- `include/linux/spinlock_types_raw.h` — struct الـ `raw_spinlock` الفعلي
- `include/linux/spinlock_api_smp.h` — prototypes لـ SMP
- `include/linux/spinlock_api_up.h` — prototypes لـ Uniprocessor
- `include/linux/spinlock_rt.h` — الـ PREEMPT_RT implementation
- `include/linux/spinlock_up.h` — الـ UP (non-debug) implementation
- `include/linux/rwlock.h` — الـ read-write lock (مبني فوق spinlock)
- `include/linux/rwlock_types.h` — تعريف `rwlock_t`

**Arch-specific:**
- `arch/x86/include/asm/spinlock.h` — x86 implementation
- `arch/arm64/include/asm/spinlock.h` — ARM64 implementation
- `arch/*/include/asm/spinlock_types.h` — `arch_spinlock_t` لكل معمارية

**Implementation:**
- `kernel/locking/spinlock.c` — الـ `_raw_spin_lock()` وأخواته
- `kernel/locking/spinlock_debug.c` — debug checks
- `kernel/locking/spinlock_rt.c` — PREEMPT_RT implementation
- `kernel/locking/qspinlock.c` — الـ queued spinlock (الـ default على معظم architectures)
- `kernel/locking/mcs_spinlock.h` — الـ MCS algorithm اللي بيبني عليه qspinlock

**Related Subsystems:**
- `include/linux/lockdep.h` — الـ lock dependency validator (يكتشف deadlocks في runtime)
- `include/linux/mutex.h` — الـ sleeping lock (للـ contexts اللي تقدر تنام فيها)
- `include/linux/seqlock.h` — optimistic read locking
## Phase 2: شرح الـ Spinlock Framework

### المشكلة: ليه الـ Spinlock موجود أصلاً؟

في أي نظام multiprocessor، عندك أكتر من CPU بتشتغل في نفس الوقت، وكل واحدة ممكن تحاول توصل لنفس الـ data structure في نفس اللحظة. من غير أي تنسيق، بتحصل **race condition** — واحدة بتقرأ بينما التانية بتكتب، أو الاتنين بيكتبوا في نفس الوقت.

مثال حقيقي: الـ network stack بيستخدم قائمة لـ packet buffers. لو CPU0 بتضيف packet وCPU1 بتشيل packet في نفس اللحظة بدون lock، ممكن الـ linked list تتكسر تماماً.

الـ sleep-based locks زي `mutex_t` بتحل المشكلة دي لكن بتحتاج الـ scheduler يتدخل — ده overhead كبير جداً لو الـ critical section صغيرة (بضع instructions). لازم يكون في آلية أسرع.

---

### الحل: الـ Busy-Wait على الـ Lock

**الـ Spinlock** هو lock بسيط: لو مش متاح، الـ CPU تفضل تدور في loop ("spinning") لحد ما يتفك. مفيش context switch، مفيش scheduler. التكلفة صغيرة جداً لو الـ critical section قصيرة.

الشرط الجوهري: **مينفعش تنام وانت شايل spinlock**. لو نمت وانت شايل spinlock، أي CPU تانية محتاجاه هتفضل spinning للأبد — deadlock.

---

### التشبيه: الـ Toilet في Office

تخيل باب تواليت بيفتح للداخل وفيه مقبض. الشغل كالتالي:

| الـ Concept | الـ Kernel Equivalent |
|---|---|
| المقبض نفسه | `arch_spinlock_t raw_lock` |
| محاولة تحريك المقبض | `arch_spin_trylock()` |
| الواقف بره بيحاول كل ثانية | الـ spinning loop في `_raw_spin_lock()` |
| دخلت وغلّقت الباب | `spin_lock()` — CPU دخلت الـ critical section |
| فتحت الباب وخرجت | `spin_unlock()` |
| واحد واقف طابور | مفيش! الـ spinlock مش قائمة، كل الناس بتدور في نفس الوقت |
| لو حد نام جوا | deadlock — باقي الناس هيفضلوا واقفين برا للأبد |

الفرق النهائي عن الـ mutex: في الـ mutex، الواحد اللي ملقاش التواليت فاضي بيروح يذاكر ويرجع. في الـ spinlock، بيوقف يحدق في المقبض كل millisecond.

---

### الصورة الكبيرة: الـ Spinlock في الـ Kernel

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Consumer Code                            │
  │   net/core/  │  fs/ext4/  │  drivers/  │  kernel/sched/        │
  └──────┬───────┴─────┬──────┴─────┬───────┴──────────┬───────────┘
         │             │            │                   │
         ▼             ▼            ▼                   ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              Generic API  (spinlock.h)                          │
  │   spin_lock()  spin_unlock()  spin_lock_irqsave()  ...         │
  └──────────────────────────────┬──────────────────────────────────┘
                                 │
                    ┌────────────┴──────────────┐
                    ▼                           ▼
         ┌──────────────────┐       ┌───────────────────────┐
         │  CONFIG_PREEMPT_RT=n     │  CONFIG_PREEMPT_RT=y  │
         │  raw_spinlock_t  │       │  rt_mutex_base         │
         └────────┬─────────┘       └───────────────────────┘
                  │
          ┌───────┴────────┐
          ▼                ▼
  ┌──────────────┐  ┌──────────────┐
  │ CONFIG_SMP=y │  │ CONFIG_SMP=n │
  │ asm/spinlock │  │ spinlock_up.h│
  │ (ticket/mcs) │  │ (NOP mostly) │
  └──────┬───────┘  └──────────────┘
         │
         ▼
  ┌──────────────────────────────────┐
  │    arch_spinlock_t               │
  │  ARM64: قائمة ticket             │
  │  x86:   قائمة ticket أو queued   │
  └──────────────────────────────────┘
```

---

### طبقات الـ Abstraction: من فوق لتحت

الـ framework عنده 3 طبقات رئيسية:

```
┌──────────────────────────────────────────────────┐
│  Layer 3: spin_lock() / spin_unlock()             │
│  الـ API اللي أي كود في الـ kernel بيستخدمه      │
│  ده wrapper على raw_spin_lock()                   │
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│  Layer 2: raw_spin_lock() / raw_spin_unlock()     │
│  بيعمل: disable preemption + IRQs لو لازم        │
│  بيعمل: lockdep annotation                       │
│  بينادي: do_raw_spin_lock()                       │
└──────────────────────┬───────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────┐
│  Layer 1: arch_spin_lock() / arch_spin_unlock()   │
│  ده الـ low-level hardware implementation        │
│  ARM64: LL/SC loop أو ticket algorithm            │
│  x86: LOCK CMPXCHG أو queued spinlock            │
└──────────────────────────────────────────────────┘
```

---

### الـ Struct Hierarchy

```
spinlock_t
└── struct spinlock
    └── union
        └── struct raw_spinlock  (rlock)
            ├── arch_spinlock_t raw_lock        ← الـ actual hardware lock
            ├── unsigned int magic              ← (CONFIG_DEBUG_SPINLOCK)
            ├── unsigned int owner_cpu          ← (CONFIG_DEBUG_SPINLOCK)
            ├── void *owner                     ← (CONFIG_DEBUG_SPINLOCK)
            └── struct lockdep_map dep_map      ← (CONFIG_DEBUG_LOCK_ALLOC)
                ├── struct lock_class_key *key
                ├── const char *name
                └── enum lockdep_wait_type wait_type_inner
```

الـ `arch_spinlock_t` نفسه بيتعرّف في `asm/spinlock_types.h` — يختلف من architecture لأخرى. في ARM64 مثلاً بيكون ticket-based:

```c
/* ARM64 arch_spinlock_t مثال مبسط */
typedef struct {
    union {
        u32 slock;
        struct __raw_tickets {
            u16 owner;   /* الـ ticket اللي شغال دلوقتي */
            u16 next;    /* الـ ticket التالي هيتوزع */
        } tickets;
    };
} arch_spinlock_t;
```

---

### الـ SMP vs UP: ليه الـ Spinlock NOP على UP؟

على **Uniprocessor (UP)**، مفيش CPU تانية تتسابق معاك، فالـ spinlock الوحيد اللي تحتاجه هو تعطيل الـ preemption أو الـ IRQs (عشان interrupt handler ميجيش في الوسط). الـ `arch_spin_lock()` بيبقى NOP تقريباً.

على **SMP**، الـ loop الفعلية لازم تشتغل لأن CPU تانية ممكن فعلاً تكون شايلة الـ lock.

---

### أنواع الـ Lock Variants: الجدول الكامل

| API | بيعمل إيه إضافي |
|---|---|
| `spin_lock(lock)` | disable preemption فقط |
| `spin_lock_bh(lock)` | disable preemption + softirqs (bottom half) |
| `spin_lock_irq(lock)` | disable preemption + **كل الـ IRQs** |
| `spin_lock_irqsave(lock, flags)` | زي فوق + بيحفظ IRQ state في `flags` |
| `spin_trylock(lock)` | بيحاول مرة واحدة، بيرجع 0 أو 1 |

**متى تستخدم إيه؟**

- لو الـ data بيتشال من interrupt handler → لازم `spin_lock_irq()` أو `spin_lock_irqsave()` عشان الـ interrupt handler نفسه محتاج نفس الـ lock
- لو في softirq context → `spin_lock_bh()` يكفي
- لو pure process context و مفيش interrupt handler يمسّ الـ data → `spin_lock()` يكفي

---

### الـ raw_spinlock vs spinlock: الفرق الجوهري

ده واحد من أهم الـ concepts في الـ kernel modern:

| | `spinlock_t` | `raw_spinlock_t` |
|---|---|---|
| على non-RT kernel | نفس الـ `raw_spinlock_t` | نفس الـ `raw_spinlock_t` |
| على PREEMPT_RT kernel | يتحول لـ `rt_mutex` (sleeping!) | يفضل spinlock حقيقي |
| لو نمت وانت شايله | على RT: مسموح | على RT: deadlock / bug |
| الاستخدام | كل الـ drivers العادية | الـ RT-critical paths فقط |

الـ `PREEMPT_RT` هو kernel config بيخلي كل الـ spinlocks قابلة للـ preemption عن طريق تحويلها لـ mutex. ده بيحسن الـ real-time latency بشكل جذري. لكن بعض الأكواد اللي بتشتغل في hardware-critical context (زي interrupt handlers) لازم تستخدم `raw_spinlock_t` عشان ما تتحولش لـ mutex أبداً.

---

### الـ Lockdep: الـ Lock Validator

**الـ Lockdep** subsystem مهم جداً لفهم الـ spinlock. هو runtime validator بيتتبع:

1. ترتيب الـ lock acquisition — لو حصل deadlock potential بيصرخ
2. مين أخد الـ lock وفين في الـ code
3. نوع الـ wait context المسموح بيه (`LD_WAIT_SPIN` vs `LD_WAIT_CONFIG` vs `LD_WAIT_SLEEP`)

```c
/* من lockdep_types.h */
enum lockdep_wait_type {
    LD_WAIT_SPIN,    /* لا ينام أبداً — raw_spinlock_t */
    LD_WAIT_CONFIG,  /* ممكن ينام على RT — spinlock_t */
    LD_WAIT_SLEEP,   /* ينام عادي — mutex */
};
```

الـ `dep_map` جوا الـ `raw_spinlock_t` هو اللي الـ lockdep بيستخدمه عشان يعرف كل معلومة عن الـ lock ده. على `CONFIG_DEBUG_LOCK_ALLOC=n` بيكون empty struct.

---

### الـ Memory Barriers و smp_mb__after_spinlock()

ده concept دقيق جداً. الـ spinlock بيعمل أكتر من مجرد exclusion — بيعمل **memory ordering** كمان.

الـ `smp_mb__after_spinlock()` بيضمن إن أي CPU أخدت الـ lock وشافت القيمة X، تشوف كل الـ writes اللي عملتها الـ CPU اللي أخدت الـ lock قبليها.

```
CPU0                          CPU1
────────────────────          ────────────────────
WRITE_ONCE(X, 1);             WRITE_ONCE(Y, 1);
spin_lock(S);                 smp_mb();
smp_mb__after_spinlock();     r1 = READ_ONCE(X);
r0 = READ_ONCE(Y);
spin_unlock(S);
```

المحظور: مينفعش r0=0 و r1=0 في نفس الوقت. الـ spinlock بيضمن إن الـ CPU1 لو شافت Y=1، CPU0 شافت X=1.

ده بيخلي الـ spinlock يشتغل كـ **RCsc (Release Consistency sequentially consistent)** lock.

---

### الـ DEFINE_LOCK_GUARD_1: الـ Scoped Locking

الـ kernel modern بيدعم الـ scoped locking عبر الـ cleanup mechanism (C23 style):

```c
/* من spinlock.h */
DEFINE_LOCK_GUARD_1(spinlock, spinlock_t,
        spin_lock(_T->lock),
        spin_unlock(_T->lock))
```

ده بيخلي الكود أبسط وأأمن:

```c
/* الطريقة القديمة */
spin_lock(&dev->lock);
/* ... */
spin_unlock(&dev->lock);  /* لو نسيتها = bug */

/* الطريقة الجديدة بـ guard */
guard(spinlock)(&dev->lock);
/* spin_unlock بيتعمل automatically عند خروج الـ scope */
```

---

### مين يمتلك إيه: الـ Framework vs الـ Driver

| المسؤولية | الـ Framework (spinlock.h) | الـ Driver / Consumer |
|---|---|---|
| تعطيل الـ preemption | نعم — تلقائي | لا |
| حفظ IRQ flags | نعم — لو استخدم irqsave | لا |
| الـ busy-wait loop | نعم — في `_raw_spin_lock` | لا |
| الـ lockdep tracking | نعم — تلقائي | لا |
| تعريف الـ arch_spinlock_t | لا — ده الـ architecture | هو |
| اللي بيتعمل جوا الـ lock | لا | الـ driver |
| اختيار نوع الـ lock | لا | الـ driver (raw vs normal) |
| تخصيص وتحرير الـ lock | جزئياً (alloc_bucket_spinlocks) | الـ driver |

---

### مثال حقيقي: Network Driver

```c
/* مثال مبسط: Ethernet driver */
struct my_eth_dev {
    spinlock_t tx_lock;          /* يحمي الـ TX queue */
    struct sk_buff_head tx_queue;
    /* ... */
};

/* في الـ interrupt handler (softirq context) */
static void my_eth_tx_complete(struct my_eth_dev *dev)
{
    struct sk_buff *skb;

    /* بنستخدم spin_lock_bh لأن الـ TX completion
     * بتيجي من softirq، وعايزين نمنع softirq تانية
     * من نفس الـ CPU تيجي تحاول تاخد الـ lock */
    spin_lock_bh(&dev->tx_lock);
    skb = skb_dequeue(&dev->tx_queue);
    spin_unlock_bh(&dev->tx_lock);

    if (skb)
        dev_kfree_skb_any(skb);
}

/* في process context (مثلاً من ndo_start_xmit) */
static netdev_tx_t my_eth_xmit(struct sk_buff *skb,
                                struct net_device *ndev)
{
    struct my_eth_dev *dev = netdev_priv(ndev);

    /* نفس الـ lock، بس من process context
     * لازم نستخدم bh عشان softirq ممكن يتنفذ
     * على نفس الـ CPU ويحاول ياخد الـ lock */
    spin_lock_bh(&dev->tx_lock);
    skb_queue_tail(&dev->tx_queue, skb);
    spin_unlock_bh(&dev->tx_lock);

    return NETDEV_TX_OK;
}
```

---

### الـ Contention Detection و spin_needbreak()

```c
/* من spinlock.h */
static inline int spin_needbreak(spinlock_t *lock)
{
    if (!preempt_model_preemptible())
        return 0;
    return spin_is_contended(lock);
}
```

ده بيُستخدم في الـ loops الطويلة عشان نتحقق هل في حد تاني واقف ينتظر الـ lock. لو آه، نفك الـ lock بسرعة ونعمل `cond_resched()` عشان نديه فرصة. بيحسن الـ fairness بشكل كبير.

---

### ملخص: الـ Core Abstraction

الـ spinlock framework بيقدم **طبقة abstraction موحدة** فوق الـ hardware primitives. الفكرة المحورية هي:

> الـ API الواحد (`spin_lock`) بيتصرف صح على SMP وUP، مع وبدون PREEMPT_RT، مع وبدون DEBUG، وفي كل الـ contexts الممكنة — من خلال شبكة من الـ macros والـ inline functions اللي بتختار الـ implementation الصح وقت الـ compile.

الـ driver بيشوف API نظيف وبسيط. الـ kernel بيهندل الـ complexity كلها من تحت.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Macros والـ Config Options — Cheatsheet

#### Config Options الأساسية

| Config Option | التأثير |
|---|---|
| `CONFIG_SMP` | يفعّل الـ real locking — بدونه الـ spinlock بيبقى NOP |
| `CONFIG_PREEMPT_RT` | يحوّل `spinlock_t` من raw spinlock لـ `rt_mutex` كامل |
| `CONFIG_DEBUG_SPINLOCK` | يضيف magic number وتتبع الـ owner — بيكتشف double-lock وغيره |
| `CONFIG_DEBUG_LOCK_ALLOC` | يضيف `lockdep_map` لكل lock — بيفعّل الـ lockdep tracking |
| `CONFIG_PREEMPTION` | بيتحكم في الـ preemption behavior جوه الـ critical section |

#### Magic Numbers والـ Constants

| Constant | القيمة | الغرض |
|---|---|---|
| `SPINLOCK_MAGIC` | `0xdead4ead` | تحقق من سلامة الـ raw_spinlock |
| `RWLOCK_MAGIC` | `0xdeaf1eed` | تحقق من سلامة الـ rwlock |
| `SPINLOCK_OWNER_INIT` | `((void *)-1L)` | قيمة افتراضية لما مفيش owner |
| `LOCK_SECTION_NAME` | `.text..lock.*` | section في الـ binary للـ lock-related code |

#### Macros الإنشاء والتهيئة

| Macro | الاستخدام |
|---|---|
| `DEFINE_SPINLOCK(x)` | تعريف وتهيئة `spinlock_t` بشكل static |
| `DEFINE_RAW_SPINLOCK(x)` | تعريف وتهيئة `raw_spinlock_t` |
| `DEFINE_RWLOCK(x)` | تعريف وتهيئة `rwlock_t` |
| `spin_lock_init(lock)` | تهيئة `spinlock_t` dynamic |
| `raw_spin_lock_init(lock)` | تهيئة `raw_spinlock_t` dynamic |

#### الـ LD_WAIT Types (لـ lockdep)

| Type | المعنى |
|---|---|
| `LD_WAIT_SPIN` | الـ lock ده spin lock — مينفعش تنام وانت شايله |
| `LD_WAIT_CONFIG` | حسب الـ config — لو RT ممكن ينام |
| `LD_LOCK_PERCPU` | الـ lock ده per-CPU — زي local spinlocks |

---

### الـ Structs المهمة

#### 1. `raw_spinlock_t` (المعرّف في `spinlock_types_raw.h`)

**الغرض:** الـ building block الأساسي لكل الـ spinlocks في الـ kernel. ده الـ lock اللي بيعمل spin حقيقي.

```c
struct raw_spinlock {
    arch_spinlock_t raw_lock;    /* الـ arch-specific lock — x86: ticket/qspinlock */

#ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int magic;          /* 0xdead4ead — للتحقق من السلامة */
    unsigned int owner_cpu;      /* رقم الـ CPU اللي شايل الـ lock */
    void *owner;                 /* pointer للـ task اللي شايل الـ lock */
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;  /* lockdep metadata */
#endif
};
typedef struct raw_spinlock raw_spinlock_t;
```

**العلاقات:**
- يحتوي على `arch_spinlock_t` — ده الـ hardware-level value
- يُستخدم داخل `spinlock_t` كـ `rlock`
- يُستخدم مباشرة في الـ RT kernel لما تحتاج lock مش بيبقى mutex

---

#### 2. `spinlock_t` (المعرّف في `spinlock_types.h`)

**الغرض:** الـ high-level spinlock اللي بيستخدمه الـ kernel code العادي. سلوكه بيتغير حسب الـ config.

**في الـ non-RT kernel:**
```c
struct spinlock {
    union {
        struct raw_spinlock rlock;  /* الـ actual lock */

#ifdef CONFIG_DEBUG_LOCK_ALLOC
        struct {
            u8 __padding[LOCK_PADSIZE];  /* padding لمحاذاة الـ dep_map */
            struct lockdep_map dep_map;  /* lockdep overlay */
        };
#endif
    };
};
typedef struct spinlock spinlock_t;
```

**في الـ PREEMPT_RT kernel:**
```c
struct spinlock {
    struct rt_mutex_base lock;   /* mutex حقيقي بدل الـ spin! */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};
```

**العلاقات:**
- يحتوي على `raw_spinlock` في الـ non-RT
- يحتوي على `rt_mutex_base` في الـ PREEMPT_RT
- الـ `spin_lock()` API بتتكلم على `spinlock_t` دايمًا

---

#### 3. `rwlock_t` (المعرّف في `rwlock_types.h`)

**الغرض:** Read-Write Lock — بيسمح لـ multiple readers في نفس الوقت، لكن writer واحد بس.

**في الـ non-RT kernel:**
```c
struct rwlock {
    arch_rwlock_t raw_lock;      /* الـ arch-specific RW lock */

#ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int magic;          /* 0xdeaf1eed */
    unsigned int owner_cpu;
    void *owner;
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};
typedef struct rwlock rwlock_t;
```

**في الـ PREEMPT_RT kernel:**
```c
struct rwlock {
    struct rwbase_rt rwbase;   /* RT-safe rwlock implementation */
    atomic_t readers;          /* عداد الـ readers الحاليين */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};
```

---

#### 4. `arch_spinlock_t` (المعرّف في `asm/spinlock_types.h`)

**الغرض:** الـ implementation الـ architecture-specific. على x86 بيستخدم **queued spinlock**.

```c
/* x86 example — قد يختلف على ARM/RISC-V */
typedef struct qspinlock {
    union {
        atomic_t val;
        /* ... ticket fields ... */
    };
} arch_spinlock_t;
```

---

#### 5. `lockdep_map` (المعرّف في `lockdep_types.h`)

**الغرض:** metadata بيستخدمه الـ lockdep subsystem لتتبع الـ lock ordering وكشف الـ deadlocks.

الـ fields المهمة:
- `name`: اسم الـ lock (string)
- `wait_type_inner`: نوع الانتظار (`LD_WAIT_SPIN`, `LD_WAIT_CONFIG`)
- `lock_type`: نوع الـ lock (`LD_LOCK_PERCPU` وغيره)

---

### مخطط العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────────────┐
                    │             spinlock_t                  │
                    │  (الـ high-level API للـ kernel code)    │
                    │                                         │
                    │  non-RT:         RT:                    │
                    │  ┌────────────┐  ┌──────────────────┐  │
                    │  │raw_spinlock│  │  rt_mutex_base   │  │
                    │  │   rlock    │  │     lock         │  │
                    │  └─────┬──────┘  └──────────────────┘  │
                    └────────│────────────────────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │        raw_spinlock_t        │
              │  (الـ raw layer — يدور spin) │
              │                              │
              │  ┌──────────────────────┐    │
              │  │   arch_spinlock_t    │    │
              │  │      raw_lock        │    │
              │  │  (x86: qspinlock)    │    │
              │  └──────────────────────┘    │
              │                              │
              │  [CONFIG_DEBUG_SPINLOCK]      │
              │  magic / owner_cpu / owner   │
              │                              │
              │  [CONFIG_DEBUG_LOCK_ALLOC]    │
              │  ┌──────────────────────┐    │
              │  │    lockdep_map       │    │
              │  │      dep_map         │    │
              │  └──────────────────────┘    │
              └──────────────────────────────┘


                    ┌─────────────────────────────────────────┐
                    │              rwlock_t                   │
                    │                                         │
                    │  non-RT:            RT:                 │
                    │  ┌─────────────┐   ┌─────────────────┐ │
                    │  │arch_rwlock_t│   │  rwbase_rt      │ │
                    │  │  raw_lock   │   │  atomic_t       │ │
                    │  └─────────────┘   │  readers        │ │
                    │                    └─────────────────┘ │
                    └─────────────────────────────────────────┘
```

---

### مخطط طبقات الـ API

```
┌──────────────────────────────────────────────────────────────┐
│                   LAYER 3: Public API                        │
│   spin_lock()  spin_unlock()  spin_lock_irq()  spin_lock_bh()│
│   spin_trylock()  spin_lock_irqsave()  spin_needbreak()      │
│   (تشتغل على spinlock_t)                                     │
└──────────────────────┬───────────────────────────────────────┘
                       │ (maps to) non-RT
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   LAYER 2: Raw API                           │
│   raw_spin_lock()  raw_spin_unlock()  raw_spin_trylock()     │
│   raw_spin_lock_irq()  raw_spin_lock_bh()  raw_spin_...()   │
│   (تشتغل على raw_spinlock_t)                                  │
└──────────────────────┬───────────────────────────────────────┘
                       │ calls
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   LAYER 1: do_raw_* functions                │
│   do_raw_spin_lock()  do_raw_spin_trylock()                  │
│   do_raw_spin_unlock()                                       │
│   (بيكلم arch_ + mmiowb_)                                    │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                   LAYER 0: arch API                          │
│   arch_spin_lock()  arch_spin_trylock()  arch_spin_unlock()  │
│   (asm/spinlock.h — inline assembly)                         │
└──────────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ Spinlock

```
   INITIALIZATION
   ──────────────
   DEFINE_SPINLOCK(my_lock)          ← static (compile time)
         OR
   spin_lock_init(&my_lock)          ← dynamic (runtime)
         │
         │  بيحط raw_lock = UNLOCKED
         │  بيحط magic = 0xdead4ead (debug)
         │  بيسجّل في lockdep
         ▼
   UNLOCKED STATE
   ──────────────
   my_lock.rlock.raw_lock = { val=0 }  (qspinlock: unlocked)

         │
         │  spin_lock(&my_lock)
         ▼
   LOCKING
   ───────
   spin_lock()
     → raw_spin_lock()
       → _raw_spin_lock()               (spinlock_api_smp.h)
         → preempt_disable()            (disable kernel preemption)
           → do_raw_spin_lock()
             → arch_spin_lock()         (busy-wait هنا!)
               → mmiowb_spin_lock()
                 → [LOCKED]
         │
         ▼
   LOCKED STATE
   ────────────
   الـ CPU بيدور في spin لو حد تاني شايل الـ lock
   (busy-waiting — مش بينام — مش بيحصل context switch)

         │
         │  spin_unlock(&my_lock)
         ▼
   UNLOCKING
   ─────────
   spin_unlock()
     → raw_spin_unlock()
       → _raw_spin_unlock()
         → do_raw_spin_unlock()
           → mmiowb_spin_unlock()
             → arch_spin_unlock()       (memory barrier + release)
               → preempt_enable()       (re-enable preemption)
                 → [UNLOCKED]

         │
         ▼
   DESTRUCTION
   ───────────
   مفيش destructor صريح — بس لو dynamic:
   كفاية إن الـ struct اللي فيه الـ lock يتحرر (kfree مثلاً)
   لو DEBUG_LOCK_ALLOC: lockdep_unregister_key()
```

---

### مخطط Call Flow التفصيلي

#### `spin_lock()` — non-RT, SMP

```
spin_lock(&lock)                           [spinlock.h:338]
  │
  └─► raw_spin_lock(&lock->rlock)          [spinlock.h:341]
        │
        └─► _raw_spin_lock(lock)           [macro → spinlock_api_smp.h]
              │
              ├─► preempt_disable()        [preempt.h — يمنع الـ preemption]
              │
              └─► do_raw_spin_lock(lock)   [spinlock.h:184]
                    │
                    ├─► __acquire(lock)    [sparse annotation فقط]
                    │
                    ├─► arch_spin_lock(&lock->raw_lock)  [asm/spinlock.h]
                    │     │
                    │     └─► [x86 queued spinlock busy-wait loop]
                    │           بيستنى لحد ما الـ lock يتحرر
                    │
                    └─► mmiowb_spin_lock()  [asm/mmiowb.h]
                          بيضمن memory ordering لـ MMIO writes
```

#### `spin_lock_irqsave()` — الأكثر استخداماً في الـ drivers

```
spin_lock_irqsave(&lock, flags)            [spinlock.h:374]
  │
  └─► raw_spin_lock_irqsave(spinlock_check(lock), flags)
        │
        └─► _raw_spin_lock_irqsave(lock)   [spinlock_api_smp.h]
              │
              ├─► local_irq_save(flags)    [irqflags.h — يحفظ ويعطّل الـ IRQs]
              │
              ├─► preempt_disable()
              │
              └─► do_raw_spin_lock(lock)
                    └─► arch_spin_lock(...)
```

#### `spin_unlock_irqrestore()` — المقابل

```
spin_unlock_irqrestore(&lock, flags)
  │
  └─► raw_spin_unlock_irqrestore(&lock->rlock, flags)
        │
        └─► _raw_spin_unlock_irqrestore(lock, flags)
              │
              ├─► do_raw_spin_unlock(lock)
              │     ├─► mmiowb_spin_unlock()
              │     └─► arch_spin_unlock(...)    [memory barrier هنا]
              │
              ├─► preempt_enable()
              │
              └─► local_irq_restore(flags)       [بيرجّع الـ IRQs كما كانت]
```

#### `atomic_dec_and_lock()` — pattern مهم جداً

```
atomic_dec_and_lock(&refcount, &lock)
  │
  ├─► atomic_dec(refcount)
  │
  └─► if (refcount == 0):
        spin_lock(lock)
        return 1           ← Lock acquired, caller must handle cleanup
      else:
        return 0           ← Lock not acquired
```

---

### الـ Lock Guards (RAII pattern)

الـ file بيعرّف guard classes باستخدام `DEFINE_LOCK_GUARD_1`:

```
DEFINE_LOCK_GUARD_1(spinlock, spinlock_t,
    spin_lock(_T->lock),           ← acquire
    spin_unlock(_T->lock))         ← release (automatic)
```

**الاستخدام:**
```c
/* بدل: */
spin_lock(&my_lock);
/* ... */
spin_unlock(&my_lock);

/* تقدر تكتب: */
guard(spinlock)(&my_lock);
/* يتحرر أوتوماتيك لما الـ scope ينتهي */
```

#### جدول الـ Guards المتاحة

| Guard Name | Lock Type | Variant |
|---|---|---|
| `spinlock` | `spinlock_t` | lock/unlock |
| `spinlock_try` | `spinlock_t` | trylock |
| `spinlock_irq` | `spinlock_t` | + disable IRQ |
| `spinlock_irq_try` | `spinlock_t` | trylock + IRQ |
| `spinlock_bh` | `spinlock_t` | + disable BH |
| `spinlock_bh_try` | `spinlock_t` | trylock + BH |
| `spinlock_irqsave` | `spinlock_t` | + save IRQ flags |
| `spinlock_irqsave_try` | `spinlock_t` | trylock + irqsave |
| `raw_spinlock` | `raw_spinlock_t` | lock/unlock |
| `raw_spinlock_irq` | `raw_spinlock_t` | + disable IRQ |
| `raw_spinlock_bh` | `raw_spinlock_t` | + disable BH |
| `raw_spinlock_irqsave` | `raw_spinlock_t` | + save IRQ flags |
| `read_lock` | `rwlock_t` | read |
| `read_lock_irq` | `rwlock_t` | read + IRQ |
| `read_lock_irqsave` | `rwlock_t` | read + irqsave |
| `write_lock` | `rwlock_t` | write |
| `write_lock_irq` | `rwlock_t` | write + IRQ |
| `write_lock_irqsave` | `rwlock_t` | write + irqsave |

---

### استراتيجية الـ Locking

#### قاعدة الاختيار الأساسية

```
هل ممكن الـ lock يتاخد من IRQ handler؟
     │
     ├── نعم ──► spin_lock_irqsave() / spin_lock_irq()
     │           (لازم تعطّل الـ interrupts عشان تمنع deadlock)
     │
     └── لأ ──► هل ممكن من softirq/tasklet؟
                  │
                  ├── نعم ──► spin_lock_bh()
                  │           (بيعطّل الـ bottom halves)
                  │
                  └── لأ ──► spin_lock()
                              (في process context بس)
```

#### متى تستخدم `raw_spinlock_t` بدل `spinlock_t`؟

```
raw_spinlock_t يُستخدم لما:

1. أنت في الـ scheduler code نفسه
2. أنت في الـ lockdep code نفسه
3. الـ lock لازم يشتغل حتى على PREEMPT_RT
   (لأن spinlock_t بيبقى mutex على RT)
4. الـ timing critical لدرجة إن mutex مقبولش

مثال: قلب الـ scheduler يستخدم raw_spinlock على rq->lock
```

#### ترتيب الـ Locking (Lock Ordering) لمنع Deadlock

```
القاعدة الذهبية:
كل الـ code لازم ياخد الـ locks بنفس الترتيب دايمًا

مثال من الـ kernel:
  [1] rq->lock      (scheduler run queue)
  [2] task->pi_lock (priority inheritance)
  [3] inode->i_lock (filesystem inode)

لو CPU0 شال [1] ثم [2]، و CPU1 حاول ياخد [2] ثم [1]
  → DEADLOCK!

الـ lockdep بيكتشف ده أوتوماتيكاً بيحط log في dmesg
```

#### ما الـ locks بتحمي إيه؟

| Lock | ما بيحميه | السياق |
|---|---|---|
| `spin_lock()` | shared data بين كذا thread | process context فقط |
| `spin_lock_bh()` | data بين process و softirq | process + softirq |
| `spin_lock_irq()` | data بين process و IRQ handler | process + IRQ |
| `spin_lock_irqsave()` | زي فوق + بيحفظ IRQ state | أكثر أمانًا |
| `read_lock()` / `write_lock()` | بيانات read-mostly | multiple readers OK |

#### `smp_mb__after_spinlock()` — الـ Memory Barrier الخفي

ده barrier مهم بيضمن إن:
- الكتابات اللي حصلت **قبل** `spin_unlock()` على CPU0 تبقى مرئية لـ CPU1 بعد `spin_lock()`
- بيحوّل الـ lock لـ **RCsc** (Release Consistency — sequentially consistent)

```c
/* CPU0 */            /* CPU1 */
WRITE(X, 1);          spin_lock(S);
spin_lock(S);         smp_mb__after_spinlock();
spin_unlock(S);       r = READ(X);   /* ← مضمون يشوف 1 */
```

بيُستخدم في `__schedule()` و `try_to_wake_up()` بالذات.

---

### مخطط الـ Nested Locking

```
raw_spin_lock_nested(lock, subclass)
  │
  └─► بيخلي lockdep يعرف إن الـ locks دي من نفس الـ class
      لكن في مستويات مختلفة (مثال: locks في tree nodes)

      Node A (subclass=0)
        └─► Node B (subclass=1)
              └─► Node C (subclass=2)

      بدون nested، الـ lockdep كان هيصرّح بـ circular dependency
      لأن كل الـ nodes من نفس الـ struct type
```

---

### الـ Bucket Spinlocks

```c
/* بيخلق array من الـ spinlocks للـ hash tables */
__alloc_bucket_spinlocks(&locks, &lock_mask, max_size, cpu_mult, gfp, name, key)
```

```
بدل lock واحد على كل الـ hash table:

  [bucket 0] → spinlock_0
  [bucket 1] → spinlock_1
  [bucket 2] → spinlock_2
  ...
  [bucket N] → spinlock_N

الـ lock_mask = N-1 (للـ hash modulo)
كل bucket بياخد lock خاص بيه → أقل contention
```

---

### ملخص العلاقات الكاملة

```
                    spinlock.h
                       │
           ┌───────────┼───────────────┐
           │           │               │
    spinlock_types.h  rwlock.h    spinlock_api_smp.h
           │                           │
    spinlock_types_raw.h         _raw_spin_lock()
           │                     _raw_spin_unlock()
    ┌──────┴──────┐
    │             │
arch_spinlock_t  lockdep_map
(asm/spinlock)  (lockdep_types.h)
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ APIs — Cheatsheet

#### Raw Spinlock API

| Function / Macro | النوع | الغرض |
|---|---|---|
| `raw_spin_lock_init(lock)` | macro | تهيئة الـ lock |
| `raw_spin_lock(lock)` | macro | acquire بدون IRQ |
| `raw_spin_unlock(lock)` | macro | release |
| `raw_spin_trylock(lock)` | macro | محاولة acquire، non-blocking |
| `raw_spin_lock_irq(lock)` | macro | acquire + disable IRQ |
| `raw_spin_unlock_irq(lock)` | macro | release + enable IRQ |
| `raw_spin_lock_irqsave(lock, flags)` | macro | acquire + save IRQ state |
| `raw_spin_unlock_irqrestore(lock, flags)` | macro | release + restore IRQ state |
| `raw_spin_lock_bh(lock)` | macro | acquire + disable BH |
| `raw_spin_unlock_bh(lock)` | macro | release + enable BH |
| `raw_spin_trylock_irq(lock)` | macro | try + disable IRQ |
| `raw_spin_trylock_bh(lock)` | macro | try + disable BH |
| `raw_spin_trylock_irqsave(lock, flags)` | macro | try + save IRQ |
| `raw_spin_is_locked(lock)` | macro | فحص حالة الـ lock |
| `raw_spin_is_contended(lock)` | macro | فحص وجود waiters |
| `raw_spin_lock_nested(lock, subclass)` | macro | acquire مع lockdep subclass |
| `raw_spin_lock_nest_lock(lock, nest_lock)` | macro | acquire مع nest annotation |
| `raw_spin_lock_irqsave_nested(lock, flags, subclass)` | macro | irqsave + subclass |

#### Spinlock API (non-RT wrapper)

| Function / Macro | النوع | الغرض |
|---|---|---|
| `spin_lock_init(lock)` | macro | تهيئة spinlock_t |
| `spin_lock(lock)` | inline | acquire |
| `spin_unlock(lock)` | inline | release |
| `spin_trylock(lock)` | inline | non-blocking acquire |
| `spin_lock_irq(lock)` | inline | acquire + disable IRQ |
| `spin_unlock_irq(lock)` | inline | release + enable IRQ |
| `spin_lock_irqsave(lock, flags)` | macro | acquire + save IRQ |
| `spin_unlock_irqrestore(lock, flags)` | inline | release + restore IRQ |
| `spin_lock_bh(lock)` | inline | acquire + disable BH |
| `spin_unlock_bh(lock)` | inline | release + enable BH |
| `spin_trylock_bh(lock)` | inline | try + disable BH |
| `spin_trylock_irq(lock)` | inline | try + disable IRQ |
| `spin_trylock_irqsave(lock, flags)` | macro | try + save IRQ |
| `spin_is_locked(lock)` | inline | فحص حالة الـ lock |
| `spin_is_contended(lock)` | inline | فحص وجود waiters |
| `spin_needbreak(lock)` | inline | هل محتاج نكسر الـ critical section؟ |
| `spin_lock_nested(lock, subclass)` | macro | acquire مع subclass |
| `spin_lock_nest_lock(lock, nest_lock)` | macro | acquire مع nest annotation |
| `spin_lock_irqsave_nested(lock, flags, subclass)` | macro | irqsave + subclass |
| `assert_spin_locked(lock)` | macro | BUG_ON لو مش locked |

#### Low-level do_raw_* Layer

| Function | الغرض |
|---|---|
| `do_raw_spin_lock(lock)` | استدعاء arch_spin_lock مباشرة |
| `do_raw_spin_trylock(lock)` | استدعاء arch_spin_trylock مباشرة |
| `do_raw_spin_unlock(lock)` | استدعاء arch_spin_unlock مباشرة |

#### Atomic + Spinlock Helpers

| Function / Macro | الغرض |
|---|---|
| `atomic_dec_and_lock(atomic, lock)` | decrement + conditional lock |
| `atomic_dec_and_lock_irqsave(atomic, lock, flags)` | decrement + conditional lock + save IRQ |
| `atomic_dec_and_raw_lock(atomic, lock)` | نفس السابق على raw_spinlock |
| `atomic_dec_and_raw_lock_irqsave(atomic, lock, flags)` | نفس السابق + IRQ save |

#### Bucket Spinlocks

| Function / Macro | الغرض |
|---|---|
| `alloc_bucket_spinlocks(locks, lock_mask, max_size, cpu_mult, gfp)` | تخصيص array من الـ locks |
| `free_bucket_spinlocks(locks)` | تحرير الـ array |

#### Memory Barrier

| Macro | الغرض |
|---|---|
| `smp_mb__after_spinlock()` | full memory barrier بعد الـ lock acquire |

---

### المجموعة الأولى: Initialization

الهدف من هذه المجموعة هو تهيئة بنى الـ lock قبل أي استخدام. الـ kernel يفرق بين non-debug build (مجرد write للـ initializer value) وdebug build (تسجيل الـ lock في الـ lockdep subsystem).

---

#### `raw_spin_lock_init`

```c
// Non-debug:
#define raw_spin_lock_init(lock) \
    do { *(lock) = __RAW_SPIN_LOCK_UNLOCKED(lock); } while (0)

// Debug:
#define raw_spin_lock_init(lock)                                    \
do {                                                                \
    static struct lock_class_key __key;                             \
    __raw_spin_lock_init((lock), #lock, &__key, LD_WAIT_SPIN);      \
} while (0)
```

**الـ macro** بتكتب الـ initializer value في الـ `raw_spinlock_t`. في الـ debug build بتستدعي `__raw_spin_lock_init` اللي بتسجل الـ lock مع الـ lockdep ببيانات الـ `lock_class_key`.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t` المطلوب تهيئتها.

**Return:** void — لا يرجع قيمة.

**Key details:**
- الـ `LD_WAIT_SPIN` بتقول للـ lockdep إن الـ waiters يـ spin (مش يـ sleep).
- الـ `static struct lock_class_key __key` بتضمن إن كل call site له class منفصل في الـ lockdep graph.
- يُستخدم في dynamic initialization؛ للـ static variables يُفضل `DEFINE_RAW_SPINLOCK`.

---

#### `spin_lock_init`

```c
// Non-debug:
#define spin_lock_init(_lock)           \
do {                                    \
    spinlock_check(_lock);              \
    *(_lock) = __SPIN_LOCK_UNLOCKED(_lock); \
} while (0)

// Debug:
#define spin_lock_init(lock)                                            \
do {                                                                    \
    static struct lock_class_key __key;                                 \
    __raw_spin_lock_init(spinlock_check(lock),                          \
                         #lock, &__key, LD_WAIT_CONFIG);                \
} while (0)
```

**الـ macro** بتهيئ `spinlock_t`. الفرق عن `raw_spin_lock_init` إن الـ wait type هو `LD_WAIT_CONFIG` (مش `LD_WAIT_SPIN`) يعني الـ lockdep عارف إن في بعض الـ configs الـ lock دي ممكن تـ sleep (RT kernels).

**Parameters:**
- `_lock` / `lock` — pointer لـ `spinlock_t`.

**Key details:**
- `spinlock_check` بترجع `&lock->rlock` وبكده بتعمل type-check ضمني.
- الفرق الجوهري بين `raw_spinlock_t` و`spinlock_t` في الـ non-RT kernel هو semantic فقط؛ في الـ RT kernel الـ `spinlock_t` بتتحول لـ `rt_mutex`.

---

#### `spinlock_check`

```c
static __always_inline raw_spinlock_t *spinlock_check(spinlock_t *lock)
{
    return &lock->rlock;
}
```

**Helper** بسيط بيرجع `&lock->rlock`. الغرض الأساسي هو الـ type-checking — لو حد حاول يمرر حاجة مش `spinlock_t *` المترجم هيشتكي.

**Parameters:**
- `lock` — pointer لـ `spinlock_t`.

**Return:** `raw_spinlock_t *` — pointer للـ `rlock` field الجوفي.

**Caller context:** مستخدمة داخل الـ macros (`spin_lock_init`, `spin_lock_nested`, `spin_lock_irqsave`).

---

### المجموعة الثانية: Low-level Arch Layer (do_raw_*)

هذه الـ functions هي أدنى طبقة قبل الـ arch-specific code. في non-debug build هي inline وبتستدعي `arch_spin_*` مباشرة. في debug build بتكون out-of-line وبتضيف checks إضافية (magic number، owner tracking).

---

#### `do_raw_spin_lock`

```c
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
    __acquire(lock);            /* sparse annotation */
    arch_spin_lock(&lock->raw_lock);
    mmiowb_spin_lock();         /* MMIO write barrier tracking */
}
```

**بتستدعي** `arch_spin_lock` على الـ `arch_spinlock_t` الجوفي. الـ `mmiowb_spin_lock()` موجودة لدعم الـ architectures اللي فيها MMIO write buffering (زي IA64 قديمًا).

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.

**Return:** void.

**Key details:**
- الـ `__acquires(lock)` هو annotation للـ sparse tool فقط، مش كود فعلي.
- في debug build بتكون extern وبتضيف فحص الـ magic و recursion detection.
- **لا تستدعيها مباشرة** — الـ API العام هو `raw_spin_lock`.

---

#### `do_raw_spin_trylock`

```c
static inline int do_raw_spin_trylock(raw_spinlock_t *lock)
{
    int ret = arch_spin_trylock(&(lock)->raw_lock);

    if (ret)
        mmiowb_spin_lock();

    return ret;
}
```

**بتحاول** تاخد الـ lock بدون انتظار. لو نجحت بتستدعي `mmiowb_spin_lock()`.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.

**Return:** `int` — 1 لو اتاخد الـ lock، 0 لو لأ.

**Key details:**
- non-blocking تمامًا.
- الـ `arch_spin_trylock` هي atomic compare-and-swap على مستوى الـ hardware.

---

#### `do_raw_spin_unlock`

```c
static inline void do_raw_spin_unlock(raw_spinlock_t *lock) __releases(lock)
{
    mmiowb_spin_unlock();
    arch_spin_unlock(&lock->raw_lock);
    __release(lock);
}
```

**بتعمل** الـ MMIO barrier أولًا، ثم تحرر الـ lock عبر `arch_spin_unlock`.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.

**Return:** void.

**Key details:**
- الترتيب مهم: الـ `mmiowb` لازم يجي **قبل** الـ unlock عشان MMIO writes تتصل قبل ما الـ lock يتحرر.

---

### المجموعة الثالثة: Raw Spinlock Acquire API

الـ SMP implementation موجودة في `spinlock_api_smp.h`. كل function في هذه المجموعة بتعمل:
1. disable preemption (و/أو IRQs/BH).
2. تسجيل الـ acquire في الـ lockdep.
3. محاولة الـ trylock أولًا (`LOCK_CONTENDED`)، ثم الـ slow path.

---

#### `__raw_spin_lock` / `_raw_spin_lock`

```c
static inline void __raw_spin_lock(raw_spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

**بتعطل الـ preemption** ثم بتاخد الـ lock. الـ `LOCK_CONTENDED` macro بتحاول الـ trylock الأول، ولو فشلت بتدخل الـ spinning loop.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.

**Return:** void.

**Key details:**
- `preempt_disable()` لازمة حتى على UP kernels عشان نضمن atomicity مع الـ preemption.
- `spin_acquire` هي lockdep annotation — NOP في non-debug builds.
- الـ `_RET_IP_` بيحفظ return address الـ caller للـ lockdep tracing.
- **Caller context:** process context أو interrupt context (مش يهم لأن preemption disabled).

**Pseudocode flow:**
```
raw_spin_lock(lock):
    preempt_disable()
    lockdep: spin_acquire(lock)
    if !trylock(lock):
        spin until arch_spin_lock succeeds
    // critical section
```

---

#### `__raw_spin_lock_irq` / `_raw_spin_lock_irq`

```c
static inline void __raw_spin_lock_irq(raw_spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    local_irq_disable();
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

**نفس** `__raw_spin_lock` لكن بتعطل الـ IRQs أيضًا. تُستخدم لما الـ lock بتُشارك بين process context و interrupt handlers.

**Key details:**
- الترتيب: `local_irq_disable()` **قبل** `preempt_disable()` عشان لو جه interrupt بين الاتنين ما يـ deadlock-ش.
- **خطر:** لو استخدمتها في interrupt handler ونسيت تـ `unlock_irq`، الـ IRQs هتظل disabled.
- **استخدم** `irqsave` لو مش متأكد إن IRQs كانوا enabled قبلها.

---

#### `__raw_spin_lock_irqsave`

```c
static inline unsigned long __raw_spin_lock_irqsave(raw_spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    unsigned long flags;

    local_irq_save(flags);       /* save EFLAGS/CPSR + disable IRQ */
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
    return flags;
}
```

**بتحفظ** حالة الـ IRQs في `flags` ثم بتعطلها. ضروري لما الكود ممكن يُستدعى من context فيه IRQs enabled أو disabled.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.

**Return:** `unsigned long` — الـ IRQ flags المحفوظة.

**الـ macro الخارجي:**
```c
#define raw_spin_lock_irqsave(lock, flags)          \
    do {                                            \
        typecheck(unsigned long, flags);            \
        flags = _raw_spin_lock_irqsave(lock);       \
    } while (0)
```

**Key details:**
- الـ `typecheck` macro بتضمن إن `flags` من النوع الصح (`unsigned long`).
- **دايمًا** استخدمها مع `raw_spin_unlock_irqrestore` — مش `raw_spin_unlock_irq`.

---

#### `__raw_spin_lock_bh` / `_raw_spin_lock_bh`

```c
static inline void __raw_spin_lock_bh(raw_spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
}
```

**بتعطل** الـ softirqs/BH فقط (مش الـ hardware IRQs). تُستخدم لما الـ lock مشتركة بين process context و softirq handlers.

**Key details:**
- الـ `SOFTIRQ_LOCK_OFFSET` هو قيمة خاصة بتفرق بين "BH disabled for lock" و"BH disabled for other reasons".
- أخف من `lock_irq` لأن hardware IRQs لسه شغالة.
- **Caller context:** process context فقط — لو في hardirq context الـ BH أصلًا disabled.

---

#### `raw_spin_lock_nested` / `raw_spin_lock_nest_lock`

```c
// مع DEBUG_LOCK_ALLOC:
#define raw_spin_lock_nested(lock, subclass) \
    _raw_spin_lock_nested(lock, subclass)

// بدون DEBUG_LOCK_ALLOC:
#define raw_spin_lock_nested(lock, subclass) \
    _raw_spin_lock(((void)(subclass), (lock)))
```

**بتسمح** للـ lockdep بفهم الـ lock ordering الصح لما بتاخد نفس الـ lock type مرتين في نفس الوقت (nested locks). الـ `subclass` بيحدد المستوى (0, 1, 2...).

**Key details:**
- مثال: `inode->i_lock` بتتاخد على مستويين مختلفين في directory traversal.
- بدون `subclass` الـ lockdep سيرفع false positive على "possible deadlock".
- الـ `SINGLE_DEPTH_NESTING` = 1 هو الـ subclass الأشهر.

---

### المجموعة الرابعة: Raw Spinlock Release API

---

#### `__raw_spin_unlock`

```c
static inline void __raw_spin_unlock(raw_spinlock_t *lock)
    __releases(lock)
{
    spin_release(&lock->dep_map, _RET_IP_);
    do_raw_spin_unlock(lock);
    preempt_enable();
}
```

**بتحرر** الـ lock ثم بتعيد تشغيل الـ preemption. الـ `spin_release` lockdep annotation.

**Key details:**
- `preempt_enable()` جاي **بعد** الـ unlock عشان لو في preemption pending تحصل بعد ما الـ lock اتحرر مش قبل.
- لو الـ preempt_enable سبّبت reschedule، الـ thread التاني ممكن يتاخد الـ lock فورًا.

---

#### `__raw_spin_unlock_irq`

```c
static inline void __raw_spin_unlock_irq(raw_spinlock_t *lock)
    __releases(lock)
{
    spin_release(&lock->dep_map, _RET_IP_);
    do_raw_spin_unlock(lock);
    local_irq_enable();
    preempt_enable();
}
```

**بتحرر** الـ lock وبتعيد تشغيل الـ IRQs. الترتيب: unlock → irq_enable → preempt_enable.

**Key details:**
- **مخاطرة:** IRQs بتتـ enable بدون النظر لحالتهم قبل الـ lock. لو كانوا disabled قبل الـ lock_irq دي مشكلة — استخدم `irqrestore`.

---

#### `__raw_spin_unlock_irqrestore`

```c
static inline void __raw_spin_unlock_irqrestore(raw_spinlock_t *lock,
                                                unsigned long flags)
    __releases(lock)
{
    spin_release(&lock->dep_map, _RET_IP_);
    do_raw_spin_unlock(lock);
    local_irq_restore(flags);
    preempt_enable();
}
```

**بتحرر** الـ lock وبترجّع حالة الـ IRQs زي ما كانت قبل `irqsave`.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.
- `flags` — الـ value اللي رجعت من `irqsave`.

**Return:** void.

**Key details:**
- `local_irq_restore(flags)` بتحط الـ EFLAGS/CPSR تاني كما كانت — قد تـ enable وقد لا.
- لازم تكون pair صح مع `raw_spin_lock_irqsave`.

---

#### `__raw_spin_unlock_bh`

```c
static inline void __raw_spin_unlock_bh(raw_spinlock_t *lock)
    __releases(lock)
{
    spin_release(&lock->dep_map, _RET_IP_);
    do_raw_spin_unlock(lock);
    __local_bh_enable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
}
```

**بتحرر** الـ lock وبتعيد تشغيل الـ BH/softirqs. ممكن تـ trigger pending softirqs في نفس اللحظة.

---

### المجموعة الخامسة: Trylock Variants

---

#### `__raw_spin_trylock`

```c
static inline int __raw_spin_trylock(raw_spinlock_t *lock)
    __cond_acquires(true, lock)
{
    preempt_disable();
    if (do_raw_spin_trylock(lock)) {
        spin_acquire(&lock->dep_map, 0, 1, _RET_IP_);
        return 1;
    }
    preempt_enable();
    return 0;
}
```

**بتحاول** تاخد الـ lock مرة واحدة. لو نجحت الـ preemption تظل disabled وترجع 1. لو فشلت ترجع preemption وترجع 0.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.

**Return:** `int` — 1 عند النجاح، 0 عند الفشل.

**Key details:**
- `spin_acquire` بتاخد argument ثالث = 1 (trylock) عشان الـ lockdep يعرف.
- **Caller:** يُستخدم في scenarios زي lock ordering أو عندما الكود عنده fallback.

---

#### `_raw_spin_trylock_irq`

```c
static __always_inline bool _raw_spin_trylock_irq(raw_spinlock_t *lock)
    __cond_acquires(true, lock)
{
    local_irq_disable();
    if (_raw_spin_trylock(lock))
        return true;
    local_irq_enable();
    return false;
}
```

**بتعطل** الـ IRQs ثم بتحاول الـ trylock. لو فشلت بتعيد IRQs وترجع false.

**Key details:**
- لو نجحت: IRQs disabled + lock held. لازم تستخدم `unlock_irq` مش `unlock`.
- pattern شائع في interrupt handlers اللي ممكن تـ race مع process context.

---

#### `_raw_spin_trylock_irqsave`

```c
static __always_inline bool _raw_spin_trylock_irqsave(raw_spinlock_t *lock,
                                                      unsigned long *flags)
    __cond_acquires(true, lock)
{
    local_irq_save(*flags);
    if (_raw_spin_trylock(lock))
        return true;
    local_irq_restore(*flags);
    return false;
}
```

**بتحفظ** IRQ state ثم بتحاول. لو فشلت بترجع الـ state.

**Parameters:**
- `lock` — pointer لـ `raw_spinlock_t`.
- `flags` — pointer لـ `unsigned long` يستقبل الـ saved state.

**Return:** `bool` — true عند النجاح.

---

#### `__raw_spin_trylock_bh`

```c
static inline int __raw_spin_trylock_bh(raw_spinlock_t *lock)
    __cond_acquires(true, lock)
{
    __local_bh_disable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
    if (do_raw_spin_trylock(lock)) {
        spin_acquire(&lock->dep_map, 0, 1, _RET_IP_);
        return 1;
    }
    __local_bh_enable_ip(_RET_IP_, SOFTIRQ_LOCK_OFFSET);
    return 0;
}
```

**نفس** trylock لكن مع BH disable بدل preempt_disable مباشرة.

---

### المجموعة السادسة: High-level Spinlock API (spin_*)

في الـ non-RT kernel هذه الـ functions مجرد wrappers على `raw_spin_*` عبر `lock->rlock`. في الـ RT kernel (`CONFIG_PREEMPT_RT`) بيتم تضمين `spinlock_rt.h` وكل الـ API بتتغير لـ `rt_mutex`.

---

#### `spin_lock`

```c
static __always_inline void spin_lock(spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    raw_spin_lock(&lock->rlock);
}
```

**الـ standard** spinlock acquire. في non-RT هي مجرد `raw_spin_lock`.

**Parameters:**
- `lock` — pointer لـ `spinlock_t`.

**Return:** void.

**Key details:**
- `__no_context_analysis` بتعطل تحليل الـ context checker — الـ function صح في أي context.
- في RT kernel بتتحول لـ `rt_mutex_lock` يعني ممكن تـ sleep.

---

#### `spin_unlock`

```c
static __always_inline void spin_unlock(spinlock_t *lock)
    __releases(lock) __no_context_analysis
{
    raw_spin_unlock(&lock->rlock);
}
```

**بتحرر** الـ spinlock. في non-RT: `raw_spin_unlock` → `do_raw_spin_unlock` → `arch_spin_unlock` → `preempt_enable`.

---

#### `spin_trylock`

```c
static __always_inline int spin_trylock(spinlock_t *lock)
    __cond_acquires(true, lock) __no_context_analysis
{
    return raw_spin_trylock(&lock->rlock);
}
```

**Return:** 1 عند النجاح، 0 عند الفشل.

**Key details:**
- `__cond_acquires(true, lock)` بتقول للـ sparse إن الـ lock بيتاخد conditionally.

---

#### `spin_lock_bh`

```c
static __always_inline void spin_lock_bh(spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    raw_spin_lock_bh(&lock->rlock);
}
```

**Acquire + disable softirqs.** تُستخدم لحماية shared data بين process context و softirq context.

**مثال عملي:** network driver بيعدل TX queue — الـ softirq (NAPI) ممكن يأثر عليه.

---

#### `spin_lock_irq`

```c
static __always_inline void spin_lock_irq(spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    raw_spin_lock_irq(&lock->rlock);
}
```

**Acquire + disable hardware IRQs.** تُستخدم لحماية shared data بين process context و interrupt handlers.

**تحذير:** بتـ assume إن IRQs كانوا enabled قبلها. لو مش متأكد استخدم `irqsave`.

---

#### `spin_lock_irqsave`

```c
#define spin_lock_irqsave(lock, flags)                  \
do {                                                    \
    raw_spin_lock_irqsave(spinlock_check(lock), flags); \
    __release(spinlock_check(lock)); __acquire(lock);   \
} while (0)
```

**أكثر استخدامًا** من `spin_lock_irq`. بتحفظ حالة الـ IRQs في `flags` قبل الـ disable.

**Parameters:**
- `lock` — pointer لـ `spinlock_t`.
- `flags` — `unsigned long` variable (passed by name, macro أخد عنوانه).

**Key details:**
- الـ `__release`/`__acquire` اللعبة الـ sparse annotations: الـ `spinlock_check` بترجع `raw_spinlock_t *` والـ lockdep internal tracking على `spinlock_t *` فالـ macro بيعمل re-annotation.
- **دايمًا** يقابلها `spin_unlock_irqrestore`.

---

#### `spin_unlock_irqrestore`

```c
static __always_inline void spin_unlock_irqrestore(spinlock_t *lock,
                                                   unsigned long flags)
    __releases(lock) __no_context_analysis
{
    raw_spin_unlock_irqrestore(&lock->rlock, flags);
}
```

**Parameters:**
- `lock` — pointer لـ `spinlock_t`.
- `flags` — الـ value اللي اتحفظ في `irqsave`.

**Return:** void.

---

#### `spin_trylock_bh`

```c
static __always_inline int spin_trylock_bh(spinlock_t *lock)
    __cond_acquires(true, lock) __no_context_analysis
{
    return raw_spin_trylock_bh(&lock->rlock);
}
```

**Return:** 1 عند النجاح (lock held + BH disabled)، 0 عند الفشل (BH re-enabled).

---

#### `spin_trylock_irq`

```c
static __always_inline int spin_trylock_irq(spinlock_t *lock)
    __cond_acquires(true, lock) __no_context_analysis
{
    return raw_spin_trylock_irq(&lock->rlock);
}
```

**Return:** 1 عند النجاح (lock held + IRQs disabled)، 0 عند الفشل (IRQs re-enabled).

---

#### `_spin_trylock_irqsave` / `spin_trylock_irqsave`

```c
static __always_inline bool _spin_trylock_irqsave(spinlock_t *lock,
                                                  unsigned long *flags)
    __cond_acquires(true, lock) __no_context_analysis
{
    return raw_spin_trylock_irqsave(spinlock_check(lock), *flags);
}

#define spin_trylock_irqsave(lock, flags) _spin_trylock_irqsave(lock, &(flags))
```

**الـ macro** بيمرر `&flags` تلقائيًا. لو نجح: lock held + IRQ state محفوظ في `flags`. لو فشل: IRQ state متمش تغييره.

---

### المجموعة السابعة: Status Queries

---

#### `spin_is_locked`

```c
static __always_inline int spin_is_locked(spinlock_t *lock)
{
    return raw_spin_is_locked(&lock->rlock);
}

// والـ raw version:
#define raw_spin_is_locked(lock) arch_spin_is_locked(&(lock)->raw_lock)
```

**بتفحص** حالة الـ lock. **لا تضمن** أي memory ordering. للـ debugging فقط.

**Return:** 1 لو locked، 0 لو unlocked.

**Key details:**
- على `CONFIG_SMP=n` + `DEBUG_SPINLOCK=n` دايمًا بترجع 0.
- **لا تستخدمها** في الـ logic الأساسي — فقط للـ assertions والـ debugging.

---

#### `spin_is_contended`

```c
static __always_inline int spin_is_contended(spinlock_t *lock)
{
    return raw_spin_is_contended(&lock->rlock);
}

#ifdef arch_spin_is_contended
#define raw_spin_is_contended(lock) arch_spin_is_contended(&(lock)->raw_lock)
#else
#define raw_spin_is_contended(lock) (((void)(lock), 0))
#endif
```

**بتفحص** لو في CPU تاني waiting على نفس الـ lock. لو الـ arch مش بتدعم الـ contention detection بترجع 0 دايمًا.

**Return:** non-zero لو في waiters، 0 لو لأ.

---

#### `spin_needbreak`

```c
static inline int spin_needbreak(spinlock_t *lock)
{
    if (!preempt_model_preemptible())
        return 0;

    return spin_is_contended(lock);
}
```

**بتفحص** هل الـ critical section الحالي محتاج ينقطع عشان thread تاني waiting. الغرض: تحسين الـ fairness وتقليل الـ latency.

**Return:** non-zero لو لازم تـ break الـ critical section (unlock + relock).

**Caller context:** loop طويلة داخل critical section.

**مثال عملي:**
```c
spin_lock(&lock);
while (work_to_do()) {
    do_work_item();
    if (spin_needbreak(&lock)) {
        spin_unlock(&lock);
        cond_resched();
        spin_lock(&lock);
    }
}
spin_unlock(&lock);
```

---

#### `rwlock_needbreak`

```c
static inline int rwlock_needbreak(rwlock_t *lock)
{
    if (!preempt_model_preemptible())
        return 0;

    return rwlock_is_contended(lock);
}
```

**نفس** `spin_needbreak` لكن للـ `rwlock_t`. تُستخدم في loops داخل read/write critical sections.

---

### المجموعة الثامنة: Memory Barrier

---

#### `smp_mb__after_spinlock`

```c
#ifndef smp_mb__after_spinlock
#define smp_mb__after_spinlock() kcsan_mb()
#endif
```

**بتضمن** full memory barrier بين الـ lock acquire السابق ومنه وبين الـ memory accesses اللاحقة.

**الهدف:** ترقية الـ spinlock لـ **RCsc** (Release Consistency sequentially consistent) بدلًا من RCtso أو RCpc.

**Key details:**
- على معظم الـ architectures (x86, ARM64 مع LSE) الـ `spin_lock` نفسه بيعمل full barrier — فالـ macro بيبقى NOP.
- ضروري على architectures زي POWER اللي الـ ACQUIRE barrier فيها weaker.
- مستخدمة في `__schedule()` و`try_to_wake_up()` لضمان صحة الـ task state visibility.
- `kcsan_mb()` هو الـ default — بيخبر الـ KCSAN (Kernel Concurrency Sanitizer) بوجود barrier.

**المثال من الكود:**
```c
// CPU0: task scheduling
spin_lock(S);
smp_mb__after_spinlock();  // ensures CPU1 sees all stores before this lock
r0 = READ_ONCE(Y);
spin_unlock(S);
```

---

### المجموعة التاسعة: Atomic + Spinlock Integration

---

#### `atomic_dec_and_lock`

```c
extern int atomic_dec_and_lock(atomic_t *atomic, spinlock_t *lock)
    __cond_acquires(true, lock);
```

**بتعمل** decrement على `atomic`. لو النتيجة 0 بتاخد `lock` وترجع 1. لو مش 0 ترجع 0 بدون اللوك.

**Parameters:**
- `atomic` — pointer لـ `atomic_t` المطلوب تـ decrement-ه.
- `lock` — pointer لـ `spinlock_t` اللي هيتاخد لو وصلنا صفر.

**Return:** `int` — 1 (مع الـ lock held) لو الـ count وصل صفر، 0 لو لأ.

**Key details:**
- الـ implementation في `lib/dec_and_lock.c`.
- مثال استخدام: reference counting على objects. لما الـ count بيوصل صفر الـ lock بيتاخد عشان destructor آمن.
- الـ race-free: الـ check والـ lock بيحصلوا atomically.

**مثال:**
```c
// reference counted object cleanup
if (atomic_dec_and_lock(&obj->refcount, &obj->lock)) {
    // we hold obj->lock, refcount == 0
    list_del(&obj->list);
    spin_unlock(&obj->lock);
    kfree(obj);
}
```

---

#### `atomic_dec_and_lock_irqsave`

```c
extern int _atomic_dec_and_lock_irqsave(atomic_t *atomic, spinlock_t *lock,
                                        unsigned long *flags)
    __cond_acquires(true, lock);

#define atomic_dec_and_lock_irqsave(atomic, lock, flags) \
    _atomic_dec_and_lock_irqsave(atomic, lock, &(flags))
```

**نفس** `atomic_dec_and_lock` لكن مع IRQ save. لو الـ lock هيتاخد الـ IRQs بيتـ disable وحالتهم بتتحفظ في `flags`.

**Return:** 1 مع (lock held + IRQs disabled)، 0 بدون.

---

#### `atomic_dec_and_raw_lock` / `atomic_dec_and_raw_lock_irqsave`

```c
extern int atomic_dec_and_raw_lock(atomic_t *atomic, raw_spinlock_t *lock)
    __cond_acquires(true, lock);

extern int _atomic_dec_and_raw_lock_irqsave(atomic_t *atomic,
                                             raw_spinlock_t *lock,
                                             unsigned long *flags)
    __cond_acquires(true, lock);

#define atomic_dec_and_raw_lock_irqsave(atomic, lock, flags) \
    _atomic_dec_and_raw_lock_irqsave(atomic, lock, &(flags))
```

**نفس** المجموعة السابقة لكن على `raw_spinlock_t` بدل `spinlock_t`. تُستخدم في RT-safe code اللي محتاج spinlock حقيقي مش rt_mutex.

---

### المجموعة العاشرة: Bucket Spinlocks

---

#### `alloc_bucket_spinlocks` / `__alloc_bucket_spinlocks`

```c
int __alloc_bucket_spinlocks(spinlock_t **locks, unsigned int *lock_mask,
                              size_t max_size, unsigned int cpu_mult,
                              gfp_t gfp, const char *name,
                              struct lock_class_key *key);

#define alloc_bucket_spinlocks(locks, lock_mask, max_size, cpu_mult, gfp) \
    ({                                                                      \
        static struct lock_class_key key;                                   \
        int ret;                                                            \
        ret = __alloc_bucket_spinlocks(locks, lock_mask, max_size,          \
                                       cpu_mult, gfp, #locks, &key);       \
        ret;                                                                \
    })
```

**بتخصص** array من الـ spinlocks للـ hash-based locking. عدد الـ locks = `min(max_size, num_cpus * cpu_mult)` — rounded to power of 2.

**Parameters:**
- `locks` — output: pointer لـ pointer لأول lock في الـ array.
- `lock_mask` — output: الـ mask للـ index (power of 2 - 1).
- `max_size` — الحد الأقصى لعدد الـ locks.
- `cpu_mult` — معامل الضرب في عدد الـ CPUs.
- `gfp` — الـ allocation flags (`GFP_KERNEL` عادةً).
- `name` — للـ lockdep.
- `key` — الـ lockdep class key.

**Return:** 0 عند النجاح، negative error code عند الفشل.

**مثال استخدام:**
```c
spinlock_t *hash_locks;
unsigned int hash_mask;

// يخصص min(1024, num_cpus*4) locks
alloc_bucket_spinlocks(&hash_locks, &hash_mask, 1024, 4, GFP_KERNEL);

// استخدام:
spinlock_t *lock = &hash_locks[hash & hash_mask];
spin_lock(lock);
// ...
spin_unlock(lock);
```

**Key details:**
- الهدف: تقليل الـ lock contention في hash tables عن طريق stripe locking.
- عدد الـ locks متناسب مع عدد الـ CPUs لتجنب false sharing.
- كل الـ locks في الـ array ليها نفس الـ lockdep class (`key` واحدة).

---

#### `free_bucket_spinlocks`

```c
void free_bucket_spinlocks(spinlock_t *locks);
```

**بتحرر** الـ array المخصص بـ `alloc_bucket_spinlocks`.

**Parameters:**
- `locks` — pointer لأول lock في الـ array.

**Return:** void.

---

### المجموعة الحادية عشرة: RAII Lock Guards (cleanup.h integration)

الـ kernel بدءًا من v6.x بيدعم RAII-style locking عبر `DEFINE_LOCK_GUARD_1` و`DEFINE_LOCK_GUARD_1_COND`. الـ guards بتـ unlock تلقائيًا عند خروج الـ scope عبر الـ `__attribute__((cleanup))`.

---

#### النمط العام للاستخدام

```c
// بدلاً من:
spin_lock(&my_lock);
if (error) {
    spin_unlock(&my_lock);
    return -EINVAL;
}
// ...
spin_unlock(&my_lock);

// تستخدم:
guard(spinlock)(&my_lock);
if (error)
    return -EINVAL;  // auto-unlock هنا
// auto-unlock عند نهاية الـ scope
```

---

#### الـ Guards المتاحة

| Guard Name | Lock Type | Lock Op | Unlock Op |
|---|---|---|---|
| `raw_spinlock` | `raw_spinlock_t` | `raw_spin_lock` | `raw_spin_unlock` |
| `raw_spinlock_try` | `raw_spinlock_t` | `raw_spin_trylock` | `raw_spin_unlock` |
| `raw_spinlock_nested` | `raw_spinlock_t` | `raw_spin_lock_nested(SINGLE_DEPTH_NESTING)` | `raw_spin_unlock` |
| `raw_spinlock_irq` | `raw_spinlock_t` | `raw_spin_lock_irq` | `raw_spin_unlock_irq` |
| `raw_spinlock_irq_try` | `raw_spinlock_t` | `raw_spin_trylock_irq` | `raw_spin_unlock_irq` |
| `raw_spinlock_bh` | `raw_spinlock_t` | `raw_spin_lock_bh` | `raw_spin_unlock_bh` |
| `raw_spinlock_bh_try` | `raw_spinlock_t` | `raw_spin_trylock_bh` | `raw_spin_unlock_bh` |
| `raw_spinlock_irqsave` | `raw_spinlock_t` | `raw_spin_lock_irqsave` | `raw_spin_unlock_irqrestore` |
| `raw_spinlock_irqsave_try` | `raw_spinlock_t` | `raw_spin_trylock_irqsave` | `raw_spin_unlock_irqrestore` |
| `spinlock` | `spinlock_t` | `spin_lock` | `spin_unlock` |
| `spinlock_try` | `spinlock_t` | `spin_trylock` | `spin_unlock` |
| `spinlock_irq` | `spinlock_t` | `spin_lock_irq` | `spin_unlock_irq` |
| `spinlock_irq_try` | `spinlock_t` | `spin_trylock_irq` | `spin_unlock_irq` |
| `spinlock_bh` | `spinlock_t` | `spin_lock_bh` | `spin_unlock_bh` |
| `spinlock_bh_try` | `spinlock_t` | `spin_trylock_bh` | `spin_unlock_bh` |
| `spinlock_irqsave` | `spinlock_t` | `spin_lock_irqsave` | `spin_unlock_irqrestore` |
| `spinlock_irqsave_try` | `spinlock_t` | `spin_trylock_irqsave` | `spin_unlock_irqrestore` |
| `read_lock` | `rwlock_t` | `read_lock` | `read_unlock` |
| `read_lock_irq` | `rwlock_t` | `read_lock_irq` | `read_unlock_irq` |
| `read_lock_irqsave` | `rwlock_t` | `read_lock_irqsave` | `read_unlock_irqrestore` |
| `write_lock` | `rwlock_t` | `write_lock` | `write_unlock` |
| `write_lock_irq` | `rwlock_t` | `write_lock_irq` | `write_unlock_irq` |
| `write_lock_irqsave` | `rwlock_t` | `write_lock_irqsave` | `write_unlock_irqrestore` |

---

#### الـ `_try` variants والـ conditional locking

الـ guards اللي اسمها `_try` بتستخدم `DEFINE_LOCK_GUARD_1_COND` — لو الـ trylock فشل الـ guard بيرجع NULL وبتقدر تتحقق:

```c
// scoped_guard مع trylock:
scoped_guard(spinlock_try, &my_lock) {
    // لو الـ trylock نجح
    do_work();
}
// لو فشل الـ block ما اتنفذش
```

---

#### `DECLARE_LOCK_GUARD_1_ATTRS` و Sparse Annotations

```c
DECLARE_LOCK_GUARD_1_ATTRS(spinlock, __acquires(_T), __releases(*(spinlock_t **)_T))
```

الـ `DECLARE_LOCK_GUARD_1_ATTRS` بتضيف الـ sparse/smatch annotations على الـ guard constructor عشان الـ static analysis tools تفهم إن الـ lock اتاخد واتحرر صح حتى مع الـ RAII pattern.

---

### ملخص الـ Call Stack الكامل

```
spin_lock(lock)
    └── raw_spin_lock(&lock->rlock)              [non-RT]
        └── _raw_spin_lock(lock)
            └── __raw_spin_lock(lock)
                ├── preempt_disable()
                ├── spin_acquire()               [lockdep]
                └── LOCK_CONTENDED()
                    ├── do_raw_spin_trylock(lock) [fast path]
                    │   └── arch_spin_trylock()  [arch asm]
                    └── do_raw_spin_lock(lock)   [slow path]
                        └── arch_spin_lock()     [arch asm, spins]

spin_unlock(lock)
    └── raw_spin_unlock(&lock->rlock)
        └── _raw_spin_unlock(lock)
            └── __raw_spin_unlock(lock)
                ├── spin_release()               [lockdep]
                ├── do_raw_spin_unlock(lock)
                │   ├── mmiowb_spin_unlock()
                │   └── arch_spin_unlock()       [arch asm]
                └── preempt_enable()
```

---

### مقارنة الـ Lock Variants

| الـ Variant | Preemption | Hardware IRQs | Softirqs/BH | متى تستخدمه |
|---|---|---|---|---|
| `spin_lock` | Disabled | Unchanged | Unchanged | process context فقط — مفيش interrupt handlers |
| `spin_lock_bh` | Disabled | Unchanged | Disabled | مشترك مع softirq handlers |
| `spin_lock_irq` | Disabled | Disabled | Disabled | مشترك مع interrupt handlers (IRQs مضمون enabled) |
| `spin_lock_irqsave` | Disabled | Disabled (saved) | Disabled | مشترك مع interrupt handlers (IRQ state غير معروف) |
| `spin_trylock` | Cond.Disabled | Unchanged | Unchanged | non-blocking acquire |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs entries

**الـ lockdep** بيكتب معلوماته في debugfs. بعد mount الـ debugfs:

```bash
mount -t debugfs none /sys/kernel/debug
```

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/locking/lockdep` | قائمة كل الـ lock classes المسجّلة |
| `/sys/kernel/debug/locking/lockdep_chains` | dependency chains بين الـ locks |
| `/sys/kernel/debug/locking/lockdep_stats` | إحصائيات الـ lockdep (عدد الـ classes، الـ chains، إلخ) |
| `/sys/kernel/debug/locking/lock_stat` | إحصائيات الـ contention لكل lock class |

```bash
# قراءة إحصائيات الـ spinlock contention
cat /sys/kernel/debug/locking/lock_stat

# قراءة الـ dependency graph
cat /sys/kernel/debug/locking/lockdep | head -100
```

مثال output من `lock_stat`:

```
class name          con-bounces    contentions   waittime-min   waittime-max waittime-total
-------------------------------------------------------------------------------------------------------
&sbdev->lock:                  2             5           0.15           4.20           8.10
&rq->lock:                    44           892           0.10          12.50        4821.33
```

**الـ con-bounces** = عدد المرات اللي اتحول فيها الـ lock من CPU لـ CPU أثناء الـ contention — ارتفاعها بيدل على **cache bouncing** شديد.

---

#### 2. sysfs entries

| المسار | الوصف |
|--------|-------|
| `/proc/sys/kernel/panic_on_oops` | لو بتعمل debug لـ spinlock في interrupt context ابقى حاطط `1` |
| `/proc/sys/kernel/softlockup_thresh` | المهلة (بالثواني) قبل ما الكيرنل يعلن soft lockup |
| `/proc/sys/kernel/hardlockup_thresh` | المهلة قبل hard lockup detector |
| `/proc/lockdep` | نفس محتوى debugfs لكن عبر procfs |
| `/proc/lockdep_stats` | إحصائيات سريعة |

```bash
# زيادة timeout الـ soft lockup لتجنب false positives أثناء الـ debug
echo 30 > /proc/sys/kernel/softlockup_thresh

# تفعيل panic عند lockup لأخذ kdump
echo 1 > /proc/sys/kernel/softlockup_panic
echo 1 > /proc/sys/kernel/hardlockup_panic
```

---

#### 3. ftrace — tracepoints وأحداث الـ spinlock

**الـ ftrace** بيوفر function tracer وأحداث متخصصة للـ locking:

```bash
# تفعيل أحداث الـ lock
echo 1 > /sys/kernel/debug/tracing/events/lock/lock_acquire/enable
echo 1 > /sys/kernel/debug/tracing/events/lock/lock_release/enable
echo 1 > /sys/kernel/debug/tracing/events/lock/lock_contended/enable
echo 1 > /sys/kernel/debug/tracing/events/lock/lock_acquired/enable

# تفعيل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# قراءة النتيجة
cat /sys/kernel/debug/tracing/trace | head -50

# إيقاف
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

مثال output:

```
# TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
# | | |        |   ||||       |         |
     kworker/0:1-25    [000] ....  1234.567890: lock_acquire: &(&rq->lock)->rlock, spin
     kworker/0:1-25    [000] ....  1234.567891: lock_contended: &(&rq->lock)->rlock
     kworker/0:1-25    [000] ....  1234.568100: lock_acquired: &(&rq->lock)->rlock
```

الفرق بين `lock_contended` و `lock_acquired` = **وقت الانتظار** = هو مصدر الـ latency.

لتتبع function معينة بتستخدم الـ `spin_lock`:

```bash
# تتبع كل الـ calls لـ _raw_spin_lock
echo _raw_spin_lock > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk و dynamic debug

تفعيل الـ **dynamic debug** لرسائل الـ spinlock subsystem:

```bash
# تفعيل debug messages في spinlock
echo 'file spinlock.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file spinlock_debug.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل رسائل الـ locking
echo 'module lockdep +p' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الرسائل
dmesg -w | grep -E 'spin|lock|deadlock'
```

لإضافة `pr_debug` مؤقت في كود الـ driver اللي بيستخدم spinlock:

```c
#define pr_fmt(fmt) "mydrv: " fmt
#include <linux/printk.h>

spin_lock(&my_lock);
pr_debug("acquired my_lock on CPU %d, pid %d\n",
         smp_processor_id(), current->pid);
/* critical section */
spin_unlock(&my_lock);
pr_debug("released my_lock\n");
```

مستويات الـ printk المناسبة:

| الحالة | المستوى |
|--------|---------|
| deadlock مشتبه فيه | `KERN_EMERG` |
| lock held too long | `KERN_WARNING` |
| contention عالية | `KERN_INFO` |
| tracing عادي | `pr_debug` |

---

#### 5. Kernel Config options للـ debugging

| الـ Config | الوصف |
|------------|-------|
| `CONFIG_DEBUG_SPINLOCK` | يفعّل magic number check وتسجيل الـ owner — أهم option |
| `CONFIG_DEBUG_LOCK_ALLOC` | يتكامل مع lockdep لتتبع alloc/free cycles |
| `CONFIG_LOCKDEP` | الـ lock dependency validator — يكشف deadlocks وordering violations |
| `CONFIG_LOCK_STAT` | يجمع إحصائيات الـ contention لكل lock |
| `CONFIG_DEBUG_LOCKDEP` | debug الـ lockdep نفسه |
| `CONFIG_LOCKUP_DETECTOR` | يكشف soft/hard lockups |
| `CONFIG_HARDLOCKUP_DETECTOR` | يستخدم NMI لكشف hard lockup على SMP |
| `CONFIG_SOFTLOCKUP_DETECTOR` | يكشف الـ kernel stuck في loop |
| `CONFIG_PROVE_LOCKING` | يثبت صحة ordering الـ locks في runtime |
| `CONFIG_SPARSE_RCU_POINTER` | يساعد في تمييز RCU-protected pointers |
| `CONFIG_KCSAN` | Kernel Concurrency Sanitizer — يكشف data races |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكشف لو حد استدعى sleep وهو شايل spinlock |

```bash
# فحص الـ config الحالي
zcat /proc/config.gz | grep -E 'DEBUG_SPIN|LOCKDEP|LOCK_STAT|LOCKUP|PROVE_LOCK'
```

---

#### 6. أدوات متخصصة

**الـ lockdep** هو الأداة الأساسية:

```bash
# فحص لو الـ lockdep شاف مشاكل
dmesg | grep -A 30 'possible circular locking'
dmesg | grep -A 20 'WARNING.*lock'
dmesg | grep 'BUG: spinlock'
```

**الـ perf** لقياس الـ lock contention:

```bash
# تسجيل أحداث الـ lock لمدة 5 ثواني
perf lock record -a -- sleep 5

# تحليل النتائج
perf lock report

# تقرير مفصّل بالـ wait times
perf lock report --details
```

مثال output من `perf lock report`:

```
Name   acquired  contended   total wait (ns)   max wait (ns)   avg wait (ns)
rq_lock     8923        234           4532100           21500            19367
zone->lock   441         12             89200            7800             7433
```

**الـ BPF/bpftrace** لـ tracing متقدم:

```bash
# تتبع كل spin_lock مع الـ stack trace لما الانتظار يتعدى 10 microseconds
bpftrace -e '
kprobe:_raw_spin_lock { @start[tid] = nsecs; }
kretprobe:_raw_spin_lock
/@start[tid]/
{
  $delta = nsecs - @start[tid];
  if ($delta > 10000) {
    printf("spin_lock held >10us: %s delta=%lld\n", comm, $delta);
    print(kstack);
  }
  delete(@start[tid]);
}'
```

---

#### 7. رسائل الـ error الشائعة

| رسالة الـ kernel | المعنى | الحل |
|-----------------|--------|------|
| `BUG: spinlock already unlocked` | double unlock — تحرير lock مرتين | فحص كل مسارات الكود، خصوصاً عند error handling |
| `BUG: spinlock wrong CPU` | unlock على CPU مختلف عن الـ lock | الـ spinlock لازم يُحرر على نفس الـ CPU |
| `BUG: spinlock recursion` | نفس الـ thread حاول يأخذ نفس الـ lock | استخدم `spin_trylock` أو إعادة تصميم الـ locking |
| `WARNING: possible circular locking dependency` | lockdep اكتشف deadlock محتمل | راجع order الـ locks — دايماً خذ الـ locks بنفس الترتيب |
| `WARNING: possible irq lock inversion` | lock يتأخذ في ISR وفي process context بدون `_irqsave` | استخدم `spin_lock_irqsave` في process context |
| `SOFTLOCKUP: stuck for Xs on CPU Y` | kernel loop بدون schedule لفترة طويلة | راجع critical sections الطويلة، أضف `cond_resched()` |
| `HARDLOCKUP: NMI watchdog` | CPU مش بيرد لـ NMI، على الأرجح في spinloop لا نهائي | deadlock حقيقي — فحص الـ vmcore |
| `BUG: sleeping function called from invalid context` | استدعاء `sleep`/`kmalloc(GFP_KERNEL)` وأنت شايل spinlock | استخدم `GFP_ATOMIC` أو اتخلى من الـ lock قبل الـ sleep |
| `BUG: spinlock bad magic` | corruption في الـ lock structure | buffer overflow أو use-after-free بالقرب من الـ lock |
| `spin_lock called when !irqs_disabled()` | استخدام `raw_spin_lock_irq` لكن الـ IRQs مش معطّلة | مشكلة في الـ context — راجع الـ call site |

---

#### 8. Strategic points لـ dump_stack() وـ WARN_ON()

```c
/* تحقق إن الـ lock مش ممسوك وأنت بتدخل critical section حساسة */
WARN_ON(spin_is_locked(&global_lock) && !irqs_disabled());

/* تحقق من صحة الـ lock قبل الاستخدام في debug builds */
#ifdef CONFIG_DEBUG_SPINLOCK
static void verify_lock_state(spinlock_t *lock)
{
    /* لو الـ magic اتعمله corruption — بنطلق stack trace */
    WARN_ON(lock->rlock.magic != SPINLOCK_MAGIC);
}
#endif

/* نقطة استراتيجية: عند unlock في unexpected path */
static void release_device_lock(struct mydev *dev)
{
    if (WARN_ON(!spin_is_locked(&dev->lock))) {
        /* بنطبع stack لمعرفة مين عمل double-unlock */
        dump_stack();
        return;
    }
    spin_unlock(&dev->lock);
}

/* تحقق إن IRQs مش enabled وأنت في IRQ context + spinlock */
static irqreturn_t my_irq_handler(int irq, void *dev)
{
    struct mydev *d = dev;
    WARN_ON_ONCE(!irqs_disabled());
    spin_lock(&d->lock);
    /* ... */
    spin_unlock(&d->lock);
    return IRQ_HANDLED;
}

/* كشف long hold time — بيفيد في profiling */
static void critical_path(spinlock_t *lock)
{
    unsigned long flags;
    ktime_t start;

    spin_lock_irqsave(lock, flags);
    start = ktime_get();

    /* ... critical section ... */

    if (ktime_us_delta(ktime_get(), start) > 100) {
        /* Lock held >100us — مش طبيعي */
        WARN_ONCE(1, "spinlock held for >100us, possible issue\n");
        dump_stack();
    }
    spin_unlock_irqrestore(lock, flags);
}
```

أهم النقاط الاستراتيجية:

| النقطة | السبب |
|--------|-------|
| قبل `spin_lock` في context غير واضح | التحقق من إن الـ preemption والـ IRQ في الحالة الصح |
| بعد unlock غير متوقع | كشف الـ double-unlock مبكراً |
| في error paths | كثير من الـ bugs بتحصل في الـ error handling |
| عند الكشف عن contention عالية | بيساعد في تحديد الـ hot lock |

---

### Hardware Level

---

#### 1. التحقق إن الـ hardware state بيطابق الـ kernel state

الـ spinlock في نهايته يعتمد على **atomic operations** في الـ hardware (LL/SC أو CMPXCHG). لو فيه مشكلة في الـ cache coherency أو الـ memory subsystem، الـ spinlock ممكن يتصرف غلط:

```bash
# فحص الـ CPU topology والـ cache hierarchy
cat /sys/devices/system/cpu/cpu0/cache/index*/type
cat /sys/devices/system/cpu/cpu0/cache/index*/shared_cpu_list

# فحص لو الـ NUMA topology بتسبب contention
numactl --hardware
numastat -c

# مقارنة الـ CPU flags — لازم يكون فيه cx8, cx16, lock prefix support
grep -m1 flags /proc/cpuinfo | tr ' ' '\n' | grep -E 'cx|lock|sse'
```

في ARM:
```bash
# التحقق من دعم الـ LDREX/STREX أو LSE atomics
grep -m1 'Features' /proc/cpuinfo | tr ' ' '\n' | grep -E 'atomics|lse'
```

---

#### 2. Register Dump Techniques

في حالة الـ hard lockup — بعد الـ crash — لازم نفحص حالة الـ lock في الذاكرة.

باستخدام `/dev/mem` أو `devmem2` (على embedded systems):

```bash
# تثبيت devmem2
apt-get install devmem2

# معرفة عنوان الـ lock في الذاكرة (من kernel symbol table)
grep my_spinlock /proc/kallsyms
# مثال output: ffffffff82a3b4c0 D my_spinlock

# قراءة قيمة الـ lock (4 bytes)
devmem2 0xffffffff82a3b4c0 w
```

من داخل `crash` utility بعد أخذ vmcore:

```bash
crash vmlinux /var/crash/vmcore

# في الـ crash prompt
crash> p my_spinlock
crash> struct raw_spinlock my_spinlock.rlock
crash> rd 0xffffffff82a3b4c0 4
```

مثال `raw_spinlock` في x86 (ticket spinlock legacy):

```
raw_lock value: 0x00010001  -> locked (next != owner)
raw_lock value: 0x00000000  -> unlocked
```

على x86 الـ qspinlock الحديث:

```
qspinlock value: 0x00000001 -> locked by one CPU, no waiters
qspinlock value: 0x00000103 -> locked + waiters present
qspinlock value: 0x00000000 -> unlocked
```

باستخدام `gdb` على vmcore:

```bash
gdb vmlinux vmcore
(gdb) p my_global_spinlock
(gdb) p my_global_spinlock.rlock.raw_lock
(gdb) x/4bx &my_global_spinlock
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ spinlock نفسه مش بيظهر على logic analyzer مباشرةً، لكن ممكن تقيس **تأثيره** على الـ hardware:

**قياس الـ interrupt latency:**
- حط GPIO toggle قبل وبعد الـ spinlock
- قيس التأخير على oscilloscope

```c
/* في kernel module للقياس */
#include <linux/gpio.h>

spin_lock_irqsave(&my_lock, flags);
gpio_set_value(DEBUG_GPIO, 1);   /* rising edge = lock acquired */

/* critical section */

gpio_set_value(DEBUG_GPIO, 0);   /* falling edge = lock released */
spin_unlock_irqrestore(&my_lock, flags);
```

**قياس الـ cache coherency traffic:**
- استخدم Performance Monitoring Units (PMU)
- `perf stat -e cache-misses,cache-references,LLC-load-misses ./workload`
- ارتفاع `LLC-load-misses` مع ارتفاع الـ contention يدل على **false sharing**

**JTAG debugging:**
- وقّف الـ CPU على breakpoint داخل `arch_spin_lock`
- افحص قيمة الـ `raw_lock` مباشرة في register

---

#### 4. Hardware Issues الشائعة وـ kernel log patterns

| المشكلة | الـ kernel log pattern | التشخيص |
|---------|----------------------|---------|
| **False sharing** في الـ cache line | `perf c2c` يظهر high HITM | ضع الـ lock في cache line منفصل بـ `____cacheline_aligned` |
| **NUMA remote access** — lock على node بعيد | `numastat` يظهر remote memory access عالي | استخدم per-CPU locks أو per-NUMA locks |
| **Memory ordering bug** في architecture غير x86 | random data corruption بعد الـ lock | تأكد من استخدام `smp_mb__after_spinlock()` |
| **CPU throttling** — thermal throttling | `SOFTLOCKUP` مع ارتفاع temperature | `sensors` / `turbostat` يأكد الـ throttling |
| **Interrupt storm** — lock held طول إعدام الـ ISR | `HARDLOCKUP` على CPU بعينه | `cat /proc/interrupts` لفحص distribution |
| **Errata في الـ CPU** في atomic ops | sporadic lockups بدون سبب واضح | فحص `errata` في documentation الـ CPU |

---

#### 5. Device Tree Debugging

الـ spinlock مش مرتبط بالـ DT بشكل مباشر، لكن الـ devices المتحكّم فيها (shared memory، interrupt controllers) ممكن تؤثر:

```bash
# فحص الـ interrupt controller configuration في DT
dtc -I fs /sys/firmware/devicetree/base | grep -A 5 'interrupt-controller'

# فحص الـ shared memory regions
dtc -I fs /sys/firmware/devicetree/base | grep -A 10 'shared-memory\|shmem'

# التحقق من الـ interrupt affinity
cat /proc/irq/*/smp_affinity_list

# لو في مشكلة في spinlock داخل interrupt handler بسبب DT misconfiguration
# افحص إن الـ interrupt مش shared بين devices بشكل غلط
cat /proc/interrupts | awk '{print $1, $NF}' | sort
```

للأجهزة المعتمدة على **hwspinlock** (multicore SoCs):

```bash
# فحص hwspinlock DT node
dtc -I fs /sys/firmware/devicetree/base | grep -B2 -A 10 'hwspinlock'

# فحص driver binding
ls -la /sys/bus/platform/drivers/hwspinlock*/
cat /sys/bus/platform/devices/*/driver_override 2>/dev/null
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ والتشغيل

**أولاً: تشخيص سريع**

```bash
# ========= فحص شامل سريع للـ locking issues =========

# 1. هل فيه lockdep warnings؟
dmesg | grep -c 'possible circular\|WARNING.*lock\|BUG.*spin'

# 2. هل فيه lockup جارٍ؟
dmesg | tail -50 | grep -E 'LOCKUP|stuck|watchdog'

# 3. إحصائيات الـ lock contention
cat /proc/lockdep_stats

# 4. أكثر الـ locks contention
cat /sys/kernel/debug/locking/lock_stat 2>/dev/null | \
  sort -k3 -rn | head -20
```

**ثانياً: تفعيل الـ lockdep الكامل**

```bash
# ========= تفعيل lockdep verbose mode =========
echo 1 > /proc/sys/kernel/prove_locking  # لو موجود
echo 1 > /proc/sys/kernel/lock_stat

# قراءة الـ stats بعد التحميل
cat /sys/kernel/debug/locking/lock_stat
```

**ثالثاً: ftrace لتتبع الـ spin contention**

```bash
# ========= تتبع كل spinlock contention =========
cd /sys/kernel/debug/tracing

# تنظيف
echo > trace
echo 0 > tracing_on

# تفعيل أحداث الـ lock
echo 1 > events/lock/lock_contended/enable
echo 1 > events/lock/lock_acquired/enable

# فلترة على process معين (مثلاً PID 1234)
echo 'common_pid == 1234' > events/lock/lock_contended/filter

echo 1 > tracing_on
sleep 10
echo 0 > tracing_on

# قراءة النتائج
cat trace | grep -v '^#' | awk '{print $5, $6}' | sort | uniq -c | sort -rn | head -20
```

مثال output ونقرأه:

```
     94 lock_contended: &(&zone->lock)->rlock
     41 lock_contended: &(&rq->lock)->rlock
      8 lock_contended: &my_driver_lock
```

الرقم على الشمال = عدد مرات الـ contention — **كلما ارتفع كلما في مشكلة performance**.

**رابعاً: perf lock analysis**

```bash
# ========= perf lock profiling =========

# تسجيل لمدة 30 ثانية على كل الـ CPUs
perf lock record -a -- sleep 30

# تقرير موجز
perf lock report

# تقرير مفصّل مع الـ wait times
perf lock report --details --verbose

# تحليل الـ contention بالـ call chain
perf lock report -c --sort=wait_total | head -30
```

مثال output مفسَّر:

```
Name                    acquired  contended  total wait (ns)  max wait (ns)  avg wait (ns)
&zone->lock                 4421        102          2341000          18400          22950
&(&rq->lock)->rlock        18923        892         84321000          95000          94530
```

لو `avg wait > 10,000 ns` (10µs) — المشكلة واضحة وتستحق التحقيق.

**خامساً: kprobes لتتبع hold time**

```bash
# ========= kprobe لقياس hold time =========

# بـ bpftrace: قياس طول hold time لـ raw_spin_lock
bpftrace -e '
kprobe:_raw_spin_lock    { @ts[tid] = nsecs; }
kprobe:_raw_spin_unlock  {
    if (@ts[tid]) {
        @hold_us = hist((nsecs - @ts[tid]) / 1000);
        delete(@ts[tid]);
    }
}
END { print(@hold_us); }
' &

BPID=$!
sleep 30
kill $BPID
```

**سادساً: فحص hard lockup بعد crash**

```bash
# ========= تحليل vmcore بعد lockup =========

# تثبيت crash utility
apt-get install crash

# فتح الـ vmcore
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/vmcore

# في الـ crash prompt:
# فحص الـ CPUs المتوقفة
crash> ps
crash> bt -a         # backtrace لكل الـ CPUs

# فحص lock معين
crash> sym my_spinlock
crash> p my_spinlock
crash> struct raw_spinlock my_spinlock.rlock

# البحث عن الـ CPU اللي شايل الـ lock
crash> foreach bt | grep -A5 'spin_lock'
```

**سابعاً: فحص الـ IRQ context والـ spinlock**

```bash
# ========= فحص IRQ distribution وتأثيرها على الـ locks =========

# عدد الـ interrupts لكل CPU
watch -n1 'cat /proc/interrupts | head -30'

# لو interrupt معين بيتركّز على CPU واحد وبيسبب starvation
# إعادة توزيع الـ affinity
echo ff > /proc/irq/42/smp_affinity    # موزّع على كل الـ CPUs

# فحص preemption count (لازم يكون 0 في process context)
# عبر kernel module:
# printk("preempt_count=%d\n", preempt_count());

# فحص lockdep context tracking
grep 'softirq\|hardirq\|in_interrupt' /sys/kernel/debug/locking/lockdep_stats 2>/dev/null
```

**ثامناً: اختبار الـ spinlock under stress**

```bash
# ========= stress test للكشف عن race conditions =========

# تشغيل kernel_test مع lockdep
modprobe test_module  # لو بتبني module للاختبار

# أو استخدام stress-ng مع kernel locking tests
stress-ng --klogbuf 0 --sysinfo 1 --spinlock 4 --timeout 60 &

# مراقبة الـ kernel log في نفس الوقت
dmesg -w | grep -E 'BUG|WARNING|LOCKUP|spin'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Deadlock في درايفر SPI على RK3562 (Industrial Gateway)

#### العنوان
**Deadlock** بين IRQ handler و process context في درايفر SPI مخصص على RK3562

#### السياق
شركة بتعمل industrial gateway بتستخدم RK3562 مع sensor array متصل بـ SPI. الدرايفر المخصص بيقرأ البيانات من الـ sensor في process context وبيعالج interrupt من نفس الـ sensor. المنتج شغال في production لمدة أسبوع وبعدين الجهاز بيتجمد فجأة.

#### المشكلة
الجهاز بيتجمد تماماً — لا response على serial console، لا kernel log جديد. الـ watchdog مش enabled في هذا الـ build. كل CPU بيبقى في حالة spin لا نهائية.

#### التحليل
الكود المشكلة:

```c
/* في process context — بيشتغل من userspace ioctl */
static long spi_sensor_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
    struct sensor_dev *dev = f->private_data;

    spin_lock(&dev->data_lock);          /* LOCK #1 اتخد هنا */
    /* ... بيقرأ بيانات ... */
    spi_sync(dev->spi, &dev->msg);       /* بيكلم SPI framework */
    spin_unlock(&dev->data_lock);
    return 0;
}

/* في IRQ context — بيتنادى من SPI controller interrupt */
static irqreturn_t spi_sensor_irq(int irq, void *data)
{
    struct sensor_dev *dev = data;

    spin_lock(&dev->data_lock);          /* محاولة LOCK #1 تاني — DEADLOCK! */
    dev->new_data_ready = true;
    spin_unlock(&dev->data_lock);
    return IRQ_HANDLED;
}
```

المشكلة واضحة لما نرجع لـ `spinlock.h`:

```c
/* spin_lock في process context — مش بتعطل interrupts */
static __always_inline void spin_lock(spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    raw_spin_lock(&lock->rlock);   /* بس بتعمل lock، مش بتعطل IRQ */
}
```

بينما لو الـ IRQ جه وإنت شايل `spin_lock` العادي:
- الـ IRQ handler بيحاول يعمل `spin_lock` على نفس الـ lock
- الـ CPU بيدخل في spin loop تنتظر release
- الـ release مش هتحصل لأن الكود الأصلي اتوقف بسبب الـ interrupt
- **CPU محجوز في loop لا نهائية = system freeze**

#### الحل
الإصلاح الصح هو استخدام `spin_lock_irqsave` في process context:

```c
static long spi_sensor_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
    struct sensor_dev *dev = f->private_data;
    unsigned long flags;

    /* irqsave = lock + disable local IRQs + save IRQ state */
    spin_lock_irqsave(&dev->data_lock, flags);
    /* الـ IRQ handler مش هيقاطع هنا */
    spi_sync(dev->spi, &dev->msg);
    spin_unlock_irqrestore(&dev->data_lock, flags);
    return 0;
}

/* الـ IRQ handler يفضل بيستخدم spin_lock العادي */
static irqreturn_t spi_sensor_irq(int irq, void *data)
{
    struct sensor_dev *dev = data;

    spin_lock(&dev->data_lock);    /* IRQ context — interrupts already off */
    dev->new_data_ready = true;
    spin_unlock(&dev->data_lock);
    return IRQ_HANDLED;
}
```

الفارق في `spinlock.h` واضح:

```c
/* spin_lock_irqsave — بتعطل interrupts وبتحفظ الـ flags */
#define spin_lock_irqsave(lock, flags)              \
do {                                                \
    raw_spin_lock_irqsave(spinlock_check(lock), flags); \
    __release(spinlock_check(lock)); __acquire(lock);   \
} while (0)

/* spin_unlock_irqrestore — بترجع الـ interrupt state زي ما كانت */
static __always_inline void spin_unlock_irqrestore(spinlock_t *lock,
                                                    unsigned long flags)
    __releases(lock) __no_context_analysis
{
    raw_spin_unlock_irqrestore(&lock->rlock, flags);
}
```

```bash
# تشخيص أثناء الـ freeze بـ JTAG أو magic sysrq
echo t > /proc/sysrq-trigger   # print all tasks
# هتشوف CPU[0] stuck في: __raw_spin_lock → arch_spin_lock
```

#### الدرس المستفاد
أي `spinlock_t` بيتشاركه process context مع IRQ handler **لازم** يتاخد بـ `spin_lock_irqsave` في الـ process context. استخدام `spin_lock` العادي مع shared IRQ resources هو deadlock guarantee مش مجرد bug.

---

### السيناريو الثاني: Data Corruption في UART Driver على STM32MP1 (IoT Sensor Hub)

#### العنوان
**Race condition** في UART receive buffer بسبب استخدام `spin_lock_bh` بدل `spin_lock_irqsave`

#### السياق
IoT hub بيستخدم STM32MP1 بيستقبل بيانات من 8 أجهزة UART في نفس الوقت. كل UART عنده circular buffer. البيانات بتتاخد من الـ UART interrupt وبتتقرأ من thread بيشتغل على network stack (softirq context). بعد تحميل شديد، الـ buffer بيبان فيه corruption.

#### المشكلة
الـ developer استخدم `spin_lock_bh` ظناً إنه كافي:

```c
struct uart_port_priv {
    spinlock_t buf_lock;
    u8 rx_buf[4096];
    int head, tail;
};

/* في UART IRQ handler — HARD IRQ context */
static irqreturn_t stm32_uart_irq(int irq, void *data)
{
    struct uart_port_priv *p = data;
    u8 byte = readb(p->base + UART_DR);

    spin_lock(&p->buf_lock);      /* IRQ — بيستخدم simple lock */
    p->rx_buf[p->head] = byte;
    p->head = (p->head + 1) % 4096;
    spin_unlock(&p->buf_lock);
    return IRQ_HANDLED;
}

/* في network softirq (BH context) — بيجمع بيانات ويبعتها */
static void stm32_uart_bh_handler(struct tasklet_struct *t)
{
    struct uart_port_priv *p = from_tasklet(p, t, tasklet);
    unsigned long flags;

    spin_lock_bh(&p->buf_lock);   /* بتعطل softirqs بس مش hard IRQs */
    /* ... بتقرأ من buf ... */
    spin_unlock_bh(&p->buf_lock);
}
```

#### التحليل
الفهم الخاطئ: `spin_lock_bh` كافي لأن handler هو softirq.

الحقيقة من `spinlock.h`:

```c
/* spin_lock_bh — بتعطل softirqs/tasklets/BH فقط */
static __always_inline void spin_lock_bh(spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    raw_spin_lock_bh(&lock->rlock);
    /* raw_spin_lock_bh → local_bh_disable() + spin_lock() */
}
```

الـ `local_bh_disable()` بتمنع softirqs من الـ preempt، لكن **hard IRQ ممكن يجي في أي وقت**. التسلسل الخطير:

```
CPU0: spin_lock_bh → disable BH → acquire lock
CPU0: بيقرأ buf[tail..head]
  ↓
HARD IRQ fires on CPU0!
  → stm32_uart_irq tries spin_lock → SPIN WAITS
  → IRQ لا يقدر يكمل، buf مش بيتكتب
  → لو CPU0 نفسه = DEADLOCK
```

على SMP: لو CPU1 هو اللي عنده الـ UART IRQ، هتحصل race window.

#### الحل
الصح هو `spin_lock_irqsave` في الـ BH handler:

```c
static void stm32_uart_bh_handler(struct tasklet_struct *t)
{
    struct uart_port_priv *p = from_tasklet(p, t, tasklet);
    unsigned long flags;

    /* irqsave = disable IRQs + disable BH + acquire lock */
    spin_lock_irqsave(&p->buf_lock, flags);
    while (p->tail != p->head) {
        u8 b = p->rx_buf[p->tail];
        p->tail = (p->tail + 1) % 4096;
        /* process b */
    }
    spin_unlock_irqrestore(&p->buf_lock, flags);
}
```

```
متى تستخدم إيه؟
┌─────────────────────┬───────────────────────────┬──────────────────────────┐
│ Process context     │ Shared with softirq only  │ spin_lock_bh             │
│ Process context     │ Shared with hard IRQ      │ spin_lock_irqsave        │
│ Softirq/Tasklet     │ Shared with hard IRQ      │ spin_lock_irqsave        │
│ Hard IRQ handler    │ (already IRQs disabled)   │ spin_lock                │
└─────────────────────┴───────────────────────────┴──────────────────────────┘
```

#### الدرس المستفاد
**الـ `spin_lock_bh` مش بدل `spin_lock_irqsave`**. قرار اختيار الـ variant مش بيعتمد على context الـ consumer بس، لكن على كل الـ contexts اللي ممكن يشتركوا في نفس الـ lock.

---

### السيناريو الثاني: Lockdep Warning أثناء Board Bring-up على i.MX8 (Automotive ECU)

#### العنوان
**Lockdep** يكشف potential deadlock في I2C driver أثناء custom board bring-up على i.MX8MQ

#### السياق
فريق bring-up بيشتغل على ECU مخصص للسيارة بيستخدم i.MX8MQ. الـ I2C bus بيتحكم في شبكة sensors (temperature، pressure). الـ kernel مبني بـ `CONFIG_DEBUG_SPINLOCK=y` و`CONFIG_DEBUG_LOCK_ALLOC=y` للـ bring-up phase. في الـ dmesg بيظهر warning غريب مش بيسبب crash فوري.

#### المشكلة
في dmesg:
```
WARNING: possible circular locking dependency detected
imx8-i2c-drv/1234 is trying to acquire lock:
  (&bus->lock){....}, at: imx_i2c_isr+0x84
but task is already holding lock:
  (&dev->pm_lock){....}, at: imx_i2c_runtime_resume+0x40
which lock ordering was established by: ...
```

#### التحليل
الـ `CONFIG_DEBUG_SPINLOCK` في `spinlock.h` بيغير الـ behavior:

```c
#ifdef CONFIG_DEBUG_SPINLOCK
  extern void __raw_spin_lock_init(raw_spinlock_t *lock, const char *name,
                   struct lock_class_key *key, short inner);

# define raw_spin_lock_init(lock)                   \
do {                                                \
    static struct lock_class_key __key;             \
    __raw_spin_lock_init((lock), #lock, &__key, LD_WAIT_SPIN); \
} while (0)
```

كل `spin_lock_init` بيسجل الـ lock مع lockdep بـ `lock_class_key` مميزة. الـ lockdep engine بيتتبع **ترتيب اتخاذ الـ locks** عبر كل الـ code paths ويكشف circular dependencies قبل ما تحصل فعلاً.

الـ driver كان بيعمل:
```
Path A: pm_lock → bus->lock   (في runtime_resume ثم ISR setup)
Path B: bus->lock → pm_lock   (في i2c_transfer ثم power management)
```

الـ lockdep شاف إن الـ ordering ممكن ينعكس = potential deadlock.

التحقق من الـ lock init في الـ kernel:
```c
/* لو spin_lock_init اتعمل صح، الـ lockdep بيسجل الـ class */
# define spin_lock_init(lock)               \
do {                                        \
    static struct lock_class_key __key;     \
    __raw_spin_lock_init(spinlock_check(lock),  \
             #lock, &__key, LD_WAIT_CONFIG); \
} while (0)
```

#### الحل
إصلاح ترتيب الـ locking:

```c
/* الـ rule: دايماً pm_lock يتاخد بعد bus->lock */
static int imx_i2c_xfer(struct i2c_adapter *adap, ...)
{
    struct imx_i2c_dev *dev = i2c_get_adapdata(adap);
    unsigned long flags;

    spin_lock_irqsave(&dev->bus_lock, flags);   /* FIRST: bus lock */
    pm_runtime_get_sync(dev->dev);               /* THEN: power */
    /* ... transfer ... */
    pm_runtime_put(dev->dev);
    spin_unlock_irqrestore(&dev->bus_lock, flags);
    return 0;
}

/* التحقق بعد الإصلاح */
static irqreturn_t imx_i2c_isr(int irq, void *data)
{
    struct imx_i2c_dev *dev = data;

    spin_lock(&dev->bus_lock);    /* نفس الترتيب — IRQ context */
    /* handle ISR */
    spin_unlock(&dev->bus_lock);
    return IRQ_HANDLED;
}
```

```bash
# تفعيل lockdep reporting
echo 1 > /proc/sys/kernel/prove_locking

# عرض lock statistics
cat /proc/lock_stat

# إعادة تشغيل lockdep بعد الإصلاح للتحقق
echo 0 > /debug/lockdep          # reset
echo 1 > /debug/lockdep          # re-enable
```

#### الدرس المستفاد
الـ `CONFIG_DEBUG_SPINLOCK` و`CONFIG_DEBUG_LOCK_ALLOC` هم أدوات **لازم تكون فايتة في bring-up phase**. الـ lockdep بيكشف bugs ممكن ما تظهرش في testing بسبب timing، لكن هتظهر في production تحت load.

---

### السيناريو الرابع: Stall في HDMI Driver على Allwinner H616 (Android TV Box)

#### العنوان
**Spinlock held too long** في HDMI hotplug handler بيسبب audio stutter على Allwinner H616

#### السياق
Android TV box بيستخدم Allwinner H616. المنتج شغال تمام لكن عند plug/unplug الـ HDMI cable أثناء تشغيل video، الـ audio بيعمل stutter أو يوقف لمدة ثانية. المستخدمين بيشتكوا من الـ issue على production.

#### المشكلة
الـ developer عمل HDMI hotplug handler بيعمل عمليات كتير تحت spinlock:

```c
struct h616_hdmi {
    spinlock_t      hpd_lock;
    struct edid     *edid_data;
    /* ... */
};

static irqreturn_t h616_hdmi_hpd_irq(int irq, void *data)
{
    struct h616_hdmi *hdmi = data;

    spin_lock(&hdmi->hpd_lock);

    /* قراءة EDID من I2C — بتاخد وقت طويل! */
    drm_get_edid(hdmi->connector, hdmi->ddc);    /* PROBLEM: sleeps! */

    /* إعادة تهيئة display pipeline */
    h616_hdmi_reconfigure_all(hdmi);              /* PROBLEM: بيستغرق ms */

    spin_unlock(&hdmi->hpd_lock);
    return IRQ_HANDLED;
}
```

#### التحليل
الـ spinlock في `spinlock.h` مصمم لـ **atomic short-duration** critical sections:

```c
static inline void do_raw_spin_lock(raw_spinlock_t *lock) __acquires(lock)
{
    __acquire(lock);
    arch_spin_lock(&lock->raw_lock);   /* busy-wait — لا sleep */
    mmiowb_spin_lock();
}
```

الـ spinlock **لا يسمح بالـ sleep** أثناء holding. الـ `drm_get_edid` بيعمل I2C transactions اللي بتستخدم `msleep` أو `wait_event`. ده بيتسبب في:

1. Spinlock بيتمسك لـ 50-200ms (وقت EDID read)
2. كل الـ CPUs اللي محتاجة نفس الـ lock هتعمل busy-spin طول الوقت ده
3. الـ audio subsystem محتاج CPU time لـ DMA buffer refill
4. الـ audio يتأخر = stutter

الـ kernel في `spinlock.h` بيوفر `spin_needbreak` لاكتشاف الـ contention:

```c
static inline int spin_needbreak(spinlock_t *lock)
{
    if (!preempt_model_preemptible())
        return 0;
    return spin_is_contended(lock);   /* في raw_spin_is_contended */
}
```

لكن المشكلة الأعمق هي إن الـ design خاطئ أصلاً — EDID read مش من الـ IRQ handler.

#### الحل
تحويل العمل الثقيل لـ work queue:

```c
struct h616_hdmi {
    spinlock_t          hpd_lock;
    bool                hpd_pending;
    struct work_struct  hpd_work;    /* work queue بدل IRQ context */
    /* ... */
};

/* IRQ handler — سريع جداً */
static irqreturn_t h616_hdmi_hpd_irq(int irq, void *data)
{
    struct h616_hdmi *hdmi = data;

    spin_lock(&hdmi->hpd_lock);
    hdmi->hpd_pending = true;        /* بس flag بسيطة */
    spin_unlock(&hdmi->hpd_lock);

    schedule_work(&hdmi->hpd_work);  /* فرّغ الـ IRQ فوراً */
    return IRQ_HANDLED;
}

/* Work queue — process context — ممكن يعمل sleep */
static void h616_hdmi_hpd_work(struct work_struct *work)
{
    struct h616_hdmi *hdmi = container_of(work, struct h616_hdmi, hpd_work);

    /* هنا ممكن نستخدم mutex بدل spinlock */
    mutex_lock(&hdmi->config_mutex);
    drm_get_edid(hdmi->connector, hdmi->ddc);   /* safe هنا */
    h616_hdmi_reconfigure_all(hdmi);
    mutex_unlock(&hdmi->config_mutex);
}
```

```bash
# قياس spinlock contention
perf stat -e lock:contention_begin,lock:contention_end \
    -p $(pgrep audio_hal) -- sleep 5

# فحص IRQ latency
cat /proc/interrupts | grep hdmi
# قارن قبل وبعد الإصلاح
```

#### الدرس المستفاد
**Spinlock = يعمل atomic operations بسرعة فقط**. أي عملية بتاخد أكتر من microseconds (I2C، memory alloc، filesystem) لازم تتنقل لـ work queue أو mutex. الـ `do_raw_spin_lock` في `spinlock.h` عمله busy-wait — كل ثانية تحت الـ lock بتضيع CPU time.

---

### السيناريو الخامس: PREEMPT_RT Bug في SPI Flash Driver على AM62x (Industrial HMI)

#### العنوان
**PREEMPT_RT incompatibility** في custom SPI flash driver على AM62x بيسبب priority inversion

#### السياق
Industrial HMI panel بيستخدم AM62x من Texas Instruments. الـ customer محتاج real-time response للـ touch input (< 1ms latency). الـ team قرر يستخدم `CONFIG_PREEMPT_RT=y`. بعد التفعيل، الـ system بيبدأ يعمل stack traces وبعض الـ operations بتتأخر بشكل غير متوقع.

#### المشكلة
الـ SPI flash driver كان بيستخدم `spinlock_t` في كود بيدعى من process context مع عمليات طويلة:

```c
/* Driver كان شغال تمام على non-RT kernel */
struct spi_flash_dev {
    spinlock_t  op_lock;
    /* ... */
};

static int spi_flash_write_page(struct spi_flash_dev *dev,
                                 u32 addr, const u8 *buf, size_t len)
{
    spin_lock(&dev->op_lock);      /* على RT: هذا ممكن يسبب مشكلة */

    spi_write(dev->spi, buf, len); /* بتاخد وقت */
    wait_for_completion(&dev->write_done); /* SLEEP هنا! */

    spin_unlock(&dev->op_lock);
    return 0;
}
```

#### التحليل
على non-RT kernel، `spinlock_t` maps to `raw_spinlock_t`:

```c
/* Non PREEMPT_RT kernel, map to raw spinlocks: */
#ifndef CONFIG_PREEMPT_RT

static __always_inline void spin_lock(spinlock_t *lock)
    __acquires(lock) __no_context_analysis
{
    raw_spin_lock(&lock->rlock);   /* direct: arch_spin_lock */
}
```

لكن مع `CONFIG_PREEMPT_RT=y`، الـ `spinlock_t` بتتحول لـ `rt_mutex`:

```c
#else  /* !CONFIG_PREEMPT_RT */
# include <linux/spinlock_rt.h>
#endif /* CONFIG_PREEMPT_RT */
```

من `spinlock_types.h`:
```c
/* PREEMPT_RT kernels map spinlock to rt_mutex */
context_lock_struct(spinlock) {
    struct rt_mutex_base    lock;   /* مش raw_spinlock! */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

الـ `rt_mutex` يسمح بالـ sleep وعنده priority inheritance. لكن المشكلة إن `wait_for_completion` تحت `spinlock` على RT كان بيخلق:
1. High-priority real-time task بيحاول يشتغل
2. بيلاقي الـ RT mutex ممسكه الـ flash writer (low priority)
3. Priority inversion scenario حتى مع inheritance

الفرق بين `raw_spinlock_t` و`spinlock_t` على RT مهم جداً:

```
على CONFIG_PREEMPT_RT=y:
┌──────────────────┬──────────────────────────────────────────┐
│ raw_spinlock_t   │ فعلاً atomic busy-wait، مش قابل للـ sleep│
│                  │ استخدم للـ really critical sections       │
│ spinlock_t       │ يتحول لـ rt_mutex — يسمح بـ sleep        │
│                  │ آمن للكود الطبيعي                         │
└──────────────────┴──────────────────────────────────────────┘
```

#### الحل
على RT kernel، إزالة `wait_for_completion` من تحت الـ lock أو تحويل الـ design:

```c
/* إصلاح: فصل الـ locking عن الـ waiting */
static int spi_flash_write_page(struct spi_flash_dev *dev,
                                 u32 addr, const u8 *buf, size_t len)
{
    int ret;

    /* mutex للـ mutual exclusion — صح على RT */
    mutex_lock(&dev->op_mutex);

    reinit_completion(&dev->write_done);
    ret = spi_async(dev->spi, &dev->msg);    /* non-blocking */
    if (ret) {
        mutex_unlock(&dev->op_mutex);
        return ret;
    }

    /* الـ wait خارج الـ lock لو ممكن، أو استخدم mutex بدل spinlock */
    wait_for_completion(&dev->write_done);

    mutex_unlock(&dev->op_mutex);
    return 0;
}
```

لو محتاج `raw_spinlock_t` لـ hardware register access سريع:

```c
/* raw_spinlock للـ register access فقط — لا sleep */
static void spi_am62x_start_transfer(struct spi_flash_dev *dev)
{
    unsigned long flags;

    raw_spin_lock_irqsave(&dev->reg_lock, flags);
    writel(CMD_START, dev->base + SPI_CMD_REG);    /* atomic: fast */
    raw_spin_unlock_irqrestore(&dev->reg_lock, flags);
}
```

```bash
# اختبار RT latency بعد الإصلاح
cyclictest -p 99 -t 4 -n -i 200 -l 10000

# فحص priority inversion incidents
cat /proc/sys/kernel/sched_rt_runtime_us
cat /proc/sys/kernel/sched_rt_period_us

# kernel config check
zcat /proc/config.gz | grep -E "PREEMPT_RT|PREEMPT="
```

#### الدرس المستفاد
الكود اللي بيشتغل صح على non-RT kernel **مش بالضرورة** هيشتغل على `PREEMPT_RT`. لما `CONFIG_PREEMPT_RT=y`، الـ `spinlock_t` بيتحول لـ `rt_mutex` (كما في `spinlock_types.h`). الـ driver اللي محتاج يشتغل على الاتنين لازم يفهم الفرق بين `spinlock_t` (rt-safe) و`raw_spinlock_t` (real raw busy-wait) وياخد قراره على أساسه.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المصدر الأساسي لتتبع تطور الـ spinlock في الـ kernel — كل مقالة جوا بتحكي عن مرحلة مختلفة من التطور:

| المقالة | الوصف |
|---------|--------|
| [Ticket spinlocks](https://lwn.net/Articles/267968/) | شرح آلية الـ ticket spinlock وإزاي بتضمن fairness |
| [MCS locks and qspinlocks](https://lwn.net/Articles/590243/) | المقالة الأهم — بتشرح ليه MCS أحسن من ticket وإزاي الـ qspinlock اتبنى فوقيه |
| [Local locks in the kernel](https://lwn.net/Articles/828477/) | بتتكلم عن `local_lock` والعلاقة بين spinlocks والـ PREEMPT_RT |
| [Improving ticket spinlocks](https://lwn.net/Articles/531254/) | محاولات تحسين ticket spinlocks قبل ما يتبدلوا بـ qspinlocks |
| [fast queue spinlocks](https://lwn.net/Articles/533636/) | تفاصيل تقنية عن الـ fast-path في qspinlocks |
| [Unified spinlock initialization](https://lwn.net/Articles/109505/) | توحيد طريقة الـ initialization عبر الـ architectures |
| [spinlock consolidation](https://lwn.net/Articles/151210/) | دمج implementations المختلفة في codebase موحد |
| [A realtime preemption overview](https://lwn.net/Articles/146861/) | تأثير PREEMPT_RT على السلوك الكامل للـ spinlocks |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة جوا شجرة الـ kernel مباشرةً:

```
Documentation/locking/spinlocks.rst    ← الدليل الرئيسي للـ spinlocks
Documentation/locking/locktypes.rst   ← الفرق بين lock types وقواعد الاستخدام
Documentation/locking/lockdep-design.txt ← تصميم الـ lockdep
Documentation/locking/mutex-design.rst   ← للمقارنة مع mutex
```

**الـ** online version:
- [Locking lessons — kernel.org](https://www.kernel.org/doc/html/latest/locking/spinlocks.html)
- [Lock types and their rules](https://www.kernel.org/doc/html/latest/locking/locktypes.html)
- [spinlocks.txt (النسخة القديمة)](https://www.kernel.org/doc/Documentation/locking/spinlocks.txt)

---

### Commits مهمة في تاريخ الـ Spinlock

| الحدث | الوصف |
|-------|--------|
| **qspinlock introduction** (~v4.2) | استبدال ticket spinlock بـ queued spinlock في x86، بداية الـ `kernel/locking/qspinlock.c` |
| **MCS lock integration** | إدخال بنية MCS كـ slow-path للـ qspinlock |
| **PREEMPT_RT spinlock** | تحويل `spinlock_t` إلى sleeping lock فوق `rt_mutex` عند تفعيل PREEMPT_RT |
| **lockdep integration** | ربط الـ spinlock بنظام الـ lockdep لكشف deadlocks |

للبحث في الـ commits:
```bash
git log --oneline -- kernel/locking/spinlock.c
git log --oneline -- include/linux/spinlock.h
git log --grep="qspinlock" --oneline
```

---

### نقاشات Mailing List

- [lore.kernel.org — locking subsystem](https://lore.kernel.org/linux-kernel/?q=spinlock)
- [qspinlock improvements patch series](https://lore.kernel.org/linux-arm-kernel/d9b1cc76-7878-43d2-bda9-b3bf074b5c9e@redhat.com/T/)
- [LKML Archive](https://lkml.org/) — ابحث بـ `spinlock` أو `qspinlock`
- [recursive spinlocks — مناقشة Linus الشهيرة](https://linux-kernel.vger.kernel.narkive.com/tJhbsqVs/recursive-spinlocks-shoot)
- [Kernel Locking Techniques — Linux Journal](https://www.linuxjournal.com/article/5833)

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 5**: Concurrency and Race Conditions
- بيغطي `spin_lock()`, `spin_lock_irqsave()`, rwlocks
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 9**: An Introduction to Kernel Synchronization
- **الفصل 10**: Kernel Synchronization Methods
- بيشرح متى تستخدم spinlock vs mutex، والـ BH context، والـ IRQ context

#### Understanding the Linux Kernel — Bovet & Cesati
- **الفصل 5**: Kernel Synchronization
- أعمق تقنياً — بيدخل في تفاصيل الـ assembly

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 14**: Kernel Debugging Techniques
- بيتكلم عن `CONFIG_DEBUG_SPINLOCK` وأدوات كشف مشاكل الـ locking في Embedded systems

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- بيغطي تطور الـ locking primitives عبر الإصدارات

---

### Kernelnewbies.org

- [Linux 4.2 — qspinlock introduction](https://kernelnewbies.org/Linux_4.2) — وثّق إدخال الـ queue-based spinlocks
- [Linux 3.12 — paravirt ticket spinlocks](https://kernelnewbies.org/Linux_3.12) — paravirtualized ticket spinlock improvements
- [Linux 5.15 — locking changes](https://kernelnewbies.org/Linux_5.15)
- [spin_lock_bh vs local_bh_disable — mailing list](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2014-June/010977.html)
- [is it safe to call spin_lock_bh() from softirq](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2011-May/001921.html)

---

### eLinux.org

- [Debugging kernel options performance impact](https://elinux.org/R-Car/Debugging-kernel-options-causes-performance-down) — تأثير `CONFIG_DEBUG_SPINLOCK` على الأداء في embedded systems
- [Kernel module binary compatibility with debug features](https://elinux.org/Kernel_module_binary_compatibility_with_debug_features) — `CONFIG_DEBUG_SPINLOCK` وتوافق الـ modules

---

### مصادر تقنية إضافية

- [Introduction to spinlocks — linux-insides](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-1.html)
- [Queued spinlocks — linux-insides](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-2.html)
- [Lock types and rules — kernel.org](https://www.kernel.org/doc/html/latest/locking/locktypes.html)
- [Real-time theory of operation](https://www.kernel.org/doc/html/next/core-api/real-time/theory.html)
- [Linux kernel locks — Real-time Ubuntu](https://documentation.ubuntu.com/real-time/latest/explanation/locks/)
- [Kernel Locking Deep Dive — Medium](https://deepdives.medium.com/kernel-locking-deep-dive-into-spinlocks-part-1-bcdc46ee8df6)

---

### الملفات المصدرية الأساسية في الـ Kernel

```
include/linux/spinlock.h          ← نقطة الدخول الرئيسية (الملف ده)
include/linux/spinlock_types.h    ← تعريف spinlock_t و raw_spinlock_t
include/linux/spinlock_api_smp.h  ← API declarations على SMP
include/linux/spinlock_api_up.h   ← API declarations على UP
include/linux/spinlock_rt.h       ← PREEMPT_RT variant
include/linux/rwlock.h            ← reader-writer locks
kernel/locking/spinlock.c         ← slow-path implementation
kernel/locking/qspinlock.c        ← queued spinlock implementation
kernel/locking/spinlock_debug.c   ← debug implementation
asm/spinlock.h                    ← architecture-specific fast-path
```

---

### Search Terms للبحث عن معلومات أكثر

```
linux kernel qspinlock implementation
linux spinlock PREEMPT_RT sleeping lock
raw_spinlock_t vs spinlock_t difference
linux kernel ticket spinlock fairness
mcs_spinlock linux kernel
spin_lock_irqsave when to use
linux lockdep spinlock validation
kernel locking best practices
CONFIG_DEBUG_SPINLOCK kernel option
linux spinlock memory ordering barriers
smp_mb__after_spinlock acquire semantics
atomic_dec_and_lock pattern
```
## Phase 8: Writing simple module

### الفكرة

**`_raw_spin_lock`** هي الدالة الأساسية اللي بتنفّذ فعلياً الـ spinlock acquisition في الـ kernel — كل `spin_lock()` في النهاية بتوصّل ليها. هنعمل **kprobe** عليها عشان نشوف مين بياخد الـ lock، من أي CPU، وإيه اسم الـ task اللي طلب.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_spinlock.c
 *
 * Hooks _raw_spin_lock() via kprobe to log every raw spinlock acquisition:
 *   - task name & PID
 *   - CPU number
 *   - lock address
 *   - whether interrupts are enabled at call time
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit         */
#include <linux/kernel.h>       /* pr_info()                                 */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe()          */
#include <linux/spinlock.h>     /* raw_spinlock_t definition                 */
#include <linux/irqflags.h>     /* irqs_disabled()                           */
#include <linux/sched.h>        /* current->comm, current->pid               */
#include <linux/smp.h>          /* smp_processor_id()                        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on _raw_spin_lock: log every raw spinlock acquisition");

/* ------------------------------------------------------------------
 * pre-handler: called right BEFORE _raw_spin_lock() executes.
 *
 * @p   : pointer to the kprobe struct (contains symbol name etc.)
 * @regs: CPU register snapshot at the probe point
 *
 * The first argument of _raw_spin_lock(raw_spinlock_t *lock) is
 * passed in the first function-argument register:
 *   x86-64  -> regs->di
 *   arm64   -> regs->regs[0]
 * We use regs_get_kernel_argument() which is arch-neutral since 5.11.
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Grab the first argument: the raw_spinlock_t pointer */
    raw_spinlock_t *lock = (raw_spinlock_t *)regs_get_kernel_argument(regs, 0);

    pr_info("kprobe_spinlock: [CPU%u] task=%-16s pid=%d  lock=%px  irqs_%s\n",
            smp_processor_id(),          /* which core is running this */
            current->comm,               /* task name (max 16 chars)   */
            current->pid,                /* task PID                   */
            lock,                        /* lock virtual address       */
            irqs_disabled() ? "OFF" : "ON"); /* interrupt state        */

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------
 * post-handler: called right AFTER _raw_spin_lock() returns.
 * We just note the lock was acquired successfully.
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("kprobe_spinlock: [CPU%u] task=%-16s pid=%d  lock acquired OK\n",
            smp_processor_id(),
            current->comm,
            current->pid);
}

/* ------------------------------------------------------------------
 * kprobe descriptor — one probe, one symbol.
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name    = "_raw_spin_lock",  /* exact kernel symbol to probe */
    .pre_handler    = handler_pre,       /* fires before the function    */
    .post_handler   = handler_post,      /* fires after the function     */
};

/* ------------------------------------------------------------------
 * module_init: register the kprobe with the kernel.
 * ------------------------------------------------------------------ */
static int __init kprobe_spinlock_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_spinlock: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_spinlock: planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit: unregister the kprobe BEFORE the module is removed.
 * Without this, the kernel would jump to freed memory on next
 * _raw_spin_lock() call — guaranteed crash.
 * ------------------------------------------------------------------ */
static void __exit kprobe_spinlock_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_spinlock: removed from %p\n", kp.addr);
}

module_init(kprobe_spinlock_init);
module_exit(kprobe_spinlock_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يجيب `struct kprobe` و `register_kprobe()` و `regs_get_kernel_argument()` |
| `linux/spinlock.h` | تعريف `raw_spinlock_t` عشان نقدر نعمل cast للـ argument |
| `linux/irqflags.h` | `irqs_disabled()` بتقول هل الـ interrupts معطّلة وقت الاستدعاء |
| `linux/sched.h` | `current->comm` و `current->pid` معرّفين هنا |
| `linux/smp.h` | `smp_processor_id()` بيرجع رقم الـ CPU الشغّال دلوقتي |

الـ includes دي بتجيب كل اللي الـ handler محتاجه من معلومات عن الـ context الحالي بدون أي overhead زيادة.

---

#### الـ `handler_pre`

بيتشغّل قبل ما `_raw_spin_lock()` تبدأ فعلياً — يعني الـ lock لسه مش متاخد. **الـ`regs_get_kernel_argument(regs, 0)`** بتجيب الـ argument الأول (الـ lock pointer) بطريقة portable على كل الـ architectures بدل ما نقول `regs->di` مباشرة وتبقى x86-only.

---

#### الـ `handler_post`

بيتشغّل بعد ما الدالة ترجع بنجاح — يعني الـ lock اتاخد. بنطبع رسالة تأكيد خفيفة. ممكن تحذفه لو الـ noise كتير، بس بيوضّح إن الـ acquisition اتمّ فعلاً.

---

#### الـ `struct kprobe kp`

الـ **`.symbol_name`** بتحدد الدالة المستهدفة بالاسم بدل الـ address — الـ kernel بيعمل lookup في الـ kallsyms تلقائياً. لو الدالة inline أو مش exported، الـ kprobe هيفشل بـ `-EINVAL` وهنشوفه في `dmesg`.

---

#### الـ `module_init`

`register_kprobe()` بتحط **breakpoint** (أو trap instruction حسب الـ arch) في جسم `_raw_spin_lock` وبتربطه بالـ handlers بتاعتنا. لو الدالة مش موجودة أو الـ kprobes مش enabled في الـ kernel config، الدالة بترجع error وبنطبعه ونخرج بسلام.

---

#### الـ `module_exit`

`unregister_kprobe()` بتشيل الـ breakpoint وبتضمن إن الـ handlers لن يتنادوا تاني. **لازم تتعمل قبل الـ unload** عشان لو الـ handler code اتحرر من الـ memory وجه call لـ `_raw_spin_lock()` — الـ kernel هيعمل oops.

---

### Makefile

```makefile
obj-m += kprobe_spinlock.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتتبع المخرجات

```bash
# بناء الـ module
make

# تحميله
sudo insmod kprobe_spinlock.ko

# مشاهدة الـ log في real-time
sudo dmesg -w | grep kprobe_spinlock

# مثال على المخرجات:
# kprobe_spinlock: planted at ffffffff810a1234 (_raw_spin_lock)
# kprobe_spinlock: [CPU2] task=kworker/2:1     pid=87   lock=ffff888003c20000  irqs_ON
# kprobe_spinlock: [CPU2] task=kworker/2:1     pid=87   lock acquired OK
# kprobe_spinlock: [CPU0] task=bash            pid=1234 lock=ffff888004a10080  irqs_OFF

# إزالته
sudo rmmod kprobe_spinlock
```

> **ملاحظة مهمة:** الـ `_raw_spin_lock` بتتنادى بكثافة جداً — في production system ممكن تغرق الـ dmesg. الأفضل تضيف **filter بـ `current->comm`** أو تستخدم **`atomic_t counter`** وتطبع كل N استدعاء فقط.
