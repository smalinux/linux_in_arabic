## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الملف `include/linux/local_lock.h` جزء من subsystem اسمه **LOCKING PRIMITIVES** — المسؤولين عنه Peter Zijlstra وIngo Molnar وWill Deacon. الكود اللي بيخدمه موجود في `kernel/locking/` وكل الـ headers في `include/linux/`.

---

### القصة: ليه احتاجوا حاجة زي دي أصلاً؟

تخيل عندك مطبخ (الـ CPU) فيه طباخ واحد بس (thread واحد شغال في وقت معين على الـ CPU دي). الطباخ ده عنده طاولة شغل خاصة بيه (الـ per-CPU data) — مش محتاج يأخذ إذن من حد تاني لما يشتغل عليها، لأنه الوحيد اللي بيستخدمها.

المشكلة: لو حد فجأة اقتحم المطبخ وبدأ يشتغل على نفس الطاولة وإنت لسه شغال (ده اللي بيحصل لما **preemption** بيحصل ويجي task تاني على نفس الـ CPU، أو لما **interrupt** بيجي) — هنا بيبقى في race condition على بيانات اللي كنت بتشتغل عليها.

الحل القديم كان: `preempt_disable()` أو `local_irq_disable()` — أي اقفل الباب كامل، ماحدش يدخل. لكن المشكلة:
1. **مش واضح** إيه اللي بتحمله بالظبط — بس رقم، مفيش اسم.
2. **الـ lockdep** (أداة kernel لاكتشاف deadlocks) مش عارف يتعلم منه.
3. لما بتيجي على **PREEMPT_RT** kernel (real-time kernel) — الـ `preempt_disable()` مش بيشتغل بنفس الطريقة، لأن RT بيسمح بـ preemption حتى وسط critical sections عشان يضمن latency منخفض.

**الـ `local_lock`** جه يحل المشاكل دي كلها بطريقة واحدة موحدة.

---

### الهدف من الملف

**الـ `local_lock`** هو wrapper بيديك اسم صريح للـ critical section اللي بتحمي فيها بيانات per-CPU. الفكرة بسيطة:

- على الـ **kernel العادي (non-RT)**: الـ `local_lock` بيعمل `preempt_disable()` بس، ومفيش lock حقيقي بياخده. الحماية بتيجي من إنه طالما الـ preemption مش شغال، مش ممكن task تانية تشتغل على نفس الـ CPU.
- على الـ **PREEMPT_RT kernel**: الـ `local_lock` بيتحول لـ per-CPU `spinlock_t` حقيقي — لأن RT محتاج lock حقيقي يضمن الـ mutual exclusion مع إنه بيسمح بـ preemption.

الملف ده هو الـ **public API** — بس macros وwrappers. الـ implementation الحقيقية في `local_lock_internal.h`.

---

### الـ Variants المتاحة

| Macro | بيعمل إيه على non-RT | بيعمل إيه على RT |
|---|---|---|
| `local_lock(lock)` | `preempt_disable()` | `migrate_disable()` + `spin_lock()` |
| `local_lock_irq(lock)` | `local_irq_disable()` | نفس `local_lock` |
| `local_lock_irqsave(lock, flags)` | `local_irq_save(flags)` | نفس `local_lock` |
| `local_trylock(lock)` | `preempt_disable()` + check acquired | `spin_trylock()` |
| `local_lock_nested_bh(lock)` | لازم تكون جوه softirq + acquire | `spin_lock()` |

---

### ليه مش بس `preempt_disable()`؟

```c
/* الطريقة القديمة — مش واضح إيه اللي بتحمي */
preempt_disable();
/* شغل على per-CPU data */
preempt_enable();

/* الطريقة الجديدة — اسم صريح */
local_lock(&my_lock);
/* شغل على per-CPU data */
local_unlock(&my_lock);
```

ميزة الـ `local_lock`:
1. **الـ lockdep** يعرف اسم الـ lock ويقدر يكتشف deadlocks.
2. الكود بيبقى **قابل للترحيل لـ PREEMPT_RT** من غير ما تغير حاجة.
3. **Static analysis tools** زي Sparse تقدر تتحقق من صحة الاستخدام.
4. الـ `owner` بيتسجل في debug mode — تعرف مين أخد الـ lock.

---

### القصة الكاملة: مشكلة PREEMPT_RT

الـ **PREEMPT_RT** kernel بيحول كتير من الـ spinlocks لـ sleeping locks عشان يضمن real-time latency. لما حاجة kayak `preempt_disable()` بتتحول لـ sleeping lock، الكود اللي كان معتمد على `preempt_disable()` بس ممكن يتكسر.

مثال: الـ SLUB allocator بيستخدم per-CPU caches. لو قلنا `preempt_disable()` على RT، وجوه حاول يعمل lock تاني بـ sleeping lock — بيحصل **BUG** لأن مش مسموح تنام وإنت preemption مش شغال.

الحل: استبدل `preempt_disable()` بـ `local_lock()`. على non-RT مفيش فرق. على RT بيتحول تلقائياً لـ `spin_lock()` حقيقي.

```c
/* في mm/slab.h */
#include <linux/local_lock.h>

struct kmem_cache_cpu {
    local_lock_t lock;  /* per-CPU lock */
    /* ... */
};

/* الاستخدام */
local_lock(&s->cpu_slab->lock);
/* ... اشتغل على الـ per-CPU slab ... */
local_unlock(&s->cpu_slab->lock);
```

---

### الـ Lock Guards (RAII style)

الملف بيعرّف **DEFINE_LOCK_GUARD_1** لكل نوع — ده بيسمح باستخدام الـ scoped locking في C:

```c
/* بدل */
local_lock(&my_lock);
do_work();
local_unlock(&my_lock);

/* تقدر تكتب */
scoped_guard(local_lock, &my_lock) {
    do_work();
}  /* بيتعمل unlock تلقائياً لما الـ scope ينتهي */
```

---

### الـ `local_trylock` — متى تستخدمه؟

**الـ `local_trylock`** بيجرب ياخد الـ lock — لو فاشل مش بيـblock. مفيد في:
- NMI handlers.
- Contexts اللي مش مسموح فيها ننتظر.

**ملاحظة مهمة**: على PREEMPT_RT، الـ `local_trylock` بيفشل دايماً لو كنت في NMI أو hardirq context — لأن الـ spinlock مش ممكن تاخده من السياقات دي.

---

### الملفات المهمة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `include/linux/local_lock.h` | الـ public API — الملف ده |
| `include/linux/local_lock_internal.h` | الـ implementation الفعلية (non-RT وRT) |
| `include/linux/percpu-defs.h` | تعريف `DEFINE_PER_CPU` وأصحابها |
| `include/linux/lockdep.h` | الـ lockdep infrastructure |
| `include/linux/spinlock.h` | الـ spinlock اللي بيستخدمه RT |
| `kernel/locking/` | كل الـ locking primitives implementations |
| `Documentation/locking/locktypes.rst` | شرح رسمي لكل أنواع الـ locks |
| `mm/slab.h` | مثال حقيقي على الاستخدام في SLUB |
| `mm/swap.c` | مثال آخر — page reclaim |
| `include/linux/mmzone.h` | استخدام في memory zones |
| `include/linux/radix-tree.h` | استخدام في radix tree |

---

### ملخص الصورة الكبيرة

```
Per-CPU Data Protection
========================

non-RT kernel:              PREEMPT_RT kernel:
─────────────               ──────────────────
local_lock(x)               local_lock(x)
    │                           │
    ▼                           ▼
preempt_disable()         migrate_disable()
    +                           +
lockdep annotation         spin_lock(per-cpu spinlock)
    │                           │
  [critical section]        [critical section]
    │                           │
    ▼                           ▼
local_unlock(x)            local_unlock(x)
    │                           │
    ▼                           ▼
preempt_enable()          spin_unlock()
                          + migrate_enable()
```

الـ `local_lock.h` هو الـ abstraction layer اللي بيخلي كود واحد يشتغل صح على الـ kernels المختلفة من غير ما الـ developer يفكر في التفاصيل.
## Phase 2: شرح الـ local_lock Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في الـ kernel، كتير من الـ subsystems بتستخدم **per-CPU data structures** — يعني كل CPU عنده نسخته الخاصة من البيانات. الفكرة الأصلية: لو كل CPU بيشتغل على نسخته بس، مش محتاج lock خالص، لأن مفيش تعارض.

لكن في الواقع الأمور مش بسيطة كده، فيه حالتين بتكسّر الفرضية دي:

**1. Preemption:**
الـ task ممكن يبدأ يقرأ per-CPU data على CPU 0، يتعمله preempt، ويكمّل على CPU 1 — اللي بياخد نسخة مختلفة خالص. الكود بقى يشتغل على بيانات غلط من غير ما يحس.

**2. Nested access من contexts تانية:**
الـ softirq أو hardirq ممكن يقاطعوا الـ task وهو في النص، ويعدّلوا على نفس الـ per-CPU variable. النتيجة: corruption من غير race condition كلاسيكي.

#### الحل التقليدي وعيوبه

الحل القديم كان ببساطة:
```c
/* القديم — بدائي وبيخبّي المشكلة */
preempt_disable();
/* access the per-CPU data */
preempt_enable();
```

أو:
```c
local_irq_save(flags);
/* access the per-CPU data */
local_irq_restore(flags);
```

المشكلة الكبيرة: ده **implicit locking** — مش واضح إيه اللي بيتحمى ولا مين بيحميه. الـ **lockdep** (الـ runtime lock validator في الـ kernel) مش شايف الـ lock ده خالص، يعني مش قادر يكتشف deadlocks أو misuse.

أسوأ من كده: لما بتشغّل **CONFIG_PREEMPT_RT** (الـ Real-Time kernel) — اللي بيخلي كل حاجة preemptible تقريباً حتى interrupt handlers — الـ `preempt_disable()` بقى مش كافي لأن الـ RT kernel بيتجاهله في حالات معينة. محتاج حل تاني.

---

### الحل — الـ local_lock Framework

الـ **local_lock** هو abstraction فوق آليات الحماية الموجودة، بيضيف:

1. **اسم صريح للـ lock** — lockdep بيشوفه ويتتبّعه.
2. **per-CPU scope** — كل CPU عنده instance مستقلة، مفيش contention بين الـ CPUs.
3. **Portability بين `!PREEMPT_RT` و `PREEMPT_RT`** — نفس الـ API، تنفيذ مختلف تحت.

الـ framework بيقدم الضمان ده: **الـ task اللي ماسك الـ lock ده هو لوحده اللي بيشتغل على الـ CPU ده في الـ critical section** (عن طريق منع preemption)، أو على الأقل هو لوحده اللي ماسك الـ per-CPU lock ده (على PREEMPT_RT).

---

### الـ Real-World Analogy — مقارنة عميقة

تخيّل **مطبخ المطعم** — عنده **محطة تقطيع** واحدة (per-CPU resource). كل طباخ بيشتغل على محطته الخاصة.

| مفهوم الـ kernel | المقابل في المطبخ |
|---|---|
| **per-CPU variable** | محطة تقطيع مخصصة لكل طباخ |
| **task** | الطباخ نفسه |
| **preemption** | مدير المطبخ اللي ممكن يوقف الطباخ ويبعته مكان تاني |
| **softirq** | زبون VIP بيطلب أمر عاجل يقاطع الطباخ |
| **local_lock** | الـ sign اللي الطباخ بيحطه "أنا شغّال هنا، متقاطعنيش" |
| **preempt_disable()** | نفس الـ sign بس مش مكتوب عليه اسمه |
| **lockdep** | مفتش الصحة اللي بيراقب هل الـ signs بتتحط صح |
| **PREEMPT_RT** | مطبخ RT لأوبرا — حتى مدير المطبخ ممكن يتوقف، فلازم lock حقيقي |

لو الطباخ (task) بدأ يقطع (يدخل الـ critical section) وحط الـ sign (local_lock)، مش هيتوقف لحد ما ينهي (أو يخلي حد يقاطعه softirq). لو المدير (scheduler) حاول ينقله لمحطة تانية، الـ lock بيمنعه.

على الـ PREEMPT_RT: الـ sign بقى **قفل حقيقي على الباب** (spinlock) — حتى لو المدير قدر يوقفه، حد تاني مش هيقدر يدخل المحطة دي.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kernel Subsystems                           │
│  (networking, memory allocator, scheduler, crypto, ...)         │
└───────────┬─────────────────────────────────────────────────────┘
            │  uses local_lock() / local_unlock()
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                  local_lock API  (linux/local_lock.h)           │
│                                                                 │
│  local_lock()          local_unlock()                           │
│  local_lock_irq()      local_unlock_irq()                       │
│  local_lock_irqsave()  local_unlock_irqrestore()                │
│  local_trylock()                                                │
│  local_lock_nested_bh()                                         │
└───────────┬─────────────────────────────────────────────────────┘
            │  delegates to __this_cpu_local_lock() + internal macros
            ▼
┌─────────────────────────────────────────────────────────────────┐
│              local_lock_internal.h  (Implementation)            │
│                                                                 │
│  ┌────────────────────────┐    ┌────────────────────────────┐   │
│  │  !CONFIG_PREEMPT_RT    │    │   CONFIG_PREEMPT_RT        │   │
│  │                        │    │                            │   │
│  │  local_lock_t = empty  │    │  local_lock_t = spinlock_t │   │
│  │  struct (+ lockdep)    │    │                            │   │
│  │                        │    │                            │   │
│  │  __local_lock()        │    │  __local_lock()            │   │
│  │    preempt_disable()   │    │    migrate_disable()       │   │
│  │    lockdep annotation  │    │    spin_lock()             │   │
│  │                        │    │                            │   │
│  │  __local_lock_irq()    │    │  __local_lock_irq()        │   │
│  │    local_irq_disable() │    │    = __local_lock()        │   │
│  │    lockdep annotation  │    │    (RT: irqs already OK)   │   │
│  └────────────────────────┘    └────────────────────────────┘   │
└───────────┬─────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Per-CPU Storage                             │
│                                                                 │
│  CPU0: local_lock_t instance   (owned by task on CPU0)          │
│  CPU1: local_lock_t instance   (owned by task on CPU1)          │
│  CPU2: local_lock_t instance   (owned by task on CPU2)          │
│  ...                                                            │
└─────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     lockdep (Validator)                         │
│                                                                 │
│  - يتتبع كل lock/unlock                                         │
│  - يكتشف deadlocks                                              │
│  - يكتشف wrong-context usage                                    │
│  - يعرف إن local_lock هو LD_LOCK_PERCPU                         │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction

الفكرة المحورية في الـ local_lock هي: **named per-CPU critical section**.

مش بس `preempt_disable()` عشوائي — ده lock بـ:
- **اسم** يعرفه lockdep (`#lockname` في الـ `dep_map`)
- **مالك** (`owner = current`) لو `CONFIG_DEBUG_LOCK_ALLOC`
- **scope محدد** (الـ per-CPU instance اللي أنت عليها)
- **نوع** (`LD_LOCK_PERCPU`) يخلي lockdep يفهم الـ semantics الصح

#### الـ struct في !PREEMPT_RT

```c
/* من local_lock_internal.h */
context_lock_struct(local_lock) {
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;   /* lockdep tracking */
    struct task_struct  *owner;    /* مين ماسك الـ lock */
#endif
    /* بدون DEBUG: الـ struct فاضي تماماً — zero overhead */
};
typedef struct local_lock local_lock_t;
```

لما `CONFIG_DEBUG_LOCK_ALLOC` مش موجود، الـ `local_lock_t` هو struct فاضي. التحسين هنا: **zero runtime overhead** — مفيش memory، مفيش operations. الحماية بتيجي من `preempt_disable()` بس، والـ lock نفسه بيختفي في الـ production build.

#### الـ struct في PREEMPT_RT

```c
/* على PREEMPT_RT */
typedef spinlock_t local_lock_t;
```

هنا الـ lock حقيقي — لأن على الـ RT kernel، الـ preempt_disable() مش بيمنع preemption في جميع الحالات بالنسبة لـ RT tasks.

#### الـ local_trylock_t

```c
context_lock_struct(local_trylock) {
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
    struct task_struct  *owner;
#endif
    u8  acquired;   /* flag: هل اتاخد الـ lock؟ */
};
```

الفرق عن `local_lock_t`: فيه `acquired` field — لأن الـ trylock محتاج يعرف هل الـ lock متاخد أصلاً (من نفس الـ CPU في context تاني) قبل ما يحاول ياخده.

---

### تفاصيل التنفيذ — كيف بيشتغل كل macro

#### `__this_cpu_local_lock(base)`

```c
/* من local_lock_internal.h */
static __always_inline local_lock_t *__this_cpu_local_lock(local_lock_t __percpu *base)
{
    return this_cpu_ptr(base);  /* بيجيب pointer للـ instance على الـ CPU الحالي */
}
```

**الـ percpu subsystem** (اللي محتاج تعرفه): بيخزّن variables بطريقة بتضمن إن كل CPU بيوصل للنسخة الخاصة بيه بسرعة، بدون cache thrashing بين الـ CPUs.

الـ `__this_cpu_local_lock` بياخد الـ per-CPU base address ويحوّله لـ pointer فعلي على الـ CPU الحالي. الـ `__percpu` annotation بتخلي الـ compiler (وأدوات زي `sparse`) يفهم إن ده per-CPU pointer مش عادي.

#### `__local_lock(lock)` — الـ !PREEMPT_RT path

```c
#define __local_lock(lock)
    do {
        preempt_disable();              /* امنع الـ task من الانتقال لـ CPU تاني */
        __local_lock_acquire(lock);     /* lockdep annotation + owner tracking */
        __acquire(lock);                /* sparse annotation */
    } while (0)
```

الترتيب مهم جداً: **أول `preempt_disable()`، بعدين الـ lockdep annotation**. لو عكست الترتيب، ممكن تتعمل preempt بين الـ annotation والـ disable، ويحصل confusion في lockdep.

#### `__local_lock_irq(lock)` — مع IRQ disable

```c
#define __local_lock_irq(lock)
    do {
        local_irq_disable();            /* امنع الـ interrupts كمان */
        __local_lock_acquire(lock);
        __acquire(lock);
    } while (0)
```

هنا `local_irq_disable()` بيعمل أكتر من `preempt_disable()` — بيمنع الـ hardirqs من مقاطعة الـ critical section. ده مهم لو الـ per-CPU data بيتعمل عليها access من interrupt context.

**لاحظ:** `local_irq_disable()` ضمناً بيعمل preemption disable لأن الـ preempt count بيتأثر بالـ IRQ state على معظم architectures.

#### `__local_trylock(lock)` — التنفيذ الأذكى

```c
#define __local_trylock(lock)
    __try_acquire_ctx_lock(lock, ({
        local_trylock_t *__tl;

        preempt_disable();
        __tl = (lock);
        if (READ_ONCE(__tl->acquired)) {    /* هل اتاخد من قبل؟ */
            preempt_enable();               /* فاشل — رجّع الـ preemption */
            __tl = NULL;
        } else {
            WRITE_ONCE(__tl->acquired, 1);  /* احجزه */
            local_trylock_acquire(
                (local_lock_t *)__tl);
        }
        !!__tl;                             /* return: 1 = نجح، 0 = فاشل */
    }))
```

الـ `READ_ONCE` / `WRITE_ONCE` هنا مش عشان thread safety (احنا على نفس الـ CPU، ومفيش race) — لكن عشان نمنع الـ compiler من إنه يعمل **reorder** أو يحذف الـ read/write كـ optimization. ده critical لأن الـ flag ممكن يتعدل من context تاني على نفس الـ CPU (مثلاً softirq أو NMI) لو `local_trylock` اتستخدم هناك.

#### الـ PREEMPT_RT path — الفرق الجوهري

```c
/* على PREEMPT_RT */
#define __local_lock(__lock)
    do {
        migrate_disable();   /* بدل preempt_disable */
        spin_lock((__lock)); /* lock حقيقي بدل annotation */
    } while (0)
```

**ليه `migrate_disable()` بدل `preempt_disable()`؟**

على الـ PREEMPT_RT، الـ `preempt_disable()` ما بيمنعش preemption للـ RT tasks — ده by design. عشان كده بيستخدموا `migrate_disable()` اللي بيمنع بس نقل الـ task لـ CPU تاني (migration)، مع إنه ممكن يتعمل له preempt على نفس الـ CPU. الـ spinlock هو اللي بيحمي الـ critical section فعلاً.

**ليه `local_lock_irq` = `local_lock` على PREEMPT_RT؟**

```c
#define __local_lock_irq(lock)  __local_lock(lock)
```

على الـ RT kernel، الـ interrupts بتشتغل في threaded context — مش في interrupt context حقيقي. عشان كده `local_irq_disable()` مش ضروري ومش كافي. الـ spinlock هو الحل الوحيد.

---

### الـ nested_bh variant

```c
#define __local_lock_nested_bh(lock)
    do {
        lockdep_assert_in_softirq(); /* لازم تكون في softirq context */
        local_lock_acquire((lock));  /* lockdep annotation بس */
        __acquire(lock);
    } while (0)
```

ده لحالة خاصة: لو أنت **داخل softirq بالفعل**، الـ softirqs مش محتاجة تتوقف تاني (احنا بالفعل فيها). بس محتاج تحمي الـ per-CPU data من softirqs تانية. الحل: `local_lock_nested_bh` بيعمل annotation للـ lockdep فقط بدون أي overhead.

---

### الـ Lock Guard Integration

الـ `DEFINE_LOCK_GUARD_1` بيعمل integration مع الـ **cleanup-based locking** (C scoped locking باستخدام `__attribute__((cleanup))`):

```c
DEFINE_LOCK_GUARD_1(local_lock, local_lock_t __percpu,
                    local_lock(_T->lock),     /* acquire */
                    local_unlock(_T->lock))   /* release تلقائي لما الـ scope ينتهي */
```

الاستخدام:

```c
/* مثال حقيقي */
DEFINE_PER_CPU(local_lock_t, my_lock);

void my_function(void)
{
    /* الـ lock بياخد تلقائياً هنا */
    CLASS(local_lock, guard)(&my_lock);

    /* critical section */
    per_cpu_data[smp_processor_id()]++;

    /* الـ lock بيتحرر تلقائياً لما الـ scope ينتهي — حتى لو في early return */
}
```

---

### الـ lockdep Integration — التفاصيل

```c
#define __local_lock_init(lock)
do {
    static struct lock_class_key __key;  /* unique key لكل lock site */

    debug_check_no_locks_freed((void *)lock, sizeof(*lock));
    lockdep_init_map_type(&(lock)->dep_map,
                          #lock,             /* اسم الـ lock كـ string */
                          &__key,
                          0,
                          LD_WAIT_CONFIG,    /* wait type: configurable */
                          LD_WAIT_INV,
                          LD_LOCK_PERCPU);   /* نوعه: per-CPU lock */
} while (0)
```

**الـ `LD_LOCK_PERCPU`** بيقول للـ lockdep: ده مش lock عادي — هو per-CPU بطبيعته، فمتشكيش لو CPUs مختلفة بياخدوه في نفس الوقت. ده valid لأن كل CPU بياخد الـ instance الخاصة بيه.

**الـ `LD_WAIT_CONFIG`** بيقول: ده lock ممكن ينتظر في "schedulable context" — يعني مش atomic, مش hardirq context.

---

### ما بيمتلكه الـ framework vs ما بيفوّضه للـ drivers

```
┌──────────────────────────────────────────────────────────┐
│            ما بيمتلكه local_lock framework               │
│                                                          │
│  ✓ الـ API الموحدة (lock/unlock/trylock/nested_bh)      │
│  ✓ الـ lockdep integration والـ annotations             │
│  ✓ الـ per-CPU instance selection (this_cpu_ptr)        │
│  ✓ الـ PREEMPT_RT compatibility layer                   │
│  ✓ الـ Lock Guard (RAII) integration                    │
│  ✓ الـ WARN_CONTEXT_ANALYSIS support                    │
│  ✓ فيريفيكيشن إن الـ owner صح (DEBUG builds)           │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│            ما بيفوّضه للـ subsystem/driver               │
│                                                          │
│  ✗ تعريف الـ lock نفسه كـ DEFINE_PER_CPU               │
│  ✗ اختيار نوع الـ lock (lock vs lock_irq vs trylock)    │
│  ✗ حماية الـ data المحددة اللي جواه                     │
│  ✗ initialization (INIT_LOCAL_LOCK أو local_lock_init)  │
│  ✗ تحديد granularity الـ critical section               │
└──────────────────────────────────────────────────────────┘
```

---

### مثال واقعي — كيف بيتستخدم في الـ kernel

الـ **slab allocator** (memory allocator في الـ kernel) بيستخدم الـ local_lock لحماية الـ per-CPU slab cache:

```c
/* مثال مشابه لما في mm/slub.c */
static DEFINE_PER_CPU(local_lock_t, s_lock);

static inline void *get_freelist(struct kmem_cache *s, ...)
{
    local_lock(&s_lock);

    /* access per-CPU freelist — آمن من preemption */
    void *obj = this_cpu_read(s->cpu_slab->freelist);
    /* ... */

    local_unlock(&s_lock);
    return obj;
}
```

بدل ما كانوا يكتبوا `preempt_disable()` ومفيش لوكدب يشوف، بقى فيه اسم صريح (`s_lock`) لـ lockdep يتتبعه.

---

### الـ Variants — متى تستخدم إيه؟

| Variant | متى تستخدمه |
|---|---|
| `local_lock` | الـ per-CPU data لا تتعمل عليها access من interrupt context |
| `local_lock_irq` | الـ per-CPU data ممكن تتعمل عليها access من hardirq |
| `local_lock_irqsave` | زي `local_lock_irq` بس محتاج تعرف الـ IRQ state القديمة |
| `local_trylock` | في contexts زي NMI أو hardirq لو عايز تحاول من غير blocking |
| `local_lock_nested_bh` | جواه softirq context — بتحمي من softirqs تانية بدون overhead |

---

### خلاصة — الصورة الكاملة

الـ **local_lock framework** هو الجواب على سؤال: "إزاي أحمي per-CPU data بشكل صح، قابل للـ debugging، وشغّال على كل kernel configurations؟"

بيحل ثلاث مشاكل في واحد:

1. **Correctness**: بيمنع preemption (أو migration على RT) عشان الـ task ما ينتقلش بين CPUs.
2. **Debuggability**: بيضيف اسم وـ owner للـ lockdep يقدر يتتبعه.
3. **Portability**: نفس الكود بيشتغل صح على `!PREEMPT_RT` (zero-cost) وعلى `PREEMPT_RT` (real spinlock).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### `enum lockdep_wait_type` — نوع سياق الانتظار

| القيمة | المعنى |
|--------|--------|
| `LD_WAIT_INV` | غير محدد / catch-all، مش بيتعمل check |
| `LD_WAIT_FREE` | wait-free — زي RCU read-side |
| `LD_WAIT_SPIN` | spin loop — زي `raw_spinlock_t` |
| `LD_WAIT_CONFIG` | preemptible في PREEMPT_RT، زي `spinlock_t` العادي |
| `LD_WAIT_SLEEP` | sleeping lock — زي `mutex_t` |

**الـ `local_lock` بيستخدم `LD_WAIT_CONFIG`** عشان هو "CONFIG-preemptible" — يعني ممكن يتعمل preempt عليه في PREEMPT_RT.

#### `enum lockdep_lock_type` — نوع الـ lock

| القيمة | المعنى |
|--------|--------|
| `LD_LOCK_NORMAL` | lock عادي |
| `LD_LOCK_PERCPU` | per-CPU lock — ده اللي بيستخدمه `local_lock` |
| `LD_LOCK_WAIT_OVERRIDE` | annotation فقط |

#### Config Options المؤثرة في الملف

| Config | الأثر |
|--------|-------|
| `CONFIG_PREEMPT_RT` | يحوّل `local_lock_t` من struct خفيف → `spinlock_t` كامل |
| `CONFIG_DEBUG_LOCK_ALLOC` | يضيف `dep_map` و`owner` للـ struct لكشف الأخطاء |
| `CONFIG_PROVE_RAW_LOCK_NESTING` | يفصل `LD_WAIT_CONFIG` عن `LD_WAIT_SPIN` في lockdep |
| `WARN_CONTEXT_ANALYSIS` | يفعّل `__this_cpu_local_lock()` كـ inline function بدل macro |

---

### الـ Structs المهمة

#### 1. `struct local_lock` / `local_lock_t`

**الغرض:** هو الـ lock object الفعلي per-CPU اللي بيحمي critical sections محدودة بـ CPU واحد.

```c
/* !CONFIG_PREEMPT_RT version */
context_lock_struct(local_lock) {
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;   /* lockdep tracking metadata */
    struct task_struct *owner;     /* task that holds the lock (debug only) */
#endif
};
typedef struct local_lock local_lock_t;
```

- في **release builds** بدون `DEBUG_LOCK_ALLOC`: الـ struct **فاضي تماماً** — zero size.
- في **PREEMPT_RT**: `local_lock_t` = `spinlock_t` كامل — يعني real locking.
- بيتحط دايماً كـ `__percpu` variable، كل CPU عنده نسخته الخاصة.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `dep_map` | `struct lockdep_map` | metadata لـ lockdep validator (debug فقط) |
| `owner` | `struct task_struct *` | مين hold الـ lock دلوقتي (debug فقط) |

#### 2. `struct local_trylock` / `local_trylock_t`

**الغرض:** نفس `local_lock_t` لكن بيدعم `trylock` semantics — يعني ممكن تفشل في الـ acquire.

```c
/* !CONFIG_PREEMPT_RT version */
context_lock_struct(local_trylock) {
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
    struct task_struct *owner;
#endif
    u8  acquired;   /* 0 = free, 1 = held — checked atomically */
};
typedef struct local_trylock local_trylock_t;
```

- الـ field الإضافي `acquired` هو اللي بيخلي الـ trylock يشتغل: بيتعمل `READ_ONCE` عشان يعرف هل في حد hold الـ lock قبل ما يحاول يعمل acquire.
- في **PREEMPT_RT**: `local_trylock_t` = `spinlock_t` كمان — بيستخدم `spin_trylock()` مباشرة.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `dep_map` | `struct lockdep_map` | lockdep tracking (debug فقط) |
| `owner` | `struct task_struct *` | مين hold الـ lock (debug فقط) |
| `acquired` | `u8` | flag — 1 لو held، 0 لو free |

#### 3. `struct lockdep_map` (من `lockdep_types.h`)

**الغرض:** الـ metadata اللي بيعرف بيه lockdep كل lock instance وبيعمل dependency tracking.

```c
struct lockdep_map {
    struct lock_class_key  *key;          /* unique class identifier */
    struct lock_class      *class_cache[NR_LOCKDEP_CACHING_CLASSES];
    const char             *name;         /* human-readable name for reports */
    u8                      wait_type_outer; /* context this lock can be taken in */
    u8                      wait_type_inner; /* context this lock presents */
    u8                      lock_type;       /* NORMAL / PERCPU / WAIT_OVERRIDE */
};
```

- الـ `local_lock` بيضبط `wait_type_inner = LD_WAIT_CONFIG` و `lock_type = LD_LOCK_PERCPU`.
- الـ `key` بتوجّه lockdep للـ `lock_class` — كل نوع lock ليه class واحدة.

#### 4. `struct lock_class_key` (من `lockdep_types.h`)

**الغرض:** معرّف static فريد لكل نوع lock في الكود. بيتعمله `static` في كل macro.

```c
struct lock_class_key {
    union {
        struct hlist_node           hash_entry;   /* for dynamic registration */
        struct lockdep_subclass_key subkeys[MAX_LOCKDEP_SUBCLASSES];
    };
};
```

- في `__local_lock_init()`: `static struct lock_class_key __key;` — يعني نفس الـ key لكل call site.
- بيتسجّل في lockdep's hash table أول مرة يتشال.

---

### Struct Relationship Diagram

```
                  __percpu variable
                  ┌─────────────────────────────┐
                  │  local_lock_t  (per-CPU)     │
                  │  ┌─────────────────────────┐ │
                  │  │  struct lockdep_map     │ │
                  │  │  ┌───────────────────┐  │ │
                  │  │  │ lock_class_key *  │──┼─┼──► struct lock_class_key
                  │  │  │ lock_class *[]    │──┼─┼──► struct lock_class
                  │  │  │ name: "my_lock"   │  │ │        │
                  │  │  │ wait_type_inner:  │  │ │        ├─► locks_after (graph)
                  │  │  │   LD_WAIT_CONFIG  │  │ │        └─► locks_before (graph)
                  │  │  │ lock_type:        │  │ │
                  │  │  │   LD_LOCK_PERCPU  │  │ │
                  │  │  └───────────────────┘  │ │
                  │  │  owner: task_struct *   │ │
                  │  └─────────────────────────┘ │
                  │  [CPU 0 copy]                │
                  ├─────────────────────────────┤
                  │  [CPU 1 copy]                │
                  ├─────────────────────────────┤
                  │  [CPU N copy]                │
                  └─────────────────────────────┘

                  local_trylock_t (per-CPU)
                  ┌─────────────────────────────┐
                  │  struct lockdep_map          │  (same layout as above)
                  │  owner: task_struct *        │
                  │  acquired: u8  ◄─── extra    │  ← الفرق الوحيد
                  └─────────────────────────────┘

CONFIG_PREEMPT_RT:
  local_lock_t  ──────────► spinlock_t (real lock)
  local_trylock_t ─────────► spinlock_t (real lock)
```

---

### Lifecycle Diagram

#### إنشاء واستخدام `local_lock_t`

```
DEFINE_PER_CPU(local_lock_t, my_lock) = INIT_LOCAL_LOCK(my_lock);
        │
        │  static init → dep_map.name="my_lock"
        │               dep_map.lock_type=LD_LOCK_PERCPU
        │               owner=NULL
        ▼
[optional] local_lock_init(&my_lock)    ← runtime init
        │
        │  lockdep_init_map_type(...)
        │  local_lock_debug_init(...)   → owner=NULL
        ▼
[acquire]  local_lock(&my_lock)
        │
        │  __this_cpu_local_lock(&my_lock)  → this_cpu_ptr(&my_lock)
        │  preempt_disable()
        │  local_lock_acquire(l)
        │    lock_map_acquire(&l->dep_map)  [lockdep]
        │    l->owner = current
        ▼
[critical section — per-CPU data protected]
        │
        ▼
[release]  local_unlock(&my_lock)
        │
        │  local_lock_release(l)
        │    l->owner = NULL
        │    lock_map_release(&l->dep_map)  [lockdep]
        │  preempt_enable()
        ▼
[done]
```

#### Lifecycle لـ `local_trylock_t`

```
DEFINE_PER_CPU(local_trylock_t, my_trylock) = INIT_LOCAL_TRYLOCK(my_trylock);
        │
        ▼
[acquire attempt]  local_trylock(&my_trylock)
        │
        │  preempt_disable()
        │  READ_ONCE(acquired) == 0?
        │    YES ──► WRITE_ONCE(acquired, 1)
        │            local_trylock_acquire(...)
        │            return true  ──► enter critical section
        │    NO  ──► preempt_enable()
        │            return false ──► caller handles failure
        ▼
[on success — critical section]
        │
        ▼
[عادةً مفيش explicit unlock هنا — بيتعمل automatically عبر lock guard أو يدوي]
        │
        │  WRITE_ONCE(acquired, 0)
        │  local_lock_release(...)
        │  preempt_enable()
        ▼
[done]
```

---

### Call Flow Diagrams

#### `local_lock(lock)` — النمط الأساسي

```
local_lock(&percpu_lock)
  └─► __local_lock( __this_cpu_local_lock(&percpu_lock) )
        │
        ├─► __this_cpu_local_lock(&percpu_lock)
        │     └─► this_cpu_ptr(&percpu_lock)   [CPU-local pointer]
        │
        └─► __local_lock(cpu_local_ptr)
              │
              ├─► preempt_disable()             [!PREEMPT_RT]
              │   migrate_disable()             [PREEMPT_RT]
              │
              ├─► __local_lock_acquire(lock)
              │     ├─► [if local_trylock_t] WRITE_ONCE(acquired, 1)
              │     └─► local_lock_acquire(l)
              │           ├─► lock_map_acquire(&dep_map)   [lockdep]
              │           └─► l->owner = current
              │
              │   [PREEMPT_RT only: spin_lock(lock)]
              │
              └─► __acquire(lock)               [sparse annotation]
```

#### `local_lock_irqsave(lock, flags)` — مع حفظ الـ interrupts

```
local_lock_irqsave(&percpu_lock, flags)
  └─► __local_lock_irqsave( this_cpu_ptr(&percpu_lock), flags )
        │
        ├─► local_irq_save(flags)     [حفظ EFLAGS وتعطيل IRQ]
        │       [!PREEMPT_RT: فعلياً يعطّل]
        │       [PREEMPT_RT: flags=0 فقط، لأن spinlock بيتكفّل بالباقي]
        │
        └─► __local_lock_acquire(lock)
              └─► [نفس السابق]
```

#### `local_trylock(lock)` — محاولة آمنة من أي context

```
local_trylock(&percpu_trylock)
  └─► __local_trylock( this_cpu_ptr(&percpu_trylock) )
        │
        ├─► [!PREEMPT_RT]
        │     preempt_disable()
        │     READ_ONCE(acquired) ──► 0? ──► WRITE_ONCE(1), acquire, return true
        │                          ──► 1? ──► preempt_enable(), return false
        │
        └─► [PREEMPT_RT]
              in_nmi() || in_hardirq()? ──► return false immediately
              migrate_disable()
              spin_trylock(lock) ──► success? return true
                                 ──► fail? migrate_enable(), return false
```

#### `local_lock_nested_bh(lock)` — داخل softirq context

```
local_lock_nested_bh(&percpu_lock)
  └─► __local_lock_nested_bh( this_cpu_ptr(&percpu_lock) )
        │
        ├─► lockdep_assert_in_softirq()    [verify we're in BH context]
        │   lockdep_assert_in_softirq_func() [PREEMPT_RT variant]
        │
        └─► local_lock_acquire(lock)  [!PREEMPT_RT — no preempt_disable, BH already does it]
            spin_lock(lock)           [PREEMPT_RT]
```

---

### Locking Strategy — الاستراتيجية الكاملة

#### المبدأ الأساسي

الـ `local_lock` مش lock بين CPUs — هو lock **على نفس الـ CPU**. غرضه:

1. **منع الـ preemption** عشان ميصلش task تاني على نفس الـ CPU للـ per-CPU data.
2. **توثيق الـ ownership** لـ lockdep عشان يكتشف locking bugs.
3. **عدم استخدام atomic operations** — أسرع من spinlock عادي.

#### جدول أنواع الـ Acquire ومتى تستخدم كل واحدة

| الـ API | يعطّل Preemption | يعطّل IRQ | يعطّل BH | متى تستخدمه |
|---------|:-:|:-:|:-:|------------|
| `local_lock` | نعم | لا | لا | per-CPU data — process context فقط |
| `local_lock_irq` | نعم | نعم | نعم | per-CPU data + IRQ handlers يقروها |
| `local_lock_irqsave` | نعم | نعم (مع حفظ) | نعم | نفس السابق لكن nested-safe |
| `local_lock_nested_bh` | لا (BH بيعملها) | لا | نعم | داخل softirq handler |
| `local_trylock` | نعم (إذا نجح) | لا | لا | contexts حساسة — NMI/hardirq آمن |
| `local_trylock_irqsave` | نعم + IRQ | نعم (إذا نجح) | نعم | نفس trylock مع IRQ |

#### ما يحميه كل lock

```
per-CPU variable:  my_data[NR_CPUS]
                        │
          ┌─────────────┴─────────────────┐
          │                               │
    process context                  softirq/IRQ
    يستخدم local_lock()            يستخدم local_lock_irq()
          │                               │
          └──────────► نفس الـ CPU ◄──────┘
                       [واحد في وقت واحد]
```

#### Lock Ordering

**مفيش lock ordering problem** بالمعنى التقليدي لأن:
- الـ `local_lock` **per-CPU** — مفيش CPU يحاول يمسك lock CPU تاني.
- بس lockdep بيعمل check إن nested locks تتمسك بنفس الترتيب.

القاعدة الوحيدة:
```
  preempt_disable / migrate_disable     ← أول حاجة
    └─► local_lock_acquire (dep_map)    ← جوا
          └─► [critical section]
        local_lock_release (dep_map)    ← عكس الترتيب
  preempt_enable / migrate_enable       ← آخر حاجة
```

لو محتاج تجمع IRQ + local_lock:
```
  local_irq_save(flags)    ← أول ما تعطّل IRQ
    local_lock_acquire()   ← تاني تعمل acquire
    ...
    local_lock_release()   ← عكس
  local_irq_restore(flags) ← آخر حاجة
```

#### PREEMPT_RT — اختلاف جوهري

```
!PREEMPT_RT:                          PREEMPT_RT:
─────────────────────────             ─────────────────────────
local_lock_t = empty struct           local_lock_t = spinlock_t
preempt_disable()                     migrate_disable()
[no real spin]                        spin_lock()
[IRQ disable = real]                  [IRQ flags ignored = 0]
[trylock checks acquired flag]        [trylock = spin_trylock()]
[NMI/hardirq: trylock works]          [NMI/hardirq: trylock fails always]
```

السبب: في PREEMPT_RT الـ spinlock نفسه sleeping-capable — فبيستخدموا `spin_lock` حقيقي عشان يحافظوا على الـ preemptibility بدل ما يعطّلوها.

#### الـ Debug Checks

عند `CONFIG_DEBUG_LOCK_ALLOC`:

| Check | متى بيحصل | لو فشل |
|-------|-----------|--------|
| `DEBUG_LOCKS_WARN_ON(l->owner)` | عند الـ acquire | lock مأخوذ بالفعل |
| `DEBUG_LOCKS_WARN_ON(l->owner != current)` | عند الـ release | حد تاني بيحاول يعمل release |
| `lockdep_assert(__tl->acquired == 0)` | عند acquire لـ trylock | double-acquire |
| `lockdep_assert(__tl->acquired == 1)` | عند release لـ trylock | release بدون acquire |
| `lockdep_assert_in_softirq()` | في `nested_bh` | استخدام خارج BH context |
## Phase 4: شرح الـ Functions

---

### ملخص شامل — Cheatsheet

#### مجموعة الـ Initialization

| Macro / Function | الوصف المختصر |
|---|---|
| `local_lock_init(lock)` | Runtime init لـ `local_lock_t __percpu` |
| `local_trylock_init(lock)` | Runtime init لـ `local_trylock_t __percpu` |
| `local_lock_debug_init(l)` | يصفّر الـ `owner` في debug mode |
| `__local_lock_init(lock)` | Internal init — يسجّل الـ lockdep map |
| `__local_trylock_init(lock)` | نفس `__local_lock_init` بـ cast |

#### مجموعة الـ Acquire

| Macro | السلوك على !RT | السلوك على RT |
|---|---|---|
| `local_lock(lock)` | `preempt_disable()` + lockdep acquire | `migrate_disable()` + `spin_lock()` |
| `local_lock_irq(lock)` | `local_irq_disable()` + lockdep acquire | `migrate_disable()` + `spin_lock()` |
| `local_lock_irqsave(lock, flags)` | `local_irq_save(flags)` + lockdep acquire | `flags=0` + `migrate_disable()` + `spin_lock()` |
| `local_trylock(lock)` | check `acquired` → set 1 → `preempt_disable()` | رفض في NMI/HARDIRQ، وإلا `spin_trylock()` |
| `local_trylock_irqsave(lock, flags)` | `irq_save` + check `acquired` | `flags=0` + `local_trylock()` |
| `local_lock_nested_bh(lock)` | assert softirq context + lockdep acquire | assert softirq + `spin_lock()` |

#### مجموعة الـ Release

| Macro | السلوك على !RT | السلوك على RT |
|---|---|---|
| `local_unlock(lock)` | lockdep release + `preempt_enable()` | `spin_unlock()` + `migrate_enable()` |
| `local_unlock_irq(lock)` | lockdep release + `local_irq_enable()` | `spin_unlock()` + `migrate_enable()` |
| `local_unlock_irqrestore(lock, flags)` | lockdep release + `local_irq_restore(flags)` | `spin_unlock()` + `migrate_enable()` |
| `local_unlock_nested_bh(lock)` | lockdep release فقط | `spin_unlock()` |

#### مجموعة الـ Helpers / Inspection

| Macro / Function | الوصف |
|---|---|
| `local_lock_is_locked(lock)` | يرجع 1 لو الـ lock مأخوذ على الـ CPU الحالي |
| `__this_cpu_local_lock(base)` | يحوّل الـ `__percpu` pointer لـ per-CPU instance |
| `local_lock_acquire(l)` | debug: يسجّل الـ owner + lockdep |
| `local_trylock_acquire(l)` | debug: نفس `local_lock_acquire` بـ `lock_map_acquire_try` |
| `local_lock_release(l)` | debug: يمسح الـ owner + lockdep release |

#### مجموعة الـ Lock Guards (RAII)

| Guard Class | الـ Lock المستخدم | الـ Unlock المستخدم |
|---|---|---|
| `local_lock` | `local_lock()` | `local_unlock()` |
| `local_lock_irq` | `local_lock_irq()` | `local_unlock_irq()` |
| `local_lock_irqsave` | `local_lock_irqsave()` | `local_unlock_irqrestore()` |
| `local_lock_nested_bh` | `local_lock_nested_bh()` | `local_unlock_nested_bh()` |
| `local_lock_init` | `local_lock_init()` | no-op |
| `local_trylock_init` | `local_trylock_init()` | no-op |

---

### فهم المفهوم الأساسي أولاً

**الـ `local_lock`** هو آلية لحماية الـ per-CPU data structures بطريقة آمنة مع الـ lockdep. المشكلة التقليدية: كود يعمل على `preempt_disable()` لحماية per-CPU data لا يظهر في الـ lockdep graph، يعني تبقى race conditions مخفية. الـ `local_lock` حلّ المشكلة دي.

**الفرق الجوهري بين !RT و RT:**
- **!RT**: الـ lock مجرد `preempt_disable()` أو `irq_disable()` مع lockdep annotation — zero overhead في الـ fast path.
- **RT (PREEMPT_RT)**: الـ lock يتحوّل لـ per-CPU `spinlock_t` حقيقي لأن الـ preemption لا يُعطَّل على RT kernels.

---

### Group 1: Initialization Functions

هذه المجموعة مسؤولة عن تهيئة الـ lock قبل استخدامه. الفرق بين الـ static init (بـ macros زي `INIT_LOCAL_LOCK`) والـ runtime init (بـ `local_lock_init`) هو أن الـ runtime init يسجّل lockdep class جديدة.

---

#### `local_lock_init(lock)`

```c
#define local_lock_init(lock)   __local_lock_init(lock)
```

الـ macro ده بيعمل runtime initialization لـ `local_lock_t __percpu` variable. بيسجّل الـ lock مع الـ lockdep subsystem بـ `lock_class_key` جديدة، وبيصفّر الـ `owner` field في debug mode.

**Parameters:**
- `lock` — pointer لـ `local_lock_t __percpu` variable (الـ per-CPU lock).

**Return value:** لا يرجع قيمة (macro يُنفَّذ كـ statement).

**Key details:**
- يستدعي `debug_check_no_locks_freed()` للتأكد إن مفيش lock مأخوذ في النفس الذاكرة.
- على !RT: يستدعي `lockdep_init_map_type()` بـ `LD_LOCK_PERCPU` و`LD_WAIT_CONFIG`.
- على RT: يستدعي `local_spin_lock_init()` اللي في الأساس `spin_lock_init()` لكن per-CPU aware.
- الـ `static struct lock_class_key __key` جوّا الـ macro يعني كل call site بيجيب lockdep class مختلفة — ده مقصود.

**Who calls it:** أي كود محتاج يعمل dynamic init لـ local lock، مثلاً في `kmem_cache_create()` أو أي subsystem بيعمل per-CPU allocations dynamically.

**Pseudocode (!RT path):**
```
__local_lock_init(lock):
    static lock_class_key __key
    debug_check_no_locks_freed(lock, sizeof(*lock))
    lockdep_init_map_type(&lock->dep_map, "lock", &__key,
                          0, LD_WAIT_CONFIG, LD_WAIT_INV,
                          LD_LOCK_PERCPU)
    lock->owner = NULL  // if CONFIG_DEBUG_LOCK_ALLOC
```

---

#### `local_trylock_init(lock)`

```c
#define local_trylock_init(lock)    __local_trylock_init(lock)
```

Runtime init لـ `local_trylock_t __percpu`. داخلياً مجرد cast لـ `local_lock_t *` وتطبيق `__local_lock_init`. الفرق الوحيد في الـ type system وليس في الـ behavior.

**Parameters:**
- `lock` — pointer لـ `local_trylock_t __percpu`.

**Return value:** لا يرجع.

**Key details:**
- الـ `local_trylock_t` بيحتوي على field إضافي `acquired: u8` على !RT بيتراك فيه إذا كان الـ lock مأخوذ.
- على RT: الـ `local_trylock_t` هو `spinlock_t` عادي زي `local_lock_t`.

**Who calls it:** كود محتاج trylock semantics — يعني مش عايز يـblock لو الـ lock مش متاح.

---

#### `local_lock_acquire(l)` — Internal Debug Helper

```c
static inline void local_lock_acquire(local_lock_t *l)
```

بيخبر الـ lockdep إن الـ lock اتأخذ، وبيسجّل الـ current task كـ owner.

**Parameters:**
- `l` — مباشرة pointer لـ `local_lock_t` (مش per-CPU، ده الـ instance الحالي بعد `this_cpu_ptr()`).

**Return value:** void.

**Key details:**
- بيستدعي `lock_map_acquire()` اللي بتسجّل الـ lock في الـ lockdep dependency graph.
- `DEBUG_LOCKS_WARN_ON(l->owner)` — يحذّر لو في owner قديم (double lock).
- بدون `CONFIG_DEBUG_LOCK_ALLOC`: الـ function فاضية تماماً — zero overhead.

---

#### `local_trylock_acquire(l)` — Internal Debug Helper

```c
static inline void local_trylock_acquire(local_lock_t *l)
```

نفس `local_lock_acquire` لكن بيستخدم `lock_map_acquire_try()` بدل `lock_map_acquire()`. الـ try variant بتخلّي الـ lockdep يفهم إن الـ acquisition ممكن يفشل.

---

#### `local_lock_release(l)` — Internal Debug Helper

```c
static inline void local_lock_release(local_lock_t *l)
```

بيتحقق إن الـ current task هو الـ owner، يصفّر الـ owner، وبيخبر الـ lockdep بالـ release.

**Key details:**
- `DEBUG_LOCKS_WARN_ON(l->owner != current)` — يكتشف unlock من task غلط.
- بيستدعي `lock_map_release()`.

---

### Group 2: Acquire Functions

المجموعة الأساسية اللي بتأخذ الـ lock. كل function بتجمع بين ثلاث طبقات: (1) منع الـ preemption أو الـ interrupts، (2) debug/lockdep annotation، (3) sparse annotation بـ `__acquire()`.

---

#### `local_lock(lock)`

```c
#define local_lock(lock)    __local_lock(__this_cpu_local_lock(lock))
```

بيأخذ الـ per-CPU local lock عن طريق تعطيل الـ preemption. ده أبسط شكل — بيحمي من الـ preemption فقط، مش من الـ interrupts.

**Parameters:**
- `lock` — `local_lock_t __percpu *` — الـ per-CPU lock variable.

**Return value:** لا يرجع.

**Key details:**
- على !RT: `preempt_disable()` + `__local_lock_acquire()`.
- على RT: `migrate_disable()` + `spin_lock()`. يلاحظ إنه على RT الـ preemption مش بيتعطّل — الـ task ممكن يتنقل لو الـ spinlock free.
- `__this_cpu_local_lock(lock)` بيحوّل الـ `__percpu` pointer لـ local CPU instance باستخدام `this_cpu_ptr()`.
- الـ sparse annotation `__acquire(lock)` بتخلّي `sparse` يتابع الـ lock balance.

**Who calls it:** أي كود بيحمي per-CPU data من الـ preemption فقط — مثلاً `kmalloc` paths، `vmalloc` per-CPU caches.

**Pseudocode (!RT):**
```
local_lock(lock):
    ptr = this_cpu_ptr(lock)
    preempt_disable()
    // __local_lock_acquire:
    if typeof(ptr) == local_trylock_t*:
        assert(ptr->acquired == 0)
        WRITE_ONCE(ptr->acquired, 1)
    local_lock_acquire(ptr)   // lockdep only
    __acquire(ptr)            // sparse only
```

---

#### `local_lock_irq(lock)`

```c
#define local_lock_irq(lock)    __local_lock_irq(__this_cpu_local_lock(lock))
```

بيأخذ الـ local lock مع تعطيل الـ interrupts. أقوى من `local_lock()` — بيحمي من الـ preemption والـ IRQ handlers.

**Parameters:**
- `lock` — `local_lock_t __percpu *`.

**Return value:** لا يرجع.

**Key details:**
- على !RT: `local_irq_disable()` + lockdep acquire.
- على RT: نفس `__local_lock` — يعني `migrate_disable()` + `spin_lock()`. الـ IRQ disable مش موجودة على RT عشان الـ interrupt handlers بتشتغل في thread context وبتـwait على الـ spinlock عادي.
- لازم يتبعه `local_unlock_irq()` مش `local_unlock()` عشان يعمل `local_irq_enable()` مش `preempt_enable()`.

**Who calls it:** كود بيتشارك الـ per-CPU data مع الـ interrupt handlers.

---

#### `local_lock_irqsave(lock, flags)`

```c
#define local_lock_irqsave(lock, flags)     \
    __local_lock_irqsave(__this_cpu_local_lock(lock), flags)
```

نفس `local_lock_irq` لكن بيحفظ الـ IRQ state في `flags` قبل التعطيل. مفيد لو الكود ممكن يُستدعى من context المـ interrupts فيه معطّلة أصلاً.

**Parameters:**
- `lock` — `local_lock_t __percpu *`.
- `flags` — `unsigned long` — مكان حفظ الـ EFLAGS/CPSR register.

**Return value:** لا يرجع. لكن `flags` بيتعبّى.

**Key details:**
- على !RT: `local_irq_save(flags)` بيعمل `pushf; cli` على x86.
- على RT: `flags = 0` دايماً عشان الـ IRQ disable مش موجودة. استخدام `flags` بعدين في `local_unlock_irqrestore` على RT آمن لأن `irq_restore(0)` = no-op.
- هذا الـ macro هو الأكثر أماناً للـ portable code.

---

#### `local_trylock(lock)`

```c
#define local_trylock(lock)     __local_trylock(__this_cpu_local_lock(lock))
```

بيحاول يأخذ الـ lock بدون blocking. يرجع true لو نجح، false لو فشل. ده الـ API الوحيد اللي بيكون safe في NMI/HARDIRQ context — مع ملاحظة مهمة.

**Parameters:**
- `lock` — `local_trylock_t __percpu *` (مش `local_lock_t` — لازم trylock type).

**Return value:** `bool` — `true` لو الـ lock اتأخذ، `false` لو لأ.

**Key details:**
- على !RT:
  - `preempt_disable()` أولاً.
  - يقرأ `acquired` بـ `READ_ONCE()`.
  - لو `acquired == 1`: يعمل `preempt_enable()` ويرجع NULL (false).
  - لو `acquired == 0`: يكتب 1 بـ `WRITE_ONCE()` ويعمل lockdep acquire.
  - الـ race condition مستحيلة هنا لأن per-CPU data + preemption disabled = single execution context.
- على RT: لو في NMI أو HARDIRQ (`in_nmi() | in_hardirq()`) يرجع false مباشرة. وإلا `migrate_disable()` + `spin_trylock()`.
- **مهم جداً**: `local_trylock` يشتغل مع `local_trylock_t` فقط مش `local_lock_t`.

**Who calls it:** كود في NMI handlers أو interrupt context محتاج يتعاون مع process context بدون deadlock — مثلاً `perf` subsystem.

**Pseudocode (!RT):**
```
local_trylock(lock):
    preempt_disable()
    tl = this_cpu_ptr(lock)
    if READ_ONCE(tl->acquired):
        preempt_enable()
        return false
    else:
        WRITE_ONCE(tl->acquired, 1)
        local_trylock_acquire((local_lock_t*)tl)
        return true
```

---

#### `local_trylock_irqsave(lock, flags)`

```c
#define local_trylock_irqsave(lock, flags)          \
    __local_trylock_irqsave(__this_cpu_local_lock(lock), flags)
```

نفس `local_trylock` لكن لو نجح بيحفظ ويعطّل الـ interrupts. لو فشل، الـ `flags` يبقى undefined والـ IRQ state يتعمل restore.

**Parameters:**
- `lock` — `local_trylock_t __percpu *`.
- `flags` — `unsigned long` — بيتعبّى بالـ IRQ state لو الـ lock اتأخذ.

**Return value:** `bool`.

**Key details:**
- على !RT: `irq_save(flags)` أولاً → check `acquired` → لو فشل: `irq_restore(flags)` → return false.
- على RT: `flags = 0` + `local_trylock()`.
- الفرق عن `local_lock_irqsave`: لو فشل، مفيش IRQs disabled — clean failure.

---

#### `local_lock_nested_bh(lock)`

```c
#define local_lock_nested_bh(_lock)     \
    __local_lock_nested_bh(__this_cpu_local_lock(_lock))
```

بيأخذ الـ local lock من داخل softirq/BH context. الـ "nested" يشير لإن الـ BH disable بيكون موجوداً أصلاً (caller في softirq context).

**Parameters:**
- `_lock` — `local_lock_t __percpu *`.

**Return value:** لا يرجع.

**Key details:**
- على !RT: `lockdep_assert_in_softirq()` + `local_lock_acquire()` فقط — مفيش `preempt_disable()` لأن الـ BH context أصلاً مش preemptible في بعض الـ configurations.
- على RT: `lockdep_assert_in_softirq_func()` + `spin_lock()`.
- ده بيسمح بـ lockdep tracking للـ per-CPU data اللي بتتشارك بين process context (بيستخدم `local_lock`) وبين softirq context (بيستخدم `local_lock_nested_bh`).

**Who calls it:** الـ networking stack — مثلاً `local_bh_disable()` + per-CPU socket buffers.

---

### Group 3: Release Functions

مرآة لمجموعة الـ Acquire. كل function لازم تيجي في نفس السياق زي نظيرتها في الـ Acquire.

---

#### `local_unlock(lock)`

```c
#define local_unlock(lock)  __local_unlock(__this_cpu_local_lock(lock))
```

بيفك الـ local lock ويعيد تفعيل الـ preemption. لازم يتناظر مع `local_lock()`.

**Parameters:**
- `lock` — `local_lock_t __percpu *`.

**Return value:** لا يرجع.

**Key details:**
- على !RT: `__release(lock)` (sparse) + `__local_lock_release()` (lockdep + acquired=0) + `preempt_enable()`.
- على RT: `spin_unlock()` + `migrate_enable()`.
- الترتيب مهم: الـ lockdep release قبل الـ preempt_enable عشان الـ lockdep annotations تكون صح.

---

#### `local_unlock_irq(lock)`

```c
#define local_unlock_irq(lock)  __local_unlock_irq(__this_cpu_local_lock(lock))
```

بيفك الـ lock ويعيد تفعيل الـ interrupts. نظير `local_lock_irq()`.

**Key details:**
- على !RT: lockdep release + `local_irq_enable()`.
- على RT: نفس `__local_unlock()`.

---

#### `local_unlock_irqrestore(lock, flags)`

```c
#define local_unlock_irqrestore(lock, flags)            \
    __local_unlock_irqrestore(__this_cpu_local_lock(lock), flags)
```

بيفك الـ lock ويرجع الـ IRQ state للحالة اللي كانت عليها قبل الـ lock. نظير `local_lock_irqsave()`.

**Parameters:**
- `lock` — `local_lock_t __percpu *`.
- `flags` — القيمة اللي اتحفظت في `local_lock_irqsave()`.

**Key details:**
- على !RT: lockdep release + `local_irq_restore(flags)`.
- على RT: `spin_unlock()` + `migrate_enable()`. الـ `flags` بيتجاهل.

---

#### `local_unlock_nested_bh(lock)`

```c
#define local_unlock_nested_bh(_lock)   \
    __local_unlock_nested_bh(__this_cpu_local_lock(_lock))
```

بيفك الـ lock في softirq context. نظير `local_lock_nested_bh()`.

**Key details:**
- على !RT: `__release()` + `local_lock_release()` فقط — مفيش `preempt_enable()`.
- على RT: `spin_unlock()`.

---

### Group 4: Inspection / Helper Functions

---

#### `local_lock_is_locked(lock)`

```c
#define local_lock_is_locked(lock)  __local_lock_is_locked(lock)
```

بيتحقق إذا كان الـ lock مأخوذاً على الـ CPU الحالي.

**Parameters:**
- `lock` — `local_trylock_t __percpu *` — لاحظ: يشتغل فقط مع trylock type على !RT لأن `acquired` field موجود فيه بس.

**Return value:**
- على !RT: `READ_ONCE(this_cpu_ptr(lock)->acquired)` — `u8` (0 أو 1).
- على RT: `rt_mutex_owner(&this_cpu_ptr(lock)->lock) == current` — `bool`.

**Key details:**
- على !RT: **لازم الـ preemption أو الـ migration تكون disabled** قبل الاستدعاء عشان `this_cpu_ptr()` تكون stable. الـ comment في الكود صريح في ده.
- على RT: بيفحص إذا كان الـ current task هو اللي ممسك الـ rt_mutex اللي جوه الـ spinlock.
- ده للـ assertion/debugging أكثر من الـ logic الأساسية.

---

#### `__this_cpu_local_lock(base)`

```c
static __always_inline local_lock_t *__this_cpu_local_lock(local_lock_t __percpu *base)
    __returns_ctx_lock(base) __attribute__((overloadable))
{
    return this_cpu_ptr(base);
}
```

بيحوّل الـ `__percpu` pointer لـ per-CPU instance للـ CPU الحالي. ده wrapper حول `this_cpu_ptr()`.

**Parameters:**
- `base` — `local_lock_t __percpu *` — الـ per-CPU variable base address.

**Return value:** `local_lock_t *` — pointer للـ instance على الـ current CPU.

**Key details:**
- موجود بس لو `WARN_CONTEXT_ANALYSIS` معرّف. وإلا `__this_cpu_local_lock` = `this_cpu_ptr` directly.
- الـ `__attribute__((overloadable))` + الـ overloaded version لـ `local_trylock_t` بتسمح للـ compiler يفرّق بين الـ types عشان الـ `_Generic` في `__local_lock_acquire` يشتغل صح.
- `__returns_ctx_lock(base)` annotation للـ context locking analysis tool.

---

### Group 5: Lock Guards (RAII Interface)

الـ lock guards بتوفّر RAII-style locking باستخدام الـ `DEFINE_LOCK_GUARD_1` macro من `<linux/cleanup.h>`. بتسمح باستخدام `guard(local_lock)(&my_lock)` أو `scoped_guard(local_lock, &my_lock)`.

---

#### `DEFINE_LOCK_GUARD_1(local_lock, ...)` والمشتقات

```c
DEFINE_LOCK_GUARD_1(local_lock, local_lock_t __percpu,
                    local_lock(_T->lock),
                    local_unlock(_T->lock))
```

بيعرّف class اسمها `class_local_lock_t` مع constructor بيستدعي `local_lock()` و destructor بيستدعي `local_unlock()`. الـ `_T` هو pointer للـ guard struct.

**الـ Guards المعرّفة:**

| Guard | Lock Call | Unlock Call | Extra Field |
|---|---|---|---|
| `local_lock` | `local_lock(_T->lock)` | `local_unlock(_T->lock)` | — |
| `local_lock_irq` | `local_lock_irq(_T->lock)` | `local_unlock_irq(_T->lock)` | — |
| `local_lock_irqsave` | `local_lock_irqsave(_T->lock, _T->flags)` | `local_unlock_irqrestore(_T->lock, _T->flags)` | `unsigned long flags` |
| `local_lock_nested_bh` | `local_lock_nested_bh(_T->lock)` | `local_unlock_nested_bh(_T->lock)` | — |
| `local_lock_init` | `local_lock_init(_T->lock)` | no-op | — |
| `local_trylock_init` | `local_trylock_init(_T->lock)` | no-op | — |

**مثال استخدام:**
```c
DEFINE_PER_CPU(local_lock_t, my_lock);

void foo(void)
{
    /* lock يتأخذ هنا ويتفك تلقائياً عند نهاية الـ scope */
    guard(local_lock)(&my_lock);
    /* per-CPU critical section */
}

/* أو بـ explicit scope */
void bar(void)
{
    scoped_guard(local_lock_irqsave, &my_lock) {
        /* IRQs disabled هنا */
    }
    /* IRQs re-enabled automatically */
}
```

**Key details:**
- الـ `DECLARE_LOCK_GUARD_1_ATTRS` + `class_*_constructor` macros بتضيف الـ sparse/lockdep annotations (`__acquires`, `__releases`) للـ guard constructors.
- الـ `local_lock_irqsave` guard له `unsigned long flags` extra field في الـ guard struct عشان يخزّن الـ IRQ flags.

---

### رسم تخطيطي للـ Call Flow

```
              User Code
                  │
    ┌─────────────▼──────────────┐
    │     local_lock(lock)       │   ← Public API (local_lock.h)
    └─────────────┬──────────────┘
                  │
    ┌─────────────▼──────────────────────────┐
    │  __local_lock(__this_cpu_local_lock())  │  ← Internal macro
    └──────────┬─────────────────┬───────────┘
               │                 │
        ┌──────▼──────┐   ┌──────▼──────┐
        │  !RT path   │   │   RT path   │
        ├─────────────┤   ├─────────────┤
        │preempt_dis()│   │migrate_dis()│
        │lockdep acq  │   │spin_lock()  │
        └─────────────┘   └─────────────┘
```

```
              local_trylock Flow (!RT)
              ========================
              preempt_disable()
                    │
              READ_ONCE(acquired)
                    │
            ┌───────┴───────┐
            │ acquired == 1 │ acquired == 0
            │               │
      preempt_enable()   WRITE_ONCE(acquired, 1)
      return false       lockdep_acquire_try()
                         return true
```

---

### ملاحظات ختامية مهمة

1. **الـ `local_lock` مش mutex** — مفيش waiting queue. الـ thread مش بيـwait لو الـ lock مأخوذ (إلا على RT مع الـ spinlock).

2. **per-CPU semantics** — الـ lock على الـ CPU الحالي فقط. ممكن CPUs مختلفة تأخذ نفس الـ `__percpu` lock في نفس الوقت عشان كل واحد بياخد الـ instance بتاعه.

3. **الـ `local_trylock_t` مختلف** — الـ `acquired` field موجود فيه بس. استخدام `local_trylock()` مع `local_lock_t` compile error.

4. **الـ lockdep integration** — الميزة الأساسية. بدونها، `preempt_disable()` كافي. الـ `local_lock` بيكشف lock ordering violations اللي `preempt_disable()` بيخبّيها.

5. **على RT** — الـ `local_lock_irqsave` بيحفظ `flags = 0` دايماً. الكود اللي بيعتمد على الـ `flags` value على !RT لازم يكون على دراية بده.
## Phase 5: دليل الـ Debugging الشامل

الـ `local_lock` subsystem بيشتغل على مستوى الـ per-CPU locking، وده بيخليه invisible تقريباً من tools التانية زي `/proc/locks`. الـ debugging بتاعه بيعتمد أساساً على الـ `lockdep` والـ `ftrace` وفهم الـ preemption state.

---

### Software Level

#### 1. debugfs Entries

الـ `local_lock` ملوش entries مباشرة في `debugfs`، لكن الـ lockdep بيكتب فيها:

```
/sys/kernel/debug/locking/   (kernel >= 6.x بعد CONFIG_DEBUG_LOCK_ALLOC)
/sys/kernel/debug/lockdep
/sys/kernel/debug/lockdep_stats
```

**إزاي تقرأهم:**

```bash
# إحصائيات عامة للـ lockdep
cat /sys/kernel/debug/lockdep_stats

# مثال على الـ output:
# lock-classes:                          1243
# direct dependencies:                  10842
# indirect dependencies:               200351
# all direct dependencies:              10842
# dependency chains:                     3671
# in-hardirq chains:                       19
# in-softirq chains:                      142
# in-process chains:                     3510
# ...
```

**تفسير الأرقام:**
- `lock-classes` ارتفاع مفاجئ = في lock classes جديدة بتتعمل dynamically (مشكلة في الـ init).
- `dependency chains` فوق 10k = الـ graph معقد جداً، ممكن يوصل لـ timeout في الـ lockdep.

```bash
# شوف الـ lock classes المسجلة (بدون فلترة)
cat /sys/kernel/debug/lockdep | grep -i "local_lock"
```

---

#### 2. sysfs Entries

```
/sys/kernel/debug/lockdep_stats      # إحصائيات الـ lockdep
/proc/lockdep_chains                 # سلاسل الـ dependencies
/proc/lockdep_stat                   # نفس lockdep_stats بشكل مختصر
/proc/locks                          # مش بيشوف local_lock (per-CPU, invisible)
```

```bash
# إحصائيات مختصرة
cat /proc/lockdep_stat

# شوف الـ lock chains الحالية
cat /proc/lockdep_chains | head -50

# تحقق من حالة الـ preemption على كل CPU
for cpu in /sys/devices/system/cpu/cpu*/; do
    echo -n "$(basename $cpu): "
    cat ${cpu}online 2>/dev/null
done
```

---

#### 3. ftrace — Tracepoints والـ Events

الـ `local_lock` بيعتمد على `preempt_disable/enable` و `local_irq_*`، وده بيتتبع عن طريق:

```bash
# تفعيل tracing للـ preemption
echo 1 > /sys/kernel/debug/tracing/events/preemptirq/preempt_disable/enable
echo 1 > /sys/kernel/debug/tracing/events/preemptirq/preempt_enable/enable

# تفعيل IRQ tracing
echo 1 > /sys/kernel/debug/tracing/events/preemptirq/irq_disable/enable
echo 1 > /sys/kernel/debug/tracing/events/preemptirq/irq_enable/enable

# تشغيل الـ tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغل الكود اللي بتختبره ...
cat /sys/kernel/debug/tracing/trace | head -100
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**لو بتحقق من deadlock أو lock inversion:**

```bash
# preemptoff tracer — بيسجل أطول فترة preemption مقفولة
echo preemptoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغل الكود ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# irqsoff tracer — بيسجل أطول فترة IRQs معطلة
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
```

**مثال على الـ output:**

```
# tracer: preemptoff
# ---------+----------- [CPU] ------ [latency in microseconds]
  kworker/0:1-42    [000]    5.123us : preempt_disable <-local_lock_acquire
  kworker/0:1-42    [000]   12.456us : preempt_enable <-local_unlock
```

لو الـ latency فوق 100µs مع `local_lock` = في مشكلة في الكود اللي جوا الـ critical section.

---

#### 4. printk والـ Dynamic Debug

الـ `local_lock` نفسه ما بيعملش printk، لكن الـ `lockdep` بيطبع تلقائياً لو في violation.

**تفعيل dynamic debug لمناطق قريبة:**

```bash
# تفعيل debug messages لكل الـ locking subsystem
echo "module lockdep +p" > /sys/kernel/debug/dynamic_debug/control

# أو بالـ file
echo "file kernel/locking/local_lock.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ per-CPU ops
echo "file include/linux/percpu-defs.h +p" > /sys/kernel/debug/dynamic_debug/control
```

**إضافة printk يدوياً في الـ kernel code (للـ development فقط):**

```c
/* في __local_lock_acquire بعد local_lock_acquire() */
pr_debug("local_lock acquired on CPU%d by task %s[%d]\n",
         smp_processor_id(), current->comm, current->pid);
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة | متى تفعّله |
|--------|---------|-----------|
| `CONFIG_DEBUG_LOCK_ALLOC` | يفعّل `lockdep_map` داخل `local_lock_t`، يتبع الـ owner | دايماً في development |
| `CONFIG_LOCKDEP` | يفعّل كل منظومة الـ lockdep | development + CI |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ lock ordering تلقائياً | لكشف الـ deadlocks |
| `CONFIG_LOCK_STAT` | إحصائيات contentions لكل lock | performance tuning |
| `CONFIG_DEBUG_PREEMPT` | يكتشف الـ preemption imbalance | لو بتشك في preempt count |
| `CONFIG_PREEMPT_TRACER` | يفعّل `preemptoff` tracer | قياس latency |
| `CONFIG_IRQSOFF_TRACER` | يفعّل `irqsoff` tracer | قياس IRQ latency |
| `CONFIG_PREEMPT_RT` | يحوّل `local_lock` لـ `spinlock_t` حقيقي | RT kernel testing |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف double-lock وunlock بدون lock | على PREEMPT_RT |
| `CONFIG_KCSAN` | يكتشف data races بين CPUs | لو بتشك في race condition |

**كيف تبني kernel للـ debugging:**

```bash
# في .config
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_LOCKDEP=y
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_PREEMPT=y
CONFIG_LOCK_STAT=y
CONFIG_PREEMPT_TRACER=y

# بناء
make -j$(nproc) bzImage
```

---

#### 6. Subsystem-Specific Tools

الـ `local_lock` ملوش tool خاص بيه، لكن الأدوات دي مفيدة:

```bash
# شوف الـ per-CPU variables في الـ kernel image
nm vmlinux | grep -i "local_lock"

# تتبع الـ lock class في /proc
cat /proc/lockdep | grep -A5 "local_lock"

# لو PREEMPT_RT — شوف spinlock contention
cat /proc/lock_stat | grep -i "spin"

# reset الـ lock stats
echo 0 > /proc/lock_stat

# شوف الـ CPU migration tracking
cat /proc/sched_debug | grep -i "migrate"
```

---

#### 7. Common Error Messages

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|------|
| `BUG: sleeping function called from invalid context` | استدعاء دالة بتنام وانت جوا `local_lock` (preempt disabled) | شيل أي `kmalloc(GFP_KERNEL)` أو `mutex_lock()` من جوا الـ critical section |
| `WARNING: CPU: X PID: Y at kernel/locking/lockdep.c:...` مع `lock held when returning to user space` | نسيت `local_unlock()` قبل الرجوع لـ userspace | راجع كل code paths، استخدم `guard(local_lock)` بدل manual lock/unlock |
| `DEBUG_LOCKS_WARN_ON(l->owner)` | محاولة acquire لـ lock شايل owner بالفعل (double lock على نفس الـ CPU) | الكود عمل `local_lock` مرتين على نفس CPU — ممنوع |
| `DEBUG_LOCKS_WARN_ON(l->owner != current)` | الـ unlock بيتعمل من task مختلف عن اللي عمل الـ lock | local_lock مش safe للنقل بين tasks — الـ acquire والـ release لازم على نفس الـ CPU |
| `lockdep: possible recursive locking detected` | نفس الـ lock class اتأخدت مرتين في نفس الـ call chain | استخدم `local_lock_nested_bh` لو الـ nesting مقصود في softirq |
| `Pid X is trying to acquire lock ... but task is already holding lock` | lock ordering violation | حدد ترتيب ثابت للـ locks |
| `preemption imbalance` | `preempt_disable` و`preempt_enable` مش متوازنين | راجع كل exception paths، استخدم `local_lock` بدل manual `preempt_disable` |
| `local_trylock: always fails in NMI/HARDIRQ on PREEMPT_RT` | `local_trylock` من NMI أو hardirq على RT kernel | ده by design — اتعامل مع الـ failure case |

---

#### 8. Strategic Points لـ dump_stack() وWARN_ON()

```c
/* في local_lock_acquire() — تحقق من الـ owner */
static inline void local_lock_acquire(local_lock_t *l)
{
    /* نقطة استراتيجية: تحقق إن الـ CPU هو نفسه */
    WARN_ON_ONCE(l->owner && l->owner != current);

    lock_map_acquire(&l->dep_map);
    DEBUG_LOCKS_WARN_ON(l->owner);
    l->owner = current;
}

/* في كود بيستخدم local_lock — تحقق من الـ context */
void my_function(void)
{
    /* تحقق إننا مش في NMI context قبل الـ lock */
    WARN_ON_ONCE(in_nmi());

    local_lock(&my_percpu_lock);

    /* تحقق من preempt_count بعد الـ lock */
    WARN_ON_ONCE(preempt_count() == 0);  /* لازم يكون != 0 */

    /* ... critical section ... */

    local_unlock(&my_percpu_lock);

    /* تحقق إن preempt_count رجع لطبيعته */
    WARN_ON_ONCE(preempt_count() != 0 && !irqs_disabled());
}

/* لو بتشك في CPU migration أثناء الـ critical section */
void debug_migration_check(void)
{
    int cpu_before, cpu_after;

    cpu_before = get_cpu();  /* بيعمل preempt_disable */
    put_cpu();

    local_lock(&my_lock);
    cpu_after = smp_processor_id();

    /* لو اتغير الـ CPU = في bug خطير */
    WARN_ON_ONCE(cpu_before != cpu_after);

    local_unlock(&my_lock);
}
```

**نقاط dump_stack() المفيدة:**

```c
/* في __local_lock إذا الـ preempt_count غلط */
#define __local_lock(lock)               \
    do {                                 \
        preempt_disable();               \
        if (unlikely(preempt_count() != 1)) { \
            pr_err("preempt_count mismatch: %d\n", preempt_count()); \
            dump_stack();                \
        }                                \
        __local_lock_acquire(lock);      \
        __acquire(lock);                 \
    } while (0)
```

---

### Hardware Level

#### 1. التحقق من Hardware State مقابل Kernel State

الـ `local_lock` بيشتغل على مستوى CPU preemption، وده بيرتبط بـ:
- **الـ preempt_count register** (في memory مش في hardware register حقيقي)
- **الـ interrupt flag** في RFLAGS/CPSR
- **الـ CPU affinity** للـ task

```bash
# تحقق من preempt_count لكل CPU عن طريق /proc/sched_debug
cat /proc/sched_debug | grep -E "cpu#|preempt_count|nr_running"

# مثال output:
# cpu#0, 1.000 MHz
#   .nr_running                    : 2
#   .nr_migratory_task             : 1
```

```bash
# تحقق من interrupt state
# عن طريق الـ irqsoff tracer أو /proc/interrupts
watch -n1 'cat /proc/interrupts | head -5'
```

---

#### 2. Register Dump Techniques

الـ `local_lock` على non-RT kernel بيعتمد على `preempt_count` اللي بيتخزن في thread_info:

```bash
# قراءة preempt_count لـ CPU محدد عن طريق devmem2
# أولاً: ابحث عن عنوان thread_info في الـ kernel
grep "thread_info" /proc/kallsyms | head -5

# استخدام crash utility لفحص الـ preempt_count
crash vmlinux /proc/kcore
# في crash prompt:
# > p current->thread_info.preempt_count

# على PREEMPT_RT — الـ spinlock بيستخدم rt_mutex:
# قراءة حالة الـ spinlock عن طريق /proc/kcore مع crash:
# > p per_cpu(my_percpu_lock, 0)
```

```bash
# فحص الـ RFLAGS (interrupt flag) عبر /proc/PID/status على x86
# IF bit (bit 9) في EFLAGS
gdb -p $(pidof my_process) -batch -ex "info registers eflags"
# 0x246 = IF set (interrupts enabled)
# 0x46  = IF clear (interrupts disabled)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو بتتحقق من timing الـ `local_lock_irq` على hardware:

```
Oscilloscope setup لقياس IRQ disable window:
┌─────────────────────────────────────────────────────┐
│  GPIO toggle قبل local_lock_irq() وبعد local_unlock_irq()  │
│                                                     │
│  CH1: GPIO (يمثل الـ critical section)              │
│  CH2: IRQ line المعنية                              │
│                                                     │
│  Expected:                                          │
│  CH1 HIGH  → IRQ disabled (no CH2 pulses)          │
│  CH1 LOW   → IRQ enabled  (CH2 pulses normal)      │
└─────────────────────────────────────────────────────┘
```

```c
/* كود GPIO toggle للـ debugging (development boards فقط) */
static inline void debug_gpio_set(int val)
{
    /* اكتب على GPIO register مباشرة - platform specific */
    writel(val, DEBUG_GPIO_BASE + GPIO_OUT_REG);
}

void my_critical_work(void)
{
    debug_gpio_set(1);          /* CH1 HIGH */
    local_lock_irq(&my_lock);
    /* critical section */
    local_unlock_irq(&my_lock);
    debug_gpio_set(0);          /* CH1 LOW */
}
```

**Logic Analyzer Setup:**
- Sample rate: >= 100 MHz لقياس latencies صغيرة
- Trigger: rising edge على GPIO channel
- Measure: pulse width = مدة الـ critical section
- Alert: لو الـ width > 50µs = مشكلة

---

#### 4. Common Hardware Issues وpatterns الـ Kernel Log

| مشكلة Hardware | Pattern في الـ Kernel Log | التفسير |
|---------------|--------------------------|---------|
| CPU hotplug أثناء local_lock | `CPU X is dead` + `BUG: unable to handle` | الـ per-CPU variable اتوصلت من CPU مات — الـ lock ماكملش |
| Cache coherency issue على NUMA | `lockdep: possible circular locking` بشكل متقطع | الـ `acquired` field في `local_trylock_t` مش بتتزامن بسرعة |
| Spurious interrupts | `unexpected IRQ trap` داخل critical section | على non-RT kernel الـ local_lock_irq محمي، لكن على RT ممكن يحصل |
| CPU frequency scaling | latency مرتفعة في preemptoff tracer | الـ CPUfreq driver بياخد وقت في الـ transition |
| TLB shootdown storms | `scheduler: CPU X task migration spike` | الـ migrate_disable على PREEMPT_RT بيتأخر |

---

#### 5. Device Tree Debugging

الـ `local_lock` مش بيرتبط بـ Device Tree مباشرة، لكن الـ drivers اللي بتستخدمه ممكن تتأثر بـ DT configuration:

```bash
# تحقق من CPU topology في DT
cat /proc/device-tree/cpus/cpu@0/compatible
cat /proc/device-tree/cpus/cpu-map/cluster0/core0/cpu

# تحقق من الـ CPU affinity للـ interrupts
cat /proc/irq/*/smp_affinity_list

# لو driver بيستخدم local_lock وبيتعامل مع hardware interrupt:
# تحقق إن الـ interrupt affinity متوافق مع الـ CPU اللي بتشتغل عليه
grep -r "local_lock" /proc/device-tree/ 2>/dev/null || echo "No DT entries for local_lock"
```

```bash
# فحص DT لـ CPU count يتطابق مع kernel's NR_CPUS
dtc -I fs /proc/device-tree 2>/dev/null | grep "device_type.*cpu" | wc -l
nproc
# الرقمين لازم يتطابقوا
```

---

### Practical Commands

#### أوامر جاهزة للـ Copy

**1. فحص الـ lockdep violations:**

```bash
# شغّل وراقب الـ dmesg في نفس الوقت
dmesg -w | grep -E "BUG|WARNING|lockdep|local_lock|preempt" &
DMESG_PID=$!

# شغّل الكود المشبوه هنا
./my_test_program

kill $DMESG_PID
```

**2. تفعيل preemptoff tracer وقياس أقصى latency:**

```bash
#!/bin/bash
# setup_preemptoff_tracer.sh

TRACE=/sys/kernel/debug/tracing

echo preemptoff > $TRACE/current_tracer
echo 0 > $TRACE/tracing_max_latency
echo 1 > $TRACE/tracing_on

echo "Running workload..."
./my_workload

echo 0 > $TRACE/tracing_on

echo "=== Max preempt-off latency ==="
cat $TRACE/tracing_max_latency
echo "microseconds"

echo "=== Trace at max latency ==="
cat $TRACE/trace | head -50
```

**مثال output:**

```
=== Max preempt-off latency ===
47
microseconds
=== Trace at max latency ===
# tracer: preemptoff
# preemptoff latency trace v1.1.5 on 6.8.0
# -----------------------------------------------------------------------
# latency: 47 us, #17/17, CPU#2 | (M:preempt VP:0, KP:0, SP:0 HP:0)
#    -----------------
#    | task: kworker/2:1-89 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
 => started at: local_lock_acquire
 => ended at:   local_unlock
#   _------=> CPU#
  kworker/2:1-89  [002]    0.000us : preempt_disable <-local_lock_acquire
  kworker/2:1-89  [002]   47.123us : preempt_enable <-local_unlock
```

**3. فحص lock stats لمعرفة الـ contention:**

```bash
# تفعيل lock stats (يحتاج CONFIG_LOCK_STAT=y)
echo 1 > /proc/sys/kernel/lock_stat

# شغّل الـ workload
./my_workload

# اعرض الـ stats مرتبة بالـ contention
cat /proc/lock_stat | sort -k3 -rn | head -20

# reset
echo 0 > /proc/lock_stat
```

**مثال output:**

```
class name          con-bounces    contentions   waittime-min   waittime-max waittime-total  acq-bounces   acquisitions   holdtime-min   holdtime-max holdtime-total
my_subsys:local_lock:W:          0             0              0              0              0             15           3847              0.11            47.12          12847.33
```

**4. فحص الـ per-CPU state مع crash:**

```bash
# على live system
crash /proc/kcore vmlinux

# في crash:
crash> p per_cpu(my_local_lock, 0)    # CPU 0
crash> p per_cpu(my_local_lock, 1)    # CPU 1
crash> foreach cpu 'p per_cpu(my_local_lock, cpu)'
```

**5. اكتشاف double-lock (local_lock مرتين على نفس CPU):**

```bash
# فعّل CONFIG_DEBUG_LOCK_ALLOC وراقب
dmesg | grep "DEBUG_LOCKS_WARN_ON(l->owner)"

# أو بشكل تفاعلي
while true; do
    dmesg -c | grep -q "WARN_ON" && {
        echo "Lock violation detected at $(date)"
        dmesg | grep -A20 "WARN_ON" | tail -25
        break
    }
    sleep 0.1
done
```

**6. التحقق من PREEMPT_RT behavior:**

```bash
# تحقق من نوع الـ kernel
uname -r | grep -q "rt" && echo "PREEMPT_RT kernel" || echo "Standard kernel"

# على RT kernel — local_lock هو spinlock حقيقي
# شوف الـ spinlock contention
cat /proc/lock_stat | grep spinlock | sort -k3 -rn | head -10

# تحقق من migrate_disable count
cat /proc/sched_debug | grep "nr_spread_over"
```

**7. سكريبت شامل للـ diagnosis:**

```bash
#!/bin/bash
# diagnose_local_lock.sh

echo "=== Kernel Version ==="
uname -r

echo -e "\n=== PREEMPT_RT Check ==="
grep -q "PREEMPT_RT" /proc/version_signature 2>/dev/null && \
    echo "RT Kernel: local_lock = spinlock" || \
    echo "Standard Kernel: local_lock = preempt_disable"

echo -e "\n=== lockdep Stats ==="
cat /proc/lockdep_stat 2>/dev/null || echo "lockdep not enabled (need CONFIG_LOCKDEP=y)"

echo -e "\n=== Recent Lock Warnings ==="
dmesg | grep -E "lockdep|WARN_ON.*owner|BUG.*lock|preempt imbalance" | tail -20

echo -e "\n=== Preemption Latency (requires CONFIG_PREEMPT_TRACER) ==="
cat /sys/kernel/debug/tracing/tracing_max_latency 2>/dev/null && \
    echo "microseconds max preempt-off" || \
    echo "Tracer not available"

echo -e "\n=== Lock Classes Count ==="
cat /proc/lockdep_stat 2>/dev/null | grep "lock-classes"

echo -e "\n=== CPUs Online ==="
cat /sys/devices/system/cpu/online

echo "=== Done ==="
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: RK3562 — Industrial Gateway — Deadlock مع `local_lock_irq` و IRQ Handler

#### العنوان
**Deadlock خفي في industrial gateway بسبب استخدام `local_lock_irq` داخل IRQ handler**

#### السياق
شركة بتطور industrial IoT gateway على RK3562. الجهاز بيجمع بيانات من 8 أجهزة استشعار عبر SPI وبيبعتها على شبكة Ethernet. الـ driver بيستخدم `local_lock_irq` لحماية per-CPU ring buffer بيتشارك بين kernel thread وـ SPI IRQ handler.

#### المشكلة
الجهاز بيـ hang تماماً بعد تشغيله لمدة تقريباً 20 دقيقة. الـ watchdog بيعمل reset. في الـ serial console بيظهر:

```
INFO: rcu_sched self-detected stall on CPU 2
INFO: task kworker/2:1 blocked for more than 120 seconds
```

#### التحليل
الـ engineer بيبص على الكود وبيلاقي:

```c
/* SPI IRQ handler — يتنفذ في hardirq context */
static irqreturn_t spi_data_irq(int irq, void *dev_id)
{
    unsigned long flags;
    /* BUG: local_lock_irq هنا في hardirq context */
    local_lock_irq(&my_percpu_lock);
    /* ... copy data to ring buffer ... */
    local_unlock_irq(&my_percpu_lock);
    return IRQ_HANDLED;
}

/* kernel thread */
static int sensor_thread(void *data)
{
    unsigned long flags;
    local_lock_irq(&my_percpu_lock);  /* <-- disable IRQs + lock */
    /* ... process ring buffer ... */
    local_unlock_irq(&my_percpu_lock);
}
```

الـ `__local_lock_irq` في `local_lock_internal.h` بيعمل `local_irq_disable()` أول حاجة:

```c
#define __local_lock_irq(lock)
    do {
        local_irq_disable();          /* disable interrupts */
        __local_lock_acquire(lock);   /* acquire the lock */
        __acquire(lock);
    } while (0)
```

المشكلة: لما الـ `sensor_thread` بيمسك الـ lock ويعطل الـ IRQs، الـ `spi_data_irq` ما بيقدر يتنفذ. لكن على PREEMPT_RT، الـ `local_lock_irq` بيتحول لـ `spin_lock` عادي مش بيعطل الـ IRQs — فالـ interrupt handler بيشتغل وبيحاول يمسك نفس الـ spinlock اللي بيمسكه الـ thread → **deadlock**.

على non-RT kernel، الـ `local_irq_disable()` بيحمي من الـ IRQ handler على نفس الـ CPU، لكن الكود اتصمم غلط لأن الـ IRQ handler نفسه بيحاول يمسك الـ lock.

**الدليل من lockdep:**
```
======================================================
WARNING: possible circular locking dependency detected
[ INFO: possible unsafe locking scenario detected ]
CPU0                    CPU1
----                    ----
local_lock_irq(&my_percpu_lock);
                        <interrupt>
                          local_lock_irq(&my_percpu_lock);  ← spin
```

#### الحل
الـ IRQ handler **لا يحتاج lock** لأنه على نفس الـ CPU والـ interrupts معطلة أصلاً. الحل الصح:

```c
/* IRQ handler — لا يحتاج local_lock لأن local_lock_irq في الـ thread
   بيعطل الـ interrupts على نفس الـ CPU */
static irqreturn_t spi_data_irq(int irq, void *dev_id)
{
    /* نكتب مباشرة في الـ per-CPU buffer — محمي بتعطيل الـ IRQs في الـ thread */
    __this_cpu_write(ring_buf.head, new_head);
    return IRQ_HANDLED;
}

/* kernel thread */
static int sensor_thread(void *data)
{
    local_lock_irq(&my_percpu_lock);  /* disable IRQs + lock */
    /* ... read ring buffer safely ... */
    local_unlock_irq(&my_percpu_lock);
}
```

لو لازم الـ IRQ handler يمسك lock، نستخدم `local_trylock_irqsave` بدلاً من ذلك، أو نعيد تصميم الـ data path.

#### الدرس المستفاد
**`local_lock_irq` مش بيحمي من نفس الـ IRQ handler على نفس الـ CPU على PREEMPT_RT**. على PREEMPT_RT الـ `local_lock_irq` = `spin_lock` عادي، فالـ IRQ handler ممكن يحاول يمسكه ويـ deadlock. القاعدة: لو الـ IRQ handler ومسك نفس الـ per-CPU data، استخدم تعطيل الـ IRQs في الجانبين أو استخدم لـ lock أخف.

---

### السيناريو 2: STM32MP1 — IoT Sensor Hub — Migration Bug مع `local_lock` على SMP

#### العنوان
**Data corruption في per-CPU cache على STM32MP1 بسبب migration بين cores بعد `local_lock`**

#### السياق
board بيشغل STM32MP1 (Cortex-A7 dual-core) كـ IoT sensor hub. الكود بيستخدم `local_lock` لحماية per-CPU statistics counter لعمليات I2C. الـ board بيشغل يومين متواصلين وبعدين الـ statistics بتبان corrupted.

#### المشكلة
الـ counters بتتجمع على CPU0 وبتكون صفر دايماً على CPU1، أو بيحصل double-counting. الـ sysfs interface بيقرأ أرقام غلط.

#### التحليل
الكود الخاطئ:

```c
DEFINE_PER_CPU(struct i2c_stats, cpu_stats);
DEFINE_PER_CPU(local_lock_t, stats_lock);

void update_i2c_stats(u32 bytes)
{
    struct i2c_stats *stats;

    local_lock(&stats_lock);          /* preempt_disable() فقط */

    stats = get_cpu_ptr(&cpu_stats);  /* BUG: get_cpu_ptr بيعمل preempt_disable مرة تانية */
    stats->bytes += bytes;
    put_cpu_ptr(&cpu_stats);          /* preempt_enable() — يرفع count لـ 0 */

    /* هنا preemption ممكن يحصل قبل local_unlock! */
    local_unlock(&stats_lock);
}
```

من `local_lock_internal.h`:

```c
#define __local_lock(lock)
    do {
        preempt_disable();            /* preempt_count++ */
        __local_lock_acquire(lock);
        __acquire(lock);
    } while (0)

#define __local_unlock(lock)
    do {
        __release(lock);
        __local_lock_release(lock);
        preempt_enable();             /* preempt_count-- */
    } while (0)
```

المشكلة: الـ `get_cpu_ptr` و `put_cpu_ptr` بيعملوا `preempt_disable/enable` إضافية. لو الـ `put_cpu_ptr` رفع الـ preempt_count لصفر قبل `local_unlock`، الـ scheduler ممكن يـ migrate الـ thread لـ CPU آخر → الـ `stats` pointer بيبقى يشاور على بيانات الـ CPU الأول بينما احنا على CPU تاني → **corruption**.

**التحقق:**
```bash
# على RK3562 أو STM32MP1
echo 1 > /proc/sys/kernel/panic_on_warn
# تفعيل lockdep
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_PROVE_LOCKING=y
```

الـ lockdep بيطلع:
```
WARNING: CPU: 1 PID: 847 at kernel/sched/core.c:...
preemption imbalance after local_unlock
```

#### الحل

```c
void update_i2c_stats(u32 bytes)
{
    struct i2c_stats *stats;

    local_lock(&stats_lock);
    /* استخدم this_cpu_ptr بدل get_cpu_ptr — لا يغير preempt_count */
    stats = this_cpu_ptr(&cpu_stats);
    stats->bytes += bytes;
    /* لا put_cpu_ptr هنا */
    local_unlock(&stats_lock);
}
```

أو استخدام الـ `DEFINE_LOCK_GUARD_1` المعرف في `local_lock.h`:

```c
void update_i2c_stats(u32 bytes)
{
    /* guard يتأكد من release صح حتى لو في error path */
    guard(local_lock)(&stats_lock);
    this_cpu_ptr(&cpu_stats)->bytes += bytes;
}
```

#### الدرس المستفاد
**`local_lock` بيضمن non-preemption مش بيضمن non-migration إلا من خلال preempt_disable**. داخل الـ critical section استخدم `this_cpu_ptr` مش `get_cpu_ptr` — الأخير بيعمل preempt_disable إضافي ممكن يبوظ الـ balance. الـ `DEFINE_LOCK_GUARD_1` في `local_lock.h` بيساعد على تجنب أخطاء الـ unlock في الـ error paths.

---

### السيناريو 3: i.MX8QM — Automotive ECU — `local_trylock` في NMI Context على PREEMPT_RT

#### العنوان
**الـ `local_trylock` بيفشل دايماً في NMI handler على i.MX8QM Automotive ECU**

#### السياق
automotive ECU على i.MX8QM (Cortex-A72 quad-core) بيشغل PREEMPT_RT kernel لضمان real-time latency. الكود بيحاول يجمع performance statistics من NMI watchdog handler باستخدام `local_trylock`.

#### المشكلة
الـ NMI-based profiler دايماً بيطلع miss rate = 100%. بيبان إنه ما بيقدر أبداً يمسك الـ lock من NMI context.

#### التحليل
الكود:

```c
/* NMI watchdog handler */
static void nmi_profile_handler(struct perf_event *event, ...)
{
    unsigned long flags;

    /* محتاج يقرأ per-CPU ring buffer */
    if (local_trylock_irqsave(&profile_lock, flags)) {
        record_sample(event);
        local_unlock_irqrestore(&profile_lock, flags);  /* BUG */
    } else {
        missed_samples++;
    }
}
```

من `local_lock_internal.h` — الجزء الخاص بـ PREEMPT_RT:

```c
#define __local_trylock(lock)
    __try_acquire_ctx_lock(lock, context_unsafe(({
        int __locked;

        if (in_nmi() | in_hardirq()) {
            __locked = 0;              /* دايماً يفشل في NMI/hardirq على RT */
        } else {
            migrate_disable();
            __locked = spin_trylock((lock));
            if (!__locked)
                migrate_enable();
        }
        __locked;
    })))
```

على PREEMPT_RT، `local_trylock` **عمداً بيفشل دايماً في NMI وـ HARDIRQ context**. الـ doc في `local_lock.h` واضح:

```c
/**
 * local_trylock - Try to acquire a per CPU local lock
 *
 * The function can be used in any context such as NMI or HARDIRQ. Due to
 * locking constrains it will _always_ fail to acquire the lock in NMI or
 * HARDIRQ context on PREEMPT_RT.
 */
```

السبب: على PREEMPT_RT الـ `local_lock_t` = `spinlock_t` عادي. الـ spinlock على PREEMPT_RT بيتحول لـ rt_mutex. الـ rt_mutex **لا يمكن** مسكها من NMI لأنها ممكن تحتاج sleep (priority inheritance).

#### الحل
لجمع البيانات من NMI context، لازم نستخدم آلية lock-free أو atomic:

```c
/* استخدم per-CPU atomic counter بدل local_lock */
DEFINE_PER_CPU(atomic64_t, sample_count);
DEFINE_PER_CPU(unsigned long [SAMPLE_RING_SIZE], sample_ring);
DEFINE_PER_CPU(unsigned int, ring_head);

static void nmi_profile_handler(struct perf_event *event, ...)
{
    /* atomic operations آمنة من NMI — لا تحتاج lock */
    unsigned int head = this_cpu_read(ring_head);
    this_cpu_write(sample_ring[head % SAMPLE_RING_SIZE],
                   (unsigned long)event->attr.config);
    this_cpu_inc(ring_head);
    atomic64_inc(this_cpu_ptr(&sample_count));
}

/* consumer في process context */
static void read_samples(void)
{
    local_lock(&profile_lock);   /* آمن هنا */
    /* ... process ring ... */
    local_unlock(&profile_lock);
}
```

أو لو لازم non-RT kernel فقط:

```c
/* non-RT فقط: local_trylock ممكن ينجح في hardirq */
#ifndef CONFIG_PREEMPT_RT
    if (local_trylock(&profile_lock)) { ... }
#else
    /* RT path: استخدم lock-free */
    this_cpu_inc(sample_count);
#endif
```

#### الدرس المستفاد
**الـ comment في `local_lock.h` مش decoration — اقرأه قبل ما تستخدم `local_trylock` في NMI**. على PREEMPT_RT الـ `local_trylock` في NMI/HARDIRQ = فشل مضمون. الـ NMI handlers دايماً محتاجة lock-free data structures أو per-CPU atomics بدون أي form من الـ locking.

---

### السيناريو 4: AM62x — Android TV Box — `local_lock_nested_bh` خارج softirq Context

#### العنوان
**Kernel panic على AM62x Android TV Box بسبب استخدام `local_lock_nested_bh` خارج softirq**

#### السياق
Android TV box على TI AM62x (Cortex-A53) بيشغل Android 13. الـ USB video capture driver بيستخدم `local_lock_nested_bh` لحماية per-CPU DMA buffer pool.

#### المشكلة
الجهاز بيـ panic فجأة أثناء streaming 4K video:

```
kernel BUG at kernel/locking/local_lock.c:XX!
Oops: BUG, code=1
PC: lockdep_assert_in_softirq+0x...
Call trace:
  __local_lock_nested_bh+0x...
  usb_video_alloc_buffer+0x...
  usb_video_ioctl+0x...           ← process context!
  v4l2_ioctl+0x...
```

#### التحليل
الكود الخاطئ:

```c
/* استُدعي من ioctl — process context */
static int usb_video_alloc_buffer(struct usb_video_dev *dev)
{
    struct dma_pool_entry *entry;

    /* BUG: nested_bh يتطلب softirq context! */
    local_lock_nested_bh(&dma_pool_lock);
    entry = __alloc_from_pool(this_cpu_ptr(&dma_pool));
    local_unlock_nested_bh(&dma_pool_lock);
    return 0;
}
```

من `local_lock_internal.h`:

```c
#define __local_lock_nested_bh(lock)
    do {
        lockdep_assert_in_softirq();   /* ← هنا الـ BUG يظهر */
        local_lock_acquire((lock));
        __acquire(lock);
    } while (0)
```

و`lockdep_assert_in_softirq()` بتعمل:

```c
/* تتحقق إننا فعلاً في softirq context */
static inline void lockdep_assert_in_softirq(void)
{
    WARN_ON_ONCE(!in_softirq() || in_irq() || in_nmi());
}
```

المقصود من `local_lock_nested_bh`: يُستخدم في softirq context لما عندك lock مسكته بـ `local_bh_disable()` في process context (اللي بيعطل softirqs) وعايز تمسكه تاني nested من softirq على نفس الـ CPU. ده **مش** بديل لـ `local_lock`.

#### الحل
لازم تختار الـ primitive الصح:

```c
/* لو الكود بيتنفذ في process context و softirq context معاً */
DEFINE_PER_CPU(local_lock_t, dma_pool_lock);

/* في process context */
static int usb_video_alloc_buffer_process(...)
{
    local_lock_bh(&dma_pool_lock);   /* disable BH + lock */
    entry = __alloc_from_pool(this_cpu_ptr(&dma_pool));
    local_unlock_bh(&dma_pool_lock);
}

/* في softirq context — نفس الـ lock */
static void usb_video_softirq_handler(...)
{
    local_lock(&dma_pool_lock);      /* preempt_disable فقط — BH معطل أصلاً */
    entry = __alloc_from_pool(this_cpu_ptr(&dma_pool));
    local_unlock(&dma_pool_lock);
}
```

أو لو الـ softirq handler محتاج يتشارك data مع handler تاني من نفس الـ softirq type على نفس الـ CPU:

```c
/* nested_bh الاستخدام الصح */
/* process context: */
local_lock_bh(&outer_lock);    /* disable BH + outer lock */
    /* ... */
    /* softirq context (nested): */
    local_lock_nested_bh(&inner_lock);   /* inner lock فقط — BH معطل */
    local_unlock_nested_bh(&inner_lock);
    /* ... */
local_unlock_bh(&outer_lock);
```

#### الدرس المستفاد
**`local_lock_nested_bh` ليس بديلاً لـ `local_lock`**. هو للـ nested locking داخل softirq context فقط. الـ `lockdep_assert_in_softirq()` في `__local_lock_nested_bh` بتكتشف الخطأ مبكراً — فعّل `CONFIG_PROVE_LOCKING` دايماً في development builds على AM62x وغيره.

---

### السيناريو 5: Allwinner H616 — Custom Board Bring-Up — `local_lock_irqsave` مع DEFINE_LOCK_GUARD_1

#### العنوان
**IRQ flags ما بترتجعش صح على Allwinner H616 بسبب استخدام خاطئ لـ Lock Guard**

#### السياق
custom board bring-up على Allwinner H616 (Cortex-A53 quad-core) كـ smart display controller. الـ engineer بيستخدم الـ `DEFINE_LOCK_GUARD_1` macros المعرفة في `local_lock.h` لـ automatic lock release.

#### المشكلة
بعض الأحيان الـ HDMI output بيتجمد لثواني. الـ trace بيكشف إن الـ interrupts بتتعطل ومابترجعش بشكل صح في error paths.

#### التحليل
الكود الخاطئ:

```c
DEFINE_PER_CPU(local_lock_t, hdmi_lock);
DEFINE_PER_CPU(struct hdmi_state, hdmi_cpu_state);

static int process_hdmi_event(struct hdmi_dev *dev, u32 event)
{
    unsigned long flags;
    struct hdmi_state *state;

    /* استخدام guard — بس بشكل غلط */
    local_lock_irqsave(&hdmi_lock, flags);

    state = this_cpu_ptr(&hdmi_cpu_state);

    if (state->suspended) {
        /* BUG: return بدون unlock — لكن flags ضاعت */
        return -EBUSY;
    }

    /* ... process event ... */
    local_unlock_irqrestore(&hdmi_lock, flags);
    return 0;
}
```

الـ `local_unlock_irqrestore` ما اتنفذتش في حالة الـ early return → الـ interrupts بقت معطلة على هذا الـ CPU.

الحل الصح هو استخدام الـ `DEFINE_LOCK_GUARD_1` الموجود في `local_lock.h`:

```c
/* من local_lock.h */
DEFINE_LOCK_GUARD_1(local_lock_irqsave, local_lock_t __percpu,
        local_lock_irqsave(_T->lock, _T->flags),
        local_unlock_irqrestore(_T->lock, _T->flags),
        unsigned long flags)
```

الـ guard ده بيخزن الـ `flags` جوا struct الـ `_T` ويعمل `local_unlock_irqrestore` أوتوماتيكي عند نهاية الـ scope.

#### الحل

```c
static int process_hdmi_event(struct hdmi_dev *dev, u32 event)
{
    struct hdmi_state *state;

    /* guard يعمل irqsave عند الدخول وirqrestore عند الخروج تلقائياً */
    guard(local_lock_irqsave)(&hdmi_lock);

    state = this_cpu_ptr(&hdmi_cpu_state);

    if (state->suspended)
        return -EBUSY;  /* الـ guard بيعمل irqrestore تلقائياً هنا */

    state->last_event = event;
    hdmi_update_output(state);
    return 0;  /* والهنا كمان */
}
```

لو محتاج manual flags management:

```c
static int process_hdmi_event(struct hdmi_dev *dev, u32 event)
{
    unsigned long flags;
    struct hdmi_state *state;
    int ret = 0;

    local_lock_irqsave(&hdmi_lock, flags);

    state = this_cpu_ptr(&hdmi_cpu_state);

    if (state->suspended) {
        ret = -EBUSY;
        goto out;        /* اعمل goto بدل return مباشر */
    }

    state->last_event = event;
    hdmi_update_output(state);

out:
    local_unlock_irqrestore(&hdmi_lock, flags);
    return ret;
}
```

**التحقق السريع على H616:**

```bash
# تفعيل detection لـ irq imbalance
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_LOCKDEP=y

# بعد reproduce:
dmesg | grep -E "(lock|irq|preempt)" | tail -20

# فحص حالة الـ interrupts على كل CPU
cat /proc/interrupts | head -5
# لو CPU2 column فيها أعداد صغيرة مقارنة بالباقي → interrupt starvation
```

**الـ `DECLARE_LOCK_GUARD_1_ATTRS` في local_lock.h:**

```c
DECLARE_LOCK_GUARD_1_ATTRS(local_lock_irqsave,
    __acquires(_T),
    __releases(*(local_lock_t __percpu **)_T))
```

ده بيضيف **sparse annotations** تساعد في static analysis — لو استخدمت `sparse` على الكود ده هتلاقي warning للـ early return بدون unlock.

#### الدرس المستفاد
**دايماً استخدم `guard(local_lock_irqsave)` بدل manual lock/unlock** لما يكون في multiple return paths. الـ `DEFINE_LOCK_GUARD_1` في `local_lock.h` مصمم خصيصاً لإزالة bugs زي ده. على boards زي H616 اللي بيتعمل عليها bring-up، فعّل `CONFIG_DEBUG_LOCK_ALLOC` دايماً لأن الـ `local_lock_acquire` بتسجل الـ owner وبتكتشف imbalance فوراً.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ `local_lock` بالتفصيل:

| المقال | الوصف |
|--------|--------|
| [Local locks in the kernel](https://lwn.net/Articles/828477/) | المقال الرئيسي اللي شرح الـ `local_lock` لما اتدمج في kernel 5.8 — بيغطي الـ design والـ motivation وازاي الـ API بيشتغل |
| [Introduce local_lock() — patch series](https://lwn.net/Articles/821663/) | الـ patch series الأصلية اللي قدمت الـ `local_lock` — بتشرح المشكلة مع `preempt_disable()` وإيه اللي `local_lock` بيحله |
| [Nested bottom-half locking for realtime kernels](https://lwn.net/Articles/978189/) | بيشرح الـ `local_lock_nested_bh()` اللي اتضاف لاحقاً — مشكلة الـ `local_bh_disable()` على PREEMPT_RT وإزاي الـ nested BH locking حلها |
| [Local locks in the kernel (spinlocks vs. RT workloads)](https://lwn.net/Articles/828616/) | تغطية تانية للـ `local_lock` مع تركيز على الفرق بين السلوك على non-RT وRT kernels |

---

### التوثيق الرسمي للـ kernel

#### ملفات `Documentation/`

```
Documentation/locking/locktypes.rst
```

- الصفحة الرسمية على: [docs.kernel.org/locking/locktypes.html](https://docs.kernel.org/locking/locktypes.html)
- بتشرح كل أنواع الـ locks في الـ kernel وإزاي `local_lock` بيختلف عنهم
- بتغطي الـ mapping الكامل بين `local_lock` operations والـ primitives الأصلية:

```
local_lock(&llock)            →  preempt_disable()
local_unlock(&llock)          →  preempt_enable()
local_lock_irq(&llock)        →  local_irq_disable()
local_unlock_irq(&llock)      →  local_irq_enable()
local_lock_irqsave(&llock, f) →  local_irq_save(f)
```

```
Documentation/kernel-hacking/locking.rst
```

- الصفحة الرسمية على: [docs.kernel.org/kernel-hacking/locking.html](https://docs.kernel.org/kernel-hacking/locking.html)
- الـ "Unreliable Guide To Locking" — دليل شامل للـ locking في الـ kernel بشكل عام

```
Documentation/core-api/real-time/differences.rst
```

- الصفحة على: [kernel.org/doc/html/next/core-api/real-time/differences.html](https://www.kernel.org/doc/html/next/core-api/real-time/differences.html)
- بيشرح إزاي الـ PREEMPT_RT kernel بيختلف في التعامل مع الـ locking والـ local_lock

#### الـ header files المباشرة في الكود

```
include/linux/local_lock.h
include/linux/local_lock_internal.h
include/linux/percpu-defs.h
include/linux/lockdep.h
```

---

### الـ Commits المهمة

#### إدخال الـ `local_lock` الأصلي

- **Patch series v2**: [lore.kernel.org — Introduce local_lock()](https://lore.kernel.org/all/20200525170756.i3jim5wqon4k5dky@linutronix.de/T/)
  - قدمها **Thomas Gleixner** في مايو 2020
  - اتدمجت في Linux **5.8**
- **tip-bot commit announcement على LKML**: [lkml.org/lkml/2020/6/1/318](https://lkml.org/lkml/2020/6/1/318)
  - الـ commit الرسمي اللي دخل على `tip/locking/core`

#### مثال تطبيق عملي — zram

- [PATCH v3 7/7 — zram: Use local lock to protect per-CPU data](https://lore.kernel.org/lkml/20200527201119.1692513-8-bigeasy@linutronix.de/)
  - بيوضح إزاي الـ subsystems بتنقل من `preempt_disable()` لـ `local_lock`

#### الـ PREEMPT-RT locking infrastructure

- [patch V5 — PREEMPT-RT locking infrastructure (72 patches)](https://lore.kernel.org/all/20210815211301.969975279@linutronix.de/t/)
  - الـ series الكبيرة اللي فيها الـ `local_lock` على RT

---

### مناقشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [lore.kernel.org — patch v2 series](https://lore.kernel.org/all/20200525170756.i3jim5wqon4k5dky@linutronix.de/T/) | النقاش الأصلي لإدخال الـ `local_lock` |
| [patchwork.kernel.org — mm: protect local lock sections with rcu_read_lock](https://patchwork.kernel.org/project/linux-mm/patch/20220222144907.023121407@redhat.com/) | مثال نقاش على استخدام `local_lock` مع الـ memory management |
| [lkml.org](https://lkml.org/) | أرشيف الـ LKML الكامل — ابحث بـ "local_lock" |

---

### الـ KernelNewbies

**الـ `local_lock` اتدمج في Linux 5.8** — صفحة الـ release على:

- [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8)
  - بتلخص كل التغييرات اللي دخلت في 5.8 بما فيها الـ `local_lock`

**صفحات تانية مفيدة على kernelnewbies:**

- [KernelGlossary](https://kernelnewbies.org/KernelGlossary) — شرح المصطلحات زي per-CPU، preemption، softirq
- [Linux_6.12](https://kernelnewbies.org/Linux_6.12) — أحدث التغييرات في الـ locking subsystem

---

### الـ eLinux.org

ما فيش صفحة مخصصة لـ `local_lock` على eLinux، لكن الصفحات دي مفيدة للسياق العام:

- [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) — موارد عامة لتطوير الـ kernel
- [elinux.org/Memory_Management](https://elinux.org/Memory_Management) — سياق استخدام الـ per-CPU data

---

### الكتب المُوصى بيها

#### Linux Device Drivers (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 5 — *Concurrency and Race Conditions*
- بيغطي الـ `spinlock_t`، `preempt_disable()`، والـ per-CPU data
- ملاحظة: الكتاب قديم (kernel 2.6) — الـ `local_lock` مش موجود فيه لكن الأساسيات صحيحة

#### Linux Kernel Development (Robert Love)
- **الإصدار**: الطبعة الثالثة (kernel 2.6.34)
- **الفصول المهمة**:
  - Chapter 9 — *An Introduction to Kernel Synchronization*
  - Chapter 10 — *Kernel Synchronization Methods*
- بيشرح `preempt_disable()`، `local_irq_disable()`، والـ per-CPU variables اللي الـ `local_lock` بيبني عليها

#### Embedded Linux Primer (Christopher Hallinan)
- **الطبعة الثانية**
- **الفصل المهم**: Chapter 16 — *Kernel Debugging Techniques* + أي فصل بيتكلم عن PREEMPT_RT
- مفيد لفهم سياق الـ real-time extensions اللي أدت لظهور الـ `local_lock`

#### Professional Linux Kernel Architecture (Wolfgang Mauerer)
- الفصول اللي بتتكلم عن SMP وlocking مفيدة جداً كمكمل

---

### Search Terms للبحث عن معلومات أكتر

```
local_lock linux kernel
local_lock_t per-cpu locking
local_lock PREEMPT_RT spinlock
local_lock vs preempt_disable kernel
local_lock_nested_bh softirq realtime
migrate_disable local_lock RT
lockdep local_lock per-CPU validation
DEFINE_PER_CPU local_lock_t
local_lock_irqsave irqrestore kernel
Thomas Gleixner local_lock locking/core
```

```bash
# البحث في سورس الكرنل مباشرة
git log --oneline --all -- include/linux/local_lock.h
git log --oneline --all -- include/linux/local_lock_internal.h
git log --grep="local_lock" --oneline | head -20

# البحث عن الاستخدام في الكود
grep -r "local_lock\b" kernel/ --include="*.c" | head -20
grep -r "DEFINE_PER_CPU.*local_lock_t" . --include="*.c"
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `local_lock` مش بيوفر functions قابلة لـ kprobe بشكل مباشر لأنه macro-only API، لكن الـ internal helper `local_lock_acquire` و`local_lock_release` هي `static inline` functions موجودة في `local_lock_internal.h`. أحسن طريقة نراقب بيها الـ per-CPU locking activity هي إننا نعمل **tracepoint** من خلال `tracepoint/preemptirq/preempt_disable` — لأن كل `local_lock()` بيعمل `preempt_disable()` كأول خطوة في non-RT kernels.

**الـ** hook الأنسب هنا هو tracepoint الجاهز `preempt_disable` اللي بيتفعّل في نفس اللحظة اللي بتتعمل فيها أي `local_lock()` — وده بيخلينا نشوف مين بيطلب الـ preemption disable ومن أي CPU.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * local_lock_tracer.c
 *
 * Trace preempt_disable events — these are triggered by local_lock()
 * on non-PREEMPT_RT kernels as the first operation inside __local_lock().
 *
 * Build: add to a Makefile with obj-m := local_lock_tracer.o
 * Load:  insmod local_lock_tracer.ko
 * Watch: dmesg -w
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/tracepoint.h>   /* for_each_kernel_tracepoint, tracepoint_probe_register */
#include <linux/preempt.h>      /* preempt_count() */
#include <linux/sched.h>        /* current, task_comm_len */
#include <trace/events/preemptirq.h> /* trace_preempt_disable tracepoint definition */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-arabic");
MODULE_DESCRIPTION("Trace preempt_disable to observe local_lock() activity");

/* ------------------------------------------------------------------ */
/* الـ tracepoint probe callback                                        */
/* ------------------------------------------------------------------ */

/*
 * handle_preempt_disable - called every time preempt_disable() fires.
 *
 * @ignore:    unused data pointer we pass at registration time (NULL here)
 * @ip:        instruction pointer of the call site (who called preempt_disable)
 * @parent_ip: caller of the caller — helps identify local_lock() wrapper
 *
 * نطبع فقط لما يكون الـ preempt_count قبل الـ disable == 0،
 * يعني ده أول مستوى من الـ disable — على الأغلب جاي من local_lock() مش من nesting.
 */
static void handle_preempt_disable(void *ignore,
                                   unsigned long ip,
                                   unsigned long parent_ip)
{
    /* Skip nested preempt_disable calls — only report first level */
    if (preempt_count() != 0)
        return;

    pr_info("local_lock_tracer: preempt_disable | cpu=%d pid=%d comm=%-16s ip=%pS parent=%pS\n",
            smp_processor_id(),
            current->pid,
            current->comm,
            (void *)ip,
            (void *)parent_ip);
}

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */

static int __init local_lock_tracer_init(void)
{
    int ret;

    /*
     * نسجّل الـ probe على tracepoint الجاهز preempt_disable.
     * الـ NULL الأول = data pointer مش محتاجينه.
     * الـ 1 الأخير = check أن الـ tracepoint موجود فعلاً.
     */
    ret = register_trace_preempt_disable(handle_preempt_disable, NULL);
    if (ret) {
        pr_err("local_lock_tracer: failed to register probe, err=%d\n", ret);
        return ret;
    }

    pr_info("local_lock_tracer: loaded — watching preempt_disable (local_lock proxy)\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */

static void __exit local_lock_tracer_exit(void)
{
    /*
     * لازم نشيل الـ probe قبل ما الـ module يتفرغ من الذاكرة،
     * عشان لو الـ tracepoint اتفعّل بعد الـ unload هيكون في use-after-free.
     */
    unregister_trace_preempt_disable(handle_preempt_disable, NULL);

    /* tracepoint_synchronize_unregister ضروري تنتظر أي callback شغّال دلوقتي */
    tracepoint_synchronize_unregister();

    pr_info("local_lock_tracer: unloaded\n");
}

module_init(local_lock_tracer_init);
module_exit(local_lock_tracer_exit);
```

---

### Makefile

```makefile
obj-m := local_lock_tracer.o
KDIR  := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | `module_init`, `module_exit`, `MODULE_LICENSE` |
| `<linux/tracepoint.h>` | `register_trace_*` / `unregister_trace_*` infrastructure |
| `<linux/preempt.h>` | `preempt_count()` — نتحقق إننا في أول مستوى disable |
| `<linux/sched.h>` | `current->pid`, `current->comm` — معلومات الـ task |
| `<trace/events/preemptirq.h>` | الـ declaration الخاص بـ `register_trace_preempt_disable` |

**الـ** header الأخير مهم — من غيره المترجم مش هيعرف signature الـ tracepoint الصح.

---

#### الـ callback `handle_preempt_disable`

**الـ** `ip` بيعطينا مين استدعى `preempt_disable()` بالضبط — لو الـ kernel مبني بـ `CONFIG_DEBUG_PREEMPT` هيبان اسم الـ function بالـ `%pS`. الـ `parent_ip` بيعطينا من استدعى ده، فممكن نشوف `__local_lock` في الـ stack.

الـ guard على `preempt_count() != 0` بيمنع الـ spam من الـ nested disables — اللي كتير منها مش جاي من `local_lock()`.

---

#### `module_init` — التسجيل

**الـ** `register_trace_preempt_disable` ماكرو جاهز من الـ kernel بيربط الـ callback بالـ tracepoint. لو رجع قيمة سالبة معناه الـ tracepoint مش مفعّل في الـ kernel الحالي (مثلاً `CONFIG_TRACE_PREEMPT_TOGGLE` مش موجود).

---

#### `module_exit` — إلغاء التسجيل

**الـ** `unregister_trace_preempt_disable` بيشيل الـ hook، لكن ممكن يكون في callback شغّال على CPU تاني في نفس اللحظة. عشان كده `tracepoint_synchronize_unregister()` بتعمل `synchronize_rcu()` وتضمن إن مفيش callback شغّال لما الـ module يتشال من الذاكرة — من غيرها هيبقى kernel crash مضمون.

---

### مثال على الـ output

```
[  142.003412] local_lock_tracer: loaded — watching preempt_disable (local_lock proxy)
[  142.014501] local_lock_tracer: preempt_disable | cpu=2 pid=1847 comm=kworker/2:1      ip=__local_lock+0x18/0x40 parent=kmem_cache_alloc+0x9c/0x200
[  142.014602] local_lock_tracer: preempt_disable | cpu=0 pid=312  comm=ksoftirqd/0     ip=__local_lock+0x18/0x40 parent=nf_conntrack_get+0x24/0x60
```

**الـ** `ip=__local_lock` بيأكد إن الـ preempt_disable جاي من `local_lock()` — وده بالضبط اللي أحنا بنراقبه.
