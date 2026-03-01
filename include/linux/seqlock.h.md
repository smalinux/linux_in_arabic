## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

الـ `seqlock.h` جزء من **LOCKING PRIMITIVES** subsystem في Linux kernel — المسؤول عن كل آليات الـ synchronization بين الـ threads والـ CPUs. الـ maintainers هم Peter Zijlstra وIngo Molnar وWill Deacon، والكود محفوظ في `kernel/locking/` وكل headers الـ locking في `include/linux/`.

---

### المشكلة اللي بيحلها الملف ده

تخيل عندك **ساعة رقمية** بتعرض الوقت — ساعة، دقيقة، ثانية. عايز تقرأ الثلاثة أرقام في نفس اللحظة بشكل consistent. المشكلة إن ممكن يكون الثانية بالظبط اللي إنت بتقرا فيه الدقيقة بتتغير، فتلاقي الوقت "12:59:00" وإنت قريت "12:59:60" — بيانات غلط.

الحل التقليدي؟ تعمل **lock** — يعني تمنع أي حد تاني يقرأ أو يكتب وإنت شغال. لكن دي مشكلة تانية: لو عندك **ألف CPU** بيقروا الوقت كل millisecond (زي ما بيحصل في الـ kernel لما بيحتاج `gettimeofday`)، هتخلق **bottleneck** ضخم — كل CPU لازم يستنى الـ lock.

الـ **seqlock** جاء بفكرة ذكية تانية خالص.

---

### الفكرة الجوهرية — القصة

بدل الـ lock، استخدم **عداد تسلسلي (sequence counter)**:

- الـ **writer** (اللي بيعدّل البيانات) بيزوّد العداد بـ 1 قبل ما يبدأ يكتب — فبيبقى **فردي (odd)** → علامة إن في كتابة جارية.
- بعد ما يخلص الكتابة، بيزوّد العداد تاني بـ 1 — فبيبقى **زوجي (even)** → علامة إن البيانات consistent.
- الـ **reader** بيقرأ العداد أول ما يبدأ، يقرأ البيانات، وبعدين يقرأ العداد تاني — لو العداد **اتغير** أو **فردي**: يُعيد المحاولة (retry).

```
Writer:    [seq=0] → seq++ → [seq=1] → كتابة → seq++ → [seq=2]
Reader:    قرأ seq=2 (زوجي ✓) → قرأ بيانات → قرأ seq=2 (نفسه ✓) → نجح!
Reader:    قرأ seq=1 (فردي ✗) → ينتظر حتى يبقى زوجي → يحاول تاني
Reader:    قرأ seq=2 → قرأ بيانات → قرأ seq=4 (اتغير ✗) → يعيد المحاولة
```

**النتيجة؟** الـ readers **مش بيعملوا lock خالص** — بيقروا بحرية ويعيدوا المحاولة لو فيه تعارض. ده بيدي performance خرافي لو الكتابة نادرة.

---

### ليه ده مهم؟ مثال حقيقي

الـ `gettimeofday()` في Linux بتتنادى **ملايين المرات في الثانية** من كل الـ processes. لو استخدمنا `mutex` عادي، كل call هتستنى الـ lock وهيبقى الـ system بطيء. الـ seqlock بيخلي **كل CPU يقرأ الوقت في نفس اللحظة** بدون أي انتظار في الغالب.

---

### الـ Types الموجودة في الملف

| النوع | الوصف |
|-------|-------|
| `seqcount_t` | العداد الخام — بدون أي حماية للـ writer، المسؤولية على المستخدم |
| `seqcount_spinlock_t` | عداد مرتبط بـ spinlock — lockdep بيتحقق إن الـ writer ماسك الـ lock |
| `seqcount_raw_spinlock_t` | نفسه لكن مع raw_spinlock |
| `seqcount_rwlock_t` | مرتبط بـ rwlock |
| `seqcount_mutex_t` | مرتبط بـ mutex (مناسب لـ PREEMPT_RT) |
| `seqcount_latch_t` | نسخة خاصة بالـ NMI — بتحتفظ بـ **نسختين** من البيانات |
| `seqlock_t` | الـ seqcount + spinlock مدمج في struct واحد — الأسهل استخداماً |

---

### الـ seqcount_latch_t — الحالة الخاصة جداً

تخيل إن الـ **NMI (Non-Maskable Interrupt)** قدر يقاطع الـ writer في النص! في الـ seqcount العادي ده هيعمل مشكلة لأن العداد هيفضل فردي إلى ما لا نهاية من ناحية الـ NMI handler.

الـ **latch** بيحل ده باحتفاظ بـ **نسختين من البيانات** (`data[0]` و`data[1]`):
- العداد الزوجي → اقرأ من `data[0]`
- العداد الفردي → اقرأ من `data[1]`

```
الكتابة:
  1. زوّد العداد → فردي → readers يروحوا data[1]
  2. عدّل data[0]
  3. زوّد العداد → زوجي → readers يروحوا data[0]
  4. عدّل data[1]
```

دايماً فيه نسخة واحدة stable. مستخدَم في الـ `vDSO` لقراءة الوقت من الـ userspace بدون syscall.

---

### ليه في أنواع كتير من seqcount_LOCKNAME_t؟

المشكلة في بعض الـ lock types إنها **بتسمح بالـ preemption** (زي `mutex`). لو الـ writer اتـ preempted وعنده العداد فردي، الـ readers هيفضلوا في **spin loop** إلى ما لا نهاية — **livelock**!

الحل: لو في **PREEMPT_RT** والـ lock قابل للـ preemption، الـ reader بيعمل `lock → unlock` على نفس الـ lock لو لاقى العداد فردي — ده بيخلي الـ writer يكمل شغله أولاً.

---

### المشهد العام بالـ ASCII

```
                    ┌─────────────────────────────────────────┐
                    │           include/linux/seqlock.h        │
                    │                                          │
                    │  ┌─────────────┐   ┌─────────────────┐  │
                    │  │ seqcount_t  │   │   seqlock_t     │  │
                    │  │ (raw/bare)  │   │ (seqcount +     │  │
                    │  └──────┬──────┘   │  spinlock)      │  │
                    │         │          └────────┬────────┘  │
                    │         ▼                   │            │
                    │  ┌──────────────────────────▼──────┐    │
                    │  │   seqcount_LOCKNAME_t variants  │    │
                    │  │  (spinlock/rwlock/mutex/raw)     │    │
                    │  └─────────────────────────────────┘    │
                    │                                          │
                    │  ┌─────────────────────────────────┐    │
                    │  │       seqcount_latch_t          │    │
                    │  │  (double-buffer for NMI safety) │    │
                    │  └─────────────────────────────────┘    │
                    └─────────────────────────────────────────┘
                              ▲              ▲
                    ┌─────────┘              └──────────┐
          include/linux/             include/linux/
          seqlock_types.h            lockdep.h / spinlock.h
          (struct definitions)       (validation & locking)
```

---

### الـ API بشكل مبسط

**Read side (بدون lock — الأسرع):**
```c
unsigned seq;
do {
    seq = read_seqbegin(&my_seqlock);       // احفظ العداد
    // ... اقرأ البيانات المحمية ...
} while (read_seqretry(&my_seqlock, seq));  // لو اتغير العداد: كرر
```

**Write side (مع lock):**
```c
write_seqlock(&my_seqlock);        // lock + زوّد العداد (فردي)
// ... عدّل البيانات ...
write_sequnlock(&my_seqlock);      // زوّد العداد (زوجي) + unlock
```

---

### الـ Constraints المهمة

- **لا تستخدمه مع pointers** — الـ writer ممكن يـ invalidate الـ pointer وإنت بتتبعه.
- الـ **write side يجب أن يكون non-preemptible** (إلا في PREEMPT_RT مع seqcount_LOCKNAME_t).
- لو الـ readers من hardirq/softirq: الـ writer لازم يـ disable interrupts/BH.
- **لا يوجد writer starvation** — بسبب الـ spinlock الداخلي.
- **لا يوجد reader starvation** أيضاً — لو الكتابة كتيرة جداً، استخدم `read_seqbegin_or_lock()`.

---

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/seqlock.h` | **الملف الرئيسي** — كل الـ API والـ inline functions |
| `include/linux/seqlock_types.h` | تعريفات الـ structs (`seqcount_t`, `seqlock_t`, وكل الـ variants) |
| `Documentation/locking/seqlock.rst` | التوثيق الرسمي مع أمثلة استخدام |
| `include/linux/lockdep.h` | التحقق من صحة الاستخدام في debug builds |
| `include/linux/spinlock.h` | الـ spinlock المدمج في `seqlock_t` |
| `include/linux/kcsan-checks.h` | دعم الـ Kernel Concurrency Sanitizer |
| `kernel/time/timekeeping.c` | أبرز مستخدم حقيقي للـ seqlock (قراءة الوقت) |
| `arch/x86/entry/vdso/vclock_gettime.c` | استخدام `seqcount_latch_t` لقراءة الوقت من userspace |
| `kernel/locking/` | بقية الـ locking primitives في نفس الـ subsystem |
## Phase 2: شرح الـ Seqlock / Seqcount Framework

---

### المشكلة اللي بيحلها الـ Subsystem ده

في الـ kernel، عندك بيانات بتتقرأ كتير جداً وبتتكتب نادراً — زي الـ `jiffies`، أو الـ `timespec` للوقت، أو الـ `vDSO` timing data. لو استخدمت **spinlock** عادي:

- كل reader لازم يـ**acquire** الـ lock — يعني readers بيـblock بعض رغم إنهم read-only.
- overhead عالي جداً على الـ fast paths.

لو استخدمت **RWlock** (reader-writer lock):

- Readers مش بيـblock بعض تمام.
- لكن writer بيستنى **كل** الـ readers يخلصوا — ده ممكن يسبب **writer starvation** لو في readers كتير.
- لا يصلحش لـ NMI context لأن الـ lock نفسه محتاج synchronization.

الـ **seqlock** جه يحل المعادلة دي: readers **بدون أي lock خالص**، وكمان **writers مش بيتجوعوا**.

---

### الحل — الفكرة الجوهرية

الفكرة بسيطة وعبقرية: **counter زوجي/فردي يلعب دور شاهد على التعديلات**.

1. Writer يزود الـ counter بـ 1 قبل ما يكتب → counter بقى **فردي** (write in progress).
2. Writer يكتب البيانات.
3. Writer يزود الـ counter بـ 1 تاني → counter بقى **زوجي** (write done).

Reader بيعمل الآتي:
1. يقرأ الـ counter (لازم يكون زوجي، يعني مفيش كتابة جارية).
2. يقرأ البيانات.
3. يتحقق إن الـ counter لسه نفس القيمة — لو اتغير، يعيد المحاولة.

الـ reader **مش بيأخد أي lock**. الـ writer هو اللي بيـserialize نفسه مع الـ writers التانيين (عن طريق spinlock منفصل في `seqlock_t`، أو lock خارجي في `seqcount_t`).

---

### التشبيه من الواقع — وأعمق من السطح

تخيل **لوحة إعلانات في مطار** بيكتب عليها موظف جدول الرحلات، والمسافرون بيقروا منها.

| عنصر الـ Analogy | المقابل في الـ Kernel |
|---|---|
| **الرقم اللي في ركن اللوحة** (رقم "إصدار" اللوحة) | الـ `sequence` counter |
| الموظف **يمسح الرقم أثناء التعديل** (يحطه فردي) | `do_raw_write_seqcount_begin()` → `s->sequence++` |
| الموظف **يكتب الرقم الجديد لما يخلص** (يرجع زوجي) | `do_raw_write_seqcount_end()` → `s->sequence++` |
| المسافر **يحفظ الرقم قبل ما يقرأ** | `read_seqcount_begin()` → بيرجع `start` |
| المسافر **يتأكد إن الرقم ما اتغيرش بعد ما قرأ** | `read_seqcount_retry()` |
| لو الرقم **اتغير → يعيد القراءة من الأول** | الـ `do { } while (read_seqcount_retry(...))` loop |
| **أكتر من موظف ممنوع يكتبوا في نفس الوقت** → في ناظر (الـ spinlock) | الـ `lock` في `seqlock_t` أو الـ external lock في `seqcount_LOCKNAME_t` |
| لو الرقم **فردي لما المسافر جه** → يستنى | الـ `while (unlikely(seq & 1)) cpu_relax()` |
| المسافرون **مش محتاجين يستنوا بعض** | Readers are truly lockless |
| لو الموظف **اتعطل وسط الكتابة** (preempted on RT) | الـ reader يـacquire ويـrelease نفس الـ lock عشان يحرر الـ writer |

---

### المعمارية الكبيرة — الـ Seqlock في الـ Kernel

```
┌─────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Consumers (من بيستخدم الـ seqlock)         │   │
│  │                                                         │   │
│  │  timekeeping subsystem    vDSO (userspace time)         │   │
│  │  (kernel/time/timekeeping.c)  net/core/ (routing)       │   │
│  │  sched clock              mm/ (mmap_seq)                │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │ uses                                │
│  ┌────────────────────────▼────────────────────────────────┐   │
│  │              seqlock / seqcount Framework               │   │
│  │                                                         │   │
│  │   seqcount_t          seqcount_LOCKNAME_t   seqlock_t   │   │
│  │   (raw counter)       (counter + lock ptr)  (counter    │   │
│  │                                              + spinlock) │   │
│  │   seqcount_latch_t    (double-buffer latch)             │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │ depends on                          │
│  ┌────────────────────────▼────────────────────────────────┐   │
│  │              Core Kernel Infrastructure                 │   │
│  │                                                         │   │
│  │   spinlock_t   rwlock_t   mutex_t   raw_spinlock_t      │   │
│  │   smp_wmb()    smp_rmb()   READ_ONCE()   WRITE_ONCE()   │   │
│  │   lockdep      kcsan       preempt_disable/enable        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions — الطبقات التلاتة

الـ framework مش واحد — فيه **تلات مستويات**، كل واحد أكثر convenience من اللي قبله:

```
seqcount_t                  ← الـ raw counter بدون أي حماية built-in
      │
      ├─── seqcount_LOCKNAME_t  ← counter + pointer للـ associated lock
      │         (lockdep يتحقق إن الـ writer ماسك الـ lock فعلاً)
      │
      └─── seqlock_t             ← seqcount_spinlock_t + spinlock embedded
                                    (الـ all-inclusive، أسهل استخدام)
```

وفي أيضاً:
```
seqcount_latch_t            ← variant خاص للـ NMI-safe double-buffer pattern
```

---

### الـ Structs التفصيلية وعلاقتها ببعض

#### 1. `seqcount_t` — اللبنة الأساسية

```c
/* من include/linux/seqlock_types.h */
typedef struct seqcount {
    unsigned sequence;          /* الـ counter نفسه — زوجي=stable, فردي=writing */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map; /* فقط لو lockdep enabled */
#endif
} seqcount_t;
```

```
seqcount_t
┌──────────────┐
│  sequence    │  ← unsigned int، LSB = write-in-progress flag
│  (dep_map)   │  ← optional، فقط لـ DEBUG_LOCK_ALLOC
└──────────────┘
```

قيمة الـ `sequence`:
- **زوجي (even)**: البيانات stable، ينفع تقرأ.
- **فردي (odd)**: كتابة جارية، الـ reader يستنى أو يـretry.

---

#### 2. `seqcount_LOCKNAME_t` — Counter مع Associated Lock

الـ macro `SEQCOUNT_LOCKNAME` في `seqlock_types.h` بيولّد 4 types:

```c
/* النتيجة بعد macro expansion */
typedef struct seqcount_spinlock {
    seqcount_t   seqcount;    /* الـ raw counter */
    spinlock_t  *lock;        /* pointer للـ associated lock */
                              /* (موجود فقط لو LOCKDEP أو PREEMPT_RT) */
} seqcount_spinlock_t;

/* نفس الشكل لـ: */
/* seqcount_raw_spinlock_t  */
/* seqcount_rwlock_t         */
/* seqcount_mutex_t          */
```

```
seqcount_spinlock_t
┌──────────────────────────┐
│  seqcount_t seqcount     │  ← الـ actual counter
│  ┌─────────┐             │
│  │sequence │             │
│  └─────────┘             │
│  spinlock_t *lock  ──────┼──→ [spinlock في مكان آخر في الـ memory]
└──────────────────────────┘
         ↑
    هنا الـ lock pointer بيُستخدم فقط لـ:
    1. lockdep: يتحقق إن الـ writer ماسك الـ lock
    2. PREEMPT_RT: الـ reader يـacquire/release لو writer preempted
```

---

#### 3. `seqlock_t` — الـ All-in-One

```c
/* من include/linux/seqlock_types.h */
struct seqlock {
    seqcount_spinlock_t seqcount;  /* الـ counter + lock pointer */
    spinlock_t          lock;      /* الـ actual spinlock embedded */
};
typedef struct seqlock seqlock_t;
```

```
seqlock_t
┌──────────────────────────────────┐
│  seqcount_spinlock_t seqcount    │
│  ┌──────────────────────────┐   │
│  │  seqcount_t seqcount     │   │
│  │  ┌─────────┐             │   │
│  │  │sequence │             │   │
│  │  └─────────┘             │   │
│  │  spinlock_t *lock ───────┼───┼──┐
│  └──────────────────────────┘   │  │
│                                  │  │
│  spinlock_t lock  ←──────────────┼──┘
│  (الـ lock بيـpoint على نفسه)    │
└──────────────────────────────────┘
```

الـ `seqcount.lock` بيـpoint على `&sl->lock` اللي موجود في نفس الـ struct — self-referential.

---

#### 4. `seqcount_latch_t` — الـ Double-Buffer Variant

```c
typedef struct {
    seqcount_t seqcount;
} seqcount_latch_t;
```

الـ latch بيشتغل بطريقة مختلفة — مش retry، لكن **double-buffer**:

```
struct latch_struct {
    seqcount_latch_t  seq;
    struct data       data[2];   ← نسختين من البيانات
};

sequence:  0  →  1  →  2  →  3  →  4
           even  odd  even  odd  even
           ↕     ↕    ↕     ↕
read[0]   write[0]  read[0]  write[0]
           write[1]         write[1]
                   read[1]
```

الـ LSB من الـ sequence بيحدد **أي نسخة يقرأ منها الـ reader** (`seq & 1`).

---

### الـ Generic Dispatch — الـ `_Generic` Magic

الـ framework بيستخدم C11 `_Generic` عشان يـdispatch الـ operations على حسب نوع الـ seqcount:

```c
#define __seqprop(s, prop) _Generic(*(s),               \
    seqcount_t:          __seqprop_##prop,               \
    seqcount_spinlock_t: __seqprop_spinlock_##prop,      \
    seqcount_rwlock_t:   __seqprop_rwlock_##prop,        \
    seqcount_mutex_t:    __seqprop_mutex_##prop)

/* مثلاً: */
#define seqprop_sequence(s) __seqprop(s, sequence)(s)
```

يعني نفس الـ macro `read_seqcount_begin(s)` بيشتغل على كل أنواع الـ seqcount — الـ compiler بيختار الـ implementation الصح في **compile time** مش runtime.

```
read_seqcount_begin(s)
         │
         ▼ _Generic dispatch (compile time)
    ┌────┴──────────────────────────────────┐
    │  seqcount_t?         → __seqprop_sequence        │
    │  seqcount_spinlock_t?→ __seqprop_spinlock_sequence│
    │  seqcount_mutex_t?   → __seqprop_mutex_sequence  │
    └───────────────────────────────────────┘
         │
         ▼ لو PREEMPT_RT && preemptible lock && seq is odd:
    acquire + release the associated lock
    (عشان الـ writer اللي اتـpreempt يكمل)
         │
         ▼
    إرجاع الـ sequence value
```

---

### الـ Memory Ordering — الأهم والأصعب

> **قبل ما نكمل**: الـ **memory barriers** موضوع مستقل في الـ kernel. باختصار: المعالجات والـ compilers ممكن يعيدوا ترتيب القراءات والكتابات — الـ barriers بتمنع ده.

#### Write Side:

```c
static inline void do_raw_write_seqcount_begin(seqcount_t *s)
{
    kcsan_nestable_atomic_begin();
    s->sequence++;    /* ① increment to ODD — signal write start */
    smp_wmb();        /* ② write memory barrier: كل الكتابات بعد كده */
                      /*    مش هتتعمل قبل الـ sequence++ */
}

static inline void do_raw_write_seqcount_end(seqcount_t *s)
{
    smp_wmb();        /* ③ write memory barrier: الكتابات اللي جوه الـ section */
                      /*    لازم تكون visible قبل الـ sequence++ */
    s->sequence++;    /* ④ increment to EVEN — signal write complete */
    kcsan_nestable_atomic_end();
}
```

```
Writer timeline:
─────────────────────────────────────────────────────────────────
seq=0(even) → seq=1(odd) ─[smp_wmb]─ WRITE DATA ─[smp_wmb]─ seq=2(even)
               ↑                                              ↑
           write starts                                  write ends
```

#### Read Side:

```c
static inline int do_read_seqcount_retry(const seqcount_t *s, unsigned start)
{
    smp_rmb();    /* read memory barrier: التأكد إن البيانات اتقرأت كلها */
                  /* قبل ما نقارن الـ sequence */
    return unlikely(READ_ONCE(s->sequence) != start);
}
```

الـ `smp_rmb()` بيضمن إن أي قراءة للبيانات في الـ critical section اتنفذت **قبل** ما نقرأ الـ sequence للمقارنة. بدونه، الـ CPU ممكن يعيد ترتيب القراءات ويشوف sequence جديد لكن بيانات قديمة.

---

### Read Flow — خطوة بخطوة

```c
/* مثال حقيقي: قراءة timekeeping data */
struct timespec64 ts;
unsigned seq;

do {
    seq = read_seqcount_begin(&tk->seq);  /* (1) */
    /* يستنى لو seq فردي (write in progress) */
    /* يحفظ الـ seq value */

    ts = tk->xtime_sec;                   /* (2) قراءة البيانات */

} while (read_seqcount_retry(&tk->seq, seq)); /* (3) */
/* smp_rmb() + تحقق إن seq ما اتغيرش */
/* لو اتغير → loop مرة تانية */
```

```
Timeline مع writer concurrent:

Reader:  [seq=2] ──────────────────── [read data] ──── [seq≠2? YES: retry]
                                              ↑
Writer:          [seq→3] ─ [write] ─ [seq→4]
                    ↑
              writer started during read window
```

---

### Write Flow — خطوة بخطوة

#### باستخدام `seqlock_t` (الأسهل):

```c
write_seqlock(&sl);
    /* spin_lock(&sl->lock)               ← يـserialize الـ writers */
    /* do_write_seqcount_begin(...)        ← seq++ (odd) + smp_wmb() */

    /* ← تعديل البيانات هنا */

write_sequnlock(&sl);
    /* do_write_seqcount_end(...)          ← smp_wmb() + seq++ (even) */
    /* spin_unlock(&sl->lock)              ← يفك الـ lock */
```

#### باستخدام `seqcount_t` (manual):

```c
/* الـ caller مسؤول إنه يكون ماسك الـ external lock */
write_seqcount_begin(&my_seqcount);
    /* seq++ (odd) + smp_wmb() */
    /* + lockdep assert إن الـ lock ماسكه */

    /* ← تعديل البيانات */

write_seqcount_end(&my_seqcount);
    /* smp_wmb() + seq++ (even) */
```

---

### الـ `raw_write_seqcount_barrier` — Ordering لا Consistency

```c
static inline void do_raw_write_seqcount_barrier(seqcount_t *s)
{
    kcsan_nestable_atomic_begin();
    s->sequence++;   /* → odd */
    smp_wmb();
    s->sequence++;   /* → even */
    kcsan_nestable_atomic_end();
}
```

ده مش begin+end عادي. الـ **barrier** ده بيضمن:
- البيانات اللي اتكتبت **قبل** الـ barrier مش هتتشاف مع البيانات اللي اتكتبت **بعده** في نفس الـ read section.
- أرخص من الـ begin/end العادي لأنه بيضم الـ `smp_wmb` الـ ending و beginning في واحدة.

Use case: لما عندك بيانات بتتكتب باستمرار وعايز تضمن إن الـ reader إما يشوف الـ old state أو الـ new state، مش intermediate.

---

### الـ `write_seqcount_invalidate` — Force Retry

```c
static inline void do_write_seqcount_invalidate(seqcount_t *s)
{
    smp_wmb();
    kcsan_nestable_atomic_begin();
    s->sequence += 2;    /* ← يزود اثنين! يبقى even لكن اتغير */
    kcsan_nestable_atomic_end();
}
```

بيزود الـ sequence باثنين (يفضل even → ما بيبدأش write section) لكن يغير القيمة. أي reader كان شايل الـ old value هيـretry. مفيد لو اتعملت تغيير بدون الدخول في write section عادي.

---

### الـ PREEMPT_RT Problem وحله

في الـ **Real-Time kernel** (`CONFIG_PREEMPT_RT`)، الـ spinlocks بتبقى **sleeping locks**. يعني لو writer اتـpreempt وسط الـ write section (sequence فردي)، الـ reader مش هيعرف يعمل `cpu_relax()` للأبد.

الحل في الـ `__seqprop_LOCKNAME_sequence()`:

```c
/* لو PREEMPT_RT && lock is preemptible && sequence is odd: */
if (preemptible && unlikely(seq & 1)) {
    lockbase##_lock(s->lock);    /* acquire الـ associated lock */
    lockbase##_unlock(s->lock);  /* release فوراً */
    /* ده هيبلوك لحد ما الـ writer يخلص ويفك الـ lock */
    seq = smp_load_acquire(&s->seqcount.sequence);  /* إقرأ تاني */
}
```

يعني الـ reader بدل ما يـspin، بيـblock بشكل صحيح على الـ lock ويستنى الـ writer يخلص.

---

### الـ Latch Pattern — للـ NMI و Interrupt Contexts

الـ `seqcount_latch_t` بيحل مشكلة تانية: لو الـ writer ممكن يتقاطع مع الـ reader من **NMI** (غير قابل للـmask)، فحتى الـ retry loop مش آمنة لأن الـ NMI ممكن يحصل وسط الـ writer نفسه.

الحل: **نسختين من البيانات**، الـ writer بيعدّل واحدة في الوقت، والـ reader دايماً يلاقي نسخة stable.

```c
struct latch_struct {
    seqcount_latch_t  seq;
    struct data       data[2];
};

/* Writer flow: */
void latch_modify(struct latch_struct *latch, ...)
{
    write_seqcount_latch_begin(&latch->seq); /* seq: 0→1 (odd) */
    modify(latch->data[0], ...);             /* عدّل النسخة 0 */
    write_seqcount_latch(&latch->seq);       /* seq: 1→2 (even) */
    modify(latch->data[1], ...);             /* عدّل النسخة 1 */
    write_seqcount_latch_end(&latch->seq);   /* seq: stays 2 */
}

/* Reader flow: */
struct entry *latch_query(struct latch_struct *latch, ...)
{
    unsigned seq, idx;
    do {
        seq = read_seqcount_latch(&latch->seq);
        idx = seq & 0x01;                        /* 0 أو 1 */
        entry = data_query(latch->data[idx], ...);
    } while (read_seqcount_latch_retry(&latch->seq, seq));
    return entry;
}
```

```
Writer states:
seq=0(even): data[0]=stable, data[1]=stable    → reader reads data[0]
seq=1(odd):  data[0]=WRITING, data[1]=stable   → reader reads data[1]
seq=2(even): data[0]=new, data[1]=stable       → reader reads data[0]
seq=3(odd):  data[0]=new, data[1]=WRITING      → reader reads data[0]
seq=4(even): data[0]=new, data[1]=new          → reader reads data[0]
```

**المفتاح**: حتى لو الـ NMI جه وسط write الـ `data[0]`، الـ reader هيقرأ `data[1]` اللي stable.

---

### الـ KCSAN Integration

> **الـ KCSAN** (Kernel Concurrency Sanitizer) هو أداة لكشف الـ data races في الـ kernel.

```c
#define KCSAN_SEQLOCK_REGION_MAX 1000

/* في بداية الـ read section: */
kcsan_atomic_next(KCSAN_SEQLOCK_REGION_MAX);
/* بيخبر KCSAN إن الـ 1000 access الجاية هي intentionally racy */
/* (بتتقرأ بدون lock، وده متوقع) */

/* في نهاية الـ read section (retry check): */
kcsan_atomic_next(0);
/* بيرجع KCSAN لوضعه الطبيعي */
```

---

### ملكية الـ Framework — ايه اللي يمتلكه وايه اللي يـdelegate

| المسؤولية | الـ Framework يتولاها | الـ Driver/Caller مسؤول عنها |
|---|---|---|
| الـ sequence counter logic | ✓ | — |
| الـ memory barriers (wmb/rmb) | ✓ | — |
| الـ PREEMPT_RT reader adaptation | ✓ | — |
| الـ KCSAN annotations | ✓ | — |
| الـ lockdep integration | ✓ | — |
| Writer serialization (في `seqlock_t`) | ✓ (spinlock embedded) | — |
| Writer serialization (في `seqcount_t`) | — | ✓ يوفر الـ external lock |
| تحديد الـ critical section boundaries | — | ✓ يستدعي begin/end |
| عدم استخدامه مع بيانات تحتوي على pointers | — | ✓ مسؤولية الـ caller |
| disable interrupts لو readers من hardirq | — | ✓ استخدم `_irqsave` variants |

---

### أمثلة حقيقية في الـ Kernel

#### `timekeeping` — أكثر مثال شهير:

```c
/* kernel/time/timekeeping.c */
static struct {
    seqcount_raw_spinlock_t seq;
    struct timekeeper       timekeeper;
} tk_core;

/* Writer (interrupt handler): */
raw_spin_lock(&timekeeper_lock);
write_seqcount_begin(&tk_core.seq);
    /* update tk_core.timekeeper */
write_seqcount_end(&tk_core.seq);
raw_spin_unlock(&timekeeper_lock);

/* Reader (any context): */
unsigned seq;
struct timekeeper *tk;
do {
    seq = read_seqcount_begin(&tk_core.seq);
    memcpy(&ts, &tk_core.timekeeper.xtime_sec, sizeof(ts));
} while (read_seqcount_retry(&tk_core.seq, seq));
```

#### `vDSO` — قراءة الوقت من userspace بدون syscall:

الـ kernel بيكتب timing data في صفحة مشتركة مع الـ userspace باستخدام seqcount، والـ userspace code بيقرأ بنفس الـ retry loop — بدون أي syscall.

---

### جدول مقارنة الأنواع

| النوع | Writer Lock | Readers | Preempt Disable | متى تستخدمه |
|---|---|---|---|---|
| `seqcount_t` | خارجي (manual) | lockless + retry | manual | عندك lock خاص بيك |
| `seqcount_spinlock_t` | spinlock خارجي | lockless + retry | تلقائي (non-RT) | lockdep + existing spinlock |
| `seqcount_mutex_t` | mutex خارجي | lockless + retry | تلقائي | writes نادرة جداً |
| `seqlock_t` | spinlock embedded | lockless + retry | تلقائي | الأسهل، general purpose |
| `seqcount_latch_t` | خارجي | lockless + always ok | لا | NMI/interrupt safe reads |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### الـ Config Options المهمة

| Option | التأثير |
|---|---|
| `CONFIG_DEBUG_LOCK_ALLOC` | يضيف `dep_map` لـ `seqcount_t` ويفعّل lockdep runtime checks |
| `CONFIG_LOCKDEP` | يفعّل كامل نظام lockdep — بدونه `lockdep_map` هيكل فاضي |
| `CONFIG_PREEMPT_RT` | يغير سلوك الـ `seqcount_LOCKNAME_t`: الـ writers مش بيعطلوا preemption — بدلها الـ readers بيعملوا lock/unlock لو لاقوا writer في المنتصف |
| `CONFIG_LOCK_STAT` | يضيف إحصائيات contention جوه `lock_class` |
| `CONFIG_PROVE_RAW_LOCK_NESTING` | يفصل بين `LD_WAIT_CONFIG` و`LD_WAIT_SPIN` في lockdep |
| `CONFIG_CC_IS_GCC` / `CONFIG_KASAN` | يضعّف optimization guard الخاص بـ `__scoped_seqlock_bug()` |

#### الـ `enum ss_state` — حالات الـ Scoped Seqlock

| القيمة | المعنى |
|---|---|
| `ss_done = 0` | انتهت الـ loop — مفيش retry |
| `ss_lock` | الـ reader شغّال في وضع locking بـ spinlock |
| `ss_lock_irqsave` | وضع locking مع تعطيل interrupts |
| `ss_lockless` | الـ reader شغّال lockless (optimistic) |

#### الـ `enum lockdep_wait_type` — أنواع انتظار الـ Lock

| القيمة | المعنى |
|---|---|
| `LD_WAIT_FREE` | wait-free — زي RCU |
| `LD_WAIT_SPIN` | يدور في حلقة — raw_spinlock_t |
| `LD_WAIT_CONFIG` | preemptible على PREEMPT_RT — spinlock_t |
| `LD_WAIT_SLEEP` | ممكن ينام — mutex_t |

#### الـ `enum bounce_type` — إحصائيات الـ Lock Bouncing

| القيمة | المعنى |
|---|---|
| `bounce_acquired_write` | اتخد الـ lock كـ writer |
| `bounce_acquired_read` | اتخد الـ lock كـ reader |
| `bounce_contended_write` | كان في تنافس كـ writer |
| `bounce_contended_read` | كان في تنافس كـ reader |

#### الـ Macros الرئيسية — Cheatsheet

| الـ Macro | الغرض |
|---|---|
| `KCSAN_SEQLOCK_REGION_MAX 1000` | عدد الـ memory accesses اللي KCSAN يعتبرها atomic جوه read section |
| `__SEQ_LOCK(expr)` | يضمّن `expr` بس لو `LOCKDEP` أو `PREEMPT_RT` مفعّلين |
| `__SEQ_RT` | `IS_ENABLED(CONFIG_PREEMPT_RT)` — يُستخدم كـ preemptible flag |
| `SEQCNT_ZERO(name)` | static initializer لـ `seqcount_t` |
| `SEQCNT_LATCH_ZERO(seq_name)` | static initializer لـ `seqcount_latch_t` |
| `__SEQLOCK_UNLOCKED(lockname)` | static initializer لـ `seqlock_t` |
| `DEFINE_SEQLOCK(sl)` | يعرّف ويبدأ `seqlock_t` statically |

---

### 1. الـ Structs المهمة

#### `seqcount_t` — العداد الأساسي

**الغرض:** ده الـ building block الأساسي — عداد رقم بسيط بيعكس حالة الـ write. الـ LSB (bit 0) = 1 يعني في writer شغّال دلوقتي.

```c
typedef struct seqcount {
    unsigned sequence;           /* العداد — زوجي = مفيش write، فردي = write جاري */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;  /* lockdep metadata — موجود بس لو DEBUG_LOCK_ALLOC */
#endif
} seqcount_t;
```

| الـ Field | النوع | الغرض |
|---|---|---|
| `sequence` | `unsigned` | العداد — زوجي = data stable، فردي = write جاري |
| `dep_map` | `struct lockdep_map` | lockdep tracking — debug only |

**الـ Invariant الحرج:** الـ writer يزوده بـ 1 (بيخليه فردي) → يكتب → يزوده تاني بـ 1 (بيخليه زوجي). الـ reader يلاقي فردي → ينتظر أو يعيد المحاولة.

---

#### `seqcount_LOCKNAME_t` — العداد مع Lock مرتبط

**الغرض:** `seqcount_t` مع pointer للـ lock اللي بيعمل serialize للـ writers. بيتولد بـ macro `SEQCOUNT_LOCKNAME` لكل نوع lock.

```c
/* المولود من الـ macro — مثلاً seqcount_spinlock_t */
typedef struct seqcount_spinlock {
    seqcount_t    seqcount;      /* العداد الفعلي */
    __SEQ_LOCK(spinlock_t *lock); /* pointer للـ lock — موجود لو LOCKDEP أو PREEMPT_RT */
} seqcount_spinlock_t;
```

الأنواع الموجودة:

| النوع | الـ Lock المرتبط | Preemptible على PREEMPT_RT |
|---|---|---|
| `seqcount_raw_spinlock_t` | `raw_spinlock_t *` | لا |
| `seqcount_spinlock_t` | `spinlock_t *` | نعم (لو `__SEQ_RT`) |
| `seqcount_rwlock_t` | `rwlock_t *` | نعم (لو `__SEQ_RT`) |
| `seqcount_mutex_t` | `struct mutex *` | نعم دايماً |

---

#### `seqcount_latch_t` — العداد ذو النسختين

**الغرض:** نوع خاص بيستخدم الـ LSB كـ index لاختيار بين نسختين من البيانات. بيسمح للـ NMI handlers بالقراءة الآمنة حتى لو الـ writer شغّال.

```c
typedef struct {
    seqcount_t seqcount;  /* نفس المبدأ — الـ LSB يحدد أي نسخة */
} seqcount_latch_t;
```

**المبدأ:**
- الـ writer يكتب في `data[seq & 1]` → يزوّد العداد → يكتب في `data[seq & 1]` تاني
- الـ reader يقرأ الـ index من الـ LSB → يقرأ من `data[idx]` → يتحقق إن العداد ما تغيرش

---

#### `seqlock_t` — الـ Sequential Lock الكامل

**الغرض:** يجمع `seqcount_spinlock_t` مع `spinlock_t` في هيكل واحد. الـ spinlock بيضمن serialization للـ writers والـ seqcount بيسمح للـ readers يشتغلوا بدون lock.

```c
struct seqlock {
    seqcount_spinlock_t seqcount;  /* العداد المرتبط بـ spinlock */
    spinlock_t lock;               /* الـ lock اللي بيعمل serialize للـ writers */
};
typedef struct seqlock seqlock_t;
```

| الـ Field | الغرض |
|---|---|
| `seqcount` | العداد — بيتابع حالة الـ write |
| `lock` | spinlock — بيمنع التعارض بين الـ writers |

---

#### `ss_tmp` — حالة الـ Scoped Seqlock

**الغرض:** struct مؤقت بيتخزن على الـ stack جوه `scoped_seqlock_read` macro. بيتابع حالة الـ for-loop اللي بتنفذ الـ optimistic read ثم تتحول لـ locking لو فشلت.

```c
struct ss_tmp {
    enum ss_state  state;         /* الحالة الحالية للـ loop */
    unsigned long  data;          /* العداد أو الـ irq flags */
    spinlock_t    *lock;          /* الـ lock لو كنا في ss_lock mode */
    spinlock_t    *lock_irqsave;  /* الـ lock لو كنا في ss_lock_irqsave mode */
};
```

---

#### `lockdep_map` — خريطة الـ Lock في Lockdep

**الغرض:** embedded جوه كل lock instance، بيربطه بـ `lock_class_key` للـ runtime deadlock detection.

```c
struct lockdep_map {
    struct lock_class_key  *key;                                    /* مفتاح التعريف */
    struct lock_class      *class_cache[NR_LOCKDEP_CACHING_CLASSES]; /* cache لأسرع وصول */
    const char             *name;                                   /* اسم الـ lock للـ debug */
    u8                      wait_type_outer;                        /* السياق اللي ممكن يتاخد فيه */
    u8                      wait_type_inner;                        /* السياق اللي بيقدمه */
    u8                      lock_type;                              /* نوع الـ lock */
};
```

---

#### `lock_class_key` — مفتاح تصنيف الـ Lock

**الغرض:** بيعرّف "نوع" الـ lock بشكل فريد لـ lockdep. عادةً static — بتاع كل lock instance من نفس النوع بيشترك في نفس الـ key.

```c
struct lock_class_key {
    union {
        struct hlist_node            hash_entry;                    /* للـ dynamic keys */
        struct lockdep_subclass_key  subkeys[MAX_LOCKDEP_SUBCLASSES]; /* 8 subclasses */
    };
};
```

---

### 2. علاقات الـ Structs

```
seqlock_t
├── seqcount_spinlock_t  seqcount
│   ├── seqcount_t       seqcount
│   │   ├── unsigned     sequence       ← العداد الفعلي
│   │   └── lockdep_map  dep_map        ← (DEBUG_LOCK_ALLOC only)
│   │       ├── lock_class_key *key
│   │       └── lock_class *class_cache[]
│   └── spinlock_t      *lock           ← (LOCKDEP/PREEMPT_RT only)
│       └── raw_spinlock_t rlock        ← أو rt_mutex_base على PREEMPT_RT
└── spinlock_t           lock           ← الـ embedded spinlock للـ writers
    └── raw_spinlock_t   rlock

seqcount_latch_t
└── seqcount_t           seqcount       ← نفس الـ seqcount_t العادي

seqcount_LOCKNAME_t  (مثال: seqcount_mutex_t)
├── seqcount_t           seqcount
└── struct mutex        *lock           ← pointer — مش embedded
```

---

### 3. رسم العلاقات بين الـ Structs (ASCII)

```
┌─────────────────────────────────────────────────────────┐
│                      seqlock_t                          │
│                                                         │
│  ┌─────────────────────────────────────┐                │
│  │       seqcount_spinlock_t seqcount  │                │
│  │                                     │                │
│  │  ┌──────────────────────────────┐   │                │
│  │  │       seqcount_t seqcount    │   │                │
│  │  │  unsigned sequence  ← [0]   │   │                │
│  │  │  lockdep_map dep_map ←[1]   │   │                │
│  │  │    └─► lock_class_key *key  │   │                │
│  │  └──────────────────────────────┘   │                │
│  │  spinlock_t *lock ──────────────────┼──────┐         │
│  └─────────────────────────────────────┘      │         │
│                                               │         │
│  ┌─────────────────────┐                      │         │
│  │  spinlock_t lock ◄──┼──────────────────────┘         │
│  │  raw_spinlock rlock │  ← يشير لنفس الـ lock          │
│  └─────────────────────┘                                │
└─────────────────────────────────────────────────────────┘

[0] = الـ bit 0 هو مؤشر "write in progress"
[1] = موجود بس لو CONFIG_DEBUG_LOCK_ALLOC
```

```
seqcount_latch_t                      latch_struct (user-defined)
┌──────────────────┐                  ┌───────────────────────────┐
│ seqcount_t       │    مضمّن في      │ seqcount_latch_t seq      │
│   unsigned seq   │ ◄────────────── │ data_struct    data[0]    │
└──────────────────┘                  │ data_struct    data[1]    │
                                      └───────────────────────────┘
                        seq & 1 ──► يحدد أي data[] يقرأه الـ reader
```

---

### 4. دورة حياة الـ Structs

#### دورة حياة `seqcount_t`

```
  STATIC INIT                    DYNAMIC INIT
  ───────────                    ────────────
  SEQCNT_ZERO(name)              seqcount_init(s)
  {.sequence = 0}                 ├─ __seqcount_init(s, #s, &key)
                                  │    ├─ lockdep_init_map(&s->dep_map, ...)
                                  │    └─ s->sequence = 0
                                  │
       ┌──────────────────────────┘
       ▼
  [READY: sequence = 0 (even)]
       │
       │  Writer                     Reader
       │  ──────                     ──────
       ├─► write_seqcount_begin(s)   ├─► read_seqcount_begin(s)
       │    ├─ seqprop_assert(s)     │    ├─ seqcount_lockdep_reader_access()
       │    ├─ preempt_disable()?    │    └─► raw_read_seqcount_begin(s)
       │    └─ s->sequence++ (odd)   │         └─ wait while (seq & 1)
       │    └─ smp_wmb()             │         └─ return seq (even)
       │                             │
       ├─► [WRITE DATA]              ├─► [READ DATA]
       │                             │
       ├─► write_seqcount_end(s)     └─► read_seqcount_retry(s, start)
       │    ├─ smp_wmb()                  ├─ smp_rmb()
       │    ├─ s->sequence++ (even)       └─ return seq != start
       │    └─ preempt_enable()?
       │
       └─► [READY again]
```

#### دورة حياة `seqlock_t`

```
  STATIC INIT                    DYNAMIC INIT
  ───────────                    ────────────
  DEFINE_SEQLOCK(sl)             seqlock_init(sl)
  __SEQLOCK_UNLOCKED(sl)          ├─ spin_lock_init(&sl->lock)
                                  └─ seqcount_spinlock_init(
                                        &sl->seqcount, &sl->lock)

       ┌───────────────────────────────────┐
       ▼                                   │
  [READY]                                  │
       │                                   │
  write_seqlock(sl)               read_seqbegin(sl)
  ├─ spin_lock(&sl->lock)         └─ read_seqcount_begin(&sl->seqcount)
  └─ s->sequence++ (odd)              └─ wait while odd → return even seq
       │                                   │
  [WRITE DATA]                        [READ DATA]
       │                                   │
  write_sequnlock(sl)             read_seqretry(sl, start)
  ├─ smp_wmb()                    └─ read_seqcount_retry(...)
  ├─ s->sequence++ (even)             └─ true → retry loop
  └─ spin_unlock(&sl->lock)           └─ false → done
       │
       └─► [READY]
```

#### دورة حياة `seqcount_latch_t`

```
  INIT
  ─────
  seqcount_latch_init(s)  ─►  s->seqcount.sequence = 0

  WRITE (externally serialized)
  ─────────────────────────────
  write_seqcount_latch_begin(s)    ← kcsan begin + smp_wmb + seq++
  modify(data[0], ...)             ← الـ readers دلوقتي بيقروا data[1]
  write_seqcount_latch(s)          ← smp_wmb + seq++
  modify(data[1], ...)             ← الـ readers دلوقتي بيقروا data[0]
  write_seqcount_latch_end(s)      ← kcsan end

  READ (lockless, NMI-safe)
  ──────────────────────────
  do {
    seq = read_seqcount_latch(s);    ← READ_ONCE(sequence)
    idx = seq & 0x01;                ← اختار النسخة
    entry = data_query(data[idx]);   ← اقرأ البيانات
  } while (read_seqcount_latch_retry(s, seq));  ← تحقق + smp_rmb
```

---

### 5. مخططات تدفق الاستدعاء

#### مسار الـ Reader في `seqcount_t`

```
read_seqcount_begin(s)
  └─► seqcount_lockdep_reader_access(seqprop_const_ptr(s))
        └─► seqcount_acquire_read(&dep_map, ...)   [lockdep only]
  └─► raw_read_seqcount_begin(s)
        └─► __read_seqcount_begin(s)
              ├─ __seq = seqprop_sequence(s)
              │     └─► smp_load_acquire(&s->sequence)
              │         [PREEMPT_RT + preemptible]:
              │         إذا كان فردي → lock(s->lock) + unlock(s->lock)
              │         ثم إعادة قراءة الـ sequence
              ├─ while (seq & 1): cpu_relax()     ← ينتظر لو writer شغّال
              └─ kcsan_atomic_next(1000)           ← KCSAN mark
              └─ return seq (even)

[READ PROTECTED DATA]

read_seqcount_retry(s, start)
  └─► do_read_seqcount_retry(seqprop_const_ptr(s), start)
        ├─ smp_rmb()                              ← memory barrier
        └─► do___read_seqcount_retry(s, start)
              ├─ kcsan_atomic_next(0)
              └─ return (READ_ONCE(s->sequence) != start)
                         ↑ true = retry needed
```

#### مسار الـ Writer في `seqcount_t`

```
write_seqcount_begin(s)
  ├─► seqprop_assert(s)
  │     └─ lockdep_assert_held(s->lock)   ← يتحقق إن الـ writer شايل الـ lock
  ├─ [if preemptible]: preempt_disable()
  └─► do_write_seqcount_begin(s)
        └─► do_write_seqcount_begin_nested(s, 0)
              ├─ seqcount_acquire(&dep_map, 0, ...)   [lockdep]
              └─► do_raw_write_seqcount_begin(s)
                    ├─ kcsan_nestable_atomic_begin()
                    ├─ s->sequence++   ← يصبح فردي
                    └─ smp_wmb()       ← barrier قبل الكتابة

[WRITE PROTECTED DATA]

write_seqcount_end(s)
  ├─► do_write_seqcount_end(s)
  │     ├─ seqcount_release(&dep_map, ...)   [lockdep]
  │     └─► do_raw_write_seqcount_end(s)
  │           ├─ smp_wmb()       ← barrier بعد الكتابة
  │           ├─ s->sequence++   ← يصبح زوجي
  │           └─ kcsan_nestable_atomic_end()
  └─ [if preemptible]: preempt_enable()
```

#### مسار الـ `seqlock_t` Writer

```
write_seqlock(sl)
  ├─ spin_lock(&sl->lock)                        ← يمنع writers تانيين
  └─► do_write_seqcount_begin(&sl->seqcount.seqcount)
        └─ (نفس مسار seqcount begin)

write_seqlock_irq(sl)          write_seqlock_bh(sl)       write_seqlock_irqsave(sl, flags)
├─ spin_lock_irq(&sl->lock)    ├─ spin_lock_bh(...)        ├─ spin_lock_irqsave(...)
└─ ...begin                    └─ ...begin                 └─ ...begin
```

#### مسار `scoped_seqlock_read` — الـ Smart Loop

```
scoped_seqlock_read(&lock, ss_lock) {
    // first iteration: ss_lockless
}
  │
  ├─ INIT: ss_tmp = { .state=ss_lockless, .data=read_seqbegin(lock) }
  │
  ▼
  loop iteration:
  ├─ [state == ss_done] → exit loop
  │
  ├─ [body executes with ss_lockless]
  │
  └─► __scoped_seqlock_next(&s, lock, ss_lock)
        ├─ state == ss_lockless:
        │    ├─ read_seqretry(lock, data) == false → state=ss_done (success)
        │    └─ read_seqretry == true (collision!) → fall through
        └─ target == ss_lock:
             ├─ s.lock = &lock->lock
             ├─ spin_lock(s.lock)           ← الآن في وضع locking
             └─ s.state = ss_lock

  loop iteration 2 (locking mode):
  ├─ [body executes again with lock held]
  └─► __scoped_seqlock_next(&s, lock, ss_lock)
        └─ state == ss_lock → state = ss_done → exit

  cleanup (__cleanup):
  └─► __scoped_seqlock_cleanup(&s)
        └─ s.lock != NULL → spin_unlock(s.lock)
```

#### مسار `read_seqbegin_or_lock` — الـ Hybrid Reader

```
int seq = 0;  /* يبدأ even */

retry:
read_seqbegin_or_lock(&lock, &seq)
  ├─ (!(*seq & 1)):  /* even → lockless */
  │    *seq = read_seqbegin(lock)   ← يجيب counter زوجي
  └─ (*seq & 1):   /* odd → locking */
       read_seqlock_excl(lock)      ← spin_lock

[read data]

if (need_seqretry(&lock, seq)):
    /* seq even AND read_seqretry returned true */
    seq = 1   /* force odd للـ iteration الجاي = يستخدم locking */
    goto retry

done_seqretry(&lock, seq)
  └─ (seq & 1) → read_sequnlock_excl(lock)  ← spin_unlock لو كنا locked
```

---

### 6. استراتيجية الـ Locking

#### جدول الـ Locks وما تحميه

| الـ Lock / آلية | بتحمي إيه | من مين |
|---|---|---|
| `spinlock_t lock` في `seqlock_t` | الـ write side | من writers تانيين وكمان من locking readers |
| `seqcount.sequence` LSB=1 | يشير لـ write-in-progress | يحمي الـ readers من قراءة بيانات غير consistent |
| `preempt_disable()` | يمنع preemption جوه write section | بيمنع deadlock لو الـ preemptor حاول يقرأ |
| `smp_wmb()` بعد seq++ | يضمن ترتيب الكتابة | يضمن إن الـ data مكتوبة قبل ما الـ sequence يتغير |
| `smp_rmb()` في retry | يضمن ترتيب القراءة | يضمن إن الـ data اتقرأت قبل ما نتحقق من الـ counter |
| `smp_load_acquire` في seqprop_sequence | load مع acquire semantics | يضمن إن ما في قراءة للـ data قبل الـ counter |

#### ترتيب الـ Locking (Lock Ordering)

```
لو محتاج seqlock_t + lock تاني:

    المسموح:
    ──────────
    spin_lock(other_lock)
      write_seqlock(sl)         ← seqlock من جوه lock تاني = OK
        [write]
      write_sequnlock(sl)
    spin_unlock(other_lock)

    الممنوع (Deadlock):
    ───────────────────
    write_seqlock(sl)
      spin_lock(other_lock)     ← ده ممكن يعمل deadlock لو
        [write]                    reader شايل other_lock وبيعمل retry
      spin_unlock(other_lock)
    write_sequnlock(sl)
```

#### الفرق بين `seqcount_t` و `seqlock_t` في الـ Locking

```
seqcount_t (بدون internal lock):
─────────────────────────────────
الـ writer MUST:
  1. يشيل external lock (spinlock, mutex, etc.)
  2. يعطّل preemption (أو يكون الـ lock بيعطّلها ضمنياً)
  3. لو readers من hardirq → يعطّل interrupts

seqlock_t (مع internal spinlock):
───────────────────────────────────
الـ writer:
  1. يستدعي write_seqlock() → بتعمل spin_lock + seq++
  2. لو من softirq context → write_seqlock_bh()
  3. لو من hardirq context → write_seqlock_irq() أو write_seqlock_irqsave()
الـ reader:
  1. lockless: read_seqbegin() / read_seqretry()
  2. locking: read_seqlock_excl() ← بيستخدم نفس الـ spinlock
```

#### ملاحظات PREEMPT_RT الخاصة

على PREEMPT_RT، الـ `spinlock_t` بيتحول لـ `rt_mutex` ـ لازم ينام. ده يكسر الـ assumption إن الـ write section مش preemptible. الحل:

```
Reader على PREEMPT_RT (seqcount_spinlock_t):
─────────────────────────────────────────────
__seqprop_spinlock_sequence(s):
  seq = smp_load_acquire(&s->seqcount.sequence)
  if (preemptible && unlikely(seq & 1)):
    /* writer شغّال وممكن اتـ preempt */
    spin_lock(s->lock)    ← ينتظر الـ writer يخلّص
    spin_unlock(s->lock)
    seq = smp_load_acquire(...)  ← يقرأ تاني بعد الانتظار
  return seq
```

ده بيضمن إن الـ reader ميـ spin-loopش لحد ما الـ preempted writer يرجع. بدل ما يعمل busy-wait، بيعمل proper sleep عن طريق الـ mutex.

#### ملاحظات KCSAN (Kernel Concurrency Sanitizer)

```
kcsan_atomic_next(KCSAN_SEQLOCK_REGION_MAX):
  ← بيقول لـ KCSAN: الـ 1000 access الجاية اعتبرها atomic
  ← ده لأن seqcount_t بتسمح بقراءة بيانات يمكن تتغير
  ← KCSAN ممكن يبلّغ عن false positives بدونه

kcsan_nestable_atomic_begin/end:
  ← بيحيط الـ write section
  ← بيقول لـ KCSAN إن الوصول ده متعمد وatomic بالنسبة للـ seqcount protocol
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ API — Cheatsheet

#### Category 1: Initialization

| Function / Macro | Type | الغرض |
|---|---|---|
| `__seqcount_init(s, name, key)` | `static inline` | تهيئة `seqcount_t` مع lockdep |
| `seqcount_init(s)` | macro | runtime init لـ `seqcount_t` |
| `SEQCNT_ZERO(name)` | macro | static initializer لـ `seqcount_t` |
| `seqcount_LOCKNAME_init(s, lock)` | macro | runtime init لـ `seqcount_LOCKNAME_t` |
| `SEQCOUNT_LOCKNAME_ZERO(name, lock)` | macro | static initializer لـ `seqcount_LOCKNAME_t` |
| `seqcount_latch_init(s)` | macro | runtime init لـ `seqcount_latch_t` |
| `SEQCNT_LATCH_ZERO(name)` | macro | static initializer لـ `seqcount_latch_t` |
| `seqlock_init(sl)` | macro | runtime init لـ `seqlock_t` |
| `DEFINE_SEQLOCK(sl)` | macro | static allocation + init لـ `seqlock_t` |

#### Category 2: seqcount_t — Reader Path

| Function / Macro | الغرض |
|---|---|
| `__read_seqcount_begin(s)` | بداية read section بدون lockdep — ينتظر لو counter فردي |
| `raw_read_seqcount_begin(s)` | alias لـ `__read_seqcount_begin` |
| `read_seqcount_begin(s)` | بداية read section مع lockdep |
| `raw_read_seqcount(s)` | قراءة raw للـ counter بدون انتظار stability |
| `raw_seqcount_begin(s)` | بداية بدون wait — بيعمل mask على LSB |
| `raw_seqcount_try_begin(s, start)` | بداية optimistic — بيرجع false لو كاتب شغال |
| `__read_seqcount_retry(s, start)` | فحص نهاية read section بدون `smp_rmb()` |
| `read_seqcount_retry(s, start)` | فحص نهاية read section مع `smp_rmb()` |

#### Category 3: seqcount_t — Writer Path

| Function / Macro | الغرض |
|---|---|
| `raw_write_seqcount_begin(s)` | بداية write section بدون lockdep |
| `raw_write_seqcount_end(s)` | نهاية write section بدون lockdep |
| `write_seqcount_begin(s)` | بداية write section مع lockdep |
| `write_seqcount_begin_nested(s, subclass)` | بداية write section مع custom lockdep nesting |
| `write_seqcount_end(s)` | نهاية write section مع lockdep |
| `raw_write_seqcount_barrier(s)` | ordering barrier بدون full write section |
| `write_seqcount_invalidate(s)` | invalidate كل الـ in-progress readers |

#### Category 4: seqcount_latch_t

| Function / Macro | الغرض |
|---|---|
| `raw_read_seqcount_latch(s)` | قراءة latch counter بدون KCSAN tracking |
| `read_seqcount_latch(s)` | قراءة latch counter مع KCSAN tracking |
| `raw_read_seqcount_latch_retry(s, start)` | فحص نهاية latch read بدون KCSAN |
| `read_seqcount_latch_retry(s, start)` | فحص نهاية latch read مع KCSAN |
| `raw_write_seqcount_latch(s)` | تقديم الـ latch counter خطوة واحدة |
| `write_seqcount_latch_begin(s)` | بداية latch write (توجيه readers للـ odd copy) |
| `write_seqcount_latch(s)` | منتصف latch write (توجيه readers للـ even copy) |
| `write_seqcount_latch_end(s)` | نهاية latch write section |

#### Category 5: seqlock_t — Lockless Reader

| Function / Macro | الغرض |
|---|---|
| `read_seqbegin(sl)` | بداية lockless read section |
| `read_seqretry(sl, start)` | فحص نهاية lockless read section |

#### Category 6: seqlock_t — Locking Reader

| Function / Macro | الغرض |
|---|---|
| `read_seqlock_excl(sl)` | بداية exclusive locking reader |
| `read_sequnlock_excl(sl)` | نهاية exclusive locking reader |
| `read_seqlock_excl_bh(sl)` | نفس الحاجة مع softirq disabled |
| `read_sequnlock_excl_bh(sl)` | unlock مع softirq re-enable |
| `read_seqlock_excl_irq(sl)` | نفس الحاجة مع hardirq disabled |
| `read_sequnlock_excl_irq(sl)` | unlock مع irq re-enable |
| `read_seqlock_excl_irqsave(lock, flags)` | lock مع irq save |
| `read_sequnlock_excl_irqrestore(sl, flags)` | unlock مع irq restore |

#### Category 7: seqlock_t — Writer Path

| Function / Macro | الغرض |
|---|---|
| `write_seqlock(sl)` | بداية write section (spin_lock + seqcount begin) |
| `write_sequnlock(sl)` | نهاية write section |
| `write_seqlock_bh(sl)` | write مع softirq disabled |
| `write_sequnlock_bh(sl)` | unlock مع softirq |
| `write_seqlock_irq(sl)` | write مع hardirq disabled |
| `write_sequnlock_irq(sl)` | unlock مع irq |
| `write_seqlock_irqsave(lock, flags)` | write مع irq save |
| `write_sequnlock_irqrestore(sl, flags)` | unlock مع irq restore |

#### Category 8: Hybrid Reader (lockless-or-locking)

| Function / Macro | الغرض |
|---|---|
| `read_seqbegin_or_lock(lock, seq)` | بداية hybrid — lockless لو even، locking لو odd |
| `need_seqretry(lock, seq)` | فحص هل محتاج retry |
| `done_seqretry(lock, seq)` | نهاية hybrid read section |
| `read_seqbegin_or_lock_irqsave(lock, seq)` | نفس الحاجة مع irqsave |
| `done_seqretry_irqrestore(lock, seq, flags)` | نهاية مع irqrestore |
| `scoped_seqlock_read(_seqlock, _target)` | C99-scoped hybrid reader — macro loop |

---

### Group 1: Initialization Functions

الـ initialization API عندنا ثلاثة مستويات: `seqcount_t` (raw counter بدون lock)، `seqcount_LOCKNAME_t` (counter مع lock مرتبط)، و `seqlock_t` (all-in-one). كل نوع عنده static initializer (لـ compile-time) ورuntime initializer (لـ dynamic allocation).

---

#### `__seqcount_init()`

```c
static inline void __seqcount_init(seqcount_t *s, const char *name,
                                   struct lock_class_key *key)
```

**الـ function دي هي الـ internal implementation** اللي بتعمل صفر للـ `sequence` counter وبتسجل الـ lock في lockdep map. بتتأكد إن الـ lock مش محتجز وقت الـ re-initialization عشان تكتشف bugs بدري.

- **`s`**: pointer للـ `seqcount_t` المراد تهيئتها
- **`name`**: اسم الـ instance لـ lockdep (بيظهر في debug output)
- **`key`**: lockdep class key — static في الـ caller عشان كل instance يتعرف lockdep إنه من نفس الـ class

**Return**: void

**Key details**: لو `CONFIG_DEBUG_LOCK_ALLOC` مش مفعل، الـ `name` و `key` بيبقوا NULL والـ lockdep calls بتتحول لـ no-ops. الـ caller هو دايمًا `seqcount_init()` macro.

---

#### `seqcount_init()` (macro)

```c
#define seqcount_init(s)                        \
    do {                                        \
        static struct lock_class_key __key;     \
        __seqcount_init((s), #s, &__key);       \
    } while (0)
```

**الـ macro الرسمي للـ runtime initialization.** بيخلي كل call site له `lock_class_key` static مستقل (لإن الـ key static في الـ macro expansion) — ده بيضمن إن كل متغير بـ اسم مختلف ياخد class مختلف في lockdep. المستخدم بيمرر بس الـ pointer.

- **`s`**: pointer لـ `seqcount_t`

**Who calls it**: أي كود محتاج يهيئ `seqcount_t` dynamically (في constructor، `init` function، إلخ).

---

#### `seqcount_LOCKNAME_init()` (macro family)

```c
#define seqcount_LOCKNAME_init(s, _lock, lockname)         \
    do {                                                    \
        seqcount_##lockname##_t *____s = (s);              \
        seqcount_init(&____s->seqcount);                   \
        __SEQ_LOCK(____s->lock = (_lock));                 \
    } while (0)

#define seqcount_raw_spinlock_init(s, lock) ...
#define seqcount_spinlock_init(s, lock)     ...
#define seqcount_rwlock_init(s, lock)       ...
#define seqcount_mutex_init(s, lock)        ...
```

**بتهيئ `seqcount_LOCKNAME_t`** اللي هي عبارة عن `seqcount_t` مع pointer لـ associated lock. ربط الـ lock بالـ counter بيمكّن lockdep إنه يتحقق إن الـ writer section بتأخذ الـ lock الصح. الـ `__SEQ_LOCK()` بيحط الـ lock pointer بس لو `CONFIG_LOCKDEP` أو `CONFIG_PREEMPT_RT` مفعلين.

- **`s`**: pointer لـ `seqcount_LOCKNAME_t`
- **`lock`**: pointer للـ associated lock (يكون موجود طول عمر الـ seqcount)

**Who calls it**: الـ subsystems اللي بتستخدم `seqcount_spinlock_t` أو غيرها — مثلًا `timekeeping_init()` بتستخدم `seqcount_raw_spinlock_init`.

---

#### `seqlock_init()` (macro)

```c
#define seqlock_init(sl)                                        \
    do {                                                        \
        spin_lock_init(&(sl)->lock);                           \
        seqcount_spinlock_init(&(sl)->seqcount, &(sl)->lock);  \
    } while (0)
```

**بيهيئ `seqlock_t` الـ all-in-one primitive.** بيهيئ الـ embedded `spinlock_t` وبعدين بيربطه بالـ `seqcount_spinlock_t`. الـ `seqlock_t` هو الـ high-level API اللي بيشيل على المبرمج مسؤولية اختيار نوع الـ lock.

- **`sl`**: pointer لـ `seqlock_t`

---

### Group 2: seqcount_t Reader Path

الـ read path في الـ seqcount بيعتمد على فكرة بسيطة: الـ writer بيزود الـ counter بـ 1 (يبقى فردي) وقت بداية الكتابة، وبيزوده تاني (يبقى زوجي) وقت نهايتها. الـ reader بيقرأ الـ counter قبل وبعد، لو اتغير يعمل retry.

```
Reader:  start = read_begin()  [يستنى counter يبقى زوجي]
         ... قرا البيانات ...
         if read_retry(start):  [لو counter اتغير → retry]
             goto start
```

---

#### `__read_seqcount_begin()` (macro)

```c
#define __read_seqcount_begin(s)                            \
({                                                           \
    unsigned __seq;                                          \
    while (unlikely((__seq = seqprop_sequence(s)) & 1))     \
        cpu_relax();                                         \
    kcsan_atomic_next(KCSAN_SEQLOCK_REGION_MAX);            \
    __seq;                                                   \
})
```

**هو الـ core implementation لبداية الـ read critical section.** بيشيل الـ current sequence value بعد ما يستنى لو كانت فردية (writer في منتصف كتابة). الـ `cpu_relax()` في الـ spin loop بيقلل الـ bus traffic على الـ x86. الـ `kcsan_atomic_next()` بيمارك الـ next N memory accesses كـ atomic لـ KCSAN.

- **`s`**: pointer لـ `seqcount_t` أو أي `seqcount_LOCKNAME_t`

**Return**: `unsigned` — قيمة الـ counter (دايمًا زوجية)، بتتمرر لـ `read_seqcount_retry()`.

**Key details**: الـ `seqprop_sequence()` على PREEMPT_RT بيعمل lock/unlock للـ associated lock لو الـ counter فردي — ده بيضمن إن preempted writer يكمل قبل ما الـ reader يتقدم. بدون `smp_rmb()` — الـ barrier بييجي في `read_seqcount_retry()`.

---

#### `read_seqcount_begin()` (macro)

```c
#define read_seqcount_begin(s)                          \
({                                                       \
    seqcount_lockdep_reader_access(seqprop_const_ptr(s));\
    raw_read_seqcount_begin(s);                          \
})
```

**الـ public API لبداية read critical section مع lockdep support.** الفرق الوحيد عن `raw_read_seqcount_begin()` هو إنه بيعمل `seqcount_lockdep_reader_access()` اللي بيبلغ lockdep إن reader access بيحصل — ده بيساعد في اكتشاف لو الـ writer مش بياخد الـ lock الصح.

**Who calls it**: الـ primary API الموصى بيه لكل readers عدا الـ hot paths اللي بتحتاج أقل overhead.

---

#### `raw_read_seqcount()` (macro)

```c
#define raw_read_seqcount(s)                        \
({                                                   \
    unsigned __seq = seqprop_sequence(s);            \
    kcsan_atomic_next(KCSAN_SEQLOCK_REGION_MAX);    \
    __seq;                                           \
})
```

**بيقرأ الـ raw counter value بدون ما ينتظر الـ counter يستقر.** الـ LSB ممكن تكون 1 (writer active). الكود الاستدعائي مسؤول عن handling الـ odd value. مفيد لو محتاج تعرف هل writer شغال ولا لأ قبل ما تحدد مسار.

**Return**: `unsigned` — raw counter value (ممكن تكون فردية).

---

#### `raw_seqcount_try_begin()` (macro)

```c
#define raw_seqcount_try_begin(s, start)    \
({                                          \
    start = raw_read_seqcount(s);           \
    !(start & 1);                           \
})
```

**Optimistic begin:** لو الـ counter زوجي (مفيش كاتب)، بيحط `start` وبيرجع `true`. لو فردي (كاتب شغال) بيرجع `false` ومش بيعمل spin wait. ده مفيد لو عندك slowpath تعمله لو writer active بدل ما تعمل speculation إنك هتحتاج retry.

**Return**: `bool` — `true` لو read section اتبدأ بنجاح.

---

#### `raw_seqcount_begin()` (macro)

```c
#define raw_seqcount_begin(s) (raw_read_seqcount(s) & ~1)
```

**بياخد snapshot بدون spin wait وبيعمل mask على الـ LSB.** لو كاتب كان شغال وقت القراءة، الـ `read_seqcount_retry()` هيفشل guaranteed لأن الـ counter هيبقى اتغير. ده بيوفر branch واحدة في الـ hot path على حساب إن retry محتمل أكتر.

**Who calls it**: very tight loops زي vDSO time reading أو scheduler hot paths.

---

#### `do___read_seqcount_retry()` / `__read_seqcount_retry()` (macro)

```c
static inline int do___read_seqcount_retry(const seqcount_t *s, unsigned start)
{
    kcsan_atomic_next(0);
    return unlikely(READ_ONCE(s->sequence) != start);
}

#define __read_seqcount_retry(s, start) \
    do___read_seqcount_retry(seqprop_const_ptr(s), start)
```

**بيتحقق لو الـ counter اتغير بدون إصدار `smp_rmb()`.** الـ caller مسؤول عن ضمان ordering عن طريق طريقة تانية (مثلًا `smp_read_barrier_depends` أو explicit barrier قبل الـ data reads). الاستخدام نادر ودقيق.

- **`s`**: pointer للـ seqcount
- **`start`**: القيمة اللي جت من `read_seqcount_begin()`

**Return**: `int` — nonzero لو retry مطلوب.

**Key details**: الـ `READ_ONCE()` بيضمن إن الـ compiler ما يعملش load merging أو hoisting.

---

#### `do_read_seqcount_retry()` / `read_seqcount_retry()` (macro)

```c
static inline int do_read_seqcount_retry(const seqcount_t *s, unsigned start)
{
    smp_rmb();
    return do___read_seqcount_retry(s, start);
}

#define read_seqcount_retry(s, start) \
    do_read_seqcount_retry(seqprop_const_ptr(s), start)
```

**ده الـ standard end-of-read function.** الـ `smp_rmb()` (read memory barrier) بيضمن إن كل الـ data reads اللي حصلت في الـ critical section كانت completed قبل ما يقارن الـ counter. ده هو الـ correct ordering protocol: load data → `smp_rmb()` → check counter.

- **`s`**: pointer للـ seqcount
- **`start`**: القيمة من `read_seqcount_begin()`

**Return**: nonzero لو الـ read section كانت invalid وتحتاج retry.

**Pseudocode flow**:
```
do {
    start = read_seqcount_begin(&sc);   // spin till even, record value
    // ... read shared data ...
} while (read_seqcount_retry(&sc, start));
//                           ^ smp_rmb() هنا بيحمي الـ data reads فوق
```

---

### Group 3: seqcount_t Writer Path

الـ write path بيزود الـ counter مرتين: مرة عند البداية (يبقى فردي) ومرة عند النهاية (يبقى زوجي). الـ `smp_wmb()` barriers بيضمنوا إن الـ writes للـ protected data ما يتسربوش خارج الـ critical section.

```
write_seqcount_begin():  seq++  (odd)  + smp_wmb()
... كتابة البيانات ...
write_seqcount_end():    smp_wmb() + seq++ (even)
```

---

#### `do_raw_write_seqcount_begin()` / `raw_write_seqcount_begin()` (macro)

```c
static inline void do_raw_write_seqcount_begin(seqcount_t *s)
{
    kcsan_nestable_atomic_begin();
    s->sequence++;
    smp_wmb();
}

#define raw_write_seqcount_begin(s)                 \
do {                                                 \
    if (seqprop_preemptible(s))                      \
        preempt_disable();                           \
    do_raw_write_seqcount_begin(seqprop_ptr(s));     \
} while (0)
```

**بيبدأ write critical section بدون lockdep.** بيزود الـ counter (يبقى فردي) وبيضع `smp_wmb()` عشان الـ readers يشوفوا الـ odd counter قبل ما يشوفوا أي data modifications. لو الـ associated lock preemptible (mutex أو spinlock على RT)، بيعمل `preempt_disable()` أول.

**Locking**: بيفترض إن الـ caller بياخد الـ writer serialization lock بنفسه قبل ما يستدعيه. مفيش implicit lock هنا.

**Who calls it**: subsystems اللي بيديروا الـ locking بنفسهم وعارفين إيه بيعملوا — زي `timekeeping`.

---

#### `do_raw_write_seqcount_end()` / `raw_write_seqcount_end()` (macro)

```c
static inline void do_raw_write_seqcount_end(seqcount_t *s)
{
    smp_wmb();
    s->sequence++;
    kcsan_nestable_atomic_end();
}
```

**ده الـ close للـ raw write section.** الـ `smp_wmb()` قبل الـ increment بيضمن إن كل الـ data stores وصلوا للـ memory قبل ما الـ counter يبقى زوجي تاني. الـ reader اللي بيشوف الـ even counter بيضمن كده إن البيانات consistent.

**Key details**: الترتيب — data stores → `smp_wmb()` → counter++ — ده أساسي. تغييره بيكسر الـ protocol.

---

#### `write_seqcount_begin()` / `write_seqcount_begin_nested()` (macros)

```c
#define write_seqcount_begin(s)             \
do {                                         \
    seqprop_assert(s);                       \
    if (seqprop_preemptible(s))              \
        preempt_disable();                   \
    do_write_seqcount_begin(seqprop_ptr(s)); \
} while (0)
```

**الـ public API للـ write section مع lockdep.** `seqprop_assert(s)` بيعمل `lockdep_assert_held(s->lock)` — لو الـ associated lock مش محتجز هيطلع lockdep warning. الـ `_nested` variant بيسمح بتحديد subclass لـ nested seqcount usage.

- **`subclass`** (في `_nested`): lockdep nesting level (0 = default)

**Who calls it**: الكود الـ general الموصى بيه لكل writers اللي بيستخدموا `seqcount_LOCKNAME_t`.

---

#### `write_seqcount_end()` (macro)

```c
#define write_seqcount_end(s)               \
do {                                         \
    do_write_seqcount_end(seqprop_ptr(s));   \
    if (seqprop_preemptible(s))              \
        preempt_enable();                    \
} while (0)
```

**بينهي الـ write section ويبلغ lockdep ويعيد تشغيل الـ preemption.** الترتيب مقلوب عن `begin`: أول بيزود الـ counter (even)، بعدين `preempt_enable()`. ده مهم عشان لو preemption اتعمل enable قبل increment الـ readers ممكن يشوفوا odd counter بدون active writer.

---

#### `do_raw_write_seqcount_barrier()` / `raw_write_seqcount_barrier()` (macro)

```c
static inline void do_raw_write_seqcount_barrier(seqcount_t *s)
{
    kcsan_nestable_atomic_begin();
    s->sequence++;
    smp_wmb();
    s->sequence++;
    kcsan_nestable_atomic_end();
}
```

**ده مش write section — ده ordering barrier بس.** بيزود الـ counter مرتين بسرعة (odd → even) عشان يضمن إن أي reader بدأ قبل الـ barrier هيعمل retry. الفرق عن الـ normal write pattern: مفيش `smp_wmb()` أول (ده بيوفر wmb واحدة). الـ writes المحيطة بالـ barrier لازم تكون declared atomic (عن طريق `WRITE_ONCE`).

**Use case**: لما عندك data updates متفرقة وعايز تضمن ordering بدون full write section.

---

#### `write_seqcount_invalidate()` (macro)

```c
static inline void do_write_seqcount_invalidate(seqcount_t *s)
{
    smp_wmb();
    kcsan_nestable_atomic_begin();
    s->sequence += 2;
    kcsan_nestable_atomic_end();
}
```

**بيعمل invalidate لكل الـ in-progress readers.** بيزود الـ counter بـ 2 (يفضل زوجي، مفيش write section) بعد `smp_wmb()`. أي reader بدأ قبل الـ invalidate هيشوف counter اتغير وهيعمل retry. مفيد لما عايز تضمن إن readers يشوفوا state جديد حتى لو مش عارف مين قرأ.

**Who calls it**: cache invalidation paths، configuration updates اللي تحتاج flush للـ stale reads.

---

### Group 4: seqcount_latch_t — Double-Buffer Pattern

الـ latch pattern هو **multi-version concurrency control** بيخلي الـ NMI handlers والـ readers اللي ممكن يقاطعوا الـ writer يشتغلوا بأمان. الحل: نتيلك نسختين من البيانات (`data[0]` و `data[1]`). الـ writer بيكتب في نسخة بينما الـ readers بيقروا من التانية.

```
الـ LSB في الـ counter بيحدد أي copy الـ reader يقرأ:
  seq & 1  == 0  → readers يقروا data[0]
  seq & 1  == 1  → readers يقروا data[1]

Write sequence:
  write_seqcount_latch_begin()  → seq++ (→ odd) → readers يقروا data[1]
  modify data[0]
  write_seqcount_latch()        → seq++ (→ even) → readers يقروا data[0]
  modify data[1]
  write_seqcount_latch_end()    → done
```

---

#### `raw_read_seqcount_latch()` / `read_seqcount_latch()`

```c
static __always_inline unsigned raw_read_seqcount_latch(const seqcount_latch_t *s)
{
    return READ_ONCE(s->seqcount.sequence);
}

static __always_inline unsigned read_seqcount_latch(const seqcount_latch_t *s)
{
    kcsan_atomic_next(KCSAN_SEQLOCK_REGION_MAX);
    return raw_read_seqcount_latch(s);
}
```

**بيقرأ الـ current latch counter.** الـ LSB بيحدد أي `data[]` copy الـ reader يستخدم. مفيش spin wait هنا — الـ latch design بيضمن دايمًا فيه نسخة valid متاحة. الـ `READ_ONCE()` بيمنع الـ compiler من reordering أو merging للـ load.

**Return**: raw counter value — استخدم `(seq & 1)` كـ index في `data[]`.

**Key details**: المفيش `smp_rmb()` هنا — الـ dependent load على الـ index بيوفر ordering implicit على architectures زي ARM. الـ `raw_read_seqcount_latch_retry()` بتضيف الـ `smp_rmb()` الصريح.

---

#### `raw_read_seqcount_latch_retry()` / `read_seqcount_latch_retry()`

```c
static __always_inline int
raw_read_seqcount_latch_retry(const seqcount_latch_t *s, unsigned start)
{
    smp_rmb();
    return unlikely(READ_ONCE(s->seqcount.sequence) != start);
}
```

**نهاية الـ latch read section.** الـ `smp_rmb()` بيضمن إن data reads حصلت قبل counter check. لو الـ counter اتغير، الـ reader لازم يقرأ تاني من الـ index الجديد.

**Full latch read pattern**:
```c
unsigned seq, idx;
do {
    seq = read_seqcount_latch(&latch->seq);
    idx = seq & 0x01;
    entry = data_query(latch->data[idx], key);
} while (read_seqcount_latch_retry(&latch->seq, seq));
```

---

#### `raw_write_seqcount_latch()`

```c
static __always_inline void raw_write_seqcount_latch(seqcount_latch_t *s)
{
    smp_wmb();    /* stores قبل increment */
    s->seqcount.sequence++;
    smp_wmb();    /* increment قبل stores التالية */
}
```

**القلب بتاع الـ latch write mechanism.** كل `smp_wmb()` له غرض محدد: الأول بيضمن إن modifications للـ current copy وصلت قبل ما الـ counter يتغير (readers يتحولوا للـ copy التانية). التاني بيضمن إن الـ counter visible قبل ما نبدأ نعدل الـ copy التانية.

---

#### `write_seqcount_latch_begin()` / `write_seqcount_latch()` / `write_seqcount_latch_end()`

```c
static __always_inline void write_seqcount_latch_begin(seqcount_latch_t *s)
{
    kcsan_nestable_atomic_begin();
    raw_write_seqcount_latch(s);
}

static __always_inline void write_seqcount_latch(seqcount_latch_t *s)
{
    raw_write_seqcount_latch(s);
}

static __always_inline void write_seqcount_latch_end(seqcount_latch_t *s)
{
    kcsan_nestable_atomic_end();
}
```

**ثلاثي الـ latch write API.** `begin` بيبدأ KCSAN tracking ويحول الـ readers للـ odd copy. `write_seqcount_latch()` (منتصف العملية) بيعدل الـ counter مرة تانية ليحول الـ readers للـ even copy بعد إن تم تعديل الـ first copy. `end` بينهي الـ KCSAN tracking.

**Caller context**: يلزم external serialization — الـ latch write sections مش thread-safe مع بعض.

---

### Group 5: seqlock_t — Lockless Reader

الـ `seqlock_t` هو الـ high-level API اللي بيجمع seqcount + spinlock. الـ lockless reader path مشابه تمامًا للـ seqcount، بس الـ abstraction أوضح.

---

#### `read_seqbegin()`

```c
static inline unsigned read_seqbegin(const seqlock_t *sl)
    __acquires_shared(sl) __no_context_analysis
{
    return read_seqcount_begin(&sl->seqcount);
}
```

**بيبدأ lockless read critical section على `seqlock_t`.** هو wrapper بسيط حول `read_seqcount_begin()`. الـ `__acquires_shared(sl)` annotation بيبلغ sparse إن shared ownership بدأت — ده لـ static analysis بس، مفيش actual lock.

**Return**: sequence count لتمريره لـ `read_seqretry()`.

**Who calls it**: أي reader محتاج يقرأ data محمية بـ `seqlock_t` بدون blocking.

---

#### `read_seqretry()`

```c
static inline unsigned read_seqretry(const seqlock_t *sl, unsigned start)
    __releases_shared(sl) __no_context_analysis
{
    return read_seqcount_retry(&sl->seqcount, start);
}
```

**بيغلق الـ lockless read section ويتحقق من consistency.** لو رجع nonzero، الـ read section كانت تحت writer activity وكل الـ data المقروءة potentially stale وتحتاج retry.

**Pattern**:
```c
unsigned start;
do {
    start = read_seqbegin(&sl);
    /* قرا البيانات */
} while (read_seqretry(&sl, start));
```

---

### Group 6: seqlock_t — Locking Reader

الـ locking reader بياخد الـ spinlock نفسه اللي بياخده الـ writer. ده بيضمن consistency تامة بدون retry، بس على حساب إنه blocking.

---

#### `read_seqlock_excl()` وعائلتها

```c
static inline void read_seqlock_excl(seqlock_t *sl)
{
    spin_lock(&sl->lock);
}
static inline void read_sequnlock_excl(seqlock_t *sl)
{
    spin_unlock(&sl->lock);
}
```

**بياخد/بيرجع الـ embedded spinlock كـ exclusive reader.** الـ locking reader بيحجب الـ writers وكمان الـ locking readers التانيين. الـ lockless readers مش محجوبين (بيعملوا retry بس لو writer حجز).

**Variants وسياق استخدامهم**:

| Function | متى تستخدمه |
|---|---|
| `read_seqlock_excl` | الـ read/write contexts كلهم process context |
| `read_seqlock_excl_bh` | لو writer أو reader تاني ممكن يجي من softirq |
| `read_seqlock_excl_irq` | لو writer أو reader تاني ممكن يجي من hardirq |
| `read_seqlock_excl_irqsave` | لو hardirq ممكن وعايز تحفظ الـ irq state |

**Key details**: الـ locking reader مش بيزود الـ sequence counter. ده معناه إن lockless readers موجودين في نفس الوقت ممكن يكملوا لو الـ locking reader ماعملش modification.

---

### Group 7: seqlock_t — Writer Path

---

#### `write_seqlock()` / `write_sequnlock()`

```c
static inline void write_seqlock(seqlock_t *sl)
{
    spin_lock(&sl->lock);
    do_write_seqcount_begin(&sl->seqcount.seqcount);
}

static inline void write_sequnlock(seqlock_t *sl)
{
    do_write_seqcount_end(&sl->seqcount.seqcount);
    spin_unlock(&sl->lock);
}
```

**الأساس بتاع الـ seqlock write path.** `write_seqlock` بياخد الـ spinlock (بيضمن writer serialization) وبعدين بيزود الـ counter (odd). `write_sequnlock` بيزود الـ counter تاني (even) وبيرجع الـ spinlock. الترتيب في `sequnlock` — counter++ قبل spin_unlock — ده أساسي: لو spinlock اترجع أول ممكن writer تاني يبدأ بدون ما الـ readers يتنبهوا إن الـ first write انتهى.

**Who calls it**: كل كود بيعدل data محمية بـ `seqlock_t` من process context.

**Variants وسياق استخدامهم**:

| Write Function | متى تستخدمه |
|---|---|
| `write_seqlock` | process context فقط |
| `write_seqlock_bh` | writers أو readers من softirq |
| `write_seqlock_irq` | writers أو readers من hardirq |
| `write_seqlock_irqsave` | hardirq ممكن، مع حفظ irq state |

---

### Group 8: seqlock_t — Hybrid Reader (lockless-or-locking)

الـ hybrid API بيحل مشكلة **writer starvation**: لو الـ writers نشطين جداً، الـ lockless readers ممكن يدوروا في retry loop بلا نهاية. الحل: بعد عدد معين من المحاولات، الـ reader يتحول لـ locking reader يقفل بيانات consistent.

---

#### `read_seqbegin_or_lock()`

```c
static inline void read_seqbegin_or_lock(seqlock_t *lock, int *seq)
{
    if (!(*seq & 1))    /* Even → lockless */
        *seq = read_seqbegin(lock);
    else                /* Odd → locking */
        read_seqlock_excl(lock);
}
```

**بيبدأ إما lockless أو locking read حسب قيمة `*seq`.** أول call من caller بتكون `*seq = 0` (even) — يبدأ lockless. لو `need_seqretry()` قرر إننا محتاجين retry، بيضع `*seq = 1` (odd) والـ call التالية تعمل locking.

- **`lock`**: pointer لـ `seqlock_t`
- **`seq`**: in/out parameter — 0 أو even = lockless، 1 أو odd = locking

**Who calls it**: filesystem code، VFS layers اللي بيقرؤوا path components.

---

#### `need_seqretry()`

```c
static inline int need_seqretry(seqlock_t *lock, int seq)
{
    return !(seq & 1) && read_seqretry(lock, seq);
}
```

**الـ retry check للـ hybrid pattern.** لو `seq` كان odd (locking reader)، بيرجع 0 دايمًا (mفيش retry محتاجة، البيانات consistent). لو كان even، بيفحص الـ seqcount counter بـ `read_seqretry()`.

**Return**: nonzero لو الـ loop لازم تتكرر.

---

#### `done_seqretry()`

```c
static inline void done_seqretry(seqlock_t *lock, int seq)
{
    if (seq & 1)
        read_sequnlock_excl(lock);
}
```

**بينهي الـ hybrid read section.** لو آخر iteration كانت locking (seq & 1)، بيرجع الـ spinlock. لو lockless، مفيش حاجة تعملها.

**Full hybrid pattern**:
```c
int seq = 0;  /* ابدأ بـ even */
do {
    read_seqbegin_or_lock(&sl, &seq);
    /* ... قرا البيانات ... */
} while (need_seqretry(&sl, seq));
done_seqretry(&sl, seq);
```

---

#### `read_seqbegin_or_lock_irqsave()` / `done_seqretry_irqrestore()`

```c
static inline unsigned long
read_seqbegin_or_lock_irqsave(seqlock_t *lock, int *seq)
{
    unsigned long flags = 0;
    if (!(*seq & 1))
        *seq = read_seqbegin(lock);
    else
        read_seqlock_excl_irqsave(lock, flags);
    return flags;
}
```

**نفس الـ hybrid pattern بس مع hardirq protection.** الـ irq disable بيحصل بس في الـ locking mode (odd). في الـ lockless mode، مفيش interrupts disabled لأن الـ lockless readers مش محتاجين حماية من IRQ interruption بحد ذاتها. الـ `flags` بيكون 0 في الـ lockless case وبيتجاهل في `done_seqretry_irqrestore()` accordingly.

---

### Group 9: Scoped Seqlock Reader (Structured Pattern)

الـ `scoped_seqlock_read` macro بيوفر **C cleanup-based scoping** للـ hybrid read pattern. بيتجنب الـ manual retry loop وبيضمن proper cleanup حتى لو حصل early return.

---

#### `scoped_seqlock_read()` (macro)

```c
#define scoped_seqlock_read(_seqlock, _target)              \
    __scoped_seqlock_read(_seqlock, _target, __UNIQUE_ID(seqlock))

#define __scoped_seqlock_read(_seqlock, _target, _s)        \
    for (struct ss_tmp _s __cleanup(__scoped_seqlock_cleanup) =     \
         { .state = ss_lockless, .data = read_seqbegin(_seqlock) }, \
         *__UNIQUE_ID(ctx) __cleanup(__scoped_seqlock_cleanup_ctx) =\
            (struct ss_tmp *)_seqlock;                              \
         _s.state != ss_done;                                       \
         __scoped_seqlock_next(&_s, _seqlock, _target))
```

**بيعمل for-loop بيشتغل على `ss_tmp` struct يمثل حالة الـ reader.** أول iteration دايمًا lockless. لو `__scoped_seqlock_next()` اكتشفت إن retry مطلوب، بتبدأ iteration جديدة بالـ mode اللي حدده الـ `_target`. الـ `__cleanup` attribute بيضمن إن الـ lock يترجع لو اتعمل break أو return.

- **`_seqlock`**: pointer لـ `seqlock_t`
- **`_target`**: `ss_lock` أو `ss_lock_irqsave` أو `ss_lockless` — النوع المراد للـ fallback

**Usage**:
```c
scoped_seqlock_read(&my_lock, ss_lock) {
    /* read critical section — auto retry/cleanup */
    local_var = shared_data->value;
}
```

---

#### `__scoped_seqlock_next()`

```c
static __always_inline void
__scoped_seqlock_next(struct ss_tmp *sst, seqlock_t *lock, enum ss_state target)
```

**هو الـ state machine بتاع الـ scoped reader.** بيتحقق من الـ current state وبيقرر الخطوة الجاية:
- لو `ss_lock` أو `ss_lock_irqsave`: يضع state = `ss_done` (انتهى الـ loop)
- لو `ss_lockless`: يفحص `read_seqretry()`. لو نجح → `ss_done`. لو فشل → ياخد الـ lock بالـ target mode وينفذ iteration جديدة.

**Pseudocode flow**:
```
switch current state:
  ss_done:       → impossible, bug
  ss_lock/irq:   → set done, exit loop
  ss_lockless:   → if seqretry ok: set done
                    else: acquire lock (target mode), set state = target
```

---

#### `__scoped_seqlock_cleanup()`

```c
static __always_inline void __scoped_seqlock_cleanup(struct ss_tmp *sst)
{
    if (sst->lock)
        spin_unlock(sst->lock);
    if (sst->lock_irqsave)
        spin_unlock_irqrestore(sst->lock_irqsave, sst->data);
}
```

**بيرجع أي locks حجزها الـ scoped reader عند خروج الـ scope.** ده الـ cleanup handler اللي بيستدعيه الـ compiler تلقائيًا لما الـ `for` loop ينتهي — سواء normally أو عن طريق `break`/`return`. بيضمن مفيش lock leaks.

---

### Group 10: seqprop — Generic Property Accessors

الـ `seqprop_*` macros بتستخدم `_Generic` (C11) عشان تشتغل مع أي نوع seqcount بدون casting.

---

#### `__seqprop()` وعائلتها

```c
#define __seqprop(s, prop) _Generic(*(s),       \
    seqcount_t:         __seqprop_##prop,        \
    seqcount_raw_spinlock_t: ...,                \
    seqcount_spinlock_t: ...,                    \
    seqcount_rwlock_t: ...,                      \
    seqcount_mutex_t: ...)

#define seqprop_ptr(s)          __seqprop(s, ptr)(s)
#define seqprop_sequence(s)     __seqprop(s, sequence)(s)
#define seqprop_preemptible(s)  __seqprop(s, preemptible)(s)
#define seqprop_assert(s)       __seqprop(s, assert)(s)
```

**ده الـ polymorphism layer.** بدلًا من إن كل function تعرف نوع الـ seqcount اللي بيتعامل معاه، `_Generic` بيختار تلقائيًا الـ implementation الصح compile-time.

| Property | الغرض |
|---|---|
| `seqprop_ptr(s)` | يرجع `seqcount_t *` من أي نوع |
| `seqprop_const_ptr(s)` | نفس الحاجة بس const |
| `seqprop_sequence(s)` | يقرأ الـ counter (مع RT spin logic) |
| `seqprop_preemptible(s)` | هل الـ associated lock preemptible؟ |
| `seqprop_assert(s)` | assert إن writer lock محتجز |

**Key detail في `seqprop_sequence()` على PREEMPT_RT**: لو الـ lock preemptible والـ counter فردي (writer active but possibly preempted)، بيعمل lock/unlock للـ associated lock. ده بيعطي الـ writer فرصة إنه يكمل — بدون ده، reader ممكن يدور في busy-wait لفترة طويلة.

---

### ملاحظات ختامية على الـ Memory Ordering

| Barrier | مكانه | الغرض |
|---|---|---|
| `smp_wmb()` بعد `seq++` (begin) | Write begin | يضمن readers يشوفوا odd counter قبل data writes |
| `smp_wmb()` قبل `seq++` (end) | Write end | يضمن data writes visible قبل even counter |
| `smp_rmb()` في `read_seqcount_retry()` | Read end | يضمن data reads complete قبل counter check |
| `smp_load_acquire()` في `seqprop_sequence()` | Read begin | implicit barrier مع dependent loads |

الـ protocol صح بس لو الـ full chain محترم: `smp_wmb()` من الـ writer → `smp_load_acquire()` من الـ reader → data reads → `smp_rmb()` → counter check. تكسير أي حلقة في السلسلة دي بيخلي الـ seqlock inconsistent.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs entries

الـ `seqlock` مش ليه entries مباشرة في debugfs، لكن الـ **lockdep** اللي بيحمي الـ `seqcount_t` بيكتب في:

```bash
# اقرأ lockdep graph كامل
cat /sys/kernel/debug/lockdep

# اقرأ كل held locks حالياً
cat /sys/kernel/debug/lockdep_chains

# اقرأ stats الـ lockdep
cat /sys/kernel/debug/lockdep_stats
```

تفسير الـ output:
- **`lockdep_stats`**: فيه `lock classes`, `direct dependencies`, `chain entries`. لو عدد الـ chains كبير جداً → مشكلة في lock nesting.
- **`lockdep_chains`**: كل سطر بيمثل chain من الـ locks المتداخلة — بيساعد تشوف لو `seqcount_t` اتعملها acquire قبل lock تاني بشكل خاطئ.

---

#### 2. sysfs entries

```bash
# كشف الـ kernel lock stats (لو مفعّل CONFIG_LOCK_STAT)
ls /sys/kernel/debug/locking/

# spinlock اللي جوا seqlock_t
cat /proc/lock_stat  # بيشمل spinlock_t المدمج
```

الـ `/proc/lock_stat` بيديك لكل lock:

| العمود | المعنى |
|--------|--------|
| `con-bounces` | مرات الـ contention مع cache bounce |
| `contentions` | إجمالي مرات الانتظار |
| `wait-time avg` | متوسط وقت الانتظار بـ nanoseconds |
| `acq-bounces` | مرات الـ acquire مع cache miss |

---

#### 3. ftrace — tracepoints/events

الـ seqlock مش عنده tracepoints مخصصة، لكن تقدر تتبعه بـ:

```bash
# تفعيل function tracing لكل دوال الـ seqlock
cd /sys/kernel/debug/tracing

echo 0 > tracing_on
echo function > current_tracer
echo 'do_raw_write_seqcount_begin' > set_ftrace_filter
echo 'do_raw_write_seqcount_end'  >> set_ftrace_filter
echo 'do_read_seqcount_retry'     >> set_ftrace_filter
echo 'read_seqcount_begin'        >> set_ftrace_filter
echo 1 > tracing_on

# بعد تشغيل الـ workload
cat trace | head -50
echo 0 > tracing_on
```

لتتبع الـ latch mechanism:

```bash
echo 'raw_write_seqcount_latch'      > set_ftrace_filter
echo 'raw_read_seqcount_latch'      >> set_ftrace_filter
echo 'raw_read_seqcount_latch_retry' >> set_ftrace_filter
```

لو حابب تشوف **stack trace** وقت كل write:

```bash
echo 1 > options/func_stack_trace
echo function > current_tracer
echo 'do_raw_write_seqcount_begin' > set_ftrace_filter
echo 1 > tracing_on
```

---

#### 4. printk و dynamic debug

الـ seqlock نفسه مش بيعمل printk، لكن تقدر تفعّل dynamic debug للـ lockdep:

```bash
# تفعيل debug messages للـ lockdep (kernel compiled with CONFIG_DYNAMIC_DEBUG)
echo 'module lockdep +p' > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل كل debug في kernel/locking/
echo 'file kernel/locking/seqlock.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
```

لو حابب تضيف printk مؤقت في الكود (للـ development فقط):

```c
/* في write section — اطبع الـ sequence قبل وبعد */
pr_debug("seqcount before write: %u\n", s->sequence);
raw_write_seqcount_begin(s);
/* ... */
raw_write_seqcount_end(s);
pr_debug("seqcount after write:  %u\n", s->sequence);
```

تفعيل الـ `pr_debug` في runtime:

```bash
echo 'file my_driver.c +p' > /sys/kernel/debug/dynamic_debug/control
```

---

#### 5. Kernel config options للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_LOCK_ALLOC` | يضيف `dep_map` للـ `seqcount_t` ويفعّل lockdep tracking |
| `CONFIG_PROVE_LOCKING` | يفعّل lockdep كامل — يكشف deadlocks ومخالفات الـ locking order |
| `CONFIG_LOCK_STAT` | يجمع إحصائيات contention في `/proc/lock_stat` |
| `CONFIG_DEBUG_SPINLOCK` | يضيف checks إضافية على الـ spinlock المدمج في `seqlock_t` |
| `CONFIG_KCSAN` | الـ Kernel Concurrency Sanitizer — بيكشف data races مرتبطة بالـ seqlock |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكشف لو الكود نام وهو في write section |
| `CONFIG_PREEMPT_RT` | يغير behavior الـ `seqcount_LOCKNAME_t` — مهم تعرفه وقت الـ debugging |
| `CONFIG_LOCKDEP` | أساس الـ lockdep framework |

الـ config الأهم للـ seqlock debugging:

```bash
# تحقق إن الـ kernel compiled بيهم
grep -E 'CONFIG_(DEBUG_LOCK_ALLOC|PROVE_LOCKING|KCSAN|LOCK_STAT)' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ subsystem

**الـ KCSAN** (Kernel Concurrency Sanitizer) هو الأداة الأهم لـ seqlock debugging:

```bash
# تشغيل kernel مبني بـ CONFIG_KCSAN=y
# الـ KCSAN بيلاحظ تلقائياً الـ seqcount_t regions اللي محاطة بـ
# kcsan_atomic_next(KCSAN_SEQLOCK_REGION_MAX)

# بعد تشغيل test، ابحث في dmesg
dmesg | grep -i 'kcsan\|data.race'
```

مثال output لـ KCSAN لما يكشف race في seqlock:

```
==================================================================
BUG: KCSAN: data-race in update_timekeeping / read_time_fast
write to 0xffffffff82a1b4c8 of 8 bytes by task 123 on cpu 1:
 update_timekeeping+0x3c/0x180
 ...
read to 0xffffffff82a1b4c8 of 8 bytes by task 456 on cpu 0:
 read_time_fast+0x18/0x60
 ...
==================================================================
```

**الـ lockdep** بيكشف:
- استخدام `write_seqcount_begin()` من غير lock مناسب
- تداخل seqlock مع lock آخر بشكل خاطئ

```bash
dmesg | grep -A 30 'WARNING: suspicious RCU\|possible deadlock\|lock order'
```

---

#### 7. رسالة الـ error → المعنى → الحل

| رسالة في dmesg | المعنى | الحل |
|---------------|--------|------|
| `WARNING: inconsistent lock state` | الـ `write_seqcount_begin()` اتنادى من context مش متسق مع الـ lock | تأكد من استخدام `write_seqlock_irq()` أو `_bh()` حسب الـ context |
| `BUG: spinlock lockup suspected on CPU#N` | الـ reader بيعمل busy-wait بسبب writer مبقاش بيعمل `write_sequnlock()` | تأكد إن كل `write_seqlock()` ليه `write_sequnlock()` مقابل |
| `BUG: KCSAN: data-race in ...` | reader بيقرأ data خارج الـ seqcount_t critical section | حط القراءات جوا `read_seqcount_begin()` / `read_seqcount_retry()` |
| `WARNING: held lock freed!` | الـ seqlock_t اتحررت من الـ memory وهي محجوزة | تأكد من lifetime الـ seqlock_t بيتجاوز مدة الـ critical section |
| `lockdep_assert_held() failure` | `write_seqcount_begin()` اتنادى من غير ما تعمل lock على الـ associated lock أول | استخدم `seqcount_LOCKNAME_t` بدل `seqcount_t` لو الـ lock معروف |
| `possible deadlock detected` | seqlock اتاخد بعد lock تاني عكس الـ order المعتاد | ثبّت order الـ lock acquisition في كل الكود |
| `BUG: scheduling while atomic` | نوم جوا write section اللي بيعمل `preempt_disable()` | لا تستخدم any sleeping function جوا write section |
| `WARNING: CPU: N PID: M... in do_raw_write_seqcount_begin` | sequence counter فردي (odd) في بداية write — يعني write section اتفتح ومش اتقفل | تحقق من كل code paths إنها بتعمل `raw_write_seqcount_end()` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* 1. تحقق إن الـ sequence number حتى (even) قبل الـ write */
static inline void debug_assert_seqcount_even(const seqcount_t *s)
{
    WARN_ON_ONCE(s->sequence & 1); /* odd = write section لسه مفتوح */
}

/* 2. في بداية الـ read section — تحقق مش داخل interrupt handler غلط */
static inline void my_read_section(const seqcount_t *s, struct my_data *d)
{
    unsigned seq;
    int retries = 0;

    do {
        /* لو الـ retries كتير → writer ممكن stuck */
        if (unlikely(retries++ > 1000)) {
            WARN_ONCE(1, "seqcount retry storm: seq=%u\n", s->sequence);
            dump_stack();
            break;
        }
        seq = read_seqcount_begin(s);
        /* ... read data ... */
    } while (read_seqcount_retry(s, seq));
}

/* 3. في الـ write path — تحقق إن preemption disabled */
static inline void my_write_begin(seqcount_t *s)
{
    WARN_ON_ONCE(preemptible()); /* seqcount_t write section يجب يكون non-preemptible */
    raw_write_seqcount_begin(s);
}

/* 4. تحقق إن seqlock_t lock محجوز قبل write */
static inline void my_seqlock_write(seqlock_t *sl)
{
    /* lockdep بيتحقق تلقائياً — لكن ممكن تضيف assertion يدوي */
    lockdep_assert_held(&sl->lock);
}

/* 5. نقطة مهمة: بعد write_seqcount_invalidate — تحقق من الـ increment */
static void debug_seqcount_invalidate(seqcount_t *s)
{
    unsigned old = s->sequence;
    write_seqcount_invalidate(s);
    /* يجب يكون زاد بـ 2 */
    WARN_ON_ONCE(s->sequence != old + 2);
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ hardware state مطابق لـ kernel state

الـ seqlock هو pure software mechanism — مفيش hardware registers مباشرة. لكن:

- **الـ memory ordering** هو جوهر الموضوع — الـ `smp_wmb()` و `smp_rmb()` بيترجموا لـ memory barriers على الـ CPU.
- على **x86**: الـ `smp_wmb()` = `asm volatile("" ::: "memory")` (compiler barrier فقط — x86 بيعمل strong ordering)
- على **ARM/RISC-V**: الـ barriers = real hardware instructions مثل `dmb ish`

للتحقق إن الـ CPU بيطبق الـ ordering:

```bash
# اقرأ memory ordering capabilities للـ CPU
cat /proc/cpuinfo | grep -i 'memory\|barrier\|model name'

# على ARM — تحقق من الـ memory model
dmesg | grep -i 'memory model\|ARM'
```

---

#### 2. Register dump techniques

الـ seqlock مش بيتعامل مع hardware registers مباشرة. لكن لو بتـ debug subsystem (مثلاً timekeeping أو networking) بيستخدم seqlock:

```bash
# قراءة physical memory عبر devmem2 (لو عارف العنوان)
# مثلاً: قراءة jiffies_lock (seqlock في timekeeping)
# أول خطوة: اعرف العنوان من System.map
grep 'jiffies\|timekeeper' /proc/kallsyms | head -10

# استخدام devmem2 (لازم تثبته)
devmem2 0xffffffff82345678 w  # قراءة 4 bytes من العنوان

# بديل بـ /dev/mem (لازم CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0x82345678)) 2>/dev/null | xxd
```

**تحذير**: الـ addresses في الـ kernel virtual — استخدم `/proc/kcore` أو `crash` utility:

```bash
# مع crash utility
crash /proc/kcore /usr/lib/debug/vmlinux
# ثم:
crash> p jiffies_lock
crash> p &jiffies_lock.seqcount.seqcount.sequence
```

---

#### 3. Logic Analyzer و Oscilloscope

الـ seqlock مش بيولّد signals كهربائية مباشرة، لكن في subsystems بتستخدمه:

**Timekeeping + seqlock:**
- لو الـ system بيأخد latency غريبة في time reads → ممكن writer بيبلك لفترة طويلة
- قيس الـ latency بـ:

```bash
# قياس زمن read_seqbegin / read_seqretry loop
cyclictest -m -p 99 -t 4 -i 100 -l 10000
# لو latency أكبر من المتوقع → seqlock writer ممكن يكون السبب
```

**على hardware لـ embedded systems:**
- شيّل GPIO pin قبل `write_seqlock()` وبعد `write_sequnlock()` — اقيس على oscilloscope
- شيّل GPIO تاني في `read_seqbegin()` loop — عدد الـ pulses = عدد الـ retries

```c
/* في الـ write side — للـ oscilloscope debugging */
gpio_set_value(DEBUG_GPIO, 1);
write_seqlock(&my_lock);
/* ... critical section ... */
write_sequnlock(&my_lock);
gpio_set_value(DEBUG_GPIO, 0);
```

---

#### 4. Common hardware issues → kernel log patterns

| المشكلة الـ hardware | Pattern في kernel log | التفسير |
|---------------------|----------------------|---------|
| CPU cache coherency مكسور (buggy hardware) | `BUG: KCSAN: data-race` بشكل متكرر حتى مع لوكد صح | الـ cache مش بيشوف الـ writes من CPU تاني |
| NUMA node مع weak memory ordering | قراءات غلط من seqcount حتى مع even sequence | الـ `smp_load_acquire` مش كافي على الـ hardware ده |
| IRQ storm خلال write section | `BUG: spinlock lockup suspected` | الـ writer اتأخر بسبب high interrupt load |
| CPU frequency scaling خلال critical section | `sched: RT throttling activated` | الـ write section استغرق وقت طويل بسبب CPU throttling |
| Hyperthreading + seqlock contention | `NMI watchdog: BUG: soft lockup - CPU#N stuck` | الـ reader loop بيستهلك الـ CPU الـ hyperthreading sibling |

```bash
# ابحث عن هذه الـ patterns
dmesg | grep -E 'lockup|data-race|spinlock|throttling|KCSAN'
```

---

#### 5. Device Tree debugging

الـ seqlock مش جزء من Device Tree مباشرة. لكن لو الـ subsystem اللي بيستخدم seqlock (مثلاً clock driver أو pinctrl) فيه مشكلة DT:

```bash
# تحقق من الـ DT nodes المحمّلة
ls /proc/device-tree/
cat /proc/device-tree/clocks/compatible

# تحقق من الـ DT compiled version
dtc -I fs -O dts /proc/device-tree 2>/dev/null | grep -A5 'clock\|timer'

# قارن الـ DT مع hardware manual
# مثلاً: لو timer frequency غلط → writer بيعمل update بمعدل غلط
cat /proc/device-tree/timer*/clock-frequency
```

لو الـ timekeeping seqlock بيعمل retries كتير → تحقق من الـ timer hardware:

```bash
# تأكد الـ timer irq بيشتغل صح
cat /proc/interrupts | grep -i 'timer\|hrtimer'
# لو الـ count مش بيزيد → الـ timer hardware مش بيتكلم مع الـ kernel
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**1. فعّل lockdep وابحث عن مشاكل الـ seqlock:**

```bash
# تأكد الـ kernel compiled بـ CONFIG_PROVE_LOCKING
grep CONFIG_PROVE_LOCKING /boot/config-$(uname -r)

# شيّل lockdep stats قبل الـ test
cat /sys/kernel/debug/lockdep_stats > /tmp/lockdep_before.txt

# شغّل الـ workload هنا...

# قارن بعدين
cat /sys/kernel/debug/lockdep_stats > /tmp/lockdep_after.txt
diff /tmp/lockdep_before.txt /tmp/lockdep_after.txt
# لو ظهر 'chain entries' زاد كتير → lock ordering جديد اتشاف
```

**2. راقب seqlock write contention بـ perf:**

```bash
# track كل call لـ write_seqlock و write_sequnlock
perf probe -x /usr/lib/debug/vmlinux 'write_seqlock'
perf probe -x /usr/lib/debug/vmlinux 'write_sequnlock'

perf record -e 'probe:write_seqlock,probe:write_sequnlock' -a sleep 5
perf report
```

**3. قيس retry rate للـ seqlock readers:**

```bash
# استخدم ftrace لعدّ مرات do_read_seqcount_retry
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'do_read_seqcount_retry' > set_ftrace_filter
echo 'do___read_seqcount_retry' >> set_ftrace_filter
echo 1 > tracing_on
sleep 10
echo 0 > tracing_on

# عدّ مرات الـ retry
grep -c 'do_read_seqcount_retry\|do___read_seqcount_retry' trace
```

**4. اقرأ حالة seqcount مباشرة بـ crash/gdb:**

```bash
# مع crash utility على live kernel
crash /proc/kcore $(ls /usr/lib/debug/lib/modules/$(uname -r)/vmlinux 2>/dev/null || echo vmlinux)

# داخل crash:
crash> p timekeeper_seq         # seqcount_t للـ timekeeping
crash> p timekeeper_seq.sequence
crash> p jiffies_lock.seqcount.seqcount.sequence
```

**5. فعّل KCSAN لاكتشاف race conditions:**

```bash
# تحقق إن kernel compiled بـ KCSAN
grep CONFIG_KCSAN /boot/config-$(uname -r)

# لو نعم، الـ KCSAN بيعمل تلقائي — راقب dmesg
dmesg -w | grep -i 'kcsan\|data.race\|BUG'

# أو شغّل test مكثف ثم:
dmesg | grep -A 20 'BUG: KCSAN'
```

**6. اكشف write sections الطويلة بـ ftrace latency:**

```bash
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo wakeup_rt > current_tracer  # أو function_graph

# trace الـ write section كاملة
echo do_raw_write_seqcount_begin > set_graph_function
echo do_raw_write_seqcount_end  >> set_graph_function
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on

# ابحث عن write sections اللي استغرقت أكتر من 1ms
grep -E '\d{4,}\.' trace | head -20
```

**7. فحص seqlock في module معين:**

```bash
# ابحث عن الـ seqlock_t instances في الـ module
modinfo my_module | grep -i seq
# أو
nm /lib/modules/$(uname -r)/kernel/drivers/my/my_module.ko | grep -i seq

# تتبع كل الدوال في الـ module المرتبطة بالـ seqlock
cd /sys/kernel/debug/tracing
echo 'my_module:*' > set_ftrace_filter  # tracing كل الـ module
echo function > current_tracer
echo 1 > tracing_on
```

---

#### تفسير الـ output

**مثال: output لـ `/proc/lock_stat` لـ seqlock_t:**

```
                          class name    con-bounces    contentions   waittime-min   waittime-max waittime-total   waittime-avg    acq-bounces   acquisitions   holdtime-min   holdtime-max holdtime-total   holdtime-avg
----------------------------------------------------------------------------------------------------------------------------------------------------------
       &sl->lock:                 1             45          0.50 us        250.00 us       5623.45 us        125.00 us            120          8500          0.10 us         50.00 us       1200.00 us          0.14 us
```

التفسير:
- **`con-bounces = 1`**: مرة واحدة الـ lock اتأخد وهو محجوز من CPU تاني مع cache bounce — طبيعي
- **`contentions = 45`**: 45 مرة ال writer اضطر يستنى — لو كتير → writers متنافسين
- **`waittime-max = 250 us`**: أطول انتظار = 250 microsecond — لو أكبر من 1ms → مشكلة
- **`acquisitions = 8500`**: إجمالي مرات الـ write lock — قسّم على الـ contentions = 0.5% contention rate (مقبول)

**مثال: output لـ ftrace لكشف write/read pattern:**

```
  <...>-1234  [001] ....   123.456789: do_raw_write_seqcount_begin <-write_seqlock
  <...>-1234  [001] ....   123.456792: do_raw_write_seqcount_end   <-write_sequnlock
  <...>-5678  [000] ....   123.456800: do_read_seqcount_retry      <-read_time
  <...>-5678  [000] ....   123.456801: do___read_seqcount_retry     <-read_time
```

التفسير:
- الـ write section استغرق: `456792 - 456789 = 3 microseconds` — طبيعي
- الـ read retry جاء بعد `456800 - 456792 = 8 microseconds` — الـ reader بدأ بعد ما الـ writer خلص، مش المفروض يعمل retry
- لو شفت retry كتير مع write sections قصيرة → الـ reads هي اللي high frequency جداً

---

#### سيناريو debugging شامل: reader يعمل infinite loop

```bash
# 1. اكتشاف المشكلة من dmesg
dmesg | grep 'soft lockup\|hung_task'
# مثلاً:
# [ 1234.5] BUG: soft lockup - CPU#0 stuck for 22s! [my_thread:1234]

# 2. اعرف على أي CPU والـ stack
echo l > /proc/sysrq-trigger  # يطبع backtrace كل الـ CPUs
dmesg | tail -50
# هتشوف:
# CPU#0: my_thread
#  read_seqbegin+0x18/0x40
#  my_time_reader+0x3c/0x80
#  ...

# 3. تحقق من الـ writer على CPU تاني
dmesg | grep 'CPU#1'
# CPU#1: my_writer_task
#  do_raw_write_seqcount_begin+0x10/0x20
#  write_seqlock+0x1c/0x30
#  my_writer+0x44/0x90
#  ...

# 4. عرّف المشكلة: writer stack على CPU#1 بيشوف انه في write section
# 5. ابحث في الكود عن missing write_sequnlock()
grep -rn 'write_seqlock\|write_sequnlock' drivers/my/ | grep -v 'write_sequnlock'
# لو عدد write_seqlock أكبر من write_sequnlock → missing unlock في error path

# 6. إصلاح: تأكد من goto err path بيعمل write_sequnlock
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: RK3562 — Industrial Gateway — قراءة timestamp بتتأخر وبتعطي نتايج غريبة

#### العنوان
**الـ seqlock في driver الـ RTC بيرجع وقت قديم على industrial gateway مبني على RK3562**

#### السياق
شركة بتبني industrial gateway بيشتغل على RK3562 وبيستخدم RTC خارجي متصل بـ I2C. الـ driver بيحفظ الـ timestamp الحالي في struct محمية بـ `seqlock_t`. في كل لحظة بيجي interrupt من الـ RTC كل ثانية، الـ ISR بيعمل write على الـ struct دي. في نفس الوقت في userspace process بتقرأ الوقت بسرعة كبيرة عشان تعمل data logging.

#### المشكلة
الـ syslog بيظهر log entries بـ timestamp فيه jump للخلف — بيرجع ثانية أو اتنين فجأة. المشكلة مش دايمة، بتحصل بس لما الـ CPU load عالية.

#### التحليل
```c
/* الـ struct المحمية */
struct rtc_cache {
    seqlock_t   lock;
    time64_t    epoch_sec;
    u32         subsec_ns;
};

/* الـ ISR — write side */
static irqreturn_t rtc_irq_handler(int irq, void *dev_id)
{
    struct rtc_cache *cache = dev_id;

    /* WRONG: المفروض write_seqlock_irq مش write_seqlock */
    write_seqlock(&cache->lock);          /* BUG: لو الـ reader شغال في hardirq هيتعلق */
    cache->epoch_sec  = read_hw_rtc_sec();
    cache->subsec_ns  = read_hw_rtc_ns();
    write_sequnlock(&cache->lock);
    return IRQ_HANDLED;
}

/* الـ reader في kernel thread */
static void log_timestamp(struct rtc_cache *cache)
{
    unsigned seq;
    time64_t sec;
    u32      nsec;

    do {
        seq  = read_seqbegin(&cache->lock);
        sec  = cache->epoch_sec;
        nsec = cache->subsec_ns;
    } while (read_seqretry(&cache->lock, seq));
    /* استخدام sec و nsec هنا */
}
```

المشكلة الحقيقية: الـ engineer استخدم `write_seqlock()` داخل ISR. لما بيجي hardirq تاني أثناء الـ write critical section، الـ spinlock الداخلي في `seqlock_t` بيتعلق (spin). الأخطر من كده أن `sequence` بقى فردي (odd) ولقيناه بيفضل فردي لو الـ ISR اتقطع قبل ما يكمل. الـ reader في الـ loop بيكمل مستنياً لكن بيقرأ بيانات نص-محدثة.

في `__read_seqcount_begin()`:
```c
/* بيستنى لحد ما sequence يبقى زوجي */
while (unlikely((__seq = seqprop_sequence(s)) & 1))
    cpu_relax();
```
لو الـ ISR اتقطع في النص، `sequence` بيفضل فردي، الـ reader بيفضل في spin loop لفترة طويلة، وبعدين لما بيخرج بيقرأ بيانات inconsistent لأن `epoch_sec` و `subsec_ns` اتكتبوا في وقتين مختلفين.

#### الحل

```c
/* الـ ISR الصح */
static irqreturn_t rtc_irq_handler(int irq, void *dev_id)
{
    struct rtc_cache *cache = dev_id;
    unsigned long flags;

    /*
     * write_seqlock_irqsave: بتعمل spin_lock_irqsave داخلياً
     * بتضمن إن محدش يقدر يقاطعنا من hardirq تاني
     */
    write_seqlock_irqsave(&cache->lock, flags);
    cache->epoch_sec = read_hw_rtc_sec();
    cache->subsec_ns = read_hw_rtc_ns();
    write_sequnlock_irqrestore(&cache->lock, flags);

    return IRQ_HANDLED;
}
```

والتحقق بـ lockdep أثناء التطوير:
```bash
# تفعيل lockdep في kernel config
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_LOCKDEP=y

# مراقبة الـ output
dmesg | grep -E "WARNING|seqlock|possible circular"
```

#### الدرس المستفاد
لو الـ write side بتحصل في hardirq context، لازم تستخدم `write_seqlock_irqsave()` مش `write_seqlock()` العادي. الـ `seqlock_t` جواه `spinlock_t` وده بيعمل مشكلة لو IRQ جا في النص. القاعدة: **مستوى الـ disable يطابق أعلى مستوى هيقرأ أو يكتب**.

---

### السيناريو الثاني: STM32MP1 — IoT Sensor Node — CPU عالق في spin loop عمياء

#### العنوان
**الـ PREEMPT_RT kernel على STM32MP1 بيعمل latency spikes كبيرة بسبب استخدام `seqcount_t` غلط مع mutex**

#### السياق
شركة embedded بتبني IoT sensor node بـ STM32MP1 وشغال عليه `PREEMPT_RT` لضمان real-time response. الـ driver بيدير sensor data وعامل `seqcount_t` خام (plain) مش `seqcount_mutex_t`، والكتابة محمية بـ `mutex`.

#### المشكلة
الـ cyclictest بيكشف latency spikes تتجاوز 2ms رغم إن الـ target أقل من 100µs. المشكلة بتظهر بس لما الـ write thread شغال.

#### التحليل
```c
/* الكود الغلط */
struct sensor_state {
    seqcount_t   seq;   /* plain seqcount — مفيش lock مربوط */
    struct mutex lock;
    int32_t      temp_mdeg;
    int32_t      pressure_pa;
};

static void update_sensor(struct sensor_state *s, int32_t t, int32_t p)
{
    mutex_lock(&s->lock);

    /* WRONG: بيستخدم raw variant بدل write_seqcount_begin */
    raw_write_seqcount_begin(&s->seq);  /* مش بيعمل preempt_disable */
    s->temp_mdeg   = t;
    s->pressure_pa = p;
    raw_write_seqcount_end(&s->seq);

    mutex_unlock(&s->lock);
}

static void read_sensor(struct sensor_state *s, int32_t *t, int32_t *p)
{
    unsigned seq;
    do {
        seq = read_seqcount_begin(&s->seq);
        *t  = s->temp_mdeg;
        *p  = s->pressure_pa;
    } while (read_seqcount_retry(&s->seq, seq));
}
```

على `PREEMPT_RT`، الـ `mutex_lock()` بيبقى sleeping lock. الـ writer بيمسك الـ mutex، بيزود `sequence` لفردي، وبعدين **بيتعمل preempt** (لأن `raw_write_seqcount_begin` مش بيعمل `preempt_disable` لما مفيش `preemptible=true`). الـ reader RT thread بيشوف `sequence` فردي، بيدخل في spin loop من `__read_seqcount_begin`:

```c
/* ده بيحصل في read_seqcount_begin */
while (unlikely((__seq = seqprop_sequence(s)) & 1))
    cpu_relax();  /* busy-wait لحد ما الـ writer يكمل */
```

لو الـ writer اتعمله preempt بـ lower-priority task، الـ RT reader بيفضل spinning لحد ما الـ scheduler يرجع للـ writer — وده latency spike.

الحل الصح هو استخدام `seqcount_mutex_t`:
```c
/* على PREEMPT_RT، seqprop_sequence لـ mutex type بيعمل كده: */
if (preemptible && unlikely(seq & 1)) {
    mutex_lock(s->lock);    /* بيستنى الـ writer يخلص */
    mutex_unlock(s->lock);
    seq = smp_load_acquire(&s->seqcount.sequence); /* يقرأ تاني */
}
```
بدل spin، الـ reader بيعمل `mutex_lock/unlock` وده بيخليه يتعمله sleep بدل busy-wait.

#### الحل

```c
struct sensor_state {
    seqcount_mutex_t seq;   /* مربوط بالـ mutex */
    struct mutex     lock;
    int32_t          temp_mdeg;
    int32_t          pressure_pa;
};

static int sensor_init(struct sensor_state *s)
{
    mutex_init(&s->lock);
    /*
     * seqcount_mutex_init: بيربط الـ seqcount بالـ mutex
     * على PREEMPT_RT بيفعّل lock/unlock mechanism في read path
     */
    seqcount_mutex_init(&s->seq, &s->lock);
    return 0;
}

static void update_sensor(struct sensor_state *s, int32_t t, int32_t p)
{
    mutex_lock(&s->lock);
    write_seqcount_begin(&s->seq); /* بيتحقق lockdep + preemptibility */
    s->temp_mdeg   = t;
    s->pressure_pa = p;
    write_seqcount_end(&s->seq);
    mutex_unlock(&s->lock);
}
```

```bash
# قياس الـ latency بعد الإصلاح
cyclictest -p 90 -t 4 -n -i 200 -l 100000
# المتوقع: max latency < 100µs على STM32MP1 بـ PREEMPT_RT
```

#### الدرس المستفاد
على `PREEMPT_RT`، لازم تستخدم `seqcount_LOCKNAME_t` المناسب للـ lock المستخدم. استخدام `seqcount_t` الخام مع `mutex` بيكسر الـ RT semantics ويعمل busy-wait بدل sleep. الـ `SEQCOUNT_LOCKNAME` macro في `seqlock.h` موجود بالظبط عشان يحل المشكلة دي.

---

### السيناريو الثالث: i.MX8 — Android TV Box — تقطيع في صورة HDMI بسبب race في clock data

#### العنوان
**الـ display clock data على i.MX8 بيتقرأ inconsistent وبيعمل HDMI flicker على Android TV box**

#### السياق
منتج Android TV box مبني على i.MX8MQ. الـ HDMI driver بيحتاج يقرأ `pixel_clock` و `htotal` و `vtotal` من struct مشتركة بين الـ CRTC update thread والـ display ISR. الكود القديم استخدم spinlock بس فيه performance issue، فالـ engineer قرر يحول لـ seqlock لأن القراءات أكتر بكتير من الكتابة.

#### المشكلة
بعد التحويل ظهر flicker في الصورة وأحياناً scrambled frame. الـ bug مش reproducible بسهولة — بيحصل كل 30-40 دقيقة.

#### التحليل
الكود المحوّل:
```c
struct display_timings {
    seqlock_t lock;
    u32       pixel_clock_khz;
    u16       hdisplay, htotal, hsync_start, hsync_end;
    u16       vdisplay, vtotal, vsync_start, vsync_end;
    bool      interlaced;
};

/* الـ reader في display ISR — hardirq context */
static void hdmi_configure_phy(struct display_timings *dt)
{
    unsigned seq;
    u32 pclk;
    u16 ht, vt;

    do {
        /* WRONG: read_seqbegin مش safe في hardirq لو writer بيستخدم write_seqlock */
        seq  = read_seqbegin(&dt->lock);
        pclk = dt->pixel_clock_khz;
        ht   = dt->htotal;
        vt   = dt->vtotal;
    } while (read_seqretry(&dt->lock, seq));

    program_hdmi_phy(pclk, ht, vt);
}

/* الـ writer في process context */
static void update_timings(struct display_timings *dt, struct drm_display_mode *mode)
{
    /* write_seqlock: بيعمل spin_lock فقط — مش بيعمل irq disable */
    write_seqlock(&dt->lock);
    dt->pixel_clock_khz = mode->clock;
    dt->hdisplay        = mode->hdisplay;
    dt->htotal          = mode->htotal;
    /* ... باقي الـ fields */
    write_sequnlock(&dt->lock);
}
```

المشكلة: `write_seqlock()` بيعمل `spin_lock()` بس — مش `spin_lock_irq()`. لو الـ display ISR جا أثناء الـ write critical section:
1. الـ `sequence` فردي (writer فتح الـ section)
2. الـ ISR بيدخل `read_seqbegin` → بيشوف sequence فردي → بيعمل `cpu_relax()` loop
3. الـ writer على نفس الـ CPU محتاج يكمل بس **هو نفسه اللي اتعمله interrupt** → الـ CPU عالق!
4. النتيجة: deadlock أو watchdog reset أو قراءة inconsistent data

```
CPU0:
  write_seqlock()        → spin_lock(&dt->lock), sequence=1 (odd)
     [HDMI ISR fires]
       read_seqbegin()   → sees sequence=1, cpu_relax() loop...
       STUCK: waiting for writer to complete
       writer: waiting for ISR to return
  DEADLOCK / watchdog
```

#### الحل

```c
/* الـ writer: لازم يعمل irq disable */
static void update_timings(struct display_timings *dt, struct drm_display_mode *mode)
{
    unsigned long flags;

    /*
     * write_seqlock_irqsave: spin_lock_irqsave داخلياً
     * بيضمن إن الـ HDMI ISR مش هيقاطع الـ write section
     */
    write_seqlock_irqsave(&dt->lock, flags);
    dt->pixel_clock_khz = mode->clock;
    dt->hdisplay        = mode->hdisplay;
    dt->htotal          = mode->htotal;
    dt->vdisplay        = mode->vdisplay;
    dt->vtotal          = mode->vtotal;
    dt->hsync_start     = mode->hsync_start;
    dt->hsync_end       = mode->hsync_end;
    dt->vsync_start     = mode->vsync_start;
    dt->vsync_end       = mode->vsync_end;
    dt->interlaced      = !!(mode->flags & DRM_MODE_FLAG_INTERLACE);
    write_sequnlock_irqrestore(&dt->lock, flags);
}

/* الـ reader في ISR: مفيش تغيير — read_seqbegin safe في hardirq */
```

```bash
# تشخيص على الـ target
echo 1 > /proc/sys/kernel/hung_task_timeout_secs
# أو تفعيل softlockup detector
echo 1 > /proc/sys/kernel/softlockup_all_cpu_backtrace
dmesg | grep -i "watchdog\|hung\|lockup"
```

#### الدرس المستفاد
لو الـ reader ممكن يتشغل في hardirq، الـ writer **لازم** يعمل `irq_disable` أثناء الـ write section. استخدام `write_seqlock()` العادي في الحالة دي بيعمل potential deadlock على نفس الـ CPU. القاعدة الذهبية: **الـ disable level في الـ writer = أعلى context هيقرأ**.

---

### السيناريو الرابع: AM62x — Automotive ECU — بيانات CAN bus محتاجة latch mechanism لـ NMI handler

#### العنوان
**استخدام `seqcount_latch_t` على AM62x عشان NMI handler يقرأ CAN bus statistics من غير corruption**

#### السياق
automotive ECU مبني على AM62x بيشغّل Linux بـ NMI handler عشان يراقب critical error conditions. الـ NMI handler محتاج يقرأ CAN bus counters (tx_frames, rx_frames, error_count) اللي بيتحدث بكثرة من softirq context. المشكلة إن الـ NMI بيقاطع أي حاجة حتى critical sections، فالـ normal seqlock مش كافي.

#### المشكلة
الـ NMI handler أحياناً بيقرأ inconsistent data: `error_count` بيتجاوز `tx_frames + rx_frames` وده مستحيل منطقياً. النتيجة: false alarms وإيقاف الـ ECU غير ضروري.

#### التحليل
الـ `seqcount_latch_t` موجود بالظبط عشان الحالة دي. الفكرة: نحتفظ بـ نسختين من البيانات (data[0] و data[1])، والـ `sequence` LSB بيحدد أي نسخة الـ reader يقرأ منها:

```
sequence حتى (even) → reader يقرأ data[seq & 1] = data[0]
sequence فردي (odd) → reader يقرأ data[1]

Write sequence:
  write_seqcount_latch_begin → sequence++ (0→1), redirect readers to data[1]
  modify data[0]
  write_seqcount_latch       → sequence++ (1→2), redirect readers back to data[0]
  modify data[1]
  write_seqcount_latch_end   → done
```

في أي لحظة، دايماً في نسخة واحدة consistent جاهزة للـ NMI.

#### الحل

```c
struct can_stats_latch {
    seqcount_latch_t seq;
    struct {
        u64 tx_frames;
        u64 rx_frames;
        u32 error_count;
        u32 bus_off_count;
    } data[2];  /* نسختين */
};

static struct can_stats_latch can_stats;

/* الـ writer في softirq context */
static void can_update_stats(u64 tx, u64 rx, u32 errs)
{
    int idx;

    /*
     * write_seqcount_latch_begin:
     * sequence++ → فردي، readers بيتحولوا لـ data[1]
     */
    write_seqcount_latch_begin(&can_stats.seq);

    idx = 0; /* نعدّل data[0] */
    can_stats.data[idx].tx_frames   = tx;
    can_stats.data[idx].rx_frames   = rx;
    can_stats.data[idx].error_count = errs;

    /*
     * write_seqcount_latch:
     * sequence++ → زوجي، readers بيرجعوا لـ data[0]
     */
    write_seqcount_latch(&can_stats.seq);

    idx = 1; /* نعدّل data[1] */
    can_stats.data[idx].tx_frames   = tx;
    can_stats.data[idx].rx_frames   = rx;
    can_stats.data[idx].error_count = errs;

    /* write_seqcount_latch_end: بس بيعمل kcsan cleanup */
    write_seqcount_latch_end(&can_stats.seq);
}

/* الـ reader في NMI handler — مش ممكن يتعمله block أبداً */
static void nmi_check_can(void)
{
    unsigned seq, idx;
    u64 tx, rx;
    u32 errs;

    do {
        /*
         * read_seqcount_latch: بيقرأ sequence
         * LSB بيحدد النسخة الصحيحة
         */
        seq = read_seqcount_latch(&can_stats.seq);
        idx = seq & 0x1;

        tx   = can_stats.data[idx].tx_frames;
        rx   = can_stats.data[idx].rx_frames;
        errs = can_stats.data[idx].error_count;

        /*
         * read_seqcount_latch_retry: بيتأكد إن الـ sequence
         * ملقيش تغيير أثناء القراءة
         */
    } while (read_seqcount_latch_retry(&can_stats.seq, seq));

    /* لو errs > tx + rx → حالة طارئة حقيقية */
    if (errs > (u32)(tx + rx))
        trigger_safe_state();
}
```

```bash
# التحقق إن الـ latch بيشتغل صح
# نراقب /proc/interrupts للـ NMI count
watch -n 0.1 'grep NMI /proc/interrupts'

# نستخدم ftrace لتتبع الـ write/read sequence
echo 'can_update_stats' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
الـ `seqcount_latch_t` هو الأداة الصح لأي كود محتاج يتقرأ من NMI أو context مش قادر يعمل retry بسهولة. التكلفة: ضعف الـ memory للبيانات. الفايدة: **read path مضمون ما يتعلقش أبداً**.

---

### السيناريو الخامس: Allwinner H616 — Custom Board Bring-up — lockdep بيصرخ من `write_seqcount_begin` بدون lock

#### العنوان
**lockdep warning على Allwinner H616 أثناء bring-up لـ custom SPI display driver**

#### السياق
engineer بيعمل bring-up لـ custom board مبنية على Allwinner H616 مع SPI display. كتب driver جديد بيستخدم `seqcount_t` لحماية display state، وفعّل `CONFIG_DEBUG_LOCK_ALLOC` و `CONFIG_PROVE_LOCKING` في الـ kernel config عشان يتأكد إن الكود صح.

#### المشكلة
في أول boot بعد تفعيل lockdep، الكرنل بيطبع:
```
WARNING: suspicious RCU usage
...
seqcount: bad sequential counter sequence number
```
وأحياناً:
```
WARN: inconsistent lock state
```

#### التحليل
الكود المشبوه:
```c
struct spi_display_state {
    seqcount_t  seq;       /* plain seqcount — مفيش lock مربوط */
    spinlock_t  spi_lock;  /* للحماية من concurrent SPI access */
    u16         width, height;
    u32         bg_color;
    bool        enabled;
};

static void display_update(struct spi_display_state *s,
                           u16 w, u16 h, u32 color)
{
    /*
     * WRONG: write_seqcount_begin مع plain seqcount_t
     * بيعمل seqprop_assert → lockdep_assert_preemption_disabled()
     * لو مش عاملين preempt_disable قبل كده → WARNING
     */
    write_seqcount_begin(&s->seq);
    s->width    = w;
    s->height   = h;
    s->bg_color = color;
    write_seqcount_end(&s->seq);
}
```

الـ `write_seqcount_begin` بيعمل `seqprop_assert(s)`:
```c
/* لـ plain seqcount_t */
static inline void __seqprop_assert(const seqcount_t *s)
{
    lockdep_assert_preemption_disabled();
    /* لو الـ preemption مش disabled → lockdep warning */
}
```

الـ engineer نسي إن `seqcount_t` الخام محتاج الـ caller يضمن non-preemptibility بنفسه. الحل إما `preempt_disable()` قبل الكتابة، أو استخدام `seqcount_spinlock_t` عشان الـ preemption يتعمل automatically.

```c
/* التسلسل الصح للأدوات حسب الاحتياج */

/* Option 1: seqcount_t خام — الـ caller مسؤول */
preempt_disable();
raw_write_seqcount_begin(&s->seq);
/* write data */
raw_write_seqcount_end(&s->seq);
preempt_enable();

/* Option 2: seqcount_spinlock_t — الأفضل */
struct spi_display_state {
    seqcount_spinlock_t seq;  /* مربوط بالـ spinlock */
    spinlock_t          spi_lock;
    /* ... */
};

static void display_update(struct spi_display_state *s, ...)
{
    spin_lock(&s->spi_lock);
    /*
     * write_seqcount_begin:
     * - seqprop_assert: lockdep_assert_held(s->seq.lock) ✓
     * - seqprop_preemptible: على non-RT → false → مش محتاج preempt_disable إضافي
     */
    write_seqcount_begin(&s->seq);
    s->width    = w;
    s->height   = h;
    s->bg_color = color;
    write_seqcount_end(&s->seq);
    spin_unlock(&s->spi_lock);
}
```

الـ `seqcount_spinlock_init` بيربط الـ pointer:
```c
/* من seqcount_LOCKNAME_init macro */
seqcount_spinlock_init(&s->seq, &s->spi_lock);
/* __SEQ_LOCK(____s->lock = (_lock)) → s->seq.lock = &s->spi_lock */
```

ولما lockdep يجي يتحقق:
```c
/* من SEQCOUNT_LOCKNAME macro في seqlock.h */
static __always_inline void
__seqprop_spinlock_assert(const seqcount_spinlock_t *s)
{
    __SEQ_LOCK(lockdep_assert_held(s->lock)); /* بيتحقق إن spi_lock ممسوك */
}
```

```bash
# إخراج lockdep كامل أثناء bring-up
dmesg | grep -A 20 "WARNING\|BUG\|seqcount"

# لو عايز تـ disable lockdep مؤقتاً عشان تكمل bring-up
# (مش مستحسن للإنتاج)
echo 0 > /proc/sys/kernel/prove_locking

# الأفضل: اصلح الكود وسيب lockdep يراقب
```

#### الدرس المستفاد
`seqcount_t` الخام للـ use cases المتقدمة بس — المبتدئ يستخدم دايماً `seqcount_LOCKNAME_t` أو `seqlock_t`. الـ lockdep integration في `seqlock.h` موجودة عشان تكتشف الأخطاء دي في التطوير **مش في الإنتاج**. فعّل `CONFIG_PROVE_LOCKING` من أول يوم في الـ bring-up.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط |
|--------|--------|
| **Official kernel docs** — `Documentation/locking/seqlock.rst` | [docs.kernel.org/locking/seqlock.html](https://docs.kernel.org/locking/seqlock.html) |
| kernel.org raw RST | [kernel.org/doc/Documentation/locking/seqlock.rst](https://www.kernel.org/doc/Documentation/locking/seqlock.rst) |
| Source header — `include/linux/seqlock.h` | [github.com/torvalds/linux — seqlock.h](https://github.com/torvalds/linux/blob/master/include/linux/seqlock.h) |
| Source header — `include/linux/seqlock_types.h` | [github.com/torvalds/linux — seqlock_types.h](https://github.com/torvalds/linux/blob/master/include/linux/seqlock_types.h) |

---

### مقالات LWN.net

**الـ** LWN.net هو أهم مرجع لمتابعة تطور الـ kernel locking subsystem.

| المقال | الأهمية |
|--------|---------|
| [Driver porting: mutual exclusion with seqlocks](https://lwn.net/Articles/22818/) | أول تعريف رسمي للـ seqlock في kernel 2.5.60 — نقطة البداية التاريخية |
| [New version of frlock (now called seqlock)](https://lwn.net/Articles/21812/) | النسخة الأصلية قبل ما تتسمى seqlock — مهمة لفهم التصميم الأولي |
| [seqlock: Extend seqcount API with associated locks](https://lwn.net/Articles/822442/) | إضافة الـ `seqcount_LOCKNAME_t` types — تحول جوهري في الـ API سنة 2020 |
| [seqlock: Introduce seqcount_latch_t](https://lwn.net/Articles/829724/) | إدخال الـ `seqcount_latch_t` كـ type مستقل بدل الاستخدام اليدوي للـ latch pattern |
| [The seqcount latch lock type](https://lwn.net/Articles/831540/) | شرح تفصيلي لآلية الـ latch وإزاي بتُستخدم في الـ vsyscall و timekeeping |
| [seqlock: Introduce PREEMPT_RT support](https://lwn.net/Articles/830680/) | كيف تغيّر الـ seqlock تحت الـ PREEMPT_RT لتجنب priority inversion |
| [seqlock: serialize against writers](https://lwn.net/Articles/296209/) | مناقشة مشكلة الـ writer serialization وإزاي اتحلت |
| [seqlock: Add new blocking reader type & use rwlock](https://lwn.net/Articles/558102/) | إضافة الـ blocking reader variant (`read_seqlock_excl`) |

---

### مقالات KernelNewbies.org

**الـ** kernelnewbies.org بيوثّق التغييرات المهمة في كل إصدار:

| الصفحة | ما يخص الـ seqlock |
|--------|-------------------|
| [Linux 5.10 — KernelNewbies](https://kernelnewbies.org/Linux_5.10) | إدخال `seqcount_latch_t` وإعادة هيكلة الـ associated locks API |
| [Linux 5.9 — KernelNewbies](https://kernelnewbies.org/Linux_5.9) | preparatory cleanups للـ seqcount associated locks |
| [Linux 2.6.36 — KernelNewbies](https://kernelnewbies.org/Linux_2_6_36) | تغييرات مبكرة على الـ seqlock subsystem |

---

### Kernel Commits المهمة

| الـ Commit / الموضوع | الوصف |
|----------------------|-------|
| [seqlock: Simplify SEQCOUNT_LOCKNAME()](https://patchew.org/linux/169713575506.3135.7190411037772365188.tip-bot2@tip-bot2/) | تبسيط الـ macro infrastructure للـ associated lock types |
| [lore.kernel.org — seqlock associated locks series](https://lore.kernel.org/all/871rnbsu57.fsf@nanos.tec.linutronix.de/) | مناقشة Thomas Gleixner حول إعادة هيكلة الـ seqcount API |
| [torvalds/linux commits history](https://github.com/torvalds/linux/commits/master/include/linux/seqlock.h) | كامل تاريخ التعديلات على `seqlock.h` |

---

### Mailing List Discussions

| الرابط | الموضوع |
|--------|---------|
| [lore.kernel.org/lkml — seqlock](https://lore.kernel.org/lkml/?q=seqlock) | أرشيف كامل لمناقشات الـ LKML عن الـ seqlock |
| [LKML: nohz seqlock sync discussion](https://lkml.indiana.edu/hypermail/linux/kernel/1308.2/02161.html) | مثال على استخدام الـ seqlock في synchronizing sleep time stats |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 5 — *Concurrency and Race Conditions*
- **الموضوع**: Section "Seqlocks" — يشرح الـ seqlock كـ alternative لـ reader/writer locks
- **متاح مجاناً**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل المهم**: Chapter 10 — *Kernel Synchronization Methods*
- **الموضوع**: Section "Sequential Locks" — مقارنة مع الـ spinlocks والـ rwlocks، ومتى تستخدم كل واحد
- **الـ ISBN**: 978-0672329463

#### Understanding the Linux Kernel, 3rd Edition — Bovet & Cesati
- **الفصل المهم**: Chapter 5 — *Kernel Synchronization*
- **الموضوع**: شرح الـ seqlock في سياق الـ timekeeping وكيف اتبنى في الـ x86 vsyscall

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل المهم**: Chapter 16 — *Kernel Debugging Techniques*
- **الموضوع**: يذكر الـ seqlock في سياق الـ real-time systems وإزاي بيتأثر تحت الـ PREEMPT_RT

---

### مقالات إضافية مفيدة

| المصدر | الرابط |
|--------|--------|
| **SeqLock — Linux Inside** (0xax) | [0xax.gitbooks.io — linux-sync-6](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-6.html) |
| **Wikipedia — Seqlock** | [en.wikipedia.org/wiki/Seqlock](https://en.wikipedia.org/wiki/Seqlock) |
| **seqlock.h على Android kernel** | [android.googlesource.com — seqlock.h](https://android.googlesource.com/kernel/common/+/a7827a2a60218b25f222b54f77ed38f57aebe08b/include/linux/seqlock.h) |
| **Kernel docs (static mirror)** | [static.lwn.net/kerneldoc/locking/seqlock.html](https://static.lwn.net/kerneldoc/locking/seqlock.html) |

---

### Search Terms للبحث عن معلومات أكتر

لو عايز تعمق أكتر، استخدم الكلمات دي في الـ search engines والـ lore.kernel.org:

```
seqlock linux kernel
seqcount_t associated locks
seqcount_latch_t timekeeping
read_seqbegin read_seqretry
sequential lock lockless reader
seqlock PREEMPT_RT priority inversion
seqlock vsyscall vdso timekeeping
KCSAN seqlock data race
write_seqlock_irqsave
seqcount spinlock rwlock associated
```
## Phase 8: Writing simple module

### الـ Hook المختار: `do_raw_write_seqcount_begin`

الدالة دي هي نقطة الدخول الحقيقية لكل **write-side critical section** في الـ `seqcount_t` — بتزوّد الـ sequence بواحد (عشان يبقى فردي) وبتحط memory barrier. كل كود في الكيرنل بيكتب على بيانات محمية بـ seqlock بيمر من هنا، فهي نقطة مراقبة ممتازة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * seqcount_write_probe.c
 *
 * Attaches a kprobe to do_raw_write_seqcount_begin() and logs every time
 * the kernel enters a seqcount write-side critical section.
 *
 * Safe to use: do_raw_write_seqcount_begin is not called in NMI context,
 * and the kprobe handler itself does not acquire any seqlock.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit, pr_info  */
#include <linux/kprobes.h>     /* struct kprobe, register/unregister_kprobe  */
#include <linux/seqlock.h>     /* seqcount_t definition                      */
#include <linux/sched.h>       /* current, task_comm_len                     */
#include <linux/ptrace.h>      /* struct pt_regs                             */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Arabic Docs");
MODULE_DESCRIPTION("kprobe on do_raw_write_seqcount_begin to trace seqcount writers");

/* -----------------------------------------------------------------------
 * الـ kprobe struct
 * بنحدد اسم الدالة المستهدفة بالنص عشان الكيرنل يعرف يحل العنوان وحده
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "do_raw_write_seqcount_begin",
};

/* -----------------------------------------------------------------------
 * الـ pre-handler — بيتنفذ قبل أول تعليمة في الدالة المستهدفة
 *
 * @p:    pointer للـ kprobe نفسها (ممكن نقرأ منها .symbol_name)
 * @regs: حالة الـ CPU registers لحظة الاعتراض
 *        - على x86_64: rdi = أول argument = الـ seqcount_t*
 *        - بنقرأ منه الـ sequence الحالي قبل ما تتزوّد
 * ----------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs->di على x86_64 بيحمل أول argument بالـ calling convention
     * وهو الـ seqcount_t* اللي بتاخده do_raw_write_seqcount_begin
     */
    seqcount_t *s = (seqcount_t *)regs->di;

    /*
     * بنطبع: اسم الـ process الحالي، الـ PID، وقيمة الـ sequence counter
     * قبل الـ increment — لو فردية معناها الكيرنل اتداخل write مرتين بدون end
     * (bug detector طبيعي!)
     */
    pr_info("seqcount_write: comm=%-16s pid=%d seq_before=%u %s\n",
            current->comm,
            current->pid,
            s->sequence,
            (s->sequence & 1) ? "[WARN: already odd!]" : "");

    return 0; /* 0 = استمر تنفيذ الدالة الأصلية */
}

/* -----------------------------------------------------------------------
 * module_init — بيسجل الـ kprobe عند تحميل الـ module
 * لو فشل register_kprobe بيرجع error code سالب
 * ----------------------------------------------------------------------- */
static int __init seqcount_probe_init(void)
{
    int ret;

    kp.pre_handler = handler_pre;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("seqcount_write: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("seqcount_write: kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit — بيشيل الـ kprobe عند إزالة الـ module
 * لازم نعمل unregister قبل ما الـ handler code يتحذف من الذاكرة،
 * وإلا الكيرنل هيتصل بـ handler مش موجود وكراش مضمون
 * ----------------------------------------------------------------------- */
static void __exit seqcount_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("seqcount_write: kprobe removed from %s\n", kp.symbol_name);
}

module_init(seqcount_probe_init);
module_exit(seqcount_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/seqlock.h>
#include <linux/sched.h>
#include <linux/ptrace.h>
```

**الـ** `kprobes.h` بيجيب الـ API بتاع الـ kprobe نفسها، و`seqlock.h` عشان نعرف شكل الـ `seqcount_t` ونقرأ الـ `.sequence` field، و`sched.h` عشان نوصل للـ `current` macro اللي بيديك الـ `task_struct` للـ process الحالي، و`ptrace.h` عشان نفهم layout الـ `pt_regs` ونجيب الـ argument الأول من `regs->di`.

#### الـ `kp` struct

```c
static struct kprobe kp = {
    .symbol_name = "do_raw_write_seqcount_begin",
};
```

بنحدد الدالة بالاسم النصي — الكيرنل بيحل العنوان الحقيقي وقت الـ `register_kprobe` عن طريق الـ kallsyms. ده أفضل من تحديد عنوان ثابت لأن العنوان بيتغير مع كل build.

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ `pre_handler` بيتنفذ **قبل** تنفيذ الدالة المستهدفة، فبنشوف قيمة الـ `sequence` قبل ما تتزود. لو لقينا القيمة فردية بالفعل معناها في bug — write section فُتحت ولم تُغلق. الـ `regs->di` على x86_64 هو أول argument (System V AMD64 ABI).

#### الـ `module_init` و `module_exit`

الـ `register_kprobe` بتزرع الـ breakpoint في الذاكرة وتربط الـ handler بيه. الـ `unregister_kprobe` في الـ exit **ضرورية جداً**: لو الـ module اتحذف من الذاكرة والـ kprobe لسه مسجلة، أي استدعاء للدالة هيجمد عند الـ breakpoint ويحاول يكمّل لكود اتحذف، والنتيجة kernel panic مضمون.

---

### Makefile للبناء

```makefile
obj-m += seqcount_write_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod seqcount_write_probe.ko

# متابعة الـ log في real-time
sudo dmesg -w | grep seqcount_write

# إزالته
sudo rmmod seqcount_write_probe
```

**مثال على output متوقع:**

```
[  123.456789] seqcount_write: kprobe planted at do_raw_write_seqcount_begin (ffffffffa1b2c3d4)
[  123.460001] seqcount_write: comm=kworker/0:1     pid=45   seq_before=1234
[  123.460102] seqcount_write: comm=systemd         pid=1    seq_before=56
[  123.460201] seqcount_write: comm=kworker/u8:2    pid=89   seq_before=100
```

> **ملاحظة أمان:** الكود بيقرأ `s->sequence` داخل الـ kprobe handler بدون seqlock لأنه read-only diagnostic — race محتملة على القيمة المقروءة مقبولة في tracing context.
