# `list.h`

> **PATH**: `include/linux/list.h`
> **Subsystem**: Core Data Structures — Linked List Infrastructure
> **الوظيفة الأساسية**: تعريف الـ doubly linked list الدائرية (circular doubly linked list) اللي يستخدمها كل الـ kernel تقريبًا لربط الـ data structures ببعض

---

### المشكلة اللي بتحلها

تخيل إنك شغال في مطبخ كبير (الـ kernel)، وعندك مئات الطباخين (الـ subsystems) كل واحد محتاج يحتفظ بقوائم — قائمة الـ processes، قائمة الـ devices، قائمة الـ file descriptors، قائمة الـ timers، إلخ.

كل واحد لو عمل قائمته الخاصة من الصفر، هيبقى عندنا مئات نسخ من نفس الكود — كل نسخة فيها bugs مختلفة، وكل واحد بيحتاج maintenance منفصل. الحل؟ نعمل **قائمة واحدة عالمية** تشتغل مع أي نوع data.

هذا بالضبط ما يفعله الـ `list.h`.

---

### الفكرة العبقرية: الـ List داخل الـ Struct مش خارجه

في لغات زي Java، اللي بتعمل linked list بتحط الـ data جوه الـ node:

```
[ data | next_ptr ] → [ data | next_ptr ] → NULL
```

الـ kernel بيعمل العكس تمامًا — بيحط الـ list node جوه الـ data:

```c
struct process {
    int pid;
    char name[64];
    struct list_head list;   /* ← الـ list node مدفون جوه الـ struct */
};
```

يعني الـ list مش بتعرف حاجة عن الـ process — هي بس بتربط nodes ببعض. وبعدين لما تيجي تاخد الـ node وترجع للـ struct الأصلي، بتستخدم الـ magic macro: **`container_of`**.

```
list_head → container_of → struct process
```

الجمال هنا إن **نفس الكود** بيشتغل مع أي struct في الـ kernel — processes، files، devices، buffers، كل حاجة.

---

### شكل الـ List في الذاكرة

الـ list دايرية (circular) — يعني آخر عنصر بيشاور على الأول، والـ head نفسه جزء من الدايرة:

```
         ┌─────────────────────────────────────────┐
         ↓                                         │
      [HEAD] ⇄ [node1] ⇄ [node2] ⇄ [node3] ⇄ ──┘
         ↑_________________________________________↑
```

لما الـ list فاضية:
```
      [HEAD]
       next ──→ HEAD (نفسه)
       prev ──→ HEAD (نفسه)
```

ده بيخلي كل العمليات بسيطة ومتسقة — مفيش حاجة اسمها NULL pointer تتحقق منها في كل عملية.

---

### العمليات الأساسية

الـ `list.h` بيوفر كل اللي محتاجه:

| العملية | الـ Function |
|--------|-------------|
| إضافة في البداية | `list_add()` |
| إضافة في النهاية | `list_add_tail()` |
| حذف node | `list_del()` |
| نقل node لـ list تاني | `list_move()`, `list_move_tail()` |
| دمج listين | `list_splice()`, `list_splice_tail()` |
| تقطيع الـ list | `list_cut_position()` |
| المرور على كل العناصر | `list_for_each_entry()` |

---

### الـ Traversal — الـ Macro الأهم

المرور على كل عناصر الـ list بيبقى كده:

```c
struct process *p;
list_for_each_entry(p, &process_list, list) {
    /* p بيشاور على كل process بالترتيب */
    printk("Process: %s\n", p->name);
}
```

الـ macro بيترجم لـ:

```c
for (p = list_first_entry(&process_list, struct process, list);
     !list_entry_is_head(p, &process_list, list);
     p = list_next_entry(p, list))
```

وفيه نسخة **safe** للحذف أثناء المرور:

```c
struct process *p, *tmp;
list_for_each_entry_safe(p, tmp, &process_list, list) {
    if (p->pid == 0)
        list_del(&p->list);  /* آمن لأن tmp بيحفظ الـ next */
}
```

---

### الـ hlist — نوع تاني للـ Hash Tables

غير الـ circular doubly linked list العادية، الـ `list.h` بيحتوي على **`hlist`** (hash list) — وده list مخصص للـ hash tables.

الفرق؟ في الـ hash table عندك آلاف الـ buckets، ولو كل bucket اتأخذ 2 pointers (زي الـ list_head العادي) هيبقى في waste كبير في الذاكرة. الـ `hlist_head` بياخد pointer واحد بس (`first`)، وبتوفر نص حجم الـ buckets.

```
hlist_head:    [ first ]
                   ↓
hlist_node: [ next | **pprev ]
                ↓
hlist_node: [ next | **pprev ]
                ↓
              NULL
```

الـ `pprev` هنا عبقري — بدل ما يشاور على الـ node اللي قبله، بيشاور على الـ `next` pointer الموجود في الـ node اللي قبله. ده بيخلي الـ deletion بسيطة من غير ما تعرف الـ head.

---

### الحماية من الـ Bugs: `CONFIG_LIST_HARDENED`

لما تشتغل بـ `CONFIG_LIST_HARDENED`، كل عملية إضافة أو حذف بتتحقق إن الـ pointers مش اتفسدت:

```c
static __always_inline bool __list_add_valid(...)
{
    /* التحقق إن prev→next == next و next→prev == prev */
    if (likely(next->prev == prev && prev->next == next && ...))
        return true;
    /* لو فيه فساد، report warning */
    ret &= __list_add_valid_or_report(new, prev, next);
    return ret;
}
```

وبعد الـ `list_del()` بيتم تسميم الـ pointers بـ **`LIST_POISON1`** و **`LIST_POISON2`** — قيم غريبة تسبب kernel panic فورًا لو حاول حد يـaccess الـ node المحذوف. ده بيكشف الـ use-after-free bugs بسرعة.

---

### الـ Memory Barriers: `WRITE_ONCE` و `READ_ONCE`

الـ compiler أحيانًا بيعيد ترتيب الـ instructions لتحسين الـ performance. في الـ single-threaded code ده مش مشكلة، لكن في kernel متعدد الـ processors (SMP) ممكن thread تاني يشوف الـ list في حالة inconsistent.

عشان كده كل الـ pointer assignments بتتم بـ `WRITE_ONCE()` و `READ_ONCE()` — عشان:
1. الـ compiler لا يعيد ترتيب الـ writes
2. كل processor يشوف آخر قيمة مكتوبة

---

### مين بيستخدم الـ `list.h`؟

الإجابة: **كل حاجة تقريبًا في الـ kernel**. من أمثلة:

- **Scheduler**: قائمة الـ runnable processes في كل CPU
- **VFS**: قائمة الـ open files لكل process
- **Network**: قوائم الـ socket buffers (sk_buff)
- **Memory Management**: قوائم الـ memory zones والـ pages
- **Device Model**: قائمة الـ devices المتصلة بكل bus
- **Timers**: قوائم الـ pending timers

---

### الملفات المكوِّنة لهذا الـ Subsystem

#### الملف الأساسي
| الملف | الدور |
|-------|-------|
| `include/linux/list.h` | التعريفات والـ implementations كاملة |

#### الـ Headers اللي يعتمد عليها
| الملف | الدور |
|-------|-------|
| `include/linux/types.h` | تعريف `struct list_head` و `struct hlist_head` و `struct hlist_node` |
| `include/linux/container_of.h` | الـ `container_of()` macro — القلب اللي بيربط الـ list node بالـ struct الأب |
| `include/linux/poison.h` | `LIST_POISON1` و `LIST_POISON2` — قيم التسميم بعد الحذف |
| `include/linux/const.h` | helper macros للـ constants |
| `include/asm/barrier.h` | `READ_ONCE`, `WRITE_ONCE`, `smp_store_release`, `smp_load_acquire` |

#### ملفات مرتبطة في نفس العائلة
| الملف | الدور |
|-------|-------|
| `include/linux/llist.h` | Lock-free linked list للـ high-performance scenarios |
| `include/linux/rculist.h` | نفس الـ list API لكن مع دعم الـ RCU (Read-Copy-Update) للـ lockless access |
| `include/linux/rculist_bl.h` | RCU + bit-locked list للـ hash tables |
| `lib/list_sort.c` | خوارزمية merge sort للـ list (لأن الـ list.h نفسه مفيش فيه sort) |
| `include/linux/list_sort.h` | الـ header للـ list_sort |

#### أمثلة استخدام في الـ kernel نفسه
| الملف | بيستخدم الـ list لـ |
|-------|---------------------|
| `include/linux/sched.h` | `struct task_struct` — كل process عنده `list_head` في قائمة الـ processes |
| `include/linux/skbuff.h` | `struct sk_buff` — network packets |
| `include/linux/fs.h` | `struct file` — open files |
| `include/linux/timer.h` | `struct timer_list` — kernel timers |
| `include/linux/workqueue.h` | `struct work_struct` — deferred work |

---

## Phase 2: شرح الـ Linked List Framework

---

### المشكلة — ليش الـ kernel يحتاج linked list خاصة فيه؟

أي نظام تشغيل حقيقي بيحتاج يتعامل مع **قوائم من الأشياء** في كل مكان:

- قائمة الـ processes الشغّالة
- قائمة الـ network packets المنتظرة
- قائمة الـ drivers المتاحة
- قائمة الـ timers المجدولة
- قائمة الـ file descriptors المفتوحة

في لغة عادية زي Java، اللي بتستخدم `LinkedList<T>` — الـ list نفسها بتحتفظ بـ pointer على الـ object. يعني الـ list هي اللي "تعرف" عن الـ objects، مش العكس.

**المشكلة في الـ kernel**: هذا النهج يعمل مشكلتين:

**أولاً — مشكلة الـ generics**: الـ C ما عندها templates. لو صنعت `linked_list` تشتغل مع أي نوع، لازم تستخدم `void*`. وهذا يعني:

```c
// الطريقة "العادية" — خطيرة وبطيئة
struct generic_list_node {
    void *data;           // pointer لأي شيء
    struct generic_list_node *next;
    struct generic_list_node *prev;
};
```

المشكلة: كل مرة تضيف item للقائمة، لازم تعمل **allocation ثاني** — واحد للـ node نفسها، وواحد للـ data. ده بطيء، وبيعمل fragmentation في الـ memory، وخطر على systems زي الـ kernel اللي فيها memory allocations ممكن تفشل.

**ثانياً — مشكلة الـ overhead**: في الـ kernel، كل microsecond مهم. Allocation ثاني = overhead مضاعف.

---

### الحل — الـ Intrusive Linked List

الـ kernel اخترع فكرة بسيطة لكنها **عبقرية**:

> بدل ما الـ list تحمل الـ data، خليّ الـ data تحمل الـ list node جوّاها.

```c
// الطريقة العادية (non-intrusive):
//  [list_node] --> [data object]   (كيانين منفصلين)

// طريقة الـ kernel (intrusive):
//  [data object [ list_node ] ]   (الـ node جوّا الـ object نفسه)
```

يعني بدل ما تخزّن `void *data` في الـ node، خليّ الـ struct الأصلي يحمل `struct list_head` جوّاه:

```c
// الـ struct الأصلي — مثلاً process
struct task_struct {
    pid_t pid;
    char name[16];
    // ... مئات الـ fields ...

    struct list_head tasks;    // <-- هنا الـ "hook" للقائمة
    struct list_head children; // ممكن أكثر من قائمة!
    struct list_head siblings;
};
```

الـ `struct list_head` نفسها بسيطة جداً:

```c
// من include/linux/types.h
struct list_head {
    struct list_head *next, *prev;
};
```

---

### التشبيه — سلسلة مفاتيح

تخيّل كل **process** هو شخص في مبنى. الطريقة العادية زي لو عندك سكرتير يحتفظ بقائمة "اسم الشخص، مكتبه" — قائمة منفصلة عن الأشخاص أنفسهم.

الطريقة الـ intrusive زي لو **كل شخص يلبس badge** فيه حلقة من معدن، وتربطهم ببعض بـ chain مباشرة — بدون قائمة خارجية. لما تمشي على الـ chain توصل للأشخاص مباشرة.

الإضافة والإزالة سريعة لأنك تمسك الشخص مباشرة وتفصل/توصل الحلقة.

---

### الـ `container_of` — السحر الحقيقي

طيب، لو أنا عندي pointer على الـ `list_head` جوّا الـ `task_struct`، كيف أرجع للـ `task_struct` الأصلي؟

هنا يجي الـ macro الأشهر في الـ kernel:

```c
// من include/linux/container_of.h
#define container_of(ptr, type, member) ({              \
    void *__mptr = (void *)(ptr);                       \
    ((type *)(__mptr - offsetof(type, member))); })
```

**كيف يشتغل بالضبط؟**

تخيّل الـ memory layout لـ `task_struct`:

```
عنوان X:     [ pid           ] ← بداية struct task_struct
عنوان X+4:   [ name          ]
عنوان X+20:  [ ...           ]
عنوان X+100: [ tasks.next    ] ← هنا الـ list_head
عنوان X+108: [ tasks.prev    ]
عنوان X+116: [ children.next ]
...
```

لو عندي `ptr` يشير على `tasks` (عنوان X+100)، وأنا أعرف إن `tasks` موجودة على offset 100 من بداية الـ struct:

```
بداية الـ struct = ptr - offset
                 = (X+100) - 100
                 = X
```

الـ `offsetof(type, member)` بيحسب هذا الـ offset تلقائياً وقت الـ compile.

```c
// مثال عملي
struct list_head *pos; // pointer على list_head جوّا task_struct

// بدون container_of — مستحيل:
// struct task_struct *task = ???

// مع container_of — سهل:
struct task_struct *task = container_of(pos, struct task_struct, tasks);
```

**لماذا هذا عبقري؟** لأنه:
1. Zero cost — بيتحوّل لعملية طرح بسيطة وقت الـ compile
2. Type safe — الـ compiler يتحقق إن الـ member موجود في الـ type
3. عام — يشتغل مع أي struct وأي member

---

### الـ Circular Doubly Linked List — البنية الكاملة

الـ list في الـ kernel مش مجرد doubly linked — هي **دائرية**. وفيها مفهوم مهم جداً: **الـ sentinel node** (أو الـ head node).

#### الـ Sentinel / Head Node

الـ `LIST_HEAD` مش عنصر حقيقي في القائمة — هو نقطة بداية فارغة تربط القائمة بنفسها:

```c
LIST_HEAD(my_list);
// = struct list_head my_list = { &my_list, &my_list }
```

قائمة فارغة:
```
  ┌──────────────────────┐
  │                      │
  ▼                      │
┌──────────┐             │
│ my_list  │─────────────┘
│ (head)   │◄────────────┐
└──────────┘             │
  │                      │
  └──────────────────────┘

  next و prev كلاهما يشيران على نفس الـ head
```

لما نضيف element واحد بـ `list_add_tail`:
```
  ┌───────────────────────────────────────┐
  │                                       │
  ▼                                       │
┌──────────┐   next    ┌──────────────┐   │
│ my_list  │──────────►│ task_struct  │   │
│ (head)   │◄──────────│  list_head   │───┘
└──────────┘   prev    └──────────────┘
```

#### ليش دائري؟

**فائدة رقم 1 — الإضافة في الطرفين O(1)**:

- `list_add(new, head)` — يضيف بعد الـ head مباشرة (في البداية)
- `list_add_tail(new, head)` — يضيف قبل الـ head (في النهاية)

بالدائرية، `head->prev` يشير دائماً على **آخر عنصر** بدون ما تحتفظ بـ tail pointer منفصل.

**فائدة رقم 2 — الحذف O(1) بدون معرفة الـ head**.

**فائدة رقم 3 — iteration بسيط** — الدائرية هي اللي تعرّفه "متى ينتهي".

---

### `list_head` vs `hlist_head` — متى تستخدم أيهما؟

```
┌─────────────────┬──────────────────────┬──────────────────────┐
│ الخاصية        │ list_head            │ hlist_head           │
├─────────────────┼──────────────────────┼──────────────────────┤
│ حجم الـ head   │ 16 bytes             │ 8 bytes              │
│ إضافة للبداية │ O(1)                 │ O(1)                 │
│ إضافة للنهاية │ O(1)                 │ O(n) — لا prev!      │
│ حذف            │ O(1)                 │ O(1) بحيلة pprev    │
│ الاستخدام      │ قوائم عامة           │ hash table buckets   │
│ الدائرية       │ نعم                  │ لا                   │
└─────────────────┴──────────────────────┴──────────────────────┘
```

---

### الـ Memory Barriers — `WRITE_ONCE` و `READ_ONCE`

الـ CPU الحديث، من أجل الأداء، يعمل **out-of-order execution**. يعني ممكن يكتب في الذاكرة بترتيب مختلف عن اللي كتبته في الكود.

**`WRITE_ONCE(x, val)`** يعمل شيئين:
1. يضمن إن الكتابة تحصل **مرة واحدة وكاملة** — مش partial write
2. يمنع الـ compiler من إعادة ترتيب هذه الكتابة أو حذفها كـ "optimization"

**`READ_ONCE(x)`** نفس الفكرة للقراءة — يضمن إن القراءة تحصل مرة واحدة من الذاكرة الفعلية.

#### `smp_store_release` و `smp_load_acquire`

للـ list operations الأكثر خطورة، الـ kernel يستخدم barriers أقوى تضمن **happens-before relationship** الكامل بين الـ CPUs.

---

### الـ Hardening — `CONFIG_LIST_HARDENED` و `CONFIG_DEBUG_LIST`

#### `LIST_POISON1` و `LIST_POISON2`

لما تحذف عنصر من القائمة:

```c
static inline void list_del(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->next = LIST_POISON1;   // 0xdead000000000100
    entry->prev = LIST_POISON2;   // 0xdead000000000200
}
```

**POISON values** مضمون إنها تقع في منطقة ذاكرة مش mapped — أي وصول إليها سيعطي immediate kernel panic. كمان القيمة `DEAD000` واضحة جداً في debugging.

---

### البنية الكاملة — كيف كل شيء مرتبط ببعض

```
include/linux/list.h
        │
        ├── struct list_head (من types.h)
        │   next + prev → دائرية مزدوجة
        │
        ├── container_of (من container_of.h)
        │   ptr - offsetof = pointer على الـ parent struct
        │
        ├── LIST_POISON1/2 (من poison.h)
        │   قيم سحرية لتسميم الـ pointers بعد الحذف
        │
        ├── WRITE_ONCE/READ_ONCE (من barrier.h)
        │   ضمان atomic reads/writes
        │
        ├── list_add / list_add_tail
        │   إضافة للبداية أو النهاية O(1)
        │
        ├── list_del / list_del_init
        │   حذف من القائمة O(1)
        │
        ├── list_for_each_entry
        │   iteration على كل العناصر
        │
        └── hlist_head / hlist_node
            نسخة مضغوطة للـ hash tables (8 bytes بدل 16)
```

---

## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags — Cheatsheet

| الخيار / القيمة | النوع | القيمة / التأثير | متى تستخدمه |
|-----------------|-------|-----------------|-------------|
| `CONFIG_LIST_HARDENED` | Kconfig bool | عند التفعيل: يفعّل `__list_valid_slowpath()` قبل كل add/del | kernel مُقوَّى ضد corruption |
| `CONFIG_DEBUG_LIST` | Kconfig bool | عند التفعيل: يُحوّل الـ inline checks لـ function calls تطبع رسائل تفصيلية | debugging وتطوير |
| `LIST_POISON1` | Poison value | `0xdead000000000100` (x86_64) | يُكتب في `next` بعد الحذف |
| `LIST_POISON2` | Poison value | `0xdead000000000200` (x86_64) | يُكتب في `prev` بعد الحذف |
| `WRITE_ONCE(x, v)` | Barrier macro | يمنع الـ compiler من تحسين الكتابة بطريقة تكسر concurrency | كل كتابة لـ next/prev في paths الحساسة |
| `READ_ONCE(x)` | Barrier macro | يمنع الـ compiler من cache-ing القراءة | كل قراءة في paths الـ concurrent |
| `smp_store_release()` | Memory barrier | full release barrier | `list_del_init_careful` وما شابهها |
| `smp_load_acquire()` | Memory barrier | full acquire barrier | `list_empty_careful` في RCU contexts |

---

### الـ Structs الأساسية

#### `struct list_head`

**الغرض:** هو الوحدة الأساسية للقائمة الدائرية المزدوجة. لا يحمل بيانات — يُدمَج داخل أي struct تبغاه ليجعله جزءاً من قائمة.

```c
/* من include/linux/types.h */
struct list_head {
    struct list_head *next;   /* المؤشر للعنصر التالي */
    struct list_head *prev;   /* المؤشر للعنصر السابق */
};
```

| الحقل | النوع | الغرض |
|-------|-------|--------|
| `next` | `struct list_head *` | يشير للعقدة التالية في القائمة. في القائمة الفارغة يشير للـ head نفسه |
| `prev` | `struct list_head *` | يشير للعقدة السابقة. في القائمة الفارغة يشير للـ head نفسه |

---

#### `struct hlist_head`

**الغرض:** رأس قائمة مُحسَّنة لجداول الـ hash. عنده مؤشر واحد بس (بدل اثنين في `list_head`) — يوفر ذاكرة ضرورية عندما يكون عندك آلاف الـ buckets.

```c
/* من include/linux/types.h */
struct hlist_head {
    struct hlist_node *first;   /* مؤشر للعقدة الأولى، أو NULL */
};
```

---

#### `struct hlist_node`

**الغرض:** عقدة قائمة مُحسَّنة للـ hash tables. يستخدم خدعة الـ `pprev` الذكية.

```c
/* من include/linux/types.h */
struct hlist_node {
    struct hlist_node  *next;    /* المؤشر للعقدة التالية */
    struct hlist_node **pprev;   /* مؤشر لـ مؤشر — الخدعة الذكية */
};
```

---

### خدعة الـ `pprev` — الشرح الكامل

```
السيناريو الأول: العقدة الأولى في القائمة
                      ┌─────────────────────────────────┐
  hlist_head          │                                 │
  ┌─────────┐         │  hlist_node (العقدة الأولى)    │
  │ first ──┼────────▶│  ┌──────┬──────────────────┐   │
  └─────────┘         │  │ next │ pprev ────────────┼───┘
  ▲                   │  └──────┴──────────────────┘   │
  │                   └─────────────────────────────────┘
  │
  pprev يشير لـ &head->first

السيناريو الثاني: العقدة الثانية في القائمة
  hlist_head         node1                node2
  ┌───────┐         ┌──────┬──────┐       ┌──────┬──────┐
  │first ─┼────────▶│next ─┼──────┼──────▶│next  │pprev─┼──────┐
  └───────┘         │pprev │      │       │      │      │      │
                    └──────┴──────┘       └──────┴──────┘      │
                        ▲                                       │
                        └───────────────────────────────────────┘
  pprev يشير لـ &node1->next
```

```c
/* حذف عقدة — بدون أي if/else */
static inline void __hlist_del(struct hlist_node *n)
{
    struct hlist_node *next = n->next;
    struct hlist_node **pprev = n->pprev;

    /* سواء pprev يشير لـ head->first أو node->next، نفس السطر يشتغل */
    WRITE_ONCE(*pprev, next);
    if (next)
        WRITE_ONCE(next->pprev, pprev);
}
```

النتيجة: كود حذف موحّد O(1) بدون أي حالة خاصة للـ head.

---

### رسم علاقات الـ Structs — ASCII Art

```
         ┌─────────────────────────────────────────────────────────────────────┐
         │                   الـ Circular Doubly Linked List                   │
         │                                                                      │
         │   ┌──────────────┐  next   ┌──────────────┐  next   ┌────────────┐ │
         │   │  list_head   │◀───────▶│  list_head   │◀───────▶│ list_head  │ │
         │   │  (HEAD/sentinel)│ prev  │  (node 1)    │  prev   │  (node 2)  │ │
         │   └──────┬───────┘         └──────┬───────┘         └─────┬──────┘ │
         │          │                         │                        │        │
         │    embedded in                embedded in            embedded in     │
         │          │                         │                        │        │
         │   ┌──────▼───────┐         ┌──────▼───────┐         ┌─────▼──────┐ │
         │   │  task_struct  │         │  task_struct  │         │task_struct │ │
         │   │  pid, mm, ..  │         │  pid, mm, ..  │         │ pid, mm,.. │ │
         │   └──────────────┘         └──────────────┘         └────────────┘ │
         │                                                                      │
         │   ↑ container_of(ptr, struct task_struct, tasks) يوصلك هنا          │
         └──────────────────────────────────────────────────────────────────────┘
```

---

### رسم دورة الحياة — Lifecycle Diagram

```
════════════════════════════════════════════════════════════
             دورة حياة list_head
════════════════════════════════════════════════════════════

  1. INIT (التهيئة)
  ─────────────────
  Static:   LIST_HEAD(my_list)
            → my_list.next = &my_list
            → my_list.prev = &my_list

  Dynamic:  INIT_LIST_HEAD(&head)
            → head->next = head
            → head->prev = head

  2. INSERT (الإدراج)
  ────────────────────
  list_add(&node->list, &head)        ← يضيف بعد الـ head (stack/LIFO)
  list_add_tail(&node->list, &head)   ← يضيف قبل الـ head (queue/FIFO)

  3. TRAVERSE (التنقل)
  ──────────────────────
  list_for_each_entry(pos, &head, member) { use(pos->data); }

  4. DELETE (الحذف)
  ──────────────────
  list_del(&node->list)
  → node->next = LIST_POISON1  (0xdead...100)
  → node->prev = LIST_POISON2  (0xdead...200)

  list_del_init(&node->list)
  → يحذف ويُعيد تهيئة العقدة (next=self, prev=self)

  5. CHECK (الفحص)
  ──────────────────
  list_empty(&head)    → head->next == head ؟
  list_is_last(node, head) → node->next == head ؟

════════════════════════════════════════════════════════════
```

---

### رسم تدفق الاستدعاءات

#### `list_add()` — الإضافة

```
list_add(&new_node->list, &head)
    │
    ▼
__list_add(new, head, head->next)
    │
    ├─── [CONFIG_LIST_HARDENED فعّال؟]
    │         ▼ نعم
    │    __list_add_valid_or_report(new, prev, next)
    │
    ▼
  الربط الفعلي:
    next->prev = new;
    new->next  = next;
    new->prev  = prev;
    WRITE_ONCE(prev->next, new);
```

#### `list_del()` — الحذف

```
list_del(&node->list)
    │
    ▼
__list_del_entry(entry)
    │
    ├─── [CONFIG_LIST_HARDENED فعّال؟]
    │         ▼ نعم
    │    __list_del_entry_valid_or_report(entry)
    │
    ▼
__list_del(entry->prev, entry->next)
    │
    ├── next->prev = prev;
    └── WRITE_ONCE(prev->next, next);

    ▼
entry->next = LIST_POISON1;
entry->prev = LIST_POISON2;
```

#### `container_of()` — الخدعة الرياضية

```
container_of(ptr, struct task_struct, tasks)

الخطوة 1: احسب offsetof(struct task_struct, tasks) = مثلاً 1024 byte
الخطوة 2: task_struct_ptr = (struct task_struct *)((char *)ptr - 1024)

  ┌───────────────────────────────────┐
  │ 0x... task_struct بداية           │ ◀─── النتيجة
  │   pid_t pid                        │
  │   ...                              │
  │ 0x...+1024 tasks.next ─────────────┼──── ptr (الذي عندنا)
  └───────────────────────────────────┘
```

---

### استراتيجية الـ Locking

**الحقيقة الصريحة: `list.h` لا يحتوي أي locking.**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         أساليب الـ Locking مع list.h                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. spinlock_t  ─── الأسرع لقوائم قصيرة العمليات                           │
│  2. mutex       ─── للعمليات الطويلة التي يمكن أن تنام                      │
│  3. RCU         ─── للقوائم كثيرة القراءة نادرة الكتابة                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

| الحالة | الـ Lock المناسب |
|---------|-----------------|
| قائمة تُعدَّل من interrupt handler | `spinlock_t` + `spin_lock_irqsave()` |
| قائمة تُقرأ كثيراً وتُكتب نادراً | `RCU` + `rculist.h` |
| عمليات قد تنام (kmalloc، I/O) | `mutex` |
| per-CPU list | لا يحتاج lock |

---

## Phase 4: دليل الـ Debugging الشامل

---

### المقدمة — ليش debugging القوائم صعب؟

تخيل إنك عندك سلسلة حلقات معدنية وحلقة واحدة منها اتكسرت. المشكلة مش هنا — المشكلة إن الانهيار يصير بعد كذا حلقة لما تحاول تسحب السلسلة. هكذا بالضبط تفسد الـ linked list في الـ kernel: العقدة تتلف في مكان، والـ crash يصير في مكان ثانٍ تماماً.

---

### جدول الـ Config Options

| Config | الوضع | ما يفعله | متى تفعّله |
|--------|-------|----------|-----------|
| `CONFIG_DEBUG_LIST` | Slow-path | يفعّل checks كاملة مع `WARN()` لكل `list_add` و `list_del` | التطوير الأولي، تشخيص الـ corruption |
| `CONFIG_LIST_HARDENED` | Fast inline | يُضيف checks مُضمَّنة (inline) في `list_add` / `list_del` | بيئات الإنتاج حيث الأمان أهم من الأداء |
| `CONFIG_KASAN` | Instrumentation | يكشف **use-after-free** على عقد القوائم | تشخيص UAF وdouble-free |
| `CONFIG_KCSAN` | Instrumentation | يكشف **data races** في الوصول المتزامن للقوائم | تشخيص race conditions في SMP |
| `CONFIG_PROVE_LOCKING` | Analysis | الـ **lockdep** — يرصد lock ordering violations | تشخيص locking bugs |
| `CONFIG_KMEMLEAK` | Runtime scan | يكشف memory leaks في الـ objects المُدارة بـ linked lists | تشخيص leak في قوائم طويلة الأمد |

```bash
# تفعيل من .config
echo "CONFIG_DEBUG_LIST=y" >> .config
echo "CONFIG_LIST_HARDENED=y" >> .config
echo "CONFIG_KASAN=y" >> .config
echo "CONFIG_KCSAN=y" >> .config
echo "CONFIG_PROVE_LOCKING=y" >> .config
echo "CONFIG_KMEMLEAK=y" >> .config
make olddefconfig
```

---

### الـ ftrace والـ Kprobes

```bash
# === تفعيل ftrace على دوال الـ list corruption ===
cd /sys/kernel/debug/tracing

echo function > current_tracer
echo "__list_add_valid_or_report
__list_del_entry_valid_or_report" > set_ftrace_filter
echo 1 > options/func_stack_trace   # يطبع الـ stack
echo 1 > tracing_on
# ... شغّل الكود ...
cat trace
```

#### استخدام الـ Kprobes

```bash
# kprobe على __list_add_valid_or_report مع طباعة الـ arguments
echo 'p:list_add_corrupt __list_add_valid_or_report new=%di prev=%si next=%dx' \
    > kprobe_events
echo 1 > events/kprobes/list_add_corrupt/enable
echo 1 > tracing_on
```

---

### الـ Dynamic Debug

```bash
# فعّل debug messages
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control
# مع stack trace
echo 'module my_driver +ps' > /sys/kernel/debug/dynamic_debug/control
```

---

### نقاط استراتيجية لـ WARN_ON و dump_stack

```c
/* 1. قبل list_del — تحقق إن العقدة مضافة فعلاً */
WARN_ON(list_empty(&entry->list));
list_del(&entry->list);

/* 2. كشف double-delete */
if (entry->list.next == LIST_POISON1) {
    pr_err("DOUBLE DELETE detected! entry=%p\n", entry);
    dump_stack();
    return;
}

/* 3. تحقق من Lock */
WARN_ON(!spin_is_locked(&my_list_lock));
```

---

### جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | السبب المحتمل | الحل |
|-----------------|--------|---------------|------|
| `list_add corruption. next->prev should be prev...` | الـ `next->prev` يحمل poison value | **use-after-free**: حذفت عقدة ثم أضفت بجانبها | فعّل KASAN |
| `list_del corruption. prev->next should be entry...` | الـ pointer الأمامي لا يشير للعقدة الحالية | **memory corruption** أو race condition | فعّل KCSAN |
| `list_add double add: new=... prev=... next=...` | إضافة عقدة موجودة أصلاً في قائمة | نسيان `list_del` قبل `list_add` | تحقق: `WARN_ON(!list_empty(&entry->list))` |
| `BUG: KASAN: use-after-free in list_add` | وصول لعقدة بعد تحريرها | UAF: حذفت الـ object قبل إزالتها من القائمة | دائماً `list_del` قبل `kfree` |

---

### الـ crash utility — تحليل الـ Kernel Dumps

```bash
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/vmcore

# فحص قائمة الـ processes
crash> list task_struct.tasks -s task_struct.pid,comm -H init_task.tasks

# فحص قائمة يدوياً
crash> struct list_head 0xffff888012345678

# لو وجدت LIST_POISON — هنا المشكلة
# 0xdead000000000100 = LIST_POISON1 = use-after-free
```

---

### الـ kmemleak — كشف Memory Leaks

```bash
echo scan > /sys/kernel/debug/kmemleak
sleep 5
cat /sys/kernel/debug/kmemleak
# المخرجات تظهر الـ objects غير المُحررة مع call stack من أين خُصصت
```

---

### أوامر سريعة للمرجع

```bash
# فحص الـ list corruption في dmesg
dmesg | grep -E "list_(add|del) corruption|double add"

# فحص KASAN errors
dmesg | grep -B5 -A30 "KASAN.*list"

# فحص lockdep warnings
dmesg | grep -A20 "possible circular locking"

# كشف kmemleak
echo scan > /sys/kernel/debug/kmemleak && sleep 5 && \
    cat /sys/kernel/debug/kmemleak | head -50

# فحص KCSAN races
dmesg | grep "KCSAN: data-race"
```

---

## Phase 5: شرح الـ Functions

---

### جدول Summary — كل الـ Functions دفعة وحدة

| Function | Type | Purpose |
|----------|------|---------|
| `LIST_HEAD_INIT(name)` | macro | تهيئة `list_head` ليشير على نفسه |
| `LIST_HEAD(name)` | macro | تعريف وتهيئة `list_head` في سطر واحد |
| `INIT_LIST_HEAD(list)` | static inline | تهيئة `list_head` في runtime |
| `HLIST_HEAD_INIT` | macro | تهيئة `hlist_head` لـ NULL |
| `HLIST_HEAD(name)` | macro | تعريف وتهيئة `hlist_head` |
| `INIT_HLIST_HEAD(ptr)` | macro | تهيئة `hlist_head` في runtime |
| `INIT_HLIST_NODE(h)` | static inline | تهيئة `hlist_node` (next=NULL, pprev=NULL) |
| `__list_add_valid()` | static inline | يتحقق من سلامة القائمة قبل الإضافة |
| `__list_add_valid_or_report()` | bool | يكتشف ويُبلغ عن تلف القائمة عند الإضافة |
| `__list_del_entry_valid()` | static inline | يتحقق من سلامة القائمة قبل الحذف |
| `__list_del_entry_valid_or_report()` | bool | يكتشف ويُبلغ عن تلف القائمة عند الحذف |
| `__list_add()` | static inline | إضافة داخلية بين `prev` و `next` |
| `list_add()` | static inline | إضافة بعد الـ head (LIFO / stack) |
| `list_add_tail()` | static inline | إضافة قبل الـ head (FIFO / queue) |
| `__list_del()` | static inline | حذف داخلي بإعادة ربط `prev` و `next` |
| `__list_del_clearprev()` | static inline | حذف داخلي مع ضبط `prev = NULL` |
| `__list_del_entry()` | static inline | يتحقق ثم يحذف entry |
| `list_del()` | static inline | حذف العقدة وتسميم مؤشراتها |
| `list_del_init()` | static inline | حذف وإعادة تهيئة العقدة (قابلة لإعادة الاستخدام) |
| `list_del_init_careful()` | static inline | مثل `list_del_init` مع release semantics |
| `list_replace()` | static inline | يستبدل `old` بـ `new` |
| `list_replace_init()` | static inline | يستبدل `old` بـ `new` ويهيئ `old` |
| `list_swap()` | static inline | يبدل موضع عقدتين |
| `list_move()` | static inline | ينقل عقدة لأول القائمة الجديدة |
| `list_move_tail()` | static inline | ينقل عقدة لآخر القائمة الجديدة |
| `list_bulk_move_tail()` | static inline | ينقل نطاق من العقد لآخر القائمة |
| `list_is_first()` | static inline | هل العقدة هي الأولى؟ |
| `list_is_last()` | static inline | هل العقدة هي الأخيرة؟ |
| `list_is_head()` | static inline | هل المؤشر هو الـ head نفسه؟ |
| `list_empty()` | static inline | هل القائمة فارغة؟ |
| `list_empty_careful()` | static inline | هل القائمة فارغة؟ (مع acquire semantics) |
| `list_is_singular()` | static inline | هل فيها عنصر واحد بالضبط؟ |
| `list_count_nodes()` | static inline | يعد العقد (O(n)) |
| `list_rotate_left()` | static inline | يحرك أول عقدة للنهاية |
| `list_rotate_to_front()` | static inline | يدور القائمة حتى تصير عقدة معينة في الأول |
| `list_cut_position()` | static inline | يقطع القائمة حتى entry (شامل) |
| `list_cut_before()` | static inline | يقطع القائمة حتى entry (غير شامل) |
| `list_splice()` | static inline | يدمج قائمة بعد الـ head |
| `list_splice_tail()` | static inline | يدمج قائمة قبل الـ head |
| `list_splice_init()` | static inline | يدمج بعد الـ head ويهيئ المصدر |
| `list_splice_tail_init()` | static inline | يدمج قبل الـ head ويهيئ المصدر |
| `list_entry()` | macro | يرجع الـ struct المحتوي على `list_head` |
| `list_first_entry()` | macro | أول struct في القائمة |
| `list_last_entry()` | macro | آخر struct في القائمة |
| `list_first_entry_or_null()` | macro | أول struct أو NULL إذا فارغة |
| `list_last_entry_or_null()` | macro | آخر struct أو NULL إذا فارغة |
| `list_next_entry()` | macro | الـ struct التالي |
| `list_prev_entry()` | macro | الـ struct السابق |
| `list_for_each()` | macro | تكرار للأمام على `list_head` |
| `list_for_each_prev()` | macro | تكرار للخلف |
| `list_for_each_safe()` | macro | تكرار آمن (يسمح بالحذف أثناء الـ loop) |
| `list_for_each_entry()` | macro | تكرار على الـ structs مباشرة |
| `list_for_each_entry_reverse()` | macro | تكرار عكسي على الـ structs |
| `list_for_each_entry_safe()` | macro | تكرار آمن على الـ structs |
| `list_safe_reset_next()` | macro | يعيد ضبط `n` بعد إعادة أخذ الـ lock |
| `hlist_unhashed()` | static inline | هل الـ node محذوف؟ |
| `hlist_empty()` | static inline | هل الـ hlist_head فارغ؟ |
| `hlist_del()` | static inline | حذف وتسميم مؤشرات |
| `hlist_del_init()` | static inline | حذف وإعادة تهيئة |
| `hlist_add_head()` | static inline | إضافة في رأس الـ hlist |
| `hlist_add_before()` | static inline | إضافة قبل عقدة معينة |
| `hlist_add_behind()` | static inline | إضافة بعد عقدة معينة |
| `hlist_move_list()` | static inline | ينقل كامل الـ hlist لـ head جديد |
| `hlist_for_each_entry()` | macro | تكرار على الـ structs في hlist |
| `hlist_for_each_entry_safe()` | macro | تكرار آمن على الـ structs |
| `hlist_count_nodes()` | static inline | يعد العقد (O(n)) |

---

### المجموعة الأولى: Initialization

#### `LIST_HEAD_INIT(name)`
```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```
يهيئ `list_head` في وقت الـ compile كجزء من تعريف متغير. يجعل الـ `next` والـ `prev` يشيران على نفس المتغير.

#### `LIST_HEAD(name)`
```c
#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)
```
يجمع التعريف والتهيئة في سطر واحد.

```c
static LIST_HEAD(module_list);   /* قائمة كل الـ modules المحملة */
```

#### `INIT_LIST_HEAD(list)`
```c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
    WRITE_ONCE(list->next, list);
    list->prev = list;
}
```
يهيئ `list_head` في **runtime**. يستخدم `WRITE_ONCE` للـ `next` لضمان correctness في بيئات concurrent.

```c
struct my_device *dev = kmalloc(sizeof(*dev), GFP_KERNEL);
INIT_LIST_HEAD(&dev->children);
INIT_LIST_HEAD(&dev->resources);
```

---

### المجموعة الثانية: List Add

#### `list_add(new, head)` — إضافة Stack-style
```c
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}
```
تُضيف `new` مباشرة **بعد** الـ `head`. **سلوك LIFO** — آخر ما تضيفه هو أول ما تراه.

```
قبل:  [HEAD] ──▶ [A] ──▶ [B]
بعد:  [HEAD] ──▶ [new] ──▶ [A] ──▶ [B]
```

#### `list_add_tail(new, head)` — إضافة Queue-style
```c
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}
```
تُضيف `new` **قبل** الـ `head` — في نهاية القائمة. **سلوك FIFO**.

```c
/* queue انتظار الـ packets */
list_add_tail(&pkt->list, &rx_queue);
```

---

### المجموعة الثالثة: List Delete

#### `list_del(entry)` — الحذف الكامل
```c
static inline void list_del(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->next = LIST_POISON1;
    entry->prev = LIST_POISON2;
}
```
يحذف `entry` من قائمته ثم **يُسمِّم** مؤشراتها. أي محاولة للوصول بعد الحذف = kernel oops.

#### `list_del_init(entry)` — حذف مع إعادة تهيئة
```c
static inline void list_del_init(struct list_head *entry)
{
    __list_del_entry(entry);
    INIT_LIST_HEAD(entry);
}
```
يحذف ثم يُعيد تهيئة العقدة — يمكن إضافتها لقائمة أخرى أو التحقق منها بـ `list_empty`.

| | `list_del` | `list_del_init` |
|--|--|--|
| بعد الحذف | مؤشرات مسمومة | مؤشرات لنفسها |
| `list_empty(entry)` | undefined behavior | يُرجع `true` |
| إعادة الاستخدام | غير آمن | آمن |

#### `list_del_init_careful(entry)` — حذف مع Release Semantics
```c
static inline void list_del_init_careful(struct list_head *entry)
{
    __list_del_entry(entry);
    WRITE_ONCE(entry->next, entry);
    smp_store_release(&entry->prev, entry);
}
```
مثل `list_del_init` لكن يستخدم `smp_store_release` — للتنسيق مع `list_empty_careful()` في بيئات lock-free.

---

### المجموعة الرابعة: List Modification

#### `list_replace(old, new)` / `list_replace_init(old, new)`
يضع `new` في نفس مكان `old` في القائمة. النسخة `_init` تُعيد تهيئة `old` بعدها.

#### `list_swap(entry1, entry2)`
يبدل موضع `entry1` و `entry2` في القائمة. يتعامل بذكاء مع حالة التجاور.

#### `list_move(list, head)` / `list_move_tail(list, head)`
يحذف `list` من قائمتها الحالية ويضيفها لأول / آخر القائمة الجديدة.

```c
list_move(&item->lru, &lru_head);       /* LRU promotion */
list_move_tail(&item->lru, &lru_head);  /* LRU demotion */
```

#### `list_bulk_move_tail(head, first, last)`
ينقل **سلسلة متصلة** من العقد (من `first` حتى `last`) لنهاية قائمة `head`. عملية O(1) مهما كانت عدد العقد المنقولة.

---

### المجموعة الخامسة: List Query

#### `list_empty(head)`
```c
static inline int list_empty(const struct list_head *head)
{
    return READ_ONCE(head->next) == head;
}
```
يتحقق إن القائمة فارغة باستخدام `READ_ONCE` لمنع compiler optimization.

#### `list_empty_careful(head)`
نسخة أقوى تستخدم `smp_load_acquire` وتتحقق من كلا المؤشرين. تعمل **بالتوافق** مع `list_del_init_careful`.

#### `list_is_singular(head)`
```c
return !list_empty(head) && (head->next == head->prev);
```
يتحقق إن القائمة تحتوي على **عقدة واحدة بالضبط**.

#### `list_count_nodes(head)`
يعد عقد القائمة — **O(n)**. استخدم باعتدال.

---

### المجموعة السادسة: Rotation / Split / Join

#### `list_cut_position(list, head, entry)`
يقطع من `head` حتى `entry` **شاملاً** وينقل لـ `list`.

```
قبل:  [head] ──▶ [A] ──▶ [B] ──▶ [C] ──▶ [D] ──▶ [E]
بعد (cut at C):
[list]: [A] ──▶ [B] ──▶ [C]
[head]: [D] ──▶ [E]
```

**استخدام نموذجي:** batch processing — خذ أول N عناصر بدون lock طويل.

#### `list_splice_init(list, head)` / `list_splice_tail_init(list, head)`
يدمج `list` في `head` ويُعيد تهيئة `list`. نمط شائع جداً:

```c
LIST_HEAD(local);
spin_lock(&global_lock);
list_splice_init(&global_queue, &local);  /* انقل الكل لـ local */
spin_unlock(&global_lock);

/* معالجة local بدون lock */
list_for_each_entry_safe(item, tmp, &local, node)
    process(item);
```

---

### المجموعة السابعة: Iteration Macros

#### `list_entry(ptr, type, member)`
```c
#define list_entry(ptr, type, member) container_of(ptr, type, member)
```
يُحوِّل مؤشر `list_head` لمؤشر للـ struct المحتوي. هذا **جوهر** كل ماكرو تكرار آخر.

#### `list_for_each_entry(pos, head, member)` — الأكثر استخداماً
```c
#define list_for_each_entry(pos, head, member)                        \
    for (pos = list_first_entry(head, typeof(*pos), member);          \
         !list_entry_is_head(pos, head, member);                      \
         pos = list_next_entry(pos, member))
```
`pos` يُصبح مباشرة مؤشراً للـ struct — لا حاجة لـ `list_entry` في كل iteration.

```c
struct packet *pkt;
list_for_each_entry(pkt, &packet_list, list) {
    process(pkt);   /* pkt جاهز مباشرة */
}
```

#### `list_for_each_entry_safe(pos, n, head, member)` — الأكثر أماناً
يمر على الـ structs مباشرة ويسمح بحذف `pos` أثناء الـ loop.

```c
struct packet *pkt, *tmp;
list_for_each_entry_safe(pkt, tmp, &rx_queue, list) {
    if (pkt->expired) {
        list_del(&pkt->list);
        kfree(pkt);
    }
}
```

**تحذير:** `_safe` يسمح فقط بحذف **العنصر الحالي** — ليس عناصر أخرى أو إضافة عناصر.

#### `list_safe_reset_next(pos, n, member)`
يُعيد ضبط `n` بعد ما أُعيد أخذ الـ lock. لما تُفلت الـ lock وسط الـ loop وتأخذه ثانيةً، القائمة قد تتغير.

---

### المجموعة الثامنة: HList Functions

#### `hlist_add_head(n, h)` — إضافة في البداية
```c
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
    struct hlist_node *first = h->first;
    WRITE_ONCE(n->next, first);
    if (first)
        WRITE_ONCE(first->pprev, &n->next);
    WRITE_ONCE(h->first, n);
    WRITE_ONCE(n->pprev, &h->first);
}
```
يُضيف `n` في **بداية** الـ hlist مع تحديث جميع الـ `pprev` pointers.

#### `hlist_for_each_entry(pos, head, member)` — الأكثر استخداماً
```c
struct cache_entry *entry;
unsigned int bucket = hash_32(key, HASH_BITS);
hlist_for_each_entry(entry, &hash_table[bucket], hash) {
    if (entry->key == key)
        return entry->value;
}
return -ENOENT;
```

---

### ملخص أنماط الاستخدام

```
الاستخدام                    الدالة المناسبة
─────────────────────────────────────────────────────────────────
قائمة global static          LIST_HEAD(name)
قائمة داخل struct (runtime)  INIT_LIST_HEAD(&s->list)
إضافة (LIFO / أول القائمة)   list_add(&node, &head)
إضافة (FIFO / آخر القائمة)  list_add_tail(&node, &head)
حذف نهائي (kfree بعده)       list_del(&node)
حذف مع إعادة استخدام         list_del_init(&node)
نقل بين قائمتين              list_move() / list_move_tail()
دمج قائمتين                  list_splice_init()
تقطيع للمعالجة batch          list_cut_position()
تكرار عادي                   list_for_each_entry(pos, head, member)
تكرار مع حذف                 list_for_each_entry_safe(pos, n, head, member)
hash table: إضافة             hlist_add_head(&node, &bucket)
hash table: بحث               hlist_for_each_entry(pos, &bucket, member)
hash table: حذف مع إعادة     hlist_del_init(&node)
```

---

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Kernel Panic غامض على STM32MP1 Industrial Gateway

#### العنوان
**double `list_del()` يُفجّر الـ gateway بعد 10 دقائق من تشغيله**

#### السياق
مصنع في ألمانيا يستخدم **STM32MP1** كـ industrial gateway لجمع بيانات من عشرات الـ sensors عبر I2C و Modbus. الـ gateway يعمل 24/7 وفجأة بدأ يـcrash بعد فترة عشوائية.

#### المشكلة
الـ driver يحتفظ بقائمة من الـ `pending_msg` structs، لكن مسار الـ timeout handler يستدعي كلاً من `i2c_msg_complete()` و `i2c_msg_error()` — وكلاهما يستدعي `list_del()` على نفس العقدة.

**بدون** `CONFIG_LIST_HARDENED`: الـ kernel يكتب على عنوان `0x00000100` (LIST_POISON1) — ذاكرة غير صالحة. الـ panic يأتي بعد 10 دقائق في مكان مختلف تماماً.

**مع** `CONFIG_LIST_HARDENED`: يكتشف المشكلة فوراً:
```
list_del corruption. prev->next should be ffff8000123abc00
but was 0000000000000100. (prev=ffff8000deadbeef)
```

#### الحل
```c
/* استخدم list_del_init بدل list_del */
static void i2c_msg_complete(struct pending_msg *msg)
{
    list_del_init(&msg->node);  /* آمن للاستدعاء مرتين */
    process_msg(msg);
    kfree(msg);
}

static void i2c_msg_error(struct pending_msg *msg)
{
    log_error(msg->id);
    if (!list_empty(&msg->node))   /* تحقق قبل الحذف */
        list_del_init(&msg->node);
    kfree(msg);
}
```

#### الدرس المستفاد
- **`list_del_init()`** هو الخيار الآمن في أي كود يحتمل double-deletion
- **`CONFIG_LIST_HARDENED`** إجباري في بيئات الـ production الـ embedded
- الـ industrial gateway لا يُقبل فيه crash — فعّل كل guards الـ kernel من اليوم الأول

---

### السيناريو الثاني: Race Condition على RK3562 Android TV Box

#### العنوان
**`list_for_each_entry()` في main thread + interrupt handler = crash متقطع في I2C driver**

#### السياق
شركة تُنتج **Android TV boxes** مبنية على **RK3562**. الـ driver يستخدم `list_for_each_entry()` في main thread بينما الـ interrupt handler يحذف entries من نفس القائمة.

#### المشكلة
```
CPU0 (main thread):          CPU1 (interrupt):
─────────────────            ──────────────────
pos = txn_A
يقرأ txn_A->node.next
= &txn_B                      txn_A يُحذف من القائمة
                              txn_A->node.next = LIST_POISON1
يقرأ txn_B->node.next ← يقرأ من عنوان مسموم
= LIST_POISON1 (0x100)!
CRASH
```

#### الحل
```c
static void i2c_process_pending(void)
{
    struct i2c_txn *txn, *tmp;
    unsigned long flags;

    spin_lock_irqsave(&txn_lock, flags);  /* أوقف الـ interrupts */
    list_for_each_entry_safe(txn, tmp, &pending_txns, node) {
        if (txn->status == TXN_DONE) {
            list_del(&txn->node);
            kfree(txn);
        }
    }
    spin_unlock_irqrestore(&txn_lock, flags);
}
```

#### الدرس المستفاد
- **`list_for_each_entry()`** ليس thread-safe — فقط للـ single-threaded contexts أو تحت lock
- الـ `_safe` variant يحفظ الـ `next` pointer مسبقاً — لكن لا يُغني عن الـ locking في SMP
- على multi-core SoC مثل RK3562، `spin_lock_irqsave` ضروري لا اختياري

---

### السيناريو الثالث: Memory Leak على AM62x IoT Sensor Board

#### العنوان
**`kfree()` بدون `list_del_init()` يُراكم dangling pointers حتى تفيض الذاكرة**

#### السياق
شركة ناشئة تبني **IoT edge sensor** على **AM62x**. كل يوم تقريباً، الجهاز يتوقف عن الاستجابة بسبب `out of memory`.

#### المشكلة
```c
static int send_packet(struct packet_desc *pkt)
{
    int ret = ethernet_send(pkt->data, pkt->len);
    if (ret < 0) {
        kfree(pkt);   /* ⚠️ يحرر الذاكرة لكن ينسى list_del! */
        return ret;
    }
    list_del(&pkt->list);
    kfree(pkt);
    return 0;
}
```

الذاكرة تُحرر لكن `prev->list.next` لا يزال يشير إليها — dangling pointer. القائمة تنمو بلا حدود.

#### الحل
```c
static void packet_destroy(struct packet_desc *pkt)
{
    /* قاعدة ذهبية: list_del_init دائماً قبل kfree */
    if (!list_empty(&pkt->list))
        list_del_init(&pkt->list);
    kfree(pkt);
}
```

#### الدرس المستفاد
- **أي struct مضاف لقائمة** يجب أن يُحذف من القائمة قبل تحرير ذاكرته — قانون لا استثناء
- اكتب **cleanup function واحدة** تجمع `list_del_init` + `kfree`
- الـ IoT devices تعمل أشهراً بدون إعادة تشغيل — Memory leak صغير اليوم = crash مضمون الشهر القادم

---

### السيناريو الرابع: Hash Table بـ O(n) بدل O(1) على Allwinner H616

#### العنوان
**دالة hash سيئة تُكدس كل الـ inodes في bucket واحد — filesystem يتباطأ بعد ساعات**

#### السياق
فريق يبني **media center** مبني على **Allwinner H616**. بعد ساعات من الاستخدام، عمليات `stat()` و `open()` تصبح بطيئة من microseconds لـ milliseconds.

#### المشكلة
```c
static unsigned int inode_hash(unsigned long ino)
{
    return ino & (INODE_HASH_SIZE - 1);  /* ⚠️ hash سيئ */
}
```

الـ filesystem يُخصص الـ inode numbers تسلسلياً. الأرقام 1025 و 2049 و 3073 كلها تقع في bucket[1] — آلاف العقد في bucket واحد!

#### التشخيص بـ `hlist_count_nodes()`
```c
for (i = 0; i < INODE_HASH_SIZE; i++) {
    size_t count = hlist_count_nodes(&inode_cache[i]);
    if (count > 0)
        seq_printf(m, "bucket[%4d]: %zu nodes\n", i, count);
}
/* النتيجة: bucket[0]: 4892 nodes ← كارثة */
```

#### الحل
```c
#include <linux/hash.h>
static unsigned int inode_hash(unsigned long ino)
{
    return hash_long(ino, INODE_HASH_BITS);  /* Knuth multiplicative hash */
}
/* تحسّن: 600x أسرع */
```

#### الدرس المستفاد
- **`hlist_count_nodes()`** موجودة تحديداً لهذا النوع من الـ diagnostics
- الـ hash function الجيدة يجب أن تعمل بغض النظر عن **النمط** في الـ keys
- استخدم دوال الـ hash الموجودة في `linux/hash.h` — هي مُختبرة ومُثبتة

---

### السيناريو الخامس: إضافة عناصر أثناء `list_for_each_entry_safe` على i.MX8 Automotive ECU

#### العنوان
**`_safe` تعني فقط "آمن للحذف" — الإضافة أثناء الـ iteration تُعطي نتائج غير متوقعة**

#### السياق
فريق يطوّر **Automotive ECU** على **i.MX8**. المهندس يظن أن `_safe` تعني "آمن لأي تعديل على القائمة" ويُضيف عناصر retry أثناء التكرار.

#### المشكلة
إضافة في نهاية القائمة → ربما تُعالج العناصر الجديدة في نفس الـ loop → **infinite loop** إذا فشل الـ CAN bus دائماً.

#### الحل — فصل الـ retry queue
```c
static void process_messages(void)
{
    struct can_msg *msg, *tmp;
    LIST_HEAD(to_retry);  /* قائمة محلية مؤقتة */

    list_for_each_entry_safe(msg, tmp, &msg_queue, node) {
        int ret = can_transmit(msg);
        list_del(&msg->node);

        if (ret == -EBUSY && msg->retries < 3) {
            msg->retries++;
            list_add_tail(&msg->node, &to_retry);  /* قائمة منفصلة */
        } else {
            kfree(msg);
        }
    }

    /* بعد انتهاء الـ loop — ادمج الـ retries */
    list_splice(&to_retry, &msg_queue);
}
```

#### الدرس المستفاد
- **`list_for_each_entry_safe()`** صُمّم لحالة واحدة: **حذف** العنصر الحالي فقط
- **الإضافة** أثناء الـ iteration = undefined behavior (سواء في البداية أو النهاية)
- القاعدة الذهبية: **لا تُعدّل القائمة التي تمشي عليها** — استخدم قائمة مؤقتة ثم `list_splice`
- في الـ automotive software، كل سلوك يجب أن يكون متوقعاً ومحدداً

---

## Phase 7: مصادر ومراجع

---

### مقالات LWN.net الأساسية

| العنوان | الرابط | ما ستتعلمه |
|---|---|---|
| Toward a better list iterator for the kernel | [lwn.net/Articles/887097](https://lwn.net/Articles/887097/) | قصة `list_entry_is_head` وكيف أصلح الـ kernel bug |
| Linux kernel design patterns - part 2 | [lwn.net/Articles/336255](https://lwn.net/Articles/336255/) | نمط الـ intrusive list |
| Single linked lists for Linux | [lwn.net/Articles/10920](https://lwn.net/Articles/10920/) | النقاش الأصلي لإضافة `hlist` للـ kernel |
| Single linked lists for Linux, v2 | [lwn.net/Articles/10952](https://lwn.net/Articles/10952/) | النسخة الثانية بعد review |
| Linked list start and end | [lwn.net/Articles/908255](https://lwn.net/Articles/908255/) | نقاش حول سلوك الـ circular list |
| Using RCU for linked lists | [lwn.net/Articles/610972](https://lwn.net/Articles/610972/) | كيف تمشي على الـ list بدون lock |
| docs: document linked lists | [lwn.net/Articles/1028381](https://lwn.net/Articles/1028381/) | الـ patch الذي أضاف توثيق رسمي |
| What is RCU, Fundamentally? | [lwn.net/Articles/262464](https://lwn.net/Articles/262464/) | الأساس — لازم تفهم RCU عشان تفهم `list_for_each_entry_rcu` |
| Lockless patterns: relaxed access | [lwn.net/Articles/846700](https://lwn.net/Articles/846700/) | فهم `READ_ONCE` و `WRITE_ONCE` |

---

### التوثيق الرسمي للـ kernel

| المسار | المحتوى |
|---|---|
| [`Documentation/core-api/list.html`](https://docs.kernel.org/core-api/list.html) | التوثيق الرسمي للـ `list_head` API كاملاً |
| [`Documentation/core-api/kernel-api.rst`](https://docs.kernel.org/core-api/kernel-api.html) | الـ kernel API الشاملة |
| [`Documentation/RCU/listRCU.html`](https://docs.kernel.org/RCU/listRCU.html) | استخدام RCU لحماية الـ read-mostly linked lists |

---

### Commits مهمة في تاريخ الـ list

| الحدث | الإصدار | التفاصيل |
|---|---|---|
| إضافة `struct list_head` الأولى | 2.1.45 (1997) | قبل Git — في الـ historical archive |
| الـ initial Git commit للـ kernel | 2.6.12-rc2 | الـ commit: `1da177e4c3f4` — April 2005 |
| إضافة `list_entry_is_head` | 5.17 | fix لـ undefined behavior في الـ loop exit check |

```bash
# ابحث في تاريخ الـ git
git log --oneline -- include/linux/list.h | head -30
```

---

### الكتب المرجعية

#### Linux Device Drivers (LDD3)
**متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- Chapter 10: Interrupt Handling — يشرح الـ list في سياق الـ interrupt handlers

#### Linux Kernel Development — Robert Love
| الفصل | الموضوع |
|---|---|
| **Chapter 6** | Kernel Data Structures — هنا بيشرح الـ `list_head` من الأول للآخر |
| **Chapter 10** | Kernel Synchronization Methods — الـ locking مع الـ lists |

#### Embedded Linux Primer — Christopher Hallinan
- Chapter 13: Kernel Initialization — كيف الـ kernel يبني الـ lists الداخلية أثناء الـ boot

---

### مواقع تعليمية إضافية

| المصدر | الرابط | ما ستجد |
|---|---|---|
| KernelNewbies FAQ/LinkedLists | [kernelnewbies.org/FAQ/LinkedLists](https://kernelnewbies.org/FAQ/LinkedLists) | شرح بسيط مع أمثلة |
| Linux Insides (0xAX) | [0xax.gitbooks.io — DataStructures](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-1.html) | شرح ممتاز خطوة بخطوة بالكود |
| Shlomi Boutnaru — list_head | [medium.com — list_head journey](https://medium.com/@boutnaru/the-linux-kernel-data-strctures-journey-struct-list-head-87fa91a5ce1c) | رحلة تاريخية |
| Shlomi Boutnaru — hlist_head | [medium.com — hlist_head journey](https://medium.com/@boutnaru/the-linux-kernel-data-strcture-journey-struct-hlist-head-19f4ceb71295) | نفس الأسلوب للـ `hlist` |

---

### كلمات البحث المفيدة

```
# الأساسيات
linux kernel "list_head" tutorial
linux kernel "intrusive linked list" explained
"container_of" macro linux kernel explained

# الـ hash lists
linux kernel "hlist_head" "hlist_node" hash table
linux kernel "hlist" vs "list_head" difference

# الـ safety والـ debug
linux "CONFIG_DEBUG_LIST" list corruption detection
linux "CONFIG_LIST_HARDENED" hardened usercopy list
linux "LIST_POISON1" "LIST_POISON2" use-after-free

# الـ iteration
linux "list_for_each_entry_safe" vs "list_for_each_entry"
linux "list_entry_is_head" 5.17 change
```

---

### ملخص سريع للأولويات

```
1. include/linux/list.h                    ← الكود نفسه — الأهم
2. docs.kernel.org/core-api/list           ← التوثيق الرسمي
3. LDD3 Chapter 6                          ← Robert Love شارح كل حاجة
4. lwn.net/Articles/887097                 ← قصة list_entry_is_head
5. kernelnewbies.org/FAQ/LinkedLists       ← مراجعة سريعة
6. 0xax linux-insides DataStructures       ← شرح بالكود خطوة بخطوة
```

---

## Phase 8: Writing simple module

---

### الفكرة العامة — إيش بنسوي؟

الـ kernel يحتوي على ميزة اسمها **`CONFIG_LIST_HARDENED`** — لما تُفعَّل، كل عملية إضافة أو حذف من القائمة تُتحقق منها. لو الـ pointers خربانة، الـ kernel يستدعي:

- **`__list_add_valid_or_report`** — تُستدعى لما يُكتشف خطأ عند إضافة عنصر
- **`__list_del_entry_valid_or_report`** — تُستدعى لما يُكتشف خطأ عند حذف عنصر

هاتان الدالتان **غير static** — يعني نقدر نربط عليهم **kprobe** ونطلع notification كل مرة يحصل corruption حقيقي في أي مكان بالنظام.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * list_corruption_watchdog.c
 *
 * موديول يراقب أحداث list corruption في الـ kernel
 * عبر kprobes على __list_add_valid_or_report و __list_del_entry_valid_or_report
 */

#include <linux/module.h>      /* module_init, module_exit, MODULE_LICENSE */
#include <linux/kernel.h>      /* pr_info, pr_warn */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/list.h>        /* struct list_head */
#include <linux/sched.h>       /* current */
#include <linux/kallsyms.h>    /* sprint_symbol */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Linux in Arabic Project");
MODULE_DESCRIPTION("Watchdog for list corruption events via kprobes");
MODULE_VERSION("1.0");

/* ─────────────────────────────────────────────────────────────────────────
 * القسم الأول: kprobe على __list_add_valid_or_report
 *
 * signature: bool __list_add_valid_or_report(struct list_head *new,
 *                                            struct list_head *prev,
 *                                            struct list_head *next)
 * ───────────────────────────────────────────────────────────────────────── */

static int handler_add(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86_64، الـ arguments تُمرَّر في:
     *   di = arg0 (new),  si = arg1 (prev),  dx = arg2 (next)
     */
    struct list_head *new_node  = (struct list_head *)regs->di;
    struct list_head *prev_node = (struct list_head *)regs->si;
    struct list_head *next_node = (struct list_head *)regs->dx;

    pr_warn("LIST CORRUPTION WATCHDOG [ADD]\n");
    pr_warn("  process : %s (pid=%d)\n", current->comm, current->pid);
    pr_warn("  new     : %px  (next=%px, prev=%px)\n",
            new_node,
            new_node  ? new_node->next  : NULL,
            new_node  ? new_node->prev  : NULL);
    pr_warn("  prev    : %px  (next=%px, prev=%px)\n",
            prev_node,
            prev_node ? prev_node->next : NULL,
            prev_node ? prev_node->prev : NULL);
    pr_warn("  next    : %px  (next=%px, prev=%px)\n",
            next_node,
            next_node ? next_node->next : NULL,
            next_node ? next_node->prev : NULL);

    /* dump_stack(); */  /* فعّلها للـ debugging المعمق */

    return 0;
}

static struct kprobe kp_add = {
    .symbol_name = "__list_add_valid_or_report",
    .pre_handler = handler_add,
};

/* ─────────────────────────────────────────────────────────────────────────
 * القسم الثاني: kprobe على __list_del_entry_valid_or_report
 *
 * signature: bool __list_del_entry_valid_or_report(struct list_head *entry)
 * ───────────────────────────────────────────────────────────────────────── */

static int handler_del(struct kprobe *p, struct pt_regs *regs)
{
    struct list_head *entry = (struct list_head *)regs->di;

    pr_warn("LIST CORRUPTION WATCHDOG [DEL]\n");
    pr_warn("  process : %s (pid=%d)\n", current->comm, current->pid);
    pr_warn("  entry   : %px\n", entry);

    if (entry) {
        pr_warn("  entry->next : %px\n", entry->next);
        pr_warn("  entry->prev : %px\n", entry->prev);

        /*
         * لو الـ next أو prev يساوي LIST_POISON1/LIST_POISON2،
         * هذا يعني double-free (حذف العقدة مرتين)
         */
        if ((unsigned long)entry->next == (unsigned long)LIST_POISON1)
            pr_warn("  *** entry->next == LIST_POISON1 — use-after-free!\n");
        if ((unsigned long)entry->prev == (unsigned long)LIST_POISON2)
            pr_warn("  *** entry->prev == LIST_POISON2 — use-after-free!\n");
    }

    return 0;
}

static struct kprobe kp_del = {
    .symbol_name = "__list_del_entry_valid_or_report",
    .pre_handler = handler_del,
};

/* ─────────────────────────────────────────────────────────────────────────
 * module_init — نُسجِّل الـ kprobes عند تحميل الموديول
 * ───────────────────────────────────────────────────────────────────────── */

static int __init list_watchdog_init(void)
{
    int ret;

    /*
     * register_kprobe تضع breakpoint على أول instruction في الدالة المستهدفة.
     * لو فشل — غالباً CONFIG_LIST_HARDENED أو CONFIG_KPROBES غير مُفعَّل
     */
    ret = register_kprobe(&kp_add);
    if (ret < 0) {
        pr_err("list_watchdog: failed to register ADD kprobe (err=%d)\n", ret);
        pr_err("list_watchdog: هل CONFIG_LIST_HARDENED و CONFIG_KPROBES مُفعَّلان؟\n");
        return ret;
    }
    pr_info("list_watchdog: kprobe ADD registered at %px\n", kp_add.addr);

    ret = register_kprobe(&kp_del);
    if (ret < 0) {
        pr_err("list_watchdog: failed to register DEL kprobe (err=%d)\n", ret);
        unregister_kprobe(&kp_add);
        return ret;
    }
    pr_info("list_watchdog: kprobe DEL registered at %px\n", kp_del.addr);

    pr_info("list_watchdog: watchdog active — monitoring for list corruption\n");
    return 0;
}

/* ─────────────────────────────────────────────────────────────────────────
 * module_exit — نُلغي تسجيل الـ kprobes عند إزالة الموديول
 *
 * لازم نسوي هذا قبل إزالة الموديول — لو تركنا الـ breakpoint،
 * الـ kernel سيحاول يرجع لـ handler الذي اختفى = kernel panic
 * ───────────────────────────────────────────────────────────────────────── */

static void __exit list_watchdog_exit(void)
{
    unregister_kprobe(&kp_del);
    unregister_kprobe(&kp_add);
    pr_info("list_watchdog: watchdog removed\n");
}

module_init(list_watchdog_init);
module_exit(list_watchdog_exit);
```

---

### Makefile

```makefile
obj-m += list_corruption_watchdog.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتشغيل
make
sudo insmod list_corruption_watchdog.ko
sudo dmesg -w | grep -E "list_watchdog|LIST CORRUPTION"

# إزالة
sudo rmmod list_corruption_watchdog
```

---

### شرح كل قسم

#### الـ Includes — ليش كل واحد موجود؟

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | ضروري لأي kernel module — `module_init`، `module_exit`، `MODULE_LICENSE` |
| `<linux/kernel.h>` | يجلب `pr_info`، `pr_warn`، `pr_err` |
| `<linux/kprobes.h>` | قلب الموديول — `struct kprobe`، `register_kprobe` |
| `<linux/list.h>` | نحتاجه للـ cast لـ `struct list_head` وللوصول لـ `next` و `prev` |
| `<linux/sched.h>` | يجلب `current` — المتغير السحري للـ task الحالي |

---

#### الـ `pre_handler` و `struct pt_regs` — كيف نقرأ الـ arguments؟

```
x86_64 calling convention (System V ABI):
┌─────────┬──────────────────────────────┐
│ Register│ Argument                     │
├─────────┼──────────────────────────────┤
│ rdi     │ arg0 — new                   │
│ rsi     │ arg1 — prev                  │
│ rdx     │ arg2 — next                  │
└─────────┴──────────────────────────────┘
```

`regs->di` = قيمة الـ `rdi` register = الـ argument الأول. نـcast لـ `struct list_head *` ونقرأ `next` و `prev` مباشرة.

---

#### فحص `LIST_POISON1` و `LIST_POISON2`

لما الـ kernel يحذف عقدة بشكل صحيح، يكتب `LIST_POISON1` في `next` و `LIST_POISON2` في `prev`. رؤية هذه القيم عند محاولة حذف العقدة مرة ثانية = **double-free** الكلاسيكية.

---

#### لماذا `unregister_kprobe` في `module_exit` ضروري؟

```
التحميل:
insmod → module_init → register_kprobe × 2 → الـ breakpoints نشطة

الإزالة:
rmmod → module_exit → unregister_kprobe × 2 → الـ breakpoints تُزال → آمن
```

بدون `unregister_kprobe`، الـ breakpoints تبقى لكن الـ handler function تختفي. أول corruption = الـ CPU يقفز لعنوان لا يوجد فيه كود = **kernel panic فوري**.

---

### رسم توضيحي — كيف يعمل الـ kprobe؟

```
الوضع العادي:
caller code ──CALL──▶ __list_add_valid_or_report ──▶ MOV ... ──▶ RET

الوضع مع kprobe:
caller code ──CALL──▶ INT3 (kernel زرع هذا)
                           │
                           ▼
                      handler_add(regs)  ← كودنا يطبع التحذير
                           │
                           ▼
                      original instruction (مُستعاد)
                           │
                           ▼
                      CMP ... ──▶ RET
```

---

### كيف تختبره؟

```bash
# 1. تأكد إن CONFIG_LIST_HARDENED مُفعَّل
grep CONFIG_LIST_HARDENED /boot/config-$(uname -r)
# يجب أن تظهر: CONFIG_LIST_HARDENED=y

# 2. تأكد إن kprobes مدعوم
grep CONFIG_KPROBES /boot/config-$(uname -r)
# يجب أن تظهر: CONFIG_KPROBES=y

# 3. حمّل الموديول
sudo insmod list_corruption_watchdog.ko

# 4. تحقق من التسجيل
sudo dmesg | tail -5
# يجب أن ترى:
# list_watchdog: kprobe ADD registered at 0xffffffff...
# list_watchdog: kprobe DEL registered at 0xffffffff...
# list_watchdog: watchdog active

# 5. راقب الـ log
sudo dmesg -w
```

> **ملاحظة:** لو الكرنل بدون `CONFIG_LIST_HARDENED`، الدالتان غير موجودتان في الـ symbol table، وسيفشل `register_kprobe` بـ `-ENOENT`. فعّل الـ config option وأعد بناء الكرنل.
