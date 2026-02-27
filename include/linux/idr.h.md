## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الـ `include/linux/idr.h` جزء من الـ **XARRAY subsystem** — واللي بيحتويه Matthew Wilcox. الملف موجود في `MAINTAINERS` تحت entry اسمه `XARRAY` جنب `include/linux/xarray.h` و `lib/idr.c`.

---

### المشكلة اللي بيحلها الـ IDR

تخيل إنك بتشتغل في مطعم كبير. كل زبون بييجي بياخد **رقم ترتيب** — 1، 2، 3، إلخ — عشان لما بيجي طلبه جاهز تناديه بالرقم مش بالاسم. لو الزبون وصل وأخد طلبه، الرقم ده بيتحرر وممكن يتدي لزبون تاني.

ده بالظبط اللي بيعمله الـ **IDR (ID Radix tree)**.

الـ kernel في كتير من الأماكن محتاج يعمل **mapping من رقم صغير integer إلى pointer** — يعني يقول: ID رقم 5 ده بيشاور على `struct file` الفلانية. لما تعمل `open()` على ملف، الـ kernel بيديك **file descriptor** — وده مجرد رقم صغير زي 3 أو 7. جوا الـ kernel في جدول بيربط الرقم ده بـ pointer للـ `struct file` الحقيقية.

المشكلة القديمة إن الكل كان بيعمل الـ solution دي بنفسه — جداول ثابتة الحجم، arrays، كل واحد بطريقته. الـ IDR جه عشان يوحد الحل ده في مكان واحد.

---

### الـ IDR بالتفصيل — بس بشكل بسيط

```
التطبيق يطلب:     "أنا عندي pointer لـ struct device، إديني ID"
الـ IDR يرد:       "خد ID رقم 42"
بعدين:            idr_find(idr, 42)  →  يرجعلك الـ pointer

لما تخلص:         idr_remove(idr, 42)  →  الـ ID اتحرر
```

الـ IDR مش array عادية — جوا بيستخدم **Radix Tree** (وده نفسه wrapper فوق الـ **XArray** في الكرنل الحديث). الميزة إنه:

- **مش بيحجز memory** لكل الـ IDs الممكنة من 0 لـ 2^32
- بيحجز فقط لما في entries فعلية
- بيبحث في O(log n) وسط ممتاز للـ sparse IDs

---

### القصة الكاملة — مثال حقيقي: File Descriptors

```
الـ user بيعمل: open("/etc/passwd", O_RDONLY)
                        ↓
الـ kernel بيعمل struct file جديدة في الـ heap
                        ↓
بيستدعي: idr_alloc(&current->files->fdtab, file_ptr, 0, INT_MAX, GFP_KERNEL)
                        ↓
الـ IDR يلاقي أصغر ID متاح (مثلاً 3)
                        ↓
بيربط:  ID 3  →  pointer للـ struct file
                        ↓
الـ syscall يرجع 3 للـ user
```

لما الـ user بعدين يعمل `read(3, ...)`:
```
الـ kernel يعمل: idr_find(&current->files->fdtab, 3)
                        ↓
يرجعله الـ pointer للـ struct file
                        ↓
يكمل القراءة منها
```

---

### الـ IDR vs الـ IDA — إيه الفرق؟

| الميزة | IDR | IDA |
|--------|-----|-----|
| الهدف | ربط ID بـ pointer | مجرد تخصيص ID بدون pointer |
| الاستخدام | file descriptors، device numbers | process IDs، minor numbers |
| الـ memory | أكتر (بيخزن pointer) | أقل (bitmap فقط) |
| الـ backend | Radix Tree / XArray | XArray مباشرة |

**الـ IDA** هو النسخة الخفيفة — لما محتاج تتبع "هل الرقم ده اتأخد ولا لأ" بس من غير ما تربطه بأي بيانات.

---

### الـ Cyclic Allocator

الـ IDR عنده mode تاني اسمه **cyclic allocation** عبر `idr_alloc_cyclic()`. بدل ما دايماً يبدأ يدور من 0، بيكمل من آخر ID اتحجز. مفيد في الـ networking مثلاً — عشان تضمن إن packet ID جديد مش نفس الـ ID اللي اتحرر من شوية.

```
الحجز الأول:  ID = 100
الحجز التاني: ID = 101   (مش 0 من الأول)
بعد حذف 100:  ID = 102   (مش 100 تاني)
```

ده بيقلل احتمالية التداخل مع IDs قديمة لسه "طايرة" في الشبكة.

---

### الـ RCU Support

الـ `idr_find()` ممكن تتنادى **بدون lock** جوا `rcu_read_lock()`. يعني قراءة الـ IDR آمنة concurrently من عدة threads في نفس الوقت. بس الـ write operations (alloc/remove) محتاجة lock خارجي.

```c
rcu_read_lock();
ptr = idr_find(&my_idr, id);   /* lock-free read */
/* استخدم ptr هنا */
rcu_read_unlock();
```

---

### الـ Cleanup API الجديد

الـ header بيوفر `DEFINE_CLASS(idr_alloc, ...)` — ده جزء من الـ **cleanup.h** infrastructure. بيضمن إن لو الـ ID اتحجز وحصل error قبل ما تمسحه يدوياً، الـ compiler بيعمل cleanup تلقائي لما الـ variable يخرج من الـ scope (زي الـ RAII في C++).

---

### الملفات الأساسية في الـ Subsystem

```
include/linux/idr.h          ← الملف ده — الـ API والـ structs
include/linux/xarray.h       ← الـ backend الحقيقي (XArray)
include/linux/radix-tree.h   ← compatibility layer فوق XArray

lib/idr.c                    ← الـ implementation الفعلي للـ IDR وIDA
lib/xarray.c                 ← الـ XArray implementation
lib/radix-tree.c             ← legacy + wrappers

Documentation/core-api/idr.rst    ← الـ docs الرسمية
Documentation/core-api/xarray.rst ← docs الـ XArray
```

---

### ملاحظة مهمة: الـ IDR Deprecated

من الـ docs الرسمية صراحة:

> "The IDR interface is deprecated; please use the XArray instead."

يعني الـ IDR دلوقتي موجود للـ backward compatibility. الكود الجديد المفروض يستخدم **XArray API** مباشرة. بس فهم الـ IDR مهم عشان:
1. كتير من الكود القديم في الـ kernel لسه بيستخدمه
2. الـ IDR هو أبسط abstraction وأسهل فهماً كـ entry point
3. الـ IDA لسه مستخدم actively مش deprecated
## Phase 2: شرح الـ IDR/IDA Framework

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

في أي نظام تشغيل، الـ kernel بيحتاج يوزّع **integer IDs** كتير جداً — process IDs، file descriptors، device minor numbers، network socket IDs، إلخ. المشكلة الحقيقية إنك محتاج تعمل حاجتين في نفس الوقت:

1. **الـ ID يكون unique** — ماينفعش اتنين objects يشاركوا نفس الـ ID.
2. **تقدر ترجع الـ object من الـ ID بسرعة** — يعني lookup بـ O(log n) أو أحسن.

**الحل البدائي** كان static array:

```c
/* الطريقة البدائية — خايسة */
#define MAX_PIDS 32768
struct task_struct *pid_table[MAX_PIDS];
```

المشكلة إن الـ array بياكل memory حتى لو فيه 3 processes بس. لو IDs ممكن تبدأ من أرقام كبيرة (زي minor numbers)، هتحجز array ضخمة فاضية.

الـ **IDR (ID Radix)** جاء يحل بالضبط:
- حجز IDs ديناميكي بدون waste
- lookup سريع بـ O(log n) عبر radix tree
- لا upper bound مسبق محتاج تعرفه

---

### الحل — الـ Kernel بيعمل إيه؟

الـ IDR بيبني فوق الـ **Radix Tree** (أو الـ XArray في الكرنل الحديث، لأن `radix_tree_root` هو `xarray` نفسه بعد التوحيد). الفكرة إن الـ integer ID نفسه بيُستخدم كـ **index** في الشجرة، والـ value هو pointer للـ object.

الـ **IDA (ID Allocator)** هو نسخة مبسطة من الـ IDR — بتحجز IDs بس من غير ما تربطها بـ pointer. مفيد لما تحتاج رقم فريد بس مش هتعمل lookup بعدين.

---

### التشبيه الواقعي — فندق رقمي

تخيل فندق كبير فيه غرف بأرقام. الـ IDR هو نظام إدارة الغرف:

| مفهوم الفندق | مفهوم الـ IDR |
|---|---|
| رقم الغرفة | الـ integer ID |
| النزيل (بياناته) | الـ `void *ptr` |
| سجل الحجز | الـ `struct idr` |
| حجز غرفة جديدة | `idr_alloc()` |
| البحث عن نزيل بالرقم | `idr_find()` |
| تسجيل مغادرة | `idr_remove()` |
| جولة على كل النزلاء | `idr_for_each_entry()` |
| الفندق نظام cyclic | `idr_alloc_cyclic()` — يبدأ من آخر رقم وُزّع |

**التشبيه الأعمق:**
- الغرف مش متتالية بالضرورة — ممكن يكون في الفندق غرف 1, 5, 1003 بس من غير ما يحجز memory للغرف الفاضية في المنتصف. ده بالضبط اللي بتعمله الـ radix tree — sparse storage.
- مدير الفندق `idr_base` يحدد إيه أول رقم ممكن يُوزَّع (ممكن تبدأ من 1000 مثلاً).
- الـ `idr_next` زي signage board بيقول "آخر غرفة اتحجزت كانت كذا"، لتسريع الـ cyclic allocation.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                     KERNEL SUBSYSTEMS (Consumers)               │
  │                                                                 │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
  │  │  PID ns  │  │  fdtable │  │  netns   │  │  misc driver │   │
  │  │ idr_alloc│  │ idr_alloc│  │ idr_alloc│  │  ida_alloc   │   │
  └──┴────┬─────┴──┴────┬─────┴──┴────┬─────┴──┴──────┬───────┘   │
          │             │             │                │
          └─────────────┴──────┬──────┘                │
                               │                       │
               ┌───────────────▼───────────────┐       │
               │         IDR API               │       │
               │  idr_alloc / idr_find         │       │
               │  idr_remove / idr_for_each    │       │
               └───────────────┬───────────────┘       │
                               │                       │
               ┌───────────────▼───────────────┐  ┌────▼───────────┐
               │      struct idr               │  │   IDA API      │
               │  ┌──────────────────────┐     │  │ ida_alloc_range│
               │  │  idr_rt: xarray      │     │  │ ida_free       │
               │  │  idr_base: u32       │     │  └────┬───────────┘
               │  │  idr_next: u32       │     │       │
               │  └──────────┬───────────┘     │  ┌────▼───────────┐
               └─────────────┼─────────────────┘  │  struct ida    │
                             │                     │  xa: xarray    │
               ┌─────────────▼─────────────────┐  └────┬───────────┘
               │         XArray / Radix Tree    │       │
               │   (include/linux/xarray.h)     │◄──────┘
               │                               │
               │  ID → void* mapping           │
               │  Sparse, dynamic, O(log n)    │
               └───────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ **IDR** بيقدم abstraction واحدة بسيطة:

> **"أنا بحوّل integer ID صغير إلى pointer بكفاءة، وأضمن إن كل ID unique."**

الـ struct المحورية:

```c
struct idr {
    struct radix_tree_root  idr_rt;   /* الشجرة الفعلية — هي xarray */
    unsigned int            idr_base; /* أصغر ID ممكن يُوزَّع */
    unsigned int            idr_next; /* cursor للـ cyclic allocator */
};
```

**`idr_rt`** — ده مش مجرد pointer، ده الـ xarray نفسه embedded جوا الـ struct. الـ xarray هو الـ backend الفعلي اللي بيخزن الـ ID → pointer mappings.

**`idr_base`** — بيتيح تقول "أنا عايز IDs تبدأ من 1 مش 0" أو "تبدأ من 100". مفيد لما الـ ID 0 له معنى خاص (زي invalid).

**`idr_next`** — بيُستخدم بس مع `idr_alloc_cyclic()`. بدل ما دايماً تدور من البداية، بتبدأ من آخر ID وُزّع — مفيد لتوزيع عادل في أنظمة زي network packet IDs.

---

### كيف الـ Structs بتترابط

```
struct idr
┌──────────────────────────────────────────────┐
│ idr_rt (struct xarray = struct radix_tree_root)│
│  ┌────────────────────────────────────────┐  │
│  │ xa_head → ┌──────────────────────┐    │  │
│  │            │   xa_node (level 2)  │    │  │
│  │            │  slots[64]:          │    │  │
│  │            │  [0] → xa_node L1   │    │  │
│  │            │  [1] → xa_node L1   │    │  │
│  │            └──────────────────────┘    │  │
│  │                      │                 │  │
│  │            ┌──────────▼───────────┐    │  │
│  │            │   xa_node (level 1)  │    │  │
│  │            │  slots[64]:          │    │  │
│  │            │  [0] → void* obj_A  │    │  │
│  │            │  [3] → void* obj_B  │    │  │
│  │            │  [7] → void* obj_C  │    │  │
│  │            └──────────────────────┘    │  │
│  │ xa_flags: IDR_RT_MARKER               │  │
│  └────────────────────────────────────────┘  │
│ idr_base = 1    (IDs start from 1)           │
│ idr_next = 8    (cyclic cursor)              │
└──────────────────────────────────────────────┘

       ID 1 → obj_A
       ID 4 → obj_B  (IDs ممكن تكون sparse)
       ID 8 → obj_C
```

كل `xa_node` فيه 64 slot (على 64-bit systems مع `XA_CHUNK_SHIFT=6`). الـ ID بيتقسم لـ 6-bit chunks كل واحد منهم بيـindex مستوى من الشجرة.

**مثال: كيف الـ ID=1003 يُخزَّن**

```
1003 في binary = 0b01111101011

يتقسم:
  Level 0 (bits 5:0)  = 0b101011 = 43  → slot[43] في الـ node الأول
  Level 1 (bits 11:6) = 0b001111 = 15  → slot[15] في الـ node التاني

الوصول: root → node[15] → node[43] → pointer للـ object
```

---

### الـ IDR_RT_MARKER — التفاصيل الدقيقة

```c
#define IDR_FREE        0
#define IDR_RT_MARKER   (ROOT_IS_IDR | (__force gfp_t) \
                          (1 << (ROOT_TAG_SHIFT + IDR_FREE)))
```

الـ `xa_flags` في الـ xarray root بتشتغل بـ dual purpose:
- **`ROOT_IS_IDR`** — flag بيقول للـ xarray إنه في IDR mode، يعني ID allocation semantics مش key-value store عادي.
- **`IDR_FREE` tag** — الـ radix tree بتدعم tagging. الـ IDR بيستخدم tag 0 عشان يـtrack أي nodes فيها slots فاضية. ده بيسرّع إيجاد free slot بدون scan كامل للشجرة.

---

### الـ IDA — الأخ الأصغر

الـ **IDA** مختلف عن الـ IDR في نقطة واحدة جوهرية: **مبيخزنش pointers**، بيحجز IDs بس.

```c
struct ida {
    struct xarray xa;   /* xarray بيخزن بitmaps مش pointers */
};

struct ida_bitmap {
    unsigned long bitmap[IDA_BITMAP_LONGS]; /* 128 bytes = 1024 bits */
};
```

الـ IDA بيستخدم الـ xarray بطريقة ذكية: بدل ما كل slot يشير لـ object، كل slot بيشير لـ `ida_bitmap` — chunk من 1024 bit. كل bit تمثل ID واحد.

```
IDA Internal Layout:
xarray index 0 → ida_bitmap → bits 0..1023    (IDs 0-1023)
xarray index 1 → ida_bitmap → bits 0..1023    (IDs 1024-2047)
xarray index 2 → ida_bitmap → bits 0..1023    (IDs 2048-3071)
```

**ليه ده أكفأ للـ IDA؟**
لأن الـ IDA مبيعملش `idr_find()` — انت مش محتاج pointer، انت بس محتاج تعرف "هل الـ ID 500 محجوز؟". الـ bitmap representation بيخلي كل xarray slot يمثل 1024 ID، فالـ memory footprint أصغر بكتير من لو خزنا pointer لكل ID.

---

### الـ Preloading — التخصيص المسبق للذاكرة

```c
void idr_preload(gfp_t gfp_mask);
static inline void idr_preload_end(void);
```

الـ radix tree ممكن تحتاج memory allocation عند إضافة node جديدة. لو انت في سياق مش بتقدر تـallocate فيه (زي spinlock مقفول)، هتتعب.

الحل: **idr_preload()** بتـallocate nodes مسبقاً في الـ per-CPU preload cache، وبتـlock الـ per-CPU context عشان thread مش يتنقل بين CPUs في الوسط.

```c
/* الطريقة الصحيحة مع spinlocks */
idr_preload(GFP_KERNEL);        /* allocate مسبقاً، بتـlock per-CPU */
    spin_lock(&my_lock);
    id = idr_alloc(&my_idr, ptr, 0, 0, GFP_NOWAIT); /* مش هتـblock */
    spin_unlock(&my_lock);
idr_preload_end();              /* تحرير الـ per-CPU lock */
```

**ملاحظة:** الـ preload cache بيشتغل عبر `radix_tree_preloads` الـ per-CPU variable اللي موجودة في `radix-tree.h`.

---

### الـ Scope-based Cleanup — DEFINE_CLASS

```c
struct __class_idr {
    struct idr *idr;
    int id;
};

DEFINE_CLASS(idr_alloc, struct __class_idr,
     if (_T.id >= 0) idr_remove(_T.idr, _T.id),  /* cleanup */
     ((struct __class_idr){
         .idr = idr,
         .id = idr_alloc(idr, ptr, start, end, gfp),
     }),
     struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
```

ده بيستخدم `cleanup.h` framework عشان يضمن إن الـ ID بيتحرر automatically لو الـ variable خرج من الـ scope. بيمنع resource leaks في paths تانية من غير ما تكتب `goto err`.

```c
/* استخدامه */
{
    CLASS(idr_alloc, token)(&my_idr, obj, 0, 100, GFP_KERNEL);
    if (token.id < 0)
        return token.id; /* error path — مفيش leak */

    /* ... استخدم token.id ... */

} /* هنا idr_remove() بتتنادى automatically لو فيه مشكلة */
```

---

### الـ Concurrency Model — إزاي تأمن الـ IDR

الـ IDR **مش thread-safe by itself** — انت مسؤول عن الـ locking. بيوفر لك macros مريحة فوق الـ xarray lock:

```c
#define idr_lock(idr)          xa_lock(&(idr)->idr_rt)
#define idr_unlock(idr)        xa_unlock(&(idr)->idr_rt)
#define idr_lock_irq(idr)      xa_lock_irq(&(idr)->idr_rt)
#define idr_unlock_irq(idr)    xa_unlock_irq(&(idr)->idr_rt)
#define idr_lock_irqsave(idr, flags)    xa_lock_irqsave(&(idr)->idr_rt, flags)
```

**الـ idr_find() مع RCU** — الـ read path يقدر يشتغل بدون أي lock لو استخدمت RCU:

```c
/*
 * RCU يسمح بـ concurrent reads بدون lock
 * لازم الـ objects نفسها تكون RCU-safe
 */
rcu_read_lock();
ptr = idr_find(&my_idr, id);
/* استخدم ptr هنا — مضمون مش هيتحذف تحتك */
rcu_read_unlock();
```

**المتطلب:** الـ objects لازم تتحرر بـ `kfree_rcu()` أو بعد `synchronize_rcu()`.

---

### الـ IDR بيملك إيه vs بيـdelegate إيه للـ Drivers

| الـ IDR بيملكه | بيـdelegate للـ Driver |
|---|---|
| إيجاد أصغر free ID في range | اختيار الـ range المناسب (start, end) |
| ضمان uniqueness للـ ID | lifetime management للـ object نفسه |
| الـ radix tree structure | الـ locking strategy |
| الـ cyclic cursor state | تحديد امتى تستخدم cyclic vs linear |
| memory management للـ tree nodes | الـ RCU freeing للـ objects |
| الـ iteration logic | الـ filter logic جوا الـ callback |

---

### الـ Iteration API — قراءة الكل بالترتيب

الـ IDR بيوفر طرق تانية للـ iteration:

```c
/* طريقة 1: callback-based */
int my_callback(int id, void *p, void *data) {
    struct my_obj *obj = p;
    /* اعمل حاجة بالـ obj */
    return 0; /* استمر */
}
idr_for_each(&my_idr, my_callback, NULL);

/* طريقة 2: loop macro (أسهل) */
struct my_obj *obj;
int id;
idr_for_each_entry(&my_idr, obj, id) {
    pr_info("ID %d → obj %p\n", id, obj);
}

/* طريقة 3: unsigned long IDs */
struct my_obj *obj;
unsigned long id, tmp;
idr_for_each_entry_ul(&my_idr, obj, tmp, id) {
    /* tmp للـ overflow detection */
}
```

الـ `idr_for_each_entry_ul` بيستخدم `tmp` عشان يـdetect لو الـ `++id` أدى overflow، عشان الـ loop متـrun على آخر entry بشكل غلط.

---

### مثال واقعي كامل — Minor Number Allocation في Driver

```c
static DEFINE_IDR(my_minor_idr); /* IDR عام للـ driver */
static DEFINE_MUTEX(minor_lock);

/* allocate minor number جديد عند probe */
int my_driver_probe(struct platform_device *pdev)
{
    struct my_device *dev;
    int minor;

    dev = kzalloc(sizeof(*dev), GFP_KERNEL);
    if (!dev)
        return -ENOMEM;

    mutex_lock(&minor_lock);
    /* حجز ID من 0 لـ 255 */
    minor = idr_alloc(&my_minor_idr, dev, 0, 256, GFP_KERNEL);
    mutex_unlock(&minor_lock);

    if (minor < 0) {
        kfree(dev);
        return minor; /* -ENOSPC أو -ENOMEM */
    }

    dev->minor = minor;
    pr_info("Registered device with minor %d\n", minor);
    return 0;
}

/* lookup سريع من minor number */
struct my_device *find_device(int minor)
{
    struct my_device *dev;
    rcu_read_lock();
    dev = idr_find(&my_minor_idr, minor);
    rcu_read_unlock();
    return dev;
}

/* تحرير عند remove */
void my_driver_remove(struct platform_device *pdev)
{
    struct my_device *dev = platform_get_drvdata(pdev);

    mutex_lock(&minor_lock);
    idr_remove(&my_minor_idr, dev->minor);
    mutex_unlock(&minor_lock);

    kfree(dev);
}
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Macros والـ Constants — Cheatsheet

#### IDR Constants & Markers

| الاسم | القيمة | الوظيفة |
|---|---|---|
| `IDR_FREE` | `0` | رقم الـ tag اللي بيتتبع الـ slots الفاضية في الـ radix tree |
| `ROOT_IS_IDR` | `(__force gfp_t)4` | flag في الـ `xa_flags` بيميّز إن الـ root دي IDR مش xarray عادي |
| `ROOT_TAG_SHIFT` | `__GFP_BITS_SHIFT` | البداية اللي بيتخزن فيها الـ root tags في أعلى بتات الـ `xa_flags` |
| `IDR_RT_MARKER` | `ROOT_IS_IDR \| (1 << (ROOT_TAG_SHIFT + IDR_FREE))` | يتحط في الـ `xa_flags` عشان يشغّل الـ IDR mode ويفعّل الـ free-tracking tag |

#### IDA Constants

| الاسم | القيمة | الوظيفة |
|---|---|---|
| `IDA_CHUNK_SIZE` | `128` | حجم كل bitmap chunk بالـ bytes |
| `IDA_BITMAP_LONGS` | `128 / sizeof(long)` | عدد الـ `unsigned long` في الـ bitmap (16 على 64-bit) |
| `IDA_BITMAP_BITS` | `IDA_BITMAP_LONGS * 8 * sizeof(long)` | عدد الـ IDs اللي بيتخزن في chunk واحدة (1024 على 64-bit) |
| `IDA_INIT_FLAGS` | `XA_FLAGS_LOCK_IRQ \| XA_FLAGS_ALLOC` | الـ XArray تشتغل بـ IRQ-safe spinlock وتفعّل الـ alloc mode |

#### Radix Tree Iterator Flags

| الاسم | القيمة | الوظيفة |
|---|---|---|
| `RADIX_TREE_ITER_TAG_MASK` | `0x0f` | low nibble بيحتوي على رقم الـ tag |
| `RADIX_TREE_ITER_TAGGED` | `0x10` | ابحث في الـ tagged slots بس |
| `RADIX_TREE_ITER_CONTIG` | `0x20` | وقّف عند أول فجوة |

#### Lock Macros (IDR)

| الـ Macro | يعمل إيه |
|---|---|
| `idr_lock(idr)` | `xa_lock` على الـ `idr_rt` — spinlock عادي |
| `idr_lock_bh(idr)` | spinlock مع تعطيل الـ bottom halves |
| `idr_lock_irq(idr)` | spinlock مع تعطيل الـ interrupts |
| `idr_lock_irqsave(idr, flags)` | spinlock مع حفظ الـ IRQ flags |

---

### 1. الـ Structs المهمة

#### `struct idr`

**الغرض**: الـ IDR هو الـ data structure الرئيسي اللي بيربط **integer ID** بـ **void pointer**. بيُستخدم في كل حاجة بتحتاج تعمل mapping من رقم لبوينتر: process IDs بديلة، file descriptors في بعض السبسيستمز، device minor numbers، إلخ.

```c
struct idr {
    struct radix_tree_root  idr_rt;    /* الـ radix tree اللي بيخزن الـ pointers */
    unsigned int            idr_base;  /* أصغر ID ممكن يتخصص (default 0) */
    unsigned int            idr_next;  /* نقطة البداية للـ cyclic allocator */
};
```

| الـ Field | النوع | الشرح |
|---|---|---|
| `idr_rt` | `struct radix_tree_root` | هو `xarray` في التحديث الجديد — بيخزن الـ pointers indexed بالـ (id - idr_base) |
| `idr_base` | `unsigned int` | لو حابب الـ IDs تبدأ من 1000 مثلاً، تحط 1000 هنا |
| `idr_next` | `unsigned int` | الـ `idr_alloc_cyclic()` بيبدأ البحث من هنا عشان يوزّع الـ IDs بالدور |

**العلاقة مع الـ Structs التانية**: `idr_rt` هو اسم آخر لـ `xarray` (بسبب `#define radix_tree_root xarray` في radix-tree.h). يعني الـ IDR في الأساس هو xarray بـ flags خاصة.

---

#### `struct ida`

**الغرض**: أبسط من IDR — بس بيخصص أرقام من غير ما يربطها بـ pointer. بيوفر memory لأنه بيخزن bit واحدة لكل ID مش pointer كامل. مثال: numbering الـ USB devices، device instances.

```c
struct ida {
    struct xarray xa;  /* الـ XArray بيخزن bitmaps مش pointers */
};
```

| الـ Field | النوع | الشرح |
|---|---|---|
| `xa` | `struct xarray` | كل entry فيه ممكن يكون `ida_bitmap*` أو value entry (بيت packed في pointer) |

---

#### `struct ida_bitmap`

**الغرض**: قطعة الـ storage الأساسية جوا الـ IDA — كل chunk بتتحكم في 1024 ID على 64-bit system.

```c
struct ida_bitmap {
    unsigned long bitmap[IDA_BITMAP_LONGS]; /* 16 longs = 128 bytes = 1024 bits */
};
```

| الـ Field | الشرح |
|---|---|
| `bitmap[]` | كل bit بتمثل ID واحد. Bit مضروبة = ID متخصص. Bit صفر = فاضي |

**ملاحظة أداء مهمة**: الـ IDA بيستخدم **value entries** لما الـ IDs القليلة في أول chunk — بدل ما يعمل `kmalloc` لـ `ida_bitmap` كاملة بيخزن الـ bits مباشرة في الـ pointer value (الـ XArray بيدعم ده بـ `xa_mk_value`). ده بيوفر memory لما IDs قليلة.

---

#### `struct __class_idr`

**الغرض**: مساعد للـ **scope-based cleanup** — بيحتفظ بالـ IDR والـ ID مع بعض عشان الـ `DEFINE_CLASS` macro تعرف تعمل cleanup تلقائي لما الـ scope ينتهي.

```c
struct __class_idr {
    struct idr *idr;  /* الـ IDR اللي اتخصص منه */
    int id;           /* الـ ID اللي اتخصص، أو -1 لو فشل */
};
```

**الاستخدام**:
```c
/* بدل إن تتذكر تعمل idr_remove() لو فشل في النص */
CLASS(idr_alloc, myid)(idr, ptr, 0, 1000, GFP_KERNEL);
if (myid.id < 0)
    return myid.id; /* تلقائياً مش محتاج cleanup */
/* ... استخدم myid.id ... */
/* لما تخرج من الـ scope، idr_remove يتعمل تلقائي */
```

---

#### `struct radix_tree_iter`

**الغرض**: حالة الـ iterator أثناء المشي على الـ radix tree/xarray — بيتستخدم داخلياً في كل عمليات الـ IDR.

```c
struct radix_tree_iter {
    unsigned long   index;       /* الـ index الحالي في الـ tree */
    unsigned long   next_index;  /* بداية الـ chunk الجاي */
    unsigned long   tags;        /* bitmask للـ tags في الـ chunk الحالي */
    struct radix_tree_node *node; /* الـ node اللي فيها الـ chunk الحالي */
};
```

---

#### `struct radix_tree_preload`

**الغرض**: per-CPU pool من الـ pre-allocated nodes عشان الـ `idr_alloc()` ميحتاجش يعمل memory allocation وهو شايل الـ lock — ده بيتجنب deadlock لو الـ allocator نفسه محتاج الـ lock.

```c
struct radix_tree_preload {
    local_lock_t lock;             /* per-CPU lock يمنع preemption */
    unsigned nr;                   /* عدد الـ nodes المتاحة */
    struct radix_tree_node *nodes; /* linked list من الـ pre-allocated nodes */
};
DECLARE_PER_CPU(struct radix_tree_preload, radix_tree_preloads);
```

---

### 2. رسم علاقات الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                         struct idr                              │
│                                                                 │
│  idr_rt ──────────────────────────────────────────────────────┐ │
│  (radix_tree_root = xarray)                                   │ │
│  idr_base = 0..N                                              │ │
│  idr_next = last cyclic position                              │ │
└───────────────────────────────────────────────────────────────┼─┘
                                                                │
                           ┌────────────────────────────────────┘
                           ▼
                  ┌─────────────────┐
                  │  struct xarray  │  (= radix_tree_root)
                  │                 │
                  │  xa_head ───────┼──► xa_node (internal)
                  │  xa_flags       │         │
                  │  xa_lock        │         ├──► xa_node (internal)
                  └─────────────────┘         │         │
                                              │         └──► void* (user ptr)
                                              └──► void* (user ptr)

┌─────────────────────────────────────────────────────────────────┐
│                         struct ida                              │
│                                                                 │
│  xa ──────────────────────────────────────────────────────────┐ │
│  (struct xarray)                                              │ │
└───────────────────────────────────────────────────────────────┼─┘
                                                                │
                           ┌────────────────────────────────────┘
                           ▼
                  ┌─────────────────┐
                  │  struct xarray  │
                  │                 │
                  │  xa_head ───────┼──► xa_node
                  │  xa_flags =     │         │
                  │  LOCK_IRQ|ALLOC │         ├──► ida_bitmap* (128 bytes)
                  └─────────────────┘         │         bitmap[16]
                                              │         (1024 IDs per chunk)
                                              └──► xa_value (packed bits)
                                                   (لو IDs قليلة جداً)

┌─────────────────────┐
│  struct __class_idr │
│                     │
│  idr ───────────────┼──► struct idr
│  id  (int)          │
└─────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### IDR Lifecycle

```
[CREATION]
    │
    ├── Static:  DEFINE_IDR(name)
    │            └─► IDR_INIT_BASE(name, 0)
    │                └─► RADIX_TREE_INIT(..., IDR_RT_MARKER)
    │
    └── Dynamic: idr_init(idr)         ← أو idr_init_base(idr, base)
                 └─► INIT_RADIX_TREE(&idr->idr_rt, IDR_RT_MARKER)
                     idr->idr_base = 0 (أو base)
                     idr->idr_next = 0
    │
    ▼
[PRELOAD — اختياري لكن مهم للـ performance]
    idr_preload(GFP_KERNEL)
    └─► radix_tree_preload()
        └─► يملي per-CPU pool بـ pre-allocated nodes
        └─► يعمل local_lock (يمنع preemption)
    │
    ▼
[ALLOCATION]
    idr_alloc(idr, ptr, start, end, gfp)
    └─► idr_alloc_u32()
        └─► radix_tree_iter_init()
        └─► idr_get_free()           ← يبحث عن slot فاضي بالـ IDR_FREE tag
        └─► radix_tree_iter_replace() ← يحط الـ ptr
        └─► radix_tree_iter_tag_clear(IDR_FREE) ← يمسح علامة "فاضي"
        └─► يرجع الـ ID
    │
    ▼
[PRELOAD END — لو عملت preload]
    idr_preload_end()
    └─► local_unlock(&radix_tree_preloads.lock)
    │
    ▼
[USAGE]
    idr_find(idr, id)
    └─► radix_tree_lookup(&idr->idr_rt, id - idr->idr_base)
        └─► يرجع الـ void* المرتبط بالـ id

    idr_replace(idr, new_ptr, id)
    └─► __radix_tree_lookup() + __radix_tree_replace()

    idr_for_each(idr, fn, data)
    └─► radix_tree_for_each_slot() + fn(id, ptr, data)
    │
    ▼
[REMOVAL]
    idr_remove(idr, id)
    └─► radix_tree_delete_item(&idr->idr_rt, id - idr->idr_base, NULL)
        └─► يمسح الـ entry ويرجع الـ pointer القديم
    │
    ▼
[TEARDOWN]
    idr_destroy(idr)
    └─► radix_tree_delete_item() for each entry
        └─► بيحرر كل الـ nodes الداخلية
        └─► بيسيب الـ idr struct نفسها فاضية (مش بيعمل kfree للـ idr)
```

#### IDA Lifecycle

```
[CREATION]
    │
    ├── Static:  DEFINE_IDA(name)
    │            └─► XARRAY_INIT(name, XA_FLAGS_LOCK_IRQ | XA_FLAGS_ALLOC)
    │
    └── Dynamic: ida_init(ida)
                 └─► xa_init_flags(&ida->xa, IDA_INIT_FLAGS)
    │
    ▼
[ALLOCATION]
    ida_alloc_range(ida, min, max, gfp)
    │
    ├── [FAST PATH] لو IDs قليلة:
    │   └─► XA_STATE على index = min/IDA_BITMAP_BITS
    │   └─► xas_find_marked(..., XA_FREE_MARK)
    │   └─► لو entry عبارة عن xa_value (packed bits):
    │       └─► find_next_zero_bit() في الـ value
    │       └─► xas_store(xa_mk_value(v | bit))
    │
    └── [NORMAL PATH] لو محتاج bitmap:
        └─► kzalloc(sizeof(ida_bitmap))
        └─► __set_bit(bit, bitmap->bitmap)
        └─► xas_store(bitmap)
        └─► لو الـ bitmap امتلأ: xas_clear_mark(XA_FREE_MARK)
    │
    ▼
[USAGE]
    ida_exists(ida, id)
    └─► ida_find_first_range(ida, id, id) == id

    ida_find_first(ida)
    └─► ida_find_first_range(ida, 0, ~0)
        └─► xa_find(..., XA_PRESENT)
        └─► find_next_bit() في الـ bitmap أو value
    │
    ▼
[FREE]
    ida_free(ida, id)
    └─► XA_STATE على index = id/IDA_BITMAP_BITS
    └─► xas_load() ← يجيب الـ bitmap
    ├── لو xa_value: يمسح الـ bit من الـ value
    └── لو ida_bitmap:
        └─► __clear_bit(bit, bitmap->bitmap)
        └─► xas_set_mark(XA_FREE_MARK)
        └─► لو الـ bitmap فاضي كله: kfree(bitmap) + xas_store(NULL)
    │
    ▼
[TEARDOWN]
    ida_destroy(ida)
    └─► xas_lock_irqsave()
    └─► xas_for_each(..., ULONG_MAX):
        └─► لو مش xa_value: kfree(bitmap)
        └─► xas_store(NULL)
    └─► xas_unlock_irqrestore()
```

---

### 4. Call Flow Diagrams

#### `idr_alloc()` — التفصيلة الكاملة

```
caller: idr_alloc(idr, ptr, start, end, GFP_KERNEL)
  │
  ├─► [validate] start >= 0
  │
  └─► idr_alloc_u32(idr, ptr, &id, end-1, gfp)
        │
        ├─► [validate] idr_rt.xa_flags & ROOT_IS_IDR
        │             (لو مش موجود، يضيف IDR_RT_MARKER — safety check)
        │
        ├─► id = (id < base) ? 0 : id - base
        │             (يحوّل الـ external ID لـ internal index)
        │
        ├─► radix_tree_iter_init(&iter, id)
        │   └─► iter.next_index = id, iter.index = 0
        │
        ├─► idr_get_free(&idr->idr_rt, &iter, gfp, max - base)
        │   └─► [داخل radix-tree.c]
        │       └─► يمشي على الـ tree بحثاً عن slot بـ IDR_FREE tag
        │       └─► لو ملقتش: يخصص nodes جديدة (من preload أو kmalloc)
        │       └─► يرجع slot** للمكان الفاضي
        │
        ├─► *nextid = iter.index + base  (يكتب الـ ID للـ caller)
        │
        ├─► radix_tree_iter_replace(&idr->idr_rt, &iter, slot, ptr)
        │   └─► memory barrier + يكتب الـ ptr في الـ slot
        │
        └─► radix_tree_iter_tag_clear(&idr->idr_rt, &iter, IDR_FREE)
            └─► يمسح علامة "فاضي" عشان البحث الجاي ميجيش هنا

  returns: id (الـ external ID = internal index + base)
```

#### `idr_find()` — Fast Path (RCU)

```
caller (تحت rcu_read_lock()): idr_find(idr, id)
  │
  └─► radix_tree_lookup(&idr->idr_rt, id - idr->idr_base)
        │
        └─► [lock-free radix tree traversal]
            ├─► يبدأ من xa_head
            ├─► يحسب الـ index في كل مستوى
            ├─► يمشي على الـ nodes بـ rcu_dereference()
            └─► يرجع الـ pointer أو NULL

  returns: void* أو NULL
  [لا يحتاج lock — RCU فقط]
```

#### `idr_alloc_cyclic()` — Wrap-around Logic

```
caller: idr_alloc_cyclic(idr, ptr, start, end, gfp)
  │
  ├─► id = idr->idr_next  (ابدأ من آخر مكان وقفنا)
  ├─► لو id < start: id = start
  │
  ├─► [محاولة أولى] idr_alloc_u32(idr, ptr, &id, max, gfp)
  │
  ├─► لو فشل بـ ENOSPC وكان id > start:
  │   └─► id = start  (ارجع للبداية — cyclic wrap)
  │   └─► [محاولة تانية] idr_alloc_u32(idr, ptr, &id, max, gfp)
  │
  └─► idr->idr_next = id + 1  (احفظ مكان التوقف)

  returns: id
```

#### `ida_alloc_range()` — Memory Allocation Strategy

```
caller: ida_alloc_range(ida, min, max, gfp)
  │
  ├─► XA_STATE(xas, &ida->xa, min / IDA_BITMAP_BITS)
  │   bit = min % IDA_BITMAP_BITS
  │
retry:
  ├─► xas_lock_irqsave()  [IRQ-safe spinlock]
  │
  ├─► xas_find_marked(..., XA_FREE_MARK)
  │   └─► يبحث عن chunk فيها IDs فاضية
  │
  ├─► [CASE 1] entry هو xa_value (packed bits — لـ IDs القليلة):
  │   ├─► find_next_zero_bit() في الـ value
  │   ├─► لو لقى: xas_store(xa_mk_value(v | bit))  ← بدون malloc
  │   └─► لو امتلأ الـ value: يحوّله لـ ida_bitmap
  │
  ├─► [CASE 2] entry هو ida_bitmap*:
  │   ├─► find_next_zero_bit() في الـ bitmap الكامل
  │   ├─► __set_bit(bit, bitmap->bitmap)
  │   └─► لو الـ bitmap امتلأ: xas_clear_mark(XA_FREE_MARK)
  │
  └─► [CASE 3] مفيش entry (أول مرة):
      ├─► لو bit صغير: xa_mk_value(1UL << bit)  ← بدون malloc
      └─► لو bit كبير: kmalloc(ida_bitmap) + set_bit

alloc:  [لو احتجنا malloc وما عندناش]
  ├─► xas_unlock_irqrestore()
  ├─► alloc = kzalloc(sizeof(ida_bitmap), gfp)
  └─► goto retry

  returns: index * IDA_BITMAP_BITS + bit  (= الـ ID المخصص)
```

---

### 5. استراتيجية الـ Locking

#### IDR — المستخدم مسؤول عن الـ Locking

الـ IDR **لا يعمل locking تلقائي** للكتابة. القاعدة:

| العملية | الـ Lock المطلوب | السبب |
|---|---|---|
| `idr_alloc()` / `idr_remove()` / `idr_replace()` | يحتاج lock خارجي (spinlock / mutex) | عمليات write تعديل بنية الـ tree |
| `idr_find()` | بدون lock — RCU كافي | read-only مع `rcu_dereference()` |
| `idr_for_each()` | RCU يكفي | قراءة فقط |
| `idr_preload()` → `idr_alloc()` → `idr_preload_end()` | يعمل `local_lock` على الـ per-CPU pool | يمنع preemption بين الـ preload والـ alloc |

**macros الـ locking المتاحة**:
```c
idr_lock(idr)           /* xa_lock — spinlock عادي */
idr_lock_bh(idr)        /* spinlock + disable BH */
idr_lock_irq(idr)       /* spinlock + disable IRQ */
idr_lock_irqsave(idr, flags) /* spinlock + save IRQ flags */
```

كل دول في الأساس بيلفوا `xa_lock` على الـ `idr->idr_rt` (اللي هو الـ xarray).

**مثال صح**:
```c
/* write path */
spin_lock(&mydev->idr_lock);
id = idr_alloc(&mydev->idr, ptr, 0, 0, GFP_ATOMIC);
spin_unlock(&mydev->idr_lock);

/* read path */
rcu_read_lock();
ptr = idr_find(&mydev->idr, id);
/* استخدم ptr هنا */
rcu_read_unlock();
```

#### IDA — Locking تلقائي

الـ IDA **يعمل locking بنفسه** — الـ caller ميحتاجش يعمل أي lock:

| العملية | الـ Lock المستخدم داخلياً |
|---|---|
| `ida_alloc_range()` | `xas_lock_irqsave()` — IRQ-safe spinlock |
| `ida_free()` | `xas_lock_irqsave()` — نفس اللوك |
| `ida_destroy()` | `xas_lock_irqsave()` — نفس اللوك |
| `ida_find_first_range()` | `xa_lock_irqsave()` |

**ليه IRQ-safe؟** لأن الـ `IDA_INIT_FLAGS` بيحتوي `XA_FLAGS_LOCK_IRQ` — ده مهم لأن الـ IDA ممكن يتعمل منه `alloc` من interrupt context (زي interrupt handler بيعمل device numbering).

#### ترتيب الـ Locks (Lock Ordering) — تجنب Deadlock

```
[من الأعلى للأسفل — دايماً خد اللوك في الترتيب ده]

  1. (اختياري) Lock خارجي للمستخدم (mutex/spinlock عنده)
       │
       ▼
  2. local_lock(&radix_tree_preloads.lock)  — بس لو عملت idr_preload()
       │
       ▼
  3. idr_lock / xa_lock على الـ idr_rt / xarray
       │
       ▼
  [memory allocator — بس لو gfp يسمح بـ blocking]
```

**تحذير مهم**: لو عملت `idr_preload(GFP_KERNEL)` لازم الـ `idr_alloc()` اللي بعدها يكون بـ `GFP_ATOMIC` لأنك شايل `local_lock` اللي بيمنع النوم:
```c
idr_preload(GFP_KERNEL);    /* ممكن ينام هنا — قبل اللوك */
idr_lock(idr);
id = idr_alloc(idr, ptr, 0, 0, GFP_ATOMIC); /* لازم ATOMIC هنا */
idr_unlock(idr);
idr_preload_end();           /* يطلق local_lock */
```

#### RCU و Memory Barriers

**الـ `idr_find()` lockless** يشتغل صح بسبب:
1. الـ `radix_tree_iter_replace()` بيعمل **memory barrier** قبل ما يكتب الـ pointer
2. الـ `radix_tree_lookup()` بيستخدم `rcu_dereference()` عشان يضمن يشوف القيمة الصح
3. الـ items المتحررة لازم تتحرر بـ `call_rcu()` أو بعد `synchronize_rcu()` مش قبل

**الضمان**: لو `idr_find()` رجع pointer مش NULL، الـ pointer ده صالح طول ما انت في الـ `rcu_read_lock()` region.

---

### ملخص العلاقات (ASCII Overview)

```
┌──────────────────────────────────────────────────────────────────┐
│                     idr.h / idr.c Overview                      │
│                                                                  │
│  ┌─────────┐    wraps    ┌──────────────────┐                   │
│  │  idr    │────────────►│  xarray          │                   │
│  │         │             │  (radix_tree)    │                   │
│  │ base    │             │  IDR_RT_MARKER   │                   │
│  │ next    │             └────────┬─────────┘                   │
│  └────┬────┘                      │ stores                      │
│       │                           ▼                             │
│       │                    void* (user pointers)                │
│       │                    indexed by (id - base)               │
│       │                                                         │
│  ┌────▼────┐                                                     │
│  │  ida    │    wraps    ┌──────────────────┐                   │
│  │         │────────────►│  xarray          │                   │
│  └─────────┘             │  LOCK_IRQ|ALLOC  │                   │
│                          └────────┬─────────┘                   │
│                                   │ stores                      │
│                          ┌────────┴──────────┐                  │
│                          │                   │                  │
│                    ida_bitmap*          xa_value                │
│                    (128 bytes)          (packed bits)           │
│                    1024 IDs/chunk       < 62 IDs                │
│                                                                  │
│  ┌──────────────────┐    DEFINE_CLASS                           │
│  │  __class_idr     │────────────────► auto idr_remove()       │
│  │  idr* + int id   │                  on scope exit           │
│  └──────────────────┘                                           │
└──────────────────────────────────────────────────────────────────┘
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### IDR Functions

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `idr_init` | `void idr_init(struct idr *)` | تهيئة IDR ديناميكي بـ base=0 |
| `idr_init_base` | `void idr_init_base(struct idr *, int base)` | تهيئة IDR مع تحديد الـ base |
| `idr_alloc` | `int idr_alloc(struct idr *, void *, int start, int end, gfp_t)` | تخصيص ID وربطه بـ pointer |
| `idr_alloc_u32` | `int idr_alloc_u32(struct idr *, void *, u32 *id, unsigned long max, gfp_t)` | تخصيص ID بنوع u32 |
| `idr_alloc_cyclic` | `int idr_alloc_cyclic(struct idr *, void *, int start, int end, gfp_t)` | تخصيص cyclic يبدأ من آخر موقع |
| `idr_remove` | `void *idr_remove(struct idr *, unsigned long id)` | حذف ID وإرجاع الـ pointer |
| `idr_find` | `void *idr_find(const struct idr *, unsigned long id)` | بحث بـ ID — lockless RCU |
| `idr_replace` | `void *idr_replace(struct idr *, void *, unsigned long id)` | استبدال الـ pointer لـ ID موجود |
| `idr_get_next` | `void *idr_get_next(struct idr *, int *nextid)` | iteration — التالي من ID معطى |
| `idr_get_next_ul` | `void *idr_get_next_ul(struct idr *, unsigned long *nextid)` | نفسه لكن بنوع `unsigned long` |
| `idr_for_each` | `int idr_for_each(const struct idr *, fn, void *data)` | iteration مع callback |
| `idr_destroy` | `void idr_destroy(struct idr *)` | تحرير كل موارد الـ IDR |
| `idr_preload` | `void idr_preload(gfp_t gfp_mask)` | prealloc nodes قبل atomic alloc |
| `idr_preload_end` | `void idr_preload_end(void)` | نهاية نافذة الـ preload |
| `idr_get_cursor` | `unsigned int idr_get_cursor(const struct idr *)` | قراءة موقع الـ cyclic allocator |
| `idr_set_cursor` | `void idr_set_cursor(struct idr *, unsigned int val)` | تعيين موقع الـ cyclic allocator |
| `idr_is_empty` | `bool idr_is_empty(const struct idr *)` | هل الـ IDR فارغ؟ |

#### IDR Locking Macros

| Macro | المكافئ |
|---|---|
| `idr_lock` / `idr_unlock` | `xa_lock` / `xa_unlock` على الـ `idr_rt` |
| `idr_lock_bh` / `idr_unlock_bh` | نسخة bottom-half safe |
| `idr_lock_irq` / `idr_unlock_irq` | نسخة IRQ-disabled |
| `idr_lock_irqsave` / `idr_unlock_irqrestore` | نسخة تحفظ IRQ flags |

#### IDR Static Init Macros

| Macro | الوظيفة |
|---|---|
| `DEFINE_IDR(name)` | تعريف وتهيئة IDR ساكن |
| `IDR_INIT(name)` | compound literal للتهيئة بـ base=0 |
| `IDR_INIT_BASE(name, base)` | compound literal مع تحديد base |

#### IDR Iteration Macros

| Macro | الحالة |
|---|---|
| `idr_for_each_entry(idr, entry, id)` | iteration كاملة من 0 |
| `idr_for_each_entry_ul(idr, entry, tmp, id)` | نفسه بـ `unsigned long` |
| `idr_for_each_entry_continue(idr, entry, id)` | استئناف من موقع محدد |
| `idr_for_each_entry_continue_ul(idr, entry, tmp, id)` | استئناف بـ `unsigned long` |

#### Cleanup Class

| Macro/Type | الوظيفة |
|---|---|
| `DEFINE_CLASS(idr_alloc, ...)` | scoped cleanup — يحذف ID تلقائياً عند خروج الـ scope |
| `struct __class_idr` | يحمل `{idr *, id}` للـ cleanup class |
| `idr_null` | sentinel يمثل IDR فارغ `{NULL, -1}` |
| `take_idr_id(id)` | يأخذ الـ ID ويصفّر المصدر (ownership transfer) |

#### IDA Functions

| Function | الغرض |
|---|---|
| `ida_alloc` | تخصيص ID من 0 إلى INT_MAX |
| `ida_alloc_min` | تخصيص ID من min إلى INT_MAX |
| `ida_alloc_max` | تخصيص ID من 0 إلى max |
| `ida_alloc_range` | تخصيص ID في نطاق [min, max] |
| `ida_free` | تحرير ID |
| `ida_destroy` | تحرير كل موارد الـ IDA |
| `ida_find_first` | أول ID مخصص من 0 |
| `ida_find_first_range` | أول ID مخصص في نطاق [min, max] |
| `ida_exists` | هل ID معين مخصص؟ |
| `ida_init` | تهيئة IDA ديناميكي |
| `ida_is_empty` | هل الـ IDA فارغ؟ |

---

### Group 1: Initialization & Static Definition

هذه المجموعة مسؤولة عن إعداد الـ `struct idr` قبل أي استخدام. الـ IDR مبني فوق الـ **radix tree** (المُعاد تسمية كـ xarray داخلياً) مع marker خاص `IDR_RT_MARKER` يميزه عن الـ xarray العادي ويفعّل تتبع free nodes.

---

#### `idr_init_base`

```c
static inline void idr_init_base(struct idr *idr, int base)
{
    INIT_RADIX_TREE(&idr->idr_rt, IDR_RT_MARKER);
    idr->idr_base = base;
    idr->idr_next = 0;
}
```

بتهيئ الـ `struct idr` ديناميكياً مع تحديد الـ base الابتدائي للـ IDs. بتستدعي `xa_init_flags` على الـ `idr_rt` مع الـ `IDR_RT_MARKER` اللي بيحدد نوع الـ tree وتفعيل الـ `IDR_FREE` tag لتتبع الـ free slots.

**Parameters:**
- `idr` — المؤشر للـ `struct idr` المراد تهيئتها
- `base` — أدنى قيمة يمكن تخصيصها كـ ID (مثلاً 1 لو عايز تتجنب ID=0)

**Return:** لا يرجع قيمة.

**Key details:** بعد الاستدعاء، الـ `idr_next = 0` دايماً بغض النظر عن الـ base — الـ `idr_next` بيتحول لموقع cyclic وليس الـ ID الأول.

**Caller:** كود التهيئة في أي subsystem يحتاج IDR ديناميكي بـ base غير صفري، مثلاً driver يريد IDs تبدأ من 1.

---

#### `idr_init`

```c
static inline void idr_init(struct idr *idr)
{
    idr_init_base(idr, 0);
}
```

Wrapper بسيط على `idr_init_base` بـ base=0. ده الاستخدام الأكثر شيوعاً.

**Parameters:**
- `idr` — المؤشر للـ `struct idr`

**Return:** لا يرجع قيمة.

**Caller:** أي كود يحتاج IDR يبدأ IDs من صفر.

---

#### `DEFINE_IDR(name)` / `IDR_INIT(name)` / `IDR_INIT_BASE(name, base)`

```c
#define DEFINE_IDR(name)        struct idr name = IDR_INIT(name)
#define IDR_INIT(name)          IDR_INIT_BASE(name, 0)
#define IDR_INIT_BASE(name, base) {                         \
    .idr_rt = RADIX_TREE_INIT(name, IDR_RT_MARKER),         \
    .idr_base = (base),                                      \
    .idr_next = 0,                                           \
}
```

`DEFINE_IDR` يعرّف IDR ساكن static ويهيئه في نفس الوقت — جاهز للاستخدام فوراً بدون `idr_init`. الـ `RADIX_TREE_INIT` بيتحول لـ `XARRAY_INIT` وبيضع الـ `IDR_RT_MARKER` في الـ `xa_flags`.

**Key details:** الـ `IDR_RT_MARKER` مركّب من `ROOT_IS_IDR` (بيخلي الـ radix tree يعرف إن ده IDR context) زائد الـ `IDR_FREE` tag flag (bit في الـ `xa_flags` يُستخدم لتتبع nodes فيها free space).

---

### Group 2: ID Allocation (Core)

المجموعة الجوهرية — كل وظيفة هنا بترسم ID عددي لـ pointer مخزون في الـ radix tree.

---

#### `idr_alloc`

```c
int idr_alloc(struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
```

بتخصص أول ID متاح في النطاق `[start, end)` وبتربطه بالـ `ptr`. لو `end == 0` يُعامَل كـ `INT_MAX + 1` (يعني مفيش حد أقصى فعلي).

**Parameters:**
- `idr` — الـ IDR handle
- `ptr` — الـ pointer المراد تخزينه (لازم يكون non-NULL وغير tagged — low bits = 00)
- `start` — أدنى ID مقبول (inclusive)
- `end` — أعلى ID مرفوض (exclusive)، 0 يعني INT_MAX+1
- `gfp` — memory allocation flags — يُستخدم لو الـ radix tree محتاج nodes جديدة

**Return:** الـ ID المخصص (≥ 0) في حالة النجاح، أو:
- `-EINVAL` — لو `start` أكبر من `end` أو الـ `ptr` invalid
- `-ENOSPC` — لو مفيش IDs متاحة في النطاق
- `-ENOMEM` — لو الذاكرة انتهت أثناء توسيع الـ tree

**Key details:** الـ caller مسؤول عن الـ locking. داخلياً بتعمل `xa_alloc()` مع تحويل الـ [start, end) لـ xa_limit. يجب الاستدعاء مع `idr_lock` أو أي spinlock مناسب إلا لو الـ caller ضامن serialization.

**Pseudocode flow:**
```
idr_alloc(idr, ptr, start, end, gfp):
    id = idr_alloc_u32 or xa_alloc internally
        → calculates range from (idr_base + start) to (idr_base + end - 1)
        → xa_alloc finds first free slot in radix tree
        → stores ptr at that slot
        → updates IDR_FREE tag if node becomes full
    return (allocated_xa_index - idr_base)
```

**Caller:** كود يحتاج ID فريد لـ resource — مثلاً file descriptor تخصيص (في بعض الـ subsystems)، أو device minor number.

---

#### `idr_alloc_u32`

```c
int __must_check idr_alloc_u32(struct idr *idr, void *ptr, u32 *id,
                                unsigned long max, gfp_t gfp);
```

نسخة متخصصة من `idr_alloc` بتكتب الـ ID في `*id` بدل ما ترجعه كـ return value. مفيدة لما الـ caller عنده `u32 *` جاهز.

**Parameters:**
- `idr` — الـ IDR handle
- `ptr` — الـ pointer المراد تخزينه
- `id` — [in/out] مؤشر لـ u32 — الـ input بيحدد الـ `start`، والـ output بيحمل الـ ID المخصص
- `max` — الحد الأقصى للـ ID (inclusive)
- `gfp` — memory allocation flags

**Return:** 0 في النجاح، أو error code سالب.

**Key details:** الـ `__must_check` attribute يجبر الـ caller على فحص الـ return value. الـ `*id` كـ input قيمته هي الـ `start` — بيسمح بـ "hint" لأين يبدأ البحث. مفيد في الـ DRM subsystem وغيره.

---

#### `idr_alloc_cyclic`

```c
int idr_alloc_cyclic(struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
```

بتخصص ID بطريقة cyclic — بتبدأ البحث من `idr->idr_next` بدل الـ `start` دايماً. لما توصل للنهاية، بتلف من الأول. الهدف هو توزيع الـ IDs على مدار الزمن بدل إعادة استخدام نفس الـ IDs الصغيرة باستمرار.

**Parameters:**
- `idr` — الـ IDR handle
- `ptr` — الـ pointer المراد تخزينه
- `start` — الحد الأدنى للنطاق
- `end` — الحد الأقصى (exclusive)
- `gfp` — memory allocation flags

**Return:** الـ ID المخصص أو error code سالب.

**Key details:** بعد نجاح التخصيص، `idr->idr_next` يُحدَّث لـ `id + 1`. الـ `idr_get_cursor`/`idr_set_cursor` بيسمحوا بالقراءة/الكتابة على `idr_next` بأمان باستخدام `READ_ONCE`/`WRITE_ONCE`.

**Caller:** مثالي لتخصيص connection IDs أو transaction IDs حيث إعادة الاستخدام السريع للـ IDs القديمة غير مرغوب فيها — مثلاً TCP port allocation patterns.

---

### Group 3: Lookup & Cursor Management

---

#### `idr_find`

```c
void *idr_find(const struct idr *idr, unsigned long id);
```

بتبحث عن الـ pointer المرتبط بالـ `id`. دي الـ **hot path** — بتُستخدم بشكل مكثف وصُممت لتكون lockless مع RCU.

**Parameters:**
- `idr` — الـ IDR handle (const — مش بتعدل فيه)
- `id` — الـ ID المراد البحث عنه

**Return:** الـ pointer المخزون، أو `NULL` لو الـ ID غير موجود.

**Key details:** يمكن استدعاؤها بدون lock داخل `rcu_read_lock()`. الـ caller مسؤول عن إدارة lifetime الـ objects — لو الـ object ممكن يُحذف بـ RCU، الـ caller يستخدم `rcu_dereference()` مناسب. لا تحتاج `idr_lock` لو استُخدمت مع RCU.

**Caller:**
```c
rcu_read_lock();
ptr = idr_find(&my_idr, id);
if (ptr)
    do_something(ptr); /* ptr still valid inside RCU section */
rcu_read_unlock();
```

---

#### `idr_get_cursor` / `idr_set_cursor`

```c
static inline unsigned int idr_get_cursor(const struct idr *idr)
{
    return READ_ONCE(idr->idr_next);
}

static inline void idr_set_cursor(struct idr *idr, unsigned int val)
{
    WRITE_ONCE(idr->idr_next, val);
}
```

`idr_get_cursor` بتقرأ الموقع الحالي للـ cyclic allocator، و`idr_set_cursor` بتضبطه. الـ `READ_ONCE`/`WRITE_ONCE` بيمنعوا الـ compiler من إعادة ترتيب أو دمج الـ accesses.

**Key details:** `idr_next` مش محمي بـ lock في الـ cyclic path — الـ `WRITE_ONCE`/`READ_ONCE` كافيين لمنع الـ data races في بعض السياقات. لو الـ caller يريد atomicity حقيقية، يستخدم `idr_lock`.

---

### Group 4: Iteration

هذه المجموعة تتيح المرور على كل الـ entries في الـ IDR بشكل منظم.

---

#### `idr_for_each`

```c
int idr_for_each(const struct idr *idr,
                 int (*fn)(int id, void *p, void *data), void *data);
```

بتمر على كل الـ entries في الـ IDR وبتستدعي الـ callback `fn` لكل واحد. لو الـ callback رجع قيمة سالبة، الـ iteration بتوقف.

**Parameters:**
- `idr` — الـ IDR handle
- `fn` — callback function: بتاخد الـ `id`، الـ `pointer`، والـ `data`
- `data` — opaque data يُمرَّر للـ callback

**Return:** 0 لو الـ iteration اكتملت، أو قيمة الـ return من الـ callback الأخير لو كانت سالبة.

**Key details:** يُستخدم في cleanup paths أو debugging. الـ caller يحمل الـ lock أثناء الـ iteration إذا كان الـ IDR قد يُعدَّل من thread آخر.

---

#### `idr_get_next` / `idr_get_next_ul`

```c
void *idr_get_next(struct idr *idr, int *nextid);
void *idr_get_next_ul(struct idr *idr, unsigned long *nextid);
```

بترجع الـ pointer للـ entry التالي من الـ `*nextid` فأكثر، وبتحدث `*nextid` ليحمل الـ ID الفعلي للـ entry المُعاد.

**Parameters:**
- `idr` — الـ IDR handle
- `nextid` — [in/out] الـ ID للبدء منه؛ يُحدَّث ليصبح الـ ID الفعلي للـ entry المُعاد

**Return:** الـ pointer، أو `NULL` لو مافيش entries بعد الموقع المحدد.

**Key details:** الـ `idr_get_next_ul` للنطاق الكامل `unsigned long`، مفيد لو الـ IDs كبيرة. الفرق الجوهري: `idr_get_next_ul` أكثر أماناً ضد الـ wraparound في بعض التطبيقات.

---

#### Iteration Macros

```c
/* يمر على كل entry من id=0 */
#define idr_for_each_entry(idr, entry, id)                  \
    for (id = 0; ((entry) = idr_get_next(idr, &(id))) != NULL; id += 1U)

/* نسخة unsigned long مع tmp لتجنب wraparound */
#define idr_for_each_entry_ul(idr, entry, tmp, id)              \
    for (tmp = 0, id = 0;                                        \
         ((entry) = tmp <= id ? idr_get_next_ul(idr, &(id)) : NULL) != NULL; \
         tmp = id, ++id)

/* استئناف iteration من موقع id محدد */
#define idr_for_each_entry_continue(idr, entry, id)             \
    for ((entry) = idr_get_next((idr), &(id));                  \
         entry;                                                  \
         ++id, (entry) = idr_get_next((idr), &(id)))

/* استئناف iteration بـ unsigned long */
#define idr_for_each_entry_continue_ul(idr, entry, tmp, id)     \
    for (tmp = id;                                               \
         ((entry) = tmp <= id ? idr_get_next_ul(idr, &(id)) : NULL) != NULL; \
         tmp = id, ++id)
```

الـ `tmp` في الـ `_ul` variants بيحل مشكلة الـ wraparound: لو `idr_get_next_ul` حدّثت `id` لقيمة أصغر (overflow)، الـ condition `tmp <= id` تكتشف ده وتوقف الـ loop. ده pattern دقيق يتجنب حلقات لا نهائية.

**مثال عملي:**
```c
struct my_device *dev;
int id;

/* iteration بسيطة */
idr_for_each_entry(&my_idr, dev, id) {
    dev_info(&dev->dev, "found device id=%d\n", id);
}

/* استئناف من id=100 فأكثر */
id = 100;
idr_for_each_entry_continue(&my_idr, dev, id) {
    process_device(dev);
}
```

---

### Group 5: Modification — Replace & Remove

---

#### `idr_replace`

```c
void *idr_replace(struct idr *idr, void *ptr, unsigned long id);
```

بتستبدل الـ pointer المرتبط بـ ID موجود بالفعل. لو الـ ID غير موجود، ترجع error.

**Parameters:**
- `idr` — الـ IDR handle
- `ptr` — الـ pointer الجديد (لازم non-NULL وغير tagged)
- `id` — الـ ID المراد تحديث قيمته

**Return:** الـ pointer القديم في حالة النجاح، أو:
- `ERR_PTR(-ENOENT)` — لو الـ ID غير موجود
- `ERR_PTR(-EINVAL)` — لو الـ ptr غير صالح

**Key details:** مفيد في بعض الـ use cases حيث الـ ID يُحجز أولاً ثم يُملأ لاحقاً (two-phase initialization pattern).

**Caller:**
```c
/* Pattern: احجز ID أولاً بـ ptr=NULL_or_placeholder ثم استبدله */
old = idr_replace(&idr, real_ptr, id);
WARN_ON(IS_ERR(old));
```

---

#### `idr_remove`

```c
void *idr_remove(struct idr *idr, unsigned long id);
```

بتحذف الـ ID من الـ IDR وبترجع الـ pointer المرتبط به.

**Parameters:**
- `idr` — الـ IDR handle
- `id` — الـ ID المراد حذفه

**Return:** الـ pointer المخزون، أو `NULL` لو الـ ID غير موجود.

**Key details:** لا تحرر الـ object نفسه — الـ caller مسؤول عن تحرير الـ pointer المُعاد. مع RCU، التحرير لازم يكون بـ `call_rcu` أو `kfree_rcu` لضمان إن الـ readers الحاليين خلصوا.

**مثال:**
```c
idr_lock(&my_idr);
ptr = idr_remove(&my_idr, id);
idr_unlock(&my_idr);
if (ptr)
    kfree_rcu(ptr, rcu_head); /* RCU-safe free */
```

---

### Group 6: Preload Mechanism

الـ IDR بيحتاج أحياناً تخصيص ذاكرة لتوسيع الـ radix tree. لو الـ caller في atomic context (يعني `GFP_ATOMIC` مش مناسب أو مرفوض)، يستخدم الـ preload mechanism.

---

#### `idr_preload`

```c
void idr_preload(gfp_t gfp_mask);
```

بتخصص nodes مسبقاً للـ radix tree وبتخزنها في الـ per-CPU preload list. بتعمل `local_lock` على الـ `radix_tree_preloads.lock` وبتأجل الـ preemption.

**Parameters:**
- `gfp_mask` — memory allocation flags للتخصيص المسبق (يمكن `GFP_KERNEL` هنا لأننا خارج الـ lock)

**Return:** لا يرجع قيمة.

**Key details:** لازم يكون متبوعاً بـ `idr_preload_end()`. بين الاستدعاءين، الـ preemption مؤجل والـ caller يمسك الـ `local_lock`. يجب الاستدعاء مع `GFP_KERNEL` (أو gfp مناسب) قبل أخذ أي spinlock.

---

#### `idr_preload_end`

```c
static inline void idr_preload_end(void)
{
    local_unlock(&radix_tree_preloads.lock);
}
```

بتحرر الـ `local_lock` على الـ per-CPU preload list وبتعيد تفعيل الـ preemption.

**Key details:** بعد `idr_preload_end()`، الـ nodes المخصصة مسبقاً قد تُفرَّغ لاحقاً. الـ pattern الصحيح:

```c
idr_preload(GFP_KERNEL);      /* prealloc nodes, disables preemption */
idr_lock(&my_idr);            /* take spinlock */
id = idr_alloc(&my_idr, ptr, 0, 0, GFP_ATOMIC); /* uses preloaded nodes */
idr_unlock(&my_idr);
idr_preload_end();            /* re-enable preemption */
```

**Caller:** كود يحتاج `idr_alloc` في سياق لا يسمح بتخصيص ذاكرة مع sleep.

---

### Group 7: Lifecycle — Cleanup & Destroy

---

#### `idr_destroy`

```c
void idr_destroy(struct idr *idr);
```

بتحرر كل الـ internal nodes للـ radix tree المستخدمة من الـ IDR. لا تحرر الـ objects المخزونة (الـ pointers) — الـ caller مسؤول عن تحريرها أولاً.

**Parameters:**
- `idr` — الـ IDR handle

**Return:** لا يرجع قيمة.

**Key details:** في debug builds أو مع `CONFIG_DEBUG_VM`، بتفحص إن الـ IDR فاضي فعلاً قبل التدمير. الـ pattern الصحيح:

```c
/* أولاً: احذف كل الـ entries يدوياً */
idr_for_each_entry(&my_idr, ptr, id)
    kfree(ptr);

/* ثانياً: دمّر الـ IDR نفسه */
idr_destroy(&my_idr);
```

**Caller:** في الـ module unload أو driver remove paths.

---

### Group 8: Scoped Cleanup (Cleanup Framework)

الـ kernel الحديث (منذ 6.x) أضاف cleanup.h لتجنب الـ "goto error" pattern. الـ IDR لديه دعم لهذا عبر `DEFINE_CLASS`.

---

#### `DEFINE_CLASS(idr_alloc, ...)` و `struct __class_idr`

```c
struct __class_idr {
    struct idr *idr;
    int id;
};

#define idr_null ((struct __class_idr){ NULL, -1 })
#define take_idr_id(id) __get_and_null(id, idr_null)

DEFINE_CLASS(idr_alloc, struct __class_idr,
     if (_T.id >= 0) idr_remove(_T.idr, _T.id),
     ((struct __class_idr){
        .idr = idr,
        .id = idr_alloc(idr, ptr, start, end, gfp),
     }),
     struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
```

`DEFINE_CLASS` يولد class اسمه `idr_alloc` ذو **automatic cleanup**: لما المتغير يخرج من الـ scope، الـ compiler يستدعي `idr_remove` تلقائياً (عبر `__attribute__((cleanup(...)))`). الـ `struct __class_idr` يحمل reference للـ IDR والـ ID المخصص معاً.

**الـ `_T`** في الـ cleanup expression هو الـ instance نفسه — الـ condition `_T.id >= 0` تضمن إن الـ cleanup يحدث فقط لو الـ allocation نجحت.

**`take_idr_id`** ينقل ownership — يقرأ الـ `struct __class_idr` ويصفّره لـ `idr_null`، فلا يحدث cleanup عند خروج الـ scope (نظير `no_free_ptr`).

**مثال عملي:**
```c
int register_device(struct idr *idr, struct device *dev)
{
    /* id_alloc class: تلقائياً يُحذف لو الـ function خرجت بـ error */
    CLASS(idr_alloc, slot)(idr, dev, 0, 0, GFP_KERNEL);
    if (slot.id < 0)
        return slot.id; /* cleanup: لا يحدث لأن id < 0 */

    int ret = device_add(dev);
    if (ret)
        return ret; /* cleanup: يستدعي idr_remove تلقائياً */

    /* نجاح — انقل ownership للخارج */
    dev->id = take_idr_id(slot).id;
    return 0; /* cleanup: slot أصبح null، مفيش idr_remove */
}
```

---

### Group 9: IDA — ID Allocator (Pointer-free Variant)

الـ **IDA** (ID Allocator) هو نسخة مبسطة من الـ IDR — بتخصص IDs عددية فقط بدون ربطها بـ pointers. مبني على الـ **xarray** مباشرة بدلاً من الـ radix tree wrapper.

```c
struct ida {
    struct xarray xa;  /* xarray مع XA_FLAGS_ALLOC | XA_FLAGS_LOCK_IRQ */
};
```

الـ `IDA_INIT_FLAGS` = `XA_FLAGS_LOCK_IRQ | XA_FLAGS_ALLOC` — هذا يعني:
- الـ internal locking يستخدم IRQ-safe spinlock
- الـ xarray في وضع allocation (يتيح `xa_alloc`)

---

#### `ida_alloc_range`

```c
int ida_alloc_range(struct ida *ida, unsigned int min, unsigned int max, gfp_t gfp);
```

الـ core function للـ IDA — بتخصص أول ID متاح في النطاق `[min, max]`.

**Parameters:**
- `ida` — الـ IDA handle
- `min` — الحد الأدنى للـ ID (inclusive)
- `max` — الحد الأقصى للـ ID (inclusive)
- `gfp` — memory allocation flags

**Return:** الـ ID المخصص (≥ 0)، أو `-ENOMEM`/`-ENOSPC`.

**Key details:** الـ IDA internal locking مدمج (بسبب `XA_FLAGS_LOCK_IRQ`) — الـ caller لا يحتاج lock خارجي. آمنة للاستدعاء من أي context.

---

#### `ida_alloc` / `ida_alloc_min` / `ida_alloc_max`

```c
/* من 0 إلى INT_MAX */
static inline int ida_alloc(struct ida *ida, gfp_t gfp)
{
    return ida_alloc_range(ida, 0, ~0, gfp);
}

/* من min إلى INT_MAX */
static inline int ida_alloc_min(struct ida *ida, unsigned int min, gfp_t gfp)
{
    return ida_alloc_range(ida, min, ~0, gfp);
}

/* من 0 إلى max */
static inline int ida_alloc_max(struct ida *ida, unsigned int max, gfp_t gfp)
{
    return ida_alloc_range(ida, 0, max, gfp);
}
```

Convenience wrappers — كل واحدة بتستدعي `ida_alloc_range` مع قيم افتراضية مناسبة.

**Return:** الـ ID المخصص أو error code سالب.

**Caller:** أي كود يحتاج ID فريد بدون pointer — مثلاً device minor numbers، network interface indices، إلخ.

**مثال:**
```c
/* تخصيص minor number لـ device */
int minor = ida_alloc_max(&minors_ida, MINORMASK, GFP_KERNEL);
if (minor < 0)
    return minor;
```

---

#### `ida_free`

```c
void ida_free(struct ida *ida, unsigned int id);
```

بتحرر ID مخصص مسبقاً. الـ internal locking مدمج.

**Parameters:**
- `ida` — الـ IDA handle
- `id` — الـ ID المراد تحريره

**Return:** لا يرجع قيمة.

**Key details:** في debug builds، بتـ WARN لو الـ ID لم يكن مخصصاً فعلاً. آمنة من أي context.

---

#### `ida_destroy`

```c
void ida_destroy(struct ida *ida);
```

بتحرر كل موارد الـ IDA الداخلية. مثل `idr_destroy`، لا تحرر أي objects خارجية (الـ IDA أصلاً مش بتخزن pointers).

**Caller:** في cleanup paths — module unload أو subsystem shutdown.

---

#### `ida_find_first_range` / `ida_find_first`

```c
int ida_find_first_range(struct ida *ida, unsigned int min, unsigned int max);

static inline int ida_find_first(struct ida *ida)
{
    return ida_find_first_range(ida, 0, ~0);
}
```

بتبحث عن أول ID **مخصص** (وليس أول ID حر) في النطاق `[min, max]`.

**Return:** الـ ID الأول المخصص، أو `-ENOENT` لو مافيش.

**Key details:** مفيدة في debugging أو لمعرفة هل الـ IDA يحتوي entries. `ida_exists` مبنية عليها:

```c
static inline bool ida_exists(struct ida *ida, unsigned int id)
{
    return ida_find_first_range(ida, id, id) == id;
}
```

---

#### `ida_init` / `ida_is_empty`

```c
static inline void ida_init(struct ida *ida)
{
    xa_init_flags(&ida->xa, IDA_INIT_FLAGS);
}

static inline bool ida_is_empty(const struct ida *ida)
{
    return xa_empty(&ida->xa);
}
```

`ida_init` للتهيئة الديناميكية (بديل `DEFINE_IDA`)، و`ida_is_empty` للتحقق من إن الـ IDA فارغ تماماً.

---

### Group 10: Locking Macros

كل الـ locking macros للـ IDR هي wrappers مباشرة على الـ xarray locking API — لأن الـ `idr_rt` هو في الحقيقة `struct xarray` (بعد الـ alias `radix_tree_root = xarray`).

```c
#define idr_lock(idr)           xa_lock(&(idr)->idr_rt)
#define idr_unlock(idr)         xa_unlock(&(idr)->idr_rt)
#define idr_lock_bh(idr)        xa_lock_bh(&(idr)->idr_rt)
#define idr_unlock_bh(idr)      xa_unlock_bh(&(idr)->idr_rt)
#define idr_lock_irq(idr)       xa_lock_irq(&(idr)->idr_rt)
#define idr_unlock_irq(idr)     xa_unlock_irq(&(idr)->idr_rt)
#define idr_lock_irqsave(idr, flags)    xa_lock_irqsave(&(idr)->idr_rt, flags)
#define idr_unlock_irqrestore(idr, flags) xa_unlock_irqrestore(&(idr)->idr_rt, flags)
```

| Variant | متى تُستخدم |
|---|---|
| `idr_lock/unlock` | process context، preemption مسموح |
| `idr_lock_bh/unlock_bh` | لو الـ IDR يُستخدم من softirq أيضاً |
| `idr_lock_irq/unlock_irq` | لو الـ IDR يُستخدم من interrupt handler |
| `idr_lock_irqsave/irqrestore` | لو IRQ state يحتاج حفظ (nested contexts) |

**Key details:** الـ IDA لا تحتاج هذه الـ macros — لديها internal locking مدمج عبر `XA_FLAGS_LOCK_IRQ`.

---

### Summary: IDR vs IDA

| الميزة | IDR | IDA |
|---|---|---|
| يخزن pointer | نعم | لا |
| Internal locking | لا (الـ caller مسؤول) | نعم (XA_FLAGS_LOCK_IRQ) |
| Base layer | radix_tree (= xarray alias) | xarray مباشرة |
| Cyclic allocation | نعم (`idr_alloc_cyclic`) | لا |
| RCU lockless lookup | نعم (`idr_find`) | لا ينطبق |
| Scoped cleanup | نعم (DEFINE_CLASS) | لا |
| متى تُستخدم | ID → pointer mapping | IDs فقط بدون data |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs — الـ Entries المرتبطة بالـ IDR/IDA

الـ IDR والـ IDA مبنيان فوق الـ **radix tree** (قديماً) والـ **XArray** (IDA حالياً)، ومش ليهم entries مباشرة في debugfs بس ممكن تلاقي معلوماتهم عن طريق:

```bash
# اتأكد إن debugfs متماونت
mount -t debugfs none /sys/kernel/debug

# شوف الـ XArray state اللي بيستخدمه IDA
ls /sys/kernel/debug/
```

الـ IDA داخلياً بيستخدم `struct xarray`، وبعض الـ subsystems بتعمل expose للـ IDA بتاعتها عبر debugfs:

```bash
# مثال: PID namespace IDR
cat /sys/kernel/debug/pid_namespace

# شوف كل entries جوه debugfs متعلقة بالـ IDs
find /sys/kernel/debug -name "*idr*" -o -name "*ida*" 2>/dev/null
```

طريقة قراءة state الـ IDR من kernel module:

```c
/* Dump all entries in an IDR via pr_debug */
static void debug_dump_idr(struct idr *idr, const char *name)
{
    void *entry;
    int id;

    pr_debug("%s IDR dump:\n", name);
    idr_for_each_entry(idr, entry, id)
        pr_debug("  id=%d -> ptr=%px\n", id, entry);
    pr_debug("%s idr_base=%u idr_next=%u\n",
             name, idr->idr_base, idr->idr_next);
}
```

---

#### 2. sysfs — الـ Entries المفيدة

الـ IDR/IDA نفسهم ملهمش sysfs nodes، بس الـ subsystems اللي بتستخدمهم بتعمل expose:

```bash
# PID max — بيتحكم في حجم الـ IDR الخاصة بالـ PIDs
cat /proc/sys/kernel/pid_max

# عدد الـ IDs المستخدمة حالياً (عدد الـ processes)
cat /proc/sys/kernel/threads-max

# شوف IDs المستخدمة في الـ network devices
ls /sys/class/net/

# شوف IDs المستخدمة في الـ block devices
ls /sys/class/block/

# perf event IDs
ls /sys/bus/event_source/devices/
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

الـ IDR/IDA مش ليهم tracepoints مخصوصة، بس نقدر نتتبع الـ functions بتاعتهم مباشرة:

```bash
# تفعيل function tracer للـ IDR functions
cd /sys/kernel/debug/tracing

echo function > current_tracer

# تتبع كل الـ idr_* functions
echo 'idr_alloc' >> set_ftrace_filter
echo 'idr_remove' >> set_ftrace_filter
echo 'idr_find' >> set_ftrace_filter
echo 'idr_replace' >> set_ftrace_filter
echo 'idr_alloc_cyclic' >> set_ftrace_filter
echo 'idr_destroy' >> set_ftrace_filter

# تفعيل التتبع
echo 1 > tracing_on
cat trace_pipe
```

```bash
# تتبع كل الـ ida_* functions
echo 'ida_alloc_range' >> set_ftrace_filter
echo 'ida_free' >> set_ftrace_filter
echo 'ida_destroy' >> set_ftrace_filter
echo 'ida_find_first_range' >> set_ftrace_filter

# استخدام function_graph لشوف الـ call tree
echo function_graph > current_tracer
echo 'ida_alloc_range' > set_graph_function
echo 1 > tracing_on
sleep 2
cat trace | head -100
```

```bash
# تتبع الـ radix_tree functions اللي بيستخدمها IDR
echo 'radix_tree_insert' >> set_ftrace_filter
echo 'radix_tree_delete' >> set_ftrace_filter
echo 'radix_tree_lookup' >> set_ftrace_filter

# تتبع الـ xarray functions اللي بيستخدمها IDA
echo 'xa_alloc' >> set_ftrace_filter
echo 'xa_erase' >> set_ftrace_filter
echo 'xas_find' >> set_ftrace_filter
```

---

#### 4. printk / Dynamic Debug

```bash
# تفعيل dynamic debug لأي module بيستخدم IDR
# مثال لـ module اسمه mydrv
echo 'module mydrv +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لملف معين
echo 'file lib/idr.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file lib/radix-tree.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file lib/xarray.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug entries الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep -E 'idr|ida|xarray'

# تفعيل كل الـ debug flags (p=print, f=func, l=line, m=module, t=thread)
echo 'file lib/idr.c +pflt' > /sys/kernel/debug/dynamic_debug/control
```

في الكود بتاعك:

```c
#define pr_fmt(fmt) "mydrv-idr: " fmt

/* استخدم pr_debug — بيشتغل مع dynamic debug */
pr_debug("allocated id=%d for ptr=%px\n", id, ptr);
pr_debug("idr state: base=%u next=%u empty=%d\n",
         idr->idr_base, idr->idr_next, idr_is_empty(idr));
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config Option | الهدف |
|---|---|
| `CONFIG_DEBUG_RADIX_TREE` | تفعيل assertions جوه الـ radix tree (أساس IDR) |
| `CONFIG_XARRAY_MULTI` | دعم الـ multi-index entries في XArray (أساس IDA) |
| `CONFIG_DEBUG_SLAB` | كشف memory corruption في allocations الـ IDR/IDA |
| `CONFIG_KASAN` | **Kernel Address Sanitizer** — بيكشف use-after-free وbuffer overflow |
| `CONFIG_KASAN_INLINE` | نسخة أسرع من KASAN |
| `CONFIG_UBSAN` | كشف undefined behavior زي integer overflow |
| `CONFIG_LOCKDEP` | تتبع locking violations في `idr_lock`/`idr_unlock` |
| `CONFIG_PROVE_LOCKING` | إثبات correctness الـ locking |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | كشف sleeping في atomic context أثناء `idr_preload` |
| `CONFIG_FTRACE` | تفعيل function tracing |
| `CONFIG_FUNCTION_TRACER` | لازم يكون enabled عشان تتتبع `idr_alloc` |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug activation |
| `CONFIG_MEMCG` | تتبع memory usage للـ IDA/IDR allocations |
| `CONFIG_DEBUG_MEMORY_INIT` | كشف uninitialized memory في الـ bitmaps |
| `CONFIG_KMEMCHECK` | فحص memory accesses (kernels قديمة) |

```bash
# تحقق من الـ config الحالي
grep -E 'CONFIG_DEBUG_RADIX|CONFIG_KASAN|CONFIG_LOCKDEP|CONFIG_DYNAMIC_DEBUG' \
    /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ crash/gdb — dump الـ IDR state من kernel dump:**

```bash
# في crash utility
crash> struct idr <address>
crash> struct radix_tree_root <idr.idr_rt address>

# طباعة كل entries في IDR
crash> idr_for_each <address>
```

**الـ /proc interfaces المفيدة:**

```bash
# شوف كل الـ allocated PIDs (IDR مشهور)
cat /proc/sys/kernel/ns_last_pid

# شوف الـ inode numbers (بتستخدم IDA)
stat /proc/self | grep Inode

# فحص الـ file descriptor IDs
ls -la /proc/self/fd/
cat /proc/self/fdinfo/0
```

**أداة pahole لفهم layout الـ struct:**

```bash
pahole -C idr vmlinux
pahole -C ida vmlinux
pahole -C ida_bitmap vmlinux
```

**Output متوقع:**

```
struct idr {
    struct radix_tree_root     idr_rt;              /*     0    16 */
    unsigned int               idr_base;            /*    16     4 */
    unsigned int               idr_next;            /*    20     4 */
    /* size: 24, cachelines: 1, members: 3 */
};
```

**سكريبت Python لـ gdb يطبع محتوى IDR:**

```python
# في gdb: source idr_debug.py
import gdb

def dump_idr(idr_addr):
    idr = gdb.Value(idr_addr).cast(gdb.lookup_type('struct idr').pointer())
    print(f"IDR at {idr_addr:#x}")
    print(f"  idr_base = {int(idr['idr_base'])}")
    print(f"  idr_next = {int(idr['idr_next'])}")
```

---

#### 7. جدول الـ Error Messages الشائعة

| Error Message / Code | المعنى | الحل |
|---|---|---|
| `idr_alloc: -ENOMEM` | ما فيش memory كفاية للـ allocation | زود الـ memory أو استخدم `idr_preload()` قبل الـ alloc |
| `idr_alloc: -ENOSPC` | الـ ID range اتملأ (عادةً `start >= end`) | راجع قيم `start` و`end`، أو استخدم `INT_MAX` كـ end |
| `idr_alloc: -EINVAL` | `start < 0` أو `end <= start` | راجع الـ parameters |
| `ida_alloc_range: -ENOSPC` | كل الـ IDs في الـ range اتخدت | حدد range أكبر أو حرر IDs قديمة |
| `WARN: idr_remove: ID not found` | بتحاول تمسح ID مش موجود | تأكد من الـ ID قبل الـ remove، راجع double-free |
| `BUG: sleeping function called from invalid context` | استدعاء `idr_alloc` بـ `GFP_KERNEL` جوه atomic context | استخدم `GFP_ATOMIC` أو `idr_preload()` قبل الـ lock |
| `kernel BUG at lib/radix-tree.c` | corruption في الـ radix tree الداخلية | فعّل `CONFIG_DEBUG_RADIX_TREE` و`CONFIG_KASAN` |
| `use-after-free in idr_find` | وصول للـ IDR بعد `idr_destroy()` | راجع lifetime management، استخدم RCU صح |
| `possible circular locking dependency` | `lockdep` اكتشف مشكلة في ترتيب `idr_lock` | راجع locking order في الكود |
| `idr_replace: -ENOENT` | الـ ID مش موجود في الـ IDR | تأكد من وجود الـ ID قبل الـ replace |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* التحقق من صحة الـ ID بعد الـ alloc */
id = idr_alloc(&my_idr, ptr, 0, MAX_ID, GFP_KERNEL);
WARN_ON(id < 0);  /* يطبع stack trace لو فشل */

/* التحقق من إن الـ IDR مش فاضي قبل operations حساسة */
WARN_ON(idr_is_empty(&my_idr));

/* التحقق من إن الـ pointer المرجع مش NULL */
entry = idr_find(&my_idr, id);
if (WARN_ON(!entry)) {
    dump_stack();  /* طباعة كامل الـ call stack */
    return -ENOENT;
}

/* في الـ cleanup path — تأكد من إن الـ IDR فاضي */
idr_destroy(&my_idr);
WARN_ON(!idr_is_empty(&my_idr));  /* لو فيه leak

/* حماية ضد cyclic allocator wraparound */
WARN_ON(idr_get_cursor(&my_idr) == 0 && !idr_is_empty(&my_idr));

/* في الـ RCU read path — تأكد من الـ lock */
WARN_ON_ONCE(!rcu_read_lock_held());
entry = idr_find(&my_idr, id);
```

---

### Hardware Level

---

#### 1. التحقق من إن الـ Hardware State بيطابق الـ Kernel State

الـ IDR/IDA هي purely software data structures، بس الـ subsystems اللي بتستخدمها بتمثل hardware resources (device IDs, interrupt numbers, DMA channels).

```bash
# مثال: verify إن الـ IRQ IDs في الـ IDR بتطابق الـ hardware
cat /proc/interrupts
# كل سطر: IRQ_number / CPU_counts / controller / device_name

# قارن مع الـ IDR في الـ irq_desc
cat /proc/irq/*/spurious 2>/dev/null

# verify الـ device IDs في الـ USB
lsusb -v | grep -E 'idVendor|idProduct|bConfigurationValue'

# قارن مع ما الـ kernel allocate
ls /sys/bus/usb/devices/

# verify network interface indices (بيستخدموا IDA)
ip link show | grep -E '^[0-9]+:'
cat /sys/class/net/eth0/ifindex
```

---

#### 2. Register Dump Techniques

الـ IDR نفسه ملوش registers، بس لو كنت بتتتبع hardware IDs:

```bash
# قراءة registers بتستخدم devmem2
# مثال: قراءة PCI device ID register
devmem2 0xF0000000 w  # Physical address من /proc/iomem

# بديل باستخدام /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xF0000000 / 4)) 2>/dev/null | xxd

# شوف الـ physical memory map
cat /proc/iomem | head -30

# قراءة PCI config space (بيحتوي على device IDs)
setpci -s 00:00.0 0x00.w  # Vendor ID
setpci -s 00:00.0 0x02.w  # Device ID
```

```bash
# شوف DMA channel IDs
cat /proc/dma

# شوف الـ resource allocation لكل الـ devices
cat /proc/bus/pci/devices | awk '{print $1, $2}' | head -20
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو بتـ debug hardware ID assignment في real-time:

```
Logic Analyzer Setup لـ I2C device IDs:
┌─────────────┐
│  SDA ───────┤ CH1 (Data)
│  SCL ───────┤ CH2 (Clock)
│  IRQ ───────┤ CH3 (Interrupt — بيحصل لما ID يتخصص)
└─────────────┘

Protocol Decode Settings:
- I2C: 400kHz standard
- Trigger: ACK bit low بعد address byte
- Capture: 1M samples @ 10MHz
```

```
للـ USB Device Enumeration (ID assignment):
- Trigger على D+ rising edge (reset release)
- تتبع SET_ADDRESS control transfer
- الـ ID الجديد بيظهر في ADDR field (7-bit)
- قارن مع /sys/bus/usb/devices/*/devnum
```

---

#### 4. Hardware Issues → Kernel Log Patterns

| Hardware Issue | Kernel Log Pattern | التشخيص |
|---|---|---|
| Device enumeration فاشل | `usb: device not accepting address X` | الـ IDR allocate ID بس الـ device ما رد |
| IRQ conflict | `IRQ X: nobody cared` | الـ IDR allocate IRQ موجود بالفعل |
| DMA ID exhaustion | `dma_alloc_coherent: no memory` | الـ IDA للـ DMA channels اتملأت |
| PCIe device hotplug | `pci: BAR X: assigned [mem ...]` | الـ IDR assign resource ID جديد |
| GPU context leak | `drm: leaking X contexts` | الـ IDA للـ GPU contexts ما اتحررتش |
| Network device rename | `renamed from eth0 to ens3` | الـ IDA لـ ifindex اتغير |

```bash
# شوف الـ kernel log للـ ID-related errors
dmesg | grep -E 'IDR|IDA|idr|ida|ENOSPC|ENOMEM' | tail -20
dmesg | grep -E 'address [0-9]+|device [0-9]+' | tail -20

# تتبع device enumeration في real-time
dmesg -w | grep -E 'new.*device|probe|address'
```

---

#### 5. Device Tree Debugging

الـ IDR/IDA بيتخصصوا IDs للـ devices اللي الـ DT بيعرفها:

```bash
# شوف الـ DT nodes اللي ليها IDs
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -E 'reg|interrupts' | head -30

# تحقق من الـ interrupt numbers في الـ DT مقابل الـ kernel
cat /proc/device-tree/interrupt-controller@*/\#interrupt-cells 2>/dev/null | xxd

# شوف الـ DT aliases (بتستخدم IDR لتحديد الترتيب)
ls /proc/device-tree/aliases/
cat /proc/device-tree/aliases/serial0 2>/dev/null

# compare DT interrupt assignments مع الـ kernel
cat /proc/interrupts
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A2 'interrupts'
```

```bash
# فحص DT overlay conflicts في IDs
dmesg | grep -E 'dt|devicetree|overlay|alias' | grep -iE 'error|conflict|warn'

# تحقق من الـ reg property (hardware address → kernel ID mapping)
find /proc/device-tree -name 'reg' -exec sh -c \
    'echo "$(dirname {}): $(cat {} | xxd -p)"' \; 2>/dev/null | head -20
```

---

### Practical Commands

---

#### مجموعة الأوامر الجاهزة للنسخ

**1. فحص شامل لحالة الـ IDR/IDA في الـ system:**

```bash
#!/bin/bash
# idr_health_check.sh — فحص شامل

echo "=== PID IDR Status ==="
echo "Max PIDs: $(cat /proc/sys/kernel/pid_max)"
echo "Current PIDs: $(ls /proc | grep -c '^[0-9]')"

echo ""
echo "=== Network IDA Status ==="
echo "Interface count: $(ip link show | grep -c '^[0-9]')"
ip link show | grep -E '^[0-9]+:' | awk '{print $1, $2}'

echo ""
echo "=== IRQ IDR Status ==="
echo "Allocated IRQs: $(cat /proc/interrupts | tail -n +2 | wc -l)"
cat /proc/interrupts | head -5

echo ""
echo "=== File Descriptor IDA ==="
echo "Max FDs: $(cat /proc/sys/fs/file-max)"
echo "Open FDs: $(cat /proc/sys/fs/file-nr | awk '{print $1}')"
```

**2. تفعيل ftrace لـ IDR/IDA:**

```bash
#!/bin/bash
# trace_idr.sh

TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo function > $TRACE/current_tracer

# IDR functions
for func in idr_alloc idr_remove idr_find idr_replace \
            idr_alloc_cyclic idr_destroy idr_preload; do
    echo $func >> $TRACE/set_ftrace_filter
done

# IDA functions
for func in ida_alloc_range ida_free ida_destroy ida_find_first_range; do
    echo $func >> $TRACE/set_ftrace_filter
done

echo 1 > $TRACE/tracing_on
echo "Tracing IDR/IDA functions... Press Ctrl+C to stop"
cat $TRACE/trace_pipe
```

**3. تتبع كل الـ ID allocations في الـ system:**

```bash
#!/bin/bash
# watch_id_allocs.sh

echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'idr_alloc
idr_alloc_cyclic
ida_alloc_range' > /sys/kernel/debug/tracing/set_graph_function

echo 1 > /sys/kernel/debug/tracing/tracing_on
timeout 5 cat /sys/kernel/debug/tracing/trace_pipe
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

---

#### مثال على الـ Output وكيفية قراءته

**Output من `cat /sys/kernel/debug/tracing/trace_pipe`:**

```
          <...>-1234  [002] .... 12345.678: idr_alloc <-my_driver_probe
          <...>-1234  [002] .... 12345.679: idr_alloc+0x0/0x60
          <...>-1234  [002] .... 12345.680: <-- idr_alloc = 5
```

**التفسير:**
- `<...>-1234` — الـ process name والـ PID
- `[002]` — الـ CPU core
- `12345.678` — الـ timestamp بالثواني
- `idr_alloc <-my_driver_probe` — الـ function ومين استدعاها
- `= 5` — الـ return value (الـ ID المخصص)

**Output من `cat /proc/interrupts`:**

```
           CPU0       CPU1       CPU2       CPU3
  0:         46          0          0          0  IR-IO-APIC    2-edge      timer
  1:          0          0          0       8523  IR-IO-APIC    1-edge      i8042
 16:          0          0          0          0  IR-IO-APIC   16-level     ehci_hcd
NMI:          0          0          0          0   Non-maskable interrupts
```

**التفسير:**
- العمود الأول: الـ IRQ number (= ID في الـ IDR)
- الأعمدة التالية: عدد الـ interrupts على كل CPU
- `IR-IO-APIC`: نوع الـ interrupt controller
- `timer`/`i8042`: اسم الـ device المرتبط بالـ ID ده

**Output من `dynamic_debug` بعد التفعيل:**

```bash
# Command
echo 'file lib/idr.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | tail -20
```

```
[12345.678] idr_alloc: start=0 end=2147483647 base=0
[12345.679] idr_alloc: allocated id=42
[12345.680] idr_remove: removing id=42
```

**Output من `pahole -C idr vmlinux`:**

```
struct idr {
    struct radix_tree_root     idr_rt;              /*     0    16 */
    unsigned int               idr_base;            /*    16     4 */
    unsigned int               idr_next;            /*    20     4 */

    /* size: 24, cachelines: 1, members: 3 */
    /* last cacheline: 24 bytes */
};
```

**التفسير:**
- `idr_base` عند offset `16` — الـ base ID اللي الـ allocation بتبدأ منه
- `idr_next` عند offset `20` — الـ cursor الحالي لـ `idr_alloc_cyclic`
- الـ struct كلها `24 bytes` = بتدخل في cacheline واحدة (performance ممتاز)

---

#### تشخيص ID Leak (سيناريو كامل)

```bash
#!/bin/bash
# diagnose_id_leak.sh
# بيكتشف لو driver بيعمل alloc IDs من غير ما يحررها

echo "=== Before Test ==="
PID_BEFORE=$(ls /proc | grep -c '^[0-9]')
FD_BEFORE=$(cat /proc/sys/fs/file-nr | awk '{print $1}')
echo "PIDs: $PID_BEFORE | FDs: $FD_BEFORE"

# شغّل الـ workload المشبوه
modprobe suspicious_driver 2>/dev/null
sleep 2

echo "=== After Test ==="
PID_AFTER=$(ls /proc | grep -c '^[0-9]')
FD_AFTER=$(cat /proc/sys/fs/file-nr | awk '{print $1}')
echo "PIDs: $PID_AFTER | FDs: $FD_AFTER"

echo ""
echo "=== Delta ==="
echo "PID leak: $((PID_AFTER - PID_BEFORE))"
echo "FD leak:  $((FD_AFTER - FD_BEFORE))"

# لو فيه leak، شوف مين المسبب
if [ $((FD_AFTER - FD_BEFORE)) -gt 10 ]; then
    echo "POSSIBLE FD/ID LEAK DETECTED!"
    lsof | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
fi
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: تسريب IDs في driver الـ SPI على RK3562

#### العنوان
**ID leak** في SPI controller driver بيسبب exhaustion كامل للـ IDR بعد كام أسبوع من التشغيل.

#### السياق
بورد industrial gateway بيشتغل على **RK3562** SoC. الـ gateway بيتحكم في 8 أجهزة SPI (sensors + actuators). الـ driver بيعمل `idr_alloc()` لكل transaction عشان يديه ID فريد للـ DMA callback.

#### المشكلة
بعد تقريباً 3 أسابيع شغل مستمر، الـ gateway بيبطّل يقبل transactions جديدة. الـ dmesg بيظهر:

```
spi-rk3562 ff4e0000.spi: failed to allocate transaction ID: -ENOSPC
```

الـ IDR اتملى بـ IDs محدش عمل `idr_remove()` ليهم.

#### التحليل
الكود في الـ driver كان:

```c
/* allocate ID for each DMA transaction */
int id = idr_alloc(&spi_dev->tx_idr, tx_ctx, 0, 0, GFP_ATOMIC);
if (id < 0)
    return id;

tx_ctx->id = id;
/* submit DMA... */
```

المشكلة: في حالة الـ DMA error path، الكود بيعمل `return -EIO` من غير ما يعمل `idr_remove()`. كل transaction فاشلة بتسيب ID محجوز للأبد.

```c
/* idr_alloc() allocates from idr_base=0 upward */
/* struct idr { radix_tree_root idr_rt; unsigned int idr_base; unsigned int idr_next; } */
/* idr_next بيتحرك للأمام — مش بيرجع لحاله */
```

الـ IDR بيبحث عن أول slot فاضي ابتداءً من `idr_next`. لما الـ IDs القديمة مش اتحررت، الـ radix tree اتملأت والـ `idr_alloc()` رجعت `-ENOSPC`.

#### الحل

**الحل المباشر** — إصلاح كل error paths:

```c
static int spi_submit_tx(struct spi_dev *spi_dev, struct tx_ctx *tx_ctx)
{
    int id, ret;

    /* allocate ID before DMA submission */
    id = idr_alloc(&spi_dev->tx_idr, tx_ctx, 0, 0, GFP_ATOMIC);
    if (id < 0)
        return id;

    tx_ctx->id = id;

    ret = dma_submit(spi_dev->dma_chan, tx_ctx);
    if (ret < 0) {
        /* MUST free ID on every error path */
        idr_remove(&spi_dev->tx_idr, id);
        return ret;
    }

    return 0;
}
```

**الحل الأنظف** — استخدام `DEFINE_CLASS(idr_alloc, ...)` اللي موجود في `idr.h`:

```c
static int spi_submit_tx(struct spi_dev *spi_dev, struct tx_ctx *tx_ctx)
{
    /*
     * CLASS(idr_alloc) automatically calls idr_remove() on scope exit
     * if the ID was successfully allocated (id >= 0).
     * Definition in idr.h:
     *   DEFINE_CLASS(idr_alloc, struct __class_idr,
     *       if (_T.id >= 0) idr_remove(_T.idr, _T.id), ...)
     */
    CLASS(idr_alloc, handle)(&spi_dev->tx_idr, tx_ctx, 0, 0, GFP_ATOMIC);

    if (handle.id < 0)
        return handle.id;

    tx_ctx->id = handle.id;

    int ret = dma_submit(spi_dev->dma_chan, tx_ctx);
    if (ret < 0)
        return ret; /* idr_remove() called automatically here */

    /* success — take ownership of the ID, prevent auto-cleanup */
    take_idr_id(handle);
    return 0;
}
```

**Debug command** للتحقق قبل الكراش:

```bash
# عدد الـ IDs المحجوزة في كل وقت
cat /proc/driver/spi_rk3562/idr_stats

# أو عبر debugfs
ls /sys/kernel/debug/spi*/
```

#### الدرس المستفاد
كل `idr_alloc()` لازم يكون ليه `idr_remove()` مقابله في كل error path بدون استثناء. الـ `DEFINE_CLASS(idr_alloc)` الموجود في `idr.h` هو الحل الأنظف لأنه بيعمل cleanup تلقائي عند خروج الـ scope، زي الـ RAII في C++.

---

### السيناريو الثاني: race condition في USB device enumeration على STM32MP1

#### العنوان
**Race condition** بين `idr_alloc()` و`idr_find()` بدون lock صحيح في USB hub driver على **STM32MP1**.

#### السياق
منتج IoT sensor hub بيشتغل على **STM32MP1**. الـ hub بيدير حتى 7 USB devices (temperature sensors, gas sensors). الـ driver بيستخدم IDR عشان يربط كل USB device بـ internal context struct.

#### المشكلة
في stress testing بيتعمل fuzz على USB connect/disconnect، الـ system بيـ crash بـ NULL pointer dereference داخل الـ sensor callback:

```
BUG: kernel NULL pointer dereference, address: 0000000000000018
PC is at sensor_data_callback+0x34/0x80 [usb_sensor]
```

#### التحليل
الـ driver عنده thread واحد بيعمل enumerate للـ devices وthread تاني بيستقبل data:

```c
/* Thread 1: USB enumeration */
int id = idr_alloc(&hub->device_idr, dev_ctx, 1, MAX_DEVICES, GFP_KERNEL);
/* ... setup dev_ctx ... */
dev_ctx->ready = true;  /* <-- يحصل بعد idr_alloc */

/* Thread 2: data callback (lockless idr_find) */
/*
 * idr_find() safe بدون lock لو في rcu_read_lock()
 * لكن الـ dev_ctx ممكن يكون allocated ومش ready لسه
 * Doc في idr.h:
 *   "idr_find() is able to be called locklessly, using RCU.
 *    The caller must ensure calls to this function are made
 *    within rcu_read_lock() regions."
 */
rcu_read_lock();
ctx = idr_find(&hub->device_idr, id);
/* ctx مش NULL لكن dev_ctx->ready لسه false */
if (ctx->sensor_ops->read_data)  /* CRASH: sensor_ops = NULL */
    ctx->sensor_ops->read_data(ctx);
rcu_read_unlock();
```

الـ `idr_find()` بيرجع الـ pointer صح (مش NULL)، لكن الـ object نفسه مش initialized كامل لأن الـ `idr_alloc()` اتعمل قبل ما الـ setup يخلص.

#### الحل

**الحل الصح**: اعمل الـ insert في IDR بعد ما الـ object يبقى fully initialized:

```c
/* الصح: initialize الـ object الأول قبل idr_alloc */
static int usb_sensor_probe(struct usb_interface *intf)
{
    struct sensor_ctx *ctx;
    int id;

    ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
    if (!ctx)
        return -ENOMEM;

    /* fully initialize BEFORE making visible via IDR */
    ctx->sensor_ops = &stm32_sensor_ops;
    ctx->ready = true;
    init_completion(&ctx->done);

    /*
     * Only now insert into IDR — after this point,
     * idr_find() can return this pointer safely
     */
    idr_lock(&hub->device_idr);
    id = idr_alloc(&hub->device_idr, ctx, 1, MAX_DEVICES, GFP_NOWAIT);
    idr_unlock(&hub->device_idr);

    if (id < 0) {
        kfree(ctx);
        return id;
    }

    ctx->id = id;
    return 0;
}
```

**لو محتاج lockless find** — استخدم `smp_store_release()` / `smp_load_acquire()`:

```c
/* في الـ probe: */
smp_store_release(&ctx->ready, true);
/* بعدين اعمل idr_alloc */

/* في الـ callback: */
rcu_read_lock();
ctx = idr_find(&hub->device_idr, id);
if (ctx && smp_load_acquire(&ctx->ready))
    ctx->sensor_ops->read_data(ctx);
rcu_read_unlock();
```

#### الدرس المستفاد
الـ `idr_find()` في `idr.h` بيوضح صراحةً إن الـ RCU بيحمي الـ lookup بس مش الـ object نفسه. **ترتيب الـ initialization** بالنسبة لـ `idr_alloc()` critical جداً — الـ object لازم يكون ready قبل ما يبقى visible للـ concurrent readers.

---

### السيناريو الثالث: exhaustion الـ IDA في HDMI audio driver على i.MX8

#### العنوان
**IDA exhaustion** في HDMI audio channel allocator على **i.MX8** بيسبب صوت ينقطع في Android TV box.

#### السياق
Android TV box بيشتغل على **i.MX8MQ**. الـ HDMI audio driver بيستخدم **IDA** (مش IDR لأن مش محتاج يربط ID بـ pointer) عشان يدير الـ audio channel slots. كل Dolby Atmos stream بياخد slot.

#### المشكلة
بعد ساعات من تشغيل الـ streaming، الصوت ينقطع وفي الـ logcat:

```
audio_hdmi_imx8: ida_alloc_range failed: -ENOSPC
audio_hdmi_imx8: no available channel slots (max=32)
```

الـ IDA اتملأت بـ slots مش اتحررت.

#### التحليل

```c
/*
 * IDA definition in idr.h:
 * struct ida { struct xarray xa; };
 * #define IDA_INIT_FLAGS (XA_FLAGS_LOCK_IRQ | XA_FLAGS_ALLOC)
 *
 * IDA_BITMAP_BITS = (128 / sizeof(long)) * sizeof(long) * 8
 * على 64-bit: (128/8) * 8 * 8 = 1024 bits per chunk
 */

/* الـ driver كان بيعمل كده: */
static int hdmi_open_stream(struct hdmi_dev *dev)
{
    int slot;

    /*
     * ida_alloc_max() wraps ida_alloc_range(ida, 0, max, gfp)
     * بيديك أول ID متاح بين 0 و max
     */
    slot = ida_alloc_max(&dev->channel_ida, MAX_HDMI_CHANNELS - 1, GFP_KERNEL);
    if (slot < 0)
        return slot;

    dev->streams[slot].active = true;
    return slot;
}

static void hdmi_close_stream(struct hdmi_dev *dev, int slot)
{
    /* BUG: conditional free — مش دايماً بيتعمل */
    if (dev->streams[slot].eos_received)
        ida_free(&dev->channel_ida, slot);
    /* لو EOS مجاش (connection drop)، الـ slot بيفضل محجوز */
}
```

الـ problem: لما الـ TV يقطع من الـ HDMI قبل ما الـ EOS يوصل، الـ `eos_received` بيفضل false والـ `ida_free()` مش بيتعمل.

**Debug**:

```bash
# على الـ target (i.MX8)
# تحقق لو الـ IDA فاضية ولا لأ
# ida_is_empty() → xa_empty(&ida->xa)
echo "check IDA state" > /sys/kernel/debug/hdmi_audio/ida_dump

# عدد الـ active slots
cat /sys/kernel/debug/hdmi_audio/active_channels
```

#### الحل

```c
static void hdmi_close_stream(struct hdmi_dev *dev, int slot)
{
    if (slot < 0 || slot >= MAX_HDMI_CHANNELS)
        return;

    dev->streams[slot].active = false;
    dev->streams[slot].eos_received = false;

    /*
     * Always free the IDA slot regardless of EOS status.
     * ida_free() is safe to call from any context —
     * IDA_INIT_FLAGS includes XA_FLAGS_LOCK_IRQ
     */
    ida_free(&dev->channel_ida, slot);
}

/* وفي الـ disconnect handler كمان: */
static void hdmi_disconnect(struct hdmi_dev *dev)
{
    int slot;
    struct hdmi_stream *stream;

    /*
     * Walk all streams and free any still-allocated IDA entries.
     * ida_find_first_range() helps find allocated IDs:
     * static inline int ida_find_first(struct ida *ida)
     *     { return ida_find_first_range(ida, 0, ~0); }
     */
    while ((slot = ida_find_first(&dev->channel_ida)) >= 0) {
        stream = &dev->streams[slot];
        stream->active = false;
        ida_free(&dev->channel_ida, slot);
    }
}
```

**للـ cleanup عند unload**:

```c
static void hdmi_remove(struct platform_device *pdev)
{
    struct hdmi_dev *dev = platform_get_drvdata(pdev);

    /*
     * ida_destroy() frees all allocated IDs and internal bitmaps
     * safe to call even if IDA is empty
     */
    ida_destroy(&dev->channel_ida);
}
```

#### الدرس المستفاد
الفرق الجوهري بين **IDR** و**IDA** في `idr.h`: الـ IDA بس بيخزن presence (bit)، مش pointer. استخدم `ida_find_first_range()` لعمل sweep كامل للـ allocated IDs في disconnect/cleanup paths. الـ `ida_destroy()` هو الـ safety net النهائي في `remove()`.

---

### السيناريو الرابع: الـ idr_base الخاطئ في CAN bus driver على AM62x

#### العنوان
User space app بيتلقى **ID = 0** غلط من CAN socket layer على **AM62x** automotive ECU.

#### السياق
Automotive ECU بيشتغل على **TI AM62x** SoC. الـ ECU بيدير شبكة CAN لـ body control module. الـ driver بيستخدم IDR عشان يخصص socket IDs لكل CAN connection. المتطلب إن الـ IDs تبدأ من 1 (0 محجوز للـ broadcast).

#### المشكلة
الـ user space app أحياناً بيستقبل `socket_id = 0` وبيعمل broadcast message بدل ما يعمل unicast، ده بيسبب مشكلة في كل الـ ECUs على الشبكة.

#### التحليل

```c
/* الـ driver الأصلي: */
struct can_socket_mgr {
    struct idr socket_idr;  /* IDR initialized with idr_init() */
};

static int can_mgr_init(struct can_socket_mgr *mgr)
{
    /*
     * idr_init() calls idr_init_base(idr, 0)
     * هنا الـ idr_base = 0
     * يعني أول ID ممكن يتخصص هو 0
     *
     * struct idr { radix_tree_root idr_rt; unsigned int idr_base; unsigned int idr_next; }
     * idr_base = 0 → idr_alloc(idr, ptr, start=0, end=0, gfp) هيرجع 0
     */
    idr_init(&mgr->socket_idr);
    return 0;
}

static int can_socket_create(struct can_socket_mgr *mgr, struct can_sock *sock)
{
    int id;

    /* BUG: start=0 يسمح بـ ID=0 */
    id = idr_alloc(&mgr->socket_idr, sock, 0, MAX_CAN_SOCKETS, GFP_KERNEL);
    if (id < 0)
        return id;

    sock->id = id;
    return id;  /* might return 0! */
}
```

الخطأ: الـ `idr_alloc()` اتكلمت بـ `start=0`، وده معناه إن أول allocation هترجع ID=0.

#### الحل

**الأسلوب الأول** — استخدم `idr_alloc()` بـ `start=1`:

```c
static int can_socket_create(struct can_socket_mgr *mgr, struct can_sock *sock)
{
    int id;

    /*
     * Pass start=1 to idr_alloc() to skip ID 0.
     * idr_alloc() searches from max(start, idr_base) upward.
     * end=0 means no upper limit (up to INT_MAX).
     */
    id = idr_alloc(&mgr->socket_idr, sock, 1, 0, GFP_KERNEL);
    if (id < 0)
        return id;

    sock->id = id;
    return id;  /* guaranteed >= 1 */
}
```

**الأسلوب الثاني** (أوضح) — استخدم `idr_init_base()` عشان الـ intent يبقى explicit:

```c
static int can_mgr_init(struct can_socket_mgr *mgr)
{
    /*
     * idr_init_base() sets idr_base = 1
     * This documents the intent: IDs start from 1
     *
     * static inline void idr_init_base(struct idr *idr, int base) {
     *     INIT_RADIX_TREE(&idr->idr_rt, IDR_RT_MARKER);
     *     idr->idr_base = base;
     *     idr->idr_next = 0;
     * }
     */
    idr_init_base(&mgr->socket_idr, 1);
    return 0;
}

static int can_socket_create(struct can_socket_mgr *mgr, struct can_sock *sock)
{
    int id;

    /* start=0 هنا OK لأن idr_base=1 بيضمن minimum هو 1 */
    id = idr_alloc(&mgr->socket_idr, sock, 0, MAX_CAN_SOCKETS, GFP_KERNEL);
    if (id < 0)
        return id;

    sock->id = id;
    return id;
}
```

**Verification**:

```bash
# على الـ AM62x target
# dump all allocated CAN socket IDs
cat /sys/kernel/debug/can_mgr/socket_ids

# verify no ID=0 exists
grep "^0 " /sys/kernel/debug/can_mgr/socket_ids && echo "BUG: ID 0 allocated!"
```

#### الدرس المستفاد
**الـ `idr_init_base()`** في `idr.h` موجود لسبب. لما عندك domain-specific constraints على الـ ID range (زي "لازم يبدأ من 1")، وثّق الـ intent عن طريق `idr_init_base()` بدل ما تعتمد على `start` parameter في كل `idr_alloc()` call. ده بيمنع bugs لما حد تاني يضيف allocation جديدة وينسى الـ `start=1`.

---

### السيناريو الخامس: الـ idr_alloc_cyclic في I2C sensor hub على Allwinner H616

#### العنوان
**ID wrap-around** غير متوقع في I2C transaction tracker على **Allwinner H616** بيسبب data corruption في IoT gateway.

#### السياق
IoT gateway بيشتغل على **Allwinner H616** SoC. الـ gateway بيدير 16 I2C sensor (humidity, pressure, light sensors). الـ driver بيستخدم `idr_alloc_cyclic()` عشان يعمل round-robin allocation للـ transaction IDs، الهدف إن الـ IDs تكون unique لفترة كافية عشان الـ acknowledgment يوصل قبل الـ reuse.

#### المشكلة
بعض الـ sensor readings بتطلع corrupted — الـ gateway بيبلغ عن قراءات مش منطقية (humidity 150%). التحقيق بيكشف إن الـ transaction ack بيتوجه للـ transaction الغلط.

#### التحليل

```c
struct i2c_hub {
    struct idr  tx_idr;
    spinlock_t  lock;
};

static int i2c_hub_init(struct i2c_hub *hub)
{
    spin_lock_init(&hub->lock);
    idr_init(&hub->tx_idr);  /* idr_base=0, idr_next=0 */
    return 0;
}

static int i2c_send_transaction(struct i2c_hub *hub, struct i2c_tx *tx)
{
    int id;
    unsigned long flags;

    spin_lock_irqsave(&hub->lock, flags);

    /*
     * idr_alloc_cyclic() starts search from idr->idr_next (cursor).
     * It wraps around when reaching `end`, then continues from `start`.
     * After allocation: idr_next = id + 1
     *
     * BUG: end=256 ولكن acknowledgments بياخدوا حتى 50ms يوصلوا
     * في 50ms الـ driver ممكن يعمل 300+ transactions
     * يعني الـ ID اتعمل reuse وهو الـ ack القديم لسه في الطريق
     */
    id = idr_alloc_cyclic(&hub->tx_idr, tx, 0, 256, GFP_ATOMIC);
    if (id < 0) {
        spin_unlock_irqrestore(&hub->lock, flags);
        return id;
    }

    spin_unlock_irqrestore(&hub->lock, flags);
    tx->id = id;
    return i2c_submit(hub, tx);
}
```

الـ `idr_alloc_cyclic()` بيستخدم `idr->idr_next` كـ cursor (ده القيمة اللي `idr_get_cursor()` بيرجعها). لما الـ IDs من 0 لـ 255 اتملوا، الـ cyclic allocator بيرجع من 0 تاني. لو transaction قديمة (ID=5) جه الـ ack بتاعها بعد ما ID=5 اتأخذ لـ transaction جديدة، الـ ack هيطبق على الغلط.

**Debug**:

```c
/* اطبع الـ cursor الحالي */
pr_debug("IDR cursor: %u\n", idr_get_cursor(&hub->tx_idr));

/*
 * idr_get_cursor() → READ_ONCE(idr->idr_next)
 * idr_set_cursor() → WRITE_ONCE(idr->idr_next, val)
 */
```

#### الحل

**الحل الأول** — وسّع الـ range عشان تضمن uniqueness window كافية:

```c
static int i2c_send_transaction(struct i2c_hub *hub, struct i2c_tx *tx)
{
    int id;
    unsigned long flags;

    spin_lock_irqsave(&hub->lock, flags);

    /*
     * Use larger range: 0 to 65535 (64K IDs).
     * At 300 tx/sec, wrap-around happens every ~218 seconds.
     * Max ACK latency is 50ms → safe margin of 218,000ms.
     *
     * idr_alloc_cyclic() uses idr->idr_next as starting point,
     * wraps at end, returns -ENOSPC only if all slots occupied.
     */
    id = idr_alloc_cyclic(&hub->tx_idr, tx, 0, 65536, GFP_ATOMIC);

    spin_unlock_irqrestore(&hub->lock, flags);

    if (id < 0)
        return id;

    tx->id = id;
    return i2c_submit(hub, tx);
}
```

**الحل الثاني** — استخدم timeout-based cleanup بدل cyclic reuse:

```c
static void i2c_tx_timeout_handler(struct timer_list *t)
{
    struct i2c_tx *tx = from_timer(tx, t, timer);
    struct i2c_hub *hub = tx->hub;

    /* Remove expired transaction from IDR */
    idr_remove(&hub->tx_idr, tx->id);
    tx->status = -ETIMEDOUT;
    complete(&tx->done);
}

static int i2c_send_transaction(struct i2c_hub *hub, struct i2c_tx *tx)
{
    int id;
    unsigned long flags;

    spin_lock_irqsave(&hub->lock, flags);

    /* Regular (non-cyclic) allocation — IDs grow monotonically */
    id = idr_alloc(&hub->tx_idr, tx, 1, 0, GFP_ATOMIC);

    spin_unlock_irqrestore(&hub->lock, flags);

    if (id < 0)
        return id;

    tx->id = id;

    /* Set 100ms timeout — cleanup ensures no stale IDs */
    timer_setup(&tx->timer, i2c_tx_timeout_handler, 0);
    mod_timer(&tx->timer, jiffies + msecs_to_jiffies(100));

    return i2c_submit(hub, tx);
}
```

**الـ cursor manipulation** لو محتاج تعمل reset يدوي:

```c
/* بعد recovery من error: reset cyclic cursor */
idr_set_cursor(&hub->tx_idr, 0);
/* WRITE_ONCE(idr->idr_next, 0) */
```

#### الدرس المستفاد
الـ `idr_alloc_cyclic()` مش بيضمن uniqueness — بيضمن بس إن الـ ID مش **currently used**. لو عندك protocol بيسمح بـ in-flight transactions (الـ ack جاي بعد كذا مللي ثانية)، لازم تحسب الـ window اللي فيها الـ ID ممكن يتعمل reuse. الـ `idr_get_cursor()` / `idr_set_cursor()` مفيدين للـ diagnostics ولـ manual reset بعد error recovery. الـ cyclic allocator مناسب للـ use cases اللي فيها الـ ID بيتحرر قبل ما الـ range تدور دورة كاملة.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ IDR وتطوره من أول ما اتعمل لحد ما اتاستبدل بـ XArray:

| المقال | السنة | الأهمية |
|--------|-------|----------|
| [idr - integer ID management](https://lwn.net/Articles/103209/) | 2004 | أول تغطية للـ IDR subsystem |
| [A simplified IDR API](https://lwn.net/Articles/536293/) | 2013 | مقال بيشرح تبسيط الـ API |
| [idr: Rewrite ida](https://lwn.net/Articles/555189/) | 2013 | إعادة كتابة الـ IDA layer |
| [Rebasing the IDR](https://lwn.net/Articles/740394/) | 2018 | تحويل الـ IDR ليشتغل فوق XArray |
| [New IDA API](https://lwn.net/Articles/758104/) | 2018 | الـ API الجديد للـ IDA في kernel 4.19 |
| [IDA: simplifying the complex task of allocating integers](https://lwn.net/Articles/764057/) | 2018 | شرح مفصل لتبسيط الـ IDA |
| [pid: replace idr api with xarray](https://lwn.net/Articles/901383/) | 2022 | إزالة الـ IDR من الـ PID allocator |

**مقالات XArray ذات الصلة** (لأن الـ IDR بيشتغل فوقه حالياً):

| المقال | السنة |
|--------|-------|
| [Introducing the eXtensible Array (XArray)](https://lwn.net/Articles/715948/) | 2017 |
| [The XArray data structure](https://lwn.net/Articles/745073/) | 2018 |
| [XArray and the mainline](https://lwn.net/Articles/757342/) | 2018 |
| [XArray documentation](https://lwn.net/Articles/739986/) | 2018 |

---

### التوثيق الرسمي للـ Kernel

الـ documentation الرسمي موجود في مسارين أساسيين:

- **`Documentation/core-api/idr.rst`** — شرح كامل للـ IDR و IDA API مع أمثلة
  - الرابط على الويب: [ID Allocation — The Linux Kernel documentation](https://docs.kernel.org/core-api/idr.html)

- **`Documentation/core-api/xarray.rst`** — توثيق الـ XArray اللي بيشتغل الـ IDR فوقه
  - الرابط على الويب: [XArray — The Linux Kernel documentation](https://docs.kernel.org/core-api/xarray.html)

- **`include/linux/idr.h`** — الـ header file الرئيسي مع kernel-doc comments

- **`lib/idr.c`** — الـ implementation الفعلي للـ IDR و IDA

---

### Commits مهمة في تاريخ الـ IDR

**الـ commits دي بتتبع تطور الـ IDR من البداية لحد الاستقرار الحالي:**

```
# أول إضافة للـ IDR (2003) — Jim Houston
# تحويل الـ IDR لـ XArray-backed — Matthew Wilcox (2018)
# تبسيط الـ IDA API — Matthew Wilcox (2018)
```

- [XArray merge tag for 4.20](https://github.com/torvalds/linux/commit/c4d6fe7311762f2e03b3c27ad38df7c40c80cc93) — الـ commit اللي دخل فيه XArray للـ mainline

- [Convert IDA to XArray](https://patchwork.kernel.org/project/linux-fsdevel/patch/20171122210739.29916-33-willy@infradead.org/) — تحويل الـ IDA لاستخدام XArray

- [drm: Use XArray instead of IDR for minors](https://git.sceen.net/linux/linux-stable.git/commit/?id=8b0a86b45ae4ffa08f96430f0a7f956741f7a637) — مثال عملي على التهجير من IDR لـ XArray

---

### نقاشات Mailing List

- **[PATCH 0/3] pid: replace idr api with xarray** — نقاش على `lore.kernel.org`:
  [https://lore.kernel.org/lkml/20220715113349.831370-4-bfoster@redhat.com/t/](https://lore.kernel.org/lkml/20220715113349.831370-4-bfoster@redhat.com/t/)

- **[PATCH] qrtr: Convert qrtr_ports from IDR to XArray** — Matthew Wilcox بيشرح ليه لازم التهجير:
  [https://lore.kernel.org/netdev/20200605120037.17427-1-willy@infradead.org/](https://lore.kernel.org/netdev/20200605120037.17427-1-willy@infradead.org/)

- **[PATCH v4 00/73] XArray version 4** — السلسلة الكاملة اللي حولت الـ IDR:
  [https://lkml.kernel.org/lkml/20171211214300.GT5858@dastard/T/](https://lkml.kernel.org/lkml/20171211214300.GT5858@dastard/T/)

- الـ [lore.kernel.org](https://www.kernel.org/lore.html) هو المكان الرئيسي لأرشيف نقاشات الـ IDR والـ XArray

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل:** Chapter 3 — Char Drivers (بيستخدم الـ IDR لتتبع الـ minor numbers)
- **الفصل:** Chapter 11 — Data Types in the Kernel
- الكتاب مجاني على: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- الـ IDR بيظهر بشكل عملي في إدارة الـ device numbers

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل:** Chapter 6 — Kernel Data Structures
- بيغطي الـ linked lists, queues, maps (بما فيها الـ IDR) والـ binary trees
- الجزء عن الـ **idr** map تحديداً في قسم "Maps"
- ده المرجع الأسرع للمبتدئين

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)
- **الفصل:** Chapter 3 — Processes (بيشرح الـ PID allocation اللي بتستخدم IDR)
- مفيد لفهم السياق الحقيقي لاستخدام الـ IDR

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- بيغطي الـ kernel subsystems من منظور الـ embedded
- مفيد لفهم ازاي الـ IDR بيُستخدم في device drivers للـ embedded systems

---

### Kernelnewbies.org

**الـ IDR بيظهر في تغييرات kernel versions التالية:**

- [Linux_2_6_20](https://kernelnewbies.org/Linux_2_6_20) — تغييرات الـ IDR في kernel 2.6.20
- [Linux_3.15-DriversArch](https://kernelnewbies.org/Linux_3.15-DriversArch) — تغييرات الـ IDR في drivers/arch
- [Linux_6.8](https://kernelnewbies.org/Linux_6.8) — أحدث تغييرات في data structures

**صفحة رئيسية للمبتدئين:**
- [KernelNewbies.org](https://kernelnewbies.org) — نقطة بداية لفهم الـ kernel subsystems

---

### eLinux.org

- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — روابط لمصادر الـ kernel بما فيها data structures
- [Device Drivers Presentations](https://elinux.org/Device_Drivers_Presentations) — عروض تشرح استخدام الـ IDR في drivers
- [EBC Exercise 26 Device Drivers](https://elinux.org/EBC_Exercise_26_Device_Drivers) — مثال عملي على device drivers

---

### مصادر إضافية مباشرة

| المصدر | الرابط |
|--------|--------|
| الـ IDR source code | `lib/idr.c` في kernel tree |
| الـ IDA source code | `lib/idr.c` (نفس الملف) |
| الـ XArray source code | `lib/xarray.c` |
| الـ Radix Tree source code | `lib/radix-tree.c` |
| الـ kernel.org Git | [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git) |
| الـ LKML Archive | [https://lkml.org/](https://lkml.org/) |

---

### Search Terms للبحث عن معلومات أكتر

لو عايز تعمق أكتر، استخدم الكلمات دي:

```
# للـ IDR
"linux kernel IDR allocator"
"idr_alloc linux kernel"
"IDR vs XArray linux"
"integer ID management linux"

# للـ IDA
"IDA linux kernel id allocator"
"ida_alloc linux"
"linux IDA xarray bitmap"

# للـ XArray (الـ backend الحالي)
"linux XArray radix tree replacement"
"Matthew Wilcox XArray kernel"
"xa_alloc linux kernel"

# للـ use cases
"linux PID allocator IDR"
"linux file descriptor IDR"
"linux device minor number IDR"
"linux POSIX timer IDR"
```
## Phase 8: Writing simple module

### الفكرة

**`idr_alloc`** هي الدالة الأكتر استخداماً في الـ IDR subsystem — كل مرة أي subsystem في الـ kernel (زي الـ PID allocator أو الـ DRM أو الـ V4L2) بيحتاج يحجز ID جديد، بيستدعي `idr_alloc`. سنستخدم **kprobe** عشان نعمل hook على الـ function دي ونطبع الـ start/end range المطلوبين وعنوان الـ pointer اللي هيتربط بالـ ID الجديد — ده بيكشف مين في الـ kernel بيحجز IDs وبأي parameters.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * idr_alloc_probe.c
 * Hooks idr_alloc() via kprobe to trace every ID allocation in the kernel.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit          */
#include <linux/kernel.h>      /* pr_info                                    */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe             */
#include <linux/idr.h>         /* struct idr — to read idr_base from the arg */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("Trace idr_alloc() calls via kprobe to inspect ID allocations");

/*
 * pre_handler — called right before idr_alloc() executes.
 *
 * Prototype of idr_alloc:
 *   int idr_alloc(struct idr *idr, void *ptr, int start, int end, gfp_t gfp);
 *
 * On x86-64, arguments follow the System V AMD64 ABI:
 *   rdi = idr,  rsi = ptr,  rdx = start,  rcx = end,  r8 = gfp
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct idr *idr  = (struct idr *)regs->di;   /* 1st arg: the IDR        */
    void       *ptr  = (void *)regs->si;          /* 2nd arg: pointer stored */
    int         start = (int)regs->dx;            /* 3rd arg: start of range */
    int         end   = (int)regs->cx;            /* 4th arg: end of range   */

    pr_info("idr_alloc: idr=%px base=%u ptr=%px start=%d end=%d\n",
            idr, idr ? idr->idr_base : 0,
            ptr, start, end);

    return 0; /* 0 = let idr_alloc() continue normally */
}

/* The kprobe descriptor — we target idr_alloc by symbol name */
static struct kprobe kp = {
    .symbol_name = "idr_alloc",   /* kernel resolves the address at load     */
    .pre_handler = handler_pre,   /* our callback fires before the function  */
};

static int __init idr_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("idr_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("idr_probe: kprobe planted at %px (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

static void __exit idr_probe_exit(void)
{
    /*
     * Must unregister before the module is unloaded —
     * leaving a kprobe pointing to freed module memory causes a panic.
     */
    unregister_kprobe(&kp);
    pr_info("idr_probe: kprobe removed\n");
}

module_init(idr_probe_init);
module_exit(idr_probe_exit);
```

---

### Makefile

```makefile
obj-m += idr_alloc_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

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
| `<linux/module.h>` | ضروري لـ `module_init` / `module_exit` والـ macros |
| `<linux/kernel.h>` | عشان `pr_info` و `pr_err` |
| `<linux/kprobes.h>` | فيه `struct kprobe` و `register_kprobe` |
| `<linux/idr.h>` | عشان نقدر نقرأ `idr->idr_base` جوه الـ handler |

الـ includes دي بس اللي محتاجينها — محتشناش حاجة زيادة.

#### الـ `pre_handler`

الـ kprobe بيحقن كود قبل أول instruction في `idr_alloc`. الـ `struct pt_regs *regs` بيحتوي على الـ CPU registers وقت الاستدعاء، وعلى x86-64 الـ ABI بيحط الـ arguments في `rdi, rsi, rdx, rcx, r8` بالترتيب — فنقدر نقرأ الـ parameters من غير ما نعدّل الـ function نفسها.

الـ `pr_info` بيطبع:
- **`idr` pointer** — نعرف منه أي instance اتعمل فيها الـ allocation
- **`idr_base`** — الـ base اللي بيبدأ منه الـ IDR ده (مفيد عشان تفرق بين PID namespace وغيره)
- **`ptr`** — عنوان الـ object اللي هيتربط بالـ ID الجديد
- **`start/end`** — الـ range المطلوب، بيكشف الـ constraints اللي الـ caller بيحطها

#### الـ `struct kprobe`

الحقل `symbol_name` بيخلي الـ kernel يحل العنوان تلقائياً من الـ kallsyms — مش محتاجين نعرف العنوان الفعلي. الـ `pre_handler` هو الـ callback اللي بيشتغل قبل الـ function، وده اللي يناسبنا عشان نشوف الـ arguments قبل ما تتغير.

#### الـ `module_init` / `module_exit`

- **`register_kprobe`** في الـ init: بيزرع الـ breakpoint في الـ kernel code. لو فشل (مثلاً الـ function `inline` أو محمية) بيرجع error فوراً بدل ما نفضّل في حالة undefined.
- **`unregister_kprobe`** في الـ exit: **إجباري** — لو شيلنا الـ module من غير ما نشيل الـ kprobe، الـ kernel هيحاول ينفذ الـ handler في module memory اتحررت فعلاً، وده بيعمل kernel panic فوري.

---

### تشغيل وملاحظة النتائج

```bash
# Build
make

# Load
sudo insmod idr_alloc_probe.ko

# Watch the output — any IDR allocation in the system will appear
sudo dmesg -w | grep idr_alloc

# Generate some activity (e.g., open a new process, load a driver)
sleep 1 &

# Unload
sudo rmmod idr_alloc_probe

# Sample output:
# idr_probe: kprobe planted at ffffffffc0abcd12 (idr_alloc)
# idr_alloc: idr=ffff888003a12040 base=1 ptr=ffff888101234500 start=1 end=0 pid_ns
# idr_alloc: idr=ffff888003b00080 base=0 ptr=ffff888100ab3000 start=0 end=0
```

**الـ `end=0`** معناه مش فيه upper limit — الـ IDR يحجز أي رقم متاح. الـ `base=1` دليل إن ده على الأغلب PID namespace لأنه بيبدأ IDR من 1 مش 0.
