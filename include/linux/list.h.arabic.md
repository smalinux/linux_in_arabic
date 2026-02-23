## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف وليه موجود؟

**الـ** `include/linux/list.h` هو أكتر ملف header اتستخدم في الـ Linux kernel على الإطلاق. ده الملف اللي بيعرّف **Circular Doubly Linked List** — هياكل البيانات الأساسية اللي الـ kernel بيبني عليها كل حاجة.

تقريباً كل subsystem في الـ kernel بيستخدم الـ `list.h` — من الـ scheduler، للـ networking، للـ filesystem، للـ device drivers.

---

### القصة: ليه محتاجين linked list في الـ kernel؟

تخيل عندك نظام تشغيل محتاج يتابع:
- كل الـ **processes** الشغالة دلوقتي (قائمة المهام)
- كل الـ **network sockets** المفتوحة
- كل الـ **devices** المتصلة بالجهاز
- كل الـ **timers** اللي هتشتغل
- كل الـ **modules** المحملة

كل دي قوائم — بتتضيف وبتتشال عناصر منها في كل لحظة، وعدد العناصر مش ثابت.

في C العادي، لو عايز تعمل linked list، بتحط pointer للـ `next` و `prev` جوه الـ struct نفسه:

```c
struct process {
    int pid;
    char name[16];
    struct process *next;  /* pointer للـ process الجاية */
    struct process *prev;  /* pointer للـ process السابقة */
};
```

**المشكلة؟** كل struct محتاج implementation منفصلة للـ add, delete, iterate. لو عندك 100 نوع struct، هتكتب نفس الكود 100 مرة.

---

### الحل العبقري في الـ Linux Kernel

بدل ما الـ list pointer يكون جزء من الـ data، الـ kernel **عكس الفكرة**:

```c
/* من include/linux/types.h — الـ struct الأساسي */
struct list_head {
    struct list_head *next, *prev;
};
```

وبعدين **تحط الـ `list_head` جوه الـ struct بتاعك**:

```c
struct process {
    int pid;
    char name[16];
    struct list_head list;  /* ده بس — مش pointers للـ process */
};
```

```
               ┌─────────────────────────────────────────────┐
               │         الـ list دايري (circular)           │
               └─────────────────────────────────────────────┘
                         ↓                        ↑
    ┌──────────────────────┐      ┌──────────────────────┐
    │   struct process     │      │   struct process     │
    │   pid=1              │      │   pid=2              │
    │   name="init"        │      │   name="bash"        │
    │  ┌────────────────┐  │      │  ┌────────────────┐  │
    │  │ list_head.next ├──┼─────►│  │ list_head.next ├──┼──► (head)
    │  │ list_head.prev │◄─┼──────┤  │ list_head.prev │  │
    │  └────────────────┘  │      │  └────────────────┘  │
    └──────────────────────┘      └──────────────────────┘
```

الـ list functions كلها بتشتغل على الـ `list_head` فقط — مش على الـ struct الكامل. وعشان ترجع للـ struct الأصلي من الـ `list_head`، بيستخدموا **الـ `container_of` macro** اللي يحسب الـ offset:

```c
/* بتديه pointer للـ list_head، وبيرجع pointer للـ struct الأصلي */
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
```

**الـ `container_of`** بيعمل إيه بالظبط؟ بياخد الـ pointer للـ `list_head` اللي جوه الـ struct، ويطرح منه الـ offset بتاعه عشان يوصل لأول الـ struct:

```c
/* إذا list_head.list عنده offset = 16 bytes من أول الـ struct */
/* container_of(ptr, struct process, list) = ptr - 16 */
```

---

### الـ Subsystem اللي ينتمي له الملف

الـ `list.h` جزء من **Linux Kernel Core Data Structures** — مش subsystem محددة بل هو **infrastructure أساسية** بتخدم كل الـ kernel.

الـ MAINTAINERS بيذكره تحت **LIST KUNIT TEST** (الـ `lib/tests/list-test.c`) بتاع David Gow.

---

### نوعين من الـ Lists في الملف

الملف بيعرّف نوعين:

| النوع | الاستخدام | الـ Head Size | الوصول للـ Tail |
|-------|-----------|--------------|----------------|
| **Doubly Linked List** (`list_head`) | الاستخدام العام | 2 pointers | O(1) |
| **Hash List** (`hlist_head` + `hlist_node`) | الـ hash tables | 1 pointer | O(n) |

**الـ `hlist`** مفيد في الـ hash tables لأن الـ bucket عادةً بيبقى عنده آلاف الـ entries، والـ 2 pointers في الـ head بيتضاعفوا في الذاكرة. الـ `hlist` بيوفر pointer واحد في الـ head.

---

### أهم الـ APIs

#### إنشاء وتهيئة List
```c
LIST_HEAD(my_list);          /* تعريف وتهيئة list في نفس الوقت */
INIT_LIST_HEAD(&my_list);    /* تهيئة list موجودة */
```

#### إضافة وحذف
```c
list_add(&entry->list, &head);       /* إضافة في الأول — مناسب للـ stack */
list_add_tail(&entry->list, &head);  /* إضافة في الآخر — مناسب للـ queue */
list_del(&entry->list);              /* حذف من الـ list */
list_del_init(&entry->list);         /* حذف وإعادة تهيئة */
list_move(&entry->list, &new_head);  /* نقل من list لـ list تانية */
```

#### الـ Iteration (الحلقات)
```c
/* iterate على الـ list_head فقط */
list_for_each(pos, &head);

/* iterate وترجع الـ struct الأصلي */
list_for_each_entry(pos, &head, member);

/* iterate آمن أثناء الحذف */
list_for_each_entry_safe(pos, n, &head, member);

/* iterate للخلف */
list_for_each_entry_reverse(pos, &head, member);
```

---

### مثال واقعي: الـ Scheduler

الـ Linux scheduler بيشيل الـ processes في run queues هي linked lists. لما process بتتخلق:

```c
/* من kernel/sched/core.c */
/* الـ task_struct بيحتوي على list_head للـ run queue */
struct task_struct {
    /* ... */
    struct list_head run_list;  /* للـ scheduling */
    /* ... */
};

/* إضافة process للـ run queue */
list_add_tail(&p->run_list, &rq->queue);

/* iterate على كل الـ processes في الـ queue */
list_for_each_entry(p, &rq->queue, run_list) {
    /* schedule الـ process */
}
```

---

### الـ Hardening والـ Security

الملف بيدعم وضعين:

- **`CONFIG_LIST_HARDENED`**: بيعمل validation بسيط inline عشان يكتشف الـ corruption بسرعة.
- **`CONFIG_DEBUG_LIST`**: بيعمل full validation مع reporting تفصيلي.

لما بتحذف node من الـ list، بيحط **poison values** (`LIST_POISON1 = 0x100`, `LIST_POISON2 = 0x122`) في الـ pointers عشان لو حد حاول يستخدم الـ node المحذوفة، يحصل fault فوراً بدل ما الـ kernel يكمل بسايلنس.

---

### الملفات المرتبطة اللي لازم تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/list.h` | الملف الأساسي — الـ doubly linked list + hlist |
| `include/linux/types.h` | تعريف `struct list_head`, `struct hlist_head`, `struct hlist_node` |
| `include/linux/container_of.h` | الـ `container_of` macro اللي بيربط الـ `list_head` بالـ struct الأصلي |
| `include/linux/poison.h` | قيم الـ `LIST_POISON1` و `LIST_POISON2` |
| `include/linux/list_bl.h` | Bit-Lock list — نفس فكرة الـ hlist بس بتستخدم bit للـ locking |
| `include/linux/list_lru.h` | LRU (Least Recently Used) list — لإدارة الـ cache |
| `include/linux/list_nulls.h` | hlist بس بـ NULL terminator خاص — للـ RCU |
| `include/linux/list_sort.h` | Merge sort للـ linked lists |
| `include/linux/list_private.h` | Helper functions داخلية |
| `lib/tests/list-test.c` | KUnit tests للـ list |

---

### ملفات الـ Subsystem الأساسية

**Core infrastructure:**
- `include/linux/list.h` — الـ API الرئيسي
- `include/linux/types.h` — الـ struct definitions
- `include/linux/container_of.h` — الـ magic macro

**Extensions:**
- `include/linux/list_bl.h` — Bit-spinlock list
- `include/linux/list_lru.h` — LRU list للـ memory management
- `include/linux/list_nulls.h` — Nulls list للـ RCU-safe traversal
- `include/linux/list_sort.h` — Merge sort
- `include/linux/llist.h` — Lock-free singly linked list

**Tests:**
- `lib/tests/list-test.c` — KUnit tests

---

### الخلاصة بجملة واحدة

الـ `list.h` هو "سكينة الجيش السويسري" للـ kernel — أي حاجة محتاجة تشيل مجموعة من العناصر وتتعامل معاها ديناميكياً بتستخدمه، وده بيخليه الملف الأكتر استخداماً في الـ kernel كله.
## Phase 2: شرح الـ Linked List Framework

### المشكلة: ليه الـ kernel محتاج linked list خاصة بيه؟

في C العادية، لو عايز تعمل linked list، بتعمل node فيها `data` و`next` pointer. المشكلة إن ده approach محدود جداً:

1. **كل struct لازم يبقى allocated على حدة** — كل node بتعمل `malloc` ليها، وده معناه overhead في الـ allocator.
2. **مش type-safe** — لو عندك list of `struct task_struct` ولازم تعمل cast في كل مرة، هيبقى في bugs.
3. **مش reusable** — كل data structure بتكتب ليها list logic من أول وجديد.
4. **الـ kernel مش بيستخدم `malloc`** — بيستخدم `kmalloc`/`slab`, وده بيغير طريقة التفكير.

الـ kernel بيدير **مئات الـ data structures** المختلفة كلها محتاجة linked lists — `struct task_struct`، `struct inode`، `struct net_device`، `struct page`، إلخ. لو كل واحدة فيهم كتبت list logic منفردة، الكود هيبقى مكرر ومليان bugs.

---

### الحل: الـ Intrusive Linked List

الـ kernel بيعمل **عكس** الـ approach التقليدي. بدل ما تحط الـ data جوه الـ node، بتحط الـ node جوه الـ data.

**التقليدي (non-intrusive):**
```c
struct node {
    struct task_struct *data;  // pointer للـ data
    struct node *next;
    struct node *prev;
};
```

**الـ kernel (intrusive):**
```c
struct task_struct {
    pid_t pid;
    char comm[16];
    // ...
    struct list_head tasks;  // الـ list node مدمج جوه الـ struct نفسه
};
```

الـ `struct list_head` هو كل حاجة:

```c
/* من include/linux/types.h */
struct list_head {
    struct list_head *next, *prev;
};
```

مجرد pointer للـ next وpointer للـ prev. مفيش data، مفيش type info. الـ magic كله في `container_of`.

---

### الـ Architecture: صورة كاملة

```
                    ┌─────────────────────────────────────────────────────┐
                    │              الـ Kernel Data Structures              │
                    └─────────────────────────────────────────────────────┘

 ┌──────────────────────────────────────────────────────────────────────────┐
 │  struct task_struct          struct net_device       struct inode        │
 │  ┌──────────────┐            ┌──────────────┐        ┌──────────────┐   │
 │  │ pid          │            │ name[IFNAMSIZ]│        │ i_ino        │   │
 │  │ comm[16]     │            │ flags        │        │ i_mode       │   │
 │  │ ...          │            │ ...          │        │ ...          │   │
 │  │ ┌──────────┐ │            │ ┌──────────┐ │        │ ┌──────────┐ │   │
 │  │ │list_head │ │            │ │list_head │ │        │ │list_head │ │   │
 │  │ │ next ────┼─┼────────────┼─┼► next ───┼─┼────────┼─┼► next    │ │   │
 │  │ │ prev ◄───┼─┼────────────┼─┼── prev   │ │        │ │  prev    │ │   │
 │  │ └──────────┘ │            │ └──────────┘ │        │ └──────────┘ │   │
 │  └──────────────┘            └──────────────┘        └──────────────┘   │
 └──────────────────────────────────────────────────────────────────────────┘

                    ▲
                    │  list_entry() / container_of()
                    │  "أنا عندي pointer على list_head، أرجعلي الـ struct الأم"
                    ▼

 ┌──────────────────────────────────────────────────────────────────────────┐
 │                        include/linux/list.h                              │
 │                                                                          │
 │  INIT_LIST_HEAD()     list_add()        list_del()      list_for_each()  │
 │  list_add_tail()      list_move()       list_splice()   list_entry()     │
 │  list_empty()         list_cut_*()      list_replace()  hlist_*()        │
 └──────────────────────────────────────────────────────────────────────────┘

                    ▲
                    │  #include <linux/list.h>
                    │

 ┌──────────────────────────────────────────────────────────────────────────┐
 │  Consumers (كل subsystem في الـ kernel)                                  │
 │                                                                          │
 │  Scheduler      Network Stack    VFS          Block Layer    IRQ         │
 │  (task lists)   (sk_buff lists)  (inode lists)(bio lists)   (handler    │
 │                                                              lists)      │
 └──────────────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: `container_of`

ده أهم macro في الـ kernel كله، وبيتدمج مع الـ list framework مباشرة.

```c
/* من include/linux/container_of.h */
#define container_of(ptr, type, member) ({              \
    void *__mptr = (void *)(ptr);                       \
    static_assert(__same_type(*(ptr), ((type *)0)->member) || \
                  __same_type(*(ptr), void),            \
                  "pointer type mismatch in container_of()"); \
    ((type *)(__mptr - offsetof(type, member))); })
```

**كيف بيشتغل بالتفصيل:**

```
struct task_struct في الميموري:
┌─────────────────────────────────────────────────┐
│ offset 0:  pid_t pid                            │
│ offset 4:  char comm[16]                        │
│ offset 20: ...                                  │
│ offset 156: struct list_head tasks ◄────────────┼── ptr (ده اللي عندنا)
│              └─ next                            │
│              └─ prev                            │
│ offset 172: ...                                 │
└─────────────────────────────────────────────────┘

container_of(ptr, struct task_struct, tasks):
    base_address = ptr - offsetof(struct task_struct, tasks)
                 = ptr - 156
                 = عنوان أول الـ struct task_struct
```

**مثال حقيقي من الـ scheduler:**

```c
struct task_struct {
    // ...
    struct list_head tasks;  // linked في run queue
    // ...
};

/* اللي بيحصل في الـ scheduler */
struct list_head *pos;
list_for_each(pos, &init_task.tasks) {
    /* pos بيشاور على list_head جوه task_struct */
    struct task_struct *task = list_entry(pos, struct task_struct, tasks);
    /* دلوقتي عندنا pointer على الـ task نفسها */
    printk("PID: %d, name: %s\n", task->pid, task->comm);
}
```

---

### الـ Circular Empty List: التصميم الذكي

الـ empty list مش `NULL` — هي head بتشاور على نفسها:

```
Empty list:
    ┌──────────────┐
    │   head       │
    │  next ───────┼──┐
    │  prev ◄──────┼──┘
    └──────────────┘

List with 2 elements:
    ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
    │   head       │     │   node A     │     │   node B     │
    │  next ───────┼────►│  next ───────┼────►│  next ───────┼──┐
 ┌──┼──── prev     │◄────┼──── prev     │◄────┼──── prev     │  │
 │  └──────────────┘     └──────────────┘     └──────────────┘  │
 └─────────────────────────────────────────────────────────────┘
```

**ليه circular؟**

- `list_add_tail` و`list_add` بيشتغلوا بنفس الـ `__list_add` الداخلي، بس بيمرروا arguments مختلفة.
- مفيش حاجة اسمها "end of list" — الـ iteration بتتوقف لما `pos == head`.
- Insert وDelete في O(1) دايماً لو عندك pointer على الـ node.

```c
/* list_add: insert بعد الـ head */
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);  /* new goes between head and head->next */
}

/* list_add_tail: insert قبل الـ head (= آخر الـ list) */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);  /* new goes between head->prev and head */
}

/* الـ actual implementation */
static inline void __list_add(struct list_head *new,
                               struct list_head *prev,
                               struct list_head *next)
{
    if (!__list_add_valid(new, prev, next))
        return;

    next->prev = new;    /* 1. الـ next بيشاور للـ new */
    new->next = next;    /* 2. الـ new بيشاور للـ next */
    new->prev = prev;    /* 3. الـ new بيشاور للـ prev */
    WRITE_ONCE(prev->next, new); /* 4. الـ prev بيشاور للـ new — LAST! */
}
```

**ليه `WRITE_ONCE` في آخر خطوة؟**
لأن ده الـ "publication point". أي thread تاني بيعمل traverse على الـ list لو شاف `prev->next == new`، لازم يلاقي الـ `new->next` و`new->prev` جاهزين وصح. الـ `WRITE_ONCE` بيضمن الـ compiler مش يعمل reorder للـ write ده قبل الـ writes التانية.

---

### الـ Deletion وPoison Values

```c
static inline void list_del(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->next = LIST_POISON1;  /* 0xdead000000000100 */
    entry->prev = LIST_POISON2;  /* 0xdead000000000200 */
}
```

الـ `LIST_POISON1` و`LIST_POISON2` هم addresses في الـ kernel virtual address space مش mapped. لو حد حاول يعمل dereference لـ node اتحذفت، الـ kernel هيعمل oops فوراً بدل ما الـ bug تظهر بعدين في مكان تاني.

ده في contrast مع `list_del_init()` اللي بتعمل reinitialize بدل poison:

```c
static inline void list_del_init(struct list_head *entry)
{
    __list_del_entry(entry);
    INIT_LIST_HEAD(entry);  /* next = prev = entry (empty list state) */
}
```

استخدم `list_del_init` لو الـ struct هيتعاد استخدامه، و`list_del` لو الـ struct هيتحذف بعدها.

---

### الـ Safe Iteration: مشكلة وحلها

```c
/* خطر! لو عملت list_del(pos) جوه الـ loop */
list_for_each(pos, head) {
    if (condition(pos))
        list_del(pos);  /* BUG: pos->next اتحذف، الـ loop هتتعطل */
}

/* الصح: استخدم _safe variant */
list_for_each_safe(pos, n, head) {
    /* n = pos->next اتحفظ قبل أي عملية */
    if (condition(pos))
        list_del(pos);  /* safe لأن n لسه سليم */
}
```

**الـ macro نفسه:**
```c
#define list_for_each_safe(pos, n, head)        \
    for (pos = (head)->next, n = pos->next;     \
         !list_is_head(pos, (head));            \
         pos = n, n = pos->next)
/* n بيتحفظ في كل iteration قبل ما pos يتحرك */
```

---

### الـ hlist: نوع تاني للـ Hash Tables

الـ `list_head` محتاج 2 pointers في الـ head (next وprev) = 16 bytes على 64-bit. في الـ hash table عندك **ملايين** buckets — ده waste كبير.

الـ `hlist` (hash list) بيحل الـ problem بـ head بـ pointer واحد بس:

```c
struct hlist_head {
    struct hlist_node *first;  /* pointer واحد بس = 8 bytes */
};

struct hlist_node {
    struct hlist_node *next;
    struct hlist_node **pprev;  /* pointer to pointer! */
};
```

**ليه `**pprev` ومش `*prev`؟**

```
hlist مع pointer to pointer:
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  head    │     │   node A     │     │   node B     │
│  first ──┼────►│  next ───────┼────►│  next = NULL │
└──────────┘     │  pprev ──────┼──┐  │  pprev ──────┼──┐
    ▲            └──────────────┘  │  └──────────────┘  │
    └───────────────────────────── ┘                     │
                                      points to A->next──┘
```

`pprev` بيشاور مش على الـ previous node بالظبط — بيشاور على **الـ field اللي بيشاور عليه**. يعني لـ node A، الـ `pprev` بيشاور على `head->first`. لـ node B، الـ `pprev` بيشاور على `A->next`.

**الفايدة:** لما بتحذف أي node، بتعمل:
```c
static inline void __hlist_del(struct hlist_node *n)
{
    struct hlist_node *next = n->next;
    struct hlist_node **pprev = n->pprev;

    WRITE_ONCE(*pprev, next);       /* سواء كان head->first أو prev->next */
    if (next)
        WRITE_ONCE(next->pprev, pprev);
}
```

من غير ما تعرف إيه اللي قبلها — head ولا node تانية. كل حاجة بتشتغل بنفس الكود.

---

### مقارنة: `list_head` vs `hlist`

| Feature | `list_head` | `hlist` |
|---------|-------------|---------|
| Head size | 16 bytes (2 pointers) | 8 bytes (1 pointer) |
| O(1) tail access | نعم | لأ |
| مناسب لـ | General lists، queues، stacks | Hash table buckets |
| Traverse | Bidirectional | Forward only |
| الـ `prev` في الـ node | pointer على node | pointer على pointer |

---

### الـ Analogy: سكة الحديد الدائرية

فكر في الـ linked list على إنها **مترو دائري** (circular metro):

| الـ Metro | الـ kernel list |
|-----------|-----------------|
| كل محطة عندها علامة للمحطة الجاية واللي فاتت | `struct list_head` بداخل كل struct |
| الـ راكب نفسه هو الـ data | الـ `task_struct` / `net_device` / etc. |
| بوابة الدخول (entrance) | الـ `head` (sentinel node) |
| الـ conductor عنده خريطة المحطات بس | الـ kernel code عنده pointer على `list_head` بس |
| لما تيجي تعرف اسم المحطة، بتروح للـ entrance وبتقرأ اللافتة | `list_entry()` = container_of = بترجع للـ struct الأم |
| تغيير ترتيب المحطات بيتعمل بتغيير علامات الاتجاه بس | pointer manipulation في O(1) |
| محطة الـ entrance ملهاش راكب — هي بس نقطة بداية | الـ `head` هو sentinel، مش data node حقيقية |
| لو الـ entrance بيشاور على نفسه → مفيش محطات | `head->next == head` → empty list |

**الجزء المهم في الـ analogy:** الـ conductor (الـ kernel code) مش محتاج يعرف كل تفاصيل كل راكب — هو بس بيتنقل بين المحطات (الـ `list_head` nodes). لما يحتاج معلومات الراكب، بيستخدم `list_entry` زي ما بيروح لـ الـ info booth بالمحطة ويسأل "مين الراكب اللي في الغرفة دي؟".

---

### الـ Hardening: `CONFIG_LIST_HARDENED`

الـ kernel بيدعم إنك تشغل validation على كل operation:

```c
static __always_inline bool __list_add_valid(struct list_head *new,
                                              struct list_head *prev,
                                              struct list_head *next)
{
    bool ret = true;

    if (!IS_ENABLED(CONFIG_DEBUG_LIST)) {
        /* fast path: minimal inline check */
        if (likely(next->prev == prev && prev->next == next &&
                   new != prev && new != next))
            return true;
        ret = false;
    }

    /* slow path: full validation + warning */
    ret &= __list_add_valid_or_report(new, prev, next);
    return ret;
}
```

الـ check بيتأكد إن:
- `next->prev == prev`: الـ consecutive nodes سليمة
- `prev->next == next`: نفس الكلام
- `new != prev && new != next`: مش بتضيف node موجودة أصلاً

لو حد عمل memory corruption واللي أدى لـ list corruption، الـ warning بيطلع قبل ما الـ kernel يعمل crash بطريقة غامضة.

---

### الـ Ownership vs Delegation

**الـ `list.h` بيملك:**
- الـ `struct list_head` و`struct hlist_head` و`struct hlist_node` — هم التعريف الرسمي
- كل الـ manipulation functions (add، del، move، splice، cut)
- كل الـ iteration macros
- الـ validation logic (hardening)
- الـ `container_of` / `list_entry` عن طريق include

**الـ `list.h` بيفوّض للـ driver/subsystem:**
- تحديد أي struct هيحتوي على `list_head`
- تحديد مين الـ head ومين الـ nodes
- الـ locking — الـ list.h مش thread-safe بذاتها، المستخدم مسؤول عن الـ spinlock/mutex
- الـ lifetime management — مين بيعمل allocate وfree للـ containing struct
- المعنى الـ semantic للـ list (run queue؟ wait queue؟ free list؟)

**مثال: الـ scheduler**
```c
/* الـ kernel/sched/core.c بيملك المعنى */
static void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
    /* الـ list.h بيوفر الـ mechanism */
    list_add_tail(&p->tasks, &rq->cfs_tasks);
    /* الـ scheduler بيوفر الـ policy */
}
```

---

### الـ Memory Layout: ليه الـ Intrusive Approach أفضل للـ cache؟

```
Non-intrusive (node allocated separately):
┌─────────────────┐        ┌──────────────────┐
│  task_struct    │        │  list node       │
│  (cache line 1) │        │  (different page)│
│  pid, comm, ... │  ┌────►│  next, prev      │
│  *node_ptr ─────┼──┘     │  *data ──────────┼──┐
└─────────────────┘        └──────────────────┘  │
                                                  ▼
                                            task_struct
                                            (yet another
                                             cache miss)

Intrusive (node embedded):
┌──────────────────────────────────────┐
│  task_struct                         │
│  pid, comm, ...                      │
│  list_head (next, prev) ◄── traverse │
│  more fields ...                     │
└──────────────────────────────────────┘
الـ list_head والـ data في نفس الـ cache line!
```

لما الـ scheduler بيعمل iterate على الـ tasks list، كل `list_head` هو جزء من الـ `task_struct` نفسها. احتمال كبير إن الـ fields المهمة (pid، state، priority) في نفس الـ cache line — يعني أقل cache misses.

---

### مثال شامل: Device Driver بيستخدم list

```c
/* driver يدير list of pending requests */
struct my_request {
    u32 command;
    u32 data;
    struct list_head list;    /* الـ node مدمج */
};

/* الـ driver state */
struct my_device {
    struct list_head pending;  /* الـ head */
    spinlock_t lock;
};

/* enqueue request */
void submit_request(struct my_device *dev, struct my_request *req)
{
    spin_lock(&dev->lock);
    list_add_tail(&req->list, &dev->pending);  /* أضف في الآخر */
    spin_unlock(&dev->lock);
}

/* process all pending */
void process_requests(struct my_device *dev)
{
    struct my_request *req, *tmp;

    spin_lock(&dev->lock);
    /* safe لأننا بنحذف جوه الـ loop */
    list_for_each_entry_safe(req, tmp, &dev->pending, list) {
        list_del(&req->list);         /* remove from list */
        spin_unlock(&dev->lock);

        do_the_work(req);            /* process */
        kfree(req);                  /* free */

        spin_lock(&dev->lock);
    }
    spin_unlock(&dev->lock);
}
```

ده exact pattern موجود في net/core، block layer، USB subsystem، وغيرهم.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Config Options والـ Poison Values — Cheatsheet

#### الـ Config Options المؤثرة على `list.h`

| Config | الأثر |
|--------|-------|
| `CONFIG_LIST_HARDENED` | يفعّل فحوصات integrity inline قبل كل add/del |
| `CONFIG_DEBUG_LIST` | يفعّل فحوصات أعمق وأبطأ — يستدعي slow-path دايماً |
| `CONFIG_ILLEGAL_POINTER_VALUE` | يضيف offset للـ poison pointers عشان تبقى في zone غير mappable |

#### الـ Poison Pointer Values

| الـ Macro | القيمة الافتراضية | الاستخدام |
|-----------|------------------|-----------|
| `LIST_POISON1` | `0x100 + DELTA` | يُكتب في `next` بعد `list_del()` |
| `LIST_POISON2` | `0x122 + DELTA` | يُكتب في `prev` بعد `list_del()` |
| `POISON_POINTER_DELTA` | `0` أو `CONFIG_ILLEGAL_POINTER_VALUE` | يُضاف للـ poison base لمنع exploitation |

> الـ poison values مش NULL عشان لو حد وصل ليها بـ NULL dereference check مش هيشوف حاجة — لكن لو dereference فعلي هيحصل page fault فوري، وده الهدف.

---

### 1. الـ Structs الأساسية

#### `struct list_head`

**الغرض:** اللبنة الأساسية لكل linked list في الـ kernel — embedded جوا أي struct محتاج يتحط في list.

```c
/* defined in include/linux/types.h */
struct list_head {
    struct list_head *next, *prev;  /* circular: يلفوا على بعض */
};
```

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `next` | `struct list_head *` | بوينتر للعنصر التالي — أو للـ head لو آخر عنصر |
| `prev` | `struct list_head *` | بوينتر للعنصر السابق — أو للـ head لو أول عنصر |

**الفكرة الجوهرية:** الـ `list_head` مش بيخزن البيانات — بيتضمن (embed) جوا الـ struct اللي فيه البيانات. عشان تطلع للـ struct الحاوي بتستخدم `container_of()` (عن طريق `list_entry()`).

**الاتصال بالـ Structs التانية:** أي struct في الـ kernel ممكن يضيف `struct list_head` كـ field فيه، وبكده يبقى جزء من list.

---

#### `struct hlist_head`

**الغرض:** رأس الـ hash list — بيستخدم pointer واحد بس (مش اتنين زي `list_head`) عشان يوفر ذاكرة في hash tables اللي فيها ملايين الـ buckets.

```c
/* defined in include/linux/types.h */
struct hlist_head {
    struct hlist_node *first;  /* أول node في الـ bucket، أو NULL */
};
```

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `first` | `struct hlist_node *` | أول عنصر في القائمة — NULL لو فاضية |

**التكلفة:** مش عندك O(1) access للـ tail — مناسب للـ hash tables اللي عادةً بتدور من الأول.

---

#### `struct hlist_node`

**الغرض:** العقدة اللي بتتضمن (embed) جوا أي struct محتاج يدخل في hash list.

```c
/* defined in include/linux/types.h */
struct hlist_node {
    struct hlist_node *next;    /* التالي في الـ bucket */
    struct hlist_node **pprev;  /* pointer-to-pointer للعنصر السابق */
};
```

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `next` | `struct hlist_node *` | العنصر التالي أو NULL |
| `pprev` | `struct hlist_node **` | بوينتر لـ `next` field في العنصر السابق (أو `&head->first`) |

**ليه `**pprev` مش `*prev` عادي؟** لأن الـ `hlist_head` يملك `*first` مش `*prev`، فلو استخدمنا `*prev` عادي مش هينفع نشيل أول عنصر من غير ما نعمل special case. الـ `**pprev` بيخليك تعدّل الـ field اللي بيشاور عليك مباشرةً — سواء كان `head->first` أو `prev_node->next`.

---

### 2. مخططات العلاقات بين الـ Structs

#### `list_head` — Circular Doubly Linked List

```
                    ┌─────────────────────────────────────┐
                    │               (circular)             │
                    ▼                                       │
┌──────────────┐  next  ┌──────────────┐  next  ┌──────────────┐
│  list HEAD   │───────▶│   node A     │───────▶│   node B     │
│  (sentinel)  │        │ (list_head)  │        │ (list_head)  │
└──────────────┘        └──────────────┘        └──────────────┘
       ▲          prev          │          prev          │
       └──────────────────────────────────────────────────┘
                    (circular prev chain)

┌────────────────────────────────────┐
│  my_struct                         │
│  ┌──────────────────────────────┐  │
│  │  int data;                   │  │
│  │  struct list_head list_node; │◀─┼── الـ list_head embedded هنا
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
         │
         │  list_entry(ptr, struct my_struct, list_node)
         │  = container_of(ptr, struct my_struct, list_node)
         ▼
   يرجعلك الـ my_struct* من الـ list_head*
```

#### `hlist_head` + `hlist_node` — Hash List

```
hash_table[0]:                   hash_table[1]:
┌────────────┐                   ┌────────────┐
│ hlist_head │                   │ hlist_head │
│  first ────┼──▶ NULL           │  first ────┼──┐
└────────────┘                   └────────────┘  │
                                                  ▼
hash_table[N]:                          ┌──────────────────┐
┌────────────┐                          │  hlist_node  A   │
│ hlist_head │                          │  next ───────────┼──▶ hlist_node B
│  first ────┼──┐                       │  pprev ──────────┼──▶ &head->first
└────────────┘  │                       └──────────────────┘
                ▼                                ▲
        ┌──────────────────┐                     │
        │  hlist_node  X   │          ┌──────────────────┐
        │  next ───────────┼──▶ NULL  │  hlist_node  B   │
        │  pprev ──────────┼──▶ &head │  next ───────────┼──▶ NULL
        └──────────────────┘          │  pprev ──────────┼──▶ &A->next
                                      └──────────────────┘

توضيح الـ pprev:
  A->pprev = &head->first      (يشاور على المكان اللي بيخزن A)
  B->pprev = &A->next          (يشاور على المكان اللي بيخزن B)
  لما بنشيل B: *B->pprev = B->next  →  A->next = NULL  ✓
```

---

### 3. دورة حياة الـ `list_head` — Creation → Usage → Teardown

```
══════════════════════════════════════════════════════
                  LIFECYCLE: list_head
══════════════════════════════════════════════════════

[1] CREATION / INITIALIZATION
    ─────────────────────────
    Static:
      LIST_HEAD(my_list);
      → struct list_head my_list = { &my_list, &my_list };
      → next و prev يشاوروا على نفس الـ head = empty list

    Dynamic:
      INIT_LIST_HEAD(&my_list);
      → WRITE_ONCE(list->next, list);
      → WRITE_ONCE(list->prev, list);

           ┌──────┐
           │ HEAD │
    prev ──┤      ├── next
           └──────┘
             ▲  │
             └──┘   (يشاور على نفسه = empty)

[2] INSERTION
    ─────────
    list_add(new, head):          list_add_tail(new, head):
    → after head (stack/LIFO)     → before head (queue/FIFO)

    Before:  HEAD ↔ A ↔ B         Before:  HEAD ↔ A ↔ B
    After:   HEAD ↔ NEW ↔ A ↔ B  After:   HEAD ↔ A ↔ B ↔ NEW

[3] TRAVERSAL
    ──────────
    list_for_each_entry(pos, head, member):
    → يبدأ من head->next
    → يمشي لحد ما pos == head (sentinel check)
    → list_entry() يرجع الـ outer struct

[4] MODIFICATION
    ────────────
    list_move()       → del + add (atomic بالنسبة للـ list state)
    list_replace()    → يحل node محل node تاني
    list_splice()     → يدمج listين

[5] DELETION
    ────────
    list_del(entry):
    → __list_del(entry->prev, entry->next)  يربط الجيران ببعض
    → entry->next = LIST_POISON1            يمنع accidental reuse
    → entry->prev = LIST_POISON2

    list_del_init(entry):
    → نفس الحذف لكن INIT_LIST_HEAD() بعدها
    → الـ entry يبقى empty list تقدر تستخدمها تاني

[6] TEARDOWN
    ────────
    مفيش free تلقائي — الـ list_head embedded في struct
    الـ caller مسؤول يعمل kfree()/vfree() على الـ outer struct
══════════════════════════════════════════════════════
```

---

### دورة حياة الـ `hlist_node`

```
══════════════════════════════════════════════════════
              LIFECYCLE: hlist_node
══════════════════════════════════════════════════════

[1] INIT HEAD
    INIT_HLIST_HEAD(head):  head->first = NULL

[2] INIT NODE
    INIT_HLIST_NODE(h):
      h->next = NULL;
      h->pprev = NULL;
    → hlist_unhashed(h) == true  (pprev == NULL)

[3] INSERT
    hlist_add_head(n, h):
      n->next  = h->first
      h->first = n
      n->pprev = &h->first

    هيكل بعد الـ insert:
      head->first → n → (old first)
      n->pprev = &head->first

[4] DELETE
    hlist_del(n):       → sets POISON, node still "hashed"
    hlist_del_init(n):  → sets NULL, node "unhashed"
    hlist_unhashed(n) يتحقق من pprev == NULL

[5] TEARDOWN
    نفس list_head: مفيش free تلقائي
══════════════════════════════════════════════════════
```

---

### 4. Call Flow Diagrams

#### إضافة عنصر — `list_add(new, head)`

```
list_add(new, head)
  └─▶ __list_add(new, head, head->next)
        ├─▶ __list_add_valid(new, prev=head, next=head->next)
        │     ├─ [CONFIG_LIST_HARDENED=n] → return true (no-op)
        │     └─ [CONFIG_LIST_HARDENED=y]
        │           ├─ inline check: next->prev==prev && prev->next==next
        │           │                && new!=prev && new!=next
        │           ├─ [OK]  → return true
        │           └─ [FAIL]→ __list_add_valid_or_report(new, prev, next)
        │                         └─▶ WARN + return false
        │
        ├─ [valid == false] → return (abort)
        │
        └─ [valid == true]
              next->prev = new
              new->next  = next
              new->prev  = prev
              WRITE_ONCE(prev->next, new)   ← atomic write للـ visibility
```

#### حذف عنصر — `list_del(entry)`

```
list_del(entry)
  └─▶ __list_del_entry(entry)
        ├─▶ __list_del_entry_valid(entry)
        │     ├─ [HARDENED=n] → return true
        │     └─ [HARDENED=y]
        │           ├─ check: prev->next==entry && next->prev==entry
        │           ├─ [OK]  → return true
        │           └─ [FAIL]→ __list_del_entry_valid_or_report(entry)
        │
        └─▶ __list_del(entry->prev, entry->next)
              next->prev = prev
              WRITE_ONCE(prev->next, next)
  │
  ├─ entry->next = LIST_POISON1   (0x100 + DELTA)
  └─ entry->prev = LIST_POISON2   (0x122 + DELTA)
```

#### `container_of` / `list_entry` — كيف تطلع للـ outer struct

```
list_for_each_entry(pos, head, member):
  ┌─────────────────────────────────────────┐
  │  pos = list_first_entry(head, T, member)│
  │    = list_entry(head->next, T, member)  │
  │    = container_of(head->next, T, member)│
  └─────────────────────────────────────────┘
            │
            ▼
  container_of(ptr, type, member):
    (type *)((char *)(ptr) - offsetof(type, member))

  مثال عملي:
  ┌──────────────────────────────────────┐
  │  struct task_struct {                │  ← outer struct
  │    pid_t pid;          ← offset 0   │
  │    ...                               │
  │    struct list_head tasks; ← off X  │  ← الـ list_head
  │    ...                               │
  │  }                                   │
  └──────────────────────────────────────┘

  ptr (list_head*) يشاور على &task->tasks
  container_of(ptr, struct task_struct, tasks)
    = (struct task_struct *)((char *)ptr - X)
    = يرجع task_struct*
```

#### `hlist_add_head` — إضافة لـ hash bucket

```
hlist_add_head(n, h)
  ├─ first = h->first
  ├─ WRITE_ONCE(n->next,   first)      ← n يشاور على القديم
  ├─ if (first):
  │    WRITE_ONCE(first->pprev, &n->next)  ← القديم يشاور على n->next
  ├─ WRITE_ONCE(h->first, n)           ← الـ head يشاور على n
  └─ WRITE_ONCE(n->pprev, &h->first)   ← n يشاور على h->first
```

#### `__hlist_del` — حذف من hash list

```
__hlist_del(n)
  ├─ next  = n->next
  ├─ pprev = n->pprev
  ├─ WRITE_ONCE(*pprev, next)           ← من يشاور على n يشاور على next
  └─ if (next):
       WRITE_ONCE(next->pprev, pprev)   ← next يشاور على ما كان n يشاور عليه
```

#### `list_splice_init` — دمج قائمتين

```
list_splice_init(list, head):
  ├─ [list_empty(list)] → return
  │
  └─▶ __list_splice(list, head, head->next)
        ├─ first = list->next   (أول عنصر في list)
        ├─ last  = list->prev   (آخر عنصر في list)
        ├─ first->prev = head   (prev يشاور على head)
        ├─ head->next  = first  (head يشاور على أول عنصر)
        ├─ last->next  = next   (آخر عنصر يشاور على ما كان بعد head)
        └─ next->prev  = last   (ما كان بعد head يشاور على last)
  └─▶ INIT_LIST_HEAD(list)      ← list فاضية دلوقتي
```

---

### 5. استراتيجية الـ Locking

#### الـ `list.h` نفسه مش فيه locks

الـ `list.h` بيوفر **الـ primitives فقط** — الـ locking مسؤولية الـ caller. لكن في تفاصيل مهمة:

#### الـ Memory Ordering — `WRITE_ONCE` و `READ_ONCE`

| الـ Primitive | الاستخدام في list.h | السبب |
|---------------|---------------------|-------|
| `WRITE_ONCE(ptr, val)` | كل pointer update حساس | يمنع compiler من split writes أو tearing |
| `READ_ONCE(ptr)` | `list_empty()`, `list_first_entry_or_null()` | يمنع compiler من caching الـ pointer |
| `smp_store_release()` | `list_del_init_careful()` | يضمن ordering مع consumer على CPU تاني |
| `smp_load_acquire()` | `list_empty_careful()` | pair مع `smp_store_release` |

#### `list_del_init_careful` + `list_empty_careful` — Lockless Pattern

```
CPU A (producer/deleter):           CPU B (consumer/checker):
  list_del_init_careful(entry)        list_empty_careful(head)
    __list_del_entry(entry)             next = smp_load_acquire(&head->next)
    WRITE_ONCE(entry->prev, entry)      return (next == head) &&
    smp_store_release(                         (next == READ_ONCE(head->prev))
      &entry->next, entry)
```

- الـ `smp_store_release` يضمن إن كل الـ memory ops قبله ظاهرة لـ CPU B
- الـ `smp_load_acquire` يضمن إن الـ reads بعده يشوفوا كل اللي عمله CPU A
- **القيد:** لازم الـ activity الوحيدة على الـ entry تكون `list_del_init()` — ممنوع `list_add()` concurrent

#### أنماط الـ Locking الشائعة مع list.h في الـ Kernel

```
Pattern 1: spinlock يحمي القائمة كلها
  spin_lock(&list_lock);
  list_add(&entry->list, &global_list);
  spin_unlock(&list_lock);

Pattern 2: RCU للقراءة + lock للكتابة
  /* Writer: */
  spin_lock(&lock);
  list_add_rcu(&entry->list, &head);   /* list_rcu.h */
  spin_unlock(&lock);

  /* Reader: */
  rcu_read_lock();
  list_for_each_entry_rcu(pos, &head, list) { ... }
  rcu_read_unlock();

Pattern 3: per-CPU lists (networking)
  /* بتستخدم __list_del_clearprev() */
  /* بيتحقق من prev != NULL بدل list_empty() */
  /* بيوفر WRITE_ONCE() overhead الإضافي */
```

#### الـ Lock Ordering مع hlist في hash tables

```
/* hash table with per-bucket lock */
struct bucket {
    spinlock_t    lock;
    struct hlist_head head;
};

bucket[hash(key)].lock  →  يحمي hlist_head + كل الـ hlist_nodes في الـ bucket

/* دايماً lock الـ bucket الأصغر index الأول لو محتاج تلوك bucket اتنين */
/* عشان تتجنب deadlock */
```

#### ليه الـ hardening validation مش بديل عن الـ locking؟

الـ `__list_add_valid` و `__list_del_entry_valid` بيشوفوا **snapshot** من الـ state في لحظة الاستدعاء — لو حصل race condition، الـ validation ممكن تعدي بنجاح ثم يحصل corruption. الـ validation بتحمي من bugs زي double-free أو use-after-free، مش من الـ concurrency.

```
Thread A:                    Thread B:
  __list_add_valid()           (nothing yet)
    → checks pass ✓
                               list_del(same_node)
                                 → corrupts pointers
  __list_add() executes
    → CORRUPTION!
```
**الحل:** lock قبل list operations في الـ concurrent contexts.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ `list_head` API (Doubly Circular Linked List)

| Function / Macro | Category | ببساطة |
|---|---|---|
| `LIST_HEAD_INIT(name)` | Init | initializer بدون declaration |
| `LIST_HEAD(name)` | Init | declare + initialize في سطر |
| `INIT_LIST_HEAD(list)` | Init | تهيئة runtime لـ list فاضية |
| `list_add(new, head)` | Insert | أضف بعد head مباشرةً (stack) |
| `list_add_tail(new, head)` | Insert | أضف قبل head (queue) |
| `list_del(entry)` | Delete | احذف وضع poison pointers |
| `list_del_init(entry)` | Delete | احذف وأعد تهيئة |
| `list_del_init_careful(entry)` | Delete | احذف مع memory ordering |
| `list_replace(old, new)` | Modify | استبدل node بتانية |
| `list_replace_init(old, new)` | Modify | استبدل + initialize القديمة |
| `list_swap(entry1, entry2)` | Modify | تبادل مكانين |
| `list_move(list, head)` | Move | انقل لأول list تانية |
| `list_move_tail(list, head)` | Move | انقل لآخر list تانية |
| `list_bulk_move_tail(head, first, last)` | Move | انقل range كامل لـ tail |
| `list_empty(head)` | Query | هل الـ list فاضية؟ |
| `list_empty_careful(head)` | Query | فاضية؟ مع memory barrier |
| `list_is_first(list, head)` | Query | هل أول عنصر؟ |
| `list_is_last(list, head)` | Query | هل آخر عنصر؟ |
| `list_is_head(list, head)` | Query | هل هو الـ head نفسه؟ |
| `list_is_singular(head)` | Query | عنصر واحد بس؟ |
| `list_count_nodes(head)` | Query | عد العناصر — O(n) |
| `list_rotate_left(head)` | Rotate | دور الـ list شمال |
| `list_rotate_to_front(list, head)` | Rotate | خلي عنصر معين أول |
| `list_cut_position(list, head, entry)` | Split | قص أول جزء شامل entry |
| `list_cut_before(list, head, entry)` | Split | قص أول جزء قبل entry |
| `list_splice(list, head)` | Merge | دمج بعد head |
| `list_splice_tail(list, head)` | Merge | دمج قبل head |
| `list_splice_init(list, head)` | Merge | دمج + تهيئة المصدر |
| `list_splice_tail_init(list, head)` | Merge | دمج tail + تهيئة المصدر |
| `list_entry(ptr, type, member)` | Access | احصل على الـ container struct |
| `list_first_entry(ptr, type, member)` | Access | أول عنصر (unsafe) |
| `list_last_entry(ptr, type, member)` | Access | آخر عنصر (unsafe) |
| `list_first_entry_or_null(...)` | Access | أول عنصر أو NULL |
| `list_last_entry_or_null(...)` | Access | آخر عنصر أو NULL |
| `list_next_entry(pos, member)` | Traversal | العنصر التالي |
| `list_prev_entry(pos, member)` | Traversal | العنصر السابق |
| `list_for_each(pos, head)` | Iteration | loop على raw list_head |
| `list_for_each_safe(pos, n, head)` | Iteration | loop آمن مع حذف |
| `list_for_each_entry(pos, head, member)` | Iteration | loop typed |
| `list_for_each_entry_safe(pos, n, head, member)` | Iteration | loop typed آمن |
| `list_for_each_entry_reverse(...)` | Iteration | loop عكسي |

#### الـ `hlist` API (Singly-headed Hash List)

| Function / Macro | Category | ببساطة |
|---|---|---|
| `HLIST_HEAD(name)` | Init | declare + initialize |
| `INIT_HLIST_HEAD(ptr)` | Init | تهيئة runtime |
| `INIT_HLIST_NODE(h)` | Init | تهيئة node |
| `hlist_unhashed(h)` | Query | هل الـ node خارج أي list؟ |
| `hlist_unhashed_lockless(h)` | Query | نفسه lockless |
| `hlist_empty(h)` | Query | هل الـ hlist فاضية؟ |
| `hlist_add_head(n, h)` | Insert | أضف في الأول |
| `hlist_add_before(n, next)` | Insert | أضف قبل node |
| `hlist_add_behind(n, prev)` | Insert | أضف بعد node |
| `hlist_add_fake(n)` | Insert | fake list لـ hlist_del |
| `hlist_del(n)` | Delete | احذف مع poison |
| `hlist_del_init(n)` | Delete | احذف + initialize |
| `hlist_move_list(old, new)` | Move | انقل كل الـ list |
| `hlist_splice_init(from, last, to)` | Merge | دمج hlists |
| `hlist_fake(h)` | Query | هل fake list؟ |
| `hlist_is_singular_node(n, h)` | Query | عنصر وحيد؟ |
| `hlist_count_nodes(head)` | Query | عد العناصر |
| `hlist_entry(ptr, type, member)` | Access | container_of |
| `hlist_for_each(pos, head)` | Iteration | loop على raw nodes |
| `hlist_for_each_safe(pos, n, head)` | Iteration | loop آمن |
| `hlist_for_each_entry(pos, head, member)` | Iteration | loop typed |
| `hlist_for_each_entry_safe(pos, n, head, member)` | Iteration | loop typed آمن |

---

### Category 1: Initialization

الـ initialization functions مسؤولة عن إنشاء list فارغة صالحة للاستخدام. الـ list في الـ kernel بتتمثل كـ circular doubly linked list، اللي لما بتكون فارغة بتـ point على نفسها. ده بيخلي كل عملية insert/delete تشتغل بشكل uniform من غير special cases للـ empty list.

---

#### `LIST_HEAD_INIT(name)`

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

**الـ macro** بتولد static initializer بيخلي الـ `list_head` يـ point على نفسه. مفيد لـ static أو global declarations.

**المعاملات:**
- `name` — اسم المتغير من نوع `struct list_head`

**Return:** initializer expression مش function

**مثال:**
```c
static struct list_head my_list = LIST_HEAD_INIT(my_list);
```

---

#### `LIST_HEAD(name)`

```c
#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)
```

بتعمل declaration + initialization في سطر واحد. الشائع في module-level أو struct member initialization.

**مثال في الـ kernel:**
```c
/* في net/core/dev.c */
LIST_HEAD(net_namespace_list);
```

---

#### `INIT_LIST_HEAD(list)`

```c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
    WRITE_ONCE(list->next, list);
    WRITE_ONCE(list->prev, list);
}
```

**ما بتعمله:** بتهيئ `list_head` في runtime بتخلي الـ `next` و `prev` يـ point على الـ struct نفسه. بتستخدم `WRITE_ONCE()` لتجنب compiler reordering في سياقات الـ concurrency.

**المعاملات:**
- `list` — pointer لـ `struct list_head` المراد تهيئتها

**Return:** void

**Key details:** الـ `WRITE_ONCE()` مش memory barrier، بس بيضمن إن الـ compiler ميعملش tearing أو reordering. لازم تستخدم explicit locking أو RCU لو في concurrency حقيقية.

**من بيستدعيها:** أي كود بيخلق list ديناميكيًا، زي `kmalloc` ثم init، أو في constructors لـ objects زي `inode_init_once()`.

---

#### `INIT_HLIST_NODE(h)`

```c
static inline void INIT_HLIST_NODE(struct hlist_node *h)
{
    h->next = NULL;
    h->pprev = NULL;
}
```

بتعمل تهيئة لـ `hlist_node` وبتخلي `pprev == NULL`، وده اللي بيستخدمه `hlist_unhashed()` كـ sentinel. مش بتستخدم `WRITE_ONCE()` لأنها مش مُستدعاة في سياق concurrency عادةً.

---

### Category 2: Internal Helpers (الـ `__xxx` Functions)

الـ internal helpers دي بتتعامل مع الـ raw pointer manipulation. المستخدم المباشر ليها بيكون الـ public API فقط، مش الـ driver code. كلها بتفترض إن الـ prev/next المحيطة معروفة مسبقًا.

---

#### `__list_add(new, prev, next)`

```c
static inline void __list_add(struct list_head *new,
                               struct list_head *prev,
                               struct list_head *next)
{
    if (!__list_add_valid(new, prev, next))
        return;

    next->prev = new;
    new->next = next;
    new->prev = prev;
    WRITE_ONCE(prev->next, new);  /* آخر write لأنه هو اللي "يُظهر" العنصر */
}
```

**ما بتعمله:** بتحشر `new` بين `prev` و `next`. الـ write الأخير هو `prev->next` لأنه بيكون visible للـ readers اللي بيمشوا forward. ده pattern مهم في الـ RCU-based list traversal.

**المعاملات:**
- `new` — العنصر الجديد
- `prev` — العنصر اللي هيكون قبله
- `next` — العنصر اللي هيكون بعده

**Key details:** الـ `__list_add_valid()` بتعمل corruption detection لو `CONFIG_LIST_HARDENED` مفعّل. بتتحقق إن `next->prev == prev` و `prev->next == next` قبل الإدراج.

---

#### `__list_del(prev, next)`

```c
static inline void __list_del(struct list_head *prev, struct list_head *next)
{
    next->prev = prev;
    WRITE_ONCE(prev->next, next);
}
```

أبسط شكل للحذف: بيربط `prev` و `next` ببعض. مش بيعدل الـ entry المحذوف نفسه. ده intentional لأن callers مختلفين بيعاملوا الـ entry المحذوف بشكل مختلف (poison vs reinit).

---

#### `__list_del_entry(entry)`

```c
static inline void __list_del_entry(struct list_head *entry)
{
    if (!__list_del_entry_valid(entry))
        return;

    __list_del(entry->prev, entry->next);
}
```

wrapper يضيف validation لـ `__list_del()`. الـ `__list_del_entry_valid()` بتتحقق إن `prev->next == entry` و `next->prev == entry`، بالتالي تكشف الـ double-free أو الـ corruption.

---

#### `__list_del_clearprev(entry)`

```c
static inline void __list_del_clearprev(struct list_head *entry)
{
    __list_del(entry->prev, entry->next);
    entry->prev = NULL;
}
```

**ما بتعمله:** بتحذف العنصر من الـ list وبتـ clear الـ `prev` pointer بس (مش الـ `next`). مصممة خصيصًا لـ per-CPU lists في الـ networking stack حيث الـ `list_empty()` مش مناسبة والتحقق من `prev == NULL` هو المؤشر على إن العنصر خارج الـ list.

**Key details:** بتتجنب الـ extra `WRITE_ONCE()` اللي بتيجي مع `list_del_init()`، ده مهم في الـ hot paths في الـ networking.

---

#### `__list_splice(list, prev, next)`

```c
static inline void __list_splice(const struct list_head *list,
                                  struct list_head *prev,
                                  struct list_head *next)
{
    struct list_head *first = list->next;
    struct list_head *last = list->prev;

    first->prev = prev;
    prev->next = first;

    last->next = next;
    next->prev = last;
}
```

الـ core splice operation. بتدخل محتويات `list` بين `prev` و `next`. مش بتـ check إذا `list` فاضية — ده مسؤولية الـ caller.

---

#### `__list_cut_position(list, head, entry)`

```c
static inline void __list_cut_position(struct list_head *list,
        struct list_head *head, struct list_head *entry)
{
    struct list_head *new_first = entry->next;
    list->next = head->next;
    list->next->prev = list;
    list->prev = entry;
    entry->next = list;
    head->next = new_first;
    new_first->prev = head;
}
```

بتنقل `[head->next .. entry]` لـ `list` الجديدة. الـ `new_first` هو العنصر اللي هيكون أول حاجة في `head` بعد القص.

---

#### Validation Helpers: `__list_add_valid` و `__list_del_entry_valid`

```c
/* CONFIG_LIST_HARDENED enabled */
static __always_inline bool __list_add_valid(struct list_head *new,
                                              struct list_head *prev,
                                              struct list_head *next)
{
    bool ret = true;
    if (!IS_ENABLED(CONFIG_DEBUG_LIST)) {
        if (likely(next->prev == prev && prev->next == next &&
                   new != prev && new != next))
            return true;
        ret = false;
    }
    ret &= __list_add_valid_or_report(new, prev, next);
    return ret;
}
```

**طبقتين من الـ validation:**

1. **`CONFIG_LIST_HARDENED` فقط:** فحص inline خفيف — بيتحقق من consistency الـ pointers المحيطة. لو فشل يستدعي slow path.
2. **`CONFIG_DEBUG_LIST` مضاف:** الـ `__list_add_valid_or_report()` بتعمل full check وبتطبع `WARN()` تفصيلي.

لو مفيش أي من الإعدادين، الـ function بترجع `true` دايمًا (zero overhead).

---

### Category 3: Insertion Functions

---

#### `list_add(new, head)`

```c
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}
```

**ما بتعمله:** بتضيف `new` مباشرةً بعد `head`. الـ result: `head → new → old_first`. ده stack semantics — آخر حاجة اتضافت هي أول حاجة تتشال لو مشيت من `head->next`.

**مثال كـ stack:**
```c
/* LRU cache: أضف المستخدم حديثاً في الأول */
list_add(&page->lru, &lru_list);
/* لاحقاً: list_first_entry(&lru_list, ...) هو الأحدث */
```

---

#### `list_add_tail(new, head)`

```c
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}
```

**ما بتعمله:** بتضيف `new` قبل `head` مباشرةً (في نهاية الـ list). Queue semantics — FIFO. الـ `head->prev` هو آخر عنصر، فالعنصر الجديد بيتضاف بعده.

**مثال:**
```c
/* Work queue: أضف المهام بالترتيب */
list_add_tail(&work->entry, &workqueue->work_list);
```

---

#### `hlist_add_head(n, h)`

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

**ما بتعمله:** بتضيف `n` في أول الـ hlist. كل الـ writes بـ `WRITE_ONCE()` عشان الـ hlist بتُستخدم كتير في سياقات RCU في hash tables.

**Key details:** الـ `pprev` مش pointer للـ node السابقة، هو **pointer-to-pointer** للـ `next` field في العنصر السابق (أو `h->first` لو في الأول). ده بيوفر O(1) deletion من غير معرفة الـ head.

```
head             n                  old_first
[first]---→[next|pprev]---→[next|pprev]
              ↑               ↑
           &head->first    &n->next
```

---

#### `hlist_add_before(n, next)`

```c
static inline void hlist_add_before(struct hlist_node *n,
                                     struct hlist_node *next)
{
    WRITE_ONCE(n->pprev, next->pprev);
    WRITE_ONCE(n->next, next);
    WRITE_ONCE(next->pprev, &n->next);
    WRITE_ONCE(*(n->pprev), n);
}
```

بتحشر `n` قبل `next`. عبقرية الـ `pprev` design واضحة هنا — مش محتاجين نعرف الـ head عشان نعمل insert في نص الـ list.

---

#### `hlist_add_behind(n, prev)`

```c
static inline void hlist_add_behind(struct hlist_node *n,
                                     struct hlist_node *prev)
{
    WRITE_ONCE(n->next, prev->next);
    WRITE_ONCE(prev->next, n);
    WRITE_ONCE(n->pprev, &prev->next);
    if (n->next)
        WRITE_ONCE(n->next->pprev, &n->next);
}
```

بتحشر `n` بعد `prev`. الـ `if (n->next)` ضروري لأن `prev` ممكن يكون آخر عنصر.

---

#### `hlist_add_fake(n)`

```c
static inline void hlist_add_fake(struct hlist_node *n)
{
    n->pprev = &n->next;
}
```

**ما بتعمله:** بتخلي الـ node تبدو إنها في list واحدها، بيخليها تـ survive `hlist_del()` بدون crash. المستخدم الحقيقي ليها هو الـ RCU path في hash tables اللي محتاج يعمل `hlist_del()` على node ممكن تكون isolated.

---

### Category 4: Deletion Functions

---

#### `list_del(entry)`

```c
static inline void list_del(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->next = LIST_POISON1;  /* 0x100 + delta */
    entry->prev = LIST_POISON2;  /* 0x122 + delta */
}
```

**ما بتعمله:** بتحذف `entry` من الـ list وبتضع poison values غير-NULL في الـ pointers. الـ poison values مصممة عشان لو حصل use-after-free، الـ dereference هيعمل page fault فوراً بدل ما يكون silent corruption.

**Key details:** بعد `list_del()`، الـ `list_empty(entry)` **مش بترجع true**. لو محتاج تعرف إن الـ entry محذوفة استخدم `list_del_init()` بدل منها.

---

#### `list_del_init(entry)`

```c
static inline void list_del_init(struct list_head *entry)
{
    __list_del_entry(entry);
    INIT_LIST_HEAD(entry);
}
```

**ما بتعمله:** بتحذف وبتعيد تهيئة الـ entry كـ empty list. بعدها `list_empty(entry)` بترجع `true`. ده الـ pattern المناسب لـ objects اللي ممكن يتضافوا تاني بعد الحذف.

**مثال:**
```c
/* driver unregister: احذف من الـ global list وأعد التهيئة */
list_del_init(&dev->list);
/* لاحقاً ممكن list_add(&dev->list, &other_list) */
```

---

#### `list_del_init_careful(entry)`

```c
static inline void list_del_init_careful(struct list_head *entry)
{
    __list_del_entry(entry);
    WRITE_ONCE(entry->prev, entry);
    smp_store_release(&entry->next, entry);  /* full memory barrier */
}
```

**ما بتعمله:** بتحذف وبتـ reinitialize مع ضمان memory ordering. الـ `smp_store_release()` على `entry->next` بيضمن إن أي memory operations قبل الحذف هتكون visible قبل ما أي CPU يشوف الـ list فاضية.

**الفرق عن `list_del_init()`:** مصممة للعمل مع `list_empty_careful()` في pair. بتستخدم في سياقات زي waitqueues اللي محتاجة ordering guarantees.

**Pseudocode Flow:**
```
delete entry from list         ← __list_del_entry
WRITE_ONCE(entry->prev = entry) ← no barrier yet
smp_store_release(entry->next = entry) ← STORE+RELEASE barrier
                               ← يضمن إن كل write سابق visible
```

---

#### `hlist_del(n)`

```c
static inline void hlist_del(struct hlist_node *n)
{
    __hlist_del(n);
    n->next = LIST_POISON1;
    n->pprev = LIST_POISON2;
}
```

نفس فلسفة `list_del()` — poison بعد الحذف. الـ `__hlist_del()` بيستغل الـ `pprev` design:

```c
static inline void __hlist_del(struct hlist_node *n)
{
    struct hlist_node *next = n->next;
    struct hlist_node **pprev = n->pprev;

    WRITE_ONCE(*pprev, next);         /* العنصر السابق يشاور على next */
    if (next)
        WRITE_ONCE(next->pprev, pprev); /* next يشاور على pprev الجديد */
}
```

ده بيشتغل سواء كان `n` أول عنصر في الـ list أو في النص — مش محتاج special case.

---

#### `hlist_del_init(n)`

```c
static inline void hlist_del_init(struct hlist_node *n)
{
    if (!hlist_unhashed(n)) {
        __hlist_del(n);
        INIT_HLIST_NODE(n);
    }
}
```

بتتحقق أولاً إن `n` فعلاً في list (`pprev != NULL`) قبل ما تحذفها. idempotent — تقدر تستدعيها أكتر من مرة بأمان. ده مفيد في cleanup paths.

---

### Category 5: Modification Functions

---

#### `list_replace(old, new)`

```c
static inline void list_replace(struct list_head *old, struct list_head *new)
{
    new->next = old->next;
    new->next->prev = new;
    new->prev = old->prev;
    new->prev->next = new;
}
```

**ما بتعمله:** بتحل `new` محل `old` في الـ list من غير ما تـ touch الـ `old` pointers. الـ `old` بعدها في state غير محددة — لو محتاج تعرف إنها حُذفت استخدم `list_replace_init()`.

---

#### `list_replace_init(old, new)`

```c
static inline void list_replace_init(struct list_head *old,
                                      struct list_head *new)
{
    list_replace(old, new);
    INIT_LIST_HEAD(old);
}
```

بعد الاستبدال، الـ `old` بتكون empty initialized list جاهزة للاستخدام تاني.

---

#### `list_swap(entry1, entry2)`

```c
static inline void list_swap(struct list_head *entry1,
                              struct list_head *entry2)
{
    struct list_head *pos = entry2->prev;  /* حفظ الموقع قبل الحذف */

    list_del(entry2);
    list_replace(entry1, entry2);
    if (pos == entry1)
        pos = entry2;            /* edge case: entry2 كانت مباشرة قبل entry1 */
    list_add(entry1, pos);
}
```

**Pseudocode:**
```
pos = entry2->prev
احذف entry2 من مكانها
ضع entry2 في مكان entry1
if pos كان entry1: pos = entry2 (لأن entry2 هي الآن في مكانها)
أضف entry1 بعد pos
```

الـ edge case الخاص مهم: لو `entry2` كانت مباشرة قبل `entry1`، بعد ما `entry1` اتحلت محل `entry2`، الـ `pos` (اللي كان `entry2->prev`) الآن هو `entry1` نفسها مش `entry2`. لازم نصحح.

---

### Category 6: Query Functions

---

#### `list_empty(head)`

```c
static inline int list_empty(const struct list_head *head)
{
    return READ_ONCE(head->next) == head;
}
```

**ما بتعمله:** بتتحقق إن `head->next == head` — الشرط الوحيد للـ list الفاضية في circular list. `READ_ONCE()` لمنع الـ compiler من caching الـ value.

**Return:** non-zero لو فاضية

---

#### `list_empty_careful(head)`

```c
static inline int list_empty_careful(const struct list_head *head)
{
    struct list_head *next = smp_load_acquire(&head->next);
    return list_is_head(next, head) && (next == READ_ONCE(head->prev));
}
```

**ما بتعمله:** بتتحقق إن `next == head` **و** `prev == head`. الـ `smp_load_acquire()` بيضمن إن أي writes من CPU تاني اللي عملت `list_del_init_careful()` هيكونوا visible قبل الـ check ده.

**متى تستخدمها:** بس لو الـ modification الوحيدة الممكنة هي `list_del_init_careful()`. لو في CPU تاني ممكن يعمل `list_add()`، مش كافية.

---

#### `list_is_singular(head)`

```c
static inline int list_is_singular(const struct list_head *head)
{
    return !list_empty(head) && (head->next == head->prev);
}
```

بتتحقق من حالتين: الـ list مش فاضية، والـ `next` و `prev` بيشاوروا على نفس العنصر (يعني عنصر واحد بس).

---

#### `list_count_nodes(head)`

```c
static inline size_t list_count_nodes(struct list_head *head)
{
    struct list_head *pos;
    size_t count = 0;

    list_for_each(pos, head)
        count++;

    return count;
}
```

**Return:** عدد العناصر — O(n)

**تحذير:** ما تستخدمهاش في hot paths. الـ list مش بتـ cache الـ size.

---

### Category 7: Rotation Functions

---

#### `list_rotate_left(head)`

```c
static inline void list_rotate_left(struct list_head *head)
{
    struct list_head *first;

    if (!list_empty(head)) {
        first = head->next;
        list_move_tail(first, head);
    }
}
```

**ما بتعمله:** بتاخد أول عنصر وبتحطه في الآخر. بمعنى تاني: بتحرك الـ "window" خطوة للأمام.

```
قبل:  head → A → B → C → head
بعد:  head → B → C → A → head
```

---

#### `list_rotate_to_front(list, head)`

```c
static inline void list_rotate_to_front(struct list_head *list,
                                         struct list_head *head)
{
    list_move_tail(head, list);
}
```

**ما بتعمله:** بتنقل الـ `head` نفسه لـ tail الـ `list`، وده effectively بيخلي `list` يكون الأول.

```
قبل:  head → A → B → list → C → head
بعد:  head → list → C → A → B → head
      (head اتنقل لبعد B)
```

---

### Category 8: Split Functions

---

#### `list_cut_position(list, head, entry)`

```c
static inline void list_cut_position(struct list_head *list,
        struct list_head *head, struct list_head *entry)
{
    if (list_empty(head))
        return;
    if (list_is_singular(head) &&
        !list_is_head(entry, head) && (entry != head->next))
        return;
    if (list_is_head(entry, head))
        INIT_LIST_HEAD(list);
    else
        __list_cut_position(list, head, entry);
}
```

**ما بتعمله:** بتقص `[head->next .. entry]` من `head` وبتحطها في `list`.

**المعاملات:**
- `list` — الـ list الجديدة اللي هتستقبل العناصر المقصوصة (لازم تكون فاضية أو يمشيلها الـ data)
- `head` — الـ source list
- `entry` — آخر عنصر في الجزء المقصوص

**Pseudocode:**
```
لو head فاضية: ارجع
لو head فيها عنصر واحد وentry مش هو: ارجع (مش معقول)
لو entry هو head نفسه: list = empty (مفيش حاجة تتقص)
غير كده: __list_cut_position
```

**مثال استخدام في الـ kernel:**
```c
/* في mm/swap.c — قص batch من LRU list */
LIST_HEAD(l_hold);
list_cut_position(&l_hold, &lru_list, batch_ptr);
```

---

#### `list_cut_before(list, head, entry)`

```c
static inline void list_cut_before(struct list_head *list,
                                    struct list_head *head,
                                    struct list_head *entry)
{
    if (head->next == entry) {
        INIT_LIST_HEAD(list);
        return;
    }
    list->next = head->next;
    list->next->prev = list;
    list->prev = entry->prev;
    list->prev->next = list;
    head->next = entry;
    entry->prev = head;
}
```

**الفرق عن `list_cut_position()`:** بتقص حتى `entry->prev` — يعني `entry` نفسها **بتفضل** في `head`. لو `entry == head`، كل العناصر بتتنقل لـ `list`.

```
قبل:  head → A → B → entry → C → head
بعد:  head → entry → C → head
      list → A → B → list
```

---

### Category 9: Merge (Splice) Functions

---

#### `list_splice(list, head)`

```c
static inline void list_splice(const struct list_head *list,
                                 struct list_head *head)
{
    if (!list_empty(list))
        __list_splice(list, head, head->next);
}
```

**ما بتعمله:** بتدمج محتويات `list` بعد `head` مباشرةً. Stack-style: عناصر `list` تيجي الأول في الـ combined list.

```
قبل:  head → A → B → head     list → X → Y → list
بعد:  head → X → Y → A → B → head
```

---

#### `list_splice_tail(list, head)`

```c
static inline void list_splice_tail(struct list_head *list,
                                     struct list_head *head)
{
    if (!list_empty(list))
        __list_splice(list, head->prev, head);
}
```

Queue-style: عناصر `list` بتتضاف في النهاية.

```
قبل:  head → A → B → head     list → X → Y → list
بعد:  head → A → B → X → Y → head
```

---

#### `list_splice_init(list, head)` و `list_splice_tail_init(list, head)`

نفس `list_splice` و `list_splice_tail` لكن بعد الدمج بيعملوا `INIT_LIST_HEAD(list)` عشان `list` تفضل valid empty list مش dangling pointers.

```c
static inline void list_splice_init(struct list_head *list,
                                     struct list_head *head)
{
    if (!list_empty(list)) {
        __list_splice(list, head, head->next);
        INIT_LIST_HEAD(list);
    }
}
```

**الاستخدام الشائع:**
```c
/* flush work items بشكل atomic */
LIST_HEAD(work_to_do);
spin_lock(&wq->lock);
list_splice_init(&wq->pending, &work_to_do);
spin_unlock(&wq->lock);
/* الآن process work_to_do بدون lock */
```

---

#### `hlist_move_list(old, new)`

```c
static inline void hlist_move_list(struct hlist_head *old,
                                    struct hlist_head *new)
{
    new->first = old->first;
    if (new->first)
        new->first->pprev = &new->first;
    old->first = NULL;
}
```

**ما بتعمله:** بتنقل كل الـ hlist من `old` لـ `new`. بتصلح الـ `pprev` pointer للعنصر الأول (اللي بيشاور على الـ head).

---

#### `hlist_splice_init(from, last, to)`

```c
static inline void hlist_splice_init(struct hlist_head *from,
                                      struct hlist_node *last,
                                      struct hlist_head *to)
{
    if (to->first)
        to->first->pprev = &last->next;
    last->next = to->first;
    to->first = from->first;
    from->first->pprev = &to->first;
    from->first = NULL;
}
```

**ما بتعمله:** بتدمج `from` في أول `to`، عناصر `from` جوا `[from->first .. last]` بتيجي في الأول. `to` ممكن تكون فاضية. بعد العملية `from` بتكون فاضية.

**المعاملات:**
- `from` — source hlist (لازم يحتوي على `last`)
- `last` — آخر node من `from` اللي هتتنقل
- `to` — destination hlist

---

### Category 10: Entry Access Macros

---

#### `list_entry(ptr, type, member)`

```c
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
```

الأساس. بيوصلك من الـ `list_head *` للـ struct الأب. بيستخدم `container_of()` اللي بيحسب offset الـ member وبيطرحه من الـ pointer.

```c
struct task_struct {
    /* ... */
    struct list_head tasks;
    /* ... */
};

struct list_head *pos;
list_for_each(pos, &init_task.tasks) {
    struct task_struct *t = list_entry(pos, struct task_struct, tasks);
    /* t الآن pointer للـ task */
}
```

---

#### `list_first_entry(ptr, type, member)` و `list_last_entry(ptr, type, member)`

```c
#define list_first_entry(ptr, type, member) \
    list_entry((ptr)->next, type, member)

#define list_last_entry(ptr, type, member) \
    list_entry((ptr)->prev, type, member)
```

**تحذير:** unsafe — مش بيتحققوا إن الـ list مش فاضية. لو الـ list فاضية، `(ptr)->next == ptr` والـ cast هيكون للـ head struct مش لـ element حقيقي. استخدم `list_first_entry_or_null()` لو مش متأكد.

---

#### `list_first_entry_or_null(ptr, type, member)`

```c
#define list_first_entry_or_null(ptr, type, member) ({     \
    struct list_head *head__ = (ptr);                       \
    struct list_head *pos__ = READ_ONCE(head__->next);      \
    pos__ != head__ ? list_entry(pos__, type, member) : NULL; \
})
```

آمن — بترجع `NULL` لو الـ list فاضية. الـ `READ_ONCE()` مهم هنا لأنه ممكن يُستخدم في lockless contexts (مع الـ RCU).

---

#### `list_next_entry_circular(pos, head, member)`

```c
#define list_next_entry_circular(pos, head, member) \
    (list_is_last(&(pos)->member, head) ?            \
    list_first_entry(head, typeof(*(pos)), member) : \
    list_next_entry(pos, member))
```

بيعمل wrap-around. لو `pos` هو آخر عنصر بيرجع الأول. مفيد في round-robin scheduling.

---

### Category 11: Iteration Macros

الـ iteration macros بتنقسم لمجموعتين رئيسيتين:

1. **Raw iteration** — على `struct list_head` مباشرةً
2. **Typed iteration** — على الـ container struct

ولكل منهم:
- Forward / Reverse
- Safe (تقدر تحذف current element أثناء الـ loop) / Unsafe
- Continue (من نقطة معينة) / From (من النقطة الحالية)

---

#### `list_for_each(pos, head)`

```c
#define list_for_each(pos, head) \
    for (pos = (head)->next; !list_is_head(pos, (head)); pos = pos->next)
```

أبسط form. `pos` هو `struct list_head *`. **Unsafe** — لو حذفت `pos` داخل الـ loop الـ `pos->next` هيكون garbage.

---

#### `list_for_each_safe(pos, n, head)`

```c
#define list_for_each_safe(pos, n, head) \
    for (pos = (head)->next, n = pos->next; \
         !list_is_head(pos, (head)); \
         pos = n, n = pos->next)
```

**ما بيميزها:** بتحفظ `pos->next` في `n` قبل ما الـ loop body تشتغل. بالتالي لو حذفت `pos` الـ iteration تكمل صح.

**تحذير:** لو حذفت `n` (التالي) داخل الـ loop، بتتعطل. **Safe against current element removal only.**

---

#### `list_for_each_entry(pos, head, member)`

```c
#define list_for_each_entry(pos, head, member)                    \
    for (pos = list_first_entry(head, typeof(*pos), member);      \
         !list_entry_is_head(pos, head, member);                  \
         pos = list_next_entry(pos, member))
```

الأكثر استخداماً في الـ kernel. `pos` بيكون pointer للـ container struct. مثال:

```c
struct device *dev;
list_for_each_entry(dev, &bus->devices, bus_list) {
    dev->driver->probe(dev);
}
```

---

#### `list_for_each_entry_safe(pos, n, head, member)`

```c
#define list_for_each_entry_safe(pos, n, head, member)               \
    for (pos = list_first_entry(head, typeof(*pos), member),         \
         n = list_next_entry(pos, member);                           \
         !list_entry_is_head(pos, head, member);                     \
         pos = n, n = list_next_entry(n, member))
```

نفس فكرة `list_for_each_safe` لكن typed. مثال شائع:

```c
struct my_obj *obj, *tmp;
list_for_each_entry_safe(obj, tmp, &my_list, node) {
    if (obj->should_delete) {
        list_del(&obj->node);
        kfree(obj);
    }
}
```

---

#### `list_for_each_entry_continue(pos, head, member)`

```c
#define list_for_each_entry_continue(pos, head, member)       \
    for (pos = list_next_entry(pos, member);                   \
         !list_entry_is_head(pos, head, member);               \
         pos = list_next_entry(pos, member))
```

بتبدأ من العنصر **التالي** لـ `pos` (مش من `pos` نفسه). مفيد لو وقفت في منتصف loop ثم اتمت من نفس نقطة الإيقاف.

---

#### `list_for_each_entry_from(pos, head, member)`

```c
#define list_for_each_entry_from(pos, head, member)           \
    for (; !list_entry_is_head(pos, head, member);            \
         pos = list_next_entry(pos, member))
```

بتبدأ من `pos` **نفسها** (شامل). الفرق الوحيد عن `continue`: بتتضمن العنصر الحالي.

---

#### `list_prepare_entry(pos, head, member)`

```c
#define list_prepare_entry(pos, head, member) \
    ((pos) ? : list_entry(head, typeof(*pos), member))
```

مصممة للاستخدام مع `list_for_each_entry_continue()`. لو `pos == NULL` بتعطيك pointer للـ head نفسه (بالـ container_of trick)، بالتالي الـ continue loop بتبدأ من أول عنصر.

```c
/* pagination: ابدأ من last_seen أو من الأول */
pos = list_prepare_entry(last_seen, &my_list, node);
list_for_each_entry_continue(pos, &my_list, node) {
    /* ... */
}
```

---

#### `list_safe_reset_next(pos, n, member)`

```c
#define list_safe_reset_next(pos, n, member) \
    n = list_next_entry(pos, member)
```

بتـ reset الـ `n` في `list_for_each_entry_safe` loop بعد ما re-acquired الـ lock. السيناريو:

```c
list_for_each_entry_safe(obj, tmp, &list, node) {
    spin_unlock(&lock);
    /* tmp ممكن يكون invalid هنا */
    do_slow_work(obj);
    spin_lock(&lock);
    list_safe_reset_next(obj, tmp, node); /* أعد تحديد tmp */
}
```

---

#### `hlist_for_each_entry(pos, head, member)`

```c
#define hlist_for_each_entry(pos, head, member)                             \
    for (pos = hlist_entry_safe((head)->first, typeof(*(pos)), member);     \
         pos;                                                                \
         pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))
```

loop على hlist. الفرق الأساسي: condition هو `pos != NULL` مش `pos != head` لأن الـ hlist مش circular.

---

#### `hlist_for_each_entry_safe(pos, n, head, member)`

```c
#define hlist_for_each_entry_safe(pos, n, head, member)                         \
    for (pos = hlist_entry_safe((head)->first, typeof(*pos), member);            \
         pos && ({ n = pos->member.next; 1; });                                  \
         pos = hlist_entry_safe(n, typeof(*pos), member))
```

الـ `({ n = pos->member.next; 1; })` هو statement expression بيحفظ `next` ويـ evaluate إلى `1` (true) دايمًا. ده pattern GNU C extension.

---

### Category 12: hlist Query Functions

---

#### `hlist_unhashed(h)` و `hlist_unhashed_lockless(h)`

```c
static inline int hlist_unhashed(const struct hlist_node *h)
{
    return !h->pprev;
}

static inline int hlist_unhashed_lockless(const struct hlist_node *h)
{
    return !READ_ONCE(h->pprev);
}
```

**ما بتعملوا:** بيتحققوا إن `pprev == NULL`، وده المؤشر على إن الـ node مش في أي list حالياً. الـ lockless version لازم في contexts اللي ممكن يكون في CPU تاني بيكتب في نفس الوقت.

---

#### `hlist_empty(h)`

```c
static inline int hlist_empty(const struct hlist_head *h)
{
    return !READ_ONCE(h->first);
}
```

`READ_ONCE()` ضروري لأن الـ hlist كتير بتتحقق منها بدون lock في hash table lookups.

---

#### `hlist_fake(h)` و `hlist_is_singular_node(n, h)`

```c
static inline bool hlist_fake(struct hlist_node *h)
{
    return h->pprev == &h->next;  /* fake list created by hlist_add_fake() */
}

static inline bool hlist_is_singular_node(struct hlist_node *n,
                                           struct hlist_head *h)
{
    return !n->next && n->pprev == &h->first;
}
```

الـ `hlist_is_singular_node()` بتتحقق من غير ما تمس الـ head (cache-friendly). الشرطان: `n->next == NULL` (آخر عنصر) و `n->pprev == &h->first` (أول عنصر أيضاً).

---

### ملاحظات ختامية مهمة

#### الـ Locking

الـ `list.h` **مش thread-safe بذاتها**. لازم الـ caller يوفر synchronization. الخيارات الشائعة في الـ kernel:

| Mechanism | متى |
|---|---|
| `spinlock_t` | عمليات سريعة، interrupt context |
| `mutex_t` | عمليات بطيئة، process context فقط |
| `RCU` | read-heavy, بتستخدم `list_for_each_entry_rcu()` من `rculist.h` |
| `seqlock_t` | read-heavy مع writers نادرين |

#### الـ WRITE_ONCE / READ_ONCE

بتمنع الـ compiler من:
- تحويل الـ write لـ multi-step operation (tearing)
- caching القيمة في register
- إزالتها كـ dead code

**مش** memory barrier — لو محتاج ordering بين CPUs استخدم `smp_store_release()` / `smp_load_acquire()`.

#### الـ Poison Pointers

```c
#define LIST_POISON1  ((void *) 0x100 + POISON_POINTER_DELTA)
#define LIST_POISON2  ((void *) 0x122 + POISON_POINTER_DELTA)
```

على معظم architectures، العناوين دي في kernel space لكن unmapped. أي dereference بيعمل page fault فوراً، وبيظهر في stack trace إن في use-after-free. `POISON_POINTER_DELTA` ممكن يكون `0xdead000000000000` على x86_64 لو `CONFIG_ILLEGAL_POINTER_VALUE` مفعّل.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

#### 1. debugfs Entries

**الـ** `list.h` نفسه مش subsystem بيعمل entries في debugfs، لكن الـ subsystems اللي بتستخدم linked lists بتعمل entries. الطريقة العامة:

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ entries الموجودة
ls /sys/kernel/debug/

# لو بتـ debug subsystem بيستخدم list زي scheduler
ls /sys/kernel/debug/sched/
cat /sys/kernel/debug/sched/debug

# لـ block layer اللي بيستخدم list_head كتير
ls /sys/kernel/debug/block/
cat /sys/kernel/debug/block/sda/rq-list   # request queue كـ list
```

لو عندك driver بيستخدم `list_head`، ممكن تضيف entry يطبع الـ list:

```c
/* في driver الـ init function */
static int my_list_show(struct seq_file *m, void *v)
{
    struct my_entry *entry;
    /* iterate using list_for_each_entry */
    list_for_each_entry(entry, &my_global_list, node) {
        seq_printf(m, "entry: id=%d val=%d\n", entry->id, entry->val);
    }
    return 0;
}
DEFINE_SHOW_ATTRIBUTE(my_list);

debugfs_create_file("my_list", 0444, parent, NULL, &my_list_fops);
```

```bash
# قراءة الـ entry
cat /sys/kernel/debug/my_driver/my_list
```

#### 2. sysfs Entries

**الـ** `list_head` لا توجد لها sysfs entries مباشرة، لكن subsystems بتكشف بيانات الـ list عبر sysfs:

```bash
# مثال: network devices مخزنة في linked list
ls /sys/class/net/
cat /sys/class/net/eth0/operstate

# block devices
ls /sys/class/block/

# عدد الـ entries في list زي modules
ls /sys/module/ | wc -l

# لـ kernel modules (مخزنة في list)
cat /sys/module/ext4/refcnt
ls /sys/module/ext4/parameters/
```

#### 3. ftrace — Tracepoints والـ Events

**الـ** `list_head` operations نفسها ملهاش tracepoints مباشرة، لكن ممكن نعمل function tracing:

```bash
# enable ftrace
mount -t tracefs nodev /sys/kernel/tracing

# تتبع كل function بتتعامل مع list operations في subsystem معين
cd /sys/kernel/tracing

# فلتر على functions بتتعامل مع list
echo '__list_add_valid_or_report' > set_ftrace_filter
echo '__list_del_entry_valid_or_report' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
# شغل الكود
cat trace

# تتبع الـ call stack لما يحصل list corruption
echo 1 > options/func_stack_trace
echo 1 > tracing_on
cat trace | head -100
echo 0 > tracing_on
```

لاستخدام **trace_events** مع subsystems تستخدم lists:

```bash
# مثال: kmem events لأن allocation بتستخدم lists
echo 1 > events/kmem/kmalloc/enable
echo 1 > events/kmem/kfree/enable
echo 1 > tracing_on
cat trace_pipe &
# شغل الكود المشبوه
echo 0 > tracing_on
```

لإضافة tracepoint مخصص في كود بيستخدم `list_head`:

```c
#include <trace/events/your_subsystem.h>

/* قبل list_add */
trace_my_list_add(new_entry);
list_add(&new_entry->node, &my_list);

/* قبل list_del */
trace_my_list_del(entry);
list_del(&entry->node);
```

#### 4. printk / Dynamic Debug

**الـ** `CONFIG_DEBUG_LIST` بيطبع تلقائيًا لما يحصل corruption. لتفعيل **dynamic debug** لكود يستخدم lists:

```bash
# تفعيل debug messages لـ module معين
echo 'module my_driver +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لـ file معين
echo 'file drivers/my/my_driver.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ debug messages
echo 'module my_driver +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p=print, f=function name, l=line number, m=module, t=thread id

# إيقاف
echo 'module my_driver -p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ control table
cat /sys/kernel/debug/dynamic_debug/control | grep my_driver
```

لإضافة debug prints في كود يستخدم `list_head`:

```c
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

/* طباعة حالة الـ list */
static void debug_dump_list(struct list_head *head, const char *name)
{
    struct my_entry *entry;
    int count = 0;

    pr_debug("=== List '%s' dump ===\n", name);
    list_for_each_entry(entry, head, node) {
        pr_debug("  [%d] ptr=%p id=%d\n", count++, entry, entry->id);
        /* safety check: لو الـ count عدى حد معقول → infinite loop */
        if (count > 10000) {
            pr_warn("List '%s' may be corrupt (>10000 entries)!\n", name);
            break;
        }
    }
    pr_debug("  Total: %d entries\n", count);
}
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة | متى تفعّله |
|---|---|---|
| `CONFIG_DEBUG_LIST` | يفعّل الفحص الكامل في كل `list_add`/`list_del` مع warning عند corruption | دايمًا في dev kernel |
| `CONFIG_LIST_HARDENED` | فحص minimal inline بدون overhead كبير في production | في production للحماية |
| `CONFIG_DEBUG_KERNEL` | يفعّل مجموعة كبيرة من debug options | في dev/test kernels |
| `CONFIG_KASAN` | يكتشف use-after-free و out-of-bounds على الـ list entries | لتتبع memory corruption |
| `CONFIG_KFENCE` | sampling-based memory safety checking خفيف | في production |
| `CONFIG_UBSAN` | يكتشف undefined behavior في pointer arithmetic | خلال development |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ locks اللي بتحمي الـ lists | في dev kernel |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ locking حول list operations | في dev kernel |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف مشاكل spinlock اللي بتحمي الـ lists | في dev kernel |
| `CONFIG_FTRACE` | يفعّل function tracing | حسب الحاجة |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dynamic debug messages | في dev kernel |
| `CONFIG_SLUB_DEBUG` | يكشف corruption في slab allocator اللي بيخزّن list entries | في dev kernel |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(DEBUG_LIST|LIST_HARDENED|KASAN|KFENCE)"

# أو
grep -E "DEBUG_LIST|LIST_HARDENED|KASAN" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

**الـ** `list_head` عبارة عن infrastructure عام، الأدوات بتكون على مستوى الـ subsystem:

```bash
# lsmod — يطبع modules list (kernel modules في linked list)
lsmod
cat /proc/modules  # نفس المعلومات raw

# /proc/slabinfo — slab allocator info لـ list entry types
cat /proc/slabinfo | grep -E "kmalloc|task_struct|dentry"

# crash utility للـ post-mortem analysis (vmcore)
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /proc/kcore
# داخل crash:
# list -H <head_addr> -s <struct_type>.member   → يطبع الـ list

# قراءة list داخل /proc/kcore بـ gdb
gdb /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /proc/kcore
# (gdb) p init_task.tasks    → يطبع task list head
# (gdb) p ((struct task_struct*)0xffff...)->tasks

# kernel live debugging بـ kgdb
echo ttyS0 > /sys/module/kgdboc/parameters/kgdboc
echo g > /proc/sysrq-trigger  # drop into kgdb
```

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|---|---|---|
| `list_add corruption. next->prev should be prev` | إضافة entry لـ list فيها corruption في الـ next pointer | فعّل `CONFIG_DEBUG_LIST`، استخدم KASAN لتحديد المكان |
| `list_add corruption. prev->next should be next` | إضافة entry لـ list فيها corruption في الـ prev pointer | تحقق من الـ locking، هل في race condition؟ |
| `list_del corruption. prev->next should be entry` | حذف entry بعد ما تم manipulate الـ list بشكل غلط | تحقق من double-free أو use-after-free |
| `list_del corruption. next->prev should be entry` | نفس السبب من الجهة الأخرى | استخدم KASAN أو KFENCE |
| `list_del on entry that was never added` | محاولة حذف entry لم تُضف للـ list (next/prev = LIST_POISON) | تحقق من initialization بـ `INIT_LIST_HEAD` |
| `BUG: unable to handle kernel NULL pointer dereference` عند traversal | إضافة entry بـ NULL pointer أو list corruption | فعّل `CONFIG_DEBUG_LIST` |
| `general protection fault` عند list traversal | pointer يشير لـ `LIST_POISON1` (0xdead000000000100) أو `LIST_POISON2` | use-after-free — استخدم KASAN |
| `kernel BUG at lib/list_debug.c` | `CONFIG_DEBUG_LIST` اكتشف corruption | اقرأ stack trace لتحديد الـ caller |
| Warning: suspicious RCU usage مع list | استخدام `list_for_each_entry_rcu` بدون RCU lock | أضف `rcu_read_lock()` / `rcu_read_unlock()` |

**قيم الـ LIST_POISON:**
```c
/* من include/linux/poison.h */
#define LIST_POISON1  ((void *) 0x100 + POISON_POINTER_DELTA)  /* next */
#define LIST_POISON2  ((void *) 0x200 + POISON_POINTER_DELTA)  /* prev */
```

لو شفت `0xdead000000000100` أو `0xdead000000000200` في crash dump، يعني الـ entry اتحذف بـ `list_del()` وبعدين اتوصل إليه تاني (**use-after-free**).

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* 1. قبل إضافة entry: تحقق من الـ initialization */
static void safe_list_add(struct my_entry *entry, struct list_head *head)
{
    /* WARN لو الـ entry مش initialized صح */
    WARN_ON(entry->node.next != NULL && entry->node.next != &entry->node);
    WARN_ON(entry->node.prev != NULL && entry->node.prev != &entry->node);

    list_add(&entry->node, head);
}

/* 2. قبل الحذف: تحقق إن الـ entry فعلًا في list */
static void safe_list_del(struct my_entry *entry)
{
    /* WARN لو الـ entry مش في list (already deleted) */
    WARN_ON(list_empty(&entry->node) &&
            entry->node.next == &entry->node);

    /* تحقق من LIST_POISON */
    WARN_ON(entry->node.next == LIST_POISON1);
    WARN_ON(entry->node.prev == LIST_POISON2);

    list_del_init(&entry->node);
}

/* 3. في الـ traversal: تحقق من الـ count */
static void check_list_sanity(struct list_head *head, size_t max_expected)
{
    size_t count = list_count_nodes(head);
    if (WARN(count > max_expected,
             "List has %zu entries, expected max %zu\n",
             count, max_expected)) {
        dump_stack();  /* اطبع call stack عشان نعرف مين وصّلنا هنا */
    }
}

/* 4. في الـ locking: تحقق إن الـ lock محموص قبل التعديل */
static void locked_list_add(struct my_entry *entry,
                            struct list_head *head,
                            spinlock_t *lock)
{
    /* يجب الاحتفاظ بـ lock قبل تعديل الـ list */
    WARN_ON(!spin_is_locked(lock));
    list_add(&entry->node, head);
}

/* 5. في cleanup: تحقق إن الـ list فاضية قبل الـ destroy */
static void my_module_exit(void)
{
    if (WARN(!list_empty(&my_global_list),
             "List not empty on exit! Memory leak!\n")) {
        struct my_entry *entry, *tmp;
        /* اطبع الـ entries المتبقية */
        list_for_each_entry_safe(entry, tmp, &my_global_list, node) {
            pr_err("  Leaked entry: id=%d\n", entry->id);
            list_del(&entry->node);
            kfree(entry);
        }
    }
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تتطابق مع حالة الـ Kernel

**الـ** `list_head` هي pure software data structure، لكن لما بتمثل hardware resources (مثلاً: list of DMA buffers، list of hardware queues)، لازم نتحقق:

```bash
# مثال: list of PCI devices يجب تتطابق مع hardware
lspci -v | grep -c "^[0-9]"        # عدد PCI devices في hardware
ls /sys/bus/pci/devices/ | wc -l    # عدد devices في kernel list
# لو الأرقام مختلفة → مشكلة في الـ enumeration

# مثال: network interfaces
ip link show | grep -c "^[0-9]"     # عدد interfaces في kernel list
ethtool -i eth0 | grep driver       # تحقق إن الـ driver recognized

# مثال: block devices
lsblk                               # من kernel list
fdisk -l 2>/dev/null | grep -c "^Disk /dev"  # من hardware
```

#### 2. Register Dump Techniques

```bash
# قراءة registers بـ devmem2 (يحتاج تثبيت)
apt install devmem2    # أو من المصدر
devmem2 0xFEDC0000 w  # اقرأ 32-bit من العنوان

# قراءة مباشرة من /dev/mem (يحتاج CONFIG_STRICT_DEVMEM=n أو kernel.perf_event_paranoid)
dd if=/dev/mem bs=4 count=1 skip=$((0xFEDC0000/4)) 2>/dev/null | xxd

# بـ io utility (من package ioport)
# inb/outb/inw/outw للـ I/O ports
io -4 -r 0x3F8   # اقرأ 32-bit من port 0x3F8

# لـ MMIO registers عبر /sys/bus/pci
# PCI config space
lspci -xxx -s 0000:00:1f.0   # hex dump لـ config space
setpci -s 0000:00:1f.0 0x04.w  # قراءة word من offset 0x04

# لـ embedded systems: direct memory map
# مثلاً لقراءة GPIO registers على ARM
devmem2 0x3F200000 w   # Raspberry Pi GPIO base
```

#### 3. Logic Analyzer / Oscilloscope Tips

لما الـ list corruption مرتبطة بـ hardware timing issues:

```
السيناريو: DMA buffer list يتعدل من interrupt handler + userspace
┌─────────────────────────────────────────────────────────┐
│  Logic Analyzer Setup:                                   │
│                                                          │
│  CH0: IRQ line          ──┐__┌──────────                │
│  CH1: DMA active        ────┘  └────────                 │
│  CH2: CPU busy (GPIO)   ──────┐    ┌───                  │
│                               └────┘                     │
│                                                          │
│  المشكلة: CH2 بيحاول يعدل الـ list وقت CH1 active      │
│  الحل: تحقق إن spinlock بيحمي الـ list في الحالتين      │
└─────────────────────────────────────────────────────────┘

نقاط القياس على الـ oscilloscope:
- قبل list_add(): set GPIO high
- بعد list_add():  set GPIO low
- في IRQ handler قبل list_del(): set GPIO2 high
- look for overlap → race condition
```

```c
/* أضف GPIO toggles للـ debugging */
#include <linux/gpio.h>

static void debug_list_add(struct my_entry *entry, struct list_head *head)
{
    gpio_set_value(DEBUG_GPIO_1, 1);   /* mark start */
    list_add(&entry->node, head);
    gpio_set_value(DEBUG_GPIO_1, 0);   /* mark end */
}
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | الـ Pattern في Kernel Log | التفسير |
|---|---|---|
| DMA write بعد free (IOMMU fault) | `DMAR: DRHD: handling fault` + kernel BUG | DMA device بتكتب في memory اتحرر، بيخرب الـ list |
| Memory ECC error | `EDAC MC0: CE memory read error` | single-bit error في الـ list pointers |
| PCIe link flap | `pcieport: AER: Uncorrected ... TLP` + device disappears | device اختفى من list فجأة |
| Power management failure | `PM: Device ... failed to resume` + list corruption | device state مش sync مع kernel list |
| Cache coherency issue (SMP) | `BUG: spinlock` أو list corruption على multi-core | كل core شايف نسخة مختلفة من الـ list |
| NVME controller reset | `nvme nvme0: I/O ... timeout` + request queue corruption | request list اتخربت |

```bash
# تحقق من ECC errors
edac-util -s 0    # 0 errors expected
cat /sys/devices/system/edac/mc/mc0/ue_count   # uncorrected errors
cat /sys/devices/system/edac/mc/mc0/ce_count   # corrected errors

# تحقق من IOMMU faults
dmesg | grep -i "iommu\|dmar\|fault"

# تحقق من memory errors
mcelog --daemon
cat /var/log/mcelog
```

#### 5. Device Tree Debugging

لو الـ list corruption بتيجي من driver بيقرأ الـ DT بشكل غلط:

```bash
# تحقق من الـ DT المحمّل
ls /sys/firmware/devicetree/base/
# أو
dtc -I fs -O dts /sys/firmware/devicetree/base/ 2>/dev/null | head -100

# تحقق من device specific node
cat /sys/firmware/devicetree/base/soc/spi@7e204000/compatible
xxd /sys/firmware/devicetree/base/soc/gpio@7e200000/reg

# compile DT وتحقق من syntax
dtc -I dts -O dtb -o test.dtb test.dts
dtc -I dtb -O dts test.dtb    # reverse: dtb → dts للتحقق

# تحقق من interrupts configuration
cat /sys/firmware/devicetree/base/soc/interrupt-controller@7e00b200/reg

# لو driver بيبني list من DT nodes
# تحقق إن عدد child nodes في DT = عدد entries في kernel list
grep -c "compatible" /sys/firmware/devicetree/base/soc/*/compatible 2>/dev/null
```

```bash
# overlay DT للـ debugging بدون reboot
dtoverlay my_debug_overlay.dtbo

# تحقق من applied overlays
ls /sys/kernel/config/device-tree/overlays/
cat /sys/kernel/config/device-tree/overlays/my_overlay/status
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. تفعيل CONFIG_DEBUG_LIST في kernel جديد:**

```bash
# في kernel build directory
scripts/config --enable CONFIG_DEBUG_LIST
scripts/config --enable CONFIG_LIST_HARDENED
make olddefconfig
make -j$(nproc)
```

**2. تشغيل KASAN kernel لاكتشاف list corruption:**

```bash
# compile kernel مع KASAN
scripts/config --enable CONFIG_KASAN
scripts/config --set-val CONFIG_KASAN_INLINE y

# شغّل وراقب الـ output
dmesg -w | grep -E "KASAN|BUG|list"
```

**3. ftrace لتتبع list operations:**

```bash
#!/bin/bash
# script: trace_list_ops.sh

TRACE=/sys/kernel/tracing

# reset
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# set tracer
echo function > $TRACE/current_tracer

# filter: فقط functions في module معين
echo ':mod:my_module' > $TRACE/set_ftrace_filter

# enable stack traces
echo 1 > $TRACE/options/func_stack_trace

# start
echo 1 > $TRACE/tracing_on
echo "Tracing... press Enter to stop"
read
echo 0 > $TRACE/tracing_on

# save results
cat $TRACE/trace > /tmp/list_trace_$(date +%s).txt
echo "Saved to /tmp/list_trace_*.txt"
```

**4. كشف list corruption في runtime بـ crash:**

```bash
# install crash
apt install crash linux-image-$(uname -r)-dbg

# فتح live kernel
sudo crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /proc/kcore

# داخل crash — اطبع task list
crash> ps | head -20
crash> list -H init_task.tasks -s task_struct.pid,comm

# اطبع list عن طريق عنوانها
crash> list -H 0xffff888001234567 -s my_struct.id

# تحقق من integrity
crash> foreach task list -H task.files.fd_array -s file.f_path.dentry
```

**5. dynamic debug للـ list operations في module:**

```bash
# تفعيل
echo 'module my_module +pflmt' > /sys/kernel/debug/dynamic_debug/control

# مشاهدة الـ output في real-time
dmesg -w &
# أو
tail -f /var/log/kern.log

# إيقاف
echo 'module my_module -p' > /sys/kernel/debug/dynamic_debug/control
```

**6. سكريبت للتحقق من صحة list_head:**

```bash
#!/bin/bash
# check_list_poison.sh — يبحث في crash dump عن LIST_POISON values

VMCORE=${1:-/proc/kcore}
VMLINUX=${2:-/usr/lib/debug/lib/modules/$(uname -r)/vmlinux}

echo "Checking for LIST_POISON in $VMCORE..."

# LIST_POISON1 = 0xdead000000000100
# LIST_POISON2 = 0xdead000000000200
crash $VMLINUX $VMCORE << 'EOF'
search -u 0xdead000000000100
search -u 0xdead000000000200
quit
EOF
```

**7. اطبع حالة list_head في gdb/kgdb:**

```bash
# في kgdb session
# اطبع list_head
(gdb) p my_list_head
(gdb) p my_list_head.next
(gdb) p my_list_head.prev

# traverse الـ list
# افترض struct my_struct { int id; struct list_head node; };
(gdb) set $cur = my_list_head.next
(gdb) while ($cur != &my_list_head)
  p ((struct my_struct*)((char*)$cur - offsetof(struct my_struct, node)))->id
  set $cur = $cur->next
end

# تحقق من LIST_POISON
(gdb) p/x my_entry.node.next
# 0xdead000000000100 = LIST_POISON1 → entry was deleted!
```

**8. مثال على output وتفسيره:**

```
[ 1234.567890] list_add corruption. next->prev should be ffff888001234500,
               but was ffff888009abcdef. (next=ffff888001234600).
```

**التفسير:**
- الـ `next` node الـ `prev` pointer بتشاور على عنوان غلط
- `ffff888001234500` هو العنوان المتوقع (الـ `prev` node)
- `ffff888009abcdef` هو القيمة الفعلية → memory corruption
- الـ `next` node نفسه على `ffff888001234600`

```
السبب الأكثر شيوعًا:
1. race condition: thread تاني عدّل الـ list بدون lock
2. use-after-free: node اتحذف وبعدين اتوصل إليه تاني
3. buffer overflow: كود جار كتب فوق الـ list pointers
```

```bash
# للتحقق من race condition:
echo 1 > /sys/kernel/debug/provoke-crash/DEADLOCK   # مش هنا
# استخدم lockdep
dmesg | grep -E "possible deadlock|lock order"

# للتحقق من use-after-free:
dmesg | grep -E "KASAN|use-after-free|slab-out-of-bounds"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Kernel Panic على Industrial Gateway بـ RK3562 بسبب Double-Free في قائمة الـ I2C Devices

#### العنوان
**Double-free على `list_head` في driver الـ I2C على RK3562 يسبب kernel panic في بيئة إنتاجية**

#### السياق
بيئة industrial gateway بتشتغل على **RK3562** بـ Linux 6.1. الجهاز بيجمّع بيانات من 12 حساس I2C موزّعين على busات مختلفة. في production، الجهاز بيتـreboot كل بضع ساعات بدون سبب واضح.

#### المشكلة
الـ crash dump بيظهر:

```
BUG: kernel NULL pointer dereference, address: 0000000000000068
...
Call trace:
 i2c_del_adapter+0x8c/0x1e0
 __list_del_entry+0x24/0x60
 list_del+0x10/0x30
```

الـ log بيظهر `LIST_POISON2` في `entry->prev` — علامة إن `list_del()` اتنادت مرتين على نفس الـ node.

#### التحليل
في `list.h`، `list_del()` بتعمل:

```c
static inline void list_del(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->next = LIST_POISON1;  /* 0x00000000DEAD000 */
    entry->prev = LIST_POISON2;  /* 0x00000000DEAD001 */
}
```

لما بتتنادى `list_del()` تاني مرة على نفس الـ entry، الـ `__list_del_entry_valid()` مع `CONFIG_LIST_HARDENED` بتكتشف إن:

```c
/* prev->next != entry — because prev is LIST_POISON2 */
if (likely(prev->next == entry && next->prev == entry))
    return true;  /* هيفشل هنا */
ret = false;
/* بيندي على __list_del_entry_valid_or_report */
```

بدون `CONFIG_LIST_HARDENED`، الـ dereference على `LIST_POISON2` بيعمل NULL pointer dereference مباشرة.

الـ bug في الـ driver: thread واحد بيعمل `i2c_del_adapter` وفي نفس الوقت الـ error handler بعمل `list_del` على نفس الـ adapter node بدون locking.

#### الحل

```c
/* في i2c-rk3562.c — إضافة mutex protection */
static void rk3562_i2c_remove_device(struct rk3562_i2c_dev *i2c_dev)
{
    mutex_lock(&i2c_dev->adapter_lock);

    /* تأكد إن الـ node لسه في الـ list قبل الحذف */
    if (!list_empty(&i2c_dev->adapter_node)) {
        list_del_init(&i2c_dev->adapter_node); /* آمن — بيعمل INIT بعد الحذف */
    }

    mutex_unlock(&i2c_dev->adapter_lock);
}
```

استخدام `list_del_init()` بدل `list_del()` لأنها بتعمل `INIT_LIST_HEAD()` بعد الحذف، فالـ node بتبقى في state آمنة ولو اتنادى عليها تاني.

```bash
# للتحقق في runtime
echo 1 > /sys/kernel/debug/dynamic_debug/control
dmesg | grep "list_del corruption"
```

#### الدرس المستفاد
`list_del()` بتحط `LIST_POISON` في الـ pointers عشان تكتشف الـ use-after-free، بس مش بتحمي من double-free بدون locking. دايمًا استخدم `list_del_init()` لما الـ node ممكن تتحذف من أكتر من context، وفعّل `CONFIG_LIST_HARDENED` في production kernels.

---

### السيناريو الثاني: Memory Leak في Android TV Box على Allwinner H616 بسبب خطأ في `list_for_each_entry_safe`

#### العنوان
**الـ HDMI hotplug على Allwinner H616 بيسبب memory leak تدريجي بيوقف الـ TV box بعد أيام**

#### السياق
Android TV box بـ **Allwinner H616** و HDMI output. المنتج في السوق وبيوصل complaints إن الأجهزة بتبطأ وبتتوقف بعد 3-5 أيام من الاستخدام المتواصل، خصوصًا لو الـ TV اتقفل وفتح كتير.

#### المشكلة
الـ HDMI driver بيخزّن قائمة `list_head` من الـ `drm_connector_state` objects. عند كل hotplug disconnect، الـ cleanup code بيتكلم على:

```c
/* الكود الغلط */
list_for_each_entry(state, &hdmi_dev->state_list, node) {
    if (state->connector == old_connector) {
        list_del(&state->node);   /* خطر! تعديل الـ list أثناء الـ iteration */
        kfree(state);
    }
}
```

#### التحليل
الـ `list_for_each_entry` macro في `list.h`:

```c
#define list_for_each_entry(pos, head, member)              \
    for (pos = list_first_entry(head, typeof(*pos), member); \
         !list_entry_is_head(pos, head, member);            \
         pos = list_next_entry(pos, member))  /* هنا المشكلة */
```

لما بنعمل `list_del()` على `pos` ثم `kfree(pos)`، الـ `pos->member.next` في نهاية الـ loop iteration بيبقى dangling pointer. الـ macro بتحاول تعمل `list_next_entry(pos, member)` على memory اتحررت.

النتيجة: إما skip entries فبتحصل memory leak، أو kernel crash لو الـ memory اتاخدت لحاجة تانية.

الحل الصح هو `list_for_each_entry_safe`:

```c
#define list_for_each_entry_safe(pos, n, head, member)          \
    for (pos = list_first_entry(head, typeof(*pos), member),    \
        n = list_next_entry(pos, member);                       \
         !list_entry_is_head(pos, head, member);                \
         pos = n, n = list_next_entry(n, member))
/* بتحفظ الـ next مسبقًا في n قبل ما تعدّل الـ pos */
```

#### الحل

```c
/* الكود الصح */
static void h616_hdmi_cleanup_states(struct h616_hdmi_dev *hdmi_dev,
                                      struct drm_connector *old_connector)
{
    struct drm_connector_state *state, *tmp;

    /* استخدم _safe لأننا بنحذف أثناء الـ iteration */
    list_for_each_entry_safe(state, tmp, &hdmi_dev->state_list, node) {
        if (state->connector == old_connector) {
            list_del(&state->node);
            kfree(state);
        }
    }
}
```

```bash
# تتبع الـ memory leak على الجهاز
cat /proc/meminfo | grep -E "MemFree|Slab"
# أو استخدم kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
**القاعدة الذهبية:** أي loop بتحذف فيها elements من `list_head` لازم تستخدم النسخة `_safe` من الـ macro. الفرق الوحيد إنها بتحفظ `next` في متغير مؤقت قبل ما تبدأ الـ iteration — تفصيلة صغيرة بتمنع crashes وleaks كبيرة.

---

### السيناريو الثالث: Race Condition في IoT Sensor Hub على STM32MP1 بسبب عدم استخدام `list_empty_careful`

#### العنوان
**الـ SPI sensor queue على STM32MP1 بتتبهدل تحت heavy load بسبب race condition في الـ list check**

#### السياق
IoT sensor hub بـ **STM32MP1** بيجمّع readings من 8 SPI sensors بسرعة عالية (1kHz لكل sensor). الـ kernel driver بيستخدم `list_head` كـ work queue بين الـ IRQ handler والـ processing thread. في الـ stress tests، بتظهر corrupted readings وأحيانًا kernel oops.

#### المشكلة
الـ IRQ handler بيعمل `list_add_tail()` على الـ pending queue، والـ worker thread بيعمل:

```c
/* في الـ worker thread — الكود الغلط */
while (!list_empty(&sensor_dev->pending_queue)) {
    /* بين list_empty والـ list_del، IRQ handler ممكن يضيف أو يشيل */
    entry = list_first_entry(&sensor_dev->pending_queue,
                              struct sensor_entry, list);
    list_del(&entry->list);
    process_sensor_data(entry);
}
```

#### التحليل
`list_empty()` في `list.h`:

```c
static inline int list_empty(const struct list_head *head)
{
    return READ_ONCE(head->next) == head;
}
```

الـ `READ_ONCE` بتضمن إن الـ read atomically، بس مفيش ضمان إن الـ list هتفضل empty بعدها. على معمارية ARM Cortex-A7 (اللي في STM32MP1)، ممكن يحصل التالي:

```
Thread:                    IRQ Handler:
list_empty() → false
                           list_del_init(entry)  /* شيل من list */
list_first_entry()         /* الـ entry اتشالت! */
list_del() → CORRUPTION
```

الصح هو `list_empty_careful()`:

```c
static inline int list_empty_careful(const struct list_head *head)
{
    struct list_head *next = smp_load_acquire(&head->next);
    /* smp_load_acquire: memory barrier يضمن ordering */
    return list_is_head(next, head) && (next == READ_ONCE(head->prev));
    /* بتتأكد إن next وprev متوافقين — لو مش كده، الـ list في منتصف تعديل */
}
```

بس دي كمان مش كافية وحدها — محتاج locking حقيقي (spinlock) مع IRQ.

#### الحل

```c
static irqreturn_t stm32_spi_sensor_irq(int irq, void *dev_id)
{
    struct sensor_dev *dev = dev_id;
    struct sensor_entry *entry;
    unsigned long flags;

    entry = kmalloc(sizeof(*entry), GFP_ATOMIC);
    if (!entry)
        return IRQ_HANDLED;

    fill_sensor_entry(entry, dev);

    spin_lock_irqsave(&dev->queue_lock, flags);
    list_add_tail(&entry->list, &dev->pending_queue);
    spin_unlock_irqrestore(&dev->queue_lock, flags);

    wake_up(&dev->wait_queue);
    return IRQ_HANDLED;
}

static void stm32_sensor_worker(struct work_struct *work)
{
    struct sensor_dev *dev = container_of(work, struct sensor_dev, work);
    LIST_HEAD(local_queue);  /* list مؤقتة على الـ stack */
    unsigned long flags;

    /* Splice the whole list atomically — أسرع من حذف واحدة واحدة */
    spin_lock_irqsave(&dev->queue_lock, flags);
    list_splice_init(&dev->pending_queue, &local_queue);
    spin_unlock_irqrestore(&dev->queue_lock, flags);

    /* اشتغل على الـ local copy بدون lock */
    struct sensor_entry *entry, *tmp;
    list_for_each_entry_safe(entry, tmp, &local_queue, list) {
        list_del(&entry->list);
        process_sensor_data(entry);
        kfree(entry);
    }
}
```

الحيلة الذكية: `list_splice_init()` بتنقل كل الـ pending entries لـ local list في عملية واحدة تحت الـ lock، ثم بنشتغل عليها بدون lock خالص.

#### الدرس المستفاد
`list_empty()` و`list_empty_careful()` مش بديل عن الـ locking. استخدام `list_splice_init()` لنقل القائمة كاملة تحت lock قصير أفضل بكتير من الـ lock طول فترة المعالجة.

---

### السيناريو الرابع: إصلاح Performance Bottleneck في Automotive ECU على i.MX8 بإعادة هيكلة الـ `hlist`

#### العنوان
**الـ CAN message routing على i.MX8 بطيء جدًا تحت load — الـ `list_head` اختيار غلط لـ hash table**

#### السياق
Automotive ECU بـ **i.MX8QM** بيشتغل كـ CAN bus gateway بين 4 CAN networks. الـ driver بيخزّن الـ message routing rules في data structure. في الـ testing بـ 10,000 messages/second، الـ latency بتوصل 8ms بدل الـ 1ms المطلوبة.

#### المشكلة
الـ developer الأصلي استخدم `list_head` عادية لكل الـ routing rules:

```c
struct can_router {
    struct list_head rules;  /* كل الـ rules في list واحدة */
    int rule_count;          /* بيوصل 500+ rule */
};

/* في الـ hot path — بيتنادى 10,000 مرة/ثانية */
static struct routing_rule *find_rule(struct can_router *router, u32 can_id)
{
    struct routing_rule *rule;
    /* O(n) search على كل الـ rules */
    list_for_each_entry(rule, &router->rules, list) {
        if (rule->can_id == can_id)
            return rule;
    }
    return NULL;
}
```

مع 500 rule، كل message بتعمل traverse لـ 250 nodes في المتوسط = 2.5M pointer dereferences/ثانية.

#### التحليل
الفرق بين `list_head` و`hlist_head` في `list.h`:

```c
/*
 * Double linked lists with a single pointer list head.
 * Mostly useful for hash tables where the two pointer list head is
 * too wasteful.
 * You lose the ability to access the tail in O(1).
 */
#define HLIST_HEAD_INIT { .first = NULL }
```

الـ `hlist_head` بتاخد 8 bytes (pointer واحد) بدل 16 bytes للـ `list_head` (pointer اتنين). ده بيخلي كل bucket في الـ hash table أصغر بكتير، وبيحسّن cache locality لما بنعمل scan على الـ hash table buckets.

#### الحل

```c
#define CAN_ROUTE_HASH_BITS  8
#define CAN_ROUTE_HASH_SIZE  (1 << CAN_ROUTE_HASH_BITS)  /* 256 bucket */

struct can_router {
    /* hash table: 256 * 8 bytes = 2KB فقط */
    struct hlist_head rule_hash[CAN_ROUTE_HASH_SIZE];
    spinlock_t lock;
};

static inline u32 can_id_hash(u32 can_id)
{
    return hash_32(can_id, CAN_ROUTE_HASH_BITS);
}

static int can_router_add_rule(struct can_router *router,
                                struct routing_rule *rule)
{
    u32 bucket = can_id_hash(rule->can_id);
    unsigned long flags;

    spin_lock_irqsave(&router->lock, flags);
    /* hlist_add_head: O(1) insert */
    hlist_add_head(&rule->hnode, &router->rule_hash[bucket]);
    spin_unlock_irqrestore(&router->lock, flags);
    return 0;
}

/* Hot path — الآن O(1) في المتوسط بدل O(n) */
static struct routing_rule *find_rule(struct can_router *router, u32 can_id)
{
    u32 bucket = can_id_hash(can_id);
    struct routing_rule *rule;

    hlist_for_each_entry(rule, &router->rule_hash[bucket], hnode) {
        if (rule->can_id == can_id)
            return rule;
    }
    return NULL;
}
```

النتيجة: الـ latency نزلت من 8ms لـ 0.3ms، وكل bucket بيتضمن 500/256 ≈ 2 rules في المتوسط.

```bash
# قياس الفرق
perf stat -e cache-misses,instructions ./can_gateway_test
```

#### الدرس المستفاد
**الـ `hlist_head` مصمم خصيصًا للـ hash tables**. لما محتاج O(1) lookup وعندك hash table بـ buckets كتيرة، الـ `hlist_head` أوفر في memory وأسرع في الـ cache بكتير من الـ `list_head`. الـ tradeoff الوحيد إنك بتخسر `list_is_last()` وـ O(1) tail access.

---

### السيناريو الخامس: Board Bring-Up على AM62x — تشخيص Hang أثناء الـ Boot بسبب Circular Reference في قائمة الـ Devices

#### العنوان
**الـ board بـ AM62x بتتوقف أثناء الـ boot في `device_add` — الـ `list_head` عندها circular reference مش متوقعة**

#### السياق
Custom industrial board بـ **TI AM62x** (Texas Instruments) للـ PLC applications. أول boot على kernel 6.6 custom، الـ system بيتوقف تمامًا بعد طباعة:

```
[    2.341] i2c i2c-0: Added multiplexer i2c-4
[    2.342] i2c i2c-0: Added multiplexer i2c-5
```

بعدها صمت تام — مفيش kernel panic، مفيش watchdog reset.

#### المشكلة
الـ platform device tree فيه خطأ: الـ I2C mux driver بيضيف كل child device لـ global devices list، بس في الـ custom board driver، في initialization code بيعمل:

```c
/* خطأ في board init code */
static int am62x_board_init_sensors(struct platform_device *pdev)
{
    struct am62x_board *board = platform_get_drvdata(pdev);
    struct sensor_group *group;

    list_for_each_entry(group, &board->sensor_groups, list) {
        /* بيضيف الـ group لنفس الـ list اللي بيعمل iteration عليها! */
        if (group->needs_subgroup) {
            struct sensor_group *sub = create_subgroup(group);
            list_add(&sub->list, &board->sensor_groups);  /* BUG! */
        }
    }
}
```

ده بيعمل loop لا نهائي: الـ `list_for_each_entry` بتزور كل entry بما فيها الـ entries الجديدة اللي بتتضاف، فالـ loop مبتخلصش أبدًا.

#### التحليل
تتبع الـ macro:

```c
#define list_for_each_entry(pos, head, member)              \
    for (pos = list_first_entry(head, typeof(*pos), member); \
         !list_entry_is_head(pos, head, member);            \
         pos = list_next_entry(pos, member))
```

الـ condition `!list_entry_is_head(pos, head, member)` بتتحقق إن `pos != head`. لما بنضيف element جديد بـ `list_add()` (بيضيف بعد الـ head)، الـ iteration مش هتوصله قبل ما تخلص... إلا لو كانت الـ iteration لسه في أول القائمة — في الحالة دي هتزور الـ elements الجديدة وتعمل infinite loop.

طريقة التشخيص:

```bash
# على الـ target — تتبع الـ hung task
echo 1 > /proc/sys/kernel/hung_task_panic
# أو اضبط timeout
echo 30 > /proc/sys/kernel/hung_task_timeout_secs

# شغّل من serial console
# الـ stack trace هيظهر
SysRq-T → يطبع stack traces لكل الـ processes
```

الـ stack trace كان:

```
[  32.000] Task am62x-init:333 blocked for 30s
[  32.000]  am62x_board_init_sensors+0x78/0x1c0
[  32.000]  list_for_each_entry (inline)
[  32.000]  platform_probe+0x44/0xa0
```

#### الحل

```c
static int am62x_board_init_sensors(struct platform_device *pdev)
{
    struct am62x_board *board = platform_get_drvdata(pdev);
    struct sensor_group *group;
    /* قائمة مؤقتة للـ subgroups المراد إضافتها */
    LIST_HEAD(pending_subgroups);

    /* Pass 1: اجمع كل الـ subgroups المطلوبة */
    list_for_each_entry(group, &board->sensor_groups, list) {
        if (group->needs_subgroup) {
            struct sensor_group *sub = create_subgroup(group);
            if (sub)
                /* أضف للـ pending list مش الـ main list */
                list_add_tail(&sub->list, &pending_subgroups);
        }
    }

    /* Pass 2: دمّج الـ pending list في الـ main list بعد ما خلصنا الـ iteration */
    list_splice_tail_init(&pending_subgroups, &board->sensor_groups);

    return 0;
}
```

الـ `list_splice_tail_init()` بتضيف كل الـ pending subgroups لآخر القائمة، وبتعمل `INIT_LIST_HEAD` على الـ pending list في نفس الوقت.

```bash
# للتحقق إن الـ list سليمة بعد الـ init
# عبر debugfs لو الـ driver بيعمل expose لـ list count
cat /sys/kernel/debug/am62x-board/sensor_count
```

#### الدرس المستفاد
**لا تعدّل الـ list اللي بتعمل عليها `list_for_each_entry`**. لما محتاج تضيف elements أثناء الـ iteration، اجمعها في list مؤقتة (`LIST_HEAD` على الـ stack) ثم استخدم `list_splice_init()` أو `list_splice_tail_init()` بعد ما تخلص الـ loop. ده pattern شايعه جدًا في الـ kernel، خصوصًا في الـ device initialization code.
## Phase 7: مصادر ومراجع

### توثيق رسمي للـ Kernel

| المصدر | الرابط | الوصف |
|--------|--------|--------|
| **Linked Lists in Linux** — kernel.org | [docs.kernel.org/core-api/list.html](https://docs.kernel.org/core-api/list.html) | التوثيق الرسمي لكل أنواع الـ linked list في الـ kernel |
| **The Linux Kernel API** | [kernel.org/doc/html/v5.0/core-api/kernel-api.html](https://www.kernel.org/doc/html/v5.0/core-api/kernel-api.html) | الـ API الكامل للـ core kernel بما فيه الـ list functions |
| **Using RCU to Protect Read-Mostly Linked Lists** | [kernel.org/doc/html/v6.1/RCU/listRCU.html](https://www.kernel.org/doc/html/v6.1/RCU/listRCU.html) | كيف تحمي الـ list بالـ RCU في الـ read-mostly scenarios |
| **Source on GitHub** | [github.com/torvalds/linux — include/linux/list.h](https://github.com/torvalds/linux/blob/master/include/linux/list.h) | الـ source الأصلي على GitHub |

---

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ `list.h` وتصميمها:

#### 1. [Toward a better list iterator for the kernel](https://lwn.net/Articles/887097/)
أهم مقالة حديثة — بتناقش إزاي الـ `list_for_each_entry` فيها مشاكل لما بتشيل entry من اللسته وانت بتعدي عليها، والـ proposals لتحسين الـ iterator. لازم تقراها لو بتشتغل مع list traversal في الـ kernel.

#### 2. [docs: document linked lists](https://lwn.net/Articles/1028381/)
مقالة بتتكلم عن الـ patch اللي أضاف التوثيق الرسمي للـ linked lists في `Documentation/core-api/list.rst` — مهمة لأنها بتلخص الـ design decisions.

#### 3. [Linux kernel design patterns — part 2](https://lwn.net/Articles/336255/)
**الـ design patterns** الأساسية في الـ kernel. الجزء التاني بيشرح `list_head` وازاي الـ circular doubly-linked list بتتفعّل من ملف واحد بدون `.c` — كله `inline functions`.

#### 4. [Single linked lists for Linux](https://lwn.net/Articles/10920/) / [v2](https://lwn.net/Articles/10952/)
النقاش الأصلي من 2003 لإضافة الـ `hlist` (singly-linked for hash tables). بتفهم منها ليه الـ `hlist_head` بيوفر ذاكرة في الـ hash buckets.

#### 5. [Using RCU for linked lists — a case study](https://lwn.net/Articles/610972/)
**الـ RCU** مع الـ linked list — الحالة العملية. كيف تقرأ بدون lock وتعمل update آمن.

#### 6. [Linked list start and end](https://lwn.net/Articles/908255/)
نقاش عن الـ `list_first_entry` و`list_last_entry` وتعديلات الـ boundary checks.

---

### مصادر التعليم

#### KernelNewbies

- **[FAQ/LinkedLists — kernelnewbies.org](https://kernelnewbies.org/FAQ/LinkedLists)**
  أوضح شرح للمبتدئين — بيجاوب "إزاي الـ kernel بتعمل linked list؟" بأمثلة عملية مع `container_of`.

#### Linux Insides (كتاب مفتوح المصدر)

- **[Doubly linked list — linux-insides](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-1.html)**
  فصل كامل عن `list_head` بشرح تفصيلي لكل macro مع رسوم توضيحية.

#### eLinux.org

- **[Linux Kernel Resources — elinux.org](https://elinux.org/Linux_Kernel_Resources)**
  مرجع مجمّع لكل resources الـ kernel بما فيها الـ mailing lists والـ documentation.

---

### مناقشات الـ Mailing List

| الموضوع | الرابط |
|---------|--------|
| LKML Archive (البحث عن `list.h` patches) | [lkml.org](https://lkml.org/) |
| `[git pull] core, x86: make LIST_POISON less deadly` | [linux.kernel.narkive.com](https://linux.kernel.narkive.com/16m7BWuG/git-pull-core-x86-make-list-poison-less-deadly) |
| `include/linux/poison.h: fix LIST_POISON{1,2} offset` — commit | [github.com/torvalds/linux@8a5e5e0](https://github.com/torvalds/linux/commit/8a5e5e02fc83aaf67053ab53b359af08c6c49aaf) |

لو عايز تبحث في تاريخ الـ patches:
```bash
git log --oneline -- include/linux/list.h
git log --oneline -- include/linux/poison.h
```

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 11** — Data Types in the Kernel
- بيشرح `list_head` وازاي تستخدمها عملياً في الـ driver
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 6** — Kernel Data Structures
- أفضل شرح لـ `list_head`, `hlist_head`, `rcu_head` في كتاب واحد
- بيشرح `container_of` بالتفصيل مع أمثلة من الـ kernel الحقيقي

#### Linux Device Drivers — O'Reilly (الإصدار التاني — مجاني)
- **[Linked Lists chapter](https://www.oreilly.com/library/view/linux-device-drivers/0596000081/ch10s05.html)**
  متاح أونلاين — بيشرح الـ list API في سياق الـ driver development

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 8** — Kernel Initialization
- بيستخدم `list_head` في أمثلة الـ device registration وفهم الـ kernel boot

#### Understanding the Linux Kernel — Bovet & Cesati
- **الفصل 3** — Processes
- بيشرح `task_struct` وكيف الـ process list مبنية على `list_head`

---

### مسارات البحث الإضافي

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
# على Google / DuckDuckGo
linux kernel "list_head" site:lore.kernel.org
linux kernel "list_for_each_entry_safe" use cases
linux kernel hlist_head hash table implementation
"container_of" linux kernel explanation
linux kernel rcu_list_for_each_entry_rcu
linux kernel list_lru implementation

# داخل الـ kernel source
grep -r "list_for_each_entry" drivers/net/ --include="*.c" | head -20
grep -r "INIT_LIST_HEAD" kernel/ --include="*.c" | wc -l
```

---

### ملخص المسار التعليمي الموصى بيه

```
المبتدئ:
  kernelnewbies.org/FAQ/LinkedLists
        ↓
  linux-insides: DataStructures/linux-datastructures-1
        ↓
  LDD3 Chapter 11

المتوسط:
  docs.kernel.org/core-api/list.html
        ↓
  LKD Robert Love Chapter 6
        ↓
  LWN: Linux kernel design patterns part 2

المتقدم:
  LWN: Toward a better list iterator (887097)
        ↓
  LWN: Using RCU for linked lists (610972)
        ↓
  git log -- include/linux/list.h  (تاريخ الـ patches)
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `list.h` كلها inline functions و macros، يعني مفيش حاجة exported قابلة للـ kprobe مباشرة على مستوى الـ symbol table. لكن الـ kernel بيستخدم الـ linked lists في كل حاجة، وأكتر function مثيرة للاهتمام وقابلة للـ hook هي **`__list_add_valid_or_report`** — دي الـ function اللي بتتكلم لما يحصل **list corruption** (لو الـ `CONFIG_LIST_HARDENED` enabled). بس دي تحتاج corruption فعلي عشان تتكلم.

الحل الأذكى: نعمل module يستخدم **kprobe** على `__list_add_valid_or_report` — دي exported symbol حقيقية موجودة في الـ kernel binary. لو مش موجودة (الـ kernel مبنيش بـ `CONFIG_LIST_HARDENED`)، نـ fallback لـ kprobe على `list_add` اللي بتتحول لـ `__list_add`.

بما إن `list_add` / `__list_add` هي inline، الـ kprobe الأمثل هو على **`__list_del_entry_valid_or_report`** أو أفضل من كده نعمل kprobe على **`INIT_LIST_HEAD`** — بس دي كمان inline.

الاختيار الصح: نضرب على **`module_alloc`** أو أي kernel function حقيقية بتستخدم list internally — لكن ده مش مرتبط بالملف مباشرة.

**القرار النهائي**: نعمل module يعمل الآتي:
1. يبني linked list خاصة بيه جوه الـ module نفسه.
2. يوضح عملية إضافة وحذف nodes باستخدام `list_add` / `list_del` و `list_for_each_entry`.
3. يستخدم **kprobe** على `__list_add_valid_or_report` لو متاحة، أو بديل: نعمل kprobe على أي function kernel حقيقية وتعلق على الـ list infrastructure.

الأنسب والأنظف: نعمل kprobe على **`kmalloc`** مش مرتبط، تعالى نروح للطريقة الصح.

المشكلة إن كل functions في `list.h` هي `static inline` — مش قابلة للـ kprobe مباشرة. الحل الواقعي هو:
- نعمل module يبني list حقيقية ويشتغل عليها.
- نثبت الـ kprobe على **`__list_add_valid_or_report`** (exported في kernels مع `CONFIG_LIST_HARDENED`) أو بديل: نعمل **tracepoint** لو موجود، أو نعمل module demo كامل يبني list ويعدد عليها.

الأسلوب الأنظف في السياق ده: **module يبني list خاصة بيه ويديها demo كامل** + يحاول يسجل kprobe على `__list_add_valid_or_report` لو الـ symbol موجودة. ده واقعي ومفيد تعليميًا.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * list_demo.c — Kernel module demonstrating linux/list.h API
 * Uses kprobe on __list_add_valid_or_report (CONFIG_LIST_HARDENED kernels)
 * to observe list corruption detection; also builds a live list and walks it.
 */

#include <linux/module.h>       /* module_init, module_exit, pr_info, etc. */
#include <linux/kernel.h>       /* KERN_INFO                               */
#include <linux/list.h>         /* struct list_head, list_add, list_del ... */
#include <linux/slab.h>         /* kmalloc / kfree                         */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe ...      */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("Demo: linux/list.h — list ops + kprobe on list corruption reporter");

/* -----------------------------------------------------------------------
 * 1. بنية البيانات اللي هنبني منها الـ linked list
 *    كل entry بتمثل "process simulation" عندها pid وname
 * --------------------------------------------------------------------- */
struct demo_entry {
    int  pid;
    char name[32];
    struct list_head node;   /* the hook that hangs this struct on the list */
};

/* الـ list head — نقطة الدخول للـ circular doubly linked list */
static LIST_HEAD(demo_list);

/* -----------------------------------------------------------------------
 * 2. kprobe على __list_add_valid_or_report
 *    الـ function دي بتتكلم بس لو في list corruption (prev/next مش consistent).
 *    في الحالة الطبيعية الـ pre-handler بيتنفذ لما تنادي الـ function،
 *    يعني بيثبت إن الـ hardening path اتعمل.
 *
 *    Signature:
 *      bool __list_add_valid_or_report(struct list_head *new,
 *                                      struct list_head *prev,
 *                                      struct list_head *next);
 *    الـ args جوه pt_regs:
 *      RDI = new, RSI = prev, RDX = next  (x86-64 calling convention)
 * --------------------------------------------------------------------- */
static int corruption_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs->di = new entry pointer
     * regs->si = prev pointer
     * regs->dx = next pointer
     * طباعة العناوين عشان نعرف مين اللي طلّع الـ corruption
     */
    pr_info("list_demo: [kprobe] __list_add_valid_or_report triggered!\n");
    pr_info("list_demo:   new=%px  prev=%px  next=%px\n",
            (void *)regs->di, (void *)regs->si, (void *)regs->dx);
    pr_info("list_demo:   Possible list corruption detected by kernel hardening\n");

    return 0; /* 0 = let the original function continue */
}

static struct kprobe list_kp = {
    .symbol_name   = "__list_add_valid_or_report",
    .pre_handler   = corruption_pre_handler,
};

/* -----------------------------------------------------------------------
 * 3. helper — ينشئ entry جديدة ويضيفها للـ list
 * --------------------------------------------------------------------- */
static struct demo_entry *alloc_entry(int pid, const char *name)
{
    struct demo_entry *e = kmalloc(sizeof(*e), GFP_KERNEL);
    if (!e)
        return NULL;

    e->pid = pid;
    strscpy(e->name, name, sizeof(e->name));
    INIT_LIST_HEAD(&e->node);   /* initialize before adding — good practice */
    return e;
}

/* -----------------------------------------------------------------------
 * 4. module_init
 *    - يحاول يسجل الـ kprobe (ممكن يفشل لو CONFIG_LIST_HARDENED=n)
 *    - يبني list من 4 entries
 *    - يعدد عليها ويطبع
 *    - يحذف entry واحدة ويتحقق إن الـ list اتقصرت
 * --------------------------------------------------------------------- */
static int __init list_demo_init(void)
{
    struct demo_entry *e, *tmp;
    int ret;
    int count = 0;

    pr_info("list_demo: module loading\n");

    /* --- kprobe registration (best-effort) --- */
    ret = register_kprobe(&list_kp);
    if (ret == 0) {
        pr_info("list_demo: kprobe registered on %s @ %px\n",
                list_kp.symbol_name, list_kp.addr);
    } else {
        /*
         * الـ symbol مش موجودة لو الـ kernel مبنيش بـ CONFIG_LIST_HARDENED،
         * أو لو الـ kprobes مش enabled — مش fatal، الـ module يكمل.
         */
        pr_warn("list_demo: kprobe registration failed (%d) — "
                "CONFIG_LIST_HARDENED may be disabled, continuing anyway\n", ret);
    }

    /* --- build the list --- */
    /*
     * list_add بتضيف عند الـ head (stack behavior: LIFO).
     * list_add_tail بتضيف عند الـ tail (queue behavior: FIFO).
     * هنستخدم الاتنين عشان نوضح الفرق في الـ traversal order.
     */
    e = alloc_entry(1,    "systemd");  if (e) list_add_tail(&e->node, &demo_list);
    e = alloc_entry(42,   "kworker");  if (e) list_add_tail(&e->node, &demo_list);
    e = alloc_entry(1337, "sshd");     if (e) list_add(&e->node, &demo_list); /* head */
    e = alloc_entry(9999, "bash");     if (e) list_add_tail(&e->node, &demo_list);

    pr_info("list_demo: list built — traversing:\n");

    /* --- traverse with list_for_each_entry --- */
    /*
     * list_for_each_entry يمشي من الـ head->next للأمام.
     * container_of (مستخدم جوه list_entry) بيجيب الـ struct كامل من الـ node pointer.
     */
    list_for_each_entry(e, &demo_list, node) {
        pr_info("list_demo:   [%d] pid=%-6d  name=%s\n",
                count++, e->pid, e->name);
    }
    pr_info("list_demo: total entries = %d\n", count);

    /* --- delete the "kworker" entry to demonstrate list_del --- */
    /*
     * list_for_each_entry_safe لازم نستخدمه لو هنحذف جوه الـ loop،
     * عشان بيحتفظ بـ pointer للـ next قبل الحذف.
     */
    list_for_each_entry_safe(e, tmp, &demo_list, node) {
        if (e->pid == 42) {
            pr_info("list_demo: removing entry pid=42 (kworker)\n");
            list_del(&e->node); /* unlinks from list, sets poison pointers */
            kfree(e);
            break;
        }
    }

    /* --- verify size after deletion --- */
    count = 0;
    list_for_each_entry(e, &demo_list, node)
        count++;
    pr_info("list_demo: entries after deletion = %d\n", count);

    pr_info("list_demo: init done\n");
    return 0;
}

/* -----------------------------------------------------------------------
 * 5. module_exit
 *    - يلغي تسجيل الـ kprobe
 *    - يمشي على الـ list ويحرر كل entry
 *    لازم نعمل cleanup بالترتيب عكس الـ init عشان نتجنب use-after-free
 * --------------------------------------------------------------------- */
static void __exit list_demo_exit(void)
{
    struct demo_entry *e, *tmp;

    pr_info("list_demo: module unloading\n");

    /*
     * unregister_kprobe لازم يتعمل قبل ما نحرر أي memory تانية،
     * عشان نضمن إن الـ pre_handler مش بيتنفذ في نفس الوقت اللي بنعمل فيه cleanup.
     */
    if (list_kp.addr)           /* only if registration succeeded */
        unregister_kprobe(&list_kp);

    /*
     * list_for_each_entry_safe ضروري هنا:
     * بنحذف كل node من الـ list ونحرر الـ memory بتاعتها.
     * لو استخدمنا list_for_each_entry العادي، كنا هنقرأ e->node.next بعد kfree(e) — UB.
     */
    list_for_each_entry_safe(e, tmp, &demo_list, node) {
        pr_info("list_demo: freeing pid=%d (%s)\n", e->pid, e->name);
        list_del(&e->node);
        kfree(e);
    }

    pr_info("list_demo: cleanup complete\n");
}

module_init(list_demo_init);
module_exit(list_demo_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية للـ module: `module_init`, `module_exit`, `pr_info` |
| `linux/list.h` | الـ API المحور: `list_add`, `list_del`, `list_for_each_entry` ... |
| `linux/slab.h` | `kmalloc` / `kfree` عشان نخصص الـ `demo_entry` structs ديناميكيًا |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |

#### الـ `demo_entry` struct

**الـ** `struct list_head node` هي الـ "خطاف" اللي بيربط الـ struct بالـ list — مش بيتخزن فيها data، بس الـ `container_of` بتعرف ترجع للـ struct كامل منها. ده النمط المعتاد في كل الـ kernel code.

#### الـ kprobe على `__list_add_valid_or_report`

الـ function دي بتتكلم بس لو في **list corruption** — يعني لو `prev->next != next` أو `next->prev != prev`. في الـ `pre_handler` بنطبع العناوين الثلاثة (الـ new node، والـ prev، والـ next) من الـ registers عشان نعرف مين السبب في الـ corruption. ده مفيد جدًا في debugging scenarios زي الـ use-after-free.

#### `list_add` vs `list_add_tail`

**الـ** `list_add` بتضيف بعد الـ head مباشرة (زي stack)، و`list_add_tail` بتضيف قبل الـ head (زي queue). في الـ traversal بـ `list_for_each_entry` اللي بيمشي forward، الـ `list_add_tail` بتحافظ على ترتيب الإضافة، بينما `list_add` بتعكسه.

#### `list_for_each_entry_safe` في الحذف

لو استخدمنا `list_for_each_entry` العادي وحذفنا `e` جوه الـ loop، الـ macro هيحاول يقرأ `e->node.next` في الـ iteration اللي بعدها — وده memory freed، يعني **undefined behavior**. الـ `_safe` variant بيحفظ الـ next pointer في `tmp` قبل ما تعمل أي حاجة.

#### الـ `module_exit` والترتيب

أول حاجة في الـ exit هي `unregister_kprobe` — عشان نضمن إن الـ `pre_handler` مش ممكن يتنفذ على CPU تانية في نفس الوقت اللي بنعمل فيه `kfree`. بعد كده نعمل `list_for_each_entry_safe` ونحرر كل الـ entries. ترتيب الـ cleanup مهم زي ترتيب الـ init بالظبط.

---

### طريقة البناء

```bash
# Makefile بسيط
obj-m += list_demo.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحميل وتتبع الـ output
sudo insmod list_demo.ko
sudo dmesg | grep list_demo
sudo rmmod list_demo
sudo dmesg | grep list_demo | tail -20
```

### الـ Output المتوقع

```
list_demo: module loading
list_demo: kprobe registered on __list_add_valid_or_report @ ffffffffXXXXXXXX
list_demo: list built — traversing:
list_demo:   [0] pid=1      name=systemd
list_demo:   [1] pid=42     name=kworker
list_demo:   [2] pid=9999   name=bash
list_demo:   [3] pid=1337   name=sshd
list_demo: total entries = 4
list_demo: removing entry pid=42 (kworker)
list_demo: entries after deletion = 3
list_demo: init done
```

> **ملحوظة**: الـ `sshd` (pid=1337) اتضاف بـ `list_add` (عند الـ head)، لكن بعد كده `bash` اتضاف بـ `list_add_tail`، فـ `sshd` وصل في النهاية في الترتيب الأخير.
