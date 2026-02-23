## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الملف `include/linux/generic-radix-tree.h` هو تعريف هيكل بيانات اسمه **Generic Radix Tree** (أو اختصارًا **genradix**)، وهو مصفوفة متفرقة (sparse array) مبنية على شجرة Radix، مُصمَّمة بشكل عام يخدم أي نوع بيانات.

الـ maintainer هو Kent Overstreet (صاحب bcache وbcachefs)، والـ subsystem هو **GENERIC RADIX TREE** — جزء من مكتبة الـ `lib/` في الـ kernel.

---

### القصة: ليش احنا محتاجين حاجة زي دي؟

تخيّل إنك بتشتغل على الـ kernel وعندك جدول ضخم جدًا — مثلًا جدول file descriptors لكل process، أو جدول inodes. المشكلة:

- لو عملت `array` عادية بحجم ثابت → هتحجز ذاكرة كتير حتى لو الـ entries فاضية.
- لو عملت `linked list` → الوصول بالـ index بطيء جدًا O(n).
- الـ **hash table** مش مرتّبة وصعبة الـ iteration.

الحل هو **Radix Tree**: شجرة كل node فيها array من pointers لـ nodes تانية، وفي الآخر الـ leaf nodes بتحتوي على البيانات الفعلية. النتيجة:

- وصول بالـ index بـ O(log n) قريب جدًا من O(1).
- بتحجز ذاكرة بس لما تحتاجها (sparse).
- سريع جدًا في الـ iteration.

الـ kernel عنده بالفعل `radix_tree` (في `lib/radix-tree.c`)، لكنه قديم ومعقد ومبني لـ `void *` pointers فقط. الـ **genradix** جاء كبديل **أبسط وأكثر type-safety** — بتعرّفه بنوع البيانات اللي هتخزنها مباشرة.

---

### تشبيه حياتي

تخيّل مكتبة فيها رفوف. كل رف بيتقسم لـ أقسام، وكل قسم بيتقسم لـ خانات. لو عايز الكتاب رقم 5000 — مش محتاج تمشي من أول كتاب للآخر، تروح مباشرة للرف الصح → القسم الصح → الخانة الصح. لو الخانة دي فاضية أصلًا، الرف ده مش موجود من الأساس — توفير في الذاكرة.

الـ genradix بيعمل نفس الكلام: شجرة من pages، كل page = node، الـ index بيترجم لـ byte offset بيقودك للـ node الصح.

---

### الفكرة التقنية ببساطة

```
genradix root
     │
     ▼
[genradix_root*]  ← pointer + depth مخبّيين في lower bits
     │
     ▼
[genradix_node]  ← interior node: array of 64 child pointers
   ├── children[0] → [genradix_node] ← leaf: 512 bytes of raw data
   ├── children[1] → [genradix_node] ← leaf: 512 bytes of raw data
   └── ...
```

- كل **node** = 512 bytes (`GENRADIX_NODE_SIZE = 1 << 9`).
- الـ **leaf nodes** بتخزن البيانات الفعلية كـ raw bytes.
- الـ **interior nodes** بتخزن pointers لـ nodes أعمق.
- الـ depth بيتخزن في الـ lower bits من الـ root pointer نفسه (pointer tagging).

---

### الـ Type Safety: الميزة الأساسية

```c
/* تعريف genradix من نوع struct foo */
static GENRADIX(struct foo) my_array;

/* الوصول للعنصر رقم 42 — بترجع struct foo* مباشرة */
struct foo *entry = genradix_ptr(&my_array, 42);

/* لو مش موجود، allocate */
entry = genradix_ptr_alloc(&my_array, 42, GFP_KERNEL);
```

الـ macro `GENRADIX(_type)` بيعمل struct جواه:
1. `struct __genradix tree` — الشجرة الفعلية.
2. `_type type[0]` — array بحجم صفر (بس لـ typeof trick).

بكده الـ compiler بيعرف نوع البيانات، والـ cast بيتعمل تلقائي.

---

### العمليات الأساسية

| Operation | Macro/Function | الوظيفة |
|---|---|---|
| `genradix_init` | macro | تهيئة شجرة فاضية |
| `genradix_free` | macro → `__genradix_free` | تحرير كل الذاكرة |
| `genradix_ptr` | macro → `__genradix_ptr` | قراءة entry بالـ index |
| `genradix_ptr_alloc` | macro → `__genradix_ptr_alloc` | قراءة أو allocate entry |
| `genradix_for_each` | macro | iteration من أول للآخر |
| `genradix_for_each_reverse` | macro | iteration عكسي |
| `genradix_prealloc` | macro → `__genradix_prealloc` | pre-allocate N entries |

---

### كيف بيتحول الـ index لـ byte offset

الدالة `__idx_to_offset` هي قلب الموضوع:

```c
/* لو حجم الـ object power of 2 */
offset = idx * obj_size;

/* لو مش power of 2 */
objs_per_page = GENRADIX_NODE_SIZE / obj_size;
offset = (idx / objs_per_page) * GENRADIX_NODE_SIZE
       + (idx % objs_per_page) * obj_size;
```

**الـ** `obj_size` لازم يكون أصغر من أو يساوي `GENRADIX_NODE_SIZE` (512 bytes) — لأن object واحد لازم يتسع في leaf node واحدة.

---

### الـ Lock-Free Design

الـ genradix مبني بـ **lock-free** approach للـ concurrent access:

- الـ `cmpxchg_release` في `__genradix_ptr_alloc` بيضمن إن التعديلات على الـ root آمنة.
- الـ `READ_ONCE` بيمنع الـ compiler من إعادة ترتيب الـ reads.
- ممكن تـ allocate من threads متعددة بدون explicit lock.

---

### من بيستخدم genradix في الـ kernel؟

| الملف | الاستخدام |
|---|---|
| `fs/proc/base.c` | جدول بيانات الـ process في procfs |
| `include/net/sctp/structs.h` | جدول الـ streams في بروتوكول SCTP |
| `fs/bcachefs/` (كتير) | قلب bcachefs filesystem |

---

### الملفات اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `include/linux/generic-radix-tree.h` | التعريفات والـ macros (الملف الحالي) |
| `lib/generic-radix-tree.c` | الـ implementation الفعلية |
| `include/linux/radix-tree.h` | الأخ الأكبر القديم (pointer-based) |
| `lib/radix-tree.c` | implementation الـ radix tree القديمة |
| `include/linux/idr.h` | IDR — استخدام آخر لـ radix tree للـ integer IDs |
| `lib/idr.c` | implementation الـ IDR |
## Phase 2: شرح الـ Generic Radix Tree Framework

### المشكلة — ليه الـ genradix موجود أصلاً؟

الـ kernel عنده `struct radix_tree_root` القديم (اللي اتحوّل بعدين لـ `xarray`) — بس ده مبني على تخزين **pointers**، مش بيانات مباشرة. يعني لو عندك array من `struct foo`، لازم تعمل `kmalloc` لكل `struct foo` على حدة وتخزن الـ pointer في الشجرة. النتيجة:

- **overhead** كبير في الـ allocation — كل عنصر allocate لوحده
- **cache misses** كتير لأن العناصر مش متجاورة في الميموري
- التعقيد في الكود — إدارة lifetimes لـ pointers متفرقة

المشكلة التانية: أحياناً بتحتاج **sparse array** — array بحجم ضخم جداً (مثلاً indexed by process ID أو inode number) لكن معظم الـ slots فاضية. Array عادية بتاكل ذاكرة بشكل مجنون. Linked list بطيئة في الـ random access.

**الـ genradix** جه يحل التلات مشاكل دول معاً: sparse array، type-safe، بيخزن البيانات بشكل مباشر (embedded، مش pointers).

---

### الحل — الـ approach اللي اتبعه الـ kernel

الـ genradix هو **radix tree of pages**. بدل ما يخزن pointers لـ objects منفصلة، كل leaf في الشجرة هو **page كاملة** بتحتوي على objects متجاورة. الـ index بيتحول لـ **byte offset** جوا الـ page المناسبة، وبعدين العنصر اتأخد مباشرة بدون أي pointer إضافي.

النقطة الذكية: الـ API بيستخدم **C macros + zero-size array trick** عشان يعمل type safety كاملة في compile time بدون أي runtime overhead. التفاصيل دي هنشرحها بعدين.

---

### التشبيه الواقعي — مكتبة الأرشيف

تخيل **مكتبة أرشيف ضخمة** فيها ملايين الوثائق، كل وثيقة رقمها بين 0 و ULONG_MAX.

| عنصر في التشبيه | المقابل في الـ genradix |
|---|---|
| رقم الوثيقة | الـ `idx` — index العنصر |
| الخزانة الكبيرة (الطابق) | `genradix_node` interior — بتقسّم المجال لأقسام |
| الدرج داخل الخزانة | `genradix_node` level أعمق |
| الصندوق الأخير | leaf node — page فيها البيانات الفعلية |
| الوثائق داخل الصندوق | objects من type `_type` متجاورة في الـ page |
| المكتبة فاضية في البداية | `root = NULL` |
| مفتش يطلب وثيقة غير موجودة | `genradix_ptr()` بيرجع `NULL` |
| مفتش يطلب وثيقة ويسجّل جديدة | `genradix_ptr_alloc()` بيخلق الـ nodes اللازمة |

**النقطة الأعمق**: الـ radix tree مش بيخزن كل وثيقة في خزانة منفصلة — ده هيبقى مكلف جداً. بدلاً من كده، الوثائق المتجاورة في الأرقام بتتحط جنب بعض في نفس الصندوق (الـ page). الصناديق بس اللي فيها وثائق فعلاً بتتخلق.

---

### المعمارية الكلية — Big Picture

```
Consumer Code (e.g., bcachefs, io_uring)
         |
         |  genradix_ptr(_radix, idx)
         |  genradix_ptr_alloc(_radix, idx, gfp)
         |  genradix_for_each(_radix, iter, p)
         v
+----------------------------------+
|        GENRADIX(_type) macro     |  <-- type-safe wrapper (compile-time only)
|   .tree  (struct __genradix)     |
|   .type[] (zero-size, for typeof)|
+----------------------------------+
         |
         |  &(_radix)->tree
         v
+----------------------------------+
|       struct __genradix          |
|   .root (struct genradix_root*)  |  <-- pointer + depth packed together
+----------------------------------+
         |
         |  genradix_root_to_node(r)
         |  genradix_root_to_depth(r)
         v
+---------------------------------------+
|         struct genradix_node          |
|  Interior:  children[GENRADIX_ARY]    |  <-- 512/8 = 64 child pointers
|    OR                                 |
|  Leaf:      data[GENRADIX_NODE_SIZE]  |  <-- 512 bytes of raw object data
+---------------------------------------+
         |
         |  tree traversal by bit-shifting offset
         v
+---------------------------------------+
|  Leaf Node (Page equivalent = 512B)   |
|  [ obj0 | obj1 | obj2 | ... | objN ]  |  <-- packed, zero-initialized
+---------------------------------------+
```

**ملاحظة مهمة**: `GENRADIX_NODE_SIZE = 512` بايت (مش صفحة 4KB كاملة). ده اختيار مقصود — أصغر من الـ page عشان يقلل الـ waste في الـ sparse use cases، ومع ذلك كافي لتجميع عدة objects متجاورة.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ genradix هو: type-safe sparse array بـ O(log N) access وـ zero fragmentation.**

الفكرة المحورية بتتجسد في ثلاث طبقات:

#### 1. الـ Type Safety Layer — الـ GENRADIX macro

```c
/*
 * GENRADIX creates an anonymous struct that embeds:
 *   - the actual tree (struct __genradix)
 *   - a zero-size array of _type just to carry type info at compile time
 */
#define GENRADIX(_type)         \
struct {                        \
    struct __genradix  tree;    \
    _type  type[0] __aligned(1);\  /* takes 0 bytes at runtime! */
}
```

الـ `type[0]` مش بياخد أي مساحة في الـ runtime — بس بيخلي الـ compiler يعرف الـ type. الـ macros بعدين بتعمل `typeof((_radix)->type[0])` عشان تعمل cast تلقائي. النتيجة: لو عملت `GENRADIX(struct foo)` وحاولت تخزن `struct bar`، الـ compiler بيطلع error.

#### 2. الـ Index-to-Offset Translation

```c
static inline size_t __idx_to_offset(size_t idx, size_t obj_size)
{
    if (!is_power_of_2(obj_size)) {
        /* Non-power-of-2 sizes: pack objects, align to page boundaries */
        size_t objs_per_page = GENRADIX_NODE_SIZE / obj_size;
        return (idx / objs_per_page) * GENRADIX_NODE_SIZE +
               (idx % objs_per_page) * obj_size;
    } else {
        /* Power-of-2: simple multiply */
        return idx * obj_size;
    }
}
```

الـ `idx` (رقم العنصر) بيتحول لـ `offset` (byte offset في الشجرة). الـ tree نفسها مش تعرف شيء عن الـ types — بس بتعرف `offset`. ده الـ separation of concerns: الـ type-aware code كله في الـ macros، والـ tree نفسها type-agnostic.

#### 3. الـ Root Encoding — Depth Packed in Pointer

```c
/*
 * The depth of the tree is packed into the low bits of the root pointer.
 * This avoids a separate field — saves memory and is cache-friendly.
 */
static inline unsigned genradix_root_to_depth(struct genradix_root *r)
{
    return (unsigned long) r & GENRADIX_DEPTH_MASK;
}

static inline struct genradix_node *genradix_root_to_node(struct genradix_root *r)
{
    return (void *) ((unsigned long) r & ~GENRADIX_DEPTH_MASK);
}
```

الـ `genradix_root` مش struct حقيقي — هو مجرد pointer تم packing فيه الـ depth في الـ low bits. ده يشتغل لأن الـ `genradix_node` ميموري-مالigned على `GENRADIX_NODE_SIZE` (512 بايت)، يعني الـ low 9 bits دايماً صفر في الـ pointer الحقيقي.

---

### بنية الـ Structs — علاقتها ببعض

```
GENRADIX(struct task_entry) my_array;
│
├── struct __genradix  tree
│   └── struct genradix_root *root   ← packed: [node_ptr | depth]
│                                               bits [63:9]   bits [2:0]
│
└── struct task_entry  type[0]       ← zero bytes, compile-time type tag


struct genradix_root *root
        │
        ├── depth = 0: root IS the leaf → direct page of data
        │
        └── depth > 0: root is interior node
                │
                └── struct genradix_node (interior)
                    └── children[64]  (pointers to next level)
                            │
                            └── struct genradix_node (interior or leaf)
                                    │
                                    └── struct genradix_node (leaf)
                                        └── data[512]
                                            ├── [0..sizeof(T)-1]   → entry 0
                                            ├── [sizeof(T)..2*sizeof(T)-1] → entry 1
                                            └── ...


struct genradix_iter {
    size_t offset;   ← byte offset in tree (used internally for traversal)
    size_t pos;      ← logical index (what the consumer sees)
};
```

---

### حساب الـ Depth — كيف تكفي الشجرة لـ ULONG_MAX؟

```
GENRADIX_NODE_SIZE  = 512 = 2^9  → GENRADIX_NODE_SHIFT = 9
GENRADIX_ARY        = 512 / 8    = 64 (على 64-bit) → GENRADIX_ARY_SHIFT = 6

Depth 0: covers 2^9  = 512 bytes
Depth 1: covers 2^(9+6)  = 2^15 = 32KB
Depth 2: covers 2^(9+12) = 2^21 = 2MB
Depth 3: covers 2^(9+18) = 2^27 = 128MB
...
Depth 9: covers 2^(9+54) = 2^63 → كافي لـ ULONG_MAX على 64-bit
```

الـ `GENRADIX_MAX_DEPTH = DIV_ROUND_UP(BITS_PER_LONG - 9, 6)` — على 64-bit ده يساوي `DIV_ROUND_UP(55, 6) = 10`.

---

### الـ Traversal Algorithm — رحلة الـ lookup

```c
static inline void *__genradix_ptr_inlined(struct __genradix *radix, size_t offset)
{
    struct genradix_root *r = READ_ONCE(radix->root);
    struct genradix_node *n = genradix_root_to_node(r);
    unsigned level  = genradix_root_to_depth(r);
    unsigned shift  = genradix_depth_shift(level); /* = 9 + 6*level */

    /* offset أكبر من اللي الشجرة بتغطيه؟ */
    if (unlikely(ilog2(offset) >= genradix_depth_shift(level)))
        return NULL;

    /* traverse interior nodes */
    while (n && shift > GENRADIX_NODE_SHIFT) {
        shift -= GENRADIX_ARY_SHIFT;           /* نزل level */
        n = n->children[offset >> shift];      /* اختار الـ child بناءً على bits */
        offset &= (1UL << shift) - 1;          /* mask الـ bits المستخدمة */
    }

    /* n هو الـ leaf الآن، offset هو byte offset داخله */
    return n ? &n->data[offset] : NULL;
}
```

**مثال عملي**: لو عندنا `GENRADIX(u32)` وبنعمل `genradix_ptr(&arr, 200)`:
- `obj_size = 4` → `offset = 200 * 4 = 800`
- `800 >= 512` يعني محتاجين depth ≥ 1
- `shift = 9 + 6 = 15`
- `children[800 >> 9] = children[1]` → الـ child رقم 1
- `offset &= (1<<9)-1 = 800 & 511 = 288`
- وصلنا للـ leaf → `&leaf->data[288]`

---

### الـ Iterator — كيف بيشتغل `genradix_for_each`

```c
/*
 * Typical usage in a driver:
 */
GENRADIX(struct my_obj) objects;

struct genradix_iter iter;
struct my_obj *obj;

genradix_for_each(&objects, iter, obj) {
    /* obj points directly into the tree's leaf page */
    /* iter.pos is the logical index */
    process(obj, iter.pos);
}
```

الـ `genradix_iter_peek` بيدور على أول slot موجود (مش NULL) عند أو بعد الـ current offset. ده مهم في الـ sparse arrays — مش بيلف على كل الـ holes.

الـ `__genradix_iter_advance` بتاخد بالها من حالة خاصة: لو الـ `obj_size` مش power of 2، ممكن آخر عنصر في الـ page يبقى جزء منه في الـ page الجاية. في الحالة دي، بيعمل `round_up` للـ offset للـ page boundary التالية.

---

### الـ Preallocation — استخدام في الكود الحساس

```c
/* Preallocate 1024 entries without blocking later */
ret = genradix_prealloc(&my_array, 1024, GFP_KERNEL);
if (ret)
    return ret;

/* Now access in atomic context — won't allocate */
obj = genradix_ptr_alloc_preallocated_inlined(&my_array, idx, &new_node, GFP_NOWAIT);
```

الـ `genradix_prealloc` بيخلق كل الـ nodes اللازمة مسبقاً. ده مفيد في contexts إزاي interrupt handlers أو spinlocks حيث الـ allocation مش مسموح أو غير مضمون.

---

### الـ Ownership — إيه اللي الـ genradix بيعمله وإيه اللي بيسيبه

| الـ genradix بيملك | الـ genradix بيفوّض للـ consumer |
|---|---|
| بنية الشجرة والـ nodes | تحديد الـ type المخزون |
| إدارة الـ pages (alloc/free) | معنى الـ index (هو رقم pid؟ inode؟ fd؟) |
| الـ traversal algorithm | الـ synchronization (مفيش locking داخلي) |
| الـ byte-offset calculations | الـ lifecycle للـ objects (construction/destruction) |
| الـ depth management | تحديد `gfp_t` flags |

**مهم جداً**: الـ genradix **لا يوفر أي locking**. لو أكتر من thread بيكتب/يقرأ في نفس الوقت، الـ consumer مسؤول عن الـ synchronization. ده بيخليه خفيف وبسيط، لكن محتاج وعي من اللي بيستخدمه.

---

### استخدام حقيقي — bcachefs

الـ bcachefs filesystem بيستخدم الـ genradix لتخزين الـ journal entries والـ btree nodes في الـ memory:

```c
/* من كود bcachefs (مبسّط) */
static GENRADIX(struct btree_key_cache_freelist) key_cache_freelist;

/* الـ index هو رقم الـ CPU أو sequence number */
struct btree_key_cache_freelist *fl =
    genradix_ptr_alloc(&key_cache_freelist, cpu, GFP_KERNEL);
```

الـ bcachefs اختار الـ genradix بالذات لأنه:
1. بياخد بيانات بشكل مباشر (لا extra allocations)
2. بيدعم iteration بشكل كفء
3. الـ sparse nature مناسبة لأن مش كل الـ indices هتُستخدم

---

### الـ Subsystem اللي محتاج تعرفه قبل كده

- **Slab Allocator** (`linux/slab.h`): الـ genradix بيستخدم `kzalloc`/`kfree` لإدارة الـ nodes. فهم الـ `gfp_t` flags (متى `GFP_KERNEL` ومتى `GFP_ATOMIC`) ضروري عشان تستخدم الـ genradix صح.
- **Memory Model / READ_ONCE**: الـ `__genradix_ptr_inlined` بيستخدم `READ_ONCE` على الـ root — ده يحتاج فهم أساسي للـ memory ordering في الـ kernel.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ Constants والـ Macros — Cheatsheet

| الاسم | القيمة | الوظيفة |
|---|---|---|
| `GENRADIX_NODE_SHIFT` | `9` | كل node بتحتوي على `2^9 = 512` byte |
| `GENRADIX_NODE_SIZE` | `512` byte | حجم كل node (page-aligned) |
| `GENRADIX_ARY` | `512 / sizeof(ptr)` | عدد الـ children في الـ interior node (عادةً 64 على 64-bit) |
| `GENRADIX_ARY_SHIFT` | `ilog2(GENRADIX_ARY)` | الـ shift المقابل لعدد الـ children (عادةً 6) |
| `GENRADIX_MAX_DEPTH` | `DIV_ROUND_UP(BITS_PER_LONG - 9, ARY_SHIFT)` | أقصى عمق للشجرة عشان تغطي كل `ULONG_MAX` |
| `GENRADIX_DEPTH_MASK` | `roundup_pow_of_two(MAX_DEPTH+1) - 1` | الـ bits الـ low في الـ root pointer اللي بتخزن الـ depth |

---

### الـ Structs الأساسية

#### 1. `struct genradix_node`

**الهدف:** اللبنة الأساسية للشجرة — ممكن تكون interior node أو leaf node.

```c
struct genradix_node {
    union {
        /* Interior node: pointers to children */
        struct genradix_node *children[GENRADIX_ARY];

        /* Leaf node: raw data bytes */
        u8 data[GENRADIX_NODE_SIZE];
    };
};
```

| الحالة | المحتوى | الوظيفة |
|---|---|---|
| **Interior node** | `children[]` — array of pointers | بتوجّه للـ nodes التانية في مستوى أعمق |
| **Leaf node** | `data[]` — raw bytes | بتخزن الـ objects الفعلية اللي المستخدم عايزها |

- الاتنين في `union` — نفس الحجم (`GENRADIX_NODE_SIZE = 512` byte).
- الـ leaf هو الـ node اللي على المستوى الأدنى (level 0)، وبيخزن الـ data مباشرةً.
- الـ interior بيخزن فقط pointers للـ children في مستوى أعمق.

---

#### 2. `struct genradix_root` (opaque)

**الهدف:** pointer مخفي داخله معلومتين في نفس الوقت.

```c
struct genradix_root; /* forward declaration only — never defined in header */
```

الـ trick هنا إن الـ pointer نفسه بيخزن:
- **الـ node pointer** في الـ high bits (بعد mask)
- **الـ depth** في الـ low bits (عن طريق `GENRADIX_DEPTH_MASK`)

```c
/* extract depth from root pointer */
static inline unsigned genradix_root_to_depth(struct genradix_root *r)
{
    return (unsigned long) r & GENRADIX_DEPTH_MASK;
}

/* extract actual node pointer from root pointer */
static inline struct genradix_node *genradix_root_to_node(struct genradix_root *r)
{
    return (void *) ((unsigned long) r & ~GENRADIX_DEPTH_MASK);
}
```

الـ depth والـ pointer محشورين في قيمة واحدة لأن الـ node pointers دايمًا aligned، فالـ low bits مضمون تكون صفر وممكن نستغلها.

---

#### 3. `struct __genradix`

**الهدف:** الـ internal tree handle — بتمسك بس pointer واحد للـ root.

```c
struct __genradix {
    struct genradix_root *root;
};
```

| الحقل | النوع | الوظيفة |
|---|---|---|
| `root` | `struct genradix_root *` | pointer للـ root node مع الـ depth مخزن في الـ low bits |

- الـ `root == NULL` معناه شجرة فاضية.
- كل الـ public APIs بتأخذ `struct __genradix *` كـ internal parameter.

---

#### 4. `GENRADIX(_type)` — الـ Type-Safe Wrapper

**الهدف:** macro بيولّد struct generic بيخزن الـ type info بدون أي memory overhead.

```c
#define GENRADIX(_type)         \
struct {                        \
    struct __genradix tree;     \
    _type type[0] __aligned(1); \
}
```

| الحقل | النوع | الوظيفة |
|---|---|---|
| `tree` | `struct __genradix` | الشجرة الفعلية |
| `type[0]` | `_type[]` | zero-size array — بس لاستخدام `typeof()` و `sizeof()` في الـ macros |

الـ `type[0]` مش بياخد أي مساحة في الـ runtime، بس بيخلي المترجم يعرف نوع الـ data المخزنة فيقدر يعمل type checking صح.

---

#### 5. `struct genradix_iter`

**الهدف:** بيتتبع الـ position أثناء الـ iteration.

```c
struct genradix_iter {
    size_t offset; /* byte offset in the tree */
    size_t pos;    /* logical index (entry number) */
};
```

| الحقل | الوظيفة |
|---|---|
| `offset` | الـ byte offset في الشجرة — ده اللي بيتستخدم في traversal الفعلي |
| `pos` | الـ logical index (رقم العنصر من 0) — ده اللي بيشوفه المستخدم |

---

### علاقات الـ Structs — ASCII Diagram

```
GENRADIX(struct foo)
┌─────────────────────────────┐
│  struct __genradix tree     │
│  ┌──────────────────────┐   │
│  │ struct genradix_root │   │
│  │  *root               │──────────────────────────────┐
│  └──────────────────────┘   │                          │
│  struct foo type[0]         │  (zero-size, type only)  │
└─────────────────────────────┘                          │
                                                         │
                               packed pointer: [node_ptr | depth]
                                                         │
                                          ┌──────────────▼──────────────┐
                                          │   struct genradix_node      │
                                          │   (interior node, depth > 0)│
                                          │  children[0] ──────────────►│ genradix_node (level-1)
                                          │  children[1] ──────────────►│ genradix_node (level-1)
                                          │  ...                        │
                                          │  children[N-1]              │
                                          └─────────────────────────────┘
                                                    │
                                                    ▼
                                          ┌─────────────────────────────┐
                                          │   struct genradix_node      │
                                          │   (leaf node, depth == 0)   │
                                          │  data[0..511]               │
                                          │  ┌────┬────┬────┬────┐      │
                                          │  │ e0 │ e1 │ e2 │... │      │
                                          │  └────┴────┴────┴────┘      │
                                          └─────────────────────────────┘
```

---

### كيف بيشتغل الـ Indexing

الـ `__idx_to_offset` بتحوّل الـ logical index لـ byte offset:

```c
static inline size_t __idx_to_offset(size_t idx, size_t obj_size)
{
    if (!is_power_of_2(obj_size)) {
        /* pack objects tightly per page, skip padding between pages */
        size_t objs_per_page = GENRADIX_NODE_SIZE / obj_size;
        return (idx / objs_per_page) * GENRADIX_NODE_SIZE
             + (idx % objs_per_page) * obj_size;
    } else {
        /* simple multiply for power-of-2 sizes */
        return idx * obj_size;
    }
}
```

**ليه الفرق؟**
- لو `obj_size` power of 2 → الـ objects مش بتبقى بتتقسم على حدود الـ page، فالضرب بسيط.
- لو مش power of 2 → ممكن object يتقسم على page boundary فنتجنب ده بـ explicit page-aligned packing.

---

### Lifecycle Diagram

```
Initialization
──────────────
genradix_init(&radix)
  └─► radix.tree.root = NULL
      (شجرة فاضية، مفيش allocation)

First Access (ptr_alloc)
────────────────────────
genradix_ptr_alloc(&radix, idx, gfp)
  └─► __genradix_idx_to_offset(idx)
        └─► __genradix_ptr_alloc(&tree, offset, NULL, gfp)
              ├─ لو الـ root فاضي: alloc root node، set depth=0
              ├─ لو الـ offset أكبر من capacity الحالية:
              │     grow tree: alloc new root، link old root كـ child
              │     زود الـ depth بـ 1
              └─ traverse حتى الـ leaf، alloc missing nodes
                   └─► رجّع pointer للـ data في الـ leaf

Read Access
────────────
genradix_ptr(&radix, idx)
  └─► __genradix_ptr(&tree, offset)
        └─► __genradix_ptr_inlined
              ├─ لو offset خارج الـ tree depth: return NULL
              └─ traverse levels حتى الـ leaf
                   └─► return &leaf->data[offset & mask]

Iteration
──────────
genradix_for_each(&radix, iter, p)
  └─► iter = genradix_iter_init(&radix, 0)
        └─► loop:
              p = genradix_iter_peek(&iter, &radix)
              (لو NULL: خلص)
              ... use p ...
              genradix_iter_advance(&iter, &radix)

Teardown
─────────
genradix_free(&radix)
  └─► __genradix_free(&tree)
        └─► traverse كل الشجرة
              └─► free كل node
                   └─► tree.root = NULL  (reinitialize)
```

---

### Call Flow Diagrams

#### `genradix_ptr_alloc` — الـ write path

```
user calls genradix_ptr_alloc(&radix, idx, gfp)
  │
  ├─► __genradix_idx_to_offset(idx, sizeof(type))
  │     └─► byte_offset
  │
  └─► __genradix_ptr_alloc(&tree, byte_offset, NULL, gfp)
        │
        ├─► READ_ONCE(tree->root) → r
        │
        ├─► [tree empty?] → kzalloc(GENRADIX_NODE_SIZE, gfp) → new root node
        │     cmpxchg(&tree->root, NULL, encode(node, depth=0))
        │
        ├─► [offset too large for current depth?]
        │     └─► alloc new_root, set new_root->children[0] = old_root
        │           cmpxchg root → encode(new_root, depth+1)
        │           retry
        │
        ├─► level = depth, shift = genradix_depth_shift(level)
        │
        └─► while shift > GENRADIX_NODE_SHIFT:
              │  shift -= GENRADIX_ARY_SHIFT
              │  child_idx = byte_offset >> shift
              │  byte_offset &= (1<<shift)-1
              │  if node->children[child_idx] == NULL:
              │    kzalloc(GENRADIX_NODE_SIZE, gfp) → new child
              │    cmpxchg(&node->children[child_idx], NULL, new_child)
              └─► node = node->children[child_idx]
                    └─► return &node->data[byte_offset]
```

#### `genradix_ptr` — الـ read path (inlined fast path)

```
user calls genradix_ptr(&radix, idx)
  │
  └─► __genradix_ptr_inlined(&tree, offset)
        │
        ├─► r = READ_ONCE(tree->root)
        ├─► n = genradix_root_to_node(r)   [strip depth bits]
        ├─► level = genradix_root_to_depth(r)
        ├─► shift = genradix_depth_shift(level)
        │
        ├─► [ilog2(offset) >= shift?] → return NULL  (out of range)
        │
        └─► while n && shift > GENRADIX_NODE_SHIFT:
              shift -= GENRADIX_ARY_SHIFT
              n = n->children[offset >> shift]
              offset &= (1<<shift)-1
                └─► return n ? &n->data[offset] : NULL
```

#### `genradix_iter_advance` — تقدّم الـ iterator

```
__genradix_iter_advance(iter, obj_size)
  │
  ├─► [overflow check: iter->offset + obj_size wraps?]
  │     └─► iter->offset = iter->pos = SIZE_MAX  (end sentinel)
  │
  ├─► iter->offset += obj_size
  │
  ├─► [obj_size not power-of-2 AND would cross page boundary?]
  │     └─► round_up(iter->offset, GENRADIX_NODE_SIZE)
  │           (تجاوز الـ padding في نهاية الـ page)
  │
  └─► iter->pos++
```

---

### الـ Tree Growth — كيف الشجرة بتكبر

```
Initial (depth=0):
  root ──► [leaf: data 0..511]

After growing (depth=1):
  root ──► [interior: children[0..63]]
                │
                children[0] ──► [leaf: data 0..511]
                children[1] ──► [leaf: data 512..1023]
                ...

After growing (depth=2):
  root ──► [interior L2: children[0..63]]
                │
                children[0] ──► [interior L1: children[0..63]]
                                      │
                                      children[0] ──► [leaf]
                                      children[1] ──► [leaf]
                                      ...
```

كل مرة الـ tree بتكبر، الـ root القديم بيبقى `children[0]` للـ root الجديد، وبيتزاد الـ depth بـ 1.

---

### الـ Locking Strategy

الـ `generic-radix-tree` **مش بيوفر locks داخلية** — ده قرار تصميمي واضح:

| الحالة | الميكانيزم المستخدم |
|---|---|
| **Read path** (`genradix_ptr`) | `READ_ONCE(tree->root)` — safe للـ concurrent reads بدون lock |
| **Write path** (`ptr_alloc`) | `cmpxchg` للـ atomic node insertion — lock-free للـ concurrent writers |
| **Tree growth** (depth change) | `cmpxchg` على الـ root pointer لضمان atomicity |
| **Iteration** | مفيش synchronization — المستخدم مسؤول عن الـ external locking لو في writers |
| **Free** (`genradix_free`) | المستخدم لازم يضمن مفيش concurrent access |

**ملاحظات عملية:**

- الـ `READ_ONCE` بيضمن إن الـ compiler ميعملش optimization غلط للـ concurrent reads.
- الـ `cmpxchg` بيضمن إن لو اتنين threads حاولوا يعملوا alloc لنفس الـ slot في نفس الوقت، واحد بس يكسب والتاني يعمل `kfree` للـ node اللي allocate-ه وبعدين يستخدم الـ winner.
- الـ `genradix_for_each` في الـ presence of concurrent writes هو **racy** — المستخدم محتاج external lock لو الـ use case بيتطلب ده.

**مثال على الـ safe usage:**

```c
/* writer side — needs external lock if concurrent */
spin_lock(&my_lock);
entry = genradix_ptr_alloc(&my_radix, idx, GFP_ATOMIC);
if (entry)
    *entry = value;
spin_unlock(&my_lock);

/* reader side — safe for concurrent reads only */
entry = genradix_ptr(&my_radix, idx); /* READ_ONCE internally */
if (entry)
    use(*entry);
```
## Phase 4: شرح الـ Functions

---

### Cheatsheet — كل الـ Functions والـ Macros دفعة واحدة

#### تصنيف: Initialization & Cleanup

| Function / Macro | النوع | الغرض |
|---|---|---|
| `genradix_init(_radix)` | macro | تهيئة genradix فارغ |
| `DEFINE_GENRADIX(_name, _type)` | macro | تعريف + تهيئة static |
| `genradix_free(_radix)` | macro → `__genradix_free` | تحرير كل الـ nodes وإعادة التهيئة |
| `genradix_alloc_node(gfp)` | inline | allocate node واحد (512 bytes) |
| `genradix_free_node(node)` | inline | تحرير node واحد |

#### تصنيف: Lookup & Allocation

| Function / Macro | النوع | الغرض |
|---|---|---|
| `genradix_ptr(_radix, _idx)` | macro → `__genradix_ptr` | جيب pointer للـ entry، `NULL` لو مش موجود |
| `genradix_ptr_inlined(_radix, _idx)` | macro → `__genradix_ptr_inlined` | نفسه لكن inlined للـ hot paths |
| `genradix_ptr_alloc(_radix, _idx, _gfp)` | macro → `__genradix_ptr_alloc` | جيب pointer مع alloc لو مش موجود |
| `genradix_ptr_alloc_inlined(...)` | macro | نفسه مع fast-path inline check أول |
| `genradix_ptr_alloc_preallocated(...)` | macro → `__genradix_ptr_alloc` | alloc مع preallocated node |
| `genradix_ptr_alloc_preallocated_inlined(...)` | macro | نفسه مع inline fast-path |
| `genradix_prealloc(_radix, _nr, _gfp)` | macro → `__genradix_prealloc` | preallocate عدد من الـ entries |

#### تصنيف: Iteration

| Function / Macro | النوع | الغرض |
|---|---|---|
| `genradix_iter_init(_radix, _idx)` | macro | ابدأ iterator من index معين |
| `genradix_iter_peek(_iter, _radix)` | macro → `__genradix_iter_peek` | جيب أول entry موجود >= position |
| `genradix_iter_peek_prev(_iter, _radix)` | macro → `__genradix_iter_peek_prev` | جيب أول entry موجود <= position |
| `genradix_iter_advance(_iter, _radix)` | macro → `__genradix_iter_advance` | قدّم الـ iterator خطوة للأمام |
| `genradix_iter_rewind(_iter, _radix)` | macro → `__genradix_iter_rewind` | رجّع الـ iterator خطوة للخلف |
| `genradix_for_each(_radix, _iter, _p)` | macro | iterate forward على كل الـ entries |
| `genradix_for_each_from(_radix, _iter, _p, _start)` | macro | iterate forward ابتداءً من index |
| `genradix_for_each_reverse(_radix, _iter, _p)` | macro | iterate backward على كل الـ entries |

#### تصنيف: Internal Helpers

| Function / Macro | النوع | الغرض |
|---|---|---|
| `genradix_depth_shift(depth)` | inline | احسب الـ bit shift لـ depth معين |
| `genradix_depth_size(depth)` | inline | احسب حجم البيانات اللي تقدر tree بهذا الـ depth تحمله |
| `genradix_root_to_depth(r)` | inline | استخرج الـ depth من الـ root pointer |
| `genradix_root_to_node(r)` | inline | استخرج الـ node pointer من الـ root |
| `__idx_to_offset(idx, obj_size)` | inline | حوّل index لـ byte offset |

---

### Group 1: Initialization & Definition

الـ genradix مش بيتحتاج heap allocation عشان يتعرّف — مجرد pointer واحد `root = NULL` يكفي. الـ macros دي بتضمن إن أي استخدام قبل أي alloc هو safe ومحدد سلوكه.

---

#### `GENRADIX(_type)`

```c
#define GENRADIX(_type)         \
struct {                        \
    struct __genradix tree;     \
    _type type[0] __aligned(1); \
}
```

بيعرّف anonymous struct بتحتوي على الـ tree الفعلية وـarray بحجم 0 من النوع المطلوب. الـ `type[0]` ده zero-cost trick: مبيأخدش space في الـ runtime لكن بيخلي الـ `typeof((_radix)->type[0])` متاح لكل الـ accessor macros عشان تعمل cast صح وتحسب `sizeof` صح. الـ `__aligned(1)` بيمنع إن alignment الـ type العجيب يأثر على حجم الـ struct.

---

#### `DEFINE_GENRADIX(_name, _type)`

```c
#define DEFINE_GENRADIX(_name, _type) \
    GENRADIX(_type) _name = __GENRADIX_INITIALIZER
```

بيعرّف متغير static/global من النوع المطلوب ومهيأ فوراً بـ `root = NULL`. مثال: `static DEFINE_GENRADIX(idr_table, struct idr_layer);`

---

#### `genradix_init(_radix)`

```c
#define genradix_init(_radix) \
do { \
    *(_radix) = (typeof(*_radix)) __GENRADIX_INITIALIZER; \
} while (0)
```

**ما بيفشلش أبداً.** بيعمل zero-init للـ `root` pointer. مفيد لما الـ genradix embedded في struct تانية ومحتاج تهيئة في runtime مش compile-time.

- **Parameters:** `_radix` — pointer للـ genradix المراد تهيئته.
- **Return:** void.
- **Locking:** لا يحتاج lock لأن السياق هو initialization — single-threaded context دايماً.

---

#### `genradix_alloc_node(gfp_mask)`

```c
static inline struct genradix_node *genradix_alloc_node(gfp_t gfp_mask)
{
    return kzalloc(GENRADIX_NODE_SIZE, gfp_mask);
}
```

بيعمل allocate لـ node واحد بحجم 512 bytes (`GENRADIX_NODE_SIZE`) ويعمله zero. الـ `kzalloc` بيضمن إن كل الـ children pointers تبقى `NULL` وكل الـ data bytes تبقى صفر — ده مهم جداً لأن الـ API بيضمن إن entries جديدة initialized to zero.

- **Parameters:** `gfp_mask` — الـ GFP flags (مثلاً `GFP_KERNEL`, `GFP_ATOMIC` في interrupt context).
- **Return:** pointer للـ node الجديد، أو `NULL` لو الـ allocation فشل.
- **Caller:** `__genradix_ptr_alloc` بشكل أساسي.

---

#### `genradix_free_node(node)`

```c
static inline void genradix_free_node(struct genradix_node *node)
{
    kfree(node);
}
```

thin wrapper على `kfree`. بيتعمل call منه أثناء tree teardown.

---

#### `__genradix_free(radix)` / `genradix_free(_radix)`

```c
void __genradix_free(struct __genradix *);

#define genradix_free(_radix) __genradix_free(&(_radix)->tree)
```

بيمشي على كل الـ tree nodes بشكل recursive (أو iterative) ويحررهم، وبعدين بيعمل `root = NULL` عشان الـ genradix يبقى valid وجاهز للاستخدام تاني.

- **Parameters:** `radix` — pointer لـ `__genradix` الداخلي.
- **Return:** void.
- **Side effects:** كل الـ memory اللي الـ tree بتملكها بتتحرر. الـ entries اللي المستخدم عنده pointers ليها بتبقى dangling — مسؤولية الـ caller.
- **Locking:** الـ caller مسؤول عن الـ external locking. مفيش internal lock.

---

### Group 2: Internal Helpers — الـ Tree Navigation

دي الـ building blocks اللي كل الـ API بيتعمد عليها. بتتعامل مع encoding الـ depth في الـ root pointer وحساب الـ offsets.

---

#### `genradix_root_to_depth(r)`

```c
static inline unsigned genradix_root_to_depth(struct genradix_root *r)
{
    return (unsigned long) r & GENRADIX_DEPTH_MASK;
}
```

الـ root pointer مش pointer عادي — الـ bits الـ low بتحمل الـ depth. ده classic kernel trick بيستغل إن الـ aligned pointers بيكون فيها low bits = 0. الـ `GENRADIX_DEPTH_MASK` بيعزل الـ depth من الـ pointer.

- **Parameters:** `r` — الـ root pointer المشفّر.
- **Return:** depth الـ tree الحالية.

---

#### `genradix_root_to_node(r)`

```c
static inline struct genradix_node *genradix_root_to_node(struct genradix_root *r)
{
    return (void *) ((unsigned long) r & ~GENRADIX_DEPTH_MASK);
}
```

بيعمل mask على الـ bits الـ low ويطلع الـ actual pointer للـ root node. لو `r == NULL` فالنتيجة `NULL` — valid للـ empty tree.

---

#### `genradix_depth_shift(depth)`

```c
static inline int genradix_depth_shift(unsigned depth)
{
    return GENRADIX_NODE_SHIFT + GENRADIX_ARY_SHIFT * depth;
}
```

الـ `GENRADIX_NODE_SHIFT = 9` (512 bytes per node). الـ `GENRADIX_ARY_SHIFT = ilog2(GENRADIX_ARY)` وهو عدد الـ bits اللي كل level بيستهلكها من الـ offset. الـ result هو: عند depth معينة، كام bit offset تقدر الـ tree تعنونه.

---

#### `genradix_depth_size(depth)`

```c
static inline size_t genradix_depth_size(unsigned depth)
{
    return 1UL << genradix_depth_shift(depth);
}
```

بيحسب عدد الـ bytes اللي tree بهذا الـ depth تقدر تعنونها. مثلاً depth=0 → 512 bytes، depth=1 → 512 * 64 = 32KB وهكذا.

---

#### `__idx_to_offset(idx, obj_size)`

```c
static inline size_t __idx_to_offset(size_t idx, size_t obj_size)
{
    if (__builtin_constant_p(obj_size))
        BUILD_BUG_ON(obj_size > GENRADIX_NODE_SIZE);
    else
        BUG_ON(obj_size > GENRADIX_NODE_SIZE);

    if (!is_power_of_2(obj_size)) {
        size_t objs_per_page = GENRADIX_NODE_SIZE / obj_size;
        return (idx / objs_per_page) * GENRADIX_NODE_SIZE +
               (idx % objs_per_page) * obj_size;
    } else {
        return idx * obj_size;
    }
}
```

ده الـ heart of the indexing system. الـ genradix الداخلي بيشتغل بـ **byte offsets** مش indices. الـ function دي بتحوّل من index منطقي لـ byte offset في الـ tree.

**ليه التعقيد؟** لو `obj_size` مش power of two (مثلاً struct بحجم 24 bytes):
- الـ packing بيتم على أساس الـ page: كل page 512 bytes تحمل `512/24 = 21` object (وبيبقى 8 bytes waste).
- الـ formula بتحسب: في أنهي page وفين في الـ page.

لو `obj_size` power of two: الـ math بتبقى `idx * obj_size` مباشرة بدون overhead.

**الـ BUG_ON:** object واحد لازم يتسع في page واحدة — هذا constraint أساسي في الـ design.

```
مثال: obj_size = 3, objs_per_page = 170 (512/3)
  idx=0   → offset=0
  idx=169 → offset=507
  idx=170 → offset=512   (page جديدة، مش 510!)
  idx=171 → offset=515
```

- **Parameters:**
  - `idx` — الـ logical index.
  - `obj_size` — `sizeof` الـ stored type.
- **Return:** byte offset داخل الـ tree.

---

### Group 3: Lookup Functions

---

#### `__genradix_ptr_inlined(radix, offset)`

```c
static inline void *__genradix_ptr_inlined(struct __genradix *radix, size_t offset)
{
    struct genradix_root *r = READ_ONCE(radix->root);
    struct genradix_node *n = genradix_root_to_node(r);
    unsigned level          = genradix_root_to_depth(r);
    unsigned shift          = genradix_depth_shift(level);

    if (unlikely(ilog2(offset) >= genradix_depth_shift(level)))
        return NULL;

    while (n && shift > GENRADIX_NODE_SHIFT) {
        shift -= GENRADIX_ARY_SHIFT;
        n = n->children[offset >> shift];
        offset &= (1UL << shift) - 1;
    }

    return n ? &n->data[offset] : NULL;
}
```

ده الـ hot-path lookup المهيأ للـ inlining. بيستخدم `READ_ONCE` للـ lockless read من الـ root — يعني آمن للاستخدام مع الـ RCU-style locking لو المستخدم بيمسك الـ read lock.

**Pseudocode flow:**
```
r = READ_ONCE(root)           // read root atomically
n = extract node from r
level = extract depth from r
shift = depth_shift(level)

if offset is out of range → return NULL

while n != NULL and still traversing interior nodes:
    shift -= ARY_SHIFT
    n = n->children[offset >> shift]   // go down one level
    offset &= (1 << shift) - 1        // strip the used bits

return n ? &n->data[offset] : NULL    // leaf data or NULL
```

- **Parameters:**
  - `radix` — الـ `__genradix` الداخلي.
  - `offset` — byte offset (من `__idx_to_offset`).
- **Return:** pointer للـ entry، `NULL` لو مش موجود.
- **Locking:** الـ `READ_ONCE` بيحمي من الـ compiler reordering. الـ caller مسؤول عن الـ memory barrier اللازم (RCU read lock أو mutex).
- **Performance:** fully inlined، بتيجي في path العمليات المتكررة.

---

#### `genradix_ptr(_radix, _idx)` / `__genradix_ptr`

```c
#define genradix_ptr(_radix, _idx)                      \
    (__genradix_cast(_radix)                            \
     __genradix_ptr(&(_radix)->tree,                    \
                    __genradix_idx_to_offset(_radix, _idx)))

void *__genradix_ptr(struct __genradix *, size_t);
```

الـ public API للـ read-only lookup. بيستخدم الـ out-of-line `__genradix_ptr` (في `.c` file) بدل الـ inlined version — اختيار للـ code size في الـ non-critical paths.

الـ macro بيعمل cast للنتيجة للـ type الصح تلقائياً بفضل `__genradix_cast(_radix)` اللي بيعمل `(typeof((_radix)->type[0]) *)`.

- **Parameters:**
  - `_radix` — الـ genradix.
  - `_idx` — الـ logical index.
- **Return:** `(T *)` pointer للـ entry، أو `NULL` لو مش موجود.
- **Who calls it:** أي كود بيحتاج يقرأ entry موجودة بدون ما يعمل alloc.

---

#### `genradix_ptr_inlined(_radix, _idx)`

```c
#define genradix_ptr_inlined(_radix, _idx)              \
    (__genradix_cast(_radix)                            \
     __genradix_ptr_inlined(&(_radix)->tree,            \
                    __genradix_idx_to_offset(_radix, _idx)))
```

نفس `genradix_ptr` لكن بيـforce الـ inlining. مفيد في hot paths زي الـ PID lookup أو أي data structure بتتعمل عليها lookups متكررة جداً في critical sections.

---

### Group 4: Allocation Functions

---

#### `__genradix_ptr_alloc(radix, offset, new_node, gfp)`

```c
void *__genradix_ptr_alloc(struct __genradix *, size_t,
                            struct genradix_node **, gfp_t);
```

ده الـ core allocation engine. بيمشي على الـ tree path من الـ root للـ leaf، وبيـalloc أي node ناقص في الطريق. الـ `new_node` parameter هو optimization: لو المستخدم preallocated node قبل ما يمسك الـ lock، بيمرره هنا عشان يتجنب alloc تاني.

**Pseudocode flow:**
```
retry:
    r = READ_ONCE(root)

    while tree is too shallow to hold offset:
        allocate new root node (or use *new_node)
        try cmpxchg(root, old_root, new_encoded_root)
        if failed → retry (another CPU grew the tree)

    traverse from root to leaf:
        for each interior level:
            if child is NULL:
                allocate new child node
                try cmpxchg(child_ptr, NULL, new_node)
                if failed → free new_node, retry traverse

    return &leaf->data[final_offset]
```

- **Parameters:**
  - `radix` — الـ tree.
  - `offset` — byte offset.
  - `new_node` — pointer لـ pointer لـ preallocated node (أو `NULL`).
  - `gfp` — flags للـ allocation.
- **Return:** pointer للـ entry (يضمن مش `NULL`)، أو `NULL` لو فضل `GFP_NOWAIT`/`GFP_ATOMIC` والـ alloc فشل.
- **Locking:** يستخدم `cmpxchg` للـ lockless concurrent growth — atomic لكن مش lock-free بالمعنى الـ wait-free. ممكن يـretry.
- **Error path:** لو `kzalloc` بتفشل، بترجع `NULL` مباشرة.

---

#### `genradix_ptr_alloc(_radix, _idx, _gfp)`

```c
#define genradix_ptr_alloc(_radix, _idx, _gfp)         \
    (__genradix_cast(_radix)                            \
     __genradix_ptr_alloc(&(_radix)->tree,              \
                    __genradix_idx_to_offset(_radix, _idx), \
                    NULL, _gfp))
```

الـ standard allocation API. بيمرر `NULL` كـ `new_node` يعني `__genradix_ptr_alloc` هو المسؤول عن كل الـ allocations.

- **Return:** `(T *)` pointer للـ entry (مضمون initialized to zero لو جديد)، أو `NULL` لو alloc فشل.
- **Who calls it:** كود بيحتاج guaranteed slot، مثلاً allocation contexts زي `idr_alloc`.

---

#### `genradix_ptr_alloc_inlined(_radix, _idx, _gfp)`

```c
#define genradix_ptr_alloc_inlined(_radix, _idx, _gfp)             \
    (__genradix_cast(_radix)                                        \
     (__genradix_ptr_inlined(&(_radix)->tree,                       \
             __genradix_idx_to_offset(_radix, _idx)) ?:            \
      __genradix_ptr_alloc(&(_radix)->tree,                         \
             __genradix_idx_to_offset(_radix, _idx),               \
             NULL, _gfp)))
```

بيعمل inline fast-path أول: لو الـ entry موجودة أصلاً يرجعها فوراً بدون function call. لو مش موجودة يعمل fallback للـ `__genradix_ptr_alloc`. ده pattern مهم للـ read-mostly workloads.

---

#### `genradix_ptr_alloc_preallocated(_radix, _idx, _new_node, _gfp)`

```c
#define genradix_ptr_alloc_preallocated(_radix, _idx, _new_node, _gfp) \
    (__genradix_cast(_radix)                                            \
     __genradix_ptr_alloc(&(_radix)->tree,                              \
                    __genradix_idx_to_offset(_radix, _idx),            \
                    _new_node, _gfp))
```

بيمرر `_new_node` كـ preallocated node. مفيد لما الـ caller بيمسك lock ومش عايز يعمل alloc جوا الـ lock. الـ pattern هو:

```c
struct genradix_node *node = genradix_alloc_node(GFP_KERNEL); /* outside lock */
spin_lock(&my_lock);
ptr = genradix_ptr_alloc_preallocated(&tree, idx, &node, GFP_NOWAIT);
spin_unlock(&my_lock);
if (node)
    genradix_free_node(node); /* wasn't used */
```

---

#### `genradix_prealloc(_radix, _nr, _gfp)` / `__genradix_prealloc`

```c
int __genradix_prealloc(struct __genradix *, size_t, gfp_t);

#define genradix_prealloc(_radix, _nr, _gfp)           \
     __genradix_prealloc(&(_radix)->tree,               \
                    __genradix_idx_to_offset(_radix, _nr + 1), \
                    _gfp)
```

بيضمن إن كل الـ nodes اللازمة لعنونة من 0 لـ `_nr` موجودة مسبقاً. مفيد في initialization paths لما عايز تضمن مفيش allocation failure بعدين.

- **Parameters:**
  - `_nr` — عدد الـ entries المراد preallocate ليها.
  - `_gfp` — GFP flags.
- **Return:** `0` في النجاح، `-ENOMEM` في الفشل.
- **Note:** بيعمل `_nr + 1` في الـ offset calculation عشان يضمن الـ page اللي بتحتوي الـ element الأخير متحجزة كلها.

---

### Group 5: Iterator Functions

الـ iteration model في الـ genradix مبني على `struct genradix_iter` اللي بتحمل الـ position بطريقتين: `pos` (logical index) و`offset` (byte offset في الـ tree).

```c
struct genradix_iter {
    size_t offset;  /* byte offset in the tree */
    size_t pos;     /* logical index */
};
```

---

#### `genradix_iter_init(_radix, _idx)`

```c
#define genradix_iter_init(_radix, _idx)                \
    ((struct genradix_iter) {                           \
        .pos    = (_idx),                               \
        .offset = __genradix_idx_to_offset((_radix), (_idx)), \
    })
```

بيعمل initialize لـ iterator من index معين. الـ compound literal بيخلي الـ init مباشرة في الـ for loop initialization expression.

---

#### `__genradix_iter_peek(iter, radix, objs_per_page)` / `genradix_iter_peek`

```c
void *__genradix_iter_peek(struct genradix_iter *, struct __genradix *, size_t);

#define genradix_iter_peek(_iter, _radix)               \
    (__genradix_cast(_radix)                            \
     __genradix_iter_peek(_iter, &(_radix)->tree,       \
                    __genradix_objs_per_page(_radix)))
```

الـ function دي مش بس بتجيب الـ entry الحالية — بتـskip الـ missing pages. لو الـ page اللي بتحتوي الـ current offset مش موجودة، بتقفز للـ page التالية الموجودة وبتحدّث الـ `iter->offset` و`iter->pos`. ده مهم للـ sparse arrays.

- **Parameters:**
  - `iter` — الـ iterator، بيتعدّل في المكان (out param لـ offset/pos update).
  - `radix` — الـ tree.
  - `objs_per_page` — عدد الـ objects في كل page (للـ skip calculation).
- **Return:** pointer للـ entry، أو `NULL` لو مفيش entries تانية.
- **Side effect:** بيعدّل `iter->offset` و`iter->pos` لو عمل skip.

---

#### `__genradix_iter_peek_prev(iter, radix, objs_per_page, obj_size_plus_rem)` / `genradix_iter_peek_prev`

```c
void *__genradix_iter_peek_prev(struct genradix_iter *, struct __genradix *,
                                 size_t, size_t);

#define genradix_iter_peek_prev(_iter, _radix)                      \
    (__genradix_cast(_radix)                                        \
     __genradix_iter_peek_prev(_iter, &(_radix)->tree,              \
                    __genradix_objs_per_page(_radix),               \
                    __genradix_obj_size(_radix) +                   \
                    __genradix_page_remainder(_radix)))
```

النسخة العكسية من `iter_peek` — بتـsearch عن أقرب موجود في الاتجاه العكسي. الـ `obj_size + page_remainder` parameter بيتعمل استخدامه لتخطي الـ padding بين الـ pages في الـ non-power-of-2 case.

---

#### `__genradix_iter_advance(iter, obj_size)` / `genradix_iter_advance`

```c
static inline void __genradix_iter_advance(struct genradix_iter *iter,
                                            size_t obj_size)
{
    if (iter->offset + obj_size < iter->offset) {
        iter->offset = SIZE_MAX;
        iter->pos    = SIZE_MAX;
        return;
    }

    iter->offset += obj_size;

    if (!is_power_of_2(obj_size) &&
        (iter->offset & (GENRADIX_NODE_SIZE - 1)) + obj_size > GENRADIX_NODE_SIZE)
        iter->offset = round_up(iter->offset, GENRADIX_NODE_SIZE);

    iter->pos++;
}
```

بيقدّم الـ iterator خطوة. ثلاث cases:

1. **Overflow check:** لو الجمع فاض → `SIZE_MAX` كـ sentinel للنهاية.
2. **Non-power-of-2:** لو الـ object التالي مش هيتسع في الـ page الحالية → اقفز لأول الـ page التالية (الـ padding bytes بتتـskip).
3. **Power-of-2:** جمع بسيط.

- **Parameters:**
  - `iter` — الـ iterator (modified in-place).
  - `obj_size` — `sizeof` الـ stored type.

---

#### `__genradix_iter_rewind(iter, obj_size)` / `genradix_iter_rewind`

```c
static inline void __genradix_iter_rewind(struct genradix_iter *iter,
                                           size_t obj_size)
{
    if (iter->offset == 0 || iter->offset == SIZE_MAX) {
        iter->offset = SIZE_MAX;
        return;
    }

    if ((iter->offset & (GENRADIX_NODE_SIZE - 1)) == 0)
        iter->offset -= GENRADIX_NODE_SIZE % obj_size;

    iter->offset -= obj_size;
    iter->pos--;
}
```

بيرجّع الـ iterator خطوة. لو الـ current offset عند بداية page ومش power-of-2 type، لازم يـsubtract الـ padding الإضافي أول عشان يوصل لآخر valid object في الـ page السابقة.

- **Parameters:** نفس `iter_advance`.
- **Return:** void.
- **Edge cases:** `offset == 0` أو `offset == SIZE_MAX` → يعمل set للـ `SIZE_MAX` كـ sentinel للانتهاء.

---

### Group 6: Iteration Macros (for loops)

---

#### `genradix_for_each_from(_radix, _iter, _p, _start)`

```c
#define genradix_for_each_from(_radix, _iter, _p, _start)   \
    for (_iter = genradix_iter_init(_radix, _start);         \
         (_p = genradix_iter_peek(&_iter, _radix)) != NULL;  \
         genradix_iter_advance(&_iter, _radix))
```

الـ base iteration macro. `_iter.pos` بيكون الـ current index في كل iteration. مثال:

```c
DEFINE_GENRADIX(pid_table, struct pid);
struct genradix_iter iter;
struct pid *p;

/* iterate all PIDs from 100 onwards */
genradix_for_each_from(&pid_table, iter, p, 100) {
    printk("pid %zu: comm=%s\n", iter.pos, p->comm);
}
```

---

#### `genradix_for_each(_radix, _iter, _p)`

```c
#define genradix_for_each(_radix, _iter, _p) \
    genradix_for_each_from(_radix, _iter, _p, 0)
```

يبدأ من الـ index 0. الأكثر استخداماً للـ full scan.

---

#### `genradix_for_each_reverse(_radix, _iter, _p)`

```c
#define genradix_last_pos(_radix) \
    (SIZE_MAX / GENRADIX_NODE_SIZE * __genradix_objs_per_page(_radix) - 1)

#define genradix_for_each_reverse(_radix, _iter, _p)            \
    for (_iter = genradix_iter_init(_radix, genradix_last_pos(_radix)); \
         (_p = genradix_iter_peek_prev(&_iter, _radix)) != NULL; \
         genradix_iter_rewind(&_iter, _radix))
```

يبدأ من أقصى valid position ممكنة (`genradix_last_pos`) ويمشي للخلف. الـ `genradix_last_pos` بيحسب أكبر valid index قبل ما الـ byte offset يفيض من `SIZE_MAX`.

---

### ملاحظات على الـ Type Safety

كل الـ public macros بتعمل cast تلقائي للـ `(T *)` الصح:

```c
/* Internal helper macros: */
#define __genradix_cast(_radix)       (typeof((_radix)->type[0]) *)
#define __genradix_obj_size(_radix)   sizeof((_radix)->type[0])
```

ده بيعني:
- `genradix_ptr` بترجع `struct foo *` مش `void *`، فالـ compiler بيقدر يـtype-check.
- `sizeof` بيتحسب compile-time من الـ type مش runtime.
- أي `genradix_for_each` الـ pointer `_p` بيكون typed صح.

---

### ملاحظات على الـ Concurrency

| Operation | Concurrency Safety |
|---|---|
| `genradix_ptr` / `genradix_ptr_inlined` | Safe مع RCU read lock — بيستخدم `READ_ONCE` |
| `genradix_ptr_alloc` | Thread-safe بفضل `cmpxchg` على الـ node pointers |
| `genradix_free` | مش thread-safe — يحتاج exclusive lock |
| `genradix_for_each` | يحتاج external locking — الـ iterator مش atomic |

الـ `cmpxchg`-based growth في `__genradix_ptr_alloc` بيعني إن concurrent writers ممكن يـcontend على نفس الـ node pointer لكن مفيش corruption. الـ loser بيـfree الـ node اللي allocateه وبيستخدم الـ winner's node.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs

الـ **generic radix tree** ما عندوش entries مخصوصة في debugfs بذاتها — هو pure library code. بس ممكن تضيف entries يدوي في الـ subsystem اللي بيستخدمه (زي bcachefs).

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# bcachefs بيستخدم genradix ويعمل entries خاصة بيه
ls /sys/kernel/debug/bcachefs/

# قراءة entry خاص بـ genradix stats لو الـ subsystem عمله
cat /sys/kernel/debug/<subsystem>/genradix_stats
```

لإضافة debugfs entry يدوي لـ genradix في الكود:

```c
#include <linux/debugfs.h>
#include <linux/seq_file.h>
#include <linux/generic-radix-tree.h>

static int genradix_debug_show(struct seq_file *s, void *v)
{
    struct __genradix *radix = s->private;
    struct genradix_root *r = READ_ONCE(radix->root);
    unsigned depth = genradix_root_to_depth(r);
    struct genradix_node *node = genradix_root_to_node(r);

    seq_printf(s, "depth:     %u\n", depth);
    seq_printf(s, "root_node: %px\n", node);
    /* max bytes addressable at this depth */
    seq_printf(s, "max_size:  %zu bytes\n",
               depth ? genradix_depth_size(depth) : 0);
    return 0;
}
DEFINE_SHOW_ATTRIBUTE(genradix_debug);

/* في init الـ subsystem */
debugfs_create_file("genradix_info", 0444, dbg_dir,
                    &my_radix.tree, &genradix_debug_fops);
```

---

#### 2. sysfs

الـ genradix نفسه ما عندوش sysfs interface. بس لأنه بيستخدم `kzalloc(GENRADIX_NODE_SIZE, ...)` — وحجم كل node = 512 bytes — ممكن تتابع الـ SLUB cache اللي بيخدمه:

```bash
# stats لـ kmalloc-512 (GENRADIX_NODE_SIZE = 512 bytes)
cat /sys/kernel/slab/kmalloc-512/alloc_calls
cat /sys/kernel/slab/kmalloc-512/free_calls
cat /sys/kernel/slab/kmalloc-512/objects
cat /sys/kernel/slab/kmalloc-512/objects_partial

# لو الـ objects بتكبر مع الوقت بدون ما تنزل = memory leak في genradix
watch -n1 'cat /sys/kernel/slab/kmalloc-512/objects'
```

---

#### 3. ftrace — Tracepoints والـ Events

الـ genradix ما عندوش tracepoints خاصة بيه. الطريقة المجدية هي tracing الـ `kmalloc` / `kfree` مع filter على حجم الـ node (512 bytes):

```bash
cd /sys/kernel/debug/tracing

# صفّر الـ trace
echo 0 > tracing_on
echo > trace

# فلتر على allocations بحجم 512 بالظبط
echo 'bytes_alloc == 512' > events/kmem/kmalloc/filter
echo 1 > events/kmem/kmalloc/enable
echo 1 > events/kmem/kfree/enable
echo 1 > tracing_on

# شغّل الـ workload، ثم اقرأ
echo 0 > tracing_on
cat trace | grep genradix | head -30
```

مثال output:

```
  kworker/0:1-23  [000] ....  456.123: kmalloc: call_site=__genradix_ptr_alloc+0x4c/0x120
                                                 ptr=0xffff888012340000
                                                 bytes_req=512 bytes_alloc=512
                                                 gfp_flags=GFP_KERNEL
```

**التفسير:**
- `call_site=__genradix_ptr_alloc+0x4c` → الـ allocation جت من genradix
- `bytes_alloc=512` → تم حجز node جديد (GENRADIX_NODE_SIZE)
- `gfp_flags=GFP_KERNEL` → allocation عادية آمنة في process context

لو عايز تتبع الـ function calls مباشرة:

```bash
cd /sys/kernel/debug/tracing
echo '__genradix_ptr_alloc' > set_ftrace_filter
echo '__genradix_ptr'      >> set_ftrace_filter
echo '__genradix_free'     >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
```

---

#### 4. printk و Dynamic Debug

الـ `lib/generic-radix-tree.c` ما بيستخدم `pr_debug` كتير — لو محتاج debug messages ضيفها بنفسك في patch مؤقت:

```bash
# تفعيل dynamic debug للملف
echo 'file lib/generic-radix-tree.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تحقق إن التفعيل نجح
grep 'generic-radix' /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ slab allocator
echo 'module slab_common +p' > /sys/kernel/debug/dynamic_debug/control
```

إضافة `pr_debug` في patch مؤقت داخل `lib/generic-radix-tree.c`:

```c
/* قبل kzalloc في genradix_alloc_node */
pr_debug("genradix: alloc node gfp=%x caller=%pS\n",
         gfp_mask, __builtin_return_address(0));

/* في __genradix_ptr_alloc بعد cmpxchg */
pr_debug("genradix: new root depth=%u node=%px\n",
         new_depth, new_node);
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_SLUB_DEBUG` | يفعّل red zones + poison + alloc/free tracking في SLUB |
| `CONFIG_KASAN` | **Kernel Address Sanitizer** — يكشف out-of-bounds وuse-after-free |
| `CONFIG_KASAN_GENERIC` | KASAN لـ x86/arm64 (software-based) |
| `CONFIG_KASAN_SW_TAGS` | KASAN لـ arm64 باستخدام pointer tagging |
| `CONFIG_KFENCE` | **Kernel Electric Fence** — sampling-based، overhead أقل من KASAN |
| `CONFIG_DEBUG_KMEMLEAK` | يكشف memory leaks في الـ kernel heap |
| `CONFIG_DEBUG_PAGEALLOC` | guard pages بعد كل page — يكشف out-of-bounds على page level |
| `CONFIG_FAILSLAB` | fault injection في الـ slab allocator (يفشّل `kzalloc` بنسبة محددة) |
| `CONFIG_DEBUG_OBJECTS` | يتبع lifetime الـ kernel objects |
| `CONFIG_LOCKDEP` | يكشف deadlocks و lock ordering violations |
| `CONFIG_DEBUG_SPINLOCK` | يكشف spinlock misuse (double lock, unlock without lock) |
| `CONFIG_UBSAN` | يكشف undefined behavior (integer overflow, etc.) |

```bash
# تحقق من الـ configs الحالية
grep -E 'SLUB_DEBUG|KASAN|KFENCE|KMEMLEAK|FAILSLAB|LOCKDEP' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

الـ genradix بيعتمد على SLUB كـ backend، فأدوات SLUB هي الأنسب:

```bash
# عرض كل slab caches مرتبة بالحجم
sudo slabinfo -s 2>/dev/null || cat /proc/slabinfo | sort -k3 -rn | head -20

# فلتر على kmalloc-512
cat /proc/slabinfo | grep kmalloc-512

# كشف memory leaks بعد unload module
echo scan > /sys/kernel/debug/kmemleak
sleep 5
cat /sys/kernel/debug/kmemleak | grep -A10 'genradix\|generic-radix'

# fault injection في الـ slab (يحتاج CONFIG_FAILSLAB=y)
echo 5   > /sys/kernel/debug/failslab/probability   # 5% فشل
echo 100 > /sys/kernel/debug/failslab/interval
echo -1  > /sys/kernel/debug/failslab/times          # للأبد
echo 1   > /sys/kernel/debug/failslab/task-filter
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `BUG: unable to handle kernel NULL pointer dereference` في `__genradix_ptr_inlined` | الـ `root` بـ NULL — أو access على genradix غير initialized | تأكد من استدعاء `genradix_init()` قبل أي `genradix_ptr()` |
| `kernel BUG at include/linux/generic-radix-tree.h:163!` | `obj_size > GENRADIX_NODE_SIZE` — الـ BUG_ON في `__idx_to_offset` | حجم الـ struct أكبر من 512 bytes — صغّره أو استخدم pointer بدل inline data |
| `general protection fault` في `genradix_root_to_node` | الـ root pointer اتعمل corrupt | concurrent access بدون لازم — راجع الـ locking، أو memory corruption من KASAN |
| `KASAN: slab-out-of-bounds` في `__genradix_iter_advance` أو `__genradix_ptr_inlined` | iterator تعدّى حدود الـ node | تحقق من `obj_size` وحساب الـ offset — وأنه `is_power_of_2` أو بيتسع كامل في page |
| `-ENOMEM` من `genradix_ptr_alloc` أو `genradix_prealloc` | `kzalloc` فشلت — نفد الـ memory | راجع الـ GFP flags (استخدم `GFP_KERNEL` مش `GFP_ATOMIC` قدر الإمكان)، وراجع memory pressure |
| `kmemleak: unreferenced object 0xffff...` بـ backtrace يشير لـ `__genradix_ptr_alloc` | genradix اتحذف بدون `genradix_free()` | أضف `genradix_free()` في cleanup path (مثال: في `.remove` callback أو module exit) |
| `WARNING: CPU: X PID: Y` مع `WARN_ON` يشير لـ `generic-radix-tree.c` | `obj_size` غير صحيح أو tree في حالة غير متسقة | راجع الـ `sizeof` اللي اتبعت لـ `GENRADIX()` — وتأكد ما فيش data corruption |
| `BUG: scheduling while atomic` أثناء `genradix_ptr_alloc` | استدعاء بـ `GFP_KERNEL` في atomic context | استبدل بـ `GFP_ATOMIC` أو استخدم preallocated variant مع `genradix_ptr_alloc_preallocated` |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* نقطة 1: في __idx_to_offset — قبل BUG_ON الموجود أصلاً */
static inline size_t __idx_to_offset(size_t idx, size_t obj_size)
{
    /* الـ BUG_ON الموجود يكفي — بس ممكن نضيف log قبله */
    if (unlikely(obj_size > GENRADIX_NODE_SIZE)) {
        pr_err("genradix: obj_size=%zu exceeds node size %u\n",
               obj_size, GENRADIX_NODE_SIZE);
        dump_stack();
    }
    BUG_ON(obj_size > GENRADIX_NODE_SIZE);
    /* ... */
}

/* نقطة 2: في __genradix_ptr_alloc بعد kzalloc فشلت */
struct genradix_node *n = kzalloc(GENRADIX_NODE_SIZE, gfp_mask);
if (!n) {
    WARN_ONCE(1, "genradix: node allocation failed at offset=%zu\n",
              offset);
    /* dump_stack() هنا لو GFP_ATOMIC وعايز تعرف مين طالب */
    dump_stack();
    return NULL;
}

/* نقطة 3: في __genradix_free — تحقق من double free */
void __genradix_free(struct __genradix *radix)
{
    WARN_ON(!radix);
    /* بعد التحرير، تأكد إن الـ root اتصفّر */
    genradix_free_recurse(radix->root);
    WRITE_ONCE(radix->root, NULL);
    WARN_ON(radix->root != NULL);  /* لو cmpxchg فيه bug */
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ genradix هو pure software data structure — ما بيتكلمش مع hardware مباشرة. بس لو بيخزن بيانات تمثّل hardware state (زي IRQ descriptors، DMA entries، أو IOMMU mappings):

```bash
# مثال: لو genradix بيخزن IRQ descriptors
# عدد الـ IRQs عند الـ hardware
cat /proc/interrupts | tail -n +2 | wc -l

# عدد الـ allocated IRQs في الـ kernel
cat /proc/sys/kernel/nr_irqs

# مثال: لو بيخزن DMA channels
cat /sys/kernel/debug/dmaengine/summary | grep -c 'channel'

# مثال: لو bcachefs بيستخدمه لـ inodes
cat /sys/kernel/debug/bcachefs/*/journal | grep 'nr_inodes'
```

---

#### 2. Register Dump Techniques

الـ genradix نفسه ما بيتعامل مع hardware registers. لو الـ subsystem اللي فوقيه محتاج register dump:

```bash
# قراءة physical memory address (يحتاج root + CONFIG_STRICT_DEVMEM=n)
sudo apt-get install -y devmem2   # لو مش مثبت
sudo devmem2 0xFEDC0000 w         # قراءة 32-bit word

# عبر /dev/mem
sudo dd if=/dev/mem bs=4 count=16 skip=$((0xFEDC0000/4)) 2>/dev/null | xxd

# عبر io utility (x86 فقط — لـ I/O ports)
sudo apt-get install -y ioport
sudo io -4 0x3F8   # مثال: UART base

# dump كامل للـ BAR0 لـ PCIe device
# أوجد الـ base address من lspci
sudo lspci -v | grep -A10 'Memory at'
# ثم
sudo devmem2 <BAR0_ADDR> w
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ genradix هو software فقط — ما بيولّد signals على الـ bus. بس لو بتـ debug driver يستخدمه مع hardware:

- **I2C/SPI**: اربط logic analyzer على SCL/SDA أو CLK/MOSI. قارن timestamp الـ transactions بوقت `__genradix_ptr_alloc` في الـ ftrace — لو التأخير غير متوقع يبقى الـ allocation هو السبب.
- **PCIe**: استخدم PCIe protocol analyzer — تحقق إن الـ DMA descriptor اللي اتخزن في الـ genradix وصل للـ device صح.
- **GPIO toggle للقياس الدقيق**: أضف GPIO toggling حوالين `genradix_ptr_alloc` لقياس وقت الـ allocation بالـ oscilloscope:

```c
#include <linux/gpio.h>

/* قبل الـ allocation */
gpio_set_value(DEBUG_GPIO, 1);
ptr = genradix_ptr_alloc(&my_radix, idx, GFP_KERNEL);
/* بعد الـ allocation */
gpio_set_value(DEBUG_GPIO, 0);
/* الـ pulse width = وقت الـ allocation */
```

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ Log |
|---|---|
| نفاد الـ DRAM أو ECC error أثناء node allocation | `EDAC MC0: CE memory read error` متبوع بـ crash في `kzalloc` |
| NUMA misalignment — allocation من wrong NUMA node | `page allocation failure` + latency spikes في الـ subsystem |
| Memory pressure من hardware DMA يستهلك الـ memory | `kswapd0: page allocation failure` قبل `-ENOMEM` من `genradix_ptr_alloc` |
| Bit flip في الـ root pointer (no ECC RAM) | `general protection fault` عشوائي في `genradix_root_to_node` |
| Stack overflow بسبب deep recursion في `genradix_free` | `BUG: stack guard page was hit` عند تحرير شجرة ضخمة |

```bash
# تحقق من ECC errors
edac-util -s 0 2>/dev/null
# أو مباشرة من sysfs
cat /sys/bus/platform/drivers/sb_edac/*/mc/mc0/ce_count 2>/dev/null

# تحقق من NUMA topology والـ free memory لكل node
numactl --hardware
cat /sys/devices/system/node/node*/meminfo | grep MemFree

# راقب الـ memory pressure
vmstat 1 5
cat /proc/meminfo | grep -E 'MemAvailable|Slab|KReclaimable'
```

---

#### 5. Device Tree Debugging

الـ genradix ما بيقرأ من الـ DT مباشرة. بس لو الـ driver اللي بيستخدم الـ genradix بيقرأ resources من الـ DT (مثال: DMA channels، IRQs):

```bash
# تحقق من الـ DT compiled
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A10 'your-device'

# قارن عدد الـ resources في الـ DT بعدد entries في الـ genradix
# مثال: عدد DMA channels في الـ DT
grep -c 'dma-channels' /sys/firmware/devicetree/base/dma-controller*/

# تحقق من الـ compatible string
cat /sys/bus/platform/devices/<your-device>/of_node/compatible

# فحص الـ reg property (base address + size)
hexdump -C /sys/bus/platform/devices/<your-device>/of_node/reg

# شوف الـ DT overlay المحمّل
ls /sys/kernel/config/device-tree/overlays/ 2>/dev/null
```

مثال: لو الـ genradix بيخزن descriptors لـ DMA channels مقروية من الـ DT:

```bash
# عدد الـ DMA channels في الـ DT
CHANNELS=$(grep -c 'dma-channels' /sys/firmware/devicetree/base/dma-controller*/ 2>/dev/null)
echo "DT channels: $CHANNELS"
# قارن بالـ entries في الـ genradix عبر debugfs أو kmemleak output
```

---

### Practical Commands

---

#### 1. Script كامل لتفعيل tracing الـ genradix allocations

```bash
#!/bin/bash
# genradix_trace.sh — full tracing for genradix node allocations

TRACE=/sys/kernel/debug/tracing

echo "Setting up genradix allocation trace..."

# صفّر
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# فلتر على allocations بحجم 512 (GENRADIX_NODE_SIZE)
echo 'bytes_alloc == 512' > $TRACE/events/kmem/kmalloc/filter
echo 1 > $TRACE/events/kmem/kmalloc/enable
echo 1 > $TRACE/events/kmem/kfree/enable

# function tracing للـ genradix functions
echo '__genradix_ptr_alloc'  > $TRACE/set_ftrace_filter
echo '__genradix_ptr'       >> $TRACE/set_ftrace_filter
echo '__genradix_free'      >> $TRACE/set_ftrace_filter
echo '__genradix_prealloc'  >> $TRACE/set_ftrace_filter
echo function > $TRACE/current_tracer

echo 1 > $TRACE/tracing_on
echo "Tracing for 30 seconds..."
sleep 30

echo 0 > $TRACE/tracing_on
cp $TRACE/trace /tmp/genradix_trace_$(date +%s).txt
echo "Saved to /tmp/genradix_trace_*.txt"
```

```bash
chmod +x genradix_trace.sh && sudo bash genradix_trace.sh
```

**مثال output وتفسيره:**

```
# --- trace output ---
  worker-42   [001] ....  789.123: __genradix_ptr_alloc <-- your_driver_func+0x80
  worker-42   [001] ....  789.124: kmalloc: call_site=__genradix_ptr_alloc+0x4c
                                            ptr=0xffff888034560000
                                            bytes_req=512 bytes_alloc=512
                                            gfp_flags=GFP_KERNEL
```

- **`<-- your_driver_func+0x80`**: الـ caller — بيساعدك تعرف مين طالب الـ allocation
- **`bytes_alloc=512`**: node جديد اتحجز بـ GENRADIX_NODE_SIZE
- **`gfp_flags=GFP_KERNEL`**: allocation آمنة في process context — لو شفت `GFP_ATOMIC` هنا يبقى في مشكلة لأن الـ genradix ممكن يينام

---

#### 2. فحص memory leaks بعد unload الـ module

```bash
# صفّر الـ kmemleak قبل الـ test
echo clear > /sys/kernel/debug/kmemleak

# load واستخدام الـ module
sudo modprobe your_module

# شغّل الـ workload
your_test_program

# unload
sudo modprobe -r your_module

# افحص الـ leaks (اسبر scan يخلص)
echo scan > /sys/kernel/debug/kmemleak
sleep 10
cat /sys/kernel/debug/kmemleak
```

**مثال output عند وجود genradix leak:**

```
unreferenced object 0xffff888012340000 (size 512):
  comm "your_module_init", pid 1234, jiffies 4295123456
  hex dump (first 32 bytes): 00 00 00 00 00 00 ...
  backtrace:
    [<0>] kzalloc include/linux/slab.h:775 inline
    [<0>] genradix_alloc_node include/linux/generic-radix-tree.h:101 inline
    [<0>] __genradix_ptr_alloc+0x4c/0x120 lib/generic-radix-tree.c:87
    [<0>] your_function+0x30/0x80 drivers/your/module.c:42
```

**التفسير:** `genradix_free()` اتنسى في cleanup — الـ backtrace بيوريك بالظبط من أي line في كودك الـ allocation حصلت.

---

#### 3. fault injection لاختبار error handling في `genradix_ptr_alloc`

```bash
# يحتاج CONFIG_FAILSLAB=y في الـ kernel config
# تحقق أولاً
grep CONFIG_FAILSLAB /boot/config-$(uname -r)

# فعّل fault injection بنسبة 5%
echo 5   > /sys/kernel/debug/failslab/probability
echo 100 > /sys/kernel/debug/failslab/interval
echo -1  > /sys/kernel/debug/failslab/times
echo 0   > /sys/kernel/debug/failslab/space
echo 1   > /sys/kernel/debug/failslab/task-filter

# شغّل الـ workload وراقب
dmesg -w | grep -iE 'genradix|ENOMEM|alloc fail' &

your_test_program

# عطّل fault injection بعد الـ test
echo 0 > /sys/kernel/debug/failslab/probability
```

---

#### 4. تحليل SLUB stats لـ genradix nodes

```bash
# عرض alloc_calls مرتبة بعدد الـ allocations
sudo cat /sys/kernel/slab/kmalloc-512/alloc_calls 2>/dev/null | sort -rn | head -20

# مثال output:
#  5102 __genradix_ptr_alloc+0x4c/0x120 age=234/1205/3421 pid=1234 cpus=0,2
# |---- | |---------------------------------| |---------------|
# count   call site                           age: min/avg/max jiffies

# متابعة الـ objects في الـ real time
watch -n 1 'cat /sys/kernel/slab/kmalloc-512/objects'
# الرقم المستقر = عدد nodes موجودة
# الرقم اللي بيكبر باستمرار = memory leak

# عدد الـ slabs المستخدمة
cat /proc/slabinfo | awk '/kmalloc-512/ {print "active:", $2, "total:", $3}'
```

---

#### 5. قياس depth وحجم الـ genradix في runtime

```c
/* helper function للـ debugging — أضفها في الـ driver */
static void dump_genradix_info(struct __genradix *radix, const char *name)
{
    struct genradix_root *r = READ_ONCE(radix->root);
    unsigned depth = genradix_root_to_depth(r);
    struct genradix_node *node = genradix_root_to_node(r);
    size_t max_bytes = depth ? genradix_depth_size(depth) : 0;

    pr_info("genradix[%s]: depth=%u root=%px max_addressable=%zu bytes (~%zu entries of 64B)\n",
            name, depth, node, max_bytes, max_bytes / 64);
}
```

```bash
# اقرأ الـ output من dmesg
dmesg | grep 'genradix\[my_subsystem\]'

# مثال output:
# genradix[my_subsystem]: depth=2 root=0xffff888034ab0000 max_addressable=262144 bytes (~4096 entries of 64B)
# depth=2 يعني الـ tree عنده 3 levels وبيستوعب حتى 256KB من البيانات
# depth بيزيد تلقائي مع نمو الـ tree — كل depth يضاعف الـ capacity
```

**جدول العلاقة بين depth والـ capacity (على x86_64):**

| depth | max bytes | سبب |
|---|---|---|
| 0 | 512 B | leaf فقط = GENRADIX_NODE_SIZE |
| 1 | 32 KB | 64 children × 512 B |
| 2 | 2 MB | 64 × 64 × 512 B |
| 3 | 128 MB | 64³ × 512 B |
| 4 | 8 GB | 64⁴ × 512 B |
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: OOM أثناء تتبع USB endpoints على AM62x

#### العنوان
**Memory leak في genradix بيأكل RAM على industrial gateway بـ AM62x**

#### السياق
Gateway صناعي بيشتغل على **TI AM62x** (Cortex-A53)، بيدير 32 USB device متصلة بـ USB hub tree كبير. الـ driver بيستخدم `GENRADIX(struct usb_ep_state)` عشان يتتبع state كل endpoint. الـ system بيشتغل كويس أول 48 ساعة، بعدين بيبدأ الـ OOM killer يشتغل ويقتل daemons.

#### المشكلة
الـ engineer لاحظ إن الـ kernel memory بترتفع بثبات. الـ `slabinfo` بيظهر إن `kmalloc-512` بيتضخم على مر الوقت.

```bash
$ cat /proc/meminfo | grep Slab
Slab:            128456 kB
$ grep kmalloc-512 /proc/slabinfo
kmalloc-512   18432  18432   512  ...
```

#### التحليل
الـ genradix بيعمل `kzalloc(GENRADIX_NODE_SIZE, gfp_mask)` في `genradix_alloc_node`:

```c
static inline struct genradix_node *genradix_alloc_node(gfp_t gfp_mask)
{
    /* GENRADIX_NODE_SIZE = 512 bytes → goes to kmalloc-512 slab */
    return kzalloc(GENRADIX_NODE_SIZE, gfp_mask);
}
```

الـ driver كان بيعمل `genradix_ptr_alloc` عند كل connect event، لكن عند disconnect كان بيعمل `genradix_free` على الـ entry مش على الـ tree كلها. المشكلة إن **`genradix_free` بتفرّغ الـ tree كلها** — مش entry واحدة بس. مفيش API لـ "free single entry" في genradix.

```c
/* هنا الغلط: المبرمج كان يظن إنه بيحرر entry واحدة */
genradix_free(&dev->ep_radix);  /* في الواقع بيحرر الكل! */

/* بعدها بيعمل alloc تاني لنفس الـ tree */
p = genradix_ptr_alloc(&dev->ep_radix, ep_idx, GFP_KERNEL);
/* كل مرة بتبني tree جديدة → nodes جديدة → leak إن الـ old root اتضاع */
```

الـ root القديمة اتضاعت لأن الـ pointer لـ `struct __genradix` اتبدل في حالة race condition بين threads.

#### الحل
```c
/* الصح: استخدم genradix_ptr لمسح entry بدل free */
struct usb_ep_state *ep = genradix_ptr(&dev->ep_radix, ep_idx);
if (ep)
    memset(ep, 0, sizeof(*ep));  /* zero the entry, don't free tree */

/* وعند teardown بس: */
genradix_free(&dev->ep_radix);
```

إضافة lock لحماية الـ tree من concurrent access:

```c
mutex_lock(&dev->ep_lock);
p = genradix_ptr_alloc(&dev->ep_radix, ep_idx, GFP_KERNEL);
mutex_unlock(&dev->ep_lock);
```

#### الدرس المستفاد
الـ **genradix مش مصمم لـ per-entry free** — بيحرر الـ tree كلها أو ما يحررش. لو محتاج تحرر entries بشكل فردي، استخدم `memset` عشان تعمل zero للـ entry، أو استخدم `IDR/XArray` اللي بيدعم per-entry deletion.

---

### السيناريو 2: BUG_ON في __idx_to_offset على STM32MP1

#### العنوان
**Kernel panic بسبب struct كبيرة في genradix على STM32MP1 audio driver**

#### السياق
Board بـ **STM32MP1** (Cortex-A7 + M4) بتشتغل في نظام audio processing صناعي. الـ engineer الجديد في الفريق كتب driver لتتبع audio channel states:

```c
struct audio_ch_state {
    u32  freq;
    u32  gain;
    u8   buf[600];  /* buffer للـ samples */
    u32  flags;
};

static GENRADIX(struct audio_ch_state) ch_radix;
```

#### المشكلة
أول ما الـ module يتحمل:

```
[    2.341] kernel BUG at include/linux/generic-radix-tree.h:163!
[    2.341] Internal error: Oops - BUG: 0 [#1] SMP ARM
```

#### التحليل
الـ panic جاي من `__idx_to_offset`:

```c
static inline size_t __idx_to_offset(size_t idx, size_t obj_size)
{
    if (__builtin_constant_p(obj_size))
        BUILD_BUG_ON(obj_size > GENRADIX_NODE_SIZE);  /* compile-time */
    else
        BUG_ON(obj_size > GENRADIX_NODE_SIZE);         /* هنا حصل */
    ...
}
```

الـ `GENRADIX_NODE_SIZE = 512` bytes، لكن `struct audio_ch_state` حجمها:

```
sizeof(u32)*3 + sizeof(u8)*600 = 12 + 600 = 612 bytes > 512
```

الـ comment في الكود واضح:

```c
/*
 * NOTE: currently, sizeof(_type) must not be larger than GENRADIX_NODE_SIZE:
 */
```

لأن كل **leaf node** في الـ genradix هو `u8 data[GENRADIX_NODE_SIZE]` — مستحيل يخزن object أكبر منه في page واحدة.

#### الحل
**الحل 1:** تصغير الـ struct:

```c
struct audio_ch_state {
    u32  freq;
    u32  gain;
    u32  flags;
    /* نقل الـ buffer لـ heap منفصل */
    u8  *buf;
};
/* sizeof = 16 bytes */
```

**الحل 2:** لو الـ buffer ضروري inline، استخدم `vmalloc` array بدل genradix:

```c
struct audio_ch_state *ch_table;
ch_table = vzalloc(MAX_CHANNELS * sizeof(struct audio_ch_state));
```

أضف `static_assert` دفاعي في الكود:

```c
static_assert(sizeof(struct audio_ch_state) <= GENRADIX_NODE_SIZE,
              "audio_ch_state too large for genradix");
```

#### الدرس المستفاد
الـ **genradix له حد صارم**: الـ object يجب أن يكون `<= GENRADIX_NODE_SIZE (512 bytes)`. هذا القيد موجود لأن كل leaf هو page واحدة بالظبط. قبل ما تستخدم genradix، اعمل `static_assert(sizeof(your_type) <= GENRADIX_NODE_SIZE)` عشان تاخد compile-time error واضحة بدل runtime panic.

---

### السيناريو 3: Race condition في genradix_ptr_alloc على RK3562 Android TV

#### العنوان
**Random kernel crash في HDMI hotplug handler على RK3562 Android TV box**

#### السياق
Android TV box بـ **Rockchip RK3562** بيعمل HDMI hotplug detection. الـ driver بيستخدم genradix عشان يخزن HDMI sink capabilities لكل connected display. الـ box عليه 2 HDMI outputs. بيحصل crash عشوائي كل بضع ساعات عند plug/unplug سريع.

```
[crash] Unable to handle kernel NULL pointer dereference at virtual address 00000000
[crash] PC is at __genradix_ptr_alloc+0x38/0xb4
```

#### المشكلة
الـ crash بيحصل في `__genradix_ptr_alloc` عند concurrent access من اتنين HDMI interrupt handlers.

#### التحليل
الـ `__genradix_ptr_alloc` بتمشي على الـ tree وبتعمل allocation بدون أي locking داخلي. الـ `genradix_root_to_node` و `genradix_root_to_depth` بيقرأوا نفس الـ pointer:

```c
static inline struct genradix_node *genradix_root_to_node(struct genradix_root *r)
{
    /* depth مخزن في lower bits من الـ pointer */
    return (void *) ((unsigned long) r & ~GENRADIX_DEPTH_MASK);
}

static inline unsigned genradix_root_to_depth(struct genradix_root *r)
{
    return (unsigned long) r & GENRADIX_DEPTH_MASK;
}
```

الـ HDMI thread 1 (port 0) كان بيعمل tree growth (depth increase)، وفي نفس الوقت الـ thread 2 (port 1) كان بيقرأ الـ root القديمة. بعد الـ growth، الـ old node اتحرر، والـ thread 2 وصل لـ dangling pointer.

الـ `genradix_ptr_alloc_preallocated` كانت ممكن تساعد لو اتستخدمت صح:

```c
/* genradix_ptr_alloc_preallocated يسمح بـ pre-allocation بدون lock */
#define genradix_ptr_alloc_preallocated(_radix, _idx, _new_node, _gfp)  \
    (__genradix_cast(_radix)                                              \
     __genradix_ptr_alloc(&(_radix)->tree,                               \
             __genradix_idx_to_offset(_radix, _idx),                     \
             _new_node, _gfp))
```

#### الحل
```c
/* في الـ driver: إضافة spinlock لكل HDMI port */
struct hdmi_state {
    spinlock_t              radix_lock;
    GENRADIX(struct sink_cap) caps;
};

/* في الـ interrupt handler: */
spin_lock_irqsave(&hdmi->radix_lock, flags);
cap = genradix_ptr_alloc(&hdmi->caps, sink_idx, GFP_ATOMIC);
spin_unlock_irqrestore(&hdmi->radix_lock, flags);
```

أو استخدام الـ inlined variant مع pre-allocated node لتقليل critical section:

```c
struct genradix_node *new_node = genradix_alloc_node(GFP_ATOMIC);

spin_lock_irqsave(&hdmi->radix_lock, flags);
cap = genradix_ptr_alloc_preallocated_inlined(
        &hdmi->caps, sink_idx, &new_node, GFP_ATOMIC);
spin_unlock_irqrestore(&hdmi->radix_lock, flags);

/* لو new_node ما اتستخدمش، نحرره برا الـ lock */
if (new_node)
    genradix_free_node(new_node);
```

#### الدرس المستفاد
الـ **genradix مفيهوش أي internal locking** — الـ caller مسؤول بالكامل عن التزامن. الـ `genradix_ptr_alloc_preallocated` بيتيح نمط ممتاز: اعمل allocation برا الـ lock، وادخل الـ lock بس لحظة الـ insert، وحرر الـ node المتبقية برا الـ lock. ده بيقلل contention بشكل كبير.

---

### السيناريو 4: خطأ في iteration على i.MX8 بسبب non-power-of-2 struct

#### العنوان
**genradix_for_each بيتخطى entries على i.MX8 IoT gateway**

#### السياق
IoT sensor gateway بـ **NXP i.MX8M Plus** بيدير مئات الـ I2C sensors. الـ subsystem بيستخدم genradix لتخزين sensor metadata:

```c
struct sensor_meta {
    u8   addr;      /* I2C address */
    u8   type;
    u8   status;
    /* sizeof = 3 bytes — NOT power of 2 */
};

static GENRADIX(struct sensor_meta) sensors;
```

الـ engineer لاحظ إن بعض الـ sensors مش بتظهر في الـ status report اللي بيتعمل كل دقيقة.

#### المشكلة
الـ loop التالي بيتخطى sensors:

```c
struct genradix_iter iter;
struct sensor_meta *s;

genradix_for_each(&sensors, iter, s) {
    report_sensor(s->addr, s->status);
}
```

عدد الـ sensors المبلّغ عنها أقل من الـ sensors المتصلة فعلاً.

#### التحليل
المشكلة في تفاعل `__idx_to_offset` مع `__genradix_iter_advance` لما الـ `obj_size` مش power of 2.

**الـ `__idx_to_offset` لـ non-power-of-2:**

```c
if (!is_power_of_2(obj_size)) {
    size_t objs_per_page = GENRADIX_NODE_SIZE / obj_size;
    /* لـ obj_size=3: objs_per_page = 512/3 = 170 */
    /* كل page بتحتوي 170 entry، وفيه 512 - 170*3 = 2 bytes waste في آخر كل page */
    return (idx / objs_per_page) * GENRADIX_NODE_SIZE +
           (idx % objs_per_page) * obj_size;
}
```

**الـ `__genradix_iter_advance` بتتعامل مع الـ page boundary:**

```c
static inline void __genradix_iter_advance(struct genradix_iter *iter,
                                           size_t obj_size)
{
    iter->offset += obj_size;

    if (!is_power_of_2(obj_size) &&
        (iter->offset & (GENRADIX_NODE_SIZE - 1)) + obj_size > GENRADIX_NODE_SIZE)
        /* skip الـ waste bytes في آخر الـ page */
        iter->offset = round_up(iter->offset, GENRADIX_NODE_SIZE);

    iter->pos++;
}
```

الـ engineer كان بيعمل `genradix_ptr_alloc` بـ indices متتالية، لكن بسبب الـ page boundary skipping، الـ `iter.pos` ما كانش بيتطابق مع الـ actual index المخزن في الـ entry. الـ entries على حدود الـ pages كانت بتتخطى لأن الـ iterator بيقفز فوقها.

#### الحل
الحل الصريح: استخدم struct بـ power-of-2 size:

```c
struct sensor_meta {
    u8   addr;
    u8   type;
    u8   status;
    u8   _pad;  /* padding → sizeof = 4 = 2^2 */
};
```

أو استخدم explicit alignment:

```c
struct sensor_meta {
    u8   addr;
    u8   type;
    u8   status;
} __aligned(4);  /* force power-of-2 size */
```

تحقق في compile time:

```c
static_assert(is_power_of_2(sizeof(struct sensor_meta)),
              "sensor_meta must be power-of-2 for reliable genradix iteration");
```

#### الدرس المستفاد
الـ **genradix يشتغل مع non-power-of-2 sizes** لكن الـ iteration بتصبح أعقد وفيها edge cases في حدود الـ pages. أفضل ممارسة: اجعل الـ struct دايماً power-of-2 bytes باستخدام padding صريح. ده بيبسط الـ offset calculation ويضمن سلامة الـ iteration.

---

### السيناريو 5: أداء بطيء في genradix_prealloc على Allwinner H616 automotive ECU

#### العنوان
**Latency spikes في ECU startup بسبب genradix tree growth على Allwinner H616**

#### السياق
Automotive ECU بيشتغل على **Allwinner H616** (Cortex-A53) في نظام ADAS. الـ ECU بيبدأ تشغيله ويحتاج يحمّل بيانات 5000 CAN message descriptor في أقل من 50ms. الـ driver يستخدم:

```c
static GENRADIX(struct can_msg_desc) can_descriptors;
```

الـ startup بياخد 180ms بدل 50ms، والـ real-time scheduler بيشكي من jitter.

#### المشكلة
الـ latency جاية من إن الـ genradix بيعمل tree growth تدريجي أثناء الـ 5000 allocations. كل ما الـ tree محتاجة depth أكبر، بتعمل `kzalloc` جديدة وبتحرك الـ root pointer.

#### التحليل
بدون prealloc، الـ flow بيكون:

```
alloc idx 0    → depth=0, 1 node,  kzalloc x1
alloc idx 64   → depth=1, 2 nodes, kzalloc x2 + copy root
alloc idx 4096 → depth=2, 3 nodes, kzalloc x3 + 2 copy ops
...
```

كل tree growth بياخد وقت ومع الـ 5000 entries، الـ kzalloc calls بتتراكم. الـ `genradix_prealloc` بيحل ده:

```c
#define genradix_prealloc(_radix, _nr, _gfp)            \
     __genradix_prealloc(&(_radix)->tree,                \
            __genradix_idx_to_offset(_radix, _nr + 1),  \
            _gfp)
```

الـ `__genradix_prealloc` بيعمل tree growth مرة واحدة للـ depth المطلوبة، وبيـpre-allocate كل الـ intermediate nodes.

**حساب الـ depth المطلوبة لـ 5000 entries:**

```c
/*
 * GENRADIX_NODE_SHIFT  = 9   → node size = 512 bytes
 * GENRADIX_ARY_SHIFT   = ilog2(512/8) = ilog2(64) = 6
 * genradix_depth_shift(depth) = 9 + 6 * depth
 *
 * لو sizeof(can_msg_desc) = 32:
 *   objs_per_page = 512/32 = 16
 *   pages needed  = ceil(5000/16) = 313
 *   depth 1 covers: 2^(9+6) = 32768 bytes → 1024 objs
 *   depth 2 covers: 2^(9+12) = 2M bytes   → 65536 objs ← كافي
 */
```

#### الحل
في initialization:

```c
static int ecu_can_init(struct ecu_dev *ecu)
{
    int ret;

    genradix_init(&ecu->can_descriptors);

    /* pre-allocate قبل ما نبدأ الـ real-time phase */
    ret = genradix_prealloc(&ecu->can_descriptors, 5000, GFP_KERNEL);
    if (ret) {
        dev_err(ecu->dev, "failed to prealloc CAN descriptors: %d\n", ret);
        return ret;
    }

    /* الآن كل genradix_ptr_alloc هتلاقي الـ nodes موجودة */
    /* مفيش kzalloc جديدة أثناء الـ real-time loading */
    return load_can_descriptors(ecu);
}
```

قياس الفرق:

```bash
# قبل:
$ perf stat -e kmem:kmalloc ./ecu_startup_test
kmem:kmalloc: 5248 events

# بعد:
$ perf stat -e kmem:kmalloc ./ecu_startup_test
kmem:kmalloc: 3 events   /* الـ prealloc نفسه بس */
```

Startup time انخفض من 180ms لـ 12ms.

نقطة مهمة: الـ `genradix_prealloc` بيعمل alloc لكل الـ nodes من index 0 لـ `_nr`، حتى لو مش هتستخدمها كلها. ده tradeoff بين latency و memory:

```c
/* prealloc بـ margin محسوب */
ret = genradix_prealloc(&ecu->can_descriptors,
                        EXPECTED_MSG_COUNT * 110 / 100,  /* +10% margin */
                        GFP_KERNEL);
```

#### الدرس المستفاد
في الأنظمة الـ real-time أو عند وجود startup latency requirements، استخدم **`genradix_prealloc`** قبل أي real-time phase. ده بيضمن إن كل الـ `kmalloc` calls بتحصل مرة واحدة في initialization، ومفيش unexpected allocation latency أثناء التشغيل الحرج. الـ `__genradix_prealloc` بيتعامل مع كل الـ depth calculations تلقائياً.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ generic radix tree والـ data structures المرتبطة بيها في الـ kernel:

| المقال | الأهمية |
|--------|---------|
| [Generic radix trees — LWN.net](https://lwn.net/Articles/755281/) | **الأهم** — بيشرح بالظبط الـ `genradix` اللي في الملف ده، وليه اتعمل كـ replacement للـ `flex_array` |
| [Trees I: Radix trees — LWN.net](https://lwn.net/Articles/175432/) | الأساس النظري للـ radix tree في الـ kernel |
| [The XArray data structure — LWN.net](https://lwn.net/Articles/745073/) | الـ XArray هو الـ successor للـ radix tree القديمة، مهم تعرف الفرق |
| [A multi-order radix tree — LWN.net](https://lwn.net/Articles/688130/) | تطور الـ radix tree قبل ظهور الـ genradix |
| [The multi-order radix tree — LWN.net](https://lwn.net/Articles/684864/) | تفاصيل تقنية إضافية عن الـ multi-order support |
| [Linux kernel design patterns - part 2 — LWN.net](https://lwn.net/Articles/336255/) | design patterns عامة في الـ kernel بتشمل data structures زي دي |

---

### التوثيق الرسمي للـ Kernel

**الـ Documentation** الرسمي موجود في:

```
Documentation/core-api/generic-radix-tree.rst
```

متاح أونلاين على:
- [Generic radix trees/sparse arrays — kernel.org docs](https://docs.kernel.org/core-api/generic-radix-tree.html)
- [Generic radix trees — kernel.org v6.17-rc3](https://www.kernel.org/doc/html/v6.17-rc3/core-api/generic-radix-tree.html)

**الـ XArray** (البديل الأحدث للـ classic radix tree):
- [XArray — kernel.org docs](https://docs.kernel.org/core-api/xarray.html)

---

### Commits المهمة

#### الـ Commit الأصلي — إدخال الـ genradix

**`ba20ba2e3743bac786dff777954c11930256075e`** — "generic radix trees"

ده الـ commit اللي Kent Overstreet بعته سنة 2018 وأنشأ فيه:
- `include/linux/generic-radix-tree.h`
- `lib/generic-radix-tree.c`

الـ commit بدّل الـ `flex_array` بالـ `genradix` في أماكن كتير في الـ kernel.

رابط الـ commit على mirror:
```
https://vcs.cs.uchicago.edu/kauffman/ubuntu-mainline-crack/commit/ba20ba2e3743bac786dff777954c11930256075e
```

الـ source code على GitHub:
- [include/linux/generic-radix-tree.h — torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/generic-radix-tree.h)

---

### مناقشات الـ Mailing List

#### Patch Series مهمة:

1. **[PATCH 1/9] lib/generic-radix-tree.c: genradix_ptr_inlined()**
   - رابط: [mail-archive.com/rcu@vger.kernel.org](https://www.mail-archive.com/rcu@vger.kernel.org/msg02432.html)
   - بيضيف الـ inlined fast path للـ pointer lookup

2. **[PATCH 2/9] lib/generic-radix-tree.c: add preallocation**
   - رابط: [mail-archive.com/rcu@vger.kernel.org](https://www.mail-archive.com/rcu@vger.kernel.org/msg02434.html)
   - بيضيف `genradix_prealloc()` اللي موجودة في آخر الـ header

3. **[PATCH 04/16] MAINTAINERS: Include radixtree.py under GENERIC RADIX TREE entry**
   - رابط: [mail-archive.com/linux-kernel@vger.kernel.org](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2593581.html)
   - بيوضح إن الـ genradix ليه maintainer entry مستقل

4. **Kent Overstreet — overflow checking patch (2025)**
   - رابط: [lkml.org/lkml/2025/10/12/391](https://lkml.org/lkml/2025/10/12/391)
   - أحدث تطوير على الـ genradix

---

### الـ Source Code — ملفات مهمة في الـ Kernel

```
include/linux/generic-radix-tree.h   ← الـ header الرئيسي
lib/generic-radix-tree.c             ← الـ implementation
lib/test-genradix.c                  ← الـ test suite
```

الـ `MAINTAINERS` entry:
```
GENERIC RADIX TREE
M:  Kent Overstreet <kent.overstreet@linux.dev>
F:  include/linux/generic-radix-tree.h
F:  lib/generic-radix-tree.c
```

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل:** Chapter 11 — Data Types in the Kernel
- بيشرح الـ type-safe patterns اللي الـ genradix بتعتمد عليها زي `typeof()` و zero-size arrays
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل:** Chapter 6 — Kernel Data Structures
- بيغطي الـ linked lists، radix trees، و red-black trees في الـ kernel
- الـ genradix مش موجود فيه لأنه جه بعد الكتاب، بس الأساس النظري مهم

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل:** Chapter 16 — Kernel Debugging Techniques
- مش بيتكلم عن الـ genradix بشكل مباشر، بس بيشرح ازاي تقرأ وتفهم الـ kernel headers

#### Linux Inside (مجاني أونلاين)
- [Radix tree — linux-insides](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-2.html)
- شرح تفصيلي للـ classic radix tree في الـ kernel مع code walkthrough

---

### مصادر kernelnewbies.org

الـ kernelnewbies ما عندهاش صفحة مخصصة للـ genradix، بس الـ changelog pages مهمة لمتابعة التطور:

- [Linux_6.1 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.1) — بيذكر تحسينات على data structures
- [Linux_6.7 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.7) — bcachefs changes بتستخدم genradix
- [Linux_6.13 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.13) — أحدث kernel changes
- [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) — صفحة عامة للمتابعة

---

### eLinux.org

الـ elinux.org مش بيغطي الـ genradix بشكل مباشر، بس:

- [Linux Kernel Resources — elinux.org](https://elinux.org/Linux_Kernel_Resources) — روابط عامة للـ kernel development resources

---

### Search Terms للبحث عن معلومات أكتر

لو حبيت تبحث أكتر، استخدم الـ terms دي:

```
genradix linux kernel
generic-radix-tree.h
__genradix struct kernel
genradix_ptr_alloc kernel
genradix bcachefs
flex_array replacement kernel
linux sparse array type-safe
GENRADIX macro kernel
genradix_for_each iteration
```

**على Git:**
```bash
git log --all --oneline -- include/linux/generic-radix-tree.h
git log --all --oneline -- lib/generic-radix-tree.c
```

**على bootlin Elixir (أسرع طريقة لتتبع الـ code):**
```
https://elixir.bootlin.com/linux/latest/source/include/linux/generic-radix-tree.h
https://elixir.bootlin.com/linux/latest/source/lib/generic-radix-tree.c
```

**للبحث في استخدامات الـ genradix في الـ kernel:**
```bash
grep -r "GENRADIX\|genradix_" --include="*.h" --include="*.c" /path/to/linux/
```
## Phase 8: Writing simple module

### الفكرة

الـ `__genradix_ptr_alloc` هي الـ exported function الأكثر إثارة في الملف — بتتحكم في allocation الـ nodes جوه الـ generic radix tree. هنعمل **kprobe** عليها عشان نتابع كل مرة الـ kernel بيحاول يعمل allocate لـ entry جديدة، ونطبع الـ offset المطلوب وفلاجات الـ gfp.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on __genradix_ptr_alloc
 * Traces every allocation attempt inside a generic radix tree.
 */

/* ---- includes ---- */
#include <linux/kernel.h>       /* pr_info, pr_err               */
#include <linux/module.h>       /* MODULE_* macros, module_init  */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/ptrace.h>       /* struct pt_regs                */
#include <linux/gfp.h>          /* gfp_t, gfp flags              */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc AR");
MODULE_DESCRIPTION("kprobe on __genradix_ptr_alloc — trace genradix allocations");

/* ---- pre-handler: runs just before the real function ---- */
/*
 * Signature of __genradix_ptr_alloc:
 *   void *__genradix_ptr_alloc(struct __genradix *radix,
 *                              size_t offset,
 *                              struct genradix_node **new_node,
 *                              gfp_t gfp);
 *
 * On x86-64:
 *   rdi = radix ptr  (arg 1)
 *   rsi = offset     (arg 2)
 *   rdx = new_node   (arg 3)
 *   rcx = gfp        (arg 4)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* read args from the ABI registers */
    void        *radix  = (void *)regs->di;
    size_t       offset =         regs->si;
    gfp_t        gfp    = (gfp_t) regs->cx;

    pr_info("genradix_alloc: radix=%px offset=%zu gfp=0x%x\n",
            radix, offset, gfp);
    return 0; /* 0 = let the real function run normally */
}

/* ---- post-handler: runs right after the real function returns ---- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * ax holds the return value on x86-64.
     * NULL means allocation failed; non-NULL means a pointer to the entry.
     */
    void *ret = (void *)regs->ax;

    if (!ret)
        pr_info("genradix_alloc: allocation FAILED (returned NULL)\n");
    else
        pr_info("genradix_alloc: allocated entry @ %px\n", ret);
}

/* ---- kprobe struct ---- */
static struct kprobe kp = {
    .symbol_name    = "__genradix_ptr_alloc",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

/* ---- init ---- */
static int __init genradix_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("genradix_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("genradix_probe: planted kprobe on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---- exit ---- */
static void __exit genradix_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("genradix_probe: kprobe removed\n");
}

module_init(genradix_probe_init);
module_exit(genradix_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

الـ `<linux/kprobes.h>` بيجيب الـ API الخاص بالـ kprobes — بدونه مفيش `register_kprobe` ولا `struct kprobe`. الـ `<linux/ptrace.h>` بيعرّف `struct pt_regs` اللي بنقرأ منها الـ registers عشان نستخرج الـ arguments.

#### الـ `handler_pre`

بيشتغل **قبل** ما الـ function تتنفذ، فيقدر يشوف الـ arguments الأصلية. بنقرأ `rdi/rsi/rcx` على x86-64 عشان نطبع مين طلب الـ allocation وبأي offset وبأي memory flags.

#### الـ `handler_post`

بيشتغل **بعد** ما الـ function ترجع عشان نعرف نجحت ولا لا. الـ return value موجود في `rax` على x86-64، فلو جه `NULL` معناه الـ allocation فشلت.

#### الـ `struct kprobe`

الـ `symbol_name` بيقول للـ kernel على أي function نحط الـ probe. الـ kernel بيحوله لـ address تلقائياً عند الـ `register_kprobe`.

#### الـ `module_init` / `module_exit`

الـ `register_kprobe` في الـ init بيزرع breakpoint في الـ kernel memory على الـ symbol المطلوب. الـ `unregister_kprobe` في الـ exit ضروري عشان يشيل الـ breakpoint قبل ما الـ module يتـunload — لو اتنسي هيـcrash الـ kernel أول ما يحاول ينفذ عند الـ address اللي بقت invalid.

---

### بناء وتشغيل الـ Module

```bash
# Makefile
obj-m += genradix_probe.o

# build
make -C /lib/modules/$(uname -r)/build M=$PWD modules

# load
sudo insmod genradix_probe.ko

# watch output
sudo dmesg -w | grep genradix

# unload
sudo rmmod genradix_probe
```

> **ملحوظة:** لازم `CONFIG_KPROBES=y` و `CONFIG_KALLSYMS=y` في الـ kernel config عشان الـ `symbol_name` lookup يشتغل.
