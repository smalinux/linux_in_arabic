## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem ده؟

الـ `debugfs.h` جزء من subsystem اسمه **DRIVER CORE, KOBJECTS, DEBUGFS AND SYSFS** — محتفظ بيه Greg Kroah-Hartman وRafael J. Wysocki. الـ subsystem ده هو اللي بيدير كل علاقة الـ kernel بالـ devices وبيوفر واجهات للـ userspace تتكلم مع الـ kernel.

---

### القصة من الأول: ليه debugfs موجود أصلاً؟

تخيل إنك بتكتب driver لـ USB أو GPU أو network card. فجأة في مشكلة — الـ device بيتصرف غريب. عايز تعرف إيه قيمة register معين جوه الـ hardware، أو عايز تشوف عداد معين جوه الـ driver زي "كام packet اتبعت؟" أو "كام interrupt جه؟".

عندك 3 خيارات قبل debugfs:

| الخيار | المشكلة |
|--------|---------|
| **`/proc`** | مصمم للـ processes، مش للـ drivers — بيتحط فيه حاجات عشوائية ومش منظم |
| **`/sys` (sysfs)** | للـ device model بس — له قواعد صارمة، مش مناسب للـ debug المؤقت |
| **`printk`** | بيملي الـ log بكلام مش محتاجه دايمًا، وبطيء |

**الـ debugfs** جه عشان يحل الأمر بطريقة بسيطة: filesystem صغير بيتـmount على `/sys/kernel/debug`، الـ driver يعمل فيه files تمثل أي variable أو register أو حالة داخلية بيحتاج يتابعها، وأي حد عايز يعرف يعمل `cat /sys/kernel/debug/my_driver/counter` وخلاص.

---

### التشبيه البسيط (ELI5)

تخيل الـ kernel زي مصنع ضخم. الـ `/proc` زي لوحة الإعلانات الرسمية للعمال، والـ `/sys` زي دليل المعدات الرسمي. أما **debugfs** فهو زي **لوحة المهندسين الداخلية** — بيكتبوا عليها أي أرقام أو حالات يحتاجوها أثناء الـ debugging، من غير ما يزقزعوا في الدليل الرسمي.

المهندس (developer) بيعمل directory باسم الـ driver بتاعه، وجواه files بأسماء واضحة زي `tx_packets` أو `reg_status` أو `error_count` — وممكن يقرأ منها ويكتب فيها من الـ userspace.

---

### ليه هو مفيد؟

- **سهل جدًا**: سطر واحد `debugfs_create_u32("counter", 0444, dir, &my_counter)` يعمل file بيقرأ قيمة variable مباشرة.
- **مؤقت بطبيعته**: مفيش حد بيحط حاجة في debugfs ويعتبرها ABI رسمي — ده بالعكس، الـ kernel لا يضمن إن الـ files دي تفضل موجودة أو بنفس الشكل.
- **منفصل عن الـ production**: ممكن تعمل kernel بدون `CONFIG_DEBUG_FS` وكل الكود اللي بيستخدمه يشتغل عادي (الـ stubs بترجع error أو تعمل no-op).
- **يدعم hardware registers مباشرة**: في `debugfs_regset32` لـ dump كامل لـ registers الـ hardware.

---

### الهدف من الـ `debugfs.h` تحديدًا

الـ header ده هو **الواجهة الكاملة** اللي أي driver أو subsystem يستخدمها. بيوفر:

1. **إنشاء files وdirectories** جوه `/sys/kernel/debug`
2. **Helpers جاهزة** لأنواع البيانات المختلفة: `u8`, `u16`, `u32`, `u64`, `bool`, `str`, `blob`, hex
3. **Hardware register dumps** عبر `debugfs_regset32`
4. **حماية من الـ race conditions** عبر `debugfs_file_get/put` و`debugfs_cancellation`
5. **Stub implementations** لما `CONFIG_DEBUG_FS` يكون معطل — الكود الباقي ميتكسرش

---

### السيناريو: driver يستخدم debugfs

```
Driver يتـload
     │
     ▼
debugfs_create_dir("my_driver", NULL)   ← ينشئ /sys/kernel/debug/my_driver/
     │
     ├── debugfs_create_u32("tx_count", 0444, dir, &tx_count)
     │         └── /sys/kernel/debug/my_driver/tx_count
     │
     ├── debugfs_create_regset32("regs", 0444, dir, &regset)
     │         └── /sys/kernel/debug/my_driver/regs  ← dump كل الـ registers
     │
     └── DEFINE_DEBUGFS_ATTRIBUTE(fops, get_fn, set_fn, "%llu\n")
               └── debugfs_create_file("threshold", 0644, dir, NULL, &fops)
                         └── قراءة وكتابة في variable complex

Driver يتـunload → debugfs_remove(dir)  ← يحذف كل الـ tree
```

---

### الـ structs الأساسية في الـ header

| Struct | الغرض |
|--------|--------|
| `debugfs_blob_wrapper` | تغليف binary data (pointer + size) لعرضه كـ file |
| `debugfs_reg32` | اسم وـoffset لـ register واحد في الـ hardware |
| `debugfs_regset32` | مجموعة registers مع base address للـ dump |
| `debugfs_u32_array` | مصفوفة u32 تتعرض كـ file |
| `debugfs_cancellation` | بيانات للـ cancellation الآمن لعمليات جارية |
| `debugfs_short_fops` | واجهة مبسطة (read/write/llseek فقط) بدون boilerplate |

---

### الـ Macro المهم: `DEFINE_DEBUGFS_ATTRIBUTE`

```c
DEFINE_DEBUGFS_ATTRIBUTE(fops, get_fn, set_fn, "%llu\n");
```

الـ macro ده بيولد `struct file_operations` كامل من مجرد getter وsetter. بدله كنت محتاج تكتب ~30 سطر. الـ `_SIGNED` variant للأرقام السالبة، والـ `_XSIGNED` هو اللي بيعمل التفريق.

---

### الفرق بين `debugfs_create_file` و`debugfs_create_file_unsafe`

- **`debugfs_create_file`**: آمنة — بتستخدم proxy يمنع الـ race بين قراءة الـ file وتحميل/تفكيك الـ module.
- **`debugfs_create_file_unsafe`**: بدون proxy — الـ driver نفسه مسؤول عن الحماية. أسرع لكن خطر لو الـ module اتـunload وفيه open file.

---

### الـ Cancellation System

مشكلة: لو userspace فتح file وبيقرأ، والـ driver اتـunload في نفس الوقت — crash مضمون. الحل:

```c
debugfs_file_get(dentry);    // "أنا بستخدم الـ file ده، متمسحوش"
// ... الـ read/write operations ...
debugfs_file_put(dentry);    // "خلاص، ممكن تمسحه"
```

وبالنسبة للعمليات الطويلة (زي blocking reads):

```c
struct debugfs_cancellation cancel;
debugfs_enter_cancellation(file, &cancel);  // سجّل callback للـ cancel
// ... العملية ...
debugfs_leave_cancellation(file, &cancel);
```

---

### الملفات المرتبطة اللي المبرمج يعرفها

#### Core Implementation
| الملف | الدور |
|-------|-------|
| `fs/debugfs/inode.c` | إنشاء الـ filesystem، mount، إدارة الـ inodes والـ dentries |
| `fs/debugfs/file.c` | proxy file_operations، الـ read/write للـ attributes، الـ cancellation |
| `fs/debugfs/internal.h` | structs الداخلية: `debugfs_inode_info`، `debugfs_fsdata` |

#### Headers مرتبطة
| الملف | الدور |
|-------|-------|
| `include/linux/debugfs.h` | **(الملف ده)** — الـ public API الكامل |
| `include/linux/fs.h` | `struct dentry`، `struct inode`، `struct file_operations` |
| `include/linux/seq_file.h` | `struct seq_file` للـ multi-line output |

#### Rust Bindings (حديثة)
| الملف | الدور |
|-------|-------|
| `rust/kernel/debugfs.rs` | bindings للـ Rust drivers |
| `rust/kernel/debugfs/` | wrappers إضافية |

#### Documentation
| الملف | الدور |
|-------|-------|
| `Documentation/filesystems/debugfs.rst` | الدليل الرسمي للاستخدام |
| `Documentation/core-api/kernel-api.rst` | ذِكر ضمن الـ kernel API |

#### أمثلة
| الملف | الدور |
|-------|-------|
| `samples/rust/rust_debugfs.rs` | مثال Rust |
| `samples/rust/rust_debugfs_scoped.rs` | مثال scoped للـ Rust |

---

### ملاحظة على الـ Stubs

لما `CONFIG_DEBUG_FS=n`، كل الـ functions بتترجع `ERR_PTR(-ENODEV)` أو بتعمل no-op. الـ callers المفروض **يـignore الـ error** من `debugfs_create_*` — الـ kernel صرّح بكده في الـ comments — عشان الـ driver يشتغل حتى لو debugfs مش موجود:

```c
/* NOTE: it's expected that most callers should _ignore_ the errors returned
 * by this function. Other debugfs functions handle the fact that the "dentry"
 * passed to them could be an error and they don't crash in that case. */
```
## Phase 2: شرح الـ debugfs Framework

### المشكلة — ليه debugfs موجود أصلاً؟

في أثناء تطوير driver أو subsystem، المهندس محتاج يعرف:
- الـ internal state بتاع الـ hardware register دلوقتي إيه؟
- الـ counter ده بيزيد ولا بيفضل ثابت؟
- الـ flag ده اتضبط ولا لا؟

**المشكلة** إن الـ kernel ما بيعملش output زي userspace — مفيش `printf` بيطلع على screen. الحلول التقليدية:

| الحل | المشكلة |
|------|---------|
| `printk` | بتملا الـ ring buffer، مش interactive |
| `/proc` | مخصوص لـ process info، API معقد، مفيش حرية في التنسيق |
| `/sys` (sysfs) | مخصوص لـ device model، بيحتاج كل file تعبر عن attribute واحد بضوابط صارمة |
| Hardware JTAG | مش دايماً متاح، بطيء |

الـ kernel كان محتاج مكان تقول فيه للـ driver: "اعمل اللي محتاجه، ماشي بدون قيود — بس فقط للـ debugging".

---

### الحل — نهج الـ kernel

**debugfs** هو filesystem بسيط جداً بيتعمل mount على `/sys/kernel/debug/`. الفكرة:

1. أي driver ممكن يعمل directory جوه debugfs
2. ينشئ files بتعبر عن أي حاجة جوهاً
3. الـ userspace يقرا أو يكتب في الـ files دي بـ `cat` / `echo` عادي
4. لما الـ kernel يتبني بدون `CONFIG_DEBUG_FS`، كل الـ API بتبقى stubs فارغة — يعني zero overhead في production

الـ design principle الأساسي: **no rules, no overhead, no guarantees** — debugfs مش ABI ومش مضمون يتثبت بين kernel versions.

---

### التشبيه الواقعي — لوحة الـ Diagnostics في المصنع

تخيل مصنع فيه ماكينة إنتاج. الماكينة بتشتغل عادي — بس عايز تفهم اللي بيحصل جوهاً:

| عنصر في التشبيه | المقابل في debugfs |
|-----------------|-------------------|
| **لوحة الـ Diagnostics** الجنب الماكينة | الـ `/sys/kernel/debug/` directory |
| **قسم خاص بالماكينة** على اللوحة | الـ directory اللي الـ driver بينشئه بـ `debugfs_create_dir()` |
| **شاشة صغيرة** بتوضح RPM الحالي | file بيعمله `debugfs_create_u32("rpm", ...)` |
| **زر reset** على اللوحة | file بـ write handler بيعمل reset لـ counter |
| **شاشة hex** بتوضح قراءات الـ registers | file بيعمله `debugfs_create_regset32()` |
| **اللوحة دي ما بتتبنيش** في الماكينة اللي بتتباع | `CONFIG_DEBUG_FS=n` → stubs فارغة |
| البيانات على اللوحة **مش ملزمة** وممكن تتغير | debugfs مش ABI — ما فيش ضمانات |
| **مهندس الصيانة** اللي بيقرا اللوحة | الـ developer اللي بيعمل `cat /sys/kernel/debug/...` |

الجزء الدقيق: اللوحة دي مش جزء من الماكينة الأساسية — لو شيلتها الماكينة تفضل تشتغل. ده بالظبط اللي بيعنيه إن debugfs fails gracefully — الـ driver المفروض يشتغل حتى لو `debugfs_create_file()` رجعت error.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        USERSPACE                                │
│                                                                 │
│   cat /sys/kernel/debug/my_driver/status                        │
│   echo 1 > /sys/kernel/debug/my_driver/reset                   │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │  VFS syscalls (read/write/open)
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     VFS LAYER                                   │
│                                                                 │
│   sys_open() → dentry lookup → inode → file_operations         │
│                                                                 │
│   (الـ VFS هو الطبقة اللي بتوحد كل filesystems في الـ kernel)  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    debugfs CORE                                 │
│            (fs/debugfs/inode.c, fs/debugfs/file.c)             │
│                                                                 │
│  ┌────────────────┐   ┌──────────────────┐   ┌──────────────┐  │
│  │  debugfs_root  │   │ debugfs_create_* │   │ simple_attr  │  │
│  │  (struct dentry│   │  functions       │   │ framework    │  │
│  │   *debugfs_root│   │                  │   │              │  │
│  │   = global)    │   │ creates inodes + │   │ get/set      │  │
│  └────────────────┘   │ dentries         │   │ callbacks    │  │
│                        └──────────────────┘   └──────────────┘  │
│                                                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │  registers/creates files
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
┌─────────────────────┐    ┌──────────────────────────────────────┐
│   DRIVER A          │    │   DRIVER B (e.g., i2c, gpu, usb)    │
│                     │    │                                      │
│ debugfs_create_dir  │    │ debugfs_create_dir                   │
│  ("my_driver", NULL)│    │  ("gpu_debug", NULL)                 │
│                     │    │                                      │
│ debugfs_create_u32  │    │ debugfs_create_regset32              │
│  ("irq_count", ...)  │    │  ("regs", ...)                       │
│                     │    │                                      │
│ debugfs_create_file │    │ debugfs_create_blob                  │
│  ("dump", ...)       │    │  ("fw_dump", ...)                    │
└─────────────────────┘    └──────────────────────────────────────┘
              │                         │
              └────────────┬────────────┘
                           │ mounts at
                           ▼
              /sys/kernel/debug/
              ├── my_driver/
              │   ├── irq_count      (r/w u32)
              │   └── dump           (custom fops)
              └── gpu_debug/
                  ├── regs           (regset32)
                  └── fw_dump        (blob)
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ core abstraction في debugfs هي: **كل حاجة بتبقى file**.

الـ kernel بيستخدم الـ **VFS layer** (Virtual File System) كـ interface موحد. الـ debugfs بيعمل نفسه filesystem حقيقي — بس بدون disk، كل حاجة في الـ RAM فقط. الـ abstraction بتتكون من:

#### 1. الـ `dentry` — الاسم في شجرة الـ filesystem

**الـ `dentry`** (directory entry) هو الـ struct اللي الـ VFS بيستخدمه علشان يمثل اسم ملف أو directory. كل `debugfs_create_*()` بترجع `struct dentry *`.

```c
struct dentry *parent = debugfs_create_dir("my_driver", NULL);
//                                                        ^^^^
//                                          NULL = root of debugfs
```

#### 2. الـ `inode` — البيانات الفعلية

الـ inode بيخزن الـ permissions وبيشاور على الـ `file_operations`. الـ debugfs بيعمل inode لكل file تنشئها.

#### 3. الـ `file_operations` — اللي بيحصل لما تعمل open/read/write

ده الـ vtable اللي الـ driver بيعرّف من خلاله سلوك الـ file.

---

### الـ Structs الأساسية وعلاقتها ببعض

```
struct debugfs_blob_wrapper          struct debugfs_reg32
┌──────────────────┐                 ┌─────────────────┐
│ void     *data   │                 │ char    *name   │
│ ulong     size   │                 │ ulong    offset │
└──────────────────┘                 └─────────────────┘
        │                                    │
        │ wrapped in                         │ array of
        ▼                                    ▼
debugfs_create_blob()          struct debugfs_regset32
                               ┌──────────────────────┐
                               │ const reg32  *regs   │◄── array of reg32
                               │ int           nregs  │
                               │ void __iomem *base   │◄── MMIO base addr
                               │ struct device *dev   │◄── for Runtime PM
                               └──────────────────────┘
                                        │
                                        ▼
                               debugfs_create_regset32()
                               → يعمل file بيطبع كل register بـ offset + value
```

```
struct debugfs_u32_array
┌────────────────────┐
│ u32  *array        │◄── pointer to actual array in driver
│ u32   n_elements   │
└────────────────────┘
         │
         ▼
debugfs_create_u32_array()
→ file بيطبع كل elements على سطر
```

---

### الـ `simple_attr` Framework — قلب الـ DEFINE_DEBUGFS_ATTRIBUTE

لما تعمل file بتقرا/تكتب قيمة بسيطة (u64)، ما محتاجش تكتب `file_operations` من الصفر. الـ framework بيوفرلك الـ macro `DEFINE_DEBUGFS_ATTRIBUTE`:

```c
/* مثال: driver عنده counter عايز يعرضه */
static u64 my_counter;

static int counter_get(void *data, u64 *val)
{
    *val = my_counter;   /* driver يقرا القيمة */
    return 0;
}

static int counter_set(void *data, u64 val)
{
    my_counter = val;    /* driver يكتب القيمة */
    return 0;
}

/* الـ macro بتولد file_operations كاملة */
DEFINE_DEBUGFS_ATTRIBUTE(counter_fops, counter_get, counter_set, "%llu\n");

/* في init: */
debugfs_create_file("counter", 0644, parent, NULL, &counter_fops);
```

الـ macro بتعمل:

```
DEFINE_DEBUGFS_ATTRIBUTE(counter_fops, get, set, fmt)
           │
           ├─► static int counter_fops_open(inode, file) {
           │       simple_attr_open(inode, file, get, set, fmt);
           │   }
           │
           └─► static const struct file_operations counter_fops = {
                   .open    = counter_fops_open,
                   .read    = debugfs_attr_read,    ← kernel handles formatting
                   .write   = debugfs_attr_write,   ← kernel handles parsing
                   .release = simple_attr_release,
               }
```

**الـ `debugfs_attr_read`** بيعمل إيه؟ بيستدعي `get()` callback → بيحول القيمة لـ string بالـ `fmt` → بيبعتها للـ userspace.
**الـ `debugfs_attr_write`** بيعمل إيه؟ بيقرا string من userspace → يحولها لـ u64 → يستدعي `set()` callback.

---

### الـ `debugfs_short_fops` — تبسيط إضافي

الـ `struct debugfs_short_fops` أبسط من `file_operations` الكاملة:

```c
struct debugfs_short_fops {
    ssize_t (*read)  (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    loff_t  (*llseek)(struct file *, loff_t, int);
};
```

مفيش `open`، مفيش `release`، مفيش module reference counting. الـ kernel بيتكفل بالـ `simple_open()` وبـ reference counting تلقائياً. الـ `debugfs_create_file()` بيستخدم `_Generic` (C11) علشان يعرف تلقائياً هل انت باعت `file_operations` أو `debugfs_short_fops`:

```c
#define debugfs_create_file(name, mode, parent, data, fops) \
    _Generic(fops,                                           \
        const struct file_operations *: debugfs_create_file_full,   \
        const struct debugfs_short_fops *: debugfs_create_file_short \
        ...                                                  \
    )(name, mode, parent, data, NULL, fops)
```

---

### الـ `seq_file` — للـ output الكبير

لما الـ output ممكن يكون أكبر من page size واحدة، الـ debugfs بيستخدم **الـ `seq_file` subsystem** (موجود في `include/linux/seq_file.h`). الـ `seq_file` بيحل مشكلة الـ buffering:

```c
struct seq_file {
    char   *buf;        /* internal buffer */
    size_t  size;       /* buffer size */
    size_t  count;      /* bytes written so far */
    loff_t  index;      /* current position for iteration */
    const struct seq_operations *op;   /* driver callbacks */
    void   *private;    /* driver-specific data */
    ...
};

struct seq_operations {
    void *(*start)(struct seq_file *m, loff_t *pos);  /* begin iteration */
    void  (*stop) (struct seq_file *m, void *v);       /* end iteration */
    void *(*next) (struct seq_file *m, void *v, loff_t *pos); /* advance */
    int   (*show) (struct seq_file *m, void *v);       /* print one item */
};
```

الـ pattern:

```
userspace: read()
     │
     ▼
seq_read()          ← kernel handles buffer + pagination
     │
     ├── start()    ← driver: "ابدأ من item رقم pos"
     ├── show()     ← driver: "اطبع الـ item الحالي"
     ├── next()     ← driver: "انتقل للـ item الجاي"
     └── stop()     ← driver: "خلص"
```

الـ `debugfs_create_devm_seqfile()` هي أبسط شكل — بتاخد `read_fn` واحدة بس وبتعمل كل حاجة تانية تلقائياً.

---

### الـ `debugfs_cancellation` — حل مشكلة الـ Race Condition

لما الـ userspace بيعمل `read()` على debugfs file، وفي نفس الوقت الـ driver بيعمل unload — في race condition خطيرة. الـ `debugfs_file_get()` / `debugfs_file_put()` بيحلوا ده.

بس في حالة أصعب: لو الـ read handler نفسه بيعمل عملية blocking طويلة (مثلاً بيستنى hardware) ومحتاج يتوقف لو الـ driver اتشال — هنا بييجي `debugfs_cancellation`:

```c
/* في الـ read handler: */
struct debugfs_cancellation cancel;

debugfs_enter_cancellation(file, &cancel);
/* العملية الطويلة هنا */
debugfs_leave_cancellation(file, &cancel);
```

لو الـ driver اتشال أثناء العملية، الـ kernel بيستدعي الـ `cancel()` callback علشان ينهي العملية بـ clean way.

---

### الـ `arch_debugfs_dir` — الـ arch-specific entries

```c
extern struct dentry *arch_debugfs_dir;
```

ده global dentry بيشير لـ directory خاص بالـ architecture. مثلاً على ARM: `/sys/kernel/debug/arm/` — الـ architecture-specific debug files بتتحط هنا.

---

### الـ Lifecycle الكامل لـ debugfs entry في Driver

```
Driver Module Init                     Driver Module Exit
─────────────────                      ─────────────────
debugfs_create_dir()                   debugfs_remove()
      │                                      │
      │ creates dentry                       │ removes dentry + all children
      ▼                                      │ (recursive by default)
debugfs_create_u32()  ←──────┐              ▼
debugfs_create_file()         │        /sys/kernel/debug/
debugfs_create_regset32()     │         └── my_driver/  ← GONE
      │                       │
      │ store dentry ptr       │
      ▼                       │
struct dentry *dir = ...  ────┘
/* يحتفظ بيه علشان يمررهولـ debugfs_remove() */
```

**نقطة مهمة:** `debugfs_remove()` دلوقتي recursive بالكامل — يعني لو حذفت الـ parent directory، كل الـ children بتتحذف تلقائياً. الـ macro `debugfs_remove_recursive` هو alias لـ `debugfs_remove`.

---

### الـ CONFIG_DEBUG_FS=n — الـ Stub Pattern

لما الـ kernel يتبني بدون debugfs، كل الـ functions بتبقى static inline stubs:

```c
/* مثال من الـ header: */
static inline struct dentry *debugfs_create_dir(const char *name,
                                                 struct dentry *parent)
{
    return ERR_PTR(-ENODEV);  /* مش NULL — علشان الـ driver يقدر يكتشف الـ error */
}

static inline void debugfs_create_u32(const char *name, umode_t mode,
                                       struct dentry *parent, u32 *value)
{ }  /* no-op */
```

الـ design decision المهم: الـ functions اللي بترجع `void` بترجع no-op — الـ driver مش محتاج يعمل أي check. الـ functions اللي بترجعوا `dentry*` بترجعوا `ERR_PTR(-ENODEV)` مش NULL — علشان الـ driver ممكن يعمل `IS_ERR()` check لو عايز.

**النتيجة:** driver مكتوب صح مش بيلمس الـ debugfs dentry تاني بعد ما بيعمله create — والـ functions التانية كلها بتتعامل مع الـ ERR_PTR بـ gracefully.

---

### ملكية الـ Subsystem — إيه اللي debugfs بيعمله وإيه اللي بيوكله للـ Driver

| المسؤولية | debugfs Core | Driver |
|------------|-------------|--------|
| إنشاء الـ inode والـ dentry | ✓ | |
| إدارة الـ filesystem tree | ✓ | |
| ربط الـ VFS بالـ file_operations | ✓ | |
| الـ locking عند file removal | ✓ | |
| formatting الـ u8/u16/.../u64 لـ ASCII | ✓ | |
| تحديد محتوى الـ file | | ✓ |
| تحديد الـ permissions المناسبة | | ✓ |
| الـ cleanup عند module unload | | ✓ (استدعاء debugfs_remove) |
| معنى البيانات اللي بتتكتب | | ✓ |
| الـ thread safety للـ data نفسها | | ✓ |

---

### مثال عملي كامل — Driver بيعرض حالة SPI controller

```c
#include <linux/debugfs.h>
#include <linux/module.h>

struct my_spi {
    void __iomem *base;
    u32           tx_count;
    u32           rx_count;
    struct dentry *debugfs_root;  /* نحتفظ بالـ root dentry */

    /* للـ regset */
    struct debugfs_regset32  regset;
};

/* تعريف الـ registers اللي عايز تعرضها */
static const struct debugfs_reg32 spi_regs[] = {
    { "CTRL",    0x00 },
    { "STATUS",  0x04 },
    { "FIFO",    0x08 },
};

static int my_spi_probe(struct platform_device *pdev)
{
    struct my_spi *spi = /* allocate ... */;

    /* إنشاء directory رئيسي */
    spi->debugfs_root = debugfs_create_dir("my_spi", NULL);

    /* counters بسيطة */
    debugfs_create_u32("tx_count", 0444, spi->debugfs_root, &spi->tx_count);
    debugfs_create_u32("rx_count", 0444, spi->debugfs_root, &spi->rx_count);

    /* dump كل الـ registers */
    spi->regset.regs  = spi_regs;
    spi->regset.nregs = ARRAY_SIZE(spi_regs);
    spi->regset.base  = spi->base;
    debugfs_create_regset32("regs", 0444, spi->debugfs_root, &spi->regset);

    return 0;
}

static int my_spi_remove(struct platform_device *pdev)
{
    struct my_spi *spi = platform_get_drvdata(pdev);

    /* حذف الـ directory وكل حاجة جوهاً دفعة واحدة */
    debugfs_remove(spi->debugfs_root);

    return 0;
}
```

بعد الـ probe:
```bash
$ ls /sys/kernel/debug/my_spi/
regs  rx_count  tx_count

$ cat /sys/kernel/debug/my_spi/tx_count
42

$ cat /sys/kernel/debug/my_spi/regs
CTRL  = 0x00000001
STATUS= 0x00000003
FIFO  = 0x000000ff
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Config Options والـ Macros — Cheatsheet

| العنصر | النوع | الغرض |
|---|---|---|
| `CONFIG_DEBUG_FS` | config option | لو مش موجود، كل الـ functions بتبقى stubs بترجع `ERR_PTR(-ENODEV)` |
| `DEFINE_DEBUGFS_ATTRIBUTE` | macro | بيعمل `file_operations` كاملة لـ unsigned value |
| `DEFINE_DEBUGFS_ATTRIBUTE_SIGNED` | macro | نفسه بس لـ signed value |
| `DEFINE_DEBUGFS_ATTRIBUTE_XSIGNED` | macro | الـ base macro اللي الاتنين بيستخدموه |
| `debugfs_create_file` | `_Generic` macro | بيختار تلقائياً بين `_full` و`_short` حسب نوع الـ `fops` |
| `debugfs_create_file_aux` | `_Generic` macro | زي السابق بس بيقبل `aux` pointer إضافي |
| `debugfs_create_file_aux_num` | macro | بيحول رقم `n` لـ `void*` وبيمرره كـ `aux` |
| `debugfs_get_aux_num` | macro | بيرجع الـ `aux` كـ `unsigned long` |
| `debugfs_remove_recursive` | macro alias | مجرد alias لـ `debugfs_remove` (اللي بتشيل recursive تلقائياً) |

| Flag / Permission | المعنى |
|---|---|
| `umode_t mode` | permissions بتاعة الـ file في الـ VFS (مثلاً `0444` للـ read-only) |
| `__iomem` | pointer لـ memory-mapped I/O، لازم يتعامل معاه بـ I/O accessors |
| `__user` | pointer لـ user-space memory، لا يُـdereference مباشرة |
| `__acquires` / `__releases` | annotations للـ sparse tool بتوضح الـ locking على الـ cancellation |

---

### 1. الـ Structs المهمة

#### `struct debugfs_blob_wrapper`

**الغرض:** بيلف binary blob عشان يتعرضه كـ read-only file في debugfs.

```c
struct debugfs_blob_wrapper {
    void *data;          /* pointer للـ binary data */
    unsigned long size;  /* حجم الـ data بالـ bytes */
};
```

**الاستخدام:** بتخليه يشاور على أي buffer في الـ kernel وتعمله `debugfs_create_blob()` فيظهر كـ file في `/sys/kernel/debug/`.

**الربط مع structs تانية:** `debugfs_create_blob()` بتاخده وبتعمل `dentry` يشاور عليه.

---

#### `struct debugfs_reg32`

**الغرض:** بيمثل register واحد له اسم وـ offset من base address.

```c
struct debugfs_reg32 {
    char *name;          /* اسم الـ register للعرض */
    unsigned long offset; /* offset من الـ base address */
};
```

**الربط:** دايماً بيجي ضمن array جوه `debugfs_regset32`.

---

#### `struct debugfs_regset32`

**الغرض:** بيجمع مجموعة registers لجهاز معين ويعمل لها file واحد في debugfs يعرضهم كلهم.

```c
struct debugfs_regset32 {
    const struct debugfs_reg32 *regs; /* array من الـ registers */
    int nregs;                         /* عدد الـ registers */
    void __iomem *base;                /* base address للجهاز */
    struct device *dev;                /* optional: للـ Runtime PM */
};
```

**الربط مع structs تانية:**
- بيشاور على `debugfs_reg32[]` كـ array
- بيشاور على `struct device` للـ Runtime PM (بيصحى الجهاز قبل ما يقرأ الـ registers)
- `debugfs_create_regset32()` بتاخده وبتربطه بـ `dentry`

---

#### `struct debugfs_u32_array`

**الغرض:** بيلف array من u32 values عشان يتعرضها كـ file واحد.

```c
struct debugfs_u32_array {
    u32 *array;       /* pointer للـ array */
    u32 n_elements;   /* عدد العناصر */
};
```

**الربط:** `debugfs_create_u32_array()` بتاخده وبتعمل file بيعرض كل العناصر مفصولة بـ spaces.

---

#### `struct debugfs_short_fops`

**الغرض:** بديل خفيف لـ `file_operations` بيدعم بس read/write/llseek — من غير module reference أو release handler.

```c
struct debugfs_short_fops {
    ssize_t (*read)(struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *);
    loff_t  (*llseek)(struct file *, loff_t, int);
};
```

**الغرض من الوجود:** الـ kernel بيتولى الـ `open` و`release` تلقائياً عبر `simple_open()`، فمش محتاج الـ driver يعمل boilerplate.

**الربط:** الـ `_Generic` macro في `debugfs_create_file` بيختار `debugfs_create_file_short()` لما الـ `fops` يكون من النوع ده.

---

#### `struct debugfs_cancellation`

**الغرض:** بيسمح لـ file operation تسجّل callback يُنادى لو جه طلب إلغاء (مثلاً لما الـ module بيتشال وفيه read/write شغالة).

```c
// context_lock_struct() macro بيضيف locking annotations تلقائياً
struct debugfs_cancellation {
    struct list_head list;              /* للربط في قائمة الـ cancellations */
    void (*cancel)(struct dentry *, void *); /* الـ callback */
    void *cancel_data;                  /* بيانات إضافية للـ callback */
};
```

**الربط:**
- بيتربط في list داخلية في الـ debugfs inode
- `debugfs_enter_cancellation()` بتضيفه للـ list (`__acquires`)
- `debugfs_leave_cancellation()` بتشيله من الـ list (`__releases`)

---

#### `struct seq_file` (من `seq_file.h`)

**الغرض:** بيمثل session لقراءة ملف بـ streaming، بيُستخدم كتير مع debugfs لعرض بيانات متعددة السطور.

```c
struct seq_file {
    char *buf;                      /* buffer مؤقت للكتابة */
    size_t size;                    /* حجم الـ buffer */
    size_t from;                    /* offset القراءة الحالي */
    size_t count;                   /* كمية البيانات في الـ buffer */
    size_t pad_until;               /* للـ padding */
    loff_t index;                   /* iterator position */
    loff_t read_pos;                /* position في الـ output stream */
    struct mutex lock;              /* يحمي العمليات على الـ seq_file */
    const struct seq_operations *op; /* iterator operations */
    int poll_event;                 /* counter لـ poll() */
    const struct file *file;        /* الـ VFS file المرتبط */
    void *private;                  /* بيانات خاصة بالـ driver */
};
```

**الربط مع debugfs:**
- `debugfs_create_devm_seqfile()` بتعمل file بيستخدم `seq_file` داخلياً
- `debugfs_print_regs32()` بتاخد `seq_file*` كـ output stream
- **الـ `mutex lock`** هو المسؤول عن serialization أثناء القراءة

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
                    ┌─────────────────────┐
                    │   struct dentry      │  ← VFS entry في /sys/kernel/debug/
                    │   (kernel/VFS)       │
                    └──────────┬──────────┘
                               │ يُنشأ بواسطة
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
  debugfs_create_file()  debugfs_create_dir()  debugfs_create_blob()
          │                                         │
          │ fops pointer                            │
          ▼                                         ▼
  ┌───────────────────┐               ┌──────────────────────────┐
  │  file_operations  │               │  debugfs_blob_wrapper    │
  │  OR               │               │  ┌────────────────────┐  │
  │  debugfs_short_   │               │  │ void *data         │  │
  │  fops             │               │  │ unsigned long size  │  │
  └───────────────────┘               │  └────────────────────┘  │
                                      └──────────────────────────┘

  ┌──────────────────────────────────────────┐
  │         debugfs_regset32                  │
  │  ┌────────────────────────────────────┐  │
  │  │ const debugfs_reg32 *regs ─────────┼──┼──► [ {name, offset}, ... ]
  │  │ int nregs                          │  │    debugfs_reg32[]
  │  │ void __iomem *base ────────────────┼──┼──► hardware registers
  │  │ struct device *dev ────────────────┼──┼──► struct device (Runtime PM)
  │  └────────────────────────────────────┘  │
  └──────────────────────────────────────────┘
             │
             │ debugfs_create_regset32()
             ▼
        struct dentry (file في debugfs)
             │
             │ read()
             ▼
        debugfs_print_regs32()
             │
             ▼
        struct seq_file
        ┌─────────────────────────┐
        │ char *buf               │
        │ struct mutex lock       │
        │ const seq_operations *op│
        │ void *private           │
        └─────────────────────────┘

  ┌──────────────────────────────────────────┐
  │         debugfs_cancellation              │
  │  ┌────────────────────────────────────┐  │
  │  │ struct list_head list              │  │◄──── inode internal list
  │  │ void (*cancel)(dentry*, void*)     │  │
  │  │ void *cancel_data                  │  │
  │  └────────────────────────────────────┘  │
  └──────────────────────────────────────────┘
```

---

### 3. مخطط دورة حياة الـ debugfs entry

```
CREATION
────────
  driver_init()
      │
      ├─► debugfs_create_dir("mydriver", NULL)
      │       └─► dentry* parent_dir
      │
      ├─► debugfs_create_u32("count", 0444, parent_dir, &my_count)
      │       └─► dentry* count_file
      │
      └─► debugfs_create_file("state", 0644, parent_dir, priv, &my_fops)
              └─► dentry* state_file

USAGE
─────
  user: cat /sys/kernel/debug/mydriver/count
      │
      ├─► VFS open() → debugfs_file_get(dentry)  [refcount++]
      │
      ├─► VFS read() → my_fops.read() أو debugfs_attr_read()
      │
      └─► VFS release() → debugfs_file_put(dentry)  [refcount--]

TEARDOWN
────────
  driver_exit()
      │
      └─► debugfs_remove(parent_dir)
              │
              ├─► بتشيل الـ dir وكل اللي جوه recursively
              │
              ├─► لو فيه file مفتوح:
              │       └─► debugfs_cancellation callbacks بتتنادى
              │           عشان الـ operations الشغالة تعرف توقف
              │
              └─► الـ dentry بيتحرر بعد ما الـ refcount يوصل 0
```

---

### 4. مخطط تدفق الاستدعاءات (Call Flow Diagrams)

#### 4.1 إنشاء file بسيط للـ u32

```
driver calls: debugfs_create_u32("val", 0644, dir, &my_val)
    │
    └─► debugfs_create_file_full("val", 0644, dir, &my_val, NULL, &u32_fops)
            │
            └─► VFS: alloc dentry + inode
                    │
                    └─► inode->i_private = &my_val
                        inode->i_fop = &u32_fops
                        └─► return dentry*
```

#### 4.2 قراءة attribute file

```
user: read(/sys/kernel/debug/mydriver/val)
    │
    ├─► VFS: open() → __fops_open()  [generated by DEFINE_DEBUGFS_ATTRIBUTE]
    │           │
    │           └─► simple_attr_open(inode, file, get_fn, set_fn, fmt)
    │                   └─► يحفظ get/set/fmt في file->private_data
    │
    ├─► VFS: read() → debugfs_attr_read(file, buf, len, ppos)
    │           │
    │           ├─► debugfs_file_get(dentry)  [protection من الـ remove]
    │           │
    │           ├─► simple_attr_read()
    │           │       └─► (*get)(file, &val)  ← driver's getter function
    │           │               └─► snprintf(buf, fmt, val)
    │           │
    │           └─► debugfs_file_put(dentry)
    │
    └─► VFS: release() → simple_attr_release()
```

#### 4.3 قراءة regset32

```
user: cat /sys/kernel/debug/mydriver/regs
    │
    └─► VFS read() → seq_read()
            │
            └─► seq_operations.show()
                    │
                    └─► debugfs_print_regs32(seq, regs, nregs, base, prefix)
                                │
                                ├─► [optional] pm_runtime_get_sync(dev)
                                │
                                ├─► for each reg in regs[]:
                                │       val = readl(base + reg.offset)
                                │       seq_printf(s, "%s%s = 0x%08x\n",
                                │                  prefix, reg.name, val)
                                │
                                └─► [optional] pm_runtime_put(dev)
```

#### 4.4 الـ cancellation flow

```
driver: module unload starts
    │
    └─► debugfs_remove(dentry)
            │
            ├─► kernel: walks cancellation list in inode
            │
            └─► for each debugfs_cancellation:
                    └─► (*cancel)(dentry, cancel_data)
                                │
                                └─► driver callback:
                                        e.g. wake_up() أو set flag
                                        عشان الـ blocked read/write يرجع


file handler (concurrent):
    debugfs_enter_cancellation(file, &canc)   [__acquires]
        └─► adds canc to inode's list
    │
    ├─► do_blocking_operation()  ← ممكن يتكنسل هنا
    │
    └─► debugfs_leave_cancellation(file, &canc)  [__releases]
            └─► removes canc from list
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks الموجودة

| الـ Lock | في إيه | بيحمي إيه |
|---|---|---|
| `struct mutex lock` في `seq_file` | `seq_file.lock` | serializes الـ read operations على نفس الـ seq_file session |
| `debugfs_file_get/put` | internal D_INODE reference counting | بيمنع الـ dentry يتحذف وفيه file مفتوح |
| `context_lock_struct` على `debugfs_cancellation` | annotations على الـ struct | بيوضح للـ sparse إن الـ enter/leave بيعملوا acquire/release |
| `simple_attr` internal mutex | في `simple_attr_open` | بيمنع concurrent read/write على نفس الـ attribute file |

#### ترتيب الـ Locks (Lock Ordering)

```
الترتيب الآمن:
─────────────
1. debugfs_file_get()        ← أول حاجة (عشان تأمن الـ dentry)
       │
       ▼
2. simple_attr mutex          ← جوه الـ simple_attr read/write
       │
       ▼
3. driver's own lock          ← لو الـ getter/setter بيمسك lock
       │
       ▼
4. seq_file->lock             ← في الـ seq_file operations
       │
       ▼
5. debugfs_file_put()         ← آخر حاجة (بعد ما تخلص)
```

#### ملاحظات مهمة على الـ Locking

- **لا تمسك أي driver lock وتنادي `debugfs_remove()`** — ده ممكن يعمل deadlock لو الـ cancellation callback حاول يمسك نفس الـ lock.
- **الـ `debugfs_file_get/put`** بيعملوا حماية من الـ TOCTOU بين الـ open والـ remove.
- **الـ `__iomem` registers** في `debugfs_regset32` بيتقرأوا بـ `readl()` اللي هي `volatile` access، محتاج Runtime PM lock لو الجهاز ممكن يكون sleeping — عشان كده الـ `dev` field موجود.
- **لو بتستخدم `debugfs_create_file_unsafe()`** فأنت مسؤول عن الـ locking بنفسك وضمان إن الـ data مش بتتحذف وفيه file مفتوح.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ APIs — Cheatsheet

#### فئة: File & Directory Creation

| Function | Signature Summary | الغرض |
|---|---|---|
| `debugfs_create_file` | `(name, mode, parent, data, fops) → dentry*` | إنشاء ملف عام بـ custom fops |
| `debugfs_create_file_aux` | `(name, mode, parent, data, aux, fops) → dentry*` | زي السابق + auxiliary data |
| `debugfs_create_file_full` | `(name, mode, parent, data, aux, fops_full) → dentry*` | Backend لـ full `file_operations` |
| `debugfs_create_file_short` | `(name, mode, parent, data, aux, fops_short) → dentry*` | Backend لـ `debugfs_short_fops` |
| `debugfs_create_file_unsafe` | `(name, mode, parent, data, fops) → dentry*` | بدون file-removal protection |
| `debugfs_create_file_size` | `(name, mode, parent, data, fops, file_size)` | زي create_file + يحدد حجم افتراضي |
| `debugfs_create_dir` | `(name, parent) → dentry*` | إنشاء directory |
| `debugfs_create_symlink` | `(name, parent, dest) → dentry*` | إنشاء symlink |
| `debugfs_create_automount` | `(name, parent, f, data) → dentry*` | automount point |

#### فئة: Scalar Value Files

| Function | Type | Format |
|---|---|---|
| `debugfs_create_u8` | `u8 *` | decimal |
| `debugfs_create_u16` | `u16 *` | decimal |
| `debugfs_create_u32` | `u32 *` | decimal |
| `debugfs_create_u64` | `u64 *` | decimal |
| `debugfs_create_ulong` | `unsigned long *` | decimal |
| `debugfs_create_x8` | `u8 *` | hex |
| `debugfs_create_x16` | `u16 *` | hex |
| `debugfs_create_x32` | `u32 *` | hex |
| `debugfs_create_x64` | `u64 *` | hex |
| `debugfs_create_xul` | `unsigned long *` | hex (x32 أو x64 حسب المنصة) |
| `debugfs_create_size_t` | `size_t *` | decimal |
| `debugfs_create_atomic_t` | `atomic_t *` | decimal |
| `debugfs_create_bool` | `bool *` | Y/N |
| `debugfs_create_str` | `char **` | string |

#### فئة: Complex Data Files

| Function | بيعمل إيه |
|---|---|
| `debugfs_create_blob` | يعرض binary blob كـ binary file |
| `debugfs_create_regset32` | يعرض set من الـ 32-bit MMIO registers |
| `debugfs_print_regs32` | يطبع الـ registers في `seq_file` |
| `debugfs_create_u32_array` | يعرض array من `u32` values |
| `debugfs_create_devm_seqfile` | seqfile مربوطة بـ device lifetime |

#### فئة: Lookup & Removal

| Function | بيعمل إيه |
|---|---|
| `debugfs_lookup` | بيدور على entry بالاسم |
| `debugfs_remove` | بيحذف entry (وكل اللي تحته recursively) |
| `debugfs_remove_recursive` | alias لـ `debugfs_remove` |
| `debugfs_lookup_and_remove` | lookup + remove في خطوة واحدة |
| `debugfs_change_name` | rename لـ dentry موجود |

#### فئة: File Access Protection

| Function | بيعمل إيه |
|---|---|
| `debugfs_file_get` | يمسك reference قبل access |
| `debugfs_file_put` | يحرر reference بعد access |
| `debugfs_enter_cancellation` | يسجل cancellation handler |
| `debugfs_leave_cancellation` | يلغي تسجيل الـ handler |

#### فئة: Attribute Read/Write Helpers

| Function | بيعمل إيه |
|---|---|
| `debugfs_attr_read` | قراءة قيمة من `simple_attr` |
| `debugfs_attr_write` | كتابة قيمة unsigned |
| `debugfs_attr_write_signed` | كتابة قيمة signed |
| `debugfs_read_file_bool` | قراءة bool من user space |
| `debugfs_write_file_bool` | كتابة bool من user space |
| `debugfs_read_file_str` | قراءة string من user space |

#### فئة: Miscellaneous

| Function | بيعمل إيه |
|---|---|
| `debugfs_initialized` | هل debugfs اتعمله mount؟ |
| `debugfs_get_aux` | يجيب الـ aux pointer من الـ file |
| `debugfs_get_aux_num` | يجيب الـ aux كـ unsigned long |

---

### Group 1: File & Directory Creation

الغرض من الـ group ده هو إنشاء entries داخل الـ debugfs VFS tree. كل entry بترجع `struct dentry *` — وده اللي بتستخدمه بعدين لو حبيت تحذف الـ entry أو تتحقق من وجودها. الـ design philosophy هنا إن الـ driver مش محتاج يـcheck errors: لو الـ dentry كان `ERR_PTR`، الـ functions التانية بتـhandle ده بنفسها وما بتـcrash-sh.

---

#### `debugfs_create_file` (macro)

```c
#define debugfs_create_file(name, mode, parent, data, fops) \
    _Generic(fops,                                           \
        const struct file_operations *: debugfs_create_file_full,   \
        const struct debugfs_short_fops *: debugfs_create_file_short, \
        struct file_operations *: debugfs_create_file_full,         \
        struct debugfs_short_fops *: debugfs_create_file_short)     \
    (name, mode, parent, data, NULL, fops)
```

**الـ macro ده بيستخدم `_Generic` عشان يختار تلقائياً** بين `debugfs_create_file_full` و`debugfs_create_file_short` بناءً على نوع الـ `fops` pointer في compile-time. لو بعتله `struct file_operations *` هيروح للـ full path، ولو بعتله `struct debugfs_short_fops *` هيروح للـ short path. الـ `aux` بيتبعت `NULL` تلقائياً.

**Parameters:**
- `name` — اسم الملف في الـ debugfs tree
- `mode` — permissions زي `0444` للـ read-only أو `0644` للـ read-write
- `parent` — الـ dentry بتاع الـ parent directory، أو `NULL` للـ root
- `data` — pointer اختياري بيتخزن في `inode->i_private`، بتوصله في الـ `open()` handler
- `fops` — إما `struct file_operations *` أو `struct debugfs_short_fops *`

**Return value:** `struct dentry *` — valid pointer في حالة النجاح، `ERR_PTR(-errno)` في حالة الفشل

**Key details:** الـ callers **يجب أن يتجاهلوا** الـ error return في الغالب، لأن كل الـ debugfs functions التانية بتـhandle الـ `ERR_PTR` بأمان.

---

#### `debugfs_create_file_full`

```c
struct dentry *debugfs_create_file_full(const char *name, umode_t mode,
                                        struct dentry *parent, void *data,
                                        const void *aux,
                                        const struct file_operations *fops);
```

ده الـ **backend الحقيقي** لإنشاء ملفات debugfs بـ full `file_operations`. بيعمل inode جديد، بيعمل dentry، وبيربطهم في الـ VFS tree. الـ `aux` pointer ده extra data فوق الـ `data`، بتوصله عن طريق `debugfs_get_aux()`.

**Parameters:**
- `name`, `mode`, `parent`, `data` — نفس الكلام الفات
- `aux` — auxiliary pointer إضافي مستقل عن `i_private`، useful لو محتاج تحط context منفصل
- `fops` — full `struct file_operations` — بتوفر `open`, `read`, `write`, `release`, إلخ

**Return value:** `struct dentry *` أو `ERR_PTR`

**Key details:** الـ module reference management بيتعمل يدوياً من خلال الـ `fops->owner`. لو الـ module اتـunload وفيه file مفتوح، ممكن يحصل use-after-free — عشان كده `debugfs_file_get/put` موجودين.

---

#### `debugfs_create_file_short`

```c
struct dentry *debugfs_create_file_short(const char *name, umode_t mode,
                                         struct dentry *parent, void *data,
                                         const void *aux,
                                         const struct debugfs_short_fops *fops);
```

نسخة مبسطة بتستخدم `struct debugfs_short_fops` اللي فيها بس `read`, `write`, `llseek`. الـ kernel بيـwrap الـ `open` و`release` تلقائياً بـ `simple_open()` و لا شيء. مش محتاج module reference أو cleanup — الـ kernel بيتكفل بيهم.

**Parameters:** نفس `debugfs_create_file_full` بس بـ `debugfs_short_fops *` بدل `file_operations *`

**Return value:** `struct dentry *` أو `ERR_PTR`

**Key details:** مفيش `module reference counting` هنا — الـ short_fops path مصمم للـ simple read/write operations اللي ما بتحتاجش لـ module lifetime management.

---

#### `debugfs_create_file_unsafe`

```c
struct dentry *debugfs_create_file_unsafe(const char *name, umode_t mode,
                                          struct dentry *parent, void *data,
                                          const struct file_operations *fops);
```

**بيعمل نفس `debugfs_create_file_full` بدون حماية** من الـ file-removal race. لو الـ `debugfs_remove()` اتنادت وفيه process لسه معاها الملف مفتوح، ممكن يحصل crash. مستخدمة بس في حالات خاصة جداً زي الـ `seq_file` اللي بتـmanage الـ lifetime بنفسها.

**Caller context:** بيتنادى بس لو الـ caller ضامن إن الـ data مش هتتحرر وفيه access جاري.

---

#### `debugfs_create_file_size`

```c
void debugfs_create_file_size(const char *name, umode_t mode,
                               struct dentry *parent, void *data,
                               const struct file_operations *fops,
                               loff_t file_size);
```

زي `debugfs_create_file_full` بس بيحدد `i_size` للـ inode بعد الإنشاء. بيفيد لو محتاج الـ `stat()` يرجع size معين — زي مثلاً لو بتـexpose binary blob وعايز `wc -c` تشتغل صح.

**Parameters:**
- `file_size` — القيمة اللي بتتحط في `inode->i_size`

**Return value:** void — بيتجاهل الـ errors داخلياً

---

#### `debugfs_create_dir`

```c
struct dentry *debugfs_create_dir(const char *name, struct dentry *parent);
```

بيعمل directory جديدة في الـ debugfs tree. الـ directories دي بتتحذف مع كل اللي جوّاها لو نادينا `debugfs_remove()` عليها.

**Parameters:**
- `name` — اسم الـ directory
- `parent` — parent directory dentry، أو `NULL` للـ root

**Return value:** `struct dentry *` أو `ERR_PTR(-ENODEV)` لو debugfs disabled

**Key details:** لو حبيت تحذف directory فيها files، `debugfs_remove()` هيتكفل بكل اللي جوّاها recursively.

---

#### `debugfs_create_symlink`

```c
struct dentry *debugfs_create_symlink(const char *name, struct dentry *parent,
                                       const char *dest);
```

بيعمل symbolic link في debugfs. الـ `dest` بيتنسخ internally. بيفيد لو عايز تعمل alias لـ path موجود.

**Parameters:**
- `dest` — الـ path اللي الـ symlink بيشير له — بيتنسخ داخلياً بـ `kstrdup`

**Return value:** `struct dentry *` أو `ERR_PTR`

---

#### `debugfs_create_automount`

```c
struct dentry *debugfs_create_automount(const char *name,
                                         struct dentry *parent,
                                         debugfs_automount_t f,
                                         void *data);
```

بيعمل **automount point** في debugfs. لما حد يحاول يـaccess الـ path ده، الـ kernel بيـcall الـ callback `f` عشان يعمل `vfsmount` جديد. بيستخدمه tracing subsystem عشان يعمل tracefs تحت `/sys/kernel/debug/tracing`.

**Parameters:**
- `f` — `debugfs_automount_t` وهي `struct vfsmount *(*)(struct dentry *, void *)` — الـ callback اللي بيرجع الـ mount
- `data` — بيتبعت للـ callback عند الـ automount

**Return value:** `struct dentry *` أو `ERR_PTR`

---

### Group 2: Lookup & Removal

الـ group ده بيـmanage دورة حياة الـ entries الموجودة في الـ tree. الـ removal في debugfs بقى unified — مفيش فرق بين حذف ملف أو directory أو recursive removal.

---

#### `debugfs_lookup`

```c
struct dentry *debugfs_lookup(const char *name, struct dentry *parent);
```

بيـsearch عن entry بالاسم تحت الـ `parent`. بيعمل `dget()` على الـ result لو لقاه — يعني الـ **caller مسؤول عن `dput()`** بعد ما خلص.

**Parameters:**
- `name` — اسم الـ entry المطلوبة
- `parent` — الـ directory اللي بيدور فيها — مش nullable

**Return value:** `struct dentry *` مع زيادة reference count، أو `NULL` لو مش موجود، أو `ERR_PTR(-ENODEV)` لو debugfs disabled

**Key details:** لازم `dput()` على الـ return value لو مش `NULL` ومش `ERR_PTR`.

---

#### `debugfs_remove`

```c
void debugfs_remove(struct dentry *dentry);
```

**ده الـ function الرئيسي للحذف** في debugfs. بيحذف الـ entry وكل اللي تحتها recursively لو كانت directory. بيـwait لحد ما كل الـ open file handles تتقفل قبل ما يكمل — ده بيضمن إن الـ data بتاعتك ما تتحررش وفيه access جاري.

**Parameters:**
- `dentry` — الـ dentry المراد حذفها — لو `NULL` أو `ERR_PTR`، الـ function بتـreturn بأمان

**Return value:** void

**Key details:**
- بيـblock لحد ما الـ file close — ده مهم جداً في الـ module unload path
- `debugfs_remove_recursive` هو مجرد `#define` لنفس الـ function
- آمن لو بعتله `NULL` أو `ERR_PTR` — بيـcheck داخلياً

**Pseudocode flow:**
```
debugfs_remove(dentry):
    if dentry is NULL or ERR_PTR → return
    mark dentry as "being removed"
    wait for all active debugfs_file_get() references to drop
    remove from VFS tree (d_drop + unlink)
    for each child: repeat recursively
    dput(dentry)
```

---

#### `debugfs_lookup_and_remove`

```c
void debugfs_lookup_and_remove(const char *name, struct dentry *parent);
```

بيـlookup الـ entry بالاسم وبعدين بيحذفها في خطوة واحدة. أسهل من `debugfs_lookup` + manual `dput` + `debugfs_remove`.

**Parameters:**
- `name` — اسم الـ entry
- `parent` — الـ parent directory

**Return value:** void — لو مش موجود، بيـreturn بهدوء

---

#### `debugfs_change_name`

```c
int debugfs_change_name(struct dentry *dentry, const char *fmt, ...) __printf(2, 3);
```

بيـrename الـ dentry في الـ debugfs tree. بيقبل printf-style format string. مفيد لو الـ name بتتغير dynamically زي لما device تتـbind لـ driver تاني.

**Parameters:**
- `dentry` — الـ entry المراد تغيير اسمها
- `fmt` — printf format string
- `...` — arguments للـ format

**Return value:** `0` في النجاح، أو `-errno` في الفشل (زي `-ENOMEM` لو الـ allocation فشلت)

---

### Group 3: Scalar Value Files

الغرض من الـ group ده هو تـexpose قيم بسيطة (integers, bools, strings) من الـ kernel إلى userspace بأقل كود ممكن. كل function بتاخد pointer للمتغير وبتعمل file بيقرأ ويكتب فيه مباشرةً.

---

#### `debugfs_create_u8` / `u16` / `u32` / `u64` / `ulong`

```c
void debugfs_create_u8(const char *name, umode_t mode,
                        struct dentry *parent, u8 *value);
// نفس الـ pattern لـ u16, u32, u64, unsigned long
```

بيعمل ملف debugfs بيعرض القيمة decimal. القراءة والكتابة مباشرة على المتغير عن طريق `simple_attr`. مفيش locking داخلي — الـ caller مسؤول عن الـ thread safety لو المتغير بيتعدل من threads تانية.

**Parameters:**
- `value` — pointer للمتغير في memory الـ kernel

**Return value:** void — errors بتتجاهل

**Key details:** الكتابة من userspace بيحدث فيها `sscanf` على الـ input string. مفيش bounds checking على القيمة المكتوبة فوق limits النوع.

---

#### `debugfs_create_x8` / `x16` / `x32` / `x64`

```c
void debugfs_create_x8(const char *name, umode_t mode,
                        struct dentry *parent, u8 *value);
// نفس الـ pattern
```

نفس `debugfs_create_uX` بالظبط لكن بيعرض القيمة hexadecimal بـ prefix `0x`. مفيد لـ register values والـ addresses.

---

#### `debugfs_create_xul`

```c
static inline void debugfs_create_xul(const char *name, umode_t mode,
                                       struct dentry *parent,
                                       unsigned long *value)
{
    if (sizeof(*value) == sizeof(u32))
        debugfs_create_x32(name, mode, parent, (u32 *)value);
    else
        debugfs_create_x64(name, mode, parent, (u64 *)value);
}
```

**portable helper** لـ `unsigned long` بالـ hex. بيـcompile-time يختار بين `x32` و`x64` حسب الـ platform architecture. على 32-bit systems = `x32`، على 64-bit systems = `x64`.

**Key details:** الـ `sizeof` check بيتحسم في compile-time، مفيش runtime overhead.

---

#### `debugfs_create_size_t`

```c
void debugfs_create_size_t(const char *name, umode_t mode,
                            struct dentry *parent, size_t *value);
```

زي `debugfs_create_ulong` بس لـ `size_t` — مفيد لـ buffer sizes وعدد العناصر.

---

#### `debugfs_create_atomic_t`

```c
void debugfs_create_atomic_t(const char *name, umode_t mode,
                               struct dentry *parent, atomic_t *value);
```

بيعرض `atomic_t` variable كـ decimal. الـ read والـ write بيستخدموا `atomic_read()` و`atomic_set()` داخلياً — يعني الـ access نفسه atomic. مفيد لـ counters زي error counts وstats.

---

#### `debugfs_create_bool`

```c
void debugfs_create_bool(const char *name, umode_t mode,
                          struct dentry *parent, bool *value);
```

بيعرض `bool` كـ `Y` أو `N` في النص. الكتابة بتقبل `Y`, `y`, `1` لـ true و`N`, `n`, `0` لـ false. الـ read والـ write handlers هما `debugfs_read_file_bool` و`debugfs_write_file_bool`.

---

#### `debugfs_create_str`

```c
void debugfs_create_str(const char *name, umode_t mode,
                         struct dentry *parent, char **value);
```

بيعرض `char *` string. الـ `value` هو pointer to pointer — يعني لو الـ string اتغيرت (realloc)، الـ debugfs file هيعرض الـ string الجديدة تلقائياً. الـ read handler هو `debugfs_read_file_str`.

**Key details:** مفيش copy عند الإنشاء — الـ pointer ده live reference. لو الـ string اتفرجت من memory وفيه read جاري: race condition. الـ caller مسؤول عن الـ locking.

---

### Group 4: Complex Data Files

---

#### `debugfs_create_blob`

```c
struct dentry *debugfs_create_blob(const char *name, umode_t mode,
                                    struct dentry *parent,
                                    struct debugfs_blob_wrapper *blob);
```

بيعمل ملف binary بيعرض الـ blob محدد بـ `debugfs_blob_wrapper`. الـ read بيـcopy الـ data as-is لـ userspace بدون أي encoding.

```c
struct debugfs_blob_wrapper {
    void *data;       /* pointer to the raw data */
    unsigned long size; /* size in bytes */
};
```

**Parameters:**
- `blob` — pointer لـ wrapper struct. الـ wrapper بقى بيتخزن in the file's inode — **لازم يفضل valid طول عمر الـ file**.

**Return value:** `struct dentry *` أو `ERR_PTR`

**Key details:** الـ data مش بيتنسخ — pointer لـ data موجودة. لو الـ data اتفرجت قبل ما الـ file يتحذف: crash.

---

#### `debugfs_create_regset32`

```c
void debugfs_create_regset32(const char *name, umode_t mode,
                               struct dentry *parent,
                               struct debugfs_regset32 *regset);
```

بيعمل ملف بيعرض **set من الـ 32-bit MMIO registers** بشكل human-readable. عند القراءة، بيـloop على كل register، بيعمل `readl()` من `base + offset`، وبيطبع `name = value`.

```c
struct debugfs_regset32 {
    const struct debugfs_reg32 *regs; /* array of reg descriptors */
    int nregs;                         /* number of registers */
    void __iomem *base;                /* mapped base address */
    struct device *dev;                /* optional, for Runtime PM */
};

struct debugfs_reg32 {
    char *name;           /* human-readable register name */
    unsigned long offset; /* offset from base */
};
```

**Parameters:**
- `regset` — بيحتوي على كل المعلومات. لو `dev` مش NULL، بيـcall `pm_runtime_get_sync()` قبل القراءة عشان يصحى الـ device

**Return value:** void

**Key details:** الـ `dev` field مهمة جداً — من غيرها على بعض platforms الـ register read هيجيب 0xDEADBEEF لو الـ device نايم.

---

#### `debugfs_print_regs32`

```c
void debugfs_print_regs32(struct seq_file *s, const struct debugfs_reg32 *regs,
                           int nregs, void __iomem *base, char *prefix);
```

**helper function** تقدر تنادي عليها من داخل أي `seq_show` function. بيطبع الـ registers في الـ `seq_file` بـ optional prefix. مفيده لو بتبني custom seq_file وعايز تضم registers جوّاها.

**Parameters:**
- `s` — الـ seq_file المراد الكتابة فيه
- `regs` — array من `debugfs_reg32`
- `nregs` — عدد الـ registers في الـ array
- `base` — الـ base address المـmap
- `prefix` — optional prefix لكل سطر، أو `NULL`

**Return value:** void

---

#### `debugfs_create_u32_array`

```c
void debugfs_create_u32_array(const char *name, umode_t mode,
                               struct dentry *parent,
                               struct debugfs_u32_array *array);
```

بيعمل ملف بيعرض array من `u32` values — كل قيمة على سطر منفصل. مفيد لـ lookup tables وhistograms وfrequency tables.

```c
struct debugfs_u32_array {
    u32 *array;       /* pointer to the array */
    u32 n_elements;   /* number of elements */
};
```

**Parameters:**
- `array` — الـ wrapper struct. الـ array نفسها مش بتتنسخ — live pointer.

---

#### `debugfs_create_devm_seqfile`

```c
void debugfs_create_devm_seqfile(struct device *dev, const char *name,
                                  struct dentry *parent,
                                  int (*read_fn)(struct seq_file *s, void *data));
```

بيعمل seq_file مربوطة بـ **device resource management (devres)**. لما الـ `dev` يتـremove، الـ file بتتحذف تلقائياً. مفيش محتاج `debugfs_remove()` يدوي.

**Parameters:**
- `dev` — الـ device المربوط الـ file بـ lifetime بتاعه
- `read_fn` — الـ callback اللي بيـgenerate الـ content. بيتنادى مع `s` و`data = dev`

**Return value:** void

**Key details:** الـ `data` اللي بيتبعت للـ `read_fn` هو الـ `dev` pointer نفسه. الـ devm allocation بتضمن cleanup تلقائي عند `device_unregister()` أو `devm_*` cleanup.

---

### Group 5: File Access Protection

ده أهم group من ناحية correctness. المشكلة اللي بيحلها: إيه اللي بيحصل لو الـ module اتـunload وفيه `read()` أو `write()` system call شغالة على ملف debugfs بتاعه؟

الـ solution هو الـ **debugfs file protection mechanism**: كل file بـ `fops` عندها implicit srcu-based protection. الـ `debugfs_file_get/put` بيضمن إن الـ file handler مش هيتحذف وفيه access جاري.

---

#### `debugfs_file_get`

```c
int debugfs_file_get(struct dentry *dentry);
```

**بيـacquire SRCU read lock على الـ debugfs file**. بيمنع حذف الـ file وطول ما الـ lock محجوز. الـ VFS layer بيـcall ده تلقائياً عند open لو الملف اتعمل بـ `debugfs_create_file_full` (مش unsafe).

**Parameters:**
- `dentry` — الـ file dentry المراد الـ access عليها

**Return value:** `0` لو تمام، أو `-EIO` لو الـ file "being removed" — في الحالة دي ما تكملش.

**Key details:**
- بيستخدم **SRCU (Sleepable RCU)** مش spinlock — يعني آمن في sleepable context
- بيـcheck `d_inode` flag عشان يشوف لو الـ removal جاري
- **بعد ما تنادي `debugfs_file_get` ناجحة، لازم تنادي `debugfs_file_put`** بعدين

---

#### `debugfs_file_put`

```c
void debugfs_file_put(struct dentry *dentry);
```

**بيـrelease الـ SRCU read lock** اللي أخدته `debugfs_file_get`. بيـunblock أي `debugfs_remove()` كان بيـwait.

**Parameters:**
- `dentry` — نفس الـ dentry اللي بعتها لـ `debugfs_file_get`

**Return value:** void

**Key details:** لو نسيت تنادي `debugfs_file_put`، الـ `debugfs_remove()` هيـblock لأبد.

---

#### `debugfs_enter_cancellation`

```c
void debugfs_enter_cancellation(struct file *file,
                                 struct debugfs_cancellation *cancellation)
    __acquires(cancellation);
```

بيـregister **cancellation callback** يتنادى لو جه طلب حذف للـ file أثناء ما الـ read/write handler شغال. المثال الكلاسيكي: handler بيـwait على event — لو الـ file اتحذفت، نحتاج نـwakeup الـ handler عشان يرجع.

```c
/* مثال: في handler بيـwait */
static ssize_t my_read(struct file *file, ...) {
    DEFINE_DEBUGFS_CANCELLATION(cancel, my_cancel_fn, &my_waitqueue);
    debugfs_enter_cancellation(file, &cancel);
    wait_event(my_waitqueue, condition);
    debugfs_leave_cancellation(file, &cancel);
    ...
}

static void my_cancel_fn(struct dentry *dentry, void *data) {
    struct wait_queue_head *wq = data;
    wake_up(wq);
}
```

**Parameters:**
- `file` — الـ open file
- `cancellation` — struct فيها الـ callback وبياناته

**Key details:**
- `__acquires(cancellation)` — sparse annotation إن الـ lock semantics محتاجة
- بيضيف الـ cancellation لـ list جوّا الـ file's inode
- لو الـ removal بدأ قبل الـ `enter_cancellation`، الـ callback بيتنادى فوراً

---

#### `debugfs_leave_cancellation`

```c
void debugfs_leave_cancellation(struct file *file,
                                  struct debugfs_cancellation *cancellation)
    __releases(cancellation);
```

بيـunregister الـ cancellation handler اللي سجلته قبل كده. لازم يتنادى بعد الـ wait ينتهي سواء normally أو بـcancellation.

**Parameters:** نفس `debugfs_enter_cancellation`

---

### Group 6: Attribute Read/Write Helpers

الـ group ده بيوفر الـ standard read/write handlers اللي بيستخدمهم `DEFINE_DEBUGFS_ATTRIBUTE`.

---

#### `DEFINE_DEBUGFS_ATTRIBUTE` (macro)

```c
#define DEFINE_DEBUGFS_ATTRIBUTE(__fops, __get, __set, __fmt)       \
    DEFINE_DEBUGFS_ATTRIBUTE_XSIGNED(__fops, __get, __set, __fmt, false)

#define DEFINE_DEBUGFS_ATTRIBUTE_SIGNED(__fops, __get, __set, __fmt) \
    DEFINE_DEBUGFS_ATTRIBUTE_XSIGNED(__fops, __get, __set, __fmt, true)
```

**بيعمل `struct file_operations` كاملة** من getter/setter functions وformat string. الـ getter/setter بياخدوا ويرجعوا `u64` أو `s64`. الـ macro بيـgenerate `_open` function تلقائياً بيـcall `simple_attr_open`.

**مثال:**
```c
static int my_val_get(void *data, u64 *val) {
    *val = my_device->some_counter;
    return 0;
}
static int my_val_set(void *data, u64 val) {
    my_device->some_counter = val;
    return 0;
}
DEFINE_DEBUGFS_ATTRIBUTE(my_val_fops, my_val_get, my_val_set, "%llu\n");

/* في probe: */
debugfs_create_file("counter", 0644, dir, dev, &my_val_fops);
```

**الـ `__fmt`** هو printf format للـ display — `%llu\n` للـ unsigned، `%lld\n` للـ signed.

---

#### `debugfs_attr_read`

```c
ssize_t debugfs_attr_read(struct file *file, char __user *buf,
                           size_t len, loff_t *ppos);
```

**الـ read handler** لـ attribute files. بيـcall الـ getter، بيـformat القيمة بالـ `fmt`، وبيـcopy للـ userspace.

**Parameters:**
- `file` — الـ open file — بيجيب الـ simple_attr context منه
- `buf` / `len` / `ppos` — standard read parameters

**Return value:** عدد الـ bytes المقروءة أو `-errno`

**Key details:** بيعمل locking داخلياً على `simple_attr` mutex عشان يمنع concurrent access.

---

#### `debugfs_attr_write`

```c
ssize_t debugfs_attr_write(struct file *file, const char __user *buf,
                            size_t len, loff_t *ppos);
```

بيـparse الـ input من userspace كـ unsigned value وبيـcall الـ setter. الـ parsing بيستخدم `kstrtoull`.

---

#### `debugfs_attr_write_signed`

```c
ssize_t debugfs_attr_write_signed(struct file *file, const char __user *buf,
                                   size_t len, loff_t *ppos);
```

نفس `debugfs_attr_write` لكن بيـparse القيمة كـ signed باستخدام `kstrtoll`. يتستخدم مع `DEFINE_DEBUGFS_ATTRIBUTE_SIGNED`.

---

#### `debugfs_read_file_bool` / `debugfs_write_file_bool`

```c
ssize_t debugfs_read_file_bool(struct file *file, char __user *user_buf,
                                size_t count, loff_t *ppos);

ssize_t debugfs_write_file_bool(struct file *file, const char __user *user_buf,
                                 size_t count, loff_t *ppos);
```

الـ read/write handlers للملفات اللي اتعملت بـ `debugfs_create_bool`. الـ read بيرجع `"Y\n"` أو `"N\n"`. الـ write بيقبل `Y/y/1` أو `N/n/0`.

**Caller context:** بيتنادى من VFS read/write path — في process context.

---

#### `debugfs_read_file_str`

```c
ssize_t debugfs_read_file_str(struct file *file, char __user *user_buf,
                               size_t count, loff_t *ppos);
```

الـ read handler للملفات اللي اتعملت بـ `debugfs_create_str`. بيعمل `kstrdup` للـ string الحالية ثم بيـcopy لـ userspace.

**Key details:** الـ `kstrdup` بيحمي من race condition لو الـ string pointer اتغير أثناء الـ copy.

---

### Group 7: Auxiliary Data Helpers

---

#### `debugfs_get_aux`

```c
void *debugfs_get_aux(const struct file *file);
```

بيرجع الـ `aux` pointer اللي بعتله في `debugfs_create_file_aux`. ده مستقل عن `inode->i_private` اللي بيرجعه `file->private_data`.

**Parameters:**
- `file` — الـ open file

**Return value:** الـ `aux` pointer أو `NULL`

**Key details:** متاح حتى لو `CONFIG_DEBUG_FS` مش معمله enable — الـ non-debug implementation موجودة.

---

#### `debugfs_get_aux_num` / `debugfs_create_file_aux_num` (macros)

```c
#define debugfs_create_file_aux_num(name, mode, parent, data, n, fops) \
    debugfs_create_file_aux(name, mode, parent, data, \
                            (void *)(unsigned long)n, fops)

#define debugfs_get_aux_num(f) (unsigned long)debugfs_get_aux(f)
```

بيخزنوا ويجيبوا integer كـ aux pointer عن طريق casting. مفيد لو الـ aux مجرد index أو ID رقمي بدل pointer لـ struct.

---

### Group 8: State Queries

---

#### `debugfs_initialized`

```c
bool debugfs_initialized(void);
```

بيرجع `true` لو debugfs filesystem اتعمله mount و جاهز للاستخدام. بيرجع `false` في اللحظات الأولى من الـ boot قبل ما الـ VFS setup يكتمل.

**Return value:** `true` / `false`

**Key details:** في الكود اللي بيشتغل مبكر في الـ boot (زي core initcalls)، الـ check ده مهم قبل ما تحاول تعمل أي debugfs entries.

---

### ملاحظة على الـ Stub Functions (CONFIG_DEBUG_FS disabled)

لما `CONFIG_DEBUG_FS` مش موجود في الـ kernel config، كل الـ functions دي بترجع إما `ERR_PTR(-ENODEV)` أو void. ده بيخلي الـ driver code يشتغل بدون أي `#ifdef` حوالين كل debugfs call. الـ design الصح هو:

```c
/* لا تعمل كده */
#ifdef CONFIG_DEBUG_FS
debugfs_create_u32("counter", 0444, dir, &count);
#endif

/* عمل كده بدلاً منه */
debugfs_create_u32("counter", 0444, dir, &count); /* stub handles disabled case */
```

الاستثناء الوحيد: `debugfs_get_aux` متاحة حتى بدون `CONFIG_DEBUG_FS` — لأن بعض الـ code محتاجها في paths مشتركة.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. الـ debugfs — الـ Entries والـ API

الـ **debugfs** هو filesystem خاص بالـ debugging بيتعمل mount على `/sys/kernel/debug`. كل driver ممكن يعمل entries فيه عن طريق الـ API في `include/linux/debugfs.h`.

##### تأكد إن debugfs متعملهوش mount:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# أو تحقق
grep debugfs /proc/mounts
```

##### الـ Entries الأساسية الموجودة دايمًا:

```
/sys/kernel/debug/
├── tracing/          ← ftrace kernel tracing
├── sleep_time        ← sleep accounting
├── gpio/             ← GPIO states
├── clk/              ← clock tree (لو CONFIG_COMMON_CLK)
│   ├── clk_summary
│   ├── clk_dump
│   └── <clk_name>/
│       ├── clk_rate
│       ├── clk_enable_count
│       └── clk_prepare_count
├── regmap/           ← register maps
└── dri/              ← DRM/GPU
```

##### قراءة الـ Entries:

```bash
# قراءة قيمة u32/u64 مباشرة
cat /sys/kernel/debug/my_driver/some_counter

# قراءة register set (لو الـ driver استخدم debugfs_create_regset32)
cat /sys/kernel/debug/my_driver/registers

# قراءة clock tree كامل
cat /sys/kernel/debug/clk/clk_summary
```

##### كيفية إنشاء debugfs entry في الـ Driver:

```c
#include <linux/debugfs.h>

struct my_dev {
    struct dentry *dbg_dir;  /* debugfs directory */
    u32 error_count;
    u32 tx_packets;
};

static int my_driver_probe(struct platform_device *pdev)
{
    struct my_dev *dev = ...;

    /* إنشاء directory خاص بالـ driver */
    dev->dbg_dir = debugfs_create_dir("my_driver", NULL);

    /* عرض عداد كـ decimal */
    debugfs_create_u32("error_count", 0444,
                       dev->dbg_dir, &dev->error_count);

    /* عرض قيمة كـ hex */
    debugfs_create_x32("tx_packets", 0444,
                       dev->dbg_dir, &dev->tx_packets);
    return 0;
}

static void my_driver_remove(struct platform_device *pdev)
{
    struct my_dev *dev = ...;
    debugfs_remove(dev->dbg_dir);  /* يحذف الـ dir وكل محتوياته */
}
```

##### الـ `debugfs_create_regset32` لعرض الـ Registers:

```c
static const struct debugfs_reg32 my_regs[] = {
    { "CTRL",    0x00 },
    { "STATUS",  0x04 },
    { "INT_EN",  0x08 },
    { "INT_CLR", 0x0C },
};

static struct debugfs_regset32 my_regset = {
    .regs    = my_regs,
    .nregs   = ARRAY_SIZE(my_regs),
    .base    = NULL,  /* يتعبى وقت الـ probe */
    .dev     = NULL,  /* للـ Runtime PM */
};

/* في الـ probe */
my_regset.base = dev->base_addr;
my_regset.dev  = &pdev->dev;
debugfs_create_regset32("registers", 0444, dev->dbg_dir, &my_regset);
```

```bash
# قراءة الـ registers بعد إنشاء الـ entry
cat /sys/kernel/debug/my_driver/registers
# Output:
# CTRL     = 0x00000001
# STATUS   = 0x00000003
# INT_EN   = 0x0000000F
# INT_CLR  = 0x00000000
```

##### الـ `DEFINE_DEBUGFS_ATTRIBUTE` للـ Read/Write:

```c
static int my_val_get(void *data, u64 *val)
{
    struct my_dev *dev = data;
    *val = dev->some_value;
    return 0;
}

static int my_val_set(void *data, u64 val)
{
    struct my_dev *dev = data;
    dev->some_value = val;
    return 0;
}

/* تعريف الـ fops بشكل آمن */
DEFINE_DEBUGFS_ATTRIBUTE(my_val_fops, my_val_get, my_val_set, "%llu\n");

/* في الـ probe */
debugfs_create_file("my_value", 0644, dev->dbg_dir, dev, &my_val_fops);
```

```bash
# قراءة
cat /sys/kernel/debug/my_driver/my_value
# كتابة
echo 42 > /sys/kernel/debug/my_driver/my_value
```

---

#### 2. الـ sysfs — الـ Entries المهمة

الـ **sysfs** على `/sys` مختلف عن debugfs: هو للـ interface الرسمي بين الـ user-space والـ kernel.

```
/sys/
├── bus/platform/devices/<dev>/   ← platform devices
├── class/<class>/<dev>/          ← class devices
├── devices/                      ← device tree
│   └── platform/<dev>/
│       ├── uevent
│       ├── driver/               ← symlink للـ driver
│       ├── power/                ← runtime PM
│       │   ├── runtime_status    ← "active" / "suspended"
│       │   ├── runtime_enabled   ← "enabled" / "disabled"
│       │   └── autosuspend_delay_ms
│       └── regulator*/           ← لو الـ device بيستخدم regulator
└── kernel/debug/ → /sys/kernel/debug  ← debugfs mount point
```

```bash
# تحقق من حالة الـ device
cat /sys/devices/platform/my_device/power/runtime_status

# كل الـ attributes الخاصة بالـ device
ls -la /sys/bus/platform/devices/my_device/

# قراءة الـ driver المرتبط
ls -la /sys/bus/platform/devices/my_device/driver
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ **ftrace** هو أقوى أداة tracing في الـ kernel. بيعمل على `/sys/kernel/debug/tracing/`.

##### تفعيل الـ function tracing للـ debugfs subsystem:

```bash
cd /sys/kernel/debug/tracing

# تتبع كل functions اللي بدأت بـ debugfs_
echo 'debugfs_*' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# شغّل الـ operation اللي بتحاول تـ debug فيها
cat /sys/kernel/debug/my_driver/some_file

echo 0 > tracing_on
cat trace
```

##### تتبع الـ events المهمة للـ VFS/debugfs:

```bash
# شوف كل الـ events المتاحة
ls /sys/kernel/debug/tracing/events/

# تفعيل events الـ file system
echo 1 > /sys/kernel/debug/tracing/events/ext4/enable   # لو بتـ debug VFS
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_open/enable
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_read/enable

# تفعيل trace الـ kmalloc لو بتشك في memory allocation
echo 1 > /sys/kernel/debug/tracing/events/kmem/kmalloc/enable

# قراءة الـ trace
cat /sys/kernel/debug/tracing/trace
```

##### تتبع الـ function graph (أحسن لفهم الـ call stack):

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'debugfs_create_file*' > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... افعل الـ operation ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

##### مثال على output الـ function_graph:

```
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)   1.234 us    |  debugfs_create_file_full() {
 0)   0.456 us    |    debugfs_create_file_unsafe();
 0)   0.123 us    |    simple_attr_open();
 0)   1.234 us    |  }
```

---

#### 4. الـ printk والـ Dynamic Debug

##### تفعيل الـ **dynamic debug** لـ subsystem معين:

```bash
# تفعيل كل الـ pr_debug في ملف معين
echo 'file drivers/my_driver/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ pr_debug في module معين
echo 'module my_module +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل وعرض الـ function name والـ line number
echo 'module my_module +pfl' > /sys/kernel/debug/dynamic_debug/control
# +p = print, +f = function name, +l = line number, +m = module name, +t = thread ID

# عرض الـ debug entries المفعّلة
cat /sys/kernel/debug/dynamic_debug/control | grep '=p'
```

##### تغيير مستوى الـ printk:

```bash
# اعرض مستوى الـ console الحالي
cat /proc/sys/kernel/printk
# 7  4  1  7
# current | default | minimum | boot-time-default

# خلي كل الـ messages تظهر على الـ console
echo 8 > /proc/sys/kernel/printk

# أو بـ sysctl
sysctl kernel.printk="8 4 1 7"
```

##### في الـ driver code — مستويات الـ printk:

```c
/* استخدام dev_* macros (أفضل من pr_* لأنه بيوضح الـ device) */
dev_err(&pdev->dev,   "critical error: %d\n", ret);    /* KERN_ERR */
dev_warn(&pdev->dev,  "unexpected state\n");             /* KERN_WARNING */
dev_info(&pdev->dev,  "initialized OK\n");               /* KERN_INFO */
dev_dbg(&pdev->dev,   "entering function %s\n", __func__); /* KERN_DEBUG, dynamic */

/* استخدام pr_* لو مفيش device */
pr_debug("debugfs: creating entry %s\n", name);  /* dynamic debug */
pr_err("debugfs: failed to create %s: %d\n", name, ret);
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_FS` | تفعيل الـ debugfs filesystem (أساسي) |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل الـ dynamic debug (pr_debug, dev_dbg) |
| `CONFIG_FTRACE` | تفعيل الـ ftrace framework |
| `CONFIG_FUNCTION_TRACER` | تتبع الـ kernel functions |
| `CONFIG_FUNCTION_GRAPH_TRACER` | تتبع الـ call graph |
| `CONFIG_KPROBES` | dynamic kernel instrumentation |
| `CONFIG_KASAN` | Kernel Address Sanitizer — يكتشف memory bugs |
| `CONFIG_SLUB_DEBUG` | debug الـ SLUB allocator |
| `CONFIG_DEBUG_KERNEL` | تفعيل مجموعة debug options |
| `CONFIG_DEBUG_INFO` | إضافة debug symbols لـ vmlinux |
| `CONFIG_LOCKDEP` | كشف الـ deadlocks |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | كشف النوم داخل atomic context |
| `CONFIG_PROVE_LOCKING` | يتحقق من صحة استخدام الـ locks |
| `CONFIG_UBSAN` | كشف Undefined Behavior |
| `CONFIG_KCOV` | code coverage للـ fuzzing |
| `CONFIG_DEBUG_OBJECTS` | tracking حياة الـ kernel objects |
| `CONFIG_DEBUG_SPINLOCK` | تتبع مشاكل الـ spinlocks |
| `CONFIG_STACKTRACE` | تفعيل الـ stack trace support |

##### تفعيل أسرع في الـ .config:

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep CONFIG_DEBUG_FS
# أو
grep CONFIG_DEBUG_FS /boot/config-$(uname -r)
```

---

#### 6. أدوات الـ Subsystem المتخصصة

##### الـ `debugfs_initialized()` — التحقق قبل الاستخدام:

```c
/* في الـ driver init */
if (!debugfs_initialized()) {
    pr_warn("debugfs not available, skipping debug entries\n");
    return;
}
```

##### الـ `debugfs_file_get/put` — حماية من الـ race conditions:

```c
/* في الـ fops->read أو fops->write */
int err = debugfs_file_get(dentry);
if (err)
    return err;

/* ... safe to access driver data ... */

debugfs_file_put(dentry);
```

##### الـ `debugfs_cancellation` — إلغاء العمليات الطويلة:

```c
/* مفيد في الـ operations اللي بتـ block لفترة طويلة */
struct debugfs_cancellation cancel;

debugfs_enter_cancellation(file, &cancel);
/* ... blocking operation ... */
debugfs_leave_cancellation(file, &cancel);
```

##### أدوات مكملة:

```bash
# perf — profiling
perf stat -e cache-misses,cache-references ./my_app

# strace — تتبع الـ syscalls من الـ user-space
strace -e openat,read,write cat /sys/kernel/debug/my_driver/regs

# inotifywait — مراقبة تغييرات الـ debugfs
inotifywait -m /sys/kernel/debug/my_driver/

# devmem2 — قراءة/كتابة الـ physical memory مباشرة
devmem2 0xFE200000 w   # قراءة 32-bit word

# /dev/mem — raw memory access
dd if=/dev/mem bs=4 count=1 skip=$((0xFE200000 / 4)) 2>/dev/null | xxd
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `debugfs: Directory 'X' with parent 'Y' already present!` | محاولة إنشاء entry موجود | تحقق إن الـ remove بيتعمل في الـ exit path |
| `debugfs: failed to create file 'X'` | فشل الإنشاء، غالبًا الـ parent NULL أو error | تحقق من الـ parent dentry، لا تـ check الـ return value |
| `-ENODEV` from debugfs API | `CONFIG_DEBUG_FS` مش مفعّل | فعّل الـ config أو تجاهل الـ error (مقصود) |
| `WARN_ON: dentry is not a directory` | الـ parent مش directory | استخدم `debugfs_create_dir` أولًا |
| `BUG: unable to handle kernel NULL pointer` عند قراءة debugfs entry | الـ driver data اتحذف قبل الـ debugfs entry | استخدم `debugfs_file_get/put` أو تأكد من ترتيب الـ cleanup |
| `seq_file: buffer overflow` | الـ seq_file buffer امتلأ | الـ kernel بيعمل re-alloc تلقائيًا، مش مشكلة عادةً |
| `WARNING: CPU: X PID: Y at fs/debugfs/file.c:ZZZ` | محاولة كتابة لـ read-only entry | تحقق من الـ mode في `debugfs_create_*` |
| `debugfs_remove: dentry is an error value` | إمرار ERR_PTR للـ remove | تحقق من الـ dentry قبل الـ remove: `if (!IS_ERR(dentry))` |

---

#### 8. نقاط استخدام `dump_stack()` و`WARN_ON()`

##### متى تستخدم `WARN_ON()`:

```c
/* للتحقق من الـ preconditions في بداية الـ functions */
int debugfs_do_something(struct dentry *parent, void *data)
{
    /* لازم الـ parent يكون directory */
    if (WARN_ON(!parent || IS_ERR(parent)))
        return -EINVAL;

    /* data مش المفروض يكون NULL هنا */
    if (WARN_ON_ONCE(!data))
        return -EINVAL;

    return 0;
}
```

##### متى تستخدم `dump_stack()`:

```c
/* لما تعوز تعرف مين استدعى الـ function دي */
void my_debugfs_callback(struct dentry *dentry, void *data)
{
    if (unexpected_condition) {
        pr_err("my_driver: unexpected state, dumping stack\n");
        dump_stack();  /* بيطبع الـ call stack في الـ kernel log */
    }
}
```

##### النقاط الاستراتيجية في الـ debugfs code:

```c
/* 1. عند فشل إنشاء الـ directory */
dev->dbg_dir = debugfs_create_dir(name, parent);
if (IS_ERR(dev->dbg_dir)) {
    /* لا تعمل WARN_ON هنا — debugfs failure مش fatal */
    dev->dbg_dir = NULL;
    return;
}

/* 2. عند الـ remove في الـ wrong order */
void my_driver_remove(struct platform_device *pdev)
{
    WARN_ON(atomic_read(&dev->open_count) > 0);  /* files مفتوحة؟ */
    debugfs_remove(dev->dbg_dir);  /* يحذف كل الـ entries تلقائيًا */
    dev->dbg_dir = NULL;
}

/* 3. في الـ read handler لو الـ state غير متوقع */
static int my_show(struct seq_file *m, void *data)
{
    struct my_dev *dev = m->private;
    if (WARN_ON(!dev->hw_initialized)) {
        seq_puts(m, "hardware not initialized\n");
        return 0;
    }
    seq_printf(m, "reg: 0x%08x\n", readl(dev->base));
    return 0;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيتطابق مع الـ Kernel State

##### مقارنة الـ register values:

```bash
# الـ kernel شايف الـ clock enabled؟
cat /sys/kernel/debug/clk/my_clk/clk_enable_count
# 1

# قارن بالـ hardware register مباشرة
devmem2 0xFE200004 w   # قراءة الـ clock control register
# Value at address 0xFE200004 (0xb8961004): 0x00000001  ← bit 0 = enabled

# الـ kernel شايف الـ GPIO كـ output؟
cat /sys/class/gpio/gpio17/direction  # "out"
# قارن بالـ hardware
devmem2 0x3F200000 w   # BCM2835 GPFSEL0 register
```

##### استخدام `/sys/kernel/debug/regmap/`:

```bash
# لو الـ driver بيستخدم regmap
ls /sys/kernel/debug/regmap/
# my-device-0x48/

cat /sys/kernel/debug/regmap/my-device-0x48/registers
# 00: 01
# 01: 83
# 02: 40
```

---

#### 2. الـ Register Dump Techniques

##### `devmem2` — الأسهل:

```bash
# تثبيت devmem2
apt-get install devmem2
# أو
yum install devmem2

# قراءة register
devmem2 0xFE200000 w        # 32-bit word read
devmem2 0xFE200000 h        # 16-bit halfword
devmem2 0xFE200000 b        # 8-bit byte

# كتابة register (خطر — تأكد من الـ register map)
devmem2 0xFE200000 w 0x01   # كتابة 0x01
```

##### `/dev/mem` — dump نطاق من الـ registers:

```bash
# قراءة 256 bytes من عنوان 0xFE200000
dd if=/dev/mem of=/tmp/regs.bin bs=256 count=1 skip=$((0xFE200000 / 256)) 2>/dev/null
xxd /tmp/regs.bin

# طريقة بديلة بـ python
python3 -c "
import mmap, os, struct
f = open('/dev/mem', 'rb')
m = mmap.mmap(f.fileno(), 256, offset=0xFE200000)
for i in range(0, 256, 4):
    val = struct.unpack('<I', m[i:i+4])[0]
    print(f'0x{0xFE200000+i:08X}: 0x{val:08X}')
m.close()
f.close()
"
```

##### `io` command — من package `elinks`:

```bash
# قراءة I/O port (x86 فقط)
io -4 -r 0x60   # قراءة 32-bit من I/O port 0x60
io -4 -w 0x60 0xFF  # كتابة
```

##### `debugfs_print_regs32` في الـ driver code:

```c
/* بدل ما تكتب seq_printf واحدة واحدة */
static int my_regs_show(struct seq_file *s, void *data)
{
    struct my_dev *dev = s->private;

    /* طباعة جميلة لكل الـ registers */
    debugfs_print_regs32(s, my_regs, ARRAY_SIZE(my_regs),
                         dev->base, "my_device");
    return 0;
}
/* Output:
 * my_device CTRL     = 0x00000001
 * my_device STATUS   = 0x00000003
 */
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

##### للـ I2C debugging:

```
Channel 1 → SDA
Channel 2 → SCL
Trigger: falling edge على SCL

نقاط التحقق:
- الـ pull-up resistors موجودين؟ (بتشوف high level صح؟)
- الـ clock speed متطابق مع الـ DT property؟
- الـ ACK bits صح؟ (low بعد كل byte)
- الـ START condition: SDA ينزل وهو SCL عالي
- الـ STOP condition: SDA يطلع وهو SCL عالي
```

##### للـ SPI debugging:

```
Channel 1 → SCLK
Channel 2 → MOSI
Channel 3 → MISO
Channel 4 → CS (Chip Select)

نقاط التحقق:
- الـ CS ينزل قبل الـ clock؟
- الـ clock polarity (CPOL) متطابق مع الـ device؟
- الـ clock phase (CPHA) صح؟
- الـ bit order (MSB first أو LSB first)؟
```

##### للـ UART debugging:

```bash
# تحقق من الـ baud rate في الـ kernel
cat /sys/class/tty/ttyS0/uartclk

# شوف الـ UART settings
stty -F /dev/ttyS0 -a
```

---

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | الـ Pattern في الـ Kernel Log |
|---|---|
| الـ I2C device مش موجود | `i2c i2c-0: i2c_master_send: failed -6 (ENXIO)` |
| الـ SPI transfer فاشل | `spi_master spi0: transfer failed -110 (ETIMEDOUT)` |
| الـ IRQ مش بيـ fire | لا يوجد interrupt messages + الـ driver متعلق في wait |
| الـ DMA transfer فاشل | `dma_request_chan: No channel of type 0x1 found` |
| الـ clock مش بيشتغل | `clk: Disabling unused clocks` + الـ device مش بيـ respond |
| الـ power domain مش متفعّل | `pm_runtime_get_sync failed: -13 (EACCES)` |
| الـ register access على عنوان غلط | `Unhandled fault: external abort on non-linefetch` |
| الـ IOMMU fault | `arm-smmu: Unhandled context fault` |
| الـ PCIe link down | `pcieport: PCIe Bus Error: severity=Uncorrected (Fatal)` |
| الـ hardware timeout | `watchdog: BUG: soft lockup - CPU#X stuck` |

---

#### 5. الـ Device Tree Debugging

##### التحقق إن الـ DT متحمّل صح:

```bash
# شوف الـ DT المحمّل فعليًا
ls /sys/firmware/devicetree/base/

# قراءة property معينة
cat /sys/firmware/devicetree/base/soc/i2c@7e804000/clock-frequency | xxd
# 00000000: 0001 86a0     ← 100000 = 100KHz

# تحويل binary DT لـ human-readable
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "my_device"

# أو باستخدام fdtdump (لو عندك الـ dtb file)
fdtdump /boot/my_board.dtb | grep -A10 "my_device"
```

##### التحقق من الـ compatible string:

```bash
# الـ device node موجود في الـ DT؟
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "my,my-device" {} \;

# شوف الـ compatible string
cat /sys/firmware/devicetree/base/soc/my_device/compatible
# my,my-device
```

##### مقارنة الـ DT مع الـ hardware:

```bash
# الـ reg property في الـ DT (الـ base address)
cat /sys/firmware/devicetree/base/soc/my_device/reg | xxd
# 00000000: 0000 0000 fe20 0000 0000 0000 0000 00ff
#           ↑ high 32-bit  ↑ address      ↑ size

# تأكد إن الـ address صح في الـ iomem
cat /proc/iomem | grep my_device
# fe200000-fe2000ff : my_device@fe200000

# الـ interrupts في الـ DT
cat /sys/firmware/devicetree/base/soc/my_device/interrupts | xxd
```

##### الـ device مش بيتـ probe؟

```bash
# شوف الـ deferred probes
cat /sys/kernel/debug/devices_deferred

# شوف الـ driver binding
ls /sys/bus/platform/drivers/my_driver/

# فحص الـ module loaded
lsmod | grep my_driver

# سبب الـ probe failure
dmesg | grep -i "my_device\|my_driver" | tail -20
```

---

### Practical Commands

#### الـ Commands الجاهزة للـ Copy

##### 1. إعداد الـ debugging environment بالكامل:

```bash
#!/bin/bash
# setup-debug.sh — إعداد بيئة الـ debugging

# mount debugfs
mount -t debugfs none /sys/kernel/debug 2>/dev/null || true

# رفع مستوى الـ printk لـ max
echo 8 > /proc/sys/kernel/printk

# تفعيل dynamic debug للـ module
echo "module my_driver +pflmt" > /sys/kernel/debug/dynamic_debug/control

# تفعيل ftrace
echo nop > /sys/kernel/debug/tracing/current_tracer
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace   # clear

echo "Debug environment ready"
```

##### 2. تتبع الـ debugfs operations:

```bash
#!/bin/bash
# trace-debugfs.sh

TRACE_DIR=/sys/kernel/debug/tracing

echo function > $TRACE_DIR/current_tracer
echo 'debugfs_*' > $TRACE_DIR/set_ftrace_filter
echo stacktrace > $TRACE_DIR/trace_options   # يطبع الـ stack مع كل call
echo 1 > $TRACE_DIR/tracing_on

sleep 5  # أو شغّل الـ operation اللي بتـ debug فيها

echo 0 > $TRACE_DIR/tracing_on
cat $TRACE_DIR/trace > /tmp/debugfs-trace.txt
echo "Trace saved to /tmp/debugfs-trace.txt"
```

##### 3. dump كل الـ debugfs entries لـ driver معين:

```bash
#!/bin/bash
# dump-driver-debug.sh <driver_name>

DRIVER=$1
DBG_PATH=/sys/kernel/debug/$DRIVER

if [ ! -d "$DBG_PATH" ]; then
    echo "Error: $DBG_PATH not found"
    exit 1
fi

echo "=== Dumping debugfs for $DRIVER ==="
find $DBG_PATH -type f | while read f; do
    echo -e "\n--- $f ---"
    timeout 2 cat "$f" 2>&1 || echo "[read timeout/error]"
done
```

##### 4. مراقبة الـ kernel log بشكل مباشر:

```bash
# مراقبة مستمرة مع highlight
dmesg -w | grep --color=always -iE "error|warn|fail|my_driver"

# أو باستخدام journalctl
journalctl -k -f | grep --color=always "my_driver"

# حفظ الـ log أثناء الـ test
dmesg -c > /dev/null  # clear الـ log
./run_my_test.sh
dmesg > /tmp/test-log.txt
```

##### 5. تحقق من حالة الـ registers عبر regset:

```bash
# لو الـ driver بيستخدم debugfs_create_regset32
watch -n 1 'cat /sys/kernel/debug/my_driver/registers'
# بيعمل refresh كل ثانية — مفيد لمراقبة الـ status registers
```

##### 6. استخدام `kprobes` من الـ command line:

```bash
# تثبيت probe على debugfs_create_file_full
echo 'p:my_probe debugfs_create_file_full name=+0(%di):string' > \
    /sys/kernel/debug/tracing/kprobe_events

echo 1 > /sys/kernel/debug/tracing/events/kprobes/my_probe/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ operation
insmod my_driver.ko

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
# Output:
# insmod-1234 [000] .... debugfs_create_file_full: name="my_entry"

# cleanup
echo '-:my_probe' >> /sys/kernel/debug/tracing/kprobe_events
```

##### 7. قراءة وتحليل الـ trace output:

```bash
# مثال على trace output وتفسيره
cat /sys/kernel/debug/tracing/trace
```

```
# tracer: function
#
# TASK-PID   CPU#  IRQS-off  NEED-RESCHED  HARDIRQ/SOFTIRQ  PREEMPT-DEPTH  DELAY
#   |          |      |           |               |                |          |
  bash-1234  [001]  ....  300.123456: debugfs_create_dir <-my_driver_probe
  bash-1234  [001]  ....  300.123457: debugfs_create_u32 <-my_driver_probe
  bash-1234  [001]  ....  300.123458: debugfs_create_x32 <-my_driver_probe
  bash-1234  [001]  ....  300.456789: debugfs_attr_read <-vfs_read
#  ↑ process  ↑cpu  ↑flags    ↑timestamp ↑function       ↑caller
```

##### 8. فحص سريع لحالة الـ debugfs subsystem:

```bash
#!/bin/bash
# check-debugfs-health.sh

echo "=== debugfs Health Check ==="

# 1. هل debugfs متعمله mount؟
if grep -q debugfs /proc/mounts; then
    echo "[OK] debugfs is mounted"
    grep debugfs /proc/mounts
else
    echo "[FAIL] debugfs not mounted"
fi

# 2. هل CONFIG_DEBUG_FS مفعّل؟
if [ -f /proc/config.gz ]; then
    if zcat /proc/config.gz | grep -q "CONFIG_DEBUG_FS=y"; then
        echo "[OK] CONFIG_DEBUG_FS=y"
    else
        echo "[FAIL] CONFIG_DEBUG_FS not enabled"
    fi
fi

# 3. قائمة الـ entries الموجودة
echo ""
echo "=== Top-level debugfs entries ==="
ls /sys/kernel/debug/ 2>/dev/null | head -20

echo ""
echo "=== Dynamic debug status ==="
grep "=p" /sys/kernel/debug/dynamic_debug/control 2>/dev/null | wc -l
echo "active dynamic debug entries"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — تشخيص تسرب في UART DMA

#### العنوان
**تسرب بيانات UART صامت في gateway صناعي — debugfs كشف المشكلة**

#### السياق
بورد RK3562 مستخدم في industrial gateway بيجمع بيانات من حساسات Modbus RTU عبر UART2. المنتج شغال في مصنع، والـ gateway بيبعت البيانات لـ SCADA server. العميل بيلاحظ إن البيانات بتوصل ناقصة كل ساعتين تقريباً بدون أي error في الـ logs.

#### المشكلة
الـ UART DMA driver بيعمل buffer flush غلط — بيكتب البيانات لـ DMA buffer لكن الـ `bytes_pending` counter مش بيتعدل صح. المشكلة مش واضحة من الـ kernel logs لأن لا panic لا warning.

#### التحليل
الـ driver بيستخدم `debugfs_create_u32()` لـ expose الـ `bytes_pending` counter:

```c
/* في uart_dma_probe() */
struct dentry *dir;
struct uart_dma_priv *priv = dev_get_drvdata(dev);

dir = debugfs_create_dir("uart2_dma", NULL);

/* expose الـ counter مباشرة من الـ struct */
debugfs_create_u32("bytes_pending", 0444, dir, &priv->bytes_pending);
debugfs_create_u32("dma_errors",    0444, dir, &priv->dma_errors);
debugfs_create_u32("flush_count",   0444, dir, &priv->flush_count);
```

الـ `debugfs_create_u32()` بتاخد pointer مباشر للـ variable — أي تغيير في الـ runtime بيظهر فوراً لما تقرأ الملف. المهندس راح يقرأ القيم كل 100ms:

```bash
# على البورد مباشرة
watch -n 0.1 'cat /sys/kernel/debug/uart2_dma/bytes_pending \
              /sys/kernel/debug/uart2_dma/flush_count'
```

لقى إن `bytes_pending` بيوصل لـ 4096 (حجم الـ buffer كامل) لكن `flush_count` مش بيزيد — يعني الـ flush مش بيتعمل لما المفروض يتعمل.

#### الحل
المشكلة كانت في الـ threshold check:

```c
/* كود قديم — غلط */
if (priv->bytes_pending > 4096)  /* لا يحدث أبداً لأن الـ buffer نفسه 4096 */
    uart_dma_flush(priv);

/* كود جديد — صح */
if (priv->bytes_pending >= (priv->buf_size * 3 / 4))  /* flush عند 75% */
    uart_dma_flush(priv);
```

```bash
# بعد الـ fix، التحقق
echo 0 > /sys/kernel/debug/uart2_dma/dma_errors
# ثم مراقبة إن الـ flush_count بيزيد بانتظام
```

#### الدرس المستفاد
الـ `debugfs_create_u32()` بتعرض الـ live value مباشرة بدون أي overhead — مثالية لـ counters بيتغيروا كتير. في الـ industrial context، اعمل دايماً debug interface للـ counters الداخلية قبل ما تشحن المنتج.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تشخيص مشكلة HDMI EDID

#### العنوان
**HDMI مش بيطلع صورة على شاشات معينة — debugfs كشف الـ EDID parsing bug**

#### السياق
TV box بيستخدم Allwinner H616، البورد شغالة مع 90% من الشاشات لكن عند بعض شاشات Samsung 4K بتيجي شاشة سودا. الـ Android UI بيشتغل (الـ adb شغال) لكن لا صورة.

#### المشكلة
الـ HDMI driver بيقرأ الـ EDID من الشاشة لكن بيفشل في parse الـ extension blocks. الـ driver مش بيـ fallback للـ safe mode وبيفضل يحاول الـ unsupported mode.

#### التحليل
الـ HDMI driver عنده debugfs interface بيستخدم `debugfs_create_blob()` لعرض raw EDID data، و`debugfs_create_regset32()` لعرض الـ register state:

```c
/* في hdmi_probe() */
struct debugfs_blob_wrapper *edid_blob;
struct debugfs_regset32 *regset;

priv->debugfs_dir = debugfs_create_dir("hdmi-h616", NULL);

/* عرض الـ raw EDID كـ blob */
edid_blob = devm_kzalloc(dev, sizeof(*edid_blob), GFP_KERNEL);
edid_blob->data = priv->edid_buf;   /* 256 bytes */
edid_blob->size = 256;
debugfs_create_blob("edid_raw", 0444, priv->debugfs_dir, edid_blob);

/* عرض الـ HDMI controller registers */
static const struct debugfs_reg32 hdmi_regs[] = {
    { "PHY_CTRL",    0x000 },
    { "VIDEO_CTRL",  0x004 },
    { "EDID_STATUS", 0x100 },
};
regset->regs  = hdmi_regs;
regset->nregs = ARRAY_SIZE(hdmi_regs);
regset->base  = priv->regs;
debugfs_create_regset32("registers", 0444, priv->debugfs_dir, regset);
```

المهندس قرأ الـ EDID raw:

```bash
# قراءة الـ EDID الخام
xxd /sys/kernel/debug/hdmi-h616/edid_raw | head -20

# قراءة الـ registers
cat /sys/kernel/debug/hdmi-h616/registers
```

لقى إن `EDID_STATUS` register بيقول `0x3` يعني "extension block read error" — الـ driver بيقرأ الـ base EDID بس فشل في الـ extension block اللي فيه الـ 4K modes.

#### الحل
```c
/* في hdmi_read_edid() */
ret = hdmi_read_edid_block(priv, 0, priv->edid_buf);  /* base block */
if (ret)
    goto use_safe_mode;

/* قراءة الـ extension blocks */
ext_count = priv->edid_buf[126];
for (i = 1; i <= ext_count; i++) {
    ret = hdmi_read_edid_block(priv, i, priv->edid_buf + i * 128);
    if (ret) {
        dev_warn(dev, "extension block %d failed, using base only\n", i);
        break;  /* كان هنا: goto fail — اتغير لـ break */
    }
}
```

```bash
# التحقق بعد الـ fix
cat /sys/kernel/debug/hdmi-h616/registers | grep EDID_STATUS
# يجب أن يكون 0x1 (base only) أو 0x0 (all blocks OK)
```

#### الدرس المستفاد
**الـ `debugfs_create_blob()`** مثالية لـ expose binary data زي EDID، firmware images، أو calibration tables. دايماً اعرض الـ raw data مش بس الـ parsed result — الـ bug ممكن يكون في الـ parsing نفسه.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — تشخيص مشكلة SPI clock

#### العنوان
**SPI sensor بيقرأ قيم غلط على STM32MP1 — debugfs كشف mismatch في الـ clock divider**

#### السياق
STM32MP1-based IoT node بيقيس ضغط الهواء عبر SPI pressure sensor (MS5611). الـ sensor المفروض يشتغل على 20 MHz لكن البيانات اللي بتيجي corrupted. البورد في early bring-up مرحلة.

#### المشكلة
الـ SPI controller driver بيحسب الـ clock divider غلط لما الـ parent clock مش على القيمة الافتراضية. الـ actual SPI clock بيطلع 25 MHz بدل 20 MHz.

#### التحليل
في مرحلة الـ bring-up، المهندس أضاف `DEFINE_DEBUGFS_ATTRIBUTE` لـ expose الـ actual clock frequency:

```c
/* في spi-stm32.c */
static int stm32_spi_clock_get(void *data, u64 *val)
{
    struct stm32_spi *spi = data;
    /* احسب الـ actual clock من الـ divider الحالي */
    u32 div = readl_relaxed(spi->base + STM32_SPI_CFG1);
    div = FIELD_GET(SPI_CFG1_MBR_MASK, div);
    *val = spi->clk_rate / (1 << (div + 1));  /* المعادلة من datasheet */
    return 0;
}

DEFINE_DEBUGFS_ATTRIBUTE(stm32_spi_clock_fops,
                         stm32_spi_clock_get, NULL, "%llu\n");

/* في probe */
priv->debugfs = debugfs_create_dir("spi-stm32.0", NULL);
debugfs_create_file("actual_clock_hz", 0444, priv->debugfs,
                    priv, &stm32_spi_clock_fops);
debugfs_create_u32("target_clock_hz", 0444, priv->debugfs,
                   &priv->cur_speed_hz);
```

القراءة:

```bash
cat /sys/kernel/debug/spi-stm32.0/actual_clock_hz
# طلع: 25000000  ← المفروض 20000000!

cat /sys/kernel/debug/spi-stm32.0/target_clock_hz
# طلع: 20000000  ← الـ target صح
```

الـ `DEFINE_DEBUGFS_ATTRIBUTE` macro بيعمل `file_operations` كاملة بـ getter function — بيعدي الـ `priv` pointer عبر `inode->i_private` وبيتحقق من الـ format تلقائياً في compile time عبر `__simple_attr_check_format`. المهندس اكتشف إن الـ clock divider formula غلط:

```c
/* غلط: يعطي divider صغير بحيث الـ clock أعلى من المطلوب */
div = ilog2(spi->clk_rate / speed_hz) - 1;

/* صح: ceiling division ضمان إن الـ clock مش يتعدى الـ target */
div = order_base_2(DIV_ROUND_UP(spi->clk_rate, speed_hz)) - 1;
```

#### الحل
```bash
# بعد الـ fix
cat /sys/kernel/debug/spi-stm32.0/actual_clock_hz
# طلع: 20000000  ← صح

# التحقق من الـ sensor data
cat /sys/bus/iio/devices/iio:device0/in_pressure_input
# طلع: 1013.25  ← قيمة منطقية للضغط الجوي
```

#### الدرس المستفاد
**الـ `DEFINE_DEBUGFS_ATTRIBUTE` macro** بيوفر safe و clean طريقة لـ expose computed values (مش بس raw variables). في الـ bring-up، استخدمه لأي قيمة محسوبة من registers — الـ clock rate، الـ baud rate، الـ voltage، إلخ. أسرع بكتير من الـ oscilloscope للـ quick sanity check.

---

### السيناريو 4: Automotive ECU على i.MX8 — تشخيص race condition في CAN bus driver

#### العنوان
**CAN bus driver بيـ crash بشكل عشوائي في i.MX8 ECU — debugfs_cancellation كشف الـ race**

#### السياق
i.MX8-based ECU في سيارة كهربائية. الـ CAN bus driver بيدي للـ userspace أداة تشخيص بتفتح `/sys/kernel/debug/can0/stats` وتقرأ منها. بعض الأوقات لما الـ CAN interface بيتوقف (مثلاً عند shutdown) والملف لسا مفتوح، بيحصل kernel oops.

#### المشكلة
الـ `release` للـ CAN driver بيعمل `kfree` للـ stats buffer بينما الـ debugfs read operation لسا شغالة — classic use-after-free.

#### التحليل
الكود الأصلي:

```c
/* خطأ: مفيش protection بين remove وread */
static ssize_t can_stats_read(struct file *file, char __user *buf,
                               size_t count, loff_t *ppos)
{
    struct can_priv *priv = file->private_data;  /* ممكن يبقى dangling! */
    /* ... قراءة من priv->stats ... */
}
```

الحل الصح هو استخدام `debugfs_file_get()` و`debugfs_file_put()` أو `debugfs_enter_cancellation()`:

```c
/* الـ struct debugfs_cancellation يخلي الـ remove ينتظر أو يلغي الـ read */
static ssize_t can_stats_read(struct file *file, char __user *buf,
                               size_t count, loff_t *ppos)
{
    struct debugfs_cancellation cancel;
    struct can_priv *priv;
    ssize_t ret;

    /*
     * debugfs_enter_cancellation() بتسجل الـ callback —
     * لو debugfs_remove() اتبعت أثناء هذه العملية،
     * الـ cancel callback بيتنفذ ويرجع -ENODEV للـ read
     */
    debugfs_enter_cancellation(file, &cancel);

    priv = file->private_data;
    if (!priv) {
        ret = -ENODEV;
        goto out;
    }

    ret = simple_read_from_buffer(buf, count, ppos,
                                  priv->stats_buf, priv->stats_len);
out:
    debugfs_leave_cancellation(file, &cancel);
    return ret;
}
```

الـ `debugfs_cancellation` struct (بيتعرف بـ `context_lock_struct`) بيحتفظ بـ `list` لـ tracking، `cancel` callback pointer، و`cancel_data`. الـ `debugfs_enter_cancellation()` بيعمل `__acquires(cancellation)` يعني الـ lockdep بيعرف إن في lock مأخوذ — لو حصل remove وفي read جارية، الـ cancel callback بيتنفذ بدل ما يحصل race.

#### الحل
```bash
# simulate الـ race condition للاختبار
# terminal 1: اقرأ في loop
while true; do cat /sys/kernel/debug/can0/stats; done &

# terminal 2: remove وأضف الـ interface
ip link set can0 down
# بدون الـ fix: kernel oops
# بعد الـ fix: -ENODEV clean error
```

```c
/* في can_remove() */
debugfs_remove(priv->debugfs_dir);  /* ينتظر أي read جارية تخلص */
kfree(priv->stats_buf);             /* آمن الآن */
```

#### الدرس المستفاد
الـ **`debugfs_cancellation`** mechanism ضروري في أي driver بيعمل `remove` — خصوصاً في embedded systems زي automotive اللي بيعمل hotplug للـ interfaces. الـ `debugfs_file_get()`/`debugfs_file_put()` بتحل نفس المشكلة بطريقة أبسط لو مش محتاج custom cancel logic.

---

### السيناريو 5: Custom Board Bring-up على AM62x — تشخيص I2C PMIC initialization

#### العنوان
**PMIC مش بيـ initialize صح على AM62x custom board — debugfs_regset32 كشف الـ wrong register values**

#### السياق
custom board bring-up بتستخدم TI AM62x SoC مع TPS65219 PMIC متوصل عبر I2C. البورد بتبوت لكن الـ voltage rails بيجوا على قيم غلط — الـ core voltage على 0.85V بدل 1.1V. الـ system unstable.

#### المشكلة
الـ PMIC driver عنده bug في الـ register initialization sequence — بيكتب الـ voltage value في الـ register الغلط أو بـ wrong bit mapping.

#### التحليل
المهندس أضاف `debugfs_regset32` لعرض كل الـ PMIC registers المهمة:

```c
/* في tps65219_probe() */
static const struct debugfs_reg32 tps65219_regs[] = {
    { "BUCK1_VOUT",   0x01 },
    { "BUCK2_VOUT",   0x02 },
    { "BUCK3_VOUT",   0x03 },
    { "LDO1_VOUT",    0x04 },
    { "LDO2_VOUT",    0x05 },
    { "ENABLE_CTRL",  0x10 },
    { "SEQUENCE_CTRL",0x11 },
    { "INT_STATUS",   0x1F },
};

struct debugfs_regset32 *regset;
regset = devm_kzalloc(dev, sizeof(*regset), GFP_KERNEL);
regset->regs  = tps65219_regs;
regset->nregs = ARRAY_SIZE(tps65219_regs);
regset->base  = priv->regmap_base;  /* mapped I2C register space */
regset->dev   = dev;                /* للـ Runtime PM — بيعمل pm_runtime_get قبل الـ read */

priv->debugfs = debugfs_create_dir("tps65219", NULL);
debugfs_create_regset32("regs", 0444, priv->debugfs, regset);

/* عرض الـ target voltages اللي الـ driver حسبها */
debugfs_create_u32("buck1_mv_target", 0444, priv->debugfs,
                   &priv->buck1_mv);
debugfs_create_u32("buck1_reg_val",   0444, priv->debugfs,
                   &priv->buck1_reg_val);
```

القراءة:

```bash
# اقرأ كل الـ registers
cat /sys/kernel/debug/tps65219/regs
# BUCK1_VOUT = 0x14  ← decimal 20
# المفروض لـ 1.1V يكون 0x24 (decimal 36) حسب الـ datasheet

# اقرأ الـ target
cat /sys/kernel/debug/tps65219/buck1_mv_target
# 1100  ← الـ driver حسب صح (1.1V)

cat /sys/kernel/debug/tps65219/buck1_reg_val
# 20  ← حول 1.1V لـ 20، المفروض 36!
```

الـ `debugfs_regset32` بيقرأ الـ registers فعلياً من الـ hardware (مع مراعاة الـ Runtime PM عبر الـ `dev` pointer في الـ struct) وبيعرضهم عبر `debugfs_print_regs32()` في format قابل للقراءة. المهندس لقى إن formula التحويل غلط:

```c
/* غلط: offset من 0.5V بـ steps 25mV */
reg_val = (mv - 500) / 25;
/* لـ 1100mv: (1100-500)/25 = 24 = 0x18 — قريب لكن لسا غلط */

/* صح: offset من 0.6V بـ steps 20mV حسب TPS65219 datasheet */
reg_val = (mv - 600) / 20;
/* لـ 1100mv: (1100-600)/20 = 25 = 0x19... */
/* الحل الصح: استخدام الـ voltage table من الـ datasheet مباشرة */
```

#### الحل
```c
/* voltage table مأخوذة مباشرة من TPS65219 datasheet Table 8-3 */
static const u16 tps65219_buck_voltages[] = {
    600, 610, 620, /* ... */
    1090, 1100,    /* index 25 = 1.1V */
    /* ... */
};

static int tps65219_mv_to_reg(u32 mv)
{
    int i;
    for (i = 0; i < ARRAY_SIZE(tps65219_buck_voltages); i++)
        if (tps65219_buck_voltages[i] == mv)
            return i;
    return -EINVAL;
}
```

```bash
# بعد الـ fix
cat /sys/kernel/debug/tps65219/regs
# BUCK1_VOUT = 0x19  ← صح!

cat /sys/kernel/debug/tps65219/buck1_reg_val
# 25  ← 1.1V

# تحقق من الـ actual voltage بـ multimeter: 1.10V
```

#### الدرس المستفاد
**الـ `debugfs_regset32` مع `dev` pointer** بيعمل Runtime PM تلقائياً قبل قراءة الـ registers — مهم جداً للـ PMIC والـ peripherals اللي بتدخل sleep. في الـ board bring-up، اعمل regset لكل register block مهم قبل ما تبدأ تكتب أي initialization code — يوفر ساعات من الـ debugging بـ oscilloscope.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

**الـ LWN.net** هو المرجع الأساسي لمتابعة تطور debugfs في kernel — كل المقالات دي مبنية على patches حقيقية ونقاشات mailing list:

| المقال | الوصف |
|--------|-------|
| [Debugfs — المقال التعريفي الأصلي](https://lwn.net/Articles/115405/) | أول coverage لـ debugfs لما Greg Kroah-Hartman قدّمه سنة 2004 |
| [debugfs - yet another in-kernel file system](https://lwn.net/Articles/115282/) | النقاش الأصلي على الـ mailing list حول تصميم debugfs كـ filesystem مستقل |
| [An updated guide to debugfs](https://lwn.net/Articles/334546/) | دليل محدّث شامل بيغطي الـ API بعد تطوره — مرجع عملي ممتاز |
| [debugfs mount point](https://lwn.net/Articles/323307/) | نقاش مهم حول standardization مكان mount الـ debugfs على `/sys/kernel/debug` |
| [docs, debugfs: start explicit debugfs documentation](https://lwn.net/Articles/759411/) | الـ patch اللي بدأ توثيق debugfs الرسمي في `Documentation/` |
| [debugfs: add tools to printk 32-bit registers](https://lwn.net/Articles/467447/) | إضافة `debugfs_regset32` و `debugfs_print_regs32` — موجودين في الـ header |
| [debugfs: New debugfs interface for creation of files, directory and symlinks](https://lwn.net/Articles/495599/) | تغييرات في الـ API لتبسيط إنشاء entries |
| [debugfs: Add access restriction option](https://lwn.net/Articles/822863/) | إضافة `CONFIG_DEBUG_FS_ALLOW_ALL` / `ALLOW_NONE` / `ALLOW_RELATED` — مهم لـ security |
| [debugfs: rules not welcome](https://lwn.net/Articles/429321/) | نقاش فلسفي مهم — ليه debugfs مش ليه ABI stability rules |
| [DebugFS on Rust](https://lwn.net/Articles/1041095/) | الـ Rust bindings لـ debugfs — أحدث تطور في الـ subsystem |
| [Kernel lockdown](https://lwn.net/Articles/706637/) | الـ lockdown mechanism اللي بيأثر على debugfs في secure boot |
| [mm: introduce shrinker debugfs interface](https://lwn.net/Articles/892329/) | مثال حديث لاستخدام debugfs في memory management |

---

### التوثيق الرسمي في kernel

#### ملفات Documentation/

```
Documentation/filesystems/debugfs.rst     ← الدليل الرسمي للـ API
Documentation/filesystems/debugfs.txt     ← النسخة القديمة (للمرجعية)
Documentation/admin-guide/sysctl/kernel.rst  ← إعدادات kernel المتعلقة بالـ debug
Documentation/security/                   ← سياسات الأمان والـ lockdown
```

**الـ kernel documentation** على الويب:
- [DebugFS — The Linux Kernel documentation](https://docs.kernel.org/filesystems/debugfs.html) — الصفحة الرسمية الكاملة
- [kernel.org raw text version](https://www.kernel.org/doc/Documentation/filesystems/debugfs.txt) — النسخة النصية القديمة

#### source files الأساسية في kernel

```
include/linux/debugfs.h          ← الـ public API (الملف اللي بندرسه)
fs/debugfs/inode.c               ← تنفيذ filesystem operations
fs/debugfs/file.c                ← تنفيذ file operations والـ helpers
fs/debugfs/internal.h            ← internal structures
```

---

### Kernel Commits المهمة

**الـ commits** دي بتحكي تاريخ تطور debugfs:

| الموضوع | البحث في git |
|---------|-------------|
| الـ commit الأصلي | `git log --all --grep="debugfs" --author="kroah" -- fs/debugfs/` |
| إضافة `debugfs_regset32` | `git log --all --grep="debugfs.*regset"` |
| تحويل `debugfs_remove` ليشمل recursive | `git log --grep="debugfs_remove_recursive"` |
| إضافة `debugfs_file_get/put` | `git log --grep="debugfs_file_get"` |
| إضافة الـ cancellation API | `git log --grep="debugfs_cancellation"` |
| إضافة `debugfs_short_fops` | `git log --grep="debugfs_short_fops"` |

```bash
# للبحث في تاريخ debugfs كاملاً
git log --oneline -- fs/debugfs/ include/linux/debugfs.h | head -50

# لمشاهدة تطور الـ API
git log --oneline --follow include/linux/debugfs.h
```

**الـ GitHub mirror** للبحث السريع:
- [torvalds/linux — debugfs.h history](https://github.com/torvalds/linux/commits/master/include/linux/debugfs.h)
- [torvalds/linux — fs/debugfs/ history](https://github.com/torvalds/linux/commits/master/fs/debugfs/)

---

### نقاشات Mailing List

الأرشيفات المفيدة للبحث في نقاشات debugfs:

- [LKML Archive — linux-kernel@vger.kernel.org](https://lore.kernel.org/linux-kernel/) — ابحث عن `debugfs` في subject
- [lore.kernel.org](https://lore.kernel.org/) — البحث: `s:debugfs l:linux-kernel`

**نقاشات بارزة:**
- نقاش `debugfs: rules not welcome` على [LWN.net/429321](https://lwn.net/Articles/429321/) — بيوضح قرار عدم تطبيق ABI stability
- نقاش kernel lockdown وتأثيره على debugfs على [LWN.net/822863](https://lwn.net/Articles/822863/)

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الكتاب المجاني**: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- **الفصل المتعلق**: Chapter 4 — *Debugging Techniques*
- بيغطي debugfs كبديل لـ /proc في debugging
- ملاحظة: الكتاب قديم (2005) لكن المفاهيم الأساسية لسه سارية

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المتعلق**: Chapter 18 — *Debugging the Linux Kernel*
- بيشرح متى تستخدم debugfs مقارنةً بـ printk وـ /proc
- مرجع ممتاز للفهم الأعمق لـ virtual filesystems في kernel

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المتعلق**: Chapter 14 — *Kernel Debugging Techniques*
- بيركز على استخدام debugfs في embedded systems
- أمثلة عملية على hardware register dumps باستخدام `debugfs_regset32`

#### Linux Kernel in a Nutshell — Greg Kroah-Hartman
- متاح مجاناً: [http://www.kroah.com/lkn/](http://www.kroah.com/lkn/)
- **مؤلفه هو نفسه** مبتكر debugfs — مرجع أساسي

---

### صفحات KernelNewbies وeLinux

**الـ kernelnewbies.org** — تغييرات debugfs في كل kernel version:

| الـ Kernel Version | الرابط | أبرز تغييرات debugfs |
|-------------------|--------|----------------------|
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) | أحدث تحديثات |
| Linux 6.9 | [kernelnewbies.org/Linux_6.9](https://kernelnewbies.org/Linux_6.9) | تحسينات الـ API |
| Linux 5.16 | [kernelnewbies.org/Linux_5.16](https://kernelnewbies.org/Linux_5.16) | security restrictions |
| Linux 4.19 | [kernelnewbies.org/Linux_4.19](https://kernelnewbies.org/Linux_4.19) | driver core integration |
| LinuxChanges | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) | الصفحة الرئيسية للتغييرات |

**الـ elinux.org** — استخدامات عملية:

| الصفحة | الصلة بـ debugfs |
|--------|-----------------|
| [Ftrace](https://elinux.org/Ftrace) | debugfs كـ interface للـ tracing في `/sys/kernel/debug/tracing/` |
| [OMAP Power Management](https://elinux.org/OMAP_Power_Management) | استخدام debugfs لعرض power domain states |
| [Debugging The Linux Kernel Using GDB](https://www.elinux.org/DebuggingTheLinuxKernelUsingGdb) | مقارنة debugfs مع GDB في kernel debugging |
| [OMAP Power Management/SmartReflex](https://elinux.org/OMAP_Power_Management/SmartReflex) | debugfs لـ SmartReflex voltage control |

---

### مراجع إضافية

#### توثيق عملي خارجي
- [How to Use debugfs — Timesys LinuxLink](https://linuxlink.timesys.com/docs/wiki/engineering/HOWTO_Use_debugfs) — دليل عملي خطوة بخطوة
- [debugfs — Wikipedia](https://en.wikipedia.org/wiki/Debugfs) — نظرة عامة وتاريخ
- [kernel_lockdown(7) man page](https://www.man7.org/linux/man-pages/man7/kernel_lockdown.7.html) — تأثير lockdown على debugfs

#### أدوات البحث في الـ kernel

```bash
# عرض debugfs entries الموجودة على النظام
ls -la /sys/kernel/debug/

# تفعيل debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# فهم config options
grep -r "DEBUG_FS" /boot/config-$(uname -r)
# أو
zcat /proc/config.gz | grep DEBUG_FS
```

---

### كلمات البحث الموصى بها

للبحث عن معلومات إضافية، استخدم الـ search terms دي:

```
# للبحث العام
"linux kernel debugfs"
"debugfs API tutorial"
"debugfs_create_file example"
"debugfs vs procfs vs sysfs"

# للبحث عن security
"debugfs lockdown"
"CONFIG_DEBUG_FS_ALLOW_ALL"
"debugfs access restriction"

# للبحث عن implementation
"fs/debugfs/file.c"
"debugfs_file_get put race condition"
"debugfs cancellation"

# للبحث عن use cases
"debugfs regset32 register dump"
"debugfs seq_file"
"DEFINE_DEBUGFS_ATTRIBUTE example"
"debugfs_create_devm_seqfile"

# في git log
git log --oneline --all -- fs/debugfs/
git log --oneline --all -- include/linux/debugfs.h
```

---

### ملخص سريع للمراجع حسب الأولوية

| الأولوية | المرجع | الاستخدام |
|---------|--------|----------|
| ★★★★★ | [docs.kernel.org/filesystems/debugfs.html](https://docs.kernel.org/filesystems/debugfs.html) | الـ API الرسمي |
| ★★★★★ | [LWN: An updated guide to debugfs](https://lwn.net/Articles/334546/) | دليل عملي شامل |
| ★★★★☆ | [LWN: debugfs introduction](https://lwn.net/Articles/115405/) | فهم التاريخ والـ rationale |
| ★★★★☆ | `include/linux/debugfs.h` + `fs/debugfs/` | الـ source code نفسه |
| ★★★★☆ | LDD3 Chapter 4 | debugging techniques context |
| ★★★☆☆ | [LWN: debugfs access restriction](https://lwn.net/Articles/822863/) | security و lockdown |
| ★★★☆☆ | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) | متابعة التطورات |
## Phase 8: Writing simple module

### الفكرة

**`debugfs_create_dir`** هي function بسيطة وآمنة نعمل عليها kprobe — كل ما بتتعمل بتعني إن driver أو subsystem بيحاول يعمل directory جديد في `/sys/kernel/debug`. نتابعها علشان نعرف مين بيعمل directories في debugfs ووقت التحميل.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * debugfs_dir_probe.c
 *
 * kprobe on debugfs_create_dir — logs every caller that creates
 * a new directory inside /sys/kernel/debug.
 */

#include <linux/module.h>       /* module_init / module_exit / MODULE_* */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/debugfs.h>      /* struct dentry — type of parent arg */
#include <linux/sched.h>        /* current, task_comm_len */
#include <linux/uaccess.h>      /* not strictly needed here, good habit */

/* ------------------------------------------------------------------
 * kprobe pre-handler
 * Called just before debugfs_create_dir executes.
 *
 * Prototype of the hooked function:
 *   struct dentry *debugfs_create_dir(const char *name,
 *                                     struct dentry *parent);
 *
 * On x86-64:
 *   regs->di = first  arg (const char *name)
 *   regs->si = second arg (struct dentry *parent)
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract the directory name from the first argument register */
    const char *dir_name = (const char *)regs->di;

    /*
     * current->comm holds the name of the task making the call.
     * %s [%d] gives comm + PID for easy correlation with userspace tools.
     */
    pr_info("debugfs_probe: debugfs_create_dir(\"%s\") called by %s [pid=%d]\n",
            dir_name ? dir_name : "(null)",
            current->comm,
            current->pid);

    return 0; /* 0 = let the real function run normally */
}

/* ------------------------------------------------------------------
 * kprobe post-handler (optional)
 * Called right after debugfs_create_dir returns.
 * regs->ax on x86-64 contains the return value (pointer to dentry).
 * ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * regs->ax = returned struct dentry *.
     * IS_ERR_VALUE checks if the pointer encodes an error (like ERR_PTR).
     */
    if (IS_ERR_VALUE(regs->ax))
        pr_info("debugfs_probe: debugfs_create_dir FAILED (err=%ld)\n",
                (long)regs->ax);
    else
        pr_info("debugfs_probe: debugfs_create_dir succeeded, dentry=0x%lx\n",
                regs->ax);
}

/* ------------------------------------------------------------------
 * kprobe descriptor — one instance per hooked symbol
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "debugfs_create_dir", /* exact exported symbol name */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------
 * module_init
 * ------------------------------------------------------------------ */
static int __init debugfs_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("debugfs_probe: register_kprobe failed (%d)\n", ret);
        return ret;
    }

    pr_info("debugfs_probe: planted kprobe at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------
 * module_exit
 * ------------------------------------------------------------------ */
static void __exit debugfs_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("debugfs_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(debugfs_probe_init);
module_exit(debugfs_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-student");
MODULE_DESCRIPTION("kprobe on debugfs_create_dir to trace debugfs dir creation");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `module_init`, `module_exit`, `MODULE_LICENSE` — أساسية لأي module |
| `linux/kprobes.h` | **struct kprobe** وكل API الـ kprobes |
| `linux/debugfs.h` | نحتاج type الـ `struct dentry` اللي بيرجعه `debugfs_create_dir` |
| `linux/sched.h` | **`current`** — pointer للـ task اللي بتنفذ الكود دلوقتي، بيدينا `comm` و `pid` |

---

#### الـ `handler_pre`

**الـ `regs->di`** على x86-64 بيحمل الـ argument الأول للـ function اللي اتعمل فيها kprobe — وده `const char *name`. بنطبع اسم الـ directory مع اسم الـ process ورقمه علشان نعرف بالظبط مين عمل الـ create ووقت التحميل.

---

#### الـ `handler_post`

بعد ما `debugfs_create_dir` ترجع، **الـ `regs->ax`** بيحمل الـ return value (الـ `struct dentry *`). بنستخدم `IS_ERR_VALUE` علشان نعرف لو الـ create فشل وبيرجع error pointer زي `ERR_PTR(-ENODEV)` — ده مهم لأن debugfs بترجع error pointer مش NULL لما تفشل.

---

#### الـ `kp` struct

**`symbol_name`** بيقول للـ kernel فين تزرع الـ probe — بيدور على الـ symbol في الـ kallsyms. بنحدد الـ handlers هنا بدل ما نعملهم runtime assignment.

---

#### الـ `module_init` و `module_exit`

**`register_kprobe`** بتزرع الـ breakpoint في الـ kernel text segment لـ symbol محدد. لازم نعمل **`unregister_kprobe`** في الـ exit علشان لو شلنا الـ module والـ probe لسه موجودة هيعمل kernel panic لما الـ execution يوصل لنقطة الـ probe ومفيش handler يخدمها.

---

### طريقة التجربة

```bash
# بناء الـ module
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod debugfs_dir_probe.ko

# نحمل أي driver بيعمل debugfs directory — مثلاً
sudo modprobe e1000e

# نشوف الـ log
sudo dmesg | grep debugfs_probe

# إزالة الـ module
sudo rmmod debugfs_dir_probe
```

**مثال على الـ output:**

```
debugfs_probe: planted kprobe at debugfs_create_dir (ffffffffc0123456)
debugfs_probe: debugfs_create_dir("e1000e") called by kworker/u4:2 [pid=312]
debugfs_probe: debugfs_create_dir succeeded, dentry=0xffff888003a1c000
debugfs_probe: kprobe removed from debugfs_create_dir
```
