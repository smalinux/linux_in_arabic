## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ XArray؟

**الـ XArray** — اختصار لـ eXtensible Array — هو هيكل بيانات في الـ Linux kernel يشتغل كـ **array ضخمة** مؤشرها من 0 لـ `ULONG_MAX`، لكنها بتاخد في الميموري بس الأماكن اللي فيها قيم فعلاً — يعني sparse array ذكية.

الـ subsystem اللي ينتمي له الملف: **XARRAY** — صُمِّم بواسطة Matthew Wilcox من Microsoft سنة 2017، والـ mailing list الرئيسي هو `linux-fsdevel` و `linux-mm`.

---

### القصة من الأول

#### المشكلة اللي كانت موجودة

قبل الـ XArray، الـ kernel كان بيستخدم هيكلين منفصلين لنفس الغرض:

1. **الـ radix tree** — شجرة للبحث السريع بالـ index.
2. **الـ IDR** — لتخصيص IDs رقمية لأشياء زي file descriptors.

المشكلة: كلاهم معقدين، وصعب التعامل معهم بشكل آمن في بيئات concurrent، والـ locking كان مسؤولية المستخدم بالكامل، وكان في bugs كتير بسببهم.

#### الحل: الـ XArray

تخيل معايا: عندك خزانة بملايين الدرج، لكن 99% من الأدراج فاضية — مش عايز تدفع إيجار لكل الأدراج. الـ XArray بتاخد بس الأدراج اللي فيها حاجة فعلاً، وداخليا بتبني شجرة متعددة المستويات (**radix tree**) عشان توصل لأي درج في خطوات قليلة.

**الـ XArray** جمع الاتنين في واحد وأضاف:
- **locking مدمج** — `spinlock_t xa_lock` جوه الـ `struct xarray` نفسها.
- **RCU support** — القراءة lockless من خلال الـ RCU.
- **Marks (Tags)** — تقدر تحط علامة على أي entry وتبحث بيها بسرعة.
- **ID allocation** — تخصيص IDs تلقائي من نطاق معين.
- **Multi-index entries** — entry واحدة تغطي نطاق من الـ indices (مهمة للـ page cache).

---

### الصورة الكبيرة: كيف بيشتغل من الجوا

#### البنية الداخلية

```
struct xarray {
    spinlock_t  xa_lock;   // الـ lock الوحيد للحماية
    gfp_t       xa_flags;  // إعدادات الـ allocation والـ locking
    void __rcu *xa_head;   // رأس الشجرة (NULL لو فاضية)
};
```

**الـ xa_head** بيحمل 3 حالات:
- `NULL` — الـ array فاضية تماماً.
- pointer مباشر — لو في entry واحدة بس على index 0.
- pointer لـ `xa_node` — لو في أكتر من entry أو أي entry على index != 0.

#### الـ xa_node: اللبنة الأساسية

```
struct xa_node {
    unsigned char shift;        // كم bit كل slot بيغطي
    unsigned char offset;       // موقعه في الـ parent
    unsigned char count;        // عدد الـ slots غير الـ NULL
    unsigned char nr_values;    // عدد الـ value entries
    struct xa_node *parent;     // الـ node الأب
    struct xarray  *array;      // الـ xarray اللي ينتمي ليه
    void *slots[XA_CHUNK_SIZE]; // الـ slots (64 slot في الغالب)
    unsigned long marks[3][...];// بت ماب للـ marks على كل slot
};
```

الشجرة بتتكون من **chunks** كل chunk فيها 64 slot (على 64-bit systems)، وكل level بيغطي 6 bits من الـ index. يعني للوصول لأي من 2^64 index محتاج على الأكتر ~11 level.

```
ASCII: شكل الشجرة للـ indices 0..255 (8-bit مثال بـ chunk=4)

              xarray
                |
              xa_node (shift=4, covers bits 7:4)
           /    |    \    \
        node  node  node  node  (shift=0, covers bits 3:0)
        /|\    ...
    [0][1][2]...[15]
    entries
```

#### encoding الـ entries: الـ 2 bits السفلية

الـ XArray بيستخدم أقل 2 bits من كل pointer عشان يعرف نوع الـ entry:

| القيمة | النوع |
|--------|-------|
| `00` | Pointer عادي (user pointer) |
| `x1` | Value entry أو tagged pointer |
| `10` | Internal entry (للـ kernel فقط) |

بالطريقة دي بيحفظ في نفس الـ `void*` إما pointer لبيانات المستخدم، أو رقم integer صغير (value entry)، أو معلومات داخلية للشجرة.

---

### طبقتان للـ API

#### 1. الـ Normal API — للاستخدام العادي

سهلة وآمنة — بتاخد وبترجع الـ lock وحدها:

```c
xa_load(xa, index);           // قراءة entry
xa_store(xa, index, ptr, gfp); // كتابة entry
xa_erase(xa, index);          // مسح entry
xa_alloc(xa, &id, ptr, limit, gfp); // تخصيص ID تلقائي
xa_for_each(xa, index, entry) { ... } // iteration
```

#### 2. الـ Advanced API (xas_*) — للـ power users

بتستخدم `struct xa_state` كـ cursor يمشي في الشجرة، وانت المسؤول عن الـ locking:

```c
XA_STATE(xas, xa, index);    // إعلان الـ cursor على الـ stack
xas_load(&xas);              // تحميل entry من موقع الـ cursor
xas_store(&xas, entry);      // كتابة في موقع الـ cursor
xas_for_each(&xas, entry, max) { ... } // O(n) iteration
```

الـ Advanced API ضروري للـ page cache لأنه بيحتاج multi-index entries وكفاءة أعلى.

---

### الـ Marks (العلامات)

الـ XArray بيدعم 3 marks (`XA_MARK_0`, `XA_MARK_1`, `XA_MARK_2`) على كل entry. الميزة الذكية: الـ mark bits بتتنقل للـ parent nodes تلقائياً، فلو عايز تلاقي كل الـ entries اللي عليها mark معين، مش محتاج تمسح كل الـ array — بتمشي في الشجرة وبتتبع الـ marks فقط.

**الـ XA_FREE_MARK** بيستخدمه الـ IDR تحت الغطا لتتبع الـ slots الفاضية.

---

### قصة: الـ page cache والـ XArray

الـ page cache في Linux بيحتاج يربط كل page بـ index (رقم الصفحة في الملف). قبل الـ XArray، كان بيستخدم الـ radix tree مباشرة مع كود معقد للـ locking.

دلوقتي:
1. كل `struct address_space` (بتمثل ملف في الذاكرة) فيها `struct xarray i_pages`.
2. الـ page fault handler بيعمل `xa_load(i_pages, page_index)` بسرعة.
3. الـ writeback code بيعمل `xa_for_each_marked(i_pages, index, page, PAGECACHE_TAG_DIRTY)` عشان يلاقي الـ dirty pages.
4. الـ THP (Transparent Huge Pages) بيستخدم multi-index entries عشان يخزن huge page واحدة على 512 index بضربة واحدة.

الـ XArray خلى الكود أبسط وأسرع وأأمن.

---

### الملفات اللي بتكوّن الـ subsystem

| الملف | الدور |
|-------|-------|
| `include/linux/xarray.h` | الـ header الرئيسي — كل الـ API والـ structs والـ inline functions |
| `lib/xarray.c` | التنفيذ الفعلي لكل الـ functions غير الـ inline |
| `include/linux/idr.h` | الـ IDR API فوق الـ XArray (backward compatible) |
| `lib/idr.c` | تنفيذ الـ IDR |
| `lib/test_xarray.c` | unit tests شاملة للـ XArray |
| `tools/testing/radix-tree/` | أدوات اختبار وـ userspace simulation |
| `Documentation/core-api/xarray.rst` | التوثيق الرسمي |
| `Documentation/core-api/idr.rst` | توثيق الـ IDR |
| `rust/kernel/xarray.rs` | Rust bindings للـ XArray API |

### الملفات اللي بتستخدم الـ XArray بشكل مكثف

| الملف | الاستخدام |
|-------|-----------|
| `include/linux/mm_types.h` | `struct address_space` فيها `struct xarray i_pages` للـ page cache |
| `include/linux/fs.h` | `struct inode` تستخدم `i_mapping->i_pages` |
| `kernel/fork.c` | `mm->mm_mt` (maple tree فوق xarray) لـ VMA tracking |
| `drivers/*/` | معظم الـ drivers بتستخدم XArray لتخصيص IDs |
## Phase 2: شرح الـ XArray Framework

### المشكلة — ليه الـ XArray اتعمل أصلاً؟

قبل الـ XArray، الـ kernel كان بيستخدم **Radix Tree** (`struct radix_tree_root`) بشكل مباشر. الـ radix tree كانت بتحل مشكلة تخزين pointers بـ integer index بكفاءة، لكن كانت فيها مشاكل حقيقية:

1. **API معقد وغير آمن:** كل caller كان لازم يدير الـ locking بنفسه — وده كان مصدر bugs كتير.
2. **مفيش abstraction للـ locking context:** محتاج spinlock؟ IRQ-safe? BH-safe? كل ده كان مسؤولية الـ caller.
3. **الـ tagged pointers محدودة:** الـ radix tree كانت بتدعم marks، بس الـ value entries (integers stored as pointers) ما كانوش جزء رسمي من الـ API.
4. **الـ RCU reads ما كانوش safe بشكل واضح:** الفرق بين locked access و RCU lockless access ما كانش clear في الـ API.
5. **مفيش ID allocation built-in:** كل subsystem كان بيعمل wrapper خاص بيه زي `idr` و`ida`.

الـ XArray اتعمل سنة 2017 بواسطة Matthew Wilcox (Microsoft) عشان يحل كل الـ pain points دي في واجهة واحدة نظيفة.

---

### الحل — الـ Approach اللي اتخده الـ Kernel

الـ XArray بيقدم **sparse array من الـ `void *` مفهرسة بـ `unsigned long`**، مع:

- **Integrated locking:** الـ `struct xarray` بيحمل `spinlock_t` جوا، والـ API بيأخد ويرجع الـ lock تلقائياً.
- **Two-tier API:** Normal API (safe, simple, يأخد ويرجع الـ lock) + Advanced API (`xas_*`) للـ users اللي عايزين control أكتر.
- **Built-in value entries:** integers مخزنة كـ pointers بـ bit tagging.
- **Built-in marks:** ثلاث marks (`XA_MARK_0/1/2`) بتتنشر لفوق في الشجرة.
- **RCU-safe reads:** الـ `xa_load()` يشتغل تحت RCU lock بدون spinlock.
- **ID allocation:** `xa_alloc()` بتوفر IDR functionality داخل نفس الـ data structure.

الـ internal implementation هي **radix tree** مش array حرفية — الاسم "array" هو الـ abstraction للـ user مش الـ implementation.

---

### التشبيه الحقيقي — مستودع كتب ذكي

تخيل **مستودع كتب ضخم** زي مستودع أمازون:

| عنصر المستودع | يقابل في الـ XArray |
|---|---|
| رقم الـ slot في الـ رف | الـ `index` (unsigned long) |
| الـ كتاب المخزن | الـ `entry` (void pointer) |
| الـ رف نفسه (64 slot) | الـ `xa_node` مع `slots[64]` |
| الـ طوابق والأقسام (hierarchy) | مستويات الـ radix tree |
| الـ فهرس الرئيسي على الباب | الـ `struct xarray` (xa_head) |
| قفل المستودع العام | الـ `xa_lock` (spinlock) |
| ورقة ملصقة حمراء على الرف | الـ mark (XA_MARK_0/1/2) |
| موظف بيجيب كتاب بالسرعة | الـ `xa_load()` تحت RCU |
| مدير المستودع بالمفتاح | الـ `xa_store()` مع lock |
| موظف متمرس بيعمل batch operations | الـ Advanced API `xas_*` |
| "الـ slot ده محجوز" | الـ `XA_ZERO_ENTRY` (reserved) |
| ملاحظة "حاول تاني" | الـ `XA_RETRY_ENTRY` |

**الفرق الجوهري عن مجرد "مستودع":** الـ marks بتتنشر من الـ leaf nodes لأعلى. يعني لو رففت كتاب واتعلمت عليه علامة "مهم"، كل node أب بيحفظ في الـ bitmap بتاعته إن في واحد من أبنائه عنده العلامة دي. النتيجة: تقدر تعمل `xa_find_marked()` بـ O(log n) وتقفز على الـ subtrees اللي مفيهاش الـ mark من غير ما تمشي كل الـ entries.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Linux Kernel                                │
├─────────────────────────────────────────────────────────────────────┤
│  CONSUMERS (subsystems بتستخدم XArray)                              │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │  Page Cache  │  │  VFS / inodes│  │  Device IDs  │             │
│  │ (address_sp) │  │  (i_pages)   │  │  (IDR-like)  │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                  │                      │
│  ┌──────┴─────────────────┴──────────────────┴──────┐              │
│  │              XArray API Layer                     │              │
│  │                                                   │              │
│  │   Normal API          │   Advanced API (xas_*)   │              │
│  │  xa_load/store/erase  │   xas_load/store/find    │              │
│  │  xa_alloc/find_marked │   xas_for_each_marked    │              │
│  │  xa_for_each          │   xas_create_range       │              │
│  └──────────────────────┬────────────────────────────┘             │
│                         │                                           │
│  ┌──────────────────────▼────────────────────────────┐             │
│  │              struct xarray                        │              │
│  │  ┌─────────────┐  ┌──────────┐  ┌─────────────┐ │              │
│  │  │  xa_lock    │  │ xa_flags │  │  xa_head    │ │              │
│  │  │ (spinlock_t)│  │ (gfp_t)  │  │ (void __rcu)│ │              │
│  │  └─────────────┘  └──────────┘  └──────┬──────┘ │              │
│  └─────────────────────────────────────────┼────────┘             │
│                                            │                        │
│  ┌─────────────────────────────────────────▼─────────────────────┐ │
│  │              Radix Tree (xa_node hierarchy)                    │ │
│  │                                                                │ │
│  │  xa_head ──► xa_node (root, shift=12)                         │ │
│  │               ├── slots[0] ──► xa_node (shift=6)              │ │
│  │               │                ├── slots[0] ──► user_ptr      │ │
│  │               │                ├── slots[1] ──► user_ptr      │ │
│  │               │                └── marks[3][1] (bitmaps)      │ │
│  │               ├── slots[1] ──► xa_node (shift=6)              │ │
│  │               └── marks[3][1] (propagated marks bitmaps)      │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Memory Subsystem (للـ xa_node allocation)                   │   │
│  │  kmem_cache / slab allocator ← GFP flags من xa_flags        │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ XArray بيقدم **sparse array من الـ `void *` indexed by `unsigned long`**، بس الجوهر الحقيقي هو:

> **كل entry في الـ XArray هو `void *` بيتفسر بناءً على الـ 2 bits الأدنى (tag bits).**

```
bit[1:0] = 00  →  Pointer entry  (user pointer, aligned, lower bits = 0)
bit[1:0] = x1  →  Value entry    (integer stored as pointer, bit0 = 1)
bit[1:0] = 10  →  Internal entry (kernel-internal, bit1 = 1, bit0 = 0)
```

الـ internal entries بيتقسموا:
- **0–62:** Sibling entries (لما entry واحدة بتاخد slots متعددة، الـ siblings بيشاوروا على الـ canonical slot)
- **256:** `XA_RETRY_ENTRY` — يعني "الـ tree اتغيرت، ابدأ تاني"
- **257:** `XA_ZERO_ENTRY` — slot محجوزة بس فاضية
- **negative (−4094 to −2):** Error codes مشفرة كـ internal entries

```c
/* encoding value 42 as XArray entry */
void *entry = xa_mk_value(42);
// entry = (void *)((42 << 1) | 1) = (void *)85
// bit0 = 1 → value entry

/* encoding pointer 0xffff8000 as XArray entry */
// pointer is naturally aligned → bit[1:0] = 00 → pointer entry

/* internal node pointer */
// xa_mk_node(node) = (void *)((unsigned long)node | 2)
// bit1 = 1, bit0 = 0 → internal entry
```

---

### الـ Key Structs وعلاقتها ببعض

#### `struct xarray` — الـ Handle الخارجي

```c
struct xarray {
    spinlock_t  xa_lock;   /* الـ lock الوحيد للـ array */
    gfp_t       xa_flags;  /* flags للـ allocation + lock type + initial marks */
    void __rcu *xa_head;   /* pointer للـ root: NULL, direct entry, أو xa_node */
};
```

الـ `xa_flags` بتحمل معلومات متعددة في نفس الـ field:
- `XA_FLAGS_LOCK_IRQ` / `XA_FLAGS_LOCK_BH`: نوع الـ lock context
- `XA_FLAGS_TRACK_FREE`: تفعيل ID allocation mode
- `XA_FLAGS_MARK(mark)`: marks أي marks متفعلة على مستوى الـ array

#### `struct xa_node` — الـ Internal Tree Node

```c
struct xa_node {
    unsigned char  shift;      /* كام bit كل slot بتمثل في الـ index */
    unsigned char  offset;     /* موقع الـ node ده في slots الـ parent */
    unsigned char  count;      /* عدد الـ non-NULL entries في slots[] */
    unsigned char  nr_values;  /* عدد الـ value entries */
    struct xa_node __rcu *parent; /* NULL للـ root */
    struct xarray  *array;    /* pointer للـ array الأم */
    union {
        struct list_head private_list; /* لـ page cache يستخدمه */
        struct rcu_head  rcu_head;     /* لـ RCU-safe free */
    };
    void __rcu *slots[XA_CHUNK_SIZE];           /* 64 slots (أو 16 في small config) */
    unsigned long marks[XA_MAX_MARKS][XA_MARK_LONGS]; /* bitmaps للـ 3 marks */
};
```

**الـ `shift` field مهم جداً:** كل node بـ `shift = S` يعني كل slot فيه بيمثل `2^S` من الـ index space. الـ root node بيكون عنده أعلى shift، والـ leaf nodes بيبقى عندها `shift = 0`.

```
Index space على 64-bit machine بـ XA_CHUNK_SHIFT = 6:

Level 3 (root): shift = 18, كل slot = 2^18 = 262144 entries
Level 2:        shift = 12, كل slot = 2^12 = 4096 entries
Level 1:        shift = 6,  كل slot = 2^6  = 64 entries
Level 0 (leaf): shift = 0,  كل slot = 1 entry مباشرة
```

#### `struct xa_state` — الـ Advanced API Cursor

```c
struct xa_state {
    struct xarray *xa;          /* الـ array اللي بنشتغل عليها */
    unsigned long  xa_index;    /* الـ index الحالي */
    unsigned char  xa_shift;    /* order للـ multi-slot entries */
    unsigned char  xa_sibs;     /* عدد الـ siblings لـ multi-slot */
    unsigned char  xa_offset;   /* الـ offset الحالي في الـ node */
    unsigned char  xa_pad;      /* padding لـ alignment */
    struct xa_node *xa_node;    /* الـ current node (أو magic values) */
    struct xa_node *xa_alloc;   /* pre-allocated node للـ store operations */
    xa_update_node_t xa_update; /* callback لما node يتحدث */
    struct list_lru  *xa_lru;   /* LRU list للـ page cache */
};
```

الـ `xa_node` بـ magic values:
- `XAS_RESTART = (struct xa_node *)3UL` — لسه ما مشيناش الـ tree
- `XAS_BOUNDS = (struct xa_node *)1UL` — خرجنا من الـ tree
- `XA_ERROR(errno) = (struct xa_node *)((errno << 2) | 2)` — في error

```
┌──────────────────────────────────────────────────────────┐
│                    xa_state state machine                 │
│                                                          │
│  XAS_RESTART ──► (xas_load/store) ──► valid node        │
│       ▲                                    │             │
│       │                                    ▼             │
│  xas_reset() ◄────────────────── XAS_BOUNDS             │
│                                            │             │
│  XA_ERROR(n) ◄─── (allocation failed) ────┘             │
│       │                                                  │
│       └──► xas_error() returns errno                    │
└──────────────────────────────────────────────────────────┘
```

---

### Diagram: كيف بيتبنى الـ Tree لـ indices متفرقة

لو خزنا entries في indices: 0, 1, 100, 10000

```
xa_head (على 64-bit، XA_CHUNK_SHIFT=6)

بعد store(0):
  xa_head ──► entry[0] مباشرة (direct, بدون xa_node)

بعد store(100):
  xa_head ──► xa_node [shift=6]
               ├── slots[0]  ──► entry[0]   (index 0..63)
               └── slots[1]  ──► entry[100] (index 64..127)

بعد store(10000):
  xa_head ──► xa_node [shift=12]
               ├── slots[0] ──► xa_node [shift=6]
               │                ├── slots[0] ──► entry[0]
               │                └── slots[1] ──► entry[100]
               └── slots[2] ──► xa_node [shift=6]
                                └── slots[41] ──► entry[10000]
                                    (10000 >> 6 = 156, 156 & 63 = 28... etc)
```

الـ tree بتنمو تلقائياً لما بنحتاج range أكبر، وبتتقلص لما بنحذف entries.

---

### الـ Marks System — كيف تتنشر لأعلى

الـ marks هي bitmap في كل `xa_node`، بـ bit واحد لكل slot. لما entry اتعلمت mark، الـ bit بتتسجل في الـ node، وأي node عنده على الأقل واحد من أبنائه marked بيتسجل marked في الـ parent.

```
                xa_node (root) marks[0] = 0b...0100...
                      |
         ─────────────────────────
         |                       |
   xa_node (level1)          xa_node (level1)
   marks[0] = 0b0100         marks[0] = 0b0000
         |
   ──────────────
   |             |
slot[2]=marked  slot[1]=not marked
  (entry)
```

الفايدة: `xa_find_marked()` تقدر تتجاهل كامل الـ subtrees اللي marks[0] فيها = 0. ده بيخليها فعلياً O(k·log n) حيث k هو عدد الـ marked entries مش O(n).

**الـ XA_FREE_MARK (= XA_MARK_0):** بيتستخدم في الـ alloc mode عشان يحدد الـ slots الفاضية. ده بيخلي `xa_alloc()` تلاقي أول slot فاضي بكفاءة باستخدام نفس الـ mark traversal.

---

### الـ Locking Model بالتفصيل

الـ XArray بيدعم ثلاث locking contexts عبر نفس الـ API:

| Function | Lock type |
|---|---|
| `xa_store()` | `spin_lock()` |
| `xa_store_bh()` | `spin_lock_bh()` |
| `xa_store_irq()` | `spin_lock_irq()` |
| `xa_load()` | RCU lock فقط (بدون spinlock) |

الـ `xa_load()` يشتغل RCU-safe لأن:
1. الـ nodes بتتحرر بـ `call_rcu()` (مش immediately).
2. الـ pointers بتتحدث بشكل atomic.
3. الـ reader تحت RCU ما هيشوفش garbage — هيشوف إما الـ old value أو الـ new value.

الـ `__rcu` annotation على `xa_head` وعلى `xa_node::slots[]` وعلى `xa_node::parent` بتوضح ده للـ **sparse** checker (static analysis tool للـ kernel) إن الـ access لازم يكون عبر `rcu_dereference*()` أو تحت lock.

---

### Two-Level API

#### Normal API — للاستخدام العادي

```c
/* Simple, lock/unlock automatic */
void *xa_load(struct xarray *xa, unsigned long index);
void *xa_store(struct xarray *xa, unsigned long index, void *entry, gfp_t gfp);
void *xa_erase(struct xarray *xa, unsigned long index);

/* Atomic compare-and-exchange */
void *xa_cmpxchg(struct xarray *xa, unsigned long index,
                 void *old, void *entry, gfp_t gfp);

/* ID allocation */
int xa_alloc(struct xarray *xa, u32 *id, void *entry,
             struct xa_limit limit, gfp_t gfp);

/* Iteration */
xa_for_each(xa, index, entry) { /* ... */ }
xa_for_each_marked(xa, index, entry, XA_MARK_1) { /* ... */ }
```

#### Advanced API — للـ Page Cache وأمثاله

```c
/* Declare state on stack */
XA_STATE(xas, &mapping->i_pages, page_index);

/* Walk the tree manually */
xas_lock(&xas);
entry = xas_load(&xas);
if (xas_retry(&xas, entry))
    goto retry;

/* Store with pre-allocated node (no allocation under lock) */
xas_store(&xas, new_page);
xas_unlock(&xas);

/* Efficient iteration (O(n) not O(n log n)) */
xas_for_each(&xas, entry, max_index) {
    /* process entry */
    if (need_to_drop_lock) {
        xas_pause(&xas);    /* save state */
        xas_unlock(&xas);
        cond_resched();
        xas_lock(&xas);
    }
}
```

الفرق الجوهري: الـ Normal API بتعمل `xas_load()` من الـ root في كل call — O(log n). الـ Advanced API بتحفظ الـ position في الـ `xa_state` وبتنزل بـ `xas_next()` من الـ current node — O(1) للـ next entry لو في نفس الـ leaf node.

---

### الـ Multi-Index Entries (CONFIG_XARRAY_MULTI)

الـ XArray بيدعم تخزين entry واحدة في range من الـ indices (بـ order = power of 2). الـ page cache بيستخدمه للـ huge pages.

```c
/* Store one entry spanning indices [0, 63] (order=6) */
XA_STATE_ORDER(xas, xa, 0, 6);
xas_store(&xas, huge_page_entry);
```

الـ slots الإضافية بتتملا بـ **sibling entries** — وهي internal entries بتحتوي على الـ offset للـ canonical slot. لما `xa_load(xa, 7)` يتعمل على index داخل الـ range، الـ XArray يشوف الـ sibling entry ويرجع الـ entry من الـ canonical slot تلقائياً.

```
slots في xa_node بعد store بـ order=6 من index 0:
  slots[0] = huge_page  ← canonical slot
  slots[1] = sibling(0) ← internal: "روح لـ slot 0"
  slots[2] = sibling(0)
  ...
  slots[63] = sibling(0)
```

---

### ماذا يمتلك الـ XArray وماذا يفوض؟

#### الـ XArray بيمتلك:

- **الـ indexing logic:** كيف بيتحول الـ integer index لمسار في الـ tree
- **الـ tree growth/shrink:** إضافة وحذف levels تلقائياً
- **الـ locking policy:** spinlock integrated + RCU reads
- **الـ mark propagation:** نشر الـ marks من الـ leaves للـ root
- **الـ entry type encoding:** الـ bit tagging لـ pointer/value/internal
- **الـ error encoding:** تمثيل الـ errnos كـ internal entries
- **الـ node allocation/freeing:** عبر `kmem_cache` مع RCU-safe free

#### الـ XArray بيفوض للـ caller:

- **معنى الـ marks:** الـ XArray مش عارف ليه MARK_0 مهم، ده الـ page cache بيقرره
- **الـ entry lifecycle:** الـ XArray مش بيعمل `kfree` على الـ stored pointers
- **الـ concurrency فوق الـ API:** لو محتاج atomic sequences من operations، الـ caller بيستخدم الـ Advanced API ويدير الـ lock بنفسه
- **الـ `xa_update_node_t` callback:** الـ page cache بيستخدمه يحافظ على LRU ordering
- **تفسير الـ value entries:** الـ XArray بيخزنهم بس مش بيفسرهم

---

### مثال حقيقي — الـ Page Cache

الـ `address_space->i_pages` هو `struct xarray` بيخزن `struct folio *` indexed بـ page index داخل الـ file:

```c
/* In mm/filemap.c - lockless page lookup under RCU */
folio = xa_load(&mapping->i_pages, index);

/* In mm/filemap.c - adding a new page */
XA_STATE(xas, &mapping->i_pages, folio->index);
xas_lock_irq(&xas);
xas_store(&xas, folio);
xas_set_mark(&xas, PAGECACHE_TAG_DIRTY);  /* XA_MARK_0 */
xas_unlock_irq(&xas);

/* Finding dirty pages for writeback */
xas_for_each_marked(&xas, folio, ULONG_MAX, PAGECACHE_TAG_DIRTY) {
    /* write folio to disk */
}
```

الـ `PAGECACHE_TAG_DIRTY` = `XA_MARK_0`، و`PAGECACHE_TAG_WRITEBACK` = `XA_MARK_1`. الـ writeback code بيلاقي الـ dirty pages بكفاءة عالية لأن الـ mark propagation بيسمحله يتجاهل الـ subtrees اللي ما فيهاش dirty pages.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Entry Encoding — تشفير القيم في الـ XArray

الـ XArray بتستخدم الـ **pointer tagging** عشان تميز بين أنواع مختلفة من الـ entries بدون allocations إضافية. الـ 2 bits الأدنى من أي pointer بيحددوا نوعه:

| الـ bits [1:0] | النوع | المعنى |
|---|---|---|
| `00` | **Pointer entry** | pointer عادي لـ user data |
| `x1` | **Value entry** | قيمة integer مخزنة كـ tagged pointer |
| `10` | **Internal entry** | للاستخدام الداخلي فقط (nodes، special markers) |

#### الـ Internal Entries الخاصة

| القيمة الداخلية | الـ Macro | المعنى |
|---|---|---|
| 0 – 62 | `xa_mk_sibling(n)` | **Sibling entry** — يشير لـ slot الرئيسي في multi-index entry |
| 256 | `XA_RETRY_ENTRY` | **Retry** — الـ slot اتغير، أعد القراءة |
| 257 | `XA_ZERO_ENTRY` | **Reserved/Zero** — محجوز لكن فاضي |
| -4094 to -2 | `XA_ERROR(errno)` | **Error** — encoded في xa_node pointer |

#### جدول تشفير الـ Entry

```
Bit layout على 64-bit:
 63                    2   1   0
 ┌──────────────────────┬───┬───┐
 │      Data/Address    │ b1│ b0│
 └──────────────────────┴───┴───┘

 b1=0, b0=0  → User pointer (aligned)
 b1=0, b0=1  → Value: data = entry >> 1
 b1=1, b0=0  → Internal: data = entry >> 2
 b1=1, b0=1  → Tagged pointer (tag في bits [1:0])
```

---

### الـ Flags والـ Enums — Cheatsheet

#### `xa_mark_t` — الـ Marks

| الـ Mark | القيمة | الاستخدام |
|---|---|---|
| `XA_MARK_0` | 0 | **XA_FREE_MARK** — لتتبع الـ free slots في alloc mode |
| `XA_MARK_1` | 1 | للاستخدام العام من قِبَل المستخدم |
| `XA_MARK_2` | 2 | للاستخدام العام من قِبَل المستخدم |
| `XA_PRESENT` | 8 | pseudo-mark داخلي — يمثل "موجود" في الـ iteration |
| `XA_MARK_MAX` | = XA_MARK_2 | أقصى mark مدعوم |
| `XA_FREE_MARK` | = XA_MARK_0 | alias لـ XA_MARK_0 في الـ allocator |

#### `enum xa_lock_type`

| القيمة | المعنى |
|---|---|
| `XA_LOCK_IRQ = 1` | الـ lock بيُأخذ مع disable interrupts |
| `XA_LOCK_BH = 2` | الـ lock بيُأخذ مع disable softirqs |

#### `XA_FLAGS_*` — flags التهيئة

| الـ Flag | القيمة | الوظيفة |
|---|---|---|
| `XA_FLAGS_LOCK_IRQ` | `XA_LOCK_IRQ` | الـ lock بيحتاج IRQ disable |
| `XA_FLAGS_LOCK_BH` | `XA_LOCK_BH` | الـ lock بيحتاج BH disable |
| `XA_FLAGS_TRACK_FREE` | `4U` | يتتبع الـ free slots (لازم لـ alloc) |
| `XA_FLAGS_ZERO_BUSY` | `8U` | index 0 محجوز (alloc يبدأ من 1) |
| `XA_FLAGS_ALLOC_WRAPPED` | `16U` | الـ cyclic alloc عمل wrap |
| `XA_FLAGS_ACCOUNT` | `32U` | يحسب allocations ضمن memory cgroup |
| `XA_FLAGS_ALLOC` | `TRACK_FREE \| MARK(FREE_MARK)` | لـ 0-based alloc |
| `XA_FLAGS_ALLOC1` | `TRACK_FREE \| ZERO_BUSY` | لـ 1-based alloc |
| `XA_FLAGS_MARK(m)` | متحوّل | يفعّل mark معيّن في الـ xa_flags |

#### `XA_CHUNK_*` — ثوابت الشجرة

| الـ Macro | القيمة | المعنى |
|---|---|---|
| `XA_CHUNK_SHIFT` | 6 (أو 4 في BASE_SMALL) | bits per level |
| `XA_CHUNK_SIZE` | 64 | عدد الـ slots في كل node |
| `XA_CHUNK_MASK` | 63 | mask لاستخراج offset داخل node |
| `XA_MAX_MARKS` | 3 | عدد الـ marks المدعومة |
| `XA_MARK_LONGS` | `BITS_TO_LONGS(64)` = 1 | حجم الـ bitmap لكل mark |
| `XA_CHECK_SCHED` | 4096 | عدد iterations قبل reschedule |

---

### الـ Structs الرئيسية

#### 1. `struct xarray` — الـ Anchor

الـ **entry point** للكل. بتعرّفيه static أو بتضميه في struct تانية.

```c
struct xarray {
    spinlock_t  xa_lock;   /* يحمي كل محتوى الـ array */
    gfp_t       xa_flags;  /* XA_FLAGS_* | GFP flags للـ allocation */
    void __rcu *xa_head;   /* NULL لو فاضي، pointer للـ entry أو xa_node */
};
```

| الـ Field | النوع | الوظيفة التفصيلية |
|---|---|---|
| `xa_lock` | `spinlock_t` | الـ lock الوحيد الخاص بالـ array — كل العمليات تأخذه |
| `xa_flags` | `gfp_t` | مزيج من `XA_FLAGS_*` وبيانات GFP allocation |
| `xa_head` | `void __rcu *` | لو `NULL`: array فاضي. لو pointer عادي: entry واحد في index 0. لو `xa_is_node()`: pointer لـ xa_node |

**حالات `xa_head`:**
```
xa_head == NULL          → array فاضي تماماً
xa_head == user_ptr      → entry واحد بس في index 0
xa_head == xa_mk_node()  → شجرة من xa_nodes
```

---

#### 2. `struct xa_node` — عقدة الشجرة الداخلية

الـ **building block** للشجرة. مش للاستخدام المباشر من المستخدم العادي.

```c
struct xa_node {
    unsigned char  shift;         /* bits متبقية per slot في هذا المستوى */
    unsigned char  offset;        /* موضع هذا الـ node في parent->slots[] */
    unsigned char  count;         /* عدد الـ slots غير الـ NULL */
    unsigned char  nr_values;     /* عدد الـ value entries + siblings */
    struct xa_node __rcu *parent; /* NULL في root node */
    struct xarray  *array;        /* pointer للـ xarray المالك */
    union {
        struct list_head private_list; /* للمستخدم المتقدم (page cache) */
        struct rcu_head  rcu_head;     /* للـ RCU free عند الحذف */
    };
    void __rcu *slots[XA_CHUNK_SIZE];          /* 64 slot */
    union {
        unsigned long tags[XA_MAX_MARKS][XA_MARK_LONGS];
        unsigned long marks[XA_MAX_MARKS][XA_MARK_LONGS];
    };
};
```

| الـ Field | الوظيفة |
|---|---|
| `shift` | لو `shift=6` يعني كل slot يغطي 2^6=64 index. الـ root عنده أكبر shift. الـ leaves عندهم `shift=0` |
| `offset` | موقع هذا الـ node في `parent->slots[offset]` |
| `count` | عداد الـ non-NULL slots (يُستخدم لحذف الـ node لو وصل للصفر) |
| `nr_values` | عدد الـ value entries (للـ accounting بالـ page cache) |
| `parent` | null في الـ root. يُستخدم للـ walk up عند الحذف |
| `array` | backreference للـ xarray — يُستخدم في callbacks وvalidation |
| `private_list` | الـ page cache بيستخدمه لتتبع الـ nodes في LRU list |
| `rcu_head` | لما الـ node يُحذف بيُستخدم لـ RCU-safe freeing |
| `slots[]` | الـ 64 slot — ممكن تحتوي user pointers أو internal entries أو xa_nodes |
| `marks[]` | bitmap لكل mark — bit مضبوط لو الـ slot عنده الـ mark ده |

---

#### 3. `struct xa_limit` — نطاق الـ ID في الـ Allocation

```c
struct xa_limit {
    u32 max;  /* أقصى ID (inclusive) */
    u32 min;  /* أدنى ID (inclusive) */
};
```

بسيطة جداً — بتحدد النطاق المسموح للـ `xa_alloc*` functions. التعريفات الجاهزة:

| الـ Macro | النطاق |
|---|---|
| `xa_limit_32b` | [0, UINT_MAX] |
| `xa_limit_31b` | [0, INT_MAX] |
| `xa_limit_16b` | [0, USHRT_MAX] |

---

#### 4. `struct xa_state` — حالة العملية (Advanced API)

الـ **cursor** اللي بيمشي في الشجرة. بيتعمل على الـ stack ومبيتخزنش.

```c
struct xa_state {
    struct xarray  *xa;        /* الـ array اللي بنشتغل عليه */
    unsigned long   xa_index;  /* الـ index الحالي */
    unsigned char   xa_shift;  /* لـ multi-index: shift للـ entry الكبيرة */
    unsigned char   xa_sibs;   /* عدد الـ sibling entries - 1 */
    unsigned char   xa_offset; /* offset الحالي داخل xa_node->slots[] */
    unsigned char   xa_pad;    /* padding لتحسين الـ code generation */
    struct xa_node *xa_node;   /* الـ node الحالي (أو state special) */
    struct xa_node *xa_alloc;  /* node مخصص مسبقاً للـ allocation */
    xa_update_node_t xa_update;/* callback لما node يتحدّث */
    struct list_lru *xa_lru;   /* لـ LRU management (page cache) */
};
```

**الـ special values لـ `xa_node`:**

| القيمة | المعنى |
|---|---|
| `XAS_RESTART` = `(xa_node *)3` | المشي لسه ما بدأش، هيبدأ من الـ root |
| `XAS_BOUNDS` = `(xa_node *)1` | وصلنا لنهاية الـ allocated nodes |
| `XA_ERROR(errno)` | حصل error، الـ state invalid |
| `NULL` | الـ xas في مستوى الـ head (array صغير جداً) |
| valid node ptr | بنشير لـ node حقيقي في الشجرة |

---

### مخطط العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────┐
│                        struct xarray                        │
│  xa_lock: spinlock_t                                        │
│  xa_flags: gfp_t                                            │
│  xa_head: void __rcu * ──────────────────────────────────┐  │
└─────────────────────────────────────────────────────────┬─┘  │
                                                          │    │
              ┌───────────────────────────────────────────┘    │
              │  (xa_is_node() == true)                        │
              ▼                                                 │
┌─────────────────────────────────────┐                        │
│         struct xa_node (root)       │◄── xa_head (tagged)    │
│  shift: 12 (or 6,18,24...)         │                        │
│  offset: 0                         │                        │
│  parent: NULL                      │                        │
│  array: ──────────────────────────►│ struct xarray           │
│  slots[64]: void* ──────┐          │                        │
│  marks[3][1]: bitmap    │          │                        │
└─────────────────────────┼──────────┘                        │
                          │                                    │
          ┌───────────────┼───────────────┐                   │
          │               │               │                   │
          ▼               ▼               ▼                   │
    ┌──────────┐    ┌──────────┐    ┌──────────┐             │
    │ xa_node  │    │ xa_node  │    │ xa_node  │             │
    │ (level1) │    │ (level1) │    │(leaf L1) │             │
    │shift: 6  │    │shift: 6  │    │shift: 0  │             │
    │parent ───┼────┼──────────┼────┼──► root  │             │
    │slots[64] │    │slots[64] │    │slots[64] │             │
    └────┬─────┘    └──────────┘    └────┬─────┘             │
         │                               │                   │
         ▼                               ▼                   │
   ┌───────────┐                  ┌────────────┐             │
   │ xa_node   │                  │ user_ptr   │             │
   │ (leaf)    │                  │ (or value) │             │
   │ shift: 0  │                  └────────────┘             │
   │ slots[64] │                                             │
   │   [0]: ──►│ user_ptr                                    │
   │   [1]: ──►│ xa_mk_value(42)                             │
   │   [2]: ──►│ XA_ZERO_ENTRY                               │
   └───────────┘
```

**الـ xa_state كـ cursor فوق الشجرة:**
```
┌────────────────────────┐
│     struct xa_state    │
│  xa ───────────────────┼──► struct xarray
│  xa_index: 137         │
│  xa_shift: 0           │
│  xa_offset: 9          │
│  xa_node ──────────────┼──► struct xa_node (leaf)
│  xa_alloc: NULL        │
│  xa_update: callback   │
└────────────────────────┘
```

---

### دورة حياة الـ XArray

#### الإنشاء والتهيئة

```
Static Definition                     Runtime Init
──────────────────                    ────────────
DEFINE_XARRAY(name)                   xa_init(xa)
  │                                     │
  └── XARRAY_INIT(name, 0)              └── xa_init_flags(xa, 0)
        │                                     │
        ├── spin_lock_init()                  ├── spin_lock_init()
        ├── xa_flags = 0                      ├── xa_flags = flags
        └── xa_head = NULL                    └── xa_head = NULL
                │
                ▼
         xarray جاهز (empty)
         xa_empty(xa) == true
```

#### تخزين entry أول مرة (index != 0)

```
xa_store(xa, index=100, ptr, GFP_KERNEL)
  │
  ├── xa_lock(xa)
  │
  ├── __xa_store(xa, 100, ptr, GFP)
  │     │
  │     ├── xas_store(&xas, ptr)
  │     │     │
  │     │     ├── xas_load() → يمشي الشجرة
  │     │     │     │
  │     │     │     ├── xa_head == NULL → XAS_RESTART
  │     │     │     └── يحتاج xa_node جديد
  │     │     │
  │     │     ├── xas_expand() → ينشئ xa_node(s) لو الشجرة محتاج تكبر
  │     │     │     │
  │     │     │     └── kmem_cache_alloc(xa_node_cachep)
  │     │     │           → node->shift, parent, array مضبوطين
  │     │     │
  │     │     └── node->slots[offset] = ptr
  │     │           → node->count++
  │     │           → xa_update() callback (لو موجود)
  │     │
  │     └── return old_entry
  │
  └── xa_unlock(xa)
```

#### الحذف والـ Teardown

```
xa_erase(xa, index)                   xa_destroy(xa)
  │                                      │
  ├── xa_lock()                          ├── xas_for_each(&xas, entry, ULONG_MAX)
  ├── __xa_erase()                       │     └── xas_store(&xas, NULL)
  │     └── xas_store(&xas, NULL)        │
  │           │                          ├── يحرر كل الـ xa_nodes
  │           └── لو count==0            │     └── rcu_call() → kfree_rcu
  │               └── xa_delete_node()  │
  │                     └── يمسح        └── xa→xa_head = NULL
  │                         node من
  │                         الشجرة
  │                         (قد يمسح
  │                          parent لو
  │                          هو كمان فاضي)
  └── xa_unlock()
```

---

### مخطط الـ Call Flow

#### القراءة بدون lock (RCU path)

```
xa_load(xa, index)
  │
  ├── rcu_read_lock()
  │
  ├── xas_load(&xas)
  │     │
  │     ├── entry = xa_head(xa)          ← rcu_dereference_check()
  │     │
  │     ├── xa_is_node(entry)?
  │     │   YES:
  │     │   ├── node = xa_to_node(entry)
  │     │   ├── while node->shift > xas->xa_shift:
  │     │   │     ├── offset = (index >> node->shift) & XA_CHUNK_MASK
  │     │   │     ├── entry = xa_entry(xa, node, offset)
  │     │   │     └── node = xa_to_node(entry)  ← walk down
  │     │   └── return slots[final_offset]
  │     │   NO:
  │     │   └── return entry (direct value)
  │     │
  │     └── لو xa_is_retry(entry): xas_reset() وأعد من الأول
  │
  ├── rcu_read_unlock()
  │
  └── return entry (أو NULL)
```

#### الكتابة مع Lock

```
xa_store(xa, index, entry, gfp)
  │
  ├── might_alloc(gfp)
  ├── xa_lock(xa)            ← spin_lock(&xa->xa_lock)
  │
  ├── __xa_store(xa, index, entry, gfp)
  │     │
  │     ├── XA_STATE(xas, xa, index)     ← على الـ stack
  │     │
  │     ├── xas_store(&xas, entry)
  │     │     │
  │     │     ├── xas_expand()            ← ينشئ nodes لو لزم
  │     │     │     └── قد يفعل:
  │     │     │         xa_unlock_irq() → alloc → xa_lock_irq()
  │     │     │
  │     │     └── xas->xa_node->slots[offset] = entry
  │     │
  │     └── xas_destroy(&xas)            ← يحرر xa_alloc لو ما اتاستعملش
  │
  ├── xa_unlock(xa)
  └── return old_entry
```

#### الـ Advanced API — xas_store مع retry

```
xas_for_each(&xas, entry, max)
  │
  ├── entry = xas_find(&xas, max)
  │     │
  │     ├── xas_load() → يمشي من XAS_RESTART
  │     └── لو retry entry → xas_reset() → يعيد المشي
  │
  └── loop:
        ├── process(entry)
        │
        ├── xas_pause(&xas)      ← لو هتنزل الـ lock
        │     └── xas->xa_node = XAS_RESTART
        │         (يضمن إعادة المشي من الصح)
        │
        └── entry = xas_next_entry(&xas, max)
              │
              ├── Fast path: offset++ في نفس الـ node
              └── Slow path: xas_find() لو وصلنا لحافة الـ node
```

---

### استراتيجية الـ Locking

#### القواعد الأساسية

الـ XArray عنده **lock واحد** فقط: `xa->xa_lock` — وهو `spinlock_t`.

| الـ Context | الـ API المناسب | الـ Lock المستخدم |
|---|---|---|
| Process context | `xa_store()`, `xa_load()` | `spin_lock()` / `spin_unlock()` |
| Softirq context | `xa_store_bh()` | `spin_lock_bh()` / `spin_unlock_bh()` |
| Interrupt context | `xa_store_irq()` | `spin_lock_irq()` / `spin_unlock_irq()` |
| مع IRQ flags | `xa_lock_irqsave()` | `spin_lock_irqsave()` |
| للقراءة فقط | `xa_load()` | RCU read lock (مش xa_lock) |

#### اللي محمي بالـ xa_lock

```
xa_lock يحمي:
├── xa->xa_head         (pointer للـ root)
├── xa_node->slots[]    (كل الـ slots في كل node)
├── xa_node->count      (عداد الـ slots)
├── xa_node->nr_values  (عداد الـ values)
├── xa_node->marks[]    (الـ bitmaps)
└── xa->xa_flags        (الـ XA_FLAGS_ALLOC_WRAPPED bit)
```

#### الـ RCU — القراءة بدون Lock

```c
/* قراءة آمنة بدون xa_lock: */
rcu_read_lock();
entry = xa_load(xa, index);     /* يستخدم rcu_dereference داخلياً */
rcu_read_unlock();

/* الكتابة تستخدم rcu_assign_pointer: */
xa_lock(xa);
rcu_assign_pointer(node->slots[offset], new_entry);
xa_unlock(xa);
/* الـ readers القديمين شايفين القيمة القديمة لحد ما يخلصوا */
```

#### التحرير الآمن (RCU-safe free)

```
xa_delete_node(node, update_fn)
  │
  ├── update_fn(node)             ← callback قبل الحذف
  ├── node يُفصل عن الشجرة
  └── call_rcu(&node->rcu_head, free_fn)
        └── free_fn يُنفَّذ بعد ما كل الـ RCU readers الحاليين يخلصوا
```

#### الـ Drop-and-Reacquire Pattern

لما الـ allocation تحتاج memory بينما الـ lock مضبوط:

```
xas_expand() بحاجة لـ xa_node جديد:
  │
  ├── أول حاجة: جرب xa_alloc (node معمول مسبقاً)
  │
  ├── لو محتاج alloc جديدة:
  │   ├── xa_unlock_irq(xa)      ← يرفع الـ lock
  │   ├── node = kmem_cache_alloc(...)
  │   ├── xa_lock_irq(xa)        ← يأخذ الـ lock تاني
  │   └── يراجع إن الشجرة ما اتغيرتش (RESTART لو اتغيرت)
  │
  └── بعد الـ alloc: xas->xa_node = XAS_RESTART لإعادة المشي
```

#### ترتيب الـ Locks (Lock Ordering)

مفيش nested locks في الـ XArray نفسه — الـ xa_lock هو الوحيد. لكن لو الـ caller بيستخدم locks تانية:

```
القاعدة: xa_lock دايما يجي بعد الـ locks الخارجية الأكبر

مثال في page cache:
  mapping->invalidate_lock (rwsem)
    └── xa_lock (spinlock)

مثال في idr/xarray allocator:
  أي lock خارجي (mutex/spinlock)
    └── xa->xa_lock
```

#### الـ Nested Locks للـ Lockdep

```c
/* لو عندك xarrays متداخلة: */
xa_lock_nested(xa, SINGLE_DEPTH_NESTING);

/* lockdep subclasses: */
xa_lock_irq_nested(xa, subclass);
xa_lock_bh_nested(xa, subclass);
```

---

### مخطط lifecycle الـ xa_node

```
         kmem_cache_alloc()
                │
                ▼
         ┌────────────┐
         │  ALLOCATED  │  shift, offset, parent, array
         │  (fresh)    │  count=0, nr_values=0
         └─────┬───────┘
               │ xas_expand() / xas_store()
               ▼
         ┌────────────┐
         │    LIVE     │  slots[] مملوءة، marks[] مضبوطة
         │  (in tree)  │  parent→slots[offset] = xa_mk_node(this)
         └─────┬───────┘
               │ count → 0 (كل الـ slots اتمسحت)
               ▼
         ┌────────────┐
         │   EMPTY     │  xa_delete_node() بيفصله
         │ (condemned) │
         └─────┬───────┘
               │ call_rcu()
               ▼
         ┌────────────┐
         │    FREE     │  kfree() بعد RCU grace period
         └────────────┘
```

---

### ملاحظات عملية مهمة

**الـ `xa_state` دايماً على الـ stack** — ما يتخزنش في heap، لأنه بيتبع حالة عملية واحدة بس.

**الـ `XA_STATE_ORDER()`** بيُستخدم لتخزين entry يغطي أكتر من index واحد (huge pages في page cache). الـ `xa_shift` و`xa_sibs` بيتحسبوا تلقائياً.

**الـ `xas_pause()`** ضروري لو هتنزل الـ xa_lock في الـ middle of iteration — من غيره الـ xas بيفضل invalid بعد ما الشجرة تتغير.

**الـ `xa_update_node_t` callback** بيتنادى وانت شايل الـ xa_lock — ما تعملش حاجة تاخد lock تاني أو تحرر memory.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Group 1: Entry Type Encoding / Decoding

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `xa_mk_value` | `void *xa_mk_value(unsigned long v)` | يحوّل integer إلى XArray value entry |
| `xa_to_value` | `unsigned long xa_to_value(const void *entry)` | يستخرج الـ integer من value entry |
| `xa_is_value` | `bool xa_is_value(const void *entry)` | يتحقق إن كان الـ entry value أم pointer |
| `xa_tag_pointer` | `void *xa_tag_pointer(void *p, unsigned long tag)` | يضيف tag على pointer |
| `xa_untag_pointer` | `void *xa_untag_pointer(void *entry)` | يزيل الـ tag من tagged pointer |
| `xa_pointer_tag` | `unsigned int xa_pointer_tag(void *entry)` | يقرأ الـ tag المخزّن في entry |
| `xa_mk_internal` | `void *xa_mk_internal(unsigned long v)` | يصنع internal entry (private) |
| `xa_to_internal` | `unsigned long xa_to_internal(const void *entry)` | يستخرج القيمة من internal entry |
| `xa_is_internal` | `bool xa_is_internal(const void *entry)` | يتحقق إن كان الـ entry internal |
| `xa_is_zero` | `bool xa_is_zero(const void *entry)` | يتحقق من zero/reserved entry |
| `xa_is_err` | `bool xa_is_err(const void *entry)` | يتحقق إن كان الـ entry يحمل errno |
| `xa_err` | `int xa_err(void *entry)` | يستخرج الـ errno من entry |

#### Group 2: Initialization & Lifecycle

| Function/Macro | الغرض |
|---|---|
| `xa_init_flags` | تهيئة XArray بـ flags محددة |
| `xa_init` | تهيئة XArray بدون flags |
| `xa_empty` | فحص إن كان الـ XArray فارغاً |
| `xa_destroy` | تحرير كل الـ nodes الداخلية |
| `DEFINE_XARRAY` | تعريف XArray ثابت في compile time |
| `DEFINE_XARRAY_FLAGS` | تعريف XArray بـ flags في compile time |
| `DEFINE_XARRAY_ALLOC` | XArray للـ ID allocation يبدأ من 0 |
| `DEFINE_XARRAY_ALLOC1` | XArray للـ ID allocation يبدأ من 1 |

#### Group 3: Normal API — Store / Load / Erase

| Function | الغرض |
|---|---|
| `xa_load` | تحميل entry بـ RCU |
| `xa_store` | تخزين entry (يمسك الـ lock داخلياً) |
| `xa_store_bh` | `xa_store` + softirq-disable |
| `xa_store_irq` | `xa_store` + irq-disable |
| `xa_erase` | مسح entry |
| `xa_erase_bh` | مسح entry + softirq-disable |
| `xa_erase_irq` | مسح entry + irq-disable |
| `xa_store_range` | تخزين entry في range من indices (multi-index) |

#### Group 4: Atomic / Conditional Operations

| Function | الغرض |
|---|---|
| `xa_cmpxchg` | compare-and-exchange على entry |
| `xa_cmpxchg_bh` | cmpxchg + softirq-disable |
| `xa_cmpxchg_irq` | cmpxchg + irq-disable |
| `xa_insert` | تخزين فقط لو الـ slot فاضي |
| `xa_insert_bh` | insert + softirq-disable |
| `xa_insert_irq` | insert + irq-disable |
| `xa_reserve` | حجز slot بدون تخزين قيمة حقيقية |
| `xa_reserve_bh` | reserve + softirq-disable |
| `xa_reserve_irq` | reserve + irq-disable |
| `xa_release` | تحرير reservation لو لم تُستخدم |

#### Group 5: ID Allocation

| Function | الغرض |
|---|---|
| `xa_alloc` | تخصيص ID في range وتخزين entry |
| `xa_alloc_bh` | alloc + softirq-disable |
| `xa_alloc_irq` | alloc + irq-disable |
| `xa_alloc_cyclic` | alloc بـ wraparound (دوراني) |
| `xa_alloc_cyclic_bh` | cyclic + softirq-disable |
| `xa_alloc_cyclic_irq` | cyclic + irq-disable |

#### Group 6: Marks

| Function | الغرض |
|---|---|
| `xa_get_mark` | قراءة mark على index معين |
| `xa_set_mark` | وضع mark على index |
| `xa_clear_mark` | مسح mark من index |
| `xa_marked` | فحص إن كان أي entry في الـ XArray عنده mark |

#### Group 7: Search & Iteration (Normal API)

| Function/Macro | الغرض |
|---|---|
| `xa_find` | البحث عن أول entry عند أو بعد index |
| `xa_find_after` | البحث عن entry بعد index (exclusive) |
| `xa_extract` | نسخ batch من entries إلى array |
| `xa_for_each` | iterate على كل entries من 0 |
| `xa_for_each_start` | iterate من start معيّن |
| `xa_for_each_range` | iterate في range محددة |
| `xa_for_each_marked` | iterate على entries بـ mark معيّن |

#### Group 8: Advanced API — xa_state Operations

| Function | الغرض |
|---|---|
| `xas_load` | تحميل entry عبر cursor |
| `xas_store` | تخزين entry عبر cursor |
| `xas_find` | البحث عن entry تالي |
| `xas_find_marked` | البحث عن marked entry تالي |
| `xas_find_conflict` | البحث عن entries تتعارض مع range |
| `xas_get_mark` | قراءة mark عبر cursor |
| `xas_set_mark` | وضع mark عبر cursor |
| `xas_clear_mark` | مسح mark عبر cursor |
| `xas_init_marks` | تصفير كل الـ marks للـ entry الحالي |
| `xas_nomem` | محاولة تخصيص node مسبقاً لتجنب failure |
| `xas_destroy` | تحرير pre-allocated node |
| `xas_pause` | إيقاف مؤقت آمن للـ iterator لـ reschedule |
| `xas_create_range` | إنشاء nodes لتغطية multi-index range |
| `xas_split` | تقسيم multi-index entry |
| `xas_split_alloc` | تخصيص nodes لـ split مسبقاً |
| `xas_reload` | إعادة قراءة entry في الموقع الحالي |
| `xas_set` | تغيير الـ index في cursor |
| `xas_advance` | القفز إلى sibling entry أخير |
| `xas_set_order` | ضبط cursor لـ multi-index entry |
| `xas_set_update` | تسجيل callback لتحديثات الـ nodes |
| `xas_next` | الانتقال إلى index التالي |
| `xas_prev` | الانتقال إلى index السابق |
| `xas_next_entry` | fast path للـ iteration على present entries |
| `xas_next_marked` | fast path للـ iteration على marked entries |
| `xas_for_each` | macro للـ iteration عبر cursor |
| `xas_for_each_marked` | macro للـ iteration على marked entries عبر cursor |
| `xas_for_each_conflict` | macro للـ iteration على conflicting entries |

#### Group 9: Lock Helpers

| Macro | السياق |
|---|---|
| `xa_lock / xa_unlock` | process context |
| `xa_lock_bh / xa_unlock_bh` | softirq context |
| `xa_lock_irq / xa_unlock_irq` | hardirq context |
| `xa_lock_irqsave / xa_unlock_irqrestore` | unknown IRQ state |
| `xa_trylock` | non-blocking |
| `xas_lock / xas_unlock / ...` | نفس الـ xa_lock لكن عبر xa_state |

---

### Group 1: Entry Type Encoding/Decoding

الـ XArray يستعمل الـ bits السفلى من الـ pointer لتمييز نوع الـ entry. البنية:

```
bit[1:0] = 00 → pointer entry (user pointer عادي)
bit[1:0] = x1 → value entry أو tagged pointer
bit[1:0] = 10 → internal entry (kernel-private)
```

ده الـ trick الأساسي اللي بيخلي الـ XArray يخزن integers وpointers في نفس الـ slot بدون overhead.

---

#### `xa_mk_value`

```c
static inline void *xa_mk_value(unsigned long v)
{
    WARN_ON((long)v < 0);
    return (void *)((v << 1) | 1);
}
```

بيحوّل integer إلى XArray entry عن طريق shift يسار بمقدار 1 وset للـ bit 0. الـ `WARN_ON` بتحمي من القيم السالبة لأن الـ MSB بعد الـ shift هيتداخل مع internal entry encoding.

- **@v**: الـ integer المراد تخزينه — لازم يكون موجب، الـ MSB محجوز.
- **Return**: `void *` صالح للتخزين في XArray، مش pointer حقيقي.
- **Key detail**: الـ `BITS_PER_XA_VALUE = BITS_PER_LONG - 1`، يعني على 64-bit تقدر تخزن 63-bit value.
- **Caller**: أي كود يريد تخزين integer مباشرة في XArray بدون heap allocation.

---

#### `xa_to_value`

```c
static inline unsigned long xa_to_value(const void *entry)
{
    return (unsigned long)entry >> 1;
}
```

عكس `xa_mk_value` — بيشيل الـ tag bit عن طريق right shift. لا يتحقق من صحة الـ entry، المسؤولية على الـ caller.

- **@entry**: entry مجلوب من XArray، لازم يكون value entry.
- **Return**: الـ integer الأصلي.
- **Caller**: بعد `xa_is_value()` يرجع true.

---

#### `xa_is_value`

```c
static inline bool xa_is_value(const void *entry)
{
    return (unsigned long)entry & 1;
}
```

بيفحص الـ bit 0. أسرع check ممكن.

- **Return**: `true` لو value entry أو tagged pointer.

---

#### `xa_tag_pointer`

```c
static inline void *xa_tag_pointer(void *p, unsigned long tag)
{
    return (void *)((unsigned long)p | tag);
}
```

بيضيف tag (0، 1، أو 3) على pointer. مختلف عن `xa_mark_t` — الـ tags دي مش بتتنشر لفوق في الـ tree ومش قابلة للـ search.

- **@p**: pointer عادي، لازم يكون aligned بحيث الـ bits السفلى صفر.
- **@tag**: 0، 1، أو 3 (مش 2 لأنه reserved للـ internal entries).
- **Use case**: page cache بتستخدمه لتمييز shadow entries.

---

#### `xa_is_err` و `xa_err`

```c
static inline bool xa_is_err(const void *entry)
{
    return unlikely(xa_is_internal(entry) &&
            entry >= xa_mk_internal(-MAX_ERRNO));
}

static inline int xa_err(void *entry)
{
    if (xa_is_err(entry))
        return (long)entry >> 2;
    return 0;
}
```

الـ XArray بيمثّل errors كـ internal entries في الـ negative space. الـ `xa_err` بيعمل arithmetic right shift للـ sign extension لاسترجاع الـ errno.

- **Key detail**: الـ errors دي مش بتتخزن في الـ array أبداً، بس بتُرجع من الـ API functions.
- **Error range**: من -2 لحد -4094 (نفس حدود `MAX_ERRNO`).

---

### Group 2: Initialization & Lifecycle

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

نقطة دخول أساسية لتهيئة XArray. بتستخدم `spin_lock_init` بدل static initializer — مطلوبة للـ XArrays المخصصة dynamically.

- **@xa**: الـ struct المراد تهيئته.
- **@flags**: `XA_FLAGS_LOCK_IRQ`، `XA_FLAGS_LOCK_BH`، `XA_FLAGS_ALLOC`، إلخ.
- **Context**: Any context لأنه مجرد initialization.
- **Caller**: `xa_init()` وأي كود بيحتاج non-default flags.

**الفرق بين الـ flags المهمة:**

| Flag | المعنى |
|---|---|
| `XA_FLAGS_LOCK_IRQ` | الـ locking context هو hardirq |
| `XA_FLAGS_LOCK_BH` | الـ locking context هو softirq |
| `XA_FLAGS_TRACK_FREE` | تتبع الـ free entries (مطلوب للـ alloc) |
| `XA_FLAGS_ALLOC` | يفعّل ID allocation من 0 |
| `XA_FLAGS_ALLOC1` | يفعّل ID allocation من 1 |
| `XA_FLAGS_ACCOUNT` | يحسب الـ memory في memory cgroup |

---

#### `xa_destroy`

```c
void xa_destroy(struct xarray *);
```

بتحرر كل الـ `xa_node` structures الداخلية للـ tree لكن مش بتحرر الـ entries نفسها (الـ user data). الـ caller مسؤول عن تحرير الـ data قبل استدعاء `xa_destroy`.

- **Context**: Process context — بتمسك وتحرر الـ xa_lock.
- **Side effect**: الـ XArray يرجع empty بعدها (`xa_head = NULL`).
- **Caller**: shutdown paths، device removal، module unload.

---

### Group 3: Normal API — Store / Load / Erase

الـ Normal API بتمسك الـ lock داخلياً وبترجع الـ old entry أو errno. مناسبة لمعظم use cases.

---

#### `xa_load`

```c
void *xa_load(struct xarray *, unsigned long index);
```

بتجيب الـ entry عند index معيّن. بتشتغل بـ RCU read lock فقط بدون spinlock — ده بيخليها lock-free في الـ common case.

- **@index**: الـ index المطلوب.
- **Return**: الـ entry (pointer أو value) أو `NULL` لو الـ slot فاضي أو محجوز.
- **Context**: Any context — RCU safe، no sleeping needed.
- **Key detail**: بترجع `NULL` للـ zero entries (reserved slots) — لو محتاج تشوف الـ zero entries استخدم Advanced API.
- **Concurrency**: آمنة مع concurrent stores بفضل RCU.

---

#### `xa_store`

```c
void *xa_store(struct xarray *, unsigned long index, void *entry, gfp_t);
```

بتخزن entry في index معيّن. لو الـ entry هو `NULL` فده يعني erase. بتمسك الـ xa_lock وقد تخصص memory.

- **@entry**: pointer عادي أو value entry أو `NULL` للمسح. مش مسموح بـ internal entries.
- **@gfp**: GFP flags للـ memory allocation — لو `GFP_NOWAIT` فالـ function ممكن ترجع error.
- **Return**: الـ old entry اللي كان في الـ slot، أو `xa_err()` لو فشل الـ allocation.
- **Locking**: بتمسك `xa_lock` عادي. استخدم `xa_store_bh/irq` لو لازم.
- **Caller**: أي write path لا يحتاج atomicity خاصة.

```c
/* مثال: تخزين device في xarray */
void *old = xa_store(&dev_table, dev->id, dev, GFP_KERNEL);
if (xa_is_err(old)) {
    ret = xa_err(old);
    /* handle error */
}
```

---

#### `xa_erase`

```c
void *xa_erase(struct xarray *, unsigned long index);
```

بتمسح الـ entry عند index وبترجع الـ old entry. مش بتخصص memory — آمنة مع `GFP_NOWAIT` semantics.

- **Return**: الـ old entry أو `NULL`.
- **Side effect**: لو الـ index كان جزء من multi-index entry، كل الـ indices المرتبطة بتتمسح.

---

#### `xa_store_range`

```c
void *xa_store_range(struct xarray *, unsigned long first,
                     unsigned long last, void *entry, gfp_t);
```

بتخزن نفس الـ entry في range من indices باستخدام multi-index mechanism. مطلوب `CONFIG_XARRAY_MULTI`.

- **@first / @last**: حدود الـ range، inclusive.
- **Use case**: page cache large pages (THP) حيث order-N page بتشغل `2^N` slots.

---

### Group 4: Atomic / Conditional Operations

---

#### `xa_cmpxchg`

```c
static inline void *xa_cmpxchg(struct xarray *xa, unsigned long index,
            void *old, void *entry, gfp_t gfp)
{
    void *curr;
    might_alloc(gfp);
    xa_lock(xa);
    curr = __xa_cmpxchg(xa, index, old, entry, gfp);
    xa_unlock(xa);
    return curr;
}
```

بتستبدل الـ entry فقط لو الـ current value يساوي `old`. الـ comparison والـ swap بيحصلوا تحت الـ lock.

- **@old**: القيمة المتوقعة في الـ slot.
- **@entry**: القيمة الجديدة.
- **Return**: القيمة الموجودة في الوقت اللي حصل فيه الـ check. لو return == old فالـ swap نجح.
- **Pattern**: `if (xa_cmpxchg(xa, i, NULL, entry, gfp) == NULL)` → نجح الـ insert.

---

#### `xa_insert`

```c
static inline int __must_check xa_insert(struct xarray *xa,
        unsigned long index, void *entry, gfp_t gfp)
```

بتخزن فقط لو الـ slot فاضي تماماً. لو فيه zero/reserved entry بترجع `-EBUSY` حتى لو `xa_load` بيرجع `NULL`.

- **Return**: 0 نجاح، `-EBUSY` فيه شيء موجود، `-ENOMEM` فشل الـ allocation.
- **Use case**: IDR-like patterns حيث لا يجب استبدال موجود.

---

#### `xa_reserve` / `xa_release`

```c
static inline int xa_reserve(struct xarray *xa, unsigned long index, gfp_t gfp)
{
    return xa_err(xa_cmpxchg(xa, index, NULL, XA_ZERO_ENTRY, gfp));
}

static inline void xa_release(struct xarray *xa, unsigned long index)
{
    xa_cmpxchg(xa, index, XA_ZERO_ENTRY, NULL, 0);
}
```

`xa_reserve` بتحجز slot بتخزين `XA_ZERO_ENTRY` فيه. الـ `xa_load` هيرجع `NULL` للـ reserved slot لكن `xa_insert` هيفشل بـ `-EBUSY`.

**Flow نموذجي:**
```
xa_reserve(xa, id, GFP_KERNEL)   ← حجز slot
... allocate & init object ...
xa_store(xa, id, obj, GFP_KERNEL) ← تخزين فعلي
/* أو في حالة فشل: */
xa_release(xa, id)                ← تحرير الحجز
```

---

### Group 5: ID Allocation

الـ XArray بيدعم IDR-like allocation. لازم الـ XArray يكون initialized بـ `XA_FLAGS_ALLOC`.

---

#### `xa_alloc`

```c
static inline int xa_alloc(struct xarray *xa, u32 *id,
        void *entry, struct xa_limit limit, gfp_t gfp)
```

بتدور على أول slot فاضي في `[limit.min, limit.max]` وبتخزن الـ entry فيه وبترجع الـ ID في `*id`.

- **@id**: output parameter — ID المخصص بيتكتب هنا.
- **@limit**: `XA_LIMIT(min, max)` — pre-defined: `xa_limit_32b`، `xa_limit_31b`، `xa_limit_16b`.
- **Return**: 0 نجاح، `-ENOMEM`، `-EBUSY` لو الـ range ممتلئ.
- **Atomicity**: الـ concurrent lookup مش هيشوف slot بـ ID غير initialized.

---

#### `xa_alloc_cyclic`

```c
static inline int xa_alloc_cyclic(struct xarray *xa, u32 *id, void *entry,
        struct xa_limit limit, u32 *next, gfp_t gfp)
```

زي `xa_alloc` لكن الـ search بيبدأ من `*next` وبيلف (wrap) لو وصل للنهاية. بتحدّث `*next` بعد كل allocation.

- **@next**: pointer للـ counter الدوراني — بيتحدّث تلقائياً.
- **Return**: 0 نجاح، error < 0 لو فشل. الـ `__xa_alloc_cyclic` بترجع 1 لو حصل wraparound.
- **Use case**: network connection IDs، file descriptors، أي resource يحتاج sequential IDs.

**الفرق الجوهري مع `xa_alloc`:**

```
xa_alloc:         يبدأ دائماً من limit.min
xa_alloc_cyclic:  يبدأ من *next، يلف عند limit.max → limit.min
```

---

### Group 6: Marks

الـ XArray يدعم 3 marks مستقلة (`XA_MARK_0`، `XA_MARK_1`، `XA_MARK_2`). كل mark بتتنشر (propagate) لفوق في الـ tree — يعني لو أي entry في subtree عنده mark، كل الـ parent nodes بيعرفوا بذلك. ده بيخلي البحث O(log n) بدل O(n).

---

#### `xa_set_mark` / `xa_clear_mark` / `xa_get_mark`

```c
void xa_set_mark(struct xarray *, unsigned long index, xa_mark_t);
void xa_clear_mark(struct xarray *, unsigned long index, xa_mark_t);
bool xa_get_mark(struct xarray *, unsigned long index, xa_mark_t);
```

بتمسك الـ xa_lock داخلياً. الـ mark بتتخزن في بيانات الـ `xa_node` في كل مستوى من الـ tree.

- **`XA_FREE_MARK` (= `XA_MARK_0`)**: مخصصة للـ IDR/alloc functionality.
- **`XA_PRESENT` (= 8)**: mark خاصة بتعني "موجود" — بتستخدمها `xa_find/xa_for_each`.
- **Locking**: بتمسك `xa_lock`. الـ `__xa_set_mark/__xa_clear_mark` بتشتغل وهو ماسك الـ lock.

---

#### `xa_marked`

```c
static inline bool xa_marked(const struct xarray *xa, xa_mark_t mark)
{
    return xa->xa_flags & XA_FLAGS_MARK(mark);
}
```

بتفحص الـ xa_flags مباشرة — O(1) check إن كان أي entry في الـ array عنده mark. الـ tree propagation بتضمن إن الـ flags دايماً consistent.

---

### Group 7: Search & Iteration (Normal API)

---

#### `xa_find`

```c
void *xa_find(struct xarray *xa, unsigned long *index,
              unsigned long max, xa_mark_t);
```

بتدور على أول entry عند أو بعد `*index` حتى `max` بالـ mark المطلوب. بتحدّث `*index` بالـ index الفعلي اللي لقت فيه الـ entry.

- **@index**: input/output — index البداية، بيتحدث بالـ index الفعلي.
- **@max**: أقصى index inclusive.
- **@mark**: `XA_PRESENT` للـ present entries، أو أي mark من 0 لـ 2.
- **Return**: الـ entry أو `NULL` لو مفيش.
- **Context**: RCU lock — بتمسكه وتحرره داخلياً.

---

#### `xa_find_after`

```c
void *xa_find_after(struct xarray *xa, unsigned long *index,
                    unsigned long max, xa_mark_t);
```

مثل `xa_find` لكن exclusive — بتبدأ البحث من `*index + 1`. بتستخدمها iteration macros للخطوة التالية.

---

#### `xa_extract`

```c
unsigned int xa_extract(struct xarray *, void **dst, unsigned long start,
                        unsigned long max, unsigned int n, xa_mark_t);
```

بتنسخ batch من entries إلى array خارجي. أسرع من iteration loop بسبب تقليل overhead الـ RCU lock acquire/release.

- **@dst**: output array.
- **@n**: عدد entries مطلوب.
- **Return**: عدد الـ entries اللي تم نسخها فعلاً.
- **Use case**: bulk operations، snapshot of IDs.

---

#### Iteration Macros

```c
/* iterate على كل entries */
xa_for_each(xa, index, entry)

/* iterate من start */
xa_for_each_start(xa, index, entry, start)

/* iterate في range */
xa_for_each_range(xa, index, entry, start, last)

/* iterate على entries بـ mark */
xa_for_each_marked(xa, index, entry, filter)
```

كلهم بيستخدموا `xa_find` و`xa_find_after` داخلياً. الـ complexity هي O(n·log(n)) بعكس `xas_for_each` اللي هي O(n).

**تحذير**: لو وصلوا retry entry بيـ spin — استخدم `xas_for_each` لو محتاج ترى retry entries.

---

### Group 8: Advanced API — xa_state

#### الـ `struct xa_state` وفكرة الـ Cursor

```c
struct xa_state {
    struct xarray *xa;        /* الـ array المستهدف */
    unsigned long xa_index;   /* الـ index الحالي */
    unsigned char xa_shift;   /* shift لـ multi-index */
    unsigned char xa_sibs;    /* عدد siblings لـ multi-index */
    unsigned char xa_offset;  /* offset داخل الـ node الحالي */
    unsigned char xa_pad;
    struct xa_node *xa_node;  /* الـ node الحالي أو special value */
    struct xa_node *xa_alloc; /* pre-allocated node */
    xa_update_node_t xa_update; /* callback */
    struct list_lru *xa_lru;  /* لـ LRU management */
};
```

الـ `xa_state` هو cursor يتذكر مكانه في الـ tree. بيتعلن على الـ stack وبيتمرر بين الـ functions. بيوفر:
- تجنب traversal من الـ root في كل عملية
- pre-allocation إمكانية لتجنب memory allocation تحت الـ lock
- callback mechanism لتحديث data structures خارجية

**Special values للـ `xa_node`:**

```
XAS_RESTART  = (struct xa_node *)3UL  → لازم يعيد المشي من الـ root
XAS_BOUNDS   = (struct xa_node *)1UL  → خرج من حدود الـ array
XA_ERROR(e)  = encoded errno           → حدث خطأ
NULL                                   → الـ array عنده entry واحد في index 0
```

---

#### إنشاء الـ `xa_state`

```c
/* Standard cursor */
XA_STATE(xas, &my_array, start_index);

/* Multi-index cursor لـ order-N entries */
XA_STATE_ORDER(xas, &my_array, index, order);
```

---

#### `xas_load`

```c
void *xas_load(struct xa_state *);
```

بتمشي الـ tree من الـ root (أو تكمل من مكانها لو الـ state صالح) وبترجع الـ entry. بتتعامل مع retry entries داخلياً.

- **Context**: RCU أو xa_lock — اعتماداً على استخدامك.
- **Key detail**: لو الـ `xas->xa_node == XAS_RESTART` بتعمل full tree walk، وإلا بتقرأ مباشرة.

**Pseudocode:**
```
if xas_invalid(xas):
    if xas_error: return NULL
    walk tree from root to xa_index
    update xa_node, xa_offset

return slot[xa_offset]  /* might be NULL, internal, or user entry */
```

---

#### `xas_store`

```c
void *xas_store(struct xa_state *, void *entry);
```

بتخزن entry في الـ position الحالية للـ cursor. أقوى من `xa_store` لأنها:
- تعرف مكانها في الـ tree (مش بتمشي من الـ root)
- بتدعم multi-index entries
- بتستدعي الـ `xa_update` callback لو موجود

- **Context**: لازم يمسك `xa_lock`.
- **Return**: الـ old entry.
- **Side effect**: بتحدث الـ marks propagation وبتحذف nodes لو أصبحت فاضية.

---

#### `xas_find`

```c
void *xas_find(struct xa_state *, unsigned long max);
```

بتحرّك الـ cursor للـ entry التالي (present) حتى `max`. بتستخدم الـ current position كنقطة انطلاق.

- **Context**: RCU أو xa_lock.
- **Return**: الـ entry أو `NULL`.
- **Key detail**: بتتجاهل الـ zero entries — بترجعها كـ `NULL`.

---

#### `xas_find_marked`

```c
void *xas_find_marked(struct xa_state *, unsigned long max, xa_mark_t);
```

مثل `xas_find` لكن بتبحث عن marked entries فقط. بتستغل الـ mark propagation في الـ tree للقفز على subtrees بدون mark — ده بيعطيها O(n) حيث n هو عدد الـ marked entries.

---

#### `xas_find_conflict`

```c
void *xas_find_conflict(struct xa_state *);
```

بتبحث عن entries تتداخل مع الـ range المحددة في الـ xas (multi-index range). بتستخدمها `xa_store_range` قبل ما تكتب.

---

#### `xas_nomem`

```c
bool xas_nomem(struct xa_state *, gfp_t);
```

لو فشلت عملية بسبب `ENOMEM`، بتحاول تخصص node مسبقاً خارج الـ lock وبتحط النتيجة في `xas->xa_alloc`. بترجع `true` لو خصصت بنجاح وبتصفّي الـ error state.

**Pattern كلاسيكي:**
```c
do {
    xas_lock(&xas);
    entry = xas_store(&xas, value);
    xas_unlock(&xas);
} while (xas_nomem(&xas, GFP_KERNEL));
```

---

#### `xas_pause`

```c
void xas_pause(struct xa_state *);
```

بتحفظ الـ position الحالية للـ iterator بطريقة تسمح بإطلاق الـ lock والـ reschedule ثم الاستمرار. بتضبط الـ `xa_node` على `XAS_RESTART` لكن بتزيد الـ index بـ 1 لتجنب إعادة معالجة نفس الـ entry.

- **Caller**: loops طويلة تحت الـ lock كل `XA_CHECK_SCHED = 4096` iteration.

---

#### `xas_create_range`

```c
void xas_create_range(struct xa_state *);
```

بتخصص وتنشئ كل الـ nodes الداخلية الضرورية لتغطية multi-index range محددة بالـ `xas_set_order`. لازم تتنادى قبل `xas_store` لضمان عدم فشل الـ allocation تحت الـ lock.

---

#### `xas_split` / `xas_split_alloc`

```c
void xas_split(struct xa_state *, void *entry, unsigned int order);
void xas_split_alloc(struct xa_state *, void *entry, unsigned int order, gfp_t);
```

`xas_split_alloc` بتخصص الـ nodes الضرورية لعملية الـ split خارج الـ lock. `xas_split` بتنفذ الـ split الفعلي تحت الـ lock — بتحوّل multi-index entry إلى individual entries.

- **Use case**: page cache عند split الـ THP (Transparent Huge Pages).

---

#### `xas_reload`

```c
static inline void *xas_reload(struct xa_state *xas)
```

بتقرأ الـ entry الحالي مجدداً بدون re-walk — مفيدة لـ lockless verification pattern.

**Pattern في page cache:**
```c
/* 1. load بدون lock */
page = xas_load(&xas);
/* 2. lock الـ page */
lock_page(page);
/* 3. تحقق إن الـ page لم تتغير */
if (xas_reload(&xas) != page)
    goto retry;
```

---

#### `xas_set` / `xas_advance` / `xas_set_order`

```c
static inline void xas_set(struct xa_state *xas, unsigned long index)
{
    xas->xa_index = index;
    xas->xa_node = XAS_RESTART;  /* يجبر full re-walk */
}

static inline void xas_advance(struct xa_state *xas, unsigned long index)
{
    /* يتحرك داخل نفس الـ node بدون re-walk */
    xas->xa_index = index;
    xas->xa_offset = (index >> shift) & XA_CHUNK_MASK;
}

static inline void xas_set_order(struct xa_state *xas,
                                  unsigned long index, unsigned int order)
```

- `xas_set`: jump لـ index جديد تماماً — بيجبر re-walk من الـ root.
- `xas_advance`: حركة سريعة داخل نفس الـ node (لـ sibling entries).
- `xas_set_order`: ضبط الـ cursor لـ multi-index entry بحجم `2^order`.

---

#### `xas_next` / `xas_prev`

```c
static inline void *xas_next(struct xa_state *xas)
{
    struct xa_node *node = xas->xa_node;
    if (unlikely(xas_not_node(node) || node->shift ||
                 xas->xa_offset == XA_CHUNK_MASK))
        return __xas_next(xas);
    xas->xa_index++;
    xas->xa_offset++;
    return xa_entry(xas->xa, node, xas->xa_offset);
}
```

Fast path للانتقال index ±1. الـ `unlikely` بتضمن إن الـ common case (نفس الـ node، لا يوجد shift) يكون inline code فقط بدون function call.

- **Return**: الـ entry في الـ index الجديد — ممكن يكون `NULL` أو internal entry.
- **Wraparound**: عند ULONG_MAX ترجع لـ 0 وعكسه.

---

#### `xas_next_entry` / `xas_next_marked`

```c
static inline void *xas_next_entry(struct xa_state *xas, unsigned long max)
static inline void *xas_next_marked(struct xa_state *xas,
                                     unsigned long max, xa_mark_t mark)
```

Fast-path iteration functions. بتتفادى الـ function call overhead في الـ common case (leaf-level node بدون internal entries).

- `xas_next_entry`: fast path لـ `xas_find`.
- `xas_next_marked`: fast path لـ `xas_find_marked` مع استخدام `xas_find_chunk` للبحث في الـ mark bitmap.

**`xas_find_chunk` (private):**
```c
static inline unsigned int xas_find_chunk(struct xa_state *xas, bool advance,
                                           xa_mark_t mark)
```
بتستخدم `__ffs` (find first set bit) أو `find_next_bit` على الـ marks bitmap للـ node. ده بيعطيها performance ممتازة على modern CPUs.

---

#### Iteration Macros (Advanced API)

```c
/* iterate على present entries */
xas_for_each(xas, entry, max)

/* iterate على marked entries */
xas_for_each_marked(xas, entry, max, mark)

/* iterate على conflicting entries */
xas_for_each_conflict(xas, entry)
```

الـ Advanced API macros بتتطلب إدارة الـ lock يدوياً. أسرع من Normal API macros بسبب O(n) traversal.

**مثال كامل:**
```c
XA_STATE(xas, &mapping->i_pages, start);
void *entry;

rcu_read_lock();
xas_for_each(&xas, entry, end) {
    if (xas_retry(&xas, entry))
        continue;
    /* process entry */
}
rcu_read_unlock();
```

---

### Group 9: xa_state Error & Status Management

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

الـ error بيتخزن encoded في `xas->xa_node` نفسه. `xas_error` بترجع 0 لو لا يوجد خطأ أو الـ errno السالب.

---

#### `xas_invalid` / `xas_valid` / `xas_reset` / `xas_retry`

```c
static inline bool xas_invalid(const struct xa_state *xas)
{
    return (unsigned long)xas->xa_node & 3;
}

static inline void xas_reset(struct xa_state *xas)
{
    xas->xa_node = XAS_RESTART;
}

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

- `xas_invalid`: الـ cursor في حالة error أو restart أو bounds.
- `xas_reset`: بتمسح الـ error state وبترجع الـ cursor لـ XAS_RESTART.
- `xas_retry`: لو الـ entry هو retry أو zero — بتعمل reset وبتشير للـ caller إنه يعيد المحاولة.

**Pattern نموذجي:**
```c
retry:
    rcu_read_lock();
    entry = xas_load(&xas);
    if (xas_retry(&xas, entry))
        goto retry;
    rcu_read_unlock();
```

---

### Group 10: Private/Internal Helpers

هذه الـ functions مكتوبة كـ `/* Private */` في الـ header — للاستخدام الداخلي فقط.

| Function | الغرض |
|---|---|
| `xa_head` | قراءة `xa->xa_head` بـ `rcu_dereference_check` |
| `xa_head_locked` | قراءة `xa->xa_head` بـ `rcu_dereference_protected` (تحت lock) |
| `xa_entry` | قراءة slot من node بـ RCU |
| `xa_entry_locked` | قراءة slot من node تحت lock |
| `xa_parent` | قراءة parent node بـ RCU |
| `xa_parent_locked` | قراءة parent node تحت lock |
| `xa_mk_node` | تحويل `xa_node*` إلى internal entry |
| `xa_to_node` | استخراج `xa_node*` من internal entry |
| `xa_is_node` | فحص إن كان entry يشير لـ node |
| `xa_mk_sibling` | إنشاء sibling entry |
| `xa_to_sibling` | قراءة sibling offset |
| `xa_is_sibling` | فحص sibling entry |
| `xa_is_retry` | فحص retry entry |
| `xa_is_advanced` | فحص إن كان entry يحتاج Advanced API |
| `xas_not_node` | cursor لا يشير لـ node صالح |
| `xas_frozen` | cursor في RESTART أو error state |
| `xas_top` | cursor في head أو RESTART أو BOUNDS |
| `xas_is_node` | cursor يشير لـ node صالح |
| `xas_find_chunk` | بحث في mark bitmap لـ node |

---

### Group 11: Debug Helpers

```c
void xa_dump(const struct xarray *);
void xa_dump_node(const struct xa_node *);

#define XA_BUG_ON(xa, x)      /* يطبع xa_dump ويـ BUG() لو x صحيح */
#define XA_NODE_BUG_ON(node, x) /* يطبع node dump ويـ BUG() */
```

مفعّلة فقط مع `#define XA_DEBUG`. بتطبع الـ tree كاملاً للـ debugging. الـ `XA_BUG_ON` موجودة في production code كـ no-ops.

---

### ملخص: متى تستخدم ماذا؟

```
الاستخدام                          | الـ API المناسب
-----------------------------------|---------------------------
قراءة بسيطة                        | xa_load
كتابة بسيطة                        | xa_store / xa_erase
Atomic check-then-write            | xa_cmpxchg / xa_insert
تخصيص ID تلقائي                    | xa_alloc / xa_alloc_cyclic
حجز مؤقت                           | xa_reserve / xa_release
Iteration بسيطة                    | xa_for_each / xa_for_each_marked
Iteration سريعة أو تحت lock        | xas_for_each / xas_for_each_marked
Multi-index (huge pages)           | xa_store_range / xas_set_order
Lockless page cache lookup         | xas_load + xas_reload
Split huge page                    | xas_split_alloc + xas_split
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — مش موجود مباشرةً للـ XArray

الـ XArray نفسه ما عندوش entries في الـ debugfs — هو data structure داخلي بيتستخدم من subsystems تانية. عشان كده الـ debugging بيتم عن طريق الـ subsystem اللي بيستخدمه.

**مثال عملي — page cache بيستخدم XArray:**

```bash
# اشوف stats الـ page cache اللي بيستخدم XArray داخلياً
cat /proc/vmstat | grep -E "pgpg|pswp|nr_mapped|nr_file_pages"

# memory stats عامة
cat /proc/meminfo | grep -E "Cached|Mapped|AnonPages"
```

**لو الـ subsystem اللي بيستخدم XArray عنده debugfs:**

```bash
# مثال — idr/xarray usage في block layer
ls /sys/kernel/debug/block/
ls /sys/kernel/debug/mm/

# شوف الـ XArray في الـ proc filesystem
cat /proc/slabinfo | grep xarray
```

---

#### 2. sysfs — entries المرتبطة بالـ XArray

الـ XArray بيُستخدم كثيراً في الـ page cache و inode management:

```bash
# stats الـ slab allocator للـ xa_node
cat /sys/kernel/slab/xa_node/alloc_calls
cat /sys/kernel/slab/xa_node/free_calls
cat /sys/kernel/slab/xa_node/objects
cat /sys/kernel/slab/xa_node/object_size

# شوف الـ reclaim pressure على الـ nodes
cat /sys/kernel/slab/xa_node/reclaim_account
```

**الـ slab info بالتفصيل:**

```bash
# اقرا الـ slab stats للـ xa_node
cat /proc/slabinfo | grep xa_node

# Output expected:
# xa_node            256  256  576  7  1 : tunables    0    0    0
#                   ^^^^  ^^^  ^^^
#                active  total  size
```

---

#### 3. ftrace — Tracepoints والـ Events

الـ XArray ما عندوش tracepoints مخصصة، لكن ممكن نتتبعه بـ function tracing:

```bash
# ابدأ function tracing على الـ XArray functions
cd /sys/kernel/debug/tracing

# trace الـ core XArray functions
echo "xa_store xa_load xa_erase xa_alloc __xa_store __xa_erase" > set_ftrace_filter
echo "xas_store xas_load xas_find xas_nomem" >> set_ftrace_filter
echo "function" > current_tracer
echo 1 > tracing_on

# شغل workload
sleep 5
echo 0 > tracing_on

# اقرا النتايج
cat trace | head -100
```

```bash
# لو عايز تتبع memory allocation جوا الـ XArray
echo "xas_nomem xa_node_alloc" > set_ftrace_filter
echo "function_graph" > current_tracer
echo 1 > tracing_on
```

**trace بالـ kprobes على functions مش exported:**

```bash
# probe على xa_node_alloc
echo 'p:xa_alloc_node xa_node_alloc xa=%di' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/xa_alloc_node/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk و Dynamic Debug

**تفعيل dynamic debug للـ XArray:**

```bash
# الـ XArray نفسه ما بيستخدمش pr_debug كتير
# لكن ممكن تفعله لو الـ caller بيستخدمه

# تفعيل كل الـ debug messages في lib/xarray.c
echo "file lib/xarray.c +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ activation
grep "xarray" /sys/kernel/debug/dynamic_debug/control
```

**إضافة printk مؤقت للـ debugging (داخل kernel code):**

```c
/* في lib/xarray.c — أضف للدالة اللي بتشك فيها */
void *xa_store(struct xarray *xa, unsigned long index, void *entry, gfp_t gfp)
{
    void *curr;
    pr_debug("xa_store: xa=%px index=%lu entry=%px\n", xa, index, entry);
    /* ... */
}
```

**تفعيل XA_DEBUG في وقت الـ build:**

```bash
# في .config أو عبر menuconfig
CONFIG_XA_DEBUG=y

# ده بيفعل الـ XA_BUG_ON() و XA_NODE_BUG_ON() macros
# اللي بيعملوا xa_dump() + BUG() لو حصل corruption
```

---

#### 5. Kernel Config options للـ Debugging

| Config Option | الغرض | التأثير |
|---|---|---|
| `CONFIG_XA_DEBUG` | يفعل `XA_BUG_ON` و `XA_NODE_BUG_ON` | dump + BUG() على corruption |
| `CONFIG_XARRAY_MULTI` | دعم multi-index entries | يفعل `xa_is_sibling()` وغيرها |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock bugs في `xa_lock` | يطلع WARN على lock misuse |
| `CONFIG_LOCKDEP` | يتتبع lock ordering للـ `xa_lock` | يكتشف deadlocks محتملة |
| `CONFIG_DEBUG_SLAB` / `CONFIG_KASAN` | يكتشف heap corruption في `xa_node` | يطلع report على use-after-free |
| `CONFIG_PROVE_LOCKING` | يتحقق من lock hierarchy | مفيد مع `rcu_dereference_check` |
| `CONFIG_SLUB_DEBUG` | debug الـ xa_node slab | يكتشف double-free وغيرها |
| `CONFIG_KASAN` | Address Sanitizer | يكتشف out-of-bounds في slots[] |
| `CONFIG_KCSAN` | Kernel Concurrency Sanitizer | يكتشف data races على الـ XArray |

**تفعيل KASAN مع XA_DEBUG:**

```bash
# في .config
CONFIG_KASAN=y
CONFIG_KASAN_GENERIC=y
CONFIG_XA_DEBUG=y
CONFIG_LOCKDEP=y
```

---

#### 6. أدوات خاصة بالـ XArray

**xa_dump() و xa_dump_node():**

الـ header بيعرّف:

```c
void xa_dump(const struct xarray *);
void xa_dump_node(const struct xa_node *);
```

استخدامها من داخل الـ kernel (مش من userspace):

```c
/* في kernel module أو في crash handler */
xa_dump(&my_xarray);
/* Output example:
 * xarray: flags=0 head=ffff888003a12000
 * xa_node: shift=6 offset=0 count=3 nr_values=0
 *   [0]: ffff888004b23400 (pointer)
 *   [1]: ffff888004b23500 (pointer)
 *   [63]: (null)
 */
```

**الـ crash tool لو الـ system اتـ panic:**

```bash
# في crash utility
crash> struct xarray <address>
crash> struct xa_node <node_address>
crash> px <xa->xa_head>   # شوف الـ head entry
```

**الـ /proc/slabinfo للـ xa_node:**

```bash
cat /proc/slabinfo | awk 'NR==1 || /xa_node/'
# Output:
# slabinfo - version: 2.1
# name      active  num_objs  objsize  ...
# xa_node     1024    1024      576    ...
```

---

#### 7. جدول Error Messages الشائعة

| رسالة الخطأ / الظاهرة | المعنى | الحل |
|---|---|---|
| `BUG: xarray: store of internal entry` | محاولة تخزين internal entry بالـ normal API | استخدم `xas_store()` بدل `xa_store()` للـ internal entries |
| `WARNING: suspicious RCU usage in xa_head` | قراءة `xa_head` بدون RCU lock أو `xa_lock` | أضف `rcu_read_lock()` أو خد الـ `xa_lock` |
| `-ENOMEM` من `xa_store` / `xa_alloc` | فشل تخصيص `xa_node` من الـ slab | مرر `GFP_KERNEL` بدل `GFP_ATOMIC` لو السياق بيسمح، أو تأكد ما في memory pressure |
| `-EBUSY` من `xa_insert` | الـ slot مش فاضي | استخدم `xa_store()` للـ overwrite، أو تحقق قبل الـ insert |
| `-EBUSY` من `xa_alloc` | الـ ID range كلها محجوزة | وسّع الـ `xa_limit` أو حرر entries قديمة |
| `XA_BUG_ON` fires + BUG() | corruption في الـ XArray tree structure | فعّل `CONFIG_KASAN` وابحث عن use-after-free في الـ `xa_node` |
| Deadlock في `xa_lock` | nested locking بدون `_nested` variant | استخدم `xa_lock_nested()` مع subclass مناسب |
| `xas->xa_node == XAS_BOUNDS` | الـ xas_for_each وصل لـ index أكبر من الـ tree | طبيعي عند نهاية الـ iteration، مش error |
| `xas->xa_node == XA_ERROR(-ENOMEM)` | `xas_nomem()` فشل | تحقق بـ `xas_error(xas)` بعد كل store |
| Data race على `xa->xa_head` | قراءة concurrent بدون RCU | استخدم `rcu_dereference()` في read path |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* 1. تحقق إن الـ XArray مش corrupted قبل iteration */
WARN_ON(!xa_empty(xa) && !xa->xa_head);

/* 2. تحقق إن الـ lock متاخد صح */
void *my_xa_store_locked(struct xarray *xa, unsigned long index, void *entry)
{
    lockdep_assert_held(&xa->xa_lock);
    return __xa_store(xa, index, entry, GFP_NOWAIT);
}

/* 3. تحقق من الـ entry type قبل الاستخدام */
void process_xa_entry(void *entry)
{
    if (WARN_ON(xa_is_internal(entry))) {
        dump_stack();
        return;
    }
    /* ... */
}

/* 4. تحقق من الـ node integrity */
static void verify_xa_node(struct xa_node *node)
{
    WARN_ON(node->count > XA_CHUNK_SIZE);
    WARN_ON(node->nr_values > node->count);
    WARN_ON(!node->array);
}

/* 5. تحقق من الـ error state بعد xas operations */
xas_store(&xas, entry);
if (WARN_ON(xas_error(&xas))) {
    pr_err("xas_store failed: %d\n", xas_error(&xas));
    dump_stack();
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ XArray هو pure software data structure — ما بيتعامل مع hardware مباشرةً. لكن الـ subsystems اللي بتستخدمه (مثل drivers) ممكن يكون فيها mismatch:

```bash
# مثال: driver بيستخدم XArray لتتبع الـ DMA buffers
# تحقق إن الـ allocated entries في الـ XArray تتطابق مع الـ hardware state

# اشوف الـ DMA buffers المخصصة من الـ driver
cat /sys/bus/pci/devices/0000:01:00.0/resource

# قارن مع عدد الـ entries في الـ XArray بتاع الـ driver
# (لازم يتساوى)
```

---

#### 2. Register Dump Techniques

الـ XArray ما عندوش hardware registers. لكن لو الـ xa_node بيتخزن hardware addresses أو BAR values:

```bash
# اقرا physical memory address محتمل يكون stored في XArray entry
# (لو عندك الـ physical address من crash dump)
devmem2 0xffff888003a12000 q   # q = quad word (64-bit)

# أو عبر /dev/mem (لو مفعل)
dd if=/dev/mem bs=8 count=1 skip=$((0xffff888003a12000 / 8)) 2>/dev/null | xxd
```

**من الـ crash utility:**

```bash
crash> rd -64 <xa_node_address> 16   # اقرا 16 quadwords من الـ node
crash> struct xa_node.slots <address> # شوف الـ slots array
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ XArray نفسه software-only، لكن لو الـ driver اللي بيستخدمه بيتعامل مع hardware:

- **راقب الـ DMA transactions**: لو الـ XArray بيتتبع DMA buffers، استخدم logic analyzer على الـ PCIe bus لتأكيد إن الـ buffer addresses صح.
- **راقب الـ IRQ lines**: لو الـ XArray بيتتبع interrupt handlers، تأكد من توقيت الـ IRQ مع الـ xa_store calls.
- **أهم signal تراقبه**: الـ spinlock acquisition time — لو الـ xa_lock متاخد وقت طويل، ده ممكن يسبب latency مرئي على hardware signals.

```
Logic Analyzer setup:
  CH1: IRQ line من الـ hardware
  CH2: GPO pin تشغله يدوياً قبل/بعد xa_store()
  Trigger: CH1 rising edge
  Goal: قيس latency بين IRQ وبين الـ xa_store completion
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ dmesg | السبب المحتمل |
|---|---|---|
| Driver فقد track لـ DMA buffer | `use-after-free in xa_load` (من KASAN) | Driver حذف entry من XArray قبل ما يخلص الـ DMA |
| IRQ storm بتملا الـ XArray | `xa_alloc: -ENOMEM` بالـ thousands | الـ XArray alloc limit وصله أو memory نفد |
| Hardware reset مع entries لسه موجودة | `WARN_ON` في الـ driver cleanup | الـ driver ما عملش `xa_destroy()` قبل الـ reset |
| Multi-CPU race على الـ XArray | `BUG: soft lockup` أو KCSAN warning | الـ IRQ-safe locking (`xa_lock_irq`) مش متستخدم |

---

#### 5. Device Tree Debugging

الـ XArray بيتستخدم أحياناً في platform drivers اللي بتقرأ من DT:

```bash
# تحقق إن الـ DT node موجود ومتعرف
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "your-device"

# تحقق إن الـ driver اتـ probe بنجاح (وبالتالي XArray اتـ initialize)
dmesg | grep "your_driver: probe"

# شوف الـ resources اللي الـ driver طلبها من DT
cat /sys/bus/platform/devices/your-device/resources
```

**فحص تطابق DT مع الـ hardware:**

```bash
# تأكد إن الـ reg property في DT بيتطابق مع الـ physical hardware
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "your-node"

# قارن مع /proc/iomem
cat /proc/iomem | grep "your-device-region"
```

---

### Practical Commands

#### جاهز للـ Copy-Paste

**1. فحص سريع للـ xa_node slab health:**

```bash
#!/bin/bash
echo "=== XArray Node Slab Stats ==="
cat /proc/slabinfo | awk 'NR==1 {print} /xa_node/ {print}'

echo ""
echo "=== XArray Node in /sys/kernel/slab ==="
if [ -d /sys/kernel/slab/xa_node ]; then
    for f in objects object_size alloc_calls free_calls; do
        echo "$f: $(cat /sys/kernel/slab/xa_node/$f 2>/dev/null || echo N/A)"
    done
fi
```

**مثال على الـ output:**

```
=== XArray Node Slab Stats ===
slabinfo - version: 2.1
# name      <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
xa_node         768         768        576        7         1

=== XArray Node in /sys/kernel/slab ===
objects: 768
object_size: 576
alloc_calls: 1024
free_calls: 256
```

**التفسير**: لو `alloc_calls - free_calls` كبير جداً مع مرور الوقت، ده memory leak في الـ XArray (عمل `xa_store` بدون `xa_destroy`).

---

**2. Function tracing للـ XArray:**

```bash
#!/bin/bash
TRACEDIR=/sys/kernel/debug/tracing

echo nop > $TRACEDIR/current_tracer
echo "" > $TRACEDIR/set_ftrace_filter

# XArray core functions
for fn in xa_store xa_load xa_erase xa_alloc \
           __xa_store __xa_erase __xa_insert __xa_alloc \
           xas_store xas_load xas_find xas_nomem \
           xa_node_alloc; do
    echo $fn >> $TRACEDIR/set_ftrace_filter 2>/dev/null
done

echo function > $TRACEDIR/current_tracer
echo 1 > $TRACEDIR/tracing_on
echo "Tracing XArray for 10 seconds..."
sleep 10
echo 0 > $TRACEDIR/tracing_on

echo "=== Top XArray callers ==="
cat $TRACEDIR/trace | grep -v "^#" | \
    awk '{print $NF}' | sort | uniq -c | sort -rn | head -20

echo "nop" > $TRACEDIR/current_tracer
```

**مثال output:**

```
=== Top XArray callers ===
   4523 xa_store+0x0/0x60
   3812 xa_load+0x0/0x40
    921 xa_alloc+0x0/0x80
     45 xas_nomem+0x0/0x30
      3 xa_destroy+0x0/0x20
```

**التفسير**: نسبة `xa_alloc / xa_destroy` المرتفعة بدون `xa_erase` = memory leak مؤكد.

---

**3. تفعيل XA_DEBUG وبناء الـ kernel:**

```bash
# في مجلد kernel source
scripts/config --enable XA_DEBUG
scripts/config --enable CONFIG_KASAN
scripts/config --enable CONFIG_KASAN_GENERIC
scripts/config --enable CONFIG_LOCKDEP

make olddefconfig
make -j$(nproc)

# بعد الـ boot، أي corruption هيظهر كـ:
# [  123.456789] XArray: WARN at lib/xarray.c:XXX
# [  123.456789] xa_dump: xarray ffff888003a12000 ...
```

---

**4. KASAN report interpretation:**

```bash
# مثال output من KASAN على XArray bug:
# ==================================================================
# BUG: KASAN: use-after-free in xa_load+0x3c/0x60
# Read of size 8 at addr ffff888003a12048 by task kworker/0:1/123
#
# Allocated by task 456:
#  xa_node_alloc+0x28/0x80
#  xas_expand+0x1a0/0x2e0
#  xas_store+0x124/0x4a0
#  xa_store+0x48/0x60
#  my_driver_add_buffer+0x3c/0x80  <-- هنا الـ original store
#
# Freed by task 789:
#  xa_node_free+0x18/0x40
#  xa_delete_node+0x88/0xc0
#  xa_erase+0x4c/0x60
#  my_driver_remove_buffer+0x2c/0x60  <-- هنا حصل early free
```

**التفسير**: الـ driver عمل `xa_erase` على buffer لسه بيتستخدم. الحل: تأكد إن كل الـ readers خلصوا قبل الـ erase، واستخدم RCU لو لازم.

---

**5. فحص lockdep لـ xa_lock:**

```bash
# فعّل CONFIG_PROVE_LOCKING ثم شوف الـ lock classes
cat /proc/lockdep | grep xa_lock

# أو من dmesg بعد lockdep warning:
dmesg | grep -A 30 "possible circular locking"

# مثال output:
# WARNING: possible circular locking dependency detected
# kworker/1:2/456 is trying to acquire lock:
# ffff888003a12000 (&xa->xa_lock){....}, at: xa_store+0x30/0x60
#
# but task is already holding lock:
# ffff888004b23000 (&dev->lock){....}, at: my_driver_callback+0x10/0x80
#
# which lock ordering was previously:
# (&dev->lock) -> (&xa->xa_lock)   [correct order]
# (&xa->xa_lock) -> (&dev->lock)   [WRONG - this path]
```

**الحل**: خلّي ترتيب الـ lock acquisition ثابت دايماً: `dev->lock` أولاً ثم `xa->xa_lock`.

---

**6. شوف إيه اللي مخزن في XArray من الـ crash dump:**

```bash
# في crash utility بعد kernel panic
crash> struct xarray my_driver_xa
# Output:
# xarray = {
#   xa_lock = {...},
#   xa_flags = 4,        # XA_FLAGS_TRACK_FREE
#   xa_head = 0xffff888003a12002  # LSBit=10 يعني internal entry (xa_node pointer)
# }

crash> struct xa_node 0xffff888003a12000  # اطرح الـ 2 من الـ tagged pointer
# Output:
# xa_node = {
#   shift = 0,
#   offset = 0,
#   count = 3,           # 3 non-NULL slots
#   nr_values = 0,       # مفيش value entries, كل entries pointers
#   parent = 0x0,        # ده الـ root node
#   array = 0xffff888001234000,
#   slots = {
#     [0] = 0xffff888004b23400,  # pointer entry (LSBits=00)
#     [1] = 0xffff888004b23500,
#     [2] = 0x0,
#     ...
#   }
# }
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Deadlock في درايفر SPI على AM62x — بوابة صناعية

#### العنوان
**Deadlock** بين IRQ handler وprocess context بسبب استخدام `xa_store()` بدل `xa_store_irq()` في تتبع SPI transactions

#### السياق
بوابة صناعية مبنية على **Texas Instruments AM62x** تشغّل Linux 6.6. الدرايفر مسؤول عن إدارة عدة SPI sensors في نفس الوقت. كل transaction بتاخد ID فريد يتخزن في XArray لتتبع الـ state.

```
SPI Master (AM62x) ──► Sensor 1 (pressure)
                  ──► Sensor 2 (temperature)
                  ──► Sensor 3 (flow meter)
```

#### المشكلة
بعد تشغيل الجهاز لفترة طويلة، النظام بيتجمد تماماً. الـ console مش بيرد. الـ watchdog بيعمل reset كل دقيقتين.

```bash
# في الـ kernel log قبل ما يتجمد:
[ 4821.334521] INFO: task kworker/0:2:89 blocked for more than 120 seconds.
[ 4821.341123]       Not tainted 6.6.0-am62x #1
[ 4821.344891] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[ 4821.352910] task:kworker/0:2     state:D stack:0     pid:89    ppid:2
[ 4821.359234] Call trace:
[ 4821.361712]  __switch_to+0x90/0xe4
[ 4821.365129]  __schedule+0x2b4/0x6b0
[ 4821.368634]  schedule+0x58/0xd0
[ 4821.371799]  schedule_preempt_disabled+0x18/0x28
[ 4821.376527]  __spin_lock_slowpath+0x64/0x80   ← stuck on xa_lock!
[ 4821.382451]  spi_sensor_complete_irq+0x4c/0x9c [spi_sensor_drv]
```

#### التحليل
**الـ flow المعطوب:**

```
Process context (kworker):
    xa_store(xa, id, entry, GFP_KERNEL)
        → xa_lock(xa)           ← يمسك xa_lock (spin_lock عادي)
        → __xa_store(...)
        → [interrupted by SPI IRQ!]

IRQ context (SPI completion):
    spi_sensor_complete_irq()
        → xa_store(xa, id, NULL, GFP_ATOMIC)  ← يحاول يمسك xa_lock
        → spin_lock(&xa->xa_lock)              ← DEADLOCK! الـ lock ممسوك من process
```

**المشكلة في الكود:** الدرايفر استخدم `xa_store()` في الاتنين contexts. الـ `xa_store()` من تعريفه في `xarray.h`:

```c
/* xa_store() uses xa_lock() which is plain spin_lock — NOT irq-safe */
void *xa_store(struct xarray *, unsigned long index, void *entry, gfp_t);
```

الـ `xa_lock` macro هو:
```c
#define xa_lock(xa)   spin_lock(&(xa)->xa_lock)
```

`spin_lock` العادي ما بيعطّل الـ IRQs، فلما الـ IRQ جه وحاول يمسك نفس الـ lock، حصل deadlock.

**الـ XArray flag** كمان كان غلط — الـ XArray اتعمل بـ:
```c
xa_init_flags(&spi_xa, 0);  /* no XA_FLAGS_LOCK_IRQ! */
```

بدل:
```c
xa_init_flags(&spi_xa, XA_FLAGS_LOCK_IRQ);
```

#### الحل

```c
/* درايفر SPI sensor بعد الإصلاح */

/* 1. تهيئة الـ XArray بـ flag صح */
static DEFINE_XARRAY_FLAGS(spi_transactions, XA_FLAGS_LOCK_IRQ);

/* 2. في process context: استخدم xa_store_irq() */
static int spi_sensor_submit(struct spi_device *spi, struct spi_msg *msg)
{
    u32 id;
    int ret;

    /* xa_alloc_irq: disables IRQs while holding xa_lock */
    ret = xa_alloc_irq(&spi_transactions, &id, msg,
                        xa_limit_32b, GFP_KERNEL);
    if (ret < 0)
        return ret;

    msg->id = id;
    return spi_async(spi, &msg->xfer);
}

/* 3. في IRQ context: استخدم __xa_erase مع xa_lock_irqsave */
static irqreturn_t spi_sensor_complete_irq(int irq, void *dev)
{
    struct spi_msg *msg = get_completed_msg(dev);
    unsigned long flags;

    /* xa_lock_irqsave: safe from any context */
    xa_lock_irqsave(&spi_transactions, flags);
    __xa_erase(&spi_transactions, msg->id);
    xa_unlock_irqrestore(&spi_transactions, flags);

    complete(&msg->done);
    return IRQ_HANDLED;
}
```

#### الدرس المستفاد
الـ `xa_store()` و`xa_erase()` العاديين بيمسكوا `spin_lock` فقط — مش safe من IRQ context. إذا كان فيه أي احتمال إن الـ XArray يتلمس من IRQ handler، لازم:
1. تعمل `xa_init_flags()` بـ `XA_FLAGS_LOCK_IRQ`
2. تستخدم variants زي `xa_store_irq()` أو `xa_lock_irqsave()` + `__xa_store()`

---

### السيناريو الثاني: Memory Leak في درايفر HDMI على RK3562 — Android TV Box

#### العنوان
**Memory leak** بسبب عدم استدعاء `xa_destroy()` عند فشل الـ probe في درايفر HDMI

#### السياق
Android TV box مبنية على **Rockchip RK3562**. الدرايفر مسؤول عن إدارة HDMI connectors ومعلومات الـ EDID. كل connector بيخزن pointer في XArray بـ index = connector ID.

#### المشكلة
عند كل محاولة فاشلة لـ `probe()` (مثلاً لما الـ HDMI clock مش جاهز)، الـ kernel بيستهلك memory أكتر وأكتر. بعد ~100 retry، الـ OOM killer بيبدأ يشتغل.

```bash
# مشهود في dmesg:
[  12.441] rk3562-hdmi: probe deferred, clock not ready
[  14.441] rk3562-hdmi: probe deferred, clock not ready
...
[ 412.221] Out of memory: Kill process 1234 (surfaceflinger) ...
[ 412.224] oom_score_adj: 0
```

#### التحليل
الدرايفر عنده `probe()` زي كده:

```c
static int hdmi_probe(struct platform_device *pdev)
{
    struct hdmi_dev *hdev;
    int i, ret;

    hdev = devm_kzalloc(&pdev->dev, sizeof(*hdev), GFP_KERNEL);
    if (!hdev)
        return -ENOMEM;

    /* XArray اتعمل صح */
    xa_init_flags(&hdev->connectors, XA_FLAGS_ALLOC);

    /* populate connectors */
    for (i = 0; i < MAX_CONNECTORS; i++) {
        struct hdmi_connector *conn = alloc_connector(i);
        ret = xa_alloc(&hdev->connectors, &conn->id,
                       conn, xa_limit_16b, GFP_KERNEL);
        if (ret)
            goto err_clock;  /* ← البق هنا */
    }

    ret = clk_prepare_enable(hdev->pclk);
    if (ret)
        goto err_clock;

    return 0;

err_clock:
    /* المشكلة: xa_destroy() مش متعمل! */
    return ret;
}
```

**ليه بيحصل leak؟**

الـ `xa_alloc()` بتخصص `xa_node` objects داخلياً في الـ XArray tree. الـ `devm_kzalloc` بيحرر struct الـ `hdev` نفسه عند الـ device removal أو الـ probe failure، لكن الـ `xa_node` objects المخصصة جوه الـ XArray **مش managed بـ devm** — اتخصصت عن طريق `kmalloc` internal في الـ XArray code.

لو ما استدعيتش `xa_destroy()` قبل ما تيجي الـ error path، الـ `xa_node` objects تفضل في الـ memory.

```c
/* التأكيد: xa_destroy() موجودة في xarray.h */
void xa_destroy(struct xarray *);
```

#### الحل

```c
static int hdmi_probe(struct platform_device *pdev)
{
    struct hdmi_dev *hdev;
    int i, ret;

    hdev = devm_kzalloc(&pdev->dev, sizeof(*hdev), GFP_KERNEL);
    if (!hdev)
        return -ENOMEM;

    xa_init_flags(&hdev->connectors, XA_FLAGS_ALLOC);

    for (i = 0; i < MAX_CONNECTORS; i++) {
        struct hdmi_connector *conn = alloc_connector(i);
        if (!conn) {
            ret = -ENOMEM;
            goto err_xa;
        }
        ret = xa_alloc(&hdev->connectors, &conn->id,
                       conn, xa_limit_16b, GFP_KERNEL);
        if (ret)
            goto err_xa;
    }

    ret = clk_prepare_enable(hdev->pclk);
    if (ret)
        goto err_xa;

    return 0;

err_xa:
    /* حرر كل الـ connectors الموجودة في الـ XArray */
    {
        unsigned long idx;
        struct hdmi_connector *conn;
        xa_for_each(&hdev->connectors, idx, conn)
            free_connector(conn);
    }
    /* حرر الـ internal xa_nodes */
    xa_destroy(&hdev->connectors);
    return ret;
}
```

**أو الأفضل:** استخدام `devm_add_action_or_reset()`:

```c
static void hdmi_xa_cleanup(void *data)
{
    struct xarray *xa = data;
    unsigned long idx;
    struct hdmi_connector *conn;

    xa_for_each(xa, idx, conn)
        free_connector(conn);
    xa_destroy(xa);
}

/* في probe: */
xa_init_flags(&hdev->connectors, XA_FLAGS_ALLOC);
ret = devm_add_action_or_reset(&pdev->dev, hdmi_xa_cleanup,
                                &hdev->connectors);
```

#### الدرس المستفاد
الـ `xa_init_flags()` بتشيّل الـ XArray على الـ stack أو embedded في struct، لكن الـ `xa_node` objects الداخلية بتتخصص بـ `kmalloc`. لازم تستدعي `xa_destroy()` دايماً في error paths. الـ `devm` ما بيحررش الـ XArray internals تلقائياً.

---

### السيناريو الثالث: Race Condition في درايفر I2C على STM32MP1 — ECU أوتوموتيف

#### العنوان
**Race condition** بسبب استخدام `xa_load()` بدون lock في context يحتاج consistency مع write operation في درايفر I2C لـ ECU أوتوموتيف

#### السياق
**Automotive ECU** مبني على **STM32MP1** بيشغّل AUTOSAR-like stack فوق Linux. الدرايفر بيدير I2C slaves (sensors + actuators). كل slave ليه struct يتخزن في XArray بـ I2C address كـ index.

المشكلة ظهرت في اختبارات stress testing عند الـ hot-plug لأجهزة I2C (بعض الـ sensors اختيارية وبتتوصل/تتفصل أثناء التشغيل).

#### المشكلة
في حالات نادرة، الـ ECU بيطلع NULL pointer dereference:

```
[ 3421.112] BUG: kernel NULL pointer dereference, address: 00000010
[ 3421.118] pc : i2c_sensor_read+0x48/0xb0 [i2c_ecg_drv]
[ 3421.123] lr : i2c_periodic_task+0x2c/0x68 [i2c_ecg_drv]
[ 3421.129] Call trace:
[ 3421.131]  i2c_sensor_read+0x48/0xb0
[ 3421.134]  i2c_periodic_task+0x2c/0x68
[ 3421.138]  kthread+0x110/0x130
```

#### التحليل
الكود:

```c
/* Thread بيقرأ sensors كل 10ms */
static int i2c_periodic_task(void *data)
{
    struct i2c_drv *drv = data;
    unsigned long addr;
    struct i2c_slave *slave;

    while (!kthread_should_stop()) {
        /* ← xa_load() بياخد RCU lock بس، مش xa_lock */
        xa_for_each(&drv->slaves, addr, slave) {
            i2c_sensor_read(slave);  /* ← لو slave اتمسح هنا؟ */
        }
        msleep(10);
    }
    return 0;
}

/* Thread تاني بيتعامل مع hot-plug */
static void i2c_slave_disconnect(struct i2c_drv *drv, u32 addr)
{
    struct i2c_slave *slave;

    slave = xa_erase(&drv->slaves, addr);  /* ← بيمسح الـ slave */
    if (slave)
        kfree(slave);  /* ← بيحرر الـ memory */
}
```

**ليه بيحصل race؟**

الـ `xa_for_each()` macro بيستخدم `xa_find()` و`xa_find_after()` اللي بياخدوا **RCU lock بس** مش `xa_lock`:

```c
/* من xarray.h: */
#define xa_for_each(xa, index, entry) \
    xa_for_each_start(xa, index, entry, 0)

/* xa_find() takes and releases the RCU lock — NOT xa_lock */
void *xa_find(struct xarray *xa, unsigned long *index,
              unsigned long max, xa_mark_t);
```

الـ RCU lock بيضمن إن الـ pointer مش هيتحرر *وانت شايله في ايدك* (لو استخدمت `rcu_read_lock()`). لكن الكود هنا:
1. خد الـ pointer من `xa_for_each()` (مع RCU)
2. الـ RCU lock اتحرر في نهاية الـ iteration step
3. الـ `kfree(slave)` حصلت قبل ما `i2c_sensor_read(slave)` تخلص

**المشكلة الجوهرية:** الـ `slave` اتحرر بـ `kfree()` مباشرة بدل RCU-safe free، فالـ RCU grace period ما اتحرمش.

#### الحل

خيارين:

**الخيار الأول:** استخدام `xa_lock` صريح حول الـ iteration:

```c
static int i2c_periodic_task(void *data)
{
    struct i2c_drv *drv = data;
    unsigned long addr;
    struct i2c_slave *slave;

    while (!kthread_should_stop()) {
        xa_lock(&drv->slaves);
        xa_for_each(&drv->slaves, addr, slave) {
            /* slave safe هنا لأن xa_lock ماسكه */
            __i2c_sensor_read_locked(slave);
        }
        xa_unlock(&drv->slaves);
        msleep(10);
    }
    return 0;
}

static void i2c_slave_disconnect(struct i2c_drv *drv, u32 addr)
{
    struct i2c_slave *slave;

    xa_lock(&drv->slaves);
    slave = __xa_erase(&drv->slaves, addr);
    xa_unlock(&drv->slaves);

    if (slave)
        kfree(slave);
}
```

**الخيار الثاني (أفضل للأداء):** RCU-safe free:

```c
static void i2c_slave_rcu_free(struct rcu_head *head)
{
    struct i2c_slave *slave = container_of(head, struct i2c_slave, rcu);
    kfree(slave);
}

static void i2c_slave_disconnect(struct i2c_drv *drv, u32 addr)
{
    struct i2c_slave *slave;

    slave = xa_erase(&drv->slaves, addr);
    if (slave)
        /* يستنى الـ RCU grace period قبل ما يحرر */
        call_rcu(&slave->rcu, i2c_slave_rcu_free);
}

/* في read task: استخدم rcu_read_lock() صريح */
static int i2c_periodic_task(void *data)
{
    struct i2c_drv *drv = data;
    unsigned long addr;
    struct i2c_slave *slave;

    while (!kthread_should_stop()) {
        rcu_read_lock();
        xa_for_each(&drv->slaves, addr, slave) {
            i2c_sensor_read(slave); /* safe: RCU grace period ضامن الـ memory */
        }
        rcu_read_unlock();
        msleep(10);
    }
    return 0;
}
```

#### الدرس المستفاد
الـ `xa_for_each()` و`xa_load()` بياخدوا RCU lock — مش `xa_lock`. لو بتحرر الـ entries بـ `kfree()` مباشرة، لازم تمسك `xa_lock` صريح حول الـ iteration. لو عايز lockless iteration، لازم تستخدم `call_rcu()` أو `synchronize_rcu()` قبل الـ `kfree()`.

---

### السيناريو الرابع: خطأ في ID Allocation على i.MX8 — IoT Gateway مع USB Devices

#### العنوان
**-EBUSY error** متكرر من `xa_alloc()` بسبب استخدام XArray بدون `XA_FLAGS_ALLOC` في درايفر USB device management

#### السياق
IoT gateway مبني على **NXP i.MX8MM** بيدير أجهزة USB متعددة (USB-to-UART adapters, USB sensors). كل جهاز بياخد ID فريد من XArray ليتعرف عليه في الـ userspace عبر sysfs.

#### المشكلة
عند توصيل أكتر من 3-4 أجهزة USB، الـ allocation بتفشل بـ `-EBUSY` حتى لما في IDs متاحة. الـ dmesg بيقول:

```
[  45.221] usb 1-1.2: new USB device attached
[  45.334] usb_iot: xa_alloc failed: -EBUSY for device usb1-1.2
[  45.340] usb_iot: device registration failed
```

#### التحليل
الدرايفر عمل:

```c
/* تهيئة غلط */
static DEFINE_XARRAY(usb_devices);  /* equivalent to xa_init_flags(&xa, 0) */

static int usb_iot_probe(struct usb_interface *intf, ...)
{
    struct usb_iot_dev *dev;
    u32 id;
    int ret;

    dev = kzalloc(sizeof(*dev), GFP_KERNEL);

    /* ← هنا المشكلة */
    ret = xa_alloc(&usb_devices, &id, dev,
                   xa_limit_16b, GFP_KERNEL);
    if (ret) {
        dev_err(&intf->dev, "xa_alloc failed: %d\n", ret);
        goto err;
    }
    ...
}
```

**ليه بيرجع `-EBUSY`؟**

الـ `xa_alloc()` من تعريفه في `xarray.h`:

```c
/**
 * Must only be operated on an xarray initialized with flag
 * XA_FLAGS_ALLOC set in xa_init_flags().
 */
static inline __must_check int xa_alloc(struct xarray *xa, u32 *id, ...)
{
    ...
    err = __xa_alloc(xa, id, entry, limit, gfp);
    ...
}
```

الـ `XA_FLAGS_ALLOC` بيساوي:
```c
#define XA_FLAGS_ALLOC  (XA_FLAGS_TRACK_FREE | XA_FLAGS_MARK(XA_FREE_MARK))
```

الـ `XA_FLAGS_TRACK_FREE` بيخلي الـ XArray يتتبع ايه الـ slots الفاضية باستخدام الـ `XA_FREE_MARK`. بدون الـ flag ده، `__xa_alloc()` ما بيلاقيش الـ free slots بشكل صح وبيرجع `-EBUSY`.

**إضافي:** الدرايفر فضّل يستخدم `DEFINE_XARRAY` (بدون `_ALLOC`)، فالـ free mark مش متفعّل أصلاً. الـ `xa_alloc()` بتحاول تلاقي slot marked كـ `XA_FREE_MARK` — وما بتلاقيش لأن ما فيش marks اتضبطت.

#### الحل

```c
/* إصلاح 1: استخدام DEFINE_XARRAY_ALLOC */
static DEFINE_XARRAY_ALLOC(usb_devices);

/* أو يدوياً: */
static struct xarray usb_devices;

static int __init usb_iot_init(void)
{
    xa_init_flags(&usb_devices, XA_FLAGS_ALLOC);
    return usb_register(&usb_iot_driver);
}
```

لو IDs محتاجة تبدأ من 1 (مثلاً لـ compatibility مع userspace):
```c
static DEFINE_XARRAY_ALLOC1(usb_devices);  /* IDs start at 1 */
```

لو عايز range محدد (مثلاً 100-199 للـ USB devices):
```c
ret = xa_alloc(&usb_devices, &id, dev,
               XA_LIMIT(100, 199),    /* struct xa_limit */
               GFP_KERNEL);
```

**تحقق بعد الإصلاح:**
```bash
# شوف الـ IDs المخصصة
cat /sys/kernel/debug/usb_iot/allocated_ids

# أو من الـ kernel:
unsigned long idx;
struct usb_iot_dev *dev;
xa_for_each(&usb_devices, idx, dev)
    pr_info("USB device at ID %lu\n", idx);
```

#### الدرس المستفاد
**الـ `xa_alloc()` تشتغل صح بس مع XArray اتعمل بـ `XA_FLAGS_ALLOC`.** الفرق بين `DEFINE_XARRAY` و`DEFINE_XARRAY_ALLOC` مش واضح من الاسم بس من الـ semantics. دايماً اقرأ الـ comment اللي جنب `xa_alloc()` في `xarray.h`:
```
Must only be operated on an xarray initialized with flag
XA_FLAGS_ALLOC set in xa_init_flags().
```

---

### السيناريو الخامس: استخدام Advanced API غلط على Allwinner H616 — Custom Board Bring-up

#### العنوان
**Kernel BUG()** بسبب تخزين internal entry عبر Advanced API في درايفر page cache مخصص على **Allwinner H616**

#### السياق
Custom embedded board مبنية على **Allwinner H616** لتطبيق media processing. المهندس كان بيعمل bring-up لـ custom memory allocator بيستخدم XArray لتتبع الـ allocated pages. استخدم الـ Advanced API (`xas_store()`) بشكل مباشر لأداء أفضل.

#### المشكلة
الـ kernel يعمل panic فوراً عند أول allocation:

```
[    2.334] kernel BUG at include/linux/xarray.h:0!
[    2.338] Internal error: Oops - BUG: 0 [#1] SMP ARM
[    2.344] Modules linked in: h616_mem_alloc(+)
[    2.349] PC is at xa_store+0x8c/0x120
[    2.353] Call trace:
[    2.355]  xa_store+0x8c/0x120
[    2.358]  xas_store+0x44/0x2a0
[    2.362]  h616_mem_alloc_page+0x38/0x80 [h616_mem_alloc]
```

#### التحليل
كود المهندس:

```c
static int h616_mem_alloc_page(struct h616_allocator *alloc, u32 idx)
{
    XA_STATE(xas, &alloc->pages, idx);
    void *entry;

    /* أراد يخزن value entry (مش pointer) */
    entry = xa_mk_value(idx);  /* ← صح، value entry */

    xas_lock(&xas);
    xas_store(&xas, entry);    /* ← هنا الـ BUG */
    xas_unlock(&xas);

    return xas_error(&xas);
}
```

**المشكلة:** المهندس فهم غلط. `xa_mk_value()` بترجع value entry صح — لكن المشكلة مش هنا.

المشكلة الحقيقية كانت في كود تاني:

```c
/* المهندس حاول يخزن zero entry يدوياً! */
static void h616_mem_reserve(struct h616_allocator *alloc, u32 idx)
{
    XA_STATE(xas, &alloc->pages, idx);

    xas_lock(&xas);
    /* ظن إن XA_ZERO_ENTRY هو "reserved" marker */
    xas_store(&xas, XA_ZERO_ENTRY);   /* ← BUG هنا! */
    xas_unlock(&xas);
}
```

**من `xarray.h`:**

```c
#define XA_ZERO_ENTRY    xa_mk_internal(257)

/* xa_is_advanced() checks: */
static inline bool xa_is_advanced(const void *entry)
{
    return xa_is_internal(entry) && (entry <= XA_RETRY_ENTRY);
}

/* XA_ZERO_ENTRY هو internal entry */
/* xas_store() بتعمل BUG_ON لو حاولت تخزن internal entry */
```

الـ `XA_ZERO_ENTRY` هو internal entry. الـ Advanced API (`xas_store()`) مسموح لها تخزنه، لكن يوجد check في الكود إن الـ XA_ZERO_ENTRY مش مفروض يتخزن مباشرة من الـ user بالطريقة دي — المفروض يتعمل عبر `xa_reserve()` اللي بتستخدمه internally.

**الصح:** استخدام `xa_reserve()` للـ reservation:

```c
/* xa_reserve() internally stores XA_ZERO_ENTRY safely */
static inline __must_check
int xa_reserve(struct xarray *xa, unsigned long index, gfp_t gfp)
{
    return xa_err(xa_cmpxchg(xa, index, NULL, XA_ZERO_ENTRY, gfp));
}
```

**وأيضاً:** المهندس كان يحاول يخزن integer values (indices) مباشرة بدل pointers. الطريقة الصح:

```c
/* value entries: bit 0 = 1 */
void *entry = xa_mk_value(42UL);    /* stores integer 42 */
unsigned long val = xa_to_value(xa_load(&xa, idx));  /* retrieves 42 */
```

لكن WARN_ON في `xa_mk_value()` بيطلع لو الـ value negative:
```c
static inline void *xa_mk_value(unsigned long v)
{
    WARN_ON((long)v < 0);  /* ← لو المهندس بعت signed negative */
    return (void *)((v << 1) | 1);
}
```

#### الحل

```c
/* إصلاح 1: استخدام xa_reserve() للـ reservation */
static int h616_mem_reserve(struct h616_allocator *alloc, u32 idx)
{
    return xa_reserve(&alloc->pages, idx, GFP_KERNEL);
}

/* إصلاح 2: تخزين value entries صح */
static int h616_mem_alloc_page(struct h616_allocator *alloc, u32 idx,
                                struct page *page)
{
    void *old;

    /* خزّن pointer للـ page مش value integer */
    old = xa_store(&alloc->pages, idx, page, GFP_KERNEL);
    if (xa_is_err(old))
        return xa_err(old);

    return 0;
}

/* إصلاح 3: لو فعلاً محتاج value integers */
static int h616_mem_store_id(struct h616_allocator *alloc,
                              u32 slot, unsigned long val)
{
    void *entry = xa_mk_value(val);  /* val must be < LONG_MAX */
    void *old;

    old = xa_store(&alloc->pages, slot, entry, GFP_KERNEL);
    return xa_err(old);
}

/* استرجاع: */
static unsigned long h616_mem_get_id(struct h616_allocator *alloc, u32 slot)
{
    void *entry = xa_load(&alloc->pages, slot);

    if (!entry || !xa_is_value(entry))
        return 0;
    return xa_to_value(entry);
}
```

**Debug commands مفيدة:**
```bash
# شوف state الـ XArray
echo "xa_dump" > /sys/kernel/debug/h616_alloc/control

# في الكود:
xa_dump(&alloc->pages);       /* prints full tree */
xa_dump_node(xas.xa_node);   /* prints specific node */
```

#### الدرس المستفاد
**الـ Internal entries مش للـ users.** الـ `XA_ZERO_ENTRY`، `XA_RETRY_ENTRY`، وكل `xa_mk_internal()` entries محجوزة للـ kernel internal use فقط. الـ `xa_is_advanced()` بتكشف دي. للـ reservation استخدم `xa_reserve()`. للـ integer storage استخدم `xa_mk_value()` / `xa_to_value()`. في الـ Advanced API، الـ `xas_store()` ما بتقبلش internal entries عشان تحافظ على integrity الـ tree.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لأي تطور في kernel. الـ XArray اتكلم عنه في سلسلة مقالات مهمة:

| المقال | الوصف |
|--------|-------|
| [Introducing the eXtensible Array (xarray)](https://lwn.net/Articles/715948/) | أول مقال بيعرّف الـ XArray وفكرته وسبب إنشاؤه كبديل لـ radix tree |
| [The XArray data structure](https://lwn.net/Articles/745073/) | تغطية عرض Matthew Wilcox في linux.conf.au 2018 — التصميم والـ API |
| [XArray documentation](https://lwn.net/Articles/739986/) | مراجعة التوثيق الرسمي والـ API النهائي |
| [XArray and the mainline](https://lwn.net/Articles/757342/) | قصة الدخول للـ mainline في kernel 4.20 |
| [Convert page cache to XArray](https://lwn.net/Articles/750540/) | التحويل الكبير — الـ page cache اتحول من radix tree لـ XArray |
| [XArray v8](https://lwn.net/Articles/748680/) | الـ patch set الإصدار 8 قبل الـ merge النهائي |

---

### التوثيق الرسمي في الـ Kernel

**الـ Documentation/** في الـ kernel هو الأساس:

```
Documentation/core-api/xarray.rst
```

- [XArray — The Linux Kernel documentation (docs.kernel.org)](https://docs.kernel.org/core-api/xarray.html)
- [XArray — kernel v5.1 docs](https://www.kernel.org/doc/html/v5.1/core-api/xarray.html)

**الملفات المصدرية** المباشرة في الـ kernel tree:

```
include/linux/xarray.h     ← الـ API العلني والـ inline functions
lib/xarray.c               ← التنفيذ الأساسي
lib/radix-tree.c           ← compat layer مع الـ radix tree القديم
```

- [linux/include/linux/xarray.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/xarray.h)
- [linux/lib/xarray.c على GitHub](https://github.com/torvalds/linux/blob/master/lib/xarray.c)
- [linux/Documentation/core-api/xarray.rst على GitHub](https://github.com/torvalds/linux/blob/master/Documentation/core-api/xarray.rst)

---

### Commits المهمة في تاريخ XArray

**الـ [GIT PULL] XArray for 4.20** — الرسالة الرسمية اللي Matthew Wilcox بعتها لـ Linus لـ merge الـ XArray:
- [Linux-Kernel Archive: [GIT PULL] XArray for 4.20](https://lkml.iu.edu/hypermail/linux/kernel/1810.2/06430.html)

**الـ commit الأساسي اللي خلّى الـ XArray يدخل kernel 4.20:**
- اتعمل merge في October 2018 مع kernel 4.20-rc1
- شمل إزالة `multiorder` support من الـ radix tree القديم

---

### نقاشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [PATCH v14 00/74: Convert page cache to XArray](https://lore.kernel.org/lkml/20180627110529.GA19606@bombadil.infradead.org/T/) | الـ patch series النهائي لتحويل الـ page cache |
| [PATCH v4 00/73: XArray version 4](https://lkml.kernel.org/lkml/20171206140648.GB32044@bombadil.infradead.org/T/) | نسخة مبكرة من الـ XArray — مفيد لفهم التطور |
| [PATCH v8 09/63: xarray: Add the xa_lock to the radix_tree_root](https://lkml.kernel.org/lkml/20180306192413.5499-10-willy@infradead.org/) | إضافة الـ locking للـ XArray |
| [Re: Endless calls to xas_split_alloc()](https://lore.kernel.org/lkml/ZRci1L6qneuZA4mo@casper.infradead.org/) | نقاش bug حقيقي مع Matthew Wilcox — مفيد لفهم الـ internals |

---

### عروض تقديمية

**الـ [Replacing the Radix Tree](https://lca-kernel.ozlabs.org/2018-Wilcox-Replacing-the-Radix-Tree.pdf)** — الـ PDF الأصلي لعرض Matthew Wilcox في linux.conf.au 2018. بيشرح:
- ليه الـ radix tree كان محتاج تغيير
- التصميم الداخلي للـ XArray
- الـ tagging و multiindex

---

### KernelNewbies

**الـ [KernelNewbies](https://kernelnewbies.org)** بيوثّق الـ XArray في سياق الـ kernel releases:

| الصفحة | الصلة بالـ XArray |
|--------|------------------|
| [InternalKernelDataTypes](https://kernelnewbies.org/InternalKernelDataTypes) | قائمة أنواع البيانات الداخلية في الـ kernel بما فيها XArray |
| [Linux_6.13](https://kernelnewbies.org/Linux_6.13) | تحويل delayed head refs وبنى perag لاستخدام XArray |
| [Linux_6.16](https://kernelnewbies.org/Linux_6.16) | إضافة Rust abstractions للـ XArray |
| [Linux_6.17](https://kernelnewbies.org/Linux_6.17) | تحسينات على XArray لـ extent buffers في btrfs |
| [MatthewWilcox](https://kernelnewbies.org/MatthewWilcox) | صفحة مؤلف الـ XArray |
| [KernelProjects/large-block-size](https://kernelnewbies.org/KernelProjects/large-block-size) | مشروع large block size الذي يعتمد على XArray بشكل أساسي |

---

### eLinux.org

**الـ elinux.org** ما عندوش صفحة مخصصة للـ XArray، لكن الصفحات دي ذات صلة:

- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — روابط لمصادر الـ kernel بشكل عام
- [Memory Management](https://elinux.org/Memory_Management) — الـ XArray جزء من منظومة الـ memory management
- [Kernel dynamic memory allocation tracking](https://elinux.org/Kernel_dynamic_memory_allocation_tracking_and_reduction) — سياق تخصيص الذاكرة

---

### كتب مُوصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الصلة:** الـ XArray ما موجودش في LDD3 (قديمة)، لكن الفصل الخاص بـ memory management وفصل الـ data structures مهمين للفهم الأساسي
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love
- **الإصدار:** الثالث (يغطي kernel 2.6)
- **الفصول ذات الصلة:**
  - Chapter 6: Kernel Data Structures — الـ linked lists والـ trees
  - Chapter 12: Memory Management — سياق الـ page cache
  - Chapter 15: The Process Address Space — أين يُستخدم XArray
- **الـ XArray** ما موجودش فيه مباشرة، لكنه الأساس النظري

#### Embedded Linux Primer — Christopher Hallinan
- **الفصول ذات الصلة:**
  - Chapter 14: Kernel Initialization — تسلسل التهيئة
  - الأجزاء المتعلقة بـ memory management في البيئات المضمّنة

#### مصدر إضافي مهم — Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي الـ radix tree بالتفصيل في الإصدارات القديمة — مفيد للمقارنة مع XArray

---

### مصطلحات البحث

لو محتاج تبحث أكتر، استخدم المصطلحات دي:

```
xarray linux kernel
xa_store xa_load xa_erase kernel API
xarray vs radix tree linux
xarray page cache linux
xa_state xas_load xas_store
DEFINE_XARRAY linux macro
xa_for_each iterator kernel
xarray multiindex CONFIG_XARRAY_MULTI
Matthew Wilcox xarray lwn
xarray locking spinlock kernel
xarray marks XA_MARK_0 XA_MARK_1
xarray RCU read side kernel
```

---

### ملخص سريع للمصادر حسب الأولوية

```
Priority 1 — ابدأ هنا:
  docs.kernel.org/core-api/xarray.html    ← التوثيق الرسمي
  lwn.net/Articles/715948/                ← المقال التعريفي
  lwn.net/Articles/745073/                ← العرض التقديمي

Priority 2 — للتعمق:
  lwn.net/Articles/757342/                ← قصة الـ merge
  lore.kernel.org patch v14               ← الـ patch series
  ozlabs PDF                              ← العرض التقديمي الكامل

Priority 3 — للمتخصصين:
  lib/xarray.c source code
  lkml mailing list threads
  kernelnewbies release pages
```
## Phase 8: Writing simple module

### الفكرة: Hook على `xa_store` باستخدام kprobe

**`xa_store`** هي الدالة المُصدَّرة الأكثر حيوية في الـ XArray API — كل مرة يُخزَّن فيها أي مؤشر في الـ array (page cache، idr، file descriptors، إلخ) تمر من هنا. نعمل عليها بـ **kprobe** عشان نشوف مين بيستخدمها ومتى.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_xastore.c
 * Hook xa_store() to log every XArray store operation kernel-wide.
 */

/* ── Includes ─────────────────────────────────────────── */
#include <linux/module.h>       /* module_init / module_exit / MODULE_* */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, … */
#include <linux/xarray.h>       /* struct xarray, xa_is_value, xa_to_value */
#include <linux/sched.h>        /* current, task_comm_len */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on xa_store() — trace XArray store operations");

/* ── kprobe definition ────────────────────────────────── */
static struct kprobe kp_xa_store;   /* one kprobe instance */

/*
 * pre_handler — runs just BEFORE xa_store() executes.
 *
 * Prototype of xa_store():
 *   void *xa_store(struct xarray *xa, unsigned long index,
 *                  void *entry, gfp_t gfp);
 *
 * On x86-64, args are in regs->di, regs->si, regs->dx, regs->cx.
 * On arm64,  args are in regs->regs[0..3].
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#ifdef CONFIG_X86_64
    struct xarray    *xa    = (struct xarray *)regs->di;   /* arg0 */
    unsigned long     index = (unsigned long)  regs->si;   /* arg1 */
    void             *entry = (void *)         regs->dx;   /* arg2 */
#elif defined(CONFIG_ARM64)
    struct xarray    *xa    = (struct xarray *)regs->regs[0];
    unsigned long     index = (unsigned long)  regs->regs[1];
    void             *entry = (void *)         regs->regs[2];
#else
    /* unsupported arch — skip */
    return 0;
#endif

    /* distinguish value-entries from pointer-entries */
    if (xa_is_value(entry)) {
        pr_info("kprobe/xa_store: comm=%-16s pid=%d  xa=%px  "
                "index=%-8lu  VALUE=%lu\n",
                current->comm, current->pid,
                xa, index,
                xa_to_value(entry));
    } else {
        pr_info("kprobe/xa_store: comm=%-16s pid=%d  xa=%px  "
                "index=%-8lu  PTR=%px\n",
                current->comm, current->pid,
                xa, index, entry);
    }

    return 0;   /* 0 = let xa_store() run normally */
}

/* ── module init ──────────────────────────────────────── */
static int __init kprobe_xastore_init(void)
{
    int ret;

    kp_xa_store.symbol_name = "xa_store";   /* target by name */
    kp_xa_store.pre_handler = handler_pre;  /* our callback */

    ret = register_kprobe(&kp_xa_store);
    if (ret < 0) {
        pr_err("kprobe_xastore: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_xastore: planted on xa_store @ %px\n",
            kp_xa_store.addr);
    return 0;
}

/* ── module exit ──────────────────────────────────────── */
static void __exit kprobe_xastore_exit(void)
{
    unregister_kprobe(&kp_xa_store);
    pr_info("kprobe_xastore: removed\n");
}

module_init(kprobe_xastore_init);
module_exit(kprobe_xastore_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_xastore.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
make
sudo insmod kprobe_xastore.ko
sudo dmesg | grep kprobe/xa_store   # شوف الـ output
sudo rmmod kprobe_xastore
```

---

### شرح كل جزء

#### `#include <linux/kprobes.h>`
ده الهيدر اللي بيجيب `struct kprobe` وكل functions التسجيل. من غيره مش هينفع نعمل hook على أي دالة kernel.

#### `#include <linux/xarray.h>`
محتاجينه عشان `xa_is_value()` و`xa_to_value()` — الـ inline helpers اللي بتفرق بين الـ value entries والـ pointer entries جوا الـ XArray.

#### `struct kprobe kp_xa_store`
الـ **kprobe** هو struct بسيط بيحتوي على اسم الدالة المستهدفة + مؤشر للـ callback. الـ kernel بيحط breakpoint افتراضي عند أول instruction في `xa_store` وبيشغّل الـ handler.

#### `handler_pre` — لماذا `pt_regs`؟
الـ kernel مش بيمررلنا الـ arguments مباشرةً — بيمررلنا `struct pt_regs` اللي بيحتوي على كل registers وقت الـ breakpoint. الـ calling convention على x86-64 بيحط الـ arguments في `rdi / rsi / rdx / rcx`، وعلى arm64 في `regs[0..3]`، فبنقرأهم مباشرةً من الـ registers.

#### `xa_is_value(entry)` / `xa_to_value(entry)`
الـ XArray بيخلّي أي odd pointer (bit0 = 1) يعمل كـ "value entry" بدل pointer. لو طبعناه كـ pointer هيكون بيانات غلط، فلازم نفحص الـ tag الأول ونحوله للرقم الحقيقي بـ `xa_to_value()`.

#### `return 0` في الـ handler
القيمة 0 تعني "استمر في تنفيذ الدالة الأصلية". لو رجّعنا 1 كان ممكن نتجاوز `xa_store` كاملةً — اللي مش عايزينه هنا.

#### `register_kprobe` في الـ init
بيحط الـ breakpoint الفعلي في الذاكرة ويحفظ الـ original bytes عشان يقدر يعمل restore لما نعمل unregister. لازم يتم في الـ init عشان الـ hook يكون فعّال من لحظة تحميل الـ module.

#### `unregister_kprobe` في الـ exit
**الأهم** — لو نسينا نعمل unregister قبل ما الـ module يتحمّل من الذاكرة، الـ handler_pre هيفضل pointer لكود اتمسح، وده بيسبب kernel panic فوري. الـ unregister بيشيل الـ breakpoint ويرجّع الـ instruction الأصلية.

---

### مثال على الـ output

```
kprobe_xastore: planted on xa_store @ ffffffff813a2c40
kprobe/xa_store: comm=kworker/0:1     pid=42    xa=ffff888003a1c000  index=0        PTR=ffff8880049b3000
kprobe/xa_store: comm=bash            pid=1337  xa=ffff888002f80000  index=3        PTR=ffff888004a12100
kprobe/xa_store: comm=chrome          pid=2048  xa=ffff888007c00000  index=12       VALUE=6
kprobe_xastore: removed
```

- السطر الأول: kworker بيخزن page في الـ page cache (pointer entry).
- السطر التاني: bash بيفتح file descriptor رقم 3 (pointer لـ `struct file`).
- السطر التالت: chrome بيخزن value integer في XArray خاص بيه.
