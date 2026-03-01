## Phase 1: الصورة الكبيرة ببساطة

### ينتمي لأي subsystem؟

الـ `rwlock.h` جزء من subsystem اسمه **LOCKING PRIMITIVES** — وده من أهم subsystems في الـ kernel، بيتولاه Peter Zijlstra و Ingo Molnar وغيرهم من كبار المطورين.

---

### قصة المشكلة — ليه احتجنا rwlock أصلاً؟

تخيل عندك **قاموس مشترك** بين 100 شخص في نفس الوقت.

- معظم الوقت الناس بتـ**قرأ** من القاموس بس — مفيش مشكلة لو 10 أشخاص قرأوا في نفس اللحظة، القاموس مش هيتأثر.
- بس لو حد واحد بدأ يـ**يكتب** — يعدّل كلمة أو يحذف — لازم الكل يوقف القراءة، عشان محدش يشوف بيانات ناقصة أو غلط.

الـ **spinlock** العادي مش كفء هنا — بيمنع حتى القراءات المتوازية ويخلي الكل ينتظر. الحل هو الـ **`rwlock` (Read-Write Lock)**:

- ممكن **قراء كتير** يشتغلوا في نفس الوقت.
- لو في **كاتب واحد** — كل القراء والكتّاب الباقيين بيستنوا.

ده الـ **readers-writer lock** — أداة synchronization بتوازن بين التوازي والصحة.

---

### ما هو `rwlock.h` بالضبط؟

الـ `include/linux/rwlock.h` هو **واجهة الـ macros** اللي بتعرّض الـ rwlock API للكود الباقي في الـ kernel. هو مش بيعمل الـ implementation نفسها — بس بيربط الـ high-level API بالـ low-level raw operations.

الفايل ده بيعمل 3 حاجات:

1. **يعرّف `rwlock_init`** — لتهيئة الـ lock قبل الاستخدام.
2. **يعرّف الـ `do_raw_*` operations** — الطبقة اللي بتنادي الـ arch-specific code مباشرة.
3. **يعرّف الـ public API macros** كـ `read_lock`, `write_lock`, `read_unlock`, إلخ — بكل صيغهم (عادي، مع IRQ، مع BH).

---

### طبقات الـ rwlock — من الأعلى للأسفل

```
كود الـ kernel (مثلاً driver أو filesystem)
         │
         ▼
   read_lock(lock)         ← rwlock.h (هنا احنا)
         │
         ▼
   _raw_read_lock(lock)    ← rwlock_api_smp.h
         │
         ▼
   do_raw_read_lock(lock)  ← rwlock.h أيضاً
         │
         ▼
   arch_read_lock(...)     ← asm/spinlock.h (architecture specific)
         │
         ▼
   CPU hardware instructions (e.g., LOCK XADD on x86)
```

---

### الـ variants المختلفة في الـ API

| الـ macro | الوظيفة |
|---|---|
| `read_lock(lock)` | قفل للقراءة فقط |
| `write_lock(lock)` | قفل حصري للكتابة |
| `read_trylock(lock)` | حاول تاخد القفل للقراءة، ولو مش متاح ارجع فوراً |
| `write_trylock(lock)` | نفس الفكرة للكتابة |
| `read_lock_irq(lock)` | قفل للقراءة مع تعطيل الـ IRQs |
| `write_lock_irq(lock)` | قفل للكتابة مع تعطيل الـ IRQs |
| `read_lock_irqsave(lock, flags)` | قفل وحفظ حالة الـ IRQs |
| `write_lock_irqsave(lock, flags)` | نفس الفكرة للكتابة |
| `read_lock_bh(lock)` | قفل مع تعطيل الـ bottom half |
| `write_lock_bh(lock)` | نفس الفكرة للكتابة |
| `read_unlock(lock)` | تحرير قفل القراءة |
| `write_unlock(lock)` | تحرير قفل الكتابة |
| `rwlock_is_contended(lock)` | هل في حد تاني بينتظر الـ lock؟ |

---

### ليه في variants كتير؟

الـ kernel بيشتغل في سياقات مختلفة:

- **Process context**: ممكن تستخدم `read_lock` العادية.
- **Interrupt context**: لازم تعطّل الـ IRQs عشان الـ interrupt handler نفسه ميجيش يطلب نفس الـ lock ويحصل deadlock — فبتستخدم `read_lock_irq` أو `read_lock_irqsave`.
- **Softirq/BH context**: نفس الفكرة مع الـ bottom half — بتستخدم `read_lock_bh`.

---

### الفرق بين `CONFIG_SMP` و `CONFIG_DEBUG_SPINLOCK`

الـ `rwlock.h` بيتغير سلوكه حسب الـ config:

- **`CONFIG_DEBUG_SPINLOCK` مفعّل**: بتتنادى functions حقيقية (extern) بدل macros — عشان يقدر يعمل runtime checks على الـ locking.
- **SMP off (UP - Uniprocessor)**: بعض الـ variants بتتصرف بشكل مختلف في كيفية تمرير الـ `flags`.

---

### الـ PREEMPT_RT خليها في بالك

في الـ kernels اللي عندها `CONFIG_PREEMPT_RT`، الـ `rwlock_t` مش بتبقى based على spinlock — بتتحول لـ **sleeping lock** مبنية على `rwbase_rt`. الـ `rwlock.h` نفسه ما بيتغيرش — التغيير بيحصل في الـ types والـ implementation تحت.

---

### الفايلات المرتبطة اللي لازم تعرفها

**Core headers:**

| الفايل | الدور |
|---|---|
| `include/linux/spinlock.h` | الـ entry point الوحيد — لازم تـ include ده مش rwlock.h مباشرة |
| `include/linux/rwlock_types.h` | يعرّف `rwlock_t` و `__RW_LOCK_UNLOCKED` |
| `include/linux/rwlock.h` | ده الفايل نفسه — الـ macros الرئيسية |
| `include/linux/rwlock_api_smp.h` | prototypes الـ `_raw_*` functions للـ SMP |
| `include/linux/rwlock_rt.h` | الـ PREEMPT_RT variant للـ rwlock |
| `include/linux/spinlock_types.h` | يـ include الـ rwlock_types.h |

**Architecture-specific:**

| الفايل | الدور |
|---|---|
| `arch/*/include/asm/spinlock.h` | الـ `arch_read_lock/arch_write_lock` الحقيقية لكل معالج |
| `arch/*/include/asm/spinlock_types.h` | يعرّف `arch_rwlock_t` |

**Implementation:**

| الفايل | الدور |
|---|---|
| `kernel/locking/spinlock.c` | الـ `_raw_read_lock` وغيرها |
| `kernel/locking/spinlock_debug.c` | الـ debug version للـ lock operations |

---

### ملخص الصورة الكبيرة

الـ `rwlock.h` هو **الواجهة النظيفة** اللي بتخلي أي كود في الـ kernel يقدر يستخدم readers-writer locking بدون ما يفكر في الـ architecture أو الـ debug mode أو SMP vs UP. هو بيجمع كل الـ variants في مكان واحد ويوفّر اسم موحد (مثلاً `read_lock`) بغض النظر عن الـ platform.
## Phase 2: شرح الـ rwlock (Read-Write Lock) Framework

---

### المشكلة اللي بيحلها الـ rwlock

في الـ kernel، في بيانات بتتقرأ كتير وبتتكتب نادراً — زي routing tables، dentries، driver registries. لو استخدمنا **spinlock** عادي، كل thread هيـ spin وينتظر حتى لو كان عايز يـ read بس — وده حرام لأن القراءات مش بتعدّل البيانات.

الـ **rwlock** بيحل ده بفكرة بسيطة:
- **Readers** كتير ممكن يشتغلوا بالتوازي في نفس الوقت.
- **Writer** واحد بس يأخد الـ lock، ولما بياخده محدش يقدر يقرأ أو يكتب.

ده بيقلل الـ contention بشكل كبير في read-heavy workloads.

---

### الفرق بين rwlock والـ spinlock

| Feature | spinlock | rwlock |
|---|---|---|
| Concurrent readers | لأ | آه |
| Concurrent writers | لأ | لأ |
| Lock granularity | exclusive فقط | shared (read) + exclusive (write) |
| Complexity | بسيط | أعلى شوية |
| Context | atomic — no sleep | atomic — no sleep (non-RT) |

---

### Big Picture Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Kernel Subsystems (Consumers)               │
│   networking stack │ VFS dentries │ device drivers │ IRQ tables  │
└─────────────────────────────┬────────────────────────────────────┘
                              │ read_lock() / write_lock()
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                  linux/rwlock.h  — Public API Layer              │
│  read_lock / write_lock / read_lock_irq / write_lock_irqsave ... │
└─────────────────────────────┬────────────────────────────────────┘
                              │ _raw_read_lock() / _raw_write_lock()
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│              linux/spinlock_api_smp.h  — Locking Logic           │
│   handles preempt_disable / IRQ save / BH disable around locks   │
└─────────────────────────────┬────────────────────────────────────┘
                              │ do_raw_read_lock() / do_raw_write_lock()
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                       rwlock_t  (struct)                         │
│   .raw_lock  ──────────────────────────────────────────────────► │
│   [.magic, .owner_cpu, .owner]  (CONFIG_DEBUG_SPINLOCK only)     │
│   [.dep_map]  (CONFIG_DEBUG_LOCK_ALLOC only)                     │
└─────────────────────────────┬────────────────────────────────────┘
                              │ arch_read_lock(&rwlock->raw_lock)
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│              asm/spinlock.h  — Architecture Layer                │
│   arch_rwlock_t  (ARM: 32-bit counter)                           │
│   arch_read_lock / arch_write_lock — inline assembly             │
└──────────────────────────────────────────────────────────────────┘
```

---

### الـ rwlock_t struct بالتفصيل

#### Non-PREEMPT_RT (الحالة العادية)

```c
/* من include/linux/rwlock_types.h */
struct rwlock {
    arch_rwlock_t raw_lock;       /* الـ lock الفعلي — arch-specific */

#ifdef CONFIG_DEBUG_SPINLOCK
    unsigned int  magic;          /* 0xdeaf1eed — للتحقق من corruption */
    unsigned int  owner_cpu;      /* رقم الـ CPU اللي شايل الـ write lock */
    void         *owner;          /* pointer للـ task اللي شايل الـ write lock */
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;   /* lockdep tracking */
#endif
};
typedef struct rwlock rwlock_t;
```

**الـ arch_rwlock_t** على ARM مثلاً بيكون 32-bit integer:
- القيمة `0x00000000` = unlocked
- قيمة موجبة (عدد القراء) = read locked
- قيمة خاصة سالبة (مثلاً `0x80000000`) = write locked

```
  Unlocked:    [  0x00000000  ]
  1 reader:    [  0x00000001  ]
  2 readers:   [  0x00000002  ]
  Write lock:  [  0x80000000  ] ← MSB set, يمنع أي قراءة أو كتابة تانية
```

#### مع PREEMPT_RT (الـ Real-Time kernel)

```c
/* من include/linux/rwlock_types.h — RT path */
struct rwlock {
    struct rwbase_rt  rwbase;   /* wraps rt_mutex + atomic reader count */
    atomic_t          readers;
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};
```

```c
/* من include/linux/rwbase_rt.h */
struct rwbase_rt {
    atomic_t         readers;       /* READER_BIAS = (1U<<31) when no readers */
    struct rt_mutex_base rtmutex;   /* للـ write locking */
};

#define READER_BIAS  (1U << 31)   /* القيمة الابتدائية — كل قراءة بتطرح منها */
#define WRITER_BIAS  (1U << 30)   /* لما writer يأخد الـ lock */
```

على RT، الـ rwlock بيبقى مبني فوق **rt_mutex** — وده لأن الـ RT kernel محتاج priority inheritance علشان يتجنب priority inversion. ده موضوع تاني بنسميه **Priority Inheritance** — باختصار: الـ task اللي شايل الـ lock بياخد أعلى priority من كل اللي بينتظروه.

---

### طبقات الـ API — من فوق لتحت

```
read_lock(lock)
    │
    ▼
_raw_read_lock(lock)           ← inline في spinlock_api_smp.h
    │  preempt_disable()
    │  do_raw_read_lock(lock)
    ▼
arch_read_lock(&lock->raw_lock)  ← inline assembly في asm/spinlock.h
```

كل variant بيضيف layer من الحماية:

| Variant | بيعمل إيه زيادة |
|---|---|
| `read_lock(lock)` | `preempt_disable` فقط |
| `read_lock_irq(lock)` | `local_irq_disable` + `preempt_disable` |
| `read_lock_bh(lock)` | `local_bh_disable` + `preempt_disable` |
| `read_lock_irqsave(lock, flags)` | save IRQ state + disable + `preempt_disable` |

---

### مثال عملي: routing table في الـ networking stack

```c
/* kernel code pattern */
static DEFINE_RWLOCK(rt_lock);  /* macro يعمل rwlock_t مع __RW_LOCK_UNLOCKED */

/* كل thread بيعمل route lookup */
void route_lookup(struct dst_entry *dst)
{
    read_lock(&rt_lock);        /* ممكن N thread يدخلوا هنا بالتوازي */
    /* ... read routing table ... */
    read_unlock(&rt_lock);
}

/* لما الـ routing table تتغير (نادراً) */
void route_update(struct rtentry *rte)
{
    write_lock(&rt_lock);       /* هنا محدش يقدر يدخل — readers أو writers */
    /* ... modify routing table ... */
    write_unlock(&rt_lock);
}
```

---

### الـ irqsave variants — ليه موجودين؟

**المشكلة:** لو كود في process context أخد `read_lock` وجه interrupt وهو شايل الـ lock، والـ interrupt handler حاول ياخد `write_lock` على نفس الـ lock — **deadlock**.

**الحل:**

```c
unsigned long flags;

/* في process context: */
read_lock_irqsave(&my_lock, flags);   /* disable IRQs + read lock */
/* ... critical section ... */
read_unlock_irqrestore(&my_lock, flags); /* restore IRQs */

/* في interrupt handler: */
read_lock(&my_lock);   /* safe لأن IRQs متوقفة في process context */
```

**الـ typecheck macro:**

```c
#define read_lock_irqsave(lock, flags)          \
    do {                                         \
        typecheck(unsigned long, flags);         /* compile-time type check */ \
        flags = _raw_read_lock_irqsave(lock);   \
    } while (0)
```

الـ `typecheck` بيعمل compile error لو حد حط `int` بدل `unsigned long` في flags — حماية من bugs صعبة الاكتشاف.

---

### فرق مهم: SMP vs UP في الـ irqsave

```c
/* CONFIG_SMP أو CONFIG_DEBUG_SPINLOCK */
#define read_lock_irqsave(lock, flags)          \
    do {                                         \
        typecheck(unsigned long, flags);         \
        flags = _raw_read_lock_irqsave(lock);   /* return value */ \
    } while (0)

/* UP (Uniprocessor) بدون debug */
#define read_lock_irqsave(lock, flags)          \
    do {                                         \
        typecheck(unsigned long, flags);         \
        _raw_read_lock_irqsave(lock, flags);    /* pass by ref */ \
    } while (0)
```

على SMP، الـ function بترجع الـ flags كـ return value. على UP، بتاخد pointer. ده بسبب فرق في الـ calling convention وتحسينات الـ inline assembly على الـ architectures المختلفة.

---

### الـ do_raw_* layer — DEBUG vs Production

```c
/* Production (no DEBUG) */
#define do_raw_read_lock(rwlock) \
    do { \
        __acquire_shared(lock);                      /* sparse annotation */ \
        arch_read_lock(&(rwlock)->raw_lock);         /* real work */ \
    } while (0)

/* Debug (CONFIG_DEBUG_SPINLOCK) */
extern void do_raw_read_lock(rwlock_t *lock) __acquires_shared(lock);
/* → بيتحقق من magic number، من double-locking، من wrong context */
```

في debug mode، الـ `do_raw_*` functions بقت real functions (مش macros) علشان تقدر تضيف checks:
- الـ `magic == RWLOCK_MAGIC` — للتأكد إن الـ struct مش corrupted
- `owner_cpu` — مين اللي شايل الـ write lock
- الـ `lockdep` integration — يتتبع الـ lock ordering

---

### الـ lockdep Integration

لو `CONFIG_DEBUG_LOCK_ALLOC` شغّال، كل `rwlock_t` فيها `struct lockdep_map dep_map`. الـ **lockdep** subsystem بيبني graph من كل الـ lock acquisitions في الـ kernel ويكتشف:
- **Deadlock cycles**: A→B→A
- **Lock order violations**: نفس الـ locks بتتاخد بترتيب مختلف في contexts مختلفة
- **Missing irq annotations**: أخدت lock في process context بدون irqsave وفي interrupt handler بدون علم

الـ `rwlock_init` macro بيـ register الـ lock مع lockdep:

```c
#ifdef CONFIG_DEBUG_SPINLOCK
#define rwlock_init(lock)                           \
do {                                                \
    static struct lock_class_key __key;             /* unique per callsite */ \
    __rwlock_init((lock), #lock, &__key);           /* register with lockdep */ \
} while (0)
#else
#define rwlock_init(lock)                           \
    do { *(lock) = __RW_LOCK_UNLOCKED(lock); } while (0)
#endif
```

الـ `lock_class_key` بيكون `static` — يعني كل مكان في الكود بيعمل `rwlock_init` بياخد class مختلف، حتى لو نفس الـ lock type. ده بيخلي lockdep يفرق بين locks مختلفة بناءً على **مكان إنشائها** في الكود.

---

### الـ write_lock_nested

```c
#ifdef CONFIG_DEBUG_LOCK_ALLOC
#define write_lock_nested(lock, subclass) _raw_write_lock_nested(lock, subclass)
#else
#define write_lock_nested(lock, subclass) _raw_write_lock(lock)
#endif
```

بيُستخدم لما كود محتاج ياخد نفس نوع الـ lock على مستويين مختلفين (مثلاً lock على parent node وlock على child node في tree). الـ `subclass` بيقول لـ lockdep "أنا عارف إن ده نفس النوع، بس ده مستوى أعمق — مش deadlock".

---

### الـ rwlock_is_contended

```c
#ifdef arch_rwlock_is_contended
#define rwlock_is_contended(lock) \
    arch_rwlock_is_contended(&(lock)->raw_lock)
#else
#define rwlock_is_contended(lock) ((void)(lock), 0)
#endif
```

بعض الـ architectures بتعرف تكتشف لو في threads تانية بتنتظر الـ lock. لو مش supported، الـ macro بترجع `0` دايماً (مش contended). ده بيُستخدم في بعض الأماكن للـ optimistic locking — "لو مش في حد بيستنى، متحتاجش تعمل memory barrier إضافي".

---

### التشابه الحقيقي — مكتبة جامعية

تخيل **مكتبة جامعية** فيها كتاب مرجعي واحد مهم:

| المكتبة | الـ rwlock |
|---|---|
| طالب بيقرأ الكتاب | `read_lock` — ممكن أكتر من طالب يقرأ في نفس الوقت |
| موظف بيحدّث محتوى الكتاب | `write_lock` — لازم يخلي كل الطلبة يمشوا الأول |
| باب المكتبة | الـ `arch_rwlock_t` — الميكانيزم الفعلي |
| قوانين المكتبة المكتوبة | `rwlock.h` — الـ API |
| أمن المكتبة | الـ debug fields (magic, owner_cpu) |
| سجل الدخول | `lockdep_map dep_map` |
| قفل الطوارئ (مانع الحريق يُقفل الباب) | `write_lock_irqsave` — بيوقف IRQs |
| باب للطلاب الـ VIP (real-time priority) | RT path مع `rt_mutex` + priority inheritance |

---

### ملخص: إيه اللي rwlock بيمتلكه وإيه اللي بيفوّضه

```
                    ┌─────────────────────────────┐
                    │  rwlock Framework يمتلك:    │
                    │  • API موحد (read/write_lock)│
                    │  • IRQ/BH/preempt management │
                    │  • Debug & lockdep support   │
                    │  • RT vs non-RT abstraction  │
                    └──────────────┬──────────────┘
                                   │ يفوّض إلى:
           ┌───────────────────────┼────────────────────────┐
           ▼                       ▼                        ▼
  ┌────────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │ arch layer     │    │ lockdep subsystem│    │ rt_mutex (RT)    │
  │ arch_read_lock │    │ lock ordering,   │    │ priority inher.  │
  │ arch_write_lock│    │ deadlock detect  │    │ sleep-capable    │
  │ (inline asm)   │    │                  │    │ wait queues      │
  └────────────────┘    └──────────────────┘    └──────────────────┘
```

**الـ rwlock framework بيمتلك:**
- الـ API الموحد وضمان الـ portability
- إدارة الـ preemption والـ interrupts حوالين الـ lock
- الـ debug infrastructure (magic, owner tracking, lockdep)
- الـ abstraction بين RT و non-RT kernel

**بيفوّض للـ arch layer:**
- الـ atomic operations الفعلية على الـ hardware register
- الـ memory barriers المناسبة لكل architecture
- التحقق من الـ contention على مستوى الـ hardware
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags — Cheatsheet

| Option / Macro | الأثر |
|---|---|
| `CONFIG_DEBUG_SPINLOCK` | يفعّل الـ debug versions للـ `do_raw_read/write_lock` — بتضيف magic number, owner_cpu, owner pointer للـ struct |
| `CONFIG_DEBUG_LOCK_ALLOC` | يفعّل الـ `lockdep_map` داخل الـ `rwlock_t` — بيتتبع lock ordering violations |
| `CONFIG_SMP` | بيغير سلوك الـ `read/write_lock_irqsave` — على SMP بترجع flags كـ return value، على UP بتاخدها كـ argument |
| `CONFIG_PREEMPT_RT` | بيغير الـ `rwlock_t` كليًا — بيستخدم `rwbase_rt` + `atomic_t readers` بدل `arch_rwlock_t` |
| `arch_rwlock_is_contended` | macro اختياري من الـ architecture — لو مش معرّف، `rwlock_is_contended` بترجع 0 دايمًا |
| `RWLOCK_MAGIC` | القيمة `0xdeaf1eed` — بتتحقق منها عند كل lock/unlock في debug mode |
| `__RW_LOCK_UNLOCKED(name)` | الـ static initializer — بيحط `raw_lock = __ARCH_RW_LOCK_UNLOCKED` وبيصفّر الباقي |
| `DEFINE_RWLOCK(x)` | يعرّف وييnitialize الـ `rwlock_t` statically في نفس السطر |

---

### الـ Structs المهمة

#### `rwlock_t` (alias لـ `struct rwlock`)

**الغرض:** الـ reader-writer lock الأساسي في الـ kernel — بيسمح لـ multiple readers بالوصول المتزامن، بس writer واحد بس في وقت واحد.

**الـ fields (non-RT path):**

| Field | النوع | الوصف |
|---|---|---|
| `raw_lock` | `arch_rwlock_t` | الـ actual hardware lock — architecture-specific، ممكن يكون `unsigned int` أو `u64` حسب الـ arch |
| `magic` | `unsigned int` | فقط لو `CONFIG_DEBUG_SPINLOCK` — القيمة `0xdeaf1eed`، بتكتشف الـ corruption |
| `owner_cpu` | `unsigned int` | فقط لو `CONFIG_DEBUG_SPINLOCK` — الـ CPU اللي حاليًا عنده الـ write lock |
| `owner` | `void *` | فقط لو `CONFIG_DEBUG_SPINLOCK` — pointer للـ `task_struct` صاحب الـ write lock |
| `dep_map` | `struct lockdep_map` | فقط لو `CONFIG_DEBUG_LOCK_ALLOC` — بيسجّل الـ lock في الـ lockdep graph |

**الـ fields (RT path — `CONFIG_PREEMPT_RT`):**

| Field | النوع | الوصف |
|---|---|---|
| `rwbase` | `struct rwbase_rt` | الـ base RT lock — بيحتوي على `rtmutex` داخليًا |
| `readers` | `atomic_t` | عداد الـ readers الحاليين — atomic لأن الـ RT kernel بيسمح بـ preemption |
| `dep_map` | `struct lockdep_map` | نفس الـ lockdep tracking |

#### `arch_rwlock_t`

**الغرض:** الـ low-level architecture-specific lock — الـ `rwlock_t` بيحتويه كـ `raw_lock`. كل architecture بتعرّفه بشكل مختلف في `asm/spinlock_types.h`.

مثلًا على x86:
```c
typedef struct {
    int lock;  /* positive = reader count, -1 = writer locked */
} arch_rwlock_t;
```

#### `struct lockdep_map`

**الغرض:** بيُستخدم بواسطة الـ `lockdep` subsystem لتتبع الـ lock dependencies وكشف الـ deadlocks. الـ `rwlock_t` بيحتويه كـ `dep_map`.

**الـ fields الأساسية:**

| Field | الوصف |
|---|---|
| `name` | اسم الـ lock — الـ string اللي بتشوفه في الـ lockdep warnings |
| `wait_type_inner` | نوع الانتظار — `LD_WAIT_CONFIG` للـ rwlocks |
| `class_cache` | cached pointer للـ `lock_class` الخاصة بالـ lock |

#### `struct lock_class_key`

**الغرض:** بيُستخدم كـ static key لتمييز كل lock instance في الـ lockdep. الـ `rwlock_init` macro بيعرّف واحد static لكل lock.

```c
/* مثال: عند rwlock_init في debug mode */
static struct lock_class_key __key;
__rwlock_init(lock, #lock, &__key);
```

---

### علاقات الـ Structs — ASCII Diagram

```
rwlock_t
┌─────────────────────────────────────────┐
│  raw_lock : arch_rwlock_t               │◄── الـ actual atomic state
│  ┌────────────────────┐                 │
│  │  lock: int         │  (x86 example)  │
│  └────────────────────┘                 │
│                                         │
│  [DEBUG_SPINLOCK only]                  │
│  magic    : unsigned int (0xdeaf1eed)   │
│  owner_cpu: unsigned int                │
│  owner    : void * ──────────────────────────► task_struct (current writer)
│                                         │
│  [DEBUG_LOCK_ALLOC only]                │
│  dep_map  : struct lockdep_map          │
│  ┌──────────────────────────────┐       │
│  │  name: const char *          │       │
│  │  class_cache: lock_class *[2]│◄──── lockdep internal graph
│  │  wait_type_inner: u8         │       │
│  └──────────────────────────────┘       │
└─────────────────────────────────────────┘
         ▲
         │ bيستخدمها
         │
┌─────────────────────┐
│ lock_class_key      │ (static per-lock-site)
│  [opaque to user]   │──► lockdep يحوّلها لـ lock_class
└─────────────────────┘
```

**الـ RT path:**
```
rwlock_t (PREEMPT_RT)
┌────────────────────────────────┐
│  rwbase : struct rwbase_rt     │
│  ┌──────────────────────────┐  │
│  │  rtmutex : rt_mutex_base │  │◄── priority-inheritance aware mutex
│  │  readers_lock : spinlock │  │
│  └──────────────────────────┘  │
│  readers : atomic_t            │◄── عداد الـ concurrent readers
│  dep_map : lockdep_map         │
└────────────────────────────────┘
```

---

### Lifecycle Diagram

```
=== Creation ===

Static:
  DEFINE_RWLOCK(my_lock)
       │
       ▼
  my_lock = __RW_LOCK_UNLOCKED(my_lock)
       │
       ├── raw_lock = __ARCH_RW_LOCK_UNLOCKED  (arch sets to 0 or RW_LOCK_BIAS)
       ├── magic    = 0xdeaf1eed               [DEBUG only]
       ├── owner    = SPINLOCK_OWNER_INIT       [DEBUG only]
       ├── owner_cpu= -1                        [DEBUG only]
       └── dep_map  = { .name = "my_lock", ... }[LOCK_ALLOC only]

Dynamic:
  rwlock_t my_lock;
  rwlock_init(&my_lock);
       │
       ├── [DEBUG_SPINLOCK=n]  *(lock) = __RW_LOCK_UNLOCKED(lock)
       └── [DEBUG_SPINLOCK=y]  __rwlock_init(lock, "my_lock", &__key)
                                    │
                                    └── lockdep_init_map() + arch init

=== Usage ===

  Read Path:                        Write Path:
  ───────────                       ────────────
  read_lock(&lock)                  write_lock(&lock)
       │                                  │
       ▼                                  ▼
  _raw_read_lock()               _raw_write_lock()
       │                                  │
       ▼                                  ▼
  do_raw_read_lock()             do_raw_write_lock()
       │                                  │
       ▼                                  ▼
  arch_read_lock(&lock->raw_lock) arch_write_lock(&lock->raw_lock)
       │                                  │
       ▼                                  ▼
  [hardware atomic op]           [hardware atomic op — spins until
  increment reader count          reader count == 0, then set -1]
       │                                  │
  [critical section]             [critical section — exclusive]
       │                                  │
       ▼                                  ▼
  read_unlock(&lock)             write_unlock(&lock)
       │                                  │
       ▼                                  ▼
  arch_read_unlock()             arch_write_unlock()

=== Teardown ===
  لا يوجد explicit teardown لـ rwlock_t
  الـ debug mode بيكتشف لو lock اتدمّر وهو locked
```

---

### Call Flow Diagrams

#### `read_lock` — الـ Full Path

```
read_lock(lock)                   [rwlock.h macro]
  │
  └─► _raw_read_lock(lock)        [spinlock_api_smp.h / up.h]
        │
        └─► do_raw_read_lock(lock)
              │
              ├── [DEBUG_SPINLOCK=y]
              │     extern void do_raw_read_lock(rwlock_t*)
              │     → يتحقق magic number
              │     → يسجّل في lockdep
              │     → يستدعي arch_read_lock()
              │
              └── [DEBUG_SPINLOCK=n]
                    __acquire_shared(lock)    [sparse annotation]
                    arch_read_lock(&lock->raw_lock)
                          │
                          └─► [x86] asm: LOCK; incl (%rdi)
                                          spin if negative
```

#### `write_lock_irqsave` — الـ Full Path (SMP)

```
write_lock_irqsave(lock, flags)   [rwlock.h macro]
  │
  ├── typecheck(unsigned long, flags)   [compile-time type check]
  │
  └── flags = _raw_write_lock_irqsave(lock)
                │
                └─► local_irq_save(flags)   [disable IRQs, save flags]
                      │
                      └─► do_raw_write_lock(lock)
                                │
                                └─► arch_write_lock(&lock->raw_lock)
                                          │
                                          └─► spin until all readers done
                                              set lock to -1 (exclusive)
```

#### `write_trylock_irqsave` — Non-blocking path

```
write_trylock_irqsave(lock, flags)
  │
  └─► _raw_write_trylock_irqsave(lock, &flags)
            │
            ├── local_irq_save(*flags)
            ├── ret = do_raw_write_trylock(lock)
            │         │
            │         └─► arch_write_trylock() → returns 1 if got lock, 0 if not
            │
            ├── if (!ret) local_irq_restore(*flags)   [restore if failed]
            └── return ret
```

#### `rwlock_is_contended` — Contention Detection

```
rwlock_is_contended(lock)
  │
  ├── [arch_rwlock_is_contended defined]
  │     └─► arch_rwlock_is_contended(&lock->raw_lock)
  │               └─► arch-specific check (e.g., waiters counter)
  │
  └── [arch_rwlock_is_contended NOT defined]
        └─► (void)(lock), 0    [always returns 0 — no contention info]
```

#### `rwlock_init` — Debug vs. Release

```
rwlock_init(lock)
  │
  ├── [CONFIG_DEBUG_SPINLOCK=y]
  │     static struct lock_class_key __key;
  │     __rwlock_init(lock, #lock, &__key)
  │           │
  │           ├── memset(lock, ...)
  │           ├── lock->magic = RWLOCK_MAGIC
  │           ├── lock->owner = SPINLOCK_OWNER_INIT
  │           ├── lock->owner_cpu = -1
  │           ├── lock->raw_lock = __ARCH_RW_LOCK_UNLOCKED
  │           └── lockdep_init_map(&lock->dep_map, name, key, 0)
  │
  └── [CONFIG_DEBUG_SPINLOCK=n]
        *(lock) = __RW_LOCK_UNLOCKED(lock)
              └─► direct struct assignment — fast path
```

---

### Locking Strategy

#### الـ rwlock في الـ Kernel — المبدأ الأساسي

الـ **reader-writer lock** بيحل مشكلة الـ spinlock العادي اللي بيمنع concurrent reads — الـ rwlock بيسمح بـ N readers في نفس الوقت، بس writer واحد فقط وهو يمنع كل الـ readers.

#### ما الذي تحميه الـ rwlock؟

```
┌─────────────────────────────────────────────────────────┐
│  rwlock_t                                               │
│  ┌──────────────────┐                                   │
│  │ raw_lock         │ ← يحمي الـ shared data structure │
│  └──────────────────┘                                   │
│                                                         │
│  read_lock()  → يحمي: القراءة من الـ shared resource   │
│  write_lock() → يحمي: الكتابة على الـ shared resource  │
└─────────────────────────────────────────────────────────┘
```

#### قواعد الـ Lock Ordering وقواعد الاستخدام

| الموقف | الـ API المناسب | السبب |
|---|---|---|
| process context فقط | `read_lock` / `write_lock` | لا interrupts، لا bottom halves |
| process context + bottom half | `read_lock_bh` / `write_lock_bh` | بيعمل `local_bh_disable()` |
| process context + interrupt handler | `read_lock_irqsave` / `write_lock_irqsave` | بيعمل `local_irq_save()` |
| داخل interrupt handler | `read_lock` / `write_lock` | الـ IRQs أصلًا disabled |
| non-blocking attempt | `read_trylock` / `write_trylock` | بيرجع 0 لو فشل — لا spinning |

#### قاعدة مهمة: لا تحمل rwlock وتنام

الـ `rwlock_t` بتعمل **busy-wait (spinning)**. لو احتجت lock وممكن تنام، استخدم `rwsem` بدلًا منها.

```c
/* WRONG — ممنوع تنام وأنت شايل rwlock */
read_lock(&my_lock);
msleep(100);           /* DEADLOCK / BUG */
read_unlock(&my_lock);

/* CORRECT — للـ sleeping context */
down_read(&my_rwsem);
msleep(100);           /* OK */
up_read(&my_rwsem);
```

#### الـ Lock Ordering مع الـ `lockdep_map`

لما `CONFIG_DEBUG_LOCK_ALLOC=y`، الـ lockdep بيبني graph للـ lock acquisition order. كل `rwlock_t` عنده:

```
lock_class_key (static, per call-site)
       │
       ▼
lock_class (runtime, per lock type)
       │
       ├── held_locks stack (per-CPU)
       └── dependency graph
              │
              └─► lockdep يكتشف circular dependencies = deadlock risk
```

مثال على lock ordering violation:

```c
/* CPU 0 */                    /* CPU 1 */
read_lock(&lock_A);            write_lock(&lock_B);
write_lock(&lock_B);           read_lock(&lock_A);   /* DEADLOCK! */
```

الـ lockdep بيكتشف ده ويطبع warning قبل ما يحصل deadlock حقيقي.

#### الـ write_lock_nested — للـ Nested Locking

```c
/* لما محتاج تاخد نفس الـ lock مرتين (subclass مختلفة) */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
  write_lock_nested(lock, subclass)  /* بيخبر lockdep إن ده nested intentional */
#else
  write_lock_nested(lock, subclass)  /* بيتحول لـ _raw_write_lock(lock) */
#endif
```

#### الـ rwlock_needbreak — للـ Preemptible Kernels

```c
/* في spinloop طويلة — بتتحقق لو في writer ينتظر */
static inline int rwlock_needbreak(rwlock_t *lock)
{
    if (!preempt_model_preemptible())
        return 0;                        /* non-preemptible: لا تكسر */
    return rwlock_is_contended(lock);    /* لو في waiter: كسر الـ loop */
}
```

#### ملخص الـ Lock Layers

```
User API              Intermediate           Hardware
─────────────         ─────────────          ─────────
read_lock()     ──►  _raw_read_lock()  ──►  do_raw_read_lock()  ──►  arch_read_lock()
write_lock()    ──►  _raw_write_lock() ──►  do_raw_write_lock() ──►  arch_write_lock()
read_trylock()  ──►  _raw_read_trylock()──► do_raw_read_trylock()──► arch_read_trylock()

كل طبقة بتضيف:
  - User API:       typecheck() + irq/bh handling
  - Intermediate:   lockdep annotations + preempt_disable/enable
  - do_raw_*:       debug checks (magic, owner)
  - arch_*:         actual CPU instructions (LOCK prefix, LL/SC, etc.)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Initialization

| Macro / Function | الوصف المختصر |
|---|---|
| `rwlock_init(lock)` | تهيئة الـ rwlock قبل الاستخدام |
| `DEFINE_RWLOCK(x)` | تعريف وتهيئة static rwlock |
| `__RW_LOCK_UNLOCKED(lockname)` | initializer expression للـ rwlock |

#### Raw Layer (do_raw_*)

| Function | الوصف المختصر |
|---|---|
| `do_raw_read_lock(lock)` | اكتساب read lock على المستوى الـ raw |
| `do_raw_read_trylock(lock)` | محاولة اكتساب read lock بدون blocking |
| `do_raw_read_unlock(lock)` | تحرير read lock على المستوى الـ raw |
| `do_raw_write_lock(lock)` | اكتساب write lock على المستوى الـ raw |
| `do_raw_write_trylock(lock)` | محاولة اكتساب write lock بدون blocking |
| `do_raw_write_unlock(lock)` | تحرير write lock على المستوى الـ raw |

#### Public Read Lock API

| Macro | الوصف المختصر |
|---|---|
| `read_lock(lock)` | اكتساب read lock |
| `read_trylock(lock)` | محاولة اكتساب read lock |
| `read_lock_irq(lock)` | اكتساب read lock مع disable IRQs |
| `read_lock_bh(lock)` | اكتساب read lock مع disable BH |
| `read_lock_irqsave(lock, flags)` | اكتساب read lock مع حفظ IRQ state |
| `read_unlock(lock)` | تحرير read lock |
| `read_unlock_irq(lock)` | تحرير read lock مع enable IRQs |
| `read_unlock_bh(lock)` | تحرير read lock مع enable BH |
| `read_unlock_irqrestore(lock, flags)` | تحرير read lock مع استعادة IRQ state |

#### Public Write Lock API

| Macro | الوصف المختصر |
|---|---|
| `write_lock(lock)` | اكتساب write lock |
| `write_trylock(lock)` | محاولة اكتساب write lock |
| `write_lock_nested(lock, subclass)` | اكتساب write lock مع lockdep nesting |
| `write_lock_irq(lock)` | اكتساب write lock مع disable IRQs |
| `write_lock_bh(lock)` | اكتساب write lock مع disable BH |
| `write_lock_irqsave(lock, flags)` | اكتساب write lock مع حفظ IRQ state |
| `write_unlock(lock)` | تحرير write lock |
| `write_unlock_irq(lock)` | تحرير write lock مع enable IRQs |
| `write_unlock_bh(lock)` | تحرير write lock مع enable BH |
| `write_unlock_irqrestore(lock, flags)` | تحرير write lock مع استعادة IRQ state |
| `write_trylock_irqsave(lock, flags)` | محاولة اكتساب write lock مع حفظ IRQ state |

#### Utility

| Macro | الوصف المختصر |
|---|---|
| `rwlock_is_contended(lock)` | فحص وجود waiters على الـ rwlock |
| `rwlock_needbreak(lock)` | فحص إذا كان يجب كسر الـ critical section |

---

### المجموعة 1: Initialization

الـ rwlock لازم يتهيئ قبل أي استخدام — ده مش اختياري. الـ initialization بتضمن إن الـ `raw_lock` الـ arch-specific يبقى في الـ unlocked state، وفي debug builds بتضيف magic value وتسجيل في الـ lockdep subsystem.

---

#### `rwlock_init(lock)`

```c
/* CONFIG_DEBUG_SPINLOCK=y */
#define rwlock_init(lock)					\
do {								\
	static struct lock_class_key __key;			\
	__rwlock_init((lock), #lock, &__key);			\
} while (0)

/* CONFIG_DEBUG_SPINLOCK=n */
#define rwlock_init(lock)					\
	do { *(lock) = __RW_LOCK_UNLOCKED(lock); } while (0)
```

**الـ rwlock_init** هو entry point الرئيسي لتهيئة أي rwlock. في الـ non-debug path بيعمل simple assignment للـ initializer struct. في الـ debug path بيستدعي `__rwlock_init` اللي بتسجل الـ lock في نظام الـ lockdep وبتعمل validation.

**Parameters:**
- `lock` — pointer لـ `rwlock_t` المطلوب تهيئته.

**Return value:** لا يُرجع قيمة (macro يعمل statement).

**Key details:**
- الـ `static struct lock_class_key __key` بتبقى unique per call-site، ده هو الـ mechanism اللي lockdep بيستخدمه يميز بين الـ lock instances المختلفة.
- في الـ non-debug path التهيئة atomic من ناحية الـ compiler لأنها single assignment.
- **لازم تتسمى قبل أول استخدام** — استخدام rwlock من غير init undefined behavior.

**Caller context:** initialization code فقط، process context.

---

#### `__RW_LOCK_UNLOCKED(lockname)`

```c
/* CONFIG_DEBUG_SPINLOCK=n */
#define __RW_LOCK_UNLOCKED(lockname) \
	(rwlock_t) { .raw_lock = __ARCH_RW_LOCK_UNLOCKED, \
	             RW_DEP_MAP_INIT(lockname) }

/* CONFIG_DEBUG_SPINLOCK=y */
#define __RW_LOCK_UNLOCKED(lockname)					\
	(rwlock_t) { .raw_lock = __ARCH_RW_LOCK_UNLOCKED,	\
	             .magic = RWLOCK_MAGIC,			\
	             .owner = SPINLOCK_OWNER_INIT,		\
	             .owner_cpu = -1,				\
	             RW_DEP_MAP_INIT(lockname) }
```

**الـ __RW_LOCK_UNLOCKED** هو compound literal بيمثل الـ rwlock في الـ unlocked state. بيُستخدم في static initialization أو تهيئة stack-allocated locks. الـ `RWLOCK_MAGIC = 0xdeaf1eed` بيُستخدم في debug builds كـ sanity check.

**Parameters:**
- `lockname` — الاسم (لـ lockdep فقط، مش runtime identifier).

**Key details:**
- الـ `__ARCH_RW_LOCK_UNLOCKED` معرف في `asm/spinlock_types.h` ومختلف per-arch.
- الـ `RW_DEP_MAP_INIT` بتتوسع لـ empty في غياب `CONFIG_DEBUG_LOCK_ALLOC`.

---

#### `DEFINE_RWLOCK(x)`

```c
#define DEFINE_RWLOCK(x)	rwlock_t x = __RW_LOCK_UNLOCKED(x)
```

بيعمل declaration وinitialization في نفس السطر لـ statically allocated rwlock. مفيد للـ module-level أو file-scope locks.

---

### المجموعة 2: Raw Layer (do_raw_*)

الـ raw layer هو الطبقة اللي بتتكلم مباشرة مع الـ arch-specific implementation. ده الـ boundary بين الـ generic kernel locking API والـ hardware-level atomic operations.

في `CONFIG_DEBUG_SPINLOCK=y` بيبقوا functions خارجية بتعمل validation إضافية. في الـ non-debug path بيبقوا macros بتستدعي الـ arch\_* functions مباشرة.

---

#### `do_raw_read_lock(rwlock)` / `do_raw_read_unlock(rwlock)`

```c
/* DEBUG=y: extern */
extern void do_raw_read_lock(rwlock_t *lock) __acquires_shared(lock);
extern void do_raw_read_unlock(rwlock_t *lock) __releases_shared(lock);

/* DEBUG=n: macros */
#define do_raw_read_lock(rwlock) \
	do { __acquire_shared(lock); arch_read_lock(&(rwlock)->raw_lock); } while (0)
#define do_raw_read_unlock(rwlock) \
	do { arch_read_unlock(&(rwlock)->raw_lock); __release_shared(lock); } while (0)
```

**الـ do_raw_read_lock** بتكتسب الـ read side من الـ rwlock عن طريق الـ arch implementation. ممكن تبلوك لو في write lock محتجز. **الـ do_raw_read_unlock** بتحرر الـ read lock وتعمل الـ memory barrier اللازمة.

**Parameters:**
- `rwlock` — pointer لـ `rwlock_t`.

**Return value:** void.

**Key details:**
- الـ `__acquires_shared` / `__releases_shared` هي annotations لـ sparse checker مش runtime.
- الـ arch_read_lock بتعمل implicit memory barrier على معظم الـ architectures.
- **لا تُستدعى مباشرة من kernel code** — استخدم `read_lock()` وأصحابها.

---

#### `do_raw_read_trylock(rwlock)`

```c
/* DEBUG=y */
extern int do_raw_read_trylock(rwlock_t *lock);

/* DEBUG=n */
#define do_raw_read_trylock(rwlock)	arch_read_trylock(&(rwlock)->raw_lock)
```

محاولة اكتساب الـ read lock بدون blocking. بترجع فوراً سواء نجحت أو لا.

**Return value:** non-zero لو اكتسب الـ lock، صفر لو مش متاح.

---

#### `do_raw_write_lock(rwlock)` / `do_raw_write_unlock(rwlock)`

```c
/* DEBUG=y: extern */
extern void do_raw_write_lock(rwlock_t *lock) __acquires(lock);
extern void do_raw_write_unlock(rwlock_t *lock) __releases(lock);

/* DEBUG=n: macros */
#define do_raw_write_lock(rwlock) \
	do { __acquire(lock); arch_write_lock(&(rwlock)->raw_lock); } while (0)
#define do_raw_write_unlock(rwlock) \
	do { arch_write_unlock(&(rwlock)->raw_lock); __release(lock); } while (0)
```

**الـ do_raw_write_lock** بتكتسب الـ exclusive write lock — بتبلوك لحد ما كل الـ readers والـ writers يخلوا. **الـ do_raw_write_unlock** بتحرر الـ write lock وتسمح للـ waiters يكملوا.

**Key details:**
- الـ write lock exclusive تماماً — مفيش reads أو writes تانية ممكنة أثناء احتجازه.
- الـ annotations `__acquires(lock)` / `__releases(lock)` (مش shared) تعكس الـ exclusive nature.
- على SMP الـ write_lock بتشوف إن كل الـ CPU caches تعمل invalidation للـ cache lines المتعلقة بالـ lock region.

---

#### `do_raw_write_trylock(rwlock)`

```c
/* DEBUG=y */
extern int do_raw_write_trylock(rwlock_t *lock);

/* DEBUG=n */
#define do_raw_write_trylock(rwlock)	arch_write_trylock(&(rwlock)->raw_lock)
```

محاولة اكتساب الـ write lock. بتنجح بس لو مفيش readers أو writers تانية.

**Return value:** non-zero لو نجح، صفر لو مش متاح.

---

### المجموعة 3: Public Read Lock API

الـ public API هي الطبقة اللي الـ kernel subsystems بتستخدمها. كل macro فيها بتعمل wrap لـ `_raw_read_*` function اللي موجودة في `spinlock_api_smp.h` أو `spinlock_api_up.h` حسب الـ config.

---

#### `read_lock(lock)` / `read_unlock(lock)`

```c
#define read_lock(lock)		_raw_read_lock(lock)
#define read_unlock(lock)	_raw_read_unlock(lock)
```

أبسط شكل للـ read locking. لا بيعمل disable للـ interrupts، لا للـ BH. مناسب لو الـ critical section مش بتتعمل منها interrupt handler أو softirq.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.

**Return value:** void.

**Key details:**
- مش آمنة لو الـ shared data ممكن تتعدل من interrupt context — استخدم `read_lock_irqsave` في الحالة دي.
- على UP non-debug builds ممكن تبقى nop.

**Caller context:** process context أو interrupt-disabled context.

---

#### `read_trylock(lock)`

```c
#define read_trylock(lock)	_raw_read_trylock(lock)
```

محاولة اكتساب الـ read lock بدون انتظار. مفيدة في الـ polling loops أو لما الـ caller عنده fallback path.

**Return value:** non-zero لو اكتسب الـ lock، صفر لو مش متاح.

**Key details:**
- بتكون useful لو عندك read-heavy code وعايز تتجنب الـ contention path.
- مش بتعمل disable IRQs — لو في interrupt handler ممكن يمسك نفس الـ lock استخدم `read_lock_irqsave`.

---

#### `read_lock_irq(lock)` / `read_unlock_irq(lock)`

```c
#define read_lock_irq(lock)		_raw_read_lock_irq(lock)
#define read_unlock_irq(lock)		_raw_read_unlock_irq(lock)
```

بتعمل lock مع disable الـ local IRQs. **الـ read_unlock_irq** بتعمل re-enable للـ IRQs بعد التحرير.

**Key details:**
- بتفترض إن الـ IRQs كانت enabled قبل الاستدعاء — لو مش متأكد استخدم `irqsave`.
- مناسبة لو عارف بالتأكيد إن الكود بيشتغل من process context مع enabled IRQs.

**Caller context:** process context بس، مع IRQs enabled قبل الاستدعاء.

---

#### `read_lock_bh(lock)` / `read_unlock_bh(lock)`

```c
#define read_lock_bh(lock)		_raw_read_lock_bh(lock)
#define read_unlock_bh(lock)		_raw_read_unlock_bh(lock)
```

بتعمل lock مع disable الـ bottom halves (softirqs). مناسبة لو الـ data المشتركة ممكن تتعدل من softirq context (مثلاً network Rx path أو timer callbacks).

**Key details:**
- الـ BH disable بيحمي من الـ softirqs بس — مش من الـ hardirqs.
- الـ `local_bh_disable()` / `local_bh_enable()` بيتعملوا داخل `_raw_read_lock_bh`.

**Caller context:** process context أو hardirq context.

---

#### `read_lock_irqsave(lock, flags)` / `read_unlock_irqrestore(lock, flags)`

```c
/* SMP or DEBUG_SPINLOCK */
#define read_lock_irqsave(lock, flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		flags = _raw_read_lock_irqsave(lock);	\
	} while (0)

/* UP */
#define read_lock_irqsave(lock, flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		_raw_read_lock_irqsave(lock, flags);	\
	} while (0)

#define read_unlock_irqrestore(lock, flags)			\
	do {							\
		typecheck(unsigned long, flags);		\
		_raw_read_unlock_irqrestore(lock, flags);	\
	} while (0)
```

**الـ read_lock_irqsave** بتحفظ الـ IRQ flags الحالية وبتعمل disable للـ local IRQs وبتكتسب الـ read lock. **الـ read_unlock_irqrestore** بتحرر الـ lock وترجع الـ IRQ state لما كان عليه.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.
- `flags` — `unsigned long` بيتحفظ فيه الـ IRQ state (مش pointer — الـ macro بيعمل assignment مباشر).

**Return value:** void.

**Key details:**
- الـ `typecheck(unsigned long, flags)` بيضمن إن الـ compiler يطلع warning لو الـ flags مش النوع الصح.
- الفرق بين SMP وUP: في SMP الـ `_raw_read_lock_irqsave` بترجع الـ flags كـ return value. في UP بياخد الـ flags كـ output parameter — ده لأسباب optimization.
- دي الـ safest variant وهي المفضلة في كود مش متأكد من الـ IRQ context.

**Pseudocode flow:**
```
read_lock_irqsave(lock, flags):
    flags = local_irq_save()      // disable IRQs, save EFLAGS/CPSR
    preempt_disable()             // إيه الـ preemption
    _raw_read_lock(lock)          // arch_read_lock on raw_lock
```

**Caller context:** أي context، الأكثر general purpose.

---

### المجموعة 4: Public Write Lock API

الـ write lock exclusive — مفيش أي reader أو writer تاني يقدر يشتغل أثناء احتجازه. ده بيخلي الـ write lock أغلى تكلفة من الـ read lock.

---

#### `write_lock(lock)` / `write_unlock(lock)`

```c
#define write_lock(lock)	_raw_write_lock(lock)
#define write_unlock(lock)	_raw_write_unlock(lock)
```

اكتساب وتحرير الـ exclusive write lock بدون IRQ manipulation. الـ write_lock بتبلوك لحد ما كل الـ readers والـ writers الحاليين يخلوا.

**Key details:**
- على SMP الـ write_lock بتبقى expensive لأنها لازم تنتظر كل الـ readers يخلصوا.
- من النادر استخدام `write_lock` من غير IRQ protection في الـ kernel الحديث.

---

#### `write_trylock(lock)`

```c
#define write_trylock(lock)	_raw_write_trylock(lock)
```

محاولة اكتساب الـ write lock. بتنجح بس لو الـ lock حر تماماً (مفيش readers ولا writers).

**Return value:** non-zero لو نجح، صفر لو فيه أي holder.

---

#### `write_lock_nested(lock, subclass)`

```c
/* CONFIG_DEBUG_LOCK_ALLOC=y */
#define write_lock_nested(lock, subclass)	_raw_write_lock_nested(lock, subclass)

/* CONFIG_DEBUG_LOCK_ALLOC=n */
#define write_lock_nested(lock, subclass)	_raw_write_lock(lock)
```

بتكتسب الـ write lock مع تحديد الـ lockdep subclass. مهمة لما تحتاج تمسك نفس الـ lock type على مستويات مختلفة من الـ hierarchy (مثلاً في trees أو lists متداخلة).

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.
- `subclass` — رقم integer يمثل الـ nesting level (0 = top level).

**Key details:**
- في الـ non-debug builds الـ subclass بيتجاهل تماماً وبيبقى nop.
- بدون `write_lock_nested` الـ lockdep هيطلع false positive warning لما تحتجز نفس الـ lock class مرتين.

---

#### `write_lock_irq(lock)` / `write_unlock_irq(lock)`

```c
#define write_lock_irq(lock)		_raw_write_lock_irq(lock)
#define write_unlock_irq(lock)		_raw_write_unlock_irq(lock)
```

اكتساب وتحرير الـ write lock مع disable/enable الـ local IRQs.

**Key details:**
- نفس القيود بتاعة `read_lock_irq` — بتفترض IRQs كانت enabled.

---

#### `write_lock_bh(lock)` / `write_unlock_bh(lock)`

```c
#define write_lock_bh(lock)		_raw_write_lock_bh(lock)
#define write_unlock_bh(lock)		_raw_write_unlock_bh(lock)
```

اكتساب وتحرير الـ write lock مع BH protection.

---

#### `write_lock_irqsave(lock, flags)` / `write_unlock_irqrestore(lock, flags)`

```c
/* SMP or DEBUG_SPINLOCK */
#define write_lock_irqsave(lock, flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		flags = _raw_write_lock_irqsave(lock);	\
	} while (0)

/* UP */
#define write_lock_irqsave(lock, flags)			\
	do {						\
		typecheck(unsigned long, flags);	\
		_raw_write_lock_irqsave(lock, flags);	\
	} while (0)

#define write_unlock_irqrestore(lock, flags)		\
	do {						\
		typecheck(unsigned long, flags);	\
		_raw_write_unlock_irqrestore(lock, flags);	\
	} while (0)
```

نفس الـ pattern بتاع `read_lock_irqsave` لكن للـ write side. الأكثر شيوعاً في الـ kernel لأنها الأكثر safety.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.
- `flags` — `unsigned long` لحفظ الـ IRQ state.

**Pseudocode flow:**
```
write_lock_irqsave(lock, flags):
    flags = local_irq_save()        // disable IRQs + save state
    preempt_disable()
    arch_write_lock(&lock->raw_lock) // spin until all readers gone
    // memory barrier implied
```

---

#### `write_trylock_irqsave(lock, flags)`

```c
#define write_trylock_irqsave(lock, flags) _raw_write_trylock_irqsave(lock, &(flags))
```

بتحاول تكتسب الـ write lock مع حفظ الـ IRQ state. لو فشلت، الـ IRQs بترجع لحالتها الأصلية.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.
- `flags` — `unsigned long` للحفظ (ملاحظة: بيتبعت كـ `&flags` للـ underlying function).

**Return value:** non-zero لو نجح في اكتساب الـ lock.

**Key details:**
- ده الوحيد في السلسلة اللي بياخد الـ flags كـ regular variable مش pointer — الـ macro بيعمل `&(flags)` تلقائياً.

---

### المجموعة 5: Utility / Status

---

#### `rwlock_is_contended(lock)`

```c
#ifdef arch_rwlock_is_contended
#define rwlock_is_contended(lock) \
	arch_rwlock_is_contended(&(lock)->raw_lock)
#else
#define rwlock_is_contended(lock)	((void)(lock), 0)
#endif
```

بتفحص إذا كان فيه threads تانية بتنتظر على الـ lock. الـ arch_rwlock_is_contended مش موجودة على كل الـ architectures — لو مش موجودة بترجع صفر دايماً.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.

**Return value:** non-zero لو في waiters، صفر لو مفيش أو الـ arch مش بيدعم الـ detection.

**Key details:**
- بتُستخدم أساساً في `rwlock_needbreak()` — بتساعد الـ scheduler يقرر هل يكسر الـ critical section أم لا.
- **مش guarantee** — ممكن ترجع false negative على بعض الـ architectures.

---

#### `rwlock_needbreak(lock)` — من `spinlock.h`

```c
static inline int rwlock_needbreak(rwlock_t *lock)
{
	if (!preempt_model_preemptible())
		return 0;

	return rwlock_is_contended(lock);
}
```

بتفحص إذا كان يجب على الـ lock holder إنه يحرر الـ lock مبكراً عشان thread تاني ينتظر. بترجع صفر على الـ non-preemptible kernels لأن الـ latency issue مش موجودة هناك.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.

**Return value:** non-zero لو يُنصح بكسر الـ critical section.

**Key details:**
- بتُستخدم في الـ long-running read critical sections زي filesystem traversal.
- مثال استخدام:

```c
read_lock(&my_lock);
while (has_more_work()) {
    do_some_work();
    if (rwlock_needbreak(&my_lock)) {
        read_unlock(&my_lock);
        cond_resched();            // يدي فرصة للـ waiters
        read_lock(&my_lock);
    }
}
read_unlock(&my_lock);
```

---

### المجموعة 6: Debug Layer Functions

دي موجودة بس مع `CONFIG_DEBUG_SPINLOCK=y` وبتكون external functions في `kernel/locking/spinlock_debug.c`.

---

#### `__rwlock_init(lock, name, key)`

```c
extern void __rwlock_init(rwlock_t *lock, const char *name,
                          struct lock_class_key *key);
```

الـ implementation الفعلي للـ initialization في debug mode. بتعمل:
1. تسجيل الـ lock في الـ lockdep subsystem بالـ class key.
2. ضبط الـ `magic = RWLOCK_MAGIC (0xdeaf1eed)`.
3. ضبط `owner = SPINLOCK_OWNER_INIT` و `owner_cpu = -1`.
4. تهيئة الـ `raw_lock` بـ `__ARCH_RW_LOCK_UNLOCKED`.

**Parameters:**
- `lock` — pointer لـ `rwlock_t`.
- `name` — string literal (اسم المتغير) للـ lockdep.
- `key` — unique key per call-site للـ lockdep class tracking.

**Key details:**
- الـ `lock_class_key` بيكون `static` في الـ call site عشان يكون unique per code location.
- لو رأيت lockdep warning فيه اسم الـ lock ده هو اللي جاي من هنا.

---

### ملاحظات معمارية هامة

#### الفرق بين SMP و UP في الـ irqsave macros

```
SMP:  flags = _raw_read_lock_irqsave(lock)      // return value
UP:   _raw_read_lock_irqsave(lock, flags)        // output param
```

ده design decision معروف في الـ kernel — على SMP بيبقى أكفأ إن الـ flags يتبعت كـ return value. على UP ممكن يكون الـ function نفسه مختلف تماماً أو nop.

#### طبقات الـ Abstraction

```
                 User Code
                    │
         read_lock() / write_lock()          ← rwlock.h (هنا)
                    │
         _raw_read_lock() / _raw_write_lock() ← spinlock_api_smp.h
                    │
         do_raw_read_lock() / do_raw_write_lock() ← rwlock.h (هنا)
                    │
         arch_read_lock() / arch_write_lock() ← asm/spinlock.h
                    │
              Hardware / LL-SC / XCHG
```

#### الـ PREEMPT_RT Impact

الـ `rwlock.h` بيتضمن فقط لما `CONFIG_PREEMPT_RT=n`. في الـ RT kernel الـ rwlock_t بيتحول لـ `rwbase_rt` اللي based على RT mutexes — ده بيغير الـ semantics جذرياً لأن الـ RT rwlock قابلة للـ sleep.

#### Memory Ordering

كل read lock acquire وwrite lock acquire/release بيعملوا implicit memory barrier عبر الـ arch implementation. على x86 ده بيتعمل عبر الـ LOCK prefix. على ARM عبر الـ LDREX/STREX مع DMB. ده بيضمن إن الـ writes اللي اتعملوا داخل الـ critical section يبقوا visible لكل الـ CPUs التانية بعد release الـ lock.
## Phase 5: دليل الـ Debugging الشامل

الـ `rwlock_t` هو subsystem داخل الـ locking framework في الـ kernel — مفيش debugfs entries خاصة بيه لوحده، لكن الـ lockdep والـ debug configs بتديك كل حاجة محتاجها.

---

### Software Level

#### 1. debugfs Entries

الـ rwlock مش بيعمل entries خاصة بيه في debugfs، لكن الـ **lockdep** بيتكلم عنه من خلال:

```bash
# اعرف كل الـ lock classes اللي اتسجلت (بما فيها rwlock_t)
mount -t debugfs none /sys/kernel/debug
cat /sys/kernel/debug/locking/lockdep

# اعرف الـ chains اللي اكتشفها lockdep
cat /sys/kernel/debug/locking/lockdep_chains

# الـ stats الكاملة للـ lockdep
cat /sys/kernel/debug/locking/lockdep_stats
```

**تفسير الـ output:**

```
all lock classes:     142          # عدد الـ lock classes المسجلة
direct dependencies:  1891
indirect dependencies: 16182
...
lock-class hash chains: 1 [max: 1]
```

لو شفت `lock-class not found` أو class بـ `usage_mask = 0` فده علامة إن الـ lock اتحذف قبل ما اتعمله unlock.

```bash
# شوف الـ held locks على thread معين
cat /proc/<pid>/lockdep
```

#### 2. sysfs Entries

```bash
# عدد الـ lock warnings اللي طلعت
cat /sys/kernel/debug/locking/lockdep_stats | grep "lock-order violations"

# الـ per-CPU stats للـ locking (لو kernel مترجم بـ CONFIG_DEBUG_SPINLOCK)
# مفيش sysfs مباشر للـ rwlock، بس الـ /proc/lock_stat بيغطيه
cat /proc/lock_stat
```

**الـ `/proc/lock_stat` output:**

```
lock_stat version 0.4
--------------------------------------------------------
                              class name    con-bounces    contentions   ...
-------------------------------------------------------
           &rw_lock:R:          0              0              0.00 ns
           &rw_lock:W:          3             12            842.00 ns
```

- **`:R`** = read lock stats
- **`:W`** = write lock stats
- **con-bounces** = كام مرة الـ lock اتحول من thread لتاني وهو contended
- **contentions** = كام مرة حد استنى على الـ lock

```bash
# reset الـ lock stats
echo 0 > /proc/lock_stat
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# روح على ftrace directory
cd /sys/kernel/debug/tracing

# شوف الـ available events اللي ليها علاقة بالـ locking
grep -r "rwlock\|lock" available_events | head -30

# فعّل lock contention events
echo 1 > events/lock/lock_acquire/enable
echo 1 > events/lock/lock_release/enable
echo 1 > events/lock/lock_contended/enable
echo 1 > events/lock/lock_acquired/enable

# ابدأ الـ tracing
echo 1 > tracing_on

# اعمل الـ workload بتاعك هنا...

# وقف وشوف النتيجة
echo 0 > tracing_on
cat trace
```

**مثال على الـ output:**

```
     kworker/0:1-23    [000] ....   123.456: lock_acquire: &my_rwlock (read)
     kworker/0:1-23    [000] ....   123.457: lock_acquired: &my_rwlock (read) [0 ns]
          mydrv-89     [001] ....   123.458: lock_contended: &my_rwlock (write)
```

- لو `lock_contended` بيظهر كتير على نفس الـ lock → performance bottleneck

```bash
# استخدم function_graph tracer لتتبع من بيعمل write_lock ومتى
echo function_graph > current_tracer
echo "do_raw_write_lock" > set_graph_function
echo 1 > tracing_on
```

#### 4. printk والـ Dynamic Debug

```bash
# فعّل الـ dynamic debug لأي module بيستخدم rwlock
echo "module mymodule +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ debug messages في ملف معين
echo "file drivers/mydriver/core.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع stack trace لكل رسالة
echo "file drivers/mydriver/core.c +ps" > /sys/kernel/debug/dynamic_debug/control
```

لو عايز تضيف logging يدوي حوالين الـ rwlock في الـ code:

```c
/* قبل write_lock */
pr_debug("%s: acquiring write lock on %p\n", __func__, &my_rwlock);
write_lock(&my_rwlock);
pr_debug("%s: write lock acquired\n", __func__);

/* لو حصل مشكلة */
if (unlikely(!write_trylock(&my_rwlock))) {
    pr_warn("%s: write_trylock failed, lock is contended\n", __func__);
}
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة | متى تستخدمه |
|---|---|---|
| `CONFIG_DEBUG_SPINLOCK` | يفعّل magic number check + owner tracking في `rwlock_t` | أي تحقيق في corruption |
| `CONFIG_DEBUG_LOCK_ALLOC` | يفعّل `lockdep_map` في كل lock (بما فيه rwlock) | تتبع lock lifecycle |
| `CONFIG_PROVE_LOCKING` | يفعّل lockdep كاملاً (deadlock detection) | **الأهم** — فعّله دايماً في dev |
| `CONFIG_LOCK_STAT` | يفعّل `/proc/lock_stat` وإحصائيات الـ contention | قياس الـ performance |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكشف لو حد عمل sleep وهو شايل rwlock | bugs في interrupt context |
| `CONFIG_LOCKDEP_SUPPORT` | arch support للـ lockdep | prerequisite |
| `CONFIG_LOCKDEP` | الـ lockdep نفسه | prerequisite |
| `CONFIG_DEBUG_MUTEXES` | مكملش للـ rwlock بس مفيد للـ locking بشكل عام | — |
| `CONFIG_PREEMPT_RT` | يحول rwlock لـ rt-aware implementation | real-time systems |
| `CONFIG_DEBUG_PREEMPT` | يكشف preemption violations | bugs في irq context |

**kernel command line لتفعيل debug في runtime:**

```bash
# في GRUB أو bootloader
linux ... loglevel=7 lockdep.nohashpointers=1
```

#### 6. أدوات خاصة بالـ Subsystem

**الـ lockdep هو الأداة الرئيسية للـ rwlock:**

```bash
# شوف الـ lock dependencies graph
cat /sys/kernel/debug/locking/lockdep | head -100

# لو الـ lockdep طلع warning، الـ kernel بيطبع في dmesg
dmesg | grep -A 50 "WARNING: possible circular locking"
dmesg | grep -A 50 "WARNING: lock held when returning to user space"
dmesg | grep -A 30 "BUG: sleeping function called from invalid context"
```

**الـ perf لقياس الـ rwlock contention:**

```bash
# قس الـ lock events
perf lock record -a -- sleep 5
perf lock report

# أو بـ perf stat
perf stat -e "lock:lock_contended,lock:lock_acquire" -a sleep 5
```

**الـ `locktorture` module لاختبار صحة الـ rwlock:**

```bash
# حمّل الـ locktorture module
modprobe locktorture torture_type=rw_lock nfakewriters=4 nfakereaders=4
# اتابع النتيجة
dmesg | grep locktorture
# وقّفه
echo 1 > /sys/kernel/debug/locktorture/torture_runnable  # toggle
rmmod locktorture
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `BUG: spinlock bad magic on CPU#N` | الـ `magic` field في الـ rwlock اتكسر (corruption) | تحقق من buffer overflow قبل الـ lock |
| `BUG: spinlock wrong owner on CPU#N` | حد تاني بيحاول يعمل unlock على lock مش بتاعه | تأكد إن نفس الـ context اللي lock هو اللي unlock |
| `BUG: sleeping function called from invalid context` | عمل sleep وهو شايل rwlock في interrupt context | استخدم `read_lock_irqsave` / `write_lock_irqsave` |
| `WARNING: possible circular locking dependency` | lockdep اكتشف deadlock محتمل | رتّب ترتيب الـ locks أو استخدم `lock_nest_lock` |
| `WARNING: lock held when returning to user space` | رجع لـ userspace وهو لسه شايل rwlock | تأكد من كل exit paths بيعملوا unlock |
| `INFO: rcu_preempt self-detected stall` | الـ write_lock شاغل الـ CPU طول | قلّل مدة الـ critical section |
| `kernel BUG at include/linux/rwlock.h` | assertion فشلت في debug mode | راجع الـ call stack الكامل في dmesg |
| `lockdep: MAX_LOCKDEP_CHAINS too low!` | استنفدنا الـ lockdep chain entries | زود `MAX_LOCKDEP_CHAINS` في kernel أو قلّل الـ lock nesting |
| `WARNING: possible recursive locking detected` | نفس الـ task بيحاول يعمل write_lock مرتين | الـ rwlock مش recursive — استخدم بنية تانية |

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* تحقق إن الـ lock اتعمله init صح قبل الاستخدام */
WARN_ON(lock->magic != RWLOCK_MAGIC);  /* بس لو CONFIG_DEBUG_SPINLOCK */

/* تحقق إنك مش بتعمل write_lock من interrupt handler بدون irqsave */
WARN_ON(in_interrupt() && !irqs_disabled());

/* تحقق إن مفيش حد شايل الـ write lock وبيطلب read lock على نفس الـ lock */
/* ده deadlock محتمل في single-threaded context */
WARN_ON(lock->owner == current);  /* في DEBUG_SPINLOCK builds */

/* لو عايز تعرف مين بيعمل write_lock من فين */
void my_write_lock_debug(rwlock_t *lock)
{
    pr_debug("write_lock from:\n");
    dump_stack();
    write_lock(lock);
}

/* في الـ error handling paths */
if (!write_trylock(&my_rwlock)) {
    WARN(1, "Failed to acquire write lock - possible deadlock\n");
    dump_stack();
    return -EDEADLK;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيتطابق مع الـ Kernel State

الـ rwlock هو software construct بحت — مفيش hardware registers خاصة بيه. لكن الـ contention بيتأثر بالـ:

- **CPU cache coherency**: الـ `raw_lock` field بيتنافس عليه الـ CPUs
- **Memory bus**: الـ atomic operations بتاعة الـ rwlock بتروح على الـ memory bus

```bash
# تحقق من الـ cache misses المرتبطة بالـ lock contention
perf stat -e cache-misses,cache-references,LLC-load-misses -a sleep 5

# شوف الـ CPU affinity للـ threads المتنافسة
taskset -p <pid>
cat /proc/<pid>/status | grep Cpus_allowed
```

#### 2. Register Dump Techniques

الـ rwlock على x86 بيستخدم الـ `arch_rwlock_t` اللي هي عادةً `atomic_t` أو struct بسيطة:

```bash
# شوف قيمة الـ rwlock في الـ memory مباشرة (بتحتاج عنوان الـ symbol)
# أولاً: اعرف العنوان
grep "my_rwlock" /proc/kallsyms
# مثال output: ffffffffc0a12340 d my_rwlock [mymodule]

# اقرأ الـ value بـ /dev/mem (بتحتاج CONFIG_DEVMEM=y)
devmem2 0xffffffffc0a12340 w

# أو بـ crash tool
crash vmlinux /proc/kcore
# ثم في الـ crash prompt:
# rd ffffffffc0a12340
```

**تفسير قيمة الـ `arch_rwlock_t` على x86:**

```
value = 0x01000000  → unlocked (RW_LOCK_BIAS)
value = 0x00000001  → one reader
value = 0x00000000  → writer holds the lock (RW_LOCK_BIAS - RW_LOCK_BIAS = 0)
value = negative    → writer waiting or holding
```

```bash
# بـ gdb على الـ vmcore
gdb vmlinux vmcore
(gdb) p my_rwlock
(gdb) p my_rwlock.raw_lock
```

#### 3. Logic Analyzer / Oscilloscope Tips

الـ rwlock ماله علاقة مباشرة بالـ hardware signals، لكن لو بتشك في timing issues:

- **ILA (Integrated Logic Analyzer)** على FPGA: تراقب الـ memory bus transactions وقت الـ rwlock operations
- **PCIe analyzer**: لو الـ rwlock بيحمي DMA buffers، راقب الـ PCIe transactions
- **JTAG/OpenOCD**: على embedded systems ممكن تعمل hardware breakpoint على عنوان الـ rwlock في الـ memory

```
# عند استخدام JTAG:
# 1. وقّف الـ CPU عند write_lock
# 2. اقرأ قيمة الـ arch_rwlock_t من memory
# 3. تأكد إنها 0 (locked) أو RW_LOCK_BIAS (unlocked)
```

#### 4. Hardware Issues وأنماطها في الـ Kernel Log

| المشكلة | النمط في dmesg | السبب |
|---|---|---|
| Cache coherency bug على NUMA | `perf` بيبان excessive LLC misses + high rwlock latency | الـ lock المتنافس عليه بيتنقل بين الـ NUMA nodes |
| Memory ordering bug على ARM | `BUG: spinlock` أو data corruption بعد unlock | مفيش `dmb` instructions في الـ arch code |
| False sharing | latency عالية رغم إن الـ contention واطي | الـ rwlock_t ومتغير تاني في نفس الـ cache line |
| SMT/Hyperthreading thrashing | latency عالية على cores متجاورين | threads متنافسة على نفس الـ physical core |

```bash
# اكتشف الـ NUMA-related rwlock issues
numastat -m
perf stat -e numa:numanode_hit,numa:numanode_miss -a sleep 5

# شوف الـ cache line sharing
pahole -C rwlock_t vmlinux  # تحقق من الـ struct size والـ alignment
```

#### 5. Device Tree Debugging (للـ embedded systems)

الـ rwlock نفسه مش مرتبط بالـ DT، لكن لو الـ driver بيستخدم rwlock لحماية hardware registers:

```bash
# تحقق من الـ DT node
dtc -I fs /sys/firmware/devicetree/base > /tmp/current.dts
# قارن مع الـ expected DTS

# تحقق من الـ clock والـ power اللي بيأثر على الـ hardware state
# اللي الـ rwlock بيحميه
cat /sys/kernel/debug/clk/clk_summary | grep mydevice
cat /sys/kernel/debug/regulator/regulator_summary
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. تفعيل كامل لـ lockdep debugging:**

```bash
#!/bin/bash
# فعّل كل الـ lock events
cd /sys/kernel/debug/tracing

echo 1 > events/lock/enable
echo 1 > tracing_on
echo "collecting lock traces for 10 seconds..."
sleep 10
echo 0 > tracing_on

# احفظ النتيجة
cp trace /tmp/lock_trace_$(date +%Y%m%d_%H%M%S).txt
echo "saved to /tmp/lock_trace_$(date +%Y%m%d_%H%M%S).txt"
```

**2. تشغيل locktorture للتحقق من صحة الـ rwlock:**

```bash
#!/bin/bash
# اختبر الـ rwlock لمدة 60 ثانية
modprobe locktorture torture_type=rw_lock \
    nfakewriters=8 \
    nfakereaders=16 \
    stutter=5 \
    shuffle_interval=3

sleep 60

# اعرف النتيجة
dmesg | tail -20 | grep -E "locktorture|Torture"
rmmod locktorture
```

**مثال output ناجح:**

```
[ 1234.567] locktorture: rw_lock END OF TEST: SUCCESS: Writes: 45231 Reads: 180924 Duration: 60
```

**مثال output فاشل:**

```
[ 1234.567] locktorture: rw_lock END OF TEST: FAILURE: n_lock_fail: 3
```

**3. مراقبة الـ lock contention في real-time:**

```bash
#!/bin/bash
# مراقبة مستمرة كل ثانية
while true; do
    echo "=== $(date) ==="
    awk '/^[[:space:]]/ { print }' /proc/lock_stat | \
        awk '$4 > 0 { printf "%-40s contentions: %s\n", $1, $4 }' | \
        sort -k3 -rn | head -10
    sleep 1
done
```

**4. استخدام perf لتحديد أبطأ الـ rwlock acquisitions:**

```bash
# record
perf lock record -a -g -- sleep 10

# report مرتب حسب الـ average wait time
perf lock report --key=avg_wait

# مثال output:
#                  Name   acquired  contended   total wait   max wait   avg wait
#
#              &sb->s_umount:R       8530          3       12.45 us    5.23 us    4.15 us
#           &inode->i_rwsem:W        234         12      892.12 us  301.45 us   74.34 us
```

**5. إيجاد كل الـ rwlock_t في module معين:**

```bash
# بعد load الـ module
nm /path/to/mymodule.ko | grep -i rwlock
# أو
readelf -s /path/to/mymodule.ko | grep rwlock

# تحقق من الـ symbol address في الـ kernel
grep "rwlock\|rw_lock" /proc/kallsyms | grep mymodule
```

**6. اكتشاف الـ deadlock يدوياً:**

```bash
# لو الـ system هنج، اعمل sysrq-t لطباعة كل الـ threads
echo t > /proc/sysrq-trigger

# شوف الـ threads اللي في حالة D (waiting on lock)
dmesg | grep -A 5 "task:.*state:D"

# أو بـ crash tool على vmcore
crash vmlinux vmcore
(gdb equivalent in crash)
> ps | grep " D "
> bt <pid>   # backtrace للـ thread اللي واقف
```

**7. تفعيل verbose mode للـ rwlock في DEBUG_SPINLOCK:**

```bash
# compile kernel بـ:
# CONFIG_DEBUG_SPINLOCK=y
# CONFIG_DEBUG_LOCK_ALLOC=y
# CONFIG_PROVE_LOCKING=y

# ثم في runtime:
dmesg -w &  # اتابع الـ messages

# أي violation هيظهر فوراً زي:
# ================================================
# WARNING: lock held when returning to user space!
# 6.x.x #1 Not tainted
# ------------------------------------------------
# myprocess/1234 is leaving the kernel with locks still held!
# 1 lock held by myprocess/1234:
#  #0: ffff888012345678 (&my_rwlock){.+.+}-{2:2}, at: my_function+0x45/0x100
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Deadlock في Driver الـ SPI على بورد صناعي بمعالج RK3562

#### العنوان
Deadlock بين `read_lock_irqsave` و `write_lock_irq` في SPI controller driver على gateway صناعي

#### السياق
شركة تبني **industrial gateway** بمعالج **RK3562** يشغّل Linux 6.6. الـ gateway بيجمع بيانات من حساسات صناعية عبر **SPI** ويرسلها للسحابة. الـ driver الخاص بإدارة قائمة الـ SPI transactions بيستخدم `rwlock_t` لحماية linked list الـ pending transfers.

#### المشكلة
النظام بيـ freeze تماماً بعد ~8 ساعات من التشغيل. لا panic، لا logs. الـ watchdog بيـ reset البورد كل مرة. في الـ serial console اللي اتشاف قبل الـ reset:

```
INFO: task spi-worker:142 blocked for more than 120 seconds.
INFO: task irq/45-spi:89 blocked for more than 120 seconds.
```

#### التحليل
الـ engineer فتح الكود ولاقى التالي:

```c
/* في الـ interrupt handler */
static irqreturn_t spi_irq_handler(int irq, void *dev_id)
{
    struct spi_master *master = dev_id;
    /* ❌ غلط: بيـ acquire write lock من interrupt context */
    write_lock_irq(&master->queue_lock);
    /* process completed transfer */
    write_unlock_irq(&master->queue_lock);
    return IRQ_HANDLED;
}

/* في الـ worker thread */
static void spi_worker(struct work_struct *work)
{
    struct spi_master *master = container_of(work, ...);
    unsigned long flags;

    /* بيـ acquire read lock بعد ما disable interrupts */
    read_lock_irqsave(&master->queue_lock, flags);
    /* iterate over pending transfers */
    read_unlock_irqrestore(&master->queue_lock, flags);
}
```

تتبع الكود في `rwlock.h`:

- الـ `write_lock_irq` بيعمل `_raw_write_lock_irq` اللي بيـ disable IRQs **بعد** ما يـ acquire الـ lock.
- الـ `read_lock_irqsave` في الـ worker بيحفظ الـ flags ويـ disable IRQs قبل ما يمسك الـ read lock.
- لو الـ worker مسك الـ read lock والـ interrupt جه في نفس اللحظة، الـ interrupt handler هيحاول يمسك write lock وهو مش هيقدر لأن في reader شاغل.
- لكن الـ worker عمل `read_lock_irqsave` يعني IRQs اتعطّلت — الـ interrupt مش هيتنفّذ أصلاً على نفس الـ CPU.
- المشكلة الحقيقية: على SMP (RK3562 = quad-core)، الـ interrupt handler بيشتغل على CPU آخر وبيطلب `write_lock_irq`، وده بيـ disable IRQs على CPU ده. الـ reader على CPU تاني مسك الـ lock وبينتظر event من الـ interrupt handler اللي هو كمان بينتظر الـ reader يخلص. **Deadlock كلاسيكي.**

بالإضافة لكده، `write_lock_irq` مش بيحفظ الـ flags — لو الـ IRQs كانت enabled قبلها هيعملها disabled، لكن لما يعمل `write_unlock_irq` هيـ enable تاني بدون ما يعرف الحالة الأصلية.

#### الحل
الحل الصح إن الـ interrupt handler يستخدم `write_lock` بس، لأن `write_lock_irq` من IRQ context هو مشكلة في حد ذاته. الأصح استخدام `write_trylock` أو إعادة تصميم المعمارية:

```c
/* الحل: استخدام write_trylock في الـ interrupt handler */
static irqreturn_t spi_irq_handler(int irq, void *dev_id)
{
    struct spi_master *master = dev_id;

    /* trylock: لو مش متاح، schedule work لـ process لاحقاً */
    if (!write_trylock(&master->queue_lock))
        return IRQ_WAKE_THREAD; /* delegate to threaded IRQ */

    /* process completed transfer */
    write_unlock(&master->queue_lock);
    return IRQ_HANDLED;
}

/* الـ worker يستخدم write_lock_irqsave لحفظ state صح */
static void spi_worker(struct work_struct *work)
{
    struct spi_master *master = container_of(work, ...);
    unsigned long flags;

    write_lock_irqsave(&master->queue_lock, flags);
    /* modify queue safely */
    write_unlock_irqrestore(&master->queue_lock, flags);
}
```

```bash
# تأكيد الـ deadlock قبل الإصلاح باستخدام lockdep
echo 1 > /proc/sys/kernel/prove_locking
dmesg | grep -i "possible deadlock"
```

#### الدرس المستفاد
الـ `write_lock_irq` و `read_lock_irqsave` مش متكافئين في حماية الـ IRQ state. على SMP، الـ interrupt handler ممكن يشتغل على core مختلف وممكن يتعارض مع write lock حتى لو IRQs اتعطلت على core تاني. الصح دايماً استخدام `write_lock_irqsave` في الـ threaded context و `write_trylock` في الـ hard IRQ context.

---

### السيناريو 2: Writer Starvation في HDMI Hotplug على Android TV Box بمعالج Allwinner H616

#### العنوان
Writer starvation يسبب تأخير لا نهائي في تحديث EDID عند توصيل شاشة HDMI جديدة

#### السياق
منتج **Android TV box** بمعالج **Allwinner H616** يشغّل Android 12 (kernel 5.15). عند توصيل شاشة **HDMI** جديدة، النظام بياخد أكتر من 30 ثانية عشان يظهر صورة. في الأحوال الطبيعية المفروض أقل من 2 ثانية.

#### المشكلة
تتبع الـ trace بيُظهر إن الـ HDMI hotplug handler بيتأخر في الحصول على write lock لتحديث الـ EDID cache:

```
[  45.231] hdmi-hotplug: waiting for write lock on edid_cache_lock...
[  75.891] hdmi-hotplug: acquired write lock after 30.66 seconds
```

#### التحليل
الـ driver بيستخدم `rwlock_t` لحماية الـ EDID cache:

```c
struct hdmi_drv {
    rwlock_t edid_cache_lock;
    struct edid *cached_edid;
    /* ... */
};
```

في كل frame render (60fps = كل ~16ms)، الـ display engine بيعمل:

```c
/* بيتكرر 60 مرة في الثانية */
static void hdmi_render_frame(struct hdmi_drv *drv)
{
    read_lock(&drv->edid_cache_lock);   /* يمسك read lock */
    /* يقرأ الـ EDID لتحديد الـ color space */
    read_unlock(&drv->edid_cache_lock);
}
```

وفي نفس الوقت الـ hotplug handler بيحاول:

```c
static void hdmi_hotplug_handler(struct hdmi_drv *drv)
{
    write_lock_irq(&drv->edid_cache_lock); /* ينتظر هنا */
    kfree(drv->cached_edid);
    drv->cached_edid = read_edid_from_monitor();
    write_unlock_irq(&drv->edid_cache_lock);
}
```

الـ `rwlock_t` في Linux kernel **مش بيضمن writer priority**. الـ `arch_read_lock` على ARM (H616 = Cortex-A53) بيستخدم implementation بتسمح لـ readers جدد يدخلوا حتى لو في writer بينتظر. النتيجة: طالما في reader جديد بييجي كل 16ms، الـ writer مش هيقدر يدخل أبداً.

الكود في `rwlock.h` بيوضح:
```c
/* do_raw_write_lock في non-debug mode */
#define do_raw_write_lock(rwlock) \
    do {__acquire(lock); arch_write_lock(&(rwlock)->raw_lock); } while (0)
```

الـ `arch_write_lock` بيـ spin حتى يلاقي مفيش readers — وده مش هيحصل لو readers بييجوا بشكل continuous.

#### الحل
الحل الأمثل تغيير الـ lock type لـ `seqlock_t` أو استخدام `RCU` للـ EDID cache، لكن الحل السريع هو تحديد الـ read path:

```c
/* حل مؤقت: استخدام read_trylock مع fallback */
static void hdmi_render_frame(struct hdmi_drv *drv)
{
    /* لو مفيش write pending، اقرأ عادي */
    if (likely(!rwlock_is_contended(drv->edid_cache_lock))) {
        read_lock(&drv->edid_cache_lock);
        /* use cached edid */
        read_unlock(&drv->edid_cache_lock);
    } else {
        /* في write pending، استخدم fallback edid */
        use_default_edid(drv);
    }
}
```

```c
/* الحل الصح على المدى البعيد: استبدال rwlock بـ RCU */
static void hdmi_render_frame(struct hdmi_drv *drv)
{
    struct edid *edid;
    rcu_read_lock();
    edid = rcu_dereference(drv->cached_edid);
    /* use edid */
    rcu_read_unlock();
}
```

```bash
# تشخيص writer starvation
echo 1 > /sys/kernel/debug/tracing/events/lock/enable
cat /sys/kernel/debug/tracing/trace | grep edid_cache_lock
```

#### الدرس المستفاد
الـ `rwlock_t` في Linux **مش fair** — لا يوجد writer priority. في أي system فيها continuous readers (مثل display pipeline)، الـ write path ممكن يتعذّب. الحل الصح من الأساس هو استخدام `seqlock` أو `RCU` للبيانات اللي بتُقرأ كتير وبتُكتب نادراً.

---

### السيناريو 3: استخدام غلط لـ `rwlock_init` في Driver على STM32MP1

#### العنوان
Kernel panic عند تحميل driver جديد لحساس I2C على STM32MP1 بسبب عدم تهيئة `rwlock_t`

#### السياق
فريق firmware بيطور driver لحساس **temperature/humidity** متصل بـ **I2C** على لوحة IoT بمعالج **STM32MP1**. الـ driver بيحفظ آخر قراءة في struct محمي بـ `rwlock_t`.

#### المشكلة
عند `insmod` الـ driver:

```
BUG: spinlock bad magic on CPU#0, kworker/0:1/42
 lock: 0xbf012a40, .magic: 00000000, .owner: <none>/-1, .owner_cpu: -1
Kernel panic - not syncing: bad rwlock magic
```

#### التحليل
الكود الغلط:

```c
struct sensor_data {
    rwlock_t data_lock;   /* ❌ لم يتم تهيئتها */
    s32 temperature;
    u32 humidity;
};

static int sensor_probe(struct i2c_client *client)
{
    struct sensor_data *data;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    /* kzalloc بتعمل zero-fill، لكن ده مش كافي لـ rwlock */

    /* ❌ الـ engineer نسي يعمل rwlock_init */

    /* أول قراءة من الـ hardware */
    write_lock(&data->data_lock); /* BOOM */
    data->temperature = read_temp_from_hw(client);
    write_unlock(&data->data_lock);
    return 0;
}
```

في `rwlock.h` مع `CONFIG_DEBUG_SPINLOCK`:

```c
#define rwlock_init(lock)               \
do {                                    \
    static struct lock_class_key __key; \
    __rwlock_init((lock), #lock, &__key); \
} while (0)
```

الـ `__rwlock_init` بتسيب `magic = RWLOCK_MAGIC` (يعني `0xdeaf1eed`). لو الـ magic مش موجود، الـ debug checks في `do_raw_write_lock` بتـ panic فوراً.

حتى بدون `CONFIG_DEBUG_SPINLOCK`:

```c
#define rwlock_init(lock) \
    do { *(lock) = __RW_LOCK_UNLOCKED(lock); } while (0)
```

الـ `__RW_LOCK_UNLOCKED` بتعمل proper initialization للـ `arch_rwlock_t` — مجرد `kzalloc` مش كافي لأن الـ unlocked state مش دايماً zero على كل architectures.

على **ARM Cortex-A7** (STM32MP1)، الـ `arch_rwlock_t` هو `u32` والـ unlocked state هو `0` — ومن المصادفة اللي بتخلي bug زي ده صعب يتكشّف على non-debug builds!

#### الحل

```c
static int sensor_probe(struct i2c_client *client)
{
    struct sensor_data *data;

    data = devm_kzalloc(&client->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    /* ✅ صح: دايماً استخدم rwlock_init */
    rwlock_init(&data->data_lock);

    write_lock(&data->data_lock);
    data->temperature = read_temp_from_hw(client);
    write_unlock(&data->data_lock);
    return 0;
}
```

أو استخدام static initialization لو الـ struct static:

```c
/* ✅ بديل: static initialization */
static struct sensor_data global_sensor = {
    .data_lock = __RW_LOCK_UNLOCKED(global_sensor.data_lock),
    .temperature = 0,
};
```

```bash
# تفعيل DEBUG_SPINLOCK لاكتشاف المشاكل مبكراً
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_LOCK_ALLOC=y
```

#### الدرس المستفاد
`kzalloc` مش بديل عن `rwlock_init`. الـ zero-fill ممكن يشتغل بصدفة على architectures معينة لكن ده **undefined behavior**. دايماً استخدم `rwlock_init()` للـ dynamic allocation أو `__RW_LOCK_UNLOCKED()` للـ static initialization.

---

### السيناريو 4: استخدام `write_lock_nested` لحل Lock Ordering على i.MX8 في Automotive ECU

#### العنوان
Lockdep يرفع warning على nested `rwlock_t` في CAN bus driver على i.MX8 لـ ECU السيارة

#### السياق
شركة automotive بتبني **ECU** (Electronic Control Unit) بمعالج **NXP i.MX8** لإدارة نظام الـ braking. الـ driver بيتعامل مع **CAN bus** messages ومحتاج يـ update nested data structures بـ write lock على مستويين.

#### المشكلة
في بداية الـ testing مع `CONFIG_DEBUG_LOCK_ALLOC=y`:

```
WARNING: possible recursive locking detected
can_drv/345 is trying to acquire lock:
ffff8880123a4c40 (&bus->route_lock){....}-{2:2}

but task is already holding lock:
ffff8880123a4c40 (&bus->route_lock){....}-{2:2}

stack backtrace:
 can_update_route_table+0x4c/0x120
 can_process_message+0x88/0x200
 can_irq_handler+0x30/0x60
```

#### التحليل
الكود الأصلي:

```c
struct can_bus {
    rwlock_t route_lock;
    /* ... */
};

static void can_update_route_table(struct can_bus *bus,
                                    struct can_frame *frame)
{
    /* المستوى الأول: lock على الـ bus الرئيسي */
    write_lock(&bus->route_lock);
    /* update primary route */

    /* المستوى التاني: lock على نفس النوع (sub-route) */
    write_lock(&bus->route_lock); /* ❌ Lockdep يشتكي */
    /* update sub-route */
    write_unlock(&bus->route_lock);

    write_unlock(&bus->route_lock);
}
```

في `rwlock.h`:

```c
#ifdef CONFIG_DEBUG_LOCK_ALLOC
#define write_lock_nested(lock, subclass) \
    _raw_write_lock_nested(lock, subclass)
#else
#define write_lock_nested(lock, subclass) \
    _raw_write_lock(lock)
#endif
```

الـ `write_lock_nested` موجود بالظبط لحالة زي دي — لما عندك **lock hierarchy** وأكتر من instance من نفس الـ lock class. الـ `subclass` بيقول لـ lockdep "ده مقصود، مش bug".

في الـ automotive context ده ضروري لأن:
1. الـ CAN bus الرئيسي عنده `route_lock` (subclass 0)
2. كل virtual channel جوا الـ bus عنده نفس النوع من الـ lock (subclass 1)
3. الـ lock order ثابت دايماً: bus lock → channel lock

#### الحل

```c
/* تعريف الـ subclasses بوضوح */
enum can_lock_subclass {
    CAN_LOCK_BUS    = 0,  /* المستوى الأول: bus level */
    CAN_LOCK_CHANNEL = 1, /* المستوى التاني: channel level */
};

static void can_update_route_table(struct can_bus *bus,
                                    struct can_channel *chan,
                                    struct can_frame *frame)
{
    /* ✅ صح: استخدام write_lock_nested مع subclass صريح */
    write_lock(&bus->route_lock);           /* subclass 0 (default) */
    /* update bus-level route */

    write_lock_nested(&chan->route_lock, CAN_LOCK_CHANNEL); /* subclass 1 */
    /* update channel-level route */
    write_unlock(&chan->route_lock);

    write_unlock(&bus->route_lock);
}
```

```bash
# التحقق من عدم وجود lockdep warnings بعد الإصلاح
dmesg | grep -E "WARNING|lockdep|possible"

# تتبع الـ lock dependencies
cat /proc/lockdep_chains | grep can_route
```

#### الدرس المستفاد
الـ `write_lock_nested` مش مجرد workaround لتخطي lockdep — هو أداة لتوثيق الـ lock hierarchy بشكل صريح. في الـ safety-critical systems زي الـ automotive، توثيق الـ lock ordering بالكود نفسه أهم من أي comment. الـ `CONFIG_DEBUG_LOCK_ALLOC` لازم يكون مفعّل في جميع مراحل الـ development.

---

### السيناريو 5: تشخيص Race Condition في USB Gadget Driver على AM62x لجهاز IoT

#### العنوان
Intermittent kernel oops في USB gadget driver على AM62x بسبب استخدام `read_lock` بدل `read_lock_irqsave`

#### السياق
منتج **IoT sensor hub** بمعالج **TI AM62x** بيستخدم **USB** في وضع gadget (device mode) لنقل بيانات الحساسات لـ PC. الجهاز بيظهر `NULL pointer dereference` بشكل متقطع (مرة كل يومين تقريباً) صعب يتعيد.

#### المشكلة

```
BUG: kernel NULL pointer dereference, address: 0000000000000018
Oops: general protection fault, maybe for address 0x18
CPU: 0 PID: 0 (swapper/0)
  usb_gadget_send_data+0x7c/0x140
  gadget_bh_callback+0x44/0x80
  __do_softirq+0x128/0x340
```

الـ oops بيحصل في softirq context — وده مش interrupt context صريح، لكن هو **atomic context**.

#### التحليل
الكود المشبوه:

```c
struct gadget_drv {
    rwlock_t   endpoint_lock;
    struct usb_ep *bulk_in;   /* ممكن يتحرّر في أي وقت */
    /* ... */
};

/* softirq callback (BH context) */
static void gadget_bh_callback(struct work_struct *work)
{
    struct gadget_drv *drv = container_of(work, ...);

    /* ❌ غلط: في BH context يجب استخدام _bh variant */
    read_lock(&drv->endpoint_lock);
    if (drv->bulk_in) {
        usb_ep_queue(drv->bulk_in, /* ... */);
    }
    read_unlock(&drv->endpoint_lock);
}

/* USB disconnect handler (process context) */
static void gadget_disconnect(struct usb_gadget *gadget)
{
    struct gadget_drv *drv = gadget_to_drv(gadget);

    write_lock_irq(&drv->endpoint_lock);
    drv->bulk_in = NULL;  /* يحرر الـ endpoint */
    write_unlock_irq(&drv->endpoint_lock);
}
```

المشكلة: الـ `read_lock` (بدون `_bh`) في الـ softirq context **لا يمنع** الـ softirq من أن يُقاطَع بـ softirq آخر من نفس النوع على نفس الـ CPU. الـ `write_lock_irq` في الـ disconnect handler بيـ disable hard IRQs لكن مش BH. النتيجة:

1. الـ softirq بيمسك `read_lock`
2. الـ disconnect handler بيشتغل على نفس الـ CPU (preempt أو softirq يتحول لـ process context)
3. الـ disconnect handler ينتظر `write_lock` — لكن الـ softirq مش هيخلص لأن هو في BH context والـ process context هيـ preempt البعض ببعض

في `rwlock.h`:
```c
/* الماكرو الصح للـ BH context */
#define read_lock_bh(lock)   _raw_read_lock_bh(lock)
#define write_lock_bh(lock)  _raw_write_lock_bh(lock)
```

الـ `_bh` variants بتعمل `local_bh_disable()` قبل ما تمسك الـ lock، وده بيمنع الـ BH (softirq/tasklet) من أنه يقاطع نفسه ويسبب inconsistency.

#### الحل

```c
/* ✅ صح: BH context يحتاج _bh variant */
static void gadget_bh_callback(struct work_struct *work)
{
    struct gadget_drv *drv = container_of(work, ...);

    read_lock_bh(&drv->endpoint_lock);
    if (drv->bulk_in) {
        usb_ep_queue(drv->bulk_in, /* ... */);
    }
    read_unlock_bh(&drv->endpoint_lock);
}

/* ✅ صح: process context مع BH disable */
static void gadget_disconnect(struct usb_gadget *gadget)
{
    struct gadget_drv *drv = gadget_to_drv(gadget);

    write_lock_bh(&drv->endpoint_lock);
    drv->bulk_in = NULL;
    write_unlock_bh(&drv->endpoint_lock);
}
```

```bash
# تفعيل lockdep لاكتشاف context mismatches
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_ATOMIC_SLEEP=y

# محاكاة الـ disconnect بشكل متكرر لإعادة إنتاج البق
for i in $(seq 1 100); do
    echo "disconnect" > /sys/bus/usb/drivers/gadget/unbind
    echo "connect"    > /sys/bus/usb/drivers/gadget/bind
    sleep 0.1
done

# مراقبة الـ oops
dmesg -w | grep -E "BUG|oops|NULL pointer"
```

#### الدرس المستفاد
جدول الـ lock variants في `rwlock.h` مش مجرد options — كل واحدة ليها **context** محدد:

| Variant | Context | وظيفتها |
|---|---|---|
| `read_lock` | Process context فقط | لا تـ disable شيء |
| `read_lock_bh` | Process + BH context | تـ disable softirq |
| `read_lock_irq` | Process + IRQ context | تـ disable hard IRQ |
| `read_lock_irqsave` | Any context | تـ disable IRQ وتحفظ flags |

الخطأ الشائع هو استخدام `read_lock` في BH/softirq context ظناً أن ده كافي. لو عندك write path في process context مع BH enabled، لازم الـ read path في BH context يستخدم `read_lock_bh` عشان يحمي نفسه.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

أهم المقالات اللي بتغطي الـ rwlock في الـ Linux kernel:

| المقال | الوصف |
|--------|-------|
| [Eliminating rwlocks and IRQF_DISABLED](https://lwn.net/Articles/364583/) | بيشرح ليه الـ rwlock بطيء مقارنةً بالـ spinlock العادي، وإزاي المطورين بيحولوا للـ RCU أو plain spinlock |
| [Fair rwlock](https://lwn.net/Articles/294330/) | بيناقش مشكلة starvation في الـ rwlock التقليدي وازاي نعمل fair implementation |
| [Introducing a queue read/write lock implementation](https://lwn.net/Articles/582200/) | مقالة عن الـ qrwlock اللي بيحل مشاكل fairness في الـ rwlock الأصلي |
| [qrwlock: Introducing a queue read/write lock implementation](https://lwn.net/Articles/573607/) | نسخة أبكر من نفس الموضوع مع تفاصيل implementation |
| [Range reader/writer locks for the kernel](https://lwn.net/Articles/724502/) | فكرة متقدمة: range-based locking بدل الـ rwlock التقليدي |
| [seqlock: Add new blocking reader type & use rwlock](https://lwn.net/Articles/558102/) | بيوضح العلاقة بين الـ seqlock والـ rwlock كـ underlying mechanism |
| [What is RCU? Part 2: Usage](https://lwn.net/Articles/263130/) | بيقارن الـ RCU بالـ rwlock ويشرح إمتى كل واحد أنسب |

---

### التوثيق الرسمي في الـ Kernel

الملفات دي في `Documentation/locking/` هي المرجع الأساسي:

- **[Lock types and their rules](https://docs.kernel.org/locking/locktypes.html)**
  — بيشرح الفرق بين spinlock و rwlock و mutex ومتى تستخدم كل واحد، خصوصاً مع PREEMPT_RT

- **[Locking lessons (spinlocks.txt)](https://docs.kernel.org/locking/spinlocks.html)**
  — الدليل العملي لاستخدام الـ spinlock والـ rwlock مع أمثلة كود حقيقية

- **[Unreliable Guide To Locking](https://docs.kernel.org/kernel-hacking/locking.html)**
  — دليل شامل للـ locking في الـ kernel بيغطي الـ rwlock بالتفصيل

- **[Sequence counters and sequential locks](https://static.lwn.net/kerneldoc/locking/seqlock.html)**
  — بديل الـ rwlock في سيناريوهات الـ read-heavy

المسارات في سورس الـ kernel:
```
Documentation/locking/spinlocks.rst
Documentation/locking/locktypes.rst
Documentation/locking/lockdep-design.rst
```

---

### Kernel Commits المهمة

| الـ Commit | الوصف |
|-----------|-------|
| [qrwlock initial RFC](https://lore.kernel.org/lkml/1373679249-27123-2-git-send-email-Waiman.Long@hp.com/) | أول patch قدّم الـ qrwlock من Waiman Long |
| [Fair qrwlock v6](https://lore.kernel.org/lkml/1384267735-43213-5-git-send-email-Waiman.Long@hp.com/) | النسخة المحسّنة من الـ qrwlock بدعم fair queuing |
| [RISC-V generic rwlock](https://patchwork.kernel.org/project/linux-riscv/cover/20190211043829.30096-1-michaeljclark@mac.com/) | استخدام الـ generic spinlock/rwlock في RISC-V |

الملفات المهمة في السورس:
```
kernel/locking/spinlock.c      — الـ _raw_* functions
kernel/locking/qrwlock.c       — الـ queued rwlock implementation
include/linux/rwlock.h         — الـ macros الرئيسية (الملف ده)
include/linux/rwlock_types.h   — تعريف الـ rwlock_t
include/asm-generic/qrwlock.h  — الـ generic arch implementation
```

---

### نقاشات الـ Mailing List

- **[rwlocks — Linus Torvalds](https://yarchive.net/comp/linux/rwlocks.html)**
  — رأي Linus المباشر في الـ rwlock ومتى يكون مبرراً ومتى لا

- **[LKML: qrwlock discussion thread](https://lore.kernel.org/lkml/1384267735-43213-5-git-send-email-Waiman.Long@hp.com/)**
  — النقاش الكامل حول إضافة الـ fair qrwlock

---

### kernelnewbies.org

| الصفحة | الصلة بالـ rwlock |
|--------|-----------------|
| [Linux 3.10](https://kernelnewbies.org/Linux_3.10) | بيذكر تحسينات الـ rwlock scalability في هذا الإصدار |
| [Linux 5.1](https://kernelnewbies.org/Linux_5.1) | تحويل epoll لاستخدام rwlock لتقليل contention |
| [Linux 5.15](https://kernelnewbies.org/Linux_5.15) | تغييرات PREEMPT-RT اللي أثرت على rwlock semantics |
| [Linux 2.6.31](https://kernelnewbies.org/Linux_2_6_31) | إضافة interrupt-enabling rwlocks |
| [BigKernelLock](https://kernelnewbies.org/BigKernelLock) | سياق تاريخي: الـ BKL وعلاقته بتطور الـ locking في الـ kernel |

---

### الكتب الموصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 5: Concurrency and Race Conditions**
  - Section: "Spinlocks" و"Reader/Writer Spinlocks"
  - بيشرح `read_lock()`, `write_lock()` مع أمثلة device driver حقيقية
  - متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 10: Kernel Synchronization Methods**
  - Section: "Reader-Writer Spin Locks"
  - بيشرح الفرق بين الـ rwlock والـ spinlock بالأرقام والـ use cases

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)
- **الفصل 5: Kernel Synchronization**
  - بيغطي الـ rwlock على مستوى الـ architecture مع تفاصيل الـ x86 implementation

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 16: Kernel Debugging Techniques**
  - بيشرح lockdep وإزاي تـ debug مشاكل الـ rwlock

#### Linux Kernel Programming — Kaiwan Billimoria
- بيغطي الـ rwlock في سياق عملي مع الـ kernel modules

---

### مصطلحات البحث

لو عايز تبحث أكتر عن الموضوع ده، استخدم الـ search terms دي:

```
rwlock_t linux kernel internals
reader writer spinlock linux
arch_read_lock arch_write_lock implementation
do_raw_read_lock kernel locking
CONFIG_DEBUG_SPINLOCK rwlock
qrwlock queued rwlock linux
rwlock vs RCU linux kernel
rwlock PREEMPT_RT linux
read_lock_irqsave kernel driver
lockdep rwlock annotation
```

---

### ملفات السورس المباشرة على GitHub

```
https://github.com/torvalds/linux/blob/master/include/linux/rwlock.h
https://github.com/torvalds/linux/blob/master/include/linux/rwlock_types.h
https://github.com/torvalds/linux/blob/master/kernel/locking/spinlock.c
https://github.com/torvalds/linux/blob/master/kernel/locking/qrwlock.c
https://github.com/torvalds/linux/blob/master/include/asm-generic/qrwlock.h
```

---

### Ubuntu Real-Time Documentation

- **[Linux kernel locks — Real-time Ubuntu](https://documentation.ubuntu.com/real-time/latest/explanation/locks/)**
  — بيشرح إزاي الـ PREEMPT_RT بيغيّر سلوك الـ rwlock ويحوّله لـ rt_mutex-based implementation
## Phase 8: Writing simple module

### الفكرة

**`do_raw_write_lock`** هي الفانكشن اللي بتشتغل وراء كل `write_lock()` لما يكون `CONFIG_DEBUG_SPINLOCK` مفعّل — بتمسك الـ rwlock في write mode وبتضمن ما فيش أي reader أو writer تاني شغال في نفس الوقت. هنحط عليها **kprobe** نطبع فيه اسم العملية اللي طلبت الـ write lock وعنوان الـ lock نفسه — ده بيكشف مين بيحتكر الـ lock بصفة exclusive.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_rwlock.c
 * Hook do_raw_write_lock() to observe which task acquires write ownership
 * of an rwlock_t. Works on kernels built with CONFIG_DEBUG_SPINLOCK=y.
 */

#include <linux/module.h>      /* module_init / module_exit / MODULE_* macros  */
#include <linux/kernel.h>      /* pr_info                                       */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, ...           */
#include <linux/rwlock_types.h>/* rwlock_t definition                           */
#include <linux/sched.h>       /* current, task_comm_len                        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("kprobe on do_raw_write_lock – observe write-lock ownership");

/* ------------------------------------------------------------------
 * pre-handler: fires just BEFORE do_raw_write_lock() executes.
 * pt_regs carries the CPU register state at the probe site.
 * On x86-64:  rdi = first argument = rwlock_t *lock
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Grab the first argument (rwlock_t *).
     * regs->di is the x86-64 first integer argument register.
     * Cast is safe because kABI guarantees the pointer fits in a register.
     */
    rwlock_t *lock = (rwlock_t *)regs->di;

    /*
     * current is a per-CPU pointer to the running task_struct.
     * current->comm is the executable name (up to TASK_COMM_LEN bytes).
     */
    pr_info("kprobe_rwlock: write_lock acquired | task=\"%s\" pid=%d lock@%px\n",
            current->comm,
            current->pid,
            lock);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------
 * post-handler: fires right AFTER do_raw_write_lock() returns.
 * Useful to measure lock-hold latency if needed; kept minimal here.
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* Nothing extra needed – pre-handler already logged everything. */
}

/* ------------------------------------------------------------------
 * kprobe descriptor – one struct per hooked symbol.
 * .symbol_name lets the kernel resolve the address at load time
 * using the exported symbol table (no hard-coded addresses needed).
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "do_raw_write_lock",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------
 * module_init: register the kprobe so the kernel starts calling
 * our handlers whenever any code path hits do_raw_write_lock().
 * ------------------------------------------------------------------ */
static int __init rwlock_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("kprobe_rwlock: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_rwlock: planted at %px (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit: MUST unregister before the module's .text is freed.
 * Without this the kernel would jump into unmapped memory on the
 * next write_lock() call → guaranteed kernel panic.
 * ------------------------------------------------------------------ */
static void __exit rwlock_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_rwlock: removed from %s\n", kp.symbol_name);
}

module_init(rwlock_probe_init);
module_exit(rwlock_probe_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_rwlock.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### Build & Load

```bash
# build
make

# load (kernel must have CONFIG_DEBUG_SPINLOCK=y for do_raw_write_lock to exist)
sudo insmod kprobe_rwlock.ko

# watch output
sudo dmesg -w | grep kprobe_rwlock

# unload
sudo rmmod kprobe_rwlock
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | بيوفر الماكروهات الأساسية للـ module زي `module_init` و `MODULE_LICENSE` |
| `linux/kprobes.h` | بيعرّف `struct kprobe` والـ API بتاعه — ده هو جوهر الـ hooking |
| `linux/rwlock_types.h` | عشان نقدر نستخدم نوع `rwlock_t *` في الـ handler بشكل صح |
| `linux/sched.h` | بيعرّف الماكرو `current` اللي بيرجع `task_struct` للـ task الشغال دلوقتي |

#### الـ pre_handler

**الـ `pt_regs *regs`** بيحمل حالة الـ CPU registers لحظة اشتغال الـ probe. على **x86-64** الـ register `rdi` (أو `regs->di`) هو أول argument — وده بالظبط `rwlock_t *lock`. بنطبع اسم الـ task وعنوان الـ lock عشان نعرف مين بيمسك الـ write lock ومتى.

#### الـ post_handler

موجود لأن الـ kernel API بيتوقعه، بس في المثال ده مش محتاجينه. ممكن تستخدمه لقياس وقت الانتظار لو حبيت.

#### الـ `struct kprobe`

**`.symbol_name`** بيخلي الـ kernel يحل عنوان الفانكشن وقت التحميل من الـ symbol table بدل ما تكتب عناوين hard-coded — أأمن وأسهل للصيانة.

#### الـ `module_init`

`register_kprobe` بيزرع breakpoint (INT3 على x86) في أول تعليمة من `do_raw_write_lock`. من اللحظة دي أي call للفانكشن هتعدّي الـ handler الأول.

#### الـ `module_exit`

`unregister_kprobe` **ضروري جداً** — لو المودول اتفرغ من الذاكرة من غير ما يشيل الـ probe، أي write_lock تاني هيجمّد الكيرنل فوراً لأنه هيقفز على عنوان معدوم.
