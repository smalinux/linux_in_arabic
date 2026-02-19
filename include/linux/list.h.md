## Phase 1: `list.h` — الهوية والسياق

# `list.h`

> **PATH**: `include/linux/list.h`
> **Subsystem**: Core Kernel / Data Structures
> **الوظيفة الأساسية**: تعريف القوائم المترابطة (Linked Lists) الدائرية المزدوجة اللي يستخدمها كل الـ kernel تقريباً لتنظيم البيانات.

---

### ما هو هذا الملف؟

تخيل إنك عندك سلسلة من الحلقات — كل حلقة مربوطة بالي قبلها وبالي بعدها، والأخيرة مربوطة بالأولى من جديد. هذا بالضبط ما يقدمه `list.h` للـ kernel.

هذا الملف هو **العمود الفقري** لتنظيم البيانات في Linux kernel. ما من subsystem تقريباً إلا ويستخدم `struct list_head` لتنظيم قوائمه — سواء كانت قائمة processes، أو قائمة devices، أو قائمة timers، أو أي شيء آخر. بدل ما كل subsystem يكتب linked list خاصة فيه، الكل يشترك في هذا الكود الموحد والمُجرَّب.

الملف يقدم نوعين من القوائم:
- **`list_head`**: قائمة دائرية مزدوجة (كل عقدة تعرف من قبلها ومن بعدها).
- **`hlist_head` / `hlist_node`**: قائمة أحادية مُحسَّنة للـ hash tables حيث الـ head يحتاج حجم أصغر.

---

### الـ Includes الرئيسية

| Include | ما الذي يجلبه |
|---------|----------------|
| `<linux/container_of.h>` | الماكرو السحري `container_of` — يحول pointer لـ `list_head` إلى pointer للـ struct الأصلي اللي يحتويه |
| `<linux/types.h>` | تعريفات `struct list_head` و `struct hlist_head` و `struct hlist_node` — الهياكل الأساسية للقوائم |
| `<linux/stddef.h>` | `NULL` وماكرو `offsetof` الذي يعتمد عليه `container_of` |
| `<linux/poison.h>` | قيم `LIST_POISON1` و `LIST_POISON2` — أرقام مقصودة تُكتب في pointers العقد المحذوفة لكشف الاستخدام الخاطئ بعد الحذف |
| `<linux/const.h>` | ماكرو `_BITUL` وغيره من الثوابت الأساسية |
| `<asm/barrier.h>` | memory barriers مثل `WRITE_ONCE` — تضمن أن الـ CPU لا يُعيد ترتيب عمليات الكتابة على القائمة بشكل خاطئ في بيئات multi-core |

---

### نقطة جوهرية: الفلسفة الغريبة (Intrusive Lists)

معظم linked lists في لغات أخرى تخزن البيانات **داخل** عقدة القائمة. Linux يعكس المعادلة:

```c
/* struct list_head مش بيحتوي data — هو يُدمَج داخل الـ struct الحقيقي */

struct task_struct {          /* الـ process */
    pid_t pid;
    struct list_head tasks;   /* ← القائمة مدموجة هنا */
    /* ... */
};
```

يعني `list_head` هو مجرد "خطاف" يُوصَل بأي struct. وعشان ترجع للـ struct الأصلي من الخطاف، تستخدم `container_of`:

```c
/* من list_head* وصل لـ task_struct* */
struct task_struct *t = container_of(ptr, struct task_struct, tasks);
```

هذا ما يجعل `list.h` **generic بالكامل** بدون templates أو void pointers.

---

### الـ Poison Values — حارس من الأخطاء

```c
/* من include/linux/poison.h */
#define LIST_POISON1  ((void *) 0x100 + POISON_POINTER_DELTA)
#define LIST_POISON2  ((void *) 0x122 + POISON_POINTER_DELTA)
```

لما تُحذف عقدة من القائمة، الـ kernel يكتب هذه القيم الغريبة في الـ `next` و `prev` pointers بدل ما يتركها تشير لعنوان قديم. لو أي كود حاول يوصل لعقدة محذوفة بالغلط، الـ CPU سيـcrash فوراً على هذا العنوان غير الصالح — بدل ما يُفسد بيانات عشوائية بصمت.

---

### الملفات ذات الصلة

| الملف | العلاقة |
|-------|---------|
| `include/linux/types.h` | يعرّف `struct list_head` و `struct hlist_head` و `struct hlist_node` |
| `include/linux/container_of.h` | يعرّف الماكرو الأساسي لاسترجاع الـ parent struct |
| `include/linux/poison.h` | يعرّف قيم الـ poison المستخدمة عند حذف العقد |
| `include/linux/rculist.h` | امتداد لـ `list.h` يضيف دعم **RCU** (Read-Copy-Update) للقوائم في بيئات concurrent |
| `lib/list_sort.c` | تنفيذ خوارزمية **merge sort** للقوائم من نوع `list_head` |

---

### الرخصة والحقوق

- **License**: `GPL-2.0` — مفتوح المصدر، ضمن شروط GNU General Public License الإصدار 2.
- لا يوجد copyright مباشر في هذا الملف — هو جزء من Linux kernel الأصلي ومساهمات متعددة على مر السنين.

---

## Phase 2: شرح الـ Linked List Framework

---

### المشكلة — ليش موجود أصلاً؟

تخيل إنك بتكتب driver لكارت شبكة. عندك قائمة من الـ packets اللي محتاج تشيلها في الذاكرة وتعالجها وتحذفها. تحتاج:

- **تضيف** packet جديد بسرعة
- **تحذف** packet من نص القائمة بدون ما تحرك باقي العناصر
- **تمشي** على كل القائمة عنصر عنصر

الـ array العادي ما يكفي — لو حذفت من النص تحتاج تحرك كل العناصر. الـ linked list هو الحل.

**المشكلة الحقيقية:** كل مبرمج kernel كان يكتب الـ linked list من الصفر لكل struct جديد. نفس الكود يتكرر آلاف المرات في الكرنل. هدر وقت، وأخطاء، وكود ما يُقرأ.

---

### الحل — نهج الـ Kernel

الـ Linux kernel حل المشكلة بطريقة ذكية جداً: **بدل ما تكتب linked list لكل نوع بيانات، تدمج الـ list_head داخل أي struct تبغاه**.

```c
/* بدل كذا — مملة وتتكرر */
struct packet_node {
    struct packet *data;
    struct packet_node *next;
    struct packet_node *prev;
};

/* الـ kernel يسوي كذا — ذكاء */
struct packet {
    char data[1500];
    int size;
    struct list_head list;   /* هذا هو السحر */
};
```

الـ `list_head` ما تعرف إيش يحتويها — هي مجرد `next` و `prev`. والـ kernel يستخدم `container_of` يرجع من الـ `list_head` للـ struct الأصلي.

---

### التشبيه — حلقات المفاتيح

تخيل عندك **حلقة مفاتيح دائرية**.

- كل **مفتاح** = struct عندك (packet، device، task)
- كل مفتاح فيه **حلقة صغيرة مدمجة** = `struct list_head`
- الحلقات الصغيرة مربوطة ببعض = الـ linked list
- تقدر تسلك أي مفتاح من الحلقة وترجع للمفتاح الكامل = `container_of`

```
┌─────────────────────────────────────────────────┐
│  حلقة المفاتيح (الـ list الدائرية)              │
│                                                  │
│  ┌────────┐     ┌────────┐     ┌────────┐        │
│  │ packet │────▶│ packet │────▶│ packet │──┐     │
│  │  #1    │◀────│  #2    │◀────│  #3    │  │     │
│  └────────┘     └────────┘     └────────┘  │     │
│       ▲                                    │     │
│       └────────────────────────────────────┘     │
│                   دائرية!                        │
└─────────────────────────────────────────────────┘
```

---

### الصورة الكبيرة — أين تقع في الـ Kernel؟

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Linux Kernel                                  │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │  Networking  │  │  Filesystem  │  │   Drivers     │               │
│  │  (sk_buff)   │  │  (dentry)    │  │  (devices)    │  ◀ مستخدمين  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                  │                       │
│         └─────────────────┴──────────────────┘                      │
│                           │                                          │
│                           ▼                                          │
│         ┌─────────────────────────────────┐                         │
│         │      include/linux/list.h        │                         │
│         │                                  │                         │
│         │  struct list_head  { next, prev }│  ◀ قلب الموضوع         │
│         │  struct hlist_head { first }     │                         │
│         │  struct hlist_node { next, pprev}│                         │
│         │                                  │                         │
│         │  list_add / list_del             │                         │
│         │  list_for_each_entry             │                         │
│         │  container_of                    │                         │
│         └─────────────────────────────────┘                         │
│                           │                                          │
│                           ▼                                          │
│         ┌─────────────────────────────────┐                         │
│         │      include/linux/types.h       │  ◀ تعريف الـ structs   │
│         └─────────────────────────────────┘                         │
│                           │                                          │
│                           ▼                                          │
│         ┌─────────────────────────────────┐                         │
│         │      include/linux/poison.h      │  ◀ حماية من الأخطاء   │
│         └─────────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

### النموذجان — list_head و hlist

الـ kernel يوفر **نوعين** من الـ linked list:

#### النوع الأول: `list_head` — القائمة الدائرية المزدوجة

```
HEAD ◀──────────────────────────────────────────▶
 │                                               │
 ▼                                               │
[sentinel] ──▶ [node1] ──▶ [node2] ──▶ [node3] ──┘
           ◀──         ◀──         ◀──
```

- **دائرية**: آخر عنصر يشير على الـ head والعكس
- **مزدوجة**: كل node عندها `next` و `prev`
- **O(1)** للإضافة من الطرفين (head أو tail)
- مناسبة لـ: قوائم الـ tasks، قوائم الـ devices، أي قائمة تحتاج traversal كامل

#### النوع الثاني: `hlist_head / hlist_node` — لجداول الـ Hash

```
bucket[0]:  hlist_head ──▶ node ──▶ node ──▶ NULL
bucket[1]:  hlist_head ──▶ NULL
bucket[2]:  hlist_head ──▶ node ──▶ NULL
bucket[N]:  hlist_head ──▶ node ──▶ node ──▶ node ──▶ NULL
```

- الـ head عنده **مؤشر واحد بس** (`first`) بدل اثنين — يوفر ذاكرة
- لو عندك 65536 bucket في جدول hash، توفر ذاكرة مهمة
- الـ `pprev` (مؤشر لمؤشر) يخلي الحذف O(1) حتى بدون head reference

---

### سر الـ `container_of` — كيف ترجع للـ Struct الأصلي؟

هذا هو الـ magic الحقيقي. تخيل عندك packet struct:

```c
struct packet {
    char  data[1500];    /* offset: 0   */
    int   size;          /* offset: 1500 */
    struct list_head list; /* offset: 1504 */
};
```

لما تمشي على القائمة، عندك مؤشر للـ `list_head` (offset 1504). كيف ترجع للـ `struct packet` كامل؟

```
الذاكرة:
┌─────────────────────────────────────┐
│ offset 0:    data[1500]             │ ◀ بغيت تعرف هذا العنوان
│ offset 1500: size (int)             │
│ offset 1504: list.next ────────────────▶ للـ packet التالي
│ offset 1508: list.prev ─────────────── للـ packet السابق
└─────────────────────────────────────┘
        ▲
        │ عنوان الـ struct = عنوان list_head - 1504
        │
container_of(ptr, struct packet, list)
= (struct packet *)((char *)ptr - offsetof(struct packet, list))
```

```c
/* استخدام حقيقي */
struct list_head *pos;
list_for_each(pos, &packet_list) {
    /* container_of يحسب العنوان رياضياً */
    struct packet *pkt = list_entry(pos, struct packet, list);
    process_packet(pkt);
}
```

---

### الـ Sentinel Node — الخدعة الذكية

القائمة الفارغة في الـ kernel مش NULL — هي **تشير على نفسها**:

```c
/* قائمة فارغة */
HEAD.next = &HEAD;
HEAD.prev = &HEAD;

/* بعد إضافة node1 */
HEAD.next = &node1;
HEAD.prev = &node1;
node1.next = &HEAD;
node1.prev = &HEAD;
```

**ليش هذا ذكي؟** لأن كل العمليات تشتغل بدون شرط `if (empty)`. ما في حالة خاصة. الكود يصير أبسط وأسرع.

```c
/* list_empty — بسيطة جداً */
static inline int list_empty(const struct list_head *head) {
    return head->next == head;  /* مجرد مقارنة واحدة */
}
```

---

### الـ Poison Pointers — الحماية من الأخطاء

لما تحذف node من القائمة، الـ kernel يحط قيم غريبة في الـ pointers بدل NULL:

```c
#define LIST_POISON1  ((void *) 0x100 + POISON_POINTER_DELTA)
#define LIST_POISON2  ((void *) 0x122 + POISON_POINTER_DELTA)
```

**ليش؟** لو أي كود غلطان حاول يقرأ من node محذوفة، الـ kernel يـcrash فوراً بـ page fault على عنوان غير قانوني. بدل ما البرنامج يكمل ويخرب بيانات ثانية.

هذا أفضل من NULL لأن بعض الأرشتكشرز ممكن تقرأ من عنوان 0 بدون crash.

---

### الـ list_for_each_safe — لما تحذف وأنت تمشي

```c
/* خطر! لو حذفت pos داخل الـ loop */
list_for_each(pos, head) {
    list_del(pos);   /* BOOM — pos->next صار poison */
}

/* آمن — يحفظ next قبل ما نحذف */
list_for_each_safe(pos, n, head) {
    list_del(pos);   /* OK — n يحفظ العنصر التالي */
}
```

---

### الـ Bulk Operations — قوة إضافية

الـ kernel ما يكتفي بإضافة وحذف عنصر واحد. عنده عمليات على قوائم كاملة:

| العملية | الوصف | الاستخدام |
|---------|-------|-----------|
| `list_splice` | دمج قائمتين في قائمة واحدة | نقل كل الـ packets |
| `list_cut_position` | قص القائمة عند نقطة معينة | أخذ batch من الطلبات |
| `list_rotate_left` | تدوير القائمة يساراً | round-robin scheduling |
| `list_bulk_move_tail` | نقل عدة عناصر للنهاية | إعادة ترتيب الأولويات |

---

### الملفات المكونة لهذا الـ Framework

```
include/linux/list.h          ← قلب الموضوع — كل الـ API
include/linux/types.h         ← تعريف struct list_head, hlist_head, hlist_node
include/linux/poison.h        ← ثوابت LIST_POISON1, LIST_POISON2
include/linux/container_of.h  ← ماكرو container_of (الأساس)
include/linux/stddef.h        ← ماكرو offsetof (يستخدمه container_of)
asm/barrier.h                 ← memory barriers للـ SMP safety
```

لا يوجد ملف `.c` — كل الكود **header-only**، كل الدوال `static inline`. يعني الـ compiler يضع الكود مباشرة في المكان اللي اتنادى منه بدون overhead لـ function call.

---

### الصورة الكاملة — مَن يستخدم list.h؟

```
┌────────────────────────────────────────────────────────────────────┐
│                        Kernel Subsystems                           │
│                                                                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │  sched   │  │   net    │  │    vfs   │  │  driver  │          │
│  │task_struct│  │ sk_buff  │  │  dentry  │  │  device  │          │
│  │.tasks    │  │.list     │  │.d_child  │  │.devres   │          │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │
│       │              │              │              │               │
│       └──────────────┴──────────────┴──────────────┘              │
│                              │                                     │
│                              ▼                                     │
│              ┌──────────────────────────┐                         │
│              │    include/linux/list.h   │                         │
│              │                          │                         │
│              │  list_head               │                         │
│              │  hlist_head/node         │                         │
│              │  list_add/del/for_each   │                         │
│              └──────────────────────────┘                         │
│                              │                                     │
│              ┌───────────────┴───────────────┐                    │
│              ▼                               ▼                    │
│   ┌──────────────────┐           ┌──────────────────┐            │
│   │  include/linux/  │           │  include/linux/  │            │
│   │    types.h       │           │  container_of.h  │            │
│   └──────────────────┘           └──────────────────┘            │
└────────────────────────────────────────────────────────────────────┘
```

الـ `list.h` هو واحد من أكثر الملفات استخداماً في الـ kernel بالكامل — تقريباً كل subsystem يعتمد عليه بشكل مباشر أو غير مباشر.

---

## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

## 1. الـ Structs الأساسية

### `struct list_head`

**الهدف:** هذا هو القلب كله. كل node في أي linked list في الكيرنل بيحتوي على نسخة من هذا الـ struct.

الفكرة الأساسية: بدل ما تعمل list من نوع معين (list من processes، list من devices...)، الكيرنل يحط `list_head` **جوّا** في أي struct تحب، وبالتالي يستخدم نفس الكود لكل شيء.

```c
// من include/linux/types.h
struct list_head {
    struct list_head *next;  // المؤشر للعنصر التالي
    struct list_head *prev;  // المؤشر للعنصر السابق
};
```

**التشبيه:** تخيل سلسلة فيها حلقات. كل حلقة `list_head` بتربط بالحلقة اللي قبلها وبعدها. العنصر الحقيقي (مثلاً معلومات process) هو الشيء المربوط بالحلقة، مش الحلقة نفسها.

**الحقول:**

| الحقل | النوع | الوظيفة |
|-------|-------|---------|
| `next` | `struct list_head *` | يشير للعنصر التالي في القائمة |
| `prev` | `struct list_head *` | يشير للعنصر السابق في القائمة |

**الخاصية المهمة — الـ Circular List:**
القائمة دايرية (circular). آخر عنصر `next` بيشير للـ head، وأول عنصر `prev` بيشير للـ head كمان. لما تكون القائمة فاضية، الـ head بيشير على نفسه في الاتجاهين.

---

### `struct hlist_head`

```c
// من include/linux/types.h
struct hlist_head {
    struct hlist_node *first;  // أول عنصر في القائمة (أو NULL)
};
```

**الهدف:** رأس قائمة مُحسَّنة للاستخدام في **hash tables**. بتاخد نص المساحة من `list_head` لأنها محتاجة بس مؤشر واحد (مش اتنين).

**التشبيه:** تخيل عندك 1024 درج في خزانة. كل درج هو `hlist_head` — بس محتاج يعرف أين أول ورقة فيه. مش محتاج يعرف آخر ورقة عشان يوفر مساحة.

---

### `struct hlist_node`

```c
// من include/linux/types.h
struct hlist_node {
    struct hlist_node  *next;    // المؤشر للعنصر التالي
    struct hlist_node **pprev;   // مؤشر لمؤشر العنصر السابق
};
```

**الهدف:** العنصر اللي بيتحط في hash table. الفرق الغريب هنا: `pprev` هو **مؤشر لمؤشر**، مش مؤشر عادي.

**ليه `**pprev` ومش `*prev` عادي؟**

الإجابة ذكية جداً:
- الـ `hlist_head` عنده `first` من نوع `hlist_node *`
- الـ `hlist_node` عنده `next` من نوع `hlist_node *`
- الاتنين نفس النوع!

بالتالي `pprev` بيشير لـ **أي مؤشر** يشير لنا — سواء كان `first` في الـ head أو `next` في العنصر السابق. هذا بيخلي الحذف O(1) بدون ما نعرف الـ head.

---

## 2. رسم علاقات الـ Structs

### النموذج الكامل: كيف `list_head` يرتبط بالـ struct الأصلي

```
      ┌─────────────────────────────────────────┐
      │           struct task_struct             │
      │  (مثال على أي struct في الكيرنل)         │
      │                                         │
      │   pid: 1234                             │
      │   comm: "bash"                          │
      │   ┌─────────────────┐                   │
      │   │  struct list_head  tasks            │ ◄── هنا يتحط list_head
      │   │  next ──────────┼───────────────────┼──► (التالي)
      │   │  prev ◄─────────┼───────────────────┼─── (السابق)
      │   └─────────────────┘                   │
      └─────────────────────────────────────────┘
                    ▲
                    │
            container_of(ptr, struct task_struct, tasks)
                    │
                    │  "من عنوان list_head، ارجع للـ struct الأصلي"
```

### القائمة الدائرية المزدوجة (Circular Doubly Linked List)

```
                         HEAD
                    ┌──────────┐
                    │ list_head│
              ┌────►│  next ───┼──────────────────────┐
              │     │  prev ◄──┼──────────────────┐   │
              │     └──────────┘                  │   │
              │                                   │   ▼
              │     ┌──────────┐             ┌──────────┐
              │     │ list_head│             │ list_head│
              │     │  next ───┼────────────►│  next ───┼──┐
              │     │  prev ◄──┼─────────────┼── prev   │  │
              │     └──────────┘             └──────────┘  │
              │      (العنصر الأول)           (العنصر الثاني)
              │                                             │
              └─────────────────────────────────────────────┘
                          (آخر عنصر next يرجع للـ HEAD)
```

**قائمة فاضية:**
```
     HEAD
┌──────────┐
│ list_head│
│  next ───┼──┐
│  prev ◄──┼──┘  ← يشير على نفسه
└──────────┘
```

---

### علاقة `hlist_head` و `hlist_node`

```
Hash Table:
┌─────────────────────────────────────────────────────┐
│  bucket[0]   │  bucket[1]   │  ...  │  bucket[N]   │
│ ┌──────────┐ │ ┌──────────┐ │       │ ┌──────────┐ │
│ │hlist_head│ │ │hlist_head│ │       │ │hlist_head│ │
│ │ first    │ │ │ first    │ │       │ │ first    │ │
│ └────┬─────┘ │ └──────────┘ │       │ └──────────┘ │
└──────┼────────────────────────────────────────────--┘
       │
       ▼
  ┌──────────┐
  │hlist_node│
  │  next ───┼──────────────► ┌──────────┐
  │  pprev ◄─┼── يشير لـ      │hlist_node│
  └──────────┘  (bucket.first)│  next ───┼──► NULL
                              │  pprev ◄─┼── يشير لـ (prev.next)
                              └──────────┘
```

**تفصيل `pprev` بالصور:**

```
قبل الإضافة:
  head.first ──► nodeA ──► nodeB ──► NULL

بعد إضافة nodeX في البداية:
  head.first ──► nodeX ──► nodeA ──► nodeB ──► NULL

nodeX.pprev = &(head.first)    ← يشير للمتغير اللي بيشير لنا
nodeA.pprev = &(nodeX.next)    ← يشير للمتغير اللي بيشير لنا
nodeB.pprev = &(nodeA.next)    ← يشير للمتغير اللي بيشير لنا

لحذف nodeX:
  *(nodeX.pprev) = nodeX.next   ← head.first = nodeA  ✓
  nodeA.pprev = &(head.first)   ← تحديث السابق
```

---

## 3. رسم دورة الحياة (Lifecycle)

### دورة حياة `list_head`

```
┌─────────────────────────────────────────────────────────┐
│                    LIFECYCLE                             │
└─────────────────────────────────────────────────────────┘

  1. BIRTH (الإنشاء)
  ─────────────────
  طريقة 1 — Static (وقت الـ compile):

    LIST_HEAD(my_list);
    ↓
    struct list_head my_list = { &my_list, &my_list };
    (next وprev يشيران على نفسهم)

  طريقة 2 — Dynamic (وقت الـ runtime):

    struct list_head *p = kmalloc(...);
    INIT_LIST_HEAD(p);
    ↓
    WRITE_ONCE(p->next, p);
    WRITE_ONCE(p->prev, p);


  2. INSERTION (الإدراج)
  ──────────────────────
                    ┌── list_add() ──► بعد الـ HEAD مباشرة (Stack/LIFO)
  list ──────────── │
                    └── list_add_tail() ──► قبل الـ HEAD (Queue/FIFO)

  مثال list_add(new, head):

  قبل:   HEAD ◄──► A ◄──► B
  بعد:   HEAD ◄──► new ◄──► A ◄──► B


  3. USAGE (الاستخدام)
  ─────────────────────
  list_for_each_entry(pos, head, member) {
      // pos يشير للـ struct الكبير
      // نستخدم container_of داخلياً
  }


  4. MANIPULATION (التعديل)
  ─────────────────────────
  list_move()      ──► نقل عنصر لأول القائمة
  list_move_tail() ──► نقل عنصر لآخر القائمة
  list_replace()   ──► تبديل عنصر بآخر
  list_swap()      ──► تبديل موقعين
  list_splice()    ──► دمج قائمتين


  5. REMOVAL (الحذف)
  ───────────────────
  طريقة 1 — list_del():
    يحذف + يضع POISON (0x100, 0x122)
    العنصر "ميت" — أي استخدام بعدها = kernel panic

  طريقة 2 — list_del_init():
    يحذف + يعيد تهيئة (next=self, prev=self)
    العنصر "فاضي" — يمكن إعادة استخدامه


  6. DEATH (الإتلاف)
  ───────────────────
  kfree(container) — تحرير الذاكرة الكاملة
```

---

### دورة حياة `hlist_node`

```
  BIRTH:
  INIT_HLIST_NODE(node)  →  node->next=NULL, node->pprev=NULL

  INSERTION:
  hlist_add_head(node, head)  →  يضاف في أول القائمة

  DELETION:
  hlist_del(node)       →  يحذف + POISON
  hlist_del_init(node)  →  يحذف + يعيد التهيئة

  CHECK STATUS:
  hlist_unhashed(node)  →  هل pprev == NULL؟ (مش في قائمة)
```

---

## 4. رسم تدفق الاستدعاء (Call Flow)

### تدفق `list_add(new, head)`

```
list_add(new, head)
        │
        ▼
__list_add(new, head, head->next)
        │
        ├──► __list_add_valid(new, prev, next)   [CONFIG_LIST_HARDENED]
        │         │
        │         ├── هل next->prev == prev ؟
        │         ├── هل prev->next == next ؟
        │         ├── هل new != prev ؟
        │         └── هل new != next ؟
        │              ↓
        │         إذا فشل: __list_add_valid_or_report()
        │
        ├──► next->prev = new
        ├──► new->next  = next
        ├──► new->prev  = prev
        └──► WRITE_ONCE(prev->next, new)   ← آخر خطوة (atomic write)
```

**ليه `WRITE_ONCE` في الأخير فقط؟** لأن لحظة كتابة `prev->next = new` هي اللحظة اللي العنصر الجديد "يظهر" للـ threads الثانية. كل الروابط الأخرى لازم تكون جاهزة قبلها.

---

### تدفق `list_del(entry)`

```
list_del(entry)
        │
        ├──► __list_del_entry(entry)
        │         │
        │         ├──► __list_del_entry_valid(entry)  [CONFIG_LIST_HARDENED]
        │         │         ├── هل prev->next == entry ؟
        │         │         └── هل next->prev == entry ؟
        │         │
        │         └──► __list_del(entry->prev, entry->next)
        │                   ├── next->prev = prev
        │                   └── WRITE_ONCE(prev->next, next)
        │
        ├──► entry->next = LIST_POISON1  (0x100)
        └──► entry->prev = LIST_POISON2  (0x122)
```

---

### تدفق `list_for_each_entry(pos, head, member)`

```
list_for_each_entry(pos, head, member)
        │
        │  يتحول لـ:
        ▼
for (pos = list_first_entry(head, typeof(*pos), member);
     !list_entry_is_head(pos, head, member);
     pos = list_next_entry(pos, member))
{
    // body
}
        │
        │  list_first_entry:
        ▼
container_of(head->next, typeof(*pos), member)
        │
        │  container_of:
        ▼
(typeof(*pos) *)( (char*)(head->next) - offsetof(typeof(*pos), member) )
        │
        │  يطرح الـ offset عشان يرجع لبداية الـ struct الكبير
        ▼
  struct task_struct * ← مثلاً
```

**التشبيه:** تخيل إن عندك سلسلة وكل حلقة مربوطة في حقيبة. `container_of` هو اللي يقول "الحلقة دي مربوطة في جانب الحقيبة — فامشي للخلف X سنتيمتر تلقى بداية الحقيبة."

---

### تدفق `list_splice(list, head)`

```
list_splice(list, head):
        │
        ├── هل list فاضية؟ → لا تعمل شيء
        │
        └──► __list_splice(list, head, head->next)
                   │
                   ├── first = list->next    (أول عنصر في list)
                   ├── last  = list->prev    (آخر عنصر في list)
                   ├── at    = head->next    (أول عنصر في head)
                   │
                   ├── first->prev = head
                   ├── WRITE_ONCE(head->next, first)
                   ├── last->next  = at
                   └── at->prev    = last

قبل:  HEAD ◄──► A ◄──► B ◄──► HEAD
      LIST ◄──► X ◄──► Y ◄──► LIST

بعد:  HEAD ◄──► X ◄──► Y ◄──► A ◄──► B ◄──► HEAD
      LIST ◄──► LIST  (فاضية — لو استخدمنا splice_init)
```

---

## 5. استراتيجية الـ Locking

### المبدأ الأساسي

`list.h` نفسه **لا يحتوي على أي lock**. هو بيوفر الأدوات، وأنت مسؤول عن الـ locking. الكيرنل بيستخدم أنماط معيارية مع القوائم:

---

### Pattern 1: `spinlock_t`

للقوائم اللي تتعدل من **interrupt context** أو من **multi-core**:

```c
struct my_list {
    struct list_head  head;
    spinlock_t        lock;    // يحمي head وكل العناصر
};

// الكتابة:
spin_lock(&list->lock);
list_add(&new_entry->node, &list->head);
spin_unlock(&list->lock);

// القراءة مع التعديل الآمن (safe iteration):
spin_lock(&list->lock);
list_for_each_entry_safe(pos, tmp, &list->head, node) {
    if (should_delete(pos))
        list_del(&pos->node);
}
spin_unlock(&list->lock);
```

---

### Pattern 2: `mutex`

للقوائم اللي تتعدل فقط من **process context** (يمكن تنام):

```c
struct my_list {
    struct list_head  head;
    struct mutex      lock;
};

mutex_lock(&list->lock);
list_add_tail(&entry->node, &list->head);
mutex_unlock(&list->lock);
```

---

### Pattern 3: RCU (Read-Copy-Update)

للقوائم اللي **القراءة كثيرة جداً** والكتابة نادرة:

```c
// القراءة (بدون lock):
rcu_read_lock();
list_for_each_entry_rcu(pos, head, node) {
    // ...
}
rcu_read_unlock();

// الكتابة:
spin_lock(&my_lock);
list_add_rcu(&entry->node, head);  // يستخدم rcu_assign_pointer داخلياً
spin_unlock(&my_lock);
```

---

### جدول الـ Locking

| الحالة | الـ Lock المناسب | السبب |
|--------|-----------------|-------|
| قراءة فقط، single-threaded | لا شيء | آمن |
| قراءة/كتابة، process context | `mutex` | يمكن ينام |
| قراءة/كتابة، interrupt context | `spinlock_t` | لا ينام |
| قراءة كثيرة، كتابة نادرة | RCU | أداء عالٍ |
| hash table buckets | `spinlock_t` per bucket | تقليل التنازع |

---

## 6. جدول الـ Config Options

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_LIST_HARDENED` | يفعّل inline checks لكشف الـ list corruption |
| `CONFIG_DEBUG_LIST` | يفعّل checks كاملة مع تقارير تفصيلية |

---

## 7. جدول الـ Poison Values

| الثابت | القيمة | الاستخدام |
|--------|--------|-----------|
| `LIST_POISON1` | `0x100 + delta` | يُكتب في `next` بعد `list_del()` |
| `LIST_POISON2` | `0x122 + delta` | يُكتب في `prev` بعد `list_del()` |

---

## Phase 4: دليل الـ Debugging الشامل

---

### مقدمة سريعة

الـ linked list في الكيرنل زي سلسلة مفاتيح — لو حلقة واحدة انكسرت أو اتسرقت، السلسلة كلها بتتخرب. المشكلة إن الـ bugs اللي بتيجي من هنا **صعبة جداً** لأنها بتتفجر في مكان تاني غير المكان اللي الـ corruption حصل فيه. الـ debugging هنا فن.

---

## 1. الـ Kernel Config Options للـ Debugging

دي أول خطوة — قبل ما تبدأ أي debugging، لازم تفعّل الـ configs الصح.

```bash
# شوف الـ configs الحالية
grep -E "CONFIG_(DEBUG_LIST|LIST_HARDENED|KASAN|KFENCE|LOCKDEP|DEBUG_KERNEL)" /boot/config-$(uname -r)
```

| Config | الوظيفة | الـ Overhead |
|--------|---------|-------------|
| `CONFIG_LIST_HARDENED` | فحص خفيف inline لكل list_add/list_del — بيوقف الـ corruption فوراً | منخفض جداً |
| `CONFIG_DEBUG_LIST` | فحص كامل مع رسائل تفصيلية + stack trace عند الـ corruption | متوسط |
| `CONFIG_KASAN` | Kernel Address Sanitizer — بيكتشف use-after-free و out-of-bounds | عالي (2-3x أبطأ) |
| `CONFIG_KFENCE` | نسخة أخف من KASAN للـ production | منخفض |
| `CONFIG_LOCKDEP` | يتتبع أي lock بيُمسك وقت عمليات الـ list | عالي |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف spinlock bugs (double-lock, unlock بدون lock) | منخفض |
| `CONFIG_PROVE_LOCKING` | يثبت رياضياً إن الـ locking order صح | عالي |
| `CONFIG_DEBUG_PAGEALLOC` | يحول الـ freed pages لـ poison — يكتشف use-after-free | عالي |
| `CONFIG_SLUB_DEBUG` | debugging للـ slab allocator (اللي بيخزن الـ structs) | متوسط |

لتفعيلها:

```bash
scripts/config --enable CONFIG_DEBUG_LIST
scripts/config --enable CONFIG_LIST_HARDENED
scripts/config --enable CONFIG_KASAN
scripts/config --enable CONFIG_KASAN_GENERIC
make olddefconfig
```

---

## 2. الـ Dynamic Debug

```bash
# تفعيل كل messages في list.c
echo "file lib/list.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع stack trace
echo "file lib/list.c +ps" > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع اسم الـ function والـ line number
echo "file lib/list.c +pflT" > /sys/kernel/debug/dynamic_debug/control
```

الـ flags اللي تفيدك:

| Flag | المعنى |
|------|---------|
| `p` | طبع الرسالة |
| `f` | أضف اسم الـ function |
| `l` | أضف رقم الـ line |
| `T` | أضف timestamp |
| `s` | أضف stack trace |

---

## 3. الـ Debugfs Entries

```bash
# تأكد إنه mounted
mount | grep debugfs
# لو مش موجود:
mount -t debugfs none /sys/kernel/debug

# Slab Allocator (اللي بيخزن الـ structs اللي في الـ lists):
cat /sys/kernel/debug/slab/kmalloc-*/alloc_calls
cat /sys/kernel/debug/slab/kmalloc-*/free_calls

# Lockdep:
cat /proc/lockdep
cat /proc/lockdep_stats
cat /proc/lockdep_chains
```

---

## 4. الـ Sysfs Entries المهمة

```bash
# حالة الـ memory و slabs
cat /proc/slabinfo

# إحصائيات الـ memory
cat /proc/meminfo | grep -E "Slab|KReclaimable"
```

---

## 5. الـ ftrace — تتبع عمليات الـ List

```bash
TRACE=/sys/kernel/tracing

# شوف الدوال المتاحة للـ tracing
grep "list_" /sys/kernel/tracing/available_filter_functions | head -20

# إعداد الـ tracer
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo function > $TRACE/current_tracer

# فلتر على دوال الـ list corruption reporting
echo "__list_add_valid_or_report" > $TRACE/set_ftrace_filter
echo "__list_del_entry_valid_or_report" >> $TRACE/set_ftrace_filter

# فعّل الـ stack trace مع كل call
echo 1 > $TRACE/options/func_stack_trace

# زوّد الـ buffer
echo 51200 > $TRACE/buffer_size_kb

# ابدأ الـ tracing
echo 1 > $TRACE/tracing_on

# وقف وشوف النتيجة
echo 0 > $TRACE/tracing_on
cat $TRACE/trace
```

### مثال Output:

```
# TASK-PID    CPU#  IRQS-OFFS  TIMESTAMP    FUNCTION
  kworker/0-45 [000] ....  1234.567890: __list_del_entry_valid_or_report <-list_del
  kworker/0-45 [000] ....  1234.567891: <stack trace>
 => __list_del_entry_valid_or_report
 => list_del
 => some_driver_cleanup   # الـ driver اللي سبب المشكلة
 => device_release
```

---

## 6. KASAN — اكتشاف الـ Use-After-Free

لما تعمل `list_del()`، الكيرنل بيحط:
```c
entry->next = LIST_POISON1;  // 0x100 + delta
entry->prev = LIST_POISON2;  // 0x122 + delta
```

لو حد وصل لـ `entry->next` بعد الحذف، هيحاول يقرأ من العنوان `0x100` — page fault فوري.

مع KASAN، الـ report بيكون أوضح:

```
==================================================================
BUG: KASAN: use-after-free in some_function+0x45/0x120
Read of size 8 at addr ffff888012345678 by task kworker/0:1/123

Allocated by task 456:
 kmalloc+0x28/0x30
 alloc_my_struct+0x45/0x80

Freed by task 123:
 kfree+0x58/0x80
 free_my_struct+0x22/0x50
==================================================================
```

```bash
# تفعيل multi-shot للـ production debugging
echo 1 > /sys/module/kasan/parameters/multi_shot
```

---

## 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel | المعنى | السبب الأكثر احتمالاً | الحل |
|-------------------|---------|----------------------|------|
| `list_add corruption. next->prev should be prev` | الـ next node مش بيشاور على الـ prev الصح | إضافة لـ list متهالكة أو مش initialized | تأكد من `INIT_LIST_HEAD()` قبل الإضافة |
| `list_add corruption. prev->next should be next` | الـ prev node مش بيشاور على الـ next الصح | نفس السبب | نفس الحل |
| `list_del corruption. next->prev should be entry` | الـ next node مش بيشاور على الـ entry | حذف نفس الـ entry مرتين (double free) | استخدم `list_del_init()` بدل `list_del()` |
| `list_del corruption. prev->next should be entry` | الـ prev node مش بيشاور على الـ entry | نفس السبب | نفس الحل |
| `BUG: unable to handle kernel NULL pointer dereference` مع عنوان `0x100` أو `0x122` | وصل لـ poison value — use-after-free في list | استخدام struct اتحذف من الـ list | تتبع الـ ownership وإضافة proper locking |
| `general protection fault` في كود list | محاولة وصل لـ unaligned/invalid pointer | list corruption أو use-after-free | فعّل KASAN وتتبع الـ freed memory |
| `BUG: KASAN: use-after-free in list_del` | وصل لـ entry بعد حذفها من الـ list | race condition | إضافة spinlock أو RCU |
| `WARNING: suspicious RCU usage` مع list operation | عملية list داخل critical section خاطئ | قراءة list بدون `rcu_read_lock()` | أضف `rcu_read_lock()` / `rcu_read_unlock()` |
| `possible circular locking dependency` | deadlock محتمل في locks بتحمي lists | أخذ locks بترتيب مختلف في threads مختلفة | استخدم lockdep وغيّر ترتيب الـ locks |

---

## 8. الـ Lockdep — اكتشاف الـ Locking Problems

```bash
# شوف الـ lockdep stats
cat /proc/lockdep_stats
# lock-classes:                        1234
# circular locking dependencies:          0  # لازم يكون zero!

# لو فيه circular locking dependencies > 0:
dmesg | grep -A 50 "possible circular locking"
```

---

## 9. dump_stack() و WARN_ON() — أماكن استراتيجية

```c
void my_list_add_safe(struct list_head *new, struct list_head *head,
                      spinlock_t *lock)
{
    WARN_ON(!list_empty(new));  // لو new مش empty، طبع warning + stack

    spin_lock(lock);

    if (WARN_ON(head->next == LIST_POISON1 ||
                head->next == LIST_POISON2)) {
        dump_stack();
        spin_unlock(lock);
        return;
    }

    list_add(new, head);
    spin_unlock(lock);
}
```

---

## 10. الـ KFENCE — البديل الخفيف للـ Production

```bash
# تفعيل KFENCE
echo 100 > /sys/module/kfence/parameters/sample_interval

# شوف إحصائيات KFENCE
cat /sys/kernel/debug/kfence/stats
```

---

## 11. سكريبت تشخيص سريع

```bash
#!/bin/bash
# quick_list_debug.sh

echo "=== Kernel Ring Buffer (آخر list errors) ==="
dmesg | grep -E "list_(add|del) corruption|LIST_POISON|use-after-free" | tail -20

echo ""
echo "=== Lockdep Status ==="
cat /proc/lockdep_stats | grep -E "circular|dependency"

echo ""
echo "=== Slab Info (memory usage) ==="
cat /proc/slabinfo | head -5

echo ""
echo "=== KASAN/KFENCE Status ==="
[ -f /sys/kernel/debug/kfence/stats ] && cat /sys/kernel/debug/kfence/stats
```

---

## 12. جدول مقارنة الأدوات

| الأداة | متى تستخدمها | الـ Overhead | بيكتشف |
|--------|-------------|-------------|---------|
| `CONFIG_LIST_HARDENED` | دايماً في production | ~1% | corruption فوري |
| `CONFIG_DEBUG_LIST` | development | ~10% | corruption مع تفاصيل |
| `KASAN` | development/CI | ~3x | use-after-free, OOB |
| `KFENCE` | staging/production | ~1% | use-after-free |
| `lockdep` | development | ~10% | deadlocks, races |
| `ftrace` | production debugging | متغير | أي function call |
| `dynamic_debug` | production | ~0% | pr_debug messages |
| `WARN_ON` | في الكود | ~0% | invariant violations |
| `crash + vmcore` | post-mortem | 0 | بعد الـ panic |

---

## Phase 5: شرح الـ Functions

---

### جدول ملخص الـ Functions

| الـ Function/Macro | الفئة | النوع | الغرض |
|---|---|---|---|
| `INIT_LIST_HEAD` | تهيئة | static inline | تهيئة قائمة فارغة |
| `list_add` | إضافة | static inline | إضافة للأمام (stack) |
| `list_add_tail` | إضافة | static inline | إضافة للخلف (queue) |
| `list_del` | حذف | static inline | حذف مع poison |
| `list_del_init` | حذف | static inline | حذف مع re-init |
| `list_del_init_careful` | حذف | static inline | حذف مع memory barriers |
| `list_replace` | استبدال | static inline | استبدال عنصر بآخر |
| `list_replace_init` | استبدال | static inline | استبدال + تهيئة القديم |
| `list_swap` | استبدال | static inline | تبادل موضعين |
| `list_move` | نقل | static inline | نقل لرأس القائمة |
| `list_move_tail` | نقل | static inline | نقل لذيل القائمة |
| `list_bulk_move_tail` | نقل | static inline | نقل مجموعة للذيل |
| `list_is_first` | استعلام | static inline | هل أول عنصر؟ |
| `list_is_last` | استعلام | static inline | هل آخر عنصر؟ |
| `list_is_head` | استعلام | static inline | هل هو الـ head؟ |
| `list_empty` | استعلام | static inline | هل القائمة فارغة؟ |
| `list_empty_careful` | استعلام | static inline | فارغة؟ (مع barriers) |
| `list_is_singular` | استعلام | static inline | عنصر واحد فقط؟ |
| `list_count_nodes` | استعلام | static inline | عدد العناصر (O(n)) |
| `list_rotate_left` | تدوير | static inline | دوران يساري |
| `list_rotate_to_front` | تدوير | static inline | دوران لعنصر محدد |
| `list_cut_position` | قطع | static inline | قطع بعد عنصر |
| `list_cut_before` | قطع | static inline | قطع قبل عنصر |
| `list_splice` | دمج | static inline | دمج في الأول |
| `list_splice_tail` | دمج | static inline | دمج في الآخر |
| `list_splice_init` | دمج | static inline | دمج + تنظيف المصدر |
| `list_splice_tail_init` | دمج | static inline | دمج في الآخر + تنظيف |
| `list_entry` | وصول | macro | الوصول للـ container |
| `list_first_entry` | وصول | macro | أول عنصر |
| `list_last_entry` | وصول | macro | آخر عنصر |
| `list_first_entry_or_null` | وصول | macro | أول عنصر أو NULL |
| `list_for_each_entry` | iteration | macro | المرور على العناصر |
| `list_for_each_entry_safe` | iteration | macro | المرور مع الحذف |
| `hlist_add_head` | hlist إضافة | static inline | إضافة لـ hash list |
| `hlist_del` | hlist حذف | static inline | حذف مع poison |
| `hlist_del_init` | hlist حذف | static inline | حذف مع re-init |
| `hlist_for_each_entry` | hlist iteration | macro | المرور على hlist |
| `hlist_for_each_entry_safe` | hlist iteration | macro | المرور مع الحذف |

> **ملاحظة مهمة:** كل الـ functions في هذا الملف هي إما `static inline` أو macros — يعني مفيش `EXPORT_SYMBOL` أبدًا. السبب إن هذا ملف header، والـ compiler بيـ"inline" كل حاجة مباشرةً في الكود اللي بيستخدمها. بالتالي، كل هذه الـ functions هي **kernel internal API** متاحة لأي kernel module عن طريق `#include <linux/list.h>`.

---

## الفئة الأولى: التهيئة (Initialization)

---

### `LIST_HEAD_INIT`

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

**ما يفعله:** ماكرو بيهيّئ قائمة في وقت الـ compile — بيخلي الـ `next` و`prev` يشاوروا على نفس الـ struct. بيُستخدم لتهيئة الـ global/static variables.

```c
static struct list_head my_list = LIST_HEAD_INIT(my_list);
```

---

### `LIST_HEAD`

```c
#define LIST_HEAD(name) struct list_head name = LIST_HEAD_INIT(name)
```

**ما يفعله:** ماكرو يعمل `struct list_head` ويهيّئه في سطر واحد.

```c
LIST_HEAD(my_devices);
/* يساوي: struct list_head my_devices = { &my_devices, &my_devices }; */
```

---

### `INIT_LIST_HEAD`

```c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
    WRITE_ONCE(list->next, list);
    WRITE_ONCE(list->prev, list);
}
```

**ما يفعله:** نفس `LIST_HEAD_INIT` بس للـ runtime — لما بتهيّئ قائمة داخل دالة أو في heap.

**المعامِلات:**
- `list`: مؤشر للـ `list_head` اللي هتهيّئه

**مثال عملي:**
```c
struct my_device *dev = kmalloc(sizeof(*dev), GFP_KERNEL);
INIT_LIST_HEAD(&dev->node);  /* لازم تهيّئ قبل ما تضيف */
```

---

## الفئة الثانية: الإضافة العامة (Public Add)

---

### `list_add`

```c
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}
```

**ما يفعله:** بيضيف عنصر جديد **بعد الـ head مباشرةً** — يعني في أول القائمة. ده بيخلي السلوك زي الـ **stack** (آخر واحد دخل، أول واحد اتعمله pop).

**التدفق:**
```
قبل:  [head] <-> [first] <-> ...
بعد:  [head] <-> [new] <-> [first] <-> ...
```

**مثال:**
```c
list_add(&a->node, &my_stack);  /* القائمة: head <-> a */
list_add(&b->node, &my_stack);  /* القائمة: head <-> b <-> a */
```

---

### `list_add_tail`

```c
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}
```

**ما يفعله:** بيضيف عنصر جديد **قبل الـ head** — يعني في آخر القائمة. ده بيخلي السلوك زي الـ **queue**.

**التدفق:**
```
قبل:  ... <-> [last] <-> [head]
بعد:  ... <-> [last] <-> [new] <-> [head]
```

---

## الفئة الثالثة: الحذف (Delete)

---

### `list_del`

```c
static inline void list_del(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->next = LIST_POISON1;
    entry->prev = LIST_POISON2;
}
```

**ما يفعله:** بيشيل العنصر من القائمة وبيحط **poison values** في الـ pointers. الـ poison values هي عناوين غير صالحة (`0x100` و`0x122`) — لو حد حاول يـ dereference بعد الحذف، الـ kernel هيـ crash فورًا.

**متى تستخدم `list_del_init` بدل `list_del`؟**

| الحالة | الأنسب |
|---|---|
| هتعمل `kfree` للعنصر فورًا | `list_del` |
| ممكن تضيفه لقائمة تانية | `list_del_init` |
| هتتحقق لو هو في قائمة بـ `list_empty` | `list_del_init` |
| تريد اكتشاف الـ use-after-free بسرعة | `list_del` |

---

### `list_del_init`

```c
static inline void list_del_init(struct list_head *entry)
{
    __list_del_entry(entry);
    INIT_LIST_HEAD(entry);
}
```

**ما يفعله:** نفس `list_del` لكن بدل الـ poison، بيعيد تهيئة الـ entry كقائمة فارغة.

---

### `list_del_init_careful`

```c
static inline void list_del_init_careful(struct list_head *entry)
{
    __list_del_entry(entry);
    WRITE_ONCE(entry->prev, entry);
    smp_store_release(&entry->next, entry);
}
```

**ما يفعله:** نفس `list_del_init` لكن بيستخدم memory barriers. مخصص للاستخدام مع `list_empty_careful` في بيئات lockless.

---

## الفئة الرابعة: الاستبدال والنقل

---

### `list_replace` / `list_replace_init`

**ما يفعلوا:** `list_replace` بيحل عنصر جديد محل عنصر قديم في نفس الموضع. `list_replace_init` يضيف إعادة تهيئة الـ `old` بعد الاستبدال.

---

### `list_swap`

**ما يفعله:** بيبادل موضع عنصرين في القائمة، مع التعامل مع حالة العنصرين المتجاورين.

---

### `list_move` / `list_move_tail`

```c
static inline void list_move(struct list_head *list, struct list_head *head)
{
    __list_del_entry(list);
    list_add(list, head);
}
```

**ما يفعلوا:** ينقل العنصر من مكانه لأول القائمة (`list_move`) أو آخرها (`list_move_tail`).

**مثال شائع:**
```c
/* نظام LRU Cache — بعد كل access، انقل للأول */
list_move(&page->lru, &lru_head);
```

---

### `list_bulk_move_tail`

**ما يفعله:** بينقل **مجموعة متتالية** من العناصر (من `first` لـ`last`) لآخر قائمة تانية دفعة واحدة.

---

## الفئة الخامسة: الاستعلام (Query)

---

### `list_empty`

```c
static inline int list_empty(const struct list_head *head)
{
    return READ_ONCE(head->next) == head;
}
```

**القيمة المُعادة:** `1` لو فارغة، `0` لو فيها عناصر

---

### `list_empty_careful`

```c
static inline int list_empty_careful(const struct list_head *head)
{
    struct list_head *next = smp_load_acquire(&head->next);
    return (next == head) && (next == head->prev);
}
```

**متى تستخدمه؟** لما thread بيمشي على القائمة بدون lock وتاني ممكن يحذف منها باستخدام `list_del_init_careful`.

---

### `list_is_singular`

**القيمة المُعادة:** `1` لو القائمة فيها **عنصر واحد بالظبط**.

---

### `list_count_nodes`

**تحذير:** O(n) — مكلف على القوائم الكبيرة.

---

## الفئة السادسة: القطع والدمج

---

### `list_cut_position`

**ما يفعله:** بيقطع القائمة من بعد `head` لحد `entry` (شامل) ويحطها في `list`.

```
قبل:  head <-> [A] <-> [B] <-> [C] <-> [D]
               (entry = B)
بعد:  head <-> [C] <-> [D]
list <-> [A] <-> [B]
```

---

### `list_splice_init`

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

**ما يفعله:** بيدمج `list` في أول `head` ويعيد تهيئة `list` — النسخة الأكثر أمانًا من `list_splice`.

---

## الفئة السابعة: ماكروهات الوصول للعناصر

---

### `list_entry`

```c
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
```

**ما يفعله:** بيحسب عنوان الـ struct الكامل من عنوان الـ `list_head` المضمّنة فيه.

```c
struct my_device *dev = list_entry(ptr, struct my_device, node);
```

---

### `list_first_entry_or_null`

```c
#define list_first_entry_or_null(ptr, type, member) ({ \
    struct list_head *head__ = (ptr); \
    struct list_head *pos__ = READ_ONCE(head__->next); \
    pos__ != head__ ? list_entry(pos__, type, member) : NULL; \
})
```

**ما يفعله:** النسخة الآمنة — بيرجع `NULL` لو القائمة فارغة. الاختيار الأفضل في معظم الأحوال.

---

## الفئة الثامنة: ماكروهات الـ Iteration

---

### `list_for_each_entry`

```c
#define list_for_each_entry(pos, head, member)                          \
    for (pos = list_first_entry(head, typeof(*pos), member);            \
         !list_entry_is_head(pos, head, member);                        \
         pos = list_next_entry(pos, member))
```

**أهم macro في الملف كله** — بيمشي على القائمة وبيديك مؤشر للـ **struct الكامل** مباشرةً.

```c
struct device *dev;
list_for_each_entry(dev, &devices, list) {
    printk("Device: %d - %s\n", dev->id, dev->name);
}
```

**تحذير:** ممنوع تحذف عناصر أثناء هذا الـ loop.

---

### `list_for_each_entry_safe`

```c
#define list_for_each_entry_safe(pos, n, head, member)                  \
    for (pos = list_first_entry(head, typeof(*pos), member),            \
         n = list_next_entry(pos, member);                              \
         !list_entry_is_head(pos, head, member);                        \
         pos = n, n = list_next_entry(n, member))
```

**النسخة الآمنة للحذف** أثناء الـ iteration:

```c
struct device *dev, *tmp;
list_for_each_entry_safe(dev, tmp, &devices, list) {
    if (dev->id == target_id) {
        list_del(&dev->list);  /* آمن! */
        kfree(dev);
    }
}
```

---

### جدول كل الـ Iterators

| الـ Iterator | يحمي من الحذف؟ | الاتجاه |
|-------------|----------------|---------|
| `list_for_each(pos, head)` | لا | أمامي |
| `list_for_each_safe(pos, n, head)` | نعم | أمامي |
| `list_for_each_prev(pos, head)` | لا | عكسي |
| `list_for_each_prev_safe(pos, n, head)` | نعم | عكسي |
| `list_for_each_entry(pos, head, member)` | لا | أمامي |
| `list_for_each_entry_reverse(pos, head, member)` | لا | عكسي |
| `list_for_each_entry_safe(pos, n, head, member)` | نعم | أمامي |
| `list_for_each_entry_safe_reverse(...)` | نعم | عكسي |
| `list_for_each_entry_continue(pos, head, member)` | لا | من pos+1 |
| `list_for_each_entry_from(pos, head, member)` | لا | من pos نفسه |
| `list_for_each_entry_continue_reverse(...)` | لا | عكسي من pos-1 |
| `list_for_each_entry_from_reverse(...)` | لا | عكسي من pos |
| `list_for_each_entry_safe_continue(...)` | نعم | من pos+1 |
| `list_for_each_entry_safe_from(...)` | نعم | من pos |

---

## الفئة التاسعة: الـ hlist (Hash List)

---

### `hlist_add_head`

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

**ما يفعله:** بيضيف node في أول الـ hlist. بيربط الـ `pprev` على `&h->first` — ده الفرق الجوهري عن الـ doubly linked list.

---

### `hlist_del` / `hlist_del_init`

نفس فلسفة `list_del` / `list_del_init` لكن للـ hlist.

---

### `hlist_for_each_entry`

```c
#define hlist_for_each_entry(pos, head, member)                         \
    for (pos = hlist_entry_safe((head)->first, typeof(*(pos)), member); \
         pos;                                                           \
         pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))
```

**مثال:**
```c
struct cached_item *item;
int bucket = hash(key) % HASH_SIZE;

hlist_for_each_entry(item, &buckets[bucket], node) {
    if (item->key == key) {
        printk("Found: %s\n", item->value);
        break;
    }
}
```

---

### `hlist_move_list`

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

**ما يفعله:** بينقل كل محتويات hlist من `old` لـ`new` ويفرّغ `old`.

---

### الفرق بين `list_head` و `hlist_head`

| | `list_head` | `hlist_head` + `hlist_node` |
|---|---|---|
| الحجم | 2 pointers (16 bytes على 64-bit) | 1 pointer للـ head (8 bytes) |
| الاستخدام | قوائم عامة | hash table buckets |
| الحذف | O(1) بسهولة | O(1) بفضل pprev الذكي |
| البحث | O(n) | O(1) بعد hash، ثم O(n) في الـ bucket |

---

### ملخص مقارنة: متى تستخدم إيه؟

| الحالة | الـ Macro/Function |
|---|---|
| إضافة لأول القائمة (stack) | `list_add` |
| إضافة لآخر القائمة (queue) | `list_add_tail` |
| حذف + استخدام مرة تانية | `list_del_init` |
| حذف + kfree فورًا | `list_del` |
| المرور على القائمة بأمان | `list_for_each_entry` |
| المرور مع حذف | `list_for_each_entry_safe` |
| الوصول لأول عنصر بأمان | `list_first_entry_or_null` |
| hash tables | `hlist_*` |
| التحقق من الفراغ بأمان | `list_empty` |
| التحقق بدون lock | `list_empty_careful` |
| نقل عنصر لقائمة تانية | `list_move` أو `list_move_tail` |
| دمج قائمتين بأمان | `list_splice_init` |

---

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: RK3562 — Race Condition في قائمة DMA Buffers

---

**العنوان:** Interrupt يدمر قائمة DMA أثناء الـ Traversal

**السياق:**

أنت مهندس في شركة تبني **industrial camera module** فوق لوحة مبنية على **RK3562**. الكاميرا بتنقل frames بشكل مستمر عبر DMA. عندك driver مخصوص يحتفظ بـ list من الـ DMA buffers الـ active:

```c
struct dma_buf_entry {
    void *vaddr;
    dma_addr_t dma_handle;
    size_t size;
    struct list_head node;
};

static LIST_HEAD(active_buffers);
```

الـ driver شغال تمام في بيئة الـ testing، بس في الإنتاج بدأت تظهر kernel panics مرة كل بضع ساعات.

---

**المشكلة:**

```
BUG: general protection fault
RIP: list_del+0x1e
list_for_each_entry_safe corruption: next->prev != entry
```

الكود الأصلي كان:

```c
/* في thread يمشي كل 100ms — مفيش lock! */
void cleanup_finished_buffers(void)
{
    struct dma_buf_entry *entry, *tmp;

    list_for_each_entry_safe(entry, tmp, &active_buffers, node) {
        if (dma_buf_is_done(entry)) {
            list_del(&entry->node);
            dma_free_coherent(dev, entry->size,
                              entry->vaddr, entry->dma_handle);
            kfree(entry);
        }
    }
}

/* Interrupt handler — بيتشغل في أي وقت */
irqreturn_t dma_irq_handler(int irq, void *data)
{
    struct dma_buf_entry *new_entry = kmalloc(...);
    /* بيضيف على القائمة بدون lock */
    list_add_tail(&new_entry->node, &active_buffers);
    return IRQ_HANDLED;
}
```

---

**التحليل:**

`list_for_each_entry_safe` مش thread-safe لوحده. الـ IRQ ممكن يشتغل وسط الـ traversal ويعدّل الـ list pointers.

من `include/linux/list.h`، الـ macro:

```c
#define list_for_each_entry_safe(pos, n, head, member)          \
    for (pos = list_first_entry(head, typeof(*pos), member),    \
        n = list_next_entry(pos, member);                       \
         !list_entry_is_head(pos, head, member);                \
         pos = n, n = list_next_entry(n, member))
```

الـ `n` بيتحسب مرة واحدة بس في كل iteration — لو حد عدّل الـ list بعد حساب `n`، الـ corruption حتصل.

---

**الحل:**

```c
static DEFINE_SPINLOCK(buf_list_lock);

void cleanup_finished_buffers(void)
{
    struct dma_buf_entry *entry, *tmp;
    LIST_HEAD(to_free);
    unsigned long flags;

    /* نشيل الـ entries من القائمة بسرعة تحت lock */
    spin_lock_irqsave(&buf_list_lock, flags);
    list_for_each_entry_safe(entry, tmp, &active_buffers, node) {
        if (dma_buf_is_done(entry))
            list_move_tail(&entry->node, &to_free);
    }
    spin_unlock_irqrestore(&buf_list_lock, flags);

    /* نحرر الـ memory برّا الـ lock */
    list_for_each_entry_safe(entry, tmp, &to_free, node) {
        list_del(&entry->node);
        dma_free_coherent(dev, entry->size,
                          entry->vaddr, entry->dma_handle);
        kfree(entry);
    }
}

irqreturn_t dma_irq_handler(int irq, void *data)
{
    unsigned long flags;
    struct dma_buf_entry *new_entry = kmalloc(sizeof(*new_entry),
                                               GFP_ATOMIC);
    if (!new_entry)
        return IRQ_HANDLED;

    spin_lock_irqsave(&buf_list_lock, flags);
    list_add_tail(&new_entry->node, &active_buffers);
    spin_unlock_irqrestore(&buf_list_lock, flags);

    return IRQ_HANDLED;
}
```

---

**الدرس المستفاد:**

`list_for_each_entry_safe` مش معناها "آمن من الـ interrupts" — معناها آمن من **حذف الـ current entry** بس. استخدم `spin_lock_irqsave` مع الـ interrupt handlers. وفكرة `list_move_tail` لـ local queue تخلي الـ critical section قصير قدر الإمكان.

---

---

### السيناريو 2: AM62x — Memory Leak في hlist لـ Network Connections

---

**العنوان:** Hash Table من Connections مش بتتنضف صح وقت الـ Cleanup

**السياق:**

**industrial gateway** على **AM62x** (Texas Instruments) بيشتغل كـ Modbus TCP gateway. الـ driver عنده hash table بيخزن فيه الـ active TCP connections:

```c
#define CONN_HASH_BITS  8
#define CONN_HASH_SIZE  (1 << CONN_HASH_BITS)

struct modbus_conn {
    uint32_t client_ip;
    uint16_t client_port;
    uint8_t  slave_id;
    struct hlist_node hash_node;
};

static struct hlist_head conn_table[CONN_HASH_SIZE];
```

بعد ساعات من connections و disconnections، الـ `/proc/slabinfo` بيظهر إن الـ `modbus_conn` slab بيكبر باستمرار.

---

**المشكلة:**

```c
void conn_close(struct modbus_conn *conn)
{
    /* المبرمج نسي hlist_del ! */
    pr_info("closing connection from %pI4\n", &conn->client_ip);
    kfree(conn);  /* حرّر الـ memory بس من غير ما يشيله من الـ hash table */
}
```

---

**التحليل:**

بعد `kfree(conn)` بدون `hlist_del`، الـ bucket بيفضل عنده dangling pointer لـ memory اتحررت:

```
conn_table[42].first → [conn_A (freed!)] → [conn_B] → NULL
```

---

**الحل:**

```c
void conn_close(struct modbus_conn *conn)
{
    hlist_del(&conn->hash_node);  /* لازم الأول */
    pr_info("closing connection from %pI4\n", &conn->client_ip);
    kfree(conn);
}

static void __exit modbus_exit(void)
{
    int i;
    struct modbus_conn *conn;
    struct hlist_node *tmp;

    for (i = 0; i < CONN_HASH_SIZE; i++) {
        hlist_for_each_entry_safe(conn, tmp,
                                   &conn_table[i], hash_node) {
            hlist_del(&conn->hash_node);
            kfree(conn);
        }
    }
}
```

---

**الدرس المستفاد:**

في الـ `hlist`، حذف الـ struct من الـ memory (`kfree`) وحذفه من الـ hash table (`hlist_del`) عمليتان منفصلتان. الأفضل دايما تعمل wrapper function `conn_destroy` تجمع فيها `hlist_del` + `kfree`.

---

---

### السيناريو 3: STM32MP1 — Deadlock في UART TX Queue

---

**العنوان:** list_for_each_entry داخل Spinlock يسبب Deadlock مع IRQ

**السياق:**

**IoT sensor hub** على **STM32MP1** بيجمع بيانات من 8 sensors عبر UART. الـ system بيـ freeze تماماً في الـ production عندما يرتفع الـ load.

---

**المشكلة:**

```c
irqreturn_t uart_tx_irq(int irq, void *data)
{
    struct uart_tx_req *req, *tmp;

    spin_lock(&tx_lock);  /* يأخد الـ lock */

    list_for_each_entry_safe(req, tmp, &tx_pending, node) {
        req->complete_cb(req->cb_data);  /* ← المشكلة: يستدعي uart_send */
        list_del(&req->node);
        kfree(req);
        break;
    }

    spin_unlock(&tx_lock);
    return IRQ_HANDLED;
}
```

الـ `complete_cb` بتستدعي `uart_send` اللي بتحاول تاخد `tx_lock` تاني → deadlock.

---

**الحل:**

```c
irqreturn_t uart_tx_irq(int irq, void *data)
{
    struct uart_tx_req *req = NULL;
    unsigned long flags;

    spin_lock_irqsave(&tx_lock, flags);
    if (!list_empty(&tx_pending)) {
        req = list_first_entry(&tx_pending,
                               struct uart_tx_req, node);
        list_del_init(&req->node);  /* list_del_init مش list_del */
    }
    spin_unlock_irqrestore(&tx_lock, flags);  /* نسيب الـ lock */

    /* نستدعي الـ callback برّا الـ lock */
    if (req) {
        if (req->complete_cb)
            req->complete_cb(req->cb_data);  /* آمن دلوقتي */
        kfree(req);
    }

    return IRQ_HANDLED;
}
```

---

**الدرس المستفاد:**

قاعدتان ذهبيتان:
1. لو spinlock بيتشارك بين IRQ context وprocess context، لازم process context يستخدم `spin_lock_irqsave`.
2. لا تستدعي callbacks وانت شايل lock — ناول الـ entry لمكان تاني وسيب الـ lock الأول.

---

---

### السيناريو 4: i.MX8 — Use-After-Free ظهر بـ KASAN من خلط list_del وlist_del_init

---

**العنوان:** KASAN بيكشف وصول لـ LIST_POISON بعد list_del غلط

**السياق:**

**automotive ECU** على **i.MX8** لإدارة CAN bus messages. الـ KASAN يطلع:

```
BUG: KASAN: use-after-free in list_add_tail+0x4c
Read of size 8 at addr ffff888012345678 by task can_worker/42

Freed by task can_timeout:
    can_msg_timeout_handler+0x38
    list_del+0x1e
```

---

**المشكلة:**

```c
/* Timeout handler */
void can_msg_timeout(struct timer_list *t)
{
    struct can_msg *msg = from_timer(msg, t, timeout_timer);

    list_del(&msg->node);   /* يحط poison pointers */
    kfree(msg);
}

/* ACK handler — ممكن يشتغل بعد الـ timeout */
void can_msg_acked(struct can_msg *msg)
{
    if (msg->state == MSG_PENDING) {
        list_del(&msg->node);   /* msg->node.next = POISON! */
        kfree(msg);
    }
}
```

الـ `list_del` في الـ timeout بيحط `LIST_POISON1` في `msg->node.next`. لما `can_msg_acked` يشتغل بعدها، يحاول يقرأ من `msg->node.next` = `0xdead000000000100` → KASAN يصرخ.

---

**الحل:**

```c
struct can_msg {
    /* ... نفس الـ fields ... */
    struct kref ref;
    bool   removed;
    spinlock_t lock;
};

void can_msg_timeout(struct timer_list *t)
{
    struct can_msg *msg = from_timer(msg, t, timeout_timer);
    unsigned long flags;

    spin_lock_irqsave(&msg->lock, flags);
    if (!msg->removed) {
        msg->removed = true;
        list_del_init(&msg->node);  /* list_del_init مش list_del */
    }
    spin_unlock_irqrestore(&msg->lock, flags);

    kref_put(&msg->ref, can_msg_release);
}

void can_msg_acked(struct can_msg *msg)
{
    unsigned long flags;

    spin_lock_irqsave(&msg->lock, flags);
    if (!msg->removed && msg->state == MSG_PENDING) {
        msg->removed = true;
        list_del_init(&msg->node);
    }
    spin_unlock_irqrestore(&msg->lock, flags);

    kref_put(&msg->ref, can_msg_release);
}
```

---

**الدرس المستفاد:**

- استخدم `list_del_init` لما تحتاج تـ check بعدين إذا كان الـ node في قائمة أم لأ.
- `list_empty(&node)` بتشتغل صح بعد `list_del_init` لكن بترجع garbage بعد `list_del`.
- الـ KASAN أداة رائعة — شغّلها في الـ development boards دايماً.

---

---

### السيناريو 5: Allwinner H616 — Workqueue وIRQ بيعدّلوا نفس القائمة

---

**العنوان:** Android TV Box بيـ Crash عشواشي بسبب List Corruption بين Workqueue وIRQ

**السياق:**

**Android TV box** على **Allwinner H616** بيستخدم driver لإدارة HDMI hotplug events. الـ system بيـ crash عشواشياً:

```
kernel BUG at lib/list_debug.c:29!
list_add corruption. next->prev should be prev
```

---

**المشكلة:**

```c
/* IRQ handler — مفيش lock */
irqreturn_t hdmi_hotplug_irq(int irq, void *data)
{
    struct hdmi_event *ev = kmalloc(sizeof(*ev), GFP_ATOMIC);
    /* ... */
    list_add_tail(&ev->node, &event_queue);  /* خطر على SMP */
    schedule_work(&hdmi_work);
    return IRQ_HANDLED;
}

/* Workqueue handler — مفيش lock */
void hdmi_process_events(struct work_struct *work)
{
    struct hdmi_event *ev, *tmp;
    list_for_each_entry_safe(ev, tmp, &event_queue, node) {
        process_hdmi_event(ev->type, ev->connector_id);
        list_del(&ev->node);
        kfree(ev);
    }
}
```

على الـ multi-core، الـ IRQ ممكن يشتغل على core مختلف في نفس اللحظة التي يشتغل فيها الـ workqueue.

---

**الحل:**

```c
static DEFINE_SPINLOCK(event_lock);

irqreturn_t hdmi_hotplug_irq(int irq, void *data)
{
    struct hdmi_event *ev;
    unsigned long flags;

    ev = kmalloc(sizeof(*ev), GFP_ATOMIC);
    if (!ev) return IRQ_HANDLED;

    ev->type = hdmi_read_status() ? HDMI_CONNECT : HDMI_DISCONNECT;
    INIT_LIST_HEAD(&ev->node);

    spin_lock_irqsave(&event_lock, flags);
    list_add_tail(&ev->node, &event_queue);
    spin_unlock_irqrestore(&event_lock, flags);

    schedule_work(&hdmi_work);
    return IRQ_HANDLED;
}

void hdmi_process_events(struct work_struct *work)
{
    struct hdmi_event *ev, *tmp;
    LIST_HEAD(local_queue);
    unsigned long flags;

    /*
     * list_splice_init — بتنقل كل event_queue لـ local_queue
     * وتخلي event_queue فاضية في عملية واحدة تحت lock
     */
    spin_lock_irqsave(&event_lock, flags);
    list_splice_init(&event_queue, &local_queue);
    spin_unlock_irqrestore(&event_lock, flags);

    /* local_queue ملكنا لوحدنا — مش محتاجين lock */
    list_for_each_entry_safe(ev, tmp, &local_queue, node) {
        process_hdmi_event(ev->type, ev->connector_id);
        list_del(&ev->node);
        kfree(ev);
    }
}
```

---

**الدرس المستفاد:**

- `list_splice_init` من أقوى أدوات الـ list API — بتخلي الـ critical section صغير جداً حتى لو الـ processing بتاخد وقت.
- على الـ SMP، افترض دايماً إن الـ IRQ ممكن يشتغل على core تاني في نفس اللحظة.
- فعّل `CONFIG_DEBUG_LIST=y` و `CONFIG_PROVE_LOCKING=y` في الـ development.

---

## Phase 7: مصادر ومراجع

---

### 7.1 مقالات LWN.net

| العنوان | الرابط | ما يخص |
|---|---|---|
| Circular linked lists in the kernel | https://lwn.net/Articles/336255/ | شرح API الـ list_head والاستخدامات الأساسية |
| Safety and linked lists | https://lwn.net/Articles/724313/ | CONFIG_DEBUG_LIST والـ hardening |
| A deeper look at list corruption | https://lwn.net/Articles/836953/ | كيف يكتشف الكيرنل فساد القوائم |
| READ_ONCE and WRITE_ONCE | https://lwn.net/Articles/624126/ | لماذا نحتاج memory ordering في القوائم |
| Rcu and linked lists | https://lwn.net/Articles/262464/ | استخدام RCU مع list_head |
| Poison pointers in the kernel | https://lwn.net/Articles/260117/ | فلسفة LIST_POISON1 وLIST_POISON2 |
| The container_of() macro | https://lwn.net/Articles/444336/ | شرح container_of والـ intrusive lists |
| hlist and hash tables in the kernel | https://lwn.net/Articles/273308/ | تصميم hlist_head واستخدامها في الـ hash tables |
| Memory barriers in the kernel | https://lwn.net/Articles/576486/ | smp_store_release وsmp_load_acquire |

---

### 7.2 توثيق رسمي داخل النواة

```
Documentation/core-api/kernel-api.rst
    — توثيق API للـ list_head وكل الماكروات
    — القسم "List Management Functions"

Documentation/core-api/memory-barriers.rst
    — شرح READ_ONCE/WRITE_ONCE
    — smp_load_acquire/smp_store_release

Documentation/RCU/listRCU.rst
    — كيف تستخدم list_head مع RCU
    — list_add_rcu, list_del_rcu, list_for_each_entry_rcu

include/linux/list.h
    — الملف الأصلي — التعليقات فيه توثيق رسمي بحد ذاتها
```

---

### 7.3 Commits مهمة في تاريخ الكيرنل

```bash
# للبحث يدوياً عن أي تغيير
git log --all --oneline --follow include/linux/list.h

# أو على الويب:
# https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/include/linux/list.h
```

---

### 7.4 نقاشات Mailing List

```
موضوع: "hardened usercopy and list validation"
https://lkml.org/lkml/2017/10/20/1

موضوع: "list: introduce list_rotate_to_front()"
https://lkml.org/lkml/2019/1/31/1

موضوع: "Add list_is_head() helper"
https://lkml.org/lkml/2021/3/15/1
```

---

### 7.5 الكتب المرجعية

#### Linux Device Drivers (LDD3) — O'Reilly

```
متاح مجاناً: https://lwn.net/Kernel/LDD3/

الفصول ذات الصلة:
- Chapter 11: Data Types in the Kernel
  → القسم عن "Linked Lists"
  → شرح list_head وكيفية استخدامها في الـ drivers
  → container_of وكيف يختلف نهج الكيرنل عن المكتبات العادية
```

#### Linux Kernel Development — Robert Love

```
الفصول ذات الصلة:
- Chapter 6: Kernel Data Structures
  → القسم "Linked Lists" — الأشمل والأوضح
  → شرح list_head بالتفصيل
  → لماذا استخدم الكيرنل intrusive lists
  → hlist ومتى تستخدمها

ملاحظة: هذا الكتاب هو الأفضل لفهم list.h بعمق
```

#### Understanding the Linux Kernel — Bovet & Cesati

```
- Chapter 3: Processes
  → list_head في task_struct
  → كيف تُربط العمليات في قوائم
```

#### Embedded Linux Primer — Christopher Hallinan

```
- Chapter 14: Linux Kernel Debugging
  → CONFIG_DEBUG_LIST وكيفية استخدامه لاكتشاف الأخطاء
```

---

### 7.6 Kernelnewbies.org

```
الصفحة الرئيسية: https://kernelnewbies.org/

صفحات ذات صلة:
- KernelHacking Guide: https://kernelnewbies.org/KernelHacking
- Kernel Glossary: https://kernelnewbies.org/KernelGlossary
- Linux Kernel Map: https://kernelnewbies.org/KernelMap
```

---

### 7.7 elinux.org

```
- Kernel data structures: https://elinux.org/Kernel_data_structures
- Debugging the kernel: https://elinux.org/Debugging_the_kernel
- Kernel Instrumentation: https://elinux.org/Kernel_Instrumentation
```

---

### 7.8 مصادر إضافية أونلاين

```
Elixir Cross Referencer — لتصفح السورس:
https://elixir.bootlin.com/linux/latest/source/include/linux/list.h
— اضغط على أي دالة وشوف كل الأماكن اللي استُخدمت فيها

The Linux Kernel API Documentation:
https://www.kernel.org/doc/html/latest/core-api/kernel-api.html
— التوثيق الرسمي المولّد من السورس
— قسم "List Management Functions"
```

---

### 7.9 كلمات البحث

```bash
# للبحث العام
"linux kernel list_head tutorial"
"linux kernel intrusive linked list"
"container_of macro linux kernel"
"hlist_head kernel hash table"
"CONFIG_DEBUG_LIST kernel"
"CONFIG_LIST_HARDENED"
"LIST_POISON1 kernel"
"kernel linked list corruption detection"
"list_for_each_entry safe"
"READ_ONCE WRITE_ONCE list kernel"
"smp_load_acquire list_empty_careful"

# للبحث في السورس مباشرة
grep -r "struct list_head" include/ --include="*.h" | wc -l
grep -r "list_for_each_entry" drivers/ --include="*.c" | head -20
grep -r "struct hlist_head" include/ --include="*.h"
```

---

### 7.10 خريطة التعلم

```
مستوى مبتدئ:
    kernelnewbies.org → LDD3 Chapter 11 → elixir.bootlin.com

مستوى متوسط:
    Robert Love Chapter 6 → LWN Articles → Documentation/core-api/

مستوى متقدم:
    git log include/linux/list.h → LKML discussions → ULK Bovet

للعمل العملي:
    elixir.bootlin.com → grep في السورس → قراءة حالات استخدام حقيقية
```
