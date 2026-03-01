## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ rwsem.h** جزء من subsystem اسمه **LOCKING PRIMITIVES** في الـ Linux kernel، بتتبعه Peter Zijlstra وIngo Molnar وWill Deacon. الكود الأساسي موجود في `kernel/locking/` والـ headers في `include/linux/`.

---

### المشكلة اللي بتحلها

تخيل عندك **مكتبة عامة** فيها كتاب واحد مهم جداً.

- **القراء** (readers): ممكن يقرؤا الكتاب مع بعض في نفس الوقت — مفيش مشكلة، الكتاب مش بيتغير.
- **الكاتب** (writer): لما حد يجي يعدّل في الكتاب، لازم **كل الناس تخرج**، ومينفعش حد يدخل تاني غيره.

ده بالظبط اللي بتحله الـ **rw_semaphore** — هو lock ذكي بيفرق بين القراءة والكتابة:

| الحالة | السلوك |
|---|---|
| قراءة فقط | أكتر من thread ممكن يقرأ في نفس الوقت |
| كتابة | thread واحد بس، ومحدش يقدر يقرأ أو يكتب في نفس الوقت |

لو استخدمت lock عادي (mutex) كان هيمنع القراءة المتزامنة وده خسارة في الـ performance. الـ rwsem بيحل الأمر بإنه يسمح بـ **shared read** و**exclusive write**.

---

### الفكرة من الـ ELI5

تخيل إن عندك **غرفة بباب**:

```
        [غرفة البيانات]
         ___________
        |           |
  قراء  |  قراء     |  قراء
  ↓     |  يدخلوا   |  ↓
  ←←←→  | بالآلاف  ←←←→
        |___________|

        لما جه الكاتب:
         ___________
        |   🔒      |
        |  LOCKED   |
        |  كاتب     |
        |  واحد بس  |
        |___________|
```

الـ `rw_semaphore` هو الـ **حارس** على الباب ده — بيتحكم في مين يدخل وإمتى.

---

### ليه ده مهم؟

في الـ kernel، في حاجات اتقرأ **ملايين المرات** في الثانية وبتتعدّل نادراً، زي:

- **vm_area_struct** (خريطة الـ virtual memory لكل process)
- **inode** (معلومات الملفات)
- **namespace** structures

لو استخدمت mutex عادي على الـ `mm->mmap_lock` مثلاً، كل thread كان هيوقف كل التانيين حتى للقراءة. الـ rwsem بيخلي الـ readers يشتغلوا بالتوازي وده **performance gain ضخم**.

---

### هيكل الـ `struct rw_semaphore`

```c
/* النسخة العادية (non-RT) */
context_lock_struct(rw_semaphore) {
    atomic_long_t count;    /* عداد يمثل حالة اللوك */
    atomic_long_t owner;    /* مين ماسك اللوك دلوقتي */
    struct optimistic_spin_queue osq; /* MCS queue للـ spinners */
    raw_spinlock_t wait_lock;         /* يحمي قايمة الانتظار */
    struct list_head wait_list;       /* قايمة المنتظرين */
};
```

الـ `count` field بيحكي القصة كلها:
- `0` = اللوك حر
- `bit 0` مضروب = في writer ماسك
- قيمة موجبة = عدد الـ readers الحاليين

---

### التصميم الداخلي: Optimistic Spinning

الـ rwsem مش بس lock عادي. فيه تكتيك اسمه **optimistic spinning**:

```
Thread جديد عايز lock:
        ↓
    هل اللوك حر؟
    /        \
  نعم        لأ
   ↓          ↓
  خد اللوك   هل الـ owner شغال على CPU دلوقتي؟
              /        \
           نعم          لأ
            ↓            ↓
        استنى spin    نام في wait_list
        (بدون sleep)
```

الفكرة: لو الـ owner مازال شغال على CPU، الأرجح هيخلص قريب — فأحسن أـ spin بدل ما تعمل sleep وتصحى تاني (ده expensive).

---

### نسختين: RT وغير RT

الـ header بيشيل **نسختين** مختلفتين حسب الـ kernel config:

#### **`CONFIG_PREEMPT_RT = n`** (النسخة العادية)
```c
struct rw_semaphore {
    atomic_long_t count;
    atomic_long_t owner;
    struct optimistic_spin_queue osq;
    raw_spinlock_t wait_lock;
    struct list_head wait_list;
};
```
بتستخدم atomic operations وspin قبل ما تنام.

#### **`CONFIG_PREEMPT_RT = y`** (نسخة Real-Time)
```c
struct rw_semaphore {
    struct rwbase_rt rwbase; /* بتحتوي على rt_mutex */
};
```
بتستخدم **rt_mutex** تحت الغطاء عشان تدعم **priority inheritance** — مهم جداً في الـ real-time systems عشان تتجنب **priority inversion**.

---

### الـ API العام

```c
/* قراءة */
down_read(sem);              /* lock للقراءة - بينام لو في writer */
down_read_trylock(sem);      /* حاول من غير ما تستنى */
down_read_interruptible(sem);/* ممكن يتقاطع بـ signal */
down_read_killable(sem);     /* بس SIGKILL يقدر يقاطعه */
up_read(sem);                /* حرر lock القراءة */

/* كتابة */
down_write(sem);             /* lock للكتابة - exclusive */
down_write_trylock(sem);     /* حاول من غير ما تستنى */
down_write_killable(sem);    /* ممكن يتقاطع بـ SIGKILL */
up_write(sem);               /* حرر lock الكتابة */

/* تحويل */
downgrade_write(sem);        /* حوّل write lock لـ read lock بدون release */
```

الـ `downgrade_write` ده حاجة ذكية — بيخليك تعمل تعديل وبعدين تفضل تقرأ بدون ما تحرر اللوك وتاخده تاني (أسرع وأأمن).

---

### الـ Scope Guards (C++ style في C)

الـ header بيوفر macros حديثة بتعمل **automatic unlock**:

```c
/* بدلاً من:
    down_read(&sem);
    ... do work ...
    up_read(&sem);           <- ممكن تنسى!
*/

/* استخدم:
    guard(rwsem_read)(&sem); /* هيعمل up_read تلقائياً لما يخرج من الـ scope */
```

ده الـ `DEFINE_LOCK_GUARD_1` macro اللي موجود في الـ header.

---

### الـ Lockdep Integration

لما بتفعّل `CONFIG_DEBUG_LOCK_ALLOC`، الـ kernel بيضيف `lockdep_map` جوا الـ struct. الـ **lockdep** subsystem بيراقب كل عملية lock/unlock وبيكشف:
- **Deadlocks** قبل ما يحصلوا
- **Lock order violations**
- **Recursive locking** اللي مش مسموح بيه في rwsem

---

### الملفات المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/rwsem.h` | **الملف ده** — الـ public API والـ struct definition |
| `kernel/locking/rwsem.c` | الـ implementation الكاملة — `down_read`, `down_write`, إلخ |
| `include/linux/rwbase_rt.h` | الـ RT base struct اللي بيستخدمه rwsem في الـ RT mode |
| `kernel/locking/rwbase_rt.c` | الـ RT implementation |
| `include/linux/osq_lock.h` | الـ MCS optimistic spin queue |
| `kernel/locking/osq_lock.c` | الـ OSQ implementation |
| `include/linux/percpu-rwsem.h` | نسخة الـ per-CPU من rwsem (للـ high-contention paths) |
| `kernel/locking/percpu-rwsem.c` | الـ per-CPU rwsem implementation |
| `include/linux/mutex.h` | الـ mutex العادي (للمقارنة) |
| `Documentation/locking/rwsem.rst` | التوثيق الرسمي |
| `include/linux/cleanup.h` | الـ DEFINE_LOCK_GUARD macros |

---

### أين بيتستخدم rwsem في الـ kernel؟

- **`mm->mmap_lock`** — أشهر rwsem في الـ kernel، بيحمي الـ virtual memory map لكل process
- **`inode->i_rwsem`** — بيحمي عمليات الـ inode (قراءة/كتابة الملفات)
- **`namespace` structures** — لحماية الـ mount points
- **`cred_guard_mutex`** — لحماية الـ credentials
## Phase 2: شرح الـ rwsem (Read-Write Semaphore) Framework

### المشكلة اللي بيحلها الـ rwsem

تخيل عندك shared data structure زي `struct inode` في الـ VFS — كتير جداً من الـ tasks بتقرأها في نفس الوقت، وبين الحين والتاني task واحد بيعدّل فيها (مثلاً تغيير الـ permissions أو الـ size). لو استخدمت **mutex** عادي، كل قارئ هيبلوك القارئ التاني — وده overhead مش محتاجه لأن القراءات المتعددة آمنة تماماً جنب بعضها.

المشكلة بالظبط:
- **Mutex**: exclusive بالكامل — قارئ واحد في وقت واحد رغم إن القراءات لا تتعارض.
- **Spinlock**: نفس المشكلة + مش ممكن تنام عليه.
- المحتاج: آلية تسمح بـ **concurrent reads** وتضمن **exclusive write**.

---

### الحل — الـ rwsem

الـ **`rw_semaphore`** (rwsem) هو sleeping lock بيقدم نوعين من الـ ownership:

| النوع | المتزامنين | يحجب مين؟ |
|-------|-----------|----------|
| **Read lock (shared)** | ∞ قارئين | Writers فقط |
| **Write lock (exclusive)** | واحد بس | كل القراءين والكتّابين |

لو ماحدش شايل write lock، أي عدد من الـ tasks ممكن يمسكوا read lock في نفس الوقت. لو جه writer، ينام لحد ما كل القراءين يخلصوا — وبعد كده ميجيش قارئ جديد لحد ما هو يخلص.

---

### Analogy عميق — مكتبة بقواعد صارمة

تخيل **مكتبة** فيها نسخة واحدة من كتاب مرجعي مهم جداً:

| عنصر المكتبة | مقابله في الـ rwsem |
|---|---|
| الكتاب نفسه | الـ shared data المحمية بالـ rwsem |
| قارئ بيشيل الكتاب ويقرأ | task شايل **read lock** |
| محرر (editor) عايز يعدّل | task شايل **write lock** |
| ورقة "الحجز" على الكتاب | الـ `count` field |
| قائمة انتظار الكاونتر | `wait_list` |
| موظف الكاونتر المنظّم | `wait_lock` (raw spinlock) |
| لوحة إعلان "الكتاب مع مين" | `owner` field |
| نظام الطابور المحسّن | `osq` (MCS optimistic spin queue) |

**القواعد:**
1. كتير من الناس ممكن يقروا الكتاب في نفس الوقت — المكتبة بتديهم كلهم نسخة (read lock مشترك).
2. لو حد عايز يعدّل — لازم يستنى لحد ما كل القراءين يرجعوا الكتاب.
3. لو القارئ بيتأخر وفيه editor في الطابور — المكتبة مش بتسمح بقراءين جدد (لمنع writer starvation).
4. بعد ما الـ editor يخلص — المكتبة بتفتح الباب للجميع.

**الفرق عن السطح**: في الـ rwsem، الـ `owner` field مش بس "مين شايل" — ده بُستخدم من الـ optimistic spinners عشان يعرفوا لو الـ owner شغّال على CPU دلوقتي فيستحق ينتظروا spinning بدل ما ينام. ده تمثيل للـ "موظف الكاونتر بيشوف لو الـ editor قاعد يشتغل فعلاً أو خرج يتغدى".

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Kernel Subsystems                          │
│                                                                 │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │   VFS   │  │   MM    │  │  procfs  │  │  device drivers  │  │
│  │ (inode/ │  │ (mmap/  │  │          │  │   (i2c, usb..)   │  │
│  │  dentry)│  │  mm_sem)│  │          │  │                  │  │
│  └────┬────┘  └────┬────┘  └────┬─────┘  └────────┬─────────┘  │
│       │            │            │                  │            │
│       └────────────┴────────────┴──────────────────┘            │
│                            │                                    │
│                            ▼                                    │
│            ┌───────────────────────────────┐                    │
│            │    rw_semaphore API Layer      │                    │
│            │                               │                    │
│            │  down_read()  / up_read()      │                    │
│            │  down_write() / up_write()     │                    │
│            │  down_read_trylock()           │                    │
│            │  down_write_trylock()          │                    │
│            │  downgrade_write()             │                    │
│            └───────────────┬───────────────┘                    │
│                            │                                    │
│           ┌────────────────┴─────────────────┐                  │
│           │                                  │                  │
│    ┌──────▼──────────────────┐   ┌───────────▼──────────────┐   │
│    │  Non-RT Path            │   │  PREEMPT_RT Path         │   │
│    │  (CONFIG_PREEMPT_RT=n)  │   │  (CONFIG_PREEMPT_RT=y)   │   │
│    │                         │   │                          │   │
│    │  struct rw_semaphore:   │   │  struct rw_semaphore:    │   │
│    │  ├─ atomic_long_t count │   │  └─ struct rwbase_rt:    │   │
│    │  ├─ atomic_long_t owner │   │     ├─ atomic_t readers  │   │
│    │  ├─ osq (MCS spinner)   │   │     └─ rt_mutex_base     │   │
│    │  ├─ raw_spinlock_t      │   │                          │   │
│    │  │  wait_lock           │   │  (priority-inheritance   │   │
│    │  └─ list_head wait_list │   │   aware, no spinning)    │   │
│    └─────────────────────────┘   └──────────────────────────┘   │
│                │                                                 │
│       ┌────────┴─────────┐                                       │
│       │                  │                                       │
│  ┌────▼────┐      ┌──────▼──────┐                                │
│  │scheduler│      │  lockdep    │                                │
│  │(sleep/  │      │  (deadlock  │                                │
│  │ wakeup) │      │  detection) │                                │
│  └─────────┘      └─────────────┘                                │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — count field وتمثيل الحالة

الفكرة المحورية في الـ Non-RT implementation هي استخدام **`atomic_long_t count`** واحدة لتمثيل كل حالات الـ lock:

```
count = 0                →  UNLOCKED (RWSEM_UNLOCKED_VALUE)
count = 0b...0001        →  WRITER LOCKED (bit 0 مضروب)
count = 0b...RRRR0       →  N readers (الـ upper bits بتعدّ القراءين)
```

**الـ bit layout** (مبسّط):
```
bit 0       = RWSEM_WRITER_LOCKED
bits 1..N   = reader count (كل reader يزوّد بـ increment محدد)
```

ده يخلي العمليات الأساسية **lock-free على الـ fast path** — بس CAS (Compare-And-Swap) على الـ `count` بدون حاجة للـ `wait_lock`.

الـ `wait_lock` (raw_spinlock) بتتقفل بس لما نضيف/نشيل entries من `wait_list` — أي في الـ slow path فقط.

---

### struct rw_semaphore بالتفصيل (Non-RT)

```c
context_lock_struct(rw_semaphore) {
    atomic_long_t count;   /* حالة الـ lock كلها في field واحدة */
    atomic_long_t owner;   /* مين شايله + flags */

#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq;  /* MCS queue للـ optimistic spinners */
#endif

    raw_spinlock_t wait_lock;    /* يحمي wait_list بس */
    struct list_head wait_list;  /* الـ tasks النايمة */

#ifdef CONFIG_DEBUG_RWSEMS
    void *magic;             /* للـ debugging */
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;  /* للـ lockdep */
#endif
};
```

**علاقة الـ fields ببعضها:**

```
┌─────────────────────────────────────────────────────┐
│              struct rw_semaphore                    │
│                                                     │
│  [count]──────────────────────────────────────────► │
│   atomic_long_t                                     │
│   "الحالة الكاملة" ← fast path بيشتغل عليه فقط    │
│                                                     │
│  [owner]──────────────────────────────────────────► │
│   atomic_long_t                                     │
│   = task_struct ptr | flags                         │
│   الـ optimistic spinners بيقروه عشان              │
│   يعرفوا لو الـ owner على CPU                       │
│                                                     │
│  [osq]────────────────────────────────────────────► │
│   optimistic_spin_queue { atomic_t tail }           │
│   MCS-like queue للـ spinners قبل ما يناموا         │
│                                                     │
│  [wait_lock]──────────────────────────────────────► │
│   raw_spinlock_t                                    │
│   يحمي [wait_list] بس — لا يُستخدم في fast path    │
│                                                     │
│  [wait_list]──────────────────────────────────────► │
│   list_head                                         │
│   ◄──► [waiter A] ◄──► [waiter B] ◄──► ...         │
│         (نايم)          (نايم)                      │
└─────────────────────────────────────────────────────┘
```

---

### الـ Three-Layer Waiting Strategy

الـ rwsem مش بتنام فوراً لما تلاقي contention — فيه 3 مراحل قبل النوم:

```
Task يطلب lock
      │
      ▼
┌─────────────────────────────────────────────┐
│  Layer 1: Fast Path (بدون أي lock)          │
│  CAS على count                              │
│  لو نجح → خذ الـ lock وامشي                │
└─────────────────┬───────────────────────────┘
                  │ فشل
                  ▼
┌─────────────────────────────────────────────┐
│  Layer 2: Optimistic Spinning (OSQ)         │
│  لو الـ owner شغّال على CPU حالياً          │
│  → spin في OSQ queue (MCS style)            │
│  بدون ما تنام → قليل latency               │
│  (CONFIG_RWSEM_SPIN_ON_OWNER)               │
└─────────────────┬───────────────────────────┘
                  │ الـ owner مش على CPU أو timeout
                  ▼
┌─────────────────────────────────────────────┐
│  Layer 3: Slow Path (نوم حقيقي)             │
│  خذ wait_lock                               │
│  أضف نفسك لـ wait_list                      │
│  نام (schedule())                           │
│  استنى لحد ما حد يصحّيك (wakeup)           │
└─────────────────────────────────────────────┘
```

**ليه المراحل دي مهمة؟**
- الـ **OSQ** (Optimistic Spin Queue) بيمنع الـ thundering herd لما كتير من الـ spinners يصحوا في نفس الوقت — كل واحد بيشوف اللي قبله في الـ MCS list بس، مش الـ lock نفسه.
- النوم الحقيقي بيتم بس لما يكون واضح إن الـ owner مش هيخلص قريب.

---

### struct rwbase_rt — الـ PREEMPT_RT Path

في الـ Real-Time kernel، الـ spinlock والـ optimistic spinning مش مقبولين لأنهم بيكسروا الـ bounded latency. الحل: استبدال كل الـ mechanism بـ **`rt_mutex`** اللي بيدعم **priority inheritance**.

```c
struct rwbase_rt {
    atomic_t        readers;  /* عداد القراءين + bias bits */
    struct rt_mutex_base rtmutex;  /* الـ mutex الأساسي */
};

#define READER_BIAS  (1U << 31)  /* initial value = no readers */
#define WRITER_BIAS  (1U << 30)  /* writer locked */
```

**المنطق:**
- `readers == READER_BIAS` → unlocked
- `readers == WRITER_BIAS` → write locked
- `readers > 0 && readers < READER_BIAS` → N readers active

الفرق الجوهري: في PREEMPT_RT، لو task بـ priority عالية استنى على rwsem، الـ holder بياخد الـ priority بتاعته مؤقتاً (priority inheritance عن طريق rt_mutex). ده بيضمن إن الـ high-priority task مش يستنى إلى الأبد.

---

### الـ owner Field — أكتر من مجرد pointer

```
atomic_long_t owner
```

الـ bits بتتوزع كده (64-bit):

```
bits 63..3  = task_struct pointer (aligned فبالكامل valid)
bit  2      = RWSEM_READER_OWNED   ← reader شايله مش writer
bit  1      = RWSEM_NONSPINNABLE   ← لا تعمل spin على اللي شايله
bit  0      = RWSEM_WRITER_LOCKED  ← نفس bit 0 في count
```

الـ optimistic spinner بيقرأ `owner` عشان يقرر: "هل الـ writer شايل الـ lock ده شغّال على CPU دلوقتي؟" — لو آه، يستمر يـ spin. لو لا، ينام فوراً. ده **speculative check** مش guaranteed.

---

### الـ downgrade_write() — من Write إلى Read

```c
extern void downgrade_write(struct rw_semaphore *sem)
    __releases(sem) __acquires_shared(sem);
```

دي operation مهمة جداً في الكود الحقيقي. مثال عملي:

```c
/* مثال: إنشاء inode جديد */
down_write(&inode->i_rwsem);

/* الجزء الحساس — تعديل */
inode->i_size = new_size;
inode->i_mtime = current_time(inode);

/* بعد التعديل، محتاج أقرأ بس */
downgrade_write(&inode->i_rwsem);  /* atomically: write→read */

/* دلوقتي قراءين تانيين ممكن يدخلوا */
do_something_read_only(inode);

up_read(&inode->i_rwsem);
```

`downgrade_write` بتحوّل الـ exclusive lock إلى shared بشكل atomic — من غير ما تعمل `up_write` و`down_read` منفصلين (اللي هيسيب gap يدخل فيه writer تاني).

---

### الـ Lock Guard API (cleanup.h integration)

الـ rwsem بيوفر macros للـ **scoped locking** باستخدام الـ `DEFINE_LOCK_GUARD_1` من `cleanup.h`:

```c
/* استخدام تقليدي — عرضة لنسيان up_read */
down_read(&my_sem);
/* ... */
up_read(&my_sem);

/* استخدام scoped — تلقائي */
scoped_guard(rwsem_read, &my_sem) {
    /* up_read بتتعمل تلقائياً عند نهاية الـ scope */
}

/* أو بالـ _try variant */
scoped_guard(rwsem_read_try, &my_sem) {
    /* بيدخل بس لو نجح trylock */
}
```

الـ variants المتاحة:

| Guard Name | Operation |
|---|---|
| `rwsem_read` | `down_read` / `up_read` |
| `rwsem_read_try` | `down_read_trylock` / `up_read` |
| `rwsem_read_intr` | `down_read_interruptible` / `up_read` |
| `rwsem_write` | `down_write` / `up_write` |
| `rwsem_write_try` | `down_write_trylock` / `up_write` |
| `rwsem_write_kill` | `down_write_killable` / `up_write` |
| `rwsem_init` | `init_rwsem` فقط (للـ dynamic init) |

---

### ما بيمتلكه الـ rwsem Subsystem vs ما بيفوّضه

**الـ rwsem بيمتلك:**
- تعريف الـ `struct rw_semaphore` وكل الـ fields
- الـ API الكامل: `down_read/write`, `up_read/write`, `trylock`, `killable`, `interruptible`
- منطق الـ three-layer waiting (fast path → OSQ → sleep)
- منطق الـ wakeup: مين يصحّى وبأي ترتيب
- الـ writer starvation prevention logic
- الـ `downgrade_write` operation
- الـ lock guard macros

**بيفوّض للـ subsystems التانية:**
- **Scheduler** (`include/linux/sched.h`): النوم الفعلي والـ wakeup — الـ rwsem بيطلب `schedule()` وبيستخدم `wake_up_process()`
- **Lockdep** (`include/linux/lockdep.h`): تتبع الـ lock ordering وكشف الـ deadlocks — الـ rwsem بيبلّغه بكل acquire/release
- **Atomic ops** (`include/linux/atomic.h`): الـ CAS operations على الـ `count` و `owner`
- **OSQ** (`include/linux/osq_lock.h`): الـ MCS spinning queue — الـ rwsem بيستخدمه as-is
- **RT mutex** (في PREEMPT_RT): كل الـ priority inheritance logic
- **Arch-specific**: الـ memory barriers المناسبة للـ platform

---

### استخدامات حقيقية في الـ Kernel

| الـ subsystem | الـ rwsem | السبب |
|---|---|---|
| VFS `inode->i_rwsem` | `down_write` عند truncate/create | تعديل الـ inode metadata |
| MM `mm->mmap_lock` | `down_read` عند page fault | القراءة من الـ VMA list |
| Namespaces | `down_write` عند clone | تعديل الـ namespace tree |
| `eBPF verifier` | `down_write` عند load | تحميل برنامج جديد |
| `cgroup` hierarchy | `down_read/write` | traversal vs modification |

مثال حقيقي من VFS:

```c
/* fs/inode.c */
void inode_lock(struct inode *inode)
{
    down_write(&inode->i_rwsem);
}

void inode_lock_shared(struct inode *inode)
{
    down_read(&inode->i_rwsem);
}
```

---

### الفرق بين rwsem والـ mutex

| الخاصية | mutex | rwsem |
|---|---|---|
| Concurrent reads | لا | نعم |
| Write exclusive | نعم | نعم |
| Sleeping lock | نعم | نعم |
| Priority inheritance (RT) | نعم (rt_mutex) | نعم (عبر rwbase_rt) |
| Optimistic spinning | نعم | نعم (OSQ) |
| Recursive locking | لا | لا |
| `downgrade` operation | لا | نعم (`downgrade_write`) |
| الاستخدام المثالي | exclusive access | read-mostly data |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Config Options والـ Flags

#### Config Options

| Config Option | التأثير |
|---|---|
| `CONFIG_PREEMPT_RT` | بيغير الـ implementation الكاملة — بيستخدم `rwbase_rt` بدل الـ atomic count |
| `CONFIG_RWSEM_SPIN_ON_OWNER` | بيضيف الـ `osq` (MCS lock) للـ optimistic spinning |
| `CONFIG_DEBUG_RWSEMS` | بيضيف الـ `magic` pointer للـ sanity checks |
| `CONFIG_DEBUG_LOCK_ALLOC` | بيضيف الـ `dep_map` للـ lockdep tracking |

#### Flags الـ `count` Field (non-RT)

| Flag / Value | القيمة | المعنى |
|---|---|---|
| `RWSEM_UNLOCKED_VALUE` | `0x0` | الـ lock فاضي خالص |
| `RWSEM_WRITER_LOCKED` | `bit[0] = 1` | في writer عنده الـ lock |
| Reader count | `bits[N..1]` | عدد الـ readers الحاليين مضروب في قيمة معينة |

#### Flags الـ `readers` Field (RT — `rwbase_rt`)

| Flag | القيمة | المعنى |
|---|---|---|
| `READER_BIAS` | `1U << 31` | القيمة الابتدائية — مفيش حد شايل اللوك |
| `WRITER_BIAS` | `1U << 30` | writer شايل اللوك حصرياً |
| `readers == READER_BIAS` | — | مفيش lock |
| `readers != READER_BIAS` | — | في lock (قراءة أو كتابة) |
| `readers == WRITER_BIAS` | — | write lock بالذات |
| `readers > 0` | — | في contention |

---

### الـ Structs المهمة

#### 1. `struct rw_semaphore` — الـ non-RT version

**الغرض:** الـ read/write semaphore الأساسي في الكيرنل — بيسمح لـ readers متعددين يدخلوا بالتوازي، وwriter واحد بس يدخل حصرياً.

```c
context_lock_struct(rw_semaphore) {
    atomic_long_t count;      /* عدد الـ readers / write lock flag */
    atomic_long_t owner;      /* task_struct* للـ current owner + flags */
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
    struct optimistic_spin_queue osq;  /* MCS queue للـ optimistic waiters */
#endif
    raw_spinlock_t wait_lock;     /* بيحمي الـ wait_list */
    struct list_head wait_list;   /* قائمة الـ tasks المنتظرة */
#ifdef CONFIG_DEBUG_RWSEMS
    void *magic;              /* pointer للـ lock نفسه للـ validation */
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;   /* lockdep metadata */
#endif
};
```

| Field | النوع | الوظيفة |
|---|---|---|
| `count` | `atomic_long_t` | الـ core state: 0=unlocked، bit0=writer، bits أعلى=reader count |
| `owner` | `atomic_long_t` | pointer للـ task_struct + status flags — بيُستخدم للـ optimistic spinning |
| `osq` | `optimistic_spin_queue` | MCS queue — الـ optimistic waiters بيـspin هنا قبل ما يروحوا sleeping |
| `wait_lock` | `raw_spinlock_t` | spinlock بيحمي الـ wait_list — non-sleepable |
| `wait_list` | `list_head` | الـ sleeping waiters مرتبين في queue |
| `magic` | `void *` | debug only — بيـcheck إن الـ struct لسه valid |
| `dep_map` | `lockdep_map` | lockdep tracking للـ lock ordering analysis |

---

#### 2. `struct rw_semaphore` — الـ RT version (`CONFIG_PREEMPT_RT`)

**الغرض:** نفس الـ API بس الـ implementation مختلفة تماماً — بتستخدم `rt_mutex` تحت من غير optimistic spinning عشان تدعم الـ priority inheritance.

```c
context_lock_struct(rw_semaphore) {
    struct rwbase_rt rwbase;      /* الـ RT-safe base implementation */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};
```

---

#### 3. `struct rwbase_rt`

**الغرض:** الـ RT base للـ rwsem — بيغلف `rt_mutex_base` ويحوّله لـ reader/writer lock.

```c
struct rwbase_rt {
    atomic_t         readers;    /* عدد الـ readers أو WRITER_BIAS */
    struct rt_mutex_base rtmutex; /* الـ mutex الأساسي */
};
```

| Field | الوظيفة |
|---|---|
| `readers` | initialized بـ `READER_BIAS` — بيتنقص لما reader يدخل، بيتحول لـ `WRITER_BIAS` لما writer يدخل |
| `rtmutex` | الـ RT mutex الحقيقي — بيدعم priority inheritance |

---

#### 4. `struct optimistic_spin_queue` (OSQ)

**الغرض:** MCS-like lock queue — الـ waiters بيـspin على الـ CPU بتاعهم بدل ما يـspin على nardo واحد مشترك — بيقلل الـ cache bouncing.

```c
struct optimistic_spin_queue {
    atomic_t tail;  /* CPU# للـ tail node في الـ queue */
};
```

الـ `tail == OSQ_UNLOCKED_VAL (0)` يعني الـ queue فاضية.

---

### رسم العلاقات بين الـ Structs

#### non-RT path

```
┌──────────────────────────────────────────────────┐
│              struct rw_semaphore                 │
│                                                  │
│  atomic_long_t  count  ──────── core state       │
│  atomic_long_t  owner  ──────── task_struct *    │
│                             +   flags            │
│                                    │             │
│                                    ▼             │
│                          ┌──────────────────┐   │
│                          │  struct          │   │
│                          │  task_struct     │   │
│                          │  (current owner) │   │
│                          └──────────────────┘   │
│                                                  │
│  struct optimistic_spin_queue  osq               │
│  ┌─────────────────────────────┐                 │
│  │  atomic_t tail              │                 │
│  │  (CPU# of tail spinner)     │                 │
│  └─────────────────────────────┘                 │
│                                                  │
│  raw_spinlock_t  wait_lock                       │
│                                                  │
│  struct list_head  wait_list ◄──────────────┐   │
│                                             │   │
└─────────────────────────────────────────────│───┘
                                              │
              ┌───────────────────────────────┘
              │
    ┌─────────┴──────────┐     ┌──────────────────┐
    │  rwsem_waiter[0]   │────►│  rwsem_waiter[1] │────► ...
    │  (list_head)       │     │  (list_head)      │
    │  task_struct *task │     │  task_struct *task│
    │  type: READER/     │     │  type: READER/    │
    │        WRITER      │     │        WRITER     │
    └────────────────────┘     └──────────────────┘
```

#### RT path

```
┌──────────────────────────────────┐
│       struct rw_semaphore (RT)   │
│                                  │
│  struct rwbase_rt  rwbase        │
│  ┌────────────────────────────┐  │
│  │  atomic_t readers          │  │
│  │  (READER_BIAS / count /    │  │
│  │   WRITER_BIAS)             │  │
│  │                            │  │
│  │  struct rt_mutex_base      │  │
│  │  rtmutex                   │  │
│  │  ┌──────────────────────┐  │  │
│  │  │  raw_spinlock_t lock │  │  │
│  │  │  struct rb_root_     │  │  │
│  │  │    cached waiters    │  │  │
│  │  │  struct task_struct  │  │  │
│  │  │    *owner            │  │  │
│  │  └──────────────────────┘  │  │
│  └────────────────────────────┘  │
│                                  │
│  struct lockdep_map  dep_map     │
└──────────────────────────────────┘
```

---

### دورة الحياة — Lifecycle Diagram

```
DECLARE_RWSEM(name)          ← static init (compile time)
        │
        │  أو
        │
init_rwsem(sem)              ← dynamic init (runtime)
  └─► __init_rwsem(sem, name, &key)
        │
        ├── count = 0 (UNLOCKED)
        ├── owner = 0
        ├── osq = OSQ_UNLOCKED
        ├── wait_lock = unlocked spinlock
        ├── wait_list = empty list
        └── dep_map initialized
        │
        ▼
   ┌─────────────────────────────────────────┐
   │            UNLOCKED STATE               │
   │          count == 0                     │
   └──────┬──────────────────────┬───────────┘
          │                      │
    down_read()            down_write()
          │                      │
          ▼                      ▼
   ┌─────────────┐        ┌─────────────────┐
   │   READ LOCK  │        │   WRITE LOCK    │
   │ count += bias│        │ count |= bit[0] │
   │ multiple OK  │        │ exclusive       │
   └──────┬───────┘        └────────┬────────┘
          │                         │
     up_read()               up_write()
          │                         │
          └──────────┬──────────────┘
                     ▼
              ┌─────────────────┐
              │  UNLOCKED STATE │
              │  wake waiters   │
              └─────────────────┘
                     │
              downgrade_write()  ← طريق تاني
              (write → read بدون unlock كامل)
```

---

### Call Flow Diagrams

#### Read Lock — Fast Path (no contention)

```
caller
  └─► down_read(sem)
        └─► rwsem_read_trylock(sem)
              └─► atomic_long_add_return(RWSEM_READER_BIAS, &sem->count)
                    │
                    ├── count has no WRITER flag? ──► lock acquired ✓
                    │
                    └── has WRITER flag?
                          └─► rwsem_read_failed()  [slow path]
```

#### Read Lock — Slow Path (contention)

```
caller
  └─► down_read(sem)
        └─► rwsem_down_read_slowpath(sem, state)
              │
              ├─► osq_lock(&sem->osq)   ← try optimistic spinning first
              │     └── spin on sem->owner while owner is on CPU
              │           ├── owner left CPU?  ──► osq_unlock(), go sleep
              │           └── lock freed?      ──► grab lock, return
              │
              └─► raw_spin_lock(&sem->wait_lock)
                    └─► list_add_tail(waiter, &sem->wait_list)
                          └─► schedule()     ← go to sleep
                                └─► [woken by up_read / up_write]
                                      └─► lock acquired ✓
```

#### Write Lock — Fast Path

```
caller
  └─► down_write(sem)
        └─► rwsem_write_trylock(sem)
              └─► atomic_long_cmpxchg(&sem->count,
                      RWSEM_UNLOCKED_VALUE,
                      RWSEM_WRITER_LOCKED)
                    │
                    ├── success? ──► set owner, return ✓
                    │
                    └── fail?   ──► rwsem_down_write_slowpath()
```

#### Write Lock — Slow Path

```
caller
  └─► rwsem_down_write_slowpath(sem, state)
        │
        ├─► osq_lock(&sem->osq)      ← optimistic spin
        │     └─► spin on sem->owner
        │           └── owner yielded CPU? ──► osq_unlock()
        │
        ├─► raw_spin_lock(&sem->wait_lock)
        │     └─► list_add_tail(waiter, &sem->wait_list)
        │
        └─► schedule()               ← sleep
              └─► [woken by up_read when last reader / up_write]
                    └─► set_owner, return ✓
```

#### Write Unlock — Wakeup Chain

```
caller
  └─► up_write(sem)
        ├─► atomic_long_andnot(RWSEM_WRITER_LOCKED, &sem->count)
        │     ← clears write lock bit
        │
        └─► rwsem_wake(sem)
              └─► raw_spin_lock(&sem->wait_lock)
                    └─► check wait_list head
                          ├── WRITER waiter? ──► wake single writer
                          └── READER waiters? ──► wake all consecutive readers
                                └─► wake_up_process(waiter->task)
```

#### downgrade_write — Write to Read

```
caller
  └─► downgrade_write(sem)
        ├─► atomic_long_add(RWSEM_READER_BIAS, &sem->count)
        │     ← يضيف reader count
        ├─► atomic_long_andnot(RWSEM_WRITER_LOCKED, &sem->count)
        │     ← يشيل write lock bit
        └─► rwsem_wake(sem)
              └─► بيصحى الـ readers المنتظرين (مش writers)
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `sem->count` | `atomic_long_t` | الـ core lock state — read/write via atomics |
| `sem->owner` | `atomic_long_t` | الـ owner pointer — written atomically |
| `sem->osq` | MCS queue | الـ optimistic spinners queue |
| `sem->wait_lock` | `raw_spinlock_t` | الـ `wait_list` — كل operations على القائمة |

#### قواعد الـ Locking

```
طبقات الـ locking من الأعلى للأسفل:

1. sem->count  (atomic — no explicit lock needed)
       │
       ▼
2. sem->osq    (MCS lock — بيتخد قبل ما نـspin على owner)
       │
       ▼
3. sem->wait_lock  (raw_spinlock — بيتخد لما نضيف/نشيل من wait_list)
       │
       ▼
4. schedule()  (نايم — مفيش lock مشغول)
```

#### مهم جداً

- **الـ `wait_lock` هو `raw_spinlock_t`** — مش `spinlock_t` — يعني مش بيتعمله preempt حتى في RT kernel، وده عشان الـ rwsem نفسه ممكن يتخد في contexts مختلفة.
- **الـ recursion ممنوع**: الـ rwsem مش reentrant — نفس الـ task لو حاول ياخده تاني هيـdeadlock.
- **الـ `sem->count` و `sem->owner` في نفس الـ cacheline** — design decision مقصود عشان الـ fast path يكون cache-friendly.
- **الـ `osq` بيتخد قبل الـ `wait_lock`** — الترتيب ده ثابت ولازم يتحرم عشان منعمل deadlock.
- **في الـ RT kernel**: مفيش optimistic spinning خالص — الـ `rt_mutex_base` جوا الـ `rwbase_rt` بيدي priority inheritance بدل كده.

#### ترتيب الـ Locking مع locks خارجية

```
القاعدة العامة في الكيرنل:
  spinlock / raw_spinlock
      ↓
  rwsem (write)
      ↓
  rwsem (read)
      ↓
  mutex
      ↓
  schedule / sleep

الـ rwsem مش بيتخد وانت شايل raw_spinlock
  إلا في الـ wait_lock نفسه (internal use).
```
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ APIs — Cheatsheet

#### Initialization & Declaration

| Function / Macro | الغرض |
|---|---|
| `DECLARE_RWSEM(name)` | Static declaration + initialization لـ rwsem |
| `init_rwsem(sem)` | Dynamic initialization لـ rwsem مع lockdep key |
| `__init_rwsem(sem, name, key)` | Core initializer (يتدعى من init_rwsem) |
| `__RWSEM_INITIALIZER(name)` | Compile-time initializer macro |

#### Locking — Read Side

| Function | Behavior | Return |
|---|---|---|
| `down_read(sem)` | Blocking، non-interruptible | void |
| `down_read_interruptible(sem)` | Blocking، قابل للـ signal | 0 أو `-EINTR` |
| `down_read_killable(sem)` | Blocking، يُوقفه SIGKILL بس | 0 أو `-EINTR` |
| `down_read_trylock(sem)` | Non-blocking attempt | 1 (success) أو 0 (fail) |
| `down_read_nested(sem, subclass)` | كـ `down_read` + lockdep subclass | void |
| `down_read_killable_nested(sem, subclass)` | كـ `down_read_killable` + lockdep subclass | 0 أو `-EINTR` |
| `down_read_non_owner(sem)` | Read lock بدون owner tracking | void |

#### Locking — Write Side

| Function | Behavior | Return |
|---|---|---|
| `down_write(sem)` | Blocking، non-interruptible | void |
| `down_write_killable(sem)` | Blocking، يُوقفه SIGKILL | 0 أو `-EINTR` |
| `down_write_trylock(sem)` | Non-blocking attempt | 1 (success) أو 0 (fail) |
| `down_write_nested(sem, subclass)` | كـ `down_write` + lockdep subclass | void |
| `down_write_killable_nested(sem, subclass)` | كـ `down_write_killable` + lockdep subclass | 0 أو `-EINTR` |
| `_down_write_nest_lock(sem, nest_lock)` | Write lock مع explicit nest lock | void |
| `down_write_nest_lock(sem, nest_lock)` | Wrapper macro للـ `_down_write_nest_lock` | void |

#### Unlocking

| Function | الغرض |
|---|---|
| `up_read(sem)` | Release read lock |
| `up_write(sem)` | Release write lock |
| `up_read_non_owner(sem)` | Release read lock من task مختلف عن اللي أخده |
| `downgrade_write(sem)` | تحويل write lock لـ read lock بدون release |

#### State Query

| Function | الغرض | Return |
|---|---|---|
| `rwsem_is_locked(sem)` | هل الـ sem محجوز؟ | bool |
| `rwsem_is_contended(sem)` | في tasks تنتظر؟ | bool |
| `rwsem_assert_held(sem)` | Assert + lockdep check | void |
| `rwsem_assert_held_write(sem)` | Assert write-held + lockdep | void |
| `rwsem_assert_held_nolockdep(sem)` | Assert بدون lockdep | void |
| `rwsem_assert_held_write_nolockdep(sem)` | Assert write بدون lockdep | void |
| `rwsem_owner(sem)` | رجّع الـ owner task_struct | `struct task_struct *` |
| `is_rwsem_reader_owned(sem)` | هل reader هو الـ owner؟ | bool |

#### RAII / Scoped Guards

| Guard Class | Lock | Unlock |
|---|---|---|
| `rwsem_read` | `down_read` | `up_read` |
| `rwsem_read_try` | `down_read_trylock` | `up_read` |
| `rwsem_read_intr` | `down_read_interruptible` | `up_read` |
| `rwsem_write` | `down_write` | `up_write` |
| `rwsem_write_try` | `down_write_trylock` | `up_write` |
| `rwsem_write_kill` | `down_write_killable` | `up_write` |
| `rwsem_init` | `init_rwsem` | — |

---

### Group 1: Initialization

هذه المجموعة مسؤولة عن إنشاء وتهيئة الـ `rw_semaphore`. اختلاف الـ implementation حسب الـ config: إما الـ classic implementation (بتستخدم `atomic_long_t count`) أو الـ RT variant (بتستخدم `rwbase_rt` فوق `rt_mutex`).

---

#### `DECLARE_RWSEM(name)`

```c
#define DECLARE_RWSEM(name) \
    struct rw_semaphore name = __RWSEM_INITIALIZER(name)
```

**الـ macro** بتعمل static allocation + initialization كلها في سطر واحد. مناسبة للـ global أو module-level rwsems. داخلياً بتمرر الاسم لـ `__RWSEM_INITIALIZER` اللي بيملّى كل الفيلدات.

- **Parameters:** `name` — اسم المتغير الجديد
- **Return:** لا يوجد (declaration statement)
- **Key details:** الـ lockdep key بيتسجّل automatically من خلال `__RWSEM_DEP_MAP_INIT`. الـ `magic` field (في `CONFIG_DEBUG_RWSEMS`) بتشاور على عنوان الـ struct نفسه كـ sanity check.

---

#### `init_rwsem(sem)`

```c
#define init_rwsem(sem)                        \
do {                                           \
    static struct lock_class_key __key;        \
    __init_rwsem((sem), #sem, &__key);         \
} while (0)
```

**الـ macro** للـ dynamic initialization — لما يكون الـ rwsem جزء من heap-allocated struct. البتستخدم `static struct lock_class_key` عشان الـ lockdep يعرف يميّز بين instances مختلفة من نفس الكود الفيزيائي.

- **Parameters:** `sem` — pointer لـ `struct rw_semaphore`
- **Key details:** الـ `__key` هي `static` عشان تعيش طول عمر الـ module. لو استخدمت `init_rwsem` في loop على structures مختلفة، كل الـ instances هتشارك نفس الـ lock class في lockdep.

---

#### `__init_rwsem(sem, name, key)`

```c
extern void __init_rwsem(struct rw_semaphore *sem, const char *name,
                         struct lock_class_key *key);
```

الـ core initializer. بتهيّي كل فيلدات الـ `rw_semaphore`: الـ `count` على `RWSEM_UNLOCKED_VALUE`، الـ `owner` على 0، الـ `wait_lock` كـ unlocked spinlock، الـ `wait_list` كـ empty list، وبتسجّل الـ lock مع lockdep.

- **Parameters:**
  - `sem` — الـ rwsem المطلوب تهيئته
  - `name` — string اسم الـ lock (للـ lockdep + debug)
  - `key` — lockdep class key
- **Return:** void
- **Key details:** Implementation موجودة في `kernel/locking/rwsem.c`. مش thread-safe في حد ذاتها — المفروض تتدعى قبل ما أي thread يقدر يوصل للـ sem.

---

### Group 2: Read-Side Locking

الـ read locks هي **shared** — متعدد readers يقدروا يمسكوا الـ lock في نفس الوقت، بس مش مع writer. كل دالة في المجموعة دي بتزود `count` بيـ `RWSEM_READER_BIAS` (أو مكافئها في RT).

---

#### `down_read(sem)`

```c
extern void down_read(struct rw_semaphore *sem) __acquires_shared(sem);
```

الأكثر استخداماً. بتمسك الـ read lock بشكل blocking كامل. لو في write lock active أو writer ينتظر، الـ task تنام في `wait_list` لحد ما يتحرر.

- **Parameters:** `sem` — الـ rwsem
- **Return:** void
- **Key details:**
  - مش interruptible — الـ task مش هتصحى لـ signals
  - في الـ non-RT implementation: بتحاول optimistic spinning على `osq` لو الـ writer لسه على CPU
  - في الـ RT variant: بتعمل `rt_mutex_lock` ضمنياً
- **Who calls it:** Any kernel code يحتاج يقرأ protected data (مثلاً VFS، mm subsystems)

**Pseudocode (non-RT):**
```
down_read(sem):
    if atomic_add_return(READER_BIAS, count) > 0:
        return  // fast path: no writer
    // slow path
    if can_spin_on_owner(sem):
        optimistic_spin()
        return if acquired
    // add to wait_list, sleep
    schedule()
```

---

#### `down_read_interruptible(sem)`

```c
extern int __must_check down_read_interruptible(struct rw_semaphore *sem)
    __cond_acquires_shared(0, sem);
```

زي `down_read` بالظبط لكن الـ task تقدر تُكسر من أي signal.

- **Parameters:** `sem` — الـ rwsem
- **Return:** `0` لو اتأخد الـ lock، `-EINTR` لو جه signal
- **Key details:** `__must_check` — لازم caller يتحقق من الـ return value. لو رجعت `-EINTR` معناه الـ lock **مش** مأخود — لا تقرأ الـ protected data.
- **Who calls it:** User-context code زي syscalls، vfs operations

---

#### `down_read_killable(sem)`

```c
extern int __must_check down_read_killable(struct rw_semaphore *sem)
    __cond_acquires_shared(0, sem);
```

الأكثر أماناً من `down_read_interruptible` — بس SIGKILL يقدر يكسرها، الـ signals العادية لأ.

- **Parameters:** `sem` — الـ rwsem
- **Return:** `0` نجح، `-EINTR` SIGKILL وصل
- **Key details:** مفضّلة في paths اللي محتاجة responsiveness ضد process termination بس مش محتاجة تتعامل مع كل signal.

---

#### `down_read_trylock(sem)`

```c
extern int down_read_trylock(struct rw_semaphore *sem)
    __cond_acquires_shared(true, sem);
```

Non-blocking. تحاول تمسك الـ read lock بدون ما تنام. بتنجح بس لو مفيش write lock active ومفيش pending writers (في بعض implementations).

- **Parameters:** `sem` — الـ rwsem
- **Return:** `1` نجح (lock مأخود)، `0` فشل (contention موجود)
- **Key details:** Safe لاستخدامها في atomic context أو interrupt context (بس rwsem عموماً sleeping lock — مش المفروض يتأخد في hardirq). مفيدة في polling loops أو speculative paths.

---

### Group 3: Write-Side Locking

الـ write locks هي **exclusive** — writer واحد بس يقدر يمسك الـ lock، وأثناء الـ write lock مفيش readers ولا writers تانيين.

---

#### `down_write(sem)`

```c
extern void down_write(struct rw_semaphore *sem) __acquires(sem);
```

بتمسك الـ write lock بشكل blocking كامل. بتـ set الـ `RWSEM_WRITER_LOCKED` bit في `count` وبتـ record الـ current task كـ owner في `sem->owner`.

- **Parameters:** `sem` — الـ rwsem
- **Return:** void
- **Key details:**
  - Writers بيـ set bit 0 (`RWSEM_WRITER_LOCKED`) في الـ `count`
  - الـ `owner` field بيتحدّث لـ `current` task_struct عشان optimistic spinners يقدروا يعرفوا لو الـ owner شغال على CPU
  - لو في readers active، الـ writer بيدخل `wait_list` وبيـ block الـ readers الجدد (writer-biased)
- **Who calls it:** VFS write paths، mmap operations، cgroup controllers

**Pseudocode:**
```
down_write(sem):
    tmp = atomic_long_cmpxchg(count, 0, WRITER_LOCKED)
    if tmp == 0:
        set_owner(current)
        return  // fast path
    // slow path
    if can_spin_on_owner(sem):
        optimistic_spin()  // spin on osq
        return if acquired
    // enqueue in wait_list as writer
    schedule()
```

---

#### `down_write_killable(sem)`

```c
extern int __must_check down_write_killable(struct rw_semaphore *sem)
    __cond_acquires(0, sem);
```

زي `down_write` بس يقدر يُكسر بـ SIGKILL.

- **Parameters:** `sem` — الـ rwsem
- **Return:** `0` نجح، `-EINTR` SIGKILL وصل
- **Key details:** لو فشلت، الـ lock مش مأخودة — lazم caller يتعامل مع الـ error path بشكل صحيح.

---

#### `down_write_trylock(sem)`

```c
extern int down_write_trylock(struct rw_semaphore *sem)
    __cond_acquires(true, sem);
```

Non-blocking write lock attempt. بتنجح بس لو الـ sem خالصاً تماماً (لا readers ولا writers).

- **Parameters:** `sem` — الـ rwsem
- **Return:** `1` نجح، `0` فشل
- **Key details:** Implementation بتعمل `cmpxchg` على `count` من 0 لـ `RWSEM_WRITER_LOCKED`. لو أي reader أو writer شغال، هترجع 0 على طول.

---

### Group 4: Unlock & Downgrade

---

#### `up_read(sem)`

```c
extern void up_read(struct rw_semaphore *sem) __releases_shared(sem);
```

بتحرر الـ read lock. بتـ decrement الـ `count` بـ `RWSEM_READER_BIAS`، ولو الـ count وصل 0 ومعناه في writers ينتظروا، بتـ wake up الـ first waiting writer.

- **Parameters:** `sem` — الـ rwsem
- **Return:** void
- **Key details:** لازم يتدعى من نفس الـ task اللي عمل `down_read`، ما عدا حالة `down_read_non_owner` / `up_read_non_owner`. عمل `up_read` أكتر من مرة على نفس الـ lock = undefined behavior + data corruption.

---

#### `up_write(sem)`

```c
extern void up_write(struct rw_semaphore *sem) __releases(sem);
```

بتحرر الـ write lock. بتـ clear الـ `RWSEM_WRITER_LOCKED` bit وبتـ clear الـ `owner` field، وبتـ wake up الـ waiters (writers أو readers حسب الترتيب).

- **Parameters:** `sem` — الـ rwsem
- **Return:** void
- **Key details:** الـ wake-up policy: بتفضّل تـ wake up writers على readers لتجنب writer starvation في بعض implementations. في الـ RT variant: بتـ release الـ underlying `rt_mutex`.

---

#### `downgrade_write(sem)`

```c
extern void downgrade_write(struct rw_semaphore *sem)
    __releases(sem) __acquires_shared(sem);
```

**Atomic downgrade** من write lock لـ read lock بدون release intermediate. الـ writer بيتحوّل لـ reader من غير ما أي writer تاني يقدر يقفز ويمسك الـ lock في النص.

- **Parameters:** `sem` — الـ rwsem
- **Return:** void
- **Key details:**
  - الـ annotations `__releases(sem) __acquires_shared(sem)` بتقول للـ lockdep إن الـ write lock اتحوّل لـ read
  - بتـ wake up الـ waiting readers لأن دلوقتي في reader active مش writer
  - Use case كلاسيكي: code يبدأ بـ write lock عشان يتحقق من الـ state، بعدين يتحول لـ read لو مش محتاج يكتب
- **Who calls it:** VFS code (مثلاً `i_rwsem` في inode locking)

---

### Group 5: State Query & Assertions

هذه المجموعة helpers للـ debugging والـ assertions. بعضها بيختفي في production builds وبعضها دايماً موجود.

---

#### `rwsem_is_locked(sem)`

```c
// Non-RT:
static inline int rwsem_is_locked(struct rw_semaphore *sem)
{
    return atomic_long_read(&sem->count) != RWSEM_UNLOCKED_VALUE;
}

// RT:
static __always_inline int rwsem_is_locked(const struct rw_semaphore *sem)
{
    return rw_base_is_locked(&sem->rwbase);
}
```

**Heuristic check** — مش guaranteed accurate في كل لحظة. بتقرأ الـ `count` بدون lock.

- **Parameters:** `sem` — الـ rwsem
- **Return:** non-zero لو محجوز، 0 لو حر
- **Key details:** مش safe للاستخدام كـ synchronization mechanism — بس للـ debugging والـ assertions.

**الـ Non-RT logic:**
- `count == 0` → unlocked
- `count & RWSEM_WRITER_LOCKED` → write locked
- `count > 0` → one or more readers

**الـ RT logic (عبر `rw_base_is_locked`):**
```c
return atomic_read(&rwb->readers) != READER_BIAS;
// READER_BIAS = (1U << 31) — القيمة الأساسية لما مفيش readers
```

---

#### `rwsem_is_contended(sem)`

```c
// Non-RT:
static inline int rwsem_is_contended(struct rw_semaphore *sem)
{
    return !list_empty(&sem->wait_list);
}

// RT:
static __always_inline int rwsem_is_contended(struct rw_semaphore *sem)
{
    return rw_base_is_contended(&sem->rwbase);
}
```

بتتحقق لو في tasks تنتظر في الـ `wait_list`. الـ comment في الكود بيوضح إنها **heuristic** — المفروض يتدعى من حد شايل الـ lock عشان يقرر مثلاً لو يحتاج يعمل `downgrade_write`.

- **Parameters:** `sem` — الـ rwsem
- **Return:** non-zero لو في waiters، 0 لو لأ
- **RT detail:** الـ `rw_base_is_contended` بتشوف `readers > 0` — في الـ RT impl الـ readers counter بيبدأ من `READER_BIAS` وبيتـ decrement لما writers ينتظروا.

---

#### `rwsem_assert_held(sem)`

```c
static inline void rwsem_assert_held(const struct rw_semaphore *sem)
    __assumes_ctx_lock(sem)
{
    if (IS_ENABLED(CONFIG_LOCKDEP))
        lockdep_assert_held(sem);
    else
        rwsem_assert_held_nolockdep(sem);
}
```

Assert إن الـ sem محجوز (read أو write) من الـ calling context. لو `CONFIG_LOCKDEP` مفعّل بيستخدم الـ lockdep للـ full validation، وإلا بيرجع للـ atomic read check.

- **Parameters:** `sem` — الـ rwsem
- **Return:** void (بس `WARN_ON` لو assertion فشلت)
- **Key details:** `__assumes_ctx_lock(sem)` annotation بتقول للـ compiler analyzer إن الـ lock مأخوذ عند هذه النقطة.

---

#### `rwsem_assert_held_write(sem)`

```c
static inline void rwsem_assert_held_write(const struct rw_semaphore *sem)
    __assumes_ctx_lock(sem)
{
    if (IS_ENABLED(CONFIG_LOCKDEP))
        lockdep_assert_held_write(sem);
    else
        rwsem_assert_held_write_nolockdep(sem);
}
```

Assert إن الـ **write** lock تحديداً هو المأخوذ.

- **Parameters:** `sem` — الـ rwsem
- **Return:** void
- **Key details الـ Non-RT:** بتتحقق إن `count & RWSEM_WRITER_LOCKED` != 0. الـ RT: بتتحقق عبر `rw_base_is_write_locked` إن `readers == WRITER_BIAS`.

---

#### `rwsem_assert_held_nolockdep(sem)`

```c
// Non-RT:
static inline void rwsem_assert_held_nolockdep(const struct rw_semaphore *sem)
    __assumes_ctx_lock(sem)
{
    WARN_ON(atomic_long_read(&sem->count) == RWSEM_UNLOCKED_VALUE);
}
```

النسخة الأخف — بدون lockdep overhead. بتطلع `WARN_ON` لو الـ sem خالي تماماً.

---

#### `rwsem_assert_held_write_nolockdep(sem)`

```c
// Non-RT:
static inline void rwsem_assert_held_write_nolockdep(const struct rw_semaphore *sem)
    __assumes_ctx_lock(sem)
{
    WARN_ON(!(atomic_long_read(&sem->count) & RWSEM_WRITER_LOCKED));
}
```

بتتحقق إن الـ bit 0 (`RWSEM_WRITER_LOCKED`) مـ set — يعني write lock active.

---

#### `rwsem_owner(sem)` و `is_rwsem_reader_owned(sem)`

```c
extern struct task_struct *rwsem_owner(struct rw_semaphore *sem);
extern bool is_rwsem_reader_owned(struct rw_semaphore *sem);
```

Available بس لما `CONFIG_DEBUG_RWSEMS` أو `CONFIG_DETECT_HUNG_TASK_BLOCKER` مفعّل.

- **`rwsem_owner`:** بتقشّر الـ flags من `sem->owner` وبترجع الـ raw `task_struct *`
- **`is_rwsem_reader_owned`:** بتتحقق من الـ `RWSEM_READER_OWNED` flag في الـ `owner` field
- **Key details:** الـ `owner` field في الـ non-RT implementation بيحمل pointer + flags في نفس الـ `atomic_long_t` لأن الـ task_struct aligned. الـ flags في الـ low bits.

---

### Group 6: Lockdep Nested Locking

هذه المجموعة موجودة بس لو `CONFIG_DEBUG_LOCK_ALLOC` مفعّل، وإلا بتتحوّل لـ macros بتدعي النسخ العادية.

الغرض: الـ lockdep بيرفض إنك تمسك نفس الـ lock مرتين (recursion detection). لكن أحياناً عندك **multiple instances** من نفس الـ lock class (مثلاً: inode locks في tree hierarchy) بترتيب محدد. الـ `_nested` APIs بتقول للـ lockdep "ده نفس الـ class بس subclass مختلف".

---

#### `down_read_nested(sem, subclass)`

```c
extern void down_read_nested(struct rw_semaphore *sem, int subclass)
    __acquires_shared(sem);
```

- **Parameters:**
  - `sem` — الـ rwsem
  - `subclass` — رقم الـ nesting level (عادة enum values زي `I_MUTEX_PARENT`, `I_MUTEX_CHILD`)
- **Return:** void
- **Use case:** VFS directory locking — parent dir lock (subclass 0) ثم child dir lock (subclass 1)

---

#### `down_write_nested(sem, subclass)`

```c
extern void down_write_nested(struct rw_semaphore *sem, int subclass)
    __acquires(sem);
```

نفس الفكرة لكن للـ write side.

---

#### `down_write_killable_nested(sem, subclass)`

```c
extern int down_write_killable_nested(struct rw_semaphore *sem, int subclass)
    __cond_acquires(0, sem);
```

- **Return:** `0` نجح، `-EINTR` SIGKILL

---

#### `_down_write_nest_lock(sem, nest_lock)` و `down_write_nest_lock(sem, nest_lock)`

```c
extern void _down_write_nest_lock(struct rw_semaphore *sem,
                                  struct lockdep_map *nest_lock)
    __acquires(sem);

#define down_write_nest_lock(sem, nest_lock)            \
do {                                                    \
    typecheck(struct lockdep_map *, &(nest_lock)->dep_map); \
    _down_write_nest_lock(sem, &(nest_lock)->dep_map);  \
} while (0)
```

بدل الـ subclass العددي، بتقول للـ lockdep "الـ sem ده nested تحت الـ nest_lock ده". الـ `typecheck` macro بيتحقق compile-time إن الـ nest_lock نوعه صح.

- **Parameters:**
  - `sem` — الـ rwsem المطلوب أخده
  - `nest_lock` — أي lock object عنده `dep_map` field (أي lock kernel عادي)
- **Key details:** المستخدمين مختلفون: خارجياً الـ macro، داخلياً الـ function الحقيقية. الـ comment في الكود بيقول صراحة إن الـ completions هي الـ abstraction الصح لهذه الحالة.

---

#### `down_read_non_owner(sem)` و `up_read_non_owner(sem)`

```c
extern void down_read_non_owner(struct rw_semaphore *sem)
    __acquires_shared(sem);
extern void up_read_non_owner(struct rw_semaphore *sem)
    __releases_shared(sem);
```

للحالات النادرة جداً اللي فيها task A بتمسك الـ read lock وتعدّي الـ responsibility لـ task B تعمل الـ release. الـ lockdep بيـ disable الـ owner tracking في الحالة دي.

- **Key details:** الـ comment في الكود صريح: "this API should be avoided — use completions". موجودة بس لـ `CONFIG_DEBUG_LOCK_ALLOC` أو بتتحوّل لـ `down_read` / `up_read` عادية.

---

### Group 7: RAII Scoped Guards

الـ rwsem.h بيعرّف كامل family من الـ scoped locking guards باستخدام الـ `cleanup.h` infrastructure. الفكرة: الـ C compiler بيدعي destructor تلقائياً لما الـ variable تخرج من الـ scope.

---

#### Read Guards

```c
DEFINE_LOCK_GUARD_1(rwsem_read, struct rw_semaphore,
    down_read(_T->lock), up_read(_T->lock))

DEFINE_LOCK_GUARD_1_COND(rwsem_read, _try,
    down_read_trylock(_T->lock))

DEFINE_LOCK_GUARD_1_COND(rwsem_read, _intr,
    down_read_interruptible(_T->lock), _RET == 0)
```

**الاستخدام:**
```c
// Unconditional:
scoped_guard(rwsem_read, &my_sem) {
    /* read protected data here */
} // up_read() called automatically

// Trylock:
CLASS(rwsem_read_try, g)(&my_sem);
if (!class_rwsem_read_try_lock_ptr(&g))
    return -EBUSY;
// lock held until g goes out of scope

// Interruptible:
ACQUIRE(rwsem_read_intr, lock)(&my_sem);
if (ACQUIRE_ERR(rwsem_read_intr, &lock))
    return -EINTR;
```

---

#### Write Guards

```c
DEFINE_LOCK_GUARD_1(rwsem_write, struct rw_semaphore,
    down_write(_T->lock), up_write(_T->lock))

DEFINE_LOCK_GUARD_1_COND(rwsem_write, _try,
    down_write_trylock(_T->lock))

DEFINE_LOCK_GUARD_1_COND(rwsem_write, _kill,
    down_write_killable(_T->lock), _RET == 0)
```

نفس المنطق بالضبط للـ write side.

---

#### `rwsem_init` Guard

```c
DEFINE_LOCK_GUARD_1(rwsem_init, struct rw_semaphore,
    init_rwsem(_T->lock), /* */)
```

Guard خاص بالـ initialization — بيدعي `init_rwsem` لما يتنشأ. الـ destructor فاضي (no unlock). مفيد لضمان إن الـ rwsem اتهيّى عند استخدام الـ scope-based patterns.

---

### ملاحظات عامة على الـ Implementation

#### الفرق بين Non-RT وRT

| الجانب | Non-RT | RT (PREEMPT_RT) |
|---|---|---|
| الـ struct | `atomic_long_t count` + `wait_lock` | `struct rwbase_rt` فوق `rt_mutex` |
| الـ Locking | Optimistic spinning + sleep | كل شئ عبر `rt_mutex` |
| Priority inheritance | مش موجود | موجود عبر `rt_mutex` |
| الـ `count` encoding | Bit flags + reader count | `atomic_t readers` مع `READER_BIAS` |

#### الـ `count` Field Encoding (Non-RT)

```
Bit 0 (RWSEM_WRITER_LOCKED): writer active
Bits 1+: reader count (encoded as RWSEM_READER_BIAS increments)
Value 0 (RWSEM_UNLOCKED_VALUE): completely free
```

#### الـ `owner` Field Encoding (Non-RT)

الـ `owner` field بيحمل `task_struct *` في الـ high bits والـ flags في الـ low bits:
- `RWSEM_READER_OWNED` — reader يمسك الـ lock
- `RWSEM_NONSPINNABLE` — optimistic spinning disabled
- Writer ownership: الـ pointer الحقيقي للـ task بدون flags
## Phase 5: دليل الـ Debugging الشامل

---

### على مستوى الـ Software

---

#### 1. مدخلات الـ debugfs

**الـ rwsem** مش ليه مدخل debugfs خاص بيه، لكن الـ **lockdep** بيعرض معلوماته من خلال:

```bash
# لو CONFIG_DEBUG_LOCK_ALLOC=y
ls /sys/kernel/debug/locking/

# اقرا كل الـ lock classes المسجلة
cat /proc/lockdep

# اقرا تاريخ اكتساب الـ locks
cat /proc/lockdep_stats

# اقرا الـ lock chains
cat /proc/lockdep_chains
```

**الـ `/proc/lockdep_stats`** بيطلع output زي ده:

```
lock-classes:                        1243 [max: 8192]
direct dependencies:                 3892 [max: 421888]
indirect dependencies:               28461
all direct dependencies:             47231
dependency chains:                    821 [max: 65536]
in-hardirq chains:                     0
in-softirq chains:                    87
in-process chains:                   734
...
```

**الـ wait_list** — مفيش debugfs مباشر، بس تقدر تشوفه عن طريق crash/gdb أو كما يلي.

---

#### 2. مدخلات الـ sysfs

**الـ rwsem** نفسه مش ليه sysfs entries، لكن العمليات المتعلقة بيه:

```bash
# مشاهدة الـ hung tasks اللي ممكن تكون blocked على rwsem
cat /proc/sys/kernel/hung_task_timeout_secs

# تفعيل كشف الـ hung tasks
echo 120 > /proc/sys/kernel/hung_task_timeout_secs

# قراءة حالة الـ tasks المحظورة
cat /proc/sys/kernel/hung_task_warnings

# عدد الـ tasks المسموح بإظهار تحذير عنها
cat /proc/sys/kernel/hung_task_all_cpu_backtrace
```

---

#### 3. استخدام الـ ftrace

##### تفعيل tracepoints الخاصة بالـ rwsem

```bash
# عرض كل الـ events المتاحة للـ locking
ls /sys/kernel/tracing/events/lock/

# تفعيل events الـ rwsem
echo 1 > /sys/kernel/tracing/events/lock/contention_begin/enable
echo 1 > /sys/kernel/tracing/events/lock/contention_end/enable

# تفعيل كل events الـ locking
echo 1 > /sys/kernel/tracing/events/lock/enable

# تفعيل tracing وقراءة النتائج
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
```

##### تتبع دوال الـ rwsem مباشرة بالـ function tracer

```bash
# تتبع down_read و down_write و up_read و up_write
echo function > /sys/kernel/tracing/current_tracer
echo 'down_read down_write up_read up_write down_write_trylock down_read_trylock downgrade_write' \
  > /sys/kernel/tracing/set_ftrace_filter

echo 1 > /sys/kernel/tracing/tracing_on
sleep 5
cat /sys/kernel/tracing/trace > /tmp/rwsem_trace.txt
echo 0 > /sys/kernel/tracing/tracing_on
```

##### تتبع rwsem_wake و rwsem_down_read_failed

```bash
# تتبع الدوال الداخلية (slow path)
echo 'rwsem_wake rwsem_down_read_slowpath rwsem_down_write_slowpath rwsem_spin_on_owner' \
  > /sys/kernel/tracing/set_ftrace_filter

echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

##### مثال على output الـ contention_begin

```
    kworker/0:1-23    [000] ..... 1234.567890: lock_contention_begin: 0xffff888012345678 (flags=0x1)
    kworker/0:1-23    [000] ..... 1234.567990: lock_contention_end:   0xffff888012345678
```

**التفسير:** الـ `flags=0x1` يعني الـ lock في حالة write-locked (`RWSEM_WRITER_LOCKED`).

---

#### 4. الـ printk والـ dynamic debug

##### تفعيل dynamic debug للـ locking subsystem

```bash
# تفعيل debug messages في ملفات الـ rwsem
echo 'file kernel/locking/rwsem.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file include/linux/rwsem.h +p'  > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ debug messages في subsystem الـ locking
echo 'module kernel +p' > /sys/kernel/debug/dynamic_debug/control

# عرض ما هو مفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep rwsem
```

##### إضافة printk مؤقتة لتتبع الحالة

```c
/* في كود يستخدم rwsem، أضف قبل down_read */
pr_debug("rwsem %p: count=0x%lx owner=0x%lx contended=%d\n",
         sem,
         atomic_long_read(&sem->count),
         atomic_long_read(&sem->owner),
         rwsem_is_contended(sem));
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الوظيفة | متى تستخدمها |
|---|---|---|
| `CONFIG_DEBUG_RWSEMS` | يضيف حقل `magic` للـ struct ويفعّل assertions | دايما في بيئة التطوير |
| `CONFIG_DEBUG_LOCK_ALLOC` | يتتبع ownership + يضيف `dep_map` | كشف use-after-free على الـ locks |
| `CONFIG_LOCKDEP` | يبني graph كامل لـ lock ordering | كشف deadlocks و lock inversions |
| `CONFIG_LOCKDEP_SUPPORT` | arch support للـ lockdep | شرط مسبق للـ LOCKDEP |
| `CONFIG_LOCK_STAT` | إحصائيات contention لكل lock | قياس performance |
| `CONFIG_DETECT_HUNG_TASK` | يكتشف tasks واقفة على lock | production debugging |
| `CONFIG_PROVE_LOCKING` | يُفعّل lockdep بالكامل | أقوى أداة للكشف المسبق |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكتشف sleep داخل atomic context | كشف down_read في spinlock context |
| `CONFIG_RWSEM_SPIN_ON_OWNER` | يفعّل MCS optimistic spinning | performance على SMP |
| `CONFIG_PREEMPT_RT` | يستبدل rwsem بـ rwbase_rt | real-time systems |

##### تفعيل في `.config`

```bash
# عرض الحالة الحالية
grep -E 'CONFIG_(DEBUG_RWSEM|LOCKDEP|LOCK_STAT|DETECT_HUNG|PROVE_LOCK)' /boot/config-$(uname -r)

# تفعيل عبر menuconfig
make menuconfig
# Kernel hacking → Lock Debugging
```

---

#### 6. أدوات خاصة بالـ Subsystem

##### الـ lock_stat — إحصائيات contention

```bash
# تفعيل lock statistics (يحتاج CONFIG_LOCK_STAT=y)
echo 1 > /proc/sys/kernel/lock_stat

# إعادة تصفير الإحصائيات
echo 0 > /proc/lock_stat

# قراءة النتائج
cat /proc/lock_stat
```

**مثال output:**

```
                              class name    con-bounces    contentions   waittime-min   waittime-max waittime-total   acq-bounces   acquisitions   holdtime-min   holdtime-max holdtime-total
---------------------------------------- -------------- -------------- -------------- -------------- -------------- -------------- -------------- -------------- -------------- --------------
                    &mm->mmap_lock-W:          1234           1500          0.50ns       12345.00ns    6789012.00ns           234          45678          0.10ns        234.00ns     567890.00ns
```

**التفسير:**
- **con-bounces**: كام مرة الـ lock انتقل بين CPUs وهو contested
- **waittime-max**: أطول وقت انتظر فيه task قبل ما يحصل على الـ lock

##### استخدام crash tool لفحص rwsem في kernel dump

```bash
# في crash prompt
crash> struct rw_semaphore <address>
crash> p ((struct rw_semaphore *)0xffff888012345678)->count
crash> p ((struct rw_semaphore *)0xffff888012345678)->owner
```

##### perf لقياس lock contention

```bash
# تتبع lock events بالـ perf
perf lock record -a -- sleep 5
perf lock report

# مثال output
Name   acquired  contended total-wait-nsec max-wait-nsec avg-wait-nsec
mmap_lock       5432       123         456789         12345          3710
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `WARNING: possible circular locking dependency detected` | lockdep اكتشف حلقة محتملة بين rwsem وlock تاني | راجع ترتيب الـ lock acquisition، استخدم `_nested()` APIs |
| `BUG: sleeping function called from invalid context` | تم استدعاء `down_read/down_write` من atomic context (IRQ, spinlock) | استخدم `down_read_trylock` أو اتأكد إن الكود مش في atomic context |
| `WARNING: lock held when returning to user space` | task رجع لـ userspace وهو لسه شايل rwsem | فيه path مش بيعمل `up_read/up_write` |
| `BUG: rwsem bad magic` | الـ magic field مش صح — corruption أو double-free | فحص memory corruption، KASAN |
| `INFO: task <name>:<pid> blocked for more than 120s` | task واقف على rwsem أكتر من الـ timeout | فحص الـ owner بـ `rwsem_owner()`، فحص deadlock |
| `WARNING: CPU: X PID: Y at kernel/locking/rwsem.c:Z` | assertion فشل في كود الـ rwsem | فحص stack trace، عادة misuse |
| `kernel BUG at kernel/locking/rwsem.c:` | bug واضح في state الـ rwsem | فحص الـ count value مباشرة |
| `possible deadlock detected` | lockdep شاف إن نفس الـ lock اتأخد مرتين | فحص recursive locking، استخدم different lock classes |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

##### في كود يستخدم rwsem

```c
/* تأكيد إن الـ rwsem محجوز قبل الوصول للبيانات المحمية */
void my_function(struct my_struct *obj)
{
    /* يطلع WARNING + stack trace لو الـ lock مش محجوز */
    rwsem_assert_held(&obj->lock);

    /* أو النسخة الأكثر تشددا */
    rwsem_assert_held_write(&obj->lock);

    /* access protected data */
}

/* كشف contention مرتفع */
static void check_rwsem_health(struct rw_semaphore *sem)
{
    if (rwsem_is_contended(sem)) {
        /* يسجل stack trace بدون panic */
        WARN_ONCE(1, "rwsem %p is contended under hot path\n", sem);
    }
}

/* كشف lock مش محجوز في مكان حساس */
static void critical_section_entry(struct rw_semaphore *sem)
{
    if (WARN_ON(!rwsem_is_locked(sem))) {
        dump_stack();
        return;
    }
    /* ... */
}
```

##### نقاط استراتيجية للوضع

```c
/* 1. عند التحقق من ownership قبل عملية destructive */
WARN_ON(!rwsem_is_locked(&my_rwsem));

/* 2. بعد down_read_trylock الفاشل في loop طويل */
int ret = down_read_trylock(&sem);
if (!ret) {
    WARN_ONCE(retry_count > 1000,
              "rwsem trylock spinning too long: %d retries\n",
              retry_count);
}

/* 3. عند كشف state غير متوقع */
long count = atomic_long_read(&sem->count);
if (WARN_ON(count < 0 && !(count & RWSEM_WRITER_LOCKED))) {
    pr_err("rwsem %p: unexpected count 0x%lx\n", sem, count);
    dump_stack();
}
```

---

### على مستوى الـ Hardware

---

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

**الـ rwsem** هو software primitive، مفيش hardware state مباشر ليه. لكن:

```bash
# تحقق إن الـ cache coherency شغال صح على SMP
# الـ count field هو atomic_long_t — اتحقق من CPU cache state

# عرض cache topology
cat /sys/devices/system/cpu/cpu0/cache/index*/coherency_line_size
cat /sys/devices/system/cpu/cpu0/cache/index*/size

# تحقق من NUMA topology لو الـ rwsem شايل contention عالي
numactl --hardware
numactl --show
```

##### المطابقة بين kernel state والـ hardware registers

```bash
# الـ atomic_long_t count بيتخزن في ذاكرة عادية
# تأكد إن الـ CPU mode صح (no speculation issues)
# على x86: اتحقق من الـ MFENCE/LFENCE/SFENCE usage

# قراءة CPU flags
grep -m1 'flags' /proc/cpuinfo | grep -E '(sse2|cx8|cx16)'
```

---

#### 2. تقنيات الـ Register Dump

**الـ rwsem** بيخزن حالته في الـ RAM مش في hardware registers، لذا:

```bash
# قراءة حالة rwsem مباشرة من الذاكرة عبر /proc/kcore (يحتاج root)
# أو عبر crash tool على kernel dump

# مثال: قراءة قيمة count لـ rwsem في عنوان معروف
# عبر crash:
crash> rd -64 0xffff888012345678 8

# عبر gdb على vmlinux + /proc/kcore:
gdb vmlinux /proc/kcore
(gdb) x/4gx 0xffff888012345678
```

##### تفسير قيمة الـ count

```
count = 0x0000000000000000  → UNLOCKED (RWSEM_UNLOCKED_VALUE)
count = 0x0000000000000001  → WRITER LOCKED (RWSEM_WRITER_LOCKED bit 0)
count = 0x000000000000FF00  → READERS present (bits 8+)
count = 0xFFFFFFFFFFFFFF00  → READERS WAITING (negative values)
```

```c
/* من rwsem.h */
#define RWSEM_UNLOCKED_VALUE    0UL
#define RWSEM_WRITER_LOCKED     (1UL << 0)  /* bit 0 = write locked */
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

**الـ rwsem** هو software lock — الـ logic analyzer لا يرى الـ lock مباشرة. لكن:

##### مراقبة أثر الـ contention على الـ bus

```
- اربط الـ logic analyzer على bus interface (PCIe, memory bus)
- ابحث عن patterns من الـ cache line bouncing:
  * ظهور متكرر لـ BusRd + BusRdX على نفس العنوان
  * العنوان هو عنوان sem->count (atomic_long_t)

- على ARM: استخدم ETM (Embedded Trace Macrocell) لتتبع memory accesses
- على x86: استخدم Intel PT (Processor Trace)
```

##### تفعيل Intel PT للتتبع

```bash
# تسجيل باستخدام perf + Intel PT
perf record -e intel_pt// -p <pid> -- sleep 1
perf script --itrace=i1000ns | grep -E '(down_read|down_write|up_read|up_write)'
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Kernel Log | السبب المحتمل |
|---|---|---|
| NUMA memory latency عالية | `lock_contention_end` بـ waittime عالي جدا | الـ sem->count في NUMA node بعيد عن الـ waiters |
| Cache thrashing على SMP | `perf stat` يظهر cache-misses عالية | الـ rwsem struct مشترك بين CPUs كتير |
| Memory corruption | `BUG: rwsem bad magic` | hardware fault أو bitflip في ECC-less RAM |
| Spurious wakeups | task يصحى وياخد الـ lock فورا بدون contention | hardware interrupt أو signal بيكسر الـ sleep |
| CPU frequency scaling | وقت الانتظار بيتغير بشكل غير منتظم | الـ CPU بيدخل C-states أثناء انتظار الـ lock |

---

#### 5. الـ Device Tree Debugging

**الـ rwsem** مش بيتأثر بالـ Device Tree مباشرة. لكن في driver code:

```bash
# لو الـ rwsem بيحمي resource مربوطة بـ DT node
# تحقق إن الـ resource اتعرّف صح

# عرض الـ DT المحمّل في الـ kernel
ls /proc/device-tree/
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -A5 'your-device'

# التحقق من إن driver اتعرّف على الـ device
dmesg | grep -i 'your-driver-name'

# التحقق من الـ probe order (يؤثر على متى يتم init_rwsem)
cat /sys/bus/platform/drivers/your-driver/uevent
```

---

### الأوامر العملية الجاهزة للنسخ

---

#### سيناريو 1: كشف Deadlock

```bash
#!/bin/bash
# تفعيل lockdep وإعداد tracing لكشف deadlock

# 1. تأكد إن الـ kernel مبني بـ CONFIG_PROVE_LOCKING=y
grep CONFIG_PROVE_LOCKING /boot/config-$(uname -r)

# 2. اقرا lockdep warnings
dmesg | grep -E '(circular locking|deadlock|DEADLOCK)' | tail -50

# 3. اقرا الـ lock chain الكاملة
cat /proc/lockdep_chains | head -200

# 4. اعرض الـ tasks الواقفة على locks
cat /proc/sysrq-trigger <<< 'd'
# أو
echo d > /proc/sysrq-trigger
# ثم اقرا dmesg
dmesg | tail -100
```

#### سيناريو 2: تحليل Contention عالي

```bash
#!/bin/bash
# قياس وتحليل الـ rwsem contention

# 1. صفّر الإحصائيات
echo 0 > /proc/lock_stat

# 2. شغّل الـ workload
# ... run your workload here ...

# 3. اقرا النتائج مرتبة حسب الـ contention
cat /proc/lock_stat | sort -k3 -rn | head -20

# 4. استخدم perf لتحديد الـ call sites
perf lock record -a -g -- sleep 10
perf lock report --call-graph

# 5. تفعيل ftrace للتتبع المفصل
echo 1 > /sys/kernel/tracing/events/lock/contention_begin/enable
echo 1 > /sys/kernel/tracing/events/lock/contention_end/enable
echo 1 > /sys/kernel/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | grep contention | head -50
```

#### سيناريو 3: تحديد الـ Owner عند Hang

```bash
#!/bin/bash
# لما task واقف على rwsem لفترة طويلة

# 1. اعرف الـ PID المشكوك فيه
cat /proc/sysrq-trigger <<< 'w'
# أو
echo w > /proc/sysrq-trigger
dmesg | grep -A3 "task.*blocked"

# 2. اقرا stack trace للـ task
cat /proc/<PID>/stack

# 3. لو عندك CONFIG_DEBUG_RWSEMS=y، استخدم crash/gdb
# في crash:
# crash> bt <PID>
# crash> struct rw_semaphore 0x<sem_addr>

# 4. اقرا /proc/<PID>/wchan لمعرفة مين بلّش الـ wait
cat /proc/<PID>/wchan
# لو رجع 'rwsem_down_read_slowpath' → مستنى read lock
# لو رجع 'rwsem_down_write_slowpath' → مستنى write lock

# 5. اقرا حالة كل الـ threads
for pid in $(ls /proc | grep '^[0-9]'); do
    wchan=$(cat /proc/$pid/wchan 2>/dev/null)
    if echo "$wchan" | grep -q 'rwsem'; then
        echo "PID $pid stuck on rwsem: $wchan"
        cat /proc/$pid/comm
    fi
done
```

#### سيناريو 4: مراقبة الـ count في Real-time

```bash
#!/bin/bash
# مراقبة حالة rwsem بشكل continuous (يحتاج معرفة عنوان الـ sem)

# أولا: اعرف عنوان الـ sem المطلوب
grep 'my_rwsem' /proc/kallsyms
# مثلا: ffffffffc0123456 D my_rwsem [my_module]

SEM_ADDR=0xffffffffc0123456

# اقرا الـ count كل ثانية
while true; do
    # count هو أول field في الـ struct (offset 0)
    VAL=$(python3 -c "
import struct, ctypes
with open('/dev/mem', 'rb') as f:
    # NOTE: يحتاج CONFIG_STRICT_DEVMEM=n أو bootloader param
    f.seek($SEM_ADDR)
    data = f.read(8)
    print(hex(struct.unpack('<Q', data)[0]))
" 2>/dev/null || echo "N/A")
    echo "$(date +%T) count=$VAL"
    sleep 1
done

# بديل أكثر أمانا: استخدم bpftrace
bpftrace -e '
kprobe:down_read {
    printf("down_read sem=%p count=%ld pid=%d comm=%s\n",
           arg0,
           ((struct rw_semaphore *)arg0)->count.counter,
           pid, comm);
}
kprobe:up_read {
    printf("up_read   sem=%p count=%ld pid=%d comm=%s\n",
           arg0,
           ((struct rw_semaphore *)arg0)->count.counter,
           pid, comm);
}
'
```

#### سيناريو 5: تفعيل DEBUG_RWSEMS وفحص magic

```bash
# 1. تأكد إن CONFIG_DEBUG_RWSEMS=y في الـ kernel
grep CONFIG_DEBUG_RWSEMS /boot/config-$(uname -r)

# 2. لو مفعّل، الـ kernel هيطبع تلقائيا لو الـ magic field تالف:
dmesg | grep -i 'rwsem bad magic'

# 3. استخدم KASAN للكشف المبكر عن corruption
# يحتاج CONFIG_KASAN=y
dmesg | grep -E 'BUG: KASAN.*rwsem'

# 4. استخدم KFENCE كبديل أخف
# يحتاج CONFIG_KFENCE=y
dmesg | grep -i 'KFENCE'
```

#### سيناريو 6: bpftrace لتتبع مفصل

```bash
# تتبع جميع عمليات الـ rwsem مع timing
bpftrace -e '
kprobe:down_write {
    @start[tid] = nsecs;
    @sem[tid] = arg0;
    printf("[%s:%d] down_write sem=%p\n", comm, pid, arg0);
}
kretprobe:down_write {
    if (@start[tid]) {
        $wait = nsecs - @start[tid];
        printf("[%s:%d] down_write acquired sem=%p wait=%ldus\n",
               comm, pid, @sem[tid], $wait/1000);
        delete(@start[tid]);
        delete(@sem[tid]);
    }
}
kprobe:up_write {
    printf("[%s:%d] up_write sem=%p\n", comm, pid, arg0);
}
kprobe:down_read_trylock / retval == 0 / {
    printf("[%s:%d] down_read_trylock FAILED sem=%p\n", comm, pid, arg0);
}
'
```

#### سيناريو 7: فحص الـ wait_list مباشرة

```bash
# في crash tool بعد panic أو kdump
crash> struct rw_semaphore 0xffff888012345678
crash> list rw_semaphore.wait_list 0xffff888012345678
# عرض كل الـ waiters:
crash> foreach task bt | grep -A5 'rwsem_down'

# عرض عدد الـ waiters
crash> p ((struct rw_semaphore *)0xffff888012345678)->wait_list

# التحقق من الـ owner
crash> p ((struct rw_semaphore *)0xffff888012345678)->owner
# bit 0 = RWSEM_READER_OWNED
# bits 1+ = pointer to task_struct of writer
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Deadlock في Driver الـ SPI على RK3562

#### العنوان
**الـ SPI driver بيتعلق وبيوقف الـ industrial gateway كلها**

#### السياق
شركة بتعمل industrial gateway على بورد مبنية على **RK3562**. الـ gateway بتجمع بيانات من حساسات عن طريق **SPI** وبتبعتها على السحابة. في الإنتاج، العملاء بيبلغوا إن الجهاز بيتجمد تماماً كل بضع ساعات ومحتاج restart يدوي.

#### المشكلة
الـ system بيتعلق ومفيش response. الـ watchdog مش شغال عشان هو نفسه عايز يمسك lock.

```bash
# الـ SysRq بيدي:
$ echo t > /proc/sysrq-trigger

# Output في dmesg:
INFO: task spi-worker:342 blocked for more than 120 seconds.
      Not tainted 6.6.0-rc3 #1
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
task:spi-worker      state:D stack:    0 pid:  342 ppid:   1
Call Trace:
 __schedule+0x4b8/0x8e0
 schedule+0x5c/0xd0
 rwsem_down_write_slowpath+0x2f4/0x6a0
 down_write+0x6c/0x80         # <-- blocked هنا
 spi_data_update+0x48/0x120   # <-- في الـ driver
```

#### التحليل
بنفتح الكود اللي بيستخدم الـ `rw_semaphore`:

```c
/* في spi_sensor_driver.c */
struct sensor_dev {
    struct rw_semaphore data_lock; /* protects sensor_buf */
    u8 sensor_buf[512];
    struct work_struct irq_work;
};

/* IRQ bottom-half handler */
static void sensor_irq_work(struct work_struct *work)
{
    struct sensor_dev *dev = container_of(work, ...);

    /* BUG: بياخد write lock */
    down_write(&dev->data_lock);
    spi_read(dev->spi, dev->sensor_buf, 512);
    up_write(&dev->data_lock);
}

/* Read path من الـ userspace */
static ssize_t sensor_read(struct file *f, char __user *buf, ...)
{
    struct sensor_dev *dev = f->private_data;

    down_read(&dev->data_lock);          /* read lock */
    /* بيعمل I/O operation وهو ماسك read lock */
    ret = some_other_subsystem_call();   /* بياخد نفس الـ lock! */
    copy_to_user(buf, dev->sensor_buf, count);
    up_read(&dev->data_lock);
}
```

المشكلة: `some_other_subsystem_call()` جوّاها بتحاول تاخد write lock على نفس الـ `rw_semaphore`. والـ writer بيستنى الـ readers يخلصوا، والـ reader بيستنى الـ writer. **الـ deadlock** جه من إن:

1. الـ `count` field في `rw_semaphore` بتبقى `!= RWSEM_UNLOCKED_VALUE (0UL)`
2. الـ `wait_list` مش فاضية — `rwsem_is_contended()` بترجع `true`
3. كل الـ tasks بتستنى في `wait_list` ومفيش حد بيخلص

```c
/* من rwsem.h — الـ functions اللي كانت بتبقى blocked */
static inline int rwsem_is_contended(struct rw_semaphore *sem)
{
    return !list_empty(&sem->wait_list); /* كانت بترجع 1 دايماً */
}
```

**الـ CONFIG_RWSEM_SPIN_ON_OWNER** كان enabled، يعني الـ optimistic spinners على `osq` lock كانوا بيسبقوا الـ waiters الحقيقيين وبيزودوا الضغط.

#### الحل

```bash
# أول حاجة: تأكيد الـ deadlock بـ lockdep
$ cat /proc/config.gz | gunzip | grep LOCKDEP
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_LOCKDEP=y

# في dmesg هيظهر:
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
spi-worker/342 is trying to acquire lock:
  (data_lock){++++}, at: sensor_irq_work+0x48
but task is already holding lock:
  (data_lock){++++}, at: sensor_read+0x30
```

الحل في الكود: فصل الـ lock عن الـ I/O path:

```c
static void sensor_irq_work(struct work_struct *work)
{
    struct sensor_dev *dev = container_of(work, ...);
    u8 tmp_buf[512];

    /* القراءة برّة الـ lock */
    spi_read(dev->spi, tmp_buf, 512);

    /* بس النسخ للـ buffer المحمي بالـ write lock */
    down_write(&dev->data_lock);
    memcpy(dev->sensor_buf, tmp_buf, 512);
    up_write(&dev->data_lock);
}
```

أو استخدام الـ scope-based API من `rwsem.h` عشان يضمن الـ release:

```c
/* باستخدام DEFINE_LOCK_GUARD_1 من rwsem.h */
static void sensor_irq_work(struct work_struct *work)
{
    struct sensor_dev *dev = container_of(work, ...);
    u8 tmp_buf[512];

    spi_read(dev->spi, tmp_buf, 512);

    /* up_write() بيتعمل تلقائياً لما scope ينتهي */
    scoped_guard(rwsem_write, &dev->data_lock) {
        memcpy(dev->sensor_buf, tmp_buf, 512);
    }
}
```

#### الدرس المستفاد
**ابعد عن الـ blocking I/O وأنت ماسك أي نوع من الـ rwsem.** الـ `rwsem.h` بيوفر `rwsem_is_contended()` — استخدمها لو عايز تعرف فيه حاجة بتستنى. وفعّل `CONFIG_DEBUG_RWSEMS` في الـ development kernel عشان الـ `magic` field يساعدك تكتشف الـ corruption بدري.

---

### السيناريو الثاني: Priority Inversion على STM32MP1 في نظام Real-Time

#### العنوان
**الـ real-time task بيتأخر بسبب rwsem في نظام PREEMPT_RT على STM32MP1**

#### السياق
منتج **automotive ECU** مبني على **STM32MP1** (Cortex-A7 + Cortex-M4). الـ Linux بيشتغل على الـ A7 بـ `CONFIG_PREEMPT_RT=y`. الـ task المسؤول عن قراءة بيانات الـ **CAN bus** وإرسالها للـ M4 لازم يشتغل في 5ms كحد أقصى. في الاختبار، الـ latency بتوصل 50ms.

#### المشكلة
الـ PREEMPT_RT kernel بيستخدم implementation تانية للـ `rw_semaphore` — مش نفس اللي في الكود العادي. من `rwsem.h`:

```c
#else /* CONFIG_PREEMPT_RT */

#include <linux/rwbase_rt.h>

context_lock_struct(rw_semaphore) {
    struct rwbase_rt rwbase;   /* <-- implementation مختلفة خالص */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map  dep_map;
#endif
};
```

الـ `rwbase_rt` بتحول الـ rwsem لـ **PI-aware mutex** — يعني الـ priority inheritance شغال. لكن الـ driver القديم كان بيستخدم `down_read()` في الـ high-priority CAN task، والـ writer (low-priority logging task) كان ماسك الـ lock لفترة طويلة.

#### التحليل

```bash
# بنشوف الـ latency بـ cyclictest
$ cyclictest -p 80 -t 1 -n -i 5000 --duration=60s
# Max latency: 48921 us  <-- كارثة!

# بنفتح الـ trace
$ echo 1 > /sys/kernel/debug/tracing/events/lock/enable
$ cat /sys/kernel/debug/tracing/trace | grep rwsem
  can-rt/156   rwsem_acquire: ... down_read ...
  logger/89    rwsem_acquire: ... down_write ...
  can-rt/156   rwsem_blocked: ... waiting for write ...
```

الـ sequence:

```
Timeline:
  t=0ms:   logger (prio 20) → down_write(&config_lock) ✓
  t=1ms:   can-rt (prio 80) → down_read(&config_lock)  BLOCKED
  t=1ms:   can-rt ينتظر في wait_list
  t=48ms:  logger ينهي كتابة الـ log → up_write()
  t=48ms:  can-rt يشتغل — تأخر 47ms!
```

في الـ PREEMPT_RT، الـ `rwsem_is_locked()` من `rwsem.h` بتشتغل على `rwbase`:

```c
/* PREEMPT_RT version */
static __always_inline int rwsem_is_locked(const struct rw_semaphore *sem)
{
    return rw_base_is_locked(&sem->rwbase); /* PI-aware check */
}
```

الـ priority inheritance موجودة لكن الـ logger مش بياخد priority الـ can-rt لأن الـ rwsem في mode القراءة (reader) مش بيعمل PI بشكل مثالي لكل الحالات.

#### الحل

**الحل الأول**: استبدال الـ `rw_semaphore` بـ `mutex` لو الـ read/write ratio مش كبير:

```c
/* بدل */
struct rw_semaphore config_lock;

/* استخدم */
struct mutex config_lock;
```

**الحل الثاني**: لو لازم rwsem، استخدم `down_read_killable()` مع timeout في الـ RT task:

```c
static int can_rt_handler(void *data)
{
    struct can_dev *dev = data;
    int ret;

    /* مش بنستخدم down_read() العادية */
    ret = down_read_killable(&dev->config_lock);
    if (ret) {
        /* تأخير مقبول — نعيد المحاولة أو نستخدم cached value */
        use_cached_config(dev);
        return 0;
    }

    process_can_frame(dev);
    up_read(&dev->config_lock);
    return 0;
}
```

**الحل الثالث** (الأمثل): فصل الـ config الثابتة من الـ dynamic state:

```c
/* config تُقرأ مرة وتُخزن في RCU-protected pointer */
/* الـ RT task يقرأ بـ rcu_read_lock() بدون blocking */
```

#### الدرس المستفاد
في `CONFIG_PREEMPT_RT`، الـ `rw_semaphore` بيتحول لـ `rwbase_rt` — behavior مختلف. الـ RT tasks لازم تتجنب `down_write()` في أي كود بيستدعيه الـ high-priority thread. دايماً اختبر الـ latency بـ `cyclictest` قبل ما تشحن الـ product.

---

### السيناريو الثالث: Memory Corruption في Driver الـ HDMI على i.MX8

#### العنوان
**الـ Android TV box بيعمل kernel panic عشوائي مرتبط بـ HDMI hotplug**

#### السياق
**Android TV box** مبني على **i.MX8M Plus**. المنتج بيعمل fine لساعات، وفجأة kernel panic بدون pattern واضح. البورد بتتوصل بتلفزيونات مختلفة عن طريق **HDMI**.

#### المشكلة

```bash
# الـ panic message:
BUG: KASAN: use-after-free in hdmi_connector_status+0x94/0x120
Read of size 8 at addr ffff888012345678

Call Trace:
 hdmi_connector_status+0x94
 drm_helper_probe_detect+0x44
 drm_kms_helper_hotplug_event+0x68
 ...

# وفي نفس الوقت:
WARNING: CPU: 2 PID: 891 at kernel/rwsem.c:...
 rwsem_assert_held_nolockdep: rwsem not held!
```

#### التحليل
الـ panic بييجي لأن الـ HDMI connector struct اتفرت وهو لسه بيتقرأ. الكود في الـ driver:

```c
struct hdmi_dev {
    struct rw_semaphore conn_lock; /* protects connector list */
    struct list_head connectors;
    /* ... */
};

/* Thread A: hotplug event */
static irqreturn_t hdmi_hotplug_irq(int irq, void *data)
{
    struct hdmi_dev *dev = data;

    /* BUG: مش ماسك أي lock! */
    list_for_each_entry(conn, &dev->connectors, node) {
        drm_helper_probe_detect(conn, false);  /* بيقرأ conn بدون lock */
    }
    return IRQ_HANDLED;
}

/* Thread B: disconnect/cleanup */
static void hdmi_disconnect(struct hdmi_dev *dev)
{
    down_write(&dev->conn_lock);
    list_del(&conn->node);
    kfree(conn);               /* بيحذف conn */
    up_write(&dev->conn_lock);
}
```

لو Thread B حذف `conn` وThread A لسه بيقرأ منه — **use-after-free**.

الـ `rwsem_assert_held_nolockdep()` من `rwsem.h` كانت ممكن تكشف الغلطة لو اتحطت في الكود:

```c
/* من rwsem.h */
static inline void rwsem_assert_held_nolockdep(const struct rw_semaphore *sem)
    __assumes_ctx_lock(sem)
{
    /* WARN لو count == 0 يعني مفيش حد ماسك الـ lock */
    WARN_ON(atomic_long_read(&sem->count) == RWSEM_UNLOCKED_VALUE);
}
```

لو الـ developer حط `rwsem_assert_held_nolockdep(&dev->conn_lock)` في بداية `hdmi_hotplug_irq`، كان هيشوف الـ WARN فوراً.

#### الحل

**الخطوة الأولى**: تفعيل `CONFIG_DEBUG_RWSEMS`:

```bash
# في .config
CONFIG_DEBUG_RWSEMS=y
# ده بيفعّل الـ magic field في struct rw_semaphore
# وبيعمل checks إضافية في كل down/up
```

**الخطوة الثانية**: تصحيح الكود:

```c
static irqreturn_t hdmi_hotplug_irq(int irq, void *data)
{
    struct hdmi_dev *dev = data;

    /* الـ IRQ context مش ممكن يستخدم down_read() مباشرة (sleeping) */
    /* نجدول work في الـ workqueue */
    schedule_work(&dev->hotplug_work);
    return IRQ_HANDLED;
}

static void hdmi_hotplug_work(struct work_struct *work)
{
    struct hdmi_dev *dev = container_of(work, ...);

    /* دلوقتي ممكن نمسك الـ lock */
    down_read(&dev->conn_lock);

    /* أو باستخدام scope guard من rwsem.h */
    scoped_guard(rwsem_read, &dev->conn_lock) {
        list_for_each_entry(conn, &dev->connectors, node) {
            drm_helper_probe_detect(conn, false);
        }
    }
    /* up_read() اتعملت تلقائياً */
}
```

**الخطوة الثالثة**: استخدام `rwsem_assert_held()` في كل function تستخدم الـ data المحمية:

```c
static void process_connector(struct hdmi_dev *dev, struct connector *conn)
{
    /* هتطلع WARN لو حد نسى الـ lock */
    rwsem_assert_held(&dev->conn_lock);

    /* بقية الكود ... */
}
```

#### الدرس المستفاد
الـ `rwsem.h` بيوفر `rwsem_assert_held()` و`rwsem_assert_held_nolockdep()` — استخدمهم في كل function بتعتمد على الـ lock. الـ `CONFIG_DEBUG_RWSEMS` بيفعّل الـ `magic` field اللي بيكشف الـ corruption. الـ IRQ handlers مش ممكن تمسك sleeping locks زي الـ rwsem — لازم تستخدم workqueue.

---

### السيناريو الرابع: Performance Regression في Driver الـ I2C على AM62x

#### العنوان
**الـ IoT sensor hub على AM62x بيستهلك CPU أكتر من اللازم بسبب rwsem contention**

#### السياق
**IoT sensor hub** بيشغّل 16 حساس عن طريق **I2C** على **TI AM62x**. كل حساس له thread بيقرأ البيانات كل 100ms. في الاختبار، الـ CPU usage وصل 80% وكان المفروض يبقى 20%.

#### المشكلة

```bash
# perf stat بيدي:
$ perf stat -e lock:contention_begin,lock:contention_end -p $(pgrep sensor-hub)

 Performance counter stats for process id '1234':
  lock:contention_begin    45,231      # 45K contention events في ثانية!
  lock:contention_end      45,229

# perf lock بيشوف مين السبب:
$ perf lock report
         Name   acquired  contended  total wait   max wait
  &hub->rwsem      45231      44891   8.234 sec   0.823 ms
```

#### التحليل
الـ driver الأصلي:

```c
struct sensor_hub {
    struct rw_semaphore rwsem; /* protects ALL sensors */
    struct sensor sensors[16];
};

/* كل thread بييجي ويمسك write lock حتى لو بس بيقرأ */
static int sensor_thread(void *data)
{
    struct sensor_hub *hub = data;

    while (!kthread_should_stop()) {
        /* BUG: write lock مش لازم لقراءة البيانات */
        down_write(&hub->rwsem);

        i2c_read(sensor->i2c, sensor->buf, 32);
        process_data(sensor);

        up_write(&hub->rwsem);
        msleep(100);
    }
}
```

لما 16 thread كلهم عايزين write lock، الـ contention هائل. وبسبب الـ `RWSEM_SPIN_ON_OWNER` اللي enabled:

```c
/* من rwsem.h */
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
struct optimistic_spin_queue osq; /* spinner MCS lock */
#endif
```

الـ 15 thread التانيين بيعملوا spin على الـ `osq` MCS lock بدل ما يناموا، وده بياكل CPU.

الـ `rwsem_is_contended()` كانت بترجع `true` دايماً:

```c
static inline int rwsem_is_contended(struct rw_semaphore *sem)
{
    return !list_empty(&sem->wait_list); /* wait_list كانت ممتلية */
}
```

#### الحل

**الخطوة الأولى**: فصل الـ lock — كل sensor له lock خاص:

```c
struct sensor {
    struct rw_semaphore sem; /* per-sensor lock */
    u8 buf[32];
    struct i2c_client *i2c;
};

struct sensor_hub {
    struct sensor sensors[16]; /* كل واحد له lock منفصل */
};
```

**الخطوة الثانية**: استخدام read lock للقراءة:

```c
static int sensor_thread(void *data)
{
    struct sensor *sensor = data;

    while (!kthread_should_stop()) {
        u8 tmp[32];

        /* القراءة من الـ I2C خارج الـ lock */
        i2c_read(sensor->i2c, tmp, 32);

        /* write lock بس لتحديث الـ buffer */
        scoped_guard(rwsem_write, &sensor->sem) {
            memcpy(sensor->buf, tmp, 32);
        }

        msleep(100);
    }
}

/* الـ consumer threads بياخدوا read lock */
static void read_sensor_data(struct sensor *sensor, u8 *out)
{
    scoped_guard(rwsem_read, &sensor->sem) {
        memcpy(out, sensor->buf, 32);
    }
    /* up_read() تلقائي */
}
```

**الخطوة الثالثة**: مراقبة الـ contention بعد التغيير:

```bash
$ perf lock report
         Name   acquired  contended  total wait   max wait
  sensor[0].sem      600          2   0.012 ms   0.006 ms
  # كل 100ms = 10 reads per second * 60 seconds = 600, contention تقريباً صفر
```

الـ CPU usage نزل من 80% لـ 18%.

#### الدرس المستفاد
**الـ `rw_semaphore` المشترك بين threads كتير بيعمل contention شديد.** الـ design الصح: fine-grained locking — lock منفصل لكل resource مستقل. واستخدم `down_read()` للقراءة لأن كتير readers ممكن يشتغلوا مع بعض. راقب الـ `rwsem_is_contended()` في الـ production code لو عايز تعمل adaptive behavior.

---

### السيناريو الخامس: Bring-up مشكلة في Driver الـ USB على Allwinner H616

#### العنوان
**الـ USB driver على H616 بييجي kernel panic في early boot أثناء bring-up**

#### السياق
فريق bring-up بيشتغل على **custom Android TV box** بـ **Allwinner H616**. الـ BSP جاي من Allwinner، وفي أول boot بعد إضافة **USB 3.0** driver جديد، الـ kernel بييجي panic قبل ما يوصل لـ userspace.

#### المشكلة

```bash
# الـ early boot panic:
Unable to handle kernel NULL pointer dereference at virtual address 00000030
Internal error: Oops: 96000006 [#1] PREEMPT SMP
CPU: 0 PID: 1 Comm: swapper/0
Call Trace:
 rwsem_down_read_slowpath+0x18/0x3e0
 down_read+0x4c/0x60
 usb_phy_get_charger+0x30/0x80   # <-- USB driver
 ...
```

#### التحليل
الـ `down_read` على `rwsem` بيعمل NULL pointer dereference. يعني الـ `rw_semaphore` نفسه مش initialized.

بننظر في الـ USB driver:

```c
struct usb_phy_h616 {
    struct usb_phy phy;
    struct rw_semaphore charger_lock; /* <-- مش اتعملتلها init */
    struct list_head chargers;
    /* ... */
};

static int h616_usb_phy_probe(struct platform_device *pdev)
{
    struct usb_phy_h616 *h616;

    h616 = devm_kzalloc(&pdev->dev, sizeof(*h616), GFP_KERNEL);
    if (!h616)
        return -ENOMEM;

    /* BUG: نسوا يعملوا init_rwsem! */
    /* المفروض: init_rwsem(&h616->charger_lock); */

    INIT_LIST_HEAD(&h616->chargers);
    usb_add_phy(&h616->phy, USB_PHY_TYPE_USB3);

    return 0;
}

static struct usb_phy *usb_phy_get_charger(struct usb_phy *phy)
{
    struct usb_phy_h616 *h616 = container_of(phy, ...);

    down_read(&h616->charger_lock); /* CRASH: uninitialized rwsem */
    /* ... */
}
```

`devm_kzalloc()` بتعمل zero-fill للـ struct. الـ `rw_semaphore` uninitialized — الـ `count` field بتكون 0 (زي `RWSEM_UNLOCKED_VALUE`) لكن الـ `wait_lock` و`wait_list` مش initialized صح.

```c
/* من rwsem.h: الـ init macro اللي المفروض اتستخدمت */
#define init_rwsem(sem)                         \
do {                                            \
    static struct lock_class_key __key;         \
                                                \
    __init_rwsem((sem), #sem, &__key);          \
} while (0)
```

`__init_rwsem()` بتعمل:
1. `atomic_long_set(&sem->count, RWSEM_UNLOCKED_VALUE)` — الـ count
2. `atomic_long_set(&sem->owner, 0)` — الـ owner
3. `raw_spin_lock_init(&sem->wait_lock)` — الـ spinlock **المهم**
4. `INIT_LIST_HEAD(&sem->wait_list)` — الـ list
5. لو `CONFIG_DEBUG_RWSEMS`: `sem->magic = sem`

بدون الـ init، لما `down_read` تحاول تمسك `wait_lock` تعمل crash.

#### الحل

**الخطوة الأولى**: إضافة `init_rwsem()` في الـ probe:

```c
static int h616_usb_phy_probe(struct platform_device *pdev)
{
    struct usb_phy_h616 *h616;

    h616 = devm_kzalloc(&pdev->dev, sizeof(*h616), GFP_KERNEL);
    if (!h616)
        return -ENOMEM;

    /* الإضافة المطلوبة */
    init_rwsem(&h616->charger_lock);

    INIT_LIST_HEAD(&h616->chargers);
    usb_add_phy(&h616->phy, USB_PHY_TYPE_USB3);

    return 0;
}
```

**أو** لو الـ struct static (في module level)، استخدام `DECLARE_RWSEM`:

```c
/* للـ static global instances */
static DECLARE_RWSEM(global_charger_lock);
```

الـ `DECLARE_RWSEM` من `rwsem.h`:

```c
#define DECLARE_RWSEM(name) \
    struct rw_semaphore name = __RWSEM_INITIALIZER(name)

#define __RWSEM_INITIALIZER(name)               \
    { __RWSEM_COUNT_INIT(name),                 \
      .owner = ATOMIC_LONG_INIT(0),             \
      __RWSEM_OPT_INIT(name)                    \
      .wait_lock = __RAW_SPIN_LOCK_UNLOCKED(name.wait_lock),\
      .wait_list = LIST_HEAD_INIT((name).wait_list), \
      __RWSEM_DEBUG_INIT(name)                  \
      __RWSEM_DEP_MAP_INIT(name) }
```

**الخطوة الثانية**: تفعيل `CONFIG_DEBUG_RWSEMS` في الـ bring-up phase:

```bash
# في kernel config للـ development builds
CONFIG_DEBUG_RWSEMS=y
CONFIG_DEBUG_LOCK_ALLOC=y
CONFIG_LOCKDEP=y
```

مع `CONFIG_DEBUG_RWSEMS`، الـ `magic` field بتتعمل set في `__init_rwsem()` لـ `sem->magic = sem`. لو الـ rwsem مش initialized، الـ `magic` بتكون `NULL` أو قيمة غريبة، وكل `down_read/write` بتتحقق منها وبتطلع WARN واضح:

```bash
# بدل الـ cryptic NULL deref:
WARNING: at kernel/rwsem.c:... rwsem_check_magic: bad magic number 0
# ده أوضح بكتير!
```

**الخطوة الثالثة**: الـ Device Tree مش محتاج تغيير هنا، لكن لو الـ USB PHY محتاج clock أو reset من الـ DT:

```dts
/* sun50i-h616-usb.dts */
&usb3phy {
    status = "okay";
    clocks = <&ccu CLK_USB_PHY1>;
    resets = <&ccu RST_USB_PHY1>;
};
```

#### الدرس المستفاد
**الـ `kzalloc()` مش بديل عن `init_rwsem()`.** الـ zero memory بتخلي `count = 0` (اللي هو `RWSEM_UNLOCKED_VALUE`) لكن الـ `wait_lock` (raw_spinlock) محتاج initialization صحيحة. دايماً في الـ bring-up، فعّل `CONFIG_DEBUG_RWSEMS` — الـ `magic` field بتكشف uninitialized rwsems فوراً بدل الـ cryptic crashes. وفي الـ probe functions، اعمل list مرتبة: allocate → initialize locks → initialize lists → register.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي المصادر الأساسية اللي بتوثق تطور الـ rwsem من بدايته لحد النسخ الحديثة:

| المقال | الأهمية |
|--------|---------|
| [read/write semaphores — أول patch رسمي (2001)](https://lwn.net/2001/0412/a/rw_semaphores.php3) | البداية التاريخية للـ rwsem في الكيرنل |
| [generic rwsem — المقال الأساسي (2007)](https://lwn.net/Articles/230442/) | شرح التصميم العام للـ `struct rw_semaphore` والـ slow path |
| [rwsem: generic rwsem implementation](https://lwn.net/Articles/212538/) | تفاصيل الـ generic implementation قبل التحسينات |
| [rwsem: Support optimistic spinning (2014)](https://lwn.net/Articles/598577/) | إضافة الـ optimistic spinning — أكبر تحسين أداء للـ rwsem |
| [locking/rwsem: Rwsem rearchitecture part 1 (2019)](https://lwn.net/Articles/780998/) | إعادة هيكلة شاملة من Waiman Long — الجزء الأول |
| [locking/rwsem: Rwsem rearchitecture part 2 (2019)](https://lwn.net/Articles/788946/) | الجزء الثاني: waiter handoff + reader optimistic spinning |
| [Kernel development — rwsem overview](https://lwn.net/Articles/123948/) | شرح مبسط لمفهوم الـ rwsem في الكيرنل |
| [Speculating on page faults](https://lwn.net/Articles/369511/) | أزمة الـ `mmap_sem` وتأثير الـ cacheline bouncing على الـ rwsem |
| [Memory management locking](https://lwn.net/Articles/591978/) | نقاشات تأمين الـ memory subsystem والـ rwsem |
| [Shrinking shrinker locking overhead](https://lwn.net/Articles/944199/) | أمثلة حديثة على تحسينات الـ rwsem في subsystems مختلفة |

---

### توثيق الكيرنل الرسمي

الملفات دي موجودة في شجرة الكيرنل مباشرة:

```
Documentation/locking/locktypes.rst          ← مقارنة شاملة بين كل أنواع الـ locks
Documentation/locking/lockdep-design.rst     ← تصميم الـ lockdep وazمنهجية nested locking
Documentation/locking/percpu-rw-semaphore.rst← الـ percpu_rw_semaphore المشتق من الـ rwsem
Documentation/locking/rwsem-design.txt       ← (نسخ قديمة) وصف التصميم الداخلي
Documentation/locking/mutex-design.rst       ← للمقارنة مع الـ mutex
```

**الـ source files المباشرة:**

```
include/linux/rwsem.h          ← الـ public API (الملف اللي بندرسه)
kernel/locking/rwsem.c         ← الـ core implementation
kernel/locking/rwbase_rt.c     ← الـ PREEMPT_RT variant
include/linux/rwbase_rt.h      ← الـ rwbase_rt struct definitions
include/linux/osq_lock.h       ← الـ MCS optimistic spin queue
```

---

### Commits مهمة في تاريخ الـ rwsem

| الـ Commit / الـ Patch Series | الوصف |
|-------------------------------|-------|
| [rwsem optimistic spinning — commit مايو 2014](https://lkml.iu.edu/hypermail/linux/kernel/1405.2/01676.html) | Commit ID: `746c198007a8` — إضافة الـ MCS-based optimistic spinning بواسطة Davidlohr Bueso |
| [Tim Chen rwsem performance patches v8 (2013)](https://lore.kernel.org/linux-mm/1380753493.11046.82.camel@schen9-DESK/) | أصل فكرة الـ optimistic spinning — 20-58% throughput improvement |
| [Rwsem rearchitecture part 1 — lore.kernel.org](https://lore.kernel.org/lkml/20190404174320.22416-1-longman@redhat.com/) | Waiman Long — 11 patches لإعادة الهيكلة |
| [Rwsem rearchitecture part 2 — lkml](https://lkml.org/lkml/2019/4/28/116) | Waiman Long — الجزء الثاني من الـ rearchitecture |

---

### نقاشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [Enable count-based spinning on reader](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1410847.html) | نقاش تقني حول الـ optimistic spinning على الـ reader-owned rwsem |
| [Time-based spinning on reader-owned rwsem](https://lore.kernel.org/patchwork/patch/1067564/) | التحول من count-based لـ time-based spinning |
| [Remove reader optimistic spinning](https://lists-ec2.96boards.org/archives/list/linux-stable-mirror@lists.linaro.org/message/EW7DCBTKDLHZMIAYLQGTB3QCKR2WYIQ2/) | نقاش إزالة الـ reader spinning في نسخ معينة |
| [Rwsem rearchitecture part 2 — patch v2](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1973148.html) | Thread كامل لنقاش الـ rearchitecture |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 5: Concurrency and Race Conditions**
- المواضيع: `semaphore`, `down()`, `up()`, الفرق بين الـ reader/writer locks
- متاح مجانًا: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 9: An Introduction to Kernel Synchronization**
- **الفصل 10: Kernel Synchronization Methods**
- بيشرح الـ `rw_semaphore` بالتفصيل مع أمثلة استخدام حقيقية
- الـ ISBN: 978-0672329463

#### Mastering Linux Kernel Development
- [فصل الـ Reader-Writer Semaphores على O'Reilly](https://www.oreilly.com/library/view/mastering-linux-kernel/9781785883057/71eacd64-1999-496a-a992-5fb7513bdbf8.xhtml)
- تغطية شاملة للـ rwsem API مع الـ PREEMPT_RT variant

#### Linux Kernel Programming — Kaiwan Billimoria
- **الفصل 15: Working with Kernel Synchronization Primitives**
- أمثلة عملية على استخدام الـ rwsem في drivers

---

### kernelnewbies.org — تحسينات الـ rwsem عبر الإصدارات

| إصدار الكيرنل | التحسين |
|---------------|---------|
| [Linux 3.9](https://kernelnewbies.org/Linux_3.10) | Opportunistic lock stealing في الـ slow path |
| [Linux 3.10](https://kernelnewbies.org/Linux_3.10) | تعميم الـ lock stealing على الـ fast path — تحسين أداء ضخم |
| [Linux 5.2](https://kernelnewbies.org/Linux_5.2) | توحيد الـ rwsem implementation وتبسيط الـ micro-optimizations |
| [Linux 6.17](https://kernelnewbies.org/Linux_6.17) | توسيع الـ hung task blocker tracking ليشمل الـ rwsem |
| [Linux 6.18](https://kernelnewbies.org/Linux_6.18) | استبدال الـ global percpu_rwsem بـ per-threadgroup في `cgroup.procs` |
| [KernelGlossary](https://kernelnewbies.org/KernelGlossary) | تعريف مصطلح rwsem في قاموس الكيرنل |
| [InternalKernelDataTypes](https://kernelnewbies.org/InternalKernelDataTypes) | قائمة بـ data types الكيرنل الداخلية بما فيها الـ rw_semaphore |

---

### elinux.org

| الرابط | المحتوى |
|--------|---------|
| [R-Car Debugging kernel options — elinux.org](https://elinux.org/R-Car/Debugging-kernel-options-causes-performance-down) | تأثير تفعيل `CONFIG_DEBUG_RWSEMS` على الأداء في الـ embedded systems |

---

### مصادر إضافية على الإنترنت

| المصدر | الرابط |
|--------|--------|
| linux-insides (0xax) — Reader/Writer semaphores | [https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-5.html](https://0xax.gitbooks.io/linux-insides/content/SyncPrim/linux-sync-5.html) |
| GitHub — kernel/locking/rwsem.c (الكود الحالي) | [https://github.com/torvalds/linux/blob/master/kernel/locking/rwsem.c](https://github.com/torvalds/linux/blob/master/kernel/locking/rwsem.c) |
| GitHub — include/linux/rwsem.h (الـ header الحالي) | [https://github.com/torvalds/linux/blob/master/include/linux/rwsem.h](https://github.com/torvalds/linux/blob/master/include/linux/rwsem.h) |
| الـ kernel docs الرسمية — percpu-rw-semaphore | [https://docs.kernel.org/locking/percpu-rw-semaphore.html](https://docs.kernel.org/locking/percpu-rw-semaphore.html) |
| Oracle Blog — rwsem optimistic spinning | [https://blogs.oracle.com/linux/new-concepts-in-scalability-and-performance](https://blogs.oracle.com/linux/new-concepts-in-scalability-and-performance) |
| TuxThink Blog — rwsem basics | [https://tuxthink.blogspot.com/2011/05/reader-writer-semaphores-in-linux.html](https://tuxthink.blogspot.com/2011/05/reader-writer-semaphores-in-linux.html) |

---

### Search Terms للبحث عن مزيد من المعلومات

```
"rw_semaphore" site:lore.kernel.org
"struct rw_semaphore" site:github.com/torvalds/linux
lwn.net rwsem scalability
"down_read" "down_write" linux kernel locking
RWSEM_WRITER_LOCKED atomic_long count
mmap_sem rwsem performance
"optimistic spin queue" osq_lock rwsem
rwsem PREEMPT_RT rwbase_rt
lockdep rwsem nested subclass
rwsem writer starvation fairness linux
"downgrade_write" rwsem use case
rwsem cacheline bouncing owner field
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على `down_write` — أشهر وأخطر entry point في الـ rwsem API. كل ما حد في الـ kernel بياخد write lock، الـ kprobe بتتفعل وبنطبع معلومات عن الـ semaphore والـ task اللي طلب الـ lock.

ده مفيد عملياً لأن `down_write` بيتوقف لو في readers تانيين، يعني ممكن نشوف مين اللي بيسبب contention في الـ locking.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * rwsem_probe.c — kprobe on down_write() to trace write-lock acquisitions
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/rwsem.h>       /* struct rw_semaphore, rwsem_is_contended */
#include <linux/sched.h>       /* current, task_pid_nr, task_comm_len */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc-AR");
MODULE_DESCRIPTION("kprobe on down_write() — trace rwsem write-lock requests");

/* ------------------------------------------------------------------ */
/* Pre-handler: called just BEFORE down_write() executes              */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64, the first argument (sem) lands in RDI.
     * regs_get_kernel_argument(regs, 0) هو الطريقة الصح
     * للوصول لأول argument بشكل portable.
     */
    struct rw_semaphore *sem =
        (struct rw_semaphore *)regs_get_kernel_argument(regs, 0);

    if (!sem)
        return 0;

    /*
     * rwsem_is_contended() بترجع true لو في tasks
     * شايلة الـ lock أو staning في wait_list.
     * ده يخلينا نعرف هل down_write هيتأخر أو لأ.
     */
    pr_info("rwsem_probe: [%s pid=%d] down_write(%px) count=0x%lx contended=%d\n",
            current->comm,
            task_pid_nr(current),
            sem,
            atomic_long_read(&sem->count),
            rwsem_is_contended(sem));

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* Post-handler: called just AFTER down_write() returns               */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * بعد ما down_write رجعت، الـ task بقى owner فعلي للـ write lock.
     * نطبع تأكيد إن الـ lock اتاخد.
     */
    pr_info("rwsem_probe: [%s pid=%d] acquired write lock\n",
            current->comm,
            task_pid_nr(current));
}

/* ------------------------------------------------------------------ */
/* kprobe struct definition                                           */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name    = "down_write",   /* الـ function اللي هنعمل probe عليها */
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init rwsem_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بتحط breakpoint وهتفعل الـ handler
     * في كل مرة kernel بيدخل down_write.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("rwsem_probe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("rwsem_probe: planted kprobe at %px (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit rwsem_probe_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتشال من الذاكرة،
     * لأن لو في call جاية لـ down_write وهي بتعدي على handler_pre،
     * هتلاقي الكود اتمسح من الذاكرة → kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("rwsem_probe: kprobe removed\n");
}

module_init(rwsem_probe_init);
module_exit(rwsem_probe_exit);
```

---

### Makefile

```makefile
obj-m += rwsem_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### بناء وتشغيل

```bash
# Build
make

# Load
sudo insmod rwsem_probe.ko

# شوف الـ output في dmesg
sudo dmesg -w | grep rwsem_probe

# Unload
sudo rmmod rwsem_probe
```

---

### شرح كل جزء

#### `#include <linux/kprobes.h>`
الـ kprobe API كله جوّاه: `struct kprobe`، `register_kprobe`، `unregister_kprobe`، و`regs_get_kernel_argument`. من غيره مفيش طريقة نعمل dynamic instrumentation.

#### `#include <linux/rwsem.h>`
محتاجينه عشان نعرف شكل الـ `struct rw_semaphore` ونقدر نقرأ `count` و`wait_list` منها بشكل type-safe. بنستخدم `rwsem_is_contended()` عشان نشوف هل في tasks بتستنى.

#### `#include <linux/sched.h>`
بيجيب تعريف `current` (الـ macro اللي بيرجع `struct task_struct *` للـ task الحالي)، و`task_pid_nr()`. ده بيخلينا نعرف مين اللي طلب الـ write lock.

#### `handler_pre` — الـ callback قبل الـ function

**الـ args:**
- `struct kprobe *p` — pointer للـ kprobe نفسه (مش محتاجينه هنا).
- `struct pt_regs *regs` — الـ CPU registers وقت الـ probe. منه بناخد الـ argument الأول (`sem`) عن طريق `regs_get_kernel_argument(regs, 0)`.

الـ `return 0` معناه "اشتغل الـ original function عادي". لو رجعنا قيمة ≠ 0، الـ kprobe هيعمل single-step للـ instruction وممكن يسبب مشاكل.

بنطبع:
- `comm` — اسم الـ process (max 16 byte).
- `pid` — الـ PID.
- `sem` — عنوان الـ rwsem في الذاكرة (مفيد للـ debugging لو في أكتر من sem).
- `count` — الـ atomic counter. لو bit 0 مضروب = writer موجود. لو عدد كبير positive = readers.
- `contended` — هل في حد بيستنى؟ لو 1 معناه الـ down_write هتبلوك.

#### `handler_post` — الـ callback بعد الـ function

بيتفعل بعد ما `down_write` ترجع بنجاح (يعني الـ task بقى owner). بيطبع تأكيد إن الـ lock اتاخد. في الـ real world ممكن تقيس الوقت بين `pre` و`post` عشان تشوف قد إيه استنى الـ writer.

#### `struct kprobe kp`

- **`symbol_name = "down_write"`** — الـ kernel بيعمل lookup في الـ kallsyms ويحط software breakpoint (int3 على x86) على أول instruction للـ function.
- **`pre_handler` و`post_handler`** — الـ callbacks المسجّلة.

#### `module_init` / `register_kprobe`

`register_kprobe` بتعمل:
1. Lookup لـ `down_write` في الـ kallsyms.
2. تحط `int3` breakpoint على الـ instruction الأولى.
3. تسجّل الـ handlers في الـ kprobe table.

لو فشلت (مثلاً الـ function مش exported أو محميّة بـ `__kprobes`)، بترجع error code سالب.

#### `module_exit` / `unregister_kprobe`

لازم نعمل unregister قبل ما الـ module code يتشال من الذاكرة. لو تركنا الـ breakpoint، أي call لـ `down_write` هتطلع handler في عنوان اتمسح → kernel panic. ده pattern أساسي في كل module بيعمل instrumentation.

---

### مثال على الـ output في dmesg

```
[ 1234.567890] rwsem_probe: planted kprobe at ffffffffc0123456 (down_write)
[ 1234.601234] rwsem_probe: [kworker/0:1 pid=42] down_write(ffff888100abc000) count=0x0 contended=0
[ 1234.601250] rwsem_probe: [kworker/0:1 pid=42] acquired write lock
[ 1234.712345] rwsem_probe: [bash pid=1337] down_write(ffff888100abc000) count=0x1 contended=1
[ 1234.850000] rwsem_probe: [bash pid=1337] acquired write lock
```

الـ line التانية بتظهر إن `bash` طلب write lock وكان في contention (count=0x1 يعني bit 0 مضروب = writer كان شايله قبليه) — ده بالظبط اللي الـ `rwsem_is_contended()` بيكشفه عن طريق الـ `wait_list`.
