## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ XArray؟

تخيل إنك بتشغّل مكتبة ضخمة — فيها ملايين الكتب. محتاج تحط كل كتاب في رف برقم معين، وتعرف تجيبه بسرعة لما حد يطلبه. المشكلة: مش عارف الكتب هتكون كام ولا أرقامها هتكون إيه. لو حجزت أرفف من 0 لحد مليون — هيبقى فيه مكان ضايع بالمليارات. لو استخدمت قاموس (hash table) — مش هتعرف تمشي من كتاب للي بعده بالترتيب.

**الـ XArray** هو الحل اللي الـ Linux kernel اخترعه لده. إنه **بنية بيانات** بتتصرف كـ array ضخمة جداً من الـ pointers، بس بتنمو وبتتقلص تلقائياً حسب الاستخدام الفعلي — ومن غير ما تحجز memory للأماكن الفاضية. جوّاها داخلياً **radix tree** (شجرة ذات قاعدة ثابتة) بس مغلّفة بـ API نظيف وآمن.

---

### المشكلة اللي جت لحلها

قبل الـ XArray، الـ kernel كان بيستخدم **radix_tree** و **IDR** (ID Radix). الاتنين كانوا بيعملوا نفس الفكرة بس كل واحد بـ API مختلف، وفيهم مشاكل:

- **الـ locking** كان manual تماماً — أي غلطة = race condition
- **الـ radix_tree API** كان معقد وصعب يتعلم
- **الـ IDR** (اللي بيخصص IDs تلقائياً) كان كود منفصل خالص

**Matthew Wilcox** من Microsoft قرر يعمل abstraction واحد يجمع الكل تحت API موحد وأبسط — ده هو الـ XArray، اتضاف في **Linux 4.17** سنة 2018.

---

### القصة الكاملة: الـ page cache

أهم مستخدم للـ XArray في الـ kernel هو **page cache** — الذاكرة المؤقتة للملفات.

لما بتفتح ملف وبتقرأه، الـ kernel بيحط صفحاته في الـ RAM (الـ pages). المشكلة: كل ملف ممكن يبقى فيه ملايين الصفحات، وكل صفحة ليها index برقم صفحتها (0، 1، 2، ...). الـ kernel محتاج:

1. يجيب صفحة بسرعة بـ index (مثلاً: أجيب الصفحة رقم 500,000 من الملف ده)
2. يمشي من صفحة للصفحة التالية بترتيب (للـ readahead مثلاً)
3. يعرف الصفحات "الـ dirty" (اللي اتعدّلت) بسرعة عشان يكتبها على الديسك

الـ XArray بيقدر يعمل الثلاتة دول بكفاءة عالية، وده السبب إنه بقى الـ backbone بتاع الـ page cache بعد ما اتحوّل من **radix_tree**.

---

### الفكرة من الداخل (بدون تعقيد)

```
XArray
  |
  xa_head ──► [xa_node: 64 slots]
                     |
               ┌─────┼─────┐
             slot0  slot1  slot2 ...
              |
         [xa_node: 64 slots]  ← level تاني
              |
         pointer للـ object الفعلي
```

- الشجرة داخلياً بـ **XA_CHUNK_SIZE = 64** slot في كل node (أو 16 لو `CONFIG_BASE_SMALL`).
- كل مستوى بيضيف 6 bits من الـ index → شجرة بـ 11 مستوى تقدر تعنوَن `2^66` entry.
- لو الـ array فاضية أو فيها entry واحدة بس — `xa_head` بيشاور عليها مباشرة بدون node.
- الـ leaf nodes بتحتوي على الـ pointers الحقيقية.

---

### أنواع الـ entries اللي ممكن تتخزن

| النوع | الوصف | مثال |
|-------|--------|-------|
| **Pointer** | أي pointer محاذي لـ 4 bytes | `kmalloc()`, `alloc_page()` |
| **Value entry** | integer من 0 لـ `LONG_MAX` | `xa_mk_value(42)` |
| **Tagged pointer** | pointer + 2-bit tag | `xa_tag_pointer(p, 1)` |
| **Internal entry** | للاستخدام الداخلي فقط | retry, sibling, zero |

الـ trick: آخر 2 bits من الـ pointer بيحددوا نوعه:
- `00` → pointer عادي
- `x1` → value أو tagged pointer
- `10` → internal entry

---

### الـ Search Marks (علامات البحث)

كل entry عنده **3 marks** مستقلة (XA_MARK_0, XA_MARK_1, XA_MARK_2). الميزة إن العلامات متكررة في كل مستوى من الشجرة — يعني لو مفيش أي entry في subtree عندها mark معين، الـ node الأب بيعرف ده بدون ما يكمل النزول. ده بيخلي البحث عن الـ dirty pages مثلاً `O(marked entries)` مش `O(total entries)`.

في الـ page cache:
- **XA_MARK_0** → صفحة "dirty" (محتاجة تتكتب)
- **XA_MARK_1** → صفحة "writeback" (بتتكتب حالياً)
- **XA_MARK_2** → صفحة "workingset shadow"

---

### الـ API: Normal vs Advanced

#### Normal API — الأبسط والأكثر استخداماً

```c
struct xarray my_xa;
xa_init(&my_xa);                          /* تهيئة */

xa_store(&my_xa, 42, my_obj, GFP_KERNEL); /* تخزين */
void *obj = xa_load(&my_xa, 42);          /* استرجاع */
xa_erase(&my_xa, 42);                     /* حذف */

/* iteration */
unsigned long index;
void *entry;
xa_for_each(&my_xa, index, entry) {
    /* process each entry */
}
```

الـ locking بيتعمل automatically — مش محتاج تفكر فيه.

#### Advanced API — للـ performance-critical code

```c
XA_STATE(xas, &my_xa, start_index); /* cursor على الـ stack */

rcu_read_lock();
entry = xas_load(&xas);   /* walk the tree */
rcu_read_unlock();
```

الـ `xa_state` بيشتغل كـ cursor بيتذكر مكانك في الشجرة — بدله بدل ما تبدأ من الأول كل مرة.

---

### الـ Locking بالتفصيل

الـ XArray بيستخدم نظامين للـ concurrency:

| الموقف | الـ Lock المستخدم |
|--------|-----------------|
| قراءة فقط | RCU read lock (بدون blocking) |
| كتابة (Normal API) | `xa_lock` spinlock (داخلي تلقائي) |
| كتابة (Advanced API) | المبرمج نفسه بيمسك `xa_lock` |
| من interrupt context | `xa_lock_irq` أو `xa_lock_bh` |

الميزة الكبيرة: القراءات بتحصل بالتوازي بدون أي lock بفضل **RCU** — اللي بيخلي الـ page cache lookups سريعة جداً.

---

### Allocating XArrays — بديل الـ IDR

لو محتاج تخصص IDs تلقائياً (زي file descriptors مثلاً):

```c
DEFINE_XARRAY_ALLOC(my_ids);

int id;
xa_alloc(&my_ids, &id, my_obj, xa_limit_32b, GFP_KERNEL);
/* id الآن = أصغر index فاضي */
```

الـ XA_MARK_0 بيتحجز داخلياً لتتبع الـ slots الفاضية في الـ allocating mode.

---

### Multi-Index Entries

ميزة نادرة في بنى البيانات: تقدر تربط range من الـ indices ببعض — لما بتخزن في أي index في الـ range، كل الـ range بتتحدّث. ده مفيد جداً للـ huge pages في الـ page cache (صفحة واحدة بتغطي مثلاً 512 index).

```
indices 64..127 → نفس الـ entry (multi-index)
xa_load(xa, 70)  ↓
xa_load(xa, 90)  ↓  → نفس الـ pointer
xa_load(xa, 127) ↓
```

---

### الملفات اللي تكوّن الـ subsystem

| الملف | الدور |
|-------|-------|
| `include/linux/xarray.h` | الـ API الكامل: structs, macros, inline functions |
| `lib/xarray.c` | الـ implementation الأساسي |
| `lib/test_xarray.c` | اختبارات شاملة للـ XArray |
| `Documentation/core-api/xarray.rst` | هذا الملف — التوثيق الرسمي |
| `rust/kernel/xarray.rs` | Rust bindings للـ XArray |

### ملفات مرتبطة مهمة

| الملف | العلاقة |
|-------|---------|
| `lib/radix-tree.c` | السلف القديم للـ XArray — كثير من دواله اتحولت |
| `lib/idr.c` | الـ IDR (ID allocator) — الآن wrapper فوق XArray |
| `mm/filemap.c` | الـ page cache — أهم مستخدم للـ XArray |
| `include/linux/radix-tree.h` | الـ compatibility layer مع الـ radix tree القديم |
| `mm/workingset.c` | بيستخدم `xas_set_update()` callback للـ shadow entries |

---

### خلاصة

**الـ XArray** هو **الـ data structure الأساسي** في الـ Linux kernel لربط index رقمي بـ pointer. جمع الـ radix_tree و IDR تحت API واحد، نظيف، آمن، وسريع. قلبه **radix tree** داخلياً، بس ما تشوفه هو API بسيط مع locking تلقائي وـ RCU للقراءة. الـ page cache هو أهم مستخدم له، وكل ملف مفتوح في الـ Linux kernel فيه XArray بيربط أرقام الصفحات بالـ pages الموجودة في الـ RAM.
## Phase 2: شرح الـ XArray Framework

---

### المشكلة اللي الـ XArray جاي يحلها

الـ kernel محتاج data structure تخزن pointers مع index عددي — زي page cache اللي محتاج يربط page number بـ `struct page*`. الحلول التقليدية كلها فيها مشاكل:

| Data Structure | المشكلة |
|---|---|
| **Linked list** | O(n) للبحث، مش cache-friendly |
| **Hash table** | مش ordered، مش بتعمل iteration ترتيبي |
| **Resizable array** | عند الـ resize لازم تنسخ كل البيانات + تغير MMU mappings |
| **Radix tree (القديم)** | الـ locking معقد، الـ API مش consistent |

الـ XArray جاء يحل الكل: ordered، O(log n) lookup، بدون resize copies، ومع locking مدمج واضح.

---

### الحل: Trie بـ Chunks ثابتة + Pointer Tagging

الـ XArray في جوهره **trie** (prefix tree) حيث كل node عندها `XA_CHUNK_SIZE` slots (64 slot على 64-bit kernel افتراضيًا). الـ index بيتقسم على chunks من 6 bits كل واحد، وكل chunk بيوصل لـ level في الـ tree.

**الـ Pointer Tagging** هو السر: الـ kernel بيستغل الـ 2 bits السفلية في كل pointer (لأن aligned pointers دايمًا `00` في السفل) عشان يفرق بين أنواع الـ entries:

```
bits[1:0] = 00  → Pointer عادي
bits[1:0] = x1  → Value entry أو tagged pointer
bits[1:0] = 10  → Internal entry (node, retry, zero, error)
```

---

### المكان في الـ Kernel

```
┌─────────────────────────────────────────────────┐
│                   Kernel Subsystems              │
│                                                 │
│   Page Cache      IRQ Domain    File Descriptors│
│   (page_tree)     (irq_data)    (fdtable)       │
│       │               │               │         │
│       └───────────────┼───────────────┘         │
│                       │                         │
│              ┌────────▼────────┐                 │
│              │    XArray API   │  ← Normal API   │
│              │  (xa_store,     │  ← Advanced API │
│              │   xa_load, ...) │                 │
│              └────────┬────────┘                 │
│                       │                         │
│              ┌────────▼────────┐                 │
│              │  Radix Tree     │                 │
│              │  Implementation │                 │
│              │  (xa_node trie) │                 │
│              └────────┬────────┘                 │
│                       │                         │
│              ┌────────▼────────┐                 │
│              │  RCU + Spinlock │                 │
│              │  Subsystem      │                 │
│              └─────────────────┘                 │
└─────────────────────────────────────────────────┘
```

> **الـ RCU** (Read-Copy-Update): subsystem في الـ kernel بيسمح بـ lockless reads من خلال ضمان إن القارئ يشوف نسخة متسقة من البيانات، حتى لو thread تاني بيعدّل فيها. القارئ بيقفل `rcu_read_lock()` اللي رخيصة جدًا.

---

### التمثيل الحقيقي للـ Tree

لنقل عندنا XArray وخزنا entries في indices 0، 100، 10000:

```
struct xarray {
    spinlock_t  xa_lock;
    gfp_t       xa_flags;
    void __rcu *xa_head;   ← يشاور على xa_node أو entry مباشرة
}

           xa_head
              │
              ▼
        ┌─────────────┐  shift=12 (level 3)
        │  xa_node    │
        │  slots[64]  │
        │  [0]──►...  │  ← الـ index 0 و 100
        │  [2]──►...  │  ← الـ index 10000
        └─────────────┘
              │
         [0] ─┼─────────────────────────────────────►
              │                                      │
              ▼                                      ▼
        ┌─────────────┐  shift=6              ┌─────────────┐
        │  xa_node    │                       │  xa_node    │
        │  slots[64]  │                       │  slots[64]  │
        │  [0]──► ptr │ ← index 0             │  [36]──►ptr │ ← index 10000
        │  [1]──► ptr │ ← index 100           └─────────────┘
        └─────────────┘
```

كل `xa_node` على 64-bit system حجمها **576 bytes** وفيها 64 slot، يعني 7 nodes في صفحة واحدة 4KB.

---

### التمثيل بالأنالوجيا: مكتبة بـ نظام رفوف هرمي

تخيل مكتبة ضخمة فيها ملايين كتاب، ومحتاج تجيب كتاب برقم معين بسرعة.

| عنصر المكتبة | الـ XArray المقابل |
|---|---|
| **رقم الكتاب** | الـ `index` (unsigned long) |
| **الكتاب نفسه** | الـ `entry` (pointer أو value) |
| **نظام الرفوف الهرمي** | الـ `xa_node` tree (trie) |
| **الـ floor → section → shelf → slot** | الـ bits المختلفة في الـ index (6 bits لكل level) |
| **بطاقة فهرسة الرف** | الـ `xa_node` node |
| **الـ 64 مكان في كل رف** | الـ `slots[XA_CHUNK_SIZE]` |
| **لاصقة اللون على الكتاب** | الـ marks (XA_MARK_0/1/2) |
| **أمين المكتبة بالباب** | الـ `xa_lock` (spinlock) |
| **الزائر اللي بيقرأ بس** | الـ RCU read lock |
| **الموظف اللي بيعدّل** | الـ xa_lock write side |
| **بطاقة "محجوز"** | الـ zero entry / reserved entry |

**المهم في الأنالوجيا:** لما الزائر يبحث عن كتاب، مش محتاج يوقف كل المكتبة. بيقدر يمشي بين الرفوف (RCU) بدون لا يتعامل مع الـ guard. لكن لما موظف يضيف أو يشيل كتاب، بيقفل الرف المحدد (spinlock) مش المكتبة كلها.

الـ **marks** زي اللاصقات الملونة على الكتب — ممكن تقول "كل الكتب اللي عليها لاصقة حمراء" وتمشي عليها بسرعة لأن كل node بتعرف إذا كان في أي slot تحتها لاصقة حمراء (الـ mark بيتنشر للأعلى في الـ tree).

---

### الـ Core Abstraction: هيكل البيانات

#### `struct xarray` — الـ Root

```c
struct xarray {
    spinlock_t   xa_lock;   /* الـ lock الوحيد للـ writers */
    gfp_t        xa_flags;  /* سلوك الـ array: alloc mode، lock type */
    void __rcu  *xa_head;   /* إما NULL أو entry مباشر أو pointer لـ xa_node */
};
```

الـ `xa_head` ممكن يكون:
- `NULL` → الـ array فاضية
- pointer عادي → entry واحد في index 0 بدون tree
- pointer لـ `xa_node` → tree كاملة

#### `struct xa_node` — الـ Internal Node

```c
struct xa_node {
    unsigned char   shift;      /* كام bit متبقي للـ levels تحته */
    unsigned char   offset;     /* موقع الـ node دي في الـ parent */
    unsigned char   count;      /* عدد الـ non-NULL slots */
    unsigned char   nr_values;  /* عدد الـ value entries */
    struct xa_node __rcu *parent; /* NULL لو root node */
    struct xarray   *array;     /* pointer للـ xarray صاحبها */
    union {
        struct list_head private_list; /* لاستخدام الـ page cache */
        struct rcu_head  rcu_head;     /* للـ deferred free */
    };
    void __rcu      *slots[XA_CHUNK_SIZE]; /* 64 slot على 64-bit */
    union {
        unsigned long tags[XA_MAX_MARKS][XA_MARK_LONGS];
        unsigned long marks[XA_MAX_MARKS][XA_MARK_LONGS]; /* نفس الشيء */
    };
};
```

الـ **marks** في كل node هي bitmask فيها bit لكل slot. لو أي entry في subtree معمول عليه mark، الـ bit المقابل في الـ node الأعلى بيتضبط. ده بيسمح بـ O(log n) search للـ marked entries بدل O(n).

#### `struct xa_state` — الـ Cursor (Advanced API)

```c
struct xa_state {
    struct xarray   *xa;        /* الـ array اللي بنشتغل عليه */
    unsigned long    xa_index;  /* الـ index الحالي */
    unsigned char    xa_shift;  /* للـ multi-index entries */
    unsigned char    xa_sibs;   /* عدد الـ siblings */
    unsigned char    xa_offset; /* offset في الـ current node */
    unsigned char    xa_pad;
    struct xa_node  *xa_node;   /* الـ node الحالي أو error/special state */
    struct xa_node  *xa_alloc;  /* pre-allocated node جاهز للاستخدام */
    xa_update_node_t xa_update; /* callback لما node تتحدث */
    struct list_lru  *xa_lru;
};
```

الـ `xa_node` في الـ xa_state بيحمل أكتر من معنى حسب قيمته:

```
xa_node == NULL       → إحنا في الـ root، entry واحد بس
xa_node == XAS_RESTART → محتاج نبدأ من الأول (الـ state قديم)
xa_node == XAS_BOUNDS  → الـ index برّا الـ tree
xa_node == XA_ERROR(e) → في error حصل، قيمته encoded في الـ pointer
xa_node == valid ptr   → إحنا في node معينة في الـ tree
```

---

### علاقة الـ Structs ببعض

```
struct xarray
    │
    │ xa_head ──────────────────────────────────────┐
    │                                               │
    └── spinlock xa_lock                            ▼
                                         ┌─────────────────┐
                                         │   xa_node (L2)  │
                                         │   shift = 6     │
                                         │   parent = NULL │
                                         │   slots[64] ────┼──► xa_node (L1) [0]
                                         │                 │    xa_node (L1) [1]
                                         │   marks[3][1]   │    ...
                                         └─────────────────┘
                                                   │
                              ┌────────────────────┤
                              ▼                    ▼
                   ┌─────────────────┐   ┌─────────────────┐
                   │   xa_node (L1)  │   │   xa_node (L1)  │
                   │   shift = 0     │   │   shift = 0     │
                   │   parent ──────►│   │   parent ──────►│
                   │   slots[64]:    │   │   slots[64]:    │
                   │    [0] → ptr0   │   │    [5] → ptr    │
                   │    [1] → ptr1   │   │    ...          │
                   └─────────────────┘   └─────────────────┘

                              ▲
struct xa_state ──────────────┘
    xa_node → يشاور على L1 node دي
    xa_index = الـ index الحالي
    xa_offset = 1   ← الـ slot الحالي
```

---

### الـ Entry Encoding بالتفصيل

كل `void *` في الـ slots ممكن يكون:

```c
/* bits[1:0] = 00: Pointer عادي (aligned) */
void *ptr = kmalloc(...);    /* آمن للتخزين مباشرة */

/* bits[1:0] = x1: Value entry */
void *val = xa_mk_value(42); /* = (42 << 1) | 1 = 0x55 */
/* استخراج: xa_to_value(val) = val >> 1 = 42 */

/* bits[1:0] = 10: Internal entry */
#define XA_ZERO_ENTRY    xa_mk_internal(257)  /* = (257 << 2) | 2 */
/* يظهر كـ NULL للـ Normal API، لكن يحجز المكان */

/* Internal entries خاصة: */
/* 0-62:  Sibling entries (multi-index) */
/* 256:   Retry entry - thread تاني بيعدّل */
/* 257:   Zero entry - محجوز */
/* < 0:   Error entries */
```

---

### الـ Multi-Index Entries

ده feature متقدمة بيخلي range من الـ indices يشاوروا على entry واحد. مثلًا page cache بيستخدمه لـ huge pages.

```
XArray مع multi-index entry للـ range [64, 127]:

Level 1 node (shift=6):
┌────────────────────────────────────────────┐
│ slot[1] → actual page_ptr  ← canonical    │
│ slot[2] → sibling(1)       ← يشاور على [1]│
│ slot[3] → sibling(1)       ← يشاور على [1]│
│ ...                                        │
│ slot[63] → sibling(1)                      │
└────────────────────────────────────────────┘

xa_load(xa, 80) → يتبع sibling(1) → يرجع نفس الـ page_ptr
xa_load(xa, 64) → يرجع page_ptr مباشرة
```

التقسيم `xas_split()` بيحوّل entry واحد كبير لـ entries أصغر. الفرق بين الطريقتين:

- `xas_split_alloc() + xas_split()`: بتقسّم كل الـ range دفعة واحدة (uniform)، تحتاج `2^(order-6)` nodes
- `xas_try_split()`: بتقسّم بشكل iterative من حول index معين فقط، تحتاج node واحدة بس في الغالب

---

### Normal API vs Advanced API

#### Normal API — الغلاف الآمن

```c
/* كل function دي تعمل lock/unlock تلقائي */
void *xa_load(struct xarray *xa, unsigned long index);
void *xa_store(struct xarray *xa, unsigned long index, void *entry, gfp_t gfp);
void *xa_erase(struct xarray *xa, unsigned long index);
void *xa_cmpxchg(struct xarray *xa, unsigned long index,
                 void *old, void *entry, gfp_t gfp);
int   xa_insert(struct xarray *xa, unsigned long index,
                void *entry, gfp_t gfp);
```

مثال عملي — driver بيخزن IRQ handlers:

```c
static DEFINE_XARRAY(irq_handlers);

/* تسجيل handler */
int register_irq_handler(unsigned long irq, struct irq_handler *h)
{
    /* xa_store بتعمل xa_lock/unlock داخليًا */
    void *old = xa_store(&irq_handlers, irq, h, GFP_KERNEL);
    return xa_err(old);  /* 0 لو نجح */
}

/* استدعاء handler من interrupt context */
void handle_irq(unsigned long irq)
{
    struct irq_handler *h;

    /* xa_load بتاخد rcu_read_lock() بس */
    rcu_read_lock();
    h = xa_load(&irq_handlers, irq);
    if (h)
        h->handler(irq, h->data);
    rcu_read_unlock();
}
```

#### Advanced API — الـ xa_state cursor

```c
/* XA_STATE بتعرف cursor على الـ stack */
XA_STATE(xas, &my_array, start_index);

xa_lock(&my_array);
xas_for_each(&xas, entry, last_index) {
    if (xas_retry(&xas, entry))  /* تعامل مع internal entries */
        continue;
    process(entry);
}
xa_unlock(&my_array);
```

مثال page cache lockless lookup:

```c
/* بدون lock تقيل، بس RCU */
rcu_read_lock();
XA_STATE(xas, &mapping->i_pages, index);
page = xas_load(&xas);
if (xa_is_value(page)) {
    /* shadow entry - الصفحة اتطردت من الـ cache */
    handle_shadow(page);
}
rcu_read_unlock();
```

---

### الـ Allocating XArray

لما تحتاج الـ kernel يختار الـ index ليك (زي IDR القديم):

```c
/* تعريف XArray بـ alloc mode */
DEFINE_XARRAY_ALLOC(my_ids);

/* تخزين مع auto-assign للـ ID */
u32 id;
int ret = xa_alloc(&my_ids, &id, my_object, xa_limit_32b, GFP_KERNEL);
/* id الآن = أصغر index فاضي >= 0 */

/* تحرير */
xa_erase(&my_ids, id);
```

في الـ alloc mode:
- `XA_MARK_0` (= `XA_FREE_MARK`) محجوزة داخليًا لتتبع الـ free slots
- تخزين `NULL` = reserved (مش free)
- حرر بـ `xa_erase()` مش `xa_store(NULL)`

---

### الـ Locking بالتفصيل

الـ XArray بيفصل بين:

```
READ operations (رخيصة):           WRITE operations (أثقل):
────────────────────────           ──────────────────────────
xa_load()    → rcu_read_lock()    xa_store()  → spinlock
xa_find()    → rcu_read_lock()    xa_erase()  → spinlock
xa_for_each()→ rcu_read_lock()    xa_alloc()  → spinlock

NO LOCK:
xa_empty()   → atomic read من xa_head
xa_marked()  → atomic read من xa_flags
```

لو بتعدّل من interrupt context:

```c
/* Initialization */
xa_init_flags(&my_array, XA_FLAGS_LOCK_IRQ);

/* من process context */
xa_store_irq(&my_array, idx, ptr, GFP_KERNEL);
/* equivalent: xa_lock_irq() + __xa_store() + xa_unlock_irq() */

/* من interrupt handler */
xa_lock(&my_array);          /* بدون irq لأن إحنا في interrupt */
__xa_erase(&my_array, idx);
xa_unlock(&my_array);
```

---

### ما بيملكه الـ XArray vs ما بيفوّضه

| الجانب | الـ XArray يملكه | بيفوّضه للـ Caller |
|---|---|---|
| **Memory layout** | هيكل الـ trie وإدارة الـ xa_node nodes | محتوى الـ entries نفسها |
| **Concurrency** | الـ spinlock والـ RCU integration | اختيار متى تاخد الـ lock وإمتى تعمل unlock |
| **Marks** | نشر الـ marks في الـ tree | معنى كل mark (DIRTY؟ WRITEBACK؟) |
| **Memory allocation** | allocation وdeallocation للـ xa_node | الـ GFP flags المناسبة |
| **Error encoding** | تحويل errors لـ internal entries | معالجة الـ errors |
| **Index assignment** | في alloc mode: اختيار أصغر free index | في normal mode: الـ caller يحدد الـ index |
| **Iteration order** | ضمان ascending order | ما بيعمل descending iteration |
| **Entry lifecycle** | متى يُنشأ وُيحذف الـ node | lifecycle الـ object المخزون نفسه |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet: Flags, Enums, و Constants المهمة

#### XA_FLAGS — بتتحدد في `xa_init_flags()` أو `DEFINE_XARRAY_FLAGS()`

| Flag | القيمة | الغرض |
|---|---|---|
| `XA_FLAGS_LOCK_IRQ` | `1` | الـ `xa_lock` يجب يكون محمي من interrupt context |
| `XA_FLAGS_LOCK_BH` | `2` | الـ `xa_lock` يجب يكون محمي من softirq context |
| `XA_FLAGS_TRACK_FREE` | `4` | بيتابع الـ slots الفاضية (لازم لـ alloc mode) |
| `XA_FLAGS_ZERO_BUSY` | `8` | الـ zero entry تُعتبر "مشغولة" (للـ ALLOC1) |
| `XA_FLAGS_ALLOC_WRAPPED` | `16` | الـ cyclic allocator لف من الأول |
| `XA_FLAGS_ACCOUNT` | `32` | يستخدم memory accounting للـ kmem cgroup |
| `XA_FLAGS_ALLOC` | `TRACK_FREE + MARK(FREE_MARK)` | الـ XArray يخصص IDs من 0 |
| `XA_FLAGS_ALLOC1` | `TRACK_FREE + ZERO_BUSY` | الـ XArray يخصص IDs من 1 |

#### xa_mark_t — بتُستخدم مع `xa_set_mark()` وغيرها

| Mark | الرقم | ملاحظة |
|---|---|---|
| `XA_MARK_0` | `0` | في الـ alloc mode بتكون `XA_FREE_MARK` — محجوزة |
| `XA_MARK_1` | `1` | متاحة للاستخدام |
| `XA_MARK_2` | `2` | متاحة للاستخدام |
| `XA_PRESENT` | `8` | internal — بتمثل أي entry موجودة (غير NULL) |
| `XA_FREE_MARK` | `= XA_MARK_0` | بتحدد الـ slot الفاضية في alloc mode |

#### enum xa_lock_type

| قيمة | المعنى |
|---|---|
| `XA_LOCK_IRQ = 1` | الـ lock يغلق الـ interrupts |
| `XA_LOCK_BH = 2` | الـ lock يغلق الـ softirqs |

#### Internal Entry Values (الـ bottom bits بتحدد النوع)

| Pattern | النوع |
|---|---|
| `xx00` | Pointer عادي (aligned 4-byte) |
| `xx10` | Internal entry |
| `xxx1` | Value entry أو tagged pointer |
| Internal `0–62` | Sibling entries |
| Internal `256` | Retry entry (`XA_RETRY_ENTRY`) |
| Internal `257` | Zero entry (`XA_ZERO_ENTRY`) |
| Internal negative (`-4094` to `-2`) | Encoded errno |

#### XA_CHUNK Constants

| Macro | القيمة (64-bit) | الغرض |
|---|---|---|
| `XA_CHUNK_SHIFT` | `6` (أو `4` لو `CONFIG_BASE_SMALL`) | عدد bits لكل level |
| `XA_CHUNK_SIZE` | `64` | عدد slots في كل node |
| `XA_CHUNK_MASK` | `63` | mask لاستخراج offset |
| `XA_MAX_MARKS` | `3` | عدد الـ mark bits لكل slot |
| `XA_CHECK_SCHED` | `4096` | كل كام iteration يجب تعمل reschedule |

#### xa_state Special Node Values

| ثابت | المعنى |
|---|---|
| `XAS_RESTART` | الـ cursor لسه ما مشيش (أو اتعمله reset) |
| `XAS_BOUNDS` | وصلنا لأبعد من الـ tree المخصصة |
| `XA_ERROR(errno)` | الـ cursor بيخزن error code |

---

### 1. الـ Structs المهمة

#### 1.1 `struct xarray` — العمود الفقري

**الغرض:** الـ anchor الوحيد اللي بيشوفه المستخدم — بيمثل الـ XArray كله.

```c
struct xarray {
    spinlock_t  xa_lock;   /* يحمي كل محتوى الـ array عند الكتابة */
    /* private: */
    gfp_t       xa_flags;  /* XA_FLAGS_* — بيحدد السلوك والـ GFP hint */
    void __rcu *xa_head;   /* NULL لو فاضي، pointer للـ entry لو index=0 فقط،
                              أو pointer لـ xa_node لو في أكتر من entry */
};
```

| Field | النوع | الدور |
|---|---|---|
| `xa_lock` | `spinlock_t` | الـ lock الوحيد للكتابة — بيحميه الـ RCU للقراءة |
| `xa_flags` | `gfp_t` | سلوك الـ array + GFP hints لتخصيص الذاكرة |
| `xa_head` | `void __rcu *` | نقطة الدخول للشجرة — بتتغير لما الشجرة تكبر |

**الـ `xa_head` بيمثل 3 حالات:**
- `NULL` → الـ array فاضي تماماً
- pointer عادي → في entry وحدة بس عند index 0
- internal entry (tagged `| 2`) → pointer لـ `xa_node` (الشجرة اتبنت)

---

#### 1.2 `struct xa_node` — عقدة الشجرة الداخلية

**الغرض:** بيمثل level واحد في الـ radix tree الداخلية. المستخدم ما بيشوفهاش مباشرة.

```c
struct xa_node {
    unsigned char  shift;       /* عدد الـ bits المتبقية في كل slot لهذا المستوى */
    unsigned char  offset;      /* موقع هذه الـ node في slots الـ parent */
    unsigned char  count;       /* عدد الـ slots غير NULL في هذه الـ node */
    unsigned char  nr_values;   /* عدد الـ value entries + siblings لـ values */
    struct xa_node __rcu *parent; /* NULL لو root node */
    struct xarray  *array;      /* الـ XArray اللي تنتمي له */
    union {
        struct list_head private_list; /* للـ workingset/shadow في page cache */
        struct rcu_head  rcu_head;     /* لتحرير الـ node بشكل آمن مع RCU */
    };
    void __rcu  *slots[XA_CHUNK_SIZE];                    /* 64 slot */
    union {
        unsigned long tags[XA_MAX_MARKS][XA_MARK_LONGS];  /* alias قديم */
        unsigned long marks[XA_MAX_MARKS][XA_MARK_LONGS]; /* 3 marks × 1 long */
    };
};
```

| Field | الغرض |
|---|---|
| `shift` | عدد bits لهذا المستوى — leaf nodes عندها `shift=0` |
| `offset` | موضع هذه الـ node في الـ parent (`slots[offset]`) |
| `count` | لو وصل 0 يتحذف الـ node |
| `nr_values` | بيُستخدم لتتبع الـ value entries بشكل منفصل |
| `parent` | للصعود في الشجرة — NULL في الـ root |
| `array` | back-pointer للـ `xarray` المالك |
| `slots[]` | الـ 64 slot — بتحتوي pointers لـ nodes أعمق أو entries للمستخدم |
| `marks[]` | bitmap: كل mark بتاخد `XA_MARK_LONGS` longs = 1 long لـ 64 slot |

---

#### 1.3 `struct xa_state` — الـ cursor للـ Advanced API

**الغرض:** بيمشي في الشجرة ويحتفظ بموضعه — بيتعلن على الـ stack.

```c
struct xa_state {
    struct xarray  *xa;         /* الـ array اللي بنشتغل عليه */
    unsigned long   xa_index;   /* الـ index المستهدف */
    unsigned char   xa_shift;   /* لـ multi-index: الـ shift للـ order */
    unsigned char   xa_sibs;    /* لـ multi-index: عدد الـ siblings - 1 */
    unsigned char   xa_offset;  /* offset داخل الـ node الحالية */
    unsigned char   xa_pad;     /* padding لتحسين كود الـ GCC */
    struct xa_node *xa_node;    /* الـ node الحالية (أو XAS_RESTART/XAS_BOUNDS/XA_ERROR) */
    struct xa_node *xa_alloc;   /* node مخصصة مسبقاً للاستخدام عند الحاجة */
    xa_update_node_t xa_update; /* callback لما الـ node تتحدث (page cache workingset) */
    struct list_lru *xa_lru;    /* لـ LRU tracking في الـ workingset */
};
```

| Field | الغرض |
|---|---|
| `xa` | الـ XArray المستهدف |
| `xa_index` | الـ index الحالي — بيتحدث أثناء الـ iteration |
| `xa_shift` | للـ multi-index entries: يحدد alignment |
| `xa_sibs` | للـ multi-index: عدد الـ sibling slots - 1 |
| `xa_offset` | موضع الـ slot في الـ xa_node الحالية |
| `xa_node` | إما node حقيقية، أو `XAS_RESTART`، أو `XAS_BOUNDS`، أو error |
| `xa_alloc` | node تم تخصيصها مسبقاً لتجنب الـ allocation تحت الـ lock |
| `xa_update` | callback بيتنادى لما count الـ node يتغير |
| `xa_lru` | مستخدمة في page cache لتتبع shadow entries |

---

#### 1.4 `struct xa_limit` — نطاق الـ ID للـ Allocating XArrays

```c
struct xa_limit {
    u32 max;  /* أقصى ID (inclusive) */
    u32 min;  /* أدنى ID (inclusive) */
};
```

**الاستخدام:**
```c
/* أمثلة جاهزة */
xa_limit_32b  // [0, UINT_MAX]
xa_limit_31b  // [0, INT_MAX]
xa_limit_16b  // [0, USHRT_MAX]

/* أو custom */
XA_LIMIT(100, 999)  // [100, 999]
```

---

### 2. علاقات الـ Structs

#### العلاقات الأساسية:

```
struct xarray
  ├── xa_lock (spinlock_t)
  ├── xa_flags (gfp_t)
  └── xa_head ──────────────────┐
                                 │
              ┌──────────────────▼──────────────────┐
              │           struct xa_node             │
              │  ┌──────────────────────────────┐   │
              │  │ parent ──────────────────────┼───┼──► parent xa_node (أو NULL)
              │  │ array  ──────────────────────┼───┼──► struct xarray
              │  │ shift, offset, count         │   │
              │  │ slots[64]:                   │   │
              │  │   [0] ──► xa_node (deeper)   │   │
              │  │   [1] ──► user pointer       │   │
              │  │   [2] ──► value entry (|1)   │   │
              │  │   [3] ──► sibling entry      │   │
              │  │   [4] ──► retry/zero entry   │   │
              │  │   ...                        │   │
              │  │ marks[3][1]:                 │   │
              │  │   marks[0] = 0b...  (bitmap) │   │
              │  │   marks[1] = 0b...           │   │
              │  │   marks[2] = 0b...           │   │
              │  └──────────────────────────────┘   │
              └──────────────────────────────────────┘

struct xa_state (على الـ stack)
  ├── xa ────────────────────────────► struct xarray
  ├── xa_index (الـ index الحالي)
  ├── xa_node ────────────────────────► struct xa_node (الـ node الحالية)
  ├── xa_alloc ───────────────────────► struct xa_node (pre-allocated)
  └── xa_update ──────────────────────► callback function
```

---

### 3. رسم الـ Tree Structure — كيف بيبان الـ XArray من الداخل

الـ XArray بيبني radix tree لها base = 64 (على 64-bit systems). كل level بيغطي 6 bits من الـ index.

```
xarray.xa_head
      │
      ▼ (لو index واحد فقط عند 0)
  [direct pointer to entry]

      أو

      ▼ (لو في أكتر من index أو index != 0)
┌─────────────────────────────────┐
│  xa_node  (shift=12, level=2)  │  ← root node: تغطي bits 12-17
│  slots[64]:                    │
│  [i] ──► xa_node (shift=6)    │  ← middle node: تغطي bits 6-11
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  xa_node  (shift=6, level=1)   │
│  slots[64]:                    │
│  [j] ──► xa_node (shift=0)    │  ← leaf node: تغطي bits 0-5
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  xa_node  (shift=0, level=0)   │  ← leaf
│  slots[64]:                    │
│  [k] ──► user_pointer          │  ← الـ entry الفعلي
└─────────────────────────────────┘
```

**مثال: index = 0x1234 (= 4660 decimal)**
```
binary: 00 0001 0010 0011 0100
              │      │      │
        bits 12-17  6-11  0-5
             = 1    = 8   = 52

root_node->slots[1] → mid_node→slots[8] → leaf_node→slots[52] → entry
```

**الـ Marks بتتكرر في كل level:**
```
leaf_node->marks[0][0] bit 52 = 1
    ↓ (propagated up)
mid_node->marks[0][0] bit 8 = 1
    ↓
root_node->marks[0][0] bit 1 = 1
    ↓
xarray->xa_flags bit (XA_FLAGS_MARK(0)) = 1
```
بكده `xa_marked(xa, XA_MARK_0)` يرجع true بدون ما تمشي في الشجرة.

---

### 4. Lifecycle Diagrams

#### 4.1 Lifecycle الـ XArray نفسه

```
[Static / Compile-time]                 [Dynamic / Runtime]
        │                                       │
DEFINE_XARRAY(name)              xa_init(xa) or xa_init_flags(xa, flags)
        │                                       │
        └───────────────┬───────────────────────┘
                        │
                        ▼
              ┌─────────────────┐
              │  xarray EMPTY   │  xa_head = NULL
              │  xa_flags set   │  xa_lock initialized
              └────────┬────────┘
                       │
          ┌────────────┼──────────────────┐
          │            │                  │
     xa_store()    xa_alloc()        xa_reserve()
          │            │                  │
          ▼            ▼                  ▼
              ┌─────────────────┐
              │  xarray IN USE  │  xa_head → entry or xa_node tree
              │  entries stored │
              └────────┬────────┘
                       │
          ┌────────────┼──────────────────┐
          │            │                  │
     xa_erase()   xa_store(NULL)    xa_destroy()
          │            │                  │
          └────────────┘                  │
                       │                  ▼
              ┌─────────────────┐  ┌─────────────────┐
              │  xarray EMPTY   │  │  xarray DEAD    │
              │  (reusable)     │  │  (all freed)    │
              └─────────────────┘  └─────────────────┘
```

#### 4.2 Lifecycle الـ xa_node

```
[طلب store لـ index جديد يحتاج node]
        │
        ▼
  kmem_cache_alloc(xa_node_cachep)
        │
        ▼
  xa_node_init(node, parent)
    - node->shift = parent->shift - XA_CHUNK_SHIFT
    - node->offset = offset في الـ parent
    - node->parent = parent
    - node->array = xa
        │
        ▼
  parent->slots[offset] = xa_mk_node(node)  [tagged as internal]
        │
        ▼
  ┌─────────────────────────────────┐
  │  xa_node ACTIVE                 │
  │  count > 0                      │
  │  بيخدم operations               │
  └──────────────┬──────────────────┘
                 │
          عمليات الحذف
          count-- لكل erase
                 │
                 ▼ count == 0
  xa_node_free(node)  OR  call_rcu(&node->rcu_head, ...)
        │
        ▼
  kmem_cache_free(xa_node_cachep, node)
```

#### 4.3 Lifecycle الـ xa_state (Advanced API)

```
XA_STATE(xas, xa, index)    ← بيتعلن على الـ stack
    xa_node = XAS_RESTART   ← ما مشيش لسه
        │
        ▼
  xas_load(xas)  أو  xas_store(xas, entry)
    → يمشي من xa_head للـ node الصح
    → xa_node يشاور على الـ leaf node
    → xa_offset = موضع الـ slot
        │
        ▼
  ┌─────────────────────────────────┐
  │  xas VALID                      │
  │  xas->xa_node → leaf xa_node    │
  │  xas->xa_offset = slot index    │
  └──────────────┬──────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
xas_next()  xas_find()  xas_pause()
    │                        │
    ▼                        ▼
  [moves to               xa_node = XAS_RESTART
  next index]             [safe to drop lock]
                               │
                          xa_lock dropped
                               │
                          xa_lock acquired
                               │
                          xas_load() again
                          [re-walks from root]
```

---

### 5. Call Flow Diagrams

#### 5.1 `xa_store()` — الـ Normal API

```
caller: xa_store(xa, index, entry, gfp)
  │
  ├─ might_alloc(gfp)         ← lockdep hint
  ├─ xa_lock(xa)              ← spin_lock(&xa->xa_lock)
  ├─ __xa_store(xa, index, entry, gfp)
  │     │
  │     ├─ XA_STATE(xas, xa, index)
  │     ├─ xas_store(&xas, entry)
  │     │     │
  │     │     ├─ xas_create(&xas, ...)    ← بيخلق الـ nodes اللازمة
  │     │     │     │
  │     │     │     ├─ [لو الـ node مش موجودة]
  │     │     │     │     kmem_cache_alloc(xa_node_cachep, gfp)
  │     │     │     │         ← لو فشل: xas_set_err(xas, -ENOMEM)
  │     │     │     │
  │     │     │     └─ يربط الـ nodes بالشجرة
  │     │     │
  │     │     ├─ slot = &node->slots[xas->xa_offset]
  │     │     ├─ old = *slot
  │     │     ├─ rcu_assign_pointer(*slot, entry)  ← atomic publish
  │     │     │
  │     │     ├─ [update marks propagation]
  │     │     │
  │     │     └─ [shrink tree if needed — xa_delete_node()]
  │     │
  │     └─ return old_entry or xa_err()
  │
  ├─ xa_unlock(xa)            ← spin_unlock(&xa->xa_lock)
  └─ return prev_entry
```

#### 5.2 `xa_load()` — القراءة بدون lock

```
caller: xa_load(xa, index)
  │
  ├─ rcu_read_lock()
  ├─ xas_load(&xas)
  │     │
  │     ├─ entry = xa_head(xa)   ← rcu_dereference
  │     │
  │     ├─ [لو xa_is_node(entry)]
  │     │     loop:
  │     │       node = xa_to_node(entry)
  │     │       offset = (index >> node->shift) & XA_CHUNK_MASK
  │     │       entry = xa_entry(xa, node, offset)  ← rcu_dereference
  │     │       [لو xa_is_node(entry)] → continue loop
  │     │
  │     ├─ [لو xa_is_retry(entry)]
  │     │       xas_reset(xas) → retry from root
  │     │
  │     └─ return entry (أو NULL)
  │
  ├─ entry = xas_load()
  ├─ [لو xa_is_zero(entry)] → entry = NULL   ← zero entries = NULL للـ user
  ├─ rcu_read_unlock()
  └─ return entry
```

#### 5.3 `xa_alloc()` — الـ ID Allocator

```
caller: xa_alloc(xa, &id, entry, limit, gfp)
  │
  ├─ xa_lock(xa)
  ├─ __xa_alloc(xa, &id, entry, limit, gfp)
  │     │
  │     ├─ [find free slot]
  │     │     xas_find_marked(&xas, limit.max, XA_FREE_MARK)
  │     │         → بيدور على أول slot فيها XA_FREE_MARK=1
  │     │         → الـ marks المدمجة بتخلي البحث سريع O(log n)
  │     │
  │     ├─ *id = xas.xa_index
  │     │
  │     ├─ xas_store(&xas, entry)
  │     │     → يحط الـ entry في الـ slot
  │     │
  │     └─ xas_clear_mark(&xas, XA_FREE_MARK)
  │           → يعلم الـ slot "مشغولة"
  │
  ├─ xa_unlock(xa)
  └─ return 0 أو -ENOMEM أو -EBUSY
```

#### 5.4 `xas_store()` مع Memory Retry Pattern

```
/* النمط الصحيح لـ Advanced API مع memory allocation */

XA_STATE(xas, xa, index);
void *entry = new_entry;

do {
    xas_lock(&xas);                    /* spin_lock */
    xas_store(&xas, entry);            /* حاول الـ store */
    xas_unlock(&xas);                  /* spin_unlock */
} while (xas_nomem(&xas, GFP_KERNEL));
/*
 * xas_nomem():
 *   لو في ENOMEM → يحاول يخصص node → يحفظها في xas->xa_alloc → يرجع true
 *   لو مفيش error → يحرر أي ذاكرة محجوزة مسبقاً → يرجع false
 *   التالي iteration: xas_store يستخدم xas->xa_alloc بدل kmalloc تحت الـ lock
 */
```

#### 5.5 Iteration Flow — `xa_for_each()`

```
xa_for_each(xa, index, entry)
  │
  ├─ index = 0
  ├─ entry = xa_find(xa, &index, ULONG_MAX, XA_PRESENT)
  │     │
  │     ├─ rcu_read_lock()
  │     ├─ XA_STATE(xas, xa, index)
  │     ├─ xas_find(&xas, ULONG_MAX)
  │     │     → يمشي الشجرة forward
  │     │     → بيتخطى الـ NULL slots
  │     │     → بيرجع أول non-NULL entry
  │     ├─ rcu_read_unlock()
  │     └─ return entry (+ index updated)
  │
  └─ [loop body runs]
       │
       ├─ entry = xa_find_after(xa, &index, ULONG_MAX, XA_PRESENT)
       │     → نفس xa_find لكن يبدأ من index+1
       │
       └─ [تكرار لحد ما entry = NULL]
```

---

### 6. استراتيجية الـ Locking

#### 6.1 جدول شامل للـ Locking

| Operation | Lock المطلوب | ملاحظة |
|---|---|---|
| `xa_empty()` | لا شيء | atomic read |
| `xa_marked()` | لا شيء | atomic read من `xa_flags` |
| `xa_load()` | `rcu_read_lock()` | بتاخده وترجعه تلقائياً |
| `xa_for_each()` | `rcu_read_lock()` | داخلي تلقائي |
| `xa_find()`, `xa_find_after()` | `rcu_read_lock()` | داخلي تلقائي |
| `xa_get_mark()` | `rcu_read_lock()` | داخلي تلقائي |
| `xa_store()` | `xa_lock` | داخلي تلقائي |
| `xa_erase()` | `xa_lock` | داخلي تلقائي |
| `xa_cmpxchg()` | `xa_lock` | داخلي تلقائي |
| `xa_alloc()` | `xa_lock` | داخلي تلقائي |
| `xa_set_mark()` | `xa_lock` | داخلي تلقائي |
| `xa_destroy()` | `xa_lock` | داخلي تلقائي |
| `__xa_store()` | caller must hold `xa_lock` | الـ `__` prefix = المستخدم مسؤول |
| `__xa_erase()` | caller must hold `xa_lock` | |
| `__xa_alloc()` | caller must hold `xa_lock` | |
| `xas_load()` | `rcu_read_lock()` أو `xa_lock` | Advanced API |
| `xas_store()` | `xa_lock` | Advanced API |
| `xas_find()` | `rcu_read_lock()` أو `xa_lock` | Advanced API |

#### 6.2 Lock Variants حسب الـ Context

```
Context المستخدم        Function المناسبة
─────────────────────── ────────────────────────────────
Process context only    xa_store(), xa_lock()
Process + softirq       xa_store_bh(), xa_lock_bh()
                        أو init بـ XA_FLAGS_LOCK_BH
Process + interrupt     xa_store_irq(), xa_lock_irq()
                        أو init بـ XA_FLAGS_LOCK_IRQ
Interrupt handler       xa_lock() فقط (الـ IRQ مغلقة أصلاً)
Softirq handler         xa_lock() فقط
```

#### 6.3 ترتيب الـ Locks (Lock Ordering) — منع الـ Deadlock

```
[Rules رسمية:]

1. xa_lock < أي lock تاني تحته في الـ hierarchy
   → لو في mutex فوق xa_lock، اقفل الـ mutex الأول

2. لو بتستخدم xa_lock_bh/irq، لازم الـ xarray يتعمل init
   بنفس الـ flags (XA_FLAGS_LOCK_BH أو XA_FLAGS_LOCK_IRQ)

3. ما تستخدمش __xa_erase() و أمثاله بدون xa_lock
   حتى لو عندك mutex أعلى — الـ xa_lock لازم لـ lockdep

4. RCU read lock يُؤخذ تلقائياً في الـ normal read API
   → لا تاخده يدوياً مع الـ normal API

[مثال عملي — process context يعدل، softirq يحذف:]

void foo_init(struct foo *foo) {
    xa_init_flags(&foo->array, XA_FLAGS_LOCK_BH); /* ضروري */
}

int foo_store(struct foo *foo, unsigned long index, void *entry) {
    int err;
    xa_lock_bh(&foo->array);      /* يغلق softirq + spinlock */
    err = xa_err(__xa_store(&foo->array, index, entry, GFP_KERNEL));
    if (!err)
        foo->count++;             /* محمي بنفس الـ lock */
    xa_unlock_bh(&foo->array);
    return err;
}

/* بُستدعى من softirq context فقط */
void foo_erase(struct foo *foo, unsigned long index) {
    xa_lock(&foo->array);         /* softirq مغلقة بالفعل */
    __xa_erase(&foo->array, index);
    foo->count--;
    xa_unlock(&foo->array);
}
```

#### 6.4 ماذا يحمي الـ xa_lock تحديداً؟

```
xa_lock يحمي:
  ├── xa->xa_head (التغييرات عليه)
  ├── كل xa_node->slots[] (الكتابة عليها)
  ├── كل xa_node->marks[][] (التعديل عليها)
  ├── xa_node->count و nr_values
  ├── xa->xa_flags (الـ mark bits اللي في الـ flags)
  └── أي بيانات خارجية بيختار المستخدم يحميها (زي foo->count في المثال)

RCU يحمي (للقراءة فقط):
  ├── الـ dereference من xa->xa_head
  ├── الـ dereference من xa_node->slots[]
  └── الـ dereference من xa_node->parent

لا يحتاج lock:
  ├── xa->xa_head == NULL (xa_empty) — atomic read
  └── xa->xa_flags & XA_FLAGS_MARK(mark) (xa_marked) — atomic read
```

#### 6.5 الـ RCU Mechanics في الكتابة

```
Writer (holds xa_lock):                    Reader (RCU read lock):

  old_node = slots[i]                        entry = rcu_dereference(slots[i])
  new_node = kmalloc(...)                    [يستخدم old value — OK]
  setup new_node
  rcu_assign_pointer(slots[i], new_node)  ← memory barrier هنا
  xa_unlock(xa)
                                             [قد يشوف old أو new — كلاهم valid]
  synchronize_rcu()  (عند free)
  kfree(old_node)    [بعد ما كل الـ readers خلصوا]
```

الـ Retry entry (`XA_RETRY_ENTRY`) هو الحل لما يشوف الـ reader node بتتبنى تحت إيده — يعمل restart من الـ head.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Normal API

| Function | Signature (مختصرة) | Lock | الوظيفة |
|---|---|---|---|
| `xa_init` | `void xa_init(xa*)` | none | تهيئة XArray فارغ |
| `xa_init_flags` | `void xa_init_flags(xa*, gfp_t)` | none | تهيئة مع flags |
| `xa_load` | `void *xa_load(xa*, ulong idx)` | RCU read | قراءة entry |
| `xa_store` | `void *xa_store(xa*, ulong idx, void*, gfp_t)` | xa_lock | كتابة entry |
| `xa_erase` | `void *xa_erase(xa*, ulong idx)` | xa_lock | حذف entry |
| `xa_insert` | `int xa_insert(xa*, ulong idx, void*, gfp_t)` | xa_lock | كتابة إن كان فارغاً |
| `xa_cmpxchg` | `void *xa_cmpxchg(xa*, ulong, old, new, gfp_t)` | xa_lock | استبدال مشروط |
| `xa_store_range` | `void *xa_store_range(xa*, ulong first, ulong last, void*, gfp_t)` | xa_lock | كتابة على نطاق |
| `xa_alloc` | `int xa_alloc(xa*, u32 *id, void*, limit, gfp_t)` | xa_lock | تخصيص ID تلقائي |
| `xa_alloc_cyclic` | `int xa_alloc_cyclic(xa*, u32 *id, void*, limit, u32 *next, gfp_t)` | xa_lock | تخصيص ID دوري |
| `xa_reserve` | `int xa_reserve(xa*, ulong idx, gfp_t)` | xa_lock | حجز slot |
| `xa_release` | `void xa_release(xa*, ulong idx)` | xa_lock | تحرير حجز |
| `xa_find` | `void *xa_find(xa*, ulong *idx, ulong max, xa_mark_t)` | RCU read | إيجاد أول entry |
| `xa_find_after` | `void *xa_find_after(xa*, ulong *idx, ulong max, xa_mark_t)` | RCU read | إيجاد entry تالي |
| `xa_extract` | `uint xa_extract(xa*, void **dst, ulong start, ulong max, uint n, xa_mark_t)` | RCU read | نسخ entries لـ array |
| `xa_destroy` | `void xa_destroy(xa*)` | xa_lock | تدمير كل الـ entries |
| `xa_empty` | `bool xa_empty(xa*)` | none | فحص إن كان فارغاً |
| `xa_marked` | `bool xa_marked(xa*, xa_mark_t)` | none | فحص وجود mark |
| `xa_get_mark` | `bool xa_get_mark(xa*, ulong idx, xa_mark_t)` | RCU read | قراءة mark على entry |
| `xa_set_mark` | `void xa_set_mark(xa*, ulong idx, xa_mark_t)` | xa_lock | ضبط mark |
| `xa_clear_mark` | `void xa_clear_mark(xa*, ulong idx, xa_mark_t)` | xa_lock | مسح mark |

#### Locked Variants (BH / IRQ)

| Pattern | BH version | IRQ version |
|---|---|---|
| `xa_store` | `xa_store_bh` | `xa_store_irq` |
| `xa_erase` | `xa_erase_bh` | `xa_erase_irq` |
| `xa_cmpxchg` | `xa_cmpxchg_bh` | `xa_cmpxchg_irq` |
| `xa_insert` | `xa_insert_bh` | `xa_insert_irq` |
| `xa_alloc` | `xa_alloc_bh` | `xa_alloc_irq` |
| `xa_reserve` | `xa_reserve_bh` | `xa_reserve_irq` |

#### Locked Internals (الـ `__xa_*` — caller holds xa_lock)

| Function | الوظيفة |
|---|---|
| `__xa_store` | store بدون lock داخلي |
| `__xa_erase` | erase بدون lock داخلي |
| `__xa_insert` | insert بدون lock داخلي |
| `__xa_cmpxchg` | cmpxchg بدون lock داخلي |
| `__xa_alloc` | alloc بدون lock داخلي |
| `__xa_alloc_cyclic` | alloc cyclic بدون lock داخلي |
| `__xa_set_mark` | set mark بدون lock داخلي |
| `__xa_clear_mark` | clear mark بدون lock داخلي |

#### Advanced API (xas_*)

| Function | الوظيفة |
|---|---|
| `XA_STATE()` | تعريف xas على الـ stack |
| `XA_STATE_ORDER()` | تعريف xas مع order لـ multi-index |
| `xas_load` | قراءة entry عبر cursor |
| `xas_store` | كتابة entry عبر cursor |
| `xas_find` | إيجاد أول entry موجود |
| `xas_find_marked` | إيجاد أول entry مُعلَّم |
| `xas_find_conflict` | إيجاد أول entry يتعارض مع range |
| `xas_next` | تحريك cursor للأمام |
| `xas_prev` | تحريك cursor للخلف |
| `xas_next_entry` | inline optimized للـ iteration |
| `xas_next_marked` | inline optimized للـ marked iteration |
| `xas_get_mark` | قراءة mark عبر cursor |
| `xas_set_mark` | ضبط mark عبر cursor |
| `xas_clear_mark` | مسح mark عبر cursor |
| `xas_init_marks` | إعادة تهيئة marks |
| `xas_set` | تغيير index في cursor |
| `xas_set_order` | تحضير cursor لـ multi-index entry |
| `xas_advance` | تخطي sibling entries |
| `xas_reload` | إعادة قراءة entry بدون walk |
| `xas_pause` | إيقاف iteration مؤقتاً |
| `xas_reset` | إعادة تعيين cursor للـ root |
| `xas_retry` | التحقق والـ retry للـ internal entries |
| `xas_error` | استخراج errno من cursor |
| `xas_set_err` | حفظ errno في cursor |
| `xas_nomem` | محاولة تخصيص memory بعد ENOMEM |
| `xas_destroy` | تحرير memory محجوزة في xas |
| `xas_create_range` | تخصيص nodes لنطاق كامل |
| `xas_split_alloc` | تخصيص nodes للـ split (بدون lock) |
| `xas_split` | تنفيذ split (مع lock) |
| `xas_try_split` | split تدريجي بـ xa_lock |
| `xas_set_update` | تسجيل callback لتحديثات nodes |

#### Entry Type Helpers

| Function | الوظيفة |
|---|---|
| `xa_mk_value(v)` | تحويل integer لـ XArray entry |
| `xa_to_value(entry)` | استخراج integer من value entry |
| `xa_is_value(entry)` | فحص إن كان value entry |
| `xa_tag_pointer(p, tag)` | إنشاء tagged pointer entry |
| `xa_untag_pointer(entry)` | استخراج pointer من tagged entry |
| `xa_pointer_tag(entry)` | استخراج الـ tag |
| `xa_is_err(entry)` | فحص إن كان entry يحمل error |
| `xa_err(entry)` | استخراج errno من entry |
| `xa_is_zero(entry)` | فحص zero entry |
| `xa_is_retry(entry)` | فحص retry entry |
| `xa_is_sibling(entry)` | فحص sibling entry |
| `xa_is_node(entry)` | فحص node entry |

---

### Group 1: Initialization & Destruction

الـ group ده مسؤول عن إنشاء الـ XArray وتجهيزه للاستخدام أو تدميره بالكامل عند الانتهاء.

---

#### `xa_init_flags`

```c
static inline void xa_init_flags(struct xarray *xa, gfp_t flags)
{
    spin_lock_init(&xa->xa_lock);
    xa->xa_flags = flags;
    xa->xa_head = NULL;
}
```

بتهيئ الـ `xarray` struct بـ flags مخصصة. بتعمل `spin_lock_init` على الـ `xa_lock`، وبتخزن الـ `flags` في `xa_flags`، وبتضبط الـ `xa_head` على `NULL` يعني array فارغة. اللي بيستخدم `XA_FLAGS_LOCK_IRQ` أو `XA_FLAGS_LOCK_BH` لازم يستخدم هذه الدالة عشان النظام يعرف إنه لازم يـ disable interrupts أو softirqs عند أخذ الـ lock.

- **`xa`**: pointer للـ `struct xarray` المراد تهيئتها
- **`flags`**: مجموعة من الـ flags زي `XA_FLAGS_ALLOC`, `XA_FLAGS_LOCK_IRQ`, `XA_FLAGS_ALLOC1`
- **Return**: void
- **Context**: Any context — safe to call from anywhere
- **Side effects**: يهيئ الـ spinlock؛ لو استخدمت `XA_FLAGS_ALLOC` يُفعَّل tracking لـ free entries عبر `XA_MARK_0`

---

#### `xa_init`

```c
static inline void xa_init(struct xarray *xa)
{
    xa_init_flags(xa, 0);
}
```

**الـ** wrapper البسيط لـ `xa_init_flags` بـ flags = 0. للـ arrays العادية اللي مش محتاجة allocation tracking أو interrupt-safe locking.

- **`xa`**: pointer للـ XArray
- **Return**: void
- **Context**: Any context

---

#### `DEFINE_XARRAY` / `DEFINE_XARRAY_ALLOC` / `DEFINE_XARRAY_ALLOC1` / `DEFINE_XARRAY_FLAGS`

```c
#define DEFINE_XARRAY(name) DEFINE_XARRAY_FLAGS(name, 0)
#define DEFINE_XARRAY_ALLOC(name) DEFINE_XARRAY_FLAGS(name, XA_FLAGS_ALLOC)
#define DEFINE_XARRAY_ALLOC1(name) DEFINE_XARRAY_FLAGS(name, XA_FLAGS_ALLOC1)
#define DEFINE_XARRAY_FLAGS(name, flags) \
    struct xarray name = XARRAY_INIT(name, flags)
```

ماكرو للتعريف الـ static وقت الـ compile. **الـ** `XARRAY_INIT` بيعمل static initializer للـ spinlock بدلاً من `spin_lock_init`. مناسب لـ global arrays أو arrays على مستوى الـ module.

| Macro | متى تستخدمه |
|---|---|
| `DEFINE_XARRAY` | XArray عادي |
| `DEFINE_XARRAY_ALLOC` | تخصيص IDs من 0 |
| `DEFINE_XARRAY_ALLOC1` | تخصيص IDs من 1 (عشان 0 reserved) |
| `DEFINE_XARRAY_FLAGS` | تحكم كامل في الـ flags |

---

#### `xa_destroy`

```c
void xa_destroy(struct xarray *);
```

بتحذف كل الـ nodes الداخلية للـ XArray وبترجعه لحالته الأولية (فارغ). **لا** بتحرر الـ pointers المخزنة في الـ slots — ده مسؤولية الـ caller. غالباً بتستخدم قبلها `xa_for_each` لتحرير الـ entries.

- **`xa`**: pointer للـ XArray المراد تدميره
- **Return**: void
- **Lock**: بتأخذ `xa_lock` داخلياً
- **Caller**: يجب إنهاء كل الـ iterations قبل استدعائها

**Pattern شائع:**
```c
void *entry;
unsigned long idx;

/* أولاً حرر الـ entries */
xa_for_each(&my_array, idx, entry)
    kfree(entry);

/* ثم دمر الـ tree */
xa_destroy(&my_array);
```

---

### Group 2: Load & Lookup

---

#### `xa_load`

```c
void *xa_load(struct xarray *, unsigned long index);
```

الدالة الأساسية للقراءة. بتقفل الـ RCU read lock، بتمشي الـ radix tree من الـ root حتى تلاقي الـ slot المطلوبة، وبترجع المحتوى. لو الـ entry كانت zero entry (reserved) بترجع `NULL`. لو ما لقتش شيء بترجع `NULL`.

- **`xa`**: XArray المراد القراءة منه
- **`index`**: الـ index (حتى `ULONG_MAX`)
- **Return**: الـ entry المخزن، أو `NULL` لو فارغ أو reserved
- **Lock**: بتأخذ RCU read lock داخلياً وبتحرره عند الإرجاع
- **Context**: Any context including interrupt
- **Key detail**: آمنة تحت RCU فقط — لو محتاج تأخذ reference على الـ object لازم تأخذ `xa_lock` أو تزود refcount تحت RCU

---

#### `xa_find`

```c
void *xa_find(struct xarray *xa, unsigned long *index,
              unsigned long max, xa_mark_t filter) __attribute__((nonnull(2)));
```

بتبحث عن أول entry موجود ابتداءً من `*index` وحتى `max`. لو الـ filter كانت `XA_PRESENT` بتبحث عن أي entry موجود. بتعدّل `*index` لقيمة الـ index اللي لاقت فيه الـ entry.

- **`xa`**: XArray
- **`*index`**: [in/out] بداية البحث، بيتحدث لـ index الـ entry المكتشفة
- **`max`**: حد أقصى للبحث (inclusive)
- **`filter`**: `XA_PRESENT` لأي entry، أو `XA_MARK_0/1/2` للمُعلَّمة
- **Return**: الـ entry الموجود أو `NULL`
- **Lock**: RCU read lock
- **Use case**: بداية الـ iteration مع `xa_for_each_range`

---

#### `xa_find_after`

```c
void *xa_find_after(struct xarray *xa, unsigned long *index,
                    unsigned long max, xa_mark_t filter) __attribute__((nonnull(2)));
```

مثل `xa_find` لكن بتبدأ البحث من `*index + 1` — يعني تتخطى الـ entry الحالية. ده هو الـ "step" في الـ iteration loops.

- **Return**: الـ entry التالي الموجود أو `NULL`
- **Note**: الفرق الوحيد عن `xa_find` إنها تتخطى الـ index الحالي دائماً

---

#### `xa_extract`

```c
unsigned int xa_extract(struct xarray *, void **dst, unsigned long start,
                        unsigned long max, unsigned int n, xa_mark_t);
```

بتنسخ entries من الـ XArray لـ plain C array (batch read). مفيدة لما تحتاج تعمل bulk processing بدون ما تبقى تمسك الـ RCU lock لفترة طويلة.

- **`dst`**: الـ array الهدف
- **`start`**: أول index
- **`max`**: آخر index (inclusive)
- **`n`**: أقصى عدد entries تنسخها
- **`filter`**: `XA_PRESENT` أو mark
- **Return**: عدد الـ entries المنسوخة فعلاً
- **Lock**: RCU read lock
- **Context**: Any context

---

### Group 3: Store, Insert, Erase

---

#### `xa_store`

```c
void *xa_store(struct xarray *, unsigned long index, void *entry, gfp_t);
```

الدالة الأساسية للكتابة. بتحل محل أي entry موجود في الـ `index` المحدد وبترجع القيمة القديمة. لو `entry == NULL` يكافئ `xa_erase`. لو الـ XArray استخدم `XA_FLAGS_ALLOC`، كتابة `NULL` بتعمل "zero entry" (reserved) مش حذف حقيقي.

**Pseudocode flow:**
```
xa_store(xa, index, entry, gfp):
    xa_lock(xa)
    old = __xa_store(xa, index, entry, gfp)
        → xas_store(xas, entry)
            → walk tree to index
            → allocate nodes if needed (may drop lock temporarily)
            → write entry to slot
            → update node counts
    xa_unlock(xa)
    return old entry (or error encoded as pointer)
```

- **`xa`**: XArray
- **`index`**: الـ index المراد الكتابة فيه
- **`entry`**: القيمة الجديدة (أو `NULL` للحذف)
- **`gfp`**: GFP flags لـ memory allocation للـ tree nodes
- **Return**: الـ entry القديم، أو `xa_err(-ENOMEM)` لو فشل التخصيص
- **Lock**: يأخذ ويحرر `xa_lock` داخلياً
- **Error check**: استخدم `xa_is_err()` أو `xa_err()` على الـ return value

---

#### `xa_store_bh` / `xa_store_irq`

```c
static inline void *xa_store_bh(struct xarray *xa, unsigned long index,
                                 void *entry, gfp_t gfp);
static inline void *xa_store_irq(struct xarray *xa, unsigned long index,
                                  void *entry, gfp_t gfp);
```

نفس `xa_store` لكن بيضيف `spin_lock_bh` أو `spin_lock_irq` قبل استدعاء `__xa_store`. يُستخدم لما بتكتب من process context وعندك code في softirq/interrupt بيقرأ أو بيعدل نفس الـ array.

- **`xa_store_bh`**: Context: Any context — disables softirqs أثناء الـ lock
- **`xa_store_irq`**: Context: Process context فقط — disables interrupts

---

#### `xa_erase`

```c
void *xa_erase(struct xarray *, unsigned long index);
```

بتحذف الـ entry الموجود في `index`، وبتمسح كل الـ marks المرتبطة به، وبترجع القيمة القديمة. لو كان الـ index جزء من multi-index entry، كل الـ indices المرتبطة بتتحذف وتنفصل.

- **Return**: الـ entry القديم أو `NULL` لو كان الـ slot فارغ
- **Lock**: `xa_lock` داخلياً
- **Marks**: كل الـ marks بتتمسح تلقائياً

---

#### `xa_insert`

```c
static inline int __must_check xa_insert(struct xarray *xa,
        unsigned long index, void *entry, gfp_t gfp);
```

بتكتب `entry` في `index` لكن فقط لو الـ slot كان فارغاً (ماعنهوش حتى reserved entry). لو فيه أي entry موجود (حتى zero entry) بترجع `-EBUSY`.

- **Return**: `0` نجاح، `-EBUSY` الـ slot مشغول، `-ENOMEM` فشل التخصيص
- **`__must_check`**: لازم تتحقق من الـ return value
- **Lock**: `xa_lock` داخلياً
- **Atomicity**: guaranteed — لو فيه race مع store آخر، المتأخر هيفشل بـ EBUSY

---

#### `xa_cmpxchg`

```c
static inline void *xa_cmpxchg(struct xarray *xa, unsigned long index,
                                void *old, void *entry, gfp_t gfp);
```

Compare-and-swap atomic على مستوى الـ XArray. لو الـ entry الحالي في `index` يساوي `old`، بيحل محله بـ `entry`. لو الـ return value يساوي `old`، الـ exchange نجح.

- **`old`**: القيمة المتوقعة الحالية
- **`entry`**: القيمة الجديدة
- **Return**: القيمة الحالية وقت تنفيذ العملية (قبل التغيير أو بعد فشله)
- **Success check**: `if (xa_cmpxchg(xa, idx, old, new, gfp) == old) /* success */`
- **Lock**: `xa_lock` داخلياً

---

#### `xa_store_range`

```c
void *xa_store_range(struct xarray *, unsigned long first, unsigned long last,
                     void *entry, gfp_t);
```

بتخزن نفس الـ entry على نطاق من indices. الـ range لازم تكون aligned power-of-two (e.g., 64-127 مسموح، 2-6 مش مسموح). بعد الكتابة، أي lookup لأي index في الـ range بيرجع نفس الـ entry.

- **`first`**: بداية الـ range
- **`last`**: نهاية الـ range (inclusive)
- **Lock**: `xa_lock` داخلياً
- **Constraint**: الـ range لازم تكون aligned power-of-two
- **Marks**: ضبط mark على أي index في الـ range قد لا يؤثر على باقي الـ indices (behavior محدود)

---

### Group 4: Reservation & Memory Safety

---

#### `xa_reserve`

```c
static inline int __must_check
int xa_reserve(struct xarray *xa, unsigned long index, gfp_t gfp)
{
    return xa_err(xa_cmpxchg(xa, index, NULL, XA_ZERO_ENTRY, gfp));
}
```

بتحجز الـ slot بدون ما تكتب entry حقيقي. بتضمن إن استدعاء `xa_store` لاحق على نفس الـ index مش هيحتاج يخصص memory. الـ Normal API بتشوف الـ slot زي `NULL` عادي.

- **Return**: `0` نجاح، `-ENOMEM` لو فشل تخصيص الـ tree nodes
- **Implementation**: بتعمل `cmpxchg` من `NULL` إلى `XA_ZERO_ENTRY` (internal entry رقم 257)
- **Use case**: لما تريد تضمن إن الـ store التالي ما يفشلش بـ ENOMEM في context خطر (atomic/irq)

---

#### `xa_release`

```c
static inline void xa_release(struct xarray *xa, unsigned long index)
{
    xa_cmpxchg(xa, index, XA_ZERO_ENTRY, NULL, 0);
}
```

بتحرر حجز عمله `xa_reserve` لو لسه ما اتكتبش فيه entry حقيقي. لو اتكتب فيه entry بالفعل بتتجاهله (لأن الـ cmpxchg هيفشل لأن القيمة مش `XA_ZERO_ENTRY` دلوقتي).

- **No-op إذا**: الـ entry اتغير بعد الـ reserve
- **الفرق عن `xa_erase`**: `xa_erase` بتحذف أي entry، `xa_release` بتحذف فقط لو لسه reserved

---

### Group 5: Allocating IDs

---

#### `xa_alloc`

```c
static inline int __must_check xa_alloc(struct xarray *xa, u32 *id,
        void *entry, struct xa_limit limit, gfp_t gfp);
```

بتلاقي أول slot فارغة في الـ XArray بين `limit.min` و `limit.max`، وبتخزن `entry` فيه، وبتكتب الـ index في `*id`. تشترط إن الـ XArray اتهيئ بـ `XA_FLAGS_ALLOC`.

**الـ mechanism:** الـ `XA_MARK_0` (الـ `XA_FREE_MARK`) بيتستخدم داخلياً لتتبع الـ free slots — الـ marks bitset في كل node بتخلي البحث عن أول free slot O(log n).

- **`id`**: [out] الـ index المخصص
- **`limit`**: `struct xa_limit { u32 min; u32 max; }`
- **Return**: `0` نجاح، `-ENOMEM`، أو `-EBUSY` لو كل الـ IDs في النطاق مستخدمة
- **Lock**: `xa_lock` داخلياً
- **Constraint**: الـ XArray لازم يتهيئ بـ `XA_FLAGS_ALLOC`، ولا تستخدم `XA_MARK_0` يدوياً

**Predefined limits:**
```c
xa_limit_32b  // [0, UINT_MAX]
xa_limit_31b  // [0, INT_MAX]
xa_limit_16b  // [0, USHRT_MAX]
```

---

#### `xa_alloc_cyclic`

```c
static inline int xa_alloc_cyclic(struct xarray *xa, u32 *id, void *entry,
        struct xa_limit limit, u32 *next, gfp_t gfp);
```

زي `xa_alloc` لكن بتبدأ البحث من `*next` بدل من `limit.min`. لو وصلت لـ `limit.max` بتـ wrap حول وترجع للـ `limit.min`. بتعدّل `*next` للـ ID التالي بعد المخصص. مثالية لـ network connection IDs أو file descriptors.

- **`next`**: [in/out] pointer لـ state variable تحافظ على position البحث بين الاستدعاءات
- **Return**: `0` نجاح، `-ENOMEM`، `-EBUSY`
- **Note**: الـ `__xa_alloc_cyclic` بيرجع `1` لو حصل wrap، `0` عادي — الـ `xa_alloc_cyclic` بتخفي ده

---

### Group 6: Mark Operations

كل entry في الـ XArray عنده 3 bits مستقلة للـ marks: `XA_MARK_0`, `XA_MARK_1`, `XA_MARK_2`. الـ marks بتتـ replicate لأعلى في الـ tree (في كل node) عشان البحث يبقى O(log n) مش O(n).

---

#### `xa_get_mark`

```c
bool xa_get_mark(struct xarray *, unsigned long index, xa_mark_t);
```

بترجع `true` لو الـ mark المحدد موجود على الـ entry في `index`. لو الـ entry `NULL` بترجع `false`.

- **Lock**: RCU read lock
- **Context**: Any context

---

#### `xa_set_mark`

```c
void xa_set_mark(struct xarray *, unsigned long index, xa_mark_t);
```

بتضبط الـ mark على الـ entry في `index` وبتـ propagate لأعلى في كل الـ ancestor nodes. لو الـ entry `NULL` ما بيحصل حاجة. لو كان multi-index entry كل الـ indices بتتعلّم.

- **Lock**: `xa_lock` داخلياً
- **Side effect**: بتعدّل الـ marks bitset في كل الـ nodes من الـ leaf للـ root

---

#### `xa_clear_mark`

```c
void xa_clear_mark(struct xarray *, unsigned long index, xa_mark_t);
```

عكس `xa_set_mark`. بتمسح الـ mark وبتعدّل الـ ancestor nodes لو مفيش entries تانية في نفس الـ node عندها نفس الـ mark.

---

#### `xa_marked`

```c
static inline bool xa_marked(const struct xarray *xa, xa_mark_t mark)
{
    return xa->xa_flags & XA_FLAGS_MARK(mark);
}
```

بتفحص لو في أي entry على الإطلاق في الـ XArray عنده هذا الـ mark. ده O(1) لأن المعلومة موجودة في `xa_flags` (root-level mark aggregate).

- **Lock**: بدون lock — atomic read للـ flags
- **Context**: Any context

---

### Group 7: Iteration Macros (Normal API)

---

#### `xa_for_each`

```c
#define xa_for_each(xa, index, entry) \
    xa_for_each_start(xa, index, entry, 0)
```

يمشي على كل الـ entries الموجودة (غير NULL) في الـ XArray من الـ index 0. آمن لتعديل الـ array أثناء الـ iteration. الـ complexity هي O(n·log n) لأن كل خطوة بتعمل `xa_find_after` من السطح.

```c
/* مثال: طباعة كل الـ entries */
unsigned long idx;
void *entry;
xa_for_each(&my_xa, idx, entry) {
    pr_info("index=%lu val=%p\n", idx, entry);
}
```

---

#### `xa_for_each_range`

```c
#define xa_for_each_range(xa, index, entry, start, last)  \
    for (index = start,                                    \
         entry = xa_find(xa, &index, last, XA_PRESENT);   \
         entry;                                            \
         entry = xa_find_after(xa, &index, last, XA_PRESENT))
```

iteration على نطاق محدد `[start, last]`. أفضل من manual loop بـ `xa_find`/`xa_find_after` لأنه أوضح syntax.

---

#### `xa_for_each_marked`

```c
#define xa_for_each_marked(xa, index, entry, filter) \
    for (index = 0, entry = xa_find(xa, &index, ULONG_MAX, filter); \
         entry; entry = xa_find_after(xa, &index, ULONG_MAX, filter))
```

يمشي على الـ entries المُعلَّمة بـ mark معين. الـ mark bitmap في الـ tree يجعل التخطي فوق الـ unmarked nodes سريعاً.

---

#### `xa_empty`

```c
static inline bool xa_empty(const struct xarray *xa)
{
    return xa->xa_head == NULL;
}
```

O(1) check — لو `xa_head == NULL` مفيش nodes ولا entries. بدون lock.

---

### Group 8: Lock Helpers

الـ XArray بيكشف الـ `xa_lock` للـ callers اللي محتاجين يعملوا "extended locking" — زي تعديل counter إضافي تحت نفس الـ lock.

```c
/* Lock macros — wrappers على xa->xa_lock spinlock */
xa_lock(xa)            // spin_lock
xa_unlock(xa)          // spin_unlock
xa_lock_bh(xa)         // spin_lock_bh
xa_unlock_bh(xa)       // spin_unlock_bh
xa_lock_irq(xa)        // spin_lock_irq
xa_unlock_irq(xa)      // spin_unlock_irq
xa_lock_irqsave(xa, flags)
xa_unlock_irqrestore(xa, flags)
xa_trylock(xa)         // spin_trylock
```

**مثال extended locking:**
```c
xa_lock_bh(&foo->array);
err = xa_err(__xa_store(&foo->array, index, entry, GFP_KERNEL));
if (!err)
    foo->count++;   /* protected by xa_lock */
xa_unlock_bh(&foo->array);
```

---

### Group 9: Entry Type Helpers

---

#### `xa_mk_value` / `xa_to_value` / `xa_is_value`

```c
static inline void *xa_mk_value(unsigned long v)
{
    WARN_ON((long)v < 0);
    return (void *)((v << 1) | 1);  /* set bit 0 */
}

static inline unsigned long xa_to_value(const void *entry)
{
    return (unsigned long)entry >> 1;
}

static inline bool xa_is_value(const void *entry)
{
    return (unsigned long)entry & 1;  /* bit 0 set = value */
}
```

**الـ XArray** بيستخدم bits 1:0 للتمييز بين أنواع الـ entries:
- `00` = pointer عادي
- `10` = internal entry
- `x1` = value entry أو tagged pointer

`xa_mk_value` بتعمل shift يساراً وتضبط bit 0. تدعم values من 0 لـ `LONG_MAX` فقط.

---

#### `xa_tag_pointer` / `xa_untag_pointer` / `xa_pointer_tag`

```c
static inline void *xa_tag_pointer(void *p, unsigned long tag)
{
    return (void *)((unsigned long)p | tag);  /* tag in bits 1:0 */
}

static inline void *xa_untag_pointer(void *entry)
{
    return (void *)((unsigned long)entry & ~3UL);
}

static inline unsigned int xa_pointer_tag(void *entry)
{
    return (unsigned long)entry & 3UL;
}
```

بتخزن 2-bit tag في الـ low bits لأي pointer مُحاذى على 4 bytes. الـ tags المتاحة: 0، 1، 3 (مش 2 لأنه internal entry marker). لا يجوز الجمع بين value entries وtagged pointers في نفس الـ XArray.

---

#### `xa_is_err` / `xa_err`

```c
static inline bool xa_is_err(const void *entry)
{
    return unlikely(xa_is_internal(entry) &&
                    entry >= xa_mk_internal(-MAX_ERRNO));
}

static inline int xa_err(void *entry)
{
    if (xa_is_err(entry))
        return (long)entry >> 2;  /* sign extension */
    return 0;
}
```

الـ XArray بيـ encode الـ errors كـ internal entries في الـ negative space. `xa_err` بتعمل sign extension صح لاستخراج الـ errno. استخدم `xa_is_err` للفحص السريع لو مش محتاج القيمة الفعلية.

```c
void *old = xa_store(&xa, idx, entry, GFP_KERNEL);
if (xa_is_err(old)) {
    pr_err("store failed: %d\n", xa_err(old));
}
```

---

### Group 10: Advanced API — xa_state

---

#### `XA_STATE` / `XA_STATE_ORDER`

```c
#define XA_STATE(name, array, index)    \
    struct xa_state name = __XA_STATE(array, index, 0, 0)

#define XA_STATE_ORDER(name, array, index, order)   \
    struct xa_state name = __XA_STATE(array,        \
            (index >> order) << order,              \
            order - (order % XA_CHUNK_SHIFT),       \
            (1U << (order % XA_CHUNK_SHIFT)) - 1)
```

**الـ `xa_state`** هو الـ cursor الأساسي للـ Advanced API. بيتعلن على الـ stack وبيحتفظ بـ:
- `xa`: الـ array
- `xa_index`: الـ index الحالي
- `xa_shift`: الـ shift للـ multi-index
- `xa_sibs`: عدد الـ siblings للـ multi-index
- `xa_node`: إما pointer للـ node الحالي، أو `XAS_RESTART`، أو error
- `xa_alloc`: pre-allocated node للاستخدام اللاحق
- `xa_update`: callback للـ node updates

`XA_STATE_ORDER` بتجهز الـ xas لعمليات multi-index.

---

#### `xas_load`

```c
void *xas_load(struct xa_state *);
```

بتمشي الـ `xa_state` للـ position الصح في الـ tree وبترجع الـ entry. لو الـ xas في state `XAS_RESTART` بتبدأ من الـ root. لو الـ xas في error state بترجع `NULL`. قد ترجع internal entries (زي retry entries) — استخدم `xas_retry()` بعدها.

- **Lock**: يجب الاحتفاظ بـ RCU read lock أو `xa_lock`
- **Context**: Advanced API — caller مسؤول عن الـ locking

**Pseudocode:**
```
xas_load(xas):
    if xas->xa_node == XAS_RESTART:
        walk from root to xa_index
    if xa_node == XAS_BOUNDS:
        return NULL  /* beyond allocated tree */
    return slot[xa_offset]
```

---

#### `xas_store`

```c
void *xas_store(struct xa_state *, void *entry);
```

بتكتب `entry` في الـ position الحالية للـ xas. لو الـ xas في error state بترجع `NULL`. لو محتاجة تخصص nodes وما قدرتش، بتضبط `ENOMEM` في الـ xas. بترجع الـ entry القديم.

- **Lock**: يجب الاحتفاظ بـ `xa_lock`
- **Memory**: قد تحتاج تعمل retry loop مع `xas_nomem`

**Pattern مع retry:**
```c
XA_STATE(xas, &xa, index);
void *old;

do {
    xa_lock(&xa);
    old = xas_store(&xas, entry);
    xa_unlock(&xa);
} while (xas_nomem(&xas, GFP_KERNEL));

if (xas_error(&xas))
    return xas_error(&xas);
```

---

#### `xas_find`

```c
void *xas_find(struct xa_state *, unsigned long max);
```

بتبحث عن أول entry موجود ابتداءً من الـ position الحالية للـ xas. لو الـ xas ما اتمشيتش (XAS_RESTART) بترجع الـ entry في الـ index الحالي. لو الـ xas مشي بالفعل بترجع الـ entry التالي.

- **Lock**: RCU read lock أو `xa_lock`
- **Returns**: الـ entry، أو `NULL` لو ما فيش entry حتى `max`

---

#### `xas_find_marked`

```c
void *xas_find_marked(struct xa_state *, unsigned long max, xa_mark_t);
```

زي `xas_find` لكن بتبحث عن entries مُعلَّمة بـ mark معين. بتستخدم الـ marks bitset في كل node لتخطي الـ nodes اللي ما عندهاش marks — ده يخلي البحث O(log n) للتخطي.

---

#### `xas_find_conflict`

```c
void *xas_find_conflict(struct xa_state *);
```

في سياق multi-index، بتلاقي أول entry موجود داخل الـ range المحدد في الـ xas. لو الـ range فارغ بترجع `NULL`. بتستخدمها قبل `xas_store` لضمان عدم وجود تعارض.

---

#### `xas_next` / `xas_prev`

```c
static inline void *xas_next(struct xa_state *xas);
static inline void *xas_prev(struct xa_state *xas);
```

بيحركوا الـ cursor بمقدار ±1 index. الـ fast path — لو في نفس الـ node — هو O(1) increment/decrement للـ offset. اللي بيخرج من الـ node boundaries بيستدعي `__xas_next`/`__xas_prev` (slow path).

- **Lock**: RCU read lock أو `xa_lock`
- **Warning**: لا تستخدمهم مع multi-index xa_state — هيظهروا sibling entries

---

#### `xas_set` / `xas_advance` / `xas_set_order`

```c
static inline void xas_set(struct xa_state *xas, unsigned long index);
static inline void xas_advance(struct xa_state *xas, unsigned long index);
static inline void xas_set_order(struct xa_state *xas, unsigned long index,
                                 unsigned int order);
```

- **`xas_set`**: بيغير الـ index وبيضبط الـ node على `XAS_RESTART` — الـ walk التالية هتبدأ من الـ root. لا تحتاج lock.
- **`xas_advance`**: بيتحرك للـ index المحدد بدون ما يغير الـ xa_node (بيبقى في نفس الـ node). مفيد لتخطي sibling entries داخل نفس الـ node.
- **`xas_set_order`**: بيجهز الـ xas لعملية multi-index بتحديد الـ order (2^order indices).

---

#### `xas_reload`

```c
static inline void *xas_reload(struct xa_state *xas)
```

بتقرأ الـ entry في الـ position الحالية بدون ما تعمل tree walk من البداية. مفيدة للـ lockless page cache lookups: بتمشي الـ tree تحت RCU، تلاقي الـ page، تـ lock الـ page، ثم تتحقق إن الـ page ما اتحركتش.

- **Guarantee**: الـ caller مسؤول إن الـ xas لسه valid (ما انتهتش صلاحيته)
- **لو الـ xas ممكن يكون error/restart**: استخدم `xas_load` بدلاً منها

---

#### `xas_pause`

```c
void xas_pause(struct xa_state *);
```

بتوقف الـ iteration مؤقتاً وبتضبط الـ xas بطريقة تضمن إن الاستئناف هيكمل من بعد الـ entry الأخير المعالج. **لازم** تستدعيها قبل ما تسيب الـ lock أثناء iteration. بعد رفع وأخذ الـ lock تاني، الـ iterator هيكمل صح.

```c
xas_for_each(&xas, entry, ULONG_MAX) {
    if (need_resched()) {
        xas_pause(&xas);    /* mark safe resume point */
        xa_unlock(&xa);
        cond_resched();
        xa_lock(&xa);
    }
    /* process entry */
}
```

---

#### `xas_reset`

```c
static inline void xas_reset(struct xa_state *xas)
{
    xas->xa_node = XAS_RESTART;
}
```

بتصفي الـ error state أو الـ walk state وبترجع الـ cursor للـ root. بتستخدمها بعد ما تـ drop الـ lock وتاخده تاني، أو بعد `xas_set_err`.

---

#### `xas_retry`

```c
static inline bool xas_retry(struct xa_state *xas, const void *entry)
{
    if (xa_is_zero(entry))
        return true;
    if (!xa_is_retry(entry))
        return false;
    xas_reset(xas);
    return true;
}
```

بتتعامل مع الـ internal entries اللي ممكن الـ Advanced API يشوفها. لو الـ entry كان zero entry (reserved) بترجع `true` عشان تتخطاه. لو retry entry بتعمل `xas_reset` وبترجع `true` للـ restart.

**Pattern أساسي:**
```c
entry = xas_load(&xas);
if (xas_retry(&xas, entry))
    goto retry;
```

---

#### `xas_error` / `xas_set_err`

```c
static inline int xas_error(const struct xa_state *xas)
{
    return xa_err(xas->xa_node);
}

static inline void xas_set_err(struct xa_state *xas, long err)
{
    xas->xa_node = XA_ERROR(err);
}
```

الـ `xa_state` بيستخدم `xa_node` field لحفظ الـ errors — بتـ encode الـ errno كـ `XA_ERROR(errno)` وهو internal entry في الـ negative space. كل functions الـ Advanced API بتفحص الـ error state قبل العمل، عشان تقدر تعمل chain من الاستدعاءات وتفحص الـ error مرة واحدة في النهاية.

---

#### `xas_nomem`

```c
bool xas_nomem(struct xa_state *, gfp_t);
```

بترجع `true` لو الـ xas في error state بسبب ENOMEM **و** قدرت تخصص memory جديدة. الهدف إنها تُستخدم كشرط الـ retry loop. لو مفيش error أو الـ error مش ENOMEM، بتحرر أي memory سبق ما خصصتها وبترجع `false`.

**التسلسل المعتمد:**
```
1. xas_lock
2. xas_store → فشل بـ ENOMEM → خزّن في xas->xa_node
3. xas_unlock
4. xas_nomem → يخصص memory (بدون lock) → يرجع true
5. goto step 1
6. xas_nomem → ما فيش error → بيحرر memory مش محتاجة → يرجع false
7. انتهى
```

---

#### `xas_destroy`

```c
void xas_destroy(struct xa_state *);
```

بتحرر أي memory خصصتها الـ xas operations ولم تُستخدم (الـ `xa_alloc` field). لازم تستدعيها بعد انتهاء الاستخدام لو استخدمت الـ retry loop مع `xas_nomem`.

---

#### `xas_init_marks`

```c
void xas_init_marks(const struct xa_state *);
```

بتعيد تهيئة الـ marks على الـ entry الحالي لحالتها الافتراضية. للـ XArrays العادية ده يعني مسح كل الـ marks. للـ XArrays المُعرَّفة بـ `XA_FLAGS_TRACK_FREE`، بتضبط `XA_MARK_0` (free mark) وبتمسح الباقي.

- **Use case**: لما تبدل entry موجود بآخر وتريد reset صريح للـ marks (لأن `xas_store` ما بتعملش reset تلقائي)

---

#### `xas_get_mark` / `xas_set_mark` / `xas_clear_mark`

```c
bool xas_get_mark(const struct xa_state *, xa_mark_t);
void xas_set_mark(const struct xa_state *, xa_mark_t);
void xas_clear_mark(const struct xa_state *, xa_mark_t);
```

Advanced API equivalents للـ `xa_get/set/clear_mark` لكن بيشتغلوا على الـ position الحالية للـ xas. أكفأ من الـ Normal API لأنهم ما بيعيدوش الـ tree walk — الـ xas موقعه معروف بالفعل. **ما بيشتغلوش** لو استدعيت `xas_pause` أو `xas_set` قبلهم مباشرة.

---

#### `xas_create_range`

```c
void xas_create_range(struct xa_state *);
```

بتخصص كل الـ `xa_node` nodes اللازمة لاستيعاب نطاق كامل من entries. لو فشل التخصيص بتضبط `ENOMEM` في الـ xas. **تُستخدم قبل الـ lock** للـ pre-allocation لضمان إن الـ store التالي ما يفشلش.

---

#### `xas_set_update`

```c
static inline void xas_set_update(struct xa_state *xas, xa_update_node_t update)
{
    xas->xa_update = update;
}
```

بتسجل callback بيتنادى في كل مرة الـ XArray يعدّل counts في node. الـ callback بياخد `struct xa_node *` وبيشتغل وهو الـ `xa_lock` محجوز والـ interrupts ممكن تكون disabled — **لا تسيب الـ lock** جوّا الـ callback. بيستخدمه workingset code في الـ page cache.

---

### Group 11: Multi-Index — Split Operations

---

#### `xas_split_alloc`

```c
void xas_split_alloc(struct xa_state *, void *entry, unsigned int order, gfp_t);
```

بتخصص الـ nodes اللازمة لتقسيم multi-index entry من `order` الحالي لـ order أصغر. **تُستدعى بدون `xa_lock`**. لو فشل التخصيص بتضبط ENOMEM في الـ xas.

---

#### `xas_split`

```c
void xas_split(struct xa_state *, void *entry, unsigned int order);
```

بتنفذ الـ split الفعلي بعد `xas_split_alloc`. **تُستدعى مع `xa_lock` محجوز**. بتحول الـ multi-index entry الواحدة لعدة entries أصغر، كل منها بـ order محدد، وكلهم بيحملوا نفس الـ `entry` value.

**الفرق بين الطريقتين:**

| الطريقة | آلية العمل | عدد nodes اللازمة |
|---|---|---|
| `xas_split_alloc` + `xas_split` | split موحد من order كبير لـ order صغير | 2^(old_order - new_order) nodes |
| `xas_try_split` | split تدريجي غير موحد للـ index المحدد | node واحدة على الأكثر |

---

#### `xas_try_split`

```c
void xas_try_split(struct xa_state *xas, void *entry, unsigned int order);
```

بتقسم الـ entry المحتوي على `xas->xa_index` بشكل تدريجي — كل مرة بتقسم نصف واحد فقط. مثال: لتقسيم order-9 إلى order-0 على index معين، بتعمل: order-9 → order-8، ثم order-8 الأيسر → order-7، إلخ. تحتاج 1 node بدل 8. **تُستدعى مع `xa_lock`**.

---

#### `xas_try_split_min_order`

```c
unsigned int xas_try_split_min_order(unsigned int order);
```

بترجع أصغر order ممكن `xas_try_split` تقدر توصله لـ entry بـ order معين. مفيدة لتحديد حدود الـ split.

---

#### `xa_get_order` / `xas_get_order`

```c
int xa_get_order(struct xarray *, unsigned long index);
int xas_get_order(struct xa_state *xas);
```

بترجعوا الـ order (2^order = حجم الـ range) للـ entry في الـ index المحدد. لو الـ entry مش multi-index بيرجع 0. `xas_get_order` أكفأ لأن الـ cursor موضعه معروف.

---

### Group 12: Advanced Iteration Macros

---

#### `xas_for_each`

```c
#define xas_for_each(xas, entry, max) \
    for (entry = xas_find(xas, max); entry; \
         entry = xas_next_entry(xas, max))
```

O(n) iteration على الـ entries الموجودة. أفضل من `xa_for_each` في الـ performance لأن `xas_next_entry` هو inline يتحرك مباشرة في نفس الـ node بدون function call overhead في الـ common case. **لازم** caller يمسك الـ RCU lock أو `xa_lock`.

---

#### `xas_for_each_marked`

```c
#define xas_for_each_marked(xas, entry, max, mark) \
    for (entry = xas_find_marked(xas, max, mark); entry; \
         entry = xas_next_marked(xas, max, mark))
```

O(n) iteration على الـ entries المُعلَّمة. `xas_next_marked` بيستخدم `find_next_bit` على الـ marks array في الـ node — O(1) لو الـ next marked entry في نفس الـ node.

---

#### `xas_for_each_conflict`

```c
#define xas_for_each_conflict(xas, entry) \
    while ((entry = xas_find_conflict(xas)))
```

يمشي على كل الـ entries اللي بتتعارض مع الـ range المحدد في الـ xas. بيستخدمها من يريد ضمان إن range معين فارغ قبل الكتابة فيه.

---

#### `xas_next_entry` / `xas_next_marked`

```c
static inline void *xas_next_entry(struct xa_state *xas, unsigned long max);
static inline void *xas_next_marked(struct xa_state *xas, unsigned long max,
                                     xa_mark_t mark);
```

**الـ** inline fast-path للـ iteration. في الـ common case (نفس الـ node، ما وصلناش للـ boundary):
- `xas_next_entry`: O(1) — increment offset وقراءة الـ slot
- `xas_next_marked`: O(1) — `find_next_bit` على الـ marks array في الـ node

لو وصلنا لـ boundary أو صادفنا internal entry بيستدعيوا `xas_find`/`xas_find_marked` كـ slow path.

---

#### `XA_CHECK_SCHED`

```c
enum { XA_CHECK_SCHED = 4096 };
```

ثابت بيحدد كم iteration قبل ما caller يعمل `xas_pause` + `cond_resched()`. يمنع starvation في الـ loops الطويلة.

```c
unsigned int processed = 0;
xas_for_each(&xas, entry, ULONG_MAX) {
    process(entry);
    if (++processed % XA_CHECK_SCHED == 0) {
        xas_pause(&xas);
        xa_unlock(&xa);
        cond_resched();
        xa_lock(&xa);
    }
}
```

---

### Group 13: Internal Entry Checkers

| Function | التنفيذ | المعنى |
|---|---|---|
| `xa_is_internal(e)` | `(ulong)e & 3) == 2` | الـ bit pattern `10` في bits 1:0 |
| `xa_is_node(e)` | `xa_is_internal(e) && (ulong)e > 4096` | node pointer مُعلَّم كـ internal |
| `xa_is_sibling(e)` | `xa_is_internal(e) && e < xa_mk_sibling(XA_CHUNK_SIZE-1)` | entry بيشير لـ canonical slot |
| `xa_is_retry(e)` | `e == XA_RETRY_ENTRY` | internal(256) — thread يعدّل حالياً |
| `xa_is_zero(e)` | `e == XA_ZERO_ENTRY` | internal(257) — reserved entry |
| `xa_is_advanced(e)` | `xa_is_internal(e) && e <= XA_RETRY_ENTRY` | فقط Advanced API يشوفه |

---

### Group 14: Debug Helpers

```c
void xa_dump(const struct xarray *);
void xa_dump_node(const struct xa_node *);
```

بيطبعوا state الـ XArray والـ nodes على الـ kernel log. بيُفعَّلوا فقط مع `XA_DEBUG` أو `XA_BUG_ON` macros. مفيد أثناء debugging الـ corruption.

```c
#define XA_BUG_ON(xa, x) do {      \
    if (x) { xa_dump(xa); BUG(); } \
} while (0)
```

---

### ملاحظات ختامية على الـ Locking Model

```
                    ┌─────────────────────────────────┐
                    │         XArray Locking           │
                    ├──────────┬───────────────────────┤
                    │ No lock  │ xa_empty, xa_marked   │
                    ├──────────┼───────────────────────┤
                    │ RCU read │ xa_load, xa_find,     │
                    │ (auto)   │ xa_extract, xa_get_mark│
                    │          │ xa_for_each*           │
                    ├──────────┼───────────────────────┤
                    │ xa_lock  │ xa_store, xa_erase,   │
                    │ (auto)   │ xa_insert, xa_cmpxchg │
                    │          │ xa_alloc, xa_reserve  │
                    │          │ xa_set/clear_mark     │
                    │          │ xa_destroy             │
                    ├──────────┼───────────────────────┤
                    │ xa_lock  │ __xa_store, __xa_erase│
                    │ (caller) │ __xa_insert, __xa_alloc│
                    │          │ __xa_set/clear_mark   │
                    ├──────────┼───────────────────────┤
                    │ RCU/lock │ xas_load, xas_find    │
                    │ (caller) │ xas_for_each, xas_next │
                    │          │ xas_store (needs lock) │
                    └──────────┴───────────────────────┘
```

**القاعدة:** الـ Normal API آمن للاستخدام المباشر. الـ Advanced API (xas_*) أسرع لكن الـ caller مسؤول عن الـ locking الصح.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات المفيدة

الـ XArray نفسه ما عندوش entries مخصصة في debugfs، لكن المستخدمين الرئيسيين زي الـ page cache ليهم entries مفيدة:

```bash
# page cache stats (XArray هو backbone بتاعها)
cat /sys/kernel/debug/mm/oom_kill

# radix tree / XArray node allocator stats
cat /proc/slabinfo | grep -E "radix_tree|xarray|xa_node"
# Output example:
# xa_node           1024   1024    552   29    4 : tunables    0    0    0 ...
```

الـ **xa_node** هو الـ internal node بتاع الـ XArray في الـ slab allocator — لو شفت عدده كبير جداً يبقى في leak.

#### 2. sysfs — المسارات المفيدة

```bash
# memory cgroup: XArray بيُستخدم في memcg charge tracking
cat /sys/fs/cgroup/memory.stat | grep -i anon

# slub stats على xa_node
cat /sys/kernel/slab/xa_node/alloc_calls
cat /sys/kernel/slab/xa_node/free_calls
cat /sys/kernel/slab/xa_node/objects
cat /sys/kernel/slab/xa_node/objects_partial
```

**التفسير**: لو `objects` بيكبر ومش بينزل — في XArray مش بيتعمله `xa_destroy()` أو entries مش بتتحذف صح.

#### 3. ftrace — الـ Tracepoints والـ Events

الـ XArray ما عندوش tracepoints مخصصة، لكن نقدر نتتبع functions بتاعته مباشرة:

```bash
# تفعيل function tracer على XArray functions
cd /sys/kernel/debug/tracing

echo function > current_tracer

# تحديد functions معينة بس
echo 'xa_store xa_load xa_erase xa_alloc xas_store xas_load' > set_ftrace_filter

echo 1 > tracing_on
# ... reproduce the bug ...
echo 0 > tracing_on
cat trace
```

```bash
# تتبع xa_store مع stack trace عشان تعرف مين بيستدعيها
echo 'xa_store' > set_ftrace_filter
echo func_stack_trace > trace_options
cat trace | head -60
```

```bash
# استخدام function_graph لرؤية الـ call tree كامل
echo function_graph > current_tracer
echo 'xa_store' > set_graph_function
echo 1 > tracing_on
```

مثال على الـ output:

```
 1)               |  xa_store() {
 1)               |    xas_store() {
 1)               |      xas_create() {
 1)   0.842 us    |        xa_node_alloc();
 1)   1.205 us    |      }
 1)   2.100 us    |    }
 1)   2.987 us    |  }
```

#### 4. printk / dynamic debug

```bash
# تفعيل dynamic debug لملف xarray.c
echo 'file lib/xarray.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ debug messages في xarray
echo 'module xarray +pflmt' > /sys/kernel/debug/dynamic_debug/control

# التحقق من التفعيل
cat /sys/kernel/debug/dynamic_debug/control | grep xarray
```

لو محتاج تضيف printk مؤقت في الكود، استخدم `pr_debug()` بدل `printk()` عشان يتحكم فيها الـ dynamic debug:

```c
/* في lib/xarray.c أو الكود بتاعك */
pr_debug("xa_store: xa=%p index=%lu entry=%p\n", xa, index, entry);
```

```bash
# مشاهدة الـ kernel log
dmesg -w
# أو
journalctl -k -f
```

#### 5. Kernel Config Options للـ Debugging

| Option | الوظيفة |
|---|---|
| `CONFIG_XARRAY_MULTI` | دعم الـ multi-index entries — لازم يكون enabled للـ testing |
| `CONFIG_DEBUG_KERNEL` | تفعيل عام للـ debugging infrastructure |
| `CONFIG_SLUB_DEBUG` | debugging على الـ slab allocator (يأثر على xa_node) |
| `CONFIG_SLUB_DEBUG_ON` | تفعيل SLUB debug تلقائياً من الـ boot |
| `CONFIG_KASAN` | اكتشاف use-after-free وbuffer overflow في xa_node |
| `CONFIG_LOCKDEP` | التحقق من صحة استخدام xa_lock |
| `CONFIG_DEBUG_SPINLOCK` | اكتشاف misuse للـ spinlock في xa_lock |
| `CONFIG_RCU_TRACE` | تتبع مشاكل الـ RCU المستخدمة في xa_load |
| `CONFIG_DEBUG_OBJECTS` | تتبع lifetime بتاع الـ objects |
| `CONFIG_PROVE_LOCKING` | إثبات صحة الـ locking order |
| `CONFIG_KCSAN` | اكتشاف data races في الـ concurrent XArray access |

```bash
# التحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'XARRAY|SLUB_DEBUG|LOCKDEP|KASAN'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# crash utility: فحص XArray في kernel dump
# بعد تحميل vmcore:
crash> struct xarray <address>
crash> struct xa_node <address>

# فحص page cache XArray لـ inode معين
crash> struct address_space.i_pages <inode_addr>

# /proc/slabinfo لمراقبة xa_node
watch -n 1 'grep xa_node /proc/slabinfo'

# vmstat لمراقبة الـ memory المستخدمة من XArray
vmstat -m | grep xa_node
```

```bash
# كسب إحصائيات الـ allocator
cat /proc/meminfo | grep -i slab
```

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `BUG: KASAN: use-after-free in xas_store` | الـ xa_node اتحرر وفي حد لسه بيستخدمه | تأكد إن الـ RCU read lock محفوظ أثناء الـ traversal |
| `WARNING: CPU: X PID: Y at lib/xarray.c:Z xa_store` | WARN_ON اتفعل داخل XArray | اتحقق من الـ stack trace، غالباً stored IS_ERR() pointer |
| `kernel BUG at lib/xarray.c:` | assertion فشلت، زي تخزين internal entry | متخزنيش pointers بـ bits 0,1 مش zero |
| `BUG: sleeping function called from invalid context` | استدعاء xa_store بـ GFP_KERNEL وheld spinlock | استخدم GFP_NOWAIT أو GFP_ATOMIC في atomic context |
| `INFO: possible circular locking dependency` | lockdep شاف deadlock محتمل في xa_lock | راجع الـ locking order، متأخذش xa_lock جوا lock تاني |
| `Kernel panic - not syncing: corrupted XArray` | الـ tree structure اتخرب | check for concurrent unsynchronized writes |
| `SLUB: Unable to allocate memory on node` | xa_node allocation فشلت | راجع memory pressure، استخدم GFP_NOWAIT |
| `general protection fault in xa_load` | الـ xarray pointer نفسه corrupt أو freed | تأكد من lifetime بتاع الـ xarray struct |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* التحقق من إن الـ entry مش internal قبل التخزين */
void my_xa_store(struct xarray *xa, unsigned long idx, void *entry)
{
    /* WARN لو حاول حد يخزن internal entry */
    if (WARN_ON(xa_is_internal(entry)))
        return;

    /* WARN لو الـ pointer مش aligned */
    if (WARN_ON((unsigned long)entry & 3))
        return;

    xa_store(xa, idx, entry, GFP_KERNEL);
}

/* التحقق من الـ locking في advanced API */
void my_xas_store(struct xa_state *xas, void *entry)
{
    /* لازم الـ xa_lock يكون held */
    lockdep_assert_held(&xas->xa->xa_lock);

    xas_store(xas, entry);

    /* dump_stack لو في error */
    if (xas_error(xas)) {
        pr_err("XArray store failed: %d\n", xas_error(xas));
        dump_stack();
    }
}

/* مراقبة node allocation failures */
static void *xa_node_alloc_debug(struct xa_state *xas, gfp_t gfp)
{
    void *node = kmem_cache_alloc(xa_node_cachep, gfp);
    if (WARN_ON_ONCE(!node && !(gfp & __GFP_NOWARN)))
        dump_stack();
    return node;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

الـ XArray هو pure software data structure — ما فيش hardware مباشر ليه. لكن المستخدمين بتوعه (زي الـ page cache وdevice drivers) ليهم hardware state:

```bash
# مثال: لو driver بيستخدم XArray لتخزين DMA buffers
# تأكد إن عدد الـ entries في XArray = عدد الـ DMA buffers الفعلية

# قراءة stats من driver (مثال: nvme)
cat /sys/class/nvme/nvme0/queue_count

# مقارنة مع XArray size عن طريق crash أو /proc/
cat /proc/driver/my_driver/xa_stats  # لو الـ driver بيعمل expose
```

```bash
# لو XArray بيتستخدم في iommu mapping
cat /sys/kernel/debug/iommu/amd/iova_ranges
# أو
cat /sys/kernel/debug/iommu/intel/domains
```

#### 2. Register Dump Techniques

الـ XArray pure software، لكن لو debugging driver يستخدم XArray لـ MMIO mappings:

```bash
# قراءة register بـ devmem2 (لو مثبت)
devmem2 0xFEA00000 w

# أو بـ /dev/mem مع xxd
dd if=/dev/mem bs=4 count=1 skip=$((0xFEA00000/4)) 2>/dev/null | xxd

# io utility لـ port I/O
io -4 0x3F8
```

```bash
# لو الـ driver يعمل map لـ registers في XArray:
# استخدم crash لقراءة الـ virtual address المخزن

crash> px (*(void **)(<xa_addr> + <offset>))
crash> rd <virt_addr> 16
```

#### 3. Logic Analyzer / Oscilloscope Tips

لو الـ XArray بيُستخدم في real-time driver وفي timing issues:

- **Logic analyzer**: ضع triggers على الـ IRQ lines اللي بتستدعي `xa_store_irq()` أو `xa_erase_irq()`.
- راقب الـ time بين الـ IRQ وبين الـ kernel processing — لو طويل، ممكن يبقى الـ XArray lock contention.
- استخدم **gpio-based timestamps** في الكود:

```c
/* في الـ IRQ handler */
gpiod_set_value(debug_gpio, 1);
xa_store_irq(&my_xa, index, entry);
gpiod_set_value(debug_gpio, 0);
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | السبب |
|---|---|---|
| Memory corruption (bad RAM) | `KASAN: use-after-free in xas_descend` | الـ xa_node data اتخرب في الـ RAM |
| PCIe AER error أثناء DMA | `pcieport: AER: ...` + `xa_load returns NULL unexpectedly` | الـ DMA buffer entry اتمسح بسبب error recovery |
| OOM killer | `Out of memory: Kill process` + `xa_node: oom` | الـ XArray فضل بيكبر ومفيش cleanup |
| CPU cache coherency (ARM) | `BUG: Bad page state` في page cache | مش عامل cache flush قبل ما تخزن في XArray |
| NUMA node allocation fail | `SLUB: Unable to allocate on node X` | الـ xa_node بيتطلب allocation من specific node |

```bash
# مراقبة EDAC (memory errors)
cat /sys/devices/system/edac/mc/mc0/ce_count  # correctable errors
cat /sys/devices/system/edac/mc/mc0/ue_count  # uncorrectable errors

# mcelog
mcelog --client
```

#### 5. Device Tree Debugging

الـ XArray مش مرتبط بالـ Device Tree مباشرة، لكن drivers اللي تستخدم XArray ممكن تكون DT-based:

```bash
# التحقق من الـ DT node بتاع الـ device
cat /proc/device-tree/soc/my-device/compatible

# مقارنة مع الـ driver المسجل
ls /sys/bus/platform/drivers/my-driver/

# التأكد إن الـ driver probe اشتغل وعمل init للـ XArray
dmesg | grep "my-driver: initialized XArray"

# لو XArray مش initialized صح بسبب DT mismatch:
dmesg | grep -i "probe failed\|no memory\|of_match"
```

```bash
# فحص DT overlays اللو في runtime overlay حدث مشكلة
ls /sys/kernel/config/device-tree/overlays/
cat /sys/kernel/config/device-tree/overlays/my-overlay/status
```

---

### Practical Commands

#### جاهز للنسخ — كل التقنيات

**1. مراقبة xa_node في الـ slab:**

```bash
# real-time monitoring
watch -n 1 'awk "/xa_node/{print \"objects:\", \$2, \"slabs:\", \$6}" /proc/slabinfo'
```

**مثال الـ output:**
```
Every 1.0s: awk "/xa_node/..."

objects: 2048  slabs: 71
```
لو الـ objects بيكبر مع الوقت بدون سبب → memory leak في XArray.

---

**2. ftrace على xa_store:**

```bash
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'xa_store xa_erase xa_alloc' > set_ftrace_filter
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace | head -100
echo nop > current_tracer  # cleanup
```

**مثال الـ output:**
```
# tracer: function
#
          <task>-1234  [002] ....  1234.567890: xa_store <-shmem_alloc_inode
          <task>-1234  [002] ....  1234.567891: xa_store <-pagecache_get_page
```
**التفسير**: شوف مين بيستدعي `xa_store` أكتر وبأي rate.

---

**3. lockdep لمشاكل الـ locking:**

```bash
# تأكد إن LOCKDEP enabled
zcat /proc/config.gz | grep CONFIG_LOCKDEP

# مشاهدة lockdep warnings
dmesg | grep -A 30 "possible circular locking"
dmesg | grep -A 30 "lock held"
```

**مثال warning:**
```
======================================================
WARNING: possible circular locking dependency detected
------------------------------------------------------
process/1234 is trying to acquire lock:
  (&xa->xa_lock){+.+.}-{2:2}
but task is already holding lock:
  (&mm->mmap_lock){++++}-{3:3}
```
**الحل**: اتحقق من الـ locking order — `mmap_lock` لازم ياخد بعد `xa_lock` مش قبله.

---

**4. KASAN لاكتشاف memory corruption:**

```bash
# تأكد من تفعيل KASAN
zcat /proc/config.gz | grep CONFIG_KASAN

# لو في crash بيظهر KASAN report:
dmesg | grep -A 50 "BUG: KASAN"
```

**مثال KASAN output:**
```
==================================================================
BUG: KASAN: use-after-free in xas_descend+0x4a/0x90
Read of size 8 at addr ffff888012345678 by task my_task/1234

Call Trace:
 xas_descend
 xas_load
 xa_load
 my_driver_lookup
```
**التفسير**: الـ `xa_node` اتحرر وفي حد لسه بيقرأ منه — غالباً RCU read lock مش محفوظ.

---

**5. crash utility — فحص XArray في kernel coredump:**

```bash
# بعد أخذ kdump
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /proc/vmcore

# في prompt الـ crash:
crash> sym xarray   # دور على global XArrays
crash> struct xarray 0xffffffff81234567
crash> struct xa_node 0xffff888012345000
crash> p ((struct xa_node *)0xffff888012345000)->slots[0]
```

---

**6. اكتشاف XArray leak بـ kmemleak:**

```bash
# تأكد من تفعيل kmemleak
zcat /proc/config.gz | grep KMEMLEAK

# تشغيل scan
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | head -50
```

**مثال output:**
```
unreferenced object 0xffff888012300000 (size 576):
  comm "my_thread", pid 1234, jiffies 4294967295
  backtrace:
    kmem_cache_alloc
    xa_node_alloc
    xas_expand
    xa_store
    my_driver_store_entry
```
**التفسير**: `xa_node` اتعمل allocate لكن ما اتحررش — `xa_destroy()` مش بيتاستدعى.

---

**7. perf لقياس performance:**

```bash
# قياس عدد calls على xa_store/xa_load
perf stat -e 'probe:xa_store,probe:xa_load' -p <pid> sleep 10

# أو بـ perf probe
perf probe --add xa_store
perf probe --add xa_load
perf record -e probe:xa_store,probe:xa_load -p <pid> sleep 5
perf report
```

---

**8. بناء kernel مع debug options:**

```bash
# في .config
scripts/config --enable CONFIG_KASAN
scripts/config --enable CONFIG_LOCKDEP
scripts/config --enable CONFIG_SLUB_DEBUG
scripts/config --enable CONFIG_DEBUG_SPINLOCK
scripts/config --enable CONFIG_PROVE_LOCKING
scripts/config --enable CONFIG_XARRAY_MULTI

make olddefconfig
make -j$(nproc)
```

---

**9. فحص إن الـ XArray فاضية:**

```bash
# عن طريق crash أو في الكود:
crash> p xa_empty(&my_xarray)
```

```c
/* في الكود بعد cleanup */
if (WARN_ON(!xa_empty(&my_xa))) {
    pr_err("XArray not empty after cleanup!\n");
    /* iterate وprint الـ remaining entries */
    unsigned long idx;
    void *entry;
    xa_for_each(&my_xa, idx, entry) {
        pr_err("  leaked entry at index %lu: %p\n", idx, entry);
    }
}
```

---

**10. تشخيص بطء xa_alloc_cyclic:**

```bash
# مراقبة الـ wrap-around
# في الكود:
pr_debug("xa_alloc_cyclic: next=%u, wrapped=%d\n", next_id, wrapped);

# فعّل dynamic debug:
echo 'func xa_alloc_cyclic +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep xa_alloc_cyclic
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — memory leak في driver بيستخدم XArray غلط

#### العنوان
**XArray بدون `xa_destroy()` في driver الـ I2C على gateway صناعي**

#### السياق
شركة بتبني industrial gateway بيشتغل على **RK3562** — بيجمع بيانات من 64 sensor عبر I2C. الـ driver بيخزن pointer لكل `sensor_data` struct في XArray بـ index هو عنوان الـ I2C slave. الـ gateway بيشتغل 24/7 وكل ساعتين بيحصل hot-reload للـ driver لما sensor جديد يتضاف.

#### المشكلة
بعد أسبوع تشغيل، الـ kernel بيـlog:
```
kmemleak: 47 new suspected memory leaks
```
وذاكرة الـ kernel بتزيد ببطء حتى يحصل OOM.

#### التحليل
الـ driver بيعمل `xa_init()` في `probe()` ويخزن:
```c
/* store sensor_data pointer at I2C address index */
xa_store(&gateway->sensors, client->addr, sensor, GFP_KERNEL);
```

لكن في `remove()`:
```c
/* BUG: only frees the pointer, doesn't clean the XArray itself */
kfree(xa_load(&gateway->sensors, client->addr));
xa_erase(&gateway->sensors, client->addr);
/* xa_destroy() never called! */
```

المشكلة: لما الـ driver يتـunload وتتـfree الـ `gateway` struct، الـ XArray نفسه ممكن يكون فيه internal nodes — الـ `xa_node` structs اللي XArray خصصها من الـ slab. الـ `xa_destroy()` هو اللي بيـwalk الـ tree ويـfree كل الـ `xa_node` objects. بدونه، الـ nodes بتفضل في الـ slab وما بترجعش.

الـ doc بتقول صراحة:
> *"you can remove all entries from an XArray by calling `xa_destroy()`"*

#### الحل
```c
static int gateway_remove(struct platform_device *pdev)
{
    struct gateway_dev *gateway = platform_get_drvdata(pdev);
    struct sensor_data *sensor;
    unsigned long index;

    /* first free all stored pointers */
    xa_for_each(&gateway->sensors, index, sensor) {
        kfree(sensor);
    }

    /* then destroy the XArray to free internal xa_node objects */
    xa_destroy(&gateway->sensors);

    return 0;
}
```

```bash
# للتحقق بعد الإصلاح
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
**الـ `xa_destroy()` مش اختياري** — حتى لو حذفت كل الـ entries بـ `xa_erase()`، الـ XArray ممكن يكون خصص `xa_node` structs داخليًا ولازم تتـfree. أي driver بيستخدم XArray لازم يستدعي `xa_destroy()` في مسار الـ cleanup.

---

### السيناريو 2: Android TV Box على Allwinner H616 — deadlock بسبب غلط في locking مع interrupt context

#### العنوان
**deadlock في USB hotplug handler بسبب استخدام `xa_store()` من interrupt context**

#### السياق
TV box بيشغل Android على **Allwinner H616**. الـ USB subsystem بيستخدم XArray لتتبع الـ connected devices. الـ developer أضاف custom HID driver بيخزن device info في XArray مشترك. الـ `xa_store()` بيتاستدعى من process context، والـ disconnect handler بيتاستدعى من softirq.

#### المشكلة
الجهاز بيـfreeze بشكل عشوائي لما USB device يتوصل أو يتنزع بسرعة. الـ serial console بيظهر:
```
INFO: task kworker:0:2 blocked for more than 120 seconds.
Possible circular locking dependency detected
```

#### التحليل
الـ driver اتعمل كده:
```c
/* init without specifying lock type — wrong for mixed contexts */
xa_init(&hid_devices);  /* defaults: no IRQ/BH protection */

/* called from process context — takes xa_lock (spinlock) */
int hid_connect(struct usb_device *udev) {
    return xa_err(xa_store(&hid_devices, udev->devnum, info, GFP_KERNEL));
}

/* called from softirq (USB disconnect BH) — also tries xa_lock */
void hid_disconnect(struct usb_device *udev) {
    xa_erase(&hid_devices, udev->devnum);  /* BUG: softirq hits xa_lock */
}
```

الـ doc تقول:
> *"If you want to modify the XArray from interrupt or softirq context, you need to initialise the array using `xa_init_flags()`, passing `XA_FLAGS_LOCK_IRQ` or `XA_FLAGS_LOCK_BH`"*

لما `xa_store()` في process context حاجز الـ spinlock وجه softirq حاول يحجز نفس الـ lock، حصل deadlock لأن الـ spinlock مش IRQ-safe بالـ default.

#### الحل
```c
/* correct init: tell XArray that BH context will use it */
xa_init_flags(&hid_devices, XA_FLAGS_LOCK_BH);

/* process context: must use _bh variant to match */
int hid_connect(struct usb_device *udev) {
    void *ret;
    /* xa_store_bh disables BH while holding lock */
    ret = xa_store_bh(&hid_devices, udev->devnum, info, GFP_KERNEL);
    return xa_err(ret);
}

/* softirq context: plain xa_erase is now safe */
void hid_disconnect(struct usb_device *udev) {
    xa_erase_bh(&hid_devices, udev->devnum);
}
```

```bash
# reproduce with lockdep enabled kernel
echo 1 > /proc/sys/kernel/prove_locking
# plug/unplug USB rapidly — lockdep will catch it before deadlock
dmesg | grep -i "locking dependency"
```

#### الدرس المستفاد
الـ `XA_FLAGS_LOCK_BH` / `XA_FLAGS_LOCK_IRQ` مش مجرد optimization — هم ضروريين لأمان الـ locking. الـ XArray مش بيعرف تلقائيًا إنك هتستخدمه من context متعددة؛ لازم تخبره وقت الـ init.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — bug في XArray cyclic allocation مع ID overflow

#### العنوان
**`xa_alloc_cyclic()` بيرجع IDs قديمة فوق resources مش اتحررت على STM32MP1**

#### السياق
IoT device على **STM32MP1** بيدير MQTT sessions. كل session بياخد ID فريد من XArray بـ `xa_alloc_cyclic()`. الـ system محدود الذاكرة — الـ max sessions 256. بعد تشغيل شهر، الـ device بدأ يخلط sessions: رسائل MQTT بتوصل لـ session غلط.

#### المشكلة
```
[ 2847291.443] mqtt: session 0x4a already active, new request ignored
[ 2847291.444] mqtt: data mismatch on session 0x4a
```

#### التحليل
الـ code الأصلي:
```c
DEFINE_XARRAY_ALLOC(&mqtt_sessions);
static u32 next_id = 0;

int mqtt_new_session(struct session_info *info) {
    u32 id;
    int ret;

    /* allocate cyclic ID in range [0, 255] */
    ret = xa_alloc_cyclic(&mqtt_sessions, &id, info,
                          XA_LIMIT(0, 255), &next_id, GFP_KERNEL);
    /* BUG: not checking return value properly */
    if (ret < 0)
        return ret;
    return id;
}
```

المشكلة: `xa_alloc_cyclic()` بترجع `1` لما الـ ID wrap-around حصل (رجعنا من 255 لـ 0). الـ developer كان بيتحقق بـ `if (ret < 0)` بس، فالـ wrap-around اتعامل معاه كـ success عادي. لكن المشكلة الأكبر: sessions قديمة ما اتمسحتش بـ `xa_erase()` قبل الـ wrap، فـ `xa_alloc_cyclic()` لقى slots مشغولة وتخطاها — لكن في حالة معينة network burst سببت كل الـ 256 slot تتملا ومفيش `xa_erase()` اتستدعى في الوقت المناسب.

الـ doc تشرح:
> *"If you want to allocate IDs up to a maximum, then wrap back around to the lowest free ID, you can use `xa_alloc_cyclic()`"*

الـ `DEFINE_XARRAY_ALLOC` بيستخدم `XA_FLAGS_ALLOC` اللي بيعتبر `NULL` stored = reserved entry.

#### الحل
```c
DEFINE_XARRAY_ALLOC(&mqtt_sessions);
static u32 next_id = 0;

int mqtt_new_session(struct session_info *info) {
    u32 id;
    int ret;

    ret = xa_alloc_cyclic(&mqtt_sessions, &id, info,
                          XA_LIMIT(0, 255), &next_id, GFP_KERNEL);

    if (ret == -EBUSY) {
        /* all 256 slots full — reject new connections */
        return -EAGAIN;
    }
    if (ret < 0)
        return ret;

    /* ret == 1 means wrapped around — valid but worth logging */
    if (ret == 1)
        dev_warn(dev, "session ID wrapped around\n");

    return (int)id;
}

void mqtt_close_session(u32 id) {
    struct session_info *info;

    /* must erase to free the slot for cyclic reuse */
    info = xa_erase(&mqtt_sessions, id);
    kfree(info);
}
```

#### الدرس المستفاد
`xa_alloc_cyclic()` بترجع `1` عند الـ wrap-around — مش error. ولازم دايمًا تتأكد إن الـ `xa_erase()` بيتاستدعى على كل entry مش محتاجها؛ وإلا الـ cyclic allocation هتـstall لما تملا كل الـ range.

---

### السيناريو 4: Custom Board Bring-up على i.MX8 — استخدام Advanced API غلط مع `xas_pause()` وcorrupted state

#### العنوان
**iterator بيفوت entries بعد `xas_pause()` من غير `xa_lock` صح على i.MX8**

#### السياق
شركة بتعمل bring-up لـ custom board على **i.MX8MP** — media processing unit. الـ driver بيدير buffer pool: كل buffer بياخد ID في XArray. محتاجين iterate على كل الـ buffers وبعدين يحرروا بعضها. لأن العملية بتاخد وقت، الـ developer استخدم `xas_pause()` عشان يـrelease الـ lock في النص.

#### المشكلة
بعض الـ buffers بتتحرر مرتين (`double free`)، وبعضها ما بيتحررش خالص. الـ KASAN بيـlog:
```
BUG: KASAN: double-free or invalid-free in media_buf_cleanup+0x84
```

#### التحليل
الـ code الغلط:
```c
void media_buf_cleanup(struct media_dev *mdev) {
    XA_STATE(xas, &mdev->buffers, 0);
    struct media_buffer *buf;

    rcu_read_lock();
    xas_for_each(&xas, buf, ULONG_MAX) {
        if (buf->flags & BUF_NEEDS_FREE) {
            /* BUG 1: dropping RCU lock without xas_pause() */
            rcu_read_unlock();
            free_media_buffer(buf);  /* takes mutex internally */
            rcu_read_lock();
            /* BUG 2: xas state is now stale — may revisit same entry */
        }
    }
    rcu_read_unlock();
}
```

الـ doc تقول:
> *"If you need to drop whichever of those locks is protecting your state and tree, you must call `xas_pause()` so that future calls do not rely on the parts of the state which were left unprotected."*

بدون `xas_pause()`، الـ `xa_state` بيظل pointing لنفس node في الـ tree. لما ترجع من الـ `rcu_read_unlock()` وتـ`rcu_read_lock()` تاني، الـ node ممكن يكون اتحرر وإعيد استخدامه (RCU grace period عدى). الـ iterator بعدين بيـtraverse state قديمة ويرجع entries اتعملت free.

#### الحل
```c
void media_buf_cleanup(struct media_dev *mdev) {
    XA_STATE(xas, &mdev->buffers, 0);
    struct media_buffer *buf;
    LIST_HEAD(to_free);

    /* Phase 1: collect buffers to free under RCU, don't free inside loop */
    rcu_read_lock();
    xas_for_each(&xas, buf, ULONG_MAX) {
        if (buf->flags & BUF_NEEDS_FREE) {
            /* pause before dropping lock — saves cursor position */
            xas_pause(&xas);
            rcu_read_unlock();

            /* do work outside lock */
            list_add(&buf->free_list, &to_free);

            rcu_read_lock();
            /* iterator resumes correctly after pause */
        }
    }
    rcu_read_unlock();

    /* Phase 2: erase from XArray then free */
    list_for_each_entry_safe(buf, tmp, &to_free, free_list) {
        xa_erase(&mdev->buffers, buf->id);
        free_media_buffer(buf);
    }
}
```

```bash
# enable KASAN on i.MX8 dev kernel to catch this early
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y
```

#### الدرس المستفاد
`xas_pause()` مش اختياري لما بتـdrop الـ lock جوه `xas_for_each()`. بدونه الـ xa_state بيفضل يشاور على tree data ممكن تكون stale بعد RCU grace period. الأفضل دايمًا: اجمع الـ entries اللي عايز تشتغل عليها تحت الـ lock، وبعدين اشتغل عليها بره الـ lock.

---

### السيناريو 5: Automotive ECU على AM62x — غلط في استخدام multi-index entries مع UART frame management

#### العنوان
**`xa_store_range()` بيخزن frame entries غلط في automotive ECU على AM62x**

#### السياق
**AM62x**-based automotive ECU بيدير CAN-to-UART bridge. كل UART frame بياخد range من indices في XArray تمثل الـ byte offsets. الـ driver بيستخدم multi-index entries عشان يربط كل bytes الـ frame بـ single `frame_desc` pointer. محتاجين iterate على الـ frames بكفاءة.

#### المشكلة
لما بيتعمل store لـ frame بعد frame، بعض الـ byte ranges بتتـoverwrite ببعضها. الـ UART data corruption بيظهر كـ garbled output:
```
uart_rx: expected 64 bytes frame, got 48 — possible range collision
```

#### التحليل
الـ code الأصلي:
```c
/* UART frame: start byte offset and length */
int uart_store_frame(struct uart_dev *udev,
                     unsigned long start, unsigned long len,
                     struct frame_desc *desc) {

    /* BUG: start=10, len=6 → trying range 10-15 */
    /* but xa_store_range requires aligned power-of-two ranges! */
    return xa_err(xa_store_range(&udev->frames, start, start + len - 1,
                                 desc, GFP_KERNEL));
}
```

الـ doc تقول بوضوح:
> *"The current implementation only allows tying ranges which are aligned powers of two together; eg indices 64-127 may be tied together, but 2-6 may not be."*

الـ range `10-15` مش aligned power-of-two، فالـ XArray مش بيربطها كـ multi-index entry صح — بيخزن entries منفصلة بـ behavior غير متوقع. لما frame تانية بتبدأ من `12`، بتـoverwrite جزء من range الأولى.

الحل الصح هو إما:
1. تـround الـ ranges لـ aligned power-of-two boundaries.
2. تستخدم XArray عادي بـ single index لكل frame وتخزن الـ length في الـ `frame_desc`.

#### الحل
```c
/* Option A: use simple single-index XArray — one entry per frame */
struct frame_desc {
    unsigned long start;  /* byte offset */
    unsigned long len;    /* frame length */
    void *data;
};

int uart_store_frame(struct uart_dev *udev,
                     unsigned long frame_id,
                     struct frame_desc *desc) {
    /* index = frame_id, not byte offset — no range issues */
    return xa_err(xa_store(&udev->frames, frame_id, desc, GFP_KERNEL));
}

/* Option B: if multi-index is truly needed, align to power-of-two */
unsigned long aligned_start = round_down(start, roundup_pow_of_two(len));
unsigned long aligned_end   = aligned_start + roundup_pow_of_two(len) - 1;

XA_STATE_ORDER(xas, &udev->frames, aligned_start,
               ilog2(roundup_pow_of_two(len)));
xas_lock(&xas);
xas_store(&xas, desc);
xas_unlock(&xas);
```

```bash
# على AM62x: تحقق من الـ XArray state عبر debugfs أو ftrace
echo 'xa_store_range:return' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/events/enable
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
**Multi-index entries في XArray مش general-purpose range storage** — هم مخصوصين لـ aligned power-of-two ranges زي page cache order-N pages. لو الـ use case مش بيتطابق مع هذا الشرط (زي arbitrary UART frame lengths)، استخدم single-index XArray وخزن الـ range metadata في الـ struct نفسها.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي غطّت الـ XArray من أول ما اتقدم لحد ما اتدمج في الـ mainline:

| المقال | الوصف |
|--------|--------|
| [Introducing the eXtensible Array (xarray)](https://lwn.net/Articles/715948/) | أول تقديم رسمي للـ XArray من Matthew Wilcox — بيشرح ليه الـ radix tree محتاج بديل |
| [The XArray data structure](https://lwn.net/Articles/745073/) | تغطية linux.conf.au 2018 — بيوضح الـ API الجديد والفرق عن الـ radix tree |
| [XArray documentation](https://lwn.net/Articles/739986/) | مناقشة patch الـ documentation الأولانية |
| [XArray and the mainline](https://lwn.net/Articles/757342/) | قصة محاولات الـ merge وإزاي وصل للـ 4.20 mainline |
| [Convert page cache to XArray](https://lwn.net/Articles/750540/) | تفاصيل تحويل الـ page cache — أهم use case للـ XArray في الكيرنل |
| [XArray v8](https://lwn.net/Articles/748680/) | مناقشة الـ patch set في نسخته الثامنة قبل الـ merge |

---

### التوثيق الرسمي في الكيرنل

**الـ XArray** موثَّق في أكتر من مكان في شجرة الكيرنل:

```
Documentation/core-api/xarray.rst          # التوثيق الرئيسي
include/linux/xarray.h                     # تعريف struct xarray و كل الـ API
lib/xarray.c                               # الـ implementation
lib/test_xarray.c                          # unit tests — مرجع عملي ممتاز
tools/testing/radix-tree/xarray-test.c     # userspace test harness
```

**الـ online version:**
- [docs.kernel.org — XArray](https://docs.kernel.org/core-api/xarray.html)
- [static.lwn.net — XArray kernel docs](https://static.lwn.net/kerneldoc/core-api/xarray.html)

---

### Commits المهمة في تاريخ الـ XArray

#### دمج الـ XArray في الـ 4.20
الـ pull request الرسمي اللي دمج الـ XArray في Linux 4.20:
- [GIT PULL — XArray for 4.20](https://lkml.iu.edu/hypermail/linux/kernel/1810.2/06430.html)
- الـ commit الأساسي عرّف `struct xarray` وبنية الـ `xa_node` ونظام الـ marks

#### Commits تانية مهمة
```bash
# تعريف struct xarray الأول (patch v4)
# https://lore.kernel.org/linux-btrfs/20171206004159.3755-7-willy@infradead.org/

# MAINTAINERS entry
# https://lkml.org/lkml/2017/12/5/1068

# patch v10 — النسخة قبل الأخيرة
# https://lore.kernel.org/linux-f2fs-devel/20180330034245.10462-17-willy@infradead.org/
```

#### تحويل الـ page cache
تحويل الـ page cache من الـ radix tree للـ XArray كان أضخم تغيير:
```bash
git log --oneline --all -- mm/filemap.c | head -20
```

---

### نقاشات الـ Mailing List

أهم النقاشات اللي شكّلت الـ XArray:

| الرابط | الموضوع |
|--------|---------|
| [LKML patch v4](https://lkml.org/lkml/2017/12/5/1068) | النقاش الأول حول إضافة MAINTAINERS entry |
| [LKML patch v6](https://lkml.kernel.org/linux-xfs/20180117202203.19756-6-willy@infradead.org/) | نسخة v6 — 99-patch series |
| [lore.kernel.org — struct xarray definition](https://lore.kernel.org/linux-btrfs/20171206004159.3755-7-willy@infradead.org/) | patch بيعرّف `struct xarray` لأول مرة |
| [LKML GIT PULL for 4.20](https://lkml.iu.edu/hypermail/linux/kernel/1810.2/06430.html) | الـ pull request الرسمي لـ Linus |

---

### مصادر kernelnewbies.org

- [InternalKernelDataTypes](https://kernelnewbies.org/InternalKernelDataTypes) — صفحة بتغطي data structures الداخلية في الكيرنل بما فيها الـ XArray
- [KernelProjects/large-block-size](https://kernelnewbies.org/KernelProjects/large-block-size) — بيوضح إزاي الـ XArray بيمكّن الـ block size أكبر من الـ page size
- [Linux_6.16 release notes](https://kernelnewbies.org/Linux_6.16) — بيشمل إضافة Rust abstractions للـ XArray
- [Linux_6.17 release notes](https://kernelnewbies.org/Linux_6.17) — تحسينات الـ XArray للـ extent buffers

---

### Presentation من linux.conf.au 2018

Matthew Wilcox قدّم الـ XArray في الـ Kernel miniconf:

- **العنوان:** "Replacing the Radix Tree"
- **الـ slides:** [ozlabs.org — Wilcox Replacing the Radix Tree (PDF)](https://lca-kernel.ozlabs.org/2018-Wilcox-Replacing-the-Radix-Tree.pdf)
- **النقطة الجوهرية:** الـ XArray بيعالج مشكلة الـ locking اللي كانت مسؤولية المستخدم في الـ radix tree

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل ذو الصلة:** Chapter 15 — Memory Mapping and DMA (بيشرح الـ page cache اللي بيستخدم الـ XArray)
- **متاح مجانًا:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل ذو الصلة:** Chapter 12 — Memory Management
- بيشرح الـ page cache وإزاي الـ data structures الداخلية بتشتغل
- الـ XArray دخل بعد إصدار الكتاب، بس الـ context مهم لفهم الـ use cases

#### Understanding the Linux Kernel — Bovet & Cesati
- **الفصل ذو الصلة:** Chapter 15 — The Page Cache
- بيفيدك تفهم ليه الـ page cache محتاج data structure متخصصة زي الـ XArray

#### Embedded Linux Primer — Christopher Hallinan
- الكتاب بيتناول استخدام الكيرنل في embedded systems
- الـ XArray مهم في هذا السياق لأنه memory-efficient أكتر من الـ linked lists

---

### GitHub — كود الـ XArray

| الملف | الرابط |
|-------|--------|
| `include/linux/xarray.h` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/xarray.h) |
| `lib/xarray.c` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/lib/xarray.c) |
| التوثيق الرسمي `.rst` | [github.com/torvalds/linux](https://github.com/torvalds/linux/blob/master/Documentation/core-api/xarray.rst) |

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
"xarray linux kernel" site:lwn.net
"xa_state" linux kernel internals
"xarray vs radix tree" linux

# بحث في الكود
git log --all --oneline --grep="xarray"
git log --all --oneline --grep="XArray"
grep -r "xa_store\|xa_load\|xa_alloc" --include="*.c" /path/to/linux/

# بحث في الـ mailing list
lore.kernel.org/linux-mm/?q=xarray
lore.kernel.org/linux-fsdevel/?q=xarray

# بحث في الـ API
man -k xa_   # مش هيشتغل في الكيرنل مباشرةً، بس في الـ userspace docs
```

---

### use cases حقيقية في الكيرنل للرجوع إليها

كود بيستخدم الـ XArray فعلًا في الكيرنل — مراجعته بتفيد في الفهم:

```c
/* mm/filemap.c — page cache: أهم use case */
struct address_space {
    struct xarray       i_pages;   /* XArray بدل الـ radix tree */
    /* ... */
};

/* drivers/gpu/drm/ — DRM subsystem بيستخدم XArray للـ objects */
/* fs/inode.c — inode cache */
/* net/ — networking structures */
```

للبحث عن الـ use cases:
```bash
grep -r "DEFINE_XARRAY\|xa_init\|XA_FLAGS" \
    --include="*.c" --include="*.h" \
    /path/to/linux/mm/ \
    /path/to/linux/drivers/ | head -30
```
## Phase 8: Writing simple module

### الفكرة

**`xa_store()`** هي الـ function الأكثر استخداماً في الـ XArray API — بتخزّن أي pointer في الـ array على index معين وبترجع القيمة القديمة. الـ page cache بيستخدمها بشكل مكثّف لتسجيل الـ pages. هنعمل **kprobe** على `xa_store` عشان نشوف كل مرة حد بيخزّن entry جديد في أي XArray في الـ kernel.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_xastore.c - Hook xa_store() to observe XArray write activity
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/xarray.h>      /* struct xarray, xa_is_value() helpers */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on xa_store() to trace XArray write operations");

/*
 * xa_store() signature:
 *   void *xa_store(struct xarray *xa, unsigned long index,
 *                  void *entry, gfp_t gfp);
 *
 * On x86-64 the args land in: rdi=xa, rsi=index, rdx=entry, rcx=gfp
 * We read them from pt_regs directly — no need to touch the live stack.
 */

/* pre-handler: يتنفذ قبل ما xa_store تبدأ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* استخراج الـ arguments من الـ registers حسب الـ x86-64 ABI */
    struct xarray *xa   = (struct xarray *)regs->di; /* arg1 */
    unsigned long index = (unsigned long)regs->si;    /* arg2 */
    void         *entry = (void *)regs->dx;           /* arg3 */

    /* تحديد نوع الـ entry المخزون */
    const char *kind;
    if (!entry)
        kind = "NULL (erase)";
    else if (xa_is_value(entry))
        kind = "value-entry";
    else
        kind = "pointer";

    pr_info("kprobe/xa_store: xa=%px index=%lu entry=%px [%s] comm=%s pid=%d\n",
            xa, index, entry, kind,
            current->comm,   /* اسم الـ process الحالي */
            current->pid);

    return 0; /* 0 = تكمل التنفيذ الطبيعي */
}

/* post-handler: يتنفذ بعد ما xa_store تخلص (اختياري هنا للتوضيح) */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* القيمة المرجعة (الـ entry القديم) موجودة في rax */
    void *old_entry = (void *)regs->ax;

    pr_info("kprobe/xa_store: returned old_entry=%px [%s]\n",
            old_entry,
            xa_is_value(old_entry) ? "value-entry" : "pointer-or-null");
}

/* تعريف الـ kprobe struct */
static struct kprobe xa_store_kp = {
    .symbol_name = "xa_store",   /* الـ function اللي هنخطف تنفيذها */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ---- init ---- */
static int __init xastore_probe_init(void)
{
    int ret;

    /* تسجيل الـ kprobe مع الـ kernel */
    ret = register_kprobe(&xa_store_kp);
    if (ret < 0) {
        pr_err("kprobe/xa_store: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe/xa_store: planted at %px\n", xa_store_kp.addr);
    return 0;
}

/* ---- exit ---- */
static void __exit xastore_probe_exit(void)
{
    /* لازم نشيل الـ kprobe قبل ما الـ module يتفرغ من الذاكرة */
    unregister_kprobe(&xa_store_kp);
    pr_info("kprobe/xa_store: removed\n");
}

module_init(xastore_probe_init);
module_exit(xastore_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info()` و `pr_err()` |
| `linux/kprobes.h` | `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/xarray.h` | `xa_is_value()` عشان نفرّق بين الـ value entries والـ pointers |

---

#### الـ `handler_pre` — قبل التنفيذ

الـ `pt_regs` بيحتوي على حالة الـ CPU registers لحظة اعتراض الـ call؛ على x86-64، الـ System V ABI بيحط الـ arguments الأولى في `rdi، rsi، rdx، rcx` بالترتيب — فبنقرأ منهم مباشرةً من غير ما نلمس الـ stack أو نعدّل الـ function. الـ return بـ 0 معناه "كمّل تنفيذ الـ function الأصلي طبيعي".

---

#### الـ `handler_post` — بعد التنفيذ

الـ post-handler بيتنفذ بعد ما `xa_store` تخلص؛ القيمة المرجعة (الـ entry القديم) بتبقى في `rax` — فبنطبعها عشان نشوف إيه اللي كان متخزّن قبل كده في نفس الـ index.

---

#### الـ `struct kprobe`

الـ `.symbol_name` بيخلّي الـ kernel يحلّ الرمز نفسه من الـ kallsyms بدل ما نحدد عنوان صلب — أسهل وأكثر portability. الـ `.pre_handler` و `.post_handler` هم الـ callbacks اللي هيتشالوا تلقائياً.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتزرع **breakpoint** (INT3 على x86) على بداية `xa_store` ووفقاً لده أي استدعاء للـ function بيعدّي من الـ handler قبل ما يكمل. `unregister_kprobe` في الـ `exit` ضرورية جداً — لو الـ module اتشال من الذاكرة والـ kprobe لسه مسجّل، أي استدعاء بعدين هيتنفذ على كود مش موجود وبتحصل **kernel panic**.

---

### كيفية البناء والتشغيل

```bash
# Makefile بسيط
cat > Makefile <<'EOF'
obj-m += kprobe_xastore.o
KDIR  ?= /lib/modules/$(shell uname -r)/build
all:
	make -C $(KDIR) M=$(PWD) modules
clean:
	make -C $(KDIR) M=$(PWD) clean
EOF

make
sudo insmod kprobe_xastore.ko

# شوف الـ output
sudo dmesg -w | grep "kprobe/xa_store"

# لما تخلص
sudo rmmod kprobe_xastore
```

---

### مثال على الـ output المتوقع

```
kprobe/xa_store: planted at ffffffffa1b23456
kprobe/xa_store: xa=ffff888103a00000 index=42 entry=ffff888104c80000 [pointer] comm=systemd pid=1
kprobe/xa_store: returned old_entry=0000000000000000 [pointer-or-null]
kprobe/xa_store: xa=ffff888103a00000 index=43 entry=0000000000000003 [value-entry] comm=kworker/0:1 pid=17
kprobe/xa_store: returned old_entry=0000000000000000 [pointer-or-null]
```

**الـ page cache** هو المستخدم الأكبر لـ `xa_store`، فبتشوف استدعاءات كتير من `kworker` threads وكمان من أي process بيقرأ ملفات — كل صفحة بتتخزّن في الـ page cache عن طريق `xa_store`.
